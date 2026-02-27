# 36. PhotonPath — Cloud Photonics Simulation & PIC Design Platform

## Implementation Plan

**MVP Scope:** Browser-based photonic layout editor with waveguide routing (straight, curved, taper) and component placement (directional coupler, ring resonator, grating coupler, MMI) rendered via Canvas2D with R-tree spatial indexing, 2D GPU-accelerated FDTD electromagnetic simulation engine (wgpu compute shaders) supporting mode source, plane wave excitation, Gaussian pulse, PML absorbing boundaries, dispersive materials (Drude-Lorentz), and simulation domains up to 2M mesh cells executed entirely in cloud with real-time progress streaming via WebSocket, finite-element eigenmode solver computing effective index (neff), group index (ng), loss, and mode profiles for rectangular/rib waveguide cross-sections with wavelength sweep for dispersion analysis, real-time 2D electromagnetic field visualization (Ex, Ey, Ez, |E|²) rendered via WebGL with interactive color maps (viridis, plasma, RdBu) and cross-section slicing, S-parameter extraction from FDTD port monitors with automated computation of insertion loss, return loss, 3-dB bandwidth, and Touchstone export for 2-port components, comprehensive optical material database (Si, SiO2, Si3N4, InP, GaAs) with refractive index and extinction coefficient (n,k) data from Palik/Aspnes references stored in PostgreSQL JSONB, SiEPIC open PDK integration with pre-defined waveguide cross-sections and design rules, GDS-II export of photonic layouts for foundry tape-out, Stripe billing with three tiers (Free/Academic $0 / Pro $199/mo / Team $599/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| FDTD Engine | Rust + wgpu 0.19 | GPU compute shaders (WGSL), portable across CUDA/Metal/Vulkan |
| Eigenmode Solver | Rust + FEM | Sparse eigenvalue solver via ARPACK bindings, Delaunay mesh generation |
| Field Export | HDF5 | Electromagnetic field data storage via `hdf5` crate |
| Material Database | PostgreSQL 16 | JSONB storage for wavelength-indexed n,k data with Lorentz model fits |
| Database | PostgreSQL 16 | Projects, users, simulations, PDKs, components, S-parameter metadata |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | Cloudflare R2 | GDS files, HDF5 field data, S-parameter Touchstone files |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| Layout Editor | Canvas2D + R-tree | Custom waveguide router with snap-to-grid and DRC |
| Field Viewer | WebGL 2.0 (custom) | GPU-accelerated field rendering with real-time color mapping |
| Eigenmode Plots | Plotly.js + WebGL | 2D mode profile rendering, dispersion curves |
| Real-time | WebSocket (Axum) | Live FDTD simulation progress, field preview streaming |
| Job Queue | Redis 7 + Tokio tasks | FDTD job scheduling, eigenmode batch processing |
| GPU Compute | Kubernetes (EKS) | NVIDIA A100/A10G nodes, Karpenter auto-scaling for GPU pods |
| Billing | Stripe | Checkout Sessions, Customer Portal, usage metering for GPU hours |
| CDN | CloudFront | Frontend assets, material database API cache |
| Monitoring | Prometheus + Grafana, Sentry | GPU utilization metrics, FDTD convergence tracking, error tracking |
| CI/CD | GitHub Actions | Rust tests, wgpu shader validation, Docker image build |

### Key Architecture Decisions

1. **wgpu for portable GPU compute rather than CUDA-only**: wgpu compiles to Vulkan/Metal/DX12/WebGPU, allowing the FDTD engine to run on NVIDIA (CUDA backend), AMD (Vulkan), Apple Silicon (Metal), and cloud providers without NVIDIA lock-in. The WGSL shader language is simpler than CUDA C++ and compiles at runtime, avoiding binary compatibility issues. Performance is within 10-15% of hand-tuned CUDA for structured-grid FDTD.

2. **2D FDTD for MVP, 3D post-launch**: 2D FDTD (Transverse Electric or Transverse Magnetic modes) covers 80%+ of silicon photonics design tasks (waveguide analysis, couplers, ring resonators, grating couplers with effective index approximation). 2D simulation completes in seconds vs. hours for 3D, enabling rapid iteration. The FDTD kernel architecture is designed for 3D but launches 2D compute workgroups for MVP. Post-MVP, 3D support requires only changing mesh generation and compute dispatch logic.

3. **Cloud-only execution (no browser WASM solver)**: Unlike SpiceSim's WASM client-side solver, photonics FDTD requires sustained multi-second GPU compute with 100MB+ memory (2M cells × 3 field components × float32). WebGPU cannot access discrete GPUs in most browsers, and integrated GPUs throttle after seconds. Cloud execution with WebSocket progress streaming provides predictable performance, allows larger simulations, and simplifies billing.

4. **Eigenmode solver using FEM rather than mode-matching**: Finite-element eigenmode solving handles arbitrary waveguide cross-sections (curved boundaries, anisotropic materials, multi-layer stacks) which are common in photonics. Mode-matching is limited to piecewise-constant index profiles. ARPACK sparse eigenvalue solver computes 5-20 modes for a 10K-DOF mesh in <1 second, fast enough for interactive design.

5. **R2 object storage for field data and GDS files**: Cloudflare R2 offers S3-compatible API with zero egress fees, critical for photonics where a single 3D FDTD simulation can generate 1-10 GB of field data. Users frequently download HDF5 field exports for post-processing in Python, and zero-egress pricing makes this viable. GDS files are typically <10 MB but accessed frequently during tape-out preparation.

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";

-- Users
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email TEXT UNIQUE NOT NULL,
    password_hash TEXT,
    name TEXT NOT NULL,
    avatar_url TEXT,
    auth_provider TEXT DEFAULT 'email',
    auth_provider_id TEXT,
    institution TEXT,  -- University name for academic verification
    plan TEXT NOT NULL DEFAULT 'free',
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    gpu_hours_used REAL DEFAULT 0.0,
    gpu_hours_limit REAL DEFAULT 2.0,  -- Varies by plan
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX users_email_idx ON users(email);
CREATE INDEX users_stripe_idx ON users(stripe_customer_id);

-- Organizations
CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    slug TEXT UNIQUE NOT NULL,
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    plan TEXT NOT NULL DEFAULT 'free',
    seat_count INTEGER DEFAULT 1,
    billing_email TEXT,
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    settings JSONB DEFAULT '{}',  -- {default_pdk, wavelength_units, field_colormap}
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
    role TEXT NOT NULL DEFAULT 'member',
    joined_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(org_id, user_id)
);
CREATE INDEX org_members_org_idx ON org_members(org_id);
CREATE INDEX org_members_user_idx ON org_members(user_id);

-- PDKs (Process Design Kits)
CREATE TABLE pdks (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,  -- "SiEPIC", "AIM Photonics", "IMEC iSiPP"
    foundry TEXT,  -- "ANT", "AIM", "IMEC"
    platform TEXT NOT NULL,  -- "soi" | "sin" | "inp" | "polymer" | "custom"
    layer_stack JSONB NOT NULL,  -- [{layer, material, thickness, z_offset}]
    design_rules JSONB NOT NULL,  -- {min_width, min_bend_radius, min_gap} per layer
    waveguide_profiles JSONB NOT NULL,  -- [{name, width, height, etch_depth, neff_estimate}]
    component_library_url TEXT,  -- S3 URL for pre-characterized components
    documentation_url TEXT,
    is_open BOOLEAN DEFAULT true,
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX pdks_platform_idx ON pdks(platform);

-- Projects
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    visibility TEXT NOT NULL DEFAULT 'private',  -- private | org | public
    pdk_id UUID REFERENCES pdks(id) ON DELETE SET NULL,
    platform TEXT NOT NULL DEFAULT 'soi',  -- soi | sin | inp | polymer | custom
    forked_from UUID REFERENCES projects(id) ON DELETE SET NULL,
    thumbnail_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);
CREATE INDEX projects_org_idx ON projects(org_id);
CREATE INDEX projects_updated_idx ON projects(updated_at DESC);

-- Components (individual photonic components)
CREATE TABLE components (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    component_type TEXT NOT NULL,  -- waveguide | coupler | resonator | modulator | filter | custom
    geometry_params JSONB NOT NULL,  -- {width, length, gap, radius, ...} specific to component type
    layout_data_url TEXT,  -- S3 URL for GDS cell data
    sparam_model_url TEXT,  -- S3 URL for Touchstone file (nullable)
    port_definitions JSONB DEFAULT '[]',  -- [{name, position, direction, mode_index}]
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX components_project_idx ON components(project_id);
CREATE INDEX components_type_idx ON components(component_type);

-- FDTD Simulations
CREATE TABLE fdtd_simulations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    component_id UUID REFERENCES components(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    dimension TEXT NOT NULL DEFAULT '2d',  -- 2d | 3d
    domain JSONB NOT NULL,  -- {x_span, y_span, z_span, mesh_accuracy, pml_layers}
    sources JSONB NOT NULL,  -- [{type, position, wavelength_center, wavelength_span, mode_index}]
    monitors JSONB NOT NULL,  -- [{type, position, field_components, frequency_points}]
    materials JSONB NOT NULL,  -- [{region, material_name, custom_nk_url}]
    mesh_override_regions JSONB DEFAULT '[]',  -- [{region, dx, dy, dz}]
    status TEXT NOT NULL DEFAULT 'queued',  -- queued | meshing | running | completed | failed | cancelled
    gpu_type TEXT,  -- a100 | a10g | t4
    gpu_hours REAL,
    estimated_cost_usd REAL,
    field_data_url TEXT,  -- S3 URL for HDF5 field data
    log_url TEXT,  -- S3 URL for simulation log
    progress_percent REAL DEFAULT 0.0,
    current_timestep INTEGER,
    total_timesteps INTEGER,
    created_by UUID NOT NULL REFERENCES users(id),
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX fdtd_project_idx ON fdtd_simulations(project_id);
CREATE INDEX fdtd_status_idx ON fdtd_simulations(status);
CREATE INDEX fdtd_created_idx ON fdtd_simulations(created_at DESC);

-- Eigenmode Jobs
CREATE TABLE eigenmode_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    cross_section JSONB NOT NULL,  -- Geometry and material definition
    wavelength_range JSONB NOT NULL,  -- {start, stop, points}
    num_modes INTEGER NOT NULL DEFAULT 5,
    boundary_conditions JSONB DEFAULT '{"type": "PEC"}',  -- PEC | PMC | PML
    status TEXT NOT NULL DEFAULT 'queued',  -- queued | running | completed | failed
    results JSONB,  -- [{mode_index, neff, ng, loss, confinement}] array
    mode_profiles_url TEXT,  -- S3 URL for field data per mode
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at TIMESTAMPTZ
);
CREATE INDEX eigenmode_project_idx ON eigenmode_jobs(project_id);
CREATE INDEX eigenmode_status_idx ON eigenmode_jobs(status);

-- S-Parameter Models
CREATE TABLE sparam_models (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    component_id UUID NOT NULL REFERENCES components(id) ON DELETE CASCADE,
    simulation_id UUID REFERENCES fdtd_simulations(id) ON DELETE SET NULL,
    port_count INTEGER NOT NULL,
    frequency_points INTEGER NOT NULL,
    sparam_data_url TEXT NOT NULL,  -- S3 URL for Touchstone file
    metrics JSONB,  -- {insertion_loss, return_loss, bandwidth_3dB, center_wavelength}
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX sparam_component_idx ON sparam_models(component_id);

-- Circuit Simulations
CREATE TABLE circuit_simulations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    topology JSONB NOT NULL,  -- {components: [{id, sparam_id, connections}]}
    wavelength_range JSONB NOT NULL,
    status TEXT NOT NULL DEFAULT 'queued',
    results_url TEXT,  -- S3 URL
    monte_carlo_config JSONB,  -- {parameter_variations, num_samples}
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at TIMESTAMPTZ
);
CREATE INDEX circuit_project_idx ON circuit_simulations(project_id);

-- Optimization Jobs
CREATE TABLE optimization_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    simulation_id UUID REFERENCES fdtd_simulations(id) ON DELETE SET NULL,
    method TEXT NOT NULL,  -- adjoint | parametric | topology | bayesian
    objective JSONB NOT NULL,
    constraints JSONB DEFAULT '[]',
    parameter_space JSONB NOT NULL,
    status TEXT NOT NULL DEFAULT 'queued',
    iterations_completed INTEGER DEFAULT 0,
    best_fom REAL,  -- Figure of merit
    history_url TEXT,  -- S3 URL for iteration history
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at TIMESTAMPTZ
);
CREATE INDEX optim_project_idx ON optimization_jobs(project_id);

-- Layouts
CREATE TABLE layouts (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    gds_data_url TEXT,  -- S3 URL for GDS-II file
    components_placed JSONB DEFAULT '[]',  -- [{component_id, position, rotation}]
    waveguide_routes JSONB DEFAULT '[]',  -- [{start, end, waypoints, width, layer}]
    drc_status TEXT DEFAULT 'unknown',  -- clean | violations | unknown
    drc_violation_count INTEGER DEFAULT 0,
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX layouts_project_idx ON layouts(project_id);

-- Materials Database
CREATE TABLE materials (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT UNIQUE NOT NULL,  -- "Si", "SiO2", "Si3N4", "InP"
    category TEXT NOT NULL,  -- semiconductor | dielectric | metal | polymer
    nk_data JSONB NOT NULL,  -- [{wavelength_um, n, k}] array
    lorentz_fit JSONB,  -- {poles: [{eps_inf, omega_0, gamma}]}
    thermo_optic_coeff REAL,  -- dn/dT in 1/K
    references TEXT,  -- "Palik 1985", "Aspnes 1983"
    is_builtin BOOLEAN DEFAULT true,
    uploaded_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX materials_name_idx ON materials(name);
CREATE INDEX materials_category_idx ON materials(category);

-- Comments
CREATE TABLE comments (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    author_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    body TEXT NOT NULL,
    anchor_type TEXT,  -- layout | simulation | eigenmode
    anchor_data JSONB,  -- {x, y} for layout position, {field, time} for simulation
    resolved BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX comments_project_idx ON comments(project_id);

-- Usage Records (for billing)
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,  -- gpu_hours | storage_gb | simulation_count
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
    pub institution: Option<String>,
    pub plan: String,
    pub stripe_customer_id: Option<String>,
    pub stripe_subscription_id: Option<String>,
    pub gpu_hours_used: f32,
    pub gpu_hours_limit: f32,
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
    pub visibility: String,
    pub pdk_id: Option<Uuid>,
    pub platform: String,
    pub forked_from: Option<Uuid>,
    pub thumbnail_url: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Pdk {
    pub id: Uuid,
    pub name: String,
    pub foundry: Option<String>,
    pub platform: String,
    pub layer_stack: serde_json::Value,
    pub design_rules: serde_json::Value,
    pub waveguide_profiles: serde_json::Value,
    pub component_library_url: Option<String>,
    pub documentation_url: Option<String>,
    pub is_open: bool,
    pub created_by: Option<Uuid>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Component {
    pub id: Uuid,
    pub project_id: Uuid,
    pub name: String,
    pub component_type: String,
    pub geometry_params: serde_json::Value,
    pub layout_data_url: Option<String>,
    pub sparam_model_url: Option<String>,
    pub port_definitions: serde_json::Value,
    pub created_by: Uuid,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct FdtdSimulation {
    pub id: Uuid,
    pub project_id: Uuid,
    pub component_id: Option<Uuid>,
    pub name: String,
    pub dimension: String,
    pub domain: serde_json::Value,
    pub sources: serde_json::Value,
    pub monitors: serde_json::Value,
    pub materials: serde_json::Value,
    pub mesh_override_regions: serde_json::Value,
    pub status: String,
    pub gpu_type: Option<String>,
    pub gpu_hours: Option<f32>,
    pub estimated_cost_usd: Option<f32>,
    pub field_data_url: Option<String>,
    pub log_url: Option<String>,
    pub progress_percent: f32,
    pub current_timestep: Option<i32>,
    pub total_timesteps: Option<i32>,
    pub created_by: Uuid,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct EigenmodeJob {
    pub id: Uuid,
    pub project_id: Uuid,
    pub name: String,
    pub cross_section: serde_json::Value,
    pub wavelength_range: serde_json::Value,
    pub num_modes: i32,
    pub boundary_conditions: serde_json::Value,
    pub status: String,
    pub results: Option<serde_json::Value>,
    pub mode_profiles_url: Option<String>,
    pub created_by: Uuid,
    pub created_at: DateTime<Utc>,
    pub completed_at: Option<DateTime<Utc>>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Material {
    pub id: Uuid,
    pub name: String,
    pub category: String,
    pub nk_data: serde_json::Value,
    pub lorentz_fit: Option<serde_json::Value>,
    pub thermo_optic_coeff: Option<f32>,
    pub references: Option<String>,
    pub is_builtin: bool,
    pub uploaded_by: Option<Uuid>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct FdtdSimulationRequest {
    pub name: String,
    pub dimension: String,
    pub domain: FdtdDomain,
    pub sources: Vec<FdtdSource>,
    pub monitors: Vec<FdtdMonitor>,
    pub materials: Vec<MaterialRegion>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct FdtdDomain {
    pub x_span: f64,  // microns
    pub y_span: f64,
    pub z_span: Option<f64>,  // None for 2D
    pub mesh_accuracy: u8,  // 1-5, higher = finer mesh
    pub pml_layers: u32,  // Typically 8-12
}

#[derive(Debug, Deserialize, Serialize)]
pub struct FdtdSource {
    pub source_type: String,  // mode | plane_wave | gaussian_beam | dipole
    pub position: [f64; 3],
    pub wavelength_center: f64,  // microns
    pub wavelength_span: f64,
    pub mode_index: Option<u32>,  // For mode sources
    pub direction: Option<String>,  // forward | backward
}

#[derive(Debug, Deserialize, Serialize)]
pub struct FdtdMonitor {
    pub monitor_type: String,  // point | line | plane | volume
    pub position: [f64; 3],
    pub size: Option<[f64; 3]>,
    pub field_components: Vec<String>,  // Ex, Ey, Ez, Hx, Hy, Hz
    pub frequency_points: u32,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct MaterialRegion {
    pub region: String,  // background | box | sphere | custom
    pub geometry: serde_json::Value,
    pub material_name: String,
}
```

---

## FDTD Solver Architecture Deep-Dive

### Governing Equations and Discretization

PhotonPath's FDTD solver implements Yee's algorithm for solving Maxwell's curl equations on a staggered Cartesian grid:

```
∂E/∂t = (1/ε₀εᵣ) ∇×H - σE/(ε₀εᵣ)
∂H/∂t = -(1/μ₀μᵣ) ∇×E
```

**Yee grid staggering**: Electric field components (Ex, Ey, Ez) and magnetic field components (Hx, Hy, Hz) are offset by half a grid cell in space and time, providing second-order accuracy:

```
Ex(i+½, j, k) at time n
Ey(i, j+½, k) at time n
Ez(i, j, k+½) at time n
Hx(i, j+½, k+½) at time n+½
Hy(i+½, j, k+½) at time n+½
Hz(i+½, j+½, k) at time n+½
```

**Update equations** (2D TM mode example for MVP):

```
Ez(i,j)ⁿ⁺¹ = Ca(i,j)·Ez(i,j)ⁿ + Cb(i,j)·[Hx(i,j+½)ⁿ⁺½ - Hx(i,j-½)ⁿ⁺½
                                         - Hy(i+½,j)ⁿ⁺½ + Hy(i-½,j)ⁿ⁺½] / Δx

Hx(i,j+½)ⁿ⁺³/² = Hx(i,j+½)ⁿ⁺½ - (Δt/μ₀Δy)[Ez(i,j+1)ⁿ⁺¹ - Ez(i,j)ⁿ⁺¹]

Hy(i+½,j)ⁿ⁺³/² = Hy(i+½,j)ⁿ⁺½ + (Δt/μ₀Δx)[Ez(i+1,j)ⁿ⁺¹ - Ez(i,j)ⁿ⁺¹]

where:
Ca(i,j) = (2ε(i,j) - σ(i,j)Δt) / (2ε(i,j) + σ(i,j)Δt)
Cb(i,j) = 2Δt / (2ε(i,j) + σ(i,j)Δt)
```

**Stability criterion** (Courant condition):

```
Δt ≤ 1 / (c √(1/Δx² + 1/Δy² + 1/Δz²))

For typical photonics mesh (Δx = Δy = 10 nm, c = 3×10⁸ m/s):
Δt ≤ 16.7 attoseconds
```

**PML absorbing boundaries**: Perfectly Matched Layer uses complex coordinate stretching to absorb outgoing waves without reflection. Uniaxial PML (UPML) formulation with polynomial grading:

```
σ(x) = σ_max · (x/d)ᵐ    for x in [0, d] (PML thickness)
```

Typical parameters: `m = 3`, `σ_max = 0.8·(m+1)/(η₀·Δx)`, `d = 10 cells`.

**Dispersive materials** (Drude-Lorentz): Silicon has wavelength-dependent refractive index. Auxiliary Differential Equation (ADE) method updates polarization density alongside E-field:

```
ε(ω) = ε_∞ + Σ Δεₖ·ωₖ² / (ωₖ² - ω² - jγₖω)

Time-domain update:
Pₖⁿ⁺¹ = C₁ₖ·Pₖⁿ + C₂ₖ·Pₖⁿ⁻¹ + C₃ₖ·(Eⁿ⁺¹ + Eⁿ)
```

### 2D FDTD GPU Kernel (wgpu/WGSL)

```rust
// fdtd-engine/src/kernels/update_e_field.wgsl

@group(0) @binding(0) var<storage, read_write> ez: array<f32>;
@group(0) @binding(1) var<storage, read> hx: array<f32>;
@group(0) @binding(2) var<storage, read> hy: array<f32>;
@group(0) @binding(3) var<storage, read> ca: array<f32>;  // Material coefficients
@group(0) @binding(4) var<storage, read> cb: array<f32>;
@group(1) @binding(0) var<uniform> params: SimParams;

struct SimParams {
    nx: u32,
    ny: u32,
    dx: f32,
    dy: f32,
    dt: f32,
};

@compute @workgroup_size(16, 16)
fn update_e_field(@builtin(global_invocation_id) id: vec3<u32>) {
    let i = id.x;
    let j = id.y;

    if (i == 0u || i >= params.nx - 1u || j == 0u || j >= params.ny - 1u) {
        return;  // Skip boundary (PML handled separately)
    }

    let idx = i + j * params.nx;
    let idx_hx_j_plus = i + (j + 1u) * params.nx;
    let idx_hx_j_minus = i + (j - 1u) * params.nx;
    let idx_hy_i_plus = (i + 1u) + j * params.nx;
    let idx_hy_i_minus = (i - 1u) + j * params.nx;

    let curl_h = (hx[idx_hx_j_plus] - hx[idx_hx_j_minus]) / (2.0 * params.dy)
               - (hy[idx_hy_i_plus] - hy[idx_hy_i_minus]) / (2.0 * params.dx);

    ez[idx] = ca[idx] * ez[idx] + cb[idx] * curl_h;
}
```

```rust
// fdtd-engine/src/kernels/update_h_field.wgsl

@group(0) @binding(0) var<storage, read_write> hx: array<f32>;
@group(0) @binding(1) var<storage, read_write> hy: array<f32>;
@group(0) @binding(2) var<storage, read> ez: array<f32>;
@group(1) @binding(0) var<uniform> params: SimParams;

@compute @workgroup_size(16, 16)
fn update_h_field(@builtin(global_invocation_id) id: vec3<u32>) {
    let i = id.x;
    let j = id.y;

    if (i >= params.nx - 1u || j >= params.ny - 1u) {
        return;
    }

    let idx = i + j * params.nx;
    let idx_ez_j_plus = i + (j + 1u) * params.nx;
    let idx_ez_i_plus = (i + 1u) + j * params.nx;

    let coeff = params.dt / (4.0e-7 * 3.14159265359);  // μ₀

    // Hx update
    hx[idx] -= coeff * (ez[idx_ez_j_plus] - ez[idx]) / params.dy;

    // Hy update
    hy[idx] += coeff * (ez[idx_ez_i_plus] - ez[idx]) / params.dx;
}
```

### Mesh Generation with Sub-pixel Averaging

Photonic structures often have curved boundaries (ring resonators, Euler bends) that don't align with Cartesian grids. Sub-pixel averaging computes the effective permittivity of partially-filled grid cells:

```rust
// fdtd-engine/src/mesh.rs

pub fn compute_effective_permittivity(
    cell: &GridCell,
    geometry: &Geometry,
    materials: &MaterialDatabase,
) -> f64 {
    const SUBPIXEL_RES: usize = 8;  // 8×8 sub-grid

    let mut eps_sum = 0.0;
    let dx_sub = cell.dx / SUBPIXEL_RES as f64;
    let dy_sub = cell.dy / SUBPIXEL_RES as f64;

    for i in 0..SUBPIXEL_RES {
        for j in 0..SUBPIXEL_RES {
            let x = cell.x_min + (i as f64 + 0.5) * dx_sub;
            let y = cell.y_min + (j as f64 + 0.5) * dy_sub;

            let material = geometry.material_at_point(x, y);
            eps_sum += materials.get_permittivity(material, wavelength);
        }
    }

    eps_sum / (SUBPIXEL_RES * SUBPIXEL_RES) as f64
}
```

This provides smooth refractive index transitions and reduces staircase errors by 10-100× compared to nearest-neighbor material assignment.

---

## Architecture Deep-Dives

### 1. FDTD Simulation Handler (Rust/Axum)

```rust
// src/api/handlers/fdtd.rs

use axum::{
    extract::{Path, State, Json},
    http::StatusCode,
    response::IntoResponse,
};
use uuid::Uuid;
use crate::{
    db::models::{FdtdSimulation, FdtdSimulationRequest},
    state::AppState,
    auth::Claims,
    error::ApiError,
};

pub async fn create_fdtd_simulation(
    State(state): State<AppState>,
    claims: Claims,
    Path(project_id): Path<Uuid>,
    Json(req): Json<FdtdSimulationRequest>,
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

    // 2. Estimate mesh size and cost
    let mesh_info = estimate_mesh_and_cost(&req.domain, req.dimension.as_str())?;

    // 3. Check GPU hour quota
    let user = sqlx::query_as!(
        crate::db::models::User,
        "SELECT * FROM users WHERE id = $1",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    if user.gpu_hours_used + mesh_info.estimated_gpu_hours > user.gpu_hours_limit {
        return Err(ApiError::QuotaExceeded(format!(
            "GPU hour limit reached. Used: {:.2}/{:.2} hours",
            user.gpu_hours_used, user.gpu_hours_limit
        )));
    }

    // 4. Create simulation record
    let sim = sqlx::query_as!(
        FdtdSimulation,
        r#"INSERT INTO fdtd_simulations
            (project_id, name, dimension, domain, sources, monitors, materials,
             mesh_override_regions, status, gpu_type, estimated_cost_usd,
             total_timesteps, created_by)
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8, 'queued', $9, $10, $11, $12)
        RETURNING *"#,
        project_id,
        req.name,
        req.dimension,
        serde_json::to_value(&req.domain)?,
        serde_json::to_value(&req.sources)?,
        serde_json::to_value(&req.monitors)?,
        serde_json::to_value(&req.materials)?,
        serde_json::json!([]),
        mesh_info.gpu_type,
        mesh_info.estimated_cost_usd,
        mesh_info.total_timesteps as i32,
        claims.user_id,
    )
    .fetch_one(&state.db)
    .await?;

    // 5. Enqueue job in Redis
    state.redis
        .rpush("fdtd:jobs", serde_json::to_string(&sim.id)?)
        .await?;

    Ok((StatusCode::CREATED, Json(sim)))
}

struct MeshInfo {
    mesh_cells: usize,
    total_timesteps: usize,
    estimated_gpu_hours: f32,
    estimated_cost_usd: f32,
    gpu_type: String,
}

fn estimate_mesh_and_cost(domain: &FdtdDomain, dimension: &str) -> Result<MeshInfo, ApiError> {
    // Mesh resolution based on accuracy setting (1-5)
    let min_ppw = match domain.mesh_accuracy {
        1 => 10.0,  // Coarse
        2 => 15.0,
        3 => 20.0,  // Default
        4 => 25.0,
        5 => 30.0,  // Fine
        _ => return Err(ApiError::BadRequest("mesh_accuracy must be 1-5")),
    };

    // Smallest wavelength in simulation determines mesh size
    let lambda_min = 1.5;  // 1.5 μm for near-IR photonics
    let dx = lambda_min / min_ppw;

    let (mesh_cells, timesteps) = if dimension == "2d" {
        let nx = (domain.x_span / dx).ceil() as usize + 2 * domain.pml_layers as usize;
        let ny = (domain.y_span / dx).ceil() as usize + 2 * domain.pml_layers as usize;
        let cells = nx * ny;

        // Simulation time = 3× round-trip time for convergence
        let max_dim = domain.x_span.max(domain.y_span);
        let sim_time = 3.0 * max_dim / 0.3;  // c = 0.3 μm/fs
        let dt = dx / (0.3 * 1.414);  // Courant stability
        let steps = (sim_time / dt).ceil() as usize;

        (cells, steps)
    } else {
        return Err(ApiError::BadRequest("3D FDTD not supported in MVP"));
    };

    // GPU time estimation (empirical: ~50M cell-updates/sec on A10G)
    let total_updates = mesh_cells * timesteps;
    let gpu_seconds = total_updates as f64 / 50e6;
    let gpu_hours = (gpu_seconds / 3600.0) as f32;

    // Cost based on GPU type
    let (gpu_type, cost_per_hour) = if mesh_cells < 500_000 {
        ("t4", 0.35)
    } else if mesh_cells < 2_000_000 {
        ("a10g", 1.00)
    } else {
        ("a100", 2.50)
    };

    Ok(MeshInfo {
        mesh_cells,
        total_timesteps: timesteps,
        estimated_gpu_hours: gpu_hours,
        estimated_cost_usd: gpu_hours * cost_per_hour,
        gpu_type: gpu_type.to_string(),
    })
}

pub async fn get_fdtd_simulation(
    State(state): State<AppState>,
    claims: Claims,
    Path((project_id, sim_id)): Path<(Uuid, Uuid)>,
) -> Result<Json<FdtdSimulation>, ApiError> {
    let sim = sqlx::query_as!(
        FdtdSimulation,
        "SELECT s.* FROM fdtd_simulations s
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

pub async fn cancel_fdtd_simulation(
    State(state): State<AppState>,
    claims: Claims,
    Path(sim_id): Path<Uuid>,
) -> Result<StatusCode, ApiError> {
    sqlx::query!(
        "UPDATE fdtd_simulations SET status = 'cancelled', completed_at = NOW()
         WHERE id = $1 AND created_by = $2 AND status IN ('queued', 'running')",
        sim_id,
        claims.user_id
    )
    .execute(&state.db)
    .await?;

    // Publish cancellation message
    state.redis.publish(
        format!("fdtd:{}:cancel", sim_id),
        "cancelled"
    ).await?;

    Ok(StatusCode::OK)
}
```

### 2. 2D FDTD Solver Core (Rust + wgpu)

```rust
// fdtd-engine/src/solver2d.rs

use wgpu::util::DeviceExt;
use std::sync::Arc;

pub struct Fdtd2DSolver {
    device: Arc<wgpu::Device>,
    queue: Arc<wgpu::Queue>,
    nx: u32,
    ny: u32,
    dx: f64,
    dy: f64,
    dt: f64,

    // GPU buffers
    ez_buffer: wgpu::Buffer,
    hx_buffer: wgpu::Buffer,
    hy_buffer: wgpu::Buffer,
    ca_buffer: wgpu::Buffer,  // Material coefficients
    cb_buffer: wgpu::Buffer,

    // Compute pipelines
    update_e_pipeline: wgpu::ComputePipeline,
    update_h_pipeline: wgpu::ComputePipeline,
    pml_e_pipeline: wgpu::ComputePipeline,
    pml_h_pipeline: wgpu::ComputePipeline,

    // Bind groups
    e_bind_group: wgpu::BindGroup,
    h_bind_group: wgpu::BindGroup,
}

impl Fdtd2DSolver {
    pub async fn new(
        device: Arc<wgpu::Device>,
        queue: Arc<wgpu::Queue>,
        nx: u32,
        ny: u32,
        dx: f64,
        dy: f64,
        materials: &MaterialGrid,
    ) -> anyhow::Result<Self> {
        // Courant stability condition
        let dt = 0.9 / (SPEED_OF_LIGHT * (1.0/(dx*dx) + 1.0/(dy*dy)).sqrt());

        // Allocate GPU buffers
        let grid_size = (nx * ny) as usize;
        let ez_buffer = device.create_buffer(&wgpu::BufferDescriptor {
            label: Some("Ez field"),
            size: (grid_size * std::mem::size_of::<f32>()) as u64,
            usage: wgpu::BufferUsages::STORAGE | wgpu::BufferUsages::COPY_SRC,
            mapped_at_creation: false,
        });

        let hx_buffer = device.create_buffer(&wgpu::BufferDescriptor {
            label: Some("Hx field"),
            size: (grid_size * std::mem::size_of::<f32>()) as u64,
            usage: wgpu::BufferUsages::STORAGE | wgpu::BufferUsages::COPY_SRC,
            mapped_at_creation: false,
        });

        let hy_buffer = device.create_buffer(&wgpu::BufferDescriptor {
            label: Some("Hy field"),
            size: (grid_size * std::mem::size_of::<f32>()) as u64,
            usage: wgpu::BufferUsages::STORAGE | wgpu::BufferUsages::COPY_SRC,
            mapped_at_creation: false,
        });

        // Initialize material coefficient buffers
        let (ca, cb) = materials.compute_update_coefficients(dt);
        let ca_buffer = device.create_buffer_init(&wgpu::util::BufferInitDescriptor {
            label: Some("Ca coefficients"),
            contents: bytemuck::cast_slice(&ca),
            usage: wgpu::BufferUsages::STORAGE,
        });

        let cb_buffer = device.create_buffer_init(&wgpu::util::BufferInitDescriptor {
            label: Some("Cb coefficients"),
            contents: bytemuck::cast_slice(&cb),
            usage: wgpu::BufferUsages::STORAGE,
        });

        // Load and compile shaders
        let e_shader = device.create_shader_module(wgpu::ShaderModuleDescriptor {
            label: Some("E-field update shader"),
            source: wgpu::ShaderSource::Wgsl(
                include_str!("kernels/update_e_field.wgsl").into()
            ),
        });

        let h_shader = device.create_shader_module(wgpu::ShaderModuleDescriptor {
            label: Some("H-field update shader"),
            source: wgpu::ShaderSource::Wgsl(
                include_str!("kernels/update_h_field.wgsl").into()
            ),
        });

        // Create bind group layouts and pipelines
        let bind_group_layout = device.create_bind_group_layout(&wgpu::BindGroupLayoutDescriptor {
            label: Some("FDTD bind group layout"),
            entries: &[
                wgpu::BindGroupLayoutEntry {
                    binding: 0,
                    visibility: wgpu::ShaderStages::COMPUTE,
                    ty: wgpu::BindingType::Buffer {
                        ty: wgpu::BufferBindingType::Storage { read_only: false },
                        has_dynamic_offset: false,
                        min_binding_size: None,
                    },
                    count: None,
                },
                // ... additional bindings for hx, hy, ca, cb
            ],
        });

        let update_e_pipeline = device.create_compute_pipeline(&wgpu::ComputePipelineDescriptor {
            label: Some("Update E-field pipeline"),
            layout: None,
            module: &e_shader,
            entry_point: "update_e_field",
        });

        let update_h_pipeline = device.create_compute_pipeline(&wgpu::ComputePipelineDescriptor {
            label: Some("Update H-field pipeline"),
            layout: None,
            module: &h_shader,
            entry_point: "update_h_field",
        });

        // Create bind groups
        let e_bind_group = device.create_bind_group(&wgpu::BindGroupDescriptor {
            label: Some("E-field bind group"),
            layout: &bind_group_layout,
            entries: &[
                wgpu::BindGroupEntry { binding: 0, resource: ez_buffer.as_entire_binding() },
                wgpu::BindGroupEntry { binding: 1, resource: hx_buffer.as_entire_binding() },
                wgpu::BindGroupEntry { binding: 2, resource: hy_buffer.as_entire_binding() },
                wgpu::BindGroupEntry { binding: 3, resource: ca_buffer.as_entire_binding() },
                wgpu::BindGroupEntry { binding: 4, resource: cb_buffer.as_entire_binding() },
            ],
        });

        Ok(Self {
            device,
            queue,
            nx,
            ny,
            dx,
            dy,
            dt,
            ez_buffer,
            hx_buffer,
            hy_buffer,
            ca_buffer,
            cb_buffer,
            update_e_pipeline,
            update_h_pipeline,
            pml_e_pipeline: update_e_pipeline.clone(),  // Simplified for MVP
            pml_h_pipeline: update_h_pipeline.clone(),
            e_bind_group,
            h_bind_group: e_bind_group.clone(),  // Actual implementation has separate groups
        })
    }

    pub async fn run(
        &mut self,
        timesteps: u32,
        sources: &[Source],
        monitors: &[Monitor],
        progress_callback: impl Fn(u32, f32),
    ) -> anyhow::Result<SimulationResults> {
        let mut encoder = self.device.create_command_encoder(&Default::default());
        let mut results = SimulationResults::new(monitors);

        for t in 0..timesteps {
            // 1. Update H-field
            {
                let mut cpass = encoder.begin_compute_pass(&wgpu::ComputePassDescriptor {
                    label: Some("Update H-field"),
                });
                cpass.set_pipeline(&self.update_h_pipeline);
                cpass.set_bind_group(0, &self.h_bind_group, &[]);

                let workgroups_x = (self.nx + 15) / 16;
                let workgroups_y = (self.ny + 15) / 16;
                cpass.dispatch_workgroups(workgroups_x, workgroups_y, 1);
            }

            // 2. Update E-field
            {
                let mut cpass = encoder.begin_compute_pass(&wgpu::ComputePassDescriptor {
                    label: Some("Update E-field"),
                });
                cpass.set_pipeline(&self.update_e_pipeline);
                cpass.set_bind_group(0, &self.e_bind_group, &[]);

                let workgroups_x = (self.nx + 15) / 16;
                let workgroups_y = (self.ny + 15) / 16;
                cpass.dispatch_workgroups(workgroups_x, workgroups_y, 1);
            }

            // 3. Add sources
            for source in sources {
                self.inject_source(&mut encoder, source, t);
            }

            // 4. Record monitor data
            if t % 10 == 0 {  // Record every 10 timesteps
                self.record_monitors(&mut encoder, &monitors, t, &mut results).await?;
            }

            // Submit commands every 100 timesteps to avoid timeout
            if t % 100 == 0 || t == timesteps - 1 {
                self.queue.submit(Some(encoder.finish()));
                encoder = self.device.create_command_encoder(&Default::default());

                progress_callback(t, t as f32 / timesteps as f32 * 100.0);
            }
        }

        Ok(results)
    }

    fn inject_source(&self, encoder: &mut wgpu::CommandEncoder, source: &Source, t: u32) {
        // Simplified source injection (actual uses compute shader or CPU update + upload)
        let time = t as f64 * self.dt;
        let value = source.waveform(time);

        // For mode source, this would inject the mode profile × amplitude
        // For plane wave, inject across a line
        // Implementation details omitted for brevity
    }

    async fn record_monitors(
        &self,
        encoder: &mut wgpu::CommandEncoder,
        monitors: &[Monitor],
        timestep: u32,
        results: &mut SimulationResults,
    ) -> anyhow::Result<()> {
        // Copy field data from GPU to staging buffer
        let staging_buffer = self.device.create_buffer(&wgpu::BufferDescriptor {
            label: Some("Staging buffer"),
            size: self.ez_buffer.size(),
            usage: wgpu::BufferUsages::MAP_READ | wgpu::BufferUsages::COPY_DST,
            mapped_at_creation: false,
        });

        encoder.copy_buffer_to_buffer(
            &self.ez_buffer,
            0,
            &staging_buffer,
            0,
            self.ez_buffer.size(),
        );

        self.queue.submit(Some(encoder.finish()));

        // Map and read data
        let buffer_slice = staging_buffer.slice(..);
        let (tx, rx) = futures::channel::oneshot::channel();
        buffer_slice.map_async(wgpu::MapMode::Read, |result| {
            tx.send(result).unwrap();
        });
        self.device.poll(wgpu::Maintain::Wait);
        rx.await??;

        let data = buffer_slice.get_mapped_range();
        let field_data: &[f32] = bytemuck::cast_slice(&data);

        // Extract monitor data
        for monitor in monitors {
            results.record(monitor, timestep, field_data);
        }

        drop(data);
        staging_buffer.unmap();

        Ok(())
    }
}
```

### 3. Eigenmode Solver (Rust FEM)

```rust
// eigenmode-solver/src/lib.rs

use nalgebra_sparse::{CsrMatrix, CooMatrix};
use arpack::EigenSolver;

pub struct EigenmodeSolver {
    mesh: FemMesh,
    materials: MaterialDatabase,
    wavelength: f64,
}

impl EigenmodeSolver {
    pub fn solve(
        &self,
        num_modes: usize,
        boundary: BoundaryCondition,
    ) -> anyhow::Result<Vec<Mode>> {
        // 1. Build FEM matrices (sparse)
        let (stiffness, mass) = self.assemble_fem_matrices()?;

        // 2. Solve generalized eigenvalue problem: K·φ = β²·M·φ
        //    where β = neff·k₀ (propagation constant)
        let eigenvalues = arpack::solve_generalized_eigenproblem(
            &stiffness,
            &mass,
            num_modes,
            arpack::Which::LargestMagnitude,
        )?;

        // 3. Extract modes
        let modes = eigenvalues.into_iter().map(|(beta_squared, eigenvector)| {
            let k0 = 2.0 * PI / self.wavelength;
            let beta = beta_squared.sqrt();
            let neff = beta / k0;

            // Compute group index via finite difference
            let ng = self.compute_group_index(neff, &eigenvector)?;

            // Compute confinement factor
            let confinement = self.compute_confinement(&eigenvector)?;

            // Extract field components from eigenvector
            let (ex, ey, ez, hx, hy, hz) = self.extract_field_components(&eigenvector);

            Ok(Mode {
                mode_index: 0,  // Filled in later
                neff,
                ng,
                loss: 0.0,  // TODO: compute from imaginary part of neff
                confinement,
                field_profile: ModeProfile { ex, ey, ez, hx, hy, hz },
            })
        }).collect::<Result<Vec<_>, _>>()?;

        Ok(modes)
    }

    fn assemble_fem_matrices(&self) -> anyhow::Result<(CsrMatrix<f64>, CsrMatrix<f64>)> {
        let ndof = self.mesh.num_nodes() * 2;  // Ex and Ey for each node
        let mut stiffness = CooMatrix::new(ndof, ndof);
        let mut mass = CooMatrix::new(ndof, ndof);

        // Loop over triangular elements
        for elem in &self.mesh.elements {
            let nodes = elem.node_indices();
            let material = self.materials.get(&elem.material_name);
            let eps_r = material.permittivity_at_wavelength(self.wavelength);

            // Compute element stiffness and mass matrices
            let k_elem = self.element_stiffness_matrix(elem, eps_r);
            let m_elem = self.element_mass_matrix(elem, eps_r);

            // Assemble into global matrices
            for i in 0..3 {
                for j in 0..3 {
                    let gi = nodes[i] * 2;
                    let gj = nodes[j] * 2;

                    stiffness.push(gi, gj, k_elem[(i*2, j*2)]);
                    stiffness.push(gi+1, gj+1, k_elem[(i*2+1, j*2+1)]);

                    mass.push(gi, gj, m_elem[(i*2, j*2)]);
                    mass.push(gi+1, gj+1, m_elem[(i*2+1, j*2+1)]);
                }
            }
        }

        Ok((CsrMatrix::from(&stiffness), CsrMatrix::from(&mass)))
    }

    fn element_stiffness_matrix(&self, elem: &TriangleElement, eps_r: f64) -> Matrix6<f64> {
        // Implement FEM stiffness matrix for 2D vector wave equation
        // Using first-order triangular elements (linear basis functions)
        // Details omitted for brevity
        todo!()
    }

    fn compute_group_index(&self, neff: f64, eigenvector: &[f64]) -> anyhow::Result<f64> {
        // Finite difference: ng = c/vg = neff - λ·dneff/dλ
        let delta_lambda = self.wavelength * 1e-4;

        // Solve at wavelength + delta
        let mut solver_plus = self.clone();
        solver_plus.wavelength += delta_lambda;
        let neff_plus = solver_plus.solve_single_mode(eigenvector)?;

        let dneff_dlambda = (neff_plus - neff) / delta_lambda;
        let ng = neff - self.wavelength * dneff_dlambda;

        Ok(ng)
    }

    fn compute_confinement(&self, eigenvector: &[f64]) -> anyhow::Result<f64> {
        // Confinement = ∫_core |E|² dA / ∫_total |E|² dA
        let mut core_energy = 0.0;
        let mut total_energy = 0.0;

        for elem in &self.mesh.elements {
            let e_squared = self.compute_element_energy(elem, eigenvector);
            total_energy += e_squared;

            if elem.material_name == "Si" || elem.material_name == "Si3N4" {
                core_energy += e_squared;
            }
        }

        Ok(core_energy / total_energy)
    }
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–5)

**Day 1: Project initialization and Rust backend scaffold**
```bash
cargo new photonpath-api --bin
cd photonpath-api
cargo add axum tokio serde serde_json sqlx uuid chrono tower-http jsonwebtoken bcrypt
cargo add --features postgres,runtime-tokio-rustls,uuid,chrono,json sqlx
cargo add aws-sdk-s3 redis wgpu
```
- `src/main.rs` — Axum app with CORS, tracing, graceful shutdown
- `src/config.rs` — Environment-based configuration (DATABASE_URL, JWT_SECRET, R2_ENDPOINT, REDIS_URL)
- `src/state.rs` — AppState struct (PgPool, Redis, S3Client, wgpu Device/Queue)
- `src/error.rs` — ApiError enum with IntoResponse for photonics-specific errors
- `Dockerfile` — Multi-stage build with GPU support (NVIDIA base image)
- `docker-compose.yml` — PostgreSQL, Redis, MinIO (R2-compatible)

**Day 2: Database schema and migrations**
- `migrations/001_initial.sql` — All 13 tables: users, organizations, org_members, pdks, projects, components, fdtd_simulations, eigenmode_jobs, sparam_models, circuit_simulations, optimization_jobs, layouts, materials, comments, usage_records
- `src/db/mod.rs` — Database pool initialization with connection pooling
- `src/db/models.rs` — All SQLx structs with FromRow derives
- Run `sqlx migrate run` to apply schema
- Seed script for materials database (Si, SiO2, Si3N4, InP, GaAs with n,k data from Palik)

**Day 3: Authentication system**
- `src/auth/mod.rs` — JWT token generation and validation middleware
- `src/auth/oauth.rs` — Google and GitHub OAuth 2.0 flow handlers
- `src/api/handlers/auth.rs` — Register, login, OAuth callback, refresh token, me endpoints
- Password hashing with bcrypt (cost factor 12)
- JWT with 24h access token + 30d refresh token
- Auth middleware extracting Claims from Authorization header
- Academic email verification for free tier (.edu, .ac.uk)

**Day 4: User and project CRUD**
- `src/api/handlers/users.rs` — Get profile, update profile, delete account, get usage stats
- `src/api/handlers/projects.rs` — Create, list, get, update, delete, fork project
- `src/api/handlers/orgs.rs` — Create org, invite member, list members, remove member
- `src/api/router.rs` — All route definitions with auth middleware
- Integration tests: auth flow, project CRUD, authorization checks

**Day 5: Material database and PDK setup**
- `src/api/handlers/materials.rs` — List materials, get material, upload custom material
- `src/api/handlers/pdks.rs` — List PDKs, get PDK details, get component library
- Material n,k data parser from CSV (wavelength, n, k columns)
- Lorentz model fitting for dispersive FDTD (multi-pole fit to n,k data)
- Seed SiEPIC PDK: layer stack (Si core 220nm, SiO2 cladding), design rules (min width 450nm, min bend radius 5µm), waveguide profiles (strip 500nm, rib 500nm/150nm etch)

### Phase 2 — FDTD Engine Core (Days 6–14)

**Day 6: wgpu GPU initialization and buffer management**
- `fdtd-engine/` — New Rust workspace member for FDTD simulation
- `fdtd-engine/src/gpu.rs` — wgpu device/queue initialization with adapter selection (prefer discrete GPU)
- `fdtd-engine/src/buffers.rs` — GPU buffer allocator for E/H fields with double-buffering
- Support both 2D (TM/TE modes) and 3D field layouts (deferred to post-MVP)
- Memory layout: Structure-of-Arrays (SoA) for coalesced GPU access

**Day 7: WGSL shader kernels for E-field and H-field updates**
- `fdtd-engine/src/kernels/update_e_field.wgsl` — E-field update kernel (curl H + material coefficients)
- `fdtd-engine/src/kernels/update_h_field.wgsl` — H-field update kernel (curl E)
- `fdtd-engine/src/kernels/pml_e.wgsl` — PML E-field update with complex coordinate stretching
- `fdtd-engine/src/kernels/pml_h.wgsl` — PML H-field update
- Workgroup size optimization: 16×16 for 2D, 8×8×8 for 3D (tuned for A10G)

**Day 8: Material coefficient computation and sub-pixel averaging**
- `fdtd-engine/src/materials.rs` — Material database lookup by name, wavelength interpolation
- `fdtd-engine/src/mesh.rs` — Cartesian mesh generator with sub-pixel averaging (8×8 sub-grid)
- `fdtd-engine/src/geometry.rs` — Geometric primitives (box, sphere, cylinder, polygon extrusion)
- Dispersive material support: ADE (Auxiliary Differential Equation) method for Drude-Lorentz

**Day 9: Source injection and boundary conditions**
- `fdtd-engine/src/sources.rs` — Source types: plane wave, mode source, Gaussian beam, dipole
- Mode source: launch computed eigenmode into simulation domain
- `fdtd-engine/src/pml.rs` — Uniaxial PML boundary implementation with polynomial grading
- Gaussian pulse temporal profile: exp(-((t-t0)/τ)²) × cos(ω0·t)
- Total-field/scattered-field (TFSF) boundary for plane wave injection

**Day 10: Field monitors and data recording**
- `fdtd-engine/src/monitors.rs` — Monitor types: point, line, plane, volume, DFT (frequency-domain)
- DFT monitor: running DFT accumulation during simulation for S-parameter extraction
- `fdtd-engine/src/recording.rs` — Efficient field data download from GPU (staging buffers)
- Decimation: record every Nth timestep to reduce data volume
- HDF5 export via `hdf5` crate: structured field data with metadata

**Day 11: 2D FDTD solver integration**
- `fdtd-engine/src/solver2d.rs` — Complete 2D FDTD solver with TM/TE mode selection
- Courant stability condition enforcement: Δt = S/(c√(1/Δx² + 1/Δy²)), S = 0.9
- Simulation loop: H-update → source injection → E-update → monitor recording
- Progress callback mechanism for WebSocket streaming
- Convergence detection: field energy plateau or max timesteps reached

**Day 12: Mesh generation from geometry description**
- `fdtd-engine/src/mesh_generator.rs` — Convert geometry JSON to discretized material grid
- Adaptive mesh refinement near material boundaries (optional, MVP uses uniform mesh)
- Mesh accuracy parameter (1-5) → cells-per-wavelength (10-30)
- Mesh memory estimation for cost calculation

**Day 13: FDTD simulation runner and job management**
- `fdtd-engine/src/runner.rs` — High-level simulation orchestration
- Multi-wavelength simulation: loop over wavelengths, extract S-parameters at each
- GPU resource management: single simulation per GPU, queue for multi-job
- Simulation checkpointing: save field state to resume interrupted runs (post-MVP)

**Day 14: S-parameter extraction from monitor data**
- `fdtd-engine/src/sparam_extraction.rs` — Extract S-parameters from DFT monitor data
- Port definition: mode overlap integral with FDTD field
- S_ij = (mode_j · transmitted_field_at_port_i) / (mode_j · source_field_at_port_j)
- Frequency-dependent S-parameters across simulation bandwidth
- Metrics: insertion loss (|S21|), return loss (|S11|), 3-dB bandwidth, center wavelength

### Phase 3 — Eigenmode Solver (Days 15–19)

**Day 15: FEM mesh generation for cross-sections**
- `eigenmode-solver/` — New Rust workspace member
- `eigenmode-solver/src/mesh.rs` — Delaunay triangulation for arbitrary 2D cross-sections
- Triangle library wrapper via `triangle-rs` or custom implementation
- Mesh refinement at material boundaries and waveguide edges
- Typical mesh size: 5K-20K nodes for silicon waveguide cross-section

**Day 16: FEM matrix assembly**
- `eigenmode-solver/src/fem.rs` — Assemble stiffness and mass matrices
- Vector wave equation: ∇×(1/μ ∇×E) - k₀²ε E = 0 → generalized eigenvalue problem K·φ = β²M·φ
- First-order triangular elements (linear basis functions)
- Sparse matrix storage via `nalgebra-sparse` CooMatrix → CsrMatrix
- Boundary condition enforcement: PEC (E_tangential = 0), PML (absorbing)

**Day 17: Eigenvalue solver integration**
- `eigenmode-solver/src/solver.rs` — ARPACK sparse eigenvalue solver via Rust bindings
- Shift-invert mode: target eigenvalues near estimated neff
- Extract top N modes sorted by effective index
- Eigenvector normalization: unit power flow through cross-section

**Day 18: Mode analysis and post-processing**
- `eigenmode-solver/src/mode.rs` — Mode struct with neff, ng, loss, field profile
- Group index computation: finite-difference neff vs. wavelength
- Confinement factor: ∫_core |E|² dA / ∫_total |E|² dA
- Effective area: (∫ |E|² dA)² / ∫ |E|⁴ dA
- Mode profile export: field components on FEM mesh for visualization

**Day 19: Wavelength and geometry sweep**
- `eigenmode-solver/src/sweep.rs` — Batch eigenmode solving over parameter ranges
- Wavelength sweep: solve at multiple wavelengths, extract dispersion (neff vs. λ)
- Geometry sweep: vary waveguide width/height/gap, plot neff contours
- Parallel execution: use Rayon for multi-threaded sweep
- Results aggregation: JSON array of {wavelength, width, neff, ng, loss}

### Phase 4 — Frontend Layout Editor (Days 20–26)

**Day 20: Frontend scaffold and Canvas2D setup**
```bash
npm create vite@latest frontend -- --template react-ts
cd frontend
npm i zustand @tanstack/react-query axios rbush plotly.js
```
- `src/App.tsx` — Router, auth context, layout with sidebar
- `src/stores/projectStore.ts` — Zustand store for project state
- `src/stores/layoutStore.ts` — Layout editor state (components, waveguides, selection)
- `src/components/LayoutEditor/Canvas.tsx` — Canvas2D with pan/zoom via transform matrix
- `src/components/LayoutEditor/Grid.tsx` — Snap-to-grid overlay (1nm, 10nm, 100nm options)

**Day 21: Waveguide drawing tools**
- `src/components/LayoutEditor/WaveguideTools.tsx` — Straight, curved (Euler/circular), taper, S-bend
- `src/components/LayoutEditor/WaveguideRenderer.tsx` — Canvas2D path rendering with width visualization
- Snap-to-port: highlight compatible ports when routing near component
- `src/lib/waveguide_routing.ts` — Auto-router using A* with bend radius and clearance constraints
- R-tree spatial index via `rbush` for fast collision detection

**Day 22: Component palette and placement**
- `src/components/LayoutEditor/ComponentPalette.tsx` — Categorized component library (Passive, Active, I/O)
- Built-in components: straight waveguide, directional coupler, ring resonator, grating coupler, MMI 1×2/2×2, Y-junction, crossing
- `src/components/LayoutEditor/ComponentRenderer.tsx` — Render component footprint from geometry params
- Drag-and-drop from palette → canvas, rotation (R key), alignment guides
- Port visualization: colored circles at component ports with direction indicators

**Day 23: Layer stack visualization**
- `src/components/LayoutEditor/LayerStack.tsx` — Cross-section view showing PDK layer stack
- Click layout → show cross-section at that XY position with material colors
- Material legend with refractive indices
- Support for multiple waveguide layers (multi-layer PICs)

**Day 24: GDS-II export**
- `src/lib/gds_export.ts` — Generate GDS-II binary format from layout data
- Waveguide → polygon path with end caps
- Component instances → GDS cell references
- Layer mapping from logical (waveguide, metal, via) to GDS layer numbers per PDK
- Export via Rust backend: `src/api/handlers/layouts.rs` endpoint generates GDS, stores in R2

**Day 25: Design rule checking (DRC)**
- `src/lib/drc.ts` — Client-side DRC engine using R-tree spatial queries
- Rules: min waveguide width, min bend radius, min waveguide spacing, taper constraints
- Violation rendering: red highlights on canvas with error messages
- `src/components/LayoutEditor/DrcPanel.tsx` — List of DRC violations with click-to-zoom

**Day 26: Layout polish and keyboard shortcuts**
- Multi-select with box drag, copy/paste, group move
- Undo/redo stack (Zustand middleware)
- Keyboard shortcuts: Ctrl+Z/Y undo/redo, Ctrl+C/V copy/paste, Ctrl+S save, R rotate, M mirror, Del delete
- Minimap showing full chip with viewport indicator
- Measurement tool: click two points → show distance and angle

### Phase 5 — Field Visualization (Days 27–31)

**Day 27: WebGL field viewer foundation**
- `src/components/FieldViewer/FieldViewer.tsx` — WebGL2 canvas for field rendering
- `src/components/FieldViewer/FieldShader.wgsl` — Fragment shader for field intensity color mapping
- Field data download from R2: HDF5 parsing in browser via `h5wasm` or pre-converted to JSON
- 2D field slice: upload to GPU texture, render with color map shader

**Day 28: Color maps and dynamic range control**
- `src/components/FieldViewer/ColorMaps.ts` — Viridis, plasma, inferno, magma, RdBu, coolwarm
- Dynamic range slider: adjust min/max field values for color mapping
- Linear vs. log scale toggle (dB scale: 20·log10(|E|/max|E|))
- Color bar legend with field value labels

**Day 29: Cross-section slicing and animation**
- `src/components/FieldViewer/SliceControls.tsx` — XY/XZ/YZ plane selector with position slider
- Field component selector: Ex, Ey, Ez, Hx, Hy, Hz, |E|², |H|², Poynting (Sx, Sy, Sz)
- Time-domain animation: playback FDTD field evolution with play/pause/speed controls
- Frame-by-frame stepping with arrow keys

**Day 30: Eigenmode profile visualization**
- `src/components/FieldViewer/ModeProfile.tsx` — 2D contour plot of eigenmode fields
- Plotly.js contour plot with custom color scale
- Overlay neff, ng, loss, confinement factor as text annotations
- Export mode profile as PNG or SVG for publications

**Day 31: Field data export**
- HDF5 export endpoint: `GET /api/fdtd/:id/export/hdf5` generates presigned R2 URL
- VTK export for ParaView visualization (3D field data)
- CSV export for 1D line monitors (distance, Ex, Ey, Ez columns)
- Screenshot capture: high-resolution PNG render of WebGL canvas

### Phase 6 — API + Job Orchestration (Days 32–36)

**Day 32: FDTD simulation API endpoints**
- `src/api/handlers/fdtd.rs` — Create FDTD simulation, get simulation, cancel simulation, get field data
- Mesh estimation and cost calculation
- GPU hour quota checking
- Job enqueue to Redis with priority (Team plan gets priority)

**Day 33: FDTD simulation worker**
- `src/workers/fdtd_worker.rs` — Redis job consumer, runs FDTD solver on GPU
- GPU allocation: lock GPU during simulation, release on completion
- Progress streaming: WebSocket broadcast every 100 timesteps
- Error handling: GPU OOM, convergence warnings, timeout
- HDF5 field data upload to R2 with metadata

**Day 34: Eigenmode job API and worker**
- `src/api/handlers/eigenmode.rs` — Create eigenmode job, get results, get mode profiles
- `src/workers/eigenmode_worker.rs` — Redis job consumer, runs FEM eigenmode solver
- Wavelength sweep parallelization across CPU cores
- Results JSON upload to database, mode profile data to R2

**Day 35: WebSocket for live simulation progress**
- `src/api/ws/mod.rs` — Axum WebSocket upgrade handler
- `src/api/ws/fdtd_progress.rs` — Subscribe to FDTD simulation progress channel
- Client receives: `{ progress_pct, current_timestep, total_timesteps, field_energy }`
- Live field preview: send downsampled 2D field slice every 500 timesteps for real-time visualization

**Day 36: S-parameter and circuit simulation**
- `src/api/handlers/sparams.rs` — Get S-parameters, export Touchstone file, fit rational polynomial
- `src/api/handlers/circuit.rs` — Create circuit simulation (S-matrix composition), get results
- Circuit solver: frequency-domain matrix multiplication of component S-parameters
- Wavelength sweep: compute circuit transmission/reflection across bandwidth

### Phase 7 — Billing + Plan Enforcement (Days 37–39)

**Day 37: Stripe integration**
- `src/api/handlers/billing.rs` — Create checkout session, customer portal session, get subscription
- `src/api/handlers/webhooks/stripe.rs` — Webhook handler for subscription events
- Plan mapping:
  - Free/Academic: 2 GPU-hours/month, 2D FDTD only, 3 projects
  - Pro ($199/mo): 200 GPU-hours/month, 2D+3D FDTD, eigenmode sweeps, unlimited projects
  - Team ($599/mo): 1000 GPU-hours/month, collaboration, custom PDK, priority queue

**Day 38: Usage tracking and limits**
- `src/middleware/plan_limits.rs` — Middleware checking GPU hour quota before job submission
- `src/services/usage.rs` — Track GPU hours per billing period
- Usage dashboard endpoint: current period usage, historical usage
- Approaching-limit warnings at 80% and 100%

**Day 39: Billing UI and feature gating**
- `frontend/src/pages/Billing.tsx` — Plan comparison, current plan, usage meter
- `frontend/src/components/billing/UsageMeter.tsx` — GPU hours bar with cost estimate
- Gate 3D FDTD, topology optimization, custom PDK upload behind paid plans
- Upgrade prompt modals when hitting limits

### Phase 8 — Validation + Testing + Deployment (Days 40–42)

**Day 40: FDTD solver validation**
- Benchmark 1: Silicon strip waveguide (500nm × 220nm) — compare neff to published values (neff ≈ 2.4 @ 1550nm)
- Benchmark 2: Directional coupler — verify coupling coefficient matches analytical coupled-mode theory
- Benchmark 3: Ring resonator — measure Q-factor and FSR (free spectral range), compare to theory
- Benchmark 4: Grating coupler — verify diffraction efficiency and center wavelength
- Automated test suite with <5% tolerance on S-parameters

**Day 41: Eigenmode solver validation**
- Benchmark 1: Rectangular Si waveguide — compare neff to COMSOL/Lumerical MODE reference
- Benchmark 2: Rib waveguide — validate TE0/TE1 mode neff and ng
- Benchmark 3: Coupling coefficient between two parallel waveguides — compare to coupled-mode theory
- Benchmark 4: Dispersion curve (neff vs. wavelength) — validate GVD calculation
- Tolerance: neff < 0.5%, ng < 2%, confinement < 1%

**Day 42: Deployment and launch**
- Kubernetes deployment to EKS with GPU node pool (NVIDIA A10G)
- Karpenter auto-scaling: scale GPU nodes based on Redis queue depth
- CloudFront CDN for frontend assets and material database API
- Prometheus + Grafana: GPU utilization, simulation throughput, API latency
- Sentry error tracking with source maps
- Landing page, documentation, beta launch announcement

---

## Critical Files

```
photonpath/
├── fdtd-engine/                          # FDTD simulation engine
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── gpu.rs                        # wgpu device/queue initialization
│   │   ├── buffers.rs                    # GPU buffer management
│   │   ├── solver2d.rs                   # 2D FDTD solver (TM/TE modes)
│   │   ├── solver3d.rs                   # 3D FDTD solver (post-MVP)
│   │   ├── materials.rs                  # Material database lookup, n/k interpolation
│   │   ├── mesh.rs                       # Mesh generation with sub-pixel averaging
│   │   ├── mesh_generator.rs             # Convert geometry to discretized grid
│   │   ├── geometry.rs                   # Geometric primitives (box, cylinder, polygon)
│   │   ├── sources.rs                    # Source injection (plane wave, mode, Gaussian)
│   │   ├── pml.rs                        # PML boundary conditions
│   │   ├── monitors.rs                   # Field monitors (point, line, plane, DFT)
│   │   ├── recording.rs                  # Field data download from GPU
│   │   ├── sparam_extraction.rs          # S-parameter extraction from DFT monitors
│   │   ├── runner.rs                     # High-level simulation orchestration
│   │   └── kernels/
│   │       ├── update_e_field.wgsl       # E-field update kernel
│   │       ├── update_h_field.wgsl       # H-field update kernel
│   │       ├── pml_e.wgsl                # PML E-field update
│   │       └── pml_h.wgsl                # PML H-field update
│   └── tests/
│       ├── validation.rs                 # FDTD solver validation benchmarks
│       └── convergence.rs                # Mesh convergence tests
│
├── eigenmode-solver/                     # Eigenmode solver
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── mesh.rs                       # Delaunay triangulation for cross-sections
│   │   ├── fem.rs                        # FEM matrix assembly
│   │   ├── solver.rs                     # ARPACK eigenvalue solver integration
│   │   ├── mode.rs                       # Mode analysis (neff, ng, confinement)
│   │   └── sweep.rs                      # Wavelength and geometry sweeps
│   └── tests/
│       └── validation.rs                 # Eigenmode solver validation
│
├── photonpath-api/                       # Rust API server
│   ├── Cargo.toml
│   ├── src/
│   │   ├── main.rs                       # Axum app setup, router, startup
│   │   ├── config.rs                     # Environment config
│   │   ├── state.rs                      # AppState (PgPool, Redis, S3, wgpu)
│   │   ├── error.rs                      # ApiError types
│   │   ├── auth/
│   │   │   ├── mod.rs                    # JWT middleware, Claims
│   │   │   └── oauth.rs                  # Google/GitHub OAuth handlers
│   │   ├── api/
│   │   │   ├── router.rs                 # Route definitions
│   │   │   ├── handlers/
│   │   │   │   ├── auth.rs               # Register, login, OAuth
│   │   │   │   ├── users.rs              # Profile CRUD, usage stats
│   │   │   │   ├── projects.rs           # Project CRUD, fork, share
│   │   │   │   ├── components.rs         # Component CRUD
│   │   │   │   ├── fdtd.rs               # FDTD simulation endpoints
│   │   │   │   ├── eigenmode.rs          # Eigenmode job endpoints
│   │   │   │   ├── sparams.rs            # S-parameter get/export/fit
│   │   │   │   ├── circuit.rs            # Circuit simulation
│   │   │   │   ├── layouts.rs            # Layout CRUD, GDS export, DRC
│   │   │   │   ├── pdks.rs               # PDK list/get
│   │   │   │   ├── materials.rs          # Material database API
│   │   │   │   ├── billing.rs            # Stripe checkout/portal
│   │   │   │   ├── usage.rs              # Usage tracking
│   │   │   │   └── webhooks/
│   │   │   │       └── stripe.rs         # Stripe webhook handler
│   │   │   └── ws/
│   │   │       ├── mod.rs                # WebSocket upgrade handler
│   │   │       └── fdtd_progress.rs      # Live FDTD progress streaming
│   │   ├── middleware/
│   │   │   └── plan_limits.rs            # Plan enforcement middleware
│   │   ├── services/
│   │   │   ├── usage.rs                  # Usage tracking service
│   │   │   └── s3.rs                     # R2 client helpers
│   │   └── workers/
│   │       ├── mod.rs                    # Worker pool management
│   │       ├── fdtd_worker.rs            # FDTD simulation execution
│   │       └── eigenmode_worker.rs       # Eigenmode job execution
│   ├── migrations/
│   │   └── 001_initial.sql               # Full database schema
│   └── tests/
│       ├── api_integration.rs            # API integration tests
│       └── simulation_e2e.rs             # End-to-end simulation tests
│
├── frontend/                             # React frontend
│   ├── package.json
│   ├── vite.config.ts
│   ├── src/
│   │   ├── App.tsx                       # Router, providers, layout
│   │   ├── main.tsx                      # Entry point
│   │   ├── stores/
│   │   │   ├── authStore.ts              # Auth state (JWT, user)
│   │   │   ├── projectStore.ts           # Project state
│   │   │   ├── layoutStore.ts            # Layout editor state
│   │   │   └── fieldStore.ts             # Field viewer state
│   │   ├── hooks/
│   │   │   ├── useFdtdProgress.ts        # WebSocket hook for FDTD progress
│   │   │   └── useFieldData.ts           # Field data loading hook
│   │   ├── lib/
│   │   │   ├── api.ts                    # Axios API client
│   │   │   ├── spatial.ts                # R-tree for waveguide routing
│   │   │   ├── gds_export.ts             # GDS-II export
│   │   │   ├── waveguide_routing.ts      # Auto-router with A*
│   │   │   └── drc.ts                    # Design rule checking
│   │   ├── pages/
│   │   │   ├── Dashboard.tsx             # Project list
│   │   │   ├── LayoutEditor.tsx          # Main layout editor
│   │   │   ├── FieldViewer.tsx           # Standalone field viewer
│   │   │   ├── PdkBrowser.tsx            # PDK and component library
│   │   │   ├── Billing.tsx               # Plan management
│   │   │   ├── Login.tsx                 # Auth pages
│   │   │   └── Register.tsx
│   │   ├── components/
│   │   │   ├── LayoutEditor/
│   │   │   │   ├── Canvas.tsx            # Canvas2D with pan/zoom
│   │   │   │   ├── Grid.tsx              # Snap grid overlay
│   │   │   │   ├── ComponentPalette.tsx  # Component library sidebar
│   │   │   │   ├── ComponentRenderer.tsx # Component footprint rendering
│   │   │   │   ├── WaveguideTools.tsx    # Waveguide drawing tools
│   │   │   │   ├── WaveguideRenderer.tsx # Waveguide path rendering
│   │   │   │   ├── LayerStack.tsx        # Cross-section view
│   │   │   │   ├── DrcPanel.tsx          # DRC violation list
│   │   │   │   └── PropertyPanel.tsx     # Component property editor
│   │   │   ├── FieldViewer/
│   │   │   │   ├── FieldViewer.tsx       # WebGL field viewer
│   │   │   │   ├── ColorMaps.ts          # Color map definitions
│   │   │   │   ├── SliceControls.tsx     # Cross-section slice controls
│   │   │   │   ├── ModeProfile.tsx       # Eigenmode profile plot
│   │   │   │   └── FieldExport.tsx       # HDF5/VTK/CSV export
│   │   │   ├── SimulationSetup/
│   │   │   │   ├── FdtdSetup.tsx         # FDTD simulation config form
│   │   │   │   ├── EigenmodeSetup.tsx    # Eigenmode job config
│   │   │   │   └── CircuitSetup.tsx      # Circuit simulation config
│   │   │   └── billing/
│   │   │       ├── PlanCard.tsx
│   │   │       └── UsageMeter.tsx
│   │   └── data/
│   │       └── materials.json            # Material database cache
│   └── public/
│
├── k8s/                                  # Kubernetes manifests
│   ├── api-deployment.yaml               # API server
│   ├── fdtd-worker-deployment.yaml       # FDTD workers (GPU nodes)
│   ├── eigenmode-worker-deployment.yaml  # Eigenmode workers (CPU)
│   ├── postgres-statefulset.yaml         # PostgreSQL
│   ├── redis-deployment.yaml             # Redis
│   └── ingress.yaml                      # NGINX ingress with TLS
│
├── docker-compose.yml                    # Local development stack
├── Cargo.toml                            # Workspace root
└── .github/
    └── workflows/
        ├── ci.yml                        # Test + lint on PR
        ├── fdtd-validation.yml           # FDTD solver validation
        └── deploy.yml                    # Build Docker images + deploy to K8s
```

---

## Solver Validation Suite

### FDTD Validation Benchmarks

**Benchmark 1: Silicon Strip Waveguide Effective Index**

**Geometry:** Si core 500nm × 220nm on SiO2 cladding, wavelength 1550nm

**Expected:** neff ≈ **2.40** (TE mode, from Lumerical MODE reference)

**Method:** 2D TM FDTD eigenmode source, measure phase advance over 10µm propagation

**Tolerance:** < 0.5% (neff = 2.40 ± 0.012)

---

**Benchmark 2: Directional Coupler Coupling Coefficient**

**Geometry:** Two parallel Si strip waveguides (500nm × 220nm), gap = 200nm, coupling length = 10µm

**Expected:** Coupling coefficient κ ≈ **0.15 µm⁻¹**, cross-coupling at 10µm ≈ **-3 dB** (from coupled-mode theory)

**Method:** 2D FDTD, mode source in port 1, measure |S21|² at output

**Tolerance:** < 5% on coupling coefficient (κ = 0.15 ± 0.0075 µm⁻¹)

---

**Benchmark 3: Ring Resonator Q-factor and FSR**

**Geometry:** Si ring (radius = 5µm, width = 500nm), coupling gap = 200nm

**Expected:** Q-factor ≈ **10,000** (from propagation loss ~3 dB/cm), FSR ≈ **20 nm** at 1550nm

**Method:** 2D FDTD broadband simulation, extract transmission spectrum, fit Lorentzian

**Tolerance:** Q < 10%, FSR < 2%

---

**Benchmark 4: Grating Coupler Efficiency**

**Geometry:** Uniform Si grating coupler (period = 630nm, etch depth = 70nm, 20 periods)

**Expected:** Peak efficiency ≈ **-3 to -5 dB** at λ = 1550nm (typical for uniform grating, from literature)

**Method:** 2D FDTD with plane wave source at 8° from vertical, measure transmission to waveguide mode

**Tolerance:** Efficiency < 20%, center wavelength < 5 nm

---

### Eigenmode Validation Benchmarks

**Benchmark 1: Rectangular Silicon Waveguide (TE0 mode)**

**Geometry:** Si core 450nm × 220nm on SiO2, wavelength = 1550nm

**Expected:** neff ≈ **2.44**, ng ≈ **4.2**, confinement ≈ **90%** (COMSOL reference)

**Method:** FEM eigenmode solver with 10K DOF mesh

**Tolerance:** neff < 0.5%, ng < 2%, confinement < 1%

---

**Benchmark 2: Rib Waveguide (TE0 and TE1 modes)**

**Geometry:** Si rib (width = 500nm, etch depth = 150nm, slab thickness = 70nm), wavelength = 1550nm

**Expected:** TE0 neff ≈ **2.60**, TE1 neff ≈ **2.30** (Lumerical MODE reference)

**Method:** FEM eigenmode solver, extract first two TE modes

**Tolerance:** neff < 0.5% for each mode

---

**Benchmark 3: Coupling Coefficient (Parallel Waveguides)**

**Geometry:** Two Si strip waveguides (500nm × 220nm), gap = 300nm, wavelength = 1550nm

**Expected:** Coupling length Lc ≈ **40 µm** (from κ = π/(2Lc) and coupled-mode theory)

**Method:** Compute super-mode and sub-mode neff, κ = π·|neff_super - neff_sub|/λ

**Tolerance:** Coupling length < 5%

---

**Benchmark 4: Dispersion Curve (Group Index vs. Wavelength)**

**Geometry:** Si strip waveguide 500nm × 220nm, wavelength range 1500-1600 nm

**Expected:** ng increases from ~4.0 at 1500nm to ~4.4 at 1600nm (monotonic, typical Si dispersion)

**Method:** Solve eigenmode at 20 wavelength points, compute ng = neff - λ·dneff/dλ

**Tolerance:** ng trend matches reference within 3%, GVD sign correct

---

## Post-MVP Roadmap

### v1.1 — 3D FDTD Simulation (Month 2-3)

**Scope:** Full 3D FDTD for out-of-plane structures (grating couplers, vertical tapers, 3D mode converters)

**Tech:** Extend wgpu kernels to 3D grid (8×8×8 workgroups), 3D PML boundaries

**Challenges:** Memory scaling (3D mesh 100-1000× larger than 2D), multi-GPU support for very large simulations

**Value:** Enables accurate grating coupler design, fiber coupling efficiency optimization, 3D photonic crystal cavities

---

### v1.2 — Multi-Port S-Parameters and Circuit Simulation (Month 3-4)

**Scope:** Extract N×N S-parameter matrices (4-port MMI, 8-port AWG), full photonic circuit simulation

**Tech:** Multi-port DFT monitors, S-matrix composition algorithm, Monte Carlo parameter variation

**Value:** System-level PIC verification, yield prediction, complex circuit design (WDM mux/demux, optical switches)

---

### v1.3 — Adjoint and Topology Optimization (Month 4-6)

**Scope:** Gradient-based optimization using adjoint FDTD, topology optimization for inverse-designed components

**Tech:** Adjoint FDTD kernel (run FDTD backward with adjoint sources), gradient computation, density-based topology parameterization

**Challenges:** Multi-objective optimization (insertion loss + bandwidth + footprint), fabrication constraint enforcement (min feature size)

**Value:** Ultra-compact components (10× smaller footprint), broadband couplers, custom wavelength filters

---

### v1.4 — Collaboration and Version Control (Month 6-7)

**Scope:** Real-time collaboration, project version history, code review-style component feedback

**Tech:** Operational transforms for concurrent layout editing, Git-like branching for layout versions

**Value:** Team workflows, design reviews, tape-out version management

---

### v1.5 — Additional PDKs and Platforms (Month 7-9)

**Scope:** Support for AMF (SiN), IMEC iSiPP (SOI), AIM Photonics (SOI), LioniX TriPleX (Si3N4), InP foundries

**Tech:** PDK upload wizard, automated DRC rule extraction, component library import from GDS

**Value:** Multi-foundry design flexibility, heterogeneous integration (Si + InP on same chip)

---

### v1.6 — Python SDK and API Access (Month 9-10)

**Scope:** Python client library for scripted simulation, batch job submission, parametric sweeps

**Tech:** REST API expansion, Python package with FDTD/eigenmode/circuit simulation classes

**Value:** Automated design space exploration, integration with external optimization frameworks (scipy, PyTorch)

---

### v2.0 — Time-Domain Circuit Simulation and Mixed-Signal (Month 10-12)

**Scope:** Transient photonic circuit simulation, co-simulation with electrical drivers/receivers (SPICE integration)

**Tech:** Inverse FFT from S-parameters, convolution-based time-domain propagation, SPICE netlist import for electrical side

**Value:** Full link budget analysis for optical transceivers, modulation and detection simulation, electro-optic co-optimization

---

## Verification

### Integration Testing Checklist

1. **Auth flow** — Register with .edu email → academic plan granted → login → JWT issued → authorized API calls
2. **Project CRUD** — Create project → select SiEPIC PDK → save → reload → verify PDK settings preserved
3. **Layout editing** — Place directional coupler → route waveguides → snap-to-port → DRC check passes
4. **FDTD simulation** — Directional coupler layout → generate FDTD setup → submit → WebSocket progress → field data in R2
5. **Eigenmode solving** — Draw waveguide cross-section → submit eigenmode job → results returned → mode profiles rendered
6. **S-parameter extraction** — FDTD completes → S-parameters extracted → Touchstone file exported
7. **Field visualization** — Load FDTD results → WebGL viewer → color map selection → cross-section slicing
8. **GDS export** — Layout editor → Export GDS-II → file downloaded → import into KLayout → verify geometry
9. **Plan limits** — Free user → attempt 3D FDTD → blocked with Pro upgrade prompt
10. **Billing** — Subscribe to Pro → Stripe checkout → webhook → GPU hours limit increased → can run 3D
11. **Concurrent simulations** — 5 users simultaneously submit FDTD jobs → all complete without GPU conflicts
12. **Error handling** — Invalid mesh parameters → meaningful error message with suggestions → no crash

### SQL Verification Queries

```sql
-- 1. FDTD simulation throughput and success rate
SELECT
  status,
  COUNT(*) AS count,
  AVG(EXTRACT(EPOCH FROM (completed_at - started_at))) AS avg_duration_sec,
  PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY EXTRACT(EPOCH FROM (completed_at - started_at))) AS median_duration_sec
FROM fdtd_simulations
WHERE created_at >= NOW() - INTERVAL '7 days'
GROUP BY status;

-- Expected: >95% 'completed', <5% 'failed', median duration <300 sec for 2D

-- 2. GPU hour consumption by plan tier
SELECT
  u.plan,
  SUM(ur.quantity) AS total_gpu_hours,
  COUNT(DISTINCT ur.user_id) AS user_count,
  SUM(ur.quantity) / COUNT(DISTINCT ur.user_id) AS avg_gpu_hours_per_user
FROM usage_records ur
JOIN users u ON ur.user_id = u.id
WHERE ur.record_type = 'gpu_hours'
  AND ur.period_start >= DATE_TRUNC('month', NOW())
GROUP BY u.plan;

-- Expected: Team plan users consume 10-50× more GPU hours than Free users

-- 3. Most popular components (by FDTD simulation count)
SELECT
  c.component_type,
  COUNT(f.id) AS simulation_count,
  AVG(EXTRACT(EPOCH FROM (f.completed_at - f.started_at))) AS avg_sim_time_sec
FROM fdtd_simulations f
JOIN components c ON f.component_id = c.id
WHERE f.status = 'completed'
  AND f.created_at >= NOW() - INTERVAL '30 days'
GROUP BY c.component_type
ORDER BY simulation_count DESC
LIMIT 10;

-- Expected: directional_coupler, ring_resonator, grating_coupler in top 5

-- 4. Eigenmode solver accuracy (mode count distribution)
SELECT
  num_modes,
  COUNT(*) AS job_count,
  AVG(EXTRACT(EPOCH FROM (completed_at - created_at))) AS avg_solve_time_sec
FROM eigenmode_jobs
WHERE status = 'completed'
  AND created_at >= NOW() - INTERVAL '30 days'
GROUP BY num_modes
ORDER BY num_modes;

-- Expected: Most jobs request 3-5 modes, solve time <10 sec for typical cross-sections

-- 5. Conversion funnel (registration → first simulation → paid conversion)
WITH funnel AS (
  SELECT
    u.id,
    u.created_at AS signup_date,
    MIN(f.created_at) AS first_sim_date,
    MIN(CASE WHEN u.plan IN ('pro', 'team') THEN u.updated_at END) AS paid_conversion_date
  FROM users u
  LEFT JOIN fdtd_simulations f ON u.id = f.created_by
  WHERE u.created_at >= NOW() - INTERVAL '90 days'
  GROUP BY u.id, u.created_at
)
SELECT
  COUNT(*) AS total_signups,
  COUNT(first_sim_date) AS ran_simulation,
  COUNT(paid_conversion_date) AS converted_to_paid,
  ROUND(100.0 * COUNT(first_sim_date) / COUNT(*), 2) AS pct_ran_sim,
  ROUND(100.0 * COUNT(paid_conversion_date) / COUNT(*), 2) AS pct_converted
FROM funnel;

-- Expected: >60% run first simulation within 7 days, >5% convert to paid within 30 days
```

---

**Total Duration:** 42 days (6 weeks) from start to launch

**Team Size:** 1 full-stack engineer (you)

**Estimated Lines of Code:** ~1400 lines (plan file), implementation ~15K lines Rust + 8K lines TypeScript

**Launch Readiness:** Beta launch with 2D FDTD, eigenmode solver, layout editor, S-parameter extraction, Free/Pro/Team billing

