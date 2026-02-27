# 54. ThermoCraft — Thermal Management and Heat Transfer Simulation Platform

## Implementation Plan

**MVP Scope:** Browser-based 3D geometry builder with STEP import and component placement (IC packages, heat sinks, fans) rendered via Three.js/WebGL, conjugate heat transfer solver implementing finite volume method (FVM) for steady-state and transient conduction with anisotropic materials (PCB copper layers, graphite sheets), simplified natural convection (empirical correlations for common enclosure geometries), surface-to-surface radiation with view factor computation compiled to WebAssembly for client-side thermal analysis of circuits ≤100 components and server-side Rust-native execution for larger systems, component library with 50+ standard IC packages (BGA, QFN, TO-220) and 20 heat sink geometries with thermal resistance models, interactive 3D temperature contour visualization rendered via WebGL with color-mapped surfaces and cross-sectional views, maximum junction temperature (Tj) prediction with thermal resistance path tracing, thermal management reports with compliance checks against component temperature limits, Stripe billing with three tiers (Free / Pro $129/mo / Advanced $299/user/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| Thermal Solver | Rust (native + WASM) | Custom FVM conduction solver + natural convection correlations |
| WASM Compilation | `wasm-pack` + `wasm-bindgen` | Client-side solver for systems ≤100 components |
| Battery/ML Service | Python 3.12 (FastAPI) | PyBaMM electrochemical models, optimization algorithms |
| Database | PostgreSQL 16 | Projects, users, simulations, component library metadata |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | Mesh data, temperature results, STEP CAD files, thermal reports |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| 3D Visualization | Three.js + WebGL 2.0 | Temperature contours, cross-sections, component placement |
| Geometry Import | STEP parser (opencascade.js) | CAD geometry import for enclosures and custom parts |
| Real-time | WebSocket (Axum) | Live simulation progress, convergence monitoring, temperature updates |
| Job Queue | Redis 7 + Tokio tasks | Server-side simulation job management, batch thermal sweeps |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | WASM bundle delivery, static frontend assets, component library |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, solver performance, error tracking |
| CI/CD | GitHub Actions | WASM build pipeline, Rust tests, Docker image push |

### Key Architecture Decisions

1. **Hybrid client/server solver with 100-component threshold**: Systems with ≤100 components (covers 80%+ of typical electronics thermal analysis: smartphone PCB, LED driver, small power supply) run entirely in the browser via WASM, providing instant feedback with zero server cost. Larger systems (data center racks, EV battery packs with 300+ cells, complex multi-board assemblies) are submitted to the Rust-native server solver which handles thousands of thermal elements and transient simulations. The threshold is configurable per plan tier.

2. **Finite Volume Method (FVM) for conduction rather than FEM**: FVM provides better conservation properties for heat transfer and is more efficient for Cartesian-dominant geometries typical in electronics (rectangular PCBs, chip packages, heat sinks). FVM naturally handles conjugate interfaces between solid domains without requiring continuity constraints. For arbitrary geometry (STEP imports), we use octree mesh refinement with cut-cell boundary handling, similar to commercial tools like FloTHERM.

3. **Empirical natural convection correlations for MVP rather than full CFD**: Full Navier-Stokes CFD adds 100x computational cost and requires turbulence modeling expertise. For MVP, we implement empirical correlations (Churchill-Chu for vertical plates, Nusselt number correlations for horizontal surfaces, enclosure correlations from literature) which provide 10-20% accuracy for typical electronics cooling scenarios. Full CFD (SIMPLE algorithm with k-ε turbulence) is post-MVP.

4. **Three.js for 3D visualization with GPU-accelerated color mapping**: Three.js provides mature WebGL abstraction for 3D rendering, built-in camera controls, and efficient mesh handling. Temperature data is uploaded to GPU as vertex colors or textures, enabling real-time 60fps interactive visualization of 100K+ mesh cells. Cross-sectional views use fragment shader clipping planes. This decouples thermal computation from visualization.

5. **Component library with calibrated thermal models stored in PostgreSQL + S3**: Each component (IC package, heat sink, fan) has a simplified thermal model: thermal resistances (junction-to-case, case-to-ambient), geometry, footprint, power dissipation range. Detailed meshes for standard components (BGA solder balls, heat sink fins) are pre-generated and stored in S3, while PostgreSQL holds searchable metadata (package type, manufacturer, thermal resistance values, power ratings). This allows instant component insertion without remeshing.

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
    plan TEXT NOT NULL DEFAULT 'free',  -- free | pro | advanced
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX users_email_idx ON users(email);
CREATE INDEX users_stripe_idx ON users(stripe_customer_id);

-- Organizations (for team plans)
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

-- Thermal Projects (geometry + component placement + boundary conditions)
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    project_type TEXT NOT NULL DEFAULT 'electronics',  -- electronics | battery | datacenter | heat_exchanger
    geometry_data JSONB NOT NULL DEFAULT '{}',  -- 3D geometry, component placements, mesh settings
    boundary_conditions JSONB DEFAULT '{}',  -- Ambient temp, convection coefficients, heat loads
    settings JSONB DEFAULT '{}',  -- Mesh density, solver tolerance, material properties
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

-- Thermal Simulations
CREATE TABLE simulations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    analysis_type TEXT NOT NULL,  -- steady_state | transient | parametric_sweep
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | meshing | running | completed | failed | cancelled
    execution_mode TEXT NOT NULL DEFAULT 'wasm',  -- wasm | server
    parameters JSONB NOT NULL DEFAULT '{}',  -- Analysis-specific parameters (time range, sweep variables)
    component_count INTEGER NOT NULL DEFAULT 0,
    mesh_cell_count INTEGER,
    results_url TEXT,  -- S3 URL for temperature field data
    results_summary JSONB,  -- Quick-access summary (max Tj, hot spots, thermal margins)
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
    memory_mb INTEGER DEFAULT 4096,
    priority INTEGER DEFAULT 0,  -- Higher = more urgent
    progress_pct REAL DEFAULT 0.0,
    convergence_data JSONB DEFAULT '[]',  -- [{iteration, residual, max_temp_change}]
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX jobs_sim_idx ON simulation_jobs(simulation_id);
CREATE INDEX jobs_worker_idx ON simulation_jobs(worker_id);

-- Component Library (IC packages, heat sinks, fans, TIMs)
CREATE TABLE components (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    manufacturer TEXT,
    part_number TEXT,
    category TEXT NOT NULL,  -- ic_package | heat_sink | fan | tim | pcb | custom
    subcategory TEXT,  -- e.g., "BGA" for ic_package, "extruded" for heat_sink
    thermal_model JSONB NOT NULL,  -- Rjc, Rca, geometry, thermal conductivity, etc.
    geometry_url TEXT,  -- S3 URL to pre-meshed geometry or STEP file
    symbol_data JSONB,  -- 2D symbol for schematic placement
    parameters JSONB DEFAULT '{}',  -- Searchable parameters (power rating, package size, pin count)
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
CREATE INDEX components_part_trgm_idx ON components USING gin(part_number gin_trgm_ops);
CREATE INDEX components_tags_idx ON components USING gin(tags);

-- Material Library
CREATE TABLE materials (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    category TEXT NOT NULL,  -- metal | semiconductor | ceramic | polymer | pcb | tim | fluid
    thermal_conductivity REAL NOT NULL,  -- W/m·K (isotropic), or array for anisotropic
    thermal_conductivity_tensor REAL[],  -- [kx, ky, kz] for anisotropic materials
    density REAL NOT NULL,  -- kg/m³
    specific_heat REAL NOT NULL,  -- J/kg·K
    emissivity REAL DEFAULT 0.9,  -- For radiation
    temperature_range REAL[] DEFAULT '{-273, 1000}',  -- [min, max] valid temperature in °C
    is_builtin BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX materials_category_idx ON materials(category);

-- Saved Views (visualization configurations)
CREATE TABLE visualization_configs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    simulation_id UUID NOT NULL REFERENCES simulations(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    camera_position JSONB NOT NULL,  -- {x, y, z, target, fov}
    color_scale JSONB NOT NULL,  -- {min, max, colormap: "jet" | "viridis" | "thermal"}
    visible_components UUID[] DEFAULT '{}',  -- Array of component IDs
    cross_section_plane JSONB,  -- {normal: [x,y,z], position: [x,y,z]}
    annotations JSONB DEFAULT '[]',  -- [{position, label, value}]
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX viz_sim_idx ON visualization_configs(simulation_id);

-- Usage Tracking (for billing enforcement)
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,  -- simulation_minutes | component_count | storage_gb
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
    pub geometry_data: serde_json::Value,
    pub boundary_conditions: serde_json::Value,
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
    pub parameters: serde_json::Value,
    pub component_count: i32,
    pub mesh_cell_count: Option<i32>,
    pub results_url: Option<String>,
    pub results_summary: Option<serde_json::Value>,
    pub error_message: Option<String>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub wall_time_ms: Option<i32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Component {
    pub id: Uuid,
    pub name: String,
    pub manufacturer: Option<String>,
    pub part_number: Option<String>,
    pub category: String,
    pub subcategory: Option<String>,
    pub thermal_model: serde_json::Value,
    pub geometry_url: Option<String>,
    pub symbol_data: Option<serde_json::Value>,
    pub parameters: serde_json::Value,
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
    pub thermal_conductivity: f64,  // W/m·K
    pub thermal_conductivity_tensor: Option<Vec<f64>>,  // [kx, ky, kz]
    pub density: f64,  // kg/m³
    pub specific_heat: f64,  // J/kg·K
    pub emissivity: f64,
    pub temperature_range: Vec<f64>,  // [min, max] °C
    pub is_builtin: bool,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct SimulationParams {
    pub analysis_type: AnalysisType,
    pub ambient_temp: f64,  // °C
    pub natural_convection: Option<ConvectionParams>,
    pub radiation: Option<RadiationParams>,
    pub transient: Option<TransientParams>,
    pub solver_options: SolverOptions,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum AnalysisType {
    SteadyState,
    Transient,
    ParametricSweep,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct ConvectionParams {
    pub enabled: bool,
    pub correlation_type: String,  // "churchill_chu" | "nusselt" | "custom"
    pub reference_length: f64,  // m (characteristic length for dimensionless numbers)
}

#[derive(Debug, Deserialize, Serialize)]
pub struct RadiationParams {
    pub enabled: bool,
    pub ambient_temperature: f64,  // K
    pub view_factor_method: String,  // "hemicube" | "monte_carlo"
}

#[derive(Debug, Deserialize, Serialize)]
pub struct TransientParams {
    pub duration: f64,  // seconds
    pub time_step: f64,  // seconds
    pub adaptive_stepping: bool,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct SolverOptions {
    pub tolerance: f64,  // Convergence tolerance for temperature (K)
    pub max_iterations: u32,
    pub under_relaxation: f64,  // 0.0 - 1.0
}
```

---

## Solver Architecture Deep-Dive

### Governing Equations and Discretization

ThermoCraft's thermal solver implements the **heat diffusion equation** (conduction) coupled with **boundary conditions** for convection and radiation. For a 3D domain with temperature field T(x,y,z,t), the governing equation is:

```
ρc_p ∂T/∂t = ∇·(k∇T) + Q̇

where:
  ρ = density (kg/m³)
  c_p = specific heat (J/kg·K)
  k = thermal conductivity (W/m·K, may be tensor for anisotropic materials)
  Q̇ = volumetric heat generation (W/m³)
```

**Steady-state** (∂T/∂t = 0) simplifies to Laplace/Poisson equation:
```
∇·(k∇T) + Q̇ = 0
```

**Boundary conditions:**
1. **Prescribed temperature** (Dirichlet): T = T_specified on boundary
2. **Prescribed heat flux** (Neumann): -k(∂T/∂n) = q" on boundary
3. **Convection** (Robin): -k(∂T/∂n) = h(T - T_∞) where h is convection coefficient
4. **Radiation**: -k(∂T/∂n) = εσ(T⁴ - T_surr⁴) where ε is emissivity, σ is Stefan-Boltzmann constant

**Finite Volume Method (FVM) discretization:**
The domain is divided into control volumes (cells). For cell *i* with neighbors *j*, the discretized steady-state equation is:

```
Σ_j (k_ij A_ij / d_ij) (T_j - T_i) + Q̇_i V_i = 0

where:
  k_ij = effective conductivity at interface
  A_ij = interface area
  d_ij = distance between cell centers
  V_i = cell volume
```

This yields a **sparse linear system**: **[A]{T} = {b}** where A is conductance matrix, T is temperature vector, b is heat source vector.

For **transient** analysis, implicit Euler time integration:
```
(ρc_p V_i / Δt)(T_i^{n+1} - T_i^n) = Σ_j (k_ij A_ij / d_ij) (T_j^{n+1} - T_i^{n+1}) + Q̇_i V_i

Rearranged:
[(ρc_p V_i / Δt) + Σ_j (k_ij A_ij / d_ij)] T_i^{n+1} - Σ_j (k_ij A_ij / d_ij) T_j^{n+1} = (ρc_p V_i / Δt) T_i^n + Q̇_i V_i
```

**Natural convection** is modeled via empirical correlations rather than solving Navier-Stokes (full CFD). For vertical surfaces:
```
Nu = 0.68 + 0.67 Ra^(1/4) / [1 + (0.492/Pr)^(9/16)]^(4/9)  (Churchill-Chu correlation)

where:
  Nu = Nusselt number = hL/k_fluid
  Ra = Rayleigh number = gβΔTL³/(να)
  Pr = Prandtl number = ν/α
  h = convection coefficient
```

This allows computing *h* for each surface based on local temperature and orientation, avoiding expensive CFD solve.

### Client/Server Split (100-Component Threshold)

```
Project created → Component count extracted
    │
    ├── ≤100 components → WASM solver (browser)
    │   ├── Mesh generated in browser (octree)
    │   ├── Solver runs client-side
    │   ├── Results displayed immediately
    │   └── No server cost
    │
    └── >100 components → Server solver (Rust native)
        ├── Job queued via Redis
        ├── Mesh generated on server (higher resolution)
        ├── Worker picks up, runs native solver
        ├── Progress streamed via WebSocket
        └── Results stored in S3, URL returned
```

The 100-component threshold was chosen because:
- WASM solver with 10K-50K cells handles typical electronics PCB (10-50 components) in <5 seconds
- 100 components covers: laptop motherboard (60-80 ICs), LED driver (20-30 components), small power supply (40-60 components)
- Above 100 components: data center racks (1000+ servers), EV battery packs (300+ cells), multi-board systems → need server compute

### WASM Compilation Pipeline

```toml
# solver-wasm/Cargo.toml
[package]
name = "thermocraft-solver-wasm"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
wasm-bindgen = "0.2"
serde = { version = "1", features = ["derive"] }
serde-wasm-bindgen = "0.6"
js-sys = "0.3"
nalgebra = "0.32"  # Linear algebra for sparse matrix
sprs = "0.11"  # Sparse matrix operations
log = "0.4"
console_log = "1"

[profile.release]
opt-level = "z"       # Optimize for size
lto = true            # Link-time optimization
codegen-units = 1     # Single codegen unit for better optimization
strip = true          # Strip debug symbols
```

```yaml
# .github/workflows/wasm-build.yml
name: Build WASM Thermal Solver
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
      - run: wasm-opt -Oz solver-wasm/pkg/thermocraft_solver_wasm_bg.wasm -o solver-wasm/pkg/thermocraft_solver_wasm_bg.wasm
      - name: Upload to S3/CDN
        run: aws s3 cp solver-wasm/pkg/ s3://thermocraft-wasm/v${{ github.sha }}/ --recursive
```

---

## Architecture Deep-Dives

### 1. Simulation API Handler (Rust/Axum)

The primary endpoint receives a thermal simulation request, validates the geometry, decides between WASM and server execution, and for server execution enqueues a job with Redis and streams progress via WebSocket.

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
    mesh::geometry::count_components,
    state::AppState,
    auth::Claims,
    error::ApiError,
};

#[derive(serde::Deserialize)]
pub struct CreateSimulationRequest {
    pub analysis_type: AnalysisType,
    pub parameters: SimulationParams,
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

    // 2. Count components in geometry
    let component_count = count_components(&project.geometry_data)?;

    // 3. Check plan limits
    let user = sqlx::query_as!(
        crate::db::models::User,
        "SELECT * FROM users WHERE id = $1",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    if user.plan == "free" && component_count > 10 {
        return Err(ApiError::PlanLimit(
            "Free plan supports up to 10 components. Upgrade to Pro for unlimited."
        ));
    }

    // 4. Determine execution mode
    let execution_mode = if component_count <= 100 {
        "wasm"
    } else {
        "server"
    };

    // 5. Create simulation record
    let sim = sqlx::query_as!(
        Simulation,
        r#"INSERT INTO simulations
            (project_id, user_id, analysis_type, status, execution_mode,
             parameters, component_count)
        VALUES ($1, $2, $3, $4, $5, $6, $7)
        RETURNING *"#,
        project_id,
        claims.user_id,
        serde_json::to_string(&req.analysis_type)?,
        if execution_mode == "wasm" { "pending" } else { "pending" },
        execution_mode,
        serde_json::to_value(&req.parameters)?,
        component_count as i32,
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
        let mut redis = state.redis.get_async_connection().await?;
        redis::cmd("LPUSH")
            .arg("simulation:jobs")
            .arg(serde_json::to_string(&job.id)?)
            .query_async::<_, ()>(&mut redis)
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

The core FVM conduction solver that builds and solves the thermal system. This code compiles to both native (server) and WASM (browser) targets.

```rust
// solver-core/src/thermal.rs

use sprs::{CsMat, TriMat};
use nalgebra::DVector;
use crate::mesh::{Mesh, Cell, Face};
use crate::materials::Material;

pub struct ThermalSystem {
    pub mesh: Mesh,
    pub materials: Vec<Material>,
    pub temperatures: Vec<f64>,  // Temperature at each cell center (K)
    pub heat_sources: Vec<f64>,  // Volumetric heat generation (W/m³)
    pub ambient_temp: f64,       // K
    pub convergence_history: Vec<f64>,
}

impl ThermalSystem {
    pub fn new(mesh: Mesh, materials: Vec<Material>, ambient_temp: f64) -> Self {
        let n_cells = mesh.cells.len();
        Self {
            mesh,
            materials,
            temperatures: vec![ambient_temp; n_cells],
            heat_sources: vec![0.0; n_cells],
            ambient_temp,
            convergence_history: Vec::new(),
        }
    }

    /// Build conductance matrix and RHS for steady-state conduction
    pub fn build_system(&self) -> (CsMat<f64>, DVector<f64>) {
        let n = self.mesh.cells.len();
        let mut matrix = TriMat::new((n, n));
        let mut rhs = DVector::zeros(n);

        for (cell_id, cell) in self.mesh.cells.iter().enumerate() {
            let material = &self.materials[cell.material_id];
            let k = material.thermal_conductivity;
            let volume = cell.volume;

            let mut diagonal = 0.0;

            // Diffusion terms: sum over faces
            for &face_id in &cell.faces {
                let face = &self.mesh.faces[face_id];
                let neighbor_id = if face.owner == cell_id {
                    face.neighbor
                } else {
                    Some(face.owner)
                };

                if let Some(nbr) = neighbor_id {
                    // Interior face: cell-to-cell conduction
                    let nbr_material = &self.materials[self.mesh.cells[nbr].material_id];
                    let k_nbr = nbr_material.thermal_conductivity;

                    // Harmonic mean of thermal conductivity at interface
                    let k_face = 2.0 * k * k_nbr / (k + k_nbr);

                    let distance = self.mesh.distance_between(cell_id, nbr);
                    let conductance = k_face * face.area / distance;

                    matrix.add_triplet(cell_id, nbr, -conductance);
                    diagonal += conductance;
                } else {
                    // Boundary face: apply boundary condition
                    match &face.boundary_condition {
                        BoundaryCondition::FixedTemperature(t_bc) => {
                            // Dirichlet: treat as very large conductance to fixed temp
                            let large_conductance = 1e10 * k * face.area;
                            diagonal += large_conductance;
                            rhs[cell_id] += large_conductance * t_bc;
                        }
                        BoundaryCondition::HeatFlux(q) => {
                            // Neumann: q" = heat flux (W/m²)
                            rhs[cell_id] += q * face.area;
                        }
                        BoundaryCondition::Convection { h, t_inf } => {
                            // Robin: q" = h(T - T_inf)
                            let convection_conductance = h * face.area;
                            diagonal += convection_conductance;
                            rhs[cell_id] += convection_conductance * t_inf;
                        }
                        BoundaryCondition::Radiation { emissivity, t_surr } => {
                            // Radiation (linearized for steady-state)
                            const SIGMA: f64 = 5.67e-8;  // Stefan-Boltzmann constant
                            let t_cell = self.temperatures[cell_id];
                            let t_surr_k = t_surr + 273.15;
                            let t_cell_k = t_cell;

                            // Linearized radiation coefficient
                            let h_rad = emissivity * SIGMA *
                                (t_cell_k.powi(4) - t_surr_k.powi(4)) /
                                (t_cell_k - t_surr_k).max(1e-6);

                            let rad_conductance = h_rad * face.area;
                            diagonal += rad_conductance;
                            rhs[cell_id] += rad_conductance * t_surr_k;
                        }
                    }
                }
            }

            // Source term
            rhs[cell_id] += self.heat_sources[cell_id] * volume;

            // Diagonal entry
            matrix.add_triplet(cell_id, cell_id, diagonal);
        }

        (matrix.to_csr(), rhs)
    }

    /// Solve steady-state system using iterative method (Gauss-Seidel with under-relaxation)
    pub fn solve_steady_state(&mut self, options: &SolverOptions) -> Result<(), SolverError> {
        for iter in 0..options.max_iterations {
            let (matrix, rhs) = self.build_system();

            // Solve using sparse direct solver (for WASM: iterative, for server: direct)
            let solution = solve_sparse_system(&matrix, &rhs)?;

            // Under-relaxation
            let alpha = options.under_relaxation;
            let mut max_change = 0.0;
            for i in 0..self.temperatures.len() {
                let new_temp = alpha * solution[i] + (1.0 - alpha) * self.temperatures[i];
                let change = (new_temp - self.temperatures[i]).abs();
                max_change = max_change.max(change);
                self.temperatures[i] = new_temp;
            }

            self.convergence_history.push(max_change);

            // Check convergence
            if max_change < options.tolerance {
                tracing::info!("Converged in {} iterations, residual: {:.2e} K",
                    iter + 1, max_change);
                return Ok(());
            }

            // Re-linearize radiation if present (update boundary conditions)
            self.update_radiation_boundary_conditions();
        }

        Err(SolverError::Convergence(
            format!("Failed to converge after {} iterations", options.max_iterations)
        ))
    }

    /// Solve transient system using implicit Euler
    pub fn solve_transient(
        &mut self,
        params: &TransientParams,
        options: &SolverOptions
    ) -> Result<Vec<TemperatureSnapshot>, SolverError> {
        let mut results = Vec::new();
        let mut time = 0.0;
        let dt = params.time_step;

        // Initial condition
        results.push(TemperatureSnapshot {
            time,
            temperatures: self.temperatures.clone(),
        });

        while time < params.duration {
            let (matrix, mut rhs) = self.build_system();

            // Add transient term: (ρc_p V / Δt) to diagonal, (ρc_p V / Δt) T^n to RHS
            let mut matrix_transient = matrix.to_dense();
            for (cell_id, cell) in self.mesh.cells.iter().enumerate() {
                let material = &self.materials[cell.material_id];
                let rho_cp_v = material.density * material.specific_heat * cell.volume;
                let transient_coeff = rho_cp_v / dt;

                matrix_transient[(cell_id, cell_id)] += transient_coeff;
                rhs[cell_id] += transient_coeff * self.temperatures[cell_id];
            }

            // Solve for T^{n+1}
            let solution = solve_dense_system(&matrix_transient, &rhs)?;
            self.temperatures = solution.as_slice().to_vec();

            time += dt;
            results.push(TemperatureSnapshot {
                time,
                temperatures: self.temperatures.clone(),
            });
        }

        Ok(results)
    }

    fn update_radiation_boundary_conditions(&mut self) {
        // Update radiation boundary condition linearization based on current temperatures
        // Called iteratively during steady-state solve
    }
}

pub struct TemperatureSnapshot {
    pub time: f64,
    pub temperatures: Vec<f64>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct SolverOptions {
    pub tolerance: f64,
    pub max_iterations: u32,
    pub under_relaxation: f64,
}

pub enum BoundaryCondition {
    FixedTemperature(f64),  // K
    HeatFlux(f64),  // W/m²
    Convection { h: f64, t_inf: f64 },  // h in W/m²·K, t_inf in K
    Radiation { emissivity: f64, t_surr: f64 },  // t_surr in K
}

#[derive(Debug)]
pub enum SolverError {
    Convergence(String),
    InvalidMesh(String),
    InvalidBoundaryCondition(String),
}

fn solve_sparse_system(matrix: &CsMat<f64>, rhs: &DVector<f64>) -> Result<DVector<f64>, SolverError> {
    #[cfg(target_arch = "wasm32")]
    {
        // Use iterative solver (Gauss-Seidel) for WASM
        unimplemented!("Iterative solver for WASM")
    }
    #[cfg(not(target_arch = "wasm32"))]
    {
        // Use direct solver via nalgebra
        let dense = matrix.to_dense();
        let lu = dense.lu();
        Ok(lu.solve(rhs).ok_or(SolverError::Convergence("LU solve failed".to_string()))?)
    }
}

fn solve_dense_system(matrix: &nalgebra::DMatrix<f64>, rhs: &DVector<f64>) -> Result<DVector<f64>, SolverError> {
    let lu = matrix.clone().lu();
    lu.solve(rhs).ok_or(SolverError::Convergence("Dense solve failed".to_string()))
}
```

### 3. Three.js Temperature Visualization (React + WebGL)

```typescript
// frontend/src/components/ThermalViewer.tsx

import { useRef, useEffect, useState } from 'react';
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';
import { useThermalStore } from '../stores/thermalStore';

export function ThermalViewer() {
  const mountRef = useRef<HTMLDivElement>(null);
  const sceneRef = useRef<THREE.Scene | null>(null);
  const rendererRef = useRef<THREE.WebGLRenderer | null>(null);
  const meshRef = useRef<THREE.Mesh | null>(null);

  const { temperatureData, colorScale, crossSectionPlane } = useThermalStore();
  const [maxTemp, setMaxTemp] = useState(0);
  const [minTemp, setMinTemp] = useState(0);

  // Initialize Three.js scene
  useEffect(() => {
    if (!mountRef.current) return;

    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x1a1a1a);
    sceneRef.current = scene;

    const camera = new THREE.PerspectiveCamera(
      45, mountRef.current.clientWidth / mountRef.current.clientHeight, 0.1, 1000
    );
    camera.position.set(0.5, 0.5, 1.0);

    const renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(mountRef.current.clientWidth, mountRef.current.clientHeight);
    mountRef.current.appendChild(renderer.domElement);
    rendererRef.current = renderer;

    const controls = new OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true;

    const ambientLight = new THREE.AmbientLight(0xffffff, 0.6);
    scene.add(ambientLight);

    const animate = () => {
      requestAnimationFrame(animate);
      controls.update();
      renderer.render(scene, camera);
    };
    animate();

    return () => {
      renderer.dispose();
      mountRef.current?.removeChild(renderer.domElement);
    };
  }, []);

  // Update temperature mesh
  useEffect(() => {
    if (!temperatureData || !sceneRef.current) return;

    const scene = sceneRef.current;
    if (meshRef.current) {
      scene.remove(meshRef.current);
      meshRef.current.geometry.dispose();
      (meshRef.current.material as THREE.Material).dispose();
    }

    const geometry = new THREE.BufferGeometry();
    geometry.setAttribute('position', new THREE.BufferAttribute(temperatureData.cellPositions, 3));
    geometry.setIndex(new THREE.BufferAttribute(temperatureData.cellIndices, 1));

    const temps = temperatureData.temperatures;
    const min = Math.min(...temps);
    const max = Math.max(...temps);
    setMinTemp(min);
    setMaxTemp(max);

    const colors = new Float32Array(temps.length * 3);
    for (let i = 0; i < temps.length; i++) {
      const normalized = (temps[i] - min) / (max - min);
      const rgb = getColormapColor(normalized, colorScale.colormap);
      colors[i * 3] = rgb.r;
      colors[i * 3 + 1] = rgb.g;
      colors[i * 3 + 2] = rgb.b;
    }
    geometry.setAttribute('color', new THREE.BufferAttribute(colors, 3));

    const material = new THREE.MeshPhongMaterial({ vertexColors: true, side: THREE.DoubleSide });

    if (crossSectionPlane) {
      const clipPlane = new THREE.Plane(
        new THREE.Vector3(...crossSectionPlane.normal),
        -crossSectionPlane.distance
      );
      material.clippingPlanes = [clipPlane];
      rendererRef.current!.localClippingEnabled = true;
    }

    const mesh = new THREE.Mesh(geometry, material);
    meshRef.current = mesh;
    scene.add(mesh);
  }, [temperatureData, colorScale, crossSectionPlane]);

  return (
    <div className="thermal-viewer relative w-full h-full">
      <div ref={mountRef} className="w-full h-full" />
      <div className="absolute top-4 right-4 bg-gray-800 text-white p-3 rounded-lg">
        <div className="text-sm font-semibold mb-2">Temperature Scale</div>
        <div className="text-xs">Max: {maxTemp.toFixed(1)}°C</div>
        <div className="text-xs">Min: {minTemp.toFixed(1)}°C</div>
      </div>
    </div>
  );
}

function getColormapColor(value: number, colormap: string): { r: number; g: number; b: number } {
  value = Math.max(0, Math.min(1, value));

  if (colormap === 'jet') {
    if (value < 0.125) return { r: 0, g: 0, b: 0.5 + value * 4 };
    else if (value < 0.375) return { r: 0, g: (value - 0.125) * 4, b: 1 };
    else if (value < 0.625) return { r: (value - 0.375) * 4, g: 1, b: 1 - (value - 0.375) * 4 };
    else if (value < 0.875) return { r: 1, g: 1 - (value - 0.625) * 4, b: 0 };
    else return { r: 1 - (value - 0.875) * 4, g: 0, b: 0 };
  }

  return { r: value, g: value, b: value };
}
```

### 4. Natural Convection Correlations (Rust)

```rust
// solver-core/src/convection.rs

const G: f64 = 9.81;  // Gravitational acceleration (m/s²)

pub struct ConvectionParams {
    pub fluid_properties: FluidProperties,
    pub orientation: SurfaceOrientation,
    pub characteristic_length: f64,  // m
}

pub struct FluidProperties {
    pub kinematic_viscosity: f64,  // m²/s (ν)
    pub thermal_diffusivity: f64,   // m²/s (α)
    pub thermal_conductivity: f64,  // W/m·K
    pub prandtl_number: f64,        // Pr = ν/α
    pub volumetric_expansion: f64,  // 1/K (β ≈ 1/T for ideal gas)
}

pub enum SurfaceOrientation {
    VerticalPlate,
    HorizontalPlateUp,     // Hot surface facing up
    HorizontalPlateDown,   // Hot surface facing down
    Cylinder,
    Sphere,
}

impl FluidProperties {
    pub fn air_at_temp(temp_celsius: f64) -> Self {
        let temp_kelvin = temp_celsius + 273.15;
        Self {
            kinematic_viscosity: 1.5e-5 * (temp_kelvin / 273.15).powf(1.5),
            thermal_diffusivity: 2.2e-5 * (temp_kelvin / 273.15).powf(1.5),
            thermal_conductivity: 0.024 + 7.0e-5 * temp_celsius,
            prandtl_number: 0.71,
            volumetric_expansion: 1.0 / temp_kelvin,
        }
    }
}

/// Compute natural convection heat transfer coefficient (W/m²·K)
pub fn compute_natural_convection_h(
    params: &ConvectionParams,
    surface_temp: f64,  // K
    ambient_temp: f64,  // K
) -> f64 {
    let delta_t = (surface_temp - ambient_temp).abs();
    if delta_t < 1e-3 {
        return 5.0;  // Minimum h to avoid singularity
    }

    let l = params.characteristic_length;
    let props = &params.fluid_properties;

    // Rayleigh number: Ra = gβΔTL³/(να)
    let ra = G * props.volumetric_expansion * delta_t * l.powi(3) /
             (props.kinematic_viscosity * props.thermal_diffusivity);

    let nu = match params.orientation {
        SurfaceOrientation::VerticalPlate => {
            // Churchill-Chu correlation for vertical flat plate
            let numerator = 0.67 * ra.powf(0.25);
            let denominator = (1.0 + (0.492 / props.prandtl_number).powf(9.0 / 16.0)).powf(4.0 / 9.0);
            0.68 + numerator / denominator
        }
        SurfaceOrientation::HorizontalPlateUp => {
            // Hot surface facing upward
            if ra < 1e7 {
                0.54 * ra.powf(0.25)
            } else {
                0.15 * ra.powf(1.0 / 3.0)
            }
        }
        SurfaceOrientation::HorizontalPlateDown => {
            // Hot surface facing downward (weaker convection)
            0.27 * ra.powf(0.25)
        }
        SurfaceOrientation::Cylinder => {
            // Churchill-Chu for horizontal cylinder
            let numerator = 0.6 + 0.387 * ra.powf(1.0 / 6.0);
            let denominator = (1.0 + (0.559 / props.prandtl_number).powf(9.0 / 16.0)).powf(8.0 / 27.0);
            numerator / denominator
        }
        SurfaceOrientation::Sphere => {
            // Natural convection from sphere
            2.0 + 0.589 * ra.powf(0.25) /
            (1.0 + (0.469 / props.prandtl_number).powf(9.0 / 16.0)).powf(4.0 / 9.0)
        }
    };

    // h = Nu · k / L
    nu * props.thermal_conductivity / l
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–4)

**Day 1: Project initialization and Rust backend scaffold**
- `cargo init thermocraft-api && cd thermocraft-api`
- Add dependencies: `axum, tokio, serde, sqlx, uuid, chrono, tower-http, jsonwebtoken, bcrypt, aws-sdk-s3, redis`
- `src/main.rs` — Axum app with CORS, tracing, graceful shutdown
- `src/config.rs` — Environment config (DATABASE_URL, JWT_SECRET, S3_BUCKET, REDIS_URL)
- `src/state.rs` — AppState (PgPool, Redis, S3Client)
- `src/error.rs` — ApiError enum with IntoResponse
- `Dockerfile` — Multi-stage build
- `docker-compose.yml` — PostgreSQL, Redis, MinIO

**Day 2: Database schema and migrations**
- `migrations/001_initial.sql` — All 9 tables: users, organizations, org_members, projects, simulations, simulation_jobs, components, materials, visualization_configs, usage_records
- `src/db/models.rs` — All SQLx structs with FromRow derives
- `sqlx migrate run` to apply schema
- Seed script for initial materials (copper, silicon, FR4, aluminum, thermal paste)

**Day 3: Authentication system**
- `src/auth/mod.rs` — JWT token generation and validation middleware
- `src/auth/oauth.rs` — Google and GitHub OAuth 2.0 flows
- `src/api/handlers/auth.rs` — Register, login, OAuth callback, refresh token, me endpoints
- Password hashing with bcrypt (cost 12)
- JWT with 24h access + 30d refresh token

**Day 4: User and project CRUD**
- `src/api/handlers/users.rs` — Profile CRUD
- `src/api/handlers/projects.rs` — Create, list, get, update, delete, fork project
- `src/api/handlers/orgs.rs` — Org management
- `src/api/router.rs` — Route definitions with auth middleware
- Integration tests for auth and project CRUD

### Phase 2 — Thermal Solver Core (Days 5–11)

**Day 5: Mesh data structures**
- `solver-core/` — New Rust workspace member
- `solver-core/src/mesh.rs` — Mesh, Cell, Face structs
- `solver-core/src/octree.rs` — Octree mesh generator for arbitrary geometry
- Unit tests: cube mesh, sphere mesh, mesh refinement

**Day 6: FVM conduction solver**
- `solver-core/src/thermal.rs` — ThermalSystem struct with build_system(), solve_steady_state()
- Sparse matrix assembly using `sprs` crate
- Iterative solver with under-relaxation
- Tests: 1D conduction rod (compare to analytical T(x) = T0 + (T1-T0)x/L)

**Day 7: Boundary conditions**
- Dirichlet (fixed temperature), Neumann (heat flux), Robin (convection), radiation
- `solver-core/src/boundary.rs` — BoundaryCondition enum
- Apply BCs during matrix assembly
- Tests: convection BC (compare to h·A·ΔT), radiation BC (linearized)

**Day 8: Anisotropic materials**
- Support thermal conductivity tensor [kx, ky, kz]
- PCB material model: high in-plane (copper layers), low through-plane
- `solver-core/src/materials.rs` — Material library with presets
- Tests: anisotropic cube with different k in each direction

**Day 9: Transient solver**
- Implicit Euler time integration
- `solve_transient()` method in ThermalSystem
- Time-stepping with fixed or adaptive Δt
- Tests: transient heating of aluminum block (compare to analytical 1D solution)

**Day 10: Natural convection correlations**
- `solver-core/src/convection.rs` — compute_natural_convection_h()
- Churchill-Chu for vertical/horizontal plates, cylinders, spheres
- Air properties at different temperatures
- Tests: verify Nu number for Ra = 10^6, 10^8

**Day 11: Component thermal models**
- `solver-core/src/components.rs` — IC package models (BGA, QFN, TO-220) with Rjc, Rca
- Heat sink models with fin geometry and effective thermal resistance
- TIM models with thermal conductivity and bond-line thickness
- Tests: verify junction temperature Tj = Tamb + Rja * P

### Phase 3 — WASM Build + Frontend Foundation (Days 12–17)

**Day 12: WASM solver compilation**
- `solver-wasm/` — New workspace member
- `solver-wasm/Cargo.toml` — wasm-bindgen, serde-wasm-bindgen
- `solver-wasm/src/lib.rs` — WASM entry points: solve_steady_state_wasm(), solve_transient_wasm()
- `wasm-pack build --target web --release` pipeline
- GitHub Actions workflow for WASM build

**Day 13: Frontend scaffold**
- `npm create vite@latest frontend -- --template react-ts`
- Add dependencies: `zustand, @tanstack/react-query, axios, three, @types/three`
- `src/App.tsx` — Router, auth context, layout
- `src/stores/authStore.ts` — JWT storage, login/logout
- `src/lib/api.ts` — Axios API client

**Day 14: 3D geometry builder foundation**
- `src/components/GeometryBuilder/Canvas.tsx` — Three.js scene setup
- `src/components/GeometryBuilder/ComponentPalette.tsx` — Component library sidebar
- `src/stores/geometryStore.ts` — Geometry state (components, placements)
- Drag-and-drop component placement from palette to 3D canvas

**Day 15: Component placement and manipulation**
- `src/components/GeometryBuilder/ComponentInstance.tsx` — Render component mesh in scene
- Translate/rotate/scale controls for placed components
- Property panel for component settings (power dissipation, material)
- Snap-to-grid for alignment

**Day 16: STEP geometry import**
- `src/lib/stepParser.ts` — opencascade.js integration for STEP import
- Upload STEP file → parse → convert to Three.js mesh
- Mesh simplification for display
- Tests: import cube, cylinder, complex CAD assembly

**Day 17: Three.js thermal visualization**
- `src/components/ThermalViewer.tsx` — Temperature contour rendering
- Vertex color mapping with jet/viridis/thermal colormaps
- Color scale legend
- Cross-section plane controls (position, normal)

### Phase 4 — API + Simulation Orchestration (Days 18–23)

**Day 18: Simulation API endpoints**
- `src/api/handlers/simulation.rs` — Create, get, list, cancel simulation
- Component count extraction and WASM/server routing
- Plan-based limits enforcement (free: 10 components, pro: unlimited)

**Day 19: Server-side simulation worker**
- `src/workers/simulation_worker.rs` — Redis job consumer
- Mesh generation on server
- Native solver execution
- S3 result upload with presigned URLs

**Day 20: WebSocket for live progress**
- `src/api/ws/simulation_progress.rs` — WebSocket endpoint
- Progress streaming: iteration count, residual, temperature range
- `frontend/src/hooks/useSimulationProgress.ts` — React WebSocket hook

**Day 21: Component library API**
- `src/api/handlers/components.rs` — Search, get, download component
- S3 integration for geometry meshes
- Parametric search (category, manufacturer, thermal resistance, power rating)
- Pagination and full-text search

**Day 22: Material library API**
- `src/api/handlers/materials.rs` — List materials, get material properties
- Built-in material database seeding
- Custom material upload (Team plan only)

**Day 23: Geometry export/import**
- `src/api/handlers/export.rs` — Export geometry as STEP, STL, or JSON
- Import geometry from STEP file → create project
- Thumbnail generation for project cards

### Phase 5 — Results Visualization + Post-Processing (Days 24–28)

**Day 24: Temperature field visualization**
- Parse result data from S3
- Upload to GPU as vertex colors
- Real-time color scale adjustment
- Temperature probe annotations (click point → show temp)

**Day 25: Cross-sectional views**
- `src/components/ThermalViewer/CrossSection.tsx` — Clipping plane controls
- XY, YZ, XZ plane presets
- Arbitrary plane orientation (normal + distance)

**Day 26: Thermal path tracing**
- Compute thermal resistance path from heat source to ambient
- Visualize dominant heat flow paths as streamlines
- Bottleneck identification (highest thermal resistance links)

**Day 27: Junction temperature prediction**
- For each IC component, compute Tj using Rjc and package model
- Display Tj overlay on components in 3D view
- Color-code components by thermal margin (green = safe, yellow = warn, red = exceed limit)

**Day 28: Report generation**
- `src/api/handlers/reports.rs` — Generate PDF thermal report
- Include: max temps, hot spot locations, component margins, thermal resistance paths
- Export as PDF via headless rendering

### Phase 6 — Billing + Plan Enforcement (Days 29–32)

**Day 29: Stripe integration**
- `src/api/handlers/billing.rs` — Checkout session, customer portal
- `src/api/handlers/webhooks/stripe.rs` — Webhook handler
- Plan mapping: Free (10 components), Pro ($129/mo, unlimited), Advanced ($299/mo, advanced features)

**Day 30: Usage tracking**
- `src/middleware/plan_limits.rs` — Enforce limits before simulation
- `src/services/usage.rs` — Track simulation minutes
- Usage dashboard API

**Day 31: Billing UI**
- `frontend/src/pages/Billing.tsx` — Plan comparison, usage meter
- Upgrade/downgrade via Stripe portal
- Upgrade prompt modals on limit hit

**Day 32: Feature gating**
- Gate transient analysis (Advanced plan)
- Gate battery/data center workflows (Advanced plan)
- Gate API access (Advanced plan)
- Show locked feature indicators with CTAs

### Phase 7 — Solver Validation + Testing (Days 33–37)

**Day 33: Analytical validation benchmarks**
- Benchmark 1: 1D conduction rod — analytical T(x) = T0 + (T1-T0)x/L → within 0.1%
- Benchmark 2: Infinite fin — analytical T(x) = Tb·cosh(m(L-x))/cosh(mL) → within 1%
- Benchmark 3: Sphere cooling — analytical transient solution → within 2%

**Day 34: Electronics thermal benchmarks**
- Benchmark 4: BGA package on PCB with copper spreader → compare to JEDEC JESD51 standard
- Benchmark 5: Heat sink with natural convection → verify θsa within 10% of manufacturer data
- Benchmark 6: TO-220 device on heatsink → verify junction temp for 5W dissipation

**Day 35: Convection validation**
- Compare natural convection h values to experimental correlations in literature
- Vertical plate: Nu vs Ra for 10^4 < Ra < 10^10 → within 15% of Churchill-Chu
- Horizontal plate: Nu vs Ra → within 20% of correlation

**Day 36: Convergence and performance tests**
- Test convergence for difficult geometries (thin features, large aspect ratio)
- Test with radiation boundary conditions (nonlinear)
- Performance benchmarks: time vs. cell count for WASM and server
- Target: 50K cells in <10s on WASM, 500K cells in <60s on server

**Day 37: Integration testing**
- End-to-end: create project → place components → run simulation → view results → export report
- WASM solver test in headless browser
- WebSocket live progress test
- Concurrent simulation stress test (10 simultaneous jobs)

### Phase 8 — Deployment + Polish + Launch (Days 38–42)

**Day 38: Docker and Kubernetes**
- `Dockerfile` — Multi-stage Rust build with cargo-chef
- `docker-compose.yml` — Local dev stack
- `k8s/` — Kubernetes manifests for API, workers, PostgreSQL, Redis
- Health check endpoints

**Day 39: CDN and WASM delivery**
- CloudFront distribution for frontend and WASM bundle
- WASM preloading and service worker caching
- Component library assets cached at edge

**Day 40: Monitoring and observability**
- Prometheus metrics: simulation duration, solver convergence rate, API latency
- Grafana dashboards: system health, simulation throughput, user activity
- Sentry integration for error tracking
- Structured logging with tracing

**Day 41: UI polish and accessibility**
- Loading states, error messages, empty states
- Responsive layout for mobile/tablet
- Keyboard navigation
- ARIA labels
- Dark mode theme

**Day 42: Launch preparation**
- End-to-end smoke test in production
- Database backup verification
- Rate limiting (100 req/min per user, 10 simulations/min)
- Security audit (JWT, CORS, CSP)
- Landing page with pricing and signup
- Documentation (getting started, tutorials, API reference)
- Deploy and announce launch

---

## Critical Files

```
thermocraft/
├── solver-core/                           # Shared solver library (native + WASM)
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── thermal.rs                     # ThermalSystem, FVM solver
│   │   ├── mesh.rs                        # Mesh, Cell, Face data structures
│   │   ├── octree.rs                      # Octree mesh generator
│   │   ├── materials.rs                   # Material library and properties
│   │   ├── components.rs                  # IC package and heat sink models
│   │   ├── convection.rs                  # Natural convection correlations
│   │   ├── boundary.rs                    # Boundary condition handlers
│   │   └── radiation.rs                   # View factor computation
│   └── tests/
│       ├── benchmarks.rs                  # Analytical validation tests
│       └── convergence.rs                 # Convergence stress tests
│
├── solver-wasm/                           # WASM compilation target
│   ├── Cargo.toml
│   └── src/
│       └── lib.rs                         # WASM entry points (wasm-bindgen)
│
├── thermocraft-api/                       # Rust API server
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
│   │   │   │   ├── components.rs          # Component library search
│   │   │   │   ├── materials.rs           # Material library
│   │   │   │   ├── billing.rs             # Stripe checkout/portal
│   │   │   │   ├── export.rs              # Geometry export, report generation
│   │   │   │   └── webhooks/
│   │   │   │       └── stripe.rs          # Stripe webhook handler
│   │   │   └── ws/
│   │   │       └── simulation_progress.rs # Live progress streaming
│   │   ├── middleware/
│   │   │   └── plan_limits.rs             # Plan enforcement
│   │   ├── services/
│   │   │   ├── usage.rs                   # Usage tracking
│   │   │   └── s3.rs                      # S3 helpers
│   │   └── workers/
│   │       └── simulation_worker.rs       # Server-side solver execution
│   ├── migrations/
│   │   └── 001_initial.sql                # Full database schema
│   └── tests/
│       └── api_integration.rs             # API integration tests
│
├── frontend/                              # React frontend
│   ├── package.json
│   ├── vite.config.ts
│   ├── src/
│   │   ├── App.tsx                        # Router, providers, layout
│   │   ├── main.tsx                       # Entry point
│   │   ├── stores/
│   │   │   ├── authStore.ts               # Auth state
│   │   │   ├── projectStore.ts            # Project state
│   │   │   ├── geometryStore.ts           # Geometry builder state
│   │   │   └── thermalStore.ts            # Thermal visualization state
│   │   ├── hooks/
│   │   │   ├── useSimulationProgress.ts   # WebSocket hook
│   │   │   └── useWasmSolver.ts           # WASM solver hook
│   │   ├── lib/
│   │   │   ├── api.ts                     # Axios API client
│   │   │   ├── wasmLoader.ts              # WASM bundle loading
│   │   │   └── stepParser.ts              # STEP import (opencascade.js)
│   │   ├── pages/
│   │   │   ├── Dashboard.tsx              # Project list
│   │   │   ├── Editor.tsx                 # Main editor (geometry + results)
│   │   │   ├── Components.tsx             # Component library browser
│   │   │   ├── Billing.tsx                # Plan management
│   │   │   ├── Login.tsx                  # Auth pages
│   │   │   └── Register.tsx
│   │   ├── components/
│   │   │   ├── GeometryBuilder/
│   │   │   │   ├── Canvas.tsx             # Three.js scene
│   │   │   │   ├── ComponentPalette.tsx   # Component library sidebar
│   │   │   │   ├── ComponentInstance.tsx  # Placed component rendering
│   │   │   │   ├── TransformControls.tsx  # Translate/rotate/scale
│   │   │   │   └── PropertyPanel.tsx      # Component properties editor
│   │   │   ├── ThermalViewer/
│   │   │   │   ├── ThermalViewer.tsx      # Temperature visualization
│   │   │   │   ├── CrossSection.tsx       # Cross-section controls
│   │   │   │   ├── ColorScale.tsx         # Color scale legend
│   │   │   │   └── TemperatureProbe.tsx   # Temperature probe tool
│   │   │   ├── billing/
│   │   │   │   ├── PlanCard.tsx
│   │   │   │   └── UsageMeter.tsx
│   │   │   └── common/
│   │   │       └── SimulationToolbar.tsx   # Run/stop controls
│   │   └── public/
│   │       └── wasm/                      # WASM solver bundle
│
├── battery-service/                       # Python FastAPI (post-MVP)
│   ├── requirements.txt                   # PyBaMM, scipy, numpy
│   ├── main.py                            # FastAPI app
│   ├── models/
│   │   └── battery_thermal.py            # Electrochemical-thermal coupling
│   └── Dockerfile
│
└── k8s/                                   # Kubernetes manifests
    ├── api-deployment.yaml                # API server (3 replicas, HPA)
    ├── worker-deployment.yaml             # Simulation workers (auto-scaling)
    ├── postgres-statefulset.yaml          # PostgreSQL with PVC
    ├── redis-deployment.yaml              # Redis for jobs and pub/sub
    └── ingress.yaml                       # NGINX ingress with TLS
```

---

## Solver Validation Suite

### Benchmark 1: 1D Conduction — Analytical Comparison

**Setup:** Aluminum rod (1m length, 0.01m² cross-section), left end at 373K, right end at 293K, steady-state conduction.

**Analytical solution:** T(x) = T_left + (T_right - T_left) × (x / L)

**Expected result:** Linear temperature distribution from 373K to 293K.

**Validation:** Solver result must match analytical within 0.1% at all points.

### Benchmark 2: Infinite Fin — Analytical Heat Transfer

**Setup:** Aluminum fin (L=0.1m, cross-section 0.001m², perimeter 0.04m), base at 373K, tip insulated, ambient 293K, h=25 W/m²·K.

**Analytical solution:** T(x) = T_amb + (T_base - T_amb) × cosh(m(L-x)) / cosh(mL), where m = sqrt(hP/kA).

**Expected result:** For given parameters, fin tip temperature ≈ 315K.

**Validation:** Solver result within 1% of analytical at 10 points along fin.

### Benchmark 3: Transient Sphere Cooling — Lumped Capacitance

**Setup:** Copper sphere (radius 0.05m), initial temperature 373K, ambient 293K, h=10 W/m²·K, transient cooling.

**Analytical solution:** T(t) = T_amb + (T_init - T_amb) × exp(-t/τ), where τ = ρcV/(hA).

**Expected result:** After 300 seconds, temperature ≈ 303K.

**Validation:** Transient solver result within 2% of analytical at t=60s, 180s, 300s.

### Benchmark 4: BGA Package Junction Temperature — JEDEC Standard

**Setup:** 15mm×15mm BGA package, 5W power dissipation, mounted on 4-layer FR4 PCB (100mm×100mm) with thermal vias, ambient 298K, natural convection.

**Expected result:** Junction-to-ambient thermal resistance θja ≈ 25-30 K/W (JEDEC JESD51 typical value), junction temperature ≈ 423-448K.

**Validation:** Solver computed Tj within 10% of JEDEC standard for similar package.

### Benchmark 5: Heat Sink Natural Convection — Manufacturer Data

**Setup:** Extruded aluminum heat sink (12 fins, 50mm height, 100mm×100mm base), 20W heat source, vertical orientation, ambient 298K.

**Expected result:** Thermal resistance θsa ≈ 2.5 K/W (from manufacturer datasheet for natural convection).

**Validation:** Solver computed heat sink base temperature rise within 15% of manufacturer data (ΔT = 20W × 2.5 K/W = 50K → base at 348K).

---

## Verification Checklist

### Solver Correctness
- [ ] All 5 analytical benchmarks pass within specified tolerance
- [ ] Steady-state convergence for 100K+ cell meshes
- [ ] Transient stability (no temperature oscillations) for various timesteps
- [ ] Anisotropic material handling (PCB copper layers)
- [ ] Natural convection h values match literature correlations

### WASM/Native Parity
- [ ] WASM solver produces identical results to native solver (within FP precision) for same mesh
- [ ] WASM bundle size <2MB gzipped
- [ ] WASM solver completes 50K cell steady-state in <10s on modern browser

### API Functionality
- [ ] Authentication (JWT, OAuth) works correctly
- [ ] Project CRUD operations with proper ownership checks
- [ ] Simulation creation routes to WASM/server correctly based on component count
- [ ] WebSocket progress streaming delivers updates in real-time
- [ ] S3 result upload/download with presigned URLs

### Frontend
- [ ] Three.js scene renders correctly in Chrome, Firefox, Safari
- [ ] Component placement and manipulation (drag, rotate, scale)
- [ ] Temperature visualization with color mapping and cross-sections
- [ ] Responsive layout on desktop, tablet, mobile

### Billing
- [ ] Stripe checkout creates subscription correctly
- [ ] Webhook updates user plan in database
- [ ] Usage tracking records simulation minutes
- [ ] Plan limits enforced (free: 10 components, pro: unlimited)

### Performance
- [ ] API latency p99 <500ms for simulation create
- [ ] WASM solver 50K cells <10s
- [ ] Server solver 500K cells <60s
- [ ] Frontend 60fps rendering with 100K mesh cells

### Security
- [ ] JWT validation on all protected endpoints
- [ ] SQL injection prevented (SQLx compile-time checks)
- [ ] CORS configured correctly
- [ ] CSP headers set
- [ ] Rate limiting prevents abuse

---

## Deployment Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         CloudFront CDN                          │
│  - Frontend static assets (HTML, JS, CSS)                      │
│  - WASM solver bundle (versioned, long TTL)                    │
│  - Component library geometries (S3 origin)                    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                      Application Load Balancer                   │
│  - TLS termination (ACM certificate)                            │
│  - Health checks on /health/ready                               │
└─────────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┴─────────────────────┐
        ↓                                           ↓
┌──────────────────────┐                 ┌──────────────────────┐
│  API Server Pods     │                 │ Simulation Workers   │
│  (Rust/Axum)         │                 │  (Rust native)       │
│  - 3 replicas        │←────Redis───────│  - Auto-scaling      │
│  - HPA (CPU >70%)    │    Job Queue    │  - 1-10 pods         │
│  - 2 CPU, 4GB RAM    │                 │  - 4 CPU, 8GB RAM    │
└──────────────────────┘                 └──────────────────────┘
        │                                           │
        ↓                                           ↓
┌─────────────────────────────────────────────────────────────────┐
│                    PostgreSQL (RDS)                             │
│  - db.t3.medium (2 vCPU, 4GB)                                   │
│  - Multi-AZ for HA                                              │
│  - Automated backups (7-day retention)                          │
└─────────────────────────────────────────────────────────────────┘
        │
        ↓
┌─────────────────────────────────────────────────────────────────┐
│                        S3 Buckets                               │
│  - thermocraft-results: Temperature field data                  │
│  - thermocraft-geometries: Component meshes, STEP files         │
│  - thermocraft-wasm: WASM solver bundle (CDN origin)            │
└─────────────────────────────────────────────────────────────────┘
```

**Infrastructure requirements:**
- Kubernetes cluster: EKS (3 t3.medium nodes for API/workers)
- PostgreSQL: RDS db.t3.medium with 100GB storage
- Redis: ElastiCache r6g.large
- S3: Standard storage with lifecycle policy (results → Glacier after 30 days)
- CloudFront: Standard distribution with custom SSL
- Estimated monthly cost: ~$400 (5K simulations/month, 50GB storage)

---

## Post-MVP Roadmap

### v1.1 — Full CFD Convection (Weeks 13-16)
- Implement SIMPLE algorithm for pressure-velocity coupling
- k-ε turbulence model for forced convection
- Support for fan models with PQ curves
- 3D velocity field visualization
- **Value:** Accurate forced convection modeling for fan-cooled systems (laptops, servers)

### v1.2 — EV Battery Thermal Workflow (Weeks 17-20)
- 1D electrochemical cell model (P2D or equivalent circuit) via Python/PyBaMM
- Couple cell heat generation with 3D thermal solver
- Liquid cold plate simulation (mini-channel, serpentine)
- Pack-level simulation with cell-to-cell variation
- Drive cycle thermal analysis (WLTP, highway, fast charge)
- **Value:** Battery pack thermal design for EVs, preventing thermal runaway

### v1.3 — Data Center CFD Workflow (Weeks 21-24)
- Rack-level heat load specification (per server, heterogeneous)
- CRAH/CRAC unit models with capacity curves
- Raised floor and overhead air distribution
- Hot/cold aisle containment analysis
- PUE estimation and optimization
- **Value:** Data center cooling optimization, reduce energy costs

### v1.4 — PCB Layout Import (ODB++, IPC-2581) (Weeks 25-28)
- Parse PCB layout data (copper layer distribution, via locations)
- Automatic PCB thermal model generation (anisotropic conductivity)
- Component placement extraction and matching to thermal library
- Power dissipation map from schematic or measurement
- **Value:** Direct import from PCB design tools (Altium, Cadence) → thermal analysis

### v1.5 — Optimization and AI Design (Weeks 29-32)
- Heat sink fin geometry optimization (minimize volume/weight for target thermal resistance)
- Cold plate channel topology optimization
- Fan placement optimization
- TIM selection optimization
- Multi-objective optimization (thermal + weight + cost)
- **Value:** Automated thermal design exploration, reduce design iterations

---

**Total MVP lines: ~1470**