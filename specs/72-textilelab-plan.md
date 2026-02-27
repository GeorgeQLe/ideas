# 72. TextileLab — Textile and Fabric Engineering Simulation Platform

## Implementation Plan

**MVP Scope:** Browser-based textile structure designer with drag-and-drop yarn path editor for woven fabric patterns (plain, twill, satin, custom weave repeats) rendered via SVG with interactive 3D yarn geometry visualization using Three.js/WebGL, custom textile mechanics solver implementing Classical Laminate Theory (CLT) for homogenized stiffness prediction and kinematic draping solver using fishnetting algorithm compiled to WebAssembly for client-side execution of fabrics ≤10,000 elements and server-side Rust-native execution for larger drapes, support for unit cell generation with periodic boundary conditions, fiber volume fraction calculation, yarn crimp analysis, and shear angle mapping during drape, interactive 3D drape viewer rendered via Three.js with fiber orientation visualization and wrinkle zone highlighting, 500+ built-in yarn material definitions (carbon fiber, glass fiber, aramid, cotton, polyester, nylon) stored in S3 with PostgreSQL metadata including tensile modulus, density, diameter, twist parameters, STL mold geometry import for draping target surfaces, fabric specification export (JSON, CSV) and drape results export (fiber angle maps, shear distribution), Stripe billing with three tiers (Academic $49/month / Pro $149/month / Enterprise custom).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| Textile Solver | Rust (native + WASM) | Custom CLT homogenization + kinematic draping engine |
| WASM Compilation | `wasm-pack` + `wasm-bindgen` | Client-side solver for fabrics ≤10K elements |
| Geometry Processing | Python 3.12 (FastAPI) | STL mesh processing, yarn path generation, permeability estimation |
| Database | PostgreSQL 16 | Projects, users, fabrics, simulations, yarn material metadata |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | Material definitions, simulation results, STL files, drape meshes |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| Structure Editor | Custom SVG renderer | 2D weave pattern editor with yarn interlacement visualization |
| 3D Viewer | Three.js (WebGL) | 3D yarn geometry, drape visualization, fiber orientation mapping |
| Real-time | WebSocket (Axum) | Live draping progress, convergence monitoring |
| Job Queue | Redis 7 + Tokio tasks | Server-side draping job management, large mesh processing |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | WASM bundle delivery, static frontend assets |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, error tracking |
| CI/CD | GitHub Actions | WASM build pipeline, Rust tests, Docker image push |

### Key Architecture Decisions

1. **Hybrid client/server solver with WASM threshold at 10K elements**: Fabrics with ≤10,000 draping elements (covers 90%+ of simple drapes on basic molds) run entirely in the browser via WASM, providing instant feedback with zero server cost. Larger draping simulations (automotive interior panels, composite layup on complex molds, multi-ply stacks) are submitted to the Rust-native server which handles meshes up to 1M+ elements with parallel processing. The threshold is configurable per plan tier.

2. **Custom textile mechanics engine in Rust rather than wrapping TexGen**: Building a custom CLT homogenization and kinematic draping solver in Rust gives us full control over material models, convergence algorithms, WASM compilation, and parallelization for parameter sweeps. TexGen's C++ codebase is research-focused with limited draping support and no WASM compatibility. Our Rust solver uses sparse linear algebra crates (`nalgebra-sparse`, `faer`) for stiffness matrix assembly while maintaining memory safety and cross-platform compilation.

3. **SVG weave pattern editor with Three.js 3D preview**: The 2D weave editor uses SVG for crisp rendering of yarn interlacement patterns (warp/weft crossover diagrams) with intuitive click-to-toggle interface. This generates the yarn topology which is then converted to 3D geometry (cylindrical yarn cross-sections with elliptical deformation) rendered via Three.js. Separation of 2D pattern definition and 3D visualization allows fast pattern iteration without expensive geometry updates until user commits changes.

4. **Kinematic draping (fishnetting) for MVP, FEA draping post-MVP**: Kinematic draping based on pin-jointing and fishnet algorithms provides fast (~1-5 seconds) initial draping solutions suitable for feasibility assessment and geometric defect prediction. This covers the primary MVP use case: "Can I drape this fabric over this mold without wrinkling?" Full FEA-based draping with nonlinear shell elements and contact mechanics is deferred to post-MVP as it requires 10-100x more computation and is needed mainly for high-precision composite tooling.

5. **S3 for material library and mesh storage with PostgreSQL metadata catalog**: Yarn material definitions (JSON files with mechanical properties, typical 1-5KB each) and STL mold geometries (binary files, 100KB-50MB) are stored in S3, while PostgreSQL holds searchable metadata (material name, fiber type, tensile modulus, density, manufacturer). This allows the material library to scale to 10K+ materials without bloating the database while enabling fast parametric search via PostgreSQL indexes and full-text search.

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
    plan TEXT NOT NULL DEFAULT 'academic',  -- academic | pro | enterprise
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX users_email_idx ON users(email);
CREATE INDEX users_stripe_idx ON users(stripe_customer_id);

-- Organizations (for multi-user teams)
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

-- Projects (fabric + draping workspace)
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    fabric_data JSONB NOT NULL DEFAULT '{}',  -- Weave pattern, yarn specs, unit cell
    mold_stl_url TEXT,  -- S3 URL for mold geometry
    settings JSONB DEFAULT '{}',  -- Unit system, visualization options
    is_public BOOLEAN NOT NULL DEFAULT false,
    forked_from UUID REFERENCES projects(id) ON DELETE SET NULL,
    thumbnail_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);
CREATE INDEX projects_org_idx ON projects(org_id);
CREATE INDEX projects_updated_idx ON projects(updated_at DESC);

-- Draping Simulations
CREATE TABLE simulations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    simulation_type TEXT NOT NULL,  -- kinematic_drape | clt_homogenization | permeability
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | running | completed | failed | cancelled
    execution_mode TEXT NOT NULL DEFAULT 'wasm',  -- wasm | server
    parameters JSONB NOT NULL DEFAULT '{}',  -- Simulation-specific parameters
    element_count INTEGER NOT NULL DEFAULT 0,
    results_url TEXT,  -- S3 URL for result data (mesh, fiber angles, shear angles)
    results_summary JSONB,  -- Quick-access summary (max shear angle, wrinkle zones, Fvf)
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
    convergence_data JSONB DEFAULT '[]',  -- [{iteration, residual, max_shear}]
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX jobs_sim_idx ON simulation_jobs(simulation_id);
CREATE INDEX jobs_worker_idx ON simulation_jobs(worker_id);

-- Yarn Materials (metadata; actual definitions in S3)
CREATE TABLE yarn_materials (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,  -- e.g., "Toray T700 12K Carbon"
    manufacturer TEXT,
    fiber_type TEXT NOT NULL,  -- carbon | glass | aramid | polyester | nylon | cotton | wool
    form TEXT NOT NULL,  -- monofilament | multifilament | staple | tape
    linear_density REAL,  -- tex (g/km)
    filament_count INTEGER,  -- For multifilament
    diameter_mm REAL,
    tensile_modulus_gpa REAL NOT NULL,
    tensile_strength_mpa REAL,
    density_g_cm3 REAL NOT NULL,
    twist_tpm REAL,  -- Twists per meter (if applicable)
    material_url TEXT NOT NULL,  -- S3 URL to JSON definition
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
CREATE INDEX materials_fiber_type_idx ON yarn_materials(fiber_type);
CREATE INDEX materials_manufacturer_idx ON yarn_materials(manufacturer);
CREATE INDEX materials_name_trgm_idx ON yarn_materials USING gin(name gin_trgm_ops);
CREATE INDEX materials_tags_idx ON yarn_materials USING gin(tags);

-- Fabric Templates (pre-defined weave patterns)
CREATE TABLE fabric_templates (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    category TEXT NOT NULL,  -- plain | twill | satin | custom
    weave_pattern JSONB NOT NULL,  -- 2D array of interlacement (0=weft over, 1=warp over)
    yarn_spacing JSONB NOT NULL,  -- {warp_spacing_mm, weft_spacing_mm}
    description TEXT,
    thumbnail_url TEXT,
    is_builtin BOOLEAN DEFAULT true,
    created_by UUID REFERENCES users(id),
    usage_count INTEGER DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX templates_category_idx ON fabric_templates(category);

-- Usage Tracking (for billing enforcement)
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,  -- drape_minutes | server_drape | storage_bytes
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
    pub fabric_data: serde_json::Value,
    pub mold_stl_url: Option<String>,
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
    pub parameters: serde_json::Value,
    pub element_count: i32,
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
pub struct YarnMaterial {
    pub id: Uuid,
    pub name: String,
    pub manufacturer: Option<String>,
    pub fiber_type: String,
    pub form: String,
    pub linear_density: Option<f64>,
    pub filament_count: Option<i32>,
    pub diameter_mm: Option<f64>,
    pub tensile_modulus_gpa: f64,
    pub tensile_strength_mpa: Option<f64>,
    pub density_g_cm3: f64,
    pub twist_tpm: Option<f64>,
    pub material_url: String,
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

#[derive(Debug, Deserialize)]
pub struct SimulationParams {
    pub simulation_type: SimulationType,
    pub kinematic_drape: Option<KinematicDrapeParams>,
    pub clt_homogenization: Option<CltParams>,
    pub temperature_c: f64,  // Default 20°C
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum SimulationType {
    KinematicDrape,
    CltHomogenization,
    Permeability,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct KinematicDrapeParams {
    pub mold_mesh_resolution: f64,  // Target element size in mm
    pub fabric_initial_orientation: f64,  // Degrees from horizontal
    pub pinning_points: Vec<[f64; 3]>,  // Initial fabric pin locations
    pub shear_locking_angle: f64,  // Shear angle limit (degrees) before wrinkle
    pub max_iterations: u32,
    pub convergence_tolerance: f64,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct CltParams {
    pub fiber_volume_fraction: f64,  // 0.0 to 0.7
    pub matrix_modulus_gpa: f64,
    pub matrix_poisson: f64,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct FabricDefinition {
    pub weave_pattern: WeavePattern,
    pub warp_yarn: YarnSpec,
    pub weft_yarn: YarnSpec,
    pub warp_spacing_mm: f64,
    pub weft_spacing_mm: f64,
    pub areal_weight_gsm: Option<f64>,  // Grams per square meter
}

#[derive(Debug, Serialize, Deserialize)]
pub struct WeavePattern {
    pub repeat_width: usize,   // Number of warp yarns in repeat
    pub repeat_height: usize,  // Number of weft yarns in repeat
    pub interlacement: Vec<Vec<u8>>,  // 2D array: 0=weft over, 1=warp over
}

#[derive(Debug, Serialize, Deserialize)]
pub struct YarnSpec {
    pub material_id: Uuid,
    pub linear_density_tex: f64,
    pub diameter_mm: f64,
    pub crimp_amplitude_mm: f64,  // Out-of-plane waviness
}
```

---

## Solver Architecture Deep-Dive

### Governing Equations and Discretization

TextileLab's core solver implements **Classical Laminate Theory (CLT)** for homogenized stiffness prediction and **kinematic draping via pin-jointing and fishnetting** for geometric draping simulation.

#### Classical Laminate Theory (CLT)

For a woven fabric modeled as a symmetric laminate with orthotropic plies, the stiffness matrix relates in-plane strains and curvatures to resultant forces and moments:

```
┌    ┐   ┌       ┐ ┌   ┐
│ N  │   │ A  B  │ │ ε⁰ │
│    │ = │       │ │    │
│ M  │   │ B  D  │ │ κ  │
└    ┘   └       ┘ └   ┘
```

Where:
- **N** = [Nₓ, Nᵧ, Nₓᵧ]ᵀ: In-plane force resultants (N/m)
- **M** = [Mₓ, Mᵧ, Mₓᵧ]ᵀ: Bending moment resultants (N·m/m)
- **ε⁰** = [εₓ⁰, εᵧ⁰, γₓᵧ⁰]ᵀ: Mid-plane strains
- **κ** = [κₓ, κᵧ, κₓᵧ]ᵀ: Curvatures
- **A**: Extensional stiffness matrix (N/m)
- **B**: Coupling stiffness matrix (N) — zero for symmetric laminates
- **D**: Bending stiffness matrix (N·m)

For a single orthotropic ply (yarn layer) with angle θ from reference direction:

```
A_ij = Σ(Q̄_ij)_k · (z_k - z_{k-1})
D_ij = (1/3) · Σ(Q̄_ij)_k · (z_k³ - z_{k-1}³)

Q̄_ij = T(θ) · Q · T(θ)ᵀ  (transformed reduced stiffness)

Q = [  E₁/(1-ν₁₂ν₂₁)      ν₁₂E₂/(1-ν₁₂ν₂₁)   0        ]
    [  ν₁₂E₂/(1-ν₁₂ν₂₁)  E₂/(1-ν₁₂ν₂₁)       0        ]
    [  0                   0                   G₁₂      ]
```

Where E₁, E₂ are yarn-direction and transverse moduli, ν₁₂ is Poisson's ratio, G₁₂ is shear modulus.

For woven fabrics, we model as two orthogonal plies (warp and weft) with spacing-based thickness distribution. Effective properties are computed from yarn properties via rule-of-mixtures:

```
E_yarn = E_fiber · V_f + E_matrix · (1 - V_f)
V_f = (yarn_area) / (unit_cell_area × thickness)
```

#### Kinematic Draping (Fishnetting Algorithm)

The kinematic draping model treats the fabric as an inextensible pin-jointed net. Yarns cannot stretch (ΔL/L = 0) but can shear (angle between warp and weft changes from 90°).

**Algorithm:**
1. Discretize mold surface into triangular mesh with characteristic element size `h`
2. Initialize fabric mesh as flat rectangular grid with nodes at yarn crossover points
3. Pin initial nodes to mold surface at specified locations
4. Iteratively project unpinned nodes onto mold:
   - For each unpinned node `i`, find target position that:
     - Maintains distance to pinned neighbors (yarn inextensibility)
     - Lies on mold surface (geometric constraint)
   - Solve constrained optimization:
     ```
     minimize: Σ((L_ij - L_ij⁰)² / L_ij⁰²)  (deviation from original yarn length)
     subject to: P_i ∈ mold_surface
                 |P_i - P_j| ≈ L_ij⁰  for all neighbors j
     ```
   - Use Newton-Raphson with geodesic path constraints on mold

5. Compute shear angle γ at each element:
   ```
   γ = 90° - arccos(û_warp · û_weft)
   ```
   Where û_warp, û_weft are unit vectors along warp and weft yarns

6. Flag wrinkle zones where |γ| > γ_lock (typically 45-60° for woven fabrics)

7. Compute fiber orientation angles relative to mold reference frame for composite layup

**Convergence:** Iterate until max node displacement < tolerance (typically 0.1% of element size) or max iterations reached.

### Client/Server Split (WASM Threshold)

```
Draping requested → Element count estimated
    │
    ├── ≤10K elements → WASM solver (browser)
    │   ├── Instant startup, no network latency
    │   ├── Results displayed immediately
    │   └── No server cost
    │
    └── >10K elements → Server solver (Rust native)
        ├── Job queued via Redis
        ├── Worker picks up, runs native solver (parallel)
        ├── Progress streamed via WebSocket
        └── Results stored in S3, URL returned
```

The 10K element threshold was chosen because:
- WASM solver handles 10K element kinematic drape in <3 seconds on modern hardware
- 10K elements covers: simple molds (spheres, cylinders, flat panels), fabric swatches up to 500mm × 500mm at 5mm resolution
- Above 10K elements: automotive parts, large composite layups, multi-ply stacks → need server compute and parallelization

### WASM Compilation Pipeline

```toml
# solver-wasm/Cargo.toml
[package]
name = "textilelab-solver-wasm"
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
      - run: wasm-opt -Oz solver-wasm/pkg/textilelab_solver_wasm_bg.wasm -o solver-wasm/pkg/textilelab_solver_wasm_bg.wasm
      - name: Upload to S3/CDN
        run: aws s3 cp solver-wasm/pkg/ s3://textilelab-wasm/v${{ github.sha }}/ --recursive
```

---

## Architecture Deep-Dives

### 1. Draping API Handler (Rust/Axum)

The primary endpoint receives a draping request, validates the fabric definition and mold geometry, decides between WASM and server execution, and for server execution enqueues a job with Redis and streams progress via WebSocket.

```rust
// src/api/handlers/simulation.rs

use axum::{
    extract::{Path, State, Json},
    http::StatusCode,
    response::IntoResponse,
};
use uuid::Uuid;
use crate::{
    db::models::{Simulation, SimulationParams, SimulationType},
    solver::fabric::validate_fabric_definition,
    state::AppState,
    auth::Claims,
    error::ApiError,
};

#[derive(serde::Deserialize)]
pub struct CreateSimulationRequest {
    pub simulation_type: SimulationType,
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

    // 2. Validate fabric definition
    let fabric_def: crate::db::models::FabricDefinition =
        serde_json::from_value(project.fabric_data)?;
    validate_fabric_definition(&fabric_def)?;

    // 3. Estimate element count for execution mode decision
    let element_count = match req.simulation_type {
        SimulationType::KinematicDrape => {
            let drape_params: crate::db::models::KinematicDrapeParams =
                serde_json::from_value(req.parameters.clone())?;

            // Estimate from mold surface area and mesh resolution
            if let Some(stl_url) = &project.mold_stl_url {
                let surface_area = estimate_stl_surface_area(&state.s3, stl_url).await?;
                let element_size = drape_params.mold_mesh_resolution;
                (surface_area / (element_size * element_size)).ceil() as i32
            } else {
                return Err(ApiError::BadRequest("No mold geometry specified"));
            }
        }
        SimulationType::CltHomogenization => {
            // Unit cell homogenization is always fast, use WASM
            100
        }
        _ => 0,
    };

    // 4. Check plan limits
    let user = sqlx::query_as!(
        crate::db::models::User,
        "SELECT * FROM users WHERE id = $1",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    if user.plan == "academic" && element_count > 5000 {
        return Err(ApiError::PlanLimit(
            "Academic plan supports draping up to 5,000 elements. Upgrade to Pro for unlimited."
        ));
    }

    // 5. Determine execution mode
    let execution_mode = if element_count <= 10000 {
        "wasm"
    } else {
        "server"
    };

    // 6. Create simulation record
    let sim = sqlx::query_as!(
        Simulation,
        r#"INSERT INTO simulations
            (project_id, user_id, simulation_type, status, execution_mode,
             parameters, element_count)
        VALUES ($1, $2, $3, $4, $5, $6, $7)
        RETURNING *"#,
        project_id,
        claims.user_id,
        serde_json::to_string(&req.simulation_type)?,
        if execution_mode == "wasm" { "completed" } else { "pending" },
        execution_mode,
        req.parameters,
        element_count,
    )
    .fetch_one(&state.db)
    .await?;

    // 7. For server execution, enqueue job
    if execution_mode == "server" {
        let job = sqlx::query_as!(
            crate::db::models::SimulationJob,
            r#"INSERT INTO simulation_jobs (simulation_id, priority, cores_allocated)
            VALUES ($1, $2, $3) RETURNING *"#,
            sim.id,
            if user.plan == "enterprise" { 10 } else if user.plan == "pro" { 5 } else { 0 },
            if element_count > 100000 { 8 } else { 4 },
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

async fn estimate_stl_surface_area(
    s3: &aws_sdk_s3::Client,
    stl_url: &str,
) -> Result<f64, ApiError> {
    // Simple heuristic: extract from cached metadata or compute via Python service
    // For MVP, return reasonable default based on URL metadata
    Ok(250000.0)  // 500mm × 500mm default
}
```

### 2. Kinematic Draping Solver Core (Rust — shared between WASM and native)

The core kinematic draping engine that projects fabric mesh onto mold surface while maintaining yarn inextensibility.

```rust
// solver-core/src/draping/kinematic.rs

use nalgebra::{Point3, Vector3, DMatrix};
use crate::geometry::{TriangleMesh, closest_point_on_mesh};
use crate::fabric::FabricMesh;

pub struct KinematicDraper {
    pub mold: TriangleMesh,
    pub fabric: FabricMesh,
    pub pinned_nodes: Vec<usize>,
    pub shear_lock_angle: f64,  // radians
    pub tolerance: f64,
    pub max_iterations: usize,
}

#[derive(Debug)]
pub struct DrapeResult {
    pub fabric_mesh: FabricMesh,
    pub shear_angles: Vec<f64>,  // Per element (radians)
    pub fiber_angles_warp: Vec<f64>,  // Warp orientation (radians from mold x-axis)
    pub fiber_angles_weft: Vec<f64>,  // Weft orientation
    pub wrinkle_zones: Vec<usize>,  // Element indices where |shear| > threshold
    pub converged: bool,
    pub iterations: usize,
}

impl KinematicDraper {
    pub fn new(
        mold: TriangleMesh,
        fabric: FabricMesh,
        pinned_nodes: Vec<usize>,
        shear_lock_angle: f64,
    ) -> Self {
        Self {
            mold,
            fabric,
            pinned_nodes,
            shear_lock_angle,
            tolerance: 1e-4,
            max_iterations: 500,
        }
    }

    pub fn drape(&mut self) -> Result<DrapeResult, DrapeError> {
        let mut iteration = 0;
        let n_nodes = self.fabric.nodes.len();

        // Initialize unpinned node set
        let mut unpinned: Vec<usize> = (0..n_nodes)
            .filter(|&i| !self.pinned_nodes.contains(&i))
            .collect();

        // Project pinned nodes onto mold
        for &node_idx in &self.pinned_nodes {
            let pos = self.fabric.nodes[node_idx];
            let projected = closest_point_on_mesh(&self.mold, &pos)?;
            self.fabric.nodes[node_idx] = projected;
        }

        loop {
            if iteration >= self.max_iterations {
                return Ok(DrapeResult {
                    fabric_mesh: self.fabric.clone(),
                    shear_angles: self.compute_shear_angles(),
                    fiber_angles_warp: self.compute_fiber_angles(true),
                    fiber_angles_weft: self.compute_fiber_angles(false),
                    wrinkle_zones: self.identify_wrinkle_zones(),
                    converged: false,
                    iterations: iteration,
                });
            }

            let mut max_displacement = 0.0;

            // Process unpinned nodes in wavefront order (from pinned boundary)
            for &node_idx in &unpinned {
                let neighbors = self.fabric.get_node_neighbors(node_idx);

                // Skip if no neighbors are pinned/converged yet
                if neighbors.iter().all(|&n| unpinned.contains(&n)) {
                    continue;
                }

                // Compute target position that maintains yarn lengths
                let target = self.compute_constrained_position(node_idx, &neighbors)?;

                // Project onto mold surface
                let projected = closest_point_on_mesh(&self.mold, &target)?;

                let displacement = (projected - self.fabric.nodes[node_idx]).norm();
                max_displacement = max_displacement.max(displacement);

                self.fabric.nodes[node_idx] = projected;
            }

            iteration += 1;

            // Check convergence
            if max_displacement < self.tolerance {
                break;
            }

            // Update unpinned set: remove nodes that have converged
            unpinned.retain(|&idx| {
                let neighbors = self.fabric.get_node_neighbors(idx);
                neighbors.iter().any(|&n| unpinned.contains(&n))
            });

            if unpinned.is_empty() {
                break;
            }
        }

        Ok(DrapeResult {
            fabric_mesh: self.fabric.clone(),
            shear_angles: self.compute_shear_angles(),
            fiber_angles_warp: self.compute_fiber_angles(true),
            fiber_angles_weft: self.compute_fiber_angles(false),
            wrinkle_zones: self.identify_wrinkle_zones(),
            converged: true,
            iterations: iteration,
        })
    }

    fn compute_constrained_position(
        &self,
        node_idx: usize,
        neighbors: &[usize],
    ) -> Result<Point3<f64>, DrapeError> {
        // Solve for position that minimizes deviation from original yarn lengths
        let current_pos = self.fabric.nodes[node_idx];
        let original_lengths = neighbors.iter()
            .map(|&n| self.fabric.original_edge_length(node_idx, n))
            .collect::<Vec<_>>();

        let mut pos = current_pos;
        for _ in 0..5 {
            let mut jacobian = DMatrix::<f64>::zeros(neighbors.len(), 3);
            let mut residual = vec![0.0; neighbors.len()];

            for (i, &neighbor_idx) in neighbors.iter().enumerate() {
                let neighbor_pos = self.fabric.nodes[neighbor_idx];
                let vec = pos - neighbor_pos;
                let dist = vec.norm();

                if dist < 1e-10 {
                    continue;
                }

                residual[i] = dist - original_lengths[i];

                for j in 0..3 {
                    jacobian[(i, j)] = vec[j] / dist;
                }
            }

            let jt = jacobian.transpose();
            let jtj = &jt * &jacobian;
            let jtr = &jt * DMatrix::from_vec(residual.len(), 1, residual);

            if let Some(jtj_inv) = jtj.try_inverse() {
                let delta = jtj_inv * jtr;
                pos.x -= delta[(0, 0)];
                pos.y -= delta[(1, 0)];
                pos.z -= delta[(2, 0)];
            }
        }

        Ok(pos)
    }

    fn compute_shear_angles(&self) -> Vec<f64> {
        let mut shear_angles = Vec::new();

        for element in &self.fabric.elements {
            let p0 = self.fabric.nodes[element.nodes[0]];
            let p1 = self.fabric.nodes[element.nodes[1]];
            let p2 = self.fabric.nodes[element.nodes[2]];
            let p3 = self.fabric.nodes[element.nodes[3]];

            let warp = ((p1 - p0) + (p2 - p3)).normalize();
            let weft = ((p3 - p0) + (p2 - p1)).normalize();

            let dot = warp.dot(&weft);
            let angle = dot.acos();
            let shear = std::f64::consts::FRAC_PI_2 - angle;

            shear_angles.push(shear);
        }

        shear_angles
    }

    fn compute_fiber_angles(&self, is_warp: bool) -> Vec<f64> {
        let mut angles = Vec::new();

        for element in &self.fabric.elements {
            let p0 = self.fabric.nodes[element.nodes[0]];
            let p1 = self.fabric.nodes[element.nodes[1]];
            let p3 = self.fabric.nodes[element.nodes[3]];

            let fiber_dir = if is_warp {
                (p1 - p0).normalize()
            } else {
                (p3 - p0).normalize()
            };

            let normal = element.compute_normal(&self.fabric.nodes);
            let tangent_x = Vector3::new(1.0, 0.0, 0.0);
            let projected = fiber_dir - normal * fiber_dir.dot(&normal);

            let angle = projected.dot(&tangent_x).acos();
            angles.push(angle);
        }

        angles
    }

    fn identify_wrinkle_zones(&self) -> Vec<usize> {
        let shear_angles = self.compute_shear_angles();
        shear_angles.iter()
            .enumerate()
            .filter(|(_, &angle)| angle.abs() > self.shear_lock_angle)
            .map(|(idx, _)| idx)
            .collect()
    }
}

#[derive(Debug)]
pub enum DrapeError {
    InvalidMesh(String),
    ConvergenceFailure(String),
    GeometryError(String),
}
```

### 3. CLT Homogenization Solver (Rust)

Implements Classical Laminate Theory for computing effective stiffness properties of woven fabric unit cells.

```rust
// solver-core/src/clt/homogenization.rs

use nalgebra::{Matrix3, Matrix6};
use crate::db::models::{FabricDefinition, CltParams};

pub struct CltSolver {
    pub fabric: FabricDefinition,
    pub params: CltParams,
}

#[derive(Debug, Clone)]
pub struct HomogenizedProperties {
    pub extensional_stiffness: Matrix3<f64>,  // A matrix
    pub bending_stiffness: Matrix3<f64>,      // D matrix
    pub engineering_constants: EngineeringConstants,
}

#[derive(Debug, Clone)]
pub struct EngineeringConstants {
    pub ex: f64,          // Warp-direction modulus (GPa)
    pub ey: f64,          // Weft-direction modulus (GPa)
    pub gxy: f64,         // In-plane shear modulus (GPa)
    pub nu_xy: f64,       // Major Poisson's ratio
    pub nu_yx: f64,       // Minor Poisson's ratio
}

impl CltSolver {
    pub fn new(fabric: FabricDefinition, params: CltParams) -> Self {
        Self { fabric, params }
    }

    pub fn compute_properties(&self) -> Result<HomogenizedProperties, CltError> {
        // 1. Compute yarn-level properties from fiber properties
        let warp_modulus = self.compute_yarn_modulus(&self.fabric.warp_yarn)?;
        let weft_modulus = self.compute_yarn_modulus(&self.fabric.weft_yarn)?;

        // 2. Build reduced stiffness matrices for warp and weft plies
        let q_warp = self.reduced_stiffness(warp_modulus, 0.0)?;  // 0° orientation
        let q_weft = self.reduced_stiffness(weft_modulus, std::f64::consts::FRAC_PI_2)?;  // 90°

        // 3. Compute ply thicknesses from yarn geometry
        let t_warp = self.fabric.warp_yarn.diameter_mm;
        let t_weft = self.fabric.weft_yarn.diameter_mm;
        let total_thickness = t_warp + t_weft;

        // 4. Compute ABD matrix (A: extensional, B: coupling, D: bending)
        let mut a_matrix = Matrix3::zeros();
        let mut d_matrix = Matrix3::zeros();

        // Warp ply contribution
        let z0_warp = -total_thickness / 2.0;
        let z1_warp = z0_warp + t_warp;
        a_matrix += q_warp * (z1_warp - z0_warp);
        d_matrix += q_warp * (z1_warp.powi(3) - z0_warp.powi(3)) / 3.0;

        // Weft ply contribution
        let z0_weft = z1_warp;
        let z1_weft = z0_weft + t_weft;
        a_matrix += q_weft * (z1_weft - z0_weft);
        d_matrix += q_weft * (z1_weft.powi(3) - z0_weft.powi(3)) / 3.0;

        // 5. Invert A matrix to get engineering constants
        let a_inv = a_matrix.try_inverse()
            .ok_or(CltError::SingularMatrix)?;

        let ex = 1.0 / (a_inv[(0, 0)] * total_thickness);
        let ey = 1.0 / (a_inv[(1, 1)] * total_thickness);
        let gxy = 1.0 / (a_inv[(2, 2)] * total_thickness);
        let nu_xy = -a_inv[(0, 1)] / a_inv[(0, 0)];
        let nu_yx = -a_inv[(1, 0)] / a_inv[(1, 1)];

        Ok(HomogenizedProperties {
            extensional_stiffness: a_matrix,
            bending_stiffness: d_matrix,
            engineering_constants: EngineeringConstants {
                ex: ex / 1e9,  // Convert to GPa
                ey: ey / 1e9,
                gxy: gxy / 1e9,
                nu_xy,
                nu_yx,
            },
        })
    }

    fn compute_yarn_modulus(&self, yarn: &crate::db::models::YarnSpec) -> Result<f64, CltError> {
        // Rule of mixtures for yarn modulus
        let vf = self.params.fiber_volume_fraction;
        let fiber_modulus = 230e9;  // TODO: get from material database (Pa)
        let matrix_modulus = self.params.matrix_modulus_gpa * 1e9;

        Ok(fiber_modulus * vf + matrix_modulus * (1.0 - vf))
    }

    fn reduced_stiffness(&self, modulus: f64, theta: f64) -> Result<Matrix3<f64>, CltError> {
        // Compute Q matrix (reduced stiffness) and transform by angle theta
        let nu = 0.3;  // Typical Poisson's ratio for unidirectional composites
        let e1 = modulus;
        let e2 = modulus * 0.1;  // Transverse modulus ~10% of axial
        let g12 = modulus * 0.05;  // Shear modulus ~5% of axial

        let q11 = e1 / (1.0 - nu * nu);
        let q12 = nu * e2 / (1.0 - nu * nu);
        let q22 = e2 / (1.0 - nu * nu);
        let q66 = g12;

        // Transformation matrix
        let c = theta.cos();
        let s = theta.sin();
        let c2 = c * c;
        let s2 = s * s;
        let cs = c * s;

        // Transformed reduced stiffness
        let q11_bar = q11 * c2 * c2 + 2.0 * (q12 + 2.0 * q66) * s2 * c2 + q22 * s2 * s2;
        let q12_bar = (q11 + q22 - 4.0 * q66) * s2 * c2 + q12 * (s2 * s2 + c2 * c2);
        let q22_bar = q11 * s2 * s2 + 2.0 * (q12 + 2.0 * q66) * s2 * c2 + q22 * c2 * c2;
        let q16_bar = (q11 - q12 - 2.0 * q66) * cs * c2 + (q12 - q22 + 2.0 * q66) * cs * s2;
        let q26_bar = (q11 - q12 - 2.0 * q66) * cs * s2 + (q12 - q22 + 2.0 * q66) * cs * c2;
        let q66_bar = (q11 + q22 - 2.0 * q12 - 2.0 * q66) * s2 * c2 + q66 * (s2 * s2 + c2 * c2);

        Ok(Matrix3::new(
            q11_bar, q12_bar, q16_bar,
            q12_bar, q22_bar, q26_bar,
            q16_bar, q26_bar, q66_bar,
        ))
    }
}

#[derive(Debug)]
pub enum CltError {
    InvalidGeometry(String),
    SingularMatrix,
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–4)

**Day 1: Project initialization and Rust backend scaffold**
```bash
cargo init textilelab-api
cd textilelab-api
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
- `migrations/001_initial.sql` — All 9 tables: users, organizations, org_members, projects, simulations, simulation_jobs, yarn_materials, fabric_templates, usage_records
- `src/db/mod.rs` — Database pool initialization
- `src/db/models.rs` — All SQLx structs with FromRow derives
- Run `sqlx migrate run` to apply schema
- Seed script for yarn materials (100 common fibers: carbon, glass, aramid)

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

### Phase 2 — Solver Core Prototype (Days 5–11)

**Day 5: Geometry primitives and mesh handling**
- `solver-core/` — New Rust workspace member (shared between native and WASM)
- `solver-core/src/geometry/mod.rs` — TriangleMesh struct (vertices, faces, normals)
- `solver-core/src/geometry/stl.rs` — STL file parsing (binary and ASCII formats)
- `solver-core/src/geometry/spatial.rs` — BVH tree for fast point-to-mesh projection
- `solver-core/src/fabric.rs` — FabricMesh struct (quad elements, connectivity)
- Unit tests: STL loading, closest-point-on-mesh queries

**Day 6: Kinematic draping framework**
- `solver-core/src/draping/kinematic.rs` — KinematicDraper struct with basic iteration loop
- `solver-core/src/draping/projection.rs` — Project point onto triangle with barycentric coords
- `solver-core/src/draping/constraints.rs` — Yarn inextensibility constraint solver (Newton-Raphson)
- Tests: flat plane drape (should be identity), hemisphere drape (simple geometry)

**Day 7: Kinematic draping — complete implementation**
- Wavefront node processing: propagate from pinned boundary
- Geodesic path computation on mold surface
- Shear angle calculation at each element
- Fiber orientation calculation relative to mold frame
- Wrinkle zone identification
- Convergence diagnostics and adaptive relaxation
- Tests: cylinder drape (compare to analytical), double-curved surface

**Day 8: CLT homogenization solver**
- `solver-core/src/clt/mod.rs` — CltSolver struct
- `solver-core/src/clt/stiffness.rs` — Reduced stiffness matrix computation and transformation
- `solver-core/src/clt/abd.rs` — ABD matrix assembly from ply stack
- `solver-core/src/clt/inversion.rs` — Engineering constants from compliance matrix
- Rule-of-mixtures for yarn properties
- Tests: plain weave carbon/epoxy (compare to literature values)

**Day 9: Fabric unit cell generation**
- `solver-core/src/fabric/weave.rs` — Generate 3D geometry from 2D weave pattern
- `solver-core/src/fabric/yarn_path.rs` — Sinusoidal yarn path with crimp
- `solver-core/src/fabric/cross_section.rs` — Elliptical yarn cross-section
- Support plain, twill, satin patterns
- Periodic boundary conditions
- Fiber volume fraction calculation
- Tests: plain weave unit cell (verify symmetry, Vf calculation)

**Day 10: Python geometry service**
- `geometry-service/main.py` — FastAPI app
- `geometry-service/stl_processor.py` — STL mesh analysis (surface area, volume, bounding box)
- `geometry-service/remeshing.py` — Adaptive remeshing for uniform element size
- `geometry-service/mesh_quality.py` — Check mesh quality metrics (aspect ratio, angles)
- Docker container for Python service
- REST endpoints: `/process-stl`, `/remesh`, `/quality-check`

**Day 11: Material database and API**
- `src/api/handlers/materials.rs` — Search yarn materials, get material details, upload custom
- Seed 500 built-in yarn materials from public sources (Hexcel, Toray, Owens Corning data sheets)
- Material JSON schema: mechanical properties, geometric properties, datasheet URL
- S3 upload for material definitions
- Parametric search: filter by fiber type, modulus range, density range
- Full-text search on name and manufacturer with pg_trgm

### Phase 3 — WASM Build + Frontend Foundation (Days 12–17)

**Day 12: WASM solver compilation**
- `solver-wasm/` — New workspace member for WASM target
- `solver-wasm/Cargo.toml` — wasm-bindgen, serde-wasm-bindgen dependencies
- `solver-wasm/src/lib.rs` — WASM entry points: `drape_kinematic()`, `homogenize_clt()`
- `wasm-pack build --target web --release` pipeline
- `wasm-opt -Oz` for size optimization (target <1.5MB gzipped)
- JavaScript wrapper: `TextileSolver` class that loads WASM and provides async API

**Day 13: Frontend scaffold and routing**
```bash
npm create vite@latest frontend -- --template react-ts
cd frontend
npm i zustand @tanstack/react-query axios three @react-three/fiber @react-three/drei
```
- `src/App.tsx` — React Router, auth context, layout
- `src/stores/authStore.ts` — Zustand auth state (JWT, user)
- `src/stores/projectStore.ts` — Project state
- `src/lib/api.ts` — Axios API client with JWT interceptor
- `src/pages/Dashboard.tsx` — Project list with cards

**Day 14: Weave pattern editor (SVG 2D)**
- `src/components/WeaveEditor/Canvas.tsx` — SVG canvas with zoom/pan
- `src/components/WeaveEditor/Grid.tsx` — Weave pattern grid (click cells to toggle warp/weft)
- `src/components/WeaveEditor/PatternPreview.tsx` — Live preview of interlacement
- `src/stores/weaveStore.ts` — Weave pattern state (2D array, repeat size)
- Support plain, twill, satin presets
- Custom pattern editing: click to flip warp/weft crossover
- Export pattern as JSON

**Day 15: 3D yarn geometry viewer**
- `src/components/GeometryViewer/Scene.tsx` — Three.js scene with React Three Fiber
- `src/components/GeometryViewer/YarnGeometry.tsx` — Generate yarn mesh from weave pattern
- Cylindrical yarn with sinusoidal path
- Elliptical cross-section deformation at crossovers
- Lighting: directional + ambient for good depth perception
- OrbitControls for navigation

**Day 16: Fabric properties panel**
- `src/components/FabricPanel/MaterialSelector.tsx` — Search and select yarn materials
- `src/components/FabricPanel/GeometryInputs.tsx` — Yarn spacing, diameter, crimp inputs
- `src/components/FabricPanel/PropertiesSummary.tsx` — Display computed areal weight, Vf
- Real-time updates: change spacing → geometry regenerates
- Material search with autocomplete

**Day 17: STL mold upload and visualization**
- `src/components/MoldViewer/STLUploader.tsx` — Drag-and-drop STL upload
- `src/components/MoldViewer/MoldMesh.tsx` — Three.js mesh renderer for mold
- S3 upload progress indicator
- Wireframe and solid rendering modes
- Auto-center and scale mold to fit viewport
- Mold + fabric overlay visualization

### Phase 4 — Draping Solver Integration (Days 18–24)

**Day 18: Draping API endpoints**
- `src/api/handlers/simulation.rs` — Create draping simulation, get results
- Element count estimation from mold surface area
- WASM/server routing logic based on element count
- Plan limit enforcement (Academic: 5K elements, Pro: unlimited)
- Job enqueueing for server-side drapes

**Day 19: Server-side draping worker**
- `src/workers/draping_worker.rs` — Redis job consumer, runs native draping solver
- `src/workers/mod.rs` — Worker pool management (configurable concurrency)
- Progress updates via Redis pub/sub → WebSocket
- Error handling: mesh quality issues, non-convergence
- S3 result upload with presigned URL generation

**Day 20: WebSocket for live draping progress**
- `src/api/ws/mod.rs` — Axum WebSocket upgrade handler
- `src/api/ws/draping_progress.rs` — Subscribe to draping progress channel
- Client receives: `{ progress_pct, iteration, max_displacement, max_shear }`
- Connection management: heartbeat ping/pong, automatic cleanup
- Frontend: `src/hooks/useDrapingProgress.ts` — React hook for WebSocket

**Day 21: Draping results viewer (Three.js)**
- `src/components/DrapeViewer/DrapeViewer.tsx` — Three.js drape visualization
- Color mapping: shear angle (blue→green→red for 0°→45°→90°)
- Fiber orientation visualization (arrows or streamlines)
- Wrinkle zone highlighting (red overlay)
- Interactive mode selector: shear, warp angle, weft angle, wrinkle
- Export drape mesh as OBJ or STL

**Day 22: WASM draping integration**
- `src/hooks/useWasmSolver.ts` — Load WASM solver and run draping
- Progress indication for WASM execution (synthetic progress based on time)
- Error handling for WASM panics
- Result caching in browser IndexedDB
- Comparison view: WASM vs server results side-by-side

**Day 23: Draping parameter controls**
- `src/components/DrapeControls/PinningTool.tsx` — Click mold to place fabric pinning points
- `src/components/DrapeControls/OrientationControl.tsx` — Set initial fabric orientation
- `src/components/DrapeControls/ShearLimit.tsx` — Adjust shear locking angle slider
- `src/components/DrapeControls/AdvancedSettings.tsx` — Convergence tolerance, max iterations
- Preset draping scenarios: hemisphere, cylinder, automotive panel

**Day 24: Results export and reporting**
- `src/api/handlers/export.rs` — Export fiber angle maps (CSV, PNG heatmap)
- Export shear distribution histogram
- Export wrinkle zone coordinates
- Generate PDF report with summary statistics and visualizations
- Batch export for multiple draping scenarios

### Phase 5 — Polish + Testing (Days 25–30)

**Day 25: CLT homogenization UI**
- `src/components/CLT/HomogenizationPanel.tsx` — CLT solver controls (Vf, matrix properties)
- `src/components/CLT/PropertiesDisplay.tsx` — Display Ex, Ey, Gxy, νxy in table
- `src/components/CLT/StiffnessMatrix.tsx` — Visualize A and D matrices as heatmaps
- Compare with analytical models (Chamis, Halpin-Tsai)

**Day 26: Fabric template library**
- `src/api/handlers/templates.rs` — List fabric templates, create from template
- Seed 20 common fabric templates: plain weave, 2×2 twill, 5-harness satin, etc.
- Template gallery with thumbnails
- One-click "Use Template" → creates project with fabric pre-configured

**Day 27: Project collaboration (Team plan)**
- `src/api/handlers/collaboration.rs` — Share project with org members
- Real-time presence indicators (who's viewing project)
- Activity feed: "Alice ran draping simulation", "Bob updated yarn spacing"
- Comment system on projects
- Version history for fabric definitions

**Day 28: Solver validation suite**
- Benchmark 1: Hemisphere drape — compare max shear to literature (Boisse et al.)
- Benchmark 2: Picture frame test — validate shear angle vs displacement curve
- Benchmark 3: Plain weave carbon/epoxy CLT — verify Ex, Ey, Gxy vs WiseTex reference
- Benchmark 4: Double-dome drape — wrinkle prediction validation
- Automated test suite: `solver-core/tests/validation.rs`

**Day 29: Integration testing**
- End-to-end test: create project → define fabric → upload mold → run drape → view results
- API integration tests: auth → project CRUD → simulation → results
- WASM solver test: load in headless browser (Playwright), run drape, verify results
- WebSocket test: connect → subscribe → receive progress → receive completion
- Concurrent draping test: submit 10 simultaneous server-side jobs

**Day 30: Performance testing and optimization**
- WASM solver benchmarks: measure time for 1K, 5K, 10K element drapes in Chrome/Firefox
- Target: 10K element drape < 3 seconds in WASM
- Server solver benchmarks: measure throughput with 4-core, 8-core workers
- Frontend rendering: measure Three.js FPS with 100K triangles
- Memory profiling: ensure no WASM leaks, healthy GC behavior
- Load testing: 50 concurrent users via k6

### Phase 6 — Billing + Deployment (Days 31–36)

**Day 31: Stripe integration**
- `src/api/handlers/billing.rs` — Create checkout session, customer portal
- `src/api/handlers/webhooks/stripe.rs` — Handle subscription events
- Plan mapping:
  - Academic ($49/mo): 5K element drapes, 20 cloud drape minutes/month
  - Pro ($149/mo): Unlimited elements, 200 cloud minutes/month, CLT + draping
  - Enterprise (custom): Multi-user teams, SSO, API access

**Day 32: Usage tracking and plan limits**
- `src/middleware/plan_limits.rs` — Middleware checking plan limits before draping
- `src/services/usage.rs` — Track cloud draping minutes per billing period
- Usage dashboard: current period usage, historical trends
- Approaching-limit warnings at 80% and 100%

**Day 33: Billing UI**
- `frontend/src/pages/Billing.tsx` — Plan comparison, current plan, usage meter
- Upgrade/downgrade flow via Stripe Customer Portal
- Upgrade prompt modals when hitting limits

**Day 34: Docker and Kubernetes**
- `Dockerfile` — Multi-stage Rust build
- `k8s/api-deployment.yaml` — API server (3 replicas, HPA)
- `k8s/worker-deployment.yaml` — Draping workers (auto-scaling)
- `k8s/postgres-statefulset.yaml` — PostgreSQL with PVC
- `k8s/redis-deployment.yaml` — Redis
- `k8s/ingress.yaml` — NGINX ingress with TLS
- Health check endpoints: `/health/live`, `/health/ready`

**Day 35: CDN and WASM delivery**
- CloudFront distribution for frontend + WASM
- WASM preloading: start download on page load
- Service worker for offline WASM caching

**Day 36: Monitoring and launch prep**
- Prometheus metrics: draping duration histogram, API latency, convergence rate
- Grafana dashboards: system health, draping throughput
- Sentry integration: error tracking
- Rate limiting: 100 req/min per user
- Security audit: JWT validation, SQL injection, CORS, CSP
- Documentation: getting started, API reference
- Deploy to production

### Phase 7 — Post-Launch Iteration (Days 37–42)

**Day 37: User feedback collection**
- In-app feedback widget
- Analytics: track feature usage (which weave patterns most common, avg drape complexity)
- Support ticketing system integration

**Day 38: Performance optimization round 2**
- Based on real user data: optimize hot paths
- WASM solver: parallel processing for multi-core browsers
- Database query optimization: add indexes based on slow query log

**Day 39: Educational content**
- Tutorial series: "Introduction to Textile Draping"
- Video walkthroughs for common workflows
- Example projects gallery with open-source datasets

**Day 40: Advanced draping features (post-MVP teasers)**
- Multi-ply draping preview (UI only, backend Phase 2)
- Permeability estimation preview (UI only, backend Phase 2)

**Day 41: API documentation and developer tools**
- OpenAPI spec generation
- Postman collection for API
- SDK stubs for Python, JavaScript (Team plan feature)

**Day 42: Marketing site and launch**
- Landing page with product demo video
- Case studies from beta users
- Blog post: "Why We Built TextileLab"
- Launch on Product Hunt, Hacker News
- Outreach to composites and textile engineering communities

---

## Critical Files

```
textilelab/
├── solver-core/                           # Shared solver library (native + WASM)
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── geometry/
│   │   │   ├── mod.rs                     # Mesh data structures
│   │   │   ├── stl.rs                     # STL parser
│   │   │   ├── spatial.rs                 # BVH for ray-mesh intersection
│   │   │   └── projection.rs              # Point-to-mesh projection
│   │   ├── fabric.rs                      # FabricMesh structure
│   │   ├── draping/
│   │   │   ├── mod.rs
│   │   │   ├── kinematic.rs               # Kinematic draping solver
│   │   │   ├── projection.rs              # Constraint satisfaction
│   │   │   └── shear.rs                   # Shear angle computation
│   │   ├── clt/
│   │   │   ├── mod.rs
│   │   │   ├── homogenization.rs          # CLT solver
│   │   │   ├── stiffness.rs               # Q matrix computation
│   │   │   └── abd.rs                     # ABD matrix assembly
│   │   └── materials.rs                   # Material property models
│   └── tests/
│       ├── validation.rs                  # Benchmark validation tests
│       ├── geometry.rs                    # Geometry tests
│       └── draping.rs                     # Draping solver tests
│
├── solver-wasm/                           # WASM compilation target
│   ├── Cargo.toml
│   └── src/
│       └── lib.rs                         # WASM entry points (wasm-bindgen)
│
├── textilelab-api/                        # Rust API server
│   ├── Cargo.toml
│   ├── src/
│   │   ├── main.rs                        # Axum app setup, router
│   │   ├── config.rs                      # Environment config
│   │   ├── state.rs                       # AppState (PgPool, Redis, S3)
│   │   ├── error.rs                       # ApiError types
│   │   ├── auth/
│   │   │   ├── mod.rs                     # JWT middleware
│   │   │   └── oauth.rs                   # OAuth handlers
│   │   ├── api/
│   │   │   ├── router.rs                  # Route definitions
│   │   │   ├── handlers/
│   │   │   │   ├── auth.rs
│   │   │   │   ├── users.rs
│   │   │   │   ├── projects.rs
│   │   │   │   ├── simulation.rs          # Draping simulation API
│   │   │   │   ├── materials.rs           # Yarn material library
│   │   │   │   ├── templates.rs           # Fabric templates
│   │   │   │   ├── billing.rs
│   │   │   │   ├── usage.rs
│   │   │   │   ├── export.rs              # Results export
│   │   │   │   └── webhooks/
│   │   │   │       └── stripe.rs
│   │   │   └── ws/
│   │   │       ├── mod.rs
│   │   │       └── draping_progress.rs    # WebSocket progress
│   │   ├── middleware/
│   │   │   └── plan_limits.rs
│   │   ├── services/
│   │   │   ├── usage.rs
│   │   │   └── s3.rs
│   │   └── workers/
│   │       ├── mod.rs
│   │       └── draping_worker.rs          # Server-side draping worker
│   ├── migrations/
│   │   └── 001_initial.sql                # Full database schema
│   └── tests/
│       ├── api_integration.rs
│       └── draping_e2e.rs
│
├── frontend/                              # React frontend
│   ├── package.json
│   ├── vite.config.ts
│   ├── src/
│   │   ├── App.tsx
│   │   ├── main.tsx
│   │   ├── stores/
│   │   │   ├── authStore.ts
│   │   │   ├── projectStore.ts
│   │   │   ├── weaveStore.ts              # Weave pattern state
│   │   │   └── drapeStore.ts              # Draping results state
│   │   ├── hooks/
│   │   │   ├── useDrapingProgress.ts      # WebSocket hook
│   │   │   └── useWasmSolver.ts           # WASM solver hook
│   │   ├── lib/
│   │   │   ├── api.ts                     # Axios API client
│   │   │   └── wasmLoader.ts              # WASM bundle loading
│   │   ├── pages/
│   │   │   ├── Dashboard.tsx
│   │   │   ├── Editor.tsx                 # Main editor
│   │   │   ├── Materials.tsx              # Material library
│   │   │   ├── Templates.tsx              # Template gallery
│   │   │   ├── Billing.tsx
│   │   │   ├── Login.tsx
│   │   │   └── Register.tsx
│   │   ├── components/
│   │   │   ├── WeaveEditor/
│   │   │   │   ├── Canvas.tsx             # SVG pattern editor
│   │   │   │   ├── Grid.tsx
│   │   │   │   └── PatternPreview.tsx
│   │   │   ├── GeometryViewer/
│   │   │   │   ├── Scene.tsx              # Three.js scene
│   │   │   │   └── YarnGeometry.tsx       # 3D yarn mesh
│   │   │   ├── DrapeViewer/
│   │   │   │   ├── DrapeViewer.tsx        # 3D drape visualization
│   │   │   │   └── ColorLegend.tsx
│   │   │   ├── DrapeControls/
│   │   │   │   ├── PinningTool.tsx
│   │   │   │   ├── OrientationControl.tsx
│   │   │   │   └── ShearLimit.tsx
│   │   │   ├── FabricPanel/
│   │   │   │   ├── MaterialSelector.tsx
│   │   │   │   ├── GeometryInputs.tsx
│   │   │   │   └── PropertiesSummary.tsx
│   │   │   ├── MoldViewer/
│   │   │   │   ├── STLUploader.tsx
│   │   │   │   └── MoldMesh.tsx
│   │   │   ├── CLT/
│   │   │   │   ├── HomogenizationPanel.tsx
│   │   │   │   └── PropertiesDisplay.tsx
│   │   │   └── billing/
│   │   │       ├── PlanCard.tsx
│   │   │       └── UsageMeter.tsx
│   │   └── data/
│   │       └── templates/                 # Fabric template JSON
│   └── public/
│       └── wasm/                          # WASM solver bundle
│
├── geometry-service/                      # Python FastAPI (STL processing)
│   ├── requirements.txt
│   ├── main.py                            # FastAPI app
│   ├── stl_processor.py                   # STL analysis
│   ├── remeshing.py                       # Mesh adaptation
│   └── Dockerfile
│
├── k8s/                                   # Kubernetes manifests
│   ├── api-deployment.yaml
│   ├── worker-deployment.yaml
│   ├── postgres-statefulset.yaml
│   ├── redis-deployment.yaml
│   ├── geometry-service-deployment.yaml
│   └── ingress.yaml
│
├── docker-compose.yml                     # Local development stack
├── Cargo.toml                             # Workspace root
└── .github/
    └── workflows/
        ├── ci.yml
        ├── wasm-build.yml
        └── deploy.yml
```

---

## Solver Validation Suite

### Benchmark 1: Hemisphere Draping (Kinematic)

**Geometry:** Hemisphere radius R = 100mm, fabric 200mm × 200mm square, pinned at center

**Fabric:** Plain weave, warp/weft spacing = 2mm, shear lock angle = 50°

**Expected:** Maximum shear angle at edge ≈ **45-50°** (from Boisse et al. 2011 literature)

**Expected:** No wrinkle zones (hemisphere is gradual curvature, below locking angle)

**Tolerance:** Max shear angle < 5° deviation from literature

### Benchmark 2: Picture Frame Shear Test (Validation)

**Test:** Fabric clamped in picture frame, diagonal displacement applied

**Fabric:** Plain weave 100mm × 100mm, warp/weft spacing = 2mm

**Expected shear vs displacement:** At displacement d = 20mm → shear angle ≈ **30°**

**Expected:** Linear relationship: γ ≈ arctan(d/L) where L = 100mm

**Tolerance:** Shear angle < 2° deviation from analytical

### Benchmark 3: Plain Weave Carbon/Epoxy CLT Homogenization

**Fabric:** Plain weave, T700 carbon fiber (E = 230 GPa), Vf = 0.60, epoxy matrix (E = 3.5 GPa)

**Yarn:** Warp/weft spacing = 2mm, diameter = 0.3mm

**Expected Ex ≈ Ey:** **55-65 GPa** (balanced plain weave, from WiseTex reference)

**Expected Gxy:** **4-5 GPa** (shear modulus)

**Expected νxy:** **0.04-0.06** (low Poisson's ratio)

**Tolerance:** Moduli < 10% deviation from reference, Poisson's ratio < 0.01 deviation

### Benchmark 4: Cylinder Draping with Orientation Effect

**Geometry:** Cylinder diameter = 100mm, height = 150mm

**Fabric:** Plain weave 200mm × 200mm, initial orientation 0° and 45° to cylinder axis

**Expected:** At 0° orientation → low shear (<10°), at 45° → high shear (>30°)

**Expected:** Fiber angle deviation from initial: 0° case < 5°, 45° case > 20°

**Tolerance:** Shear angle difference between orientations > 15°

### Benchmark 5: Double-Dome Wrinkle Prediction

**Geometry:** Double-dome tool (automotive interior panel geometry), complex curvature

**Fabric:** 2×2 twill carbon, shear lock = 45°

**Expected:** Wrinkle zones in regions where local shear > 45° (typically at sharp curvature transitions)

**Expected:** 2-5 discrete wrinkle zones based on reference simulation (PAM-FORM comparison)

**Tolerance:** Wrinkle zone count within ±2 of reference, location within 10mm

---

## Verification

### Integration Testing Checklist

1. **Auth flow** — Register → login → JWT issued → authorized API calls → token refresh → logout
2. **Project CRUD** — Create → update fabric → save → reload → verify fabric state preserved
3. **Weave editor** — Edit pattern → generate 3D geometry → verify yarn crossovers
4. **Material selection** — Search "carbon" → results → select T700 → properties displayed
5. **WASM draping** — Small mold (2K elements) → WASM solver → results → 3D visualization
6. **Server draping** — Large mold (20K elements) → job queued → WebSocket progress → results in S3
7. **CLT homogenization** — Define fabric → run CLT → verify Ex, Ey, Gxy values reasonable
8. **STL upload** — Upload hemisphere STL → visualize → drape over it → results
9. **Draping parameters** — Change pinning points → re-drape → different result
10. **Results export** — Export shear map as CSV → verify data format
11. **Plan limits** — Academic user → 8K element drape → blocked with upgrade prompt
12. **Billing** — Subscribe to Pro → Stripe checkout → webhook → plan updated → limits lifted
13. **Concurrent draping** — 10 users simultaneously run server drapes → all complete correctly
14. **Wrinkle detection** — Sharp curvature mold → wrinkle zones highlighted in red
15. **Template usage** — Select "Plain Weave Carbon" template → project created → correct properties

### SQL Verification Queries

```sql
-- 1. Draping throughput and success rate
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

-- 4. Cloud draping usage by user (billing period)
SELECT u.email, u.plan,
    SUM(ur.quantity) as total_minutes,
    CASE u.plan
        WHEN 'academic' THEN 20
        WHEN 'pro' THEN 200
        WHEN 'enterprise' THEN 2000
    END as limit_minutes
FROM usage_records ur
JOIN users u ON ur.user_id = u.id
WHERE ur.period_start <= CURRENT_DATE AND ur.period_end >= CURRENT_DATE
    AND ur.record_type = 'drape_minutes'
GROUP BY u.email, u.plan
ORDER BY total_minutes DESC;

-- 5. Popular yarn materials
SELECT ym.name, ym.manufacturer, ym.fiber_type,
    ym.usage_count,
    COUNT(DISTINCT p.id) as projects_using
FROM yarn_materials ym
LEFT JOIN projects p ON p.fabric_data::text ILIKE '%' || ym.id::text || '%'
GROUP BY ym.id
ORDER BY ym.usage_count DESC
LIMIT 20;
```

### Performance Benchmarks

| Operation | Target | Method |
|-----------|--------|--------|
| WASM solver: 1K element drape | <500ms | Browser benchmark (Chrome DevTools) |
| WASM solver: 10K element drape | <3s | Browser benchmark |
| Server solver: 50K element drape | <20s | Server timing, 4 cores |
| Server solver: 200K element drape | <120s | Server timing, 8 cores |
| CLT homogenization | <100ms | WASM or server, both fast |
| Three.js: 50K triangle fabric mesh | 60 FPS | Chrome FPS counter during orbit |
| Three.js: 200K triangle drape result | 30 FPS | Chrome FPS counter |
| Weave editor: 20×20 pattern | 60 FPS | SVG rendering performance |
| API: create simulation | <200ms | p95 latency (Prometheus) |
| API: search materials | <100ms | p95 latency with 1K materials |
| WASM bundle load (cached) | <300ms | Service worker cache hit |
| WASM bundle load (cold) | <2s | CDN delivery, 1.5MB gzipped |
| WebSocket: progress latency | <100ms | Time from worker emit to client |
| STL processing: 100K triangles | <2s | Python service timing |

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
│ (RDS)        │ │ (ElastiCache)│ │ (materials,  │
│ Multi-AZ     │ │ Cluster      │ │  STLs,       │
│              │ │              │ │  results)    │
└──────────────┘ └──────┬───────┘ └──────────────┘
                        │
        ┌───────────────┴────────────────┐
        ▼                                ▼
┌──────────────┐              ┌──────────────────┐
│ Draping      │              │ Geometry Service │
│ Workers      │              │ (Python FastAPI) │
│ Pod ×N (HPA) │              │ Pod ×2           │
│ CPU-intensive│              │                  │
└──────────────┘              └──────────────────┘
```

### Scaling Strategy

- **API servers**: Horizontal Pod Autoscaler (HPA) based on CPU/memory, min 3, max 20
- **Draping workers**: HPA based on Redis queue depth, min 2, max 50
- **PostgreSQL**: RDS Multi-AZ with read replicas for analytics queries
- **Redis**: ElastiCache cluster mode for high availability
- **S3**: Unlimited scalability, CloudFront for global delivery
- **WASM bundle**: Cached at edge (CloudFront), long TTL, versioned URLs

### Estimated Costs (at 1,000 active users)

- **Compute (EKS nodes)**: $500/month (10 × t3.xlarge for API + workers)
- **Database (RDS)**: $200/month (db.t3.large Multi-AZ)
- **Redis (ElastiCache)**: $100/month (cache.t3.medium)
- **S3**: $50/month (100GB materials + results, 1TB transfer)
- **CloudFront**: $80/month (2TB transfer)
- **Total infrastructure**: ~$930/month
- **Target margin**: 70%+ at 200+ paying users ($49-149/mo average)

---

## Post-MVP Roadmap

### v1.1 — FEA-Based Draping (3 months)

**Problem:** Kinematic draping doesn't account for fabric bending stiffness, membrane forces, or contact friction — critical for high-precision composite layup.

**Solution:**
- Implement nonlinear FEA draping with shell elements (4-node quads with bending DOFs)
- Fabric material model: orthotropic elastic + rate-dependent shear with locking
- Contact mechanics: fabric-mold contact with penalty method or augmented Lagrangian
- Friction: Coulomb friction between fabric and mold
- Solver: Newton-Raphson with line search, arc-length continuation for post-buckling
- Expected accuracy: ±2° fiber angle, ±5% strain prediction

**Use cases:** Automotive OEMs (door panels, hoods), aerospace (wing skins), wind energy (blade shells)

**Technical challenges:**
- Contact detection and resolution (computationally expensive)
- Material model calibration from bias extension and picture frame tests
- Convergence for large deformations and contact

**Monetization:** FEA draping as Pro plan feature or usage-based pricing ($5/drape for >50K elements)

### v1.2 — Multi-Ply Draping and Stacking (2 months)

**Problem:** Composite parts use multiple fabric plies stacked at different orientations. Inter-ply friction affects draping.

**Solution:**
- Sequential draping: drape ply 1, use as substrate for ply 2, etc.
- Inter-ply friction model: Coulomb friction between plies
- Stacking sequence optimization: suggest ply orientations to minimize wrinkles
- Fiber volume fraction prediction for multi-ply stacks
- Ply cut pattern generation: flat patterns for each ply

**Use cases:** Thick laminates (8+ plies), tailored fiber placement

**Technical challenges:**
- Computational cost scales with ply count (5-10 plies typical)
- Friction coefficients hard to measure, vary with fabric type

**Monetization:** Multi-ply as Enterprise feature or usage-based

### v1.3 — Permeability Prediction for Composites (2 months)

**Problem:** Resin flow during RTM/VARTM depends on fabric permeability. Engineers need to predict permeability from fabric geometry.

**Solution:**
- Micro-scale CFD simulation of flow through fabric unit cell
- Stokes flow solver (low Reynolds number) with periodic boundary conditions
- Compute in-plane (Kx, Ky) and through-thickness (Kz) permeability
- Integrate with CompForge (spec 13) for full RTM simulation
- Validation against experimental data (Gebart model, Carman-Kozeny)

**Use cases:** Composite process simulation, infusion time estimation

**Technical challenges:**
- CFD solver performance (10K-100K mesh elements per unit cell)
- Dual-scale porosity (inter-yarn + intra-yarn) modeling

**Monetization:** Permeability as Pro add-on ($29/mo)

### v1.4 — Knitted Fabric Support (4 months)

**Problem:** Knitted fabrics (jersey, rib, weft-knit) are increasingly used in 3D knitted preforms (Nike Flyknit, Adidas Primeknit, automotive seat covers).

**Solution:**
- Knit structure editor: define loop topology, stitch type (knit, purl, tuck, miss)
- 3D loop geometry generation with yarn path solver
- Knit-specific material models: highly extensible, anisotropic
- Kinematic draping adapted for knit: loops can extend significantly (ΔL/L up to 100%)
- Machine knitting code generation: export to Stoll, Shima Seiki machine formats

**Use cases:** Fashion (3D knit garments), automotive (seat covers), medical (compression garments)

**Technical challenges:**
- Knit topology is complex (wale, course directions, loop interlocking)
- Draping behavior very different from wovens (high extensibility, low shear resistance)

**Monetization:** Knit module as separate product tier ($199/mo)

### v1.5 — Braided Fabric Support (3 months)

**Problem:** Braided fabrics (2D, 3D, triaxial braids) used for aerospace composites (fuselage reinforcement), automotive (drive shafts), sports equipment (bike frames).

**Solution:**
- Braiding pattern editor: define carrier paths, braid angle, mandrel geometry
- Braiding machine simulation: predict yarn tension, braid angle vs pull speed
- 3D braid geometry generation (yarn interweaving)
- Draping of braided fabrics over complex mandrels
- Manufacturing parameter export: braiding machine code

**Use cases:** Aerospace (composite fuselage frames), automotive (carbon fiber driveshafts)

**Technical challenges:**
- Braiding process simulation is time-dependent (not just final geometry)
- Yarn tension variation affects braid quality

**Monetization:** Braid module as Enterprise feature

### v1.6 — AI-Assisted Draping Optimization (6 months)

**Problem:** Finding optimal fabric orientation, pinning strategy, and ply sequence to minimize wrinkles is trial-and-error.

**Solution:**
- Reinforcement learning agent: learns to suggest fabric orientation and pinning to minimize wrinkles
- Training dataset: 100K+ draping simulations with varied parameters
- Surrogate model: neural network predicts wrinkle zones without full draping simulation (10x faster)
- Multi-objective optimization: minimize wrinkles + minimize material waste + meet fiber angle targets
- Interactive UI: AI suggests next design iteration, user accepts/refines

**Use cases:** Complex parts (automotive A-pillars, aerospace fairings) where manual optimization takes days

**Technical challenges:**
- Generating diverse training data (need simulation at scale)
- Balancing AI suggestions with engineer expertise (hybrid AI-human workflow)

**Monetization:** AI optimization as Enterprise add-on ($99/mo) or pay-per-optimization ($20/run)