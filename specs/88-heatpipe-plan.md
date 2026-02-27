# 88. HeatPipe — Heat Pipe and Thermal Device Design Platform

## Implementation Plan

**MVP Scope:** Browser-based 1D/2D heat pipe design and analysis tool with interactive device geometry editor (cylindrical heat pipes, flat heat pipes, and basic vapor chambers), physics-based two-phase thermal-hydraulic solver implementing operating limit calculations (capillary, boiling, entrainment, viscous, sonic), wick structure design module with three wick types (sintered powder, axial grooves, screen mesh) and property calculators (capillary pressure via Young-Laplace, permeability via Kozeny-Carman, effective thermal conductivity via effective medium theory), working fluid database with 15 common fluids (water, methanol, ethanol, acetone, ammonia, and selected refrigerants) and temperature-dependent thermophysical properties (density, viscosity, surface tension, latent heat, vapor pressure), performance envelope generation (Qmax vs. temperature) with orientation effects (gravity-assisted, horizontal, gravity-opposed), thermal resistance breakdown and decomposition (evaporator wall, evaporator wick, vapor core, condenser wick, condenser wall), material compatibility matrix for common case-wick-fluid combinations, parametric sweep for single-variable sensitivity analysis, Stripe billing with three tiers (Free / Pro $99/mo / Advanced $249/user/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| Thermal Solver | Rust (native + WASM) | 1D/2D heat pipe thermal-hydraulic models, operating limit calculations |
| WASM Compilation | `wasm-pack` + `wasm-bindgen` | Client-side solver for simple heat pipes, instant feedback |
| ML/Optimization | Python 3.12 (FastAPI) | Surrogate models for rapid parametric exploration, fluid recommendation |
| Database | PostgreSQL 16 | Projects, users, simulations, fluid properties, wick correlations |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | Simulation results, performance envelopes, device geometry snapshots |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| Device Visualization | Canvas + WebGL | 2D device cross-section, temperature/pressure field rendering |
| Plot Rendering | Custom Canvas | Performance envelopes, thermal resistance breakdown, parametric sweeps |
| Real-time | WebSocket (Axum) | Live simulation progress for server-side jobs |
| Job Queue | Redis 7 + Tokio tasks | Server-side simulation job management for large parametric sweeps |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, error tracking |
| CI/CD | GitHub Actions | WASM build pipeline, Rust tests, Docker image push |

### Key Architecture Decisions

1. **Hybrid client/server solver with WASM for simple 1D heat pipes**: Simple 1D cylindrical heat pipe steady-state analysis (operating limit calculations, performance envelope generation) runs in the browser via WASM for instant feedback with zero server cost. Complex 2D vapor chamber simulations, transient analysis, and large parametric sweeps (100+ points) are executed server-side. This gives immediate results for 90%+ of exploratory design tasks while reserving server compute for detailed analysis.

2. **Custom Rust thermal-hydraulic solver rather than wrapping general-purpose CFD**: Heat pipe physics are governed by well-established 1D/2D correlations and limit equations (Chi 1976, Faghri 2016) that don't require full Navier-Stokes CFD. A purpose-built Rust solver implementing Modified Darcy flow in the wick, vapor core pressure drop correlations, and interfacial heat/mass transfer gives results 1000× faster than CFD with sufficient accuracy for design. WASM compilation of a custom Rust solver is straightforward, whereas WASM-compiling OpenFOAM or Fluent is infeasible.

3. **Canvas-based device geometry editor with WebGL thermal field visualization**: The device geometry editor uses Canvas 2D for interactive drawing of heat pipe cross-sections (evaporator, adiabatic, condenser sections; wick layers; vapor core). Temperature and pressure fields computed by the solver are rendered using WebGL with color maps for instant visual feedback. This is simpler than full 3D CAD integration while providing sufficient fidelity for 1D/2D heat pipe design.

4. **PostgreSQL for fluid property tables with cubic spline interpolation in Rust**: Working fluid thermophysical properties (ρ_l, ρ_v, μ_l, μ_v, σ, h_fg, k_l, P_sat) are stored as tabular data in PostgreSQL and loaded into the Rust solver. The solver uses cubic spline interpolation (via `splines` crate) for fast, accurate property evaluation at arbitrary temperatures. This approach is more flexible than hardcoding correlations and allows easy addition of new fluids.

5. **S3 for simulation result storage with PostgreSQL metadata**: Full simulation results (temperature profiles, pressure distributions, operating limit curves) are stored in S3 as binary arrays, while PostgreSQL holds metadata (device geometry, simulation parameters, key performance metrics). This allows efficient retrieval of summary data for dashboard display while deferring large result downloads until the user requests detailed plots.

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";  -- For fuzzy text search on fluids

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

-- Organizations (for Advanced plan team features)
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

-- Projects (heat pipe design workspace)
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    device_type TEXT NOT NULL,  -- cylindrical_heat_pipe | flat_heat_pipe | vapor_chamber | loop_heat_pipe | thermosiphon
    geometry_data JSONB NOT NULL DEFAULT '{}',  -- Full device geometry (lengths, diameters, wick structure)
    settings JSONB DEFAULT '{}',  -- Simulation defaults, unit preferences
    is_public BOOLEAN NOT NULL DEFAULT false,
    forked_from UUID REFERENCES projects(id) ON DELETE SET NULL,
    thumbnail_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);
CREATE INDEX projects_org_idx ON projects(org_id);
CREATE INDEX projects_device_type_idx ON projects(device_type);
CREATE INDEX projects_updated_idx ON projects(updated_at DESC);

-- Simulations
CREATE TABLE simulations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    analysis_type TEXT NOT NULL,  -- steady_state_1d | steady_state_2d | transient | performance_envelope | parametric_sweep
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | running | completed | failed | cancelled
    execution_mode TEXT NOT NULL DEFAULT 'wasm',  -- wasm | server
    geometry_snapshot JSONB NOT NULL,  -- Geometry at time of simulation
    parameters JSONB NOT NULL DEFAULT '{}',  -- Analysis-specific parameters (temperature, orientation, heat load, etc.)
    working_fluid TEXT NOT NULL,  -- e.g., "water", "methanol", "ammonia"
    wick_type TEXT NOT NULL,  -- sintered_powder | axial_grooves | screen_mesh
    results_url TEXT,  -- S3 URL for result data
    results_summary JSONB,  -- Quick-access summary (Qmax, thermal resistance, limiting factor)
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
    cores_allocated INTEGER DEFAULT 2,
    memory_mb INTEGER DEFAULT 4096,
    priority INTEGER DEFAULT 0,  -- Higher = more urgent
    progress_pct REAL DEFAULT 0.0,
    convergence_data JSONB DEFAULT '[]',  -- [{iteration, residual, temperature_change}]
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX jobs_sim_idx ON simulation_jobs(simulation_id);
CREATE INDEX jobs_worker_idx ON simulation_jobs(worker_id);

-- Working Fluids (reference database)
CREATE TABLE working_fluids (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT UNIQUE NOT NULL,  -- e.g., "water", "methanol", "ammonia"
    chemical_formula TEXT,
    description TEXT,
    temp_range_min_c REAL NOT NULL,  -- Minimum operating temperature (°C)
    temp_range_max_c REAL NOT NULL,  -- Maximum operating temperature (°C)
    properties JSONB NOT NULL,  -- Temperature-dependent property tables (T, rho_l, rho_v, mu_l, mu_v, sigma, h_fg, k_l, P_sat)
    toxicity TEXT,  -- low | moderate | high
    flammability TEXT,  -- non_flammable | flammable | highly_flammable
    environmental_impact TEXT,  -- low | moderate | high (ODP, GWP)
    cost_category TEXT,  -- low | moderate | high
    is_cryogenic BOOLEAN DEFAULT false,
    is_high_temperature BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX fluids_name_idx ON working_fluids(name);
CREATE INDEX fluids_temp_range_idx ON working_fluids(temp_range_min_c, temp_range_max_c);

-- Material Compatibility (case-wick-fluid combinations)
CREATE TABLE material_compatibility (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    case_material TEXT NOT NULL,  -- copper | aluminum | stainless_steel | titanium | nickel
    wick_material TEXT NOT NULL,  -- copper_powder | copper_mesh | copper_grooves | stainless_steel_mesh | nickel_powder
    fluid_name TEXT NOT NULL REFERENCES working_fluids(name),
    compatibility TEXT NOT NULL,  -- excellent | good | acceptable | poor | incompatible
    max_operating_temp_c REAL,  -- Maximum safe operating temperature for this combination
    ncg_generation TEXT,  -- none | low | moderate | high (non-condensable gas generation)
    life_test_data TEXT,  -- Reference to published life test data or internal testing
    notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX compat_case_idx ON material_compatibility(case_material);
CREATE INDEX compat_fluid_idx ON material_compatibility(fluid_name);
CREATE UNIQUE INDEX compat_unique_idx ON material_compatibility(case_material, wick_material, fluid_name);

-- Wick Correlations (empirical data for wick property calculations)
CREATE TABLE wick_correlations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    wick_type TEXT NOT NULL,  -- sintered_powder | axial_grooves | screen_mesh
    subtype TEXT,  -- e.g., "rectangular_groove", "triangular_groove", "composite"
    correlation_type TEXT NOT NULL,  -- capillary_pressure | permeability | effective_conductivity | porosity
    formula TEXT NOT NULL,  -- Mathematical formula as string (for display/documentation)
    parameters JSONB NOT NULL,  -- Correlation coefficients and constants
    reference TEXT,  -- Citation to literature source
    valid_range JSONB,  -- Valid range of input parameters
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX wick_corr_type_idx ON wick_correlations(wick_type, correlation_type);

-- Performance Templates (pre-configured device templates)
CREATE TABLE device_templates (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    description TEXT,
    device_type TEXT NOT NULL,
    geometry_data JSONB NOT NULL,
    working_fluid TEXT NOT NULL REFERENCES working_fluids(name),
    wick_type TEXT NOT NULL,
    use_case TEXT,  -- e.g., "smartphone_cooling", "satellite_thermal", "ev_battery_cooling"
    thumbnail_url TEXT,
    is_public BOOLEAN DEFAULT true,
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX templates_device_type_idx ON device_templates(device_type);
CREATE INDEX templates_use_case_idx ON device_templates(use_case);

-- Usage Tracking (for billing enforcement)
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,  -- simulation_count | server_simulation_minutes | storage_bytes
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

use chrono::{DateTime, Utc};
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
    pub device_type: String,
    pub geometry_data: serde_json::Value,
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
    pub analysis_type: String,
    pub status: String,
    pub execution_mode: String,
    pub geometry_snapshot: serde_json::Value,
    pub parameters: serde_json::Value,
    pub working_fluid: String,
    pub wick_type: String,
    pub results_url: Option<String>,
    pub results_summary: Option<serde_json::Value>,
    pub error_message: Option<String>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub wall_time_ms: Option<i32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct WorkingFluid {
    pub id: Uuid,
    pub name: String,
    pub chemical_formula: Option<String>,
    pub description: Option<String>,
    pub temp_range_min_c: f64,
    pub temp_range_max_c: f64,
    pub properties: serde_json::Value,  // Temperature-dependent property tables
    pub toxicity: Option<String>,
    pub flammability: Option<String>,
    pub environmental_impact: Option<String>,
    pub cost_category: Option<String>,
    pub is_cryogenic: bool,
    pub is_high_temperature: bool,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct MaterialCompatibility {
    pub id: Uuid,
    pub case_material: String,
    pub wick_material: String,
    pub fluid_name: String,
    pub compatibility: String,
    pub max_operating_temp_c: Option<f64>,
    pub ncg_generation: Option<String>,
    pub life_test_data: Option<String>,
    pub notes: Option<String>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct SimulationParams {
    pub analysis_type: AnalysisType,
    pub operating_temperature_c: f64,
    pub orientation_deg: f64,  // 0 = horizontal, 90 = evaporator down (gravity-assisted), -90 = evaporator up (gravity-opposed)
    pub heat_load_w: Option<f64>,  // Fixed heat load for steady-state, None for performance envelope
    pub ambient_temperature_c: Option<f64>,  // For condenser boundary condition
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum AnalysisType {
    SteadyState1d,
    SteadyState2d,
    Transient,
    PerformanceEnvelope,
    ParametricSweep,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct DeviceGeometry {
    pub device_type: DeviceType,
    pub total_length_m: f64,
    pub evaporator_length_m: f64,
    pub condenser_length_m: f64,
    pub outer_diameter_m: Option<f64>,  // For cylindrical
    pub width_m: Option<f64>,  // For flat/vapor chamber
    pub height_m: Option<f64>,  // For flat/vapor chamber
    pub wall_thickness_m: f64,
    pub wick: WickGeometry,
    pub vapor_core_diameter_m: f64,
    pub case_material: String,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum DeviceType {
    CylindricalHeatPipe,
    FlatHeatPipe,
    VaporChamber,
    LoopHeatPipe,
    Thermosiphon,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct WickGeometry {
    pub wick_type: WickType,
    pub wick_thickness_m: f64,
    pub wick_material: String,
    // Sintered powder parameters
    pub particle_diameter_m: Option<f64>,
    pub porosity: Option<f64>,
    // Groove parameters
    pub groove_width_m: Option<f64>,
    pub groove_depth_m: Option<f64>,
    pub groove_spacing_m: Option<f64>,
    pub groove_shape: Option<String>,  // rectangular | triangular | trapezoidal
    // Mesh parameters
    pub mesh_number: Option<i32>,
    pub wire_diameter_m: Option<f64>,
    pub num_layers: Option<i32>,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum WickType {
    SinteredPowder,
    AxialGrooves,
    ScreenMesh,
    Composite,
}
```

---

## Solver Architecture Deep-Dive

### Governing Equations and Physics

HeatPipe's solver implements 1D and 2D steady-state thermal-hydraulic models for two-phase heat transfer devices. The core physics are:

**1. Vapor Core Flow (1D momentum equation):**

For the vapor phase flowing from evaporator to condenser, the pressure drop is:

```
ΔP_vapor = (f_v · ρ_v · L_eff / (2 · D_v)) · u_v²

where:
  f_v = friction factor (laminar: 16/Re, turbulent: Blasius correlation)
  ρ_v = vapor density at operating temperature
  L_eff = effective flow length (evaporator + adiabatic + condenser)
  D_v = vapor core diameter
  u_v = vapor velocity = Q / (ρ_v · h_fg · A_v)
  Q = heat transport (W)
  h_fg = latent heat of vaporization (J/kg)
  A_v = vapor core cross-sectional area (m²)
```

**2. Liquid Flow in Wick (Modified Darcy's Law):**

Liquid returns from condenser to evaporator through the wick via capillary action. The pressure drop in the liquid is:

```
ΔP_liquid = (μ_l / K) · (Q / (ρ_l · h_fg · A_w)) · L_eff

where:
  μ_l = liquid dynamic viscosity (Pa·s)
  K = wick permeability (m²) — depends on wick type and geometry
  A_w = wick cross-sectional area for liquid flow (m²)
  ρ_l = liquid density (kg/m³)
```

**3. Capillary Pressure (Young-Laplace Equation):**

The maximum capillary pressure the wick can generate is:

```
ΔP_capillary = (2 · σ · cos(θ)) / r_eff

where:
  σ = surface tension (N/m)
  θ = contact angle (typically 0° for good wetting, cos(θ) ≈ 1)
  r_eff = effective pore radius (m) — depends on wick type
```

**4. Operating Limit Equations:**

The heat pipe can fail via several mechanisms. The maximum heat transport is the minimum of all limits:

**Capillary Limit (most common):** Capillary pressure must overcome liquid and vapor pressure drops plus gravitational head:

```
Q_cap = (ΔP_capillary - ρ_l · g · L · sin(θ_tilt)) / (μ_l / (K · ρ_l · h_fg · A_w) · L_eff + f_v · L_eff / (2 · D_v · ρ_v · h_fg · A_v))
```

**Boiling Limit:** Nucleate boiling in wick causes vapor bubbles that block liquid return:

```
Q_boil = (2π · L_e · k_eff · T_op) / ln(r_n / r_w) · (1 / (h_fg · ρ_v))

where:
  k_eff = effective thermal conductivity of wick (W/m·K)
  T_op = operating temperature (K)
  r_n = nucleation site radius (typically 1-10 µm)
  r_w = wick pore radius
  L_e = evaporator length
```

**Entrainment Limit:** High vapor velocity shears liquid droplets from wick surface:

```
Q_ent = A_v · h_fg · ρ_v^0.5 · (σ / (2 · r_w))^0.5

Based on Weber number criterion: We = ρ_v · u_v² · r_w / σ < We_crit ≈ 1
```

**Viscous Limit:** Vapor pressure drop exceeds vapor generation pressure at low temperatures:

```
Q_visc = (r_v² · A_v · h_fg · ρ_v · P_sat(T)) / (16 · μ_v · L_eff)

Typically dominates at startup or low temperature operation
```

**Sonic Limit:** Vapor flow reaches Mach 1, choking the flow:

```
Q_sonic = A_v · h_fg · ρ_v · (γ · R_specific · T_op)^0.5

where:
  γ = heat capacity ratio (typically 1.33 for vapors)
  R_specific = gas constant / molecular weight
Rarely limiting except for liquid metal heat pipes
```

**5. Thermal Resistance Network:**

The overall thermal resistance from evaporator to condenser is the sum of series resistances:

```
R_total = R_evap_wall + R_evap_wick + R_vapor + R_cond_wick + R_cond_wall

R_evap_wall = t_wall / (k_wall · A_evap)
R_evap_wick = t_wick / (k_eff · A_evap)
R_vapor = L_eff / (k_v · A_vapor)  — typically negligible due to phase change
R_cond_wick = t_wick / (k_eff · A_cond)
R_cond_wall = t_wall / (k_wall · A_cond)
```

For a given heat load Q and thermal resistance R_total, the temperature drop is:

```
ΔT = Q · R_total
```

### Client/Server Split (WASM Threshold)

```
Heat pipe design created → Analysis requested
    │
    ├── 1D steady-state analysis → WASM solver (browser)
    │   ├── Operating limit calculations (5 limits computed)
    │   ├── Performance envelope (Qmax vs. T sweep)
    │   ├── Thermal resistance breakdown
    │   └── Results displayed immediately (<1 second)
    │
    └── 2D vapor chamber OR transient OR large parametric sweep → Server solver
        ├── Job queued via Redis
        ├── Worker picks up, runs native Rust solver
        ├── Progress streamed via WebSocket
        └── Results stored in S3, URL returned
```

1D cylindrical heat pipe steady-state analysis runs entirely in WASM because:
- Operating limit calculations are closed-form equations (no iteration required)
- Performance envelope sweep (50 temperature points) takes <500ms in WASM
- Covers 80%+ of initial design exploration tasks
- Zero server cost, instant feedback

2D vapor chamber analysis requires server execution because:
- 2D finite-element thermal solver with iterative convergence
- Mesh generation and assembly requires more memory
- Typical solve time 5-30 seconds depending on mesh resolution
- Results include full 2D temperature/pressure fields (10-100 MB)

### WASM Compilation Pipeline

```toml
# solver-wasm/Cargo.toml
[package]
name = "heatpipe-solver-wasm"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
wasm-bindgen = "0.2"
serde = { version = "1", features = ["derive"] }
serde-wasm-bindgen = "0.6"
js-sys = "0.3"
splines = "4.3"  # Cubic spline interpolation for fluid properties
nalgebra = "0.33"  # Linear algebra for wick property calculations
log = "0.4"
console_log = "1"

[profile.release]
opt-level = "z"       # Optimize for size
lto = true            # Link-time optimization
codegen-units = 1     # Single codegen unit for better optimization
strip = true          # Strip debug symbols
```

---

## Architecture Deep-Dives

### 1. Heat Pipe Simulation API Handler (Rust/Axum)

The primary endpoint receives a simulation request, validates the device geometry, decides between WASM and server execution, and for server execution enqueues a job.

```rust
// src/api/handlers/simulation.rs

use axum::{
    extract::{Path, State, Json},
    http::StatusCode,
    response::IntoResponse,
};
use uuid::Uuid;
use crate::{
    db::models::{Simulation, SimulationParams, DeviceGeometry},
    state::AppState,
    auth::Claims,
    error::ApiError,
};

#[derive(serde::Deserialize)]
pub struct CreateSimulationRequest {
    pub analysis_type: crate::db::models::AnalysisType,
    pub parameters: SimulationParams,
    pub working_fluid: String,
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

    // 2. Validate working fluid exists
    let fluid = sqlx::query_as!(
        crate::db::models::WorkingFluid,
        "SELECT * FROM working_fluids WHERE name = $1",
        req.working_fluid
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::BadRequest("Invalid working fluid"))?;

    // 3. Validate operating temperature is within fluid range
    if req.parameters.operating_temperature_c < fluid.temp_range_min_c
        || req.parameters.operating_temperature_c > fluid.temp_range_max_c
    {
        return Err(ApiError::BadRequest(
            &format!("Operating temperature {:.1}°C is outside valid range for {} ({:.1}°C to {:.1}°C)",
                req.parameters.operating_temperature_c,
                fluid.name,
                fluid.temp_range_min_c,
                fluid.temp_range_max_c)
        ));
    }

    // 4. Parse device geometry from project
    let geometry: DeviceGeometry = serde_json::from_value(project.geometry_data.clone())?;

    // 5. Check plan limits
    let user = sqlx::query_as!(
        crate::db::models::User,
        "SELECT * FROM users WHERE id = $1",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    // Free plan: 1D analysis only, no parametric sweeps
    if user.plan == "free" {
        if matches!(req.analysis_type, crate::db::models::AnalysisType::SteadyState2d) {
            return Err(ApiError::PlanLimit(
                "2D vapor chamber analysis requires Pro plan or higher."
            ));
        }
        if matches!(req.analysis_type, crate::db::models::AnalysisType::ParametricSweep) {
            return Err(ApiError::PlanLimit(
                "Parametric sweeps require Pro plan or higher."
            ));
        }
    }

    // 6. Determine execution mode
    let execution_mode = match req.analysis_type {
        crate::db::models::AnalysisType::SteadyState1d => "wasm",
        crate::db::models::AnalysisType::PerformanceEnvelope => "wasm",
        _ => "server",  // 2D, transient, parametric all go to server
    };

    // 7. Extract wick type from geometry
    let wick_type = match geometry.wick.wick_type {
        crate::db::models::WickType::SinteredPowder => "sintered_powder",
        crate::db::models::WickType::AxialGrooves => "axial_grooves",
        crate::db::models::WickType::ScreenMesh => "screen_mesh",
        crate::db::models::WickType::Composite => "composite",
    };

    // 8. Create simulation record
    let sim = sqlx::query_as!(
        Simulation,
        r#"INSERT INTO simulations
            (project_id, user_id, analysis_type, status, execution_mode,
             geometry_snapshot, parameters, working_fluid, wick_type)
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9)
        RETURNING *"#,
        project_id,
        claims.user_id,
        serde_json::to_string(&req.analysis_type)?,
        if execution_mode == "wasm" { "completed" } else { "pending" },
        execution_mode,
        project.geometry_data,
        serde_json::to_value(&req.parameters)?,
        req.working_fluid,
        wick_type,
    )
    .fetch_one(&state.db)
    .await?;

    // 9. For server execution, enqueue job
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
        let mut redis_conn = state.redis.get_multiplexed_async_connection().await?;
        redis::cmd("LPUSH")
            .arg("simulation:jobs")
            .arg(job.id.to_string())
            .query_async::<_, ()>(&mut redis_conn)
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
```

### 2. Heat Pipe Solver Core (Rust — shared between WASM and native)

The core 1D heat pipe solver that computes operating limits and performance envelope.

```rust
// solver-core/src/heat_pipe_1d.rs

use nalgebra::Vector3;
use std::f64::consts::PI;

#[derive(Debug, Clone)]
pub struct HeatPipeGeometry {
    pub total_length_m: f64,
    pub evaporator_length_m: f64,
    pub condenser_length_m: f64,
    pub outer_diameter_m: f64,
    pub wall_thickness_m: f64,
    pub wick_thickness_m: f64,
    pub vapor_core_diameter_m: f64,
}

#[derive(Debug, Clone)]
pub struct WickProperties {
    pub wick_type: WickType,
    pub permeability_m2: f64,
    pub effective_pore_radius_m: f64,
    pub porosity: f64,
    pub effective_thermal_conductivity_w_mk: f64,
}

#[derive(Debug, Clone)]
pub enum WickType {
    SinteredPowder,
    AxialGrooves,
    ScreenMesh,
}

#[derive(Debug, Clone)]
pub struct FluidProperties {
    pub name: String,
    pub temperature_c: f64,
    pub rho_liquid_kg_m3: f64,
    pub rho_vapor_kg_m3: f64,
    pub mu_liquid_pa_s: f64,
    pub mu_vapor_pa_s: f64,
    pub surface_tension_n_m: f64,
    pub latent_heat_j_kg: f64,
    pub k_liquid_w_mk: f64,
    pub k_vapor_w_mk: f64,
    pub vapor_pressure_pa: f64,
}

#[derive(Debug, Clone)]
pub struct OperatingLimits {
    pub capillary_limit_w: f64,
    pub boiling_limit_w: f64,
    pub entrainment_limit_w: f64,
    pub viscous_limit_w: f64,
    pub sonic_limit_w: f64,
    pub limiting_factor: String,
    pub max_heat_transport_w: f64,
}

#[derive(Debug, Clone)]
pub struct ThermalResistance {
    pub evap_wall_k_w: f64,
    pub evap_wick_k_w: f64,
    pub vapor_k_w: f64,
    pub cond_wick_k_w: f64,
    pub cond_wall_k_w: f64,
    pub total_k_w: f64,
}

pub struct HeatPipeSolver {
    pub geometry: HeatPipeGeometry,
    pub wick: WickProperties,
    pub fluid: FluidProperties,
    pub wall_material_k_w_mk: f64,
    pub orientation_deg: f64,  // 0 = horizontal, 90 = evap down, -90 = evap up
}

impl HeatPipeSolver {
    pub fn new(
        geometry: HeatPipeGeometry,
        wick: WickProperties,
        fluid: FluidProperties,
        wall_material_k_w_mk: f64,
        orientation_deg: f64,
    ) -> Self {
        Self {
            geometry,
            wick,
            fluid,
            wall_material_k_w_mk,
            orientation_deg,
        }
    }

    /// Calculate all operating limits and determine maximum heat transport
    pub fn calculate_operating_limits(&self) -> OperatingLimits {
        let cap = self.capillary_limit();
        let boil = self.boiling_limit();
        let ent = self.entrainment_limit();
        let visc = self.viscous_limit();
        let sonic = self.sonic_limit();

        let limits = vec![
            ("Capillary", cap),
            ("Boiling", boil),
            ("Entrainment", ent),
            ("Viscous", visc),
            ("Sonic", sonic),
        ];

        let (limiting_factor, max_heat) = limits
            .iter()
            .min_by(|a, b| a.1.partial_cmp(&b.1).unwrap())
            .unwrap();

        OperatingLimits {
            capillary_limit_w: cap,
            boiling_limit_w: boil,
            entrainment_limit_w: ent,
            viscous_limit_w: visc,
            sonic_limit_w: sonic,
            limiting_factor: limiting_factor.to_string(),
            max_heat_transport_w: *max_heat,
        }
    }

    /// Capillary limit (most common limiting factor)
    fn capillary_limit(&self) -> f64 {
        const G: f64 = 9.81;  // m/s²

        // Capillary pressure (Young-Laplace)
        let delta_p_cap = 2.0 * self.fluid.surface_tension_n_m / self.wick.effective_pore_radius_m;

        // Gravitational pressure penalty (positive for gravity-assisted, negative for gravity-opposed)
        let tilt_rad = self.orientation_deg.to_radians();
        let delta_p_gravity = self.fluid.rho_liquid_kg_m3 * G * self.geometry.total_length_m * tilt_rad.sin();

        // Effective length for pressure drop calculation
        let l_eff = self.geometry.evaporator_length_m
            + (self.geometry.total_length_m - self.geometry.evaporator_length_m - self.geometry.condenser_length_m)
            + self.geometry.condenser_length_m;

        // Wick cross-sectional area for liquid flow
        let r_outer = self.geometry.outer_diameter_m / 2.0 - self.geometry.wall_thickness_m;
        let r_inner = r_outer - self.wick.wick_thickness_m;
        let a_wick = PI * (r_outer.powi(2) - r_inner.powi(2));

        // Vapor core cross-sectional area
        let r_vapor = self.geometry.vapor_core_diameter_m / 2.0;
        let a_vapor = PI * r_vapor.powi(2);

        // Liquid pressure drop coefficient
        let c_liquid = (self.fluid.mu_liquid_pa_s * l_eff)
            / (self.wick.permeability_m2 * self.fluid.rho_liquid_kg_m3 * self.fluid.latent_heat_j_kg * a_wick);

        // Vapor pressure drop coefficient
        let re_v = 1e5;  // Estimate, typically turbulent
        let f_v = if re_v < 2300.0 {
            16.0 / re_v
        } else {
            0.079 / re_v.powf(0.25)  // Blasius correlation
        };
        let c_vapor = (f_v * l_eff)
            / (2.0 * self.geometry.vapor_core_diameter_m * self.fluid.rho_vapor_kg_m3 * self.fluid.latent_heat_j_kg * a_vapor);

        // Capillary limit
        let q_cap = (delta_p_cap - delta_p_gravity) / (c_liquid + c_vapor);

        q_cap.max(0.0)  // Can't be negative
    }

    /// Boiling limit
    fn boiling_limit(&self) -> f64 {
        // Nucleation site radius (empirical, typically 1-10 µm)
        let r_nucleation = 5e-6;  // m

        let r_outer = self.geometry.outer_diameter_m / 2.0 - self.geometry.wall_thickness_m;

        let temp_k = self.fluid.temperature_c + 273.15;

        // Simplified boiling limit (Chi 1976, Eq. 3.56)
        let q_boil = (2.0 * PI * self.geometry.evaporator_length_m * self.wick.effective_thermal_conductivity_w_mk * temp_k)
            / (self.fluid.latent_heat_j_kg * self.fluid.rho_vapor_kg_m3 * (r_nucleation / r_outer).ln());

        q_boil
    }

    /// Entrainment limit (Weber number criterion)
    fn entrainment_limit(&self) -> f64 {
        let r_vapor = self.geometry.vapor_core_diameter_m / 2.0;
        let a_vapor = PI * r_vapor.powi(2);

        // Weber number critical value (typically ~1)
        // We = rho_v * u_v^2 * r_pore / sigma < 1
        // Rearranging for max velocity, then Q = rho_v * u_v * A_v * h_fg
        let q_ent = self.fluid.rho_vapor_kg_m3.sqrt() * a_vapor * self.fluid.latent_heat_j_kg
            * (self.fluid.surface_tension_n_m / (2.0 * self.wick.effective_pore_radius_m)).sqrt();

        q_ent
    }

    /// Viscous limit (typically dominates at low temperature/startup)
    fn viscous_limit(&self) -> f64 {
        let r_vapor = self.geometry.vapor_core_diameter_m / 2.0;
        let a_vapor = PI * r_vapor.powi(2);

        let l_eff = self.geometry.evaporator_length_m
            + (self.geometry.total_length_m - self.geometry.evaporator_length_m - self.geometry.condenser_length_m)
            + self.geometry.condenser_length_m;

        // Viscous limit (laminar vapor flow)
        let q_visc = (r_vapor.powi(2) * a_vapor * self.fluid.latent_heat_j_kg
            * self.fluid.rho_vapor_kg_m3 * self.fluid.vapor_pressure_pa)
            / (16.0 * self.fluid.mu_vapor_pa_s * l_eff);

        q_visc
    }

    /// Sonic limit (rarely limiting except liquid metals)
    fn sonic_limit(&self) -> f64 {
        let r_vapor = self.geometry.vapor_core_diameter_m / 2.0;
        let a_vapor = PI * r_vapor.powi(2);

        let temp_k = self.fluid.temperature_c + 273.15;

        // Assume gamma = 1.33 for typical vapors, molecular weight estimated from density
        let gamma = 1.33;
        let r_specific = 8314.0 / 18.0;  // J/(kg·K) for water, approximate

        let sound_speed = (gamma * r_specific * temp_k).sqrt();

        let q_sonic = a_vapor * self.fluid.latent_heat_j_kg * self.fluid.rho_vapor_kg_m3 * sound_speed;

        q_sonic
    }

    /// Calculate thermal resistance breakdown for a given heat load
    pub fn calculate_thermal_resistance(&self, heat_load_w: f64) -> ThermalResistance {
        let r_outer = self.geometry.outer_diameter_m / 2.0 - self.geometry.wall_thickness_m;
        let r_inner_wick = r_outer - self.wick.wick_thickness_m;

        // Evaporator areas
        let a_evap_outer = 2.0 * PI * (self.geometry.outer_diameter_m / 2.0) * self.geometry.evaporator_length_m;
        let a_evap_inner = 2.0 * PI * r_outer * self.geometry.evaporator_length_m;

        // Condenser areas
        let a_cond_outer = 2.0 * PI * (self.geometry.outer_diameter_m / 2.0) * self.geometry.condenser_length_m;
        let a_cond_inner = 2.0 * PI * r_outer * self.geometry.condenser_length_m;

        // Evaporator wall resistance (radial conduction)
        let r_evap_wall = (self.geometry.outer_diameter_m / 2.0).ln()
            / (2.0 * PI * self.wall_material_k_w_mk * self.geometry.evaporator_length_m)
            - r_outer.ln() / (2.0 * PI * self.wall_material_k_w_mk * self.geometry.evaporator_length_m);

        // Evaporator wick resistance
        let r_evap_wick = (r_outer / r_inner_wick).ln()
            / (2.0 * PI * self.wick.effective_thermal_conductivity_w_mk * self.geometry.evaporator_length_m);

        // Vapor core resistance (typically negligible due to phase change, but include conduction for completeness)
        let l_eff = self.geometry.total_length_m - self.geometry.evaporator_length_m - self.geometry.condenser_length_m;
        let a_vapor = PI * (self.geometry.vapor_core_diameter_m / 2.0).powi(2);
        let r_vapor = l_eff / (self.fluid.k_vapor_w_mk * a_vapor);

        // Condenser wick resistance
        let r_cond_wick = (r_outer / r_inner_wick).ln()
            / (2.0 * PI * self.wick.effective_thermal_conductivity_w_mk * self.geometry.condenser_length_m);

        // Condenser wall resistance
        let r_cond_wall = (self.geometry.outer_diameter_m / 2.0).ln()
            / (2.0 * PI * self.wall_material_k_w_mk * self.geometry.condenser_length_m)
            - r_outer.ln() / (2.0 * PI * self.wall_material_k_w_mk * self.geometry.condenser_length_m);

        let r_total = r_evap_wall + r_evap_wick + r_vapor + r_cond_wick + r_cond_wall;

        ThermalResistance {
            evap_wall_k_w: r_evap_wall,
            evap_wick_k_w: r_evap_wick,
            vapor_k_w: r_vapor,
            cond_wick_k_w: r_cond_wick,
            cond_wall_k_w: r_cond_wall,
            total_k_w: r_total,
        }
    }

    /// Generate performance envelope (Qmax vs. temperature)
    pub fn performance_envelope(&self, temp_range_c: (f64, f64), num_points: usize) -> Vec<(f64, f64)> {
        let (t_min, t_max) = temp_range_c;
        let dt = (t_max - t_min) / (num_points - 1) as f64;

        let mut envelope = Vec::with_capacity(num_points);

        for i in 0..num_points {
            let temp_c = t_min + dt * i as f64;

            // Create a new solver instance with updated fluid properties at this temperature
            // (In practice, fluid properties would be interpolated from database tables)
            let mut solver_at_temp = self.clone();
            solver_at_temp.fluid.temperature_c = temp_c;
            // Update temperature-dependent properties (simplified here)
            // In real implementation, this would call fluid property interpolation functions

            let limits = solver_at_temp.calculate_operating_limits();
            envelope.push((temp_c, limits.max_heat_transport_w));
        }

        envelope
    }
}

impl Clone for HeatPipeSolver {
    fn clone(&self) -> Self {
        Self {
            geometry: self.geometry.clone(),
            wick: self.wick.clone(),
            fluid: self.fluid.clone(),
            wall_material_k_w_mk: self.wall_material_k_w_mk,
            orientation_deg: self.orientation_deg,
        }
    }
}
```

### 3. Wick Property Calculator (Rust)

Calculates wick permeability, capillary pressure, and effective thermal conductivity from manufacturing parameters.

```rust
// solver-core/src/wick_properties.rs

use std::f64::consts::PI;

/// Calculate sintered powder wick properties from particle diameter and porosity
pub fn sintered_powder_properties(
    particle_diameter_m: f64,
    porosity: f64,
    wick_material_k_w_mk: f64,
    fluid_k_w_mk: f64,
) -> (f64, f64, f64) {
    // Kozeny-Carman permeability
    let permeability_m2 = (particle_diameter_m.powi(2) * porosity.powi(3))
        / (150.0 * (1.0 - porosity).powi(2));

    // Effective pore radius (approximation based on particle packing)
    let effective_pore_radius_m = particle_diameter_m * porosity / (6.0 * (1.0 - porosity));

    // Effective thermal conductivity (Maxwell-Eucken model for solid-fluid composite)
    let k_eff = fluid_k_w_mk * (
        (2.0 * fluid_k_w_mk + wick_material_k_w_mk - 2.0 * (1.0 - porosity) * (fluid_k_w_mk - wick_material_k_w_mk))
        / (2.0 * fluid_k_w_mk + wick_material_k_w_mk + (1.0 - porosity) * (fluid_k_w_mk - wick_material_k_w_mk))
    );

    (permeability_m2, effective_pore_radius_m, k_eff)
}

/// Calculate axial groove wick properties
pub fn axial_groove_properties(
    groove_width_m: f64,
    groove_depth_m: f64,
    groove_spacing_m: f64,
    num_grooves: usize,
    groove_shape: &str,  // "rectangular" | "triangular" | "trapezoidal"
    wick_material_k_w_mk: f64,
    fluid_k_w_mk: f64,
) -> (f64, f64, f64) {
    // Hydraulic radius depends on groove shape
    let (hydraulic_radius_m, flow_area_m2) = match groove_shape {
        "rectangular" => {
            let rh = (groove_width_m * groove_depth_m) / (2.0 * (groove_width_m + groove_depth_m));
            let area = groove_width_m * groove_depth_m * num_grooves as f64;
            (rh, area)
        },
        "triangular" => {
            let rh = groove_width_m / 4.0;  // Approximation for triangular cross-section
            let area = 0.5 * groove_width_m * groove_depth_m * num_grooves as f64;
            (rh, area)
        },
        _ => {
            // Default to rectangular
            let rh = (groove_width_m * groove_depth_m) / (2.0 * (groove_width_m + groove_depth_m));
            let area = groove_width_m * groove_depth_m * num_grooves as f64;
            (rh, area)
        }
    };

    // Permeability for open channels (Darcy-Weisbach)
    let permeability_m2 = hydraulic_radius_m.powi(2) / 8.0;

    // Effective pore radius for capillary pressure (depends on groove geometry)
    let effective_pore_radius_m = match groove_shape {
        "rectangular" => groove_width_m / 2.0,
        "triangular" => groove_width_m / (2.0 * 2.0_f64.sqrt()),  // Corner radius
        _ => groove_width_m / 2.0,
    };

    // Effective thermal conductivity (parallel combination of solid fins and liquid)
    let porosity = flow_area_m2 / (PI * groove_spacing_m * num_grooves as f64);  // Approximate
    let k_eff = porosity * fluid_k_w_mk + (1.0 - porosity) * wick_material_k_w_mk;

    (permeability_m2, effective_pore_radius_m, k_eff)
}

/// Calculate screen mesh wick properties
pub fn screen_mesh_properties(
    mesh_number: i32,  // Wires per inch
    wire_diameter_m: f64,
    num_layers: i32,
    wick_material_k_w_mk: f64,
    fluid_k_w_mk: f64,
) -> (f64, f64, f64) {
    // Mesh spacing
    let spacing_m = 0.0254 / mesh_number as f64;  // Convert mesh number to meters

    // Porosity (open area fraction)
    let porosity = ((spacing_m - wire_diameter_m) / spacing_m).powi(2);

    // Permeability (empirical correlation for woven mesh)
    let permeability_m2 = (wire_diameter_m.powi(2) * porosity.powi(3)) / (122.0 * (1.0 - porosity).powi(2));

    // Effective pore radius (half the spacing between wires)
    let effective_pore_radius_m = (spacing_m - wire_diameter_m) / 2.0;

    // Effective thermal conductivity (series combination through layers)
    let single_layer_k = porosity * fluid_k_w_mk + (1.0 - porosity) * wick_material_k_w_mk;
    let k_eff = single_layer_k;  // Simplified; actual would account for contact resistance between layers

    (permeability_m2, effective_pore_radius_m, k_eff)
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–4)

**Day 1: Project initialization and Rust backend scaffold**
```bash
cargo init heatpipe-api
cd heatpipe-api
cargo add axum tokio serde serde_json sqlx uuid chrono tower-http jsonwebtoken bcrypt
cargo add --features postgres,runtime-tokio-rustls,uuid,chrono,json sqlx
cargo add aws-sdk-s3 redis nalgebra splines
```
- `src/main.rs` — Axum app with CORS, tracing, graceful shutdown
- `src/config.rs` — Environment-based configuration (DATABASE_URL, JWT_SECRET, S3_BUCKET, REDIS_URL)
- `src/state.rs` — AppState struct (PgPool, Redis, S3Client)
- `src/error.rs` — ApiError enum with IntoResponse
- `Dockerfile` — Multi-stage build (builder + runtime)
- `docker-compose.yml` — PostgreSQL, Redis, MinIO (S3-compatible)

**Day 2: Database schema and migrations**
- `migrations/001_initial.sql` — All 9 tables: users, organizations, org_members, projects, simulations, simulation_jobs, working_fluids, material_compatibility, wick_correlations, device_templates, usage_records
- `src/db/mod.rs` — Database pool initialization
- `src/db/models.rs` — All SQLx structs with FromRow derives
- Run `sqlx migrate run` to apply schema
- Seed script for working fluids database (15 common fluids with temperature-dependent property tables)
- Seed script for material compatibility matrix (copper, aluminum, stainless steel cases with common fluids)
- Seed script for wick correlations (Kozeny-Carman for sintered powder, groove geometry formulas)

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

### Phase 2 — Solver Core 1D Heat Pipe (Days 5–12)

**Day 5: Fluid property database and interpolation**
- `solver-core/` — New Rust workspace member (shared between native and WASM)
- `solver-core/src/fluid_properties.rs` — FluidProperties struct, cubic spline interpolation for temperature-dependent properties
- Load fluid property tables from PostgreSQL into memory at solver initialization
- Implement interpolation functions: `get_density_liquid(T)`, `get_viscosity_vapor(T)`, `get_surface_tension(T)`, etc.
- Unit tests: interpolate water properties at 25°C, 50°C, 100°C and compare to reference values

**Day 6: Wick property calculators**
- `solver-core/src/wick_properties.rs` — Functions for sintered powder, axial grooves, screen mesh
- Kozeny-Carman permeability for sintered powder
- Young-Laplace capillary pressure for effective pore radius
- Maxwell-Eucken effective thermal conductivity for porous media
- Unit tests: verify permeability calculations against published data (Chi 1976 Table 3.1)

**Day 7: 1D heat pipe geometry and thermal resistance**
- `solver-core/src/heat_pipe_1d.rs` — HeatPipeGeometry struct, ThermalResistance calculation
- Radial conduction resistance through evaporator/condenser walls and wicks
- Axial conduction resistance through vapor core
- Thermal resistance breakdown into 5 components
- Unit tests: simple heat pipe with known geometry, verify total thermal resistance

**Day 8: Operating limit calculations — Capillary and Viscous**
- `solver-core/src/heat_pipe_1d.rs` — Capillary limit with gravity effects (orientation angle)
- Modified Darcy flow for liquid pressure drop in wick
- Vapor pressure drop (laminar/turbulent friction factor)
- Viscous limit (low-temperature startup)
- Unit tests: compare capillary limit to published results for water heat pipe at 25°C horizontal

**Day 9: Operating limit calculations — Boiling, Entrainment, Sonic**
- Boiling limit (nucleate boiling in wick pores)
- Entrainment limit (Weber number criterion)
- Sonic limit (vapor choking, rarely limiting)
- `calculate_operating_limits()` function that returns minimum of all 5 limits
- Unit tests: verify entrainment limit calculation for methanol heat pipe

**Day 10: Performance envelope generation**
- `performance_envelope()` function: sweep temperature, calculate Qmax at each point
- Integrate with fluid property interpolation to update properties at each temperature
- Output: Vec<(T_c, Qmax_w)> for plotting
- Unit tests: generate envelope for water heat pipe from 20°C to 100°C, verify Qmax increases with temperature

**Day 11: Orientation effects and parametric studies**
- Implement gravity term in capillary limit: `ρ_l * g * L * sin(θ)`
- Test gravity-assisted (90°), horizontal (0°), gravity-opposed (-90°) orientations
- Parametric sweep function: vary single parameter (diameter, wick thickness, fluid, etc.), generate output
- Unit tests: verify gravity-opposed reduces Qmax by expected amount

**Day 12: Solver validation against literature**
- Benchmark 1: Water heat pipe, L=200mm, D=6mm, sintered copper wick, 50°C horizontal → compare Qmax to Chi 1976 Fig 3.10
- Benchmark 2: Methanol heat pipe, L=300mm, D=8mm, axial grooves, 25°C vertical (evap down) → compare to Faghri 2016 Table 4.2
- Benchmark 3: Ammonia heat pipe, L=500mm, D=10mm, screen mesh, 0°C horizontal → compare thermal resistance
- Document validation results with <10% error tolerance

### Phase 3 — WASM Build + Frontend Device Editor (Days 13–18)

**Day 13: WASM solver compilation**
- `solver-wasm/` — New workspace member for WASM target
- `solver-wasm/Cargo.toml` — wasm-bindgen, serde-wasm-bindgen dependencies
- `solver-wasm/src/lib.rs` — WASM entry points: `solve_1d_steady_state()`, `calculate_operating_limits()`, `performance_envelope()`
- `wasm-pack build --target web --release` pipeline
- `wasm-opt -Oz` for size optimization (target <500KB gzipped)
- JavaScript wrapper: `HeatPipeSolver` class that loads WASM and provides async API

**Day 14: Frontend scaffold and device geometry editor foundation**
```bash
npm create vite@latest frontend -- --template react-ts
cd frontend
npm i zustand @tanstack/react-query axios konva react-konva
```
- `src/App.tsx` — Router, auth context, layout
- `src/stores/projectStore.ts` — Zustand store for project state
- `src/stores/geometryStore.ts` — Device geometry state
- `src/components/DeviceEditor/Canvas.tsx` — Canvas 2D for device cross-section drawing
- `src/components/DeviceEditor/Grid.tsx` — Grid overlay for dimension reference

**Day 15: Device geometry editor — cylindrical heat pipe**
- `src/components/DeviceEditor/CylindricalEditor.tsx` — Interactive editor for cylindrical heat pipe geometry
- Inputs: total length, evaporator length, condenser length, diameter, wall thickness, wick thickness
- Visual representation: 2D side view with labeled sections (evaporator, adiabatic, condenser)
- Cross-section view: concentric circles showing wall, wick, vapor core
- Real-time dimension validation (wick thickness < (D - wall)/2, lengths add up, etc.)

**Day 16: Wick structure configuration panel**
- `src/components/DeviceEditor/WickPanel.tsx` — Wick type selector and parameter inputs
- Sintered powder: particle diameter, porosity
- Axial grooves: width, depth, spacing, shape (rectangular/triangular)
- Screen mesh: mesh number, wire diameter, number of layers
- Live preview of wick cross-section geometry
- Calculated properties display: permeability, capillary pressure, effective conductivity (computed via WASM)

**Day 17: Working fluid selection and material compatibility**
- `src/components/DeviceEditor/FluidSelector.tsx` — Dropdown with 15 common fluids, filterable by temperature range
- Fluid property preview: temperature range, latent heat, toxicity, flammability, cost
- Material selector: case material (copper, aluminum, stainless steel, titanium)
- Compatibility indicator: green (excellent), yellow (acceptable), red (incompatible) based on database lookup
- Warning messages for incompatible combinations

**Day 18: Operating conditions and simulation setup**
- `src/components/SimulationPanel/ConditionsPanel.tsx` — Operating temperature, orientation angle, heat load
- Temperature slider with fluid valid range indicators
- Orientation selector: horizontal, evaporator down, evaporator up, custom angle
- Analysis type selector: steady-state 1D, performance envelope, parametric sweep
- Run button with plan limit checks (WASM vs. server routing)

### Phase 4 — Results Visualization + API Integration (Days 19–24)

**Day 19: Canvas-based plot rendering**
- `src/components/Plots/PerformanceEnvelopePlot.tsx` — Canvas 2D plot for Qmax vs. T
- Auto-scaling axes with engineering notation (W, kW)
- Grid lines, axis labels, legend
- Hover tooltip showing exact values
- Export as PNG button

**Day 20: Thermal resistance breakdown visualization**
- `src/components/Plots/ThermalResistanceChart.tsx` — Stacked bar chart showing 5 resistance components
- Color-coded: evaporator wall (red), evaporator wick (orange), vapor (yellow), condenser wick (green), condenser wall (blue)
- Percentage labels on each segment
- Total thermal resistance displayed prominently

**Day 21: Operating limits spider chart**
- `src/components/Plots/OperatingLimitsChart.tsx` — Radar/spider chart showing all 5 operating limits
- Normalized scale (0-100%) based on max of all limits
- Limiting factor highlighted in red
- Interactive hover to show exact value for each limit

**Day 22: Simulation API integration**
- `src/hooks/useWasmSolver.ts` — React hook for WASM solver (load + run)
- `src/hooks/useSimulation.ts` — React hook for API-based server simulation
- WASM path: instant results, update UI directly
- Server path: poll status endpoint or subscribe via WebSocket for progress
- Result caching in React Query

**Day 23: Result history and comparison**
- `src/components/ResultsPanel/ResultHistory.tsx` — List of previous simulation results for current project
- Quick summary cards: analysis type, temperature, Qmax, limiting factor
- Click to reload full results and visualizations
- Comparison mode: select 2+ results, overlay plots for side-by-side comparison

**Day 24: Export and reporting**
- `src/components/Export/ExportPanel.tsx` — Export buttons for CSV, PNG, PDF
- CSV export: performance envelope data (T, Qmax), thermal resistance breakdown, operating limits table
- PNG export: all plots as high-res images
- PDF report generation (client-side): device geometry summary, operating conditions, key results, plots
- Uses `jsPDF` and `html2canvas` for PDF generation

### Phase 5 — Server-Side Solver + Job Orchestration (Days 25–29)

**Day 25: Server-side simulation worker**
- `src/workers/simulation_worker.rs` — Redis job consumer, runs native solver
- Load fluid properties from database
- Run 1D solver or 2D solver (placeholder for 2D in MVP, fully implement post-MVP)
- Progress updates via Redis pub/sub
- Store results in S3 as binary arrays (temperature profiles, pressure distributions)

**Day 26: WebSocket for live simulation progress**
- `src/api/ws/mod.rs` — Axum WebSocket upgrade handler
- `src/api/ws/simulation_progress.rs` — Subscribe to simulation progress channel
- Client receives: `{ progress_pct, current_iteration, estimated_time_remaining }`
- Connection management: heartbeat ping/pong, automatic cleanup on disconnect
- Frontend: `src/hooks/useSimulationProgress.ts` — React hook for WebSocket subscription

**Day 27: Parametric sweep execution**
- `src/workers/parametric_sweep.rs` — Execute parametric sweeps server-side
- User specifies: parameter to vary (e.g., diameter), range, number of points
- Worker runs solver N times, collects results
- Output: array of (parameter_value, Qmax, thermal_resistance, limiting_factor)
- Store full sweep results in S3, summary in PostgreSQL

**Day 28: Material compatibility API and recommendations**
- `src/api/handlers/materials.rs` — Get compatibility matrix, get recommendations
- Recommendation endpoint: input operating temperature + case material → ranked list of compatible fluids
- Ranking criteria: merit number (liquid transport factor), compatibility rating, cost
- Frontend: automatic fluid recommendations in device editor

**Day 29: Device template library**
- `src/api/handlers/templates.rs` — List templates, get template, create project from template
- Seed database with 5-10 common device templates:
  - Smartphone cooling heat pipe (copper/water, 100mm, 3mm diameter)
  - Laptop vapor chamber (copper/water, 150mm × 100mm, 0.5mm thick)
  - Satellite heat pipe (aluminum/ammonia, 500mm, 8mm diameter)
  - EV battery thermal link (aluminum/methanol, 300mm, 6mm diameter)
  - LED cooling heat pipe (copper/water, 50mm, 2mm diameter)
- Frontend: template gallery page with preview images and one-click "Use Template"

### Phase 6 — Billing + Plan Enforcement (Days 30–33)

**Day 30: Stripe integration**
- `src/api/handlers/billing.rs` — Create checkout session, create customer portal session, get subscription status
- `src/api/handlers/webhooks/stripe.rs` — Handle subscription events: `checkout.session.completed`, `customer.subscription.updated`, `customer.subscription.deleted`
- Plan mapping:
  - Free: 1D analysis only, performance envelopes, 3 projects, WASM execution only
  - Pro ($99/mo): All 1D features, parametric sweeps, 20 projects, basic 2D vapor chamber (post-MVP)
  - Advanced ($249/user/mo): Full 2D/3D, transient analysis, optimization, API access, unlimited projects

**Day 31: Usage tracking and plan limits**
- `src/middleware/plan_limits.rs` — Middleware checking plan limits before simulation execution
- `src/services/usage.rs` — Track server simulation count per billing period
- Free plan: block 2D analysis, parametric sweeps, server execution
- Pro plan: allow up to 500 server simulations/month
- Advanced plan: unlimited server simulations
- Usage dashboard: `src/api/handlers/usage.rs` — Current period usage, historical usage

**Day 32: Billing UI**
- `frontend/src/pages/Billing.tsx` — Plan comparison table, current plan display, usage meter
- `frontend/src/components/billing/PlanCard.tsx` — Plan feature comparison cards
- `frontend/src/components/billing/UsageMeter.tsx` — Simulation usage bar with threshold indicators
- Upgrade/downgrade flow via Stripe Customer Portal
- Upgrade prompt modals when hitting plan limits

**Day 33: Feature gating**
- Gate 2D vapor chamber analysis behind Pro plan (placeholder for post-MVP)
- Gate parametric sweeps behind Pro plan
- Gate transient analysis behind Advanced plan (post-MVP)
- Gate API access behind Advanced plan
- Show locked feature indicators with upgrade CTAs in UI

### Phase 7 — Validation + Testing (Days 34–38)

**Day 34: Solver validation — literature benchmarks**
- Benchmark 1: Chi 1976 Fig 3.10 — Water heat pipe, verify Qmax within 10%
- Benchmark 2: Faghri 2016 Table 4.2 — Methanol heat pipe with grooves, verify capillary limit within 15%
- Benchmark 3: Peterson 1994 Example 2.3 — Copper-water heat pipe thermal resistance
- Automated test suite: `solver-core/tests/validation.rs` with assertions
- Document validation report with error analysis

**Day 35: Solver validation — orientation and wick types**
- Test gravity-assisted vs. gravity-opposed for same heat pipe (expect 30-50% difference in Qmax)
- Compare sintered powder vs. axial grooves vs. screen mesh for same geometry
- Verify wick property calculations match published correlations (Kozeny-Carman, groove hydraulic radius)
- Performance envelope shape validation (should increase monotonically with temperature up to fluid limit)

**Day 36: Integration testing**
- End-to-end test: create project → configure geometry → select fluid/wick → run simulation → view results → export CSV
- API integration tests: auth → project CRUD → simulation → results retrieval
- WASM solver test: load in headless browser (Playwright), run simulation, verify results match native solver
- WebSocket test: connect → subscribe → receive progress → receive completion
- Concurrent simulation test: submit 5 simultaneous server-side jobs

**Day 37: Performance testing and optimization**
- WASM solver benchmarks: measure time for operating limit calculation, performance envelope generation (50 points)
- Target: operating limits <50ms, performance envelope <500ms in WASM
- Server solver benchmarks: 1D steady-state solve time, parametric sweep (100 points) throughput
- Frontend rendering: measure plot render time with 500-point performance envelope
- Load testing: 20 concurrent users running simulations via k6

**Day 38: UI/UX polish and accessibility**
- Loading states for all async operations (solver running, results loading)
- Error messages with actionable suggestions (e.g., "Wick too thick for vapor core, reduce wick thickness to < X mm")
- Empty states: new project, no simulation history, template gallery
- Responsive layout: device editor and plots work on tablets
- Keyboard navigation: tab through device editor inputs, arrow keys to adjust sliders
- ARIA labels on all interactive elements, screen reader testing

### Phase 8 — Deployment + Launch (Days 39–42)

**Day 39: Docker and Kubernetes configuration**
- `Dockerfile` — Multi-stage Rust build (cargo-chef for layer caching)
- `docker-compose.yml` — Full local dev stack (API, PostgreSQL, Redis, MinIO, frontend)
- `k8s/` — Kubernetes manifests:
  - `api-deployment.yaml` — API server (3 replicas, HPA)
  - `worker-deployment.yaml` — Simulation workers (auto-scaling based on queue depth)
  - `postgres-statefulset.yaml` — PostgreSQL with PVC
  - `redis-deployment.yaml` — Redis for job queue and pub/sub
  - `ingress.yaml` — NGINX ingress with TLS
- Health check endpoints: `/health/live`, `/health/ready`

**Day 40: CDN and WASM delivery**
- CloudFront distribution for:
  - Frontend static assets (HTML, JS, CSS) — cached at edge
  - WASM solver bundle (~500KB) — cached at edge with long TTL and versioned URLs
- WASM preloading: start downloading WASM bundle on page load before user needs it
- Service worker for offline WASM caching (progressive enhancement)
- S3 bucket configuration for simulation results with presigned URL generation

**Day 41: Monitoring, logging, and observability**
- Prometheus metrics: simulation duration histogram, solver convergence rate, API latency percentiles
- Grafana dashboards: system health, simulation throughput, user activity, plan distribution
- Sentry integration: frontend and backend error tracking with source maps
- Structured logging (tracing + JSON) for all API requests and solver events
- Alert rules: high error rate, queue backlog, database connection pool exhaustion

**Day 42: Launch preparation and final testing**
- End-to-end smoke test in production environment
- Database backup and restore verification
- Rate limiting: 60 req/min per user, 10 simulations/min per user
- Security audit: JWT validation, SQL injection (SQLx prevents this), CORS configuration, CSP headers
- Landing page with product overview, use cases, pricing, and signup
- Documentation: getting started guide, heat pipe design basics, API reference
- Deploy to production, enable monitoring alerts, soft launch to beta users

---

## Critical Files

```
heatpipe/
├── solver-core/                           # Shared solver library (native + WASM)
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── heat_pipe_1d.rs                # 1D heat pipe solver
│   │   ├── fluid_properties.rs            # Fluid property interpolation
│   │   ├── wick_properties.rs             # Wick permeability, capillary pressure
│   │   ├── materials.rs                   # Case material properties
│   │   └── utils.rs                       # Math utilities
│   └── tests/
│       ├── validation.rs                  # Solver validation benchmarks
│       └── wick_tests.rs                  # Wick property tests
│
├── solver-wasm/                           # WASM compilation target
│   ├── Cargo.toml
│   └── src/
│       └── lib.rs                         # WASM entry points (wasm-bindgen)
│
├── heatpipe-api/                          # Rust API server
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
│   │   │   │   ├── fluids.rs              # Fluid database queries
│   │   │   │   ├── materials.rs           # Material compatibility
│   │   │   │   ├── templates.rs           # Device templates
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
│   │   │   └── s3.rs                      # S3 client helpers
│   │   └── workers/
│   │       ├── mod.rs                     # Worker pool management
│   │       ├── simulation_worker.rs       # Server-side simulation execution
│   │       └── parametric_sweep.rs        # Parametric sweep execution
│   ├── migrations/
│   │   └── 001_initial.sql                # Full database schema
│   └── tests/
│       ├── api_integration.rs             # API integration tests
│       └── simulation_e2e.rs              # End-to-end simulation tests
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
│   │   │   ├── geometryStore.ts           # Device geometry state
│   │   │   └── simulationStore.ts         # Simulation results state
│   │   ├── hooks/
│   │   │   ├── useSimulationProgress.ts   # WebSocket hook for live progress
│   │   │   ├── useWasmSolver.ts           # WASM solver hook (load + run)
│   │   │   └── useFluidProperties.ts      # Fluid database queries
│   │   ├── lib/
│   │   │   ├── api.ts                     # Axios API client
│   │   │   └── wasmLoader.ts              # WASM bundle loading
│   │   ├── pages/
│   │   │   ├── Dashboard.tsx              # Project list
│   │   │   ├── Editor.tsx                 # Main editor (geometry + results split)
│   │   │   ├── Templates.tsx              # Device template gallery
│   │   │   ├── Billing.tsx                # Plan management
│   │   │   ├── Login.tsx                  # Auth pages
│   │   │   └── Register.tsx
│   │   ├── components/
│   │   │   ├── DeviceEditor/
│   │   │   │   ├── Canvas.tsx             # Canvas 2D device visualization
│   │   │   │   ├── CylindricalEditor.tsx  # Cylindrical geometry editor
│   │   │   │   ├── WickPanel.tsx          # Wick configuration
│   │   │   │   ├── FluidSelector.tsx      # Fluid selection + compatibility
│   │   │   │   └── ConditionsPanel.tsx    # Operating conditions
│   │   │   ├── Plots/
│   │   │   │   ├── PerformanceEnvelopePlot.tsx  # Qmax vs T plot
│   │   │   │   ├── ThermalResistanceChart.tsx   # Resistance breakdown
│   │   │   │   └── OperatingLimitsChart.tsx     # Spider chart
│   │   │   ├── ResultsPanel/
│   │   │   │   ├── ResultHistory.tsx      # Simulation history
│   │   │   │   └── ComparisonView.tsx     # Side-by-side comparison
│   │   │   ├── Export/
│   │   │   │   └── ExportPanel.tsx        # CSV/PNG/PDF export
│   │   │   ├── billing/
│   │   │   │   ├── PlanCard.tsx
│   │   │   │   └── UsageMeter.tsx
│   │   │   └── common/
│   │   │       └── SimulationToolbar.tsx  # Run/stop/analysis selector
│   │   └── data/
│   │       └── templates/                 # Pre-built device templates (JSON)
│   └── public/
│       └── wasm/                          # WASM solver bundle (loaded at runtime)
│
├── ml-service/                            # Python FastAPI (optimization, ML)
│   ├── requirements.txt
│   ├── main.py                            # FastAPI app
│   ├── optimization/
│   │   ├── parametric_sweep.py            # Parametric sweep orchestration
│   │   └── multi_objective.py             # Multi-objective optimization (post-MVP)
│   └── Dockerfile
│
├── k8s/                                   # Kubernetes manifests
│   ├── api-deployment.yaml
│   ├── worker-deployment.yaml
│   ├── postgres-statefulset.yaml
│   ├── redis-deployment.yaml
│   ├── ml-service-deployment.yaml
│   └── ingress.yaml
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

### Benchmark 1: Water Heat Pipe — Capillary Limit (1D Steady-State)

**Circuit:** Cylindrical copper heat pipe with sintered copper wick
- Geometry: L=200mm, D=6mm, t_wall=0.5mm, t_wick=0.5mm
- Wick: Sintered copper powder, d_particle=50µm, porosity=0.5
- Fluid: Water at T=50°C
- Orientation: Horizontal (0°)

**Expected (Chi 1976 Fig 3.10):** Q_cap ≈ **35-40 W**

**Tolerance:** < 15% (heat pipe correlations have inherent uncertainty)

### Benchmark 2: Methanol Heat Pipe — Axial Groove (Gravity-Assisted)

**Circuit:** Aluminum heat pipe with rectangular axial grooves
- Geometry: L=300mm, D=8mm, grooves: 12 grooves, w=0.5mm, d=0.5mm
- Fluid: Methanol at T=25°C
- Orientation: Vertical, evaporator down (90°)

**Expected (Faghri 2016 Table 4.2):** Q_cap ≈ **55 W** (gravity-assisted)

**Tolerance:** < 20% (groove geometry approximations introduce variation)

### Benchmark 3: Ammonia Heat Pipe — Screen Mesh (Thermal Resistance)

**Circuit:** Stainless steel heat pipe with stainless steel screen mesh
- Geometry: L=500mm, D=10mm, mesh: 100 mesh, 4 layers
- Fluid: Ammonia at T=0°C
- Heat load: Q=20W

**Expected (Peterson 1994 Example 2.3):** R_thermal ≈ **0.4-0.5 K/W**

**Tolerance:** < 15% (thermal resistance measurements have ±10% experimental error)

### Benchmark 4: Orientation Effects — Gravity-Opposed vs. Horizontal

**Circuit:** Water heat pipe, L=200mm, D=6mm, sintered wick
- Fluid: Water at T=50°C
- Test 1: Horizontal (0°) → Q_cap = Q_ref
- Test 2: Gravity-opposed (-90°, evaporator up) → expect Q_cap ≈ **0.6 × Q_ref**

**Expected:** 30-50% reduction in Q_cap for gravity-opposed orientation

**Tolerance:** < 10% of expected reduction

### Benchmark 5: Performance Envelope Shape — Water 20-100°C

**Circuit:** Water heat pipe, standard geometry
- Temperature range: 20°C to 100°C (50 points)

**Expected behavior:**
- Q_cap increases monotonically with T (higher vapor pressure → lower viscous limit dominance)
- Q_entrainment increases with T (higher surface tension at low T)
- At low T (<40°C): viscous limit may dominate
- At high T (>70°C): capillary or entrainment limit dominates

**Tolerance:** Monotonicity must hold, no inflection points except at limit transitions

---

## Verification

### Integration Testing Checklist

1. **Auth flow** — Register → login → JWT issued → authorized API calls → token refresh → logout
2. **Project CRUD** — Create → update geometry → save → reload → verify geometry state preserved
3. **Device editing** — Configure cylindrical heat pipe → set wick parameters → select fluid → validate compatibility
4. **WASM simulation** — 1D heat pipe → WASM solver → operating limits calculated → results displayed (<1s)
5. **Server simulation** — Parametric sweep (100 points) → job queued → worker picks up → WebSocket progress → results in S3
6. **Results visualization** — Performance envelope plot → thermal resistance chart → operating limits spider chart
7. **Fluid database** — Search "water" → properties returned → temperature range validated
8. **Material compatibility** — Copper case + water → "excellent" rating → aluminum + acetone → warning
9. **Template usage** — Select "Smartphone cooling" template → project created → simulate → expected Qmax
10. **Plan limits** — Free user → parametric sweep → blocked with upgrade prompt
11. **Billing** — Subscribe to Pro → Stripe checkout → webhook → plan updated → limits lifted
12. **Concurrent users** — 10 users simultaneously running WASM simulations → all complete correctly
13. **Export** — Run simulation → export CSV → verify data format → export PDF → verify plots rendered
14. **Error handling** — Invalid geometry (wick too thick) → meaningful error message → no crash
15. **Orientation effects** — Horizontal vs. gravity-opposed → verify 30-50% Qmax reduction

### SQL Verification Queries

```sql
-- 1. Simulation throughput and success rate
SELECT
    execution_mode,
    status,
    COUNT(*) as count,
    AVG(wall_time_ms)::int as avg_time_ms,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY wall_time_ms)::int as p95_time_ms
FROM simulations
WHERE created_at >= NOW() - INTERVAL '7 days'
GROUP BY execution_mode, status;

-- 2. Analysis type distribution
SELECT analysis_type, COUNT(*) as count,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 1) as pct
FROM simulations
WHERE created_at >= NOW() - INTERVAL '30 days'
GROUP BY analysis_type ORDER BY count DESC;

-- 3. User plan distribution and conversion
SELECT plan, COUNT(*) as users,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 1) as pct
FROM users GROUP BY plan ORDER BY users DESC;

-- 4. Server simulation usage by user (billing period)
SELECT u.email, u.plan,
    SUM(ur.quantity) as total_simulations,
    CASE u.plan
        WHEN 'free' THEN 0
        WHEN 'pro' THEN 500
        WHEN 'advanced' THEN 999999
    END as limit_simulations
FROM usage_records ur
JOIN users u ON ur.user_id = u.id
WHERE ur.period_start <= CURRENT_DATE AND ur.period_end >= CURRENT_DATE
    AND ur.record_type = 'simulation_count'
GROUP BY u.email, u.plan
ORDER BY total_simulations DESC;

-- 5. Fluid popularity
SELECT wf.name, wf.chemical_formula,
    COUNT(DISTINCT s.project_id) as projects_using,
    COUNT(s.id) as total_simulations
FROM working_fluids wf
LEFT JOIN simulations s ON s.working_fluid = wf.name
WHERE s.created_at >= NOW() - INTERVAL '30 days'
GROUP BY wf.id
ORDER BY total_simulations DESC
LIMIT 10;
```

### Performance Benchmarks

| Operation | Target | Method |
|-----------|--------|--------|
| WASM solver: Operating limits (1D) | <50ms | Browser benchmark (Chrome DevTools) |
| WASM solver: Performance envelope (50 pts) | <500ms | Browser benchmark |
| Server solver: Parametric sweep (100 pts) | <30s | Server timing, 2 cores |
| Frontend: Performance envelope plot render | <100ms | React profiler |
| Frontend: WASM bundle load (cached) | <200ms | Service worker cache hit |
| Frontend: WASM bundle load (cold) | <1.5s | CDN delivery, 500KB gzipped |
| API: Create simulation | <100ms | p95 latency (Prometheus) |
| API: Get fluid properties | <50ms | p95 latency with 15 fluids in DB |
| WebSocket: Progress latency | <100ms | Time from worker emit to client receive |
| Wick property calculation | <10ms | Rust solver (all 3 wick types) |

---

## Deployment Architecture

### Infrastructure

```
┌─────────────────────────────────────────────────┐
│                  CloudFront CDN                   │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────┐ │
│  │ Frontend SPA │  │ WASM Bundle │  │ Fluid    │ │
│  │ (HTML/JS/CSS)│  │ (versioned) │  │ Data     │ │
│  └─────────────┘  └─────────────┘  └──────────┘ │
└─────────────────────────┬───────────────────────┘
                          │
              ┌───────────┴───────────┐
              │   AWS ALB (HTTPS)     │
              │   TLS termination     │
              └───────────┬───────────┘
                          │
        ┌─────────────────┼─────────────────┐
        ▼                 ▼                 ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ API Server   │  │ API Server   │  │ API Server   │
│ (Rust/Axum)  │  │ (Rust/Axum)  │  │ (Rust/Axum)  │
│ Pod ×3 (HPA) │  │              │  │              │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘
       │                 │                 │
       └─────────────────┼─────────────────┘
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ PostgreSQL   │ │ Redis        │ │ S3 Bucket    │
│ (RDS r6g)    │ │ (ElastiCache)│ │ (Results +   │
│ Multi-AZ     │ │ Cluster mode │ │  Data)       │
└──────────────┘ └──────┬───────┘ └──────────────┘
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ Sim Worker   │ │ Sim Worker   │ │ Sim Worker   │
│ (Rust native)│ │ (Rust native)│ │ (Rust native)│
│ c7g.xlarge   │ │ c7g.xlarge   │ │ c7g.xlarge   │
│ HPA: 1-10    │ │              │ │              │
└──────────────┘ └──────────────┘ └──────────────┘
```

### Scaling Strategy

- **API servers**: Horizontal pod autoscaler (HPA) at 70% CPU, min 3, max 10
- **Simulation workers**: HPA based on Redis queue depth — scale up when queue > 3 jobs, scale down when idle 5 min
- **Worker instances**: AWS c7g.xlarge (4 vCPU, 8GB RAM) for compute-optimized thermal simulation
- **Database**: RDS PostgreSQL r6g.large with read replica for fluid property queries
- **Redis**: ElastiCache r6g.large cluster with 1 primary + 1 replica

### S3 Lifecycle Policy

```json
{
  "Rules": [
    {
      "ID": "simulation-results-lifecycle",
      "Prefix": "results/",
      "Status": "Enabled",
      "Transitions": [
        { "Days": 30, "StorageClass": "STANDARD_IA" },
        { "Days": 90, "StorageClass": "GLACIER" }
      ],
      "Expiration": { "Days": 365 }
    },
    {
      "ID": "fluid-data-keep",
      "Prefix": "fluids/",
      "Status": "Enabled",
      "Transitions": [
        { "Days": 365, "StorageClass": "STANDARD_IA" }
      ]
    },
    {
      "ID": "wasm-bundles-keep",
      "Prefix": "wasm/",
      "Status": "Enabled",
      "NoncurrentVersionExpiration": { "NoncurrentDays": 30 }
    }
  ]
}
```

---

## Post-MVP Roadmap

| Feature | Description | Priority |
|---------|-------------|----------|
| 2D Vapor Chamber Simulation | Full 2D finite-element thermal solver for vapor chambers with thin-film wick models, support column placement optimization, and hotspot spreading analysis with temperature contour visualization | High |
| Transient Analysis | Startup transient simulation from frozen state to steady-state operation, pulsed heat load response, thermal mass effects, and wick dryout/rewet dynamics | High |
| Loop Heat Pipe Modeling | Complete LHP simulation with compensation chamber, primary wick, vapor line, condenser, and liquid return line including startup overshoot prediction and depriming scenarios | High |
| Multi-Objective Optimization | Genetic algorithm optimization to maximize Qmax while minimizing weight, volume, or thermal resistance subject to geometric and manufacturing constraints | Medium |
| Advanced Wick Structures | Composite/hybrid wick models (bi-porous structures), arterial wicks, and user-defined wick correlations with experimental characterization data import | Medium |
| Variable Conductance Heat Pipe | VCHP modeling with non-condensable gas (NCG) reservoir for active temperature control, gas inventory optimization, and control curve generation | Medium |
| Pulsating Heat Pipe | 1D slug-plug flow model for pulsating/oscillating heat pipes with frequency prediction and multi-turn configuration optimization | Low |
| Cryogenic and High-Temperature Fluids | Extended fluid database with cryogenic fluids (nitrogen, neon, helium) and high-temperature alkali metals (sodium, potassium, lithium, cesium) | Low |
| Manufacturing Integration | DXF/STEP export for heat pipe mechanical design, wick manufacturing specification generation, and fill charge calculation with vacuum bakeout procedures | Low |