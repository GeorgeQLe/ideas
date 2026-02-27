# NanoDesign — Nanoelectronics and Quantum Transport Simulation

## Implementation Plan

**MVP Scope:** Browser-based atomic structure builder with drag-and-drop crystal structure assembly, material database (Si, Ge, GaAs, InAs, graphene, MoS2) with automatic lattice matching and dangling bond termination rendered via Three.js, custom quantum transport engine implementing Non-Equilibrium Green's Function (NEGF) method with recursive Green's function algorithm compiled to Rust for ballistic transport in devices ≤10,000 atoms and server-side execution with GPU acceleration (CUDA cuBLAS) for larger devices and self-consistent NEGF-Poisson, empirical tight-binding Hamiltonian generation (sp3d5s* for Si/Ge, sp3 for III-V, pz for graphene) with strain effects via Bir-Pikus deformation potentials, ballistic quantum transport solver with semi-infinite contact self-energies using Sancho-Rubio method, self-consistent NEGF-Poisson iteration for electrostatic coupling, I-V characteristic calculation with transfer and output curves rendered via WebGL, transmission spectrum T(E) visualization with channel decomposition, local density of states (LDOS) spatial maps with Three.js volumetric rendering using marching cubes, subthreshold swing SS and DIBL extraction for transistor metrics, 50+ pre-built device templates (Si nanowire FET, GAA nanosheet, FinFET, graphene ribbon FET, MoS2 transistor, tunnel FET) stored in S3 with PostgreSQL metadata, Python FastAPI service for DFT Hamiltonian import from Wannier90 format and SPICE model parameter extraction via least-squares fitting to quantum I-V data, three-tier Stripe billing (Academic $99/mo lab with 5 seats and 200 GPU-hours, Pro $399/mo with 500 GPU-hours and API access, Enterprise custom pricing with on-premise deployment).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| NEGF Solver | Rust (native + WASM) + CUDA | Recursive Green's function with GPU matrix operations via `cudarc` |
| WASM Compilation | `wasm-pack` + `wasm-bindgen` | Client-side ballistic solver for devices ≤10K atoms |
| Hamiltonian | Rust + Python 3.12 | Tight-binding in Rust, DFT import via Python FastAPI + PythTB/Wannier90 |
| Poisson Solver | Rust | 3D multigrid solver for electrostatics, conjugate gradient with AMG preconditioner |
| Python Services | FastAPI 0.109 | DFT Hamiltonian processing, SPICE extraction, variability Monte Carlo |
| Database | PostgreSQL 16 | Projects, users, simulations, materials, device templates |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth, bcrypt password hashing |
| Object Storage | AWS S3 | Hamiltonians, simulation results (I-V, LDOS, transmission), structure files |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| 3D Visualization | Three.js + React Three Fiber | Atomic structure, bond rendering, LDOS volumetric visualization |
| Waveform/I-V Plots | WebGL 2.0 (custom) | GPU-accelerated I-V curves, transmission spectra, Bode plots |
| Real-time | WebSocket (Axum) | Live simulation progress, convergence monitoring |
| Job Queue | Redis 7 + Tokio tasks | Server-side quantum transport job management, GPU resource allocation |
| GPU Compute | AWS p3/p4d instances | CUDA-accelerated NEGF for self-consistent simulations, large Hamiltonians |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | WASM bundle delivery, static frontend assets |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, error tracking, GPU utilization |
| CI/CD | GitHub Actions | WASM build pipeline, Rust tests, CUDA kernel compilation, Docker image push |

### Key Architecture Decisions

1. **Hybrid client/server NEGF solver with 10K-atom WASM threshold**: Devices with ≤10,000 atoms (covers 80%+ of nanowire, FinFET, and 2D material simulations in ballistic regime) run entirely in the browser via WASM, providing instant I-V curves with zero server cost. Devices exceeding 10K atoms or requiring self-consistent NEGF-Poisson with scattering are submitted to GPU-accelerated servers. The threshold is configurable per plan tier.

2. **Custom NEGF implementation in Rust+CUDA rather than wrapping KWANT or NanoTCAD**: Building a custom recursive Green's function solver in Rust with CUDA kernels gives full control over memory layout for GPU efficiency, WASM compilation for browser execution, and self-consistent iteration strategies. KWANT is Python-only and lacks self-consistent Poisson integration. NanoTCAD ViDES is MATLAB-based and not production-ready. Our Rust solver uses GPU-accelerated sparse matrix operations for the retarded Green's function computation and is memory-safe for WASM deployment.

3. **Three.js for atomic structure visualization with instanced rendering**: Three.js with React Three Fiber provides GPU-instanced rendering for 100K+ atoms (needed for 3D devices like GAA nanosheets) with interactive rotation, pan, and zoom. Each atom is rendered as a sphere with color-coded element type (Si=gray, Ge=purple, Ga=green, As=red, C=black, Mo=blue, S=yellow). Bonds are rendered as cylinders with automatic detection via neighbor distance thresholds. LDOS spatial maps use volumetric rendering with isosurface extraction via marching cubes.

4. **Tight-binding Hamiltonian generation in Rust with Python DFT import bridge**: Empirical tight-binding (sp3d5s* for Si/Ge, sp3 for III-V, pz for 2D materials) is implemented in Rust for performance and WASM compatibility. For advanced materials requiring DFT accuracy, a Python FastAPI microservice accepts Wannier90 `.hr` files (from Quantum ESPRESSO or VASP) and converts them to our sparse Hamiltonian format. This separation allows WASM to remain lightweight while supporting advanced users.

5. **S3 for Hamiltonian and result storage with PostgreSQL metadata catalog**: Device structure Hamiltonians (sparse CSR matrices, typically 1-100MB each) are stored in S3, while PostgreSQL holds searchable metadata (material, geometry, atom count, energy cutoff). This allows the library to scale to 10K+ templates without bloating the database while enabling fast parametric search via PostgreSQL indexes and full-text search.

6. **Recursive Green's Function (RGF) algorithm for O(N) scaling**: Standard NEGF requires inverting the full Hamiltonian matrix (O(N³) for N atoms), which is prohibitive for 3D devices with 50K+ atoms. RGF exploits the locality of quantum transport by computing Green's functions slice-by-slice in the transport direction, reducing complexity to O(N·M²) where M is the transverse cross-section size. This is implemented in Rust with CUDA kernels for GPU parallelization.

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
    plan TEXT NOT NULL DEFAULT 'free',  -- free | academic | pro | enterprise
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX users_email_idx ON users(email);
CREATE INDEX users_stripe_idx ON users(stripe_customer_id);

-- Organizations (for Academic/Enterprise plans)
CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    slug TEXT UNIQUE NOT NULL,
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    plan TEXT NOT NULL DEFAULT 'academic',
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    settings JSONB DEFAULT '{}',
    max_seats INTEGER DEFAULT 5,
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

-- Materials Database
CREATE TABLE materials (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT UNIQUE NOT NULL,  -- "Si", "GaAs", "graphene", "MoS2"
    chemical_formula TEXT NOT NULL,
    crystal_structure TEXT NOT NULL,  -- fcc, diamond, hexagonal, orthorhombic
    lattice_parameters JSONB NOT NULL,  -- {a, b, c, alpha, beta, gamma}
    basis_atoms JSONB NOT NULL,  -- [{element, position: [x, y, z]}]
    category TEXT NOT NULL,  -- semiconductor | 2d_material | insulator | metal
    bandgap_ev REAL,
    effective_mass JSONB,  -- {electron, hole} for semiconductor
    hamiltonian_type TEXT NOT NULL,  -- sp3d5s_star | sp3 | pz | dft
    tight_binding_params JSONB,  -- Slater-Koster parameters or null for DFT
    description TEXT,
    references TEXT[],
    is_builtin BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX materials_category_idx ON materials(category);
CREATE INDEX materials_name_trgm_idx ON materials USING gin(name gin_trgm_ops);

-- Device Templates
CREATE TABLE device_templates (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    category TEXT NOT NULL,  -- nanowire_fet | gaa_fet | finfet | tfet | graphene_fet | molecular_junction
    description TEXT,
    preview_image_url TEXT,
    geometry_type TEXT NOT NULL,  -- 1d | 2d | 3d
    material_stack JSONB NOT NULL,  -- [{material_id, thickness_nm, dopant_type, dopant_conc}]
    structure_params JSONB NOT NULL,  -- Gate length, channel width, fin height, etc.
    atom_count_estimate INTEGER,
    default_bias_conditions JSONB,  -- {Vgs_range, Vds_range, Vbs}
    tags TEXT[] DEFAULT '{}',
    is_verified BOOLEAN DEFAULT false,
    is_public BOOLEAN DEFAULT true,
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX templates_category_idx ON device_templates(category);
CREATE INDEX templates_tags_idx ON device_templates USING gin(tags);

-- Projects (device structure + simulation workspace)
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    device_data JSONB NOT NULL DEFAULT '{}',  -- Full atomic structure, geometry parameters
    template_id UUID REFERENCES device_templates(id),
    settings JSONB DEFAULT '{}',  -- Visualization settings, probe positions
    is_public BOOLEAN NOT NULL DEFAULT false,
    forked_from UUID REFERENCES projects(id) ON DELETE SET NULL,
    thumbnail_url TEXT,
    atom_count INTEGER DEFAULT 0,
    hamiltonian_url TEXT,  -- S3 URL to precomputed Hamiltonian if cached
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);
CREATE INDEX projects_org_idx ON projects(org_id);
CREATE INDEX projects_template_idx ON projects(template_id);
CREATE INDEX projects_updated_idx ON projects(updated_at DESC);

-- Simulation Runs
CREATE TABLE simulations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    simulation_type TEXT NOT NULL,  -- ballistic | scattering | self_consistent
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | running | completed | failed | cancelled
    execution_mode TEXT NOT NULL DEFAULT 'wasm',  -- wasm | server_cpu | server_gpu
    parameters JSONB NOT NULL DEFAULT '{}',  -- Bias voltages, temperature, scattering options
    atom_count INTEGER NOT NULL DEFAULT 0,
    results_url TEXT,  -- S3 URL for result data (I-V, transmission, LDOS)
    results_summary JSONB,  -- Quick-access summary (SS, DIBL, Vth, Ion, Ioff)
    error_message TEXT,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    wall_time_ms INTEGER,
    gpu_time_ms INTEGER,
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
    gpu_allocated BOOLEAN DEFAULT false,
    gpu_device_id INTEGER,
    cores_allocated INTEGER DEFAULT 4,
    memory_mb INTEGER DEFAULT 8192,
    priority INTEGER DEFAULT 0,  -- Higher = more urgent (Academic=5, Pro=10, Enterprise=20)
    progress_pct REAL DEFAULT 0.0,
    convergence_data JSONB DEFAULT '[]',  -- [{iteration, residual, max_delta_potential}]
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX jobs_sim_idx ON simulation_jobs(simulation_id);
CREATE INDEX jobs_worker_idx ON simulation_jobs(worker_id);
CREATE INDEX jobs_status_idx ON simulation_jobs(gpu_allocated, started_at);

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
CREATE INDEX usage_org_period_idx ON usage_records(org_id, period_start, period_end);

-- SPICE Model Extractions (calibrated compact models from quantum results)
CREATE TABLE spice_extractions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    model_type TEXT NOT NULL,  -- bsim_cmg | hspice_level73 | verilog_a
    extracted_parameters JSONB NOT NULL,
    calibration_error REAL,  -- RMS error vs. quantum I-V data
    spice_file_url TEXT,  -- S3 URL to .lib file
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX spice_project_idx ON spice_extractions(project_id);
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
    pub chemical_formula: String,
    pub crystal_structure: String,
    pub lattice_parameters: serde_json::Value,
    pub basis_atoms: serde_json::Value,
    pub category: String,
    pub bandgap_ev: Option<f32>,
    pub effective_mass: Option<serde_json::Value>,
    pub hamiltonian_type: String,
    pub tight_binding_params: Option<serde_json::Value>,
    pub description: Option<String>,
    pub references: Vec<String>,
    pub is_builtin: bool,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct DeviceTemplate {
    pub id: Uuid,
    pub name: String,
    pub category: String,
    pub description: Option<String>,
    pub preview_image_url: Option<String>,
    pub geometry_type: String,
    pub material_stack: serde_json::Value,
    pub structure_params: serde_json::Value,
    pub atom_count_estimate: Option<i32>,
    pub default_bias_conditions: Option<serde_json::Value>,
    pub tags: Vec<String>,
    pub is_verified: bool,
    pub is_public: bool,
    pub created_by: Option<Uuid>,
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
    pub device_data: serde_json::Value,
    pub template_id: Option<Uuid>,
    pub settings: serde_json::Value,
    pub is_public: bool,
    pub forked_from: Option<Uuid>,
    pub thumbnail_url: Option<String>,
    pub atom_count: i32,
    pub hamiltonian_url: Option<String>,
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
    pub parameters: serde_json::Value,
    pub atom_count: i32,
    pub results_url: Option<String>,
    pub results_summary: Option<serde_json::Value>,
    pub error_message: Option<String>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub wall_time_ms: Option<i32>,
    pub gpu_time_ms: Option<i32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct SimulationJob {
    pub id: Uuid,
    pub simulation_id: Uuid,
    pub worker_id: Option<String>,
    pub gpu_allocated: bool,
    pub gpu_device_id: Option<i32>,
    pub cores_allocated: i32,
    pub memory_mb: i32,
    pub priority: i32,
    pub progress_pct: f32,
    pub convergence_data: serde_json::Value,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct SimulationParams {
    pub simulation_type: SimulationType,
    pub vgs_range: Option<VoltageRange>,  // Gate voltage sweep
    pub vds_range: Option<VoltageRange>,  // Drain voltage sweep
    pub vbs: f64,  // Bulk/substrate voltage
    pub temperature: f64,  // Kelvin, default 300K
    pub scattering: Option<ScatteringParams>,
    pub self_consistent: bool,  // NEGF-Poisson self-consistency
    pub convergence_threshold: f64,  // Default 1e-4 for potential
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum SimulationType {
    Ballistic,           // No scattering, single bias point
    BallisticSweep,      // No scattering, I-V sweep
    Scattering,          // With phonon scattering
    SelfConsistent,      // Full NEGF-Poisson iteration
}

#[derive(Debug, Deserialize, Serialize)]
pub struct VoltageRange {
    pub start: f64,
    pub stop: f64,
    pub step: f64,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct ScatteringParams {
    pub acoustic_phonon: bool,
    pub optical_phonon: bool,
    pub surface_roughness: bool,
    pub phonon_energy_ev: Option<f64>,  // For optical phonons
    pub deformation_potential_ev: Option<f64>,  // For acoustic phonons
}
```

---

## Quantum Transport Solver Architecture

### Governing Equations and Discretization

NanoDesign's core solver implements the **Non-Equilibrium Green's Function (NEGF) formalism**, the standard method for quantum transport in nanoscale devices. For a device with Hamiltonian **H** (N×N matrix for N orbitals) connected to left (L) and right (R) semi-infinite contacts, the key quantities are:

**1. Retarded Green's Function:**

```
G^r(E) = [(E + iη)I - H - Σ_L^r(E) - Σ_R^r(E)]^(-1)
```

Where:
- **E**: Energy at which to compute transport
- **η**: Infinitesimal broadening (typically 1e-6 eV)
- **H**: Device Hamiltonian matrix from tight-binding or DFT
- **Σ_L^r, Σ_R^r**: Self-energies describing coupling to left and right contacts
- **I**: Identity matrix

**2. Contact Self-Energies (Sancho-Rubio Iterative Method):**

For semi-infinite periodic contacts, the self-energy is computed iteratively:

```
Σ_L^r(E) = τ_L^† · g_L^r(E) · τ_L

where g_L^r(E) is the surface Green's function of the semi-infinite left contact,
computed via Sancho-Rubio iteration until convergence.
```

**3. Transmission Function:**

```
T(E) = Trace[Γ_L(E) · G^r(E) · Γ_R(E) · G^a(E)]

where:
Γ_L/R(E) = i[Σ_L/R^r(E) - Σ_L/R^a(E)]  (broadening matrices)
G^a(E) = [G^r(E)]^†  (advanced Green's function)
```

**4. Current (Landauer-Büttiker Formula):**

```
I(V) = (2e/h) ∫ T(E) [f_L(E, μ_L) - f_R(E, μ_R)] dE

where:
f_L/R = Fermi-Dirac distribution at left/right contacts
μ_L = E_F (equilibrium Fermi level)
μ_R = E_F - eV (shifted by applied bias V)
```

**5. Self-Consistent NEGF-Poisson:**

For devices with electrostatic gating (MOSFETs, TFETs), the Hamiltonian depends on the electrostatic potential, which depends on the charge density, which depends on the Green's function. Iteration:

```
(a) Solve Poisson's equation:  ∇²φ = -ρ(r)/ε
    where ρ(r) = -e·n(r) + N_D - N_A  (electron density + dopants)

(b) Compute electron density from NEGF:
    n(r) = ∫ dE · A(r, E) · [f_L(E) + f_R(E)]/2
    where A(r, E) = Trace[G^<(r, E)]  (local spectral function)

(c) Update Hamiltonian on-site energies:  H_ii → H_ii - eφ(r_i)

(d) Repeat (a)-(c) until ||φ_new - φ_old|| < convergence_threshold
```

### Recursive Green's Function (RGF) Algorithm

Standard matrix inversion of (E·I - H - Σ) scales as O(N³), prohibitive for N > 10,000. RGF exploits the sparsity structure in the transport direction:

**Assume device is sliced in transport direction (z) into M layers:**

```
Layer 1 ← τ₁ → Layer 2 ← τ₂ → ... ← τₘ₋₁ → Layer M

H = [H₁   τ₁†   0    ...   0  ]
    [τ₁   H₂    τ₂†  ...   0  ]
    [0    τ₂    H₃   ...   0  ]
    [⋮    ⋮     ⋮    ⋱    ⋮  ]
    [0    0     0    ... H_M  ]
```

**Forward recursion** (compute Green's functions from left contact):

```
g₁ = [E·I - H₁ - Σ_L]^(-1)
for i = 2 to M:
    g_i = [E·I - H_i - τ_{i-1}·g_{i-1}·τ_{i-1}†]^(-1)
```

**Backward recursion** (incorporate right contact):

```
G_M = [E·I - H_M - Σ_R]^(-1)
for i = M-1 to 1:
    G_i = g_i + g_i·τ_i†·G_{i+1}·τ_i·g_i
```

Complexity: **O(M·N_slice³)** where N_slice is atoms per transverse slice (typically 100-500), much better than O(N³) for large M.

### Client/Server Split (WASM vs. GPU)

```
Device structure uploaded → Atom count extracted
    │
    ├── ≤10,000 atoms + ballistic → WASM solver (browser)
    │   ├── Instant Hamiltonian generation (tight-binding)
    │   ├── Ballistic NEGF (no self-consistency)
    │   ├── I-V displayed in <5 seconds
    │   └── No server cost
    │
    └── >10,000 atoms OR self-consistent → Server GPU solver
        ├── Job queued via Redis with priority
        ├── GPU worker: CUDA kernels for matrix ops
        ├── Self-consistent NEGF-Poisson iteration
        ├── Progress streamed via WebSocket
        └── Results (I-V, LDOS) stored in S3

Plan-based thresholds:
- Academic: ballistic only, ≤10K atoms WASM, ≤50K atoms server CPU
- Pro: scattering, ≤100K atoms server GPU, 500 GPU-hours/month
- Enterprise: unlimited, multi-GPU, on-premise option
```

### WASM Compilation Pipeline

```toml
# negf-wasm/Cargo.toml
[package]
name = "nanodesign-negf-wasm"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
wasm-bindgen = "0.2"
serde = { version = "1", features = ["derive"] }
serde-wasm-bindgen = "0.6"
js-sys = "0.3"
nalgebra = { version = "0.33", features = ["serde-serialize"] }
nalgebra-sparse = "0.10"
num-complex = "0.4"
log = "0.4"
console_log = "1"
rayon = "1.8"  # Parallelism in WASM via Web Workers

[profile.release]
opt-level = 3
lto = true
codegen-units = 1
```

---

## Architecture Deep-Dives

### 1. Hamiltonian Generation API Handler (Rust/Axum)

Generates tight-binding Hamiltonians from device structures, handles caching, and routes to Python service for DFT imports.

```rust
// src/api/handlers/hamiltonian.rs

use axum::{
    extract::{Path, State, Json},
    http::StatusCode,
    response::IntoResponse,
};
use uuid::Uuid;
use crate::{
    db::models::Project,
    hamiltonian::{
        tight_binding::generate_tb_hamiltonian,
        geometry::build_device_structure,
    },
    state::AppState,
    auth::Claims,
    error::ApiError,
};

#[derive(serde::Deserialize)]
pub struct GenerateHamiltonianRequest {
    pub force_regenerate: bool,  // Skip cache
}

pub async fn generate_hamiltonian(
    State(state): State<AppState>,
    claims: Claims,
    Path(project_id): Path<Uuid>,
    Json(req): Json<GenerateHamiltonianRequest>,
) -> Result<impl IntoResponse, ApiError> {
    // 1. Verify project ownership
    let project = sqlx::query_as!(
        Project,
        "SELECT * FROM projects WHERE id = $1 AND (owner_id = $2 OR org_id IN (
            SELECT org_id FROM org_members WHERE user_id = $2
        ))",
        project_id,
        claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Project not found"))?;

    // 2. Check for cached Hamiltonian
    if !req.force_regenerate && project.hamiltonian_url.is_some() {
        return Ok((StatusCode::OK, Json(serde_json::json!({
            "status": "cached",
            "hamiltonian_url": project.hamiltonian_url,
            "atom_count": project.atom_count,
        }))));
    }

    // 3. Build atomic structure from device_data
    let device = build_device_structure(&project.device_data)?;
    let atom_count = device.atoms.len();

    if atom_count > 200_000 {
        return Err(ApiError::ValidationError(
            "Device exceeds maximum 200,000 atoms. Simplify geometry."
        ));
    }

    // 4. Fetch materials for this device
    let material_ids: Vec<Uuid> = device.get_material_ids();
    let materials = sqlx::query_as!(
        crate::db::models::Material,
        "SELECT * FROM materials WHERE id = ANY($1)",
        &material_ids
    )
    .fetch_all(&state.db)
    .await?;

    // 5. Generate tight-binding Hamiltonian
    let hamiltonian = generate_tb_hamiltonian(&device, &materials)?;

    // 6. Serialize and upload to S3
    let s3_key = format!("hamiltonians/{}/{}.h5", project_id, uuid::Uuid::new_v4());
    let hamiltonian_bytes = hamiltonian.to_hdf5_bytes()?;

    state.s3.put_object()
        .bucket(&state.config.s3_bucket)
        .key(&s3_key)
        .body(hamiltonian_bytes.into())
        .content_type("application/x-hdf5")
        .send()
        .await?;

    let hamiltonian_url = format!("s3://{}/{}", state.config.s3_bucket, s3_key);

    // 7. Update project with Hamiltonian URL and atom count
    sqlx::query!(
        "UPDATE projects SET hamiltonian_url = $1, atom_count = $2, updated_at = NOW()
         WHERE id = $3",
        hamiltonian_url,
        atom_count as i32,
        project_id
    )
    .execute(&state.db)
    .await?;

    Ok((StatusCode::OK, Json(serde_json::json!({
        "status": "generated",
        "hamiltonian_url": hamiltonian_url,
        "atom_count": atom_count,
        "matrix_size": hamiltonian.size(),
        "sparsity": hamiltonian.sparsity(),
    }))))
}
```

### 2. Ballistic NEGF Solver Core (Rust — shared between WASM and native)

The core ballistic quantum transport solver using the recursive Green's function algorithm.

```rust
// negf-core/src/solver/ballistic.rs

use nalgebra_sparse::{CsrMatrix, CooMatrix};
use num_complex::Complex64;
use std::f64::consts::PI;

pub struct BallisticSolver {
    pub hamiltonian: CsrMatrix<Complex64>,  // Device Hamiltonian
    pub contact_left: ContactRegion,
    pub contact_right: ContactRegion,
    pub n_layers: usize,  // Layers in transport direction
    pub layer_size: usize,  // Orbitals per layer
}

pub struct ContactRegion {
    pub h_contact: CsrMatrix<Complex64>,  // Contact Hamiltonian (1 layer)
    pub tau: CsrMatrix<Complex64>,  // Inter-layer coupling
}

impl BallisticSolver {
    /// Compute transmission T(E) at given energy E
    pub fn compute_transmission(&self, energy: f64) -> Result<f64, SolverError> {
        let e = Complex64::new(energy, 1e-6);  // Add infinitesimal broadening

        // 1. Compute contact self-energies via Sancho-Rubio
        let sigma_left = self.contact_left.self_energy(e)?;
        let sigma_right = self.contact_right.self_energy(e)?;

        // 2. Compute broadening matrices Γ = i(Σ^r - Σ^a)
        let gamma_left = &sigma_left - &sigma_left.conjugate_transpose();
        let gamma_left = gamma_left.map(|z| z * Complex64::new(0.0, 1.0));

        let gamma_right = &sigma_right - &sigma_right.conjugate_transpose();
        let gamma_right = gamma_right.map(|z| z * Complex64::new(0.0, 1.0));

        // 3. Recursive Green's function algorithm
        let gr = self.recursive_greens_function(e, &sigma_left, &sigma_right)?;

        // 4. Compute transmission: T = Trace[Γ_L · G^r · Γ_R · G^a]
        let ga = gr.conjugate_transpose();
        let t_matrix = &gamma_left * &gr * &gamma_right * &ga;

        // Trace of transmission matrix
        let transmission = t_matrix.diagonal().iter()
            .map(|z| z.re)
            .sum::<f64>();

        Ok(transmission.max(0.0))  // Transmission is non-negative
    }

    /// Recursive Green's function (RGF) algorithm
    fn recursive_greens_function(
        &self,
        energy: Complex64,
        sigma_left: &CsrMatrix<Complex64>,
        sigma_right: &CsrMatrix<Complex64>,
    ) -> Result<CsrMatrix<Complex64>, SolverError> {
        let n = self.layer_size;
        let identity = CsrMatrix::identity(n);

        // Extract layer Hamiltonians and couplings
        let mut g_forward = Vec::with_capacity(self.n_layers);

        // Forward recursion from left contact
        let mut g = (&identity * energy - self.get_layer_hamiltonian(0)? - sigma_left)
            .inverse()?;
        g_forward.push(g.clone());

        for i in 1..self.n_layers {
            let h_i = self.get_layer_hamiltonian(i)?;
            let tau_prev = self.get_coupling(i - 1)?;

            // g_i = [E·I - H_i - τ_{i-1}·g_{i-1}·τ_{i-1}†]^(-1)
            let tau_dag = tau_prev.conjugate_transpose();
            let effective_h = h_i + &tau_prev * &g * &tau_dag;
            g = (&identity * energy - effective_h).inverse()?;
            g_forward.push(g.clone());
        }

        // Backward recursion incorporating right contact
        let last_layer = self.n_layers - 1;
        let mut gr = (&identity * energy
            - self.get_layer_hamiltonian(last_layer)?
            - sigma_right)
            .inverse()?;

        for i in (0..last_layer).rev() {
            let tau_i = self.get_coupling(i)?;
            let tau_dag = tau_i.conjugate_transpose();
            let g_i = &g_forward[i];

            // G_i = g_i + g_i·τ_i†·G_{i+1}·τ_i·g_i
            let update = g_i * &tau_dag * &gr * &tau_i * g_i;
            gr = g_i + &update;
        }

        Ok(gr)
    }

    /// Compute current I(V) using Landauer-Büttiker formula
    pub fn compute_current(
        &self,
        voltage: f64,
        temperature: f64,
        fermi_energy: f64,
    ) -> Result<f64, SolverError> {
        let kb = 8.617e-5;  // Boltzmann constant in eV/K
        let kt = kb * temperature;

        // Energy integration range and step
        let e_min = fermi_energy - 10.0 * kt;
        let e_max = fermi_energy + voltage + 10.0 * kt;
        let n_points = 200;
        let de = (e_max - e_min) / (n_points as f64);

        let mut current = 0.0;

        for i in 0..n_points {
            let energy = e_min + (i as f64 + 0.5) * de;
            let transmission = self.compute_transmission(energy)?;

            // Fermi-Dirac distributions
            let f_left = fermi_dirac(energy - fermi_energy, kt);
            let f_right = fermi_dirac(energy - fermi_energy - voltage, kt);

            current += transmission * (f_left - f_right) * de;
        }

        // Multiply by conductance quantum 2e²/h
        let g0 = 2.0 * 1.602e-19 * 1.602e-19 / 6.626e-34;  // Siemens
        current *= g0;

        Ok(current)
    }

    /// Extract Hamiltonian submatrix for layer i
    fn get_layer_hamiltonian(&self, layer: usize) -> Result<CsrMatrix<Complex64>, SolverError> {
        let start = layer * self.layer_size;
        let end = start + self.layer_size;
        Ok(self.hamiltonian.slice_rows_cols(start, end, start, end))
    }

    /// Extract coupling matrix between layer i and i+1
    fn get_coupling(&self, layer: usize) -> Result<CsrMatrix<Complex64>, SolverError> {
        let start_i = layer * self.layer_size;
        let start_j = (layer + 1) * self.layer_size;
        Ok(self.hamiltonian.slice_rows_cols(start_i, start_i + self.layer_size,
                                            start_j, start_j + self.layer_size))
    }
}

impl ContactRegion {
    /// Sancho-Rubio iterative method for semi-infinite contact self-energy
    pub fn self_energy(&self, energy: Complex64) -> Result<CsrMatrix<Complex64>, SolverError> {
        let n = self.h_contact.nrows();
        let identity = CsrMatrix::identity(n);
        let tol = 1e-10;
        let max_iter = 100;

        // Initialize
        let mut alpha = self.tau.clone();
        let mut beta = self.tau.conjugate_transpose();
        let mut eps_s = self.h_contact.clone();

        for iter in 0..max_iter {
            let g = (&identity * energy - &eps_s).inverse()?;
            let alpha_g = &alpha * &g;
            let beta_g = &beta * &g;

            let eps_new = &eps_s + &alpha_g * &beta + &beta_g * &alpha;
            let alpha_new = &alpha_g * &alpha;
            let beta_new = &beta_g * &beta;

            // Check convergence
            let delta = (&eps_new - &eps_s).norm();
            if delta < tol {
                return Ok((&identity * energy - &eps_new).inverse()? * &self.tau);
            }

            eps_s = eps_new;
            alpha = alpha_new;
            beta = beta_new;
        }

        Err(SolverError::Convergence(format!(
            "Sancho-Rubio did not converge after {} iterations", max_iter
        )))
    }
}

fn fermi_dirac(energy: f64, kt: f64) -> f64 {
    1.0 / (1.0 + (energy / kt).exp())
}

#[derive(Debug)]
pub enum SolverError {
    Convergence(String),
    Singular(String),
    InvalidDevice(String),
}
```

### 3. Three.js Atomic Structure Viewer (React + R3F)

Interactive 3D visualization of atomic structures with instanced rendering for performance.

```typescript
// frontend/src/components/AtomicViewer/AtomicStructure.tsx

import { useRef, useMemo } from 'react';
import { Canvas, useFrame } from '@react-three/fiber';
import { OrbitControls, PerspectiveCamera, Html } from '@react-three/drei';
import * as THREE from 'three';
import { useDeviceStore } from '../../stores/deviceStore';

interface Atom {
  position: [number, number, number];
  element: string;
  orbital_energy?: number;  // For LDOS coloring
}

interface Bond {
  atom1: number;
  atom2: number;
}

const ELEMENT_COLORS: Record<string, string> = {
  Si: '#8c8c8c',
  Ge: '#9966cc',
  Ga: '#66cc66',
  As: '#cc6666',
  In: '#6699cc',
  C: '#333333',
  Mo: '#5588aa',
  S: '#ffff00',
  H: '#ffffff',
};

const ELEMENT_RADII: Record<string, number> = {
  Si: 0.12, Ge: 0.13, Ga: 0.14, As: 0.12, In: 0.15,
  C: 0.08, Mo: 0.14, S: 0.10, H: 0.05,
};

export function AtomicStructure() {
  const { atoms, bonds, showBonds, ldosData, selectedAtomIndex } = useDeviceStore();

  return (
    <div className="w-full h-full bg-gray-950">
      <Canvas>
        <PerspectiveCamera makeDefault position={[5, 5, 10]} />
        <OrbitControls enableDamping dampingFactor={0.05} />

        <ambientLight intensity={0.4} />
        <directionalLight position={[10, 10, 5]} intensity={0.8} />
        <directionalLight position={[-10, -10, -5]} intensity={0.3} />

        <InstancedAtoms atoms={atoms} ldosData={ldosData} />
        {showBonds && <Bonds atoms={atoms} bonds={bonds} />}

        {selectedAtomIndex !== null && (
          <AtomLabel atom={atoms[selectedAtomIndex]} index={selectedAtomIndex} />
        )}

        <gridHelper args={[20, 20, '#444444', '#222222']} />
        <axesHelper args={[2]} />
      </Canvas>
    </div>
  );
}

// GPU-instanced rendering for efficient display of 100K+ atoms
function InstancedAtoms({ atoms, ldosData }: { atoms: Atom[], ldosData?: number[] }) {
  const meshRef = useRef<THREE.InstancedMesh>(null);

  // Group atoms by element for separate instanced meshes (same material/geometry)
  const atomsByElement = useMemo(() => {
    const groups: Record<string, Atom[]> = {};
    atoms.forEach(atom => {
      if (!groups[atom.element]) groups[atom.element] = [];
      groups[atom.element].push(atom);
    });
    return groups;
  }, [atoms]);

  return (
    <>
      {Object.entries(atomsByElement).map(([element, elementAtoms]) => (
        <InstancedElement
          key={element}
          element={element}
          atoms={elementAtoms}
          ldosData={ldosData}
        />
      ))}
    </>
  );
}

function InstancedElement({
  element,
  atoms,
  ldosData
}: {
  element: string,
  atoms: Atom[],
  ldosData?: number[]
}) {
  const meshRef = useRef<THREE.InstancedMesh>(null);
  const tempObject = useMemo(() => new THREE.Object3D(), []);
  const tempColor = useMemo(() => new THREE.Color(), []);

  const radius = ELEMENT_RADII[element] || 0.1;
  const baseColor = ELEMENT_COLORS[element] || '#888888';

  // Setup instance matrices and colors
  useMemo(() => {
    if (!meshRef.current) return;

    atoms.forEach((atom, i) => {
      tempObject.position.set(...atom.position);
      tempObject.updateMatrix();
      meshRef.current!.setMatrixAt(i, tempObject.matrix);

      // Color by LDOS if available, otherwise by element
      if (ldosData && ldosData[i] !== undefined) {
        const intensity = Math.min(Math.max(ldosData[i], 0), 1);
        tempColor.setHSL(0.6 - intensity * 0.6, 1.0, 0.3 + intensity * 0.4);
      } else {
        tempColor.set(baseColor);
      }
      meshRef.current!.setColorAt(i, tempColor);
    });

    meshRef.current.instanceMatrix.needsUpdate = true;
    if (meshRef.current.instanceColor) {
      meshRef.current.instanceColor.needsUpdate = true;
    }
  }, [atoms, ldosData, tempObject, tempColor, baseColor]);

  return (
    <instancedMesh ref={meshRef} args={[undefined, undefined, atoms.length]}>
      <sphereGeometry args={[radius, 16, 12]} />
      <meshPhongMaterial vertexColors metalness={0.3} roughness={0.7} />
    </instancedMesh>
  );
}

function Bonds({ atoms, bonds }: { atoms: Atom[], bonds: Bond[] }) {
  const geometry = useMemo(() => {
    const cylinderGeo = new THREE.CylinderGeometry(0.02, 0.02, 1, 8);
    cylinderGeo.translate(0, 0.5, 0);  // Move origin to one end
    return cylinderGeo;
  }, []);

  return (
    <>
      {bonds.map((bond, i) => {
        const pos1 = new THREE.Vector3(...atoms[bond.atom1].position);
        const pos2 = new THREE.Vector3(...atoms[bond.atom2].position);
        const direction = new THREE.Vector3().subVectors(pos2, pos1);
        const length = direction.length();

        const orientation = new THREE.Matrix4();
        orientation.lookAt(pos1, pos2, new THREE.Object3D().up);
        orientation.multiply(new THREE.Matrix4().makeRotationX(Math.PI / 2));

        return (
          <mesh key={i} position={pos1.toArray()} matrix={orientation}>
            <cylinderGeometry args={[0.02, 0.02, length, 8]} />
            <meshPhongMaterial color="#666666" />
          </mesh>
        );
      })}
    </>
  );
}

function AtomLabel({ atom, index }: { atom: Atom, index: number }) {
  return (
    <Html position={atom.position} distanceFactor={10}>
      <div className="bg-black bg-opacity-80 text-white text-xs px-2 py-1 rounded pointer-events-none">
        <div>Atom {index}</div>
        <div>{atom.element}</div>
        {atom.orbital_energy !== undefined && (
          <div>E: {atom.orbital_energy.toFixed(3)} eV</div>
        )}
      </div>
    </Html>
  );
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–4)

**Day 1: Project initialization and Rust backend scaffold**
```bash
cargo init nanodesign-api
cd nanodesign-api
cargo add axum tokio serde serde_json sqlx uuid chrono tower-http jsonwebtoken bcrypt
cargo add --features postgres,runtime-tokio-rustls,uuid,chrono,json sqlx
cargo add aws-sdk-s3 redis hdf5 ndarray
```
- `src/main.rs` — Axum app with CORS, tracing, graceful shutdown
- `src/config.rs` — Environment config (DATABASE_URL, JWT_SECRET, S3_BUCKET, REDIS_URL, CUDA_VISIBLE_DEVICES)
- `src/state.rs` — AppState struct (PgPool, Redis, S3Client, GPU device pool)
- `src/error.rs` — ApiError enum with IntoResponse, quantum-specific errors
- `Dockerfile` — Multi-stage build (CUDA base image for GPU workers, slim for API)
- `docker-compose.yml` — PostgreSQL, Redis, MinIO (S3-compatible), pgAdmin

**Day 2: Database schema and migrations**
- `migrations/001_initial.sql` — All tables: users, organizations, org_members, materials, device_templates, projects, simulations, simulation_jobs, usage_records, spice_extractions
- `src/db/mod.rs` — Database pool initialization with connection pooling (max 20 connections)
- `src/db/models.rs` — All SQLx structs with FromRow derives
- Run `sqlx migrate run` to apply schema
- Seed script: `scripts/seed_materials.sql` — Insert Si, Ge, GaAs, InAs, graphene, MoS2 with tight-binding parameters
- Seed script: `scripts/seed_templates.sql` — Insert 10 basic device templates (Si nanowire, FinFET, graphene ribbon)

**Day 3: Authentication system**
- `src/auth/mod.rs` — JWT token generation (HS256, 24h access + 7d refresh), validation middleware
- `src/auth/oauth.rs` — Google and GitHub OAuth 2.0 flow with PKCE
- `src/api/handlers/auth.rs` — Register (email + password), login, OAuth callback, refresh token, me, logout endpoints
- Password hashing with bcrypt (cost factor 12)
- Auth middleware: extract Claims from Authorization: Bearer header, verify signature, check expiration
- Integration tests: register → login → access protected endpoint → refresh → logout

**Day 4: User, org, and project CRUD**
- `src/api/handlers/users.rs` — Get profile, update profile, upload avatar (S3), delete account with cascade
- `src/api/handlers/orgs.rs` — Create org, invite member (send email), list members, update role, remove member
- `src/api/handlers/projects.rs` — Create project (from scratch or template), list projects (with pagination), get project, update device_data, delete, fork (deep copy), share (public link)
- `src/api/router.rs` — All route definitions with auth middleware, CORS, request logging
- Authorization checks: owner-only for delete, org members for read/write based on project.org_id
- Integration tests: project CRUD flow, org membership flow, authorization edge cases

### Phase 2 — Tight-Binding Hamiltonian Generation (Days 5–10)

**Day 5: Crystal structure and geometry primitives**
- `hamiltonian-core/` — New Rust workspace member (shared between native and WASM)
- `hamiltonian-core/src/crystal.rs` — CrystalStructure struct (lattice vectors, basis atoms, symmetry group)
- `hamiltonian-core/src/geometry.rs` — AtomicDevice struct (list of atoms with positions, elements, layer assignments)
- `hamiltonian-core/src/builder/nanowire.rs` — Build Si/Ge nanowire with [110] transport direction, hexagonal cross-section, hydrogen passivation
- `hamiltonian-core/src/builder/planar.rs` — Build planar MOSFET structure with gate oxide, substrate
- Unit tests: build Si nanowire 3nm diameter, verify atom count vs. analytical estimate

**Day 6: Tight-binding parameter database**
- `hamiltonian-core/src/tb_params/mod.rs` — Tight-binding parameter trait (Slater-Koster hopping integrals)
- `hamiltonian-core/src/tb_params/si_ge.rs` — sp3d5s* parameters for Si and Ge (Jancu 1998 parameters), spin-orbit coupling
- `hamiltonian-core/src/tb_params/iii_v.rs` — sp3 parameters for GaAs, InAs, InP (Jancu 2006)
- `hamiltonian-core/src/tb_params/graphene.rs` — pz nearest-neighbor and next-nearest-neighbor for graphene
- `hamiltonian-core/src/tb_params/mos2.rs` — d-orbital tight-binding for MoS2 (Rostami 2015 parameters)
- Parameter interpolation for alloys (e.g., Si₁₋ₓGeₓ via linear interpolation or Vegard's law)

**Day 7: Tight-binding Hamiltonian assembly**
- `hamiltonian-core/src/hamiltonian/builder.rs` — HamiltonianBuilder: iterate over atom pairs, compute distances, apply Slater-Koster angular transformations, populate sparse matrix
- `hamiltonian-core/src/hamiltonian/slater_koster.rs` — SK transformation matrices for sp3, sp3d5s*, pz orbitals
- Neighbor search: KD-tree for efficient nearest-neighbor finding (cutoff radius 5Å)
- Strain effects: modify hopping integrals via Bir-Pikus deformation potentials
- Output: CSR sparse matrix (nalgebra-sparse), Hermitian check
- Unit tests: Si dimer (2 atoms, 20 orbitals for sp3d5s*), verify Hamiltonian eigenvalues match expected bands

**Day 8: Band structure calculation and validation**
- `hamiltonian-core/src/band_structure.rs` — Compute E(k) dispersion by diagonalizing H(k) at k-points along high-symmetry paths
- Brillouin zone k-point generation: Γ-X-W-K-Γ for FCC, Γ-K-M-Γ for hexagonal
- Eigenvalue solver: LAPACK via `ndarray-linalg` for dense H(k), Lanczos for sparse
- Effective mass extraction: fit parabola to E(k) near band extrema
- Validation: Si bulk band structure → verify indirect bandgap at Γ-X (Eg ≈ 1.1 eV at 300K), effective masses (m*_e ≈ 0.26 m₀)
- Tests: graphene band structure → verify Dirac cone at K point, linear dispersion

**Day 9: Device Hamiltonian with open boundary conditions**
- `hamiltonian-core/src/device/mod.rs` — DeviceHamiltonian struct: central device + left/right contact regions
- `hamiltonian-core/src/device/partition.rs` — Partition atomic structure into contact/device/contact based on z-coordinate
- Contact region extraction: periodic in x-y (transverse), semi-infinite in z (transport)
- Layer-by-layer Hamiltonian structure for RGF: H = [H₁, τ₁; τ₁†, H₂, τ₂; ...; τₙ₋₁†, Hₙ]
- Identify layer size (orbitals per transverse slice) for RGF algorithm
- Unit tests: Si nanowire device → verify Hamiltonian block structure, layer sizes match cross-section

**Day 10: Hamiltonian caching and serialization**
- `hamiltonian-core/src/io/hdf5.rs` — Serialize DeviceHamiltonian to HDF5 format (CSR indices, data, layer structure, metadata)
- HDF5 structure: `/hamiltonian/data`, `/hamiltonian/indices`, `/hamiltonian/indptr`, `/device/atoms`, `/device/lattice`, `/metadata/version`
- Compression: gzip level 6 for large Hamiltonians (50K+ atoms)
- Deserialization with validation: check Hermiticity, positive definiteness of overlap matrix
- S3 upload/download integration: presigned URLs for client-side download of precomputed Hamiltonians
- Performance benchmark: serialize 10K-atom Si nanowire Hamiltonian, target <500ms, <5MB file size

### Phase 3 — Ballistic NEGF Solver (Days 11–16)

**Day 11: Contact self-energy via Sancho-Rubio**
- `negf-core/` — New Rust workspace member for quantum transport
- `negf-core/src/contact/sancho_rubio.rs` — Iterative Sancho-Rubio algorithm for surface Green's function
- Handle both real and complex energies (E + iη)
- Convergence criterion: ||ε_s^(n+1) - ε_s^(n)|| < 1e-10, max 100 iterations
- Special handling for band edges (slow convergence near Van Hove singularities)
- Unit tests: 1D infinite chain with nearest-neighbor hopping → verify self-energy matches analytical formula
- Benchmark: Sancho-Rubio for 100×100 2D contact → target <50ms per energy point

**Day 12: Recursive Green's function (RGF) core algorithm**
- `negf-core/src/solver/rgf.rs` — RGF forward and backward recursion
- Forward recursion: compute g_i from left contact, store all layers
- Backward recursion: incorporate right contact, compute full G_i
- Memory optimization: only store diagonal blocks of G for LDOS, off-diagonal for current
- Parallelization: independent energy points processed in parallel via Rayon
- Complex matrix operations: use `nalgebra` with `Complex64`, custom sparse inversion via LU decomposition
- Unit tests: 1D tight-binding chain → verify G matches analytical (E - H - Σ)^(-1)

**Day 13: Transmission and current calculation**
- `negf-core/src/solver/transmission.rs` — Compute T(E) = Trace[Γ_L G^r Γ_R G^a]
- Broadening matrices: Γ_L/R = i(Σ^r - Σ^a) from contact self-energies
- Trace computation: efficient via diagonal extraction (avoid full matrix multiplication)
- Energy grid generation: adaptive refinement near transmission peaks
- Current integration: `negf-core/src/solver/current.rs` — Landauer-Büttiker formula with adaptive quadrature (Gauss-Kronrod)
- Fermi-Dirac integrals with temperature: handle zero-temperature limit (step function) and finite T
- Unit tests: 1D chain with single barrier → verify T(E) ∈ [0, 1], current vs. voltage (Ohmic at low bias)

**Day 14: LDOS and spectral function**
- `negf-core/src/solver/ldos.rs` — Local density of states: ρ(r, E) = -1/π Im[G^r(r, r; E)]
- Extract diagonal blocks of retarded Green's function per layer
- Spatial interpolation: map orbital-resolved LDOS to real-space grid via Gaussian smoothing
- Energy-resolved LDOS for band diagram visualization
- Integrated LDOS: electron density n(r) = ∫ dE ρ(r, E) f(E)
- Output format: 3D array (x, y, z, LDOS) serialized to HDF5 for frontend rendering
- Unit tests: uniform 1D system → verify LDOS matches analytical DOS

**Day 15: Ballistic solver integration and I-V sweep**
- `negf-core/src/solver/iv_sweep.rs` — I-V characteristic sweep over Vds and Vgs ranges
- Voltage application: shift contact chemical potentials (μ_L = E_F, μ_R = E_F - eV_ds), gate modulates on-site energies
- Gate capacitance model (simple): ΔE_onsite = -e·V_gs / (t_ox / ε_ox) assuming parallel-plate capacitor
- Parallel execution: different bias points processed independently, collect results
- Output: I-V table, extracted metrics (I_on, I_off, SS, DIBL, V_th via linear extrapolation)
- Performance target: 10×10 I-V sweep (100 bias points) for 5K-atom device in <30 seconds on CPU

**Day 16: WASM compilation of ballistic solver**
- `negf-wasm/` — WASM wrapper around negf-core
- `negf-wasm/src/lib.rs` — WASM entry points: `compute_iv_sweep()`, `compute_transmission()`, `compute_ldos()`
- Accept Hamiltonian as binary blob (avoid WASM file I/O), deserialize in-memory
- Return results as Float64Array for efficient transfer to JavaScript
- Memory management: explicit allocation/deallocation for large arrays
- Build pipeline: `wasm-pack build --target web --release`, `wasm-opt -O3 -oz`
- Target WASM bundle size: <3MB gzipped (includes sparse matrix lib, complex arithmetic)
- Browser test: load WASM, run 5K-atom Si nanowire I-V sweep, verify <10 seconds in Chrome

### Phase 4 — Frontend Device Builder + Visualization (Days 17–22)

**Day 17: Frontend scaffold and Three.js setup**
```bash
npm create vite@latest frontend -- --template react-ts
cd frontend
npm i zustand @tanstack/react-query axios three @react-three/fiber @react-three/drei
npm i -D @types/three
```
- `src/App.tsx` — Router (react-router-dom), auth context, layout (sidebar + main canvas + right panel)
- `src/stores/deviceStore.ts` — Zustand store for device structure (atoms, bonds, materials, geometry params)
- `src/stores/simulationStore.ts` — Simulation state (parameters, results, progress)
- `src/components/Layout/Sidebar.tsx` — Template library, recent projects, material database
- `src/components/Layout/RightPanel.tsx` — Property editor, simulation controls, results viewer
- Three.js canvas setup: `src/components/AtomicViewer/Canvas.tsx` with OrbitControls, PerspectiveCamera

**Day 18: Device structure builder UI**
- `src/components/DeviceBuilder/TemplateSelector.tsx` — Gallery of device templates with preview images
- `src/components/DeviceBuilder/GeometryEditor.tsx` — Form for geometry parameters (gate length, width, thickness, doping)
- `src/components/DeviceBuilder/MaterialSelector.tsx` — Dropdown for channel material (Si, Ge, GaAs, InAs, graphene, MoS2)
- Live preview: modify geometry param → debounced API call to generate structure → update 3D view
- API endpoint: `POST /api/projects/:id/generate-structure` → calls Rust structure builder → returns atom positions
- Validation: min/max constraints on geometry (gate length 5-100nm, diameter 1-20nm), atom count limit per plan

**Day 19: Atomic structure 3D rendering**
- `src/components/AtomicViewer/AtomicStructure.tsx` — Instanced sphere rendering for atoms (completed in Architecture section above)
- Bond rendering: cylinder geometry between bonded atoms (distance < 3Å threshold)
- Element color coding: periodic table colors (Si gray, Ge purple, Ga green, As red, C black, Mo blue, S yellow)
- Camera controls: orbit (drag), pan (right-click drag), zoom (scroll), reset view button
- Performance: test rendering 100K atom structure (large GAA device) → target 60fps with GPU instancing
- Selection: raycasting for atom picking, highlight selected atom, show properties in panel

**Day 20: Material database and property visualization**
- `src/pages/Materials.tsx` — Searchable table of materials with properties (bandgap, lattice constant, category)
- `src/components/Materials/BandStructure.tsx` — Plot E-k dispersion from precomputed band data (fetched from API)
- `src/components/Materials/CrystalStructure.tsx` — Unit cell visualization with lattice vectors
- API endpoints:
  - `GET /api/materials` → list all materials with search/filter
  - `GET /api/materials/:id` → material details
  - `GET /api/materials/:id/band-structure` → precomputed E-k data
- Click material card → show detailed properties modal with band structure, DOS, references

**Day 21: Simulation control panel**
- `src/components/Simulation/ControlPanel.tsx` — Simulation type selector (ballistic, scattering, self-consistent)
- Bias configuration: Vgs range [start, stop, step], Vds range [start, stop, step], Vbs fixed
- Temperature slider (77K - 400K), default 300K
- Scattering options (checkboxes): acoustic phonon, optical phonon, surface roughness (grayed out for ballistic)
- Estimate atom count, execution mode (WASM/server), compute time (based on historical data)
- Plan limit warnings: "This simulation requires GPU (Pro plan or higher)"
- Run button → create simulation via API, poll status or subscribe to WebSocket for progress

**Day 22: WebSocket progress monitoring**
- `src/hooks/useSimulationProgress.ts` — React hook for WebSocket subscription to simulation progress
- Connect to `wss://api.nanodesign.com/ws/simulations/:id/progress` with JWT auth
- Receive progress updates: `{ progress_pct, current_bias_point, eta_seconds }`
- Display progress bar with percentage, current bias (Vgs, Vds), estimated time remaining
- Handle WebSocket reconnection on disconnect, cleanup on unmount
- Visual feedback: progress spinner, "Running on GPU worker-07" status text
- Cancel button → send cancel request to API, gracefully terminate simulation job

### Phase 5 — Results Visualization + Analysis (Days 23–28)

**Day 23: I-V characteristic plotting**
- `src/components/Results/IVCurve.tsx` — WebGL-accelerated line plot for I-V data
- Transfer characteristics: Id vs. Vgs (log scale) for multiple Vds values, linear scale for linear region
- Output characteristics: Id vs. Vds for multiple Vgs values
- GPU rendering via custom WebGL shaders (similar to waveform viewer architecture)
- Interactive cursor: hover to show (Vgs, Id) values, click to add marker
- Dual Y-axis: linear current (A) on left, log current on right for subthreshold view
- Export: download as CSV, PNG image (1920×1080), SVG vector graphic

**Day 24: Transmission spectrum visualization**
- `src/components/Results/TransmissionSpectrum.tsx` — T(E) vs. energy plot
- Energy axis relative to Fermi level (E - E_F in eV)
- Overlay with contact DOS to show band edges
- Channel decomposition: stack plot showing contribution from different transverse modes
- Integration markers: highlight energy window contributing to current at given bias
- Zoom and pan: scroll to zoom energy range, drag to pan
- Measurement tools: cursor readout, T(E_F) value extraction, effective bandgap from T(E) onset

**Day 25: LDOS spatial visualization**
- `src/components/Results/LDOSViewer.tsx` — 3D volumetric rendering of local density of states
- Marching cubes isosurface extraction for specific LDOS threshold
- Color mapping: blue (low) → green → yellow → red (high) LDOS intensity
- Energy slider: select energy E to display ρ(r, E) at that energy slice
- Overlay atomic structure (transparent spheres) with LDOS isosurface
- Slice view: cut-plane visualization (XY, XZ, YZ planes) with LDOS heatmap
- Performance: use Three.js VolumeRenderShader, test with 50×50×200 grid → target 30fps

**Day 26: Device metrics extraction**
- `src/components/Results/MetricsPanel.tsx` — Automated extraction of key device metrics
- Threshold voltage (V_th): linear extrapolation of Id-Vgs at max transconductance point
- Subthreshold swing (SS): SS = d(Vgs) / d(log₁₀ Id) in mV/dec, extract min SS below V_th
- DIBL (Drain-Induced Barrier Lowering): ΔV_th / ΔV_ds for low/high Vds
- On/Off ratio: I_on (at Vgs = Vdd, Vds = Vdd) / I_off (at Vgs = 0, Vds = Vdd)
- Transconductance: gm = ∂Id / ∂Vgs, find peak gm
- Display metrics in table with comparison to ITRS/IRDS targets (e.g., SS < 70 mV/dec, Ion/Ioff > 10⁶)

**Day 27: Comparison and benchmarking tools**
- `src/components/Results/ComparisonView.tsx` — Side-by-side I-V comparison for multiple simulations
- Select up to 4 simulations from project history, plot on same axes with different colors
- Technology comparison: compare Si vs. Ge vs. III-V nanowires at same geometry
- Geometry sweep results: plot metric (SS, V_th, Ion) vs. geometry parameter (gate length, diameter)
- Export comparison table: CSV with all metrics for selected simulations
- ITRS benchmark overlay: plot historical ITRS targets (2005-2025) on same axes as simulation results

**Day 28: LDOS and charge density animation**
- `src/components/Results/LDOSAnimation.tsx` — Animate LDOS(E) as energy sweeps through band
- Energy sweep: automatically step through E from valence band to conduction band
- Playback controls: play/pause, speed (0.5×, 1×, 2×), loop
- Export: render frames to PNG sequence, client-side video encoding via WebCodecs API (MP4)
- Charge density visualization: n(r) = ∫ρ(r, E)f(E)dE integrated LDOS
- Potential profile: overlay electrostatic potential φ(r) from self-consistent Poisson solution (when available)
- Streamline view: current density vectors J(r) = ∫ dE v(r, E) ρ(r, E) [f_L - f_R] for occupied states

### Phase 6 — Self-Consistent NEGF-Poisson (Days 29–33)

**Day 29: 3D Poisson solver**
- `poisson-core/` — New Rust workspace for electrostatics
- `poisson-core/src/solver/multigrid.rs` — 3D multigrid Poisson solver for ∇²φ = -ρ/ε
- Finite difference discretization on regular 3D grid
- V-cycle multigrid: restriction (fine→coarse), coarse solve (Gauss-Seidel), prolongation (coarse→fine)
- Boundary conditions: Dirichlet (fixed φ at contacts), Neumann (∂φ/∂n = 0 at insulating boundaries)
- Dielectric permittivity: spatially varying ε(r) for oxide (ε_r = 3.9 for SiO₂), semiconductor (ε_r = 11.7 for Si), air (ε_r = 1)
- Unit tests: 1D parallel-plate capacitor → verify φ(x) = V·x/d linear profile
- Benchmark: 100×100×300 grid Poisson solve → target <100ms on CPU

**Day 30: NEGF-Poisson coupling**
- `negf-core/src/solver/self_consistent.rs` — Self-consistent iteration loop
- Iteration procedure:
  1. Solve Poisson: ∇²φ = -e(n - p + N_D - N_A) / ε with current potential guess
  2. Update Hamiltonian: H_ii^new = H_ii^old - eφ(r_i) for each atom i
  3. Solve NEGF: compute G^r, G^<, extract charge density n(r) from diagonal of G^<
  4. Mix potentials: φ^next = α·φ^new + (1-α)·φ^old (α = 0.1 to 0.3 for stability)
  5. Check convergence: ||φ^next - φ^old|| < threshold (default 1e-4 V)
  6. Repeat until converged or max iterations (default 50)
- Charge density extraction: n(r) = ∫ dE A(r, E) f(E) where A = i[G^< - (G^<)^†]
- Doping profile: read N_D(r), N_A(r) from device geometry (uniform or delta-doping)
- Convergence acceleration: Anderson mixing with history depth 5

**Day 31: Self-consistent solver parallelization**
- Distribute energy points across CPU cores for NEGF at each Poisson iteration
- GPU acceleration: offload matrix operations (inversion, multiplication) to CUDA
- `negf-core/src/cuda/rgf_kernels.cu` — CUDA kernels for block-wise RGF recursion
- cuBLAS for complex matrix operations (Zgemm, Zgetrf, Zgetri)
- Memory management: pinned host memory, asynchronous H2D/D2H transfers
- Multi-GPU: partition energy grid across multiple GPUs for large self-consistent runs
- Benchmark: 10K-atom Si nanowire, self-consistent NEGF-Poisson, 5 iterations → target <5 minutes on single V100 GPU

**Day 32: Server-side GPU worker**
- `src/workers/gpu_worker.rs` — GPU-accelerated simulation worker
- CUDA initialization: detect available GPUs, set device, allocate workspace memory
- Job dispatch: Redis queue with priority (Enterprise > Pro > Academic), GPU affinity
- Result streaming: periodic S3 upload of intermediate I-V points (allow partial results if job interrupted)
- Error handling: CUDA out-of-memory → fall back to CPU RGF, timeout (max 2 hours per simulation)
- GPU resource tracking: record gpu_time_ms per simulation, aggregate for billing
- Worker health monitoring: Prometheus metrics (GPU utilization, memory, temperature, job throughput)

**Day 33: Self-consistent simulation UI and progress**
- `src/components/Simulation/SelfConsistentOptions.tsx` — Convergence threshold slider, max iterations, mixing parameter
- Progress display: current iteration, residual ||φ^(n+1) - φ^(n)||, estimated remaining iterations
- Intermediate results: show I-V from latest completed bias point during sweep
- Convergence plot: real-time line plot of residual vs. iteration number
- WebSocket messages: `{ iteration, residual, current_bias, eta_minutes }`
- Warning if not converged after max iterations: "Poisson did not converge. Results may be inaccurate. Try increasing max iterations or relaxing threshold."

### Phase 7 — Python Services + Advanced Features (Days 34–37)

**Day 34: Python FastAPI service scaffold**
```bash
mkdir python-services
cd python-services
python -m venv venv
source venv/bin/activate
pip install fastapi uvicorn numpy scipy h5py pythtb wannier90
```
- `main.py` — FastAPI app with CORS, health check endpoint
- `routers/dft_import.py` — DFT Hamiltonian import from Wannier90 `.hr` format
- `routers/spice_extraction.py` — SPICE model parameter fitting from quantum I-V
- `services/wannier90.py` — Parse Wannier90 Hamiltonian file, convert to HDF5
- `services/bsim_fit.py` — Least-squares fitting of BSIM-CMG parameters to I-V data
- Dockerfile with Python 3.12, dependencies, expose port 8001
- docker-compose integration: Python service + Rust API + PostgreSQL + Redis

**Day 35: DFT Hamiltonian import**
- Parse Wannier90 `.hr` file: Hamiltonian in Wannier basis H(R) for lattice vectors R
- Fourier transform: H(k) = Σ_R e^(ik·R) H(R) to get k-space Hamiltonian
- Convert to tight-binding format: extract hopping integrals for nearest/next-nearest neighbors
- Handle spin-orbit coupling: 2×2 spinor structure in Wannier Hamiltonian
- Upload to S3: store as HDF5 with metadata (source DFT code, k-point mesh, energy cutoff)
- API endpoint: `POST /api/python/import-dft` (multipart form-data for `.hr` file upload)
- Validate: compute band structure from imported H, compare to DFT bands (user uploads reference)

**Day 36: SPICE model extraction**
- Fetch quantum I-V data from S3 (simulation results)
- BSIM-CMG parameter space: Vth0, U0 (mobility), Rdsw (source/drain resistance), SS_sat, DIBL
- Objective function: RMS error between quantum I_ds(V_gs, V_ds) and BSIM model
- Optimization: scipy.optimize.minimize with L-BFGS-B, parameter bounds from physical limits
- Generate SPICE `.lib` file with extracted parameters
- Upload to S3, create spice_extractions table entry
- API endpoint: `POST /api/python/extract-spice/:simulation_id`
- Return: extracted parameters JSON, calibration error (RMS %), S3 URL to .lib file

**Day 37: Variability and Monte Carlo**
- Random dopant fluctuation (RDF): generate N_D(r) with discrete random dopant positions
- Interface roughness: perturb oxide/semiconductor interface atoms with Gaussian distribution (σ = 0.3nm)
- Line-edge roughness: modulate gate edge with autocorrelated roughness (Gaussian random field)
- Monte Carlo loop: generate M samples (M = 100-1000), run NEGF for each, collect I-V statistics
- Output: mean I-V, std deviation, percentiles (5%, 95%), V_th distribution histogram
- API endpoint: `POST /api/python/variability/:project_id` (long-running job, Redis queue)
- Results: S3 JSON with per-sample I-V + aggregate statistics
- UI: `src/components/Results/VariabilityView.tsx` → V_th histogram, I-V envelope plot (mean ± 2σ)

### Phase 8 — Billing, Validation, Polish (Days 38–42)

**Day 38: Stripe billing integration**
- `src/api/handlers/billing.rs` — Create checkout session, customer portal session, subscription status
- `src/api/handlers/webhooks/stripe.rs` — Webhook handler for Stripe events
- Events: `checkout.session.completed` → create subscription, `customer.subscription.updated` → update plan, `customer.subscription.deleted` → downgrade to free, `invoice.payment_failed` → suspend account
- Plan tiers:
  - Free: ballistic only, ≤5K atoms WASM, 10 simulations/month, no GPU
  - Academic ($99/mo lab, 5 seats): ballistic, ≤50K atoms CPU, 200 GPU-hours/month
  - Pro ($399/mo): scattering, self-consistent, ≤100K atoms GPU, 500 GPU-hours/month, API access, SPICE extraction
  - Enterprise (custom): unlimited, multi-GPU, on-premise deployment, custom material Hamiltonian development
- Webhook signature verification (Stripe webhook secret)

**Day 39: Usage tracking and enforcement**
- `src/middleware/plan_limits.rs` — Check plan limits before simulation execution
- Track GPU hours per billing period (monthly reset)
- Usage dashboard: `src/api/handlers/usage.rs` → current period GPU hours, atom count histogram, simulation count
- Frontend: `src/pages/Billing.tsx` → plan comparison table, current usage meter (progress bar), upgrade CTA
- Limit warnings: "You've used 450/500 GPU-hours this month. Upgrade to Enterprise for unlimited."
- Soft limit: allow overages up to 10%, charge overage fees ($2/GPU-hour over limit)
- Hard limit: block new simulations when overage limit reached

**Day 40: Solver validation benchmarks**
- Benchmark 1: 1D quantum wire with parabolic confinement → verify quantized conductance G = (2e²/h)·N where N = number of occupied subbands
- Benchmark 2: Si nanowire 3nm diameter → compare ballistic I-V to published results (Luisier 2011 Nano Letters)
- Benchmark 3: Graphene nanoribbon → verify conductance quantization, band gap vs. width for armchair ribbons
- Benchmark 4: Resonant tunneling diode (double barrier) → verify Breit-Wigner resonance peak in T(E)
- Benchmark 5: Self-consistent Si MOSFET → compare V_th, SS, DIBL to Sentaurus TCAD (classical) and published NEGF results
- Automated test suite: `negf-core/tests/benchmarks.rs` with tolerance checks (I-V within 5%, T(E) within 2%)

**Day 41: Performance optimization and caching**
- Hamiltonian caching: store precomputed H for common templates (Si nanowire 3nm, 5nm, 10nm diameters)
- I-V result caching: Redis cache for ballistic simulations (deterministic), TTL 7 days
- WASM solver optimization: profile with Chrome DevTools, optimize hot loops, inline small functions
- GPU kernel optimization: coalesced memory access, shared memory for block-wise operations, stream concurrency
- Database query optimization: add missing indexes (simulations.created_at, projects.atom_count), analyze slow queries
- CDN caching: cache WASM bundle, Three.js assets, material database JSON (immutable, 1-year TTL)

**Day 42: Documentation, examples, and polish**
- User guide: `docs/user-guide.md` → getting started, building first device, running simulation, interpreting results
- API documentation: OpenAPI spec generated from Axum routes, hosted at `/api/docs`
- Example gallery: 10 pre-built example projects (Si nanowire FET, graphene ribbon, InAs TFET, MoS2 transistor)
- Video tutorials: record 5-minute tutorial videos for key workflows (upload to S3, embed in app)
- Error messages: user-friendly error text ("Simulation failed: Poisson did not converge. Try reducing bias step size.") instead of stack traces
- Loading states: skeleton loaders for 3D viewer, simulation results, material database
- Responsive design: test on 1920×1080, 1440p, 4K displays, ensure readable text and UI scaling
- Final QA: end-to-end test flows, cross-browser testing (Chrome, Firefox, Safari), accessibility audit (ARIA labels, keyboard navigation)

---

## Validation Benchmarks

### Benchmark 1: 1D Quantum Wire Conductance Quantization
**Setup:** Infinite 1D quantum wire with parabolic confinement (ℏω = 10 meV), ballistic transport, T = 0K.
**Expected:** Conductance G = (2e²/h)·N where N is number of occupied subbands below Fermi level.
**Target:** G = 77.5 µS (2e²/h) for single subband, 155 µS for two subbands, etc.
**Validation:** Compute I-V at low bias (linear regime), extract G = dI/dV, verify G matches quantized values within 0.1%.

### Benchmark 2: Si Nanowire Ballistic I-V vs. Literature
**Setup:** Si nanowire, [110] transport, 3nm diameter hexagonal cross-section, H-passivated, Vds = 0.5V sweep.
**Expected:** Reproduce I-V from Luisier et al. Nano Letters 2011 Figure 2 (ballistic limit, sp3d5s* tight-binding).
**Target:** Ion ≈ 1200 µA/µm at Vgs = 0.5V, SS ≈ 65 mV/dec, Ion/Ioff ≈ 10⁵.
**Validation:** Extract SS and Ion from our simulation, compare to published values, require agreement within 10%.

### Benchmark 3: Graphene Nanoribbon Conductance and Band Gap
**Setup:** Armchair graphene nanoribbon (AGNR), width N = 7, 13, 19 atoms (families 3p+1), ballistic.
**Expected:** Band gap Eg scales as 1/N, conductance at E > Eg is 2e²/h per valley.
**Target:** N=7 AGNR → Eg ≈ 1.2 eV, N=13 → Eg ≈ 0.7 eV, N=19 → Eg ≈ 0.5 eV.
**Validation:** Compute transmission spectrum T(E), extract Eg from onset energy, verify scaling law Eg ∝ 1/N within 5%.

### Benchmark 4: Resonant Tunneling Diode Peak-to-Valley Ratio
**Setup:** GaAs/AlGaAs double-barrier structure, barrier thickness 3nm, well width 5nm, ballistic.
**Expected:** Resonant peak in I-V when E_F aligns with quantum well bound state, valley at higher bias.
**Target:** Peak-to-valley ratio (PVR) ≈ 3-5, peak position V_peak ≈ 0.15V for typical structure.
**Validation:** Compute I-V, identify peak and valley currents, verify PVR within 20% of experimental RTD devices.

### Benchmark 5: Self-Consistent Si MOSFET vs. TCAD
**Setup:** Si gate-all-around nanowire FET, 5nm diameter, 20nm gate length, self-consistent NEGF-Poisson.
**Expected:** Compare to Sentaurus TCAD (classical drift-diffusion) and published NEGF results for Vth, SS, DIBL.
**Target:** Vth ≈ 0.3V, SS ≈ 70 mV/dec (quantum limit), DIBL ≈ 100 mV/V, Ion/Ioff ≈ 10⁶.
**Validation:** Run self-consistent simulation, extract metrics, verify SS approaches 60 mV/dec (room T quantum limit) while TCAD shows degraded SS > 80 mV/dec due to classical treatment.

---

## Post-MVP Roadmap

### v1.1 — Scattering (Phonon, Roughness) (Weeks 13-16)
- Electron-phonon scattering: acoustic (deformation potential), optical (polar and non-polar)
- Self-energy Σ^scatt(E) in Born approximation, incorporate into NEGF as Σ^r = Σ^contact + Σ^scatt
- Surface roughness scattering: Gaussian-correlated interface potential fluctuations
- Mobility degradation: compare ballistic vs. scattering I-V, extract effective mobility
- Validation: Si nanowire mobility vs. diameter (compare to experimental data from Colinge 2006)

### v1.2 — 2D Materials Full Stack (Weeks 17-20)
- Extend tight-binding to TMDs: MoS2, WSe2, MoTe2 with d-orbital basis
- Heterojunction builder: graphene/hBN, MoS2/WSe2 van der Waals heterostructures
- Interlayer coupling: account for weak van der Waals hopping (tight-binding or DFT)
- Strain engineering: apply biaxial/uniaxial strain to 2D materials, modify band structure
- Validation: MoS2 transistor I-V vs. experimental (Radisavljevic 2011 Nature Nano)

### v1.3 — Spin Transport and Spintronics (Weeks 21-24)
- Spin-resolved NEGF: 2×2 spin-space Green's functions G^r_↑↑, G^r_↑↓, etc.
- Spin-orbit coupling: Rashba, Dresselhaus, intrinsic SOC from tight-binding
- Magnetic contacts: ferromagnetic leads with exchange splitting, compute spin-polarized current
- Spin transfer torque: calculate STT magnitude and direction, coupling to LLG micromagnetics
- TMR calculation: tunneling magnetoresistance in Fe/MgO/Fe magnetic tunnel junction
- Validation: TMR vs. barrier thickness (compare to Jullière/Slonczewski models)

### v1.4 — Collaboration and Teams (Weeks 25-26)
- Real-time collaboration: Yjs CRDT for simultaneous device editing
- Project sharing: invite collaborators with read/write/admin permissions
- Comments and annotations: threaded comments on atoms, regions, simulation results
- Version history: Git-like version control for device structures, diff view
- Activity feed: see team member edits, simulations, comments in real-time

### v1.5 — Python API and Jupyter Integration (Weeks 27-28)
- Python SDK: `pip install nanodesign-sdk`, authenticate with API key
- Programmatic device creation: `device = Device.from_template('si_nanowire')`, `device.set_gate_length(20)`
- Submit simulations: `sim = device.simulate(vgs=[0, 0.5], vds=[0, 0.5])`
- Fetch results: `iv_data = sim.get_iv()`, `ldos = sim.get_ldos(energy=0.1)`
- Jupyter notebook examples: interactive device building, parameter sweeps, plotting with matplotlib
- Validation: reproduce tutorial examples via API
