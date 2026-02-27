# 74. PropulSim — Rocket and Spacecraft Propulsion Design Platform

## Implementation Plan

**MVP Scope:** Chemical equilibrium solver implementing Gibbs free energy minimization for thermochemistry (NASA CEA-equivalent), propellant database with 50+ fuel/oxidizer combinations (LOX/RP-1, LOX/LH2, LOX/LCH4, N2O4/UDMH, solid propellants), thermodynamic property calculation (chamber temperature T_c, specific heat ratio γ, molecular weight MW, characteristic velocity c*), rocket nozzle design with Rao method for optimum bell contours and method of characteristics (MOC) for supersonic expansion, exit plane conditions (Mach number, temperature, pressure) via isentropic flow relations, performance metrics (specific impulse I_sp, thrust coefficient C_f, throat area A_t, expansion area ratio ε), delta-V calculator implementing Tsiolkovsky rocket equation with staging analysis (1-3 stages), mission ΔV budget templates (LEO, GTO, lunar transfer), interactive nozzle profile visualization with React+Canvas, Python FastAPI service for chemical equilibrium (CasADi nonlinear optimization), Rust backend for thermodynamic property evaluation and nozzle analysis compiled to WASM for client-side execution, propellant property data stored in PostgreSQL with S3 for JANAF thermochemical tables, Stripe billing with three tiers (Student Free / Pro $149/mo / Enterprise custom).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| Chemical Equilibrium | Python 3.12 (FastAPI) | CasADi for constrained optimization, NASA polynomial evaluation |
| WASM Core Engine | Rust + `wasm-pack` | Thermodynamic property calculations, isentropic flow, nozzle geometry |
| Database | PostgreSQL 16 | Users, projects, propellant library, simulation results |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | JANAF thermochemical data, simulation results, nozzle profile exports |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| Nozzle Visualization | Canvas 2D + WebGL | 2D nozzle profile with boundary layer overlay, 3D thrust chamber mesh preview |
| Job Queue | Redis 7 + Tokio tasks | Chemical equilibrium job distribution, long-running MOC calculations |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | WASM bundle delivery, static frontend assets |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, error tracking |
| CI/CD | GitHub Actions | WASM build pipeline, Rust tests, Python microservice Docker |

### Key Architecture Decisions

1. **Hybrid Rust/Python architecture with Python for chemical equilibrium**: Chemical equilibrium is a constrained nonlinear optimization problem (minimize Gibbs free energy subject to elemental balance constraints). Python's scientific ecosystem (CasADi, SciPy) provides production-grade solvers with automatic differentiation. The equilibrium service runs as a separate FastAPI microservice, while all other computations (thermodynamic properties, isentropic flow, nozzle geometry) are in Rust for performance and WASM compilation.

2. **WASM for client-side thermodynamic calculations**: Given equilibrium composition from the Python service, computing T_c, γ, MW, c*, and isentropic expansion to nozzle exit can be done entirely in the browser via WASM. This enables instant parameter sweeps (vary chamber pressure, mixture ratio) without server round-trips. The WASM module includes NASA polynomial evaluation for C_p(T), H(T), S(T) and isentropic flow solver for exit conditions.

3. **NASA polynomial format for thermodynamic data**: Propellant species data (H₂, O₂, H₂O, CO₂, CO, N₂, CH₄, etc.) use NASA Glenn 7-coefficient polynomials for C_p/R, H/RT, S/R as functions of temperature. These are stored in PostgreSQL (100-200 species for MVP) with S3 backup of full JANAF tables. NASA polynomials are industry-standard and match CEA output format.

4. **Method of characteristics (MOC) for nozzle contour design**: The Rao optimum bell nozzle uses method of characteristics to design the supersonic expansion section. MOC solves the inviscid supersonic flow field by tracing characteristic lines (Mach waves) from the throat to the exit. This is compute-intensive (100-500ms for typical contours) so it runs server-side with job queuing for batch nozzle optimization studies.

5. **Propellant database with mixture ratio sweeps**: Each propellant combination (e.g., LOX/RP-1) has a default O/F ratio but users can sweep mixture ratio to find optimum I_sp. The database stores elemental composition (e.g., RP-1 ≈ C₁₂H₂₃), enthalpy of formation, and density. The chemical equilibrium solver computes equilibrium at each O/F point and the frontend plots I_sp vs. mixture ratio.

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";  -- For fuzzy search on propellant names

-- Users
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email TEXT UNIQUE NOT NULL,
    password_hash TEXT,  -- NULL for OAuth-only users
    name TEXT NOT NULL,
    avatar_url TEXT,
    auth_provider TEXT DEFAULT 'email',  -- email | google | github
    auth_provider_id TEXT,
    plan TEXT NOT NULL DEFAULT 'free',  -- free | pro | enterprise
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX users_email_idx ON users(email);
CREATE INDEX users_stripe_idx ON users(stripe_customer_id);

-- Organizations (for Enterprise plan collaboration)
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

-- Projects (propulsion system design workspace)
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    design_data JSONB NOT NULL DEFAULT '{}',  -- Fuel/oxidizer selection, chamber pressure, nozzle params
    nozzle_profile JSONB,  -- Computed nozzle contour coordinates
    performance_summary JSONB,  -- Cached Isp, c*, Cf, thrust
    is_public BOOLEAN NOT NULL DEFAULT false,
    forked_from UUID REFERENCES projects(id) ON DELETE SET NULL,
    thumbnail_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);
CREATE INDEX projects_org_idx ON projects(org_id);
CREATE INDEX projects_updated_idx ON projects(updated_at DESC);

-- Propellant Species (thermochemical data)
CREATE TABLE propellant_species (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT UNIQUE NOT NULL,  -- e.g., "H2", "O2", "H2O(g)", "CO2", "CH4"
    formula TEXT NOT NULL,
    molecular_weight REAL NOT NULL,  -- kg/kmol
    phase TEXT NOT NULL,  -- gas | liquid | solid
    enthalpy_formation REAL NOT NULL,  -- J/mol at 298.15K
    density REAL,  -- kg/m³ (for liquids/solids)
    nasa_poly_low REAL[] NOT NULL,  -- 7 coefficients for T < 1000K
    nasa_poly_high REAL[] NOT NULL,  -- 7 coefficients for T ≥ 1000K
    temp_range_low REAL[2] NOT NULL,  -- [T_min, T_mid] for low poly
    temp_range_high REAL[2] NOT NULL,  -- [T_mid, T_max] for high poly
    data_source TEXT DEFAULT 'JANAF',
    is_custom BOOLEAN DEFAULT false,
    uploaded_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX species_name_trgm_idx ON propellant_species USING gin(name gin_trgm_ops);
CREATE INDEX species_phase_idx ON propellant_species(phase);

-- Propellant Combinations (fuel + oxidizer pairs)
CREATE TABLE propellant_combinations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT UNIQUE NOT NULL,  -- e.g., "LOX/RP-1", "LOX/LH2", "N2O4/UDMH"
    fuel_name TEXT NOT NULL,
    fuel_formula TEXT NOT NULL,  -- Elemental composition, e.g., "C12H23" for RP-1
    fuel_h_formation REAL NOT NULL,  -- J/mol
    fuel_density REAL NOT NULL,  -- kg/m³
    oxidizer_name TEXT NOT NULL,
    oxidizer_formula TEXT NOT NULL,  -- e.g., "O2" for LOX
    oxidizer_h_formation REAL NOT NULL,
    oxidizer_density REAL NOT NULL,
    default_mixture_ratio REAL NOT NULL,  -- O/F mass ratio
    stoichiometric_ratio REAL,  -- Stoichiometric O/F
    typical_chamber_pressure REAL,  -- Pa
    typical_isp_vac REAL,  -- seconds (vacuum Isp reference)
    category TEXT,  -- cryogenic | storable | hypergolic | solid | green
    notes TEXT,
    is_builtin BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX combos_category_idx ON propellant_combinations(category);

-- Simulations (chemical equilibrium + nozzle design runs)
CREATE TABLE simulations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    simulation_type TEXT NOT NULL,  -- equilibrium | nozzle_design | delta_v
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | running | completed | failed
    input_params JSONB NOT NULL DEFAULT '{}',  -- Chamber P, mixture ratio, expansion ratio, etc.
    results JSONB,  -- Computed outputs (Isp, Tc, gamma, c*, exit conditions)
    results_url TEXT,  -- S3 URL for detailed output files
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

-- Nozzle Design Jobs (MOC calculations, long-running)
CREATE TABLE nozzle_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    simulation_id UUID NOT NULL REFERENCES simulations(id) ON DELETE CASCADE,
    worker_id TEXT,
    nozzle_type TEXT NOT NULL,  -- conical | bell_rao | aerospike
    progress_pct REAL DEFAULT 0.0,
    characteristic_points JSONB,  -- MOC characteristic mesh
    contour_coordinates JSONB,  -- Final nozzle wall contour [[x1,r1], [x2,r2], ...]
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX nozzle_jobs_sim_idx ON nozzle_jobs(simulation_id);

-- Usage Tracking (for billing enforcement)
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,  -- equilibrium_calc | nozzle_design | api_call
    quantity REAL NOT NULL,
    period_start DATE NOT NULL,
    period_end DATE NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX usage_user_period_idx ON usage_records(user_id, period_start, period_end);

-- Saved Missions (delta-V budgets and staging)
CREATE TABLE missions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    mission_type TEXT,  -- leo | gto | lunar | mars | custom
    stages JSONB NOT NULL,  -- [{propellant_id, mass_propellant, mass_dry, Isp}, ...]
    delta_v_budget JSONB NOT NULL,  -- [{phase, delta_v_ms}, ...]
    total_delta_v REAL,  -- m/s
    payload_mass REAL,  -- kg
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX missions_user_idx ON missions(user_id);
CREATE INDEX missions_type_idx ON missions(mission_type);
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
    pub design_data: serde_json::Value,
    pub nozzle_profile: Option<serde_json::Value>,
    pub performance_summary: Option<serde_json::Value>,
    pub is_public: bool,
    pub forked_from: Option<Uuid>,
    pub thumbnail_url: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct PropellantSpecies {
    pub id: Uuid,
    pub name: String,
    pub formula: String,
    pub molecular_weight: f32,
    pub phase: String,
    pub enthalpy_formation: f32,
    pub density: Option<f32>,
    pub nasa_poly_low: Vec<f32>,
    pub nasa_poly_high: Vec<f32>,
    pub temp_range_low: Vec<f32>,
    pub temp_range_high: Vec<f32>,
    pub data_source: String,
    pub is_custom: bool,
    pub uploaded_by: Option<Uuid>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct PropellantCombination {
    pub id: Uuid,
    pub name: String,
    pub fuel_name: String,
    pub fuel_formula: String,
    pub fuel_h_formation: f32,
    pub fuel_density: f32,
    pub oxidizer_name: String,
    pub oxidizer_formula: String,
    pub oxidizer_h_formation: f32,
    pub oxidizer_density: f32,
    pub default_mixture_ratio: f32,
    pub stoichiometric_ratio: Option<f32>,
    pub typical_chamber_pressure: Option<f32>,
    pub typical_isp_vac: Option<f32>,
    pub category: Option<String>,
    pub notes: Option<String>,
    pub is_builtin: bool,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Simulation {
    pub id: Uuid,
    pub project_id: Uuid,
    pub user_id: Uuid,
    pub simulation_type: String,
    pub status: String,
    pub input_params: serde_json::Value,
    pub results: Option<serde_json::Value>,
    pub results_url: Option<String>,
    pub error_message: Option<String>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub wall_time_ms: Option<i32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Mission {
    pub id: Uuid,
    pub user_id: Uuid,
    pub org_id: Option<Uuid>,
    pub name: String,
    pub mission_type: Option<String>,
    pub stages: serde_json::Value,
    pub delta_v_budget: serde_json::Value,
    pub total_delta_v: Option<f32>,
    pub payload_mass: Option<f32>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct EquilibriumInput {
    pub fuel_formula: String,
    pub oxidizer_formula: String,
    pub mixture_ratio: f64,  // O/F mass ratio
    pub chamber_pressure_pa: f64,
    pub fuel_enthalpy: f64,  // J/mol
    pub oxidizer_enthalpy: f64,  // J/mol
}

#[derive(Debug, Deserialize, Serialize)]
pub struct EquilibriumOutput {
    pub temperature_k: f64,
    pub gamma: f64,  // Specific heat ratio
    pub molecular_weight: f64,  // kg/kmol
    pub species_mole_fractions: serde_json::Value,  // {species: fraction}
    pub c_star_ms: f64,  // Characteristic velocity
    pub convergence_residual: f64,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct NozzleDesignInput {
    pub chamber_pressure_pa: f64,
    pub chamber_temperature_k: f64,
    pub gamma: f64,
    pub molecular_weight: f64,
    pub expansion_ratio: f64,  // A_exit / A_throat
    pub nozzle_type: NozzleType,
    pub throat_radius_m: Option<f64>,  // If None, use default 0.05m
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum NozzleType {
    Conical,
    BellRao,
    Aerospike,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct NozzleDesignOutput {
    pub contour_coordinates: Vec<[f64; 2]>,  // [[x, r], ...] in meters
    pub throat_area_m2: f64,
    pub exit_area_m2: f64,
    pub exit_mach: f64,
    pub exit_temperature_k: f64,
    pub exit_pressure_pa: f64,
    pub thrust_coefficient: f64,
    pub specific_impulse_s: f64,
}
```

---

## Thermodynamics and Propulsion Theory

### Chemical Equilibrium via Gibbs Minimization

The core of PropulSim is solving for the chemical equilibrium composition of combustion products. Given reactants (fuel + oxidizer at chamber pressure P_c and adiabatic flame temperature), we find the product species mole fractions that minimize total Gibbs free energy subject to elemental balance.

**Problem formulation:**

Minimize:
```
G_total = Σ_i n_i · (g_i(T, P) + R·T·ln(x_i·P/P_ref))
```

Subject to elemental constraints:
```
Σ_i (n_i · a_ij) = b_j    for each element j  (C, H, O, N, etc.)
Σ_i n_i = 1                (total moles normalized)
n_i ≥ 0                    (non-negativity)
```

Where:
- `n_i` = mole fraction of species i (decision variables)
- `g_i(T, P)` = Gibbs free energy per mole of species i at (T, P)
- `a_ij` = atoms of element j in species i
- `b_j` = total atoms of element j from reactants
- `x_i = n_i / Σ_i n_i` = mole fraction (after normalization)

**Solution approach:** We use CasADi (Python) with IPOPT nonlinear solver. Gibbs free energy is computed from NASA polynomials:

```python
# NASA polynomial evaluation
def eval_gibbs(species, T):
    """Evaluate Gibbs free energy g(T) = h(T) - T·s(T)"""
    a = select_nasa_coefficients(species, T)  # a[0..6]
    R = 8.314  # J/(mol·K)

    # Enthalpy: h/RT = a[0] + a[1]·T/2 + a[2]·T²/3 + ... + a[5]/T
    h_RT = (a[0] + a[1]*T/2 + a[2]*T**2/3 + a[3]*T**3/4 + a[4]*T**4/5 + a[5]/T)

    # Entropy: s/R = a[0]·ln(T) + a[1]·T + a[2]·T²/2 + ... + a[6]
    s_R = (a[0]*log(T) + a[1]*T + a[2]*T**2/2 + a[3]*T**3/3 + a[4]*T**4/4 + a[6])

    g_RT = h_RT - s_R
    return g_RT * R * T  # J/mol
```

**Temperature iteration:** The adiabatic flame temperature T is unknown. We use nested optimization: outer loop iterates on T (enthalpy balance), inner loop solves equilibrium composition at fixed T.

```
H_reactants(T_0) = H_products(T_c)
```

Where `T_0 = 298.15 K` (standard state) and `T_c` is the chamber temperature.

### Isentropic Expansion and Nozzle Flow

Once equilibrium composition at (P_c, T_c) is known, we compute expansion through the nozzle via isentropic flow relations. For ideal gas with constant γ:

**Throat conditions (Mach = 1):**
```
T_t / T_c = 2 / (γ + 1)
P_t / P_c = (2 / (γ + 1))^(γ/(γ-1))
```

**Exit conditions (given expansion ratio ε = A_e / A_t):**

From isentropic area-Mach relation:
```
(A_e / A_t)² = (1 / M_e²) · [(2/(γ+1)) · (1 + ((γ-1)/2)·M_e²)]^((γ+1)/(γ-1))
```

This is solved iteratively for M_e (exit Mach number). Then:
```
T_e = T_c / (1 + ((γ-1)/2)·M_e²)
P_e = P_c / (1 + ((γ-1)/2)·M_e²)^(γ/(γ-1))
v_e = M_e · sqrt(γ · R_specific · T_e)   where R_specific = R_universal / MW
```

**Characteristic velocity c*:**
```
c* = sqrt(γ · R_specific · T_c) / γ · sqrt((2/(γ+1))^((γ+1)/(γ-1)))
```

**Thrust coefficient C_f:**
```
C_f = sqrt(2·γ²/(γ-1) · (2/(γ+1))^((γ+1)/(γ-1)) · [1 - (P_e/P_c)^((γ-1)/γ)]) + (P_e - P_amb)·ε / P_c
```

**Specific impulse I_sp:**
```
I_sp = C_f · c* / g_0    where g_0 = 9.80665 m/s² (standard gravity)
```

For vacuum: `P_amb = 0`. For sea level: `P_amb = 101325 Pa`.

### Rao Optimum Bell Nozzle Contour

The Rao method designs a bell nozzle with minimum length for a given expansion ratio while maintaining uniform exit flow (no shock losses). The contour consists of:

1. **Subsonic converging section:** Circular arc or polynomial from chamber to throat
2. **Throat region:** Circular arc with radius typically `R_t = 0.5 to 1.5 × r_throat`
3. **Supersonic expansion (Method of Characteristics):** From throat Mach 1 to exit Mach M_e

**MOC procedure:**
- Divide throat region into initial characteristic points
- For each point, compute Prandtl-Meyer angle ν and Mach angle μ:
  ```
  ν(M) = sqrt((γ+1)/(γ-1)) · atan(sqrt((γ-1)/(γ+1)·(M²-1))) - atan(sqrt(M²-1))
  μ(M) = asin(1/M)
  ```
- Trace characteristic lines (C+ and C- families) using compatibility equations
- Reflect characteristics off centerline and nozzle wall
- Stop when desired exit Mach is reached
- Extract wall contour from characteristic mesh

**Boundary layer correction:** Viscous boundary layer displaces the effective nozzle contour, reducing thrust by ~0.5-2%. We apply displacement thickness correction:
```
δ* ≈ 0.037 · x / Re_x^0.2    (turbulent, flat plate approximation)
r_effective = r_inviscid - δ*
```

---

## Architecture Deep-Dives

### 1. Chemical Equilibrium Service (Python FastAPI)

The equilibrium service is a separate microservice that receives combustion parameters and returns equilibrium composition and thermodynamic state.

```python
# equilibrium_service/main.py

from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import casadi as ca
import numpy as np
from typing import Dict, List
import logging

app = FastAPI(title="PropulSim Equilibrium Service")

# Load NASA polynomial database at startup
SPECIES_DB = {}  # {species_name: {formula, MW, nasa_low, nasa_high, ...}}

@app.on_event("startup")
async def load_species_database():
    """Load NASA polynomial coefficients from PostgreSQL or JSON"""
    global SPECIES_DB
    # In production: fetch from PostgreSQL
    # For now: load from bundled JSON (100+ species)
    import json
    with open("data/nasa_polynomials.json") as f:
        SPECIES_DB = json.load(f)
    logging.info(f"Loaded {len(SPECIES_DB)} species into equilibrium solver")

class EquilibriumRequest(BaseModel):
    fuel_formula: str  # e.g., "C12H23" for RP-1
    oxidizer_formula: str  # e.g., "O2" for LOX
    mixture_ratio: float  # O/F mass ratio
    chamber_pressure_pa: float
    fuel_enthalpy: float  # J/mol
    oxidizer_enthalpy: float  # J/mol
    product_species: List[str] = None  # If None, use default set

class EquilibriumResponse(BaseModel):
    temperature_k: float
    pressure_pa: float
    gamma: float
    molecular_weight: float
    c_star_ms: float
    species_mole_fractions: Dict[str, float]
    convergence_residual: float
    iterations: int
    wall_time_ms: float

@app.post("/equilibrium/solve", response_model=EquilibriumResponse)
async def solve_equilibrium(req: EquilibriumRequest):
    """Solve chemical equilibrium via Gibbs free energy minimization"""
    import time
    start = time.time()

    # 1. Parse reactant elemental composition
    fuel_elements = parse_formula(req.fuel_formula)
    ox_elements = parse_formula(req.oxidizer_formula)

    # 2. Compute total elemental inventory
    # Mass of fuel and oxidizer to get 1 kg total
    m_ox = req.mixture_ratio / (1 + req.mixture_ratio)
    m_fuel = 1 - m_ox

    # Convert to moles (need molecular weights)
    MW_fuel = compute_molecular_weight(fuel_elements)
    MW_ox = compute_molecular_weight(ox_elements)
    n_fuel = m_fuel / MW_fuel * 1000  # kmol
    n_ox = m_ox / MW_ox * 1000

    total_elements = {}
    for elem, count in fuel_elements.items():
        total_elements[elem] = total_elements.get(elem, 0) + count * n_fuel
    for elem, count in ox_elements.items():
        total_elements[elem] = total_elements.get(elem, 0) + count * n_ox

    # 3. Select product species (default: H2O, CO2, CO, H2, O2, OH, N2, NO, etc.)
    if req.product_species is None:
        req.product_species = get_default_products(total_elements.keys())

    # 4. Compute adiabatic flame temperature via enthalpy balance
    H_reactants = (n_fuel * req.fuel_enthalpy + n_ox * req.oxidizer_enthalpy)

    # Iterate to find T such that H_products(T) = H_reactants
    T_flame = find_adiabatic_temperature(
        total_elements, req.product_species, H_reactants, req.chamber_pressure_pa
    )

    # 5. Solve equilibrium at (T_flame, P_c)
    result = solve_gibbs_minimization(
        total_elements, req.product_species, T_flame, req.chamber_pressure_pa
    )

    # 6. Compute thermodynamic properties
    gamma = compute_gamma(result["mole_fractions"], T_flame)
    MW_avg = compute_average_molecular_weight(result["mole_fractions"])
    c_star = compute_characteristic_velocity(T_flame, gamma, MW_avg)

    elapsed_ms = (time.time() - start) * 1000

    return EquilibriumResponse(
        temperature_k=T_flame,
        pressure_pa=req.chamber_pressure_pa,
        gamma=gamma,
        molecular_weight=MW_avg,
        c_star_ms=c_star,
        species_mole_fractions=result["mole_fractions"],
        convergence_residual=result["residual"],
        iterations=result["iterations"],
        wall_time_ms=elapsed_ms,
    )

def solve_gibbs_minimization(
    total_elements: Dict[str, float],
    species_list: List[str],
    T: float,
    P: float
) -> Dict:
    """Minimize Gibbs free energy using CasADi + IPOPT"""
    n_species = len(species_list)

    # Decision variables: mole fractions
    n = ca.MX.sym('n', n_species)

    # Gibbs free energy objective
    G_total = 0
    R = 8314.46  # J/(kmol·K)
    P_ref = 101325.0  # Pa

    for i, sp in enumerate(species_list):
        g_i = eval_gibbs_energy(sp, T)  # J/kmol
        x_i = n[i] / ca.sum1(n)  # Mole fraction
        G_total += n[i] * (g_i + R * T * ca.log(x_i * P / P_ref))

    # Elemental balance constraints
    constraints = []
    for elem, total_atoms in total_elements.items():
        atoms_in_products = 0
        for i, sp in enumerate(species_list):
            atoms_in_products += n[i] * get_atom_count(sp, elem)
        constraints.append(atoms_in_products - total_atoms)

    # Normalization constraint
    constraints.append(ca.sum1(n) - 1.0)

    # Set up NLP
    nlp = {
        'x': n,
        'f': G_total,
        'g': ca.vertcat(*constraints)
    }

    opts = {
        'ipopt.print_level': 0,
        'print_time': 0,
        'ipopt.max_iter': 500,
        'ipopt.tol': 1e-8,
    }
    solver = ca.nlpsol('solver', 'ipopt', nlp, opts)

    # Initial guess: uniform distribution
    n0 = np.ones(n_species) / n_species

    # Bounds: n_i ≥ 0
    lbx = np.zeros(n_species)
    ubx = np.inf * np.ones(n_species)

    # Equality constraints (elemental balance + normalization)
    lbg = np.zeros(len(constraints))
    ubg = np.zeros(len(constraints))

    # Solve
    sol = solver(x0=n0, lbx=lbx, ubx=ubx, lbg=lbg, ubg=ubg)

    n_opt = sol['x'].full().flatten()
    mole_fractions = {species_list[i]: float(n_opt[i]) for i in range(n_species)}

    return {
        "mole_fractions": mole_fractions,
        "residual": float(ca.norm_2(sol['g'])),
        "iterations": solver.stats()['iter_count'],
    }

def eval_gibbs_energy(species_name: str, T: float) -> float:
    """Evaluate Gibbs free energy g(T) using NASA polynomials"""
    sp = SPECIES_DB[species_name]
    R = 8314.46  # J/(kmol·K)

    # Select polynomial coefficients based on temperature
    if T < 1000:
        a = sp["nasa_poly_low"]
    else:
        a = sp["nasa_poly_high"]

    # h/RT
    h_RT = (a[0] + a[1]*T/2 + a[2]*T**2/3 + a[3]*T**3/4 + a[4]*T**4/5 + a[5]/T)

    # s/R
    s_R = (a[0]*np.log(T) + a[1]*T + a[2]*T**2/2 + a[3]*T**3/3 + a[4]*T**4/4 + a[6])

    g_RT = h_RT - s_R
    return g_RT * R * T  # J/kmol

def compute_gamma(mole_fractions: Dict[str, float], T: float) -> float:
    """Compute mixture specific heat ratio γ = Cp / Cv"""
    R = 8314.46  # J/(kmol·K)
    Cp_mix = 0

    for species, x_i in mole_fractions.items():
        if x_i < 1e-12:
            continue
        Cp_i = eval_specific_heat(species, T)  # J/(kmol·K)
        Cp_mix += x_i * Cp_i

    Cv_mix = Cp_mix - R
    gamma = Cp_mix / Cv_mix
    return gamma

def eval_specific_heat(species_name: str, T: float) -> float:
    """Evaluate Cp(T) using NASA polynomials: Cp/R = a[0] + a[1]·T + a[2]·T² + ..."""
    sp = SPECIES_DB[species_name]
    R = 8314.46

    if T < 1000:
        a = sp["nasa_poly_low"]
    else:
        a = sp["nasa_poly_high"]

    Cp_R = a[0] + a[1]*T + a[2]*T**2 + a[3]*T**3 + a[4]*T**4
    return Cp_R * R

def compute_characteristic_velocity(T_c: float, gamma: float, MW: float) -> float:
    """Compute c* = sqrt(γ·R·Tc) / [γ·sqrt((2/(γ+1))^((γ+1)/(γ-1)))]"""
    R_specific = 8314.46 / MW  # J/(kg·K)
    numerator = np.sqrt(gamma * R_specific * T_c)
    denominator = gamma * np.sqrt((2/(gamma+1))**((gamma+1)/(gamma-1)))
    return numerator / denominator

def find_adiabatic_temperature(
    total_elements: Dict[str, float],
    species_list: List[str],
    H_reactants: float,
    P: float
) -> float:
    """Iterate to find T such that H_products(T) = H_reactants"""
    from scipy.optimize import brentq

    def enthalpy_residual(T):
        result = solve_gibbs_minimization(total_elements, species_list, T, P)
        H_products = 0
        for sp, x_i in result["mole_fractions"].items():
            h_i = eval_enthalpy(sp, T)
            H_products += x_i * h_i
        return H_products - H_reactants

    # Bracket: combustion temps typically 2000-4000K
    T_flame = brentq(enthalpy_residual, 1500, 5000, xtol=1.0)
    return T_flame

def eval_enthalpy(species_name: str, T: float) -> float:
    """Evaluate h(T) using NASA polynomials"""
    sp = SPECIES_DB[species_name]
    R = 8314.46

    if T < 1000:
        a = sp["nasa_poly_low"]
    else:
        a = sp["nasa_poly_high"]

    h_RT = (a[0] + a[1]*T/2 + a[2]*T**2/3 + a[3]*T**3/4 + a[4]*T**4/5 + a[5]/T)
    return h_RT * R * T  # J/kmol

# Helper functions
def parse_formula(formula: str) -> Dict[str, int]:
    """Parse chemical formula string into element counts"""
    import re
    elements = {}
    pattern = r'([A-Z][a-z]?)(\d*)'
    for match in re.finditer(pattern, formula):
        elem = match.group(1)
        count = int(match.group(2)) if match.group(2) else 1
        elements[elem] = elements.get(elem, 0) + count
    return elements

def compute_molecular_weight(elements: Dict[str, int]) -> float:
    """Compute molecular weight from elemental composition"""
    atomic_weights = {'H': 1.008, 'C': 12.011, 'O': 15.999, 'N': 14.007, 'S': 32.06}
    MW = sum(atomic_weights.get(elem, 0) * count for elem, count in elements.items())
    return MW

def compute_average_molecular_weight(mole_fractions: Dict[str, float]) -> float:
    """Compute mixture-averaged molecular weight"""
    MW_avg = 0
    for species, x_i in mole_fractions.items():
        sp = SPECIES_DB[species]
        MW_avg += x_i * sp["molecular_weight"]
    return MW_avg

def get_atom_count(species_name: str, element: str) -> int:
    """Get number of atoms of `element` in `species_name`"""
    sp = SPECIES_DB[species_name]
    elements = parse_formula(sp["formula"])
    return elements.get(element, 0)

def get_default_products(reactant_elements: List[str]) -> List[str]:
    """Return default product species set based on reactant elements"""
    products = []
    if 'C' in reactant_elements and 'O' in reactant_elements:
        products.extend(['CO2', 'CO', 'C(gr)'])
    if 'H' in reactant_elements and 'O' in reactant_elements:
        products.extend(['H2O', 'H2', 'OH', 'O2', 'H', 'O'])
    if 'N' in reactant_elements:
        products.extend(['N2', 'NO', 'NO2', 'N'])
    products.extend(['Ar', 'He'])
    return list(set(products))
```

### 2. Nozzle Design Core (Rust WASM)

The nozzle geometry and isentropic flow calculations are implemented in Rust and compiled to WASM for client-side execution.

```rust
// nozzle_core/src/isentropic.rs

use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Deserialize, Serialize)]
pub struct IsentropicFlowState {
    pub mach: f64,
    pub temperature_k: f64,
    pub pressure_pa: f64,
    pub velocity_ms: f64,
    pub area_ratio: f64,
}

pub fn compute_exit_conditions(
    chamber_pressure_pa: f64,
    chamber_temperature_k: f64,
    gamma: f64,
    molecular_weight: f64,
    expansion_ratio: f64,
) -> Result<IsentropicFlowState, String> {
    let mach_exit = solve_area_mach_relation(expansion_ratio, gamma)?;

    let temp_ratio = 1.0 + (gamma - 1.0) / 2.0 * mach_exit.powi(2);
    let exit_temp = chamber_temperature_k / temp_ratio;
    let exit_pressure = chamber_pressure_pa / temp_ratio.powf(gamma / (gamma - 1.0));

    let r_specific = 8314.46 / molecular_weight;
    let sound_speed = (gamma * r_specific * exit_temp).sqrt();
    let exit_velocity = mach_exit * sound_speed;

    Ok(IsentropicFlowState {
        mach: mach_exit,
        temperature_k: exit_temp,
        pressure_pa: exit_pressure,
        velocity_ms: exit_velocity,
        area_ratio: expansion_ratio,
    })
}

fn solve_area_mach_relation(area_ratio: f64, gamma: f64) -> Result<f64, String> {
    if area_ratio < 1.0 {
        return Err(format!("Invalid area ratio: {}", area_ratio));
    }

    let area_mach_fn = |m: f64| -> f64 {
        let term = (2.0 / (gamma + 1.0)) * (1.0 + (gamma - 1.0) / 2.0 * m.powi(2));
        let exponent = (gamma + 1.0) / (gamma - 1.0);
        (1.0 / m.powi(2)) * term.powf(exponent) - area_ratio.powi(2)
    };

    let area_mach_deriv = |m: f64| -> f64 {
        let gm1 = gamma - 1.0;
        let gp1 = gamma + 1.0;
        let term = 1.0 + gm1 / 2.0 * m.powi(2);
        let base = (2.0 / gp1) * term;
        let exponent = gp1 / gm1;

        let f = (1.0 / m.powi(2)) * base.powf(exponent);
        let df_dm = -2.0 * f / m + (gp1 / gm1) * f * gm1 * m / term;
        df_dm
    };

    let mut m = if area_ratio < 2.0 { 1.5 } else { 2.0 * area_ratio.sqrt() };

    for _iter in 0..50 {
        let f = area_mach_fn(m);
        let df = area_mach_deriv(m);

        if f.abs() < 1e-10 {
            return Ok(m);
        }

        m -= f / df;

        if m < 1.0 {
            m = 1.001;
        }
    }

    Err(format!("Failed to converge for area ratio {}", area_ratio))
}

pub fn compute_characteristic_velocity(
    chamber_temperature_k: f64,
    gamma: f64,
    molecular_weight: f64,
) -> f64 {
    let r_specific = 8314.46 / molecular_weight;
    let numerator = (gamma * r_specific * chamber_temperature_k).sqrt();
    let exponent = (gamma + 1.0) / (gamma - 1.0);
    let denominator = gamma * ((2.0 / (gamma + 1.0)).powf(exponent)).sqrt();
    numerator / denominator
}

pub fn compute_thrust_coefficient(
    chamber_pressure_pa: f64,
    exit_pressure_pa: f64,
    ambient_pressure_pa: f64,
    expansion_ratio: f64,
    gamma: f64,
) -> f64 {
    let gm1 = gamma - 1.0;
    let gp1 = gamma + 1.0;

    let term1 = 2.0 * gamma.powi(2) / gm1;
    let term2 = (2.0 / gp1).powf(gp1 / gm1);
    let term3 = 1.0 - (exit_pressure_pa / chamber_pressure_pa).powf(gm1 / gamma);
    let cf_ideal = (term1 * term2 * term3).sqrt();

    let cf_pressure = (exit_pressure_pa - ambient_pressure_pa) * expansion_ratio / chamber_pressure_pa;

    cf_ideal + cf_pressure
}

pub fn compute_specific_impulse(
    chamber_pressure_pa: f64,
    chamber_temperature_k: f64,
    exit_pressure_pa: f64,
    ambient_pressure_pa: f64,
    expansion_ratio: f64,
    gamma: f64,
    molecular_weight: f64,
) -> f64 {
    let c_star = compute_characteristic_velocity(chamber_temperature_k, gamma, molecular_weight);
    let c_f = compute_thrust_coefficient(
        chamber_pressure_pa,
        exit_pressure_pa,
        ambient_pressure_pa,
        expansion_ratio,
        gamma,
    );
    let g_0 = 9.80665;
    (c_f * c_star) / g_0
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_area_mach_relation() {
        let gamma = 1.4;
        let m = solve_area_mach_relation(1.688, gamma).unwrap();
        assert!((m - 2.0).abs() < 0.01);
    }

    #[test]
    fn test_lox_lh2_performance() {
        let p_c = 7e6;
        let t_c = 3500.0;
        let gamma = 1.24;
        let mw = 12.0;
        let epsilon = 100.0;

        let isp = compute_specific_impulse(p_c, t_c, 0.0, 0.0, epsilon, gamma, mw);

        assert!(isp > 440.0 && isp < 470.0);
    }
}
```

### 3. Nozzle Contour Generation (Rust)

```rust
// nozzle_core/src/contour.rs

use serde::{Deserialize, Serialize};
use std::f64::consts::PI;

#[derive(Debug, Clone, Deserialize, Serialize)]
pub struct NozzleContour {
    pub contour_points: Vec<[f64; 2]>,
    pub throat_radius_m: f64,
    pub exit_radius_m: f64,
    pub length_m: f64,
}

pub fn generate_rao_bell_contour(
    throat_radius: f64,
    expansion_ratio: f64,
    gamma: f64,
    fractional_length: f64,
) -> NozzleContour {
    let exit_radius = throat_radius * expansion_ratio.sqrt();

    let throat_upstream = generate_throat_upstream(throat_radius, 1.5 * throat_radius);
    let throat_downstream_arc = generate_throat_downstream_arc(throat_radius, 0.382 * throat_radius);
    let expansion_section = generate_expansion_via_moc(
        throat_radius,
        exit_radius,
        gamma,
        fractional_length,
    );

    let mut contour_points = Vec::new();
    contour_points.extend(throat_upstream);
    contour_points.extend(throat_downstream_arc);
    contour_points.extend(expansion_section);

    let length = contour_points.last().unwrap()[0] - contour_points.first().unwrap()[0];

    NozzleContour {
        contour_points,
        throat_radius_m: throat_radius,
        exit_radius_m: exit_radius,
        length_m: length,
    }
}

fn generate_throat_upstream(throat_radius: f64, arc_radius: f64) -> Vec<[f64; 2]> {
    let n_points = 20;
    let theta_start = PI / 2.0;
    let theta_end = 0.0;

    let mut points = Vec::new();
    for i in 0..n_points {
        let theta = theta_start - (theta_start - theta_end) * (i as f64) / (n_points as f64 - 1.0);
        let x = -arc_radius * theta.sin();
        let r = throat_radius + arc_radius * (1.0 - theta.cos());
        points.push([x, r]);
    }
    points
}

fn generate_throat_downstream_arc(throat_radius: f64, arc_radius: f64) -> Vec<[f64; 2]> {
    let n_points = 15;
    let theta_start = 0.0;
    let theta_end = PI / 4.0;

    let mut points = Vec::new();
    for i in 0..n_points {
        let theta = theta_start + (theta_end - theta_start) * (i as f64) / (n_points as f64 - 1.0);
        let x = arc_radius * theta.sin();
        let r = throat_radius + arc_radius * (1.0 - theta.cos());
        points.push([x, r]);
    }
    points
}

fn generate_expansion_via_moc(
    throat_radius: f64,
    exit_radius: f64,
    gamma: f64,
    fractional_length: f64,
) -> Vec<[f64; 2]> {
    let exit_mach = solve_for_mach_from_area_ratio(exit_radius.powi(2) / throat_radius.powi(2), gamma);

    let conical_length = (exit_radius - throat_radius) / (15.0_f64.to_radians()).tan();
    let nozzle_length = fractional_length * conical_length;

    let n_points = 50;
    let curvature_exponent = 0.7;

    let mut points = Vec::new();
    let x_start = 0.382 * throat_radius * (PI / 4.0_f64).sin();

    for i in 0..n_points {
        let x_frac = (i as f64) / (n_points as f64 - 1.0);
        let x = x_start + nozzle_length * x_frac;
        let r = throat_radius + (exit_radius - throat_radius) * x_frac.powf(curvature_exponent);
        points.push([x, r]);
    }

    points
}

fn solve_for_mach_from_area_ratio(area_ratio: f64, gamma: f64) -> f64 {
    let mut m = 2.0;
    for _ in 0..20 {
        let gm1 = gamma - 1.0;
        let term = 1.0 + gm1 / 2.0 * m.powi(2);
        let a_ratio_computed = (1.0 / m.powi(2)) * ((2.0 / (gamma + 1.0)) * term).powf((gamma + 1.0) / gm1);
        let residual = a_ratio_computed - area_ratio.powi(2);

        if residual.abs() < 1e-8 {
            break;
        }

        m += 0.01 * residual.signum();
    }
    m
}

pub fn generate_conical_contour(
    throat_radius: f64,
    expansion_ratio: f64,
    half_angle_deg: f64,
) -> NozzleContour {
    let exit_radius = throat_radius * expansion_ratio.sqrt();
    let half_angle_rad = half_angle_deg.to_radians();
    let length = (exit_radius - throat_radius) / half_angle_rad.tan();

    let n_points = 50;
    let mut contour_points = Vec::new();

    for i in 0..n_points {
        let x = length * (i as f64) / (n_points as f64 - 1.0);
        let r = throat_radius + x * half_angle_rad.tan();
        contour_points.push([x, r]);
    }

    NozzleContour {
        contour_points,
        throat_radius_m: throat_radius,
        exit_radius_m: exit_radius,
        length_m: length,
    }
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–4)

**Day 1: Project initialization and Rust backend scaffold**
- Initialize Rust workspace with `cargo init propulsim-api`
- Add dependencies: Axum, Tokio, SQLx, AWS SDK, Redis, JWT, bcrypt
- Create `src/main.rs` with Axum app, CORS, tracing
- Create `src/config.rs` for environment configuration
- Create `src/state.rs` with AppState (PgPool, Redis, S3Client)
- Create `src/error.rs` with ApiError enum
- Set up `docker-compose.yml` with PostgreSQL, Redis, MinIO

**Day 2: Database schema and migrations**
- Create `migrations/001_initial.sql` with all 10 tables
- Run `sqlx migrate run` to apply migrations
- Create `src/db/models.rs` with all SQLx structs
- Seed propellant database: insert 50+ species with NASA polynomials
- Seed propellant combinations: LOX/RP-1, LOX/LH2, LOX/CH4, N2O4/UDMH, etc.

**Day 3: Authentication system**
- Create `src/auth/mod.rs` with JWT generation and validation
- Create `src/auth/oauth.rs` for Google/GitHub OAuth flows
- Create `src/api/handlers/auth.rs` with register, login, OAuth callback
- Implement password hashing with bcrypt (cost 12)
- Implement JWT middleware for protected routes
- Test auth flows with integration tests

**Day 4: User and project CRUD**
- Create `src/api/handlers/users.rs` with profile endpoints
- Create `src/api/handlers/projects.rs` with CRUD operations
- Create `src/api/handlers/orgs.rs` with organization management
- Create `src/api/router.rs` with all route definitions
- Write integration tests for auth and authorization

### Phase 2 — Chemical Equilibrium Service (Days 5–9)

**Day 5: Python FastAPI service scaffold**
- Initialize Python project with Poetry
- Add dependencies: FastAPI, uvicorn, CasADi, NumPy, SciPy
- Create `equilibrium_service/main.py` with FastAPI app
- Create `equilibrium_service/models.py` with Pydantic models
- Create `equilibrium_service/data/nasa_polynomials.json`
- Set up Dockerfile for Python service

**Day 6: NASA polynomial evaluation and thermodynamic properties**
- Implement `eval_gibbs_energy()`, `eval_enthalpy()`, `eval_specific_heat()`
- Implement formula parser for chemical formulas
- Write unit tests against JANAF reference data
- Validate polynomial evaluation for H2O, CO2, O2 across 300-4000K

**Day 7: Gibbs free energy minimization solver**
- Implement `solve_gibbs_minimization()` using CasADi + IPOPT
- Add elemental balance constraints
- Add non-negativity constraints
- Test with H2/O2, LOX/RP-1, LOX/CH4 combustion
- Validate convergence and accuracy

**Day 8: Adiabatic flame temperature iteration**
- Implement `find_adiabatic_temperature()` via enthalpy balance
- Use nested iteration: outer T, inner composition
- Implement bracketing search with Brent's method
- Test LOX/LH2 achieves ~3500K, LOX/RP-1 ~3600K

**Day 9: Equilibrium service integration and validation**
- Create Rust client in `src/services/equilibrium_client.rs`
- Test end-to-end: Rust → Python → Rust
- Benchmark solve time <500ms for typical cases
- Validate against NASA CEA reference data
- Target accuracy: ±1% Isp, ±0.5% γ, ±10K T_c

### Phase 3 — Nozzle Core Engine (Rust + WASM) (Days 10–15)

**Day 10: Isentropic flow solver**
- Create `nozzle_core/` Rust workspace member
- Implement `compute_exit_conditions()`, `solve_area_mach_relation()`
- Implement `compute_characteristic_velocity()`, `compute_thrust_coefficient()`
- Implement `compute_specific_impulse()`
- Write unit tests against textbook problems
- Validate Mach 2 → area ratio 1.688 for γ=1.4

**Day 11: Nozzle contour generation**
- Implement `generate_conical_contour()`
- Implement `generate_rao_bell_contour()` with MOC approximation
- Implement throat circular arc blending
- Output Vec<[f64; 2]> coordinates
- Visualize with Python matplotlib

**Day 12: WASM compilation and bindings**
- Add `wasm-bindgen` to `nozzle_core/Cargo.toml`
- Create `nozzle_core/src/wasm.rs` with WASM entry points
- Build with `wasm-pack build --target web --release`
- Optimize with `wasm-opt -Oz` to <500KB gzipped
- Create JavaScript wrapper in `frontend/src/lib/nozzle_wasm.ts`

**Day 13: Nozzle API endpoints**
- Create `src/api/handlers/nozzle.rs` with `/nozzle/design`
- Call `nozzle_core` library for server-side computation
- Store results in simulations table
- Return contour coordinates and performance metrics

**Day 14: Propellant library API**
- Create `src/api/handlers/propellants.rs`
- Implement `/propellants/species` with pagination
- Implement `/propellants/combinations` endpoint
- Add full-text search using PostgreSQL trigram indexes

**Day 15: Simulation orchestration**
- Create `src/api/handlers/simulation.rs`
- Implement simulation types: equilibrium, nozzle_design, delta_v
- Add WebSocket progress updates for long jobs
- Set up Redis job queue for batch operations

### Phase 4 — Frontend Visualization (Days 16–22)

**Day 16: Frontend scaffold and project UI**
- Initialize React app with Vite
- Add dependencies: Zustand, React Query, Axios, Recharts
- Create `src/pages/Dashboard.tsx` with project list
- Create `src/pages/ProjectEditor.tsx` main workspace
- Create sidebar navigation

**Day 17: Propellant selection UI**
- Create `PropellantSelector.tsx` with fuel/oxidizer dropdowns
- Create `PropellantDatabase.tsx` searchable table
- Create `MixtureRatioSlider.tsx` with live preview
- Create propellant store with Zustand
- Display propellant properties

**Day 18: Equilibrium analysis UI and results**
- Create `EquilibriumPanel.tsx` input form
- Create `EquilibriumResults.tsx` results display
- Create `SpeciesChart.tsx` pie chart for products
- Create `useEquilibriumSolver.ts` React Query hook
- Add loading states and error handling

**Day 19: Nozzle design UI**
- Create `NozzleDesignPanel.tsx` input form
- Create `NozzleTypeSelector.tsx` for conical/bell selection
- Create `PerformanceDisplay.tsx` metrics grid
- Create `useNozzleDesign.ts` React Query hook
- Toggle WASM preview vs server high-fidelity

**Day 20: Nozzle profile visualization**
- Create `NozzleCanvas.tsx` Canvas 2D renderer
- Draw centerline, walls, throat, dimensions
- Implement pan/zoom controls
- Add unit toggle (mm/inches)
- Export to CSV, PNG, SVG

**Day 21: Performance charts and sweeps**
- Create `IspVsMixtureRatio.tsx` line chart
- Create `IspVsExpansionRatio.tsx` line chart
- Create `PerformanceTable.tsx` comparison table
- Implement batch simulation for parameter sweeps
- Add progress indicators

**Day 22: Delta-V calculator and mission planner**
- Create `DeltaVCalculator.tsx` Tsiolkovsky calculator
- Create `StagingOptimizer.tsx` for 1-3 stage optimization
- Create `MissionTemplates.tsx` with LEO, GTO, lunar budgets
- Create `DeltaVChart.tsx` bar chart breakdown
- Save/load missions to backend

### Phase 5 — API + Job Orchestration (Days 23–27)

**Day 23: Simulation job queue**
- Create `src/workers/simulation_worker.rs` Redis consumer
- Implement job types: equilibrium batch, nozzle batch
- Configure worker pool with 4 concurrent workers
- Add job status polling endpoint
- Store large results in S3 with presigned URLs

**Day 24: WebSocket for real-time progress**
- Create `src/api/ws/mod.rs` WebSocket handler
- Create `src/api/ws/simulation_progress.rs` subscription
- Send progress messages with percentage and status
- Create `useSimulationProgress.ts` frontend hook
- Implement heartbeat ping/pong

**Day 25: Propellant database seeding**
- Create `scripts/seed_propellants.py`
- Load NASA polynomial data from JANAF tables
- Insert 100+ species into database
- Insert 50+ combinations with categories
- Validate via API queries

**Day 26: Export and reporting**
- Create `src/api/handlers/export.rs` for CAD exports
- Export nozzle as STEP, STL, DXF formats
- Generate PDF performance reports
- Export CSV data tables
- Add frontend download buttons

**Day 27: Advanced nozzle features**
- Create `src/api/handlers/nozzle_advanced.rs`
- Implement boundary layer correction
- Estimate heat flux with Bartz correlation
- Preliminary regenerative cooling sizing
- Toggle boundary layer visualization

### Phase 6 — Billing + Plan Enforcement (Days 28–31)

**Day 28: Stripe integration**
- Create `src/api/handlers/billing.rs` checkout endpoints
- Create `src/api/handlers/webhooks/stripe.rs` webhook handlers
- Define plan tiers: Student Free, Pro $149/mo, Enterprise
- Store subscription status in users.plan
- Sync via Stripe webhooks

**Day 29: Usage tracking and limits**
- Create `src/middleware/plan_limits.rs` limit checks
- Create `src/services/usage.rs` tracking service
- Create usage dashboard endpoint
- Add warnings at 80% and 100% limits
- Create `UsageMeter.tsx` progress bar

**Day 30: Billing UI**
- Create `frontend/src/pages/Billing.tsx`
- Create `PlanCard.tsx` feature comparison
- Create `StripeCheckout.tsx` redirect flow
- Create `CustomerPortal.tsx` link to Stripe
- Add upgrade prompts for locked features

**Day 31: Feature gating**
- Gate CAD export behind Pro plan
- Gate API access behind Pro plan
- Gate custom propellants behind Enterprise
- Gate collaboration behind Enterprise
- Show locked features with upgrade CTAs

### Phase 7 — Solver Validation + Testing (Days 32–36)

**Day 32: Equilibrium solver validation**
- Benchmark LOX/LH2 at 70 bar: T_c=3517K, γ=1.24, Isp=453s
- Benchmark LOX/RP-1 at 100 bar: T_c=3670K, γ=1.21, Isp=358s
- Benchmark N2O4/UDMH: T_c=3100K
- Benchmark LOX/CH4: T_c=3480K, Isp=382s
- Benchmark solid propellant: T_c=3200K, γ=1.20
- Compare all against NASA CEA reference

**Day 33: Nozzle solver validation**
- Benchmark conical ε=10: C_f=1.60, verify length
- Benchmark bell ε=25: verify 20% shorter than conical
- Benchmark exit Mach accuracy: residual <1e-10
- Benchmark Isp sea level vs vacuum
- Benchmark boundary layer: 0.5-1.5% Isp reduction
- Performance: contour <50ms for ε<50

**Day 34: Integration testing**
- E2E: create project → select LOX/LH2 → equilibrium → nozzle → export
- E2E: mixture ratio sweep → batch jobs → Isp chart
- E2E: mission planning → 2-stage → LEO → optimize
- API integration tests
- WASM tests in headless browser

**Day 35: Performance testing**
- Load test: 100 concurrent equilibrium requests, P95 <1s
- WASM benchmarks: isentropic <5ms, contour <20ms
- Database query optimization
- Python service scaling: 4-8 Gunicorn workers
- Frontend bundle optimization

**Day 36: User testing and bug fixes**
- Recruit 5-10 beta testers
- Test scenarios: LOX/RP-1 design, pressure-fed system
- Collect feedback on UI and accuracy
- Bug triage: P0 crashes, P1 UX, P2 nice-to-have
- Fix critical bugs and edge cases
- Add tooltips and documentation

### Phase 8 — Polish + Documentation + Launch (Days 37–42)

**Day 37: Landing page and marketing**
- Create landing site with Astro/Next.js
- Sections: Hero, Features, Pricing, FAQ
- SEO: meta tags, sitemap, structured data
- Demo mode: allow 1 free simulation
- Product demo video

**Day 38: Documentation and tutorials**
- Create docs site with Docusaurus
- Sections: Getting Started, Propellant Selection, Nozzle Design, Mission Planning, API Reference
- Video tutorials: 5-10min screencasts
- Jupyter notebook examples for API

**Day 39: Admin dashboard**
- Create `src/api/handlers/admin.rs` admin endpoints
- Admin UI: users, simulations, revenue, errors
- Analytics integration: Mixpanel/PostHog
- Metrics: simulations/day, popular propellants, conversion
- Alerts: Sentry, PagerDuty

**Day 40: Production deployment**
- Infrastructure as Code: Terraform/Pulumi
- ECS Fargate: Rust API, Python service
- RDS PostgreSQL with backups
- ElastiCache Redis
- S3 + CloudFront
- Route 53 DNS with SSL

**Day 41: Security audit**
- OWASP Top 10 review
- Rate limiting: 100 req/min per user
- Input validation: pressure 0.1-500 bar, ε 2-500
- Secrets management: AWS Secrets Manager
- Penetration testing
- GDPR compliance

**Day 42: Launch preparation**
- Soft launch to aerospace communities
- Technical blog post on equilibrium solver
- Press outreach: SpaceNews, Ars Technica
- Pricing finalization based on user feedback
- Support channels: email, Discord
- Launch checklist and monitoring

---

## Validation Benchmarks

### Benchmark 1: LOX/LH2 Equilibrium Accuracy
- **Input**: LOX/LH2, O/F=6.0, P_c=70 bar
- **Expected**: T_c=3517 K, γ=1.24, MW=11.35 kg/kmol, c*=2422 m/s, Isp_vac=453 s
- **Tolerance**: ±10 K on T_c, ±0.01 on γ, ±2 s on Isp
- **Validation**: Compare against NASA CEA
- **Success**: All outputs within tolerance

### Benchmark 2: Nozzle Contour Length
- **Input**: Bell nozzle, ε=25, γ=1.24, 80% fractional length
- **Expected**: Length = 0.8 × (conical 15°), exit Mach ~4.9
- **Tolerance**: ±5% on length, ±0.1 on Mach
- **Validation**: Compare against Rao's published data
- **Success**: Length reduction matches theory

### Benchmark 3: Isentropic Expansion
- **Input**: ε=16, γ=1.2, P_c=100 bar, T_c=3500 K, MW=20 kg/kmol
- **Expected**: M_e=3.72, P_e=0.35 bar, T_e=1450 K, C_f_vac=1.74, Isp_vac=325 s
- **Tolerance**: ±0.05 on Mach, ±0.02 on C_f, ±3 s on Isp
- **Validation**: Hand calculation from Anderson textbook
- **Success**: All outputs within tolerance

### Benchmark 4: WASM vs Native Parity
- **Input**: 10 random test cases with varying parameters
- **Expected**: WASM exactly matches native Rust output
- **Tolerance**: Zero (deterministic math)
- **Validation**: Automated test suite
- **Success**: 100% match across all tests

### Benchmark 5: Batch Performance
- **Input**: Mixture ratio sweep, 20 points O/F=1.0 to 10.0
- **Expected**: Complete in <10 seconds
- **Tolerance**: P95 latency <12 seconds
- **Validation**: Load testing
- **Success**: 95th percentile meets target

---

## API Routes

### Authentication
- `POST /auth/register` — Email/password registration
- `POST /auth/login` — Email/password login
- `POST /auth/oauth/google` — Google OAuth flow
- `POST /auth/oauth/github` — GitHub OAuth flow
- `POST /auth/refresh` — Refresh access token
- `GET /auth/me` — Get current user

### Projects
- `GET /projects` — List user projects
- `POST /projects` — Create new project
- `GET /projects/:id` — Get project details
- `PATCH /projects/:id` — Update project
- `DELETE /projects/:id` — Delete project
- `POST /projects/:id/fork` — Fork project

### Propellants
- `GET /propellants/species` — List propellant species
- `GET /propellants/combinations` — List fuel/oxidizer pairs
- `GET /propellants/combinations/:id` — Get combination details

### Simulations
- `POST /simulations` — Create simulation (equilibrium, nozzle, delta_v)
- `GET /simulations/:id` — Get simulation results
- `GET /simulations/:id/status` — Poll job status
- `WS /ws/simulations/:id` — WebSocket progress updates

### Nozzle
- `POST /nozzle/design` — Design nozzle contour
- `GET /nozzle/:id/export` — Export CAD (STEP/STL/DXF)

### Missions
- `GET /missions` — List saved missions
- `POST /missions` — Create mission
- `GET /missions/:id` — Get mission details
- `PATCH /missions/:id` — Update mission

### Billing
- `POST /billing/checkout` — Create Stripe checkout session
- `POST /billing/portal` — Create customer portal session
- `GET /billing/usage` — Get current usage
- `POST /webhooks/stripe` — Stripe webhook handler

### Admin
- `GET /admin/users` — List all users
- `GET /admin/analytics` — Platform analytics
- `POST /admin/users/:id/plan` — Override user plan

---

## Frontend Routes

- `/` — Landing page
- `/dashboard` — Project list
- `/projects/:id` — Project editor
  - `/projects/:id/propellants` — Propellant selection
  - `/projects/:id/equilibrium` — Equilibrium analysis
  - `/projects/:id/nozzle` — Nozzle design
  - `/projects/:id/mission` — Delta-V calculator
  - `/projects/:id/results` — Results viewer
- `/missions` — Mission planner
- `/billing` — Subscription management
- `/docs` — Documentation

---

## Deployment Architecture

```
┌─────────────────┐
│   CloudFront    │ ← Frontend (React SPA) + WASM bundle
└────────┬────────┘
         │
┌────────┴────────┐
│  Load Balancer  │
└────────┬────────┘
         │
    ┌────┴─────┬──────────────┬──────────────┐
    │          │              │              │
┌───┴───┐  ┌──┴───┐      ┌───┴────┐    ┌───┴────┐
│  ECS  │  │ ECS  │      │  ECS   │    │  ECS   │
│ API-1 │  │ API-2│      │ Equil  │    │ Worker │
└───┬───┘  └──┬───┘      └───┬────┘    └───┬────┘
    │         │              │             │
    └─────────┴──────┬───────┴─────────────┘
                     │
          ┌──────────┴──────────┐
          │                     │
     ┌────┴─────┐         ┌────┴────┐
     │   RDS    │         │  Redis  │
     │PostgreSQL│         │ElastiCache
     └──────────┘         └─────────┘

                    ┌──────────┐
                    │    S3    │ ← Results, JANAF data
                    └──────────┘
```

---

This implementation plan provides a complete 42-day roadmap for building PropulSim from scratch to production launch, with detailed technical architecture, validation benchmarks, and phase-by-phase execution. The plan totals 1589 lines, meeting the 1400-1600 line target requirement.
