# 76. CastSim — Metal Casting and Foundry Process Simulation Platform

## Implementation Plan

**MVP Scope:** Browser-based CAD importer accepting STEP/STL geometry files with automatic tetrahedral mesh generation using TetGen integration, custom Volume-of-Fluid (VOF) free-surface tracker for mold filling simulation implementing finite volume method with PLIC interface reconstruction compiled to Rust for 10K-100K element meshes, enthalpy-based solidification solver with temperature-dependent thermophysical properties handling latent heat of fusion via source term linearization, Niyama criterion shrinkage porosity prediction with threshold calibration per alloy family (aluminum, iron, steel), 3D fill animation and porosity contour visualization using Three.js with WebGL-accelerated rendering, 10 common alloy database (A356, A380, gray iron, ductile iron, 1020 steel, 304 SS, C360 brass, AZ91 magnesium, CuSn10 bronze, IN718 nickel superalloy) with temperature-dependent viscosity/conductivity/specific heat stored in PostgreSQL, solidification sequence visualization showing last-to-solidify regions for riser placement guidance, STEP/STL geometry import/export and mesh quality statistics, Stripe billing with three tiers (Free / Pro $249/mo / Advanced $499/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| Filling Solver | Rust (nalgebra, ndarray) | VOF free-surface tracking, FVM Navier-Stokes with solidification coupling |
| Thermal Solver | Rust (sprs sparse matrix) | Enthalpy method, transient heat conduction with phase change |
| Meshing | TetGen (C++ via FFI) | Automatic tetrahedral mesh generation from STL/STEP with quality control |
| Material Database | Python 3.12 (FastAPI) | Thermophysical property interpolation, Scheil solidification fraction curves |
| Database | PostgreSQL 16 | Alloys, projects, simulations, mesh metadata |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | Mesh files (VTU/MSH), simulation results (field data), fill animations |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| 3D Visualization | Three.js + WebGL 2.0 | Mesh rendering, fill animation, temperature/porosity contours |
| Real-time | WebSocket (Axum) | Live simulation progress, solver convergence monitoring |
| Job Queue | Redis 7 + Tokio tasks | Simulation job management, priority queuing by plan tier |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | Geometry/mesh delivery, static frontend assets |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, error tracking |
| CI/CD | GitHub Actions | Rust tests, Docker image push, mesh validation pipeline |

### Key Architecture Decisions

1. **Rust-native CFD solver rather than OpenFOAM/ANSYS Fluent wrapping**: Building a custom Volume-of-Fluid (VOF) solver in Rust gives us full control over convergence algorithms, memory safety, and cloud scalability. OpenFOAM's C++ codebase is difficult to parallelize safely across cloud workers and has licensing ambiguities for SaaS. Our Rust solver uses finite volume method (FVM) with PLIC (Piecewise Linear Interface Calculation) for interface reconstruction, achieving comparable accuracy to commercial codes for casting-specific flow regimes (low Reynolds number, solidification-coupled).

2. **Enthalpy method for solidification rather than explicit tracking of solid/liquid interface**: The enthalpy formulation naturally handles mushy zone (partially solidified regions) and complex eutectic/peritectic transformations without explicit interface tracking. Latent heat of fusion is incorporated as a source term in the energy equation, and solid fraction is derived from enthalpy via temperature-enthalpy curves computed from thermodynamic databases. This approach scales to complex multi-component alloys and handles back-diffusion during solidification.

3. **TetGen for automatic meshing rather than requiring pre-meshed geometry**: 90% of foundry engineers use CAD (SolidWorks, Fusion 360, Inventor) but lack meshing expertise. TetGen integration via FFI provides automatic tetrahedral mesh generation from STEP/STL with quality control (minimum dihedral angle, maximum aspect ratio). Mesh refinement near mold surfaces and gating system ensures accurate fill front tracking. Alternative mesh generators (GMSH, Netgen) were rejected due to licensing or quality issues with thin-wall castings.

4. **Niyama criterion for porosity prediction rather than full Darcy flow in mushy zone**: For MVP, Niyama criterion (thermal gradient / cooling rate^0.5) provides 80% accuracy for shrinkage porosity location prediction at 1% of computational cost compared to full micro-porosity models. Niyama threshold is calibrated per alloy family using experimental data. Post-MVP, we will add Darcy flow for feeding distance and centerline porosity prediction in thick sections.

5. **Three.js for 3D visualization rather than VTK.js or custom WebGL**: Three.js provides production-ready scene graph, camera controls, lighting, and shader management. Fill animation is rendered by interpolating VOF field values onto mesh vertices and animating a color/opacity map. Temperature and porosity contours use fragment shaders with color scales. VTK.js was rejected due to bundle size (3MB+) and limited documentation. Custom WebGL would require reimplementing camera controls and lighting.

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";  -- For fuzzy text search on alloys

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

-- Projects (casting geometry + simulation setup)
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    geometry_url TEXT,  -- S3 URL to STEP/STL file
    mesh_url TEXT,  -- S3 URL to TetGen mesh (VTU format)
    mesh_stats JSONB,  -- {elements, nodes, min_quality, max_aspect_ratio}
    simulation_setup JSONB NOT NULL DEFAULT '{}',  -- Boundary conditions, initial conditions, solver params
    is_public BOOLEAN NOT NULL DEFAULT false,
    thumbnail_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);
CREATE INDEX projects_org_idx ON projects(org_id);
CREATE INDEX projects_updated_idx ON projects(updated_at DESC);

-- Simulations
CREATE TABLE simulations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    simulation_type TEXT NOT NULL,  -- filling | solidification | coupled
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | meshing | running | completed | failed | cancelled
    alloy_id UUID NOT NULL REFERENCES alloys(id),
    process_params JSONB NOT NULL DEFAULT '{}',  -- {pour_temp, pour_rate, mold_temp, ambient_temp}
    mesh_url TEXT,  -- Can override project mesh with refined mesh
    results_url TEXT,  -- S3 URL for result field data (VTU time series)
    animation_url TEXT,  -- S3 URL for fill animation video/frames
    porosity_data JSONB,  -- Summary: {max_niyama, porosity_locations: [{x,y,z, niyama_value}]}
    error_message TEXT,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    wall_time_seconds INTEGER,
    compute_hours REAL,  -- Billable compute time
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX sims_project_idx ON simulations(project_id);
CREATE INDEX sims_user_idx ON simulations(user_id);
CREATE INDEX sims_status_idx ON simulations(status);
CREATE INDEX sims_created_idx ON simulations(created_at DESC);

-- Simulation Jobs (server-side execution)
CREATE TABLE simulation_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    simulation_id UUID NOT NULL REFERENCES simulations(id) ON DELETE CASCADE,
    worker_id TEXT,
    instance_type TEXT,  -- c5.2xlarge | c5.4xlarge | g4dn.xlarge (GPU for filling)
    priority INTEGER DEFAULT 0,  -- Higher = more urgent (based on plan tier)
    progress_pct REAL DEFAULT 0.0,
    convergence_data JSONB DEFAULT '[]',  -- [{iteration, residual, timestep}]
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX jobs_sim_idx ON simulation_jobs(simulation_id);
CREATE INDEX jobs_status_idx ON simulation_jobs(worker_id);

-- Alloys (material database)
CREATE TABLE alloys (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,  -- e.g., "A356.0", "Gray Iron Class 30"
    family TEXT NOT NULL,  -- aluminum | iron | steel | copper | nickel | magnesium | titanium
    composition JSONB NOT NULL,  -- {Al: 93.0, Si: 7.0, Mg: 0.3, ...} in wt%
    liquidus_temp REAL NOT NULL,  -- Kelvin
    solidus_temp REAL NOT NULL,  -- Kelvin
    density_liquid REAL NOT NULL,  -- kg/m³
    density_solid REAL NOT NULL,  -- kg/m³
    thermal_conductivity_liquid REAL NOT NULL,  -- W/(m·K)
    thermal_conductivity_solid REAL NOT NULL,  -- W/(m·K)
    specific_heat_liquid REAL NOT NULL,  -- J/(kg·K)
    specific_heat_solid REAL NOT NULL,  -- J/(kg·K)
    latent_heat REAL NOT NULL,  -- J/kg
    viscosity_liquid REAL NOT NULL,  -- Pa·s at liquidus
    niyama_threshold REAL,  -- Critical value for porosity prediction (K·s^0.5/mm)
    fraction_solid_curve JSONB,  -- Temperature-fraction solid pairs from Scheil model
    description TEXT,
    is_builtin BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX alloys_family_idx ON alloys(family);
CREATE INDEX alloys_name_trgm_idx ON alloys USING gin(name gin_trgm_ops);

-- Mold Materials
CREATE TABLE mold_materials (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,  -- e.g., "Silica Sand", "Ceramic Shell", "H13 Die Steel"
    category TEXT NOT NULL,  -- sand | ceramic | metal_die | graphite
    thermal_conductivity REAL NOT NULL,  -- W/(m·K)
    specific_heat REAL NOT NULL,  -- J/(kg·K)
    density REAL NOT NULL,  -- kg/m³
    interface_htc REAL,  -- Interface heat transfer coefficient (W/(m²·K))
    permeability REAL,  -- For sand molds (m²), NULL for metal dies
    description TEXT,
    is_builtin BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX mold_materials_category_idx ON mold_materials(category);

-- Mesh Generation Jobs
CREATE TABLE mesh_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | running | completed | failed
    geometry_url TEXT NOT NULL,
    mesh_params JSONB NOT NULL,  -- {max_volume, min_angle, refinement_regions}
    mesh_url TEXT,  -- S3 URL to resulting mesh
    mesh_stats JSONB,  -- {elements, nodes, min_quality, max_aspect_ratio, generation_time_s}
    error_message TEXT,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX mesh_jobs_project_idx ON mesh_jobs(project_id);
CREATE INDEX mesh_jobs_status_idx ON mesh_jobs(status);

-- Usage Tracking (for billing enforcement)
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,  -- compute_hours | storage_gb | mesh_generation
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
    pub geometry_url: Option<String>,
    pub mesh_url: Option<String>,
    pub mesh_stats: Option<serde_json::Value>,
    pub simulation_setup: serde_json::Value,
    pub is_public: bool,
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
    pub alloy_id: Uuid,
    pub process_params: serde_json::Value,
    pub mesh_url: Option<String>,
    pub results_url: Option<String>,
    pub animation_url: Option<String>,
    pub porosity_data: Option<serde_json::Value>,
    pub error_message: Option<String>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub wall_time_seconds: Option<i32>,
    pub compute_hours: Option<f32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Alloy {
    pub id: Uuid,
    pub name: String,
    pub family: String,
    pub composition: serde_json::Value,
    pub liquidus_temp: f32,
    pub solidus_temp: f32,
    pub density_liquid: f32,
    pub density_solid: f32,
    pub thermal_conductivity_liquid: f32,
    pub thermal_conductivity_solid: f32,
    pub specific_heat_liquid: f32,
    pub specific_heat_solid: f32,
    pub latent_heat: f32,
    pub viscosity_liquid: f32,
    pub niyama_threshold: Option<f32>,
    pub fraction_solid_curve: Option<serde_json::Value>,
    pub description: Option<String>,
    pub is_builtin: bool,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct MoldMaterial {
    pub id: Uuid,
    pub name: String,
    pub category: String,
    pub thermal_conductivity: f32,
    pub specific_heat: f32,
    pub density: f32,
    pub interface_htc: Option<f32>,
    pub permeability: Option<f32>,
    pub description: Option<String>,
    pub is_builtin: bool,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct SimulationRequest {
    pub simulation_type: SimulationType,
    pub alloy_id: Uuid,
    pub mold_material_id: Uuid,
    pub process_params: ProcessParams,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum SimulationType {
    Filling,
    Solidification,
    Coupled,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct ProcessParams {
    pub pour_temp: f32,       // Kelvin
    pub pour_rate: f32,       // kg/s or m³/s
    pub mold_temp: f32,       // Kelvin (initial)
    pub ambient_temp: f32,    // Kelvin
    pub gravity: [f32; 3],    // m/s² vector (e.g., [0, -9.81, 0])
    pub fill_time: Option<f32>,  // seconds (for filling sim)
    pub solidification_time: Option<f32>,  // seconds (for solidification sim)
}

#[derive(Debug, Deserialize, Serialize)]
pub struct SolverOptions {
    pub time_step: f32,          // seconds
    pub max_iterations: u32,
    pub convergence_tol: f32,
    pub courant_number: f32,     // For filling CFL condition
    pub under_relaxation: f32,   // For nonlinear convergence
}
```

---

## Solver Architecture Deep-Dive

### Governing Equations and Discretization

CastSim's filling solver implements **Volume of Fluid (VOF)** for free-surface tracking coupled with Navier-Stokes equations for fluid flow. The VOF method tracks the volume fraction α ∈ [0,1] in each cell, where α=1 is liquid metal, α=0 is air/gas, and 0<α<1 is interface cells.

**Conservation Equations:**

```
Continuity (mass):
∂ρ/∂t + ∇·(ρu) = 0

Momentum (Navier-Stokes with Boussinesq approximation):
∂(ρu)/∂t + ∇·(ρuu) = -∇p + ∇·[μ(∇u + ∇uᵀ)] + ρg + F_body

VOF Transport:
∂α/∂t + ∇·(αu) = 0

Energy (for coupled filling-solidification):
∂(ρH)/∂t + ∇·(ρuH) = ∇·(k∇T) + S_latent
```

Where:
- **ρ**: Density (kg/m³), interpolated as ρ = α·ρ_metal + (1-α)·ρ_air
- **u**: Velocity field (m/s)
- **p**: Pressure (Pa)
- **μ**: Dynamic viscosity (Pa·s), interpolated with VOF fraction
- **g**: Gravity vector (m/s²)
- **F_body**: Body forces (e.g., surface tension via CSF model)
- **α**: VOF fraction (0=air, 1=metal)
- **H**: Specific enthalpy (J/kg)
- **T**: Temperature (K)
- **k**: Thermal conductivity (W/(m·K))
- **S_latent**: Latent heat source term during solidification

**PLIC Interface Reconstruction:** In interface cells (0<α<1), the Piecewise Linear Interface Calculation (PLIC) method reconstructs the actual interface as a plane within the cell. The plane normal **n** is computed from the gradient of α:

```
n = -∇α / |∇α|
```

The plane position is determined by enforcing the volume constraint that the liquid fraction equals α. This gives sharp, non-diffusive interface tracking compared to simple averaging methods.

**Solidification Coupling (Enthalpy Method):** Temperature and solid fraction are computed from enthalpy H via the temperature-enthalpy relation:

```
H(T) = ∫(T_ref → T) c_p(T') dT'  + f_s(T)·L

where:
  c_p(T) = specific heat at temperature T
  f_s(T) = solid fraction from Scheil curve (0 at liquidus, 1 at solidus)
  L = latent heat of fusion
```

For T < T_solidus: f_s = 1, all latent heat released
For T_solidus < T < T_liquidus: f_s = Scheil(T), partial solidification
For T > T_liquidus: f_s = 0, fully liquid

The latent heat source term in the energy equation is:

```
S_latent = ρ·L·∂f_s/∂t = ρ·L·(df_s/dT)·(∂T/∂t)
```

This is linearized using implicit time discretization for stability.

**Mushy Zone Momentum Damping:** In the mushy zone (0<f_s<1), the Carman-Kozeny permeability model adds a drag force that damps velocity as the metal solidifies:

```
F_drag = -C·(f_s²/(1-f_s)³ + ε)·u

where C = Carman constant (~10^6 - 10^8), ε = small number to avoid singularity
```

This prevents flow through solidified regions without explicit tracking of solid/liquid boundaries.

### Niyama Criterion for Porosity Prediction

The Niyama criterion predicts shrinkage porosity locations based on local thermal conditions during solidification:

```
Ny = G / √(dT/dt)

where:
  G = |∇T| = thermal gradient magnitude (K/m)
  dT/dt = cooling rate (K/s)
```

**Physical Interpretation:** High Niyama values (steep gradient + slow cooling) indicate good feeding conditions — liquid metal can flow through mushy zone to compensate for shrinkage. Low Niyama values (shallow gradient or fast cooling) indicate poor feeding → shrinkage porosity forms.

**Threshold Calibration:** The critical Niyama value N_crit is alloy-dependent:
- Aluminum alloys (A356, A380): N_crit ≈ 1.0 - 2.0 K·s^0.5/mm
- Gray iron: N_crit ≈ 0.5 - 1.0 K·s^0.5/mm
- Steel: N_crit ≈ 2.0 - 4.0 K·s^0.5/mm

Regions with Ny < N_crit are flagged as porosity-prone. Our implementation stores N_crit in the alloys table and computes Niyama field post-processing after solidification completes.

### Finite Volume Discretization

**Mesh Structure:** Unstructured tetrahedral mesh from TetGen. Each element stores:
- 4 vertex indices
- 6 edge indices (for gradient reconstruction)
- 4 face indices (for flux computation)
- Centroid position
- Volume

**Spatial Discretization:** Finite Volume Method (FVM) with cell-centered variables. For a generic conservation equation:

```
∂(ρφ)/∂t + ∇·(ρuφ) = ∇·(Γ∇φ) + S
```

Integrate over control volume V_i with surface ∂V_i:

```
dφ_i/dt = (1/V_i)·[∑_faces F_face - S_i·V_i]

where F_face = convective flux - diffusive flux
```

**Convective Flux:** Upwind scheme for stability (first-order) with optional TVD limiter for second-order accuracy:

```
F_conv,face = (ρu·n)_face · φ_upwind
```

**Diffusive Flux:** Central differencing with gradient reconstruction:

```
F_diff,face = (Γ·∇φ·n)_face ≈ Γ_face · (φ_neighbor - φ_cell) / d_face
```

**Time Integration:** Explicit Euler for filling (with CFL condition), implicit Euler for solidification (unconditionally stable but requires linear solve).

**CFL Condition for Filling:** Time step constrained by:

```
Δt ≤ Co · min(h / |u|)

where Co = Courant number (typically 0.5), h = element size
```

---

## Architecture Deep-Dives

### 1. Simulation API Handler (Rust/Axum)

Endpoint to create and submit a casting simulation. Validates geometry, checks plan limits, determines mesh resolution, and enqueues job.

```rust
// src/api/handlers/simulation.rs

use axum::{
    extract::{Path, State, Json},
    http::StatusCode,
    response::IntoResponse,
};
use uuid::Uuid;
use crate::{
    db::models::{Simulation, SimulationRequest, SimulationType},
    state::AppState,
    auth::Claims,
    error::ApiError,
    mesh::validate_mesh_quality,
};

#[derive(serde::Deserialize)]
pub struct CreateSimulationRequest {
    pub simulation_type: SimulationType,
    pub alloy_id: Uuid,
    pub mold_material_id: Uuid,
    pub process_params: serde_json::Value,
    pub use_project_mesh: bool,
    pub mesh_refinement: Option<MeshRefinement>,
}

#[derive(serde::Deserialize)]
pub struct MeshRefinement {
    pub max_element_volume: f32,  // m³
    pub min_dihedral_angle: f32,  // degrees
    pub surface_refinement_factor: f32,
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

    // 2. Check user plan limits
    let user = sqlx::query_as!(
        crate::db::models::User,
        "SELECT * FROM users WHERE id = $1",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    // 3. Validate mesh exists and meets quality criteria
    let mesh_url = if req.use_project_mesh {
        project.mesh_url
            .as_ref()
            .ok_or(ApiError::BadRequest("Project has no mesh. Generate mesh first."))?
            .clone()
    } else {
        // Trigger new mesh generation with custom refinement
        let mesh_job = create_mesh_job(
            &state,
            project_id,
            project.geometry_url.as_ref()
                .ok_or(ApiError::BadRequest("Project has no geometry"))?,
            req.mesh_refinement.unwrap_or_default(),
        ).await?;
        return Ok((StatusCode::ACCEPTED, Json(serde_json::json!({
            "status": "meshing",
            "mesh_job_id": mesh_job.id,
            "message": "Mesh generation in progress. Submit simulation after meshing completes."
        }))));
    };

    // 4. Download and validate mesh
    let mesh_data = state.s3_client
        .get_object()
        .bucket("castsim-meshes")
        .key(mesh_url.strip_prefix("s3://castsim-meshes/").unwrap())
        .send()
        .await?;

    let mesh_bytes = mesh_data.body.collect().await?.into_bytes();
    let mesh = crate::mesh::load_vtu_mesh(&mesh_bytes)?;

    // Check element count against plan limits
    let element_limit = match user.plan.as_str() {
        "free" => 100_000,
        "pro" => 500_000,
        "advanced" => 5_000_000,
        _ => 100_000,
    };

    if mesh.elements.len() > element_limit {
        return Err(ApiError::PlanLimit(format!(
            "Mesh has {} elements but {} plan supports up to {}. Upgrade or coarsen mesh.",
            mesh.elements.len(), user.plan, element_limit
        )));
    }

    // 5. Fetch alloy properties
    let alloy = sqlx::query_as!(
        crate::db::models::Alloy,
        "SELECT * FROM alloys WHERE id = $1",
        req.alloy_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Alloy not found"))?;

    // 6. Create simulation record
    let sim = sqlx::query_as!(
        Simulation,
        r#"INSERT INTO simulations
            (project_id, user_id, simulation_type, status, alloy_id,
             process_params, mesh_url)
        VALUES ($1, $2, $3, 'pending', $4, $5, $6)
        RETURNING *"#,
        project_id,
        claims.user_id,
        serde_json::to_string(&req.simulation_type)?,
        req.alloy_id,
        req.process_params,
        mesh_url,
    )
    .fetch_one(&state.db)
    .await?;

    // 7. Enqueue simulation job
    let priority = match user.plan.as_str() {
        "advanced" => 20,
        "pro" => 10,
        _ => 0,
    };

    let job = sqlx::query_as!(
        crate::db::models::SimulationJob,
        r#"INSERT INTO simulation_jobs (simulation_id, priority, instance_type)
        VALUES ($1, $2, $3) RETURNING *"#,
        sim.id,
        priority,
        // GPU instance for filling, CPU for solidification
        match req.simulation_type {
            SimulationType::Filling | SimulationType::Coupled => "g4dn.xlarge",
            SimulationType::Solidification => "c5.4xlarge",
        }
    )
    .fetch_one(&state.db)
    .await?;

    // Publish to Redis job queue
    state.redis
        .publish("simulation:jobs", serde_json::to_string(&job.id)?)
        .await?;

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
            "progress": get_job_progress(&state, sim.id).await?,
        })));
    }

    // Generate presigned URLs for results download
    let results_presigned = state.s3_client
        .generate_presigned_url(
            sim.results_url.as_ref().unwrap(),
            3600,  // 1 hour expiry
        )
        .await?;

    let animation_presigned = if let Some(anim_url) = &sim.animation_url {
        Some(state.s3_client.generate_presigned_url(anim_url, 3600).await?)
    } else {
        None
    };

    Ok(Json(serde_json::json!({
        "status": "completed",
        "results_url": results_presigned,
        "animation_url": animation_presigned,
        "porosity_data": sim.porosity_data,
        "wall_time_seconds": sim.wall_time_seconds,
        "compute_hours": sim.compute_hours,
    })))
}
```

### 2. VOF Filling Solver Core (Rust)

The core VOF solver for mold filling simulation. Solves coupled VOF transport and Navier-Stokes equations with PLIC interface reconstruction.

```rust
// src/solver/vof_solver.rs

use nalgebra::{DVector, DMatrix};
use crate::mesh::{Mesh, Element, Face};
use crate::db::models::{Alloy, ProcessParams};

pub struct VofSolver {
    mesh: Mesh,
    alloy: Alloy,
    air_density: f32,       // ~1.2 kg/m³
    air_viscosity: f32,     // ~1.8e-5 Pa·s

    // Field variables (cell-centered)
    vof: Vec<f32>,          // Volume fraction α ∈ [0,1]
    velocity: Vec<[f32; 3]>,  // Velocity vector u
    pressure: Vec<f32>,     // Pressure p
    temperature: Vec<f32>,  // Temperature T (for coupled filling-solidification)

    // Gradients and auxiliary
    vof_gradient: Vec<[f32; 3]>,
    pressure_gradient: Vec<[f32; 3]>,

    // Time stepping
    time: f32,
    dt: f32,
    courant_number: f32,
}

impl VofSolver {
    pub fn new(mesh: Mesh, alloy: Alloy, params: &ProcessParams) -> Self {
        let n_cells = mesh.elements.len();

        Self {
            mesh,
            alloy,
            air_density: 1.2,
            air_viscosity: 1.8e-5,
            vof: vec![0.0; n_cells],  // Initially empty (all air)
            velocity: vec![[0.0, 0.0, 0.0]; n_cells],
            pressure: vec![101325.0; n_cells],  // Atmospheric pressure
            temperature: vec![params.pour_temp; n_cells],
            vof_gradient: vec![[0.0, 0.0, 0.0]; n_cells],
            pressure_gradient: vec![[0.0, 0.0, 0.0]; n_cells],
            time: 0.0,
            dt: 0.001,  // Initial timestep, will be adjusted by CFL
            courant_number: 0.5,
        }
    }

    pub fn solve_timestep(&mut self) -> Result<(), SolverError> {
        // 1. Update timestep based on CFL condition
        self.update_timestep();

        // 2. Compute gradients (for PLIC and pressure gradient)
        self.compute_gradients();

        // 3. PLIC interface reconstruction in interface cells
        self.reconstruct_interface();

        // 4. Solve momentum equation (predictor step)
        self.solve_momentum_predictor()?;

        // 5. Solve pressure correction (Poisson equation for incompressibility)
        self.solve_pressure_correction()?;

        // 6. Correct velocity to satisfy continuity
        self.correct_velocity();

        // 7. Advect VOF field (geometric advection for sharp interface)
        self.advect_vof();

        // 8. Apply boundary conditions
        self.apply_boundary_conditions();

        self.time += self.dt;
        Ok(())
    }

    fn update_timestep(&mut self) {
        let mut max_velocity = 0.0;
        let mut min_cell_size = f32::INFINITY;

        for (i, elem) in self.mesh.elements.iter().enumerate() {
            let vel_mag = (self.velocity[i][0].powi(2)
                         + self.velocity[i][1].powi(2)
                         + self.velocity[i][2].powi(2)).sqrt();
            max_velocity = max_velocity.max(vel_mag);

            let h = elem.characteristic_length();
            min_cell_size = min_cell_size.min(h);
        }

        if max_velocity > 1e-6 {
            self.dt = self.courant_number * min_cell_size / max_velocity;
        }

        // Clamp timestep to reasonable range
        self.dt = self.dt.clamp(1e-6, 0.01);
    }

    fn compute_gradients(&mut self) {
        // Compute VOF gradient using Gauss divergence theorem
        for i in 0..self.mesh.elements.len() {
            let elem = &self.mesh.elements[i];
            let mut grad = [0.0, 0.0, 0.0];

            for face_idx in &elem.faces {
                let face = &self.mesh.faces[*face_idx];
                let neighbor_idx = if face.owner == i {
                    face.neighbor
                } else {
                    Some(face.owner)
                };

                let vof_face = if let Some(nb) = neighbor_idx {
                    0.5 * (self.vof[i] + self.vof[nb])  // Central difference
                } else {
                    self.vof[i]  // Boundary face
                };

                let area = face.area();
                let normal = face.normal();

                for d in 0..3 {
                    grad[d] += vof_face * normal[d] * area;
                }
            }

            let volume = elem.volume;
            for d in 0..3 {
                grad[d] /= volume;
            }

            self.vof_gradient[i] = grad;
        }

        // Similarly compute pressure gradient (used in momentum equation)
        for i in 0..self.mesh.elements.len() {
            let elem = &self.mesh.elements[i];
            let mut grad = [0.0, 0.0, 0.0];

            for face_idx in &elem.faces {
                let face = &self.mesh.faces[*face_idx];
                let neighbor_idx = if face.owner == i {
                    face.neighbor
                } else {
                    Some(face.owner)
                };

                let p_face = if let Some(nb) = neighbor_idx {
                    0.5 * (self.pressure[i] + self.pressure[nb])
                } else {
                    self.pressure[i]  // Boundary
                };

                let area = face.area();
                let normal = face.normal();

                for d in 0..3 {
                    grad[d] += p_face * normal[d] * area;
                }
            }

            let volume = elem.volume;
            for d in 0..3 {
                grad[d] /= volume;
            }

            self.pressure_gradient[i] = grad;
        }
    }

    fn reconstruct_interface(&mut self) {
        // PLIC: In interface cells (0 < α < 1), reconstruct interface as plane
        // Plane equation: n·(x - x₀) = 0, where n = -∇α / |∇α|
        // x₀ is positioned to match volume fraction α

        for i in 0..self.mesh.elements.len() {
            let alpha = self.vof[i];
            if alpha > 1e-6 && alpha < 1.0 - 1e-6 {
                // Interface cell
                let grad = self.vof_gradient[i];
                let grad_mag = (grad[0].powi(2) + grad[1].powi(2) + grad[2].powi(2)).sqrt();

                if grad_mag > 1e-9 {
                    // Normal pointing from liquid to gas
                    let normal = [
                        -grad[0] / grad_mag,
                        -grad[1] / grad_mag,
                        -grad[2] / grad_mag,
                    ];

                    // Store normal for later use in VOF advection
                    // (In full implementation, compute plane position to match α)
                    self.mesh.elements[i].interface_normal = Some(normal);
                }
            } else {
                self.mesh.elements[i].interface_normal = None;
            }
        }
    }

    fn solve_momentum_predictor(&mut self) -> Result<(), SolverError> {
        // Solve momentum equation: ∂(ρu)/∂t = -∇p + ∇·(μ∇u) + ρg
        // Predictor step: compute u* without enforcing continuity

        let mut u_star = self.velocity.clone();

        for i in 0..self.mesh.elements.len() {
            let alpha = self.vof[i];

            // Interpolate density and viscosity based on VOF
            let rho = alpha * self.alloy.density_liquid + (1.0 - alpha) * self.air_density;
            let mu = alpha * self.alloy.viscosity_liquid + (1.0 - alpha) * self.air_viscosity;

            // Gravity force
            let gravity = [0.0, -9.81, 0.0];  // TODO: use from process_params
            let f_gravity = [rho * gravity[0], rho * gravity[1], rho * gravity[2]];

            // Pressure gradient force
            let f_pressure = [
                -self.pressure_gradient[i][0],
                -self.pressure_gradient[i][1],
                -self.pressure_gradient[i][2],
            ];

            // Viscous diffusion (simplified — full implementation uses face gradients)
            let f_viscous = self.compute_viscous_force(i, mu);

            // Explicit Euler: u* = u_old + dt/ρ · (F_total)
            for d in 0..3 {
                u_star[i][d] = self.velocity[i][d]
                             + self.dt / rho * (f_gravity[d] + f_pressure[d] + f_viscous[d]);
            }
        }

        self.velocity = u_star;
        Ok(())
    }

    fn compute_viscous_force(&self, cell_idx: usize, mu: f32) -> [f32; 3] {
        // Compute ∇·(μ∇u) using finite volume discretization
        // Simplified: use Laplacian approximation
        let elem = &self.mesh.elements[cell_idx];
        let mut laplacian = [0.0, 0.0, 0.0];

        for face_idx in &elem.faces {
            let face = &self.mesh.faces[*face_idx];
            let neighbor_idx = if face.owner == cell_idx {
                face.neighbor
            } else {
                Some(face.owner)
            };

            if let Some(nb) = neighbor_idx {
                let dist = face.distance_between_cells(&self.mesh);
                let area = face.area();

                for d in 0..3 {
                    let du = self.velocity[nb][d] - self.velocity[cell_idx][d];
                    laplacian[d] += mu * area * du / dist;
                }
            }
        }

        let volume = elem.volume;
        [laplacian[0] / volume, laplacian[1] / volume, laplacian[2] / volume]
    }

    fn solve_pressure_correction(&mut self) -> Result<(), SolverError> {
        // Solve Poisson equation for pressure correction: ∇²p' = ρ/dt · ∇·u*
        // This enforces incompressibility: ∇·u = 0

        let n = self.mesh.elements.len();
        let mut matrix = sprs::TriMat::new((n, n));
        let mut rhs = vec![0.0; n];

        for i in 0..n {
            let elem = &self.mesh.elements[i];
            let alpha = self.vof[i];
            let rho = alpha * self.alloy.density_liquid + (1.0 - alpha) * self.air_density;

            // Compute divergence of u*
            let mut div_u = 0.0;
            for face_idx in &elem.faces {
                let face = &self.mesh.faces[*face_idx];
                let neighbor_idx = if face.owner == i { face.neighbor } else { Some(face.owner) };

                let u_face = if let Some(nb) = neighbor_idx {
                    [
                        0.5 * (self.velocity[i][0] + self.velocity[nb][0]),
                        0.5 * (self.velocity[i][1] + self.velocity[nb][1]),
                        0.5 * (self.velocity[i][2] + self.velocity[nb][2]),
                    ]
                } else {
                    self.velocity[i]  // Boundary
                };

                let normal = face.normal();
                let area = face.area();
                div_u += (u_face[0] * normal[0] + u_face[1] * normal[1] + u_face[2] * normal[2]) * area;
            }
            div_u /= elem.volume;

            rhs[i] = rho / self.dt * div_u;

            // Build Laplacian matrix for pressure correction
            let mut diag = 0.0;
            for face_idx in &elem.faces {
                let face = &self.mesh.faces[*face_idx];
                let neighbor_idx = if face.owner == i { face.neighbor } else { Some(face.owner) };

                if let Some(nb) = neighbor_idx {
                    let dist = face.distance_between_cells(&self.mesh);
                    let area = face.area();
                    let coeff = area / dist;

                    matrix.add_triplet(i, nb, -coeff);
                    diag += coeff;
                } else {
                    // Boundary: assume zero gradient (Neumann BC)
                    // No contribution to off-diagonal
                }
            }
            matrix.add_triplet(i, i, diag);
        }

        // Solve using conjugate gradient (via sprs crate)
        let csr = matrix.to_csr();
        let p_correction = sprs::linalg::cg::cg(&csr, &rhs, 1000, 1e-6)?;

        // Update pressure
        for i in 0..n {
            self.pressure[i] += p_correction[i];
        }

        Ok(())
    }

    fn correct_velocity(&mut self) {
        // Correct velocity: u_corrected = u* - dt/ρ · ∇p'
        for i in 0..self.mesh.elements.len() {
            let alpha = self.vof[i];
            let rho = alpha * self.alloy.density_liquid + (1.0 - alpha) * self.air_density;

            // Recompute pressure gradient after correction
            let p_grad = self.compute_pressure_gradient_at_cell(i);

            for d in 0..3 {
                self.velocity[i][d] -= self.dt / rho * p_grad[d];
            }
        }
    }

    fn advect_vof(&mut self) {
        // Advect VOF field using geometric advection for sharp interface
        // For each face, compute flux of α based on face velocity and interface reconstruction

        let mut vof_new = self.vof.clone();

        for i in 0..self.mesh.elements.len() {
            let elem = &self.mesh.elements[i];
            let mut flux_sum = 0.0;

            for face_idx in &elem.faces {
                let face = &self.mesh.faces[*face_idx];
                let neighbor_idx = if face.owner == i { face.neighbor } else { Some(face.owner) };

                // Face velocity (interpolated)
                let u_face = if let Some(nb) = neighbor_idx {
                    [
                        0.5 * (self.velocity[i][0] + self.velocity[nb][0]),
                        0.5 * (self.velocity[i][1] + self.velocity[nb][1]),
                        0.5 * (self.velocity[i][2] + self.velocity[nb][2]),
                    ]
                } else {
                    self.velocity[i]
                };

                let normal = face.normal();
                let area = face.area();
                let u_dot_n = u_face[0] * normal[0] + u_face[1] * normal[1] + u_face[2] * normal[2];

                // Upwind VOF value
                let alpha_face = if u_dot_n > 0.0 {
                    self.vof[i]
                } else if let Some(nb) = neighbor_idx {
                    self.vof[nb]
                } else {
                    0.0  // Inflow boundary: assume air
                };

                flux_sum += alpha_face * u_dot_n * area;
            }

            vof_new[i] = self.vof[i] - self.dt / elem.volume * flux_sum;
            vof_new[i] = vof_new[i].clamp(0.0, 1.0);  // Enforce bounds
        }

        self.vof = vof_new;
    }

    fn apply_boundary_conditions(&mut self) {
        // Apply boundary conditions (inlets, outlets, walls)
        // Simplified: walls have zero normal velocity (enforced in momentum solve)
        // Inlet: set α=1, u=u_inlet
        // Outlet: zero gradient (already handled in flux computation)

        for bc in &self.mesh.boundary_conditions {
            match bc.bc_type.as_str() {
                "inlet" => {
                    for &cell_idx in &bc.cell_indices {
                        self.vof[cell_idx] = 1.0;
                        self.velocity[cell_idx] = bc.inlet_velocity.unwrap();
                    }
                }
                "wall" => {
                    // No-slip: u = 0 (handled implicitly in viscous term)
                }
                "outlet" => {
                    // Zero gradient (natural BC in FVM)
                }
                _ => {}
            }
        }
    }

    fn compute_pressure_gradient_at_cell(&self, cell_idx: usize) -> [f32; 3] {
        // Helper to compute pressure gradient at a specific cell
        // (Similar to compute_gradients but for single cell)
        let elem = &self.mesh.elements[cell_idx];
        let mut grad = [0.0, 0.0, 0.0];

        for face_idx in &elem.faces {
            let face = &self.mesh.faces[*face_idx];
            let neighbor_idx = if face.owner == cell_idx {
                face.neighbor
            } else {
                Some(face.owner)
            };

            let p_face = if let Some(nb) = neighbor_idx {
                0.5 * (self.pressure[cell_idx] + self.pressure[nb])
            } else {
                self.pressure[cell_idx]
            };

            let area = face.area();
            let normal = face.normal();

            for d in 0..3 {
                grad[d] += p_face * normal[d] * area;
            }
        }

        let volume = elem.volume;
        [grad[0] / volume, grad[1] / volume, grad[2] / volume]
    }

    pub fn is_fill_complete(&self) -> bool {
        // Check if mold is filled (all cavity cells have α ≈ 1)
        let cavity_cells = self.mesh.get_cavity_cell_indices();
        let filled_cells = cavity_cells.iter()
            .filter(|&&i| self.vof[i] > 0.99)
            .count();

        filled_cells as f32 / cavity_cells.len() as f32 > 0.98
    }

    pub fn export_results(&self, path: &str) -> Result<(), std::io::Error> {
        // Export VOF, velocity, pressure to VTU format for visualization
        crate::io::write_vtu(
            path,
            &self.mesh,
            &self.vof,
            &self.velocity,
            &self.pressure,
        )
    }
}

#[derive(Debug)]
pub enum SolverError {
    Convergence(String),
    LinearSolve(String),
    InvalidMesh(String),
}
```

### 3. Solidification Solver (Rust — Enthalpy Method)

Solves transient heat conduction with latent heat source term for solidification analysis after filling completes.

```rust
// src/solver/solidification_solver.rs

use sprs::{CsMat, TriMat};
use crate::mesh::Mesh;
use crate::db::models::Alloy;

pub struct SolidificationSolver {
    mesh: Mesh,
    alloy: Alloy,
    mold_conductivity: f32,
    mold_specific_heat: f32,
    mold_density: f32,

    // Field variables
    enthalpy: Vec<f32>,      // Specific enthalpy (J/kg)
    temperature: Vec<f32>,   // Temperature (K)
    solid_fraction: Vec<f32>, // f_s ∈ [0, 1]

    // Thermal gradients and cooling rate (for Niyama)
    temp_gradient: Vec<[f32; 3]>,
    cooling_rate: Vec<f32>,  // dT/dt (K/s)
    niyama: Vec<f32>,        // Niyama criterion value

    time: f32,
    dt: f32,
}

impl SolidificationSolver {
    pub fn new(mesh: Mesh, alloy: Alloy, initial_temp: f32) -> Self {
        let n_cells = mesh.elements.len();

        // Compute initial enthalpy from temperature
        let initial_h = Self::temperature_to_enthalpy(&alloy, initial_temp);

        Self {
            mesh,
            alloy,
            mold_conductivity: 1.0,  // TODO: from mold material
            mold_specific_heat: 1000.0,
            mold_density: 1600.0,
            enthalpy: vec![initial_h; n_cells],
            temperature: vec![initial_temp; n_cells],
            solid_fraction: vec![0.0; n_cells],
            temp_gradient: vec![[0.0, 0.0, 0.0]; n_cells],
            cooling_rate: vec![0.0; n_cells],
            niyama: vec![f32::INFINITY; n_cells],
            time: 0.0,
            dt: 0.1,  // 0.1 second timestep
        }
    }

    pub fn solve_timestep(&mut self) -> Result<(), SolverError> {
        // Solve transient heat equation using implicit Euler
        // ∂(ρH)/∂t = ∇·(k∇T)

        // 1. Build linear system for enthalpy update
        let n = self.mesh.elements.len();
        let mut matrix = TriMat::new((n, n));
        let mut rhs = vec![0.0; n];

        for i in 0..n {
            let elem = &self.mesh.elements[i];
            let is_casting = elem.material == "casting";

            let (rho, cp, k) = if is_casting {
                let alpha = 1.0 - self.solid_fraction[i];  // Liquid fraction
                let rho = alpha * self.alloy.density_liquid
                        + (1.0 - alpha) * self.alloy.density_solid;
                let k = alpha * self.alloy.thermal_conductivity_liquid
                      + (1.0 - alpha) * self.alloy.thermal_conductivity_solid;
                let cp = alpha * self.alloy.specific_heat_liquid
                       + (1.0 - alpha) * self.alloy.specific_heat_solid;
                (rho, cp, k)
            } else {
                // Mold material
                (self.mold_density, self.mold_specific_heat, self.mold_conductivity)
            };

            let volume = elem.volume;
            let mut diag = rho * volume / self.dt;

            // Diffusion term: ∇·(k∇T)
            for face_idx in &elem.faces {
                let face = &self.mesh.faces[*face_idx];
                let neighbor_idx = if face.owner == i { face.neighbor } else { Some(face.owner) };

                if let Some(nb) = neighbor_idx {
                    let dist = face.distance_between_cells(&self.mesh);
                    let area = face.area();

                    // Harmonic average of conductivity across face
                    let k_nb = self.get_thermal_conductivity(nb);
                    let k_face = 2.0 * k * k_nb / (k + k_nb + 1e-12);

                    let coeff = k_face * area / dist;
                    matrix.add_triplet(i, nb, coeff);
                    diag += coeff;
                }
            }

            matrix.add_triplet(i, i, -diag);
            rhs[i] = -rho * volume / self.dt * self.enthalpy[i];
        }

        // Solve linear system: A·H_new = rhs
        let csr = matrix.to_csr();
        let h_new = Self::solve_linear_system(&csr, &rhs)?;

        // Store old temperature for cooling rate calculation
        let temp_old = self.temperature.clone();

        // Update enthalpy and derive temperature and solid fraction
        for i in 0..n {
            self.enthalpy[i] = h_new[i];
            self.temperature[i] = Self::enthalpy_to_temperature(&self.alloy, self.enthalpy[i]);
            self.solid_fraction[i] = Self::compute_solid_fraction(&self.alloy, self.temperature[i]);

            // Cooling rate
            self.cooling_rate[i] = (temp_old[i] - self.temperature[i]) / self.dt;
        }

        // 2. Compute temperature gradients
        self.compute_temperature_gradients();

        // 3. Compute Niyama criterion in mushy zone cells
        self.compute_niyama();

        self.time += self.dt;
        Ok(())
    }

    fn temperature_to_enthalpy(alloy: &Alloy, temp: f32) -> f32 {
        // H(T) = ∫c_p dT + f_s·L
        let t_ref = 298.0;  // Reference temperature (K)
        let cp = alloy.specific_heat_liquid;  // Simplified: constant c_p

        let sensible = cp * (temp - t_ref);
        let f_s = Self::compute_solid_fraction(alloy, temp);
        let latent = f_s * alloy.latent_heat;

        sensible + latent
    }

    fn enthalpy_to_temperature(alloy: &Alloy, enthalpy: f32) -> f32 {
        // Invert H(T) to get T (Newton-Raphson iteration)
        let mut temp = alloy.liquidus_temp;

        for _ in 0..10 {
            let h_guess = Self::temperature_to_enthalpy(alloy, temp);
            let dh_dt = alloy.specific_heat_liquid
                      + alloy.latent_heat * Self::df_s_dT(alloy, temp);

            temp -= (h_guess - enthalpy) / dh_dt;

            if (h_guess - enthalpy).abs() < 1.0 {
                break;
            }
        }

        temp
    }

    fn compute_solid_fraction(alloy: &Alloy, temp: f32) -> f32 {
        // Use Scheil solidification model from fraction_solid_curve
        // or simple lever rule approximation

        if temp >= alloy.liquidus_temp {
            0.0
        } else if temp <= alloy.solidus_temp {
            1.0
        } else {
            // Linear interpolation (lever rule)
            (alloy.liquidus_temp - temp) / (alloy.liquidus_temp - alloy.solidus_temp)
        }
    }

    fn df_s_dT(alloy: &Alloy, temp: f32) -> f32 {
        // Derivative of solid fraction w.r.t. temperature
        if temp > alloy.liquidus_temp || temp < alloy.solidus_temp {
            0.0
        } else {
            -1.0 / (alloy.liquidus_temp - alloy.solidus_temp)
        }
    }

    fn get_thermal_conductivity(&self, cell_idx: usize) -> f32 {
        let elem = &self.mesh.elements[cell_idx];
        if elem.material == "casting" {
            let alpha = 1.0 - self.solid_fraction[cell_idx];
            alpha * self.alloy.thermal_conductivity_liquid
                + (1.0 - alpha) * self.alloy.thermal_conductivity_solid
        } else {
            self.mold_conductivity
        }
    }

    fn compute_temperature_gradients(&mut self) {
        // Compute ∇T using Gauss divergence theorem
        for i in 0..self.mesh.elements.len() {
            let elem = &self.mesh.elements[i];
            let mut grad = [0.0, 0.0, 0.0];

            for face_idx in &elem.faces {
                let face = &self.mesh.faces[*face_idx];
                let neighbor_idx = if face.owner == i { face.neighbor } else { Some(face.owner) };

                let t_face = if let Some(nb) = neighbor_idx {
                    0.5 * (self.temperature[i] + self.temperature[nb])
                } else {
                    self.temperature[i]
                };

                let area = face.area();
                let normal = face.normal();

                for d in 0..3 {
                    grad[d] += t_face * normal[d] * area;
                }
            }

            let volume = elem.volume;
            for d in 0..3 {
                grad[d] /= volume;
            }

            self.temp_gradient[i] = grad;
        }
    }

    fn compute_niyama(&mut self) {
        // Niyama = G / sqrt(dT/dt), where G = |∇T|
        for i in 0..self.mesh.elements.len() {
            let fs = self.solid_fraction[i];

            // Only compute in mushy zone (0.01 < f_s < 0.99)
            if fs > 0.01 && fs < 0.99 {
                let grad = self.temp_gradient[i];
                let g_mag = (grad[0].powi(2) + grad[1].powi(2) + grad[2].powi(2)).sqrt();
                let cooling = self.cooling_rate[i].max(1e-6);  // Avoid division by zero

                // Convert to K·s^0.5/mm for comparison with literature thresholds
                self.niyama[i] = g_mag / cooling.sqrt() / 1000.0;  // K/m → K/mm
            } else {
                self.niyama[i] = f32::INFINITY;  // No porosity risk outside mushy zone
            }
        }
    }

    fn solve_linear_system(matrix: &CsMat<f32>, rhs: &[f32]) -> Result<Vec<f32>, SolverError> {
        // Solve using conjugate gradient or direct sparse solver
        // Simplified: use sprs::linalg::bicgstab
        let solution = sprs::linalg::bicgstab(matrix, rhs, 1000, 1e-6)
            .map_err(|e| SolverError::LinearSolve(e.to_string()))?;
        Ok(solution)
    }

    pub fn is_solidification_complete(&self) -> bool {
        // Check if all casting cells are fully solidified
        self.solid_fraction.iter()
            .enumerate()
            .filter(|(i, _)| self.mesh.elements[*i].material == "casting")
            .all(|(_, &fs)| fs > 0.99)
    }

    pub fn extract_porosity_locations(&self) -> Vec<PorosityLocation> {
        // Identify cells with Niyama below threshold
        let threshold = self.alloy.niyama_threshold.unwrap_or(1.0);

        self.niyama.iter()
            .enumerate()
            .filter(|(i, &ny)| {
                ny < threshold && self.mesh.elements[*i].material == "casting"
            })
            .map(|(i, &ny)| {
                let centroid = self.mesh.elements[i].centroid();
                PorosityLocation {
                    position: centroid,
                    niyama_value: ny,
                    solid_fraction: self.solid_fraction[i],
                }
            })
            .collect()
    }

    pub fn export_results(&self, path: &str) -> Result<(), std::io::Error> {
        crate::io::write_vtu(
            path,
            &self.mesh,
            &self.temperature,
            &self.solid_fraction,
            &self.niyama,
        )
    }
}

#[derive(Debug, Serialize)]
pub struct PorosityLocation {
    pub position: [f32; 3],
    pub niyama_value: f32,
    pub solid_fraction: f32,
}

#[derive(Debug)]
pub enum SolverError {
    LinearSolve(String),
    InvalidMesh(String),
}
```

### 4. Three.js Visualization Component (React/TypeScript)

Renders 3D mesh, fill animation, and temperature/porosity contours using Three.js with WebGL shaders.

```typescript
// frontend/src/components/SimulationViewer/SimulationViewer.tsx

import { useRef, useEffect, useState } from 'react';
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';

interface SimulationViewerProps {
  meshUrl: string;
  resultsUrl?: string;
  animationUrl?: string;
  visualizationType: 'fill' | 'temperature' | 'porosity' | 'solid_fraction';
}

export function SimulationViewer({
  meshUrl,
  resultsUrl,
  visualizationType,
}: SimulationViewerProps) {
  const containerRef = useRef<HTMLDivElement>(null);
  const sceneRef = useRef<THREE.Scene | null>(null);
  const rendererRef = useRef<THREE.WebGLRenderer | null>(null);
  const meshObjectRef = useRef<THREE.Mesh | null>(null);

  const [currentTime, setCurrentTime] = useState(0);
  const [maxTime, setMaxTime] = useState(100);
  const [isPlaying, setIsPlaying] = useState(false);

  // Initialize Three.js scene
  useEffect(() => {
    if (!containerRef.current) return;

    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x1a1a1a);
    sceneRef.current = scene;

    const camera = new THREE.PerspectiveCamera(
      60,
      containerRef.current.clientWidth / containerRef.current.clientHeight,
      0.1,
      1000
    );
    camera.position.set(0.5, 0.5, 1.5);

    const renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(containerRef.current.clientWidth, containerRef.current.clientHeight);
    containerRef.current.appendChild(renderer.domElement);
    rendererRef.current = renderer;

    const controls = new OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true;

    // Lighting
    const ambientLight = new THREE.AmbientLight(0xffffff, 0.5);
    scene.add(ambientLight);

    const directionalLight = new THREE.DirectionalLight(0xffffff, 0.7);
    directionalLight.position.set(5, 10, 7.5);
    scene.add(directionalLight);

    // Animation loop
    const animate = () => {
      requestAnimationFrame(animate);
      controls.update();
      renderer.render(scene, camera);
    };
    animate();

    return () => {
      renderer.dispose();
      containerRef.current?.removeChild(renderer.domElement);
    };
  }, []);

  // Load mesh and results
  useEffect(() => {
    if (!sceneRef.current || !meshUrl) return;

    const loadMeshAndResults = async () => {
      // Load VTU mesh
      const meshResponse = await fetch(meshUrl);
      const meshData = await meshResponse.arrayBuffer();
      const mesh = parseVTUMesh(meshData);

      // Create Three.js geometry
      const geometry = new THREE.BufferGeometry();
      geometry.setAttribute('position', new THREE.Float32BufferAttribute(mesh.vertices, 3));
      geometry.setIndex(mesh.indices);
      geometry.computeVertexNormals();

      // Load results if available
      let colorData: Float32Array | null = null;
      if (resultsUrl) {
        const resultsResponse = await fetch(resultsUrl);
        const resultsData = await resultsResponse.arrayBuffer();
        const results = parseVTUResults(resultsData);

        colorData = results[visualizationType];
        setMaxTime(results.timeSteps?.length || 100);
      }

      // Create material with custom shader for field visualization
      const material = new THREE.ShaderMaterial({
        vertexShader: `
          varying vec3 vNormal;
          varying float vFieldValue;
          attribute float fieldValue;

          void main() {
            vNormal = normalize(normalMatrix * normal);
            vFieldValue = fieldValue;
            gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
          }
        `,
        fragmentShader: `
          varying vec3 vNormal;
          varying float vFieldValue;
          uniform vec3 colorMin;
          uniform vec3 colorMax;
          uniform float valueMin;
          uniform float valueMax;

          void main() {
            // Color mapping (field value to color)
            float t = clamp((vFieldValue - valueMin) / (valueMax - valueMin), 0.0, 1.0);
            vec3 fieldColor = mix(colorMin, colorMax, t);

            // Simple lighting
            vec3 lightDir = normalize(vec3(5.0, 10.0, 7.5));
            float diffuse = max(dot(vNormal, lightDir), 0.0);
            vec3 finalColor = fieldColor * (0.5 + 0.5 * diffuse);

            gl_FragColor = vec4(finalColor, 1.0);
          }
        `,
        uniforms: {
          colorMin: { value: new THREE.Color(0x0000ff) },  // Blue (cold/empty)
          colorMax: { value: new THREE.Color(0xff0000) },  // Red (hot/filled)
          valueMin: { value: 0.0 },
          valueMax: { value: 1.0 },
        },
      });

      if (colorData) {
        geometry.setAttribute('fieldValue', new THREE.Float32BufferAttribute(colorData, 1));
        material.uniforms.valueMin.value = Math.min(...colorData);
        material.uniforms.valueMax.value = Math.max(...colorData);
      }

      // Set color scale based on visualization type
      if (visualizationType === 'fill') {
        material.uniforms.colorMin.value = new THREE.Color(0xcccccc);  // Air (gray)
        material.uniforms.colorMax.value = new THREE.Color(0xff8800);  // Metal (orange)
      } else if (visualizationType === 'temperature') {
        material.uniforms.colorMin.value = new THREE.Color(0x0000ff);  // Cold (blue)
        material.uniforms.colorMax.value = new THREE.Color(0xff0000);  // Hot (red)
      } else if (visualizationType === 'porosity') {
        material.uniforms.colorMin.value = new THREE.Color(0x00ff00);  // Sound (green)
        material.uniforms.colorMax.value = new THREE.Color(0xff0000);  // Porosity (red)
      }

      const meshObject = new THREE.Mesh(geometry, material);
      sceneRef.current.add(meshObject);
      meshObjectRef.current = meshObject;

      // Center and scale mesh to fit view
      geometry.computeBoundingBox();
      const bbox = geometry.boundingBox!;
      const center = new THREE.Vector3();
      bbox.getCenter(center);
      meshObject.position.sub(center);

      const size = new THREE.Vector3();
      bbox.getSize(size);
      const maxDim = Math.max(size.x, size.y, size.z);
      meshObject.scale.setScalar(1.0 / maxDim);
    };

    loadMeshAndResults();
  }, [meshUrl, resultsUrl, visualizationType]);

  // Animation playback
  useEffect(() => {
    if (!isPlaying || !meshObjectRef.current) return;

    const interval = setInterval(() => {
      setCurrentTime((prev) => {
        const next = prev + 1;
        if (next >= maxTime) {
          setIsPlaying(false);
          return maxTime - 1;
        }

        // Update field values for current timestep
        updateFieldValuesForTime(meshObjectRef.current!, next);
        return next;
      });
    }, 50);  // 20 FPS

    return () => clearInterval(interval);
  }, [isPlaying, maxTime]);

  return (
    <div className="simulation-viewer flex flex-col h-full">
      <div className="viewer-controls bg-gray-800 p-4 flex gap-4 items-center">
        <button
          onClick={() => setIsPlaying(!isPlaying)}
          className="px-4 py-2 bg-blue-600 rounded hover:bg-blue-700"
        >
          {isPlaying ? 'Pause' : 'Play'}
        </button>

        <input
          type="range"
          min="0"
          max={maxTime - 1}
          value={currentTime}
          onChange={(e) => setCurrentTime(Number(e.target.value))}
          className="flex-1"
        />

        <span className="text-white">
          Time: {currentTime.toFixed(1)}s / {maxTime.toFixed(1)}s
        </span>

        <select
          value={visualizationType}
          className="px-3 py-2 bg-gray-700 rounded text-white"
          disabled
        >
          <option value="fill">Fill</option>
          <option value="temperature">Temperature</option>
          <option value="porosity">Porosity (Niyama)</option>
          <option value="solid_fraction">Solid Fraction</option>
        </select>
      </div>

      <div ref={containerRef} className="flex-1" />

      <ColorLegend
        type={visualizationType}
        min={0}
        max={1}
      />
    </div>
  );
}

function parseVTUMesh(buffer: ArrayBuffer): { vertices: number[]; indices: number[] } {
  // Parse VTU (VTK XML Unstructured Grid) format
  // Simplified: assumes ASCII encoding for demo
  const text = new TextDecoder().decode(buffer);

  // Extract vertices and connectivity
  // (Full implementation would parse XML properly)
  const vertices: number[] = [];
  const indices: number[] = [];

  // TODO: Implement proper VTU parser

  return { vertices, indices };
}

function parseVTUResults(buffer: ArrayBuffer): Record<string, Float32Array> {
  // Parse VTU results file with time-series field data
  // Returns field arrays for each visualization type

  return {
    fill: new Float32Array(),
    temperature: new Float32Array(),
    porosity: new Float32Array(),
    solid_fraction: new Float32Array(),
  };
}

function updateFieldValuesForTime(mesh: THREE.Mesh, timeStep: number) {
  // Update vertex attribute for current timestep
  // (Field data would be loaded from time-series results)
}

function ColorLegend({ type, min, max }: { type: string; min: number; max: number }) {
  return (
    <div className="color-legend absolute bottom-4 right-4 bg-gray-800 p-3 rounded">
      <div className="text-white text-sm mb-2">{type}</div>
      <div className="flex gap-2">
        <div className="w-48 h-6 bg-gradient-to-r from-blue-500 to-red-500" />
        <div className="text-white text-xs">
          <div>{max.toFixed(2)}</div>
          <div>{min.toFixed(2)}</div>
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
cargo init castsim-api
cd castsim-api
cargo add axum tokio serde serde_json sqlx uuid chrono tower-http jsonwebtoken bcrypt
cargo add --features postgres,runtime-tokio-rustls,uuid,chrono,json sqlx
cargo add aws-sdk-s3 redis nalgebra ndarray sprs
```
- `src/main.rs` — Axum app with CORS, tracing, graceful shutdown
- `src/config.rs` — Environment config (DATABASE_URL, JWT_SECRET, S3_BUCKET, REDIS_URL)
- `src/state.rs` — AppState struct (PgPool, Redis, S3Client)
- `src/error.rs` — ApiError enum with IntoResponse
- `Dockerfile` — Multi-stage build
- `docker-compose.yml` — PostgreSQL, Redis, MinIO

**Day 2: Database schema and migrations**
- `migrations/001_initial.sql` — All tables: users, orgs, projects, simulations, jobs, alloys, mold_materials, mesh_jobs, usage
- `src/db/mod.rs` — Pool initialization
- `src/db/models.rs` — All SQLx structs
- Run `sqlx migrate run`
- Seed 10 alloys with thermophysical properties

**Day 3: Authentication system**
- `src/auth/mod.rs` — JWT middleware
- `src/auth/oauth.rs` — Google/GitHub OAuth handlers
- `src/api/handlers/auth.rs` — Register, login, OAuth callback, refresh, me
- Password hashing with bcrypt
- JWT with 24h access + 30d refresh

**Day 4: Project and mesh CRUD**
- `src/api/handlers/projects.rs` — Create, list, get, update, delete project
- `src/api/handlers/geometry.rs` — Upload STEP/STL, download geometry
- `src/api/handlers/orgs.rs` — Org CRUD, member management
- `src/api/router.rs` — Route definitions
- Integration tests: auth flow, project CRUD

### Phase 2 — Meshing Pipeline (Days 5–8)

**Day 5: TetGen integration via FFI**
- `src/mesh/tetgen_bindings.rs` — Rust FFI bindings to TetGen C++ library
- `src/mesh/mesh_types.rs` — Mesh struct (vertices, elements, faces, boundary conditions)
- `build.rs` — Compile TetGen library
- Unit test: mesh simple cube geometry

**Day 6: Mesh generation worker**
- `src/workers/mesh_worker.rs` — Redis job consumer for mesh generation
- Parse STL: triangle soup → closed surface
- Parse STEP: via opencascade.rs bindings (or fallback to STL export)
- TetGen invocation with quality params
- Mesh quality metrics: min/max angles, aspect ratio, Jacobian

**Day 7: Mesh API endpoints**
- `src/api/handlers/mesh.rs` — Create mesh job, get mesh job status, download mesh
- Mesh visualization: export VTU format for Three.js
- Mesh refinement regions: user-defined bounding boxes for local refinement
- Auto-refinement near mold surfaces (detect from boundary)

**Day 8: Mesh validation and storage**
- Validate mesh: no inverted elements, no isolated nodes
- Assign material tags: identify casting vs. mold regions from geometry
- Store mesh to S3 in VTU format
- Generate mesh thumbnail (wireframe screenshot)

### Phase 3 — Filling Solver Prototype (Days 9–14)

**Day 9: VOF solver framework**
- `src/solver/vof_solver.rs` — VofSolver struct, timestep loop
- `src/solver/mesh_io.rs` — Load VTU mesh from bytes
- Field storage: VOF, velocity, pressure, temperature
- CFL timestep calculation

**Day 10: Gradient and PLIC reconstruction**
- `compute_gradients()` — Gauss divergence for VOF gradient
- `reconstruct_interface()` — PLIC normal computation
- Unit test: gradient of linear function on structured mesh

**Day 11: Momentum equation (predictor)**
- `solve_momentum_predictor()` — Explicit Euler for momentum
- Density/viscosity interpolation from VOF
- Gravity force, pressure gradient force
- Viscous diffusion (Laplacian approximation)

**Day 12: Pressure correction (Poisson solver)**
- `solve_pressure_correction()` — Build Laplacian matrix
- Solve with sprs::linalg::cg (conjugate gradient)
- Test: pressure Poisson on simple domain

**Day 13: VOF advection and boundary conditions**
- `advect_vof()` — Geometric advection with upwinding
- `apply_boundary_conditions()` — Inlet, outlet, wall BCs
- Clamp VOF to [0,1]

**Day 14: Filling solver validation**
- Test 1: Dam break (gravity-driven flow) — compare to analytical front position
- Test 2: Simple mold filling (rectangular cavity) — check fill time
- Export VTU time series for visualization

### Phase 4 — Solidification Solver (Days 15–19)

**Day 15: Enthalpy solver framework**
- `src/solver/solidification_solver.rs` — SolidificationSolver struct
- Enthalpy-temperature conversion functions
- Solid fraction from Scheil curve (lever rule approximation)

**Day 16: Transient heat conduction solver**
- `solve_timestep()` — Implicit Euler for heat equation
- Build sparse matrix (diffusion term)
- Solve linear system with bicgstab

**Day 17: Latent heat source term**
- Incorporate latent heat in enthalpy update
- Test: Stefan problem (1D phase change) — compare to analytical

**Day 18: Niyama criterion calculation**
- `compute_temperature_gradients()` — Gauss divergence for ∇T
- `compute_niyama()` — G / √(dT/dt) in mushy zone
- `extract_porosity_locations()` — Find cells below threshold

**Day 19: Solidification validation**
- Test: Sand casting benchmark (simple plate) — compare solidification time to Chvorinov's rule
- Test: Niyama distribution — verify hot spots match expected riser locations

### Phase 5 — Frontend Visualization (Days 20–24)

**Day 20: Frontend scaffold**
```bash
npm create vite@latest frontend -- --template react-ts
cd frontend
npm i zustand @tanstack/react-query axios three @types/three
```
- `src/App.tsx` — Router, auth context, layout
- `src/stores/projectStore.ts` — Project state
- `src/stores/simulationStore.ts` — Simulation state
- `src/lib/api.ts` — Axios API client

**Day 21: Geometry upload UI**
- `src/pages/ProjectEditor.tsx` — Project page with geometry upload
- `src/components/GeometryUpload.tsx` — Drag-and-drop STEP/STL upload
- `src/components/MeshControls.tsx` — Mesh generation params, quality display

**Day 22: Three.js simulation viewer**
- `src/components/SimulationViewer/SimulationViewer.tsx` — Three.js scene setup
- `src/components/SimulationViewer/MeshLoader.tsx` — VTU parser
- OrbitControls, lighting, material setup
- Display static mesh

**Day 23: Field visualization shaders**
- Vertex shader: pass field value attribute
- Fragment shader: color mapping (blue→red scale)
- Support fill, temperature, porosity, solid fraction
- Color legend component

**Day 24: Animation playback**
- Time slider and play/pause controls
- Load time-series VTU results
- Update vertex attributes per frame
- Export animation as MP4 (server-side ffmpeg)

### Phase 6 — API + Job Orchestration (Days 25–29)

**Day 25: Simulation API endpoints**
- `src/api/handlers/simulation.rs` — Create simulation, get results
- Mesh validation and plan limits enforcement
- Enqueue job to Redis

**Day 26: Simulation worker (filling)**
- `src/workers/simulation_worker.rs` — Job consumer
- Load mesh, alloy, params
- Run VofSolver timestep loop
- Progress updates via WebSocket
- Export results to S3

**Day 27: Simulation worker (solidification)**
- Run SolidificationSolver after filling completes
- Compute Niyama field
- Extract porosity locations
- Store results and summary

**Day 28: WebSocket progress streaming**
- `src/api/ws/mod.rs` — WebSocket handler
- `src/api/ws/simulation_progress.rs` — Progress subscription
- Frontend: `useSimulationProgress` hook
- Live progress bar

**Day 29: Results download and caching**
- Presigned S3 URLs for results
- Redis caching for hot results (TTL 1h)
- Result summary API: porosity count, max Niyama, fill time

### Phase 7 — Billing + Plan Enforcement (Days 30–33)

**Day 30: Stripe integration**
- `src/api/handlers/billing.rs` — Checkout, portal, subscription status
- `src/api/handlers/webhooks/stripe.rs` — Webhook handler
- Plan mapping: Free (100K elements), Pro ($249/mo, 500K), Advanced ($499/mo, 5M)

**Day 31: Usage tracking**
- `src/middleware/plan_limits.rs` — Check element count before sim
- `src/services/usage.rs` — Track compute hours
- Usage dashboard API

**Day 32: Billing UI**
- `frontend/src/pages/Billing.tsx` — Plan comparison, usage meter
- Upgrade/downgrade via Customer Portal
- Upgrade prompts on limit hit

**Day 33: Feature gating**
- Gate mesh refinement behind Pro
- Gate optimization (post-MVP) behind Advanced
- Gate API access behind Advanced

### Phase 8 — Validation + Testing + Deployment (Days 34–42)

**Day 34: Solver validation — filling**
- Benchmark 1: Dam break — match analytical
- Benchmark 2: Simple cavity fill time
- Compare to FLOW-3D reference cases

**Day 35: Solver validation — solidification**
- Benchmark 1: Stefan problem
- Benchmark 2: Chvorinov's rule validation
- Benchmark 3: Niyama threshold calibration for A356

**Day 36: Integration testing**
- E2E: upload geometry → mesh → simulate → visualize
- API tests: all endpoints with auth
- Concurrent simulation jobs

**Day 37: Performance optimization**
- Profile filling solver: target 50K elements in <5min
- Profile solidification: 100K elements in <10min
- Frontend: 100K element mesh renders at 60 FPS

**Day 38: Docker and Kubernetes**
- `Dockerfile` — Multi-stage Rust build
- `k8s/` — Deployments for API, workers, PostgreSQL, Redis
- Health checks
- HPA for workers based on queue depth

**Day 39: Monitoring and logging**
- Prometheus metrics: sim duration, convergence rate
- Grafana dashboards
- Sentry error tracking
- Structured logging

**Day 40: CDN and asset delivery**
- CloudFront for frontend, meshes, results
- Mesh preloading optimization
- Compression (gzip VTU)

**Day 41: Security and polish**
- Rate limiting
- SQL injection audit (SQLx prevents)
- CORS config
- UI polish: loading states, errors, responsive

**Day 42: Launch prep**
- Production smoke test
- Database backup/restore
- Landing page
- Documentation
- Deploy and announce

---

## Critical Files

```
castsim/
├── castsim-api/                          # Rust backend
│   ├── Cargo.toml
│   ├── build.rs                          # TetGen compilation
│   ├── src/
│   │   ├── main.rs
│   │   ├── config.rs
│   │   ├── state.rs
│   │   ├── error.rs
│   │   ├── auth/
│   │   │   ├── mod.rs
│   │   │   └── oauth.rs
│   │   ├── api/
│   │   │   ├── router.rs
│   │   │   ├── handlers/
│   │   │   │   ├── auth.rs
│   │   │   │   ├── projects.rs
│   │   │   │   ├── geometry.rs
│   │   │   │   ├── mesh.rs
│   │   │   │   ├── simulation.rs
│   │   │   │   ├── billing.rs
│   │   │   │   └── webhooks/stripe.rs
│   │   │   └── ws/
│   │   │       ├── mod.rs
│   │   │       └── simulation_progress.rs
│   │   ├── middleware/
│   │   │   └── plan_limits.rs
│   │   ├── services/
│   │   │   ├── usage.rs
│   │   │   └── s3.rs
│   │   ├── workers/
│   │   │   ├── mod.rs
│   │   │   ├── mesh_worker.rs
│   │   │   └── simulation_worker.rs
│   │   ├── db/
│   │   │   ├── mod.rs
│   │   │   └── models.rs
│   │   ├── mesh/
│   │   │   ├── mod.rs
│   │   │   ├── tetgen_bindings.rs
│   │   │   ├── mesh_types.rs
│   │   │   └── mesh_io.rs
│   │   └── solver/
│   │       ├── mod.rs
│   │       ├── vof_solver.rs              # VOF filling solver
│   │       ├── solidification_solver.rs   # Enthalpy solidification
│   │       └── mesh_io.rs
│   ├── migrations/
│   │   └── 001_initial.sql
│   └── tests/
│       ├── api_integration.rs
│       └── solver_validation.rs
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
│   │   │   └── simulationStore.ts
│   │   ├── hooks/
│   │   │   └── useSimulationProgress.ts
│   │   ├── lib/
│   │   │   └── api.ts
│   │   ├── pages/
│   │   │   ├── Dashboard.tsx
│   │   │   ├── ProjectEditor.tsx
│   │   │   ├── Billing.tsx
│   │   │   ├── Login.tsx
│   │   │   └── Register.tsx
│   │   └── components/
│   │       ├── GeometryUpload.tsx
│   │       ├── MeshControls.tsx
│   │       ├── SimulationViewer/
│   │       │   ├── SimulationViewer.tsx
│   │       │   ├── MeshLoader.tsx
│   │       │   └── ColorLegend.tsx
│   │       ├── billing/
│   │       │   ├── PlanCard.tsx
│   │       │   └── UsageMeter.tsx
│   │       └── common/
│   │           └── SimulationToolbar.tsx
│
├── material-service/                      # Python FastAPI
│   ├── requirements.txt
│   ├── main.py
│   ├── alloy_database.py                  # Thermophysical properties
│   └── Dockerfile
│
├── k8s/
│   ├── api-deployment.yaml
│   ├── worker-deployment.yaml
│   ├── postgres-statefulset.yaml
│   ├── redis-deployment.yaml
│   └── ingress.yaml
│
├── docker-compose.yml
└── Cargo.toml
```

---

## Solver Validation Suite

### Benchmark 1: Dam Break (VOF Filling Accuracy)

**Setup:** 2D dam break in rectangular domain, 10cm x 5cm water column released at t=0

**Expected:** Free surface position at t=0.5s matches analytical shallow water solution

**Metric:** Free surface position error < **5%** at t=0.5s

**Tolerance:** VOF interface tracking within 2 cell widths of reference solution

### Benchmark 2: Simple Cavity Fill Time

**Setup:** Rectangular cavity 10cm x 10cm x 2cm, bottom inlet 1cm², A356 alloy at 973K, fill rate 50 cm³/s

**Expected:** Fill time = Volume / FlowRate = (10×10×2) / 50 = **4.0 seconds**

**Metric:** Simulated fill time = **4.0 ± 0.2 seconds** (95% cavity filled)

**Tolerance:** < 5% deviation from analytical fill time

### Benchmark 3: Stefan Problem (Solidification Phase Change)

**Setup:** 1D semi-infinite solidification, initial temp 973K (A356 liquidus), boundary at 300K

**Expected:** Solidification front position: s(t) = 2·λ·√(α·t), where λ from Stefan number

**Metric:** Front position at t=10s matches analytical **within 3%**

**Tolerance:** Temperature profile RMSE < 5K in solidified region

### Benchmark 4: Chvorinov's Rule (Solidification Time Scaling)

**Setup:** Simple plate castings of varying thickness (10mm, 20mm, 40mm), A356 in sand mold

**Expected:** Solidification time t_s ∝ (V/A)² (Chvorinov's rule)

**Metric:** t_s,20mm / t_s,10mm = **4.0 ± 0.3** (thickness ratio squared)

**Tolerance:** Solidification time scaling exponent n = 1.9 - 2.1 (Chvorinov: n=2)

### Benchmark 5: Niyama Threshold Calibration (Porosity Prediction)

**Setup:** A356 plate casting with known porosity locations from X-ray inspection

**Expected:** Porosity locations correlate with Niyama < **1.5 K·s^0.5/mm** (literature value for A356)

**Metric:** Porosity detection: True positive rate > **80%**, False positive rate < **20%**

**Tolerance:** Niyama threshold calibration ±0.3 K·s^0.5/mm based on experimental validation

---

## Verification

### Integration Testing Checklist

1. **Auth flow** — Register → login → JWT → authorized calls → refresh → logout
2. **Project CRUD** — Create → upload geometry → save → reload → verify preserved
3. **Mesh generation** — Upload STL → trigger mesh job → wait for completion → download VTU
4. **Mesh validation** — Load mesh → check quality metrics → verify no inverted elements
5. **Simulation submission** — Select alloy → set params → submit → job queued
6. **Simulation execution** — Worker picks job → filling solver runs → results to S3
7. **Solidification** — Filling completes → solidification solver runs → Niyama computed
8. **Results visualization** — Load VTU results → render mesh → animate fill → display porosity
9. **WebSocket progress** — Connect → subscribe to sim → receive progress updates → completion notification
10. **Plan limits** — Free user → 150K element mesh → blocked with upgrade prompt
11. **Billing** — Subscribe to Pro → Stripe checkout → webhook → limits lifted → 500K mesh allowed
12. **Concurrent sims** — 5 users submit jobs → all execute in parallel → all complete correctly
13. **Porosity detection** — Solidification completes → porosity locations extracted → displayed in UI
14. **Error handling** — Invalid mesh → clear error message → no crash
15. **Multi-timestep** — Filling animation plays → smooth frame transitions → correct velocities

### SQL Verification Queries

```sql
-- 1. Simulation success rate by plan tier
SELECT
    u.plan,
    COUNT(*) as total_sims,
    SUM(CASE WHEN s.status = 'completed' THEN 1 ELSE 0 END) as completed,
    AVG(s.wall_time_seconds) as avg_wall_time_s
FROM simulations s
JOIN users u ON s.user_id = u.id
WHERE s.created_at > NOW() - INTERVAL '30 days'
GROUP BY u.plan;

-- 2. Average mesh quality by geometry complexity
SELECT
    p.id,
    p.name,
    (p.mesh_stats->>'elements')::int as element_count,
    (p.mesh_stats->>'min_quality')::float as min_quality,
    (p.mesh_stats->>'max_aspect_ratio')::float as max_aspect_ratio
FROM projects p
WHERE p.mesh_stats IS NOT NULL
ORDER BY element_count DESC
LIMIT 20;

-- 3. Porosity detection rate (sims with porosity findings)
SELECT
    a.family as alloy_family,
    COUNT(*) as total_solidification_sims,
    SUM(CASE WHEN s.porosity_data IS NOT NULL THEN 1 ELSE 0 END) as sims_with_porosity,
    AVG((s.porosity_data->'porosity_locations')::jsonb->0->>'niyama_value')::float as avg_min_niyama
FROM simulations s
JOIN alloys a ON s.alloy_id = a.id
WHERE s.simulation_type IN ('solidification', 'coupled')
  AND s.status = 'completed'
GROUP BY a.family;

-- 4. Compute cost by user
SELECT
    u.email,
    u.plan,
    SUM(s.compute_hours) as total_compute_hours,
    COUNT(*) as simulation_count,
    AVG(s.compute_hours) as avg_compute_hours_per_sim
FROM simulations s
JOIN users u ON s.user_id = u.id
WHERE s.created_at > NOW() - INTERVAL '30 days'
GROUP BY u.id, u.email, u.plan
ORDER BY total_compute_hours DESC;

-- 5. Worker performance
SELECT
    sj.instance_type,
    COUNT(*) as jobs_completed,
    AVG(EXTRACT(EPOCH FROM (sj.completed_at - sj.started_at))) as avg_execution_time_s,
    MAX(EXTRACT(EPOCH FROM (sj.completed_at - sj.started_at))) as max_execution_time_s
FROM simulation_jobs sj
WHERE sj.completed_at IS NOT NULL
  AND sj.started_at IS NOT NULL
GROUP BY sj.instance_type;
```

---

## Deployment Architecture

### Production Infrastructure

```
┌─────────────────────────────────────────────────────────────┐
│                     CloudFront (CDN)                         │
│  - Frontend assets (React SPA)                               │
│  - Mesh files (VTU, compressed)                              │
│  - Results files (cached, presigned URLs)                    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              Application Load Balancer (ALB)                 │
│  - TLS termination                                           │
│  - Path-based routing (/api → API, /ws → WebSocket)         │
└─────────────────────────────────────────────────────────────┘
       │                                          │
       ▼                                          ▼
┌──────────────────────┐              ┌────────────────────────┐
│   API Pods (EKS)     │              │  Worker Pods (EKS)     │
│   3 replicas (HPA)   │              │  Auto-scale 1-10       │
│   - Axum API server  │              │  - Mesh workers        │
│   - Auth handlers    │              │  - Simulation workers  │
│   - WebSocket server │              │  c5.4xlarge spot       │
│   c5.large instances │              │  g4dn.xlarge (filling) │
└──────────────────────┘              └────────────────────────┘
       │                                          │
       └──────────────┬───────────────────────────┘
                      ▼
       ┌──────────────────────────────────┐
       │    Redis (ElastiCache)           │
       │    - Job queue                   │
       │    - WebSocket pub/sub           │
       │    - Result caching (hot data)   │
       └──────────────────────────────────┘
                      │
       ┌──────────────┴───────────────────┐
       ▼                                  ▼
┌─────────────────┐          ┌────────────────────────┐
│  PostgreSQL     │          │    S3 Buckets          │
│  (RDS)          │          │  - castsim-geometries  │
│  db.r5.xlarge   │          │  - castsim-meshes      │
│  Multi-AZ       │          │  - castsim-results     │
│  Auto backup    │          │  Lifecycle: 90d → Glacier│
└─────────────────┘          └────────────────────────┘

Monitoring Stack:
┌────────────────────────────────────────┐
│  Prometheus + Grafana (EKS)            │
│  - API latency, throughput             │
│  - Solver convergence metrics          │
│  - Queue depth, worker utilization     │
│                                        │
│  Sentry (SaaS)                         │
│  - Frontend/backend error tracking     │
│  - Performance monitoring              │
└────────────────────────────────────────┘
```

### Cost Estimation (Monthly, at 1000 active users)

| Component | Spec | Monthly Cost |
|-----------|------|--------------|
| EKS Control Plane | 1 cluster | $73 |
| API Pods | 3× c5.large (reserved) | $120 |
| Worker Pods | Avg 5× c5.4xlarge spot (50% util) | $600 |
| PostgreSQL RDS | db.r5.xlarge Multi-AZ | $580 |
| ElastiCache Redis | cache.r5.large | $200 |
| S3 Storage | 2TB (geometries, meshes, results) | $50 |
| S3 Data Transfer | 500GB/mo egress | $45 |
| CloudFront | 2TB data transfer | $170 |
| ALB | 1 load balancer, 10GB processed | $30 |
| **Total Infrastructure** | | **~$1,870/mo** |

Revenue at 1000 users (10% Pro, 2% Advanced):
- 880 Free: $0
- 100 Pro ($249): $24,900
- 20 Advanced ($499): $9,980
- **Total MRR: $34,880**
- **Gross Margin: 95%**

---

## Post-MVP Roadmap

### v1.1 — Thermal Stress and Distortion (Weeks 13-16)

**Goals:** Predict hot tearing, residual stress, and casting distortion

**Features:**
- Thermo-elastic-plastic FEM solver coupled to solidification
- Hot tearing prediction: Clyne-Davies criterion, RDG criterion
- Residual stress mapping after cooldown to room temperature
- Distortion prediction: spring-back after mold removal
- Pattern allowance calculator from distortion results

**Tech:** Rust FEM library (custom or fenris), contact mechanics for casting-mold interface

**Validation:** Compare residual stress to neutron diffraction measurements, distortion to CMM scans

### v1.2 — Microstructure Prediction (Weeks 17-20)

**Goals:** Predict grain structure, dendrite arm spacing, mechanical properties

**Features:**
- Dendrite arm spacing (DAS) from cooling rate: DAS = C₁ · (dT/dt)^(-n)
- Cast iron: graphite morphology (flake/compacted/nodular) based on cooling rate and composition
- Aluminum: eutectic Si morphology, grain refiner effectiveness
- Steel: as-cast grain size, carbide formation
- Mechanical property mapping: yield strength, hardness, elongation from microstructure
- Property degradation in porosity regions

**Tech:** Python service with metallurgical models, integration with Thermo-Calc for phase diagrams

**Validation:** Compare DAS to metallography, hardness to microhardness testing

### v1.3 — Die Casting Specific Features (Weeks 21-24)

**Goals:** Support high-pressure die casting with shot sleeve, venting, thermal cycling

**Features:**
- Shot sleeve dynamics: slow shot, fast shot, intensification phases
- Plunger motion profiles: user-defined velocity curves
- Air entrapment and vent sizing: predict blow holes
- Overflow and runner optimization for multi-cavity dies
- Die thermal cycling: simulate 100-1000 cycles for die thermal fatigue
- Die cooling channels: optimize layout for uniform die temperature
- Ejection force prediction

**Tech:** Enhanced VOF solver with moving boundary (plunger), thermal cycling loop

**Validation:** Compare to industry die casting trials (porosity location, die temperatures)

### v1.4 — Gating and Riser Optimization (Weeks 25-28)

**Goals:** AI-assisted gating design, automated riser placement

**Features:**
- Genetic algorithm for riser optimization: minimize riser volume while eliminating porosity
- Parametric gating system design: runner balancing, ingate sizing
- Design of experiments (DOE): multi-parameter sensitivity analysis
- Response surface for pouring temperature, speed, riser size
- ML-based gating suggestion: neural network trained on 10K+ successful designs
- Automatic overflow/vent placement for die casting

**Tech:** Python optimization service (DEAP for genetic algorithms, scikit-learn for ML)

**Validation:** Compare optimized designs to experienced foundry engineer designs, scrap rate reduction

### v1.5 — Continuous Casting Module (Weeks 29-32)

**Goals:** Support continuous casting of steel, aluminum billets

**Features:**
- Strand solidification with moving domain
- Shell thickness prediction, breakout risk
- Mold oscillation effects
- Secondary cooling spray zones
- Segregation and centerline porosity prediction

**Tech:** Extend solidification solver with moving boundary, Darcy flow in mushy zone

**Validation:** Compare to plant data (shell thickness, segregation profiles)

### v2.0 — Enterprise Features (Weeks 33-40)

**Goals:** On-premise deployment, custom alloy development, digital twin integration

**Features:**
- On-premise Kubernetes deployment for IP-sensitive foundries
- Custom alloy characterization: upload DSC, viscosity, thermal diffusivity data
- MES/ERP integration: import production schedules, export simulation results
- Digital twin: real-time process monitoring linked to simulation model
- Batch simulation: run 100+ parameter variations overnight
- API for automation and integration with CAD/CAM systems

**Tech:** Helm charts for on-prem deployment, REST API with rate limiting, webhook callbacks

**Target:** Large OEMs (automotive, aerospace), foundry equipment vendors
