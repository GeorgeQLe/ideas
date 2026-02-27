# 39. ReactorSim — Cloud Nuclear Reactor Simulation Platform

## Implementation Plan

**MVP Scope:** Browser-based 3D reactor core designer with drag-and-drop fuel assembly placement and interactive cross-section visualization rendered via Three.js/WebGL, GPU-accelerated Monte Carlo neutron transport engine supporting continuous-energy and multi-group modes compiled to WebAssembly for small problems (<1000 particles) and server-side CUDA execution for full-core simulations (100M+ particles), deterministic Sn discrete ordinates solver (S4-S16) for 2D/3D Cartesian geometries with diffusion synthetic acceleration, single-phase thermal-hydraulic subchannel analysis with coupled neutronics iteration via Doppler and moderator density feedback, burnup/depletion solver using CRAM matrix exponential method with 500+ actinides and fission products, k-effective eigenvalue calculations with Shannon entropy source convergence diagnostics, 5 pre-built ICSBEP benchmark cases for V&V validation, ENDF/B-VIII.0 nuclear data library with on-the-fly Doppler broadening, interactive power/flux/burnup field overlays on 3D geometry, project workspaces with audit trail, Stripe billing with three tiers (Academic free / Pro $299/mo / Team $799/mo + seats).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| Monte Carlo Solver | Rust + CUDA (cuRAND) | Custom event-based tracking, GPU kernels for particle transport |
| WASM MC Solver | `wasm-pack` + `wasm-bindgen` | Client-side solver for small problems (<1000 particles) |
| Sn Transport Solver | Rust + GPU sweep kernels | Discrete ordinates with DSA acceleration, cuSPARSE integration |
| Thermal-Hydraulics | Rust subchannel solver | Single-phase conservation equations, IAPWS-IF97 steam tables |
| Depletion Solver | Rust CRAM implementation | Chebyshev rational approximation for matrix exponential |
| Nuclear Data Processing | Python 3.12 (FastAPI) | ENDF/B-VIII.0 parsing, Doppler broadening, multi-group collapse |
| Database | PostgreSQL 16 | Projects, users, simulations, reactor models, audit logs |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + SAML/OIDC | Enterprise SSO, bcrypt password hashing, role-based access |
| Object Storage | AWS S3 | Nuclear data libraries, mesh files, simulation results, reports |
| Frontend Framework | React 19 + TypeScript | Vite build system, Zustand state management |
| 3D Visualization | Three.js + custom shaders | Instanced mesh rendering for fuel pins, GPU scalar field overlays |
| Core Designer UI | React Flow + D3.js | Loading pattern editor, 2D assembly maps, power distribution plots |
| Real-time | WebSocket (Axum) | Live simulation progress, convergence monitoring, fission source updates |
| Job Queue | Redis 7 + Tokio tasks | GPU simulation job scheduling, priority queuing |
| GPU Compute | CUDA 12.x (NVIDIA A100/H100) | Kubernetes GPU node pools, NCCL for multi-GPU |
| Document Generation | Typst (Rust) | NRC-formatted regulatory reports with programmatic templates |
| Monitoring | Prometheus + Grafana, Sentry | GPU utilization metrics, simulation throughput, error tracking |
| CI/CD | GitHub Actions | WASM build, CUDA kernel compilation, Docker image push |

### Key Architecture Decisions

1. **Hybrid WASM/GPU Monte Carlo with 1000-particle threshold**: Simulations with <1000 particles (educational examples, criticality benchmarks, simple pin cells) run entirely in the browser via WASM, providing instant feedback with zero server cost. Full-core reactor simulations (100M-1B particles) are submitted to GPU compute workers with CUDA kernels optimized for massively parallel event-based tracking. The threshold is configurable per plan tier.

2. **CUDA-optimized Monte Carlo over OpenCL or vendor-agnostic approaches**: NVIDIA GPUs dominate HPC nuclear simulation (A100/H100 provide 10-50x speedup over MCNP on CPU), and CUDA provides mature cuRAND for random number generation, atomic operations for tally accumulation, and extensive profiling tools. Event-based particle tracking kernels exploit GPU warp execution with minimal divergence through particle sorting by material/energy.

3. **Three.js for 3D reactor visualization with instanced mesh rendering**: Full-core PWR models contain 50,000+ fuel pins — rendering each as separate geometry is infeasible. Instanced mesh rendering draws all pins of the same type in a single draw call, with per-instance attributes (position, burnup, power) uploaded to GPU buffers. Scalar field overlays (flux, temperature) use custom fragment shaders with color map lookup textures.

4. **Sn solver for rapid scoping calculations and MC source initialization**: Monte Carlo requires thousands of inactive cycles to converge the fission source distribution, consuming 50%+ of total runtime. Running a fast Sn calculation first provides a high-quality initial source guess, reducing MC inactive cycles by 5-10x. Sn also enables rapid design iteration during early-stage core layout before committing to expensive MC runs.

5. **PostgreSQL for reactor model versioning with JSONB geometry storage**: Reactor core layouts (assembly positions, enrichment maps, burnable absorber patterns) are stored as JSONB to enable flexible schema evolution while maintaining SQL queryability. Each model edit creates a new version with parent linkage, enabling full design history tracking and branching for parametric studies. Large mesh files (tetrahedral geometry) are stored in S3 with PostgreSQL metadata.

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";  -- For fuzzy text search

-- Users
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email TEXT UNIQUE NOT NULL,
    password_hash TEXT,  -- NULL for SSO-only users
    name TEXT NOT NULL,
    avatar_url TEXT,
    auth_provider TEXT DEFAULT 'email',  -- email | saml | oidc
    auth_provider_id TEXT,
    credentials TEXT,  -- Nuclear engineering certifications (optional)
    plan TEXT NOT NULL DEFAULT 'academic',  -- academic | pro | team | enterprise
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_login_at TIMESTAMPTZ
);
CREATE INDEX users_email_idx ON users(email);
CREATE INDEX users_stripe_idx ON users(stripe_customer_id);

-- Organizations (for Team/Enterprise plan)
CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    slug TEXT UNIQUE NOT NULL,
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    plan TEXT NOT NULL DEFAULT 'team',
    nqa1_mode BOOLEAN DEFAULT false,  -- Strict QA audit trail per 10 CFR 50 Appendix B
    nuclear_data_libraries TEXT[] DEFAULT '{"endfb8"}',  -- Licensed libraries available
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
    role TEXT NOT NULL DEFAULT 'modeler',  -- admin | modeler | reviewer | viewer
    joined_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(org_id, user_id)
);
CREATE INDEX org_members_org_idx ON org_members(org_id);
CREATE INDEX org_members_user_idx ON org_members(user_id);

-- Projects
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    org_id UUID REFERENCES organizations(id) ON DELETE CASCADE,
    created_by UUID NOT NULL REFERENCES users(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    reactor_type TEXT NOT NULL,  -- PWR | BWR | VVER | SMR | MSR | HTGR | SFR
    visibility TEXT NOT NULL DEFAULT 'private',  -- private | org | public
    qa_level TEXT NOT NULL DEFAULT 'preliminary',  -- preliminary | verified | nrc_submittal
    thumbnail_url TEXT,
    forked_from UUID REFERENCES projects(id) ON DELETE SET NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_org_idx ON projects(org_id);
CREATE INDEX projects_created_by_idx ON projects(created_by);
CREATE INDEX projects_updated_idx ON projects(updated_at DESC);

-- Reactor Models (versioned geometry)
CREATE TABLE reactor_models (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    parent_version_id UUID REFERENCES reactor_models(id) ON DELETE SET NULL,
    geometry_type TEXT NOT NULL,  -- csg | mesh
    geometry_data JSONB,  -- CSG geometry or S3 reference for mesh
    core_layout JSONB NOT NULL,  -- Assembly positions, symmetry, dimensions
    materials JSONB NOT NULL DEFAULT '[]',  -- Nuclide compositions, densities, temperatures
    boundary_conditions JSONB DEFAULT '{}',  -- vacuum | reflective | white per surface
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(project_id, version)
);
CREATE INDEX models_project_idx ON reactor_models(project_id);
CREATE INDEX models_version_idx ON reactor_models(project_id, version DESC);

-- Fuel Assembly Templates
CREATE TABLE fuel_assembly_templates (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    reactor_type TEXT NOT NULL,
    lattice_type TEXT NOT NULL,  -- square | hex
    pin_layout JSONB NOT NULL,  -- Pin types, positions, enrichments
    burnable_absorber_config JSONB,  -- Gd | IFBA | WABA patterns
    dimensions JSONB NOT NULL,  -- pitch, height, num_pins
    is_builtin BOOLEAN DEFAULT false,
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX assembly_org_idx ON fuel_assembly_templates(org_id);
CREATE INDEX assembly_type_idx ON fuel_assembly_templates(reactor_type);

-- Simulations
CREATE TABLE simulations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    model_id UUID NOT NULL REFERENCES reactor_models(id) ON DELETE CASCADE,
    submitted_by UUID NOT NULL REFERENCES users(id),
    sim_type TEXT NOT NULL,  -- eigenvalue | fixed_source | transient | depletion | kinetics
    solver TEXT NOT NULL,  -- monte_carlo | sn | diffusion | coupled
    parameters JSONB NOT NULL,  -- Particles, batches, Sn order, mesh, tolerances
    nuclear_data_library TEXT NOT NULL DEFAULT 'endfb8',  -- endfb8 | jeff33 | jendl5
    status TEXT NOT NULL DEFAULT 'queued',  -- queued | running | converging | completed | failed | cancelled
    execution_mode TEXT NOT NULL DEFAULT 'gpu',  -- wasm | gpu
    gpu_hours_used REAL DEFAULT 0.0,
    cost_usd REAL DEFAULT 0.0,
    error_message TEXT,
    submitted_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    parent_simulation_id UUID REFERENCES simulations(id) ON DELETE SET NULL
);
CREATE INDEX sims_project_idx ON simulations(project_id);
CREATE INDEX sims_model_idx ON simulations(model_id);
CREATE INDEX sims_user_idx ON simulations(submitted_by);
CREATE INDEX sims_status_idx ON simulations(status);
CREATE INDEX sims_submitted_idx ON simulations(submitted_at DESC);

-- Simulation Jobs (for GPU execution only)
CREATE TABLE simulation_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    simulation_id UUID NOT NULL REFERENCES simulations(id) ON DELETE CASCADE,
    worker_id TEXT,
    gpu_type TEXT,  -- A100 | H100
    cores_allocated INTEGER DEFAULT 1,
    memory_gb INTEGER DEFAULT 32,
    priority INTEGER DEFAULT 0,  -- Higher = more urgent
    progress_pct REAL DEFAULT 0.0,
    convergence_data JSONB DEFAULT '[]',  -- [{cycle, keff, entropy, residual}]
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX jobs_sim_idx ON simulation_jobs(simulation_id);
CREATE INDEX jobs_worker_idx ON simulation_jobs(worker_id);
CREATE INDEX jobs_priority_idx ON simulation_jobs(priority DESC, created_at ASC);

-- Simulation Results
CREATE TABLE simulation_results (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    simulation_id UUID NOT NULL REFERENCES simulations(id) ON DELETE CASCADE,
    result_type TEXT NOT NULL,  -- keff | flux_mesh | power_dist | tally | depletion | kinetics
    summary_data JSONB,  -- k-eff±σ, peaking factors, cycle length, etc.
    full_result_path TEXT,  -- S3 URL to HDF5 mesh tallies, isotopic inventories
    convergence_data JSONB,  -- Entropy history, tally relative errors
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX results_sim_idx ON simulation_results(simulation_id);
CREATE INDEX results_type_idx ON simulation_results(result_type);

-- Depletion States (for burnup calculations)
CREATE TABLE depletion_states (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    simulation_id UUID NOT NULL REFERENCES simulations(id) ON DELETE CASCADE,
    burnup_step INTEGER NOT NULL,
    burnup_mwd_mtu REAL NOT NULL,
    time_days REAL NOT NULL,
    isotopic_inventory_path TEXT,  -- S3 URL to full nuclide vector per region
    power_distribution_path TEXT,  -- S3 URL to pin/assembly power at this step
    eigenvalue REAL,
    eigenvalue_uncertainty REAL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(simulation_id, burnup_step)
);
CREATE INDEX depletion_sim_idx ON depletion_states(simulation_id);
CREATE INDEX depletion_step_idx ON depletion_states(simulation_id, burnup_step);

-- Benchmark Cases (for V&V)
CREATE TABLE benchmark_cases (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    benchmark_suite TEXT NOT NULL,  -- ICSBEP | IRPhEP | custom
    case_id TEXT NOT NULL UNIQUE,  -- e.g., "LEU-COMP-THERM-001"
    name TEXT NOT NULL,
    description TEXT,
    category TEXT,  -- thermal | fast | intermediate, HEU | LEU
    reference_keff REAL NOT NULL,
    reference_uncertainty REAL NOT NULL,
    model_path TEXT NOT NULL,  -- S3 URL to pre-built ReactorSim model
    source_document TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX benchmark_suite_idx ON benchmark_cases(benchmark_suite);
CREATE INDEX benchmark_case_idx ON benchmark_cases(case_id);

-- V&V Results
CREATE TABLE vv_results (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    benchmark_id UUID NOT NULL REFERENCES benchmark_cases(id) ON DELETE CASCADE,
    simulation_id UUID NOT NULL REFERENCES simulations(id) ON DELETE CASCADE,
    calculated_keff REAL NOT NULL,
    statistical_uncertainty REAL NOT NULL,
    ce_ratio REAL NOT NULL,  -- C/E ratio
    delta_keff REAL NOT NULL,  -- (calc - ref) in pcm
    pass_fail BOOLEAN NOT NULL,  -- Within 2σ of reference
    run_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX vv_benchmark_idx ON vv_results(benchmark_id);
CREATE INDEX vv_sim_idx ON vv_results(simulation_id);

-- Regulatory Documents
CREATE TABLE regulatory_documents (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    doc_type TEXT NOT NULL,  -- FSAR_CH4 | FSAR_CH15 | 10CFR5059 | TECH_SPEC
    template_id TEXT,
    status TEXT NOT NULL DEFAULT 'draft',  -- draft | review | approved | submitted
    content_path TEXT,  -- S3 URL to generated PDF/Word
    linked_simulations UUID[] DEFAULT '{}',  -- Simulation IDs for traceability
    review_comments JSONB DEFAULT '[]',  -- Threaded comments
    created_by UUID NOT NULL REFERENCES users(id),
    approved_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX docs_project_idx ON regulatory_documents(project_id);
CREATE INDEX docs_type_idx ON regulatory_documents(doc_type);

-- Audit Log (immutable, append-only)
CREATE TABLE audit_log (
    id BIGSERIAL PRIMARY KEY,
    timestamp TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    user_id UUID REFERENCES users(id),
    action TEXT NOT NULL,  -- simulation_submit | result_view | model_edit | approval | export
    resource_type TEXT NOT NULL,
    resource_id UUID NOT NULL,
    details JSONB DEFAULT '{}',  -- Parameters changed, values before/after
    ip_address INET,
    session_id TEXT
);
CREATE INDEX audit_timestamp_idx ON audit_log(timestamp DESC);
CREATE INDEX audit_user_idx ON audit_log(user_id, timestamp DESC);
CREATE INDEX audit_resource_idx ON audit_log(resource_type, resource_id);

-- Usage Tracking (for billing enforcement)
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,  -- gpu_hours | storage_gb | simulations
    quantity REAL NOT NULL,
    period_start DATE NOT NULL,
    period_end DATE NOT NULL,
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
    pub credentials: Option<String>,
    pub plan: String,
    pub stripe_customer_id: Option<String>,
    pub stripe_subscription_id: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
    pub last_login_at: Option<DateTime<Utc>>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Organization {
    pub id: Uuid,
    pub name: String,
    pub slug: String,
    pub owner_id: Uuid,
    pub plan: String,
    pub nqa1_mode: bool,
    pub nuclear_data_libraries: Vec<String>,
    pub stripe_customer_id: Option<String>,
    pub stripe_subscription_id: Option<String>,
    pub settings: serde_json::Value,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Project {
    pub id: Uuid,
    pub org_id: Option<Uuid>,
    pub created_by: Uuid,
    pub name: String,
    pub description: String,
    pub reactor_type: String,
    pub visibility: String,
    pub qa_level: String,
    pub thumbnail_url: Option<String>,
    pub forked_from: Option<Uuid>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct ReactorModel {
    pub id: Uuid,
    pub project_id: Uuid,
    pub name: String,
    pub version: i32,
    pub parent_version_id: Option<Uuid>,
    pub geometry_type: String,
    pub geometry_data: Option<serde_json::Value>,
    pub core_layout: serde_json::Value,
    pub materials: serde_json::Value,
    pub boundary_conditions: serde_json::Value,
    pub created_by: Uuid,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Simulation {
    pub id: Uuid,
    pub project_id: Uuid,
    pub model_id: Uuid,
    pub submitted_by: Uuid,
    pub sim_type: String,
    pub solver: String,
    pub parameters: serde_json::Value,
    pub nuclear_data_library: String,
    pub status: String,
    pub execution_mode: String,
    pub gpu_hours_used: f32,
    pub cost_usd: f32,
    pub error_message: Option<String>,
    pub submitted_at: DateTime<Utc>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub parent_simulation_id: Option<Uuid>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct SimulationJob {
    pub id: Uuid,
    pub simulation_id: Uuid,
    pub worker_id: Option<String>,
    pub gpu_type: Option<String>,
    pub cores_allocated: i32,
    pub memory_gb: i32,
    pub priority: i32,
    pub progress_pct: f32,
    pub convergence_data: serde_json::Value,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct SimulationParams {
    pub solver_type: SolverType,
    pub monte_carlo: Option<MonteCarloParams>,
    pub sn: Option<SnParams>,
    pub temperature_kelvin: f64,  // Default 300K
    pub convergence: ConvergenceParams,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum SolverType {
    MonteCarlo,
    Sn,
    Coupled,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct MonteCarloParams {
    pub n_particles: u64,
    pub n_active_cycles: u32,
    pub n_inactive_cycles: u32,
    pub energy_mode: String,  // "continuous" | "multigroup"
}

#[derive(Debug, Deserialize, Serialize)]
pub struct SnParams {
    pub quadrature_order: u32,  // 4, 8, 16, 32
    pub mesh_cells_x: u32,
    pub mesh_cells_y: u32,
    pub mesh_cells_z: u32,
    pub num_groups: u32,
    pub use_dsa: bool,  // Diffusion synthetic acceleration
}

#[derive(Debug, Deserialize, Serialize)]
pub struct ConvergenceParams {
    pub keff_tolerance: f64,  // Default 1e-5
    pub fission_source_tolerance: f64,  // Default 1e-4
    pub entropy_bins: u32,  // Default 10 for each dimension
}
```

---

## Monte Carlo Solver Architecture Deep-Dive

### Governing Physics and Algorithms

ReactorSim's Monte Carlo solver implements event-based continuous-energy neutron transport using the standard Monte Carlo method employed by MCNP, Serpent, and OpenMC. The core algorithm simulates individual neutron histories through the reactor geometry, tracking collisions, scattering, absorption, and fission events using continuous-energy cross-section data.

**Neutron Transport Equation:**
The time-independent Boltzmann transport equation for neutron flux φ(r, E, Ω) is:

```
Ω·∇φ(r,E,Ω) + Σₜ(r,E)φ(r,E,Ω) = ∫∫ Σₛ(r,E'→E,Ω'→Ω)φ(r,E',Ω')dE'dΩ'
                                  + χ(E)/4π ∫∫ νΣf(r,E')φ(r,E',Ω')dE'dΩ'
```

Where:
- **Σₜ(r,E)**: Total macroscopic cross-section (interaction probability per unit path length)
- **Σₛ(r,E'→E,Ω'→Ω)**: Scattering cross-section (energy and angle transfer)
- **νΣf(r,E')**: Fission production cross-section (neutrons produced per fission)
- **χ(E)**: Fission spectrum (energy distribution of fission neutrons)

Monte Carlo solves this by sampling particle paths:

1. **Free flight distance**: `d = -ln(ξ) / Σₜ(E)` where ξ ~ U(0,1)
2. **Collision type**: Sample from Σ_scatter, Σ_absorption, Σ_fission probabilities
3. **Scattering angle**: Sample from angular distribution (isotropic, anisotropic via Legendre expansion)
4. **Energy change**: Sample from scattering kernel S(E'→E) or fission spectrum χ(E)
5. **Fission multiplicity**: Sample ν from P(ν) distribution (Poisson or tabulated)

**Eigenvalue (k-effective) Calculation:**
For critical reactor problems, k-eff is the dominant eigenvalue:

```
k_eff^(n+1) = (# fission neutrons produced in cycle n) / (# source neutrons in cycle n)

Estimated k_eff = (1/N) Σ k_eff^(n)  for n ∈ [inactive_cycles, total_cycles]

σ_k = sqrt( (1/(N-1)) Σ (k_eff^(n) - k̄)² )
```

**Fission Source Convergence:**
Shannon entropy H measures fission source convergence across spatial mesh:

```
H = - Σᵢ pᵢ ln(pᵢ)

where pᵢ = (# fission sites in bin i) / (total # fission sites)
```

Convergence is declared when H stabilizes (std dev < 0.05) for the last 50% of inactive cycles.

### GPU Particle Tracking Implementation

CUDA kernel design for massively parallel particle tracking:

```cuda
// solver-gpu/src/kernels/transport.cu

#include <curand_kernel.h>

struct Particle {
    double x, y, z;        // Position (cm)
    double u, v, w;        // Direction cosines
    double energy;         // Energy (MeV)
    double weight;         // Statistical weight
    int material;          // Current material ID
    int cell;              // Current geometry cell ID
    bool alive;            // Active particle flag
};

struct CrossSectionData {
    int n_energies;
    double* energy_grid;   // Energy points (MeV)
    double* total_xs;      // Total cross-section (barns)
    double* scatter_xs;
    double* absorption_xs;
    double* fission_xs;
    double* nu;            // Neutrons per fission
};

__global__ void track_particles_kernel(
    Particle* particles,
    int n_particles,
    CrossSectionData* xs_data,
    GeometryData* geometry,
    TallyAccumulator* tallies,
    curandState* rng_states,
    int max_collisions
) {
    int idx = blockIdx.x * blockDim.x + threadIdx.x;
    if (idx >= n_particles) return;

    Particle p = particles[idx];
    curandState rng = rng_states[idx];

    for (int coll = 0; coll < max_collisions && p.alive; ++coll) {
        // 1. Sample free flight distance
        double xi = curand_uniform_double(&rng);
        double total_xs = lookup_cross_section(xs_data, p.material, p.energy, XS_TOTAL);
        double distance = -log(xi) / total_xs;

        // 2. Transport to next collision or boundary
        double d_boundary = distance_to_boundary(geometry, p.x, p.y, p.z, p.u, p.v, p.w, p.cell);

        if (distance < d_boundary) {
            // Collision occurs before boundary
            p.x += distance * p.u;
            p.y += distance * p.v;
            p.z += distance * p.w;

            // 3. Sample collision type
            double scatter_xs = lookup_cross_section(xs_data, p.material, p.energy, XS_SCATTER);
            double absorb_xs = lookup_cross_section(xs_data, p.material, p.energy, XS_ABSORPTION);
            double fission_xs = lookup_cross_section(xs_data, p.material, p.energy, XS_FISSION);

            double r = curand_uniform_double(&rng) * total_xs;

            if (r < scatter_xs) {
                // Elastic or inelastic scattering
                sample_scattering_angle(&p, xs_data, &rng);
                sample_scattering_energy(&p, xs_data, &rng);
            } else if (r < scatter_xs + absorb_xs) {
                // Absorption (capture)
                p.alive = false;
                atomicAdd(&tallies->absorption_rate[p.cell], p.weight);
            } else {
                // Fission
                p.alive = false;  // Kill parent neutron

                // Bank fission site for next cycle
                double nu_avg = lookup_cross_section(xs_data, p.material, p.energy, XS_NU);
                int nu_sample = sample_fission_multiplicity(nu_avg, &rng);

                for (int n = 0; n < nu_sample; ++n) {
                    int bank_idx = atomicAdd(&tallies->fission_bank_count, 1);
                    if (bank_idx < MAX_FISSION_BANK_SIZE) {
                        FissionSite site;
                        site.x = p.x; site.y = p.y; site.z = p.z;
                        site.energy = sample_fission_spectrum(&rng);
                        tallies->fission_bank[bank_idx] = site;
                    }
                }
                atomicAdd(&tallies->fission_rate[p.cell], p.weight);
            }
        } else {
            // Cross boundary
            p.x += d_boundary * p.u;
            p.y += d_boundary * p.v;
            p.z += d_boundary * p.w;

            // Update cell and material
            int new_cell = find_cell(geometry, p.x, p.y, p.z);
            if (new_cell == -1) {
                // Leaked out of geometry
                p.alive = false;
                atomicAdd(&tallies->leakage_count, 1);
            } else {
                p.cell = new_cell;
                p.material = geometry->cell_materials[new_cell];
            }
        }

        // 4. Accumulate flux tally (track-length estimator)
        atomicAdd(&tallies->flux[p.cell], p.weight * distance);
    }

    particles[idx] = p;
    rng_states[idx] = rng;
}

__device__ double lookup_cross_section(
    CrossSectionData* xs, int material, double energy, int xs_type
) {
    // Binary search on energy grid + linear interpolation
    int n = xs->n_energies;
    int lo = 0, hi = n - 1;

    while (hi - lo > 1) {
        int mid = (lo + hi) / 2;
        if (energy < xs->energy_grid[mid]) hi = mid;
        else lo = mid;
    }

    double E0 = xs->energy_grid[lo];
    double E1 = xs->energy_grid[hi];
    double f = (energy - E0) / (E1 - E0);

    double* data = (xs_type == XS_TOTAL) ? xs->total_xs :
                   (xs_type == XS_SCATTER) ? xs->scatter_xs :
                   (xs_type == XS_ABSORPTION) ? xs->absorption_xs :
                   (xs_type == XS_FISSION) ? xs->fission_xs : xs->nu;

    return data[lo] + f * (data[hi] - data[lo]);
}
```

### Optimization Strategies for GPU Efficiency

1. **Particle sorting by material**: Group particles in the same material together to minimize divergence when looking up cross-sections (all threads in a warp access the same XS data).

2. **Cross-section caching in shared memory**: Pre-load frequently accessed XS data (U-235, O-16 in fuel) into shared memory to reduce global memory bandwidth.

3. **Atomic operation minimization**: Use thread-local tally accumulators and flush to global memory at end of kernel to reduce atomic contention.

4. **Warp-level fission banking**: Use warp shuffle instructions to aggregate fission sites within a warp before global atomic increment.

5. **Persistent kernel execution**: Keep GPU fully occupied by launching just enough thread blocks to saturate all SMs and process particles in batches.

---

## Sn Discrete Ordinates Solver Architecture

### Discretization and Sweep Algorithm

The deterministic Sn method solves the transport equation by discretizing angle, space, and energy:

```
Ωₘ·∇ψₘ,ᵢ,g + Σₜ,g ψₘ,ᵢ,g = Σₛ,g'→g ∑ₙ wₙ ψₙ,ᵢ,g' + χg (νΣf / k) ∑g' ∑ₙ wₙ ψₙ,ᵢ,g'

where:
  m = angular direction index (S₄ has 24 directions, S₈ has 80, S₁₆ has 288)
  i = spatial cell index
  g = energy group index
  wₙ = angular quadrature weight
```

**Source iteration (power method for k-eigenvalue):**

```
1. Initialize flux ψ⁽⁰⁾ and k⁽⁰⁾ = 1.0
2. For iteration n = 1, 2, ... until convergence:
   a. Build scattering + fission source:
      Q_m,i,g = Σₛ,g'→g ∑ₙ wₙ ψₙ,ᵢ,g' + χg (νΣf / k⁽ⁿ⁻¹⁾) ∑g' ∑ₙ wₙ ψₙ,ᵢ,g'
   b. Sweep through mesh for all angles/groups to solve:
      Ωₘ·∇ψₘ,ᵢ,g + Σₜ,g ψₘ,ᵢ,g = Q_m,i,g
   c. Update eigenvalue:
      k⁽ⁿ⁾ = k⁽ⁿ⁻¹⁾ · (new fission rate) / (old fission rate)
   d. Check convergence:
      |k⁽ⁿ⁾ - k⁽ⁿ⁻¹⁾| < ε_k  AND  ||ψ⁽ⁿ⁾ - ψ⁽ⁿ⁻¹⁾|| < ε_ψ
```

**Spatial differencing** (diamond difference for Cartesian mesh):

```
Face-averaged flux at i+1/2:
ψ_{i+1/2} = (ψᵢ + ψᵢ₊₁) / 2

Transport update for cell i:
ψᵢ₊₁ = [ (2μ/Δx)ψ_{i+1/2} + Q_i ] / [ (μ/Δx) + Σₜ ]

where μ = Ωₘ · x̂  (direction cosine in x-direction)
```

**Diffusion Synthetic Acceleration (DSA)** to accelerate convergence:

After each source iteration, solve the low-order diffusion equation:

```
-∇·D∇φ + Σₐφ = Q + (νΣf / k)φ

where D = 1/(3Σₜ) is the diffusion coefficient
```

Use the diffusion solution δφ to correct the Sn flux:
```
ψ⁽ⁿ⁺¹/²⁾ ← ψ⁽ⁿ⁺¹/²⁾ + α(δφ - φ⁽ⁿ⁾)
```

This typically reduces iterations by 5-10x for scattering-dominated problems.

### Rust Sn Solver Implementation Outline

```rust
// solver-sn/src/sweep.rs

use nalgebra_sparse::CsrMatrix;

pub struct SnMesh {
    pub nx: usize,
    pub ny: usize,
    pub nz: usize,
    pub dx: f64,
    pub dy: f64,
    pub dz: f64,
    pub cell_materials: Vec<usize>,  // Material ID per cell
}

pub struct SnQuadrature {
    pub order: usize,  // 4, 8, 16, 32
    pub n_angles: usize,
    pub mu: Vec<f64>,   // Direction cosines (x, y, z)
    pub eta: Vec<f64>,
    pub xi: Vec<f64>,
    pub weights: Vec<f64>,
}

pub struct SnSolver {
    pub mesh: SnMesh,
    pub quadrature: SnQuadrature,
    pub num_groups: usize,
    pub xs_data: MultiGroupXS,
    pub flux: Vec<f64>,  // Scalar flux [group][cell]
    pub angular_flux: Vec<f64>,  // Angular flux [angle][group][cell]
    pub fission_source: Vec<f64>,  // [group][cell]
}

impl SnSolver {
    pub fn solve_eigenvalue(&mut self, max_iter: usize, tol: f64) -> Result<f64, SolverError> {
        let mut keff = 1.0;
        let mut keff_old = 1.0;

        // Initial flux guess (uniform)
        self.flux.fill(1.0);

        for iter in 0..max_iter {
            keff_old = keff;

            // 1. Build scattering + fission source
            self.compute_source(keff);

            // 2. Sweep through all angles and groups
            for g in 0..self.num_groups {
                for m in 0..self.quadrature.n_angles {
                    self.sweep_angle_group(m, g);
                }

                // Update scalar flux for group g
                self.update_scalar_flux(g);
            }

            // 3. Compute new k-effective
            let fission_rate_new = self.compute_fission_rate();
            let fission_rate_old = self.compute_fission_rate_from_source();
            keff = keff_old * fission_rate_new / fission_rate_old;

            // 4. Check convergence
            let dk = (keff - keff_old).abs();
            if dk < tol {
                return Ok(keff);
            }

            // 5. Optional: DSA acceleration
            if iter % 5 == 0 {
                self.diffusion_acceleration(keff)?;
            }
        }

        Err(SolverError::Convergence(format!("Failed to converge after {} iterations", max_iter)))
    }

    fn sweep_angle_group(&mut self, angle_idx: usize, group: usize) {
        let (mu, eta, xi) = (
            self.quadrature.mu[angle_idx],
            self.quadrature.eta[angle_idx],
            self.quadrature.xi[angle_idx],
        );

        // Determine sweep order (upwind) based on direction cosines
        let (i_start, i_end, i_step) = if mu > 0.0 { (0, self.mesh.nx, 1) } else { (self.mesh.nx - 1, 0, -1) };
        let (j_start, j_end, j_step) = if eta > 0.0 { (0, self.mesh.ny, 1) } else { (self.mesh.ny - 1, 0, -1) };
        let (k_start, k_end, k_step) = if xi > 0.0 { (0, self.mesh.nz, 1) } else { (self.mesh.nz - 1, 0, -1) };

        // Sweep through mesh in upwind order
        for k in iterate_range(k_start, k_end, k_step) {
            for j in iterate_range(j_start, j_end, j_step) {
                for i in iterate_range(i_start, i_end, i_step) {
                    let cell_idx = self.mesh.cell_index(i, j, k);
                    let mat = self.mesh.cell_materials[cell_idx];

                    // Diamond difference update
                    let total_xs = self.xs_data.total_xs(mat, group);
                    let source = self.fission_source[group * self.mesh.num_cells() + cell_idx];

                    // Incoming flux from upstream faces
                    let psi_in_x = if mu > 0.0 && i > 0 {
                        self.angular_flux[self.face_flux_idx(i - 1, j, k, 0)]
                    } else {
                        0.0  // Boundary condition
                    };

                    // Similar for y and z faces...

                    // Balance equation: (Ω·∇ + Σₜ)ψ = Q
                    let psi_cell = self.diamond_difference_solve(
                        psi_in_x, psi_in_y, psi_in_z,
                        mu, eta, xi,
                        self.mesh.dx, self.mesh.dy, self.mesh.dz,
                        total_xs,
                        source
                    );

                    self.angular_flux[self.angular_flux_idx(angle_idx, group, cell_idx)] = psi_cell;
                }
            }
        }
    }

    fn diamond_difference_solve(
        &self,
        psi_in_x: f64, psi_in_y: f64, psi_in_z: f64,
        mu: f64, eta: f64, xi: f64,
        dx: f64, dy: f64, dz: f64,
        sigma_t: f64,
        source: f64,
    ) -> f64 {
        let coeff = sigma_t + mu.abs() / dx + eta.abs() / dy + xi.abs() / dz;
        let rhs = source +
            (mu.abs() / dx) * psi_in_x +
            (eta.abs() / dy) * psi_in_y +
            (xi.abs() / dz) * psi_in_z;
        rhs / coeff
    }
}
```

---

## Thermal-Hydraulic Coupling Architecture

### Single-Phase Subchannel Equations

The subchannel TH solver divides the reactor core into coolant subchannels (typically one subchannel per fuel pin or four subchannels per assembly). For each axial node `z` and subchannel `i`, the conservation equations are:

**Mass conservation:**
```
d(ρᵢAᵢ)/dt + d(ṁᵢ)/dz = wᵢⱼ
```

**Momentum conservation:**
```
d(ṁᵢ)/dt + d(ṁᵢ²/(ρᵢAᵢ))/dz = -Aᵢ dP/dz - fᵢ ṁᵢ|ṁᵢ|/(2ρᵢDₕ,ᵢAᵢ) - ρᵢAᵢg
```

**Energy conservation:**
```
d(ρᵢAᵢhᵢ)/dt + d(ṁᵢhᵢ)/dz = q'ᵢ + ∑ⱼ wᵢⱼhⱼ
```

Where:
- **ρᵢ**: Coolant density in subchannel i (kg/m³) from IAPWS-IF97
- **Aᵢ**: Flow area (m²)
- **ṁᵢ**: Mass flow rate (kg/s)
- **wᵢⱼ**: Crossflow from subchannel j to i (turbulent mixing + void drift)
- **P**: Pressure (MPa)
- **fᵢ**: Friction factor (Blasius or Churchill correlation)
- **Dₕ**: Hydraulic diameter (m)
- **hᵢ**: Enthalpy (J/kg)
- **q'ᵢ**: Linear heat rate from fuel pin (W/m)

**Coupled iteration with neutronics:**

```
1. Initialize: T_fuel, T_coolant, ρ_coolant from nominal conditions
2. For each coupling iteration:
   a. Run neutronics (MC or Sn) with current temperatures/densities
      → Get power distribution q'''(x,y,z)
   b. Map power to subchannel linear heat rates q'ᵢ(z)
   c. Solve TH equations → new T_coolant, ρ_coolant, ΔP
   d. Compute fuel temperature from 1D radial heat conduction:
      T_fuel = T_coolant + q' / (2πhₑₑ) + q' ln(r_clad/r_fuel) / (2πk_fuel)
   e. Update XS with Doppler broadening:
      σ(E, T_fuel_new) via sigma1 method or interpolation
   f. Check convergence:
      |k_eff - k_eff_old| < tol  AND  ||T - T_old||₂ < tol
3. Converged → final power distribution, temperatures, DNBR margins
```

**Fuel temperature calculation** (radial 1D steady-state):

```rust
// solver-th/src/fuel_temperature.rs

pub fn compute_fuel_centerline_temperature(
    linear_heat_rate: f64,  // W/m
    coolant_temp: f64,      // K
    fuel_radius: f64,       // m
    clad_radius: f64,       // m
    gap_conductance: f64,   // W/m²·K (Ross-Stoute model)
    fuel_conductivity: f64, // W/m·K (function of T and burnup)
    clad_conductivity: f64, // W/m·K
) -> f64 {
    use std::f64::consts::PI;

    let q_prime = linear_heat_rate;

    // Temperature rise from coolant to clad outer surface (convection)
    let h_eff = 10000.0;  // Convective heat transfer coefficient W/m²·K (Dittus-Boelter)
    let delta_T_conv = q_prime / (2.0 * PI * clad_radius * h_eff);

    // Temperature rise across cladding (conduction)
    let delta_T_clad = (q_prime / (2.0 * PI * clad_conductivity))
        * (clad_radius / fuel_radius).ln();

    // Temperature rise across gap (gas conductance)
    let delta_T_gap = q_prime / (2.0 * PI * fuel_radius * gap_conductance);

    // Temperature rise within fuel pellet (conduction, non-uniform k(T))
    let delta_T_fuel = q_prime / (4.0 * PI * fuel_conductivity);  // Simplified

    let t_centerline = coolant_temp + delta_T_conv + delta_T_clad + delta_T_gap + delta_T_fuel;

    t_centerline
}
```

---

## Architecture Deep-Dives

### 1. Simulation Submission API Handler (Rust/Axum)

Primary endpoint for submitting reactor simulations, validates model, routes to WASM or GPU execution, and enqueues GPU jobs.

```rust
// src/api/handlers/simulation.rs

use axum::{
    extract::{Path, State, Json},
    http::StatusCode,
    response::IntoResponse,
};
use uuid::Uuid;
use crate::{
    db::models::{Simulation, SimulationParams, SolverType},
    state::AppState,
    auth::Claims,
    error::ApiError,
};

#[derive(serde::Deserialize)]
pub struct CreateSimulationRequest {
    pub solver_type: SolverType,
    pub parameters: serde_json::Value,
}

pub async fn create_simulation(
    State(state): State<AppState>,
    claims: Claims,
    Path((project_id, model_id)): Path<(Uuid, Uuid)>,
    Json(req): Json<CreateSimulationRequest>,
) -> Result<impl IntoResponse, ApiError> {
    // 1. Verify project and model ownership
    let model = sqlx::query_as!(
        crate::db::models::ReactorModel,
        "SELECT rm.* FROM reactor_models rm
         JOIN projects p ON rm.project_id = p.id
         WHERE rm.id = $1 AND p.id = $2
         AND (p.created_by = $3 OR p.org_id IN (
             SELECT org_id FROM org_members WHERE user_id = $3
         ))",
        model_id,
        project_id,
        claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Reactor model not found"))?;

    // 2. Parse parameters
    let params: SimulationParams = serde_json::from_value(req.parameters.clone())?;

    // 3. Estimate computational cost and check plan limits
    let user = sqlx::query_as!(
        crate::db::models::User,
        "SELECT * FROM users WHERE id = $1",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    let n_particles = params.monte_carlo.as_ref().map(|mc| mc.n_particles).unwrap_or(0);

    if user.plan == "academic" && n_particles > 10_000 {
        return Err(ApiError::PlanLimit(
            "Academic plan limited to 10,000 particles. Upgrade to Pro for full-core MC."
        ));
    }

    // 4. Determine execution mode
    let execution_mode = if n_particles < 1000 && req.solver_type == SolverType::MonteCarlo {
        "wasm"
    } else {
        "gpu"
    };

    // 5. Create simulation record
    let sim = sqlx::query_as!(
        Simulation,
        r#"INSERT INTO simulations
            (project_id, model_id, submitted_by, sim_type, solver, parameters,
             nuclear_data_library, status, execution_mode)
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9)
        RETURNING *"#,
        project_id,
        model_id,
        claims.user_id,
        "eigenvalue",  // Default for MVP
        serde_json::to_string(&req.solver_type)?,
        req.parameters,
        "endfb8",
        if execution_mode == "wasm" { "completed" } else { "queued" },
        execution_mode,
    )
    .fetch_one(&state.db)
    .await?;

    // 6. For GPU execution, enqueue job
    if execution_mode == "gpu" {
        let priority = match user.plan.as_str() {
            "enterprise" => 100,
            "team" => 50,
            "pro" => 10,
            _ => 0,
        };

        let job = sqlx::query_as!(
            crate::db::models::SimulationJob,
            r#"INSERT INTO simulation_jobs (simulation_id, priority, memory_gb)
            VALUES ($1, $2, $3) RETURNING *"#,
            sim.id,
            priority,
            64,  // Default 64GB for full-core MC
        )
        .fetch_one(&state.db)
        .await?;

        // Publish to Redis job queue
        let job_msg = serde_json::json!({
            "job_id": job.id,
            "simulation_id": sim.id,
            "priority": priority,
        });
        state.redis
            .publish("simulation:jobs", serde_json::to_string(&job_msg)?)
            .await?;
    }

    // 7. Audit log
    sqlx::query!(
        "INSERT INTO audit_log (user_id, action, resource_type, resource_id, details)
         VALUES ($1, $2, $3, $4, $5)",
        claims.user_id,
        "simulation_submit",
        "simulation",
        sim.id,
        req.parameters,
    )
    .execute(&state.db)
    .await?;

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
         AND (p.created_by = $3 OR p.org_id IN (
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
```

### 2. GPU Monte Carlo Worker (Rust + CUDA Integration)

Background worker that picks up GPU simulation jobs, runs CUDA kernels, streams progress, and stores results in S3.

```rust
// src/workers/gpu_monte_carlo_worker.rs

use std::sync::Arc;
use aws_sdk_s3::Client as S3Client;
use redis::AsyncCommands;
use sqlx::PgPool;
use uuid::Uuid;

use crate::{
    solver_gpu::MonteCarloEngine,
    db::models::{SimulationJob, SimulationParams},
    websocket::ProgressBroadcaster,
};

pub struct GpuMonteCarloWorker {
    db: PgPool,
    redis: redis::Client,
    s3: S3Client,
    broadcaster: Arc<ProgressBroadcaster>,
    gpu_device: i32,
}

impl GpuMonteCarloWorker {
    pub async fn run(&self) -> anyhow::Result<()> {
        let mut conn = self.redis.get_async_connection().await?;
        tracing::info!("GPU MC worker started on device {}, waiting for jobs...", self.gpu_device);

        loop {
            // Priority-based pop from Redis sorted set
            let job_msg: Option<String> = conn.brpop("simulation:jobs", 30.0).await?;
            let Some(job_msg) = job_msg else { continue };

            let job_info: serde_json::Value = serde_json::from_str(&job_msg)?;
            let job_id: Uuid = serde_json::from_value(job_info["job_id"].clone())?;

            if let Err(e) = self.process_job(job_id).await {
                tracing::error!("Job {job_id} failed: {e}");
                self.mark_failed(job_id, &e.to_string()).await?;
            }
        }
    }

    async fn process_job(&self, job_id: Uuid) -> anyhow::Result<()> {
        // 1. Claim job and fetch simulation details
        let job = sqlx::query_as!(
            SimulationJob,
            "UPDATE simulation_jobs SET started_at = NOW(), worker_id = $2, gpu_type = $3
             WHERE id = $1 RETURNING *",
            job_id,
            hostname::get()?.to_string_lossy().to_string(),
            format!("A100-{}", self.gpu_device)
        )
        .fetch_one(&self.db)
        .await?;

        let sim = sqlx::query_as!(
            crate::db::models::Simulation,
            "UPDATE simulations SET status = 'running', started_at = NOW()
             WHERE id = $1 RETURNING *",
            job.simulation_id
        )
        .fetch_one(&self.db)
        .await?;

        // 2. Load reactor model and nuclear data
        let model = sqlx::query_as!(
            crate::db::models::ReactorModel,
            "SELECT * FROM reactor_models WHERE id = $1",
            sim.model_id
        )
        .fetch_one(&self.db)
        .await?;

        let params: SimulationParams = serde_json::from_value(sim.parameters.clone())?;
        let mc_params = params.monte_carlo.ok_or(anyhow::anyhow!("Missing MC parameters"))?;

        // 3. Initialize CUDA Monte Carlo engine
        let mut mc_engine = MonteCarloEngine::new(self.gpu_device)?;
        mc_engine.load_geometry(&model.geometry_data, &model.core_layout)?;
        mc_engine.load_cross_sections(&sim.nuclear_data_library, params.temperature_kelvin).await?;

        // 4. Run simulation with progress callback
        let progress_tx = self.broadcaster.get_channel(sim.id);

        let result = mc_engine.run_eigenvalue(
            mc_params.n_particles,
            mc_params.n_inactive_cycles,
            mc_params.n_active_cycles,
            |cycle, keff, entropy| {
                // Progress callback from GPU
                let pct = (cycle as f32 / (mc_params.n_inactive_cycles + mc_params.n_active_cycles) as f32) * 100.0;

                let _ = progress_tx.send(serde_json::json!({
                    "progress_pct": pct,
                    "current_cycle": cycle,
                    "keff": keff,
                    "entropy": entropy,
                }));

                // Update database every 10 cycles
                if cycle % 10 == 0 {
                    tokio::task::block_in_place(|| {
                        tokio::runtime::Handle::current().block_on(async {
                            let conv_data = serde_json::json!([{"cycle": cycle, "keff": keff, "entropy": entropy}]);
                            let _ = sqlx::query!(
                                "UPDATE simulation_jobs SET progress_pct = $1, convergence_data = convergence_data || $2::jsonb
                                 WHERE id = $3",
                                pct, conv_data, job_id
                            )
                            .execute(&self.db)
                            .await;
                        })
                    });
                }
            }
        )?;

        // 5. Extract tallies and prepare result data
        let flux_mesh = mc_engine.get_flux_tally()?;
        let power_dist = mc_engine.get_power_distribution()?;

        let summary = serde_json::json!({
            "keff": result.keff,
            "keff_std": result.keff_std,
            "shannon_entropy_final": result.shannon_entropy.last().unwrap(),
            "active_cycles": mc_params.n_active_cycles,
            "total_particles": mc_params.n_particles,
        });

        // 6. Save results to S3
        let result_key = format!("results/{}/{}/mc_result.h5", sim.project_id, sim.id);
        let result_hdf5 = mc_engine.export_hdf5(&result)?;

        self.s3.put_object()
            .bucket("reactorsim-results")
            .key(&result_key)
            .body(result_hdf5.into())
            .content_type("application/x-hdf5")
            .send()
            .await?;

        let results_url = format!("s3://reactorsim-results/{}", result_key);

        // 7. Update database
        let gpu_hours = result.wall_time_sec / 3600.0;
        let cost_usd = gpu_hours * 2.5;  // $2.50/GPU-hour

        sqlx::query!(
            "UPDATE simulations SET status = 'completed', completed_at = NOW(),
             gpu_hours_used = $2, cost_usd = $3
             WHERE id = $1",
            sim.id, gpu_hours as f32, cost_usd as f32
        )
        .execute(&self.db)
        .await?;

        sqlx::query!(
            "INSERT INTO simulation_results
             (simulation_id, result_type, summary_data, full_result_path, convergence_data)
             VALUES ($1, $2, $3, $4, $5)",
            sim.id,
            "keff",
            summary,
            results_url,
            serde_json::to_value(&result.convergence_history)?
        )
        .execute(&self.db)
        .await?;

        sqlx::query!(
            "UPDATE simulation_jobs SET completed_at = NOW(), progress_pct = 100.0
             WHERE id = $1",
            job_id
        )
        .execute(&self.db)
        .await?;

        // 8. Track usage
        let period_start = chrono::Utc::now().date_naive();
        let period_end = period_start + chrono::Duration::days(30);

        sqlx::query!(
            "INSERT INTO usage_records (user_id, record_type, quantity, period_start, period_end)
             VALUES ($1, 'gpu_hours', $2, $3, $4)",
            sim.submitted_by,
            gpu_hours,
            period_start,
            period_end
        )
        .execute(&self.db)
        .await?;

        // 9. Notify client
        self.broadcaster.send(sim.id, serde_json::json!({
            "status": "completed",
            "keff": result.keff,
            "keff_std": result.keff_std,
            "results_url": results_url,
        }))?;

        tracing::info!("Job {job_id} completed: k-eff = {:.5} ± {:.5}", result.keff, result.keff_std);
        Ok(())
    }

    async fn mark_failed(&self, job_id: Uuid, error: &str) -> anyhow::Result<()> {
        // ... similar to SpiceSim implementation
        Ok(())
    }
}
```

### 3. Three.js Reactor Core Viewer (React + WebGL)

Interactive 3D visualization with instanced rendering for 50,000+ fuel pins and scalar field overlays.

```typescript
// frontend/src/components/ReactorViewer/ReactorViewer.tsx

import { useRef, useEffect, useState } from 'react';
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';
import { useReactorStore } from '../../stores/reactorStore';

interface FuelPinInstance {
  position: [number, number, number];
  enrichment: number;
  burnup: number;
  power: number;
}

export function ReactorViewer() {
  const canvasRef = useRef<HTMLCanvasElement>(null);
  const { model, powerDistribution, scalarField } = useReactorStore();
  const [crossSection, setCrossSection] = useState({ enabled: false, z: 0 });

  useEffect(() => {
    if (!canvasRef.current || !model) return;

    // Initialize Three.js scene
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x1a1a1a);

    const camera = new THREE.PerspectiveCamera(
      60,
      canvasRef.current.width / canvasRef.current.height,
      0.1,
      1000
    );
    camera.position.set(300, 200, 300);

    const renderer = new THREE.WebGLRenderer({
      canvas: canvasRef.current,
      antialias: true,
    });
    renderer.setSize(canvasRef.current.width, canvasRef.current.height);
    renderer.setPixelRatio(window.devicePixelRatio);

    const controls = new OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true;

    // Lighting
    const ambientLight = new THREE.AmbientLight(0xffffff, 0.4);
    scene.add(ambientLight);
    const directionalLight = new THREE.DirectionalLight(0xffffff, 0.6);
    directionalLight.position.set(100, 200, 100);
    scene.add(directionalLight);

    // Core boundary (vessel)
    const vesselGeometry = new THREE.CylinderGeometry(
      model.core_layout.vessel_radius,
      model.core_layout.vessel_radius,
      model.core_layout.active_height,
      64,
      1,
      true
    );
    const vesselMaterial = new THREE.MeshStandardMaterial({
      color: 0x444444,
      transparent: true,
      opacity: 0.15,
      side: THREE.DoubleSide,
    });
    const vessel = new THREE.Mesh(vesselGeometry, vesselMaterial);
    scene.add(vessel);

    // Fuel pins using instanced mesh rendering
    const pinGeometry = new THREE.CylinderGeometry(
      model.pin_radius,
      model.pin_radius,
      model.pin_height,
      16
    );

    const instanceCount = model.fuel_pins.length;
    const instancedMesh = new THREE.InstancedMesh(
      pinGeometry,
      new THREE.MeshStandardMaterial({ color: 0x00ff00 }),
      instanceCount
    );

    const matrix = new THREE.Matrix4();
    const color = new THREE.Color();

    model.fuel_pins.forEach((pin: FuelPinInstance, i: number) => {
      // Position matrix
      matrix.setPosition(pin.position[0], pin.position[1], pin.position[2]);
      instancedMesh.setMatrixAt(i, matrix);

      // Color by power (scalar field overlay)
      if (scalarField === 'power' && powerDistribution) {
        const powerNormalized = pin.power / powerDistribution.max_power;
        color.setHSL(0.7 - powerNormalized * 0.7, 1.0, 0.5);  // Blue (low) → Red (high)
        instancedMesh.setColorAt(i, color);
      } else if (scalarField === 'burnup') {
        const burnupNormalized = pin.burnup / 60.0;  // 0-60 GWd/MTU
        color.setHSL(0.33, 1.0 - burnupNormalized, 0.5);  // Green (fresh) → Yellow (depleted)
        instancedMesh.setColorAt(i, color);
      }
    });

    instancedMesh.instanceMatrix.needsUpdate = true;
    if (instancedMesh.instanceColor) instancedMesh.instanceColor.needsUpdate = true;

    scene.add(instancedMesh);

    // Cross-section plane
    let clipPlane: THREE.Plane | null = null;
    if (crossSection.enabled) {
      clipPlane = new THREE.Plane(new THREE.Vector3(0, 1, 0), -crossSection.z);
      renderer.clippingPlanes = [clipPlane];

      // Visualize plane
      const planeHelper = new THREE.PlaneHelper(clipPlane, 200, 0xffff00);
      scene.add(planeHelper);
    }

    // Animation loop
    function animate() {
      requestAnimationFrame(animate);
      controls.update();
      renderer.render(scene, camera);
    }
    animate();

    // Cleanup
    return () => {
      renderer.dispose();
      pinGeometry.dispose();
      vesselGeometry.dispose();
      scene.clear();
    };
  }, [model, powerDistribution, scalarField, crossSection]);

  return (
    <div className="reactor-viewer w-full h-full relative">
      <canvas ref={canvasRef} className="w-full h-full" />

      <div className="absolute top-4 right-4 bg-gray-800 bg-opacity-90 p-4 rounded text-white">
        <h3 className="text-sm font-bold mb-2">Scalar Field Overlay</h3>
        <select
          className="bg-gray-700 text-white p-1 rounded"
          onChange={(e) => useReactorStore.setState({ scalarField: e.target.value })}
        >
          <option value="">None</option>
          <option value="power">Power Density</option>
          <option value="flux">Thermal Flux</option>
          <option value="burnup">Burnup</option>
          <option value="temperature">Fuel Temperature</option>
        </select>

        <h3 className="text-sm font-bold mt-4 mb-2">Cross-Section</h3>
        <label className="flex items-center">
          <input
            type="checkbox"
            checked={crossSection.enabled}
            onChange={(e) => setCrossSection({ ...crossSection, enabled: e.target.checked })}
            className="mr-2"
          />
          Enable
        </label>
        {crossSection.enabled && (
          <input
            type="range"
            min={-model?.core_layout.active_height / 2 || 0}
            max={model?.core_layout.active_height / 2 || 100}
            step={1}
            value={crossSection.z}
            onChange={(e) => setCrossSection({ ...crossSection, z: parseFloat(e.target.value) })}
            className="w-full mt-2"
          />
        )}
      </div>
    </div>
  );
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–4)

**Day 1: Project initialization and Rust backend scaffold**
```bash
cargo init reactorsim-api
cd reactorsim-api
cargo add axum tokio serde serde_json sqlx uuid chrono tower-http jsonwebtoken bcrypt
cargo add --features postgres,runtime-tokio-rustls,uuid,chrono,json sqlx
cargo add aws-sdk-s3 redis typst
```
- `src/main.rs` — Axum app with CORS, tracing, graceful shutdown
- `src/config.rs` — Environment configuration (DATABASE_URL, JWT_SECRET, S3_BUCKET, REDIS_URL, CUDA_DEVICE)
- `src/state.rs` — AppState struct (PgPool, Redis, S3Client)
- `src/error.rs` — ApiError enum with IntoResponse
- `Dockerfile` — Multi-stage build with CUDA runtime
- `docker-compose.yml` — PostgreSQL, Redis, MinIO

**Day 2: Database schema and migrations**
- `migrations/001_initial.sql` — All tables: users, organizations, org_members, projects, reactor_models, fuel_assembly_templates, simulations, simulation_jobs, simulation_results, depletion_states, benchmark_cases, vv_results, regulatory_documents, audit_log, usage_records
- `src/db/mod.rs` — Database pool initialization
- `src/db/models.rs` — All SQLx structs with FromRow derives
- Run `sqlx migrate run` to apply schema
- Seed script for 5 ICSBEP benchmark cases

**Day 3: Authentication system**
- `src/auth/mod.rs` — JWT token generation and validation middleware
- `src/auth/saml.rs` — SAML/OIDC SSO flow handlers (Okta, Azure AD)
- `src/api/handlers/auth.rs` — Register, login, SSO callback, refresh token, me endpoints
- Password hashing with bcrypt (cost factor 12)
- JWT with 24h access token + 30d refresh token
- Auth middleware with role extraction (admin/modeler/reviewer/viewer)

**Day 4: User, organization, and project CRUD**
- `src/api/handlers/users.rs` — Get profile, update credentials, delete account
- `src/api/handlers/projects.rs` — Create, list, get, update, delete, fork project
- `src/api/handlers/orgs.rs` — Create org, invite member (email), list members, update roles, remove member
- `src/api/router.rs` — All route definitions with auth + role middleware
- Integration tests: auth flow, project CRUD, authorization checks (modeler cannot approve, reviewer can)

### Phase 2 — Monte Carlo Core (Days 5–12)

**Day 5: CUDA Monte Carlo kernel scaffolding**
- `solver-gpu/` — New Rust workspace with CUDA integration via `cudarc` or `cuda-sys`
- `solver-gpu/src/kernels/transport.cu` — Basic particle tracking kernel stub
- `solver-gpu/src/lib.rs` — Rust FFI bindings to CUDA kernels
- CUDA memory management: device allocation for particles, cross-sections, tallies
- Unit test: single particle, single collision in infinite medium

**Day 6: Cross-section data structures and loading**
- `solver-gpu/src/cross_sections.rs` — ENDF/B-VIII.0 pointwise cross-section loader
- Binary search + linear interpolation on energy grid (GPU-optimized)
- Cross-section caching strategy: load U-235, U-238, O-16, H-1 to GPU global memory
- Doppler broadening via sigma1 method or pre-tabulated at T = 300K, 600K, 900K, 1200K
- Test: verify σₜ, σₛ, σf, ν for U-235 at thermal energy

**Day 7: Geometry engine (CSG for simple models)**
- `solver-gpu/src/geometry.rs` — Constructive solid geometry: spheres, cylinders, boxes, half-spaces
- `solver-gpu/src/kernels/geometry.cu` — `distance_to_boundary()` and `find_cell()` GPU kernels
- Universe and lattice support for repeated fuel pin structures
- Ray-tracing through pin lattice with material ID lookup
- Test: particle tracking through 3×3 pin lattice, verify cell transitions

**Day 8: Event-based Monte Carlo tracking kernel**
- `solver-gpu/src/kernels/transport.cu` — Full tracking loop: free flight, collision sampling, boundary crossing
- cuRAND for random number generation (thread-local RNG state)
- Collision type sampling: scatter vs. absorption vs. fission
- Isotropic scattering in LAB frame (simplified, no anisotropy yet)
- Test: criticality of infinite medium U-235 solution, compare k-eff to analytical

**Day 9: Fission source iteration and k-eigenvalue**
- `solver-gpu/src/eigenvalue.rs` — Power iteration: inactive cycles, active cycles, k-eff accumulation
- Fission bank: atomic append to global fission site array
- Source normalization between cycles
- Shannon entropy calculation for source convergence diagnostics
- Test: critical sphere (Godiva), k-eff = 1.0 within 3σ

**Day 10: Tally accumulation (flux, power, reaction rates)**
- `solver-gpu/src/tallies.rs` — Track-length estimator for flux, collision estimator for reaction rates
- Mesh tally: regular Cartesian grid overlaid on geometry
- Atomic additions to tally bins (handle contention)
- Statistical uncertainty propagation: batch means method
- Test: thermal flux in water moderator, compare to analytical diffusion solution

**Day 11: Performance optimization**
- Particle sorting by material (reduce warp divergence during XS lookup)
- Coalesced memory access patterns for XS data
- Shared memory caching of frequently accessed XS (U-235 thermal)
- Persistent kernel execution with particle batching
- Benchmark: 1M particles in PWR pin cell, target <10 seconds on A100

**Day 12: Rust API integration and HDF5 export**
- `solver-gpu/src/api.rs` — High-level API: `MonteCarloEngine::run_eigenvalue()`
- Export results to HDF5 format: k-eff, tallies, convergence history, fission source distribution
- S3 upload integration
- Integration test: Run full pin cell eigenvalue, verify k-eff and flux shape

### Phase 3 — Sn Solver + Nuclear Data Service (Days 13–18)

**Day 13: Multi-group cross-section library**
- `nuclear-data-service/` — Python FastAPI service for XS processing
- `nuclear-data-service/endf_parser.py` — Parse ENDF/B-VIII.0 files (MF=3 cross-sections, MF=5 energy distributions)
- Generate 7-group, 47-group, 252-group libraries with flux-weighted collapse
- Export to JSON format for Rust consumption
- Test: U-235 fission XS at thermal energy matches ENDF exactly

**Day 14: Sn angular quadrature generation**
- `solver-sn/` — New Rust workspace member
- `solver-sn/src/quadrature.rs` — Level-symmetric quadrature sets (S4, S8, S16)
- Gauss-Legendre-Chebyshev product quadrature for octant symmetry
- Test: verify quadrature weights sum to 4π

**Day 15: Sn sweep algorithm for 2D Cartesian**
- `solver-sn/src/sweep.rs` — Diamond difference spatial discretization
- Upwind sweep based on direction cosines
- Source iteration for k-eigenvalue problems
- Test: 1D slab reactor (infinite in y, z), compare k-eff to analytical

**Day 16: Diffusion synthetic acceleration (DSA)**
- `solver-sn/src/dsa.rs` — Low-order diffusion equation solve via conjugate gradient
- Diffusion coefficient D = 1/(3Σₜ) with boundary conditions
- Accelerated source update after each Sn sweep
- Test: verify DSA reduces iterations by 5x for water-moderated lattice

**Day 17: 3D Cartesian Sn solver**
- Extend sweep to 3D with corner balance spatial differencing
- GPU acceleration: parallel sweep over energy groups
- Test: 3D PWR assembly, compare to MC reference

**Day 18: Coupled Sn + MC workflow**
- Run Sn first to get fission source distribution
- Use Sn flux as initial guess for MC inactive cycles
- Verify MC inactive cycles reduced from 100 to 20
- Integration test: full-core BWR Sn → MC handoff

### Phase 4 — Frontend 3D Viewer + Core Designer (Days 19–24)

**Day 19: React frontend scaffold**
```bash
npm create vite@latest frontend -- --template react-ts
cd frontend
npm i three @react-three/fiber @react-three/drei zustand @tanstack/react-query axios react-flow-renderer d3
```
- `src/App.tsx` — Router, auth context, layout
- `src/stores/authStore.ts` — Auth state (JWT, user, plan)
- `src/stores/reactorStore.ts` — Reactor model state
- `src/lib/api.ts` — Axios API client with JWT interceptor

**Day 20: Three.js reactor core viewer foundation**
- `src/components/ReactorViewer/ReactorViewer.tsx` — Three.js scene setup, camera, orbit controls
- `src/components/ReactorViewer/FuelPinInstances.tsx` — Instanced mesh for 50,000+ pins
- Cylinder geometry for fuel pins, vessel boundary
- Basic color-coding by enrichment
- Test: Render 17×17 PWR assembly with 264 pins at 60 FPS

**Day 21: Scalar field overlays (power, flux, burnup)**
- Custom fragment shader for color map lookup (blue → green → yellow → red)
- Upload per-instance attributes: power, flux, burnup
- Color scale legend with min/max values
- Toggle between scalar fields via dropdown
- Test: Verify power peaking visualization matches tally data

**Day 22: Cross-section plane cutting**
- Three.js clipping planes for axial and radial cuts
- PlaneHelper to visualize cutting plane position
- Slider control for plane position (z-axis for axial)
- Arbitrary angle plane support (future enhancement)
- Test: Axial cut at mid-plane shows correct pin layout

**Day 23: Core loading pattern editor (2D)**
- `src/components/CoreDesigner/LoadingPatternEditor.tsx` — React Flow or custom 2D canvas
- Drag-and-drop fuel assemblies from palette to core positions
- Quarter-core symmetry enforcement (mirror placements)
- Assembly color-coding by type (fresh, once-burned, twice-burned)
- Test: 15×15 PWR core layout with quarter-core symmetry

**Day 24: Assembly template library and parametric design**
- `src/components/CoreDesigner/AssemblyPalette.tsx` — Library of assembly templates
- Pin layout editor: enrichment zones, burnable absorber positions
- Save custom assembly templates to database
- Parametric sweeps: vary enrichment, generate multiple loading patterns
- Test: Create custom assembly with 3-zone enrichment profile

### Phase 5 — Thermal-Hydraulics + Coupled Iteration (Days 25–29)

**Day 25: Subchannel TH solver foundation**
- `solver-th/` — New Rust workspace member
- `solver-th/src/subchannel.rs` — Single-phase mass, momentum, energy conservation equations
- `solver-th/src/steam_tables.rs` — IAPWS-IF97 water properties (ρ, h, Cp as functions of T, P)
- Finite volume discretization on 1D axial mesh per subchannel
- Test: Single heated channel, verify outlet temperature vs. analytical

**Day 26: Fuel temperature calculation**
- `solver-th/src/fuel_temp.rs` — 1D radial heat conduction through fuel pellet, gap, cladding
- Ross-Stoute gap conductance model
- Fuel thermal conductivity degradation with burnup
- Test: Compare centerline temperature to FRAPCON reference

**Day 27: Doppler feedback in Monte Carlo**
- `solver-gpu/src/doppler.rs` — On-the-fly Doppler broadening of cross-sections
- Temperature-dependent XS interpolation: σ(E, T_fuel)
- Update cross-sections each coupling iteration
- Test: Verify Doppler reactivity coefficient (dk/dT) for UO2 fuel

**Day 28: Coupled neutronics-TH iteration (Picard)**
- `solver-coupled/src/picard.rs` — Fixed-point iteration: Neutronics → TH → Neutronics
- Convergence criteria: |k_eff - k_eff_old| < 1e-5 AND ||T - T_old||₂ < 1K
- Relaxation factor α = 0.5 to stabilize oscillations
- Test: PWR hot full power with feedback, k-eff with/without TH

**Day 29: DNBR calculation and CHF correlation**
- `solver-th/src/chf.rs` — W-3 correlation for PWR CHF prediction
- Departure from nucleate boiling ratio (DNBR) = CHF / q''
- Identify hot channels with minimum DNBR
- Test: Verify DNBR > 1.3 safety limit for design basis conditions

### Phase 6 — Depletion Solver (Days 30–33)

**Day 30: Bateman equation solver (CRAM)**
- `solver-depletion/` — New Rust workspace member
- `solver-depletion/src/cram.rs` — Chebyshev rational approximation for matrix exponential
- Build transmutation matrix A (decay + reaction rates)
- Solve N(t) = exp(At) N(0) via CRAM-16
- Test: U-235 burnup, verify Pu-239 buildup vs. ORIGEN

**Day 31: Multi-step depletion with predictor-corrector**
- `solver-depletion/src/predictor_corrector.rs` — CE/LI or CE/CE methods
- Predictor: flux at start of step → mid-step nuclide vector
- Corrector: re-run neutronics at mid-step → final nuclide vector
- Test: 3-step burnup (0, 15, 30 GWd/MTU), compare to Serpent

**Day 32: Isotopic inventory tracking**
- Track 500+ nuclides: U, Pu, Am, Cm chains + key fission products (Xe-135, Sm-149)
- Export isotopic masses per region to HDF5
- Decay heat calculation via ANS 5.1 standard
- Test: Spent fuel decay heat at 1 day, 1 year, 10 years cooling

**Day 33: Integration with Monte Carlo**
- `solver-coupled/src/depletion.rs` — Depletion-coupled MC: Run MC → update nuclide densities → next burnup step
- Multi-cycle capability: track assemblies across shuffles
- Equilibrium cycle search (future enhancement)
- Test: Single-assembly burnup to 50 GWd/MTU, verify reactivity vs. burnup curve

### Phase 7 — Billing + V&V + Polish (Days 34–38)

**Day 34: Stripe integration**
- `src/api/handlers/billing.rs` — Create checkout session, customer portal, subscription status
- `src/api/handlers/webhooks/stripe.rs` — Subscription events handling
- Plan mapping: Academic (free), Pro ($299/mo), Team ($799/mo + $149/seat), Enterprise (custom)
- Test: Subscribe to Pro, verify plan updated and limits lifted

**Day 35: Usage tracking and enforcement**
- `src/middleware/plan_limits.rs` — Enforce GPU-hour quotas per plan
- Usage dashboard: `src/api/handlers/usage.rs` — Current billing period usage
- Approaching-limit warnings at 80% and 100%
- Test: Free user hits 10 GPU-hour limit, next simulation blocked

**Day 36: ICSBEP benchmark V&V suite**
- Load 5 pre-built ICSBEP cases: LEU-COMP-THERM-001, PU-MET-FAST-001, etc.
- Run MC simulations, compare k-eff to reference ± 2σ
- Automated regression testing via GitHub Actions
- Generate V&V report: C/E ratios, pass/fail status
- Test: All 5 benchmarks pass within 2σ

**Day 37: Regulatory document generation (Typst)**
- `src/services/document_generation.rs` — NRC FSAR Chapter 4 template in Typst
- Auto-populate tables: core parameters, reactivity coefficients, power distribution
- Generate figures: axial power profile, radial peaking map
- Export to PDF with traceability (simulation IDs linked)
- Test: Generate Chapter 4 for PWR reference design

**Day 38: Audit trail and NQA-1 compliance**
- Enable audit_log for all critical operations (simulation submit, approval, export)
- Immutable log with append-only constraint
- Provenance tracking: code version, nuclear data library, input parameters
- QA-level review workflow: submit → review → approve
- Test: Verify audit trail completeness for NRC-submittal project

### Phase 8 — Deployment + Launch (Days 39–42)

**Day 39: Kubernetes + GPU node pools**
- `k8s/gpu-worker-deployment.yaml` — DaemonSet for GPU workers with NVIDIA device plugin
- `k8s/api-deployment.yaml` — API server with HPA (3-10 replicas)
- `k8s/postgres-statefulset.yaml` — PostgreSQL with persistent volume
- `k8s/redis-deployment.yaml` — Redis for job queue
- Health checks: `/health/live`, `/health/ready`
- Test: Deploy to staging cluster, verify GPU job execution

**Day 40: Monitoring and alerting**
- Prometheus metrics: GPU utilization, simulation throughput, k-eff convergence rate, API latency
- Grafana dashboards: system health, user activity, simulation queue depth
- Sentry error tracking with source maps
- Alerts: GPU node down, simulation failure rate >10%, database connection pool exhaustion
- Test: Trigger alert conditions, verify PagerDuty notifications

**Day 41: Documentation and landing page**
- Documentation site: getting started, reactor types, simulation parameters, API reference
- Landing page with product overview, pricing, demo video (3D viewer flythrough)
- Academic tier signup flow with .edu email verification
- Blog post: "GPU-Accelerated Monte Carlo: 50x Faster Than MCNP"
- Test: End-to-end user journey from signup to first simulation

**Day 42: Production deployment and launch**
- Database backup and restore verification
- CDN setup for frontend assets and nuclear data libraries
- Rate limiting: 100 req/min per user, 10 simulations/hour
- Security audit: JWT validation, SQL injection (SQLx prevents), CORS, CSP headers
- Deploy to production, enable monitoring, soft launch to 10 beta users
- Collect feedback, iterate on UX issues, announce public launch

---

## Critical Files

```
reactorsim/
├── solver-gpu/                           # CUDA Monte Carlo solver
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── api.rs                        # High-level API
│   │   ├── cross_sections.rs             # ENDF data loader
│   │   ├── geometry.rs                   # CSG geometry
│   │   ├── tallies.rs                    # Flux, power tallies
│   │   ├── eigenvalue.rs                 # K-eff power iteration
│   │   └── doppler.rs                    # Temperature-dependent XS
│   ├── kernels/
│   │   ├── transport.cu                  # Particle tracking kernel
│   │   ├── geometry.cu                   # Boundary distance, cell finding
│   │   └── tallies.cu                    # Tally accumulation
│   └── tests/
│       ├── critical_sphere.rs            # Godiva benchmark
│       ├── pin_cell.rs                   # PWR pin eigenvalue
│       └── performance.rs                # GPU performance benchmarks
│
├── solver-sn/                            # Deterministic Sn solver
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── quadrature.rs                 # Angular quadrature
│   │   ├── sweep.rs                      # Transport sweep
│   │   ├── dsa.rs                        # Diffusion acceleration
│   │   └── eigenvalue.rs                 # Power iteration
│   └── tests/
│       └── slab_reactor.rs               # 1D benchmark
│
├── solver-th/                            # Thermal-hydraulics solver
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── subchannel.rs                 # Conservation equations
│   │   ├── steam_tables.rs               # IAPWS-IF97
│   │   ├── fuel_temp.rs                  # Radial heat conduction
│   │   └── chf.rs                        # CHF correlations
│   └── tests/
│       └── heated_channel.rs             # Single-channel validation
│
├── solver-depletion/                     # Burnup/depletion solver
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── cram.rs                       # CRAM matrix exponential
│   │   ├── predictor_corrector.rs        # CE/LI method
│   │   └── decay_heat.rs                 # ANS 5.1 standard
│   └── tests/
│       └── u235_burnup.rs                # Isotopic evolution
│
├── solver-coupled/                       # Coupled multi-physics
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── picard.rs                     # Fixed-point iteration
│   │   └── depletion.rs                  # Depletion-coupled MC
│   └── tests/
│       └── pwr_hot_full_power.rs         # Coupled N-TH test
│
├── reactorsim-api/                       # Rust API server
│   ├── Cargo.toml
│   ├── src/
│   │   ├── main.rs                       # Axum app, router
│   │   ├── config.rs                     # Environment config
│   │   ├── state.rs                      # AppState
│   │   ├── error.rs                      # ApiError
│   │   ├── auth/
│   │   │   ├── mod.rs                    # JWT middleware
│   │   │   └── saml.rs                   # SSO handlers
│   │   ├── api/
│   │   │   ├── router.rs                 # Routes
│   │   │   ├── handlers/
│   │   │   │   ├── auth.rs               # Auth endpoints
│   │   │   │   ├── users.rs              # User CRUD
│   │   │   │   ├── projects.rs           # Project management
│   │   │   │   ├── models.rs             # Reactor model CRUD
│   │   │   │   ├── simulation.rs         # Sim submission
│   │   │   │   ├── results.rs            # Results retrieval
│   │   │   │   ├── billing.rs            # Stripe integration
│   │   │   │   ├── benchmarks.rs         # V&V benchmarks
│   │   │   │   └── webhooks/
│   │   │   │       └── stripe.rs         # Stripe webhooks
│   │   │   └── ws/
│   │   │       └── simulation_progress.rs
│   │   ├── middleware/
│   │   │   └── plan_limits.rs            # Usage enforcement
│   │   ├── services/
│   │   │   ├── document_generation.rs    # Typst reports
│   │   │   └── s3.rs                     # S3 helpers
│   │   └── workers/
│   │       ├── mod.rs                    # Worker pool
│   │       └── gpu_monte_carlo_worker.rs
│   ├── migrations/
│   │   └── 001_initial.sql
│   └── tests/
│       ├── api_integration.rs
│       └── simulation_e2e.rs
│
├── nuclear-data-service/                 # Python XS processing
│   ├── requirements.txt
│   ├── main.py                           # FastAPI app
│   ├── endf_parser.py                    # ENDF reader
│   ├── doppler_broadening.py             # Sigma1 method
│   ├── multigroup_collapse.py            # Generate MG libraries
│   └── Dockerfile
│
├── frontend/                             # React frontend
│   ├── package.json
│   ├── vite.config.ts
│   ├── src/
│   │   ├── App.tsx
│   │   ├── main.tsx
│   │   ├── stores/
│   │   │   ├── authStore.ts
│   │   │   ├── reactorStore.ts
│   │   │   └── simulationStore.ts
│   │   ├── hooks/
│   │   │   └── useSimulationProgress.ts
│   │   ├── lib/
│   │   │   └── api.ts
│   │   ├── pages/
│   │   │   ├── Dashboard.tsx
│   │   │   ├── ReactorEditor.tsx
│   │   │   ├── CoreDesigner.tsx
│   │   │   ├── SimulationMonitor.tsx
│   │   │   ├── Results.tsx
│   │   │   ├── Benchmarks.tsx
│   │   │   ├── Billing.tsx
│   │   │   └── Login.tsx
│   │   ├── components/
│   │   │   ├── ReactorViewer/
│   │   │   │   ├── ReactorViewer.tsx     # Three.js scene
│   │   │   │   ├── FuelPinInstances.tsx
│   │   │   │   ├── ScalarFieldOverlay.tsx
│   │   │   │   └── CrossSectionPlane.tsx
│   │   │   ├── CoreDesigner/
│   │   │   │   ├── LoadingPatternEditor.tsx
│   │   │   │   ├── AssemblyPalette.tsx
│   │   │   │   └── SymmetryControls.tsx
│   │   │   └── common/
│   │   │       └── SimulationToolbar.tsx
│   │   └── data/
│   │       └── benchmarks/               # ICSBEP JSON models
│   └── public/
│
├── k8s/                                  # Kubernetes manifests
│   ├── api-deployment.yaml
│   ├── gpu-worker-daemonset.yaml
│   ├── postgres-statefulset.yaml
│   ├── redis-deployment.yaml
│   ├── nuclear-data-service-deployment.yaml
│   └── ingress.yaml
│
├── docker-compose.yml                    # Local dev stack
├── Cargo.toml                            # Workspace root
└── .github/
    └── workflows/
        ├── ci.yml                        # Test + lint
        ├── cuda-build.yml                # Build GPU kernels
        ├── benchmark-vv.yml              # Automated V&V
        └── deploy.yml                    # K8s deployment
```

---

## Solver Validation Suite

### Benchmark 1: Godiva Critical Sphere (Eigenvalue)

**System:** Bare sphere of HEU (93.71 w% U-235), radius = 8.741 cm, density = 18.74 g/cm³

**Expected:** k_eff = **1.0000** (critical)

**Reference:** ICSBEP HEU-MET-FAST-001

**Tolerance:** k_eff = 1.0000 ± 0.0030 (within 3σ statistical + systematic uncertainty)

### Benchmark 2: LEU-COMP-THERM-001 (Low-Enriched Thermal Lattice)

**System:** Square lattice of UO₂ fuel rods (4.31 w% U-235) in light water, pitch = 1.6 cm

**Expected:** k_eff = **1.00698 ± 0.00086**

**Reference:** ICSBEP LEU-COMP-THERM-001

**Tolerance:** |C/E - 1| < 0.003 (C/E ratio within 0.3%)

### Benchmark 3: PWR Pin Cell (Eigenvalue + Flux)

**System:** Single PWR fuel pin (3.1 w% U-235), cladding, water moderator

**Expected (k-inf):** **1.185** (infinite lattice)

**Expected (flux peak):** Thermal flux peak at fuel-moderator interface

**Tolerance:** k-inf ± 0.005, flux shape RMS < 2%

### Benchmark 4: Fuel Temperature Doppler Coefficient

**System:** UO₂ pin cell at T_fuel = 600K vs. 1200K, hold T_coolant constant

**Expected:** Δk/ΔT = **-2.5 pcm/K** (negative Doppler coefficient)

**Tolerance:** ± 0.5 pcm/K

### Benchmark 5: Coupled Neutronics-TH (PWR Hot Full Power)

**System:** 17×17 PWR assembly at 100% power, inlet T = 290°C, P = 15.5 MPa

**Expected:** k_eff (no feedback) = 1.20, k_eff (with feedback) = **1.05** (6-7% reactivity defect)

**Expected:** Max fuel centerline T < **1800 K**, DNBR > **1.3**

**Tolerance:** k_eff ± 0.01, T_fuel ± 50K, DNBR ± 0.1

---

## Verification

### Integration Testing Checklist

1. **Auth flow** — Register → login (email/SAML) → JWT issued → authorized API calls → refresh → logout
2. **Project CRUD** — Create → update reactor type → save → reload → verify model state preserved
3. **Reactor model editing** — Create CSG geometry → set materials → validate (no overlaps) → save version
4. **Core designer** — Place assemblies → enforce quarter-core symmetry → generate full core → export
5. **WASM MC simulation** — Small problem (100 particles) → WASM solver → k-eff returned → displayed
6. **GPU MC simulation** — Full-core (100M particles) → job queued → worker claims → progress via WebSocket → results in S3
7. **Sn solver** — 2D assembly → Sn eigenvalue → compare to MC reference → within 200 pcm
8. **Coupled N-TH** — PWR at power → Picard iteration → converges in <10 iterations → DNBR computed
9. **Depletion** — Single assembly → burnup to 50 GWd/MTU → isotopic inventory → Pu-239 buildup verified
10. **Benchmark V&V** — Run ICSBEP case → k-eff within 2σ of reference → pass/fail recorded
11. **Scalar field visualization** — Load power distribution → display in 3D viewer → color map accurate
12. **Cross-section cutting** — Enable axial plane → drag slider → geometry sliced correctly
13. **Plan limits** — Academic user → submit 100M particle job → blocked with upgrade prompt
14. **Billing** — Subscribe to Pro → Stripe checkout → webhook → plan updated → GPU-hour quota increased
15. **Regulatory document** — Generate FSAR Chapter 4 → verify tables/figures populated → PDF downloaded with traceability

### SQL Verification Queries

```sql
-- 1. Simulation throughput and success rate
SELECT
    solver,
    execution_mode,
    status,
    COUNT(*) as count,
    AVG(EXTRACT(EPOCH FROM (completed_at - started_at)))::int as avg_time_sec,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY EXTRACT(EPOCH FROM (completed_at - started_at)))::int as p95_time_sec
FROM simulations
WHERE submitted_at >= NOW() - INTERVAL '7 days'
GROUP BY solver, execution_mode, status;

-- 2. Simulation type distribution
SELECT sim_type, solver, COUNT(*) as count,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 1) as pct
FROM simulations
WHERE submitted_at >= NOW() - INTERVAL '30 days'
GROUP BY sim_type, solver
ORDER BY count DESC;

-- 3. User plan distribution and GPU usage
SELECT plan, COUNT(*) as users,
    ROUND(AVG(
        SELECT SUM(quantity) FROM usage_records
        WHERE user_id = users.id AND record_type = 'gpu_hours'
        AND period_start <= CURRENT_DATE AND period_end >= CURRENT_DATE
    ), 2) as avg_gpu_hours_this_period
FROM users
GROUP BY plan
ORDER BY users DESC;

-- 4. GPU hours consumed by user (billing period)
SELECT u.email, u.plan,
    SUM(ur.quantity) as total_gpu_hours,
    CASE u.plan
        WHEN 'academic' THEN 10
        WHEN 'pro' THEN 100
        WHEN 'team' THEN 500
        WHEN 'enterprise' THEN 99999
    END as limit_gpu_hours,
    ROUND(100.0 * SUM(ur.quantity) / CASE u.plan
        WHEN 'academic' THEN 10
        WHEN 'pro' THEN 100
        WHEN 'team' THEN 500
        WHEN 'enterprise' THEN 99999
    END, 1) as usage_pct
FROM usage_records ur
JOIN users u ON ur.user_id = u.id
WHERE ur.period_start <= CURRENT_DATE AND ur.period_end >= CURRENT_DATE
    AND ur.record_type = 'gpu_hours'
GROUP BY u.email, u.plan
ORDER BY total_gpu_hours DESC;

-- 5. V&V benchmark pass rate
SELECT bc.benchmark_suite,
    COUNT(*) as total_runs,
    SUM(CASE WHEN vv.pass_fail THEN 1 ELSE 0 END) as passed,
    ROUND(100.0 * SUM(CASE WHEN vv.pass_fail THEN 1 ELSE 0 END) / COUNT(*), 1) as pass_rate_pct,
    AVG(vv.ce_ratio) as avg_ce_ratio,
    STDDEV(vv.ce_ratio) as std_ce_ratio
FROM vv_results vv
JOIN benchmark_cases bc ON vv.benchmark_id = bc.id
WHERE vv.created_at >= NOW() - INTERVAL '30 days'
GROUP BY bc.benchmark_suite
ORDER BY pass_rate_pct DESC;

-- 6. Most active projects
SELECT p.name, p.reactor_type,
    COUNT(DISTINCT s.id) as num_simulations,
    SUM(s.gpu_hours_used) as total_gpu_hours,
    MAX(s.submitted_at) as last_simulation,
    COUNT(DISTINCT rm.id) as num_model_versions
FROM projects p
LEFT JOIN simulations s ON p.id = s.project_id
LEFT JOIN reactor_models rm ON p.id = rm.project_id
WHERE p.created_at >= NOW() - INTERVAL '90 days'
GROUP BY p.id, p.name, p.reactor_type
ORDER BY num_simulations DESC
LIMIT 20;
```

### Performance Benchmarks

| Operation | Target | Method |
|-----------|--------|--------|
| WASM MC: 100-particle eigenvalue | <5s | Browser benchmark (Chrome DevTools) |
| WASM MC: 1000-particle eigenvalue | <30s | Browser benchmark |
| GPU MC: 1M particles (pin cell) | <10s | A100 single GPU, CUDA timing |
| GPU MC: 100M particles (PWR assembly) | <5 min | A100 single GPU |
| GPU MC: 1B particles (full core) | <30 min | 4x A100 GPUs with NCCL |
| Sn: 2D assembly (S8, 47 groups) | <20s | CPU 8-core, Rust native |
| Sn: 3D core (S8, 7 groups) | <2 min | GPU-accelerated sweep |
| Coupled N-TH: PWR assembly | <10 iterations | Picard convergence, 5 min total |
| Depletion: 3-step burnup | <20 min | MC + CRAM depletion |
| 3D Viewer: 50K fuel pins render | 60 FPS | Three.js instanced mesh, Chrome |
| 3D Viewer: Scalar field update | <100ms | GPU buffer upload, color map shader |
| API: create simulation | <200ms | p95 latency (Prometheus) |
| API: get results (10MB HDF5) | <500ms | S3 presigned URL generation |
| WebSocket: progress latency | <50ms | Time from worker emit to client receive |
| ENDF XS lookup (GPU) | <100 cycles | Binary search + interpolation per particle |

---

## Deployment Architecture

### Infrastructure

```
┌──────────────────────────────────────────────────┐
│                CloudFront CDN                     │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────┐│
│  │ Frontend SPA │  │ WASM MC      │  │ Nuclear ││
│  │ (React build)│  │ Solver       │  │ Data    ││
│  └──────────────┘  └──────────────┘  └─────────┘│
└───────────────────────┬──────────────────────────┘
                        │
            ┌───────────┴───────────┐
            │   AWS ALB (HTTPS)     │
            │   TLS termination     │
            └───────────┬───────────┘
                        │
      ┌─────────────────┼─────────────────┐
      ▼                 ▼                 ▼
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│ API Server  │  │ API Server  │  │ API Server  │
│ (Rust/Axum) │  │ (Rust/Axum) │  │ (Rust/Axum) │
│ Pod ×3 HPA  │  │             │  │             │
└──────┬──────┘  └──────┬──────┘  └──────┬──────┘
       │                │                │
       └────────────────┼────────────────┘
                        │
       ┌────────────────┼────────────────┐
       ▼                ▼                ▼
┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│ PostgreSQL  │ │ Redis       │ │ S3 Bucket   │
│ (RDS r6g)   │ │(ElastiCache)│ │ (Results +  │
│ Multi-AZ    │ │ Cluster     │ │  XS Data)   │
└─────────────┘ └──────┬──────┘ └─────────────┘
                       │
       ┌───────────────┼───────────────┐
       ▼               ▼               ▼
┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│ GPU Worker  │ │ GPU Worker  │ │ GPU Worker  │
│ CUDA MC     │ │ CUDA MC     │ │ CUDA MC     │
│ p3.8xlarge  │ │ p3.8xlarge  │ │ p3.8xlarge  │
│ 4×V100 GPU  │ │ 4×V100 GPU  │ │ 4×V100 GPU  │
│ HPA: 1-10   │ │             │ │             │
└─────────────┘ └─────────────┘ └─────────────┘
       │               │               │
       └───────────────┴───────────────┘
                       │
                       ▼
               ┌─────────────┐
               │ Nuclear Data│
               │ Service     │
               │(FastAPI/Py) │
               │ Pod ×2      │
               └─────────────┘
```

### Scaling Strategy

- **API servers**: Horizontal pod autoscaler (HPA) at 70% CPU, min 3, max 10
- **GPU workers**: HPA based on Redis queue depth — scale up when queue >3 jobs, scale down when idle 10 min
- **Worker instances**: AWS p3.8xlarge (4×V100 GPUs, 32 vCPU, 244GB RAM) or p4d.24xlarge (8×A100 GPUs) for production
- **Database**: RDS PostgreSQL r6g.2xlarge with read replica for analytics queries
- **Redis**: ElastiCache r6g.xlarge cluster with 1 primary + 2 replicas for HA
- **S3 storage**: Standard tier for active results (30 days), transition to S3-IA (90 days), Glacier (365 days), expire after 2 years

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
        { "Days": 90, "StorageClass": "GLACIER_IR" },
        { "Days": 365, "StorageClass": "DEEP_ARCHIVE" }
      ],
      "Expiration": { "Days": 730 }
    },
    {
      "ID": "nuclear-data-libraries",
      "Prefix": "nuclear-data/",
      "Status": "Enabled",
      "Transitions": [
        { "Days": 180, "StorageClass": "STANDARD_IA" }
      ]
    },
    {
      "ID": "wasm-solvers",
      "Prefix": "wasm/",
      "Status": "Enabled",
      "NoncurrentVersionExpiration": { "NoncurrentDays": 30 }
    },
    {
      "ID": "regulatory-documents",
      "Prefix": "documents/",
      "Status": "Enabled"
    }
  ]
}
```

### GPU Cost Optimization

1. **Spot instances for non-critical jobs**: Academic tier and non-urgent Pro simulations run on spot instances (60-70% cost savings)
2. **GPU sharing**: Multiple small MC jobs (1M-10M particles) scheduled on same GPU via CUDA streams
3. **Auto-scaling policies**: Scale down to 1 GPU worker during low-usage hours (midnight-6am UTC)
4. **Reserved capacity**: Purchase 1-year reserved instances for 3 baseline GPU workers (40% discount)
5. **Regional optimization**: Run workers in us-east-1 (cheapest GPU regions), replicate to eu-west-1 for European customers

---

## Post-MVP Roadmap

| Feature | Description | Priority |
|---------|-------------|----------|
| **Multi-group Monte Carlo** | Support 47-group and 252-group MC for faster variance reduction compared to continuous-energy, with automatic group importance sampling | High |
| **Hexagonal geometry (VVER, SFR)** | Hexagonal lattice support for VVER-1000 and sodium-cooled fast reactors with 120° rotational symmetry enforcement | High |
| **Two-phase thermal-hydraulics** | Void fraction tracking, drift-flux models, and CHF/DNBR calculation for BWR and accident scenarios with critical heat flux correlations | High |
| **Reactor kinetics and transients** | Point kinetics and spatial kinetics (IQS method) for control rod ejection, LOCA, and other Chapter 15 transient analyses | Medium |
| **CAD geometry import** | STEP/IGES file import with automatic conversion to CSG or unstructured mesh via OpenCascade for complex SMR geometries | Medium |
| **Variance reduction (CADIS)** | Consistent adjoint-driven importance sampling for deep-penetration shielding calculations with automatic weight window generation from adjoint Sn | Medium |
| **Monte Carlo multi-GPU scaling** | Domain decomposition across 8-16 GPUs with fission bank migration via NCCL for 10B+ particle full-core calculations | Medium |
| **Equilibrium cycle search** | Automated iteration to find equilibrium fuel loading pattern and burnup distribution with shuffling optimization | Medium |
| **Sensitivity and uncertainty** | Adjoint-based sensitivity coefficients and total Monte Carlo for comprehensive uncertainty budgets propagated from nuclear data covariances | Low |
| **Photon transport** | Coupled neutron-photon transport for dose-rate calculations and gamma heating in shielding and ex-core structures | Low |
| **Real-time collaboration** | CRDT-based reactor model co-editing with presence indicators, shared cursor positions, and comment threads | Low |
| **Custom nuclear data libraries** | Support for proprietary vendor evaluations and user-uploaded ENDF files with automated validation and QA workflows | Low |

---
