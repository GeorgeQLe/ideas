# 61. CombustiQ — Combustion and Reacting Flow Simulation Platform

## Implementation Plan

**MVP Scope:** Browser-based combustion mechanism manager with 0D chemical kinetics reactor simulations (constant pressure, constant volume, perfectly stirred reactor) supporting Chemkin and Cantera YAML format mechanism import, mechanism reduction via DRG (Directed Relation Graph) algorithm to reduce 2000+ species mechanisms to 50-100 for CFD, 2D/3D reacting flow CFD with RANS turbulence (k-ε, k-ω SST), eddy dissipation concept (EDC) combustion model with finite-rate chemistry integration via CVODE, GPU-accelerated chemistry solver for 10-50x speedup over CPU implementation, pre-built mechanisms (GRI-Mech 3.0 methane/air, H2/syngas, NH3), laminar flame speed and ignition delay time calculators for mechanism validation, basic NOx emissions prediction via thermal Zeldovich mechanism, WebGPU-based 3D visualization for temperature and species concentration fields with contour plots and streamlines, PostgreSQL for mechanisms/projects/simulations and S3 for mesh/solution storage, Stripe billing with three tiers (Academic $149/mo per lab / Pro $499/mo / Enterprise custom).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| CFD Solver | Rust | Custom finite volume method, SIMPLE/PISO pressure-velocity coupling |
| Chemistry Solver | Rust + CUDA | GPU-accelerated ODE integration via custom CVODE port |
| Mechanism Reduction | Rust + Python (FastAPI) | DRG/DRGEP algorithms, sensitivity analysis via Python/Cantera |
| Database | PostgreSQL 16 | Users, mechanisms, projects, simulations, usage tracking |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | Mechanism files, mesh data, solution fields, visualization snapshots |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| 3D Visualization | WebGPU | GPU-accelerated contour plots, streamlines, volume rendering |
| Real-time | WebSocket (Axum) | Live simulation progress, convergence monitoring |
| Job Queue | Redis 7 + Tokio tasks | Server-side simulation job management, priority queue |
| Compute | AWS GPU instances | p3.2xlarge (V100) or p4d.24xlarge for large simulations |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | Static frontend assets, WebGPU shader delivery |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, error tracking, solver convergence stats |
| CI/CD | GitHub Actions | Rust tests, Docker image push, GPU solver tests |

### Key Architecture Decisions

1. **GPU-accelerated chemistry integration for 10-50x speedup**: Chemical kinetics is the bottleneck in combustion CFD — for detailed mechanisms with 100+ species, chemistry integration takes 90%+ of total wall time. By porting CVODE (the industry-standard stiff ODE solver from LLNL) to CUDA and batching cell-wise chemistry across GPU cores, we achieve 10-50x speedup depending on mechanism complexity. A 1M-cell simulation with GRI-Mech 3.0 (53 species) that takes 48 hours on CPU runs in 1-2 hours on a single V100 GPU.

2. **Mechanism reduction pipeline integrated into platform**: Detailed mechanisms (e.g., Jet-A surrogate with 2000+ species) are too expensive for 3D CFD. The platform includes DRG (Directed Relation Graph) and DRGEP (with error propagation) algorithms that automatically reduce mechanisms to 50-100 species while preserving ignition delay, flame speed, and extinction characteristics within user-specified tolerance (typically 10-20% error). This is a Python/Cantera pipeline exposed via FastAPI, with results validated against experimental databases.

3. **Low-Mach number formulation for combustion CFD**: Most practical combustion applications (gas turbines, industrial burners, IC engines except near TDC) are low-Mach number flows where acoustic waves are not important. The low-Mach formulation filters acoustic waves by solving a pressure Poisson equation (SIMPLE/PISO algorithm) rather than the compressible Euler/Navier-Stokes, reducing timestep restrictions by 100-1000x and enabling efficient simulation of slow combustion processes.

4. **Eddy Dissipation Concept (EDC) for turbulence-chemistry interaction**: RANS simulations cannot directly couple finite-rate chemistry because turbulent mixing occurs at sub-grid scales. EDC (Magnussen model) is the industry-standard approach: it assumes chemistry occurs in fine structures occupying a volume fraction of the cell, with a timescale derived from turbulence dissipation rate. This is more accurate than simple eddy dissipation (which assumes infinitely fast chemistry) and more computationally tractable than transported PDF methods.

5. **Cantera-compatible thermodynamics and kinetics**: Rather than building a new chemical mechanism format, CombustiQ uses Cantera's thermodynamics (NASA polynomials, JANAF tables) and kinetics (Arrhenius, falloff, pressure-dependent) data structures. This allows import of any Chemkin or Cantera YAML mechanism from the literature. Thermodynamic properties are pre-tabulated for common species to avoid recomputation in CFD.

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";  -- For fuzzy text search on mechanisms

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

-- Organizations (for labs/companies)
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

-- Chemical Mechanisms
CREATE TABLE mechanisms (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    description TEXT,
    format TEXT NOT NULL,  -- chemkin | cantera_yaml
    n_species INTEGER NOT NULL,
    n_reactions INTEGER NOT NULL,
    fuel_type TEXT,  -- methane | hydrogen | ammonia | syngas | jet_a | diesel | gasoline
    file_url TEXT NOT NULL,  -- S3 URL to mechanism file
    thermo_url TEXT,  -- S3 URL to thermodynamic database
    transport_url TEXT,  -- S3 URL to transport database
    validation_data JSONB,  -- {ignition_delay: {}, flame_speed: {}, extinction: {}}
    is_builtin BOOLEAN DEFAULT false,
    is_public BOOLEAN DEFAULT false,
    uploaded_by UUID REFERENCES users(id),
    download_count INTEGER DEFAULT 0,
    tags TEXT[] DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX mechanisms_fuel_idx ON mechanisms(fuel_type);
CREATE INDEX mechanisms_name_trgm_idx ON mechanisms USING gin(name gin_trgm_ops);
CREATE INDEX mechanisms_tags_idx ON mechanisms USING gin(tags);

-- Reduced Mechanisms (derived from parent mechanisms)
CREATE TABLE reduced_mechanisms (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    parent_id UUID NOT NULL REFERENCES mechanisms(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    reduction_method TEXT NOT NULL,  -- drg | drgep | sensitivity
    n_species INTEGER NOT NULL,
    n_reactions INTEGER NOT NULL,
    error_metrics JSONB NOT NULL,  -- {ignition_delay_error: 0.12, flame_speed_error: 0.08}
    file_url TEXT NOT NULL,
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX reduced_mech_parent_idx ON reduced_mechanisms(parent_id);

-- Projects (0D reactor or CFD case)
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    project_type TEXT NOT NULL,  -- reactor_0d | cfd_2d | cfd_3d
    mechanism_id UUID REFERENCES mechanisms(id),
    configuration JSONB NOT NULL DEFAULT '{}',  -- Reactor conditions or CFD setup
    mesh_url TEXT,  -- S3 URL for CFD mesh
    mesh_cells INTEGER,
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

-- Simulations
CREATE TABLE simulations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | running | completed | failed | cancelled
    execution_mode TEXT NOT NULL DEFAULT 'server',  -- server | gpu
    parameters JSONB NOT NULL DEFAULT '{}',  -- Time integration, tolerances, output frequency
    results_url TEXT,  -- S3 URL for result data
    results_summary JSONB,  -- Quick-access summary (max temp, emissions, ignition delay)
    error_message TEXT,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    wall_time_ms INTEGER,
    gpu_hours REAL DEFAULT 0,  -- For billing
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
    gpu_type TEXT,  -- v100 | a100 | cpu_only
    gpu_memory_gb INTEGER,
    priority INTEGER DEFAULT 0,  -- Higher = more urgent
    progress_pct REAL DEFAULT 0.0,
    convergence_data JSONB DEFAULT '[]',  -- [{iteration, residual, max_temperature}]
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX jobs_sim_idx ON simulation_jobs(simulation_id);
CREATE INDEX jobs_worker_idx ON simulation_jobs(worker_id);

-- Validation Benchmarks (for mechanism validation)
CREATE TABLE validation_benchmarks (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    benchmark_type TEXT NOT NULL,  -- ignition_delay | flame_speed | extinction | species_profile
    fuel TEXT NOT NULL,
    conditions JSONB NOT NULL,  -- {temperature, pressure, equivalence_ratio, diluent}
    experimental_data JSONB NOT NULL,  -- Reference values from literature
    reference_citation TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX benchmarks_type_idx ON validation_benchmarks(benchmark_type);
CREATE INDEX benchmarks_fuel_idx ON validation_benchmarks(fuel);

-- Usage Tracking (for billing enforcement)
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,  -- gpu_hours | cpu_hours | storage_gb
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
pub struct Mechanism {
    pub id: Uuid,
    pub name: String,
    pub description: Option<String>,
    pub format: String,
    pub n_species: i32,
    pub n_reactions: i32,
    pub fuel_type: Option<String>,
    pub file_url: String,
    pub thermo_url: Option<String>,
    pub transport_url: Option<String>,
    pub validation_data: Option<serde_json::Value>,
    pub is_builtin: bool,
    pub is_public: bool,
    pub uploaded_by: Option<Uuid>,
    pub download_count: i32,
    pub tags: Vec<String>,
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
    pub mechanism_id: Option<Uuid>,
    pub configuration: serde_json::Value,
    pub mesh_url: Option<String>,
    pub mesh_cells: Option<i32>,
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
    pub parameters: serde_json::Value,
    pub results_url: Option<String>,
    pub results_summary: Option<serde_json::Value>,
    pub error_message: Option<String>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub wall_time_ms: Option<i32>,
    pub gpu_hours: f32,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct ReactorParams {
    pub reactor_type: ReactorType,
    pub temperature: f64,      // K
    pub pressure: f64,         // Pa
    pub equivalence_ratio: f64,
    pub fuel: String,
    pub oxidizer: String,
    pub diluent: Option<String>,
    pub end_time: f64,         // seconds
    pub rtol: f64,             // Relative tolerance for ODE solver
    pub atol: f64,             // Absolute tolerance
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum ReactorType {
    ConstantPressure,
    ConstantVolume,
    PerfectlyStirredReactor,
    PlugFlowReactor,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct CfdParams {
    pub turbulence_model: TurbulenceModel,
    pub combustion_model: CombustionModel,
    pub time_integration: TimeIntegration,
    pub end_time: f64,
    pub timestep: f64,
    pub output_frequency: i32,
    pub convergence_tol: f64,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum TurbulenceModel {
    KEpsilon,
    KOmegaSst,
    Laminar,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum CombustionModel {
    EddyDissipationConcept,
    Flamelet,
    FiniteRate,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum TimeIntegration {
    Simple,
    Piso,
    Pimple,
}
```

---

## Solver Architecture Deep-Dive

### Governing Equations for Reacting Flows

CombustiQ solves the **low-Mach number reactive Navier-Stokes equations** for combustion CFD:

**Continuity:**
```
∂ρ/∂t + ∇·(ρu) = 0
```

**Momentum (with low-Mach approximation):**
```
∂(ρu)/∂t + ∇·(ρuu) = -∇p + ∇·τ + ρg

where τ = μ(∇u + (∇u)ᵀ - 2/3(∇·u)I)  (viscous stress tensor)
```

**Species conservation (for each species k = 1, ..., N):**
```
∂(ρYₖ)/∂t + ∇·(ρuYₖ) = -∇·Jₖ + ω̇ₖWₖ

where:
  Yₖ = mass fraction of species k
  Jₖ = -ρDₖ∇Yₖ  (Fick's law diffusion)
  ω̇ₖ = net molar production rate from reactions (mol/m³/s)
  Wₖ = molecular weight of species k
```

**Energy equation:**
```
∂(ρh)/∂t + ∇·(ρuh) = ∇·(λ∇T) + ∇·(Σₖ hₖJₖ) - Σₖ ω̇ₖWₖhₖ + Q̇ᵣₐd

where:
  h = Σₖ Yₖhₖ(T)  (mixture specific enthalpy)
  hₖ(T) = species enthalpy from NASA polynomials
  Q̇ᵣₐd = radiative heat transfer (discrete ordinates or P1 approximation)
```

**Equation of state (ideal gas):**
```
p = ρRT/W̄

where W̄ = (Σₖ Yₖ/Wₖ)⁻¹  (mean molecular weight)
```

### Chemical Kinetics

For a mechanism with `M` reactions, the net production rate of species `k` is:

```
ω̇ₖ = Σⱼ₌₁ᴹ νₖⱼ(kfⱼ∏ᵢ[Xᵢ]^ν'ᵢⱼ - kᵣⱼ∏ᵢ[Xᵢ]^ν"ᵢⱼ)

where:
  νₖⱼ = ν"ₖⱼ - ν'ₖⱼ  (stoichiometric coefficient: products - reactants)
  kfⱼ = Aⱼ T^βⱼ exp(-Eⱼ/RT)  (forward rate, Arrhenius form)
  kᵣⱼ = kfⱼ/Kₑqⱼ  (reverse rate from equilibrium constant)
  [Xᵢ] = molar concentration of species i
```

**Three-body reactions** (with enhanced collision efficiency factors `αᵢ`):
```
kfⱼ = kfⱼ⁰ Σᵢ αᵢ[Xᵢ]
```

**Falloff reactions** (pressure-dependent, Lindemann/Troe form):
```
kfⱼ = kfⱼ∞ (k₀[M]/(kfⱼ∞ + k₀[M])) Fⱼ

where Fⱼ is the Troe blending function
```

### Turbulence-Chemistry Interaction: Eddy Dissipation Concept (EDC)

In RANS turbulence models, turbulent fluctuations are not resolved, so the mean reaction rate `⟨ω̇ₖ⟩` cannot be computed directly from mean concentrations due to nonlinearity. EDC assumes chemistry occurs in **fine structures** occupying a volume fraction `ξ*`:

```
ξ* = Cξ (νε/k²)^(1/2)

where:
  ν = kinematic viscosity
  ε = turbulence dissipation rate (from k-ε or k-ω model)
  k = turbulence kinetic energy
  Cξ = 2.1377 (calibrated constant)
```

The **residence time in fine structures** is:
```
τ* = Cτ (ν/ε)^(1/2)
```

The **mean reaction rate** in the cell is:
```
⟨ω̇ₖ⟩ = ρ (ξ*²)/(τ*(1 - ξ*³)) (Yₖ* - Yₖ)

where Yₖ* is the fine-structure mass fraction computed by integrating finite-rate chemistry:
  dYₖ*/dt = ω̇ₖ(Yₖ*, T*) Wₖ/ρ  over time τ*
```

This system is integrated **cell-by-cell on the GPU** using a custom CUDA CVODE port, achieving 10-50x speedup over CPU.

### GPU Acceleration of Chemistry

The chemistry integration bottleneck:
- For GRI-Mech 3.0 (53 species, 325 reactions), each cell requires solving a 53-dimensional stiff ODE system
- A 1M-cell mesh requires 1M independent ODE solves per CFD timestep
- Typical transient simulations require 10,000-100,000 CFD timesteps

**GPU parallelization strategy:**
1. **Batching:** Group cells into blocks of 256-1024 (one CUDA block per batch)
2. **Warp-level parallelism:** Each warp (32 threads) solves one cell's chemistry
3. **Shared memory for thermodynamic data:** NASA polynomial coefficients, molecular weights stored in shared memory (reduces global memory traffic by 10x)
4. **Asynchronous transfer:** CFD solver on CPU, chemistry on GPU, overlapping computation via CUDA streams

**Custom CVODE GPU port:**
- BDF (backward differentiation formula) method for stiff ODEs
- Newton iteration with direct dense solve (53×53 Jacobian per cell)
- Adaptive timestep control with error estimation
- Convergence in typically 5-15 iterations per chemistry timestep

### Low-Mach Number Pressure-Velocity Coupling: SIMPLE Algorithm

The low-Mach formulation decouples thermodynamic pressure `p₀(t)` (uniform in space) from hydrodynamic pressure `p'(x,t)` (spatial variations):

```
p(x,t) = p₀(t) + p'(x,t)

where p₀ >> p' for low Mach number
```

**SIMPLE (Semi-Implicit Method for Pressure-Linked Equations) algorithm:**

1. **Momentum predictor:** Solve momentum equations with guessed pressure field `p'ⁿ`:
   ```
   ρuⁿ⁺¹* = ρuⁿ + Δt[-∇·(ρuu) - ∇p'ⁿ + ∇·τ]ⁿ
   ```

2. **Pressure correction:** Enforce continuity by solving Poisson equation:
   ```
   ∇·[(1/aₚ)∇p''] = ∇·u*

   where aₚ is the momentum matrix diagonal coefficient
   ```

3. **Velocity correction:**
   ```
   uⁿ⁺¹ = u* - (Δt/aₚ)∇p''
   pⁿ⁺¹ = p'ⁿ + αₚp''  (under-relaxation, αₚ ≈ 0.3)
   ```

4. **Species and energy update:** Solve species and energy equations explicitly with corrected velocity.

5. **Convergence check:** If residuals < tolerance, proceed to next timestep. Otherwise, repeat steps 1-5.

---

## Architecture Deep-Dives

### 1. Mechanism Import and Thermodynamics (Rust)

The mechanism parser supports Chemkin and Cantera YAML formats, extracting species, reactions, thermodynamic data, and transport properties.

```rust
// src/chemistry/mechanism.rs

use serde::{Deserialize, Serialize};
use std::collections::HashMap;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Mechanism {
    pub name: String,
    pub species: Vec<Species>,
    pub reactions: Vec<Reaction>,
    pub thermo_db: ThermoDatabase,
    pub transport_db: Option<TransportDatabase>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Species {
    pub name: String,
    pub composition: HashMap<String, i32>,  // e.g., {"C": 1, "H": 4} for CH4
    pub molecular_weight: f64,              // kg/kmol
    pub charge: i32,
    pub nasa_low: NasaPoly,                 // NASA 7-coefficient polynomials for T < T_mid
    pub nasa_high: NasaPoly,                // for T >= T_mid
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct NasaPoly {
    pub t_min: f64,
    pub t_max: f64,
    pub coeffs: [f64; 7],  // [a1, a2, a3, a4, a5, a6, a7]
}

impl NasaPoly {
    /// Compute dimensionless enthalpy h/(RT) from NASA polynomials
    pub fn enthalpy_rt(&self, t: f64) -> f64 {
        let a = &self.coeffs;
        a[0] + a[1]*t/2.0 + a[2]*t*t/3.0 + a[3]*t*t*t/4.0
            + a[4]*t*t*t*t/5.0 + a[5]/t
    }

    /// Compute dimensionless entropy s/R
    pub fn entropy_r(&self, t: f64) -> f64 {
        let a = &self.coeffs;
        a[0]*t.ln() + a[1]*t + a[2]*t*t/2.0 + a[3]*t*t*t/3.0
            + a[4]*t*t*t*t/4.0 + a[6]
    }

    /// Compute specific heat Cp/R
    pub fn cp_r(&self, t: f64) -> f64 {
        let a = &self.coeffs;
        a[0] + a[1]*t + a[2]*t*t + a[3]*t*t*t + a[4]*t*t*t*t
    }
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Reaction {
    pub index: usize,
    pub equation: String,  // e.g., "H + O2 <=> OH + O"
    pub reactants: HashMap<usize, f64>,  // species_idx -> stoich_coeff
    pub products: HashMap<usize, f64>,
    pub reversible: bool,
    pub rate_constant: RateConstant,
    pub third_body: Option<ThirdBodyData>,
    pub falloff: Option<FalloffData>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "type")]
pub enum RateConstant {
    Arrhenius {
        a: f64,      // Pre-exponential factor
        beta: f64,   // Temperature exponent
        ea: f64,     // Activation energy (J/mol)
    },
    Falloff {
        k_inf: Box<RateConstant>,
        k_0: Box<RateConstant>,
        troe: Option<TroeParams>,
    },
}

impl RateConstant {
    /// Evaluate forward rate constant at temperature T (K)
    pub fn eval(&self, t: f64, m: f64) -> f64 {
        match self {
            RateConstant::Arrhenius { a, beta, ea } => {
                a * t.powf(*beta) * (-ea / (8314.46 * t)).exp()
            }
            RateConstant::Falloff { k_inf, k_0, troe } => {
                let k_inf_val = k_inf.eval(t, m);
                let k_0_val = k_0.eval(t, m);
                let pr = k_0_val * m / k_inf_val;
                let f = troe.as_ref().map_or(1.0, |tr| tr.eval(t, pr));
                k_inf_val * (pr / (1.0 + pr)) * f
            }
        }
    }
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TroeParams {
    pub a: f64,
    pub t3: f64,
    pub t1: f64,
    pub t2: Option<f64>,
}

impl TroeParams {
    pub fn eval(&self, t: f64, pr: f64) -> f64 {
        let log_f_cent = ((1.0 - self.a) * (-t / self.t3).exp()
                         + self.a * (-t / self.t1).exp()
                         + self.t2.map_or(0.0, |t2| (-t2 / t).exp())).ln();
        let c = -0.4 - 0.67 * log_f_cent;
        let n = 0.75 - 1.27 * log_f_cent;
        let log_pr = pr.ln();
        let log_f = log_f_cent / (1.0 + ((log_pr + c) / (n - 0.14 * (log_pr + c))).powi(2));
        log_f.exp()
    }
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ThirdBodyData {
    pub default_efficiency: f64,
    pub efficiencies: HashMap<usize, f64>,  // species_idx -> efficiency
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct FalloffData {
    pub reactant_idx: usize,  // Species acting as third body
}

#[derive(Debug, Clone)]
pub struct ThermoDatabase {
    pub species_data: Vec<Species>,
}

impl ThermoDatabase {
    /// Get species enthalpy at temperature T (J/kg)
    pub fn enthalpy(&self, species_idx: usize, t: f64) -> f64 {
        let sp = &self.species_data[species_idx];
        let nasa = if t < sp.nasa_low.t_max {
            &sp.nasa_low
        } else {
            &sp.nasa_high
        };
        let h_rt = nasa.enthalpy_rt(t);
        h_rt * 8314.46 * t / sp.molecular_weight  // J/kg
    }

    /// Get species specific heat Cp at temperature T (J/kg/K)
    pub fn cp(&self, species_idx: usize, t: f64) -> f64 {
        let sp = &self.species_data[species_idx];
        let nasa = if t < sp.nasa_low.t_max {
            &sp.nasa_low
        } else {
            &sp.nasa_high
        };
        let cp_r = nasa.cp_r(t);
        cp_r * 8314.46 / sp.molecular_weight  // J/kg/K
    }
}

#[derive(Debug, Clone)]
pub struct TransportDatabase {
    // Placeholder for future transport property implementation
}

/// Parse Cantera YAML mechanism file
pub fn parse_cantera_yaml(yaml_str: &str) -> Result<Mechanism, MechanismError> {
    // YAML parsing implementation using serde_yaml
    // Extract species, reactions, thermodynamics, transport
    todo!("Cantera YAML parser")
}

/// Parse Chemkin format mechanism
pub fn parse_chemkin(
    mech_str: &str,
    thermo_str: &str,
    transport_str: Option<&str>,
) -> Result<Mechanism, MechanismError> {
    // Chemkin text format parser
    // Handle REACTIONS, SPECIES, THERMO sections
    todo!("Chemkin parser")
}

#[derive(Debug)]
pub enum MechanismError {
    ParseError(String),
    InvalidSpecies(String),
    InvalidReaction(String),
}
```

### 2. 0D Reactor Simulations (Rust + CVODE)

0D reactors simulate spatially uniform conditions, solving only the chemical kinetics ODEs. This is used for mechanism validation (ignition delay, flame speed) and rapid parametric studies.

```rust
// src/chemistry/reactor.rs

use crate::chemistry::mechanism::{Mechanism, Species, Reaction};
use ndarray::{Array1, Array2};

pub struct ConstantPressureReactor {
    pub mechanism: Mechanism,
    pub temperature: f64,     // K
    pub pressure: f64,        // Pa
    pub mass_fractions: Array1<f64>,  // Yₖ for each species
}

impl ConstantPressureReactor {
    pub fn new(
        mechanism: Mechanism,
        temperature: f64,
        pressure: f64,
        composition: &[(usize, f64)],  // (species_idx, mole_fraction)
    ) -> Self {
        let n_species = mechanism.species.len();
        let mut mass_fractions = Array1::<f64>::zeros(n_species);

        // Convert mole fractions to mass fractions
        let total_moles: f64 = composition.iter().map(|(_, x)| x).sum();
        let mut total_mass = 0.0;
        for &(idx, x) in composition {
            let mass = x * mechanism.species[idx].molecular_weight;
            mass_fractions[idx] = mass;
            total_mass += mass;
        }
        mass_fractions /= total_mass;

        Self {
            mechanism,
            temperature,
            pressure,
            mass_fractions,
        }
    }

    /// Compute species production rates ω̇ₖ (kg/m³/s)
    pub fn production_rates(&self, t_current: f64, y: &Array1<f64>) -> Array1<f64> {
        let n_species = self.mechanism.species.len();
        let mut omega = Array1::<f64>::zeros(n_species);

        // Extract state: y = [Y₀, Y₁, ..., Yₙ, T]
        let mass_fractions = y.slice(ndarray::s![..n_species]);
        let temperature = y[n_species];

        // Compute molar concentrations [Xₖ] = ρYₖ/Wₖ
        let density = self.density(temperature, &mass_fractions);
        let mut concentrations = Array1::<f64>::zeros(n_species);
        for k in 0..n_species {
            concentrations[k] = density * mass_fractions[k]
                / self.mechanism.species[k].molecular_weight * 1000.0;  // mol/m³
        }

        // Evaluate each reaction
        for rxn in &self.mechanism.reactions {
            let kf = rxn.rate_constant.eval(temperature, 0.0);  // Forward rate

            // Compute equilibrium constant for reverse rate
            let delta_g = self.gibbs_change(rxn, temperature);
            let k_eq = (-delta_g / (8314.46 * temperature)).exp();
            let kr = kf / k_eq;

            // Forward rate: kf * ∏[Xᵢ]^ν'ᵢ
            let mut rate_f = kf;
            for (&species_idx, &stoich) in &rxn.reactants {
                rate_f *= concentrations[species_idx].powf(stoich);
            }

            // Reverse rate: kr * ∏[Xᵢ]^ν"ᵢ
            let mut rate_r = kr;
            for (&species_idx, &stoich) in &rxn.products {
                rate_r *= concentrations[species_idx].powf(stoich);
            }

            let net_rate = rate_f - rate_r;

            // Update production rates: ω̇ₖ += νₖⱼ * net_rate * Wₖ
            for (&species_idx, &stoich) in &rxn.reactants {
                let nu = -stoich;  // Reactants have negative stoichiometry
                omega[species_idx] += nu * net_rate
                    * self.mechanism.species[species_idx].molecular_weight / 1000.0;
            }
            for (&species_idx, &stoich) in &rxn.products {
                omega[species_idx] += stoich * net_rate
                    * self.mechanism.species[species_idx].molecular_weight / 1000.0;
            }
        }

        omega
    }

    /// ODE right-hand side: dy/dt = f(t, y)
    /// For constant pressure: dYₖ/dt = ω̇ₖ/ρ, dT/dt = -(Σₖ hₖ ω̇ₖ)/(ρCₚ)
    pub fn rhs(&self, t: f64, y: &Array1<f64>, dydt: &mut Array1<f64>) {
        let n_species = self.mechanism.species.len();
        let mass_fractions = y.slice(ndarray::s![..n_species]);
        let temperature = y[n_species];

        let density = self.density(temperature, &mass_fractions);
        let omega = self.production_rates(t, y);

        // Species equations: dYₖ/dt = ω̇ₖ/ρ
        for k in 0..n_species {
            dydt[k] = omega[k] / density;
        }

        // Energy equation: dT/dt = -(Σₖ hₖ ω̇ₖ)/(ρCₚ)
        let mut cp = 0.0;
        let mut enthalpy_source = 0.0;
        for k in 0..n_species {
            let h_k = self.mechanism.thermo_db.enthalpy(k, temperature);
            let cp_k = self.mechanism.thermo_db.cp(k, temperature);
            cp += mass_fractions[k] * cp_k;
            enthalpy_source += h_k * omega[k];
        }
        dydt[n_species] = -enthalpy_source / (density * cp);
    }

    /// Compute mixture density ρ = pW̄/(RT)
    fn density(&self, temperature: f64, mass_fractions: &ndarray::ArrayBase<impl ndarray::Data<Elem = f64>, ndarray::Dim<[usize; 1]>>) -> f64 {
        let mut mean_mol_weight_inv = 0.0;
        for k in 0..self.mechanism.species.len() {
            mean_mol_weight_inv += mass_fractions[k] / self.mechanism.species[k].molecular_weight;
        }
        let mean_mol_weight = 1.0 / mean_mol_weight_inv;
        self.pressure * mean_mol_weight / (8314.46 * temperature)
    }

    /// Compute Gibbs free energy change ΔG for reaction
    fn gibbs_change(&self, rxn: &Reaction, temperature: f64) -> f64 {
        let mut delta_g = 0.0;
        for (&species_idx, &stoich) in &rxn.products {
            let sp = &self.mechanism.species[species_idx];
            let nasa = if temperature < sp.nasa_low.t_max {
                &sp.nasa_low
            } else {
                &sp.nasa_high
            };
            let g_rt = nasa.enthalpy_rt(temperature) - nasa.entropy_r(temperature);
            delta_g += stoich * g_rt;
        }
        for (&species_idx, &stoich) in &rxn.reactants {
            let sp = &self.mechanism.species[species_idx];
            let nasa = if temperature < sp.nasa_low.t_max {
                &sp.nasa_low
            } else {
                &sp.nasa_high
            };
            let g_rt = nasa.enthalpy_rt(temperature) - nasa.entropy_r(temperature);
            delta_g -= stoich * g_rt;
        }
        delta_g * 8314.46 * temperature
    }

    /// Integrate reactor from t=0 to t=end_time using CVODE
    pub fn integrate(&mut self, end_time: f64, rtol: f64, atol: f64) -> ReactorResult {
        let n_species = self.mechanism.species.len();
        let n_vars = n_species + 1;  // Yₖ + T

        // Initial state: [Y₀, Y₁, ..., Yₙ, T]
        let mut y = Array1::<f64>::zeros(n_vars);
        y.slice_mut(ndarray::s![..n_species]).assign(&self.mass_fractions);
        y[n_species] = self.temperature;

        // Time integration (simplified — production would use sundials CVODE bindings)
        let mut time_points = vec![0.0];
        let mut states = vec![y.clone()];

        let dt_init = 1e-9;  // Start with very small timestep
        let mut t = 0.0;
        let mut dt = dt_init;

        while t < end_time {
            let mut dydt = Array1::<f64>::zeros(n_vars);
            self.rhs(t, &y, &mut dydt);

            // Simple explicit Euler (production uses BDF from CVODE)
            let y_new = &y + &(&dydt * dt);

            // Clamp mass fractions to [0, 1]
            for k in 0..n_species {
                y_new[k] = y_new[k].max(0.0).min(1.0);
            }

            // Accept step and advance
            y = y_new;
            t += dt;

            time_points.push(t);
            states.push(y.clone());

            // Adaptive timestep (simplified)
            dt = (dt * 1.2).min(end_time - t).max(1e-12);
        }

        ReactorResult {
            time: time_points,
            states,
            species_names: self.mechanism.species.iter().map(|s| s.name.clone()).collect(),
        }
    }
}

pub struct ReactorResult {
    pub time: Vec<f64>,
    pub states: Vec<Array1<f64>>,  // Each state is [Y₀, ..., Yₙ, T]
    pub species_names: Vec<String>,
}

impl ReactorResult {
    /// Extract temperature history
    pub fn temperature(&self) -> Vec<f64> {
        self.states.iter().map(|state| state[state.len() - 1]).collect()
    }

    /// Extract mass fraction history for a species
    pub fn mass_fraction(&self, species_idx: usize) -> Vec<f64> {
        self.states.iter().map(|state| state[species_idx]).collect()
    }

    /// Compute ignition delay time (time when temperature rises by 400K)
    pub fn ignition_delay(&self) -> Option<f64> {
        let t0 = self.states[0][self.states[0].len() - 1];
        let threshold = t0 + 400.0;

        for (i, state) in self.states.iter().enumerate() {
            let temp = state[state.len() - 1];
            if temp >= threshold {
                return Some(self.time[i]);
            }
        }
        None
    }
}
```

### 3. CFD Solver — SIMPLE Pressure-Velocity Coupling (Rust)

The finite volume CFD solver implements SIMPLE algorithm for pressure-velocity coupling in low-Mach combustion flows.

```rust
// src/cfd/simple.rs

use crate::cfd::mesh::Mesh;
use crate::cfd::fields::{ScalarField, VectorField};
use crate::chemistry::mechanism::Mechanism;
use nalgebra_sparse::CsrMatrix;

pub struct SimpleSolver {
    pub mesh: Mesh,
    pub mechanism: Mechanism,
    pub velocity: VectorField,
    pub pressure: ScalarField,
    pub pressure_correction: ScalarField,
    pub temperature: ScalarField,
    pub density: ScalarField,
    pub species: Vec<ScalarField>,  // One field per species
    pub turbulent_ke: ScalarField,
    pub turbulent_epsilon: ScalarField,
    pub mu_eff: ScalarField,  // Effective viscosity (laminar + turbulent)
    pub dt: f64,
    pub iteration: u32,
}

impl SimpleSolver {
    pub fn new(mesh: Mesh, mechanism: Mechanism, dt: f64) -> Self {
        let n_cells = mesh.n_cells;
        let n_species = mechanism.species.len();

        Self {
            mesh,
            mechanism,
            velocity: VectorField::zeros(n_cells),
            pressure: ScalarField::zeros(n_cells),
            pressure_correction: ScalarField::zeros(n_cells),
            temperature: ScalarField::new(vec![300.0; n_cells]),
            density: ScalarField::new(vec![1.0; n_cells]),
            species: vec![ScalarField::zeros(n_cells); n_species],
            turbulent_ke: ScalarField::new(vec![1.0; n_cells]),
            turbulent_epsilon: ScalarField::new(vec![1.0; n_cells]),
            mu_eff: ScalarField::new(vec![1e-5; n_cells]),
            dt,
            iteration: 0,
        }
    }

    pub fn solve_iteration(&mut self) -> IterationResult {
        // 1. Momentum predictor
        let u_star = self.solve_momentum_predictor();

        // 2. Pressure correction
        self.solve_pressure_correction(&u_star);

        // 3. Velocity and pressure correction
        self.correct_velocity(&u_star);
        self.correct_pressure();

        // 4. Turbulence (k-ε)
        self.solve_turbulence_k();
        self.solve_turbulence_epsilon();
        self.update_turbulent_viscosity();

        // 5. Energy equation
        self.solve_energy();

        // 6. Species transport
        for i in 0..self.species.len() {
            self.solve_species_transport(i);
        }

        // 7. Chemistry integration (GPU)
        self.integrate_chemistry();

        // 8. Update density from ideal gas law
        self.update_density();

        // 9. Compute residuals
        self.iteration += 1;
        self.compute_residuals()
    }

    fn solve_momentum_predictor(&mut self) -> VectorField {
        let mut A = CsrMatrix::zeros(self.mesh.n_cells, self.mesh.n_cells);
        let mut b = vec![0.0; self.mesh.n_cells * 3];

        for cell in 0..self.mesh.n_cells {
            let rho = self.density[cell];
            let vol = self.mesh.volumes[cell];

            // Transient term
            let diag = rho * vol / self.dt;

            // Convection + diffusion coefficients
            let conv_coeff = self.compute_convection_coeff(cell);
            let diff_coeff = self.mu_eff[cell];

            A.add(cell, cell, diag + conv_coeff);

            // Off-diagonal (neighbors)
            for &face in &self.mesh.cell_faces[cell] {
                if let Some(nb) = self.mesh.face_neighbor(face, cell) {
                    let face_area = self.mesh.face_areas[face];
                    let face_dist = self.mesh.face_distances[face];
                    let coeff = diff_coeff * face_area / face_dist;
                    A.add(cell, nb, -coeff);
                    A.add(cell, cell, coeff);
                }
            }

            // Source terms: pressure gradient + transient
            let grad_p = self.compute_gradient(&self.pressure, cell);
            let u_old = self.velocity[cell];

            b[cell * 3] = -grad_p[0] * vol + rho * u_old[0] * vol / self.dt;
            b[cell * 3 + 1] = -grad_p[1] * vol + rho * u_old[1] * vol / self.dt;
            b[cell * 3 + 2] = -grad_p[2] * vol + rho * u_old[2] * vol / self.dt;
        }

        // Solve sparse system
        let u_flat = solve_sparse(&A, &b);
        VectorField::from_flat(&u_flat, self.mesh.n_cells)
    }

    fn solve_pressure_correction(&mut self, u_star: &VectorField) {
        let mut A = CsrMatrix::zeros(self.mesh.n_cells, self.mesh.n_cells);
        let mut b = vec![0.0; self.mesh.n_cells];

        for cell in 0..self.mesh.n_cells {
            let A_p = self.compute_momentum_diagonal(cell);

            // Compute divergence of u*
            let mut div_u_star = 0.0;
            for &face in &self.mesh.cell_faces[cell] {
                let u_f = self.interpolate_to_face(u_star, face);
                let normal = &self.mesh.face_normals[face];
                let area = self.mesh.face_areas[face];
                let flux = (u_f[0] * normal[0] + u_f[1] * normal[1] + u_f[2] * normal[2]) * area;

                if self.mesh.face_owner[face] == cell {
                    div_u_star += flux;
                } else {
                    div_u_star -= flux;
                }
            }

            // RHS: enforce mass conservation
            b[cell] = -self.density[cell] * div_u_star;

            // Laplacian coefficients for pressure correction
            for &face in &self.mesh.cell_faces[cell] {
                if let Some(nb) = self.mesh.face_neighbor(face, cell) {
                    let area = self.mesh.face_areas[face];
                    let dist = self.mesh.face_distances[face];
                    let coeff = area / (dist * A_p);
                    A.add(cell, nb, -coeff);
                    A.add(cell, cell, coeff);
                }
            }
        }

        // Set reference pressure at one cell (pressure is determined up to constant)
        A.set_row(0, &[(0, 1.0)]);
        b[0] = 0.0;

        // Solve for pressure correction
        let p_corr = solve_sparse(&A, &b);
        self.pressure_correction = ScalarField::new(p_corr);
    }

    fn correct_velocity(&mut self, u_star: &VectorField) {
        for cell in 0..self.mesh.n_cells {
            let A_p = self.compute_momentum_diagonal(cell);
            let grad_p_corr = self.compute_gradient(&self.pressure_correction, cell);

            self.velocity[cell][0] = u_star[cell][0] - (self.dt / A_p) * grad_p_corr[0];
            self.velocity[cell][1] = u_star[cell][1] - (self.dt / A_p) * grad_p_corr[1];
            self.velocity[cell][2] = u_star[cell][2] - (self.dt / A_p) * grad_p_corr[2];
        }
    }

    fn correct_pressure(&mut self) {
        let alpha_p = 0.3;  // Under-relaxation factor
        for cell in 0..self.mesh.n_cells {
            self.pressure[cell] += alpha_p * self.pressure_correction[cell];
        }
    }

    fn solve_energy(&mut self) {
        // Energy equation: ∂(ρh)/∂t + ∇·(ρuh) = ∇·(λ∇T) + Σ hₖ ω̇ₖ
        let mut A = CsrMatrix::zeros(self.mesh.n_cells, self.mesh.n_cells);
        let mut b = vec![0.0; self.mesh.n_cells];

        for cell in 0..self.mesh.n_cells {
            let rho = self.density[cell];
            let vol = self.mesh.volumes[cell];
            let cp = self.compute_mixture_cp(cell);

            // Transient + convection
            let diag = rho * cp * vol / self.dt;
            A.add(cell, cell, diag);

            // Diffusion (thermal conductivity)
            let lambda = 0.025;  // Simplified
            for &face in &self.mesh.cell_faces[cell] {
                if let Some(nb) = self.mesh.face_neighbor(face, cell) {
                    let area = self.mesh.face_areas[face];
                    let dist = self.mesh.face_distances[face];
                    let coeff = lambda * area / dist;
                    A.add(cell, nb, -coeff);
                    A.add(cell, cell, coeff);
                }
            }

            // Source: heat release from chemistry (computed in chemistry step)
            let heat_release = self.compute_heat_release(cell);
            b[cell] = rho * cp * self.temperature[cell] * vol / self.dt + heat_release * vol;
        }

        let T_new = solve_sparse(&A, &b);
        self.temperature = ScalarField::new(T_new);
    }

    fn solve_species_transport(&mut self, species_idx: usize) {
        // Species equation: ∂(ρYₖ)/∂t + ∇·(ρuYₖ) = ∇·(ρD∇Yₖ) + ω̇ₖWₖ
        let mut A = CsrMatrix::zeros(self.mesh.n_cells, self.mesh.n_cells);
        let mut b = vec![0.0; self.mesh.n_cells];

        for cell in 0..self.mesh.n_cells {
            let rho = self.density[cell];
            let vol = self.mesh.volumes[cell];

            // Transient
            let diag = rho * vol / self.dt;
            A.add(cell, cell, diag);

            // Diffusion
            let D_eff = 1e-5;  // Simplified diffusivity
            for &face in &self.mesh.cell_faces[cell] {
                if let Some(nb) = self.mesh.face_neighbor(face, cell) {
                    let area = self.mesh.face_areas[face];
                    let dist = self.mesh.face_distances[face];
                    let coeff = rho * D_eff * area / dist;
                    A.add(cell, nb, -coeff);
                    A.add(cell, cell, coeff);
                }
            }

            // Source from chemistry (updated in chemistry integration)
            let Y_old = self.species[species_idx][cell];
            b[cell] = rho * Y_old * vol / self.dt;
        }

        let Y_new = solve_sparse(&A, &b);

        // Normalize to ensure Σ Yₖ = 1
        for cell in 0..self.mesh.n_cells {
            self.species[species_idx][cell] = Y_new[cell].max(0.0).min(1.0);
        }
    }

    fn integrate_chemistry(&mut self) {
        // Prepare data for GPU chemistry integration
        let n_species = self.mechanism.species.len();
        let n_cells = self.mesh.n_cells;

        let mut Y_flat = vec![0.0; n_cells * n_species];
        for cell in 0..n_cells {
            for k in 0..n_species {
                Y_flat[cell * n_species + k] = self.species[k][cell];
            }
        }

        let T_vec: Vec<f64> = self.temperature.data.clone();
        let rho_vec: Vec<f64> = self.density.data.clone();

        // Call GPU chemistry kernel (placeholder — actual implementation uses CUDA)
        let (Y_new, T_new, omega) = crate::chemistry::gpu::integrate_batch(
            &Y_flat,
            &T_vec,
            &rho_vec,
            self.dt,
            &self.mechanism,
        );

        // Update fields from GPU results
        for cell in 0..n_cells {
            for k in 0..n_species {
                self.species[k][cell] = Y_new[cell * n_species + k];
            }
            self.temperature[cell] = T_new[cell];
        }
    }

    fn update_density(&mut self) {
        // Ideal gas law: ρ = pW̄/(RT)
        for cell in 0..self.mesh.n_cells {
            let T = self.temperature[cell];
            let p = self.pressure[cell];

            // Compute mean molecular weight
            let mut mean_mw_inv = 0.0;
            for k in 0..self.mechanism.species.len() {
                let Y_k = self.species[k][cell];
                let W_k = self.mechanism.species[k].molecular_weight;
                mean_mw_inv += Y_k / W_k;
            }
            let mean_mw = 1.0 / mean_mw_inv;

            self.density[cell] = p * mean_mw / (8314.46 * T);
        }
    }

    fn compute_residuals(&self) -> IterationResult {
        // Compute L2 residuals for velocity, pressure, temperature
        let mut u_res = 0.0;
        let mut p_res = 0.0;
        let mut T_res = 0.0;

        // (Simplified residual computation)

        IterationResult {
            iteration: self.iteration,
            u_residual: u_res,
            p_residual: p_res,
            T_residual: T_res,
            converged: u_res < 1e-5 && p_res < 1e-5 && T_res < 1e-5,
        }
    }

    // Helper methods (simplified implementations)
    fn compute_convection_coeff(&self, cell: usize) -> f64 { 0.0 }
    fn compute_gradient(&self, field: &ScalarField, cell: usize) -> [f64; 3] { [0.0; 3] }
    fn compute_momentum_diagonal(&self, cell: usize) -> f64 { 1.0 }
    fn interpolate_to_face(&self, field: &VectorField, face: usize) -> [f64; 3] { [0.0; 3] }
    fn solve_turbulence_k(&mut self) {}
    fn solve_turbulence_epsilon(&mut self) {}
    fn update_turbulent_viscosity(&mut self) {}
    fn compute_mixture_cp(&self, cell: usize) -> f64 { 1000.0 }
    fn compute_heat_release(&self, cell: usize) -> f64 { 0.0 }
}

pub struct IterationResult {
    pub iteration: u32,
    pub u_residual: f64,
    pub p_residual: f64,
    pub T_residual: f64,
    pub converged: bool,
}

// Sparse linear solver (placeholder)
fn solve_sparse(A: &CsrMatrix<f64>, b: &[f64]) -> Vec<f64> {
    // Use conjugate gradient or BiCGStab
    vec![0.0; b.len()]
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–5)

**Day 1: Project initialization and Rust backend scaffold**
```bash
cargo init combustiq-api
cd combustiq-api
cargo add axum tokio serde serde_json sqlx uuid chrono tower-http jsonwebtoken bcrypt
cargo add --features postgres,runtime-tokio-rustls,uuid,chrono,json sqlx
cargo add aws-sdk-s3 redis ndarray
```
- `src/main.rs` — Axum app with CORS, tracing, graceful shutdown
- `src/config.rs` — Environment-based configuration (DATABASE_URL, JWT_SECRET, S3_BUCKET, REDIS_URL, GPU_ENDPOINT)
- `src/state.rs` — AppState struct (PgPool, Redis, S3Client)
- `src/error.rs` — ApiError enum with IntoResponse for combustion-specific errors
- `Dockerfile` — Multi-stage build (builder + runtime with CUDA support)
- `docker-compose.yml` — PostgreSQL, Redis, MinIO (S3-compatible), Grafana

**Day 2: Database schema and migrations**
- `migrations/001_initial.sql` — All 10 tables: users, organizations, org_members, mechanisms, reduced_mechanisms, projects, simulations, simulation_jobs, validation_benchmarks, usage_records
- `src/db/mod.rs` — Database pool initialization with connection pooling
- `src/db/models.rs` — All SQLx structs with FromRow derives for combustion-specific types
- Run `sqlx migrate run` to apply schema
- Seed script for built-in mechanisms: GRI-Mech 3.0, H2/O2, NH3 (upload to S3, insert metadata)

**Day 3: Authentication system**
- `src/auth/mod.rs` — JWT token generation and validation middleware
- `src/auth/oauth.rs` — Google and GitHub OAuth 2.0 flow handlers
- `src/api/handlers/auth.rs` — Register, login, OAuth callback, refresh token, me endpoints
- Password hashing with bcrypt (cost factor 12)
- JWT with 24h access token + 30d refresh token
- Auth middleware that extracts Claims from Authorization header
- Plan-based authorization checks (academic/pro/enterprise)

**Day 4: User and organization CRUD**
- `src/api/handlers/users.rs` — Get profile, update profile, delete account, list users in org
- `src/api/handlers/orgs.rs` — Create org, invite member, list members, remove member, update org settings
- `src/api/router.rs` — All route definitions with auth middleware
- Integration tests: auth flow, org CRUD, authorization checks for academic labs

**Day 5: Mechanism and project models**
- `src/api/handlers/mechanisms.rs` — List mechanisms, get mechanism, upload mechanism, search mechanisms
- `src/api/handlers/projects.rs` — Create, list, get, update, delete, fork project
- S3 integration for mechanism file storage (Chemkin/Cantera YAML upload)
- Mechanism metadata extraction: count species/reactions from file
- Project configuration JSONB validation for reactor vs CFD types

### Phase 2 — Chemistry Core (Days 6–12)

**Day 6: Mechanism parser — Cantera YAML**
- `src/chemistry/mechanism.rs` — Mechanism, Species, Reaction, RateConstant structs
- `src/chemistry/parser/cantera.rs` — Parse Cantera YAML format (species, reactions, phases)
- `serde_yaml` integration for robust YAML parsing
- Extract NASA polynomials for thermodynamics (7-coefficient format)
- Extract Arrhenius parameters for reactions (A, β, Ea)
- Unit tests: parse GRI-Mech 3.0, verify species count (53), reactions (325)

**Day 7: Mechanism parser — Chemkin format**
- `src/chemistry/parser/chemkin.rs` — Parse legacy Chemkin text format
- Sections: ELEMENTS, SPECIES, THERMO, REACTIONS
- Handle continuation lines, comments, mixed case
- Support for three-body reactions (M), falloff reactions (LOW/TROE), pressure-dependent reactions
- Unit tests: parse classic Chemkin mechanisms (H2/O2, CO/H2/O2)

**Day 8: Thermodynamics — NASA polynomials**
- `src/chemistry/thermo.rs` — ThermoDatabase implementation
- Methods: `enthalpy(species, T)`, `entropy(species, T)`, `gibbs(species, T)`, `cp(species, T)`
- Temperature range handling: low-T (200-1000K) vs high-T (1000-6000K) polynomials
- Mixture properties: mean molecular weight, mixture Cp, mixture enthalpy
- Unit tests: verify H2O enthalpy at 298K matches JANAF tables, Cp continuity at 1000K

**Day 9: Kinetics — reaction rates**
- `src/chemistry/kinetics.rs` — Reaction rate evaluation
- Forward rate: Arrhenius k_f = A T^β exp(-Ea/RT)
- Reverse rate: k_r = k_f / K_eq from Gibbs free energy
- Three-body enhancement: effective concentration [M] = Σ α_i [X_i]
- Falloff reactions: Lindemann and Troe blending functions
- Unit tests: verify H + O2 <=> OH + O rate at 1500K matches literature

**Day 10: 0D reactor — constant pressure**
- `src/chemistry/reactor/constant_pressure.rs` — ConstantPressureReactor implementation
- ODE system: dY_k/dt = ω̇_k/ρ, dT/dt = -(Σ h_k ω̇_k)/(ρC_p)
- Production rate calculation: loop over reactions, accumulate ν_kj * (rate_f - rate_r)
- Density from ideal gas law: ρ = pW̄/(RT)
- Integration interface (prepared for CVODE, initially simple Euler)
- Unit tests: H2/O2 ignition at 1000K/1atm, verify temperature rise

**Day 11: 0D reactor — constant volume and PSR**
- `src/chemistry/reactor/constant_volume.rs` — ConstantVolumeReactor (dT/dt different due to dV=0)
- `src/chemistry/reactor/psr.rs` — PerfectlyStirredReactor with residence time τ
- PSR ODEs: dY_k/dt = (Y_k^in - Y_k)/τ + ω̇_k/ρ
- Stiff ODE solver integration (CVODE wrapper via `sundials-sys` or pure Rust `diffsol`)
- Unit tests: PSR extinction strain rate, verify against Cantera reference

**Day 12: Laminar flame speed and ignition delay calculators**
- `src/chemistry/analysis/ignition_delay.rs` — Compute ignition delay (time to T₀ + 400K)
- `src/chemistry/analysis/flame_speed.rs` — 1D flame solver for laminar flame speed S_L
- Flame speed via shooting method or premixed flame solution (simplified BVP solver)
- API endpoints: POST /analysis/ignition-delay, POST /analysis/flame-speed
- Unit tests: GRI-Mech 3.0 methane/air at φ=1.0, verify S_L ≈ 37 cm/s at 298K

### Phase 3 — Mechanism Reduction (Days 13–16)

**Day 13: Python FastAPI service for mechanism reduction**
```bash
mkdir python-services/mechanism_reduction
cd python-services/mechanism_reduction
python -m venv venv
source venv/bin/activate
pip install fastapi uvicorn cantera numpy scipy
```
- `app.py` — FastAPI app with /reduce/drg endpoint
- Cantera integration for mechanism loading and validation
- S3 download/upload for mechanism files (boto3)
- Docker container for Python service

**Day 14: DRG algorithm implementation**
- `reduction/drg.py` — Directed Relation Graph reduction algorithm
- Build interaction coefficient matrix: r_AB = max_j |ν_Aj ω_j| / max_k |ν_Ak ω_k|
- Graph search from target species (fuel, O2, major products)
- Threshold tuning based on error tolerance
- Species/reaction filtering to produce reduced mechanism
- Unit tests: reduce GRI-Mech 3.0 from 53 to ~20 species, verify error < 15%

**Day 15: DRGEP (with error propagation) and sensitivity analysis**
- `reduction/drgep.py` — DRGEP extension that propagates path-dependent errors
- `reduction/sensitivity.py` — Brute-force sensitivity: remove each species, measure error
- Validation against experimental data: ignition delay, flame speed, extinction
- Error metrics: max relative error over all validation conditions
- Integration with Rust backend: call Python service via HTTP

**Day 16: Reduction API and frontend integration**
- `src/api/handlers/reduction.rs` — Create reduction job, poll status, download reduced mechanism
- Async job queue: submit reduction to Python worker via Redis
- Progress tracking: number of species tested, current error estimate
- Frontend: `src/pages/MechanismReduction.tsx` — Upload mechanism, select targets, run reduction
- Display original vs reduced species/reactions, error metrics table

### Phase 4 — CFD Solver Foundation (Days 17–24)

**Day 17: Mesh data structures**
- `src/cfd/mesh.rs` — Mesh struct (cells, faces, vertices)
- Unstructured mesh support: tetrahedra, hexahedra, prisms
- Face-based data structure for finite volume method
- Geometric computations: cell volume, face area, face normal, cell centroid
- Mesh I/O: import Gmsh .msh format, export VTK for visualization
- Unit tests: cube mesh, verify volumes sum correctly

**Day 18: Finite volume discretization**
- `src/cfd/fvm.rs` — Finite volume method framework
- Convection: upwind, central differencing, QUICK schemes
- Diffusion: central differencing with non-orthogonal correction
- Gradient reconstruction: Green-Gauss cell-based, least-squares
- Interpolation: linear, upwind for convection-diffusion
- Unit tests: 1D heat conduction, compare to analytical solution

**Day 19: SIMPLE algorithm for pressure-velocity coupling**
- `src/cfd/simple.rs` — SIMPLE (Semi-Implicit Method for Pressure-Linked Equations)
- Momentum predictor: solve momentum with guessed pressure
- Pressure correction: Poisson equation ∇·[(1/a_p)∇p''] = ∇·u*
- Velocity correction: u = u* - (Δt/a_p)∇p''
- Under-relaxation for stability: α_u = 0.7, α_p = 0.3
- Sparse linear solver: conjugate gradient (CG) for symmetric, BiCGSTAB for asymmetric
- Unit tests: lid-driven cavity, verify velocity profile

**Day 20: Turbulence models — k-ε**
- `src/cfd/turbulence/k_epsilon.rs` — Standard k-ε turbulence model
- Transport equations for k (turbulent kinetic energy) and ε (dissipation rate)
- Turbulent viscosity: μ_t = C_μ ρ k²/ε
- Wall functions: log-law for near-wall treatment
- Production and dissipation terms
- Unit tests: channel flow, verify k and ε profiles

**Day 21: Turbulence models — k-ω SST**
- `src/cfd/turbulence/k_omega_sst.rs` — k-ω SST (Shear Stress Transport) model
- Blending between k-ω (near wall) and k-ε (free stream)
- Cross-diffusion term, blending function F1
- Superior for adverse pressure gradients and separating flows
- Unit tests: backward-facing step, compare to DNS data

**Day 22: Species transport equations**
- `src/cfd/species.rs` — Scalar transport for mass fractions Y_k
- Convection-diffusion equation: ∂(ρY_k)/∂t + ∇·(ρuY_k) = ∇·(ρD_k∇Y_k) + ω̇_k W_k
- Bounded schemes to ensure 0 ≤ Y_k ≤ 1
- Diffusion coefficient: laminar D_k + turbulent D_t = μ_t/(ρ Sc_t)
- Source term from chemistry (ω̇_k from mechanism)
- Unit tests: diffusion flame, verify species profiles

**Day 23: Energy equation with combustion**
- `src/cfd/energy.rs` — Enthalpy transport
- Energy equation: ∂(ρh)/∂t + ∇·(ρuh) = ∇·(λ∇T) + ∇·(Σ h_k J_k) - Σ ω̇_k W_k h_k
- Temperature from enthalpy: iterative solve h = Σ Y_k h_k(T)
- Heat release from chemistry: -Σ ω̇_k W_k h_k
- Radiation models: optically thin (simple), discrete ordinates (advanced)
- Unit tests: adiabatic flame temperature, verify against CEA

**Day 24: EDC combustion model**
- `src/cfd/combustion/edc.rs` — Eddy Dissipation Concept implementation
- Fine structure volume fraction: ξ* = C_ξ (νε/k²)^0.5
- Fine structure residence time: τ* = C_τ (ν/ε)^0.5
- Chemistry integration in fine structures: solve 0D reactor for time τ*
- Mean reaction rate: ⟨ω̇_k⟩ = ρ (ξ*²)/(τ*(1 - ξ*³)) (Y_k* - Y_k)
- Cell-by-cell chemistry calls (prepared for GPU acceleration)
- Unit tests: premixed flame, verify flame thickness

### Phase 5 — GPU Chemistry Acceleration (Days 25–29)

**Day 25: CUDA kernel for chemistry integration**
- `src/gpu/chemistry_kernel.cu` — CUDA kernel for parallel ODE integration
- One thread block per batch of cells (256-1024 cells)
- Warp-level parallelism: each warp solves one cell's chemistry
- Shared memory for mechanism data: rate constants, thermodynamic coefficients
- Global memory for cell state: Y_k, T, ρ
- Launch kernel: `chemistry_kernel<<<n_blocks, block_size>>>(state, mechanism, dt)`

**Day 26: GPU ODE solver (CVODE-inspired BDF)**
- `src/gpu/ode_solver.cu` — Backward differentiation formula (BDF) on GPU
- BDF2 for stiff ODEs: (3y^(n+1) - 4y^n + y^(n-1))/(2Δt) = f(y^(n+1))
- Newton iteration: J Δy = -F where J = ∂f/∂y (Jacobian)
- Direct dense solve per cell (53×53 for GRI-Mech) via cuBLAS batched operations
- Adaptive timestep: error estimation, accept/reject, adjust dt
- Unit tests: integrate H2/O2 kinetics on GPU, compare to CPU CVODE

**Day 27: GPU-CPU data transfer and batching**
- `src/gpu/transfer.rs` — Rust bindings to CUDA via `cuda-sys` or `cudarc`
- Allocate device memory: `cudaMalloc` for state arrays
- Transfer host → device: cell temperatures, mass fractions, densities
- Execute chemistry kernel asynchronously
- Transfer device → host: updated Y_k, T, ω̇_k
- Overlap transfers with computation via CUDA streams
- Benchmarks: 1M cells, GRI-Mech 3.0, measure speedup vs CPU

**Day 28: Integration into CFD solver**
- `src/cfd/combustion/edc_gpu.rs` — GPU-accelerated EDC model
- Before CFD timestep: gather cell states into contiguous arrays
- Call GPU chemistry kernel for all cells in parallel
- After kernel: scatter results back to cells
- Fallback to CPU for small meshes (<10K cells)
- Validation: compare GPU vs CPU results for identical mesh/mechanism

**Day 29: GPU performance optimization**
- Memory coalescing: ensure aligned access patterns
- Occupancy optimization: tune block size (128/256/512 threads)
- Register usage: reduce spilling to local memory
- Shared memory: cache mechanism data, reduce global memory reads
- Multi-GPU support: partition mesh across GPUs for very large cases
- Benchmark suite: 10K, 100K, 1M cells; GRI-Mech 3.0, H2 (10 species), detailed (200 species)
- Target: 10x speedup for GRI-Mech 3.0, 50x for detailed mechanisms

### Phase 6 — Frontend Visualization (Days 30–35)

**Day 30: Frontend scaffold and project setup**
```bash
npm create vite@latest frontend -- --template react-ts
cd frontend
npm i zustand @tanstack/react-query axios three @react-three/fiber @react-three/drei
npm i plotly.js react-plotly.js recharts
```
- `src/App.tsx` — Router, auth context, layout with sidebar
- `src/stores/projectStore.ts` — Zustand store for project state
- `src/stores/simulationStore.ts` — Simulation state (status, progress, results)
- `src/pages/Dashboard.tsx` — Project list, create new, recent simulations
- `src/components/Layout.tsx` — Top nav, sidebar, main content area

**Day 31: Mechanism library and upload**
- `src/pages/Mechanisms.tsx` — Browse built-in mechanisms, search, upload custom
- `src/components/MechanismCard.tsx` — Display mechanism details (species, reactions, fuel type)
- `src/components/MechanismUpload.tsx` — Drag-and-drop Chemkin/YAML upload
- Mechanism validation: parse file client-side (basic), full validation server-side
- Tag filtering: methane, hydrogen, ammonia, syngas, jet_a
- Download mechanism files, view thermodynamic data

**Day 32: 0D reactor configuration UI**
- `src/pages/ReactorSimulation.tsx` — Configure constant-P, constant-V, PSR, PFR
- `src/components/ReactorConfig.tsx` — Input fields: T, P, φ, fuel, oxidizer, diluent
- `src/components/CompositionEditor.tsx` — Species mole fractions editor
- Preset compositions: stoichiometric CH4/air, lean H2/air, rich syngas
- Submit simulation, WebSocket for progress
- Display results: temperature vs time, species vs time (Plotly line charts)

**Day 33: Ignition delay and flame speed tools**
- `src/pages/MechanismValidation.tsx` — Validation tools dashboard
- `src/components/IgnitionDelayPlot.tsx` — Ignition delay vs temperature (Arrhenius plot)
- `src/components/FlameSpeedPlot.tsx` — Laminar flame speed vs equivalence ratio
- Compare against experimental data (overlay benchmarks)
- Batch analysis: sweep T or φ, plot family of curves
- Export data as CSV

**Day 34: 3D CFD mesh viewer (WebGPU)**
- `src/components/MeshViewer.tsx` — 3D mesh visualization with @react-three/fiber
- Load mesh from S3 (VTK format), parse on client
- Render: wireframe, surface, edges
- Camera controls: orbit, pan, zoom (drei OrbitControls)
- Mesh statistics overlay: cell count, min/max volume, aspect ratio
- Boundary coloring: inlet (blue), outlet (red), wall (gray)

**Day 35: 3D solution field visualization (WebGPU contours)**
- `src/components/FieldViewer.tsx` — Scalar field visualization (temperature, species, velocity)
- WebGPU compute shader for contour generation
- Color maps: jet, viridis, inferno for temperature; custom for species
- Slice planes: XY, YZ, XZ slices through 3D domain
- Iso-surfaces: render constant-temperature surfaces (flame front)
- Streamlines: particle traces colored by temperature
- Animation: play through transient timesteps

### Phase 7 — Simulation Workflow (Days 36–39)

**Day 36: CFD project configuration UI**
- `src/pages/CfdProject.tsx` — Configure 2D/3D reacting flow case
- `src/components/MeshUpload.tsx` — Upload mesh (.msh, .vtk, .stl)
- `src/components/BoundaryConditions.tsx` — Set inlet (velocity, composition, T), outlet (pressure), wall (adiabatic/isothermal)
- `src/components/TurbulenceConfig.tsx` — Select k-ε or k-ω SST, set initial k and ε
- `src/components/CombustionConfig.tsx` — Select EDC, set model constants
- `src/components/SolverSettings.tsx` — Timestep, end time, convergence tolerance, output frequency

**Day 37: Simulation job submission and monitoring**
- `src/api/handlers/simulation.rs` — Create CFD simulation, validate configuration
- Job queueing: submit to Redis, priority based on plan (enterprise > pro > academic)
- Worker allocation: assign GPU type (V100 / A100 / CPU-only) based on mesh size
- WebSocket progress: iteration number, residuals, current time, max temperature
- `src/components/SimulationMonitor.tsx` — Real-time residual plots, convergence metrics
- Email notification on completion

**Day 38: Results post-processing**
- `src/api/handlers/results.rs` — Download solution fields from S3, extract slices
- `src/components/ResultsPanel.tsx` — Field selector, time selector for transients
- Derived fields: vorticity magnitude, heat release rate, flame index
- Quantitative probes: line plots (temperature along centerline), point monitors
- Export: VTK for ParaView, PNG for reports, CSV for data

**Day 39: Emissions prediction and reporting**
- `src/components/EmissionsPanel.tsx` — Display NOx, CO, soot, UHC concentrations
- Zeldovich NOx calculation: thermal NOx formation pathway
- Emission indices: g/kg_fuel for NOx, CO
- 3D visualization: pollutant formation zones (highlight high-NOx regions)
- Compliance check: compare against EPA/Euro standards
- PDF report generation: summary, operating conditions, emissions table, contour plots

### Phase 8 — Validation and Launch (Days 40–42)

**Day 40: Solver validation benchmarks**
- Benchmark 1: 0D constant-P reactor, GRI-Mech 3.0, φ=1.0, T=1000K, P=1atm — verify ignition delay ≈ 0.85 ms (vs Cantera 0.84 ms)
- Benchmark 2: Laminar flame speed, CH4/air, φ=1.0, 298K — verify S_L ≈ 37 cm/s (vs exp. 38 cm/s)
- Benchmark 3: 2D premixed flame, compare temperature profile to FlameMaster reference
- Benchmark 4: 2D diffusion flame (Burke-Schumann), verify mixture fraction and flame location
- Benchmark 5: 3D swirl burner with EDC, compare to experimental PIV/PLIF data (temperature, OH)
- Error metrics: L2 norm of temperature difference < 5%, ignition delay error < 10%

**Day 41: End-to-end testing and performance**
- E2E test 1: Create account → upload mechanism → run 0D reactor → view results
- E2E test 2: Create CFD project → upload mesh → configure BC → run simulation → visualize
- E2E test 3: Mechanism reduction → validate reduced mechanism → use in CFD
- Load testing: 100 concurrent users, 1000 simulations queued
- GPU worker scaling: verify auto-scaling based on queue depth
- Storage limits: 10GB per academic user, 100GB per pro, unlimited enterprise

**Day 42: Documentation and launch prep**
- User documentation: quickstart guide, tutorials (0D reactor, 2D flame, mechanism reduction)
- API documentation: OpenAPI spec for REST endpoints
- Mechanism library: populate with 20+ built-in mechanisms (GRI-Mech 3.0, H2/syngas, NH3, ethanol, n-heptane, iso-octane, Jet-A surrogates)
- Validation database: 50+ benchmark cases from literature
- Billing integration: Stripe subscription webhooks, usage metering
- Marketing site: landing page, pricing, case studies (gas turbine, IC engine)
- Launch: ProductHunt, Hacker News, post to combustion engineering forums

---

## Validation Benchmarks

### Benchmark 1: Ignition Delay Time — Hydrogen/Air

**Conditions:**
- Fuel: H2/air
- Equivalence ratio: φ = 1.0 (stoichiometric)
- Temperature: 1000K, 1200K, 1500K
- Pressure: 1 atm, 10 atm
- Reactor: Constant volume

**Expected Results (from experimental data):**
- T=1000K, P=1atm: τ_ign ≈ 3.2 ms
- T=1200K, P=1atm: τ_ign ≈ 0.42 ms
- T=1500K, P=1atm: τ_ign ≈ 0.055 ms
- T=1000K, P=10atm: τ_ign ≈ 0.85 ms

**Acceptance Criteria:**
- CombustiQ result within ±15% of experimental value
- Matches Cantera reference simulation within ±5%

### Benchmark 2: Laminar Flame Speed — Methane/Air

**Conditions:**
- Fuel: CH4/air (GRI-Mech 3.0)
- Equivalence ratio: φ = 0.6, 0.8, 1.0, 1.2, 1.4
- Temperature: 298K (unburned)
- Pressure: 1 atm
- Solver: 1D premixed flame

**Expected Results (from experimental data):**
- φ=0.6: S_L ≈ 12 cm/s
- φ=0.8: S_L ≈ 26 cm/s
- φ=1.0: S_L ≈ 37 cm/s
- φ=1.2: S_L ≈ 35 cm/s
- φ=1.4: S_L ≈ 25 cm/s

**Acceptance Criteria:**
- CombustiQ result within ±10% of experimental value at each φ
- Peak flame speed at φ ≈ 1.05 (slightly rich)

### Benchmark 3: 2D Laminar Diffusion Flame

**Conditions:**
- Fuel: CH4 jet (center), air co-flow
- Velocity: 0.5 m/s (fuel), 0.1 m/s (air)
- Temperature: 300K (inlet)
- Mesh: 50K cells, refined near flame
- Chemistry: GRI-Mech 3.0 (53 species)

**Expected Results:**
- Flame height: 8-10 cm
- Peak temperature: 2200-2300 K (slightly below adiabatic due to radiation)
- Stoichiometric contour (Z=0.055): matches flame brush
- OH concentration: peak at stoichiometric mixture fraction

**Acceptance Criteria:**
- Flame height within ±15% of experimental measurement
- Peak temperature within ±100K
- OH profile correlates with heat release (R² > 0.9)

### Benchmark 4: Mechanism Reduction — GRI-Mech 3.0

**Reduction Task:**
- Original: GRI-Mech 3.0, 53 species, 325 reactions
- Target: ~20 species (CH4, O2, N2, CO2, H2O, CO, H2, OH, H, O, HO2, H2O2, CH3, CH2O, HCO, C2H4, C2H6, NO + Ar)
- Method: DRG with error tolerance 15%
- Validation conditions: T=1000-1500K, P=1-10atm, φ=0.7-1.3

**Expected Results:**
- Reduced mechanism: 18-22 species, 80-120 reactions
- Ignition delay error: <15% max, <8% average
- Flame speed error: <12% max, <6% average
- Reduction speedup in CFD: 2-3x (fewer species ODE)

**Acceptance Criteria:**
- Reduced mechanism species count: 18-24
- Ignition delay error < 15% for all validation conditions
- Flame speed error < 12% for φ=0.7-1.3

### Benchmark 5: GPU Chemistry Acceleration

**Test Case:**
- Mesh: 1M cells
- Mechanism: GRI-Mech 3.0 (53 species, 325 reactions)
- CFD timestep: 1000 chemistry integrations (EDC)
- Hardware: NVIDIA V100 GPU vs 32-core CPU

**Expected Results:**
- CPU time: ~180 seconds per CFD timestep
- GPU time: ~8-12 seconds per CFD timestep
- Speedup: 15-22x
- Numerical difference: <0.1% (GPU vs CPU temperature/species)

**Acceptance Criteria:**
- GPU speedup > 10x for GRI-Mech 3.0
- GPU speedup > 25x for detailed mechanisms (>150 species)
- Temperature difference (GPU - CPU) RMS < 5K
- Mass fraction difference RMS < 1e-4

---

## Post-MVP Roadmap

### v1.1 — Large Eddy Simulation (LES) (Weeks 13-16)
- Smagorinsky, WALE, dynamic SGS turbulence models
- Thickened flame model for LES combustion
- Transient simulation with adaptive timestep
- Turbulent flame visualization (Q-criterion vortices, heat release iso-surfaces)

### v1.2 — Spray Combustion (Weeks 17-20)
- Lagrangian particle tracking for fuel droplets
- Breakup models: KH-RT, WAVE
- Evaporation: rapid mixing, multi-component
- Spray-wall interaction: Bai-Gosman
- GDI and diesel injection workflows

### v1.3 — Soot and Advanced Emissions (Weeks 21-23)
- Soot models: two-equation (Moss-Brookes), method of moments (MOMIC)
- Prompt NOx, fuel-NOx, N2O pathway
- Particulate matter size distribution
- Polycyclic aromatic hydrocarbons (PAH)

### v1.4 — Thermoacoustic Instabilities (Weeks 24-26)
- Helmholtz solver for acoustic modes
- Flame transfer function extraction from LES
- Linear stability analysis
- Rayleigh index calculation

### v1.5 — IC Engine and Gas Turbine Workflows (Weeks 27-30)
- IC engine: moving mesh, valve motion, piston motion, HCCI/RCCI
- Gas turbine: swirl burner, dilution jets, liner cooling, emissions staging
- Industry-specific reporting templates

### Enterprise Features
- On-premise / air-gapped deployment (Docker Swarm / Kubernetes)
- Custom mechanism development service
- Integration with test bench data (DAS systems, SCADA)
- Dedicated HPC allocation (reserved GPU capacity)
- White-label branding

---

## Performance Targets

| Metric | Target | Notes |
|--------|--------|-------|
| 0D reactor (GRI-Mech 3.0, 1s) | <5 seconds | Including chemistry integration to convergence |
| Ignition delay calculation | <10 seconds | Single condition, constant-V reactor |
| Laminar flame speed | <30 seconds | 1D premixed flame, adaptive grid |
| Mechanism reduction (GRI-Mech) | <5 minutes | DRG, 10 validation conditions |
| 2D CFD (100K cells, 100 steps) | <20 minutes | RANS, EDC, GRI-Mech 3.0, GPU-accelerated |
| 3D CFD (1M cells, 1000 steps) | <8 hours | RANS, EDC, GRI-Mech 3.0, single V100 GPU |
| GPU chemistry speedup | >10x | GRI-Mech 3.0 vs CPU baseline |
| Field visualization load time | <3 seconds | 1M cells, single scalar field, WebGPU |

---

## Risk Mitigation

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| GPU chemistry accuracy issues | Medium | High | Extensive validation against CPU CVODE, unit tests per mechanism, double-precision on GPU |
| Mechanism parser bugs (Chemkin) | High | Medium | Parse 50+ mechanisms from literature, compare to Cantera, fallback to Cantera Python for validation |
| CFD convergence failures | High | High | Implement source stepping, under-relaxation tuning, provide solver diagnostics, fallback to laminar |
| GPU availability / cost | Medium | High | Auto-scaling GPU workers, queue management, CPU fallback, reserved capacity for Enterprise |
| Mechanism licensing (proprietary) | Low | Medium | Focus on open mechanisms (GRI-Mech, LLNL, CRECK), require users to upload proprietary if needed |
| Competitive response (ANSYS) | Low | Low | Target underserved academic/SME markets, differentiate on GPU acceleration and cloud-native |

---

## Dependencies

**Critical Path:**
1. Chemistry core (mechanism parser + thermodynamics + kinetics + 0D reactor) → Days 6-12
2. CFD solver foundation (mesh + FVM + SIMPLE + turbulence) → Days 17-24
3. GPU chemistry integration → Days 25-29
4. Frontend visualization (WebGPU) → Days 30-35

**Parallel Tracks:**
- Authentication + database can proceed independently → Days 1-5
- Mechanism reduction service (Python) can be developed in parallel with Rust chemistry → Days 13-16
- Frontend project/mechanism management can be built while CFD solver develops → Days 30-31

**External Dependencies:**
- Cantera Python library for mechanism reduction validation
- CUDA Toolkit 12.x for GPU chemistry
- Gmsh or OpenFOAM for mesh generation (users provide meshes initially)
- AWS GPU instances (p3/p4) for compute

---

## Success Metrics

### Technical Metrics
- 0D reactor ignition delay within 10% of Cantera for GRI-Mech 3.0
- 1D flame speed within 10% of experimental data for CH4/air
- 2D CFD flame height within 15% of experimental (Sandia Flame D)
- GPU chemistry speedup >10x for 53-species mechanism
- CFD solver convergence rate: >90% of cases converge within 1000 iterations

### User Metrics
- 100 registered users in first month
- 20 active academic labs (5+ users each)
- 1000 0D reactor simulations run
- 100 CFD simulations completed
- Average CFD simulation time: <4 hours for 100K-cell cases

### Business Metrics
- 10 paying Academic subscriptions ($149/mo) in first 3 months
- 5 Pro subscriptions ($499/mo) in first 6 months
- 1 Enterprise customer in first year
- Monthly recurring revenue (MRR): $5K by month 6
- GPU utilization: >60% (cost efficiency)

