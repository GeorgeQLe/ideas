# 71. TurboGen — Turbomachinery Design and Analysis Platform

## Implementation Plan

**MVP Scope:** Web-based turbomachinery design tool with 1D meanline analysis for single-stage axial compressor and turbine supporting velocity triangle construction (blade speed, absolute/relative velocities, flow angles, work coefficient, flow coefficient, degree of reaction) calculated from cycle requirements, loss estimation via Ainley-Mathieson (turbines) and Lieblein-Koch-Smith (compressors) correlations with performance metrics (efficiency, pressure ratio, loading), parameterized 2D blade profile generator supporting NACA 65 and C4 profiles with camber line and thickness distribution control rendered as SVG with geometric validation, 3D blade stacking editor for spanwise variation of camber/stagger/thickness with linear/parabolic lean and 3D visualization via Three.js, simplified single-passage steady RANS CFD with structured O-mesh generation compiled to WASM for client-side execution of problems ≤50K cells and server-side native Rust execution for larger meshes, turbulence model support (SA, k-ω SST), performance extraction (mass flow, pressure ratio, efficiency, stage loading), interactive meridional view with stage geometry and flow properties, STEP/IGES export of blade geometry, Stripe billing with three tiers (Academic $99/month / Pro $299/month / Enterprise custom).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| CFD Solver | Rust (native + WASM) | Custom 3D RANS solver with SA/k-ω SST turbulence, structured mesh |
| WASM Compilation | `wasm-pack` + `wasm-bindgen` | Client-side CFD for meshes ≤50K cells |
| Meanline Solver | Rust (native + WASM) | Velocity triangle calculations, loss correlations |
| Scientific Services | Python 3.12 (FastAPI) | Blade profile optimization, mesh quality metrics |
| Database | PostgreSQL 16 | Projects, users, simulations, blade libraries |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | Blade geometries (STEP/IGES), CFD results, mesh files |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| 3D Visualization | Three.js + React Three Fiber | Blade geometry, meridional view, flow field rendering |
| 2D Blade Editor | SVG + Canvas | Profile editing with Bezier control points |
| CFD Visualization | WebGL 2.0 (custom shaders) | Contour plots, vector fields, streamlines |
| Real-time | WebSocket (Axum) | Live CFD convergence monitoring, residual plots |
| Job Queue | Redis 7 + Tokio tasks | Server-side CFD job management, mesh generation |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | WASM bundle delivery, static frontend assets |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, error tracking |
| CI/CD | GitHub Actions | WASM build pipeline, Rust tests, Docker image push |

### Key Architecture Decisions

1. **Hybrid client/server CFD solver with WASM threshold at 50K cells**: Single-passage steady RANS simulations with ≤50K cells (covers 90%+ of preliminary design iterations) run entirely in the browser via WASM, providing instant feedback with zero server cost. Larger meshes and multi-stage simulations are submitted to the Rust-native server solver with MPI parallelization. The threshold is configurable per plan tier.

2. **Custom turbomachinery CFD solver rather than wrapping OpenFOAM**: Building a custom structured-mesh RANS solver in Rust gives us full control over WASM compilation, rotating reference frames, mixing plane interfaces, and turbomachinery-specific boundary conditions. OpenFOAM's C++ codebase is 2M+ LOC with entrenched global state that prevents WASM compilation and thread-safe parallelization. Our Rust solver uses proven algorithms (SIMPLE/SIMPLEC pressure-velocity coupling, Rhie-Chow interpolation) while maintaining memory safety.

3. **Meanline and blade design decoupled from CFD mesh generation**: The meanline module produces velocity triangles and preliminary blade shapes that feed into the 3D blade designer. The blade designer exports analytical NURBS surfaces that are meshed on-demand. This separation allows rapid design iterations (meanline + blade shape in <1s) before committing to expensive CFD validation.

4. **Three.js for blade visualization, WebGL for CFD post-processing**: Three.js provides high-quality 3D rendering of blade surfaces with PBR materials, environment mapping, and interactive camera controls. CFD results (pressure/temperature contours, velocity vectors, streamlines) are rendered via custom WebGL shaders that handle 1M+ triangles with GPU-accelerated colormapping and isosurface extraction.

5. **S3 for geometry storage with PostgreSQL metadata catalog**: Blade geometries (STEP/IGES files, typically 100KB-5MB) and CFD meshes (binary, 10MB-500MB) are stored in S3, while PostgreSQL holds searchable metadata (stage type, flow coefficient, loading, pressure ratio, efficiency). This allows the blade library to scale to 10K+ designs without bloating the database while enabling fast parametric search via PostgreSQL indexes.

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

-- Organizations
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

-- Projects
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    machine_type TEXT NOT NULL,
    design_data JSONB NOT NULL DEFAULT '{}',
    settings JSONB DEFAULT '{}',
    is_public BOOLEAN NOT NULL DEFAULT false,
    forked_from UUID REFERENCES projects(id) ON DELETE SET NULL,
    thumbnail_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);
CREATE INDEX projects_org_idx ON projects(org_id);
CREATE INDEX projects_type_idx ON projects(machine_type);
CREATE INDEX projects_updated_idx ON projects(updated_at DESC);

-- Meanline Analysis Runs
CREATE TABLE meanline_runs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    status TEXT NOT NULL DEFAULT 'pending',
    input_parameters JSONB NOT NULL,
    results JSONB,
    error_message TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at TIMESTAMPTZ
);
CREATE INDEX meanline_project_idx ON meanline_runs(project_id);
CREATE INDEX meanline_user_idx ON meanline_runs(user_id);
CREATE INDEX meanline_created_idx ON meanline_runs(created_at DESC);

-- Blade Geometry Library
CREATE TABLE blade_geometries (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID REFERENCES projects(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT,
    machine_type TEXT NOT NULL,
    profile_type TEXT NOT NULL,
    geometry_parameters JSONB NOT NULL,
    step_url TEXT,
    iges_url TEXT,
    preview_image_url TEXT,
    tags TEXT[] DEFAULT '{}',
    is_public BOOLEAN NOT NULL DEFAULT false,
    is_validated BOOLEAN NOT NULL DEFAULT false,
    uploaded_by UUID REFERENCES users(id) ON DELETE SET NULL,
    download_count INTEGER DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX blade_machine_idx ON blade_geometries(machine_type);
CREATE INDEX blade_profile_idx ON blade_geometries(profile_type);
CREATE INDEX blade_name_trgm_idx ON blade_geometries USING gin(name gin_trgm_ops);
CREATE INDEX blade_tags_idx ON blade_geometries USING gin(tags);
CREATE INDEX blade_public_idx ON blade_geometries(is_public) WHERE is_public = true;

-- CFD Simulations
CREATE TABLE simulations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    simulation_type TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending',
    execution_mode TEXT NOT NULL DEFAULT 'wasm',
    mesh_parameters JSONB NOT NULL,
    solver_parameters JSONB NOT NULL,
    boundary_conditions JSONB NOT NULL,
    cell_count INTEGER,
    mesh_url TEXT,
    results_url TEXT,
    results_summary JSONB,
    convergence_history JSONB,
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

-- Server CFD Jobs
CREATE TABLE simulation_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    simulation_id UUID NOT NULL REFERENCES simulations(id) ON DELETE CASCADE,
    worker_id TEXT,
    cores_allocated INTEGER DEFAULT 4,
    memory_mb INTEGER DEFAULT 16384,
    priority INTEGER DEFAULT 0,
    progress_pct REAL DEFAULT 0.0,
    current_iteration INTEGER DEFAULT 0,
    max_iterations INTEGER DEFAULT 1000,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX jobs_sim_idx ON simulation_jobs(simulation_id);
CREATE INDEX jobs_status_idx ON simulation_jobs(worker_id);

-- Visualization Configurations
CREATE TABLE visualization_configs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    simulation_id UUID NOT NULL REFERENCES simulations(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    config_type TEXT NOT NULL,
    parameters JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX viz_sim_idx ON visualization_configs(simulation_id);

-- Usage Tracking
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,
    quantity REAL NOT NULL,
    period_start DATE NOT NULL,
    period_end DATE NOT NULL,
    metadata JSONB DEFAULT '{}',
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
    pub machine_type: String,
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
    pub simulation_type: String,
    pub status: String,
    pub execution_mode: String,
    pub mesh_parameters: serde_json::Value,
    pub solver_parameters: serde_json::Value,
    pub boundary_conditions: serde_json::Value,
    pub cell_count: Option<i32>,
    pub mesh_url: Option<String>,
    pub results_url: Option<String>,
    pub results_summary: Option<serde_json::Value>,
    pub convergence_history: Option<serde_json::Value>,
    pub error_message: Option<String>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub wall_time_ms: Option<i32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct MeanlineInput {
    pub machine_type: MachineType,
    pub inlet_total_pressure: f64,
    pub inlet_total_temperature: f64,
    pub mass_flow_rate: f64,
    pub rotational_speed: f64,
    pub hub_radius: f64,
    pub tip_radius: f64,
    pub flow_coefficient: f64,
    pub work_coefficient: f64,
    pub reaction_degree: f64,
    pub number_of_blades: u32,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum MachineType {
    AxialTurbine,
    AxialCompressor,
    CentrifugalCompressor,
    Pump,
    Fan,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct VelocityComponents {
    pub absolute_velocity: f64,
    pub relative_velocity: f64,
    pub axial_velocity: f64,
    pub tangential_velocity: f64,
    pub absolute_flow_angle: f64,
    pub relative_flow_angle: f64,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct StagePerformance {
    pub pressure_ratio: f64,
    pub temperature_ratio: f64,
    pub isentropic_efficiency: f64,
    pub polytropic_efficiency: f64,
    pub specific_work: f64,
    pub power: f64,
    pub loading_coefficient: f64,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct CFDSolverParameters {
    pub turbulence_model: TurbulenceModel,
    pub max_iterations: u32,
    pub convergence_tolerance: f64,
    pub cfl_number: f64,
    pub discretization: DiscretizationScheme,
    pub pressure_velocity_coupling: String,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum TurbulenceModel {
    SpalartAllmaras,
    KOmegaSST,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum DiscretizationScheme {
    Upwind,
    Muscl,
    Quick,
}
```

---

## Solver Architecture Deep-Dive

### Governing Equations and Discretization

TurboGen's CFD core implements the **Reynolds-Averaged Navier-Stokes (RANS)** equations for compressible flow in rotating reference frames. For a control volume rotating at angular velocity **Ω**, the governing equations in conservative form are:

```
∂ρ/∂t + ∇·(ρV_rel) = 0                                    (Continuity)

∂(ρV_rel)/∂t + ∇·(ρV_rel⊗V_rel) = -∇p + ∇·τ + ρf_rot     (Momentum)

∂(ρE)/∂t + ∇·((ρE + p)V_rel) = ∇·(k∇T) + ∇·(τ·V_rel)    (Energy)
```

Where:
- **V_rel** = V_abs - Ω × r (relative velocity in rotating frame)
- **f_rot** = -2Ω × V_rel - Ω × (Ω × r) (Coriolis and centrifugal forces)
- **τ** = (μ + μ_t)(∇V + (∇V)^T - (2/3)(∇·V)I) (viscous + turbulent stress)
- **E** = e + V²/2 (total energy per unit mass)

**Spatial discretization** uses finite volume method on structured hexahedral meshes:

```
∫_V (∂U/∂t) dV + ∮_S (F_conv - F_diff) · n dS = ∫_V S dV

where U = [ρ, ρV_rel, ρE]^T
      F_conv = [ρV_rel, ρV_rel⊗V_rel + pI, (ρE + p)V_rel]^T
      F_diff = [0, τ, k∇T + τ·V_rel]^T
```

**Convective fluxes** use Rhie-Chow momentum interpolation:

```
V_f = (V_f)_interp + (1/a_P)[∇p_cell - (∇p)_f]
```

**Turbulence closure** via Spalart-Allmaras:

```
∂(ρν̃)/∂t + ∇·(ρV_rel ν̃) = C_b1 S̃ ρν̃ + (1/σ)∇·((μ + ρν̃)∇ν̃) - C_w1 f_w ρ(ν̃/d)²

where S̃ = S + (ν̃/κ²d²)f_v2 (modified vorticity), d = wall distance
```

**Pressure-velocity coupling** uses SIMPLEC:

```
1. Guess pressure p*
2. Solve momentum → V*
3. Solve pressure correction: ∇·(D∇p') = ∇·(ρV*)
4. Correct: V = V* - D∇p', p = p* + αp'
5. Iterate until convergence
```

### Meanline Analysis Framework

Meanline computes 1D velocity triangles and performance using conservation laws and loss correlations.

**Velocity triangles:**

```
Blade speed: U = Ω · r_mean
Axial velocity: V_ax = φ · U (from flow coefficient)
Work: Δh_0 = U(V_θ,out - V_θ,in)  (Euler turbine equation)
```

**Loss models:**

Ainley-Mathieson (turbines):

```
Y_total = Y_profile + Y_secondary + Y_tip_clearance
η_isen = (h_0,isen - h_in) / (h_0,actual - h_in)
```

Lieblein (compressors):

```
DF = 1 - (W_2/W_1) + ΔV_θ/(2σW_1)  (diffusion factor)
ω = K_p · DF² + K_s(σ, AR, Re)
```

---

## Architecture Deep-Dives

### 1. Meanline Analysis API Handler (Rust/Axum)

```rust
// src/api/handlers/meanline.rs

use axum::{extract::{Path, State, Json}, http::StatusCode, response::IntoResponse};
use uuid::Uuid;
use crate::{
    db::models::{MeanlineInput, MeanlineResults, MeanlineRun},
    solvers::meanline::{compute_meanline, MeanlineError},
    state::AppState,
    auth::Claims,
    error::ApiError,
};

pub async fn create_meanline_run(
    State(state): State<AppState>,
    claims: Claims,
    Path(project_id): Path<Uuid>,
    Json(req): Json<MeanlineInput>,
) -> Result<impl IntoResponse, ApiError> {
    let project = sqlx::query_as!(
        crate::db::models::Project,
        "SELECT * FROM projects WHERE id = $1 AND (owner_id = $2 OR org_id IN (
            SELECT org_id FROM org_members WHERE user_id = $2
        ))",
        project_id, claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Project not found"))?;

    validate_meanline_input(&req)?;

    let run = sqlx::query_as!(
        MeanlineRun,
        r#"INSERT INTO meanline_runs (project_id, user_id, status, input_parameters)
        VALUES ($1, $2, 'pending', $3) RETURNING *"#,
        project_id, claims.user_id, serde_json::to_value(&req)?
    )
    .fetch_one(&state.db)
    .await?;

    let results = match compute_meanline(&req) {
        Ok(res) => res,
        Err(e) => {
            sqlx::query!(
                "UPDATE meanline_runs SET status = 'failed', error_message = $2 WHERE id = $1",
                run.id, e.to_string()
            )
            .execute(&state.db)
            .await?;
            return Err(ApiError::MeanlineError(e.to_string()));
        }
    };

    let completed = sqlx::query_as!(
        MeanlineRun,
        "UPDATE meanline_runs SET status = 'completed', results = $2, completed_at = NOW()
         WHERE id = $1 RETURNING *",
        run.id, serde_json::to_value(&results)?
    )
    .fetch_one(&state.db)
    .await?;

    Ok((StatusCode::CREATED, Json(completed)))
}

fn validate_meanline_input(input: &MeanlineInput) -> Result<(), ApiError> {
    if input.mass_flow_rate <= 0.0 {
        return Err(ApiError::Validation("Mass flow rate must be positive"));
    }
    if input.rotational_speed <= 0.0 {
        return Err(ApiError::Validation("Rotational speed must be positive"));
    }
    if input.tip_radius <= input.hub_radius {
        return Err(ApiError::Validation("Tip radius must exceed hub radius"));
    }
    Ok(())
}
```

### 2. Meanline Solver Core (Rust)

```rust
// solvers/meanline/src/lib.rs

use std::f64::consts::PI;

const R_AIR: f64 = 287.05;
const GAMMA: f64 = 1.4;

pub fn compute_meanline(input: &MeanlineInput) -> Result<MeanlineResults, MeanlineError> {
    let r_mean = (input.hub_radius + input.tip_radius) / 2.0;
    let blade_speed = input.rotational_speed * r_mean;
    let axial_velocity = input.flow_coefficient * blade_speed;
    let annulus_area = PI * (input.tip_radius.powi(2) - input.hub_radius.powi(2));

    let cp = GAMMA * R_AIR / (GAMMA - 1.0);
    let inlet_static_temp = input.inlet_total_temperature -
        (axial_velocity.powi(2)) / (2.0 * cp);
    let inlet_static_pressure = input.inlet_total_pressure *
        (inlet_static_temp / input.inlet_total_temperature).powf(GAMMA / (GAMMA - 1.0));
    let inlet_density = inlet_static_pressure / (R_AIR * inlet_static_temp);

    let computed_mass_flow = inlet_density * axial_velocity * annulus_area;
    if (computed_mass_flow - input.mass_flow_rate).abs() / input.mass_flow_rate > 0.05 {
        return Err(MeanlineError::PhysicallyInvalid(
            format!("Mass flow mismatch: {:.3} vs {:.3} kg/s",
                input.mass_flow_rate, computed_mass_flow)
        ));
    }

    let triangles = match input.machine_type {
        MachineType::AxialTurbine => compute_turbine_triangles(input, blade_speed, axial_velocity)?,
        MachineType::AxialCompressor => compute_compressor_triangles(input, blade_speed, axial_velocity)?,
        _ => return Err(MeanlineError::PhysicallyInvalid("Only axial machines in MVP")),
    };

    let performance = compute_stage_performance(
        input, &triangles, blade_speed, inlet_static_temp, inlet_static_pressure
    )?;

    let blade_dims = compute_blade_dimensions(input, &triangles, blade_speed)?;

    Ok(MeanlineResults { velocity_triangles: triangles, stage_performance: performance, blade_dimensions: blade_dims })
}

fn compute_turbine_triangles(
    input: &MeanlineInput, blade_speed: f64, axial_velocity: f64
) -> Result<VelocityTriangles, MeanlineError> {
    let specific_work = input.work_coefficient * blade_speed.powi(2);
    let v1_theta = 0.0;
    let v1 = axial_velocity;
    let w1 = ((axial_velocity).powi(2) + (v1_theta - blade_speed).powi(2)).sqrt();
    let alpha1 = 0.0;
    let beta1 = ((v1_theta - blade_speed) / axial_velocity).atan().to_degrees();

    let v2_theta = v1_theta + specific_work / blade_speed;
    let reaction = input.reaction_degree;
    let v2_squared = v1.powi(2) + 2.0 * specific_work * (1.0 - reaction);
    if v2_squared < 0.0 {
        return Err(MeanlineError::PhysicallyInvalid("Negative exit velocity"));
    }
    let v2 = v2_squared.sqrt();
    let alpha2 = (v2_theta / axial_velocity).atan().to_degrees();
    let w2 = ((axial_velocity).powi(2) + (v2_theta - blade_speed).powi(2)).sqrt();
    let beta2 = ((v2_theta - blade_speed) / axial_velocity).atan().to_degrees();

    Ok(VelocityTriangles {
        rotor_inlet: VelocityComponents {
            absolute_velocity: v1, relative_velocity: w1, axial_velocity,
            tangential_velocity: v1_theta, absolute_flow_angle: alpha1, relative_flow_angle: beta1
        },
        rotor_outlet: VelocityComponents {
            absolute_velocity: v2, relative_velocity: w2, axial_velocity,
            tangential_velocity: v2_theta, absolute_flow_angle: alpha2, relative_flow_angle: beta2
        },
        stator_inlet: VelocityComponents {
            absolute_velocity: v2, relative_velocity: v2, axial_velocity,
            tangential_velocity: v2_theta, absolute_flow_angle: alpha2, relative_flow_angle: alpha2
        },
        stator_outlet: VelocityComponents {
            absolute_velocity: axial_velocity, relative_velocity: axial_velocity,
            axial_velocity, tangential_velocity: 0.0, absolute_flow_angle: 0.0, relative_flow_angle: 0.0
        },
    })
}

fn compute_compressor_triangles(
    input: &MeanlineInput, blade_speed: f64, axial_velocity: f64
) -> Result<VelocityTriangles, MeanlineError> {
    let specific_work = input.work_coefficient * blade_speed.powi(2);
    let v1_theta = 0.0;
    let v1 = axial_velocity;
    let w1 = ((axial_velocity).powi(2) + (v1_theta - blade_speed).powi(2)).sqrt();
    let beta1 = ((v1_theta - blade_speed) / axial_velocity).atan().to_degrees();

    let v2_theta = v1_theta + specific_work / blade_speed;
    let v2 = (axial_velocity.powi(2) + v2_theta.powi(2)).sqrt();
    let alpha2 = (v2_theta / axial_velocity).atan().to_degrees();
    let w2 = ((axial_velocity).powi(2) + (v2_theta - blade_speed).powi(2)).sqrt();
    let beta2 = ((v2_theta - blade_speed) / axial_velocity).atan().to_degrees();

    Ok(VelocityTriangles {
        rotor_inlet: VelocityComponents {
            absolute_velocity: v1, relative_velocity: w1, axial_velocity,
            tangential_velocity: v1_theta, absolute_flow_angle: 0.0, relative_flow_angle: beta1
        },
        rotor_outlet: VelocityComponents {
            absolute_velocity: v2, relative_velocity: w2, axial_velocity,
            tangential_velocity: v2_theta, absolute_flow_angle: alpha2, relative_flow_angle: beta2
        },
        stator_inlet: VelocityComponents {
            absolute_velocity: v2, relative_velocity: v2, axial_velocity,
            tangential_velocity: v2_theta, absolute_flow_angle: alpha2, relative_flow_angle: alpha2
        },
        stator_outlet: VelocityComponents {
            absolute_velocity: axial_velocity, relative_velocity: axial_velocity,
            axial_velocity, tangential_velocity: 0.0, absolute_flow_angle: 0.0, relative_flow_angle: 0.0
        },
    })
}

fn compute_stage_performance(
    input: &MeanlineInput, triangles: &VelocityTriangles,
    blade_speed: f64, inlet_temp: f64, inlet_pressure: f64
) -> Result<StagePerformance, MeanlineError> {
    let cp = GAMMA * R_AIR / (GAMMA - 1.0);
    let specific_work = blade_speed *
        (triangles.rotor_outlet.tangential_velocity - triangles.rotor_inlet.tangential_velocity);

    let temp_ratio_isen = 1.0 + specific_work / (cp * inlet_temp);
    let pressure_ratio_isen = temp_ratio_isen.powf(GAMMA / (GAMMA - 1.0));

    let loss_coeff = match input.machine_type {
        MachineType::AxialTurbine => ainley_mathieson_loss(triangles),
        MachineType::AxialCompressor => lieblein_loss(triangles),
        _ => 0.05,
    };

    let isen_eff = 1.0 / (1.0 + loss_coeff);
    let actual_temp_ratio = 1.0 + specific_work / (cp * inlet_temp * isen_eff);
    let actual_pressure_ratio = actual_temp_ratio.powf(GAMMA / (GAMMA - 1.0));
    let power = input.mass_flow_rate * specific_work;
    let loading_coeff = specific_work / blade_speed.powi(2);

    Ok(StagePerformance {
        pressure_ratio: actual_pressure_ratio, temperature_ratio: actual_temp_ratio,
        isentropic_efficiency: isen_eff, polytropic_efficiency: isen_eff.powf(GAMMA / (GAMMA - 1.0)),
        specific_work, power, loading_coefficient: loading_coeff
    })
}

fn ainley_mathieson_loss(triangles: &VelocityTriangles) -> f64 {
    let beta1 = triangles.rotor_inlet.relative_flow_angle.to_radians();
    let beta2 = triangles.rotor_outlet.relative_flow_angle.to_radians();
    let deflection = (beta1 - beta2).abs();
    let y_profile = 0.04 + 0.06 * (deflection / (PI / 2.0)).powi(2);
    let y_secondary = 0.02;
    y_profile + y_secondary
}

fn lieblein_loss(triangles: &VelocityTriangles) -> f64 {
    let w1 = triangles.rotor_inlet.relative_velocity;
    let w2 = triangles.rotor_outlet.relative_velocity;
    let delta_v_theta = triangles.rotor_outlet.tangential_velocity -
                        triangles.rotor_inlet.tangential_velocity;
    let solidity = 1.0;
    let df = 1.0 - (w2 / w1) + delta_v_theta.abs() / (2.0 * solidity * w1);
    0.08 * df.powi(2)
}

fn compute_blade_dimensions(
    input: &MeanlineInput, triangles: &VelocityTriangles, blade_speed: f64
) -> Result<BladeDimensions, MeanlineError> {
    let r_mean = (input.hub_radius + input.tip_radius) / 2.0;
    let blade_height = input.tip_radius - input.hub_radius;
    let pitch = 2.0 * PI * r_mean / (input.number_of_blades as f64);

    let zweifel_target = 0.9;
    let beta1 = triangles.rotor_inlet.relative_flow_angle.to_radians();
    let beta2 = triangles.rotor_outlet.relative_flow_angle.to_radians();
    let solidity = (beta1.tan() + beta2.tan()) / (2.0 * zweifel_target);
    let chord = solidity * pitch;
    let aspect_ratio = blade_height / chord;

    let stator_solidity = 1.2 * solidity;
    let stator_chord = stator_solidity * pitch;

    Ok(BladeDimensions {
        rotor_chord: chord, stator_chord, rotor_pitch: pitch, stator_pitch: pitch,
        rotor_solidity: solidity, stator_solidity, rotor_aspect_ratio: aspect_ratio,
        stator_aspect_ratio: blade_height / stator_chord
    })
}
```

### 3. Blade Profile Generator (React + SVG)

```typescript
// frontend/src/components/BladeProfileEditor.tsx

import { useRef, useEffect, useState, useCallback } from 'react';

interface BladeProfile {
  profileType: 'naca65' | 'c4' | 'custom';
  chord: number;
  maxCamber: number;
  maxCamberPosition: number;
  maxThickness: number;
  stagger: number;
}

export function BladeProfileEditor() {
  const svgRef = useRef<SVGSVGElement>(null);
  const [profile, setProfile] = useState<BladeProfile>({
    profileType: 'naca65', chord: 0.05, maxCamber: 0.04,
    maxCamberPosition: 0.4, maxThickness: 0.08, stagger: 35
  });
  const [points, setPoints] = useState<{x: number, y: number}[]>([]);

  const generateProfile = useCallback((params: BladeProfile) => {
    const nPoints = 100;
    const pts: {x: number, y: number}[] = [];

    for (let i = 0; i <= nPoints; i++) {
      const x = i / nPoints;
      const yCamber = params.maxCamber * (
        x < params.maxCamberPosition
          ? (2 * params.maxCamberPosition * x - x * x) / (params.maxCamberPosition ** 2)
          : ((1 - 2 * params.maxCamberPosition) + 2 * params.maxCamberPosition * x - x * x) /
            ((1 - params.maxCamberPosition) ** 2)
      );

      const yThickness = 5 * params.maxThickness * (
        0.2969 * Math.sqrt(x) - 0.1260 * x - 0.3516 * x ** 2 +
        0.2843 * x ** 3 - 0.1015 * x ** 4
      );

      const dyCamber = params.maxCamber * (
        x < params.maxCamberPosition
          ? (2 * params.maxCamberPosition - 2 * x) / (params.maxCamberPosition ** 2)
          : (2 * params.maxCamberPosition - 2 * x) / ((1 - params.maxCamberPosition) ** 2)
      );
      const theta = Math.atan(dyCamber);

      if (i <= nPoints / 2) {
        const xU = x - yThickness * Math.sin(theta);
        const yU = yCamber + yThickness * Math.cos(theta);
        pts.push({ x: xU * params.chord, y: yU * params.chord });
      } else {
        const xL = x + yThickness * Math.sin(theta);
        const yL = yCamber - yThickness * Math.cos(theta);
        pts.push({ x: xL * params.chord, y: yL * params.chord });
      }
    }

    const staggerRad = (params.stagger * Math.PI) / 180;
    const cos = Math.cos(staggerRad);
    const sin = Math.sin(staggerRad);
    return pts.map(p => ({ x: p.x * cos - p.y * sin, y: p.x * sin + p.y * cos }));
  }, []);

  useEffect(() => {
    setPoints(generateProfile(profile));
  }, [profile, generateProfile]);

  const pathData = points.length > 0
    ? `M ${points[0].x},${points[0].y} ` +
      points.slice(1).map(p => `L ${p.x},${p.y}`).join(' ') + ' Z'
    : '';

  return (
    <div className="blade-profile-editor flex flex-col h-full">
      <div className="toolbar bg-gray-800 p-4 flex gap-4">
        <div>
          <label className="text-white text-sm">Profile Type</label>
          <select className="ml-2 bg-gray-700 text-white px-2 py-1 rounded"
            value={profile.profileType}
            onChange={(e) => setProfile({ ...profile, profileType: e.target.value as any })}>
            <option value="naca65">NACA 65</option>
            <option value="c4">NACA C4</option>
          </select>
        </div>
        <div>
          <label className="text-white text-sm">Chord (m)</label>
          <input type="number" className="ml-2 bg-gray-700 text-white px-2 py-1 rounded w-20"
            value={profile.chord} onChange={(e) => setProfile({ ...profile, chord: parseFloat(e.target.value) })}
            step={0.01} />
        </div>
        <div>
          <label className="text-white text-sm">Max Camber</label>
          <input type="number" className="ml-2 bg-gray-700 text-white px-2 py-1 rounded w-20"
            value={profile.maxCamber} onChange={(e) => setProfile({ ...profile, maxCamber: parseFloat(e.target.value) })}
            step={0.01} min={0} max={0.2} />
        </div>
        <div>
          <label className="text-white text-sm">Stagger (°)</label>
          <input type="number" className="ml-2 bg-gray-700 text-white px-2 py-1 rounded w-20"
            value={profile.stagger} onChange={(e) => setProfile({ ...profile, stagger: parseFloat(e.target.value) })}
            step={1} min={-90} max={90} />
        </div>
      </div>

      <div className="flex-1 bg-gray-900 flex items-center justify-center p-8">
        <svg ref={svgRef} className="w-full h-full" viewBox="-0.05 -0.08 0.1 0.16"
          preserveAspectRatio="xMidYMid meet">
          <defs>
            <pattern id="grid" width="0.01" height="0.01" patternUnits="userSpaceOnUse">
              <path d="M 0.01 0 L 0 0 0 0.01" fill="none" stroke="rgba(255,255,255,0.1)" strokeWidth="0.0002" />
            </pattern>
          </defs>
          <rect x="-0.05" y="-0.08" width="0.1" height="0.16" fill="url(#grid)" />
          <line x1="-0.05" y1="0" x2="0.05" y2="0" stroke="rgba(255,255,255,0.3)" strokeWidth="0.0005" />
          <path d={pathData} fill="rgba(59, 130, 246, 0.3)" stroke="rgb(59, 130, 246)" strokeWidth="0.001" />
          <line x1="0" y1="0" x2={profile.chord} y2="0" stroke="rgba(255, 255, 255, 0.5)"
            strokeWidth="0.0003" strokeDasharray="0.005 0.002" />
        </svg>
      </div>

      <div className="info-panel bg-gray-800 p-4 text-white text-sm">
        <div className="grid grid-cols-4 gap-4">
          <div><div className="text-gray-400">Chord</div><div>{(profile.chord * 1000).toFixed(1)} mm</div></div>
          <div><div className="text-gray-400">Max Camber</div>
            <div>{(profile.maxCamber * 100).toFixed(2)}%</div></div>
          <div><div className="text-gray-400">Max Thickness</div>
            <div>{(profile.maxThickness * 100).toFixed(2)}%</div></div>
          <div><div className="text-gray-400">Stagger</div><div>{profile.stagger.toFixed(1)}°</div></div>
        </div>
      </div>
    </div>
  );
}
```

### 4. CFD Mesh Generator (Rust)

```rust
// solvers/cfd/src/mesh/structured.rs

use nalgebra::Point3;

pub struct StructuredMesh {
    pub points: Vec<Point3<f64>>,
    pub ni: usize,  // Axial direction
    pub nj: usize,  // Radial direction
    pub nk: usize,  // Tangential direction
}

impl StructuredMesh {
    pub fn generate_single_passage_o_mesh(
        blade_geometry: &BladeGeometry,
        params: &MeshParameters,
    ) -> Result<Self, MeshError> {
        let ni = params.axial_cells;
        let nj = params.radial_cells;
        let nk = params.tangential_cells;

        let mut points = Vec::with_capacity(ni * nj * nk);

        // O-mesh topology: wrap around blade leading edge
        for i in 0..ni {
            let x_frac = (i as f64) / ((ni - 1) as f64);
            let x = blade_geometry.axial_chord * x_frac;

            for j in 0..nj {
                let r_frac = (j as f64) / ((nj - 1) as f64);
                let r = blade_geometry.hub_radius +
                    r_frac * (blade_geometry.tip_radius - blade_geometry.hub_radius);

                for k in 0..nk {
                    let theta_frac = (k as f64) / ((nk - 1) as f64);
                    let theta = blade_geometry.pitch * theta_frac;

                    // Blade surface distance function
                    let dist = blade_geometry.signed_distance(x, r, theta);

                    // O-mesh wrapping: near blade, use normal direction
                    let (x_mesh, theta_mesh) = if dist.abs() < params.first_layer_thickness * 10.0 {
                        let normal = blade_geometry.surface_normal(x, r, theta);
                        (x + dist * normal.x, theta + dist * normal.z / r)
                    } else {
                        (x, theta)
                    };

                    points.push(Point3::new(x_mesh, r, theta_mesh));
                }
            }
        }

        Ok(Self { points, ni, nj, nk })
    }

    pub fn cell_count(&self) -> usize {
        (self.ni - 1) * (self.nj - 1) * (self.nk - 1)
    }
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–4)

**Day 1: Project initialization**
- Cargo workspace with `turbogen-api`, `solvers/meanline`, `solvers/cfd`, `solvers/cfd-wasm`
- Axum app scaffold, config, state, error types
- Docker Compose: PostgreSQL, Redis, MinIO

**Day 2: Database schema**
- All 10 tables: users, organizations, org_members, projects, meanline_runs, blade_geometries, simulations, simulation_jobs, visualization_configs, usage_records
- SQLx models with FromRow derives
- Initial seed data

**Day 3: Authentication**
- JWT middleware, OAuth 2.0 (Google, GitHub)
- Register, login, refresh endpoints
- Auth middleware extracting Claims

**Day 4: Project and user CRUD**
- User profile endpoints
- Project CRUD with machine_type field
- Organization management
- Integration tests

### Phase 2 — Meanline Solver (Days 5–8)

**Day 5: Meanline core solver**
- `solvers/meanline/` — Rust library
- Velocity triangle calculations for axial turbines and compressors
- Euler turbine equation implementation
- Unit tests: velocity triangle validation

**Day 6: Loss correlations**
- Ainley-Mathieson turbine loss model
- Lieblein compressor loss model with diffusion factor
- Performance metrics: efficiency, pressure ratio, power
- Tests: compare to published stage data

**Day 7: Blade dimension sizing**
- Zweifel loading coefficient
- Solidity calculation from deflection angles
- Chord and pitch sizing
- Aspect ratio computation
- Tests: typical design points

**Day 8: Meanline API integration**
- `src/api/handlers/meanline.rs` endpoints
- Input validation middleware
- Run meanline analysis inline (<100ms)
- Store results in database
- Frontend hook: `useMeanlineRun`

### Phase 3 — Blade Design UI (Days 9–14)

**Day 9: Frontend scaffold**
- Vite + React + TypeScript
- Zustand stores: auth, project, blade, simulation
- Router setup, layout components
- Three.js setup with React Three Fiber

**Day 10: Blade profile editor — SVG**
- 2D profile editor component
- NACA 65 profile generation algorithm
- NACA C4 circular arc profile
- Interactive parameter controls
- Real-time SVG rendering with pan/zoom

**Day 11: Profile geometric validation**
- Camber and thickness distribution plots
- Leading/trailing edge radius validation
- Stagger angle effects
- Export profile coordinates as JSON

**Day 12: 3D blade stacking**
- Spanwise property distribution (hub to tip)
- Linear/parabolic lean and sweep
- Three.js blade surface rendering with PBR materials
- Camera controls (orbit, pan, zoom)

**Day 13: Blade geometry export**
- STEP file generation via OpenCascade bindings
- IGES export for legacy CAD compatibility
- S3 upload with presigned URLs
- Blade library browser UI

**Day 14: Meridional view**
- 2D hub/tip contours
- Stage stacking visualization
- Flow path annotation
- Interactive section selection

### Phase 4 — CFD Mesh Generation (Days 15–19)

**Day 15: Structured mesh generator — core**
- `solvers/cfd/src/mesh/structured.rs`
- O-mesh topology for single passage
- Blade surface projection
- Hub and tip endwall cells

**Day 16: Mesh quality metrics**
- Cell aspect ratio, skewness, orthogonality
- Y+ estimation from wall distance
- Quality report generation
- Python service: `mesh-service/quality.py` for advanced checks

**Day 17: Boundary layer meshing**
- First cell height from Y+ target (Y+ = 1 for SA)
- Geometric growth rate (typically 1.2)
- Normal extrusion from blade surface
- Prism layer generation

**Day 18: Periodic boundary conditions**
- Single-passage mesh with pitch angle
- Periodic face identification
- Rotational periodicity transformation
- Verification: mass flow conservation

**Day 19: Mesh export and storage**
- Binary mesh format (custom, optimized for read speed)
- S3 upload with metadata (cell count, quality metrics)
- Mesh preview image generation
- Database record linking mesh to project

### Phase 5 — CFD Solver Core (Days 20–28)

**Day 20: RANS equation discretization**
- `solvers/cfd/src/solver/rans.rs`
- Finite volume discretization
- Cell-centered variables (ρ, ρV, ρE)
- Face flux computation

**Day 21: Convective flux schemes**
- Upwind scheme (first-order)
- MUSCL scheme (second-order with limiters)
- Rhie-Chow interpolation for pressure-velocity coupling
- Flux limiters: minmod, van Leer

**Day 22: Diffusive fluxes**
- Viscous stress tensor computation
- Gradient reconstruction (least squares)
- Heat conduction flux
- Turbulent viscosity contribution

**Day 23: Rotating reference frame**
- Coriolis and centrifugal source terms
- Relative velocity formulation
- Rotating frame transformations
- Tests: solid body rotation verification

**Day 24: Spalart-Allmaras turbulence model**
- SA transport equation discretization
- Wall distance computation (BFS from wall faces)
- Production and destruction terms
- Turbulent viscosity from ν̃

**Day 25: SIMPLEC pressure-velocity coupling**
- Momentum predictor step
- Pressure correction equation assembly
- Pressure and velocity correction
- Under-relaxation factors (0.3 for pressure, 0.7 for velocity)

**Day 26: Boundary conditions**
- Inlet: total pressure, total temperature, flow direction
- Outlet: static pressure
- Wall: no-slip, adiabatic/isothermal
- Periodic: rotational symmetry
- Blade surface: wall functions or low-Re treatment

**Day 27: Convergence monitoring**
- Residual calculation (L2 norm of equation imbalances)
- Mass flow convergence
- Performance metrics convergence (pressure ratio, efficiency)
- Automatic stopping criteria

**Day 28: Performance extraction**
- Mass-averaged quantities at inlet/outlet
- Pressure ratio, temperature ratio
- Isentropic efficiency
- Torque and power
- Loading coefficient

### Phase 6 — CFD WASM Build (Days 29–32)

**Day 29: WASM compilation setup**
- `solvers/cfd-wasm/` — wasm-bindgen target
- Entry point: `run_steady_rans(mesh, bc, params)`
- Memory management: avoid allocations in hot loop
- Build pipeline: `wasm-pack build --target web --release`

**Day 30: WASM solver optimization**
- Profile with Chrome DevTools
- Optimize hot loops: flux computation, linear solver
- Use `wasm-opt -O3` for size/speed
- Target: 50K cells in <30s

**Day 31: Linear solver for WASM**
- Sparse matrix storage (CSR format)
- GMRES iterative solver (no external dependencies)
- ILU(0) preconditioner
- Convergence tolerance 1e-6

**Day 32: WASM frontend integration**
- `frontend/src/hooks/useCFDSolver.ts` — load WASM module
- Progress callback from WASM to JS
- Result data transfer (Float64Array)
- Worker thread execution to avoid UI blocking

### Phase 7 — CFD Server Execution + Results (Days 33–37)

**Day 33: Server CFD worker**
- `src/workers/cfd_worker.rs` — Redis job consumer
- Native CFD solver execution (no WASM overhead)
- Multi-threaded linear solver (Rayon)
- Progress streaming via WebSocket

**Day 34: Simulation API**
- `src/api/handlers/simulation.rs` — create, get, cancel simulation
- Cell count threshold routing (WASM vs server)
- Job priority based on plan tier
- WebSocket progress subscription

**Day 35: Results post-processing**
- Parse binary result data
- Compute derived quantities (Mach number, entropy, vorticity)
- Generate contour data for visualization
- S3 storage with presigned download URLs

**Day 36: CFD visualization UI**
- WebGL contour renderer with custom shaders
- Colormap selection (viridis, jet, coolwarm)
- Slice plane controls (axial, radial, tangential)
- Vector field rendering (velocity arrows)

**Day 37: Convergence plots**
- Real-time residual curves (WebSocket updates)
- Mass flow imbalance vs iteration
- Performance metrics history
- Auto-refresh every 10 iterations

### Phase 8 — Billing + Testing + Deployment (Days 38–42)

**Day 38: Stripe billing**
- Checkout session creation
- Plan tiers: Academic $99/month, Pro $299/month, Enterprise custom
- Webhook handlers: subscription updates
- Usage tracking: CFD hours, geometry exports

**Day 39: Plan enforcement**
- Middleware checking plan limits
- Academic: 5 seats, meanline + single-passage CFD
- Pro: full workflow, multi-stage, off-design
- Feature gating in UI with upgrade prompts

**Day 40: Solver validation**
- Benchmark 1: Known velocity triangles vs meanline output
- Benchmark 2: NASA Rotor 37 compressor stage (published data)
- Benchmark 3: VKI LS-89 turbine cascade
- Automated test suite with <5% error tolerance

**Day 41: Integration and performance testing**
- End-to-end: meanline → blade design → mesh → CFD → results
- Load testing: 20 concurrent server simulations
- WASM performance: 50K cells in <30s target
- Memory profiling: no leaks, GC behavior

**Day 42: Deployment and launch**
- Kubernetes manifests: API (3 replicas), workers (auto-scale), PostgreSQL, Redis
- CloudFront CDN for WASM and frontend assets
- Prometheus + Grafana monitoring
- Sentry error tracking
- Landing page with demo video
- Documentation: tutorials, validation cases
- Production deployment and launch announcement

---

## Critical Files

```
turbogen/
├── solvers/
│   ├── meanline/
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs                      # Velocity triangles, loss models
│   │       ├── ainley_mathieson.rs         # Turbine losses
│   │       ├── lieblein.rs                 # Compressor losses
│   │       └── blade_sizing.rs             # Zweifel loading, chord/pitch
│   ├── cfd/
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── mesh/
│   │       │   ├── structured.rs           # O-mesh generation
│   │       │   ├── boundary_layer.rs       # Prism layers, Y+ control
│   │       │   └── quality.rs              # Aspect ratio, skewness metrics
│   │       ├── solver/
│   │       │   ├── rans.rs                 # RANS equation discretization
│   │       │   ├── convective.rs           # Upwind, MUSCL, Rhie-Chow
│   │       │   ├── diffusive.rs            # Viscous and turbulent fluxes
│   │       │   ├── rotating_frame.rs       # Coriolis, centrifugal terms
│   │       │   ├── turbulence_sa.rs        # Spalart-Allmaras model
│   │       │   ├── simplec.rs              # Pressure-velocity coupling
│   │       │   └── boundary_conditions.rs  # Inlet, outlet, wall, periodic
│   │       ├── linear/
│   │       │   ├── gmres.rs                # GMRES iterative solver
│   │       │   └── ilu.rs                  # ILU(0) preconditioner
│   │       └── postprocess/
│   │           ├── performance.rs          # Efficiency, pressure ratio
│   │           └── derived.rs              # Mach, entropy, vorticity
│   └── cfd-wasm/
│       ├── Cargo.toml
│       └── src/
│           └── lib.rs                      # WASM entry points
│
├── turbogen-api/
│   ├── Cargo.toml
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
│   │   │   └── handlers/
│   │   │       ├── auth.rs
│   │   │       ├── users.rs
│   │   │       ├── projects.rs
│   │   │       ├── meanline.rs             # Meanline analysis endpoints
│   │   │       ├── blade.rs                # Blade geometry CRUD
│   │   │       ├── simulation.rs           # CFD simulation endpoints
│   │   │       ├── billing.rs              # Stripe integration
│   │   │       └── webhooks/
│   │   │           └── stripe.rs
│   │   ├── middleware/
│   │   │   └── plan_limits.rs
│   │   └── workers/
│   │       ├── mod.rs
│   │       └── cfd_worker.rs               # Server-side CFD execution
│   ├── migrations/
│   │   └── 001_initial.sql
│   └── tests/
│       ├── meanline_integration.rs
│       ├── cfd_validation.rs               # NASA Rotor 37, VKI LS-89
│       └── e2e.rs
│
├── frontend/
│   ├── package.json
│   ├── vite.config.ts
│   └── src/
│       ├── App.tsx
│       ├── main.tsx
│       ├── stores/
│       │   ├── authStore.ts
│       │   ├── projectStore.ts
│       │   ├── bladeStore.ts
│       │   └── simulationStore.ts
│       ├── hooks/
│       │   ├── useMeanlineRun.ts
│       │   ├── useCFDSolver.ts             # WASM solver hook
│       │   └── useSimulationProgress.ts    # WebSocket progress
│       ├── lib/
│       │   ├── api.ts
│       │   └── wasmLoader.ts
│       ├── pages/
│       │   ├── Dashboard.tsx
│       │   ├── ProjectEditor.tsx
│       │   ├── MeanlineDesign.tsx
│       │   ├── BladeDesign.tsx
│       │   ├── CFDSetup.tsx
│       │   ├── Results.tsx
│       │   └── Billing.tsx
│       ├── components/
│       │   ├── meanline/
│       │   │   ├── VelocityTrianglePlot.tsx
│       │   │   ├── ParameterInputs.tsx
│       │   │   └── PerformanceDisplay.tsx
│       │   ├── blade/
│       │   │   ├── BladeProfileEditor.tsx  # 2D profile with SVG
│       │   │   ├── BladeStackingEditor.tsx # 3D stacking controls
│       │   │   ├── MeridionalView.tsx      # Hub/tip contours
│       │   │   └── Blade3DViewer.tsx       # Three.js rendering
│       │   ├── cfd/
│       │   │   ├── MeshSettings.tsx
│       │   │   ├── BoundaryConditions.tsx
│       │   │   ├── SolverSettings.tsx
│       │   │   └── ConvergencePlot.tsx     # Real-time residuals
│       │   ├── results/
│       │   │   ├── ContourViewer.tsx       # WebGL contours
│       │   │   ├── VectorField.tsx         # Velocity vectors
│       │   │   ├── PerformanceMetrics.tsx
│       │   │   └── SliceControls.tsx
│       │   └── billing/
│       │       ├── PlanCard.tsx
│       │       └── UsageMeter.tsx
│       └── data/
│           └── templates/                  # Example designs (JSON)
│
├── mesh-service/                           # Python FastAPI
│   ├── requirements.txt
│   ├── main.py
│   ├── quality_check.py                    # Advanced mesh quality
│   └── optimization.py                     # Blade profile optimization
│
├── k8s/
│   ├── api-deployment.yaml
│   ├── worker-deployment.yaml
│   ├── postgres-statefulset.yaml
│   ├── redis-deployment.yaml
│   └── ingress.yaml
│
├── docker-compose.yml
├── Cargo.toml
└── .github/
    └── workflows/
        ├── ci.yml
        ├── wasm-build.yml
        └── deploy.yml
```

---

## Solver Validation Suite

### Benchmark 1: Ainley-Mathieson Turbine Stage

**Input:** Axial turbine, P_0,in = 500 kPa, T_0,in = 1200 K, ṁ = 10 kg/s, Ω = 10,000 RPM, φ = 0.65, ψ = 2.0, R = 0.5

**Expected pressure ratio:** **2.15 ± 0.05** (from Ainley-Mathieson correlation)

**Expected efficiency:** **0.88 ± 0.02** (typical for this loading)

**Expected power:** **1.42 MW ± 5%** (from Euler equation)

### Benchmark 2: Lieblein Compressor Stage

**Input:** Axial compressor, P_0,in = 101 kPa, T_0,in = 288 K, ṁ = 20 kg/s, Ω = 15,000 RPM, φ = 0.5, ψ = 0.35, DF = 0.45

**Expected pressure ratio:** **1.28 ± 0.03** (from work coefficient)

**Expected efficiency:** **0.85 ± 0.03** (from diffusion factor loss)

**Expected loading coefficient:** **0.35 ± 0.01**

### Benchmark 3: NASA Rotor 37 (CFD Validation)

**Geometry:** NASA Rotor 37 transonic compressor rotor (public geometry)

**Operating point:** 100% speed, near-peak efficiency

**Expected pressure ratio at midspan:** **2.05 ± 0.05** (NASA measured)

**Expected isentropic efficiency:** **0.877 ± 0.01** (NASA measured)

**Expected mass flow:** **20.19 kg/s ± 1%**

**Tolerance:** Pressure ratio <2%, efficiency <1%, mass flow <1%

### Benchmark 4: VKI LS-89 Turbine Cascade (CFD Validation)

**Geometry:** VKI LS-89 high-pressure turbine cascade (public)

**Operating conditions:** Inlet Mach 0.15, exit Mach 0.92, Re = 10^6

**Expected outlet flow angle:** **65.5° ± 1°** (VKI measured)

**Expected total pressure loss coefficient:** **0.036 ± 0.005** (VKI measured)

**Expected isentropic Mach distribution** matches published data within 3%

### Benchmark 5: 50K-Cell WASM Performance

**Mesh:** 100×50×10 O-mesh (axial × radial × tangential) = 50K cells

**Analysis:** Steady RANS, SA turbulence, 500 iterations

**Expected wall time (Chrome, M1 Pro):** **<30 seconds**

**Expected memory usage:** **<500 MB**

**Convergence:** Residuals drop below 1e-4

---

## Verification

### Integration Testing Checklist

1. **Auth flow** — Register → login → JWT → API calls → refresh → logout
2. **Meanline analysis** — Input parameters → compute → verify velocity triangles match analytical
3. **Blade profile generation** — NACA 65 parameters → SVG rendering → verify coordinates
4. **3D blade stacking** — Hub/tip profiles → lean/sweep → Three.js display
5. **Mesh generation** — Blade geometry → O-mesh → cell count, quality metrics
6. **WASM CFD** — 30K cell mesh → WASM solver → results in <20s
7. **Server CFD** — 100K cell mesh → job queued → worker executes → WebSocket progress → results in S3
8. **Results visualization** — Load CFD results → contours display → slice planes work
9. **Convergence monitoring** — Residual curves update in real-time via WebSocket
10. **Blade library** — Upload blade → STEP export → S3 storage → search/download
11. **Performance extraction** — CFD results → pressure ratio, efficiency, mass flow
12. **Plan limits** — Academic user → attempt multi-stage → blocked with upgrade prompt
13. **Billing** — Subscribe to Pro → Stripe checkout → webhook → plan updated
14. **Concurrent simulations** — 10 users running CFD simultaneously → all complete
15. **NASA Rotor 37 validation** — Run benchmark → results within 2% of published data

### SQL Verification Queries

```sql
-- Simulation throughput by plan tier
SELECT u.plan, COUNT(*) as sim_count, AVG(s.wall_time_ms) as avg_time_ms
FROM simulations s JOIN users u ON s.user_id = u.id
WHERE s.status = 'completed' AND s.created_at > NOW() - INTERVAL '7 days'
GROUP BY u.plan;

-- Meanline analysis success rate
SELECT status, COUNT(*) as count, COUNT(*) * 100.0 / SUM(COUNT(*)) OVER() as pct
FROM meanline_runs
WHERE created_at > NOW() - INTERVAL '30 days'
GROUP BY status;

-- Blade library downloads
SELECT name, machine_type, download_count
FROM blade_geometries
WHERE is_public = true
ORDER BY download_count DESC
LIMIT 20;

-- CFD execution mode split
SELECT execution_mode, COUNT(*) as count
FROM simulations
WHERE created_at > NOW() - INTERVAL '7 days'
GROUP BY execution_mode;

-- Average convergence iterations
SELECT simulation_type,
       AVG((convergence_history->-1->>'iteration')::int) as avg_iterations
FROM simulations
WHERE status = 'completed' AND convergence_history IS NOT NULL
GROUP BY simulation_type;
```

---

## Deployment Architecture

### Infrastructure Stack

```
┌─────────────────────────────────────────────────────────────┐
│                      CloudFront CDN                          │
│  - Frontend static assets (HTML, JS, CSS)                   │
│  - WASM solver bundle (~3MB, cached at edge)                │
│  - Blade geometry previews                                  │
└────────────────┬────────────────────────────────────────────┘
                 │
┌────────────────▼───────────────────────────────────────────┐
│              Application Load Balancer                      │
└────────────────┬───────────────────────────────────────────┘
                 │
       ┌─────────┴──────────┐
       │                    │
┌──────▼──────┐      ┌──────▼──────────┐
│ API Servers │      │  CFD Workers    │
│ (3 replicas)│      │ (auto-scaling)  │
│ Axum/Rust   │      │ Native solver   │
└──────┬──────┘      └─────────┬───────┘
       │                       │
       └───────┬───────────────┘
               │
     ┌─────────┴──────────┐
     │                    │
┌────▼─────┐      ┌───────▼────┐       ┌──────────┐
│PostgreSQL│      │   Redis    │       │    S3    │
│(Primary +│      │ (job queue,│       │(Geometry,│
│ Replica) │      │  pub/sub)  │       │  meshes, │
└──────────┘      └────────────┘       │ results) │
                                        └──────────┘
```

### Kubernetes Resources

- **API Deployment:** 3 replicas, 2 CPU / 4GB RAM each, HPA (2-10 replicas based on request rate)
- **Worker Deployment:** 2-8 replicas, 4 CPU / 16GB RAM each, HPA (based on Redis queue depth)
- **PostgreSQL StatefulSet:** 3 replicas (1 primary + 2 read replicas), 8 CPU / 32GB RAM, 500GB SSD
- **Redis Deployment:** 1 replica, 2 CPU / 8GB RAM, persistent volume
- **Ingress:** NGINX with TLS (Let's Encrypt)

### Monitoring and Observability

- **Prometheus:** Metrics collection (API latency, solver time, queue depth, error rate)
- **Grafana:** Dashboards (system health, solver performance, user activity, billing metrics)
- **Sentry:** Error tracking with source maps (frontend + backend)
- **Structured logging:** JSON logs with trace IDs, indexed in Elasticsearch

---

## Post-MVP Roadmap

### v1.1 — Throughflow Analysis (2 months)

- 2D streamline curvature method (SLC)
- Spanwise property distribution
- Radial equilibrium equation solver
- Off-design performance prediction
- Multi-stage stacking

**Value:** Enables multi-stage axial turbines and compressors with accurate spanwise flow variation

### v1.2 — Centrifugal Machines (2 months)

- Centrifugal compressor impeller design (meridional contours, blade angles)
- Diffuser design (vaned, vaneless)
- Pump hydraulic design (specific speed, impeller, volute)
- Slip factor correlations
- NPSH prediction

**Value:** Expands addressable market to turbochargers, pumps, HVAC centrifugal compressors

### v1.3 — Unsteady CFD and Rotor-Stator Interaction (3 months)

- Unsteady RANS with sliding mesh interface
- Time-accurate simulation of rotor passing
- FFT of unsteady pressure for acoustic forcing
- Forced response analysis (blade vibration from rotor-stator interaction)

**Value:** Required for high-fidelity design, predicting vibration and noise

### v1.4 — Aeroelastics and Aeroacoustics (3 months)

- Campbell diagram: natural frequencies vs rotational speed
- Flutter prediction: aerodynamic damping via energy method
- Forced response from unsteady CFD
- Tyler-Sofrin acoustic modes for fan noise
- Duct propagation and attenuation

**Value:** Critical for aerospace and power generation applications to avoid resonance and meet noise regulations

### v1.5 — Off-Design Performance Maps (2 months)

- Operating line and surge margin prediction
- Compressor/turbine maps (pressure ratio vs. mass flow, efficiency islands)
- Choke and surge boundary detection
- Multi-point optimization for wide operating range

**Value:** Design for system integration, part-load efficiency

### v1.6 — Collaboration and Parametric Studies (1 month)

- Real-time collaboration (multiplayer editing)
- Parametric design sweeps (automate meanline + CFD over parameter ranges)
- Design of experiments (DOE) with response surfaces
- Git-style version control for designs

**Value:** Team workflows, automated optimization

### v1.7 — Advanced Optimization (3 months)

- Gradient-free optimization (genetic algorithms, particle swarm)
- Adjoint-based gradient computation for CFD (shape optimization)
- Multi-objective optimization (efficiency vs. weight vs. cost)
- Integration with external tools (MATLAB, Python scripts)

**Value:** Automated design space exploration, optimal blade shapes

### v1.8 — On-Premise Deployment (1 month)

- Docker Compose for single-server deployment
- Air-gapped installation support
- LDAP/SAML authentication integration
- License key management

**Value:** Enterprise customers with security/IP requirements (aerospace, defense)
