# 82. PackSim — Packaging Engineering Simulation Platform

## Implementation Plan

**MVP Scope:** Browser-based packaging designer with drag-and-drop product/box/cushion assembly editor rendered via Three.js, STEP/STL import for product CAD geometry, explicit dynamic finite element solver for drop test simulation compiled to WebAssembly for small models (≤10K elements) and server-side Rust-native execution with CUDA acceleration for large models (>10K elements), foam material library (EPS, PE, PU, EPP densities 1-6 lb/ft³) with viscoelastic constitutive models (Ogden hyperfoam, crushable foam), single flat-face drop test from ISTA/ASTM heights (12", 18", 24", 30", 36", 48") onto rigid/compliant surface, G-force extraction at product center-of-gravity and critical component locations, corrugated box compression strength (BCT) calculator using McKee formula and basic FEA with orthotropic board properties, cushion curve database with static loading calculator (bearing area, static stress, optimal density selection), pallet pattern generator for single-SKU load optimization (GMA 48×40", EUR 1200×800mm pallets) with column/interlocking stacking and cube utilization percentage, automatic mesh generation from STL/STEP with adaptive refinement at impact zones, PostgreSQL storage for projects/materials/simulations, S3 for CAD files and simulation results, PDF report with G-force summary, cushion performance, BCT estimate, and pallet layout visualization, Stripe billing with Free/Pro/Advanced tiers.

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| FEA Solver (Drop Test) | Rust + CUDA | Explicit dynamics with contact, viscoelastic foam, GPU-accelerated |
| WASM Compilation | `wasm-pack` + `wasm-bindgen` | Client-side solver for models ≤10K elements |
| ML/Optimization | Python 3.12 (FastAPI) | Cushion recommendation, pallet optimization heuristics |
| Database | PostgreSQL 16 | Projects, users, simulations, material library, cushion curves |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | CAD files (STEP/STL), simulation results, mesh data |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| 3D Visualization | Three.js + WebGL 2.0 | Product assembly viewer, drop animation, stress contours |
| CAD Import | Python (pythonOCC, Open3D) | STEP/STL parsing, mesh generation, geometry repair |
| Real-time | WebSocket (Axum) | Live simulation progress, mesh generation status |
| Job Queue | Redis 7 + Tokio tasks | Server-side FEA job management, priority queue |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | WASM bundle delivery, static frontend assets |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, error tracking |
| CI/CD | GitHub Actions | WASM build pipeline, Rust tests, Docker image push |

### Key Architecture Decisions

1. **Hybrid client/server FEA solver with 10K element threshold**: Drop test models with ≤10K finite elements (covers simple box+cushion assemblies with coarse mesh) run entirely in the browser via WASM, providing instant feedback for quick iterations. Models >10K elements are submitted to the server with CUDA-accelerated explicit solver for full-resolution analysis. The threshold is configurable per plan tier (Free: 5K, Pro: 10K, Advanced: unlimited server-side).

2. **Explicit dynamics solver in Rust with CUDA rather than wrapping LS-DYNA**: Building a custom explicit FEA solver in Rust gives us full control over material models (viscoelastic foam), contact algorithms, WASM compilation, and GPU acceleration via CUDA kernels. LS-DYNA/Abaqus licensing costs $20K-$50K/seat and requires batch file workflows. Our Rust solver uses central difference time integration with adaptive contact detection via spatial hashing, compiled to both WASM and native+CUDA targets.

3. **Three.js for 3D packaging assembly editor**: Three.js provides GPU-accelerated rendering, camera controls (orbit, pan, zoom), and integrated support for GLTF/STL import. The editor allows drag-and-drop placement of product (imported CAD), cushion blocks (parametric generation), and corrugated box (unfolded die template from FEFCO library). Real-time collision visualization shows product-cushion contact zones before simulation.

4. **Python service for CAD import and mesh generation**: STEP file parsing via pythonOCC, mesh generation via GMSH/Netgen, and geometry repair via Open3D. The Python service runs as a separate FastAPI application that handles the CAD-to-mesh pipeline, outputting tetrahedral or hexahedral meshes in VTK format. This is decoupled from the Rust FEA solver to leverage Python's rich CAD ecosystem (pythonOCC, trimesh, Open3D).

5. **PostgreSQL for material library with cushion curve lookup**: Foam cushion curves (static loading: stress vs. thickness for various densities) are stored as JSONB arrays in PostgreSQL for fast parametric queries. Each material has temperature-dependent properties (E, nu, density, Ogden parameters) stored relationally. The cushion curve calculator runs in WASM for instant feedback: product weight + bearing area → recommended foam density.

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";

-- Users
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email TEXT UNIQUE NOT NULL,
    password_hash TEXT,
    name TEXT NOT NULL,
    avatar_url TEXT,
    auth_provider TEXT DEFAULT 'email',
    auth_provider_id TEXT,
    plan TEXT NOT NULL DEFAULT 'free',
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX users_email_idx ON users(email);
CREATE INDEX users_stripe_idx ON users(stripe_customer_id);

-- Organizations (for Team plan)
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
    role TEXT NOT NULL DEFAULT 'member',
    joined_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(org_id, user_id)
);
CREATE INDEX org_members_org_idx ON org_members(org_id);
CREATE INDEX org_members_user_idx ON org_members(user_id);

-- Projects (packaging design workspace)
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    assembly_data JSONB NOT NULL DEFAULT '{}',
    product_cad_url TEXT,
    box_style TEXT DEFAULT 'RSC',
    cushion_config JSONB DEFAULT '{}',
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
    analysis_type TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending',
    execution_mode TEXT NOT NULL DEFAULT 'wasm',
    parameters JSONB NOT NULL DEFAULT '{}',
    element_count INTEGER NOT NULL DEFAULT 0,
    mesh_url TEXT,
    results_url TEXT,
    results_summary JSONB,
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

-- Server Simulation Jobs
CREATE TABLE simulation_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    simulation_id UUID NOT NULL REFERENCES simulations(id) ON DELETE CASCADE,
    worker_id TEXT,
    cores_allocated INTEGER DEFAULT 4,
    memory_mb INTEGER DEFAULT 8192,
    gpu_allocated BOOLEAN DEFAULT false,
    priority INTEGER DEFAULT 0,
    progress_pct REAL DEFAULT 0.0,
    timestep_data JSONB DEFAULT '[]',
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX jobs_sim_idx ON simulation_jobs(simulation_id);
CREATE INDEX jobs_worker_idx ON simulation_jobs(worker_id);

-- Foam Material Library
CREATE TABLE foam_materials (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    manufacturer TEXT,
    category TEXT NOT NULL,
    density_lb_ft3 REAL NOT NULL,
    youngs_modulus_psi REAL NOT NULL,
    poisson_ratio REAL NOT NULL,
    ogden_mu REAL[],
    ogden_alpha REAL[],
    crushable_k REAL,
    crushable_psi_t REAL,
    cushion_curve JSONB,
    temperature_range TEXT DEFAULT '-20C to 60C',
    recyclable BOOLEAN DEFAULT false,
    bio_based BOOLEAN DEFAULT false,
    cost_per_lb REAL,
    datasheet_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX foam_category_idx ON foam_materials(category);
CREATE INDEX foam_density_idx ON foam_materials(density_lb_ft3);

-- Corrugated Board Library
CREATE TABLE corrugated_boards (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    flute_type TEXT NOT NULL,
    ect_lb_in REAL NOT NULL,
    bct_basis_lb REAL NOT NULL,
    caliper_in REAL NOT NULL,
    flat_crush_psi REAL NOT NULL,
    flexural_stiffness REAL NOT NULL,
    liner_weight_lb_msf REAL NOT NULL,
    medium_weight_lb_msf REAL NOT NULL,
    orthotropic_ex_psi REAL NOT NULL,
    orthotropic_ey_psi REAL NOT NULL,
    orthotropic_ez_psi REAL NOT NULL,
    is_double_wall BOOLEAN DEFAULT false,
    manufacturer TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX board_flute_idx ON corrugated_boards(flute_type);

-- Pallet Patterns (cached optimization results)
CREATE TABLE pallet_patterns (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    pallet_type TEXT NOT NULL,
    box_l_in REAL NOT NULL,
    box_w_in REAL NOT NULL,
    box_h_in REAL NOT NULL,
    box_weight_lb REAL NOT NULL,
    pattern_type TEXT NOT NULL,
    boxes_per_layer INTEGER NOT NULL,
    num_layers INTEGER NOT NULL,
    total_boxes INTEGER NOT NULL,
    cube_utilization_pct REAL NOT NULL,
    overhang_in REAL DEFAULT 0.0,
    pattern_json JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX pallet_dims_idx ON pallet_patterns(pallet_type, box_l_in, box_w_in, box_h_in);

-- Usage Tracking
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,
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
    pub assembly_data: serde_json::Value,
    pub product_cad_url: Option<String>,
    pub box_style: String,
    pub cushion_config: serde_json::Value,
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
    pub element_count: i32,
    pub mesh_url: Option<String>,
    pub results_url: Option<String>,
    pub results_summary: Option<serde_json::Value>,
    pub error_message: Option<String>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub wall_time_ms: Option<i32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct FoamMaterial {
    pub id: Uuid,
    pub name: String,
    pub manufacturer: Option<String>,
    pub category: String,
    pub density_lb_ft3: f32,
    pub youngs_modulus_psi: f32,
    pub poisson_ratio: f32,
    pub ogden_mu: Vec<f32>,
    pub ogden_alpha: Vec<f32>,
    pub crushable_k: Option<f32>,
    pub crushable_psi_t: Option<f32>,
    pub cushion_curve: serde_json::Value,
    pub temperature_range: String,
    pub recyclable: bool,
    pub bio_based: bool,
    pub cost_per_lb: Option<f32>,
    pub datasheet_url: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct CorrugatedBoard {
    pub id: Uuid,
    pub name: String,
    pub flute_type: String,
    pub ect_lb_in: f32,
    pub bct_basis_lb: f32,
    pub caliper_in: f32,
    pub flat_crush_psi: f32,
    pub flexural_stiffness: f32,
    pub liner_weight_lb_msf: f32,
    pub medium_weight_lb_msf: f32,
    pub orthotropic_ex_psi: f32,
    pub orthotropic_ey_psi: f32,
    pub orthotropic_ez_psi: f32,
    pub is_double_wall: bool,
    pub manufacturer: Option<String>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct DropTestParams {
    pub drop_height_in: f64,
    pub surface_type: SurfaceType,
    pub orientation: DropOrientation,
    pub gravity: f64,
    pub contact_stiffness: f64,
    pub contact_damping: f64,
    pub timestep_s: f64,
    pub duration_s: f64,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum SurfaceType {
    Rigid,
    Concrete,
    Wood,
    Foam,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum DropOrientation {
    FlatFace,
    Edge,
    Corner,
    Custom { angle_x: f64, angle_y: f64 },
}

#[derive(Debug, Deserialize, Serialize)]
pub struct CushionCalculatorInput {
    pub product_weight_lb: f64,
    pub bearing_area_in2: f64,
    pub drop_height_in: f64,
    pub fragility_g: f64,
}

#[derive(Debug, Serialize)]
pub struct CushionRecommendation {
    pub material_id: Uuid,
    pub material_name: String,
    pub density_lb_ft3: f32,
    pub thickness_in: f64,
    pub static_stress_psi: f64,
    pub predicted_peak_g: f64,
    pub safety_factor: f64,
}
```

---

## Solver Architecture Deep-Dive

### Governing Equations and Discretization

PackSim's drop test solver implements **explicit time integration** for the equations of motion with **contact dynamics**. For a finite element mesh with `n` nodes and `d=3` degrees of freedom per node, the semi-discrete system is:

```
M ü + C u̇ + F_int(u) = F_ext + F_contact

where:
  M: lumped mass matrix (diagonal, n×d)
  C: damping matrix (Rayleigh damping: C = α M + β K)
  F_int: internal forces from element stresses
  F_ext: external forces (gravity, applied loads)
  F_contact: contact forces from penalty or barrier methods
```

**Explicit central difference** time integration (Leap-Frog scheme):

```
ü_n = M^(-1) [F_ext + F_contact - F_int(u_n) - C u̇_n]
u̇_{n+1/2} = u̇_{n-1/2} + ü_n · Δt
u_{n+1} = u_n + u̇_{n+1/2} · Δt
```

The critical timestep for stability (Courant condition):

```
Δt_crit = min_e [ L_e / c_e ]

where:
  L_e: characteristic element length
  c_e: wave speed = sqrt(E / ρ)  for elastic material
```

For foam materials, c ≈ 50-200 m/s → Δt ≈ 1e-6 to 1e-5 seconds for typical element sizes (1-5 mm).

**Viscoelastic foam material model** (Ogden hyperfoam):

```
Strain energy density:
W = Σ_{i=1}^N (2μ_i/α_i²) (λ₁^α_i + λ₂^α_i + λ₃^α_i - 3) + Σ_{i=1}^N (1/β_i)(J^(-α_i β_i) - 1)

Cauchy stress:
σ = (1/J) ∂W/∂F · F^T

where:
  λ_i: principal stretches
  J: volumetric stretch (J = det(F))
  μ_i, α_i: shear parameters
  β_i: bulk compressibility parameter
```

For typical EPS foam (1.0 lb/ft³): μ₁=5 psi, α₁=4, β₁=0.01 (near-incompressible). PE foam (2.0 lb/ft³): μ₁=15 psi, α₁=3.5, β₁=0.02.

**Contact algorithm** uses penalty method with spatial hashing for broad-phase collision detection:

```
F_contact = k_n δ^(3/2)  (Hertzian contact)

where:
  k_n: contact stiffness (user-specified or auto: ~10× material E)
  δ: penetration depth (node-to-surface distance when negative)
```

Friction: Coulomb model with μ_static = 0.3-0.6 (foam-corrugated), μ_dynamic = 0.2-0.4.

### Client/Server Split (Element Count Threshold)

```
Project uploaded → Mesh generated (Python service)
    │
    ├── ≤10K elements → WASM solver (browser)
    │   ├── Instant startup, no queue wait
    │   ├── Results displayed in <30 seconds
    │   └── No server compute cost
    │
    └── >10K elements → Server solver (Rust + CUDA)
        ├── Job queued via Redis
        ├── Worker with GPU picks up job
        ├── Progress streamed via WebSocket
        └── Results stored in S3, URL returned
```

The 10K element threshold was chosen because:
- WASM solver with explicit integration handles 10K tetrahedral elements (30K DOF) for 0.05s drop in ~20 seconds on modern hardware
- 10K elements covers: simple box + corner cushions (2K + 3K + 5K), basic product geometry (5K box, 3K cushion, 2K product)
- Above 10K: high-resolution meshes for complex products, full ISTA multi-drop sequences, detailed corrugated board ply modeling

### WASM Compilation Pipeline

```toml
# solver-wasm/Cargo.toml
[package]
name = "packsim-solver-wasm"
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
rayon = { version = "1.7", features = ["wasm-bindgen"] }
log = "0.4"
console_log = "1"

[profile.release]
opt-level = "z"
lto = true
codegen-units = 1
strip = true
wasm-opt = true
```

---

## Architecture Deep-Dives

### 1. Drop Test Simulation API Handler (Rust/Axum)

The primary endpoint receives a drop test request, validates the assembly, decides between WASM and server execution, and for server execution enqueues a GPU job with Redis.

```rust
// src/api/handlers/simulation.rs

use axum::{
    extract::{Path, State, Json},
    http::StatusCode,
    response::IntoResponse,
};
use uuid::Uuid;
use crate::{
    db::models::{Simulation, DropTestParams},
    state::AppState,
    auth::Claims,
    error::ApiError,
};

#[derive(serde::Deserialize)]
pub struct CreateDropTestRequest {
    pub drop_height_in: f64,
    pub surface_type: String,
    pub orientation: String,
    pub mesh_resolution: String,
}

pub async fn create_drop_test(
    State(state): State<AppState>,
    claims: Claims,
    Path(project_id): Path<Uuid>,
    Json(req): Json<CreateDropTestRequest>,
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

    // 2. Validate product CAD is uploaded
    if project.product_cad_url.is_none() {
        return Err(ApiError::BadRequest("Product CAD must be uploaded before simulation"));
    }

    // 3. Check plan limits
    let user = sqlx::query_as!(
        crate::db::models::User,
        "SELECT * FROM users WHERE id = $1",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    let max_elements = match user.plan.as_str() {
        "free" => 5_000,
        "pro" => 10_000,
        "advanced" => 100_000,
        _ => 5_000,
    };

    // 4. Trigger mesh generation (async via Python service)
    let mesh_resolution_factor = match req.mesh_resolution.as_str() {
        "coarse" => 1.0,
        "medium" => 0.5,
        "fine" => 0.25,
        _ => 1.0,
    };

    let mesh_req = serde_json::json!({
        "project_id": project_id,
        "product_cad_url": project.product_cad_url,
        "assembly_data": project.assembly_data,
        "resolution_factor": mesh_resolution_factor,
        "max_elements": max_elements,
    });

    let mesh_response: serde_json::Value = state.http_client
        .post(&format!("{}/mesh/generate", state.config.python_service_url))
        .json(&mesh_req)
        .send()
        .await?
        .json()
        .await?;

    let element_count: i32 = mesh_response["element_count"].as_i64().unwrap_or(0) as i32;
    let mesh_url = mesh_response["mesh_url"].as_str().unwrap_or("").to_string();

    if element_count > max_elements as i32 {
        return Err(ApiError::PlanLimit(
            format!("Mesh exceeds plan limit ({} elements). Upgrade or reduce resolution.", element_count)
        ));
    }

    // 5. Determine execution mode
    let execution_mode = if element_count <= 10_000 && user.plan != "free" {
        "wasm"
    } else {
        "server"
    };

    // 6. Create simulation record
    let params = serde_json::json!({
        "drop_height_in": req.drop_height_in,
        "surface_type": req.surface_type,
        "orientation": req.orientation,
        "gravity": 386.4,
        "timestep_s": 1e-6,
        "duration_s": 0.05,
    });

    let sim = sqlx::query_as!(
        Simulation,
        r#"INSERT INTO simulations
            (project_id, user_id, analysis_type, status, execution_mode,
             parameters, element_count, mesh_url)
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8)
        RETURNING *"#,
        project_id,
        claims.user_id,
        "drop_test",
        if execution_mode == "wasm" { "pending" } else { "pending" },
        execution_mode,
        params,
        element_count,
        mesh_url,
    )
    .fetch_one(&state.db)
    .await?;

    // 7. For server execution, enqueue GPU job
    if execution_mode == "server" {
        let job = sqlx::query_as!(
            crate::db::models::SimulationJob,
            r#"INSERT INTO simulation_jobs (simulation_id, priority, gpu_allocated)
            VALUES ($1, $2, $3) RETURNING *"#,
            sim.id,
            if user.plan == "advanced" { 10 } else { 0 },
            true,
        )
        .fetch_one(&state.db)
        .await?;

        state.redis
            .publish("simulation:jobs:gpu", serde_json::to_string(&job.id)?)
            .await?;
    }

    Ok((StatusCode::CREATED, Json(sim)))
}

pub async fn get_simulation_results(
    State(state): State<AppState>,
    claims: Claims,
    Path((project_id, sim_id)): Path<(Uuid, Uuid)>,
) -> Result<Json<serde_json::Value>, ApiError> {
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

    if sim.status != "completed" {
        return Ok(Json(serde_json::json!({
            "status": sim.status,
            "progress_pct": 0.0,
        })));
    }

    // Generate presigned S3 URL for results download
    let results_url = sim.results_url.ok_or(ApiError::NotFound("Results not available"))?;
    let presigned_url = state.s3_client
        .generate_presigned_url(&results_url, 3600)
        .await?;

    Ok(Json(serde_json::json!({
        "status": "completed",
        "results_url": presigned_url,
        "summary": sim.results_summary,
        "wall_time_ms": sim.wall_time_ms,
    })))
}
```

### 2. Explicit FEA Solver Core (Rust — shared between WASM and native)

The core explicit dynamics solver that integrates equations of motion with contact handling. Compiles to both WASM (browser) and native+CUDA (server).

```rust
// solver-core/src/explicit.rs

use nalgebra::{Vector3, Matrix3};
use crate::mesh::{TetrahedralMesh, Node, Element};
use crate::material::{Material, FoamMaterial};

pub struct ExplicitSolver {
    pub mesh: TetrahedralMesh,
    pub materials: Vec<Box<dyn Material>>,
    pub dt: f64,
    pub time: f64,
    pub gravity: Vector3<f64>,

    // State vectors (n_nodes × 3)
    pub positions: Vec<Vector3<f64>>,
    pub velocities: Vec<Vector3<f64>>,
    pub accelerations: Vec<Vector3<f64>>,
    pub forces: Vec<Vector3<f64>>,
    pub masses: Vec<f64>,

    // Contact detection
    pub contact_pairs: Vec<ContactPair>,
    pub spatial_hash: SpatialHash,
}

impl ExplicitSolver {
    pub fn new(mesh: TetrahedralMesh, materials: Vec<Box<dyn Material>>, dt: f64) -> Self {
        let n_nodes = mesh.nodes.len();
        Self {
            positions: mesh.nodes.iter().map(|n| n.position).collect(),
            velocities: vec![Vector3::zeros(); n_nodes],
            accelerations: vec![Vector3::zeros(); n_nodes],
            forces: vec![Vector3::zeros(); n_nodes],
            masses: vec![0.0; n_nodes],
            contact_pairs: Vec::new(),
            spatial_hash: SpatialHash::new(0.01),
            mesh,
            materials,
            dt,
            time: 0.0,
            gravity: Vector3::new(0.0, 0.0, -386.4),
        }
    }

    pub fn initialize(&mut self) {
        // Compute lumped mass matrix
        for elem in &self.mesh.elements {
            let mat = &self.materials[elem.material_id];
            let volume = self.compute_tet_volume(elem);
            let mass_per_node = mat.density() * volume / 4.0;

            for &node_id in &elem.node_ids {
                self.masses[node_id] += mass_per_node;
            }
        }
    }

    pub fn step(&mut self) -> Result<(), SolverError> {
        // 1. Clear forces
        self.forces.iter_mut().for_each(|f| *f = Vector3::zeros());

        // 2. Apply gravity
        for (i, mass) in self.masses.iter().enumerate() {
            self.forces[i] += self.gravity * mass;
        }

        // 3. Compute internal forces from stress
        self.compute_internal_forces()?;

        // 4. Detect and apply contact forces
        self.update_spatial_hash();
        self.detect_contacts();
        self.apply_contact_forces();

        // 5. Explicit time integration (Leap-Frog)
        for i in 0..self.positions.len() {
            if self.mesh.nodes[i].is_fixed {
                continue;
            }

            // a_n = F_n / m
            self.accelerations[i] = self.forces[i] / self.masses[i];

            // v_{n+1/2} = v_{n-1/2} + a_n * dt
            self.velocities[i] += self.accelerations[i] * self.dt;

            // u_{n+1} = u_n + v_{n+1/2} * dt
            self.positions[i] += self.velocities[i] * self.dt;
        }

        self.time += self.dt;
        Ok(())
    }

    fn compute_internal_forces(&mut self) -> Result<(), SolverError> {
        for elem in &self.mesh.elements {
            let mat = &self.materials[elem.material_id];

            // Get element nodal positions
            let x: Vec<Vector3<f64>> = elem.node_ids.iter()
                .map(|&i| self.positions[i])
                .collect();

            // Compute deformation gradient F
            let F = self.compute_deformation_gradient(elem, &x)?;

            // Material model: σ = f(F)
            let stress = mat.compute_stress(&F)?;

            // Compute nodal forces from stress (weak form integration)
            let nodal_forces = self.compute_nodal_forces_from_stress(elem, &stress)?;

            // Accumulate to global force vector
            for (i, &node_id) in elem.node_ids.iter().enumerate() {
                self.forces[node_id] += nodal_forces[i];
            }
        }
        Ok(())
    }

    fn compute_deformation_gradient(
        &self,
        elem: &Element,
        x: &[Vector3<f64>],
    ) -> Result<Matrix3<f64>, SolverError> {
        // Reference configuration (t=0)
        let X: Vec<Vector3<f64>> = elem.node_ids.iter()
            .map(|&i| self.mesh.nodes[i].position)
            .collect();

        // Shape function derivatives (constant for linear tet)
        let dN = compute_tet_shape_derivatives(&X)?;

        // F = ∂x/∂X = Σ x_i ⊗ dN_i
        let mut F = Matrix3::zeros();
        for i in 0..4 {
            F += x[i] * dN[i].transpose();
        }

        Ok(F)
    }

    fn detect_contacts(&mut self) {
        self.contact_pairs.clear();

        // Broad phase: spatial hash
        let candidate_pairs = self.spatial_hash.find_collision_candidates();

        // Narrow phase: node-to-surface distance
        for (node_id, surface_id) in candidate_pairs {
            let dist = self.compute_node_surface_distance(node_id, surface_id);
            if dist < 0.0 {
                self.contact_pairs.push(ContactPair {
                    node_id,
                    surface_id,
                    penetration: -dist,
                    normal: self.compute_surface_normal(surface_id),
                });
            }
        }
    }

    fn apply_contact_forces(&mut self) {
        for contact in &self.contact_pairs {
            // Penalty method: F = k_n * δ^(3/2)  (Hertzian)
            let k_n = 1e6;
            let force_magnitude = k_n * contact.penetration.powf(1.5);
            let force = contact.normal * force_magnitude;

            self.forces[contact.node_id] += force;
        }
    }

    fn compute_tet_volume(&self, elem: &Element) -> f64 {
        let x: Vec<Vector3<f64>> = elem.node_ids.iter()
            .map(|&i| self.mesh.nodes[i].position)
            .collect();

        let v1 = x[1] - x[0];
        let v2 = x[2] - x[0];
        let v3 = x[3] - x[0];

        v1.cross(&v2).dot(&v3).abs() / 6.0
    }

    pub fn extract_max_acceleration(&self, node_ids: &[usize]) -> f64 {
        node_ids.iter()
            .map(|&i| self.accelerations[i].norm())
            .max_by(|a, b| a.partial_cmp(b).unwrap())
            .unwrap_or(0.0)
    }
}

#[derive(Debug)]
pub struct ContactPair {
    pub node_id: usize,
    pub surface_id: usize,
    pub penetration: f64,
    pub normal: Vector3<f64>,
}

#[derive(Debug)]
pub enum SolverError {
    Convergence(String),
    InvalidMesh(String),
    MaterialFailure(String),
}
```

### 3. Three.js Packaging Assembly Editor (React + TypeScript)

The frontend 3D editor for drag-and-drop assembly of product, cushions, and box with real-time collision visualization.

```typescript
// frontend/src/components/AssemblyEditor.tsx

import { useRef, useEffect, useState } from 'react';
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';
import { STLLoader } from 'three/examples/jsm/loaders/STLLoader';
import { useAssemblyStore } from '../stores/assemblyStore';

interface AssemblyComponent {
  id: string;
  type: 'product' | 'cushion' | 'box';
  mesh: THREE.Mesh;
  position: THREE.Vector3;
  rotation: THREE.Euler;
}

export function AssemblyEditor() {
  const containerRef = useRef<HTMLDivElement>(null);
  const sceneRef = useRef<THREE.Scene | null>(null);
  const cameraRef = useRef<THREE.PerspectiveCamera | null>(null);
  const rendererRef = useRef<THREE.WebGLRenderer | null>(null);
  const controlsRef = useRef<OrbitControls | null>(null);

  const { components, addComponent, updateComponent } = useAssemblyStore();
  const [selectedComponent, setSelectedComponent] = useState<string | null>(null);

  useEffect(() => {
    if (!containerRef.current) return;

    // Initialize Three.js scene
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0xf0f0f0);
    sceneRef.current = scene;

    const camera = new THREE.PerspectiveCamera(
      45,
      containerRef.current.clientWidth / containerRef.current.clientHeight,
      0.1,
      1000
    );
    camera.position.set(20, 20, 20);
    cameraRef.current = camera;

    const renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(containerRef.current.clientWidth, containerRef.current.clientHeight);
    renderer.shadowMap.enabled = true;
    renderer.shadowMap.type = THREE.PCFSoftShadowMap;
    containerRef.current.appendChild(renderer.domElement);
    rendererRef.current = renderer;

    // Orbit controls
    const controls = new OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true;
    controls.dampingFactor = 0.05;
    controlsRef.current = controls;

    // Lighting
    const ambientLight = new THREE.AmbientLight(0xffffff, 0.6);
    scene.add(ambientLight);

    const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8);
    directionalLight.position.set(10, 20, 10);
    directionalLight.castShadow = true;
    scene.add(directionalLight);

    // Ground plane
    const groundGeometry = new THREE.PlaneGeometry(50, 50);
    const groundMaterial = new THREE.MeshStandardMaterial({
      color: 0xcccccc,
      side: THREE.DoubleSide,
    });
    const ground = new THREE.Mesh(groundGeometry, groundMaterial);
    ground.rotation.x = -Math.PI / 2;
    ground.receiveShadow = true;
    scene.add(ground);

    // Grid helper
    const gridHelper = new THREE.GridHelper(50, 50);
    scene.add(gridHelper);

    // Animation loop
    const animate = () => {
      requestAnimationFrame(animate);
      controls.update();
      renderer.render(scene, camera);
    };
    animate();

    // Handle window resize
    const handleResize = () => {
      if (!containerRef.current) return;
      const width = containerRef.current.clientWidth;
      const height = containerRef.current.clientHeight;
      camera.aspect = width / height;
      camera.updateProjectionMatrix();
      renderer.setSize(width, height);
    };
    window.addEventListener('resize', handleResize);

    return () => {
      window.removeEventListener('resize', handleResize);
      renderer.dispose();
      controls.dispose();
    };
  }, []);

  const loadProductCAD = async (file: File) => {
    const loader = new STLLoader();
    const geometry = await new Promise<THREE.BufferGeometry>((resolve) => {
      loader.load(URL.createObjectURL(file), resolve);
    });

    geometry.computeVertexNormals();
    const material = new THREE.MeshStandardMaterial({
      color: 0x4080ff,
      metalness: 0.3,
      roughness: 0.6,
    });
    const mesh = new THREE.Mesh(geometry, material);
    mesh.castShadow = true;
    mesh.receiveShadow = true;

    sceneRef.current?.add(mesh);
    addComponent({
      id: `product-${Date.now()}`,
      type: 'product',
      position: { x: 0, y: 5, z: 0 },
      rotation: { x: 0, y: 0, z: 0 },
      mesh: mesh,
    });
  };

  const addCushionBlock = (type: 'corner' | 'edge' | 'full') => {
    let geometry: THREE.BufferGeometry;

    if (type === 'corner') {
      geometry = new THREE.BoxGeometry(3, 3, 3);
    } else if (type === 'edge') {
      geometry = new THREE.BoxGeometry(8, 2, 2);
    } else {
      geometry = new THREE.BoxGeometry(10, 1, 10);
    }

    const material = new THREE.MeshStandardMaterial({
      color: 0xffcc00,
      transparent: true,
      opacity: 0.7,
    });
    const mesh = new THREE.Mesh(geometry, material);
    mesh.castShadow = true;

    sceneRef.current?.add(mesh);
    addComponent({
      id: `cushion-${Date.now()}`,
      type: 'cushion',
      position: { x: 0, y: 1, z: 0 },
      rotation: { x: 0, y: 0, z: 0 },
      mesh: mesh,
    });
  };

  const addBox = (style: 'RSC' | 'HSC' | 'FOL') => {
    const geometry = new THREE.BoxGeometry(12, 10, 12);
    const edges = new THREE.EdgesGeometry(geometry);
    const lineMaterial = new THREE.LineBasicMaterial({ color: 0x8b4513, linewidth: 2 });
    const wireframe = new THREE.LineSegments(edges, lineMaterial);

    const boxMaterial = new THREE.MeshStandardMaterial({
      color: 0xd2b48c,
      transparent: true,
      opacity: 0.3,
      side: THREE.DoubleSide,
    });
    const mesh = new THREE.Mesh(geometry, boxMaterial);
    mesh.add(wireframe);

    sceneRef.current?.add(mesh);
    addComponent({
      id: `box-${Date.now()}`,
      type: 'box',
      position: { x: 0, y: 5, z: 0 },
      rotation: { x: 0, y: 0, z: 0 },
      mesh: mesh,
    });
  };

  return (
    <div className="assembly-editor flex h-full">
      <div className="sidebar w-64 bg-gray-100 p-4 space-y-4">
        <div>
          <h3 className="font-semibold mb-2">Product</h3>
          <input
            type="file"
            accept=".stl,.step,.stp"
            onChange={(e) => e.target.files?.[0] && loadProductCAD(e.target.files[0])}
            className="text-sm"
          />
        </div>

        <div>
          <h3 className="font-semibold mb-2">Cushions</h3>
          <button
            onClick={() => addCushionBlock('corner')}
            className="w-full bg-yellow-500 text-white px-3 py-2 rounded mb-2"
          >
            Add Corner Block
          </button>
          <button
            onClick={() => addCushionBlock('edge')}
            className="w-full bg-yellow-600 text-white px-3 py-2 rounded mb-2"
          >
            Add Edge Block
          </button>
          <button
            onClick={() => addCushionBlock('full')}
            className="w-full bg-yellow-700 text-white px-3 py-2 rounded"
          >
            Add Full Pad
          </button>
        </div>

        <div>
          <h3 className="font-semibold mb-2">Box</h3>
          <button
            onClick={() => addBox('RSC')}
            className="w-full bg-brown-500 text-white px-3 py-2 rounded"
          >
            Add RSC Box
          </button>
        </div>

        {selectedComponent && (
          <div className="mt-4 p-3 bg-white rounded shadow">
            <h3 className="font-semibold mb-2">Properties</h3>
            <label className="block text-sm mb-1">Position X</label>
            <input type="number" className="w-full border px-2 py-1 rounded mb-2" />
            <label className="block text-sm mb-1">Position Y</label>
            <input type="number" className="w-full border px-2 py-1 rounded mb-2" />
            <label className="block text-sm mb-1">Position Z</label>
            <input type="number" className="w-full border px-2 py-1 rounded" />
          </div>
        )}
      </div>

      <div ref={containerRef} className="flex-1" />
    </div>
  );
}
```

### 4. Python CAD Import and Mesh Generation Service (FastAPI)

Separate Python service that handles STEP/STL import, geometry repair, and tetrahedral mesh generation using GMSH.

```python
# python-service/app/mesh_generator.py

from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import pythonOCC.Core.STEPControl as STEPControl
import pythonOCC.Core.TopExp as TopExp
import pythonOCC.Core.TopoDS as TopoDS
import gmsh
import numpy as np
import boto3
import uuid
import json
from typing import List, Dict

app = FastAPI()
s3_client = boto3.client('s3')

class MeshGenerationRequest(BaseModel):
    project_id: str
    product_cad_url: str
    assembly_data: Dict
    resolution_factor: float = 1.0
    max_elements: int = 100000

class MeshGenerationResponse(BaseModel):
    mesh_url: str
    element_count: int
    node_count: int
    volume_in3: float

@app.post("/mesh/generate", response_model=MeshGenerationResponse)
async def generate_mesh(req: MeshGenerationRequest):
    try:
        # 1. Download CAD file from S3
        local_cad_path = f"/tmp/{uuid.uuid4()}.step"
        bucket, key = parse_s3_url(req.product_cad_url)
        s3_client.download_file(bucket, key, local_cad_path)

        # 2. Load STEP file using pythonOCC
        step_reader = STEPControl.STEPControl_Reader()
        status = step_reader.ReadFile(local_cad_path)
        if status != 1:
            raise HTTPException(status_code=400, detail="Failed to read STEP file")

        step_reader.TransferRoot()
        shape = step_reader.Shape()

        # 3. Initialize GMSH
        gmsh.initialize()
        gmsh.model.add("product")

        # Import shape into GMSH (via OpenCASCADE kernel)
        gmsh.model.occ.importShapes(shape)
        gmsh.model.occ.synchronize()

        # 4. Define mesh size based on resolution factor
        characteristic_length = estimate_characteristic_length(shape) * req.resolution_factor
        gmsh.option.setNumber("Mesh.CharacteristicLengthMin", characteristic_length * 0.5)
        gmsh.option.setNumber("Mesh.CharacteristicLengthMax", characteristic_length * 2.0)

        # 5. Generate 3D tetrahedral mesh
        gmsh.model.mesh.generate(3)
        gmsh.model.mesh.optimize("Netgen")

        # 6. Extract mesh data
        node_tags, node_coords, _ = gmsh.model.mesh.getNodes()
        elem_types, elem_tags, elem_node_tags = gmsh.model.mesh.getElements(dim=3)

        # Tetrahedral elements only (type 4)
        tet_indices = [i for i, t in enumerate(elem_types) if t == 4]
        if not tet_indices:
            raise HTTPException(status_code=400, detail="No tetrahedral elements generated")

        tets = elem_node_tags[tet_indices[0]].reshape(-1, 4)
        element_count = len(tets)
        node_count = len(node_tags)

        if element_count > req.max_elements:
            gmsh.finalize()
            raise HTTPException(
                status_code=400,
                detail=f"Mesh too large: {element_count} elements (max {req.max_elements})"
            )

        # 7. Compute volume
        volume_mm3 = compute_mesh_volume(node_coords.reshape(-1, 3), tets)
        volume_in3 = volume_mm3 / 16387.064  # mm³ to in³

        # 8. Export mesh in JSON format
        mesh_data = {
            "nodes": node_coords.tolist(),
            "elements": tets.tolist(),
            "element_count": element_count,
            "node_count": node_count,
        }

        mesh_filename = f"meshes/{req.project_id}/{uuid.uuid4()}.json"
        s3_client.put_object(
            Bucket="packsim-data",
            Key=mesh_filename,
            Body=json.dumps(mesh_data),
            ContentType="application/json"
        )

        mesh_url = f"s3://packsim-data/{mesh_filename}"

        gmsh.finalize()

        return MeshGenerationResponse(
            mesh_url=mesh_url,
            element_count=element_count,
            node_count=node_count,
            volume_in3=volume_in3,
        )

    except Exception as e:
        gmsh.finalize()
        raise HTTPException(status_code=500, detail=str(e))

def estimate_characteristic_length(shape) -> float:
    """Estimate appropriate mesh size from shape bounding box."""
    from OCC.Core.Bnd import Bnd_Box
    from OCC.Core.BRepBndLib import brepbndlib_Add

    bbox = Bnd_Box()
    brepbndlib_Add(shape, bbox)
    xmin, ymin, zmin, xmax, ymax, zmax = bbox.Get()

    diagonal = np.sqrt((xmax - xmin)**2 + (ymax - ymin)**2 + (zmax - zmin)**2)
    return diagonal / 20.0  # ~20 elements along diagonal

def compute_mesh_volume(nodes: np.ndarray, tets: np.ndarray) -> float:
    """Compute total volume of tetrahedral mesh."""
    volume = 0.0
    for tet in tets:
        v0, v1, v2, v3 = nodes[tet - 1]  # GMSH uses 1-indexing
        a = v1 - v0
        b = v2 - v0
        c = v3 - v0
        volume += abs(np.dot(a, np.cross(b, c))) / 6.0
    return volume

def parse_s3_url(s3_url: str) -> tuple:
    """Parse s3://bucket/key to (bucket, key)."""
    parts = s3_url.replace("s3://", "").split("/", 1)
    return parts[0], parts[1]
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–4)

**Day 1: Project initialization and Rust backend scaffold**
```bash
cargo init packsim-api
cd packsim-api
cargo add axum tokio serde serde_json sqlx uuid chrono tower-http jsonwebtoken bcrypt
cargo add --features postgres,runtime-tokio-rustls,uuid,chrono,json sqlx
cargo add aws-sdk-s3 redis reqwest
```
- `src/main.rs` — Axum app with CORS, tracing, graceful shutdown
- `src/config.rs` — Environment-based configuration (DATABASE_URL, JWT_SECRET, S3_BUCKET, REDIS_URL, PYTHON_SERVICE_URL)
- `src/state.rs` — AppState struct (PgPool, Redis, S3Client, HTTP client)
- `src/error.rs` — ApiError enum with IntoResponse
- `Dockerfile` — Multi-stage build (builder + runtime)
- `docker-compose.yml` — PostgreSQL, Redis, MinIO (S3-compatible), Python service

**Day 2: Database schema and migrations**
- `migrations/001_initial.sql` — All 9 tables: users, organizations, org_members, projects, simulations, simulation_jobs, foam_materials, corrugated_boards, pallet_patterns, usage_records
- `src/db/mod.rs` — Database pool initialization
- `src/db/models.rs` — All SQLx structs with FromRow derives
- Run `sqlx migrate run` to apply schema
- Seed script for foam materials (EPS 1.0, 1.5, 2.0 lb/ft³, PE foam 1.7, 2.2, 3.0 lb/ft³) and corrugated boards (C-flute, B-flute, E-flute)

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

### Phase 2 — Solver Core — Explicit FEA Prototype (Days 5–12)

**Day 5: FEA framework and mesh structures**
- `solver-core/` — New Rust workspace member (shared between native and WASM)
- `solver-core/src/mesh.rs` — TetrahedralMesh, Node, Element structs
- `solver-core/src/material.rs` — Material trait, elastic material implementation
- `solver-core/src/explicit.rs` — ExplicitSolver struct with initialization
- Unit tests: single element, mesh volume calculation

**Day 6: Linear elastic material and stress computation**
- `solver-core/src/material/elastic.rs` — Linear elastic material (E, nu)
- Deformation gradient computation from nodal displacements
- Cauchy stress from St. Venant-Kirchhoff model
- Element stiffness matrix assembly
- Tests: uniaxial tension, pure shear

**Day 7: Ogden hyperfoam material model**
- `solver-core/src/material/ogden_foam.rs` — Ogden hyperfoam constitutive model
- Principal stretch computation from deformation gradient
- Strain energy density and stress derivation
- Material parameter fitting from foam compression data
- Tests: uniaxial compression (compare to analytical), volumetric compression

**Day 8: Explicit time integration**
- `solver-core/src/explicit.rs` — Central difference time integration implementation
- Lumped mass matrix assembly
- Critical timestep calculation (Courant condition)
- Velocity Verlet scheme
- Tests: free fall under gravity, elastic wave propagation

**Day 9: Contact detection and penalty method**
- `solver-core/src/contact.rs` — SpatialHash for broad-phase collision detection
- Node-to-surface distance calculation
- Penalty contact force application
- Hertzian contact law
- Tests: sphere drop on rigid plane, two-sphere collision

**Day 10: Drop test assembly and boundary conditions**
- `solver-core/src/assembly.rs` — Combine product, cushion, box meshes
- Initial velocity from drop height
- Fixed surface boundary condition
- Material assignment per mesh region
- Tests: simple box drop, multi-material assembly

**Day 11: Result extraction and post-processing**
- `solver-core/src/results.rs` — Acceleration history extraction
- Peak G-force calculation at specified node sets
- Stress and deformation field export
- Time series data serialization to binary format
- Tests: verify G-force extraction, energy conservation

**Day 12: Solver validation benchmarks**
- Benchmark: elastic cube drop — compare to analytical solution
- Benchmark: foam compression — verify stress-strain curve matches material model
- Benchmark: two-body contact — verify momentum conservation
- Automated test suite with tolerance checks

### Phase 3 — WASM Build + Python Mesh Service (Days 13–18)

**Day 13: WASM solver compilation**
- `solver-wasm/` — New workspace member for WASM target
- `solver-wasm/Cargo.toml` — wasm-bindgen, serde-wasm-bindgen dependencies
- `solver-wasm/src/lib.rs` — WASM entry points: `run_drop_test()`, `get_results()`
- `wasm-pack build --target web --release` pipeline
- `wasm-opt -Oz` for size optimization (target <3MB gzipped)
- JavaScript wrapper: `DropTestSolver` class with async API

**Day 14: Python service setup and STEP import**
- `python-service/` — FastAPI application
- `python-service/requirements.txt` — pythonOCC, gmsh, boto3, numpy
- `python-service/app/main.py` — FastAPI app setup
- `python-service/app/step_import.py` — STEP file loading via pythonOCC
- STEP geometry validation and repair
- Bounding box extraction
- Tests: load sample STEP files, verify geometry extraction

**Day 15: Tetrahedral mesh generation**
- `python-service/app/mesh_generator.py` — GMSH integration for tet mesh generation
- Adaptive mesh sizing from bounding box
- Element quality checks (aspect ratio, volume)
- Mesh coarsening/refinement based on resolution parameter
- Export mesh in JSON format (nodes, elements)
- Tests: generate mesh, verify element count, check element quality

**Day 16: Mesh generation API**
- `/mesh/generate` endpoint: accepts STEP URL, resolution, max elements
- S3 upload of generated mesh JSON
- WebSocket progress updates during mesh generation
- Error handling: invalid geometry, mesh too large
- Integration test: full mesh generation pipeline

**Day 17: Python service deployment**
- `python-service/Dockerfile` — Python environment with pythonOCC
- Docker Compose integration with main stack
- Health check endpoint
- Logging and error reporting
- Performance test: mesh generation time vs. complexity

**Day 18: CAD upload and storage**
- `src/api/handlers/cad.rs` — CAD file upload endpoint
- S3 upload with presigned URL generation
- File type validation (STEP, STL)
- Project CAD URL update
- Frontend: CAD file upload component

### Phase 4 — Frontend 3D Editor (Days 19–24)

**Day 19: Frontend scaffold and Three.js setup**
```bash
npm create vite@latest frontend -- --template react-ts
cd frontend
npm i zustand @tanstack/react-query axios three @types/three
```
- `src/App.tsx` — Router, auth context, layout
- `src/stores/assemblyStore.ts` — Zustand store for assembly state
- `src/components/AssemblyEditor/Canvas.tsx` — Three.js scene initialization
- Three.js orbit controls, lighting, ground plane

**Day 20: Product CAD loading and visualization**
- `src/components/AssemblyEditor/ProductLoader.tsx` — STL/STEP loader
- STLLoader integration
- Mesh material and shading
- Camera auto-fit to loaded geometry
- Tests: load sample STL, verify rendering

**Day 21: Cushion and box parametric generation**
- `src/components/AssemblyEditor/CushionCreator.tsx` — Parametric cushion geometry
- Box geometry with FEFCO style selection
- Drag-and-drop placement in scene
- Transform controls (move, rotate, scale)
- Component property panel

**Day 22: Assembly interaction and editing**
- Component selection and highlighting
- Multi-component selection
- Delete, duplicate, copy/paste
- Snap-to-grid positioning
- Undo/redo stack

**Day 23: Collision visualization**
- Real-time bounding box intersection detection
- Highlight overlapping components
- Clearance visualization
- Contact area preview

**Day 24: Assembly save and load**
- Serialize assembly to JSON (component positions, rotations, materials)
- Save to project in database
- Load assembly from project
- Auto-save on changes (debounced)

### Phase 5 — Simulation Execution + Results (Days 25–29)

**Day 25: Simulation API endpoints**
- `src/api/handlers/simulation.rs` — Create drop test, get results, cancel simulation
- Mesh generation trigger via Python service
- Element count validation and plan limit checks
- WASM vs server execution decision

**Day 26: Server-side simulation worker**
- `src/workers/simulation_worker.rs` — Redis job consumer, runs native solver
- Load mesh from S3
- Execute drop test simulation
- Progress tracking and database updates
- Result upload to S3

**Day 27: WebSocket for live simulation progress**
- `src/api/ws/mod.rs` — Axum WebSocket upgrade handler
- `src/api/ws/simulation_progress.rs` — Subscribe to simulation progress channel
- Progress broadcast from worker via Redis pub/sub
- Frontend: `src/hooks/useSimulationProgress.ts` — React hook for WebSocket subscription

**Day 28: WASM solver frontend integration**
- `src/hooks/useWasmSolver.ts` — Load WASM module, run simulation
- Mesh download from S3
- Run simulation in browser with progress updates
- Extract results from WASM
- Display G-force vs. time chart

**Day 29: Results visualization**
- `src/components/ResultsViewer/GForceChart.tsx` — Plot G-force time history
- `src/components/ResultsViewer/AnimationPlayer.tsx` — Play back drop animation
- Stress contour visualization on Three.js mesh
- Peak G-force summary card
- Pass/fail indicator based on fragility threshold

### Phase 6 — Cushion Calculator + BCT + Pallet (Days 30–33)

**Day 30: Cushion curve database**
- Seed foam_materials table with cushion curve data (10-20 materials)
- `src/api/handlers/materials.rs` — Search foams, get cushion curve
- Frontend: `src/components/CushionCalculator.tsx` — Input product weight, bearing area, drop height
- WASM cushion curve lookup and recommendation

**Day 31: McKee BCT calculator**
- `src/calculators/bct.rs` — McKee formula implementation
- Input: ECT, box dimensions, perimeter, caliper
- Moisture and creep derating factors
- `src/api/handlers/bct.rs` — BCT calculation endpoint
- Frontend: BCT calculator UI

**Day 32: Pallet pattern generator**
- `python-service/app/pallet_optimizer.py` — 2D bin packing algorithm
- Column and interlocking pattern generation
- Cube utilization calculation
- Overhang validation
- `/pallet/optimize` endpoint
- Frontend: Pallet pattern 2D visualization

**Day 33: Pallet pattern 3D visualization**
- Three.js 3D pallet layout rendering
- Animate box placement layer by layer
- Export pallet pattern as JSON
- Save pattern to database for reuse

### Phase 7 — Billing + Plan Enforcement (Days 34–37)

**Day 34: Stripe integration**
- `src/api/handlers/billing.rs` — Create checkout session, create customer portal session
- `src/api/handlers/webhooks/stripe.rs` — Webhook handler (subscription events)
- Plan mapping: Free (5K elements), Pro ($149/mo, 10K elements), Advanced ($329/mo, unlimited)

**Day 35: Usage tracking and plan limits**
- `src/middleware/plan_limits.rs` — Middleware checking plan limits before simulation
- `src/services/usage.rs` — Track simulation minutes per billing period
- Usage record insertion after server-side simulation
- Usage dashboard endpoint

**Day 36: Billing UI**
- `frontend/src/pages/Billing.tsx` — Plan comparison, current plan, usage meter
- Upgrade/downgrade flow via Stripe Customer Portal
- Upgrade prompt modals when hitting limits

**Day 37: Feature gating**
- Gate full-resolution server simulations behind Pro/Advanced
- Gate PDF report export behind Pro
- Gate API access behind Advanced
- Show locked feature indicators with upgrade CTAs

### Phase 8 — Polish + Deployment + Launch (Days 38–42)

**Day 38: Solver validation and testing**
- End-to-end test: upload CAD → generate mesh → run WASM simulation → view results
- Integration test: server-side simulation with WebSocket progress
- Validation: compare WASM vs server results for same mesh
- Performance benchmarks: WASM (5K, 10K elements), server (50K, 100K elements)

**Day 39: PDF report generation**
- `python-service/app/report_generator.py` — Generate PDF with matplotlib/reportlab
- Include: G-force chart, peak summary, BCT calculation, pallet layout
- S3 upload and presigned URL
- `/reports/generate` endpoint
- Frontend: download PDF button

**Day 40: Docker and Kubernetes configuration**
- `Dockerfile` — Multi-stage Rust build
- `docker-compose.yml` — Full local dev stack
- `k8s/` — Kubernetes manifests (API, worker, Python service, PostgreSQL, Redis)
- Health check endpoints
- HPA for API and worker pods

**Day 41: Monitoring, logging, and polish**
- Prometheus metrics: simulation duration, mesh generation time, API latency
- Grafana dashboards
- Sentry integration
- UI polish: loading states, error messages, responsive layout

**Day 42: Launch preparation and final testing**
- End-to-end smoke test in production
- Database backup verification
- Rate limiting configuration
- Security audit
- Landing page and documentation
- Deploy to production

---

## Critical Files

```
packsim/
├── solver-core/                           # Shared FEA solver library (native + WASM)
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── mesh.rs                        # TetrahedralMesh, Node, Element
│   │   ├── explicit.rs                    # Explicit dynamics solver
│   │   ├── assembly.rs                    # Multi-material assembly
│   │   ├── contact.rs                     # Contact detection, penalty forces
│   │   ├── results.rs                     # G-force extraction, serialization
│   │   └── material/
│   │       ├── mod.rs                     # Material trait
│   │       ├── elastic.rs                 # Linear elastic
│   │       ├── ogden_foam.rs              # Ogden hyperfoam
│   │       └── corrugated.rs              # Orthotropic corrugated board
│   └── tests/
│       ├── benchmarks.rs                  # Validation benchmarks
│       └── contact_tests.rs               # Contact algorithm tests
│
├── solver-wasm/                           # WASM compilation target
│   ├── Cargo.toml
│   └── src/
│       └── lib.rs                         # WASM entry points
│
├── packsim-api/                           # Rust API server
│   ├── Cargo.toml
│   ├── src/
│   │   ├── main.rs                        # Axum app setup
│   │   ├── config.rs                      # Environment config
│   │   ├── state.rs                       # AppState
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
│   │   │   │   ├── simulation.rs          # Drop test simulation
│   │   │   │   ├── cad.rs                 # CAD upload
│   │   │   │   ├── materials.rs           # Foam/board library
│   │   │   │   ├── bct.rs                 # BCT calculator
│   │   │   │   ├── billing.rs
│   │   │   │   └── webhooks/
│   │   │   │       └── stripe.rs
│   │   │   └── ws/
│   │   │       ├── mod.rs
│   │   │       └── simulation_progress.rs
│   │   ├── middleware/
│   │   │   └── plan_limits.rs
│   │   ├── services/
│   │   │   ├── usage.rs
│   │   │   └── s3.rs
│   │   ├── calculators/
│   │   │   ├── bct.rs                     # McKee formula
│   │   │   └── cushion.rs                 # Cushion curve lookup
│   │   └── workers/
│   │       ├── mod.rs
│   │       └── simulation_worker.rs       # Server-side FEA execution
│   ├── migrations/
│   │   └── 001_initial.sql
│   └── tests/
│       ├── api_integration.rs
│       └── simulation_e2e.rs
│
├── python-service/                        # Python FastAPI (mesh + reports)
│   ├── requirements.txt
│   ├── app/
│   │   ├── main.py                        # FastAPI app
│   │   ├── step_import.py                 # pythonOCC STEP loader
│   │   ├── mesh_generator.py              # GMSH tet mesh generation
│   │   ├── pallet_optimizer.py            # 2D bin packing
│   │   └── report_generator.py            # PDF report with charts
│   └── Dockerfile
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
│   │   │   └── assemblyStore.ts
│   │   ├── hooks/
│   │   │   ├── useSimulationProgress.ts
│   │   │   └── useWasmSolver.ts
│   │   ├── lib/
│   │   │   ├── api.ts
│   │   │   └── wasmLoader.ts
│   │   ├── pages/
│   │   │   ├── Dashboard.tsx
│   │   │   ├── Editor.tsx                 # Assembly editor + results
│   │   │   ├── Materials.tsx              # Foam/board library browser
│   │   │   ├── Billing.tsx
│   │   │   ├── Login.tsx
│   │   │   └── Register.tsx
│   │   └── components/
│   │       ├── AssemblyEditor/
│   │       │   ├── Canvas.tsx             # Three.js scene
│   │       │   ├── ProductLoader.tsx      # STL/STEP import
│   │       │   ├── CushionCreator.tsx     # Parametric cushions
│   │       │   ├── BoxCreator.tsx         # Box geometry
│   │       │   ├── TransformControls.tsx  # Move/rotate/scale
│   │       │   └── PropertyPanel.tsx
│   │       ├── ResultsViewer/
│   │       │   ├── GForceChart.tsx        # Time history chart
│   │       │   ├── AnimationPlayer.tsx    # Drop animation playback
│   │       │   ├── StressContour.tsx      # 3D stress visualization
│   │       │   └── SummaryCard.tsx
│   │       ├── CushionCalculator.tsx      # Static loading calculator
│   │       ├── BctCalculator.tsx          # McKee BCT
│   │       ├── PalletOptimizer.tsx        # Pallet pattern generator
│   │       └── billing/
│   │           ├── PlanCard.tsx
│   │           └── UsageMeter.tsx
│   └── public/
│       └── wasm/                          # WASM solver bundle
│
├── k8s/                                   # Kubernetes manifests
│   ├── api-deployment.yaml
│   ├── worker-deployment.yaml
│   ├── python-service-deployment.yaml
│   ├── postgres-statefulset.yaml
│   ├── redis-deployment.yaml
│   └── ingress.yaml
│
├── docker-compose.yml
├── Cargo.toml                             # Workspace root
└── .github/
    └── workflows/
        ├── ci.yml
        ├── wasm-build.yml
        └── deploy.yml
```

---

## Solver Validation Suite

### Benchmark 1: Elastic Cube Drop (Sanity Check)

**Setup:** Rigid 2" cube (E=10,000 psi, nu=0.3, rho=0.01 lb/in³) dropped from 12" onto rigid surface

**Expected peak G:** G = sqrt(2 × g × h) / v_impact ≈ **19.2 G** (assuming instant deceleration)

**Tolerance:** < 5% (explicit integration error)

### Benchmark 2: EPS Foam Compression (Material Model Validation)

**Setup:** EPS foam block (1.0 lb/ft³, Ogden: μ₁=5 psi, α₁=4) compressed to 50% strain

**Expected stress at 50% strain:** σ ≈ **4.8 psi** (from Ogden model)

**Tolerance:** < 1% (constitutive model exact for quasi-static)

### Benchmark 3: Drop Test with PE Foam Cushion (Full System)

**Setup:** 5 lb product (6"×4"×2") with 2" PE foam cushion (2.2 lb/ft³) dropped from 30" onto rigid surface

**Expected peak G:** **45-55 G** (based on cushion curve data for PE 2.2 lb/ft³ at 30" drop)

**Tolerance:** < 10% (contact stiffness and foam model uncertainty)

### Benchmark 4: Energy Conservation (Physics Check)

**Setup:** 1 lb object dropped from 24" in vacuum (no damping, no contact)

**Expected energy:** E_initial = m × g × h = **24 in·lbf**, E_final (kinetic at impact) = **24 in·lbf**

**Tolerance:** Energy drift < 0.1% over simulation

### Benchmark 5: Contact Momentum Conservation (Two-Body Collision)

**Setup:** Two identical spheres (1 lb each), one stationary, one moving at 100 in/s, elastic collision

**Expected:** v₁_after = 0 in/s, v₂_after = 100 in/s (perfect momentum transfer)

**Tolerance:** < 2% (penalty contact approximation error)

---

## Verification

### Integration Testing Checklist

1. **Auth flow** — Register → login → JWT issued → authorized API calls → token refresh
2. **Project CRUD** — Create → update assembly → save → reload → verify assembly preserved
3. **CAD upload** — Upload STEP file → stored in S3 → project updated with URL
4. **Mesh generation** — Trigger mesh generation → Python service called → mesh stored in S3
5. **WASM simulation** — Small model (5K elements) → WASM solver → G-force results displayed
6. **Server simulation** — Large model (50K elements) → job queued → worker executes → WebSocket progress → results in S3
7. **Results visualization** — Load results → G-force chart → animation playback → stress contours
8. **Cushion calculator** — Input product weight, area → foam recommendation → verify density selection
9. **BCT calculator** — Input box dimensions, ECT → BCT estimate → verify McKee formula
10. **Pallet optimizer** — Input box dims → pattern generated → verify cube utilization
11. **Plan limits** — Free user → 8K element model → blocked with upgrade prompt
12. **Billing** — Subscribe to Pro → Stripe checkout → webhook → plan updated → limits lifted
13. **Concurrent simulations** — 5 users run server sims simultaneously → all complete
14. **PDF report** — Generate report → includes G-force, BCT, pallet → download PDF
15. **Error handling** — Invalid mesh → meaningful error → no crash

### SQL Verification Queries

```sql
-- 1. Simulation success rate and performance
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

-- 3. User plan distribution
SELECT plan, COUNT(*) as users,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 1) as pct
FROM users GROUP BY plan ORDER BY users DESC;

-- 4. Foam material usage
SELECT fm.name, fm.category, fm.density_lb_ft3,
    COUNT(DISTINCT s.project_id) as projects_using
FROM foam_materials fm
LEFT JOIN simulations s ON s.parameters::text ILIKE '%' || fm.id::text || '%'
GROUP BY fm.id
ORDER BY projects_using DESC
LIMIT 10;

-- 5. Average mesh element count by plan
SELECT u.plan,
    AVG(s.element_count)::int as avg_elements,
    MAX(s.element_count) as max_elements
FROM simulations s
JOIN users u ON s.user_id = u.id
GROUP BY u.plan;
```

### Performance Benchmarks

| Operation | Target | Method |
|-----------|--------|--------|
| WASM solver: 5K elements, 0.05s drop | <10s | Browser benchmark (Chrome DevTools) |
| WASM solver: 10K elements, 0.05s drop | <25s | Browser benchmark |
| Server solver: 50K elements, 0.05s drop | <60s | Server timing, GPU acceleration |
| Server solver: 100K elements, 0.05s drop | <180s | Server timing, GPU acceleration |
| Mesh generation: 10K elements | <15s | Python service timing |
| Mesh generation: 50K elements | <60s | Python service timing |
| Assembly editor: 50 components | 60 FPS | Chrome FPS counter |
| Results animation: 5K nodes, 5000 frames | 30 FPS | Three.js rendering performance |
| API: create simulation | <200ms | p95 latency (Prometheus) |
| API: search materials | <100ms | p95 latency with 100 materials in DB |
| WASM bundle load (cached) | <500ms | Service worker cache hit |
| WASM bundle load (cold) | <4s | CDN delivery, 3MB gzipped |
| WebSocket: progress latency | <150ms | Time from worker emit to client receive |

---

## Deployment Architecture

### Infrastructure

```
┌─────────────────────────────────────────────────┐
│                  CloudFront CDN                   │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────┐ │
│  │ Frontend SPA │  │ WASM Bundle │  │ CAD      │ │
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
│ (RDS r6g)    │ │ (ElastiCache)│ │ (CAD + Mesh  │
│ Multi-AZ     │ │ Cluster mode │ │  + Results)  │
└──────────────┘ └──────┬───────┘ └──────────────┘
                        │
        ┌───────────────┼───────────────┬──────────────┐
        ▼               ▼               ▼              ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ FEA Worker   │ │ FEA Worker   │ │ Python Mesh  │ │ Python Mesh  │
│ (Rust+CUDA)  │ │ (Rust+CUDA)  │ │ Service      │ │ Service      │
│ g5.xlarge GPU│ │ g5.xlarge GPU│ │ c7g.2xlarge  │ │ c7g.2xlarge  │
│ HPA: 1-10    │ │              │ │ HPA: 1-5     │ │              │
└──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘
```

### Scaling Strategy

- **API servers**: HPA at 70% CPU, min 3, max 10
- **FEA workers**: HPA based on Redis GPU queue depth (scale up when > 3 jobs, scale down when idle 5 min)
- **Python mesh service**: HPA at 80% CPU, min 1, max 5
- **Worker instances**: AWS g5.xlarge (NVIDIA A10G GPU) for FEA, c7g.2xlarge for mesh generation
- **Database**: RDS PostgreSQL r6g.large with read replica for material queries
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
      "ID": "mesh-files-lifecycle",
      "Prefix": "meshes/",
      "Status": "Enabled",
      "Transitions": [
        { "Days": 7, "StorageClass": "STANDARD_IA" }
      ],
      "Expiration": { "Days": 90 }
    },
    {
      "ID": "cad-files-keep",
      "Prefix": "cad/",
      "Status": "Enabled",
      "Transitions": [
        { "Days": 90, "StorageClass": "STANDARD_IA" }
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
| Multi-Drop ISTA Sequences | Simulate full ISTA 3A/3E protocols with sequential drops from multiple orientations, tracking cumulative damage and cushion permanent set across 10-20 drops | High |
| Transport Vibration Simulation | Random vibration using ISTA/ASTM D4169 PSD profiles with frequency response analysis, resonance detection, and fatigue damage accumulation via Miner's rule | High |
| Thermal Packaging Simulation | Transient heat transfer FEA for EPS/PU insulated shippers with PCM (gel packs, dry ice) modeling and payload temperature prediction for cold chain qualification | High |
| Corrugated Board FEA with Moisture/Creep | Full orthotropic shell element FEA for box compression with Kellicutt-Landt moisture derating and power-law creep modeling for warehouse stacking duration | Medium |
| AI-Based Cushion Recommendation | ML model (PyTorch) trained on simulation database to recommend optimal foam type, density, and thickness given product weight, fragility, and drop height | Medium |
| Mixed-SKU Pallet Optimization | 3D bin packing with constraint satisfaction for mixed SKU loads, considering stacking strength compatibility and load stability across multiple box sizes | Medium |
| CUDA Acceleration for Large Models | GPU kernels for element force computation and contact detection to handle 100K-500K element models with 10× speedup over CPU-only solver | Medium |
| Real-Time Collaboration | CRDT-based assembly editing with cursor presence, commenting on components, and shared simulation results for team packaging design reviews | Low |
| PLM/ERP Integration | SAP and Oracle connector for SKU data sync, bill of materials import, and packaging specification export to enterprise systems | Low |
