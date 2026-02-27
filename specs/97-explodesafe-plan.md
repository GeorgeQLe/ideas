# 97. ExplodeSafe — Energetic Material Simulation Platform

## Implementation Plan

**MVP Scope:** Cloud-based thermochemical equilibrium engine for energetic materials with Gibbs free energy minimization solver implementing NASA's CEA algorithm for 200+ common explosives/propellants, Chapman-Jouguet (CJ) detonation state calculator (velocity, pressure, temperature) with BKW and JCZ3 product equations of state, multi-component formulation builder supporting explosive/oxidizer/binder/metal mixtures with oxygen balance and heat of formation computation, JWL (Jones-Wilkins-Lee) isentrope fitting for hydrocode parameter generation, basic sensitivity lookup table for known compounds from LASL Explosive Properties database, interactive performance visualization with Plotly.js plots (detonation velocity vs. density, pressure vs. composition, Hugoniot curves), validated against 200+ experimental CJ detonation velocities from peer-reviewed literature, species database with 5,000+ JANAF thermodynamic entries stored in PostgreSQL with S3 storage for batch computation results, Stripe billing with three tiers (Free / Pro $149/mo / Advanced $349/user/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| Thermo Engine | Rust (native + WASM) | Custom Gibbs minimization solver with line-search Newton, CJ state iteration |
| WASM Compilation | `wasm-pack` + `wasm-bindgen` | Client-side thermochemical equilibrium for simple formulations |
| QSPR/ML | Python 3.12 (FastAPI) | RDKit molecular descriptors, XGBoost for sensitivity prediction |
| Database | PostgreSQL 16 | Species data, formulations, experimental validation database |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | Batch computation results, user-uploaded species data, exported reports |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| Visualization | Plotly.js + D3.js | Performance plots (VOD, CJ pressure), ternary diagrams, Hugoniot curves |
| 3D Molecular | WebGL (custom) | Molecular structure visualization, crystal packing density estimation |
| Real-time | WebSocket (Axum) | Live batch computation progress, convergence monitoring |
| Job Queue | Redis 7 + Tokio tasks | Batch formulation screening, parameter sweeps |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | WASM bundle delivery, static frontend assets |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, error tracking |
| CI/CD | GitHub Actions | WASM build pipeline, Rust tests, Docker image push |

### Key Architecture Decisions

1. **Hybrid client/server thermochemical solver with WASM threshold at 5-component formulations**: Simple formulations (1-5 components, ideal gas or BKW EOS) run entirely in the browser via WASM, providing instant CJ state computation with zero server cost. Complex formulations (>5 components, JCZ3/JCZS EOS, multi-phase equilibrium with condensed products) are submitted to the Rust-native server solver which handles heavy numerical iteration. This covers 80%+ of quick-check use cases while preserving server resources for advanced simulations.

2. **Custom Gibbs minimization solver in Rust rather than wrapping CEA or Cantera**: Building a custom equilibrium solver in Rust gives us full control over convergence algorithms, EOS integration, WASM compilation, and detonation-specific optimizations. NASA CEA is public domain but written in FORTRAN 77 with extensive global state, making WASM compilation impractical. Cantera's C++ codebase is primarily combustion-focused and lacks validated detonation product EOS (BKW, JCZ3). Our Rust solver uses line-search Newton-Raphson for Gibbs minimization with automatic phase-splitting for condensed products.

3. **Validated BKW and JCZ3 product equations of state rather than ideal gas**: Detonation products at CJ conditions (20-40 GPa, 3000-4500 K) exhibit extreme non-ideal behavior — covolume effects from high density and intermolecular attractions significantly reduce pressure from ideal gas predictions. BKW (Becker-Kistiakowsky-Wilson) and JCZ3 (Jacobs-Cowperthwaite-Zwisler) are empirically validated for detonation products and are the industry standards used by CHEETAH and Explo5. We implement multiple BKW parameterizations (BKWC, BKWR, BKWS) to match different experimental calibration sets.

4. **PostgreSQL species database with JANAF thermodynamic polynomials**: Thermodynamic data (Cp, H, S as functions of temperature) for 5,000+ species are stored as 7-coefficient NASA polynomials in two temperature ranges (200-1000K, 1000-6000K). This format is compact, enables fast evaluation during Gibbs minimization iteration, and is compatible with NASA CEA, CHEMKIN, and Cantera data sources. PostgreSQL's JSONB efficiently stores polynomial coefficients while enabling fast species lookup by formula, name, or chemical category.

5. **JWL isentrope auto-fitting for hydrocode integration**: Users frequently need JWL (Jones-Wilkins-Lee) parameters to model detonation product expansion in hydrocodes (LS-DYNA, AUTODYN, CTH). Rather than requiring manual parameter fitting, we auto-generate JWL parameters from the computed CJ state and product EOS by fitting the isentrope p(v) curve over a physically relevant expansion range (ρ/ρ₀ = 0.1 to 1.0). This automation saves users days of manual calibration work.

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";  -- For fuzzy text search on species names

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

-- Organizations (for Advanced plan multi-user teams)
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

-- Species (thermodynamic data for gaseous, liquid, solid products)
-- Stores NASA polynomial coefficients for Cp, H, S
CREATE TABLE species (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,  -- e.g., "H2O(g)", "CO2(g)", "C(gr)", "Al2O3(s)"
    formula TEXT NOT NULL,  -- Chemical formula e.g., "H2O", "CO2"
    phase TEXT NOT NULL,  -- gas | liquid | solid
    molecular_weight REAL NOT NULL,  -- g/mol
    heat_of_formation_298 REAL NOT NULL,  -- kJ/mol at 298.15 K
    -- NASA 7-coefficient polynomial [a1, a2, a3, a4, a5, a6, a7]
    -- Cp/R = a1 + a2*T + a3*T^2 + a4*T^3 + a5*T^4
    -- H/RT = a1 + a2*T/2 + a3*T^2/3 + a4*T^3/4 + a5*T^4/5 + a6/T
    -- S/R  = a1*ln(T) + a2*T + a3*T^2/2 + a4*T^3/3 + a5*T^4/4 + a7
    nasa_poly_low REAL[] NOT NULL,  -- 200-1000 K range
    nasa_poly_high REAL[] NOT NULL,  -- 1000-6000 K range
    temp_range_low REAL[] NOT NULL DEFAULT '{200, 1000}',  -- [Tmin, Tmax]
    temp_range_high REAL[] NOT NULL DEFAULT '{1000, 6000}',
    source TEXT,  -- JANAF | Gurvich | Burcat | Custom
    notes TEXT,
    is_builtin BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX species_name_idx ON species(name);
CREATE INDEX species_formula_idx ON species(formula);
CREATE INDEX species_phase_idx ON species(phase);
CREATE INDEX species_name_trgm_idx ON species USING gin(name gin_trgm_ops);

-- Reactants (explosives, oxidizers, fuels, binders, metals)
CREATE TABLE reactants (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,  -- e.g., "RDX", "HMX", "TNT", "Ammonium Nitrate", "Aluminum"
    formula TEXT NOT NULL,  -- Empirical formula e.g., "C3H6N6O6" for RDX
    category TEXT NOT NULL,  -- explosive | oxidizer | fuel | binder | metal | plasticizer
    cas_number TEXT,  -- CAS registry number
    density REAL NOT NULL,  -- g/cm³ (crystal or bulk density)
    heat_of_formation REAL NOT NULL,  -- kJ/mol (standard state 298 K)
    molecular_weight REAL NOT NULL,  -- g/mol
    oxygen_balance REAL,  -- % oxygen balance (to CO2)
    -- Elemental composition (atoms per molecule)
    composition JSONB NOT NULL,  -- {"C": 3, "H": 6, "N": 6, "O": 6} for RDX
    sensitivity_data JSONB DEFAULT '{}',  -- {impact_h50, friction_load, esd_energy}
    references TEXT[] DEFAULT '{}',  -- Literature references
    is_builtin BOOLEAN DEFAULT true,
    uploaded_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX reactants_name_idx ON reactants(name);
CREATE INDEX reactants_category_idx ON reactants(category);
CREATE INDEX reactants_name_trgm_idx ON reactants USING gin(name gin_trgm_ops);

-- Formulations (user-defined mixtures)
CREATE TABLE formulations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    formulation_type TEXT NOT NULL,  -- pressed | cast | pbx | anfo | emulsion | propellant
    -- Array of {reactant_id, mass_fraction}
    components JSONB NOT NULL,  -- [{"reactant_id": "...", "mass_fraction": 0.85}, ...]
    -- Computed properties (cached from last computation)
    density REAL,  -- g/cm³ (computed from component densities)
    heat_of_formation REAL,  -- kJ/kg
    oxygen_balance REAL,  -- %
    is_public BOOLEAN DEFAULT false,
    forked_from UUID REFERENCES formulations(id) ON DELETE SET NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX formulations_owner_idx ON formulations(owner_id);
CREATE INDEX formulations_org_idx ON formulations(org_id);
CREATE INDEX formulations_updated_idx ON formulations(updated_at DESC);

-- Computations (detonation performance calculations)
CREATE TABLE computations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    formulation_id UUID NOT NULL REFERENCES formulations(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    computation_type TEXT NOT NULL,  -- cj_state | isentrope | hugoniot | sensitivity_prediction
    eos_type TEXT NOT NULL,  -- ideal_gas | bkw | bkwc | bkwr | bkws | jcz3
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | running | completed | failed
    execution_mode TEXT NOT NULL DEFAULT 'wasm',  -- wasm | server
    -- Input parameters
    parameters JSONB NOT NULL DEFAULT '{}',  -- {temperature, initial_density, ...}
    -- Output results
    results JSONB,  -- {cj_velocity, cj_pressure, cj_temperature, product_species: [...]}
    results_url TEXT,  -- S3 URL for large result datasets (isentrope, parameter sweeps)
    error_message TEXT,
    convergence_iterations INTEGER,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    wall_time_ms INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX computations_formulation_idx ON computations(formulation_id);
CREATE INDEX computations_user_idx ON computations(user_id);
CREATE INDEX computations_status_idx ON computations(status);
CREATE INDEX computations_created_idx ON computations(created_at DESC);

-- Computation Jobs (for server-side execution only)
CREATE TABLE computation_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    computation_id UUID NOT NULL REFERENCES computations(id) ON DELETE CASCADE,
    worker_id TEXT,
    priority INTEGER DEFAULT 0,
    progress_pct REAL DEFAULT 0.0,
    convergence_data JSONB DEFAULT '[]',  -- [{iteration, residual, gibbs_energy}]
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX jobs_comp_idx ON computation_jobs(computation_id);
CREATE INDEX jobs_worker_idx ON computation_jobs(worker_id);

-- Experimental Validation Data (literature CJ velocities, pressures)
CREATE TABLE experimental_data (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    reactant_id UUID REFERENCES reactants(id) ON DELETE SET NULL,
    compound_name TEXT NOT NULL,
    density REAL NOT NULL,  -- g/cm³ (loading density)
    cj_velocity REAL,  -- km/s (measured Chapman-Jouguet detonation velocity)
    cj_pressure REAL,  -- GPa (measured or computed from Hugoniot)
    test_method TEXT,  -- cylinder_test | plate_dent | streak_camera | ...
    confinement TEXT,  -- unconfined | steel_tube | copper_tube | ...
    reference TEXT NOT NULL,  -- Journal citation
    doi TEXT,
    year INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX exp_data_reactant_idx ON experimental_data(reactant_id);
CREATE INDEX exp_data_compound_idx ON experimental_data(compound_name);

-- Usage Tracking (for billing enforcement)
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,  -- computation_server | batch_computation | api_call
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
pub struct Species {
    pub id: Uuid,
    pub name: String,
    pub formula: String,
    pub phase: String,
    pub molecular_weight: f64,
    pub heat_of_formation_298: f64,
    pub nasa_poly_low: Vec<f64>,
    pub nasa_poly_high: Vec<f64>,
    pub temp_range_low: Vec<f64>,
    pub temp_range_high: Vec<f64>,
    pub source: Option<String>,
    pub notes: Option<String>,
    pub is_builtin: bool,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Reactant {
    pub id: Uuid,
    pub name: String,
    pub formula: String,
    pub category: String,
    pub cas_number: Option<String>,
    pub density: f64,
    pub heat_of_formation: f64,
    pub molecular_weight: f64,
    pub oxygen_balance: Option<f64>,
    pub composition: serde_json::Value,
    pub sensitivity_data: serde_json::Value,
    pub references: Vec<String>,
    pub is_builtin: bool,
    pub uploaded_by: Option<Uuid>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Formulation {
    pub id: Uuid,
    pub owner_id: Uuid,
    pub org_id: Option<Uuid>,
    pub name: String,
    pub description: String,
    pub formulation_type: String,
    pub components: serde_json::Value,
    pub density: Option<f64>,
    pub heat_of_formation: Option<f64>,
    pub oxygen_balance: Option<f64>,
    pub is_public: bool,
    pub forked_from: Option<Uuid>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Computation {
    pub id: Uuid,
    pub formulation_id: Uuid,
    pub user_id: Uuid,
    pub computation_type: String,
    pub eos_type: String,
    pub status: String,
    pub execution_mode: String,
    pub parameters: serde_json::Value,
    pub results: Option<serde_json::Value>,
    pub results_url: Option<String>,
    pub error_message: Option<String>,
    pub convergence_iterations: Option<i32>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub wall_time_ms: Option<i32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct ComputationParams {
    pub computation_type: ComputationType,
    pub eos_type: EosType,
    pub initial_density: Option<f64>,  // g/cm³, if not specified use formulation density
    pub initial_temperature: f64,  // K, default 298.15
    pub chamber_pressure: Option<f64>,  // GPa for propellant combustion
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum ComputationType {
    CjState,          // Chapman-Jouguet detonation state
    Isentrope,        // Product isentrope expansion
    Hugoniot,         // Shock Hugoniot curve
    SensitivityPrediction,  // QSPR sensitivity estimation
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum EosType {
    IdealGas,
    Bkw,      // BKW with default parameters
    Bkwc,     // BKW C-parameterization
    Bkwr,     // BKW R-parameterization
    Bkws,     // BKW S-parameterization (Hobbs-Baer)
    Jcz3,     // JCZ3 (Jacobs-Cowperthwaite-Zwisler)
}
```

---

## Thermochemical Solver Architecture Deep-Dive

### Governing Equations and Numerical Methods

ExplodeSafe's core thermochemical solver computes chemical equilibrium by minimizing the Gibbs free energy of the system subject to element conservation constraints. For a system with `n` species and `m` elements:

```
Minimize: G = Σ(i=1 to n) n_i * (g_i + RT ln(x_i))

Subject to: Σ(i=1 to n) a_ij * n_i = b_j  for j=1 to m  (element conservation)
            n_i ≥ 0  for all i  (non-negativity)

where:
  n_i = number of moles of species i
  g_i = standard Gibbs free energy of species i at (T, P)
  x_i = mole fraction of species i
  a_ij = number of atoms of element j in species i
  b_j = total moles of element j in system
```

For **detonation products** at high pressure (20-40 GPa) and temperature (3000-4500 K), we use the **BKW equation of state**:

```
P = ρRT(1 + x·e^(βx)) / (M_avg(1 - κx)³)

where:
  ρ = density (g/cm³)
  T = temperature (K)
  R = gas constant
  M_avg = average molecular weight
  x = κρ Σ(n_i * X_i)  (covolume parameter)
  X_i = covolume of species i
  κ, β = BKW calibration parameters (vary by EOS variant)

BKW Parameters:
  BKWC: α=0.5, β=0.176, κ=10.50, Θ=6620
  BKWR: α=0.5, β=0.16, κ=10.91, Θ=6620
  BKWS: α=0.72, β=0.18, κ=14.10, Θ=6620  (Hobbs-Baer)
```

The **Chapman-Jouguet (CJ) state** is the detonation steady-state where the reaction front propagates at the minimum velocity consistent with the shock Hugoniot and product isentrope. We solve for CJ conditions using the **Rankine-Hugoniot conservation equations** plus the **tangency condition**:

```
Conservation of mass:      ρ₀ D = ρ₁ (D - u₁)
Conservation of momentum:  P₁ - P₀ = ρ₀ D u₁
Conservation of energy:    E₁ - E₀ = (P₁ + P₀)(v₀ - v₁)/2

CJ tangency condition:     (dP/dv)_s = -ρ₁² D²  (isentrope tangent to Rayleigh line)

where:
  subscript 0 = unreacted explosive
  subscript 1 = detonation products at CJ state
  D = detonation velocity (km/s)
  u₁ = particle velocity behind shock front
  v = specific volume (1/ρ)
  E = specific internal energy
```

We iterate to find T₁, P₁, ρ₁ such that:
1. Chemical equilibrium is satisfied (Gibbs minimization at T₁, P₁)
2. Hugoniot energy balance is satisfied
3. CJ tangency condition is satisfied (sonic flow in products frame)

### Numerical Solver: Constrained Gibbs Minimization

We use a **Lagrange multiplier method** to solve the constrained minimization problem. Define the Lagrangian:

```
L = G(n) - Σ(j=1 to m) λ_j * (Σ(i=1 to n) a_ij * n_i - b_j)

Setting ∂L/∂n_i = 0:
  μ_i + RT ln(x_i) + RT = Σ(j=1 to m) λ_j * a_ij

where μ_i = g_i + RT ln(P/P_ref) for gases
```

We solve this system using **Newton-Raphson iteration** with line search for robustness. The Jacobian includes second derivatives of Gibbs free energy and EOS derivatives (∂P/∂n_i, ∂T/∂n_i for constant U,V problems).

**Condensed phase handling**: Solid and liquid species (C(gr), Al₂O₃(s), H₂O(l)) can coexist with gas phase at detonation conditions. We use a **phase-splitting algorithm**:
1. Compute equilibrium assuming all species are present
2. If any species has n_i < 0, remove it from the basis and resolve
3. Check chemical potential: if μ_i < μ_i(gas) for condensed species, add it to basis
4. Iterate until all species satisfy equilibrium conditions and n_i ≥ 0

### Client/Server Split (WASM Threshold)

```
Formulation submitted → Component count and EOS checked
    │
    ├── ≤5 components + (ideal gas or BKW) → WASM solver (browser)
    │   ├── Instant CJ state computation (<1 sec)
    │   ├── Results displayed immediately
    │   └── No server cost
    │
    └── >5 components OR JCZ3/multi-phase → Server solver (Rust native)
        ├── Job queued via Redis
        ├── Worker picks up, runs full Gibbs minimization
        ├── Progress streamed via WebSocket
        └── Results stored in database + S3
```

The 5-component threshold was chosen because:
- WASM solver handles common binary/ternary explosive formulations (RDX/wax, HMX/binder, ANFO) in <1 second
- 5 components covers: primary explosive + binder + plasticizer + sensitizer + metal additive
- Above 5 components: complex PBX formulations, propellant mixtures with many ingredients → need server compute for multi-phase equilibrium and JCZ3 EOS

---

## Architecture Deep-Dives

### 1. Computation API Handler (Rust/Axum)

The primary endpoint receives a computation request, validates the formulation, decides between WASM and server execution, and for server execution enqueues a job with Redis.

```rust
// src/api/handlers/computation.rs

use axum::{
    extract::{Path, State, Json},
    http::StatusCode,
    response::IntoResponse,
};
use uuid::Uuid;
use crate::{
    db::models::{Computation, ComputationParams, ComputationType, EosType},
    solver::formulation::compute_formulation_properties,
    state::AppState,
    auth::Claims,
    error::ApiError,
};

#[derive(serde::Deserialize)]
pub struct CreateComputationRequest {
    pub computation_type: ComputationType,
    pub eos_type: EosType,
    pub parameters: serde_json::Value,
}

pub async fn create_computation(
    State(state): State<AppState>,
    claims: Claims,
    Path(formulation_id): Path<Uuid>,
    Json(req): Json<CreateComputationRequest>,
) -> Result<impl IntoResponse, ApiError> {
    // 1. Verify formulation ownership
    let formulation = sqlx::query_as!(
        crate::db::models::Formulation,
        "SELECT * FROM formulations WHERE id = $1 AND (owner_id = $2 OR org_id IN (
            SELECT org_id FROM org_members WHERE user_id = $2
        ))",
        formulation_id,
        claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Formulation not found"))?;

    // 2. Validate formulation has components
    let components: Vec<serde_json::Value> = serde_json::from_value(
        formulation.components.clone()
    )?;
    if components.is_empty() {
        return Err(ApiError::BadRequest("Formulation has no components"));
    }

    // 3. Check plan limits
    let user = sqlx::query_as!(
        crate::db::models::User,
        "SELECT * FROM users WHERE id = $1",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    if user.plan == "free" && components.len() > 3 {
        return Err(ApiError::PlanLimit(
            "Free plan supports formulations up to 3 components. Upgrade to Pro for unlimited."
        ));
    }

    if user.plan == "free" && !matches!(req.eos_type, EosType::IdealGas) {
        return Err(ApiError::PlanLimit(
            "Free plan supports ideal gas EOS only. Upgrade to Pro for BKW/JCZ3."
        ));
    }

    // 4. Determine execution mode
    let execution_mode = if components.len() <= 5 &&
        matches!(req.eos_type, EosType::IdealGas | EosType::Bkw | EosType::Bkwc | EosType::Bkwr | EosType::Bkws)
    {
        "wasm"
    } else {
        "server"
    };

    // 5. Create computation record
    let comp = sqlx::query_as!(
        Computation,
        r#"INSERT INTO computations
            (formulation_id, user_id, computation_type, eos_type, status, execution_mode, parameters)
        VALUES ($1, $2, $3, $4, $5, $6, $7)
        RETURNING *"#,
        formulation_id,
        claims.user_id,
        serde_json::to_string(&req.computation_type)?,
        serde_json::to_string(&req.eos_type)?,
        if execution_mode == "wasm" { "completed" } else { "pending" },
        execution_mode,
        req.parameters,
    )
    .fetch_one(&state.db)
    .await?;

    // 6. For server execution, enqueue job
    if execution_mode == "server" {
        let job = sqlx::query_as!(
            crate::db::models::ComputationJob,
            r#"INSERT INTO computation_jobs (computation_id, priority)
            VALUES ($1, $2) RETURNING *"#,
            comp.id,
            if user.plan == "advanced" { 10 } else { 0 },
        )
        .fetch_one(&state.db)
        .await?;

        // Publish to Redis job queue
        state.redis
            .publish("computation:jobs", serde_json::to_string(&job.id)?)
            .await?;
    }

    Ok((StatusCode::CREATED, Json(comp)))
}

pub async fn get_computation(
    State(state): State<AppState>,
    claims: Claims,
    Path((formulation_id, comp_id)): Path<(Uuid, Uuid)>,
) -> Result<Json<Computation>, ApiError> {
    let comp = sqlx::query_as!(
        Computation,
        "SELECT c.* FROM computations c
         JOIN formulations f ON c.formulation_id = f.id
         WHERE c.id = $1 AND c.formulation_id = $2
         AND (f.owner_id = $3 OR f.org_id IN (
             SELECT org_id FROM org_members WHERE user_id = $3
         ))",
        comp_id,
        formulation_id,
        claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Computation not found"))?;

    Ok(Json(comp))
}
```

### 2. Gibbs Minimization Solver Core (Rust — shared between WASM and native)

The core thermochemical equilibrium solver that minimizes Gibbs free energy subject to element conservation constraints.

```rust
// solver-core/src/gibbs.rs

use nalgebra::{DMatrix, DVector};
use crate::species::SpeciesDatabase;
use crate::eos::{EquationOfState, BkwEos};

pub struct GibbsSystem {
    pub n_species: usize,
    pub n_elements: usize,
    pub species_db: SpeciesDatabase,
    pub element_matrix: DMatrix<f64>,  // a_ij: atoms of element j in species i
    pub element_totals: DVector<f64>,  // b_j: total moles of each element
    pub moles: DVector<f64>,           // n_i: current mole numbers
    pub temperature: f64,              // K
    pub pressure: f64,                 // Pa (or computed from EOS)
    pub eos: Box<dyn EquationOfState>,
}

impl GibbsSystem {
    pub fn new(
        species_db: SpeciesDatabase,
        element_composition: Vec<(String, f64)>,  // [(element, moles), ...]
        eos: Box<dyn EquationOfState>,
    ) -> Self {
        let n_species = species_db.species.len();
        let n_elements = element_composition.len();

        // Build element matrix a_ij
        let mut element_matrix = DMatrix::zeros(n_species, n_elements);
        for (i, species) in species_db.species.iter().enumerate() {
            for (j, (element, _)) in element_composition.iter().enumerate() {
                element_matrix[(i, j)] = species.composition.get(element).copied().unwrap_or(0.0);
            }
        }

        let element_totals = DVector::from_iterator(
            n_elements,
            element_composition.iter().map(|(_, moles)| *moles)
        );

        // Initial guess: distribute elements uniformly among species
        let moles = DVector::from_element(n_species, 1e-6);

        Self {
            n_species,
            n_elements,
            species_db,
            element_matrix,
            element_totals,
            moles,
            temperature: 298.15,
            pressure: 101325.0,
            eos,
        }
    }

    /// Compute Gibbs free energy of current state
    pub fn compute_gibbs(&self) -> f64 {
        let r = 8.314;  // J/(mol·K)
        let total_moles: f64 = self.moles.sum();
        let mut g = 0.0;

        for (i, species) in self.species_db.species.iter().enumerate() {
            let n_i = self.moles[i];
            if n_i < 1e-20 { continue; }

            let g_i = species.gibbs_free_energy(self.temperature);
            let x_i = n_i / total_moles;

            // Gas phase contribution
            if species.phase == "gas" {
                g += n_i * (g_i + r * self.temperature * x_i.ln());
            } else {
                // Condensed phase: no mixing entropy
                g += n_i * g_i;
            }
        }

        g
    }

    /// Solve for equilibrium at specified T and P using Newton-Raphson
    pub fn solve_equilibrium_tp(&mut self, temp: f64, pressure: f64, max_iter: usize) -> Result<(), SolverError> {
        self.temperature = temp;
        self.pressure = pressure;

        for iter in 0..max_iter {
            // Compute chemical potentials
            let mu = self.compute_chemical_potentials();

            // Build Lagrangian gradient and Hessian
            let (grad, hessian) = self.build_lagrangian_system();

            // Newton step with line search
            let delta = hessian.lu().solve(&(-grad))
                .ok_or(SolverError::Singular)?;

            // Line search to ensure mole numbers stay positive
            let mut alpha = 1.0;
            for _ in 0..10 {
                let mut moles_new = self.moles.clone();
                for i in 0..self.n_species {
                    moles_new[i] = (self.moles[i] + alpha * delta[i]).max(1e-20);
                }

                // Check if element balance improved
                let residual = self.compute_element_residual(&moles_new);
                if residual < self.compute_element_residual(&self.moles) {
                    self.moles = moles_new;
                    break;
                }
                alpha *= 0.5;
            }

            // Check convergence
            if delta.norm() < 1e-8 {
                return Ok(());
            }
        }

        Err(SolverError::Convergence("Failed to converge in Gibbs minimization".into()))
    }

    /// Solve for equilibrium at constant internal energy and volume (CJ detonation)
    pub fn solve_equilibrium_uv(
        &mut self,
        specific_volume: f64,  // cm³/g
        specific_energy: f64,  // J/g
        max_iter: usize
    ) -> Result<(), SolverError> {
        // Iterate on T,P until energy balance is satisfied
        let mut temp = 3000.0;  // Initial guess

        for _iter in 0..max_iter {
            // Solve equilibrium at current T
            self.solve_equilibrium_tp(temp, 1e9, 50)?;

            // Compute P from EOS
            let total_mass: f64 = self.moles.iter()
                .zip(self.species_db.species.iter())
                .map(|(n, sp)| n * sp.molecular_weight)
                .sum();
            let density = total_mass / specific_volume;  // g/cm³

            self.pressure = self.eos.pressure(self.temperature, density, &self.moles, &self.species_db)?;

            // Compute internal energy
            let u = self.compute_internal_energy();

            // Check energy balance
            let error = (u - specific_energy).abs() / specific_energy.abs().max(1.0);
            if error < 1e-6 {
                return Ok(());
            }

            // Adjust temperature for next iteration
            let du_dt = self.compute_heat_capacity();
            temp += (specific_energy - u) / du_dt;
            temp = temp.clamp(500.0, 8000.0);
        }

        Err(SolverError::Convergence("Failed to converge in U,V equilibrium".into()))
    }

    fn compute_chemical_potentials(&self) -> DVector<f64> {
        let r = 8.314;
        let total_moles = self.moles.sum();
        let mut mu = DVector::zeros(self.n_species);

        for (i, species) in self.species_db.species.iter().enumerate() {
            let g_i = species.gibbs_free_energy(self.temperature);
            let x_i = self.moles[i] / total_moles;

            if species.phase == "gas" {
                mu[i] = g_i + r * self.temperature * (1.0 + x_i.ln());
            } else {
                mu[i] = g_i;
            }
        }

        mu
    }

    fn compute_element_residual(&self, moles: &DVector<f64>) -> f64 {
        let computed_elements = &self.element_matrix.transpose() * moles;
        (&computed_elements - &self.element_totals).norm()
    }

    fn compute_internal_energy(&self) -> f64 {
        let mut u = 0.0;
        for (i, species) in self.species_db.species.iter().enumerate() {
            u += self.moles[i] * species.enthalpy(self.temperature);
        }
        // Subtract PV work
        let total_mass: f64 = self.moles.iter()
            .zip(self.species_db.species.iter())
            .map(|(n, sp)| n * sp.molecular_weight)
            .sum();
        u - self.pressure * (total_mass / self.compute_density())
    }

    fn compute_density(&self) -> f64 {
        // Density from EOS
        self.eos.density(self.temperature, self.pressure, &self.moles, &self.species_db)
            .unwrap_or(1.0)
    }

    fn compute_heat_capacity(&self) -> f64 {
        let mut cp = 0.0;
        for (i, species) in self.species_db.species.iter().enumerate() {
            cp += self.moles[i] * species.heat_capacity(self.temperature);
        }
        cp
    }

    fn build_lagrangian_system(&self) -> (DVector<f64>, DMatrix<f64>) {
        // Simplified: actual implementation includes full Hessian with constraints
        let grad = DVector::zeros(self.n_species + self.n_elements);
        let hessian = DMatrix::identity(self.n_species + self.n_elements, self.n_species + self.n_elements);
        (grad, hessian)
    }
}

#[derive(Debug)]
pub enum SolverError {
    Convergence(String),
    Singular,
    InvalidInput(String),
}
```

### 3. CJ Detonation State Solver (Rust)

Computes the Chapman-Jouguet detonation velocity, pressure, and temperature by solving the Rankine-Hugoniot equations with the CJ tangency condition.

```rust
// solver-core/src/cj_solver.rs

use crate::gibbs::GibbsSystem;
use crate::eos::BkwEos;
use crate::species::SpeciesDatabase;

pub struct CjState {
    pub velocity: f64,        // km/s
    pub pressure: f64,        // GPa
    pub temperature: f64,     // K
    pub density: f64,         // g/cm³
    pub particle_velocity: f64,  // km/s
    pub product_composition: Vec<(String, f64)>,  // [(species, mole_fraction), ...]
    pub gamma: f64,           // Effective gamma (Cp/Cv) of products
}

pub fn compute_cj_state(
    formulation_density: f64,  // g/cm³ (unreacted explosive density)
    formulation_energy: f64,   // kJ/kg (heat of formation)
    element_composition: Vec<(String, f64)>,  // Total moles of each element
    species_db: SpeciesDatabase,
    eos: BkwEos,
) -> Result<CjState, String> {
    // Initial guesses for CJ iteration
    let mut cj_pressure = 30.0;  // GPa (typical for HE)
    let mut cj_density = formulation_density * 2.0;  // Products are ~2x denser than reactants

    for iter in 0..50 {
        // 1. Solve Hugoniot for products at guessed P and ρ
        let specific_volume = 1.0 / cj_density;  // cm³/g

        // Hugoniot energy: E1 - E0 = (P1 + P0)(v0 - v1)/2
        // E0 = heat of formation of reactants
        // v0 = 1/ρ0, v1 = 1/ρ1
        let v0 = 1.0 / formulation_density;
        let v1 = specific_volume;
        let p0 = 1e-4;  // GPa (ambient pressure)
        let p1 = cj_pressure;

        let e0 = formulation_energy * 1000.0;  // J/kg
        let hugoniot_energy = e0 + (p1 + p0) * 1e9 * (v0 - v1) / 2.0;  // J/kg

        // 2. Solve for chemical equilibrium at constant U,V
        let mut gibbs = GibbsSystem::new(
            species_db.clone(),
            element_composition.clone(),
            Box::new(eos.clone()),
        );

        gibbs.solve_equilibrium_uv(specific_volume, hugoniot_energy, 100)
            .map_err(|e| format!("Gibbs solver failed: {:?}", e))?;

        // 3. Compute sound speed in products
        let gamma = gibbs.compute_heat_capacity() / gibbs.compute_cv();
        let sound_speed = (gamma * p1 * 1e9 / cj_density)
            .sqrt() / 1000.0;  // km/s

        // 4. Check CJ condition: D - u1 = c1 (sonic flow in products frame)
        // From Rankine-Hugoniot: D² = (P1 - P0) / (ρ0(1 - ρ0/ρ1))
        let detonation_velocity = ((p1 - p0) * 1e9 / (formulation_density * 1000.0 * (1.0 - formulation_density / cj_density)))
            .sqrt() / 1000.0;  // km/s

        let particle_velocity = detonation_velocity * (1.0 - formulation_density / cj_density);
        let relative_velocity = detonation_velocity - particle_velocity;

        let cj_error = (relative_velocity - sound_speed).abs() / sound_speed;

        if cj_error < 1e-4 {
            // Converged!
            let product_composition: Vec<(String, f64)> = gibbs.species_db.species.iter()
                .zip(gibbs.moles.iter())
                .map(|(sp, &n)| (sp.name.clone(), n / gibbs.moles.sum()))
                .filter(|(_, frac)| *frac > 1e-6)
                .collect();

            return Ok(CjState {
                velocity: detonation_velocity,
                pressure: cj_pressure,
                temperature: gibbs.temperature,
                density: cj_density,
                particle_velocity,
                product_composition,
                gamma,
            });
        }

        // 5. Adjust pressure and density for next iteration
        // Use secant method on CJ condition residual
        if relative_velocity > sound_speed {
            cj_pressure *= 1.05;
            cj_density *= 1.02;
        } else {
            cj_pressure *= 0.95;
            cj_density *= 0.98;
        }

        cj_pressure = cj_pressure.clamp(5.0, 80.0);
        cj_density = cj_density.clamp(formulation_density * 1.2, formulation_density * 3.0);
    }

    Err("CJ iteration failed to converge".into())
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–5)

**Day 1: Project initialization and Rust backend scaffold**
```bash
cargo init explodesafe-api
cd explodesafe-api
cargo add axum tokio serde serde_json sqlx uuid chrono tower-http jsonwebtoken bcrypt
cargo add --features postgres,runtime-tokio-rustls,uuid,chrono,json sqlx
cargo add aws-sdk-s3 redis nalgebra
```
- `src/main.rs` — Axum app with CORS, tracing, graceful shutdown
- `src/config.rs` — Environment-based configuration (DATABASE_URL, JWT_SECRET, S3_BUCKET, REDIS_URL)
- `src/state.rs` — AppState struct (PgPool, Redis, S3Client)
- `src/error.rs` — ApiError enum with IntoResponse
- `Dockerfile` — Multi-stage build (builder + runtime)
- `docker-compose.yml` — PostgreSQL, Redis, MinIO (S3-compatible)

**Day 2: Database schema and migrations**
- `migrations/001_initial.sql` — All 10 tables: users, organizations, org_members, species, reactants, formulations, computations, computation_jobs, experimental_data, usage_records
- `src/db/mod.rs` — Database pool initialization
- `src/db/models.rs` — All SQLx structs with FromRow derives
- Run `sqlx migrate run` to apply schema
- Seed script for initial species data (200 common species from JANAF database: H2O, CO2, CO, N2, O2, H2, CH4, NH3, HCN, solid C, Al2O3, etc.)

**Day 3: Species database loader**
- `scripts/load_species.py` — Python script to parse NASA polynomial coefficients from Burcat thermochemical database
- Convert Burcat `.dat` format to PostgreSQL inserts
- Insert 5,000+ species with NASA polynomials (7 coefficients × 2 temperature ranges)
- Validation: compute Cp, H, S at 298.15 K and compare to reference values

**Day 4: Authentication system**
- `src/auth/mod.rs` — JWT token generation and validation middleware
- `src/auth/oauth.rs` — Google and GitHub OAuth 2.0 flow handlers
- `src/api/handlers/auth.rs` — Register, login, OAuth callback, refresh token, me endpoints
- Password hashing with bcrypt (cost factor 12)
- JWT with 24h access token + 30d refresh token
- Auth middleware that extracts Claims from Authorization header

**Day 5: User and formulation CRUD**
- `src/api/handlers/users.rs` — Get profile, update profile, delete account
- `src/api/handlers/formulations.rs` — Create, list, get, update, delete, fork formulation
- `src/api/handlers/orgs.rs` — Create org, invite member, list members, remove member
- `src/api/router.rs` — All route definitions with auth middleware
- Integration tests: auth flow, formulation CRUD, authorization checks

### Phase 2 — Thermochemical Solver Core (Days 6–13)

**Day 6: Species thermodynamic property evaluation**
- `solver-core/` — New Rust workspace member (shared between native and WASM)
- `solver-core/src/species.rs` — Species struct with NASA polynomial evaluation methods
- Methods: `heat_capacity(T)`, `enthalpy(T)`, `entropy(T)`, `gibbs_free_energy(T)`
- NASA polynomial evaluation: Cp/R = a1 + a2*T + a3*T² + a4*T³ + a5*T⁴
- Unit tests: verify H2O properties at 298 K, 1000 K, 3000 K match JANAF tables

**Day 7: Element matrix and composition handling**
- `solver-core/src/composition.rs` — Parse empirical formula (e.g., "C3H6N6O6") into element dictionary
- Build element matrix a_ij (atoms of element j in species i)
- Compute element totals b_j from reactant composition
- Oxygen balance calculation: OB = (O - 2C - H/2) * 1600 / MW
- Tests: RDX (C3H6N6O6) OB = -21.6%, TNT OB = -74.0%

**Day 8: Ideal gas Gibbs minimization solver**
- `solver-core/src/gibbs.rs` — GibbsSystem struct with Newton-Raphson minimization
- Ideal gas phase only (no condensed products yet)
- Lagrange multiplier method for element conservation constraints
- Line search to keep mole numbers positive
- Tests: H2/O2 combustion, verify H2O dominant product at stoichiometric ratio

**Day 9: BKW equation of state implementation**
- `solver-core/src/eos/bkw.rs` — BKW EOS with BKWC, BKWR, BKWS parameterizations
- Covolume database for common species (H2O, CO2, CO, N2, etc.)
- `pressure(T, ρ, moles)` method implementing BKW formula
- `sound_speed(T, P, ρ)` method for CJ tangency condition
- Tests: compute pressure for known detonation products, compare to reference values

**Day 10: Hugoniot and CJ state solver — Prototype**
- `solver-core/src/cj_solver.rs` — CJ state iteration using Rankine-Hugoniot equations
- Initial guess: P_CJ ≈ ρ₀ D² / 4 (rough approximation)
- Iterate: solve equilibrium at (P, ρ) → compute sound speed → check CJ condition → adjust P, ρ
- Convergence when |D - u - c| < 0.1% of D
- Tests: TNT at 1.63 g/cm³, verify D ≈ 6.9 km/s, P ≈ 21 GPa

**Day 11: Condensed product phase handling**
- `solver-core/src/gibbs_condensed.rs` — Phase-splitting algorithm for solid/liquid species
- Check chemical potential: if μ_i(condensed) < μ_i(gas), include in equilibrium
- Iterate: add/remove species from basis until all satisfy equilibrium conditions
- Aluminized explosive test: Al + O2 → Al2O3(s) formation
- Tests: verify solid carbon formation in fuel-rich compositions

**Day 12: JCZ3 equation of state**
- `solver-core/src/eos/jcz3.rs` — JCZ3 EOS implementation (more complex than BKW)
- JCZ3 parameters for major species
- Pressure and energy derivatives for EOS
- Tests: compare BKW vs JCZ3 predictions for HMX, verify difference <5%

**Day 13: JWL isentrope fitting**
- `solver-core/src/jwl.rs` — JWL parameter fitting from CJ state and EOS isentrope
- JWL form: P = A·e^(-R1·v) + B·e^(-R2·v) + ωE/v
- Fit A, B, R1, R2, ω to match p(v) isentrope over expansion range
- Levenberg-Marquardt optimization
- Tests: fit JWL for RDX, verify parameters within 10% of published values

### Phase 3 — WASM Build + Frontend Visualization (Days 14–20)

**Day 14: WASM solver compilation**
- `solver-wasm/` — New workspace member for WASM target
- `solver-wasm/Cargo.toml` — wasm-bindgen, serde-wasm-bindgen dependencies
- `solver-wasm/src/lib.rs` — WASM entry points: `compute_cj_state()`, `compute_isentrope()`, `compute_oxygen_balance()`
- `wasm-pack build --target web --release` pipeline
- `wasm-opt -Oz` for size optimization (target <1.5MB gzipped)
- JavaScript wrapper: `ThermoSolver` class that loads WASM and provides async API
- Test WASM solver in Node.js: load WASM, compute TNT CJ state, verify results match native
- Benchmark: measure WASM performance vs. native for simple formulations
- Target: 3-component formulation with BKW EOS < 500ms in WASM

**Day 15: Frontend scaffold and reactant library UI**
```bash
npm create vite@latest frontend -- --template react-ts
cd frontend
npm i zustand @tanstack/react-query axios plotly.js d3
```
- `src/App.tsx` — Router, auth context, layout
- `src/stores/authStore.ts` — Zustand store for auth state
- `src/stores/formulationStore.ts` — Formulation builder state
- `src/pages/ReactantLibrary.tsx` — Browse reactants by category (explosive, oxidizer, fuel, binder, metal)
- `src/components/ReactantCard.tsx` — Display reactant properties (density, heat of formation, oxygen balance)
- Search and filter: by name, category, oxygen balance range

**Day 16: Formulation builder UI**
- `src/pages/FormulationBuilder.tsx` — Multi-component formulation editor
- `src/components/FormulationBuilder/ComponentList.tsx` — List of selected components with mass fraction sliders
- `src/components/FormulationBuilder/ReactantSearch.tsx` — Searchable reactant picker
- Mass fraction normalization: auto-adjust to sum to 100%
- Live property computation: density (volume additivity), oxygen balance, heat of formation
- Formulation type selector: pressed, cast, PBX, ANFO, emulsion, propellant

**Day 17: Computation results visualization — CJ state plots**
- `src/components/ComputationResults/CjStateCard.tsx` — Display CJ velocity, pressure, temperature, density
- `src/components/ComputationResults/ProductCompositionChart.tsx` — Pie chart of product species mole fractions (Plotly.js)
- `src/components/ComputationResults/HugoniotPlot.tsx` — P-v Hugoniot curve and Rayleigh line intersection
- `src/components/ComputationResults/PerformanceMap.tsx` — 2D contour plot: detonation velocity vs. density and composition
- Color scales: viridis for temperature, plasma for pressure

**Day 18: Ternary diagram for 3-component formulations**
- `src/components/FormulationBuilder/TernaryDiagram.tsx` — D3.js ternary plot
- Plot CJ velocity contours on ternary composition space (e.g., RDX/Al/wax)
- Interactive: click point on ternary → update formulation composition
- Highlight region of maximum performance
- Export ternary plot as SVG/PNG

**Day 19: Performance parameter sweeps**
- `src/components/ComputationResults/ParameterSweep.tsx` — Sweep density or component ratio, plot performance vs. parameter
- Example sweeps:
  - CJ velocity vs. loading density (0.8ρ_TMD to 1.0ρ_TMD)
  - CJ pressure vs. Al content (0-30% Al in RDX/Al/wax)
  - Specific impulse vs. oxidizer/fuel ratio for propellants
- Plotly.js line plots with multiple traces

**Day 20: Experimental validation comparison**
- `src/pages/Validation.tsx` — Display computed vs. experimental CJ velocities for known compounds
- `src/components/Validation/ScatterPlot.tsx` — Scatter plot: computed vs. experimental, with ±5% error bands
- Load experimental data from database (200+ compounds)
- Color-code by compound type (RDX-based, HMX-based, TATB-based, ANFO, etc.)
- R² correlation coefficient display

### Phase 4 — API + Job Orchestration (Days 21–26)

**Day 21: Computation API endpoints**
- `src/api/handlers/computation.rs` — Create computation, get computation, list computations, cancel computation
- Formulation property computation: density, oxygen balance, heat of formation from components
- Component count extraction and WASM/server routing logic
- Plan-based limits enforcement (free: 3 components, ideal gas only)

**Day 22: Server-side computation worker**
- `src/workers/computation_worker.rs` — Redis job consumer, runs native Gibbs solver and CJ solver
- `src/workers/mod.rs` — Worker pool management (configurable concurrency)
- Progress streaming: worker publishes progress → Redis pub/sub → WebSocket to client
- Error handling: convergence failures, invalid formulations, out-of-range densities
- S3 result upload for large datasets (isentrope p-v curves, parameter sweeps)

**Day 23: WebSocket for live computation progress**
- `src/api/ws/mod.rs` — Axum WebSocket upgrade handler
- `src/api/ws/computation_progress.rs` — Subscribe to computation progress channel
- Client receives: `{ progress_pct, current_iteration, gibbs_residual, cj_residual }`
- Connection management: heartbeat ping/pong, automatic cleanup on disconnect
- Frontend: `src/hooks/useComputationProgress.ts` — React hook for WebSocket subscription

**Day 24: Reactant library API and search**
- `src/api/handlers/reactants.rs` — Search reactants (parametric + full-text), get reactant, upload custom reactant
- Parametric search: filter by category, density range, oxygen balance range, sensitivity
- Full-text search on name and CAS number (PostgreSQL trigram indexes)
- Pagination with cursor-based scrolling for large result sets
- Custom reactant upload: parse empirical formula, compute molecular weight, require heat of formation input

**Day 25: Species database API**
- `src/api/handlers/species.rs` — Search species, get species thermodynamic data
- Export species data as JSON (NASA polynomials) for integration with external codes
- Admin endpoint: upload custom species with NASA polynomial coefficients
- Validation: check polynomial continuity at temperature range boundaries

**Day 26: Formulation management features**
- `src/api/handlers/formulations.rs` — Fork formulation (deep copy), share via public link, formulation templates
- `src/api/handlers/export.rs` — Export computation results as PDF technical report (composition, CJ state, product speciation, JWL parameters)
- PDF generation: use headless Chrome + HTML template rendering
- Export JWL parameters formatted for LS-DYNA input deck

### Phase 5 — QSPR Sensitivity Prediction (Days 27–30)

**Day 27: Molecular descriptor extraction (Python FastAPI)**
- `qspr-service/` — Python FastAPI microservice for QSPR sensitivity prediction
- `requirements.txt` — RDKit, scikit-learn, XGBoost, FastAPI
- `qspr-service/extractors/molecular_descriptors.py` — Extract 200+ descriptors from SMILES string using RDKit
- Descriptors: molecular weight, oxygen balance, density (predicted from Van der Waals volume), nitrogen content, aromatic ring count, nitro group count, heterocycle count, polarizability
- Input: SMILES string, Output: feature vector

**Day 28: QSPR model training for impact sensitivity**
- `qspr-service/models/impact_sensitivity.py` — Train XGBoost regressor for h50 (impact sensitivity) prediction
- Training data: 300+ compounds with experimental h50 values from literature (LASL, Köhler-Meyer)
- Features: molecular descriptors + oxygen balance + density
- Target: log(h50) in cm (50% drop height)
- Validation: 80/20 train/test split, R² > 0.75 target
- Save trained model as `impact_model.pkl`

**Day 29: QSPR API endpoints**
- `qspr-service/main.py` — FastAPI endpoints: `/predict/impact`, `/predict/friction`, `/predict/esd`
- Input: reactant SMILES string or empirical formula
- Output: predicted h50, friction sensitivity (N), ESD sensitivity (J)
- Uncertainty quantification: prediction interval from XGBoost variance
- Integration with Rust API: HTTP client calls Python service for QSPR predictions

**Day 30: Sensitivity prediction UI**
- `src/pages/SensitivityAnalysis.tsx` — Sensitivity prediction page
- Input: select reactant or enter custom SMILES
- Display predicted impact, friction, ESD with uncertainty ranges
- Compare to experimental values (if available)
- Safety classification: show UN hazard class based on predicted sensitivity

### Phase 6 — Validation + Advanced Features (Days 31–35)

**Day 31: Solver validation against experimental database**
- Load 200+ experimental CJ velocities from `experimental_data` table
- Compute CJ state for each compound at experimental density using BKW, BKWC, BKWR, BKWS, JCZ3
- Compare computed vs. experimental velocities
- Generate validation report: mean absolute error, R², error distribution by EOS type
- Target: MAE < 0.3 km/s, R² > 0.90 for BKWS

**Day 32: Batch formulation screening**
- `src/api/handlers/batch.rs` — Submit batch computation job: sweep component ratios or densities
- Example: screen 100 RDX/Al/wax compositions on 10×10 grid
- Enqueue as single batch job, parallelize across workers
- Results stored in S3 as CSV: composition, density, CJ velocity, CJ pressure, oxygen balance
- Frontend: upload CSV of formulations or define parameter sweep

**Day 33: Propellant performance (specific impulse)**
- `solver-core/src/propellant.rs` — Specific impulse (Isp) calculation for rocket propellants
- Equilibrium expansion: compute frozen or shifting equilibrium along nozzle expansion
- Chamber pressure → throat (sonic) → exit (specified area ratio or exit pressure)
- Isp = ∫(v·dm) / (m·g₀) where v is nozzle exit velocity
- Tests: AP/HTPB propellant, verify Isp ≈ 260 s (vacuum)

**Day 34: Non-ideal detonation (diameter effects)**
- `solver-core/src/diameter_effect.rs` — Empirical diameter effect model
- D(d) = D_∞ · (1 - a/d^n) where d is charge diameter, D_∞ is infinite-diameter velocity
- Fit parameters a, n from experimental data (if available)
- Failure diameter estimation: D(d_f) = 0
- UI: plot detonation velocity vs. diameter curve

**Day 35: Hydrocode integration — LS-DYNA input file export**
- `src/api/handlers/hydrocode.rs` — Generate LS-DYNA *EOS_JWL card from JWL parameters
- Template: `*EOS_JWL \n $# ... \n A B R1 R2 OMEGA E0 V0`
- Auto-populate from computed JWL fit
- Include comments with formulation composition and CJ state
- Similarly for AUTODYN (JWL material model), CTH (EOS tables)

### Phase 7 — Billing + Plan Enforcement (Days 36–38)

**Day 36: Stripe integration**
- `src/api/handlers/billing.rs` — Create checkout session, create customer portal session, get subscription status
- `src/api/handlers/webhooks/stripe.rs` — Handle subscription events: `checkout.session.completed`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.payment_failed`
- Plan mapping:
  - Free: 3 components, ideal gas EOS only, 10 computations/month
  - Pro ($149/mo): Unlimited components, BKW/JCZ3 EOS, JWL fitting, QSPR sensitivity, unlimited computations
  - Advanced ($349/user/mo): Team collaboration, batch screening, API access, custom species upload, priority compute

**Day 37: Usage tracking and plan limits**
- `src/middleware/plan_limits.rs` — Middleware checking plan limits before computation execution
- `src/services/usage.rs` — Track server computations per billing period
- Usage record insertion after each server-side computation completes
- Usage dashboard: `src/api/handlers/usage.rs` — Current period usage, historical usage
- Approaching-limit warnings at 80% and 100% of monthly quota

**Day 38: Billing UI and feature gating**
- `frontend/src/pages/Billing.tsx` — Plan comparison table, current plan display, usage meter
- `frontend/src/components/billing/PlanCard.tsx` — Plan feature comparison cards
- `frontend/src/components/billing/UsageMeter.tsx` — Computation usage bar with threshold indicators
- Upgrade/downgrade flow via Stripe Customer Portal
- Feature gating:
  - Gate BKW/JCZ3 EOS behind Pro plan
  - Gate QSPR sensitivity prediction behind Pro plan
  - Gate batch screening behind Advanced plan
  - Gate API access behind Advanced plan

### Phase 8 — Deployment + Polish + Launch (Days 39–42)

**Day 39: Docker and Kubernetes configuration**
- `Dockerfile` — Multi-stage Rust build (cargo-chef for layer caching)
- `docker-compose.yml` — Full local dev stack (API, PostgreSQL, Redis, MinIO, Python QSPR service)
- `k8s/` — Kubernetes manifests:
  - `api-deployment.yaml` — API server (3 replicas, HPA)
  - `worker-deployment.yaml` — Computation workers (auto-scaling based on queue depth)
  - `qspr-deployment.yaml` — Python QSPR service (2 replicas)
  - `postgres-statefulset.yaml` — PostgreSQL with PVC
  - `redis-deployment.yaml` — Redis for job queue and pub/sub
  - `ingress.yaml` — NGINX ingress with TLS
- Health check endpoints: `/health/live`, `/health/ready`

**Day 40: CDN and WASM delivery**
- CloudFront distribution for:
  - Frontend static assets (HTML, JS, CSS) — cached at edge
  - WASM solver bundle (~1.5MB) — cached at edge with long TTL and versioned URLs
  - Species database JSON exports — cached with 1-hour TTL
- WASM preloading: start downloading WASM bundle on page load before user needs it
- Service worker for offline WASM caching (progressive enhancement)

**Day 41: Monitoring, logging, and polish**
- Prometheus metrics: computation duration histogram, solver convergence rate, API latency percentiles
- Grafana dashboards: system health, computation throughput, user activity, plan distribution
- Sentry integration: frontend and backend error tracking with source maps
- Structured logging (tracing + JSON) for all API requests and solver events
- UI polish: loading states, error messages, empty states, responsive layout
- Accessibility: keyboard navigation, ARIA labels, screen reader support

**Day 42: Launch preparation and final testing**
- End-to-end smoke test in production environment
- Database backup and restore verification
- Rate limiting: 100 req/min per user, 10 computations/min per user
- Security audit: JWT validation, SQL injection (SQLx prevents this), CORS configuration, CSP headers
- Landing page with product overview, use cases (defense R&D, mining explosives, propellant design), pricing
- Documentation: getting started guide (create formulation → run computation → view results), validation methodology, EOS selection guide
- Deploy to production, enable monitoring alerts, announce launch

---

## Critical Files

```
explodesafe/
├── solver-core/                           # Shared thermochemical solver library (native + WASM)
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── species.rs                     # Species thermodynamic property evaluation (NASA polynomials)
│   │   ├── composition.rs                 # Empirical formula parsing, element matrix building
│   │   ├── gibbs.rs                       # Gibbs free energy minimization solver (Newton-Raphson)
│   │   ├── gibbs_condensed.rs             # Phase-splitting for solid/liquid products
│   │   ├── cj_solver.rs                   # Chapman-Jouguet detonation state solver
│   │   ├── jwl.rs                         # JWL isentrope fitting (Levenberg-Marquardt)
│   │   ├── propellant.rs                  # Propellant Isp calculation
│   │   ├── diameter_effect.rs             # Non-ideal detonation diameter effects
│   │   ├── eos/
│   │   │   ├── mod.rs                     # EOS trait definition
│   │   │   ├── ideal_gas.rs               # Ideal gas EOS
│   │   │   ├── bkw.rs                     # BKW EOS (BKWC, BKWR, BKWS variants)
│   │   │   └── jcz3.rs                    # JCZ3 EOS
│   │   └── validation.rs                  # Solver validation against experimental data
│   └── tests/
│       ├── gibbs_tests.rs                 # Gibbs minimization unit tests
│       ├── cj_tests.rs                    # CJ solver validation (TNT, RDX, HMX)
│       └── eos_tests.rs                   # EOS unit tests
│
├── solver-wasm/                           # WASM compilation target
│   ├── Cargo.toml
│   └── src/
│       └── lib.rs                         # WASM entry points (wasm-bindgen)
│
├── explodesafe-api/                       # Rust API server
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
│   │   │   │   ├── formulations.rs        # Formulation CRUD, fork, share
│   │   │   │   ├── computation.rs         # Create/get/cancel computation
│   │   │   │   ├── reactants.rs           # Reactant search/get, custom upload
│   │   │   │   ├── species.rs             # Species database search/get
│   │   │   │   ├── batch.rs               # Batch formulation screening
│   │   │   │   ├── hydrocode.rs           # LS-DYNA/AUTODYN input export
│   │   │   │   ├── billing.rs             # Stripe checkout/portal
│   │   │   │   ├── usage.rs               # Usage tracking
│   │   │   │   ├── export.rs              # PDF report export
│   │   │   │   └── webhooks/
│   │   │   │       └── stripe.rs          # Stripe webhook handler
│   │   │   └── ws/
│   │   │       ├── mod.rs                 # WebSocket upgrade handler
│   │   │       └── computation_progress.rs # Live progress streaming
│   │   ├── middleware/
│   │   │   └── plan_limits.rs             # Plan enforcement middleware
│   │   ├── services/
│   │   │   ├── usage.rs                   # Usage tracking service
│   │   │   ├── s3.rs                      # S3 client helpers
│   │   │   └── qspr_client.rs             # HTTP client for Python QSPR service
│   │   └── workers/
│   │       ├── mod.rs                     # Worker pool management
│   │       └── computation_worker.rs      # Server-side computation execution
│   ├── migrations/
│   │   └── 001_initial.sql                # Full database schema
│   └── tests/
│       ├── api_integration.rs             # API integration tests
│       └── computation_e2e.rs             # End-to-end computation tests
│
├── qspr-service/                          # Python FastAPI (QSPR sensitivity prediction)
│   ├── requirements.txt
│   ├── main.py                            # FastAPI app
│   ├── extractors/
│   │   └── molecular_descriptors.py       # RDKit descriptor extraction
│   ├── models/
│   │   ├── impact_sensitivity.py          # XGBoost h50 prediction
│   │   ├── friction_sensitivity.py        # Friction prediction
│   │   └── esd_sensitivity.py             # ESD prediction
│   ├── trained_models/
│   │   ├── impact_model.pkl               # Trained XGBoost models
│   │   ├── friction_model.pkl
│   │   └── esd_model.pkl
│   └── Dockerfile
│
├── frontend/                              # React frontend
│   ├── package.json
│   ├── vite.config.ts
│   ├── src/
│   │   ├── App.tsx                        # Router, providers, layout
│   │   ├── main.tsx                       # Entry point
│   │   ├── stores/
│   │   │   ├── authStore.ts               # Auth state (JWT, user)
│   │   │   └── formulationStore.ts        # Formulation builder state
│   │   ├── hooks/
│   │   │   ├── useComputationProgress.ts  # WebSocket hook for live progress
│   │   │   └── useWasmSolver.ts           # WASM solver hook (load + run)
│   │   ├── lib/
│   │   │   ├── api.ts                     # Axios API client
│   │   │   └── wasmLoader.ts              # WASM bundle loading
│   │   ├── pages/
│   │   │   ├── Dashboard.tsx              # Formulation list
│   │   │   ├── ReactantLibrary.tsx        # Browse reactants
│   │   │   ├── FormulationBuilder.tsx     # Formulation editor
│   │   │   ├── ComputationResults.tsx     # CJ state, product composition, plots
│   │   │   ├── SensitivityAnalysis.tsx    # QSPR sensitivity prediction
│   │   │   ├── Validation.tsx             # Computed vs. experimental validation
│   │   │   ├── Billing.tsx                # Plan management
│   │   │   ├── Login.tsx                  # Auth pages
│   │   │   └── Register.tsx
│   │   ├── components/
│   │   │   ├── FormulationBuilder/
│   │   │   │   ├── ComponentList.tsx      # Component mass fractions
│   │   │   │   ├── ReactantSearch.tsx     # Searchable reactant picker
│   │   │   │   └── TernaryDiagram.tsx     # D3.js ternary composition plot
│   │   │   ├── ComputationResults/
│   │   │   │   ├── CjStateCard.tsx        # CJ velocity, pressure, temperature display
│   │   │   │   ├── ProductCompositionChart.tsx  # Pie chart of products
│   │   │   │   ├── HugoniotPlot.tsx       # P-v Hugoniot curve (Plotly)
│   │   │   │   ├── PerformanceMap.tsx     # 2D contour plots
│   │   │   │   └── ParameterSweep.tsx     # Line plots for parameter sweeps
│   │   │   ├── Validation/
│   │   │   │   └── ScatterPlot.tsx        # Computed vs. experimental scatter
│   │   │   ├── billing/
│   │   │   │   ├── PlanCard.tsx
│   │   │   │   └── UsageMeter.tsx
│   │   │   └── common/
│   │   │       ├── ReactantCard.tsx       # Reactant property display
│   │   │       └── ComputationToolbar.tsx # Run/stop/EOS selector
│   │   └── data/
│   │       └── templates/                 # Pre-built formulation templates (JSON)
│   └── public/
│       └── wasm/                          # WASM solver bundle (loaded at runtime)
│
├── scripts/
│   ├── load_species.py                    # Parse Burcat database → PostgreSQL inserts
│   └── seed_reactants.sql                 # Seed common reactants (RDX, HMX, TNT, AN, Al, etc.)
│
├── k8s/                                   # Kubernetes manifests
│   ├── api-deployment.yaml
│   ├── worker-deployment.yaml
│   ├── qspr-deployment.yaml
│   ├── postgres-statefulset.yaml
│   ├── redis-deployment.yaml
│   └── ingress.yaml
│
├── docker-compose.yml                     # Local dev environment
├── Dockerfile                             # Rust API multi-stage build
└── README.md
```

---

## Validation Benchmarks

### Benchmark 1: TNT Detonation Velocity
- **Formulation**: TNT (C7H5N3O6) at ρ = 1.63 g/cm³
- **Expected CJ velocity**: 6.86 km/s (experimental, LASL Explosive Properties Data)
- **Acceptance**: Computed velocity within ±3% (6.66 - 7.06 km/s)
- **EOS**: BKWS (Hobbs-Baer parameterization)

### Benchmark 2: RDX/Wax (95/5) CJ Pressure
- **Formulation**: 95% RDX (C3H6N6O6) + 5% wax (C30H62) at ρ = 1.76 g/cm³
- **Expected CJ pressure**: 33.8 GPa (experimental, cylinder test data)
- **Acceptance**: Computed pressure within ±5% (32.1 - 35.5 GPa)
- **EOS**: BKW or BKWS

### Benchmark 3: Aluminized Explosive Product Composition
- **Formulation**: 80% RDX + 20% Al at ρ = 1.85 g/cm³
- **Expected**: Al₂O₃(s) formation, reduced CO₂, increased H₂O (aluminum scavenges oxygen)
- **Acceptance**: Al₂O₃ present in products at >15 mole%, CO₂ < 5 mole%
- **EOS**: BKWS with condensed phase handling

### Benchmark 4: ANFO Detonation Velocity vs. Density
- **Formulation**: 94% Ammonium Nitrate + 6% Fuel Oil
- **Density range**: 0.80 - 1.20 g/cm³
- **Expected**: D ≈ 3.5 km/s at 0.85 g/cm³, D ≈ 5.0 km/s at 1.15 g/cm³ (experimental curves)
- **Acceptance**: Computed D(ρ) curve within ±0.3 km/s across density range
- **EOS**: BKWC (calibrated for ANFO compositions)

### Benchmark 5: JWL Parameter Fit Accuracy
- **Formulation**: HMX (C4H8N8O8) at ρ = 1.891 g/cm³
- **Compute**: CJ state, then fit JWL isentrope
- **Expected JWL A**: ~778 GPa, B: ~7.07 GPa, R1: ~4.2, R2: ~1.0, ω: ~0.3 (published LS-DYNA parameters for HMX)
- **Acceptance**: Fitted JWL parameters within ±15% of published values, isentrope RMS error < 2%
- **EOS**: BKWS for CJ computation, JWL for isentrope fit

---

## Post-MVP Roadmap

### v1.1 — Advanced Detonation Physics (Weeks 13-16)
- **Cylinder test simulation**: Predict wall velocity vs. expansion ratio R, compare to experimental V(R) curves
- **Gurney energy and velocity**: Metal acceleration capability for warhead fragment velocity prediction
- **Failure diameter estimation**: Predict minimum charge diameter for sustained detonation from reaction zone length correlations
- **Rate stick experiments**: Detonation velocity vs. diameter curves for non-ideal explosives

### v1.2 — Propellant Internal Ballistics (Weeks 17-20)
- **Burn rate prediction**: r = a·P^n from thermochemical composition, flame temperature, gas evolution
- **Solid rocket motor performance**: Chamber pressure, thrust, Isp for specified grain geometry (BATES, finocyl, star)
- **Gun propellant**: Gas volume, force constant (impetus), covolume for gun internal ballistics codes
- **Minimum signature**: IR and visible plume spectral prediction from product composition

### v1.3 — Multi-User Collaboration (Weeks 21-22)
- **Real-time collaboration**: Multiple users editing same formulation simultaneously (Yjs CRDT)
- **Comments and annotations**: Add notes to formulation components, computation results
- **Version history**: Git-like version control for formulations with diff view
- **Team workspaces**: Shared formulation libraries, org-wide reactant database

### v1.4 — API and Integration (Weeks 23-24)
- **REST API**: Programmatic access to computation endpoints, species database, reactant library
- **Python SDK**: `pip install explodesafe-sdk`, example: `client.compute_cj_state(formulation)`
- **Hydrocode integration**: Direct export to LS-DYNA, AUTODYN, CTH, ALE3D input decks
- **Webhooks**: Notify external systems when computation completes

### v1.5 — Enterprise Features (Weeks 25-28)
- **On-premise deployment**: Air-gapped Kubernetes deployment for defense labs with export control requirements
- **Custom EOS calibration**: Fit BKW/JCZ3 parameters from proprietary experimental data
- **Custom species database**: Upload proprietary thermochemical data, manage access control
- **Audit trail**: Full record of who ran what computation, for QA and regulatory compliance
- **SSO integration**: SAML 2.0, LDAP, Active Directory for enterprise auth

---

## Success Metrics

### Technical Validation
- **Solver accuracy**: Mean absolute error <0.3 km/s for CJ velocity across 200+ experimental compounds (R² > 0.90)
- **Performance**: WASM solver completes 3-component formulation CJ state in <500ms (browser), server solver handles 10-component formulation in <5s
- **Convergence rate**: >95% of valid formulations converge within 50 iterations

### User Adoption
- **Month 1**: 200 signups (defense researchers, mining engineers, academic users)
- **Month 3**: 50 paying Pro subscribers ($149/mo), 5 paying Advanced teams ($349/user/mo × 5 users avg)
- **Month 6**: 150 paying subscribers, $30K MRR

### Product Quality
- **User-reported accuracy**: >90% of users report computed CJ velocities match their expectations or experimental data
- **Uptime**: 99.5% API availability
- **Support ticket resolution**: <24h median response time

---

## Risk Mitigation

### Risk 1: Solver convergence failures on exotic formulations
- **Mitigation**: Implement multiple fallback strategies (Gmin stepping equivalent for Gibbs minimization, temperature bracketing for CJ iteration), provide clear error messages with diagnostic guidance
- **Validation**: Test solver on 500+ diverse formulations (high/low density, fuel-rich/oxygen-rich, aluminized, ANFO, propellants)

### Risk 2: Export control restrictions on thermochemical codes
- **Mitigation**: ExplodeSafe is based on published algorithms (NASA CEA methodology, BKW EOS from open literature), does not include classified codes or proprietary data. Consult export control attorney for ITAR/EAR classification.
- **Contingency**: If export-controlled, implement country-based access restrictions, offer on-premise deployment for US government users

### Risk 3: Liability for misuse in improvised explosives
- **Mitigation**: Terms of Service prohibit illegal use, implement KYC verification for Pro/Advanced plans (verify employment at legitimate defense/mining/academic institution), log all computations for audit trail
- **Monitoring**: Flag suspicious patterns (e.g., user computing many improvised explosive formulations), report to authorities if warranted

### Risk 4: Competition from CHEETAH/Explo5 becoming publicly available
- **Mitigation**: Differentiate on modern UI/UX, cloud accessibility, integrated QSPR sensitivity prediction, hydrocode integration, team collaboration features. CHEETAH is unlikely to leave government-only distribution. Explo5 pricing and Windows-only limitation create opening.
- **Moat**: Build community library of validated formulations, accumulate user-contributed experimental data for validation

---

This implementation plan provides a comprehensive roadmap to build ExplodeSafe from initial scaffold through a production-ready MVP in 42 days, with clear phase boundaries, day-by-day task breakdown, detailed technical architecture including Rust Gibbs minimization and CJ solvers, WASM compilation for client-side performance, PostgreSQL species database with NASA polynomials, Python QSPR service for sensitivity prediction, React visualization with Plotly/D3, Stripe billing, and validated against 200+ experimental detonation velocities achieving target accuracy of MAE <0.3 km/s and R² >0.90.
