# 77. BlastCalc — Explosion and Blast Effect Simulation Platform

## Implementation Plan

**MVP Scope:** Browser-based blast scenario designer with 3D visualization (Three.js/Deck.gl) for detonation point placement, target structures, and terrain features, empirical blast load calculator implementing Kingery-Bulmash equations and UFC 3-340-02 methodologies compiled to WebAssembly for instant free-field and reflected overpressure/impulse calculations at distances up to 1000m for charge sizes 0.1kg to 50,000kg TNT-equivalent, TNT equivalence library covering 20+ explosive types (C4, PETN, ANFO, RDX, HMX, dynamite, emulsion explosives), Friedlander waveform generator for time-history blast loading with positive and negative phase for structural analysis input, SDOF (single-degree-of-freedom) structural response solver for reinforced concrete walls, steel columns/beams, and glazing panels with pressure-impulse diagram generation and damage assessment per UFC 3-340-02 and ASCE 59-11 criteria (superficial/moderate/heavy/hazardous/blowout), primary fragment trajectory calculator using Gurney equations for initial velocity and ballistic integration with aerodynamic drag plus empirical penetration models (Thor, JTCG/ME, NDRC), standoff distance calculator per FEMA 426 and GSA blast protection standards, threat library covering VBIED (100kg to 2000kg) and PBIED (5kg to 50kg) per military/security standards, PDF report generation with blast load summary, structural response, damage level predictions, and compliance assessment, Stripe billing with three tiers (Free / Pro $199/mo / Advanced $499/user/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| Empirical Blast Engine | Rust (native + WASM) | Kingery-Bulmash polynomials, UFC 3-340-02 calculations, Friedlander waveform generation |
| SDOF Solver | Rust (native + WASM) | Single-degree-of-freedom structural response, P-I diagram generation |
| Fragment Tracker | Rust (native) | Gurney equations, ballistic trajectory integration, penetration models |
| WASM Compilation | `wasm-pack` + `wasm-bindgen` | Client-side blast calculations for instant feedback |
| Report Generation | Python 3.12 (FastAPI) | PDF generation with matplotlib plots, compliance checking, GIS integration |
| Database | PostgreSQL 16 | Users, projects, scenarios, explosive library, material properties, threat library |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | Simulation results, generated reports, scenario snapshots, CAD/GIS imports |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| 3D Visualization | Three.js + Deck.gl | Blast pressure contours, fragment trajectories, structural damage visualization, GIS terrain |
| Real-time | WebSocket (Axum) | Live calculation progress for large parameter sweeps |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | WASM bundle delivery, static frontend assets, report downloads |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, error tracking |
| CI/CD | GitHub Actions | WASM build pipeline, Rust tests, Docker image push |

### Key Architecture Decisions

1. **Client-side WASM for empirical blast calculations with instant results**: All Kingery-Bulmash, UFC 3-340-02 reflected pressure, Friedlander waveform, and SDOF structural response calculations run in the browser via WASM, providing instant feedback for parametric studies with zero server cost. This covers 95%+ of design-level protective structure assessment workflows. Server-side Python service handles only PDF report generation and advanced CFD simulations (post-MVP). WASM bundle size <1.5MB gzipped, loads in <500ms on typical connections.

2. **Unified blast solver in Rust rather than wrapping legacy Fortran tools**: Building a custom empirical blast engine in Rust gives us control over WASM compilation, parametric sweep performance, and integration with structural response. Legacy tools like CONWEP (Fortran, command-line, restricted distribution) and BlastX (Fortran, empirical tables) cannot compile to WASM and lack the flexibility for modern web integration. Our Rust implementation uses polynomial fits identical to Kingery-Bulmash but with proper validation against UFC 3-340-02 test data and AUTODYN benchmarks.

3. **Three.js for 3D scenario visualization with Deck.gl for GIS layers**: Three.js provides full 3D rendering of detonation points, structures, blast pressure contours (isosurfaces), and fragment trajectories. Deck.gl overlays add GIS terrain, satellite imagery, and site plans for real-world protective design scenarios. This combination allows intuitive spatial scenario setup (drag-and-drop threat placement, standoff measurement) while supporting data-driven visualizations (pressure heatmaps, fragment density maps). Canvas/SVG alternatives rejected because blast visualization requires true 3D perspective and GPU-accelerated rendering for real-time interaction with 100K+ fragment trajectories.

4. **Separate SDOF solver for rapid structural response rather than full FEA in MVP**: SDOF methodology per UFC 3-340-02 captures blast-loaded structural component behavior (walls, beams, columns, glazing) with 1000x faster computation than full finite element analysis while maintaining accuracy within 10% for design-level assessment. This enables real-time parametric sweeps (standoff distance, charge size, component properties) in the browser. Full 3D Lagrangian FEA with damage models deferred to post-MVP Advanced tier for users requiring breach prediction, progressive collapse, and coupled fluid-structure interaction.

5. **PostgreSQL for explosive properties and material database with S3 for results**: Explosive library (TNT equivalence factors, JWL parameters, fragment characteristics for 100+ explosives), structural material properties (concrete, steel, masonry, glazing with strain-rate dependencies), and threat scenarios (VBIED/PBIED definitions per FEMA 426) stored in PostgreSQL with indexed parametric search. Generated PDF reports, scenario snapshots, and imported CAD/GIS files stored in S3. This allows the material database to scale to 1000+ materials and 500+ explosive types while enabling fast search and compliance checking.

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";  -- For fuzzy text search on explosives/materials

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

-- Organizations (for Advanced plan collaboration)
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

-- Projects (blast scenario workspace)
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    scenario_data JSONB NOT NULL DEFAULT '{}',  -- Detonation points, structures, terrain, receivers
    settings JSONB DEFAULT '{}',  -- Units (metric/imperial), coordinate system, visualization preferences
    is_public BOOLEAN NOT NULL DEFAULT false,
    thumbnail_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);
CREATE INDEX projects_org_idx ON projects(org_id);
CREATE INDEX projects_updated_idx ON projects(updated_at DESC);

-- Blast Scenarios (calculation runs within a project)
CREATE TABLE scenarios (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    explosive_id UUID NOT NULL,  -- References explosives table
    charge_mass REAL NOT NULL,  -- kg TNT-equivalent
    detonation_point JSONB NOT NULL,  -- {x, y, z, burst_type: "surface" | "air" | "buried"}
    targets JSONB NOT NULL DEFAULT '[]',  -- [{id, type, position, properties}]
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | running | completed | failed
    results_url TEXT,  -- S3 URL for detailed results
    results_summary JSONB,  -- Quick-access summary (max pressure, impulse, damage levels)
    error_message TEXT,
    computation_mode TEXT NOT NULL DEFAULT 'wasm',  -- wasm | server
    completed_at TIMESTAMPTZ,
    wall_time_ms INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX scenarios_project_idx ON scenarios(project_id);
CREATE INDEX scenarios_user_idx ON scenarios(user_id);
CREATE INDEX scenarios_status_idx ON scenarios(status);
CREATE INDEX scenarios_created_idx ON scenarios(created_at DESC);

-- Explosives Library
CREATE TABLE explosives (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,  -- e.g., "TNT", "C4", "ANFO"
    category TEXT NOT NULL,  -- military | commercial | improvised
    tnt_equivalence REAL NOT NULL,  -- Relative to TNT = 1.0
    density REAL NOT NULL,  -- g/cm³
    detonation_velocity REAL,  -- m/s
    jwl_a REAL,  -- JWL equation of state parameters (for CFD)
    jwl_b REAL,
    jwl_r1 REAL,
    jwl_r2 REAL,
    jwl_omega REAL,
    gurney_constant REAL,  -- √(2E) for fragment velocity calculations
    description TEXT,
    is_builtin BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX explosives_category_idx ON explosives(category);
CREATE INDEX explosives_name_trgm_idx ON explosives USING gin(name gin_trgm_ops);

-- Structural Materials Library
CREATE TABLE materials (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,  -- e.g., "Concrete 4000psi", "Steel ASTM A36"
    category TEXT NOT NULL,  -- concrete | steel | masonry | glazing | wood | composite
    density REAL NOT NULL,  -- kg/m³
    elastic_modulus REAL,  -- Pa
    yield_strength REAL,  -- Pa (for steel)
    compressive_strength REAL,  -- Pa (for concrete)
    tensile_strength REAL,  -- Pa
    dif_compression REAL,  -- Dynamic increase factor for compression
    dif_tension REAL,  -- Dynamic increase factor for tension
    strain_rate_params JSONB,  -- {C, p} for Cowper-Symonds model
    is_builtin BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX materials_category_idx ON materials(category);
CREATE INDEX materials_name_trgm_idx ON materials USING gin(name gin_trgm_ops);

-- Threat Library (VBIED, PBIED definitions)
CREATE TABLE threats (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,  -- e.g., "Small VBIED", "Large PBIED"
    threat_type TEXT NOT NULL,  -- vbied | pbied | military_munition
    charge_mass_min REAL NOT NULL,  -- kg TNT-equivalent
    charge_mass_max REAL NOT NULL,
    explosive_id UUID REFERENCES explosives(id),
    standoff_min REAL,  -- Minimum expected standoff distance (m)
    standoff_max REAL,
    description TEXT,
    reference_standard TEXT,  -- e.g., "FEMA 426", "GSA 2019", "UFC 4-010-01"
    is_builtin BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX threats_type_idx ON threats(threat_type);

-- Structural Components (within a scenario for SDOF analysis)
CREATE TABLE components (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    scenario_id UUID NOT NULL REFERENCES scenarios(id) ON DELETE CASCADE,
    component_type TEXT NOT NULL,  -- wall | column | beam | slab | glazing
    material_id UUID NOT NULL REFERENCES materials(id),
    geometry JSONB NOT NULL,  -- {thickness, width, height, reinforcement_ratio, etc.}
    support_conditions TEXT NOT NULL,  -- fixed | pinned | cantilever | two_way
    sdof_params JSONB,  -- {mass, stiffness, resistance_function, load_factor, transformation_factor}
    blast_load JSONB,  -- {peak_pressure, impulse, duration, waveform_data}
    response_results JSONB,  -- {max_displacement, max_stress, damage_level, ductility_ratio}
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX components_scenario_idx ON components(scenario_id);
CREATE INDEX components_type_idx ON components(component_type);

-- Generated Reports
CREATE TABLE reports (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    scenario_id UUID NOT NULL REFERENCES scenarios(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    report_type TEXT NOT NULL,  -- blast_summary | structural_assessment | compliance_check | vulnerability_assessment
    report_url TEXT NOT NULL,  -- S3 URL to PDF
    format TEXT NOT NULL DEFAULT 'pdf',
    parameters JSONB DEFAULT '{}',  -- Report generation options
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX reports_scenario_idx ON reports(scenario_id);
CREATE INDEX reports_user_idx ON reports(user_id);
CREATE INDEX reports_created_idx ON reports(created_at DESC);

-- Usage Tracking (for billing enforcement)
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,  -- scenario_calculation | report_generation | api_call
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
    pub scenario_data: serde_json::Value,
    pub settings: serde_json::Value,
    pub is_public: bool,
    pub thumbnail_url: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Scenario {
    pub id: Uuid,
    pub project_id: Uuid,
    pub user_id: Uuid,
    pub name: String,
    pub explosive_id: Uuid,
    pub charge_mass: f32,
    pub detonation_point: serde_json::Value,
    pub targets: serde_json::Value,
    pub status: String,
    pub results_url: Option<String>,
    pub results_summary: Option<serde_json::Value>,
    pub error_message: Option<String>,
    pub computation_mode: String,
    pub completed_at: Option<DateTime<Utc>>,
    pub wall_time_ms: Option<i32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Explosive {
    pub id: Uuid,
    pub name: String,
    pub category: String,
    pub tnt_equivalence: f32,
    pub density: f32,
    pub detonation_velocity: Option<f32>,
    pub jwl_a: Option<f32>,
    pub jwl_b: Option<f32>,
    pub jwl_r1: Option<f32>,
    pub jwl_r2: Option<f32>,
    pub jwl_omega: Option<f32>,
    pub gurney_constant: Option<f32>,
    pub description: Option<String>,
    pub is_builtin: bool,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Material {
    pub id: Uuid,
    pub name: String,
    pub category: String,
    pub density: f32,
    pub elastic_modulus: Option<f32>,
    pub yield_strength: Option<f32>,
    pub compressive_strength: Option<f32>,
    pub tensile_strength: Option<f32>,
    pub dif_compression: Option<f32>,
    pub dif_tension: Option<f32>,
    pub strain_rate_params: Option<serde_json::Value>,
    pub is_builtin: bool,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Threat {
    pub id: Uuid,
    pub name: String,
    pub threat_type: String,
    pub charge_mass_min: f32,
    pub charge_mass_max: f32,
    pub explosive_id: Option<Uuid>,
    pub standoff_min: Option<f32>,
    pub standoff_max: Option<f32>,
    pub description: Option<String>,
    pub reference_standard: Option<String>,
    pub is_builtin: bool,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Component {
    pub id: Uuid,
    pub scenario_id: Uuid,
    pub component_type: String,
    pub material_id: Uuid,
    pub geometry: serde_json::Value,
    pub support_conditions: String,
    pub sdof_params: Option<serde_json::Value>,
    pub blast_load: Option<serde_json::Value>,
    pub response_results: Option<serde_json::Value>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct BlastCalculationParams {
    pub explosive_id: Uuid,
    pub charge_mass: f64,  // kg TNT-equivalent
    pub burst_type: BurstType,
    pub detonation_point: Point3D,
    pub receivers: Vec<ReceiverPoint>,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum BurstType {
    Surface,
    Air,
    Buried,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct Point3D {
    pub x: f64,  // meters
    pub y: f64,
    pub z: f64,  // height above ground
}

#[derive(Debug, Deserialize, Serialize)]
pub struct ReceiverPoint {
    pub id: String,
    pub position: Point3D,
    pub receiver_type: String,  // structure | person | equipment
    pub normal_vector: Option<Point3D>,  // For reflected pressure calculation
}

#[derive(Debug, Deserialize, Serialize)]
pub struct SdofParams {
    pub component_type: ComponentType,
    pub material_id: Uuid,
    pub width: f64,  // m
    pub height: f64,  // m
    pub thickness: f64,  // m
    pub support_conditions: SupportConditions,
    pub reinforcement_ratio: Option<f64>,  // For RC elements
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum ComponentType {
    RcWall,
    RcSlab,
    SteelBeam,
    SteelColumn,
    Glazing,
    MasonryWall,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum SupportConditions {
    Fixed,
    Pinned,
    Cantilever,
    TwoWay,
}
```

---

## Solver Architecture Deep-Dive

### Governing Equations and Discretization

BlastCalc's empirical blast engine implements the **Kingery-Bulmash equations**, the standard for blast load prediction in protective design per UFC 3-340-02 and ASCE 59-11. These equations are polynomial fits to experimental data from thousands of TNT detonation tests conducted by the US Army.

#### Free-Air Blast Pressure (Spherical Burst)

For a spherical burst in free air, the peak incident overpressure *P<sub>so</sub>* and scaled distance *Z* relationship:

```
Z = R / W^(1/3)

where:
  R = standoff distance (m)
  W = charge mass (kg TNT-equivalent)
  Z = scaled distance (m/kg^(1/3))

Kingery-Bulmash polynomial (piecewise for different Z ranges):

For 0.05 ≤ Z < 1.0:
  log10(P_so / P_atm) = C0 + C1*U + C2*U² + C3*U³ + C4*U⁴
  where U = log10(Z)

P_so = peak incident overpressure (Pa)
i_so = positive phase specific impulse (Pa·s)
t_a = shock arrival time (s)
t_d = positive phase duration (s)
P_so⁻ = negative phase peak pressure (Pa)
i_so⁻ = negative phase specific impulse (Pa·s)
```

Each parameter (pressure, impulse, duration, arrival time) has unique polynomial coefficients covering scaled distances from Z = 0.05 (near-field) to Z = 40 (far-field).

#### Surface Burst (Hemispherical)

For a surface burst, the charge is reflected by the ground, effectively doubling the energy in the hemisphere:

```
W_equiv = 2 * W  (for hemispherical burst)

Then apply free-air equations with W_equiv
```

#### Reflected Pressure (Normal Reflection)

When a shock wave strikes a rigid surface at normal incidence, the reflected overpressure *P<sub>r</sub>* is:

```
For P_so / P_atm < 10:
  P_r = 2 * P_so * (7 * P_atm + 4 * P_so) / (7 * P_atm + P_so)

For P_so / P_atm ≥ 10:
  P_r = 8 * P_so  (high-pressure limit)
```

For oblique incidence at angle *θ* from normal:

```
P_r(θ) = P_r(0°) * cos²(θ) + P_so

where P_r(0°) is normal reflected pressure
```

#### Friedlander Waveform (Time-History)

The time-varying blast pressure is approximated by the Friedlander equation:

```
P(t) = P_so * (1 - t/t_d) * exp(-b * t/t_d)   for 0 ≤ t ≤ t_d (positive phase)
P(t) = P_so⁻ * (1 - (t-t_d)/t_d⁻) * exp(-b * (t-t_d)/t_d⁻)   for t_d < t ≤ t_d + t_d⁻ (negative phase)

where:
  b = decay coefficient (typically 2.0-3.0, fitted to match impulse)
  t_d = positive phase duration
  t_d⁻ = negative phase duration
  P_so⁻ = negative phase peak (typically -0.3 * P_so)
```

The positive phase impulse constraint:

```
i_so = ∫[0 to t_d] P(t) dt

Solving for b:
  b = t_d * P_so / i_so - 1
```

### SDOF Structural Response

UFC 3-340-02 methodology transforms a blast-loaded structural component into an equivalent single-degree-of-freedom (SDOF) system:

```
M * ÿ + C * ẏ + R(y) = F(t) * KL

where:
  M = equivalent mass = mass_factor * ρ * A * thickness
  y = mid-span displacement
  R(y) = resistance function (material-dependent, includes plastic hinges)
  F(t) = applied blast load (Friedlander waveform)
  KL = load transformation factor (varies with support conditions and load distribution)

mass_factor:
  - Uniformly loaded simply supported beam: 0.50
  - Uniformly loaded fixed-fixed beam: 0.41
  - Uniformly loaded two-way slab (fixed): 0.33
```

#### Resistance Function for RC Wall (Flexural Response)

For a reinforced concrete wall with fixed supports:

```
R(y) = K_elastic * y                    for y ≤ y_elastic
R(y) = R_ultimate                       for y_elastic < y ≤ y_ultimate
R(y) = R_ultimate * (1 - (y - y_ultimate)/(y_failure - y_ultimate))   for y > y_ultimate

where:
  K_elastic = stiffness = 384 * E * I / (5 * L⁴) * KL_M  (for uniformly loaded fixed-fixed)
  R_ultimate = ultimate flexural capacity = 8 * M_n / L² * KL_M
  M_n = nominal moment capacity = A_s * f_y * (d - a/2)
  y_ultimate = ultimate displacement (plastic hinge formation)
  y_failure = failure displacement (tensile rebar rupture or concrete crushing)
```

Dynamic increase factors (DIF) for strain-rate effects:

```
f'_c,dynamic = f'_c,static * DIF_c
DIF_c = (ε̇_c / ε̇_c,static)^1.026α_s   for compression

f_y,dynamic = f_y,static * DIF_s
DIF_s = (f_y / 414)^(-0.2) * (ε̇_s / 10⁻⁴)^0.02   for steel yield
```

#### Numerical Integration (Newmark-β Method)

The SDOF equation of motion is integrated using Newmark-β:

```
At timestep n+1:

y_{n+1} = y_n + Δt * ẏ_n + (Δt²/2) * [(1 - 2β) * ÿ_n + 2β * ÿ_{n+1}]
ẏ_{n+1} = ẏ_n + Δt * [(1 - γ) * ÿ_n + γ * ÿ_{n+1}]

For unconditional stability: β = 0.25, γ = 0.5 (average acceleration method)

Solve for ÿ_{n+1}:
  ÿ_{n+1} = (F_{n+1} * KL - R(y_{n+1}) - C * ẏ_{n+1}) / M

Iterative solution (Newton-Raphson) at each timestep for nonlinear R(y).
```

#### Pressure-Impulse (P-I) Diagram

A P-I diagram defines damage boundaries in pressure-impulse space:

```
Iso-damage curve for damage level D:

(P/P_0)^a + (i/i_0)^b = 1

where:
  P_0 = asymptotic pressure (pure impulsive regime, i → ∞)
  i_0 = asymptotic impulse (pure quasi-static regime, P → ∞)
  a, b = exponents (typically 1.0 to 2.0 depending on material and failure mode)

For each damage level (superficial, moderate, heavy, hazardous, blowout):
  - Run SDOF analysis for range of (P, i) combinations
  - Identify displacement thresholds: y_superficial, y_moderate, y_heavy, y_hazardous, y_blowout
  - Fit iso-damage curves to (P, i) pairs that produce each threshold displacement
```

### Fragment Trajectory and Penetration

#### Gurney Equation for Initial Fragment Velocity

For a cylindrical casing with explosive fill:

```
V_0 = √(2E) * √(C/M) / (1 + C/(2M))

where:
  V_0 = initial fragment velocity (m/s)
  √(2E) = Gurney constant for explosive (m/s): TNT = 2440, C4 = 2700, HMX = 2970
  C = charge mass (kg)
  M = casing mass (kg)
```

For a spherical casing:

```
V_0 = √(2E) * √(C/M) / (1 + C/(1.6M))
```

#### Ballistic Trajectory with Drag

Fragment trajectory in air with aerodynamic drag:

```
m * dv/dt = -0.5 * ρ_air * Cd * A * v²

where:
  m = fragment mass (kg)
  ρ_air = air density (1.225 kg/m³ at sea level)
  Cd = drag coefficient (depends on shape):
       - Sphere: 0.47
       - Cube tumbling: 1.05
       - Thin plate: 1.17
       - Streamlined: 0.04-0.10
  A = projected area (m²)
  v = fragment velocity vector (m/s)

Numerical integration (RK4):
  dx/dt = v_x,  dv_x/dt = -D * v_x * |v|
  dy/dt = v_y,  dv_y/dt = -D * v_y * |v|
  dz/dt = v_z,  dv_z/dt = -D * v_z * |v| - g

where D = (0.5 * ρ_air * Cd * A) / m
```

#### Penetration Models

**Thor Equation (steel penetration):**

```
t = 3.84e-4 * √(M * V_i * sec(θ)) - 0.25 * d

where:
  t = penetration depth (inches)
  M = fragment mass (grains, 1 grain = 0.0648 g)
  V_i = impact velocity (ft/s)
  θ = impact obliquity angle (degrees from normal)
  d = fragment diameter (inches)

Perforation condition: t ≥ t_plate
```

**NDRC Equation (concrete penetration):**

```
x/d = K * N * (V_i/1000)^1.8

where:
  x = penetration depth (inches)
  d = projectile diameter (inches)
  K = nose shape factor (0.72 flat, 0.84 blunt, 1.0 ogival, 1.14 very sharp)
  N = (M/d³)^0.215  (projectile mass scaling)
  M = projectile mass (lb)
  V_i = impact velocity (ft/s)

Perforation velocity:
  V_perf = 1000 * (x_p / (K * N))^(1/1.8)
  where x_p = plate thickness

Residual velocity after perforation:
  V_res = √(V_i² - V_perf²)
```

### Client-Side WASM Execution

All empirical calculations run in-browser via WASM:

```
User inputs (charge mass, standoff, explosive type, targets)
    ↓
WASM Blast Solver
    ├── Kingery-Bulmash overpressure/impulse for each receiver point (50-100 points: 0.5ms)
    ├── Friedlander waveform generation for structural analysis (1000 timesteps: 2ms)
    ├── SDOF structural response integration (Newmark-β, 1000 timesteps, 5 components: 15ms)
    ├── Gurney fragment velocity calculation (10-1000 fragments: 1ms)
    ├── Ballistic trajectory integration (RK4, 100 timesteps per fragment, 100 fragments: 50ms)
    └── Penetration assessment (100 fragments × 10 targets: 5ms)
    ↓
Results displayed immediately in 3D visualization
Total computation time: <100ms for typical scenario
```

WASM bundle size target: <1.5MB gzipped (includes all polynomial coefficients, numerical solvers).

---

## Architecture Deep-Dives

### 1. Blast Scenario API Handler (Rust/Axum)

The primary endpoint receives a blast scenario configuration, validates explosive and target parameters, routes to WASM (client-side) or server (for large parameter sweeps), and stores results.

```rust
// src/api/handlers/scenario.rs

use axum::{
    extract::{Path, State, Json},
    http::StatusCode,
    response::IntoResponse,
};
use uuid::Uuid;
use crate::{
    db::models::{Scenario, BlastCalculationParams, Explosive, Material},
    blast::validation::validate_scenario,
    state::AppState,
    auth::Claims,
    error::ApiError,
};

#[derive(serde::Deserialize)]
pub struct CreateScenarioRequest {
    pub name: String,
    pub explosive_id: Uuid,
    pub charge_mass: f64,  // kg TNT-equivalent
    pub burst_type: String,
    pub detonation_point: serde_json::Value,
    pub targets: Vec<serde_json::Value>,
}

pub async fn create_scenario(
    State(state): State<AppState>,
    claims: Claims,
    Path(project_id): Path<Uuid>,
    Json(req): Json<CreateScenarioRequest>,
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

    // 2. Validate explosive and charge mass
    let explosive = sqlx::query_as!(
        Explosive,
        "SELECT * FROM explosives WHERE id = $1",
        req.explosive_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Explosive not found"))?;

    if req.charge_mass < 0.1 || req.charge_mass > 50000.0 {
        return Err(ApiError::Validation(
            "Charge mass must be between 0.1kg and 50,000kg TNT-equivalent"
        ));
    }

    // 3. Check plan limits
    let user = sqlx::query_as!(
        crate::db::models::User,
        "SELECT * FROM users WHERE id = $1",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    if user.plan == "free" && req.charge_mass > 1000.0 {
        return Err(ApiError::PlanLimit(
            "Free plan supports charge sizes up to 1000kg. Upgrade to Pro for larger scenarios."
        ));
    }

    // 4. Determine computation mode (WASM always for single scenario, server for sweeps)
    let computation_mode = "wasm";  // Single-point calculations always client-side

    // 5. Create scenario record
    let scenario = sqlx::query_as!(
        Scenario,
        r#"INSERT INTO scenarios
            (project_id, user_id, name, explosive_id, charge_mass,
             detonation_point, targets, status, computation_mode)
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9)
        RETURNING *"#,
        project_id,
        claims.user_id,
        req.name,
        req.explosive_id,
        req.charge_mass as f32,
        serde_json::json!({
            "x": 0.0, "y": 0.0, "z": 0.0,
            "burst_type": req.burst_type
        }),
        serde_json::to_value(&req.targets)?,
        "pending",
        computation_mode,
    )
    .fetch_one(&state.db)
    .await?;

    // 6. Return scenario (client will compute via WASM)
    Ok((StatusCode::CREATED, Json(scenario)))
}

pub async fn get_scenario(
    State(state): State<AppState>,
    claims: Claims,
    Path((project_id, scenario_id)): Path<(Uuid, Uuid)>,
) -> Result<Json<Scenario>, ApiError> {
    let scenario = sqlx::query_as!(
        Scenario,
        "SELECT s.* FROM scenarios s
         JOIN projects p ON s.project_id = p.id
         WHERE s.id = $1 AND s.project_id = $2
         AND (p.owner_id = $3 OR p.org_id IN (
             SELECT org_id FROM org_members WHERE user_id = $3
         ))",
        scenario_id,
        project_id,
        claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Scenario not found"))?;

    Ok(Json(scenario))
}

pub async fn update_scenario_results(
    State(state): State<AppState>,
    claims: Claims,
    Path((project_id, scenario_id)): Path<(Uuid, Uuid)>,
    Json(results): Json<serde_json::Value>,
) -> Result<impl IntoResponse, ApiError> {
    // Called by client after WASM computation completes
    // Stores results summary in database for quick access

    let updated = sqlx::query!(
        "UPDATE scenarios
         SET status = 'completed',
             results_summary = $1,
             completed_at = NOW(),
             wall_time_ms = EXTRACT(EPOCH FROM (NOW() - created_at))::int * 1000
         WHERE id = $2 AND project_id = $3
         AND user_id = $4
         RETURNING id",
        results,
        scenario_id,
        project_id,
        claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Scenario not found or unauthorized"))?;

    Ok(StatusCode::OK)
}
```

### 2. Blast Solver Core (Rust — compiled to WASM)

The core blast load calculator implementing Kingery-Bulmash equations, Friedlander waveform generation, and SDOF structural response.

```rust
// blast-solver/src/kingery_bulmash.rs

use std::f64::consts::PI;

/// Kingery-Bulmash polynomial coefficients for peak incident overpressure
/// Source: UFC 3-340-02, Appendix C
const PSO_COEFFICIENTS: &[(f64, f64, [f64; 5])] = &[
    // (Z_min, Z_max, [C0, C1, C2, C3, C4])
    (0.05, 0.3, [1.965, 0.021, -0.132, -0.034, 0.0]),
    (0.3, 1.0, [1.886, -0.444, -0.538, 0.379, -0.153]),
    (1.0, 2.0, [1.662, -1.218, 0.176, -0.106, 0.0]),
    (2.0, 10.0, [1.127, -1.828, 0.290, 0.0, 0.0]),
    (10.0, 40.0, [-0.325, -2.383, 0.0, 0.0, 0.0]),
];

/// Kingery-Bulmash coefficients for positive phase impulse
const ISO_COEFFICIENTS: &[(f64, f64, [f64; 5])] = &[
    (0.05, 0.3, [-0.614, 0.326, 0.155, -0.048, 0.0]),
    (0.3, 1.0, [-0.540, 0.523, -0.353, 0.085, 0.0]),
    (1.0, 2.0, [-0.318, 0.355, -0.344, 0.0, 0.0]),
    (2.0, 10.0, [-0.220, 0.305, -0.106, 0.0, 0.0]),
    (10.0, 40.0, [-0.762, 0.154, 0.0, 0.0, 0.0]),
];

/// Kingery-Bulmash coefficients for positive phase duration
const TD_COEFFICIENTS: &[(f64, f64, [f64; 5])] = &[
    (0.05, 0.3, [-0.613, 0.825, 0.0, 0.0, 0.0]),
    (0.3, 1.0, [-0.384, 0.825, -0.197, 0.0, 0.0]),
    (1.0, 2.0, [-0.264, 0.778, -0.108, 0.0, 0.0]),
    (2.0, 10.0, [-0.237, 0.772, -0.042, 0.0, 0.0]),
    (10.0, 40.0, [-0.354, 0.719, 0.0, 0.0, 0.0]),
];

/// Kingery-Bulmash coefficients for shock arrival time
const TA_COEFFICIENTS: &[(f64, f64, [f64; 5])] = &[
    (0.05, 0.3, [-0.580, 1.122, 0.0, 0.0, 0.0]),
    (0.3, 1.0, [-0.389, 1.122, -0.080, 0.0, 0.0]),
    (1.0, 2.0, [-0.264, 1.086, -0.051, 0.0, 0.0]),
    (2.0, 10.0, [-0.221, 1.073, -0.017, 0.0, 0.0]),
    (10.0, 40.0, [-0.295, 1.048, 0.0, 0.0, 0.0]),
];

pub struct BlastParameters {
    pub peak_overpressure: f64,    // Pa
    pub impulse: f64,              // Pa·s
    pub arrival_time: f64,         // s
    pub duration: f64,             // s
    pub dynamic_pressure: f64,     // Pa
    pub shock_velocity: f64,       // m/s
}

pub fn calculate_free_air_blast(
    charge_mass: f64,  // kg TNT-equivalent
    standoff: f64,     // m
    burst_type: BurstType,
) -> BlastParameters {
    // Apply burst type scaling
    let w_equiv = match burst_type {
        BurstType::Surface => charge_mass * 2.0,  // Hemispherical reflection
        BurstType::Air => charge_mass,
        BurstType::Buried => charge_mass * 0.5,  // Energy absorption by soil
    };

    // Scaled distance
    let z = standoff / w_equiv.powf(1.0 / 3.0);

    // Ambient conditions
    const P_ATM: f64 = 101325.0;  // Pa
    const C_SOUND: f64 = 340.0;   // m/s

    // Evaluate Kingery-Bulmash polynomials
    let log_z = z.log10();

    let pso_normalized = eval_kb_polynomial(log_z, PSO_COEFFICIENTS);
    let peak_overpressure = P_ATM * 10_f64.powf(pso_normalized);

    let iso_normalized = eval_kb_polynomial(log_z, ISO_COEFFICIENTS);
    let impulse = (P_ATM * w_equiv.powf(1.0 / 3.0) / C_SOUND) * 10_f64.powf(iso_normalized);

    let td_normalized = eval_kb_polynomial(log_z, TD_COEFFICIENTS);
    let duration = (w_equiv.powf(1.0 / 3.0) / C_SOUND) * 10_f64.powf(td_normalized);

    let ta_normalized = eval_kb_polynomial(log_z, TA_COEFFICIENTS);
    let arrival_time = (w_equiv.powf(1.0 / 3.0) / C_SOUND) * 10_f64.powf(ta_normalized);

    // Dynamic pressure (requires shock velocity)
    let mach = 1.0 + (peak_overpressure / P_ATM) * 6.0 / 7.0;
    let shock_velocity = C_SOUND * mach;
    let dynamic_pressure = 5.0 * peak_overpressure.powi(2) / (2.0 * P_ATM + peak_overpressure);

    BlastParameters {
        peak_overpressure,
        impulse,
        arrival_time,
        duration,
        dynamic_pressure,
        shock_velocity,
    }
}

fn eval_kb_polynomial(log_z: f64, coeffs: &[(f64, f64, [f64; 5])]) -> f64 {
    // Find appropriate coefficient set for scaled distance range
    for &(z_min, z_max, c) in coeffs {
        if log_z >= z_min.log10() && log_z < z_max.log10() {
            let u = log_z;
            return c[0] + c[1]*u + c[2]*u.powi(2) + c[3]*u.powi(3) + c[4]*u.powi(4);
        }
    }
    // Extrapolate if out of range
    let &(_, _, c) = coeffs.last().unwrap();
    let u = log_z;
    c[0] + c[1]*u
}

pub fn calculate_reflected_pressure(
    incident_pressure: f64,
    angle_of_incidence: f64,  // radians from normal
) -> f64 {
    const P_ATM: f64 = 101325.0;
    let pso_ratio = incident_pressure / P_ATM;

    // Normal reflected pressure (angle = 0)
    let pr_normal = if pso_ratio < 10.0 {
        2.0 * incident_pressure * (7.0 * P_ATM + 4.0 * incident_pressure)
            / (7.0 * P_ATM + incident_pressure)
    } else {
        8.0 * incident_pressure  // High-pressure limit
    };

    // Oblique reflection factor
    let cos_theta = angle_of_incidence.cos();
    pr_normal * cos_theta.powi(2) + incident_pressure
}

pub fn generate_friedlander_waveform(
    peak_pressure: f64,
    impulse: f64,
    duration: f64,
    num_points: usize,
) -> Vec<(f64, f64)> {
    // Solve for decay coefficient b to match impulse
    let b = duration * peak_pressure / impulse - 1.0;
    let b = b.max(0.5).min(5.0);  // Clamp to reasonable range

    let mut waveform = Vec::with_capacity(num_points);

    for i in 0..num_points {
        let t = (i as f64) / (num_points as f64) * duration * 2.0;

        let pressure = if t <= duration {
            // Positive phase
            peak_pressure * (1.0 - t/duration) * (-b * t/duration).exp()
        } else {
            // Negative phase (simplified: -30% of peak, same duration)
            let t_neg = t - duration;
            let p_neg = -0.3 * peak_pressure;
            let dur_neg = duration;
            p_neg * (1.0 - t_neg/dur_neg) * (-2.0 * t_neg/dur_neg).exp()
        };

        waveform.push((t, pressure));
    }

    waveform
}

#[derive(Debug, Clone, Copy)]
pub enum BurstType {
    Surface,
    Air,
    Buried,
}
```

### 3. SDOF Structural Response Solver (Rust — WASM)

Implements single-degree-of-freedom structural dynamics with Newmark-β integration for blast-loaded components.

```rust
// blast-solver/src/sdof.rs

use crate::kingery_bulmash::generate_friedlander_waveform;

pub struct SdofSystem {
    pub mass: f64,              // kg
    pub stiffness: f64,         // N/m
    pub damping_ratio: f64,     // dimensionless (typically 0.05-0.10)
    pub resistance_func: ResistanceFunction,
    pub load_factor: f64,       // KL (load transformation factor)
}

#[derive(Clone)]
pub struct ResistanceFunction {
    pub y_elastic: f64,         // Elastic limit displacement (m)
    pub r_elastic: f64,         // Elastic resistance (N)
    pub y_ultimate: f64,        // Ultimate displacement (m)
    pub r_ultimate: f64,        // Ultimate resistance (N)
    pub y_failure: f64,         // Failure displacement (m)
}

impl ResistanceFunction {
    pub fn eval(&self, y: f64) -> f64 {
        if y <= self.y_elastic {
            // Elastic regime
            (y / self.y_elastic) * self.r_elastic
        } else if y <= self.y_ultimate {
            // Plastic regime (constant resistance)
            self.r_ultimate
        } else if y <= self.y_failure {
            // Post-ultimate (degradation)
            let frac = (y - self.y_ultimate) / (self.y_failure - self.y_ultimate);
            self.r_ultimate * (1.0 - frac)
        } else {
            // Complete failure
            0.0
        }
    }
}

pub struct SdofResponse {
    pub time: Vec<f64>,
    pub displacement: Vec<f64>,
    pub velocity: Vec<f64>,
    pub acceleration: Vec<f64>,
    pub resistance: Vec<f64>,
    pub max_displacement: f64,
    pub max_velocity: f64,
    pub ductility_ratio: f64,
    pub damage_level: DamageLevel,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum DamageLevel {
    Superficial,    // < y_elastic
    Moderate,       // < 2 * y_elastic
    Heavy,          // < y_ultimate
    Hazardous,      // < 1.5 * y_ultimate
    Blowout,        // ≥ 1.5 * y_ultimate
}

pub fn solve_sdof_response(
    system: &SdofSystem,
    blast_load: &[(f64, f64)],  // (time, pressure) pairs
    dt: f64,
) -> SdofResponse {
    let n = blast_load.len();

    let mut time = vec![0.0; n];
    let mut y = vec![0.0; n];
    let mut v = vec![0.0; n];
    let mut a = vec![0.0; n];
    let mut r = vec![0.0; n];

    // Damping coefficient
    let omega_n = (system.stiffness / system.mass).sqrt();
    let c = 2.0 * system.damping_ratio * omega_n * system.mass;

    // Newmark-β parameters (average acceleration method)
    let beta = 0.25;
    let gamma = 0.5;

    // Initial conditions
    y[0] = 0.0;
    v[0] = 0.0;
    r[0] = system.resistance_func.eval(y[0]);
    let f0 = blast_load[0].1 * system.load_factor;
    a[0] = (f0 - r[0] - c * v[0]) / system.mass;
    time[0] = blast_load[0].0;

    // Time integration
    for i in 1..n {
        time[i] = blast_load[i].0;
        let dt = time[i] - time[i-1];

        // Predictor
        let y_pred = y[i-1] + dt * v[i-1] + (dt*dt / 2.0) * (1.0 - 2.0*beta) * a[i-1];
        let v_pred = v[i-1] + dt * (1.0 - gamma) * a[i-1];

        // Corrector (Newton-Raphson for nonlinear resistance)
        let f_ext = blast_load[i].1 * system.load_factor;
        let mut y_curr = y_pred;
        let mut v_curr = v_pred;

        for _iter in 0..10 {  // Max 10 N-R iterations
            let r_curr = system.resistance_func.eval(y_curr);
            let residual = system.mass * a[i-1] + c * v_curr + r_curr - f_ext;

            if residual.abs() < 1e-6 {
                break;
            }

            // Tangent stiffness (numerical derivative)
            let dy = 1e-8;
            let r_plus = system.resistance_func.eval(y_curr + dy);
            let k_tangent = (r_plus - r_curr) / dy;

            let jacobian = c * gamma / (beta * dt) + k_tangent;
            let dy_corr = -residual / jacobian;

            y_curr += dy_corr;
            v_curr = v_pred + (gamma / (beta * dt)) * dy_corr;
        }

        y[i] = y_curr;
        v[i] = v_curr;
        r[i] = system.resistance_func.eval(y[i]);
        a[i] = (f_ext - r[i] - c * v[i]) / system.mass;
    }

    // Post-processing
    let max_displacement = y.iter().cloned().fold(0.0, f64::max);
    let max_velocity = v.iter().map(|v| v.abs()).fold(0.0, f64::max);
    let ductility_ratio = max_displacement / system.resistance_func.y_elastic;

    let damage_level = classify_damage(max_displacement, &system.resistance_func);

    SdofResponse {
        time,
        displacement: y,
        velocity: v,
        acceleration: a,
        resistance: r,
        max_displacement,
        max_velocity,
        ductility_ratio,
        damage_level,
    }
}

fn classify_damage(max_disp: f64, rf: &ResistanceFunction) -> DamageLevel {
    if max_disp < rf.y_elastic {
        DamageLevel::Superficial
    } else if max_disp < 2.0 * rf.y_elastic {
        DamageLevel::Moderate
    } else if max_disp < rf.y_ultimate {
        DamageLevel::Heavy
    } else if max_disp < 1.5 * rf.y_ultimate {
        DamageLevel::Hazardous
    } else {
        DamageLevel::Blowout
    }
}

/// Build SDOF system parameters for a reinforced concrete wall
pub fn build_rc_wall_sdof(
    width: f64,        // m
    height: f64,       // m
    thickness: f64,    // m
    f_c: f64,          // Concrete compressive strength (Pa)
    f_y: f64,          // Steel yield strength (Pa)
    rho: f64,          // Reinforcement ratio
) -> SdofSystem {
    const CONCRETE_DENSITY: f64 = 2400.0;  // kg/m³
    const E_CONCRETE: f64 = 25e9;  // Pa (typical for 4000psi concrete)

    // Equivalent mass (fixed-fixed boundary, mass factor = 0.41)
    let mass = 0.41 * CONCRETE_DENSITY * width * height * thickness;

    // Elastic stiffness (fixed-fixed uniformly loaded plate)
    let i_moment = width * thickness.powi(3) / 12.0;  // Second moment of area
    let stiffness = 384.0 * E_CONCRETE * i_moment / (5.0 * height.powi(4));

    // Ultimate moment capacity (simplified rectangular section)
    let d = thickness * 0.9;  // Effective depth
    let a_s = rho * width * thickness;  // Steel area
    let a = a_s * f_y / (0.85 * f_c * width);  // Neutral axis depth
    let m_n = a_s * f_y * (d - a / 2.0);  // Nominal moment

    // Ultimate resistance (load transformation)
    let r_ultimate = 8.0 * m_n / height.powi(2);

    // Elastic limit displacement
    let y_elastic = r_ultimate / stiffness;

    // Failure displacement (assume 10x elastic for ductile RC)
    let y_failure = 10.0 * y_elastic;
    let y_ultimate = 3.0 * y_elastic;  // Plastic hinge formation

    let resistance_func = ResistanceFunction {
        y_elastic,
        r_elastic: r_ultimate,  // Bilinear: elastic = ultimate for simplification
        y_ultimate,
        r_ultimate,
        y_failure,
    };

    SdofSystem {
        mass,
        stiffness,
        damping_ratio: 0.05,  // 5% damping typical for RC
        resistance_func,
        load_factor: 0.64,  // KL for uniformly loaded fixed-fixed plate
    }
}
```

### 4. Fragment Trajectory Solver (Rust)

Calculates initial fragment velocities using Gurney equations and integrates ballistic trajectories with aerodynamic drag.

```rust
// blast-solver/src/fragments.rs

use std::f64::consts::PI;

pub struct Fragment {
    pub mass: f64,           // kg
    pub diameter: f64,       // m
    pub shape_factor: f64,   // Cd * A / m  (effective drag parameter)
    pub position: [f64; 3],  // m (x, y, z)
    pub velocity: [f64; 3],  // m/s
}

pub struct FragmentTrajectory {
    pub positions: Vec<[f64; 3]>,
    pub velocities: Vec<[f64; 3]>,
    pub times: Vec<f64>,
    pub impact_velocity: f64,
    pub range: f64,
}

pub fn calculate_initial_velocity_gurney(
    explosive_mass: f64,     // kg
    casing_mass: f64,        // kg
    sqrt_2e: f64,            // Gurney constant (m/s)
    geometry: CasingGeometry,
) -> f64 {
    let c_over_m = explosive_mass / casing_mass;

    match geometry {
        CasingGeometry::Cylindrical => {
            sqrt_2e * (c_over_m / (1.0 + c_over_m / 2.0)).sqrt()
        }
        CasingGeometry::Spherical => {
            sqrt_2e * (c_over_m / (1.0 + c_over_m / 1.6)).sqrt()
        }
    }
}

#[derive(Clone, Copy)]
pub enum CasingGeometry {
    Cylindrical,
    Spherical,
}

pub fn integrate_trajectory(
    fragment: &Fragment,
    initial_angle: f64,  // radians from horizontal
    initial_azimuth: f64,  // radians (0 = +x direction)
    dt: f64,             // timestep (s)
    max_time: f64,       // s
) -> FragmentTrajectory {
    const G: f64 = 9.81;  // m/s²
    const RHO_AIR: f64 = 1.225;  // kg/m³

    let mut pos = fragment.position;
    let v0 = (fragment.velocity[0].powi(2)
            + fragment.velocity[1].powi(2)
            + fragment.velocity[2].powi(2)).sqrt();

    let mut vel = [
        v0 * initial_angle.cos() * initial_azimuth.cos(),
        v0 * initial_angle.cos() * initial_azimuth.sin(),
        v0 * initial_angle.sin(),
    ];

    let mut positions = vec![pos];
    let mut velocities = vec![vel];
    let mut times = vec![0.0];

    let mut t = 0.0;

    // RK4 integration
    while t < max_time && pos[2] >= 0.0 {
        let (dv_dt, dp_dt) = compute_derivatives(&pos, &vel, fragment, RHO_AIR, G);

        // RK4 steps
        let k1_v = scale_vec(&dv_dt, dt);
        let k1_p = scale_vec(&dp_dt, dt);

        let v_mid1 = add_vec(&vel, &scale_vec(&k1_v, 0.5));
        let p_mid1 = add_vec(&pos, &scale_vec(&k1_p, 0.5));
        let (dv2, dp2) = compute_derivatives(&p_mid1, &v_mid1, fragment, RHO_AIR, G);
        let k2_v = scale_vec(&dv2, dt);
        let k2_p = scale_vec(&dp2, dt);

        let v_mid2 = add_vec(&vel, &scale_vec(&k2_v, 0.5));
        let p_mid2 = add_vec(&pos, &scale_vec(&k2_p, 0.5));
        let (dv3, dp3) = compute_derivatives(&p_mid2, &v_mid2, fragment, RHO_AIR, G);
        let k3_v = scale_vec(&dv3, dt);
        let k3_p = scale_vec(&dp3, dt);

        let v_end = add_vec(&vel, &k3_v);
        let p_end = add_vec(&pos, &k3_p);
        let (dv4, dp4) = compute_derivatives(&p_end, &v_end, fragment, RHO_AIR, G);
        let k4_v = scale_vec(&dv4, dt);
        let k4_p = scale_vec(&dp4, dt);

        // Update state
        vel = add_vec(&vel, &scale_vec(
            &add_vec(&k1_v, &add_vec(
                &scale_vec(&add_vec(&k2_v, &k3_v), 2.0),
                &k4_v
            )),
            1.0/6.0
        ));
        pos = add_vec(&pos, &scale_vec(
            &add_vec(&k1_p, &add_vec(
                &scale_vec(&add_vec(&k2_p, &k3_p), 2.0),
                &k4_p
            )),
            1.0/6.0
        ));

        t += dt;

        positions.push(pos);
        velocities.push(vel);
        times.push(t);
    }

    let impact_velocity = (vel[0].powi(2) + vel[1].powi(2) + vel[2].powi(2)).sqrt();
    let range = (pos[0].powi(2) + pos[1].powi(2)).sqrt();

    FragmentTrajectory {
        positions,
        velocities,
        times,
        impact_velocity,
        range,
    }
}

fn compute_derivatives(
    _pos: &[f64; 3],
    vel: &[f64; 3],
    fragment: &Fragment,
    rho_air: f64,
    g: f64,
) -> ([f64; 3], [f64; 3]) {
    let v_mag = (vel[0].powi(2) + vel[1].powi(2) + vel[2].powi(2)).sqrt();

    // Drag force = 0.5 * rho * Cd * A * v²
    let drag_coeff = 0.5 * rho_air * fragment.shape_factor * v_mag;

    let dv_dt = [
        -drag_coeff * vel[0],
        -drag_coeff * vel[1],
        -drag_coeff * vel[2] - g,
    ];

    let dp_dt = *vel;

    (dv_dt, dp_dt)
}

fn add_vec(a: &[f64; 3], b: &[f64; 3]) -> [f64; 3] {
    [a[0] + b[0], a[1] + b[1], a[2] + b[2]]
}

fn scale_vec(v: &[f64; 3], s: f64) -> [f64; 3] {
    [v[0] * s, v[1] * s, v[2] * s]
}

/// Thor equation for steel penetration
pub fn penetrate_steel_thor(
    fragment_mass: f64,      // kg
    fragment_diameter: f64,  // m
    impact_velocity: f64,    // m/s
    plate_thickness: f64,    // m
    obliquity: f64,          // radians from normal
) -> PenetrationResult {
    // Convert to Imperial units for Thor equation
    let m_grains = fragment_mass * 15432.36;  // kg to grains
    let d_inches = fragment_diameter * 39.3701;  // m to inches
    let v_fps = impact_velocity * 3.28084;  // m/s to ft/s
    let t_inches = plate_thickness * 39.3701;

    // Thor equation: t = 3.84e-4 * √(M * V * sec(θ)) - 0.25 * d
    let penetration_inches = 3.84e-4 * (m_grains * v_fps * obliquity.cos().recip()).sqrt()
                             - 0.25 * d_inches;
    let penetration_m = penetration_inches * 0.0254;

    if penetration_m >= plate_thickness {
        // Perforation
        let v_perf_fps = ((penetration_inches + 0.25 * d_inches) / 3.84e-4).powi(2)
                         / (m_grains * obliquity.cos().recip());
        let v_residual_fps = (v_fps.powi(2) - v_perf_fps).max(0.0).sqrt();
        let v_residual_ms = v_residual_fps / 3.28084;

        PenetrationResult {
            perforated: true,
            penetration_depth: plate_thickness,
            residual_velocity: v_residual_ms,
        }
    } else {
        PenetrationResult {
            perforated: false,
            penetration_depth: penetration_m.max(0.0),
            residual_velocity: 0.0,
        }
    }
}

pub struct PenetrationResult {
    pub perforated: bool,
    pub penetration_depth: f64,  // m
    pub residual_velocity: f64,  // m/s
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–4)

**Day 1: Project initialization and Rust backend scaffold**
```bash
cargo init blastcalc-api
cd blastcalc-api
cargo add axum tokio serde serde_json sqlx uuid chrono tower-http jsonwebtoken bcrypt
cargo add --features postgres,runtime-tokio-rustls,uuid,chrono,json sqlx
cargo add aws-sdk-s3
```
- `src/main.rs` — Axum app with CORS, tracing, graceful shutdown
- `src/config.rs` — Environment-based configuration (DATABASE_URL, JWT_SECRET, S3_BUCKET)
- `src/state.rs` — AppState struct (PgPool, S3Client)
- `src/error.rs` — ApiError enum with IntoResponse
- `Dockerfile` — Multi-stage build (builder + runtime)
- `docker-compose.yml` — PostgreSQL, MinIO (S3-compatible)

**Day 2: Database schema and migrations**
- `migrations/001_initial.sql` — All 10 tables: users, organizations, org_members, projects, scenarios, explosives, materials, threats, components, reports, usage_records
- `src/db/mod.rs` — Database pool initialization
- `src/db/models.rs` — All SQLx structs with FromRow derives
- Run `sqlx migrate run` to apply schema
- Seed script for explosives library (TNT, C4, ANFO, PETN, RDX, HMX, Dynamite, ANFO, emulsion explosives) with TNT equivalence factors and Gurney constants

**Day 3: Authentication system**
- `src/auth/mod.rs` — JWT token generation and validation middleware
- `src/auth/oauth.rs` — Google and GitHub OAuth 2.0 flow handlers
- `src/api/handlers/auth.rs` — Register, login, OAuth callback, refresh token, me endpoints
- Password hashing with bcrypt (cost factor 12)
- JWT with 24h access token + 30d refresh token
- Auth middleware that extracts Claims from Authorization header

**Day 4: User and project CRUD**
- `src/api/handlers/users.rs` — Get profile, update profile, delete account
- `src/api/handlers/projects.rs` — Create, list, get, update, delete project
- `src/api/handlers/orgs.rs` — Create org, invite member, list members, remove member
- `src/api/router.rs` — All route definitions with auth middleware
- Integration tests: auth flow, project CRUD, authorization checks

### Phase 2 — Blast Solver Core — Prototype (Days 5–10)

**Day 5: Kingery-Bulmash blast load calculator**
- `blast-solver/` — New Rust workspace member (shared between native and WASM)
- `blast-solver/src/kingery_bulmash.rs` — Polynomial evaluation for peak overpressure, impulse, duration, arrival time
- Coefficient tables for 5 scaled distance ranges (Z = 0.05 to 40)
- Surface burst scaling (2x charge mass for hemispherical)
- Unit tests: verify against UFC 3-340-02 reference values for standard charges (1kg, 10kg, 100kg at 10m, 50m, 100m)

**Day 6: Reflected pressure and Friedlander waveform**
- `blast-solver/src/reflection.rs` — Normal and oblique reflected pressure calculations
- `blast-solver/src/waveform.rs` — Friedlander waveform generation with positive and negative phase
- Decay coefficient fitting to match impulse constraint
- Tests: verify reflected pressure matches UFC 3-340-02 charts, verify impulse integration equals Kingery-Bulmash prediction

**Day 7: SDOF structural response framework**
- `blast-solver/src/sdof.rs` — SdofSystem struct, ResistanceFunction, Newmark-β integration
- Bilinear and trilinear resistance functions (elastic-plastic-degrading)
- Tests: simple spring-mass system with sinusoidal load (verify against analytical solution), elastic beam under impulse load

**Day 8: SDOF material models — RC walls and steel beams**
- `blast-solver/src/sdof_builders.rs` — `build_rc_wall_sdof()`, `build_steel_beam_sdof()`, `build_glazing_sdof()`
- RC wall: flexural capacity calculation, reinforcement ratio effects, dynamic increase factors
- Steel beam: plastic hinge formation, strain-rate effects (Cowper-Symonds)
- Tests: compare RC wall response to UFC 3-340-02 design charts, verify steel beam ductility ratios

**Day 9: Pressure-impulse diagram generation**
- `blast-solver/src/pi_diagram.rs` — Sweep (P, i) parameter space, run SDOF for each point, classify damage levels
- Iso-damage curve fitting (power law: (P/P0)^a + (i/i0)^b = 1)
- Generate 5 curves: superficial, moderate, heavy, hazardous, blowout
- Tests: verify P-I diagram for RC wall matches literature data (ASCE 59-11 reference curves)

**Day 10: Fragment velocity and trajectory**
- `blast-solver/src/fragments.rs` — Gurney equation for initial velocity, RK4 ballistic integration with drag
- Drag coefficient library (sphere, cube, plate, streamlined)
- Tests: verify Gurney velocity for C4 casing (compare to explosive engineering references), verify trajectory range vs. analytical ballistic formula (no drag)

### Phase 3 — WASM Build + Frontend Visualization (Days 11–18)

**Day 11: WASM solver compilation**
- `blast-solver-wasm/` — New workspace member for WASM target
- `blast-solver-wasm/Cargo.toml` — wasm-bindgen, serde-wasm-bindgen dependencies
- `blast-solver-wasm/src/lib.rs` — WASM entry points: `calculate_blast_loads()`, `solve_sdof()`, `generate_pi_diagram()`, `calculate_fragment_trajectories()`
- `wasm-pack build --target web --release` pipeline
- Target bundle size <1.5MB gzipped
- JavaScript wrapper: `BlastSolver` class that loads WASM and provides async API

**Day 12: Frontend scaffold and 3D visualization foundation**
```bash
npm create vite@latest frontend -- --template react-ts
cd frontend
npm i zustand @tanstack/react-query axios three @react-three/fiber @react-three/drei deck.gl
```
- `src/App.tsx` — Router, auth context, layout
- `src/stores/projectStore.ts` — Zustand store for project state
- `src/stores/scenarioStore.ts` — Scenario state (detonation point, targets, explosives, results)
- `src/components/SceneViewer/SceneCanvas.tsx` — Three.js canvas with orbit controls, grid, lighting
- `src/lib/wasmLoader.ts` — WASM module initialization

**Day 13: 3D scenario editor — detonation point and target placement**
- `src/components/SceneViewer/DetonationPoint.tsx` — Draggable sphere for detonation point, height adjustment
- `src/components/SceneViewer/StructureModel.tsx` — Simple box geometries for walls, buildings, barriers
- `src/components/SceneViewer/ReceiverGrid.tsx` — Grid of receiver points for blast load sampling
- `src/components/SceneViewer/Toolbar.tsx` — Add structure, add receiver, delete, reset view
- Click-to-place interaction: raycasting onto ground plane
- Property panel: edit structure dimensions, material, position

**Day 14: Blast pressure contour visualization**
- `src/components/SceneViewer/PressureContours.tsx` — Deck.gl ContourLayer for overpressure heatmap
- Compute overpressure at grid points (50×50 grid covering scene)
- Color scale: green (<10kPa) → yellow (10-50kPa) → orange (50-200kPa) → red (>200kPa)
- Isoline rendering for key pressure thresholds (20kPa eardrum rupture, 35kPa structural damage onset, 83kPa serious structural damage)
- Toggle layer visibility

**Day 15: Fragment trajectory visualization**
- `src/components/SceneViewer/FragmentTrajectories.tsx` — Three.js line geometries for fragment paths
- Color-code by final velocity (hazardous vs. non-hazardous)
- Fragment density heatmap (Deck.gl HexagonLayer) showing impact distribution
- Time-slider animation: show fragment positions at time t
- Toggle individual trajectory visibility (performance: limit to 100 trajectories rendered)

**Day 16: GIS layer integration**
- `src/components/SceneViewer/GISLayer.tsx` — Deck.gl TileLayer for satellite imagery (MapBox/OpenStreetMap)
- Coordinate system switcher: relative (0,0 origin) vs. geographic (lat/lon)
- Import site plan: upload GeoJSON, KML, or image overlay
- Snap structures to imported geometry
- Standoff distance measurement tool (line tool with distance readout)

**Day 17: Results panel and damage visualization**
- `src/components/Results/BlastLoadsTable.tsx` — Table of receiver points with overpressure, impulse, arrival time
- `src/components/Results/StructuralResponsePanel.tsx` — For each target structure: max displacement, damage level, ductility ratio
- `src/components/Results/DamageLevelIndicator.tsx` — Color-coded damage level badges
- `src/components/Results/PIChart.tsx` — Pressure-impulse diagram with scenario point plotted
- Structure color-coding in 3D view by damage level: green (superficial) → yellow (moderate) → orange (heavy) → red (hazardous) → black (blowout)

**Day 18: Scenario management UI**
- `src/pages/Project.tsx` — Project dashboard with scenario list
- `src/pages/ScenarioEditor.tsx` — Full scenario editor (3D view + side panels)
- `src/components/ExplosivePicker.tsx` — Dropdown for explosive type, charge mass input with TNT-equivalent calculation
- `src/components/ThreatLibrary.tsx` — Pre-defined threat scenarios (Small VBIED, Large VBIED, PBIED) with one-click apply
- `src/components/StandoffCalculator.tsx` — Given building type and threat, calculate required standoff per GSA/FEMA criteria
- Save/load scenario state to backend

### Phase 4 — API + Report Generation (Days 19–24)

**Day 19: Scenario API endpoints**
- `src/api/handlers/scenario.rs` — Create scenario, get scenario, list scenarios, update scenario results, delete scenario
- Explosive and material library endpoints: list, search, get by ID
- Threat library endpoints: list threats, get threat by ID
- Plan-based limits enforcement (free: 1000kg max charge, 10 scenarios/month)

**Day 20: Report generation service (Python FastAPI)**
```bash
mkdir report-service && cd report-service
pip install fastapi uvicorn matplotlib reportlab pandas numpy
```
- `report_service/main.py` — FastAPI app
- `report_service/reports/blast_summary.py` — Generate PDF with blast load summary, P-I diagrams, pressure contours
- `report_service/reports/structural_assessment.py` — Structural response summary, damage predictions, displacement plots
- `report_service/reports/compliance_check.py` — Check against UFC 4-010-01, ASCE 59-11, GSA criteria
- Use matplotlib for plots, ReportLab for PDF assembly
- Endpoint: `/generate-report` (POST) accepts scenario data, returns S3 URL to PDF

**Day 21: Report generation integration**
- `src/api/handlers/reports.rs` — Create report request, get report, list reports for scenario
- Call Python report service via HTTP, wait for completion, store S3 URL in database
- `frontend/src/components/Reports/ReportGenerator.tsx` — Select report type, generate, download PDF
- Report preview: embed PDF.js viewer in modal

**Day 22: Explosive and material library seeding**
- Seed 20 explosive types: TNT, C4, PETN, RDX, HMX, Composition B, ANFO, emulsion, dynamite, TNT/RDX mixes, improvised explosives
- TNT equivalence factors from UFC 3-340-02 and DoD sources
- Seed 30 structural materials: concrete (3000-8000 psi), steel (ASTM A36, A572, A992), masonry (CMU, brick), glazing (annealed, tempered, laminated, blast-resistant)
- Material properties: density, moduli, strengths, DIF parameters
- Seed 15 threat definitions: VBIED small/medium/large, PBIED small/medium/large, military munitions (155mm, 500lb bomb, 2000lb bomb, RPG, IED)

**Day 23: Compliance checking logic**
- `blast-solver/src/compliance.rs` — Functions to check standoff distance, structural response, glazing hazard against standards
- UFC 4-010-01: minimum standoff distances for building categories vs. threat levels
- ASCE 59-11: component damage acceptance criteria (Life Safety, Immediate Occupancy, Collapse Prevention)
- GSA blast criteria: window glazing hazard rating, peak deflection limits
- Return compliance status: pass/fail/marginal with explanation

**Day 24: Parametric sweep support**
- `src/api/handlers/sweeps.rs` — Create parametric sweep (vary charge mass, standoff, or structure properties)
- Server-side execution: queue sweep jobs, run WASM solver for each parameter set
- Progress tracking via WebSocket
- Results aggregation: generate CSV export, contour plots of damage level vs. parameters
- Frontend: `src/components/Sweeps/ParametricSweepEditor.tsx` — Define sweep parameters, visualize results as heatmap

### Phase 5 — Advanced Features (Days 25–29)

**Day 25: Penetration models integration**
- `blast-solver/src/penetration.rs` — Thor equation (steel), NDRC equation (concrete), empirical models for masonry, soil
- Fragment penetration assessment: for each fragment impact, compute penetration depth, perforation yes/no
- Behind-armor debris (BAD) prediction: residual velocity after perforation
- Tests: verify penetration equations against experimental data from literature

**Day 26: Multi-scenario comparison**
- `frontend/src/pages/Comparison.tsx` — Select 2-5 scenarios, display side-by-side or overlay
- Overlay pressure contours from multiple scenarios in different colors
- Table comparison: blast loads, structural response, damage levels
- Export comparison report

**Day 27: Fragment hazard zones and Pk mapping**
- `blast-solver/src/pk_mapping.rs` — Compute Pk (probability of kill/incapacitation) for personnel exposure
- Fragment kinetic energy thresholds: 79J (penetrate skin), 160J (life-threatening)
- Aggregate fragments in spatial bins, compute Pk contours
- Frontend: `src/components/SceneViewer/FragmentHazardZones.tsx` — Render Pk contours (0.01, 0.1, 0.5, 0.9) as colored zones

**Day 28: Glazing hazard assessment**
- `blast-solver/src/glazing.rs` — ASTM E1300 window breakage prediction, glass fragment throw distance
- Glazing hazard levels (GSA criteria): Safe, Very Low, Low, Medium, High
- Account for laminated vs. annealed glass, frame fixity
- Frontend: visualize glazing hazard rating for each window in 3D scene

**Day 29: Data export and API access**
- API endpoints for data export: `/scenarios/{id}/export` (JSON, CSV formats)
- Export blast load grid, structural response time-histories, fragment data
- REST API documentation (OpenAPI/Swagger)
- API key management for Advanced plan users
- Rate limiting: 100 requests/hour for free, 1000 requests/hour for Pro, unlimited for Advanced

### Phase 6 — Billing + Plan Enforcement (Days 30–33)

**Day 30: Stripe integration**
- `src/api/handlers/billing.rs` — Create checkout session, create customer portal session, get subscription status
- `src/api/handlers/webhooks/stripe.rs` — Handle subscription events: `checkout.session.completed`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.payment_failed`
- Plan mapping:
  - Free: 1000kg max charge, 10 scenarios/month, basic explosives/materials, WASM only
  - Pro ($199/mo): 10,000kg max charge, unlimited scenarios, full explosive/material library, PDF reports, compliance checking
  - Advanced ($499/user/mo): Unlimited charge size, parametric sweeps, API access, collaboration, fragment hazard analysis

**Day 31: Usage tracking and plan limits**
- `src/middleware/plan_limits.rs` — Middleware checking plan limits before scenario creation
- `src/services/usage.rs` — Track scenario calculations per billing period
- Usage record insertion after each scenario calculation completes
- Usage dashboard: `src/api/handlers/usage.rs` — Current period usage, historical usage
- Approaching-limit warnings at 80% and 100% of scenario quota

**Day 32: Billing UI**
- `frontend/src/pages/Billing.tsx` — Plan comparison table, current plan display, usage meter
- `frontend/src/components/billing/PlanCard.tsx` — Plan feature comparison cards
- `frontend/src/components/billing/UsageMeter.tsx` — Scenarios/month usage bar with threshold indicators
- Upgrade/downgrade flow via Stripe Customer Portal
- Upgrade prompt modals when hitting plan limits (e.g., "Charge size 5000kg requires Pro plan.")

**Day 33: Feature gating**
- Gate PDF report generation behind Pro plan
- Gate parametric sweeps behind Advanced plan
- Gate API access behind Advanced plan
- Gate fragment hazard analysis behind Advanced plan
- Show locked feature indicators with upgrade CTAs in UI
- Admin endpoint: override plan for specific users (beta testers, partners)

### Phase 7 — Solver Validation + Testing (Days 34–38)

**Day 34: Empirical blast load validation**
- Benchmark 1: 100kg TNT surface burst at 50m — verify overpressure = 48.9 kPa, impulse = 228 Pa·s (UFC 3-340-02 Table C-1)
- Benchmark 2: 1000kg TNT free-air burst at 100m — verify overpressure = 13.1 kPa, duration = 14.8 ms
- Benchmark 3: Reflected pressure normal incidence — 50 kPa incident → 164 kPa reflected (UFC 3-340-02 Figure 2-14)
- Benchmark 4: Friedlander waveform impulse integration — verify computed impulse matches Kingery-Bulmash within 1%
- Automated test suite: `blast-solver/tests/benchmarks.rs` with assertions within 2% tolerance

**Day 35: SDOF structural response validation**
- Benchmark 5: RC wall (250mm thick, f'c=30MPa, ρ=0.5%, 4m×4m fixed-fixed) under 100kPa, 200 Pa·s blast load — verify max displacement = 38mm, damage level = Heavy (compare to SBEDS spreadsheet)
- Benchmark 6: Steel W12×26 beam (simply supported, 6m span) under 50kPa uniform blast — verify max displacement = 125mm, ductility ratio = 8.3
- Benchmark 7: Laminated glazing (10mm, 2m×2m, fixed) under 20kPa, 50 Pa·s — verify no failure (Safe hazard level)
- Compare P-I diagram shapes to published ASCE 59-11 curves for RC walls

**Day 36: Fragment trajectory validation**
- Benchmark 8: Gurney velocity for C4 (√2E=2700 m/s) with C/M=0.3 cylindrical casing — verify V0 = 1566 m/s (analytical Gurney)
- Benchmark 9: Ballistic trajectory no drag (V0=1000 m/s, 45° angle) — verify range = 102 km (analytical: V0²/g)
- Benchmark 10: Ballistic trajectory with drag (1kg sphere, V0=500 m/s, 45°) — verify range ≈ 3.2 km (compare to numerical integration reference)
- Thor penetration: 10g fragment, 1500 m/s, 10mm steel plate — verify perforation with V_residual ≈ 1200 m/s

**Day 37: Integration testing**
- End-to-end test: create project → set up scenario → compute blast loads (WASM) → generate SDOF response → generate report → download PDF
- API integration tests: auth → project CRUD → scenario CRUD → explosive library search → report generation
- WASM solver test: load in headless browser (Playwright), run calculation, verify results match native solver
- Concurrent scenario test: create 5 scenarios simultaneously, verify all complete successfully

**Day 38: Performance testing and optimization**
- WASM solver benchmarks: measure time for 100-receiver grid, 500-receiver grid in Chrome/Firefox/Safari
- Target: 100 receivers (blast load + SDOF for 5 structures) < 200ms in WASM
- Target: 1000-fragment trajectory calculation < 500ms
- Report generation benchmark: PDF with 10 plots < 5 seconds
- Database query optimization: add indexes for common queries, verify <50ms response times
- Frontend bundle size: target total JS+WASM < 3MB gzipped

### Phase 8 — Polish + Launch Prep (Days 39–42)

**Day 39: Documentation and onboarding**
- Getting started guide: create first scenario, interpret results, generate report
- Video tutorials: basic blast load calculation, SDOF structural response, fragment analysis
- Help tooltips throughout UI explaining technical terms (scaled distance, P-I diagram, ductility ratio)
- FAQ page: common questions about blast standards, model limitations, plan features
- API documentation: endpoint reference, authentication, example requests

**Day 40: Error handling and edge cases**
- Validate all user inputs: charge mass ranges, geometric constraints, material property limits
- Handle WASM calculation failures gracefully: display error messages, suggest parameter adjustments
- Handle SDOF non-convergence: retry with smaller timestep, fallback to simplified resistance function
- Frontend error boundaries: catch React errors, display user-friendly messages
- Logging and monitoring: structured logs, error tracking (Sentry), performance metrics (Prometheus)

**Day 41: UI polish and accessibility**
- Consistent styling: design system with color palette, typography, spacing
- Dark mode support
- Responsive design: mobile-friendly layout for dashboard and results viewing (3D editor desktop-only)
- Accessibility: ARIA labels, keyboard navigation, screen reader support
- Loading states: skeletons, spinners, progress bars for async operations
- Empty states: helpful prompts when no projects/scenarios exist

**Day 42: Deployment and launch**
- Production deployment: AWS ECS for Rust API, Lambda for Python report service, RDS for PostgreSQL, S3 for storage, CloudFront for CDN
- Environment configuration: production secrets, database migrations, S3 bucket setup
- Monitoring setup: CloudWatch logs, Grafana dashboards for API latency/error rates, Sentry for error tracking
- Backup strategy: daily RDS snapshots, S3 versioning for reports
- Launch checklist: DNS configuration, SSL certificates, email service (transactional emails for auth), status page
- Soft launch: invite beta users, collect feedback, monitor for issues

---

## Critical Files

```
blastcalc/
├── backend/
│   ├── src/
│   │   ├── main.rs                    # Axum app entry point, routing
│   │   ├── config.rs                  # Environment configuration
│   │   ├── state.rs                   # AppState (DB pool, S3 client)
│   │   ├── error.rs                   # ApiError enum
│   │   ├── auth/
│   │   │   ├── mod.rs                 # JWT middleware
│   │   │   └── oauth.rs               # Google/GitHub OAuth
│   │   ├── api/
│   │   │   ├── handlers/
│   │   │   │   ├── auth.rs            # Auth endpoints
│   │   │   │   ├── projects.rs        # Project CRUD
│   │   │   │   ├── scenario.rs        # Scenario CRUD, results storage
│   │   │   │   ├── explosives.rs      # Explosive library API
│   │   │   │   ├── materials.rs       # Material library API
│   │   │   │   ├── reports.rs         # Report generation API
│   │   │   │   ├── billing.rs         # Stripe integration
│   │   │   │   └── webhooks/
│   │   │   │       └── stripe.rs      # Stripe webhook handler
│   │   │   └── router.rs              # Route definitions
│   │   ├── db/
│   │   │   ├── mod.rs                 # Database pool init
│   │   │   └── models.rs              # SQLx structs (User, Project, Scenario, etc.)
│   │   └── middleware/
│   │       └── plan_limits.rs         # Plan enforcement middleware
│   ├── migrations/
│   │   └── 001_initial.sql            # Database schema
│   ├── Cargo.toml
│   └── Dockerfile
├── blast-solver/                       # Core solver (Rust, compiles to native + WASM)
│   ├── src/
│   │   ├── lib.rs                     # Module exports
│   │   ├── kingery_bulmash.rs         # KB equations, polynomial evaluation
│   │   ├── reflection.rs              # Reflected pressure calculations
│   │   ├── waveform.rs                # Friedlander waveform generation
│   │   ├── sdof.rs                    # SDOF structural response solver
│   │   ├── sdof_builders.rs           # Material-specific SDOF system builders
│   │   ├── pi_diagram.rs              # P-I diagram generation
│   │   ├── fragments.rs               # Gurney velocity, trajectory integration
│   │   ├── penetration.rs             # Thor, NDRC penetration models
│   │   ├── compliance.rs              # UFC/GSA/ASCE compliance checking
│   │   └── pk_mapping.rs              # Fragment hazard Pk calculation
│   ├── tests/
│   │   └── benchmarks.rs              # Validation benchmarks
│   └── Cargo.toml
├── blast-solver-wasm/                  # WASM wrapper
│   ├── src/
│   │   └── lib.rs                     # wasm-bindgen entry points
│   ├── Cargo.toml
│   └── package.json
├── report-service/                     # Python FastAPI for PDF generation
│   ├── main.py                        # FastAPI app
│   ├── reports/
│   │   ├── blast_summary.py           # Blast load summary PDF
│   │   ├── structural_assessment.py   # Structural response PDF
│   │   └── compliance_check.py        # Compliance report PDF
│   ├── requirements.txt
│   └── Dockerfile
├── frontend/
│   ├── src/
│   │   ├── App.tsx                    # Main app, routing
│   │   ├── pages/
│   │   │   ├── Dashboard.tsx          # Project list
│   │   │   ├── Project.tsx            # Project detail, scenario list
│   │   │   ├── ScenarioEditor.tsx     # Full scenario editor (3D + panels)
│   │   │   ├── Comparison.tsx         # Multi-scenario comparison
│   │   │   └── Billing.tsx            # Billing, plan management
│   │   ├── components/
│   │   │   ├── SceneViewer/
│   │   │   │   ├── SceneCanvas.tsx    # Three.js canvas
│   │   │   │   ├── DetonationPoint.tsx
│   │   │   │   ├── StructureModel.tsx
│   │   │   │   ├── ReceiverGrid.tsx
│   │   │   │   ├── PressureContours.tsx # Deck.gl contours
│   │   │   │   ├── FragmentTrajectories.tsx
│   │   │   │   ├── GISLayer.tsx       # Satellite imagery overlay
│   │   │   │   └── FragmentHazardZones.tsx
│   │   │   ├── Results/
│   │   │   │   ├── BlastLoadsTable.tsx
│   │   │   │   ├── StructuralResponsePanel.tsx
│   │   │   │   ├── DamageLevelIndicator.tsx
│   │   │   │   └── PIChart.tsx        # P-I diagram plot
│   │   │   ├── ExplosivePicker.tsx
│   │   │   ├── ThreatLibrary.tsx
│   │   │   ├── StandoffCalculator.tsx
│   │   │   └── Reports/
│   │   │       └── ReportGenerator.tsx
│   │   ├── stores/
│   │   │   ├── projectStore.ts        # Project state (Zustand)
│   │   │   └── scenarioStore.ts       # Scenario state
│   │   ├── lib/
│   │   │   └── wasmLoader.ts          # WASM module initialization
│   │   └── hooks/
│   │       └── useBlastCalculation.ts # WASM blast calculation hook
│   ├── package.json
│   └── vite.config.ts
├── docker-compose.yml                  # Local dev: Postgres, MinIO, Redis
└── .github/
    └── workflows/
        ├── wasm-build.yml              # Build and deploy WASM solver
        ├── backend.yml                 # Build and deploy Rust API
        └── frontend.yml                # Build and deploy React frontend
```

---

## Solver Validation Suite

### Benchmark 1: Kingery-Bulmash Free-Air Overpressure
**Objective:** Validate free-air blast load calculations against UFC 3-340-02 reference data.

**Setup:**
- Explosive: 100 kg TNT spherical burst
- Standoff distances: 10m, 25m, 50m, 100m, 200m
- Burst type: Free-air (spherical)

**Expected Results (UFC 3-340-02 Table C-1):**

| Standoff (m) | Scaled Distance Z | Peak Overpressure (kPa) | Positive Impulse (Pa·s) | Duration (ms) |
|--------------|-------------------|-------------------------|-------------------------|---------------|
| 10 | 2.15 | 189.6 | 651 | 5.1 |
| 25 | 5.38 | 35.8 | 201 | 9.8 |
| 50 | 10.76 | 10.3 | 98 | 17.2 |
| 100 | 21.51 | 3.2 | 51 | 32.1 |
| 200 | 43.03 | 1.1 | 27 | 61.8 |

**Acceptance Criteria:** Calculated values within ±2% of reference for overpressure, ±5% for impulse and duration.

---

### Benchmark 2: Reflected Pressure (Normal Incidence)
**Objective:** Validate reflected pressure calculations for shock wave interaction with rigid surfaces.

**Setup:**
- Incident overpressure range: 10 kPa to 1000 kPa
- Angle of incidence: 0° (normal reflection)

**Expected Results (UFC 3-340-02 Figure 2-14):**

| Incident P (kPa) | Reflected P (kPa) | Reflection Factor |
|------------------|-------------------|-------------------|
| 10 | 28.2 | 2.82 |
| 50 | 164.0 | 3.28 |
| 100 | 349.0 | 3.49 |
| 200 | 742.0 | 3.71 |
| 500 | 1980.0 | 3.96 |
| 1000 | 4100.0 | 4.10 |

**Acceptance Criteria:** Calculated reflected pressure within ±3% of reference values across pressure range.

---

### Benchmark 3: SDOF RC Wall Response
**Objective:** Validate SDOF structural response solver against UFC 3-340-02 design methodology.

**Setup:**
- Component: Reinforced concrete wall, fixed-fixed boundary conditions
- Dimensions: 4m (width) × 4m (height) × 250mm (thickness)
- Material: Concrete f'c = 30 MPa, Steel f_y = 420 MPa
- Reinforcement: ρ = 0.5% (vertical), 0.3% (horizontal)
- Blast load: Peak overpressure = 100 kPa, Impulse = 200 Pa·s, Duration = 20 ms

**Expected Results (UFC 3-340-02 Design Charts):**
- Maximum displacement: 38 ± 3 mm
- Ductility ratio: 6.2 ± 0.5
- Support rotation: 2.1° ± 0.2°
- Damage level: Heavy (plastic hinge formation, repairable)

**Acceptance Criteria:** Displacement within ±8% of reference, damage level classification matches UFC 3-340-02 criteria.

---

### Benchmark 4: Gurney Fragment Velocity
**Objective:** Validate initial fragment velocity calculations using Gurney equations.

**Setup:**
- Explosive: C4 (√2E = 2700 m/s)
- Casing geometry: Cylindrical steel casing
- Charge-to-mass ratios: C/M = 0.1, 0.3, 0.5, 1.0, 2.0

**Expected Results (Analytical Gurney Equation):**

| C/M | Initial Velocity V₀ (m/s) |
|-----|---------------------------|
| 0.1 | 772 |
| 0.3 | 1566 |
| 0.5 | 2021 |
| 1.0 | 2400 |
| 2.0 | 2571 |

**Acceptance Criteria:** Calculated velocity within ±1% of analytical Gurney formula (exact match expected for implemented equation).

---

### Benchmark 5: Pressure-Impulse Diagram Generation
**Objective:** Validate P-I diagram generation and damage level classification.

**Setup:**
- Component: RC wall (same as Benchmark 3)
- Parameter sweep: Pressure range 10-500 kPa, Impulse range 50-1000 Pa·s
- Generate iso-damage curves for 5 damage levels

**Expected Results (ASCE 59-11 Reference Curves):**
- Superficial damage curve: P₀ = 15 kPa, i₀ = 75 Pa·s
- Moderate damage curve: P₀ = 45 kPa, i₀ = 150 Pa·s
- Heavy damage curve: P₀ = 120 kPa, i₀ = 320 Pa·s
- Hazardous damage curve: P₀ = 250 kPa, i₀ = 650 Pa·s
- Blowout curve: P₀ = 400 kPa, i₀ = 1100 Pa·s

**Acceptance Criteria:** Iso-damage curve asymptotes within ±10% of published ASCE 59-11 reference data for RC walls.

---

## Verification Checklist

### Solver Accuracy
- [ ] Kingery-Bulmash overpressure within ±2% of UFC 3-340-02 reference for Z = 0.5 to 40
- [ ] Impulse calculations within ±5% of UFC reference data
- [ ] Reflected pressure within ±3% of UFC Figure 2-14 across pressure range 10-1000 kPa
- [ ] Friedlander waveform impulse integration matches Kingery-Bulmash impulse within ±1%
- [ ] SDOF displacement predictions within ±8% of UFC design charts for RC walls, steel beams
- [ ] Gurney velocity calculations match analytical formula within ±1%
- [ ] P-I diagram iso-damage curves match ASCE 59-11 reference within ±10%
- [ ] Thor penetration equation matches experimental data within ±15% (acceptable for empirical model)
- [ ] NDRC concrete penetration matches test data within ±20%
- [ ] Fragment trajectory range (with drag) matches numerical reference within ±5%

### Functional Requirements
- [ ] User can create blast scenario with detonation point and targets
- [ ] WASM solver computes blast loads for 100 receivers in <200ms
- [ ] 3D visualization displays pressure contours with correct color scaling
- [ ] Fragment trajectories render smoothly with 100+ paths
- [ ] SDOF response computed for 5 structures in <50ms (WASM)
- [ ] P-I diagram generated with 5 damage level curves
- [ ] PDF report generated in <5 seconds with plots and compliance summary
- [ ] Standoff calculator provides correct minimum standoff per GSA/FEMA criteria
- [ ] Explosive library search returns correct TNT equivalence factors
- [ ] Material library provides accurate DIF and strain-rate parameters
- [ ] Threat library applies correct charge sizes for VBIED/PBIED scenarios
- [ ] Compliance checker identifies violations of UFC 4-010-01, ASCE 59-11, GSA criteria

### Performance
- [ ] WASM bundle size <1.5MB gzipped
- [ ] WASM module loads in <500ms on typical connection
- [ ] Frontend initial load <3MB total (JS + WASM + assets gzipped)
- [ ] API response time <100ms (p95) for scenario CRUD
- [ ] Database query time <50ms (p95) for common queries
- [ ] 3D scene renders at 60fps with 50 structures + 100 fragments
- [ ] Pressure contour layer updates in <100ms when toggled
- [ ] Parametric sweep (20 parameter sets) completes in <3 seconds

### Security & Reliability
- [ ] JWT auth with secure token expiration and refresh
- [ ] All user inputs validated and sanitized
- [ ] SQL injection prevented via parameterized queries (SQLx compile-time checks)
- [ ] XSS prevented via React auto-escaping
- [ ] CORS configured for production domain only
- [ ] Rate limiting: 100 req/hour (free), 1000 req/hour (pro), unlimited (advanced)
- [ ] Stripe webhooks verified with signature
- [ ] S3 URLs presigned with 1-hour expiration
- [ ] Database backups daily, 30-day retention
- [ ] Error tracking active (Sentry) with alert thresholds
- [ ] Logs structured (JSON) and shipped to CloudWatch

### Billing & Plans
- [ ] Free plan limits enforced: 1000kg max charge, 10 scenarios/month
- [ ] Pro plan unlocks: 10,000kg charge, unlimited scenarios, PDF reports
- [ ] Advanced plan unlocks: unlimited charge, parametric sweeps, API access, collaboration
- [ ] Usage tracking accurate within 1% of actual scenario count
- [ ] Stripe subscription status synced within 5 minutes of webhook
- [ ] Payment failure gracefully handled: warning email, grace period, downgrade
- [ ] Upgrade/downgrade flow completes successfully via Customer Portal
- [ ] Feature gates enforced: locked features show upgrade prompts

---

## Deployment Architecture

### Production Stack (AWS)

**Compute:**
- **ECS Fargate** for Rust API backend (Auto-scaling: 2-10 tasks, target CPU 70%)
- **Lambda** for Python report generation service (15-minute timeout, 3GB memory)
- **CloudFront** CDN for frontend static assets and WASM bundle

**Storage:**
- **RDS PostgreSQL 16** (db.t3.medium initially, Multi-AZ for production)
- **S3** buckets:
  - `blastcalc-reports` (generated PDFs, 90-day lifecycle)
  - `blastcalc-scenarios` (scenario snapshots, indefinite retention)
  - `blastcalc-wasm` (WASM bundles, versioned)
  - `blastcalc-assets` (frontend static assets, served via CloudFront)

**Networking:**
- **VPC** with public and private subnets across 2 AZs
- **ALB** for Rust API (HTTPS with ACM certificate)
- **NAT Gateway** for private subnet outbound traffic (RDS, Lambda)
- **Security Groups**: Strict ingress rules (ALB → ECS on port 8000, Lambda → RDS on 5432)

**Monitoring:**
- **CloudWatch Logs** for API and Lambda logs (7-day retention)
- **CloudWatch Metrics** for ECS CPU/memory, RDS connections, Lambda invocations
- **Grafana Cloud** for unified dashboards (API latency, error rates, scenario throughput)
- **Sentry** for error tracking and alerting (Slack integration)
- **Status page** (powered by Statuspage.io or custom) showing service health

### CI/CD Pipeline (GitHub Actions)

**On push to `main`:**
1. **Rust API:** Build Docker image, push to ECR, update ECS task definition, trigger rolling deployment
2. **WASM Solver:** `wasm-pack build --release`, optimize with `wasm-opt -Oz`, upload to S3 with version tag
3. **Frontend:** `npm run build`, upload to S3, invalidate CloudFront cache
4. **Report Service:** Build Docker image, deploy Lambda function via AWS SAM

**On pull request:**
1. Run Rust tests (`cargo test`), WASM tests (Playwright headless browser), frontend tests (Vitest)
2. Run `sqlx prepare` to verify database queries compile
3. Lint checks: `cargo clippy`, `eslint`, `prettier`

### Disaster Recovery
- **RDS automated backups:** Daily snapshots, 7-day retention, point-in-time recovery enabled
- **S3 versioning:** Enabled for all buckets, 30-day lifecycle to Glacier for old versions
- **Database restore procedure:** Documented runbook, tested quarterly, RTO = 4 hours, RPO = 24 hours
- **Failover:** Multi-AZ RDS automatic failover <60 seconds, ECS tasks across 2 AZs

---

## Post-MVP Roadmap

### Version 1.1 — CFD Blast Propagation (Months 3–5)
**Goal:** Add high-fidelity CFD solver for complex blast scenarios that empirical methods cannot handle (urban terrain, tunnels, multi-room buildings).

**Features:**
- Euler solver with adaptive mesh refinement (AMR) in Rust/CUDA
- JWL equation of state for detonation products, ideal gas for air
- Chapman-Jouguet detonation model for initiation wave
- Multi-material Euler: air, explosive, soil (for buried charges), water (underwater explosions)
- GPU-accelerated for real-time preview at reduced resolution
- Import CAD geometry (STL, OBJ) for urban terrain, building interiors
- Blast wave channeling between buildings, diffraction around corners
- Multi-room blast with venting through doorways/windows
- Server-side execution (GPU instances), results stored in S3
- 3D pressure field visualization (isosurfaces, slicing planes)

**Technical Approach:**
- Implement finite volume method with Godunov-type Riemann solver (HLLC or Roe)
- Octree-based AMR with refinement criteria on pressure gradient and shock front
- CUDA kernels for parallel flux computation, time integration (explicit RK3)
- Typical mesh: 1M-10M cells, 0.5-2 hour solve time on V100 GPU
- Validation: compare to AUTODYN reference simulations for standard test cases (blast in urban street canyon, corridor blast)

**Plan Placement:** Advanced tier only ($499/user/mo), limited to 10 CFD runs/month.

---

### Version 1.2 — Coupled Fluid-Structure Interaction (Months 6–8)
**Goal:** Predict wall breach, structural collapse, and progressive damage with two-way coupling between blast wave and deforming structure.

**Features:**
- 3D Lagrangian finite element structural model in Rust
- Damage models: concrete damage plasticity (CDP), steel Johnson-Cook plasticity, masonry discrete element
- Strain-rate effects: Cowper-Symonds for steel, DIF curves for concrete per UFC 3-340-02
- Erosion algorithm: remove failed elements (concrete spall, breach), redistribute loads
- Two-way FSI coupling: blast pressure loads structure → structure deforms → changes flow field
- Progressive collapse analysis: sequential removal of failed members, load redistribution
- Breach prediction: thickness of concrete/masonry wall perforated by blast
- Visualization: animated structural deformation with damage contours (strain, stress, plastic strain)

**Technical Approach:**
- Explicit FEA with central difference time integration (conditionally stable, Δt ~ 1 μs)
- Hourglass control for 8-node hexahedral elements
- Contact algorithm for debris interaction
- Couple to Euler solver via ghost fluid method: pressure from Euler → traction on Lagrangian surface, structural velocity → boundary condition for Euler
- Typical model: 100K-500K elements, 2-8 hour solve time on 16-core CPU
- Validation: compare to LS-DYNA benchmark problems (RC panel under blast, steel frame collapse)

**Plan Placement:** Advanced tier, unlimited FSI runs.

---

### Version 1.3 — Mining Blast Design Module (Months 9–10)
**Goal:** Serve mining engineers with blast pattern design, fragmentation prediction, vibration compliance.

**Features:**
- Drill pattern designer: burden, spacing, hole depth, diameter, stemming, charge distribution
- Timing optimization: delay sequence for fragmentation control and vibration mitigation (electronic detonators)
- Fragmentation prediction: Kuz-Ram model, Swebrec distribution, mean fragment size, oversize prediction
- Calibration: upload muckpile images, fit size distribution, update Kuz-Ram parameters
- Ground vibration: scaled distance law (USBM RI 8507), site-specific attenuation curves, PPV at receiver locations
- Airblast: overpressure prediction with terrain shielding, atmospheric effects (temperature, humidity)
- Flyrock: Lundborg model, initial velocity from burden/stemming ratio, ballistic trajectory, exclusion zone mapping
- Regulatory compliance: OSMRE (US), AS 2187 (Australia), BS 7385 (UK) vibration limits, automatic pass/fail assessment
- Wall control blasting: pre-split, smooth wall, trim blast spacing/charge recommendations
- Underground blasting: drift rounds (burn cut, V-cut), ring blasting for stopes

**Technical Approach:**
- Empirical models in Rust, fast parameter sweeps for timing optimization
- Image analysis (Python + OpenCV) for fragment size distribution from photos
- GIS integration: import mine site plans, contour maps for vibration propagation modeling
- Validation: compare fragmentation to Swebrec reference data, vibration predictions to monitoring data from operating mines

**Plan Placement:** New "Mining" tier at $299/user/mo, or add-on to Advanced tier for $150/mo.

---

### Version 1.4 — GovCloud and Classified Deployment (Months 11–12)
**Goal:** Enable defense and security customers to use BlastCalc for classified work with ITAR/EAR compliance.

**Features:**
- AWS GovCloud (US) deployment for FedRAMP compliance
- On-premise / air-gapped installation option for classified networks
- ITAR-compliant data handling: no data leaves customer environment
- Custom explosive characterization: upload JWL parameters from test data, fit to detonation velocity and pressure measurements
- Custom threat library: classified munitions, IED threat profiles
- Integration with force protection planning tools (JFPP, AFCESA)
- Detailed audit logs: all user actions, data access, calculation runs
- Multi-level security: separate projects by classification level (FOUO, Secret, Top Secret)

**Technical Approach:**
- Kubernetes deployment for on-premise with Helm charts
- Encrypted database at rest and in transit
- HSM-based key management for encryption keys
- Compliance documentation: security controls matrix, data flow diagrams, FedRAMP authorization package

**Plan Placement:** Enterprise custom pricing (typically $50K-$200K/year depending on deployment type and user count).

---

### Version 2.0 — Advanced Fragment Models and Behind-Armor Effects (Months 13–15)
**Goal:** Add primary fragmentation (Mott distribution), advanced penetration, behind-armor debris (BAD) for detailed lethality assessment.

**Features:**
- Mott distribution for natural fragmentation: fragment count, mass distribution, velocity distribution
- Fragment shape modeling: cube, cylinder, irregular (affects drag and penetration)
- Primary fragments: from cased munitions (artillery shells, bombs, grenades)
- Secondary fragments: blast-accelerated building debris (glass, concrete spall, metal panels)
- Advanced penetration: laminated armor, composite armor, spaced armor
- Behind-armor debris (BAD): residual velocity, spray cone angle, secondary fragment mass/velocity
- Ricochet modeling: critical angle for bounce vs. penetration, post-ricochet trajectory
- Shaped charge jet penetration: for anti-armor applications (Birkhoff equation, residual penetration)
- Personnel injury models: AIS (Abbreviated Injury Scale) prediction, Pk for head/torso/extremity injuries
- Armor plate optimization: recommend plate material, thickness, spacing for given threat

**Technical Approach:**
- Mott statistics: N(m) = N0 * (m/m_avg)^(-1-α) * exp(-m/m_avg), calibrate α from test data
- Fragment launch angle distribution: uniform, conical, or biased based on casing geometry
- Penetration: Walker-Anderson equation for long-rod penetrators, Florence equation for shaped charges
- BAD spray cone: momentum-based debris generation, break-up models
- Validation: compare fragment distributions to Arena test data, penetration to ballistic test reports

**Plan Placement:** Advanced tier feature, enabled by default for all Advanced users.

---

### Version 2.1 — Machine Learning for Damage Prediction (Months 16–18)
**Goal:** Train ML models on CFD/FEA simulation data to enable instant damage predictions without running full physics simulations.

**Features:**
- Surrogate model: ML regression (XGBoost, neural network) trained on 10K+ CFD/FEA runs
- Inputs: charge size, standoff, explosive type, structure geometry, material properties
- Outputs: peak displacement, damage level, breach yes/no, probability of collapse
- Uncertainty quantification: confidence intervals on predictions
- Active learning: suggest scenarios for full simulation to improve model accuracy in unexplored regions
- Explainability: SHAP values showing feature importance (e.g., "standoff distance most influential for this prediction")
- Transfer learning: fine-tune model with customer-specific simulation data

**Technical Approach:**
- Generate training dataset: automated parametric sweeps with CFD/FEA solvers (1 week GPU time)
- Feature engineering: scaled distance, structural aspect ratio, reinforcement ratio, impulse-to-resistance ratio
- Model architecture: gradient-boosted trees for tabular data, or ResNet for image-based inputs (structure geometry rendered as depth map)
- Train on GPU, deploy as TorchScript model in Rust backend for inference
- Validation: 90%+ accuracy on hold-out test set, compare to full simulations for edge cases

**Plan Placement:** Advanced tier, labeled as "Instant Damage Prediction (ML-powered)".

---

This implementation plan provides a detailed 42-day roadmap for building BlastCalc's MVP, covering all aspects from solver architecture to frontend visualization, billing integration, and deployment. The plan emphasizes Rust/WASM for high-performance empirical calculations, React/Three.js for 3D visualization, and rigorous validation against UFC 3-340-02 and ASCE 59-11 standards.
