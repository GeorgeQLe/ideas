# 89. FoamSim — Polymer and Foam Process Simulation Platform

## Implementation Plan

**MVP Scope:** Browser-based CAD import with STL/STEP upload and automatic midplane extraction, Hele-Shaw (2.5D) mold filling solver with Cross-WLF non-Newtonian viscosity and PETSc sparse solver running server-side for meshes ≤50K elements, 50+ thermoplastic materials (PP, PE, ABS, PA6, PA66, PC, POM, PBT, PET) with Cross-WLF viscosity and Tait pvT parameters from supplier datasheets, fill time visualization via Three.js WebGL with animated filling pattern showing pressure/temperature/shear rate contours on 3D geometry, weld line and air trap detection with automatic defect flagging, basic packing simulation with Tait equation of state for volumetric shrinkage, isotropic shrinkage prediction, single-cavity single-gate analysis with gate location comparison (2-4 positions), Stripe billing with three tiers (Free / Pro $149/mo / Advanced $349/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async, Tower middleware, JWT auth |
| Hele-Shaw Solver | Rust (native) | 2.5D FEM with Cross-WLF viscosity, PETSc GMRES+AMG |
| Foam Solver | Rust | Classical nucleation theory + cell growth ODEs |
| Mesh Generation | Python 3.12 (FastAPI) | Gmsh for mesh, midplane extraction, quality checks |
| Material Fitting | Python 3.12 (FastAPI) | Scipy curve_fit for viscosity calibration |
| Database | PostgreSQL 16 | Projects, users, simulations, materials, recipes |
| ORM / Query | SQLx (Rust) | Compile-time checked queries |
| Auth | JWT + OAuth 2.0 | Google/GitHub OAuth, bcrypt passwords |
| Storage | AWS S3 | CAD files, meshes, results, animations |
| Frontend | React 19 + TypeScript | Vite, Zustand state |
| 3D Viz | Three.js + React Three Fiber | WebGL filling animation, contour plots |
| Real-time | WebSocket (Axum) | Live progress, convergence monitoring |
| Job Queue | Redis 7 + Tokio tasks | Simulation job distribution |
| Billing | Stripe | Checkout, Customer Portal, Webhooks |
| Monitoring | Prometheus + Grafana, Sentry | Metrics, error tracking |

### Key Architecture Decisions

1. **Hele-Shaw (2.5D) solver for MVP**: Assumes thin-wall flow (thickness << lateral dimensions), reducing 3D Navier-Stokes to 2.5D pressure-diffusion on midplane with gap-averaged viscosity. Covers 85%+ of injection-molded parts with 50-100x faster solve times than full 3D. Architecture supports 3D upgrade post-MVP.

2. **Cross-WLF viscosity model**: Industry standard combining Cross shear-thinning with WLF temperature dependence. Used by Moldflow, Moldex3D, Sigmasoft. All major resin suppliers provide Cross-WLF parameters in datasheets.

3. **Automatic midplane extraction**: Medial axis transform via Gmsh Python API extracts 2D midplane from 3D CAD solid, eliminating manual midplane creation (Moldflow requires this). Reduces setup from hours to minutes.

4. **PETSc sparse solver**: GMRES/BiCGStab iterative solvers with AMG preconditioning for 50K-500K unknown pressure systems. 10-50x faster than direct methods for large sparse systems.

5. **Material database from supplier datasheets**: 50+ thermoplastics with Cross-WLF viscosity, Tait pvT, thermal, crystallization parameters from BASF Ultrasim, DuPont, SABIC CAMPUS databases.

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";

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

CREATE TABLE org_members (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    org_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role TEXT NOT NULL DEFAULT 'member',
    joined_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(org_id, user_id)
);

CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    cad_file_url TEXT,
    mesh_data_url TEXT,
    part_metadata JSONB DEFAULT '{}',
    gate_locations JSONB DEFAULT '[]',
    settings JSONB DEFAULT '{}',
    is_public BOOLEAN NOT NULL DEFAULT false,
    thumbnail_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE simulations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    simulation_type TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending',
    material_id UUID REFERENCES materials(id),
    process_params JSONB NOT NULL DEFAULT '{}',
    mesh_element_count INTEGER DEFAULT 0,
    results_url TEXT,
    results_summary JSONB,
    defects JSONB DEFAULT '[]',
    error_message TEXT,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    wall_time_ms INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE simulation_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    simulation_id UUID NOT NULL REFERENCES simulations(id) ON DELETE CASCADE,
    worker_id TEXT,
    cores_allocated INTEGER DEFAULT 4,
    memory_mb INTEGER DEFAULT 8192,
    priority INTEGER DEFAULT 0,
    progress_pct REAL DEFAULT 0.0,
    convergence_data JSONB DEFAULT '[]',
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE materials (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    trade_name TEXT,
    manufacturer TEXT,
    grade TEXT,
    material_class TEXT NOT NULL,
    filler_type TEXT,
    filler_fraction REAL DEFAULT 0.0,
    viscosity_d1 REAL NOT NULL,
    viscosity_d2 REAL NOT NULL,
    viscosity_d3 REAL NOT NULL,
    viscosity_a1 REAL NOT NULL,
    viscosity_a2 REAL NOT NULL,
    viscosity_tau_star REAL NOT NULL,
    viscosity_n REAL NOT NULL,
    viscosity_t_star REAL NOT NULL,
    pvt_b1m REAL NOT NULL,
    pvt_b2m REAL NOT NULL,
    pvt_b3m REAL NOT NULL,
    pvt_b4m REAL NOT NULL,
    pvt_b1s REAL NOT NULL,
    pvt_b2s REAL NOT NULL,
    pvt_b3s REAL NOT NULL,
    pvt_b4s REAL NOT NULL,
    pvt_b5 REAL NOT NULL,
    pvt_b6 REAL NOT NULL,
    pvt_b7 REAL NOT NULL,
    pvt_b8 REAL NOT NULL,
    pvt_b9 REAL NOT NULL,
    thermal_conductivity_melt REAL NOT NULL,
    thermal_conductivity_solid REAL NOT NULL,
    specific_heat_melt REAL NOT NULL,
    specific_heat_solid REAL NOT NULL,
    glass_transition_temp REAL,
    melt_temp REAL,
    no_flow_temp REAL NOT NULL,
    is_crystalline BOOLEAN DEFAULT false,
    recommended_melt_temp_min REAL,
    recommended_melt_temp_max REAL,
    recommended_mold_temp_min REAL,
    recommended_mold_temp_max REAL,
    datasheet_url TEXT,
    tags TEXT[] DEFAULT '{}',
    is_verified BOOLEAN DEFAULT false,
    is_builtin BOOLEAN DEFAULT true,
    usage_count INTEGER DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX materials_class_idx ON materials(material_class);
CREATE INDEX materials_name_trgm_idx ON materials USING gin(name gin_trgm_ops);

CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    record_type TEXT NOT NULL,
    quantity REAL NOT NULL,
    period_start DATE NOT NULL,
    period_end DATE NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
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
    #[serde(skip)]
    pub password_hash: Option<String>,
    pub name: String,
    pub plan: String,
    pub stripe_customer_id: Option<String>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Project {
    pub id: Uuid,
    pub owner_id: Uuid,
    pub name: String,
    pub cad_file_url: Option<String>,
    pub mesh_data_url: Option<String>,
    pub part_metadata: serde_json::Value,
    pub gate_locations: serde_json::Value,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Simulation {
    pub id: Uuid,
    pub project_id: Uuid,
    pub user_id: Uuid,
    pub simulation_type: String,
    pub status: String,
    pub material_id: Option<Uuid>,
    pub process_params: serde_json::Value,
    pub mesh_element_count: i32,
    pub results_url: Option<String>,
    pub results_summary: Option<serde_json::Value>,
    pub defects: serde_json::Value,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Material {
    pub id: Uuid,
    pub name: String,
    pub material_class: String,
    pub viscosity_d1: f64,
    pub viscosity_a1: f64,
    pub viscosity_a2: f64,
    pub viscosity_tau_star: f64,
    pub viscosity_n: f64,
    pub viscosity_t_star: f64,
    pub pvt_b1m: f64,
    pub pvt_b3m: f64,
    pub thermal_conductivity_melt: f64,
    pub no_flow_temp: f64,
    pub recommended_melt_temp_min: Option<f64>,
    pub recommended_melt_temp_max: Option<f64>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct ProcessParams {
    pub injection_time: f64,
    pub melt_temperature: f64,
    pub mold_temperature: f64,
    pub fill_control: FillControl,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum FillControl {
    ConstantFlowRate { flow_rate: f64 },
    ConstantInjectionTime { time: f64 },
}
```

---

## Solver Architecture Deep-Dive

### Governing Equations and Discretization

Hele-Shaw gap-averaged momentum and continuity for thin-wall flow (h << L):

```
Pressure equation:
  ∇·(S(x,y,t)∇P) = ∂h_eff/∂t

  S = h³/(12η_eff)  (conductance)
  h_eff = h - 2δ    (effective thickness after frozen layer)

Cross-WLF viscosity:
  η(γ̇,T,P) = η₀(T,P) / [1 + (η₀γ̇/τ*)^(1-n)]
  η₀(T,P) = D₁·exp[-A₁(T-T*)/(A₂+T-T*)]
  T* = D₂ + D₃P

Energy equation:
  ρCp(∂T/∂t + u·∇T) = k∇²T + ηγ̇² + (h_mold/h)(T_mold-T)

Frozen layer:
  δ(t) = (h/2)·erf(√(αt/h²))
```

**FEM discretization:** P1-P1 stabilized triangular elements with SUPG for advection-dominated energy equation. Weak form:

```
∫_Ω (S∇P)·∇φ dΩ = ∫_Ω (∂h_eff/∂t)φ dΩ
```

**Time marching:** Adaptive timestep based on CFL condition: Δt < h²/(α + u_max·h). Picard iteration for viscosity: recompute η(γ̇,T) → rebuild S matrix → resolve until convergence.

### PETSc Solver Integration

The pressure system is large (50K-500K unknowns) and sparse. We use PETSc via Rust FFI for industrial-strength iterative solvers.

```rust
// solver-core/src/petsc.rs

use petsc_sys::*;
use std::ffi::CString;

pub struct PetscSolver {
    ksp: KSP,
    mat: Mat,
    x: Vec_,
    b: Vec_,
    n_nodes: usize,
}

impl PetscSolver {
    pub fn new(n: usize, nnz_per_row: usize) -> Self {
        unsafe {
            // Initialize PETSc if not already done
            let argc = 0i32;
            let argv: *mut *mut i8 = std::ptr::null_mut();
            PetscInitialize(&argc as *const i32, &argv as *const *mut *mut i8,
                           std::ptr::null(), std::ptr::null());

            // Create sparse matrix
            let mut mat = std::ptr::null_mut();
            MatCreate(PETSC_COMM_SELF, &mut mat);
            MatSetSizes(mat, n as i32, n as i32, n as i32, n as i32);
            MatSetType(mat, MATSEQAIJ);
            MatSeqAIJSetPreallocation(mat, nnz_per_row as i32, std::ptr::null());

            // Create vectors
            let mut x = std::ptr::null_mut();
            let mut b = std::ptr::null_mut();
            VecCreate(PETSC_COMM_SELF, &mut x);
            VecSetSizes(x, n as i32, n as i32);
            VecSetFromOptions(x);
            VecDuplicate(x, &mut b);

            // Create Krylov solver
            let mut ksp = std::ptr::null_mut();
            KSPCreate(PETSC_COMM_SELF, &mut ksp);
            KSPSetOperators(ksp, mat, mat);
            KSPSetType(ksp, KSPGMRES);
            KSPGMRESSetRestart(ksp, 30);  // GMRES(30)

            // Configure algebraic multigrid preconditioner
            let mut pc = std::ptr::null_mut();
            KSPGetPC(ksp, &mut pc);
            PCSetType(pc, PCHYPRE);
            let hypre_type = CString::new("boomeramg").unwrap();
            PCHYPRESetType(pc, hypre_type.as_ptr());

            // Set tolerances: rtol=1e-8, atol=1e-30, dtol=1e4, maxits=500
            KSPSetTolerances(ksp, 1e-8, 1e-30, 1e4, 500);

            Self { ksp, mat, x, b, n_nodes: n }
        }
    }

    pub fn set_matrix(&mut self, row_ptr: &[usize], col_idx: &[usize], values: &[f64]) {
        unsafe {
            MatZeroEntries(self.mat);

            // CSR format: row_ptr[i]..row_ptr[i+1] gives column indices for row i
            for i in 0..self.n_nodes {
                let start = row_ptr[i];
                let end = row_ptr[i + 1];
                let ncols = (end - start) as i32;

                let cols: Vec<i32> = col_idx[start..end].iter().map(|&c| c as i32).collect();
                let vals = &values[start..end];

                MatSetValues(
                    self.mat,
                    1,
                    &(i as i32),
                    ncols,
                    cols.as_ptr(),
                    vals.as_ptr(),
                    INSERT_VALUES as u32,
                );
            }

            MatAssemblyBegin(self.mat, MAT_FINAL_ASSEMBLY);
            MatAssemblyEnd(self.mat, MAT_FINAL_ASSEMBLY);
        }
    }

    pub fn set_rhs(&mut self, rhs: &[f64]) {
        unsafe {
            let mut ptr: *mut f64 = std::ptr::null_mut();
            VecGetArray(self.b, &mut ptr);
            std::ptr::copy_nonoverlapping(rhs.as_ptr(), ptr, self.n_nodes);
            VecRestoreArray(self.b, &mut ptr);
        }
    }

    pub fn solve(&mut self, pressure: &mut [f64]) -> Result<usize, SolverError> {
        unsafe {
            KSPSolve(self.ksp, self.b, self.x);

            // Check convergence
            let mut reason: KSPConvergedReason = 0;
            KSPGetConvergedReason(self.ksp, &mut reason);

            if reason < 0 {
                return Err(SolverError::Diverged(format!(
                    "PETSc solver diverged with reason {}", reason
                )));
            }

            // Get iteration count
            let mut iters = 0i32;
            KSPGetIterationNumber(self.ksp, &mut iters);

            // Copy solution back
            let mut ptr: *const f64 = std::ptr::null();
            VecGetArrayRead(self.x, &mut ptr);
            std::ptr::copy_nonoverlapping(ptr, pressure.as_mut_ptr(), self.n_nodes);
            VecRestoreArrayRead(self.x, &mut ptr);

            Ok(iters as usize)
        }
    }
}

impl Drop for PetscSolver {
    fn drop(&mut self) {
        unsafe {
            MatDestroy(&mut self.mat);
            VecDestroy(&mut self.x);
            VecDestroy(&mut self.b);
            KSPDestroy(&mut self.ksp);
        }
    }
}

#[derive(Debug)]
pub enum SolverError {
    Convergence(String),
    Diverged(String),
    InvalidMatrix(String),
}
```

**Typical performance:** GMRES(30) with BoomerAMG requires 10-50 iterations for 50K unknowns, solving in 0.5-2 seconds per timestep on 4-core CPU.

---

## Architecture Deep-Dives

### 1. Simulation API Handler (Rust)

```rust
// src/api/handlers/simulation.rs

use axum::{extract::{Path, State, Json}, http::StatusCode};
use uuid::Uuid;

pub async fn create_simulation(
    State(state): State<AppState>,
    claims: Claims,
    Path(project_id): Path<Uuid>,
    Json(req): Json<CreateSimulationRequest>,
) -> Result<impl IntoResponse, ApiError> {
    let project = sqlx::query_as!(Project,
        "SELECT * FROM projects WHERE id = $1 AND owner_id = $2",
        project_id, claims.user_id
    ).fetch_optional(&state.db).await?
    .ok_or(ApiError::NotFound("Project not found"))?;

    // Trigger mesh generation if needed
    let mesh_url = if project.mesh_data_url.is_none() {
        state.mesh_service.generate_mesh(&project.cad_file_url.unwrap(), project_id).await?
    } else {
        project.mesh_data_url.unwrap()
    };

    // Fetch material
    let material = sqlx::query_as!(Material,
        "SELECT * FROM materials WHERE id = $1", req.material_id
    ).fetch_one(&state.db).await?;

    // Check plan limits
    let user = sqlx::query_as!(User, "SELECT * FROM users WHERE id = $1", claims.user_id)
        .fetch_one(&state.db).await?;

    let mesh: Mesh = serde_json::from_slice(&state.s3_client.get_object()
        .bucket(&state.config.s3_bucket).key(&mesh_url).send().await?.body.collect().await?)?;

    if user.plan == "free" && mesh.elements.len() > 10_000 {
        return Err(ApiError::PlanLimit("Free plan: max 10K elements"));
    }

    // Create simulation
    let sim = sqlx::query_as!(Simulation,
        r#"INSERT INTO simulations (project_id, user_id, simulation_type, material_id,
           process_params, mesh_element_count) VALUES ($1, $2, $3, $4, $5, $6) RETURNING *"#,
        project_id, claims.user_id, req.simulation_type, req.material_id,
        serde_json::to_value(&req.process_params)?, mesh.elements.len() as i32
    ).fetch_one(&state.db).await?;

    // Enqueue job
    state.redis.lpush::<_, _, ()>("simulation:jobs", sim.id.to_string()).await?;

    Ok((StatusCode::CREATED, Json(sim)))
}
```

### 2. Hele-Shaw Filling Solver (Rust)

The core solver that implements the Hele-Shaw filling algorithm with adaptive timestepping and Picard iteration for nonlinear viscosity.

```rust
// solver-core/src/filling.rs

use crate::{mesh::Mesh, material::Material, petsc::PetscSolver};

pub struct HeleShawSolver {
    pub mesh: Mesh,
    pub material: Material,
    pub melt_temp: f64,
    pub mold_temp: f64,
    pub injection_time: f64,

    // State vectors (per node or per element)
    pub pressure: Vec<f64>,           // Pa, per node
    pub temperature: Vec<f64>,        // K, per element
    pub fill_fraction: Vec<f64>,      // 0-1, per element
    pub velocity: Vec<(f64, f64)>,    // m/s, per element
    pub shear_rate: Vec<f64>,         // 1/s, per element
    pub viscosity: Vec<f64>,          // Pa·s, per element
    pub frozen_layer_thickness: Vec<f64>,  // m, per element

    pub time: f64,
    pub timestep: f64,
    pub iteration: usize,
    pub fill_volume: f64,
    pub total_volume: f64,
}

impl HeleShawSolver {
    pub fn new(
        mesh: Mesh,
        material: Material,
        melt_temp: f64,
        mold_temp: f64,
        injection_time: f64,
    ) -> Self {
        let n_nodes = mesh.nodes.len();
        let n_elems = mesh.elements.len();

        let total_volume: f64 = mesh.elements.iter()
            .map(|e| e.area * e.thickness)
            .sum();

        Self {
            mesh,
            material,
            melt_temp,
            mold_temp,
            injection_time,
            pressure: vec![0.0; n_nodes],
            temperature: vec![melt_temp + 273.15; n_elems],
            fill_fraction: vec![0.0; n_elems],
            velocity: vec![(0.0, 0.0); n_elems],
            shear_rate: vec![0.0; n_elems],
            viscosity: vec![0.0; n_elems],
            frozen_layer_thickness: vec![0.0; n_elems],
            time: 0.0,
            timestep: 0.001,
            iteration: 0,
            fill_volume: 0.0,
            total_volume,
        }
    }

    pub fn run(&mut self) -> Result<FillingResult, SolverError> {
        let mut result = FillingResult::new();
        let target_flow_rate = self.total_volume / self.injection_time;

        self.initialize_gates();
        let mut petsc = PetscSolver::new(self.mesh.nodes.len(), 20);

        while self.fill_volume < self.total_volume && self.time < self.injection_time * 2.0 {
            // Picard iteration for nonlinear viscosity
            for picard_iter in 0..10 {
                let viscosity_old = self.viscosity.clone();

                self.update_viscosity();

                let (row_ptr, col_idx, values) = self.assemble_pressure_system();
                petsc.set_matrix(&row_ptr, &col_idx, &values);
                petsc.set_rhs(&self.assemble_rhs(target_flow_rate));

                let iters = petsc.solve(&mut self.pressure)?;

                // Check Picard convergence
                let visc_change: f64 = self.viscosity.iter().zip(&viscosity_old)
                    .map(|(new, old)| ((new - old) / old).abs())
                    .sum::<f64>() / self.viscosity.len() as f64;

                if visc_change < 0.01 || picard_iter == 9 {
                    break;
                }
            }

            self.compute_velocity_from_pressure();
            self.advance_fill_front(self.timestep);
            self.solve_temperature(self.timestep);
            self.update_frozen_layer();

            if self.iteration % 10 == 0 {
                self.detect_defects();
                result.add_snapshot(self.time, &self.fill_fraction, &self.pressure, &self.temperature);
            }

            // Adaptive timestep
            let max_vel = self.velocity.iter()
                .map(|(vx, vy)| (vx*vx + vy*vy).sqrt())
                .fold(0.0, f64::max);

            let cfl_dt = 0.5 * self.mesh.min_element_size() / (max_vel + 1e-10);
            self.timestep = cfl_dt.min(0.01).max(1e-5);

            self.time += self.timestep;
            self.iteration += 1;

            if self.iteration % 50 == 0 {
                tracing::info!(
                    "t={:.4}s, fill={:.1}%, max_P={:.1}MPa, dt={:.2}ms",
                    self.time,
                    self.fill_volume / self.total_volume * 100.0,
                    self.pressure.iter().fold(0.0, |a, &b| a.max(b)) / 1e6,
                    self.timestep * 1000.0
                );
            }
        }

        result.finalize(self.time, self.iteration);
        Ok(result)
    }

    fn initialize_gates(&mut self) {
        // Mark gate elements as initially filled
        for gate in &self.mesh.gate_nodes {
            let elem_id = self.mesh.find_element_containing_node(*gate);
            self.fill_fraction[elem_id] = 1.0;
            self.fill_volume += self.mesh.elements[elem_id].area * self.mesh.elements[elem_id].thickness;
        }
    }

    fn update_viscosity(&mut self) {
        for (i, elem) in self.mesh.elements.iter().enumerate() {
            if self.fill_fraction[i] < 0.01 { continue; }

            let T = self.temperature[i];
            let gamma_dot = self.shear_rate[i];
            let P = self.element_avg_pressure(elem);

            // Cross-WLF model
            let T_star = self.material.viscosity_t_star + self.material.viscosity_d3 * P;
            let denom = self.material.viscosity_a2 + T - T_star;

            if denom.abs() < 1e-6 {
                self.viscosity[i] = 1e8;  // Near glass transition, very high viscosity
                continue;
            }

            let eta_0 = self.material.viscosity_d1 * f64::exp(
                -self.material.viscosity_a1 * (T - T_star) / denom
            );

            let tau_star = self.material.viscosity_tau_star;
            let n = self.material.viscosity_n;

            let shear_term = eta_0 * gamma_dot / tau_star;
            let eta = eta_0 / (1.0 + shear_term.powf(1.0 - n));

            self.viscosity[i] = eta.clamp(1.0, 1e8);
        }
    }

    fn compute_velocity_from_pressure(&mut self) {
        for (i, elem) in self.mesh.elements.iter().enumerate() {
            if self.fill_fraction[i] < 0.01 { continue; }

            let grad_p = self.compute_pressure_gradient(elem);
            let h_eff = elem.thickness - 2.0 * self.frozen_layer_thickness[i];

            if h_eff < 1e-6 {
                self.velocity[i] = (0.0, 0.0);
                self.shear_rate[i] = 0.0;
                continue;
            }

            let eta = self.viscosity[i];
            let coeff = -h_eff.powi(2) / (12.0 * eta);

            self.velocity[i] = (coeff * grad_p.0, coeff * grad_p.1);

            let u_mag = (self.velocity[i].0.powi(2) + self.velocity[i].1.powi(2)).sqrt();
            self.shear_rate[i] = 2.0 * u_mag / h_eff;
        }
    }

    fn compute_pressure_gradient(&self, elem: &Element) -> (f64, f64) {
        // P1 element: ∇P = ∑ P_i ∇N_i
        let (grad_n0, grad_n1, grad_n2) = elem.shape_gradients();
        let p0 = self.pressure[elem.nodes[0]];
        let p1 = self.pressure[elem.nodes[1]];
        let p2 = self.pressure[elem.nodes[2]];

        let grad_px = p0 * grad_n0.0 + p1 * grad_n1.0 + p2 * grad_n2.0;
        let grad_py = p0 * grad_n0.1 + p1 * grad_n1.1 + p2 * grad_n2.1;

        (grad_px, grad_py)
    }

    fn advance_fill_front(&mut self, dt: f64) {
        for (i, elem) in self.mesh.elements.iter().enumerate() {
            if self.fill_fraction[i] >= 0.99 { continue; }

            let flux_in = self.compute_volume_flux_into_element(elem);
            let volume_change = flux_in * dt;
            let elem_volume = elem.area * elem.thickness;

            let df = volume_change / elem_volume;
            self.fill_fraction[i] = (self.fill_fraction[i] + df).min(1.0).max(0.0);

            // Enforce atmospheric pressure at melt front
            if self.fill_fraction[i] > 0.0 && self.fill_fraction[i] < 1.0 {
                for &node_id in &elem.nodes {
                    self.pressure[node_id] = 101325.0;  // 1 atm
                }
            }
        }

        self.fill_volume = self.mesh.elements.iter().enumerate()
            .map(|(i, e)| e.area * e.thickness * self.fill_fraction[i])
            .sum();
    }

    fn solve_temperature(&mut self, dt: f64) {
        // Simple explicit energy equation: ∂T/∂t = -u·∇T + α∇²T + Q_visc + Q_mold
        for (i, elem) in self.mesh.elements.iter().enumerate() {
            if self.fill_fraction[i] < 0.01 { continue; }

            let alpha = self.material.thermal_conductivity_melt /
                       (self.material.density * self.material.specific_heat_melt);

            let Q_visc = self.viscosity[i] * self.shear_rate[i].powi(2) /
                        (self.material.density * self.material.specific_heat_melt);

            let h_mold = 2.0 * self.material.thermal_conductivity_melt / elem.thickness;
            let Q_mold = h_mold * (self.mold_temp + 273.15 - self.temperature[i]) /
                        (self.material.density * self.material.specific_heat_melt * elem.thickness);

            let dT_dt = Q_visc + Q_mold;  // Simplified, ignoring advection and conduction
            self.temperature[i] += dT_dt * dt;

            self.temperature[i] = self.temperature[i].max(self.mold_temp + 273.15)
                                                     .min(self.melt_temp + 373.15);
        }
    }

    fn update_frozen_layer(&mut self) {
        for (i, elem) in self.mesh.elements.iter().enumerate() {
            if self.fill_fraction[i] < 0.01 { continue; }

            let contact_time = self.time;  // Simplified: assume element contacted mold at t=0
            let alpha = self.material.thermal_conductivity_melt /
                       (self.material.density * self.material.specific_heat_melt);

            let delta = (elem.thickness / 2.0) * f64::erf(f64::sqrt(alpha * contact_time / elem.thickness.powi(2)));

            self.frozen_layer_thickness[i] = delta.min(elem.thickness / 2.0);
        }
    }

    fn detect_defects(&mut self) {
        // Weld line detection: elements with conflicting flow directions
        // Air trap detection: unfilled pockets surrounded by filled regions
        // Implementation omitted for brevity
    }
}

pub struct FillingResult {
    pub snapshots: Vec<FillingSnapshot>,
    pub final_time: f64,
    pub iterations: usize,
}

pub struct FillingSnapshot {
    pub time: f64,
    pub fill_fraction: Vec<f64>,
    pub pressure: Vec<f64>,
    pub temperature: Vec<f64>,
}
```

### 3. Three.js Filling Animation (React)

The frontend 3D viewer renders filling animation with real-time contour plots and interactive controls.

```typescript
// frontend/src/components/FillingAnimation.tsx

import { useEffect, useRef, useState } from 'react';
import { Canvas, useFrame } from '@react-three/fiber';
import { OrbitControls } from '@react-three/drei';
import * as THREE from 'three';

interface MeshData {
  vertices: Float32Array;
  triangles: Uint32Array;
  elementCentroids: Float32Array;
}

interface FillingSnapshot {
  time: number;
  fillFraction: Float32Array;
  pressure: Float32Array;
  temperature: Float32Array;
}

interface FillingAnimationProps {
  simulationId: string;
  meshData: MeshData;
  snapshots: FillingSnapshot[];
}

export function FillingAnimation({ simulationId, meshData, snapshots }: FillingAnimationProps) {
  const [currentTime, setCurrentTime] = useState(0);
  const [isPlaying, setIsPlaying] = useState(false);
  const [displayMode, setDisplayMode] = useState<'fill' | 'pressure' | 'temperature'>('fill');

  const maxTime = snapshots[snapshots.length - 1]?.time || 1.0;

  useEffect(() => {
    if (!isPlaying) return;

    const interval = setInterval(() => {
      setCurrentTime(t => {
        const next = t + 0.02;
        if (next >= maxTime) {
          setIsPlaying(false);
          return maxTime;
        }
        return next;
      });
    }, 20);  // 50 FPS

    return () => clearInterval(interval);
  }, [isPlaying, maxTime]);

  const getCurrentSnapshot = (): FillingSnapshot => {
    if (snapshots.length === 0) return null;
    if (currentTime <= snapshots[0].time) return snapshots[0];
    if (currentTime >= snapshots[snapshots.length - 1].time) return snapshots[snapshots.length - 1];

    // Linear interpolation
    for (let i = 0; i < snapshots.length - 1; i++) {
      if (currentTime >= snapshots[i].time && currentTime <= snapshots[i + 1].time) {
        const t = (currentTime - snapshots[i].time) / (snapshots[i + 1].time - snapshots[i].time);
        const s0 = snapshots[i];
        const s1 = snapshots[i + 1];

        const fillFraction = new Float32Array(s0.fillFraction.length);
        const pressure = new Float32Array(s0.pressure.length);
        const temperature = new Float32Array(s0.temperature.length);

        for (let j = 0; j < fillFraction.length; j++) {
          fillFraction[j] = s0.fillFraction[j] + t * (s1.fillFraction[j] - s0.fillFraction[j]);
          pressure[j] = s0.pressure[j] + t * (s1.pressure[j] - s0.pressure[j]);
          temperature[j] = s0.temperature[j] + t * (s1.temperature[j] - s0.temperature[j]);
        }

        return { time: currentTime, fillFraction, pressure, temperature };
      }
    }
    return snapshots[snapshots.length - 1];
  };

  return (
    <div className="filling-animation h-full flex flex-col bg-gray-900">
      <div className="controls p-4 bg-gray-800 flex items-center gap-4">
        <button
          onClick={() => setIsPlaying(!isPlaying)}
          className="px-4 py-2 bg-blue-600 rounded hover:bg-blue-700 text-white"
        >
          {isPlaying ? 'Pause' : 'Play'}
        </button>

        <button
          onClick={() => setCurrentTime(0)}
          className="px-3 py-2 bg-gray-700 rounded hover:bg-gray-600 text-white"
        >
          Reset
        </button>

        <input
          type="range"
          min={0}
          max={maxTime}
          step={0.01}
          value={currentTime}
          onChange={e => setCurrentTime(parseFloat(e.target.value))}
          className="flex-1"
        />

        <span className="text-white font-mono min-w-[120px]">
          {currentTime.toFixed(3)}s / {maxTime.toFixed(3)}s
        </span>

        <select
          value={displayMode}
          onChange={e => setDisplayMode(e.target.value as any)}
          className="px-3 py-2 bg-gray-700 text-white rounded"
        >
          <option value="fill">Fill Fraction</option>
          <option value="pressure">Pressure (MPa)</option>
          <option value="temperature">Temperature (°C)</option>
        </select>
      </div>

      <div className="flex-1 relative">
        <Canvas camera={{ position: [0, 0, 200], fov: 50 }}>
          <ambientLight intensity={0.4} />
          <directionalLight position={[10, 10, 5]} intensity={0.8} />
          <directionalLight position={[-10, -10, -5]} intensity={0.3} />

          <PartMesh
            meshData={meshData}
            snapshot={getCurrentSnapshot()}
            displayMode={displayMode}
          />

          <OrbitControls enableDamping dampingFactor={0.05} />
          <gridHelper args={[500, 50, 0x444444, 0x222222]} />
          <axesHelper args={[100]} />
        </Canvas>
      </div>

      <ColorBar displayMode={displayMode} />
    </div>
  );
}

function PartMesh({ meshData, snapshot, displayMode }: {
  meshData: MeshData;
  snapshot: FillingSnapshot;
  displayMode: 'fill' | 'pressure' | 'temperature';
}) {
  const meshRef = useRef<THREE.Mesh>();
  const [geometry, setGeometry] = useState<THREE.BufferGeometry>(null);

  useEffect(() => {
    const geo = new THREE.BufferGeometry();
    geo.setAttribute('position', new THREE.BufferAttribute(meshData.vertices, 3));
    geo.setIndex(new THREE.BufferAttribute(meshData.triangles, 1));
    geo.computeVertexNormals();
    setGeometry(geo);
  }, [meshData]);

  useEffect(() => {
    if (!geometry || !snapshot) return;

    const colors = new Float32Array(meshData.vertices.length);
    const nTriangles = meshData.triangles.length / 3;

    for (let i = 0; i < nTriangles; i++) {
      const i0 = meshData.triangles[i * 3];
      const i1 = meshData.triangles[i * 3 + 1];
      const i2 = meshData.triangles[i * 3 + 2];

      let value = 0;
      let color: THREE.Color;

      if (displayMode === 'fill') {
        value = snapshot.fillFraction[i];
        color = getColorForFillFraction(value);
      } else if (displayMode === 'pressure') {
        value = snapshot.pressure[i] / 1e6;  // Pa to MPa
        color = getColorForPressure(value);
      } else {
        value = snapshot.temperature[i] - 273.15;  // K to °C
        color = getColorForTemperature(value);
      }

      // Set RGB for all 3 vertices of triangle
      for (const idx of [i0, i1, i2]) {
        colors[idx * 3] = color.r;
        colors[idx * 3 + 1] = color.g;
        colors[idx * 3 + 2] = color.b;
      }
    }

    geometry.setAttribute('color', new THREE.BufferAttribute(colors, 3));
    geometry.attributes.color.needsUpdate = true;
  }, [geometry, snapshot, displayMode]);

  if (!geometry) return null;

  return (
    <mesh ref={meshRef} geometry={geometry}>
      <meshStandardMaterial
        vertexColors
        side={THREE.DoubleSide}
        metalness={0.2}
        roughness={0.7}
      />
    </mesh>
  );
}

function getColorForFillFraction(f: number): THREE.Color {
  if (f < 0.01) return new THREE.Color(0.2, 0.2, 0.2);
  if (f < 0.33) return new THREE.Color(0, 0.5 + f * 1.5, 1);
  if (f < 0.67) return new THREE.Color(0, 1, 1 - (f - 0.33) * 3);
  if (f < 0.99) return new THREE.Color((f - 0.67) * 3, 1, 0);
  return new THREE.Color(1, 0, 0);
}

function getColorForPressure(p_mpa: number): THREE.Color {
  const t = Math.min(p_mpa / 100.0, 1.0);
  return new THREE.Color().setHSL(0.66 - t * 0.66, 1, 0.5);
}

function getColorForTemperature(t_c: number): THREE.Color {
  const tNorm = Math.max(0, Math.min(1, (t_c - 50) / 250));
  return new THREE.Color().setHSL(0.66 - tNorm * 0.66, 1, 0.5);
}

function ColorBar({ displayMode }: { displayMode: string }) {
  const labels = {
    fill: ['0%', '25%', '50%', '75%', '100%'],
    pressure: ['0 MPa', '25 MPa', '50 MPa', '75 MPa', '100 MPa'],
    temperature: ['50°C', '100°C', '150°C', '200°C', '250°C'],
  };

  return (
    <div className="color-bar p-4 bg-gray-800 flex items-center gap-4">
      <div className="w-64 h-8 rounded" style={{
        background: 'linear-gradient(to right, #0088ff, #00ff88, #ffff00, #ff0000)'
      }} />
      <div className="flex justify-between flex-1 text-white text-xs font-mono">
        {labels[displayMode].map((label, i) => (
          <span key={i}>{label}</span>
        ))}
      </div>
    </div>
  );
}
```

### 4. Mesh Generation Service (Python)

The Python service handles CAD import, midplane extraction, and mesh generation using Gmsh.

```python
# mesh-service/app/main.py

from fastapi import FastAPI, HTTPException, BackgroundTasks
from pydantic import BaseModel
import boto3
import gmsh
import trimesh
import numpy as np
from typing import List, Tuple
import json

app = FastAPI()

s3 = boto3.client('s3')

class MeshRequest(BaseModel):
    cad_file_url: str
    project_id: str
    target_element_size: float = 2.0  # mm

@app.post("/generate-mesh")
async def generate_mesh(req: MeshRequest, background_tasks: BackgroundTasks):
    # Download CAD from S3
    bucket, key = parse_s3_url(req.cad_file_url)
    local_path = f"/tmp/{req.project_id}.stl"
    s3.download_file(bucket, key, local_path)

    # Load with trimesh
    mesh = trimesh.load(local_path)

    # Extract midplane
    midplane_mesh = extract_midplane(mesh, req.target_element_size)

    # Upload to S3
    mesh_json = serialize_mesh(midplane_mesh)
    mesh_url = f"s3://{bucket}/meshes/{req.project_id}.json"
    s3.put_object(
        Bucket=bucket,
        Key=f"meshes/{req.project_id}.json",
        Body=json.dumps(mesh_json).encode('utf-8')
    )

    return {"mesh_url": mesh_url, "element_count": len(midplane_mesh['elements'])}

def extract_midplane(mesh: trimesh.Trimesh, target_size: float) -> dict:
    gmsh.initialize()
    gmsh.model.add("midplane")

    # Import STL geometry
    gmsh.merge(mesh.export(file_type='stl'))

    # Compute medial axis transform (simplified version)
    # In production: use skimage.morphology.skeletonize_3d on voxelized geometry

    # For MVP: detect dominant plane and project
    pca = np.linalg.pca(mesh.vertices)
    normal = pca[2]  # Normal to dominant plane

    # Project vertices to plane
    projected = project_to_plane(mesh.vertices, normal)

    # Generate 2D mesh on projected points
    gmsh.model.geo.addSurfaceLoop([1], 1)
    gmsh.model.addPhysicalGroup(2, [1], 1)
    gmsh.model.mesh.setSize(gmsh.model.getEntities(0), target_size / 1000.0)  # Convert to meters
    gmsh.model.mesh.generate(2)

    # Extract nodes and elements
    node_tags, node_coords, _ = gmsh.model.mesh.getNodes()
    elem_tags, elem_nodes = gmsh.model.mesh.getElementsByType(2)  # Type 2 = triangles

    nodes = np.array(node_coords).reshape(-1, 3)
    elements = np.array(elem_nodes).reshape(-1, 3) - 1  # Convert to 0-indexed

    # Compute thickness for each element by raycasting
    thicknesses = compute_thicknesses(elements, nodes, mesh)

    gmsh.finalize()

    return {
        "nodes": nodes.tolist(),
        "elements": elements.tolist(),
        "thicknesses": thicknesses.tolist(),
    }

def compute_thicknesses(elements, nodes, original_mesh):
    thicknesses = []
    for elem in elements:
        centroid = nodes[elem].mean(axis=0)
        # Raycast along thickness direction to find top/bottom surfaces
        thickness = raycast_thickness(centroid, original_mesh)
        thicknesses.append(thickness)
    return np.array(thicknesses)
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–5)

**Day 1:** Rust backend scaffold (Axum, SQLx, S3, Redis), `src/main.rs`, `src/config.rs`, `src/state.rs`, `docker-compose.yml` with Postgres/Redis/MinIO.

**Day 2:** Database schema `migrations/001_initial.sql` (9 tables), `src/db/models.rs` SQLx structs, seed 50 materials from CAMPUS database.

**Day 3:** Auth system: JWT tokens, OAuth 2.0 (Google/GitHub), `src/auth/mod.rs`, bcrypt password hashing, integration tests.

**Day 4:** CRUD handlers: `src/api/handlers/users.rs`, `projects.rs`, auth middleware, authorization checks.

**Day 5:** CAD upload: `src/api/handlers/cad.rs` multipart STL/STEP upload to S3, format validation, thumbnail generation.

### Phase 2 — Mesh Generation Service (Days 6–9)

**Day 6:** Python FastAPI service, `mesh-service/app/main.py`, Gmsh wrapper, uniform tetrahedral meshing, S3 upload.

**Day 7:** Midplane extraction: medial axis transform, project to 2D, assign thickness, validate volume match within 5%.

**Day 8:** Mesh quality checks: aspect ratio, skewness, automatic refinement, reject degenerates.

**Day 9:** Boundary conditions: gate node identification, wall edges, export BC JSON. Integration test: 100×100×2mm box, 5-10K elements in <30s.

### Phase 3 — Hele-Shaw Solver Core (Days 10–18)

**Day 10:** Solver workspace, `solver-core/src/material.rs` Cross-WLF viscosity, Tait pvT, unit tests vs CAMPUS data.

**Day 11:** `solver-core/src/mesh.rs` Mesh/Element structs, connectivity, spatial queries.

**Day 12-13:** `solver-core/src/petsc.rs` PETSc FFI, GMRES+AMG, COO→CSR conversion, unit tests (Laplace on unit square).

**Day 14-15:** `solver-core/src/filling.rs` HeleShawSolver, main loop, Picard iteration, fill front advancement, adaptive timestep.

**Day 16:** `solver-core/src/temperature.rs` energy equation with SUPG, viscous heating, mold cooling BC.

**Day 17:** `solver-core/src/frozen_layer.rs` frozen thickness δ(t), reduce h_eff, remove frozen elements.

**Day 18:** `solver-core/src/defects.rs` weld line (multi-direction inflow), air trap (flood fill), short shot detection.

### Phase 4 — Simulation Worker + Results (Days 19–22)

**Day 19:** `src/workers/simulation_worker.rs` Redis job consumer, download mesh, run solver, progress updates.

**Day 20:** `src/websocket/mod.rs` WebSocket progress streaming, broadcast per simulation.

**Day 21:** Results serialization: MessagePack + zstd compression, S3 upload, summary JSON.

**Day 22:** `src/api/handlers/results.rs` download endpoint, Three.js JSON format, downsample to 50K elements for browser.

### Phase 5 — Frontend Visualization (Days 23–29)

**Day 23:** React Three Fiber setup, `src/components/Viewer3D.tsx` basic 3D viewer with OrbitControls.

**Day 24:** `src/components/FillingAnimation.tsx` time slider, play/pause, interpolate snapshots, per-vertex coloring.

**Day 25:** Contour plots: toggle fill/pressure/temperature, color mapping, color bar legend, LOD mesh simplification.

**Day 26:** `src/components/DefectMarkers.tsx` weld line overlays, air trap markers, clickable details.

**Day 27:** `src/pages/Dashboard.tsx` project list, thumbnails, create project, drag-drop CAD upload.

**Day 28:** `src/pages/SimulationSetup.tsx` multi-step wizard: select material, process params, gate placement.

**Day 29:** `src/pages/SimulationProgress.tsx` WebSocket live progress, `SimulationResults.tsx` animation viewer, summary cards.

### Phase 6 — Material Database + Recipes (Days 30–33)

**Day 30:** `src/pages/MaterialBrowser.tsx` filter by class/manufacturer, fuzzy search, material cards.

**Day 31:** `src/pages/MaterialDetail.tsx` viscosity charts (η vs γ̇), pvT charts, recommended process windows.

**Day 32:** Custom material upload: CSV rheometer data, Python `/fit-viscosity` endpoint, scipy curve_fit, display fitted curve.

**Day 33:** Process recipes: save/apply successful params, public recipe sharing, rating system.

### Phase 7 — Billing + Plan Enforcement (Days 34–38)

**Day 34:** Stripe integration: `src/api/handlers/billing.rs` Checkout Sessions, webhooks, update user.plan.

**Day 35:** Plan limits: Free (3 projects, 10K elements, filling only), Pro (50K, packing), Advanced (foam, priority). Middleware enforcement.

**Day 36:** `src/pages/Usage.tsx` usage dashboard, quota progress bars, upgrade CTA.

**Day 37:** Organization billing: shared projects, permission levels, usage attribution.

**Day 38:** Test edge cases: project limit, element limit, downgrades, concurrent submissions.

### Phase 8 — Polish + Deployment (Days 39–42)

**Day 39:** Error handling: structured logging (tracing), user-friendly solver errors, Sentry integration.

**Day 40:** Performance: database indexes, CloudFront CDN, frontend code splitting, solver profiling. Target: 50K elements in <5 min.

**Day 41:** Documentation: landing page, quick start guide, video tutorial, API docs (OpenAPI), material DB guide.

**Day 42:** Production deploy: AWS ECS Fargate (API), EC2 c6i.2xlarge (workers), RDS Postgres, ElastiCache Redis. CI/CD, Grafana dashboards, PagerDuty alerts, k6 load testing.

---

## Critical Files

```
foamsim/
├── backend/
│   ├── src/
│   │   ├── main.rs
│   │   ├── config.rs
│   │   ├── state.rs
│   │   ├── auth/mod.rs
│   │   ├── api/handlers/{simulation.rs, results.rs, materials.rs, billing.rs}
│   │   ├── db/models.rs
│   │   └── workers/simulation_worker.rs
│   └── migrations/001_initial.sql
├── solver-core/
│   └── src/{material.rs, mesh.rs, petsc.rs, filling.rs, temperature.rs, defects.rs}
├── mesh-service/
│   └── app/{main.py, gmsh_wrapper.py, midplane.py, quality.py}
├── frontend/
│   └── src/{components/FillingAnimation.tsx, pages/Dashboard.tsx}
└── docker-compose.yml
```

---

## Solver Validation Suite

### Benchmark 1: 1D Channel (Analytical)

**Setup:** 100×10×2mm channel, gate at x=0, PP η₀=500 Pa·s, Q=1e-6 m³/s.

**Expected:** Fill time 2.0±0.05s, max pressure 7.2±0.3 MPa, linear P(x) (R²>0.99).

**Pass:** Fill time within 2.5%, pressure within 5%.

---

### Benchmark 2: Center-Gated Disk

**Setup:** 100mm diameter, 2mm thick, ABS, 10 cm³/s, T_melt=250°C, T_mold=60°C.

**Expected:** Fill time 1.85±0.10s (Moldflow: 1.82s), max pressure 25±3 MPa (Moldflow: 24.3 MPa), radial symmetry (sector std dev <5%), weld line at opposite edge.

**Pass:** Within 10% of Moldflow fill time, 15% pressure.

---

### Benchmark 3: Box with Ribs

**Setup:** 80×60×2mm box + 4 ribs (10×3mm), PA66 30% glass, gate at corner, 1.5s injection.

**Expected:** Fill 1.45±0.15s, max pressure 45±8 MPa, ribs fill first (>80% before corners), air trap in far corner, weld lines at 4 corners.

**Pass:** Ribs fill early, air trap detected, 4 weld lines, pressure within 20%.

---

### Benchmark 4: Thin-Wall Fast Freeze

**Setup:** 140×70×0.8mm smartphone cover, PC, T_melt=300°C, T_mold=90°C, 0.6s injection.

**Expected:** Fill 0.58±0.08s, short shot if t>0.8s, frozen layer 0.15-0.25mm (30-60% wall), pressure spike >80 MPa near end.

**Pass:** Complete fill at 0.6s, short shot at 0.9s.

---

### Benchmark 5: Multi-Gate Balancing

**Setup:** 150×100×2.5mm plate, 2 gates at x=25mm and x=125mm, PP, T_melt=220°C.

**Expected:** Fill 2.1±0.2s, weld line at x=75mm (balanced), weld temp 160-180°C, quality 0.6-0.7. If 10% flow imbalance: weld shifts to x=80mm.

**Pass:** Weld position within 5mm, shifts correctly with imbalance.

---

## Verification Checklist

### Solver Correctness
- [ ] All 5 benchmarks pass criteria
- [ ] Cross-WLF matches CAMPUS curves (R²>0.95) for PP/ABS/PA6
- [ ] Tait pvT density ±2% vs literature (0-100 MPa, 100-300°C)
- [ ] Energy balance: heat in - heat out = enthalpy change ±10%
- [ ] Mass conservation: filled volume = ∫Q dt ±1%

### Mesh Quality
- [ ] >95% elements aspect ratio <5
- [ ] Mesh volume matches STL ±5%
- [ ] Gate nodes within 2mm of user location
- [ ] 100% recall on boundary edges

### Performance
- [ ] 10K mesh generation <30s
- [ ] 50K simulation <5 min (4-core)
- [ ] PETSc <50 iterations/timestep
- [ ] Frontend 30+ FPS for 100K triangles

### Defect Detection
- [ ] Weld lines: >90% recall, >85% precision
- [ ] Air traps: 100% recall on dead-end pockets
- [ ] False positive rate <10%
- [ ] Short shot flagged at 30% reduced injection time

### API/Integration
- [ ] Correct HTTP status codes
- [ ] WebSocket updates every 1-2s
- [ ] S3 uploads <100MB with retry
- [ ] Stripe webhooks <5s to update plan

### Security
- [ ] JWT 24h expiry, refresh 30d
- [ ] Bcrypt cost 12 (~300ms)
- [ ] Authorization checks pass
- [ ] No SQL injection (parameterized queries)
- [ ] S3 presigned URLs 1h expiry

---

## Deployment Architecture

```
CloudFront CDN → ALB → ECS Fargate (API 2×2vCPU) → EC2 Auto Scaling (Workers c6i.2xlarge)
                   ↓                                      ↓
               RDS Postgres + ElastiCache Redis + S3 (CAD/mesh/results)
```

**Scaling:** API: CPU>70% or >1000 req/min → max 10 tasks. Workers: queue depth>10/worker → max 20 instances.

**Cost:** $485/month baseline (2 API, 2 workers, db.t4g.medium, 1TB S3).

---

## Post-MVP Roadmap

### v1.1 — 3D Navier-Stokes + Packing (Months 2-3)
VOF 3D filling for thick parts, GPU acceleration (CUDA), fountain flow visualization, compressible packing phase, gate freeze-off, volumetric shrinkage.

### v1.2 — Foam Injection (Month 4)
Classical nucleation theory, cell growth ODEs (Amon-Denson), skin-core structure, CBA decomposition kinetics (azodicarbonamide), cell size distribution.

### v1.3 — Fiber Orientation + Warpage (Month 5)
Folgar-Tucker fiber orientation tensor, anisotropic shrinkage, thermo-viscoelastic FEM warpage, warpage compensation tool.

### v1.4 — Thermoforming + Rubber (Month 6)
Sheet heating (IR), hyperelastic stretching (Mooney-Rivlin), plug optimization, wall thickness prediction, rubber vulcanization (Kamal-Sourour), scorch prediction.

### v1.5 — AI Optimization (Months 7-8)
DOE multi-parameter sweep, Gaussian process response surfaces, Pareto fronts (cycle time vs quality), AI process window advisor (gradient boosted trees), gate location optimization (genetic algorithm).

### v1.6 — Enterprise (Month 9)
Real-time collaboration (WebSocket), version history, on-premise K8s, SSO (SAML/LDAP), multi-plant recipe management, audit trails.
