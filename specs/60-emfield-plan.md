# 60. EMField — Electromagnetic Field Simulation and Antenna Design Platform

## Implementation Plan

**MVP Scope:** Browser-based 3D parametric geometry editor with CAD import (STEP/STL/DXF) and automatic tetrahedral/hexahedral mesh generation, GPU-accelerated FDTD (Finite-Difference Time-Domain) solver compiled to WebAssembly for client-side execution of electrically small problems (<1 wavelength³) and server-side CUDA for larger problems, broadband S-parameter extraction (1-port, 2-port, N-port) with automatic de-embedding, 3D far-field radiation pattern visualization via WebGL/Three.js with interactive rotation and polar/rectangular plots, material database with 200+ common dielectrics and metals (εr, tanδ, σ), 5 antenna templates (microstrip patch, dipole, PIFA, horn, Vivaldi) with AI-assisted parametric optimization using genetic algorithms, field visualization (E-field, H-field, J-field, power density) with 3D volume rendering and 2D slice planes, VSWR and impedance plots with Smith chart, PostgreSQL metadata + S3 blob storage for meshes and field data, Stripe billing with three tiers (Free / Pro $199/mo / Advanced $449/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| FDTD Solver Core | Rust (native + WASM) | Custom Yee-grid FDTD with PML absorbing boundaries, compiled to WASM for client, native for server |
| GPU FDTD | CUDA (C++) + Rust bindings | GPU-accelerated field updates for large-scale server simulations |
| WASM Compilation | `wasm-pack` + `wasm-bindgen` | Client-side solver for electrically small problems (< 1M cells) |
| Geometry & Meshing | Rust | Delaunay tetrahedral mesher, octree-based hexahedral mesher, CSG boolean operations |
| Optimization | Python 3.12 (FastAPI) | Genetic algorithm, particle swarm, CMA-ES for antenna parameter optimization |
| Database | PostgreSQL 16 | Projects, users, simulations, antenna templates, material properties |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | Mesh files, field data (HDF5), geometry (STEP), results (S-parameters, patterns) |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| 3D Geometry Editor | Three.js + React Three Fiber | CSG operations, parametric primitives, CAD import via OpenCascade.js |
| 3D Visualization | Three.js + WebGL 2.0 | Volume rendering for field data, mesh preview, radiation patterns |
| Real-time | WebSocket (Axum) | Live simulation progress, field animation streaming |
| Job Queue | Redis 7 + Tokio tasks | Server-side simulation job management, optimization runs |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | WASM bundle delivery, static frontend assets |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, solver performance, error tracking |
| CI/CD | GitHub Actions | WASM build, CUDA compilation, Rust tests, Docker image push |

### Key Architecture Decisions

1. **Hybrid client/server FDTD solver with WASM threshold at 1M cells**: Electrically small problems (< 1 wavelength³, typically <1M FDTD cells) run entirely in the browser via WASM, providing instant feedback with zero server cost. This covers most antenna designs at lower frequencies (sub-6 GHz) and small microwave components. Larger problems (5G mmWave arrays, full vehicle EMC, electrically large RCS) are submitted to GPU-accelerated server clusters. The threshold is configurable per plan tier.

2. **FDTD as primary MVP solver rather than FEM or MoM**: FDTD is the most straightforward to parallelize on GPUs, handles broadband frequency sweeps in a single run (unlike FEM which requires one solve per frequency), and has well-established WASM compilation pathways. The Yee grid update equations are embarrassingly parallel and memory-bandwidth bound, making them ideal for GPU acceleration. FEM and MoM solvers are deferred to post-MVP as they require complex sparse matrix factorization that is harder to compile to WASM.

3. **Three.js for 3D geometry editor with CSG via OpenCascade.js**: Three.js provides mature WebGL rendering, camera controls, and scene management. For parametric CAD features (boolean operations, fillets, chamfers, extrusions), we use OpenCascade.js (OpenCascade compiled to WASM), which is the industry-standard geometry kernel used by FreeCAD and other CAD tools. This allows STEP/IGES import and precise geometric operations without requiring desktop CAD software.

4. **Octree-based hexahedral meshing for FDTD rather than unstructured tetrahedral**: FDTD requires a structured Cartesian grid (Yee grid), so we use an octree to adaptively refine the grid near geometry edges and material interfaces while keeping coarse cells in homogeneous regions. This produces an axis-aligned hexahedral mesh suitable for FDTD. Tetrahedral meshes are used only for FEM (post-MVP). The octree mesher provides 10-100x memory reduction compared to uniform grids while maintaining FDTD stability.

5. **HDF5 for field data storage with chunked compression**: 3D electromagnetic field data is massive (6 vector components × millions of cells × hundreds of timesteps = GB to TB). We store field data in HDF5 format with gzip compression and chunked access patterns, enabling streaming partial field data to the client for visualization without downloading entire datasets. S3 supports byte-range requests on HDF5, allowing efficient remote access to specific field slices or timesteps.

6. **Genetic algorithm for antenna optimization via Python FastAPI service**: Antenna optimization requires evaluating hundreds to thousands of design variants. We run a Python FastAPI service that manages optimization campaigns: it generates parameter sets, submits simulation jobs to the Rust backend, collects results, and evolves the next generation. Python's mature ecosystem (scipy, deap, pymoo) provides battle-tested genetic algorithm implementations. The Rust backend focuses on high-performance FDTD solving, while Python handles the optimization loop.

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
    plan TEXT NOT NULL DEFAULT 'free',  -- free | pro | advanced | enterprise
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX users_email_idx ON users(email);
CREATE INDEX users_stripe_idx ON users(stripe_customer_id);

-- Organizations (for Team/Enterprise plans)
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

-- Projects (geometry + simulation workspace)
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    geometry_data JSONB NOT NULL DEFAULT '{}',  -- CSG tree, primitives, parameters
    mesh_url TEXT,  -- S3 URL for generated mesh (HDF5 or VTK format)
    mesh_stats JSONB DEFAULT '{}',  -- {cell_count, node_count, min_cell_size, max_cell_size}
    settings JSONB DEFAULT '{}',  -- Frequency range, boundary conditions, excitations
    is_public BOOLEAN NOT NULL DEFAULT false,
    forked_from UUID REFERENCES projects(id) ON DELETE SET NULL,
    thumbnail_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);
CREATE INDEX projects_org_idx ON projects(org_id);
CREATE INDEX projects_updated_idx ON projects(updated_at DESC);

-- Simulations
CREATE TABLE simulations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    solver_type TEXT NOT NULL DEFAULT 'fdtd',  -- fdtd | fem | mom (MVP: fdtd only)
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | meshing | running | completed | failed | cancelled
    execution_mode TEXT NOT NULL DEFAULT 'wasm',  -- wasm | server
    parameters JSONB NOT NULL DEFAULT '{}',  -- Frequency range, excitations, boundary conditions, mesh refinement
    mesh_cell_count BIGINT NOT NULL DEFAULT 0,
    results_url TEXT,  -- S3 URL for field data (HDF5), S-parameters (JSON)
    results_summary JSONB,  -- S-parameters, resonant frequencies, peak gain, efficiency
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

-- Server Simulation Jobs (for GPU-accelerated server execution)
CREATE TABLE simulation_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    simulation_id UUID NOT NULL REFERENCES simulations(id) ON DELETE CASCADE,
    worker_id TEXT,
    gpu_type TEXT,  -- p4d | p5 | g5 (AWS instance types)
    cores_allocated INTEGER DEFAULT 1,
    gpu_memory_gb INTEGER DEFAULT 0,
    priority INTEGER DEFAULT 0,  -- Higher = more urgent
    progress_pct REAL DEFAULT 0.0,
    timestep_current BIGINT DEFAULT 0,
    timestep_total BIGINT DEFAULT 0,
    convergence_data JSONB DEFAULT '[]',  -- [{timestep, max_E_field, max_H_field}]
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX jobs_sim_idx ON simulation_jobs(simulation_id);
CREATE INDEX jobs_worker_idx ON simulation_jobs(worker_id);

-- Materials Database
CREATE TABLE materials (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL UNIQUE,
    category TEXT NOT NULL,  -- dielectric | metal | magnetic | pec | pmc
    permittivity_real DOUBLE PRECISION,  -- εr (real part)
    permittivity_imag DOUBLE PRECISION,  -- εr (imaginary part, from tanδ)
    permeability_real DOUBLE PRECISION,  -- μr (real part)
    permeability_imag DOUBLE PRECISION,  -- μr (imaginary part)
    conductivity DOUBLE PRECISION,  -- σ (S/m), for metals
    loss_tangent DOUBLE PRECISION,  -- tanδ, for dielectrics
    frequency_ghz DOUBLE PRECISION,  -- Frequency at which properties are specified
    description TEXT,
    datasheet_url TEXT,
    is_builtin BOOLEAN DEFAULT true,
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX materials_category_idx ON materials(category);
CREATE INDEX materials_name_trgm_idx ON materials USING gin(name gin_trgm_ops);

-- Antenna Templates
CREATE TABLE antenna_templates (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    category TEXT NOT NULL,  -- patch | dipole | horn | helix | vivaldi | slot | pifa | monopole
    description TEXT,
    frequency_range_min_ghz DOUBLE PRECISION,
    frequency_range_max_ghz DOUBLE PRECISION,
    typical_gain_dbi DOUBLE PRECISION,
    typical_bandwidth_pct DOUBLE PRECISION,
    geometry_function TEXT NOT NULL,  -- Python/Rust function name that generates geometry
    parameters JSONB NOT NULL,  -- [{name, description, min, max, default, unit}]
    thumbnail_url TEXT,
    is_builtin BOOLEAN DEFAULT true,
    download_count INTEGER DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX templates_category_idx ON antenna_templates(category);

-- Optimization Campaigns
CREATE TABLE optimization_campaigns (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    algorithm TEXT NOT NULL,  -- genetic | particle_swarm | cma_es
    objectives JSONB NOT NULL,  -- [{metric: 'gain', target: 10, weight: 1.0}]
    constraints JSONB NOT NULL,  -- [{metric: 'vswr', max: 2.0, frequency_ghz: 5.8}]
    parameters JSONB NOT NULL,  -- [{name: 'patch_length', min: 20, max: 50, unit: 'mm'}]
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | running | completed | failed
    generation_current INTEGER DEFAULT 0,
    generation_total INTEGER DEFAULT 50,
    population_size INTEGER DEFAULT 20,
    best_design JSONB,  -- Best parameter set found so far
    best_fitness DOUBLE PRECISION,
    results_url TEXT,  -- S3 URL for full Pareto front or optimization history
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX campaigns_project_idx ON optimization_campaigns(project_id);
CREATE INDEX campaigns_status_idx ON optimization_campaigns(status);

-- Field Visualizations (saved 3D views)
CREATE TABLE field_visualizations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    simulation_id UUID NOT NULL REFERENCES simulations(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    field_type TEXT NOT NULL,  -- e_field | h_field | j_field | power_density | sar
    timestep INTEGER,  -- Timestep index for transient, NULL for frequency-domain
    frequency_ghz DOUBLE PRECISION,  -- Frequency for harmonic fields
    slice_plane JSONB,  -- {axis: 'x'|'y'|'z', position: 0.05} for 2D slices
    color_map TEXT DEFAULT 'jet',  -- jet | viridis | plasma | inferno
    scale_min DOUBLE PRECISION,
    scale_max DOUBLE PRECISION,
    camera_state JSONB,  -- Three.js camera position, target, zoom for saved views
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX viz_sim_idx ON field_visualizations(simulation_id);

-- Usage Tracking (for billing enforcement)
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,  -- compute_minutes | mesh_cells | storage_gb
    quantity REAL NOT NULL,
    period_start DATE NOT NULL,
    period_end DATE NOT NULL,
    metadata JSONB DEFAULT '{}',
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
    pub geometry_data: serde_json::Value,
    pub mesh_url: Option<String>,
    pub mesh_stats: serde_json::Value,
    pub settings: serde_json::Value,
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
    pub solver_type: String,
    pub status: String,
    pub execution_mode: String,
    pub parameters: serde_json::Value,
    pub mesh_cell_count: i64,
    pub results_url: Option<String>,
    pub results_summary: Option<serde_json::Value>,
    pub error_message: Option<String>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub wall_time_ms: Option<i32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct SimulationJob {
    pub id: Uuid,
    pub simulation_id: Uuid,
    pub worker_id: Option<String>,
    pub gpu_type: Option<String>,
    pub cores_allocated: i32,
    pub gpu_memory_gb: i32,
    pub priority: i32,
    pub progress_pct: f32,
    pub timestep_current: i64,
    pub timestep_total: i64,
    pub convergence_data: serde_json::Value,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Material {
    pub id: Uuid,
    pub name: String,
    pub category: String,
    pub permittivity_real: Option<f64>,
    pub permittivity_imag: Option<f64>,
    pub permeability_real: Option<f64>,
    pub permeability_imag: Option<f64>,
    pub conductivity: Option<f64>,
    pub loss_tangent: Option<f64>,
    pub frequency_ghz: Option<f64>,
    pub description: Option<String>,
    pub datasheet_url: Option<String>,
    pub is_builtin: bool,
    pub created_by: Option<Uuid>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct AntennaTemplate {
    pub id: Uuid,
    pub name: String,
    pub category: String,
    pub description: Option<String>,
    pub frequency_range_min_ghz: Option<f64>,
    pub frequency_range_max_ghz: Option<f64>,
    pub typical_gain_dbi: Option<f64>,
    pub typical_bandwidth_pct: Option<f64>,
    pub geometry_function: String,
    pub parameters: serde_json::Value,
    pub thumbnail_url: Option<String>,
    pub is_builtin: bool,
    pub download_count: i32,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct SimulationParams {
    pub frequency_ghz_min: f64,
    pub frequency_ghz_max: f64,
    pub frequency_points: u32,  // For broadband FDTD
    pub excitations: Vec<Excitation>,
    pub boundaries: BoundaryConditions,
    pub mesh_refinement: MeshRefinement,
    pub duration_ns: Option<f64>,  // For time-domain FDTD
}

#[derive(Debug, Deserialize, Serialize)]
pub struct Excitation {
    pub excitation_type: String,  // port | plane_wave | dipole_source | gaussian_beam
    pub position: [f64; 3],  // (x, y, z) in meters
    pub direction: [f64; 3],  // Unit vector
    pub polarization: String,  // linear_x | linear_y | circular_left | circular_right
    pub impedance_ohms: f64,  // For port excitations
    pub waveform: String,  // gaussian_pulse | sinusoid | custom
}

#[derive(Debug, Deserialize, Serialize)]
pub struct BoundaryConditions {
    pub boundary_type: String,  // pml | pec | pmc | periodic
    pub pml_layers: u32,  // Number of PML layers (typically 8-12)
    pub pml_sigma_max: f64,  // PML conductivity profile parameter
}

#[derive(Debug, Deserialize, Serialize)]
pub struct MeshRefinement {
    pub max_cell_size_m: f64,  // Maximum cell size in meters
    pub min_cell_size_m: f64,  // Minimum cell size near geometry
    pub cells_per_wavelength: f64,  // Typical 10-20 for FDTD accuracy
    pub adaptive: bool,  // Octree adaptive refinement
}

#[derive(Debug, Deserialize, Serialize)]
pub struct OptimizationCampaign {
    pub algorithm: String,
    pub objectives: Vec<OptimizationObjective>,
    pub constraints: Vec<OptimizationConstraint>,
    pub parameters: Vec<OptimizationParameter>,
    pub generation_total: i32,
    pub population_size: i32,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct OptimizationObjective {
    pub metric: String,  // gain | efficiency | vswr | bandwidth | axial_ratio
    pub target: Option<f64>,
    pub weight: f64,
    pub minimize: bool,  // false = maximize
}

#[derive(Debug, Deserialize, Serialize)]
pub struct OptimizationConstraint {
    pub metric: String,
    pub min: Option<f64>,
    pub max: Option<f64>,
    pub frequency_ghz: Option<f64>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct OptimizationParameter {
    pub name: String,
    pub min: f64,
    pub max: f64,
    pub unit: String,
}
```

---

## FDTD Solver Architecture Deep-Dive

### Yee Grid and Update Equations

EMField's FDTD solver implements the **Yee algorithm**, the standard finite-difference solution to Maxwell's equations. The electromagnetic fields are discretized on a staggered 3D Cartesian grid (the Yee grid):

```
E-field components (Ex, Ey, Ez) and H-field components (Hx, Hy, Hz)
are offset by half a cell in space and time:

        Ez(i,j,k+1/2)
             ^
             |
    Ey(i,j+1/2,k) --> Ex(i+1/2,j,k)
```

**Maxwell's curl equations** in differential form:
```
∇ × E = -μ ∂H/∂t
∇ × H = ε ∂E/∂t + σE
```

**Discretized Yee update equations** (3D, for Ex component):
```
Ex^{n+1/2}(i+1/2,j,k) = Ca(i+1/2,j,k) · Ex^{n-1/2}(i+1/2,j,k)
    + Cb(i+1/2,j,k) · [
        (Hz^n(i+1/2,j+1/2,k) - Hz^n(i+1/2,j-1/2,k)) / Δy
      - (Hy^n(i+1/2,j,k+1/2) - Hy^n(i+1/2,j,k-1/2)) / Δz
    ]

where:
Ca = (2ε - σΔt) / (2ε + σΔt)
Cb = 2Δt / (2ε + σΔt)
```

Similar equations apply for Ey, Ez, Hx, Hy, Hz. The algorithm alternates between updating E-fields (at timesteps n+1/2) and H-fields (at timesteps n).

**Stability condition (Courant-Friedrichs-Lewy):**
```
Δt ≤ 1 / (c · √(1/Δx² + 1/Δy² + 1/Δz²))

where c = speed of light in the medium
```

For a uniform grid with Δx = Δy = Δz = Δ, this simplifies to:
```
Δt ≤ Δ / (c√3)
```

This constraint ensures numerical stability but requires small timesteps for fine meshes.

### Perfectly Matched Layer (PML) Absorbing Boundaries

To simulate open-space radiation (antennas, scattering), we need absorbing boundary conditions that prevent reflections from the computational domain edges. EMField uses **Convolutional PML (CPML)**, the state-of-the-art absorbing boundary.

PML is an artificial lossy material that absorbs outgoing waves with near-zero reflection (typically <-60 dB) regardless of angle or frequency. The CPML formulation uses auxiliary variables to accumulate field history:

```
Ex_pml^{n+1/2} = b_y · Ex_pml^{n-1/2} + a_y · (Hz^{n+1/2} - Hz^{n-1/2}) / Δy

Ex^{n+1/2} += Cb · Ex_pml^{n+1/2}

where a_y, b_y are CPML coefficients computed from σ_y(y) conductivity profile
```

PML is applied in 8-12 grid layers at each boundary face. The conductivity σ increases polynomially from the PML inner edge to the outer edge:

```
σ(d) = σ_max · (d / d_pml)^m

where d = distance from PML start, d_pml = PML thickness, m = 3 or 4
```

### Client/Server Split (WASM Threshold)

```
Geometry uploaded → Mesh generated → Cell count extracted
    │
    ├── <1M cells → WASM solver (browser)
    │   ├── Instant startup, no network latency
    │   ├── Results displayed immediately
    │   ├── Suitable for: single-element antennas, small microwave components
    │   └── No server cost
    │
    └── ≥1M cells → Server GPU solver (CUDA)
        ├── Job queued via Redis
        ├── Worker picks up, runs GPU-accelerated FDTD
        ├── Progress streamed via WebSocket
        ├── Suitable for: antenna arrays, full vehicle EMC, large RCS
        └── Results stored in S3 (HDF5), URL returned
```

The 1M-cell threshold was chosen because:
- WASM FDTD handles 1M cells × 10K timesteps in ~30-60 seconds on modern hardware
- 1M cells covers: patch antennas at <6 GHz (0.5λ domain ≈ 50mm, 0.5mm cells → 100³ = 1M cells), horn antennas (5cm aperture, 0.5mm cells → 100³), small microstrip circuits
- Above 1M cells: 5G mmWave arrays (28/39 GHz), phased arrays with 16+ elements, automotive EMC (full vehicle body) → need GPU acceleration

---

## Architecture Deep-Dives

### 1. FDTD Solver Core (Rust — shared between WASM and native)

The core FDTD engine that updates E and H fields on the Yee grid. This code compiles to both native (server) and WASM (browser) targets.

```rust
// fdtd-core/src/grid.rs

use ndarray::{Array3, Zip};

/// Yee grid for 3D FDTD simulation
/// E and H field components are staggered in space
pub struct YeeGrid {
    pub nx: usize,
    pub ny: usize,
    pub nz: usize,
    pub dx: f64,
    pub dy: f64,
    pub dz: f64,
    pub dt: f64,

    // E-field components (V/m)
    pub ex: Array3<f64>,  // Ex(i+1/2, j, k)
    pub ey: Array3<f64>,  // Ey(i, j+1/2, k)
    pub ez: Array3<f64>,  // Ez(i, j, k+1/2)

    // H-field components (A/m)
    pub hx: Array3<f64>,  // Hx(i, j+1/2, k+1/2)
    pub hy: Array3<f64>,  // Hy(i+1/2, j, k+1/2)
    pub hz: Array3<f64>,  // Hz(i+1/2, j+1/2, k)

    // Material properties (cell-centered)
    pub epsilon: Array3<f64>,  // Relative permittivity εr
    pub mu: Array3<f64>,       // Relative permeability μr
    pub sigma: Array3<f64>,    // Conductivity σ (S/m)

    // Update coefficients (precomputed for speed)
    pub ca_ex: Array3<f64>,
    pub cb_ex: Array3<f64>,
    // ... similar for Ey, Ez, Hx, Hy, Hz
}

impl YeeGrid {
    /// Create a new Yee grid with given dimensions and cell sizes
    pub fn new(nx: usize, ny: usize, nz: usize, dx: f64, dy: f64, dz: f64) -> Self {
        // Compute stable timestep (Courant limit)
        let c0 = 2.998e8;  // Speed of light (m/s)
        let dt = 0.99 / (c0 * (1.0/(dx*dx) + 1.0/(dy*dy) + 1.0/(dz*dz)).sqrt());

        let shape = (nx, ny, nz);
        Self {
            nx, ny, nz, dx, dy, dz, dt,
            ex: Array3::zeros(shape),
            ey: Array3::zeros(shape),
            ez: Array3::zeros(shape),
            hx: Array3::zeros(shape),
            hy: Array3::zeros(shape),
            hz: Array3::zeros(shape),
            epsilon: Array3::from_elem(shape, 1.0),
            mu: Array3::from_elem(shape, 1.0),
            sigma: Array3::zeros(shape),
            ca_ex: Array3::zeros(shape),
            cb_ex: Array3::zeros(shape),
        }
    }

    /// Precompute update coefficients from material properties
    pub fn compute_coefficients(&mut self) {
        let eps0 = 8.854e-12;  // Vacuum permittivity (F/m)

        Zip::from(&mut self.ca_ex)
            .and(&self.epsilon)
            .and(&self.sigma)
            .for_each(|ca, &eps, &sig| {
                let eps_abs = eps0 * eps;
                *ca = (2.0 * eps_abs - sig * self.dt) / (2.0 * eps_abs + sig * self.dt);
            });

        Zip::from(&mut self.cb_ex)
            .and(&self.epsilon)
            .and(&self.sigma)
            .for_each(|cb, &eps, &sig| {
                let eps_abs = eps0 * eps;
                *cb = (2.0 * self.dt) / (2.0 * eps_abs + sig * self.dt);
            });

        // Similar for Ey, Ez, Hx, Hy, Hz coefficients
    }

    /// Update E-field components (one timestep)
    pub fn update_e_fields(&mut self) {
        let inv_dy = 1.0 / self.dy;
        let inv_dz = 1.0 / self.dz;

        // Update Ex from Hz and Hy
        for i in 0..self.nx-1 {
            for j in 1..self.ny-1 {
                for k in 1..self.nz-1 {
                    let curl_h =
                        (self.hz[[i,j,k]] - self.hz[[i,j-1,k]]) * inv_dy
                      - (self.hy[[i,j,k]] - self.hy[[i,j,k-1]]) * inv_dz;

                    self.ex[[i,j,k]] = self.ca_ex[[i,j,k]] * self.ex[[i,j,k]]
                                     + self.cb_ex[[i,j,k]] * curl_h;
                }
            }
        }

        // Update Ey from Hx and Hz
        let inv_dx = 1.0 / self.dx;
        for i in 1..self.nx-1 {
            for j in 0..self.ny-1 {
                for k in 1..self.nz-1 {
                    let curl_h =
                        (self.hx[[i,j,k]] - self.hx[[i,j,k-1]]) * inv_dz
                      - (self.hz[[i,j,k]] - self.hz[[i-1,j,k]]) * inv_dx;

                    self.ey[[i,j,k]] = self.ca_ex[[i,j,k]] * self.ey[[i,j,k]]
                                     + self.cb_ex[[i,j,k]] * curl_h;
                }
            }
        }

        // Update Ez from Hy and Hx
        for i in 1..self.nx-1 {
            for j in 1..self.ny-1 {
                for k in 0..self.nz-1 {
                    let curl_h =
                        (self.hy[[i,j,k]] - self.hy[[i-1,j,k]]) * inv_dx
                      - (self.hx[[i,j,k]] - self.hx[[i,j-1,k]]) * inv_dy;

                    self.ez[[i,j,k]] = self.ca_ex[[i,j,k]] * self.ez[[i,j,k]]
                                     + self.cb_ex[[i,j,k]] * curl_h;
                }
            }
        }
    }

    /// Update H-field components (one timestep)
    pub fn update_h_fields(&mut self) {
        let mu0 = 1.2566e-6;  // Vacuum permeability (H/m)
        let inv_dx = 1.0 / self.dx;
        let inv_dy = 1.0 / self.dy;
        let inv_dz = 1.0 / self.dz;

        // Update Hx from Ey and Ez
        for i in 0..self.nx {
            for j in 0..self.ny-1 {
                for k in 0..self.nz-1 {
                    let curl_e =
                        (self.ey[[i,j,k+1]] - self.ey[[i,j,k]]) * inv_dz
                      - (self.ez[[i,j+1,k]] - self.ez[[i,j,k]]) * inv_dy;

                    let ch = self.dt / (mu0 * self.mu[[i,j,k]]);
                    self.hx[[i,j,k]] += ch * curl_e;
                }
            }
        }

        // Update Hy from Ez and Ex
        for i in 0..self.nx-1 {
            for j in 0..self.ny {
                for k in 0..self.nz-1 {
                    let curl_e =
                        (self.ez[[i+1,j,k]] - self.ez[[i,j,k]]) * inv_dx
                      - (self.ex[[i,j,k+1]] - self.ex[[i,j,k]]) * inv_dz;

                    let ch = self.dt / (mu0 * self.mu[[i,j,k]]);
                    self.hy[[i,j,k]] += ch * curl_e;
                }
            }
        }

        // Update Hz from Ex and Ey
        for i in 0..self.nx-1 {
            for j in 0..self.ny-1 {
                for k in 0..self.nz {
                    let curl_e =
                        (self.ex[[i,j+1,k]] - self.ex[[i,j,k]]) * inv_dy
                      - (self.ey[[i+1,j,k]] - self.ey[[i,j,k]]) * inv_dx;

                    let ch = self.dt / (mu0 * self.mu[[i,j,k]]);
                    self.hz[[i,j,k]] += ch * curl_e;
                }
            }
        }
    }

    /// Run FDTD simulation for a given number of timesteps
    pub fn run(&mut self, timesteps: usize, excitation: &mut dyn Excitation) -> FdtdResult {
        let mut result = FdtdResult::new(timesteps);

        for n in 0..timesteps {
            // Update H-fields (at timestep n)
            self.update_h_fields();

            // Update E-fields (at timestep n+1/2)
            self.update_e_fields();

            // Apply excitation (e.g., feed a port, inject plane wave)
            excitation.apply(self, n);

            // Record fields at probe points or boundaries
            if n % 10 == 0 {
                result.record_fields(n, self);
            }
        }

        result
    }
}

/// Trait for field excitations (ports, plane waves, sources)
pub trait Excitation {
    fn apply(&mut self, grid: &mut YeeGrid, timestep: usize);
}

/// Gaussian pulse excitation at a point
pub struct GaussianPulseSource {
    pub i: usize,
    pub j: usize,
    pub k: usize,
    pub component: FieldComponent,  // Ex | Ey | Ez
    pub t0: f64,  // Pulse center time
    pub tau: f64, // Pulse width
}

pub enum FieldComponent {
    Ex, Ey, Ez, Hx, Hy, Hz,
}

impl Excitation for GaussianPulseSource {
    fn apply(&mut self, grid: &mut YeeGrid, timestep: usize) {
        let t = timestep as f64 * grid.dt;
        let amplitude = (-((t - self.t0) / self.tau).powi(2)).exp();

        match self.component {
            FieldComponent::Ez => {
                grid.ez[[self.i, self.j, self.k]] += amplitude;
            }
            // Similar for other components
            _ => {}
        }
    }
}

#[derive(Debug)]
pub struct FdtdResult {
    pub timesteps: usize,
    pub field_snapshots: Vec<FieldSnapshot>,
}

impl FdtdResult {
    pub fn new(timesteps: usize) -> Self {
        Self {
            timesteps,
            field_snapshots: Vec::new(),
        }
    }

    pub fn record_fields(&mut self, timestep: usize, grid: &YeeGrid) {
        // Record a subset of field data (e.g., center plane or probe points)
        // Full 3D field data is too large to store every timestep
        self.field_snapshots.push(FieldSnapshot {
            timestep,
            // ... store probe values or 2D slices
        });
    }
}

#[derive(Debug)]
pub struct FieldSnapshot {
    pub timestep: usize,
    // ... field data
}
```

### 2. Simulation API Handler (Rust/Axum)

The primary endpoint receives a simulation request, generates the mesh, decides between WASM and server execution, and for server execution enqueues a GPU job.

```rust
// src/api/handlers/simulation.rs

use axum::{
    extract::{Path, State, Json},
    http::StatusCode,
    response::IntoResponse,
};
use uuid::Uuid;
use crate::{
    db::models::{Simulation, SimulationParams},
    meshing::octree_mesher::generate_mesh,
    state::AppState,
    auth::Claims,
    error::ApiError,
};

#[derive(serde::Deserialize)]
pub struct CreateSimulationRequest {
    pub frequency_ghz_min: f64,
    pub frequency_ghz_max: f64,
    pub frequency_points: u32,
    pub excitations: Vec<crate::db::models::Excitation>,
    pub boundaries: crate::db::models::BoundaryConditions,
    pub mesh_refinement: crate::db::models::MeshRefinement,
}

pub async fn create_simulation(
    State(state): State<AppState>,
    claims: Claims,
    Path(project_id): Path<Uuid>,
    Json(req): Json<CreateSimulationRequest>,
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

    // 2. Generate mesh from geometry
    let geometry = parse_geometry(&project.geometry_data)?;
    let mesh = generate_mesh(
        &geometry,
        req.mesh_refinement.max_cell_size_m,
        req.mesh_refinement.min_cell_size_m,
        req.mesh_refinement.adaptive,
    )?;

    let cell_count = mesh.cell_count();

    // 3. Check plan limits
    let user = sqlx::query_as!(
        crate::db::models::User,
        "SELECT * FROM users WHERE id = $1",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    if user.plan == "free" && cell_count > 100_000 {
        return Err(ApiError::PlanLimit(
            "Free plan supports meshes up to 100K cells. Upgrade to Pro for unlimited."
        ));
    }

    // 4. Determine execution mode
    let execution_mode = if cell_count <= 1_000_000 {
        "wasm"
    } else {
        "server"
    };

    // 5. Upload mesh to S3
    let mesh_data = mesh.to_hdf5()?;
    let mesh_key = format!("meshes/{}/{}.h5", project_id, Uuid::new_v4());
    state.s3.put_object()
        .bucket(&state.config.s3_bucket)
        .key(&mesh_key)
        .body(mesh_data.into())
        .content_type("application/x-hdf5")
        .send()
        .await?;
    let mesh_url = format!("s3://{}/{}", state.config.s3_bucket, mesh_key);

    // 6. Create simulation record
    let sim = sqlx::query_as!(
        Simulation,
        r#"INSERT INTO simulations
            (project_id, user_id, solver_type, status, execution_mode,
             parameters, mesh_cell_count)
        VALUES ($1, $2, 'fdtd', $3, $4, $5, $6)
        RETURNING *"#,
        project_id,
        claims.user_id,
        if execution_mode == "wasm" { "completed" } else { "pending" },
        execution_mode,
        serde_json::to_value(&req)?,
        cell_count as i64,
    )
    .fetch_one(&state.db)
    .await?;

    // 7. For server execution, enqueue GPU job
    if execution_mode == "server" {
        let job = sqlx::query_as!(
            crate::db::models::SimulationJob,
            r#"INSERT INTO simulation_jobs (simulation_id, priority, gpu_type)
            VALUES ($1, $2, 'p5') RETURNING *"#,
            sim.id,
            if user.plan == "advanced" { 10 } else { 0 },
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

pub async fn get_simulation(
    State(state): State<AppState>,
    claims: Claims,
    Path((project_id, sim_id)): Path<(Uuid, Uuid)>,
) -> Result<Json<Simulation>, ApiError> {
    let sim = sqlx::query_as!(
        Simulation,
        "SELECT s.* FROM simulations s
         JOIN projects p ON s.project_id = p.id
         WHERE s.id = $1 AND s.project_id = $2
         AND (p.owner_id = $3 OR p.org_id IN (
             SELECT org_id FROM org_members WHERE user_id = $3
         ))",
        sim_id,
        project_id,
        claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Simulation not found"))?;

    Ok(Json(sim))
}

fn parse_geometry(geometry_data: &serde_json::Value) -> Result<Geometry, ApiError> {
    // Parse CSG tree from JSON
    // ...
    todo!()
}
```

### 3. 3D Geometry Editor Component (React + Three.js)

The frontend geometry editor that allows CSG modeling, CAD import, and material assignment.

```typescript
// frontend/src/components/GeometryEditor/GeometryEditor.tsx

import { Canvas } from '@react-three/fiber';
import { OrbitControls, Grid, GizmoHelper, GizmoViewport } from '@react-three/drei';
import { useGeometryStore } from '../../stores/geometryStore';
import { GeometryPalette } from './GeometryPalette';
import { GeometryTree } from './GeometryTree';
import { MaterialPanel } from './MaterialPanel';
import { MeshPreview } from './MeshPreview';
import { CSGOperations } from './CSGOperations';

export function GeometryEditor() {
  const { primitives, selectedPrimitive, setSelectedPrimitive } = useGeometryStore();

  return (
    <div className="flex h-full">
      {/* Left sidebar: Primitives palette and CSG operations */}
      <div className="w-64 bg-gray-800 border-r border-gray-700 p-4">
        <GeometryPalette />
        <CSGOperations />
        <GeometryTree />
      </div>

      {/* Center: 3D viewport */}
      <div className="flex-1 relative">
        <Canvas camera={{ position: [5, 5, 5], fov: 50 }}>
          <ambientLight intensity={0.5} />
          <directionalLight position={[10, 10, 10]} intensity={0.8} />
          <Grid
            args={[20, 20]}
            cellSize={0.01}
            cellThickness={0.5}
            sectionSize={0.1}
            fadeDistance={30}
          />

          {/* Render all geometry primitives */}
          {primitives.map((prim) => (
            <GeometryPrimitive
              key={prim.id}
              primitive={prim}
              selected={prim.id === selectedPrimitive}
              onClick={() => setSelectedPrimitive(prim.id)}
            />
          ))}

          <OrbitControls makeDefault />
          <GizmoHelper alignment="bottom-right" margin={[80, 80]}>
            <GizmoViewport />
          </GizmoHelper>
        </Canvas>

        {/* Toolbar */}
        <div className="absolute top-4 left-4 flex gap-2">
          <button className="btn-icon" title="Box">
            <BoxIcon />
          </button>
          <button className="btn-icon" title="Cylinder">
            <CylinderIcon />
          </button>
          <button className="btn-icon" title="Sphere">
            <SphereIcon />
          </button>
          <button className="btn-icon" title="Import CAD">
            <ImportIcon />
          </button>
        </div>
      </div>

      {/* Right sidebar: Material assignment and mesh settings */}
      <div className="w-80 bg-gray-800 border-l border-gray-700 p-4">
        {selectedPrimitive ? (
          <>
            <MaterialPanel primitiveId={selectedPrimitive} />
            <MeshPreview primitiveId={selectedPrimitive} />
          </>
        ) : (
          <div className="text-gray-400 text-sm">
            Select a geometry primitive to edit materials and mesh settings.
          </div>
        )}
      </div>
    </div>
  );
}

interface GeometryPrimitiveProps {
  primitive: Primitive;
  selected: boolean;
  onClick: () => void;
}

function GeometryPrimitive({ primitive, selected, onClick }: GeometryPrimitiveProps) {
  const color = selected ? '#4ade80' : primitive.material.color || '#94a3b8';

  switch (primitive.type) {
    case 'box':
      return (
        <mesh
          position={primitive.position}
          rotation={primitive.rotation}
          onClick={onClick}
        >
          <boxGeometry args={[primitive.width, primitive.height, primitive.depth]} />
          <meshStandardMaterial color={color} opacity={0.8} transparent />
        </mesh>
      );

    case 'cylinder':
      return (
        <mesh
          position={primitive.position}
          rotation={primitive.rotation}
          onClick={onClick}
        >
          <cylinderGeometry args={[primitive.radius, primitive.radius, primitive.height, 32]} />
          <meshStandardMaterial color={color} opacity={0.8} transparent />
        </mesh>
      );

    case 'sphere':
      return (
        <mesh
          position={primitive.position}
          onClick={onClick}
        >
          <sphereGeometry args={[primitive.radius, 32, 32]} />
          <meshStandardMaterial color={color} opacity={0.8} transparent />
        </mesh>
      );

    default:
      return null;
  }
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–5)

**Day 1: Project initialization and Rust backend scaffold**
```bash
cargo init emfield-api
cd emfield-api
cargo add axum tokio serde serde_json sqlx uuid chrono tower-http jsonwebtoken bcrypt
cargo add --features postgres,runtime-tokio-rustls,uuid,chrono,json sqlx
cargo add aws-sdk-s3 redis ndarray hdf5
```
- `src/main.rs` — Axum app with CORS, tracing, graceful shutdown
- `src/config.rs` — Environment-based configuration (DATABASE_URL, JWT_SECRET, S3_BUCKET, REDIS_URL)
- `src/state.rs` — AppState struct (PgPool, Redis, S3Client)
- `src/error.rs` — ApiError enum with IntoResponse
- `Dockerfile` — Multi-stage build (builder + runtime)
- `docker-compose.yml` — PostgreSQL, Redis, MinIO (S3-compatible)

**Day 2: Database schema and migrations**
- `migrations/001_initial.sql` — All tables: users, organizations, org_members, projects, simulations, simulation_jobs, materials, antenna_templates, optimization_campaigns, field_visualizations, usage_records
- `src/db/mod.rs` — Database pool initialization
- `src/db/models.rs` — All SQLx structs with FromRow derives
- Run `sqlx migrate run` to apply schema
- Seed script for materials database (air, FR4, Rogers RO4003C, copper, PEC, 50+ common materials)

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

**Day 5: Materials database API**
- `src/api/handlers/materials.rs` — List materials, search materials, get material by ID, create custom material
- Seed 200+ materials: common dielectrics (FR4, Rogers, Arlon, Taconic, PTFE), metals (copper, aluminum, gold, silver), magnetic materials (ferrite), PEC/PMC
- Material search: filter by category, permittivity range, loss tangent, frequency
- Material property interpolation for frequency-dependent εr and tanδ
- Tests: material search, custom material upload (Pro+ plan gated)

### Phase 2 — FDTD Solver Core — Prototype (Days 6–14)

**Day 6: Yee grid and basic field updates**
- `fdtd-core/` — New Rust workspace member (shared between native and WASM)
- `fdtd-core/src/grid.rs` — YeeGrid struct with 3D arrays for Ex, Ey, Ez, Hx, Hy, Hz
- `fdtd-core/src/materials.rs` — Material property assignment to grid cells
- Precompute update coefficients (Ca, Cb) from ε, μ, σ
- Implement `update_e_fields()` and `update_h_fields()` with Yee stencil
- Unit tests: uniform plane wave in free space (verify E/H = η₀ = 377Ω)

**Day 7: Courant stability and timestep calculation**
- Implement CFL stability check: `Δt ≤ Δ / (c√3)` for 3D
- Automatic timestep computation from cell sizes
- Non-uniform grid support: variable Δx, Δy, Δz per cell (for octree meshing)
- Tests: verify stability for uniform and non-uniform grids

**Day 8: PML absorbing boundaries**
- `fdtd-core/src/pml.rs` — Convolutional PML (CPML) implementation
- CPML auxiliary variables (ψ arrays) for E and H field history
- Polynomial conductivity grading: σ(d) = σ_max · (d/d_pml)^m
- Apply PML in 8-layer regions at all 6 domain boundaries
- Tests: plane wave incident on PML, measure reflection coefficient (<-60 dB)

**Day 9: Field excitations — ports and sources**
- `fdtd-core/src/excitation.rs` — Excitation trait and implementations
- GaussianPulseSource: broadband Gaussian pulse for time-domain excitation
- SinusoidalSource: single-frequency continuous wave
- PlaneWaveSource: TFSF (Total-Field/Scattered-Field) plane wave injection
- Tests: Gaussian pulse spectrum (FFT), plane wave propagation

**Day 10: Port boundary conditions and S-parameter extraction**
- `fdtd-core/src/port.rs` — Waveguide port implementation with mode injection
- Voltage and current recording at port reference planes
- S-parameter computation: S11 = Vrefl/Vinc, S21 = Vtrans/Vinc
- Broadband S-parameters from time-domain to frequency-domain (FFT)
- Tests: two-port microstrip line (verify S11 ≈ 0, S21 ≈ 1 at matched impedance)

**Day 11: Far-field radiation pattern calculation**
- `fdtd-core/src/farfield.rs` — Near-field to far-field transformation (NF2FF)
- Record tangential E and H fields on a Huygens surface enclosing antenna
- Apply equivalence principle: surface currents J_s = n × H, M_s = -n × E
- Compute radiated fields at far-field observation points via radiation integrals
- Tests: half-wave dipole (verify doughnut pattern, 2.15 dBi gain)

**Day 12: Material models and dispersion**
- Frequency-dependent permittivity: Debye, Lorentz, or Drude models
- Dispersive material implementation via auxiliary differential equations (ADE)
- Conductor modeling: finite conductivity σ or surface impedance boundary condition (SIBC)
- Tests: skin-depth validation in copper at 1 GHz, 10 GHz

**Day 13: Result data structures and HDF5 export**
- `fdtd-core/src/results.rs` — FdtdResult struct for field data, S-parameters, patterns
- HDF5 export: store 3D field arrays with compression (gzip level 6)
- Chunked storage: enable efficient slice/timestep extraction
- Metadata: simulation parameters, mesh stats, convergence info
- Tests: write and read HDF5, verify data integrity

**Day 14: FDTD validation benchmarks**
- Benchmark 1: Free-space wavelength — run 10 GHz plane wave, measure λ = c/f = 30mm
- Benchmark 2: Dielectric wavelength — εr=2.2 (FR4), verify λ = λ₀/√εr = 20.2mm
- Benchmark 3: Half-wave dipole — verify resonance at L ≈ λ/2, gain ≈ 2.15 dBi
- Benchmark 4: Microstrip transmission line — Z₀ = 50Ω, verify S11 < -20dB
- Automated test suite with <1% error tolerance

### Phase 3 — Meshing Engine (Days 15–19)

**Day 15: Octree spatial subdivision**
- `meshing/src/octree.rs` — Octree data structure for adaptive grid refinement
- Recursive octree subdivision based on geometry proximity and feature size
- Leaf cells become FDTD grid cells
- Balance constraint: neighboring cells differ by at most 2:1 refinement ratio
- Tests: octree generation for box, sphere, verify cell count and balance

**Day 16: Geometry representation and CSG**
- `meshing/src/geometry.rs` — Primitive types: Box, Cylinder, Sphere, Cone
- CSG tree: Union, Intersection, Difference operations
- Ray-casting for inside/outside tests: shoot ray from point, count intersections
- Signed distance function (SDF) evaluation for primitives
- Tests: CSG operations (box - cylinder), ray-casting accuracy

**Day 17: Octree-based hexahedral meshing**
- `meshing/src/hex_mesher.rs` — Convert octree to structured hexahedral mesh
- Material assignment: query CSG tree at cell centers
- Interface refinement: force small cells at material boundaries
- Export mesh to HDF5: cell coordinates, material indices, connectivity
- Tests: mesh sphere (verify surface cell refinement), mesh patch antenna

**Day 18: CAD import via OpenCascade**
- `meshing/src/cad_import.rs` — STEP/IGES import via OpenCascade bindings
- Convert STEP B-rep to triangulated surface mesh (tessellation)
- Voxelize triangulated surface onto octree grid
- Material assignment from CAD assembly structure
- Tests: import STEP file (simple box), verify geometry match

**Day 19: Mesh quality metrics and validation**
- Cell size distribution: min, max, mean, median
- Aspect ratio check: flag cells with high anisotropy (Δx/Δy > 10)
- Mesh convergence study: halve cell size, compare S11 change (<0.1 dB)
- Mesh preview generation: extract surface triangles for WebGL rendering
- Tests: mesh quality for patch antenna, horn antenna

### Phase 4 — WASM Build + GPU Solver (Days 20–26)

**Day 20: WASM solver compilation**
- `fdtd-wasm/` — New workspace member for WASM target
- `fdtd-wasm/Cargo.toml` — wasm-bindgen, wasm-bindgen-futures, console_log
- `fdtd-wasm/src/lib.rs` — WASM entry points: `run_fdtd()`, `extract_sparams()`, `compute_farfield()`
- `wasm-pack build --target web --release` pipeline
- `wasm-opt -Oz` for size optimization (target <5MB gzipped)
- JavaScript wrapper: `FdtdSolver` class that loads WASM and provides async API

**Day 21: WASM progress streaming via SharedArrayBuffer**
- Use SharedArrayBuffer for lock-free progress updates from WASM to main thread
- WASM writes timestep progress to SAB, main thread reads and updates UI
- Fallback: postMessage for browsers without SAB (Safari)
- Tests: run 100K timesteps, verify progress updates every 1000 timesteps

**Day 22: CUDA GPU FDTD kernel**
- `fdtd-gpu/cuda/fdtd_kernels.cu` — CUDA kernels for E and H field updates
- Thread mapping: one thread per grid cell, 3D block/grid layout
- Shared memory optimization: load adjacent cells into shared memory for stencil
- Benchmark: 10M cells × 10K timesteps, measure throughput (cell-updates/sec)
- Target: >1 billion cell-updates/sec on NVIDIA A100

**Day 23: CUDA PML and excitation kernels**
- `fdtd-gpu/cuda/pml_kernels.cu` — CPML auxiliary variable updates on GPU
- `fdtd-gpu/cuda/excitation_kernels.cu` — Apply port excitations and sources
- Multi-GPU support: domain decomposition, halo exchange via NCCL
- Tests: single-GPU vs multi-GPU (2× A100), verify field agreement and speedup

**Day 24: Rust-CUDA bindings and orchestration**
- `fdtd-gpu/src/lib.rs` — Rust FFI bindings to CUDA kernels
- Memory management: allocate device memory, copy mesh and materials to GPU
- Kernel launch: grid/block sizing, stream-based async execution
- Result download: copy S-parameters and field data back to host
- Tests: Rust → CUDA round-trip, verify results match CPU FDTD

**Day 25: Server-side simulation worker with GPU**
- `src/workers/simulation_worker.rs` — Redis job consumer, runs GPU FDTD
- Job assignment: match job to worker with requested GPU type (p5, p4d, g5)
- Progress streaming: worker publishes timestep progress → Redis pub/sub → WebSocket to client
- Error handling: OOM detection, NaN detection, timeout handling
- S3 result upload: HDF5 field data, JSON S-parameters and pattern data

**Day 26: WebSocket for live simulation progress**
- `src/api/ws/mod.rs` — Axum WebSocket upgrade handler
- `src/api/ws/simulation_progress.rs` — Subscribe to simulation progress channel
- Client receives: `{ progress_pct, timestep_current, timestep_total, max_e_field }`
- Connection management: heartbeat ping/pong, automatic cleanup on disconnect
- Frontend: `src/hooks/useSimulationProgress.ts` — React hook for WebSocket subscription

### Phase 5 — Frontend 3D Editor + Visualization (Days 27–33)

**Day 27: Frontend scaffold and 3D editor foundation**
```bash
npm create vite@latest frontend -- --template react-ts
cd frontend
npm i zustand @tanstack/react-query axios three @react-three/fiber @react-three/drei
```
- `src/App.tsx` — Router, auth context, layout
- `src/stores/projectStore.ts` — Zustand store for project state
- `src/stores/geometryStore.ts` — Geometry state (primitives, CSG tree, materials)
- `src/components/GeometryEditor/Canvas.tsx` — Three.js canvas with OrbitControls
- `src/components/GeometryEditor/Grid.tsx` — Reference grid (10cm × 10cm)

**Day 28: Geometry primitives and CSG operations**
- `src/components/GeometryEditor/GeometryPalette.tsx` — Primitive library sidebar (Box, Cylinder, Sphere, Cone, custom mesh)
- `src/components/GeometryEditor/GeometryPrimitive.tsx` — Render primitive with Three.js meshes
- `src/components/GeometryEditor/CSGOperations.tsx` — Union, Intersection, Difference buttons
- `src/lib/csg.ts` — Client-side CSG preview using three-bvh-csg library
- Drag-and-drop primitive placement, transform gizmo (translate, rotate, scale)
- Property panel: dimensions, position, rotation, material assignment

**Day 29: Material assignment and visualization**
- `src/components/GeometryEditor/MaterialPanel.tsx` — Material selector with search and filter
- Material preview: show color-coded materials (dielectrics = blue, metals = gray, PEC = silver)
- Material database fetch: query materials API with pagination
- Custom material creation (Pro+ plan): specify εr, tanδ, σ, μr
- Visual feedback: highlight selected primitive, color by material category

**Day 30: CAD import interface**
- `src/components/GeometryEditor/CADImport.tsx` — File upload dialog for STEP/STL/DXF
- Parse STEP in backend (OpenCascade), return surface mesh or voxel representation
- Preview imported geometry in 3D viewport
- Material assignment to imported CAD parts
- Tests: import STEP (horn antenna), verify geometry match and meshing

**Day 31: 3D field visualization (volume rendering)**
- `src/components/FieldViewer/FieldViewer.tsx` — WebGL volume rendering for E-field, H-field, power density
- `src/lib/volumeRenderer.ts` — Custom fragment shader for ray-marching through 3D field data
- Color maps: jet, viridis, plasma, turbo with configurable min/max scale
- Slice planes: extract 2D XY/YZ/XZ planes at arbitrary positions
- Opacity control: alpha blending for semi-transparent volume visualization
- Tests: render 100³ field grid, verify frame rate >30 fps

**Day 32: Radiation pattern visualization**
- `src/components/PatternViewer/PatternViewer.tsx` — 3D radiation pattern (sphere mesh colored by gain)
- Polar plot: 2D gain vs. angle for E-plane and H-plane cuts
- Rectangular plot: gain (dBi) vs. theta/phi with 3dB beamwidth annotation
- Pattern metrics panel: peak gain, half-power beamwidth, front-to-back ratio, sidelobe level
- Interactive rotation: click-drag 3D pattern, highlight specific angles
- Tests: half-wave dipole pattern (verify doughnut shape, 2.15 dBi)

**Day 33: S-parameter and VSWR plots**
- `src/components/ResultsViewer/SparamPlot.tsx` — S11/S21 magnitude and phase vs. frequency
- Smith chart: `src/components/ResultsViewer/SmithChart.tsx` — plot impedance trajectory
- VSWR plot: compute VSWR = (1 + |Γ|) / (1 - |Γ|), highlight VSWR < 2 bandwidth
- Marker cursors: click to place markers, display frequency and S-parameter values
- Export data: CSV download for S-parameters, patterns, VSWR
- Tests: microstrip patch (verify resonance, impedance match)

### Phase 6 — Antenna Templates + Optimization (Days 34–38)

**Day 34: Antenna template database**
- Seed 5 antenna templates: microstrip patch, half-wave dipole, PIFA, pyramidal horn, Vivaldi
- `src/api/handlers/templates.rs` — List templates, get template, instantiate template
- Template instantiation: call geometry generation function with user parameters
- Parameter validation: enforce min/max ranges, physical constraints
- Tests: instantiate patch antenna (length, width, substrate), verify geometry

**Day 35: Parametric antenna generation**
- `src/templates/patch.rs` — Microstrip patch antenna generator: compute L, W from frequency and εr
- `src/templates/dipole.rs` — Half-wave dipole with balun and feed
- `src/templates/pifa.rs` — PIFA (Planar Inverted-F Antenna) for mobile devices
- `src/templates/horn.rs` — Pyramidal horn with aperture dimensions
- `src/templates/vivaldi.rs` — Vivaldi exponential taper antenna for UWB
- Frontend: template wizard with parameter inputs and 3D preview

**Day 36: Optimization service (Python FastAPI)**
- `optimization-service/` — New Python FastAPI service
- `optimization-service/src/genetic.py` — Genetic algorithm implementation using `deap` library
- Fitness evaluation: submit FDTD simulation for each individual, collect S-parameters and gain
- Multi-objective optimization: NSGA-II for gain/bandwidth/size trade-offs
- Parallelization: evaluate population in parallel (20-50 simulations/generation)
- Tests: optimize patch length/width for max gain at 2.4 GHz

**Day 37: Optimization API integration**
- `src/api/handlers/optimization.rs` — Create campaign, get campaign status, get results
- Campaign workflow: user specifies objectives/constraints → service generates population → submit simulations → evolve → repeat
- Progress tracking: generation number, best fitness, Pareto front
- Results visualization: plot fitness history, Pareto front scatter plot
- Tests: end-to-end optimization (10 generations, 20 individuals)

**Day 38: Optimization UI**
- `src/components/Optimization/CampaignWizard.tsx` — Step-by-step wizard: select parameters, set objectives, set constraints
- `src/components/Optimization/ProgressPanel.tsx` — Live progress: generation, best fitness, convergence plot
- `src/components/Optimization/ParetoFront.tsx` — Scatter plot of Pareto-optimal designs (gain vs. bandwidth)
- One-click apply: select design from Pareto front, load geometry into editor
- Tests: create campaign, run 5 generations, verify Pareto front

### Phase 7 — Billing + Testing + Deployment (Days 39–42)

**Day 39: Stripe integration**
- `src/api/handlers/billing.rs` — Create checkout session, create customer portal session, get subscription status
- `src/api/handlers/webhooks/stripe.rs` — Handle subscription events: `checkout.session.completed`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.payment_failed`
- Plan mapping: Free (100K cells), Pro ($199/mo, 1M cells, 1000 compute minutes/month), Advanced ($449/mo, unlimited, API access)
- Usage tracking: record compute minutes per simulation, aggregate monthly usage, enforce limits

**Day 40: Usage tracking and plan enforcement**
- `src/middleware/plan_limits.rs` — Middleware checking plan limits before simulation execution
- `src/services/usage.rs` — Track compute minutes per billing period
- Usage record insertion after each simulation completes
- Usage dashboard: current period usage, historical usage, approaching-limit warnings
- Tests: verify limit enforcement, usage recording accuracy

**Day 41: End-to-end testing**
- Full workflow test: create project → draw geometry → assign materials → create simulation → view results
- WASM solver test: load in headless browser, run simulation, verify results match native
- GPU solver test: submit large job, monitor progress, verify S-parameters and pattern
- Optimization test: run 10-generation campaign, verify Pareto front
- Load testing: 50 concurrent users, 100 simulations/hour
- Performance benchmarks: WASM (100K cells × 10K timesteps < 60s), GPU (1M cells × 20K timesteps < 5min)

**Day 42: Production deployment**
- ECS Fargate: deploy API (4 tasks, auto-scaling)
- EC2 p4d: deploy GPU workers (1 instance, spot, auto-scaling on queue depth)
- RDS PostgreSQL 16: Multi-AZ, automated backups
- ElastiCache Redis 7: cluster mode for job queue and caching
- S3 buckets: meshes (7-day lifecycle), results (90-day), materials (indefinite), CDN (WASM)
- CloudFront: distribute frontend and WASM bundles
- Monitoring: Prometheus + Grafana + Sentry
- DNS: Route53, SSL certificates
- Soft launch: 50 beta users, monitor performance and errors

---

## Solver Validation Suite

### Benchmark 1: Half-Wave Dipole at 1 GHz

**Setup:** 2 arms × 7.5cm, 1mm radius copper, 50Ω feed, 1 GHz, 40K cells (λ/40), 10 PML layers, 10K timesteps

**Expected Results:**
- Input impedance: 73 + j42.5 Ω (theory)
- Resonance: 1.00 ± 0.02 GHz
- S11 at resonance: ≤ -10 dB
- Peak gain: 2.15 ± 0.3 dBi
- Pattern: Donut shape, nulls at θ=0°/180°, max at θ=90°
- 3dB beamwidth: ~78° in E-plane

**Validation:** Compare to ANSYS HFSS or NEC2 (free MoM solver)

### Benchmark 2: Microstrip Patch at 2.4 GHz

**Setup:** FR4 (ε_r=4.4, tan(δ)=0.02, h=1.6mm), 38×30mm copper patch, coax probe feed, 2-3 GHz sweep, 65K cells (λ/18), 15K timesteps

**Expected Results:**
- Resonance: 2.40 ± 0.05 GHz (design: f_r = c/(2L√ε_eff))
- -10dB bandwidth: 80-100 MHz (3.3-4.2%)
- Input impedance: ~50 + j0 Ω
- S11 at resonance: < -20 dB
- Peak gain: 6.5 ± 0.5 dBi (broadside)
- Efficiency: 85-90% (FR4 losses)
- HPBW: E-plane ~70°, H-plane ~80°

**Validation:** Measured prototype or HFSS/CST

### Benchmark 3: Rectangular Waveguide WR-90 (X-band)

**Setup:** Copper walls, a=22.86mm, b=10.16mm, L=100mm, TE10 mode at 9.5 GHz, 80K cells, 12K timesteps

**Expected Results:**
- Cutoff frequency: f_c = c/(2a) = 6.56 GHz (TE10)
- S11 at 9.5 GHz: < -30 dB
- S21 at 9.5 GHz: ≈ -0.1 to -0.3 dB (copper loss)
- Propagation constant: β = 2π/λ_g ≈ 37.8mm at 9.5 GHz
- Field: E_y sinusoidal across width a

**Validation:** Analytical TE10 dispersion, skin-depth loss calculation

### Benchmark 4: Pyramidal Horn at 10 GHz

**Setup:** 50×40mm aperture, 80mm length, 20° flare, WR-90 feed, 10 GHz, 120K cells, 18K timesteps

**Expected Results:**
- Gain: 15.0 ± 0.8 dBi (G = 4πA_eff/λ²)
- Directivity: 16.5 dBi (70% aperture efficiency)
- HPBW: E-plane ~25°, H-plane ~30°
- Sidelobe level: < -20 dB
- S11: < -15 dB
- Efficiency: 90-95%

**Validation:** Commercial X-band horn datasheets

### Benchmark 5: PIFA for 5G at 28 GHz

**Setup:** 4×3mm top plate, h=1.2mm, shorting pin 0.5mm dia, Rogers RO4003C (ε_r=3.55, tan(δ)=0.0027), 24-32 GHz sweep, 90K cells (λ/71), 20K timesteps

**Expected Results:**
- Resonance: 28.0 ± 0.5 GHz
- -10dB bandwidth: 1.5-2.0 GHz (5.4-7.1%)
- Peak gain: 3.5 ± 0.5 dBi
- Efficiency: 75-85%
- Input impedance: 50 ± 5 Ω
- S11 at resonance: < -15 dB

**Validation:** CST MWS or measured mmWave prototypes

---

## Verification Checklist

### Solver Accuracy
- [ ] Dipole: gain within 0.3 dB, impedance within 5%
- [ ] Patch: resonance within 2%, bandwidth within 15%
- [ ] Waveguide: S21 loss matches analytical skin-depth
- [ ] Horn: gain within 0.8 dB, HPBW within 3°
- [ ] PIFA: efficiency >75%, resonance within 2%
- [ ] Courant stability: no divergence at dt = 0.99 CFL
- [ ] PML: reflection < -40 dB normal incidence

### Meshing Quality
- [ ] Octree refinement: cell size ≤ λ/20
- [ ] Material boundaries: cells align within 0.01λ
- [ ] CSG boolean ops: no gaps/overlaps
- [ ] Mesh size: matches analytical V/(λ/20)³ within 20%
- [ ] Neighbor connectivity: all cells have valid neighbors

### Performance
- [ ] WASM: 100K cells × 10K timesteps in <60s (M2 Max / RTX 4060)
- [ ] GPU: 1M cells × 20K timesteps in <5min (A100)
- [ ] Multi-GPU: 4-GPU runs 3.5-3.8× faster than 1-GPU
- [ ] S-parameter FFT: 10K samples → 1K freq in <1s
- [ ] Near-to-far: 100K surface samples → 3D pattern in <10s

### API Functionality
- [ ] Simulation creation: mesh + enqueue <5s
- [ ] WebSocket progress: updates every 100 timesteps, <500ms latency
- [ ] S3 upload: 500MB HDF5 in <30s
- [ ] Presigned URLs: 1-hour expiry works
- [ ] Plan enforcement: free blocks >100K cells

### UI/UX
- [ ] 3D editor: CSG renders correctly, no glitches
- [ ] Material assignment: colors match properties
- [ ] S-parameter plot: resonance markers, bandwidth annotations
- [ ] Radiation pattern 3D: interactive rotation, gain legend
- [ ] Field animation: 30 FPS playback, readable heatmap
- [ ] Templates: all 5 templates generate valid geometry, simulate successfully

### Billing & Auth
- [ ] Stripe checkout: updates users.plan immediately
- [ ] Webhooks: subscription cancel downgrades plan
- [ ] Compute credits: deducted on completion, blocks if insufficient
- [ ] OAuth: Google/GitHub flows complete
- [ ] JWT refresh: auto-refresh on 401

---

## Post-MVP Roadmap

### v1.1 — FEM Solver + Advanced Materials (Weeks 9-12)
- Frequency-domain FEM solver for resonant structures
- Tetrahedral mesh generation (Delaunay)
- Sparse matrix assembly, MUMPS/PETSc solver integration
- Frequency-dependent materials: Debye, Lorentz, Drude dispersive models
- Anisotropic materials: tensorial ε, μ
- Use cases: Dielectric resonator antennas, cavity filters, metamaterials

### v1.2 — MoM Solver + RCS Computation (Weeks 13-16)
- Method of Moments surface integral equation solver
- RWG basis functions on triangular surface mesh
- MLFMA (Multi-Level Fast Multipole) for O(N log N) complexity
- Radar cross-section: monostatic and bistatic
- PEC and lossy ground plane support
- Use cases: Aircraft RCS, ship radar signature, large reflector antennas

### v1.3 — Antenna Array Analysis + Beam Steering (Weeks 17-20)
- N-element array analysis with mutual coupling
- Active impedance calculation
- Beam steering: arbitrary phase/amplitude excitation
- Array factor visualization: element pattern × array factor
- Adaptive beamforming: null steering, sidelobe reduction
- Use cases: 5G beamforming, radar phased arrays, satellite comms

### v1.4 — EMC/EMI Analysis + PCB Import (Weeks 21-24)
- PCB layout import: Gerber, ODB++, IPC-2581
- Trace radiation analysis: identify hotspots
- Cable coupling: crosstalk, common-mode radiation
- Shielding effectiveness: enclosures with apertures, seams
- EMC standard pre-compliance: FCC Part 15, CISPR 32, MIL-STD-461G overlays
- Use cases: Pre-compliance EMC, reduce test lab failures by 80%

### v1.5 — Multi-Physics + Thermal Coupling (Weeks 25-28)
- Thermal simulation: heat dissipation from high-power antennas
- EM-thermal coupling: temperature-dependent dielectric properties
- Structural deformation: thermal expansion affects performance
- Power handling: predict breakdown voltage, arcing
- Use cases: Radar transmit antennas, satellite TWTAs, microwave ovens
