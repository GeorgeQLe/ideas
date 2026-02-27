# 79. FireSim — Fire Dynamics, Smoke Spread, and Evacuation Simulation Platform

## Implementation Plan

**MVP Scope:** Cloud-based fire and evacuation simulation platform with browser-based 3D geometry builder using CAD/DXF import with extrusion to 3D meshes, custom Large Eddy Simulation (LES) CFD fire solver in Rust with combustion (mixture fraction model, gray gas radiation via finite volume method, soot transport for smoke visibility), prescribed heat release rate (HRR) fire curves (t-squared, NFPA design fires) and material-driven pyrolysis model for fire spread on surfaces, smoke detector and sprinkler RTI activation models with spray suppression, agent-based egress simulation with density-dependent movement speed (Fruin/SFPE correlations), pre-movement time distributions, visibility-impaired wayfinding from CFD soot coupling, FED (Fractional Effective Dose) toxicity accumulation for CO/HCN/heat, ASET vs. RSET tenability analysis with safety factors per SFPE/BS 7974, steel member temperature calculation via lumped-mass method (EN 1993-1-2) and 2D FEM heat transfer with ISO 834/natural fire exposure from CFD gas temperatures, Three.js volumetric smoke rendering with isosurface/slice plots for temperature/visibility/toxicity, egress animation with congestion heatmaps, 10 design fire scenarios (office, retail, residential, atrium, parking), PDF fire engineering report generation with ASET/RSET summary and code compliance matrix for NFPA 101/IBC, Stripe billing with Free (zone model only, single room, 100 occupants) / Pro $249/mo (CFD 500K cells, full egress, steel temps, 500 compute hours) / Advanced $499/mo (multi-mesh unlimited cells, pyrolysis, smoke control, structural capacity, multi-scenario, BIM/IFC import, API).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| CFD Fire Solver | Rust + CUDA | LES solver with combustion, radiation, soot transport — GPU-accelerated |
| Egress Solver | Rust (native) | Agent-based model, pathfinding (A*), density-speed, FED accumulation |
| Structural Thermal | Rust (native) | 2D FEM heat transfer, lumped-mass steel/concrete models |
| Detection/Suppression | Rust (native) | RTI models embedded in CFD solver, spray droplet transport |
| Scenario Engine | Python 3.12 (FastAPI) | Design fire library, fuel load database, code compliance checks, BIM/IFC parsing |
| Database | PostgreSQL 16 | Projects, users, simulations, design fires, material properties, fuel loads |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | Meshes, CFD results (field data: temperature, soot, velocity), egress animations |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| 3D Viewer | Three.js (r165) | Volumetric smoke rendering, egress animation, structural temperature contours |
| Geometry Builder | Custom React + Three.js | CAD/DXF import, 3D extrusion, mesh generation, obstruction placement |
| CFD Meshing | Rust (custom octree) | Cartesian mesh with embedded boundaries, automatic refinement near obstructions |
| Real-time | WebSocket (Axum) | Live CFD solver progress, egress timestep streaming |
| Job Queue | Redis 7 + Tokio tasks | CFD job management, multi-scenario Monte Carlo distribution |
| Compute | AWS GPU (g5.xlarge+) | CFD simulations on CUDA-capable instances, spot instances for Monte Carlo |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | Static frontend assets, large result file delivery |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, CFD solver performance, error tracking |
| CI/CD | GitHub Actions | Rust tests, CUDA build pipeline, Docker image push |

### Key Architecture Decisions

1. **Rust-based LES CFD solver with CUDA acceleration for fire simulation**: Building a custom Large Eddy Simulation solver in Rust gives us full control over combustion models, GPU parallelization, and cloud scalability. FDS (Fire Dynamics Simulator) is the industry standard but is written in Fortran with MPI parallelization that requires clusters — our Rust+CUDA solver runs efficiently on single GPU instances (g5.xlarge, A10G GPU) with 10-30x speedup over CPU-only execution. We implement FDS-compatible physics (pressure-velocity coupling via projection method, mixture fraction combustion, gray gas radiation, Lagrangian droplet spray) while maintaining memory safety and cloud-native job orchestration.

2. **Hybrid zone model (fast preview) and CFD solver (production) with automatic mesh generation**: 90% of fire engineering studies start with a quick zone model calculation (CFAST-equivalent two-zone smoke filling) to get ballpark smoke layer height and time to untenable conditions. We provide a Python-based zone model (solves algebraic plume entrainment and heat balance equations in <1 second) for instant preview, then users can upgrade to full CFD for complex geometries, natural ventilation, or atrium scenarios. The CFD mesh is generated automatically from imported geometry via octree refinement with target cell sizes of 0.1-0.5m (D*/10 rule where D* is characteristic fire diameter).

3. **Tightly coupled CFD-egress simulation with real-time visibility and toxicity feedback**: Unlike separate fire and egress tools (FDS+Pathfinder) where smoke data must be manually transferred, our egress agents read soot density, gas temperature, and toxicant concentrations directly from the CFD grid at each timestep. Agent movement speed is reduced when visibility <10m (SFPE guidelines), and FED (Fractional Effective Dose) is accumulated along each agent's path for CO (Purser model), HCN, heat, and O2 depletion. This coupling produces defensible ASET vs. RSET results without manual post-processing.

4. **Three.js volumetric rendering with GPU-accelerated raycasting for smoke visualization**: Smoke density and temperature fields (3D scalar arrays from CFD) are rendered as volumetric textures using Three.js raymarching shaders. This allows real-time fly-through visualization of smoke spread with adjustable opacity transfer functions (smoke density → alpha channel). Isosurface rendering (marching cubes) extracts constant-visibility surfaces (e.g., 10m visibility boundary) for ASET determination. Slice planes show temperature/velocity/toxicity at user-defined heights and times.

5. **S3 for large field data with PostgreSQL metadata and Redis hot-cache for recent simulations**: CFD simulations produce 1-10 GB of field data (temperature, velocity, soot density at every cell and timestep). Raw field data is stored in S3 as compressed binary (HDF5 or custom format), while PostgreSQL holds simulation metadata (HRR, mesh size, runtime, summary measurements like max temperature, smoke layer height). Recently accessed results (last 24 hours) are cached in Redis to avoid S3 latency for interactive 3D visualization.

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
    auth_provider TEXT DEFAULT 'email',  -- email | google | github
    auth_provider_id TEXT,
    plan TEXT NOT NULL DEFAULT 'free',  -- free | pro | advanced | enterprise
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX users_email_idx ON users(email);
CREATE INDEX users_stripe_idx ON users(stripe_customer_id);

-- Organizations (for Advanced/Enterprise)
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

-- Projects (geometry + simulation setup)
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    geometry_data JSONB NOT NULL DEFAULT '{}',  -- Rooms, obstructions, vents, doors
    simulation_config JSONB DEFAULT '{}',  -- Fire location, HRR curve, occupancy, detectors
    is_public BOOLEAN NOT NULL DEFAULT false,
    forked_from UUID REFERENCES projects(id) ON DELETE SET NULL,
    thumbnail_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);
CREATE INDEX projects_org_idx ON projects(org_id);
CREATE INDEX projects_updated_idx ON projects(updated_at DESC);

-- Simulations (fire + egress runs)
CREATE TABLE simulations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    simulation_type TEXT NOT NULL,  -- zone_model | cfd | egress | coupled
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | meshing | running | completed | failed | cancelled
    execution_mode TEXT NOT NULL DEFAULT 'cloud',  -- cloud | local (for future desktop app)
    mesh_resolution TEXT,  -- coarse | medium | fine | custom
    cell_count INTEGER DEFAULT 0,
    fire_parameters JSONB NOT NULL DEFAULT '{}',  -- HRR curve, location, fuel type
    egress_parameters JSONB DEFAULT '{}',  -- Occupant count, distributions, pre-movement
    structural_parameters JSONB DEFAULT '{}',  -- Member IDs, exposure type
    results_url TEXT,  -- S3 URL for field data
    results_summary JSONB,  -- ASET, RSET, max temps, smoke layer height, last out time
    error_message TEXT,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    wall_time_sec INTEGER,
    compute_time_sec INTEGER,  -- Billable compute time
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX sims_project_idx ON simulations(project_id);
CREATE INDEX sims_user_idx ON simulations(user_id);
CREATE INDEX sims_status_idx ON simulations(status);
CREATE INDEX sims_created_idx ON simulations(created_at DESC);

-- Cloud Simulation Jobs (for GPU job scheduling)
CREATE TABLE simulation_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    simulation_id UUID NOT NULL REFERENCES simulations(id) ON DELETE CASCADE,
    worker_id TEXT,
    gpu_type TEXT,  -- a10g | t4 | a100
    instance_id TEXT,
    cores_allocated INTEGER DEFAULT 8,
    memory_gb INTEGER DEFAULT 32,
    gpu_memory_gb INTEGER DEFAULT 24,
    priority INTEGER DEFAULT 0,
    progress_pct REAL DEFAULT 0.0,
    current_timestep INTEGER DEFAULT 0,
    total_timesteps INTEGER,
    convergence_data JSONB DEFAULT '[]',  -- [{timestep, residual, CFL, max_temp}]
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX jobs_sim_idx ON simulation_jobs(simulation_id);
CREATE INDEX jobs_worker_idx ON simulation_jobs(worker_id);
CREATE INDEX jobs_status_idx ON simulation_jobs(started_at, completed_at);

-- Design Fires (library of standard scenarios)
CREATE TABLE design_fires (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    occupancy_type TEXT NOT NULL,  -- office | retail | residential | assembly | parking | industrial
    fire_type TEXT NOT NULL,  -- t_squared | prescribed_hrr | pyrolysis
    hrr_curve JSONB NOT NULL,  -- Time-HRR pairs or growth rate parameters
    soot_yield REAL DEFAULT 0.1,  -- kg/kg
    co_yield REAL DEFAULT 0.04,
    hcn_yield REAL DEFAULT 0.0,
    heat_of_combustion REAL DEFAULT 25000.0,  -- kJ/kg
    description TEXT,
    reference TEXT,  -- SFPE Handbook, NFPA 72, EN 1991-1-2
    is_builtin BOOLEAN DEFAULT true,
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX fires_occupancy_idx ON design_fires(occupancy_type);

-- Material Properties (for pyrolysis and structural)
CREATE TABLE materials (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    category TEXT NOT NULL,  -- fuel | structure | insulation
    density REAL,  -- kg/m³
    specific_heat REAL,  -- J/(kg·K)
    conductivity REAL,  -- W/(m·K)
    emissivity REAL,  -- 0-1
    ignition_temp REAL,  -- °C
    pyrolysis_params JSONB,  -- Arrhenius parameters for decomposition
    structural_params JSONB,  -- Strength reduction factors vs. temperature
    is_builtin BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX materials_category_idx ON materials(category);

-- Fuel Load Database (for parametric fire design)
CREATE TABLE fuel_loads (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    occupancy_type TEXT NOT NULL,
    description TEXT,
    mean_fuel_load_mj_m2 REAL NOT NULL,  -- MJ/m² floor area
    std_dev_mj_m2 REAL,
    percentile_80_mj_m2 REAL,
    reference TEXT,  -- Eurocode, SFPE, local code
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX fuel_loads_occupancy_idx ON fuel_loads(occupancy_type);

-- Egress Occupancy Parameters (pre-movement, speed distributions)
CREATE TABLE occupancy_parameters (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    occupancy_type TEXT NOT NULL,
    alarm_type TEXT NOT NULL,  -- voice | bell | visible
    pre_movement_mean_sec REAL NOT NULL,
    pre_movement_std_sec REAL NOT NULL,
    distribution_type TEXT DEFAULT 'lognormal',
    unimpeded_speed_mean REAL DEFAULT 1.2,  -- m/s
    unimpeded_speed_std REAL DEFAULT 0.3,
    reference TEXT,  -- SFPE, PD 7974-6, Proulx
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX occ_params_type_idx ON occupancy_parameters(occupancy_type);

-- Usage Tracking (for billing)
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,  -- compute_hours | storage_gb_month | api_calls
    quantity REAL NOT NULL,
    simulation_id UUID REFERENCES simulations(id),
    period_start DATE NOT NULL,
    period_end DATE NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX usage_user_period_idx ON usage_records(user_id, period_start, period_end);
CREATE INDEX usage_sim_idx ON usage_records(simulation_id);

-- Saved Report Configurations
CREATE TABLE report_configs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    simulation_id UUID NOT NULL REFERENCES simulations(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    report_type TEXT NOT NULL,  -- aset_rset | structural | smoke_control | full
    sections JSONB NOT NULL,  -- Which sections to include, custom text
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX report_sim_idx ON report_configs(simulation_id);
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
    pub geometry_data: serde_json::Value,
    pub simulation_config: serde_json::Value,
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
    pub mesh_resolution: Option<String>,
    pub cell_count: i32,
    pub fire_parameters: serde_json::Value,
    pub egress_parameters: Option<serde_json::Value>,
    pub structural_parameters: Option<serde_json::Value>,
    pub results_url: Option<String>,
    pub results_summary: Option<serde_json::Value>,
    pub error_message: Option<String>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub wall_time_sec: Option<i32>,
    pub compute_time_sec: Option<i32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct SimulationJob {
    pub id: Uuid,
    pub simulation_id: Uuid,
    pub worker_id: Option<String>,
    pub gpu_type: Option<String>,
    pub instance_id: Option<String>,
    pub cores_allocated: i32,
    pub memory_gb: i32,
    pub gpu_memory_gb: i32,
    pub priority: i32,
    pub progress_pct: f32,
    pub current_timestep: i32,
    pub total_timesteps: Option<i32>,
    pub convergence_data: serde_json::Value,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct DesignFire {
    pub id: Uuid,
    pub name: String,
    pub occupancy_type: String,
    pub fire_type: String,
    pub hrr_curve: serde_json::Value,
    pub soot_yield: f32,
    pub co_yield: f32,
    pub hcn_yield: f32,
    pub heat_of_combustion: f32,
    pub description: Option<String>,
    pub reference: Option<String>,
    pub is_builtin: bool,
    pub created_by: Option<Uuid>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Material {
    pub id: Uuid,
    pub name: String,
    pub category: String,
    pub density: Option<f32>,
    pub specific_heat: Option<f32>,
    pub conductivity: Option<f32>,
    pub emissivity: Option<f32>,
    pub ignition_temp: Option<f32>,
    pub pyrolysis_params: Option<serde_json::Value>,
    pub structural_params: Option<serde_json::Value>,
    pub is_builtin: bool,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct FireParameters {
    pub fire_type: FireType,
    pub location: FireLocation,
    pub hrr_data: HRRData,
    pub soot_yield: f64,
    pub co_yield: f64,
    pub hcn_yield: f64,
    pub radiative_fraction: f64,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum FireType {
    TSquared,
    PrescribedHRR,
    Pyrolysis,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct FireLocation {
    pub x: f64,
    pub y: f64,
    pub z: f64,
    pub area: f64,  // m²
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(untagged)]
pub enum HRRData {
    TSquared { alpha: f64 },  // kW/s²
    Prescribed { times: Vec<f64>, hrr_values: Vec<f64> },  // kW
}

#[derive(Debug, Deserialize, Serialize)]
pub struct EgressParameters {
    pub occupant_count: u32,
    pub pre_movement_dist: PreMovementDist,
    pub speed_dist: SpeedDist,
    pub familiar_fraction: f64,  // 0-1, fraction familiar with layout
    pub impaired_fraction: f64,  // Reduced mobility
}

#[derive(Debug, Deserialize, Serialize)]
pub struct PreMovementDist {
    pub mean_sec: f64,
    pub std_sec: f64,
    pub distribution: String,  // lognormal | normal | uniform
}

#[derive(Debug, Deserialize, Serialize)]
pub struct SpeedDist {
    pub mean_m_s: f64,
    pub std_m_s: f64,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct StructuralParameters {
    pub members: Vec<StructuralMember>,
    pub exposure_type: ExposureType,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct StructuralMember {
    pub id: String,
    pub member_type: String,  // steel_column | steel_beam | concrete_slab
    pub section: String,  // e.g., "W14x90", "300x300"
    pub protection: Option<Protection>,
    pub location: (f64, f64, f64),
}

#[derive(Debug, Deserialize, Serialize)]
pub struct Protection {
    pub protection_type: String,  // intumescent | board | spray
    pub thickness_mm: f64,
    pub conductivity: f64,
    pub density: f64,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum ExposureType {
    ISO834,
    Hydrocarbon,
    External,
    Natural,  // From CFD
}
```

---

## Solver Architecture Deep-Dive

### Governing Equations and Discretization

FireSim's CFD fire solver implements **Large Eddy Simulation (LES)** for low-Mach-number buoyancy-driven flows with combustion, radiation, and soot transport. The governing equations are the filtered Navier-Stokes equations for mass, momentum, and energy:

**Mass conservation (incompressible low-Mach form):**
```
∇·u = 0
```

**Momentum (with Boussinesq buoyancy):**
```
∂u/∂t + (u·∇)u = -∇p/ρ₀ + ν∇²u + g(1 - T/T₀)
```

**Energy (temperature evolution):**
```
∂T/∂t + u·∇T = α∇²T + Q̇_combustion - Q̇_radiation + Q̇_spray
```

**Mixture fraction (combustion):**
```
∂Z/∂t + u·∇Z = D∇²Z
```
where Z = 0 is pure air, Z = 1 is pure fuel. Heat release rate Q̇ = Δh_c · χ · ∇²Z (fast chemistry limit).

**Soot mass fraction (smoke):**
```
∂Y_s/∂t + u·∇Y_s = D∇²Y_s + y_s · Q̇ / Δh_c
```
where y_s is soot yield (kg soot / kg fuel).

**Radiation (gray gas finite volume method):**
Solve radiative transfer equation (RTE) for intensity I at each discrete solid angle:
```
∇·(I Ω) + κI = κI_b
```
where κ is absorption coefficient (Planck mean, from soot), I_b is blackbody intensity (Stefan-Boltzmann). The divergence of radiative flux gives Q̇_radiation.

**Pressure-velocity coupling:** Projection method (Chorin's fractional step):
1. Predictor: advance momentum without pressure gradient → intermediate velocity u*
2. Solve Poisson equation for pressure correction: ∇²φ = ∇·u* / Δt
3. Corrector: u^(n+1) = u* - Δt∇φ

**Time integration:** Explicit Runge-Kutta 2nd order for advection and diffusion, with CFL-based adaptive timestep:
```
Δt = C_CFL · min(Δx / |u|, Δy / |v|, Δz / |w|, √(Δx² / α))
```
Typical C_CFL = 0.5-0.8.

**Spatial discretization:** Cartesian grid (uniform or octree-refined) with 2nd-order central differences for diffusion, 5th-order WENO (Weighted Essentially Non-Oscillatory) for advection to capture sharp fronts (flame, smoke interface).

**Turbulence modeling:** Smagorinsky LES subgrid-scale model:
```
ν_sgs = (C_s Δ)² |S̄|
```
where Δ = (Δx Δy Δz)^(1/3) is filter width, |S̄| is resolved strain rate magnitude, C_s ≈ 0.1-0.2.

### CFD-Egress Coupling

```
CFD Solver (GPU) ──────> Field Data (T, ρ_smoke, Y_CO, visibility)
                            │
                            ↓ (interpolate to agent position)
                            │
Egress Solver (CPU) ──────> Agent speed reduction, FED accumulation
```

At each egress timestep (0.1-0.5s), agents query the CFD grid:
- **Visibility** from soot: V = C / κ, where C ≈ 3-8 (Seader-Ou formula), κ is extinction coefficient
- **Movement speed** reduced if V < 10m: v_actual = v_max · f(V), where f is visibility factor from SFPE
- **FED accumulation** for CO: dFED_CO/dt = C_CO / LC50_CO, where LC50_CO depends on exposure time (Purser)
- **Temperature tenability**: check if T_agent > 60°C (threshold for incapacitation)

Agent becomes incapacitated if FED > 0.3 (ISO 13571) or T > 100°C.

### Structural Fire Coupling

For steel members, lumped-mass temperature rise (EN 1993-1-2):
```
Δθ_steel = (h_net · A_m / V) / (c_steel · ρ_steel) · Δt
```
where h_net is net heat flux (convection + radiation from CFD gas temperature and radiation solver), A_m/V is section factor (exposed surface area per unit volume).

For 2D FEM heat transfer (concrete, complex steel sections):
```
ρc ∂T/∂t = ∇·(k∇T) + Q̇_internal
```
Boundary conditions from CFD: convective BC with h_conv · (T_gas - T_surface) + σε(T_rad⁴ - T_surface⁴).

Temperature-dependent strength reduction factors applied per Eurocode for subsequent structural analysis (out of scope for MVP).

---

## Architecture Deep-Dives

### 1. CFD Simulation API Handler (Rust/Axum)

The primary endpoint receives a simulation request, validates geometry, generates mesh, decides compute resources (GPU type based on mesh size), enqueues job with Redis, and streams progress via WebSocket.

```rust
// src/api/handlers/simulation.rs

use axum::{
    extract::{Path, State, Json},
    http::StatusCode,
    response::IntoResponse,
};
use uuid::Uuid;
use crate::{
    db::models::{Simulation, FireParameters, EgressParameters, StructuralParameters},
    mesher::generate_mesh,
    state::AppState,
    auth::Claims,
    error::ApiError,
};

#[derive(serde::Deserialize)]
pub struct CreateSimulationRequest {
    pub simulation_type: String,  // zone_model | cfd | egress | coupled
    pub mesh_resolution: Option<String>,  // coarse | medium | fine
    pub fire_parameters: FireParameters,
    pub egress_parameters: Option<EgressParameters>,
    pub structural_parameters: Option<StructuralParameters>,
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

    // 2. Estimate mesh size
    let geometry = &project.geometry_data;
    let mesh_res = req.mesh_resolution.as_deref().unwrap_or("medium");
    let target_dx = match mesh_res {
        "coarse" => 0.5,   // 50 cm cells
        "medium" => 0.25,  // 25 cm cells
        "fine" => 0.1,     // 10 cm cells
        _ => 0.25,
    };

    let mesh_info = generate_mesh(geometry, target_dx)?;
    let cell_count = mesh_info.cell_count;

    // 3. Check plan limits
    let user = sqlx::query_as!(
        crate::db::models::User,
        "SELECT * FROM users WHERE id = $1",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    if user.plan == "free" && req.simulation_type != "zone_model" {
        return Err(ApiError::PlanLimit(
            "Free plan only supports zone model. Upgrade to Pro for CFD."
        ));
    }

    if user.plan == "pro" && cell_count > 500_000 {
        return Err(ApiError::PlanLimit(
            "Pro plan supports up to 500K cells. Upgrade to Advanced for unlimited."
        ));
    }

    // 4. Determine GPU requirements
    let (gpu_type, memory_gb) = if cell_count < 100_000 {
        ("t4", 16)  // T4 for small meshes
    } else if cell_count < 1_000_000 {
        ("a10g", 24)  // A10G for medium
    } else {
        ("a100", 80)  // A100 for large
    };

    // 5. Create simulation record
    let sim = sqlx::query_as!(
        Simulation,
        r#"INSERT INTO simulations
            (project_id, user_id, simulation_type, status, execution_mode,
             mesh_resolution, cell_count, fire_parameters, egress_parameters,
             structural_parameters)
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10)
        RETURNING *"#,
        project_id,
        claims.user_id,
        req.simulation_type,
        if req.simulation_type == "zone_model" { "running" } else { "pending" },
        "cloud",
        Some(mesh_res.to_string()),
        cell_count as i32,
        serde_json::to_value(&req.fire_parameters)?,
        req.egress_parameters.map(|p| serde_json::to_value(&p)).transpose()?,
        req.structural_parameters.map(|p| serde_json::to_value(&p)).transpose()?,
    )
    .fetch_one(&state.db)
    .await?;

    // 6. For zone model, run immediately (fast)
    if req.simulation_type == "zone_model" {
        tokio::spawn(async move {
            let result = crate::solvers::zone_model::run_zone_model(
                &geometry,
                &req.fire_parameters,
            ).await;
            // Update database with result
        });
        return Ok((StatusCode::CREATED, Json(sim)));
    }

    // 7. For CFD/coupled, enqueue job
    let job = sqlx::query_as!(
        crate::db::models::SimulationJob,
        r#"INSERT INTO simulation_jobs
            (simulation_id, gpu_type, memory_gb, gpu_memory_gb, priority)
        VALUES ($1, $2, $3, $4, $5) RETURNING *"#,
        sim.id,
        gpu_type,
        memory_gb,
        if gpu_type == "a100" { 80 } else if gpu_type == "a10g" { 24 } else { 16 },
        if user.plan == "advanced" { 10 } else { 0 },
    )
    .fetch_one(&state.db)
    .await?;

    // Publish to Redis job queue
    state.redis
        .lpush::<_, _, ()>("simulation:jobs", serde_json::to_string(&job.id)?)
        .await?;

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
        return Ok((StatusCode::OK, Json(sim)));
    }

    // Check if results are in Redis cache
    let cache_key = format!("results:{}", sim.id);
    if let Some(cached) = state.redis.get::<_, Option<String>>(&cache_key).await? {
        return Ok((StatusCode::OK, Json(serde_json::from_str(&cached)?)));
    }

    // Fetch from S3 and cache in Redis
    let results_url = sim.results_url.ok_or(ApiError::NotFound("Results not found"))?;
    let s3_data = state.s3.get_object()
        .bucket("firesim-results")
        .key(&results_url.trim_start_matches("s3://firesim-results/"))
        .send()
        .await?;

    let body = s3_data.body.collect().await?.into_bytes();

    // Cache for 1 hour
    state.redis.setex(&cache_key, 3600, body.as_ref()).await?;

    Ok((StatusCode::OK, body.into_response()))
}
```

### 2. CFD Fire Solver Core (Rust + CUDA)

The core LES solver that implements pressure-velocity coupling, combustion, radiation, and soot transport. This code uses CUDA kernels for GPU acceleration.

```rust
// cfd-solver/src/les_solver.rs

use cuda_runtime_sys::*;
use std::ptr;

pub struct LESGrid {
    pub nx: usize,
    pub ny: usize,
    pub nz: usize,
    pub dx: f64,
    pub dy: f64,
    pub dz: f64,
    pub cells: usize,

    // Device pointers (GPU memory)
    pub d_u: *mut f64,      // Velocity x
    pub d_v: *mut f64,      // Velocity y
    pub d_w: *mut f64,      // Velocity z
    pub d_p: *mut f64,      // Pressure
    pub d_T: *mut f64,      // Temperature
    pub d_Z: *mut f64,      // Mixture fraction
    pub d_soot: *mut f64,   // Soot mass fraction
    pub d_co: *mut f64,     // CO mass fraction
}

impl LESGrid {
    pub fn new(nx: usize, ny: usize, nz: usize, dx: f64, dy: f64, dz: f64) -> Self {
        let cells = nx * ny * nz;
        let size = cells * std::mem::size_of::<f64>();

        unsafe {
            let mut d_u = ptr::null_mut();
            let mut d_v = ptr::null_mut();
            let mut d_w = ptr::null_mut();
            let mut d_p = ptr::null_mut();
            let mut d_T = ptr::null_mut();
            let mut d_Z = ptr::null_mut();
            let mut d_soot = ptr::null_mut();
            let mut d_co = ptr::null_mut();

            cudaMalloc(&mut d_u as *mut *mut _ as *mut *mut std::ffi::c_void, size);
            cudaMalloc(&mut d_v as *mut *mut _ as *mut *mut std::ffi::c_void, size);
            cudaMalloc(&mut d_w as *mut *mut _ as *mut *mut std::ffi::c_void, size);
            cudaMalloc(&mut d_p as *mut *mut _ as *mut *mut std::ffi::c_void, size);
            cudaMalloc(&mut d_T as *mut *mut _ as *mut *mut std::ffi::c_void, size);
            cudaMalloc(&mut d_Z as *mut *mut _ as *mut *mut std::ffi::c_void, size);
            cudaMalloc(&mut d_soot as *mut *mut _ as *mut *mut std::ffi::c_void, size);
            cudaMalloc(&mut d_co as *mut *mut _ as *mut *mut std::ffi::c_void, size);

            Self {
                nx, ny, nz, dx, dy, dz, cells,
                d_u, d_v, d_w, d_p, d_T, d_Z, d_soot, d_co,
            }
        }
    }

    pub fn idx(&self, i: usize, j: usize, k: usize) -> usize {
        i + j * self.nx + k * self.nx * self.ny
    }
}

pub struct FireSolver {
    pub grid: LESGrid,
    pub time: f64,
    pub dt: f64,
    pub fire_hrr: Vec<(f64, f64)>,  // (time, HRR in kW)
    pub fire_location: (usize, usize, usize),
    pub soot_yield: f64,
    pub co_yield: f64,
    pub chi_r: f64,  // Radiative fraction
}

impl FireSolver {
    pub fn new(
        nx: usize, ny: usize, nz: usize,
        dx: f64, dy: f64, dz: f64,
        fire_hrr: Vec<(f64, f64)>,
        fire_location: (usize, usize, usize),
    ) -> Self {
        Self {
            grid: LESGrid::new(nx, ny, nz, dx, dy, dz),
            time: 0.0,
            dt: 0.01,  // Initial guess, will be adaptive
            fire_hrr,
            fire_location,
            soot_yield: 0.1,
            co_yield: 0.04,
            chi_r: 0.35,
        }
    }

    /// Main timestep loop
    pub fn step(&mut self) -> Result<(), SolverError> {
        // 1. Compute CFL-limited timestep
        self.dt = self.compute_dt()?;

        // 2. Advection step (WENO5 on GPU)
        self.advect_scalars()?;

        // 3. Diffusion step (implicit solve on GPU)
        self.diffuse_scalars()?;

        // 4. Combustion source terms
        self.apply_combustion()?;

        // 5. Radiation solver (FVM gray gas)
        self.solve_radiation()?;

        // 6. Momentum predictor (no pressure gradient)
        self.momentum_predictor()?;

        // 7. Pressure Poisson solve (conjugate gradient on GPU)
        self.solve_pressure_poisson()?;

        // 8. Velocity correction
        self.velocity_corrector()?;

        // 9. Soot and CO transport with source terms
        self.transport_species()?;

        self.time += self.dt;
        Ok(())
    }

    fn compute_dt(&self) -> Result<f64, SolverError> {
        // Launch CUDA kernel to find max |u|, |v|, |w| on device
        let max_vel = self.get_max_velocity_gpu()?;
        let dt_cfl = 0.5 * self.grid.dx.min(self.grid.dy).min(self.grid.dz) / max_vel;
        let dt_diff = 0.25 * self.grid.dx.powi(2) / (1e-5 + 1e-5);  // ν + α
        Ok(dt_cfl.min(dt_diff).min(0.1))  // Cap at 0.1s
    }

    fn advect_scalars(&mut self) -> Result<(), SolverError> {
        // Call CUDA kernel for WENO5 advection of T, Z, soot
        unsafe {
            let threads = dim3 { x: 8, y: 8, z: 8, _unused: [0; 1] };
            let blocks = dim3 {
                x: ((self.grid.nx + 7) / 8) as u32,
                y: ((self.grid.ny + 7) / 8) as u32,
                z: ((self.grid.nz + 7) / 8) as u32,
                _unused: [0; 1],
            };

            // External CUDA kernel defined in .cu file
            extern "C" {
                fn advect_weno5_kernel(
                    d_T: *mut f64, d_u: *const f64, d_v: *const f64, d_w: *const f64,
                    nx: u32, ny: u32, nz: u32, dx: f64, dy: f64, dz: f64, dt: f64,
                );
            }

            advect_weno5_kernel(
                self.grid.d_T,
                self.grid.d_u, self.grid.d_v, self.grid.d_w,
                self.grid.nx as u32, self.grid.ny as u32, self.grid.nz as u32,
                self.grid.dx, self.grid.dy, self.grid.dz, self.dt,
            );
            cudaDeviceSynchronize();
        }
        Ok(())
    }

    fn apply_combustion(&mut self) -> Result<(), SolverError> {
        // HRR from fire curve at current time
        let hrr = self.interpolate_hrr(self.time);  // kW
        let (i_fire, j_fire, k_fire) = self.fire_location;
        let cell_volume = self.grid.dx * self.grid.dy * self.grid.dz;

        // Q̇''' = HRR / V_cell (kW/m³)
        let q_release = hrr / cell_volume;

        // Launch kernel to add source term to energy equation at fire cell
        // Also generate soot and CO based on yields
        unsafe {
            extern "C" {
                fn combustion_source_kernel(
                    d_T: *mut f64, d_Z: *mut f64, d_soot: *mut f64, d_co: *mut f64,
                    i_fire: u32, j_fire: u32, k_fire: u32,
                    q_release: f64, soot_yield: f64, co_yield: f64,
                    dt: f64, nx: u32, ny: u32, nz: u32,
                );
            }

            combustion_source_kernel(
                self.grid.d_T, self.grid.d_Z, self.grid.d_soot, self.grid.d_co,
                i_fire as u32, j_fire as u32, k_fire as u32,
                q_release, self.soot_yield, self.co_yield,
                self.dt, self.grid.nx as u32, self.grid.ny as u32, self.grid.nz as u32,
            );
            cudaDeviceSynchronize();
        }
        Ok(())
    }

    fn solve_radiation(&mut self) -> Result<(), SolverError> {
        // Gray gas FVM radiation solver on GPU
        // Simplified: divergence of radiative flux from soot absorption
        unsafe {
            extern "C" {
                fn radiation_fvm_kernel(
                    d_T: *const f64, d_soot: *const f64, d_qrad: *mut f64,
                    nx: u32, ny: u32, nz: u32, dx: f64, dy: f64, dz: f64,
                );
            }

            let mut d_qrad = ptr::null_mut();
            cudaMalloc(
                &mut d_qrad as *mut *mut _ as *mut *mut std::ffi::c_void,
                self.grid.cells * std::mem::size_of::<f64>(),
            );

            radiation_fvm_kernel(
                self.grid.d_T, self.grid.d_soot, d_qrad,
                self.grid.nx as u32, self.grid.ny as u32, self.grid.nz as u32,
                self.grid.dx, self.grid.dy, self.grid.dz,
            );

            // Subtract radiative loss from temperature field
            // (kernel applies q_rad to d_T)

            cudaFree(d_qrad as *mut std::ffi::c_void);
            cudaDeviceSynchronize();
        }
        Ok(())
    }

    fn solve_pressure_poisson(&mut self) -> Result<(), SolverError> {
        // Conjugate gradient solver for ∇²p = ∇·u* / dt on GPU
        // Uses cuSPARSE for sparse matrix operations
        // Simplified here, full implementation uses cuSPARSE + cuBLAS
        Ok(())
    }

    fn interpolate_hrr(&self, time: f64) -> f64 {
        // Linear interpolation of HRR curve
        for i in 0..self.fire_hrr.len() - 1 {
            let (t0, q0) = self.fire_hrr[i];
            let (t1, q1) = self.fire_hrr[i + 1];
            if time >= t0 && time <= t1 {
                return q0 + (q1 - q0) * (time - t0) / (t1 - t0);
            }
        }
        self.fire_hrr.last().map(|(_, q)| *q).unwrap_or(0.0)
    }

    // Stubs for other methods
    fn diffuse_scalars(&mut self) -> Result<(), SolverError> { Ok(()) }
    fn momentum_predictor(&mut self) -> Result<(), SolverError> { Ok(()) }
    fn velocity_corrector(&mut self) -> Result<(), SolverError> { Ok(()) }
    fn transport_species(&mut self) -> Result<(), SolverError> { Ok(()) }
    fn get_max_velocity_gpu(&self) -> Result<f64, SolverError> { Ok(1.0) }
}

#[derive(Debug)]
pub enum SolverError {
    CudaError(String),
    Divergence(String),
}
```

### 3. Egress Agent-Based Simulation (Rust)

The egress solver that models individual occupants with pathfinding, density-dependent speed, and FED accumulation coupled to CFD results.

```rust
// egress-solver/src/agent_model.rs

use std::collections::HashMap;
use rand::distributions::{Distribution, LogNormal};
use rand::thread_rng;

pub struct EgressGrid {
    pub nx: usize,
    pub ny: usize,
    pub dx: f64,
    pub dy: f64,
    pub obstacles: Vec<Vec<bool>>,  // 2D grid of walkable cells
    pub exits: Vec<(usize, usize)>,
}

pub struct Agent {
    pub id: usize,
    pub x: f64,
    pub y: f64,
    pub z: f64,  // Floor level
    pub state: AgentState,
    pub target: Option<(usize, usize)>,  // Target cell
    pub speed_max: f64,  // m/s, unimpeded
    pub speed_current: f64,
    pub pre_movement_time: f64,  // Seconds before starting evacuation
    pub start_time: f64,
    pub fed_co: f64,
    pub fed_heat: f64,
    pub fed_total: f64,
    pub familiar: bool,
    pub path: Vec<(usize, usize)>,
}

#[derive(Debug, Clone, Copy, PartialEq)]
pub enum AgentState {
    PreMovement,
    Moving,
    Exited,
    Incapacitated,
}

pub struct EgressSolver {
    pub grid: EgressGrid,
    pub agents: Vec<Agent>,
    pub time: f64,
    pub dt: f64,
    pub cfd_fields: Option<CFDFields>,  // Coupled from CFD
}

pub struct CFDFields {
    pub temperature: Vec<Vec<f64>>,  // 2D slice at agent height
    pub soot_density: Vec<Vec<f64>>,
    pub co_concentration: Vec<Vec<f64>>,  // ppm
}

impl EgressSolver {
    pub fn new(grid: EgressGrid, occupant_count: usize, params: &EgressParameters) -> Self {
        let mut rng = thread_rng();
        let pre_mvmt_dist = LogNormal::new(
            params.pre_movement_dist.mean_sec.ln(),
            params.pre_movement_dist.std_sec / params.pre_movement_dist.mean_sec,
        ).unwrap();

        let speed_dist = rand_distr::Normal::new(
            params.speed_dist.mean_m_s,
            params.speed_dist.std_m_s,
        ).unwrap();

        let agents: Vec<Agent> = (0..occupant_count)
            .map(|id| {
                // Random initial position in walkable cells
                let (x, y) = Self::random_walkable_position(&grid, &mut rng);
                Agent {
                    id,
                    x,
                    y,
                    z: 0.0,
                    state: AgentState::PreMovement,
                    target: None,
                    speed_max: speed_dist.sample(&mut rng).max(0.3),
                    speed_current: 0.0,
                    pre_movement_time: pre_mvmt_dist.sample(&mut rng),
                    start_time: 0.0,
                    fed_co: 0.0,
                    fed_heat: 0.0,
                    fed_total: 0.0,
                    familiar: rng.gen::<f64>() < params.familiar_fraction,
                    path: vec![],
                }
            })
            .collect();

        Self {
            grid,
            agents,
            time: 0.0,
            dt: 0.1,  // 0.1s egress timestep
            cfd_fields: None,
        }
    }

    pub fn step(&mut self) -> Result<(), EgressError> {
        for agent in &mut self.agents {
            match agent.state {
                AgentState::PreMovement => {
                    if self.time >= agent.pre_movement_time {
                        agent.state = AgentState::Moving;
                        agent.start_time = self.time;
                        // Assign target exit (nearest for familiar, random for unfamiliar)
                        agent.target = Some(self.select_exit(agent));
                        // Compute path using A*
                        if let Some(target) = agent.target {
                            agent.path = self.astar_path(
                                (agent.x / self.grid.dx) as usize,
                                (agent.y / self.grid.dy) as usize,
                                target.0,
                                target.1,
                            );
                        }
                    }
                }
                AgentState::Moving => {
                    // Move along path
                    if let Some((target_i, target_j)) = agent.path.first() {
                        let target_x = *target_i as f64 * self.grid.dx;
                        let target_y = *target_j as f64 * self.grid.dy;
                        let dx = target_x - agent.x;
                        let dy = target_y - agent.y;
                        let dist = (dx * dx + dy * dy).sqrt();

                        // Density at current cell
                        let density = self.compute_local_density(agent.x, agent.y);

                        // Speed reduction from density (Fruin)
                        let speed_density = self.speed_from_density(density, agent.speed_max);

                        // Speed reduction from visibility (CFD coupling)
                        let visibility = self.compute_visibility(agent.x, agent.y);
                        let speed_visibility = if visibility < 10.0 {
                            speed_density * (0.3 + 0.7 * visibility / 10.0)
                        } else {
                            speed_density
                        };

                        agent.speed_current = speed_visibility;

                        // Move toward next waypoint
                        let step_dist = agent.speed_current * self.dt;
                        if step_dist >= dist {
                            agent.x = target_x;
                            agent.y = target_y;
                            agent.path.remove(0);

                            // Check if reached exit
                            if agent.path.is_empty() {
                                agent.state = AgentState::Exited;
                            }
                        } else {
                            agent.x += (dx / dist) * step_dist;
                            agent.y += (dy / dist) * step_dist;
                        }
                    }

                    // Update FED
                    self.update_fed(agent)?;

                    // Check incapacitation
                    if agent.fed_total > 0.3 {
                        agent.state = AgentState::Incapacitated;
                    }
                }
                _ => {}
            }
        }

        self.time += self.dt;
        Ok(())
    }

    fn compute_local_density(&self, x: f64, y: f64) -> f64 {
        // Count agents within 2m radius
        let radius = 2.0;
        let count = self.agents.iter()
            .filter(|a| {
                let dx = a.x - x;
                let dy = a.y - y;
                (dx * dx + dy * dy).sqrt() < radius && a.state == AgentState::Moving
            })
            .count();
        count as f64 / (std::f64::consts::PI * radius * radius)  // persons/m²
    }

    fn speed_from_density(&self, density: f64, speed_max: f64) -> f64 {
        // SFPE correlation: v = v_max · (1 - a·ρ^b)
        // For a=0.266, b=0.4 (Fruin)
        if density < 0.01 {
            return speed_max;
        }
        speed_max * (1.0 - 0.266 * density.powf(0.4)).max(0.1)
    }

    fn compute_visibility(&self, x: f64, y: f64) -> f64 {
        // Query CFD soot density at agent position
        if let Some(ref fields) = self.cfd_fields {
            let i = (x / self.grid.dx) as usize;
            let j = (y / self.grid.dy) as usize;
            if i < fields.soot_density.len() && j < fields.soot_density[0].len() {
                let soot_density = fields.soot_density[i][j];  // kg/m³
                let extinction_coeff = 8700.0 * soot_density;  // m⁻¹ (Mulholland)
                let visibility = 3.0 / extinction_coeff;  // Seader-Ou, C=3 for light-reflecting
                return visibility;
            }
        }
        100.0  // Default: unlimited visibility
    }

    fn update_fed(&mut self, agent: &mut Agent) -> Result<(), EgressError> {
        if let Some(ref fields) = self.cfd_fields {
            let i = (agent.x / self.grid.dx) as usize;
            let j = (agent.y / self.grid.dy) as usize;

            if i < fields.temperature.len() && j < fields.temperature[0].len() {
                let temp = fields.temperature[i][j];  // °C
                let co_ppm = fields.co_concentration[i][j];

                // FED for CO (Purser, approximation)
                let fed_co_rate = co_ppm / 35000.0;  // Simplified LC50
                agent.fed_co += fed_co_rate * self.dt / 60.0;

                // FED for heat (convective, T > 60°C is threshold)
                let fed_heat_rate = if temp > 60.0 {
                    (temp - 60.0) / 40.0  // Simplified
                } else {
                    0.0
                };
                agent.fed_heat += fed_heat_rate * self.dt / 60.0;

                agent.fed_total = agent.fed_co + agent.fed_heat;
            }
        }
        Ok(())
    }

    fn select_exit(&self, agent: &Agent) -> (usize, usize) {
        if agent.familiar {
            // Choose nearest exit
            self.grid.exits.iter()
                .min_by_key(|(ex_i, ex_j)| {
                    let ex_x = *ex_i as f64 * self.grid.dx;
                    let ex_y = *ex_j as f64 * self.grid.dy;
                    let dx = ex_x - agent.x;
                    let dy = ex_y - agent.y;
                    ((dx * dx + dy * dy).sqrt() * 1000.0) as i64
                })
                .copied()
                .unwrap_or(self.grid.exits[0])
        } else {
            // Random exit
            use rand::seq::SliceRandom;
            *self.grid.exits.choose(&mut thread_rng()).unwrap()
        }
    }

    fn astar_path(&self, start_i: usize, start_j: usize, goal_i: usize, goal_j: usize) -> Vec<(usize, usize)> {
        // Simplified A* pathfinding
        // Full implementation would use priority queue and heuristic
        vec![(goal_i, goal_j)]  // Stub: direct path
    }

    fn random_walkable_position(grid: &EgressGrid, rng: &mut impl rand::Rng) -> (f64, f64) {
        loop {
            let i = rng.gen_range(0..grid.nx);
            let j = rng.gen_range(0..grid.ny);
            if !grid.obstacles[i][j] {
                return (i as f64 * grid.dx, j as f64 * grid.dy);
            }
        }
    }

    pub fn get_aset(&self, visibility_threshold: f64, temp_threshold: f64) -> f64 {
        // ASET = time when any location becomes untenable
        // This would query CFD fields over time
        // Stub: return 120s
        120.0
    }

    pub fn get_rset(&self) -> f64 {
        // RSET = time when last agent exits
        self.agents.iter()
            .filter(|a| a.state == AgentState::Exited)
            .map(|a| self.time)
            .max_by(|a, b| a.partial_cmp(b).unwrap())
            .unwrap_or(self.time)
    }
}

#[derive(Debug)]
pub enum EgressError {
    PathNotFound,
    InvalidPosition,
}

use crate::db::models::EgressParameters;
use rand::Rng;
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–4)

**Day 1: Project initialization and Rust backend scaffold**
```bash
cargo init firesim-api
cd firesim-api
cargo add axum tokio serde serde_json sqlx uuid chrono tower-http jsonwebtoken bcrypt
cargo add --features postgres,runtime-tokio-rustls,uuid,chrono,json sqlx
cargo add aws-sdk-s3 redis
```
- `src/main.rs` — Axum app with CORS, tracing, graceful shutdown
- `src/config.rs` — Environment-based configuration (DATABASE_URL, JWT_SECRET, S3_BUCKET, REDIS_URL)
- `src/state.rs` — AppState struct (PgPool, Redis, S3Client)
- `src/error.rs` — ApiError enum with IntoResponse
- `Dockerfile` — Multi-stage build (Rust builder + runtime with CUDA support)
- `docker-compose.yml` — PostgreSQL, Redis, MinIO (S3-compatible)

**Day 2: Database schema and migrations**
- `migrations/001_initial.sql` — All 11 tables: users, organizations, org_members, projects, simulations, simulation_jobs, design_fires, materials, fuel_loads, occupancy_parameters, usage_records, report_configs
- `src/db/mod.rs` — Database pool initialization
- `src/db/models.rs` — All SQLx structs with FromRow derives
- Run `sqlx migrate run` to apply schema
- Seed script for design fires (10 scenarios: office cubicle, retail display, upholstered furniture, atrium, parking, warehouse)
- Seed script for materials (steel, concrete, gypsum, timber, common fuels)

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

### Phase 2 — Geometry Builder + Mesher (Days 5–10)

**Day 5: Frontend scaffold and Three.js setup**
```bash
npm create vite@latest frontend -- --template react-ts
cd frontend
npm i zustand @tanstack/react-query axios three @react-three/fiber @react-three/drei
npm i -D @types/three
```
- `src/App.tsx` — Router, auth context, layout
- `src/stores/projectStore.ts` — Zustand store for project state
- `src/stores/geometryStore.ts` — Geometry state (rooms, obstructions, vents, doors)
- `src/components/GeometryEditor/Viewport3D.tsx` — Three.js canvas with orbit controls
- `src/components/GeometryEditor/Toolbar.tsx` — Tool palette (draw room, add obstruction, place vent/door)

**Day 6: CAD/DXF import and 2D floor plan viewer**
- `src/lib/dxf_parser.ts` — DXF file parser (use dxf-parser library)
- `src/components/GeometryEditor/FloorPlanImporter.tsx` — Upload DXF, preview 2D entities
- `src/components/GeometryEditor/FloorPlan2D.tsx` — SVG-based 2D floor plan viewer with pan/zoom
- Extract polylines, circles, arcs from DXF → convert to room boundaries
- Manual room detection: user clicks to confirm detected rectangles as rooms

**Day 7: 3D extrusion and obstruction placement**
- `src/components/GeometryEditor/RoomExtruder.tsx` — Specify floor-to-ceiling height, extrude rooms to 3D boxes
- `src/components/GeometryEditor/ObstructionTool.tsx` — Click-and-drag to place cuboid obstructions (furniture, equipment)
- `src/components/GeometryEditor/VentTool.tsx` — Place vents (openings, windows) on walls with area specification
- `src/components/GeometryEditor/DoorTool.tsx` — Place doors with width and discharge coefficient
- Transform controls (translate, rotate, scale) for editing 3D geometry in Three.js

**Day 8: Fire source placement and HRR curve editor**
- `src/components/FireEditor/FireSourceMarker.tsx` — 3D marker (flame icon) for fire location
- `src/components/FireEditor/HRRCurveEditor.tsx` — Graph editor for HRR vs. time (t-squared, prescribed, or design fire library)
- `src/components/FireEditor/DesignFirePicker.tsx` — Browse design fire library (office, retail, etc.), insert into project
- `src/components/FireEditor/FuelProperties.tsx` — Edit soot yield, CO yield, heat of combustion

**Day 9: Mesh generation — octree-based Cartesian meshing (Rust)**
- `mesher/src/octree.rs` — Octree spatial partitioning for adaptive refinement
- `mesher/src/cartesian_mesh.rs` — Generate uniform Cartesian cells, refine near obstructions/fire
- `mesher/src/boundary.rs` — Identify solid boundary cells (wall, obstruction) via embedded boundary method
- `mesher/src/export.rs` — Export mesh to custom binary format (cell coordinates, neighbor indices, boundary flags)
- API endpoint: `POST /api/projects/:id/generate-mesh` — generates mesh, stores in S3, returns cell count and preview

**Day 10: Mesh preview and validation in frontend**
- `src/components/MeshViewer/MeshPreview3D.tsx` — Three.js rendering of mesh cells (wireframe or solid with transparency)
- `src/components/MeshViewer/MeshStats.tsx` — Display cell count, min/max cell size, estimated memory usage
- Mesh quality checks: aspect ratio warnings, excessive refinement warnings
- User can iterate: adjust target cell size, refine fire region, regenerate mesh

### Phase 3 — Zone Model (Fast Preview) (Days 11–13)

**Day 11: Python zone model solver (two-zone CFAST-equivalent)**
```bash
cd zone-solver
python -m venv venv
source venv/bin/activate
pip install numpy scipy fastapi uvicorn
```
- `zone_solver/plume.py` — McCaffrey or Heskestad plume entrainment correlation
- `zone_solver/layer.py` — Upper (hot) and lower (cool) layer heat and mass balance ODEs
- `zone_solver/solver.py` — Solve ODEs (scipy.integrate.solve_ivp) for layer height, temperatures over time
- `zone_solver/main.py` — FastAPI endpoint `POST /solve_zone_model` accepting geometry, fire HRR, returns time-series

**Day 12: Zone model API integration**
- Rust backend calls Python zone solver via HTTP (or spawn Python subprocess)
- `src/solvers/zone_model.rs` — Wrapper that formats inputs, calls Python service, parses outputs
- Returns: time-series of smoke layer height, upper layer temperature, time to tenability threshold

**Day 13: Zone model results visualization**
- `src/components/Results/ZoneModelResults.tsx` — Line charts (Recharts) for layer height vs. time, temperature vs. time
- `src/components/Results/TenabilityTimeline.tsx` — Mark time when upper layer descends below 2m (ASET)
- Comparison table: analytical correlations vs. zone model predictions

### Phase 4 — CFD Solver Core (Days 14–24)

**Day 14: CUDA development setup and basic LES framework**
- Install CUDA toolkit 12.x, verify nvcc compiler
- `cfd-solver/Cargo.toml` — add cuda-runtime-sys, cudarc
- `cfd-solver/src/les_solver.rs` — LESGrid struct, device memory allocation
- `cfd-solver/cuda/advection.cu` — WENO5 advection kernel (temperature, mixture fraction)
- Build pipeline: compile .cu files with nvcc, link into Rust binary

**Day 15: Pressure-velocity coupling (projection method)**
- `cfd-solver/cuda/momentum.cu` — Momentum predictor kernel (advection + diffusion + buoyancy)
- `cfd-solver/src/poisson.rs` — Pressure Poisson solver using cuSPARSE conjugate gradient
- `cfd-solver/cuda/projection.cu` — Velocity corrector kernel (subtract pressure gradient)
- Test: lid-driven cavity (verify velocity field converges)

**Day 16: Combustion model (mixture fraction + fast chemistry)**
- `cfd-solver/cuda/combustion.cu` — Mixture fraction advection-diffusion
- Heat release Q̇ = Δh_c · χ · ∇²Z (simplified fast chemistry)
- Source term stamping: add Q̇ to energy equation at fire cells
- Test: verify plume buoyancy rise from point heat source

**Day 17: Radiation solver (gray gas FVM)**
- `cfd-solver/src/radiation.rs` — Finite volume radiation solver
- `cfd-solver/cuda/radiation.cu` — RTE sweep over discrete ordinates (S4 or S6 quadrature)
- Absorption coefficient κ from soot: κ = 1600·T·Y_soot (Planck mean, simplified)
- Radiative heat loss Q̇_rad = -∇·q_rad subtracted from energy equation
- Test: verify radiative cooling of hot gas layer

**Day 18: Soot and CO transport**
- `cfd-solver/cuda/species.cu` — Advection-diffusion for soot mass fraction Y_soot
- Soot generation: d(Y_soot)/dt = y_s · Q̇ / Δh_c at fire cells
- CO transport: advection-diffusion for Y_CO with CO generation from y_co
- Test: verify soot spread in plume, optical density calculation

**Day 19: Boundary conditions and embedded boundaries**
- `cfd-solver/src/boundary.rs` — No-slip wall BC, open vent BC, symmetry BC
- Embedded boundary method: ghost cells for curved/slanted obstructions
- `cfd-solver/cuda/boundary.cu` — Apply BC kernels (wall temperature, vent pressure)

**Day 20: Adaptive timestep and CFL stability**
- `cfd-solver/src/timestep.rs` — Compute CFL-limited timestep from max velocity on GPU
- Reduce kernel to find max |u|, |v|, |w| across entire grid
- Adaptive dt clamped to [1e-4, 0.1] seconds

**Day 21: Solver orchestration and checkpointing**
- `cfd-solver/src/solver.rs` — Main FireSolver::run() loop: initialize → timestep loop → output
- Output snapshots every N timesteps (e.g., every 1 second of simulation time)
- `cfd-solver/src/output.rs` — Write field data (T, u, v, w, soot, CO) to HDF5 or custom binary
- Checkpoint/restart capability: save solver state, resume from checkpoint

**Day 22: Sprinkler activation (RTI model)**
- `cfd-solver/src/sprinkler.rs` — RTI (Response Time Index) model for thermal element
- Sprinkler link temperature: dT_link/dt = √(u_ceiling) · (T_gas - T_link) / RTI
- Activation when T_link > T_activation (e.g., 68°C for standard sprinkler)
- Record activation time, pass to spray model

**Day 23: Water spray suppression (Lagrangian droplet model)**
- `cfd-solver/src/spray.rs` — Lagrangian particle tracking for water droplets
- `cfd-solver/cuda/spray.cu` — Droplet trajectory, evaporation, momentum/energy coupling to gas phase
- HRR suppression: reduce Q̇ by factor based on water application rate (empirical)
- Test: verify spray cools gas temperature, suppresses fire

**Day 24: Smoke detector activation (obscuration model)**
- `cfd-solver/src/detector.rs` — Photoelectric smoke detector: activation when optical density > threshold
- Optical density OD = κ · L (extinction coefficient × path length)
- Ionization detector: activation from soot particle count (simplified)
- Record detection times for alarm sequence

### Phase 5 — Egress Solver + Coupling (Days 25–29)

**Day 25: Egress grid and agent initialization**
- `egress-solver/src/agent_model.rs` — Agent struct, EgressGrid, AgentState enum
- `egress-solver/src/init.rs` — Distribute agents across walkable cells, sample pre-movement and speed from distributions
- 2D grid aligned with CFD horizontal plane (project 3D CFD to 2D slice at z=1.5m for visibility/temp)

**Day 26: Pathfinding (A*) and movement**
- `egress-solver/src/pathfinding.rs` — A* algorithm on 2D grid with obstacle avoidance
- `egress-solver/src/movement.rs` — Move agents along path, density-dependent speed (Fruin)
- Collision avoidance: prevent multiple agents from occupying same cell

**Day 27: CFD coupling (visibility and toxicity)**
- `egress-solver/src/cfd_coupling.rs` — Interpolate CFD fields (T, soot, CO) to agent positions
- Visibility calculation from soot: V = C / κ_ext
- FED accumulation: integrate CO, heat, HCN exposure over time
- Speed reduction when visibility < 10m (SFPE guidelines)

**Day 28: Exit flow and congestion**
- `egress-solver/src/exit_flow.rs` — Model exit discharge capacity (persons/m/s)
- Queue formation at doors, merge conflicts at stairwells
- Density limits: cap local density at 5-7 persons/m² (crush threshold)

**Day 29: ASET vs. RSET analysis**
- `egress-solver/src/tenability.rs` — Determine ASET from CFD (time to untenable conditions at any location)
- RSET = pre-movement + travel time (last agent out)
- Safety margin: ASET - RSET (should be positive with factor per code, e.g., 1.5x)
- `src/api/handlers/tenability.rs` — API endpoint returning ASET, RSET, margin, pass/fail

### Phase 6 — Structural Fire Module (Days 30–33)

**Day 30: Steel member temperature (lumped-mass method)**
- `structural-fire/src/steel_lumped.rs` — EN 1993-1-2 lumped-mass calculation
- Section factor A_m/V from member geometry (I-beam, column, tube)
- Heat flux h_net from CFD gas temperature and radiation
- Solve dθ/dt = (h_net · A_m/V) / (c · ρ)

**Day 31: 2D FEM heat transfer (concrete, complex sections)**
- `structural-fire/src/fem_heat.rs` — 2D finite element heat conduction solver
- Triangular mesh of cross-section, boundary conditions from fire exposure
- Material properties: temperature-dependent conductivity, specific heat
- Moisture evaporation plateau at 100°C for concrete

**Day 32: Fire exposure curves (ISO 834, natural fire)**
- `structural-fire/src/fire_curves.rs` — Standard fire: T = 20 + 345·log₁₀(8t + 1)
- Hydrocarbon: T = 20 + 1080·(1 - 0.325·e^(-0.167t) - 0.675·e^(-2.5t))
- Natural fire from CFD: query gas temperature time-history at member location, apply as BC

**Day 33: Structural capacity assessment (temperature-reduced strength)**
- `structural-fire/src/capacity.rs` — Eurocode reduction factors k_y(T), k_E(T) for yield strength, modulus
- Calculate reduced capacity: M_fi = k_y · M_ambient, N_fi = k_y · N_ambient
- Comparison to applied loads (user input or simplified gravity load estimate)
- Pass/fail for fire resistance duration

### Phase 7 — Worker + Job Orchestration (Days 34–36)

**Day 34: Simulation worker and GPU job scheduling**
- `src/workers/simulation_worker.rs` — Redis job consumer, launches CFD/egress solvers
- AWS EC2 instance management: launch g5.xlarge on-demand or spot for jobs
- `src/workers/gpu_manager.rs` — Track available GPU workers, assign jobs by priority
- Progress streaming: worker publishes timestep progress → Redis pub/sub → WebSocket to client

**Day 35: S3 result upload and caching**
- Worker writes field data to S3 after simulation completes
- `src/api/handlers/results.rs` — Presigned S3 URLs for client download
- Redis cache: store recent results (last 24h) for fast access

**Day 36: WebSocket for live progress**
- `src/api/ws/mod.rs` — Axum WebSocket upgrade handler
- `src/api/ws/simulation_progress.rs` — Subscribe client to simulation progress channel
- Client receives: `{ progress_pct, current_timestep, max_temp, smoke_layer_height, cfl }`
- Frontend: `src/hooks/useSimulationProgress.ts` — React hook for WebSocket subscription

### Phase 8 — Results Visualization + Reporting (Days 37–42)

**Day 37: 3D volumetric smoke rendering (Three.js)**
- `src/components/Results/VolumetricRenderer.tsx` — Three.js volumetric raymarching shader
- Load 3D soot density field from S3, convert to 3D texture
- Opacity transfer function: soot density → alpha channel
- Interactive fly-through with orbit controls

**Day 38: Slice planes and isosurfaces**
- `src/components/Results/SlicePlane.tsx` — Temperature/velocity/toxicity slice at user-defined Z height
- Color mapping: temperature (blue → red), velocity (vector arrows), CO concentration (heatmap)
- `src/components/Results/Isosurface.tsx` — Marching cubes for visibility boundary (10m visibility surface)

**Day 39: Egress animation and congestion heatmaps**
- `src/components/Results/EgressAnimation.tsx` — Animated agents moving along paths with timestamps
- Agent color coding: green (moving), yellow (slowed by visibility), red (incapacitated)
- `src/components/Results/CongestionHeatmap.tsx` — Density heatmap overlaid on floor plan
- Cumulative evacuation curve: number of agents exited vs. time

**Day 40: Tenability timeline and measurements**
- `src/components/Results/TenabilityTimeline.tsx` — Timeline showing detection, alarm, ASET, RSET
- Key locations (exits, stairs): plot temperature, visibility, FED vs. time
- Automated ASET determination: time when any point exceeds tenability criteria

**Day 41: PDF report generation**
- `src/services/report_generator.rs` — Use HTML-to-PDF library (headless Chrome or wkhtmltopdf)
- Report sections: project info, fire scenario, mesh details, ASET/RSET summary, structural assessment, code compliance matrix
- Include snapshots (smoke spread images, evacuation curves, temperature plots)
- Templated HTML with dynamic data injection

**Day 42: Code compliance matrix and multi-scenario comparison**
- `src/components/Results/ComplianceMatrix.tsx` — Table checking NFPA 101, IBC requirements
- Travel distance, exit capacity, ASET margin, detector spacing
- Multi-scenario table: compare ASET/RSET across different fire locations/sizes
- Export comparison to CSV

---

## Critical Files

```
firesim/
├── backend/
│   ├── Cargo.toml
│   ├── src/
│   │   ├── main.rs                          # Axum app entry, routes
│   │   ├── config.rs                        # Environment config
│   │   ├── state.rs                         # AppState (DB, Redis, S3)
│   │   ├── error.rs                         # ApiError enum
│   │   ├── auth/
│   │   │   ├── mod.rs                       # JWT middleware
│   │   │   └── oauth.rs                     # Google/GitHub OAuth
│   │   ├── db/
│   │   │   ├── mod.rs                       # DB pool init
│   │   │   └── models.rs                    # SQLx structs (User, Project, Simulation, etc.)
│   │   ├── api/
│   │   │   ├── router.rs                    # Route definitions
│   │   │   ├── handlers/
│   │   │   │   ├── auth.rs                  # Register, login, OAuth
│   │   │   │   ├── projects.rs              # Project CRUD
│   │   │   │   ├── simulation.rs            # Create/get simulation, results
│   │   │   │   ├── tenability.rs            # ASET/RSET analysis
│   │   │   │   ├── billing.rs               # Stripe integration
│   │   │   │   └── design_fires.rs          # Design fire library API
│   │   │   └── ws/
│   │   │       └── simulation_progress.rs   # WebSocket progress stream
│   │   ├── workers/
│   │   │   ├── simulation_worker.rs         # Redis job consumer, launches solvers
│   │   │   └── gpu_manager.rs               # GPU instance management
│   │   ├── solvers/
│   │   │   └── zone_model.rs                # Zone model wrapper (calls Python)
│   │   └── services/
│   │       └── report_generator.rs          # PDF report generation
│   └── migrations/
│       └── 001_initial.sql                  # Database schema
├── cfd-solver/
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── les_solver.rs                    # Main CFD solver (LESGrid, FireSolver)
│   │   ├── poisson.rs                       # Pressure Poisson solver (cuSPARSE CG)
│   │   ├── radiation.rs                     # Gray gas FVM radiation
│   │   ├── sprinkler.rs                     # RTI activation model
│   │   ├── spray.rs                         # Lagrangian droplet spray
│   │   ├── detector.rs                      # Smoke detector activation
│   │   ├── boundary.rs                      # Boundary conditions, embedded boundaries
│   │   ├── timestep.rs                      # Adaptive CFL timestep
│   │   └── output.rs                        # Field data output (HDF5/binary)
│   └── cuda/
│       ├── advection.cu                     # WENO5 advection kernel
│       ├── momentum.cu                      # Momentum predictor kernel
│       ├── projection.cu                    # Velocity corrector kernel
│       ├── combustion.cu                    # Combustion source term kernel
│       ├── radiation.cu                     # RTE sweep kernel
│       ├── species.cu                       # Soot/CO transport kernel
│       ├── boundary.cu                      # Boundary condition kernels
│       └── spray.cu                         # Droplet tracking kernel
├── egress-solver/
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── agent_model.rs                   # Agent struct, EgressSolver, movement
│   │   ├── pathfinding.rs                   # A* pathfinding
│   │   ├── cfd_coupling.rs                  # Interpolate CFD fields to agents
│   │   ├── exit_flow.rs                     # Exit discharge, congestion
│   │   └── tenability.rs                    # ASET determination, FED
├── structural-fire/
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── steel_lumped.rs                  # Lumped-mass steel temperature (EN 1993-1-2)
│   │   ├── fem_heat.rs                      # 2D FEM heat transfer
│   │   ├── fire_curves.rs                   # ISO 834, hydrocarbon, natural
│   │   └── capacity.rs                      # Temperature-reduced strength
├── mesher/
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── octree.rs                        # Octree spatial partitioning
│   │   ├── cartesian_mesh.rs                # Cartesian mesh generation
│   │   ├── boundary.rs                      # Identify solid boundaries
│   │   └── export.rs                        # Export mesh to binary
├── zone-solver/
│   ├── requirements.txt
│   ├── zone_solver/
│   │   ├── plume.py                         # Plume entrainment correlations
│   │   ├── layer.py                         # Two-zone heat/mass balance ODEs
│   │   ├── solver.py                        # ODE solver (scipy.integrate)
│   │   └── main.py                          # FastAPI endpoint
├── frontend/
│   ├── package.json
│   ├── src/
│   │   ├── App.tsx                          # Main app, router
│   │   ├── stores/
│   │   │   ├── projectStore.ts              # Zustand project state
│   │   │   └── geometryStore.ts             # Geometry state
│   │   ├── components/
│   │   │   ├── GeometryEditor/
│   │   │   │   ├── Viewport3D.tsx           # Three.js viewport
│   │   │   │   ├── FloorPlanImporter.tsx    # DXF import
│   │   │   │   ├── RoomExtruder.tsx         # 3D extrusion
│   │   │   │   ├── ObstructionTool.tsx      # Place obstructions
│   │   │   │   └── VentTool.tsx             # Place vents/doors
│   │   │   ├── FireEditor/
│   │   │   │   ├── FireSourceMarker.tsx     # Fire location marker
│   │   │   │   ├── HRRCurveEditor.tsx       # HRR curve graph editor
│   │   │   │   └── DesignFirePicker.tsx     # Design fire library picker
│   │   │   ├── MeshViewer/
│   │   │   │   ├── MeshPreview3D.tsx        # Mesh visualization
│   │   │   │   └── MeshStats.tsx            # Mesh statistics
│   │   │   ├── Results/
│   │   │   │   ├── ZoneModelResults.tsx     # Zone model charts
│   │   │   │   ├── VolumetricRenderer.tsx   # 3D volumetric smoke (Three.js raymarching)
│   │   │   │   ├── SlicePlane.tsx           # Temperature/velocity/CO slice
│   │   │   │   ├── Isosurface.tsx           # Marching cubes visibility boundary
│   │   │   │   ├── EgressAnimation.tsx      # Animated agent evacuation
│   │   │   │   ├── CongestionHeatmap.tsx    # Density heatmap
│   │   │   │   ├── TenabilityTimeline.tsx   # ASET/RSET timeline
│   │   │   │   └── ComplianceMatrix.tsx     # Code compliance table
│   │   │   └── billing/
│   │   │       ├── PlanCard.tsx             # Plan comparison cards
│   │   │       └── UsageMeter.tsx           # Compute hours usage bar
│   │   ├── hooks/
│   │   │   └── useSimulationProgress.ts     # WebSocket progress hook
│   │   └── lib/
│   │       └── dxf_parser.ts                # DXF file parser
│   └── public/
├── Dockerfile.backend                       # Multi-stage: Rust + CUDA runtime
├── Dockerfile.cfd-solver                    # CUDA-enabled CFD worker
├── docker-compose.yml                       # PostgreSQL, Redis, MinIO
└── .github/
    └── workflows/
        ├── backend.yml                      # Rust tests, Docker build
        └── cfd-build.yml                    # CUDA build, push solver image
```

---

## Solver Validation Suite

### Benchmark 1: Zone Model vs. Analytical — Plume Entrainment

**Scenario:** 500 kW steady fire in 10m x 10m x 4m room with single door (0.9m x 2m).

**Zone model prediction:**
- Plume mass flow rate at ceiling: ṁ_p = 0.071 · Q_c^(1/3) · z^(5/3) where Q_c = χ_c · Q
- For χ_c = 0.7 (convective fraction), Q_c = 350 kW, z = 4m: ṁ_p ≈ 2.1 kg/s
- Smoke layer descent: z_interface = 4 - (ṁ_p · t) / (ρ · A_room) until vent opening
- Expected: layer descends to 2m (head height) at ~90 seconds

**Validation:**
- Run zone solver with Q = 500 kW constant
- Check layer height z(t=90s) ≈ 2.0 ± 0.1m
- Check upper layer temperature T_upper(90s) ≈ 200-250°C (from heat balance)

**Pass criteria:** Layer height within 10% of analytical, temperature within 20%.

### Benchmark 2: CFD Fire — Plume Centerline Temperature

**Scenario:** 100 kW axisymmetric plume in open space (no ceiling), compare to McCaffrey correlation.

**McCaffrey correlation (centerline temperature excess):**
```
ΔT_cl = 22.0 · Q^(2/3) / z^(5/3)  (for z/Q^(2/5) < 0.2, flame region)
```
For Q = 100 kW, z = 1m: ΔT_cl ≈ 475 K

**CFD setup:**
- 20m x 20m x 20m domain, fire at center base
- Mesh: 0.1m cells near fire, 0.5m cells far field (~200K cells)
- Run to 60s (quasi-steady state)

**Expected:**
- Centerline temperature at z=1m: T ≈ 20 + 475 = 495°C
- Plume velocity at z=1m: u_z ≈ 1.5-2.0 m/s

**Pass criteria:** T_centerline(z=1m) = 495 ± 50°C, u_z = 1.8 ± 0.3 m/s.

### Benchmark 3: Egress — SFPE Evacuation Time

**Scenario:** 100 occupants evacuating from 20m x 20m room via single 1m-wide door.

**SFPE hydraulic model:**
- Specific flow rate: F_s ≈ 1.3 persons/(m·s) for corridor
- Door capacity: N = F_s · W · t = 1.3 · 1.0 · t
- Time to clear 100 people: t = 100 / 1.3 ≈ 77 seconds (plus pre-movement)
- With pre-movement mean 30s: RSET ≈ 107 seconds

**Agent-based simulation:**
- 100 agents, pre-movement lognormal(mean=30s, std=10s)
- Speed 1.2 m/s unimpeded, density-dependent reduction at door

**Expected:**
- Last agent exits at t ≈ 100-120 seconds
- Peak density at door: 3-4 persons/m²

**Pass criteria:** RSET = 110 ± 15 seconds, peak density < 5 persons/m².

### Benchmark 4: Structural Fire — Steel Temperature Rise

**Scenario:** Unprotected steel I-beam (W14x90) exposed to ISO 834 standard fire.

**Analytical (lumped-mass EN 1993-1-2):**
- Section factor A_m/V ≈ 140 m⁻¹ (I-beam, 4-sided exposure)
- Net heat flux h_net = h_conv·(T_fire - T_steel) + εσ(T_fire⁴ - T_steel⁴)
- Solve dθ/dt = (h_net · A_m/V) / (c_steel · ρ_steel)
- At t=30 min, ISO 834 fire T_fire = 842°C, steel T_steel ≈ 750-800°C

**FireSim lumped-mass solver:**
- Input: W14x90 section, 4-sided exposure, ISO 834 curve
- Output: T_steel(t) for t = 0 to 60 min

**Expected:** T_steel(30min) = 775 ± 30°C

**Pass criteria:** Within 5% of Eurocode reference calculation.

### Benchmark 5: Coupled CFD-Egress — Smoke-Impaired Evacuation

**Scenario:** 1 MW fire in 30m x 30m atrium, 200 occupants, smoke reduces visibility, compare ASET vs. RSET.

**CFD:**
- Run to 180s, track visibility at 2m height across floor area
- ASET = time when visibility < 10m anywhere along egress path
- Expected ASET ≈ 100-120 seconds (smoke spreads rapidly in open atrium)

**Egress:**
- 200 agents, pre-movement mean 45s (alert office), path to 4 exits
- Speed reduced when visibility < 10m (coupling from CFD)
- Expected RSET ≈ 140-160 seconds

**Safety margin:** ASET - RSET should be negative (unsafe condition), demonstrating need for smoke control or faster egress.

**Pass criteria:** ASET < RSET (identifies hazard), agent speeds reduced >30% when V < 10m.

---

## Verification Checklist

**CFD Solver:**
- [ ] Mass conservation: ∫(∇·u) dV < 1e-6 per timestep
- [ ] Energy conservation: total energy (thermal + kinetic) conserved within 1% (excluding radiation losses)
- [ ] CFL stability: no blow-up for CFL < 1.0, solver runs to completion
- [ ] Grid independence: results change <5% when mesh refined by 2x
- [ ] Plume centerline temperature matches McCaffrey within 10%
- [ ] Radiation heat flux at walls matches Stefan-Boltzmann within 15%
- [ ] Sprinkler activation time within 10% of FDTs/DETACT correlations
- [ ] Smoke layer interface height matches zone model for simple geometries

**Egress Solver:**
- [ ] Agent pathfinding: all agents reach exits (no stuck agents)
- [ ] Density-speed correlation: Fruin curve reproduced within 10%
- [ ] FED accumulation: ISO 13571 CO model matches reference within 5%
- [ ] RSET matches SFPE hydraulic model for simple corridor within 15%
- [ ] No agents overlap (collision avoidance working)
- [ ] Pre-movement distribution: lognormal sampling verified with chi-squared test

**Structural Fire:**
- [ ] Lumped-mass steel temperature: EN 1993-1-2 examples reproduced within 5%
- [ ] 2D FEM concrete temperature: EN 1992-1-2 benchmark within 10%
- [ ] ISO 834 fire curve: T(30min) = 842°C ± 1°C
- [ ] Strength reduction factors k_y(T) match Eurocode tables

**Integration:**
- [ ] CFD → Egress coupling: visibility field interpolated correctly (spot checks)
- [ ] CFD → Structural coupling: gas temperature time-history extracted at member locations
- [ ] Mesh generation: no orphan cells, all cells have 6 neighbors (Cartesian) or flagged as boundary
- [ ] S3 upload/download: field data integrity verified with checksums
- [ ] WebSocket progress: updates received every 1-2 seconds during simulation

**Performance:**
- [ ] Zone model completes in <5 seconds for single-room scenario
- [ ] CFD 100K cells: runtime <30 minutes for 180s simulation on g5.xlarge (A10G GPU)
- [ ] CFD 500K cells: runtime <2 hours on g5.xlarge
- [ ] Egress 500 agents: runtime <10 seconds for 300s evacuation
- [ ] Mesh generation 500K cells: <30 seconds
- [ ] Report PDF generation: <20 seconds including plots

---

## Deployment Architecture

```
User Browser ←→ CloudFront (CDN) ←→ React Frontend (S3 static hosting)
                                          ↓
                                      Axum API (ECS Fargate)
                                          ↓
                    ┌─────────────────────┼─────────────────────┐
                    ↓                     ↓                     ↓
              PostgreSQL RDS         Redis ElastiCache      S3 Buckets
            (metadata, users)      (job queue, cache)    (meshes, results)
                                          ↓
                                  Simulation Workers (EC2 GPU instances)
                                          ↓
                      ┌───────────────────┼───────────────────┐
                      ↓                   ↓                   ↓
              CFD Solver (CUDA)    Egress Solver     Structural Solver
                  (g5.xlarge)         (CPU)              (CPU)
```

**Infrastructure:**
- **Frontend:** React SPA hosted on S3, served via CloudFront CDN
- **API:** Rust Axum backend on ECS Fargate (auto-scaling 2-10 tasks)
- **Database:** PostgreSQL 16 on RDS (db.t3.large, Multi-AZ for production)
- **Cache:** Redis on ElastiCache (cache.t3.medium)
- **Storage:** S3 buckets (firesim-meshes, firesim-results, firesim-reports) with lifecycle policies (delete old results after 90 days)
- **Workers:** EC2 GPU instances (g5.xlarge with A10G for CFD, c5.4xlarge for egress/structural)
  - Spot instances for non-urgent jobs (80% cost savings)
  - On-demand instances for Advanced plan (guaranteed availability)
  - Auto Scaling Group launches instances when Redis job queue depth > 5

**Monitoring:**
- Prometheus + Grafana for infrastructure metrics (CPU, GPU utilization, memory, queue depth)
- Sentry for error tracking (frontend and backend)
- CloudWatch Logs for centralized logging
- Custom metrics: simulation completion rate, average runtime per cell count, ASET/RSET distribution

**CI/CD:**
- GitHub Actions for Rust backend (cargo test, clippy, build Docker image)
- GitHub Actions for CUDA solver (nvcc compile, push worker image to ECR)
- Frontend: Vite build, upload to S3, invalidate CloudFront cache
- Database migrations: run via CI/CD on merge to main (sqlx migrate run)

---

## Post-MVP Roadmap

### v1.1 — Advanced Fire Physics (Weeks 14-18)
- **Pyrolysis fire spread model:** Arrhenius decomposition for wall linings, upholstered furniture — predict fire spread across surfaces (not just prescribed HRR)
- **Mechanical smoke control:** HVAC duct network model, exhaust fans, pressurization systems, makeup air — integrate with CFD solver
- **Multi-mesh CFD:** Domain decomposition across multiple GPUs for >10M cell simulations (large buildings, long tunnels)
- **Gaseous suppression:** Clean agent (FM-200, Novec 1230) concentration buildup, inerting thresholds, hold time calculation
- **Travelling fire methodology:** Non-uniform fire exposure moving across floor plate (large open-plan offices per MCS, Rein)

### v1.2 — BIM Integration and Collaboration (Weeks 19-22)
- **IFC import:** Parse IFC files (Revit, ArchiCAD) to extract geometry, detect rooms/doors/stairs automatically
- **Revit plugin:** Direct export from Revit to FireSim with material assignments
- **Real-time collaboration:** Multiple users editing same project (operational transform or CRDT)
- **AHJ review portal:** Shareable links for authorities, comment threads on simulation results, approval workflow
- **API access:** RESTful API for programmatic simulation submission (integrate with BIM automation pipelines)

### v1.3 — Probabilistic Fire Risk and Optimization (Weeks 23-27)
- **Monte Carlo fire risk:** Sample fire location, HRR, pre-movement distributions, run 100-1000 scenarios, generate probability distributions for ASET, RSET
- **Sensitivity analysis:** Vary fire growth rate, detector spacing, occupant density — identify critical parameters
- **Design optimization:** Automated placement of detectors, sprinklers, exits to minimize RSET or maximize ASET margin (genetic algorithm or gradient-free optimization)
- **Fire load database expansion:** Add 50+ occupancy types with fuel load distributions from literature (SFPE, Eurocode, local codes)

### v1.4 — Structural Fire Capacity and Advanced Analysis (Weeks 28-32)
- **Structural capacity assessment:** Temperature-dependent moment-rotation curves, connection behavior, plastic hinge formation
- **Travelling fire analysis:** Automated worst-case fire path identification for large floor plates
- **Concrete spalling model:** Moisture transport, pressure buildup, spalling risk assessment
- **Timber charring:** Advanced charring rate (EN 1995-1-2), effective cross-section reduction, gypsum protection
- **Performance-based design reports:** Automated generation of fire engineering briefs per BS 7974 / SFPE workflow

### v1.5 — WUI Fire and Outdoor Scenarios (Weeks 33-38)
- **Wildland-urban interface (WUI) fire:** Vegetation fuel models (grass, shrub, timber), surface fire spread (Rothermel), ember transport
- **Wind-driven fire:** Ambient wind profiles, topographic slope effects on fire spread rate
- **Facade fire spread:** External wall cladding fire propagation (post-Grenfell), BS 8414 compliance testing
- **Tunnel fire:** Longitudinal ventilation, critical velocity, backlayering length (PIARC/NFPA 502)
- **Vehicle fire:** Car park fire scenarios, fire spread car-to-car, structural fire exposure to columns

### v1.6 — Training and Certification Program (Weeks 39-42)
- **FireSim Academy:** Video courses on fire engineering fundamentals, software tutorials, case studies
- **Certification program:** FireSim Certified Fire Engineer credential (exam-based, CPD points)
- **Webinar series:** Monthly live webinars with fire engineering experts, Q&A on complex scenarios
- **Example project library:** 50+ validated example projects (high-rise, atrium, stadium, data center) for training and benchmarking
