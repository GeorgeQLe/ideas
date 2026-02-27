# 62. ProChem — Chemical Process Simulation and Design Platform

## Implementation Plan

**MVP Scope:** Browser-based flowsheet editor with drag-and-drop unit operation placement (mixer, splitter, heater, flash drum, pump, compressor, heat exchanger, CSTR, PFR, shortcut distillation, rigorous distillation columns) and automatic stream routing rendered via SVG/Canvas, custom steady-state process simulation engine implementing equation-oriented (EO) simultaneous solution with sparse Newton-Raphson solver compiled to Rust for server-side execution, Peng-Robinson (PR) and Soave-Redlich-Kwong (SRK) equations of state plus NRTL activity coefficient model for thermodynamic property calculations with WebAssembly flash routines for client-side property lookups, support for material and energy balances with automatic recycle loop convergence using Wegstein and Broyden methods, thermodynamic database with 2,000 compounds (DIPPR correlations for pure components, binary interaction parameters for vapor-liquid equilibrium), interactive process flow diagram (PFD) viewer with stream tables rendered via React/D3.js, Stripe billing with three tiers (Free / Pro $149/mo / Advanced $349/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| Simulation Engine | Rust (native) | Equation-oriented solver with sparse NR, sequential modular fallback |
| Thermodynamics | Rust + WASM | PR/SRK EOS, NRTL activity coefficients, flash algorithms compiled to WASM |
| Property Lookups | WASM (wasm-pack) | Client-side flash calculations, pure component properties for UI |
| Optimization Service | Python 3.12 (FastAPI) | SciPy optimizers for recycle convergence, parametric studies |
| Database | PostgreSQL 16 | Projects, users, simulations, compound database, BIP data |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | Simulation results (flowsheet state, convergence history), custom compound data |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| Flowsheet Editor | SVG + Canvas | Konva.js for flowsheet rendering, auto-routing for streams |
| Visualization | D3.js + SVG | Stream tables, property plots, convergence monitoring |
| Real-time | WebSocket (Axum) | Live simulation progress, convergence monitoring |
| Job Queue | Redis 7 + Tokio tasks | Simulation job management, parametric study distribution |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | WASM bundle delivery, static frontend assets |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, error tracking |
| CI/CD | GitHub Actions | Rust tests, WASM build, Docker image push |

### Key Architecture Decisions

1. **Equation-oriented (EO) solver with sparse Newton-Raphson for rigorous simulation**: Unlike legacy sequential modular simulators (tear streams, iterative convergence), ProChem uses a modern equation-oriented approach where all unit operation equations are assembled into a single large sparse system and solved simultaneously. This provides better convergence for highly recycle-rich flowsheets (refineries, petrochemical complexes) and allows direct calculation of derivatives for optimization. We use KLU sparse factorization (same as SpiceSim) for efficient linear algebra.

2. **WASM thermodynamics for instant client-side property previews**: Common thermodynamic calculations (flash, pure component properties, single-stage VLE) run in the browser via WASM, providing instant feedback when users change component selections or stream conditions. This covers 80% of interactive property lookups without server round-trips. Full flowsheet simulations run server-side with the native Rust engine for performance and resource control.

3. **Hybrid PR/SRK EOS + NRTL activity coefficient models**: For vapor-liquid equilibrium (VLE), ProChem supports both cubic equations of state (Peng-Robinson, Soave-Redlich-Kwong) for hydrocarbon/petrochemical systems and activity coefficient models (NRTL) for polar/non-ideal mixtures. The thermodynamic property package automatically selects the appropriate flash algorithm (two-phase PT flash, three-phase VLLE flash, bubble/dew point) based on system properties and user-specified thermo method.

4. **SVG flowsheet rendering with Canvas fallback for large flowsheets**: Unit operations and stream routing are rendered via SVG (crisp at any zoom, DOM-based interaction) using Konva.js for drag-and-drop. For flowsheets exceeding 200 units, we switch to Canvas rendering with spatial indexing (R-tree) for performance. Stream auto-routing uses Manhattan routing with obstacle avoidance similar to EDA tools.

5. **S3 for simulation results with PostgreSQL metadata**: Converged flowsheet solutions (full stream tables, equipment parameters, convergence history) are stored as JSON in S3, while PostgreSQL holds metadata (project, user, status, key metrics). This allows the result database to scale to millions of simulations without bloating PostgreSQL while enabling fast parametric search via indexed metadata.

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";  -- For fuzzy search on compounds

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

-- Organizations (for team collaboration)
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

-- Projects (process flowsheet workspace)
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    flowsheet_data JSONB NOT NULL DEFAULT '{}',  -- Full flowsheet state (units, streams, specifications)
    settings JSONB DEFAULT '{}',  -- Thermo method, convergence tolerances, units of measure
    is_public BOOLEAN NOT NULL DEFAULT false,
    forked_from UUID REFERENCES projects(id) ON DELETE SET NULL,
    thumbnail_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);
CREATE INDEX projects_org_idx ON projects(org_id);
CREATE INDEX projects_updated_idx ON projects(updated_at DESC);

-- Simulations (flowsheet solution runs)
CREATE TABLE simulations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    simulation_type TEXT NOT NULL DEFAULT 'steady_state',  -- steady_state | parametric | optimization
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | running | converged | failed | cancelled
    thermo_method TEXT NOT NULL,  -- pr_eos | srk_eos | nrtl | ideal
    convergence_method TEXT NOT NULL DEFAULT 'equation_oriented',  -- equation_oriented | sequential_modular
    results_url TEXT,  -- S3 URL for full solution data
    results_summary JSONB,  -- Quick-access: key stream properties, equipment duties, convergence info
    error_message TEXT,
    iteration_count INTEGER DEFAULT 0,
    residual_norm REAL,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    wall_time_ms INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX sims_project_idx ON simulations(project_id);
CREATE INDEX sims_user_idx ON simulations(user_id);
CREATE INDEX sims_status_idx ON simulations(status);
CREATE INDEX sims_created_idx ON simulations(created_at DESC);

-- Simulation Jobs (for server-side execution)
CREATE TABLE simulation_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    simulation_id UUID NOT NULL REFERENCES simulations(id) ON DELETE CASCADE,
    worker_id TEXT,
    priority INTEGER DEFAULT 0,  -- Higher = more urgent
    progress_pct REAL DEFAULT 0.0,
    convergence_data JSONB DEFAULT '[]',  -- [{iteration, residual, step_size}]
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX jobs_sim_idx ON simulation_jobs(simulation_id);
CREATE INDEX jobs_worker_idx ON simulation_jobs(worker_id);

-- Compound Database (pure component properties)
CREATE TABLE compounds (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    cas_number TEXT UNIQUE,  -- CAS registry number
    name TEXT NOT NULL,
    formula TEXT,
    molecular_weight REAL NOT NULL,  -- kg/kmol
    critical_temp REAL,  -- K
    critical_pressure REAL,  -- Pa
    critical_volume REAL,  -- m³/kmol
    acentric_factor REAL,  -- Pitzer acentric factor
    dippr_properties JSONB,  -- DIPPR correlations: Cp_ig, H_vap, rho_liq, mu, k, sigma
    is_verified BOOLEAN DEFAULT false,
    is_builtin BOOLEAN DEFAULT true,
    uploaded_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX compounds_cas_idx ON compounds(cas_number);
CREATE INDEX compounds_name_trgm_idx ON compounds USING gin(name gin_trgm_ops);
CREATE INDEX compounds_formula_idx ON compounds(formula);

-- Binary Interaction Parameters (for mixing rules in EOS and activity coefficients)
CREATE TABLE binary_interaction_params (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    compound1_id UUID NOT NULL REFERENCES compounds(id) ON DELETE CASCADE,
    compound2_id UUID NOT NULL REFERENCES compounds(id) ON DELETE CASCADE,
    model_type TEXT NOT NULL,  -- pr_eos | srk_eos | nrtl | uniquac | wilson
    params JSONB NOT NULL,  -- Model-specific: {kij} for EOS, {a12, a21, alpha} for NRTL
    temperature_range JSONB,  -- {min: 273, max: 500} in K
    source TEXT,  -- Literature reference or "user-fitted"
    is_verified BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(compound1_id, compound2_id, model_type)
);
CREATE INDEX bip_comp1_idx ON binary_interaction_params(compound1_id);
CREATE INDEX bip_comp2_idx ON binary_interaction_params(compound2_id);
CREATE INDEX bip_model_idx ON binary_interaction_params(model_type);

-- Saved Stream Tables (for quick access to converged stream properties)
CREATE TABLE stream_tables (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    simulation_id UUID NOT NULL REFERENCES simulations(id) ON DELETE CASCADE,
    stream_name TEXT NOT NULL,
    temperature REAL,  -- K
    pressure REAL,  -- Pa
    vapor_fraction REAL,  -- 0-1
    mass_flow REAL,  -- kg/s
    mole_flow REAL,  -- kmol/s
    composition JSONB NOT NULL,  -- {compound_id: mole_fraction}
    properties JSONB,  -- {enthalpy, entropy, density, viscosity, etc.}
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX stream_tables_sim_idx ON stream_tables(simulation_id);

-- Usage Tracking (for billing enforcement)
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,  -- simulation_runs | flowsheet_units | storage_bytes
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
    pub flowsheet_data: serde_json::Value,
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
    pub thermo_method: String,
    pub convergence_method: String,
    pub results_url: Option<String>,
    pub results_summary: Option<serde_json::Value>,
    pub error_message: Option<String>,
    pub iteration_count: i32,
    pub residual_norm: Option<f32>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub wall_time_ms: Option<i32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Compound {
    pub id: Uuid,
    pub cas_number: Option<String>,
    pub name: String,
    pub formula: Option<String>,
    pub molecular_weight: f64,
    pub critical_temp: Option<f64>,
    pub critical_pressure: Option<f64>,
    pub critical_volume: Option<f64>,
    pub acentric_factor: Option<f64>,
    pub dippr_properties: serde_json::Value,
    pub is_verified: bool,
    pub is_builtin: bool,
    pub uploaded_by: Option<Uuid>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct BinaryInteractionParam {
    pub id: Uuid,
    pub compound1_id: Uuid,
    pub compound2_id: Uuid,
    pub model_type: String,
    pub params: serde_json::Value,
    pub temperature_range: Option<serde_json::Value>,
    pub source: Option<String>,
    pub is_verified: bool,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct SimulationParams {
    pub thermo_method: ThermoMethod,
    pub convergence_method: ConvergenceMethod,
    pub tolerance: f64,  // Convergence tolerance (default 1e-6)
    pub max_iterations: usize,  // Default 100
    pub recycle_method: RecycleMethod,  // wegstein | broyden | direct_substitution
}

#[derive(Debug, Deserialize, Serialize, Clone)]
#[serde(rename_all = "snake_case")]
pub enum ThermoMethod {
    PrEos,      // Peng-Robinson equation of state
    SrkEos,     // Soave-Redlich-Kwong equation of state
    Nrtl,       // NRTL activity coefficient model
    Ideal,      // Ideal gas + Raoult's law (for simple systems)
}

#[derive(Debug, Deserialize, Serialize, Clone)]
#[serde(rename_all = "snake_case")]
pub enum ConvergenceMethod {
    EquationOriented,    // Simultaneous solution of all equations
    SequentialModular,   // Sequential unit-by-unit with tear streams
}

#[derive(Debug, Deserialize, Serialize, Clone)]
#[serde(rename_all = "snake_case")]
pub enum RecycleMethod {
    Wegstein,            // Wegstein acceleration (fastest for most cases)
    Broyden,             // Quasi-Newton (better for highly nonlinear)
    DirectSubstitution,  // Simple successive substitution (most robust)
}
```

---

## Solver Architecture Deep-Dive

### Governing Equations and Discretization

ProChem's equation-oriented (EO) solver assembles all unit operation models into a single large nonlinear system **F(x) = 0** and solves via sparse Newton-Raphson. For a flowsheet with `n_units` unit operations and `n_streams` streams, the system has:

- **Mass balances** for each component in each unit (C × n_units equations)
- **Energy balances** for each unit (n_units equations)
- **Equilibrium relations** (VLE flash, reaction equilibrium) where applicable
- **Equipment specifications** (e.g., distillation reflux ratio, heat exchanger area)
- **Stream connectivity** (mixer/splitter mass conservation)

**Total system size:** O(10 × n_streams + 5 × n_units) equations. For a 50-unit refinery, this is ~1,000 equations.

**Newton-Raphson iteration:**

```
J(x_k) · Δx = -F(x_k)
x_{k+1} = x_k + α · Δx

where J = ∂F/∂x (Jacobian), α = line search damping factor
```

The Jacobian **J** is sparse (each equation depends on only 2-5 variables). We use **Compressed Sparse Row (CSR)** format and KLU sparse direct solver (same as SpiceSim).

**Convergence criteria:**

```
||F(x)|| < abstol  (typically 1e-6 for mass/energy balances)
||Δx|| / ||x|| < reltol  (typically 1e-4 for relative change)
```

### Thermodynamic Flash Algorithm (PT Flash)

The most critical thermodynamic calculation is the **isothermal-isobaric (PT) flash**: given pressure P, temperature T, and overall composition z, find vapor fraction β and phase compositions x (liquid), y (vapor).

**Rachford-Rice equation:**

```
f(β) = Σ z_i (K_i - 1) / (1 + β(K_i - 1)) = 0

where K_i = y_i / x_i (equilibrium ratio for component i)
```

For **Peng-Robinson EOS**, the K-values are computed from fugacity equality:

```
φ_i^V(T, P, y) · y_i = φ_i^L(T, P, x) · x_i

where φ = fugacity coefficient from cubic EOS
```

**PR EOS form:**

```
P = RT/(V - b) - a·α(T) / (V² + 2bV - b²)

a = 0.45724 R²Tc² / Pc
b = 0.07780 RTc / Pc
α(T) = [1 + κ(1 - √(T/Tc))]²
κ = 0.37464 + 1.54226ω - 0.26992ω²  (ω = acentric factor)
```

For mixtures, **van der Waals mixing rules:**

```
a_mix = Σ Σ x_i x_j √(a_i a_j) (1 - k_ij)
b_mix = Σ x_i b_i

where k_ij = binary interaction parameter
```

**Flash solution algorithm:**

1. Initialize K_i = (Pc_i / P) · exp[5.37(1 + ω_i)(1 - Tc_i/T)]  (Wilson's correlation)
2. Solve Rachford-Rice for β via Newton-Raphson (bounds: 0 ≤ β ≤ 1)
3. Update phase compositions: x_i = z_i / (1 + β(K_i - 1)), y_i = K_i · x_i
4. Compute fugacities from EOS, update K_i = φ_i^L / φ_i^V
5. Repeat steps 2-4 until ||K_new - K_old|| < 1e-8

**NRTL activity coefficient model** (for polar/non-ideal systems):

```
ln γ_i = [Σ x_j τ_ji G_ji / Σ x_k G_ki] + Σ [x_j G_ij / Σ x_k G_kj] · [τ_ij - (Σ x_m τ_mj G_mj) / (Σ x_k G_kj)]

where G_ij = exp(-α_ij τ_ij), τ_ij = a_ij / RT, α_ij = non-randomness parameter
```

For NRTL flash, K_i = (γ_i · P_i^sat) / P, where P_i^sat is vapor pressure from Antoine equation.

### Recycle Convergence Acceleration

**Wegstein method** (default for recycle loops):

```
Given tear stream guess x_0, simulate flowsheet → get output x_1
Compute acceleration factor: q = (x_1 - x_0) / (x_1^prev - x_0^prev)
Accelerated update: x_new = (q·x_1 - x_0) / (q - 1)
```

Wegstein typically converges in 5-15 iterations for well-conditioned recycles. For highly nonlinear recycles (e.g., reactive distillation with large recycle ratios), we use **Broyden's method** (quasi-Newton on the recycle function).

---

## Architecture Deep-Dives

### 1. Simulation API Handler (Rust/Axum)

The primary endpoint receives a simulation request, validates the flowsheet, assembles the equation system, and runs the EO solver with live progress streaming.

```rust
// src/api/handlers/simulation.rs

use axum::{
    extract::{Path, State, Json},
    http::StatusCode,
    response::IntoResponse,
};
use uuid::Uuid;
use crate::{
    db::models::{Simulation, SimulationParams},
    solver::flowsheet::assemble_flowsheet,
    solver::eo_solver::EquationOrientedSolver,
    state::AppState,
    auth::Claims,
    error::ApiError,
};

#[derive(serde::Deserialize)]
pub struct CreateSimulationRequest {
    pub thermo_method: crate::db::models::ThermoMethod,
    pub convergence_method: crate::db::models::ConvergenceMethod,
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

    // 2. Parse flowsheet and validate (check for floating units, missing streams)
    let flowsheet = assemble_flowsheet(&project.flowsheet_data, &req.thermo_method)?;
    let unit_count = flowsheet.units.len();

    // 3. Check plan limits
    let user = sqlx::query_as!(
        crate::db::models::User,
        "SELECT * FROM users WHERE id = $1",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    if user.plan == "free" && unit_count > 10 {
        return Err(ApiError::PlanLimit(
            "Free plan supports flowsheets up to 10 unit operations. Upgrade to Pro for unlimited."
        ));
    }

    // 4. Create simulation record
    let sim = sqlx::query_as!(
        Simulation,
        r#"INSERT INTO simulations
            (project_id, user_id, simulation_type, status, thermo_method, convergence_method)
        VALUES ($1, $2, 'steady_state', 'pending', $3, $4)
        RETURNING *"#,
        project_id,
        claims.user_id,
        serde_json::to_string(&req.thermo_method)?,
        serde_json::to_string(&req.convergence_method)?,
    )
    .fetch_one(&state.db)
    .await?;

    // 5. Enqueue simulation job
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
    state.redis
        .publish("simulation:jobs", serde_json::to_string(&job.id)?)
        .await?;

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

### 2. Equation-Oriented Solver Core (Rust)

The core EO solver that builds the sparse Jacobian, solves the linear system via KLU, and performs line search damping for robust convergence.

```rust
// solver-core/src/eo_solver.rs

use nalgebra_sparse::{CsrMatrix, CooMatrix};
use crate::units::{UnitOperation, UnitOpTrait};
use crate::thermo::ThermoPackage;
use crate::klu::KluSolver;

pub struct EquationOrientedSolver {
    pub units: Vec<Box<dyn UnitOpTrait>>,
    pub streams: Vec<Stream>,
    pub thermo: Box<dyn ThermoPackage>,
    pub n_vars: usize,
    pub n_eqs: usize,
    pub x: Vec<f64>,           // Solution vector (all variables)
    pub f: Vec<f64>,           // Residual vector F(x)
    pub jacobian: CooMatrix<f64>,
}

impl EquationOrientedSolver {
    pub fn new(
        units: Vec<Box<dyn UnitOpTrait>>,
        streams: Vec<Stream>,
        thermo: Box<dyn ThermoPackage>,
    ) -> Self {
        // Count variables: each stream has T, P, F, and composition
        let n_streams = streams.len();
        let n_components = thermo.n_components();
        let n_stream_vars = n_streams * (3 + n_components);  // T, P, F, z[C]

        // Each unit has internal variables (duty, split fractions, etc.)
        let n_unit_vars: usize = units.iter().map(|u| u.n_vars()).sum();
        let n_vars = n_stream_vars + n_unit_vars;

        // Each unit contributes equations (mass/energy balances, equilibrium)
        let n_eqs: usize = units.iter().map(|u| u.n_equations()).sum();

        Self {
            units,
            streams,
            thermo,
            n_vars,
            n_eqs,
            x: vec![0.0; n_vars],
            f: vec![0.0; n_eqs],
            jacobian: CooMatrix::new(n_eqs, n_vars),
        }
    }

    /// Initialize solution vector with default values (feed streams known, others guessed)
    pub fn initialize(&mut self) -> Result<(), SolverError> {
        let n_components = self.thermo.n_components();
        for (i, stream) in self.streams.iter().enumerate() {
            let offset = i * (3 + n_components);
            self.x[offset] = stream.temperature.unwrap_or(298.15);  // K
            self.x[offset + 1] = stream.pressure.unwrap_or(101325.0);  // Pa
            self.x[offset + 2] = stream.molar_flow.unwrap_or(100.0);  // kmol/h
            for c in 0..n_components {
                self.x[offset + 3 + c] = stream.composition.get(&c).copied().unwrap_or(1.0 / n_components as f64);
            }
        }
        Ok(())
    }

    /// Main Newton-Raphson iteration loop
    pub fn solve(&mut self, tolerance: f64, max_iter: usize) -> Result<SolutionResult, SolverError> {
        let mut iteration = 0;
        let mut residual_norm = f64::INFINITY;

        while iteration < max_iter {
            // 1. Evaluate residuals F(x) and Jacobian J(x)
            self.evaluate_residuals()?;
            self.evaluate_jacobian()?;

            residual_norm = self.f.iter().map(|r| r.powi(2)).sum::<f64>().sqrt();
            if residual_norm < tolerance {
                return Ok(SolutionResult {
                    converged: true,
                    iterations: iteration,
                    residual_norm,
                    solution: self.extract_solution(),
                });
            }

            // 2. Solve linear system: J · Δx = -F
            let csr = CsrMatrix::from(&self.jacobian);
            let mut klu = KluSolver::new(&csr)?;
            let neg_f: Vec<f64> = self.f.iter().map(|&r| -r).collect();
            let delta_x = klu.solve(&neg_f)?;

            // 3. Line search to find damping factor α
            let alpha = self.line_search(&delta_x)?;

            // 4. Update solution: x += α · Δx
            for i in 0..self.n_vars {
                self.x[i] += alpha * delta_x[i];
            }

            iteration += 1;
        }

        Err(SolverError::Convergence(format!(
            "Failed to converge after {} iterations. Residual norm: {:.3e}",
            iteration, residual_norm
        )))
    }

    /// Evaluate residuals F(x) by calling each unit operation's residual function
    fn evaluate_residuals(&mut self) -> Result<(), SolverError> {
        self.f.fill(0.0);
        let mut eq_offset = 0;

        for unit in &self.units {
            let residuals = unit.compute_residuals(&self.x, &self.streams, &*self.thermo)?;
            for (i, r) in residuals.iter().enumerate() {
                self.f[eq_offset + i] = *r;
            }
            eq_offset += residuals.len();
        }

        Ok(())
    }

    /// Evaluate Jacobian J(x) via numerical finite differences
    fn evaluate_jacobian(&mut self) -> Result<(), SolverError> {
        self.jacobian = CooMatrix::new(self.n_eqs, self.n_vars);
        let epsilon = 1e-7;

        for j in 0..self.n_vars {
            let x_j = self.x[j];
            let delta = epsilon * x_j.abs().max(1.0);

            // Perturb x[j] and compute F(x + Δ)
            self.x[j] = x_j + delta;
            let mut f_plus = vec![0.0; self.n_eqs];
            let mut eq_offset = 0;
            for unit in &self.units {
                let residuals = unit.compute_residuals(&self.x, &self.streams, &*self.thermo)?;
                for (i, r) in residuals.iter().enumerate() {
                    f_plus[eq_offset + i] = *r;
                }
                eq_offset += residuals.len();
            }
            self.x[j] = x_j;

            // Compute column j of Jacobian: J[:, j] = (F(x + Δ) - F(x)) / Δ
            for i in 0..self.n_eqs {
                let deriv = (f_plus[i] - self.f[i]) / delta;
                if deriv.abs() > 1e-16 {
                    self.jacobian.push(i, j, deriv);
                }
            }
        }

        Ok(())
    }

    /// Line search to find damping factor α that reduces ||F||
    fn line_search(&self, delta_x: &[f64]) -> Result<f64, SolverError> {
        let current_norm = self.f.iter().map(|r| r.powi(2)).sum::<f64>().sqrt();

        for &alpha in &[1.0, 0.5, 0.25, 0.125, 0.0625] {
            let mut x_trial = self.x.clone();
            for i in 0..self.n_vars {
                x_trial[i] += alpha * delta_x[i];
            }

            // Compute ||F(x_trial)||
            // (simplified: in production, call evaluate_residuals on trial point)
            // For now, accept α = 1.0 (full Newton step)
            return Ok(alpha);
        }

        Ok(0.1)  // Fallback damping
    }

    fn extract_solution(&self) -> FlowsheetSolution {
        // Extract converged stream properties and unit duties
        FlowsheetSolution {
            streams: self.streams.clone(),
            units: vec![],  // Placeholder
            residual_norm: self.f.iter().map(|r| r.powi(2)).sum::<f64>().sqrt(),
        }
    }
}

#[derive(Debug)]
pub struct SolutionResult {
    pub converged: bool,
    pub iterations: usize,
    pub residual_norm: f64,
    pub solution: FlowsheetSolution,
}

#[derive(Debug)]
pub enum SolverError {
    Convergence(String),
    Singular(String),
    InvalidFlowsheet(String),
}
```

### 3. Peng-Robinson Flash Calculator (Rust + WASM)

The PT flash algorithm using Peng-Robinson EOS, compiled to both native Rust (server) and WASM (browser).

```rust
// thermo-core/src/flash/pr_flash.rs

use crate::eos::peng_robinson::{PengRobinson, compute_fugacity_coefficient};

pub struct PrFlash {
    pub eos: PengRobinson,
    pub n_components: usize,
}

impl PrFlash {
    pub fn new(eos: PengRobinson) -> Self {
        Self {
            n_components: eos.components.len(),
            eos,
        }
    }

    /// Two-phase PT flash: given P, T, z, find β, x, y
    pub fn pt_flash(
        &self,
        pressure: f64,    // Pa
        temperature: f64, // K
        z: &[f64],        // Overall composition (mole fractions)
    ) -> Result<FlashResult, FlashError> {
        let n = self.n_components;

        // 1. Initialize K-values using Wilson's correlation
        let mut k_values: Vec<f64> = self.eos.components.iter().map(|comp| {
            let tc = comp.critical_temp;
            let pc = comp.critical_pressure;
            let omega = comp.acentric_factor;
            (pc / pressure) * (5.37 * (1.0 + omega) * (1.0 - tc / temperature)).exp()
        }).collect();

        let mut beta = 0.5;  // Initial vapor fraction guess
        let mut x = vec![0.0; n];
        let mut y = vec![0.0; n];

        // 2. Rachford-Rice iteration
        for iteration in 0..100 {
            // Solve Rachford-Rice for β
            beta = self.rachford_rice_solve(z, &k_values)?;

            // Update phase compositions
            for i in 0..n {
                x[i] = z[i] / (1.0 + beta * (k_values[i] - 1.0));
                y[i] = k_values[i] * x[i];
            }

            // Normalize (handle numerical error)
            let sum_x: f64 = x.iter().sum();
            let sum_y: f64 = y.iter().sum();
            for i in 0..n {
                x[i] /= sum_x;
                y[i] /= sum_y;
            }

            // Compute fugacity coefficients from PR EOS
            let phi_liquid = self.eos.fugacity_coefficients(temperature, pressure, &x, Phase::Liquid);
            let phi_vapor = self.eos.fugacity_coefficients(temperature, pressure, &y, Phase::Vapor);

            // Update K-values: K_i = φ_i^L / φ_i^V
            let mut k_new = vec![0.0; n];
            for i in 0..n {
                k_new[i] = phi_liquid[i] / phi_vapor[i];
            }

            // Check convergence
            let k_error: f64 = k_new.iter().zip(&k_values).map(|(kn, ko)| ((kn - ko) / ko).powi(2)).sum::<f64>().sqrt();
            if k_error < 1e-8 {
                return Ok(FlashResult {
                    vapor_fraction: beta,
                    liquid_composition: x,
                    vapor_composition: y,
                    k_values: k_new,
                    iterations: iteration,
                });
            }

            k_values = k_new;
        }

        Err(FlashError::Convergence("PT flash did not converge".to_string()))
    }

    /// Solve Rachford-Rice equation for vapor fraction β
    fn rachford_rice_solve(&self, z: &[f64], k: &[f64]) -> Result<f64, FlashError> {
        let n = z.len();

        // Bounds: β ∈ [β_min, β_max] where RR function changes sign
        let beta_min = (z.iter().zip(k).map(|(&zi, &ki)| zi / (1.0 - ki)).fold(f64::NEG_INFINITY, f64::max)).max(0.0);
        let beta_max = (z.iter().zip(k).map(|(&zi, &ki)| zi / (ki - 1.0)).fold(f64::INFINITY, f64::min)).min(1.0);

        if beta_max <= beta_min {
            // Single phase (all liquid or all vapor)
            return Ok(if beta_min > 0.5 { 1.0 } else { 0.0 });
        }

        // Newton-Raphson on Rachford-Rice
        let mut beta = (beta_min + beta_max) / 2.0;
        for _ in 0..50 {
            let mut f = 0.0;
            let mut df = 0.0;
            for i in 0..n {
                let denom = 1.0 + beta * (k[i] - 1.0);
                f += z[i] * (k[i] - 1.0) / denom;
                df -= z[i] * (k[i] - 1.0).powi(2) / denom.powi(2);
            }

            if f.abs() < 1e-10 {
                return Ok(beta.clamp(0.0, 1.0));
            }

            beta -= f / df;
            beta = beta.clamp(beta_min, beta_max);
        }

        Ok(beta.clamp(0.0, 1.0))
    }
}

#[derive(Debug, Clone)]
pub struct FlashResult {
    pub vapor_fraction: f64,
    pub liquid_composition: Vec<f64>,
    pub vapor_composition: Vec<f64>,
    pub k_values: Vec<f64>,
    pub iterations: usize,
}

#[derive(Debug)]
pub enum FlashError {
    Convergence(String),
    InvalidInput(String),
}

#[derive(Debug, Clone, Copy)]
pub enum Phase {
    Liquid,
    Vapor,
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–4)

**Day 1: Project initialization and Rust backend scaffold**
```bash
cargo init prochem-api
cd prochem-api
cargo add axum tokio serde serde_json sqlx uuid chrono tower-http jsonwebtoken bcrypt
cargo add --features postgres,runtime-tokio-rustls,uuid,chrono,json sqlx
cargo add aws-sdk-s3 redis
```
- `src/main.rs` — Axum app with CORS, tracing, graceful shutdown
- `src/config.rs` — Environment-based configuration (DATABASE_URL, JWT_SECRET, S3_BUCKET, REDIS_URL)
- `src/state.rs` — AppState struct (PgPool, Redis, S3Client)
- `src/error.rs` — ApiError enum with IntoResponse
- `Dockerfile` — Multi-stage build (builder + runtime)
- `docker-compose.yml` — PostgreSQL, Redis, MinIO (S3-compatible)

**Day 2: Database schema and migrations**
- `migrations/001_initial.sql` — All 9 tables: users, organizations, org_members, projects, simulations, simulation_jobs, compounds, binary_interaction_params, stream_tables, usage_records
- `src/db/mod.rs` — Database pool initialization
- `src/db/models.rs` — All SQLx structs with FromRow derives
- Run `sqlx migrate run` to apply schema
- Seed script for initial compound database (top 500 industrial chemicals with DIPPR properties)

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

### Phase 2 — Thermodynamics + Flash Algorithms (Days 5–10)

**Day 5: Pure component property framework**
- `thermo-core/` — New Rust workspace member (shared between native and WASM)
- `thermo-core/src/compound.rs` — Compound struct with critical properties, DIPPR correlations
- `thermo-core/src/dippr.rs` — DIPPR correlation evaluators (Cp_ig, H_vap, rho_liq, mu, k, sigma)
- `thermo-core/src/properties.rs` — Antoine vapor pressure, liquid density, enthalpy/entropy
- Unit tests: verify DIPPR correlations for water, methane, benzene against literature

**Day 6: Peng-Robinson equation of state**
- `thermo-core/src/eos/peng_robinson.rs` — PR EOS implementation: a, b parameters, Z-factor cubic solve, fugacity coefficients
- Van der Waals mixing rules with binary interaction parameters (k_ij)
- Cubic equation solver via analytical Cardano's method
- Tests: pure component fugacities, binary mixture VLE for ethane-propane

**Day 7: Soave-Redlich-Kwong equation of state**
- `thermo-core/src/eos/soave_rk.rs` — SRK EOS (similar structure to PR, different a/b coefficients)
- Tests: compare SRK vs PR for hydrocarbon VLE, verify against NIST data

**Day 8: PT flash algorithm**
- `thermo-core/src/flash/pt_flash.rs` — Isothermal-isobaric flash with Rachford-Rice
- Wilson's K-value initialization
- Fugacity iteration until K-value convergence
- Tests: flash calculations for methanol-water, propane-butane, benzene-toluene

**Day 9: NRTL activity coefficient model**
- `thermo-core/src/activity/nrtl.rs` — NRTL model with binary interaction parameters (a_ij, α_ij)
- Activity coefficient calculation with numerical stability improvements
- NRTL flash: K_i = γ_i · P_i^sat / P
- Tests: ethanol-water VLE at 1 atm, compare to experimental data

**Day 10: WASM compilation for client-side flash**
- `thermo-wasm/` — New workspace member for WASM target
- `thermo-wasm/Cargo.toml` — wasm-bindgen, serde-wasm-bindgen dependencies
- `thermo-wasm/src/lib.rs` — WASM entry points: `pt_flash()`, `bubble_point()`, `dew_point()`, `get_properties()`
- `wasm-pack build --target web --release` pipeline
- JavaScript wrapper: `ThermoCalculator` class with async API
- Tests: verify WASM results match native Rust

### Phase 3 — Unit Operation Models (Days 11–18)

**Day 11: Stream and flowsheet framework**
- `solver-core/src/stream.rs` — Stream struct (composition, T, P, flow rate, phase)
- `solver-core/src/flowsheet.rs` — Flowsheet graph (units, streams, connectivity)
- `solver-core/src/units/mod.rs` — UnitOpTrait definition (compute_residuals, n_equations, n_vars)
- DOF (degree of freedom) analysis for flowsheet validation

**Day 12: Basic unit operations — mixer, splitter, heater**
- `solver-core/src/units/mixer.rs` — Mixer: mass/energy balance, composition mixing
- `solver-core/src/units/splitter.rs` — Splitter: split fractions, isothermal/isenthalpic
- `solver-core/src/units/heater.rs` — Heater/cooler: energy balance with specified duty or outlet T
- Tests: verify mass/energy conservation, composition propagation

**Day 13: Flash drum and phase separator**
- `solver-core/src/units/flash.rs` — Flash drum: call PT flash, split into vapor/liquid outlets
- Energy balance with flash calculation embedded
- Tests: flash drum with methanol-water feed at various T/P

**Day 14: Pump and compressor models**
- `solver-core/src/units/pump.rs` — Pump: isentropic/polytropic efficiency, power calculation, outlet pressure
- `solver-core/src/units/compressor.rs` — Compressor: multi-stage with intercooling, efficiency curves
- Tests: verify power consumption against analytical formulas

**Day 15: Heat exchanger model**
- `solver-core/src/units/heatx.rs` — Heat exchanger: hot/cold side energy balance, LMTD or ε-NTU method
- Countercurrent, co-current, and crossflow configurations
- U·A specification or outlet temperature specification
- Tests: simple HX with water cooling, verify LMTD calculation

**Day 16: CSTR and PFR reactor models**
- `solver-core/src/units/cstr.rs` — Continuous stirred-tank reactor: reaction kinetics, residence time, energy balance with heat of reaction
- `solver-core/src/units/pfr.rs` — Plug-flow reactor: ODE integration along reactor length
- Simple reaction kinetics: power-law rate expressions
- Tests: isothermal CSTR with A→B reaction, compare to analytical solution

**Day 17: Shortcut distillation column (Fenske-Underwood-Gilliland)**
- `solver-core/src/units/shortcut_column.rs` — Fenske (minimum stages), Underwood (minimum reflux), Gilliland (actual stages)
- Specify: reflux ratio, distillate composition, bottoms composition
- Tests: benzene-toluene column at 1 atm, verify stage count

**Day 18: Rigorous distillation column (MESH equations)**
- `solver-core/src/units/distillation.rs` — Rigorous column: Mass balance, Equilibrium, Summation, Heat balance (MESH) for each stage
- Bubble-point method for tray-by-tray calculation
- Reflux, reboil, feed stage specification
- Tests: simple binary column, compare to shortcut method

### Phase 4 — Equation-Oriented Solver (Days 19–24)

**Day 19: Sparse Jacobian assembly**
- `solver-core/src/eo_solver.rs` — EquationOrientedSolver struct with COO matrix for Jacobian
- Variable indexing: streams (T, P, F, composition), unit internals (duty, split fractions)
- Equation indexing: unit residuals in sequence
- Finite-difference Jacobian computation with sparsity detection

**Day 20: KLU sparse solver integration**
- `solver-core/src/klu.rs` — KLU wrapper via suitesparse-sys bindings
- CSR matrix conversion from COO
- Tests: solve small linear systems, verify against dense solver

**Day 21: Newton-Raphson with line search**
- Implement full Newton-Raphson loop in `eo_solver.rs`
- Line search damping to ensure ||F|| monotonically decreases
- Convergence monitoring: residual norm, solution change
- Tests: simple flowsheet (heater → flash → cooler), verify convergence

**Day 22: Recycle loop convergence — Wegstein method**
- `solver-core/src/recycle/wegstein.rs` — Wegstein acceleration for tear streams
- Outer loop: update tear stream guess, run flowsheet, compute new tear stream
- Tests: simple recycle (reactor → separator → recycle to reactor)

**Day 23: Recycle loop convergence — Broyden method**
- `solver-core/src/recycle/broyden.rs` — Quasi-Newton Broyden update for recycle Jacobian
- Better for highly nonlinear recycles (large recycle ratios)
- Tests: compare Wegstein vs Broyden on stiff recycle problem

**Day 24: Flowsheet initialization and warm-start**
- Smart initialization: propagate feed streams forward, guess intermediate streams
- Sequential modular pre-solve to get initial guess for EO solver
- Tests: verify initialization accelerates convergence by 2-3x

### Phase 5 — Frontend Flowsheet Editor (Days 25–31)

**Day 25: Frontend scaffold and flowsheet canvas**
```bash
npm create vite@latest frontend -- --template react-ts
cd frontend
npm i zustand @tanstack/react-query axios konva react-konva
```
- `src/App.tsx` — Router, auth context, layout
- `src/stores/flowsheetStore.ts` — Zustand store for flowsheet state (units, streams)
- `src/components/FlowsheetCanvas.tsx` — Konva Stage/Layer for flowsheet rendering
- Pan/zoom with mouse drag and wheel

**Day 26: Unit operation palette and placement**
- `src/components/UnitPalette.tsx` — Categorized unit operation library sidebar (Separators, Reactors, Heat Transfer, Utilities)
- `src/components/UnitSymbol.tsx` — SVG/Konva symbol renderer for each unit type
- Drag-and-drop from palette → canvas
- Unit selection, move, delete (Del key)
- Property panel: unit name, specifications

**Day 27: Stream drawing and connectivity**
- `src/components/StreamConnector.tsx` — Click outlet port → click inlet port to create stream
- Stream auto-routing: Manhattan routing with obstacle avoidance
- Stream labels with flow rate, composition preview
- Cross-probing: hover stream → highlight in stream table

**Day 28: Stream table and property display**
- `src/components/StreamTable.tsx` — D3.js table with columns: Stream, T, P, Vapor Frac, Flow, Composition
- Unit system selector (SI / US Customary / Engineering)
- Copy stream data to clipboard as CSV
- Stream property plots (T-P diagram, composition pie chart)

**Day 29: Unit specification panel**
- `src/components/UnitSpecPanel.tsx` — Dynamic form for unit specifications based on unit type
- Flash drum: specify P, T (or Q=0 for adiabatic)
- Heater: specify T_out or Q
- Distillation: specify reflux ratio, distillate rate, number of stages
- Validation: check for under/over-specified units (DOF analysis)

**Day 30: Compound selector and thermodynamic method**
- `src/components/CompoundSelector.tsx` — Search 2,000 compounds by name or CAS number
- Multi-select with composition input (mole or mass fractions)
- `src/components/ThermoMethodSelector.tsx` — Radio buttons: PR EOS, SRK EOS, NRTL, Ideal
- Client-side flash preview using WASM (instant feedback on property changes)

**Day 31: Flowsheet validation and simulation trigger**
- `src/components/FlowsheetValidator.tsx` — Pre-simulation checks: floating units, unconnected streams, missing specs
- DOF analysis displayed per unit
- "Run Simulation" button → API call to create simulation
- Loading state with progress bar

### Phase 6 — Simulation Execution + Results (Days 32–36)

**Day 32: Simulation worker and job processing**
- `src/workers/simulation_worker.rs` — Redis job consumer, runs EO solver
- `src/workers/mod.rs` — Worker pool management (configurable concurrency)
- Progress streaming: worker publishes iteration count + residual → Redis pub/sub → WebSocket to client
- S3 result upload: converged flowsheet JSON with full stream tables

**Day 33: WebSocket for live simulation progress**
- `src/api/ws/simulation_progress.rs` — Subscribe to simulation progress channel
- Client receives: `{ iteration, residual_norm, current_unit, status }`
- Frontend: `src/hooks/useSimulationProgress.ts` — React hook for WebSocket subscription
- Convergence monitor panel: real-time residual plot with D3.js

**Day 34: Result visualization — converged flowsheet**
- `src/components/ConvergedFlowsheet.tsx` — Display converged flowsheet with stream values overlaid
- Stream thickness proportional to flow rate
- Color-coded streams by temperature or composition
- Tooltip: hover stream → full property popup

**Day 35: Result tables and export**
- `src/components/ResultTables.tsx` — Detailed stream table, unit summary table (duties, efficiencies)
- Export to Excel (XLSX), CSV, PDF
- Equipment datasheets: standardized format for pumps, HX, columns

**Day 36: Error handling and convergence diagnostics**
- Non-convergent simulation → display residual plot, identify problematic units
- Suggestions: "Mixer-301 mass balance residual is high. Check inlet stream compositions."
- Retry with relaxed tolerances or different initialization
- Save failed simulation state for debugging

### Phase 7 — Billing + Plan Enforcement (Days 37–39)

**Day 37: Stripe integration**
- `src/api/handlers/billing.rs` — Create checkout session, create customer portal session, get subscription status
- `src/api/handlers/webhooks/stripe.rs` — Handle subscription events
- Plan mapping:
  - Free: 10 unit operations, 3 projects, PR/SRK EOS only, basic compounds
  - Pro ($149/mo): Unlimited units, NRTL, full compound database, report export
  - Advanced ($349/mo): Dynamic simulation (Phase 8+), optimization, API access, team collaboration

**Day 38: Usage tracking and plan limits**
- `src/middleware/plan_limits.rs` — Middleware checking plan limits before simulation execution
- `src/services/usage.rs` — Track simulation runs per billing period
- Approaching-limit warnings at 80% and 100% of monthly quota

**Day 39: Billing UI and feature gating**
- `frontend/src/pages/Billing.tsx` — Plan comparison table, current plan display, usage meter
- Upgrade/downgrade flow via Stripe Customer Portal
- Gate advanced features (dynamic sim, optimization) behind Advanced plan
- Show locked feature indicators with upgrade CTAs

### Phase 8 — Testing + Deployment (Days 40–42)

**Day 40: Solver validation benchmarks**
- Benchmark 1: Methanol-water flash at 1 atm, 80°C → verify vapor fraction within 1%
- Benchmark 2: Benzene-toluene distillation (10 stages, reflux=2) → verify product purities
- Benchmark 3: Simple recycle (reactor → flash → recycle) → verify convergence in <20 iterations
- Benchmark 4: Heat exchanger network (3 HX) → verify LMTD and duties
- Automated test suite: `solver-core/tests/benchmarks.rs`

**Day 41: Integration testing and performance**
- End-to-end: create project → draw flowsheet → run simulation → view results → export
- API integration tests: auth → project CRUD → simulation → results
- WASM thermo test: verify client-side flash matches server-side
- WebSocket test: connect → subscribe → receive progress → receive completion
- Performance: 20-unit flowsheet convergence in <10 seconds

**Day 42: Deployment and launch**
- `k8s/` — Kubernetes manifests: API, workers, PostgreSQL, Redis
- CloudFront CDN for WASM bundle and frontend assets
- Prometheus + Grafana dashboards: simulation throughput, convergence rate, API latency
- Sentry error tracking
- Rate limiting: 50 req/min per user, 5 simulations/hour for free tier
- Landing page with product overview, pricing, signup
- Documentation: getting started guide, unit operation reference, thermodynamic model guide
- Deploy to production, enable monitoring, announce launch

---

## Critical Files

```
prochem/
├── thermo-core/                           # Thermodynamics library (native + WASM)
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── compound.rs                    # Pure component properties
│   │   ├── dippr.rs                       # DIPPR correlations
│   │   ├── properties.rs                  # Vapor pressure, density, enthalpy
│   │   ├── eos/
│   │   │   ├── mod.rs
│   │   │   ├── peng_robinson.rs           # PR equation of state
│   │   │   └── soave_rk.rs                # SRK equation of state
│   │   ├── activity/
│   │   │   ├── mod.rs
│   │   │   └── nrtl.rs                    # NRTL activity coefficient model
│   │   └── flash/
│   │       ├── mod.rs
│   │       └── pt_flash.rs                # Isothermal-isobaric flash
│   └── tests/
│       ├── eos_validation.rs              # EOS verification
│       └── flash_validation.rs            # Flash algorithm tests
│
├── thermo-wasm/                           # WASM compilation target
│   ├── Cargo.toml
│   └── src/
│       └── lib.rs                         # WASM entry points (wasm-bindgen)
│
├── solver-core/                           # Process simulation engine
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── stream.rs                      # Stream data structure
│   │   ├── flowsheet.rs                   # Flowsheet graph and connectivity
│   │   ├── eo_solver.rs                   # Equation-oriented solver with sparse NR
│   │   ├── klu.rs                         # KLU sparse solver wrapper
│   │   ├── units/
│   │   │   ├── mod.rs                     # UnitOpTrait definition
│   │   │   ├── mixer.rs
│   │   │   ├── splitter.rs
│   │   │   ├── heater.rs
│   │   │   ├── flash.rs
│   │   │   ├── pump.rs
│   │   │   ├── compressor.rs
│   │   │   ├── heatx.rs                   # Heat exchanger
│   │   │   ├── cstr.rs                    # CSTR reactor
│   │   │   ├── pfr.rs                     # PFR reactor
│   │   │   ├── shortcut_column.rs         # Fenske-Underwood-Gilliland
│   │   │   └── distillation.rs            # Rigorous MESH column
│   │   └── recycle/
│   │       ├── mod.rs
│   │       ├── wegstein.rs                # Wegstein acceleration
│   │       └── broyden.rs                 # Broyden quasi-Newton
│   └── tests/
│       ├── benchmarks.rs                  # Solver validation benchmarks
│       └── convergence.rs                 # Convergence stress tests
│
├── prochem-api/                           # Rust API server
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
│   │   │   │   ├── compounds.rs           # Compound search/get
│   │   │   │   ├── billing.rs             # Stripe checkout/portal
│   │   │   │   └── webhooks/
│   │   │   │       └── stripe.rs          # Stripe webhook handler
│   │   │   └── ws/
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
│       └── api_integration.rs             # API integration tests
│
├── frontend/                              # React frontend
│   ├── package.json
│   ├── vite.config.ts
│   ├── src/
│   │   ├── App.tsx                        # Router, providers, layout
│   │   ├── main.tsx                       # Entry point
│   │   ├── stores/
│   │   │   ├── authStore.ts               # Auth state (JWT, user)
│   │   │   ├── flowsheetStore.ts          # Flowsheet editor state
│   │   │   └── simulationStore.ts         # Simulation results state
│   │   ├── hooks/
│   │   │   ├── useSimulationProgress.ts   # WebSocket hook for live progress
│   │   │   └── useThermoWasm.ts           # WASM thermo calculator hook
│   │   ├── lib/
│   │   │   ├── api.ts                     # Axios API client
│   │   │   └── wasmLoader.ts              # WASM bundle loading
│   │   ├── pages/
│   │   │   ├── Dashboard.tsx              # Project list
│   │   │   ├── FlowsheetEditor.tsx        # Main editor (canvas + panels)
│   │   │   ├── Results.tsx                # Simulation results viewer
│   │   │   ├── Compounds.tsx              # Compound database browser
│   │   │   └── Billing.tsx                # Plan management
│   │   └── components/
│   │       ├── FlowsheetCanvas.tsx        # Konva flowsheet renderer
│   │       ├── UnitPalette.tsx            # Unit operation library
│   │       ├── UnitSymbol.tsx             # Unit symbols (Konva shapes)
│   │       ├── StreamConnector.tsx        # Stream drawing tool
│   │       ├── StreamTable.tsx            # Stream property table (D3.js)
│   │       ├── UnitSpecPanel.tsx          # Unit specification form
│   │       ├── CompoundSelector.tsx       # Compound search and selection
│   │       ├── ThermoMethodSelector.tsx   # Thermo method picker
│   │       ├── FlowsheetValidator.tsx     # Pre-sim validation checks
│   │       ├── ConvergedFlowsheet.tsx     # Result visualization
│   │       └── ResultTables.tsx           # Detailed result tables
│   └── public/
│       └── wasm/                          # WASM thermo bundle
│
├── optimization-service/                  # Python FastAPI (optional, for Phase 8+)
│   ├── requirements.txt
│   ├── main.py                            # FastAPI app
│   ├── optimizers/
│   │   ├── scipy_wrapper.py               # SciPy minimize wrapper
│   │   └── pyomo_nlp.py                   # Pyomo IPOPT interface
│   └── Dockerfile
│
├── k8s/                                   # Kubernetes manifests
│   ├── api-deployment.yaml
│   ├── worker-deployment.yaml
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

### Benchmark 1: Methanol-Water Flash (PT Flash)

**Conditions:** P = 101325 Pa (1 atm), T = 353.15 K (80°C), z_MeOH = 0.4, z_H2O = 0.6

**Thermo Method:** NRTL activity coefficient model with standard parameters

**Expected:** Vapor fraction β ≈ **0.35**, y_MeOH ≈ **0.68**, x_MeOH ≈ **0.26**

**Tolerance:** β < 2%, composition < 3% (NRTL parameter sensitivity)

### Benchmark 2: Benzene-Toluene Distillation (Rigorous Column)

**Configuration:** Feed at stage 5 (of 10 total stages), F = 100 kmol/h, z_benzene = 0.5, reflux ratio R = 2.0, P = 101325 Pa

**Thermo Method:** Peng-Robinson EOS (ideal for hydrocarbons)

**Expected:** Distillate purity x_D,benzene > **0.95**, Bottoms purity x_B,toluene > **0.95**, Reboiler duty ≈ **3.2 MW**

**Tolerance:** Purity < 1%, duty < 5%

### Benchmark 3: Simple Recycle Convergence (Reactor + Flash + Recycle)

**Flowsheet:** Feed → CSTR (A→B, k=0.1 s⁻¹) → Flash (T=350K, P=5 bar) → Vapor to product, Liquid recycles to reactor

**Feed:** F = 50 kmol/h, pure A

**Expected:** Converged recycle flow ≈ **35 kmol/h**, Reactor conversion ≈ **0.75**, Wegstein iterations < **15**

**Tolerance:** Flow < 2%, conversion < 3%, iterations < 20

### Benchmark 4: Heat Exchanger LMTD Verification

**Configuration:** Hot stream (oil): T_in = 150°C, T_out = 80°C, ṁ = 10 kg/s, Cp = 2.1 kJ/kg·K. Cold stream (water): T_in = 20°C, ṁ = 15 kg/s, Cp = 4.18 kJ/kg·K. Countercurrent.

**Expected:** LMTD ≈ **54.9 K**, Heat duty Q ≈ **1.47 MW**, T_out,cold ≈ **43.4°C**

**Tolerance:** LMTD < 1%, Duty < 0.5%, T_out < 0.5 K

### Benchmark 5: Pump Power Calculation

**Conditions:** Liquid water at 25°C, ρ = 997 kg/m³, ṁ = 50 kg/s, ΔP = 500 kPa (5 bar head), η_pump = 0.75

**Expected:** Hydraulic power P_hyd = **25.1 kW**, Shaft power P_shaft = **33.5 kW**

**Tolerance:** < 1% (analytical formula verification)

---

## Verification

### Integration Testing Checklist

1. **Auth flow** — Register → login → JWT issued → authorized API calls → token refresh → logout
2. **Project CRUD** — Create → update flowsheet → save → reload → verify flowsheet state preserved
3. **Flowsheet editing** — Place units → connect streams → set specs → validate DOF → verify netlist generation
4. **Client-side flash** — Select compounds → set T/P → WASM flash → results displayed instantly
5. **Server simulation** — Full flowsheet → job queued → worker picks up → WebSocket progress → results in S3
6. **Result visualization** — Converged flowsheet → stream table → export to Excel → verify data correctness
7. **Compound database** — Search "methanol" → results returned → select → use in flowsheet
8. **Recycle convergence** — Draw recycle flowsheet → simulate → verify Wegstein converges
9. **Plan limits** — Free user → 15-unit flowsheet → blocked with upgrade prompt
10. **Billing** — Subscribe to Pro → Stripe checkout → webhook → plan updated → limits lifted
11. **Concurrent users** — 5 users simultaneously running simulations → all complete correctly
12. **Error handling** — Non-convergent flowsheet → meaningful error with residual plot → no crash

### SQL Verification Queries

```sql
-- 1. Simulation throughput and success rate
SELECT
    status,
    COUNT(*) as count,
    AVG(wall_time_ms)::int as avg_time_ms,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY wall_time_ms)::int as p95_time_ms,
    AVG(iteration_count)::int as avg_iterations
FROM simulations
WHERE created_at >= NOW() - INTERVAL '7 days'
GROUP BY status;

-- 2. Thermodynamic method distribution
SELECT thermo_method, COUNT(*) as count,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 1) as pct
FROM simulations
WHERE created_at >= NOW() - INTERVAL '30 days'
GROUP BY thermo_method ORDER BY count DESC;

-- 3. User plan distribution and conversion
SELECT plan, COUNT(*) as users,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 1) as pct
FROM users GROUP BY plan ORDER BY users DESC;

-- 4. Compound usage popularity
SELECT c.name, c.formula, COUNT(DISTINCT s.project_id) as projects_using
FROM compounds c
JOIN stream_tables st ON st.composition::jsonb ? c.id::text
JOIN simulations s ON st.simulation_id = s.id
WHERE s.created_at >= NOW() - INTERVAL '90 days'
GROUP BY c.id
ORDER BY projects_using DESC
LIMIT 20;

-- 5. Average flowsheet complexity by plan
SELECT u.plan,
    AVG(jsonb_array_length(p.flowsheet_data->'units'))::int as avg_units,
    AVG(jsonb_array_length(p.flowsheet_data->'streams'))::int as avg_streams
FROM projects p
JOIN users u ON p.owner_id = u.id
GROUP BY u.plan ORDER BY avg_units DESC;
```

### Performance Benchmarks

| Operation | Target | Method |
|-----------|--------|--------|
| WASM flash: binary mixture | <50ms | Browser benchmark (Chrome DevTools) |
| WASM flash: 10-component mixture | <200ms | Browser benchmark |
| Server simulation: 20-unit flowsheet | <10s | Server timing, 4 cores |
| Server simulation: 100-unit refinery | <2min | Server timing, 8 cores |
| Recycle convergence: simple loop | <5s | Wegstein method, 10-15 iterations |
| Flowsheet canvas: 50 units render | 60 FPS | Konva performance |
| API: create simulation | <150ms | p95 latency (Prometheus) |
| API: search compounds | <80ms | p95 latency with 2K compounds |
| WASM bundle load (cached) | <300ms | Service worker cache hit |
| WASM bundle load (cold) | <2s | CDN delivery, ~1.5MB gzipped |
| WebSocket: progress latency | <100ms | Time from worker emit to client receive |

---

## Deployment Architecture

### Infrastructure

```
┌─────────────────────────────────────────────────┐
│                  CloudFront CDN                   │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────┐ │
│  │ Frontend SPA │  │ WASM Bundle │  │ Compound │ │
│  │ (HTML/JS/CSS)│  │ (versioned) │  │ Data     │ │
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
│ Multi-AZ     │ │ Cluster mode │ │  Compounds)  │
└──────────────┘ └──────┬───────┘ └──────────────┘
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ Sim Worker   │ │ Sim Worker   │ │ Sim Worker   │
│ (Rust native)│ │ (Rust native)│ │ (Rust native)│
│ c7g.2xlarge  │ │ c7g.2xlarge  │ │ c7g.2xlarge  │
│ HPA: 2-20    │ │              │ │              │
└──────────────┘ └──────────────┘ └──────────────┘
```

### Scaling Strategy

- **API servers**: Horizontal pod autoscaler (HPA) at 70% CPU, min 3, max 10
- **Simulation workers**: HPA based on Redis queue depth — scale up when queue > 3 jobs, scale down when idle 5 min
- **Worker instances**: AWS c7g.2xlarge (8 vCPU, 16GB RAM) for compute-optimized simulation
- **Database**: RDS PostgreSQL r6g.xlarge with read replicas for compound search queries
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
      "ID": "compound-data-keep",
      "Prefix": "compounds/",
      "Status": "Enabled",
      "Transitions": [
        { "Days": 180, "StorageClass": "STANDARD_IA" }
      ]
    }
  ]
}
```

---

## Post-MVP Roadmap

| Feature | Description | Priority |
|---------|-------------|----------|
| Dynamic Simulation | Convert steady-state flowsheet to dynamic model with DAE solver (BDF method), PID controller modeling, startup/shutdown scenarios, and transient safety analysis for pressure relief and blowdown | High |
| Equipment Sizing and Costing | Automatic equipment sizing (column diameter, tray spacing, HX area, pump power) with cost estimation using Guthrie/Lang correlations, CAPEX/OPEX summary, NPV/IRR economic analysis | High |
| Energy Optimization (Pinch Analysis) | Heat integration using pinch analysis, composite curves and grand composite curves, minimum utility targeting, heat exchanger network (HEN) synthesis, energy cost minimization | High |
| Electrolyte Thermodynamics | Electrolyte NRTL and Pitzer models for aqueous ionic solutions, mean spherical approximation (MSA), applications in CO2 capture, wastewater treatment, battery electrolytes | Medium |
| Batch Process Simulation | Recipe-driven batch simulation with time-sequenced operations, batch reactor kinetics, crystallizer modeling, gantt chart scheduling for multi-product campaigns | Medium |
| Sensitivity Analysis and Optimization | Parametric sweeps of process variables (reflux ratio, feed rate, temperature), multi-objective optimization (minimize cost, maximize yield), gradient-free optimizers (Nelder-Mead, PSO) | Medium |
| Petroleum Characterization | Pseudo-component generation from TBP/D86 distillation curves, API gravity correlations, crude oil assay import, applications in refinery simulation (CDU, FCC, hydrocracker) | Low |
| Real-Time Collaboration | CRDT-based flowsheet editing with cursor presence, commenting on units/streams, shared simulation results for team design reviews, version control integration | Low |
