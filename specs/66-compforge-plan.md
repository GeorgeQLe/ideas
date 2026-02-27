# 66. CompForge — Composite Materials Design and Analysis Platform

## Implementation Plan

**MVP Scope:** Browser-based laminate designer with drag-and-drop ply stacking interface for symmetric/balanced/quasi-isotropic layups (up to 100 plies), Classical Lamination Theory (CLT) engine compiled to WebAssembly for client-side ABD matrix calculation and laminate property computation (Ex, Ey, Gxy, νxy, αx, αy), material database with 50 common composite systems (T300/914, IM7/8552, AS4/3501-6, E-glass/epoxy, Kevlar/epoxy) stored in PostgreSQL with A/B-basis allowables from CMH-17, failure criteria implementation (Tsai-Wu, max stress, max strain, Hashin fiber/matrix modes) with first-ply failure envelope generation via WebGL-rendered carpet plots showing laminate strength vs. ply angle and thickness, simple layered shell FEA solver in Rust (Mindlin-Reissner theory, ply-by-ply stress recovery for 2D flat panels up to 10,000 elements), interactive 3D layup visualizer using Three.js showing ply orientations and thickness distribution, PDF report generator with laminate properties, failure envelopes, margin of safety calculations per CMH-17 format, Stripe billing with three tiers (Free / Pro $149/mo / Advanced $349/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| CLT Engine | Rust (native + WASM) | Custom ABD matrix solver, ply-by-ply stress calculations |
| WASM Compilation | `wasm-pack` + `wasm-bindgen` | Client-side CLT for laminates ≤100 plies |
| FEA Solver | Rust (native) | Layered shell elements, progressive failure analysis |
| Optimization Service | Python 3.12 (FastAPI) | Genetic algorithm for stacking sequence, SciPy for sizing |
| Database | PostgreSQL 16 | Projects, users, materials, laminates, simulation results |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | Material datasheets, FEA meshes, simulation results, PDF reports |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| 3D Visualization | Three.js + React Three Fiber | Layup visualizer, ply orientation display |
| Plotting | WebGL 2.0 (custom) | GPU-accelerated carpet plots, failure envelopes |
| Real-time | WebSocket (Axum) | Live FEA progress, optimization status |
| Job Queue | Redis 7 + Tokio tasks | Server-side FEA job management, optimization jobs |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | WASM bundle delivery, static frontend assets |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, error tracking |
| CI/CD | GitHub Actions | WASM build pipeline, Rust tests, Docker image push |

### Key Architecture Decisions

1. **Hybrid client/server CLT with WASM threshold at 100 plies**: Laminates with ≤100 plies (covers 95%+ of typical composite structures) run entirely in the browser via WASM, providing instant ABD matrix calculation and failure analysis with zero server cost. Laminates exceeding 100 plies or requiring full 3D FEA are submitted to the Rust-native server. The 100-ply threshold was chosen because WASM CLT handles this with sub-100ms computation time on modern hardware.

2. **Custom CLT engine in Rust rather than wrapping existing libraries**: Building a custom CLT solver in Rust gives us full control over numerical stability (critical for highly anisotropic laminates), WASM compilation, and integration with failure criteria. Existing Python/MATLAB tools have poor performance for parametric studies requiring 1000+ ABD calculations. Our Rust implementation uses `nalgebra` for 6×6 matrix inversion with explicit checking for near-singular cases (highly unbalanced laminates).

3. **Three.js layup visualizer with per-ply extrusion**: Three.js provides hardware-accelerated 3D rendering for visualizing complex stacking sequences. Each ply is rendered as an extruded polygon with texture mapping for fiber orientation (displayed as directional arrows). This is critical for catching design errors (e.g., accidental [0/90]s instead of [0/±45/90]s) that would be hard to spot in tabular form. WebGL allows rendering 100+ plies at 60fps with real-time pan/zoom/rotate.

4. **WebGL carpet plot renderer separate from Three.js**: Carpet plots (laminate properties vs. design variables like ply angle) require dense 2D color-mapped rendering of 100K+ evaluation points. This uses a dedicated WebGL2 shader pipeline with GPU-interpolated colormaps (viridis, jet) and isoline overlays. Data is computed by WASM in batches and uploaded to GPU textures. This is decoupled from the 3D visualizer to allow independent interaction.

5. **S3 for material database with PostgreSQL metadata catalog**: Material datasheets (SPICE-like `.mat` files with ply properties, typically 5-20KB each) are stored in S3, while PostgreSQL holds searchable metadata (manufacturer, material system, fiber type, A/B-basis allowables, temperature range, moisture content). This allows the material library to scale to 10K+ material systems without bloating the database while enabling fast parametric search via PostgreSQL indexes and full-text search on material names.

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
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
    plan TEXT NOT NULL DEFAULT 'free',  -- free | pro | advanced
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
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

-- Material Database (composite ply properties)
CREATE TABLE materials (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,  -- e.g., "T300/914" or "IM7/8552"
    manufacturer TEXT,  -- e.g., "Hexcel", "Toray"
    material_system TEXT NOT NULL,  -- carbon_epoxy | glass_epoxy | kevlar_epoxy | natural_fiber
    fiber_type TEXT,  -- T300, IM7, AS4, E-glass, S-glass, Kevlar-49
    resin_type TEXT,  -- epoxy_914, epoxy_8552, epoxy_3501-6
    form TEXT NOT NULL,  -- unidirectional | woven | prepreg | dry_fabric
    ply_thickness REAL NOT NULL,  -- meters (typical 0.000125 = 0.125mm)
    density REAL NOT NULL,  -- kg/m³
    -- Elastic properties (aligned with fiber direction = 1, transverse = 2)
    E1 REAL NOT NULL,  -- Longitudinal modulus (Pa)
    E2 REAL NOT NULL,  -- Transverse modulus (Pa)
    G12 REAL NOT NULL,  -- In-plane shear modulus (Pa)
    nu12 REAL NOT NULL,  -- Major Poisson's ratio
    -- Strength allowables (A-basis and B-basis per CMH-17)
    F1t_a REAL,  -- Longitudinal tensile strength, A-basis (Pa)
    F1t_b REAL,  -- Longitudinal tensile strength, B-basis (Pa)
    F1c_a REAL,  -- Longitudinal compressive strength, A-basis (Pa)
    F1c_b REAL,  -- Longitudinal compressive strength, B-basis (Pa)
    F2t_a REAL,  -- Transverse tensile strength, A-basis (Pa)
    F2t_b REAL,  -- Transverse tensile strength, B-basis (Pa)
    F2c_a REAL,  -- Transverse compressive strength, A-basis (Pa)
    F2c_b REAL,  -- Transverse compressive strength, B-basis (Pa)
    F12_a REAL,  -- In-plane shear strength, A-basis (Pa)
    F12_b REAL,  -- In-plane shear strength, B-basis (Pa)
    -- Thermal properties
    alpha1 REAL,  -- CTE in fiber direction (1/K)
    alpha2 REAL,  -- CTE transverse to fiber (1/K)
    -- Environmental
    max_service_temp REAL,  -- Kelvin
    moisture_absorption REAL,  -- % weight at saturation
    datasheet_url TEXT,  -- S3 URL to PDF datasheet
    material_file_url TEXT NOT NULL,  -- S3 URL to .mat file with full property set
    is_builtin BOOLEAN DEFAULT true,
    uploaded_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX materials_name_idx ON materials(name);
CREATE INDEX materials_system_idx ON materials(material_system);
CREATE INDEX materials_name_trgm_idx ON materials USING gin(name gin_trgm_ops);

-- Projects (laminate designs + FEA models)
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    project_type TEXT NOT NULL DEFAULT 'laminate',  -- laminate | fea | optimization
    laminate_data JSONB NOT NULL DEFAULT '{}',  -- Stacking sequence, ply materials, orientation angles
    geometry_data JSONB DEFAULT '{}',  -- For FEA: nodes, elements, boundary conditions
    settings JSONB DEFAULT '{}',  -- Analysis settings, units, coordinate system
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

-- Laminate Analyses (CLT calculations)
CREATE TABLE laminate_analyses (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    analysis_type TEXT NOT NULL,  -- clt_basic | failure_envelope | carpet_plot
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | running | completed | failed
    execution_mode TEXT NOT NULL DEFAULT 'wasm',  -- wasm | server
    ply_count INTEGER NOT NULL DEFAULT 0,
    stacking_sequence JSONB NOT NULL,  -- [{material_id, angle, thickness}]
    load_case JSONB NOT NULL,  -- {Nx, Ny, Nxy, Mx, My, Mxy, deltaT}
    results_url TEXT,  -- S3 URL for result data
    results_summary JSONB,  -- Quick-access: Ex, Ey, Gxy, nu_xy, alpha_x, alpha_y, failure_indices
    error_message TEXT,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    wall_time_ms INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX laminate_analyses_project_idx ON laminate_analyses(project_id);
CREATE INDEX laminate_analyses_user_idx ON laminate_analyses(user_id);
CREATE INDEX laminate_analyses_status_idx ON laminate_analyses(status);

-- FEA Simulations
CREATE TABLE fea_simulations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    analysis_type TEXT NOT NULL,  -- linear_static | buckling | progressive_failure
    status TEXT NOT NULL DEFAULT 'pending',
    element_count INTEGER NOT NULL DEFAULT 0,
    node_count INTEGER NOT NULL DEFAULT 0,
    load_cases JSONB NOT NULL,  -- [{name, forces, moments, boundary_conditions}]
    solver_options JSONB DEFAULT '{}',
    results_url TEXT,  -- S3 URL for result data (nodal displacements, element stresses)
    results_summary JSONB,  -- Max stress, max displacement, failure locations
    error_message TEXT,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    wall_time_ms INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX fea_sims_project_idx ON fea_simulations(project_id);
CREATE INDEX fea_sims_user_idx ON fea_simulations(user_id);
CREATE INDEX fea_sims_status_idx ON fea_simulations(status);

-- Server Jobs (for server-side execution)
CREATE TABLE server_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    job_type TEXT NOT NULL,  -- laminate_analysis | fea_simulation | optimization
    reference_id UUID NOT NULL,  -- ID of the analysis/simulation
    worker_id TEXT,
    cores_allocated INTEGER DEFAULT 1,
    memory_mb INTEGER DEFAULT 4096,
    priority INTEGER DEFAULT 0,  -- Higher = more urgent
    progress_pct REAL DEFAULT 0.0,
    progress_message TEXT,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX server_jobs_type_idx ON server_jobs(job_type);
CREATE INDEX server_jobs_ref_idx ON server_jobs(reference_id);
CREATE INDEX server_jobs_status_idx ON server_jobs(worker_id);

-- Usage Tracking (for billing enforcement)
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,  -- fea_elements | cloud_hours | storage_bytes
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
pub struct Material {
    pub id: Uuid,
    pub name: String,
    pub manufacturer: Option<String>,
    pub material_system: String,
    pub fiber_type: Option<String>,
    pub resin_type: Option<String>,
    pub form: String,
    pub ply_thickness: f64,
    pub density: f64,
    pub e1: f64,
    pub e2: f64,
    pub g12: f64,
    pub nu12: f64,
    pub f1t_a: Option<f64>,
    pub f1t_b: Option<f64>,
    pub f1c_a: Option<f64>,
    pub f1c_b: Option<f64>,
    pub f2t_a: Option<f64>,
    pub f2t_b: Option<f64>,
    pub f2c_a: Option<f64>,
    pub f2c_b: Option<f64>,
    pub f12_a: Option<f64>,
    pub f12_b: Option<f64>,
    pub alpha1: Option<f64>,
    pub alpha2: Option<f64>,
    pub max_service_temp: Option<f64>,
    pub moisture_absorption: Option<f64>,
    pub datasheet_url: Option<String>,
    pub material_file_url: String,
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
    pub project_type: String,
    pub laminate_data: serde_json::Value,
    pub geometry_data: serde_json::Value,
    pub settings: serde_json::Value,
    pub is_public: bool,
    pub forked_from: Option<Uuid>,
    pub thumbnail_url: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct LaminateAnalysis {
    pub id: Uuid,
    pub project_id: Uuid,
    pub user_id: Uuid,
    pub analysis_type: String,
    pub status: String,
    pub execution_mode: String,
    pub ply_count: i32,
    pub stacking_sequence: serde_json::Value,
    pub load_case: serde_json::Value,
    pub results_url: Option<String>,
    pub results_summary: Option<serde_json::Value>,
    pub error_message: Option<String>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub wall_time_ms: Option<i32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct FeaSimulation {
    pub id: Uuid,
    pub project_id: Uuid,
    pub user_id: Uuid,
    pub analysis_type: String,
    pub status: String,
    pub element_count: i32,
    pub node_count: i32,
    pub load_cases: serde_json::Value,
    pub solver_options: serde_json::Value,
    pub results_url: Option<String>,
    pub results_summary: Option<serde_json::Value>,
    pub error_message: Option<String>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub wall_time_ms: Option<i32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct Ply {
    pub material_id: Uuid,
    pub angle: f64,  // degrees, 0° = fiber along x-axis
    pub thickness: f64,  // meters (often overridden by material default)
}

#[derive(Debug, Deserialize, Serialize)]
pub struct LoadCase {
    pub nx: f64,  // N/m (force per unit width)
    pub ny: f64,
    pub nxy: f64,
    pub mx: f64,  // N·m/m (moment per unit width)
    pub my: f64,
    pub mxy: f64,
    pub delta_t: f64,  // Temperature change from cure (K)
}

#[derive(Debug, Deserialize, Serialize)]
pub struct CltResults {
    pub abd_matrix: [[f64; 6]; 6],  // 6×6 ABD matrix
    pub abd_inverse: [[f64; 6]; 6],
    pub laminate_thickness: f64,  // meters
    pub ex: f64,  // Effective moduli (Pa)
    pub ey: f64,
    pub gxy: f64,
    pub nu_xy: f64,
    pub alpha_x: f64,  // Effective CTE (1/K)
    pub alpha_y: f64,
    pub mid_plane_strains: [f64; 3],  // [εx, εy, γxy] at mid-plane
    pub mid_plane_curvatures: [f64; 3],  // [κx, κy, κxy]
    pub ply_stresses: Vec<PlyStress>,  // Per-ply stresses at top/bottom
    pub failure_indices: Vec<FailureIndex>,  // Per-ply failure indices
}

#[derive(Debug, Deserialize, Serialize)]
pub struct PlyStress {
    pub ply_index: usize,
    pub z_top: f64,
    pub z_bottom: f64,
    pub sigma1_top: f64,
    pub sigma2_top: f64,
    pub tau12_top: f64,
    pub sigma1_bottom: f64,
    pub sigma2_bottom: f64,
    pub tau12_bottom: f64,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct FailureIndex {
    pub ply_index: usize,
    pub criterion: String,  // tsai_wu | max_stress | max_strain | hashin
    pub index: f64,  // <1.0 = safe, >=1.0 = failure
    pub failure_mode: Option<String>,  // fiber_tension | matrix_compression, etc.
}
```

---

## CLT Solver Architecture Deep-Dive

### Governing Equations and Discretization

CompForge's core CLT engine implements **Classical Lamination Theory**, the industry-standard analytical framework for laminated composite analysis. For a laminate with `n` plies, each ply `k` has elastic properties (E₁, E₂, G₁₂, ν₁₂) and orientation angle θ. The laminate constitutive relation is:

```
┌    ┐   ┌       ┐ ┌   ┐
│ N  │   │ A   B │ │ ε⁰│
│    │ = │       │ │   │
│ M  │   │ B   D │ │ κ │
└    ┘   └       ┘ └   ┘
```

Where:
- **N** = [Nx, Ny, Nxy]ᵀ: In-plane force resultants (N/m)
- **M** = [Mx, My, Mxy]ᵀ: Moment resultants (N·m/m)
- **ε⁰** = [εx⁰, εy⁰, γxy⁰]ᵀ: Mid-plane strains
- **κ** = [κx, κy, κxy]ᵀ: Curvatures
- **A** (3×3): Extensional stiffness matrix
- **B** (3×3): Coupling stiffness matrix (zero for symmetric laminates)
- **D** (3×3): Bending stiffness matrix

**ABD Matrix Construction:**

For each ply k at position z_k from the mid-plane:

1. **Ply reduced stiffness matrix Q̄** (in global coordinates after rotation by θ):
```
Q̄₁₁ = Q₁₁cos⁴θ + 2(Q₁₂+2Q₆₆)sin²θcos²θ + Q₂₂sin⁴θ
Q̄₁₂ = (Q₁₁+Q₂₂-4Q₆₆)sin²θcos²θ + Q₁₂(sin⁴θ+cos⁴θ)
Q̄₂₂ = Q₁₁sin⁴θ + 2(Q₁₂+2Q₆₆)sin²θcos²θ + Q₂₂cos⁴θ
Q̄₁₆ = (Q₁₁-Q₁₂-2Q₆₆)sinθcos³θ + (Q₁₂-Q₂₂+2Q₆₆)sin³θcosθ
Q̄₂₆ = (Q₁₁-Q₁₂-2Q₆₆)sin³θcosθ + (Q₁₂-Q₂₂+2Q₆₆)sinθcos³θ
Q̄₆₆ = (Q₁₁+Q₂₂-2Q₁₂-2Q₆₆)sin²θcos²θ + Q₆₆(sin⁴θ+cos⁴θ)

where Qᵢⱼ are the ply stiffness in material coordinates:
Q₁₁ = E₁/(1-ν₁₂ν₂₁)
Q₂₂ = E₂/(1-ν₁₂ν₂₁)
Q₁₂ = ν₁₂E₂/(1-ν₁₂ν₂₁)
Q₆₆ = G₁₂
```

2. **Sum over all plies:**
```
Aᵢⱼ = Σ Q̄ᵢⱼᵏ(zₖ - zₖ₋₁)                 (extensional stiffness)
Bᵢⱼ = ½Σ Q̄ᵢⱼᵏ(zₖ² - zₖ₋₁²)                (coupling stiffness)
Dᵢⱼ = ⅓Σ Q̄ᵢⱼᵏ(zₖ³ - zₖ₋₁³)                (bending stiffness)
```

3. **Invert ABD matrix** to get compliance:
```
┌   ┐   ┌       ┐⁻¹ ┌   ┐
│ ε⁰│   │ A   B │   │ N │
│   │ = │       │   │   │
│ κ │   │ B   D │   │ M │
└   ┘   └       ┘   └   ┘
```

The 6×6 inversion is done via LU decomposition with explicit singularity checks (laminates with all 0° or all 90° plies can have near-zero terms leading to conditioning issues).

**Effective Laminate Properties:**

From the inverted ABD matrix (call it **a** for compliance), extract:
```
Ex = 1/(h·a₁₁)    (effective longitudinal modulus)
Ey = 1/(h·a₂₂)    (effective transverse modulus)
Gxy = 1/(h·a₆₆)   (effective shear modulus)
νxy = -a₁₂/a₁₁    (major Poisson's ratio)

where h = total laminate thickness
```

**Thermal Strains:**

For a temperature change ΔT, each ply has thermal strains:
```
εᵗʰ = [α₁, α₂, 0]ᵀ · ΔT    (in material coordinates)
```

These are transformed to global coordinates and integrated to produce thermal force/moment resultants:
```
Nᵗʰ = Σ Q̄ᵏ·αᵏ·ΔT·(zₖ - zₖ₋₁)
Mᵗʰ = ½Σ Q̄ᵏ·αᵏ·ΔT·(zₖ² - zₖ₋₁²)
```

**Ply-by-Ply Stress Recovery:**

Once mid-plane strains ε⁰ and curvatures κ are known, compute strains at any z:
```
ε(z) = ε⁰ + z·κ
```

Transform to ply material coordinates and compute stresses:
```
σ = Q·ε    (in material coordinates 1-2)
```

This gives σ₁, σ₂, τ₁₂ for each ply at top and bottom interfaces.

**Failure Criteria:**

1. **Max Stress:**
```
FI = max(|σ₁|/F₁, |σ₂|/F₂, |τ₁₂|/F₁₂)
where F₁ = F₁ₜ if σ₁>0 else F₁c, etc.
```

2. **Tsai-Wu (interactive quadratic):**
```
F₁σ₁ + F₂σ₂ + F₁₁σ₁² + F₂₂σ₂² + F₆₆τ₁₂² + 2F₁₂σ₁σ₂ < 1
where:
F₁ = 1/F₁ₜ - 1/F₁c
F₂ = 1/F₂ₜ - 1/F₂c
F₁₁ = 1/(F₁ₜ·F₁c)
F₂₂ = 1/(F₂ₜ·F₂c)
F₆₆ = 1/F₁₂²
F₁₂ = -0.5·√(F₁₁·F₂₂)    (interaction term)
```

3. **Hashin (separate fiber/matrix modes):**
```
Fiber tension (σ₁≥0):     (σ₁/F₁ₜ)² + (τ₁₂/F₁₂)² < 1
Fiber compression (σ₁<0): (σ₁/F₁c)² < 1
Matrix tension (σ₂≥0):    (σ₂/F₂ₜ)² + (τ₁₂/F₁₂)² < 1
Matrix compression (σ₂<0): (σ₂/2S₂₃)² + [(F₂c/2S₂₃)²-1]·(σ₂/F₂c) + (τ₁₂/F₁₂)² < 1
```

### Client/Server Split (WASM Threshold)

```
Laminate designed → Ply count extracted
    │
    ├── ≤100 plies → WASM CLT (browser)
    │   ├── Instant ABD calculation (<50ms)
    │   ├── Results displayed immediately
    │   └── No server cost
    │
    └── >100 plies OR FEA requested → Server solver (Rust native)
        ├── Job queued via Redis
        ├── Worker picks up, runs native solver
        ├── Progress streamed via WebSocket
        └── Results stored in S3, URL returned
```

The 100-ply threshold was chosen because:
- WASM CLT with `nalgebra` handles 100-ply ABD inversion in <50ms on modern hardware
- 100 plies covers: aerospace wing skins (40-80 plies), pressure vessels (30-60 plies), wind turbine blades (50-90 plies)
- Above 100 plies: thick structures requiring 3D FEA, or optimization sweeps with 1000+ evaluations → need server compute

### WASM Compilation Pipeline

```toml
# clt-wasm/Cargo.toml
[package]
name = "compforge-clt-wasm"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
wasm-bindgen = "0.2"
serde = { version = "1", features = ["derive"] }
serde-wasm-bindgen = "0.6"
js-sys = "0.3"
nalgebra = "0.32"
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
name: Build WASM CLT Solver
on:
  push:
    paths: ['clt-wasm/**', 'clt-core/**']
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
      - run: cd clt-wasm && wasm-pack build --target web --release
      - run: wasm-opt -Oz clt-wasm/pkg/compforge_clt_wasm_bg.wasm -o clt-wasm/pkg/compforge_clt_wasm_bg.wasm
      - name: Upload to S3/CDN
        run: aws s3 cp clt-wasm/pkg/ s3://compforge-wasm/v${{ github.sha }}/ --recursive
```

---

## Architecture Deep-Dives

### 1. Laminate Analysis API Handler (Rust/Axum)

The primary endpoint receives a laminate design, validates the stacking sequence, decides between WASM and server execution, and for server execution enqueues a job with Redis and streams progress via WebSocket.

```rust
// src/api/handlers/laminate.rs

use axum::{
    extract::{Path, State, Json},
    http::StatusCode,
    response::IntoResponse,
};
use uuid::Uuid;
use crate::{
    db::models::{LaminateAnalysis, Ply, LoadCase},
    state::AppState,
    auth::Claims,
    error::ApiError,
};

#[derive(serde::Deserialize)]
pub struct CreateLaminateAnalysisRequest {
    pub analysis_type: String,  // clt_basic | failure_envelope | carpet_plot
    pub stacking_sequence: Vec<Ply>,
    pub load_case: LoadCase,
    pub failure_criteria: Vec<String>,  // ["tsai_wu", "max_stress", "hashin"]
}

pub async fn create_laminate_analysis(
    State(state): State<AppState>,
    claims: Claims,
    Path(project_id): Path<Uuid>,
    Json(req): Json<CreateLaminateAnalysisRequest>,
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

    // 2. Validate stacking sequence
    if req.stacking_sequence.is_empty() {
        return Err(ApiError::BadRequest("Stacking sequence cannot be empty"));
    }
    if req.stacking_sequence.len() > 500 {
        return Err(ApiError::BadRequest("Maximum 500 plies supported"));
    }

    // 3. Check plan limits
    let user = sqlx::query_as!(
        crate::db::models::User,
        "SELECT * FROM users WHERE id = $1",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    let ply_count = req.stacking_sequence.len() as i32;
    if user.plan == "free" && ply_count > 10 {
        return Err(ApiError::PlanLimit(
            "Free plan supports laminates up to 10 plies. Upgrade to Pro for 100 plies."
        ));
    }

    // 4. Determine execution mode
    let execution_mode = if ply_count <= 100 && req.analysis_type != "carpet_plot" {
        "wasm"
    } else {
        "server"
    };

    // 5. Create analysis record
    let analysis = sqlx::query_as!(
        LaminateAnalysis,
        r#"INSERT INTO laminate_analyses
            (project_id, user_id, analysis_type, status, execution_mode,
             ply_count, stacking_sequence, load_case)
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8)
        RETURNING *"#,
        project_id,
        claims.user_id,
        req.analysis_type,
        if execution_mode == "wasm" { "completed" } else { "pending" },
        execution_mode,
        ply_count,
        serde_json::to_value(&req.stacking_sequence)?,
        serde_json::to_value(&req.load_case)?,
    )
    .fetch_one(&state.db)
    .await?;

    // 6. For server execution, enqueue job
    if execution_mode == "server" {
        let job = sqlx::query!(
            r#"INSERT INTO server_jobs (job_type, reference_id, priority)
            VALUES ($1, $2, $3) RETURNING id"#,
            "laminate_analysis",
            analysis.id,
            if user.plan == "advanced" { 10 } else { 0 },
        )
        .fetch_one(&state.db)
        .await?;

        // Publish to Redis job queue
        let mut conn = state.redis.get_async_connection().await?;
        redis::cmd("LPUSH")
            .arg("jobs:laminate")
            .arg(job.id.to_string())
            .query_async(&mut conn)
            .await?;
    }

    Ok((StatusCode::CREATED, Json(analysis)))
}

pub async fn get_laminate_analysis(
    State(state): State<AppState>,
    claims: Claims,
    Path((project_id, analysis_id)): Path<(Uuid, Uuid)>,
) -> Result<Json<LaminateAnalysis>, ApiError> {
    let analysis = sqlx::query_as!(
        LaminateAnalysis,
        "SELECT la.* FROM laminate_analyses la
         JOIN projects p ON la.project_id = p.id
         WHERE la.id = $1 AND la.project_id = $2
         AND (p.owner_id = $3 OR p.org_id IN (
             SELECT org_id FROM org_members WHERE user_id = $3
         ))",
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

### 2. CLT Core Solver (Rust — shared between WASM and native)

The core CLT solver that builds ABD matrices, performs ply stress recovery, and evaluates failure criteria. This code compiles to both native (server) and WASM (browser) targets.

```rust
// clt-core/src/abd.rs

use nalgebra::{Matrix3, Matrix6, Vector3, Vector6};

pub struct CltSolver {
    pub plies: Vec<Ply>,
    pub abd: Matrix6<f64>,
    pub abd_inv: Matrix6<f64>,
    pub total_thickness: f64,
}

#[derive(Clone, Debug)]
pub struct Ply {
    pub material: Material,
    pub angle_deg: f64,
    pub thickness: f64,
}

#[derive(Clone, Debug)]
pub struct Material {
    pub e1: f64,
    pub e2: f64,
    pub g12: f64,
    pub nu12: f64,
    pub alpha1: f64,
    pub alpha2: f64,
    pub f1t: f64,
    pub f1c: f64,
    pub f2t: f64,
    pub f2c: f64,
    pub f12: f64,
}

impl CltSolver {
    pub fn new(plies: Vec<Ply>) -> Self {
        let mut solver = Self {
            plies,
            abd: Matrix6::zeros(),
            abd_inv: Matrix6::zeros(),
            total_thickness: 0.0,
        };
        solver.compute_abd();
        solver
    }

    /// Compute ABD matrix from ply stacking sequence
    fn compute_abd(&mut self) {
        let n = self.plies.len();
        let mut z_positions = vec![0.0; n + 1];

        // Compute z-coordinates (start from -h/2)
        self.total_thickness = self.plies.iter().map(|p| p.thickness).sum();
        z_positions[0] = -self.total_thickness / 2.0;
        for i in 0..n {
            z_positions[i + 1] = z_positions[i] + self.plies[i].thickness;
        }

        // Initialize ABD matrix
        let mut a = Matrix3::zeros();
        let mut b = Matrix3::zeros();
        let mut d = Matrix3::zeros();

        for (k, ply) in self.plies.iter().enumerate() {
            let q_bar = Self::compute_q_bar(&ply.material, ply.angle_deg);
            let z0 = z_positions[k];
            let z1 = z_positions[k + 1];

            // A matrix (extensional)
            a += q_bar * (z1 - z0);

            // B matrix (coupling)
            b += q_bar * 0.5 * (z1.powi(2) - z0.powi(2));

            // D matrix (bending)
            d += q_bar * (1.0 / 3.0) * (z1.powi(3) - z0.powi(3));
        }

        // Assemble 6×6 ABD matrix
        self.abd.fixed_view_mut::<3, 3>(0, 0).copy_from(&a);
        self.abd.fixed_view_mut::<3, 3>(0, 3).copy_from(&b);
        self.abd.fixed_view_mut::<3, 3>(3, 0).copy_from(&b);
        self.abd.fixed_view_mut::<3, 3>(3, 3).copy_from(&d);

        // Invert ABD matrix
        self.abd_inv = self.abd.try_inverse()
            .expect("ABD matrix is singular - check laminate configuration");
    }

    /// Compute reduced stiffness matrix Q̄ in global coordinates
    fn compute_q_bar(mat: &Material, angle_deg: f64) -> Matrix3<f64> {
        let theta = angle_deg.to_radians();
        let c = theta.cos();
        let s = theta.sin();
        let c2 = c * c;
        let s2 = s * s;
        let c4 = c2 * c2;
        let s4 = s2 * s2;

        // Reduced stiffness in material coordinates
        let nu21 = mat.nu12 * mat.e2 / mat.e1;
        let denom = 1.0 - mat.nu12 * nu21;
        let q11 = mat.e1 / denom;
        let q22 = mat.e2 / denom;
        let q12 = mat.nu12 * mat.e2 / denom;
        let q66 = mat.g12;

        // Rotate to global coordinates
        let q_bar_11 = q11 * c4 + 2.0 * (q12 + 2.0 * q66) * s2 * c2 + q22 * s4;
        let q_bar_12 = (q11 + q22 - 4.0 * q66) * s2 * c2 + q12 * (s4 + c4);
        let q_bar_22 = q11 * s4 + 2.0 * (q12 + 2.0 * q66) * s2 * c2 + q22 * c4;
        let q_bar_16 = (q11 - q12 - 2.0 * q66) * s * c * c2
                     + (q12 - q22 + 2.0 * q66) * s * c * s2;
        let q_bar_26 = (q11 - q12 - 2.0 * q66) * s * c * s2
                     + (q12 - q22 + 2.0 * q66) * s * c * c2;
        let q_bar_66 = (q11 + q22 - 2.0 * q12 - 2.0 * q66) * s2 * c2 + q66 * (s4 + c4);

        Matrix3::new(
            q_bar_11, q_bar_12, q_bar_16,
            q_bar_12, q_bar_22, q_bar_26,
            q_bar_16, q_bar_26, q_bar_66,
        )
    }

    /// Solve for mid-plane strains and curvatures given loads
    pub fn solve(&self, loads: &LoadCase) -> CltResults {
        // Construct load vector [Nx, Ny, Nxy, Mx, My, Mxy]
        let load_vec = Vector6::new(
            loads.nx, loads.ny, loads.nxy,
            loads.mx, loads.my, loads.mxy,
        );

        // Solve for strains and curvatures
        let strain_curv = self.abd_inv * load_vec;
        let mid_strains = Vector3::new(strain_curv[0], strain_curv[1], strain_curv[2]);
        let curvatures = Vector3::new(strain_curv[3], strain_curv[4], strain_curv[5]);

        // Compute effective laminate properties
        let h = self.total_thickness;
        let a_inv = self.abd_inv.fixed_view::<3, 3>(0, 0);
        let ex = 1.0 / (h * a_inv[(0, 0)]);
        let ey = 1.0 / (h * a_inv[(1, 1)]);
        let gxy = 1.0 / (h * a_inv[(2, 2)]);
        let nu_xy = -a_inv[(0, 1)] / a_inv[(0, 0)];

        // Effective CTE (simplified - assumes no thermal gradients)
        let alpha_x = 0.0;  // TODO: compute from thermal loads
        let alpha_y = 0.0;

        // Ply-by-ply stress recovery
        let ply_stresses = self.compute_ply_stresses(&mid_strains, &curvatures);

        CltResults {
            abd_matrix: self.abd.into(),
            abd_inverse: self.abd_inv.into(),
            laminate_thickness: self.total_thickness,
            ex, ey, gxy, nu_xy,
            alpha_x, alpha_y,
            mid_plane_strains: [mid_strains[0], mid_strains[1], mid_strains[2]],
            mid_plane_curvatures: [curvatures[0], curvatures[1], curvatures[2]],
            ply_stresses,
            failure_indices: Vec::new(),  // Computed in next step
        }
    }

    /// Compute stresses in each ply
    fn compute_ply_stresses(&self, eps0: &Vector3<f64>, kappa: &Vector3<f64>) -> Vec<PlyStress> {
        let mut z = -self.total_thickness / 2.0;
        let mut stresses = Vec::new();

        for (i, ply) in self.plies.iter().enumerate() {
            let z_bottom = z;
            let z_top = z + ply.thickness;

            // Strains at top and bottom
            let eps_bottom = eps0 + kappa * z_bottom;
            let eps_top = eps0 + kappa * z_top;

            // Transform to material coordinates and compute stresses
            let sigma_bottom = Self::compute_ply_stress(ply, &eps_bottom);
            let sigma_top = Self::compute_ply_stress(ply, &eps_top);

            stresses.push(PlyStress {
                ply_index: i,
                z_top, z_bottom,
                sigma1_top: sigma_top[0],
                sigma2_top: sigma_top[1],
                tau12_top: sigma_top[2],
                sigma1_bottom: sigma_bottom[0],
                sigma2_bottom: sigma_bottom[1],
                tau12_bottom: sigma_bottom[2],
            });

            z = z_top;
        }

        stresses
    }

    fn compute_ply_stress(ply: &Ply, eps_global: &Vector3<f64>) -> Vector3<f64> {
        let theta = ply.angle_deg.to_radians();
        let c = theta.cos();
        let s = theta.sin();

        // Transform strain to material coordinates
        let t_eps = Matrix3::new(
            c*c, s*s, 2.0*s*c,
            s*s, c*c, -2.0*s*c,
            -s*c, s*c, c*c - s*s,
        );
        let eps_material = t_eps * eps_global;

        // Compute stress in material coordinates
        let mat = &ply.material;
        let nu21 = mat.nu12 * mat.e2 / mat.e1;
        let denom = 1.0 - mat.nu12 * nu21;
        let q11 = mat.e1 / denom;
        let q22 = mat.e2 / denom;
        let q12 = mat.nu12 * mat.e2 / denom;
        let q66 = mat.g12;

        Vector3::new(
            q11 * eps_material[0] + q12 * eps_material[1],
            q12 * eps_material[0] + q22 * eps_material[1],
            q66 * eps_material[2],
        )
    }
}

#[derive(Debug)]
pub struct CltResults {
    pub abd_matrix: [[f64; 6]; 6],
    pub abd_inverse: [[f64; 6]; 6],
    pub laminate_thickness: f64,
    pub ex: f64,
    pub ey: f64,
    pub gxy: f64,
    pub nu_xy: f64,
    pub alpha_x: f64,
    pub alpha_y: f64,
    pub mid_plane_strains: [f64; 3],
    pub mid_plane_curvatures: [f64; 3],
    pub ply_stresses: Vec<PlyStress>,
    pub failure_indices: Vec<FailureIndex>,
}

#[derive(Debug)]
pub struct PlyStress {
    pub ply_index: usize,
    pub z_top: f64,
    pub z_bottom: f64,
    pub sigma1_top: f64,
    pub sigma2_top: f64,
    pub tau12_top: f64,
    pub sigma1_bottom: f64,
    pub sigma2_bottom: f64,
    pub tau12_bottom: f64,
}

#[derive(Debug)]
pub struct FailureIndex {
    pub ply_index: usize,
    pub criterion: String,
    pub index: f64,
    pub failure_mode: Option<String>,
}

#[derive(Debug)]
pub struct LoadCase {
    pub nx: f64, pub ny: f64, pub nxy: f64,
    pub mx: f64, pub my: f64, pub mxy: f64,
}
```

### 3. Three.js Layup Visualizer (React + React Three Fiber)

The frontend 3D visualizer that renders composite layups with per-ply orientation arrows and interactive inspection.

```typescript
// frontend/src/components/LayupVisualizer.tsx

import { Canvas, useFrame } from '@react-three/fiber';
import { OrbitControls, Text } from '@react-three/drei';
import { useMemo, useRef } from 'react';
import * as THREE from 'three';
import { useLayupStore } from '../stores/layupStore';

interface PlyMeshProps {
  ply: {
    material_id: string;
    angle: number;
    thickness: number;
    z_position: number;
  };
  index: number;
  color: string;
  isSelected: boolean;
  onClick: () => void;
}

function PlyMesh({ ply, index, color, isSelected, onClick }: PlyMeshProps) {
  const meshRef = useRef<THREE.Mesh>(null);
  const arrowRef = useRef<THREE.Group>(null);

  // Animate selection
  useFrame(() => {
    if (meshRef.current) {
      const target = isSelected ? 1.1 : 1.0;
      meshRef.current.scale.x += (target - meshRef.current.scale.x) * 0.1;
      meshRef.current.scale.y += (target - meshRef.current.scale.y) * 0.1;
    }
  });

  // Create fiber direction arrows
  const arrows = useMemo(() => {
    const arrowCount = 8;
    const spacing = 0.2;
    const angleRad = (ply.angle * Math.PI) / 180;
    const positions: [number, number, number][] = [];

    for (let i = 0; i < arrowCount; i++) {
      const x = (i - arrowCount / 2) * spacing * Math.cos(angleRad);
      const y = (i - arrowCount / 2) * spacing * Math.sin(angleRad);
      positions.push([x, y, ply.z_position + ply.thickness / 2]);
    }

    return positions;
  }, [ply.angle, ply.z_position, ply.thickness]);

  return (
    <group>
      {/* Ply slab */}
      <mesh
        ref={meshRef}
        position={[0, 0, ply.z_position + ply.thickness / 2]}
        onClick={onClick}
      >
        <boxGeometry args={[1.5, 1.5, ply.thickness]} />
        <meshStandardMaterial
          color={color}
          transparent
          opacity={isSelected ? 0.9 : 0.7}
          emissive={isSelected ? color : '#000000'}
          emissiveIntensity={isSelected ? 0.3 : 0}
        />
      </mesh>

      {/* Fiber direction arrows */}
      <group ref={arrowRef}>
        {arrows.map((pos, i) => (
          <mesh key={i} position={pos} rotation={[0, 0, (ply.angle * Math.PI) / 180]}>
            <coneGeometry args={[0.02, 0.08, 8]} />
            <meshBasicMaterial color="#ffffff" />
          </mesh>
        ))}
      </group>

      {/* Ply label */}
      <Text
        position={[0.85, 0.85, ply.z_position + ply.thickness / 2]}
        fontSize={0.05}
        color="#ffffff"
        anchorX="left"
        anchorY="middle"
      >
        {`Ply ${index + 1}: ${ply.angle}°`}
      </Text>
    </group>
  );
}

export function LayupVisualizer() {
  const { plies, selectedPlyIndex, selectPly, materials } = useLayupStore();

  const plyColors = useMemo(() => {
    // Color code by angle
    return plies.map(ply => {
      const normalizedAngle = ((ply.angle % 180) + 180) % 180;
      const hue = normalizedAngle * 2; // 0° = red, 90° = cyan
      return `hsl(${hue}, 70%, 50%)`;
    });
  }, [plies]);

  const totalThickness = useMemo(() => {
    return plies.reduce((sum, p) => sum + p.thickness, 0);
  }, [plies]);

  return (
    <div className="w-full h-full bg-gray-900">
      <Canvas camera={{ position: [2, 2, 2], fov: 50 }}>
        <ambientLight intensity={0.5} />
        <directionalLight position={[10, 10, 5]} intensity={1} />
        <directionalLight position={[-10, -10, -5]} intensity={0.3} />

        {/* Coordinate system axes */}
        <axesHelper args={[1]} />

        {/* Ground plane reference */}
        <mesh rotation={[-Math.PI / 2, 0, 0]} position={[0, 0, -totalThickness / 2 - 0.01]}>
          <planeGeometry args={[2, 2]} />
          <meshBasicMaterial color="#333333" transparent opacity={0.3} />
        </mesh>

        {/* Render all plies */}
        {plies.map((ply, index) => {
          const z_position = plies
            .slice(0, index)
            .reduce((sum, p) => sum + p.thickness, -totalThickness / 2);

          return (
            <PlyMesh
              key={index}
              ply={{ ...ply, z_position }}
              index={index}
              color={plyColors[index]}
              isSelected={selectedPlyIndex === index}
              onClick={() => selectPly(index)}
            />
          );
        })}

        <OrbitControls makeDefault />
      </Canvas>

      {/* Info panel */}
      <div className="absolute top-4 right-4 bg-gray-800 text-white p-4 rounded-lg shadow-lg">
        <h3 className="text-lg font-bold mb-2">Laminate Summary</h3>
        <div className="text-sm space-y-1">
          <div>Total Plies: {plies.length}</div>
          <div>Total Thickness: {(totalThickness * 1000).toFixed(2)} mm</div>
          {selectedPlyIndex !== null && (
            <div className="mt-3 pt-3 border-t border-gray-600">
              <div className="font-semibold">Selected Ply {selectedPlyIndex + 1}</div>
              <div>Angle: {plies[selectedPlyIndex].angle}°</div>
              <div>Thickness: {(plies[selectedPlyIndex].thickness * 1000).toFixed(3)} mm</div>
            </div>
          )}
        </div>
      </div>
    </div>
  );
}
```

### 4. Server-Side Analysis Worker (Rust + Redis)

Background worker that picks up server-side laminate analysis jobs from Redis, runs the CLT solver, computes failure envelopes/carpet plots, streams progress via WebSocket, and stores results in S3.

```rust
// src/workers/laminate_worker.rs

use std::sync::Arc;
use aws_sdk_s3::Client as S3Client;
use redis::AsyncCommands;
use sqlx::PgPool;
use uuid::Uuid;

use crate::clt::{
    abd::{CltSolver, Ply, Material, LoadCase},
    failure::{evaluate_tsai_wu, evaluate_max_stress, evaluate_hashin},
};
use crate::db::models::LaminateAnalysis;
use crate::websocket::ProgressBroadcaster;

pub struct LaminateWorker {
    db: PgPool,
    redis: redis::Client,
    s3: S3Client,
    broadcaster: Arc<ProgressBroadcaster>,
}

impl LaminateWorker {
    pub async fn run(&self) -> anyhow::Result<()> {
        let mut conn = self.redis.get_async_connection().await?;
        tracing::info!("Laminate worker started, listening for jobs...");

        loop {
            // Block-pop from Redis job queue
            let job_id: Option<(String, String)> = conn.brpop("jobs:laminate", 30).await?;
            let Some((_, job_id)) = job_id else { continue };
            let job_id: Uuid = job_id.parse()?;

            if let Err(e) = self.process_job(job_id).await {
                tracing::error!("Job {job_id} failed: {e}");
                self.mark_failed(job_id, &e.to_string()).await?;
            }
        }
    }

    async fn process_job(&self, job_id: Uuid) -> anyhow::Result<()> {
        // 1. Fetch job and analysis details
        let analysis = sqlx::query_as!(
            LaminateAnalysis,
            "UPDATE laminate_analyses SET status = 'running', started_at = NOW()
             WHERE id = $1 RETURNING *",
            job_id
        )
        .fetch_one(&self.db)
        .await?;

        // 2. Load materials from database
        let plies: Vec<Ply> = self.load_plies(&analysis.stacking_sequence).await?;
        let load_case: LoadCase = serde_json::from_value(analysis.load_case.clone())?;

        // 3. Run analysis based on type
        let results = match analysis.analysis_type.as_str() {
            "clt_basic" => {
                self.run_clt_basic(&plies, &load_case).await?
            }
            "failure_envelope" => {
                self.run_failure_envelope(&plies, job_id).await?
            }
            "carpet_plot" => {
                self.run_carpet_plot(&plies, job_id).await?
            }
            _ => anyhow::bail!("Unknown analysis type: {}", analysis.analysis_type),
        };

        // 4. Upload results to S3
        let s3_key = format!("analyses/{}/{}.json", analysis.project_id, analysis.id);
        let results_json = serde_json::to_vec(&results)?;
        self.s3.put_object()
            .bucket("compforge-results")
            .key(&s3_key)
            .body(results_json.into())
            .content_type("application/json")
            .send()
            .await?;

        let results_url = format!("s3://compforge-results/{}", s3_key);

        // 5. Update database with completed status
        let summary = serde_json::json!({
            "ex": results.ex,
            "ey": results.ey,
            "gxy": results.gxy,
            "nu_xy": results.nu_xy,
            "thickness": results.laminate_thickness,
        });

        sqlx::query!(
            "UPDATE laminate_analyses SET status = 'completed', results_url = $2,
             results_summary = $3, completed_at = NOW(),
             wall_time_ms = EXTRACT(EPOCH FROM (NOW() - started_at))::int * 1000
             WHERE id = $1",
            analysis.id, results_url, summary
        )
        .execute(&self.db)
        .await?;

        // 6. Notify client via WebSocket
        self.broadcaster.send(analysis.id, serde_json::json!({
            "status": "completed",
            "results_url": results_url,
        }))?;

        tracing::info!("Job {job_id} completed successfully");
        Ok(())
    }

    async fn load_plies(&self, stacking_seq: &serde_json::Value) -> anyhow::Result<Vec<Ply>> {
        let ply_specs: Vec<crate::db::models::Ply> = serde_json::from_value(stacking_seq.clone())?;
        let mut plies = Vec::new();

        for spec in ply_specs {
            let mat = sqlx::query_as!(
                crate::db::models::Material,
                "SELECT * FROM materials WHERE id = $1",
                spec.material_id
            )
            .fetch_one(&self.db)
            .await?;

            plies.push(Ply {
                material: Material {
                    e1: mat.e1,
                    e2: mat.e2,
                    g12: mat.g12,
                    nu12: mat.nu12,
                    alpha1: mat.alpha1.unwrap_or(0.0),
                    alpha2: mat.alpha2.unwrap_or(0.0),
                    f1t: mat.f1t_b.unwrap_or(1e9),
                    f1c: mat.f1c_b.unwrap_or(1e9),
                    f2t: mat.f2t_b.unwrap_or(1e9),
                    f2c: mat.f2c_b.unwrap_or(1e9),
                    f12: mat.f12_b.unwrap_or(1e9),
                },
                angle_deg: spec.angle,
                thickness: spec.thickness,
            });
        }

        Ok(plies)
    }

    async fn run_clt_basic(&self, plies: &[Ply], load_case: &LoadCase) -> anyhow::Result<CltResults> {
        let solver = CltSolver::new(plies.to_vec());
        let mut results = solver.solve(load_case);

        // Evaluate failure criteria for all plies
        for stress in &results.ply_stresses {
            let ply = &plies[stress.ply_index];

            // Check both top and bottom of ply
            for (suffix, sigma1, sigma2, tau12) in [
                ("_top", stress.sigma1_top, stress.sigma2_top, stress.tau12_top),
                ("_bottom", stress.sigma1_bottom, stress.sigma2_bottom, stress.tau12_bottom),
            ] {
                let tsai_wu_fi = evaluate_tsai_wu(sigma1, sigma2, tau12, &ply.material);
                results.failure_indices.push(FailureIndex {
                    ply_index: stress.ply_index,
                    criterion: format!("tsai_wu{}", suffix),
                    index: tsai_wu_fi,
                    failure_mode: if tsai_wu_fi >= 1.0 { Some("combined".to_string()) } else { None },
                });

                let (max_stress_fi, mode) = evaluate_max_stress(sigma1, sigma2, tau12, &ply.material);
                results.failure_indices.push(FailureIndex {
                    ply_index: stress.ply_index,
                    criterion: format!("max_stress{}", suffix),
                    index: max_stress_fi,
                    failure_mode: mode,
                });
            }
        }

        Ok(results)
    }

    async fn run_failure_envelope(&self, plies: &[Ply], job_id: Uuid) -> anyhow::Result<CltResults> {
        // Sweep over Nx and Ny to generate failure envelope
        let n_points = 100;
        let n_max = 1e6; // 1 MN/m

        // TODO: Implement 2D sweep and first-ply failure detection

        Ok(CltResults::default())
    }

    async fn run_carpet_plot(&self, plies: &[Ply], job_id: Uuid) -> anyhow::Result<CltResults> {
        // Sweep over ply angles to generate carpet plot
        // TODO: Implement parametric sweep

        Ok(CltResults::default())
    }

    async fn mark_failed(&self, job_id: Uuid, error: &str) -> anyhow::Result<()> {
        sqlx::query!(
            "UPDATE laminate_analyses SET status = 'failed', error_message = $2,
             completed_at = NOW() WHERE id = $1",
            job_id, error
        )
        .execute(&self.db)
        .await?;
        Ok(())
    }
}

use crate::clt::abd::{CltResults, FailureIndex};
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–5)

**Day 1: Project initialization and Rust backend scaffold**
```bash
cargo init compforge-api
cd compforge-api
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
- `migrations/001_initial.sql` — All 8 tables: users, organizations, org_members, materials, projects, laminate_analyses, fea_simulations, server_jobs, usage_records
- `src/db/mod.rs` — Database pool initialization
- `src/db/models.rs` — All SQLx structs with FromRow derives
- Run `sqlx migrate run` to apply schema
- Seed script for initial 50 materials (T300/914, IM7/8552, AS4/3501-6, E-glass/epoxy, etc.)

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

**Day 5: Materials API and database seeding**
- `src/api/handlers/materials.rs` — Search materials (parametric + full-text), get material, list by system
- Parametric search: filter by material_system, fiber_type, resin_type, form
- Pagination with cursor-based scrolling for large result sets
- Material database seed: 50 common composite systems with CMH-17 A/B-basis allowables
- Material file generator: create `.mat` JSON files with full property sets, upload to S3

### Phase 2 — CLT Core Solver (Days 6–12)

**Day 6: ABD matrix computation framework**
- `clt-core/` — New Rust workspace member (shared between native and WASM)
- `clt-core/src/abd.rs` — CltSolver struct with ABD matrix assembly
- `clt-core/src/material.rs` — Material and Ply structs
- Implement Q-bar transformation (ply stiffness rotation to global coordinates)
- ABD matrix assembly via integration over ply thickness
- Unit tests: single ply [0], cross-ply [0/90], quasi-isotropic [0/±45/90]s

**Day 7: ABD inversion and laminate properties**
- 6×6 matrix inversion via `nalgebra::Matrix6::try_inverse()`
- Explicit singularity checking with meaningful error messages
- Extract effective laminate properties: Ex, Ey, Gxy, νxy from compliance matrix
- Symmetric laminate detection (B matrix should be zero)
- Tests: balanced vs. unbalanced laminates, symmetric vs. asymmetric

**Day 8: Ply-by-ply stress recovery**
- Solve for mid-plane strains ε⁰ and curvatures κ from loads
- Compute strains at any z-position through thickness
- Transform strains to ply material coordinates
- Compute ply stresses σ₁, σ₂, τ₁₂ at top and bottom of each ply
- Tests: pure tension (Nx only), pure bending (Mx only), combined loading

**Day 9: Failure criteria implementation**
- `clt-core/src/failure.rs` — Failure criterion evaluators
- Max stress criterion: separate tension/compression limits
- Max strain criterion: with strain allowables
- Tsai-Wu interactive quadratic criterion with F₁₂ interaction term
- Hashin criterion: separate fiber/matrix tension/compression modes
- Tests: verify against published failure envelopes from literature

**Day 10: Thermal loads and hygroscopic effects**
- Thermal force/moment resultants from ΔT and ply CTEs
- Effective laminate CTEs: αx, αy from thermal compliance
- Thermal-only stress recovery (ΔT with no mechanical loads)
- Cure residual stress calculation (ΔT = -150°C from cure temperature)
- Tests: [0/90] laminate thermal expansion, cure residual stress distribution

**Day 11: First-ply failure envelope generation**
- `clt-core/src/envelope.rs` — 2D load space sweep (Nx, Ny)
- For each load point: compute ply stresses, evaluate all failure criteria
- Find first-ply failure boundary in Nx-Ny space
- Identify critical ply and failure mode at each boundary point
- Output polygon vertices for envelope rendering
- Tests: compare against classical lamination theory handbook examples

**Day 12: Carpet plots and parametric sweeps**
- `clt-core/src/carpet.rs` — Sweep over design variables (ply angles, thicknesses)
- Compute laminate properties (Ex, Ey, etc.) at each design point
- Generate 2D grid data for contour plotting
- Support for multi-objective plots (Ex vs. Ey, strength vs. stiffness)
- Tests: [0/θ/0] laminate sweep over θ, verify symmetry properties

### Phase 3 — WASM Build + Frontend Visualization (Days 13–19)

**Day 13: WASM CLT compilation**
- `clt-wasm/` — New workspace member for WASM target
- `clt-wasm/Cargo.toml` — wasm-bindgen, serde-wasm-bindgen, nalgebra dependencies
- `clt-wasm/src/lib.rs` — WASM entry points: `compute_abd()`, `compute_failure()`, `generate_envelope()`
- `wasm-pack build --target web --release` pipeline
- `wasm-opt -Oz` for size optimization (target <1.5MB gzipped)
- JavaScript wrapper: `CltSolver` class that loads WASM and provides async API

**Day 14: Frontend scaffold and laminate designer foundation**
```bash
npm create vite@latest frontend -- --template react-ts
cd frontend
npm i zustand @tanstack/react-query axios three @react-three/fiber @react-three/drei
```
- `src/App.tsx` — Router, auth context, layout
- `src/stores/projectStore.ts` — Zustand store for project state
- `src/stores/layupStore.ts` — Laminate stacking sequence state
- `src/components/LaminateDesigner/StackingTable.tsx` — Table editor for ply sequence
- Drag-and-drop ply reordering, add/remove/duplicate plies

**Day 15: Stacking sequence editor**
- `src/components/LaminateDesigner/PlyRow.tsx` — Single ply editor (material select, angle input, thickness)
- Material selector with search and filtering by system/fiber
- Angle input with common presets (0, ±45, 90) and custom values
- Thickness override or use material default
- Ply copy/paste, bulk angle operations (mirror, rotate)
- Symmetry enforcement: auto-generate symmetric half

**Day 16: Three.js layup visualizer — basic rendering**
- `src/components/LayupVisualizer.tsx` — Three.js canvas with extruded plies
- Each ply rendered as thin box geometry with color-coded angle
- Fiber direction arrows overlaid on each ply surface
- Camera controls: orbit, pan, zoom with smooth animations
- Lighting: ambient + two directional lights for depth perception

**Day 17: Three.js layup visualizer — interaction**
- Ply selection on click with highlight effect (emissive glow)
- Selected ply info panel: index, material, angle, thickness, z-position
- Ply visibility toggles: show/hide individual plies, toggle arrows
- Cross-section view: slice through thickness at arbitrary z-plane
- Exploded view animation: separate plies along z-axis for inspection

**Day 18: CLT results visualization — properties display**
- `src/components/ResultsPanel/PropertiesDisplay.tsx` — Effective laminate properties table
- Display: Ex, Ey, Gxy, νxy, αx, αy with engineering units
- ABD matrix display in collapsible matrix view
- Laminate classification: symmetric, balanced, quasi-isotropic detection
- Export properties to CSV

**Day 19: CLT results visualization — stress plots**
- `src/components/ResultsPanel/StressPlot.tsx` — Through-thickness stress distribution
- Plot σ₁, σ₂, τ₁₂ vs. z-position for each ply
- Interactive hover to see stress values at specific z
- Failure envelope overlays: show failure indices on same axes
- Color-coded plies matching layup visualizer colors

### Phase 4 — API + Job Orchestration (Days 20–25)

**Day 20: Laminate analysis API endpoints**
- `src/api/handlers/laminate.rs` — Create analysis, get analysis, list analyses, cancel analysis
- Ply count extraction and WASM/server routing logic
- Plan-based limits enforcement (free: 10 plies, pro: 100 plies, advanced: unlimited)
- Validation: check material IDs exist, angles are valid, thicknesses > 0

**Day 21: Server-side CLT worker**
- `src/workers/laminate_worker.rs` — Redis job consumer, runs native CLT solver
- Material loading from PostgreSQL with caching in worker memory
- Progress streaming: worker publishes progress → Redis pub/sub → WebSocket to client
- Envelope generation: 2D sweep with progress updates every 10%
- S3 result upload with presigned URL generation for client download

**Day 22: WebSocket for live analysis progress**
- `src/api/ws/mod.rs` — Axum WebSocket upgrade handler
- `src/api/ws/analysis_progress.rs` — Subscribe to analysis progress channel
- Client receives: `{ progress_pct, current_step, iterations_complete, estimated_remaining_sec }`
- Connection management: heartbeat ping/pong, automatic cleanup on disconnect
- Frontend: `src/hooks/useAnalysisProgress.ts` — React hook for WebSocket subscription

**Day 23: Failure envelope and carpet plot generation**
- Envelope API: specify load ranges (Nx_min, Nx_max, Ny_min, Ny_max), failure criterion
- Carpet plot API: specify parameter sweeps (angle_1_range, angle_2_range), property to plot
- Server-side parallel computation: use Rayon for multi-threaded sweep
- Result format: JSON with polygon vertices for envelope, grid data for carpet plot
- Frontend rendering: WebGL-based contour plots with isolines and colormap

**Day 24: Report generation service**
- `report-service/` — Python FastAPI service for PDF generation
- `report-service/generator.py` — Generate CMH-17 style analysis report
- Report sections: laminate definition, material properties, ABD matrices, stress distributions, failure analysis, margin of safety summary
- Template: Jinja2 HTML → WeasyPrint PDF rendering
- Charts: matplotlib for stress plots, PIL for layup diagrams
- S3 upload and presigned URL return

**Day 25: Project management features**
- `src/api/handlers/projects.rs` — Fork project (deep copy with new owner), share via public link
- Project templates: standard layups ([0/90]s, quasi-isotropic, etc.)
- Thumbnail generation: Three.js server-side rendering with headless-gl
- Auto-save: frontend debounced PATCH every 5 seconds on laminate changes
- Version history: store snapshots of laminate_data on significant changes

### Phase 5 — WebGL Plotting + Results UI (Days 26–30)

**Day 26: WebGL carpet plot renderer**
- `src/components/CarpetPlot/WebGLRenderer.tsx` — Custom WebGL2 shader for dense 2D heatmaps
- Vertex shader: map grid coordinates to clip space
- Fragment shader: texture lookup for data values, colormap interpolation (viridis, jet, plasma)
- GPU texture upload: 2D texture from carpet plot grid data
- Isoline overlay: edge detection shader for drawing contours at specific values

**Day 27: Carpet plot interaction**
- Mouse hover: display exact (x, y, value) at cursor position
- Click to mark point: add annotation with design point coordinates
- Zoom and pan: 2D camera controls for detailed inspection
- Colormap selection: dropdown to switch between colormaps
- Export: download as PNG or SVG with axes and labels

**Day 28: Failure envelope visualization**
- `src/components/FailureEnvelope/EnvelopeRenderer.tsx` — SVG-based 2D plot
- Axes: Nx (horizontal), Ny (vertical) with engineering units (N/m or kN/m)
- Envelope boundary: filled polygon with semi-transparent color
- Critical ply labels: annotate which ply fails first at each boundary segment
- Load point marker: show current applied load (Nx, Ny) relative to envelope
- Margin of safety calculation: distance from load point to boundary

**Day 29: Results export and sharing**
- Export laminate definition: JSON, CSV with ply table
- Export ABD matrices: CSV, LaTeX table format
- Export stress plots: PNG, SVG with high resolution for publications
- Export failure envelope: PNG, SVG, CSV data points
- Share analysis link: generate read-only public URL for results viewer
- Embed results: iframe code for embedding visualizations in external sites

**Day 30: Optimization integration setup**
- `optimization-service/` — Python FastAPI service with SciPy, pygad (genetic algorithm)
- `optimization-service/optimizer.py` — Stacking sequence optimizer
- API endpoints: create optimization run, get status, get best designs
- Objective functions: minimize weight, maximize stiffness, satisfy strength constraints
- Design variables: ply angles (discrete), ply counts (integer)
- Constraints: symmetry, balance, manufacturing rules (ply-drop restrictions)

### Phase 6 — Billing + Plan Enforcement (Days 31–34)

**Day 31: Stripe integration**
- `src/api/handlers/billing.rs` — Create checkout session, create customer portal session, get subscription status
- `src/api/handlers/webhooks/stripe.rs` — Handle subscription events: `checkout.session.completed`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.payment_failed`
- Plan mapping:
  - Free: 10 plies, CLT only, 3 projects
  - Pro ($149/mo): 100 plies, failure envelopes, carpet plots, unlimited projects, PDF reports
  - Advanced ($349/mo): Unlimited plies, FEA (up to 100K elements), optimization, API access

**Day 32: Usage tracking and plan limits**
- `src/middleware/plan_limits.rs` — Middleware checking plan limits before analysis execution
- `src/services/usage.rs` — Track FEA element-hours per billing period
- Usage record insertion after each server-side analysis completes
- Usage dashboard: `src/api/handlers/usage.rs` — Current period usage, historical usage
- Approaching-limit warnings at 80% and 100% of FEA elements

**Day 33: Billing UI**
- `frontend/src/pages/Billing.tsx` — Plan comparison table, current plan display, usage meter
- `frontend/src/components/billing/PlanCard.tsx` — Plan feature comparison cards
- `frontend/src/components/billing/UsageMeter.tsx` — FEA elements usage bar with threshold indicators
- Upgrade/downgrade flow via Stripe Customer Portal
- Upgrade prompt modals when hitting plan limits (e.g., "Laminate has 120 plies. Upgrade to Advanced for unlimited.")

**Day 34: Feature gating**
- Gate failure envelope generation behind Pro plan
- Gate carpet plots behind Pro plan
- Gate PDF report generation behind Pro plan
- Gate FEA simulation behind Advanced plan
- Gate optimization behind Advanced plan
- Show locked feature indicators with upgrade CTAs in UI
- Admin endpoint: override plan for specific users (beta testers, academic partners)

### Phase 7 — CLT Validation + Testing (Days 35–38)

**Day 35: CLT validation — basic laminates**
- Benchmark 1: Single [0°] ply — verify Ex = E₁, Ey = E₂, Gxy = G₁₂, νxy = ν₁₂
- Benchmark 2: Cross-ply [0/90] — verify Ex = (E₁+E₂)/2, symmetric properties
- Benchmark 3: Quasi-isotropic [0/±45/90]s — verify Ex ≈ Ey, νxy ≈ 0.3
- Benchmark 4: Angle-ply [±45]s — verify high shear stiffness Gxy
- Automated test suite: `clt-core/tests/benchmarks.rs` with assertions within 0.1% tolerance

**Day 36: CLT validation — failure predictions**
- Benchmark 5: T300/914 [0]₈ under Nx = 500 kN/m — verify Tsai-Wu FI = **0.85** (safe)
- Benchmark 6: IM7/8552 [±45]₂s under Nxy = 200 kN/m — verify first-ply failure at Nxy = **235 kN/m**
- Benchmark 7: AS4/3501-6 [0/90]₂s under Nx + Ny — verify biaxial failure envelope area = **12.5 MN²/m²**
- Benchmark 8: E-glass/epoxy [0₂/90₂]s under bending Mx = 50 N — verify max ply stress σ₁ = **120 MPa**
- Compare results against CMH-17 Volume 3 example problems

**Day 37: CLT validation — thermal and hygroscopic**
- Benchmark 9: [0/90]s T300/914 with ΔT = -150°C (cure) — verify transverse cracking stress σ₂ = **-45 MPa**
- Benchmark 10: [±45]s with moisture absorption 1% — verify hygroscopic expansion vs. analytical
- Test thermal CTE extraction: verify αx, αy match handbook values for standard layups
- Stress-free temperature effects on ABD matrices

**Day 38: Integration and performance testing**
- End-to-end test: create project → design [0/±45/90]s laminate → run CLT → view results → generate report
- API integration tests: auth → project CRUD → analysis → results
- WASM solver test: load in headless browser (Playwright), run 50-ply analysis, verify results match native solver
- WebSocket test: connect → subscribe → receive progress → receive completion
- Performance benchmarks:
  - WASM CLT: 100-ply laminate < 50ms (Chrome)
  - Server CLT: 100-ply laminate < 10ms (native)
  - Failure envelope: 100×100 grid points < 5 seconds
  - Carpet plot: 50×50 grid < 2 seconds

### Phase 8 — Deployment + Polish + Launch (Days 39–42)

**Day 39: Docker and Kubernetes configuration**
- `Dockerfile` — Multi-stage Rust build (cargo-chef for layer caching)
- `docker-compose.yml` — Full local dev stack (API, PostgreSQL, Redis, MinIO, frontend)
- `k8s/` — Kubernetes manifests:
  - `api-deployment.yaml` — API server (3 replicas, HPA)
  - `worker-deployment.yaml` — CLT workers (auto-scaling based on queue depth)
  - `postgres-statefulset.yaml` — PostgreSQL with PVC
  - `redis-deployment.yaml` — Redis for job queue and pub/sub
  - `optimization-service-deployment.yaml` — Python optimization service
  - `report-service-deployment.yaml` — Python report generator
  - `ingress.yaml` — NGINX ingress with TLS
- Health check endpoints: `/health/live`, `/health/ready`

**Day 40: CDN and WASM delivery**
- CloudFront distribution for:
  - Frontend static assets (HTML, JS, CSS) — cached at edge
  - WASM CLT bundle (~1.5MB) — cached at edge with long TTL and versioned URLs
  - Material datasheets from S3 — cached with 1-hour TTL
- WASM preloading: start downloading WASM bundle on page load before user needs it
- Service worker for offline WASM caching (progressive enhancement)
- Material database CDN: edge caching for material search API responses

**Day 41: Monitoring, logging, and polish**
- Prometheus metrics: analysis duration histogram, ABD solver convergence rate, API latency percentiles
- Grafana dashboards: system health, analysis throughput, user activity
- Sentry integration: frontend and backend error tracking with source maps
- Structured logging (tracing + JSON) for all API requests and solver events
- UI polish: loading states, error messages, empty states, responsive layout
- Accessibility: keyboard navigation in laminate designer, ARIA labels on interactive elements

**Day 42: Launch preparation and final testing**
- End-to-end smoke test in production environment
- Database backup and restore verification
- Rate limiting: 100 req/min per user, 10 analyses/min per user
- Security audit: JWT validation, SQL injection (SQLx prevents this), CORS configuration, CSP headers
- Landing page with product overview, pricing, and signup
- Documentation: getting started guide, CLT theory primer, material database guide
- Deploy to production, enable monitoring alerts, announce launch

---

## Critical Files

```
compforge/
├── clt-core/                              # Shared CLT library (native + WASM)
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── abd.rs                         # ABD matrix computation and inversion
│   │   ├── material.rs                    # Material and Ply structs
│   │   ├── failure.rs                     # Failure criteria (Tsai-Wu, max stress, Hashin)
│   │   ├── envelope.rs                    # Failure envelope generation
│   │   ├── carpet.rs                      # Carpet plot parametric sweeps
│   │   └── thermal.rs                     # Thermal/hygroscopic effects
│   └── tests/
│       ├── benchmarks.rs                  # CLT validation benchmarks
│       └── failure_tests.rs               # Failure criteria validation

├── clt-wasm/                              # WASM compilation target
│   ├── Cargo.toml
│   └── src/
│       └── lib.rs                         # WASM entry points (wasm-bindgen)

├── compforge-api/                         # Rust API server
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
│   │   │   │   ├── laminate.rs            # Create/get/cancel analysis
│   │   │   │   ├── materials.rs           # Material search/get
│   │   │   │   ├── billing.rs             # Stripe checkout/portal
│   │   │   │   ├── usage.rs               # Usage tracking
│   │   │   │   └── webhooks/
│   │   │   │       └── stripe.rs          # Stripe webhook handler
│   │   │   └── ws/
│   │   │       ├── mod.rs                 # WebSocket upgrade handler
│   │   │       └── analysis_progress.rs   # Live progress streaming
│   │   ├── middleware/
│   │   │   └── plan_limits.rs             # Plan enforcement middleware
│   │   ├── services/
│   │   │   ├── usage.rs                   # Usage tracking service
│   │   │   └── s3.rs                      # S3 client helpers
│   │   └── workers/
│   │       ├── mod.rs                     # Worker pool management
│   │       └── laminate_worker.rs         # Server-side CLT execution
│   ├── migrations/
│   │   └── 001_initial.sql                # Full database schema
│   └── tests/
│       ├── api_integration.rs             # API integration tests
│       └── analysis_e2e.rs                # End-to-end analysis tests

├── frontend/                              # React frontend
│   ├── package.json
│   ├── vite.config.ts
│   ├── src/
│   │   ├── App.tsx                        # Router, providers, layout
│   │   ├── main.tsx                       # Entry point
│   │   ├── stores/
│   │   │   ├── authStore.ts               # Auth state (JWT, user)
│   │   │   ├── projectStore.ts            # Project state
│   │   │   ├── layupStore.ts              # Laminate stacking sequence state
│   │   │   └── resultsStore.ts            # Analysis results state
│   │   ├── hooks/
│   │   │   ├── useAnalysisProgress.ts     # WebSocket hook for live progress
│   │   │   └── useWasmClt.ts              # WASM CLT solver hook (load + run)
│   │   ├── lib/
│   │   │   ├── api.ts                     # Axios API client
│   │   │   └── wasmLoader.ts              # WASM bundle loading
│   │   ├── pages/
│   │   │   ├── Dashboard.tsx              # Project list
│   │   │   ├── LaminateEditor.tsx         # Main laminate designer
│   │   │   ├── Materials.tsx              # Material database browser
│   │   │   ├── Billing.tsx                # Plan management
│   │   │   ├── Login.tsx                  # Auth pages
│   │   │   └── Register.tsx
│   │   ├── components/
│   │   │   ├── LaminateDesigner/
│   │   │   │   ├── StackingTable.tsx      # Ply sequence table editor
│   │   │   │   ├── PlyRow.tsx             # Single ply editor
│   │   │   │   ├── MaterialSelector.tsx   # Material picker with search
│   │   │   │   └── SymmetryTools.tsx      # Symmetry enforcement controls
│   │   │   ├── LayupVisualizer.tsx        # Three.js 3D layup viewer
│   │   │   ├── ResultsPanel/
│   │   │   │   ├── PropertiesDisplay.tsx  # Laminate properties table
│   │   │   │   ├── StressPlot.tsx         # Through-thickness stress plot
│   │   │   │   └── FailureIndices.tsx     # Failure index table
│   │   │   ├── CarpetPlot/
│   │   │   │   └── WebGLRenderer.tsx      # WebGL 2D heatmap renderer
│   │   │   ├── FailureEnvelope/
│   │   │   │   └── EnvelopeRenderer.tsx   # SVG envelope plotter
│   │   │   ├── billing/
│   │   │   │   ├── PlanCard.tsx
│   │   │   │   └── UsageMeter.tsx
│   │   │   └── common/
│   │   │       ├── AnalysisToolbar.tsx    # Run/cancel/export controls
│   │   │       └── MaterialSearch.tsx     # Material search component
│   └── public/
│       └── wasm/                          # WASM CLT bundle (loaded at runtime)

├── report-service/                        # Python FastAPI (report generation)
│   ├── requirements.txt
│   ├── main.py                            # FastAPI app
│   ├── generator.py                       # PDF report generator (WeasyPrint)
│   ├── templates/
│   │   └── cmh17_report.html              # Jinja2 report template
│   └── Dockerfile

├── optimization-service/                  # Python FastAPI (stacking sequence optimization)
│   ├── requirements.txt
│   ├── main.py                            # FastAPI app
│   ├── optimizer.py                       # Genetic algorithm optimizer
│   └── Dockerfile

├── k8s/                                   # Kubernetes manifests
│   ├── api-deployment.yaml
│   ├── worker-deployment.yaml
│   ├── postgres-statefulset.yaml
│   ├── redis-deployment.yaml
│   ├── report-service-deployment.yaml
│   ├── optimization-service-deployment.yaml
│   └── ingress.yaml

├── docker-compose.yml                     # Local development stack
├── Cargo.toml                             # Workspace root
└── .github/
    └── workflows/
        ├── ci.yml                         # Test + lint on PR
        ├── wasm-build.yml                 # Build + deploy WASM bundle
        └── deploy.yml                     # Build Docker images + deploy to K8s
```

---

## CLT Solver Validation Suite

### Benchmark 1: Single Unidirectional Ply (Ex, Ey extraction)

**Laminate:** Single T300/914 ply [0°], thickness = 0.125 mm

**Material:** E₁ = 138 GPa, E₂ = 11 GPa, G₁₂ = 5.5 GPa, ν₁₂ = 0.28

**Expected:** Ex = E₁ = **138.0 GPa**, Ey = E₂ = **11.0 GPa**, Gxy = G₁₂ = **5.5 GPa**, νxy = ν₁₂ = **0.28**

**Tolerance:** < 0.01% (should match exactly after accounting for ν₂₁ correction)

### Benchmark 2: Cross-Ply Laminate (Symmetry, balanced properties)

**Laminate:** [0/90]s AS4/3501-6, 4 plies total, each 0.125 mm

**Material:** E₁ = 142 GPa, E₂ = 10.3 GPa, G₁₂ = 7.2 GPa, ν₁₂ = 0.27

**Expected:** Ex ≈ Ey ≈ **76.2 GPa** (average of E₁ and E₂), B matrix = **0** (symmetric), νxy ≈ **0.30**

**Tolerance:** Ex, Ey < 1%, B matrix elements < 1e-6

### Benchmark 3: Quasi-Isotropic Laminate (In-plane isotropy)

**Laminate:** [0/±45/90]s IM7/8552, 8 plies total, each 0.131 mm

**Material:** E₁ = 171 GPa, E₂ = 9.08 GPa, G₁₂ = 5.29 GPa, ν₁₂ = 0.32

**Expected:** Ex ≈ Ey ≈ **56.3 GPa**, Gxy ≈ **21.7 GPa**, νxy ≈ **0.30**, A₁₆ = A₂₆ ≈ **0** (balanced)

**Tolerance:** |Ex - Ey| / Ex < 0.1%, |νxy - 0.3| < 3%, A₁₆, A₂₆ < 1e-6

### Benchmark 4: Tsai-Wu Failure Index (Uniaxial tension)

**Laminate:** [0]₈ T300/914, total thickness = 1.0 mm, under Nx = 500 kN/m

**Material strengths:** F₁ₜ = 1500 MPa, F₁c = 1200 MPa, F₂ₜ = 50 MPa, F₂c = 200 MPa, F₁₂ = 70 MPa

**Expected:** Ply stress σ₁ = Nx / h = **500 MPa**, Tsai-Wu FI = **0.33** (safe, margin = 2.0)

**Tolerance:** FI < 5%

### Benchmark 5: First-Ply Failure Envelope Area (Biaxial)

**Laminate:** [0/90]₂s AS4/3501-6, total thickness = 1.0 mm

**Strengths (B-basis):** F₁ₜ = 1950 MPa, F₁c = 1100 MPa, F₂ₜ = 48 MPa, F₂c = 200 MPa, F₁₂ = 79 MPa

**Expected envelope area (Nx-Ny space):** Approximately **12.5 MN²/m²** (from literature)

**Expected first-ply failure under pure Nx:** Nx_fail ≈ **1100 kN/m** (fiber compression in 0° plies)

**Expected first-ply failure under pure Ny:** Ny_fail ≈ **200 kN/m** (matrix compression in 90° plies)

**Tolerance:** Envelope area < 10%, Nx_fail < 5%, Ny_fail < 5%

---

## Verification

### Integration Testing Checklist

1. **Auth flow** — Register → login → JWT issued → authorized API calls → token refresh → logout
2. **Project CRUD** — Create → update laminate → save → reload → verify stacking sequence preserved
3. **Laminate design** — Add ply → set material/angle → reorder → duplicate → delete → verify state
4. **WASM CLT** — 20-ply laminate → WASM solver → ABD matrices returned → properties computed
5. **Server CLT** — 120-ply laminate → job queued → worker picks up → WebSocket progress → results in S3
6. **Three.js visualizer** — 50-ply laminate → load visualizer → rotate/zoom → select ply → verify info panel
7. **Material library** — Search "T300" → results returned → select material → use in laminate
8. **Failure analysis** — Apply loads → compute failure indices → verify critical ply identified
9. **Failure envelope** — Generate envelope → 100×100 grid → boundary polygon rendered → export PNG
10. **Plan limits** — Free user → 15-ply laminate → blocked with upgrade prompt
11. **Billing** — Subscribe to Pro → Stripe checkout → webhook → plan updated → limits lifted
12. **Report generation** — Run analysis → request PDF → report service generates → download link returned
13. **Cross-platform WASM** — Test CLT solver in Chrome, Firefox, Safari → verify consistent results
14. **Error handling** — Singular ABD matrix (all 0°) → meaningful error message → suggest fix
15. **Performance** — 100-ply carpet plot (50×50 grid) → complete in <2s → verify no memory leaks

### SQL Verification Queries

```sql
-- 1. Analysis throughput and success rate
SELECT
    execution_mode,
    status,
    COUNT(*) as count,
    AVG(wall_time_ms)::int as avg_time_ms,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY wall_time_ms)::int as p95_time_ms
FROM laminate_analyses
WHERE created_at >= NOW() - INTERVAL '7 days'
GROUP BY execution_mode, status;

-- 2. Analysis type distribution
SELECT analysis_type, COUNT(*) as count,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 1) as pct
FROM laminate_analyses
WHERE created_at >= NOW() - INTERVAL '30 days'
GROUP BY analysis_type ORDER BY count DESC;

-- 3. User plan distribution and conversion
SELECT plan, COUNT(*) as users,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 1) as pct
FROM users GROUP BY plan ORDER BY users DESC;

-- 4. Material popularity
SELECT m.name, m.manufacturer, m.material_system,
    COUNT(DISTINCT la.project_id) as projects_using,
    AVG(la.ply_count) as avg_ply_count
FROM materials m
JOIN laminate_analyses la ON la.stacking_sequence::text ILIKE '%' || m.id::text || '%'
WHERE la.created_at >= NOW() - INTERVAL '30 days'
GROUP BY m.id
ORDER BY projects_using DESC
LIMIT 20;

-- 5. Average laminate complexity by plan
SELECT u.plan,
    AVG(la.ply_count) as avg_plies,
    MAX(la.ply_count) as max_plies,
    COUNT(DISTINCT la.project_id) as projects
FROM laminate_analyses la
JOIN users u ON la.user_id = u.id
WHERE la.created_at >= NOW() - INTERVAL '30 days'
GROUP BY u.plan;
```

### Performance Benchmarks

| Operation | Target | Method |
|-----------|--------|--------|
| WASM CLT: 10-ply ABD | <10ms | Browser benchmark (Chrome DevTools) |
| WASM CLT: 100-ply ABD | <50ms | Browser benchmark |
| Server CLT: 100-ply ABD | <10ms | Server timing, native Rust |
| Failure envelope: 100×100 grid | <5s | Server timing, parallel sweep |
| Carpet plot: 50×50 grid | <2s | Server timing, parallel sweep |
| Three.js: render 100 plies | 60 FPS | Chrome FPS counter during rotation |
| WebGL carpet plot: 100×100 | 60 FPS | Chrome FPS counter during pan/zoom |
| API: create analysis | <150ms | p95 latency (Prometheus) |
| API: search materials | <80ms | p95 latency with 1000 materials in DB |
| WASM bundle load (cached) | <300ms | Service worker cache hit |
| WASM bundle load (cold) | <2s | CDN delivery, 1.5MB gzipped |
| WebSocket: progress latency | <80ms | Time from worker emit to client receive |
| PDF report generation | <5s | Python service timing (8-page report) |

---

## Deployment Architecture

### Infrastructure

```
┌─────────────────────────────────────────────────┐
│                  CloudFront CDN                   │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────┐ │
│  │ Frontend SPA │  │ WASM Bundle │  │ Material │ │
│  │ (HTML/JS/CSS)│  │ (versioned) │  │ Sheets   │ │
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
│ Multi-AZ     │ │ Cluster mode │ │  Materials)  │
└──────────────┘ └──────┬───────┘ └──────────────┘
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ CLT Worker   │ │ CLT Worker   │ │ CLT Worker   │
│ (Rust native)│ │ (Rust native)│ │ (Rust native)│
│ c7g.xlarge   │ │ c7g.xlarge   │ │ c7g.xlarge   │
│ HPA: 2-10    │ │              │ │              │
└──────────────┘ └──────────────┘ └──────────────┘
        │               │               │
        └───────────────┼───────────────┘
                        ▼
                ┌───────────────┐
                │ Report Svc    │
                │ (Python/Fast) │
                │ Pod ×2        │
                └───────────────┘
```

### Scaling Strategy

- **API servers**: Horizontal pod autoscaler (HPA) at 70% CPU, min 3, max 10
- **CLT workers**: HPA based on Redis queue depth — scale up when queue > 5 jobs, scale down when idle 5 min
- **Worker instances**: AWS c7g.xlarge (4 vCPU, 8GB RAM) for compute-optimized CLT calculations
- **Database**: RDS PostgreSQL r6g.large with read replicas for material search queries
- **Redis**: ElastiCache r6g.large cluster with 1 primary + 2 replicas

### S3 Lifecycle Policy

```json
{
  "Rules": [
    {
      "ID": "analysis-results-lifecycle",
      "Prefix": "analyses/",
      "Status": "Enabled",
      "Transitions": [
        { "Days": 30, "StorageClass": "STANDARD_IA" },
        { "Days": 90, "StorageClass": "GLACIER" }
      ],
      "Expiration": { "Days": 365 }
    },
    {
      "ID": "material-datasheets-keep",
      "Prefix": "materials/",
      "Status": "Enabled",
      "Transitions": [
        { "Days": 180, "StorageClass": "STANDARD_IA" }
      ]
    },
    {
      "ID": "wasm-bundles-keep",
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
| Simple FEA Solver | 2D layered shell FEA (10K elements) with ply-by-ply stress recovery, linear static analysis only, drag-and-drop mesh import (Gmsh .msh files), basic load/boundary condition UI | High |
| Progressive Failure Analysis | Ply-by-ply stiffness degradation after first-ply failure, iterative load stepping until last-ply failure, identify failure sequence and ultimate strength | High |
| Manufacturing Simulation | Basic ply draping on flat-to-curved molds using kinematic draping algorithm, flat pattern generation for ply cutting with nesting optimization | High |
| Stacking Sequence Optimization | Genetic algorithm for optimal ply angles and counts given design constraints (stiffness, strength, weight), manufacturing constraints (symmetry, ply-drop rules) | Medium |
| Sandwich Panel Analysis | Face sheet + core modeling with specific failure modes (wrinkling, crimping, core shear failure), honeycomb and foam core database | Medium |
| Bolted Joint Analysis | Bearing/bypass load interaction, open-hole tension/compression strength prediction, fastener pattern optimization | Medium |
| Advanced Failure Criteria | Puck, LaRC03/04, Cuntze criteria for improved matrix cracking and delamination prediction | Low |
| Damage Tolerance | Impact damage modeling (BVID), compression after impact (CAI) strength prediction, delamination growth under fatigue loading | Low |
