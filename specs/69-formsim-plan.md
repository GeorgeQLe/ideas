# 69. FormSim — Metal Forming and Stamping Simulation Platform

## Implementation Plan

**MVP Scope:** Browser-based die face setup with STEP/IGES import for punch, die, and binder geometry with automatic tool mesh generation, blank definition (shape, thickness, material, rolling direction) with 10 common steel/aluminum materials using Hill48 anisotropic plasticity, single-stage deep-draw simulation via explicit dynamic FEA using Belytschko-Tsay shell elements with penalty-based contact and adaptive remeshing compiled to WebAssembly for client-side execution of models ≤20K elements and server-side Rust-native execution for larger models, formability prediction with thinning contour visualization and forming limit diagram (FLD) comparison overlaid on 3D geometry, implicit static springback analysis showing displacement magnitude and angular deviation from nominal, interactive 3D result viewer rendered via WebGL with pan/zoom/rotate and color-mapped contour display of thinning/thickening/strain/springback, 300+ built-in material models (DC04, DP600, DP780, DP980, TRIP780, AA5182, AA6016, AZ31, Ti-6Al-4V) stored in S3 with PostgreSQL metadata, basic formability PDF report generation with thinning histogram and FLD overlay, Stripe billing with three tiers (Pro $199/mo / Advanced $449/user/mo / Enterprise custom).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| Explicit FEA Solver | Rust + CUDA | Custom explicit dynamic shell solver with GPU-accelerated element assembly |
| WASM Compilation | `wasm-pack` + `wasm-bindgen` | Client-side solver for models ≤20K elements |
| Implicit Springback | Rust (native) | Static implicit solver with Newton-Raphson, server-side only |
| Material Models | Rust | Hill48, Barlat Yld2000-2d, BBC2005, Chaboche hardening |
| Die Compensation | Rust | Surface morphing via radial basis functions (RBF) |
| Optimization Engine | Python 3.12 (FastAPI) | Genetic algorithm for blank shape and BHF optimization |
| ML Microstructure | Python 3.12 (FastAPI) | Grain size prediction via JMAK model for hot forming |
| Database | PostgreSQL 16 | Projects, users, simulations, material metadata |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | Material cards, die geometry (STEP), mesh files, result data |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| 3D Viewer | Three.js + WebGPU | GPU-accelerated mesh rendering, contour shading |
| CAD Import | OpenCascade (WASM) | STEP/IGES parsing and tessellation in browser |
| Real-time | WebSocket (Axum) | Live simulation progress, convergence monitoring |
| Job Queue | Redis 7 + Tokio tasks | Server-side simulation job management, priority queues |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | CAD library delivery, static frontend assets |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, error tracking |
| CI/CD | GitHub Actions | WASM build pipeline, Rust tests, Docker image push |

### Key Architecture Decisions

1. **Hybrid client/server solver with WASM threshold at 20K elements**: Models with ≤20,000 shell elements (covers 90%+ of single-part stamping simulations with 1-2mm mesh) run entirely in the browser via WASM, providing instant feedback with zero server cost. Models exceeding 20K elements are submitted to the Rust-native + CUDA server solver which handles multi-hundred-thousand-element models and multi-stage simulations. The threshold is configurable per plan tier.

2. **Custom explicit FEA solver in Rust rather than wrapping LS-DYNA**: Building a custom explicit solver in Rust gives us full control over GPU acceleration, WASM compilation, and cloud-native parallelization. LS-DYNA's licensing model (per-core licensing, no cloud redistribution) makes cloud deployment prohibitively expensive. Our Rust solver implements Belytschko-Tsay underintegrated shell elements (the same formulation used by LS-DYNA and PAM-STAMP) with hourglass control and adaptive timestep via Courant condition.

3. **Three.js/WebGPU viewer with contour shading**: Three.js provides a mature 3D scene graph and built-in mesh handling. WebGPU (with WebGL2 fallback) enables GPU-accelerated contour interpolation directly in fragment shaders, supporting smooth color-mapped visualization of thinning, strain, and springback for meshes with 100K+ nodes. Canvas2D/SVG alternatives cannot handle the geometry complexity or 3D rotation/perspective.

4. **OpenCascade WASM for STEP import**: OpenCascade Technology (OCCT) is the industry-standard kernel for STEP/IGES parsing. The opencascade.js WASM build (via Emscripten) runs entirely in the browser, allowing die face import without server upload. This enables instant preview of imported geometry and client-side tessellation control (chord tolerance) for mesh quality tuning.

5. **S3 for material storage with PostgreSQL metadata catalog**: Material models (JSON files with stress-strain curves, FLC data, anisotropy coefficients, typically 5-50KB each) are stored in S3, while PostgreSQL holds searchable metadata (material grade, manufacturer, thickness range, yield strength, UTS). This allows the material library to scale to 10K+ materials without bloating the database while enabling fast parametric search via PostgreSQL indexes and full-text search on grade names.

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
    plan TEXT NOT NULL DEFAULT 'pro',  -- pro | advanced | enterprise
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX users_email_idx ON users(email);
CREATE INDEX users_stripe_idx ON users(stripe_customer_id);

-- Organizations (for Advanced and Enterprise plans)
CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    slug TEXT UNIQUE NOT NULL,
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    plan TEXT NOT NULL DEFAULT 'advanced',
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

-- Projects (die design + simulation workspace)
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    process_type TEXT NOT NULL DEFAULT 'stamping',  -- stamping | forging | extrusion | rolling | hydroforming
    die_geometry JSONB NOT NULL DEFAULT '{}',  -- Tool geometries (punch, die, binder), blank definition
    settings JSONB DEFAULT '{}',  -- Mesh size, contact parameters, friction
    is_public BOOLEAN NOT NULL DEFAULT false,
    forked_from UUID REFERENCES projects(id) ON DELETE SET NULL,
    thumbnail_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);
CREATE INDEX projects_org_idx ON projects(org_id);
CREATE INDEX projects_updated_idx ON projects(updated_at DESC);

-- Simulation Runs
CREATE TABLE simulations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    simulation_type TEXT NOT NULL,  -- forming | springback | die_stress | multi_stage
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | running | completed | failed | cancelled
    execution_mode TEXT NOT NULL DEFAULT 'wasm',  -- wasm | server | gpu
    mesh_data_url TEXT,  -- S3 URL for mesh file (if not generated on-the-fly)
    element_count INTEGER NOT NULL DEFAULT 0,
    node_count INTEGER NOT NULL DEFAULT 0,
    parameters JSONB NOT NULL DEFAULT '{}',  -- Forming parameters (BHF, friction, etc.)
    results_url TEXT,  -- S3 URL for result data (nodal displacements, stresses, strains)
    results_summary JSONB,  -- Quick-access summary (max thinning, splits, wrinkles)
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
    cores_allocated INTEGER DEFAULT 8,
    memory_mb INTEGER DEFAULT 16384,
    gpu_allocated BOOLEAN DEFAULT false,
    priority INTEGER DEFAULT 0,  -- Higher = more urgent
    progress_pct REAL DEFAULT 0.0,
    current_timestep INTEGER DEFAULT 0,
    total_timesteps INTEGER DEFAULT 0,
    convergence_data JSONB DEFAULT '[]',  -- [{timestep, energy_ratio, dt}]
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX jobs_sim_idx ON simulation_jobs(simulation_id);
CREATE INDEX jobs_worker_idx ON simulation_jobs(worker_id);

-- Material Models (metadata; actual model files in S3)
CREATE TABLE materials (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    grade TEXT NOT NULL,  -- e.g., "DP780", "AA6016-T4"
    material_type TEXT NOT NULL,  -- steel | aluminum | magnesium | titanium | copper | superalloy
    manufacturer TEXT,  -- e.g., "ArcelorMittal", "Novelis"
    thickness_min_mm REAL,
    thickness_max_mm REAL,
    yield_strength_mpa REAL,
    tensile_strength_mpa REAL,
    elongation_pct REAL,
    r_value REAL,  -- Lankford coefficient (average)
    n_value REAL,  -- Strain hardening exponent
    yield_model TEXT NOT NULL,  -- von_mises | hill48 | yld2000 | bbc2005
    hardening_model TEXT NOT NULL,  -- isotropic | kinematic | combined
    flc_url TEXT,  -- S3 URL to forming limit curve data
    stress_strain_url TEXT NOT NULL,  -- S3 URL to true stress-strain curve
    anisotropy_params JSONB,  -- Hill48 R-values, Barlat coefficients
    description TEXT,
    tags TEXT[] DEFAULT '{}',
    is_verified BOOLEAN DEFAULT false,
    is_builtin BOOLEAN DEFAULT true,
    uploaded_by UUID REFERENCES users(id),
    download_count INTEGER DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX materials_grade_idx ON materials(grade);
CREATE INDEX materials_type_idx ON materials(material_type);
CREATE INDEX materials_grade_trgm_idx ON materials USING gin(grade gin_trgm_ops);
CREATE INDEX materials_tags_idx ON materials USING gin(tags);

-- Die Templates (pre-configured tool geometries)
CREATE TABLE die_templates (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    category TEXT NOT NULL,  -- cup_draw | box_draw | flange | hem | progressive | transfer
    thumbnail_url TEXT,
    die_geometry JSONB NOT NULL,
    settings JSONB DEFAULT '{}',
    description TEXT,
    download_count INTEGER DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX templates_category_idx ON die_templates(category);

-- Saved Result Configurations
CREATE TABLE result_configs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    simulation_id UUID NOT NULL REFERENCES simulations(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    contour_type TEXT NOT NULL,  -- thinning | strain | stress | springback
    view_settings JSONB NOT NULL,  -- {camera_position, color_scale, iso_surfaces}
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX result_configs_sim_idx ON result_configs(simulation_id);

-- Usage Tracking (for billing enforcement)
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,  -- simulation_minutes | gpu_minutes | storage_gb
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
    pub process_type: String,
    pub die_geometry: serde_json::Value,
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
    pub mesh_data_url: Option<String>,
    pub element_count: i32,
    pub node_count: i32,
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
    pub gpu_allocated: bool,
    pub priority: i32,
    pub progress_pct: f32,
    pub current_timestep: i32,
    pub total_timesteps: i32,
    pub convergence_data: serde_json::Value,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Material {
    pub id: Uuid,
    pub grade: String,
    pub material_type: String,
    pub manufacturer: Option<String>,
    pub thickness_min_mm: Option<f32>,
    pub thickness_max_mm: Option<f32>,
    pub yield_strength_mpa: Option<f32>,
    pub tensile_strength_mpa: Option<f32>,
    pub elongation_pct: Option<f32>,
    pub r_value: Option<f32>,
    pub n_value: Option<f32>,
    pub yield_model: String,
    pub hardening_model: String,
    pub flc_url: Option<String>,
    pub stress_strain_url: String,
    pub anisotropy_params: Option<serde_json::Value>,
    pub description: Option<String>,
    pub tags: Vec<String>,
    pub is_verified: bool,
    pub is_builtin: bool,
    pub uploaded_by: Option<Uuid>,
    pub download_count: i32,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct SimulationParams {
    pub blank_holder_force_n: f64,
    pub friction_coefficient: f64,
    pub draw_depth_mm: f64,
    pub punch_velocity_mm_s: f64,
    pub temperature_c: f64,
    pub mesh_size_mm: f64,
    pub adaptive_remesh: bool,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct MaterialParams {
    pub grade: String,
    pub thickness_mm: f64,
    pub rolling_direction_deg: f64,  // 0, 45, 90
}

#[derive(Debug, Deserialize, Serialize)]
pub struct BlankGeometry {
    pub shape: String,  // circle | rectangle | polygon
    pub dimensions: Vec<f64>,  // [radius] or [width, height] or [[x,y],...]
    pub material: MaterialParams,
}
```

---

## Solver Architecture Deep-Dive

### Governing Equations and Discretization

FormSim's core solver implements **explicit dynamic finite element analysis** for sheet metal forming. For a shell element mesh with `n` nodes and `m` elements, the governing equation is the principle of virtual work:

```
M ü + C u̇ = F_ext - F_int

where:
M     = Lumped mass matrix (diagonal for explicit)
C     = Damping matrix (optional, typically mass-proportional)
ü, u̇ = Nodal acceleration and velocity vectors
F_ext = External forces (contact, blank holder force, gravity)
F_int = Internal forces (element stresses integrated over volume)
```

**Shell element formulation** uses the Belytschko-Tsay 4-node underintegrated shell element (the industry standard for sheet metal forming):

- **Mid-surface nodes**: 4 nodes with 6 DOF each (3 translations + 3 rotations)
- **Integration**: Single Gauss point in-plane (2×2 reduced to 1) + 7 through-thickness integration points for plasticity
- **Hourglass control**: Stabilization forces added to suppress zero-energy modes from underintegration
- **Membrane + bending**: Combined membrane and bending strains with Kirchhoff-Love thin-shell assumption

**Explicit time integration** uses the central difference method:

```
a_n = M^(-1) (F_ext,n - F_int,n)  # Acceleration at step n
v_{n+1/2} = v_{n-1/2} + a_n Δt     # Velocity at mid-step
u_{n+1} = u_n + v_{n+1/2} Δt       # Displacement at step n+1

where Δt is adaptive timestep based on Courant condition:
Δt = α min(L_e / c_d)  for all elements

L_e   = Element characteristic length
c_d   = Dilatational wave speed = sqrt((E(1-ν))/((1+ν)(1-2ν)ρ))
α     = Scale factor (typically 0.9 for stability)
```

**Plasticity** is integrated via the radial return algorithm with anisotropic yield:

```
Trial stress: σ_trial = σ_n + E Δε_elastic
Yield check:  f(σ_trial) = σ_eq(σ_trial) - σ_y(ε_p) > 0?

If yielding:
  Plastic multiplier: Δλ = f / (H + 3G)  (von Mises)
  Return mapping: σ_{n+1} = σ_trial - 2G Δλ ∂f/∂σ
  Update plastic strain: ε_p += Δλ

For Hill48 anisotropy:
  σ_eq² = F(σ_y - σ_z)² + G(σ_z - σ_x)² + H(σ_x - σ_y)² + 2Lτ_yz² + 2Mτ_zx² + 2Nτ_xy²

  where F, G, H, L, M, N are functions of Lankford R-values:
  F = 1/(1+R_0), G = R_0/(1+R_0), H = R_0/(1+R_0), N = (R_0+R_90)(1+2R_45)/(2R_90(1+R_0))
```

**Contact** uses penalty-based enforcement:

```
If penetration d < 0 (node penetrated tool surface):
  Contact force: F_c = k_p |d| n̂  (normal direction)
  Friction force: F_f = μ |F_c| t̂  (tangential, Coulomb friction)

  where k_p = penalty stiffness (elastic_modulus × 100 / element_size)
```

**Springback** is computed via implicit static analysis after forming:

```
K(u) Δu = -R(u)  (Newton-Raphson)

where:
K(u) = Tangent stiffness matrix (from current stress state)
R(u) = Residual (internal forces after removing tools)
Δu   = Incremental displacement correction

Iterate until ||R|| < tolerance (typically 1% of applied force)
```

### Client/Server Split (WASM Threshold)

```
Model uploaded → Element count extracted
    │
    ├── ≤20K elements → WASM solver (browser)
    │   ├── Instant startup, no network latency
    │   ├── Results displayed immediately (~5-30 seconds)
    │   └── No server cost
    │
    └── >20K elements → Server solver (Rust native + optional GPU)
        ├── Job queued via Redis
        ├── Worker picks up, runs native solver (optionally on GPU)
        ├── Progress streamed via WebSocket
        └── Results stored in S3, URL returned
```

The 20,000-element threshold was chosen because:
- WASM solver with 20K elements handles typical single-part stamping (100mm × 100mm blank, 2mm mesh) in <30 seconds on modern hardware
- 20K elements covers: cup draws (5K-15K), simple box stampings (10K-20K), flanging operations (3K-10K)
- Above 20K elements: multi-stage tooling, large automotive panels (50K-200K), progressive dies → need server compute and GPU acceleration

### WASM Compilation Pipeline

```toml
# solver-wasm/Cargo.toml
[package]
name = "formsim-solver-wasm"
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
rayon = { version = "1.10", features = ["wasm-bindgen"] }  # WASM threads
log = "0.4"
console_log = "1"

[profile.release]
opt-level = "z"       # Optimize for size
lto = true            # Link-time optimization
codegen-units = 1     # Single codegen unit for better optimization
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
      - run: wasm-opt -Oz solver-wasm/pkg/formsim_solver_wasm_bg.wasm -o solver-wasm/pkg/formsim_solver_wasm_bg.wasm
      - name: Upload to S3/CDN
        run: aws s3 cp solver-wasm/pkg/ s3://formsim-wasm/v${{ github.sha }}/ --recursive
```

---

## Architecture Deep-Dives

### 1. Simulation API Handler (Rust/Axum)

The primary endpoint receives a simulation request, validates the die geometry and blank, decides between WASM and server execution, and for server execution enqueues a job with Redis and streams progress via WebSocket.

```rust
// src/api/handlers/simulation.rs

use axum::{
    extract::{Path, State, Json},
    http::StatusCode,
    response::IntoResponse,
};
use uuid::Uuid;
use crate::{
    db::models::{Simulation, SimulationParams, BlankGeometry},
    mesh::generator::generate_blank_mesh,
    state::AppState,
    auth::Claims,
    error::ApiError,
};

#[derive(serde::Deserialize)]
pub struct CreateSimulationRequest {
    pub simulation_type: String,  // forming | springback | multi_stage
    pub parameters: SimulationParams,
    pub blank: BlankGeometry,
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

    // 2. Generate blank mesh from geometry
    let mesh = generate_blank_mesh(&req.blank, req.parameters.mesh_size_mm)?;
    let element_count = mesh.element_count();
    let node_count = mesh.node_count();

    // 3. Check plan limits
    let user = sqlx::query_as!(
        crate::db::models::User,
        "SELECT * FROM users WHERE id = $1",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    if user.plan == "pro" && element_count > 50000 {
        return Err(ApiError::PlanLimit(
            "Pro plan supports models up to 50K elements. Upgrade to Advanced for unlimited."
        ));
    }

    // 4. Determine execution mode
    let execution_mode = if element_count <= 20000 {
        "wasm"
    } else if element_count <= 100000 {
        "server"
    } else {
        "gpu"  // Large models use GPU acceleration
    };

    // 5. Upload mesh to S3 for server execution
    let mesh_url = if execution_mode != "wasm" {
        let mesh_bytes = mesh.to_binary()?;
        let s3_key = format!("meshes/{}/{}.mesh", project_id, Uuid::new_v4());
        state.s3.put_object()
            .bucket("formsim-data")
            .key(&s3_key)
            .body(mesh_bytes.into())
            .content_type("application/octet-stream")
            .send()
            .await?;
        Some(format!("s3://formsim-data/{}", s3_key))
    } else {
        None
    };

    // 6. Create simulation record
    let sim = sqlx::query_as!(
        Simulation,
        r#"INSERT INTO simulations
            (project_id, user_id, simulation_type, status, execution_mode,
             mesh_data_url, element_count, node_count, parameters)
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9)
        RETURNING *"#,
        project_id,
        claims.user_id,
        req.simulation_type,
        if execution_mode == "wasm" { "completed" } else { "pending" },
        execution_mode,
        mesh_url,
        element_count as i32,
        node_count as i32,
        serde_json::to_value(&req.parameters)?,
    )
    .fetch_one(&state.db)
    .await?;

    // 7. For server execution, enqueue job
    if execution_mode != "wasm" {
        let priority = match user.plan.as_str() {
            "enterprise" => 20,
            "advanced" => 10,
            _ => 0,
        };

        let job = sqlx::query_as!(
            crate::db::models::SimulationJob,
            r#"INSERT INTO simulation_jobs
                (simulation_id, priority, gpu_allocated, cores_allocated)
            VALUES ($1, $2, $3, $4) RETURNING *"#,
            sim.id,
            priority,
            execution_mode == "gpu",
            if execution_mode == "gpu" { 16 } else { 8 },
        )
        .fetch_one(&state.db)
        .await?;

        // Publish to Redis job queue
        let queue = if execution_mode == "gpu" { "simulation:gpu_jobs" } else { "simulation:jobs" };
        state.redis
            .publish(queue, serde_json::to_string(&job.id)?)
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

### 2. Explicit FEA Solver Core (Rust — shared between WASM and native)

The core explicit solver that performs time integration for the forming simulation. This code compiles to both native (server) and WASM (browser) targets.

```rust
// solver-core/src/explicit.rs

use nalgebra::{Vector3, Matrix3};
use crate::shell::{ShellElement, ShellMesh};
use crate::material::{Material, YieldModel};
use crate::contact::{ContactPair, detect_contact, compute_contact_force};

pub struct ExplicitSolver {
    pub mesh: ShellMesh,
    pub material: Material,
    pub tools: Vec<ToolGeometry>,

    // State vectors
    pub displacement: Vec<Vector3<f64>>,
    pub velocity: Vec<Vector3<f64>>,
    pub acceleration: Vec<Vector3<f64>>,
    pub mass: Vec<f64>,  // Lumped mass (diagonal)

    // Integration parameters
    pub timestep: f64,
    pub current_time: f64,
    pub target_time: f64,
    pub mass_scaling: f64,  // For faster convergence in quasi-static

    // Contact state
    pub contact_pairs: Vec<ContactPair>,
    pub friction_coeff: f64,
}

impl ExplicitSolver {
    pub fn new(mesh: ShellMesh, material: Material, tools: Vec<ToolGeometry>) -> Self {
        let n_nodes = mesh.nodes.len();
        let mass = Self::compute_lumped_mass(&mesh, &material);

        Self {
            mesh,
            material,
            tools,
            displacement: vec![Vector3::zeros(); n_nodes],
            velocity: vec![Vector3::zeros(); n_nodes],
            acceleration: vec![Vector3::zeros(); n_nodes],
            mass,
            timestep: 0.0,
            current_time: 0.0,
            target_time: 1.0,
            mass_scaling: 1.0,
            contact_pairs: Vec::new(),
            friction_coeff: 0.15,
        }
    }

    /// Compute lumped mass matrix (diagonal, row-sum from consistent mass)
    fn compute_lumped_mass(mesh: &ShellMesh, material: &Material) -> Vec<f64> {
        let mut mass = vec![0.0; mesh.nodes.len()];

        for elem in &mesh.elements {
            let elem_mass = material.density * elem.area() * elem.thickness / 4.0;
            for &node_idx in &elem.nodes {
                mass[node_idx] += elem_mass;
            }
        }

        mass
    }

    /// Compute stable timestep via Courant condition
    fn compute_timestep(&self) -> f64 {
        let mut dt_min = f64::INFINITY;

        // Wave speed: c_d = sqrt(E(1-nu) / (rho(1+nu)(1-2nu)))
        let e = self.material.youngs_modulus;
        let nu = self.material.poissons_ratio;
        let rho = self.material.density;
        let c_d = ((e * (1.0 - nu)) / (rho * (1.0 + nu) * (1.0 - 2.0 * nu))).sqrt();

        for elem in &self.mesh.elements {
            let char_length = elem.characteristic_length();
            let dt_elem = char_length / c_d;
            dt_min = dt_min.min(dt_elem);
        }

        0.9 * dt_min * self.mass_scaling.sqrt()  // Safety factor
    }

    /// Main time integration loop
    pub fn solve(&mut self) -> Result<FormingResult, SolverError> {
        let mut result = FormingResult::new();

        // Initial timestep
        self.timestep = self.compute_timestep();

        let mut step_count = 0;
        while self.current_time < self.target_time {
            step_count += 1;

            // 1. Detect contact
            self.contact_pairs = detect_contact(&self.mesh, &self.tools, &self.displacement);

            // 2. Compute internal forces (element stress)
            let f_internal = self.compute_internal_forces();

            // 3. Compute external forces (contact, BHF, gravity)
            let f_external = self.compute_external_forces();

            // 4. Compute acceleration (a = M^-1 (F_ext - F_int))
            for i in 0..self.mesh.nodes.len() {
                if self.mass[i] > 1e-12 {
                    self.acceleration[i] = (f_external[i] - f_internal[i]) / self.mass[i];
                } else {
                    self.acceleration[i] = Vector3::zeros();
                }
            }

            // 5. Apply boundary conditions (fixed nodes)
            self.apply_boundary_conditions();

            // 6. Central difference time integration
            for i in 0..self.mesh.nodes.len() {
                self.velocity[i] += self.acceleration[i] * self.timestep;
                self.displacement[i] += self.velocity[i] * self.timestep;
            }

            self.current_time += self.timestep;

            // 7. Adaptive timestep (recompute every 10 steps)
            if step_count % 10 == 0 {
                self.timestep = self.compute_timestep();
            }

            // 8. Save checkpoint for result visualization
            if step_count % 50 == 0 {
                result.add_frame(self.current_time, &self.displacement, &self.compute_strains());
            }
        }

        // Final result processing
        result.thinning = self.compute_thinning();
        result.fld_status = self.check_forming_limit();
        result.max_thinning_pct = result.thinning.iter().cloned().fold(0.0, f64::max) * 100.0;

        Ok(result)
    }

    /// Compute internal forces from element stresses
    fn compute_internal_forces(&mut self) -> Vec<Vector3<f64>> {
        let mut forces = vec![Vector3::zeros(); self.mesh.nodes.len()];

        for elem in &mut self.mesh.elements {
            // Get element nodal displacements
            let u_elem: Vec<Vector3<f64>> = elem.nodes.iter()
                .map(|&i| self.displacement[i])
                .collect();

            // Compute strains at Gauss point
            let (strain, _b_matrix) = elem.compute_strain(&u_elem);

            // Constitutive update (plasticity)
            let stress = self.material.update_stress(&strain, &mut elem.history);

            // Internal force = B^T * sigma * volume
            let f_elem = elem.compute_internal_force(&stress);

            // Assemble into global force vector
            for (local_idx, &global_idx) in elem.nodes.iter().enumerate() {
                forces[global_idx] += f_elem[local_idx];
            }
        }

        forces
    }

    /// Compute external forces (contact, BHF, gravity)
    fn compute_external_forces(&self) -> Vec<Vector3<f64>> {
        let mut forces = vec![Vector3::zeros(); self.mesh.nodes.len()];

        // Contact forces
        for contact in &self.contact_pairs {
            let (f_normal, f_friction) = compute_contact_force(
                contact, &self.velocity, self.friction_coeff
            );
            forces[contact.node_idx] += f_normal + f_friction;
        }

        // Gravity
        let g = Vector3::new(0.0, 0.0, -9.81);
        for i in 0..self.mesh.nodes.len() {
            forces[i] += self.mass[i] * g;
        }

        forces
    }

    /// Apply boundary conditions
    fn apply_boundary_conditions(&mut self) {
        // For forming: blank holder nodes have prescribed displacement
        // (handled via contact with rigid tools in this implementation)
    }

    /// Compute thinning at each element
    fn compute_thinning(&self) -> Vec<f64> {
        self.mesh.elements.iter().map(|elem| {
            let thickness_final = elem.history.thickness;
            let thickness_initial = elem.thickness;
            (thickness_initial - thickness_final) / thickness_initial
        }).collect()
    }

    /// Compute strains for visualization
    fn compute_strains(&self) -> Vec<StrainState> {
        self.mesh.elements.iter().map(|elem| {
            elem.history.strain_state.clone()
        }).collect()
    }

    /// Check forming limit diagram (FLD)
    fn check_forming_limit(&self) -> Vec<FLDStatus> {
        let flc = self.material.forming_limit_curve.as_ref()
            .expect("Material must have FLC for formability check");

        self.mesh.elements.iter().map(|elem| {
            let major = elem.history.strain_state.major_strain;
            let minor = elem.history.strain_state.minor_strain;

            let limit = flc.lookup(minor);
            if major > limit {
                FLDStatus::Split
            } else if major > 0.9 * limit {
                FLDStatus::Warning
            } else {
                FLDStatus::Safe
            }
        }).collect()
    }
}

#[derive(Debug)]
pub struct FormingResult {
    pub frames: Vec<ResultFrame>,
    pub thinning: Vec<f64>,
    pub fld_status: Vec<FLDStatus>,
    pub max_thinning_pct: f64,
}

impl FormingResult {
    pub fn new() -> Self {
        Self {
            frames: Vec::new(),
            thinning: Vec::new(),
            fld_status: Vec::new(),
            max_thinning_pct: 0.0,
        }
    }

    pub fn add_frame(&mut self, time: f64, displacement: &[Vector3<f64>], strains: &[StrainState]) {
        self.frames.push(ResultFrame {
            time,
            displacement: displacement.to_vec(),
            strains: strains.to_vec(),
        });
    }
}

#[derive(Debug, Clone)]
pub enum FLDStatus {
    Safe,
    Warning,
    Split,
}

#[derive(Debug)]
pub enum SolverError {
    Convergence(String),
    InvalidMesh(String),
}
```

### 3. Three.js Result Viewer Component (React + WebGPU)

The frontend 3D viewer that renders the formed part with color-mapped contours for thinning, strain, and springback.

```typescript
// frontend/src/components/ResultViewer.tsx

import { useRef, useEffect, useCallback, useState } from 'react';
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';
import { useResultStore } from '../stores/resultStore';

interface ResultViewerProps {
  simulationId: string;
}

export function ResultViewer({ simulationId }: ResultViewerProps) {
  const canvasRef = useRef<HTMLCanvasElement>(null);
  const sceneRef = useRef<THREE.Scene | null>(null);
  const rendererRef = useRef<THREE.WebGLRenderer | null>(null);
  const meshRef = useRef<THREE.Mesh | null>(null);

  const {
    resultData,
    contourType,
    colorScale,
    setColorScale,
    animationFrame,
    setAnimationFrame
  } = useResultStore();

  const [playing, setPlaying] = useState(false);

  // Initialize Three.js scene
  useEffect(() => {
    if (!canvasRef.current) return;

    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x1a1a1a);
    sceneRef.current = scene;

    const camera = new THREE.PerspectiveCamera(
      45,
      canvasRef.current.clientWidth / canvasRef.current.clientHeight,
      0.1,
      1000
    );
    camera.position.set(0, 0, 150);

    const renderer = new THREE.WebGLRenderer({
      canvas: canvasRef.current,
      antialias: true,
      powerPreference: 'high-performance'
    });
    renderer.setSize(canvasRef.current.clientWidth, canvasRef.current.clientHeight);
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    rendererRef.current = renderer;

    const controls = new OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true;
    controls.dampingFactor = 0.05;

    // Lighting
    const ambientLight = new THREE.AmbientLight(0xffffff, 0.6);
    scene.add(ambientLight);
    const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8);
    directionalLight.position.set(50, 50, 50);
    scene.add(directionalLight);

    // Animation loop
    const animate = () => {
      requestAnimationFrame(animate);
      controls.update();
      renderer.render(scene, camera);
    };
    animate();

    // Handle window resize
    const handleResize = () => {
      if (!canvasRef.current) return;
      camera.aspect = canvasRef.current.clientWidth / canvasRef.current.clientHeight;
      camera.updateProjectionMatrix();
      renderer.setSize(canvasRef.current.clientWidth, canvasRef.current.clientHeight);
    };
    window.addEventListener('resize', handleResize);

    return () => {
      window.removeEventListener('resize', handleResize);
      renderer.dispose();
      controls.dispose();
    };
  }, []);

  // Load and display result mesh
  useEffect(() => {
    if (!sceneRef.current || !resultData) return;

    // Remove old mesh
    if (meshRef.current) {
      sceneRef.current.remove(meshRef.current);
      meshRef.current.geometry.dispose();
      if (Array.isArray(meshRef.current.material)) {
        meshRef.current.material.forEach(m => m.dispose());
      } else {
        meshRef.current.material.dispose();
      }
    }

    // Create geometry from result data
    const geometry = new THREE.BufferGeometry();

    // Get frame data (for animation)
    const frame = resultData.frames[animationFrame] || resultData.frames[0];

    // Vertices (displaced positions)
    const vertices = new Float32Array(frame.displacement.length * 3);
    for (let i = 0; i < frame.displacement.length; i++) {
      vertices[i * 3] = resultData.mesh.nodes[i].x + frame.displacement[i].x;
      vertices[i * 3 + 1] = resultData.mesh.nodes[i].y + frame.displacement[i].y;
      vertices[i * 3 + 2] = resultData.mesh.nodes[i].z + frame.displacement[i].z;
    }
    geometry.setAttribute('position', new THREE.BufferAttribute(vertices, 3));

    // Faces (element connectivity)
    const indices = new Uint32Array(resultData.mesh.elements.length * 6);  // 2 triangles per quad
    for (let i = 0; i < resultData.mesh.elements.length; i++) {
      const elem = resultData.mesh.elements[i];
      // Quad to triangles: [0,1,2] and [0,2,3]
      indices[i * 6] = elem.nodes[0];
      indices[i * 6 + 1] = elem.nodes[1];
      indices[i * 6 + 2] = elem.nodes[2];
      indices[i * 6 + 3] = elem.nodes[0];
      indices[i * 6 + 4] = elem.nodes[2];
      indices[i * 6 + 5] = elem.nodes[3];
    }
    geometry.setIndex(new THREE.BufferAttribute(indices, 1));

    // Vertex colors based on contour type
    const colors = computeVertexColors(resultData, frame, contourType, colorScale);
    geometry.setAttribute('color', new THREE.BufferAttribute(colors, 3));

    geometry.computeVertexNormals();

    // Material with vertex colors
    const material = new THREE.MeshPhongMaterial({
      vertexColors: true,
      side: THREE.DoubleSide,
      flatShading: false,
      shininess: 30,
    });

    const mesh = new THREE.Mesh(geometry, material);
    sceneRef.current.add(mesh);
    meshRef.current = mesh;

  }, [resultData, animationFrame, contourType, colorScale]);

  // Animation playback
  useEffect(() => {
    if (!playing || !resultData) return;

    const interval = setInterval(() => {
      setAnimationFrame((prev) => {
        if (prev >= resultData.frames.length - 1) {
          setPlaying(false);
          return 0;
        }
        return prev + 1;
      });
    }, 100);  // 10 FPS playback

    return () => clearInterval(interval);
  }, [playing, resultData, setAnimationFrame]);

  return (
    <div className="result-viewer flex flex-col h-full">
      <div className="toolbar flex items-center gap-4 px-4 py-2 bg-gray-800 border-b border-gray-700">
        <select
          value={contourType}
          onChange={(e) => useResultStore.setState({ contourType: e.target.value })}
          className="px-3 py-1 bg-gray-700 rounded"
        >
          <option value="thinning">Thinning</option>
          <option value="major_strain">Major Strain</option>
          <option value="minor_strain">Minor Strain</option>
          <option value="springback">Springback</option>
          <option value="stress">von Mises Stress</option>
        </select>

        <button
          onClick={() => setPlaying(!playing)}
          className="px-4 py-1 bg-blue-600 hover:bg-blue-700 rounded"
        >
          {playing ? 'Pause' : 'Play'}
        </button>

        <input
          type="range"
          min={0}
          max={(resultData?.frames.length || 1) - 1}
          value={animationFrame}
          onChange={(e) => setAnimationFrame(parseInt(e.target.value))}
          className="flex-1"
        />

        <span className="text-sm text-gray-400">
          Frame {animationFrame + 1} / {resultData?.frames.length || 1}
        </span>
      </div>

      <div className="flex-1 relative">
        <canvas ref={canvasRef} className="w-full h-full" />
        <ColorScaleLegend contourType={contourType} colorScale={colorScale} />
      </div>

      <FormabilitySummary resultData={resultData} />
    </div>
  );
}

// Compute vertex colors from element data via averaging
function computeVertexColors(
  resultData: any,
  frame: any,
  contourType: string,
  colorScale: { min: number; max: number }
): Float32Array {
  const nNodes = resultData.mesh.nodes.length;
  const colors = new Float32Array(nNodes * 3);
  const vertexValues = new Float32Array(nNodes);
  const vertexCounts = new Int32Array(nNodes);

  // Accumulate element values to vertices
  for (let i = 0; i < resultData.mesh.elements.length; i++) {
    const elem = resultData.mesh.elements[i];
    let value = 0;

    switch (contourType) {
      case 'thinning':
        value = resultData.thinning[i] * 100;  // Convert to percentage
        break;
      case 'major_strain':
        value = frame.strains[i].major_strain * 100;
        break;
      case 'minor_strain':
        value = frame.strains[i].minor_strain * 100;
        break;
      case 'springback':
        value = frame.springback ? frame.springback[i] : 0;
        break;
      case 'stress':
        value = frame.strains[i].von_mises_stress;
        break;
    }

    for (const nodeIdx of elem.nodes) {
      vertexValues[nodeIdx] += value;
      vertexCounts[nodeIdx]++;
    }
  }

  // Average and map to colors
  for (let i = 0; i < nNodes; i++) {
    const avgValue = vertexCounts[i] > 0 ? vertexValues[i] / vertexCounts[i] : 0;
    const normalized = (avgValue - colorScale.min) / (colorScale.max - colorScale.min);
    const clamped = Math.max(0, Math.min(1, normalized));

    // Color map: blue (low) -> green -> yellow -> red (high)
    const rgb = valueToColor(clamped);
    colors[i * 3] = rgb.r;
    colors[i * 3 + 1] = rgb.g;
    colors[i * 3 + 2] = rgb.b;
  }

  return colors;
}

function valueToColor(t: number): { r: number; g: number; b: number } {
  // Jet colormap approximation
  const r = Math.max(0, Math.min(1, 1.5 - Math.abs(4 * t - 3)));
  const g = Math.max(0, Math.min(1, 1.5 - Math.abs(4 * t - 2)));
  const b = Math.max(0, Math.min(1, 1.5 - Math.abs(4 * t - 1)));
  return { r, g, b };
}

function ColorScaleLegend({ contourType, colorScale }: any) {
  return (
    <div className="absolute top-4 right-4 bg-gray-800 bg-opacity-90 p-3 rounded">
      <div className="text-sm font-semibold mb-2">{contourType}</div>
      <div className="w-8 h-40 bg-gradient-to-b from-red-500 via-yellow-500 via-green-500 to-blue-500"></div>
      <div className="text-xs mt-1">{colorScale.max.toFixed(2)}</div>
      <div className="text-xs mt-10">{colorScale.min.toFixed(2)}</div>
    </div>
  );
}

function FormabilitySummary({ resultData }: any) {
  if (!resultData) return null;

  const safeCount = resultData.fld_status.filter((s: string) => s === 'Safe').length;
  const warningCount = resultData.fld_status.filter((s: string) => s === 'Warning').length;
  const splitCount = resultData.fld_status.filter((s: string) => s === 'Split').length;
  const total = resultData.fld_status.length;

  return (
    <div className="p-4 bg-gray-800 border-t border-gray-700">
      <div className="grid grid-cols-4 gap-4 text-center">
        <div>
          <div className="text-2xl font-bold">{resultData.max_thinning_pct.toFixed(1)}%</div>
          <div className="text-xs text-gray-400">Max Thinning</div>
        </div>
        <div>
          <div className="text-2xl font-bold text-green-500">{safeCount}</div>
          <div className="text-xs text-gray-400">Safe ({(100 * safeCount / total).toFixed(1)}%)</div>
        </div>
        <div>
          <div className="text-2xl font-bold text-yellow-500">{warningCount}</div>
          <div className="text-xs text-gray-400">Warning ({(100 * warningCount / total).toFixed(1)}%)</div>
        </div>
        <div>
          <div className="text-2xl font-bold text-red-500">{splitCount}</div>
          <div className="text-xs text-gray-400">Split ({(100 * splitCount / total).toFixed(1)}%)</div>
        </div>
      </div>
    </div>
  );
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–4)

**Day 1: Project initialization and Rust backend scaffold**
```bash
cargo init formsim-api
cd formsim-api
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
- `migrations/001_initial.sql` — All 9 tables: users, organizations, org_members, projects, simulations, simulation_jobs, materials, die_templates, result_configs, usage_records
- `src/db/mod.rs` — Database pool initialization
- `src/db/models.rs` — All SQLx structs with FromRow derives
- Run `sqlx migrate run` to apply schema
- Seed script for initial materials (DC04, DP600, DP780, AA5182, AA6016)

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

### Phase 2 — Solver Core — Shell FEA Prototype (Days 5–12)

**Day 5: Shell element framework**
- `solver-core/` — New Rust workspace member (shared between native and WASM)
- `solver-core/src/shell/element.rs` — ShellElement struct (4-node Belytschko-Tsay quad)
- `solver-core/src/shell/mesh.rs` — ShellMesh struct with nodes, elements, connectivity
- `solver-core/src/shell/shape_functions.rs` — Bilinear shape functions and derivatives
- Unit tests: single element strain computation, Jacobian determinant

**Day 6: Material models — elastoplasticity**
- `solver-core/src/material/elastic.rs` — Linear elastic constitutive model
- `solver-core/src/material/plasticity.rs` — von Mises plasticity with isotropic hardening
- `solver-core/src/material/anisotropic.rs` — Hill48 anisotropic yield surface
- Radial return algorithm for plastic integration
- Tests: uniaxial tension, pure shear, biaxial loading

**Day 7: Explicit time integration**
- `solver-core/src/explicit.rs` — ExplicitSolver struct with central difference integration
- `solver-core/src/explicit/timestep.rs` — Courant condition for stable timestep
- `solver-core/src/explicit/mass.rs` — Lumped mass matrix computation
- Tests: single element wave propagation, patch test (uniform stress)

**Day 8: Internal force computation**
- `solver-core/src/shell/internal_force.rs` — Element internal force via B-matrix
- Through-thickness integration with 7 Gauss points (Simpson's rule)
- Hourglass control for underintegrated elements
- Tests: cantilever beam bending (compare to analytical deflection)

**Day 9: Contact detection and enforcement**
- `solver-core/src/contact/detection.rs` — Node-to-surface contact detection with spatial hashing
- `solver-core/src/contact/penalty.rs` — Penalty-based contact force computation
- `solver-core/src/contact/friction.rs` — Coulomb friction model
- Tests: rigid sphere contact, sliding block with friction

**Day 10: Tool geometry representation**
- `solver-core/src/tool/geometry.rs` — ToolGeometry struct (triangulated surface mesh)
- `solver-core/src/tool/import.rs` — Import from STL (tessellated die face)
- `solver-core/src/tool/normal.rs` — Surface normal computation and interpolation
- Bounding box and spatial indexing for fast contact queries

**Day 11: Blank mesh generation**
- `solver-core/src/mesh/blank.rs` — Generate quad mesh for circular, rectangular, and polygonal blanks
- `solver-core/src/mesh/generator.rs` — Structured quad meshing with target element size
- `solver-core/src/mesh/quality.rs` — Mesh quality metrics (aspect ratio, skewness)
- Tests: mesh convergence study (compare 1mm, 2mm, 4mm meshes)

**Day 12: Forming simulation integration test**
- Implement cup draw benchmark: flat circular blank → cylindrical cup
- Compare to analytical prediction: drawing force, wall thickness distribution
- Verify max thinning location (cup bottom radius)
- Tests: ≤10% error vs. analytical for simple geometries

### Phase 3 — WASM Build + Frontend 3D Viewer (Days 13–18)

**Day 13: WASM solver compilation**
- `solver-wasm/` — New workspace member for WASM target
- `solver-wasm/Cargo.toml` — wasm-bindgen, serde-wasm-bindgen dependencies
- `solver-wasm/src/lib.rs` — WASM entry points: `run_forming_simulation()`
- `wasm-pack build --target web --release` pipeline
- `wasm-opt -Oz` for size optimization (target <5MB gzipped)
- JavaScript wrapper: `FormSimSolver` class that loads WASM and provides async API

**Day 14: Frontend scaffold and Three.js setup**
```bash
npm create vite@latest frontend -- --template react-ts
cd frontend
npm i zustand @tanstack/react-query axios three @react-three/fiber @react-three/drei
```
- `src/App.tsx` — Router, auth context, layout
- `src/stores/projectStore.ts` — Zustand store for project state
- `src/stores/resultStore.ts` — Result visualization state (contour type, animation frame)
- `src/components/ResultViewer/Scene.tsx` — Three.js scene with OrbitControls
- Basic sphere rendering test

**Day 15: Mesh rendering and contour shading**
- `src/components/ResultViewer/MeshRenderer.tsx` — BufferGeometry from result data
- `src/components/ResultViewer/ContourShader.tsx` — Vertex color computation (thinning, strain)
- Color scale legend component
- Tests: render pre-computed cup draw result, verify colors

**Day 16: Animation playback**
- `src/components/ResultViewer/AnimationControls.tsx` — Play/pause, scrubber, frame counter
- Frame-by-frame displacement update
- Smooth interpolation between frames (optional)
- Tests: playback performance with 100-frame result

**Day 17: Die face import (OpenCascade WASM)**
- `npm i opencascade.js` — WASM build of OpenCascade CAD kernel
- `src/lib/cadImport.ts` — Load STEP/IGES, extract surface geometry, tessellate to STL
- `src/components/DieSetup/CADImporter.tsx` — File upload, preview, mesh quality control
- Tests: import sample STEP file, verify vertex count

**Day 18: Die setup UI**
- `src/components/DieSetup/DieSetupEditor.tsx` — Blank definition form (shape, size, material)
- `src/components/DieSetup/MaterialSelector.tsx` — Material library search and select
- `src/components/DieSetup/ProcessParams.tsx` — BHF, friction, draw depth inputs
- Preview: show blank + die assembly in 3D viewer
- Tests: create project, define circular blank, select DP780 material

### Phase 4 — API + Job Orchestration (Days 19–24)

**Day 19: Simulation API endpoints**
- `src/api/handlers/simulation.rs` — Create simulation, get simulation, list simulations, cancel simulation
- Blank mesh generation from geometry definition
- Element count extraction and WASM/server/GPU routing logic
- Plan-based limits enforcement (Pro: 50K elements, Advanced: unlimited)

**Day 20: Server-side simulation worker**
- `src/workers/simulation_worker.rs` — Redis job consumer, runs native solver
- `src/workers/mod.rs` — Worker pool management (configurable concurrency)
- Progress streaming: worker publishes progress → Redis pub/sub → WebSocket to client
- Error handling: convergence failures, timeouts, out-of-memory
- S3 result upload with presigned URL generation for client download

**Day 21: WebSocket for live simulation progress**
- `src/api/ws/mod.rs` — Axum WebSocket upgrade handler
- `src/api/ws/simulation_progress.rs` — Subscribe to simulation progress channel
- Client receives: `{ progress_pct, current_timestep, total_timesteps, energy_ratio }`
- Connection management: heartbeat ping/pong, automatic cleanup on disconnect
- Frontend: `src/hooks/useSimulationProgress.ts` — React hook for WebSocket subscription

**Day 22: Material library API**
- `src/api/handlers/materials.rs` — Search materials (parametric + full-text), get material, download data
- S3 integration for material file storage (stress-strain curves, FLC data)
- Material categories: steel, aluminum, magnesium, titanium, copper, superalloy
- Parametric search: filter by type, thickness range, yield strength, UTS
- Pagination with cursor-based scrolling for large result sets

**Day 23: Die template library**
- `src/api/handlers/templates.rs` — List templates, get template, use template (create project from template)
- Pre-built templates: cup draw, box draw, flange, hem, progressive die stations
- Template thumbnails stored in S3
- Tests: use cup draw template, verify geometry loaded correctly

**Day 24: Project import/export**
- `src/api/handlers/export.rs` — Export project as JSON, export result mesh as STL/VTK
- `src/api/handlers/import.rs` — Import project from JSON, import die geometry from STEP/IGES
- Auto-save: frontend debounced PATCH every 5 seconds on geometry changes
- Tests: export → import → verify identical geometry

### Phase 5 — Springback + Formability Analysis (Days 25–29)

**Day 25: Implicit springback solver**
- `solver-core/src/implicit.rs` — Implicit static solver with Newton-Raphson
- `solver-core/src/implicit/stiffness.rs` — Tangent stiffness matrix assembly
- `solver-core/src/implicit/solver.rs` — Sparse direct solver (UMFPACK or similar)
- Remove contact constraints, apply gravity, solve for equilibrium
- Tests: beam springback after bending (compare to analytical)

**Day 26: Springback visualization**
- `src/components/ResultViewer/SpringbackView.tsx` — Overlay formed vs. springback geometry
- Displacement magnitude contour
- Angular deviation measurement
- GD&T-aware conformance check (compare to nominal CAD)
- Tests: verify springback magnitude for DP780 vs. DC04

**Day 27: Forming limit diagram (FLD) implementation**
- `solver-core/src/formability/fld.rs` — FLC data structure (major vs. minor strain curve)
- `solver-core/src/formability/check.rs` — Check element strain state against FLC
- Keeler-Brazier empirical FLC generation for materials without experimental data
- Tests: verify FLC lookup, split detection

**Day 28: FLD overlay visualization**
- `src/components/FormabilityPanel/FLDPlot.tsx` — 2D plot: major strain vs. minor strain with FLC curve
- Element scatter plot colored by safety margin
- Highlight critical elements on 3D mesh when clicked in FLD plot
- Tests: cup draw → verify elements in safe zone

**Day 29: Formability report generation**
- `src/api/handlers/report.rs` — Generate PDF report with thinning histogram, FLD overlay, summary statistics
- Python service: `report-service/` — FastAPI with matplotlib for plotting
- S3 upload of generated PDF
- Tests: generate report for cup draw, verify PDF contains expected plots

### Phase 6 — Billing + Plan Enforcement (Days 30–33)

**Day 30: Stripe integration**
- `src/api/handlers/billing.rs` — Create checkout session, create customer portal session, get subscription status
- `src/api/handlers/webhooks/stripe.rs` — Handle subscription events: `checkout.session.completed`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.payment_failed`
- Plan mapping:
  - Pro ($199/mo): 50K elements, 100 simulations/month, basic materials, thinning + FLD
  - Advanced ($449/user/mo): Unlimited elements, unlimited simulations, all materials, springback, multi-stage, API access
  - Enterprise (custom): On-premise, custom materials, die compensation, optimization

**Day 31: Usage tracking and plan limits**
- `src/middleware/plan_limits.rs` — Middleware checking plan limits before simulation execution
- `src/services/usage.rs` — Track simulations per billing period, GPU minutes
- Usage record insertion after each simulation completes
- Usage dashboard: `src/api/handlers/usage.rs` — Current period usage, historical usage
- Approaching-limit warnings at 80% and 100% of monthly quota

**Day 32: Billing UI**
- `frontend/src/pages/Billing.tsx` — Plan comparison table, current plan display, usage meter
- `frontend/src/components/billing/PlanCard.tsx` — Plan feature comparison cards
- `frontend/src/components/billing/UsageMeter.tsx` — Simulation quota usage bar
- Upgrade/downgrade flow via Stripe Customer Portal
- Upgrade prompt modals when hitting plan limits (e.g., "Model has 75K elements. Upgrade to Advanced for unlimited.")

**Day 33: Feature gating**
- Gate springback analysis behind Advanced plan
- Gate multi-stage forming behind Advanced plan
- Gate die compensation behind Enterprise plan
- Gate API access behind Advanced plan
- Show locked feature indicators with upgrade CTAs in UI
- Admin endpoint: override plan for specific users (beta testers, partners)

### Phase 7 — Solver Validation + Testing (Days 34–38)

**Day 34: Solver validation — benchmark problems**
- Benchmark 1: Yoshida buckling test — square blank drawn into rectangular die, verify wrinkle prediction
- Benchmark 2: NUMISHEET cup draw benchmark — verify max thinning location and magnitude
- Benchmark 3: S-rail springback — compare to experimental data from NUMISHEET 2005
- Automated test suite: `solver-core/tests/benchmarks.rs` with assertions within 10% tolerance

**Day 35: Material model validation**
- Validate Hill48 anisotropy: uniaxial tension at 0°, 45°, 90° to rolling direction
- Validate hardening: load-unload-reload cycle, verify Bauschinger effect with kinematic hardening
- Validate FLC: Marciniak test simulation, verify necking onset matches experimental FLC
- Compare stress-strain curves to material test data

**Day 36: Convergence and performance testing**
- Mesh convergence study: 1mm, 2mm, 4mm, 8mm element sizes → verify thinning converges
- Timestep sensitivity: verify results independent of timestep (within stability limit)
- Mass scaling: verify quasi-static results independent of mass scaling factor (1× to 100×)
- Large model performance: 100K-element model, measure wall time, memory usage

**Day 37: Integration testing**
- End-to-end test: create project → import die → define blank → run simulation → view results → generate report
- API integration tests: auth → project CRUD → simulation → springback → results
- WASM solver test: load in headless browser (Playwright), run simulation, verify results match native solver
- WebSocket test: connect → subscribe → receive progress → receive completion
- Concurrent simulation test: submit 10 simultaneous server-side jobs

**Day 38: Performance testing and optimization**
- WASM solver benchmarks: measure time for 5K, 10K, 20K element models in Chrome/Firefox/Safari
- Target: 20K-element model < 30 seconds in WASM
- Server solver benchmarks: measure throughput (simulations/hour) with 8-core workers
- Target: 100K-element model < 5 minutes on server (without GPU)
- GPU solver benchmarks: 500K-element model < 10 minutes on single GPU
- Frontend rendering: measure FPS with 50K-triangle mesh, contour shading
- Target: 60 FPS pan/zoom with 50K triangles

### Phase 8 — Deployment + Polish + Launch (Days 39–42)

**Day 39: Docker and Kubernetes configuration**
- `Dockerfile` — Multi-stage Rust build (cargo-chef for layer caching)
- `docker-compose.yml` — Full local dev stack (API, PostgreSQL, Redis, MinIO, report service, frontend)
- `k8s/` — Kubernetes manifests:
  - `api-deployment.yaml` — API server (3 replicas, HPA)
  - `worker-deployment.yaml` — CPU simulation workers (auto-scaling based on queue depth)
  - `gpu-worker-deployment.yaml` — GPU workers (g5.2xlarge with T4 GPUs)
  - `postgres-statefulset.yaml` — PostgreSQL with PVC
  - `redis-deployment.yaml` — Redis for job queue and pub/sub
  - `ingress.yaml` — NGINX ingress with TLS
- Health check endpoints: `/health/live`, `/health/ready`

**Day 40: CDN and asset delivery**
- CloudFront distribution for:
  - Frontend static assets (HTML, JS, CSS) — cached at edge
  - WASM solver bundle (~5MB) — cached at edge with long TTL and versioned URLs
  - Material files from S3 — cached with 1-hour TTL
  - Die templates and thumbnails — cached
- WASM preloading: start downloading WASM bundle on page load before user needs it
- Service worker for offline WASM caching (progressive enhancement)

**Day 41: Monitoring, logging, and polish**
- Prometheus metrics: simulation duration histogram, solver convergence rate, API latency percentiles, GPU utilization
- Grafana dashboards: system health, simulation throughput, user activity, error rates
- Sentry integration: frontend and backend error tracking with source maps
- Structured logging (tracing + JSON) for all API requests and solver events
- UI polish: loading states, error messages, empty states, responsive layout
- Accessibility: keyboard navigation in 3D viewer, ARIA labels on controls

**Day 42: Launch preparation and final testing**
- End-to-end smoke test in production environment
- Database backup and restore verification
- Rate limiting: 100 req/min per user, 10 simulations/min per user
- Security audit: JWT validation, SQL injection (SQLx prevents this), CORS configuration, CSP headers
- Landing page with product overview, pricing, and signup
- Documentation: getting started guide, material library reference, benchmark validation
- Deploy to production, enable monitoring alerts, announce launch

---

## Critical Files

```
formsim/
├── solver-core/                           # Shared solver library (native + WASM)
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── shell/
│   │   │   ├── mod.rs
│   │   │   ├── element.rs                 # Belytschko-Tsay shell element
│   │   │   ├── mesh.rs                    # Mesh data structure
│   │   │   ├── shape_functions.rs         # Shape functions and derivatives
│   │   │   └── internal_force.rs          # Internal force computation
│   │   ├── material/
│   │   │   ├── mod.rs
│   │   │   ├── elastic.rs                 # Linear elasticity
│   │   │   ├── plasticity.rs              # von Mises plasticity
│   │   │   ├── anisotropic.rs             # Hill48, Yld2000-2d
│   │   │   ├── hardening.rs               # Isotropic + kinematic hardening
│   │   │   └── damage.rs                  # GISSMO damage model
│   │   ├── explicit.rs                    # Explicit time integration
│   │   ├── implicit.rs                    # Implicit springback solver
│   │   ├── contact/
│   │   │   ├── mod.rs
│   │   │   ├── detection.rs               # Node-to-surface contact detection
│   │   │   ├── penalty.rs                 # Penalty enforcement
│   │   │   └── friction.rs                # Coulomb friction
│   │   ├── tool/
│   │   │   ├── mod.rs
│   │   │   ├── geometry.rs                # Tool surface representation
│   │   │   └── import.rs                  # STL import
│   │   ├── mesh/
│   │   │   ├── mod.rs
│   │   │   ├── blank.rs                   # Blank mesh generation
│   │   │   ├── generator.rs               # Quad meshing
│   │   │   └── quality.rs                 # Mesh quality metrics
│   │   └── formability/
│   │       ├── mod.rs
│   │       ├── fld.rs                     # Forming limit diagram
│   │       └── check.rs                   # Formability check
│   └── tests/
│       ├── benchmarks.rs                  # Solver validation benchmarks
│       ├── cup_draw.rs                    # Cup draw benchmark
│       └── springback.rs                  # Springback validation
│
├── solver-wasm/                           # WASM compilation target
│   ├── Cargo.toml
│   └── src/
│       └── lib.rs                         # WASM entry points (wasm-bindgen)
│
├── formsim-api/                           # Rust API server
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
│   │   │   │   ├── materials.rs           # Material search/get
│   │   │   │   ├── templates.rs           # Die template library
│   │   │   │   ├── billing.rs             # Stripe checkout/portal
│   │   │   │   ├── usage.rs               # Usage tracking
│   │   │   │   ├── export.rs              # STL/VTK/JSON export
│   │   │   │   ├── report.rs              # PDF report generation
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
│   │       ├── simulation_worker.rs       # CPU simulation execution
│   │       └── gpu_worker.rs              # GPU-accelerated execution
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
│   │   │   └── resultStore.ts             # Result visualization state
│   │   ├── hooks/
│   │   │   ├── useSimulationProgress.ts   # WebSocket hook for live progress
│   │   │   └── useWasmSolver.ts           # WASM solver hook (load + run)
│   │   ├── lib/
│   │   │   ├── api.ts                     # Axios API client
│   │   │   ├── cadImport.ts               # OpenCascade WASM STEP import
│   │   │   └── wasmLoader.ts              # WASM bundle loading
│   │   ├── pages/
│   │   │   ├── Dashboard.tsx              # Project list
│   │   │   ├── DieSetup.tsx               # Die geometry + blank setup
│   │   │   ├── Results.tsx                # Simulation results viewer
│   │   │   ├── Materials.tsx              # Material library browser
│   │   │   ├── Templates.tsx              # Die template gallery
│   │   │   ├── Billing.tsx                # Plan management
│   │   │   ├── Login.tsx                  # Auth pages
│   │   │   └── Register.tsx
│   │   ├── components/
│   │   │   ├── DieSetup/
│   │   │   │   ├── CADImporter.tsx        # STEP/IGES import
│   │   │   │   ├── BlankEditor.tsx        # Blank geometry definition
│   │   │   │   ├── MaterialSelector.tsx   # Material search and select
│   │   │   │   └── ProcessParams.tsx      # BHF, friction, draw depth
│   │   │   ├── ResultViewer/
│   │   │   │   ├── Scene.tsx              # Three.js scene
│   │   │   │   ├── MeshRenderer.tsx       # Mesh + contour shading
│   │   │   │   ├── AnimationControls.tsx  # Playback controls
│   │   │   │   ├── SpringbackView.tsx     # Springback overlay
│   │   │   │   └── ContourShader.tsx      # Color mapping
│   │   │   ├── FormabilityPanel/
│   │   │   │   ├── FLDPlot.tsx            # Major vs. minor strain plot
│   │   │   │   └── Summary.tsx            # Thinning summary
│   │   │   ├── billing/
│   │   │   │   ├── PlanCard.tsx
│   │   │   │   └── UsageMeter.tsx
│   │   │   └── common/
│   │   │       ├── SimulationToolbar.tsx   # Run/stop/export controls
│   │   │       └── MaterialCard.tsx        # Material display component
│   │   └── data/
│   │       └── templates/                 # Pre-built die templates (JSON)
│   └── public/
│       └── wasm/                          # WASM solver bundle (loaded at runtime)
│
├── report-service/                        # Python FastAPI (PDF report generation)
│   ├── requirements.txt
│   ├── main.py                            # FastAPI app
│   ├── generators/
│   │   ├── formability_report.py          # Generate formability PDF
│   │   └── plots.py                       # Matplotlib plots (FLD, histogram)
│   └── Dockerfile
│
├── k8s/                                   # Kubernetes manifests
│   ├── api-deployment.yaml
│   ├── worker-deployment.yaml
│   ├── gpu-worker-deployment.yaml
│   ├── postgres-statefulset.yaml
│   ├── redis-deployment.yaml
│   ├── report-service-deployment.yaml
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

### Benchmark 1: NUMISHEET Cup Draw (Deep Drawing)

**Setup:** Circular blank (100mm diameter, 1mm thick DP780) drawn into cylindrical cup (50mm diameter punch, 52mm die opening, 10mm corner radius)

**Expected max thinning:** At punch corner radius, approximately **18-22%** thinning

**Expected draw force:** Peak force approximately **45-55 kN** (varies with BHF)

**Tolerance:** Thinning < 10%, force < 15%

### Benchmark 2: Yoshida Buckling Test (Wrinkle Prediction)

**Setup:** Square blank (300mm × 300mm, 0.8mm thick DC04) drawn into rectangular die (100mm × 200mm)

**Expected:** Wrinkles (compressive minor strain) form in flange regions along short edges

**Expected wrinkle height:** Approximately **5-8mm** peak-to-trough

**Tolerance:** Wrinkle location correct, height < 20%

### Benchmark 3: S-Rail Springback (NUMISHEET 2005)

**Setup:** S-shaped rail component (TRIP780, 1.2mm thick) formed then released

**Expected springback angle:** Section A rotation approximately **2.5-3.5°**

**Expected springback displacement:** Max vertical displacement approximately **15-20mm**

**Tolerance:** Angle < 15%, displacement < 20% (springback is notoriously difficult to predict accurately)

### Benchmark 4: Marciniak Test (Forming Limit Validation)

**Setup:** Circular blank with center groove drawn to failure

**Expected:** Necking onset occurs when major strain reaches FLC for given minor strain path

**Expected FLC0 (plane strain):** For DP780 approximately **0.22-0.28** major strain

**Tolerance:** FLC0 < 10% of experimental data

### Benchmark 5: Mesh Convergence (Convergence Study)

**Setup:** Cup draw with 1mm, 2mm, 4mm, 8mm element sizes

**Expected:** Max thinning converges to within **5%** between 1mm and 2mm meshes

**Expected:** Wall time scales approximately as O(n^1.5) for explicit solver

**Tolerance:** Thinning at 1mm and 2mm meshes differ by < 5%

---

## Verification

### Integration Testing Checklist

1. **Auth flow** — Register → login → JWT issued → authorized API calls → token refresh → logout
2. **Project CRUD** — Create → update die geometry → save → reload → verify geometry preserved
3. **Die import** — Upload STEP file → preview → tessellate → verify mesh quality
4. **Blank setup** — Define circular blank → select material (DP780) → set thickness (1mm) → preview
5. **WASM simulation** — Small model (5K elements) → WASM solver → results returned → 3D viewer displays
6. **Server simulation** — Large model (50K elements) → job queued → worker picks up → WebSocket progress → results in S3
7. **Result visualization** — Load 20K-element result → rotate/zoom → change contour type → verify colors
8. **Springback** — Run forming → run springback → overlay geometries → measure angular deviation
9. **FLD check** — View FLD plot → click element in split zone → 3D mesh highlights element
10. **Material library** — Search "DP780" → results returned → select material → use in project
11. **Template usage** — Select "Cup Draw" template → project created → simulate → expected thinning pattern
12. **Plan limits** — Pro user → model with 75K elements → blocked with upgrade prompt
13. **Billing** — Subscribe to Advanced → Stripe checkout → webhook → plan updated → limits lifted
14. **Concurrent users** — 5 users simultaneously running server simulations → all complete correctly
15. **Error handling** — Degenerate mesh → meaningful error message → no crash

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

-- 2. Simulation type distribution
SELECT simulation_type, COUNT(*) as count,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 1) as pct
FROM simulations
WHERE created_at >= NOW() - INTERVAL '30 days'
GROUP BY simulation_type ORDER BY count DESC;

-- 3. User plan distribution
SELECT plan, COUNT(*) as users,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 1) as pct
FROM users GROUP BY plan ORDER BY users DESC;

-- 4. Simulation usage by user (billing period)
SELECT u.email, u.plan,
    COUNT(s.id) as simulations_count,
    CASE u.plan
        WHEN 'pro' THEN 100
        WHEN 'advanced' THEN 999999
        WHEN 'enterprise' THEN 999999
    END as limit
FROM users u
LEFT JOIN simulations s ON u.id = s.user_id
    AND s.created_at >= DATE_TRUNC('month', CURRENT_DATE)
GROUP BY u.email, u.plan
ORDER BY simulations_count DESC;

-- 5. Material popularity
SELECT m.grade, m.material_type,
    m.download_count,
    COUNT(DISTINCT s.project_id) as projects_using
FROM materials m
LEFT JOIN simulations s ON s.parameters->>'material_grade' = m.grade
GROUP BY m.id
ORDER BY m.download_count DESC
LIMIT 20;
```

### Performance Benchmarks

| Operation | Target | Method |
|-----------|--------|--------|
| WASM solver: 5K-element forming | <10s | Browser benchmark (Chrome DevTools) |
| WASM solver: 20K-element forming | <30s | Browser benchmark |
| Server solver: 50K-element forming | <3min | Server timing, 8 cores |
| Server solver: 100K-element forming | <10min | Server timing, 8 cores |
| GPU solver: 500K-element forming | <10min | Server timing, T4 GPU |
| Springback: 20K-element model | <30s | Server timing, 8 cores |
| 3D viewer: 50K-triangle mesh render | 60 FPS | Chrome FPS counter during rotation |
| STEP import: 1MB file | <5s | Browser timing with OpenCascade WASM |
| API: create simulation | <200ms | p95 latency (Prometheus) |
| API: search materials | <100ms | p95 latency with 1K materials in DB |
| WASM bundle load (cached) | <500ms | Service worker cache hit |
| WASM bundle load (cold) | <5s | CDN delivery, 5MB gzipped |
| WebSocket: progress latency | <100ms | Time from worker emit to client receive |
| Mesh generation: 10K elements | <1s | Server-side Rust mesher |

---

## Deployment Architecture

### Infrastructure

```
┌─────────────────────────────────────────────────┐
│                  CloudFront CDN                   │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────┐ │
│  │ Frontend SPA │  │ WASM Bundle │  │ Material │ │
│  │ (HTML/JS/CSS)│  │ (versioned) │  │ Files    │ │
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
│ Multi-AZ     │ │ Cluster mode │ │  Materials)  │
└──────────────┘ └──────┬───────┘ └──────────────┘
                        │
        ┌───────────────┼───────────────────┐
        ▼               ▼                   ▼
┌──────────────┐ ┌──────────────┐   ┌──────────────┐
│ CPU Worker   │ │ CPU Worker   │   │ GPU Worker   │
│ (Rust)       │ │ (Rust)       │   │ (Rust+CUDA)  │
│ c7g.2xlarge  │ │ c7g.2xlarge  │   │ g5.2xlarge   │
│ HPA: 2-10    │ │              │   │ HPA: 0-5     │
└──────────────┘ └──────────────┘   └──────────────┘

                 ┌──────────────┐
                 │ Report Svc   │
                 │ (Python/     │
                 │  FastAPI)    │
                 └──────────────┘
```

### Scaling Strategy

- **API servers**: Horizontal pod autoscaler (HPA) at 70% CPU, min 3, max 10
- **CPU workers**: HPA based on Redis queue depth (`simulation:jobs`) — scale up when queue > 3 jobs, scale down when idle 5 min, min 2, max 10
- **GPU workers**: HPA based on GPU queue depth (`simulation:gpu_jobs`) — scale up when queue > 1 job, scale down when idle 10 min, min 0, max 5
- **Worker instances**: AWS c7g.2xlarge (8 vCPU, 16GB RAM) for CPU workers, g5.2xlarge (8 vCPU, 24GB RAM, 1× NVIDIA T4) for GPU workers
- **Database**: RDS PostgreSQL r6g.xlarge with read replicas for material search queries
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
        { "Days": 30, "StorageClass": "STANDARD_IA" },
        { "Days": 90, "StorageClass": "GLACIER" }
      ],
      "Expiration": { "Days": 365 }
    },
    {
      "ID": "material-files-keep",
      "Prefix": "materials/",
      "Status": "Enabled",
      "Transitions": [
        { "Days": 180, "StorageClass": "STANDARD_IA" }
      ]
    },
    {
      "ID": "mesh-files-temp",
      "Prefix": "meshes/",
      "Status": "Enabled",
      "Expiration": { "Days": 7 }
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
| Multi-Stage Forming | Sequential operations (draw → trim → flange → springback) with automatic transfer and trimmed-edge handling for progressive and transfer dies | High |
| Die Compensation | Automatic die surface morphing via radial basis functions (RBF) to compensate for springback, with iterative simulate → compensate → re-simulate loop until conformance | High |
| Advanced Anisotropic Models | Barlat Yld2000-2d and BBC2005 yield surfaces for accurate prediction of aluminum and AHSS forming, with 8+ R-value inputs for full planar anisotropy | High |
| Blank Shape Optimization | Genetic algorithm optimization of initial blank outline to minimize trim scrap while avoiding splits and wrinkles, with automated blank nesting | Medium |
| Blank Holder Force Optimization | Trajectory optimization of BHF vs. punch displacement to minimize wrinkling and splitting, with force-controlled servo press simulation | Medium |
| Hot Stamping | Temperature-dependent material properties, austenitization heating, rapid die quenching, and martensite phase transformation for press-hardened steels (PHS) | Medium |
| Microstructure Prediction | JMAK grain size evolution during hot forming and forging with temperature history tracking for property prediction in titanium and superalloys | Medium |
| Forging and Extrusion | Closed-die and open-die forging with flash formation, direct and indirect extrusion with billet heating and preform design optimization | Low |
| Die Stress Analysis | Tool stress and fatigue life estimation for punch, die, and binder with elastic tool deformation and die breakage prediction | Low |
| Real-Time Collaboration | CRDT-based die geometry editing with cursor presence, commenting on results, and shared simulation history for team design reviews | Low |
