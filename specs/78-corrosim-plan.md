# 78. CorroSim — Corrosion Engineering and Protection Design Platform

## Implementation Plan

**MVP Scope:** Browser-based corrosion prediction platform with CO2 corrosion rate calculator implementing mechanistic de Waard-Milliams model enhanced with film formation kinetics, multiphase flow coupling (stratified/slug/annular regimes) for pipeline internal corrosion prediction along 1D pipeline profiles ≤100 km, water chemistry module calculating pH/CO2 partial pressure/ionic strength from gas and water composition using Pitzer ion interaction model compiled to WebAssembly for client-side execution, materials comparison engine with database of 50 common alloys (carbon steel through duplex stainless) and corrosion rates across CO2/H2S environments, NACE MR0175/ISO 15156 environmental cracking compliance checker for SSC/HIC susceptibility, interactive pipeline profile visualizer with SVG/Canvas showing corrosion rate along length with flow regime transitions, metal loss anomaly assessment using ASME B31G Modified and DNV-RP-F101 remaining strength calculations, corrosion allowance and remaining life calculator, Stripe billing with three tiers (Free / Pro $199/mo / Advanced $449/user/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| Corrosion Solver | Rust (native + WASM) | Custom mechanistic CO2/H2S corrosion model with film kinetics |
| WASM Compilation | `wasm-pack` + `wasm-bindgen` | Client-side corrosion rate calculation for pipelines ≤100 km |
| Water Chemistry | Python 3.12 (FastAPI) | Pitzer ion interaction model, pH/scaling equilibrium calculations |
| Database | PostgreSQL 16 | Projects, users, pipeline models, materials database, inspection data |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | Pipeline simulation results, inspection data imports, report PDFs |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| Pipeline Visualizer | SVG + Canvas 2D | Pipeline profile renderer with corrosion rate heatmap overlay |
| Materials Database | PostgreSQL + JSONB | 500+ alloy properties, corrosion rate curves, environmental limits |
| Real-time | WebSocket (Axum) | Live simulation progress for long pipeline runs |
| Job Queue | Redis 7 + Tokio tasks | Server-side pipeline simulation job management |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | WASM bundle delivery, static frontend assets |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, error tracking |
| CI/CD | GitHub Actions | WASM build pipeline, Rust tests, Docker image push |

### Key Architecture Decisions

1. **Hybrid client/server solver with WASM threshold at 100 km pipeline length**: Pipelines with ≤100 km length (covers 80%+ of flowline, gathering, and facility pigging segments) run entirely in the browser via WASM, providing instant corrosion rate profiles with zero server cost. Pipelines exceeding 100 km (long-distance transmission, offshore export lines) are submitted to the Rust-native server solver which handles multi-segment pipelines with elevation profiles and parametric studies. The threshold is configurable per plan tier.

2. **Custom mechanistic corrosion model in Rust rather than wrapping commercial tools**: Building a custom implementation of de Waard-Milliams, Norsok M-506, and proprietary film formation kinetics in Rust gives us full control over parameter sensitivity, WASM compilation, and integration with multiphase flow correlations. Commercial tools like OLI or Multicorp have licensing restrictions that prevent cloud deployment. Our Rust solver uses `nalgebra` for numerical integration of film growth ODEs while maintaining memory safety and WASM compatibility.

3. **Python FastAPI service for water chemistry rather than pure Rust**: Water chemistry calculations (Pitzer ion interaction model, multi-species equilibrium, scaling tendency) rely on SciPy optimization and existing validated Python libraries (Phreeqc bindings, Pyomo). Rewriting in Rust would introduce validation risk. The Python service is called from Rust backend for water composition → pH/partial pressure conversion, then corrosion model runs in Rust with those inputs. FastAPI is containerized separately and scales independently.

4. **SVG pipeline profile renderer with Canvas heatmap overlay**: The pipeline profile (elevation vs. distance) is rendered as SVG for crisp display of elevation changes, equipment locations, and inspection points. The corrosion rate heatmap is rendered on a Canvas overlay using color gradients (green → yellow → red for low → high corrosion) to avoid SVG performance degradation with 1000+ data points. This decoupling allows independent pan/zoom on profile geometry and corrosion rate updates without full re-render.

5. **PostgreSQL materials database with JSONB for corrosion rate curves**: Material properties (500+ alloys from carbon steel to nickel superalloys) are stored in PostgreSQL with JSONB columns for corrosion rate vs. temperature/pH/H2S curves, polarization data, and environmental cracking domain diagrams. This allows the materials database to scale to 10K+ alloys with flexible schema evolution while enabling fast parametric search via PostgreSQL indexes and GIN indexes on JSONB fields for environmental condition queries.

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";  -- For fuzzy text search on materials

-- Users
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email TEXT UNIQUE NOT NULL,
    password_hash TEXT,  -- NULL for OAuth-only users
    name TEXT NOT NULL,
    avatar_url TEXT,
    auth_provider TEXT DEFAULT 'email',  -- email | google | github
    auth_provider_id TEXT,
    plan TEXT NOT NULL DEFAULT 'free',  -- free | pro | advanced
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX users_email_idx ON users(email);
CREATE INDEX users_stripe_idx ON users(stripe_customer_id);

-- Organizations (for Advanced plan)
CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    slug TEXT UNIQUE NOT NULL,
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    plan TEXT NOT NULL DEFAULT 'free',
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    settings JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX orgs_slug_idx ON organizations(slug);
CREATE INDEX orgs_owner_idx ON organizations(owner_id);

-- Organization Members
CREATE TABLE org_members (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    org_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role TEXT NOT NULL DEFAULT 'member',  -- owner | admin | member
    joined_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(org_id, user_id)
);
CREATE INDEX org_members_org_idx ON org_members(org_id);
CREATE INDEX org_members_user_idx ON org_members(user_id);

-- Pipeline Projects
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    project_type TEXT NOT NULL DEFAULT 'pipeline',  -- pipeline | vessel | offshore_structure
    pipeline_data JSONB NOT NULL DEFAULT '{}',  -- Pipeline geometry, segments, fluids
    settings JSONB DEFAULT '{}',  -- Units, solver settings
    is_public BOOLEAN NOT NULL DEFAULT false,
    forked_from UUID REFERENCES projects(id) ON DELETE SET NULL,
    thumbnail_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);
CREATE INDEX projects_org_idx ON projects(org_id);
CREATE INDEX projects_type_idx ON projects(project_type);
CREATE INDEX projects_updated_idx ON projects(updated_at DESC);

-- Corrosion Simulations
CREATE TABLE simulations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    simulation_type TEXT NOT NULL,  -- co2_corrosion | h2s_corrosion | galvanic | environmental_cracking
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | running | completed | failed | cancelled
    execution_mode TEXT NOT NULL DEFAULT 'wasm',  -- wasm | server
    input_data JSONB NOT NULL DEFAULT '{}',  -- Fluid composition, flow conditions, material
    parameters JSONB NOT NULL DEFAULT '{}',  -- Model parameters, options
    pipeline_length_km REAL NOT NULL DEFAULT 0.0,
    results_url TEXT,  -- S3 URL for result data
    results_summary JSONB,  -- Quick-access summary (max corrosion rate, critical locations)
    error_message TEXT,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    wall_time_ms INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX sims_project_idx ON simulations(project_id);
CREATE INDEX sims_user_idx ON simulations(user_id);
CREATE INDEX sims_status_idx ON simulations(status);
CREATE INDEX sims_created_idx ON simulations(created_at DESC);

-- Server Simulation Jobs (for server-side execution only)
CREATE TABLE simulation_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    simulation_id UUID NOT NULL REFERENCES simulations(id) ON DELETE CASCADE,
    worker_id TEXT,
    cores_allocated INTEGER DEFAULT 1,
    memory_mb INTEGER DEFAULT 2048,
    priority INTEGER DEFAULT 0,  -- Higher = more urgent
    progress_pct REAL DEFAULT 0.0,
    progress_data JSONB DEFAULT '{}',  -- Current position, iteration counts
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX jobs_sim_idx ON simulation_jobs(simulation_id);
CREATE INDEX jobs_status_idx ON simulation_jobs(worker_id);

-- Materials Database
CREATE TABLE materials (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,  -- e.g., "ASTM A106 Grade B", "AISI 316L"
    material_class TEXT NOT NULL,  -- carbon_steel | low_alloy_steel | stainless_steel | duplex | nickel_alloy | titanium | copper
    uns_number TEXT,  -- Unified Numbering System (e.g., "S31603" for 316L)
    composition JSONB NOT NULL,  -- Chemical composition in wt% (Fe, Cr, Ni, Mo, C, etc.)
    mechanical_properties JSONB,  -- Yield, tensile, hardness
    corrosion_data JSONB NOT NULL,  -- Corrosion rate curves vs. environment
    environmental_limits JSONB,  -- NACE MR0175 limits, max H2S, max temperature
    cost_per_kg REAL,  -- Material cost for economics
    density REAL,  -- kg/m³
    tags TEXT[] DEFAULT '{}',
    is_verified BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX materials_class_idx ON materials(material_class);
CREATE INDEX materials_name_trgm_idx ON materials USING gin(name gin_trgm_ops);
CREATE INDEX materials_uns_idx ON materials(uns_number);
CREATE INDEX materials_tags_idx ON materials USING gin(tags);
CREATE INDEX materials_corrosion_idx ON materials USING gin(corrosion_data);

-- Inspection Data (imported from ILI, UT, coupon tests)
CREATE TABLE inspection_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    inspection_date DATE NOT NULL,
    inspection_type TEXT NOT NULL,  -- ili_mfl | ili_ut | manual_ut | coupon | er_probe
    location_km REAL,  -- Distance along pipeline
    measurement_type TEXT NOT NULL,  -- wall_thickness | corrosion_rate | pitting_depth
    value REAL NOT NULL,
    unit TEXT NOT NULL,
    metadata JSONB DEFAULT '{}',  -- Run ID, clock position, feature ID
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX inspection_project_idx ON inspection_records(project_id);
CREATE INDEX inspection_date_idx ON inspection_records(inspection_date DESC);
CREATE INDEX inspection_location_idx ON inspection_records(location_km);

-- Anomaly Assessments (B31G, DNV-RP-F101)
CREATE TABLE anomalies (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    inspection_record_id UUID REFERENCES inspection_records(id) ON DELETE SET NULL,
    location_km REAL NOT NULL,
    length_mm REAL NOT NULL,
    depth_mm REAL NOT NULL,
    width_mm REAL,
    clock_position INTEGER,  -- 0-12 o'clock
    assessment_method TEXT NOT NULL,  -- b31g_modified | b31g_effective_area | dnv_rp_f101
    safe_pressure_mpa REAL,
    failure_pressure_mpa REAL,
    design_pressure_mpa REAL,
    safety_factor REAL,
    status TEXT NOT NULL DEFAULT 'active',  -- active | repaired | monitored
    assessed_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX anomalies_project_idx ON anomalies(project_id);
CREATE INDEX anomalies_location_idx ON anomalies(location_km);
CREATE INDEX anomalies_status_idx ON anomalies(status);

-- Saved Reports
CREATE TABLE reports (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    report_type TEXT NOT NULL,  -- corrosion_assessment | materials_selection | rbi | anomaly_report
    report_data JSONB NOT NULL,
    pdf_url TEXT,  -- S3 URL for generated PDF
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX reports_project_idx ON reports(project_id);
CREATE INDEX reports_type_idx ON reports(report_type);

-- Usage Tracking (for billing enforcement)
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,  -- simulation_minutes | api_calls | storage_bytes
    quantity REAL NOT NULL,
    period_start DATE NOT NULL,
    period_end DATE NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX usage_user_period_idx ON usage_records(user_id, period_start, period_end);
```

### Rust SQLx Structs

```rust
// src/db/models.rs

use chrono::{DateTime, Utc, NaiveDate};
use serde::{Deserialize, Serialize};
use sqlx::FromRow;
use uuid::Uuid;

#[derive(Debug, FromRow, Serialize)]
pub struct User {
    pub id: Uuid,
    pub email: String,
    #[sqlx(skip)]
    #[serde(skip)]
    pub password_hash: Option<String>,
    pub name: String,
    pub avatar_url: Option<String>,
    pub auth_provider: String,
    pub auth_provider_id: Option<String>,
    pub plan: String,
    pub stripe_customer_id: Option<String>,
    pub stripe_subscription_id: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Project {
    pub id: Uuid,
    pub owner_id: Uuid,
    pub org_id: Option<Uuid>,
    pub name: String,
    pub description: String,
    pub project_type: String,
    pub pipeline_data: serde_json::Value,
    pub settings: serde_json::Value,
    pub is_public: bool,
    pub forked_from: Option<Uuid>,
    pub thumbnail_url: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Simulation {
    pub id: Uuid,
    pub project_id: Uuid,
    pub user_id: Uuid,
    pub simulation_type: String,
    pub status: String,
    pub execution_mode: String,
    pub input_data: serde_json::Value,
    pub parameters: serde_json::Value,
    pub pipeline_length_km: f32,
    pub results_url: Option<String>,
    pub results_summary: Option<serde_json::Value>,
    pub error_message: Option<String>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub wall_time_ms: Option<i32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Material {
    pub id: Uuid,
    pub name: String,
    pub material_class: String,
    pub uns_number: Option<String>,
    pub composition: serde_json::Value,
    pub mechanical_properties: Option<serde_json::Value>,
    pub corrosion_data: serde_json::Value,
    pub environmental_limits: Option<serde_json::Value>,
    pub cost_per_kg: Option<f32>,
    pub density: Option<f32>,
    pub tags: Vec<String>,
    pub is_verified: bool,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct InspectionRecord {
    pub id: Uuid,
    pub project_id: Uuid,
    pub inspection_date: NaiveDate,
    pub inspection_type: String,
    pub location_km: Option<f32>,
    pub measurement_type: String,
    pub value: f32,
    pub unit: String,
    pub metadata: serde_json::Value,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Anomaly {
    pub id: Uuid,
    pub project_id: Uuid,
    pub inspection_record_id: Option<Uuid>,
    pub location_km: f32,
    pub length_mm: f32,
    pub depth_mm: f32,
    pub width_mm: Option<f32>,
    pub clock_position: Option<i32>,
    pub assessment_method: String,
    pub safe_pressure_mpa: Option<f32>,
    pub failure_pressure_mpa: Option<f32>,
    pub design_pressure_mpa: Option<f32>,
    pub safety_factor: Option<f32>,
    pub status: String,
    pub assessed_at: DateTime<Utc>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct CO2CorrosionInput {
    pub temperature_c: f64,
    pub pressure_mpa: f64,
    pub co2_partial_pressure_mpa: f64,
    pub h2s_partial_pressure_mpa: Option<f64>,
    pub ph: Option<f64>,  // Calculated if not provided
    pub flow_velocity_m_s: f64,
    pub pipe_diameter_mm: f64,
    pub water_cut: f64,  // Volume fraction water
    pub chloride_ppm: f64,
    pub acetic_acid_ppm: Option<f64>,
    pub material_name: String,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct PipelineProfile {
    pub segments: Vec<PipelineSegment>,
    pub total_length_km: f64,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct PipelineSegment {
    pub start_km: f64,
    pub end_km: f64,
    pub elevation_m: f64,
    pub diameter_mm: f64,
    pub wall_thickness_mm: f64,
    pub material_name: String,
    pub fluid_conditions: FluidConditions,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct FluidConditions {
    pub pressure_mpa: f64,
    pub temperature_c: f64,
    pub gas_rate_m3_d: f64,
    pub oil_rate_m3_d: Option<f64>,
    pub water_rate_m3_d: f64,
    pub gas_composition: GasComposition,
    pub water_composition: WaterComposition,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct GasComposition {
    pub co2_mole_frac: f64,
    pub h2s_mole_frac: Option<f64>,
    pub ch4_mole_frac: f64,
    pub n2_mole_frac: Option<f64>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct WaterComposition {
    pub salinity_ppm: f64,
    pub chloride_ppm: f64,
    pub bicarbonate_ppm: f64,
    pub ph: Option<f64>,  // Can be calculated
}
```

---

## Solver Architecture Deep-Dive

### Governing Equations and Discretization

CorroSim's core solver implements **mechanistic electrochemical corrosion models** based on de Waard-Milliams, Norsok M-506, and proprietary film formation kinetics. For CO2 corrosion of carbon steel, the corrosion rate is controlled by:

```
Corrosion rate = f(electrochemistry, mass transfer, protective film)

V_corr = (i_corr × M) / (n × F × ρ)

where:
- i_corr: corrosion current density (A/m²)
- M: atomic mass of iron (55.845 g/mol)
- n: charge transfer number (2 for Fe → Fe²⁺)
- F: Faraday constant (96,485 C/mol)
- ρ: density of iron (7,874 kg/m³)
```

**Electrochemical kinetics** (cathodic and anodic reactions):

Cathodic (H⁺ reduction in CO2 environment):
```
H⁺ + e⁻ → ½H₂             (direct reduction)
H₂CO₃ + e⁻ → H + HCO₃⁻    (carbonic acid reduction)
HCO₃⁻ + e⁻ → H + CO₃²⁻    (bicarbonate reduction)
```

Anodic (iron dissolution):
```
Fe → Fe²⁺ + 2e⁻
```

The corrosion current density is determined by Butler-Volmer kinetics and mass transfer limiting current:

```
i_corr = min(i_electrochemical, i_mass_transfer)

i_electrochemical = i₀ × [exp(βa×η/RT) - exp(-βc×η/RT)]

i_mass_transfer = n×F×k_m×C_bulk

where:
- i₀: exchange current density (function of T, pH, pCO2)
- βa, βc: anodic and cathodic Tafel slopes
- η: overpotential
- k_m: mass transfer coefficient (from Sherwood number correlation)
- C_bulk: bulk concentration of H⁺ or H₂CO₃
```

**Film formation kinetics** (FeCO3 protective scale):

The formation of iron carbonate (siderite) film provides protection by reducing ion transport:

```
dδ/dt = k_precip × (S - 1) - k_dissolution

where:
- δ: film thickness
- S: supersaturation = (a_Fe²⁺ × a_CO₃²⁻) / K_sp
- K_sp: solubility product (function of temperature)
- k_precip: precipitation rate constant
- k_dissolution: film dissolution rate
```

Film effect on corrosion rate:
```
V_corr,film = V_corr,bare / (1 + δ/δ_crit)^n_prot

where:
- δ_crit: critical thickness for full protection (~50 μm)
- n_prot: protection exponent (typically 2-3)
```

**Multiphase flow coupling** (flow regime effects on mass transfer):

Flow regime determines wall shear stress and liquid holdup, which affect mass transfer coefficient:

Stratified flow:
```
k_m = 0.023 × (Re_liq^0.83) × (Sc^0.33) × (D/D_h)
```

Slug flow:
```
k_m = k_m,stratified × (1 + 5.5 × f_slug)
where f_slug: slug frequency
```

Annular flow:
```
k_m = 0.05 × (Re_film^0.8) × (Sc^0.33) × (D/δ_film)
where δ_film: liquid film thickness
```

**Pipeline profile integration**:

Corrosion rate is calculated at discrete locations along the pipeline, accounting for pressure, temperature, and flow regime changes:

```
For each segment i from inlet to outlet:
  1. Calculate P(i), T(i) from pressure drop and heat transfer
  2. Calculate flow regime (stratified/slug/annular) from Taitel-Dukler map
  3. Calculate water pH from pCO2, T, salinity (Pitzer model)
  4. Calculate corrosion current density i_corr(i)
  5. Integrate film growth: δ(i+1) = δ(i) + dδ/dt × Δt
  6. Calculate corrosion rate: V_corr(i) = f(i_corr, δ)
```

### Client/Server Split (WASM Threshold)

```
Pipeline uploaded → Total length extracted
    │
    ├── ≤100 km → WASM solver (browser)
    │   ├── Instant startup, no network latency
    │   ├── Results displayed immediately
    │   └── No server cost
    │
    └── >100 km → Server solver (Rust native)
        ├── Job queued via Redis
        ├── Worker picks up, runs native solver
        ├── Progress streamed via WebSocket
        └── Results stored in S3, URL returned
```

The 100 km threshold was chosen because:
- WASM solver handles 100 km pipeline (500 segments @ 200m spacing) in <3 seconds on modern hardware
- 100 km covers: flowlines (5-20 km), gathering systems (10-50 km), processing facility piping (1-5 km)
- Above 100 km: long-distance transmission (500+ km), offshore export lines → need server compute for detailed elevation profiles and parametric studies

### WASM Compilation Pipeline

```toml
# corrosion-wasm/Cargo.toml
[package]
name = "corrosim-solver-wasm"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
wasm-bindgen = "0.2"
serde = { version = "1", features = ["derive"] }
serde-wasm-bindgen = "0.6"
js-sys = "0.3"
nalgebra = "0.33"
log = "0.4"
console_log = "1"

[profile.release]
opt-level = "z"       # Optimize for size
lto = true            # Link-time optimization
codegen-units = 1     # Single codegen unit for better optimization
strip = true          # Strip debug symbols
wasm-opt = true       # Run wasm-opt
```

---

## Architecture Deep-Dives

### 1. Corrosion Simulation API Handler (Rust/Axum)

The primary endpoint receives a corrosion simulation request, validates the pipeline data, decides between WASM and server execution, and for server execution enqueues a job with Redis and streams progress via WebSocket.

```rust
// src/api/handlers/simulation.rs

use axum::{
    extract::{Path, State, Json},
    http::StatusCode,
    response::IntoResponse,
};
use uuid::Uuid;
use crate::{
    db::models::{Simulation, CO2CorrosionInput, PipelineProfile},
    state::AppState,
    auth::Claims,
    error::ApiError,
};

#[derive(serde::Deserialize)]
pub struct CreateSimulationRequest {
    pub simulation_type: String,  // co2_corrosion | h2s_corrosion | galvanic
    pub pipeline_profile: PipelineProfile,
    pub parameters: serde_json::Value,
}

pub async fn create_simulation(
    State(state): State<AppState>,
    claims: Claims,
    Path(project_id): Path<Uuid>,
    Json(req): Json<CreateSimulationRequest>,
) -> Result<impl IntoResponse, ApiError> {
    // 1. Verify project ownership
    let project = sqlx::query_as!(
        crate::db::models::Project,
        "SELECT * FROM projects WHERE id = $1 AND (owner_id = $2 OR org_id IN (
            SELECT org_id FROM org_members WHERE user_id = $2
        ))",
        project_id,
        claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Project not found"))?;

    // 2. Validate pipeline profile and extract total length
    let total_length_km = req.pipeline_profile.total_length_km;

    if req.pipeline_profile.segments.is_empty() {
        return Err(ApiError::BadRequest("Pipeline must have at least one segment"));
    }

    // 3. Check plan limits
    let user = sqlx::query_as!(
        crate::db::models::User,
        "SELECT * FROM users WHERE id = $1",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    if user.plan == "free" && total_length_km > 10.0 {
        return Err(ApiError::PlanLimit(
            "Free plan supports pipelines up to 10 km. Upgrade to Pro for unlimited."
        ));
    }

    // 4. Determine execution mode
    let execution_mode = if total_length_km <= 100.0 {
        "wasm"
    } else {
        "server"
    };

    // 5. Create simulation record
    let input_data = serde_json::json!({
        "pipeline_profile": req.pipeline_profile,
    });

    let sim = sqlx::query_as!(
        Simulation,
        r#"INSERT INTO simulations
            (project_id, user_id, simulation_type, status, execution_mode,
             input_data, parameters, pipeline_length_km)
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8)
        RETURNING *"#,
        project_id,
        claims.user_id,
        req.simulation_type,
        if execution_mode == "wasm" { "completed" } else { "pending" },
        execution_mode,
        input_data,
        req.parameters,
        total_length_km as f32,
    )
    .fetch_one(&state.db)
    .await?;

    // 6. For server execution, enqueue job
    if execution_mode == "server" {
        let job = sqlx::query_as!(
            crate::db::models::SimulationJob,
            r#"INSERT INTO simulation_jobs (simulation_id, priority)
            VALUES ($1, $2) RETURNING *"#,
            sim.id,
            if user.plan == "advanced" { 10 } else { 0 },
        )
        .fetch_one(&state.db)
        .await?;

        // Publish to Redis job queue
        state.redis
            .publish("corrosion:jobs", serde_json::to_string(&job.id)?)
            .await?;
    }

    Ok((StatusCode::CREATED, Json(sim)))
}

pub async fn get_simulation(
    State(state): State<AppState>,
    claims: Claims,
    Path((project_id, sim_id)): Path<(Uuid, Uuid)>,
) -> Result<Json<Simulation>, ApiError> {
    let sim = sqlx::query_as!(
        Simulation,
        "SELECT s.* FROM simulations s
         JOIN projects p ON s.project_id = p.id
         WHERE s.id = $1 AND s.project_id = $2
         AND (p.owner_id = $3 OR p.org_id IN (
             SELECT org_id FROM org_members WHERE user_id = $3
         ))",
        sim_id,
        project_id,
        claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Simulation not found"))?;

    Ok(Json(sim))
}

pub async fn list_simulations(
    State(state): State<AppState>,
    claims: Claims,
    Path(project_id): Path<Uuid>,
) -> Result<Json<Vec<Simulation>>, ApiError> {
    let sims = sqlx::query_as!(
        Simulation,
        "SELECT s.* FROM simulations s
         JOIN projects p ON s.project_id = p.id
         WHERE s.project_id = $1
         AND (p.owner_id = $2 OR p.org_id IN (
             SELECT org_id FROM org_members WHERE user_id = $2
         ))
         ORDER BY s.created_at DESC
         LIMIT 100",
        project_id,
        claims.user_id
    )
    .fetch_all(&state.db)
    .await?;

    Ok(Json(sims))
}
```

### 2. CO2 Corrosion Solver Core (Rust — shared between WASM and native)

The core corrosion solver that implements de Waard-Milliams model with film kinetics. This code compiles to both native (server) and WASM (browser) targets.

```rust
// corrosion-core/src/co2_corrosion.rs

use nalgebra::DVector;

pub struct CO2CorrosionSolver {
    pub temperature_c: f64,
    pub pressure_mpa: f64,
    pub pco2_mpa: f64,
    pub ph2s_mpa: Option<f64>,
    pub ph: f64,
    pub velocity_m_s: f64,
    pub diameter_mm: f64,
    pub water_cut: f64,
    pub chloride_ppm: f64,
    pub film_thickness_um: f64,  // Current film thickness
}

impl CO2CorrosionSolver {
    pub fn new(input: &crate::models::CO2CorrosionInput) -> Self {
        Self {
            temperature_c: input.temperature_c,
            pressure_mpa: input.pressure_mpa,
            pco2_mpa: input.co2_partial_pressure_mpa,
            ph2s_mpa: input.h2s_partial_pressure_mpa,
            ph: input.ph.unwrap_or(5.0),  // Default, will be recalculated
            velocity_m_s: input.flow_velocity_m_s,
            diameter_mm: input.pipe_diameter_mm,
            water_cut: input.water_cut,
            chloride_ppm: input.chloride_ppm,
            film_thickness_um: 0.0,
        }
    }

    /// Calculate corrosion rate in mm/year using de Waard-Milliams model
    /// with film formation kinetics enhancement
    pub fn calculate_corrosion_rate(&self) -> CorrosionResult {
        // 1. Calculate bare steel corrosion rate (no film protection)
        let cr_bare = self.de_waard_milliams_rate();

        // 2. Calculate FeCO3 film supersaturation and growth
        let supersaturation = self.calculate_supersaturation();
        let film_protection_factor = self.film_protection_factor();

        // 3. Apply film protection
        let cr_film = cr_bare / film_protection_factor;

        // 4. Calculate mass transfer limiting rate
        let cr_mass_transfer = self.mass_transfer_limited_rate();

        // 5. Actual corrosion rate is minimum of electrochemical and mass transfer
        let corrosion_rate_mm_yr = cr_film.min(cr_mass_transfer);

        CorrosionResult {
            corrosion_rate_mm_yr,
            bare_rate_mm_yr: cr_bare,
            film_protection_factor,
            supersaturation,
            mass_transfer_rate_mm_yr: cr_mass_transfer,
            film_thickness_um: self.film_thickness_um,
        }
    }

    /// de Waard-Milliams mechanistic model for CO2 corrosion
    fn de_waard_milliams_rate(&self) -> f64 {
        let t_k = self.temperature_c + 273.15;

        // Electrochemical rate constant (Arrhenius temperature dependence)
        // log10(k) = A - B/T
        let a = 5.8;
        let b = 1710.0;
        let log_k = a - b / t_k;
        let k_electrochem = 10_f64.powf(log_k);

        // CO2 partial pressure effect (power law)
        let pco2_bar = self.pco2_mpa * 10.0;
        let pco2_effect = pco2_bar.powf(0.62);

        // pH effect (cathodic reaction rate)
        let ph_effect = 10_f64.powf(-self.ph);

        // Combined bare steel rate (mm/year)
        let cr_bare = k_electrochem * pco2_effect * ph_effect * 3.65e3;

        cr_bare
    }

    /// Calculate FeCO3 supersaturation for film formation
    fn calculate_supersaturation(&self) -> f64 {
        let t_k = self.temperature_c + 273.15;

        // Solubility product of FeCO3 (temperature dependent)
        // log10(Ksp) = -59.3498 - 0.041377×T - 2.1963/T + 24.5724×log10(T)
        let log_ksp = -59.3498 - 0.041377 * t_k - 2.1963 / t_k + 24.5724 * t_k.log10();
        let ksp = 10_f64.powf(log_ksp);

        // Estimate Fe²⁺ concentration from corrosion rate (assuming accumulation)
        let fe2_molarity = 1e-4;  // Typical range 1e-5 to 1e-3 M

        // CO3²⁻ concentration from pH and pCO2
        let h2co3_molarity = self.pco2_mpa * 10.0 / 33.0;  // Henry's law
        let ka1 = 10_f64.powf(-6.35);  // First dissociation constant
        let ka2 = 10_f64.powf(-10.33);  // Second dissociation constant
        let h_molarity = 10_f64.powf(-self.ph);
        let co3_molarity = (ka1 * ka2 * h2co3_molarity) / (h_molarity * h_molarity);

        // Supersaturation S = (a_Fe²⁺ × a_CO3²⁻) / Ksp
        let ion_product = fe2_molarity * co3_molarity;
        let supersaturation = ion_product / ksp;

        supersaturation.max(0.0)
    }

    /// Film protection factor based on current film thickness
    fn film_protection_factor(&self) -> f64 {
        let delta_crit = 50.0;  // Critical thickness for full protection (μm)
        let n_prot = 2.5;  // Protection exponent

        if self.film_thickness_um < 0.1 {
            return 1.0;  // No film, no protection
        }

        let protection = 1.0 + (self.film_thickness_um / delta_crit).powf(n_prot);
        protection
    }

    /// Mass transfer limited corrosion rate
    fn mass_transfer_limited_rate(&self) -> f64 {
        // Calculate Reynolds number
        let velocity = self.velocity_m_s;
        let diameter = self.diameter_mm / 1000.0;  // Convert to m
        let kinematic_viscosity = 1e-6;  // m²/s for water at ~20°C
        let re = velocity * diameter / kinematic_viscosity;

        // Schmidt number for H⁺ in water
        let diffusivity = 9e-9;  // m²/s
        let sc = kinematic_viscosity / diffusivity;

        // Sherwood number (mass transfer correlation)
        // Sh = 0.023 × Re^0.83 × Sc^0.33 (turbulent flow)
        let sh = if re > 2300.0 {
            0.023 * re.powf(0.83) * sc.powf(0.33)
        } else {
            // Laminar flow: Sh = 3.66 (fully developed)
            3.66
        };

        // Mass transfer coefficient (m/s)
        let k_m = sh * diffusivity / diameter;

        // Limiting current density (A/m²)
        let faraday = 96485.0;  // C/mol
        let h_concentration = 10_f64.powf(-self.ph);  // mol/L
        let h_concentration_m3 = h_concentration * 1000.0;  // mol/m³
        let i_lim = 2.0 * faraday * k_m * h_concentration_m3;

        // Convert current density to corrosion rate (mm/year)
        // CR = (i × M) / (n × F × ρ) × (365.25 × 24 × 3600) × 1000
        let m_fe = 55.845;  // g/mol
        let rho_fe = 7874.0;  // kg/m³ = g/L
        let n = 2.0;  // electrons per Fe atom
        let seconds_per_year = 365.25 * 24.0 * 3600.0;

        let cr_mt = (i_lim * m_fe * seconds_per_year * 1000.0) / (n * faraday * rho_fe * 1000.0);

        cr_mt
    }

    /// Update film thickness based on precipitation/dissolution kinetics
    pub fn update_film_thickness(&mut self, dt_hours: f64) {
        let supersaturation = self.calculate_supersaturation();

        if supersaturation > 1.0 {
            // Film precipitation
            let k_precip = 0.1;  // μm/hour at S=2
            let growth_rate = k_precip * (supersaturation - 1.0).powf(1.5);
            self.film_thickness_um += growth_rate * dt_hours;
        } else {
            // Film dissolution
            let k_diss = 0.05;  // μm/hour
            let dissolution_rate = k_diss * (1.0 - supersaturation);
            self.film_thickness_um -= dissolution_rate * dt_hours;
            self.film_thickness_um = self.film_thickness_um.max(0.0);
        }

        // Cap maximum film thickness
        self.film_thickness_um = self.film_thickness_um.min(200.0);
    }
}

#[derive(Debug, Clone)]
pub struct CorrosionResult {
    pub corrosion_rate_mm_yr: f64,
    pub bare_rate_mm_yr: f64,
    pub film_protection_factor: f64,
    pub supersaturation: f64,
    pub mass_transfer_rate_mm_yr: f64,
    pub film_thickness_um: f64,
}

/// Calculate corrosion rate profile along pipeline
pub fn solve_pipeline_profile(
    pipeline: &crate::models::PipelineProfile,
) -> Result<PipelineCorrosionProfile, SolverError> {
    let mut results = Vec::new();

    for segment in &pipeline.segments {
        // Calculate average conditions for segment
        let input = CO2CorrosionInput {
            temperature_c: segment.fluid_conditions.temperature_c,
            pressure_mpa: segment.fluid_conditions.pressure_mpa,
            co2_partial_pressure_mpa: segment.fluid_conditions.gas_composition.co2_mole_frac
                * segment.fluid_conditions.pressure_mpa,
            h2s_partial_pressure_mpa: segment.fluid_conditions.gas_composition.h2s_mole_frac
                .map(|frac| frac * segment.fluid_conditions.pressure_mpa),
            ph: segment.fluid_conditions.water_composition.ph,
            flow_velocity_m_s: calculate_flow_velocity(segment),
            pipe_diameter_mm: segment.diameter_mm,
            water_cut: calculate_water_cut(segment),
            chloride_ppm: segment.fluid_conditions.water_composition.chloride_ppm,
            acetic_acid_ppm: None,
            material_name: segment.material_name.clone(),
        };

        let mut solver = CO2CorrosionSolver::new(&input);

        // Run time evolution for 1 year (to allow film to form)
        let n_steps = 100;
        let dt_hours = (365.25 * 24.0) / n_steps as f64;

        for _ in 0..n_steps {
            solver.update_film_thickness(dt_hours);
        }

        let result = solver.calculate_corrosion_rate();

        results.push(SegmentCorrosionResult {
            start_km: segment.start_km,
            end_km: segment.end_km,
            corrosion_rate_mm_yr: result.corrosion_rate_mm_yr,
            film_thickness_um: result.film_thickness_um,
            supersaturation: result.supersaturation,
        });
    }

    Ok(PipelineCorrosionProfile { segments: results })
}

fn calculate_flow_velocity(segment: &crate::models::PipelineSegment) -> f64 {
    let total_volume_rate_m3_s = (
        segment.fluid_conditions.gas_rate_m3_d +
        segment.fluid_conditions.oil_rate_m3_d.unwrap_or(0.0) +
        segment.fluid_conditions.water_rate_m3_d
    ) / 86400.0;

    let area_m2 = std::f64::consts::PI * (segment.diameter_mm / 2000.0).powi(2);
    total_volume_rate_m3_s / area_m2
}

fn calculate_water_cut(segment: &crate::models::PipelineSegment) -> f64 {
    let total_liquid = segment.fluid_conditions.oil_rate_m3_d.unwrap_or(0.0)
        + segment.fluid_conditions.water_rate_m3_d;

    if total_liquid < 1e-6 {
        return 0.0;
    }

    segment.fluid_conditions.water_rate_m3_d / total_liquid
}

#[derive(Debug, Clone, serde::Serialize)]
pub struct PipelineCorrosionProfile {
    pub segments: Vec<SegmentCorrosionResult>,
}

#[derive(Debug, Clone, serde::Serialize)]
pub struct SegmentCorrosionResult {
    pub start_km: f64,
    pub end_km: f64,
    pub corrosion_rate_mm_yr: f64,
    pub film_thickness_um: f64,
    pub supersaturation: f64,
}

#[derive(Debug)]
pub enum SolverError {
    InvalidInput(String),
    NumericalError(String),
}
```

### 3. NACE MR0175 Compliance Checker (Rust)

Environmental cracking (SSC/HIC) susceptibility assessment based on NACE MR0175/ISO 15156 standards.

```rust
// corrosion-core/src/environmental_cracking.rs

use crate::materials::Material;

pub struct EnvironmentalCrackingChecker {
    pub temperature_c: f64,
    pub ph2s_mpa: f64,
    pub ph: f64,
    pub chloride_ppm: f64,
}

impl EnvironmentalCrackingChecker {
    /// Check NACE MR0175/ISO 15156 compliance for SSC
    pub fn check_ssc_compliance(&self, material: &Material) -> SSCCompliance {
        // Determine environmental severity level per ISO 15156-2
        let severity = self.determine_severity_level();

        // Check material hardness limits
        let max_hardness_hrc = match severity {
            SeverityLevel::Level0 => None,  // No H2S
            SeverityLevel::Level1 => Some(26.0),  // pH > 5.5, low H2S
            SeverityLevel::Level2 => Some(24.0),  // pH 4.5-5.5, moderate H2S
            SeverityLevel::Level3 => Some(22.0),  // pH < 4.5 or high H2S
        };

        let material_hardness = material.mechanical_properties
            .get("hardness_hrc")
            .and_then(|v| v.as_f64())
            .unwrap_or(0.0);

        let compliant = match max_hardness_hrc {
            None => true,  // No restrictions
            Some(limit) => material_hardness <= limit,
        };

        let recommendation = if !compliant {
            format!(
                "Material hardness {:.1} HRC exceeds limit {:.1} HRC for SSC resistance. \
                Consider: (1) Stress relief heat treatment, (2) Lower strength grade, \
                (3) CRA upgrade (13Cr, duplex, or 316L SS)",
                material_hardness,
                max_hardness_hrc.unwrap()
            )
        } else {
            "Material complies with NACE MR0175/ISO 15156 for SSC resistance.".to_string()
        };

        SSCCompliance {
            compliant,
            severity_level: severity,
            max_hardness_hrc,
            actual_hardness_hrc: material_hardness,
            recommendation,
        }
    }

    /// Check HIC (Hydrogen Induced Cracking) susceptibility
    pub fn check_hic_susceptibility(&self, material: &Material) -> HICAssessment {
        // HIC primarily affects carbon and low-alloy steels in sour service
        let susceptible = material.material_class == "carbon_steel"
            && self.ph2s_mpa > 0.0003  // 0.3 kPa threshold
            && self.ph < 5.0;

        let risk_level = if !susceptible {
            RiskLevel::Low
        } else if self.ph2s_mpa < 0.003 && self.ph > 4.0 {
            RiskLevel::Medium
        } else {
            RiskLevel::High
        };

        let recommendation = match risk_level {
            RiskLevel::Low => "HIC risk is low for these conditions.".to_string(),
            RiskLevel::Medium => {
                "Moderate HIC risk. Consider: (1) HIC-resistant steel per NACE TM0284, \
                (2) pH stabilization >5.0, (3) H2S scavenger injection.".to_string()
            }
            RiskLevel::High => {
                "High HIC risk. Recommend: (1) CRA clad or solid (13Cr minimum), \
                (2) HIC testing per NACE TM0284 with CLR < 15%, \
                (3) Continuous pH monitoring and control.".to_string()
            }
        };

        HICAssessment {
            susceptible,
            risk_level,
            h2s_partial_pressure_kpa: self.ph2s_mpa * 1000.0,
            ph: self.ph,
            recommendation,
        }
    }

    fn determine_severity_level(&self) -> SeverityLevel {
        if self.ph2s_mpa < 1e-6 {
            return SeverityLevel::Level0;  // No H2S
        }

        let ph2s_kpa = self.ph2s_mpa * 1000.0;

        // ISO 15156-2 severity classification
        if self.ph >= 5.5 && ph2s_kpa < 10.0 {
            SeverityLevel::Level1
        } else if self.ph >= 4.5 && self.ph < 5.5 && ph2s_kpa < 100.0 {
            SeverityLevel::Level2
        } else {
            SeverityLevel::Level3
        }
    }
}

#[derive(Debug, Clone, Copy, serde::Serialize)]
pub enum SeverityLevel {
    Level0,  // Sweet service (no H2S)
    Level1,  // Mild sour
    Level2,  // Moderate sour
    Level3,  // Severe sour
}

#[derive(Debug, Clone, Copy, serde::Serialize)]
pub enum RiskLevel {
    Low,
    Medium,
    High,
}

#[derive(Debug, serde::Serialize)]
pub struct SSCCompliance {
    pub compliant: bool,
    pub severity_level: SeverityLevel,
    pub max_hardness_hrc: Option<f64>,
    pub actual_hardness_hrc: f64,
    pub recommendation: String,
}

#[derive(Debug, serde::Serialize)]
pub struct HICAssessment {
    pub susceptible: bool,
    pub risk_level: RiskLevel,
    pub h2s_partial_pressure_kpa: f64,
    pub ph: f64,
    pub recommendation: String,
}
```

### 4. B31G Anomaly Assessment (Rust)

Metal loss anomaly remaining strength calculation using ASME B31G Modified criteria.

```rust
// corrosion-core/src/anomaly_assessment.rs

pub struct AnomalyAssessment {
    pub pipe_diameter_mm: f64,
    pub wall_thickness_mm: f64,
    pub smys_mpa: f64,  // Specified Minimum Yield Strength
    pub design_pressure_mpa: f64,
    pub defect_length_mm: f64,
    pub defect_depth_mm: f64,
}

impl AnomalyAssessment {
    /// Calculate remaining strength using ASME B31G Modified
    pub fn calculate_b31g_modified(&self) -> B31GResult {
        let d_over_t = self.defect_depth_mm / self.wall_thickness_mm;

        if d_over_t > 0.8 {
            return B31GResult {
                method: "B31G Modified".to_string(),
                safe_pressure_mpa: 0.0,
                failure_pressure_mpa: 0.0,
                safety_factor: 0.0,
                assessment: "FAIL - Depth > 80% wall thickness, immediate repair required".to_string(),
            };
        }

        // Calculate Folias factor M
        let l_over_dt = self.defect_length_mm /
            (self.pipe_diameter_mm * self.wall_thickness_mm).sqrt();

        let m = if l_over_dt < 50.0 {
            (1.0 + 0.6275 * l_over_dt.powi(2) - 0.003375 * l_over_dt.powi(4)).sqrt()
        } else {
            0.032 * l_over_dt.powi(2) + 3.3
        };

        // Failure pressure (Barlow equation with defect)
        let hoop_stress_intact = (self.design_pressure_mpa * self.pipe_diameter_mm)
            / (2.0 * self.wall_thickness_mm);

        let failure_pressure = (2.0 * self.wall_thickness_mm * self.smys_mpa)
            / (self.pipe_diameter_mm * m)
            * (1.0 - d_over_t) / (1.0 - d_over_t / m);

        // Safe operating pressure (with safety factor)
        let safety_factor_required = 1.39;  // ASME B31.4/B31.8 typical
        let safe_pressure = failure_pressure / safety_factor_required;

        let current_sf = failure_pressure / self.design_pressure_mpa;

        let assessment = if current_sf >= safety_factor_required {
            "PASS - Anomaly is acceptable for continued operation".to_string()
        } else if current_sf >= 1.0 {
            format!(
                "MONITOR - Safety factor {:.2} is below required {:.2}. \
                Reduce pressure or schedule repair within 6 months.",
                current_sf, safety_factor_required
            )
        } else {
            "FAIL - Immediate repair or pressure reduction required".to_string()
        };

        B31GResult {
            method: "ASME B31G Modified".to_string(),
            safe_pressure_mpa: safe_pressure,
            failure_pressure_mpa: failure_pressure,
            safety_factor: current_sf,
            assessment,
        }
    }

    /// Calculate remaining strength using DNV-RP-F101
    pub fn calculate_dnv_rp_f101(&self) -> B31GResult {
        let d_over_t = self.defect_depth_mm / self.wall_thickness_mm;

        // DNV uses flow stress (SMYS × 1.15 typically)
        let flow_stress_mpa = self.smys_mpa * 1.15;

        // Calculate beta (correction factor for defect geometry)
        let beta = 0.4;  // Simplified; full DNV uses lookup tables

        // Failure pressure
        let p_burst = (2.0 * self.wall_thickness_mm * flow_stress_mpa)
            / self.pipe_diameter_mm
            * (1.0 - beta * d_over_t) / (1.0 - beta * d_over_t);

        let safety_factor_required = 1.5;  // DNV typical for corrosion defects
        let safe_pressure = p_burst / safety_factor_required;

        let current_sf = p_burst / self.design_pressure_mpa;

        let assessment = if current_sf >= safety_factor_required {
            "PASS per DNV-RP-F101".to_string()
        } else {
            format!("FAIL per DNV-RP-F101 - SF {:.2} < required {:.2}",
                current_sf, safety_factor_required)
        };

        B31GResult {
            method: "DNV-RP-F101".to_string(),
            safe_pressure_mpa: safe_pressure,
            failure_pressure_mpa: p_burst,
            safety_factor: current_sf,
            assessment,
        }
    }
}

#[derive(Debug, serde::Serialize)]
pub struct B31GResult {
    pub method: String,
    pub safe_pressure_mpa: f64,
    pub failure_pressure_mpa: f64,
    pub safety_factor: f64,
    pub assessment: String,
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–4)

**Day 1: Project initialization and Rust backend scaffold**
```bash
cargo init corrosim-api
cd corrosim-api
cargo add axum tokio serde serde_json sqlx uuid chrono tower-http jsonwebtoken bcrypt
cargo add --features postgres,runtime-tokio-rustls,uuid,chrono,json sqlx
cargo add aws-sdk-s3 redis
```
- `src/main.rs` — Axum app with CORS, tracing, graceful shutdown
- `src/config.rs` — Environment-based configuration (DATABASE_URL, JWT_SECRET, S3_BUCKET, REDIS_URL)
- `src/state.rs` — AppState struct (PgPool, Redis, S3Client)
- `src/error.rs` — ApiError enum with IntoResponse
- `Dockerfile` — Multi-stage build (builder + runtime)
- `docker-compose.yml` — PostgreSQL, Redis, MinIO (S3-compatible), Python chemistry service

**Day 2: Database schema and migrations**
- `migrations/001_initial.sql` — All 10 tables: users, organizations, org_members, projects, simulations, simulation_jobs, materials, inspection_records, anomalies, reports, usage_records
- `src/db/mod.rs` — Database pool initialization
- `src/db/models.rs` — All SQLx structs with FromRow derives
- Run `sqlx migrate run` to apply schema
- Seed script for initial materials database (50 common alloys with corrosion data)

**Day 3: Authentication system**
- `src/auth/mod.rs` — JWT token generation and validation middleware
- `src/auth/oauth.rs` — Google and GitHub OAuth 2.0 flow handlers
- `src/api/handlers/auth.rs` — Register, login, OAuth callback, refresh token, me endpoints
- Password hashing with bcrypt (cost factor 12)
- JWT with 24h access token + 30d refresh token
- Auth middleware that extracts Claims from Authorization header

**Day 4: User and project CRUD**
- `src/api/handlers/users.rs` — Get profile, update profile, delete account
- `src/api/handlers/projects.rs` — Create, list, get, update, delete, fork project
- `src/api/handlers/orgs.rs` — Create org, invite member, list members, remove member
- `src/api/router.rs` — All route definitions with auth middleware
- Integration tests: auth flow, project CRUD, authorization checks

### Phase 2 — Solver Core — Corrosion Models (Days 5–12)

**Day 5: Corrosion solver framework**
- `corrosion-core/` — New Rust workspace member (shared between native and WASM)
- `corrosion-core/src/lib.rs` — Module exports
- `corrosion-core/src/models.rs` — Data structures (CO2CorrosionInput, PipelineProfile, CorrosionResult)
- `corrosion-core/src/constants.rs` — Physical constants (Faraday, universal gas constant, etc.)
- Unit tests: basic data structure validation

**Day 6: de Waard-Milliams CO2 corrosion model**
- `corrosion-core/src/co2_corrosion.rs` — CO2CorrosionSolver implementation
- Electrochemical rate calculation with temperature and pCO2 dependence
- pH effect on cathodic reaction rate
- Tests: compare against published de Waard-Milliams reference data (60°C, 1 bar CO2 → 5-10 mm/yr)

**Day 7: Film formation kinetics**
- `corrosion-core/src/film_kinetics.rs` — FeCO3 precipitation and dissolution model
- Supersaturation calculation (Ksp temperature dependence)
- Film growth ODE integration (Euler method)
- Film protection factor vs. thickness curve
- Tests: verify film thickness evolution reaches steady state, protection factor reduces CR by 5-10x

**Day 8: Mass transfer and flow regime coupling**
- `corrosion-core/src/mass_transfer.rs` — Sherwood number correlations for turbulent/laminar flow
- `corrosion-core/src/flow_regime.rs` — Taitel-Dukler flow regime map (stratified/slug/annular)
- Mass transfer limited corrosion rate calculation
- Tests: verify Re > 10,000 gives turbulent Sh, Re < 2,300 gives laminar Sh

**Day 9: Pipeline profile solver**
- `corrosion-core/src/pipeline.rs` — solve_pipeline_profile() function
- Segment-by-segment calculation with pressure/temperature/flow regime variation
- Film thickness state propagation between segments
- Tests: 10 km pipeline with 5 segments, verify smooth CR profile

**Day 10: H2S corrosion model**
- `corrosion-core/src/h2s_corrosion.rs` — H2S contribution to corrosion rate
- Additive effect with CO2 corrosion
- FeS film formation (different from FeCO3)
- Tests: verify H2S presence increases CR by 2-5x at same pH

**Day 11: Environmental cracking assessment**
- `corrosion-core/src/environmental_cracking.rs` — EnvironmentalCrackingChecker implementation
- NACE MR0175 severity level classification
- SSC hardness limits per severity level
- HIC risk assessment based on pH/H2S
- Tests: verify Level 3 severity requires HRC < 22, Level 1 allows HRC < 26

**Day 12: Anomaly assessment (B31G, DNV)**
- `corrosion-core/src/anomaly_assessment.rs` — AnomalyAssessment implementation
- B31G Modified: Folias factor, failure pressure, safety factor
- DNV-RP-F101: flow stress, beta factor, burst pressure
- Tests: 50% depth, L/D=5 defect → SF ≈ 1.5-2.0 (verify against manual calculation)

### Phase 3 — WASM Build + Water Chemistry Service (Days 13–18)

**Day 13: WASM solver compilation**
- `corrosion-wasm/` — New workspace member for WASM target
- `corrosion-wasm/Cargo.toml` — wasm-bindgen, serde-wasm-bindgen dependencies
- `corrosion-wasm/src/lib.rs` — WASM entry points: `solve_co2_corrosion()`, `solve_pipeline_profile()`, `check_ssc_compliance()`, `assess_anomaly()`
- `wasm-pack build --target web --release` pipeline
- `wasm-opt -Oz` for size optimization (target <1MB gzipped)
- JavaScript wrapper: `CorrosionSolver` class that loads WASM and provides async API

**Day 14: Water chemistry service (Python FastAPI)**
- `chemistry-service/` — Python FastAPI microservice
- `requirements.txt` — scipy, numpy, pydantic
- `chemistry-service/main.py` — FastAPI app with /calculate_ph endpoint
- `chemistry-service/pitzer.py` — Pitzer ion interaction model for ionic strength effects
- Input: gas composition (CO2, H2S mole fractions), water composition (salinity, bicarbonate)
- Output: pH, CO2 partial pressure, H2S partial pressure, scaling tendency
- Docker container, exposed on port 8001

**Day 15: Integrate water chemistry with Rust backend**
- `src/services/chemistry.rs` — HTTP client calling Python chemistry service
- API handler wrapper: POST /api/chemistry/calculate → calls Python service
- Cache chemistry results in Redis (1 hour TTL) for repeat calculations
- Tests: verify pH calculation matches expected values (e.g., 10 bar CO2, 10,000 ppm Cl → pH ≈ 4.5-5.0)

**Day 16: Materials database seeding**
- `scripts/seed_materials.rs` — Seed script for materials table
- 50 materials: A106 Gr B, API 5L X65, 13Cr (S41000), 22Cr duplex (S31803), 25Cr super duplex (S32750), 316L SS (S31603), Inconel 625, titanium Gr 2
- Corrosion data: CO2 corrosion rates at 20°C, 60°C, 100°C; H2S limits; chloride SCC threshold
- JSONB structure: `{ "co2_rates": [{"temp_c": 60, "pco2_bar": 1, "rate_mm_yr": 0.5}], "nace_limits": {...} }`
- Run seed script: `cargo run --bin seed_materials`

**Day 17: Materials API endpoints**
- `src/api/handlers/materials.rs` — List materials, search materials (by class, UNS, environment), get material detail
- Parametric search: filter by max H2S, max temperature, min PREN
- Full-text search on name using pg_trgm
- Pagination with limit/offset
- Tests: search "316L" → returns S31603, search "carbon_steel" → returns 20+ results

**Day 18: Materials comparison engine**
- `src/api/handlers/materials_comparison.rs` — Compare materials endpoint
- Input: environment (T, pCO2, pH2S, Cl), list of material IDs
- Output: corrosion rate, environmental cracking risk, cost per m² (material + corrosion allowance)
- Ranking by total cost of ownership over design life
- Tests: compare A106 Gr B vs. 13Cr vs. 316L for 60°C, 5 bar CO2 → 13Cr has lowest lifecycle cost

### Phase 4 — API + Job Orchestration (Days 19–24)

**Day 19: Simulation API endpoints**
- `src/api/handlers/simulation.rs` — Create simulation, get simulation, list simulations, cancel simulation
- Pipeline profile validation (segments must be contiguous, positive lengths)
- Pipeline length extraction and WASM/server routing logic
- Plan-based limits enforcement (free: 10 km, pro: unlimited, advanced: unlimited + priority queue)

**Day 20: Server-side simulation worker**
- `src/workers/simulation_worker.rs` — Redis job consumer, runs native solver
- `src/workers/mod.rs` — Worker pool management (configurable concurrency)
- Progress streaming: worker publishes progress → Redis pub/sub → WebSocket to client
- Error handling: invalid input, numerical errors, timeouts
- S3 result upload with presigned URL generation for client download

**Day 21: WebSocket for live simulation progress**
- `src/api/ws/mod.rs` — Axum WebSocket upgrade handler
- `src/api/ws/simulation_progress.rs` — Subscribe to simulation progress channel
- Client receives: `{ progress_pct, current_segment_km, eta_seconds }`
- Connection management: heartbeat ping/pong, automatic cleanup on disconnect
- Frontend: `src/hooks/useSimulationProgress.ts` — React hook for WebSocket subscription

**Day 22: Inspection data import**
- `src/api/handlers/inspection.rs` — Upload inspection data (CSV), list inspection records, get inspection record
- CSV parser for ILI (in-line inspection) data: distance, wall thickness, depth, clock position
- Anomaly auto-detection: depth > 20% WT → create anomaly record
- Tests: upload 100-row CSV → 15 anomalies detected

**Day 23: Anomaly assessment API**
- `src/api/handlers/anomaly.rs` — Create anomaly, assess anomaly (B31G or DNV), list anomalies, update anomaly status
- Assessment endpoint: POST /api/projects/{id}/anomalies/{anomaly_id}/assess
- Input: assessment method (b31g_modified | dnv_rp_f101)
- Output: safe pressure, failure pressure, safety factor, assessment (PASS/MONITOR/FAIL)
- Tests: assess 50% depth defect → safety factor ≈ 1.8, PASS

**Day 24: Corrosion allowance and remaining life**
- `src/api/handlers/life_assessment.rs` — Calculate corrosion allowance, calculate remaining life
- Corrosion allowance: CA = CR × design_life (e.g., 2 mm/yr × 20 yr = 40 mm)
- Remaining life: RL = (current_thickness - min_thickness) / CR
- Integration with inspection data: use measured thickness from ILI
- Tests: 10 mm WT, 6.4 mm min, 2 mm/yr → RL = 1.8 years

### Phase 5 — Frontend + Visualization (Days 25–29)

**Day 25: Frontend scaffold and pipeline editor foundation**
```bash
npm create vite@latest frontend -- --template react-ts
cd frontend
npm i zustand @tanstack/react-query axios d3-scale
```
- `src/App.tsx` — Router, auth context, layout
- `src/stores/projectStore.ts` — Zustand store for project state
- `src/stores/pipelineStore.ts` — Pipeline editor state (segments, fluids, materials)
- `src/components/PipelineEditor/Canvas.tsx` — SVG canvas for pipeline profile
- `src/components/PipelineEditor/SegmentEditor.tsx` — Edit segment properties (length, diameter, material, fluid)
- Drag-and-drop segment ordering, add/remove segments

**Day 26: Pipeline profile visualizer**
- `src/components/PipelineVisualizer/ProfileView.tsx` — SVG rendering of elevation profile vs. distance
- `src/components/PipelineVisualizer/HeatmapOverlay.tsx` — Canvas overlay with corrosion rate heatmap (green → yellow → red gradient)
- D3 scales for distance (X-axis) and elevation/corrosion rate (Y-axis)
- Pan/zoom with mouse wheel and drag
- Tooltip on hover: show distance, elevation, corrosion rate, film thickness

**Day 27: Materials selector and comparison**
- `src/components/MaterialsSelector/MaterialSearch.tsx` — Search materials by name, class, UNS
- `src/components/MaterialsSelector/MaterialCard.tsx` — Display material properties, corrosion rates, NACE limits
- `src/components/MaterialsComparison/ComparisonTable.tsx` — Side-by-side comparison of selected materials
- Highlight best material (lowest lifecycle cost, lowest environmental cracking risk)

**Day 28: NACE compliance and anomaly assessment UI**
- `src/components/Compliance/SSCChecker.tsx` — Input environment, select material, display SSC compliance result
- `src/components/Compliance/HICChecker.tsx` — HIC risk assessment with recommendations
- `src/components/Anomaly/AnomalyForm.tsx` — Input defect dimensions (length, depth, width, clock position)
- `src/components/Anomaly/AssessmentResult.tsx` — Display B31G/DNV results (safe pressure, safety factor, PASS/MONITOR/FAIL)
- Color-coded status: green (PASS), yellow (MONITOR), red (FAIL)

**Day 29: Results dashboard and reporting**
- `src/components/Dashboard/ProjectsList.tsx` — List all projects with thumbnails, last updated
- `src/components/Dashboard/RecentSimulations.tsx` — Recent simulation results with max corrosion rate
- `src/components/Results/CorrosionReport.tsx` — Detailed simulation results: CR profile chart, max CR location, material recommendations
- Export to PDF: generate PDF report with pipeline profile, corrosion rate table, anomaly assessment summary
- CSV export: corrosion rate profile data for external analysis

### Phase 6 — Billing + Plan Enforcement (Days 30–33)

**Day 30: Stripe integration**
- `src/api/handlers/billing.rs` — Create checkout session, create customer portal session, get subscription status
- `src/api/handlers/webhooks/stripe.rs` — Handle subscription events: `checkout.session.completed`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.payment_failed`
- Plan mapping:
  - Free: 10 km pipelines, CO2 corrosion only, 3 projects
  - Pro ($199/mo): Unlimited length, H2S corrosion, environmental cracking, materials comparison, unlimited projects
  - Advanced ($449/user/mo): Team collaboration, inspection data import, anomaly assessment, API access, priority queue

**Day 31: Usage tracking and plan limits**
- `src/middleware/plan_limits.rs` — Middleware checking plan limits before simulation execution
- `src/services/usage.rs` — Track API calls, simulation count per billing period
- Usage record insertion after each simulation completes
- Usage dashboard: `src/api/handlers/usage.rs` — Current period usage, historical usage
- Approaching-limit warnings at 80% and 100% of quota

**Day 32: Billing UI**
- `frontend/src/pages/Billing.tsx` — Plan comparison table, current plan display, usage meter
- `frontend/src/components/billing/PlanCard.tsx` — Plan feature comparison cards
- `frontend/src/components/billing/UsageMeter.tsx` — Simulation count usage bar with threshold indicators
- Upgrade/downgrade flow via Stripe Customer Portal
- Upgrade prompt modals when hitting plan limits (e.g., "Pipeline length 150 km exceeds Free limit. Upgrade to Pro for unlimited.")

**Day 33: Feature gating**
- Gate H2S corrosion and environmental cracking behind Pro plan
- Gate inspection data import and anomaly assessment behind Advanced plan
- Gate API access behind Advanced plan
- Show locked feature indicators with upgrade CTAs in UI
- Admin endpoint: override plan for specific users (beta testers, partners)

### Phase 7 — Solver Validation + Testing (Days 34–38)

**Day 34: Solver validation — CO2 corrosion benchmarks**
- Benchmark 1: 60°C, 1 bar CO2, pH 5.0 → verify CR ≈ 5-10 mm/yr (de Waard-Milliams reference)
- Benchmark 2: 80°C, 5 bar CO2, pH 4.5 → verify CR ≈ 20-30 mm/yr
- Benchmark 3: Film formation equilibrium → verify film thickness reaches 50-100 μm after 1 year, protection factor 5-10x
- Automated test suite: `corrosion-core/tests/benchmarks.rs` with assertions within 20% tolerance

**Day 35: Solver validation — environmental cracking**
- Benchmark 4: 0.5 kPa H2S, pH 5.0, 22 HRC → verify SSC Level 2, compliant
- Benchmark 5: 10 kPa H2S, pH 4.0, 26 HRC → verify SSC Level 3, non-compliant
- Benchmark 6: 5 kPa H2S, pH 4.5, carbon steel → verify HIC High Risk
- Compare results against ISO 15156 standard tables

**Day 36: Solver validation — anomaly assessment**
- Benchmark 7: 50% depth, L/√(Dt) = 5 → verify B31G SF ≈ 1.8-2.0
- Benchmark 8: 30% depth, 100 mm length → verify DNV SF ≈ 2.5-3.0
- Benchmark 9: 80% depth → verify immediate FAIL, SF < 1.0
- Compare results against published case studies from ASME/DNV

**Day 37: Integration testing**
- End-to-end test: create project → define pipeline → run simulation → view results → assess anomaly
- API integration tests: auth → project CRUD → simulation → results
- WASM solver test: load in headless browser (Playwright), run simulation, verify results match native solver
- WebSocket test: connect → subscribe → receive progress → receive completion
- Concurrent simulation test: submit 5 simultaneous server-side jobs

**Day 38: Performance testing and optimization**
- WASM solver benchmarks: measure time for 10 km, 50 km, 100 km pipelines in Chrome/Firefox/Safari
- Target: 100 km pipeline (500 segments) < 3 seconds in WASM
- Server solver benchmarks: measure throughput (simulations/minute) with various pipeline lengths
- Frontend rendering: measure Canvas heatmap FPS with 1000, 5000, 10000 data points
- Target: 60 FPS pan/zoom with 5000 data points
- Memory profiling: ensure WASM solver doesn't leak, frontend GC behavior is healthy
- Load testing: 20 concurrent users running simulations via k6

### Phase 8 — Deployment + Polish + Launch (Days 39–42)

**Day 39: Docker and Kubernetes configuration**
- `Dockerfile` — Multi-stage Rust build (cargo-chef for layer caching)
- `docker-compose.yml` — Full local dev stack (API, PostgreSQL, Redis, MinIO, chemistry-service, frontend)
- `k8s/` — Kubernetes manifests:
  - `api-deployment.yaml` — API server (3 replicas, HPA)
  - `worker-deployment.yaml` — Simulation workers (auto-scaling based on queue depth)
  - `postgres-statefulset.yaml` — PostgreSQL with PVC
  - `redis-deployment.yaml` — Redis for job queue and pub/sub
  - `chemistry-deployment.yaml` — Python chemistry service (2 replicas)
  - `ingress.yaml` — NGINX ingress with TLS
- Health check endpoints: `/health/live`, `/health/ready`

**Day 40: CDN and WASM delivery**
- CloudFront distribution for:
  - Frontend static assets (HTML, JS, CSS) — cached at edge
  - WASM solver bundle (~1MB) — cached at edge with long TTL and versioned URLs
  - Report PDFs from S3 — cached with 1-hour TTL
- WASM preloading: start downloading WASM bundle on page load before user needs it
- Service worker for offline WASM caching (progressive enhancement)

**Day 41: Monitoring, logging, and polish**
- Prometheus metrics: simulation duration histogram, solver success rate, API latency percentiles
- Grafana dashboards: system health, simulation throughput, user activity
- Sentry integration: frontend and backend error tracking with source maps
- Structured logging (tracing + JSON) for all API requests and solver events
- UI polish: loading states, error messages, empty states, responsive layout
- Accessibility: keyboard navigation in pipeline editor, ARIA labels on interactive elements

**Day 42: Launch preparation and final testing**
- End-to-end smoke test in production environment
- Database backup and restore verification
- Rate limiting: 100 req/min per user, 5 simulations/min per user
- Security audit: JWT validation, SQL injection (SQLx prevents this), CORS configuration, CSP headers
- Landing page with product overview, pricing, and signup
- Documentation: getting started guide, pipeline modeling tutorial, NACE compliance guide
- Deploy to production, enable monitoring alerts, announce launch

---

## Critical Files

```
corrosim/
├── corrosion-core/                        # Shared solver library (native + WASM)
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── models.rs                      # Data structures (CO2CorrosionInput, PipelineProfile, etc.)
│   │   ├── constants.rs                   # Physical constants (Faraday, R, etc.)
│   │   ├── co2_corrosion.rs               # de Waard-Milliams CO2 corrosion model
│   │   ├── h2s_corrosion.rs               # H2S corrosion contribution
│   │   ├── film_kinetics.rs               # FeCO3 film precipitation/dissolution
│   │   ├── mass_transfer.rs               # Sherwood number correlations
│   │   ├── flow_regime.rs                 # Taitel-Dukler flow regime map
│   │   ├── pipeline.rs                    # Pipeline profile solver
│   │   ├── environmental_cracking.rs      # NACE MR0175 SSC/HIC assessment
│   │   ├── anomaly_assessment.rs          # B31G Modified and DNV-RP-F101
│   │   └── materials.rs                   # Material properties helpers
│   └── tests/
│       ├── benchmarks.rs                  # Solver validation benchmarks
│       └── integration.rs                 # Integration tests
│
├── corrosion-wasm/                        # WASM compilation target
│   ├── Cargo.toml
│   └── src/
│       └── lib.rs                         # WASM entry points (wasm-bindgen)
│
├── corrosim-api/                          # Rust API server
│   ├── Cargo.toml
│   ├── src/
│   │   ├── main.rs                        # Axum app setup, router, startup
│   │   ├── config.rs                      # Environment config
│   │   ├── state.rs                       # AppState (PgPool, Redis, S3)
│   │   ├── error.rs                       # ApiError types
│   │   ├── auth/
│   │   │   ├── mod.rs                     # JWT middleware, Claims
│   │   │   └── oauth.rs                   # Google/GitHub OAuth handlers
│   │   ├── api/
│   │   │   ├── router.rs                  # Route definitions
│   │   │   ├── handlers/
│   │   │   │   ├── auth.rs                # Register, login, OAuth
│   │   │   │   ├── users.rs               # Profile CRUD
│   │   │   │   ├── projects.rs            # Project CRUD, fork, share
│   │   │   │   ├── simulation.rs          # Create/get/cancel simulation
│   │   │   │   ├── materials.rs           # Material search/get/compare
│   │   │   │   ├── materials_comparison.rs # Materials comparison engine
│   │   │   │   ├── inspection.rs          # Inspection data import
│   │   │   │   ├── anomaly.rs             # Anomaly assessment
│   │   │   │   ├── life_assessment.rs     # Corrosion allowance, remaining life
│   │   │   │   ├── billing.rs             # Stripe checkout/portal
│   │   │   │   ├── usage.rs               # Usage tracking
│   │   │   │   └── webhooks/
│   │   │   │       └── stripe.rs          # Stripe webhook handler
│   │   │   └── ws/
│   │   │       ├── mod.rs                 # WebSocket upgrade handler
│   │   │       └── simulation_progress.rs # Live progress streaming
│   │   ├── middleware/
│   │   │   └── plan_limits.rs             # Plan enforcement middleware
│   │   ├── services/
│   │   │   ├── usage.rs                   # Usage tracking service
│   │   │   ├── s3.rs                      # S3 client helpers
│   │   │   └── chemistry.rs               # HTTP client for chemistry service
│   │   └── workers/
│   │       ├── mod.rs                     # Worker pool management
│   │       └── simulation_worker.rs       # Server-side simulation execution
│   ├── migrations/
│   │   └── 001_initial.sql                # Full database schema
│   └── tests/
│       ├── api_integration.rs             # API integration tests
│       └── simulation_e2e.rs              # End-to-end simulation tests
│
├── chemistry-service/                     # Python FastAPI water chemistry
│   ├── requirements.txt
│   ├── main.py                            # FastAPI app
│   ├── pitzer.py                          # Pitzer ion interaction model
│   ├── equilibrium.py                     # Multi-species equilibrium solver
│   └── Dockerfile
│
├── frontend/                              # React frontend
│   ├── package.json
│   ├── vite.config.ts
│   ├── src/
│   │   ├── App.tsx                        # Router, providers, layout
│   │   ├── main.tsx                       # Entry point
│   │   ├── stores/
│   │   │   ├── authStore.ts               # Auth state (JWT, user)
│   │   │   ├── projectStore.ts            # Project state
│   │   │   └── pipelineStore.ts           # Pipeline editor state
│   │   ├── hooks/
│   │   │   ├── useSimulationProgress.ts   # WebSocket hook for live progress
│   │   │   └── useWasmSolver.ts           # WASM solver hook (load + run)
│   │   ├── lib/
│   │   │   ├── api.ts                     # Axios API client
│   │   │   └── wasmLoader.ts              # WASM bundle loading
│   │   ├── pages/
│   │   │   ├── Dashboard.tsx              # Project list
│   │   │   ├── Editor.tsx                 # Pipeline editor
│   │   │   ├── Materials.tsx              # Materials database browser
│   │   │   ├── Billing.tsx                # Plan management
│   │   │   ├── Login.tsx                  # Auth pages
│   │   │   └── Register.tsx
│   │   ├── components/
│   │   │   ├── PipelineEditor/
│   │   │   │   ├── Canvas.tsx             # SVG canvas for pipeline profile
│   │   │   │   ├── SegmentEditor.tsx      # Edit segment properties
│   │   │   │   └── FluidEditor.tsx        # Edit fluid composition
│   │   │   ├── PipelineVisualizer/
│   │   │   │   ├── ProfileView.tsx        # SVG elevation profile
│   │   │   │   ├── HeatmapOverlay.tsx     # Canvas corrosion rate heatmap
│   │   │   │   └── Tooltip.tsx            # Hover tooltip
│   │   │   ├── MaterialsSelector/
│   │   │   │   ├── MaterialSearch.tsx     # Search materials
│   │   │   │   └── MaterialCard.tsx       # Material properties display
│   │   │   ├── MaterialsComparison/
│   │   │   │   └── ComparisonTable.tsx    # Side-by-side comparison
│   │   │   ├── Compliance/
│   │   │   │   ├── SSCChecker.tsx         # SSC compliance checker
│   │   │   │   └── HICChecker.tsx         # HIC risk assessment
│   │   │   ├── Anomaly/
│   │   │   │   ├── AnomalyForm.tsx        # Defect input form
│   │   │   │   └── AssessmentResult.tsx   # B31G/DNV results
│   │   │   ├── Results/
│   │   │   │   └── CorrosionReport.tsx    # Simulation results display
│   │   │   ├── billing/
│   │   │   │   ├── PlanCard.tsx
│   │   │   │   └── UsageMeter.tsx
│   │   │   └── common/
│   │   │       └── SimulationToolbar.tsx   # Run/stop/export controls
│   │   └── utils/
│   │       └── formatters.ts              # Number formatting (engineering notation)
│   └── public/
│       └── wasm/                          # WASM solver bundle (loaded at runtime)
│
├── k8s/                                   # Kubernetes manifests
│   ├── api-deployment.yaml
│   ├── worker-deployment.yaml
│   ├── postgres-statefulset.yaml
│   ├── redis-deployment.yaml
│   ├── chemistry-deployment.yaml
│   └── ingress.yaml
│
├── scripts/
│   └── seed_materials.rs                 # Materials database seed script
│
├── docker-compose.yml                     # Local development stack
├── Cargo.toml                             # Workspace root
└── .github/
    └── workflows/
        ├── ci.yml                         # Test + lint on PR
        ├── wasm-build.yml                 # Build + deploy WASM bundle
        └── deploy.yml                     # Build Docker images + deploy to K8s
```

---

## Solver Validation Suite

### Benchmark 1: CO2 Corrosion — Sweet Service (60°C, 1 bar CO2)

**Conditions:** Temperature = 60°C, pCO2 = 1 bar (0.1 MPa), pH = 5.0, velocity = 1 m/s, no film

**Expected:** Bare steel corrosion rate ≈ **5-10 mm/yr** (de Waard-Milliams reference data)

**Tolerance:** < 20% (model calibration uncertainty)

### Benchmark 2: CO2 Corrosion — High Temperature (80°C, 5 bar CO2)

**Conditions:** Temperature = 80°C, pCO2 = 5 bar (0.5 MPa), pH = 4.5, velocity = 2 m/s, no film

**Expected:** Bare steel corrosion rate ≈ **20-30 mm/yr** (exponential temperature dependence)

**Tolerance:** < 20%

### Benchmark 3: Film Formation — FeCO3 Protection

**Conditions:** Temperature = 60°C, pCO2 = 1 bar, pH = 6.0 (high pH → high supersaturation), 1 year exposure

**Expected:** Film thickness ≈ **50-100 μm**, protection factor ≈ **5-10x** (reduces CR to 1-2 mm/yr)

**Tolerance:** Film thickness < 30%, protection factor < 30%

### Benchmark 4: SSC Compliance — Level 2 Severity (Moderate Sour)

**Conditions:** pH2S = 0.5 kPa, pH = 5.0, temperature = 40°C

**Expected:** Severity Level = **Level 2**, max hardness = **24 HRC**

Material with 22 HRC → **Compliant**

**Tolerance:** Exact classification per ISO 15156-2

### Benchmark 5: SSC Compliance — Level 3 Severity (Severe Sour)

**Conditions:** pH2S = 10 kPa, pH = 4.0, temperature = 60°C

**Expected:** Severity Level = **Level 3**, max hardness = **22 HRC**

Material with 26 HRC → **Non-compliant**

**Tolerance:** Exact classification per ISO 15156-2

### Benchmark 6: HIC Risk — High Risk

**Conditions:** pH2S = 5 kPa, pH = 4.5, carbon steel, no HIC testing

**Expected:** Risk Level = **High**, recommendation = "CRA clad or HIC-tested steel required"

**Tolerance:** Exact risk level classification

### Benchmark 7: B31G Modified — 50% Depth Defect

**Conditions:** Pipe OD = 508 mm, WT = 12.7 mm, defect length = 100 mm, defect depth = 6.35 mm (50%), SMYS = 359 MPa (X52), design pressure = 10 MPa

**Expected:** L/√(Dt) ≈ 5.0, Folias factor M ≈ 1.8-2.0, safety factor ≈ **1.8-2.0**, assessment = **PASS**

**Tolerance:** SF < 10%

### Benchmark 8: DNV-RP-F101 — 30% Depth Defect

**Conditions:** Pipe OD = 508 mm, WT = 12.7 mm, defect length = 100 mm, defect depth = 3.8 mm (30%), SMYS = 448 MPa (X65), design pressure = 12 MPa

**Expected:** Safety factor ≈ **2.5-3.0**, assessment = **PASS**

**Tolerance:** SF < 10%

### Benchmark 9: B31G Modified — Critical Defect (80% Depth)

**Conditions:** Pipe OD = 508 mm, WT = 12.7 mm, defect length = 100 mm, defect depth = 10.2 mm (80%), SMYS = 359 MPa (X52), design pressure = 10 MPa

**Expected:** Safety factor < **1.0**, assessment = **FAIL** — immediate repair required

**Tolerance:** Exact failure classification

---

## Verification

### Integration Testing Checklist

1. **Auth flow** — Register → login → JWT issued → authorized API calls → token refresh → logout
2. **Project CRUD** — Create → update pipeline → save → reload → verify pipeline state preserved
3. **Pipeline editing** — Add segment → set fluid composition → set material → verify total length calculation
4. **WASM simulation** — Small pipeline (5 km, 3 segments) → WASM solver → results returned → profile displays
5. **Server simulation** — Large pipeline (150 km, 50 segments) → job queued → worker picks up → WebSocket progress → results in S3
6. **Pipeline visualizer** — Load 100-segment result → pan/zoom → hover tooltip → verify CR values
7. **Materials search** — Search "316L" → results returned → select material → use in pipeline
8. **Materials comparison** — Compare A106 Gr B vs. 13Cr vs. 316L → lifecycle cost ranking displayed
9. **SSC compliance** — Input sour environment → select carbon steel → non-compliant warning → recommend CRA upgrade
10. **Anomaly assessment** — Input 50% depth defect → assess with B31G → safety factor displayed → PASS/MONITOR/FAIL
11. **Plan limits** — Free user → 15 km pipeline → blocked with upgrade prompt
12. **Billing** — Subscribe to Pro → Stripe checkout → webhook → plan updated → limits lifted
13. **Concurrent users** — 5 users simultaneously running server simulations → all complete correctly
14. **Inspection import** — Upload CSV with 100 rows → anomalies auto-detected → assessment recommended
15. **Error handling** — Invalid pipeline (negative length) → meaningful error message → no crash

### SQL Verification Queries

```sql
-- 1. Simulation throughput and success rate
SELECT
    DATE_TRUNC('day', created_at) AS day,
    COUNT(*) AS total_simulations,
    SUM(CASE WHEN status = 'completed' THEN 1 ELSE 0 END) AS completed,
    SUM(CASE WHEN status = 'failed' THEN 1 ELSE 0 END) AS failed,
    AVG(wall_time_ms) / 1000.0 AS avg_time_seconds
FROM simulations
WHERE created_at > NOW() - INTERVAL '30 days'
GROUP BY day
ORDER BY day DESC;

-- 2. Materials usage distribution
SELECT
    material_class,
    COUNT(*) AS usage_count
FROM (
    SELECT DISTINCT
        (pipeline_data->'segments'->0->>'material_name')::TEXT AS material_name
    FROM projects
) AS materials
JOIN materials m ON m.name = materials.material_name
GROUP BY material_class
ORDER BY usage_count DESC;

-- 3. Anomaly assessment distribution
SELECT
    assessment_method,
    status,
    COUNT(*) AS count,
    AVG(safety_factor) AS avg_safety_factor
FROM anomalies
GROUP BY assessment_method, status
ORDER BY assessment_method, status;

-- 4. Plan distribution
SELECT
    plan,
    COUNT(*) AS user_count,
    SUM(CASE WHEN created_at > NOW() - INTERVAL '30 days' THEN 1 ELSE 0 END) AS new_users_30d
FROM users
GROUP BY plan
ORDER BY user_count DESC;

-- 5. Top corrosion rate locations (from simulations)
SELECT
    p.name AS project_name,
    s.results_summary->>'max_corrosion_rate_mm_yr' AS max_cr,
    s.results_summary->>'max_cr_location_km' AS location,
    s.created_at
FROM simulations s
JOIN projects p ON s.project_id = p.id
WHERE s.status = 'completed'
    AND s.results_summary IS NOT NULL
ORDER BY (s.results_summary->>'max_corrosion_rate_mm_yr')::FLOAT DESC
LIMIT 10;
```

---

## Deployment Architecture

### Production Infrastructure (AWS)

```
┌─────────────────────────────────────────────────────────────────┐
│                        CloudFront CDN                           │
│  - Frontend static assets (HTML, JS, CSS)                       │
│  - WASM solver bundle (~1MB gzipped, versioned)                 │
│  - Report PDFs from S3                                          │
└─────────────────────┬───────────────────────────────────────────┘
                      │
        ┌─────────────┴──────────────┐
        │                            │
┌───────▼────────┐          ┌────────▼────────┐
│  Frontend SPA  │          │  API Gateway    │
│  (S3 + CF)     │          │  (ALB)          │
└────────────────┘          └────────┬────────┘
                                     │
                      ┌──────────────┴──────────────┐
                      │                             │
              ┌───────▼────────┐          ┌────────▼────────┐
              │  API Servers   │          │  Workers        │
              │  (ECS Fargate) │          │  (ECS Fargate)  │
              │  3 replicas    │          │  Auto-scaling   │
              │  HPA on CPU    │          │  2-10 replicas  │
              └───────┬────────┘          └────────┬────────┘
                      │                            │
        ┌─────────────┼────────────────────────────┼──────────┐
        │             │                            │          │
┌───────▼──────┐ ┌────▼─────┐ ┌─────────┐  ┌──────▼──────┐  │
│ PostgreSQL   │ │  Redis   │ │   S3    │  │  Chemistry  │  │
│ RDS (Multi-AZ│ │ ElastiC. │ │         │  │  Service    │  │
│ r6g.2xlarge) │ │ r6g.large│ │ Results │  │  (Fargate)  │  │
└──────────────┘ └──────────┘ │ Reports │  │  2 replicas │  │
                               └─────────┘  └─────────────┘  │
                                                              │
┌─────────────────────────────────────────────────────────────┘
│  Monitoring & Observability
│  - Prometheus (metrics scraping)
│  - Grafana (dashboards)
│  - Sentry (error tracking)
│  - CloudWatch Logs (structured logging)
└─────────────────────────────────────────────────────────────┘
```

### Scaling Strategy

**API Servers:**
- Horizontal auto-scaling: 3-10 replicas based on CPU > 70%
- Each replica handles ~50 req/s
- Target: 500 req/s peak capacity

**Simulation Workers:**
- Horizontal auto-scaling: 2-10 replicas based on Redis queue depth
- Each worker runs 1 simulation at a time (CPU-bound)
- Target: 10 concurrent long-running simulations

**Database:**
- PostgreSQL RDS Multi-AZ for high availability
- Read replicas (2) for materials search queries
- Connection pooling: max 100 connections per API replica

**Redis:**
- ElastiCache for job queue and WebSocket pub/sub
- Single node (r6g.large) with automatic failover
- Persistence: AOF (append-only file) for job queue durability

**S3:**
- Simulation results stored in S3 Standard
- Lifecycle policy: transition to S3 IA after 90 days, Glacier after 1 year
- CloudFront for edge caching of frequently accessed results

---

## Post-MVP Roadmap

### v1.1 — 3D Cathodic Protection Modeling (Months 3-5)

**Problem:** Offshore structures, subsea pipelines, and reinforced concrete require cathodic protection (CP) system design to prevent corrosion. Current tools (BEASY, COMSOL) cost $20K+ per seat and are complex to use. Market: 5,000+ CP engineers worldwide, offshore wind farms need CP design for 1000+ turbine foundations.

**Solution:**
- 3D boundary element method (BEM) solver for CP potential and current distribution
- Sacrificial anode CP: aluminum/zinc/magnesium anode sizing, consumption rate, life prediction
- Impressed current CP: rectifier sizing, anode ground bed design, current distribution optimization
- Geometry import: STL/STEP file import for complex structures (jacket, mooring, concrete foundation)
- Anode placement optimization: genetic algorithm to minimize anode count while meeting -850 mV CSE protection criterion
- Time-dependent CP: anode consumption over 25-year design life, coating degradation, seasonal soil resistivity changes

**Technical:**
- Rust BEM solver using `faer` for dense linear algebra (boundary integral equations)
- WebGL/Three.js for 3D structure visualization with potential colormap overlay
- Python optimization service (PyGAD genetic algorithm) for anode placement

**Pricing:** Add to Advanced plan ($449/user/mo), or CP-only plan ($299/mo)

### v1.2 — Risk-Based Inspection (RBI) per API 581 (Months 4-6)

**Problem:** Asset owners spend $5-10B/year on inspection (ILI, UT, RT) but struggle to prioritize which equipment to inspect first. API 581 RBI provides a framework for probability of failure (PoF) × consequence (CoF) ranking, but manual implementation takes weeks per site.

**Solution:**
- Automated PoF calculation: integrate corrosion rate prediction with damage factor (DF), inspection effectiveness, and remaining life
- CoF calculation: consequence categories (safety, environmental, economic) based on fluid inventory, toxicity, flammability
- Risk matrix: 5×5 matrix (PoF vs. CoF) with color-coded ranking (low/medium/medium-high/high risk)
- Inspection planning: recommend inspection interval based on target risk level and corrosion rate uncertainty
- Fleet-wide dashboard: aggregate risk across 100+ assets, prioritize inspection budget allocation

**Technical:**
- API 581 damage factor calculations in Rust
- Consequence modeling: dispersion models (Gaussian plume for toxic release), fire radiation (API 521)
- PostgreSQL: store PoF/CoF for trending over time
- React dashboard: risk matrix heatmap, Pareto chart (top 20 high-risk items)

**Pricing:** Add to Advanced plan ($449/user/mo)

### v1.3 — Hydrogen Infrastructure Support (Months 5-7)

**Problem:** Hydrogen economy (blue/green H2 production, pipelines, storage) introduces new corrosion challenges: hydrogen embrittlement (HE), hydrogen-induced cracking (HIC), hydrogen permeation through steel walls. Existing tools don't have H2-specific corrosion models. Market: 500+ hydrogen projects planned globally, $300B investment by 2030.

**Solution:**
- Hydrogen embrittlement susceptibility: calculate critical hydrogen concentration in steel wall based on stress, microstructure, and H2 partial pressure
- Hydrogen diffusion modeling: FEM-based H2 transport through pipe wall with trap binding effects (carbides, dislocations)
- Material qualification: screen materials for H2 service per ASME B31.12 and CHMC guidelines
- Pressure cycling effects: fatigue crack growth rate enhancement due to H2 environment
- Blended H2/CH4 pipelines: corrosion and embrittlement for repurposed natural gas pipelines (up to 20% H2 blend)

**Technical:**
- Rust FEM solver for H2 diffusion (1D through-wall, 2D for welds)
- Trap binding kinetics (Oriani equilibrium, McNabb-Foster kinetics)
- Material database extension: 100+ H2-compatible materials (Cr-Mo steels, austenitic SS, Inconel)

**Pricing:** H2 module add-on: $99/mo on top of Pro/Advanced plan

### v1.4 — Digital Twin Integration (Months 6-9)

**Problem:** Operators want real-time corrosion monitoring linked to predictive models for continuous updating. SCADA systems provide pressure/temperature/flow, but corrosion prediction runs offline in spreadsheets. Market: 100+ large operators with 10,000+ km pipeline networks, $50K-$200K/year per digital twin deployment.

**Solution:**
- SCADA integration: REST API to ingest real-time P/T/flow data from pipeline SCADA (AVEVA, Honeywell, Emerson)
- Automatic model update: recalculate corrosion rate every 1 hour based on current operating conditions
- Anomaly detection: flag sudden corrosion rate increase (e.g., flow regime change from stratified to slug)
- Inspection data correlation: auto-update corrosion model calibration when new ILI data arrives
- Predictive maintenance: forecast when wall thickness will reach minimum allowable, trigger inspection or repair work order

**Technical:**
- REST API for SCADA data push (JSON, OPC UA, or MQTT)
- Time-series database (TimescaleDB extension on PostgreSQL) for corrosion rate history
- Background job: hourly corrosion recalculation for active projects
- Alerting: email/SMS when corrosion rate exceeds threshold or remaining life < 6 months

**Pricing:** Enterprise plan (custom), minimum $2K/month for 10 digital twins

### v1.5 — Offshore and Subsea Corrosion (Months 7-10)

**Problem:** Offshore oil & gas and offshore wind operators face unique corrosion challenges: seawater (high chloride, biofouling), top-of-line corrosion in subsea flowlines, galvanic corrosion at dissimilar metal junctions (Ti clad to carbon steel), microbiologically influenced corrosion (MIC). Market: 10,000+ offshore platforms, 50,000 km subsea pipelines.

**Solution:**
- Seawater corrosion models: chloride pitting, crevice corrosion at flanges, MIC risk assessment
- Top-of-line corrosion (TLC): condensation rate at pipe crown, droplet chemistry (concentrated chloride, low pH), localized corrosion prediction
- Galvanic corrosion: coupled multi-electrode model for CRA clad/weld overlay to carbon steel transitions
- Under-deposit corrosion: sand/scale accumulation risk, localized chemistry under deposits
- Marine growth: biofouling effect on oxygen transport, MIC (sulfate-reducing bacteria, acid-producing bacteria)

**Technical:**
- Python water chemistry service extension: seawater equilibrium (Pitzer model for high ionic strength)
- Rust galvanic solver: multi-electrode Evans diagram, area ratio effects
- MIC risk scoring: empirical model based on temperature, sulfate, organic carbon, flow velocity

**Pricing:** Offshore module add-on: $149/mo on top of Pro/Advanced plan
