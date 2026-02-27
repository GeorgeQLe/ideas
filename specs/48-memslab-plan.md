# 48. MEMSLab — MEMS/NEMS Design, Simulation, and Fabrication Planning Platform

## Implementation Plan

**MVP Scope:** Browser-based 2D mask layout editor with DRC (design rule checking) for ≤8 layers supporting Boolean operations and parameterized MEMS cell generation (comb drives, cantilevers, serpentine springs, membranes), process flow editor with 40+ calibrated MEMS fabrication steps (LPCVD/PECVD deposition, wet/dry/DRIE etching, thermal oxidation, CMP, wafer bonding, sacrificial release), voxel-based 3D process simulator (50 nm resolution) compiled to WebAssembly for client-side execution of ≤100K voxel structures and server-side Rust execution for larger geometries, multi-physics FEA engine supporting mechanical (static, modal, buckling), electrostatic (capacitance extraction, pull-in voltage), and thermal analyses with iterative electromechanical coupling, GDS-II import/export, PolyMUMPs foundry PDK with design rules and process stack, Stripe billing with three tiers (Academic $79/mo / Pro $299/mo / Enterprise custom).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| Process Simulator | Rust (native + WASM) | Voxel-based level-set method, Boolean CSG operations |
| FEA Solver | Rust + `nalgebra-sparse` | Custom multi-physics with direct sparse solver (MUMPS via `mumps-sys`) |
| WASM Compilation | `wasm-pack` + `wasm-bindgen` | Client-side for ≤100K voxels, process + FEA |
| Layout Engine | Rust + WASM | GDS-II parser/writer, DRC engine, Boolean ops via `geo` crate |
| Mesh Generation | Rust (Gmsh API) | Tetrahedral meshing from voxel surfaces, adaptive refinement |
| Model Extraction | Python 3.12 (FastAPI) | Material property extraction, PDK validation, optimization |
| Database | PostgreSQL 16 | Projects, users, simulations, PDKs, material libraries |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | GDS files, mesh files, simulation results, process snapshots |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| Layout Editor | Canvas 2D + OffscreenCanvas | High-performance polygon rendering, R-tree spatial index |
| 3D Visualization | Three.js + WebGL 2.0 | Voxel volume rendering, FEA result visualization |
| Real-time | WebSocket (Axum) | Live simulation progress, mesh generation status |
| Job Queue | Redis 7 + Tokio tasks | Server-side simulation job management, parametric sweeps |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | WASM bundle delivery, material library assets |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, error tracking |
| CI/CD | GitHub Actions | WASM build pipeline, Rust tests, Docker image push |

### Key Architecture Decisions

1. **Voxel-based process simulation with 50nm resolution and WASM threshold at 100K voxels**: Voxel representation simplifies Boolean operations (deposition, etching, masking) compared to analytical surface tracking. 50nm resolution provides accuracy for modern MEMS features (1-100µm) while keeping memory manageable. Structures ≤100K voxels (e.g., 100×100×10 µm³ device) simulate in browser via WASM in <5 seconds; larger structures offload to server with multi-threaded Rust. Voxel data stored as sparse octrees for 10-50× memory compression.

2. **Level-set method for accurate etch profile prediction**: Anisotropic wet etching (KOH, TMAH on Si<100>) creates complex crystallographic features ({111} planes). Level-set function Φ(x,y,z,t) represents the surface implicitly; Hamilton-Jacobi PDE ∂Φ/∂t + F|∇Φ| = 0 with orientation-dependent etch rate F(n̂) propagates the surface. DRIE (Bosch process) alternates between isotropic etch and polymer deposition, producing scalloped sidewalls; level-set captures this with sub-voxel accuracy via signed distance function.

3. **Sparse FEA with iterative electromechanical coupling**: Electrostatic problems scale as O(N³) for dense matrices but MEMS capacitance matrices are sparse (<0.1% fill). MUMPS direct solver via `mumps-sys` handles 100K DOF in <10 seconds. For electromechanical devices (capacitive sensors, resonators), coupling is sequential: solve electrostatic for force → solve mechanical for displacement → update geometry → iterate until convergence (typically 3-5 iterations). Pull-in instability detected via negative stiffness eigenvalue.

4. **Parameterized cell library with constraint-based generation**: MEMS structures are highly parameterized: comb drive (finger width, gap, length, count), serpentine spring (beam width, turns, spacing). Cells defined as Rust procedural macros that generate polygons from parameters while enforcing design constraints (min feature size, aspect ratio limits). Boolean layer operations (AND, OR, XOR, NOT) enable complex multi-layer structures. Cells exported as GDS subcells for reuse.

5. **PolyMUMPs PDK as MVP foundry target with extensible PDK framework**: PolyMUMPs (MEMSCAP) is the most accessible MEMS process for prototyping: 3 polysilicon structural layers, 2 sacrificial oxides, 1 metal layer, $5K for multi-project wafer run. PDK includes: layer stack definition (materials, thicknesses), design rules (min width, spacing, enclosure), process flow (40 steps: deposition, photolithography, etch, release), calibrated material properties (E, ν, ρ, α, σ_residual). Framework supports adding custom PDKs via YAML config + calibration data.

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";  -- Fuzzy search on materials, PDKs

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

-- Organizations (for team/enterprise plans)
CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    slug TEXT UNIQUE NOT NULL,
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    plan TEXT NOT NULL DEFAULT 'academic',
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

-- Projects (mask layout + process flow + simulations)
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    device_type TEXT,  -- accelerometer | gyroscope | pressure_sensor | rf_switch | microfluidic | custom
    layout_data JSONB NOT NULL DEFAULT '{}',  -- Mask layers, polygons, cells
    process_flow JSONB NOT NULL DEFAULT '[]',  -- Ordered list of process steps
    pdk_id UUID REFERENCES pdks(id) ON DELETE SET NULL,
    settings JSONB DEFAULT '{}',  -- Grid, snap, units, DRC settings
    is_public BOOLEAN NOT NULL DEFAULT false,
    forked_from UUID REFERENCES projects(id) ON DELETE SET NULL,
    thumbnail_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);
CREATE INDEX projects_org_idx ON projects(org_id);
CREATE INDEX projects_pdk_idx ON projects(pdk_id);
CREATE INDEX projects_updated_idx ON projects(updated_at DESC);

-- Process Design Kits
CREATE TABLE pdks (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,  -- e.g., "PolyMUMPs", "XFAB XMB10"
    foundry TEXT NOT NULL,  -- MEMSCAP | XFAB | TSMC | Custom
    version TEXT NOT NULL,
    layer_stack JSONB NOT NULL,  -- [{name, material_id, thickness_nm, sequence}]
    design_rules JSONB NOT NULL,  -- {min_width, min_spacing, enclosure_rules}
    process_library JSONB NOT NULL,  -- Available process steps with calibrated parameters
    is_builtin BOOLEAN DEFAULT true,
    is_public BOOLEAN DEFAULT false,
    uploaded_by UUID REFERENCES users(id),
    download_count INTEGER DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX pdks_foundry_idx ON pdks(foundry);
CREATE INDEX pdks_name_trgm_idx ON pdks USING gin(name gin_trgm_ops);

-- Material Library
CREATE TABLE materials (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,  -- "Polysilicon", "Silicon <100>", "Silicon Dioxide", "Aluminum"
    category TEXT NOT NULL,  -- structural | sacrificial | metal | dielectric
    properties JSONB NOT NULL,
    -- {
    --   youngs_modulus_pa: 160e9,
    --   poisson_ratio: 0.22,
    --   density_kg_m3: 2330,
    --   thermal_expansion_k: 2.6e-6,
    --   thermal_conductivity_w_mk: 148,
    --   electrical_resistivity_ohm_m: 1e-3,
    --   permittivity_relative: 11.7,
    --   residual_stress_pa: -20e6,
    --   yield_strength_pa: 7e9,
    --   fracture_toughness_pa_sqrt_m: 0.9e6
    -- }
    is_builtin BOOLEAN DEFAULT true,
    source TEXT,  -- Reference: paper, datasheet, measurement
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX materials_category_idx ON materials(category);
CREATE INDEX materials_name_trgm_idx ON materials USING gin(name gin_trgm_ops);

-- Process Simulations
CREATE TABLE process_simulations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | running | completed | failed
    execution_mode TEXT NOT NULL DEFAULT 'wasm',  -- wasm | server
    voxel_count INTEGER NOT NULL DEFAULT 0,
    resolution_nm INTEGER NOT NULL DEFAULT 50,
    bounding_box JSONB,  -- {x_min, x_max, y_min, y_max, z_min, z_max} in nm
    structure_url TEXT,  -- S3 URL for voxel octree data
    mesh_url TEXT,  -- S3 URL for generated FEA mesh
    error_message TEXT,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    wall_time_ms INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX process_sims_project_idx ON process_simulations(project_id);
CREATE INDEX process_sims_user_idx ON process_simulations(user_id);
CREATE INDEX process_sims_status_idx ON process_simulations(status);
CREATE INDEX process_sims_created_idx ON process_simulations(created_at DESC);

-- FEA Simulations
CREATE TABLE fea_simulations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    process_simulation_id UUID REFERENCES process_simulations(id) ON DELETE SET NULL,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    analysis_type TEXT NOT NULL,  -- mechanical_static | mechanical_modal | electrostatic | thermal | coupled_em
    status TEXT NOT NULL DEFAULT 'pending',
    execution_mode TEXT NOT NULL DEFAULT 'wasm',
    mesh_info JSONB,  -- {n_nodes, n_elements, element_type, mesh_quality}
    boundary_conditions JSONB NOT NULL,
    material_assignments JSONB NOT NULL,
    parameters JSONB NOT NULL DEFAULT '{}',  -- Analysis-specific: loads, voltages, temperatures
    results_url TEXT,  -- S3 URL for result data (displacements, stresses, fields)
    results_summary JSONB,  -- {max_displacement, max_stress, resonant_frequencies, capacitance, pull_in_voltage}
    error_message TEXT,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    wall_time_ms INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX fea_sims_project_idx ON fea_simulations(project_id);
CREATE INDEX fea_sims_process_idx ON fea_simulations(process_simulation_id);
CREATE INDEX fea_sims_user_idx ON fea_simulations(user_id);
CREATE INDEX fea_sims_status_idx ON fea_simulations(status);
CREATE INDEX fea_sims_type_idx ON fea_simulations(analysis_type);
CREATE INDEX fea_sims_created_idx ON fea_simulations(created_at DESC);

-- Simulation Jobs (for server-side execution)
CREATE TABLE simulation_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    job_type TEXT NOT NULL,  -- process | fea
    reference_id UUID NOT NULL,  -- process_simulation_id or fea_simulation_id
    worker_id TEXT,
    cores_allocated INTEGER DEFAULT 1,
    memory_mb INTEGER DEFAULT 4096,
    priority INTEGER DEFAULT 0,
    progress_pct REAL DEFAULT 0.0,
    progress_data JSONB DEFAULT '{}',  -- {current_step, total_steps, convergence_residual}
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX jobs_reference_idx ON simulation_jobs(reference_id);
CREATE INDEX jobs_worker_idx ON simulation_jobs(worker_id);
CREATE INDEX jobs_status_idx ON simulation_jobs(worker_id, started_at);

-- Usage Tracking
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,  -- compute_hours | storage_gb | foundry_submission
    quantity REAL NOT NULL,
    period_start DATE NOT NULL,
    period_end DATE NOT NULL,
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX usage_user_period_idx ON usage_records(user_id, period_start, period_end);
CREATE INDEX usage_org_period_idx ON usage_records(org_id, period_start, period_end);
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
    pub device_type: Option<String>,
    pub layout_data: serde_json::Value,
    pub process_flow: serde_json::Value,
    pub pdk_id: Option<Uuid>,
    pub settings: serde_json::Value,
    pub is_public: bool,
    pub forked_from: Option<Uuid>,
    pub thumbnail_url: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Pdk {
    pub id: Uuid,
    pub name: String,
    pub foundry: String,
    pub version: String,
    pub layer_stack: serde_json::Value,
    pub design_rules: serde_json::Value,
    pub process_library: serde_json::Value,
    pub is_builtin: bool,
    pub is_public: bool,
    pub uploaded_by: Option<Uuid>,
    pub download_count: i32,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Material {
    pub id: Uuid,
    pub name: String,
    pub category: String,
    pub properties: serde_json::Value,
    pub is_builtin: bool,
    pub source: Option<String>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct ProcessSimulation {
    pub id: Uuid,
    pub project_id: Uuid,
    pub user_id: Uuid,
    pub status: String,
    pub execution_mode: String,
    pub voxel_count: i32,
    pub resolution_nm: i32,
    pub bounding_box: Option<serde_json::Value>,
    pub structure_url: Option<String>,
    pub mesh_url: Option<String>,
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
    pub process_simulation_id: Option<Uuid>,
    pub user_id: Uuid,
    pub analysis_type: String,
    pub status: String,
    pub execution_mode: String,
    pub mesh_info: Option<serde_json::Value>,
    pub boundary_conditions: serde_json::Value,
    pub material_assignments: serde_json::Value,
    pub parameters: serde_json::Value,
    pub results_url: Option<String>,
    pub results_summary: Option<serde_json::Value>,
    pub error_message: Option<String>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub wall_time_ms: Option<i32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum AnalysisType {
    MechanicalStatic,
    MechanicalModal,
    Electrostatic,
    Thermal,
    CoupledEM,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct ProcessStep {
    pub step_type: String,  // deposit | etch | oxidation | dope | bond | release
    pub layer_name: String,
    pub material: String,
    pub parameters: serde_json::Value,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct BoundaryCondition {
    pub bc_type: String,  // fixed | load | voltage | temperature
    pub region: String,  // "bottom_surface" | "anchor_pad" | etc
    pub value: f64,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct MaterialProperties {
    pub youngs_modulus_pa: f64,
    pub poisson_ratio: f64,
    pub density_kg_m3: f64,
    pub thermal_expansion_k: f64,
    pub thermal_conductivity_w_mk: f64,
    pub electrical_resistivity_ohm_m: Option<f64>,
    pub permittivity_relative: f64,
    pub residual_stress_pa: f64,
}
```

---

## Process Simulation Architecture Deep-Dive

### Voxel Representation and Level-Set Method

MEMSLab's process simulator uses a **voxel grid** with 50nm resolution to represent the 3D structure during fabrication. Each voxel stores:
- Material ID (8 bits) — maps to material library entry
- Level-set value φ (16-bit signed) — signed distance to nearest surface (negative inside, positive outside)
- Process metadata (8 bits) — etch depth, doping concentration

For a typical 100×100×20 µm device at 50nm resolution: 2000×2000×400 = 1.6B voxels (uncompressed 6.4GB). Using sparse octree compression:
- Homogeneous regions (e.g., bulk silicon substrate) → single node
- Active regions (interfaces, patterned layers) → leaf voxels
- Achieves 10-50× compression → 128-640MB

### Process Step Implementation

**1. Deposition (LPCVD, PECVD, PVD, ALD)**
```
For each surface voxel (φ ≈ 0):
    φ_new(x,y,z) = φ_old(x,y,z) - thickness
    if φ_new < 0: material[x,y,z] = deposit_material
```
Conformal coating: thickness independent of orientation. Non-conformal (evaporation): thickness ∝ cos(θ) relative to surface normal.

**2. Wet Etching (Isotropic, Anisotropic KOH/TMAH)**

Isotropic (HF for oxide):
```
Rate F = constant
∂φ/∂t = -F
```

Anisotropic (KOH on Si<100>):
```
Rate F(n̂) depends on crystallographic orientation:
F(<100>) = R_100
F(<110>) = R_110 ≈ R_100 / 2
F(<111>) = R_111 ≈ R_100 / 100  (etch stop)

Surface propagates via level-set PDE:
∂φ/∂t + F(n̂)|∇φ| = 0
n̂ = ∇φ / |∇φ|
```

Discretization: 5th-order WENO scheme for spatial derivatives, 3rd-order TVD Runge-Kutta for time integration. Stable timestep Δt ≤ Δx / (2·max(F)).

**3. Dry Etching (RIE, DRIE Bosch)**

DRIE Bosch process alternates:
- SF6 plasma etch (isotropic, rate R_etch)
- C4F8 polymer deposition (sidewall passivation, thickness d_pass)

Creates scalloped sidewalls with period λ = cycle_time × R_etch. Implemented as:
```
for cycle in 1..N_cycles:
    // Etch phase
    φ -= R_etch × t_etch
    // Deposit polymer on sidewalls
    if |∇φ · vertical| < 0.5:  // Sidewall criterion
        φ -= d_pass
```

**4. Sacrificial Release (HF vapor, XeF2)**

Removes sacrificial material (e.g., SiO2) while preserving structural material:
```
For each voxel with material == sacrificial:
    if any neighbor voxel is void or etchant-accessible:
        material = void
        φ = +∞
```
Etchant diffusion modeled via Eikonal equation |∇T| = 1/v where T is arrival time, v is diffusion speed.

### WebAssembly Compilation

```toml
# process-sim-wasm/Cargo.toml
[package]
name = "memslab-process-wasm"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib"]

[dependencies]
wasm-bindgen = "0.2"
serde = { version = "1", features = ["derive"] }
serde-wasm-bindgen = "0.6"
js-sys = "0.3"
ndarray = { version = "0.15", features = ["serde"] }
rayon = "1.8"  # Parallel processing (uses Web Workers in WASM)

[profile.release]
opt-level = "z"
lto = true
codegen-units = 1
```

```rust
// process-sim-wasm/src/lib.rs

use wasm_bindgen::prelude::*;
use ndarray::{Array3, s};

#[wasm_bindgen]
pub struct ProcessSimulator {
    voxels: Array3<u8>,  // Material IDs
    levelset: Array3<i16>,  // Signed distance × 100 (for 16-bit storage)
    resolution_nm: u32,
    bounds: [usize; 3],  // [nx, ny, nz]
}

#[wasm_bindgen]
impl ProcessSimulator {
    #[wasm_bindgen(constructor)]
    pub fn new(x_nm: u32, y_nm: u32, z_nm: u32, resolution_nm: u32) -> Self {
        let nx = (x_nm / resolution_nm) as usize;
        let ny = (y_nm / resolution_nm) as usize;
        let nz = (z_nm / resolution_nm) as usize;

        Self {
            voxels: Array3::zeros((nx, ny, nz)),
            levelset: Array3::from_elem((nx, ny, nz), i16::MAX),
            resolution_nm,
            bounds: [nx, ny, nz],
        }
    }

    pub fn deposit_layer(&mut self, material_id: u8, thickness_nm: u32, mask_layer: Option<Vec<u8>>) {
        let thickness_voxels = (thickness_nm / self.resolution_nm) as usize;

        for ix in 0..self.bounds[0] {
            for iy in 0..self.bounds[1] {
                // Check mask
                if let Some(ref mask) = mask_layer {
                    let mask_idx = iy * self.bounds[0] + ix;
                    if mask[mask_idx] == 0 { continue; }
                }

                // Find top surface
                let mut z_top = 0;
                for iz in (0..self.bounds[2]).rev() {
                    if self.voxels[[ix, iy, iz]] != 0 {
                        z_top = iz + 1;
                        break;
                    }
                }

                // Deposit layer
                let z_end = (z_top + thickness_voxels).min(self.bounds[2]);
                for iz in z_top..z_end {
                    self.voxels[[ix, iy, iz]] = material_id;
                    self.levelset[[ix, iy, iz]] = -((iz - z_top) as i16 * 100);
                }
            }
        }
    }

    pub fn etch_isotropic(&mut self, material_id: u8, depth_nm: u32) {
        let depth_voxels = (depth_nm / self.resolution_nm) as i16;

        // Update level-set
        for ix in 0..self.bounds[0] {
            for iy in 0..self.bounds[1] {
                for iz in 0..self.bounds[2] {
                    if self.voxels[[ix, iy, iz]] == material_id {
                        self.levelset[[ix, iy, iz]] += depth_voxels * 100;
                        if self.levelset[[ix, iy, iz]] > 0 {
                            self.voxels[[ix, iy, iz]] = 0;  // Material removed
                        }
                    }
                }
            }
        }
    }

    pub fn get_voxel_data(&self) -> Vec<u8> {
        self.voxels.as_slice().unwrap().to_vec()
    }

    pub fn get_surface_mesh(&self) -> JsValue {
        // Extract isosurface at φ=0 using marching cubes
        let vertices = Vec::new();
        let triangles = Vec::new();

        // Marching cubes implementation...
        // (simplified for brevity)

        serde_wasm_bindgen::to_value(&(vertices, triangles)).unwrap()
    }
}
```

---

## FEA Solver Architecture Deep-Dive

### Governing Equations

**Mechanical Static (Linear Elasticity)**
```
Equilibrium: ∇·σ + f = 0
Constitutive: σ = C:ε
Kinematics: ε = ½(∇u + (∇u)ᵀ)

Weak form: ∫_Ω δε:C:ε dV = ∫_Ω δu·f dV + ∫_∂Ω δu·t dS

FEM discretization: K u = F
K_ij = ∫_Ω Bᵢᵀ C Bⱼ dV  (stiffness matrix)
F_i = ∫_Ω Nᵢ f dV + ∫_∂Ω Nᵢ t dS  (force vector)
```

**Mechanical Modal (Eigenvalue Problem)**
```
Free vibration: ∇·σ + ρ ∂²u/∂t² = 0
Harmonic solution: u(x,t) = û(x) e^(iωt)

Eigenvalue problem: (K - ω² M) û = 0
M_ij = ∫_Ω ρ Nᵢ Nⱼ dV  (mass matrix)
```

**Electrostatic (Poisson Equation)**
```
∇·(ε∇V) = -ρ  (charge density)
E = -∇V  (electric field)
Capacitance: C = Q/V = ∫_∂Ω ε E·n̂ dS / V

Weak form: ∫_Ω ε ∇δV·∇V dV = ∫_Ω δV ρ dV

FEM: K_e V = Q
K_e,ij = ∫_Ω ε ∇Nᵢ·∇Nⱼ dV
```

**Coupled Electromechanical**
```
Electrostatic force on surface: f_e = ½ ε E² n̂
Maxwell stress tensor: σ_e = ε(E⊗E - ½E²I)

Iterative coupling:
1. Solve electrostatic: K_e V = Q → E, f_e
2. Solve mechanical: K u = F + f_e → u
3. Update mesh geometry: x_new = x_old + u
4. Repeat until ||u_new - u_old|| < tol

Pull-in detection: det(K_eff) = 0 where K_eff = K_mech - ∂f_e/∂u
```

### Sparse Direct Solver Integration

```rust
// fea-solver/src/sparse_solver.rs

use mumps_sys::*;
use nalgebra_sparse::{CsrMatrix, CooMatrix};

pub struct MumpsSolver {
    sym: i32,  // 0 = unsymmetric, 2 = symmetric positive definite
}

impl MumpsSolver {
    pub fn solve_symmetric(&self, K: &CsrMatrix<f64>, F: &[f64]) -> Result<Vec<f64>, SolverError> {
        let n = K.nrows() as i32;
        let nnz = K.nnz() as i32;

        // Convert CSR to COO (MUMPS requirement)
        let (row_indices, col_indices, values): (Vec<i32>, Vec<i32>, Vec<f64>) =
            csr_to_coo_mumps(K);

        unsafe {
            let mut mumps_data = DMUMPS_STRUC_C {
                sym: 2,  // Symmetric positive definite
                par: 1,  // Host is involved in computation
                comm_fortran: 0,  // Use Fortran communicator
                ..std::mem::zeroed()
            };

            // Initialize MUMPS
            mumps_data.job = -1;
            dmumps_c(&mut mumps_data);

            // Analysis phase
            mumps_data.job = 1;
            mumps_data.n = n;
            mumps_data.nz = nnz;
            mumps_data.irn = row_indices.as_ptr() as *mut i32;
            mumps_data.jcn = col_indices.as_ptr() as *mut i32;
            mumps_data.a = values.as_ptr() as *mut f64;
            dmumps_c(&mut mumps_data);

            // Factorization phase
            mumps_data.job = 2;
            dmumps_c(&mut mumps_data);

            // Solve phase
            mumps_data.job = 3;
            let mut solution = F.to_vec();
            mumps_data.rhs = solution.as_mut_ptr();
            dmumps_c(&mut mumps_data);

            // Cleanup
            mumps_data.job = -2;
            dmumps_c(&mut mumps_data);

            Ok(solution)
        }
    }

    pub fn solve_eigenvalue(
        &self, K: &CsrMatrix<f64>, M: &CsrMatrix<f64>, n_modes: usize
    ) -> Result<(Vec<f64>, Vec<Vec<f64>>), SolverError> {
        // Use shift-invert Lanczos for smallest eigenvalues
        // (K - σM)⁻¹ M v = λ v where λ = 1/(ω² - σ)

        let sigma = 0.0;  // Shift (find modes near 0 Hz)
        let K_shifted = shift_matrix(K, M, sigma)?;

        // Lanczos iteration to find n_modes eigenpairs
        // (Implementation uses ARPACK or custom Lanczos)

        unimplemented!("Eigenvalue solver requires ARPACK binding")
    }
}
```

---

## Architecture Deep-Dives

### 1. Process Simulation API Handler (Rust/Axum)

```rust
// src/api/handlers/process_simulation.rs

use axum::{
    extract::{Path, State, Json},
    http::StatusCode,
    response::IntoResponse,
};
use uuid::Uuid;
use crate::{
    db::models::{ProcessSimulation, ProcessStep},
    process::runner::run_process_flow,
    state::AppState,
    auth::Claims,
    error::ApiError,
};

#[derive(serde::Deserialize)]
pub struct CreateProcessSimRequest {
    pub resolution_nm: Option<u32>,  // Default 50nm
}

pub async fn create_process_simulation(
    State(state): State<AppState>,
    claims: Claims,
    Path(project_id): Path<Uuid>,
    Json(req): Json<CreateProcessSimRequest>,
) -> Result<impl IntoResponse, ApiError> {
    let resolution_nm = req.resolution_nm.unwrap_or(50);

    // 1. Fetch project
    let project = sqlx::query_as!(
        crate::db::models::Project,
        "SELECT * FROM projects WHERE id = $1 AND (owner_id = $2 OR org_id IN (
            SELECT org_id FROM org_members WHERE user_id = $2
        ))",
        project_id, claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Project not found"))?;

    // 2. Extract bounding box from layout
    let layout: serde_json::Value = project.layout_data;
    let bbox = calculate_bounding_box(&layout)?;
    let voxel_count = estimate_voxel_count(&bbox, resolution_nm);

    // 3. Check plan limits
    let user = sqlx::query_as!(
        crate::db::models::User,
        "SELECT * FROM users WHERE id = $1",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    if user.plan == "academic" && voxel_count > 500_000 {
        return Err(ApiError::PlanLimit(
            "Academic plan supports structures up to 500K voxels. Upgrade to Pro."
        ));
    }

    // 4. Determine execution mode
    let execution_mode = if voxel_count <= 100_000 { "wasm" } else { "server" };

    // 5. Create simulation record
    let sim = sqlx::query_as!(
        ProcessSimulation,
        r#"INSERT INTO process_simulations
            (project_id, user_id, status, execution_mode, voxel_count, resolution_nm, bounding_box)
        VALUES ($1, $2, $3, $4, $5, $6, $7)
        RETURNING *"#,
        project_id,
        claims.user_id,
        if execution_mode == "wasm" { "pending" } else { "pending" },
        execution_mode,
        voxel_count as i32,
        resolution_nm as i32,
        serde_json::to_value(&bbox)?
    )
    .fetch_one(&state.db)
    .await?;

    // 6. For server execution, enqueue job
    if execution_mode == "server" {
        let job = sqlx::query!(
            r#"INSERT INTO simulation_jobs (job_type, reference_id, priority, memory_mb)
            VALUES ('process', $1, $2, $3) RETURNING id"#,
            sim.id,
            if user.plan == "enterprise" { 10 } else { 0 },
            (voxel_count / 50_000 * 512).min(32768) as i32  // ~10MB per 50K voxels
        )
        .fetch_one(&state.db)
        .await?;

        state.redis
            .publish("process:jobs", serde_json::to_string(&job.id)?)
            .await?;
    }

    Ok((StatusCode::CREATED, Json(sim)))
}

pub async fn get_process_structure(
    State(state): State<AppState>,
    claims: Claims,
    Path((project_id, sim_id)): Path<(Uuid, Uuid)>,
) -> Result<impl IntoResponse, ApiError> {
    let sim = sqlx::query_as!(
        ProcessSimulation,
        "SELECT s.* FROM process_simulations s
         JOIN projects p ON s.project_id = p.id
         WHERE s.id = $1 AND s.project_id = $2
         AND (p.owner_id = $3 OR p.org_id IN (
             SELECT org_id FROM org_members WHERE user_id = $3
         ))",
        sim_id, project_id, claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Process simulation not found"))?;

    // Return presigned S3 URL for structure download
    let structure_url = sim.structure_url
        .ok_or(ApiError::NotReady("Structure not yet generated"))?;

    let presigned = state.s3
        .presign_get_object(&structure_url, 3600)
        .await?;

    Ok(Json(serde_json::json!({
        "simulation": sim,
        "download_url": presigned,
    })))
}

fn calculate_bounding_box(layout: &serde_json::Value) -> Result<BoundingBox, ApiError> {
    // Extract all polygons from layout layers and compute bounding box
    let layers = layout["layers"].as_array()
        .ok_or(ApiError::InvalidInput("Invalid layout format"))?;

    let mut x_min = f64::MAX;
    let mut x_max = f64::MIN;
    let mut y_min = f64::MAX;
    let mut y_max = f64::MIN;

    for layer in layers {
        let polygons = layer["polygons"].as_array().unwrap_or(&vec![]);
        for poly in polygons {
            let points = poly["points"].as_array().unwrap();
            for pt in points {
                let x = pt["x"].as_f64().unwrap();
                let y = pt["y"].as_f64().unwrap();
                x_min = x_min.min(x);
                x_max = x_max.max(x);
                y_min = y_min.min(y);
                y_max = y_max.max(y);
            }
        }
    }

    Ok(BoundingBox {
        x_min_nm: (x_min * 1e9) as u32,
        x_max_nm: (x_max * 1e9) as u32,
        y_min_nm: (y_min * 1e9) as u32,
        y_max_nm: (y_max * 1e9) as u32,
        z_min_nm: 0,
        z_max_nm: 10_000,  // Default 10µm height
    })
}
```

### 2. FEA Simulation Handler

```rust
// src/api/handlers/fea_simulation.rs

use axum::{extract::{Path, State, Json}, http::StatusCode, response::IntoResponse};
use uuid::Uuid;
use crate::{
    db::models::{FeaSimulation, AnalysisType, BoundaryCondition},
    state::AppState,
    auth::Claims,
    error::ApiError,
};

#[derive(serde::Deserialize)]
pub struct CreateFeaSimRequest {
    pub process_simulation_id: Uuid,
    pub analysis_type: AnalysisType,
    pub boundary_conditions: Vec<BoundaryCondition>,
    pub parameters: serde_json::Value,
}

pub async fn create_fea_simulation(
    State(state): State<AppState>,
    claims: Claims,
    Path(project_id): Path<Uuid>,
    Json(req): Json<CreateFeaSimRequest>,
) -> Result<impl IntoResponse, ApiError> {
    // 1. Verify project and process simulation ownership
    let process_sim = sqlx::query_as!(
        crate::db::models::ProcessSimulation,
        "SELECT s.* FROM process_simulations s
         JOIN projects p ON s.project_id = p.id
         WHERE s.id = $1 AND s.project_id = $2
         AND (p.owner_id = $3 OR p.org_id IN (
             SELECT org_id FROM org_members WHERE user_id = $3
         ))",
        req.process_simulation_id, project_id, claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Process simulation not found"))?;

    if process_sim.status != "completed" {
        return Err(ApiError::InvalidInput("Process simulation not completed"));
    }

    // 2. Fetch mesh information
    let mesh_url = process_sim.mesh_url
        .ok_or(ApiError::NotReady("Mesh not generated"))?;
    let mesh_info = fetch_mesh_metadata(&state.s3, &mesh_url).await?;

    // 3. Fetch project materials
    let project = sqlx::query_as!(
        crate::db::models::Project,
        "SELECT * FROM projects WHERE id = $1",
        project_id
    )
    .fetch_one(&state.db)
    .await?;

    let material_assignments = extract_material_assignments(&project)?;

    // 4. Check plan limits
    let user = sqlx::query_as!(
        crate::db::models::User,
        "SELECT * FROM users WHERE id = $1",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    let n_dof = mesh_info["n_nodes"].as_u64().unwrap() * 3;  // 3 DOF per node for mechanical
    if user.plan == "academic" && n_dof > 300_000 {
        return Err(ApiError::PlanLimit(
            "Academic plan supports meshes up to 100K nodes (300K DOF). Upgrade to Pro."
        ));
    }

    // 5. Determine execution mode (WASM threshold: 50K DOF)
    let execution_mode = if n_dof <= 50_000 { "wasm" } else { "server" };

    // 6. Create FEA simulation record
    let sim = sqlx::query_as!(
        FeaSimulation,
        r#"INSERT INTO fea_simulations
            (project_id, process_simulation_id, user_id, analysis_type, status,
             execution_mode, mesh_info, boundary_conditions, material_assignments, parameters)
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10)
        RETURNING *"#,
        project_id,
        req.process_simulation_id,
        claims.user_id,
        serde_json::to_string(&req.analysis_type)?,
        "pending",
        execution_mode,
        mesh_info,
        serde_json::to_value(&req.boundary_conditions)?,
        material_assignments,
        req.parameters
    )
    .fetch_one(&state.db)
    .await?;

    // 7. Enqueue job if server execution
    if execution_mode == "server" {
        let memory_mb = (n_dof / 1000 * 128).min(65536) as i32;  // ~128MB per 1K DOF
        sqlx::query!(
            r#"INSERT INTO simulation_jobs (job_type, reference_id, priority, memory_mb, cores_allocated)
            VALUES ('fea', $1, $2, $3, $4)"#,
            sim.id,
            if user.plan == "enterprise" { 10 } else { 0 },
            memory_mb,
            if n_dof > 500_000 { 8 } else { 4 }
        )
        .execute(&state.db)
        .await?;

        state.redis
            .publish("fea:jobs", serde_json::to_string(&sim.id)?)
            .await?;
    }

    Ok((StatusCode::CREATED, Json(sim)))
}
```

### 3. Layout Editor Component (React + Canvas)

```typescript
// frontend/src/components/LayoutEditor.tsx

import { useRef, useEffect, useCallback, useState } from 'react';
import { useLayoutStore } from '../stores/layoutStore';
import RBush from 'rbush';

interface Polygon {
  id: string;
  layer: string;
  points: Array<{x: number, y: number}>;  // Coordinates in nm
  bbox: {minX: number, minY: number, maxX: number, maxY: number};
}

interface Layer {
  name: string;
  color: string;
  visible: boolean;
  locked: boolean;
}

export function LayoutEditor() {
  const canvasRef = useRef<HTMLCanvasElement>(null);
  const offscreenRef = useRef<OffscreenCanvas | null>(null);
  const { polygons, layers, selectedLayer, viewport, setViewport } = useLayoutStore();

  const [spatialIndex] = useState(() => new RBush<Polygon>());
  const [isDragging, setIsDragging] = useState(false);
  const [dragStart, setDragStart] = useState<{x: number, y: number} | null>(null);

  // Initialize offscreen canvas for rendering
  useEffect(() => {
    const canvas = canvasRef.current;
    if (!canvas) return;

    offscreenRef.current = canvas.transferControlToOffscreen();
  }, []);

  // Update spatial index when polygons change
  useEffect(() => {
    spatialIndex.clear();
    spatialIndex.load(polygons.map(p => ({
      ...p,
      minX: p.bbox.minX,
      minY: p.bbox.minY,
      maxX: p.bbox.maxX,
      maxY: p.bbox.maxY,
    })));
  }, [polygons, spatialIndex]);

  // Render loop
  const render = useCallback(() => {
    const canvas = canvasRef.current;
    if (!canvas) return;

    const ctx = canvas.getContext('2d');
    if (!ctx) return;

    const { centerX, centerY, scale } = viewport;  // scale = nm per pixel
    const w = canvas.width;
    const h = canvas.height;

    // Clear
    ctx.fillStyle = '#1a1a1a';
    ctx.fillRect(0, 0, w, h);

    // World-to-screen transform
    const worldToScreen = (x_nm: number, y_nm: number) => ({
      x: (x_nm - centerX) / scale + w / 2,
      y: -(y_nm - centerY) / scale + h / 2,  // Y-axis flipped
    });

    // Query visible polygons
    const visibleBounds = {
      minX: centerX - (w / 2) * scale,
      maxX: centerX + (w / 2) * scale,
      minY: centerY - (h / 2) * scale,
      maxY: centerY + (h / 2) * scale,
    };
    const visiblePolygons = spatialIndex.search(visibleBounds);

    // Render polygons by layer
    for (const layer of layers) {
      if (!layer.visible) continue;

      ctx.fillStyle = layer.color + '80';  // 50% opacity
      ctx.strokeStyle = layer.color;
      ctx.lineWidth = 1;

      for (const poly of visiblePolygons) {
        if (poly.layer !== layer.name) continue;

        ctx.beginPath();
        const first = worldToScreen(poly.points[0].x, poly.points[0].y);
        ctx.moveTo(first.x, first.y);
        for (let i = 1; i < poly.points.length; i++) {
          const pt = worldToScreen(poly.points[i].x, poly.points[i].y);
          ctx.lineTo(pt.x, pt.y);
        }
        ctx.closePath();
        ctx.fill();
        ctx.stroke();
      }
    }

    // Render grid
    renderGrid(ctx, viewport, w, h);

    // Render ruler
    renderRuler(ctx, viewport, w, h);
  }, [polygons, layers, viewport, spatialIndex]);

  useEffect(() => {
    const animId = requestAnimationFrame(render);
    return () => cancelAnimationFrame(animId);
  }, [render]);

  // Pan and zoom
  const handleWheel = useCallback((e: React.WheelEvent) => {
    e.preventDefault();
    const canvas = canvasRef.current;
    if (!canvas) return;

    const rect = canvas.getBoundingClientRect();
    const mouseX = e.clientX - rect.left;
    const mouseY = e.clientY - rect.top;

    // Mouse position in world coordinates before zoom
    const worldX = viewport.centerX + (mouseX - canvas.width / 2) * viewport.scale;
    const worldY = viewport.centerY - (mouseY - canvas.height / 2) * viewport.scale;

    // Zoom
    const zoomFactor = e.deltaY > 0 ? 1.2 : 0.8;
    const newScale = viewport.scale * zoomFactor;

    // Adjust center to keep mouse position fixed
    const newCenterX = worldX - (mouseX - canvas.width / 2) * newScale;
    const newCenterY = worldY + (mouseY - canvas.height / 2) * newScale;

    setViewport({ centerX: newCenterX, centerY: newCenterY, scale: newScale });
  }, [viewport, setViewport]);

  const handleMouseDown = useCallback((e: React.MouseEvent) => {
    if (e.button === 1 || e.shiftKey) {  // Middle button or Shift+Left = pan
      setIsDragging(true);
      setDragStart({ x: e.clientX, y: e.clientY });
    }
  }, []);

  const handleMouseMove = useCallback((e: React.MouseEvent) => {
    if (!isDragging || !dragStart) return;

    const dx = (e.clientX - dragStart.x) * viewport.scale;
    const dy = -(e.clientY - dragStart.y) * viewport.scale;

    setViewport({
      ...viewport,
      centerX: viewport.centerX - dx,
      centerY: viewport.centerY - dy,
    });
    setDragStart({ x: e.clientX, y: e.clientY });
  }, [isDragging, dragStart, viewport, setViewport]);

  const handleMouseUp = useCallback(() => {
    setIsDragging(false);
    setDragStart(null);
  }, []);

  return (
    <div className="layout-editor flex h-full bg-gray-900">
      <LayerPanel layers={layers} selectedLayer={selectedLayer} />
      <div className="flex-1 relative">
        <canvas
          ref={canvasRef}
          className="w-full h-full cursor-crosshair"
          width={1920}
          height={1080}
          onWheel={handleWheel}
          onMouseDown={handleMouseDown}
          onMouseMove={handleMouseMove}
          onMouseUp={handleMouseUp}
        />
        <ToolBar />
      </div>
      <PropertiesPanel />
    </div>
  );
}

function renderGrid(
  ctx: CanvasRenderingContext2D,
  viewport: {centerX: number, centerY: number, scale: number},
  w: number,
  h: number
) {
  const gridSpacing = calculateGridSpacing(viewport.scale);  // Adaptive grid

  ctx.strokeStyle = '#333';
  ctx.lineWidth = 1;

  // Vertical lines
  const x0 = viewport.centerX - (w / 2) * viewport.scale;
  const x1 = viewport.centerX + (w / 2) * viewport.scale;
  const gridX0 = Math.floor(x0 / gridSpacing) * gridSpacing;
  for (let x = gridX0; x < x1; x += gridSpacing) {
    const screenX = (x - viewport.centerX) / viewport.scale + w / 2;
    ctx.beginPath();
    ctx.moveTo(screenX, 0);
    ctx.lineTo(screenX, h);
    ctx.stroke();
  }

  // Horizontal lines (similar)
  // ...
}

function calculateGridSpacing(scale: number): number {
  // scale = nm/pixel
  // Show grid at 1µm, 10µm, 100µm, 1mm intervals depending on zoom
  const targetPixels = 50;  // Target grid spacing in pixels
  const targetNm = scale * targetPixels;
  const exponent = Math.floor(Math.log10(targetNm));
  const base = Math.pow(10, exponent);
  const mantissa = targetNm / base;

  if (mantissa < 2) return base;
  if (mantissa < 5) return 2 * base;
  return 5 * base;
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–5)

**Day 1: Project initialization**
```bash
cargo init memslab-api
cd memslab-api
cargo add axum tokio serde serde_json sqlx uuid chrono tower-http jsonwebtoken bcrypt anyhow tracing
cargo add --features postgres,runtime-tokio-rustls,uuid,chrono,json sqlx
cargo add aws-sdk-s3 redis
```
- `src/main.rs` — Axum app with CORS, tracing, graceful shutdown
- `src/config.rs` — Environment config (DATABASE_URL, JWT_SECRET, S3_BUCKET, REDIS_URL)
- `src/state.rs` — AppState (PgPool, Redis, S3Client)
- `src/error.rs` — ApiError enum with IntoResponse
- `Dockerfile` — Multi-stage build (Rust builder + minimal runtime)
- `docker-compose.yml` — PostgreSQL 16, Redis 7, MinIO (local S3)

**Day 2-3: Database schema and migrations**
- `migrations/001_initial.sql` — All 11 tables
- `src/db/mod.rs` — Database pool initialization with connection pooling
- `src/db/models.rs` — All SQLx structs with FromRow, Serialize, Deserialize
- Run `sqlx migrate run`
- Seed script: PolyMUMPs PDK, basic materials (Si, SiO2, Poly-Si, Al), layer templates

**Day 4: Authentication system**
- `src/auth/mod.rs` — JWT middleware, token generation/validation
- `src/auth/oauth.rs` — Google/GitHub OAuth 2.0 handlers
- `src/api/handlers/auth.rs` — Register, login, OAuth callback, refresh, me
- Password hashing with bcrypt cost 12
- JWT: 24h access token + 30d refresh token
- Auth middleware extracting Claims from Bearer token

**Day 5: User and project CRUD**
- `src/api/handlers/users.rs` — Profile CRUD
- `src/api/handlers/projects.rs` — Create, list, get, update, delete, fork
- `src/api/handlers/orgs.rs` — Org management, member invites
- `src/api/router.rs` — Route definitions with auth guards
- Integration tests for auth flow and authorization

### Phase 2 — Process Simulator Core (Days 6–14)

**Day 6: Voxel data structures**
- `process-sim-core/src/voxel.rs` — VoxelGrid struct with sparse octree storage
- `process-sim-core/src/octree.rs` — Octree implementation with compression
- Memory layout: nodes store material ID + level-set value
- Tests: octree insert/query, memory compression ratio validation

**Day 7: Basic process steps (deposition, isotropic etch)**
- `process-sim-core/src/steps/deposit.rs` — Conformal/non-conformal deposition
- `process-sim-core/src/steps/etch_isotropic.rs` — Uniform etch rate
- `process-sim-core/src/mask.rs` — Mask layer application (Boolean AND with photoresist)
- Tests: single-layer deposition, masked etch, multi-layer stack

**Day 8: Level-set method framework**
- `process-sim-core/src/levelset.rs` — Level-set PDE solver (5th-order WENO, TVD RK3)
- `process-sim-core/src/spatial_derivatives.rs` — WENO5 scheme for ∇φ
- `process-sim-core/src/time_integration.rs` — TVD RK3 integrator
- Tests: sphere advection (volume conservation), curvature flow

**Day 9: Anisotropic wet etching**
- `process-sim-core/src/steps/etch_anisotropic.rs` — KOH/TMAH on Si<100>
- Crystallographic etch rates: <100> (R₀), <110> (0.5R₀), <111> (0.01R₀)
- Level-set with orientation-dependent speed F(n̂)
- Tests: V-groove formation, {111} plane emergence, undercut prediction

**Day 10: DRIE (Bosch process)**
- `process-sim-core/src/steps/drie.rs` — Alternating etch/passivation cycles
- Sidewall polymer deposition model
- Scallop height prediction from cycle time
- Tests: high-aspect-ratio trench (10:1), sidewall angle measurement

**Day 11-12: Advanced process steps**
- `process-sim-core/src/steps/oxidation.rs` — Thermal oxidation (Deal-Grove model)
- `process-sim-core/src/steps/cmp.rs` — Chemical-mechanical planarization
- `process-sim-core/src/steps/bond.rs` — Wafer bonding (fusion, anodic)
- `process-sim-core/src/steps/release.rs` — Sacrificial release (HF, XeF2) with diffusion
- Tests: oxidation rate vs temperature, CMP planarity, release time vs geometry

**Day 13: Process flow runner**
- `process-sim-core/src/runner.rs` — Sequential execution of process steps
- Checkpoint/resume support for long simulations
- Progress callback interface
- Tests: Full PolyMUMPs process flow (40 steps), validate against known structures

**Day 14: WASM compilation**
- `process-sim-wasm/` — WASM target crate
- `process-sim-wasm/src/lib.rs` — WASM bindings with wasm-bindgen
- Expose: `new()`, `add_step()`, `run()`, `get_voxel_data()`, `get_surface_mesh()`
- `wasm-pack build --target web --release`
- Tests: WASM vs native output equivalence

### Phase 3 — Mesh Generation + FEA Solver (Days 15–23)

**Day 15: Surface extraction**
- `mesh-gen/src/marching_cubes.rs` — Marching cubes for isosurface at φ=0
- Lookup table for 256 cube configurations
- Vertex interpolation for sub-voxel accuracy
- Tests: sphere meshing (compare volume to analytical), STL export

**Day 16: Tetrahedral meshing**
- `mesh-gen/src/gmsh_wrapper.rs` — Rust bindings to Gmsh API
- Surface mesh → 3D tetrahedral mesh with size field
- Mesh quality metrics (min/max aspect ratio, min dihedral angle)
- Tests: mesh convergence study (10K, 50K, 200K elements)

**Day 17-18: FEA framework (linear elasticity)**
- `fea-solver/src/element.rs` — Tet4 and Tet10 element formulations
- `fea-solver/src/assembly.rs` — Global stiffness matrix assembly (parallel)
- `fea-solver/src/boundary_conditions.rs` — Dirichlet and Neumann BC application
- `fea-solver/src/material.rs` — Isotropic linear elastic material
- Tests: cantilever beam (tip deflection vs analytical), pure bending

**Day 19: Sparse solver integration**
- `fea-solver/src/sparse_solver.rs` — MUMPS wrapper for K u = F
- `fea-solver/Cargo.toml` — Add `mumps-sys` dependency
- Handle symmetric positive definite matrices
- Tests: solve performance scaling (1K, 10K, 100K DOF)

**Day 20: Mechanical modal analysis**
- `fea-solver/src/modal.rs` — Mass matrix assembly
- Eigenvalue solver stub (requires ARPACK or shift-invert Lanczos)
- Extract first N natural frequencies and mode shapes
- Tests: simply-supported beam (compare frequencies to analytical)

**Day 21: Electrostatic solver**
- `fea-solver/src/electrostatic.rs` — Poisson equation assembly
- Capacitance extraction via charge integration
- Electric field post-processing
- Tests: parallel-plate capacitor, comb drive capacitance vs gap

**Day 22: Coupled electromechanical**
- `fea-solver/src/coupled_em.rs` — Iterative coupling loop
- Maxwell stress tensor → surface traction
- Pull-in detection via eigenvalue sign change
- Tests: parallel-plate actuator pull-in voltage

**Day 23: Result post-processing**
- `fea-solver/src/postprocess.rs` — Stress calculation, principal stresses
- Von Mises stress for failure prediction
- Result serialization to binary format (HDF5 or custom)
- Tests: stress concentration factor validation

### Phase 4 — Layout Editor Frontend (Days 24–29)

**Day 24: Canvas rendering setup**
- React component `LayoutEditor.tsx`
- Canvas 2D context with high-DPI support (devicePixelRatio)
- World-to-screen coordinate transform
- Pan and zoom with mouse wheel
- Tests: viewport transform correctness

**Day 25: Polygon rendering**
- Render filled polygons with stroke
- Layer visibility and color management
- R-tree spatial index (rbush) for visibility culling
- Tests: render performance with 10K polygons

**Day 26: Drawing tools**
- Rectangle, polygon, path tools
- Snap-to-grid (1nm, 10nm, 100nm, 1µm)
- Vertex editing (select, move, delete)
- Tests: polygon creation, editing, deletion

**Day 27: Boolean operations**
- Integrate `polygon-clipping` library (Vatti algorithm)
- Layer Boolean: AND, OR, XOR, NOT
- Offset (buffer) operation for proximity checks
- Tests: union, intersection, difference correctness

**Day 28: Parameterized cell library**
- Cell definition API: `Cell { name, parameters, generator }`
- Comb drive generator: finger width, gap, length, count
- Serpentine spring generator: beam width, turns, spacing
- Cantilever, membrane, flexure cells
- Tests: parameter sweep, constraint validation

**Day 29: GDS-II import/export**
- Parse GDS-II binary format (layer 0-255, datatype)
- Export layout to GDS-II with proper scaling (1nm database unit)
- Tests: round-trip (export → import → compare)

### Phase 5 — 3D Visualization + Integration (Days 30–34)

**Day 30: Three.js scene setup**
- React component `ProcessViewer3D.tsx`
- OrbitControls for camera manipulation
- Lighting: ambient + directional
- Tests: scene initialization, camera controls

**Day 31: Voxel volume rendering**
- Fetch voxel data from WASM or S3
- Convert to Three.js BufferGeometry
- Material coloring by layer
- Tests: render 100×100×100 voxel cube

**Day 32: Surface mesh rendering**
- Fetch triangle mesh from marching cubes
- Smooth shading with vertex normals
- Wireframe overlay toggle
- Tests: mesh quality visualization

**Day 33: FEA result visualization**
- Displacement field: color map on mesh
- Stress field: von Mises color scale
- Mode shape animation
- Tests: color map accuracy, animation smoothness

**Day 34: Process step animation**
- Step-by-step playback of process flow
- Show current layer state after each step
- Layer visibility controls
- Tests: 10-step process animation

### Phase 6 — Backend Integration + Job Queue (Days 35–38)

**Day 35: Process simulation worker**
- `src/workers/process_worker.rs` — Redis job consumer
- Fetch job → load project → run process sim → upload to S3 → update DB
- Progress streaming via WebSocket
- Tests: end-to-end server-side process sim

**Day 36: FEA simulation worker**
- `src/workers/fea_worker.rs` — FEA job consumer
- Fetch mesh → load BCs → run FEA → upload results → update DB
- Convergence monitoring
- Tests: end-to-end FEA solve

**Day 37: WebSocket real-time updates**
- `src/websocket/mod.rs` — Axum WebSocket handler
- Broadcast progress updates to connected clients
- Per-simulation room management
- Tests: multi-client broadcast

**Day 38: S3 presigned URLs**
- `src/storage/s3.rs` — Presigned GET for result download
- Presigned PUT for large file uploads (GDS-II)
- Tests: upload/download large files (100MB+)

### Phase 7 — PolyMUMPs PDK + DRC (Days 39–41)

**Day 39: PolyMUMPs PDK definition**
- Layer stack: Nitride, Poly0, Oxide1, Poly1, Oxide2, Poly2, Metal
- Material properties for each layer
- Process flow: 40 steps from bare Si to released structures
- Seed into database

**Day 40: Design rule checking engine**
- `drc-engine/src/rules.rs` — Min width, min spacing, enclosure checks
- `drc-engine/src/runner.rs` — Run all rules on layout
- Error reporting with violation locations
- Tests: intentional DRC violations, verify detection

**Day 41: DRC integration**
- API endpoint: POST /projects/:id/drc
- Frontend: display DRC errors as overlay on layout
- Highlight violations with red markers
- Tests: API integration, frontend display

### Phase 8 — Billing + Polish (Days 42)

**Day 42: Stripe integration**
- `src/billing/stripe.rs` — Checkout session creation
- Webhook handler for subscription events
- Plan enforcement in API middleware
- Frontend: Pricing page, subscription management
- Tests: mock Stripe webhooks, plan limits

**Final validation:** Run full workflow (layout → process → FEA) on 3 benchmark devices:
1. Comb-drive actuator (capacitance, pull-in voltage)
2. Cantilever beam resonator (first 3 modes)
3. Pressure sensor diaphragm (deflection, stress)

---

## Validation Plan

### Benchmark 1: Parallel-Plate Capacitor

**Geometry:** 100µm × 100µm plates, 2µm gap, Poly-Si 2µm thick
**Process:** Deposit Poly0 2µm, pattern, deposit Oxide1 2µm, deposit Poly1 2µm, pattern, release
**Expected:**
- Capacitance: C = ε₀ε_r A/d = (8.854e-12)(11.7)(100e-6)²/(2e-6) = **5.17 fF**
- Pull-in voltage: V_pi = sqrt(8kd³/27ε₀εrA) ≈ **25-30V** (depends on residual stress)

**Validation:**
- Process sim: Verify 2µm gap after release (±50nm)
- FEA capacitance: Match analytical within 5%
- FEA pull-in: Match within 10% (stress uncertainty)

### Benchmark 2: Cantilever Beam Resonator

**Geometry:** L=200µm, w=10µm, t=2µm Poly-Si
**Process:** Poly0 anchor, Oxide1 sacrificial, Poly1 beam, release
**Expected:**
- First mode (bending): f₁ = (1.875²/2π) × sqrt(EI/ρAL⁴) = (1.875²/2π) × sqrt((160e9)(10e-6)(2e-6)³/12 / (2330)(10e-6)(2e-6)(200e-6)⁴)) ≈ **18.5 kHz**
- Second mode: f₂ ≈ 6.27 × f₁ ≈ **116 kHz**
- Third mode: f₃ ≈ 17.55 × f₁ ≈ **325 kHz**

**Validation:**
- Process sim: Verify beam released (no residual oxide underneath)
- FEA modal: Match analytical frequencies within 3% (mesh convergence)

### Benchmark 3: Pressure Sensor Diaphragm

**Geometry:** 500µm × 500µm square diaphragm, t=10µm Si, 1bar pressure
**Process:** Bulk Si etch from backside (KOH), stop at membrane
**Expected:**
- Center deflection: w_center ≈ 0.0138 × P × a⁴ / (E × t³) = 0.0138 × 1e5 × (500e-6)⁴ / (160e9 × (10e-6)³) ≈ **5.4 µm**
- Max stress: σ_max ≈ 0.308 × P × a² / t² ≈ **385 MPa**

**Validation:**
- Process sim: Verify Si<111> plane angle (54.7°) at membrane edge
- FEA static: Match deflection within 10% (boundary condition sensitivity)
- FEA stress: Peak stress location at membrane center, magnitude within 15%

---

## Post-MVP Roadmap

### v1.1 — Microfluidics (Weeks 13-16)
- Fluidic FEA: Stokes flow, pressure-driven channels
- Droplet generation simulation (volume-of-fluid)
- Mixing efficiency analysis
- Microfluidic cell library (T-junction, Y-mixer, serpentine mixer)

### v1.2 — Reliability Analysis (Weeks 17-20)
- Fatigue life prediction (S-N curves, Miner's rule)
- Stiction risk calculator (surface energy vs restoring force)
- Shock simulation (transient acceleration loads)
- Packaging stress (die attach, wire bond, encapsulation models)

### v1.3 — Additional Foundry PDKs (Weeks 21-24)
- XFAB XMB10 (piezoelectric AlN)
- TSMC MEMS (high-volume production)
- Custom PDK builder (upload process calibration data)

### v1.4 — Foundry Integration (Weeks 25-28)
- Automated DRC with foundry-specific rules
- Process traveler document generation (PDF)
- Cost estimation API (area, process complexity, volume)
- Direct submission to foundry portals (MEMSCAP, XFAB APIs)

### v1.5 — AI-Assisted Design (Weeks 29-32)
- Topology optimization for minimal stress
- Generative design for target frequency response
- Parametric optimization via surrogate models

---

## Risk Mitigation

| Risk | Impact | Mitigation |
|------|--------|------------|
| FEA solver accuracy | High | Validate against analytical solutions + COMSOL cross-check |
| Process simulation calibration | High | Collaborate with foundry for metrology data, publish calibration methodology |
| WASM performance bottleneck | Medium | Optimize octree compression, offload to server for >100K voxels |
| Mesh quality issues | High | Implement adaptive refinement, quality metrics, user warnings |
| Foundry PDK licensing | Medium | Start with open foundries (PolyMUMPs), negotiate partnerships |

---

## Success Metrics

**Technical:**
- Process simulation accuracy: 3D structure dimensions within 5% of fabricated devices
- FEA accuracy: Resonant frequencies within 3% of measurements, capacitance within 5%
- Performance: 100K voxel process sim in <10s (WASM), 100K DOF FEA in <30s (server)

**Product:**
- 50 paying academic labs ($79/mo) by month 6
- 20 Pro users ($299/mo) by month 9
- 3 Enterprise contracts ($20K+/year) by month 12
- <2s layout editor response time for 1000-polygon designs
- >90% user satisfaction on simulation accuracy (post-fabrication survey)

**Business:**
- $100K ARR by month 12
- 80% gross margin (excluding compute)
- <5% monthly churn

---

## Critical Files

```
memslab/
├── process-sim-core/                      # Process simulation library (native + WASM)
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── voxel.rs                       # VoxelGrid with sparse octree
│   │   ├── octree.rs                      # Octree compression
│   │   ├── levelset.rs                    # Level-set PDE solver
│   │   ├── spatial_derivatives.rs         # WENO5 scheme
│   │   ├── time_integration.rs            # TVD RK3
│   │   ├── mask.rs                        # Mask layer operations
│   │   ├── steps/
│   │   │   ├── mod.rs
│   │   │   ├── deposit.rs                 # Conformal/directional deposition
│   │   │   ├── etch_isotropic.rs          # Uniform etch
│   │   │   ├── etch_anisotropic.rs        # KOH/TMAH crystallographic
│   │   │   ├── drie.rs                    # DRIE Bosch process
│   │   │   ├── oxidation.rs               # Deal-Grove thermal oxidation
│   │   │   ├── cmp.rs                     # Chemical-mechanical polish
│   │   │   ├── bond.rs                    # Wafer bonding
│   │   │   └── release.rs                 # Sacrificial release
│   │   └── runner.rs                      # Process flow executor
│   └── tests/
│       ├── basic_steps.rs                 # Single-step validation
│       └── polymumps_flow.rs              # Full PolyMUMPs process

├── process-sim-wasm/                      # WASM compilation target
│   ├── Cargo.toml
│   └── src/
│       └── lib.rs                         # WASM bindings (wasm-bindgen)

├── mesh-gen/                              # Mesh generation library
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── marching_cubes.rs              # Surface extraction
│   │   └── gmsh_wrapper.rs                # Gmsh API for tet meshing

├── fea-solver/                            # FEA solver library
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── element.rs                     # Tet4/Tet10 elements
│   │   ├── assembly.rs                    # Global stiffness/mass matrix
│   │   ├── boundary_conditions.rs         # Dirichlet/Neumann BCs
│   │   ├── material.rs                    # Material properties
│   │   ├── sparse_solver.rs               # MUMPS wrapper
│   │   ├── modal.rs                       # Eigenvalue solver
│   │   ├── electrostatic.rs               # Poisson solver
│   │   ├── coupled_em.rs                  # Electromechanical coupling
│   │   └── postprocess.rs                 # Stress, displacement, field extraction
│   └── tests/
│       ├── cantilever_beam.rs             # Mechanical validation
│       └── parallel_plate.rs              # Electrostatic validation

├── layout-engine/                         # GDS-II and DRC engine
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── gds_parser.rs                  # GDS-II binary parser
│   │   ├── gds_writer.rs                  # GDS-II export
│   │   ├── drc/
│   │   │   ├── mod.rs
│   │   │   ├── rules.rs                   # DRC rule definitions
│   │   │   └── runner.rs                  # DRC execution
│   │   └── cells/
│   │       ├── mod.rs                     # Parameterized cell API
│   │       ├── comb_drive.rs              # Comb drive generator
│   │       ├── serpentine_spring.rs       # Spring generator
│   │       └── cantilever.rs              # Cantilever generator

├── memslab-api/                           # Rust API server
│   ├── Cargo.toml
│   ├── src/
│   │   ├── main.rs                        # Axum app setup
│   │   ├── config.rs                      # Environment config
│   │   ├── state.rs                       # AppState (PgPool, Redis, S3)
│   │   ├── error.rs                       # ApiError types
│   │   ├── auth/
│   │   │   ├── mod.rs                     # JWT middleware
│   │   │   └── oauth.rs                   # OAuth handlers
│   │   ├── api/
│   │   │   ├── router.rs                  # Route definitions
│   │   │   ├── handlers/
│   │   │   │   ├── auth.rs                # Register, login
│   │   │   │   ├── users.rs               # Profile CRUD
│   │   │   │   ├── projects.rs            # Project CRUD
│   │   │   │   ├── process_simulation.rs  # Process sim API
│   │   │   │   ├── fea_simulation.rs      # FEA sim API
│   │   │   │   ├── pdks.rs                # PDK management
│   │   │   │   ├── materials.rs           # Material library
│   │   │   │   ├── billing.rs             # Stripe checkout
│   │   │   │   └── webhooks/
│   │   │   │       └── stripe.rs          # Stripe webhooks
│   │   │   └── ws/
│   │   │       ├── mod.rs                 # WebSocket handler
│   │   │       └── simulation_progress.rs # Progress streaming
│   │   ├── middleware/
│   │   │   └── plan_limits.rs             # Plan enforcement
│   │   ├── services/
│   │   │   ├── s3.rs                      # S3 helpers
│   │   │   └── usage.rs                   # Usage tracking
│   │   └── workers/
│   │       ├── mod.rs                     # Worker pool
│   │       ├── process_worker.rs          # Process sim worker
│   │       └── fea_worker.rs              # FEA worker
│   ├── migrations/
│   │   └── 001_initial.sql                # Database schema
│   └── tests/
│       ├── api_integration.rs
│       └── simulation_e2e.rs

├── model-service/                         # Python FastAPI (optimization)
│   ├── requirements.txt
│   ├── main.py                            # FastAPI app
│   ├── optimization/
│   │   ├── parametric.py                  # Parameter sweep optimization
│   │   └── topology.py                    # Topology optimization (future)
│   └── Dockerfile

├── frontend/                              # React frontend
│   ├── package.json
│   ├── vite.config.ts
│   ├── src/
│   │   ├── App.tsx
│   │   ├── main.tsx
│   │   ├── stores/
│   │   │   ├── authStore.ts
│   │   │   ├── projectStore.ts
│   │   │   ├── layoutStore.ts             # Layout editor state
│   │   │   ├── processStore.ts            # Process flow state
│   │   │   └── feaStore.ts                # FEA results state
│   │   ├── hooks/
│   │   │   ├── useSimulationProgress.ts   # WebSocket progress
│   │   │   └── useWasmProcess.ts          # WASM process sim
│   │   ├── lib/
│   │   │   ├── api.ts                     # Axios client
│   │   │   └── wasmLoader.ts              # WASM loading
│   │   ├── pages/
│   │   │   ├── Dashboard.tsx              # Project list
│   │   │   ├── Editor.tsx                 # Main editor
│   │   │   ├── PDKLibrary.tsx             # PDK browser
│   │   │   └── Billing.tsx                # Plan management
│   │   ├── components/
│   │   │   ├── LayoutEditor/
│   │   │   │   ├── Canvas.tsx             # Canvas 2D editor
│   │   │   │   ├── LayerPanel.tsx         # Layer controls
│   │   │   │   ├── ToolBar.tsx            # Drawing tools
│   │   │   │   └── PropertiesPanel.tsx    # Selection properties
│   │   │   ├── ProcessFlow/
│   │   │   │   ├── ProcessFlowEditor.tsx  # Process step sequencer
│   │   │   │   └── StepLibrary.tsx        # Available process steps
│   │   │   ├── ProcessViewer3D/
│   │   │   │   ├── Viewer3D.tsx           # Three.js 3D viewer
│   │   │   │   └── VoxelRenderer.tsx      # Voxel visualization
│   │   │   ├── FeaViewer/
│   │   │   │   ├── FeaViewer.tsx          # FEA result viewer
│   │   │   │   ├── DisplacementField.tsx  # Displacement visualization
│   │   │   │   ├── StressField.tsx        # Stress visualization
│   │   │   │   └── ModeShapes.tsx         # Modal visualization
│   │   │   └── billing/
│   │   │       ├── PlanCard.tsx
│   │   │       └── UsageMeter.tsx
│   │   └── data/
│   │       └── templates/                 # Device templates
│   └── public/
│       └── wasm/                          # WASM bundles

├── k8s/                                   # Kubernetes manifests
│   ├── api-deployment.yaml
│   ├── process-worker-deployment.yaml
│   ├── fea-worker-deployment.yaml
│   ├── postgres-statefulset.yaml
│   ├── redis-deployment.yaml
│   ├── model-service-deployment.yaml
│   └── ingress.yaml

├── docker-compose.yml                     # Local dev stack
├── Cargo.toml                             # Workspace root
└── .github/
    └── workflows/
        ├── ci.yml                         # Test + lint
        ├── wasm-build.yml                 # WASM build + deploy
        └── deploy.yml                     # Docker + K8s deploy
```

---

## Solver Validation Suite

### Benchmark 1: Parallel-Plate Capacitor

**Geometry:** 100µm × 100µm plates, 2µm gap, Poly-Si 2µm thick

**Process:** Deposit Poly0 2µm → pattern → deposit Oxide1 2µm → deposit Poly1 2µm → pattern → HF release

**Expected:**
- **Capacitance:** C = ε₀ε_r A/d = (8.854e-12)(11.7)(100e-6)²/(2e-6) = **5.17 fF**
- **Pull-in voltage:** V_pi = √(8kd³/27ε₀ε_rA) where k depends on residual stress ≈ **25-30V**

**Validation:**
- Process sim: Verify 2µm gap after release (±50nm tolerance from voxel resolution)
- FEA capacitance: Match analytical within 5% (mesh convergence dependent)
- FEA pull-in: Match within 10% (stress uncertainty is primary error source)

### Benchmark 2: Cantilever Beam Resonator

**Geometry:** L=200µm, w=10µm, t=2µm Poly-Si, E=160 GPa, ρ=2330 kg/m³

**Process:** Poly0 anchor → Oxide1 sacrificial → Poly1 beam → HF release

**Expected:**
- **First mode (bending):** f₁ = (λ₁²/2πL²)√(EI/ρA) where λ₁=1.875
  - f₁ = (1.875²/2π(200e-6)²) × √((160e9)(10e-6)(2e-6)³/12 / (2330)(10e-6)(2e-6))
  - f₁ ≈ **18.5 kHz**
- **Second mode:** f₂ ≈ 6.27 × f₁ ≈ **116 kHz**
- **Third mode:** f₃ ≈ 17.55 × f₁ ≈ **325 kHz**

**Validation:**
- Process sim: Verify beam fully released (no residual oxide underneath)
- FEA modal: Match analytical frequencies within 3% (requires mesh refinement at anchor)

### Benchmark 3: Pressure Sensor Diaphragm

**Geometry:** 500µm × 500µm square diaphragm, t=10µm Si<100>, 1 bar pressure differential

**Process:** Backside KOH etch to membrane thickness

**Expected:**
- **Center deflection:** w_center ≈ 0.0138 × P × a⁴ / (E × t³)
  - w_center = 0.0138 × 1e5 × (500e-6)⁴ / (160e9 × (10e-6)³) ≈ **5.4 µm**
- **Max stress:** σ_max ≈ 0.308 × P × a² / t² ≈ **385 MPa** (at membrane center)

**Validation:**
- Process sim: Verify Si<111> plane angle (54.7°) at membrane edge from KOH anisotropic etch
- FEA static: Match deflection within 10% (boundary condition sensitivity at edge)
- FEA stress: Peak stress location at membrane center, magnitude within 15%

---

## Verification

### Integration Testing Checklist

1. **Auth flow** — Register → login → JWT → authorized API calls → refresh → logout
2. **Project CRUD** — Create → update layout → save → reload → verify state
3. **Layout editing** — Place polygon → Boolean ops → parameterized cell → GDS export
4. **Process flow** — Add steps → configure parameters → run WASM sim → view 3D structure
5. **Server process** — Large structure (200K voxels) → job queued → worker runs → S3 upload
6. **Mesh generation** — Process structure → marching cubes → tet mesh → quality check
7. **FEA WASM** — Small mesh (10K DOF) → WASM FEA → results display
8. **FEA server** — Large mesh (200K DOF) → job queued → worker solves → results in S3
9. **Result visualization** — Load displacement field → color map → animate mode shapes
10. **DRC** — Layout with violations → run DRC → violations highlighted
11. **PDK workflow** — Load PolyMUMPs PDK → design → process sim → verify layer stack
12. **Plan limits** — Academic user → 600K voxel structure → blocked with upgrade prompt
13. **Billing** — Subscribe to Pro → Stripe checkout → webhook → plan updated
14. **Concurrent users** — 10 users simultaneously running FEA → all complete
15. **Error handling** — Non-convergent FEA (unstable) → error message → no crash

### SQL Verification Queries

```sql
-- 1. Process simulation throughput and success rate
SELECT
    execution_mode,
    status,
    COUNT(*) as count,
    AVG(wall_time_ms)::int as avg_time_ms,
    AVG(voxel_count) as avg_voxels
FROM process_simulations
WHERE created_at >= NOW() - INTERVAL '7 days'
GROUP BY execution_mode, status;

-- 2. FEA simulation analysis type distribution
SELECT
    analysis_type,
    COUNT(*) as count,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 1) as pct,
    AVG(wall_time_ms)::int as avg_time_ms
FROM fea_simulations
WHERE created_at >= NOW() - INTERVAL '30 days'
    AND status = 'completed'
GROUP BY analysis_type
ORDER BY count DESC;

-- 3. User plan distribution
SELECT
    plan,
    COUNT(*) as users,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 1) as pct
FROM users
GROUP BY plan
ORDER BY users DESC;

-- 4. Compute usage by user (billing period)
SELECT
    u.email,
    u.plan,
    SUM(ur.quantity) as total_hours,
    CASE u.plan
        WHEN 'academic' THEN 200
        WHEN 'pro' THEN 999999
        WHEN 'enterprise' THEN 999999
    END as limit_hours
FROM usage_records ur
JOIN users u ON ur.user_id = u.id
WHERE ur.period_start <= CURRENT_DATE
    AND ur.period_end >= CURRENT_DATE
    AND ur.record_type = 'compute_hours'
GROUP BY u.email, u.plan
ORDER BY total_hours DESC;

-- 5. PDK and material popularity
SELECT
    p.name as pdk_name,
    p.foundry,
    COUNT(DISTINCT pr.id) as projects_using
FROM pdks p
LEFT JOIN projects pr ON pr.pdk_id = p.id
GROUP BY p.id
ORDER BY projects_using DESC;

-- 6. Device type distribution
SELECT
    device_type,
    COUNT(*) as count
FROM projects
WHERE device_type IS NOT NULL
GROUP BY device_type
ORDER BY count DESC;
```

### Performance Benchmarks

| Operation | Target | Method |
|-----------|--------|--------|
| WASM process: 50K voxels, 10 steps | <5s | Browser benchmark (Chrome DevTools) |
| WASM process: 100K voxels, 20 steps | <15s | Browser benchmark |
| Server process: 500K voxels, 40 steps (PolyMUMPs) | <60s | Server timing, 4 cores |
| Marching cubes: 100K voxel structure | <2s | Surface mesh extraction |
| Gmsh tet meshing: 50K node mesh | <10s | Adaptive refinement to quality 0.6 |
| WASM FEA: 10K DOF mechanical static | <3s | Browser benchmark |
| WASM FEA: 50K DOF modal (10 modes) | <15s | Browser benchmark |
| Server FEA: 200K DOF mechanical static | <30s | MUMPS direct solver, 8 cores |
| Server FEA: 200K DOF modal (50 modes) | <120s | Shift-invert Lanczos |
| Layout editor: 1000 polygons render | 60 FPS | Canvas 2D performance |
| 3D viewer: 100K triangle mesh | 60 FPS | Three.js + WebGL |
| FEA visualization: 50K node displacement field | 30 FPS | Color-mapped mesh rendering |
| API: create process simulation | <150ms | p95 latency (Prometheus) |
| API: search PDKs | <50ms | p95 latency with full-text search |
| WASM bundle load (cached) | <500ms | Service worker cache hit |
| WASM bundle load (cold) | <4s | CDN delivery, 3MB gzipped |
| WebSocket: progress latency | <100ms | Worker emit to client receive |
| GDS-II parse: 10MB file | <2s | Rust parser |

---

## Deployment Architecture

### Infrastructure

```
┌───────────────────────────────────────────────┐
│              CloudFront CDN                    │
│  ┌─────────────┐  ┌──────────┐  ┌──────────┐ │
│  │ Frontend SPA│  │ WASM     │  │ Material │ │
│  │ (React)     │  │ Bundles  │  │ Library  │ │
│  └─────────────┘  └──────────┘  └──────────┘ │
└───────────────────────┬───────────────────────┘
                        │
            ┌───────────┴───────────┐
            │   AWS ALB (HTTPS)     │
            │   TLS termination     │
            └───────────┬───────────┘
                        │
      ┌─────────────────┼─────────────────┐
      ▼                 ▼                 ▼
┌──────────┐     ┌──────────┐     ┌──────────┐
│ API      │     │ API      │     │ API      │
│ Server   │     │ Server   │     │ Server   │
│ Pod ×3   │     │ (HPA)    │     │          │
└────┬─────┘     └────┬─────┘     └────┬─────┘
     │                │                │
     └────────────────┼────────────────┘
                      │
     ┌────────────────┼────────────────┐
     ▼                ▼                ▼
┌──────────┐   ┌──────────┐   ┌──────────┐
│PostgreSQL│   │  Redis   │   │ S3 Bucket│
│ (RDS)    │   │(ElastiC) │   │(Results, │
│Multi-AZ  │   │ Cluster  │   │ Meshes)  │
└──────────┘   └────┬─────┘   └──────────┘
                    │
     ┌──────────────┼──────────────┐
     ▼              ▼              ▼
┌──────────┐  ┌──────────┐  ┌──────────┐
│ Process  │  │ Process  │  │ FEA      │
│ Worker   │  │ Worker   │  │ Worker   │
│c7g.2xl   │  │c7g.2xl   │  │c7g.4xl   │
│HPA: 2-10 │  │          │  │HPA: 1-8  │
└──────────┘  └──────────┘  └──────────┘
```

### Scaling Strategy

- **API servers**: HPA at 70% CPU, min 3, max 10 pods
- **Process workers**: HPA based on Redis queue depth (scale up at 5 jobs, down after 5min idle)
- **FEA workers**: HPA based on queue depth with larger instance types (c7g.4xlarge for 200K+ DOF)
- **Worker instances**: AWS Graviton3 (c7g) for ARM64 Rust performance
- **Database**: RDS PostgreSQL r6g.xlarge with read replicas for PDK/material queries
- **Redis**: ElastiCache r6g.large cluster, 1 primary + 2 replicas

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
        { "Days": 90, "StorageClass": "GLACIER_IR" }
      ],
      "Expiration": { "Days": 365 }
    },
    {
      "ID": "mesh-files-lifecycle",
      "Prefix": "meshes/",
      "Status": "Enabled",
      "Transitions": [
        { "Days": 7, "StorageClass": "STANDARD_IA" }
      ],
      "Expiration": { "Days": 90 }
    },
    {
      "ID": "gds-files-keep",
      "Prefix": "layouts/",
      "Status": "Enabled",
      "Transitions": [
        { "Days": 180, "StorageClass": "STANDARD_IA" }
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

| Feature | Description | Priority | Timeline |
|---------|-------------|----------|----------|
| Microfluidics Module | Stokes flow solver, droplet generation (VOF), mixing efficiency analysis, microfluidic cell library (T-junction, Y-mixer) | High | Weeks 13-16 |
| Reliability Analysis | Fatigue life (S-N curves), stiction risk calculator, shock/vibration simulation, packaging stress (die attach, wire bond) | High | Weeks 17-20 |
| Additional Foundry PDKs | XFAB XMB10 (piezoelectric AlN), TSMC MEMS (high-volume), Teledyne DALSA (optical MEMS), custom PDK builder | High | Weeks 21-24 |
| Thermal Analysis Enhancement | Transient thermal simulation, Joule heating, thermal-mechanical coupling for thermal actuators, IR camera calibration | Medium | Weeks 25-28 |
| Foundry Integration | Automated DRC with foundry rules, process traveler PDF generation, cost estimation API, direct submission to foundry portals (MEMSCAP, XFAB) | Medium | Weeks 29-32 |
| Optical MEMS Support | Ray tracing for micro-mirrors, waveguide mode solver, grating diffraction analysis, integrated photonics | Medium | Weeks 33-36 |
| Piezoelectric/Piezoresistive FEA | Full PZT/AlN piezoelectric coupling, piezoresistive gauge factor calculation, acoustic wave propagation | Low | Weeks 37-40 |
| AI-Assisted Design | Topology optimization for minimal stress, generative design for target frequency response, parametric optimization via surrogate models | Low | Weeks 41-44 |

---

## Risk Mitigation

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| FEA solver accuracy for nonlinear materials | High | Medium | Validate against COMSOL cross-checks, publish validation benchmark suite, partner with university for experimental validation |
| Process simulation calibration requires foundry data | High | High | Start with published literature data (KOH rates, DRIE profiles), partner with PolyMUMPs for calibration, offer calibration service for enterprise customers |
| WASM performance bottleneck for large structures | Medium | Medium | Optimize octree compression (10-50× achieved), offload >100K voxels to server, use Web Workers for parallel processing |
| Mesh quality issues cause FEA divergence | High | Medium | Implement adaptive refinement, quality metrics (aspect ratio, dihedral angle), user warnings, automatic remeshing |
| Foundry PDK licensing restrictions | Medium | High | Start with open foundries (PolyMUMPs), negotiate partnerships with XFAB/TSMC, offer bring-your-own-PDK for enterprise |
| Level-set method stability for complex geometries | Medium | Low | Use 5th-order WENO (proven stable), adaptive timestep, fallback to simpler schemes if divergence detected |
| Competitive response from CoventorWare/IntelliSuite | Medium | High | Move fast on cloud-native features they can't match, target underserved segments (academic, startups), build community |

---

## Launch Strategy

### Week 1-2: Alpha Release (Internal Testing)
- Deploy to staging environment
- Internal team stress testing: create 20 test projects across device types
- Performance profiling and optimization
- Bug triage and critical fixes

### Week 3-4: Private Beta (50 Users)
- Invite 25 academic labs + 25 MEMS engineers from personal networks
- Beta feedback form: usability, accuracy, feature requests
- Weekly office hours for beta users (Zoom)
- Monitor error rates, simulation success rates, usage patterns

### Week 5-6: Public Beta (Open Registration)
- Landing page with product video, interactive demo
- Blog post: "MEMSLab: Democratizing MEMS Design"
- Post to:
  - Hacker News
  - r/MEMS, r/engineering
  - LinkedIn (target MEMS engineers, university professors)
  - MEMS Industry Group forum
- Academic plan promo: 3 months free for universities
- Target: 200 signups, 100 active projects

### Week 7-8: Launch (v1.0)
- Full production deployment with monitoring
- Public launch announcement:
  - Press release to EE Times, Semiconductor Engineering
  - Conference submissions (IEEE MEMS, Transducers)
  - Webinar: "Introduction to MEMSLab" (record and publish)
- Outreach:
  - Contact 50 MEMS design groups at universities
  - Contact 20 MEMS product companies for enterprise trials
- Pricing goes live:
  - Academic: $79/mo (10 seats, 200 compute hours/mo)
  - Pro: $299/mo (unlimited)
  - Enterprise: Custom (starting $20K/year)
- Target: 500 users, 50 paying (10% conversion), $15K MRR

### Month 3-6: Growth Phase
- Content marketing:
  - Tutorial series: "Design a Comb Drive Actuator" (5 parts)
  - Case studies: "120dB SPL MEMS Microphone Design"
  - Validation reports: "MEMSLab vs. Measured Devices"
- Partnerships:
  - MEMSCAP: official PolyMUMPs design partner
  - Universities: integrate into MEMS courses (MIT, Berkeley, Stanford, CMU)
- Features:
  - v1.1: Microfluidics module
  - v1.2: Reliability analysis
- Target: 2000 users, 100 paying (20 academic, 70 pro, 10 enterprise), $50K MRR

### Month 6-12: Scale Phase
- Additional PDKs: XFAB XMB10, TSMC MEMS
- Foundry integration: direct submission API
- Sales team: hire 2 MEMS application engineers for enterprise
- Conference presence: booth at IEEE MEMS, Transducers
- Target: 5000 users, 200 paying (30 academic, 150 pro, 20 enterprise), $100K MRR ($1.2M ARR)

---

## Success Metrics

### Technical Validation
- **Process simulation accuracy:** 3D structure dimensions within 5% of SEM measurements for 10 fabricated devices
- **FEA accuracy:** Resonant frequencies within 3%, capacitance within 5%, pull-in voltage within 10% of measurements
- **Performance:** 100K voxel process in <10s (WASM), 200K DOF FEA in <30s (server), 60 FPS layout editor

### Product Metrics
- **User acquisition:** 5000 registered users by month 12
- **Activation:** 60% of new users complete first simulation within 7 days
- **Retention:** 70% MAU/WAU ratio (weekly active users return monthly)
- **Conversion:** 10% free → paid conversion rate, 5% academic → pro upgrade rate
- **NPS:** >40 (promoters - detractors)

### Business Metrics
- **Revenue:** $100K ARR by month 12 (200 paying users × $500 ARPU)
- **CAC:** <$200 (target payback in 4 months at $50 ARPU)
- **LTV/CAC:** >3 (assume 24-month retention)
- **Gross margin:** 80% (excluding compute, which scales with usage-based pricing)
- **Churn:** <5% monthly (enterprise <2%, pro <7%, academic <10%)
