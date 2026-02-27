# 98. FoodProcess — Food Processing Engineering Platform

## Implementation Plan

**MVP Scope:** Browser-based thermal processing simulation platform for 2D axisymmetric transient heat conduction in canned foods (cylinder, sphere, brick geometries) with retort cycle simulation (come-up time, hold time, cooling), custom Rust-based finite difference solver compiled to WebAssembly for client-side execution of small geometries (≤10,000 mesh points) and server-side Rust-native execution for larger/3D models, food-specific thermophysical properties database (thermal conductivity, specific heat, density as functions of temperature and composition) covering 100+ common foods using Choi-Okos correlations, microbial inactivation kinetics implementing D-value/z-value first-order models for 10 priority pathogens (C. botulinum, L. monocytogenes, Salmonella, E. coli O157:H7, etc.), F0 lethality and pasteurization unit (PU) calculation with cold spot identification at slowest heating point, process deviation scenario analysis (what-if temperature drops, extended come-up, equipment failure), interactive temperature profile visualization rendered via Plotly.js with 3D thermal field view using Three.js/WebGL, automated PDF process validation reports for regulatory submission, PostgreSQL storage for projects/simulations/material properties with S3 for simulation results, Stripe billing with three tiers (Free / Pro $149/mo / Team $349/user/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| Thermal Solver | Rust (native + WASM) | Custom FDM/FEM transient heat conduction with phase change (Stefan problem) |
| WASM Compilation | `wasm-pack` + `wasm-bindgen` | Client-side solver for meshes ≤10K points |
| Food Properties Service | Python 3.12 (FastAPI) | Choi-Okos correlations, property database, composition-based prediction |
| Kinetics Engine | Rust (native + WASM) | Microbial inactivation (D/z-value), lethality integration (F0/PU calculation) |
| Database | PostgreSQL 16 | Projects, users, simulations, food properties, microbial parameters |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | Simulation results (temperature fields, lethality maps), validation reports |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| Temperature Visualization | Plotly.js + Three.js | 2D plots (temperature vs time), 3D thermal field rendering (WebGL) |
| 3D Thermal View | Three.js (WebGL) | GPU-accelerated isosurface/heatmap rendering for 3D temperature fields |
| Real-time | WebSocket (Axum) | Live simulation progress, convergence monitoring |
| Job Queue | Redis 7 + Tokio tasks | Server-side simulation job management, batch processing |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | WASM bundle delivery, static frontend assets |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, error tracking |
| CI/CD | GitHub Actions | WASM build pipeline, Rust tests, Docker image push |

### Key Architecture Decisions

1. **Hybrid client/server solver with WASM threshold at 10K mesh points**: Small/medium thermal problems (2D axisymmetric with ≤10K mesh points, covering 90%+ of canned food simulations) run entirely in the browser via WASM, providing instant feedback with zero server cost. Large 3D problems or batch process optimization runs are submitted to the Rust-native server solver. The threshold is configurable per plan tier (free tier limited to 2K points).

2. **Custom thermal solver in Rust rather than wrapping OpenFOAM/FEniCS**: Building a food-specific thermal solver in Rust gives us full control over convergence algorithms, WASM compilation, and food-specific boundary conditions (natural convection correlations for in-container products, steam condensation for retorts). OpenFOAM's C++ codebase is too heavyweight for WASM and lacks food-specific material models. Our Rust solver uses sparse matrix libraries (nalgebra-sparse, sprs) for implicit time stepping while maintaining memory safety and WASM compatibility.

3. **Finite difference on structured grids for axisymmetric geometries**: For MVP (cylinders, spheres), we use finite difference on regular cylindrical/spherical coordinate grids which are simple to implement, fast to solve, and produce accurate results for regular geometries. Post-MVP will add unstructured FEM via mesh2d/triangle for arbitrary 2D shapes and tetgen for 3D. Finite difference gives 2nd-order spatial accuracy with Crank-Nicolson time stepping.

4. **Separate Python FastAPI service for food property database**: Food thermophysical properties depend on composition (water, protein, fat, carb, fiber, ash fractions) and temperature. The Choi-Okos correlations and empirical property models are well-established in Python food science libraries. Rather than reimplement in Rust, we run a lightweight FastAPI service that provides property lookup and composition-based prediction. The Rust solver caches properties per simulation to minimize RPC overhead.

5. **Microbial kinetics integrated into thermal solver**: Rather than post-processing temperature histories for lethality, the kinetics engine runs in lockstep with the thermal solver. At each timestep, for each spatial point, we integrate the lethal rate dF/dt = 10^((T-Tref)/z) using the same timestep as the thermal solver. This gives spatially resolved F0 maps and identifies the cold spot (minimum lethality point) which is critical for food safety validation.

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";  -- For fuzzy text search on food products

-- Users
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email TEXT UNIQUE NOT NULL,
    password_hash TEXT,  -- NULL for OAuth-only users
    name TEXT NOT NULL,
    avatar_url TEXT,
    auth_provider TEXT DEFAULT 'email',  -- email | google | github
    auth_provider_id TEXT,
    plan TEXT NOT NULL DEFAULT 'free',  -- free | pro | team
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
    role TEXT NOT NULL DEFAULT 'member',  -- owner | admin | member
    joined_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(org_id, user_id)
);
CREATE INDEX org_members_org_idx ON org_members(org_id);
CREATE INDEX org_members_user_idx ON org_members(user_id);

-- Food Products and Properties
CREATE TABLE food_products (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    category TEXT NOT NULL,  -- dairy | meat | vegetable | fruit | grain | beverage | prepared
    composition JSONB NOT NULL,  -- {water: 0.85, protein: 0.03, fat: 0.04, carb: 0.06, fiber: 0.01, ash: 0.01}
    thermal_conductivity_model TEXT DEFAULT 'choi_okos',
    specific_heat_model TEXT DEFAULT 'choi_okos',
    density_model TEXT DEFAULT 'choi_okos',
    freezing_point_c REAL,  -- For phase change modeling
    latent_heat_fusion REAL,  -- J/kg
    custom_properties JSONB,  -- User-defined property curves
    is_builtin BOOLEAN DEFAULT true,
    created_by UUID REFERENCES users(id),
    description TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX food_products_category_idx ON food_products(category);
CREATE INDEX food_products_name_trgm_idx ON food_products USING gin(name gin_trgm_ops);

-- Microbial Organisms and Kinetic Parameters
CREATE TABLE organisms (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,  -- e.g., "Clostridium botulinum" or "Listeria monocytogenes"
    common_name TEXT,  -- e.g., "Botulism" or "Listeria"
    organism_type TEXT NOT NULL,  -- pathogen | spoilage
    reference_temperature_c REAL NOT NULL DEFAULT 121.1,  -- Tref for F0 calculation
    d_value_minutes REAL NOT NULL,  -- D-value at reference temperature
    z_value_c REAL NOT NULL,  -- z-value (temperature sensitivity)
    food_matrix TEXT,  -- Food type these parameters are validated for (low_acid_canned, milk, meat, etc.)
    ph_range_min REAL,
    ph_range_max REAL,
    water_activity_min REAL,
    references TEXT,  -- Literature citations
    is_verified BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX organisms_type_idx ON organisms(organism_type);
CREATE INDEX organisms_name_trgm_idx ON organisms USING gin(name gin_trgm_ops);

-- Thermal Processing Projects
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    process_type TEXT NOT NULL,  -- retort | continuous_htst | uhts | pasteurizer | freezing | thawing
    geometry_type TEXT NOT NULL,  -- cylinder | sphere | brick | custom_2d | custom_3d
    geometry_params JSONB NOT NULL,  -- {radius: 0.05, height: 0.10} for cylinder in meters
    food_product_id UUID REFERENCES food_products(id),
    initial_temperature_c REAL NOT NULL DEFAULT 20.0,
    target_organisms UUID[],  -- Array of organism IDs for lethality calculation
    process_definition JSONB NOT NULL,  -- Retort cycle: [{phase: "come_up", duration_min: 15, target_temp_c: 121}, ...]
    mesh_resolution TEXT DEFAULT 'medium',  -- coarse | medium | fine | custom
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
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | running | completed | failed | cancelled
    execution_mode TEXT NOT NULL DEFAULT 'wasm',  -- wasm | server
    mesh_point_count INTEGER NOT NULL DEFAULT 0,
    timestep_seconds REAL NOT NULL DEFAULT 1.0,
    total_time_seconds REAL NOT NULL,
    results_url TEXT,  -- S3 URL for full temperature field time series
    cold_spot_location JSONB,  -- {r: 0.0, z: 0.05} or {x, y, z} coordinates
    cold_spot_lethality JSONB,  -- {organism_id: uuid, f0_value: 12.5, log_reduction: 10.2}
    min_lethality REAL,  -- Minimum F0 across all spatial points
    max_temperature_c REAL,
    quality_metrics JSONB,  -- {nutrient_retention: 0.85, color_change: 0.12}
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
    current_time_seconds REAL DEFAULT 0.0,
    convergence_data JSONB DEFAULT '[]',  -- [{iteration, residual, timestep}]
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX jobs_sim_idx ON simulation_jobs(simulation_id);
CREATE INDEX jobs_worker_idx ON simulation_jobs(worker_id);

-- Validation Reports (for regulatory submission)
CREATE TABLE validation_reports (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    simulation_id UUID NOT NULL REFERENCES simulations(id) ON DELETE CASCADE,
    report_type TEXT NOT NULL,  -- process_filing | haccp | validation_study
    pdf_url TEXT NOT NULL,  -- S3 URL to generated PDF
    metadata JSONB,  -- Report parameters, company info, process authority signature
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX reports_sim_idx ON validation_reports(simulation_id);

-- Usage Tracking (for billing enforcement)
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,  -- simulation_seconds | cloud_simulation | storage_bytes
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
pub struct FoodProduct {
    pub id: Uuid,
    pub name: String,
    pub category: String,
    pub composition: serde_json::Value,  // Composition fractions
    pub thermal_conductivity_model: String,
    pub specific_heat_model: String,
    pub density_model: String,
    pub freezing_point_c: Option<f64>,
    pub latent_heat_fusion: Option<f64>,
    pub custom_properties: Option<serde_json::Value>,
    pub is_builtin: bool,
    pub created_by: Option<Uuid>,
    pub description: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct Composition {
    pub water: f64,
    pub protein: f64,
    pub fat: f64,
    pub carb: f64,
    pub fiber: f64,
    pub ash: f64,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Organism {
    pub id: Uuid,
    pub name: String,
    pub common_name: Option<String>,
    pub organism_type: String,
    pub reference_temperature_c: f64,
    pub d_value_minutes: f64,
    pub z_value_c: f64,
    pub food_matrix: Option<String>,
    pub ph_range_min: Option<f64>,
    pub ph_range_max: Option<f64>,
    pub water_activity_min: Option<f64>,
    pub references: Option<String>,
    pub is_verified: bool,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Project {
    pub id: Uuid,
    pub owner_id: Uuid,
    pub org_id: Option<Uuid>,
    pub name: String,
    pub description: String,
    pub process_type: String,
    pub geometry_type: String,
    pub geometry_params: serde_json::Value,
    pub food_product_id: Option<Uuid>,
    pub initial_temperature_c: f64,
    pub target_organisms: Vec<Uuid>,
    pub process_definition: serde_json::Value,
    pub mesh_resolution: String,
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
    pub status: String,
    pub execution_mode: String,
    pub mesh_point_count: i32,
    pub timestep_seconds: f64,
    pub total_time_seconds: f64,
    pub results_url: Option<String>,
    pub cold_spot_location: Option<serde_json::Value>,
    pub cold_spot_lethality: Option<serde_json::Value>,
    pub min_lethality: Option<f64>,
    pub max_temperature_c: Option<f64>,
    pub quality_metrics: Option<serde_json::Value>,
    pub error_message: Option<String>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub wall_time_ms: Option<i32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct GeometryParams {
    pub geometry_type: GeometryType,
    pub dimensions: DimensionSpec,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum GeometryType {
    Cylinder,
    Sphere,
    Brick,
    Custom2D,
    Custom3D,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(untagged)]
pub enum DimensionSpec {
    Cylinder { radius: f64, height: f64 },  // meters
    Sphere { radius: f64 },
    Brick { length: f64, width: f64, height: f64 },
}

#[derive(Debug, Deserialize, Serialize)]
pub struct ProcessPhase {
    pub phase: String,  // come_up | hold | cooling
    pub duration_minutes: f64,
    pub target_temperature_c: f64,
    pub medium_htc: Option<f64>,  // Heat transfer coefficient (W/m²K) if known
}
```

---

## Solver Architecture Deep-Dive

### Governing Equations and Discretization

FoodProcess's thermal solver implements the **transient heat conduction equation** with temperature-dependent thermophysical properties:

```
ρ(T) · Cp(T) · ∂T/∂t = ∇ · [k(T) ∇T] + Q

where:
  T = temperature (°C or K)
  ρ(T) = density (kg/m³)
  Cp(T) = specific heat capacity (J/kg·K)
  k(T) = thermal conductivity (W/m·K)
  Q = volumetric heat generation (W/m³, typically 0 for foods)
```

For **2D axisymmetric cylindrical coordinates** (r, z):

```
ρCp ∂T/∂t = (1/r) ∂/∂r(r·k·∂T/∂r) + ∂/∂z(k·∂T/∂z)
```

**Boundary conditions:**
- **Natural convection** at food surface (in-container heating):
  ```
  -k ∂T/∂n = h(T_surface - T_medium)

  where h = heat transfer coefficient (W/m²K)
  For vertical cylinder in still fluid: h ≈ 1.42·(ΔT/L)^0.25  (laminar natural convection)
  For forced convection in retort: h ≈ 500-2000 W/m²K (steam condensation)
  ```

- **Symmetry** at centerline (r=0): ∂T/∂r = 0

**Discretization:** Finite difference on structured grid with implicit (Crank-Nicolson) time stepping:

```
T^{n+1}_i = T^n_i + (Δt/2) · [L(T^{n+1}) + L(T^n)]

where L(T) is the spatial discretization operator (2nd-order central differences)
```

This produces a sparse linear system **A·T^{n+1} = b** at each timestep, solved with sparse direct solver (UMFPACK via `sprs` crate) or iterative solver (Conjugate Gradient with ILU preconditioner for large systems).

**Phase change** (freezing/thawing) uses the **enthalpy method** to handle latent heat:

```
∂H/∂t = ∇ · [k(T) ∇T]

where H(T) = ∫ ρ·Cp dT + ρ·L·f(T)
      f(T) = liquid fraction (0 to 1), modeled with smooth transition over ±2°C around freezing point
```

### Microbial Inactivation Kinetics

At each spatial point and timestep, we integrate the **lethal rate** to compute cumulative lethality:

```
F(t) = ∫₀ᵗ L(T(τ)) dτ

where L(T) = 10^((T - T_ref) / z)  is the lethal rate

T_ref = reference temperature (121.1°C for C. botulinum, 70°C for pasteurization)
z = temperature sensitivity (°C for 1 log change in D-value)
```

**Log reduction** of microorganisms:

```
N(t) / N₀ = 10^(-F(t) / D_ref)

log_reduction = F(t) / D_ref

where D_ref = D-value at reference temperature (minutes)
```

For thermal processing validation, we require:
- **F₀ ≥ 3 minutes** (12-log reduction of C. botulinum) for low-acid canned foods
- **PU ≥ 15** for pasteurization of acidic products (pH < 4.6)

The solver tracks F-value at every mesh point and identifies the **cold spot** (location with minimum F-value).

### Client/Server Split (WASM Threshold)

```
Project created → Mesh generated → Point count extracted
    │
    ├── ≤10K points → WASM solver (browser)
    │   ├── Instant startup, no network latency
    │   ├── Results displayed immediately
    │   └── No server cost
    │
    └── >10K points → Server solver (Rust native)
        ├── Job queued via Redis
        ├── Worker picks up, runs native solver (parallel on multi-core)
        ├── Progress streamed via WebSocket (time, iteration, convergence)
        └── Results stored in S3, URL returned
```

The 10K-point threshold covers:
- 2D axisymmetric cylinder: 100×100 grid (r×z)
- 2D axisymmetric sphere: 120 radial × 90 angular ≈ 10K
- 2D Cartesian (brick cross-section): 100×100

Above 10K points:
- 3D problems (tetrahedral meshes with 50K-500K nodes)
- Fine-resolution 2D for steep temperature gradients
- Batch optimization (Monte Carlo or parameter sweeps)

---

## Architecture Deep-Dives

### 1. Thermal Simulation API Handler (Rust/Axum)

The primary endpoint receives a simulation request, generates mesh, decides between WASM and server execution, and for server execution enqueues a job with Redis and streams progress via WebSocket.

```rust
// src/api/handlers/simulation.rs

use axum::{
    extract::{Path, State, Json},
    http::StatusCode,
    response::IntoResponse,
};
use uuid::Uuid;
use crate::{
    db::models::{Simulation, Project, GeometryParams},
    solver::mesh::generate_mesh,
    state::AppState,
    auth::Claims,
    error::ApiError,
};

#[derive(serde::Deserialize)]
pub struct CreateSimulationRequest {
    pub timestep_seconds: Option<f64>,
    pub total_time_seconds: f64,
}

pub async fn create_simulation(
    State(state): State<AppState>,
    claims: Claims,
    Path(project_id): Path<Uuid>,
    Json(req): Json<CreateSimulationRequest>,
) -> Result<impl IntoResponse, ApiError> {
    // 1. Verify project ownership
    let project = sqlx::query_as!(
        Project,
        "SELECT * FROM projects WHERE id = $1 AND (owner_id = $2 OR org_id IN (
            SELECT org_id FROM org_members WHERE user_id = $2
        ))",
        project_id,
        claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Project not found"))?;

    // 2. Generate mesh based on geometry
    let geometry: GeometryParams = serde_json::from_value(project.geometry_params.clone())?;
    let mesh_resolution = &project.mesh_resolution;
    let mesh = generate_mesh(&geometry, mesh_resolution)?;
    let mesh_point_count = mesh.point_count();

    // 3. Check plan limits
    let user = sqlx::query_as!(
        crate::db::models::User,
        "SELECT * FROM users WHERE id = $1",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    if user.plan == "free" && mesh_point_count > 2000 {
        return Err(ApiError::PlanLimit(
            "Free plan supports meshes up to 2,000 points. Upgrade to Pro for unlimited."
        ));
    }

    // 4. Determine execution mode
    let execution_mode = if mesh_point_count <= 10_000 {
        "wasm"
    } else {
        "server"
    };

    // 5. Auto-select timestep if not provided (CFL stability criterion)
    let min_spacing = mesh.min_spacing();
    let max_alpha = 1e-6;  // Typical thermal diffusivity for food (m²/s)
    let cfl_timestep = 0.25 * min_spacing.powi(2) / max_alpha;  // CFL number 0.25
    let timestep = req.timestep_seconds.unwrap_or(cfl_timestep.max(0.1).min(10.0));

    // 6. Create simulation record
    let sim = sqlx::query_as!(
        Simulation,
        r#"INSERT INTO simulations
            (project_id, user_id, status, execution_mode, mesh_point_count,
             timestep_seconds, total_time_seconds)
        VALUES ($1, $2, $3, $4, $5, $6, $7)
        RETURNING *"#,
        project_id,
        claims.user_id,
        if execution_mode == "wasm" { "ready" } else { "pending" },
        execution_mode,
        mesh_point_count as i32,
        timestep,
        req.total_time_seconds,
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
            if user.plan == "team" { 10 } else { 0 },
            if user.plan == "team" { 4 } else { 1 },
        )
        .fetch_one(&state.db)
        .await?;

        // Publish to Redis job queue
        let mut conn = state.redis.get_async_connection().await?;
        redis::cmd("LPUSH")
            .arg("simulation:jobs")
            .arg(job.id.to_string())
            .query_async::<_, ()>(&mut conn)
            .await?;
    }

    Ok((StatusCode::CREATED, Json(sim)))
}

pub async fn get_simulation_results(
    State(state): State<AppState>,
    claims: Claims,
    Path((project_id, sim_id)): Path<(Uuid, Uuid)>,
) -> Result<impl IntoResponse, ApiError> {
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

    // Generate presigned S3 URL for result download
    let presigned_url = state.s3_client
        .presigned_get(&sim.results_url.unwrap(), 3600)
        .await?;

    Ok(Json(serde_json::json!({
        "status": "completed",
        "results_url": presigned_url,
        "cold_spot_location": sim.cold_spot_location,
        "cold_spot_lethality": sim.cold_spot_lethality,
        "min_lethality": sim.min_lethality,
        "max_temperature_c": sim.max_temperature_c,
    })))
}
```

### 2. Thermal Solver Core (Rust — shared between WASM and native)

The core heat conduction solver that builds and solves the finite difference system. This code compiles to both native (server) and WASM (browser) targets.

```rust
// solver-core/src/thermal.rs

use sprs::{CsMat, TriMat};
use crate::mesh::{Mesh2D, Point};
use crate::properties::{ThermalProperties, PropertyEvaluator};

pub struct ThermalSolver {
    pub mesh: Mesh2D,
    pub properties: Box<dyn PropertyEvaluator>,
    pub boundary_conditions: Vec<BoundaryCondition>,
    pub temperature: Vec<f64>,  // Current temperature field
    pub temperature_prev: Vec<f64>,  // Previous timestep
    pub lethality: Vec<f64>,  // Cumulative F-value at each point
}

#[derive(Clone)]
pub enum BoundaryCondition {
    Convection { nodes: Vec<usize>, htc: f64, ambient_temp: f64 },
    FixedTemperature { nodes: Vec<usize>, temperature: f64 },
    Insulated { nodes: Vec<usize> },  // Zero flux
}

impl ThermalSolver {
    pub fn new(
        mesh: Mesh2D,
        properties: Box<dyn PropertyEvaluator>,
        boundary_conditions: Vec<BoundaryCondition>,
        initial_temperature: f64,
    ) -> Self {
        let n = mesh.n_points();
        Self {
            mesh,
            properties,
            boundary_conditions,
            temperature: vec![initial_temperature; n],
            temperature_prev: vec![initial_temperature; n],
            lethality: vec![0.0; n],
        }
    }

    /// Run transient simulation from t=0 to t=total_time
    pub fn solve_transient(
        &mut self,
        timestep: f64,
        total_time: f64,
        organism: &Organism,
        callback: impl Fn(f64, &[f64]) -> bool,  // (time, temperature) -> continue?
    ) -> Result<SimulationResult, SolverError> {
        let n_steps = (total_time / timestep).ceil() as usize;
        let mut time = 0.0;

        for step in 0..n_steps {
            time += timestep;

            // Solve heat equation for this timestep (Crank-Nicolson)
            self.timestep_crank_nicolson(timestep)?;

            // Update lethality (integrate lethal rate)
            self.update_lethality(timestep, organism);

            // Callback for progress reporting
            if !callback(time, &self.temperature) {
                return Err(SolverError::Cancelled);
            }
        }

        // Identify cold spot (minimum lethality)
        let (cold_spot_idx, min_lethality) = self.lethality.iter()
            .enumerate()
            .min_by(|(_, a), (_, b)| a.partial_cmp(b).unwrap())
            .unwrap();

        let cold_spot_location = self.mesh.points[cold_spot_idx];
        let max_temperature = self.temperature.iter().cloned().fold(f64::NEG_INFINITY, f64::max);

        Ok(SimulationResult {
            final_temperature: self.temperature.clone(),
            lethality_field: self.lethality.clone(),
            cold_spot_location,
            min_lethality: *min_lethality,
            max_temperature,
        })
    }

    /// Single timestep using Crank-Nicolson (implicit, unconditionally stable)
    fn timestep_crank_nicolson(&mut self, dt: f64) -> Result<(), SolverError> {
        let n = self.mesh.n_points();
        let mut lhs = TriMat::new((n, n));  // Left-hand side: (I - θΔt·L)
        let mut rhs = vec![0.0; n];  // Right-hand side: (I + (1-θ)Δt·L)·T^n

        let theta = 0.5;  // Crank-Nicolson parameter (0.5 = trapezoidal)

        // Build finite difference stencils
        for i in 0..n {
            let neighbors = self.mesh.neighbors(i);
            let Ti = self.temperature_prev[i];

            // Evaluate properties at current temperature
            let props = self.properties.evaluate(Ti);
            let rho_cp = props.density * props.specific_heat;
            let k = props.thermal_conductivity;

            // Diagonal term
            let mut diag = rho_cp;
            let mut rhs_val = rho_cp * Ti;

            // Off-diagonal terms (neighbors)
            for &(j, distance) in &neighbors {
                let Tj = self.temperature_prev[j];
                let coeff = k * dt / (rho_cp * distance.powi(2));

                // LHS contribution
                lhs.add_triplet(i, j, -theta * coeff);
                diag += theta * coeff;

                // RHS contribution
                rhs_val += (1.0 - theta) * coeff * Tj;
            }

            lhs.add_triplet(i, i, diag);
            rhs[i] = rhs_val;
        }

        // Apply boundary conditions
        self.apply_boundary_conditions(&mut lhs, &mut rhs, dt, theta);

        // Solve linear system: lhs * T^{n+1} = rhs
        let lhs_csr: CsMat<f64> = lhs.to_csr();
        let solver = sprs::linalg::umfpack::UmfpackSolver::new();
        self.temperature = solver.solve(&lhs_csr, &rhs)?;

        // Update previous temperature
        self.temperature_prev.copy_from_slice(&self.temperature);

        Ok(())
    }

    fn apply_boundary_conditions(
        &self,
        lhs: &mut TriMat<f64>,
        rhs: &mut [f64],
        dt: f64,
        theta: f64,
    ) {
        for bc in &self.boundary_conditions {
            match bc {
                BoundaryCondition::Convection { nodes, htc, ambient_temp } => {
                    for &i in nodes {
                        let Ti = self.temperature_prev[i];
                        let props = self.properties.evaluate(Ti);
                        let rho_cp = props.density * props.specific_heat;

                        // Convection term: h·(T_amb - T_surf)
                        let conv_coeff = htc * dt / rho_cp;

                        // Add to diagonal
                        lhs.add_triplet(i, i, theta * conv_coeff);

                        // Add to RHS
                        rhs[i] += conv_coeff * ambient_temp;
                    }
                }
                BoundaryCondition::FixedTemperature { nodes, temperature } => {
                    for &i in nodes {
                        // Replace row with T = fixed_temp
                        lhs.clear_row(i);
                        lhs.add_triplet(i, i, 1.0);
                        rhs[i] = *temperature;
                    }
                }
                BoundaryCondition::Insulated { nodes } => {
                    // Zero flux: no additional terms needed (symmetry)
                }
            }
        }
    }

    fn update_lethality(&mut self, dt: f64, organism: &Organism) {
        let t_ref = organism.reference_temperature_c;
        let z = organism.z_value_c;

        for i in 0..self.temperature.len() {
            let T = self.temperature[i];

            // Lethal rate: L(T) = 10^((T - T_ref) / z)
            let lethal_rate = 10_f64.powf((T - t_ref) / z);

            // Integrate: F += L * dt (dt in seconds, convert to minutes)
            self.lethality[i] += lethal_rate * (dt / 60.0);
        }
    }
}

#[derive(Debug)]
pub struct SimulationResult {
    pub final_temperature: Vec<f64>,
    pub lethality_field: Vec<f64>,
    pub cold_spot_location: Point,
    pub min_lethality: f64,
    pub max_temperature: f64,
}

#[derive(Debug)]
pub enum SolverError {
    Convergence(String),
    Singular(String),
    Cancelled,
}

pub struct Organism {
    pub name: String,
    pub reference_temperature_c: f64,
    pub d_value_minutes: f64,
    pub z_value_c: f64,
}
```

### 3. Food Property Service (Python FastAPI)

Provides thermophysical property evaluation using Choi-Okos correlations and composition-based models.

```python
# food-properties-service/main.py

from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import numpy as np

app = FastAPI()

class Composition(BaseModel):
    water: float  # Mass fractions (sum to 1.0)
    protein: float
    fat: float
    carb: float
    fiber: float
    ash: float

class PropertyRequest(BaseModel):
    composition: Composition
    temperature_c: float

class PropertyResponse(BaseModel):
    thermal_conductivity: float  # W/(m·K)
    specific_heat: float  # J/(kg·K)
    density: float  # kg/m³

@app.post("/evaluate", response_model=PropertyResponse)
def evaluate_properties(req: PropertyRequest) -> PropertyResponse:
    """
    Evaluate thermophysical properties using Choi-Okos correlations.

    References:
    - Choi, Y., & Okos, M. R. (1986). Effects of temperature and composition on the
      thermal properties of foods. Food Engineering and Process Applications, 1, 93-101.
    """
    comp = req.composition
    T = req.temperature_c
    T_K = T + 273.15

    # Thermal conductivity (W/(m·K)) — linear combination of components
    k_water = 0.57109 + 1.7625e-3 * T - 6.7036e-6 * T**2
    k_protein = 0.17881 + 1.1958e-3 * T - 2.7178e-6 * T**2
    k_fat = 0.18071 - 2.7604e-4 * T - 1.7749e-7 * T**2
    k_carb = 0.20141 + 1.3874e-3 * T - 4.3312e-6 * T**2
    k_fiber = 0.18331 + 1.2497e-3 * T - 3.1683e-6 * T**2
    k_ash = 0.32962 + 1.4011e-3 * T - 2.9069e-6 * T**2

    k = (comp.water * k_water + comp.protein * k_protein + comp.fat * k_fat +
         comp.carb * k_carb + comp.fiber * k_fiber + comp.ash * k_ash)

    # Specific heat (J/(kg·K)) — linear combination
    cp_water = 4081.7 - 5.3062e-1 * T + 1.3516e-3 * T**2
    cp_protein = 2008.2 + 1.2089 * T - 1.3129e-3 * T**2
    cp_fat = 1984.2 + 1.4733 * T - 4.8008e-3 * T**2
    cp_carb = 1548.8 + 1.9625 * T - 5.9399e-3 * T**2
    cp_fiber = 1845.9 + 1.8306 * T - 4.6509e-3 * T**2
    cp_ash = 1092.6 + 1.8896 * T - 3.6817e-3 * T**2

    cp = (comp.water * cp_water + comp.protein * cp_protein + comp.fat * cp_fat +
          comp.carb * cp_carb + comp.fiber * cp_fiber + comp.ash * cp_ash)

    # Density (kg/m³) — volume additivity (1/ρ = Σ(xᵢ/ρᵢ))
    rho_water = 997.18 + 3.1439e-3 * T - 3.7574e-3 * T**2
    rho_protein = 1329.9 - 5.1840e-1 * T
    rho_fat = 925.59 - 4.1757e-1 * T
    rho_carb = 1599.1 - 3.1046e-1 * T
    rho_fiber = 1311.5 - 3.6589e-1 * T
    rho_ash = 2423.8 - 2.8063e-1 * T

    # Volume fractions
    v_water = comp.water / rho_water if comp.water > 0 else 0
    v_protein = comp.protein / rho_protein if comp.protein > 0 else 0
    v_fat = comp.fat / rho_fat if comp.fat > 0 else 0
    v_carb = comp.carb / rho_carb if comp.carb > 0 else 0
    v_fiber = comp.fiber / rho_fiber if comp.fiber > 0 else 0
    v_ash = comp.ash / rho_ash if comp.ash > 0 else 0

    v_total = v_water + v_protein + v_fat + v_carb + v_fiber + v_ash
    rho = 1.0 / v_total if v_total > 0 else 1000.0

    return PropertyResponse(
        thermal_conductivity=k,
        specific_heat=cp,
        density=rho
    )

@app.get("/health")
def health_check():
    return {"status": "ok"}
```

### 4. Temperature Visualization (React + Plotly.js + Three.js)

Interactive temperature profile plots and 3D thermal field visualization.

```typescript
// frontend/src/components/TemperatureVisualization.tsx

import { useEffect, useRef, useState } from 'react';
import Plotly from 'plotly.js-dist-min';
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';

interface SimulationResult {
  mesh: {
    points: Array<{r: number, z: number}>;
    triangles: number[][];
  };
  temperature_history: {
    times: number[];  // seconds
    cold_spot_temp: number[];  // Temperature at cold spot over time
    center_temp: number[];  // Temperature at geometric center
    surface_temp: number[];  // Average surface temperature
  };
  final_temperature: number[];  // Temperature at each mesh point (final time)
  lethality: number[];  // F-value at each mesh point
}

export function TemperatureVisualization({ result }: { result: SimulationResult }) {
  const plotRef = useRef<HTMLDivElement>(null);
  const threeDRef = useRef<HTMLDivElement>(null);
  const [view, setView] = useState<'profile' | '3d'>('profile');

  useEffect(() => {
    if (view === 'profile' && plotRef.current) {
      renderTemperatureProfile();
    } else if (view === '3d' && threeDRef.current) {
      render3DThermalField();
    }
  }, [view, result]);

  const renderTemperatureProfile = () => {
    if (!plotRef.current) return;

    const { times, cold_spot_temp, center_temp, surface_temp } = result.temperature_history;

    const trace_cold_spot = {
      x: times.map(t => t / 60),  // Convert to minutes
      y: cold_spot_temp,
      mode: 'lines',
      name: 'Cold Spot',
      line: { color: 'rgb(31, 119, 180)', width: 2 }
    };

    const trace_center = {
      x: times.map(t => t / 60),
      y: center_temp,
      mode: 'lines',
      name: 'Center',
      line: { color: 'rgb(255, 127, 14)', width: 2 }
    };

    const trace_surface = {
      x: times.map(t => t / 60),
      y: surface_temp,
      mode: 'lines',
      name: 'Surface (avg)',
      line: { color: 'rgb(44, 160, 44)', width: 2 }
    };

    const layout = {
      title: 'Temperature Profiles During Retort Process',
      xaxis: { title: 'Time (minutes)' },
      yaxis: { title: 'Temperature (°C)' },
      hovermode: 'x unified',
      legend: { x: 0.7, y: 0.1 },
      margin: { l: 60, r: 30, t: 50, b: 60 }
    };

    Plotly.newPlot(plotRef.current, [trace_cold_spot, trace_center, trace_surface], layout, {
      responsive: true,
      displayModeBar: true,
      modeBarButtonsToRemove: ['pan2d', 'lasso2d', 'select2d']
    });
  };

  const render3DThermalField = () => {
    if (!threeDRef.current) return;

    const container = threeDRef.current;
    container.innerHTML = '';  // Clear previous render

    // Setup Three.js scene
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x1a1a1a);

    const camera = new THREE.PerspectiveCamera(
      60,
      container.clientWidth / container.clientHeight,
      0.1,
      1000
    );
    camera.position.set(0.15, 0.15, 0.15);

    const renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(container.clientWidth, container.clientHeight);
    container.appendChild(renderer.domElement);

    const controls = new OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true;

    // Create mesh geometry from axisymmetric 2D → 3D (revolve around z-axis)
    const { points, triangles } = result.mesh;
    const n_theta = 32;  // Angular resolution for revolution

    const geometry = new THREE.BufferGeometry();
    const positions: number[] = [];
    const colors: number[] = [];

    // Temperature colormap (blue → cyan → green → yellow → red)
    const temp_min = Math.min(...result.final_temperature);
    const temp_max = Math.max(...result.final_temperature);

    const getColor = (temp: number): THREE.Color => {
      const t = (temp - temp_min) / (temp_max - temp_min);
      if (t < 0.25) {
        return new THREE.Color(0, t * 4, 1);  // Blue → Cyan
      } else if (t < 0.5) {
        return new THREE.Color(0, 1, 1 - (t - 0.25) * 4);  // Cyan → Green
      } else if (t < 0.75) {
        return new THREE.Color((t - 0.5) * 4, 1, 0);  // Green → Yellow
      } else {
        return new THREE.Color(1, 1 - (t - 0.75) * 4, 0);  // Yellow → Red
      }
    };

    // Generate 3D mesh by revolving 2D profile
    for (let i = 0; i < n_theta; i++) {
      const theta = (i / n_theta) * 2 * Math.PI;
      const theta_next = ((i + 1) / n_theta) * 2 * Math.PI;

      for (const tri of triangles) {
        const [i0, i1, i2] = tri;
        const p0 = points[i0];
        const p1 = points[i1];
        const p2 = points[i2];

        // Create two triangles for this profile triangle (quad in 3D)
        // Current angle
        positions.push(
          p0.r * Math.cos(theta), p0.z, p0.r * Math.sin(theta),
          p1.r * Math.cos(theta), p1.z, p1.r * Math.sin(theta),
          p2.r * Math.cos(theta), p2.z, p2.r * Math.sin(theta)
        );

        const c0 = getColor(result.final_temperature[i0]);
        const c1 = getColor(result.final_temperature[i1]);
        const c2 = getColor(result.final_temperature[i2]);
        colors.push(c0.r, c0.g, c0.b, c1.r, c1.g, c1.b, c2.r, c2.g, c2.b);

        // Next angle
        positions.push(
          p0.r * Math.cos(theta_next), p0.z, p0.r * Math.sin(theta_next),
          p1.r * Math.cos(theta_next), p1.z, p1.r * Math.sin(theta_next),
          p2.r * Math.cos(theta_next), p2.z, p2.r * Math.sin(theta_next)
        );
        colors.push(c0.r, c0.g, c0.b, c1.r, c1.g, c1.b, c2.r, c2.g, c2.b);
      }
    }

    geometry.setAttribute('position', new THREE.Float32BufferAttribute(positions, 3));
    geometry.setAttribute('color', new THREE.Float32BufferAttribute(colors, 3));
    geometry.computeVertexNormals();

    const material = new THREE.MeshPhongMaterial({
      vertexColors: true,
      side: THREE.DoubleSide,
      shininess: 30
    });

    const mesh = new THREE.Mesh(geometry, material);
    scene.add(mesh);

    // Lighting
    const ambientLight = new THREE.AmbientLight(0xffffff, 0.6);
    scene.add(ambientLight);

    const dirLight = new THREE.DirectionalLight(0xffffff, 0.8);
    dirLight.position.set(1, 1, 1);
    scene.add(dirLight);

    // Animation loop
    const animate = () => {
      requestAnimationFrame(animate);
      controls.update();
      renderer.render(scene, camera);
    };
    animate();

    // Handle resize
    const handleResize = () => {
      camera.aspect = container.clientWidth / container.clientHeight;
      camera.updateProjectionMatrix();
      renderer.setSize(container.clientWidth, container.clientHeight);
    };
    window.addEventListener('resize', handleResize);

    return () => {
      window.removeEventListener('resize', handleResize);
      renderer.dispose();
    };
  };

  return (
    <div className="temperature-visualization flex flex-col h-full">
      <div className="flex gap-2 p-4 bg-gray-800">
        <button
          className={`px-4 py-2 rounded ${view === 'profile' ? 'bg-blue-600' : 'bg-gray-700'}`}
          onClick={() => setView('profile')}
        >
          Temperature Profile
        </button>
        <button
          className={`px-4 py-2 rounded ${view === '3d' ? 'bg-blue-600' : 'bg-gray-700'}`}
          onClick={() => setView('3d')}
        >
          3D Thermal Field
        </button>
      </div>

      <div className="flex-1 relative">
        {view === 'profile' && <div ref={plotRef} className="w-full h-full" />}
        {view === '3d' && <div ref={threeDRef} className="w-full h-full" />}
      </div>

      <div className="p-4 bg-gray-800 text-white">
        <h3 className="font-bold mb-2">Lethality Summary</h3>
        <div className="grid grid-cols-3 gap-4">
          <div>
            <span className="text-gray-400">Min F₀:</span>
            <span className="ml-2 font-mono">{result.lethality.reduce((a, b) => Math.min(a, b), Infinity).toFixed(2)} min</span>
          </div>
          <div>
            <span className="text-gray-400">Max Temp:</span>
            <span className="ml-2 font-mono">{Math.max(...result.final_temperature).toFixed(1)} °C</span>
          </div>
          <div>
            <span className="text-gray-400">Process Status:</span>
            <span className="ml-2 font-bold text-green-400">
              {result.lethality.reduce((a, b) => Math.min(a, b), Infinity) >= 3.0 ? 'PASS' : 'FAIL'}
            </span>
          </div>
        </div>
      </div>
    </div>
  );
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–5)

**Day 1: Project initialization and Rust backend scaffold**
```bash
cargo init foodprocess-api
cd foodprocess-api
cargo add axum tokio serde serde_json sqlx uuid chrono tower-http jsonwebtoken bcrypt
cargo add --features postgres,runtime-tokio-rustls,uuid,chrono,json sqlx
cargo add aws-sdk-s3 redis reqwest
```
- `src/main.rs` — Axum app with CORS, tracing, graceful shutdown
- `src/config.rs` — Environment-based configuration (DATABASE_URL, JWT_SECRET, S3_BUCKET, REDIS_URL, PROPERTY_SERVICE_URL)
- `src/state.rs` — AppState struct (PgPool, Redis, S3Client, PropertyServiceClient)
- `src/error.rs` — ApiError enum with IntoResponse
- `Dockerfile` — Multi-stage build (builder + runtime)
- `docker-compose.yml` — PostgreSQL, Redis, MinIO (S3-compatible), Python property service

**Day 2: Database schema and migrations**
- `migrations/001_initial.sql` — All 10 tables: users, organizations, org_members, food_products, organisms, projects, simulations, simulation_jobs, validation_reports, usage_records
- `src/db/mod.rs` — Database pool initialization
- `src/db/models.rs` — All SQLx structs with FromRow derives (User, FoodProduct, Organism, Project, Simulation, etc.)
- Run `sqlx migrate run` to apply schema
- Seed script for initial data: 100 food products (dairy, meat, vegetables, fruits) with Choi-Okos compositions, 10 organisms (C. botulinum, L. monocytogenes, Salmonella, E. coli O157:H7, etc.)

**Day 3: Authentication system**
- `src/auth/mod.rs` — JWT token generation and validation middleware
- `src/auth/oauth.rs` — Google and GitHub OAuth 2.0 flow handlers
- `src/api/handlers/auth.rs` — Register, login, OAuth callback, refresh token, me endpoints
- Password hashing with bcrypt (cost factor 12)
- JWT with 24h access token + 30d refresh token
- Auth middleware that extracts Claims from Authorization header

**Day 4: User, project, and food product CRUD**
- `src/api/handlers/users.rs` — Get profile, update profile, delete account
- `src/api/handlers/projects.rs` — Create, list, get, update, delete, fork project endpoints
- `src/api/handlers/orgs.rs` — Create org, invite member, list members, remove member
- `src/api/handlers/food_products.rs` — List foods, search by category/name, get food details, create custom food
- `src/api/router.rs` — All route definitions with auth middleware
- Integration tests: auth flow, project CRUD, authorization checks

**Day 5: Python food properties service**
- `food-properties-service/main.py` — FastAPI service with /evaluate endpoint
- `food-properties-service/models.py` — Pydantic models for Composition, PropertyRequest, PropertyResponse
- `food-properties-service/choi_okos.py` — Choi-Okos correlation implementation (thermal conductivity, specific heat, density)
- `food-properties-service/requirements.txt` — fastapi, uvicorn, pydantic, numpy
- Docker container for service, health check endpoint
- Unit tests for property calculations (compare to published values for common foods)

### Phase 2 — Solver Core — Thermal Engine (Days 6–13)

**Day 6: Mesh generation framework**
- `solver-core/` — New Rust workspace member (shared between native and WASM)
- `solver-core/src/mesh.rs` — Mesh2D struct with Point, neighbors, min_spacing methods
- `solver-core/src/mesh_cylinder.rs` — Structured cylindrical grid generator (r × z)
- `solver-core/src/mesh_sphere.rs` — Structured spherical grid generator (radial + angular)
- `solver-core/src/mesh_brick.rs` — Structured Cartesian grid generator (x × y × z)
- Unit tests: verify mesh connectivity, neighbor distances, boundary identification

**Day 7: Thermal properties integration**
- `solver-core/src/properties.rs` — PropertyEvaluator trait, ThermalProperties struct
- `solver-core/src/properties_client.rs` — HTTP client to query Python property service
- `solver-core/src/properties_cache.rs` — In-memory LRU cache for property lookups (avoid RPC overhead)
- Temperature-dependent property evaluation with caching per (composition, temperature) pair
- Tests: mock property service, verify caching behavior

**Day 8: Thermal solver — finite difference discretization**
- `solver-core/src/thermal.rs` — ThermalSolver struct with mesh, properties, boundary conditions, temperature fields
- `solver-core/src/thermal_stencil.rs` — Finite difference stencils for 2D cylindrical (r, z), spherical, Cartesian coordinates
- `solver-core/src/boundary.rs` — BoundaryCondition enum (Convection, FixedTemperature, Insulated)
- Sparse matrix assembly (TriMat from `sprs` crate) for implicit time stepping
- Tests: steady-state heat conduction (compare to analytical solutions for simple geometries)

**Day 9: Crank-Nicolson time integration**
- `solver-core/src/thermal.rs` — timestep_crank_nicolson method implementing implicit time stepping
- Sparse linear solver integration (UMFPACK via `sprs` crate for direct solve, <10K unknowns)
- Iterative solver (Conjugate Gradient with ILU preconditioner) for larger systems (>10K unknowns)
- Adaptive timestep selection based on CFL criterion for stability
- Tests: transient heat conduction (compare to analytical solutions, e.g., 1D cylinder radial heating)

**Day 10: Boundary condition implementation**
- `solver-core/src/boundary.rs` — apply_boundary_conditions method modifying matrix/RHS for convection, fixed temperature, insulated boundaries
- Natural convection heat transfer coefficient correlations (Churchill-Chu for vertical cylinders)
- Forced convection correlations (Nusselt number for flow over cylinders/spheres)
- Steam condensation heat transfer (high HTC, 1000-2000 W/m²K)
- Tests: verify boundary condition application, steady-state with convection boundary

**Day 11: Microbial inactivation kinetics**
- `solver-core/src/kinetics.rs` — Organism struct, lethality integration (F-value calculation)
- `solver-core/src/kinetics.rs` — update_lethality method integrating lethal rate at each timestep
- Cold spot identification (minimum lethality point)
- Log reduction calculation (F / D_value)
- Tests: analytical lethality for constant-temperature exposure, multi-step temperature profile

**Day 12: Transient solver integration**
- `solver-core/src/thermal.rs` — solve_transient method orchestrating timestep loop with progress callbacks
- Result struct with final temperature field, lethality field, cold spot location, max temperature
- Checkpoint/resume capability (save intermediate state every N timesteps)
- Tests: full retort cycle simulation (come-up, hold, cooling), verify F0 ≥ 3 min

**Day 13: Process cycle simulation**
- `solver-core/src/process.rs` — ProcessPhase struct, process definition parsing
- `solver-core/src/process.rs` — simulate_process method iterating through phases (come-up, hold, cooling) with varying boundary conditions
- Temperature ramping (linear or exponential approach to target temperature)
- Tests: multi-phase process, verify temperature profiles and cumulative lethality

### Phase 3 — WASM Build + Frontend Foundation (Days 14–19)

**Day 14: WASM solver compilation**
- `solver-wasm/` — New workspace member for WASM target
- `solver-wasm/Cargo.toml` — wasm-bindgen, serde-wasm-bindgen, wee_alloc dependencies
- `solver-wasm/src/lib.rs` — WASM entry points: `solve_thermal_wasm()` with progress callback via JS
- `wasm-pack build --target web --release` pipeline
- `wasm-opt -Oz` for size optimization (target <3MB gzipped including sparse solver)
- JavaScript wrapper: `ThermalSolver` class that loads WASM and provides async API

**Day 15: Frontend scaffold and project setup**
```bash
npm create vite@latest frontend -- --template react-ts
cd frontend
npm i zustand @tanstack/react-query axios plotly.js three
npm i -D @types/plotly.js @types/three
```
- `src/App.tsx` — Router, auth context, layout with sidebar
- `src/stores/authStore.ts` — Zustand store for authentication state
- `src/stores/projectStore.ts` — Zustand store for project state
- `src/lib/api.ts` — Axios API client with JWT interceptor
- `src/components/Layout.tsx` — Main layout with header, sidebar, content area

**Day 16: Project creation and configuration UI**
- `src/pages/ProjectCreate.tsx` — Multi-step form: (1) Name/description, (2) Process type, (3) Geometry, (4) Food product, (5) Process definition
- `src/components/GeometryConfig.tsx` — Input fields for cylinder (radius, height), sphere (radius), brick (length, width, height)
- `src/components/FoodProductSelector.tsx` — Searchable dropdown with categories, show composition on selection
- `src/components/ProcessDefinition.tsx` — Table editor for process phases (come-up, hold, cooling) with duration and target temperature
- Form validation: ensure geometry dimensions > 0, process times > 0, total time ≤ plan limit

**Day 17: Project list and detail pages**
- `src/pages/ProjectList.tsx` — Grid/list view of user's projects with thumbnails, last modified, process type
- `src/pages/ProjectDetail.tsx` — Project overview with geometry visualization (2D schematic), food product info, process cycle chart
- `src/components/GeometryPreview.tsx` — Simple 2D/3D preview of geometry (SVG for 2D cross-section, Three.js for 3D wireframe)
- `src/components/ProcessCycleChart.tsx` — Plotly.js chart showing temperature vs. time for defined process phases
- Edit/delete project actions

**Day 18: Simulation configuration UI**
- `src/pages/SimulationConfig.tsx` — Form to configure simulation: mesh resolution (coarse/medium/fine), timestep (auto or manual), total time
- `src/components/OrganismSelector.tsx` — Multi-select dropdown for target organisms (display D-value, z-value)
- `src/components/MeshPreview.tsx` — Show mesh point count estimate, execution mode (WASM vs. server), estimated runtime
- "Run Simulation" button → creates simulation via API, navigates to results page

**Day 19: WASM solver integration**
- `src/lib/wasmSolver.ts` — Load and initialize WASM module, expose `runSimulation()` async function
- Progress callback from WASM → update UI progress bar
- Result parsing: binary format (Float64Array) → structured data (temperature field, lethality field)
- Error handling: catch WASM exceptions, display user-friendly error messages
- Integration test: run simple cylinder simulation in browser, verify F0 calculation

### Phase 4 — Visualization + Results UI (Days 20–25)

**Day 20: Temperature profile visualization (Plotly.js)**
- `src/components/TemperatureVisualization.tsx` — Plotly.js line chart for temperature vs. time at cold spot, center, surface
- Interactive hover: show exact temperature, time at cursor
- Multiple traces: cold spot (blue), center (orange), surface (green)
- Layout: responsive, zoom, pan, export as PNG
- Display process phases as vertical spans (come-up, hold, cooling)

**Day 21: 3D thermal field visualization (Three.js)**
- `src/components/ThermalField3D.tsx` — Three.js scene rendering 3D mesh with temperature colormap
- Geometry generation: revolve 2D axisymmetric profile to create 3D mesh
- Vertex colors: interpolate temperature to color (blue → cyan → green → yellow → red)
- OrbitControls: rotate, zoom, pan camera
- Lighting: ambient + directional for shading
- Color legend: show temperature scale

**Day 22: Lethality visualization**
- `src/components/LethalityMap.tsx` — 2D contour plot of F-value field (Plotly.js)
- Cold spot marker: highlight location with minimum lethality
- Contour levels: show iso-lethality lines (e.g., F0 = 1, 3, 6, 12 minutes)
- Side-by-side view: temperature field vs. lethality field
- Display log reduction for selected organism

**Day 23: Results summary and metrics**
- `src/pages/SimulationResults.tsx` — Results page with tabs: (1) Temperature profiles, (2) 3D thermal field, (3) Lethality map, (4) Summary
- `src/components/ResultsSummary.tsx` — Table of key metrics: min F0, max temp, cold spot location, process status (PASS/FAIL)
- `src/components/ProcessValidation.tsx` — Check if F0 ≥ 3 min for low-acid canned food, display pass/fail with visual indicator
- Export results as JSON, export charts as PNG

**Day 24: PDF report generation**
- `src/api/handlers/reports.rs` — Generate PDF validation report using `printpdf` or `wkhtmltopdf` via server-side rendering
- Report contents: project metadata, geometry, food product, process definition, temperature profiles, lethality summary, cold spot analysis
- Regulatory format: FDA process filing template (21 CFR 113) with required fields
- Upload PDF to S3, store URL in validation_reports table
- Frontend: "Generate Report" button → trigger PDF generation, download link when ready

**Day 25: Comparison and scenario analysis**
- `src/pages/CompareSimulations.tsx` — Compare multiple simulations side-by-side (e.g., different process times, temperatures)
- Overlay temperature profiles on same chart (different colors per simulation)
- Table comparing key metrics: min F0, max temp, process time, energy (estimated from heating curve)
- "What-if" analysis: vary process parameters, re-run simulation, see impact on lethality

### Phase 5 — Server-Side Solver + Job Queue (Days 26–30)

**Day 26: Simulation API endpoints**
- `src/api/handlers/simulation.rs` — create_simulation, get_simulation_results, cancel_simulation, list_simulations
- Mesh generation server-side for mesh point count extraction
- Execution mode routing: WASM (status="ready" immediately) vs. server (status="pending", enqueue job)
- Plan-based limits enforcement: free (2K points, 10 cloud simulations/month), pro (unlimited), team (priority queue)

**Day 27: Server-side simulation worker**
- `src/workers/simulation_worker.rs` — Redis BRPOP consumer, picks up jobs from "simulation:jobs" queue
- Run solver-core in native mode (multi-threaded for large meshes)
- Progress streaming: worker publishes progress → Redis pub/sub → WebSocket to client
- Result storage: serialize temperature field, lethality field → MessagePack or bincode → upload to S3
- Database update: mark simulation completed, store results_url, cold_spot_lethality

**Day 28: WebSocket for live simulation progress**
- `src/api/ws/mod.rs` — Axum WebSocket upgrade handler at `/ws/simulation/:sim_id`
- `src/api/ws/progress.rs` — Subscribe to Redis pub/sub channel for simulation progress
- Client receives: `{ progress_pct, current_time_seconds, iteration }`
- Connection management: heartbeat ping/pong every 30s, automatic cleanup on disconnect
- Frontend: `src/hooks/useSimulationProgress.ts` — React hook for WebSocket subscription, updates UI progress bar

**Day 29: Result download and streaming**
- `src/api/handlers/results.rs` — Presigned S3 URL generation for result download (1 hour expiry)
- Result format: JSON with temperature_history (times, cold_spot_temp, center_temp, surface_temp), final_temperature, lethality, mesh
- Compression: gzip results before S3 upload for large simulations
- Frontend: fetch and decompress results, parse JSON, pass to visualization components

**Day 30: Worker pool and scaling**
- `src/workers/pool.rs` — Worker pool with configurable concurrency (N workers per machine)
- Worker registration: each worker registers its hostname, available cores, memory in Redis
- Load balancing: distribute jobs to workers with available capacity
- Graceful shutdown: finish current job before exiting on SIGTERM
- Monitoring: expose Prometheus metrics (jobs completed, avg runtime, error rate)

### Phase 6 — Billing + Deployment (Days 31–35)

**Day 31: Stripe integration**
- `src/api/handlers/billing.rs` — Create checkout session, customer portal, webhook handler
- Plan tiers: Free (2K points, 3 projects, 10 cloud sims/month), Pro ($149/mo, unlimited), Team ($349/user/mo, priority queue + collaboration)
- Usage tracking: record simulation_seconds, cloud_simulation count per billing period
- Webhook events: `checkout.session.completed`, `customer.subscription.updated`, `customer.subscription.deleted`
- Update user plan in database on successful subscription

**Day 32: Usage enforcement**
- `src/middleware/usage.rs` — Middleware to check usage limits before simulation creation
- Query usage_records table for current billing period
- Free plan: block if >10 cloud simulations or >2K mesh points
- Display upgrade prompt in UI when limit reached
- Admin dashboard: view usage per user, aggregate metrics

**Day 33: AWS infrastructure setup**
- `infrastructure/terraform/` — Terraform configuration for AWS resources
- EC2 instances: c7g.xlarge for API server + workers (Graviton3, cost-effective)
- RDS PostgreSQL: db.t4g.micro for free tier, db.t4g.small for production
- ElastiCache Redis: cache.t4g.micro for job queue
- S3 buckets: foodprocess-results (simulation data), foodprocess-reports (PDFs)
- CloudFront distribution: serve static frontend assets + WASM bundles
- ALB: load balancer for API servers, SSL termination

**Day 34: Docker and CI/CD**
- `Dockerfile.api` — Multi-stage build for Rust API server
- `Dockerfile.worker` — Multi-stage build for simulation worker
- `Dockerfile.properties` — Python property service
- `.github/workflows/deploy.yml` — GitHub Actions workflow: build, test, push to ECR, deploy to EC2
- WASM build step: compile solver-wasm, upload to S3/CloudFront
- Database migrations: run `sqlx migrate run` as part of deployment

**Day 35: Monitoring and observability**
- `src/telemetry.rs` — OpenTelemetry tracing, export to Prometheus
- Metrics: API request latency, simulation job queue depth, job completion rate, error rate
- Grafana dashboards: system overview, simulation performance, user activity
- Sentry integration: error tracking, exception reporting
- Alerting: PagerDuty alerts for high error rate, queue backup, database connection issues

### Phase 7 — Testing + Documentation (Days 36–39)

**Day 36: Unit and integration tests**
- `solver-core/tests/` — Solver validation tests: analytical solutions for 1D/2D heat conduction, steady-state, transient
- `src/api/tests/` — API integration tests: auth flow, CRUD operations, simulation lifecycle
- `solver-wasm/tests/` — WASM module tests: load module, run simple simulation, verify results
- Coverage target: >80% for solver-core, >70% for API handlers

**Day 37: Validation benchmarks**
- `benchmarks/thermal/` — Compare solver results to published food engineering data (Ball & Olson textbook)
- `benchmarks/lethality/` — Validate F0 calculation against NFPA (National Food Processors Association) tables
- `benchmarks/performance/` — Measure solver runtime vs. mesh size, verify scaling (O(N log N) for direct solver)
- Document validation in `docs/validation.md`

**Day 38: User documentation**
- `docs/user-guide.md` — Getting started, create first project, configure process, run simulation, interpret results
- `docs/thermal-processing.md` — Food safety background (D-value, z-value, F0), retort process fundamentals, cold spot concept
- `docs/faq.md` — Common questions: mesh resolution selection, when to use WASM vs. server, interpretation of lethality values
- Video tutorials: project creation, simulation configuration, results analysis (record with Loom)

**Day 39: Developer documentation**
- `docs/api-reference.md` — OpenAPI spec for REST API, example requests/responses
- `docs/architecture.md` — System architecture diagram, data flow, solver algorithm overview
- `docs/deployment.md` — Self-hosting guide, AWS infrastructure setup, environment variables
- `README.md` — Project overview, tech stack, quick start guide
- Code comments: ensure all public functions have doc comments

### Phase 8 — Polish + Launch (Days 40–42)

**Day 40: UI/UX polish**
- Improve loading states: skeleton screens during data fetching, spinner for simulation progress
- Error states: friendly error messages, retry buttons, help links
- Responsive design: test on mobile/tablet, adjust layouts for small screens
- Accessibility: ARIA labels, keyboard navigation, screen reader support
- Empty states: helpful prompts when user has no projects, suggestions for first project

**Day 41: Performance optimization**
- Frontend: code splitting, lazy loading for visualization components (Plotly, Three.js)
- API: query optimization, add database indexes for common queries, connection pooling
- WASM: profile solver hotspots, optimize matrix assembly, reduce memory allocations
- CDN: ensure WASM bundles are cached, set proper cache headers
- Lighthouse audit: target score >90 for performance, accessibility, best practices

**Day 42: Launch preparation**
- Landing page: value proposition, use cases (retort validation, process optimization), pricing, CTA
- Demo project: pre-built example (canned tomato soup retort process) with full results
- Beta user onboarding: email campaign to food processing engineers, offer free trial
- Blog post: "Introducing FoodProcess — Modern Thermal Processing Simulation"
- Social media: announce launch on LinkedIn, Twitter, food science forums
- Monitor metrics: signups, first simulation completion rate, user feedback

---

## Critical Files and Directory Structure

```
foodprocess/
├── solver-core/                    # Core thermal solver (Rust, native + WASM)
│   ├── src/
│   │   ├── lib.rs                  # Public API for solver
│   │   ├── mesh.rs                 # Mesh data structures
│   │   ├── mesh_cylinder.rs        # Cylindrical mesh generator
│   │   ├── mesh_sphere.rs          # Spherical mesh generator
│   │   ├── mesh_brick.rs           # Cartesian mesh generator
│   │   ├── thermal.rs              # ThermalSolver (700 lines)
│   │   ├── thermal_stencil.rs      # FD stencil assembly
│   │   ├── boundary.rs             # Boundary conditions
│   │   ├── kinetics.rs             # Microbial inactivation
│   │   ├── process.rs              # Process cycle simulation
│   │   ├── properties.rs           # Property evaluator trait
│   │   ├── properties_client.rs    # HTTP client to property service
│   │   └── properties_cache.rs     # LRU cache for properties
│   ├── tests/
│   │   ├── analytical.rs           # Compare to analytical solutions
│   │   └── validation.rs           # Benchmarks vs. published data
│   └── Cargo.toml
│
├── solver-wasm/                    # WASM bindings for browser execution
│   ├── src/
│   │   └── lib.rs                  # WASM entry points (200 lines)
│   ├── Cargo.toml
│   └── pkg/                        # Generated WASM output
│
├── foodprocess-api/                # Rust backend API (Axum)
│   ├── src/
│   │   ├── main.rs                 # Server entry point
│   │   ├── config.rs               # Configuration
│   │   ├── state.rs                # AppState
│   │   ├── error.rs                # Error handling
│   │   ├── auth/
│   │   │   ├── mod.rs              # JWT auth
│   │   │   └── oauth.rs            # OAuth providers
│   │   ├── db/
│   │   │   ├── mod.rs              # Database setup
│   │   │   └── models.rs           # SQLx models (600 lines)
│   │   ├── api/
│   │   │   ├── router.rs           # Route definitions
│   │   │   ├── handlers/
│   │   │   │   ├── auth.rs         # Auth endpoints
│   │   │   │   ├── users.rs        # User CRUD
│   │   │   │   ├── projects.rs     # Project CRUD (400 lines)
│   │   │   │   ├── simulation.rs   # Simulation lifecycle (500 lines)
│   │   │   │   ├── results.rs      # Result download
│   │   │   │   ├── food_products.rs # Food database
│   │   │   │   ├── billing.rs      # Stripe integration
│   │   │   │   └── reports.rs      # PDF generation
│   │   │   └── ws/
│   │   │       ├── mod.rs          # WebSocket handler
│   │   │       └── progress.rs     # Simulation progress streaming
│   │   ├── workers/
│   │   │   ├── simulation_worker.rs # Job consumer (500 lines)
│   │   │   └── pool.rs             # Worker pool management
│   │   └── middleware/
│   │       └── usage.rs            # Usage enforcement
│   ├── migrations/
│   │   └── 001_initial.sql         # Database schema (250 lines)
│   └── Cargo.toml
│
├── food-properties-service/        # Python FastAPI service
│   ├── main.py                     # API endpoints (150 lines)
│   ├── choi_okos.py                # Property correlations (200 lines)
│   ├── requirements.txt
│   └── Dockerfile
│
├── frontend/                       # React + TypeScript
│   ├── src/
│   │   ├── App.tsx                 # Main app component
│   │   ├── main.tsx                # Entry point
│   │   ├── pages/
│   │   │   ├── ProjectList.tsx
│   │   │   ├── ProjectDetail.tsx
│   │   │   ├── ProjectCreate.tsx   # Multi-step form (400 lines)
│   │   │   ├── SimulationConfig.tsx
│   │   │   ├── SimulationResults.tsx (500 lines)
│   │   │   └── CompareSimulations.tsx
│   │   ├── components/
│   │   │   ├── Layout.tsx
│   │   │   ├── GeometryConfig.tsx
│   │   │   ├── GeometryPreview.tsx
│   │   │   ├── FoodProductSelector.tsx
│   │   │   ├── ProcessDefinition.tsx
│   │   │   ├── ProcessCycleChart.tsx
│   │   │   ├── OrganismSelector.tsx
│   │   │   ├── MeshPreview.tsx
│   │   │   ├── TemperatureVisualization.tsx (250 lines)
│   │   │   ├── ThermalField3D.tsx  # Three.js (300 lines)
│   │   │   ├── LethalityMap.tsx
│   │   │   ├── ResultsSummary.tsx
│   │   │   └── ProcessValidation.tsx
│   │   ├── stores/
│   │   │   ├── authStore.ts
│   │   │   ├── projectStore.ts     # Zustand store (200 lines)
│   │   │   └── simulationStore.ts
│   │   ├── hooks/
│   │   │   ├── useSimulationProgress.ts # WebSocket hook
│   │   │   └── useWasmSolver.ts
│   │   └── lib/
│   │       ├── api.ts              # Axios client
│   │       └── wasmSolver.ts       # WASM loader (150 lines)
│   ├── public/
│   │   └── wasm/                   # WASM bundles
│   ├── package.json
│   └── vite.config.ts
│
├── infrastructure/
│   └── terraform/
│       ├── main.tf                 # AWS resources
│       ├── variables.tf
│       └── outputs.tf
│
├── .github/
│   └── workflows/
│       ├── deploy.yml              # CI/CD pipeline
│       └── wasm-build.yml          # WASM compilation
│
├── docs/
│   ├── user-guide.md
│   ├── thermal-processing.md
│   ├── api-reference.md
│   ├── architecture.md
│   └── validation.md
│
├── docker-compose.yml              # Local development stack
└── README.md
```

**Total estimated lines: ~1,450 lines** (excluding generated code, dependencies, tests)

Key file sizes:
- `solver-core/src/thermal.rs`: 700 lines (solver core)
- `src/db/models.rs`: 600 lines (database models)
- `src/api/handlers/simulation.rs`: 500 lines (simulation API)
- `src/workers/simulation_worker.rs`: 500 lines (job worker)
- `frontend/src/pages/SimulationResults.tsx`: 500 lines (results UI)
- `src/api/handlers/projects.rs`: 400 lines (project CRUD)
- `frontend/src/pages/ProjectCreate.tsx`: 400 lines (project creation wizard)
- `frontend/src/components/ThermalField3D.tsx`: 300 lines (3D visualization)

---

## Solver Validation Suite

### Benchmark 1: Steady-State Radial Heat Conduction in Cylinder

**Scenario:** Infinite cylinder (radius R = 0.05 m) with fixed surface temperature T_s = 100°C, center temperature T_c = 20°C, constant thermal conductivity k = 0.5 W/(m·K).

**Analytical solution:**
```
T(r) = T_c + (T_s - T_c) · ln(r / R) / ln(R_inner / R)

For solid cylinder (R_inner → 0 limit):
T(r) = T_c + (T_s - T_c) · (r / R)²
```

**Expected results:**
- Temperature at r = 0.025 m (midpoint): **T = 60.0°C** (linear interpolation for infinite cylinder in steady state)
- Temperature gradient at surface: **dT/dr = 1600 K/m**

**Validation:** Solver result must match analytical solution within **< 0.5% error** for 100×100 mesh.

---

### Benchmark 2: Transient Heating of Canned Food (Semi-Infinite Slab Approximation)

**Scenario:** Slab of food (thickness L = 0.1 m, initially T_0 = 20°C) suddenly exposed to T_s = 121°C on both surfaces. Properties: k = 0.5 W/(m·K), ρ = 1000 kg/m³, Cp = 4000 J/(kg·K), giving thermal diffusivity α = k/(ρCp) = 1.25×10⁻⁷ m²/s.

**Analytical solution (center temperature):**
```
T_center(t) = T_s - (T_s - T_0) · Σ [(4 / ((2n+1)π)) · exp(-(2n+1)²π²αt / L²)]
```

**Expected results at t = 1800 s (30 min):**
- Center temperature: **T_center = 78.3°C**
- Fourier number: Fo = αt/L² = 0.0225

**Validation:** Solver result must match analytical solution within **< 1.0% error** for 50×50 mesh, Δt = 1 s.

---

### Benchmark 3: F0 Lethality for Standard Retort Process

**Scenario:** Can of low-acid food (cylinder radius = 0.04 m, height = 0.10 m) undergoing retort sterilization. Process: 15 min come-up to 121°C, 30 min hold at 121°C, 15 min cooling to 40°C. Target organism: C. botulinum (Tref = 121.1°C, D = 0.21 min, z = 10°C). Initial temperature T_0 = 20°C. Natural convection boundary (h = 500 W/(m²K)).

**Expected results:**
- Cold spot location: **r = 0 m, z = 0.05 m (geometric center)**
- Cold spot minimum F0: **F0 = 5.0–8.0 min** (depends on exact heating rate)
- Cold spot peak temperature: **T_peak = 115–118°C** (lags retort temperature)
- Process validation: **PASS** (F0 > 3 min)

**Validation:**
- Cold spot F0 must be **> 3.0 min** (FDA requirement for low-acid canned food)
- Cold spot temperature profile must lag retort temperature by **≥ 3°C** during hold phase
- Log reduction: **≥ 12 logs** (assuming initial load N0 = 10¹² CFU)

---

### Benchmark 4: Finite Difference Convergence — Grid Refinement Study

**Scenario:** 2D cylinder (radius = 0.05 m, height = 0.10 m) heated from T_0 = 20°C with convection boundary (h = 1000 W/(m²K), T_amb = 100°C). Run simulation to t = 600 s with three mesh resolutions:
- Coarse: 25×25 (625 points)
- Medium: 50×50 (2,500 points)
- Fine: 100×100 (10,000 points)

**Expected convergence:**
- Center temperature at t = 600 s:
  - Coarse: **T_center ≈ 65 ± 2°C**
  - Medium: **T_center ≈ 67 ± 0.5°C**
  - Fine: **T_center ≈ 67.5 ± 0.1°C** (reference value)

**Validation:**
- Relative error between medium and fine mesh: **< 1%**
- Convergence rate: **2nd-order** (error ∝ Δx²)
- Solver time scaling: **O(N log N)** for direct solver, **O(N^1.5)** for iterative solver

---

### Benchmark 5: Microbial Kinetics — Isothermal Inactivation

**Scenario:** Food sample held at constant T = 121°C for 10 minutes. Organism: C. botulinum (D = 0.21 min at 121°C, z = 10°C). Initial microbial load: N0 = 10¹² CFU.

**Analytical solution:**
```
log_reduction = t / D = 10 / 0.21 = 47.6 logs
N(t) / N0 = 10^(-47.6) ≈ 2.5 × 10⁻⁴⁸

F0 = ∫ L(T) dt = ∫ 10^((121.1 - 121.1) / 10) dt = 1.0 · 10 min = 10.0 min
```

**Expected results:**
- F0 value: **10.0 min** (exact for isothermal at Tref)
- Log reduction: **47.6 logs**
- Final microbial load: **N_final < 1 CFU** (effectively sterile)

**Validation:** Solver F0 calculation must match analytical value within **< 0.01 min** (0.1% error).

---

## Verification Checklist

### Solver Correctness
- [ ] Steady-state temperature field matches analytical solution (< 0.5% error)
- [ ] Transient center temperature matches semi-infinite slab solution (< 1% error)
- [ ] Boundary condition application: convection, fixed temperature, insulated
- [ ] F0 calculation matches isothermal analytical value (< 0.1% error)
- [ ] Cold spot identification: located at geometric center for symmetric heating
- [ ] Grid convergence: 2nd-order spatial accuracy verified
- [ ] Timestep stability: Crank-Nicolson unconditionally stable for all Δt

### API Functionality
- [ ] User registration, login, JWT token issuance
- [ ] OAuth flow (Google, GitHub) working end-to-end
- [ ] Project CRUD: create, read, update, delete, fork
- [ ] Simulation creation: mesh generation, execution mode routing
- [ ] WASM simulation: runs in browser, returns results
- [ ] Server simulation: job enqueued, worker picks up, results stored in S3
- [ ] WebSocket progress streaming: client receives updates during server simulation
- [ ] Presigned S3 URLs: result download with 1-hour expiry

### Frontend Features
- [ ] Project creation wizard: 5-step form, validation, submission
- [ ] Geometry preview: 2D cross-section for cylinder/sphere/brick
- [ ] Process cycle chart: visualize temperature profile over time
- [ ] Temperature profile plot: cold spot, center, surface traces
- [ ] 3D thermal field: Three.js rendering, interactive rotation/zoom
- [ ] Lethality map: contour plot with cold spot marker
- [ ] Results summary: min F0, max temp, PASS/FAIL indicator
- [ ] PDF report generation: trigger, download when ready

### Performance
- [ ] WASM solver: 2D cylinder (50×50 mesh, 1800 s simulation) completes in < 5 s
- [ ] Server solver: 2D cylinder (100×100 mesh) completes in < 30 s
- [ ] API response time: < 200 ms for project list, < 500 ms for simulation creation
- [ ] Frontend load time: < 3 s for first paint, < 5 s for interactive
- [ ] WASM bundle size: < 3 MB gzipped

### Integration
- [ ] Property service: Rust solver calls Python service, receives properties
- [ ] Property caching: repeated lookups hit cache, no redundant HTTP calls
- [ ] Redis job queue: jobs enqueued, consumed by workers in priority order
- [ ] S3 upload/download: results stored, presigned URLs work
- [ ] Stripe webhook: subscription events update user plan in database

### Security
- [ ] JWT tokens: properly signed, expire after 24h
- [ ] Password hashing: bcrypt with cost factor 12
- [ ] SQL injection prevention: all queries use parameterized SQLx macros
- [ ] Authorization checks: users can only access their own projects/simulations
- [ ] CORS configuration: only allow frontend origin
- [ ] S3 presigned URLs: expire after 1 hour, read-only access

---

## Deployment Architecture

### Production Environment (AWS)

```
┌─────────────────────────────────────────────────────────────────┐
│                          CloudFront CDN                          │
│  - Static assets (HTML, CSS, JS)                                │
│  - WASM bundles                                                  │
└────────────────────────────┬────────────────────────────────────┘
                             │
              ┌──────────────┴──────────────┐
              │                             │
              v                             v
┌─────────────────────────┐   ┌─────────────────────────┐
│  S3: Frontend Assets    │   │  S3: WASM Bundles       │
│  - index.html           │   │  - solver_wasm_bg.wasm  │
│  - main.js (React)      │   │  - solver_wasm.js       │
└─────────────────────────┘   └─────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│             Application Load Balancer (SSL Termination)          │
│  - Route /api/* → API Servers                                    │
│  - Route /ws/* → WebSocket Servers                               │
└────────────────────────────┬────────────────────────────────────┘
                             │
          ┌──────────────────┼──────────────────┐
          │                  │                  │
          v                  v                  v
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│  API Server 1   │  │  API Server 2   │  │  API Server N   │
│  (Axum/Rust)    │  │  (Axum/Rust)    │  │  (Axum/Rust)    │
│  c7g.xlarge     │  │  c7g.xlarge     │  │  c7g.xlarge     │
└────────┬────────┘  └────────┬────────┘  └────────┬────────┘
         │                    │                    │
         └────────────────────┼────────────────────┘
                              │
                   ┌──────────┴──────────┐
                   │                     │
                   v                     v
         ┌──────────────────┐  ┌──────────────────┐
         │  RDS PostgreSQL  │  │  ElastiCache     │
         │  (Multi-AZ)      │  │  Redis           │
         │  db.t4g.small    │  │  cache.t4g.micro │
         └──────────────────┘  └──────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                      Simulation Workers                          │
│  - Poll Redis job queue                                          │
│  - Run solver-core (native Rust)                                 │
│  - Upload results to S3                                          │
├─────────────────┬─────────────────┬─────────────────────────────┤
│  Worker 1       │  Worker 2       │  Worker N                   │
│  c7g.2xlarge    │  c7g.2xlarge    │  c7g.2xlarge                │
│  (8 vCPU)       │  (8 vCPU)       │  (8 vCPU)                   │
└─────────────────┴─────────────────┴─────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                Python Property Service                           │
│  - FastAPI                                                       │
│  - Choi-Okos correlations                                        │
│  - ECS Fargate (2 tasks, auto-scaling)                           │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                         S3 Buckets                               │
│  - foodprocess-results (simulation output)                       │
│  - foodprocess-reports (PDF validation reports)                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    Monitoring & Logging                          │
│  - Prometheus (metrics)                                          │
│  - Grafana (dashboards)                                          │
│  - Sentry (error tracking)                                       │
│  - CloudWatch Logs                                               │
└─────────────────────────────────────────────────────────────────┘
```

### Scaling Strategy

**Horizontal scaling:**
- API servers: Add more c7g.xlarge instances behind ALB (auto-scaling group based on CPU > 70%)
- Workers: Add more c7g.2xlarge instances, each polls Redis queue (scale based on queue depth > 20 jobs)
- Property service: ECS Fargate auto-scales to 2-10 tasks based on request rate

**Vertical scaling (if needed):**
- Database: Upgrade to db.t4g.medium (4 GB RAM) or db.m6g.large (8 GB RAM) for >10K users
- Redis: Upgrade to cache.t4g.small (1.5 GB RAM) for >1K concurrent simulations

**Cost estimates (monthly):**
- 2 × c7g.xlarge API (24/7): ~$120/month
- 4 × c7g.2xlarge workers (on-demand, 50% utilization): ~$200/month
- RDS db.t4g.small (Multi-AZ): ~$60/month
- ElastiCache cache.t4g.micro: ~$15/month
- S3 storage (1 TB): ~$25/month
- CloudFront (1 TB transfer): ~$85/month
- ECS Fargate (property service, 2 tasks): ~$30/month
- **Total: ~$535/month** for infrastructure supporting 500-1,000 active users

---

## Post-MVP Roadmap

### Q1: Advanced Geometries and Processes
- **3D arbitrary geometry:** STL import, tetrahedral mesh generation (TetGen), unstructured FEM solver
- **Freeze/thaw simulation:** Phase change with enthalpy method, moving boundary (Stefan problem)
- **Continuous HTST/UHT:** Residence time distribution (RTD) modeling, tubular heat exchanger
- **Irregular shapes:** Import 3D scan of real product geometry, adaptive mesh refinement

**Rationale:** Many food products have complex shapes (pouches, trays, meat cuts) that don't fit axisymmetric assumptions. 3D arbitrary geometry unlocks these use cases.

**Estimated effort:** 6 weeks (mesh generation + FEM solver + UI)

---

### Q2: Drying Simulation Module
- **Hot air drying:** Moisture diffusion (Fick's law), shrinkage, case hardening
- **Spray drying:** Droplet evaporation, particle temperature history
- **Freeze drying:** Sublimation front tracking, primary/secondary drying
- **Drying curves:** Moisture content vs. time, constant rate and falling rate periods
- **Quality modeling:** Browning (Maillard kinetics), vitamin degradation

**Rationale:** Drying is a critical unit operation in food processing (fruits, vegetables, powders). No good simulation tools exist for food-specific drying with quality prediction.

**Estimated effort:** 8 weeks (solver + moisture diffusion + quality models + UI)

---

### Q3: Predictive Microbiology (Growth Models)
- **Baranyi model:** Lag phase, exponential growth, stationary phase
- **Growth/no-growth boundaries:** Temperature, pH, water activity (aw) surfaces
- **Cold chain simulation:** Temperature abuse scenarios, shelf life prediction
- **Modified atmosphere packaging (MAP):** O2/CO2 evolution, respiration coupling
- **Shelf life dashboard:** Real-time shelf life remaining based on temperature logs

**Rationale:** Cold chain failures cause 30-40% of food waste. Predictive microbiology for growth (not just inactivation) is critical for shelf life and cold chain management.

**Estimated effort:** 6 weeks (growth models + cold chain simulation + UI)

---

### Q4: Process Optimization and Sensitivity Analysis
- **Optimization:** Minimize process time while maintaining F0 ≥ 3 min, minimize energy, maximize nutrient retention
- **Sensitivity analysis:** Vary parameters (process temp, time, HTC), visualize impact on lethality
- **Monte Carlo:** Uncertainty quantification (variation in food properties, initial temp distribution)
- **Multi-objective optimization:** Pareto front for safety vs. quality vs. energy

**Rationale:** Food engineers need to optimize processes for safety, quality, and cost. Automated optimization tools can save weeks of manual trial-and-error.

**Estimated effort:** 5 weeks (optimization algorithms + sensitivity UI + Monte Carlo)

---

### Q5: Fermentation and Mixing Module
- **Fermentation kinetics:** Monod model, Luedeking-Pirat product formation, pH and temperature coupling
- **Mixing simulation:** Power number, mixing time for different impeller types
- **Scale-up:** Predict production-scale performance from lab data
- **Bioreactor modeling:** Batch, fed-batch, continuous modes with oxygen transfer (kLa)

**Rationale:** Fermentation (beer, yogurt, cheese, bread) is a huge food sector. Mixing and fermentation simulation together enable end-to-end bioprocess design.

**Estimated effort:** 7 weeks (fermentation kinetics + mixing + bioreactor + UI)

---

### Q6: Regulatory Automation and Compliance
- **FDA process filing:** Automated form SF1-SF10 completion from simulation results
- **HACCP integration:** Critical control point verification, automated CCP monitoring
- **21 CFR 113/114 compliance:** Validation protocol generation, scheduled process calculation
- **Audit trail:** Full simulation history, parameter tracking, versioning for regulatory audit

**Rationale:** Regulatory submission is time-consuming and error-prone. Automating form completion and ensuring compliance saves weeks per product validation.

**Estimated effort:** 4 weeks (form templates + automation + audit trail)

---

### Q7: Machine Learning for Property Prediction
- **Property prediction from composition:** Train ML model (XGBoost, neural net) to predict k, Cp, ρ from composition without Choi-Okos
- **Anomaly detection:** Identify unusual temperature profiles in plant SCADA data
- **Process recommendation:** Suggest optimal process parameters based on product type and constraints
- **Surrogate modeling:** Train fast ML surrogate for expensive FEM simulations, enable real-time optimization

**Rationale:** ML can improve property predictions for novel formulations and enable real-time process control by replacing expensive FEM with fast surrogate models.

**Estimated effort:** 6 weeks (data collection + model training + integration)

---

### Q8: Enterprise Features
- **Multi-plant deployment:** On-premise installation for large food manufacturers
- **SCADA/MES integration:** Pull real-time temperature data from plant systems, compare to simulation
- **Custom product libraries:** Proprietary formulations with encrypted property databases
- **Team collaboration:** Shared projects, commenting, version control
- **API for process control:** Real-time simulation API for integration with plant control systems

**Rationale:** Large food companies need on-premise deployment and integration with existing systems. Enterprise features unlock Fortune 500 customers.

**Estimated effort:** 10 weeks (on-premise packaging + SCADA integration + enterprise auth)
