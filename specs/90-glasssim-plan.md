# 90. GlassSim — Glass Forming and Optical Fiber Simulation Platform

## Implementation Plan

**MVP Scope:** Browser-based glass process simulation platform with 2D axisymmetric viscous flow solver implementing temperature-dependent Vogel-Fulcher-Tammann (VFT) viscosity spanning 12+ orders of magnitude (10¹ to 10¹³ Pa·s), radiative heat transfer in semitransparent glass using Rosseland diffusion approximation for optically thick regions, optical fiber draw tower simulation predicting neck-down profile geometry and fiber tension from preform (25-150mm) to fiber (125µm) under viscous drawing with free surface tracking, annealing simulation with Tool-Narayanaswamy-Moynihan (TNM) structural relaxation model computing fictive temperature evolution and residual stress through glass transition, glass composition database with VFT parameters and spectral absorption data for 20 common compositions (soda-lime, borosilicate, E-glass, fused silica), temperature/viscosity/stress contour visualization rendered via WebGL with Three.js for rotating 3D axisymmetric geometry view, process parameter sensitivity analysis sweeping furnace temperature and draw speed, Stripe billing with three tiers (Student Free / Pro $149/mo / Advanced $349/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| FEM Solver Core | Rust (native + WASM) | Stabilized Galerkin FEM for high-viscosity Stokes flow, direct sparse solver |
| VFT Viscosity | Rust solver module | η(T) = η₀·exp(A/(T-T₀)), 12+ order magnitude range handling |
| Radiation Solver | Rust module | Rosseland diffusion approximation, spectral band absorption |
| Structural Relaxation | Rust module | TNM model for fictive temperature evolution, viscoelastic stress |
| WASM Compilation | `wasm-pack` + `wasm-bindgen` | Client-side 1D/2D annealing simulation ≤5K DOF |
| AI/ML Optimization | Python 3.12 (FastAPI) | Bayesian optimization for process parameters, surrogate modeling |
| Database | PostgreSQL 16 | Projects, users, simulations, glass compositions, material properties |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | Mesh files, simulation results (field data), geometry snapshots |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| 3D Visualization | Three.js + WebGL | Axisymmetric mesh rendering, temperature/stress contours, rotation |
| 2D Plotting | Canvas API + D3 | Temperature profiles, viscosity curves, process parameter sweeps |
| Real-time | WebSocket (Axum) | Live simulation progress, convergence monitoring, residual tracking |
| Job Queue | Redis 7 + Tokio tasks | Server-side simulation job management, priority queue |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | WASM bundle delivery, static frontend assets, mesh file caching |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, solver convergence stats, error tracking |
| CI/CD | GitHub Actions | Rust tests, WASM build pipeline, Docker image push |

### Key Architecture Decisions

1. **Rust FEM solver for glass viscosity range handling**: Glass viscosity from melt (10 Pa·s) to solid (10¹³ Pa·s) spans 12+ orders of magnitude, governed by VFT equation η(T) = η₀·exp(A/(T-T₀)). Standard CFD solvers struggle with this extreme range. Our Rust FEM solver uses stabilized Galerkin methods (SUPG for advection, PSPG for pressure) and logarithmic viscosity formulation to maintain numerical stability. Direct sparse solver (UMFPACK) handles ill-conditioned matrices from viscosity gradients.

2. **Axisymmetric 2D formulation for fiber draw and annealing**: Optical fiber draw and tube/rod annealing exhibit axial symmetry, reducing 3D simulation to 2D (r, z) with 100× speedup and WASM feasibility. Governing equations in cylindrical coordinates with 1/r geometric terms. This covers 80% of glass process engineering use cases while maintaining sub-minute solve times.

3. **Rosseland diffusion for semitransparent radiation**: Glass is semitransparent to visible/near-IR (absorption coefficient α ~ 1-100 m⁻¹ depending on composition). Full radiative transfer (DOM/P1) is expensive. Rosseland diffusion approximation treats radiation as enhanced thermal conductivity k_eff = k + 16σn²T³/(3α), valid for optically thick glass (thickness >> 1/α). Captures 90% of radiative effects with 10× speedup vs. DOM.

4. **TNM structural relaxation for residual stress prediction**: Residual stress in annealed/tempered glass arises from structural relaxation through glass transition (Tg ± 50°C). Tool-Narayanaswamy-Moynihan model tracks fictive temperature Tf (structure state) via relaxation ODE: dTf/dt = -(Tf - T)/τ(T, Tf) where τ is WLF-shifted relaxation time. Stress computed from thermal strain with structure-dependent modulus. This is the industry-standard model (used by Corning, Schott) unavailable in general FEA tools.

5. **S3 for mesh and result storage with PostgreSQL metadata**: Simulation meshes (1K-100K elements, 100KB-10MB) and field results (temperature, viscosity, stress arrays) stored in S3, while PostgreSQL holds metadata (glass composition, VFT parameters, boundary conditions, convergence history). Enables parametric studies with 1000+ simulations without database bloat. Results streamed to frontend via presigned S3 URLs.

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";  -- For fuzzy text search on glass compositions

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
    organization TEXT,  -- Company or university name
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX users_email_idx ON users(email);
CREATE INDEX users_stripe_idx ON users(stripe_customer_id);

-- Organizations (for multi-user collaboration)
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

-- Glass Compositions (built-in + user-uploaded)
CREATE TABLE glass_compositions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,  -- e.g., "Soda-Lime Float Glass", "Borosilicate 3.3", "E-Glass"
    category TEXT NOT NULL,  -- soda_lime | borosilicate | aluminosilicate | fused_silica | e_glass | specialty
    composition_oxide JSONB,  -- {"SiO2": 72.0, "Na2O": 14.0, "CaO": 9.0, "MgO": 4.0, "Al2O3": 1.0} (wt%)
    vft_params JSONB NOT NULL,  -- {"eta_inf": 1e-3, "A": 4000, "T0": 500} (Pa·s, K, K) for η = η_inf * exp(A/(T-T0))
    density REAL NOT NULL,  -- kg/m³ at 20°C
    thermal_conductivity REAL NOT NULL,  -- W/(m·K) at 300K (or JSONB for T-dependent)
    specific_heat REAL NOT NULL,  -- J/(kg·K) at 300K
    thermal_expansion REAL NOT NULL,  -- 1/K, linear CTE
    youngs_modulus REAL NOT NULL,  -- GPa at room temperature
    poisson_ratio REAL NOT NULL,  -- dimensionless
    glass_transition_temp REAL NOT NULL,  -- K, approximate Tg (50% relaxation)
    softening_point REAL NOT NULL,  -- K, η = 10^6.6 Pa·s
    annealing_point REAL NOT NULL,  -- K, η = 10^12 Pa·s
    strain_point REAL NOT NULL,  -- K, η = 10^13.5 Pa·s
    absorption_spectrum JSONB,  -- {"bands": [{"lambda_min": 0.4, "lambda_max": 0.7, "alpha": 5}, ...]} (μm, m⁻¹)
    refractive_index REAL,  -- At 589nm (sodium D-line)
    notes TEXT,
    is_builtin BOOLEAN DEFAULT true,
    uploaded_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX glass_comp_name_trgm_idx ON glass_compositions USING gin(name gin_trgm_ops);
CREATE INDEX glass_comp_category_idx ON glass_compositions(category);

-- Projects (simulation workspace)
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    process_type TEXT NOT NULL,  -- fiber_draw | annealing | float_glass | container_forming | bending | tempering
    geometry_data JSONB NOT NULL DEFAULT '{}',  -- Geometry definition (preform dimensions, boundary conditions)
    glass_composition_id UUID REFERENCES glass_compositions(id),
    settings JSONB DEFAULT '{}',  -- Simulation settings (mesh density, solver tolerance, timestep)
    is_public BOOLEAN NOT NULL DEFAULT false,
    forked_from UUID REFERENCES projects(id) ON DELETE SET NULL,
    thumbnail_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);
CREATE INDEX projects_org_idx ON projects(org_id);
CREATE INDEX projects_process_type_idx ON projects(process_type);
CREATE INDEX projects_updated_idx ON projects(updated_at DESC);

-- Simulation Runs
CREATE TABLE simulations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    simulation_type TEXT NOT NULL,  -- steady_state | transient_thermal | fiber_draw | annealing_schedule
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | running | completed | failed | cancelled
    execution_mode TEXT NOT NULL DEFAULT 'wasm',  -- wasm | server
    parameters JSONB NOT NULL DEFAULT '{}',  -- Simulation-specific parameters (furnace temp, draw speed, etc.)
    mesh_url TEXT,  -- S3 URL for mesh file (if generated)
    dof_count INTEGER NOT NULL DEFAULT 0,  -- Degrees of freedom (mesh size indicator)
    results_url TEXT,  -- S3 URL for result field data
    results_summary JSONB,  -- Quick-access summary (max temp, max stress, fiber tension, etc.)
    convergence_data JSONB,  -- [{iteration, residual_norm, solve_time_ms}]
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
    cores_allocated INTEGER DEFAULT 4,
    memory_mb INTEGER DEFAULT 8192,
    priority INTEGER DEFAULT 0,  -- Higher = more urgent (Advanced plan gets +10)
    progress_pct REAL DEFAULT 0.0,
    current_iteration INTEGER DEFAULT 0,
    total_iterations INTEGER,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX jobs_sim_idx ON simulation_jobs(simulation_id);
CREATE INDEX jobs_worker_idx ON simulation_jobs(worker_id);

-- Material Property Fitting Jobs (Python service)
CREATE TABLE material_fit_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    job_type TEXT NOT NULL,  -- vft_fit | relaxation_fit | absorption_spectrum
    input_data JSONB NOT NULL,  -- User-uploaded viscosity measurements or DSC data
    output_params JSONB,  -- Fitted VFT parameters or relaxation times
    status TEXT NOT NULL DEFAULT 'pending',
    error_message TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at TIMESTAMPTZ
);
CREATE INDEX material_fit_user_idx ON material_fit_jobs(user_id);

-- Process Optimization Runs (Bayesian optimization for process parameters)
CREATE TABLE optimization_runs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    objective TEXT NOT NULL,  -- minimize_stress | target_fiber_diameter | minimize_defects
    parameters JSONB NOT NULL,  -- Parameter bounds (furnace_temp: [1800, 2200], draw_speed: [0.5, 5.0])
    iterations_completed INTEGER DEFAULT 0,
    iterations_total INTEGER NOT NULL,
    best_params JSONB,  -- Best parameter set found
    best_objective_value REAL,
    status TEXT NOT NULL DEFAULT 'pending',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at TIMESTAMPTZ
);
CREATE INDEX opt_runs_project_idx ON optimization_runs(project_id);
CREATE INDEX opt_runs_status_idx ON optimization_runs(status);

-- Usage Tracking (for billing enforcement)
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,  -- simulation_minutes | dof_hours | optimization_iterations
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
    pub organization: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct GlassComposition {
    pub id: Uuid,
    pub name: String,
    pub category: String,
    pub composition_oxide: serde_json::Value,  // {"SiO2": 72.0, ...}
    pub vft_params: serde_json::Value,  // {"eta_inf": 1e-3, "A": 4000, "T0": 500}
    pub density: f32,  // kg/m³
    pub thermal_conductivity: f32,  // W/(m·K)
    pub specific_heat: f32,  // J/(kg·K)
    pub thermal_expansion: f32,  // 1/K
    pub youngs_modulus: f32,  // GPa
    pub poisson_ratio: f32,
    pub glass_transition_temp: f32,  // K
    pub softening_point: f32,  // K
    pub annealing_point: f32,  // K
    pub strain_point: f32,  // K
    pub absorption_spectrum: Option<serde_json::Value>,
    pub refractive_index: Option<f32>,
    pub notes: Option<String>,
    pub is_builtin: bool,
    pub uploaded_by: Option<Uuid>,
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
    pub process_type: String,  // fiber_draw | annealing | ...
    pub geometry_data: serde_json::Value,
    pub glass_composition_id: Option<Uuid>,
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
    pub mesh_url: Option<String>,
    pub dof_count: i32,
    pub results_url: Option<String>,
    pub results_summary: Option<serde_json::Value>,
    pub convergence_data: Option<serde_json::Value>,
    pub error_message: Option<String>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub wall_time_ms: Option<i32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct VftParams {
    pub eta_inf: f64,  // Pa·s, low-temperature viscosity limit
    pub A: f64,  // K, Vogel-Fulcher A parameter
    pub T0: f64,  // K, Vogel-Fulcher T0 (divergence temperature)
}

impl VftParams {
    /// Compute viscosity η(T) = η_inf * exp(A / (T - T0))
    pub fn viscosity(&self, temperature_k: f64) -> f64 {
        if temperature_k <= self.T0 + 1.0 {
            return 1e20;  // Clamped for numerical stability
        }
        self.eta_inf * (self.A / (temperature_k - self.T0)).exp()
    }

    /// Compute log10(viscosity) for better numerical conditioning
    pub fn log10_viscosity(&self, temperature_k: f64) -> f64 {
        if temperature_k <= self.T0 + 1.0 {
            return 20.0;
        }
        self.eta_inf.log10() + self.A / (temperature_k - self.T0) / 2.302585093
    }
}

#[derive(Debug, Deserialize, Serialize)]
pub struct FiberDrawParams {
    pub furnace_temp_k: f64,  // K, furnace hot zone temperature
    pub draw_speed_m_per_s: f64,  // m/s, fiber draw speed at take-up
    pub preform_feed_rate_m_per_s: f64,  // m/s, preform feed into furnace
    pub preform_diameter_mm: f64,  // mm
    pub target_fiber_diameter_um: f64,  // μm
    pub furnace_length_mm: f64,  // mm, hot zone length
}

#[derive(Debug, Deserialize, Serialize)]
pub struct AnnealingParams {
    pub initial_temp_k: f64,  // K, uniform initial temperature
    pub cooling_schedule: Vec<CoolingSegment>,  // Piecewise cooling schedule
    pub geometry_type: String,  // "cylinder" | "plate" | "sphere"
    pub characteristic_dimension_mm: f64,  // Radius or thickness
    pub surface_heat_transfer_coeff: f64,  // W/(m²·K), h for convection boundary
    pub ambient_temp_k: f64,  // K
}

#[derive(Debug, Deserialize, Serialize)]
pub struct CoolingSegment {
    pub duration_s: f64,
    pub cooling_rate_k_per_s: f64,  // Negative for cooling
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(tag = "type")]
pub enum SimulationParams {
    FiberDraw(FiberDrawParams),
    Annealing(AnnealingParams),
    SteadyStateThermal {
        boundary_conditions: serde_json::Value,
    },
}
```

---

## Solver Architecture Deep-Dive

### Governing Equations and Discretization

GlassSim's core solver implements **coupled thermal-fluid-structural analysis** for glass processing. The three coupled systems are:

#### 1. Energy Equation (Heat Transfer with Radiation)

For glass with temperature-dependent properties and radiative transport:

```
ρ c_p (∂T/∂t + v·∇T) = ∇·(k ∇T) + ∇·q_rad + Q_viscous

where:
  ρ = density (kg/m³)
  c_p = specific heat (J/(kg·K))
  T = temperature (K)
  v = velocity field (m/s)
  k = thermal conductivity (W/(m·K))
  q_rad = radiative heat flux (W/m²)
  Q_viscous = η(∇v + ∇v^T):∇v (viscous dissipation, typically negligible for glass)
```

**Rosseland diffusion approximation** for semitransparent radiation:

```
q_rad = -k_rad ∇T

where k_rad = 16σn²T³/(3α)  (radiative conductivity)
  σ = Stefan-Boltzmann constant = 5.67e-8 W/(m²·K⁴)
  n = refractive index (~1.5 for glass)
  α = absorption coefficient (m⁻¹, wavelength-averaged)
```

This allows radiation to be treated as enhanced thermal conductivity: **k_eff = k + k_rad**.

#### 2. Momentum Equation (Viscous Stokes Flow)

For high-viscosity creeping flow (Re << 1), inertia is negligible:

```
∇·σ = ρg

where σ = -pI + 2η(T)D  (stress tensor)
  p = pressure (Pa)
  I = identity tensor
  η(T) = VFT viscosity (Pa·s)
  D = (∇v + ∇v^T)/2  (strain rate tensor)
  g = gravity (m/s²)

∇·v = 0  (incompressibility)
```

**VFT viscosity model** (12+ orders of magnitude variation):

```
η(T) = η_inf · exp(A / (T - T0))

Typical soda-lime glass:
  η_inf = 10^(-3.5) Pa·s
  A = 4500 K
  T0 = 520 K

At T=1000K: η ≈ 10^4 Pa·s (forming range)
At T=800K:  η ≈ 10^8 Pa·s (annealing range)
At T=700K:  η ≈ 10^13 Pa·s (solid, below strain point)
```

#### 3. Structural Relaxation (TNM Model for Residual Stress)

**Fictive temperature** Tf tracks the structural state of glass (equilibrium at high T, frozen-in at low T):

```
dTf/dt = -(Tf - T) / τ(T, Tf)

where τ(T, Tf) = τ0 · exp(x·A/(Tf - T0) + (1-x)·A/(T - T0))
  τ0 = relaxation time constant (s)
  x = structure parameter (0.5 to 0.7)
  A, T0 = same as VFT parameters
```

**Thermal stress** from mismatch between cooling rate and structural relaxation:

```
σ_thermal = E/(1-ν) · α · (T - Tf)

where E = Young's modulus (GPa)
  ν = Poisson's ratio
  α = thermal expansion coefficient (1/K)
```

Residual stress frozen in when cooling rate exceeds relaxation rate (below Tg).

### Finite Element Discretization (Galerkin FEM)

**Taylor-Hood P2-P1 elements** for velocity-pressure (stable mixed formulation):
- Velocity: quadratic Lagrange polynomials (6 nodes per triangle in 2D)
- Pressure: linear Lagrange polynomials (3 nodes per triangle)
- Temperature: quadratic Lagrange (same as velocity)

**Weak form** for thermal equation (steady-state):

```
∫_Ω (k + k_rad(T)) ∇T · ∇φ dΩ = ∫_∂Ω q_boundary · φ dS

Nonlinear due to k_rad(T³) → Newton-Raphson iteration
```

**Weak form** for Stokes flow with VFT viscosity:

```
∫_Ω 2η(T) D(v) : D(w) dΩ - ∫_Ω p ∇·w dΩ = ∫_Ω ρg · w dΩ
∫_Ω q ∇·v dΩ = 0

where w = velocity test function, q = pressure test function
Nonlinear due to η(T) → iterative solve coupled with thermal
```

**Logarithmic viscosity formulation** for numerical stability:

Instead of solving with η directly (range 10¹ to 10¹³), solve with ζ = log₁₀(η):

```
η = 10^ζ  where ζ = log₁₀(η_inf) + A/(T - T0)/ln(10)

Jacobian dη/dT = -η · A / (T - T0)²
```

This avoids matrix ill-conditioning from extreme viscosity ratios.

### Client/Server Split (WASM Threshold)

```
Simulation requested → DOF count extracted
    │
    ├── ≤5000 DOF → WASM solver (browser)
    │   ├── 1D annealing (radial heat transfer)
    │   ├── 2D annealing (axisymmetric cylinder)
    │   ├── Instant results (<10s)
    │   └── No server cost
    │
    └── >5000 DOF → Server solver (Rust native)
        ├── 2D fiber draw (complex neck-down mesh)
        ├── Full transient annealing with TNM
        ├── Job queued via Redis
        ├── Worker picks up, runs native solver
        ├── Progress streamed via WebSocket
        └── Results stored in S3
```

The 5000 DOF threshold was chosen because:
- WASM solver handles 5K DOF 2D steady-state thermal in <5 seconds
- 5K DOF covers: 1D radial annealing (100 nodes), 2D cylinder annealing (50×50 mesh)
- Above 5K DOF: fiber draw with refined neck-down mesh (10K-50K), full 3D (future)

---

## Architecture Deep-Dives

### 1. Simulation API Handler (Rust/Axum)

The primary endpoint receives a simulation request, validates geometry and parameters, decides between WASM and server execution, and for server execution enqueues a job with Redis and streams progress via WebSocket.

```rust
// src/api/handlers/simulation.rs

use axum::{
    extract::{Path, State, Json},
    http::StatusCode,
    response::IntoResponse,
};
use uuid::Uuid;
use crate::{
    db::models::{Simulation, SimulationParams, Project},
    solver::mesh::estimate_dof_count,
    state::AppState,
    auth::Claims,
    error::ApiError,
};

#[derive(serde::Deserialize)]
pub struct CreateSimulationRequest {
    pub simulation_type: String,  // steady_state | transient_thermal | fiber_draw | annealing_schedule
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

    // 2. Parse parameters and estimate DOF count
    let params: SimulationParams = serde_json::from_value(req.parameters.clone())?;
    let dof_count = estimate_dof_count(&project.geometry_data, &params)?;

    // 3. Check plan limits
    let user = sqlx::query_as!(
        crate::db::models::User,
        "SELECT * FROM users WHERE id = $1",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    if user.plan == "free" && dof_count > 1000 {
        return Err(ApiError::PlanLimit(
            "Free plan supports simulations up to 1000 DOF. Upgrade to Pro for 10K DOF."
        ));
    }

    if user.plan == "pro" && dof_count > 10000 {
        return Err(ApiError::PlanLimit(
            "Pro plan supports simulations up to 10K DOF. Upgrade to Advanced for unlimited."
        ));
    }

    // 4. Determine execution mode
    let execution_mode = if dof_count <= 5000 && req.simulation_type != "fiber_draw" {
        "wasm"
    } else {
        "server"
    };

    // 5. Create simulation record
    let sim = sqlx::query_as!(
        Simulation,
        r#"INSERT INTO simulations
            (project_id, user_id, simulation_type, status, execution_mode,
             parameters, dof_count)
        VALUES ($1, $2, $3, $4, $5, $6, $7)
        RETURNING *"#,
        project_id,
        claims.user_id,
        req.simulation_type,
        if execution_mode == "wasm" { "pending" } else { "pending" },
        execution_mode,
        req.parameters,
        dof_count as i32,
    )
    .fetch_one(&state.db)
    .await?;

    // 6. For server execution, enqueue job
    if execution_mode == "server" {
        let job = sqlx::query_as!(
            crate::db::models::SimulationJob,
            r#"INSERT INTO simulation_jobs (simulation_id, priority, cores_allocated)
            VALUES ($1, $2, $3) RETURNING *"#,
            sim.id,
            if user.plan == "advanced" { 10 } else { 0 },
            if dof_count > 20000 { 8 } else { 4 },
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

### 2. VFT Viscosity Solver Core (Rust — shared between WASM and native)

The core viscous flow solver with temperature-dependent VFT viscosity and stabilized FEM for extreme viscosity ratios.

```rust
// solver-core/src/viscosity.rs

use crate::material::VftParams;

/// Compute viscosity field from temperature field using VFT model
pub fn compute_viscosity_field(
    temperatures: &[f64],
    vft: &VftParams,
) -> Vec<f64> {
    temperatures.iter()
        .map(|&T| vft.viscosity(T))
        .collect()
}

/// Compute viscosity Jacobian dη/dT for Newton-Raphson
pub fn viscosity_jacobian(temperature_k: f64, vft: &VftParams) -> f64 {
    if temperature_k <= vft.T0 + 1.0 {
        return 0.0;  // Clamped
    }
    let eta = vft.viscosity(temperature_k);
    -eta * vft.A / ((temperature_k - vft.T0).powi(2))
}

// solver-core/src/fem/stokes.rs

use nalgebra_sparse::{CsrMatrix, CooMatrix};
use crate::mesh::Mesh2D;
use crate::material::VftParams;

pub struct StokesSystem {
    pub mesh: Mesh2D,
    pub vft: VftParams,
    pub gravity: [f64; 2],  // [gx, gy] in m/s²
    pub matrix: CooMatrix<f64>,  // [(2*n_v + n_p) × (2*n_v + n_p)] velocity-pressure system
    pub rhs: Vec<f64>,
    pub solution: Vec<f64>,  // [vx_1, vy_1, ..., vx_nv, vy_nv, p_1, ..., p_np]
}

impl StokesSystem {
    pub fn new(mesh: Mesh2D, vft: VftParams, gravity: [f64; 2]) -> Self {
        let n_vel_nodes = mesh.nodes.len();  // P2 velocity nodes
        let n_pressure_nodes = mesh.n_p1_nodes();  // P1 pressure nodes
        let size = 2 * n_vel_nodes + n_pressure_nodes;

        Self {
            mesh,
            vft,
            gravity,
            matrix: CooMatrix::new(size, size),
            rhs: vec![0.0; size],
            solution: vec![0.0; size],
        }
    }

    /// Assemble Stokes system with VFT viscosity from temperature field
    pub fn assemble(&mut self, temperatures: &[f64]) -> Result<(), String> {
        self.matrix = CooMatrix::new(self.matrix.nrows(), self.matrix.ncols());
        self.rhs.fill(0.0);

        let n_vel = self.mesh.nodes.len();
        let n_p = self.mesh.n_p1_nodes();

        // Loop over elements (P2 triangles for velocity, P1 for pressure)
        for elem in &self.mesh.elements {
            // Quadrature: 3-point Gaussian quadrature for triangles
            let quad_pts = [(1.0/6.0, 1.0/6.0, 1.0/6.0),
                            (2.0/3.0, 1.0/6.0, 1.0/6.0),
                            (1.0/6.0, 2.0/3.0, 1.0/6.0)];

            for &(xi, eta, weight) in &quad_pts {
                // Evaluate temperature at quadrature point (from P2 interpolation)
                let T_quad = self.interpolate_temperature(elem, temperatures, xi, eta);
                let viscosity = self.vft.viscosity(T_quad);

                // P2 velocity shape functions and gradients
                let (N_vel, dN_vel) = self.mesh.shape_functions_p2(elem, xi, eta);
                let jac_det = self.mesh.jacobian_determinant(elem, &dN_vel);

                // P1 pressure shape functions
                let N_p = self.mesh.shape_functions_p1(elem, xi, eta);

                // Assemble viscous term: ∫ 2η D(v):D(w) dΩ
                for i in 0..6 {  // P2 has 6 nodes per element
                    let gi_x = elem.nodes[i];
                    let gi_y = elem.nodes[i];

                    for j in 0..6 {
                        let gj_x = elem.nodes[j];
                        let gj_y = elem.nodes[j];

                        // K_ij = 2η ∫ (∂Ni/∂x ∂Nj/∂x + 0.5 ∂Ni/∂y ∂Nj/∂y) dΩ
                        let k_xx = 2.0 * viscosity * (dN_vel[i][0] * dN_vel[j][0] +
                                                       0.5 * dN_vel[i][1] * dN_vel[j][1]) * jac_det * weight;
                        let k_yy = 2.0 * viscosity * (dN_vel[i][1] * dN_vel[j][1] +
                                                       0.5 * dN_vel[i][0] * dN_vel[j][0]) * jac_det * weight;
                        let k_xy = 2.0 * viscosity * 0.5 * (dN_vel[i][0] * dN_vel[j][1] +
                                                             dN_vel[i][1] * dN_vel[j][0]) * jac_det * weight;

                        self.matrix.push(2*gi_x, 2*gj_x, k_xx);
                        self.matrix.push(2*gi_y+1, 2*gj_y+1, k_yy);
                        self.matrix.push(2*gi_x, 2*gj_y+1, k_xy);
                        self.matrix.push(2*gi_y+1, 2*gj_x, k_xy);
                    }

                    // Assemble pressure coupling: -∫ p ∇·w dΩ and ∫ q ∇·v dΩ
                    for k in 0..3 {  // P1 has 3 nodes per element
                        let gk_p = elem.pressure_nodes[k] + 2*n_vel;
                        // B_T block: -∫ ∇·w p dΩ
                        self.matrix.push(2*gi_x, gk_p, -N_p[k] * dN_vel[i][0] * jac_det * weight);
                        self.matrix.push(2*gi_y+1, gk_p, -N_p[k] * dN_vel[i][1] * jac_det * weight);
                        // B block: ∫ q ∇·v dΩ
                        self.matrix.push(gk_p, 2*gi_x, -N_p[k] * dN_vel[i][0] * jac_det * weight);
                        self.matrix.push(gk_p, 2*gi_y+1, -N_p[k] * dN_vel[i][1] * jac_det * weight);
                    }

                    // Gravity RHS: ∫ ρg·w dΩ
                    let rho = 2500.0;  // kg/m³ typical glass density
                    self.rhs[2*gi_x] += rho * self.gravity[0] * N_vel[i] * jac_det * weight;
                    self.rhs[2*gi_y+1] += rho * self.gravity[1] * N_vel[i] * jac_det * weight;
                }
            }
        }

        Ok(())
    }

    /// Solve linear system Ax = b using sparse direct solver
    pub fn solve(&mut self) -> Result<(), String> {
        let csr = CsrMatrix::from(&self.matrix);
        // Use UMFPACK or similar sparse direct solver
        let solver = umfpack::UmfpackSolver::new(&csr)
            .map_err(|e| format!("Solver initialization failed: {}", e))?;
        self.solution = solver.solve(&self.rhs)
            .map_err(|e| format!("Linear solve failed: {}", e))?;
        Ok(())
    }

    fn interpolate_temperature(&self, elem: &Element, temps: &[f64], xi: f64, eta: f64) -> f64 {
        let (N, _) = self.mesh.shape_functions_p2(elem, xi, eta);
        elem.nodes.iter().zip(N.iter())
            .map(|(&node_id, &Ni)| temps[node_id] * Ni)
            .sum()
    }
}
```

### 3. Annealing Visualization Component (React + Three.js)

The frontend 3D viewer that renders axisymmetric glass geometry with temperature/stress contours using Three.js WebGL.

```typescript
// frontend/src/components/Visualization/AnnealingViewer.tsx

import { useRef, useEffect, useState } from 'react';
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';
import { useSimulationStore } from '../../stores/simulationStore';

interface AnnealingViewerProps {
  resultData: {
    mesh: {
      nodes: Array<[number, number]>;  // Axisymmetric (r, z)
      elements: Array<[number, number, number]>;  // Triangle node indices
    };
    temperature: number[];  // Temperature at each node (K)
    stress: number[];  // Residual stress at each node (MPa)
  };
  fieldType: 'temperature' | 'stress';
}

export function AnnealingViewer({ resultData, fieldType }: AnnealingViewerProps) {
  const canvasRef = useRef<HTMLCanvasElement>(null);
  const sceneRef = useRef<THREE.Scene | null>(null);
  const rendererRef = useRef<THREE.WebGLRenderer | null>(null);
  const meshRef = useRef<THREE.Mesh | null>(null);
  const [colorScale, setColorScale] = useState<'viridis' | 'coolwarm'>('viridis');

  useEffect(() => {
    if (!canvasRef.current) return;

    // Initialize Three.js scene
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x1a1a1a);
    sceneRef.current = scene;

    const camera = new THREE.PerspectiveCamera(
      45,
      canvasRef.current.clientWidth / canvasRef.current.clientHeight,
      0.1,
      1000
    );
    camera.position.set(50, 50, 100);

    const renderer = new THREE.WebGLRenderer({ canvas: canvasRef.current, antialias: true });
    renderer.setSize(canvasRef.current.clientWidth, canvasRef.current.clientHeight);
    rendererRef.current = renderer;

    const controls = new OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true;

    // Lighting
    const ambientLight = new THREE.AmbientLight(0xffffff, 0.6);
    scene.add(ambientLight);
    const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8);
    directionalLight.position.set(10, 20, 10);
    scene.add(directionalLight);

    // Animation loop
    function animate() {
      requestAnimationFrame(animate);
      controls.update();
      renderer.render(scene, camera);
    }
    animate();

    return () => {
      renderer.dispose();
      controls.dispose();
    };
  }, []);

  // Update mesh geometry and colors when result data changes
  useEffect(() => {
    if (!sceneRef.current || !resultData) return;

    // Remove old mesh
    if (meshRef.current) {
      sceneRef.current.remove(meshRef.current);
      meshRef.current.geometry.dispose();
      (meshRef.current.material as THREE.Material).dispose();
    }

    // Create axisymmetric 3D geometry from 2D (r, z) mesh
    const geometry = create3DGeometryFromAxisymmetric(resultData.mesh, 32);  // 32 angular segments

    // Map field values to vertex colors
    const fieldValues = fieldType === 'temperature' ? resultData.temperature : resultData.stress;
    const colors = mapFieldToColors(fieldValues, fieldType, colorScale);
    geometry.setAttribute('color', new THREE.Float32BufferAttribute(colors, 3));

    const material = new THREE.MeshPhongMaterial({ vertexColors: true, side: THREE.DoubleSide });
    const mesh = new THREE.Mesh(geometry, material);
    meshRef.current = mesh;
    sceneRef.current.add(mesh);

  }, [resultData, fieldType, colorScale]);

  return (
    <div className="relative w-full h-full">
      <canvas ref={canvasRef} className="w-full h-full" />
      <div className="absolute top-4 right-4 bg-gray-800 p-3 rounded">
        <div className="text-white text-sm mb-2">Field: {fieldType}</div>
        <select
          value={colorScale}
          onChange={(e) => setColorScale(e.target.value as 'viridis' | 'coolwarm')}
          className="bg-gray-700 text-white px-2 py-1 rounded"
        >
          <option value="viridis">Viridis</option>
          <option value="coolwarm">Cool-Warm</option>
        </select>
      </div>
      <ColorBar fieldType={fieldType} colorScale={colorScale} min={Math.min(...(fieldType === 'temperature' ? resultData.temperature : resultData.stress))} max={Math.max(...(fieldType === 'temperature' ? resultData.temperature : resultData.stress))} />
    </div>
  );
}

// Create 3D geometry by revolving 2D axisymmetric mesh around z-axis
function create3DGeometryFromAxisymmetric(
  mesh2D: { nodes: Array<[number, number]>; elements: Array<[number, number, number]> },
  nTheta: number  // Angular segments
): THREE.BufferGeometry {
  const positions: number[] = [];
  const indices: number[] = [];

  // Generate 3D positions by rotating 2D (r, z) around z-axis
  for (let i = 0; i <= nTheta; i++) {
    const theta = (i / nTheta) * 2 * Math.PI;
    const cosTheta = Math.cos(theta);
    const sinTheta = Math.sin(theta);

    for (const [r, z] of mesh2D.nodes) {
      const x = r * cosTheta;
      const y = r * sinTheta;
      positions.push(x, y, z);
    }
  }

  // Generate triangle indices
  const nNodes2D = mesh2D.nodes.length;
  for (let i = 0; i < nTheta; i++) {
    for (const [n0, n1, n2] of mesh2D.elements) {
      const base = i * nNodes2D;
      const baseNext = ((i + 1) % (nTheta + 1)) * nNodes2D;

      // Two triangles per 2D element per angular segment (quad → 2 tris)
      indices.push(base + n0, base + n1, baseNext + n0);
      indices.push(baseNext + n0, base + n1, baseNext + n1);
      indices.push(base + n1, base + n2, baseNext + n1);
      indices.push(baseNext + n1, base + n2, baseNext + n2);
      indices.push(base + n2, base + n0, baseNext + n2);
      indices.push(baseNext + n2, base + n0, baseNext + n0);
    }
  }

  const geometry = new THREE.BufferGeometry();
  geometry.setAttribute('position', new THREE.Float32BufferAttribute(positions, 3));
  geometry.setIndex(indices);
  geometry.computeVertexNormals();
  return geometry;
}

// Map field values to RGB colors using colormap
function mapFieldToColors(
  fieldValues: number[],
  fieldType: 'temperature' | 'stress',
  colorScale: 'viridis' | 'coolwarm'
): number[] {
  const min = Math.min(...fieldValues);
  const max = Math.max(...fieldValues);
  const range = max - min;

  const colors: number[] = [];
  for (const value of fieldValues) {
    const t = (value - min) / range;  // Normalize to [0, 1]
    const [r, g, b] = colorScale === 'viridis' ? viridis(t) : coolwarm(t);
    colors.push(r, g, b);
  }
  return colors;
}

// Viridis colormap approximation
function viridis(t: number): [number, number, number] {
  const r = 0.282 * (1 - t)**4 + 0.242 * (1 - t)**3 + 0.753 * (1 - t)**2 + 0.995 * (1 - t) + 0.987 * t;
  const g = 0.004 * (1 - t)**4 + 0.424 * (1 - t)**2 + 0.995 * t;
  const b = 0.331 * (1 - t)**3 + 0.710 * (1 - t)**2 + 0.130 * t;
  return [r, g, b];
}

// Cool-warm diverging colormap
function coolwarm(t: number): [number, number, number] {
  if (t < 0.5) {
    const s = 2 * t;
    return [s, s, 1];  // Blue → White
  } else {
    const s = 2 * (1 - t);
    return [1, s, s];  // White → Red
  }
}

interface ColorBarProps {
  fieldType: 'temperature' | 'stress';
  colorScale: 'viridis' | 'coolwarm';
  min: number;
  max: number;
}

function ColorBar({ fieldType, colorScale, min, max }: ColorBarProps) {
  const canvasRef = useRef<HTMLCanvasElement>(null);

  useEffect(() => {
    if (!canvasRef.current) return;
    const canvas = canvasRef.current;
    const ctx = canvas.getContext('2d')!;
    const gradient = ctx.createLinearGradient(0, 0, 0, canvas.height);

    // Sample colormap at 10 points
    for (let i = 0; i <= 10; i++) {
      const t = i / 10;
      const [r, g, b] = colorScale === 'viridis' ? viridis(t) : coolwarm(t);
      gradient.addColorStop(1 - t, `rgb(${r*255},${g*255},${b*255})`);
    }

    ctx.fillStyle = gradient;
    ctx.fillRect(0, 0, canvas.width, canvas.height);
  }, [colorScale]);

  const unit = fieldType === 'temperature' ? 'K' : 'MPa';

  return (
    <div className="absolute bottom-4 right-4 bg-gray-800 p-3 rounded flex items-center gap-3">
      <canvas ref={canvasRef} width={30} height={200} className="border border-gray-600" />
      <div className="text-white text-xs">
        <div>{max.toFixed(1)} {unit}</div>
        <div className="mt-20">{((max + min) / 2).toFixed(1)} {unit}</div>
        <div className="mt-20">{min.toFixed(1)} {unit}</div>
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
cargo init glasssim-api
cd glasssim-api
cargo add axum tokio serde serde_json sqlx uuid chrono tower-http jsonwebtoken bcrypt
cargo add --features postgres,runtime-tokio-rustls,uuid,chrono,json sqlx
cargo add aws-sdk-s3 redis nalgebra-sparse
```
- `src/main.rs` — Axum app with CORS, tracing, graceful shutdown
- `src/config.rs` — Environment-based configuration (DATABASE_URL, JWT_SECRET, S3_BUCKET, REDIS_URL)
- `src/state.rs` — AppState struct (PgPool, Redis, S3Client)
- `src/error.rs` — ApiError enum with IntoResponse
- `Dockerfile` — Multi-stage build (builder + runtime)
- `docker-compose.yml` — PostgreSQL, Redis, MinIO (S3-compatible)

**Day 2: Database schema and migrations**
- `migrations/001_initial.sql` — All 9 tables: users, organizations, org_members, glass_compositions, projects, simulations, simulation_jobs, material_fit_jobs, optimization_runs, usage_records
- `src/db/mod.rs` — Database pool initialization
- `src/db/models.rs` — All SQLx structs with FromRow derives, VftParams implementation
- Run `sqlx migrate run` to apply schema
- Seed script for initial glass compositions (20 common types: soda-lime, borosilicate 3.3, fused silica, E-glass, S-glass, aluminosilicate, etc.) with VFT parameters from literature

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

**Day 5: Glass composition API**
- `src/api/handlers/glass_compositions.rs` — List compositions, get composition, search by category/name
- `src/api/handlers/glass_compositions.rs` — Upload custom composition (Advanced plan only)
- VFT viscosity calculator endpoint: POST {temperature_k, composition_id} → {viscosity_pa_s, log10_viscosity}
- Composition comparison endpoint: compare VFT curves for multiple compositions
- Unit tests: VFT viscosity computation, temperature-viscosity curve generation

### Phase 2 — Solver Core — Thermal + VFT Flow (Days 6–14)

**Day 6: Mesh generation and FEM framework**
- `solver-core/` — New Rust workspace member (shared between native and WASM)
- `solver-core/src/mesh/mod.rs` — Mesh2D struct for axisymmetric (r, z) triangular mesh
- `solver-core/src/mesh/generator.rs` — Simple structured mesh generation for rectangle, cylinder
- `solver-core/src/mesh/refine.rs` — Adaptive refinement (split triangles in high-gradient regions)
- `solver-core/src/fem/shape_functions.rs` — P1 and P2 Lagrange shape functions and gradients for triangles
- Unit tests: mesh generation, shape function values at quadrature points

**Day 7: Thermal solver — conduction only**
- `solver-core/src/fem/thermal.rs` — Steady-state heat conduction solver ∇·(k∇T) = 0
- Assembly of stiffness matrix K and RHS vector f
- Boundary conditions: Dirichlet (fixed temperature), Neumann (fixed flux), Robin (convection h(T - T_amb))
- Direct sparse solve using `nalgebra-sparse` CG solver or UMFPACK
- Tests: 1D conduction with analytical solution, 2D cylinder radial conduction

**Day 8: Transient thermal solver**
- `solver-core/src/fem/thermal_transient.rs` — Transient heat equation ρc_p ∂T/∂t = ∇·(k∇T)
- Backward Euler time integration: (M/Δt + K) T^{n+1} = M/Δt T^n + f^{n+1}
- Mass matrix M assembly
- Adaptive timestep based on maximum temperature change per step
- Tests: transient cooling of cylinder (compare to analytical solution for Bi << 1)

**Day 9: Rosseland radiation model**
- `solver-core/src/radiation/rosseland.rs` — Radiative conductivity k_rad = 16σn²T³/(3α)
- Enhanced thermal conductivity k_eff = k + k_rad(T)
- Nonlinear thermal equation (k_rad depends on T³) → Newton-Raphson iteration
- Jacobian computation: dK/dT from dk_rad/dT = 48σn²T²/(3α)
- Tests: radiation-dominated heat transfer in semitransparent slab

**Day 10: VFT viscosity and Stokes flow solver — linear case**
- `solver-core/src/viscosity/vft.rs` — VFT viscosity model implementation, log₁₀(η) formulation
- `solver-core/src/fem/stokes.rs` — Stokes flow solver for constant viscosity (prototype)
- Mixed velocity-pressure formulation with Taylor-Hood P2-P1 elements
- Saddle-point system solve using Schur complement or direct sparse solver
- Tests: Poiseuille flow in channel (analytical parabolic velocity profile)

**Day 11: Coupled thermal-Stokes solver**
- `solver-core/src/fem/coupled_thermal_stokes.rs` — Coupled solver: thermal equation → VFT viscosity → Stokes flow
- Picard iteration: solve thermal → update η(T) → solve Stokes → update convection → repeat until convergence
- Convergence criterion: ||T^{k+1} - T^k|| < tol and ||v^{k+1} - v^k|| < tol
- Tests: natural convection in cavity (Rayleigh-Bénard), glass flow in channel with temperature gradient

**Day 12: Free surface tracking (ALE)**
- `solver-core/src/fem/ale.rs` — Arbitrary Lagrangian-Eulerian mesh motion for free surface
- Mesh velocity w = (v·n)n on free surface (normal motion only)
- Mesh smoothing (Laplacian smoothing) for interior nodes
- Update mesh coordinates after each timestep
- Tests: droplet spreading under gravity, fiber drawing (simplified 1D neck-down)

**Day 13: Fiber draw solver — axisymmetric**
- `solver-core/src/applications/fiber_draw.rs` — Fiber draw simulation: preform → neck-down → fiber
- Geometry: axisymmetric (r, z) with free surface r(z)
- Coupled thermal (furnace heating) + viscous flow (extensional flow under tension)
- Surface tension boundary condition: Δp = γ/R (Young-Laplace)
- Fiber tension computation: F = ∫ σ_zz dA at fiber end
- Tests: fiber draw with known analytical solution (isothermal Newtonian fiber)

**Day 14: Annealing solver with TNM structural relaxation**
- `solver-core/src/applications/annealing.rs` — Annealing simulation: transient thermal + structural relaxation
- TNM model: ODE for fictive temperature dTf/dt = -(Tf - T)/τ(T, Tf)
- WLF relaxation time: τ(T, Tf) = τ0 exp(x A/(Tf - T0) + (1 - x) A/(T - T0))
- Thermal stress: σ = E/(1 - ν) α (T - Tf), integrate over cooling path
- Output: residual stress field at room temperature
- Tests: annealing of glass rod (compare to commercial software if available)

### Phase 3 — WASM Build + Frontend Visualization (Days 15–21)

**Day 15: WASM solver compilation**
- `solver-wasm/` — New workspace member for WASM target
- `solver-wasm/Cargo.toml` — wasm-bindgen, serde-wasm-bindgen dependencies
- `solver-wasm/src/lib.rs` — WASM entry points: `solve_annealing_1d()`, `solve_annealing_2d()`, `solve_steady_thermal()`
- `wasm-pack build --target web --release` pipeline
- `wasm-opt -Oz` for size optimization (target <3MB gzipped)
- JavaScript wrapper: `GlassSimSolver` class that loads WASM and provides async API

**Day 16: Frontend scaffold and Three.js setup**
```bash
npm create vite@latest frontend -- --template react-ts
cd frontend
npm i zustand @tanstack/react-query axios three @types/three
```
- `src/App.tsx` — Router, auth context, layout
- `src/stores/projectStore.ts` — Zustand store for project state
- `src/stores/simulationStore.ts` — Simulation state (parameters, results, visualization settings)
- `src/components/Visualization/Scene3D.tsx` — Three.js scene with OrbitControls, lighting
- `src/lib/wasmLoader.ts` — WASM bundle loading utility

**Day 17: 3D axisymmetric geometry viewer**
- `src/components/Visualization/AxisymmetricViewer.tsx` — Three.js renderer for axisymmetric meshes
- `src/lib/geometryUtils.ts` — create3DGeometryFromAxisymmetric() function (revolve 2D mesh)
- Color mapping for temperature/stress fields using vertex colors
- Colormaps: viridis, coolwarm, plasma (implemented as JS functions)
- Pan/zoom/rotate with OrbitControls
- Axis labels and grid

**Day 18: Temperature field visualization**
- `src/components/Visualization/TemperatureViewer.tsx` — Temperature contour rendering
- Color bar with min/max values and customizable colormap
- Isotherm lines (contour lines at fixed temperature intervals)
- Export view as PNG (canvas.toBlob())
- Animation support for transient results (time slider)

**Day 19: Stress field visualization**
- `src/components/Visualization/StressViewer.tsx` — Residual stress contour rendering
- Von Mises stress computation from stress tensor components
- Principal stress directions (vector field overlay)
- Export stress data as CSV for post-processing

**Day 20: 2D plotting — temperature/viscosity profiles**
- `src/components/Plotting/ProfilePlot.tsx` — Line plots using Canvas API + D3 scales
- Radial temperature profile T(r) at fixed z
- Axial temperature profile T(z) at fixed r
- Viscosity profile log₁₀(η) vs. position
- Export plot as SVG or PNG

**Day 21: Process parameter sensitivity plots**
- `src/components/Plotting/SensitivityPlot.tsx` — Multi-line plots for parameter sweeps
- X-axis: swept parameter (e.g., furnace temperature)
- Y-axis: objective (e.g., max residual stress, fiber diameter)
- Interactive legend to toggle series visibility
- Export data as CSV

### Phase 4 — API + Job Orchestration (Days 22–28)

**Day 22: Simulation API endpoints**
- `src/api/handlers/simulation.rs` — Create simulation, get simulation, list simulations, cancel simulation
- DOF count estimation from geometry JSON
- WASM/server routing logic based on DOF count and simulation type
- Plan-based limits enforcement (free: 1K DOF, pro: 10K DOF, advanced: unlimited)

**Day 23: Server-side simulation worker**
- `src/workers/simulation_worker.rs` — Redis job consumer, runs native solver
- `src/workers/mod.rs` — Worker pool management (configurable concurrency)
- Mesh generation on server using Triangle or gmsh integration
- Solver execution with progress monitoring
- S3 result upload: mesh, temperature field, stress field, convergence history

**Day 24: WebSocket for live simulation progress**
- `src/api/ws/mod.rs` — Axum WebSocket upgrade handler
- `src/api/ws/simulation_progress.rs` — Subscribe to simulation progress channel
- Client receives: `{ progress_pct, current_iteration, total_iterations, residual_norm, solve_time_ms }`
- Connection management: heartbeat ping/pong, automatic cleanup on disconnect
- Frontend: `src/hooks/useSimulationProgress.ts` — React hook for WebSocket subscription

**Day 25: Material property fitting API (Python FastAPI)**
- `material-service/` — Python FastAPI service for material property fitting
- `material-service/fit_vft.py` — Fit VFT parameters from user-uploaded viscosity vs. temperature data
- Nonlinear least-squares fitting using scipy.optimize.curve_fit
- Input: CSV with columns [temperature_k, viscosity_pa_s]
- Output: fitted {eta_inf, A, T0} with R² goodness-of-fit
- `/api/material/fit-vft` endpoint called from Rust API

**Day 26: Process optimization API (Bayesian optimization)**
- `material-service/optimize.py` — Bayesian optimization for process parameter tuning
- Uses scikit-optimize (skopt) for Gaussian Process regression
- Objective: minimize residual stress, target fiber diameter, etc.
- Parameters: furnace temperature, draw speed, cooling rate
- Rust API endpoint: `POST /api/projects/{id}/optimize` → creates optimization_runs record
- Python worker polls database, runs optimization loop, updates database with best params

**Day 27: Mesh and result download**
- `src/api/handlers/results.rs` — Download mesh as VTK or OBJ, download field data as VTK or CSV
- Presigned S3 URLs for large result files (>10MB)
- Result caching: hot results in Redis (TTL 1 hour), cold results in S3
- Result summary endpoint: quick statistics without downloading full field data

**Day 28: Project templates**
- `src/api/handlers/templates.rs` — List templates, create project from template
- Templates: optical fiber draw (standard SMF-28), glass rod annealing, plate annealing, tube annealing
- Each template has pre-defined geometry, glass composition, boundary conditions
- Template gallery page with preview thumbnails

### Phase 5 — Process-Specific UIs (Days 29–33)

**Day 29: Fiber draw parameter UI**
- `frontend/src/pages/FiberDrawSetup.tsx` — Input form for fiber draw parameters
- Inputs: preform diameter, target fiber diameter, furnace temperature, draw speed, furnace length
- Real-time validation: draw ratio = (d_preform / d_fiber)² must match feed rate / draw speed
- Preset configurations: standard SMF (125 μm), MMF (50 μm, 62.5 μm), specialty fiber
- Preview: sketch of preform-to-fiber geometry

**Day 30: Annealing schedule UI**
- `frontend/src/pages/AnnealingSetup.tsx` — Input form for annealing parameters
- Geometry: cylinder, plate, sphere with dimensions
- Cooling schedule: piecewise segments with duration and cooling rate
- Standard schedules: fast cool (production), slow cool (low stress), two-stage (coarse + fine annealing)
- Schedule visualization: temperature vs. time plot

**Day 31: Glass composition selector**
- `frontend/src/components/GlassCompositionSelector.tsx` — Searchable dropdown for glass compositions
- Filter by category: soda-lime, borosilicate, fused silica, etc.
- Display key properties: Tg, softening point, annealing point, VFT parameters
- VFT curve preview: log₁₀(η) vs. temperature plot
- Custom composition upload (Advanced plan): CSV with oxide composition and measured viscosity data

**Day 32: Simulation dashboard**
- `frontend/src/pages/SimulationDashboard.tsx` — Overview of simulation runs for a project
- Table: simulation type, status, DOF count, wall time, created date
- Filters: by status (completed, running, failed), by type
- Quick actions: view results, re-run with different parameters, delete
- Bulk operations: export multiple results, compare results side-by-side

**Day 33: Result comparison tool**
- `frontend/src/pages/ResultComparison.tsx` — Side-by-side comparison of 2-4 simulation results
- Synchronized 3D views with linked camera positions
- Difference plot: ΔT or Δσ between simulations
- Parameter table: show differences in input parameters
- Export comparison report as PDF

### Phase 6 — Billing + Plan Enforcement (Days 34–37)

**Day 34: Stripe integration**
- `src/api/handlers/billing.rs` — Create checkout session, create customer portal session, get subscription status
- `src/api/handlers/webhooks/stripe.rs` — Handle subscription events: `checkout.session.completed`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.payment_failed`
- Plan mapping:
  - Free (Student): 1K DOF, 1D/2D annealing only, 3 projects
  - Pro ($149/mo): 10K DOF, 2D fiber draw, radiative heat transfer, 15 projects, report export
  - Advanced ($349/mo): Unlimited DOF, process optimization, API access, custom compositions, unlimited projects

**Day 35: Usage tracking and plan limits**
- `src/middleware/plan_limits.rs` — Middleware checking plan limits before simulation execution
- `src/services/usage.rs` — Track DOF-hours per billing period (DOF count × wall time hours)
- Usage record insertion after each server-side simulation completes
- Usage dashboard: `src/api/handlers/usage.rs` — Current period usage, historical usage
- Approaching-limit warnings at 80% and 100% of DOF-hours quota

**Day 36: Billing UI**
- `frontend/src/pages/Billing.tsx` — Plan comparison table, current plan display, usage meter
- `frontend/src/components/billing/PlanCard.tsx` — Plan feature comparison cards
- `frontend/src/components/billing/UsageMeter.tsx` — DOF-hours usage bar with threshold indicators
- Upgrade/downgrade flow via Stripe Customer Portal
- Upgrade prompt modals when hitting plan limits (e.g., "Simulation requires 15K DOF. Upgrade to Advanced for unlimited.")

**Day 37: Feature gating**
- Gate process optimization behind Advanced plan
- Gate custom composition upload behind Advanced plan
- Gate API access behind Advanced plan
- Gate report generation (PDF export) behind Pro plan
- Show locked feature indicators with upgrade CTAs in UI
- Admin endpoint: override plan for specific users (beta testers, partners)

### Phase 7 — Validation + Testing (Days 38–40)

**Day 38: Solver validation — thermal problems**
- Benchmark 1: 1D radial conduction in cylinder — compare to analytical solution r²T''  + rT' = 0
- Benchmark 2: Transient cooling of sphere — compare to analytical Fourier series solution
- Benchmark 3: Radiation-dominated cooling — compare to Stefan's law for optically thick slab
- Automated test suite: `solver-core/tests/benchmarks.rs` with assertions within 1% tolerance

**Day 39: Solver validation — glass processes**
- Benchmark 4: Fiber draw isothermal — compare to analytical solution for Newtonian extensional flow
- Benchmark 5: Annealing stress prediction — compare to published data from Narayanaswamy 1971 paper
- Benchmark 6: VFT viscosity span — verify numerical stability for η ranging 10¹ to 10¹³ Pa·s
- Performance benchmarks: time vs. DOF count, convergence rate vs. viscosity ratio

**Day 40: Integration and performance testing**
- End-to-end test: create project → set parameters → run simulation → view results → export
- API integration tests: auth → project CRUD → simulation → results
- WASM solver test: load in headless browser (Playwright), run 1D annealing, verify results match native solver
- WebSocket test: connect → subscribe → receive progress → receive completion
- Concurrent simulation test: submit 10 simultaneous server-side jobs
- Performance: WASM 5K DOF annealing <10s, server 50K DOF fiber draw <5min

### Phase 8 — Deployment + Polish + Launch (Days 41–42)

**Day 41: Docker and Kubernetes configuration**
- `Dockerfile` — Multi-stage Rust build (cargo-chef for layer caching)
- `docker-compose.yml` — Full local dev stack (API, PostgreSQL, Redis, MinIO, material-service, frontend)
- `k8s/` — Kubernetes manifests:
  - `api-deployment.yaml` — API server (3 replicas, HPA)
  - `worker-deployment.yaml` — Simulation workers (auto-scaling based on queue depth)
  - `material-service-deployment.yaml` — Python FastAPI service
  - `postgres-statefulset.yaml` — PostgreSQL with PVC
  - `redis-deployment.yaml` — Redis for job queue and pub/sub
  - `ingress.yaml` — NGINX ingress with TLS
- Health check endpoints: `/health/live`, `/health/ready`
- Monitoring: Prometheus metrics (simulation duration histogram, solver convergence rate, API latency)
- Grafana dashboards: system health, simulation throughput, user activity

**Day 42: Launch preparation and final testing**
- End-to-end smoke test in production environment
- Database backup and restore verification
- Rate limiting: 100 req/min per user, 5 simulations/min per user
- Security audit: JWT validation, SQL injection (SQLx prevents this), CORS configuration, CSP headers
- Landing page with product overview, pricing, glass process animations
- Documentation: getting started guide, fiber draw tutorial, annealing tutorial, glass composition guide
- Deploy to production, enable monitoring alerts, announce launch

---

## Critical Files

```
glasssim/
├── solver-core/                           # Shared solver library (native + WASM)
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── mesh/
│   │   │   ├── mod.rs                     # Mesh2D struct for axisymmetric meshes
│   │   │   ├── generator.rs               # Structured mesh generation
│   │   │   └── refine.rs                  # Adaptive refinement
│   │   ├── fem/
│   │   │   ├── mod.rs
│   │   │   ├── shape_functions.rs         # P1/P2 Lagrange shape functions
│   │   │   ├── thermal.rs                 # Steady-state heat conduction
│   │   │   ├── thermal_transient.rs       # Transient heat equation
│   │   │   ├── stokes.rs                  # Stokes viscous flow solver
│   │   │   ├── coupled_thermal_stokes.rs  # Coupled thermal-flow solver
│   │   │   └── ale.rs                     # Arbitrary Lagrangian-Eulerian free surface
│   │   ├── viscosity/
│   │   │   ├── mod.rs
│   │   │   └── vft.rs                     # VFT viscosity model
│   │   ├── radiation/
│   │   │   ├── mod.rs
│   │   │   └── rosseland.rs               # Rosseland diffusion approximation
│   │   ├── relaxation/
│   │   │   ├── mod.rs
│   │   │   └── tnm.rs                     # Tool-Narayanaswamy-Moynihan structural relaxation
│   │   ├── applications/
│   │   │   ├── mod.rs
│   │   │   ├── fiber_draw.rs              # Optical fiber draw simulation
│   │   │   └── annealing.rs               # Glass annealing with residual stress
│   │   └── material/
│   │       ├── mod.rs
│   │       └── glass.rs                   # Glass material properties (VFT, thermal, mechanical)
│   └── tests/
│       ├── benchmarks.rs                  # Solver validation benchmarks
│       ├── thermal.rs                     # Thermal solver tests
│       └── stokes.rs                      # Stokes flow tests
│
├── solver-wasm/                           # WASM compilation target
│   ├── Cargo.toml
│   └── src/
│       └── lib.rs                         # WASM entry points (wasm-bindgen)
│
├── glasssim-api/                          # Rust API server
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
│   │   │   │   ├── glass_compositions.rs  # Glass composition CRUD, search
│   │   │   │   ├── templates.rs           # Project templates
│   │   │   │   ├── results.rs             # Result download, mesh export
│   │   │   │   ├── billing.rs             # Stripe checkout/portal
│   │   │   │   ├── usage.rs               # Usage tracking
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
│   │   │   └── simulationStore.ts         # Simulation state (params, results)
│   │   ├── hooks/
│   │   │   ├── useSimulationProgress.ts   # WebSocket hook for live progress
│   │   │   └── useWasmSolver.ts           # WASM solver hook (load + run)
│   │   ├── lib/
│   │   │   ├── api.ts                     # Axios API client
│   │   │   ├── wasmLoader.ts              # WASM bundle loading
│   │   │   └── geometryUtils.ts           # Axisymmetric 3D geometry generation
│   │   ├── pages/
│   │   │   ├── Dashboard.tsx              # Project list
│   │   │   ├── ProjectEditor.tsx          # Project setup (geometry, glass, params)
│   │   │   ├── FiberDrawSetup.tsx         # Fiber draw parameter input
│   │   │   ├── AnnealingSetup.tsx         # Annealing parameter input
│   │   │   ├── SimulationDashboard.tsx    # Simulation runs list
│   │   │   ├── ResultViewer.tsx           # 3D result visualization
│   │   │   ├── ResultComparison.tsx       # Side-by-side result comparison
│   │   │   ├── Billing.tsx                # Plan management
│   │   │   ├── Login.tsx                  # Auth pages
│   │   │   └── Register.tsx
│   │   ├── components/
│   │   │   ├── Visualization/
│   │   │   │   ├── Scene3D.tsx            # Three.js scene setup
│   │   │   │   ├── AxisymmetricViewer.tsx # 3D viewer for axisymmetric meshes
│   │   │   │   ├── TemperatureViewer.tsx  # Temperature field rendering
│   │   │   │   └── StressViewer.tsx       # Stress field rendering
│   │   │   ├── Plotting/
│   │   │   │   ├── ProfilePlot.tsx        # 1D profile plots (T(r), η(z))
│   │   │   │   └── SensitivityPlot.tsx    # Parameter sweep plots
│   │   │   ├── GlassCompositionSelector.tsx  # Glass composition dropdown
│   │   │   ├── billing/
│   │   │   │   ├── PlanCard.tsx
│   │   │   │   └── UsageMeter.tsx
│   │   │   └── common/
│   │   │       └── SimulationToolbar.tsx  # Run/stop/export controls
│   │   └── data/
│   │       └── templates/                 # Project templates (JSON geometry)
│   └── public/
│       └── wasm/                          # WASM solver bundle (loaded at runtime)
│
├── material-service/                      # Python FastAPI (material fitting, optimization)
│   ├── requirements.txt
│   ├── main.py                            # FastAPI app
│   ├── fit_vft.py                         # VFT parameter fitting from viscosity data
│   ├── optimize.py                        # Bayesian optimization for process parameters
│   └── Dockerfile
│
├── k8s/                                   # Kubernetes manifests
│   ├── api-deployment.yaml
│   ├── worker-deployment.yaml
│   ├── material-service-deployment.yaml
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

### Benchmark 1: 1D Radial Conduction in Cylinder (Steady-State Thermal)

**Problem:** Hollow cylinder, inner radius r_i = 10 mm at T_i = 500 K, outer radius r_o = 50 mm at T_o = 300 K, thermal conductivity k = 1.0 W/(m·K).

**Analytical solution:** T(r) = T_i + (T_o - T_i) × ln(r/r_i) / ln(r_o/r_i)

**Expected at r = 30 mm:** T(30mm) = 500 + (300 - 500) × ln(30/10) / ln(50/10) = **431.4 K**

**Tolerance:** < 0.5% (FEM discretization error with 50-element mesh)

### Benchmark 2: Transient Cooling of Glass Sphere (Transient Thermal)

**Problem:** Sphere radius R = 25 mm, initially at T_0 = 800 K, cooled in air at T_amb = 300 K with convection h = 10 W/(m²·K), thermal diffusivity α = 5e-7 m²/s.

**Biot number:** Bi = hR/k = 10 × 0.025 / 1.0 = 0.25 (small, nearly lumped capacitance)

**Lumped capacitance approximation:** T(t) ≈ T_amb + (T_0 - T_amb) × exp(-t/τ) where τ = ρc_pV/(hA) ≈ ρc_pR/(3h)

**Expected at t = 500 s:** T_center ≈ 300 + 500 × exp(-500 × 3 × 10 / (2500 × 1000 × 0.025)) ≈ **540 K**

**Tolerance:** < 5% (lumped capacitance is approximate for Bi=0.25)

### Benchmark 3: Fiber Draw Isothermal Newtonian (Stokes Flow + Free Surface)

**Problem:** Isothermal fiber draw, T = 1400 K constant, η = 10^4 Pa·s, preform diameter d_p = 10 mm, fiber diameter d_f = 125 μm, draw speed v_f = 1 m/s.

**Mass conservation:** v_p A_p = v_f A_f → v_p = v_f (d_f / d_p)² = 1 × (0.125/10)² = **1.5625e-4 m/s**

**Fiber tension** (extensional viscosity approximation): F ≈ 3η v_f (d_p²/d_f²) / L_neck where L_neck ≈ 50 mm

**Expected tension:** F ≈ 3 × 10^4 × 1 × (0.01² / 0.000125²) / 0.05 ≈ **3.84 N**

**Tolerance:** < 10% (extensional viscosity approximation vs. full Stokes solution)

### Benchmark 4: Annealing Residual Stress (TNM Structural Relaxation)

**Problem:** Soda-lime glass cylinder, radius R = 25 mm, cooled from 600 K (above Tg ≈ 820 K) to 300 K at constant cooling rate -1 K/s. VFT: η_inf = 10^(-3.5) Pa·s, A = 4500 K, T_0 = 520 K. TNM: τ_0 = 1e-12 s, x = 0.6.

**Expected:** Residual stress peak near surface due to thermal gradient during cooling through Tg.

**Narayanaswamy 1971 data:** For similar cooling rate, max surface stress σ_max ≈ **15-25 MPa** (tensile).

**Tolerance:** < 20% (TNM parameters vary by composition, comparison to literature trend)

### Benchmark 5: VFT Viscosity Numerical Stability (Extreme Range)

**Problem:** Soda-lime glass VFT parameters: η_inf = 10^(-3.5) Pa·s, A = 4500 K, T_0 = 520 K. Compute viscosity over temperature range 700-1500 K.

**Expected viscosity span:** η(700K) ≈ **10^13 Pa·s**, η(1500K) ≈ **10^2 Pa·s** (11 orders of magnitude)

**Expected numerical stability:** Solver converges within 20 iterations for Stokes flow with spatially varying temperature field spanning this range.

**Tolerance:** Convergence achieved, solution physically reasonable (no negative pressures, velocity magnitude consistent with boundary conditions)

---

## Verification

### Integration Testing Checklist

1. **Auth flow** — Register → login → JWT issued → authorized API calls → token refresh → logout
2. **Project CRUD** — Create → update geometry → save → reload → verify geometry preserved
3. **Glass composition** — Search "soda-lime" → select → use in project → VFT curve displayed
4. **WASM simulation** — 1D annealing (100 nodes) → WASM solver → results returned → temperature profile displayed
5. **Server simulation** — 2D fiber draw (10K nodes) → job queued → worker picks up → WebSocket progress → results in S3
6. **Field visualization** — Load temperature field → Three.js renders → rotate/zoom → colormap change → export PNG
7. **Stress visualization** — Load stress field → contours displayed → principal stress vectors → export CSV
8. **Parameter sweep** — Define furnace temp sweep (1900-2100 K, 5 steps) → queue 5 simulations → compare fiber diameters
9. **Template usage** — Select "SMF-28 Fiber Draw" template → project created → simulate → verify diameter ≈ 125 μm
10. **Plan limits** — Free user → 2K DOF simulation → blocked with upgrade prompt
11. **Billing** — Subscribe to Pro → Stripe checkout → webhook → plan updated → 10K DOF limit lifted
12. **Concurrent users** — 10 users simultaneously running server simulations → all complete correctly
13. **Material fitting** — Upload viscosity CSV → fit VFT → verify R² > 0.99
14. **Optimization** — Run Bayesian optimization (10 iterations) → optimal parameters returned → re-simulate with best params
15. **Error handling** — Non-convergent simulation → meaningful error message with convergence plot → no crash

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

-- 3. User plan distribution and conversion
SELECT plan, COUNT(*) as users,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 1) as pct
FROM users GROUP BY plan ORDER BY users DESC;

-- 4. DOF-hours usage by user (billing period)
SELECT u.email, u.plan,
    SUM(ur.quantity) as total_dof_hours,
    CASE u.plan
        WHEN 'free' THEN 10
        WHEN 'pro' THEN 1000
        WHEN 'advanced' THEN 999999
    END as limit_dof_hours
FROM usage_records ur
JOIN users u ON ur.user_id = u.id
WHERE ur.period_start <= CURRENT_DATE AND ur.period_end >= CURRENT_DATE
    AND ur.record_type = 'dof_hours'
GROUP BY u.email, u.plan
ORDER BY total_dof_hours DESC;

-- 5. Glass composition popularity
SELECT gc.name, gc.category,
    COUNT(DISTINCT p.id) as projects_using
FROM glass_compositions gc
LEFT JOIN projects p ON p.glass_composition_id = gc.id
GROUP BY gc.id
ORDER BY projects_using DESC
LIMIT 20;
```

### Performance Benchmarks

| Operation | Target | Method |
|-----------|--------|--------|
| WASM 1D annealing (100 nodes) | <1s | Browser benchmark (Chrome DevTools) |
| WASM 2D annealing (5K DOF) | <10s | Browser benchmark |
| Server 2D fiber draw (10K DOF) | <30s | Server timing, 4 cores |
| Server 2D coupled (50K DOF) | <5min | Server timing, 8 cores |
| Three.js viewer (100K triangles) | 60 FPS | Chrome FPS counter during rotation |
| Mesh generation 2D (1K nodes) | <2s | Server-side Triangle integration |
| API: create simulation | <200ms | p95 latency (Prometheus) |
| API: search glass compositions | <100ms | p95 latency with 100+ compositions |
| WASM bundle load (cached) | <500ms | Service worker cache hit |
| WASM bundle load (cold) | <4s | CDN delivery, 3MB gzipped |
| WebSocket: progress latency | <100ms | Time from worker emit to client receive |

---

## Deployment Architecture

```
CloudFront CDN → [Frontend, WASM Bundle, S3 Results]
         ↓
    AWS ALB (HTTPS)
         ↓
  ┌──────┴──────┬──────────┬─────────────┐
  ↓             ↓          ↓             ↓
API Pod ×3   Worker ×N   Material    PostgreSQL
                         Service
  ↓             ↓          ↓
Redis        S3 Bucket  ElastiCache
```

**Scaling:**
- API: HPA based on CPU (target 70%)
- Workers: HPA based on Redis queue depth (1 worker per 3 queued jobs)
- PostgreSQL: RDS with read replicas for analytics queries
- Redis: ElastiCache cluster mode for high availability

**Monitoring:**
- Prometheus: solver convergence rate (iterations to converge), API latency p50/p95/p99, simulation queue depth
- Grafana: simulation throughput (sims/hour), user activity (DAU/MAU), solver performance (DOF vs. wall time scatter plot)
- Sentry: frontend errors (WASM loading failures, Three.js rendering crashes), backend errors (solver divergence, S3 upload failures)

**Alerts:**
- API p95 latency > 1s for 5 minutes
- Simulation worker queue depth > 50 for 10 minutes
- Database connection pool exhaustion
- S3 upload failure rate > 1%
- WASM solver failure rate > 5%

---

## Post-MVP Roadmap

### v1.1 — 3D Container Glass Forming (Q2)
- Blow-blow and press-blow process simulation
- Full 3D tetrahedral mesh solver
- Multi-stage forming sequence (gob → parison → final blow)
- Wall thickness distribution prediction
- Mold contact heat transfer modeling

### v1.2 — Float Glass and Tin Bath (Q2)
- Molten tin bath simulation with free surface
- Glass ribbon spreading dynamics under gravity
- Top heater control zone optimization
- Annealing lehr integration (continuous cooling)
- Ribbon width control via edge pull

### v1.3 — Advanced Radiation Models (Q3)
- Discrete ordinates (DOM) solver for directional radiation
- P1 spherical harmonics approximation
- Wavelength-dependent spectral resolution (8 bands: UV, visible, near-IR, far-IR)
- View factor calculation for enclosure radiation (molds, heaters, furnace walls)
- Participating media radiation for furnace atmosphere

### v1.4 — Tempering and Chemical Strengthening (Q3)
- Rapid quench simulation (air jet modeling with CFD-coupled cooling)
- Parabolic stress profile validation for tempered glass
- Ion exchange diffusion (Na+/K+) for chemical strengthening
- Fragmentation pattern prediction from stress field
- Automotive windshield compliance testing (ECE R43)

### v1.5 — AI-Driven Optimization (Q4)
- Gaussian process surrogate models for expensive simulations
- Bayesian optimization for process tuning (10-100× faster than grid search)
- Multi-objective Pareto optimization (minimize stress AND maximize throughput)
- Reinforcement learning for adaptive control strategies
- Transfer learning from one glass composition to another

### v1.6 — Cloud HPC Integration (Q4)
- AWS Batch for large-scale parameter sweeps (1000+ simulations)
- GPU-accelerated 3D flow solver (CUDA) for 1M+ DOF simulations
- Parallel mesh refinement with distributed memory
- Real-time simulation on 100K+ nodes with iterative solver preconditioning
- Cost optimization (spot instances for batch jobs)

### v2.0 — Enterprise Features (2026)
- On-premise deployment option (Kubernetes Helm chart)
- Multi-user collaboration (real-time co-editing of geometry and parameters)
- Custom glass composition characterization service (send us samples, we measure VFT/TNM params)
- Integration with plant control systems (OPC-UA for real-time furnace data)
- Multi-furnace production line simulation (optimize entire factory)
- Predictive maintenance (ML models for furnace failure prediction from simulation)