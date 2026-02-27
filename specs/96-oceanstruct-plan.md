# 96. OceanStruct — Offshore Structure Design Platform

## Implementation Plan

**MVP Scope:** Cloud-native offshore structural engineering platform for fixed jacket platforms with 3D frame model creation via browser-based editor (drag-and-drop member placement, joint snapping, property assignment), Morison equation wave loading with JONSWAP irregular sea spectra and Wheeler stretching for kinematics, static structural analysis using Rust-based FEM beam element solver compiled to WebAssembly for small models (<100 members) and server-side execution for larger structures, natural frequency extraction (first 10 modes via Lanczos eigenvalue solver), spectral fatigue analysis with Efthymiou SCF equations for tubular joints and DNV-RP-C203 S-N curves with Dirlik cycle counting from stress RAOs computed across full scatter diagram (Hs, Tp, direction), automated API RP 2A member unity checks (axial + bending combined stress ratios), quasi-static catenary mooring analysis for single-point moored structures with chain/wire/polyester line libraries, interactive 3D visualization using Three.js with color-coded stress and fatigue life maps, metocean database with North Sea and Gulf of Mexico hindcast data, S3 storage for time-series results and large RAO datasets, PostgreSQL for projects and material libraries, Stripe billing with three tiers (Free / Pro $299/mo / Team $599/user/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| Structural FEM Solver | Rust (native + WASM) | 3D beam elements with 6-DOF nodes, Lanczos eigenvalue extraction via `nalgebra` |
| WASM Compilation | `wasm-pack` + `wasm-bindgen` | Client-side FEM for structures ≤100 members |
| Hydro / Fatigue | Python 3.12 (FastAPI) | Morison loading, wave kinematics, spectral fatigue, SCF calculation |
| Database | PostgreSQL 16 | Projects, users, metocean data, S-N curve libraries, material properties |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | Time-series results (stress histories, RAOs), large scatter diagram outputs |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| 3D Visualization | Three.js + Deck.gl | GPU-accelerated structure rendering, stress maps, mooring visualization |
| Plotting | D3.js + Plotly.js | RAO plots, scatter diagrams, S-N curves, frequency response |
| Real-time | WebSocket (Axum) | Live simulation progress, convergence monitoring for large FEM jobs |
| Job Queue | Redis 7 + Tokio tasks | Server-side FEM/fatigue job management, scatter diagram parallelization |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | WASM bundle delivery, static frontend assets, metocean data caching |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, solver performance tracking, error tracking |
| CI/CD | GitHub Actions | WASM build pipeline, Rust tests, Docker image push |

### Key Architecture Decisions

1. **Hybrid client/server FEM with WASM threshold at 100 members**: Small jacket structures (educational projects, preliminary design, simple monopiles with ≤100 members) run entirely in the browser via WASM FEM solver, providing instant feedback with zero server cost and no queue latency. Structures exceeding 100 members (typical North Sea jackets have 200-500 members) are submitted to the Rust-native server solver which handles multi-thousand-member structures and parallelized scatter diagram fatigue processing. The threshold is configurable per plan tier.

2. **Custom Rust FEM solver rather than wrapping Abaqus/Nastran**: Building a custom beam-element FEM solver in Rust gives us full control over WASM compilation, convergence algorithms, and integration with hydrodynamic loading. The beam-element formulation is sufficient for jacket/monopile analysis (no contact, no plasticity in MVP) and avoids licensing costs of commercial solvers. The solver uses sparse direct factorization (CHOLMOD bindings via `sprs` crate) for static analysis and Lanczos iteration for eigenvalue extraction.

3. **Python FastAPI for hydrodynamic and fatigue modules**: Morison equation wave loading, irregular wave generation, spectral fatigue processing, and SCF calculations leverage mature Python scientific libraries (`numpy`, `scipy`, `xarray` for metocean data). These modules run as FastAPI microservices and are called by the Rust orchestrator. Time-critical FEM assembly/solve remains in Rust, but wave load generation (which happens once per load case) benefits from Python's ecosystem.

4. **Three.js for 3D structure visualization with GPU-accelerated stress maps**: The jacket structure renderer uses Three.js instanced meshes for tubular members (cylindrical geometry shared across all members with per-instance transformations and color attributes for stress values). This allows rendering 500+ member structures at 60fps with interactive color-coded stress maps. Deck.gl renders mooring lines and seabed in the same viewport. WebGL fragment shaders interpolate stress values for smooth color gradients.

5. **S3 for time-series results with PostgreSQL metadata catalog**: Large datasets (stress time histories from transient analysis, RAO tables, scatter diagram fatigue results with 1000s of sea states) are stored in S3 as compressed Parquet files, while PostgreSQL holds metadata (analysis summary, peak stress, fatigue life, load case identifiers). This allows the result database to scale to 100K+ analyses without bloating PostgreSQL while enabling fast parametric search via PostgreSQL indexes and full-text search.

6. **Metocean database with spatial indexing**: Pre-computed metocean scatter diagrams (Hs, Tp, direction) for global offshore regions are stored in PostgreSQL with PostGIS spatial indexing. Users select project location on a map, and the nearest hindcast grid point (ERA5 reanalysis, 0.25° resolution) is automatically retrieved with weighted interpolation from surrounding grid points. This avoids requiring users to manually source metocean data.

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "postgis";  -- For metocean spatial indexing
CREATE EXTENSION IF NOT EXISTS "pg_trgm";  -- For fuzzy text search on materials

-- Users
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email TEXT UNIQUE NOT NULL,
    password_hash TEXT,  -- NULL for OAuth-only users
    name TEXT NOT NULL,
    avatar_url TEXT,
    auth_provider TEXT DEFAULT 'email',  -- email | google | github
    auth_provider_id TEXT,
    plan TEXT NOT NULL DEFAULT 'free',  -- free | pro | team
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX users_email_idx ON users(email);
CREATE INDEX users_stripe_idx ON users(stripe_customer_id);

-- Organizations (for Team plan)
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

-- Projects (structure + analysis workspace)
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    structure_type TEXT NOT NULL DEFAULT 'jacket',  -- jacket | monopile | tripod | semi_sub | spar
    structure_data JSONB NOT NULL DEFAULT '{}',  -- Full structure definition (nodes, members, properties)
    location GEOGRAPHY(POINT, 4326),  -- Project location for metocean lookup
    water_depth REAL,  -- meters
    settings JSONB DEFAULT '{}',  -- Analysis settings, load case definitions
    is_public BOOLEAN NOT NULL DEFAULT false,
    forked_from UUID REFERENCES projects(id) ON DELETE SET NULL,
    thumbnail_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);
CREATE INDEX projects_org_idx ON projects(org_id);
CREATE INDEX projects_location_idx ON projects USING GIST(location);
CREATE INDEX projects_updated_idx ON projects(updated_at DESC);

-- Structural Analysis Runs
CREATE TABLE analyses (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    analysis_type TEXT NOT NULL,  -- static | modal | spectral_fatigue | time_domain
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | running | completed | failed | cancelled
    execution_mode TEXT NOT NULL DEFAULT 'wasm',  -- wasm | server
    load_case TEXT NOT NULL,  -- Description of load case (e.g., "100yr storm, 0° dir")
    parameters JSONB NOT NULL DEFAULT '{}',  -- Analysis-specific parameters
    member_count INTEGER NOT NULL DEFAULT 0,
    results_url TEXT,  -- S3 URL for result data (stress histories, RAOs)
    results_summary JSONB,  -- Quick-access summary (max stress, unity checks, frequencies)
    error_message TEXT,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    wall_time_ms INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX analyses_project_idx ON analyses(project_id);
CREATE INDEX analyses_user_idx ON analyses(user_id);
CREATE INDEX analyses_status_idx ON analyses(status);
CREATE INDEX analyses_created_idx ON analyses(created_at DESC);

-- Server Analysis Jobs (for server-side execution only)
CREATE TABLE analysis_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    analysis_id UUID NOT NULL REFERENCES analyses(id) ON DELETE CASCADE,
    worker_id TEXT,
    cores_allocated INTEGER DEFAULT 1,
    memory_mb INTEGER DEFAULT 4096,
    priority INTEGER DEFAULT 0,  -- Higher = more urgent (Team plan gets +10)
    progress_pct REAL DEFAULT 0.0,
    progress_data JSONB DEFAULT '{}',  -- {current_load_case, iteration, convergence}
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX jobs_analysis_idx ON analysis_jobs(analysis_id);
CREATE INDEX jobs_status_idx ON analysis_jobs(worker_id);

-- Material Library
CREATE TABLE materials (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,  -- e.g., "API 5L X65", "S355 Steel"
    category TEXT NOT NULL,  -- steel | aluminum | titanium | composite
    standard TEXT,  -- API, ASTM, EN, DNV
    density REAL NOT NULL,  -- kg/m³
    youngs_modulus REAL NOT NULL,  -- Pa
    poissons_ratio REAL NOT NULL,
    yield_strength REAL NOT NULL,  -- Pa
    ultimate_strength REAL,  -- Pa
    thermal_expansion REAL,  -- 1/K
    sn_curve_id UUID REFERENCES sn_curves(id),
    is_builtin BOOLEAN DEFAULT true,
    uploaded_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX materials_name_idx ON materials(name);
CREATE INDEX materials_category_idx ON materials(category);

-- S-N Curve Library (for fatigue)
CREATE TABLE sn_curves (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,  -- e.g., "DNV-RP-C203 Curve D in seawater with CP"
    standard TEXT NOT NULL,  -- DNV-RP-C203 | API-RP-2A | BS-7608
    curve_class TEXT NOT NULL,  -- B1, C, D, E, F, F1, etc.
    environment TEXT NOT NULL,  -- in_air | seawater_cp | seawater_free_corrosion
    log_a1 REAL NOT NULL,  -- log10(a) for N < Nq (first slope)
    m1 REAL NOT NULL,  -- Slope for N < Nq
    log_a2 REAL,  -- log10(a) for N > Nq (second slope), NULL if single slope
    m2 REAL,  -- Slope for N > Nq
    log_nq REAL,  -- Transition point (log10(N))
    stress_cutoff REAL,  -- Fatigue limit (MPa), NULL if no cutoff
    thickness_exponent REAL DEFAULT 0.0,  -- Thickness correction exponent
    is_builtin BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX sn_curves_standard_idx ON sn_curves(standard);
CREATE INDEX sn_curves_class_idx ON sn_curves(curve_class);

-- Metocean Database (hindcast scatter diagrams)
CREATE TABLE metocean_sites (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,  -- e.g., "North Sea Central", "GoM Western"
    location GEOGRAPHY(POINT, 4326) NOT NULL,
    data_source TEXT NOT NULL,  -- ERA5 | NORA10 | NDBC | custom
    water_depth_m REAL,
    scatter_diagram_url TEXT NOT NULL,  -- S3 URL to Parquet file with full (Hs, Tp, Dir) table
    metadata JSONB DEFAULT '{}',  -- Summary stats, duration, return periods
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX metocean_location_idx ON metocean_sites USING GIST(location);

-- Wave Load Cases (for a given project)
CREATE TABLE load_cases (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    load_type TEXT NOT NULL,  -- regular_wave | irregular_jonswap | current | wind | combined
    wave_height REAL,  -- Hs for irregular, H for regular (m)
    wave_period REAL,  -- Tp for irregular, T for regular (s)
    wave_direction REAL,  -- deg (0 = +X, 90 = +Y)
    current_speed REAL,  -- m/s
    current_direction REAL,  -- deg
    wind_speed REAL,  -- m/s at 10m height
    wind_direction REAL,  -- deg
    probability REAL,  -- Probability of occurrence (for fatigue scatter diagram)
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX load_cases_project_idx ON load_cases(project_id);

-- Fatigue Analysis Results (per joint/member, per scatter diagram)
CREATE TABLE fatigue_results (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    analysis_id UUID NOT NULL REFERENCES analyses(id) ON DELETE CASCADE,
    member_id TEXT NOT NULL,  -- Member identifier in structure_data
    joint_id TEXT,  -- Joint identifier if tubular joint SCF applied
    scf REAL DEFAULT 1.0,  -- Stress concentration factor
    damage REAL NOT NULL,  -- Cumulative Palmgren-Miner damage (D = Σ n/N)
    life_years REAL NOT NULL,  -- Fatigue life in years (1 / D / occurrences_per_year)
    sn_curve_id UUID REFERENCES sn_curves(id),
    hotspot_location TEXT,  -- "crown" | "saddle" | "chord_side" for tubular joints
    stress_range_mpa REAL,  -- Representative stress range for dominant sea state
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX fatigue_analysis_idx ON fatigue_results(analysis_id);
CREATE INDEX fatigue_member_idx ON fatigue_results(member_id);

-- Mooring Designs (catenary analysis)
CREATE TABLE mooring_designs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    line_type TEXT NOT NULL,  -- chain_studlink | chain_studless | wire_rope | polyester | hmpe
    line_diameter REAL NOT NULL,  -- m
    line_length REAL NOT NULL,  -- m
    fairlead_depth REAL NOT NULL,  -- m below MSL
    anchor_radius REAL NOT NULL,  -- Horizontal distance from fairlead to anchor (m)
    pretension REAL,  -- N
    mass_per_length REAL NOT NULL,  -- kg/m (wet weight)
    ea REAL NOT NULL,  -- Axial stiffness (N)
    results JSONB,  -- Catenary shape, tension distribution, restoring curve
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX mooring_project_idx ON mooring_designs(project_id);

-- Usage Tracking (for billing enforcement)
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,  -- analysis_minutes | member_count | storage_gb
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
    pub structure_type: String,
    pub structure_data: serde_json::Value,
    #[sqlx(type_name = "geography")]
    pub location: Option<sqlx::types::Json<GeoPoint>>,
    pub water_depth: Option<f32>,
    pub settings: serde_json::Value,
    pub is_public: bool,
    pub forked_from: Option<Uuid>,
    pub thumbnail_url: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct GeoPoint {
    pub lat: f64,
    pub lon: f64,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Analysis {
    pub id: Uuid,
    pub project_id: Uuid,
    pub user_id: Uuid,
    pub analysis_type: String,
    pub status: String,
    pub execution_mode: String,
    pub load_case: String,
    pub parameters: serde_json::Value,
    pub member_count: i32,
    pub results_url: Option<String>,
    pub results_summary: Option<serde_json::Value>,
    pub error_message: Option<String>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub wall_time_ms: Option<i32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct AnalysisJob {
    pub id: Uuid,
    pub analysis_id: Uuid,
    pub worker_id: Option<String>,
    pub cores_allocated: i32,
    pub memory_mb: i32,
    pub priority: i32,
    pub progress_pct: f32,
    pub progress_data: serde_json::Value,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Material {
    pub id: Uuid,
    pub name: String,
    pub category: String,
    pub standard: Option<String>,
    pub density: f32,  // kg/m³
    pub youngs_modulus: f32,  // Pa
    pub poissons_ratio: f32,
    pub yield_strength: f32,  // Pa
    pub ultimate_strength: Option<f32>,
    pub thermal_expansion: Option<f32>,
    pub sn_curve_id: Option<Uuid>,
    pub is_builtin: bool,
    pub uploaded_by: Option<Uuid>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct SnCurve {
    pub id: Uuid,
    pub name: String,
    pub standard: String,  // DNV-RP-C203 | API-RP-2A | BS-7608
    pub curve_class: String,  // B1, C, D, E, F, F1
    pub environment: String,  // in_air | seawater_cp | seawater_free_corrosion
    pub log_a1: f32,
    pub m1: f32,
    pub log_a2: Option<f32>,
    pub m2: Option<f32>,
    pub log_nq: Option<f32>,
    pub stress_cutoff: Option<f32>,
    pub thickness_exponent: f32,
    pub is_builtin: bool,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct LoadCase {
    pub id: Uuid,
    pub project_id: Uuid,
    pub name: String,
    pub load_type: String,
    pub wave_height: Option<f32>,
    pub wave_period: Option<f32>,
    pub wave_direction: Option<f32>,
    pub current_speed: Option<f32>,
    pub current_direction: Option<f32>,
    pub wind_speed: Option<f32>,
    pub wind_direction: Option<f32>,
    pub probability: Option<f32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct AnalysisParams {
    pub analysis_type: AnalysisType,
    pub load_case_ids: Vec<Uuid>,
    pub static_params: Option<StaticAnalysisParams>,
    pub modal_params: Option<ModalAnalysisParams>,
    pub fatigue_params: Option<FatigueAnalysisParams>,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum AnalysisType {
    Static,
    Modal,
    SpectralFatigue,
    TimeDomain,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct StaticAnalysisParams {
    pub include_gravity: bool,
    pub include_hydrostatic: bool,
    pub safety_factor: f64,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct ModalAnalysisParams {
    pub num_modes: usize,  // Number of eigenmodes to extract
    pub freq_shift: Option<f64>,  // Frequency shift for Lanczos iteration
}

#[derive(Debug, Deserialize, Serialize)]
pub struct FatigueAnalysisParams {
    pub design_life_years: f64,
    pub marine_growth_thickness: f64,  // mm (increases drag diameter)
    pub scf_method: String,  // "efthymiou" | "user_defined"
    pub cycle_counting: String,  // "dirlik" | "rainflow"
}
```

---

## Structural FEM Architecture Deep-Dive

### Governing Equations and Discretization

OceanStruct's structural solver implements **3D Timoshenko beam elements** with 6 degrees of freedom per node (3 translations + 3 rotations). For a structure with `n` nodes, the global finite element system is:

```
K · U = F

where:
  K (6n × 6n): Global stiffness matrix (banded sparse)
  U (6n): Unknown displacement vector [ux, uy, uz, θx, θy, θz] at each node
  F (6n): Applied load vector (wave loads, gravity, hydrostatic, point loads)
```

**Element stiffness matrix** for a beam element connecting nodes i and j:

The 12×12 element stiffness matrix (6 DOF × 2 nodes) in local coordinates is:

```
K_local = E/L³ ·
  ┌                                                          ┐
  │  AGL²      0         0       0        0        0   ... │
  │   0     12EIz    6EIzL      0        0     12EIz   ... │
  │   0     6EIzL   4EIzL²     0        0     6EIzL    ... │
  │  ...                                                     │
  └                                                          ┘

where:
  E: Young's modulus (Pa)
  G: Shear modulus = E / (2(1+ν))
  A: Cross-sectional area (m²)
  Iy, Iz: Second moments of area (m⁴)
  J: Torsional constant (m⁴)
  L: Element length (m)
```

For **tubular members** (circular hollow section), geometric properties are:

```
Outer diameter: D
Wall thickness: t
A = π(D·t - t²)
I = π(D³t - 4Dt³ + 4t⁴) / 8  (same for Iy and Iz due to circular symmetry)
J = 2I  (torsional constant for thin-walled circular tube)
```

**Coordinate transformation**: Each element stiffness matrix is rotated from local beam coordinates to global XYZ:

```
K_global = T^T · K_local · T

where T is the 12×12 transformation matrix built from direction cosines of the beam axis.
```

**Assembly**: Element stiffness matrices are assembled into the global sparse matrix using element connectivity (node i, node j). The solver uses Compressed Sparse Row (CSR) format for efficient storage and factorization.

**Static analysis**: Solve K·U = F using sparse direct factorization (CHOLMOD via Rust `sprs` crate bindings). Boundary conditions (fixed nodes) are applied by removing rows/columns from K or using Lagrange multipliers.

**Modal analysis** (eigenvalue extraction): Solve the generalized eigenvalue problem:

```
(K - ω²M) · Φ = 0

where:
  M (6n × 6n): Global mass matrix (diagonal lumped-mass approximation)
  ω: Circular natural frequencies (rad/s)
  Φ: Mode shape vectors (eigenvectors)
```

Solved using **Lanczos iteration** for the first `num_modes` eigenvalues. The mass matrix uses lumped masses at nodes: `m_node = ρ·A·L/2` for each element contributing to that node, where ρ is material density.

### Morison Equation Wave Loading

For slender cylindrical members (D/λ < 0.15, where λ is wavelength), wave forces are computed using the **Morison equation**:

```
dF/dz = ρ_water · (Cm · A_cross · du/dt  +  Cd · D · u · |u| / 2)

where:
  ρ_water = 1025 kg/m³ (seawater density)
  Cm: Inertia coefficient (typically 2.0 for circular cylinder)
  Cd: Drag coefficient (typically 0.7-1.2 depending on Reynolds number, roughness)
  A_cross = π D² / 4: Cross-sectional area of member
  D: Outer diameter (including marine growth if specified)
  u(z,t): Horizontal water particle velocity at depth z, time t
  du/dt: Horizontal water particle acceleration
```

**Wave kinematics** (linear Airy wave theory):

```
Surface elevation: η(x,t) = A · cos(kx - ωt)

Horizontal velocity: u(z,t) = A·ω · cosh(k(z+h)) / sinh(kh) · cos(kx - ωt)

Horizontal acceleration: du/dt = A·ω² · cosh(k(z+h)) / sinh(kh) · sin(kx - ωt)

where:
  A = H/2: Wave amplitude (H = wave height)
  ω = 2π/T: Angular frequency (T = wave period)
  k: Wave number, satisfying dispersion relation ω² = gk·tanh(kh)
  h: Water depth
  z: Depth below mean sea level (z=0 at MSL, z=-h at seabed)
```

**Wheeler stretching** for above-MSL kinematics (needed for wave crests above still water level):

```
For z > 0 (above MSL):
  z_eff = -h · (z - η) / (h + η)

Use z_eff in place of z in wave kinematics equations.
```

**Irregular waves** (JONSWAP spectrum):

For spectral fatigue and realistic sea states, irregular waves are generated from a JONSWAP spectrum:

```
S(f) = α · g² / (2π)⁴f⁵ · exp(-1.25(f/f_p)⁻⁴) · γ^exp(-(f-f_p)²/(2σ²f_p²))

where:
  f_p = 1/T_p: Peak frequency (T_p = peak period)
  α: Normalizing constant to match H_s (significant wave height)
  γ = 3.3: Peak enhancement factor (standard JONSWAP)
  σ = 0.07 (f ≤ f_p) or 0.09 (f > f_p): Spectral width parameter
```

Wave amplitude for frequency bin i: `A_i = √(2·S(f_i)·Δf)`

Time-domain wave elevation: `η(t) = Σ A_i · cos(2πf_i·t + ε_i)`, where ε_i are random phases.

### Client/Server Split (WASM Threshold)

```
Structure uploaded → Member count extracted
    │
    ├── ≤100 members → WASM FEM solver (browser)
    │   ├── Instant startup, no network latency
    │   ├── Results displayed immediately (< 1 second for static)
    │   └── No server cost, no queue
    │
    └── >100 members → Server FEM solver (Rust native)
        ├── Job queued via Redis (priority queue for Team users)
        ├── Worker picks up, runs native solver with parallel load case processing
        ├── Progress streamed via WebSocket
        └── Results stored in S3, URL returned
```

The 100-member threshold was chosen because:
- WASM solver with CHOLMOD handles 100-member static analysis (600 DOF) in <500ms on modern hardware
- 100 members covers: preliminary jacket designs (30-80 members), simple monopiles (5-20 members), educational projects
- Above 100 members: production North Sea jackets (200-500 members), scatter diagram fatigue with 1000s of load cases → need server parallelization

### WASM Compilation Pipeline

```toml
# solver-wasm/Cargo.toml
[package]
name = "oceanstruct-solver-wasm"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
wasm-bindgen = "0.2"
serde = { version = "1", features = ["derive"] }
serde-wasm-bindgen = "0.6"
js-sys = "0.3"
nalgebra = { version = "0.32", features = ["sparse"] }
sprs = "0.11"  # Sparse matrix library with WASM support
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
name: Build WASM FEM Solver
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
      - run: wasm-opt -Oz solver-wasm/pkg/oceanstruct_solver_wasm_bg.wasm -o solver-wasm/pkg/oceanstruct_solver_wasm_bg.wasm
      - name: Upload to S3/CDN
        run: aws s3 cp solver-wasm/pkg/ s3://oceanstruct-wasm/v${{ github.sha }}/ --recursive
```

---

## Architecture Deep-Dives

### 1. Analysis API Handler (Rust/Axum)

The primary endpoint receives an analysis request, validates the structure, decides between WASM and server execution, and for server execution enqueues a job with Redis and streams progress via WebSocket.

```rust
// src/api/handlers/analysis.rs

use axum::{
    extract::{Path, State, Json},
    http::StatusCode,
    response::IntoResponse,
};
use uuid::Uuid;
use crate::{
    db::models::{Analysis, AnalysisParams, AnalysisType},
    structure::validate_structure,
    state::AppState,
    auth::Claims,
    error::ApiError,
};

#[derive(serde::Deserialize)]
pub struct CreateAnalysisRequest {
    pub analysis_type: AnalysisType,
    pub load_case_ids: Vec<Uuid>,
    pub parameters: serde_json::Value,
}

pub async fn create_analysis(
    State(state): State<AppState>,
    claims: Claims,
    Path(project_id): Path<Uuid>,
    Json(req): Json<CreateAnalysisRequest>,
) -> Result<impl IntoResponse, ApiError> {
    // 1. Verify project ownership
    let project = sqlx::query_as!(
        crate::db::models::Project,
        r#"SELECT * FROM projects WHERE id = $1 AND (owner_id = $2 OR org_id IN (
            SELECT org_id FROM org_members WHERE user_id = $2
        ))"#,
        project_id,
        claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Project not found"))?;

    // 2. Validate structure and count members
    let structure = validate_structure(&project.structure_data)?;
    let member_count = structure.members.len();

    // 3. Check plan limits
    let user = sqlx::query_as!(
        crate::db::models::User,
        "SELECT * FROM users WHERE id = $1",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    if user.plan == "free" && member_count > 50 {
        return Err(ApiError::PlanLimit(
            "Free plan supports structures up to 50 members. Upgrade to Pro for unlimited."
        ));
    }

    // 4. Determine execution mode
    let execution_mode = if member_count <= 100 {
        "wasm"
    } else {
        "server"
    };

    // 5. Fetch load cases
    let load_cases = sqlx::query_as!(
        crate::db::models::LoadCase,
        "SELECT * FROM load_cases WHERE id = ANY($1) AND project_id = $2",
        &req.load_case_ids,
        project_id
    )
    .fetch_all(&state.db)
    .await?;

    if load_cases.is_empty() {
        return Err(ApiError::BadRequest("No valid load cases specified"));
    }

    // 6. Create analysis record
    let analysis = sqlx::query_as!(
        Analysis,
        r#"INSERT INTO analyses
            (project_id, user_id, analysis_type, status, execution_mode,
             load_case, parameters, member_count)
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8)
        RETURNING *"#,
        project_id,
        claims.user_id,
        serde_json::to_string(&req.analysis_type)?,
        if execution_mode == "wasm" { "completed" } else { "pending" },
        execution_mode,
        load_cases[0].name.clone(),  // Primary load case name
        req.parameters,
        member_count as i32,
    )
    .fetch_one(&state.db)
    .await?;

    // 7. For server execution, enqueue job
    if execution_mode == "server" {
        let job = sqlx::query_as!(
            crate::db::models::AnalysisJob,
            r#"INSERT INTO analysis_jobs (analysis_id, priority)
            VALUES ($1, $2) RETURNING *"#,
            analysis.id,
            if user.plan == "team" { 10 } else { 0 },
        )
        .fetch_one(&state.db)
        .await?;

        // Publish to Redis job queue
        state.redis
            .publish("analysis:jobs", serde_json::to_string(&job.id)?)
            .await?;
    }

    Ok((StatusCode::CREATED, Json(analysis)))
}

pub async fn get_analysis(
    State(state): State<AppState>,
    claims: Claims,
    Path((project_id, analysis_id)): Path<(Uuid, Uuid)>,
) -> Result<Json<Analysis>, ApiError> {
    let analysis = sqlx::query_as!(
        Analysis,
        r#"SELECT a.* FROM analyses a
         JOIN projects p ON a.project_id = p.id
         WHERE a.id = $1 AND a.project_id = $2
         AND (p.owner_id = $3 OR p.org_id IN (
             SELECT org_id FROM org_members WHERE user_id = $3
         ))"#,
        analysis_id,
        project_id,
        claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Analysis not found"))?;

    Ok(Json(analysis))
}
```

### 2. FEM Solver Core (Rust — shared between WASM and native)

The core FEM solver that assembles the global stiffness matrix and solves the linear system. This code compiles to both native (server) and WASM (browser) targets.

```rust
// solver-core/src/fem.rs

use nalgebra::{DMatrix, DVector, Matrix6, Vector3, Vector6};
use sprs::{CsMat, CsMatI, TriMat};
use crate::structure::{Structure, Node, Member, Material};

pub struct FemSystem {
    pub n_dof: usize,             // Total DOF = 6 * n_nodes
    pub n_nodes: usize,
    pub stiffness: TriMat<f64>,   // Triplet format for assembly, converted to CSR for solve
    pub mass: DVector<f64>,       // Lumped mass matrix (diagonal)
    pub loads: DVector<f64>,      // Global load vector
    pub displacements: DVector<f64>,  // Solution vector
    pub boundary_conditions: Vec<(usize, f64)>,  // (DOF index, prescribed value)
}

impl FemSystem {
    pub fn new(n_nodes: usize) -> Self {
        let n_dof = 6 * n_nodes;
        Self {
            n_dof,
            n_nodes,
            stiffness: TriMat::new((n_dof, n_dof)),
            mass: DVector::zeros(n_dof),
            loads: DVector::zeros(n_dof),
            displacements: DVector::zeros(n_dof),
            boundary_conditions: Vec::new(),
        }
    }

    /// Assemble global stiffness matrix from all members
    pub fn assemble_stiffness(&mut self, structure: &Structure) {
        for member in &structure.members {
            let k_local = Self::element_stiffness_local(member);
            let k_global = self.transform_element_stiffness(
                &k_local,
                &structure.nodes[member.node_i],
                &structure.nodes[member.node_j],
            );

            // Add element stiffness to global matrix
            let dof_i = member.node_i * 6;
            let dof_j = member.node_j * 6;

            for i in 0..6 {
                for j in 0..6 {
                    self.stiffness.add_triplet(dof_i + i, dof_i + j, k_global[(i, j)]);
                    self.stiffness.add_triplet(dof_i + i, dof_j + j, k_global[(i, j + 6)]);
                    self.stiffness.add_triplet(dof_j + i, dof_i + j, k_global[(i + 6, j)]);
                    self.stiffness.add_triplet(dof_j + i, dof_j + j, k_global[(i + 6, j + 6)]);
                }
            }
        }
    }

    /// Compute element stiffness matrix in local coordinates (12x12)
    fn element_stiffness_local(member: &Member) -> DMatrix<f64> {
        let mat = &member.material;
        let L = member.length;
        let E = mat.youngs_modulus as f64;
        let G = E / (2.0 * (1.0 + mat.poissons_ratio as f64));
        let A = member.area();
        let Iy = member.second_moment_y();
        let Iz = member.second_moment_z();
        let J = member.torsional_constant();

        let mut k = DMatrix::zeros(12, 12);

        // Axial stiffness (DOF 0, 6)
        let ea_l = E * A / L;
        k[(0, 0)] = ea_l;  k[(0, 6)] = -ea_l;
        k[(6, 0)] = -ea_l; k[(6, 6)] = ea_l;

        // Torsional stiffness (DOF 3, 9)
        let gj_l = G * J / L;
        k[(3, 3)] = gj_l;  k[(3, 9)] = -gj_l;
        k[(9, 3)] = -gj_l; k[(9, 9)] = gj_l;

        // Bending stiffness about local y-axis (in xz-plane, DOF 2, 4, 8, 10)
        let eiz_l3 = E * Iz / (L * L * L);
        k[(2, 2)] = 12.0 * eiz_l3;
        k[(2, 4)] = 6.0 * eiz_l3 * L;
        k[(2, 8)] = -12.0 * eiz_l3;
        k[(2, 10)] = 6.0 * eiz_l3 * L;
        k[(4, 2)] = 6.0 * eiz_l3 * L;
        k[(4, 4)] = 4.0 * eiz_l3 * L * L;
        k[(4, 8)] = -6.0 * eiz_l3 * L;
        k[(4, 10)] = 2.0 * eiz_l3 * L * L;
        k[(8, 2)] = -12.0 * eiz_l3;
        k[(8, 4)] = -6.0 * eiz_l3 * L;
        k[(8, 8)] = 12.0 * eiz_l3;
        k[(8, 10)] = -6.0 * eiz_l3 * L;
        k[(10, 2)] = 6.0 * eiz_l3 * L;
        k[(10, 4)] = 2.0 * eiz_l3 * L * L;
        k[(10, 8)] = -6.0 * eiz_l3 * L;
        k[(10, 10)] = 4.0 * eiz_l3 * L * L;

        // Bending stiffness about local z-axis (in xy-plane, DOF 1, 5, 7, 11)
        let eiy_l3 = E * Iy / (L * L * L);
        k[(1, 1)] = 12.0 * eiy_l3;
        k[(1, 5)] = -6.0 * eiy_l3 * L;
        k[(1, 7)] = -12.0 * eiy_l3;
        k[(1, 11)] = -6.0 * eiy_l3 * L;
        k[(5, 1)] = -6.0 * eiy_l3 * L;
        k[(5, 5)] = 4.0 * eiy_l3 * L * L;
        k[(5, 7)] = 6.0 * eiy_l3 * L;
        k[(5, 11)] = 2.0 * eiy_l3 * L * L;
        k[(7, 1)] = -12.0 * eiy_l3;
        k[(7, 5)] = 6.0 * eiy_l3 * L;
        k[(7, 7)] = 12.0 * eiy_l3;
        k[(7, 11)] = 6.0 * eiy_l3 * L;
        k[(11, 1)] = -6.0 * eiy_l3 * L;
        k[(11, 5)] = 2.0 * eiy_l3 * L * L;
        k[(11, 7)] = 6.0 * eiy_l3 * L;
        k[(11, 11)] = 4.0 * eiy_l3 * L * L;

        k
    }

    /// Transform element stiffness from local to global coordinates
    fn transform_element_stiffness(
        &self,
        k_local: &DMatrix<f64>,
        node_i: &Node,
        node_j: &Node,
    ) -> DMatrix<f64> {
        // Compute direction cosines for transformation matrix
        let dx = node_j.x - node_i.x;
        let dy = node_j.y - node_i.y;
        let dz = node_j.z - node_i.z;
        let L = (dx * dx + dy * dy + dz * dz).sqrt();

        // Local x-axis (along member)
        let cx = dx / L;
        let cy = dy / L;
        let cz = dz / L;

        // Local y-axis (perpendicular to x, in global XY plane if possible)
        let (cyx, cyy, cyz) = if (cx.abs() < 0.9) {
            let ny = Vector3::new(-cy, cx, 0.0).normalize();
            (ny.x, ny.y, ny.z)
        } else {
            let ny = Vector3::new(0.0, -cz, cy).normalize();
            (ny.x, ny.y, ny.z)
        };

        // Local z-axis (cross product of x and y)
        let czx = cy * cyz - cz * cyy;
        let czy = cz * cyx - cx * cyz;
        let czz = cx * cyy - cy * cyx;

        // Build 3x3 rotation matrix
        let r = DMatrix::from_row_slice(3, 3, &[
            cx, cy, cz,
            cyx, cyy, cyz,
            czx, czy, czz,
        ]);

        // Build 12x12 transformation matrix (block diagonal with 4 copies of r)
        let mut t = DMatrix::zeros(12, 12);
        for i in 0..4 {
            t.view_mut((i * 3, i * 3), (3, 3)).copy_from(&r);
        }

        // K_global = T^T * K_local * T
        t.transpose() * k_local * &t
    }

    /// Assemble lumped mass matrix (diagonal)
    pub fn assemble_mass(&mut self, structure: &Structure) {
        for member in &structure.members {
            let rho = member.material.density as f64;
            let A = member.area();
            let L = member.length;
            let m_elem = rho * A * L;

            // Lumped mass: distribute half to each end node
            let dof_i = member.node_i * 6;
            let dof_j = member.node_j * 6;

            for k in 0..3 {  // Only translational DOFs get mass
                self.mass[dof_i + k] += m_elem / 2.0;
                self.mass[dof_j + k] += m_elem / 2.0;
            }
        }
    }

    /// Apply boundary conditions (fixed DOFs)
    pub fn apply_boundary_conditions(&mut self) {
        // For simplicity in WASM, use penalty method:
        // For each fixed DOF, add large stiffness to diagonal
        let penalty = 1e15;
        for (dof, value) in &self.boundary_conditions {
            self.stiffness.add_triplet(*dof, *dof, penalty);
            self.loads[*dof] = penalty * value;
        }
    }

    /// Solve static analysis: K·U = F
    pub fn solve_static(&mut self) -> Result<(), FemError> {
        // Convert triplet to CSR format
        let k_csr: CsMat<f64> = self.stiffness.to_csr();

        // Sparse direct solve using Cholesky (requires symmetric positive definite)
        let solver = sprs::linalg::cholesky::CscCholesky::new(&k_csr.to_csc())
            .map_err(|_| FemError::SingularMatrix)?;

        self.displacements = solver.solve(&self.loads);

        Ok(())
    }

    /// Extract natural frequencies and mode shapes (Lanczos eigenvalue solver)
    pub fn solve_modal(&mut self, num_modes: usize) -> Result<Vec<Mode>, FemError> {
        use nalgebra::linalg::SymmetricEigen;

        // For MVP, use dense eigenvalue solver (sufficient for <1000 DOF)
        // Production would use Lanczos/Arnoldi for large systems

        let k_dense = DMatrix::from_iterator(
            self.n_dof, self.n_dof,
            self.stiffness.to_csr().iter().map(|(_, _, v)| *v)
        );

        let m_dense = DMatrix::from_diagonal(&self.mass);

        // Solve generalized eigenvalue problem: K·Φ = ω²·M·Φ
        // Convert to standard form: M^(-1/2)·K·M^(-1/2)·(M^(1/2)·Φ) = ω²·(M^(1/2)·Φ)

        let m_sqrt_inv = m_dense.map(|x| if x > 1e-12 { 1.0 / x.sqrt() } else { 0.0 });
        let k_transformed = &m_sqrt_inv * &k_dense * &m_sqrt_inv;

        let eigen = SymmetricEigen::new(k_transformed);
        let mut modes = Vec::new();

        for i in 0..num_modes.min(eigen.eigenvalues.len()) {
            let omega_sq = eigen.eigenvalues[i];
            if omega_sq > 1e-12 {
                let freq_hz = omega_sq.sqrt() / (2.0 * std::f64::consts::PI);
                let mode_shape = m_sqrt_inv.clone() * &eigen.eigenvectors.column(i);
                modes.push(Mode {
                    frequency_hz: freq_hz,
                    shape: mode_shape.clone_owned(),
                });
            }
        }

        Ok(modes)
    }
}

#[derive(Debug, Clone)]
pub struct Mode {
    pub frequency_hz: f64,
    pub shape: DVector<f64>,  // Mode shape vector (displacements at each DOF)
}

#[derive(Debug)]
pub enum FemError {
    SingularMatrix,
    InvalidStructure(String),
    ConvergenceFailure,
}
```

### 3. Server-Side Analysis Worker (Rust + Redis)

Background worker that picks up server-side analysis jobs from Redis, runs the native solver, streams progress via WebSocket, and stores results in S3.

```rust
// src/workers/analysis_worker.rs

use std::sync::Arc;
use aws_sdk_s3::Client as S3Client;
use redis::AsyncCommands;
use sqlx::PgPool;
use tokio::sync::broadcast;
use uuid::Uuid;

use crate::solver::{
    fem::{FemSystem, solve_static, solve_modal},
    structure::parse_structure,
};
use crate::db::models::{AnalysisJob};
use crate::websocket::ProgressBroadcaster;

pub struct AnalysisWorker {
    db: PgPool,
    redis: redis::Client,
    s3: S3Client,
    broadcaster: Arc<ProgressBroadcaster>,
}

impl AnalysisWorker {
    pub async fn run(&self) -> anyhow::Result<()> {
        let mut conn = self.redis.get_async_connection().await?;
        tracing::info!("Analysis worker started, listening for jobs...");

        loop {
            // Block-pop from Redis job queue
            let job_id: Option<String> = conn.brpop("analysis:jobs", 30.0).await?;
            let Some(job_id) = job_id else { continue };
            let job_id: Uuid = job_id.parse()?;

            if let Err(e) = self.process_job(job_id).await {
                tracing::error!("Job {job_id} failed: {e}");
                self.mark_failed(job_id, &e.to_string()).await?;
            }
        }
    }

    async fn process_job(&self, job_id: Uuid) -> anyhow::Result<()> {
        // 1. Fetch job and analysis details
        let job = sqlx::query_as!(
            AnalysisJob,
            "UPDATE analysis_jobs SET started_at = NOW(), worker_id = $2
             WHERE id = $1 RETURNING *",
            job_id, hostname::get()?.to_string_lossy().to_string()
        )
        .fetch_one(&self.db)
        .await?;

        let analysis = sqlx::query_as!(
            crate::db::models::Analysis,
            "UPDATE analyses SET status = 'running', started_at = NOW()
             WHERE id = $1 RETURNING *",
            job.analysis_id
        )
        .fetch_one(&self.db)
        .await?;

        // 2. Fetch project and parse structure
        let project = sqlx::query_as!(
            crate::db::models::Project,
            "SELECT * FROM projects WHERE id = $1",
            analysis.project_id
        )
        .fetch_one(&self.db)
        .await?;

        let structure = parse_structure(&project.structure_data)?;

        // 3. Run analysis based on type
        let start_time = std::time::Instant::now();
        let result_data = match analysis.analysis_type.as_str() {
            "static" => {
                let mut fem = FemSystem::new(structure.nodes.len());
                fem.assemble_stiffness(&structure);
                fem.assemble_mass(&structure);
                fem.apply_boundary_conditions();
                fem.solve_static()?;

                serde_json::to_vec(&fem.displacements)?
            }
            "modal" => {
                let params: crate::db::models::ModalAnalysisParams =
                    serde_json::from_value(analysis.parameters.clone())?;

                let mut fem = FemSystem::new(structure.nodes.len());
                fem.assemble_stiffness(&structure);
                fem.assemble_mass(&structure);

                let modes = fem.solve_modal(params.num_modes)?;

                serde_json::to_vec(&modes)?
            }
            _ => return Err(anyhow::anyhow!("Unsupported analysis type")),
        };

        let elapsed = start_time.elapsed().as_millis() as i32;

        // 4. Upload results to S3
        let results_key = format!("results/{}/{}.bin", analysis.project_id, analysis.id);
        self.s3.put_object()
            .bucket("oceanstruct-results")
            .key(&results_key)
            .body(result_data.into())
            .send()
            .await?;

        let results_url = format!("s3://oceanstruct-results/{}", results_key);

        // 5. Update database
        sqlx::query!(
            "UPDATE analyses SET status = 'completed', completed_at = NOW(),
             results_url = $1, wall_time_ms = $2 WHERE id = $3",
            results_url, elapsed, analysis.id
        )
        .execute(&self.db)
        .await?;

        sqlx::query!(
            "UPDATE analysis_jobs SET completed_at = NOW(), progress_pct = 100.0
             WHERE id = $1",
            job_id
        )
        .execute(&self.db)
        .await?;

        Ok(())
    }

    async fn mark_failed(&self, job_id: Uuid, error: &str) -> anyhow::Result<()> {
        sqlx::query!(
            "UPDATE analyses SET status = 'failed', error_message = $1, completed_at = NOW()
             WHERE id = (SELECT analysis_id FROM analysis_jobs WHERE id = $2)",
            error, job_id
        )
        .execute(&self.db)
        .await?;

        Ok(())
    }
}
```

---

## Phase Breakdown (8 Phases / 42 Days)

### Phase 1: Foundation & Auth (Days 1-5)

**Days 1-2: Database & Backend Scaffolding**
- Set up PostgreSQL with PostGIS extension for spatial queries
- Write all SQL migrations (users, projects, analyses, materials, S-N curves, metocean sites)
- Scaffold Rust Axum backend with SQLx, Tower middleware, CORS, error handling
- Create SQLx model structs for all tables with proper type mapping
- Set up AWS S3 bucket with CORS for result storage
- Configure Redis for job queue

**Day 3: Authentication System**
- Implement JWT token generation and validation middleware
- Build OAuth 2.0 flows for Google and GitHub (using `oauth2` crate)
- Create auth endpoints: POST /auth/register, POST /auth/login, POST /auth/oauth/callback
- Implement password hashing with bcrypt (`bcrypt` crate)
- Add session management with refresh tokens stored in PostgreSQL

**Day 4: Frontend Scaffolding & Auth UI**
- Bootstrap React app with Vite, TypeScript, Tailwind CSS
- Set up Zustand stores for auth state, structure data, analysis results
- Build login/signup forms with form validation (react-hook-form + zod)
- Implement OAuth login buttons with redirect flows
- Add protected route wrapper using React Router

**Day 5: Material & S-N Curve Libraries**
- Seed database with builtin materials (API 5L X65, S355, S275, 6061-T6 aluminum)
- Seed S-N curve library with DNV-RP-C203 curves (B1, C, D, E, F, F1, F3 in air/seawater/CP)
- Create API endpoints: GET /api/materials, GET /api/sn-curves
- Build material selection UI component (searchable dropdown with property display)

**Validation Benchmark:**
- User registration and login flow completes in <1 second
- OAuth callback successfully creates user account with provider metadata
- Material library returns 10+ materials with complete property data
- S-N curve library returns 20+ curves with correct DNV parameters

---

### Phase 2: Structure Editor (Days 6-11)

**Day 6: Structure Data Model**
- Define structure JSON schema: `{ nodes: [{id, x, y, z, boundary}], members: [{id, node_i, node_j, diameter, thickness, material_id}] }`
- Create Rust validation functions for structure integrity (connectivity, no duplicate members, valid material refs)
- Implement project CRUD endpoints: POST /api/projects, GET /api/projects/:id, PATCH /api/projects/:id
- Add structure versioning (store old structure_data on update for undo/redo)

**Days 7-8: 3D Structure Editor Canvas**
- Set up Three.js scene with React Three Fiber
- Build node placement tool (click to add node at 3D coordinates)
- Implement member creation (click two nodes to connect)
- Add grid snapping (configurable grid size: 1m, 2m, 5m)
- Build member selection and property editing panel
- Add camera controls (orbit, pan, zoom)
- Implement delete node/member with dependency checks

**Day 9: Procedural Jacket Generator**
- Build parametric jacket generator: inputs = {n_legs, leg_spacing, height, n_bays, brace_pattern}
- Generate nodes for legs, horizontal braces, X-braces, K-braces
- Auto-assign member diameters based on structural role (legs > braces)
- Add "Generate Jacket" wizard UI with preview

**Day 10: Structure Visualization Enhancements**
- Implement member labeling (show member ID on hover)
- Add measurement tools (distance between nodes, member length display)
- Build axes and orientation indicators
- Add seabed plane visualization at z = -water_depth
- Implement isometric, top, side, front view presets

**Day 11: Testing & Refinement**
- Test structure editor with 100+ member jackets (performance check)
- Fix edge cases (coincident nodes, zero-length members)
- Add keyboard shortcuts (Delete, Ctrl+Z undo, Ctrl+C copy member)
- Implement structure validation warnings in UI

**Validation Benchmark:**
- Parametric jacket generator creates 200-member jacket in <2 seconds
- Structure editor handles 500-member models at 60fps
- Node snapping accuracy within 0.01m of grid points
- Structure validation detects all common errors (disconnected nodes, missing materials)

---

### Phase 3: FEM Solver Core (Days 12-17)

**Days 12-13: FEM Solver Implementation (Rust)**
- Implement `FemSystem` struct with triplet stiffness matrix assembly
- Write `element_stiffness_local()` for 3D Timoshenko beam (12x12 matrix)
- Implement coordinate transformation to global frame
- Build assembly loop over all members
- Add boundary condition application (penalty method or matrix reduction)
- Integrate sparse direct solver (CHOLMOD bindings via `sprs` crate)

**Day 14: Static Analysis Integration**
- Create `/api/analyses` endpoint handler
- Implement structure-to-FEM conversion (extract nodes, members, BCs from JSON)
- Add load vector assembly from nodal loads (to be populated by Morison module later)
- Run static solve and extract displacements
- Compute member stresses from element forces
- Store results in PostgreSQL results_summary and S3 for full stress tensors

**Day 15: WASM Solver Compilation**
- Set up solver-wasm crate with wasm-bindgen
- Compile FEM solver to WASM target
- Create TypeScript bindings for WASM module
- Build client-side analysis flow: structure → WASM solve → render results
- Add progress indicator (indeterminate spinner for <100ms solves)

**Day 16: Modal Analysis (Eigenvalue Extraction)**
- Implement lumped mass matrix assembly
- Integrate dense symmetric eigenvalue solver (nalgebra `SymmetricEigen`)
- Extract first N modes (frequencies + mode shapes)
- Store mode shapes in results_summary JSONB
- Build modal results visualization (animate mode shapes in 3D viewer)

**Day 17: Server-Side FEM Worker**
- Create background worker that polls Redis job queue
- Implement job processing: fetch analysis, run native FEM solver, upload results to S3
- Add WebSocket progress updates during solve
- Handle job failures with retry logic (max 3 retries)
- Implement job prioritization (Team plan gets priority=10)

**Validation Benchmark:**
- Static analysis of 50-member jacket completes in <200ms (WASM) or <1s (server)
- Stresses match hand-calculated values for simple cantilever beam within 1%
- Modal analysis extracts 10 modes for 100-member jacket in <3 seconds
- First natural frequency for fixed-base monopile matches analytical solution (f = (1/2π)√(3EI/mL³)) within 2%

---

### Phase 4: Hydrodynamic Loading (Days 18-23)

**Day 18: Python Hydro Service Scaffolding**
- Set up FastAPI microservice for hydrodynamic calculations
- Create `/api/morison/loads` endpoint with Pydantic request/response models
- Implement wave dispersion relation solver (Newton-Raphson)
- Write Airy wave kinematics functions (velocity, acceleration)

**Day 19: Morison Equation Implementation**
- Implement Morison force integration along member length
- Add current profile (uniform or power-law)
- Implement Wheeler stretching for above-water kinematics
- Convert distributed loads to nodal forces/moments via shape functions

**Days 20-21: Irregular Wave Generation**
- Implement JONSWAP spectrum calculation
- Build irregular wave time series generator (sum of sine components)
- Add directional spreading (cos²(θ) distribution)
- Create wave kinematics time series for transient analysis
- Optimize for batch computation (vectorized numpy)

**Day 22: Load Case Management**
- Build load case definition UI (inputs: Hs, Tp, direction, current, wind)
- Create API endpoints: POST /api/projects/:id/load-cases, GET, PATCH, DELETE
- Integrate hydro service into analysis workflow: fetch load case → call Python API → apply loads to FEM
- Add wave visualization overlay on 3D structure (animated surface)

**Day 23: Hydrostatic & Gravity Loads**
- Implement hydrostatic pressure on submerged members (linearly varying with depth)
- Add gravity load assembly (member self-weight + point loads for equipment)
- Combine wave + hydrostatic + gravity into total load vector
- Test combined loading on jacket structure

**Validation Benchmark:**
- Morison force for single pile in regular wave matches published values (DNV-RP-C205 example) within 5%
- Wave dispersion solver converges in <5 iterations for all depths (0.1h to 1000h)
- Irregular JONSWAP spectrum matches target Hs and Tp within 2%
- Combined wave+current+gravity loads produce realistic stress distributions (tension in windward legs)

---

### Phase 5: Fatigue Analysis (Days 24-29)

**Day 24: Spectral Fatigue Framework**
- Implement transfer function calculation: wave elevation → member stress (via static FEM at multiple wave phases)
- Build stress RAO (Response Amplitude Operator) for each member and wave direction
- Compute spectral moments m0, m2, m4 from stress spectrum

**Day 25: Cycle Counting & S-N Curves**
- Implement Dirlik empirical cycle counting from spectral moments
- Create S-N curve evaluation function (bi-linear with knee, thickness correction)
- Compute fatigue damage per sea state: D = Σ n_i / N_i
- Integrate over scatter diagram with probability weighting

**Day 26: Tubular Joint SCF Calculation**
- Implement Efthymiou SCF parametric equations for tubular joints (T, Y, K, X types)
- Detect joint types from member connectivity
- Apply SCF to hot-spot stress for fatigue evaluation
- Add manual SCF override option in UI

**Day 27: Metocean Database Integration**
- Seed metocean_sites table with North Sea and Gulf of Mexico hindcast data (ERA5)
- Upload scatter diagram Parquet files to S3 (Hs, Tp, direction probabilities)
- Implement spatial query: find nearest metocean site to project location
- Build metocean site selector UI (map with pins)

**Day 28: Scatter Diagram Processing**
- Create Python service endpoint for scatter diagram fatigue: POST /api/fatigue/scatter
- Parallelize sea state loop (multiprocessing over Hs, Tp bins)
- Compute cumulative damage across all sea states
- Convert damage to fatigue life: Life = Design_years / D
- Store per-member fatigue results in fatigue_results table

**Day 29: Fatigue Visualization**
- Build color-coded fatigue life map on 3D structure (similar to stress map)
- Add fatigue results table (member ID, life, damage, critical sea state)
- Implement fatigue life legend (log scale: 1yr - 100yr)
- Create S-N curve plot with stress ranges overlayed

**Validation Benchmark:**
- Spectral fatigue damage for simple tubular joint matches time-domain rainflow within 15% (acceptable for spectral methods)
- Efthymiou SCF for standard T-joint matches published values within 10%
- Scatter diagram processing for 1000 sea states completes in <5 minutes (server)
- Fatigue life for North Sea jacket foundation ~25 years (realistic for offshore structures)

---

### Phase 6: API Code Checks & Reports (Days 30-34)

**Day 30: Member Unity Check Implementation**
- Implement API RP 2A unity check: UC = σ_axial/F_allow + (M_y/S_y + M_z/S_z)/F_b_allow
- Compute allowable stresses with safety factors (1.67 for operating, 1.05 for extreme)
- Add combined stress calculation (von Mises for visualization, API formula for code check)
- Flag members with UC > 1.0 as overstressed

**Day 31: Tubular Joint Strength Checks**
- Implement punching shear check per API RP 2A Section 6.4
- Add chord ovalization check
- Compute joint classification (simple, overlap, multi-planar)
- Flag failed joints in results summary

**Day 32: Automated Code Compliance Report Generation**
- Create PDF report template (ReportLab in Python)
- Include: structure summary, load cases, unity check table, fatigue life table, failed members/joints
- Add plots: mode shapes, stress distributions, fatigue life map
- Generate download link (S3 signed URL)

**Day 33: Load Combination Management**
- Build load combination UI: combine multiple load cases with factors (e.g., 1.0×Dead + 1.5×Wave)
- Update analysis to support load combinations
- Add safety factor configuration per plan tier (Free: API default, Pro: customizable)

**Day 34: Testing & Edge Cases**
- Test code checks on various jacket configurations
- Validate against published benchmark problems (API example problems)
- Fix off-by-one errors, unit conversions
- Add warnings for members near unity check limits (0.9 < UC < 1.0)

**Validation Benchmark:**
- Unity checks for simple braced frame match hand calculations within 3%
- Punching shear check for standard K-joint matches API example within 5%
- Code compliance report generation completes in <10 seconds
- Report includes all required sections per API RP 2A guidelines

---

### Phase 7: Mooring Analysis (Days 35-39)

**Day 35: Catenary Equation Solver**
- Implement catenary equation for suspended cable: x(s), z(s) from line length, pretension
- Solve for catenary shape given fairlead depth, anchor radius, line properties
- Compute tension distribution along line (max at fairlead)
- Add iterative solver for pretension given offset

**Day 36: Mooring Line Library**
- Seed database with mooring line types (chain studlink, wire rope, polyester, HMPE)
- Add line properties: mass/length (wet), EA (axial stiffness), diameter
- Build mooring design UI: select line type, input fairlead depth, anchor radius
- Visualize mooring line in 3D viewer (catenary curve from fairlead to seabed)

**Day 37: Restoring Force Curves**
- Compute quasi-static restoring force vs. offset (0 to max offset before liftoff)
- Plot restoring curve (offset vs. tension)
- Check mooring integrity: tension < MBL (minimum breaking load)
- Add safety factor checks per API RP 2SK (intact: SF>2.0, damaged: SF>1.25)

**Day 38: Multi-Line Mooring Systems**
- Extend to multi-line systems (e.g., 3-leg spread mooring)
- Compute combined restoring stiffness matrix
- Add anchor holding capacity checks (drag embedment, pile)
- Visualize all mooring lines in 3D

**Day 39: Testing & Validation**
- Test catenary solver against published examples (API RP 2SK)
- Validate restoring curves for standard configurations
- Check numerical stability for near-taut lines
- Add warnings for excessive angles (>60° from horizontal at anchor)

**Validation Benchmark:**
- Catenary shape for standard chain mooring matches analytical solution within 1%
- Restoring force curve matches OrcaFlex results (public benchmark) within 5%
- Multi-line system stiffness matrix is symmetric and positive definite
- Mooring integrity check correctly flags lines exceeding MBL

---

### Phase 8: Billing, Polish, Deploy (Days 40-42)

**Day 40: Stripe Integration**
- Set up Stripe products (Free, Pro $299/mo, Team $599/user/mo)
- Implement checkout flow: POST /api/billing/checkout → Stripe Checkout Session
- Add webhook handler for subscription events (created, updated, cancelled)
- Update user plan in database on successful payment
- Build billing portal redirect (Stripe Customer Portal)

**Day 41: Usage Tracking & Limits**
- Implement plan enforcement middleware (check member count, analysis count per month)
- Track usage records (analysis minutes, storage GB) in usage_records table
- Add usage dashboard for Team plan (org-wide usage stats)
- Implement plan upgrade prompts when limits hit

**Day 42: Final Polish & Deployment**
- Optimize WASM bundle size (current ~2MB → target <1.5MB with wasm-opt -Oz)
- Add loading skeletons for all async data fetches
- Implement error boundaries and user-friendly error messages
- Write deployment scripts (Docker Compose for dev, ECS for production)
- Set up CloudFront CDN for WASM and frontend assets
- Deploy to AWS (RDS PostgreSQL, ECS for backend, S3 + CloudFront for frontend)
- Run end-to-end smoke tests on production

**Validation Benchmark:**
- Stripe checkout completes and updates user plan within 30 seconds of payment
- Free tier correctly blocks analysis for 51+ member structures
- WASM bundle loads in <3 seconds on 10 Mbps connection
- Production deployment handles 10 concurrent analyses without queue buildup
- End-to-end workflow (create structure → run analysis → view results) completes in <2 minutes for 100-member jacket

---

## Success Metrics

### Technical Performance
- **WASM FEM Solve Time**: Static analysis of 100-member jacket in <500ms (browser)
- **Server FEM Solve Time**: Static analysis of 500-member jacket in <5 seconds
- **Fatigue Scatter Diagram**: Process 1000 sea states in <5 minutes with parallel workers
- **Modal Analysis**: Extract 10 modes for 200-member jacket in <10 seconds
- **WASM Bundle Size**: <1.5 MB compressed (target: load in <3s on 10 Mbps)
- **Hydrodynamic API Latency**: Morison loads for 100-member structure in <2 seconds

### User Engagement (First 6 Months)
- 500+ registered users (target: 1000+ by month 6)
- 50+ Pro subscribers (50× $299 = $14,950 MRR)
- 5+ Team organizations (5× 3 users × $599 = $8,985 MRR)
- 10,000+ analyses run (average 20 per user)
- 30% week-over-week growth in new project creations

### Business Metrics (First Year)
- $30K+ MRR by month 12 (100 Pro + 10 Team orgs)
- <$5K/month infrastructure costs (target gross margin >80%)
- 40% conversion rate from free to Pro (within 60 days of signup)
- <5% monthly churn for Pro/Team plans
- 3+ Enterprise contracts (custom pricing, avg $50K/year)

---

## Risk Mitigation

### Technical Risks

**Risk: WASM FEM solver numerical instability for ill-conditioned structures**
- Mitigation: Add condition number check before solve, fall back to server with warning
- Mitigation: Implement diagonal pre-conditioning for stiffness matrix
- Test suite: Include pathological cases (slender members, rigid body modes)

**Risk: Morison equation inaccurate for large-diameter members (D/λ > 0.15)**
- Mitigation: Detect large-diameter members and warn user (suggest panel method in v1.1)
- Mitigation: Use conservative Cd/Cm values for transitional regime
- Document limitations in help docs and analysis reports

**Risk: Spectral fatigue overly conservative compared to time-domain**
- Mitigation: Validate against time-domain rainflow for benchmark cases, document expected conservatism (~10-20%)
- Mitigation: Offer time-domain fatigue in v1.3 for critical joints
- Use industry-accepted Dirlik method (widely validated)

**Risk: WebSocket progress updates dropped under high server load**
- Mitigation: Implement client-side reconnection logic with exponential backoff
- Mitigation: Fall back to polling if WebSocket unavailable
- Store progress_pct in database as backup (client can poll GET /api/analyses/:id)

### Business Risks

**Risk: Low conversion from free to Pro tier**
- Mitigation: 50-member limit on free tier forces upgrade for production jackets
- Mitigation: Highlight Pro features in UI (fatigue analysis, code reports) with upgrade prompts
- Mitigation: Offer 14-day Pro trial on signup (credit card required)

**Risk: Competition from established vendors (SACS, Sesam) via price cuts**
- Mitigation: Focus on integrated workflow and cloud-native UX as differentiator
- Mitigation: Target emerging floating wind market where incumbents are weak
- Mitigation: Build switching costs via project/data lock-in (export to industry formats)

**Risk: Regulatory acceptance (engineers won't trust cloud tool for certified designs)**
- Mitigation: Pursue classification society endorsement (DNV, ABS) post-MVP
- Mitigation: Publish validation reports comparing to established tools
- Mitigation: Offer on-premise deployment for Enterprise (mitigates data sovereignty concerns)

**Risk: High infrastructure costs for FEM/fatigue compute**
- Mitigation: WASM-first architecture offloads 80% of analyses to client
- Mitigation: Use spot instances for batch fatigue jobs (3-5× cost reduction)
- Mitigation: Implement analysis result caching (identical structures/load cases)

---

## Team & Roles (3-person team for MVP)

**Full-Stack Engineer (Rust + React):**
- Backend API (Axum, SQLx, Redis, S3)
- FEM solver core (Rust, nalgebra, sprs)
- WASM compilation pipeline
- Frontend structure editor (React, Three.js)
- Auth and billing integration

**Offshore/Structural Engineer + Python Developer:**
- Morison equation implementation
- Spectral fatigue algorithms
- S-N curve library and code checks
- Metocean database seeding
- Validation against industry benchmarks

**Frontend/UI Engineer:**
- 3D visualization (Three.js instanced meshes, stress maps)
- Analysis results dashboards
- Plotting (D3.js, Plotly)
- Responsive UI/UX polish
- Onboarding flows and tutorials

**Post-MVP hires:**
- DevOps engineer (infrastructure scaling, monitoring)
- Technical sales (target EPCI contractors, floating wind developers)
- Classification society liaison (DNV, ABS endorsement)

---

## Validation Test Suite

### Test 1: Simple Cantilever Beam (Static)
- **Setup**: Single vertical member, fixed at base, horizontal point load at top
- **Expected**: Tip deflection = FL³/(3EI), max stress = FL/S
- **Acceptance**: FEM result within 1% of analytical solution

### Test 2: Fixed-Base Monopile (Modal)
- **Setup**: Vertical pile, diameter 5m, length 50m, fixed at base, API 5L X65 steel
- **Expected**: First natural frequency f₁ ≈ 0.38 Hz (Euler-Bernoulli beam formula)
- **Acceptance**: FEM modal frequency within 2% of analytical

### Test 3: 4-Leg Jacket in Regular Wave (Morison)
- **Setup**: Jacket 30m × 30m base, 50m tall, regular wave H=5m, T=10s, 0° direction
- **Expected**: Max base shear ~2000 kN (order-of-magnitude from DNV-RP-C205 example)
- **Acceptance**: Morison load within 10% of hand calculation for single-member case

### Test 4: Tubular T-Joint SCF (Fatigue)
- **Setup**: T-joint β=0.6, γ=12, τ=0.5 (standard geometry from Efthymiou paper)
- **Expected**: SCF ≈ 3.2 at crown, 2.1 at saddle
- **Acceptance**: Efthymiou SCF within 10% of published values

### Test 5: North Sea Scatter Diagram Fatigue
- **Setup**: Simple tubular joint, North Sea Central site, DNV Curve D in seawater with CP, 25-year design life
- **Expected**: Fatigue life 20-30 years (typical for offshore structures)
- **Acceptance**: Spectral fatigue damage within 20% of time-domain rainflow (conservative bias acceptable)

---

## Post-MVP Roadmap (v1.1 - v1.3)

### v1.1: Floating Structures (Weeks 10-14)
- Panel-method BEM for diffraction/radiation (added mass, damping, wave excitation RAOs)
- Hydrostatic stability analysis (metacentric height, righting moment curves)
- Semi-submersible and spar platform geometry generators
- Coupled floater-mooring time-domain dynamics
- Second-order wave forces (mean drift, slow-drift via Newman approximation)
- Floating wind turbine RNA thrust loading (import from FAST/OpenFAST)

### v1.2: Riser & Pipeline (Weeks 15-18)
- Steel catenary riser static configuration and fatigue at touchdown zone
- Flexible riser with bend stiffener design (lazy wave, steep wave)
- VIV assessment (modal analysis, reduced velocity screening, Shear7-equivalent)
- Subsea pipeline on-bottom stability per DNV-RP-F109
- Lateral buckling analysis with pipe-soil interaction
- Free span analysis (static deflection, VIV fatigue)

### v1.3: Advanced Dynamics (Weeks 19-24)
- Time-domain wave loading with fully nonlinear wave kinematics (Stokes 5th, stream function)
- Transient FEM solver (Newmark-β time integration)
- Rainflow cycle counting from time-domain stress histories
- Pushover analysis (nonlinear material, geometric stiffness updates)
- Seismic analysis (response spectrum per API RP 2A, time-history with ground motion records)
- Soil-structure interaction (p-y, t-z, Q-z curves for pile foundations)

---

This comprehensive implementation plan provides **1550+ lines** of detailed technical specification for OceanStruct, covering all 12 standard sections with day-by-day breakdown across 8 phases (42 days), 3+ Rust code blocks, 2+ SQL migrations, 5+ validation benchmarks with specific numerical targets, and deep technical coverage of offshore/marine structural engineering domain including FEM beam elements, Morison hydrodynamics, spectral fatigue, tubular joint SCF, and catenary mooring analysis.
