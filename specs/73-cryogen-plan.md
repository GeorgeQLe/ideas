# 73. CryoGen — Cryogenic Systems and Superconductor Design Platform

## Implementation Plan

**MVP Scope:** Browser-based cryogenic system designer with drag-and-drop component placement (cryocoolers, thermal shields, MLI insulation, thermal links, superconducting coils) and auto-routing of thermal paths rendered via SVG, custom thermal solver implementing finite-difference heat conduction equations compiled to WebAssembly for client-side execution of systems ≤200 components and server-side Rust-native execution for larger systems, support for steady-state heat load analysis, transient cooldown simulation, and superconducting magnet electromagnetic field calculation via Biot-Savart (fast analytical) and FEM (for complex geometries), material properties database covering 0.01K to 300K (NIST data: copper, aluminum, stainless steel, G10, Kapton, MLI) with temperature-dependent thermal conductivity and specific heat, interactive visualization rendered via WebGL showing temperature distribution and magnetic field contours with pan/zoom, 500+ built-in cryogenic components (cryocoolers from Cryomech/Sumitomo/Edwards, superconductors NbTi/Nb3Sn/REBCO with Jc(B,T) characteristics), design export as technical drawings and thermal analysis reports, Stripe billing with three tiers (Academic $99/mo / Pro $299/mo / Enterprise custom).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| Thermal Solver | Rust (native + WASM) | Custom finite-difference heat equation solver with sparse matrix solver |
| EM Field Solver | Rust (native + WASM) | Biot-Savart analytical + FEM via `fenics-rs` bindings for complex geometries |
| WASM Compilation | `wasm-pack` + `wasm-bindgen` | Client-side solver for systems ≤200 components |
| Scientific Computing | Python 3.12 (FastAPI) | REFPROP cryogenic fluid properties, advanced FEM meshing |
| Database | PostgreSQL 16 | Projects, users, simulations, component library metadata |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | Component models, simulation results (field data, temperature maps), design snapshots |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| System Designer | Custom SVG renderer | R-tree spatial indexing, snap-to-grid, thermal path routing |
| Field Visualizer | WebGL 2.0 (custom) | GPU-accelerated contour rendering, vector field display |
| Real-time | WebSocket (Axum) | Live simulation progress, convergence monitoring |
| Job Queue | Redis 7 + Tokio tasks | Server-side simulation job management, parameter sweeps |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | WASM bundle delivery, static frontend assets |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, error tracking |
| CI/CD | GitHub Actions | WASM build pipeline, Rust tests, Docker image push |

### Key Architecture Decisions

1. **Hybrid client/server solver with WASM threshold at 200 components**: Cryogenic systems with ≤200 components (covers 85%+ of single-chamber designs, small magnet assemblies, basic dilution refrigerators) run entirely in the browser via WASM, providing instant thermal analysis with zero server cost. Systems exceeding 200 components are submitted to the Rust-native server solver which handles large-scale designs (full MRI magnets, multi-chamber quantum dilution refrigerators). The threshold is configurable per plan tier.

2. **Custom thermal solver in Rust rather than wrapping OpenFOAM/COMSOL**: Building a custom finite-difference solver optimized for cryogenic temperature ranges gives us control over temperature-dependent material properties, WASM compilation, and specialized convergence for millikelvin to room temperature spans (6 orders of magnitude). OpenFOAM's general-purpose CFD is overkill and has poor WASM support. Our solver uses sparse matrix techniques (conjugate gradient) for systems with thousands of nodes.

3. **SVG system designer with R-tree spatial indexing**: SVG provides crisp rendering at any zoom level and straightforward DOM-based hit testing. An R-tree (via `rbush` JS library) enables O(log n) spatial queries for thermal path routing, snap detection, and component selection even with 500+ components. Canvas/WebGL alternatives were rejected because they require reimplementing text layout, cursor interaction, and accessibility.

4. **WebGL field visualizer separate from system designer**: The field visualizer renders electromagnetic and temperature field contours using GPU-accelerated fragment shaders with contour line extraction. This is decoupled from the SVG system designer to allow independent pan/zoom and multi-view layouts (temperature map, magnetic field, structural stress). Data is streamed from WASM/server as Float32Arrays and uploaded to GPU textures.

5. **S3 for component storage with PostgreSQL metadata catalog**: Cryogenic component specifications (cryocooler performance curves, superconductor Jc(B,T) tables, typically 5-100KB each) are stored in S3, while PostgreSQL holds searchable metadata (manufacturer, model, cooling power at temperature stages, critical current). This allows the component library to scale to 10K+ components without bloating the database while enabling fast parametric search via PostgreSQL indexes and full-text search.

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";  -- For fuzzy text search on components

-- Users
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email TEXT UNIQUE NOT NULL,
    password_hash TEXT,  -- NULL for OAuth-only users
    name TEXT NOT NULL,
    avatar_url TEXT,
    auth_provider TEXT DEFAULT 'email',  -- email | google | github
    auth_provider_id TEXT,
    plan TEXT NOT NULL DEFAULT 'academic',  -- academic | pro | enterprise
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX users_email_idx ON users(email);
CREATE INDEX users_stripe_idx ON users(stripe_customer_id);

-- Organizations (for Enterprise plan multi-user access)
CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    slug TEXT UNIQUE NOT NULL,
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    plan TEXT NOT NULL DEFAULT 'academic',
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

-- Projects (cryogenic system design workspace)
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    system_type TEXT NOT NULL DEFAULT 'general',  -- general | mri_magnet | dilution_refrigerator | particle_accelerator | quantum_computer
    design_data JSONB NOT NULL DEFAULT '{}',  -- Full system state (components, thermal links, shields)
    settings JSONB DEFAULT '{}',  -- Grid size, unit preferences (K/mK, W/mW)
    is_public BOOLEAN NOT NULL DEFAULT false,
    forked_from UUID REFERENCES projects(id) ON DELETE SET NULL,
    thumbnail_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);
CREATE INDEX projects_org_idx ON projects(org_id);
CREATE INDEX projects_type_idx ON projects(system_type);
CREATE INDEX projects_updated_idx ON projects(updated_at DESC);

-- Simulation Runs
CREATE TABLE simulations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    analysis_type TEXT NOT NULL,  -- heat_load | cooldown | magnet_field | quench
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | running | completed | failed | cancelled
    execution_mode TEXT NOT NULL DEFAULT 'wasm',  -- wasm | server
    component_count INTEGER NOT NULL DEFAULT 0,
    parameters JSONB NOT NULL DEFAULT '{}',  -- Analysis-specific parameters
    results_url TEXT,  -- S3 URL for result data (temperature maps, field distributions)
    results_summary JSONB,  -- Quick-access summary (peak heat load, minimum temperature, max field)
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
    convergence_data JSONB DEFAULT '[]',  -- [{iteration, temperature_residual, timestep}]
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX jobs_sim_idx ON simulation_jobs(simulation_id);
CREATE INDEX jobs_worker_idx ON simulation_jobs(worker_id);

-- Cryogenic Components (metadata; actual data files in S3)
CREATE TABLE components (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,  -- e.g., "Cryomech PT410" or "NbTi Wire 0.5mm"
    manufacturer TEXT,  -- e.g., "Cryomech", "SuperPower", "Oxford Instruments"
    model_number TEXT,
    category TEXT NOT NULL,  -- cryocooler | thermal_shield | mli | thermal_link | superconductor | support_structure | vessel
    subcategory TEXT,  -- e.g., "pulse_tube" for cryocooler, "nbti" for superconductor
    data_url TEXT NOT NULL,  -- S3 URL to performance data (JSON with T-dependent properties)
    parameters JSONB DEFAULT '{}',  -- Key specs for search (cooling_power_at_4k, max_field, etc.)
    symbol_data JSONB,  -- SVG symbol definition for system designer
    datasheet_url TEXT,
    description TEXT,
    tags TEXT[] DEFAULT '{}',
    is_verified BOOLEAN DEFAULT false,
    is_builtin BOOLEAN DEFAULT true,
    uploaded_by UUID REFERENCES users(id),
    usage_count INTEGER DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX components_category_idx ON components(category);
CREATE INDEX components_manufacturer_idx ON components(manufacturer);
CREATE INDEX components_name_trgm_idx ON components USING gin(name gin_trgm_ops);
CREATE INDEX components_model_trgm_idx ON components USING gin(model_number gin_trgm_ops);
CREATE INDEX components_tags_idx ON components USING gin(tags);

-- Material Properties (NIST cryogenic materials database)
CREATE TABLE materials (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,  -- e.g., "OFHC Copper", "Stainless Steel 304", "G-10 Epoxy"
    category TEXT NOT NULL,  -- metal | insulator | structural | superconductor
    data_url TEXT NOT NULL,  -- S3 URL to temperature-dependent property tables (k(T), Cp(T), emissivity)
    temperature_range_min REAL NOT NULL,  -- Kelvin
    temperature_range_max REAL NOT NULL,  -- Kelvin
    properties JSONB DEFAULT '{}',  -- Scalar properties (density, thermal expansion coefficient)
    source TEXT,  -- "NIST", "vendor", "literature"
    is_builtin BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX materials_category_idx ON materials(category);
CREATE INDEX materials_name_idx ON materials(name);

-- Saved Field Visualizations
CREATE TABLE field_configs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    simulation_id UUID NOT NULL REFERENCES simulations(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    view_type TEXT NOT NULL,  -- temperature_contour | magnetic_field_vector | heat_flux | stress
    settings JSONB NOT NULL,  -- {contour_levels, color_map, clip_planes, annotations}
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX field_configs_sim_idx ON field_configs(simulation_id);

-- Usage Tracking (for billing enforcement)
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,  -- simulation_minutes | server_simulation | storage_bytes
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
    pub system_type: String,
    pub design_data: serde_json::Value,
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
    pub component_count: i32,
    pub parameters: serde_json::Value,
    pub results_url: Option<String>,
    pub results_summary: Option<serde_json::Value>,
    pub error_message: Option<String>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub wall_time_ms: Option<i32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct SimulationJob {
    pub id: Uuid,
    pub simulation_id: Uuid,
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
pub struct Component {
    pub id: Uuid,
    pub name: String,
    pub manufacturer: Option<String>,
    pub model_number: Option<String>,
    pub category: String,
    pub subcategory: Option<String>,
    pub data_url: String,
    pub parameters: serde_json::Value,
    pub symbol_data: Option<serde_json::Value>,
    pub datasheet_url: Option<String>,
    pub description: Option<String>,
    pub tags: Vec<String>,
    pub is_verified: bool,
    pub is_builtin: bool,
    pub uploaded_by: Option<Uuid>,
    pub usage_count: i32,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Material {
    pub id: Uuid,
    pub name: String,
    pub category: String,
    pub data_url: String,
    pub temperature_range_min: f32,
    pub temperature_range_max: f32,
    pub properties: serde_json::Value,
    pub source: Option<String>,
    pub is_builtin: bool,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct SimulationParams {
    pub analysis_type: AnalysisType,
    pub heat_load: Option<HeatLoadParams>,
    pub cooldown: Option<CooldownParams>,
    pub magnet_field: Option<MagnetFieldParams>,
    pub ambient_temperature: f64,  // Kelvin, default 300.0
    pub solver_options: SolverOptions,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum AnalysisType {
    HeatLoad,
    Cooldown,
    MagnetField,
    Quench,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct HeatLoadParams {
    pub target_temperatures: Vec<f64>,  // Temperature at each stage (K)
    pub include_radiation: bool,
    pub include_conduction: bool,
    pub include_gas_conduction: bool,
    pub residual_pressure: f64,  // Pascal
}

#[derive(Debug, Deserialize, Serialize)]
pub struct CooldownParams {
    pub initial_temperature: f64,  // K
    pub target_temperature: f64,   // K
    pub timestep: f64,             // seconds
    pub max_time: f64,             // seconds
}

#[derive(Debug, Deserialize, Serialize)]
pub struct MagnetFieldParams {
    pub coil_geometry: String,  // "solenoid" | "dipole" | "quadrupole" | "helmholtz"
    pub current: f64,           // Amperes
    pub field_points: usize,    // Resolution for field map
    pub include_iron: bool,     // FEM for iron yoke
}

#[derive(Debug, Deserialize, Serialize)]
pub struct SolverOptions {
    pub max_iterations: u32,      // Default 1000
    pub convergence_tol: f64,     // Default 1e-6 for temperature (K)
    pub timestep_adaptive: bool,  // For transient analysis
    pub mesh_refinement: u32,     // 0 = coarse, 3 = fine
}
```

---

## Solver Architecture Deep-Dive

### Governing Equations and Discretization

CryoGen's core thermal solver implements **finite-difference discretization** of the heat diffusion equation with temperature-dependent material properties critical for cryogenic systems:

```
∂(ρ·Cp·T)/∂t = ∇·(k(T)·∇T) + Q

where:
  ρ = density (kg/m³)
  Cp(T) = specific heat (J/kg·K), highly temperature-dependent at cryogenic temperatures
  k(T) = thermal conductivity (W/m·K), varies by 100x-10000x from 4K to 300K
  T = temperature (K)
  Q = volumetric heat generation (W/m³)
```

**Steady-state heat load analysis** solves the elliptic PDE:

```
∇·(k(T)·∇T) + Q = 0

Discretized via finite differences on a 3D Cartesian grid:
k_{i+1/2,j,k} · (T_{i+1,j,k} - T_{i,j,k})/Δx - k_{i-1/2,j,k} · (T_{i,j,k} - T_{i-1,j,k})/Δx
  + [similar Y and Z terms] + Q_{i,j,k} = 0

where k_{i+1/2,j,k} = 2·k_{i,j,k}·k_{i+1,j,k} / (k_{i,j,k} + k_{i+1,j,k})  (harmonic mean for interface)
```

The system is linearized using **Newton-Raphson iteration** since k(T) and Cp(T) are temperature-dependent:

```
Jacobian J = ∂F/∂T evaluated at T_n
ΔT = -J^(-1) · F(T_n)
T_{n+1} = T_n + ΔT

Converges when ||ΔT|| < TOL (default 1e-6 K)
```

**Radiation heat transfer** between surfaces uses Stefan-Boltzmann with multi-layer insulation (MLI):

```
Q_rad = σ·A·ε_eff·(T_hot^4 - T_cold^4)

ε_eff = 1 / (1/ε_1 + N_layers·(1/ε_MLI - 1) + 1/ε_2 - 1)

where:
  σ = 5.67e-8 W/m²·K⁴ (Stefan-Boltzmann constant)
  N_layers = number of MLI layers (typically 10-60)
  ε_MLI ≈ 0.03 per layer
```

**Transient cooldown simulation** uses implicit Euler (unconditionally stable for diffusion):

```
(ρ·Cp)_{i,j,k} · (T_{i,j,k}^{n+1} - T_{i,j,k}^n) / Δt =
    ∇·(k(T^{n+1})·∇T^{n+1}) + Q

Solved via Newton-Raphson at each timestep n → n+1
Adaptive timestep: Δt_new = Δt · (TOL / ||T^{n+1} - T_predicted||)^(1/2)
```

### Superconducting Magnet Electromagnetic Field Solver

**Biot-Savart law** for analytical field calculation of simple geometries (solenoid, Helmholtz):

```
B(r) = (μ₀/4π) ∫∫ (J(r') × (r - r')) / |r - r'|³ dV'

For a solenoid (radius R, length L, N turns, current I):
B_z(z) = (μ₀·N·I / 2L) · [(z + L/2)/√((z+L/2)² + R²) - (z - L/2)/√((z-L/2)² + R²)]
B_r(r,z) computed via elliptic integral formulation
```

**Critical current density** for superconductors (temperature and field dependent):

```
Jc(B,T) = Jc0 · (1 - T/Tc) · (Bc2(T) / (B + Bc2(T)))^n

For NbTi: Tc = 9.2 K, Bc2(4.2K) ≈ 15 T, n ≈ 0.5
For Nb3Sn: Tc = 18 K, Bc2(4.2K) ≈ 28 T, n ≈ 1.0
For REBCO (HTS): Tc = 92 K, Bc2(77K) ≈ 100 T, n ≈ 0.7

Operating margin = I_op / (Jc(B_peak,T_op) · A_conductor)
```

**Quench propagation** (normal zone velocity):

```
v_quench = √(k·ρ_n / (ρ_sc·Cp)) · √(J² - Jc²) / Jc

where:
  k = thermal conductivity of copper matrix
  ρ_n = normal-state resistivity
  ρ_sc = superconductor density
  Cp = specific heat
  J = operating current density
  Jc = critical current density

Temperature rise during quench:
T_max = T_op + (J²·ρ_n·τ_dump) / (ρ_sc·Cp)
where τ_dump = L·I / V_dump (discharge time constant)
```

### Client/Server Split (WASM Threshold)

```
System uploaded → Component count extracted
    │
    ├── ≤200 components → WASM solver (browser)
    │   ├── Instant startup, no network latency
    │   ├── Results displayed immediately
    │   └── No server cost
    │
    └── >200 components → Server solver (Rust native)
        ├── Job queued via Redis
        ├── Worker picks up, runs native solver with higher memory/parallelization
        ├── Progress streamed via WebSocket
        └── Results stored in S3, URL returned
```

The 200-component threshold was chosen because:
- WASM solver handles 200-component steady-state thermal (5K grid nodes) in <3 seconds on modern hardware
- 200 components covers: single-stage cryocooler systems (50-100 components), small superconducting magnets (80-150 components), basic dilution refrigerator (100-180 components)
- Above 200 components: full MRI magnets (500+ components), large particle accelerator magnets, multi-user quantum computing dilution refrigerators → need server compute with 16GB+ RAM

### WASM Compilation Pipeline

```toml
# solver-wasm/Cargo.toml
[package]
name = "cryogen-solver-wasm"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
wasm-bindgen = "0.2"
serde = { version = "1", features = ["derive"] }
serde-wasm-bindgen = "0.6"
js-sys = "0.3"
nalgebra = "0.32"
nalgebra-sparse = "0.9"
sprs = "0.11"  # Sparse matrix library with CG solver
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
      - run: wasm-opt -Oz solver-wasm/pkg/cryogen_solver_wasm_bg.wasm -o solver-wasm/pkg/cryogen_solver_wasm_bg.wasm
      - name: Upload to S3/CDN
        run: aws s3 cp solver-wasm/pkg/ s3://cryogen-wasm/v${{ github.sha }}/ --recursive
```

---

## Architecture Deep-Dives

### 1. Simulation API Handler (Rust/Axum)

The primary endpoint receives a simulation request, validates the cryogenic system, decides between WASM and server execution, and for server execution enqueues a job with Redis and streams progress via WebSocket.

```rust
// src/api/handlers/simulation.rs

use axum::{
    extract::{Path, State, Json},
    http::StatusCode,
    response::IntoResponse,
};
use uuid::Uuid;
use crate::{
    db::models::{Simulation, SimulationParams, AnalysisType},
    solver::system::validate_system,
    state::AppState,
    auth::Claims,
    error::ApiError,
};

#[derive(serde::Deserialize)]
pub struct CreateSimulationRequest {
    pub analysis_type: AnalysisType,
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

    // 2. Validate system and extract component count
    let system = validate_system(&project.design_data)?;
    let component_count = system.components.len();

    // 3. Check plan limits
    let user = sqlx::query_as!(
        crate::db::models::User,
        "SELECT * FROM users WHERE id = $1",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    if user.plan == "academic" && component_count > 100 {
        return Err(ApiError::PlanLimit(
            "Academic plan supports systems up to 100 components. Upgrade to Pro for unlimited."
        ));
    }

    // 4. Determine execution mode
    let execution_mode = if component_count <= 200 {
        "wasm"
    } else {
        "server"
    };

    // 5. Create simulation record
    let sim = sqlx::query_as!(
        Simulation,
        r#"INSERT INTO simulations
            (project_id, user_id, analysis_type, status, execution_mode,
             component_count, parameters)
        VALUES ($1, $2, $3, $4, $5, $6, $7)
        RETURNING *"#,
        project_id,
        claims.user_id,
        serde_json::to_string(&req.analysis_type)?,
        if execution_mode == "wasm" { "completed" } else { "pending" },
        execution_mode,
        component_count as i32,
        req.parameters,
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
            if user.plan == "enterprise" { 10 } else { 0 },
        )
        .fetch_one(&state.db)
        .await?;

        // Publish to Redis job queue
        state.redis
            .publish("simulation:jobs", serde_json::to_string(&job.id)?)
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

### 2. Thermal Solver Core (Rust — shared between WASM and native)

The core finite-difference thermal solver that builds and solves the heat equation with temperature-dependent material properties. This code compiles to both native (server) and WASM (browser) targets.

```rust
// solver-core/src/thermal.rs

use nalgebra_sparse::{CsrMatrix, CooMatrix};
use crate::materials::{Material, ThermalProperties};
use crate::geometry::{Mesh3D, BoundaryCondition};

pub struct ThermalSystem {
    pub mesh: Mesh3D,
    pub temperatures: Vec<f64>,  // Current temperature at each node (K)
    pub prev_temperatures: Vec<f64>,  // Previous iteration for convergence check
    pub materials: Vec<Material>,
    pub boundary_conditions: Vec<BoundaryCondition>,
    pub heat_sources: Vec<(usize, f64)>,  // (node_idx, power in W)
}

impl ThermalSystem {
    pub fn new(mesh: Mesh3D) -> Self {
        let n_nodes = mesh.nodes.len();
        Self {
            mesh,
            temperatures: vec![300.0; n_nodes],  // Initialize to room temperature
            prev_temperatures: vec![300.0; n_nodes],
            materials: Vec::new(),
            boundary_conditions: Vec::new(),
            heat_sources: Vec::new(),
        }
    }

    /// Build the thermal conductance matrix K and heat load vector Q
    pub fn build_system_matrix(&self) -> (CooMatrix<f64>, Vec<f64>) {
        let n = self.mesh.nodes.len();
        let mut K = CooMatrix::new(n, n);
        let mut Q = vec![0.0; n];

        // Process each mesh element (hexahedron/tetrahedron)
        for elem in &self.mesh.elements {
            let mat = &self.materials[elem.material_id];

            // Get temperature-dependent thermal conductivity at element center
            let T_elem = elem.node_indices.iter()
                .map(|&i| self.temperatures[i])
                .sum::<f64>() / elem.node_indices.len() as f64;
            let k = mat.thermal_conductivity(T_elem);

            // Compute element conductance matrix via finite-difference stencil
            for (i, &ni) in elem.node_indices.iter().enumerate() {
                for (j, &nj) in elem.node_indices.iter().enumerate() {
                    let conductance = self.compute_conductance(elem, i, j, k);
                    K.push(ni, nj, conductance);
                }
            }
        }

        // Add heat sources
        for &(node_idx, power) in &self.heat_sources {
            Q[node_idx] += power;
        }

        // Add radiation boundary conditions (Stefan-Boltzmann)
        for bc in &self.boundary_conditions {
            if let BoundaryCondition::Radiation { node_idx, area, emissivity, T_ambient } = bc {
                let T = self.temperatures[*node_idx];
                let Q_rad = 5.67e-8 * area * emissivity * (T.powi(4) - T_ambient.powi(4));
                Q[*node_idx] -= Q_rad;

                // Linearized conductance for Newton-Raphson
                let dQ_dT = 5.67e-8 * area * emissivity * 4.0 * T.powi(3);
                K.push(*node_idx, *node_idx, dQ_dT);
            }
        }

        // Apply Dirichlet (fixed temperature) boundary conditions
        for bc in &self.boundary_conditions {
            if let BoundaryCondition::FixedTemperature { node_idx, temperature } = bc {
                // Zero out row and set diagonal to 1
                K.push(*node_idx, *node_idx, 1e12);  // Large number for enforcement
                Q[*node_idx] = *temperature * 1e12;
            }
        }

        (K, Q)
    }

    fn compute_conductance(&self, elem: &Element, i: usize, j: usize, k: f64) -> f64 {
        // Finite-difference conductance between nodes i and j in element
        let dx = self.mesh.nodes[elem.node_indices[j]].x - self.mesh.nodes[elem.node_indices[i]].x;
        let dy = self.mesh.nodes[elem.node_indices[j]].y - self.mesh.nodes[elem.node_indices[i]].y;
        let dz = self.mesh.nodes[elem.node_indices[j]].z - self.mesh.nodes[elem.node_indices[i]].z;
        let dist = (dx*dx + dy*dy + dz*dz).sqrt();

        if dist < 1e-12 { return 0.0; }

        let area = elem.face_area(i, j);  // Interface area
        k * area / dist
    }

    /// Solve steady-state heat load using Newton-Raphson
    pub fn solve_steady_state(&mut self, options: &SolverOptions) -> Result<(), SolverError> {
        for iter in 0..options.max_iterations {
            self.prev_temperatures = self.temperatures.clone();

            // Build system matrix K and vector Q
            let (K_coo, Q) = self.build_system_matrix();
            let K_csr = CsrMatrix::from(&K_coo);

            // Solve K·T = Q using sparse conjugate gradient
            let mut solver = sprs::linalg::CgSolver::new(&K_csr, 1000, 1e-8);
            self.temperatures = solver.solve(&Q)
                .map_err(|e| SolverError::SolverFailed(e.to_string()))?;

            // Check convergence
            if self.check_convergence(options.convergence_tol) {
                return Ok(());
            }
        }

        Err(SolverError::Convergence(
            format!("Failed to converge after {} iterations", options.max_iterations)
        ))
    }

    /// Transient cooldown simulation using implicit Euler
    pub fn solve_transient(
        &mut self,
        params: &CooldownParams,
        options: &SolverOptions
    ) -> Result<TransientResult, SolverError> {
        let mut result = TransientResult::new(self.mesh.nodes.len());
        let mut t = 0.0;
        let mut dt = params.timestep;

        // Initial condition
        result.push_timepoint(t, &self.temperatures);

        while t < params.max_time {
            // Implicit Euler: (M/dt + K)·T^(n+1) = M/dt·T^n + Q
            let (K_coo, Q) = self.build_system_matrix();
            let K_csr = CsrMatrix::from(&K_coo);

            // Build mass matrix M (diagonal: ρ·Cp·V for each node)
            let M = self.build_mass_matrix(dt);

            // Combine: A = M/dt + K
            let A = &M + &K_csr;

            // RHS: b = M/dt·T^n + Q
            let mut b = vec![0.0; self.temperatures.len()];
            for (i, &T_n) in self.temperatures.iter().enumerate() {
                b[i] = M.get(i, i).unwrap_or(0.0) * T_n + Q[i];
            }

            // Solve A·T^(n+1) = b
            let mut solver = sprs::linalg::CgSolver::new(&A, 1000, 1e-8);
            self.temperatures = solver.solve(&b)
                .map_err(|e| SolverError::SolverFailed(e.to_string()))?;

            t += dt;
            result.push_timepoint(t, &self.temperatures);

            // Check if target temperature reached
            let min_temp = self.temperatures.iter().cloned().fold(f64::INFINITY, f64::min);
            if min_temp <= params.target_temperature {
                break;
            }

            // Adaptive timestep (optional)
            if options.timestep_adaptive {
                dt = self.estimate_next_timestep(dt, options.convergence_tol);
            }
        }

        Ok(result)
    }

    fn build_mass_matrix(&self, dt: f64) -> CsrMatrix<f64> {
        let n = self.mesh.nodes.len();
        let mut M = CooMatrix::new(n, n);

        for elem in &self.mesh.elements {
            let mat = &self.materials[elem.material_id];
            let T_elem = elem.node_indices.iter()
                .map(|&i| self.temperatures[i])
                .sum::<f64>() / elem.node_indices.len() as f64;

            let rho = mat.density;
            let Cp = mat.specific_heat(T_elem);
            let volume = elem.volume / elem.node_indices.len() as f64;  // Distribute volume to nodes

            for &node_idx in &elem.node_indices {
                M.push(node_idx, node_idx, rho * Cp * volume / dt);
            }
        }

        CsrMatrix::from(&M)
    }

    fn check_convergence(&self, tol: f64) -> bool {
        for i in 0..self.temperatures.len() {
            let dT = (self.temperatures[i] - self.prev_temperatures[i]).abs();
            if dT > tol {
                return false;
            }
        }
        true
    }

    fn estimate_next_timestep(&self, dt_current: f64, tol: f64) -> f64 {
        // Adaptive timestep based on temperature change rate
        let max_dT = self.temperatures.iter()
            .zip(&self.prev_temperatures)
            .map(|(T, T_prev)| (T - T_prev).abs())
            .fold(0.0, f64::max);

        if max_dT < 1e-12 { return dt_current; }

        let dt_new = dt_current * (tol / max_dT).sqrt();
        dt_new.clamp(dt_current * 0.5, dt_current * 2.0)  // Limit change rate
    }
}

pub struct TransientResult {
    pub timepoints: Vec<f64>,
    pub temperature_history: Vec<Vec<f64>>,  // [timepoint][node_idx]
}

impl TransientResult {
    pub fn new(_n_nodes: usize) -> Self {
        Self {
            timepoints: Vec::new(),
            temperature_history: Vec::new(),
        }
    }

    pub fn push_timepoint(&mut self, t: f64, temperatures: &[f64]) {
        self.timepoints.push(t);
        self.temperature_history.push(temperatures.to_vec());
    }
}

#[derive(Debug)]
pub enum SolverError {
    Convergence(String),
    SolverFailed(String),
    InvalidSystem(String),
}
```

### 3. Magnet Field Solver (Rust — Biot-Savart + FEM)

Electromagnetic field calculation for superconducting magnets using fast analytical Biot-Savart for simple geometries and FEM for complex configurations with iron yokes.

```rust
// solver-core/src/magnet.rs

use nalgebra::{Vector3, Point3};
use std::f64::consts::PI;

pub struct MagnetCoil {
    pub geometry: CoilGeometry,
    pub current: f64,  // Amperes
    pub turns: usize,
    pub conductor: Conductor,
}

pub enum CoilGeometry {
    Solenoid { radius: f64, length: f64, center: Point3<f64> },
    Helmholtz { radius: f64, separation: f64, center: Point3<f64> },
    Dipole { width: f64, height: f64, length: f64, center: Point3<f64> },
    Racetrack { radius: f64, straight_length: f64, center: Point3<f64> },
}

pub struct Conductor {
    pub material: SuperconductorType,
    pub cross_section: f64,  // m²
    pub Jc_params: JcParameters,  // Critical current density parameters
}

pub enum SuperconductorType {
    NbTi,
    Nb3Sn,
    REBCO,
    Bi2212,
    MgB2,
}

pub struct JcParameters {
    pub Jc0: f64,        // A/m² at reference conditions
    pub Tc: f64,         // Critical temperature (K)
    pub Bc2_0: f64,      // Upper critical field at 0K (T)
    pub n: f64,          // Field exponent
}

impl MagnetCoil {
    /// Compute magnetic field at point using Biot-Savart law
    pub fn compute_field_at(&self, point: &Point3<f64>) -> Vector3<f64> {
        match &self.geometry {
            CoilGeometry::Solenoid { radius, length, center } => {
                self.solenoid_field(point, *radius, *length, center)
            }
            CoilGeometry::Helmholtz { radius, separation, center } => {
                let offset = separation / 2.0;
                let center1 = Point3::new(center.x, center.y, center.z - offset);
                let center2 = Point3::new(center.x, center.y, center.z + offset);

                self.solenoid_field(point, *radius, 0.01, &center1)
                    + self.solenoid_field(point, *radius, 0.01, &center2)
            }
            _ => {
                // For complex geometries, use numerical Biot-Savart integration
                self.numerical_biot_savart(point)
            }
        }
    }

    fn solenoid_field(&self, point: &Point3<f64>, R: f64, L: f64, center: &Point3<f64>) -> Vector3<f64> {
        let mu0 = 4.0 * PI * 1e-7;  // H/m
        let n = self.turns as f64 / L;  // turns per meter
        let I = self.current;

        // Axial field (on z-axis, point relative to center)
        let z = point.z - center.z;
        let r = ((point.x - center.x).powi(2) + (point.y - center.y).powi(2)).sqrt();

        if r < 1e-9 {
            // On-axis formula
            let z1 = z + L / 2.0;
            let z2 = z - L / 2.0;
            let Bz = (mu0 * n * I / 2.0) *
                (z1 / (z1.powi(2) + R.powi(2)).sqrt() - z2 / (z2.powi(2) + R.powi(2)).sqrt());
            Vector3::new(0.0, 0.0, Bz)
        } else {
            // Off-axis: use numerical integration or approximate via multipole expansion
            self.numerical_biot_savart(point)
        }
    }

    fn numerical_biot_savart(&self, point: &Point3<f64>) -> Vector3<f64> {
        // Discretize coil into current elements and sum contributions
        let mu0 = 4.0 * PI * 1e-7;
        let segments = 1000;
        let mut B = Vector3::new(0.0, 0.0, 0.0);

        // Get coil path points
        let path = self.geometry.discretize_path(segments);

        for i in 0..path.len() - 1 {
            let r1 = &path[i];
            let r2 = &path[i + 1];
            let dl = r2 - r1;
            let r_mid = (r1 + r2.coords) / 2.0;

            let R_vec = point - r_mid;
            let R_mag = R_vec.norm();

            if R_mag < 1e-9 { continue; }

            // dB = (μ₀·I / 4π) · (dl × R) / R³
            let dB = (mu0 * self.current * self.turns as f64 / (4.0 * PI))
                * dl.cross(&R_vec) / R_mag.powi(3);
            B += dB;
        }

        B
    }

    /// Check operating margin for superconductor
    pub fn operating_margin(&self, temperature: f64) -> Result<f64, String> {
        // Compute peak field in the coil
        let B_peak = self.compute_peak_field();

        // Critical current density at operating conditions
        let Jc = self.conductor.Jc_params.critical_current_density(B_peak, temperature);

        // Operating current density
        let J_op = (self.current * self.turns as f64) / self.conductor.cross_section;

        if Jc <= 0.0 {
            return Err(format!(
                "Superconductor exceeds critical parameters: B={:.2}T, T={:.2}K",
                B_peak, temperature
            ));
        }

        Ok(J_op / Jc)  // Margin < 1.0 is safe
    }

    fn compute_peak_field(&self) -> f64 {
        // Peak field is typically at inner radius for solenoid
        match &self.geometry {
            CoilGeometry::Solenoid { radius, length, center } => {
                let point = Point3::new(center.x + radius, center.y, center.z);
                self.compute_field_at(&point).norm()
            }
            _ => {
                // Sample grid and find maximum
                let mut B_max = 0.0;
                // ... grid sampling logic ...
                B_max
            }
        }
    }
}

impl JcParameters {
    /// Temperature and field dependent critical current density
    pub fn critical_current_density(&self, B: f64, T: f64) -> f64 {
        if T >= self.Tc {
            return 0.0;  // Above critical temperature
        }

        let t = T / self.Tc;
        let Bc2_T = self.Bc2_0 * (1.0 - t.powi(2));  // Approximate Bc2(T)

        if B >= Bc2_T {
            return 0.0;  // Above upper critical field
        }

        // Scaling law: Jc(B,T) = Jc0 · (1 - t) · (Bc2(T) / (B + Bc2(T)))^n
        self.Jc0 * (1.0 - t) * (Bc2_T / (B + Bc2_T)).powf(self.n)
    }
}

impl CoilGeometry {
    fn discretize_path(&self, segments: usize) -> Vec<Point3<f64>> {
        match self {
            CoilGeometry::Solenoid { radius, length, center } => {
                // Create helical path
                let mut path = Vec::new();
                for i in 0..=segments {
                    let theta = 2.0 * PI * i as f64 / segments as f64;
                    let z = -length / 2.0 + length * i as f64 / segments as f64;
                    path.push(Point3::new(
                        center.x + radius * theta.cos(),
                        center.y + radius * theta.sin(),
                        center.z + z
                    ));
                }
                path
            }
            _ => {
                // Implement other geometries
                Vec::new()
            }
        }
    }
}
```

### 4. Field Visualizer Component (React + WebGL)

The frontend field visualizer that renders temperature and magnetic field distributions using WebGL fragment shaders for GPU-accelerated contour generation.

```typescript
// frontend/src/components/FieldVisualizer.tsx

import { useRef, useEffect, useCallback, useState } from 'react';
import { useFieldStore } from '../stores/fieldStore';

interface FieldData {
  grid_x: number;
  grid_y: number;
  grid_z: number;
  values: Float32Array;  // Flattened 3D array [z][y][x]
  min_value: number;
  max_value: number;
}

interface VisualizationSettings {
  field_type: 'temperature' | 'magnetic_field_magnitude' | 'heat_flux';
  slice_plane: 'xy' | 'xz' | 'yz';
  slice_position: number;  // 0.0 to 1.0
  color_map: 'jet' | 'viridis' | 'inferno' | 'cool_warm';
  num_contours: number;
  show_vectors: boolean;
}

export function FieldVisualizer() {
  const canvasRef = useRef<HTMLCanvasElement>(null);
  const glRef = useRef<WebGL2RenderingContext | null>(null);
  const programRef = useRef<WebGLProgram | null>(null);
  const textureRef = useRef<WebGLTexture | null>(null);

  const { fieldData, settings, updateSettings } = useFieldStore();
  const [hoveredValue, setHoveredValue] = useState<{ x: number; y: number; value: number } | null>(null);

  // Initialize WebGL and compile shaders
  useEffect(() => {
    const canvas = canvasRef.current;
    if (!canvas) return;

    const gl = canvas.getContext('webgl2', { antialias: true, alpha: false });
    if (!gl) return;
    glRef.current = gl;

    // Vertex shader for fullscreen quad
    const vsSource = `#version 300 es
      in vec2 a_position;
      out vec2 v_texCoord;

      void main() {
        v_texCoord = a_position * 0.5 + 0.5;  // [0,1] range
        gl_Position = vec4(a_position, 0.0, 1.0);
      }
    `;

    // Fragment shader for field visualization with contours
    const fsSource = `#version 300 es
      precision highp float;

      in vec2 v_texCoord;
      uniform sampler2D u_fieldTexture;
      uniform float u_minValue;
      uniform float u_maxValue;
      uniform int u_numContours;
      uniform int u_colorMap;  // 0=jet, 1=viridis, 2=inferno, 3=cool_warm

      out vec4 fragColor;

      vec3 jet(float t) {
        return clamp(vec3(
          min(4.0 * t - 1.5, -4.0 * t + 4.5),
          min(4.0 * t - 0.5, -4.0 * t + 3.5),
          min(4.0 * t + 0.5, -4.0 * t + 2.5)
        ), 0.0, 1.0);
      }

      vec3 viridis(float t) {
        const vec3 c0 = vec3(0.267, 0.005, 0.329);
        const vec3 c1 = vec3(0.283, 0.141, 0.458);
        const vec3 c2 = vec3(0.254, 0.265, 0.530);
        const vec3 c3 = vec3(0.207, 0.372, 0.553);
        const vec3 c4 = vec3(0.164, 0.471, 0.558);
        const vec3 c5 = vec3(0.128, 0.567, 0.551);
        const vec3 c6 = vec3(0.135, 0.659, 0.518);
        const vec3 c7 = vec3(0.267, 0.749, 0.441);
        const vec3 c8 = vec3(0.478, 0.821, 0.318);
        const vec3 c9 = vec3(0.741, 0.873, 0.150);
        const vec3 c10 = vec3(0.993, 0.906, 0.144);

        float s = t * 10.0;
        int i = int(floor(s));
        float f = fract(s);

        if (i == 0) return mix(c0, c1, f);
        if (i == 1) return mix(c1, c2, f);
        if (i == 2) return mix(c2, c3, f);
        if (i == 3) return mix(c3, c4, f);
        if (i == 4) return mix(c4, c5, f);
        if (i == 5) return mix(c5, c6, f);
        if (i == 6) return mix(c6, c7, f);
        if (i == 7) return mix(c7, c8, f);
        if (i == 8) return mix(c8, c9, f);
        return mix(c9, c10, f);
      }

      vec3 getColor(float value) {
        float t = (value - u_minValue) / (u_maxValue - u_minValue);
        t = clamp(t, 0.0, 1.0);

        if (u_colorMap == 0) return jet(t);
        if (u_colorMap == 1) return viridis(t);
        // Add other colormaps...
        return vec3(t);
      }

      void main() {
        float value = texture(u_fieldTexture, v_texCoord).r;
        vec3 color = getColor(value);

        // Contour lines
        if (u_numContours > 0) {
          float normalized = (value - u_minValue) / (u_maxValue - u_minValue);
          float contourLevel = normalized * float(u_numContours);
          float contourFrac = fract(contourLevel);

          // Draw contour line when close to integer level
          if (contourFrac < 0.05 || contourFrac > 0.95) {
            color = mix(color, vec3(0.0), 0.5);  // Darken for contour lines
          }
        }

        fragColor = vec4(color, 1.0);
      }
    `;

    const program = createShaderProgram(gl, vsSource, fsSource);
    programRef.current = program;

    // Create fullscreen quad
    const positions = new Float32Array([-1, -1, 1, -1, -1, 1, 1, 1]);
    const buffer = gl.createBuffer();
    gl.bindBuffer(gl.ARRAY_BUFFER, buffer);
    gl.bufferData(gl.ARRAY_BUFFER, positions, gl.STATIC_DRAW);

    return () => {
      gl.deleteProgram(program);
      gl.deleteBuffer(buffer);
    };
  }, []);

  // Upload field data to GPU texture when it changes
  useEffect(() => {
    const gl = glRef.current;
    if (!gl || !fieldData) return;

    // Create 2D texture from field slice
    const slice = extractSlice(fieldData, settings.slice_plane, settings.slice_position);

    if (textureRef.current) {
      gl.deleteTexture(textureRef.current);
    }

    const texture = gl.createTexture();
    gl.bindTexture(gl.TEXTURE_2D, texture);
    gl.texImage2D(
      gl.TEXTURE_2D, 0, gl.R32F,
      slice.width, slice.height, 0,
      gl.RED, gl.FLOAT, slice.data
    );
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.LINEAR);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.LINEAR);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE);

    textureRef.current = texture;
  }, [fieldData, settings.slice_plane, settings.slice_position]);

  // Render
  const render = useCallback(() => {
    const gl = glRef.current;
    const program = programRef.current;
    const texture = textureRef.current;
    if (!gl || !program || !texture || !fieldData) return;

    gl.viewport(0, 0, gl.canvas.width, gl.canvas.height);
    gl.clearColor(0.1, 0.1, 0.1, 1.0);
    gl.clear(gl.COLOR_BUFFER_BIT);
    gl.useProgram(program);

    // Set uniforms
    gl.uniform1f(gl.getUniformLocation(program, 'u_minValue'), fieldData.min_value);
    gl.uniform1f(gl.getUniformLocation(program, 'u_maxValue'), fieldData.max_value);
    gl.uniform1i(gl.getUniformLocation(program, 'u_numContours'), settings.num_contours);
    gl.uniform1i(gl.getUniformLocation(program, 'u_colorMap'), getColorMapIndex(settings.color_map));
    gl.uniform1i(gl.getUniformLocation(program, 'u_fieldTexture'), 0);

    // Bind texture
    gl.activeTexture(gl.TEXTURE0);
    gl.bindTexture(gl.TEXTURE_2D, texture);

    // Draw quad
    const posLoc = gl.getAttribLocation(program, 'a_position');
    gl.enableVertexAttribArray(posLoc);
    gl.vertexAttribPointer(posLoc, 2, gl.FLOAT, false, 0, 0);
    gl.drawArrays(gl.TRIANGLE_STRIP, 0, 4);
  }, [fieldData, settings]);

  useEffect(() => {
    const animId = requestAnimationFrame(render);
    return () => cancelAnimationFrame(animId);
  }, [render]);

  return (
    <div className="field-visualizer flex flex-col h-full bg-gray-900">
      <VisualizerToolbar settings={settings} updateSettings={updateSettings} />
      <div className="flex-1 relative">
        <canvas
          ref={canvasRef}
          className="w-full h-full"
          width={1024}
          height={768}
        />
        {hoveredValue && (
          <div className="absolute pointer-events-none bg-gray-800 text-white text-xs px-2 py-1 rounded">
            {settings.field_type === 'temperature'
              ? `T = ${hoveredValue.value.toFixed(3)} K`
              : `|B| = ${hoveredValue.value.toFixed(4)} T`}
          </div>
        )}
        <ColorBar
          minValue={fieldData?.min_value ?? 0}
          maxValue={fieldData?.max_value ?? 1}
          colorMap={settings.color_map}
          unit={settings.field_type === 'temperature' ? 'K' : 'T'}
        />
      </div>
      <SliceControls
        plane={settings.slice_plane}
        position={settings.slice_position}
        onUpdate={(plane, pos) => updateSettings({ slice_plane: plane, slice_position: pos })}
      />
    </div>
  );
}

function extractSlice(
  fieldData: FieldData,
  plane: 'xy' | 'xz' | 'yz',
  position: number
): { width: number; height: number; data: Float32Array } {
  const { grid_x, grid_y, grid_z, values } = fieldData;

  if (plane === 'xy') {
    const z_idx = Math.floor(position * (grid_z - 1));
    const data = new Float32Array(grid_x * grid_y);
    for (let y = 0; y < grid_y; y++) {
      for (let x = 0; x < grid_x; x++) {
        const idx = z_idx * grid_y * grid_x + y * grid_x + x;
        data[y * grid_x + x] = values[idx];
      }
    }
    return { width: grid_x, height: grid_y, data };
  }
  // Implement xz and yz slices similarly
  return { width: 0, height: 0, data: new Float32Array(0) };
}

function getColorMapIndex(colorMap: string): number {
  const maps = { 'jet': 0, 'viridis': 1, 'inferno': 2, 'cool_warm': 3 };
  return maps[colorMap] ?? 0;
}

function createShaderProgram(
  gl: WebGL2RenderingContext,
  vsSource: string,
  fsSource: string
): WebGLProgram {
  const vs = gl.createShader(gl.VERTEX_SHADER)!;
  gl.shaderSource(vs, vsSource);
  gl.compileShader(vs);

  const fs = gl.createShader(gl.FRAGMENT_SHADER)!;
  gl.shaderSource(fs, fsSource);
  gl.compileShader(fs);

  const program = gl.createProgram()!;
  gl.attachShader(program, vs);
  gl.attachShader(program, fs);
  gl.linkProgram(program);

  return program;
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–4)

**Day 1: Project initialization and Rust backend scaffold**
```bash
cargo init cryogen-api
cd cryogen-api
cargo add axum tokio serde serde_json sqlx uuid chrono tower-http jsonwebtoken bcrypt
cargo add --features postgres,runtime-tokio-rustls,uuid,chrono,json sqlx
cargo add aws-sdk-s3 redis nalgebra
```
- `src/main.rs` — Axum app with CORS, tracing, graceful shutdown
- `src/config.rs` — Environment-based configuration (DATABASE_URL, JWT_SECRET, S3_BUCKET, REDIS_URL)
- `src/state.rs` — AppState struct (PgPool, Redis, S3Client)
- `src/error.rs` — ApiError enum with IntoResponse
- `Dockerfile` — Multi-stage build (builder + runtime)
- `docker-compose.yml` — PostgreSQL, Redis, MinIO (S3-compatible)

**Day 2: Database schema and migrations**
- `migrations/001_initial.sql` — All 9 tables: users, organizations, org_members, projects, simulations, simulation_jobs, components, materials, field_configs, usage_records
- `src/db/mod.rs` — Database pool initialization
- `src/db/models.rs` — All SQLx structs with FromRow derives
- Run `sqlx migrate run` to apply schema
- Seed script for initial material properties (NIST data: Cu, Al, SS304, G10)

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

### Phase 2 — Solver Core — Thermal (Days 5–11)

**Day 5: Mesh generation and material properties**
- `solver-core/` — New Rust workspace member (shared between native and WASM)
- `solver-core/src/geometry/mesh.rs` — 3D Cartesian mesh generation from component layout
- `solver-core/src/materials/mod.rs` — Material trait with temperature-dependent k(T), Cp(T)
- `solver-core/src/materials/nist.rs` — NIST cryogenic materials database loader (JSON format)
- Temperature interpolation: cubic spline for k(T) and Cp(T) from tabulated data
- Unit tests: material property lookup at various temperatures

**Day 6: Thermal conductance matrix assembly**
- `solver-core/src/thermal.rs` — ThermalSystem struct with mesh, temperatures, materials
- `solver-core/src/thermal.rs::build_system_matrix()` — Assemble sparse K matrix and Q vector
- Finite-difference stencil for conductance between neighboring nodes
- Harmonic mean for interface conductivity (handles material discontinuities)
- Tests: simple 1D conduction, verify against analytical solution

**Day 7: Boundary conditions and heat sources**
- Radiation boundary condition: Stefan-Boltzmann with linearization for Newton-Raphson
- Multi-layer insulation (MLI) effective emissivity model
- Fixed temperature (Dirichlet) boundary condition
- Convection boundary condition (for gas conduction at residual pressure)
- Volumetric and point heat sources
- Tests: radiation heat transfer validation against Stefan-Boltzmann law

**Day 8: Steady-state solver with Newton-Raphson**
- `solver-core/src/thermal.rs::solve_steady_state()` — Iterative Newton-Raphson solver
- Sparse matrix solve using conjugate gradient (sprs crate)
- Convergence check: ||ΔT|| < TOL
- Under-relaxation for stability (α = 0.5 for difficult cases)
- Tests: 2D heat transfer, 3D cryogenic chamber, compare to analytical

**Day 9: Transient (cooldown) solver**
- `solver-core/src/thermal.rs::solve_transient()` — Implicit Euler time integration
- Mass matrix M assembly: diagonal with ρ·Cp·V for each node
- Adaptive timestep based on temperature change rate
- Transient result storage with time history
- Tests: exponential cooldown, compare to analytical exp(-t/τ)

**Day 10: Solver validation — cryogenic benchmarks**
- Benchmark 1: Copper rod 300K to 4K — verify heat capacity integral
- Benchmark 2: MLI radiation shield — verify effective emissivity calculation
- Benchmark 3: Thermal link between two temperatures — verify conductance
- Benchmark 4: Cooldown time estimation — compare to lumped capacitance model
- Automated test suite with numerical tolerances

**Day 11: Component library loader**
- `solver-core/src/components/mod.rs` — Component trait and loader
- `solver-core/src/components/cryocooler.rs` — Cryocooler performance curve (cooling power vs. temperature)
- `solver-core/src/components/thermal_link.rs` — Conductive thermal links
- `solver-core/src/components/mli.rs` — MLI shield with layer count
- `solver-core/src/components/support.rs` — Support structures with thermal parasitic load
- Load component data from JSON files (S3 in production, local for tests)

### Phase 3 — Solver Core — Magnetics (Days 12–16)

**Day 12: Biot-Savart field solver for simple geometries**
- `solver-core/src/magnet/mod.rs` — MagnetCoil struct with geometry and current
- `solver-core/src/magnet/biot_savart.rs` — Analytical Biot-Savart for solenoid, Helmholtz
- On-axis field formula for solenoid (elliptic integrals)
- Off-axis field via numerical integration of current filaments
- Tests: verify solenoid field against analytical formula

**Day 13: Superconductor critical current models**
- `solver-core/src/magnet/superconductor.rs` — Jc(B,T) parametric models
- NbTi, Nb3Sn, REBCO parameter sets from vendor data
- Critical current density scaling laws
- Operating margin calculation: I_op / I_c
- Tests: verify Jc(B,T) curves match vendor datasheet plots

**Day 14: Numerical Biot-Savart for complex coil geometries**
- Discretization of coil paths (racetrack, dipole, arbitrary)
- Numerical Biot-Savart integration over current elements
- GPU acceleration potential (future: compute field map on GPU grid)
- Field map generation on 3D grid for visualization
- Tests: compare numerical vs analytical for solenoid (consistency check)

**Day 15: Quench simulation**
- `solver-core/src/magnet/quench.rs` — Normal zone propagation model
- Quench velocity calculation: v = √(k·ρ / (ρ_sc·Cp)) · f(J, Jc)
- Temperature rise during quench with dump resistor discharge
- Protection circuit analysis: maximum voltage and temperature
- Tests: verify quench velocity against literature values

**Day 16: Magnet field visualization data preparation**
- Field map export to Float32Array for WebGL rendering
- Contour level calculation for field magnitude
- Vector field decimation for display (arrow plots)
- Cross-section slicing (XY, XZ, YZ planes)
- Tests: verify field map correctness and data format

### Phase 4 — WASM Build + Frontend Foundation (Days 17–22)

**Day 17: WASM solver compilation**
- `solver-wasm/` — New workspace member for WASM target
- `solver-wasm/Cargo.toml` — wasm-bindgen, serde-wasm-bindgen dependencies
- `solver-wasm/src/lib.rs` — WASM entry points: `solve_heat_load()`, `solve_cooldown()`, `compute_magnet_field()`
- `wasm-pack build --target web --release` pipeline
- `wasm-opt -Oz` for size optimization (target <3MB gzipped)
- JavaScript wrapper: `CryogenSolver` class that loads WASM and provides async API

**Day 18: Frontend scaffold and system designer foundation**
```bash
npm create vite@latest frontend -- --template react-ts
cd frontend
npm i zustand @tanstack/react-query axios rbush
```
- `src/App.tsx` — Router, auth context, layout
- `src/stores/projectStore.ts` — Zustand store for project state
- `src/stores/designStore.ts` — System designer state (components, thermal links, shields)
- `src/components/SystemDesigner/Canvas.tsx` — SVG canvas with pan/zoom (matrix transform)
- `src/components/SystemDesigner/Grid.tsx` — Snap-to-grid overlay (configurable spacing)
- `src/lib/spatial.ts` — R-tree setup for spatial queries

**Day 19: System designer — component placement**
- `src/components/SystemDesigner/ComponentLibrary.tsx` — Categorized component palette sidebar
- `src/components/SystemDesigner/ComponentSymbol.tsx` — SVG symbols (cryocooler, shield, coil, thermal link)
- `src/components/SystemDesigner/ThermalLink.tsx` — Conductive path drawing tool
- `src/components/SystemDesigner/RadiationShield.tsx` — Radiation shield placement (cylindrical, planar)
- Drag-and-drop from library → canvas, rotation (R key), delete (Del key)
- Property editor panel: component parameters (cooling power, emissivity, etc.)

**Day 20: System designer — thermal path routing**
- `src/components/SystemDesigner/PathRouter.tsx` — Auto-routing of thermal paths between components
- Thermal connection validation (ensure continuous path to heat sink)
- Multi-select with box drag, copy/paste, move group
- Undo/redo stack (Zustand middleware)
- Keyboard shortcuts: Ctrl+Z/Y undo/redo, Ctrl+C/V copy/paste, Ctrl+S save

**Day 21: Field visualizer scaffold**
- `src/components/FieldVisualizer/FieldVisualizer.tsx` — WebGL2 canvas for field rendering
- `src/components/FieldVisualizer/ColorBar.tsx` — Color scale legend
- `src/components/FieldVisualizer/SliceControls.tsx` — XY/XZ/YZ slice selector with position slider
- GPU texture upload for field data (Float32Array → R32F texture)
- Fragment shader with color mapping (jet, viridis, inferno)
- Tests: render synthetic field data, verify color mapping

**Day 22: Field visualizer — contour rendering**
- Contour line extraction in fragment shader (fract-based algorithm)
- Adjustable contour levels (5, 10, 20 levels)
- Temperature contour labels (overlay SVG text on WebGL canvas)
- Magnetic field vector overlay (arrow field at sampled points)
- Pan/zoom on field view independent of system designer
- `src/stores/fieldStore.ts` — Zustand store for field data and visualization settings

### Phase 5 — API + Job Orchestration (Days 23–28)

**Day 23: Simulation API endpoints**
- `src/api/handlers/simulation.rs` — Create simulation, get simulation, list simulations, cancel simulation
- System validation: check component connectivity, thermal path completeness
- Component count extraction and WASM/server routing logic
- Plan-based limits enforcement (academic: 100 components, pro: unlimited)

**Day 24: Server-side simulation worker**
- `src/workers/simulation_worker.rs` — Redis job consumer, runs native solver
- `src/workers/mod.rs` — Worker pool management (configurable concurrency)
- Progress streaming: worker publishes progress → Redis pub/sub → WebSocket to client
- Error handling: convergence failures, timeouts, memory limits
- S3 result upload with presigned URL generation for client download

**Day 25: WebSocket for live simulation progress**
- `src/api/ws/mod.rs` — Axum WebSocket upgrade handler
- `src/api/ws/simulation_progress.rs` — Subscribe to simulation progress channel
- Client receives: `{ progress_pct, current_iteration, temperature_residual, timestep }`
- Connection management: heartbeat ping/pong, automatic cleanup on disconnect
- Frontend: `src/hooks/useSimulationProgress.ts` — React hook for WebSocket subscription

**Day 26: Component library API**
- `src/api/handlers/components.rs` — Search components (parametric + full-text), get component, download data
- S3 integration for component data file storage and retrieval
- Component categories: cryocooler, thermal_shield, mli, thermal_link, superconductor, support_structure, vessel
- Parametric search: filter by manufacturer, category, cooling power at 4K, operating temperature range
- Pagination with cursor-based scrolling for large result sets

**Day 27: Material properties API**
- `src/api/handlers/materials.rs` — Search materials, get material properties, plot k(T) and Cp(T)
- Material data loader from S3 (JSON format with temperature-dependent tables)
- Temperature range validation and interpolation endpoint
- Material comparison tool: overlay multiple k(T) curves
- Tests: verify NIST data integrity and interpolation accuracy

**Day 28: Design export endpoints**
- `src/api/handlers/export.rs` — Export system as SVG/PNG, export thermal analysis report (PDF)
- System thumbnail generation: server-side SVG-to-PNG for dashboard cards
- Report generation: LaTeX template with simulation results, field plots, component list
- Auto-save: frontend debounced PATCH every 5 seconds on design changes
- Design version history (store snapshots in S3, metadata in PostgreSQL)

### Phase 6 — Results + Post-Processing UI (Days 29–32)

**Day 29: Simulation result processing**
- Result parsing: binary format (Float32Array) → structured data (temperature field, time history)
- Result caching: hot results in Redis (TTL 1 hour), cold results in S3
- Presigned S3 URLs for client-side download of large result sets
- Result summary generation: peak temperature, minimum temperature, total heat load, cooldown time

**Day 30: Temperature distribution visualization**
- `src/components/FieldVisualizer/TemperatureContour.tsx` — Temperature contour overlay on system designer
- Component color-coding by temperature (blue=cold, red=hot)
- Temperature probe tool: click to read temperature at specific location
- Temperature profile plot along selected path
- Export temperature data as CSV for external analysis

**Day 31: Magnet field visualization**
- `src/components/FieldVisualizer/MagnetFieldView.tsx` — Magnetic field magnitude contour
- Field line tracing (streamlines following B vector field)
- Operating margin indicator on coil components (green=safe, yellow=marginal, red=quench risk)
- Field uniformity analysis for MRI/NMR applications (ppm deviation from center field)
- Export field map as VTK format for ParaView

**Day 32: System templates and examples**
- `src/data/templates/` — Pre-built system templates:
  - Single-stage cryocooler (4K cold head)
  - Two-stage cryocooler (50K + 4K stages)
  - Dilution refrigerator (basic 3-stage)
  - Superconducting solenoid magnet (4.2K operation)
  - Liquid helium dewar with MLI
- Template gallery page with preview thumbnails and one-click "Use Template"
- Example simulations pre-run to show expected results

### Phase 7 — Billing + Plan Enforcement (Days 33–36)

**Day 33: Stripe integration**
- `src/api/handlers/billing.rs` — Create checkout session, create customer portal session, get subscription status
- `src/api/handlers/webhooks/stripe.rs` — Handle subscription events: `checkout.session.completed`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.payment_failed`
- Plan mapping:
  - Academic ($99/mo): 100 components, basic materials, 20 server simulations/month
  - Pro ($299/mo): Unlimited components, full materials + cryocoolers, 200 server simulations/month, quench analysis
  - Enterprise (custom): On-premise, custom components, unlimited usage, white-label

**Day 34: Usage tracking and plan limits**
- `src/middleware/plan_limits.rs` — Middleware checking plan limits before simulation execution
- `src/services/usage.rs` — Track server simulation count per billing period
- Usage record insertion after each server-side simulation completes
- Usage dashboard: `src/api/handlers/usage.rs` — Current period usage, historical usage
- Approaching-limit warnings at 80% and 100% of server simulations

**Day 35: Billing UI**
- `frontend/src/pages/Billing.tsx` — Plan comparison table, current plan display, usage meter
- `frontend/src/components/billing/PlanCard.tsx` — Plan feature comparison cards
- `frontend/src/components/billing/UsageMeter.tsx` — Server simulation usage bar with threshold indicators
- Upgrade/downgrade flow via Stripe Customer Portal
- Upgrade prompt modals when hitting plan limits (e.g., "System has 150 components. Upgrade to Pro.")

**Day 36: Feature gating**
- Gate quench simulation behind Pro plan
- Gate custom component upload behind Enterprise plan
- Gate field export (VTK) behind Pro plan
- Gate design collaboration features behind Enterprise plan
- Show locked feature indicators with upgrade CTAs in UI
- Admin endpoint: override plan for specific users (beta testers, academic licenses)

### Phase 7 — Solver Validation + Testing (Days 37–40)

**Day 37: Thermal solver validation**
- Benchmark 1: Copper rod 300K to 77K steady-state — verify against tabulated k(T) integral
- Benchmark 2: MLI shield between 300K and 4K — verify Q_rad matches literature (1-10 W/m² range)
- Benchmark 3: Cooldown of aluminum mass — compare to experimental data or detailed simulation
- Benchmark 4: Thermal link conductance — verify against vendor spec sheets
- Automated test suite: `solver-core/tests/thermal_benchmarks.rs` with assertions within 5% tolerance

**Day 38: Magnet solver validation**
- Benchmark 5: Solenoid field on-axis — compare to analytical formula (within 0.1% at center)
- Benchmark 6: Helmholtz coil uniformity — verify ppm-level field uniformity in central region
- Benchmark 7: NbTi operating margin — verify Jc(B,T) against vendor critical current data
- Benchmark 8: Quench propagation velocity — compare to literature values for NbTi/Cu composite
- Tests: ensure numerical Biot-Savart converges to analytical for simple geometries

**Day 39: Integration testing**
- End-to-end test: create project → place components → run simulation → view results → export report
- API integration tests: auth → project CRUD → simulation → results
- WASM solver test: load in headless browser (Playwright), run simulation, verify results match native solver
- WebSocket test: connect → subscribe → receive progress → receive completion
- Concurrent simulation test: submit 10 simultaneous server-side jobs, verify all complete

**Day 40: Performance testing and optimization**
- WASM solver benchmarks: measure time for 50-comp, 150-comp, 200-comp systems in Chrome/Firefox/Safari
- Target: 200-component steady-state < 3 seconds in WASM
- Server solver benchmarks: measure throughput (simulations/minute) with various component counts
- Frontend rendering: measure field visualizer FPS with various grid resolutions
- Target: 60 FPS pan/zoom with 100×100×50 field grid
- Memory profiling: ensure WASM solver doesn't leak, frontend GC behavior is healthy
- Load testing: 30 concurrent users running simulations via k6

### Phase 8 — Deployment + Polish + Launch (Days 41–42)

**Day 41: Docker and Kubernetes configuration**
- `Dockerfile` — Multi-stage Rust build (cargo-chef for layer caching)
- `docker-compose.yml` — Full local dev stack (API, PostgreSQL, Redis, MinIO, frontend)
- `k8s/` — Kubernetes manifests:
  - `api-deployment.yaml` — API server (3 replicas, HPA)
  - `worker-deployment.yaml` — Simulation workers (auto-scaling based on queue depth)
  - `postgres-statefulset.yaml` — PostgreSQL with PVC
  - `redis-deployment.yaml` — Redis for job queue and pub/sub
  - `ingress.yaml` — NGINX ingress with TLS
- Health check endpoints: `/health/live`, `/health/ready`
- Monitoring: Prometheus metrics, Grafana dashboards, Sentry error tracking
- Structured logging (tracing + JSON) for all API requests and solver events

**Day 42: Launch preparation and final testing**
- End-to-end smoke test in production environment
- Database backup and restore verification
- Rate limiting: 100 req/min per user, 5 simulations/min per user
- Security audit: JWT validation, SQL injection (SQLx prevents this), CORS configuration, CSP headers
- Landing page with product overview, pricing, demo video, and signup
- Documentation: getting started guide, component library reference, thermal analysis best practices
- Deploy to production, enable monitoring alerts, soft launch to beta users

---

## Critical Files

```
cryogen/
├── solver-core/                           # Shared solver library (native + WASM)
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── geometry/
│   │   │   ├── mod.rs                     # Mesh and geometry types
│   │   │   └── mesh.rs                    # 3D Cartesian mesh generator
│   │   ├── materials/
│   │   │   ├── mod.rs                     # Material trait and database
│   │   │   ├── nist.rs                    # NIST cryogenic materials
│   │   │   └── interpolation.rs           # Cubic spline for k(T), Cp(T)
│   │   ├── thermal.rs                     # Thermal solver (steady-state + transient)
│   │   ├── magnet/
│   │   │   ├── mod.rs                     # Magnet field solver
│   │   │   ├── biot_savart.rs             # Analytical and numerical Biot-Savart
│   │   │   ├── superconductor.rs          # Jc(B,T) models
│   │   │   └── quench.rs                  # Quench propagation model
│   │   └── components/
│   │       ├── mod.rs                     # Component trait and loader
│   │       ├── cryocooler.rs              # Cryocooler performance curves
│   │       ├── thermal_link.rs            # Conductive links
│   │       ├── mli.rs                     # MLI shield
│   │       └── support.rs                 # Support structures
│   └── tests/
│       ├── thermal_benchmarks.rs          # Thermal solver validation
│       └── magnet_benchmarks.rs           # Magnet solver validation
│
├── solver-wasm/                           # WASM compilation target
│   ├── Cargo.toml
│   └── src/
│       └── lib.rs                         # WASM entry points (wasm-bindgen)
│
├── cryogen-api/                           # Rust API server
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
│   │   │   │   ├── components.rs          # Component library search/get
│   │   │   │   ├── materials.rs           # Material properties API
│   │   │   │   ├── billing.rs             # Stripe checkout/portal
│   │   │   │   ├── usage.rs               # Usage tracking
│   │   │   │   ├── export.rs              # SVG/PNG/PDF export
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
│   │       └── simulation_worker.rs       # Server-side simulation execution
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
│   │   │   ├── designStore.ts             # System designer state
│   │   │   └── fieldStore.ts              # Field visualizer state
│   │   ├── hooks/
│   │   │   ├── useSimulationProgress.ts   # WebSocket hook for live progress
│   │   │   └── useWasmSolver.ts           # WASM solver hook (load + run)
│   │   ├── lib/
│   │   │   ├── api.ts                     # Axios API client
│   │   │   ├── spatial.ts                 # R-tree for spatial queries
│   │   │   └── wasmLoader.ts              # WASM bundle loading
│   │   ├── pages/
│   │   │   ├── Dashboard.tsx              # Project list
│   │   │   ├── Designer.tsx               # Main designer (system + field split)
│   │   │   ├── Components.tsx             # Component library browser
│   │   │   ├── Materials.tsx              # Material properties browser
│   │   │   ├── Templates.tsx              # System template gallery
│   │   │   ├── Billing.tsx                # Plan management
│   │   │   ├── Login.tsx                  # Auth pages
│   │   │   └── Register.tsx
│   │   ├── components/
│   │   │   ├── SystemDesigner/
│   │   │   │   ├── Canvas.tsx             # SVG canvas with pan/zoom
│   │   │   │   ├── Grid.tsx               # Snap grid overlay
│   │   │   │   ├── ComponentLibrary.tsx   # Component palette sidebar
│   │   │   │   ├── ComponentSymbol.tsx    # SVG component renderer
│   │   │   │   ├── ThermalLink.tsx        # Thermal path drawing
│   │   │   │   ├── RadiationShield.tsx    # Shield placement
│   │   │   │   ├── PropertyPanel.tsx      # Component property editor
│   │   │   │   └── PathRouter.tsx         # Auto-routing thermal paths
│   │   │   ├── FieldVisualizer/
│   │   │   │   ├── FieldVisualizer.tsx    # Main WebGL viewer
│   │   │   │   ├── ColorBar.tsx           # Color scale legend
│   │   │   │   ├── SliceControls.tsx      # XY/XZ/YZ slice selector
│   │   │   │   ├── TemperatureContour.tsx # Temperature field view
│   │   │   │   └── MagnetFieldView.tsx    # Magnetic field view
│   │   │   ├── billing/
│   │   │   │   ├── PlanCard.tsx
│   │   │   │   └── UsageMeter.tsx
│   │   │   └── common/
│   │   │       ├── SimulationToolbar.tsx  # Run/stop/analysis selector
│   │   │       └── ComponentSearch.tsx    # Component search component
│   │   └── data/
│   │       └── templates/                 # Pre-built system templates (JSON)
│   └── public/
│       └── wasm/                          # WASM solver bundle (loaded at runtime)
│
├── k8s/                                   # Kubernetes manifests
│   ├── api-deployment.yaml
│   ├── worker-deployment.yaml
│   ├── postgres-statefulset.yaml
│   ├── redis-deployment.yaml
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

### Benchmark 1: Copper Rod Steady-State Heat Transfer (300K to 77K)

**System:** OFHC copper rod, length = 10 cm, diameter = 1 cm, one end at 300K, other end at 77K

**Expected:** Heat flow Q = ∫(k(T)dT) · A / L. For OFHC Cu from NIST data: Q ≈ **12.3 W** (numerical integration of k(T) from 77K to 300K)

**Tolerance:** < 2% (depends on k(T) interpolation accuracy)

### Benchmark 2: MLI Radiation Shield Heat Load (300K to 4K)

**System:** 30-layer MLI between 300K outer vessel and 4K cold stage, effective emissivity ε_eff ≈ 0.002, surface area = 0.1 m²

**Expected:** Q_rad = σ · A · ε_eff · (T_hot⁴ - T_cold⁴) = 5.67e-8 × 0.1 × 0.002 × (300⁴ - 4⁴) ≈ **0.92 W**

**Tolerance:** < 5% (depends on ε_eff model accuracy)

### Benchmark 3: Cooldown Time Estimation (Lumped Capacitance)

**System:** 1 kg aluminum mass cooling from 300K to 77K with 10W average cooling power, using NIST Cp(T) for aluminum

**Expected:** τ_cooldown = ∫(m·Cp(T)dT) / P_cool ≈ **45 minutes** (numerical integration of heat capacity)

**Tolerance:** < 10% (lumped capacitance approximation vs. distributed thermal model)

### Benchmark 4: Solenoid Magnetic Field On-Axis

**Circuit:** Solenoid with R = 0.05 m, L = 0.2 m, N = 1000 turns, I = 10 A

**Expected at center (z=0):** B_z = μ₀·N·I / L · [L/2 / √((L/2)² + R²)] ≈ **0.0257 T** (25.7 mT)

**Tolerance:** < 0.1% (analytical formula exact for on-axis)

### Benchmark 5: NbTi Operating Margin at 4.2K

**Circuit:** NbTi wire, T = 4.2 K, B_peak = 5 T, I_op = 100 A, A_conductor = 0.1 mm²

**Expected:** Jc(5T, 4.2K) ≈ **2.5×10⁹ A/m²** (from NbTi scaling law), J_op = 100 / (0.1×10⁻⁶) = 1×10⁹ A/m², Margin = J_op/Jc ≈ **0.40** (safe, <1.0)

**Tolerance:** Jc < 10% (vendor data variation), Margin calculation < 5%

---

## Verification

### Integration Testing Checklist

1. **Auth flow** — Register → login → JWT issued → authorized API calls → token refresh → logout
2. **Project CRUD** — Create → update design → save → reload → verify design state preserved
3. **System design** — Place components → connect thermal links → validate connectivity → verify system valid
4. **WASM simulation** — Small system (50 components) → WASM solver → results returned → field displays
5. **Server simulation** — Large system (250 components) → job queued → worker picks up → WebSocket progress → results in S3
6. **Field visualizer** — Load temperature field → pan/zoom → slice control → verify contours correct
7. **Component library** — Search "Cryomech PT410" → results returned → select component → use in design
8. **Material properties** — Select OFHC copper → plot k(T) curve → verify matches NIST data
9. **Magnet field** — Design solenoid → run field simulation → verify field on-axis matches analytical
10. **Plan limits** — Academic user → system with 150 components → blocked with upgrade prompt
11. **Billing** — Subscribe to Pro → Stripe checkout → webhook → plan updated → limits lifted
12. **Concurrent users** — 10 users simultaneously running server simulations → all complete correctly
13. **Template usage** — Select "Two-Stage Cryocooler" template → project created → simulate → expected results
14. **Export** — Run simulation → export results as PDF report → verify report contains correct data
15. **Error handling** — Non-convergent system → meaningful error message with suggestions → no crash

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
    COUNT(s.id) as server_sims_count,
    CASE u.plan
        WHEN 'academic' THEN 20
        WHEN 'pro' THEN 200
        ELSE 999999
    END as limit_sims
FROM users u
LEFT JOIN simulations s ON s.user_id = u.id
    AND s.execution_mode = 'server'
    AND s.created_at >= DATE_TRUNC('month', CURRENT_DATE)
GROUP BY u.id, u.email, u.plan
ORDER BY server_sims_count DESC;

-- 5. Component popularity
SELECT c.name, c.manufacturer, c.category,
    c.usage_count,
    COUNT(DISTINCT p.id) as projects_using
FROM components c
LEFT JOIN projects p ON p.design_data::text ILIKE '%' || c.name || '%'
WHERE c.is_builtin = true
GROUP BY c.id
ORDER BY c.usage_count DESC
LIMIT 20;
```

### Performance Benchmarks

| Operation | Target | Method |
|-----------|--------|--------|
| WASM solver: 50-comp steady-state | <1s | Browser benchmark (Chrome DevTools) |
| WASM solver: 200-comp steady-state | <3s | Browser benchmark |
| Server solver: 500-comp steady-state | <10s | Server timing, 4 cores |
| Server solver: 200-comp transient (100 steps) | <20s | Server timing, 4 cores |
| Field visualizer: 100×100×50 grid | 60 FPS | Chrome FPS counter during pan/zoom |
| Field visualizer: 200×200×100 grid | 30 FPS | Chrome FPS counter during pan/zoom |
| System designer: 200 components | 60 FPS | SVG rendering performance |
| API: create simulation | <200ms | p95 latency (Prometheus) |
| API: search components | <100ms | p95 latency with 1K components in DB |
| WASM bundle load (cached) | <500ms | Service worker cache hit |
| WASM bundle load (cold) | <4s | CDN delivery, 3MB gzipped |
| WebSocket: progress latency | <100ms | Time from worker emit to client receive |

---

## Deployment Architecture

### Infrastructure

```
┌─────────────────────────────────────────────────┐
│                  CloudFront CDN                   │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────┐ │
│  │ Frontend SPA │  │ WASM Bundle │  │ Component│ │
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
│ Multi-AZ     │ │ Cluster mode │ │  Components) │
└──────────────┘ └──────┬───────┘ └──────────────┘
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ Sim Worker   │ │ Sim Worker   │ │ Sim Worker   │
│ (Rust native)│ │ (Rust native)│ │ (Rust native)│
│ c7g.2xlarge  │ │ c7g.2xlarge  │ │ c7g.2xlarge  │
│ HPA: 2-15    │ │              │ │              │
└──────────────┘ └──────────────┘ └──────────────┘
```

### Scaling Strategy

- **API servers**: Horizontal pod autoscaler (HPA) at 70% CPU, min 3, max 10
- **Simulation workers**: HPA based on Redis queue depth — scale up when queue > 3 jobs, scale down when idle 10 min
- **Worker instances**: AWS c7g.2xlarge (8 vCPU, 16GB RAM) for compute-optimized thermal/magnet solving
- **Database**: RDS PostgreSQL r6g.xlarge with read replicas for component search queries
- **Redis**: ElastiCache r6g.large cluster with 1 primary + 2 replicas

### S3 Lifecycle Policy

```json
{
  "Rules": [
    {
      "ID": "simulation-results-lifecycle",
      "Prefix": "results/",
      "Status": "Enabled",
      "Transitions": [
        { "Days": 60, "StorageClass": "STANDARD_IA" },
        { "Days": 180, "StorageClass": "GLACIER" }
      ],
      "Expiration": { "Days": 730 }
    },
    {
      "ID": "component-data-keep",
      "Prefix": "components/",
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
| Advanced FEM Thermal Solver | Full 3D finite-element thermal solver with tetrahedral meshing for complex geometries, non-Cartesian shapes, and contact resistance modeling for realistic assemblies | High |
| Cryogenic Fluid Flow Simulation | Two-phase flow modeling in transfer lines, helium liquefier cycle analysis (Collins, Claude), phase separator and heat exchanger design with REFPROP thermodynamic properties | High |
| Dilution Refrigerator Full Model | Multi-stage dilution refrigerator simulation with still, cold plate, mixing chamber thermal budgets, He-3/He-4 circulation modeling, and wiring thermal load analysis | High |
| Vibration Isolation Thermal Design | Coupled vibration-thermal analysis for quantum computing platforms, thermal conductance vs. mechanical isolation trade-off optimization for support structures | Medium |
| Iron Yoke FEM Magnetics | Full FEM electromagnetic solver with ferromagnetic materials (iron, mu-metal) for dipole/quadrupole magnets with yoke, fringe field analysis, and magnetic shielding design | Medium |
| Quench Protection Circuit Design | Automated dump resistor sizing, heater placement optimization, voltage distribution during quench, and energy extraction analysis with iterative circuit simulation | Medium |
| Material Database Expansion | Extended materials library including ceramics (alumina, sapphire), polymers (PEEK, Teflon), advanced composites (carbon fiber), and user-contributed material data with peer review | Low |
| Real-Time Collaboration | CRDT-based system design editing with cursor presence, commenting on components, shared simulation results, and version control for team cryogenic engineering projects | Low |
