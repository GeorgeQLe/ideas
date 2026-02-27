# 38. StructAI — AI Structural Engineering Analysis & Design Platform

## Implementation Plan

**MVP Scope:** Browser-based 3D structural modeler with grid-based beam/column/brace placement rendered via Three.js, GPU-accelerated finite element analysis solver in Rust with CUDA for linear static, P-delta, and modal analysis supporting models up to 5,000 nodes, automatic steel design checks per AISC 360-22 (flexure, shear, axial, combined forces) with demand-to-capacity ratios displayed on 3D model using color gradients, load case creation (dead, live, user-defined) with automatic ASCE 7 LRFD combination generation, interactive 3D results visualization showing deformed shapes at configurable scale and member force diagrams (axial, shear, moment), cross-section database search from AISC shape library (W, HSS, L, C sections) with parametric property calculation, PDF calculation report generator with member-by-member design checks and governing load combination identification, Stripe billing with three tiers (Free / Pro $149/mo / Team $399/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| FEA Solver | Rust + CUDA (rust-cuda) | Custom GPU-accelerated sparse matrix assembly and factorization |
| Linear Algebra | cuSOLVER (GPU), Eigen (CPU) | GPU sparse direct solver for large models, CPU fallback |
| Code Checking | Rust | AISC 360-22, ACI 318-19, ASCE 7-22 design engines with configurable editions |
| AI Optimization | Python 3.12 (FastAPI) | Genetic algorithm + particle swarm, GPU-parallel fitness evaluation |
| Database | PostgreSQL 16 | Projects, models, users, analysis jobs, design results |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and Microsoft OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | IFC files, analysis results (binary mesh data), PDF reports |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| 3D Rendering | Three.js with WebGL | Custom structural element renderers, instanced rendering for large models |
| Real-time | WebSocket (Axum) | Live analysis progress, convergence monitoring |
| Job Queue | Redis 7 + Tokio tasks | FEA job management, GPU resource allocation |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| Search | Meilisearch | Cross-section database search, code clause lookup |
| Compute | Kubernetes (EKS) | GPU node pools (NVIDIA A100/T4), Karpenter auto-scaling |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, error tracking |
| CI/CD | GitHub Actions | Rust tests, Docker image build, EKS deployment |

### Key Architecture Decisions

1. **GPU-accelerated FEA solver with CUDA threshold at 5,000 nodes**: Models with ≤5,000 nodes (covers 95%+ of typical building structures: 5-20 story buildings, standard lateral systems) run on GPU cluster nodes with NVIDIA T4 GPUs, completing linear static analysis in 30–90 seconds. Larger models (high-rise, complex geometries) queue for A100 GPU nodes with 40GB VRAM supporting 50K+ nodes. The GPU threshold is configurable per plan tier to manage compute costs.

2. **Custom Rust+CUDA solver rather than wrapping commercial kernels**: Building a domain-specific FEA solver in Rust with CUDA allows us to optimize for structural analysis patterns (sparse, banded stiffness matrices from frame and shell elements), implement building-code-specific convergence algorithms (P-delta iteration with AISC direct analysis requirements), and deploy to cloud GPU instances without license restrictions. This gives 10x performance advantage over CPU-based commercial solvers while maintaining full numerical validation.

3. **Three.js structural visualization with instanced rendering**: Three.js enables high-quality 3D rendering in the browser with WebGL acceleration. Instanced rendering allows efficient display of 10K+ beam elements with identical geometries but different transforms and colors. R-tree spatial indexing via `rbush` enables O(log n) picking and selection. Custom shaders render stress contours directly on mesh geometry with GPU interpolation.

4. **PostgreSQL JSONB for flexible model schema**: Structural models have highly variable schemas (different element types, material properties, load definitions) that evolve as we add features. JSONB columns allow flexible storage of schematic data, load parameters, and analysis settings while maintaining queryability for project lists and metadata. Large binary results (nodal displacements, element forces) live in S3.

5. **Separate Python service for AI optimization**: The genetic algorithm optimization engine requires different libraries (NumPy, PyTorch for GPU parallelization) and update cadence than the core FEA solver. A dedicated Python FastAPI service handles optimization runs, interfaces with the Rust solver via HTTP for fitness evaluation, and writes optimized designs back to PostgreSQL. This separation allows independent scaling and algorithm experimentation.

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";  -- For fuzzy text search on cross-sections

-- Users
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email TEXT UNIQUE NOT NULL,
    password_hash TEXT,  -- NULL for OAuth-only users
    name TEXT NOT NULL,
    avatar_url TEXT,
    auth_provider TEXT DEFAULT 'email',  -- email | google | microsoft
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
    role TEXT NOT NULL DEFAULT 'member',  -- owner | admin | engineer | reviewer | viewer
    joined_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(org_id, user_id)
);
CREATE INDEX org_members_org_idx ON org_members(org_id);
CREATE INDEX org_members_user_idx ON org_members(user_id);

-- Projects
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    building_type TEXT DEFAULT 'general',  -- general | commercial | residential | industrial
    code_edition JSONB NOT NULL DEFAULT '{"steel": "AISC360-22", "concrete": "ACI318-19", "seismic": "ASCE7-22"}',
    units TEXT NOT NULL DEFAULT 'imperial',  -- imperial | metric
    status TEXT NOT NULL DEFAULT 'active',  -- active | archived
    is_public BOOLEAN NOT NULL DEFAULT false,
    thumbnail_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);
CREATE INDEX projects_org_idx ON projects(org_id);
CREATE INDEX projects_updated_idx ON projects(updated_at DESC);

-- Structural Models
CREATE TABLE structural_models (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    version INTEGER NOT NULL DEFAULT 1,
    grid_system JSONB NOT NULL DEFAULT '{"x_spacings": [20], "y_spacings": [20], "z_levels": [0, 12]}',
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(project_id, version)
);
CREATE INDEX models_project_idx ON structural_models(project_id);

-- Nodes
CREATE TABLE nodes (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    model_id UUID NOT NULL REFERENCES structural_models(id) ON DELETE CASCADE,
    x REAL NOT NULL,
    y REAL NOT NULL,
    z REAL NOT NULL,
    support_conditions JSONB DEFAULT NULL,  -- {Fx: true, Fy: true, Fz: true, Mx: false, My: false, Mz: false}
    mass_source JSONB DEFAULT NULL,  -- {value: 10.5, direction: "z"}
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX nodes_model_idx ON nodes(model_id);
CREATE INDEX nodes_coords_idx ON nodes(model_id, x, y, z);

-- Frame Elements (beams, columns, braces)
CREATE TABLE frame_elements (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    model_id UUID NOT NULL REFERENCES structural_models(id) ON DELETE CASCADE,
    node_i_id UUID NOT NULL REFERENCES nodes(id) ON DELETE CASCADE,
    node_j_id UUID NOT NULL REFERENCES nodes(id) ON DELETE CASCADE,
    element_type TEXT NOT NULL,  -- beam | column | brace
    section_id UUID NOT NULL REFERENCES cross_sections(id),
    material_id UUID NOT NULL REFERENCES materials(id),
    releases JSONB DEFAULT NULL,  -- {i_end: {Mx: true, My: false, Mz: false}, j_end: {...}}
    offsets JSONB DEFAULT NULL,  -- {i_end: {dx: 0, dy: 0, dz: 0}, j_end: {...}}
    design_group TEXT DEFAULT NULL,
    unbraced_length_override REAL DEFAULT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX frame_elements_model_idx ON frame_elements(model_id);
CREATE INDEX frame_elements_nodes_idx ON frame_elements(node_i_id, node_j_id);

-- Area Elements (walls, slabs, shells)
CREATE TABLE area_elements (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    model_id UUID NOT NULL REFERENCES structural_models(id) ON DELETE CASCADE,
    node_ids UUID[] NOT NULL,  -- 3 or 4 nodes for tri or quad
    element_type TEXT NOT NULL,  -- wall | slab | shell
    thickness REAL NOT NULL,
    material_id UUID NOT NULL REFERENCES materials(id),
    mesh_size REAL DEFAULT NULL,
    local_axes JSONB DEFAULT NULL,
    diaphragm_id UUID REFERENCES diaphragms(id) ON DELETE SET NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX area_elements_model_idx ON area_elements(model_id);

-- Diaphragms
CREATE TABLE diaphragms (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    model_id UUID NOT NULL REFERENCES structural_models(id) ON DELETE CASCADE,
    story_level REAL NOT NULL,
    type TEXT NOT NULL DEFAULT 'rigid',  -- rigid | semi_rigid | flexible
    mass_center JSONB DEFAULT NULL,  -- {x: 50, y: 50}
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX diaphragms_model_idx ON diaphragms(model_id);

-- Cross Sections
CREATE TABLE cross_sections (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID REFERENCES projects(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    shape_type TEXT NOT NULL,  -- W | HSS | L | C | custom
    properties JSONB NOT NULL,  -- {A, Ix, Iy, Sx, Sy, Zx, Zy, rx, ry, J, Cw}
    dimensions JSONB NOT NULL,  -- {d, bf, tf, tw, ...} (shape-specific)
    is_builtin BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX sections_project_idx ON cross_sections(project_id);
CREATE INDEX sections_name_trgm_idx ON cross_sections USING gin(name gin_trgm_ops);
CREATE INDEX sections_builtin_idx ON cross_sections(is_builtin) WHERE is_builtin = true;

-- Materials
CREATE TABLE materials (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID REFERENCES projects(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    type TEXT NOT NULL,  -- steel | concrete | wood | other
    properties JSONB NOT NULL,  -- {E, Fy, Fu, fc, density, poisson, alpha}
    code_grade TEXT DEFAULT NULL,  -- "A992" | "4000psi" | etc
    is_builtin BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX materials_project_idx ON materials(project_id);
CREATE INDEX materials_builtin_idx ON materials(is_builtin) WHERE is_builtin = true;

-- Load Cases
CREATE TABLE load_cases (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    type TEXT NOT NULL,  -- dead | live | wind | seismic | snow | user
    auto_generated BOOLEAN NOT NULL DEFAULT false,
    generation_params JSONB DEFAULT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX load_cases_project_idx ON load_cases(project_id);

-- Loads
CREATE TABLE loads (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    load_case_id UUID NOT NULL REFERENCES load_cases(id) ON DELETE CASCADE,
    target_type TEXT NOT NULL,  -- node | frame_element | area_element
    target_id UUID NOT NULL,
    load_type TEXT NOT NULL,  -- force | moment | distributed | pressure | temperature
    values JSONB NOT NULL,  -- {Fx, Fy, Fz, Mx, My, Mz, w, direction}
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX loads_case_idx ON loads(load_case_id);
CREATE INDEX loads_target_idx ON loads(target_type, target_id);

-- Load Combinations
CREATE TABLE load_combinations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    type TEXT NOT NULL,  -- LRFD | ASD
    factors JSONB NOT NULL,  -- [{load_case_id, factor}]
    auto_generated BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX combinations_project_idx ON load_combinations(project_id);

-- Analysis Jobs
CREATE TABLE analysis_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    model_id UUID NOT NULL REFERENCES structural_models(id) ON DELETE CASCADE,
    analysis_type TEXT NOT NULL,  -- linear | pdelta | modal | response_spectrum | time_history | pushover | buckling
    status TEXT NOT NULL DEFAULT 'queued',  -- queued | running | completed | failed | cancelled
    gpu_node_id TEXT,
    parameters JSONB NOT NULL DEFAULT '{}',
    node_count INTEGER NOT NULL DEFAULT 0,
    dof_count INTEGER NOT NULL DEFAULT 0,
    results_url TEXT,  -- S3 URL
    results_summary JSONB,  -- Quick-access summary stats
    error_message TEXT,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    wall_time_seconds REAL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX jobs_project_idx ON analysis_jobs(project_id);
CREATE INDEX jobs_user_idx ON analysis_jobs(user_id);
CREATE INDEX jobs_status_idx ON analysis_jobs(status);
CREATE INDEX jobs_created_idx ON analysis_jobs(created_at DESC);

-- Design Results
CREATE TABLE design_results (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    analysis_job_id UUID NOT NULL REFERENCES analysis_jobs(id) ON DELETE CASCADE,
    element_id UUID NOT NULL,  -- References frame_elements or area_elements
    element_type TEXT NOT NULL,  -- frame | area
    code_edition TEXT NOT NULL,
    governing_combination_id UUID REFERENCES load_combinations(id),
    dcr REAL NOT NULL,  -- Demand-to-capacity ratio
    check_type TEXT NOT NULL,  -- flexure | shear | axial | combined | drift | punching_shear
    detailed_checks JSONB NOT NULL,  -- [{clause, description, demand, capacity, dcr}]
    status TEXT NOT NULL,  -- pass | marginal | fail
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX design_results_project_idx ON design_results(project_id);
CREATE INDEX design_results_job_idx ON design_results(analysis_job_id);
CREATE INDEX design_results_element_idx ON design_results(element_id);

-- Optimization Runs
CREATE TABLE optimization_runs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    analysis_job_id UUID REFERENCES analysis_jobs(id),
    objective TEXT NOT NULL,  -- min_weight | min_cost | min_carbon
    status TEXT NOT NULL DEFAULT 'running',  -- running | completed | failed
    constraints JSONB NOT NULL DEFAULT '{}',
    generations_completed INTEGER DEFAULT 0,
    best_fitness REAL DEFAULT NULL,
    convergence_history JSONB DEFAULT '[]',
    optimized_sections JSONB DEFAULT NULL,  -- [{element_id, original_section, optimized_section}]
    weight_reduction_pct REAL DEFAULT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at TIMESTAMPTZ
);
CREATE INDEX optimization_runs_project_idx ON optimization_runs(project_id);
CREATE INDEX optimization_runs_status_idx ON optimization_runs(status);

-- Comments (for collaboration)
CREATE TABLE comments (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    target_type TEXT NOT NULL,  -- element | result | general
    target_id UUID,
    content TEXT NOT NULL,
    position JSONB,  -- {x, y, z} for 3D anchored comments
    resolved BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX comments_project_idx ON comments(project_id);
CREATE INDEX comments_target_idx ON comments(target_type, target_id);
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
    pub building_type: String,
    pub code_edition: serde_json::Value,
    pub units: String,
    pub status: String,
    pub is_public: bool,
    pub thumbnail_url: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct StructuralModel {
    pub id: Uuid,
    pub project_id: Uuid,
    pub version: i32,
    pub grid_system: serde_json::Value,
    pub is_active: bool,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Node {
    pub id: Uuid,
    pub model_id: Uuid,
    pub x: f32,
    pub y: f32,
    pub z: f32,
    pub support_conditions: Option<serde_json::Value>,
    pub mass_source: Option<serde_json::Value>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct FrameElement {
    pub id: Uuid,
    pub model_id: Uuid,
    pub node_i_id: Uuid,
    pub node_j_id: Uuid,
    pub element_type: String,
    pub section_id: Uuid,
    pub material_id: Uuid,
    pub releases: Option<serde_json::Value>,
    pub offsets: Option<serde_json::Value>,
    pub design_group: Option<String>,
    pub unbraced_length_override: Option<f32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct CrossSection {
    pub id: Uuid,
    pub project_id: Option<Uuid>,
    pub name: String,
    pub shape_type: String,
    pub properties: serde_json::Value,
    pub dimensions: serde_json::Value,
    pub is_builtin: bool,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Material {
    pub id: Uuid,
    pub project_id: Option<Uuid>,
    pub name: String,
    pub r#type: String,
    pub properties: serde_json::Value,
    pub code_grade: Option<String>,
    pub is_builtin: bool,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct LoadCase {
    pub id: Uuid,
    pub project_id: Uuid,
    pub name: String,
    pub r#type: String,
    pub auto_generated: bool,
    pub generation_params: Option<serde_json::Value>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct LoadCombination {
    pub id: Uuid,
    pub project_id: Uuid,
    pub name: String,
    pub r#type: String,
    pub factors: serde_json::Value,
    pub auto_generated: bool,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct AnalysisJob {
    pub id: Uuid,
    pub project_id: Uuid,
    pub user_id: Uuid,
    pub model_id: Uuid,
    pub analysis_type: String,
    pub status: String,
    pub gpu_node_id: Option<String>,
    pub parameters: serde_json::Value,
    pub node_count: i32,
    pub dof_count: i32,
    pub results_url: Option<String>,
    pub results_summary: Option<serde_json::Value>,
    pub error_message: Option<String>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub wall_time_seconds: Option<f32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct DesignResult {
    pub id: Uuid,
    pub project_id: Uuid,
    pub analysis_job_id: Uuid,
    pub element_id: Uuid,
    pub element_type: String,
    pub code_edition: String,
    pub governing_combination_id: Option<Uuid>,
    pub dcr: f32,
    pub check_type: String,
    pub detailed_checks: serde_json::Value,
    pub status: String,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct AnalysisParams {
    pub analysis_type: AnalysisType,
    pub load_combination_ids: Vec<Uuid>,
    pub pdelta: Option<PdeltaParams>,
    pub modal: Option<ModalParams>,
    pub nonlinear: Option<NonlinearParams>,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum AnalysisType {
    Linear,
    Pdelta,
    Modal,
    ResponseSpectrum,
    TimeHistory,
    Pushover,
    Buckling,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct PdeltaParams {
    pub max_iterations: u32,  // Default 10
    pub convergence_tol: f64,  // Default 0.001
}

#[derive(Debug, Deserialize, Serialize)]
pub struct ModalParams {
    pub num_modes: u32,  // Number of modes to extract
    pub freq_cutoff: Option<f64>,  // Maximum frequency (Hz)
    pub eigensolver: String,  // lanczos | subspace
}

#[derive(Debug, Deserialize, Serialize)]
pub struct NonlinearParams {
    pub max_iterations: u32,
    pub convergence_tol: f64,
    pub load_steps: u32,
}
```

---

## FEA Solver Architecture Deep-Dive

### Governing Equations and Discretization

StructAI's core solver implements the **direct stiffness method** for finite element analysis. For a structure with `n` degrees of freedom, the fundamental equilibrium equation is:

```
[K]{u} = {F}

Where:
- [K] (n×n): Global stiffness matrix (sparse, symmetric, positive-definite)
- {u} (n×1): Nodal displacement vector
- {F} (n×1): Applied force vector
```

**Frame Element (Beam-Column)**: 3D beam element with 12 DOF (6 per node: ux, uy, uz, θx, θy, θz). Local element stiffness matrix accounts for axial, bending (Euler-Bernoulli), shear, and torsional rigidity:

```
k_local = [
    EA/L      symmetric about diagonal
    12EI_z/L³  (bending about local z)
    12EI_y/L³  (bending about local y)
    GJ/L       (torsion)
    ...
]

Transform to global coordinates: K_global = [T]ᵀ [k_local] [T]
```

**P-delta Analysis**: Geometric nonlinearity from axial loads causing additional moments in deflected configuration. Implemented via iterative process:

```
1. Solve linear system: [K₀]{u₀} = {F}
2. Extract axial forces P from {u₀}
3. Build geometric stiffness [Kₐ] based on P and deformed shape
4. Update: [K] = [K₀] + [Kₐ]
5. Resolve: [K]{u₁} = {F}
6. Repeat until ||u_{i+1} - u_i|| < tolerance
```

**Modal Analysis**: Eigenvalue problem to find natural frequencies ω and mode shapes φ:

```
[K]{φ} = ω² [M]{φ}

Where:
- [M]: Mass matrix (lumped or consistent)
- ω: Circular frequency (rad/s)
- φ: Mode shape vector

Solved using Lanczos algorithm for sparse symmetric matrices
```

**Matrix Assembly and Solve Pipeline**:
1. Element stiffness generation (parallel across elements)
2. Global stiffness assembly using COO format (Coordinate List)
3. Conversion to CSR format (Compressed Sparse Row) for solve
4. Factorization: Cholesky for symmetric positive-definite, LU for P-delta iterations
5. Forward/back substitution for displacement solution
6. Element force recovery via {f_elem} = [k_elem]([T]{u_global})

### GPU Acceleration Strategy

**CPU vs GPU Decision Tree**:
```
Model assembled → DOF count extracted
    │
    ├── ≤1,500 DOF → CPU solve (Eigen sparse Cholesky)
    │   ├── Instant (<5s), no GPU queue wait
    │   └── Covers: 3-5 story buildings, simple frames
    │
    └── >1,500 DOF → GPU solve (cuSPARSE + cuSOLVER)
        ├── Job queued via Redis
        ├── GPU node allocation (T4 for ≤30K DOF, A100 for larger)
        ├── CUDA kernel for parallel stiffness assembly
        ├── cuSOLVER Cholesky factorization
        └── Results written to S3

Threshold chosen because:
- 1,500 DOF ≈ 250 frame elements (typical 8-12 story building)
- CPU solve <5s, GPU overhead (queue + transfer) ~10-20s
- Above 1,500 DOF: GPU speedup 5-15x over CPU
```

**CUDA Kernel Optimization**:
- Element stiffness matrix generation parallelized (1 thread per element)
- Atomic adds for global matrix assembly (sparse COO format)
- cuSPARSE COO-to-CSR conversion on GPU
- cuSOLVER csrlsvchol for sparse Cholesky solve
- Result extraction and force recovery vectorized

---

## Architecture Deep-Dives

### 1. Analysis API Handler (Rust/Axum)

The primary endpoint receives an analysis request, validates the model, decides between CPU and GPU execution, and enqueues GPU jobs with Redis while streaming progress via WebSocket.

```rust
// src/api/handlers/analysis.rs

use axum::{
    extract::{Path, State, Json},
    http::StatusCode,
    response::IntoResponse,
};
use uuid::Uuid;
use crate::{
    db::models::{AnalysisJob, AnalysisParams, AnalysisType},
    solver::model_extractor::extract_solver_model,
    state::AppState,
    auth::Claims,
    error::ApiError,
};

#[derive(serde::Deserialize)]
pub struct CreateAnalysisRequest {
    pub analysis_type: AnalysisType,
    pub load_combination_ids: Vec<Uuid>,
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
        "SELECT * FROM projects WHERE id = $1 AND (owner_id = $2 OR org_id IN (
            SELECT org_id FROM org_members WHERE user_id = $2
        ))",
        project_id,
        claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Project not found"))?;

    // 2. Get active structural model
    let model = sqlx::query_as!(
        crate::db::models::StructuralModel,
        "SELECT * FROM structural_models WHERE project_id = $1 AND is_active = true",
        project_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("No active structural model found"))?;

    // 3. Extract full model data (nodes, elements, materials, sections)
    let solver_model = extract_solver_model(&state.db, model.id).await?;
    let node_count = solver_model.nodes.len();
    let dof_count = node_count * 6;  // 6 DOF per node

    // 4. Check plan limits
    let user = sqlx::query_as!(
        crate::db::models::User,
        "SELECT * FROM users WHERE id = $1",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    if user.plan == "free" && node_count > 100 {
        return Err(ApiError::PlanLimit(
            "Free plan supports models up to 100 nodes (600 DOF). Upgrade to Pro for 5,000 nodes."
        ));
    }

    if user.plan == "pro" && node_count > 5000 {
        return Err(ApiError::PlanLimit(
            "Pro plan supports models up to 5,000 nodes. Upgrade to Team for 50,000 nodes."
        ));
    }

    // 5. Validate load combinations exist
    let combination_count = sqlx::query_scalar!(
        "SELECT COUNT(*) FROM load_combinations WHERE id = ANY($1) AND project_id = $2",
        &req.load_combination_ids,
        project_id
    )
    .fetch_one(&state.db)
    .await?;

    if combination_count != Some(req.load_combination_ids.len() as i64) {
        return Err(ApiError::BadRequest("Invalid load combination IDs"));
    }

    // 6. Determine execution mode
    let execution_mode = if dof_count <= 1500 {
        "cpu"
    } else {
        "gpu"
    };

    // 7. Create analysis job record
    let job = sqlx::query_as!(
        AnalysisJob,
        r#"INSERT INTO analysis_jobs
            (project_id, user_id, model_id, analysis_type, status,
             parameters, node_count, dof_count)
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8)
        RETURNING *"#,
        project_id,
        claims.user_id,
        model.id,
        serde_json::to_string(&req.analysis_type)?,
        if execution_mode == "cpu" { "running" } else { "queued" },
        serde_json::json!({
            "load_combination_ids": req.load_combination_ids,
            "analysis_params": req.parameters,
            "execution_mode": execution_mode
        }),
        node_count as i32,
        dof_count as i32,
    )
    .fetch_one(&state.db)
    .await?;

    // 8. For GPU execution, enqueue job; for CPU, spawn immediate task
    if execution_mode == "gpu" {
        // Publish to Redis job queue with priority
        let priority = match user.plan.as_str() {
            "team" => 10,
            "pro" => 5,
            _ => 0,
        };

        state.redis
            .zadd("analysis:jobs", job.id.to_string(), priority)
            .await?;
    } else {
        // Spawn CPU analysis task
        let state_clone = state.clone();
        tokio::spawn(async move {
            if let Err(e) = crate::solver::cpu::run_analysis(
                state_clone.db.clone(),
                state_clone.broadcaster.clone(),
                job.id
            ).await {
                tracing::error!("CPU analysis failed for job {}: {}", job.id, e);
            }
        });
    }

    Ok((StatusCode::CREATED, Json(job)))
}

pub async fn get_analysis_status(
    State(state): State<AppState>,
    claims: Claims,
    Path((project_id, job_id)): Path<(Uuid, Uuid)>,
) -> Result<Json<AnalysisJob>, ApiError> {
    let job = sqlx::query_as!(
        AnalysisJob,
        "SELECT j.* FROM analysis_jobs j
         JOIN projects p ON j.project_id = p.id
         WHERE j.id = $1 AND j.project_id = $2
         AND (p.owner_id = $3 OR p.org_id IN (
             SELECT org_id FROM org_members WHERE user_id = $3
         ))",
        job_id,
        project_id,
        claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Analysis job not found"))?;

    Ok(Json(job))
}

pub async fn get_analysis_results(
    State(state): State<AppState>,
    claims: Claims,
    Path((project_id, job_id)): Path<(Uuid, Uuid)>,
) -> Result<impl IntoResponse, ApiError> {
    let job = sqlx::query_as!(
        AnalysisJob,
        "SELECT j.* FROM analysis_jobs j
         JOIN projects p ON j.project_id = p.id
         WHERE j.id = $1 AND j.project_id = $2 AND j.status = 'completed'
         AND (p.owner_id = $3 OR p.org_id IN (
             SELECT org_id FROM org_members WHERE user_id = $3
         ))",
        job_id,
        project_id,
        claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Analysis job not found or not completed"))?;

    let results_url = job.results_url.ok_or(ApiError::NotFound("Results not available"))?;

    // Generate presigned S3 URL for download
    let presigned_url = state.s3.generate_presigned_url(&results_url, 3600).await?;

    Ok(Json(serde_json::json!({
        "job_id": job.id,
        "status": job.status,
        "results_url": presigned_url,
        "summary": job.results_summary,
        "wall_time_seconds": job.wall_time_seconds
    })))
}
```

### 2. FEA Solver Core (Rust with CUDA)

The core solver that builds stiffness matrices and solves the structural system. This code compiles for both CPU (Eigen) and GPU (CUDA) backends.

```rust
// solver-core/src/frame_element.rs

use nalgebra::{Matrix6, Vector6, Matrix3, Vector3};

pub struct FrameElement {
    pub id: String,
    pub node_i: usize,  // Global node index
    pub node_j: usize,
    pub section: CrossSection,
    pub material: Material,
    pub releases: Option<EndReleases>,
    pub offsets: Option<EndOffsets>,
}

#[derive(Clone)]
pub struct CrossSection {
    pub area: f64,          // A (in²)
    pub inertia_y: f64,     // Iy (in⁴)
    pub inertia_z: f64,     // Iz (in⁴)
    pub torsion_const: f64, // J (in⁴)
}

#[derive(Clone)]
pub struct Material {
    pub elastic_modulus: f64,  // E (ksi)
    pub shear_modulus: f64,    // G (ksi)
    pub yield_stress: f64,     // Fy (ksi)
    pub density: f64,          // (lb/in³)
}

pub struct EndReleases {
    pub i_end: [bool; 6],  // [Fx, Fy, Fz, Mx, My, Mz]
    pub j_end: [bool; 6],
}

pub struct EndOffsets {
    pub i_end: Vector3<f64>,  // [dx, dy, dz]
    pub j_end: Vector3<f64>,
}

impl FrameElement {
    /// Generate local 12×12 stiffness matrix for 3D frame element
    pub fn local_stiffness_matrix(&self, length: f64) -> Matrix12x12 {
        let e = self.material.elastic_modulus;
        let g = self.material.shear_modulus;
        let a = self.section.area;
        let iy = self.section.inertia_y;
        let iz = self.section.inertia_z;
        let j = self.section.torsion_const;

        let l = length;
        let l2 = l * l;
        let l3 = l2 * l;

        let mut k = Matrix12x12::zeros();

        // Axial stiffness (DOF 0, 6)
        k[(0, 0)] = e * a / l;
        k[(0, 6)] = -e * a / l;
        k[(6, 0)] = -e * a / l;
        k[(6, 6)] = e * a / l;

        // Bending about local z-axis (DOF 1, 5, 7, 11)
        k[(1, 1)] = 12.0 * e * iz / l3;
        k[(1, 5)] = 6.0 * e * iz / l2;
        k[(1, 7)] = -12.0 * e * iz / l3;
        k[(1, 11)] = 6.0 * e * iz / l2;

        k[(5, 1)] = 6.0 * e * iz / l2;
        k[(5, 5)] = 4.0 * e * iz / l;
        k[(5, 7)] = -6.0 * e * iz / l2;
        k[(5, 11)] = 2.0 * e * iz / l;

        k[(7, 1)] = -12.0 * e * iz / l3;
        k[(7, 5)] = -6.0 * e * iz / l2;
        k[(7, 7)] = 12.0 * e * iz / l3;
        k[(7, 11)] = -6.0 * e * iz / l2;

        k[(11, 1)] = 6.0 * e * iz / l2;
        k[(11, 5)] = 2.0 * e * iz / l;
        k[(11, 7)] = -6.0 * e * iz / l2;
        k[(11, 11)] = 4.0 * e * iz / l;

        // Bending about local y-axis (DOF 2, 4, 8, 10)
        k[(2, 2)] = 12.0 * e * iy / l3;
        k[(2, 4)] = -6.0 * e * iy / l2;
        k[(2, 8)] = -12.0 * e * iy / l3;
        k[(2, 10)] = -6.0 * e * iy / l2;

        k[(4, 2)] = -6.0 * e * iy / l2;
        k[(4, 4)] = 4.0 * e * iy / l;
        k[(4, 8)] = 6.0 * e * iy / l2;
        k[(4, 10)] = 2.0 * e * iy / l;

        k[(8, 2)] = -12.0 * e * iy / l3;
        k[(8, 4)] = 6.0 * e * iy / l2;
        k[(8, 8)] = 12.0 * e * iy / l3;
        k[(8, 10)] = 6.0 * e * iy / l2;

        k[(10, 2)] = -6.0 * e * iy / l2;
        k[(10, 4)] = 2.0 * e * iy / l;
        k[(10, 8)] = 6.0 * e * iy / l2;
        k[(10, 10)] = 4.0 * e * iy / l;

        // Torsion (DOF 3, 9)
        k[(3, 3)] = g * j / l;
        k[(3, 9)] = -g * j / l;
        k[(9, 3)] = -g * j / l;
        k[(9, 9)] = g * j / l;

        k
    }

    /// Transform local stiffness to global coordinate system
    pub fn global_stiffness_matrix(
        &self,
        node_i_coords: &Vector3<f64>,
        node_j_coords: &Vector3<f64>,
    ) -> Matrix12x12 {
        let dx = node_j_coords.x - node_i_coords.x;
        let dy = node_j_coords.y - node_i_coords.y;
        let dz = node_j_coords.z - node_i_coords.z;
        let length = (dx * dx + dy * dy + dz * dz).sqrt();

        let k_local = self.local_stiffness_matrix(length);

        // Build transformation matrix (local to global rotation)
        let t = self.transformation_matrix(node_i_coords, node_j_coords);

        // K_global = T^T * K_local * T
        t.transpose() * k_local * t
    }

    /// Build 12×12 transformation matrix from local to global coordinates
    fn transformation_matrix(
        &self,
        node_i: &Vector3<f64>,
        node_j: &Vector3<f64>,
    ) -> Matrix12x12 {
        let dx = node_j.x - node_i.x;
        let dy = node_j.y - node_i.y;
        let dz = node_j.z - node_i.z;
        let length = (dx * dx + dy * dy + dz * dz).sqrt();

        // Local x-axis along element
        let x_local = Vector3::new(dx / length, dy / length, dz / length);

        // Local y-axis perpendicular to x in XY plane (or XZ if vertical)
        let y_local = if x_local.z.abs() > 0.9 {
            Vector3::new(1.0, 0.0, 0.0).cross(&x_local).normalize()
        } else {
            Vector3::new(0.0, 0.0, 1.0).cross(&x_local).normalize()
        };

        // Local z-axis completes right-handed system
        let z_local = x_local.cross(&y_local);

        // 3×3 rotation matrix
        let r = Matrix3::from_columns(&[x_local, y_local, z_local]);

        // Build 12×12 transformation matrix (block diagonal with 4 copies of r)
        let mut t = Matrix12x12::zeros();
        for i in 0..4 {
            let offset = i * 3;
            t.fixed_slice_mut::<3, 3>(offset, offset).copy_from(&r);
        }

        t
    }

    /// Apply end releases (pin connections) by zeroing moment DOFs
    pub fn apply_releases(&self, k_global: &mut Matrix12x12) {
        if let Some(releases) = &self.releases {
            for (i, &released) in releases.i_end.iter().enumerate() {
                if released {
                    // Zero out row and column for released DOF
                    for j in 0..12 {
                        k_global[(i, j)] = 0.0;
                        k_global[(j, i)] = 0.0;
                    }
                    k_global[(i, i)] = 1.0;  // Keep diagonal for stability
                }
            }
            for (i, &released) in releases.j_end.iter().enumerate() {
                let dof = i + 6;
                if released {
                    for j in 0..12 {
                        k_global[(dof, j)] = 0.0;
                        k_global[(j, dof)] = 0.0;
                    }
                    k_global[(dof, dof)] = 1.0;
                }
            }
        }
    }
}

// Type alias for 12×12 matrix
type Matrix12x12 = nalgebra::SMatrix<f64, 12, 12>;
```

### 3. GPU Worker with CUDA (Rust + CUDA kernels)

Background worker that picks up GPU analysis jobs from Redis, runs CUDA-accelerated solve, and stores results in S3.

```rust
// src/workers/gpu_analysis_worker.rs

use std::sync::Arc;
use aws_sdk_s3::Client as S3Client;
use redis::AsyncCommands;
use sqlx::PgPool;
use uuid::Uuid;

use crate::solver::gpu::cuda_solver::CudaSolver;
use crate::solver::model_extractor::extract_solver_model;
use crate::websocket::ProgressBroadcaster;

pub struct GpuAnalysisWorker {
    db: PgPool,
    redis: redis::Client,
    s3: S3Client,
    broadcaster: Arc<ProgressBroadcaster>,
    gpu_id: usize,
}

impl GpuAnalysisWorker {
    pub async fn run(&self) -> anyhow::Result<()> {
        let mut conn = self.redis.get_async_connection().await?;
        tracing::info!("GPU worker started on GPU {}, listening for jobs...", self.gpu_id);

        loop {
            // Pop from priority queue (highest priority first)
            let job_result: Option<(String, f64)> = conn
                .zpopmax("analysis:jobs", 1)
                .await?;

            let Some((job_id_str, _priority)) = job_result else {
                tokio::time::sleep(tokio::time::Duration::from_secs(5)).await;
                continue;
            };

            let job_id: Uuid = job_id_str.parse()?;

            if let Err(e) = self.process_job(job_id).await {
                tracing::error!("GPU job {job_id} failed: {e}");
                self.mark_failed(job_id, &e.to_string()).await?;
            }
        }
    }

    async fn process_job(&self, job_id: Uuid) -> anyhow::Result<()> {
        // 1. Update job status to running
        let job = sqlx::query_as!(
            crate::db::models::AnalysisJob,
            r#"UPDATE analysis_jobs
               SET status = 'running', started_at = NOW(), gpu_node_id = $2
               WHERE id = $1 RETURNING *"#,
            job_id,
            format!("gpu-{}", self.gpu_id)
        )
        .fetch_one(&self.db)
        .await?;

        // 2. Extract structural model
        let solver_model = extract_solver_model(&self.db, job.model_id).await?;

        // 3. Broadcast progress: model loaded
        self.broadcaster.send(job.id, serde_json::json!({
            "status": "running",
            "stage": "model_loaded",
            "node_count": solver_model.nodes.len(),
            "element_count": solver_model.frame_elements.len()
        }))?;

        // 4. Initialize CUDA solver
        let mut cuda_solver = CudaSolver::new(self.gpu_id)?;

        // 5. Build stiffness matrix on GPU
        self.broadcaster.send(job.id, serde_json::json!({
            "stage": "building_stiffness"
        }))?;

        cuda_solver.build_global_stiffness(&solver_model)?;

        // 6. Apply boundary conditions
        cuda_solver.apply_supports(&solver_model.supports)?;

        // 7. Solve for each load combination
        let load_combo_ids: Vec<Uuid> = serde_json::from_value(
            job.parameters["load_combination_ids"].clone()
        )?;

        let mut all_results = Vec::new();

        for (i, combo_id) in load_combo_ids.iter().enumerate() {
            self.broadcaster.send(job.id, serde_json::json!({
                "stage": "solving",
                "combination": i + 1,
                "total_combinations": load_combo_ids.len()
            }))?;

            // Load force vector for this combination
            let force_vector = extract_load_vector(&self.db, *combo_id, &solver_model).await?;

            // Solve K*u = F on GPU
            let displacements = cuda_solver.solve(&force_vector)?;

            // Recover element forces
            let element_forces = cuda_solver.recover_element_forces(
                &solver_model,
                &displacements
            )?;

            all_results.push(crate::solver::results::CombinationResult {
                combination_id: *combo_id,
                displacements,
                element_forces,
                reactions: cuda_solver.extract_reactions(&solver_model.supports)?,
            });
        }

        // 8. Serialize results and upload to S3
        self.broadcaster.send(job.id, serde_json::json!({
            "stage": "saving_results"
        }))?;

        let results_binary = bincode::serialize(&all_results)?;
        let s3_key = format!("results/{}/{}.bin", job.project_id, job.id);

        self.s3
            .put_object()
            .bucket("structai-results")
            .key(&s3_key)
            .body(results_binary.into())
            .content_type("application/octet-stream")
            .send()
            .await?;

        let results_url = format!("s3://structai-results/{}", s3_key);

        // 9. Compute summary statistics
        let max_displacement = all_results
            .iter()
            .flat_map(|r| &r.displacements)
            .map(|d| d.abs())
            .fold(0.0, f64::max);

        let summary = serde_json::json!({
            "max_displacement": max_displacement,
            "num_combinations": all_results.len(),
            "num_nodes": solver_model.nodes.len(),
            "num_elements": solver_model.frame_elements.len()
        });

        // 10. Update database with completed status
        sqlx::query!(
            r#"UPDATE analysis_jobs
               SET status = 'completed', results_url = $2, results_summary = $3,
                   completed_at = NOW(),
                   wall_time_seconds = EXTRACT(EPOCH FROM (NOW() - started_at))
               WHERE id = $1"#,
            job.id,
            results_url,
            summary
        )
        .execute(&self.db)
        .await?;

        // 11. Notify client via WebSocket
        self.broadcaster.send(job.id, serde_json::json!({
            "status": "completed",
            "results_url": results_url,
            "summary": summary
        }))?;

        tracing::info!("GPU job {job_id} completed successfully");
        Ok(())
    }

    async fn mark_failed(&self, job_id: Uuid, error: &str) -> anyhow::Result<()> {
        sqlx::query!(
            "UPDATE analysis_jobs SET status = 'failed', error_message = $2, completed_at = NOW() WHERE id = $1",
            job_id, error
        )
        .execute(&self.db)
        .await?;

        Ok(())
    }
}

async fn extract_load_vector(
    db: &PgPool,
    combination_id: Uuid,
    model: &crate::solver::model::SolverModel,
) -> anyhow::Result<Vec<f64>> {
    // Get load combination factors
    let combo = sqlx::query!(
        "SELECT factors FROM load_combinations WHERE id = $1",
        combination_id
    )
    .fetch_one(db)
    .await?;

    let factors: Vec<(Uuid, f64)> = serde_json::from_value(combo.factors)?;

    // Build force vector (6 DOF per node)
    let mut force_vector = vec![0.0; model.nodes.len() * 6];

    for (load_case_id, factor) in factors {
        let loads = sqlx::query!(
            r#"SELECT target_type, target_id, load_type, values
               FROM loads WHERE load_case_id = $1"#,
            load_case_id
        )
        .fetch_all(db)
        .await?;

        for load in loads {
            // Apply factored load to force vector
            // (implementation details for different load types)
            // ...
        }
    }

    Ok(force_vector)
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–5)

**Day 1: Project initialization and Rust backend scaffold**
```bash
cargo init structai-api
cd structai-api
cargo add axum tokio serde serde_json sqlx uuid chrono tower-http jsonwebtoken bcrypt
cargo add --features postgres,runtime-tokio-rustls,uuid,chrono,json sqlx
cargo add aws-sdk-s3 redis
```
- `src/main.rs` — Axum app with CORS, tracing, graceful shutdown, health check endpoint
- `src/config.rs` — Environment-based configuration (DATABASE_URL, JWT_SECRET, S3_BUCKET, REDIS_URL, GPU_ENABLED)
- `src/state.rs` — AppState struct (PgPool, Redis, S3Client, ProgressBroadcaster)
- `src/error.rs` — ApiError enum with IntoResponse for consistent error handling
- `Dockerfile` — Multi-stage build (builder with Rust toolchain + runtime with slim Debian)
- `docker-compose.yml` — PostgreSQL 16, Redis 7, MinIO (S3-compatible), Meilisearch

**Day 2: Database schema and migrations**
- `migrations/001_initial.sql` — All core tables: users, organizations, org_members, projects, structural_models, nodes, frame_elements, area_elements, diaphragms, cross_sections, materials, load_cases, loads, load_combinations
- `migrations/002_analysis.sql` — Analysis tables: analysis_jobs, design_results, optimization_runs, comments
- `src/db/mod.rs` — Database pool initialization with connection pooling config
- `src/db/models.rs` — All SQLx structs with FromRow derives and Serialize for JSON responses
- Run `sqlx migrate run` to apply schema
- Seed script for AISC cross-section database (W-shapes, HSS, L, C from AISC v15.0)

**Day 3: Authentication system**
- `src/auth/mod.rs` — JWT token generation and validation, Claims struct, auth middleware extractor
- `src/auth/oauth.rs` — Google and Microsoft OAuth 2.0 flow handlers with state parameter verification
- `src/api/handlers/auth.rs` — POST `/api/auth/register`, POST `/api/auth/login`, POST `/api/auth/oauth/{provider}`, POST `/api/auth/refresh`, GET `/api/auth/me`
- Password hashing with bcrypt (cost factor 12), secure random salt generation
- JWT with 24h access token + 30d refresh token, refresh token rotation on use
- Auth middleware that extracts Claims from Authorization: Bearer header

**Day 4: User and organization CRUD**
- `src/api/handlers/users.rs` — GET `/api/users/me`, PATCH `/api/users/me`, DELETE `/api/users/me`
- `src/api/handlers/orgs.rs` — POST `/api/orgs`, GET `/api/orgs/:slug`, POST `/api/orgs/:id/members`, GET `/api/orgs/:id/members`, DELETE `/api/orgs/:id/members/:user_id`
- `src/api/middleware/auth.rs` — RequireAuth middleware, role-based access control helpers
- Integration tests: full auth flow, token refresh, organization creation and member management

**Day 5: Project CRUD and model scaffold**
- `src/api/handlers/projects.rs` — POST `/api/projects`, GET `/api/projects`, GET `/api/projects/:id`, PATCH `/api/projects/:id`, DELETE `/api/projects/:id`
- `src/api/handlers/models.rs` — POST `/api/projects/:id/models`, GET `/api/projects/:id/models/active`
- `src/api/router.rs` — Route definitions with auth middleware, CORS configuration
- Basic project authorization checks (owner or org member)
- Integration tests: project CRUD, ownership verification, multi-user access

### Phase 2 — FEA Solver Core (Days 6–14)

**Day 6: Frame element stiffness matrix generation**
- `solver-core/` — New Rust workspace member (library crate)
- `solver-core/src/element/frame.rs` — FrameElement struct, local_stiffness_matrix(), transformation_matrix(), global_stiffness_matrix()
- `solver-core/src/section.rs` — CrossSection struct with section properties (A, Ix, Iy, J, Sx, Sy, Zx, Zy, rx, ry, Cw)
- `solver-core/src/material.rs` — Material struct (E, G, Fy, density, poisson)
- Unit tests: cantilever beam stiffness, simply supported beam, portal frame
- Validation against textbook examples (Logan, "A First Course in FEM")

**Day 7: Global stiffness assembly (CPU)**
- `solver-core/src/assembly.rs` — Global stiffness matrix assembly in COO format (triplet: row, col, value)
- `solver-core/src/model.rs` — SolverModel struct (nodes, elements, boundary conditions)
- DOF mapping: 6 DOF per node (ux, uy, uz, θx, θy, θz), global numbering scheme
- Support conditions: fixed, pinned, roller with per-DOF constraint flags
- Unit tests: 2D truss assembly, 3D frame assembly, verify symmetry and positive-definiteness

**Day 8: Sparse matrix solve (CPU backend with Eigen)**
- `solver-core/src/solver/cpu.rs` — CPU solver using Eigen C++ library via CXX bridge
- `solver-core/eigen_bridge.cpp` — C++ wrapper for Eigen SparseQR and SimplicialLLT solvers
- COO to CSR conversion for efficient solve
- Cholesky factorization for symmetric positive-definite stiffness matrices
- Unit tests: solve cantilever beam under point load, compare to analytical deflection

**Day 9: Element force recovery**
- `solver-core/src/recovery.rs` — Extract element forces from global displacement vector
- Local element forces: axial (P), shear (Vy, Vz), moment (Mx, My, Mz), torsion (T)
- Force diagrams: axial force diagram, shear force diagram, bending moment diagram
- Unit tests: simply supported beam with uniform load, verify max moment = wL²/8

**Day 10: Load case handling**
- `solver-core/src/loads.rs` — PointLoad, DistributedLoad, MomentLoad structs
- Load application to global force vector
- Coordinate transformation for element-local distributed loads
- Unit tests: point load at mid-span, uniform distributed load, trapezoidal load

**Day 11: Load combination assembly**
- `solver-core/src/combinations.rs` — Apply load factors and combine multiple load cases
- ASCE 7-22 LRFD combinations: 1.4D, 1.2D+1.6L, 1.2D+1.0L+1.0W, etc.
- ASD combinations: D, D+L, D+0.75L+0.75W, etc.
- Unit tests: verify correct combination coefficients, envelope results across combinations

**Day 12: P-delta analysis**
- `solver-core/src/analysis/pdelta.rs` — Iterative P-delta solver with geometric stiffness
- Geometric stiffness matrix [Kσ] based on axial forces and element geometry
- Convergence check: ||u_{k+1} - u_k|| / ||u_k|| < tolerance
- Unit tests: leaning column (compare to hand calculation), multi-story frame amplification factor

**Day 13: Modal analysis (eigenvalue solver)**
- `solver-core/src/analysis/modal.rs` — Natural frequency and mode shape extraction
- Mass matrix assembly: consistent mass or lumped mass
- Generalized eigenvalue problem: [K]{φ} = ω²[M]{φ}
- Lanczos algorithm for sparse symmetric eigenvalue problems (via ARPACK bindings)
- Unit tests: cantilever beam fundamental frequency (compare to analytical f₁ = 3.516 √(EI/mL⁴))

**Day 14: CPU solver integration and validation**
- `solver-core/src/lib.rs` — Public API for solver: analyze_linear(), analyze_pdelta(), analyze_modal()
- Comprehensive validation suite: AISC Design Guide 28 benchmark problems, ASCE 7 drift examples
- Performance benchmarking: 500 node model, 1000 node model, 2000 node model (CPU timing baseline)
- Documentation: solver theory manual, usage examples, API docs

### Phase 3 — CUDA GPU Acceleration (Days 15–21)

**Day 15: CUDA environment setup**
- Install CUDA Toolkit 12.3, cuDNN, NVIDIA drivers on development machine
- `solver-gpu/` — New Rust workspace member with CUDA build
- `solver-gpu/build.rs` — Custom build script to compile .cu files with nvcc
- `solver-gpu/Cargo.toml` — Dependencies: `cuda-sys`, `cudarc`, `bindgen`
- Hello-world CUDA kernel: vector addition, verify GPU execution

**Day 16: CUDA stiffness matrix assembly kernel**
- `solver-gpu/kernels/assembly.cu` — CUDA kernel for parallel element stiffness generation
- 1 thread per element: compute local stiffness → transform to global → atomic add to COO arrays
- Shared memory optimization for transformation matrix reuse
- Kernel launch configuration: grid size, block size tuning for T4/A100 GPUs
- Unit test: compare CPU vs GPU assembly results, verify bitwise identical matrices

**Day 17: cuSPARSE and cuSOLVER integration**
- `solver-gpu/src/sparse.rs` — Wrapper for cuSPARSE COO-to-CSR conversion
- `solver-gpu/src/solve.rs` — cuSOLVER csrlsvchol (Cholesky factorization and solve)
- GPU memory management: host-to-device copy, device allocation, pinned memory for transfers
- Unit test: solve small system (100 DOF), compare CPU vs GPU solution, verify < 1e-10 relative error

**Day 18: GPU kernel optimization**
- Memory coalescing: ensure contiguous global memory access patterns
- Warp divergence reduction: minimize branching in CUDA kernels
- Stream-based parallelization: overlap memory transfer with computation
- Profiling with NVIDIA Nsight Compute: identify bottlenecks, optimize occupancy
- Benchmark: 5,000 node model (30K DOF) — target <60s solve time on T4 GPU

**Day 19: GPU force recovery kernel**
- `solver-gpu/kernels/recovery.cu` — Parallel element force recovery from global displacements
- Per-element kernel: fetch node displacements → transform to local → compute local forces
- Output: axial, shear, moment at element ends (i and j nodes)
- Unit test: cantilever beam under load, verify forces match CPU solver

**Day 20: GPU P-delta iteration**
- `solver-gpu/src/pdelta.rs` — GPU-accelerated P-delta with geometric stiffness on device
- Iterative loop: solve → extract axial forces → build Kσ → update K → re-solve
- Convergence check on GPU to avoid host-device transfer overhead
- Benchmark: 2,000 node frame with P-delta, compare iteration count vs CPU (should match)

**Day 21: GPU solver integration and benchmarking**
- `solver-gpu/src/lib.rs` — Public API matching CPU solver interface
- Automatic CPU/GPU dispatch based on model size and GPU availability
- End-to-end benchmarks: 100, 500, 1K, 5K, 10K, 20K node models
- Performance targets: 10x speedup vs CPU for models >5K DOF
- Validation: compare GPU vs CPU results on all benchmark problems (max error < 1e-8)

### Phase 4 — Frontend 3D Modeler (Days 22–28)

**Day 22: React project setup and Three.js viewport**
- `frontend/` — New Vite + React + TypeScript project
- Install dependencies: `three`, `@react-three/fiber`, `@react-three/drei`, `zustand`, `tailwindcss`
- `src/components/Viewport3D.tsx` — Basic Three.js canvas with orbit controls, grid, axes
- `src/stores/modelStore.ts` — Zustand store for nodes, elements, selections
- Basic camera controls: orbit, pan, zoom, frame-all

**Day 23: Grid system and node placement**
- `src/components/GridEditor.tsx` — Dialog for defining X/Y/Z grid lines with spacing inputs
- `src/components/Viewport3D.tsx` — Render grid lines as Three.LineSegments, snap-to-grid cursor
- Click-to-place nodes at grid intersections, display node markers as small spheres
- Node selection: click to select, Ctrl+click for multi-select, drag-select box
- `src/hooks/useGridSnap.ts` — Custom hook for snapping 3D cursor to nearest grid point

**Day 24: Frame element creation and rendering**
- `src/components/DrawBeamTool.tsx` — Tool for drawing beams between nodes (click node i → click node j)
- `src/renderers/FrameElementRenderer.tsx` — Instanced mesh rendering for frame elements
- Element coloring: by element type (beam = blue, column = red, brace = green)
- Element selection: raycasting for click-picking, highlight selected elements
- `src/components/PropertyPanel.tsx` — Right panel showing selected element properties

**Day 25: Cross-section assignment and database**
- `src/api/sections.ts` — API client for GET `/api/sections/library?search=W14`
- `src/components/SectionSearchDialog.tsx` — Searchable list of AISC sections with Meilisearch integration
- Assign section to selected elements, display section name as label in 3D viewport
- `src/stores/sectionStore.ts` — Cache for loaded sections, optimistic UI updates

**Day 26: Support conditions and loads**
- `src/components/SupportTool.tsx` — Tool for assigning supports (fixed, pinned, roller) to nodes
- Visual representation: fixed = cube at node, pinned = sphere, roller = triangle
- `src/components/LoadTool.tsx` — Apply point loads (arrow glyph) and distributed loads (multiple arrows along element)
- Load magnitude input with unit selection (kip, kN, lb, N)
- `src/renderers/LoadRenderer.tsx` — Arrow helpers for forces, arc for moments

**Day 27: Load case and combination management**
- `src/components/LoadCasePanel.tsx` — Left sidebar for creating/managing load cases
- Load case list: Dead, Live, Wind, Snow, User-defined with icons
- `src/components/CombinationGenerator.tsx` — Button to auto-generate ASCE 7 LRFD combinations
- Display generated combinations in expandable tree view
- Visual feedback: highlight active load case, show which loads belong to which case

**Day 28: Model persistence and API integration**
- `src/api/models.ts` — saveModel(), loadModel() API calls
- Debounced auto-save every 5 seconds when model changes
- `src/hooks/useModelSync.ts` — Custom hook for bi-directional sync between Zustand and backend
- Optimistic updates: immediate UI feedback, rollback on API error
- Loading states: skeleton loaders for initial project load

### Phase 5 — Analysis Execution + Results Visualization (Days 29–35)

**Day 29: Analysis submission UI**
- `src/components/AnalysisDialog.tsx` — Dialog for configuring and submitting analysis
- Analysis type selector: Linear Static, P-Delta, Modal, Response Spectrum
- Load combination multi-select with "Select All" checkbox
- Parameter inputs: P-delta tolerance, number of modes for modal analysis
- Submit button → POST `/api/projects/:id/analyze`

**Day 30: WebSocket connection for live progress**
- `src/api/websocket.ts` — WebSocket client connecting to `/ws/projects/:id`
- `src/stores/analysisStore.ts` — Store for job status, progress percentage, current stage
- `src/components/AnalysisProgressBar.tsx` — Progress bar component with stage labels
- Real-time updates: "Building stiffness", "Solving combination 3/12", "Saving results"
- Toast notifications on completion or error

**Day 31: Results data fetching and parsing**
- `src/api/results.ts` — GET `/api/jobs/:id/results` → download presigned S3 URL
- Fetch and deserialize binary result data (MessagePack or bincode format)
- `src/types/results.ts` — TypeScript types for CombinationResult, Displacements, ElementForces
- `src/stores/resultsStore.ts` — Store parsed results, index by combination ID

**Day 32: Deformed shape visualization**
- `src/components/DeformedShapeRenderer.tsx` — Render deformed structure with displacement scale factor
- Overlay undeformed (ghost) structure with low opacity gray wireframe
- Displacement scale slider: 1x, 10x, 50x, 100x, Auto (fit to viewport)
- Color coding: map displacement magnitude to color gradient (blue = low, red = high)
- Animation toggle: oscillate between undeformed and deformed at 1 Hz

**Day 33: Member force diagrams**
- `src/components/ForceDiagramRenderer.tsx` — Draw force diagrams along elements
- Axial force diagram: constant along element, display as color-coded tube thickness
- Shear force diagram: linear variation, display as offset colored ribbon
- Moment diagram: quadratic variation, display as ribbon perpendicular to element
- Click element → show all force diagrams with numerical values at critical points

**Day 34: Result data tables**
- `src/components/DisplacementTable.tsx` — Sortable table: Node ID, ux, uy, uz, θx, θy, θz
- `src/components/ReactionTable.tsx` — Support reactions: Node ID, Fx, Fy, Fz, Mx, My, Mz
- `src/components/ElementForceTable.tsx` — Element forces: Element ID, Axial (i), Shear (i), Moment (i), ...
- CSV export button for all tables
- Filter by combination: dropdown to switch between load combinations

**Day 35: Modal analysis results**
- `src/components/ModeShapeRenderer.tsx` — Animated mode shape visualization
- Mode selector: dropdown listing mode 1, 2, 3, ... with frequencies (f₁ = 2.34 Hz, ...)
- Animation: sinusoidal oscillation at visual frequency (not actual frequency)
- Modal participation factors display: % of mass in X, Y, Z directions per mode
- Mode shape table: mode number, frequency, period, participation factors

### Phase 6 — Steel Design Checks (Days 36–42)

**Day 36: AISC 360-22 flexure design module**
- `design-engine/src/aisc360/flexure.rs` — Chapter F: flexure strength calculation
- Section classification: compact, noncompact, slender (flanges and webs)
- Nominal flexural strength Mn: plastic moment Mp, lateral-torsional buckling Mcr, local buckling
- Design flexural strength φMn (LRFD with φ=0.90) or Mn/Ω (ASD with Ω=1.67)
- Unit tests: W14x22 compact section, W21x44 LTB-controlled, slender HSS

**Day 37: AISC 360-22 shear and axial design**
- `design-engine/src/aisc360/shear.rs` — Chapter G: shear strength
- Web shear strength: Cv coefficient based on h/tw ratio, tension field action
- `design-engine/src/aisc360/compression.rs` — Chapter E: compression strength
- Effective length factor K, slenderness ratio KL/r, flexural buckling stress Fcr
- Unit tests: stocky column (inelastic buckling), slender column (elastic buckling)

**Day 38: AISC 360-22 combined forces (interaction)**
- `design-engine/src/aisc360/interaction.rs` — Chapter H: combined axial and flexure
- H1 interaction: doubly/singly symmetric members with P/Pc ≥ 0.2
- H2 interaction: P/Pc < 0.2 (simplified)
- Interaction equations: (Pr/Pc) + (8/9)(Mrx/Mcx + Mry/Mcy) ≤ 1.0
- Unit tests: beam-column with various P/M ratios, verify DCR calculation

**Day 39: Design check orchestration**
- `design-engine/src/lib.rs` — check_member() function: flexure, shear, axial, interaction
- Determine governing check: max DCR across all limit states
- Clause references: "AISC 360-22 F2.1", "AISC 360-22 H1-1a"
- `src/api/handlers/design.rs` — POST `/api/projects/:id/design-check`
- Iterate over all frame elements, run checks for each load combination
- Store results in design_results table

**Day 40: DCR visualization on 3D model**
- `src/components/DCRRenderer.tsx` — Color-code elements by DCR (green < 0.7, yellow 0.7-0.9, red > 0.9, magenta > 1.0)
- Color gradient shader: smooth interpolation based on DCR value
- Legend component showing color scale with DCR breakpoints
- Click element → show detailed design check results in side panel

**Day 41: Design check dashboard**
- `src/components/DesignDashboard.tsx` — Summary view: pie chart of elements by DCR range
- Sortable table: Element ID, Section, Governing Check, DCR, Combination, Status (Pass/Fail)
- Filter controls: by element type, by status, by DCR range
- Export to CSV: full design check results for all members

**Day 42: Detailed design check report**
- `src/components/DesignDetailPanel.tsx` — Clause-by-clause breakdown for selected element
- Display: Check description, Demand, Capacity, DCR, Pass/Fail, Code clause reference
- Expandable sections: Flexure, Shear, Axial, Combined Forces, Deflection
- Copy button: copy detailed checks to clipboard for paste into documentation

## Validation Benchmarks

### 1. Cantilever Beam Deflection (Analytical Verification)

**Problem**: 10 ft cantilever beam, W12x26 section, A992 steel (E = 29,000 ksi), point load P = 5 kip at free end.

**Analytical Solution**:
- Deflection at tip: δ = PL³ / (3EI) = (5 × 10³ × 12³) / (3 × 29000 × 204) = 0.4827 in
- Maximum moment: M = PL = 5 × 10 × 12 = 600 kip-in = 50 kip-ft
- Maximum stress: σ = M/S = 600 / 33.4 = 17.96 ksi

**StructAI Target**: δ = 0.4827 ± 0.0001 in, M = 50.00 ± 0.01 kip-ft, σ = 17.96 ± 0.01 ksi

**Validation Criteria**: Pass if all values within ±0.1% of analytical solution.

### 2. Portal Frame P-Delta Amplification

**Problem**: 20 ft tall portal frame, W14x22 columns, rigid beam, 100 kip vertical load on beam, 10 kip horizontal load.

**Analytical Solution (per AISC Design Guide 28)**:
- First-order drift: Δ₁ = 10 × 20³ / (2 × 12EI) = 0.689 in (for W14x22: I = 199 in⁴)
- P-delta amplification factor: B₂ = 1 / (1 - Σ(Pu)/(0.85ΣPe)) ≈ 1.15
- Second-order drift: Δ₂ = B₂ × Δ₁ = 1.15 × 0.689 = 0.792 in

**StructAI Target**: Δ₂ = 0.792 ± 0.010 in, B₂ = 1.15 ± 0.02

**Validation Criteria**: Pass if P-delta analysis converges in <10 iterations and drift within ±2% of analytical.

### 3. Simply Supported Beam Natural Frequency

**Problem**: 30 ft simply supported beam, W16x26 section, A992 steel (density = 0.490 kip/ft³).

**Analytical Solution**:
- Mass per unit length: m = A × ρ = 7.68 in² × 0.490 / (12³) = 0.00217 kip-s²/in²
- Fundamental frequency: f₁ = (π/2L²) × √(EI/m) = (π / (2 × 360²)) × √(29000 × 301 / 0.00217) = 5.83 Hz

**StructAI Target**: f₁ = 5.83 ± 0.05 Hz

**Validation Criteria**: Pass if modal analysis extracts frequency within ±1% of analytical.

### 4. W14x22 Steel Beam Flexural Capacity (AISC 360-22)

**Problem**: W14x22 beam, Fy = 50 ksi, Lb = 10 ft (laterally unbraced length), Cb = 1.0.

**Hand Calculation (AISC 360-22 Chapter F)**:
- Section is compact (Table B4.1b check: λf = bf/(2tf) = 7.51 < λpf = 9.15)
- Mp = Fy × Zx = 50 × 29.0 = 1450 kip-in = 120.8 kip-ft
- Lp = 1.76ry√(E/Fy) = 1.76 × 1.33 × √(29000/50) = 56.4 in = 4.70 ft
- Lr = 1.95rts(E/(0.7Fy))√(J/(Sxho) + √((J/(Sxho))² + 6.76(0.7Fy/E)²)) ≈ 14.2 ft
- Since Lp < Lb < Lr: Lateral-torsional buckling controls
- Mn = Cb[Mp - (Mp - 0.7FySx)(Lb - Lp)/(Lr - Lp)] = 1.0[1450 - (1450 - 0.7×50×26.0)(10×12 - 56.4)/(14.2×12 - 56.4)] = 1220 kip-in = 101.7 kip-ft
- φMn = 0.9 × 101.7 = 91.5 kip-ft (LRFD)

**StructAI Target**: φMn = 91.5 ± 0.5 kip-ft, controlling limit state = "AISC 360-22 F2.2 (LTB)"

**Validation Criteria**: Pass if design capacity within ±1% and correct limit state identified.

### 5. 5-Story Moment Frame Linear Static Analysis (AISC Benchmark)

**Problem**: 5-story, 3-bay moment frame from AISC Design Guide 28 Example 1.
- Story height: 13 ft (1st), 12 ft (2nd-5th)
- Bay width: 30 ft × 3 bays
- Beams: W21x44 (all floors)
- Columns: W14x90 (1st-2nd story), W14x68 (3rd-4th story), W14x53 (5th story)
- Loading: Dead = 80 psf, Live = 50 psf, uniform on beams
- Load combination: 1.2D + 1.6L

**AISC Design Guide 28 Published Results**:
- Roof drift: 1.23 in
- Max column base moment: 452 kip-ft
- Max beam moment (1st floor): 183 kip-ft

**StructAI Target**: Roof drift = 1.23 ± 0.05 in, base moment = 452 ± 5 kip-ft, beam moment = 183 ± 3 kip-ft

**Validation Criteria**: Pass if all values within ±3% of published AISC results. This validates full workflow: model input, load application, analysis, force recovery.

---

## Critical Files

```
structai/
├── solver-core/                           # Shared FEA solver library
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── element/
│   │   │   ├── mod.rs                     # Element trait definition
│   │   │   ├── frame.rs                   # 3D frame element (beam-column)
│   │   │   ├── shell.rs                   # 4-node shell element
│   │   │   └── solid.rs                   # 8-node brick element
│   │   ├── assembly.rs                    # Global stiffness matrix assembly
│   │   ├── model.rs                       # SolverModel struct (nodes, elements, BCs)
│   │   ├── solver/
│   │   │   ├── mod.rs
│   │   │   ├── cpu.rs                     # CPU solver (Eigen wrapper)
│   │   │   └── results.rs                 # Result structs
│   │   ├── analysis/
│   │   │   ├── mod.rs
│   │   │   ├── linear.rs                  # Linear static analysis
│   │   │   ├── pdelta.rs                  # P-delta analysis
│   │   │   ├── modal.rs                   # Modal analysis (eigenvalue)
│   │   │   ├── response_spectrum.rs       # Response spectrum analysis
│   │   │   └── time_history.rs            # Nonlinear time-history
│   │   ├── loads.rs                       # Load structs (point, distributed, pressure)
│   │   ├── section.rs                     # CrossSection struct
│   │   ├── material.rs                    # Material struct
│   │   └── validation/
│   │       ├── benchmarks.rs              # Validation benchmark problems
│   │       └── nafems.rs                  # NAFEMS FEA benchmarks
│   └── tests/
│       ├── frame_tests.rs                 # Frame element unit tests
│       ├── assembly_tests.rs              # Assembly validation
│       └── solver_tests.rs                # Solver accuracy tests
│
├── solver-gpu/                            # CUDA GPU acceleration
│   ├── Cargo.toml
│   ├── build.rs                           # Custom build for CUDA
│   ├── src/
│   │   ├── lib.rs
│   │   ├── assembly.rs                    # GPU stiffness assembly wrapper
│   │   ├── sparse.rs                      # cuSPARSE wrapper
│   │   ├── solve.rs                       # cuSOLVER wrapper
│   │   ├── pdelta.rs                      # GPU P-delta iteration
│   │   └── recovery.rs                    # GPU force recovery
│   └── kernels/
│       ├── assembly.cu                    # CUDA kernel: parallel stiffness
│       ├── recovery.cu                    # CUDA kernel: force recovery
│       └── pdelta.cu                      # CUDA kernel: geometric stiffness
│
├── design-engine/                         # Code compliance checking
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── aisc360/
│   │   │   ├── mod.rs
│   │   │   ├── flexure.rs                 # Chapter F: flexural strength
│   │   │   ├── shear.rs                   # Chapter G: shear strength
│   │   │   ├── compression.rs             # Chapter E: compression strength
│   │   │   ├── tension.rs                 # Chapter D: tension strength
│   │   │   ├── interaction.rs             # Chapter H: combined forces
│   │   │   └── stability.rs               # Chapter C: stability analysis
│   │   ├── aci318/
│   │   │   ├── mod.rs
│   │   │   ├── flexure.rs                 # ACI 318 Chapter 22: flexure
│   │   │   ├── shear.rs                   # ACI 318 Chapter 22: shear
│   │   │   ├── axial.rs                   # ACI 318 Chapter 22: axial
│   │   │   └── interaction.rs             # P-M interaction diagrams
│   │   └── asce7/
│   │       ├── mod.rs
│   │       ├── combinations.rs            # Load combination generation
│   │       ├── drift.rs                   # Story drift checks
│   │       └── seismic.rs                 # Seismic design provisions
│   └── tests/
│       ├── aisc_tests.rs                  # AISC design verification
│       └── aci_tests.rs                   # ACI design verification
│
├── structai-api/                          # Rust API server
│   ├── Cargo.toml
│   ├── src/
│   │   ├── main.rs                        # Axum app setup, router, startup
│   │   ├── config.rs                      # Environment config
│   │   ├── state.rs                       # AppState (PgPool, Redis, S3)
│   │   ├── error.rs                       # ApiError types
│   │   ├── auth/
│   │   │   ├── mod.rs                     # JWT middleware, Claims
│   │   │   └── oauth.rs                   # Google/Microsoft OAuth handlers
│   │   ├── api/
│   │   │   ├── router.rs                  # Route definitions
│   │   │   ├── handlers/
│   │   │   │   ├── auth.rs                # Register, login, OAuth
│   │   │   │   ├── users.rs               # Profile CRUD
│   │   │   │   ├── orgs.rs                # Organization management
│   │   │   │   ├── projects.rs            # Project CRUD
│   │   │   │   ├── models.rs              # Structural model CRUD
│   │   │   │   ├── analysis.rs            # Analysis job creation/status
│   │   │   │   ├── design.rs              # Design check execution
│   │   │   │   ├── optimization.rs        # AI optimization runs
│   │   │   │   ├── sections.rs            # Cross-section library search
│   │   │   │   ├── billing.rs             # Stripe checkout/portal
│   │   │   │   ├── ifc.rs                 # IFC/BIM import
│   │   │   │   ├── reports.rs             # PDF report generation
│   │   │   │   └── webhooks/
│   │   │   │       └── stripe.rs          # Stripe webhook handler
│   │   │   └── ws/
│   │   │       ├── mod.rs                 # WebSocket upgrade handler
│   │   │       └── analysis_progress.rs   # Live analysis progress
│   │   ├── middleware/
│   │   │   ├── auth.rs                    # Auth middleware
│   │   │   └── plan_limits.rs             # Plan enforcement
│   │   ├── services/
│   │   │   ├── s3.rs                      # S3 client helpers
│   │   │   └── meilisearch.rs             # Search client
│   │   ├── solver/
│   │   │   ├── model_extractor.rs         # Extract solver model from DB
│   │   │   └── cpu.rs                     # CPU analysis execution
│   │   └── workers/
│   │       ├── mod.rs                     # Worker pool management
│   │       └── gpu_analysis_worker.rs     # GPU analysis execution
│   ├── migrations/
│   │   ├── 001_initial.sql                # Core tables
│   │   └── 002_analysis.sql               # Analysis tables
│   └── tests/
│       ├── api_integration.rs             # API integration tests
│       └── analysis_e2e.rs                # End-to-end analysis tests
│
├── optimization-service/                  # Python AI optimization
│   ├── requirements.txt
│   ├── main.py                            # FastAPI app
│   ├── genetic_algorithm.py               # GA optimization engine
│   ├── fitness.py                         # Fitness evaluation (calls Rust solver)
│   ├── constraints.py                     # Constraint handling
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
│   │   │   ├── projectStore.ts            # Project state
│   │   │   ├── modelStore.ts              # Structural model state
│   │   │   ├── analysisStore.ts           # Analysis job state
│   │   │   ├── resultsStore.ts            # Results cache
│   │   │   ├── sectionStore.ts            # Cross-section database
│   │   │   └── designStore.ts             # Design check results
│   │   ├── hooks/
│   │   │   ├── useAnalysisProgress.ts     # WebSocket hook for live progress
│   │   │   ├── useModelSync.ts            # Bi-directional model sync
│   │   │   └── useGridSnap.ts             # Grid snapping logic
│   │   ├── lib/
│   │   │   ├── api.ts                     # Axios API client
│   │   │   └── three-utils.ts             # Three.js helpers
│   │   ├── pages/
│   │   │   ├── Dashboard.tsx              # Project list
│   │   │   ├── Editor.tsx                 # Main editor (3D modeler)
│   │   │   ├── Results.tsx                # Analysis results viewer
│   │   │   ├── DesignCheck.tsx            # Design check dashboard
│   │   │   ├── Billing.tsx                # Plan management
│   │   │   ├── Login.tsx                  # Auth pages
│   │   │   └── Register.tsx
│   │   ├── components/
│   │   │   ├── Viewport3D/
│   │   │   │   ├── Viewport3D.tsx         # Three.js canvas
│   │   │   │   ├── GridEditor.tsx         # Grid definition dialog
│   │   │   │   ├── FrameElementRenderer.tsx # Instanced beam rendering
│   │   │   │   ├── SupportRenderer.tsx    # Support symbols
│   │   │   │   └── LoadRenderer.tsx       # Load arrows and symbols
│   │   │   ├── Tools/
│   │   │   │   ├── DrawBeamTool.tsx
│   │   │   │   ├── SupportTool.tsx
│   │   │   │   ├── LoadTool.tsx
│   │   │   │   └── SectionSearchDialog.tsx
│   │   │   ├── Analysis/
│   │   │   │   ├── AnalysisDialog.tsx     # Analysis configuration
│   │   │   │   ├── AnalysisProgressBar.tsx
│   │   │   │   └── LoadCasePanel.tsx
│   │   │   ├── Results/
│   │   │   │   ├── DeformedShapeRenderer.tsx
│   │   │   │   ├── ForceDiagramRenderer.tsx
│   │   │   │   ├── ModeShapeRenderer.tsx
│   │   │   │   ├── DisplacementTable.tsx
│   │   │   │   └── ReactionTable.tsx
│   │   │   ├── Design/
│   │   │   │   ├── DCRRenderer.tsx        # Color-coded DCR visualization
│   │   │   │   ├── DesignDashboard.tsx
│   │   │   │   └── DesignDetailPanel.tsx
│   │   │   └── common/
│   │   │       ├── PropertyPanel.tsx
│   │   │       └── CombinationGenerator.tsx
│   │   └── types/
│   │       ├── model.ts
│   │       ├── results.ts
│   │       └── design.ts
│   └── public/
│
├── k8s/                                   # Kubernetes manifests
│   ├── api-deployment.yaml
│   ├── gpu-worker-deployment.yaml
│   ├── postgres-statefulset.yaml
│   ├── redis-deployment.yaml
│   ├── meilisearch-deployment.yaml
│   ├── optimization-service-deployment.yaml
│   └── ingress.yaml
│
├── docker-compose.yml                     # Local development stack
├── Cargo.toml                             # Workspace root
└── .github/
    └── workflows/
        ├── ci.yml                         # Test + lint on PR
        ├── gpu-build.yml                  # Build CUDA solver
        └── deploy.yml                     # Build Docker images + deploy to K8s
```

---

## Solver Validation Suite

### Benchmark 1: Simply Supported Beam Deflection (Analytical Verification)

**Problem**: 20 ft simply supported beam, W16x26 section, A992 steel (E = 29,000 ksi), uniform load w = 2 kip/ft.

**Analytical Solution**:
- Maximum deflection at midspan: δ = 5wL⁴ / (384EI) = 5 × 2 × (20×12)⁴ / (384 × 29000 × 301) = **0.768 in**
- Maximum moment at midspan: M = wL² / 8 = 2 × 20² / 8 = **100 kip-ft**
- Maximum stress: σ = M/S = 100 × 12 / 38.4 = **31.25 ksi**

**StructAI Target**: δ = 0.768 ± 0.002 in, M = 100.0 ± 0.1 kip-ft, σ = 31.25 ± 0.05 ksi

**Validation Criteria**: Pass if all values within ±0.3% of analytical solution.

### Benchmark 2: Cantilever Column Buckling (Eigenvalue Analysis)

**Problem**: 12 ft cantilever column, W10x33 section, A992 steel (E = 29,000 ksi), fixed at base.

**Analytical Solution**:
- Effective length factor K = 2.0 (cantilever)
- Critical buckling load: Pcr = π²EI / (KL)² = π² × 29000 × 171 / (2.0 × 12×12)² = **187.4 kip**

**StructAI Target**: Pcr = 187.4 ± 2.0 kip

**Validation Criteria**: Pass if buckling load within ±1% of analytical.

### Benchmark 3: Portal Frame Fundamental Frequency

**Problem**: 15 ft tall portal frame, W14x22 columns, W18x35 beam, A992 steel (density = 490 lb/ft³).

**Analytical Solution** (from ASCE 7 approximate formula):
- Fundamental period: T₁ ≈ 0.1N (where N = number of stories above base) = **0.1 sec**
- Fundamental frequency: f₁ = 1/T₁ = **10.0 Hz** (approximate)

**StructAI Target**: f₁ = 8-12 Hz (broad range due to approximate formula)

**Validation Criteria**: Pass if modal analysis extracts frequency in expected range, mode shape shows sway mode.

### Benchmark 4: 2D Truss Deflection (NAFEMS Benchmark)

**Problem**: NAFEMS Standard Benchmark FV52 — cantilever truss with point load at free end.
- 10 members, 6 nodes, all members A = 1.0 in², E = 10,000 ksi
- Point load P = 10 kip at node 6

**Published NAFEMS Result**: Vertical deflection at node 6 = **0.150 in**

**StructAI Target**: δ = 0.150 ± 0.001 in

**Validation Criteria**: Pass if deflection within ±0.5% of NAFEMS published value.

### Benchmark 5: Multi-Story Frame Story Drift (ASCE 7 Compliance)

**Problem**: 3-story moment frame, 12 ft story heights, W14x90 columns, W21x44 beams.
- Lateral load: 100 kip at roof, 100 kip at 2nd floor, 100 kip at 1st floor
- Load combination: 1.0E (earthquake)

**ASCE 7-22 Allowable Drift**: Δa = 0.020hsx / Cd (for Cd = 5.5) = 0.020 × 12×12 / 5.5 = **0.524 in per story**

**Analysis Target**:
- 1st story drift: < 0.524 in
- 2nd story drift: < 0.524 in
- 3rd story drift: < 0.524 in

**StructAI Target**: Calculate story drifts, verify all < allowable, generate passing drift report.

**Validation Criteria**: Pass if analysis correctly identifies inter-story drifts and flags violations if any.

---

## Verification

### Integration Testing Checklist

1. **Auth flow** — Register → login → JWT issued → authorized API calls → token refresh → logout
2. **Project CRUD** — Create → update model → save → reload → verify model state preserved
3. **3D modeling** — Place nodes on grid → draw beams → assign sections → verify geometry
4. **Section library** — Search "W14" → results returned → select W14x22 → properties displayed
5. **Load application** — Define dead load → apply uniform load on beam → load vector correctly assembled
6. **Load combinations** — Generate ASCE 7 LRFD combinations → verify 1.2D+1.6L created
7. **CPU analysis** — Small model (50 DOF) → CPU solve → results returned in <5s
8. **GPU analysis** — Large model (5000 DOF) → GPU job queued → WebSocket progress → results in S3
9. **Deformed shape** — Load results → render deformed shape → scale slider adjusts displacement
10. **Force diagrams** — Click element → moment diagram displayed → values match hand calculation
11. **Modal analysis** — Extract 5 modes → frequencies displayed → mode shapes animated
12. **Design checks** — Run AISC checks → DCRs calculated → elements colored by DCR
13. **Design detail** — Click over-stressed element → detailed clause-by-clause checks shown
14. **PDF report** — Generate calculation report → PDF downloaded → contains all member checks
15. **Plan limits** — Free user → large model → blocked with upgrade prompt
16. **Billing** — Subscribe to Pro → Stripe checkout → webhook → plan updated → limits lifted
17. **Collaboration** — Share project → viewer accesses read-only 3D model → no edit allowed
18. **IFC import** — Upload Revit IFC → nodes and elements extracted → model displays in 3D
19. **Optimization** — Run AI optimization → section sizes updated → weight reduced → DCRs still pass
20. **Error handling** — Singular stiffness matrix (missing support) → error message with fix suggestion

### SQL Verification Queries

```sql
-- 1. Analysis job throughput and success rate
SELECT
    analysis_type,
    status,
    COUNT(*) as count,
    AVG(wall_time_seconds)::int as avg_time_sec,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY wall_time_seconds)::int as p95_time_sec
FROM analysis_jobs
WHERE created_at >= NOW() - INTERVAL '7 days'
GROUP BY analysis_type, status;

-- 2. User plan distribution and conversion
SELECT plan, COUNT(*) as users,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 1) as pct
FROM users GROUP BY plan ORDER BY users DESC;

-- 3. Most common cross-sections used
SELECT cs.name, cs.shape_type,
    COUNT(DISTINCT fe.id) as usage_count,
    COUNT(DISTINCT fe.model_id) as models_using
FROM cross_sections cs
JOIN frame_elements fe ON fe.section_id = cs.id
WHERE cs.is_builtin = true
GROUP BY cs.id
ORDER BY usage_count DESC
LIMIT 20;

-- 4. Design check failure rate by member type
SELECT
    fe.element_type,
    COUNT(*) as total_checks,
    SUM(CASE WHEN dr.status = 'fail' THEN 1 ELSE 0 END) as failures,
    ROUND(100.0 * SUM(CASE WHEN dr.status = 'fail' THEN 1 ELSE 0 END) / COUNT(*), 1) as failure_pct,
    AVG(dr.dcr) as avg_dcr
FROM design_results dr
JOIN frame_elements fe ON dr.element_id = fe.id AND dr.element_type = 'frame'
GROUP BY fe.element_type;

-- 5. GPU utilization and queue depth
SELECT
    DATE_TRUNC('hour', created_at) as hour,
    COUNT(*) FILTER (WHERE status = 'queued') as queued,
    COUNT(*) FILTER (WHERE status = 'running') as running,
    COUNT(*) FILTER (WHERE status = 'completed') as completed,
    AVG(wall_time_seconds) FILTER (WHERE status = 'completed')::int as avg_solve_time
FROM analysis_jobs
WHERE created_at >= NOW() - INTERVAL '24 hours'
    AND parameters->>'execution_mode' = 'gpu'
GROUP BY hour
ORDER BY hour DESC;
```

### Performance Benchmarks

| Operation | Target | Method |
|-----------|--------|--------|
| CPU solver: 500 DOF linear static | <5s | Server timing benchmark |
| GPU solver: 5,000 DOF linear static | <60s | GPU timing with T4 |
| GPU solver: 30,000 DOF linear static | <180s | GPU timing with A100 |
| P-delta iteration: 2,000 DOF | <10 iterations | Convergence monitoring |
| Modal analysis: 1,000 DOF, 10 modes | <30s | Lanczos eigenvalue timing |
| Design checks: 100 members, 12 combinations | <10s | Design engine benchmark |
| 3D viewport: 1,000 elements render | 60 FPS | Chrome FPS counter |
| 3D viewport: 10,000 elements render | 30 FPS | Chrome FPS with instancing |
| API: create analysis job | <200ms | p95 latency (Prometheus) |
| API: search cross-sections | <100ms | p95 latency with Meilisearch |
| WebSocket: progress update latency | <100ms | Time from worker to client |
| IFC import: 500 element model | <15s | Server-side parsing + DB insert |
| PDF report generation: 50 page calc | <20s | Server-side rendering with reportlab |

---

## Deployment Architecture

### Infrastructure

```
┌─────────────────────────────────────────────────┐
│                  CloudFront CDN                  │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────┐ │
│  │ Frontend SPA │  │ Static      │  │ Reports  │ │
│  │ (HTML/JS/CSS)│  │ Assets      │  │ (PDF)    │ │
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
│ Multi-AZ     │ │ Cluster mode │ │  IFC files)  │
└──────────────┘ └──────┬───────┘ └──────────────┘
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ GPU Worker   │ │ GPU Worker   │ │ GPU Worker   │
│ (CUDA solver)│ │ (CUDA solver)│ │ (CUDA solver)│
│ g5.xlarge    │ │ g5.xlarge    │ │ g5.xlarge    │
│ NVIDIA T4    │ │ NVIDIA T4    │ │ NVIDIA T4    │
│ HPA: 2-10    │ │              │ │              │
└──────────────┘ └──────────────┘ └──────────────┘
```

### Scaling Strategy

- **API servers**: Horizontal pod autoscaler (HPA) at 70% CPU, min 3, max 15
- **GPU workers**: HPA based on Redis queue depth — scale up when queue > 3 jobs, scale down when idle 10 min
- **Worker instances**: AWS g5.xlarge (4 vCPU, 16GB RAM, NVIDIA T4 GPU) for GPU-accelerated FEA
- **Large job instances**: g5.4xlarge (NVIDIA A100) for models >30K DOF, launched on-demand
- **Database**: RDS PostgreSQL r6g.xlarge with 1 read replica for section search queries
- **Redis**: ElastiCache r6g.large cluster with 1 primary + 2 replicas
- **Object storage**: S3 Standard with lifecycle policy (IA after 30 days, Glacier after 90 days)

### Kubernetes Resource Limits

```yaml
# API Server
resources:
  requests:
    memory: "512Mi"
    cpu: "500m"
  limits:
    memory: "1Gi"
    cpu: "1000m"

# GPU Worker
resources:
  requests:
    memory: "8Gi"
    cpu: "3000m"
    nvidia.com/gpu: 1
  limits:
    memory: "14Gi"
    cpu: "4000m"
    nvidia.com/gpu: 1
```

### S3 Lifecycle Policy

```json
{
  "Rules": [
    {
      "ID": "analysis-results-lifecycle",
      "Prefix": "results/",
      "Status": "Enabled",
      "Transitions": [
        { "Days": 30, "StorageClass": "STANDARD_IA" },
        { "Days": 90, "StorageClass": "GLACIER" }
      ],
      "Expiration": { "Days": 730 }
    },
    {
      "ID": "reports-lifecycle",
      "Prefix": "reports/",
      "Status": "Enabled",
      "Transitions": [
        { "Days": 90, "StorageClass": "STANDARD_IA" }
      ],
      "Expiration": { "Days": 365 }
    }
  ]
}
```

---

## Post-MVP Roadmap

### v1.1 — BIM Integration & Concrete Design (Weeks 11-14)

**Features:**
- IFC 2x3/4 import from Revit and Tekla with automatic element recognition
- Concrete design per ACI 318-19: flexural reinforcement, shear design, P-M interaction
- Area element support: walls and slabs with mesh generation
- BIM change detection: compare model versions and highlight differences
- Export analysis results to IFC for BIM coordination

**Effort:** 4 weeks
- Week 11: IFC parser integration (IfcOpenShell Python bindings), element mapping, geometry cleanup
- Week 12: Area element stiffness matrix, mesh generation, assembly integration
- Week 13: ACI 318 concrete design module (flexure, shear, axial, interaction)
- Week 14: Concrete design visualization, rebar schedule generation, report integration

**Success Metrics:**
- Import 80%+ of Revit structural models without manual cleanup
- Concrete design checks pass validation against ACI manual calculations
- Area element analysis matches SAP2000 shell results within 5%

### v1.2 — Seismic Analysis & AI Optimization (Weeks 15-20)

**Features:**
- Response spectrum analysis per ASCE 7-22 with CQC/SRSS modal combination
- Equivalent lateral force (ELF) procedure with automatic base shear calculation
- Story drift checks and graphical drift profile diagrams
- AI design optimization: genetic algorithm to minimize weight while satisfying all code checks
- Topology optimization for bracing layout suggestions

**Effort:** 6 weeks
- Week 15: Response spectrum analysis implementation, modal combination methods
- Week 16: ELF procedure, ASCE 7 seismic weight and force distribution
- Week 17: Story drift calculation and visualization, drift check reporting
- Week 18-19: Python optimization service: genetic algorithm, constraint handling, Rust solver integration
- Week 20: Topology optimization for brace placement, optimization UI and dashboard

**Success Metrics:**
- Response spectrum analysis matches ETABS modal results within 3%
- AI optimization reduces structural weight by 15-25% while maintaining all DCRs < 0.95
- Optimization converges in <50 generations for typical 200-element frame

### v1.3 — Nonlinear Analysis & Connections (Weeks 21-26)

**Features:**
- Nonlinear time-history analysis with Newmark-beta integration
- Pushover analysis with capacity curve generation and hinge formation tracking
- Material nonlinearity: fiber-based beam elements, concrete cracking, steel plasticity
- Connection design: shear tabs, moment connections, base plates per AISC 360 Chapter J
- Bolt group and weld group analysis with IC method

**Effort:** 6 weeks
- Week 21: Newmark-beta time-history integrator, ground motion record input
- Week 22: Pushover analysis, load-controlled and displacement-controlled modes
- Week 23: Fiber-based beam element with material nonlinearity, plastic hinge models
- Week 24-25: AISC Chapter J connection design (shear, moment, base plates)
- Week 26: Connection design UI, detail drawings, bolt/weld pattern generation

**Success Metrics:**
- Time-history analysis matches OpenSees results for benchmark ground motions
- Pushover capacity curve matches experimental test data within 10%
- Connection designs pass independent PE review for code compliance

### v1.4 — International Codes & Advanced Features (Weeks 27-32)

**Features:**
- Eurocode 2/3/8 design checks with configurable National Annexes
- AS/NZS 4100 (steel) and AS 3600 (concrete) for Australia/New Zealand
- Wind load auto-generation per ASCE 7, Eurocode 1, AS/NZS 1170
- Snow load generation with drift and sliding loads
- Performance-based seismic design with acceptance criteria

**Effort:** 6 weeks
- Week 27: Eurocode 3 steel design module
- Week 28: Eurocode 2 concrete design module
- Week 29: AS/NZS code modules (4100 steel, 3600 concrete)
- Week 30: Wind load generation: ASCE 7 Method 2, Eurocode analytical procedure
- Week 31: Snow load generation, drift loads, unbalanced loads
- Week 32: Performance-based design acceptance criteria (ASCE 41), integration testing

**Success Metrics:**
- Eurocode design checks validated against European software (Sofistik, Dlubal)
- Wind load generation matches hand calculations per code
- Performance-based design workflow validated by structural PE

### v2.0 — Enterprise & API Access (Weeks 33-42)

**Features:**
- SSO/SAML integration for enterprise authentication
- Audit logs for all model changes and analysis runs
- API access for scripting and automation (Python SDK, REST API)
- On-premise deployment option with Docker Compose or K8s Helm chart
- Custom report templates with regulatory compliance formatting
- Data residency options (US, EU, Asia-Pacific regions)

**Effort:** 10 weeks
- Week 33-34: SAML integration, SSO providers (Okta, Azure AD)
- Week 35: Audit logging system, activity dashboard
- Week 36-37: Public API design, OpenAPI spec, rate limiting
- Week 38-39: Python SDK for programmatic model creation and analysis
- Week 40-41: On-premise deployment packaging, Helm charts, installation docs
- Week 42: Custom report templates, YAML-based template engine, enterprise pilot program

**Success Metrics:**
- 5+ enterprise customers (>50 seats) signed
- API handles 10K+ requests/day across all customers
- On-premise deployment completes in <2 hours with single command
- 95% customer satisfaction score from enterprise pilot users
