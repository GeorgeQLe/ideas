# 67. PharmaFlow — Pharmaceutical Process Development and Scale-Up Platform

## Implementation Plan

**MVP Scope:** Browser-based reaction scheme builder with drag-and-drop component placement (reactants, intermediates, products, catalysts) and reaction pathway visualization via SVG, kinetic modeling engine implementing power-law and Arrhenius rate equations with nonlinear least-squares parameter estimation compiled to WebAssembly for client-side fitting of datasets ≤1000 data points and server-side Rust-native execution for larger datasets or Bayesian MCMC estimation, batch and semi-batch reactor simulation with mass and energy balance (jacket heat transfer, adiabatic temperature rise, heat of reaction) using LSODA adaptive-step ODE solver, scale-up prediction using geometric similarity and constant tip speed/power-per-volume/mixing time correlations with equipment database covering 1L-10,000L reactors, interactive result visualization via D3.js for concentration-time profiles, temperature trajectories, and sensitivity analysis (Tornado charts), parameter estimation UI showing residual plots, correlation matrices, and 95% confidence intervals, CSV import for experimental data (time, temperature, concentration, conversion) with unit handling (mM, M, g/L, wt%), PDF report generation with fitted kinetic parameters, reactor simulation results, and scale-up recommendations, Stripe billing with three tiers (Academic $99/month / Pro $299/month / Enterprise custom).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| Kinetic Solver | Rust (native + WASM) | Custom ODE solver with LSODA implementation, nonlinear least-squares via Levenberg-Marquardt |
| WASM Compilation | `wasm-pack` + `wasm-bindgen` | Client-side kinetic fitting for datasets ≤1000 points |
| Bayesian Estimation | Python 3.12 (FastAPI) | PyMC3/NumPyro for MCMC parameter estimation with uncertainty quantification |
| Database | PostgreSQL 16 | Projects, users, kinetic models, reactor configurations, experimental datasets |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | Experimental datasets, simulation results, PDF reports, equipment database |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| Reaction Scheme Editor | Custom SVG renderer | Force-directed graph layout for reaction pathways, drag-and-drop components |
| Result Visualization | D3.js v7 | Time-series plots, sensitivity Tornado charts, residual plots with pan/zoom |
| 3D Reactor Visualization | Three.js r160 | Optional 3D visualization of reactor geometry for scale-up scenarios |
| Real-time | WebSocket (Axum) | Live simulation progress, parameter estimation convergence monitoring |
| Job Queue | Redis 7 + Tokio tasks | Server-side MCMC jobs, Monte Carlo simulations, batch PDF generation |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | WASM bundle delivery, static frontend assets, equipment database |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, error tracking |
| CI/CD | GitHub Actions | WASM build pipeline, Rust tests, Docker image push |

### Key Architecture Decisions

1. **Hybrid client/server kinetic solver with 1000-point WASM threshold**: Parameter estimation for datasets with ≤1000 data points (covers 95%+ of lab-scale kinetic studies with 20-50 timepoints and 5-20 species) runs entirely in the browser via WASM, providing instant feedback with zero server cost. Larger datasets or Bayesian MCMC estimation are submitted to the server. The threshold is configurable per plan tier.

2. **Custom ODE solver in Rust with LSODA rather than wrapping ODEPACK/Sundials**: Building a custom LSODA (Livermore Solver for Ordinary Differential Equations with Automatic stiffness detection) in Rust gives us full control over WASM compilation, parameter sensitivity calculation, and thread-safe parallel evaluation for uncertainty quantification. Foreign function interface (FFI) to C-based ODEPACK would break WASM compatibility and introduce unsafe code. Our Rust implementation uses Adams-Moulton (non-stiff) and BDF (stiff) methods with automatic switching.

3. **SVG reaction scheme editor with force-directed layout**: SVG gives crisp rendering at any zoom level and straightforward DOM-based hit testing for reaction components. Force-directed graph layout (via D3-force) automatically positions species and reactions to minimize edge crossings while allowing manual repositioning. Canvas/WebGL alternatives were rejected because they require reimplementing text layout and accessibility.

4. **D3.js for result visualization separate from Three.js reactor 3D**: D3.js handles all 2D scientific plots (concentration vs time, Arrhenius plots, parity plots, sensitivity charts) with interactive pan/zoom and crosshair tooltips. Three.js provides optional 3D visualization of reactor geometry, impeller position, jacket configuration for scale-up scenarios. Decoupling allows users to toggle 3D view independently.

5. **S3 for experimental data and equipment database with PostgreSQL metadata**: Raw experimental data (typically 10-100KB CSV files) and equipment specifications (reactor volumes, impeller types, heat transfer correlations) are stored in S3, while PostgreSQL holds searchable metadata (experiment date, species names, equipment vendor, size range). This allows the platform to scale to 100K+ experiments without bloating the database while enabling fast parametric search via PostgreSQL indexes and full-text search.

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";  -- For fuzzy text search on models

-- Users
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email TEXT UNIQUE NOT NULL,
    password_hash TEXT,  -- NULL for OAuth-only users
    name TEXT NOT NULL,
    avatar_url TEXT,
    auth_provider TEXT DEFAULT 'email',  -- email | google | github
    auth_provider_id TEXT,
    plan TEXT NOT NULL DEFAULT 'free',  -- free | academic | pro | enterprise
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    organization TEXT,  -- Company or university name
    role TEXT,  -- Job title
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

-- Projects (reaction scheme + kinetic model workspace)
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    project_type TEXT NOT NULL DEFAULT 'kinetic',  -- kinetic | reactor | crystallization | bioreactor
    reaction_scheme JSONB NOT NULL DEFAULT '{}',  -- Species, reactions, stoichiometry
    settings JSONB DEFAULT '{}',  -- Display preferences, units
    is_public BOOLEAN NOT NULL DEFAULT false,
    forked_from UUID REFERENCES projects(id) ON DELETE SET NULL,
    thumbnail_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);
CREATE INDEX projects_org_idx ON projects(org_id);
CREATE INDEX projects_type_idx ON projects(project_type);
CREATE INDEX projects_updated_idx ON projects(updated_at DESC);

-- Experimental Datasets
CREATE TABLE datasets (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    experiment_date DATE,
    data_url TEXT NOT NULL,  -- S3 URL to CSV file
    data_summary JSONB NOT NULL,  -- {num_rows, num_species, time_range, species_names, units}
    conditions JSONB DEFAULT '{}',  -- Temperature, pressure, initial concentrations
    metadata JSONB DEFAULT '{}',  -- Lab notebook reference, analyst, equipment
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX datasets_project_idx ON datasets(project_id);
CREATE INDEX datasets_user_idx ON datasets(user_id);

-- Kinetic Models (fitted parameters)
CREATE TABLE kinetic_models (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    dataset_id UUID REFERENCES datasets(id) ON DELETE SET NULL,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    rate_laws JSONB NOT NULL,  -- [{reaction_id, law_type: "power_law" | "arrhenius" | "mm", parameters}]
    fitted_parameters JSONB NOT NULL,  -- {param_name: {value, std_error, ci_lower, ci_upper}}
    estimation_method TEXT NOT NULL,  -- nlls | mcmc | bayesian_vi
    goodness_of_fit JSONB NOT NULL,  -- {r_squared, rmse, aic, bic, residuals_url}
    parameter_correlation JSONB,  -- Correlation matrix for parameter uncertainty
    status TEXT NOT NULL DEFAULT 'draft',  -- draft | converged | failed
    error_message TEXT,
    estimation_time_ms INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX models_project_idx ON kinetic_models(project_id);
CREATE INDEX models_dataset_idx ON kinetic_models(dataset_id);
CREATE INDEX models_status_idx ON kinetic_models(status);

-- Reactor Simulations
CREATE TABLE reactor_simulations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    kinetic_model_id UUID NOT NULL REFERENCES kinetic_models(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    reactor_type TEXT NOT NULL,  -- batch | semibatch | cstr | pfr | flow
    reactor_config JSONB NOT NULL,  -- Volume, temperature profile, feed rates, initial concentrations
    scale TEXT NOT NULL DEFAULT 'lab',  -- lab | pilot | manufacturing
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | running | completed | failed
    execution_mode TEXT NOT NULL DEFAULT 'wasm',  -- wasm | server
    results_url TEXT,  -- S3 URL for time-series result data
    results_summary JSONB,  -- {max_conversion, yield, selectivity, final_temp, adiabatic_rise}
    error_message TEXT,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    wall_time_ms INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX sims_project_idx ON reactor_simulations(project_id);
CREATE INDEX sims_model_idx ON reactor_simulations(kinetic_model_id);
CREATE INDEX sims_status_idx ON reactor_simulations(status);
CREATE INDEX sims_created_idx ON reactor_simulations(created_at DESC);

-- Scale-Up Analyses
CREATE TABLE scaleup_analyses (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    base_simulation_id UUID NOT NULL REFERENCES reactor_simulations(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    base_scale JSONB NOT NULL,  -- {volume_L, impeller_type, rpm, power_per_volume}
    target_scales JSONB NOT NULL,  -- [{volume_L, predicted_rpm, predicted_temp_profile, correlation_used}]
    correlation_method TEXT NOT NULL,  -- constant_tip_speed | constant_pv | constant_mixing_time | constant_kla
    predictions JSONB NOT NULL,  -- {scale: {conversion, yield, temp_max, mixing_time}}
    confidence_intervals JSONB,  -- Parameter uncertainty propagation
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX scaleup_project_idx ON scaleup_analyses(project_id);
CREATE INDEX scaleup_base_sim_idx ON scaleup_analyses(base_simulation_id);

-- Sensitivity Analyses
CREATE TABLE sensitivity_analyses (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    simulation_id UUID NOT NULL REFERENCES reactor_simulations(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    analysis_type TEXT NOT NULL,  -- local | global_sobol
    parameters JSONB NOT NULL,  -- [{param_name, base_value, range_low, range_high}]
    results JSONB NOT NULL,  -- {param_name: {sensitivity_index, delta_yield, delta_conversion}}
    results_url TEXT,  -- S3 URL for detailed sensitivity data
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX sensitivity_sim_idx ON sensitivity_analyses(simulation_id);

-- Equipment Database (reactor geometries, impellers, heat transfer correlations)
CREATE TABLE equipment_specs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    equipment_type TEXT NOT NULL,  -- reactor | impeller | jacket | coil
    vendor TEXT NOT NULL,
    model TEXT NOT NULL,
    specifications JSONB NOT NULL,  -- {volume_L, diameter_m, height_m, jacket_area_m2, impeller_diameter_m}
    heat_transfer_correlation JSONB,  -- {correlation_name, coefficients, valid_range}
    mixing_correlation JSONB,  -- {correlation_name, coefficients, valid_range}
    is_builtin BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX equipment_type_idx ON equipment_specs(equipment_type);
CREATE INDEX equipment_vendor_idx ON equipment_specs(vendor);

-- Server Jobs (for server-side parameter estimation and simulation)
CREATE TABLE computation_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    job_type TEXT NOT NULL,  -- parameter_estimation | reactor_simulation | sensitivity_analysis | scaleup
    reference_id UUID NOT NULL,  -- ID of kinetic_model, reactor_simulation, etc.
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    worker_id TEXT,
    cores_allocated INTEGER DEFAULT 1,
    memory_mb INTEGER DEFAULT 4096,
    priority INTEGER DEFAULT 0,  -- Higher = more urgent
    progress_pct REAL DEFAULT 0.0,
    progress_data JSONB DEFAULT '{}',  -- {iteration, objective_value, convergence_metric}
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX jobs_ref_idx ON computation_jobs(reference_id);
CREATE INDEX jobs_user_idx ON computation_jobs(user_id);
CREATE INDEX jobs_status_idx ON computation_jobs(worker_id);

-- Report Generation
CREATE TABLE reports (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    report_type TEXT NOT NULL,  -- kinetic_model | reactor_simulation | scaleup_analysis | full_project
    content_config JSONB NOT NULL,  -- {include_residuals, include_sensitivity, include_3d_reactor}
    pdf_url TEXT,
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | generating | completed | failed
    error_message TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at TIMESTAMPTZ
);
CREATE INDEX reports_project_idx ON reports(project_id);
CREATE INDEX reports_status_idx ON reports(status);

-- Usage Tracking (for billing enforcement)
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,  -- compute_minutes | dataset_storage_mb | report_generation
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

use chrono::{DateTime, Utc, NaiveDate};
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
    pub role: Option<String>,
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
    pub project_type: String,
    pub reaction_scheme: serde_json::Value,
    pub settings: serde_json::Value,
    pub is_public: bool,
    pub forked_from: Option<Uuid>,
    pub thumbnail_url: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Dataset {
    pub id: Uuid,
    pub project_id: Uuid,
    pub user_id: Uuid,
    pub name: String,
    pub description: String,
    pub experiment_date: Option<NaiveDate>,
    pub data_url: String,
    pub data_summary: serde_json::Value,
    pub conditions: serde_json::Value,
    pub metadata: serde_json::Value,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct KineticModel {
    pub id: Uuid,
    pub project_id: Uuid,
    pub dataset_id: Option<Uuid>,
    pub user_id: Uuid,
    pub name: String,
    pub description: String,
    pub rate_laws: serde_json::Value,
    pub fitted_parameters: serde_json::Value,
    pub estimation_method: String,
    pub goodness_of_fit: serde_json::Value,
    pub parameter_correlation: Option<serde_json::Value>,
    pub status: String,
    pub error_message: Option<String>,
    pub estimation_time_ms: Option<i32>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct ReactorSimulation {
    pub id: Uuid,
    pub project_id: Uuid,
    pub kinetic_model_id: Uuid,
    pub user_id: Uuid,
    pub name: String,
    pub reactor_type: String,
    pub reactor_config: serde_json::Value,
    pub scale: String,
    pub status: String,
    pub execution_mode: String,
    pub results_url: Option<String>,
    pub results_summary: Option<serde_json::Value>,
    pub error_message: Option<String>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub wall_time_ms: Option<i32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct ReactionScheme {
    pub species: Vec<Species>,
    pub reactions: Vec<Reaction>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct Species {
    pub id: String,
    pub name: String,
    pub cas_number: Option<String>,
    pub molecular_weight: f64,  // g/mol
    pub phase: String,  // liquid | gas | solid
    pub initial_concentration: Option<f64>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct Reaction {
    pub id: String,
    pub name: String,
    pub reactants: Vec<Stoichiometry>,
    pub products: Vec<Stoichiometry>,
    pub rate_law: RateLaw,
    pub heat_of_reaction: Option<f64>,  // kJ/mol, negative = exothermic
}

#[derive(Debug, Deserialize, Serialize)]
pub struct Stoichiometry {
    pub species_id: String,
    pub coefficient: f64,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(tag = "type")]
pub enum RateLaw {
    PowerLaw {
        rate_constant: Parameter,
        orders: Vec<ReactionOrder>,
    },
    Arrhenius {
        pre_exponential: Parameter,  // A, units depend on order
        activation_energy: Parameter,  // Ea, kJ/mol
        orders: Vec<ReactionOrder>,
    },
    MichaelisMenten {
        vmax: Parameter,
        km: Parameter,
        substrate_id: String,
    },
}

#[derive(Debug, Deserialize, Serialize)]
pub struct ReactionOrder {
    pub species_id: String,
    pub order: f64,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct Parameter {
    pub name: String,
    pub value: f64,
    pub units: String,
    pub bounds: Option<(f64, f64)>,
    pub fixed: bool,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct ReactorConfig {
    pub reactor_type: ReactorType,
    pub volume: f64,  // L
    pub temperature_control: TemperatureControl,
    pub initial_conditions: Vec<InitialCondition>,
    pub feed_streams: Vec<FeedStream>,
    pub integration_settings: IntegrationSettings,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum ReactorType {
    Batch,
    SemiBatch,
    Cstr,
    Pfr,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(tag = "type")]
pub enum TemperatureControl {
    Isothermal { temperature: f64 },
    TemperatureProfile { profile: Vec<(f64, f64)> },  // (time, temp)
    Jacket {
        jacket_temp: f64,
        ua_value: f64,  // Overall heat transfer coefficient × area, W/K
    },
    Adiabatic,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct InitialCondition {
    pub species_id: String,
    pub concentration: f64,  // mol/L
}

#[derive(Debug, Deserialize, Serialize)]
pub struct FeedStream {
    pub species_id: String,
    pub flow_rate: f64,  // L/min
    pub concentration: f64,  // mol/L
    pub start_time: f64,
    pub end_time: Option<f64>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct IntegrationSettings {
    pub t_end: f64,
    pub max_step: f64,
    pub atol: f64,
    pub rtol: f64,
}
```

---

## Solver Architecture Deep-Dive

### Governing Equations and Discretization

PharmaFlow's kinetic solver implements systems of **ordinary differential equations (ODEs)** representing species mass balances in reacting systems. For a batch reactor with `N` species and `R` reactions:

```
dC_i/dt = Σ_{j=1}^R ν_{ij} · r_j    for i = 1, ..., N

where:
- C_i: concentration of species i [mol/L]
- ν_{ij}: stoichiometric coefficient of species i in reaction j (negative for reactants, positive for products)
- r_j: reaction rate for reaction j [mol/(L·s)]
```

For **power-law kinetics**:
```
r_j = k_j · Π_{i=1}^N C_i^{α_{ij}}

where:
- k_j: rate constant for reaction j
- α_{ij}: reaction order of species i in reaction j
```

For **Arrhenius temperature dependence**:
```
k_j(T) = A_j · exp(-E_aj / (R·T))

where:
- A_j: pre-exponential factor
- E_aj: activation energy [kJ/mol]
- R: universal gas constant = 8.314 J/(mol·K)
- T: absolute temperature [K]
```

**Energy balance** for batch reactor (with heat transfer):
```
ρ·V·Cp · dT/dt = Σ_{j=1}^R (-ΔH_rxn,j) · r_j · V - UA·(T - T_jacket)

where:
- ρ: density [kg/L]
- V: reactor volume [L]
- Cp: heat capacity [kJ/(kg·K)]
- ΔH_rxn,j: heat of reaction j [kJ/mol], negative for exothermic
- UA: overall heat transfer coefficient × area [W/K]
- T_jacket: jacket temperature [K]
```

For **semi-batch reactors** with feed streams:
```
dC_i/dt = Σ_{j=1}^R ν_{ij} · r_j + Σ_{k=1}^F (F_k/V) · (C_{i,k} - C_i)

dV/dt = Σ_{k=1}^F F_k

where:
- F_k: volumetric flow rate of feed stream k [L/s]
- C_{i,k}: concentration of species i in feed stream k [mol/L]
```

### Parameter Estimation — Nonlinear Least Squares

Given experimental data `{(t_n, C_obs,i,n)}` for `N_obs` observations and `N_species` species, parameter estimation minimizes the residual sum of squares:

```
min_θ  SSR = Σ_{n=1}^{N_obs} Σ_{i=1}^{N_species} w_i · (C_obs,i,n - C_model,i(t_n; θ))²

where:
- θ: parameter vector (rate constants, activation energies, etc.)
- w_i: weight for species i (typically 1/σ_i² for weighted least squares)
- C_model,i(t_n; θ): model prediction at time t_n with parameters θ
```

We use the **Levenberg-Marquardt algorithm**, which interpolates between gradient descent and Gauss-Newton:

```
θ_{k+1} = θ_k - (J^T J + λ diag(J^T J))^{-1} · J^T · r

where:
- J: Jacobian matrix ∂r/∂θ (computed via forward finite differences)
- r: residual vector
- λ: damping parameter (adjusted adaptively)
```

**Convergence criteria**:
- Relative change in parameters: ||θ_{k+1} - θ_k|| / ||θ_k|| < 1e-6
- Relative change in objective: |SSR_{k+1} - SSR_k| / SSR_k < 1e-8
- Gradient norm: ||∇SSR|| < 1e-8

**Uncertainty quantification** (frequentist):
```
Covariance matrix: Σ_θ ≈ σ²·(J^T J)^{-1}

Standard errors: SE(θ_i) = √(Σ_θ[i,i])

95% confidence intervals: θ_i ± 1.96·SE(θ_i)

where σ² = SSR / (N_obs·N_species - N_params)
```

### ODE Integration — LSODA with Automatic Stiffness Detection

The LSODA (Livermore Solver for Ordinary Differential Equations with Automatic stiffness detection) algorithm switches between:

**Adams-Moulton** (non-stiff, explicit):
```
y_{n+1} = y_n + h·Σ_{i=0}^k β_i·f(t_{n-i}, y_{n-i})

Adaptive order 1-12, typical for non-stiff kinetics
```

**BDF (Backward Differentiation Formula)** (stiff, implicit):
```
Σ_{i=0}^k α_i·y_{n+1-i} = h·β·f(t_{n+1}, y_{n+1})

Adaptive order 1-5, used for stiff systems (fast and slow reactions)
Requires Newton iteration: y_{n+1}^{(m+1)} = y_{n+1}^{(m)} - [I - h·β·J]^{-1}·g
```

**Stiffness detection** via ratio of spectral radii:
```
If max(|eigenvalues(J)|) / min(|eigenvalues(J)|) > 1000 → switch to BDF
```

**Adaptive timestep** via local error estimation:
```
LTE = ||y_{n+1,k} - y_{n+1,k-1}|| / ||y_{n+1,k}||

h_new = 0.9 · h · (RTOL / LTE)^{1/(k+1)}  clamped to [h_min, h_max]
```

### Client/Server Split (WASM Threshold)

```
Dataset uploaded → Row count × Species count extracted
    │
    ├── ≤1000 data points → WASM solver (browser)
    │   ├── Instant parameter fitting, no network latency
    │   ├── Results displayed immediately with residual plots
    │   └── No server cost
    │
    └── >1000 data points OR Bayesian MCMC → Server solver
        ├── Job queued via Redis
        ├── Worker picks up, runs native solver or PyMC3
        ├── Progress streamed via WebSocket
        └── Results stored in S3, URL returned
```

The 1000-point threshold covers:
- Typical kinetic study: 20-50 timepoints × 5-20 species = 100-1000 points
- High-throughput screening: 96-well plate with 10 timepoints × 3 species = 2880 points → server
- WASM fitting completes in <5 seconds for 1000 points on modern hardware

### WASM Compilation Pipeline

```toml
# solver-wasm/Cargo.toml
[package]
name = "pharmaflow-solver-wasm"
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
      - run: wasm-opt -Oz solver-wasm/pkg/pharmaflow_solver_wasm_bg.wasm -o solver-wasm/pkg/pharmaflow_solver_wasm_bg.wasm
      - name: Upload to S3/CDN
        run: aws s3 cp solver-wasm/pkg/ s3://pharmaflow-wasm/v${{ github.sha }}/ --recursive
```

---

## Architecture Deep-Dives

### 1. Parameter Estimation API Handler (Rust/Axum)

The primary endpoint receives experimental data, validates the reaction scheme, decides between WASM and server execution, and for server execution enqueues a parameter estimation job with Redis and streams convergence progress via WebSocket.

```rust
// src/api/handlers/parameter_estimation.rs

use axum::{
    extract::{Path, State, Json},
    http::StatusCode,
    response::IntoResponse,
};
use uuid::Uuid;
use crate::{
    db::models::{KineticModel, Dataset, ReactionScheme, RateLaw},
    state::AppState,
    auth::Claims,
    error::ApiError,
};

#[derive(serde::Deserialize)]
pub struct CreateEstimationRequest {
    pub dataset_id: Uuid,
    pub rate_laws: Vec<RateLaw>,
    pub estimation_method: EstimationMethod,
    pub options: EstimationOptions,
}

#[derive(serde::Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum EstimationMethod {
    Nlls,  // Nonlinear least squares
    Mcmc,  // Bayesian MCMC (server only)
}

#[derive(serde::Deserialize)]
pub struct EstimationOptions {
    pub max_iterations: Option<u32>,
    pub convergence_tol: Option<f64>,
    pub mcmc_samples: Option<u32>,
}

pub async fn create_estimation(
    State(state): State<AppState>,
    claims: Claims,
    Path(project_id): Path<Uuid>,
    Json(req): Json<CreateEstimationRequest>,
) -> Result<impl IntoResponse, ApiError> {
    // 1. Verify project ownership and dataset
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

    let dataset = sqlx::query_as!(
        Dataset,
        "SELECT * FROM datasets WHERE id = $1 AND project_id = $2",
        req.dataset_id,
        project_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Dataset not found"))?;

    // 2. Parse dataset summary to determine size
    let data_summary: serde_json::Value = dataset.data_summary;
    let num_rows = data_summary["num_rows"].as_i64().unwrap_or(0);
    let num_species = data_summary["num_species"].as_i64().unwrap_or(0);
    let data_points = num_rows * num_species;

    // 3. Check plan limits
    let user = sqlx::query_as!(
        crate::db::models::User,
        "SELECT * FROM users WHERE id = $1",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    if user.plan == "free" && data_points > 500 {
        return Err(ApiError::PlanLimit(
            "Free plan supports datasets up to 500 data points. Upgrade to Academic for unlimited."
        ));
    }

    if matches!(req.estimation_method, EstimationMethod::Mcmc) && user.plan == "free" {
        return Err(ApiError::PlanLimit(
            "Bayesian MCMC estimation requires Academic plan or higher."
        ));
    }

    // 4. Determine execution mode
    let execution_mode = if data_points <= 1000 && matches!(req.estimation_method, EstimationMethod::Nlls) {
        "wasm"
    } else {
        "server"
    };

    // 5. Create kinetic model record
    let model = sqlx::query_as!(
        KineticModel,
        r#"INSERT INTO kinetic_models
            (project_id, dataset_id, user_id, name, rate_laws,
             fitted_parameters, estimation_method, goodness_of_fit, status)
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9)
        RETURNING *"#,
        project_id,
        req.dataset_id,
        claims.user_id,
        format!("Model for {}", dataset.name),
        serde_json::to_value(&req.rate_laws)?,
        serde_json::json!({}),  // Empty until fitting completes
        serde_json::to_string(&req.estimation_method)?,
        serde_json::json!({}),  // Empty until fitting completes
        if execution_mode == "wasm" { "draft" } else { "pending" },
    )
    .fetch_one(&state.db)
    .await?;

    // 6. For server execution, enqueue job
    if execution_mode == "server" {
        let job = sqlx::query!(
            r#"INSERT INTO computation_jobs
                (job_type, reference_id, user_id, priority, cores_allocated, memory_mb)
            VALUES ($1, $2, $3, $4, $5, $6) RETURNING id"#,
            "parameter_estimation",
            model.id,
            claims.user_id,
            if user.plan == "enterprise" { 10 } else if user.plan == "pro" { 5 } else { 0 },
            if matches!(req.estimation_method, EstimationMethod::Mcmc) { 4 } else { 1 },
            if matches!(req.estimation_method, EstimationMethod::Mcmc) { 8192 } else { 4096 },
        )
        .fetch_one(&state.db)
        .await?;

        // Publish to Redis job queue
        state.redis
            .publish("computation:jobs", serde_json::to_string(&job.id)?)
            .await?;
    }

    Ok((StatusCode::CREATED, Json(model)))
}

pub async fn get_estimation_status(
    State(state): State<AppState>,
    claims: Claims,
    Path((project_id, model_id)): Path<(Uuid, Uuid)>,
) -> Result<Json<KineticModel>, ApiError> {
    let model = sqlx::query_as!(
        KineticModel,
        "SELECT m.* FROM kinetic_models m
         JOIN projects p ON m.project_id = p.id
         WHERE m.id = $1 AND m.project_id = $2
         AND (p.owner_id = $3 OR p.org_id IN (
             SELECT org_id FROM org_members WHERE user_id = $3
         ))",
        model_id,
        project_id,
        claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Kinetic model not found"))?;

    Ok(Json(model))
}
```

### 2. ODE Solver Core (Rust — shared between WASM and native)

The core LSODA solver that integrates the reaction kinetics ODEs. This code compiles to both native (server) and WASM (browser) targets.

```rust
// solver-core/src/ode_solver.rs

use nalgebra::{DVector, DMatrix};
use crate::kinetics::{ReactionSystem, StateVector};

pub struct LsodaSolver {
    pub system: ReactionSystem,
    pub state: StateVector,
    pub time: f64,
    pub step_size: f64,
    pub method: IntegrationMethod,
    pub order: usize,
    pub atol: f64,
    pub rtol: f64,
    pub max_steps: usize,
    pub stiffness_detector: StiffnessDetector,
}

#[derive(Debug, Clone, Copy, PartialEq)]
pub enum IntegrationMethod {
    AdamsMoulton,  // Non-stiff, explicit
    Bdf,           // Stiff, implicit
}

impl LsodaSolver {
    pub fn new(system: ReactionSystem, initial_state: StateVector) -> Self {
        Self {
            system,
            state: initial_state,
            time: 0.0,
            step_size: 1e-4,
            method: IntegrationMethod::AdamsMoulton,
            order: 2,
            atol: 1e-8,
            rtol: 1e-6,
            max_steps: 100000,
            stiffness_detector: StiffnessDetector::new(),
        }
    }

    /// Integrate from t to t_end, returning time-series data
    pub fn integrate_to(&mut self, t_end: f64) -> Result<IntegrationResult, SolverError> {
        let mut result = IntegrationResult::new();
        result.push(self.time, self.state.clone());

        let mut steps = 0;
        while self.time < t_end && steps < self.max_steps {
            // Adjust step size to not overshoot t_end
            let h = self.step_size.min(t_end - self.time);

            // Perform one integration step
            self.step(h)?;

            // Check for stiffness and switch method if needed
            if self.stiffness_detector.is_stiff(&self.system, &self.state) {
                if self.method == IntegrationMethod::AdamsMoulton {
                    self.method = IntegrationMethod::Bdf;
                    self.order = 2;  // Reset order when switching
                }
            }

            result.push(self.time, self.state.clone());
            steps += 1;
        }

        if steps >= self.max_steps {
            return Err(SolverError::MaxStepsExceeded);
        }

        Ok(result)
    }

    /// Perform one integration step using current method
    fn step(&mut self, h: f64) -> Result<(), SolverError> {
        match self.method {
            IntegrationMethod::AdamsMoulton => self.step_adams(h),
            IntegrationMethod::Bdf => self.step_bdf(h),
        }
    }

    /// Adams-Moulton step (explicit)
    fn step_adams(&mut self, h: f64) -> Result<(), SolverError> {
        // Predictor: Adams-Bashforth
        let f0 = self.system.compute_derivatives(&self.state, self.time);
        let y_pred = &self.state + &(f0.clone() * h);

        // Corrector: Adams-Moulton (implicit, but solved via predictor-corrector)
        let f1 = self.system.compute_derivatives(&y_pred, self.time + h);
        let y_corr = &self.state + &((f0 + f1) * (h / 2.0));

        // Local error estimate
        let error = (&y_corr - &y_pred).norm() / (1.0 + y_corr.norm());

        if error > self.rtol {
            // Reject step, reduce step size
            self.step_size *= 0.5;
            return Ok(());
        }

        // Accept step
        self.state = y_corr;
        self.time += h;

        // Adapt step size
        if error < self.rtol / 10.0 {
            self.step_size *= 1.5;
        }

        Ok(())
    }

    /// BDF step (implicit, requires Newton iteration)
    fn step_bdf(&mut self, h: f64) -> Result<(), SolverError> {
        // BDF2: (3/2)y_{n+1} - 2y_n + (1/2)y_{n-1} = h·f(t_{n+1}, y_{n+1})
        // Rearrange: y_{n+1} - (2h/3)·f(t_{n+1}, y_{n+1}) = (4/3)y_n - (1/3)y_{n-1}

        let y_n = self.state.clone();
        let rhs = &y_n * (4.0 / 3.0);  // Simplified for BDF1 initially

        // Newton iteration: solve g(y) = y - (2h/3)·f(t+h, y) - rhs = 0
        let mut y_new = y_n.clone();
        for _iter in 0..10 {
            let f = self.system.compute_derivatives(&y_new, self.time + h);
            let g = &y_new - &(f.clone() * (2.0 * h / 3.0)) - &rhs;

            // Jacobian: J = I - (2h/3)·∂f/∂y
            let jac = self.system.compute_jacobian(&y_new, self.time + h);
            let jac_mod = DMatrix::identity(y_new.len(), y_new.len()) - jac * (2.0 * h / 3.0);

            // Solve: Δy = -J^{-1}·g
            let delta_y = jac_mod.try_inverse()
                .ok_or(SolverError::SingularJacobian)?
                * g;

            y_new = y_new - delta_y;

            if delta_y.norm() / (1.0 + y_new.norm()) < self.atol {
                break;
            }
        }

        self.state = y_new;
        self.time += h;

        Ok(())
    }
}

pub struct StateVector {
    pub concentrations: DVector<f64>,  // [C_1, C_2, ..., C_N]
    pub temperature: f64,  // K
    pub volume: f64,  // L
}

impl StateVector {
    pub fn len(&self) -> usize {
        self.concentrations.len() + 2  // + T + V
    }

    pub fn as_dvector(&self) -> DVector<f64> {
        let mut vec = DVector::zeros(self.len());
        vec.rows_mut(0, self.concentrations.len()).copy_from(&self.concentrations);
        vec[self.concentrations.len()] = self.temperature;
        vec[self.concentrations.len() + 1] = self.volume;
        vec
    }
}

pub struct IntegrationResult {
    pub times: Vec<f64>,
    pub states: Vec<StateVector>,
}

impl IntegrationResult {
    pub fn new() -> Self {
        Self {
            times: Vec::new(),
            states: Vec::new(),
        }
    }

    pub fn push(&mut self, time: f64, state: StateVector) {
        self.times.push(time);
        self.states.push(state);
    }
}

struct StiffnessDetector {
    last_eigenvalue_ratio: f64,
}

impl StiffnessDetector {
    fn new() -> Self {
        Self {
            last_eigenvalue_ratio: 1.0,
        }
    }

    fn is_stiff(&mut self, system: &ReactionSystem, state: &StateVector) -> bool {
        // Estimate eigenvalue ratio of Jacobian
        // Simplified heuristic: if max(|λ|) / min(|λ|) > 1000, system is stiff
        let jac = system.compute_jacobian(state, 0.0);

        // Use power iteration to estimate dominant eigenvalue (expensive, done infrequently)
        // For now, use a simpler heuristic based on reaction rate spread
        let rates = system.compute_reaction_rates(state, 0.0);
        if rates.is_empty() {
            return false;
        }

        let max_rate = rates.iter().map(|r| r.abs()).fold(0.0, f64::max);
        let min_rate = rates.iter().map(|r| r.abs()).filter(|r| *r > 1e-12).fold(f64::INFINITY, f64::min);

        if min_rate.is_infinite() {
            return false;
        }

        self.last_eigenvalue_ratio = max_rate / min_rate;
        self.last_eigenvalue_ratio > 1000.0
    }
}

#[derive(Debug)]
pub enum SolverError {
    MaxStepsExceeded,
    SingularJacobian,
    InvalidState,
}
```

### 3. Parameter Estimation Core (Rust — Levenberg-Marquardt)

The nonlinear least-squares solver that fits kinetic parameters to experimental data.

```rust
// solver-core/src/parameter_estimation.rs

use nalgebra::{DVector, DMatrix};
use crate::ode_solver::{LsodaSolver, StateVector};
use crate::kinetics::ReactionSystem;

pub struct ParameterEstimator {
    pub system: ReactionSystem,
    pub experimental_data: ExperimentalData,
    pub parameters: Vec<Parameter>,
    pub options: EstimationOptions,
}

pub struct ExperimentalData {
    pub times: Vec<f64>,
    pub concentrations: Vec<Vec<f64>>,  // [timepoint][species]
    pub species_names: Vec<String>,
    pub weights: Vec<f64>,  // Per-species weights
}

pub struct Parameter {
    pub name: String,
    pub value: f64,
    pub bounds: (f64, f64),
    pub fixed: bool,
}

pub struct EstimationOptions {
    pub max_iterations: usize,
    pub convergence_tol: f64,
    pub initial_lambda: f64,  // LM damping parameter
}

impl Default for EstimationOptions {
    fn default() -> Self {
        Self {
            max_iterations: 100,
            convergence_tol: 1e-6,
            initial_lambda: 1e-3,
        }
    }
}

pub struct EstimationResult {
    pub parameters: Vec<Parameter>,
    pub covariance: DMatrix<f64>,
    pub ssr: f64,  // Sum of squared residuals
    pub r_squared: f64,
    pub rmse: f64,
    pub aic: f64,
    pub bic: f64,
    pub residuals: Vec<Vec<f64>>,
    pub iterations: usize,
    pub converged: bool,
}

impl ParameterEstimator {
    pub fn estimate(&mut self) -> Result<EstimationResult, EstimationError> {
        let mut theta = self.get_parameter_vector();
        let mut lambda = self.options.initial_lambda;
        let mut ssr = self.compute_ssr(&theta)?;

        let mut iterations = 0;
        let mut converged = false;

        while iterations < self.options.max_iterations {
            // Compute Jacobian via forward finite differences
            let jacobian = self.compute_jacobian(&theta)?;
            let residuals = self.compute_residuals(&theta)?;

            // Levenberg-Marquardt update: (J^T·J + λ·diag(J^T·J))·Δθ = J^T·r
            let jtj = jacobian.transpose() * &jacobian;
            let jtr = jacobian.transpose() * &residuals;

            let damping = DMatrix::from_diagonal(&jtj.diagonal()) * lambda;
            let lhs = jtj + damping;

            let delta_theta = lhs.try_inverse()
                .ok_or(EstimationError::SingularMatrix)?
                * jtr;

            // Try update
            let theta_new = &theta - &delta_theta;

            // Check bounds
            if !self.check_bounds(&theta_new) {
                lambda *= 10.0;
                continue;
            }

            let ssr_new = self.compute_ssr(&theta_new)?;

            if ssr_new < ssr {
                // Accept update
                let relative_change = (ssr - ssr_new).abs() / ssr;
                theta = theta_new;
                ssr = ssr_new;
                lambda /= 10.0;

                if relative_change < self.options.convergence_tol {
                    converged = true;
                    break;
                }
            } else {
                // Reject update, increase damping
                lambda *= 10.0;
            }

            iterations += 1;
        }

        // Compute final statistics
        let residuals = self.compute_residuals(&theta)?;
        let jacobian = self.compute_jacobian(&theta)?;

        let n_obs = self.experimental_data.times.len() * self.experimental_data.species_names.len();
        let n_params = theta.len();
        let sigma_sq = ssr / (n_obs - n_params) as f64;

        let jtj = jacobian.transpose() * jacobian;
        let covariance = jtj.try_inverse()
            .ok_or(EstimationError::SingularMatrix)?
            * sigma_sq;

        // Update parameter values and compute confidence intervals
        self.update_parameters(&theta);

        let r_squared = self.compute_r_squared(&residuals)?;
        let rmse = (ssr / n_obs as f64).sqrt();
        let aic = n_obs as f64 * (ssr / n_obs as f64).ln() + 2.0 * n_params as f64;
        let bic = n_obs as f64 * (ssr / n_obs as f64).ln() + (n_params as f64) * (n_obs as f64).ln();

        Ok(EstimationResult {
            parameters: self.parameters.clone(),
            covariance,
            ssr,
            r_squared,
            rmse,
            aic,
            bic,
            residuals: self.residuals_to_2d(&residuals),
            iterations,
            converged,
        })
    }

    fn compute_ssr(&self, theta: &DVector<f64>) -> Result<f64, EstimationError> {
        let residuals = self.compute_residuals(theta)?;
        Ok(residuals.dot(&residuals))
    }

    fn compute_residuals(&self, theta: &DVector<f64>) -> Result<DVector<f64>, EstimationError> {
        // Update system with trial parameters
        let mut system = self.system.clone();
        system.update_parameters(theta);

        // Simulate
        let initial_state = self.get_initial_state();
        let mut solver = LsodaSolver::new(system, initial_state);

        let mut residuals = Vec::new();
        for (i, &t) in self.experimental_data.times.iter().enumerate() {
            let result = solver.integrate_to(t)?;
            let state = result.states.last().unwrap();

            for (j, &c_obs) in self.experimental_data.concentrations[i].iter().enumerate() {
                let c_model = state.concentrations[j];
                let weight = self.experimental_data.weights[j];
                residuals.push(weight.sqrt() * (c_obs - c_model));
            }
        }

        Ok(DVector::from_vec(residuals))
    }

    fn compute_jacobian(&self, theta: &DVector<f64>) -> Result<DMatrix<f64>, EstimationError> {
        let n_residuals = self.experimental_data.times.len() * self.experimental_data.species_names.len();
        let n_params = theta.len();
        let mut jacobian = DMatrix::zeros(n_residuals, n_params);

        let r0 = self.compute_residuals(theta)?;
        let epsilon = 1e-7;

        for j in 0..n_params {
            let mut theta_perturbed = theta.clone();
            theta_perturbed[j] += epsilon * theta[j].abs().max(1.0);

            let r_perturbed = self.compute_residuals(&theta_perturbed)?;
            let dr = (r_perturbed - &r0) / (epsilon * theta[j].abs().max(1.0));

            jacobian.set_column(j, &dr);
        }

        Ok(jacobian)
    }

    fn compute_r_squared(&self, residuals: &DVector<f64>) -> Result<f64, EstimationError> {
        let ssr = residuals.dot(residuals);

        // Compute total sum of squares
        let mut y_mean = 0.0;
        let n = self.experimental_data.concentrations.iter().map(|v| v.len()).sum::<usize>();
        for concs in &self.experimental_data.concentrations {
            for &c in concs {
                y_mean += c;
            }
        }
        y_mean /= n as f64;

        let mut sst = 0.0;
        for concs in &self.experimental_data.concentrations {
            for &c in concs {
                sst += (c - y_mean).powi(2);
            }
        }

        Ok(1.0 - ssr / sst)
    }

    fn get_parameter_vector(&self) -> DVector<f64> {
        DVector::from_vec(
            self.parameters.iter()
                .filter(|p| !p.fixed)
                .map(|p| p.value)
                .collect()
        )
    }

    fn check_bounds(&self, theta: &DVector<f64>) -> bool {
        self.parameters.iter()
            .filter(|p| !p.fixed)
            .zip(theta.iter())
            .all(|(param, &val)| val >= param.bounds.0 && val <= param.bounds.1)
    }

    fn update_parameters(&mut self, theta: &DVector<f64>) {
        let mut idx = 0;
        for param in &mut self.parameters {
            if !param.fixed {
                param.value = theta[idx];
                idx += 1;
            }
        }
    }

    fn get_initial_state(&self) -> StateVector {
        // Extract initial concentrations from experimental data
        let initial_concs = DVector::from_vec(self.experimental_data.concentrations[0].clone());
        StateVector {
            concentrations: initial_concs,
            temperature: 298.15,  // Default to 25°C, should come from metadata
            volume: 1.0,  // Default 1L for batch reactor
        }
    }

    fn residuals_to_2d(&self, residuals: &DVector<f64>) -> Vec<Vec<f64>> {
        let n_species = self.experimental_data.species_names.len();
        residuals.as_slice()
            .chunks(n_species)
            .map(|chunk| chunk.to_vec())
            .collect()
    }
}

#[derive(Debug)]
pub enum EstimationError {
    SingularMatrix,
    IntegrationFailed,
    InvalidData,
}
```

### 4. Scale-Up Correlation Engine

Predicts reactor performance at manufacturing scale from lab-scale data using engineering correlations.

```rust
// src/scaleup/correlations.rs

use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ReactorGeometry {
    pub volume: f64,  // L
    pub diameter: f64,  // m
    pub height: f64,  // m
    pub impeller_diameter: f64,  // m
    pub impeller_type: String,  // "pitched_blade" | "rushton" | "anchor"
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OperatingConditions {
    pub rpm: f64,
    pub temperature: f64,  // K
    pub viscosity: f64,  // Pa·s
    pub density: f64,  // kg/m³
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum ScaleUpMethod {
    ConstantTipSpeed,
    ConstantPowerPerVolume,
    ConstantMixingTime,
    ConstantReynolds,
}

pub struct ScaleUpCalculator {
    pub base_geometry: ReactorGeometry,
    pub base_conditions: OperatingConditions,
    pub method: ScaleUpMethod,
}

impl ScaleUpCalculator {
    pub fn predict_conditions(
        &self,
        target_geometry: &ReactorGeometry,
    ) -> OperatingConditions {
        match self.method {
            ScaleUpMethod::ConstantTipSpeed => {
                // Tip speed: v = π·D·N
                // v_base = v_target → N_target = N_base · (D_base / D_target)
                let rpm_target = self.base_conditions.rpm
                    * (self.base_geometry.impeller_diameter / target_geometry.impeller_diameter);

                OperatingConditions {
                    rpm: rpm_target,
                    ..self.base_conditions.clone()
                }
            },
            ScaleUpMethod::ConstantPowerPerVolume => {
                // Power number: Np = P / (ρ·N³·D⁵)
                // P/V = Np·ρ·N³·D⁵ / V
                // For geometric similarity: D ∝ V^(1/3)
                // Constant P/V: N_target = N_base · (V_base / V_target)^(1/3) · (D_target / D_base)^(-5/3)

                let volume_ratio = self.base_geometry.volume / target_geometry.volume;
                let diameter_ratio = target_geometry.impeller_diameter / self.base_geometry.impeller_diameter;

                let rpm_target = self.base_conditions.rpm
                    * volume_ratio.powf(1.0 / 3.0)
                    * diameter_ratio.powf(-5.0 / 3.0);

                OperatingConditions {
                    rpm: rpm_target,
                    ..self.base_conditions.clone()
                }
            },
            ScaleUpMethod::ConstantMixingTime => {
                // Mixing time correlation: θ_m = C · N^(-1) · D^(-2/3) · ν^(1/6)
                // Constant θ_m: N_target = N_base · (D_base / D_target)^(2/3)

                let diameter_ratio = self.base_geometry.impeller_diameter / target_geometry.impeller_diameter;
                let rpm_target = self.base_conditions.rpm * diameter_ratio.powf(2.0 / 3.0);

                OperatingConditions {
                    rpm: rpm_target,
                    ..self.base_conditions.clone()
                }
            },
            ScaleUpMethod::ConstantReynolds => {
                // Reynolds number: Re = ρ·N·D² / μ
                // Constant Re: N_target = N_base · (D_base / D_target)²

                let diameter_ratio = self.base_geometry.impeller_diameter / target_geometry.impeller_diameter;
                let rpm_target = self.base_conditions.rpm * diameter_ratio.powi(2);

                OperatingConditions {
                    rpm: rpm_target,
                    ..self.base_conditions.clone()
                }
            },
        }
    }

    pub fn predict_heat_transfer(&self, target_geometry: &ReactorGeometry) -> f64 {
        // Overall heat transfer coefficient × area: UA [W/K]
        // Correlation: Nu = C·Re^a·Pr^b (Dittus-Boelter for turbulent)
        // Nu = h·D / k_fluid

        let re_base = self.reynolds_number(&self.base_geometry, &self.base_conditions);
        let target_conditions = self.predict_conditions(target_geometry);
        let re_target = self.reynolds_number(target_geometry, &target_conditions);

        // Simplified: h ∝ Re^0.67 for turbulent
        let h_ratio = (re_target / re_base).powf(0.67);

        // Area scaling: A ∝ D²
        let area_ratio = (target_geometry.diameter / self.base_geometry.diameter).powi(2);

        // Assume base UA = 100 W/K for 1L reactor (typical jacketed flask)
        let base_ua = 100.0 * (self.base_geometry.volume / 1.0).powf(0.67);

        base_ua * h_ratio * area_ratio
    }

    fn reynolds_number(&self, geom: &ReactorGeometry, cond: &OperatingConditions) -> f64 {
        let n_rps = cond.rpm / 60.0;  // Convert to rev/s
        cond.density * n_rps * geom.impeller_diameter.powi(2) / cond.viscosity
    }
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–4)

**Day 1: Project initialization and Rust backend scaffold**
- `cargo init pharmaflow-api`, add dependencies: axum, tokio, serde, sqlx, uuid, chrono, aws-sdk-s3, redis
- `src/main.rs` — Axum app with CORS, tracing, graceful shutdown
- `src/config.rs` — Environment-based config (DATABASE_URL, JWT_SECRET, S3_BUCKET, REDIS_URL)
- `src/state.rs` — AppState with PgPool, Redis, S3Client
- `src/error.rs` — ApiError enum with IntoResponse
- `Dockerfile` and `docker-compose.yml` — PostgreSQL, Redis, MinIO

**Day 2: Database schema and migrations**
- `migrations/001_initial.sql` — All 12 tables defined above
- `src/db/models.rs` — All SQLx structs with FromRow derives
- Run `sqlx migrate run`, seed equipment database with 50+ reactor specs (Mettler Toledo, Eppendorf, Sartorius, etc.)

**Day 3: Authentication system**
- `src/auth/mod.rs` — JWT middleware with token validation
- `src/auth/oauth.rs` — Google and GitHub OAuth 2.0 flows
- `src/api/handlers/auth.rs` — Register, login, OAuth callback, refresh, me endpoints
- Password hashing with bcrypt (cost 12)

**Day 4: User and project CRUD**
- `src/api/handlers/users.rs` — Profile CRUD, account deletion
- `src/api/handlers/projects.rs` — Project CRUD, forking
- `src/api/handlers/orgs.rs` — Org management, member invites
- Integration tests for auth and authorization

### Phase 2 — Solver Core Prototype (Days 5–11)

**Day 5: ODE solver framework**
- `solver-core/` workspace member
- `solver-core/src/ode_solver.rs` — LSODA structure, IntegrationMethod enum
- Adams-Moulton predictor-corrector stub
- BDF stub with Newton iteration outline

**Day 6: Reaction system modeling**
- `solver-core/src/kinetics/mod.rs` — ReactionSystem struct
- `solver-core/src/kinetics/species.rs` — Species with properties
- `solver-core/src/kinetics/reactions.rs` — Reaction with stoichiometry, rate laws
- `compute_derivatives()` method for dC/dt calculation
- Unit tests: simple A → B first-order reaction

**Day 7: Adams-Moulton integration**
- Implement full Adams-Moulton predictor-corrector in `step_adams()`
- Adaptive timestep control via LTE estimation
- Test with non-stiff systems: exponential decay, oscillator

**Day 8: BDF integration for stiff systems**
- Implement BDF2 in `step_bdf()` with Newton iteration
- Jacobian computation via finite differences
- Test with stiff system: fast-slow reaction pair

**Day 9: Stiffness detection and switching**
- `StiffnessDetector` implementation using reaction rate ratios
- Automatic method switching during integration
- Test on moderately stiff kinetics

**Day 10: Energy balance integration**
- Extend StateVector to include temperature
- Add energy balance dT/dt to derivative computation
- Adiabatic and jacket temperature control modes
- Test: exothermic reaction with temperature rise

**Day 11: Semi-batch reactor support**
- Add feed streams to ReactorConfig
- Volume dynamics dV/dt
- Dilution terms in mass balance
- Test: semi-batch with feed addition

### Phase 3 — Parameter Estimation (Days 12–16)

**Day 12: Levenberg-Marquardt framework**
- `solver-core/src/parameter_estimation.rs` — ParameterEstimator struct
- `compute_ssr()` and `compute_residuals()` methods
- LM damping parameter adaptation
- Test with known parameters (recovery test)

**Day 13: Jacobian computation**
- `compute_jacobian()` via forward finite differences
- Parallel evaluation for multi-parameter systems
- Test sensitivity to perturbation size

**Day 14: Uncertainty quantification**
- Covariance matrix computation
- Standard errors and confidence intervals
- Correlation matrix output
- Test with synthetic noisy data

**Day 15: Goodness-of-fit statistics**
- R², RMSE, AIC, BIC calculation
- Residual plot data generation
- Parity plot data
- Test model discrimination (AIC comparison)

**Day 16: WASM compilation for parameter estimation**
- Compile parameter estimation to WASM
- `solver-wasm/src/lib.rs` — WASM bindings for estimate_parameters()
- Test in browser with <1000 point dataset
- Benchmark: 500 points, 3 species, 2 parameters → <3s

### Phase 4 — WASM Build + Frontend Foundations (Days 17–22)

**Day 17: WASM build pipeline**
- `solver-wasm/Cargo.toml` with wasm-bindgen
- GitHub Actions workflow for WASM build
- CloudFront distribution for WASM bundle delivery
- Version pinning and cache invalidation

**Day 18: React project setup**
- `npm create vite@latest frontend -- --template react-ts`
- Zustand stores for app state
- React Router for navigation
- Tailwind CSS setup
- API client with axios

**Day 19: Authentication UI**
- Login/register forms
- OAuth buttons for Google/GitHub
- Protected route wrapper
- Auth context provider
- Token refresh logic

**Day 20: Project dashboard**
- Project list view with search/filter
- Create project modal
- Project card with thumbnail
- Recent projects section
- Public projects gallery

**Day 21: Reaction scheme editor — SVG rendering**
- `components/ReactionSchemeEditor.tsx` — SVG canvas
- Species node rendering with drag-and-drop
- Reaction arrow rendering
- Selection and move interactions
- Zoom and pan controls

**Day 22: Reaction scheme editor — D3 force layout**
- Integrate D3-force for auto-layout
- Collision detection between nodes
- Edge bundling for reaction arrows
- Manual position override
- Save/load scheme state

### Phase 5 — Dataset Management + Visualization (Days 23–27)

**Day 23: CSV upload and parsing**
- `components/DatasetUpload.tsx` — drag-and-drop file upload
- CSV parsing with Papa Parse
- Unit detection and conversion (mM, M, g/L, wt%)
- Data summary computation
- S3 upload with progress

**Day 24: Dataset preview**
- Table view of uploaded data
- Column mapping UI (time, species concentrations)
- Statistical summary (min, max, mean, std)
- Data validation and error highlighting
- Edit/delete dataset

**Day 25: D3.js visualization setup**
- `components/ResultsChart.tsx` — base D3 chart component
- Axes with pan/zoom via d3-zoom
- Crosshair and tooltip
- Legend with toggle visibility
- Export to PNG/SVG

**Day 26: Concentration-time plots**
- Multi-series line plots
- Experimental data as scatter points
- Model predictions as lines
- Residuals as separate pane
- Auto-scaling axes

**Day 27: Sensitivity Tornado charts**
- Horizontal bar chart for parameter sensitivities
- Color coding by impact direction
- Interactive hover for exact values
- Sort by absolute magnitude
- Export data table

### Phase 6 — Parameter Estimation Integration (Days 28–32)

**Day 28: WASM solver integration**
- Load WASM module in React
- `useWasmSolver` hook
- Call estimate_parameters from browser
- Progress callback for iterations
- Error handling and retry

**Day 29: Parameter estimation UI**
- Rate law selector (power law, Arrhenius, MM)
- Parameter bounds input
- Initial guess specification
- Estimation method selector (NLLS/MCMC)
- Start estimation button

**Day 30: Real-time convergence monitoring**
- Iteration count display
- SSR vs iteration plot
- Parameter trajectory plot
- Convergence indicators
- Cancel button

**Day 31: Results display**
- Fitted parameters table with std errors
- 95% CI display
- Correlation matrix heatmap
- Goodness-of-fit metrics
- Residual plots (time, normal Q-Q, histogram)

**Day 32: Server-side parameter estimation**
- Implement server-side job worker
- WebSocket progress streaming
- Database job status tracking
- S3 results storage
- Client polling for completion

### Phase 7 — Reactor Simulation + Scale-Up (Days 33–38)

**Day 33: Reactor configuration UI**
- Reactor type selector
- Volume, temperature, initial concentration inputs
- Feed stream configuration for semi-batch
- Jacket temperature control settings
- Integration tolerances

**Day 34: Reactor simulation execution**
- WASM/server routing logic
- Simulation API integration
- Progress monitoring
- Result fetching from S3
- Cache simulation results

**Day 35: Simulation results visualization**
- Concentration vs time for all species
- Temperature trajectory
- Volume trajectory for semi-batch
- Conversion and yield calculations
- Selectivity analysis

**Day 36: Scale-up correlation engine**
- Implement scale-up methods in Rust
- Equipment database integration
- Geometry selector UI
- Correlation method selector
- Predicted conditions display

**Day 37: Scale-up analysis UI**
- Base simulation selector
- Target scale input (10L, 100L, 1000L, 10000L)
- Side-by-side comparison chart
- RPM, UA, mixing time predictions
- Confidence intervals from parameter uncertainty

**Day 38: Three.js 3D reactor visualization**
- `components/ReactorViewer3D.tsx` — Three.js scene
- Reactor vessel geometry
- Impeller model
- Jacket visualization
- Rotate/zoom controls
- Optional feature (can be toggled off)

### Phase 8 — Reporting + Billing + Polish (Days 39–42)

**Day 39: PDF report generation**
- Python FastAPI service for PDF generation
- LaTeX template with kinetic model section
- Parameter table, plots embedded
- Reactor simulation results
- Scale-up recommendations
- S3 storage of generated PDFs

**Day 40: Stripe integration**
- Stripe Checkout Sessions for subscription
- Customer Portal for subscription management
- Webhook handler for subscription events
- Plan limit enforcement middleware
- Usage tracking and metering

**Day 41: Polish and bug fixes**
- Responsive design tweaks
- Loading states and skeletons
- Error boundaries
- Toast notifications
- Keyboard shortcuts
- Accessibility improvements

**Day 42: Final testing and deployment**
- End-to-end tests with Playwright
- Load testing with k6
- Production deployment to AWS (ECS + RDS + ElastiCache)
- Monitoring setup (Prometheus, Grafana, Sentry)
- Documentation and onboarding flow

---

## Critical Files

```
pharmaflow/
├── backend/
│   ├── src/
│   │   ├── main.rs                          # Axum app entry point
│   │   ├── config.rs                        # Environment config
│   │   ├── state.rs                         # AppState with DB, Redis, S3
│   │   ├── error.rs                         # ApiError enum
│   │   ├── auth/
│   │   │   ├── mod.rs                       # JWT middleware
│   │   │   └── oauth.rs                     # OAuth flows
│   │   ├── api/
│   │   │   ├── router.rs                    # Route definitions
│   │   │   └── handlers/
│   │   │       ├── auth.rs                  # Auth endpoints
│   │   │       ├── projects.rs              # Project CRUD
│   │   │       ├── datasets.rs              # Dataset upload/management
│   │   │       ├── parameter_estimation.rs  # Parameter estimation API (300 lines)
│   │   │       ├── reactor_simulations.rs   # Reactor simulation API
│   │   │       └── scaleup.rs               # Scale-up analysis API
│   │   ├── db/
│   │   │   ├── mod.rs                       # DB pool init
│   │   │   └── models.rs                    # SQLx structs (500 lines)
│   │   ├── scaleup/
│   │   │   └── correlations.rs              # Scale-up engine (150 lines)
│   │   └── workers/
│   │       └── computation_worker.rs        # Background job worker (200 lines)
│   ├── migrations/
│   │   └── 001_initial.sql                  # Database schema (300 lines)
│   ├── Cargo.toml
│   └── Dockerfile
├── solver-core/
│   ├── src/
│   │   ├── lib.rs                           # Module exports
│   │   ├── ode_solver.rs                    # LSODA implementation (350 lines)
│   │   ├── parameter_estimation.rs          # Levenberg-Marquardt (400 lines)
│   │   ├── kinetics/
│   │   │   ├── mod.rs                       # ReactionSystem
│   │   │   ├── species.rs                   # Species struct
│   │   │   └── reactions.rs                 # Reaction, RateLaw enums
│   │   └── utils.rs                         # Helper functions
│   └── Cargo.toml
├── solver-wasm/
│   ├── src/
│   │   └── lib.rs                           # WASM bindings (200 lines)
│   └── Cargo.toml
├── python-service/
│   ├── main.py                              # FastAPI app
│   ├── bayesian_estimation.py              # PyMC3 MCMC
│   └── pdf_generator.py                    # LaTeX PDF generation
├── frontend/
│   ├── src/
│   │   ├── main.tsx                         # React entry point
│   │   ├── App.tsx                          # Root component with routing
│   │   ├── api/
│   │   │   └── client.ts                    # Axios API client
│   │   ├── components/
│   │   │   ├── ReactionSchemeEditor.tsx     # SVG reaction scheme (400 lines)
│   │   │   ├── DatasetUpload.tsx            # CSV upload
│   │   │   ├── ParameterEstimationUI.tsx    # Parameter estimation UI (300 lines)
│   │   │   ├── ResultsChart.tsx             # D3.js base chart (250 lines)
│   │   │   ├── ReactorConfigUI.tsx          # Reactor configuration form
│   │   │   ├── ScaleUpAnalysis.tsx          # Scale-up UI
│   │   │   └── ReactorViewer3D.tsx          # Three.js 3D viewer (200 lines)
│   │   ├── hooks/
│   │   │   ├── useWasmSolver.ts             # WASM solver hook
│   │   │   └── useWebSocket.ts              # WebSocket hook for progress
│   │   ├── stores/
│   │   │   ├── authStore.ts                 # Zustand auth state
│   │   │   ├── projectStore.ts              # Project state
│   │   │   └── simulationStore.ts           # Simulation state
│   │   └── utils/
│   │       ├── units.ts                     # Unit conversion
│   │       └── formatting.ts                # Number formatting
│   ├── package.json
│   └── vite.config.ts
├── .github/
│   └── workflows/
│       ├── wasm-build.yml                   # WASM CI pipeline
│       ├── backend-tests.yml                # Rust tests
│       └── deploy.yml                       # Production deployment
└── docker-compose.yml                       # Local dev environment
```

---

## Solver Validation Suite

### Benchmark 1: First-Order Decay — Analytical Comparison

**System:** A → B, power-law kinetics with k = 0.1 min⁻¹

**Initial conditions:** [A]₀ = 1.0 M, [B]₀ = 0.0 M, T = 298 K, V = 1 L (batch)

**Analytical solution:**
```
[A](t) = [A]₀ · exp(-k·t) = exp(-0.1·t)
[B](t) = [A]₀ · (1 - exp(-k·t)) = 1 - exp(-0.1·t)
```

**Expected values at t = 10 min:**
- [A] = 0.3679 M
- [B] = 0.6321 M

**Validation criteria:**
- Relative error < 0.1% for [A] and [B]
- Solver should use Adams-Moulton (non-stiff)
- Integration time < 100 ms for 1000 timepoints

### Benchmark 2: Reversible Reaction — Equilibrium

**System:** A ⇌ B with k₁ = 0.2 min⁻¹ (forward), k₂ = 0.05 min⁻¹ (reverse)

**Initial conditions:** [A]₀ = 1.0 M, [B]₀ = 0.0 M

**Equilibrium:**
```
K_eq = k₁/k₂ = 4.0
[A]_eq = [A]₀ / (1 + K_eq) = 0.2 M
[B]_eq = K_eq · [A]_eq = 0.8 M
```

**Expected values at t = 100 min (equilibrium):**
- [A] = 0.200 ± 0.001 M
- [B] = 0.800 ± 0.001 M

**Validation criteria:**
- Equilibrium reached within 1% by t = 50 min
- Mass balance: [A] + [B] = 1.0 M (±0.001 M) at all times

### Benchmark 3: Exothermic Batch Reactor — Adiabatic Temperature Rise

**System:** A → B, ΔH_rxn = -50 kJ/mol, k(T) = 10¹⁰ · exp(-8000/T) min⁻¹

**Initial conditions:** [A]₀ = 2.0 M, T₀ = 300 K, V = 1 L, ρ = 1 kg/L, Cp = 4.18 kJ/(kg·K)

**Energy balance:** ρ·V·Cp·dT/dt = (-ΔH_rxn)·r·V

**Expected adiabatic temperature rise:**
```
ΔT_max = [A]₀ · (-ΔH_rxn) / (ρ·Cp) = 2.0 · 50 / (1 · 4.18) = 23.9 K
T_max ≈ 300 + 23.9 = 323.9 K
```

**Validation criteria:**
- T_max = 323.9 ± 1.0 K
- Complete conversion ([A] < 0.01 M) at equilibrium
- Solver should detect stiffness and switch to BDF due to temperature-dependent kinetics

### Benchmark 4: Semi-Batch with Feed Addition

**System:** A + B → C, power-law k = 1.0 L/(mol·min)

**Initial:** [A]₀ = 1.0 M, [B]₀ = 0.0 M, V₀ = 1 L

**Feed:** [B]_feed = 2.0 M, F = 0.1 L/min, starts at t = 0, ends at t = 10 min

**Expected at t = 10 min:**
```
V = V₀ + F·t = 1 + 0.1·10 = 2.0 L

Moles of B added = [B]_feed · F · t = 2.0 · 0.1 · 10 = 2.0 mol

Numerical integration required for [A], [B], [C] due to dilution and reaction
Approximate: [C](10) ≈ 0.6–0.7 M (numerical target from reference solver)
```

**Validation criteria:**
- Volume trajectory V(t) = 1 + 0.1·t (exact)
- [C](10) within 5% of reference solution (validated against MATLAB ode15s)
- Mass balance: total moles in = moles out + moles accumulated

### Benchmark 5: Parameter Estimation Recovery Test

**True parameters:** k = 0.15 min⁻¹ (first-order A → B)

**Synthetic data:** 20 timepoints (0–30 min), [A] = exp(-0.15·t) + noise (σ = 0.01)

**Initial guess:** k₀ = 0.2 min⁻¹

**Expected fitted value:**
- k_fit = 0.150 ± 0.005 min⁻¹
- R² > 0.99
- Convergence within 10 LM iterations

**Validation criteria:**
- Recovered k within 5% of true value
- Standard error SE(k) < 0.01
- Residuals are normally distributed (Shapiro-Wilk p > 0.05)

---

## Verification Checklist

**Authentication & Authorization:**
- [ ] JWT token generation and validation
- [ ] OAuth 2.0 flows (Google, GitHub)
- [ ] Password reset flow
- [ ] Permission checks for project ownership
- [ ] Organization member role enforcement

**Reaction Scheme Editor:**
- [ ] Drag-and-drop species and reactions
- [ ] Force-directed auto-layout
- [ ] Save/load scheme to database
- [ ] Export to JSON
- [ ] Validation: stoichiometry balance warnings

**Dataset Management:**
- [ ] CSV upload with progress
- [ ] Unit conversion (mM, M, g/L, wt%)
- [ ] Data validation and error detection
- [ ] S3 storage with presigned URLs
- [ ] Dataset versioning

**Parameter Estimation (WASM):**
- [ ] WASM module loading in browser
- [ ] Levenberg-Marquardt convergence for ≤1000 points
- [ ] Real-time iteration updates
- [ ] Residual plots generation
- [ ] Confidence intervals calculation
- [ ] Pass all 5 solver validation benchmarks

**Parameter Estimation (Server):**
- [ ] Redis job queueing
- [ ] Worker picks up jobs by priority
- [ ] WebSocket progress streaming
- [ ] MCMC sampling with PyMC3 (1000+ points)
- [ ] Results stored in S3

**Reactor Simulation:**
- [ ] Batch reactor: isothermal and adiabatic
- [ ] Semi-batch with feed streams
- [ ] Jacket temperature control
- [ ] Energy balance integration
- [ ] Concentration-time plots
- [ ] Temperature trajectory plots

**Scale-Up Analysis:**
- [ ] Constant tip speed correlation
- [ ] Constant P/V correlation
- [ ] Constant mixing time correlation
- [ ] Heat transfer coefficient prediction
- [ ] Confidence interval propagation
- [ ] Equipment database integration (50+ reactors)

**Visualization:**
- [ ] D3.js concentration-time plots with pan/zoom
- [ ] Residual plots (time series, Q-Q, histogram)
- [ ] Tornado charts for sensitivity
- [ ] Correlation matrix heatmap
- [ ] Three.js 3D reactor viewer

**Billing & Usage:**
- [ ] Stripe Checkout integration
- [ ] Subscription webhook handling
- [ ] Plan limit enforcement (data points, compute minutes)
- [ ] Usage tracking and metering
- [ ] Customer Portal for subscription management

**Performance:**
- [ ] WASM parameter estimation <5s for 1000 points
- [ ] Reactor simulation <10s for 10,000 timepoints (server)
- [ ] API response time <200ms for CRUD operations
- [ ] S3 presigned URL generation <100ms
- [ ] WebSocket latency <50ms

---

## Deployment Architecture

### Production Infrastructure (AWS)

```
                                    ┌─────────────────┐
                                    │   CloudFront    │
                                    │   (CDN + SSL)   │
                                    └────────┬────────┘
                                             │
                     ┌───────────────────────┼───────────────────────┐
                     │                       │                       │
              ┌──────▼──────┐         ┌─────▼─────┐         ┌──────▼──────┐
              │   S3 (SPA)  │         │    ALB    │         │ S3 (WASM +  │
              │   React     │         │ (backend) │         │  datasets)  │
              └─────────────┘         └─────┬─────┘         └─────────────┘
                                            │
                                ┌───────────┴───────────┐
                                │                       │
                         ┌──────▼──────┐        ┌──────▼──────┐
                         │   ECS Task  │        │   ECS Task  │
                         │  (Axum API) │        │  (Axum API) │
                         │   2 vCPU    │        │   2 vCPU    │
                         │   4 GB RAM  │        │   4 GB RAM  │
                         └──────┬──────┘        └──────┬──────┘
                                │                       │
                                └───────────┬───────────┘
                                            │
                        ┌───────────────────┼───────────────────┐
                        │                   │                   │
                 ┌──────▼──────┐    ┌──────▼──────┐    ┌──────▼──────┐
                 │  RDS (PG16) │    │ ElastiCache │    │  ECS Tasks  │
                 │ db.t3.medium│    │   (Redis)   │    │  (Workers)  │
                 │  100GB SSD  │    │   r6g.large │    │  4 vCPU ea  │
                 └─────────────┘    └─────────────┘    └──────┬──────┘
                                                                │
                                                        ┌───────▼───────┐
                                                        │ Python (ECS)  │
                                                        │ PyMC3 + PDF   │
                                                        │   4 vCPU      │
                                                        │   8 GB RAM    │
                                                        └───────────────┘
```

**Scaling Configuration:**
- Backend API: Auto-scaling 2–10 tasks based on CPU (target 70%)
- Worker tasks: Auto-scaling 1–5 tasks based on Redis queue depth (target <10 jobs/worker)
- RDS: Read replicas for analytics queries (optional)
- ElastiCache: Redis cluster mode with 2 shards for HA

**Monitoring:**
- Prometheus scrapes `/metrics` from all services
- Grafana dashboards: API latency, solver execution time, queue depth, error rate
- Sentry for error tracking and alerting
- CloudWatch logs aggregation

**Cost Estimate (monthly, moderate usage):**
- ECS tasks (API): $100/mo (2 × t3.medium)
- ECS workers: $50/mo (1 × t3.medium avg)
- RDS: $90/mo (db.t3.medium + 100GB storage)
- ElastiCache: $80/mo (r6g.large)
- S3 + CloudFront: $30/mo (10TB transfer, 1TB storage)
- **Total: ~$350/mo** for 100 active users, 1000 simulations/day

---

## Post-MVP Roadmap

### v1.1 — Crystallization Modeling (Weeks 7–10)

**New Features:**
- Solubility modeling: van't Hoff, NRTL-SAC for solvent screening
- Population balance equations (PBE) solver using method of moments
- Cooling and anti-solvent crystallization profiles
- Particle size distribution (PSD) prediction
- Polymorphism risk assessment

**Database Changes:**
- New table: `crystallization_models` with PBE parameters
- New table: `solubility_data` for species-solvent pairs
- Equipment specs: crystallizers, filters, dryers

**Solver Additions:**
- PBE solver using method of moments (3-4 moments)
- Nucleation and growth rate correlations
- Agglomeration and breakage kernels

**Validation Benchmarks:**
- Cooling crystallization: compare PSD to experimental data from literature (Mullin, 2001)
- Anti-solvent crystallization: validate supersaturation profile

**Estimated Effort:** 3–4 weeks, 5000 lines of new code

### v1.2 — Biologics Process Development (Weeks 11–15)

**New Features:**
- Cell culture modeling: CHO cell metabolism (glucose, glutamine, lactate, ammonia)
- Bioreactor models: batch, fed-batch, perfusion
- Oxygen transfer (KLa) estimation and correlation
- Chromatography: general rate model for Protein A, IEX, HIC, SEC
- Ultrafiltration/diafiltration (UF/DF)

**Database Changes:**
- New table: `bioreactor_simulations` with cell density, viability, titer
- New table: `chromatography_models` with binding isotherms, column parameters
- Equipment specs: bioreactors (2L–2000L), chromatography columns

**Solver Additions:**
- Cell metabolism ODE system with Monod kinetics
- Chromatography column discretization (finite volume method)
- Membrane fouling model for UF/DF

**Validation Benchmarks:**
- Fed-batch bioreactor: compare titer trajectory to industrial case study
- Protein A chromatography: validate breakthrough curve vs experimental data

**Estimated Effort:** 4–5 weeks, 6000 lines of new code

### v1.3 — Model-Based Design of Experiments (MBDoE) (Weeks 16–18)

**New Features:**
- D-optimal experimental design for parameter estimation
- Sequential design: select next experiment to maximize information gain
- Design space calculation: identify regions where all CQAs are within spec
- Proven acceptable ranges (PAR) and normal operating ranges (NOR)
- Monte Carlo simulation for process variability

**Database Changes:**
- New table: `design_spaces` with parameter ranges, CQA constraints
- New table: `experimental_designs` with recommended conditions

**Implementation:**
- Fisher information matrix (FIM) computation
- D-optimality criterion maximization via genetic algorithm
- Monte Carlo sampling for robustness analysis

**Validation Benchmarks:**
- Compare D-optimal design to literature case studies (pharmaceutical DoE)
- Verify design space boundaries match regulatory examples

**Estimated Effort:** 2–3 weeks, 3000 lines of new code

### v1.4 — Regulatory Documentation Automation (Weeks 19–20)

**New Features:**
- ICH Q8(R2) Pharmaceutical Development report generation
- ICH Q11 Development and Manufacture of Drug Substances
- Process flow diagram (PFD) generation with control strategy annotation
- Risk assessment: fishbone diagrams, FMEA tables
- CQA and CPP identification tools

**Implementation:**
- LaTeX templates for ICH-compliant reports
- Automated CQA/CPP tables from sensitivity analysis
- Risk matrix generation from simulation results
- PDF export with embedded plots and tables

**Validation:**
- Review sample reports with regulatory affairs consultants
- Compare to real CMC submissions (anonymized)

**Estimated Effort:** 1–2 weeks, 2000 lines of new code

### v1.5 — Flow Chemistry (Weeks 21–22)

**New Features:**
- Plug flow reactor (PFR) with axial dispersion
- Tubular reactor with radial heat transfer
- Segmented flow (Taylor flow) modeling
- Microreactor scale-down from batch
- Residence time distribution (RTD) analysis

**Solver Additions:**
- PDE solver for axial dispersion (finite difference method)
- 2D axisymmetric reactor model (radial + axial)
- RTD convolution for reactor networks

**Validation Benchmarks:**
- Compare PFR simulation to analytical solution (Danckwerts boundary conditions)
- Validate microreactor scale-down vs literature data

**Estimated Effort:** 1–2 weeks, 2500 lines of new code

---

**Total Post-MVP Additions:** 10–15 weeks, ~20K lines of code, expanding to full pharmaceutical process development suite covering small molecules, biologics, continuous manufacturing, and regulatory compliance.
