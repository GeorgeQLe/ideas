# 55. SeismiQ — Seismic Analysis and Earthquake Engineering Platform

## Implementation Plan

**MVP Scope:** Browser-based 3D structural model builder with visual grid-based placement and automatic member generation for steel/concrete moment frames (AISC/ACI sections), custom finite element analysis (FEA) solver in Rust implementing direct stiffness method with sparse matrix solving via SuperLU compiled to WebAssembly for small models (≤100 elements) and server-side Rust-native execution for larger structures, site-specific probabilistic seismic hazard analysis (PSHA) using USGS National Seismic Hazard Model APIs and global ground motion prediction equations (GMPEs), automated ASCE 7-22/IBC 2024 seismic design with equivalent lateral force (ELF) procedure and response spectrum analysis (RSA) with CQC modal combination, interactive 3D structural visualization via Three.js with deformed shape and mode shape rendering, ASCE 7 code compliance checking for seismic design category, R-factors, drift limits, and irregularities, PDF engineering report generation with code references and calculations, Stripe billing with three tiers (Free / Professional $149/mo / Advanced $349/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| FEA Solver | Rust (native + WASM) | Custom 3D frame/shell FEA with direct stiffness method, SuperLU sparse solver |
| WASM Compilation | `wasm-pack` + `wasm-bindgen` | Client-side solver for structures ≤100 elements |
| PSHA Engine | Rust (native + WASM) | Hazard integration, GMPE evaluation, OpenQuake compatibility |
| Site Response | Rust (native) | 1D equivalent linear SHAKE method + nonlinear Newmark integration |
| Loss Estimation | Python 3.12 (FastAPI) | FEMA P-58 fragility convolution, Monte Carlo simulation |
| Database | PostgreSQL 16 + PostGIS | Projects, earthquake catalogs, building codes, hazard curves, fragility data |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | Ground motion records, FEA results, hazard curves, fragility data |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| 3D Visualization | Three.js + React-Three-Fiber | GPU-accelerated 3D structural rendering, deformed shapes, mode shapes |
| Chart Library | D3.js | Hazard curves, response spectra, fragility curves, IDA plots |
| Real-time | WebSocket (Axum) | Live analysis progress, convergence monitoring for nonlinear analysis |
| Job Queue | Redis 7 + Tokio tasks | Server-side analysis job management, cloud distribution for time-history suites |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | WASM bundle delivery, static frontend assets, ground motion records |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, error tracking |
| CI/CD | GitHub Actions | WASM build pipeline, Rust tests, Docker image push |

### Key Architecture Decisions

1. **Hybrid client/server FEA solver with WASM threshold at 100 elements**: Structures with ≤100 frame elements (covers 80%+ of low-rise buildings and educational examples) run entirely in the browser via WASM, providing instant modal analysis and response spectrum results with zero server cost. Structures exceeding 100 elements or requiring nonlinear analysis are submitted to the Rust-native server solver which handles multi-thousand-element models and distributed time-history analysis. The threshold is configurable per plan tier.

2. **Custom FEA engine in Rust rather than wrapping OpenSees**: Building a custom 3D frame FEA solver in Rust gives us full control over element formulations, WASM compilation, and parallelization for parameter studies. OpenSees is C++ with extensive global state that makes WASM compilation impossible and thread-safety problematic. Our Rust solver uses SuperLU sparse direct solver (same algorithm used by commercial FEA) while maintaining memory safety and WASM compatibility. For nonlinear analysis (Post-MVP), we'll implement fiber sections and plastic hinges in Rust.

3. **Integrated PSHA engine with USGS API fallback**: We implement PSHA calculations in Rust (hazard integration over fault sources + GMPE evaluation) for full control and WASM compatibility, allowing site hazard analysis to run client-side for instant feedback. For regions with USGS National Seismic Hazard Model coverage, we provide API integration as a fast path. This dual approach supports global projects while leveraging authoritative US data where available.

4. **Three.js for 3D structural visualization with GPU instancing**: Three.js provides mature WebGL rendering with excellent React integration via React-Three-Fiber. We use GPU instancing for efficient rendering of thousands of structural members, animated deformed shapes with color-coded stress/displacement, and interactive mode shape visualization. Canvas 2D was rejected due to lack of 3D perspective and poor performance with complex structures.

5. **PostgreSQL + PostGIS for spatial hazard queries and earthquake catalog**: PostGIS enables efficient spatial queries for finding nearby earthquake faults, selecting ground motion records by distance-magnitude bins (for hazard deaggregation), and portfolio risk assessment with building location queries. Earthquake catalogs (USGS ANSS, global GCMT) are stored with geometry indexes for fast radius searches. Ground motion metadata is in PostgreSQL while waveform time series are in S3.

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "postgis";  -- Spatial queries for seismic sources
CREATE EXTENSION IF NOT EXISTS "pg_trgm";  -- Fuzzy text search on building codes

-- Users
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email TEXT UNIQUE NOT NULL,
    password_hash TEXT,  -- NULL for OAuth-only users
    name TEXT NOT NULL,
    avatar_url TEXT,
    auth_provider TEXT DEFAULT 'email',  -- email | google | github
    auth_provider_id TEXT,
    plan TEXT NOT NULL DEFAULT 'free',  -- free | professional | advanced
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX users_email_idx ON users(email);
CREATE INDEX users_stripe_idx ON users(stripe_customer_id);

-- Organizations (for team collaboration — Post-MVP)
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

-- Projects (structural model + analysis workspace)
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    project_type TEXT NOT NULL DEFAULT 'building',  -- building | bridge | tower | portfolio
    model_data JSONB NOT NULL DEFAULT '{}',  -- Full structural model (nodes, elements, loads, constraints)
    settings JSONB DEFAULT '{}',  -- Units, code edition, analysis options
    site_location GEOGRAPHY(POINT),  -- Site coordinates for PSHA
    site_class TEXT,  -- ASCE 7 site class: A | B | C | D | E
    seismic_design_category TEXT,  -- SDC: A | B | C | D | E | F
    is_public BOOLEAN NOT NULL DEFAULT false,
    forked_from UUID REFERENCES projects(id) ON DELETE SET NULL,
    thumbnail_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);
CREATE INDEX projects_org_idx ON projects(org_id);
CREATE INDEX projects_updated_idx ON projects(updated_at DESC);
CREATE INDEX projects_location_idx ON projects USING GIST(site_location);

-- Structural Analysis Runs
CREATE TABLE analyses (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    analysis_type TEXT NOT NULL,  -- modal | elf | rsa | pushover | time_history
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | running | completed | failed | cancelled
    execution_mode TEXT NOT NULL DEFAULT 'wasm',  -- wasm | server
    model_snapshot JSONB NOT NULL,  -- Model state at analysis time
    parameters JSONB NOT NULL DEFAULT '{}',  -- Analysis-specific parameters
    element_count INTEGER NOT NULL DEFAULT 0,
    node_count INTEGER NOT NULL DEFAULT 0,
    results_url TEXT,  -- S3 URL for result data
    results_summary JSONB,  -- Quick-access summary (periods, mode shapes, drift ratios)
    error_message TEXT,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    wall_time_ms INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX analyses_project_idx ON analyses(project_id);
CREATE INDEX analyses_user_idx ON analyses(user_id);
CREATE INDEX analyses_status_idx ON analyses(status);
CREATE INDEX analyses_created_idx ON analyses(created_at DESC);

-- Server Analysis Jobs (for server-side execution only)
CREATE TABLE analysis_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    analysis_id UUID NOT NULL REFERENCES analyses(id) ON DELETE CASCADE,
    worker_id TEXT,
    cores_allocated INTEGER DEFAULT 4,
    memory_mb INTEGER DEFAULT 8192,
    priority INTEGER DEFAULT 0,  -- Higher = more urgent
    progress_pct REAL DEFAULT 0.0,
    convergence_data JSONB DEFAULT '[]',  -- [{iteration, residual, displacement_norm}]
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX jobs_analysis_idx ON analysis_jobs(analysis_id);
CREATE INDEX jobs_worker_idx ON analysis_jobs(worker_id);

-- Seismic Hazard Analysis Results
CREATE TABLE hazard_analyses (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    site_location GEOGRAPHY(POINT) NOT NULL,
    site_class TEXT NOT NULL,
    analysis_type TEXT NOT NULL,  -- psha | dsha | site_response
    gmpe_models TEXT[] NOT NULL,  -- Ground motion prediction equations used
    return_periods INTEGER[] DEFAULT '{475, 975, 2475}',  -- Years
    results_url TEXT,  -- S3 URL for hazard curves, spectra
    design_spectrum JSONB,  -- ASCE 7 design spectrum (periods, Sa values)
    deaggregation JSONB,  -- Hazard deaggregation (M-R bins)
    status TEXT NOT NULL DEFAULT 'pending',
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    wall_time_ms INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX hazard_project_idx ON hazard_analyses(project_id);
CREATE INDEX hazard_location_idx ON hazard_analyses USING GIST(site_location);

-- Earthquake Fault Sources (for PSHA calculations)
CREATE TABLE fault_sources (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    fault_type TEXT NOT NULL,  -- active | potentially_active | blind_thrust | subduction
    geometry GEOGRAPHY(LINESTRING) NOT NULL,  -- Fault trace
    dip REAL,  -- Dip angle (degrees)
    rake REAL,  -- Rake angle (degrees)
    slip_rate REAL,  -- mm/year
    max_magnitude REAL NOT NULL,
    recurrence_params JSONB,  -- Gutenberg-Richter a-b values or characteristic model
    source_region TEXT,  -- us_california | us_pnw | nz | japan | global
    data_source TEXT,  -- USGS NSHM | GNS NZ | custom
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX faults_geometry_idx ON fault_sources USING GIST(geometry);
CREATE INDEX faults_region_idx ON fault_sources(source_region);

-- Ground Motion Records (metadata only; waveforms in S3)
CREATE TABLE ground_motion_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    event_id TEXT NOT NULL,  -- USGS or database event ID
    station_code TEXT NOT NULL,
    event_magnitude REAL NOT NULL,
    event_location GEOGRAPHY(POINT) NOT NULL,
    station_location GEOGRAPHY(POINT) NOT NULL,
    rupture_distance_km REAL,  -- Closest distance to rupture
    vs30 REAL,  -- Site shear wave velocity (m/s)
    pga REAL,  -- Peak ground acceleration (g)
    pgv REAL,  -- Peak ground velocity (cm/s)
    duration REAL,  -- Significant duration (s)
    waveform_url TEXT NOT NULL,  -- S3 URL to acceleration time series
    components TEXT[] DEFAULT '{"H1", "H2", "V"}',  -- Available components
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX gm_event_idx ON ground_motion_records(event_id);
CREATE INDEX gm_location_idx ON ground_motion_records USING GIST(event_location);
CREATE INDEX gm_magnitude_idx ON ground_motion_records(event_magnitude);
CREATE INDEX gm_vs30_idx ON ground_motion_records(vs30);

-- Building Code Provisions (ASCE 7, Eurocode 8, etc.)
CREATE TABLE code_provisions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    code_name TEXT NOT NULL,  -- ASCE_7_22 | IBC_2024 | EUROCODE_8_2004 | NZS_1170_5
    edition TEXT NOT NULL,
    region TEXT,  -- Applicable region/country
    provision_type TEXT NOT NULL,  -- seismic_parameters | load_combinations | drift_limits | detailing
    provision_data JSONB NOT NULL,  -- Structured code data (tables, formulas, limits)
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX code_name_idx ON code_provisions(code_name, edition);

-- Fragility Curves (for FEMA P-58 loss estimation — Post-MVP)
CREATE TABLE fragility_curves (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    component_id TEXT NOT NULL,  -- FEMA P-58 component ID
    component_name TEXT NOT NULL,
    description TEXT,
    damage_states JSONB NOT NULL,  -- [{ds_id, median, beta, repair_cost, downtime}]
    edp_type TEXT NOT NULL,  -- Engineering demand parameter: drift | acceleration | force
    structural_system TEXT,  -- moment_frame | shear_wall | braced_frame
    material TEXT,  -- concrete | steel | wood | masonry
    source TEXT DEFAULT 'FEMA_P58',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX fragility_component_idx ON fragility_curves(component_id);
CREATE INDEX fragility_system_idx ON fragility_curves(structural_system, material);

-- Usage Tracking (for billing enforcement)
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,  -- analysis_minutes | cloud_time_history | storage_bytes
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
    pub project_type: String,
    pub model_data: serde_json::Value,
    pub settings: serde_json::Value,
    #[sqlx(skip)]  // PostGIS geometry handling via custom type
    pub site_location: Option<GeoPoint>,
    pub site_class: Option<String>,
    pub seismic_design_category: Option<String>,
    pub is_public: bool,
    pub forked_from: Option<Uuid>,
    pub thumbnail_url: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Analysis {
    pub id: Uuid,
    pub project_id: Uuid,
    pub user_id: Uuid,
    pub analysis_type: String,
    pub status: String,
    pub execution_mode: String,
    pub model_snapshot: serde_json::Value,
    pub parameters: serde_json::Value,
    pub element_count: i32,
    pub node_count: i32,
    pub results_url: Option<String>,
    pub results_summary: Option<serde_json::Value>,
    pub error_message: Option<String>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub wall_time_ms: Option<i32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct AnalysisJob {
    pub id: Uuid,
    pub analysis_id: Uuid,
    pub worker_id: Option<String>,
    pub cores_allocated: i32,
    pub memory_mb: i32,
    pub priority: i32,
    pub progress_pct: f32,
    pub convergence_data: serde_json::Value,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct HazardAnalysis {
    pub id: Uuid,
    pub project_id: Uuid,
    pub user_id: Uuid,
    #[sqlx(skip)]
    pub site_location: GeoPoint,
    pub site_class: String,
    pub analysis_type: String,
    pub gmpe_models: Vec<String>,
    pub return_periods: Vec<i32>,
    pub results_url: Option<String>,
    pub design_spectrum: Option<serde_json::Value>,
    pub deaggregation: Option<serde_json::Value>,
    pub status: String,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub wall_time_ms: Option<i32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct FaultSource {
    pub id: Uuid,
    pub name: String,
    pub fault_type: String,
    #[sqlx(skip)]
    pub geometry: LineString,
    pub dip: Option<f64>,
    pub rake: Option<f64>,
    pub slip_rate: Option<f64>,
    pub max_magnitude: f64,
    pub recurrence_params: serde_json::Value,
    pub source_region: String,
    pub data_source: String,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct GroundMotionRecord {
    pub id: Uuid,
    pub event_id: String,
    pub station_code: String,
    pub event_magnitude: f64,
    #[sqlx(skip)]
    pub event_location: GeoPoint,
    #[sqlx(skip)]
    pub station_location: GeoPoint,
    pub rupture_distance_km: Option<f64>,
    pub vs30: Option<f64>,
    pub pga: Option<f64>,
    pub pgv: Option<f64>,
    pub duration: Option<f64>,
    pub waveform_url: String,
    pub components: Vec<String>,
    pub metadata: serde_json::Value,
    pub created_at: DateTime<Utc>,
}

// Custom types for PostGIS geometry
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct GeoPoint {
    pub lat: f64,
    pub lon: f64,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct LineString {
    pub coordinates: Vec<GeoPoint>,
}

// Analysis parameter types
#[derive(Debug, Deserialize, Serialize)]
pub struct AnalysisParams {
    pub analysis_type: AnalysisType,
    pub modal: Option<ModalParams>,
    pub elf: Option<ElfParams>,
    pub rsa: Option<RsaParams>,
    pub time_history: Option<TimeHistoryParams>,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum AnalysisType {
    Modal,
    Elf,           // Equivalent Lateral Force
    Rsa,           // Response Spectrum Analysis
    Pushover,      // Nonlinear static pushover
    TimeHistory,   // Dynamic time-history analysis
}

#[derive(Debug, Deserialize, Serialize)]
pub struct ModalParams {
    pub num_modes: u32,  // Number of modes to extract (default 12)
    pub include_mass_participation: bool,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct ElfParams {
    pub code: String,  // "ASCE_7_22" | "IBC_2024" | "EUROCODE_8"
    pub importance_factor: f64,  // Ie (typically 1.0, 1.25, or 1.5)
    pub response_modification_factor: f64,  // R factor based on system type
    pub load_direction: String,  // "X" | "Y" | "both"
}

#[derive(Debug, Deserialize, Serialize)]
pub struct RsaParams {
    pub spectrum_url: String,  // S3 URL to design spectrum or hazard result
    pub damping_ratio: f64,  // Typically 0.05 (5%)
    pub combination_method: String,  // "CQC" | "SRSS"
    pub scale_factor: f64,  // Scale spectrum ordinates
}

#[derive(Debug, Deserialize, Serialize)]
pub struct TimeHistoryParams {
    pub ground_motion_ids: Vec<Uuid>,  // Ground motion record IDs
    pub duration: f64,  // Analysis duration (seconds)
    pub timestep: f64,  // Integration timestep (seconds)
    pub damping_ratio: f64,
    pub integration_method: String,  // "newmark" | "hht_alpha"
}
```

---

## FEA Solver Architecture Deep-Dive

### Governing Equations and Discretization

SeismiQ's core FEA solver implements the **Direct Stiffness Method** for 3D frame structures. For a structure with `n` degrees of freedom (DOFs), the static equilibrium equation is:

```
K · u = F

where:
  K (n×n): Global stiffness matrix (symmetric, sparse)
  u (n):   Unknown displacement vector (nodal translations and rotations)
  F (n):   Applied force vector (loads + reactions)
```

**Modal analysis** solves the generalized eigenvalue problem for natural frequencies ωᵢ and mode shapes φᵢ:

```
(K - ωᵢ² M) φᵢ = 0

where:
  M (n×n): Global mass matrix (consistent or lumped)
  ωᵢ:      Circular frequency of mode i (rad/s)
  φᵢ (n):  Mode shape i (normalized displacement pattern)
```

**Response spectrum analysis** computes maximum modal responses using the design spectrum S(T, ζ) and combines them:

```
uᵢ,max = Γᵢ · S(Tᵢ, ζ) · φᵢ     (maximum response for mode i)

u_total = √(Σᵢ Σⱼ ρᵢⱼ · uᵢ,max · uⱼ,max)    (CQC modal combination)

where:
  Γᵢ = φᵢᵀ M r / (φᵢᵀ M φᵢ)  (modal participation factor)
  r:   Influence vector (1 for earthquake direction, 0 otherwise)
  Tᵢ:  Period of mode i = 2π/ωᵢ
  ρᵢⱼ: Cross-correlation coefficient (for CQC combination)
```

**Dynamic time-history analysis** uses Newmark-β time integration:

```
M ü(t) + C u̇(t) + K u(t) = F(t) + M r üg(t)

where:
  üg(t): Ground acceleration time series
  C:     Rayleigh damping matrix C = α M + β K

Newmark-β predictor-corrector:
  u_{n+1} = u_n + Δt u̇_n + (Δt²/2)[(1-2β)ü_n + 2β ü_{n+1}]
  u̇_{n+1} = u̇_n + Δt[(1-γ)ü_n + γ ü_{n+1}]

  Typical: β = 0.25, γ = 0.5 (average acceleration, unconditionally stable)
```

### Element Formulation — 3D Beam-Column (Elastic)

Each beam-column element has 12 DOFs (6 per node: 3 translations + 3 rotations). The local element stiffness matrix in the element coordinate system is:

```
k_local =
  ┌                                                                          ┐
  │  EA/L      0         0         0         0         0      -EA/L  ...    │
  │  0      12EI_z/L³  0         0         0      6EI_z/L²     0     ...    │
  │  0         0      12EI_y/L³  0     -6EI_y/L²    0          0     ...    │
  │  0         0         0      GJ/L       0         0          0     ...    │
  │  0         0     -6EI_y/L²   0      4EI_y/L      0          0     ...    │
  │  0      6EI_z/L²    0         0         0      4EI_z/L      0     ...    │
  │ -EA/L      0         0         0         0         0       EA/L   ...    │
  │  ...                                                         (symmetric)  │
  └                                                                          ┘

where:
  E: Young's modulus
  A: Cross-sectional area
  I_y, I_z: Moments of inertia about local y and z axes
  G: Shear modulus
  J: Torsional constant
  L: Element length
```

The element stiffness is transformed to global coordinates via rotation matrix T:

```
K_element = Tᵀ k_local T

where T is a 12×12 block-diagonal transformation based on element orientation.
```

### Assembly and Solution Strategy

```
Element loop → Stamp K_element into global K (CSR sparse format)
              → Stamp M_element into global M (consistent or lumped)
              → Apply boundary conditions (zero rows/cols for fixed DOFs)
              → Solve K u = F using SuperLU sparse direct solver (LU factorization)

For modal analysis:
  → Solve (K - λ M) φ = 0 using Lanczos or subspace iteration (ARPACK bindings)
  → Extract first N eigenvalues/eigenvectors (N typically 12-30 modes)
  → Normalize mode shapes to unit modal mass

For response spectrum:
  → Extract modal data (periods, shapes, participation factors)
  → For each mode: compute spectral displacement from S(T, ζ)
  → Combine modal maxima using CQC or SRSS
  → Compute element forces from combined displacements

For time-history:
  → Form effective stiffness K_eff = K + a₀ M + a₁ C (Newmark constants)
  → Factor K_eff once (reused for all timesteps if linear)
  → At each timestep: update load vector, solve for ü_{n+1}, update u and u̇
```

### Client/Server Split (WASM Threshold)

```
Structure uploaded → Element count extracted
    │
    ├── ≤100 elements → WASM solver (browser)
    │   ├── Instant modal analysis (<500ms)
    │   ├── Response spectrum results displayed immediately
    │   └── No server cost
    │
    └── >100 elements → Server solver (Rust native)
        ├── Job queued via Redis
        ├── Worker picks up, runs native solver with ARPACK
        ├── Progress streamed via WebSocket
        └── Results stored in S3, URL returned
```

The 100-element threshold was chosen because:
- WASM solver handles 100-element modal analysis (12 modes) in <500ms on modern hardware
- 100 elements covers: 3-story moment frame (60 elements), 5-story with minor complexity (90 elements)
- Above 100 elements: high-rise buildings, complex irregularities, time-history suites → need server compute

### WASM Compilation Pipeline

```toml
# solver-wasm/Cargo.toml
[package]
name = "seismiq-solver-wasm"
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
nalgebra-sparse = "0.10"
superlu-src = { version = "0.6", features = ["wasm"] }  # SuperLU sparse solver
log = "0.4"
console_log = "1"

[profile.release]
opt-level = "z"       # Optimize for size
lto = true            # Link-time optimization
codegen-units = 1     # Single codegen unit
strip = true          # Strip debug symbols
wasm-opt = true       # Run wasm-opt
```

```yaml
# .github/workflows/wasm-build.yml
name: Build WASM Solver
on:
  push:
    paths: ['solver-wasm/**', 'solver-core/**']
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with:
          targets: wasm32-unknown-unknown
      - uses: cargo-bins/cargo-binstall@main
      - run: cargo binstall -y wasm-pack wasm-opt
      - run: cd solver-wasm && wasm-pack build --target web --release
      - run: wasm-opt -Oz solver-wasm/pkg/seismiq_solver_wasm_bg.wasm -o solver-wasm/pkg/seismiq_solver_wasm_bg.wasm
      - name: Upload to S3/CDN
        run: aws s3 cp solver-wasm/pkg/ s3://seismiq-wasm/v${{ github.sha }}/ --recursive
```

---

## Architecture Deep-Dives

### 1. Analysis API Handler (Rust/Axum)

The primary endpoint receives an analysis request, validates the structural model, decides between WASM and server execution, and for server execution enqueues a job with Redis and streams progress via WebSocket.

```rust
// src/api/handlers/analysis.rs

use axum::{
    extract::{Path, State, Json},
    http::StatusCode,
    response::IntoResponse,
};
use uuid::Uuid;
use crate::{
    db::models::{Analysis, AnalysisParams, AnalysisType},
    solver::model::validate_structural_model,
    state::AppState,
    auth::Claims,
    error::ApiError,
};

#[derive(serde::Deserialize)]
pub struct CreateAnalysisRequest {
    pub analysis_type: AnalysisType,
    pub parameters: serde_json::Value,
}

pub async fn create_analysis(
    State(state): State<AppState>,
    claims: Claims,
    Path(project_id): Path<Uuid>,
    Json(req): Json<CreateAnalysisRequest>,
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

    // 2. Validate structural model
    let model = validate_structural_model(&project.model_data)?;
    let element_count = model.elements.len();
    let node_count = model.nodes.len();

    // 3. Check plan limits
    let user = sqlx::query_as!(
        crate::db::models::User,
        "SELECT * FROM users WHERE id = $1",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    if user.plan == "free" && element_count > 50 {
        return Err(ApiError::PlanLimit(
            "Free plan supports structures up to 50 elements. Upgrade to Professional for unlimited."
        ));
    }

    if user.plan == "free" && matches!(req.analysis_type, AnalysisType::TimeHistory) {
        return Err(ApiError::PlanLimit(
            "Time-history analysis requires Professional or Advanced plan."
        ));
    }

    // 4. Determine execution mode
    let execution_mode = if element_count <= 100
        && !matches!(req.analysis_type, AnalysisType::TimeHistory | AnalysisType::Pushover) {
        "wasm"
    } else {
        "server"
    };

    // 5. Create analysis record
    let analysis = sqlx::query_as!(
        Analysis,
        r#"INSERT INTO analyses
            (project_id, user_id, analysis_type, status, execution_mode,
             model_snapshot, parameters, element_count, node_count)
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9)
        RETURNING *"#,
        project_id,
        claims.user_id,
        serde_json::to_string(&req.analysis_type)?,
        if execution_mode == "wasm" { "completed" } else { "pending" },
        execution_mode,
        project.model_data,
        req.parameters,
        element_count as i32,
        node_count as i32,
    )
    .fetch_one(&state.db)
    .await?;

    // 6. For server execution, enqueue job
    if execution_mode == "server" {
        let job = sqlx::query_as!(
            crate::db::models::AnalysisJob,
            r#"INSERT INTO analysis_jobs (analysis_id, priority, cores_allocated)
            VALUES ($1, $2, $3) RETURNING *"#,
            analysis.id,
            if user.plan == "advanced" { 10 } else { 0 },
            if element_count > 1000 { 8 } else { 4 },
        )
        .fetch_one(&state.db)
        .await?;

        // Publish to Redis job queue
        state.redis
            .publish("analysis:jobs", serde_json::to_string(&job.id)?)
            .await?;
    }

    Ok((StatusCode::CREATED, Json(analysis)))
}

pub async fn get_analysis(
    State(state): State<AppState>,
    claims: Claims,
    Path((project_id, analysis_id)): Path<(Uuid, Uuid)>,
) -> Result<Json<Analysis>, ApiError> {
    let analysis = sqlx::query_as!(
        Analysis,
        "SELECT a.* FROM analyses a
         JOIN projects p ON a.project_id = p.id
         WHERE a.id = $1 AND a.project_id = $2
         AND (p.owner_id = $3 OR p.org_id IN (
             SELECT org_id FROM org_members WHERE user_id = $3
         ))",
        analysis_id,
        project_id,
        claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Analysis not found"))?;

    Ok(Json(analysis))
}
```

### 2. FEA Solver Core (Rust — shared between WASM and native)

The core 3D frame FEA solver that assembles global stiffness and mass matrices, solves for modes and displacements. This code compiles to both native (server) and WASM (browser) targets.

```rust
// solver-core/src/frame_solver.rs

use nalgebra::{DMatrix, DVector};
use nalgebra_sparse::{CooMatrix, CsrMatrix};
use crate::elements::{Element, BeamColumn3D};
use crate::superlu::SuperLUSolver;

pub struct StructuralModel {
    pub nodes: Vec<Node>,
    pub elements: Vec<Box<dyn Element>>,
    pub constraints: Vec<Constraint>,
    pub loads: Vec<Load>,
    pub mass_assignments: Vec<MassAssignment>,
}

#[derive(Debug, Clone)]
pub struct Node {
    pub id: usize,
    pub x: f64,
    pub y: f64,
    pub z: f64,
}

#[derive(Debug, Clone)]
pub struct Constraint {
    pub node_id: usize,
    pub dof: Vec<usize>,  // DOFs to constrain (0=Ux, 1=Uy, 2=Uz, 3=Rx, 4=Ry, 5=Rz)
}

#[derive(Debug, Clone)]
pub struct Load {
    pub node_id: usize,
    pub force: [f64; 6],  // Fx, Fy, Fz, Mx, My, Mz
}

#[derive(Debug, Clone)]
pub struct MassAssignment {
    pub node_id: usize,
    pub mass: f64,  // Lumped mass (kg)
}

pub struct GlobalMatrices {
    pub n_dof: usize,
    pub stiffness: CsrMatrix<f64>,
    pub mass: CsrMatrix<f64>,
    pub dof_map: Vec<Option<usize>>,  // Maps local DOF to global DOF (None if constrained)
}

impl StructuralModel {
    pub fn assemble_matrices(&self) -> Result<GlobalMatrices, SolverError> {
        let n_nodes = self.nodes.len();
        let n_dof_total = n_nodes * 6;  // 6 DOF per node

        // Build DOF map accounting for constraints
        let mut dof_map = vec![Some(0); n_dof_total];
        let mut constrained_dofs = std::collections::HashSet::new();

        for constraint in &self.constraints {
            for &local_dof in &constraint.dof {
                let global_dof = constraint.node_id * 6 + local_dof;
                constrained_dofs.insert(global_dof);
                dof_map[global_dof] = None;
            }
        }

        let mut active_dof_count = 0;
        for (i, entry) in dof_map.iter_mut().enumerate() {
            if !constrained_dofs.contains(&i) {
                *entry = Some(active_dof_count);
                active_dof_count += 1;
            }
        }

        let n_dof = active_dof_count;

        // Assemble stiffness and mass matrices in COO format
        let mut k_coo = CooMatrix::new(n_dof, n_dof);
        let mut m_coo = CooMatrix::new(n_dof, n_dof);

        // Element stiffness assembly
        for element in &self.elements {
            let k_elem = element.stiffness_matrix()?;
            let dof_indices = element.global_dof_indices();

            for (i_local, &i_global) in dof_indices.iter().enumerate() {
                let Some(i) = dof_map[i_global] else { continue };
                for (j_local, &j_global) in dof_indices.iter().enumerate() {
                    let Some(j) = dof_map[j_global] else { continue };
                    k_coo.push(i, j, k_elem[(i_local, j_local)]);
                }
            }
        }

        // Lumped mass assembly
        for mass_assign in &self.mass_assignments {
            let node_dof_start = mass_assign.node_id * 6;
            for offset in 0..3 {  // Only translational DOFs get mass
                let global_dof = node_dof_start + offset;
                if let Some(i) = dof_map[global_dof] {
                    m_coo.push(i, i, mass_assign.mass);
                }
            }
        }

        Ok(GlobalMatrices {
            n_dof,
            stiffness: CsrMatrix::from(&k_coo),
            mass: CsrMatrix::from(&m_coo),
            dof_map,
        })
    }

    pub fn solve_static(&self) -> Result<StaticResult, SolverError> {
        let matrices = self.assemble_matrices()?;

        // Build load vector
        let mut f = DVector::zeros(matrices.n_dof);
        for load in &self.loads {
            for (dof_offset, &force_component) in load.force.iter().enumerate() {
                let global_dof = load.node_id * 6 + dof_offset;
                if let Some(i) = matrices.dof_map[global_dof] {
                    f[i] += force_component;
                }
            }
        }

        // Solve K u = F using SuperLU
        let solver = SuperLUSolver::new(&matrices.stiffness)?;
        let u = solver.solve(&f)?;

        // Expand solution to include constrained DOFs (displacement = 0)
        let mut u_full = vec![0.0; self.nodes.len() * 6];
        for (global_dof, u_val) in matrices.dof_map.iter().zip(u.iter()) {
            if let Some(i) = global_dof {
                u_full[*i] = *u_val;
            }
        }

        Ok(StaticResult {
            displacements: u_full,
            node_count: self.nodes.len(),
        })
    }

    pub fn solve_modal(&self, num_modes: usize) -> Result<ModalResult, SolverError> {
        let matrices = self.assemble_matrices()?;

        // Solve generalized eigenvalue problem (K - λ M) φ = 0
        // Using Lanczos iteration via ARPACK bindings
        let eigenvalues = crate::eigensolve::lanczos_sparse(
            &matrices.stiffness,
            &matrices.mass,
            num_modes,
        )?;

        let mut modes = Vec::new();
        for (lambda, eigenvector) in eigenvalues {
            let omega = lambda.sqrt();  // rad/s
            let frequency = omega / (2.0 * std::f64::consts::PI);  // Hz
            let period = 1.0 / frequency;  // seconds

            // Expand eigenvector to full DOF space
            let mut mode_shape = vec![0.0; self.nodes.len() * 6];
            for (global_dof_idx, &active_dof) in matrices.dof_map.iter().enumerate() {
                if let Some(i) = active_dof {
                    mode_shape[global_dof_idx] = eigenvector[i];
                }
            }

            // Calculate modal mass participation
            let gamma = self.modal_participation_factor(&mode_shape, &matrices);

            modes.push(ModeData {
                mode_number: modes.len() + 1,
                period,
                frequency,
                circular_frequency: omega,
                mode_shape,
                participation_factor_x: gamma.0,
                participation_factor_y: gamma.1,
                participation_factor_z: gamma.2,
            });
        }

        Ok(ModalResult {
            modes,
            node_count: self.nodes.len(),
        })
    }

    fn modal_participation_factor(
        &self,
        mode_shape: &[f64],
        matrices: &GlobalMatrices,
    ) -> (f64, f64, f64) {
        // Γ_x = (φᵀ M r_x) / (φᵀ M φ)
        // r_x is influence vector: 1 for X-translation DOFs, 0 otherwise

        let n_nodes = self.nodes.len();
        let mut phi_m_rx = 0.0;
        let mut phi_m_ry = 0.0;
        let mut phi_m_rz = 0.0;
        let mut phi_m_phi = 0.0;

        for node_id in 0..n_nodes {
            if let Some(mass_assign) = self.mass_assignments.iter()
                .find(|m| m.node_id == node_id) {
                let m = mass_assign.mass;
                let ux = mode_shape[node_id * 6 + 0];
                let uy = mode_shape[node_id * 6 + 1];
                let uz = mode_shape[node_id * 6 + 2];

                phi_m_rx += m * ux;
                phi_m_ry += m * uy;
                phi_m_rz += m * uz;
                phi_m_phi += m * (ux * ux + uy * uy + uz * uz);
            }
        }

        (
            phi_m_rx / phi_m_phi,
            phi_m_ry / phi_m_phi,
            phi_m_rz / phi_m_phi,
        )
    }
}

#[derive(Debug)]
pub struct StaticResult {
    pub displacements: Vec<f64>,  // 6 DOF per node
    pub node_count: usize,
}

#[derive(Debug)]
pub struct ModalResult {
    pub modes: Vec<ModeData>,
    pub node_count: usize,
}

#[derive(Debug, Clone)]
pub struct ModeData {
    pub mode_number: usize,
    pub period: f64,  // seconds
    pub frequency: f64,  // Hz
    pub circular_frequency: f64,  // rad/s
    pub mode_shape: Vec<f64>,  // Normalized displacement pattern
    pub participation_factor_x: f64,
    pub participation_factor_y: f64,
    pub participation_factor_z: f64,
}

#[derive(Debug)]
pub enum SolverError {
    InvalidModel(String),
    SingularMatrix(String),
    ConvergenceFailed(String),
    InvalidElement(String),
}
```

### 3. 3D Structural Viewer Component (React + Three.js)

The frontend 3D viewer that renders structural models, deformed shapes, and mode shapes using GPU-accelerated Three.js with interactive camera controls.

```typescript
// frontend/src/components/StructuralViewer/StructuralViewer.tsx

import { useRef, useEffect, useState } from 'react';
import { Canvas, useFrame, useThree } from '@react-three/fiber';
import { OrbitControls, Line } from '@react-three/drei';
import * as THREE from 'three';
import { useStructuralStore } from '../../stores/structuralStore';

interface Node {
  id: number;
  x: number;
  y: number;
  z: number;
}

interface Element {
  id: number;
  nodeI: number;
  nodeJ: number;
  sectionType: string;
  color?: string;
}

interface DeformedShape {
  nodeDisplacements: number[];  // 6 DOF per node [ux, uy, uz, rx, ry, rz, ...]
  scaleFactor: number;
}

export function StructuralViewer() {
  const { nodes, elements, analysisResult, deformedShapeScale } = useStructuralStore();
  const [showDeformed, setShowDeformed] = useState(false);
  const [selectedMode, setSelectedMode] = useState<number | null>(null);

  return (
    <div className="structural-viewer w-full h-full bg-gray-900 relative">
      <Canvas camera={{ position: [50, 50, 50], fov: 50 }}>
        <ambientLight intensity={0.4} />
        <directionalLight position={[10, 10, 10]} intensity={0.8} />
        <OrbitControls makeDefault />

        {/* Ground grid */}
        <gridHelper args={[100, 20, 0x444444, 0x222222]} />

        {/* Structural members */}
        <StructuralMembers
          nodes={nodes}
          elements={elements}
          deformed={showDeformed ? analysisResult?.deformedShape : null}
          modeShape={selectedMode !== null ? analysisResult?.modes[selectedMode] : null}
        />

        {/* Node markers */}
        <NodeMarkers nodes={nodes} />

        {/* Coordinate axes */}
        <axesHelper args={[5]} />
      </Canvas>

      {/* Controls overlay */}
      <div className="absolute top-4 right-4 bg-gray-800 text-white p-4 rounded space-y-2">
        <label className="flex items-center space-x-2">
          <input
            type="checkbox"
            checked={showDeformed}
            onChange={(e) => setShowDeformed(e.target.checked)}
          />
          <span>Show Deformed Shape</span>
        </label>

        {analysisResult?.modes && (
          <div>
            <label className="block mb-1">Mode Shape:</label>
            <select
              value={selectedMode ?? ''}
              onChange={(e) => setSelectedMode(e.target.value ? parseInt(e.target.value) : null)}
              className="w-full bg-gray-700 rounded px-2 py-1"
            >
              <option value="">None</option>
              {analysisResult.modes.map((mode, idx) => (
                <option key={idx} value={idx}>
                  Mode {mode.mode_number}: T = {mode.period.toFixed(3)}s
                </option>
              ))}
            </select>
          </div>
        )}

        <div>
          <label className="block mb-1">Scale: {deformedShapeScale.toFixed(1)}×</label>
          <input
            type="range"
            min="1"
            max="100"
            value={deformedShapeScale}
            onChange={(e) => useStructuralStore.setState({ deformedShapeScale: parseFloat(e.target.value) })}
            className="w-full"
          />
        </div>
      </div>
    </div>
  );
}

function StructuralMembers({
  nodes,
  elements,
  deformed,
  modeShape,
}: {
  nodes: Node[];
  elements: Element[];
  deformed: DeformedShape | null;
  modeShape: any | null;
}) {
  const nodeMap = new Map(nodes.map(n => [n.id, n]));

  return (
    <group>
      {elements.map(element => {
        const nodeI = nodeMap.get(element.nodeI);
        const nodeJ = nodeMap.get(element.nodeJ);
        if (!nodeI || !nodeJ) return null;

        let posI = new THREE.Vector3(nodeI.x, nodeI.y, nodeI.z);
        let posJ = new THREE.Vector3(nodeJ.x, nodeJ.y, nodeJ.z);

        // Apply deformed shape or mode shape
        if (deformed && deformed.nodeDisplacements) {
          const scale = deformed.scaleFactor;
          posI.add(new THREE.Vector3(
            deformed.nodeDisplacements[element.nodeI * 6 + 0] * scale,
            deformed.nodeDisplacements[element.nodeI * 6 + 1] * scale,
            deformed.nodeDisplacements[element.nodeI * 6 + 2] * scale,
          ));
          posJ.add(new THREE.Vector3(
            deformed.nodeDisplacements[element.nodeJ * 6 + 0] * scale,
            deformed.nodeDisplacements[element.nodeJ * 6 + 1] * scale,
            deformed.nodeDisplacements[element.nodeJ * 6 + 2] * scale,
          ));
        } else if (modeShape) {
          const scale = 10.0;  // Exaggerate mode shapes
          posI.add(new THREE.Vector3(
            modeShape.mode_shape[element.nodeI * 6 + 0] * scale,
            modeShape.mode_shape[element.nodeI * 6 + 1] * scale,
            modeShape.mode_shape[element.nodeI * 6 + 2] * scale,
          ));
          posJ.add(new THREE.Vector3(
            modeShape.mode_shape[element.nodeJ * 6 + 0] * scale,
            modeShape.mode_shape[element.nodeJ * 6 + 1] * scale,
            modeShape.mode_shape[element.nodeJ * 6 + 2] * scale,
          ));
        }

        const points = [posI, posJ];
        const color = element.color || (deformed || modeShape ? '#3b82f6' : '#10b981');

        return (
          <Line
            key={element.id}
            points={points}
            color={color}
            lineWidth={2}
          />
        );
      })}
    </group>
  );
}

function NodeMarkers({ nodes }: { nodes: Node[] }) {
  return (
    <group>
      {nodes.map(node => (
        <mesh key={node.id} position={[node.x, node.y, node.z]}>
          <sphereGeometry args={[0.2, 16, 16]} />
          <meshStandardMaterial color="#ef4444" />
        </mesh>
      ))}
    </group>
  );
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–5)

**Day 1: Project initialization and Rust backend scaffold**
```bash
cargo init seismiq-api
cd seismiq-api
cargo add axum tokio serde serde_json sqlx uuid chrono tower-http jsonwebtoken bcrypt
cargo add --features postgres,runtime-tokio-rustls,uuid,chrono,json,postgres-types sqlx
cargo add aws-sdk-s3 redis postgis geojson
```
- `src/main.rs` — Axum app with CORS, tracing, graceful shutdown
- `src/config.rs` — Environment-based configuration (DATABASE_URL, JWT_SECRET, S3_BUCKET, REDIS_URL)
- `src/state.rs` — AppState struct (PgPool, Redis, S3Client)
- `src/error.rs` — ApiError enum with IntoResponse
- `Dockerfile` — Multi-stage build (builder + runtime)
- `docker-compose.yml` — PostgreSQL with PostGIS, Redis, MinIO (S3-compatible)

**Day 2: Database schema and migrations**
- `migrations/001_initial.sql` — All 11 tables: users, organizations, org_members, projects, analyses, analysis_jobs, hazard_analyses, fault_sources, ground_motion_records, code_provisions, fragility_curves, usage_records
- `src/db/mod.rs` — Database pool initialization with PostGIS types
- `src/db/models.rs` — All SQLx structs with FromRow derives, PostGIS geometry helpers
- Run `sqlx migrate run` to apply schema
- Seed script for initial code provisions (ASCE 7-22 seismic parameters tables)

**Day 3: PostGIS integration and spatial types**
- `src/db/postgis.rs` — Custom SQLx encode/decode for PostGIS GEOGRAPHY types
- GeoPoint, LineString custom types with WKB (well-known binary) encoding
- Spatial query helpers: find faults within radius, find ground motions by distance-magnitude bin
- Test spatial queries with sample earthquake catalog data
- Seed global fault database from USGS Quaternary Faults (California, PNW) and GNS New Zealand

**Day 4: Authentication system**
- `src/auth/mod.rs` — JWT token generation and validation middleware
- `src/auth/oauth.rs` — Google and GitHub OAuth 2.0 flow handlers
- `src/api/handlers/auth.rs` — Register, login, OAuth callback, refresh token, me endpoints
- Password hashing with bcrypt (cost factor 12)
- JWT with 24h access token + 30d refresh token
- Auth middleware that extracts Claims from Authorization header

**Day 5: User and project CRUD**
- `src/api/handlers/users.rs` — Get profile, update profile, delete account
- `src/api/handlers/projects.rs` — Create, list, get, update, delete, fork project
- `src/api/handlers/orgs.rs` — Create org, invite member, list members, remove member
- `src/api/router.rs` — All route definitions with auth middleware
- Integration tests: auth flow, project CRUD, authorization checks

### Phase 2 — FEA Solver Core (Days 6–14)

**Day 6: Matrix framework and sparse solvers**
- `solver-core/` — New Rust workspace member (shared between native and WASM)
- `solver-core/src/matrix.rs` — COO and CSR sparse matrix types (via nalgebra-sparse)
- `solver-core/src/superlu.rs` — SuperLU sparse direct solver bindings
- Unit tests: simple 2×2 and 3×3 systems, verify sparse assembly

**Day 7: 3D beam-column element**
- `solver-core/src/elements/beam_column_3d.rs` — 12-DOF elastic beam-column element
- Local stiffness matrix formulation (axial, bending, torsion, shear)
- Rotation matrix T for local → global coordinate transformation
- Material properties: E (Young's modulus), G (shear modulus), A (area), I_y, I_z (moments of inertia), J (torsional constant)
- Unit tests: cantilever beam tip deflection, simply supported beam midspan deflection

**Day 8: Structural model assembly**
- `solver-core/src/frame_solver.rs` — StructuralModel struct with nodes, elements, constraints, loads
- Global stiffness matrix assembly: loop over elements, transform and stamp into global K
- Apply boundary conditions: zero rows/columns for constrained DOFs
- Build DOF map: local node DOF → active global DOF index
- Tests: portal frame static analysis, compare to hand calculations

**Day 9: Mass matrix and modal analysis setup**
- `solver-core/src/mass.rs` — Lumped mass matrix assembly (mass at translational DOFs only)
- Consistent mass matrix for beam-column element (Post-MVP)
- `solver-core/src/eigensolve.rs` — Interface for eigenvalue solvers (ARPACK bindings via `arpack-src`)
- Lanczos iteration for sparse symmetric generalized eigenvalue problem
- Tests: single-bay frame natural periods, compare to analytical formulas

**Day 10: Modal analysis solver**
- `solver-core/src/modal.rs` — Full modal analysis: solve (K - λ M) φ = 0
- Extract first N modes (periods, frequencies, mode shapes)
- Normalize mode shapes to unit modal mass
- Calculate modal mass participation factors Γ_x, Γ_y, Γ_z
- Cumulative mass participation to check if enough modes captured
- Tests: 3-story frame modal analysis, verify first-mode period

**Day 11: Response spectrum analysis**
- `solver-core/src/rsa.rs` — Response spectrum analysis procedure
- Load design spectrum from ASCE 7 or user-provided spectrum
- For each mode: compute spectral displacement S_d = S_a(T) / ω²
- Modal displacement: u_i = Γ_i · S_d(T_i) · φ_i
- CQC modal combination: ρ_ij = 8 ζ² (1 + r) r^(3/2) / [(1 - r²)² + 4 ζ² r (1 + r)²], r = ω_j/ω_i
- SRSS combination (simpler, conservative): u_total = √(Σ u_i²)
- Tests: 5-story frame RSA, verify base shear matches ELF

**Day 12: Equivalent lateral force (ELF) procedure**
- `solver-core/src/elf.rs` — ASCE 7-22 ELF procedure implementation
- Compute seismic base shear: V = C_s · W, where C_s = S_DS / (R / I_e)
- Vertical distribution of forces: F_x = C_vx · V, C_vx = w_x h_x^k / Σ(w_i h_i^k)
- Exponent k = 1 for T ≤ 0.5s, k = 2 for T ≥ 2.5s, linear interpolation between
- Apply lateral forces at each floor level, run static analysis
- Calculate story drifts and check against drift limits (0.020 h for most systems)
- Tests: simple shear building ELF, verify forces and drifts

**Day 13: ASCE 7 code compliance checking**
- `solver-core/src/code/asce7.rs` — ASCE 7-22 seismic provisions implementation
- Seismic design category (SDC) determination from S_DS, S_D1, and occupancy category
- R / Ω₀ / C_d factor selection based on structural system type
- Irregularity detection: torsional (plan), soft story, mass, geometric, in-plane discontinuity
- Drift limit checks: story drift ≤ Δ_a = 0.020 h (most systems), 0.010 h (masonry), 0.025 h (moment frames with certain detailing)
- Redundancy factor ρ: check number of bays resisting earthquake forces
- Tests: various building configurations, verify SDC and R-factors

**Day 14: Model validation and section library**
- `solver-core/src/sections/` — Section property library
- AISC steel shapes: W, HSS, L, C sections with tabulated properties (A, I, Z, S)
- ACI concrete sections: rectangular, T-beam, circular with reinforcement
- Section property calculator: A, I_y, I_z, J for arbitrary shapes
- Model validation: check for floating nodes, duplicate elements, invalid section properties
- Tests: section property calculations, model validation catches common errors

### Phase 3 — WASM Build + Frontend 3D Visualization (Days 15–21)

**Day 15: WASM solver compilation**
- `solver-wasm/` — New workspace member for WASM target
- `solver-wasm/Cargo.toml` — wasm-bindgen, serde-wasm-bindgen, nalgebra-sparse dependencies
- `solver-wasm/src/lib.rs` — WASM entry points: `solve_modal()`, `solve_static()`, `solve_rsa()`
- SuperLU compiled to WASM (verify sparse solver works in WASM context)
- `wasm-pack build --target web --release` pipeline
- `wasm-opt -Oz` for size optimization (target <3MB gzipped)
- JavaScript wrapper: `SeismiqSolver` class that loads WASM and provides async API

**Day 16: Frontend scaffold and Three.js setup**
```bash
npm create vite@latest frontend -- --template react-ts
cd frontend
npm i zustand @tanstack/react-query axios three @react-three/fiber @react-three/drei
npm i d3 @types/d3 recharts
```
- `src/App.tsx` — Router, auth context, layout
- `src/stores/structuralStore.ts` — Zustand store for structural model state
- `src/stores/analysisStore.ts` — Analysis state (modal results, deformed shapes)
- `src/components/StructuralViewer/Canvas.tsx` — Three.js canvas with OrbitControls
- `src/lib/wasmLoader.ts` — WASM bundle loading with caching

**Day 17: 3D structural model rendering**
- `src/components/StructuralViewer/StructuralViewer.tsx` — Main 3D viewer component
- `src/components/StructuralViewer/StructuralMembers.tsx` — Render beam-column elements as lines
- `src/components/StructuralViewer/NodeMarkers.tsx` — Render nodes as small spheres
- GPU instancing for efficient rendering of 1000+ members
- Color-coding by element type (beam, column, brace)
- Camera controls: orbit, pan, zoom with mouse/touch

**Day 18: Deformed shape and mode shape visualization**
- `src/components/StructuralViewer/DeformedShape.tsx` — Render deformed shape with displacement scaling
- `src/components/StructuralViewer/ModeShapeViewer.tsx` — Animated mode shape playback (sinusoidal oscillation)
- Color gradient based on displacement magnitude (blue = low, red = high)
- Scale slider to exaggerate deformations (1× to 100×)
- Mode selector dropdown with period/frequency display

**Day 19: Model builder UI — grid and nodes**
- `src/components/ModelBuilder/GridControls.tsx` — Grid spacing controls (X, Y, Z bay dimensions)
- `src/components/ModelBuilder/NodeEditor.tsx` — Add, move, delete nodes
- `src/components/ModelBuilder/SnapGrid.tsx` — Snap-to-grid overlay in 3D viewport
- Click-to-place nodes in 3D space
- Node coordinate display and manual input
- Tests: create simple 2×2 grid, verify node coordinates

**Day 20: Model builder UI — elements and sections**
- `src/components/ModelBuilder/ElementEditor.tsx` — Connect two nodes to create beam-column element
- `src/components/ModelBuilder/SectionLibrary.tsx` — Browse and select AISC/ACI sections
- `src/components/ModelBuilder/SectionAssignment.tsx` — Assign sections to elements
- Element highlight on hover/select
- Material property editor (E, G for steel/concrete)
- Auto-detect element type from orientation (vertical = column, horizontal = beam)

**Day 21: Model builder UI — constraints and loads**
- `src/components/ModelBuilder/ConstraintEditor.tsx` — Apply fixed/pinned/roller supports at nodes
- Visual glyph for each constraint type (fixed = triangle, pinned = circle with dot)
- `src/components/ModelBuilder/LoadEditor.tsx` — Apply point loads at nodes (force vector arrows)
- Floor mass assignment for seismic analysis (lumped mass at each floor level)
- Load combination selector (D, L, Lr, S, E per ASCE 7)

### Phase 4 — PSHA Engine + Hazard API (Days 22–28)

**Day 22: Ground motion prediction equations (GMPEs)**
- `psha-core/` — New Rust workspace member for PSHA calculations
- `psha-core/src/gmpe/nga_west2.rs` — NGA-West2 GMPEs (ASK14, BSSA14, CB14, CY14)
- Inputs: magnitude M, rupture distance R, site Vs30, fault mechanism
- Outputs: median spectral acceleration Sa(T), logarithmic standard deviation σ
- Implement for periods T = 0.01s to 10s (discrete points for interpolation)
- Tests: compare outputs to NGA-West2 validation spreadsheets

**Day 23: Probabilistic seismic hazard integration**
- `psha-core/src/hazard.rs` — PSHA hazard curve calculation
- For each source: integrate over magnitude distribution (Gutenberg-Richter or characteristic)
- For each (M, R) pair: evaluate GMPE, compute P(Sa > x | M, R)
- Sum contributions from all sources: λ(Sa > x) = Σ_sources ν · P(Sa > x | source)
- Hazard curve output: Sa values vs. annual frequency of exceedance λ
- Uniform hazard spectrum (UHS): for target return period, find Sa(T) at each period
- Tests: simple point-source hazard, compare to analytical solution

**Day 24: Fault source modeling**
- `psha-core/src/sources/fault.rs` — Finite fault source with rupture area and slip distribution
- Distance metrics: R_rup (closest to rupture), R_jb (Joyner-Boore, horizontal), R_x, R_y
- Fault geometry: dip, strike, rake, depth to top of rupture
- Moment-magnitude scaling: rupture area A vs. magnitude M_w
- Characteristic earthquake model: modal magnitude with truncated Gaussian
- Tests: calculate distances for various site-fault geometries

**Day 25: USGS NSHM API integration**
- `psha-core/src/usgs_api.rs` — HTTP client for USGS National Seismic Hazard Model
- Endpoints: design spectrum for site coordinates, hazard curves, deaggregation
- Response parsing: extract Sa values, return periods, deaggregation M-R bins
- Cache responses in PostgreSQL (keyed by lat/lon/site_class) for 30-day TTL
- Fallback to local PSHA engine if site outside USGS coverage or API unavailable
- Tests: query USGS API for known site (e.g., San Francisco), verify spectrum

**Day 26: Site response analysis — SHAKE equivalent linear**
- `psha-core/src/site_response/shake.rs` — 1D equivalent linear site response (SHAKE algorithm)
- Soil column: layers with thickness, Vs (shear wave velocity), density, G/G_max and damping curves
- Bedrock input motion (acceleration time series), propagate upward through layers
- Iterative equivalent linear: update shear modulus and damping based on strain level
- Output: surface acceleration time series, amplification spectrum
- Tests: compare to reference SHAKE91 results for standard soil profiles

**Day 27: Design spectrum generation (ASCE 7)**
- `psha-core/src/code/asce7_spectrum.rs` — ASCE 7-22 design spectrum construction
- Inputs: S_S (MCE at T=0.2s), S_1 (MCE at T=1.0s), site class
- Site coefficients F_a, F_v based on site class and S_S/S_1
- Adjusted MCE: S_MS = F_a S_S, S_M1 = F_v S_1
- Design spectral accelerations: S_DS = (2/3) S_MS, S_D1 = (2/3) S_M1
- Design spectrum shape: S_a(T) = S_DS for T ≤ T_0, linear transition, constant S_D1/T for T ≥ T_S
- Tests: verify spectrum for various site classes, compare to ASCE 7 examples

**Day 28: Hazard deaggregation**
- `psha-core/src/deagg.rs` — Hazard deaggregation by magnitude-distance-epsilon
- For target Sa(T) at given exceedance rate, find contributing (M, R, ε) bins
- Output: 2D histogram (M-R bins) showing contribution to hazard
- Modal (M, R, ε): the dominant scenario for design
- Useful for ground motion record selection matching modal scenario
- Tests: deaggregate hazard for high-seismicity site, verify modal M-R

### Phase 5 — Analysis API + Job Orchestration (Days 29–33)

**Day 29: Analysis API endpoints**
- `src/api/handlers/analysis.rs` — Create analysis, get analysis, list analyses, cancel analysis
- Model validation before analysis submission
- Element/node count extraction and WASM/server routing logic
- Plan-based limits enforcement (free: 50 elements, professional: unlimited, advanced: cloud time-history)

**Day 30: Server-side analysis worker**
- `src/workers/analysis_worker.rs` — Redis job consumer, runs native FEA solver
- `src/workers/mod.rs` — Worker pool management (configurable concurrency)
- Progress streaming: worker publishes progress → Redis pub/sub → WebSocket to client
- Error handling: singular matrix, divergence, memory limits
- S3 result upload: modal data (periods, mode shapes), RSA displacements, drift ratios
- Presigned URL generation for client download

**Day 31: WebSocket for live analysis progress**
- `src/api/ws/mod.rs` — Axum WebSocket upgrade handler
- `src/api/ws/analysis_progress.rs` — Subscribe to analysis progress channel
- Client receives: `{ progress_pct, current_mode, iteration, residual }`
- Connection management: heartbeat ping/pong, automatic cleanup on disconnect
- Frontend: `src/hooks/useAnalysisProgress.ts` — React hook for WebSocket subscription

**Day 32: Hazard analysis API**
- `src/api/handlers/hazard.rs` — Create hazard analysis, get hazard results
- PSHA calculation: call USGS API or run local hazard integration
- Design spectrum generation per ASCE 7
- Hazard deaggregation on demand
- S3 storage for hazard curves (Sa vs. λ) and spectra
- Results summary: S_S, S_1, S_DS, S_D1, SDC determination

**Day 33: Ground motion record database**
- Seed ground motion metadata from PEER NGA-West2 database
- `src/api/handlers/ground_motions.rs` — Search ground motions by M-R bin, Vs30, PGA range
- Download waveform time series from S3 (acceleration in g, time in seconds)
- Record selection for time-history analysis: match target spectrum, scale records
- Conditional spectrum matching (CS): select/scale records to match CS at target period
- Tests: search for M7 R<20km records at Vs30≈300, verify results

### Phase 6 — Results Visualization + Reporting (Days 34–37)

**Day 34: Modal analysis results UI**
- `src/components/Results/ModalTable.tsx` — Table of modes (period, frequency, mass participation)
- Cumulative mass participation chart (bar chart showing % for X, Y, Z directions)
- Mode shape animation controls (play/pause, speed slider)
- Export modal data to CSV
- Verify cumulative mass participation >90% in each direction

**Day 35: Response spectrum charts**
- `src/components/Results/SpectrumChart.tsx` — D3.js line chart for design spectrum
- Plot Sa (g) vs. T (seconds), log or linear scale toggle
- Overlay multiple spectra: code spectrum, USGS spectrum, user-uploaded
- Highlight fundamental period with vertical line
- Interactive hover for Sa value at any period
- Export chart as PNG or SVG

**Day 36: Drift and demand charts**
- `src/components/Results/DriftChart.tsx` — Story drift chart (floor elevation vs. drift ratio)
- Horizontal bar chart: drift ratio Δ/h for each story, color-coded by code limit
- Code compliance indicator: green if drift < limit, red if exceeded
- Inter-story drift ratio (IDR) calculation from floor displacements
- Export drift table to PDF

**Day 37: PDF engineering report generation**
- `src/api/handlers/reports.rs` — Generate PDF report for analysis
- Report sections:
  1. Project summary (name, location, date, engineer)
  2. Structural system description (frame type, material, heights, spans)
  3. Seismic hazard parameters (S_S, S_1, SDC, site class)
  4. Modal analysis results (periods, mass participation table)
  5. Response spectrum analysis results (base shear, story forces, drifts)
  6. Code compliance checks (drift limits, irregularities, redundancy)
  7. Design spectrum plot
  8. Drift chart
  9. Code references (ASCE 7-22 sections cited)
- PDF generation via headless Chrome (Puppeteer) rendering HTML template
- Download link with presigned S3 URL (TTL 24h)

### Phase 7 — Billing + Plan Enforcement (Days 38–40)

**Day 38: Stripe integration**
- `src/api/handlers/billing.rs` — Create checkout session, create customer portal session, get subscription status
- `src/api/handlers/webhooks/stripe.rs` — Handle subscription events: `checkout.session.completed`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.payment_failed`
- Plan mapping:
  - Free: 50 elements, ELF + modal + RSA, 3 projects, no time-history
  - Professional ($149/mo): Unlimited elements, all analysis types, 10 time-history analyses/month, report generation
  - Advanced ($349/mo): Everything in Professional, unlimited time-history, API access, site response analysis, FEMA P-58 (Post-MVP)

**Day 39: Usage tracking and plan limits**
- `src/middleware/plan_limits.rs` — Middleware checking plan limits before analysis execution
- `src/services/usage.rs` — Track time-history analysis count per billing period
- Usage record insertion after each analysis completes (for time-history and cloud compute time)
- Usage dashboard: `src/api/handlers/usage.rs` — Current period usage, historical usage
- Approaching-limit warnings at 80% and 100% of time-history quota

**Day 40: Billing UI and feature gating**
- `frontend/src/pages/Billing.tsx` — Plan comparison table, current plan display, usage meter
- `frontend/src/components/billing/PlanCard.tsx` — Plan feature comparison cards
- `frontend/src/components/billing/UsageMeter.tsx` — Time-history analysis usage bar
- Upgrade/downgrade flow via Stripe Customer Portal
- Upgrade prompt modals when hitting plan limits (e.g., "50-element limit reached. Upgrade to Professional.")
- Gate time-history analysis behind Professional plan
- Gate PDF report generation behind Professional plan
- Gate API access behind Advanced plan
- Show locked feature indicators with upgrade CTAs in UI

### Phase 8 — Testing, Polish, Deployment (Days 41–42)

**Day 41: Solver validation and integration testing**
- Validation benchmark 1: Portal frame static analysis — compare to hand calculations (< 0.1% error)
- Validation benchmark 2: 5-story moment frame modal analysis — compare to SAP2000 reference (first 3 periods < 1% error)
- Validation benchmark 3: Response spectrum analysis base shear — match ASCE 7 ELF base shear (< 5% difference)
- Validation benchmark 4: PSHA hazard curve — compare to USGS NSHM at 5 California sites (Sa values < 10% error)
- Integration tests: create project → build model → run modal → view mode shapes → run RSA → check drifts → generate report
- WASM solver test: load in headless browser, run modal analysis, verify results match native solver
- WebSocket test: connect → subscribe → run server analysis → receive progress updates → receive completion

**Day 42: Deployment and launch**
- Docker and Kubernetes configuration:
  - `Dockerfile` — Multi-stage Rust build (cargo-chef for layer caching)
  - `k8s/api-deployment.yaml` — API server (3 replicas, HPA)
  - `k8s/worker-deployment.yaml` — Analysis workers (auto-scaling based on queue depth)
  - `k8s/postgres-statefulset.yaml` — PostgreSQL with PostGIS and PVC
  - `k8s/redis-deployment.yaml` — Redis for job queue
  - `k8s/ingress.yaml` — NGINX ingress with TLS
- CloudFront distribution for WASM bundle and static assets
- Monitoring: Prometheus metrics (analysis duration, solver convergence rate, API latency)
- Grafana dashboards: system health, analysis throughput, user activity
- Sentry integration for error tracking
- Landing page with product overview, pricing, signup
- Documentation: getting started guide, ASCE 7 compliance reference, example projects
- Deploy to production, enable monitoring alerts, announce launch

---

## Critical Files

```
seismiq/
├── solver-core/                           # Shared FEA solver library (native + WASM)
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── matrix.rs                      # Sparse matrix COO/CSR types
│   │   ├── superlu.rs                     # SuperLU sparse solver wrapper
│   │   ├── eigensolve.rs                  # ARPACK eigenvalue solver bindings
│   │   ├── frame_solver.rs                # Main FEA solver (assembly, solve)
│   │   ├── modal.rs                       # Modal analysis solver
│   │   ├── rsa.rs                         # Response spectrum analysis
│   │   ├── elf.rs                         # Equivalent lateral force procedure
│   │   ├── mass.rs                        # Mass matrix assembly
│   │   ├── elements/
│   │   │   ├── mod.rs                     # Element trait definition
│   │   │   ├── beam_column_3d.rs          # 3D elastic beam-column
│   │   │   └── shell_4node.rs             # 4-node shell (Post-MVP)
│   │   ├── sections/
│   │   │   ├── mod.rs                     # Section property calculator
│   │   │   ├── aisc_steel.rs              # AISC steel shapes database
│   │   │   └── aci_concrete.rs            # ACI concrete sections
│   │   ├── code/
│   │   │   ├── mod.rs
│   │   │   ├── asce7.rs                   # ASCE 7-22 seismic provisions
│   │   │   └── irregularity.rs            # Irregularity detection
│   │   └── validation.rs                  # Model validation (floating nodes, etc.)
│   └── tests/
│       ├── benchmarks.rs                  # Solver validation benchmarks
│       ├── modal_tests.rs                 # Modal analysis tests
│       └── code_compliance.rs             # Code compliance tests
│
├── psha-core/                             # PSHA engine (native + WASM)
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── hazard.rs                      # Probabilistic seismic hazard integration
│   │   ├── deagg.rs                       # Hazard deaggregation
│   │   ├── gmpe/
│   │   │   ├── mod.rs                     # GMPE trait
│   │   │   ├── nga_west2.rs               # NGA-West2 GMPEs (ASK14, BSSA14, CB14, CY14)
│   │   │   └── global.rs                  # Global GMPEs (Boore-Atkinson, etc.)
│   │   ├── sources/
│   │   │   ├── mod.rs                     # Seismic source trait
│   │   │   ├── fault.rs                   # Finite fault source
│   │   │   └── point.rs                   # Point source
│   │   ├── site_response/
│   │   │   ├── mod.rs
│   │   │   ├── shake.rs                   # SHAKE equivalent linear
│   │   │   └── newmark.rs                 # Nonlinear Newmark integration
│   │   ├── code/
│   │   │   └── asce7_spectrum.rs          # ASCE 7 design spectrum
│   │   └── usgs_api.rs                    # USGS NSHM API client
│   └── tests/
│       ├── gmpe_validation.rs             # GMPE validation vs. NGA-West2
│       └── hazard_tests.rs                # Hazard calculation tests
│
├── solver-wasm/                           # WASM compilation target for FEA
│   ├── Cargo.toml
│   └── src/
│       └── lib.rs                         # WASM entry points (wasm-bindgen)
│
├── psha-wasm/                             # WASM compilation target for PSHA
│   ├── Cargo.toml
│   └── src/
│       └── lib.rs                         # WASM entry points for hazard analysis
│
├── seismiq-api/                           # Rust API server
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
│   │   │   │   ├── analysis.rs            # Create/get/cancel analysis
│   │   │   │   ├── hazard.rs              # Hazard analysis endpoints
│   │   │   │   ├── ground_motions.rs      # Ground motion search/download
│   │   │   │   ├── reports.rs             # PDF report generation
│   │   │   │   ├── billing.rs             # Stripe checkout/portal
│   │   │   │   ├── usage.rs               # Usage tracking
│   │   │   │   └── webhooks/
│   │   │   │       └── stripe.rs          # Stripe webhook handler
│   │   │   └── ws/
│   │   │       ├── mod.rs                 # WebSocket upgrade handler
│   │   │       └── analysis_progress.rs   # Live progress streaming
│   │   ├── middleware/
│   │   │   └── plan_limits.rs             # Plan enforcement middleware
│   │   ├── services/
│   │   │   ├── usage.rs                   # Usage tracking service
│   │   │   ├── s3.rs                      # S3 client helpers
│   │   │   └── pdf.rs                     # PDF generation via Puppeteer
│   │   ├── db/
│   │   │   ├── mod.rs                     # Database pool initialization
│   │   │   ├── models.rs                  # SQLx structs
│   │   │   └── postgis.rs                 # PostGIS custom types
│   │   └── workers/
│   │       ├── mod.rs                     # Worker pool management
│   │       └── analysis_worker.rs         # Server-side FEA execution
│   ├── migrations/
│   │   └── 001_initial.sql                # Full database schema with PostGIS
│   └── tests/
│       ├── api_integration.rs             # API integration tests
│       └── analysis_e2e.rs                # End-to-end analysis tests
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
│   │   │   ├── structuralStore.ts         # Structural model state
│   │   │   ├── analysisStore.ts           # Analysis results state
│   │   │   └── hazardStore.ts             # Hazard analysis state
│   │   ├── hooks/
│   │   │   ├── useAnalysisProgress.ts     # WebSocket hook for live progress
│   │   │   ├── useWasmSolver.ts           # WASM FEA solver hook
│   │   │   └── useWasmPsha.ts             # WASM PSHA engine hook
│   │   ├── lib/
│   │   │   ├── api.ts                     # Axios API client
│   │   │   ├── wasmLoaderFea.ts           # WASM FEA bundle loading
│   │   │   └── wasmLoaderPsha.ts          # WASM PSHA bundle loading
│   │   ├── pages/
│   │   │   ├── Dashboard.tsx              # Project list
│   │   │   ├── Editor.tsx                 # Main editor (3D + panels)
│   │   │   ├── HazardAnalysis.tsx         # PSHA interface
│   │   │   ├── Results.tsx                # Analysis results viewer
│   │   │   ├── Billing.tsx                # Plan management
│   │   │   ├── Login.tsx                  # Auth pages
│   │   │   └── Register.tsx
│   │   ├── components/
│   │   │   ├── StructuralViewer/
│   │   │   │   ├── StructuralViewer.tsx   # Main 3D viewer (Three.js)
│   │   │   │   ├── StructuralMembers.tsx  # Render beam-columns
│   │   │   │   ├── NodeMarkers.tsx        # Render nodes
│   │   │   │   ├── DeformedShape.tsx      # Deformed shape rendering
│   │   │   │   ├── ModeShapeViewer.tsx    # Animated mode shapes
│   │   │   │   └── LoadVisualizer.tsx     # Force/moment vector arrows
│   │   │   ├── ModelBuilder/
│   │   │   │   ├── GridControls.tsx       # Grid spacing controls
│   │   │   │   ├── NodeEditor.tsx         # Node add/edit/delete
│   │   │   │   ├── ElementEditor.tsx      # Element creation
│   │   │   │   ├── SectionLibrary.tsx     # AISC/ACI section browser
│   │   │   │   ├── SectionAssignment.tsx  # Assign sections to elements
│   │   │   │   ├── ConstraintEditor.tsx   # Boundary conditions
│   │   │   │   └── LoadEditor.tsx         # Apply loads
│   │   │   ├── Results/
│   │   │   │   ├── ModalTable.tsx         # Modal analysis results table
│   │   │   │   ├── SpectrumChart.tsx      # Design spectrum chart (D3.js)
│   │   │   │   ├── DriftChart.tsx         # Story drift chart
│   │   │   │   └── ReportPreview.tsx      # PDF report preview
│   │   │   ├── Hazard/
│   │   │   │   ├── SiteLocationPicker.tsx # Map-based site picker
│   │   │   │   ├── HazardCurveChart.tsx   # Hazard curve plot
│   │   │   │   ├── DeaggregationPlot.tsx  # M-R deaggregation heatmap
│   │   │   │   └── DesignSpectrumView.tsx # ASCE 7 design spectrum
│   │   │   ├── billing/
│   │   │   │   ├── PlanCard.tsx
│   │   │   │   └── UsageMeter.tsx
│   │   │   └── common/
│   │   │       ├── AnalysisToolbar.tsx    # Run/stop/analysis type selector
│   │   │       └── CodeSelector.tsx       # ASCE 7 / Eurocode 8 / NZS selector
│   │   └── data/
│   │       └── templates/                 # Pre-built structural templates (JSON)
│   └── public/
│       └── wasm/                          # WASM bundles (FEA + PSHA)
│
├── loss-estimation/                       # Python FastAPI (FEMA P-58 — Post-MVP)
│   ├── requirements.txt
│   ├── main.py                            # FastAPI app
│   ├── fragility/
│   │   ├── fema_p58.py                    # FEMA P-58 fragility database
│   │   └── convolution.py                 # Monte Carlo loss estimation
│   └── Dockerfile
│
├── k8s/                                   # Kubernetes manifests
│   ├── api-deployment.yaml
│   ├── worker-deployment.yaml
│   ├── postgres-statefulset.yaml
│   ├── redis-deployment.yaml
│   ├── loss-service-deployment.yaml
│   └── ingress.yaml
│
├── docker-compose.yml                     # Local development stack
├── Cargo.toml                             # Workspace root
└── .github/
    └── workflows/
        ├── ci.yml                         # Test + lint on PR
        ├── wasm-build.yml                 # Build + deploy WASM bundles
        └── deploy.yml                     # Build Docker images + deploy to K8s
```

---

## Solver Validation Suite

### Benchmark 1: Portal Frame Static Analysis (Gravity Load)

**Structure:** 2-story, 1-bay portal frame (3 columns, 2 beams). Columns: W14×90 (E=200 GPa), beams: W18×50. Bay width 6m, story height 4m each.

**Loading:** Uniform distributed load 10 kN/m on each beam (gravity)

**Expected:** Midspan beam deflection ≈ **15.2 mm** (calculated via 5wL⁴/384EI for simply supported, adjusted for frame action)

**Tolerance:** < 2% (finite element discretization introduces small error)

### Benchmark 2: 5-Story Moment Frame Modal Analysis

**Structure:** 5-story, 3-bay moment frame. Columns: W14×90, beams: W18×50. Bay width 6m, story heights 4m. Seismic mass 100,000 kg per floor.

**Expected fundamental period:** T₁ ≈ **0.50 s** (Rayleigh estimate: T₁ = 2π√(Σm_i δ_i / Σm_i δ_i² g), where δ_i from static lateral load)

**Expected second period:** T₂ ≈ **0.18 s**

**Expected third period:** T₃ ≈ **0.10 s**

**Tolerance:** Periods < 3%, mass participation X-direction > 85% cumulative for first 3 modes

### Benchmark 3: Response Spectrum Analysis Base Shear vs. ELF

**Structure:** 5-story moment frame (same as Benchmark 2)

**Design spectrum:** ASCE 7-22 with S_DS = 1.0g, S_D1 = 0.6g, site class D

**Expected RSA base shear:** Should match ELF base shear V = C_s W within **10%** (ASCE 7 allows 85% of ELF for regular structures with RSA, but should be close)

**ELF base shear:** V = 0.1 W (for R=8 moment frame, I_e=1.0, C_s = S_DS/(R/I_e) = 1.0/8 = 0.125, capped at 0.1 for this period range)

**Tolerance:** Base shear difference < 10%, story drifts match within 15%

### Benchmark 4: PSHA Hazard Curve (Berkeley, CA)

**Site:** Berkeley, CA (37.87°N, -122.27°W), site class C (Vs30 = 500 m/s)

**Expected (from USGS 2014 NSHM):** Sa(0.2s) at 2475-year return period ≈ **1.5g**

**Expected:** Sa(1.0s) at 2475-year return period ≈ **0.6g**

**Tolerance:** < 15% difference from USGS values (acceptable given different source models and GMPEs)

### Benchmark 5: SHAKE Site Response Amplification

**Soil column:** 30m soft clay (Vs = 150 m/s, ρ = 1800 kg/m³) over bedrock (Vs = 760 m/s)

**Input motion:** 0.3g PGA bedrock motion

**Expected surface PGA:** ≈ **0.5g** (amplification factor ~1.7 for soft soil at moderate shaking)

**Expected amplification at T=0.6s:** ≈ **2.5** (peak amplification near site period T_site = 4H/Vs ≈ 0.8s)

**Tolerance:** Amplification factors < 20%, surface PGA < 15%

---

## Verification

### Integration Testing Checklist

1. **Auth flow** — Register → login → JWT issued → authorized API calls → token refresh → logout
2. **Project CRUD** — Create → update model → save → reload → verify model state preserved
3. **Model building** — Place nodes → create elements → assign sections → set constraints → verify JSON structure
4. **WASM modal analysis** — Small frame (20 elements) → WASM solver → modal results returned → mode shapes display
5. **Server analysis** — Large frame (150 elements) → job queued → worker picks up → WebSocket progress → results in S3
6. **3D visualization** — Load modal results → view mode shapes → animate → deformed shape rendering
7. **PSHA** — Enter site coordinates → USGS API call → design spectrum displayed → S_DS/S_D1 extracted
8. **Response spectrum analysis** — Load design spectrum → run RSA → drift ratios calculated → code compliance checked
9. **PDF report** — Generate report → verify all sections present → download link works
10. **Plan limits** — Free user → 60-element model → blocked with upgrade prompt
11. **Billing** — Subscribe to Professional → Stripe checkout → webhook → plan updated → limits lifted
12. **Concurrent users** — 5 users simultaneously running server analyses → all complete correctly
13. **Mode shape animation** — Select mode → play animation → sinusoidal oscillation at correct frequency
14. **Ground motion search** — Search M7 R<20km Vs30≈300 → results returned → download waveform
15. **Error handling** — Singular matrix (floating nodes) → meaningful error message → no crash

### SQL Verification Queries

```sql
-- 1. Analysis throughput and success rate
SELECT
    execution_mode,
    status,
    COUNT(*) as count,
    AVG(wall_time_ms)::int as avg_time_ms,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY wall_time_ms)::int as p95_time_ms
FROM analyses
WHERE created_at >= NOW() - INTERVAL '7 days'
GROUP BY execution_mode, status;

-- 2. Analysis type distribution
SELECT analysis_type, COUNT(*) as count,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 1) as pct
FROM analyses
WHERE created_at >= NOW() - INTERVAL '30 days'
GROUP BY analysis_type ORDER BY count DESC;

-- 3. User plan distribution
SELECT plan, COUNT(*) as users,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 1) as pct
FROM users GROUP BY plan ORDER BY users DESC;

-- 4. Time-history analysis usage by user (billing period)
SELECT u.email, u.plan,
    COUNT(*) as time_history_count,
    CASE u.plan
        WHEN 'free' THEN 0
        WHEN 'professional' THEN 10
        WHEN 'advanced' THEN 999999
    END as limit_count
FROM analyses a
JOIN users u ON a.user_id = u.id
WHERE a.analysis_type = 'time_history'
    AND a.created_at >= DATE_TRUNC('month', CURRENT_DATE)
    AND a.created_at < DATE_TRUNC('month', CURRENT_DATE) + INTERVAL '1 month'
GROUP BY u.email, u.plan
ORDER BY time_history_count DESC;

-- 5. Most analyzed structural systems
SELECT
    p.model_data->>'system_type' as system_type,
    COUNT(DISTINCT a.id) as analysis_count,
    AVG((p.model_data->>'element_count')::int) as avg_elements
FROM projects p
JOIN analyses a ON a.project_id = p.id
WHERE p.model_data->>'system_type' IS NOT NULL
GROUP BY system_type
ORDER BY analysis_count DESC
LIMIT 10;
```

---

## Performance Benchmarks

| Operation | Target | Method |
|-----------|--------|--------|
| WASM solver: 50-element modal (12 modes) | <300ms | Browser benchmark (Chrome DevTools) |
| WASM solver: 100-element modal (12 modes) | <800ms | Browser benchmark |
| Server solver: 500-element modal (30 modes) | <5s | Server timing, 4 cores |
| Server solver: 1000-element RSA | <10s | Server timing, 8 cores |
| PSHA: single-site hazard curve | <2s | WASM or server (local calculation) |
| USGS API: design spectrum fetch | <500ms | HTTP request latency |
| 3D viewer: 200-element model rendering | 60 FPS | Three.js performance monitor |
| 3D viewer: animated mode shape (500 elements) | 30 FPS | Three.js performance monitor |
| API: create analysis | <150ms | p95 latency (Prometheus) |
| API: search ground motions | <100ms | p95 latency with PostGIS spatial index |
| WASM FEA bundle load (cached) | <400ms | Service worker cache hit |
| WASM FEA bundle load (cold) | <2s | CDN delivery, ~2MB gzipped |
| WebSocket: progress latency | <100ms | Time from worker emit to client receive |
| PDF report generation | <8s | Puppeteer headless Chrome rendering |

---

## Deployment Architecture

### Infrastructure

```
┌─────────────────────────────────────────────────┐
│                  CloudFront CDN                  │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────┐│
│  │ Frontend SPA │  │ WASM Bundles│  │ Ground   ││
│  │ (HTML/JS/CSS)│  │ (FEA + PSHA)│  │ Motions  ││
│  └─────────────┘  └─────────────┘  └──────────┘│
└─────────────────────┬──────────────────────────┘
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
    ┌────────────────────┼────────────────┐
    ▼                    ▼                ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ PostgreSQL   │ │ Redis        │ │ S3 Bucket    │
│ + PostGIS    │ │ (ElastiCache)│ │ (Results +   │
│ (RDS r6g)    │ │ Cluster mode │ │  GMs)        │
│ Multi-AZ     │ │              │ │              │
└──────────────┘ └──────┬───────┘ └──────────────┘
                        │
    ┌───────────────────┼───────────────┐
    ▼                   ▼               ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ FEA Worker   │ │ FEA Worker   │ │ FEA Worker   │
│ (Rust native)│ │ (Rust native)│ │ (Rust native)│
│ c7g.2xlarge  │ │ c7g.2xlarge  │ │ c7g.2xlarge  │
│ HPA: 2-10    │ │              │ │              │
└──────────────┘ └──────────────┘ └──────────────┘
```

### Scaling Strategy

- **API servers**: Horizontal pod autoscaler (HPA) at 70% CPU, min 3, max 8
- **FEA workers**: HPA based on Redis queue depth — scale up when queue > 3 jobs, scale down when idle 5 min
- **Worker instances**: AWS c7g.2xlarge (8 vCPU, 16GB RAM) for compute-optimized FEA
- **Database**: RDS PostgreSQL r6g.xlarge with read replicas for hazard/ground motion queries
- **Redis**: ElastiCache r6g.large cluster with 1 primary + 2 replicas

### S3 Lifecycle Policy

```json
{
  "Rules": [
    {
      "ID": "analysis-results-lifecycle",
      "Prefix": "results/",
      "Status": "Enabled",
      "Transitions": [
        { "Days": 30, "StorageClass": "STANDARD_IA" },
        { "Days": 90, "StorageClass": "GLACIER" }
      ],
      "Expiration": { "Days": 365 }
    },
    {
      "ID": "ground-motions-keep",
      "Prefix": "ground_motions/",
      "Status": "Enabled",
      "Transitions": [
        { "Days": 180, "StorageClass": "STANDARD_IA" }
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
| Nonlinear Pushover Analysis | Static pushover with fiber-section beam-columns, plastic hinge models (concentrated plasticity), and capacity curve / demand point determination per ASCE 41-17 for performance-based evaluation | High |
| Nonlinear Time-History Analysis | Dynamic analysis with geometric nonlinearity (P-Δ), material nonlinearity (fiber sections, Menegotto-Pinto steel, Mander concrete), and cloud-distributed ground motion suites (50-100 records) | High |
| FEMA P-58 Loss Estimation | Component-level fragility convolution, Monte Carlo simulation for repair costs/downtime/casualties, portfolio-level risk aggregation with exceedance curves (PML, AAL) | High |
| Eurocode 8 and NZS 1170.5 Support | Full code compliance checking for European and New Zealand seismic codes, including capacity design rules, ductility factors, and force-based design procedures | Medium |
| Site Response Analysis (Full) | 1D nonlinear site response with advanced constitutive models (pressure-dependent, cyclic degradation), 2D/3D site response for complex geology, basin effects | Medium |
| Incremental Dynamic Analysis (IDA) | Run 10+ ground motions at incrementally scaled intensities, extract collapse fragility curves via lognormal fitting, estimate collapse probability for FEMA P-695 | Medium |
| Bridge Seismic Analysis | Line-girder models for multi-span bridges, abutment/bearing modeling, irregular geometry support, Caltrans SDC compliance checking | Low |
| Soil-Structure Interaction | Foundation impedance functions (springs/dashpots), kinematic interaction factors, embedment effects, direct SSI via FEA substructuring | Low |
