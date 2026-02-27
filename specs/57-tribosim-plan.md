# 57. TriboSim — Tribology and Wear Simulation Platform

## Implementation Plan

**MVP Scope:** Browser-based tribology workbench with drag-and-drop geometry builder for 2D contact scenarios (sphere-on-flat, cylinder-on-flat, arbitrary profiles via SVG import) rendered in WebGL, custom contact mechanics solver implementing Hertzian analytical solution and FFT-based semi-analytical method (SAM) for non-Hertzian contact compiled to WebAssembly for client-side execution of problems ≤50,000 surface nodes and server-side Rust-native execution for larger problems, elastohydrodynamic lubrication (EHL) solver coupling Reynolds equation with elastic deformation via multigrid iteration, Archard wear model with material-pair database (50 pairs: steel/steel, steel/bronze, ceramic/polymer, etc.), rolling bearing life calculator implementing ISO 281 L10 and aISO methods with contamination and misalignment factors, interactive visualization of contact pressure distribution, film thickness maps, and wear depth profiles rendered via WebGL with pan/zoom and color-mapped isolines, lubricant database with 100 oils (viscosity-temperature curves, pressure-viscosity coefficients), PostgreSQL storage for projects/materials/lubricants with S3 for surface measurement data and simulation results, STEP geometry import for 3D bearing models, Stripe billing with three tiers (Free / Pro $129/mo / Advanced $299/user/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| Contact Solver | Rust (native + WASM) | FFT-based SAM for contact pressure, conjugate gradient for elastic deformation |
| EHL Solver | Rust (server-side) | Multigrid Reynolds solver coupled with contact mechanics |
| Wear Simulation | Rust (server-side) | Surface evolution, adaptive remeshing for long-term wear |
| WASM Compilation | `wasm-pack` + `wasm-bindgen` | Client-side Hertzian + simple EHL for ≤50K nodes |
| Scientific Computing | Python 3.12 (FastAPI) | Surface roughness analysis, FFT preprocessing, bearing catalog search |
| Database | PostgreSQL 16 | Projects, users, materials, lubricants, bearing catalog |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | Surface profilometry data, EHL result fields, wear progression snapshots |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| Geometry Editor | Custom WebGL renderer | 2D profile editor with Bézier curves, SVG import, snap-to-grid |
| Contact Visualization | WebGL 2.0 (custom) | GPU-accelerated pressure maps, 3D surface deformation rendering |
| 3D Bearing Viewer | Three.js | STEP file import, bearing component assembly visualization |
| Real-time | WebSocket (Axum) | Live simulation progress, EHL convergence monitoring |
| Job Queue | Redis 7 + Tokio tasks | Server-side simulation job management, parametric sweep distribution |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | WASM bundle delivery, static frontend assets |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, error tracking |
| CI/CD | GitHub Actions | WASM build pipeline, Rust tests, Docker image push |

### Key Architecture Decisions

1. **Hybrid client/server solver with WASM threshold at 50K surface nodes**: Contact problems with ≤50,000 surface mesh nodes (covers 90%+ of 2D and simple 3D contacts like single bearing rollers) run entirely in the browser via WASM, providing instant feedback with zero server cost. Hertzian analytical solutions and basic EHL film thickness formulas execute in <1 second. Larger problems (full bearing assemblies, rough surface contact, thermal EHL) are submitted to the Rust-native server solver which handles multi-million-node FFT convolutions and iterative EHL solutions. The threshold is configurable per plan tier.

2. **FFT-based Semi-Analytical Method (SAM) for contact mechanics**: The SAM approach uses FFT convolution to compute elastic deformation from pressure distribution, avoiding full finite element analysis while maintaining high accuracy for smooth and moderately rough surfaces. This reduces a 50K-node contact problem from minutes (FEA) to seconds (FFT). The frequency-domain influence coefficients are precomputed and cached. For elastic-plastic contact and severely rough surfaces, we fall back to simplified models (Greenwood-Williamson for rough surfaces) or queue server-side FEA when needed.

3. **Multigrid solver for Reynolds equation in EHL**: Elastohydrodynamic lubrication couples the Reynolds equation (pressure field in fluid film) with elastic deformation (solid mechanics) in a highly nonlinear system. We use a Full Approximation Scheme (FAS) multigrid solver for the Reynolds equation, which converges in 10-30 V-cycles for typical EHL problems, dramatically faster than relaxation methods. The coupling loop alternates between Reynolds solve and elastic deformation update via FFT until convergence (typically 5-15 outer iterations).

4. **WebGL for contact pressure and film thickness visualization**: Contact problems produce 2D scalar fields (pressure, film thickness, wear depth) that must be visualized with high dynamic range and smooth color gradients. We use WebGL fragment shaders with bilinear texture sampling and custom color maps (Viridis, Turbo, isolines) to render these fields at 60fps even for 100K+ node results. Pan/zoom uses GPU matrix transforms. This is decoupled from the geometry editor to allow split-screen layout (geometry on left, results on right).

5. **Material-pair database with wear coefficients from literature**: Wear prediction requires empirical coefficients (Archard's k, oxidative wear transition temperatures, fretting wear maps) that vary by material pair, surface finish, and lubrication regime. We curate a database of 50+ material pairs with coefficients extracted from published papers (ASM handbooks, Tribology International, Wear journal). Each entry includes reference DOI, test conditions (load, speed, temperature), and uncertainty bounds. Users can add custom pairs with Admin approval.

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";  -- For fuzzy text search on materials/lubricants

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

-- Organizations (for Advanced plan team collaboration)
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

-- Projects (contact scenario + analysis workspace)
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    project_type TEXT NOT NULL,  -- contact | bearing | gear | seal | wear_simulation
    geometry_data JSONB NOT NULL DEFAULT '{}',  -- Geometry definition (profiles, dimensions)
    analysis_settings JSONB DEFAULT '{}',  -- Load, speed, temperature, lubricant selection
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

-- Simulation Runs
CREATE TABLE simulations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    simulation_type TEXT NOT NULL,  -- hertzian | sam_contact | ehl | wear_evolution | bearing_life
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | running | completed | failed | cancelled
    execution_mode TEXT NOT NULL DEFAULT 'wasm',  -- wasm | server
    input_params JSONB NOT NULL DEFAULT '{}',  -- Load, speed, material properties, mesh density
    node_count INTEGER NOT NULL DEFAULT 0,
    results_url TEXT,  -- S3 URL for result data (pressure field, film thickness, etc.)
    results_summary JSONB,  -- Quick-access summary (max pressure, min film thickness, life estimate)
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
    priority INTEGER DEFAULT 0,  -- Higher = more urgent
    progress_pct REAL DEFAULT 0.0,
    convergence_data JSONB DEFAULT '[]',  -- [{iteration, residual, pressure_max}]
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX jobs_sim_idx ON simulation_jobs(simulation_id);
CREATE INDEX jobs_worker_idx ON simulation_jobs(worker_id);

-- Materials Database
CREATE TABLE materials (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,  -- e.g., "AISI 52100 bearing steel"
    category TEXT NOT NULL,  -- metal | ceramic | polymer | composite
    elastic_modulus REAL NOT NULL,  -- GPa
    poisson_ratio REAL NOT NULL,
    yield_strength REAL,  -- MPa (NULL for ceramics without clear yield)
    hardness_hv REAL,  -- Vickers hardness
    density REAL,  -- kg/m³
    thermal_conductivity REAL,  -- W/(m·K)
    specific_heat REAL,  -- J/(kg·K)
    description TEXT,
    reference_url TEXT,  -- Datasheet or paper DOI
    is_builtin BOOLEAN DEFAULT true,
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX materials_category_idx ON materials(category);
CREATE INDEX materials_name_trgm_idx ON materials USING gin(name gin_trgm_ops);

-- Material Pairs (wear coefficients)
CREATE TABLE material_pairs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    material_1_id UUID NOT NULL REFERENCES materials(id) ON DELETE CASCADE,
    material_2_id UUID NOT NULL REFERENCES materials(id) ON DELETE CASCADE,
    lubrication_regime TEXT NOT NULL,  -- dry | boundary | mixed | full_film
    archard_k REAL,  -- Archard wear coefficient (dimensionless)
    friction_coefficient REAL,
    temperature_c REAL DEFAULT 25.0,  -- Test temperature
    normal_load_n REAL,  -- Test load (for reference)
    sliding_speed_ms REAL,  -- Test speed (for reference)
    reference_doi TEXT,  -- Paper reference
    notes TEXT,
    is_verified BOOLEAN DEFAULT false,
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(material_1_id, material_2_id, lubrication_regime)
);
CREATE INDEX material_pairs_mat1_idx ON material_pairs(material_1_id);
CREATE INDEX material_pairs_mat2_idx ON material_pairs(material_2_id);

-- Lubricants Database
CREATE TABLE lubricants (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,  -- e.g., "ISO VG 68 mineral oil"
    base_type TEXT NOT NULL,  -- mineral | synthetic | bio
    viscosity_grade TEXT,  -- ISO VG designation
    kinematic_viscosity_40c REAL,  -- mm²/s
    kinematic_viscosity_100c REAL,  -- mm²/s
    viscosity_index INTEGER,
    pressure_viscosity_coeff REAL,  -- GPa⁻¹ (Barus/Roelands α)
    density_15c REAL,  -- kg/m³
    thermal_conductivity REAL,  -- W/(m·K)
    specific_heat REAL,  -- J/(kg·K)
    pour_point_c REAL,
    flash_point_c REAL,
    description TEXT,
    datasheet_url TEXT,
    is_builtin BOOLEAN DEFAULT true,
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX lubricants_grade_idx ON lubricants(viscosity_grade);
CREATE INDEX lubricants_name_trgm_idx ON lubricants USING gin(name gin_trgm_ops);

-- Bearing Catalog (standard rolling bearings)
CREATE TABLE bearings (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    manufacturer TEXT,  -- SKF, NSK, Timken, etc.
    designation TEXT NOT NULL,  -- e.g., "6205"
    bearing_type TEXT NOT NULL,  -- radial_ball | angular_contact | cylindrical_roller | tapered_roller | spherical_roller | needle
    bore_diameter REAL NOT NULL,  -- mm
    outer_diameter REAL NOT NULL,  -- mm
    width REAL NOT NULL,  -- mm
    dynamic_load_rating REAL,  -- C (kN)
    static_load_rating REAL,  -- C₀ (kN)
    limiting_speed_oil INTEGER,  -- rpm
    limiting_speed_grease INTEGER,  -- rpm
    mass_kg REAL,
    step_model_url TEXT,  -- S3 URL to STEP file
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX bearings_designation_idx ON bearings(designation);
CREATE INDEX bearings_type_idx ON bearings(bearing_type);
CREATE INDEX bearings_bore_idx ON bearings(bore_diameter);

-- Surface Measurement Data (for rough surface contact)
CREATE TABLE surface_measurements (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    measurement_type TEXT NOT NULL,  -- profilometer | afm | optical
    data_url TEXT NOT NULL,  -- S3 URL to raw measurement file (CSV/HDF5)
    resolution_um REAL,  -- Lateral resolution
    extent_x_um REAL,  -- Scan size X
    extent_y_um REAL,  -- Scan size Y (NULL for 1D profiles)
    roughness_ra_um REAL,
    roughness_rq_um REAL,
    roughness_rsk REAL,
    roughness_rku REAL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX surface_project_idx ON surface_measurements(project_id);

-- Usage Tracking (for billing enforcement)
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,  -- simulation_hours | storage_gb | ehl_solves
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
    pub project_type: String,
    pub geometry_data: serde_json::Value,
    pub analysis_settings: serde_json::Value,
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
    pub input_params: serde_json::Value,
    pub node_count: i32,
    pub results_url: Option<String>,
    pub results_summary: Option<serde_json::Value>,
    pub error_message: Option<String>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub wall_time_ms: Option<i32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Material {
    pub id: Uuid,
    pub name: String,
    pub category: String,
    pub elastic_modulus: f32,  // GPa
    pub poisson_ratio: f32,
    pub yield_strength: Option<f32>,  // MPa
    pub hardness_hv: Option<f32>,
    pub density: Option<f32>,
    pub thermal_conductivity: Option<f32>,
    pub specific_heat: Option<f32>,
    pub description: Option<String>,
    pub reference_url: Option<String>,
    pub is_builtin: bool,
    pub created_by: Option<Uuid>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct MaterialPair {
    pub id: Uuid,
    pub material_1_id: Uuid,
    pub material_2_id: Uuid,
    pub lubrication_regime: String,
    pub archard_k: Option<f32>,
    pub friction_coefficient: Option<f32>,
    pub temperature_c: f32,
    pub normal_load_n: Option<f32>,
    pub sliding_speed_ms: Option<f32>,
    pub reference_doi: Option<String>,
    pub notes: Option<String>,
    pub is_verified: bool,
    pub created_by: Option<Uuid>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Lubricant {
    pub id: Uuid,
    pub name: String,
    pub base_type: String,
    pub viscosity_grade: Option<String>,
    pub kinematic_viscosity_40c: Option<f32>,
    pub kinematic_viscosity_100c: Option<f32>,
    pub viscosity_index: Option<i32>,
    pub pressure_viscosity_coeff: Option<f32>,  // GPa⁻¹
    pub density_15c: Option<f32>,
    pub thermal_conductivity: Option<f32>,
    pub specific_heat: Option<f32>,
    pub pour_point_c: Option<f32>,
    pub flash_point_c: Option<f32>,
    pub description: Option<String>,
    pub datasheet_url: Option<String>,
    pub is_builtin: bool,
    pub created_by: Option<Uuid>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Bearing {
    pub id: Uuid,
    pub manufacturer: Option<String>,
    pub designation: String,
    pub bearing_type: String,
    pub bore_diameter: f32,
    pub outer_diameter: f32,
    pub width: f32,
    pub dynamic_load_rating: Option<f32>,  // kN
    pub static_load_rating: Option<f32>,  // kN
    pub limiting_speed_oil: Option<i32>,
    pub limiting_speed_grease: Option<i32>,
    pub mass_kg: Option<f32>,
    pub step_model_url: Option<String>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct ContactSimParams {
    pub geometry: GeometryParams,
    pub material_1_id: Uuid,
    pub material_2_id: Uuid,
    pub normal_load_n: f64,  // Normal load
    pub tangential_load_n: Option<f64>,
    pub mesh_density: MeshDensity,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct GeometryParams {
    pub contact_type: ContactType,
    pub radius_1: Option<f64>,  // mm (for sphere/cylinder)
    pub radius_2: Option<f64>,  // mm (for sphere/cylinder)
    pub profile_data: Option<Vec<(f64, f64)>>,  // Custom profile [(x, y)]
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum ContactType {
    SphereOnFlat,
    CylinderOnFlat,
    SphereOnSphere,
    CylinderOnCylinder,
    CustomProfile,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum MeshDensity {
    Coarse,   // ~5K nodes
    Medium,   // ~20K nodes
    Fine,     // ~50K nodes
    VeryFine, // ~100K nodes (server only)
}

#[derive(Debug, Deserialize)]
pub struct EhlSimParams {
    pub contact: ContactSimParams,
    pub lubricant_id: Uuid,
    pub rolling_speed_ms: f64,  // Mean rolling speed
    pub slide_to_roll_ratio: f64,  // (v1 - v2) / vmean
    pub temperature_c: f64,
    pub thermal_effects: bool,  // Include thermal EHL
}

#[derive(Debug, Deserialize)]
pub struct WearSimParams {
    pub contact: ContactSimParams,
    pub material_pair_id: Uuid,
    pub sliding_velocity_ms: f64,
    pub num_cycles: u64,
    pub wear_model: WearModel,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum WearModel {
    Archard,
    EnergyDissipation,
    Oxidative,
}
```

---

## Solver Architecture Deep-Dive

### Governing Equations and Discretization

TriboSim's core solvers implement three coupled physical domains: **contact mechanics**, **lubrication**, and **wear**.

#### 1. Hertzian Contact Mechanics (Analytical)

For smooth elastic bodies in contact (sphere-on-flat, cylinder-on-flat, sphere-on-sphere), the Hertzian solution provides analytical expressions for contact radius, maximum pressure, and pressure distribution.

**Point contact (sphere-on-sphere or sphere-on-flat):**

```
Contact radius:  a = (3 F R* / (4 E*))^(1/3)

Max pressure:    p₀ = 3F / (2πa²)

Pressure dist:   p(r) = p₀ √(1 - r²/a²)  for r ≤ a

where:
  F = normal load (N)
  R* = 1/R* = 1/R₁ + 1/R₂  (reduced radius, mm⁻¹)
  E* = 1/E* = (1-ν₁²)/E₁ + (1-ν₂²)/E₂  (reduced modulus, GPa⁻¹)
```

**Line contact (cylinder-on-cylinder or cylinder-on-flat):**

```
Contact half-width:  b = √(4 F R* / (π L E*))

Max pressure:        p₀ = 2F / (π b L)

Pressure dist:       p(x) = p₀ √(1 - x²/b²)  for |x| ≤ b

where:
  L = contact length (mm)
```

The Hertzian solver executes in WASM in <1ms and provides instant feedback for simple geometries.

#### 2. Semi-Analytical Method (SAM) for Non-Hertzian Contact

For non-Hertzian geometries (rough surfaces, worn profiles, edge contact), we use the **FFT-based semi-analytical method**. The elastic deformation field `u(x,y)` caused by pressure distribution `p(x,y)` is:

```
u(x,y) = (1-ν²)/(πE) ∬ p(ξ,η) / √((x-ξ)² + (y-η)²) dξ dη
```

This is a **convolution integral** that can be computed efficiently via FFT:

```
u(x,y) = IFFT[ FFT[p(x,y)] · FFT[K(x,y)] ]

where K(x,y) = (1-ν²)/(πE) · 1/√(x² + y²)  (influence coefficient)
```

**Iterative contact solver:**
1. Initialize pressure `p₀ = 0` everywhere
2. Compute deformation: `u = IFFT[ FFT[p] · K_fft ]`
3. Update gap: `g = g₀ + u₁ + u₂ - δ` (δ = approach distance)
4. Update pressure: `p = p + α · E* · (g < 0 ? -g : 0)` (active set method)
5. Enforce equilibrium: `∑p_i · dA = F_applied`
6. Repeat steps 2-5 until `||p_new - p_old|| / ||p_old|| < 1e-4`

Convergence typically takes 10-50 iterations. FFT dominates cost: O(N log N) for N surface nodes.

#### 3. Reynolds Equation for Elastohydrodynamic Lubrication (EHL)

EHL couples the **Reynolds equation** (fluid mechanics) with **elastic deformation** (solid mechanics):

```
Reynolds equation (2D steady-state):
  ∂/∂x(ρh³/η · ∂p/∂x) + ∂/∂y(ρh³/η · ∂p/∂y) = 6(u₁+u₂) ∂(ρh)/∂x + 12(u₁-u₂) ∂(ρh)/∂y

where:
  p = pressure (Pa)
  h = film thickness (µm) = h₀ + x²/(2Rx) + y²/(2Ry) + u_elastic
  ρ = density (pressure-dependent: ρ = ρ₀(1 + 0.6p/10⁹))
  η = viscosity (pressure-dependent: η = η₀ exp(α·p))
  u₁, u₂ = surface velocities (m/s)
```

**Film thickness coupling:**

```
h(x,y) = h₀ + (x²/(2Rx) + y²/(2Ry))  [geometry]
         + u_elastic(x,y)             [elastic deformation from p]
         - δ                          [rigid body approach]
```

**Discretization:** Central finite differences on uniform grid, giving a sparse nonlinear system. We solve using **Full Approximation Scheme (FAS) multigrid**:

1. **V-cycle**: Relax on fine grid → restrict to coarse grid → solve coarse correction → interpolate back → post-smooth
2. **Outer iteration**: Alternate between Reynolds solve (pressure) and elastic deformation update (FFT) until convergence
3. Typical convergence: 10-30 V-cycles × 5-15 outer iterations

For **thermal EHL**, we add the energy equation:

```
ρc_p(u ∂T/∂x + v ∂T/∂y) = k(∂²T/∂x² + ∂²T/∂y²) + η(∂u/∂z)²
```

Solved via finite differences coupled to Reynolds/elastic loop.

#### 4. Archard Wear Model

Wear depth evolution follows **Archard's law**:

```
dh/dt = k · p · v_sliding / H

where:
  k = dimensionless wear coefficient (from material_pairs table)
  p = contact pressure (Pa)
  v_sliding = sliding velocity (m/s)
  H = hardness of softer material (Pa)
```

For cyclic loading (bearings, gears), we accumulate wear over N cycles:

```
Δh = k · (p · v_sliding / H) · (N · t_cycle)
```

After wear accumulation, the surface profile is updated: `z_new = z_old - Δh`, and contact/EHL analysis is re-run with the worn geometry. Adaptive remeshing is applied when local curvature exceeds threshold.

### Client/Server Split (WASM Threshold)

```
Analysis request → Estimate computational cost
    │
    ├── Hertzian (analytical) → WASM (instant)
    │
    ├── SAM contact ≤50K nodes → WASM (~2-10s)
    │   ├── FFT in browser (fftw-wasm)
    │   └── Results displayed immediately
    │
    ├── EHL (basic formulas) → WASM (~1s)
    │   └── Hamrock-Dowson film thickness
    │
    └── SAM >50K nodes | Full EHL | Thermal EHL | Wear evolution → Server
        ├── Job queued via Redis
        ├── Worker picks up, runs native solver with AVX2/FFTW
        ├── Progress streamed via WebSocket
        └── Results stored in S3, URL returned
```

The 50K-node threshold was chosen because:
- WASM FFT handles 50K nodes (224×224 grid) in ~3 seconds on modern hardware
- 50K nodes covers: single bearing roller contact, simple gear tooth contact, 2D line contact problems
- Above 50K nodes: full bearing assemblies (10+ rollers), 3D rough surface contact, thermal EHL → need server compute with SIMD and multi-threading

### WASM Compilation Pipeline

```toml
# solver-wasm/Cargo.toml
[package]
name = "tribosim-solver-wasm"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
wasm-bindgen = "0.2"
serde = { version = "1", features = ["derive"] }
serde-wasm-bindgen = "0.6"
js-sys = "0.3"
web-sys = { version = "0.3", features = ["console"] }
rustfft = "6.1"  # Pure Rust FFT for WASM
nalgebra = "0.32"
log = "0.4"
console_log = "1"

[profile.release]
opt-level = "z"       # Optimize for size
lto = true            # Link-time optimization
codegen-units = 1     # Single codegen unit
strip = true          # Strip debug symbols
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
      - run: cargo binstall -y wasm-pack
      - run: cd solver-wasm && wasm-pack build --target web --release
      - name: Upload to S3/CDN
        run: aws s3 cp solver-wasm/pkg/ s3://tribosim-wasm/v${{ github.sha }}/ --recursive
```

---

## Architecture Deep-Dives

### 1. Contact Simulation API Handler (Rust/Axum)

The primary endpoint receives a contact analysis request, validates geometry, determines WASM vs. server execution, and for server execution enqueues a job.

```rust
// src/api/handlers/simulation.rs

use axum::{
    extract::{Path, State, Json},
    http::StatusCode,
    response::IntoResponse,
};
use uuid::Uuid;
use crate::{
    db::models::{Simulation, ContactSimParams, MeshDensity},
    solver::geometry::estimate_node_count,
    state::AppState,
    auth::Claims,
    error::ApiError,
};

#[derive(serde::Deserialize)]
pub struct CreateContactSimRequest {
    pub params: ContactSimParams,
}

pub async fn create_contact_simulation(
    State(state): State<AppState>,
    claims: Claims,
    Path(project_id): Path<Uuid>,
    Json(req): Json<CreateContactSimRequest>,
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

    // 2. Estimate computational cost
    let node_count = estimate_node_count(&req.params.geometry, &req.params.mesh_density);

    // 3. Check plan limits
    let user = sqlx::query_as!(
        crate::db::models::User,
        "SELECT * FROM users WHERE id = $1",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    if user.plan == "free" && node_count > 10_000 {
        return Err(ApiError::PlanLimit(
            "Free plan supports contact problems up to 10,000 nodes. Upgrade to Pro for unlimited."
        ));
    }

    if user.plan == "free" && matches!(req.params.mesh_density, MeshDensity::VeryFine) {
        return Err(ApiError::PlanLimit(
            "Very fine mesh requires Pro plan or higher."
        ));
    }

    // 4. Determine execution mode
    let execution_mode = if node_count <= 50_000
        && req.params.geometry.contact_type != crate::db::models::ContactType::CustomProfile
    {
        "wasm"
    } else {
        "server"
    };

    // 5. Create simulation record
    let sim = sqlx::query_as!(
        Simulation,
        r#"INSERT INTO simulations
            (project_id, user_id, simulation_type, status, execution_mode,
             input_params, node_count)
        VALUES ($1, $2, $3, $4, $5, $6, $7)
        RETURNING *"#,
        project_id,
        claims.user_id,
        "sam_contact",
        if execution_mode == "wasm" { "completed" } else { "pending" },
        execution_mode,
        serde_json::to_value(&req.params)?,
        node_count as i32,
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
            if node_count > 200_000 { 8 } else { 4 },
        )
        .fetch_one(&state.db)
        .await?;

        // Publish to Redis job queue
        state.redis
            .publish("simulation:jobs", serde_json::to_string(&job.id)?)
            .await?;
    }

    Ok((StatusCode::CREATED, Json(sim)))
}

pub async fn create_ehl_simulation(
    State(state): State<AppState>,
    claims: Claims,
    Path(project_id): Path<Uuid>,
    Json(req): Json<crate::db::models::EhlSimParams>,
) -> Result<impl IntoResponse, ApiError> {
    // EHL always runs on server due to multigrid complexity
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

    let user = sqlx::query_as!(
        crate::db::models::User,
        "SELECT * FROM users WHERE id = $1",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    if user.plan == "free" {
        return Err(ApiError::PlanLimit(
            "EHL analysis requires Pro plan or higher."
        ));
    }

    if req.thermal_effects && user.plan != "advanced" {
        return Err(ApiError::PlanLimit(
            "Thermal EHL requires Advanced plan."
        ));
    }

    let node_count = estimate_node_count(&req.contact.geometry, &req.contact.mesh_density);

    let sim = sqlx::query_as!(
        Simulation,
        r#"INSERT INTO simulations
            (project_id, user_id, simulation_type, status, execution_mode,
             input_params, node_count)
        VALUES ($1, $2, $3, $4, $5, $6, $7)
        RETURNING *"#,
        project_id,
        claims.user_id,
        if req.thermal_effects { "thermal_ehl" } else { "ehl" },
        "pending",
        "server",
        serde_json::to_value(&req)?,
        node_count as i32,
    )
    .fetch_one(&state.db)
    .await?;

    let job = sqlx::query_as!(
        crate::db::models::SimulationJob,
        r#"INSERT INTO simulation_jobs (simulation_id, priority, cores_allocated, memory_mb)
        VALUES ($1, $2, $3, $4) RETURNING *"#,
        sim.id,
        if user.plan == "advanced" { 10 } else { 5 },
        8,
        if req.thermal_effects { 16384 } else { 8192 },
    )
    .fetch_one(&state.db)
    .await?;

    state.redis
        .publish("simulation:jobs", serde_json::to_string(&job.id)?)
        .await?;

    Ok((StatusCode::CREATED, Json(sim)))
}
```

### 2. Contact Solver Core (Rust — shared between WASM and native)

The FFT-based SAM solver for contact mechanics.

```rust
// solver-core/src/contact/sam.rs

use rustfft::{FftPlanner, num_complex::Complex};
use crate::materials::Material;

pub struct SamContactSolver {
    pub nx: usize,
    pub ny: usize,
    pub dx: f64,  // Grid spacing (mm)
    pub dy: f64,
    pub material_1: Material,
    pub material_2: Material,
    pub geometry: GeometryProfile,
}

pub struct ContactResult {
    pub pressure: Vec<f64>,       // Pa, size nx*ny
    pub gap: Vec<f64>,            // µm
    pub contact_area: f64,        // mm²
    pub max_pressure: f64,        // Pa
    pub approach_distance: f64,   // µm (rigid body approach)
}

impl SamContactSolver {
    pub fn new(
        nx: usize, ny: usize,
        extent_x_mm: f64, extent_y_mm: f64,
        mat1: Material, mat2: Material,
        geometry: GeometryProfile,
    ) -> Self {
        Self {
            nx, ny,
            dx: extent_x_mm / (nx as f64),
            dy: extent_y_mm / (ny as f64),
            material_1: mat1,
            material_2: mat2,
            geometry,
        }
    }

    pub fn solve(&self, normal_load_n: f64, max_iter: usize) -> Result<ContactResult, String> {
        let n = self.nx * self.ny;
        let mut pressure = vec![0.0; n];
        let mut gap = vec![0.0; n];
        let mut deformation = vec![0.0; n];

        // Compute reduced modulus
        let e_star = 1.0 / (
            (1.0 - self.material_1.poisson_ratio.powi(2)) / (self.material_1.elastic_modulus * 1e9)
            + (1.0 - self.material_2.poisson_ratio.powi(2)) / (self.material_2.elastic_modulus * 1e9)
        );  // Pa

        // Precompute FFT of influence coefficients
        let influence_fft = self.compute_influence_fft();

        // Initial guess for approach distance
        let mut delta = self.estimate_initial_approach(normal_load_n, e_star);

        // Conjugate gradient iterations
        for iter in 0..max_iter {
            // 1. Compute elastic deformation via FFT convolution
            self.compute_deformation_fft(&pressure, &influence_fft, &mut deformation);

            // 2. Compute gap
            for i in 0..self.nx {
                for j in 0..self.ny {
                    let idx = i * self.ny + j;
                    let x = (i as f64 - self.nx as f64 / 2.0) * self.dx;
                    let y = (j as f64 - self.ny as f64 / 2.0) * self.dy;
                    let z_geom = self.geometry.height_at(x, y);
                    gap[idx] = z_geom + deformation[idx] - delta;
                }
            }

            // 3. Update pressure (active set method)
            let mut pressure_old = pressure.clone();
            for i in 0..n {
                if gap[i] < 0.0 {
                    // In contact: pressure increases
                    pressure[i] += 0.5 * e_star * (-gap[i] * 1e-6);  // Convert µm → m
                    pressure[i] = pressure[i].max(0.0);
                } else {
                    // Not in contact
                    pressure[i] = 0.0;
                }
            }

            // 4. Enforce load equilibrium
            let total_force: f64 = pressure.iter().sum::<f64>() * self.dx * self.dy * 1e-6;  // mm² → m²
            let load_error = (total_force - normal_load_n).abs() / normal_load_n;

            if load_error > 0.01 {
                // Adjust approach distance to match load
                delta += 0.1 * (total_force - normal_load_n) / (normal_load_n / 1e-6);
            }

            // 5. Check convergence
            let pressure_change: f64 = pressure.iter()
                .zip(&pressure_old)
                .map(|(p, p_old)| (p - p_old).abs())
                .sum::<f64>() / pressure.iter().sum::<f64>();

            if pressure_change < 1e-4 && load_error < 0.01 {
                // Converged
                let contact_area = pressure.iter().filter(|&&p| p > 0.0).count() as f64 * self.dx * self.dy;
                let max_pressure = pressure.iter().cloned().fold(0.0, f64::max);

                return Ok(ContactResult {
                    pressure,
                    gap: gap.iter().map(|&g| g * 1e3).collect(),  // mm → µm
                    contact_area,
                    max_pressure,
                    approach_distance: delta * 1e3,  // mm → µm
                });
            }
        }

        Err(format!("Contact solver did not converge in {} iterations", max_iter))
    }

    fn compute_influence_fft(&self) -> Vec<Complex<f64>> {
        // Compute FFT of influence coefficient K(x,y) = (1-ν²)/(πE) · 1/√(x²+y²)
        let n = self.nx * self.ny;
        let mut influence = vec![0.0; n];

        let e_star = 1.0 / (
            (1.0 - self.material_1.poisson_ratio.powi(2)) / (self.material_1.elastic_modulus * 1e9)
            + (1.0 - self.material_2.poisson_ratio.powi(2)) / (self.material_2.elastic_modulus * 1e9)
        );

        for i in 0..self.nx {
            for j in 0..self.ny {
                let x = (i as f64 - self.nx as f64 / 2.0) * self.dx;
                let y = (j as f64 - self.ny as f64 / 2.0) * self.dy;
                let r = (x.powi(2) + y.powi(2)).sqrt();
                if r > 1e-9 {
                    influence[i * self.ny + j] = 1.0 / (std::f64::consts::PI * e_star * r * 1e-3);  // mm → m
                }
            }
        }

        // 2D FFT
        let mut planner = FftPlanner::new();
        let fft = planner.plan_fft_forward(self.nx);

        let mut buffer: Vec<Complex<f64>> = influence.iter().map(|&x| Complex::new(x, 0.0)).collect();
        // Apply row-wise FFT
        for i in 0..self.nx {
            let mut row: Vec<Complex<f64>> = buffer[i*self.ny..(i+1)*self.ny].to_vec();
            fft.process(&mut row);
            buffer[i*self.ny..(i+1)*self.ny].copy_from_slice(&row);
        }
        // Apply column-wise FFT
        let fft_col = planner.plan_fft_forward(self.ny);
        for j in 0..self.ny {
            let mut col: Vec<Complex<f64>> = (0..self.nx).map(|i| buffer[i*self.ny + j]).collect();
            fft_col.process(&mut col);
            for i in 0..self.nx {
                buffer[i*self.ny + j] = col[i];
            }
        }

        buffer
    }

    fn compute_deformation_fft(
        &self,
        pressure: &[f64],
        influence_fft: &[Complex<f64>],
        deformation: &mut [f64],
    ) {
        // Compute u = IFFT[ FFT[p] · K_fft ]
        let mut planner = FftPlanner::new();
        let fft = planner.plan_fft_forward(self.nx);
        let ifft = planner.plan_fft_inverse(self.nx);

        // FFT of pressure
        let mut p_fft: Vec<Complex<f64>> = pressure.iter().map(|&p| Complex::new(p, 0.0)).collect();

        // Row-wise FFT
        for i in 0..self.nx {
            let mut row: Vec<Complex<f64>> = p_fft[i*self.ny..(i+1)*self.ny].to_vec();
            fft.process(&mut row);
            p_fft[i*self.ny..(i+1)*self.ny].copy_from_slice(&row);
        }
        // Column-wise FFT
        let fft_col = planner.plan_fft_forward(self.ny);
        for j in 0..self.ny {
            let mut col: Vec<Complex<f64>> = (0..self.nx).map(|i| p_fft[i*self.ny + j]).collect();
            fft_col.process(&mut col);
            for i in 0..self.nx {
                p_fft[i*self.ny + j] = col[i];
            }
        }

        // Multiply in frequency domain
        for i in 0..(self.nx * self.ny) {
            p_fft[i] *= influence_fft[i];
        }

        // IFFT
        let ifft_col = planner.plan_fft_inverse(self.ny);
        for j in 0..self.ny {
            let mut col: Vec<Complex<f64>> = (0..self.nx).map(|i| p_fft[i*self.ny + j]).collect();
            ifft_col.process(&mut col);
            for i in 0..self.nx {
                p_fft[i*self.ny + j] = col[i];
            }
        }
        for i in 0..self.nx {
            let mut row: Vec<Complex<f64>> = p_fft[i*self.ny..(i+1)*self.ny].to_vec();
            ifft.process(&mut row);
            p_fft[i*self.ny..(i+1)*self.ny].copy_from_slice(&row);
        }

        // Extract real part and normalize
        let norm = (self.nx * self.ny) as f64;
        for i in 0..(self.nx * self.ny) {
            deformation[i] = p_fft[i].re / norm;
        }
    }

    fn estimate_initial_approach(&self, load_n: f64, e_star: f64) -> f64 {
        // Use Hertzian estimate if applicable
        if let Some(radius) = self.geometry.equivalent_radius_mm() {
            let a = (3.0 * load_n * radius * 1e-3 / (4.0 * e_star)).powf(1.0/3.0);  // m
            a * 1e3  // Convert to mm
        } else {
            1e-3  // 1 µm initial guess
        }
    }
}

pub enum GeometryProfile {
    SphereOnFlat { radius_mm: f64 },
    CylinderOnFlat { radius_mm: f64 },
    Custom { heights: Vec<f64>, nx: usize, ny: usize },
}

impl GeometryProfile {
    pub fn height_at(&self, x: f64, y: f64) -> f64 {
        match self {
            GeometryProfile::SphereOnFlat { radius_mm } => {
                (x.powi(2) + y.powi(2)) / (2.0 * radius_mm)
            }
            GeometryProfile::CylinderOnFlat { radius_mm } => {
                x.powi(2) / (2.0 * radius_mm)
            }
            GeometryProfile::Custom { heights, nx, ny } => {
                // Bilinear interpolation (simplified)
                0.0  // TODO: implement interpolation
            }
        }
    }

    pub fn equivalent_radius_mm(&self) -> Option<f64> {
        match self {
            GeometryProfile::SphereOnFlat { radius_mm } => Some(*radius_mm),
            GeometryProfile::CylinderOnFlat { radius_mm } => Some(*radius_mm),
            GeometryProfile::Custom { .. } => None,
        }
    }
}
```

### 3. Contact Visualization Component (React + WebGL)

The frontend pressure map viewer that renders 2D scalar fields using WebGL with custom color maps.

```rust
// frontend/src/components/ContactViewer.tsx

import { useRef, useEffect, useCallback, useState } from 'react';
import { useContactStore } from '../stores/contactStore';

interface ContactResultData {
  nx: number;
  ny: number;
  extent_x_mm: number;
  extent_y_mm: number;
  pressure: Float32Array;  // Pa, size nx*ny
  max_pressure: number;
  contact_area_mm2: number;
}

export function ContactPressureViewer({ result }: { result: ContactResultData }) {
  const canvasRef = useRef<HTMLCanvasElement>(null);
  const glRef = useRef<WebGL2RenderingContext | null>(null);
  const programRef = useRef<WebGLProgram | null>(null);
  const [colorMap, setColorMap] = useState<'viridis' | 'turbo' | 'isolines'>('viridis');
  const [pressureRange, setPressureRange] = useState({ min: 0, max: result.max_pressure });

  // Initialize WebGL
  useEffect(() => {
    const canvas = canvasRef.current;
    if (!canvas) return;

    const gl = canvas.getContext('webgl2', { antialias: false });
    if (!gl) return;
    glRef.current = gl;

    // Compile shaders for scalar field rendering
    const vsSource = `#version 300 es
      in vec2 a_position;  // Normalized device coords [-1, 1]
      in vec2 a_texCoord;  // Texture coords [0, 1]
      out vec2 v_texCoord;

      void main() {
        gl_Position = vec4(a_position, 0.0, 1.0);
        v_texCoord = a_texCoord;
      }
    `;

    const fsSource = `#version 300 es
      precision highp float;
      in vec2 v_texCoord;
      uniform sampler2D u_pressureField;
      uniform float u_maxPressure;
      uniform int u_colorMap;  // 0=viridis, 1=turbo, 2=isolines
      out vec4 fragColor;

      // Viridis colormap (simplified)
      vec3 viridis(float t) {
        const vec3 c0 = vec3(0.267, 0.005, 0.329);
        const vec3 c1 = vec3(0.283, 0.141, 0.458);
        const vec3 c2 = vec3(0.254, 0.265, 0.530);
        const vec3 c3 = vec3(0.207, 0.372, 0.553);
        const vec3 c4 = vec3(0.164, 0.471, 0.558);
        const vec3 c5 = vec3(0.128, 0.567, 0.551);
        const vec3 c6 = vec3(0.135, 0.659, 0.518);
        const vec3 c7 = vec3(0.267, 0.749, 0.441);
        const vec3 c8 = vec3(0.478, 0.821, 0.318);
        const vec3 c9 = vec3(0.741, 0.873, 0.150);
        const vec3 c10 = vec3(0.993, 0.906, 0.144);

        if (t < 0.1) return mix(c0, c1, t * 10.0);
        else if (t < 0.2) return mix(c1, c2, (t - 0.1) * 10.0);
        else if (t < 0.3) return mix(c2, c3, (t - 0.2) * 10.0);
        else if (t < 0.4) return mix(c3, c4, (t - 0.3) * 10.0);
        else if (t < 0.5) return mix(c4, c5, (t - 0.4) * 10.0);
        else if (t < 0.6) return mix(c5, c6, (t - 0.5) * 10.0);
        else if (t < 0.7) return mix(c6, c7, (t - 0.6) * 10.0);
        else if (t < 0.8) return mix(c7, c8, (t - 0.7) * 10.0);
        else if (t < 0.9) return mix(c8, c9, (t - 0.8) * 10.0);
        else return mix(c9, c10, (t - 0.9) * 10.0);
      }

      void main() {
        float pressure = texture(u_pressureField, v_texCoord).r;
        float t = pressure / u_maxPressure;

        if (u_colorMap == 0) {
          fragColor = vec4(viridis(t), 1.0);
        } else if (u_colorMap == 2) {
          // Isolines
          float level = floor(t * 10.0) / 10.0;
          vec3 color = viridis(level);
          if (fract(t * 10.0) < 0.1) {
            color = vec3(0.0);  // Black isoline
          }
          fragColor = vec4(color, 1.0);
        } else {
          // Default: simple gradient
          fragColor = vec4(t, 0.5, 1.0 - t, 1.0);
        }
      }
    `;

    const program = createShaderProgram(gl, vsSource, fsSource);
    programRef.current = program;

    return () => {
      gl.deleteProgram(program);
    };
  }, []);

  // Upload pressure data as texture and render
  const render = useCallback(() => {
    const gl = glRef.current;
    const program = programRef.current;
    if (!gl || !program) return;

    gl.viewport(0, 0, gl.canvas.width, gl.canvas.height);
    gl.clearColor(0.1, 0.1, 0.1, 1.0);
    gl.clear(gl.COLOR_BUFFER_BIT);
    gl.useProgram(program);

    // Create texture from pressure data
    const texture = gl.createTexture();
    gl.bindTexture(gl.TEXTURE_2D, texture);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.LINEAR);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.LINEAR);

    // Upload pressure field
    gl.texImage2D(
      gl.TEXTURE_2D,
      0,
      gl.R32F,  // Single-channel float
      result.ny,
      result.nx,
      0,
      gl.RED,
      gl.FLOAT,
      result.pressure
    );

    // Create quad covering viewport
    const positions = new Float32Array([
      -1, -1,  0, 0,  // Bottom-left (pos, tex)
       1, -1,  1, 0,  // Bottom-right
      -1,  1,  0, 1,  // Top-left
       1,  1,  1, 1,  // Top-right
    ]);

    const buffer = gl.createBuffer();
    gl.bindBuffer(gl.ARRAY_BUFFER, buffer);
    gl.bufferData(gl.ARRAY_BUFFER, positions, gl.STATIC_DRAW);

    const posLoc = gl.getAttribLocation(program, 'a_position');
    const texLoc = gl.getAttribLocation(program, 'a_texCoord');
    gl.enableVertexAttribArray(posLoc);
    gl.vertexAttribPointer(posLoc, 2, gl.FLOAT, false, 16, 0);
    gl.enableVertexAttribArray(texLoc);
    gl.vertexAttribPointer(texLoc, 2, gl.FLOAT, false, 16, 8);

    // Set uniforms
    const maxPressureLoc = gl.getUniformLocation(program, 'u_maxPressure');
    const colorMapLoc = gl.getUniformLocation(program, 'u_colorMap');
    gl.uniform1f(maxPressureLoc, pressureRange.max);
    gl.uniform1i(colorMapLoc, colorMap === 'viridis' ? 0 : colorMap === 'turbo' ? 1 : 2);

    gl.drawArrays(gl.TRIANGLE_STRIP, 0, 4);

    gl.deleteBuffer(buffer);
    gl.deleteTexture(texture);
  }, [result, colorMap, pressureRange]);

  useEffect(() => {
    render();
  }, [render]);

  return (
    <div className="contact-viewer flex flex-col h-full bg-gray-900">
      <div className="toolbar flex items-center gap-4 p-2 bg-gray-800">
        <label className="text-white text-sm">
          Color Map:
          <select
            value={colorMap}
            onChange={(e) => setColorMap(e.target.value as any)}
            className="ml-2 bg-gray-700 text-white px-2 py-1 rounded"
          >
            <option value="viridis">Viridis</option>
            <option value="turbo">Turbo</option>
            <option value="isolines">Isolines</option>
          </select>
        </label>
        <div className="text-white text-sm">
          Max Pressure: {(result.max_pressure / 1e6).toFixed(1)} MPa
        </div>
        <div className="text-white text-sm">
          Contact Area: {result.contact_area_mm2.toFixed(2)} mm²
        </div>
      </div>
      <div className="flex-1 relative">
        <canvas
          ref={canvasRef}
          width={result.ny}
          height={result.nx}
          className="w-full h-full"
        />
      </div>
      <ColorBar max={pressureRange.max} colorMap={colorMap} />
    </div>
  );
}

function ColorBar({ max, colorMap }: { max: number; colorMap: string }) {
  return (
    <div className="color-bar flex items-center gap-2 p-2 bg-gray-800">
      <div className="h-4 flex-1" style={{
        background: colorMap === 'viridis'
          ? 'linear-gradient(to right, #440154, #31688e, #35b779, #fde724)'
          : 'linear-gradient(to right, blue, cyan, yellow, red)'
      }} />
      <span className="text-white text-xs">0</span>
      <span className="text-white text-xs">{(max / 1e6).toFixed(0)} MPa</span>
    </div>
  );
}

function createShaderProgram(
  gl: WebGL2RenderingContext,
  vsSource: string,
  fsSource: string
): WebGLProgram {
  const vs = gl.createShader(gl.VERTEX_SHADER)!;
  gl.shaderSource(vs, vsSource);
  gl.compileShader(vs);

  const fs = gl.createShader(gl.FRAGMENT_SHADER)!;
  gl.shaderSource(fs, fsSource);
  gl.compileShader(fs);

  const program = gl.createProgram()!;
  gl.attachShader(program, vs);
  gl.attachShader(program, fs);
  gl.linkProgram(program);

  return program;
}
```

### 4. Bearing Life Calculator (Rust — ISO 281)

Rolling bearing life calculator implementing ISO 281 with advanced life factors.

```rust
// solver-core/src/bearing/life.rs

use crate::bearing::catalog::Bearing;

pub struct BearingLifeCalculator {
    pub bearing: Bearing,
    pub radial_load_n: f64,
    pub axial_load_n: f64,
    pub speed_rpm: f64,
    pub temperature_c: f64,
    pub contamination: ContaminationLevel,
    pub misalignment_deg: f64,
}

#[derive(Debug, Clone, Copy)]
pub enum ContaminationLevel {
    HighCleanliness,  // ISO 4406 13/10
    NormalCleanliness,  // ISO 4406 16/13
    TypicalCleanliness,  // ISO 4406 18/15
    SevereCont,  // ISO 4406 21/18
}

pub struct LifeResult {
    pub l10_hours: f64,        // Basic L10 life
    pub l10m_hours: f64,       // Modified L10 life with aISO factor
    pub c_over_p: f64,         // Dynamic load ratio
    pub equivalent_load_n: f64,
    pub a_iso: f64,            // ISO 281 life modification factor
}

impl BearingLifeCalculator {
    pub fn calculate(&self) -> LifeResult {
        // 1. Calculate equivalent dynamic load P
        let (x, y) = self.load_factors();
        let p_radial = x * self.radial_load_n;
        let p_axial = y * self.axial_load_n;
        let equivalent_load = p_radial + p_axial;

        // 2. Basic dynamic load rating C
        let c = self.bearing.dynamic_load_rating.unwrap_or(1000.0) * 1000.0;  // kN → N

        // 3. Basic L10 life (million revolutions)
        let exponent = if self.bearing.bearing_type.contains("ball") { 3.0 } else { 10.0/3.0 };
        let l10_revs = (c / equivalent_load).powf(exponent);  // million revs

        // 4. Convert to hours
        let l10_hours = (l10_revs * 1e6) / (self.speed_rpm * 60.0);

        // 5. Calculate ISO 281 life modification factor aISO
        let a_iso = self.calculate_a_iso(c, equivalent_load);

        // 6. Modified life L10m
        let l10m_hours = a_iso * l10_hours;

        LifeResult {
            l10_hours,
            l10m_hours,
            c_over_p: c / equivalent_load,
            equivalent_load_n: equivalent_load,
            a_iso,
        }
    }

    fn load_factors(&self) -> (f64, f64) {
        // Simplified load factors (X, Y) for radial and axial loads
        // Proper implementation should consult bearing type and F_a/F_r ratio
        match self.bearing.bearing_type.as_str() {
            "radial_ball" => {
                let ratio = self.axial_load_n / self.radial_load_n.max(1.0);
                if ratio <= 0.172 {
                    (1.0, 0.0)
                } else {
                    (0.56, 2.0 * ratio)
                }
            }
            "angular_contact" => {
                let ratio = self.axial_load_n / self.radial_load_n.max(1.0);
                if ratio <= 0.68 {
                    (1.0, 0.0)
                } else {
                    (0.40, 0.87 * ratio)
                }
            }
            "cylindrical_roller" => (1.0, 0.0),  // Pure radial
            "tapered_roller" => (0.4, 0.4 * self.axial_load_n / self.radial_load_n.max(1.0)),
            _ => (1.0, 0.0)
        }
    }

    fn calculate_a_iso(&self, c: f64, p: f64) -> f64 {
        // ISO 281:2007 aISO = a1 · a2 · a3
        // a1: life adjustment for reliability (default 1.0 for 90% reliability)
        // a2: life adjustment for material properties
        // a3: life adjustment for operating conditions

        let a1 = 1.0;  // 90% reliability (L10)

        // a2: material factor (depends on steel quality, default 1.0 for high-quality bearing steel)
        let a2 = 1.0;

        // a3: operating conditions (lubrication, contamination, temperature)
        let a3 = self.calculate_a3(c, p);

        a1 * a2 * a3
    }

    fn calculate_a3(&self, c: f64, p: f64) -> f64 {
        // Simplified a3 calculation based on contamination and load ratio
        // Full ISO 281 requires lubrication regime, viscosity ratio κ, etc.

        let eta_c = match self.contamination {
            ContaminationLevel::HighCleanliness => 1.0,
            ContaminationLevel::NormalCleanliness => 0.8,
            ContaminationLevel::TypicalCleanliness => 0.5,
            ContaminationLevel::SevereCont => 0.1,
        };

        let c_over_p = c / p;

        // Simplified formula (actual ISO 281 uses lookup tables)
        if c_over_p < 1.5 {
            0.1  // Overloaded, minimal life extension
        } else {
            eta_c * (0.1 + 0.9 * (1.0 - (-c_over_p / 10.0).exp()))
        }
    }
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–4)

**Day 1: Project initialization and Rust backend scaffold**
```bash
cargo init tribosim-api
cd tribosim-api
cargo add axum tokio serde serde_json sqlx uuid chrono tower-http jsonwebtoken bcrypt
cargo add --features postgres,runtime-tokio-rustls,uuid,chrono,json sqlx
cargo add aws-sdk-s3 redis rustfft nalgebra
```
- `src/main.rs` — Axum app with CORS, tracing, graceful shutdown
- `src/config.rs` — Environment-based configuration (DATABASE_URL, JWT_SECRET, S3_BUCKET, REDIS_URL)
- `src/state.rs` — AppState struct (PgPool, Redis, S3Client)
- `src/error.rs` — ApiError enum with IntoResponse
- `Dockerfile` — Multi-stage build (builder + runtime)
- `docker-compose.yml` — PostgreSQL, Redis, MinIO (S3-compatible)

**Day 2: Database schema and migrations**
- `migrations/001_initial.sql` — All 10 tables: users, organizations, org_members, projects, simulations, simulation_jobs, materials, material_pairs, lubricants, bearings, surface_measurements, usage_records
- `src/db/mod.rs` — Database pool initialization
- `src/db/models.rs` — All SQLx structs with FromRow derives
- Run `sqlx migrate run` to apply schema
- Seed script for initial materials (steels, ceramics, polymers) and lubricants (ISO VG grades)

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

### Phase 2 — Contact Solver Core (Days 5–11)

**Day 5: Hertzian contact solver (analytical)**
- `solver-core/` — New Rust workspace member (shared between native and WASM)
- `solver-core/src/contact/hertzian.rs` — Analytical Hertzian formulas for point and line contact
- `solver-core/src/materials.rs` — Material struct with elastic properties
- Unit tests: sphere-on-flat, cylinder-on-flat, verify contact radius and max pressure against textbook examples

**Day 6: FFT library integration and influence coefficients**
- `solver-core/src/contact/fft.rs` — FFT utilities using `rustfft`
- `solver-core/src/contact/influence.rs` — Precompute influence coefficient matrix K(x,y) = 1/(πE*r)
- 2D FFT of influence coefficients for convolution
- Unit tests: verify FFT round-trip, check influence coefficient values

**Day 7: SAM contact solver — basic implementation**
- `solver-core/src/contact/sam.rs` — SamContactSolver struct with iterative solver
- Geometry profiles: sphere-on-flat, cylinder-on-flat
- Pressure update via active set method
- Load equilibrium enforcement
- Tests: compare to Hertzian for simple geometries (should match within 1%)

**Day 8: SAM contact solver — advanced features**
- Custom profile support (import height data from array)
- Adaptive relaxation parameter tuning for faster convergence
- Gap and deformation field computation
- Contact area calculation
- Tests: verify convergence for rough geometries

**Day 9: Reynolds equation discretization**
- `solver-core/src/ehl/reynolds.rs` — Reynolds equation finite difference discretization
- Sparse matrix assembly (CSR format via `nalgebra-sparse`)
- Pressure-viscosity relation (Barus/Roelands)
- Density-pressure relation
- Tests: 1D slider bearing (compare to analytical solution)

**Day 10: EHL multigrid solver**
- `solver-core/src/ehl/multigrid.rs` — FAS multigrid solver for Reynolds equation
- V-cycle with Gauss-Seidel relaxation
- Coarse grid construction and interpolation operators
- Tests: 2D EHL line contact (compare film thickness to Hamrock-Dowson formulas)

**Day 11: EHL coupling with elastic deformation**
- `solver-core/src/ehl/coupled.rs` — Outer loop coupling Reynolds solver with FFT-based deformation
- Film thickness update: h = h_geometric + u_elastic
- Convergence criteria for coupled solution
- Tests: point contact EHL (verify central/minimum film thickness)

### Phase 3 — WASM Build + Frontend Foundation (Days 12–17)

**Day 12: WASM solver compilation**
- `solver-wasm/` — New workspace member for WASM target
- `solver-wasm/Cargo.toml` — wasm-bindgen, serde-wasm-bindgen dependencies
- `solver-wasm/src/lib.rs` — WASM entry points: `solve_hertzian()`, `solve_sam_contact()`, `solve_ehl_basic()`
- `wasm-pack build --target web --release` pipeline
- JavaScript wrapper: `TriboSolver` class that loads WASM and provides async API

**Day 13: Frontend scaffold**
```bash
npm create vite@latest frontend -- --template react-ts
cd frontend
npm i zustand @tanstack/react-query axios three @react-three/fiber @react-three/drei
```
- `src/App.tsx` — Router, auth context, layout
- `src/stores/projectStore.ts` — Zustand store for project state
- `src/stores/contactStore.ts` — Contact analysis state (geometry, materials, results)
- `src/lib/api.ts` — Axios API client

**Day 14: Geometry editor — 2D profile drawing**
- `src/components/GeometryEditor/Canvas2D.tsx` — WebGL canvas for 2D profile editing
- `src/components/GeometryEditor/ProfileTools.tsx` — Tools: line, arc, Bézier curve, point editing
- `src/components/GeometryEditor/GeometryLibrary.tsx` — Predefined shapes (sphere, cylinder, flat, custom)
- Snap-to-grid and measurement overlays
- SVG import for custom profiles

**Day 15: Material and lubricant selectors**
- `src/components/MaterialSelector.tsx` — Search and select materials from database
- `src/components/LubricantSelector.tsx` — Search and select lubricants
- Property display panels (E, ν, hardness for materials; viscosity, α for lubricants)
- Custom material entry (Pro plan feature)

**Day 16: Contact visualization — pressure maps**
- `src/components/ContactViewer.tsx` — WebGL pressure map viewer (from Architecture Deep-Dive #3)
- Color map selection: Viridis, Turbo, Isolines
- Pressure range adjustment and color bar
- Export pressure field as PNG

**Day 17: 3D surface deformation viewer**
- `src/components/SurfaceViewer3D.tsx` — Three.js 3D surface renderer
- Geometry: height field mesh from deformation data
- Color-mapped by pressure or gap
- Interactive rotation, pan, zoom
- Lighting and material shading

### Phase 4 — API + Job Orchestration (Days 18–22)

**Day 18: Contact simulation API endpoints**
- `src/api/handlers/simulation.rs` — Create contact simulation, get simulation (from Architecture Deep-Dive #1)
- Geometry validation and node count estimation
- WASM/server routing logic
- Plan-based limits enforcement

**Day 19: EHL simulation API endpoints**
- `src/api/handlers/simulation.rs` — `create_ehl_simulation()` (from Architecture Deep-Dive #1)
- Lubricant validation (check exists in database)
- Server-side execution only (no WASM for full EHL)

**Day 20: Server-side simulation worker**
- `src/workers/simulation_worker.rs` — Redis job consumer
- Contact solver execution (SAM)
- EHL solver execution (multigrid)
- Result serialization and S3 upload
- Error handling: convergence failures, timeouts

**Day 21: WebSocket for live simulation progress**
- `src/api/ws/mod.rs` — Axum WebSocket upgrade handler
- `src/api/ws/simulation_progress.rs` — Subscribe to simulation progress channel
- Client receives: `{ progress_pct, iteration, residual, max_pressure }`
- Frontend: `src/hooks/useSimulationProgress.ts` — React hook for WebSocket subscription

**Day 22: Material and lubricant API endpoints**
- `src/api/handlers/materials.rs` — Search materials, get material, create custom material (Pro)
- `src/api/handlers/lubricants.rs` — Search lubricants, get lubricant, create custom lubricant (Advanced)
- Parametric search: filter by category, E range, hardness range
- Pagination with cursor-based scrolling

### Phase 5 — Bearing Analysis + Wear Simulation (Days 23–27)

**Day 23: Bearing catalog API and life calculator**
- `src/api/handlers/bearings.rs` — Search bearings, get bearing details
- `solver-core/src/bearing/life.rs` — ISO 281 L10 and aISO calculator (from Architecture Deep-Dive #4)
- Bearing life API endpoint: input loads/speed → output L10/L10m
- Frontend: `src/components/BearingLifeCalculator.tsx`

**Day 24: Wear model implementation**
- `solver-core/src/wear/archard.rs` — Archard wear model implementation
- `solver-core/src/wear/evolution.rs` — Surface evolution over N cycles
- Material pair coefficient lookup from database
- Tests: verify wear depth accumulation

**Day 25: Wear simulation API and worker**
- `src/api/handlers/simulation.rs` — `create_wear_simulation()`
- Worker: iterative contact solve → wear accumulation → geometry update → repeat
- Progress: cycles completed, max wear depth, contact area evolution
- Result: wear depth map, final geometry

**Day 26: Surface measurement data handling**
- `src/api/handlers/surface.rs` — Upload surface measurement file (CSV/HDF5), list measurements
- Python service: `surface-service/` — Parse profilometry data, compute roughness statistics
- S3 upload for raw measurement data
- Frontend: `src/components/SurfaceDataUpload.tsx`

**Day 27: Bearing geometry visualization (Three.js)**
- `src/components/BearingViewer3D.tsx` — Three.js viewer for bearing assemblies
- STEP file import (via conversion service or opencascade.js)
- Component highlighting (inner race, outer race, rollers, cage)
- Contact patch visualization overlay

### Phase 6 — Results Visualization + Export (Days 28–31)

**Day 28: Result post-processing**
- Result parsing: binary format (pressure field, gap field) → structured data
- Result caching: hot results in Redis (TTL 1 hour), cold results in S3
- Presigned S3 URLs for client-side download
- Result summary generation: max pressure, contact area, min film thickness

**Day 29: Interactive result exploration**
- Cross-section plots: pressure(x) along a line
- Histogram of pressure distribution
- Contact area vs. load parametric sweep
- Film thickness vs. speed parametric sweep
- Export plots as PNG/SVG

**Day 30: Report generation**
- `src/api/handlers/export.rs` — Generate PDF report with geometry, inputs, results, plots
- Report template with company branding (Advanced plan)
- Include: pressure maps, film thickness maps, life estimates, material properties
- Frontend: `src/components/ReportGenerator.tsx`

**Day 31: CSV/MATLAB export**
- Export pressure field as CSV (x, y, p)
- Export film thickness field as CSV
- MATLAB .mat file export (Advanced plan)
- Export bearing life results as structured JSON

### Phase 7 — Billing + Plan Enforcement (Days 32–35)

**Day 32: Stripe integration**
- `src/api/handlers/billing.rs` — Create checkout session, create customer portal session
- `src/api/handlers/webhooks/stripe.rs` — Handle subscription events
- Plan mapping:
  - Free: Hertzian only, 10K nodes max, 3 projects
  - Pro ($129/mo): SAM contact ≤50K nodes, basic EHL, Archard wear, bearing life, unlimited projects
  - Advanced ($299/user/mo): Unlimited nodes, thermal EHL, custom materials/lubricants, API access, team collaboration

**Day 33: Usage tracking and plan limits**
- `src/middleware/plan_limits.rs` — Middleware checking plan limits before simulation execution
- `src/services/usage.rs` — Track server simulation hours per billing period
- Usage dashboard: current period usage, historical usage
- Approaching-limit warnings at 80% and 100% of quota

**Day 34: Billing UI**
- `frontend/src/pages/Billing.tsx` — Plan comparison table, current plan, usage meter
- `frontend/src/components/billing/PlanCard.tsx`
- `frontend/src/components/billing/UsageMeter.tsx`
- Upgrade/downgrade flow via Stripe Customer Portal
- Upgrade prompt modals when hitting limits

**Day 35: Feature gating**
- Gate thermal EHL behind Advanced plan
- Gate custom materials/lubricants behind Pro/Advanced
- Gate API access behind Advanced
- Gate report generation with branding behind Advanced
- Show locked feature indicators with upgrade CTAs

### Phase 8 — Validation + Testing + Launch (Days 36–42)

**Day 36: Solver validation — Hertzian and SAM**
- Benchmark 1: Steel sphere on flat (see Validation Suite below)
- Benchmark 2: Cylinder on cylinder (line contact)
- Benchmark 3: SAM vs. Hertzian (verify convergence to same result)
- Automated test suite: `solver-core/tests/benchmarks.rs`

**Day 37: Solver validation — EHL and bearing life**
- Benchmark 4: EHL film thickness (see Validation Suite below)
- Benchmark 5: Bearing L10 life (see Validation Suite below)
- Compare results against published data from tribology literature

**Day 38: Integration testing**
- End-to-end test: create project → define geometry → select materials → run contact sim → view results
- API integration tests: auth → project CRUD → simulation → results
- WASM solver test: load in headless browser, run Hertzian, verify results
- WebSocket test: connect → subscribe → receive progress
- Concurrent simulation test: 5 simultaneous server-side jobs

**Day 39: Performance testing and optimization**
- WASM solver benchmarks: 10K, 30K, 50K node SAM contact in browser
- Target: 50K nodes < 10 seconds in WASM
- Server solver benchmarks: 100K, 500K node SAM contact
- Target: 100K nodes < 30 seconds on 4 cores
- Frontend rendering: WebGL pressure map with 100K nodes at 60 FPS

**Day 40: Docker and Kubernetes configuration**
- `Dockerfile` — Multi-stage Rust build with cargo-chef
- `docker-compose.yml` — Full local dev stack
- `k8s/` — Kubernetes manifests (API, workers, PostgreSQL, Redis)
- Health check endpoints: `/health/live`, `/health/ready`
- HPA configuration for API and workers

**Day 41: Monitoring, logging, and polish**
- Prometheus metrics: simulation duration, convergence iterations, API latency
- Grafana dashboards: system health, simulation throughput
- Sentry integration: error tracking with source maps
- UI polish: loading states, error messages, tooltips, responsive layout
- Accessibility: keyboard navigation, ARIA labels

**Day 42: Launch preparation and final testing**
- End-to-end smoke test in production
- Database backup and restore verification
- Rate limiting: 60 req/min per user
- Security audit: JWT validation, SQL injection protection, CORS, CSP
- Landing page with product overview, pricing, signup
- Documentation: getting started, bearing life guide, contact mechanics primer
- Deploy to production, enable monitoring, announce launch

---

## Critical Files

```
tribosim/
├── solver-core/                           # Shared solver library (native + WASM)
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── materials.rs                   # Material properties (E, ν, hardness)
│   │   ├── contact/
│   │   │   ├── mod.rs
│   │   │   ├── hertzian.rs                # Analytical Hertzian contact
│   │   │   ├── sam.rs                     # FFT-based SAM solver
│   │   │   ├── fft.rs                     # FFT utilities
│   │   │   └── influence.rs               # Influence coefficient computation
│   │   ├── ehl/
│   │   │   ├── mod.rs
│   │   │   ├── reynolds.rs                # Reynolds equation discretization
│   │   │   ├── multigrid.rs               # FAS multigrid solver
│   │   │   ├── coupled.rs                 # EHL coupling loop
│   │   │   └── thermal.rs                 # Thermal EHL (energy equation)
│   │   ├── wear/
│   │   │   ├── mod.rs
│   │   │   ├── archard.rs                 # Archard wear model
│   │   │   ├── evolution.rs               # Surface evolution over cycles
│   │   │   └── remesh.rs                  # Adaptive remeshing
│   │   └── bearing/
│   │       ├── mod.rs
│   │       ├── catalog.rs                 # Bearing catalog types
│   │       └── life.rs                    # ISO 281 L10/aISO calculator
│   └── tests/
│       ├── benchmarks.rs                  # Solver validation benchmarks
│       └── convergence.rs                 # Convergence stress tests
│
├── solver-wasm/                           # WASM compilation target
│   ├── Cargo.toml
│   └── src/
│       └── lib.rs                         # WASM entry points (wasm-bindgen)
│
├── tribosim-api/                          # Rust API server
│   ├── Cargo.toml
│   ├── src/
│   │   ├── main.rs                        # Axum app setup
│   │   ├── config.rs                      # Environment config
│   │   ├── state.rs                       # AppState (PgPool, Redis, S3)
│   │   ├── error.rs                       # ApiError types
│   │   ├── auth/
│   │   │   ├── mod.rs                     # JWT middleware
│   │   │   └── oauth.rs                   # Google/GitHub OAuth
│   │   ├── api/
│   │   │   ├── router.rs                  # Route definitions
│   │   │   ├── handlers/
│   │   │   │   ├── auth.rs
│   │   │   │   ├── users.rs
│   │   │   │   ├── projects.rs
│   │   │   │   ├── simulation.rs          # Contact/EHL/wear simulation endpoints
│   │   │   │   ├── materials.rs           # Material CRUD
│   │   │   │   ├── lubricants.rs          # Lubricant CRUD
│   │   │   │   ├── bearings.rs            # Bearing catalog search
│   │   │   │   ├── surface.rs             # Surface measurement upload
│   │   │   │   ├── billing.rs             # Stripe integration
│   │   │   │   ├── export.rs              # Report/CSV export
│   │   │   │   └── webhooks/
│   │   │   │       └── stripe.rs
│   │   │   └── ws/
│   │   │       ├── mod.rs                 # WebSocket handler
│   │   │       └── simulation_progress.rs
│   │   ├── middleware/
│   │   │   └── plan_limits.rs
│   │   ├── services/
│   │   │   ├── usage.rs
│   │   │   └── s3.rs
│   │   └── workers/
│   │       ├── mod.rs
│   │       └── simulation_worker.rs       # Server-side solver execution
│   ├── migrations/
│   │   └── 001_initial.sql                # Full database schema
│   └── tests/
│       └── api_integration.rs
│
├── frontend/                              # React frontend
│   ├── package.json
│   ├── vite.config.ts
│   ├── src/
│   │   ├── App.tsx
│   │   ├── main.tsx
│   │   ├── stores/
│   │   │   ├── authStore.ts
│   │   │   ├── projectStore.ts
│   │   │   ├── contactStore.ts            # Contact analysis state
│   │   │   └── bearingStore.ts            # Bearing analysis state
│   │   ├── hooks/
│   │   │   ├── useSimulationProgress.ts   # WebSocket hook
│   │   │   └── useWasmSolver.ts           # WASM solver hook
│   │   ├── lib/
│   │   │   ├── api.ts                     # Axios API client
│   │   │   └── wasmLoader.ts              # WASM bundle loading
│   │   ├── pages/
│   │   │   ├── Dashboard.tsx
│   │   │   ├── ContactEditor.tsx          # Main contact analysis page
│   │   │   ├── BearingAnalysis.tsx        # Bearing life calculator page
│   │   │   ├── MaterialLibrary.tsx        # Material browser
│   │   │   ├── LubricantLibrary.tsx       # Lubricant browser
│   │   │   ├── Billing.tsx
│   │   │   └── Login.tsx
│   │   ├── components/
│   │   │   ├── GeometryEditor/
│   │   │   │   ├── Canvas2D.tsx           # 2D profile editor
│   │   │   │   ├── ProfileTools.tsx       # Drawing tools
│   │   │   │   └── GeometryLibrary.tsx    # Predefined shapes
│   │   │   ├── ContactViewer/
│   │   │   │   ├── ContactPressureViewer.tsx  # WebGL pressure map
│   │   │   │   └── SurfaceViewer3D.tsx    # Three.js 3D surface
│   │   │   ├── BearingViewer/
│   │   │   │   ├── BearingViewer3D.tsx    # Three.js bearing assembly
│   │   │   │   └── BearingLifeCalculator.tsx
│   │   │   ├── MaterialSelector.tsx
│   │   │   ├── LubricantSelector.tsx
│   │   │   ├── SurfaceDataUpload.tsx
│   │   │   ├── ReportGenerator.tsx
│   │   │   ├── billing/
│   │   │   │   ├── PlanCard.tsx
│   │   │   │   └── UsageMeter.tsx
│   │   │   └── common/
│   │   │       └── SimulationToolbar.tsx
│   │   └── public/
│   │       └── wasm/                      # WASM solver bundle
│
├── surface-service/                       # Python FastAPI (surface analysis)
│   ├── requirements.txt
│   ├── main.py                            # FastAPI app
│   ├── surface_analysis.py                # Roughness statistics (Ra, Rq, etc.)
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
        ├── ci.yml
        ├── wasm-build.yml
        └── deploy.yml
```

---

## Solver Validation Suite

### Benchmark 1: Steel Sphere on Flat (Hertzian Contact)

**Geometry:** Steel sphere (R = 10 mm) on flat steel surface

**Materials:** Both AISI 52100 bearing steel (E = 210 GPa, ν = 0.3)

**Load:** F = 100 N

**Expected (Hertzian analytical):**
- Contact radius: a = (3FR*/(4E*))^(1/3) = (3×100×0.005/(4×115e9))^(1/3) = **0.172 mm**
- Max pressure: p₀ = 3F/(2πa²) = **809 MPa**
- Reduced modulus: E* = 1/((1-0.3²)/210e9 + (1-0.3²)/210e9) = **115 GPa**

**Tolerance:** Contact radius < 0.5%, max pressure < 1%

### Benchmark 2: Cylinder on Cylinder (Line Contact)

**Geometry:** Steel cylinder (R₁ = 20 mm) on steel cylinder (R₂ = 30 mm), contact length L = 10 mm

**Materials:** Both AISI 52100 (E = 210 GPa, ν = 0.3)

**Load:** F = 500 N

**Expected (Hertzian):**
- Contact half-width: b = √(4FR*/(πLE*)) where R* = (1/20 + 1/30)^(-1) = 12 mm → b = **0.206 mm**
- Max pressure: p₀ = 2F/(πbL) = **493 MPa**

**Tolerance:** Contact width < 0.5%, max pressure < 1%

### Benchmark 3: SAM vs. Hertzian Convergence

**Geometry:** Same as Benchmark 1 (sphere on flat)

**Method:** Solve with SAM on 128×128 grid, compare to Hertzian analytical

**Expected:** SAM should converge to Hertzian solution within **<2%** for max pressure and contact area after proper mesh refinement

**Tolerance:** Pressure < 2%, contact area < 3%

### Benchmark 4: EHL Film Thickness (Hamrock-Dowson Formula)

**Geometry:** Steel ball on flat, R = 10 mm

**Materials:** Steel (E = 210 GPa, ν = 0.3)

**Lubricant:** ISO VG 68 mineral oil (η₀ = 0.068 Pa·s at 40°C, α = 2.2e-8 Pa⁻¹)

**Conditions:** Load F = 50 N, rolling speed U = 1 m/s, temperature T = 40°C

**Expected (Hamrock-Dowson):**
- Central film thickness: h_c ≈ 3.63 × R × (αE*)^0.68 × (η₀U/(E*R))^0.68 × (F/(E*R²))^(-0.073) ≈ **0.85 µm**
- Minimum film thickness: h_min ≈ 2.69 × ... ≈ **0.63 µm**

**Tolerance:** h_c < 10%, h_min < 15% (multigrid converged solution vs. empirical formula)

### Benchmark 5: Rolling Bearing L10 Life (ISO 281)

**Bearing:** 6205 deep groove ball bearing (C = 14.0 kN, C₀ = 6.55 kN)

**Load:** Pure radial load F_r = 2000 N (F_a = 0)

**Speed:** 3000 rpm

**Expected (ISO 281 basic):**
- Equivalent load: P = F_r = 2000 N
- L10 (million revs): (C/P)³ = (14000/2000)³ = **343 million revolutions**
- L10 (hours): 343e6 / (3000 × 60) = **1906 hours**

**With contamination (Normal Cleanliness) and moderate C/P ratio:**
- aISO factor ≈ 0.7 (simplified)
- L10m ≈ 0.7 × 1906 ≈ **1334 hours**

**Tolerance:** L10 < 1% (analytical), L10m < 10% (depends on aISO model accuracy)

---

## Verification

### Integration Testing Checklist

1. **Auth flow** — Register → login → JWT issued → authorized API calls → logout
2. **Project CRUD** — Create → update geometry → save → reload → verify geometry preserved
3. **Material selection** — Search "steel" → results returned → select AISI 52100 → properties displayed
4. **Hertzian simulation** — Select sphere-on-flat → set R, F → solve → instant results (<1s)
5. **SAM WASM simulation** — Custom profile, 20K nodes → WASM solver → results in ~5s
6. **SAM server simulation** — 80K nodes → job queued → worker picks up → WebSocket progress → results in S3
7. **EHL simulation** — Line contact → select lubricant → server execution → film thickness map
8. **Bearing life calculation** — Select bearing from catalog → input loads/speed → L10/L10m calculated
9. **Wear simulation** — Contact + material pair → 10K cycles → surface evolution displayed
10. **Pressure map viewer** — Load result → change color map → pan/zoom → export PNG
11. **3D surface viewer** — Load deformation → rotate → zoom → verify rendering
12. **Plan limits** — Free user → try EHL → blocked with upgrade prompt
13. **Billing** — Subscribe to Pro → Stripe checkout → webhook → plan updated → EHL enabled
14. **Concurrent users** — 5 users simultaneously running server simulations → all complete
15. **Error handling** — Non-convergent contact problem → meaningful error message → no crash

### SQL Verification Queries

```sql
-- 1. Simulation throughput and success rate
SELECT
    simulation_type,
    status,
    COUNT(*) as count,
    AVG(wall_time_ms)::int as avg_time_ms,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY wall_time_ms)::int as p95_time_ms
FROM simulations
WHERE created_at >= NOW() - INTERVAL '7 days'
GROUP BY simulation_type, status;

-- 2. Simulation type distribution
SELECT simulation_type, COUNT(*) as count,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 1) as pct
FROM simulations
WHERE created_at >= NOW() - INTERVAL '30 days'
GROUP BY simulation_type ORDER BY count DESC;

-- 3. User plan distribution
SELECT plan, COUNT(*) as users,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 1) as pct
FROM users GROUP BY plan ORDER BY users DESC;

-- 4. Server simulation usage by user (billing period)
SELECT u.email, u.plan,
    SUM(ur.quantity) as total_hours,
    CASE u.plan
        WHEN 'free' THEN 0
        WHEN 'pro' THEN 50
        WHEN 'advanced' THEN 200
    END as limit_hours
FROM usage_records ur
JOIN users u ON ur.user_id = u.id
WHERE ur.period_start <= CURRENT_DATE AND ur.period_end >= CURRENT_DATE
    AND ur.record_type = 'simulation_hours'
GROUP BY u.email, u.plan
ORDER BY total_hours DESC;

-- 5. Most popular materials
SELECT m.name, m.category,
    COUNT(DISTINCT mp.id) as material_pairs,
    COUNT(DISTINCT s.project_id) as projects_using
FROM materials m
LEFT JOIN material_pairs mp ON mp.material_1_id = m.id OR mp.material_2_id = m.id
LEFT JOIN simulations s ON s.input_params::jsonb->>'material_1_id' = m.id::text
    OR s.input_params::jsonb->>'material_2_id' = m.id::text
GROUP BY m.id
ORDER BY projects_using DESC
LIMIT 20;
```

### Performance Benchmarks

| Operation | Target | Method |
|-----------|--------|--------|
| WASM Hertzian solve | <10ms | Browser benchmark (Chrome DevTools) |
| WASM SAM 10K nodes | <2s | Browser benchmark |
| WASM SAM 50K nodes | <10s | Browser benchmark |
| Server SAM 100K nodes | <30s | Server timing, 4 cores |
| Server SAM 500K nodes | <120s | Server timing, 8 cores |
| Server EHL 64×64 grid | <20s | Server timing, multigrid convergence |
| Pressure map WebGL render 100K nodes | 60 FPS | Chrome FPS counter during pan/zoom |
| 3D surface viewer Three.js | 60 FPS | Chrome FPS counter, 50K triangles |
| API: create simulation | <200ms | p95 latency (Prometheus) |
| API: search materials | <100ms | p95 latency with 500+ materials in DB |
| WASM bundle load (cached) | <300ms | Service worker cache hit |
| WASM bundle load (cold) | <2s | CDN delivery, 1.5MB gzipped |
| WebSocket: progress latency | <100ms | Time from worker emit to client receive |

---

## Deployment Architecture

### Infrastructure

```
┌─────────────────────────────────────────────────┐
│                  CloudFront CDN                   │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────┐ │
│  │ Frontend SPA │  │ WASM Bundle │  │ Surface  │ │
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
│ Multi-AZ     │ │ Cluster mode │ │  Surface)    │
└──────────────┘ └──────┬───────┘ └──────────────┘
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ Sim Worker   │ │ Sim Worker   │ │ Sim Worker   │
│ (Rust native)│ │ (Rust native)│ │ (Rust native)│
│ c7g.4xlarge  │ │ c7g.4xlarge  │ │ c7g.4xlarge  │
│ HPA: 1-10    │ │              │ │              │
└──────────────┘ └──────────────┘ └──────────────┘
```

### Scaling Strategy

- **API servers**: HPA at 70% CPU, min 3, max 10
- **Simulation workers**: HPA based on Redis queue depth — scale up when queue > 3 jobs, scale down when idle 10 min
- **Worker instances**: AWS c7g.4xlarge (16 vCPU, 32GB RAM) for multi-threaded FFT and multigrid
- **Database**: RDS PostgreSQL r6g.large with read replica for material/lubricant search
- **Redis**: ElastiCache r6g.large cluster

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
      "ID": "surface-data-keep",
      "Prefix": "surface/",
      "Status": "Enabled",
      "Transitions": [
        { "Days": 90, "StorageClass": "STANDARD_IA" }
      ]
    },
    {
      "ID": "wasm-bundles-versioned",
      "Prefix": "wasm/",
      "Status": "Enabled",
      "NoncurrentVersionExpiration": { "NoncurrentDays": 30 }
    }
  ]
}
```

---

## Post-MVP Roadmap

| Feature | Description | Priority |
|---------|-------------|----------|
| Thermal EHL with Temperature Mapping | Full thermal EHL solver coupling energy equation with Reynolds and elastic deformation, producing temperature distribution maps that affect viscosity and failure modes (scuffing risk zones) | High |
| Rough Surface Contact (Deterministic) | Import measured surface topography from profilometry (AFM, white-light interferometry), run deterministic contact solver accounting for each asperity, predict real contact area and leakage paths | High |
| Parametric Sweep and Optimization | Sweep load, speed, or geometry parameters automatically with parallel job execution, extract trends (contact area vs. load), run constrained optimization to minimize wear or maximize life | High |
| Gear Tooth Contact Analysis | Specialized module for spur, helical, and bevel gears with tooth profile import, load distribution along contact line, scuffing and micropitting risk per ISO 6336, transmission error prediction | Medium |
| Seal Design and Leakage Prediction | Radial lip seal and mechanical face seal analysis with hydrodynamic reverse pumping effects, contact pressure distribution, leakage rate estimation, and wear progression over seal life | Medium |
| Digital Twin Integration | Connect to live sensor data (vibration, temperature, load) from industrial machines, run physics-informed RUL estimation, trigger alerts when simulated damage exceeds thresholds, fleet-level analytics | Medium |
| Fretting Wear and Fatigue | Specialized fretting analysis for oscillating contacts (blade roots, shrink fits), fretting wear maps (regime identification), nucleation of fretting fatigue cracks, life estimation per Ruiz criterion | Low |
| API for Programmatic Access | REST API allowing external tools to submit simulations, retrieve results, and integrate TriboSim into automated design workflows or optimization loops (MATLAB/Python clients) | Low |
