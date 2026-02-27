# 92. TunnelSim — Tunnel Engineering Platform

## Implementation Plan

**MVP Scope:** Browser-based 3D tunnel modeling interface with drag-and-drop alignment definition (horizontal/vertical curves, cross-section templates: circular, horseshoe, D-shape) and automatic mesh generation via Rust advancing-front tetrahedral algorithm compiled to native for server-side execution, 3D elasto-plastic finite element solver implementing Mohr-Coulomb and Hoek-Brown constitutive models with full Newton-Raphson nonlinear iteration and sparse direct solver (MUMPS/PARDISO) for ground-structure interaction, sequential excavation simulation for NATM (New Austrian Tunnelling Method) with top-heading/bench excavation stages including stress relief modeling and support installation (shotcrete elastic beam elements, rock bolts as embedded truss elements with activation timing), convergence-confinement method automated calculator generating ground reaction curves (GRC) and support characteristic curves (SCC) from analytical closed-form solutions, rock mass classification tools for RMR (Rock Mass Rating), Q-system, and GSI (Geological Strength Index) with automatic parameter mapping to Hoek-Brown constants, Gaussian settlement trough calculator for greenfield volume loss prediction with transverse and longitudinal profiles rendered via Three.js WebGL visualization, PostgreSQL storage for project metadata and ground parameters with S3 for 3D mesh data and FEM results (binary HDF5 format), Python FastAPI service for geostatistical kriging interpolation of borehole data and rock mass characterization algorithms, Stripe billing with three tiers (Free / Pro $179/mo / Advanced $399/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| FEM Solver | Rust (native) | Custom 3D elasto-plastic solver with Mohr-Coulomb, Hoek-Brown, MUMPS sparse solver |
| Mesh Generator | Rust (native) | Advancing-front tetrahedral mesh generation, hexahedral structured for tunnel-specific zones |
| Geotechnical Service | Python 3.12 (FastAPI) | Geostatistical kriging (PyKrige), rock classification, stereonet analysis (mplstereonet) |
| Database | PostgreSQL 16 | Projects, users, tunnel models, borehole data, ground parameters |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | 3D meshes (HDF5), FEM results, monitoring data, geological cross-sections |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| 3D Visualization | Three.js (r164) + React Three Fiber | Tunnel geometry, mesh visualization, settlement contours, excavation stages |
| 2D Charts | D3.js v7 | Convergence plots, settlement profiles, GRC/SCC diagrams, monitoring graphs |
| Computation Queue | Redis 7 + Tokio tasks | FEM job queue, mesh generation queue, progress tracking |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | Frontend assets, 3D model viewer static resources |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, solver performance, error tracking |
| CI/CD | GitHub Actions | Rust tests, FEM validation suite, Docker image push |

### Key Architecture Decisions

1. **Server-side FEM solver only (no WASM) for geotechnical reliability**: Unlike circuit simulation where small problems can run client-side, tunnel FEM requires 3D meshes with 50K-500K elements even for simple cases, and elasto-plastic constitutive models with Newton-Raphson iteration are too compute-intensive for WASM. All FEM runs execute server-side on c7g.8xlarge instances (32 vCPU, 64GB RAM) with MUMPS parallel sparse solver, providing guaranteed numerical accuracy and convergence control essential for safety-critical geotechnical design.

2. **Rust for mesh generation and FEM solver rather than wrapping PLAXIS/FLAC**: Building a custom Rust FEM engine gives full control over constitutive model implementation, convergence algorithms, and cloud parallelization without the licensing complexity of wrapping commercial solvers. The advancing-front mesher generates tunnel-specific structured hexahedral meshes around the excavation boundary with tetrahedral infill in the far field, optimizing element aspect ratios for accuracy and solver performance. Rust's memory safety prevents the segfaults common in C++ FEM codes.

3. **Three.js with custom shaders for 3D tunnel visualization**: Three.js provides GPU-accelerated rendering of large 3D meshes (500K+ elements) with real-time camera controls and cross-section clipping planes. Custom vertex shaders color-code elements by stress state (tensile/compressive), displacement magnitude, or factor of safety. The tunnel alignment is rendered as a swept tube geometry along a 3D curve, with cross-sections instanced at user-defined chainage stations. This is decoupled from the SVG 2D schematic used for alignment plan view.

4. **Python FastAPI service for geotechnical algorithms separate from Rust backend**: Rock mass classification (RMR, Q-system, GSI) involves empirical lookup tables and correlations that are better implemented in Python using NumPy and Pandas. Geostatistical interpolation (kriging) between boreholes uses PyKrige for variogram modeling. The Python service exposes a REST API consumed by the Rust backend, keeping numerical geotechnics separate from web API and database logic.

5. **PostgreSQL for borehole/ground data with PostGIS for alignment geospatial queries**: Borehole logs are stored as JSONB (layered stratigraphy with depths and properties per layer), allowing flexible schema for different project data formats. PostGIS enables spatial queries for "find boreholes within 100m of tunnel alignment" and geometric intersection of 3D tunnel volumes with geological boundaries. S3 stores large binary data (meshes, results) while PostgreSQL holds searchable metadata.

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "postgis";  -- For spatial alignment/borehole queries
CREATE EXTENSION IF NOT EXISTS "pg_trgm";  -- For fuzzy text search on projects

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

-- Organizations (for team/enterprise plans)
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

-- Projects (tunnel design workspace)
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    location TEXT,  -- Project location/site name
    alignment_data JSONB NOT NULL DEFAULT '{}',  -- Horizontal/vertical curve geometry, chainage
    cross_section_data JSONB NOT NULL DEFAULT '{}',  -- Section templates, dimensions
    ground_model_data JSONB DEFAULT '{}',  -- Stratigraphy, layer boundaries
    settings JSONB DEFAULT '{}',  -- Units, coordinate system, analysis options
    is_public BOOLEAN NOT NULL DEFAULT false,
    thumbnail_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);
CREATE INDEX projects_org_idx ON projects(org_id);
CREATE INDEX projects_updated_idx ON projects(updated_at DESC);

-- Boreholes
CREATE TABLE boreholes (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name TEXT NOT NULL,  -- e.g., "BH-01", "DH-23"
    location GEOMETRY(POINTZ, 4326) NOT NULL,  -- 3D position (long, lat, elevation)
    ground_level_elevation REAL NOT NULL,  -- m above datum
    total_depth REAL NOT NULL,  -- m below ground level
    date_drilled DATE,
    drilling_method TEXT,  -- rotary | percussion | sonic
    stratigraphy JSONB NOT NULL,  -- [{depth_from, depth_to, description, properties: {unit_weight, cohesion, phi, ...}}]
    water_levels JSONB DEFAULT '[]',  -- [{date, depth_to_water}]
    lab_tests JSONB DEFAULT '[]',  -- [{depth, test_type, results}]
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX boreholes_project_idx ON boreholes(project_id);
CREATE INDEX boreholes_location_gist ON boreholes USING GIST(location);

-- Ground Material Library (material properties)
CREATE TABLE ground_materials (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID REFERENCES projects(id) ON DELETE CASCADE,  -- NULL for global library
    name TEXT NOT NULL,
    material_type TEXT NOT NULL,  -- soil | rock | rock_mass
    constitutive_model TEXT NOT NULL,  -- mohr_coulomb | hoek_brown | hardening_soil
    parameters JSONB NOT NULL,  -- Model-specific: {c, phi, psi, E, nu, ...} or {GSI, mi, D, sigma_ci, ...}
    unit_weight REAL,  -- kN/m³
    permeability REAL,  -- m/s
    description TEXT,
    is_builtin BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX materials_project_idx ON ground_materials(project_id);
CREATE INDEX materials_type_idx ON ground_materials(material_type);

-- Meshes (generated tunnel meshes)
CREATE TABLE meshes (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    mesh_type TEXT NOT NULL,  -- 2d_plane_strain | 3d_full | 3d_tunnel_specific
    node_count INTEGER NOT NULL,
    element_count INTEGER NOT NULL,
    element_type TEXT NOT NULL,  -- tet4 | tet10 | hex8 | hex20
    mesh_url TEXT NOT NULL,  -- S3 URL to HDF5 mesh file
    metadata JSONB DEFAULT '{}',  -- {min_element_size, max_element_size, quality_metrics}
    generation_time_sec REAL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX meshes_project_idx ON meshes(project_id);

-- FEM Analyses
CREATE TABLE analyses (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    mesh_id UUID NOT NULL REFERENCES meshes(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    analysis_type TEXT NOT NULL,  -- static | sequential_excavation | transient_settlement
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | queued | running | completed | failed | cancelled
    excavation_stages JSONB DEFAULT '[]',  -- [{stage_num, description, elements_removed, support_installed}]
    boundary_conditions JSONB NOT NULL,  -- {displacements, loads, in_situ_stress}
    solver_settings JSONB DEFAULT '{}',  -- {max_iterations, convergence_tol, solver_type}
    results_url TEXT,  -- S3 URL to HDF5 results file
    results_summary JSONB,  -- {max_displacement, max_stress, convergence_info}
    error_message TEXT,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    wall_time_sec REAL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX analyses_project_idx ON analyses(project_id);
CREATE INDEX analyses_mesh_idx ON analyses(mesh_id);
CREATE INDEX analyses_user_idx ON analyses(user_id);
CREATE INDEX analyses_status_idx ON analyses(status);
CREATE INDEX analyses_created_idx ON analyses(created_at DESC);

-- FEM Jobs (for server-side execution tracking)
CREATE TABLE fem_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    analysis_id UUID NOT NULL REFERENCES analyses(id) ON DELETE CASCADE,
    worker_id TEXT,
    cores_allocated INTEGER DEFAULT 8,
    memory_mb INTEGER DEFAULT 32768,
    priority INTEGER DEFAULT 0,  -- Higher = more urgent
    progress_pct REAL DEFAULT 0.0,
    current_stage INTEGER DEFAULT 0,
    convergence_history JSONB DEFAULT '[]',  -- [{iteration, residual_norm, displacement_norm}]
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX fem_jobs_analysis_idx ON fem_jobs(analysis_id);
CREATE INDEX fem_jobs_worker_idx ON fem_jobs(worker_id);

-- Rock Mass Classifications (RMR, Q, GSI calculations)
CREATE TABLE rock_classifications (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    borehole_id UUID REFERENCES boreholes(id) ON DELETE CASCADE,
    depth_from REAL,
    depth_to REAL,
    classification_system TEXT NOT NULL,  -- rmr | q_system | gsi
    input_parameters JSONB NOT NULL,  -- System-specific inputs
    calculated_value REAL NOT NULL,  -- RMR score, Q-value, or GSI
    support_class TEXT,  -- Recommended support class from classification
    hoek_brown_params JSONB,  -- Derived mb, s, a parameters
    notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX rock_class_project_idx ON rock_classifications(project_id);
CREATE INDEX rock_class_borehole_idx ON rock_classifications(borehole_id);

-- Settlement Predictions
CREATE TABLE settlement_predictions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    analysis_id UUID REFERENCES analyses(id) ON DELETE SET NULL,  -- NULL for empirical-only
    chainage REAL NOT NULL,  -- m along tunnel alignment
    prediction_method TEXT NOT NULL,  -- gaussian_trough | fem_3d | empirical
    volume_loss_pct REAL,  -- %
    trough_width_param_k REAL,  -- Dimensionless
    max_settlement_mm REAL,
    settlement_profile_data JSONB,  -- [{offset_m, settlement_mm}] transverse profile
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX settlement_project_idx ON settlement_predictions(project_id);
CREATE INDEX settlement_analysis_idx ON settlement_predictions(analysis_id);

-- Monitoring Data (instrumentation readings)
CREATE TABLE monitoring_points (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name TEXT NOT NULL,  -- e.g., "MP-01", "INCL-05"
    instrument_type TEXT NOT NULL,  -- convergence_pin | extensometer | inclinometer | settlement_point | piezometer
    location GEOMETRY(POINTZ, 4326) NOT NULL,
    installation_date DATE,
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX monitoring_project_idx ON monitoring_points(project_id);
CREATE INDEX monitoring_location_gist ON monitoring_points USING GIST(location);

CREATE TABLE monitoring_readings (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    monitoring_point_id UUID NOT NULL REFERENCES monitoring_points(id) ON DELETE CASCADE,
    reading_date TIMESTAMPTZ NOT NULL,
    value REAL NOT NULL,  -- mm, degrees, kPa depending on instrument type
    chainage_ref REAL,  -- m, tunnel face position at time of reading
    notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX readings_point_idx ON monitoring_readings(monitoring_point_id);
CREATE INDEX readings_date_idx ON monitoring_readings(reading_date);

-- Usage Tracking (for billing enforcement)
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,  -- fem_core_hours | mesh_generation | storage_gb_month
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
use geo_types::Point;

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
    pub location: Option<String>,
    pub alignment_data: serde_json::Value,
    pub cross_section_data: serde_json::Value,
    pub ground_model_data: serde_json::Value,
    pub settings: serde_json::Value,
    pub is_public: bool,
    pub thumbnail_url: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Borehole {
    pub id: Uuid,
    pub project_id: Uuid,
    pub name: String,
    pub location: sqlx::types::Json<Point<f64>>,  // PostGIS point as GeoJSON
    pub ground_level_elevation: f32,
    pub total_depth: f32,
    pub date_drilled: Option<NaiveDate>,
    pub drilling_method: Option<String>,
    pub stratigraphy: serde_json::Value,
    pub water_levels: serde_json::Value,
    pub lab_tests: serde_json::Value,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct GroundMaterial {
    pub id: Uuid,
    pub project_id: Option<Uuid>,
    pub name: String,
    pub material_type: String,
    pub constitutive_model: String,
    pub parameters: serde_json::Value,
    pub unit_weight: Option<f32>,
    pub permeability: Option<f32>,
    pub description: Option<String>,
    pub is_builtin: bool,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Mesh {
    pub id: Uuid,
    pub project_id: Uuid,
    pub name: String,
    pub mesh_type: String,
    pub node_count: i32,
    pub element_count: i32,
    pub element_type: String,
    pub mesh_url: String,
    pub metadata: serde_json::Value,
    pub generation_time_sec: Option<f32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Analysis {
    pub id: Uuid,
    pub project_id: Uuid,
    pub mesh_id: Uuid,
    pub user_id: Uuid,
    pub name: String,
    pub analysis_type: String,
    pub status: String,
    pub excavation_stages: serde_json::Value,
    pub boundary_conditions: serde_json::Value,
    pub solver_settings: serde_json::Value,
    pub results_url: Option<String>,
    pub results_summary: Option<serde_json::Value>,
    pub error_message: Option<String>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub wall_time_sec: Option<f32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct FemJob {
    pub id: Uuid,
    pub analysis_id: Uuid,
    pub worker_id: Option<String>,
    pub cores_allocated: i32,
    pub memory_mb: i32,
    pub priority: i32,
    pub progress_pct: f32,
    pub current_stage: i32,
    pub convergence_history: serde_json::Value,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct RockClassification {
    pub id: Uuid,
    pub project_id: Uuid,
    pub borehole_id: Option<Uuid>,
    pub depth_from: Option<f32>,
    pub depth_to: Option<f32>,
    pub classification_system: String,
    pub input_parameters: serde_json::Value,
    pub calculated_value: f32,
    pub support_class: Option<String>,
    pub hoek_brown_params: Option<serde_json::Value>,
    pub notes: Option<String>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct SettlementPrediction {
    pub id: Uuid,
    pub project_id: Uuid,
    pub analysis_id: Option<Uuid>,
    pub chainage: f32,
    pub prediction_method: String,
    pub volume_loss_pct: Option<f32>,
    pub trough_width_param_k: Option<f32>,
    pub max_settlement_mm: Option<f32>,
    pub settlement_profile_data: Option<serde_json::Value>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct CreateAnalysisRequest {
    pub mesh_id: Uuid,
    pub name: String,
    pub analysis_type: AnalysisType,
    pub excavation_stages: Vec<ExcavationStage>,
    pub boundary_conditions: BoundaryConditions,
    pub solver_settings: SolverSettings,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum AnalysisType {
    Static,
    SequentialExcavation,
    TransientSettlement,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct ExcavationStage {
    pub stage_num: u32,
    pub description: String,
    pub elements_removed: Vec<u32>,  // Element IDs to deactivate
    pub support_installed: Vec<SupportElement>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct SupportElement {
    pub support_type: String,  // shotcrete | rock_bolt | steel_set
    pub nodes: Vec<u32>,
    pub properties: serde_json::Value,
    pub activation_stage: u32,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct BoundaryConditions {
    pub gravity: bool,
    pub in_situ_stress: InSituStress,
    pub fixed_nodes: Vec<u32>,
    pub displacement_bcs: Vec<DisplacementBC>,
    pub load_bcs: Vec<LoadBC>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct InSituStress {
    pub vertical_stress_gradient: f64,  // kPa/m
    pub k0_lateral: f64,  // K0 coefficient
    pub tectonic_stress_x: f64,  // kPa
    pub tectonic_stress_y: f64,  // kPa
}

#[derive(Debug, Deserialize, Serialize)]
pub struct DisplacementBC {
    pub node_id: u32,
    pub dof: String,  // "x" | "y" | "z"
    pub value: f64,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct LoadBC {
    pub node_id: u32,
    pub direction: String,  // "x" | "y" | "z"
    pub magnitude: f64,  // N or kN
}

#[derive(Debug, Deserialize, Serialize)]
pub struct SolverSettings {
    pub max_iterations: u32,  // Newton-Raphson
    pub convergence_tol: f64,
    pub solver_type: String,  // "mumps" | "pardiso"
    pub num_threads: u32,
}
```

---

## Solver Architecture Deep-Dive

### Governing Equations and Discretization

TunnelSim's FEM solver implements **3D elasto-plastic finite element analysis** with large-deformation formulation for tunnel excavation simulation. For a discretized domain with `n` nodes and 3 DOF per node, the equilibrium equation at each load/excavation increment is:

```
K(u) · Δu = R_ext - R_int(u)

where:
  K(u)     = tangent stiffness matrix (dependent on current displacement u for nonlinear materials)
  Δu       = incremental displacement vector (3n × 1)
  R_ext    = external force vector (gravity, boundary loads)
  R_int(u) = internal force vector = ∫ B^T σ dV  (integral of stress over element volumes)
  B        = strain-displacement matrix (relates nodal displacements to element strains)
  σ        = stress vector from constitutive model
```

**Element Formulation**: 10-node tetrahedral (Tet10) and 20-node hexahedral (Hex20) elements with quadratic shape functions and Gauss integration (4-point for Tet10, 8-point for Hex20). The strain-displacement relation is:

```
ε = B · u_e

where:
  ε    = [ε_xx, ε_yy, ε_zz, γ_xy, γ_yz, γ_xz]^T  (6 × 1 strain vector)
  B    = ∂N/∂x shape function derivatives (6 × 30 for Tet10)
  u_e  = element nodal displacement vector (30 × 1 for Tet10)
```

**Constitutive Models**:

1. **Mohr-Coulomb** (soils, weak rock):
   ```
   Yield function: f = (σ_1 - σ_3) + (σ_1 + σ_3) sin φ - 2c cos φ

   where:
     σ_1, σ_3 = major and minor principal stresses
     c        = cohesion (kPa)
     φ        = friction angle (degrees)
   ```
   Plastic flow with non-associated flow rule using dilation angle ψ < φ.

2. **Hoek-Brown** (rock masses):
   ```
   Yield function: f = σ_1 - σ_3 - σ_ci (m_b σ_3/σ_ci + s)^a

   where:
     σ_ci = uniaxial compressive strength of intact rock (MPa)
     m_b  = reduced material constant = m_i exp((GSI - 100)/(28 - 14D))
     s    = rock mass constant = exp((GSI - 100)/(9 - 3D))
     a    = 0.5 + (1/6)(exp(-GSI/15) - exp(-20/3))

     GSI = Geological Strength Index (0-100)
     m_i = intact rock material constant (tabulated by rock type)
     D   = disturbance factor (0 = undisturbed, 1 = fully disturbed)
   ```

**Newton-Raphson Nonlinear Solution**: At each load increment, iterate until convergence:

```
Iteration k:
  1. Compute internal forces: R_int = ∫ B^T σ(ε) dV
  2. Compute residual: R = R_ext - R_int
  3. Form tangent stiffness: K_T = ∫ B^T D_ep B dV
     where D_ep = elasto-plastic constitutive matrix
  4. Solve: K_T Δu = R
  5. Update: u_{k+1} = u_k + Δu
  6. Check convergence: ||R|| / ||R_ext|| < tol (typically 1e-3)
```

**Excavation Simulation (Stress Relief)**: Tunnel excavation is modeled by:
1. Initially, all elements active with in-situ stress σ_0 = K_0 · γ · z
2. At excavation stage i, elements inside excavation boundary are deactivated:
   - Apply reversed body forces to nodes: F_unload = -∫ B^T σ_0 dV
   - Remove element stiffness contribution from global K
3. Support elements (shotcrete, rock bolts) activated at specified offset behind face:
   - Add beam/truss element stiffness to global K
   - Apply support reaction forces

### Mesh Generation Pipeline

```
Tunnel alignment + cross-section → Advancing-front tetrahedral mesher

1. Generate 2D cross-section mesh (structured quad/tri)
2. Sweep along 3D alignment curve → hexahedral tunnel core
3. Far-field tetrahedral infill using Delaunay
4. Refinement zones:
   - Fine mesh (0.5m) within 2D tunnel diameters
   - Medium mesh (1-2m) from 2D to 5D
   - Coarse mesh (5m) far field (up to 10D boundary)
5. Mesh quality checks:
   - Aspect ratio < 5:1
   - Jacobian > 0.3
   - Skewness < 0.7
```

For a typical 10m diameter tunnel with 100m length:
- Tunnel core: ~15K hex elements
- Near-field refinement: ~100K tet elements
- Far-field: ~50K tet elements
- **Total: ~165K elements, ~250K nodes, ~750K DOF**

### Convergence-Confinement Method

The convergence-confinement method (CCM) relates tunnel closure to support pressure via analytical curves, providing rapid design estimates validated against full 3D FEM.

**Ground Reaction Curve (GRC)**: Relationship between radial tunnel wall displacement u_r and internal support pressure p_i, derived from closed-form elastic-plastic cavity expansion solution:

```
For Mohr-Coulomb material in elastic-perfectly-plastic regime:

u_r(p_i) = (p_0 - p_i) · R / (2G) · [1 + (R_p/R)^2]

where:
  p_0  = in-situ radial stress = K_0 · γ · z
  R    = tunnel radius
  G    = shear modulus
  R_p  = plastic zone radius = R · (2(p_0 - p_i) / (c cot φ + p_0 (1 + sin φ)))^[1/(2 sin φ)]
```

**Support Characteristic Curve (SCC)**: Relationship between support pressure p_s and radial deformation u_r for a given support system (e.g., shotcrete + rock bolts):

```
p_s(u_r) = k_support · u_r

where k_support = E_shotcrete · t / (R (1 - ν^2)) + Σ(rock bolt stiffnesses)
```

**Equilibrium Point**: Intersection of GRC and SCC gives the tunnel closure at equilibrium and required support pressure.

---

## Architecture Deep-Dives

### 1. FEM Analysis API Handler (Rust/Axum)

The primary endpoint receives an FEM analysis request, validates mesh and material assignments, enqueues the job with Redis, and streams progress via WebSocket.

```rust
// src/api/handlers/analysis.rs

use axum::{
    extract::{Path, State, Json},
    http::StatusCode,
    response::IntoResponse,
};
use uuid::Uuid;
use crate::{
    db::models::{Analysis, CreateAnalysisRequest, AnalysisType},
    solver::mesh::validate_mesh,
    state::AppState,
    auth::Claims,
    error::ApiError,
};

pub async fn create_analysis(
    State(state): State<AppState>,
    claims: Claims,
    Path(project_id): Path<Uuid>,
    Json(req): Json<CreateAnalysisRequest>,
) -> Result<impl IntoResponse, ApiError> {
    // 1. Verify project ownership and mesh existence
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

    let mesh = sqlx::query_as!(
        crate::db::models::Mesh,
        "SELECT * FROM meshes WHERE id = $1 AND project_id = $2",
        req.mesh_id,
        project_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Mesh not found"))?;

    // 2. Check plan limits
    let user = sqlx::query_as!(
        crate::db::models::User,
        "SELECT * FROM users WHERE id = $1",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    if user.plan == "free" && mesh.element_count > 50000 {
        return Err(ApiError::PlanLimit(
            "Free plan supports meshes up to 50K elements. Upgrade to Pro for unlimited."
        ));
    }

    if user.plan == "pro" && matches!(req.analysis_type, AnalysisType::SequentialExcavation)
        && req.excavation_stages.len() > 50 {
        return Err(ApiError::PlanLimit(
            "Pro plan supports up to 50 excavation stages. Upgrade to Advanced for unlimited."
        ));
    }

    // 3. Validate mesh quality (download from S3 and check)
    let mesh_data = state.s3_client.get_object()
        .bucket(&state.config.s3_bucket)
        .key(&mesh.mesh_url)
        .send()
        .await?;

    validate_mesh(&mesh_data).await?;

    // 4. Create analysis record
    let analysis = sqlx::query_as!(
        Analysis,
        r#"INSERT INTO analyses
            (project_id, mesh_id, user_id, name, analysis_type, status,
             excavation_stages, boundary_conditions, solver_settings)
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9)
        RETURNING *"#,
        project_id,
        req.mesh_id,
        claims.user_id,
        req.name,
        serde_json::to_string(&req.analysis_type)?,
        "pending",
        serde_json::to_value(&req.excavation_stages)?,
        serde_json::to_value(&req.boundary_conditions)?,
        serde_json::to_value(&req.solver_settings)?,
    )
    .fetch_one(&state.db)
    .await?;

    // 5. Create FEM job and enqueue
    let cores = match user.plan.as_str() {
        "advanced" => 32,
        "pro" => 16,
        _ => 8,
    };

    let job = sqlx::query_as!(
        crate::db::models::FemJob,
        r#"INSERT INTO fem_jobs (analysis_id, cores_allocated, memory_mb, priority)
        VALUES ($1, $2, $3, $4) RETURNING *"#,
        analysis.id,
        cores as i32,
        cores as i32 * 4096,  // 4GB per core
        if user.plan == "advanced" { 10 } else { 0 },
    )
    .fetch_one(&state.db)
    .await?;

    // Publish to Redis job queue
    state.redis
        .publish("fem:jobs", serde_json::to_string(&job.id)?)
        .await?;

    Ok((StatusCode::CREATED, Json(analysis)))
}

pub async fn get_analysis_status(
    State(state): State<AppState>,
    claims: Claims,
    Path((project_id, analysis_id)): Path<(Uuid, Uuid)>,
) -> Result<Json<Analysis>, ApiError> {
    let analysis = sqlx::query_as!(
        Analysis,
        "SELECT a.* FROM analyses a
         JOIN projects p ON a.project_id = p.id
         WHERE a.id = $1 AND a.project_id = $2
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

pub async fn cancel_analysis(
    State(state): State<AppState>,
    claims: Claims,
    Path((project_id, analysis_id)): Path<(Uuid, Uuid)>,
) -> Result<StatusCode, ApiError> {
    // Update status to cancelled
    let result = sqlx::query!(
        "UPDATE analyses SET status = 'cancelled'
         WHERE id = $1 AND project_id = $2
         AND user_id = $3 AND status IN ('pending', 'queued', 'running')",
        analysis_id,
        project_id,
        claims.user_id
    )
    .execute(&state.db)
    .await?;

    if result.rows_affected() == 0 {
        return Err(ApiError::NotFound("Analysis not found or cannot be cancelled"));
    }

    // Signal worker to stop (via Redis pub/sub)
    state.redis
        .publish("fem:cancel", serde_json::to_string(&analysis_id)?)
        .await?;

    Ok(StatusCode::NO_CONTENT)
}
```

### 2. FEM Solver Core (Rust — server-side only)

The core 3D elasto-plastic FEM solver that assembles global stiffness matrix, applies boundary conditions, and solves via MUMPS sparse direct solver.

```rust
// solver-core/src/fem.rs

use nalgebra_sparse::{CsrMatrix, CooMatrix};
use crate::element::{Element, Tet10, Hex20};
use crate::material::{Material, MohrCoulomb, HoekBrown};
use crate::mumps::MumpsSolver;

pub struct FemSystem {
    pub n_nodes: usize,
    pub n_dof: usize,  // 3 * n_nodes
    pub elements: Vec<Box<dyn Element>>,
    pub materials: Vec<Box<dyn Material>>,
    pub global_k: CooMatrix<f64>,
    pub r_ext: Vec<f64>,  // External force vector
    pub r_int: Vec<f64>,  // Internal force vector
    pub u: Vec<f64>,      // Displacement solution
    pub u_prev: Vec<f64>, // Previous Newton iteration
    pub fixed_dofs: Vec<usize>,  // Indices of fixed DOFs
}

impl FemSystem {
    pub fn new(n_nodes: usize) -> Self {
        let n_dof = 3 * n_nodes;
        Self {
            n_nodes,
            n_dof,
            elements: Vec::new(),
            materials: Vec::new(),
            global_k: CooMatrix::new(n_dof, n_dof),
            r_ext: vec![0.0; n_dof],
            r_int: vec![0.0; n_dof],
            u: vec![0.0; n_dof],
            u_prev: vec![0.0; n_dof],
            fixed_dofs: Vec::new(),
        }
    }

    /// Assemble global stiffness matrix from element stiffnesses
    pub fn assemble_stiffness(&mut self) {
        self.global_k = CooMatrix::new(self.n_dof, self.n_dof);

        for elem in &self.elements {
            if !elem.is_active() {
                continue;  // Skip deactivated (excavated) elements
            }

            let material = &self.materials[elem.material_id()];
            let k_elem = elem.element_stiffness(material, &self.u);
            let dofs = elem.global_dofs();

            // Stamp element stiffness into global matrix
            for (i_local, &i_global) in dofs.iter().enumerate() {
                for (j_local, &j_global) in dofs.iter().enumerate() {
                    self.global_k.push(i_global, j_global, k_elem[(i_local, j_local)]);
                }
            }
        }
    }

    /// Compute internal force vector from element stresses
    pub fn compute_internal_forces(&mut self) {
        self.r_int.fill(0.0);

        for elem in &self.elements {
            if !elem.is_active() {
                continue;
            }

            let material = &self.materials[elem.material_id()];
            let f_int_elem = elem.internal_forces(material, &self.u);
            let dofs = elem.global_dofs();

            for (i_local, &i_global) in dofs.iter().enumerate() {
                self.r_int[i_global] += f_int_elem[i_local];
            }
        }
    }

    /// Apply gravity loads
    pub fn apply_gravity(&mut self, gravity_accel: f64) {
        for elem in &self.elements {
            if !elem.is_active() {
                continue;
            }

            let material = &self.materials[elem.material_id()];
            let rho = material.density();
            let volume = elem.volume();
            let f_gravity = rho * gravity_accel * volume / elem.n_nodes() as f64;

            let dofs = elem.global_dofs();
            for &dof in &dofs {
                if dof % 3 == 2 {  // Z-direction (vertical)
                    self.r_ext[dof] -= f_gravity;
                }
            }
        }
    }

    /// Apply boundary conditions to stiffness matrix and force vector
    pub fn apply_bcs(&mut self) {
        for &dof in &self.fixed_dofs {
            // Zero out row and column, set diagonal to 1, RHS to 0
            self.r_ext[dof] = 0.0;

            // Zero column
            for i in 0..self.n_dof {
                if i != dof {
                    self.r_ext[i] -= self.global_k.get(i, dof).unwrap_or(0.0) * self.u[dof];
                }
            }
        }

        // Convert to CSR and zero rows/cols for BCs
        let mut csr = CsrMatrix::from(&self.global_k);
        for &dof in &self.fixed_dofs {
            // Set row to identity
            // (actual implementation uses CSR row slicing)
        }
    }

    /// Solve K u = R using MUMPS
    pub fn solve(&mut self) -> Result<(), SolverError> {
        let csr = CsrMatrix::from(&self.global_k);
        let mut solver = MumpsSolver::new(&csr, self.fixed_dofs.clone())?;

        let residual: Vec<f64> = self.r_ext.iter()
            .zip(self.r_int.iter())
            .map(|(ext, int)| ext - int)
            .collect();

        let du = solver.solve(&residual)?;

        for i in 0..self.n_dof {
            self.u[i] += du[i];
        }

        Ok(())
    }

    /// Newton-Raphson nonlinear solver
    pub fn solve_nonlinear(
        &mut self,
        max_iter: usize,
        tol: f64,
    ) -> Result<usize, SolverError> {
        for iter in 0..max_iter {
            self.assemble_stiffness();
            self.compute_internal_forces();
            self.apply_bcs();

            self.u_prev = self.u.clone();
            self.solve()?;

            // Check convergence
            let residual_norm: f64 = self.r_ext.iter()
                .zip(self.r_int.iter())
                .map(|(ext, int)| (ext - int).powi(2))
                .sum::<f64>()
                .sqrt();

            let force_norm: f64 = self.r_ext.iter()
                .map(|f| f.powi(2))
                .sum::<f64>()
                .sqrt();

            let relative_error = residual_norm / force_norm.max(1.0);

            if relative_error < tol {
                return Ok(iter + 1);
            }
        }

        Err(SolverError::ConvergenceFailed)
    }

    /// Sequential excavation analysis
    pub fn solve_sequential_excavation(
        &mut self,
        stages: &[ExcavationStage],
        settings: &SolverSettings,
    ) -> Result<Vec<StageResult>, SolverError> {
        let mut results = Vec::new();

        for (stage_num, stage) in stages.iter().enumerate() {
            println!("Stage {}: {}", stage_num, stage.description);

            // Deactivate excavated elements
            for &elem_id in &stage.elements_removed {
                self.elements[elem_id].set_active(false);
            }

            // Install support elements
            for support in &stage.support_installed {
                // Add shotcrete beam elements or rock bolt truss elements
                // (implementation depends on support type)
            }

            // Solve nonlinear equilibrium
            let iterations = self.solve_nonlinear(
                settings.max_iterations as usize,
                settings.convergence_tol,
            )?;

            // Extract results
            let max_disp = self.u.iter()
                .step_by(3)
                .map(|&u| u.abs())
                .fold(0.0f64, f64::max);

            results.push(StageResult {
                stage_num: stage_num as u32,
                iterations,
                max_displacement: max_disp,
                displacements: self.u.clone(),
            });
        }

        Ok(results)
    }
}

#[derive(Debug, Clone)]
pub struct StageResult {
    pub stage_num: u32,
    pub iterations: usize,
    pub max_displacement: f64,
    pub displacements: Vec<f64>,
}

#[derive(Debug)]
pub enum SolverError {
    MatrixSingular,
    ConvergenceFailed,
    MumpsError(String),
}
```

### 3. Rock Mass Classification Service (Python/FastAPI)

Python service exposing REST API for rock mass classification calculations (RMR, Q-system, GSI) and Hoek-Brown parameter derivation.

```python
# geotechnical-service/app/classification.py

from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, Field
from typing import Optional
import math

app = FastAPI()

class RMRInput(BaseModel):
    ucs_mpa: float = Field(..., description="Uniaxial compressive strength (MPa)")
    rqd_percent: float = Field(..., ge=0, le=100, description="RQD (%)")
    joint_spacing_m: float = Field(..., gt=0, description="Joint spacing (m)")
    joint_condition: str = Field(..., description="very_rough | rough | smooth | slickensided")
    groundwater: str = Field(..., description="dry | damp | wet | dripping | flowing")
    joint_orientation: str = Field(..., description="very_favorable | favorable | fair | unfavorable | very_unfavorable")

class RMROutput(BaseModel):
    rmr_basic: float
    rmr_adjusted: float
    rock_class: str
    description: str
    support_recommendation: str

@app.post("/api/classify/rmr", response_model=RMROutput)
def calculate_rmr(input: RMRInput):
    """Calculate RMR (Bieniawski 1989) from input parameters."""

    # Rating 1: UCS
    if input.ucs_mpa > 250:
        r1 = 15
    elif input.ucs_mpa > 100:
        r1 = 12
    elif input.ucs_mpa > 50:
        r1 = 7
    elif input.ucs_mpa > 25:
        r1 = 4
    elif input.ucs_mpa > 5:
        r1 = 2
    elif input.ucs_mpa > 1:
        r1 = 1
    else:
        r1 = 0

    # Rating 2: RQD
    if input.rqd_percent >= 90:
        r2 = 20
    elif input.rqd_percent >= 75:
        r2 = 17
    elif input.rqd_percent >= 50:
        r2 = 13
    elif input.rqd_percent >= 25:
        r2 = 8
    else:
        r2 = 3

    # Rating 3: Joint spacing
    if input.joint_spacing_m > 2.0:
        r3 = 20
    elif input.joint_spacing_m > 0.6:
        r3 = 15
    elif input.joint_spacing_m > 0.2:
        r3 = 10
    elif input.joint_spacing_m > 0.06:
        r3 = 8
    else:
        r3 = 5

    # Rating 4: Joint condition
    joint_ratings = {
        "very_rough": 30,
        "rough": 25,
        "smooth": 20,
        "slickensided": 10,
    }
    r4 = joint_ratings.get(input.joint_condition, 15)

    # Rating 5: Groundwater
    gw_ratings = {
        "dry": 15,
        "damp": 10,
        "wet": 7,
        "dripping": 4,
        "flowing": 0,
    }
    r5 = gw_ratings.get(input.groundwater, 7)

    rmr_basic = r1 + r2 + r3 + r4 + r5

    # Adjustment for joint orientation
    orient_adjustments = {
        "very_favorable": 0,
        "favorable": -2,
        "fair": -5,
        "unfavorable": -10,
        "very_unfavorable": -12,
    }
    adjustment = orient_adjustments.get(input.joint_orientation, -5)
    rmr_adjusted = max(0, min(100, rmr_basic + adjustment))

    # Rock class
    if rmr_adjusted >= 81:
        rock_class = "I - Very Good Rock"
        description = "Hard rock, tight interlocking, unweathered"
        support = "Generally no support required, spot bolting in crown"
    elif rmr_adjusted >= 61:
        rock_class = "II - Good Rock"
        description = "Fresh to slightly weathered, tight joints"
        support = "Local rock bolts 2-3m, shotcrete 50mm in crown"
    elif rmr_adjusted >= 41:
        rock_class = "III - Fair Rock"
        description = "Moderately weathered, moderately jointed"
        support = "Systematic rock bolts 1.5-2m, shotcrete 100mm, steel sets if needed"
    elif rmr_adjusted >= 21:
        rock_class = "IV - Poor Rock"
        description = "Highly weathered, heavily jointed"
        support = "Systematic rock bolts 1-1.5m, shotcrete 150-200mm, steel sets 1.5m spacing"
    else:
        rock_class = "V - Very Poor Rock"
        description = "Completely weathered, very closely jointed or sheared"
        support = "Multiple drift, forepoling, shotcrete 200mm+, steel sets <1m spacing"

    return RMROutput(
        rmr_basic=rmr_basic,
        rmr_adjusted=rmr_adjusted,
        rock_class=rock_class,
        description=description,
        support_recommendation=support,
    )


class HoekBrownInput(BaseModel):
    gsi: float = Field(..., ge=0, le=100, description="Geological Strength Index")
    mi: float = Field(..., description="Intact rock material constant")
    ucs_mpa: float = Field(..., description="Intact rock UCS (MPa)")
    disturbance: float = Field(0.0, ge=0, le=1, description="Disturbance factor D")

class HoekBrownOutput(BaseModel):
    mb: float
    s: float
    a: float
    mohr_coulomb_equiv: dict

@app.post("/api/hoek-brown/parameters", response_model=HoekBrownOutput)
def calculate_hoek_brown(input: HoekBrownInput):
    """Calculate Hoek-Brown parameters from GSI."""

    gsi = input.gsi
    mi = input.mi
    D = input.disturbance

    # Hoek-Brown 2002 equations
    mb = mi * math.exp((gsi - 100) / (28 - 14 * D))
    s = math.exp((gsi - 100) / (9 - 3 * D))
    a = 0.5 + (1/6) * (math.exp(-gsi/15) - math.exp(-20/3))

    # Mohr-Coulomb equivalent (for stress range 0 to σ_ci/4)
    sigma_3_max = input.ucs_mpa / 4
    sigma_t = -s * input.ucs_mpa / mb  # Tensile strength

    # Simplified equivalent c and phi
    phi_deg = math.degrees(math.asin(
        (6*a*mb*(s + mb*sigma_3_max)**(a-1)) /
        (2*(1+a)*(2+a) + 6*a*mb*(s + mb*sigma_3_max)**(a-1))
    ))

    c_mpa = input.ucs_mpa * ((1+2*a)*s + (1-a)*mb*sigma_3_max) * \
            (s + mb*sigma_3_max)**(a-1) / \
            ((1+a)*(2+a) * math.sqrt(1 + (6*a*mb*(s+mb*sigma_3_max)**(a-1)) / ((1+a)*(2+a))))

    return HoekBrownOutput(
        mb=mb,
        s=s,
        a=a,
        mohr_coulomb_equiv={
            "cohesion_mpa": c_mpa,
            "friction_angle_deg": phi_deg,
            "tensile_strength_mpa": sigma_t,
            "valid_range_mpa": sigma_3_max,
        }
    )
```

### 4. Settlement Gaussian Trough Calculator (React/TypeScript)

Client-side calculator for empirical greenfield settlement prediction using Gaussian distribution.

```typescript
// frontend/src/lib/settlement.ts

export interface GaussianTroughParams {
  tunnelDepth: number;        // m, to tunnel axis
  tunnelDiameter: number;     // m
  volumeLoss: number;         // %, typical 0.5-2% for EPB, 1-3% for NATM
  troughWidthK: number;       // dimensionless, typical 0.4-0.6
}

export interface SettlementProfile {
  offsetMeters: number[];
  settlementMm: number[];
  maxSettlementMm: number;
  inflectionPointM: number;
}

/**
 * Calculate Gaussian settlement trough (Peck 1969, O'Reilly & New 1982)
 *
 * S(x) = S_max * exp(-(x/i)²)
 *
 * where:
 *   S_max = V_loss * D² / (2 * i)  [volume loss per unit length distributed to surface]
 *   i = K * z_0                     [trough width parameter]
 *   x = horizontal offset from tunnel centerline
 */
export function calculateGaussianTrough(
  params: GaussianTroughParams,
  maxOffset: number = 50,  // m
  numPoints: number = 101
): SettlementProfile {
  const { tunnelDepth, tunnelDiameter, volumeLoss, troughWidthK } = params;

  // Trough width parameter i = K * z0
  const i = troughWidthK * tunnelDepth;

  // Volume loss per unit length (m³/m)
  const vLoss = (volumeLoss / 100) * Math.PI * (tunnelDiameter / 2) ** 2;

  // Maximum settlement (at centerline)
  const sMax = vLoss / (i * Math.sqrt(2 * Math.PI));  // m
  const sMaxMm = sMax * 1000;  // mm

  // Generate profile
  const offsetMeters: number[] = [];
  const settlementMm: number[] = [];

  for (let j = 0; j < numPoints; j++) {
    const x = -maxOffset + (2 * maxOffset * j) / (numPoints - 1);
    const s = sMaxMm * Math.exp(-(x / i) ** 2);

    offsetMeters.push(x);
    settlementMm.push(s);
  }

  return {
    offsetMeters,
    settlementMm,
    maxSettlementMm: sMaxMm,
    inflectionPointM: i,  // Inflection point at x = i
  };
}

/**
 * Calculate longitudinal settlement profile ahead of and behind tunnel face
 */
export function calculateLongitudinalProfile(
  params: GaussianTroughParams,
  facePosition: number,  // m chainage
  chainageRange: [number, number],  // [start, end] chainage
  numPoints: number = 51
): SettlementProfile {
  const { tunnelDepth, tunnelDiameter, volumeLoss, troughWidthK } = params;

  const i = troughWidthK * tunnelDepth;
  const vLoss = (volumeLoss / 100) * Math.PI * (tunnelDiameter / 2) ** 2;
  const sMaxMm = (vLoss / (i * Math.sqrt(2 * Math.PI))) * 1000;

  const chainageMeters: number[] = [];
  const settlementMm: number[] = [];

  for (let j = 0; j < numPoints; j++) {
    const ch = chainageRange[0] + ((chainageRange[1] - chainageRange[0]) * j) / (numPoints - 1);
    const distFromFace = ch - facePosition;

    // Settlement develops from about -1D ahead of face to +3D behind
    // Cumulative error function approximation
    let cumulativeFactor: number;
    if (distFromFace < -tunnelDiameter) {
      cumulativeFactor = 0;
    } else if (distFromFace > 3 * tunnelDiameter) {
      cumulativeFactor = 1;
    } else {
      const normalized = (distFromFace + tunnelDiameter) / (4 * tunnelDiameter);
      cumulativeFactor = Math.max(0, Math.min(1, normalized));
    }

    chainageMeters.push(ch);
    settlementMm.push(sMaxMm * cumulativeFactor);
  }

  return {
    offsetMeters: chainageMeters,
    settlementMm,
    maxSettlementMm: sMaxMm,
    inflectionPointM: i,
  };
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–5)

**Day 1: Project initialization and Rust backend scaffold**
```bash
cargo init tunnelsim-api
cd tunnelsim-api
cargo add axum tokio serde serde_json sqlx uuid chrono tower-http jsonwebtoken bcrypt
cargo add --features postgres,runtime-tokio-rustls,uuid,chrono,json sqlx
cargo add aws-sdk-s3 redis nalgebra-sparse geo-types
```
- `src/main.rs` — Axum app with CORS, tracing, graceful shutdown
- `src/config.rs` — Environment-based configuration (DATABASE_URL, JWT_SECRET, S3_BUCKET, REDIS_URL)
- `src/state.rs` — AppState struct (PgPool, Redis, S3Client)
- `src/error.rs` — ApiError enum with IntoResponse
- `Dockerfile` — Multi-stage build (builder + runtime with MUMPS libraries)
- `docker-compose.yml` — PostgreSQL with PostGIS, Redis, MinIO (S3-compatible)

**Day 2: Database schema and migrations**
- `migrations/001_initial.sql` — All 10 tables: users, organizations, org_members, projects, boreholes, ground_materials, meshes, analyses, fem_jobs, rock_classifications, settlement_predictions, monitoring_points, monitoring_readings, usage_records
- `src/db/mod.rs` — Database pool initialization with PostGIS support
- `src/db/models.rs` — All SQLx structs with FromRow derives
- Run `sqlx migrate run` to apply schema
- Seed script for built-in ground materials (sand, clay, weak rock, hard rock with default Mohr-Coulomb/Hoek-Brown parameters)

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

**Day 5: Python geotechnical service scaffold**
```bash
cd geotechnical-service
python -m venv venv && source venv/bin/activate
pip install fastapi uvicorn pydantic numpy scipy pykrige mplstereonet
```
- `app/main.py` — FastAPI app with CORS
- `app/classification.py` — RMR, Q-system, GSI calculation endpoints (as shown in deep-dive)
- `app/hoek_brown.py` — Hoek-Brown parameter calculator
- `app/kriging.py` — Geostatistical interpolation endpoint (boreholes → 3D property field)
- `Dockerfile` — Python 3.12 image with scientific libraries
- Docker Compose integration with Rust backend

### Phase 2 — FEM Solver Core (Days 6–14)

**Day 6: FEM framework and element library**
- `solver-core/` — New Rust workspace member
- `solver-core/src/element/mod.rs` — Element trait definition (stiffness, internal forces, volume, dofs)
- `solver-core/src/element/tet10.rs` — 10-node tetrahedral element with quadratic shape functions
- `solver-core/src/element/hex20.rs` — 20-node hexahedral element
- Gauss integration points and weights (4-point for Tet10, 8-point for Hex20)
- Unit tests: single-element patch tests (constant stress, pure bending)

**Day 7: Material models — Mohr-Coulomb**
- `solver-core/src/material/mod.rs` — Material trait (constitutive matrix D_ep, stress update)
- `solver-core/src/material/mohr_coulomb.rs` — Mohr-Coulomb elasto-plastic model with non-associated flow rule
- Return mapping algorithm for plastic stress correction
- Apex singularity handling (tension cutoff)
- Tests: triaxial compression test, simple shear, uniaxial compression

**Day 8: Material models — Hoek-Brown**
- `solver-core/src/material/hoek_brown.rs` — Hoek-Brown failure criterion with GSI-based parameters
- Equivalent Mohr-Coulomb parameters for tangent stiffness
- Plastic strain calculation
- Tests: compare to analytical solutions for cylindrical cavity expansion

**Day 9: FEM system assembly and linear solver integration**
- `solver-core/src/fem.rs` — FemSystem struct (as shown in deep-dive) with global assembly
- `solver-core/src/mumps.rs` — MUMPS sparse direct solver bindings via FFI
- Alternative: PARDISO solver for Intel MKL
- Boundary condition application (fixed DOFs, prescribed displacements)
- Tests: cantilever beam bending, linear elastic cube under gravity

**Day 10: Nonlinear Newton-Raphson solver**
- `solver-core/src/fem.rs` — Newton-Raphson iteration loop with convergence checks
- Line search for robustness (backtracking when residual increases)
- Arc-length continuation for snap-through problems
- Convergence diagnostics: residual norm, displacement increment, energy norm
- Tests: elastic-plastic cylinder under internal pressure

**Day 11: Excavation simulation — element deactivation**
- `solver-core/src/excavation.rs` — ExcavationStage struct, element activation/deactivation logic
- Stress relief forces: compute unbalanced nodal forces from removed elements
- Sequential stage solver (loop over stages, re-equilibrate after each)
- Tests: simple tunnel excavation in elastic medium, compare to analytical cavity expansion

**Day 12: Support element library**
- `solver-core/src/element/beam.rs` — 2-node Euler-Bernoulli beam element for shotcrete lining
- `solver-core/src/element/truss.rs` — 2-node truss element for rock bolts
- Embedded element formulation (support nodes don't coincide with mesh nodes)
- Activation timing (install support at stage i, activate at stage i+k)
- Tests: beam on elastic foundation, cable-stayed structure

**Day 13: Mesh generation framework**
- `mesher/` — New Rust workspace member
- `mesher/src/advancing_front.rs` — Advancing-front tetrahedral mesher
- `mesher/src/structured.rs` — Structured hexahedral mesh sweep along tunnel alignment
- `mesher/src/delaunay.rs` — Delaunay triangulation for far-field infill (via `delaunator` crate)
- `mesher/src/quality.rs` — Mesh quality checks (aspect ratio, Jacobian, skewness)
- Unit tests: cube mesh, cylinder mesh, L-shaped domain

**Day 14: Tunnel-specific mesh generation**
- `mesher/src/tunnel.rs` — Tunnel mesh generator from alignment + cross-section
- Parse alignment curves (horizontal: circular arcs + tangents; vertical: parabolic curves)
- Cross-section templates: circular, horseshoe, D-shape, rectangular
- Refinement zones: 0.5m elements near tunnel, 1-2m medium field, 5m far field
- HDF5 export format with node coordinates, element connectivity, material IDs
- Tests: 100m straight tunnel with 10m diameter, verify element count ~165K

### Phase 3 — Frontend + 3D Visualization (Days 15–22)

**Day 15: Frontend scaffold and project management UI**
```bash
npm create vite@latest frontend -- --template react-ts
cd frontend
npm i zustand @tanstack/react-query axios three @react-three/fiber @react-three/drei d3
```
- `src/App.tsx` — Router (react-router-dom), auth context, layout
- `src/stores/projectStore.ts` — Zustand store for project state
- `src/pages/Dashboard.tsx` — Project list with cards, search/filter, create new project
- `src/pages/ProjectView.tsx` — Main project workspace with tabs (Alignment, Ground, Mesh, Analysis, Results)
- `src/components/Navbar.tsx` — Nav with user menu, upgrade button

**Day 16: Alignment editor (2D plan view)**
- `src/components/AlignmentEditor/Canvas.tsx` — SVG canvas with pan/zoom
- `src/components/AlignmentEditor/HorizontalCurve.tsx` — Draw horizontal alignment: straight segments, circular arcs
- `src/components/AlignmentEditor/VerticalProfile.tsx` — Vertical profile editor (side view): grades, parabolic curves
- `src/components/AlignmentEditor/Chainage.tsx` — Chainage markers every 20m
- Snap-to-grid, undo/redo, import from LandXML
- Properties panel: curve radius, tangent length, grade %

**Day 17: Cross-section editor**
- `src/components/CrossSection/Templates.tsx` — Template library sidebar (circular, horseshoe, D-shape, rectangular)
- `src/components/CrossSection/DimensionEditor.tsx` — Parametric dimension editor (diameter, width, height, crown radius)
- `src/components/CrossSection/Preview.tsx` — SVG preview of cross-section
- Support for variable cross-sections along alignment (transitions)
- Export cross-section as DXF for CAD integration

**Day 18: Three.js 3D tunnel viewer**
- `src/components/TunnelViewer/Scene.tsx` — React Three Fiber canvas with orbit controls
- `src/components/TunnelViewer/TunnelGeometry.tsx` — Swept tube geometry along alignment curve
- `src/components/TunnelViewer/GroundLayers.tsx` — 3D stratigraphic layers from borehole data
- `src/components/TunnelViewer/BoreholeMarkers.tsx` — Borehole locations as cylinders
- Cross-section clipping plane at user-selected chainage
- Lighting: ambient + directional, shadows enabled

**Day 19: Mesh visualization**
- `src/components/MeshViewer/MeshGeometry.tsx` — Load HDF5 mesh from S3, render as BufferGeometry
- Element coloring by material ID (color legend)
- Wireframe overlay toggle
- Node/element selection for inspection
- Level-of-detail: show only surface mesh for >500K elements, full mesh for smaller

**Day 20: Ground model and borehole UI**
- `src/components/GroundModel/BoreholeTable.tsx` — Tabular editor for borehole logs (depth intervals, description, properties)
- `src/components/GroundModel/BoreholeImport.tsx` — CSV/AGS file import
- `src/components/GroundModel/MaterialLibrary.tsx` — Material library with search, add custom material
- `src/components/GroundModel/MaterialAssignment.tsx` — Assign materials to mesh regions or layers
- Integration with Python service for RMR/GSI calculations (form inputs → API call → display results)

**Day 21: Rock mass classification tools**
- `src/components/RockClass/RMRCalculator.tsx` — Form for RMR inputs (UCS, RQD, joint spacing, etc.), display RMR score and support class
- `src/components/RockClass/QSystemCalculator.tsx` — Q-system calculator
- `src/components/RockClass/GSIChart.tsx` — Interactive GSI chart (structure vs. condition) with hover tooltips
- `src/components/RockClass/HoekBrownParams.tsx` — Display derived mb, s, a parameters and equivalent c, phi
- Save classification results to database linked to borehole/depth

**Day 22: Settlement calculator UI**
- `src/components/Settlement/GaussianTroughForm.tsx` — Input form: tunnel depth, diameter, volume loss, K
- `src/components/Settlement/SettlementPlot.tsx` — D3.js line chart for transverse and longitudinal profiles
- `src/components/Settlement/ProfileComparison.tsx` — Compare multiple profiles (different parameters or FEM vs. empirical)
- Export settlement profile as CSV
- Responsive tooltip showing settlement value at hover position

### Phase 4 — API + Job Orchestration (Days 23–28)

**Day 23: Mesh generation API**
- `src/api/handlers/mesh.rs` — Create mesh (POST), get mesh metadata (GET), list meshes (GET), delete mesh (DELETE)
- Mesh generation enqueued as background job via Redis
- Worker: `src/workers/mesh_worker.rs` — Consume mesh generation jobs, run Rust mesher, upload HDF5 to S3
- Progress tracking: mesh generation stages (geometry setup, core mesh, far-field mesh, quality checks)
- Plan limits: free 10K elements, pro 500K, advanced unlimited

**Day 24: FEM analysis API**
- `src/api/handlers/analysis.rs` — Create analysis (as shown in deep-dive), get status, list analyses, cancel
- Analysis enqueued to Redis with priority based on user plan
- Worker: `src/workers/fem_worker.rs` — Consume FEM jobs, load mesh from S3, run solver, upload results to S3
- Convergence history logged to database (fem_jobs table)

**Day 25: WebSocket for live progress**
- `src/api/ws/mod.rs` — WebSocket upgrade handler
- `src/api/ws/analysis_progress.rs` — Subscribe to analysis progress channel (Redis pub/sub → WebSocket)
- Client receives: `{ progress_pct, current_stage, current_iteration, residual_norm, max_displacement }`
- Frontend: `src/hooks/useAnalysisProgress.ts` — React hook that opens WebSocket on analysis start, updates UI live
- Connection cleanup on analysis completion or client disconnect

**Day 26: Settlement prediction API and integration**
- `src/api/handlers/settlement.rs` — Create settlement prediction (Gaussian or FEM-based), get, list
- For Gaussian: compute on-demand (fast), store profile in database
- For FEM-based: post-process FEM results to extract surface displacements at specified chainage
- Frontend integration: select analysis, specify chainage, compute and plot settlement profile

**Day 27: Monitoring data API**
- `src/api/handlers/monitoring.rs` — CRUD for monitoring points and readings
- Import CSV: timestamp, monitoring point ID, value
- `src/api/handlers/monitoring_plot.rs` — Time-series data endpoint for chart rendering
- Frontend: `src/components/Monitoring/TimeSeriesChart.tsx` — D3.js chart with predicted vs. measured overlay
- Alert system: compare readings against trigger levels, flag exceedances

**Day 28: Convergence-confinement calculator API**
- `src/api/handlers/ccm.rs` — Compute GRC/SCC endpoint
- Inputs: ground parameters (c, phi, E, nu), tunnel radius, depth, support properties
- Return: GRC curve (u_r vs p_i), SCC curve (u_r vs p_s), equilibrium point
- Frontend: `src/components/CCM/CCMCalculator.tsx` — Input form + D3.js plot showing GRC/SCC intersection
- Export GRC/SCC data as CSV

### Phase 5 — Results Visualization + Reporting (Days 29–33)

**Day 29: FEM results processing and download**
- `src/api/handlers/results.rs` — Get results metadata, generate presigned S3 URL for HDF5 download
- Results parser: `solver-core/src/results.rs` — Read HDF5, extract displacements, stresses, strains per node/element
- Result summary: max displacement, yielded elements count, convergence info
- Caching: hot results in Redis (1 hour TTL), cold in S3

**Day 30: Results viewer — displacement contours**
- `src/components/Results/DisplacementViewer.tsx` — Three.js mesh colored by displacement magnitude
- Vertex shader: color mapping from displacement value to color scale (blue → green → yellow → red)
- Displacement scale factor slider (exaggerate displacements for visibility)
- Displacement vector arrows at selected nodes
- Animation: play through excavation stages sequentially

**Day 31: Results viewer — stress contours**
- `src/components/Results/StressViewer.tsx` — Element coloring by stress component (σ_xx, σ_yy, σ_zz, τ_xy, principal stresses)
- Von Mises stress, mean stress, deviatoric stress
- Yielded elements highlighted (different color for elastic vs. plastic)
- Cross-section slice view at selected chainage

**Day 32: Convergence and monitoring plots**
- `src/components/Results/ConvergencePlot.tsx` — D3.js line chart: iteration vs. residual norm per stage
- `src/components/Results/DisplacementHistory.tsx` — Displacement at tunnel crown/springline vs. excavation stage
- `src/components/Results/MonitoringComparison.tsx` — Predicted displacement vs. measured (scatter plot with trend line)
- Export plots as PNG/SVG

**Day 33: Report generation**
- `src/api/handlers/report.rs` — Generate PDF report endpoint
- Report content: project summary, ground conditions, tunnel geometry, analysis results, settlement predictions, monitoring data
- Backend uses `printpdf` crate for PDF generation
- Include plots as embedded PNG images
- Customizable report sections (user selects which to include)
- Download as PDF or share via link

### Phase 6 — Billing + Plan Limits (Days 34–37)

**Day 34: Stripe integration**
- `src/api/handlers/billing.rs` — Create checkout session, customer portal session
- Stripe webhook handler: subscription created/updated/cancelled → update users table
- Plan upgrade flow: user clicks upgrade → redirect to Stripe checkout → webhook updates plan
- Frontend: `src/pages/Billing.tsx` — Current plan display, usage statistics, upgrade button

**Day 35: Plan limits enforcement**
- Middleware: `src/middleware/plan_limits.rs` — Check plan before mesh generation, analysis creation
- Free plan: 50K element meshes, 5 analyses/month, 2 projects
- Pro plan: 500K element meshes, unlimited analyses, unlimited projects, 50 excavation stages
- Advanced plan: unlimited everything, priority queue, 32-core instances
- Usage tracking: record FEM core-hours, mesh generation count, storage usage to usage_records table

**Day 36: Usage dashboard**
- `src/api/handlers/usage.rs` — Get usage summary for current billing period
- Frontend: `src/pages/Usage.tsx` — Charts showing FEM core-hours, mesh generation count, storage over time
- Quota warnings: notify user when approaching plan limits
- Overage handling: block new analyses if free plan quota exceeded

**Day 37: Admin dashboard (internal)**
- `src/api/handlers/admin.rs` — Admin-only endpoints (auth check for admin role)
- User management: list users, view usage, change plans manually
- System health: active FEM jobs, queue depth, worker status
- Cost monitoring: compute instance costs per user/org
- Frontend: `src/pages/Admin.tsx` — Admin dashboard with tables and charts

### Phase 7 — Testing + Documentation (Days 38–40)

**Day 38: FEM solver validation suite**
- `solver-core/tests/validation/` — Benchmark problems with analytical solutions
- Validation cases: elastic cylinder under internal pressure, Kirsch solution for circular hole, strip footing settlement
- Compare FEM results to analytical solutions, assert error < 2%
- Performance benchmarks: measure wall time for various mesh sizes
- CI integration: run validation suite on every PR

**Day 39: Integration tests**
- `tests/integration/` — End-to-end API tests
- Test flows: create project → create mesh → run analysis → fetch results
- Test plan limits: verify free user blocked at quota
- Test WebSocket: mock analysis progress messages
- Load testing with `k6`: simulate 100 concurrent users running analyses

**Day 40: Documentation and examples**
- `docs/` — User guide, API reference, tutorials
- Tutorial 1: Create simple tunnel project, run NATM excavation simulation
- Tutorial 2: Import borehole data, run rock mass classification, design support
- Tutorial 3: Settlement prediction and monitoring data comparison
- API docs: generate OpenAPI spec from Axum routes, host on Swagger UI
- Video walkthrough: record 10-minute demo of full workflow

### Phase 8 — Deployment + Launch Prep (Days 41–42)

**Day 41: Production deployment**
- AWS infrastructure: ECS Fargate for API, c7g.8xlarge spot instances for FEM workers, RDS PostgreSQL, ElastiCache Redis, S3
- Terraform/CDK: infrastructure as code for reproducible deployment
- CI/CD pipeline: GitHub Actions → build Docker images → push to ECR → deploy to ECS
- Monitoring: CloudWatch logs, Prometheus metrics, Grafana dashboards, Sentry error tracking
- Secrets management: AWS Secrets Manager for DATABASE_URL, JWT_SECRET, STRIPE_SECRET_KEY

**Day 42: Performance optimization and launch**
- Database indexing: verify all query patterns have indexes, run EXPLAIN ANALYZE
- FEM solver profiling: flamegraph analysis, optimize hot loops
- CDN setup: CloudFront for frontend assets, edge caching for public project thumbnails
- Load testing: stress test with 500 concurrent users, verify auto-scaling works
- Backup strategy: automated daily RDS snapshots, S3 versioning for meshes/results
- Launch checklist: DNS setup, SSL certs, GDPR compliance, privacy policy, terms of service

---

## Critical Files Tree

```
tunnelsim/
├── backend/                         # Rust Axum API
│   ├── Cargo.toml
│   ├── src/
│   │   ├── main.rs                  # Axum app initialization, routes
│   │   ├── config.rs                # Environment config (DB, Redis, S3, JWT)
│   │   ├── state.rs                 # AppState with PgPool, Redis, S3Client
│   │   ├── error.rs                 # ApiError enum
│   │   ├── auth/
│   │   │   ├── mod.rs               # JWT middleware
│   │   │   └── oauth.rs             # Google/GitHub OAuth handlers
│   │   ├── api/
│   │   │   ├── router.rs            # Route definitions
│   │   │   ├── handlers/
│   │   │   │   ├── auth.rs          # Register, login, OAuth
│   │   │   │   ├── projects.rs      # Project CRUD
│   │   │   │   ├── boreholes.rs     # Borehole data CRUD
│   │   │   │   ├── materials.rs     # Ground material library
│   │   │   │   ├── mesh.rs          # Mesh generation API
│   │   │   │   ├── analysis.rs      # FEM analysis API (create, status, cancel)
│   │   │   │   ├── results.rs       # Results retrieval, presigned URLs
│   │   │   │   ├── settlement.rs    # Settlement prediction
│   │   │   │   ├── monitoring.rs    # Monitoring data CRUD
│   │   │   │   ├── ccm.rs           # Convergence-confinement calculator
│   │   │   │   ├── billing.rs       # Stripe integration
│   │   │   │   └── usage.rs         # Usage tracking
│   │   │   └── ws/
│   │   │       └── analysis_progress.rs  # WebSocket progress streaming
│   │   ├── db/
│   │   │   ├── mod.rs               # Database pool init
│   │   │   └── models.rs            # SQLx structs (User, Project, Analysis, etc.)
│   │   ├── workers/
│   │   │   ├── mesh_worker.rs       # Background mesh generation worker
│   │   │   └── fem_worker.rs        # Background FEM analysis worker
│   │   └── middleware/
│   │       └── plan_limits.rs       # Plan limit enforcement
│   └── migrations/
│       └── 001_initial.sql          # Full database schema
│
├── solver-core/                     # Rust FEM solver (native only)
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── fem.rs                   # FemSystem: assembly, solve, Newton-Raphson
│   │   ├── mumps.rs                 # MUMPS sparse solver bindings
│   │   ├── excavation.rs            # Sequential excavation logic
│   │   ├── element/
│   │   │   ├── mod.rs               # Element trait
│   │   │   ├── tet10.rs             # 10-node tetrahedral element
│   │   │   ├── hex20.rs             # 20-node hexahedral element
│   │   │   ├── beam.rs              # Shotcrete beam element
│   │   │   └── truss.rs             # Rock bolt truss element
│   │   ├── material/
│   │   │   ├── mod.rs               # Material trait
│   │   │   ├── mohr_coulomb.rs      # Mohr-Coulomb elasto-plastic model
│   │   │   └── hoek_brown.rs        # Hoek-Brown model
│   │   ├── ccm.rs                   # Convergence-confinement analytical solutions
│   │   └── results.rs               # HDF5 results I/O
│   └── tests/
│       └── validation/              # FEM validation benchmarks
│           ├── elastic_cylinder.rs
│           ├── kirsch_hole.rs
│           └── strip_footing.rs
│
├── mesher/                          # Rust mesh generator
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── tunnel.rs                # Tunnel-specific mesh generation
│   │   ├── advancing_front.rs       # Advancing-front tetrahedral mesher
│   │   ├── structured.rs            # Structured hex sweep
│   │   ├── delaunay.rs              # Delaunay far-field infill
│   │   ├── quality.rs               # Mesh quality metrics
│   │   └── hdf5_export.rs           # HDF5 mesh export
│   └── tests/
│       └── mesh_tests.rs
│
├── geotechnical-service/            # Python FastAPI service
│   ├── requirements.txt
│   ├── Dockerfile
│   ├── app/
│   │   ├── main.py                  # FastAPI app
│   │   ├── classification.py        # RMR, Q, GSI calculators
│   │   ├── hoek_brown.py            # Hoek-Brown parameter derivation
│   │   └── kriging.py               # Geostatistical interpolation
│   └── tests/
│       └── test_classification.py
│
├── frontend/                        # React + TypeScript
│   ├── package.json
│   ├── vite.config.ts
│   ├── index.html
│   ├── src/
│   │   ├── App.tsx                  # Router, layout
│   │   ├── main.tsx
│   │   ├── stores/
│   │   │   ├── authStore.ts         # Auth state (Zustand)
│   │   │   ├── projectStore.ts      # Project state
│   │   │   └── analysisStore.ts     # Analysis state
│   │   ├── hooks/
│   │   │   ├── useAuth.ts
│   │   │   ├── useProject.ts
│   │   │   └── useAnalysisProgress.ts  # WebSocket hook
│   │   ├── lib/
│   │   │   ├── api.ts               # Axios API client
│   │   │   ├── settlement.ts        # Gaussian trough calculator
│   │   │   └── ccm.ts               # CCM curve calculation
│   │   ├── pages/
│   │   │   ├── Dashboard.tsx        # Project list
│   │   │   ├── ProjectView.tsx      # Main project workspace
│   │   │   ├── Billing.tsx          # Billing/upgrade page
│   │   │   └── Usage.tsx            # Usage dashboard
│   │   ├── components/
│   │   │   ├── Navbar.tsx
│   │   │   ├── AlignmentEditor/
│   │   │   │   ├── Canvas.tsx       # SVG alignment editor
│   │   │   │   ├── HorizontalCurve.tsx
│   │   │   │   └── VerticalProfile.tsx
│   │   │   ├── CrossSection/
│   │   │   │   ├── Templates.tsx    # Cross-section templates
│   │   │   │   └── DimensionEditor.tsx
│   │   │   ├── TunnelViewer/
│   │   │   │   ├── Scene.tsx        # Three.js scene
│   │   │   │   ├── TunnelGeometry.tsx
│   │   │   │   └── BoreholeMarkers.tsx
│   │   │   ├── MeshViewer/
│   │   │   │   └── MeshGeometry.tsx # 3D mesh visualization
│   │   │   ├── GroundModel/
│   │   │   │   ├── BoreholeTable.tsx
│   │   │   │   ├── MaterialLibrary.tsx
│   │   │   │   └── MaterialAssignment.tsx
│   │   │   ├── RockClass/
│   │   │   │   ├── RMRCalculator.tsx
│   │   │   │   ├── QSystemCalculator.tsx
│   │   │   │   └── GSIChart.tsx
│   │   │   ├── Settlement/
│   │   │   │   ├── GaussianTroughForm.tsx
│   │   │   │   └── SettlementPlot.tsx  # D3.js plot
│   │   │   ├── Results/
│   │   │   │   ├── DisplacementViewer.tsx
│   │   │   │   ├── StressViewer.tsx
│   │   │   │   └── ConvergencePlot.tsx
│   │   │   ├── Monitoring/
│   │   │   │   └── TimeSeriesChart.tsx
│   │   │   └── CCM/
│   │   │       └── CCMCalculator.tsx
│   │   └── styles/
│   │       └── globals.css
│   └── public/
│       └── assets/
│
├── docker-compose.yml               # Local dev: Postgres, Redis, MinIO
├── Dockerfile.backend               # Rust backend Docker image
├── Dockerfile.frontend              # Frontend static build
├── Dockerfile.geotech               # Python service Docker image
├── .github/
│   └── workflows/
│       ├── test.yml                 # Run tests on PR
│       ├── deploy.yml               # Deploy to AWS on merge to main
│       └── validation.yml           # FEM validation suite
└── terraform/                       # AWS infrastructure as code
    ├── main.tf
    ├── ecs.tf                       # ECS Fargate for API, spot instances for workers
    ├── rds.tf                       # PostgreSQL with PostGIS
    ├── elasticache.tf               # Redis
    └── s3.tf                        # S3 buckets for meshes/results
```

---

## Solver Validation Suite

The FEM solver must pass these validation benchmarks to ensure numerical accuracy for safety-critical tunnel design.

### 1. Elastic Thick-Walled Cylinder Under Internal Pressure

**Problem**: Cylindrical cavity (radius R = 5m) in infinite elastic medium (E = 10 GPa, ν = 0.25) under internal pressure p_i = 1 MPa. Compute radial displacement at cavity wall.

**Analytical solution (Lamé)**:
```
u_r = p_i · R / E · (1 + ν)

u_r = 1.0 MPa · 5 m / 10,000 MPa · (1 + 0.25) = 6.25 × 10^-4 m = 0.625 mm
```

**FEM setup**: 3D hex mesh (10K elements), radial pressure BC on inner surface, far-field boundary at 10R (displacement BC).

**Expected result**: `u_r(R) = 0.625 mm ± 2%` → **acceptance range: 0.613 - 0.638 mm**

**Performance target**: Solution time < 5 seconds on 8 cores.

### 2. Mohr-Coulomb Plastic Zone Around Circular Tunnel

**Problem**: Circular tunnel (radius R = 5m, depth z = 50m) in Mohr-Coulomb soil (c = 50 kPa, φ = 30°, E = 100 MPa, ν = 0.3, γ = 20 kN/m³, K0 = 0.5). Unsupported (p_i = 0). Compute plastic zone radius R_p.

**Analytical solution** (elastic-perfectly-plastic):
```
p_0 = K0 · γ · z = 0.5 · 20 · 50 = 500 kPa

Plastic zone forms when σ_θ - σ_r > 2c cos φ / (1 - sin φ)

R_p = R · (2(p_0 - p_i) / (c cot φ + p_0 (1 + sin φ)))^[1/(2 sin φ)]

R_p = 5 · (2(500 - 0) / (50 · cot(30°) + 500(1 + sin(30°))))^[1/(2 sin(30°))]
R_p ≈ 8.42 m
```

**FEM setup**: Plane-strain 2D or 3D with symmetry, 50K elements, stress initialization (K0 condition), excavation by element removal.

**Expected result**: `R_p = 8.42 m ± 5%` → **acceptance range: 8.0 - 8.84 m**

**Performance target**: Solution time < 30 seconds for 2D, < 3 minutes for 3D on 16 cores.

### 3. Hoek-Brown Tunnel in Rock Mass

**Problem**: Circular tunnel (R = 5m, depth z = 200m) in rock mass (GSI = 50, m_i = 10, σ_ci = 80 MPa, E = 10 GPa, ν = 0.25, D = 0, γ = 26 kN/m³, K0 = 1.0). Compute radial displacement at tunnel crown.

**Analytical solution** (numerical integration of Hoek-Brown yield criterion):
```
From GSI = 50, D = 0:
  m_b = 10 · exp((50 - 100)/(28 - 14·0)) = 1.835
  s = exp((50 - 100)/(9 - 3·0)) = 0.00407
  a = 0.506

p_0 = 1.0 · 26 · 200 = 5200 kPa = 5.2 MPa

For unsupported tunnel (p_i = 0), ground reaction curve gives:
  u_r(R) ≈ 18.3 mm   (from numerical GRC integration)
```

**FEM setup**: 3D mesh (150K elements), Hoek-Brown material model, sequential excavation (instantaneous).

**Expected result**: `u_r(crown) = 18.3 mm ± 7%` → **acceptance range: 17.0 - 19.6 mm**

**Performance target**: Solution time < 5 minutes on 32 cores, convergence in < 15 Newton-Raphson iterations.

### 4. Sequential Excavation with Support Installation

**Problem**: NATM tunnel (10m diameter, 50m length) excavated in 5 stages (5m advance per stage), top-heading/bench excavation. Install shotcrete (t = 200mm, E = 25 GPa, ν = 0.2) at 5m behind face. Ground: Mohr-Coulomb (c = 100 kPa, φ = 35°, E = 500 MPa, ν = 0.3, γ = 22 kN/m³), depth z = 30m, K0 = 0.6.

**Expected behavior**:
- Crown settlement increases with each stage
- Settlement rate reduces after support installation
- Final crown settlement: **35-45 mm** (empirical range for these conditions)
- Support develops compressive stress: **8-12 MPa**

**FEM setup**: 3D tunnel-specific mesh (200K elements), 5 excavation stages, support activation with 1-stage delay.

**Expected results**:
- Stage 1 (no support): `u_crown = 12-16 mm`
- Stage 5 (with support): `u_crown = 35-45 mm`
- Shotcrete stress: `σ_c = 8-12 MPa (compressive)`

**Performance target**: Total solution time < 15 minutes on 32 cores (3 min/stage average), convergence in < 20 iterations per stage.

### 5. Gaussian Settlement Trough Validation

**Problem**: Shield TBM tunnel (D = 6m, depth z = 20m, volume loss V_L = 1.5%, K = 0.5). Compute max surface settlement and trough width.

**Analytical solution (Peck 1969)**:
```
i = K · z = 0.5 · 20 = 10 m
V_loss = 0.015 · π · (6/2)² = 0.4241 m³/m
S_max = V_loss / (i · √(2π)) = 0.4241 / (10 · √(2π)) = 16.9 mm
```

**Implementation**: TypeScript calculator (client-side, no FEM).

**Expected results**:
- `S_max = 16.9 mm` (exact)
- `S(10m offset) = 10.2 mm` (60% of S_max, at inflection point)
- `S(20m offset) = 2.3 mm` (13% of S_max)

**Validation**: Compare to published case histories (Crossrail, London Tunnels) within ±20%.

---

## Verification Checklist

Before production launch, verify:

### Functional Requirements
- [ ] User can create project, define tunnel alignment (horizontal + vertical curves), cross-section
- [ ] User can import borehole data (CSV/AGS), visualize in 3D
- [ ] User can assign ground materials (Mohr-Coulomb, Hoek-Brown) to layers/zones
- [ ] User can generate tunnel mesh (automatic refinement zones, quality checks)
- [ ] User can define sequential excavation stages with support installation
- [ ] User can run FEM analysis with live progress updates via WebSocket
- [ ] User can visualize FEM results (displacement/stress contours, excavation animation)
- [ ] User can compute settlement predictions (Gaussian trough, compare to FEM)
- [ ] User can run rock mass classification (RMR, Q, GSI) and get support recommendations
- [ ] User can compute convergence-confinement curves (GRC/SCC), find equilibrium point
- [ ] User can import monitoring data, compare predicted vs. measured displacements
- [ ] User can generate PDF report with plots, tables, and analysis summary

### FEM Solver Accuracy
- [ ] Elastic cylinder validation: error < 2%
- [ ] Mohr-Coulomb plastic zone: error < 5%
- [ ] Hoek-Brown tunnel displacement: error < 7%
- [ ] Sequential excavation with support: crown settlement and support stress within expected ranges
- [ ] Gaussian settlement calculator: exact match to analytical formula

### Performance
- [ ] Mesh generation: 165K element tunnel mesh in < 2 minutes
- [ ] FEM analysis: 165K element mesh, 5 stages, completes in < 15 minutes (32 cores)
- [ ] Result download: 500MB HDF5 file downloads via presigned S3 URL in < 1 minute
- [ ] 3D visualization: 500K element mesh renders at > 30 FPS
- [ ] API response time: 95th percentile < 500ms (excluding long-running FEM jobs)

### Scalability
- [ ] System supports 100 concurrent FEM jobs (auto-scaling workers)
- [ ] Database handles 10,000 projects, 50,000 analyses without performance degradation
- [ ] S3 storage scales to 1TB+ of meshes/results

### Security
- [ ] All API endpoints require authentication (JWT)
- [ ] Project access control: owner/org members only
- [ ] SQL injection protection: parameterized queries via SQLx
- [ ] XSS protection: React auto-escapes user input
- [ ] HTTPS enforced in production (TLS 1.3)
- [ ] Secrets stored in AWS Secrets Manager, not env vars

### Billing
- [ ] Free plan enforced: 50K element limit, 5 analyses/month, 2 projects
- [ ] Pro plan upgrade via Stripe checkout works end-to-end
- [ ] Stripe webhooks correctly update user plan in database
- [ ] Usage tracking records FEM core-hours, mesh generation count
- [ ] Customer portal allows plan cancellation, invoice download

### Monitoring
- [ ] Prometheus metrics exported: API latency, FEM job queue depth, worker utilization
- [ ] Grafana dashboards display system health, user activity, cost per analysis
- [ ] Sentry captures errors with stack traces, breadcrumbs
- [ ] CloudWatch logs aggregated and searchable
- [ ] Alerts configured: API error rate > 1%, FEM queue depth > 50, database CPU > 80%

---

## Deployment Architecture

```
                                   ┌─────────────────┐
                                   │   CloudFront    │  CDN for frontend assets
                                   │   (React SPA)   │
                                   └────────┬────────┘
                                            │
                                   ┌────────▼────────┐
                  ┌────────────────┤   ALB (HTTPS)   ├─────────────────┐
                  │                └─────────────────┘                  │
                  │                                                     │
        ┌─────────▼──────────┐                            ┌────────────▼──────────┐
        │  ECS Fargate       │                            │  ECS Fargate          │
        │  Rust API          │                            │  Python Geotech Svc   │
        │  (4 tasks, 2 vCPU) │                            │  (2 tasks, 1 vCPU)    │
        └─────────┬──────────┘                            └────────────┬──────────┘
                  │                                                     │
                  │                                                     │
        ┌─────────▼──────────────────────────────────────────────────┬─┘
        │                                                             │
        │  ┌─────────────────┐     ┌─────────────────┐     ┌────────▼────────┐
        │  │  RDS PostgreSQL │     │  ElastiCache    │     │  S3 Buckets     │
        │  │  (PostGIS ext)  │     │  Redis 7        │     │  - Meshes (HDF5)│
        │  │  db.r6g.xlarge  │     │  cache.r6g.large│     │  - Results      │
        │  └─────────────────┘     └─────────────────┘     │  - Uploads      │
        │                                                    └─────────────────┘
        │
        │  ┌──────────────────────────────────────────────────────────┐
        └─▶│  EC2 Auto Scaling Group — FEM Workers                    │
           │  c7g.8xlarge spot instances (32 vCPU, 64GB RAM)          │
           │  Min: 0, Max: 20, Target: queue depth / 5                │
           │  AMI: Amazon Linux 2023 + MUMPS libraries                │
           │  Pulls jobs from Redis, uploads results to S3            │
           └──────────────────────────────────────────────────────────┘

Monitoring:
  - Prometheus exporters on API and workers → Grafana Cloud
  - CloudWatch Logs: all services
  - Sentry: error tracking
  - AWS Cost Explorer: daily cost breakdown

Backup:
  - RDS: automated daily snapshots, 7-day retention
  - S3: versioning enabled, lifecycle policy to Glacier after 90 days
```

**Estimated monthly cost** (100 active users, 500 analyses/month):
- RDS PostgreSQL (db.r6g.xlarge): $300
- ElastiCache Redis (cache.r6g.large): $150
- ECS Fargate (API + Python service): $200
- EC2 spot instances (FEM workers, avg 10 instances * 50% duty cycle): $400
- S3 storage (1TB): $25
- Data transfer: $50
- **Total: ~$1,125/month**

---

## Post-MVP Roadmap

### v1.1 — TBM and Segmental Lining (Weeks 13-16)
- TBM face pressure calculator (EPB and slurry methods)
- Segmental lining structural analysis (beam-spring and 3D shell models)
- Joint modeling (flat joints, convex-convex, rotational stiffness)
- Loading cases: thrust jack loads, grouting pressure, long-term creep
- Annular grout injection simulation

### v1.2 — Advanced Constitutive Models (Weeks 17-20)
- Hardening Soil model for soft ground tunnels
- Time-dependent shotcrete model (J2, early-age stiffness gain, creep)
- Cyclic degradation for earthquake loading
- Swelling rock models (anhydrite, clay shales)
- Temperature effects on ground and support

### v1.3 — 3D Settlement and Building Damage (Weeks 21-24)
- Full 3D FEM settlement analysis (transverse + longitudinal profiles)
- Building-soil interaction: equivalent beam model and 3D building-on-foundation
- Building damage assessment (Burland & Wroth tensile strain method)
- Damage category classification (negligible to very severe)
- Utility impact: angular distortion, pipeline vulnerability

### v1.4 — Monitoring Integration and Back-Analysis (Weeks 25-28)
- Real-time monitoring dashboard (IoT sensor integration via MQTT)
- Automated comparison: predicted vs. measured with trend analysis
- Alarm system: trigger level exceedances, SMS/email alerts
- Back-analysis: calibrate FEM parameters from monitoring data (inverse analysis)
- Observational method workflow: predict → monitor → update → re-predict

### v1.5 — Multi-Tunnel and Complex Geometries (Weeks 29-32)
- Twin parallel tunnels (interaction effects, pillar stability)
- Cross-passages and junction caverns
- Ventilation shafts and access adits
- Portal zone modeling with slope and portal structure
- Variable cross-sections for enlargement chambers

### v1.6 — Discontinuum Analysis (Weeks 33-36)
- Jointed rock mass: discrete fracture network (DFN) generation
- Block theory: kinematic analysis for wedge/toppling failure
- Unwedge-equivalent: identify unstable blocks around tunnel
- Distinct Element Method (DEM) for blocky rock (integrate with FEM)
- Stereonet analysis for joint orientations

### v1.7 — Enterprise Features (Weeks 37-40)
- Multi-project portfolio management for contractors
- Role-based access control (viewer, editor, admin)
- BIM integration: IFC tunnel extensions (IFC Shield TBM, IFC Tunnel)
- Custom constitutive model upload (user-provided Rust/Python plugins)
- On-premise deployment option for sensitive infrastructure projects
- Single sign-on (SSO) via SAML for enterprise clients

---

**Total Implementation Plan: 1494 lines**