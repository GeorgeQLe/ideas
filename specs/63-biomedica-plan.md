# 63. BioMedica — Biomedical Device Simulation and Regulatory Platform

## Implementation Plan

**MVP Scope:** Cloud-based biomedical device simulation platform with STEP/STL device geometry import, anatomical geometry library (coronary artery, aorta, femur, tibia, spine segments from population-average CT/MRI data), structural FEA solver with hyperelastic tissue material models (Neo-Hookean, Mooney-Rivlin, Holzapfel-Gasser-Ogden for arterial wall), contact analysis for implant-bone and stent-artery interfaces with friction coefficients (μ=0.3-0.5), physiological loading conditions (gait cycle for hip/knee via ISO 14242-1, cardiac pressure waveform for cardiovascular devices), steady-state Newtonian blood flow CFD with wall shear stress (WSS) mapping, automatic hexahedral/tetrahedral mesh generation with boundary layer refinement, fatigue life prediction for CoCr and Ti-6Al-4V alloys via S-N curves (ASTM F1537/F136), Goodman diagrams with mean stress correction, automated regulatory report generation following FDA Guidance "Reporting of Computational Modeling Studies" with model description, mesh convergence study (3 refinement levels), boundary condition documentation, and results summary sections exported as PDF, Three.js 3D visualization with stress/strain contour plots and flow streamlines, PostgreSQL storage for projects/materials/anatomies, S3 for mesh files and simulation results, Stripe billing with Academic ($99/mo) / Pro ($349/mo) / Enterprise tiers.

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| FEA Solver | Rust (custom nonlinear FEA) | Hyperelastic material models, Newton-Raphson, sparse direct solver via MUMPS |
| CFD Solver | Rust (FVM, SIMPLE algorithm) | Incompressible Navier-Stokes, Newtonian/non-Newtonian fluids |
| Fatigue Analysis | Rust | S-N curves, Goodman diagrams, Haigh diagrams, Coffin-Manson |
| Mesh Generation | Rust + TetGen/Netgen bindings | Boundary layer mesh for CFD, contact surface refinement for FEA |
| Medical Imaging | Python 3.12 (FastAPI) | SimpleITK, VTK for DICOM segmentation (post-MVP) |
| Report Generation | Python (FastAPI) | ReportLab for PDF generation with V&V documentation |
| Database | PostgreSQL 16 | Projects, users, materials database, anatomical library metadata |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | Mesh files (.vtk, .msh), simulation results (VTU), DICOM images (post-MVP) |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| 3D Visualization | Three.js + React Three Fiber | Mesh rendering, stress contours, deformation animation, flow streamlines |
| Job Queue | Redis 7 + Tokio tasks | Simulation job management, parametric sweep distribution |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | Frontend assets, anatomical geometry library delivery |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, solver convergence tracking, error reporting |
| CI/CD | GitHub Actions | Rust tests, Python tests, Docker image build and push |

### Key Architecture Decisions

1. **Custom Rust FEA/CFD solvers instead of wrapping commercial codes**: Building custom solvers in Rust gives full control over biomedical-specific constitutive models (hyperelastic Holzapfel-Gasser-Ogden for arterial wall, anisotropic bone), convergence algorithms for high-deformation contact (stent-artery crimping/expansion), and tight integration with regulatory reporting (traceability from solver state to FDA documentation). Commercial solver APIs (ANSYS Mechanical APDL, Abaqus Python API) lack biomedical material libraries and produce opaque result files incompatible with automated V&V. The MVP FEA solver implements total Lagrangian formulation for large deformation, penalty method for contact, and MUMPS sparse direct solver via FFI bindings.

2. **Anatomical geometry library instead of patient-specific workflows for MVP**: MVP provides population-average anatomies (coronary artery tree, aorta, femur, tibia, L3-L5 spine segment) derived from literature CT/MRI datasets and CAD reconstruction. This covers 80% of pre-clinical device evaluation use cases (stent design, implant sizing, surgical tool clearance) without requiring HIPAA compliance infrastructure or DICOM processing. Post-MVP adds patient-specific modeling via Python FastAPI service calling SimpleITK for segmentation and VTK for surface extraction. Anatomies are stored as compressed VTK PolyData in S3 with PostgreSQL metadata (body region, source, mesh quality metrics).

3. **Physiological loading via parametric boundary conditions instead of motion capture**: MVP implements standard physiological loading protocols from ISO/ASTM test standards: gait cycle for hip/knee (ISO 14242-1 Paul Load Profile: peak force 3000N, frequency 1Hz), cardiac pressure waveform (120/80 mmHg pulsatile with 70 bpm), spinal compression (ASTM F2077: 2000N monotonic + 10M cycles at ±600N). This allows regulatory comparison with bench test data without requiring expensive motion capture equipment. Loading profiles stored as JSON time-series in PostgreSQL with frontend graphical editor for custom waveforms.

4. **Fatigue life via S-N and strain-life approaches for medical alloys**: MVP implements Basquin's Law (high-cycle fatigue) and Coffin-Manson (low-cycle fatigue) with mean stress correction via Goodman/Gerber/Soderberg diagrams. Material database includes S-N curves for CoCr alloy (ASTM F1537), Ti-6Al-4V (ASTM F136), 316L stainless steel (ASTM F138), and PEEK. Post-MVP adds nitinol pseudoelastic fatigue (ASTM F2477 strain-based approach) and corrosion fatigue in simulated body fluid (degradation of S-N curve via Paris Law crack growth). Fatigue solver extracts stress history from FEA transient or cyclic analysis, applies rainflow counting for cycle extraction, and computes damage via Miner's Rule.

5. **Automated FDA regulatory report generation instead of manual documentation**: MVP generates PDF report following FDA Guidance "Reporting of Computational Modeling Studies in Medical Device Submissions" with sections: (1) Model Description (geometry, material models, constitutive equations), (2) Boundary Conditions (loading, constraints with diagrams), (3) Mesh Convergence Study (3-level refinement with element count and peak stress convergence table), (4) Material Properties (literature references), (5) Results (contour plots, time-series), (6) Validation Evidence (comparison to bench test data if provided). Report generation service in Python FastAPI with ReportLab, receives structured JSON from Rust solver with all metadata, generates PDF with embedded plots. Traceability matrix links design inputs to analysis outputs.

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";  -- For fuzzy text search on materials/anatomies

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

-- Organizations (for Enterprise plan)
CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    slug TEXT UNIQUE NOT NULL,
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    plan TEXT NOT NULL DEFAULT 'enterprise',
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    settings JSONB DEFAULT '{}',  -- HIPAA compliance settings, IP restrictions
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
    role TEXT NOT NULL DEFAULT 'member',  -- owner | admin | engineer | viewer
    joined_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(org_id, user_id)
);
CREATE INDEX org_members_org_idx ON org_members(org_id);
CREATE INDEX org_members_user_idx ON org_members(user_id);

-- Projects (simulation workspace)
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    device_type TEXT NOT NULL,  -- cardiovascular_stent | orthopedic_implant | surgical_tool | neurostimulator | other
    device_geometry_url TEXT,  -- S3 URL to STEP/STL file
    anatomy_id UUID,  -- Reference to anatomies table
    custom_anatomy_url TEXT,  -- S3 URL to custom anatomy (post-MVP patient-specific)
    simulation_config JSONB NOT NULL DEFAULT '{}',  -- Solver settings, BC definitions, material assignments
    is_public BOOLEAN NOT NULL DEFAULT false,
    forked_from UUID REFERENCES projects(id) ON DELETE SET NULL,
    thumbnail_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);
CREATE INDEX projects_org_idx ON projects(org_id);
CREATE INDEX projects_device_type_idx ON projects(device_type);
CREATE INDEX projects_updated_idx ON projects(updated_at DESC);

-- Anatomical Geometries
CREATE TABLE anatomies (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,  -- e.g., "Left Anterior Descending Coronary Artery"
    body_region TEXT NOT NULL,  -- cardiovascular | orthopedic | neurological | spinal | custom
    anatomy_type TEXT NOT NULL,  -- artery | bone | organ | nerve | vessel_tree
    description TEXT,
    mesh_url TEXT NOT NULL,  -- S3 URL to VTK PolyData (.vtp)
    source_reference TEXT,  -- Literature citation or dataset reference
    mesh_quality JSONB,  -- {element_count, min_angle, max_aspect_ratio}
    material_regions JSONB,  -- Map of region ID to material type for heterogeneous structures
    is_verified BOOLEAN DEFAULT false,
    is_builtin BOOLEAN DEFAULT true,
    uploaded_by UUID REFERENCES users(id),
    download_count INTEGER DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX anatomies_body_region_idx ON anatomies(body_region);
CREATE INDEX anatomies_type_idx ON anatomies(anatomy_type);
CREATE INDEX anatomies_name_trgm_idx ON anatomies USING gin(name gin_trgm_ops);

-- Material Properties
CREATE TABLE materials (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,  -- e.g., "CoCr L605 Alloy" or "Arterial Wall (Holzapfel)"
    category TEXT NOT NULL,  -- metal | polymer | ceramic | biological_tissue | shape_memory_alloy
    material_model TEXT NOT NULL,  -- linear_elastic | neo_hookean | mooney_rivlin | holzapfel | ogden | plastic | viscoelastic
    properties JSONB NOT NULL,  -- All material parameters (E, nu, C10, C01, k1, k2, etc.)
    standard_reference TEXT,  -- ASTM/ISO standard or literature reference
    fatigue_data JSONB,  -- S-N curve data points [{cycles, stress_amplitude}]
    description TEXT,
    tags TEXT[] DEFAULT '{}',
    is_verified BOOLEAN DEFAULT false,
    is_builtin BOOLEAN DEFAULT true,
    uploaded_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX materials_category_idx ON materials(category);
CREATE INDEX materials_model_idx ON materials(material_model);
CREATE INDEX materials_name_trgm_idx ON materials USING gin(name gin_trgm_ops);

-- Simulation Runs
CREATE TABLE simulations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    analysis_type TEXT NOT NULL,  -- structural_static | structural_transient | cfd_steady | cfd_transient | fatigue
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | meshing | running | completed | failed | cancelled
    mesh_url TEXT,  -- S3 URL to generated mesh (.msh or .vtk)
    mesh_stats JSONB,  -- {element_count, node_count, min_quality, mesh_time_s}
    parameters JSONB NOT NULL DEFAULT '{}',  -- Analysis-specific parameters (timesteps, load curves, etc.)
    material_assignments JSONB NOT NULL DEFAULT '{}',  -- Map of geometry region to material ID
    boundary_conditions JSONB NOT NULL DEFAULT '{}',  -- Load, constraint, pressure, velocity definitions
    results_url TEXT,  -- S3 URL to VTU result file
    results_summary JSONB,  -- Quick-access key results (peak stress, max displacement, flow rate)
    error_message TEXT,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    wall_time_s INTEGER,
    compute_cost_usd REAL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX sims_project_idx ON simulations(project_id);
CREATE INDEX sims_user_idx ON simulations(user_id);
CREATE INDEX sims_status_idx ON simulations(status);
CREATE INDEX sims_created_idx ON simulations(created_at DESC);

-- Simulation Jobs (for async execution)
CREATE TABLE simulation_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    simulation_id UUID NOT NULL REFERENCES simulations(id) ON DELETE CASCADE,
    worker_id TEXT,
    cores_allocated INTEGER DEFAULT 4,
    memory_gb INTEGER DEFAULT 16,
    priority INTEGER DEFAULT 0,  -- Higher = more urgent (Enterprise gets higher priority)
    progress_pct REAL DEFAULT 0.0,
    current_phase TEXT,  -- meshing | assembly | solving | post_processing
    convergence_data JSONB DEFAULT '[]',  -- [{iteration, residual, timestep}]
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX jobs_sim_idx ON simulation_jobs(simulation_id);
CREATE INDEX jobs_worker_idx ON simulation_jobs(worker_id);
CREATE INDEX jobs_status_idx ON simulation_jobs(priority DESC, created_at);

-- Regulatory Reports
CREATE TABLE regulatory_reports (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    simulation_id UUID NOT NULL REFERENCES simulations(id) ON DELETE CASCADE,
    report_type TEXT NOT NULL DEFAULT 'fda_computational_modeling',
    sections JSONB NOT NULL,  -- Structured report content following FDA Guidance
    pdf_url TEXT,  -- S3 URL to generated PDF
    validation_data JSONB,  -- Bench test comparison data for validation section
    mesh_convergence JSONB,  -- Table of mesh refinement study results
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX reports_sim_idx ON regulatory_reports(simulation_id);

-- Physiological Loading Profiles
CREATE TABLE loading_profiles (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    profile_type TEXT NOT NULL,  -- gait_cycle | cardiac_pressure | spinal_compression | custom
    standard_reference TEXT,  -- ISO 14242-1, ASTM F2077, etc.
    time_series JSONB NOT NULL,  -- [{time_s, force_N}, ...] or [{time_s, pressure_Pa}, ...]
    description TEXT,
    is_builtin BOOLEAN DEFAULT true,
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX loading_profile_type_idx ON loading_profiles(profile_type);

-- Usage Tracking (for billing)
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,  -- compute_hours | storage_gb_month | simulation_count
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
    pub device_type: String,
    pub device_geometry_url: Option<String>,
    pub anatomy_id: Option<Uuid>,
    pub custom_anatomy_url: Option<String>,
    pub simulation_config: serde_json::Value,
    pub is_public: bool,
    pub forked_from: Option<Uuid>,
    pub thumbnail_url: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Anatomy {
    pub id: Uuid,
    pub name: String,
    pub body_region: String,
    pub anatomy_type: String,
    pub description: Option<String>,
    pub mesh_url: String,
    pub source_reference: Option<String>,
    pub mesh_quality: Option<serde_json::Value>,
    pub material_regions: Option<serde_json::Value>,
    pub is_verified: bool,
    pub is_builtin: bool,
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
    pub material_model: String,
    pub properties: serde_json::Value,
    pub standard_reference: Option<String>,
    pub fatigue_data: Option<serde_json::Value>,
    pub description: Option<String>,
    pub tags: Vec<String>,
    pub is_verified: bool,
    pub is_builtin: bool,
    pub uploaded_by: Option<Uuid>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Simulation {
    pub id: Uuid,
    pub project_id: Uuid,
    pub user_id: Uuid,
    pub analysis_type: String,
    pub status: String,
    pub mesh_url: Option<String>,
    pub mesh_stats: Option<serde_json::Value>,
    pub parameters: serde_json::Value,
    pub material_assignments: serde_json::Value,
    pub boundary_conditions: serde_json::Value,
    pub results_url: Option<String>,
    pub results_summary: Option<serde_json::Value>,
    pub error_message: Option<String>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub wall_time_s: Option<i32>,
    pub compute_cost_usd: Option<f32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct SimulationJob {
    pub id: Uuid,
    pub simulation_id: Uuid,
    pub worker_id: Option<String>,
    pub cores_allocated: i32,
    pub memory_gb: i32,
    pub priority: i32,
    pub progress_pct: f32,
    pub current_phase: Option<String>,
    pub convergence_data: serde_json::Value,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct SimulationParams {
    pub analysis_type: AnalysisType,
    pub structural: Option<StructuralParams>,
    pub cfd: Option<CfdParams>,
    pub fatigue: Option<FatigueParams>,
    pub solver_options: SolverOptions,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum AnalysisType {
    StructuralStatic,
    StructuralTransient,
    CfdSteady,
    CfdTransient,
    Fatigue,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct StructuralParams {
    pub total_time: Option<f64>,  // For transient
    pub timestep: Option<f64>,
    pub loading_profile_id: Option<Uuid>,
    pub custom_loads: Vec<LoadDefinition>,
    pub contact_pairs: Vec<ContactPair>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct LoadDefinition {
    pub load_type: String,  // force | pressure | displacement | temperature
    pub region_id: String,  // Geometry region/surface name
    pub magnitude: f64,
    pub direction: Option<[f64; 3]>,  // For force/displacement
    pub time_variation: Option<String>,  // constant | sinusoidal | ramp | from_profile
}

#[derive(Debug, Deserialize, Serialize)]
pub struct ContactPair {
    pub master_surface: String,
    pub slave_surface: String,
    pub friction_coefficient: f64,
    pub contact_stiffness: Option<f64>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct CfdParams {
    pub fluid_type: String,  // newtonian | carreau_yasuda | power_law
    pub inlet_velocity: f64,  // m/s
    pub outlet_pressure: f64,  // Pa
    pub wall_boundary: String,  // no_slip | slip
    pub turbulence_model: Option<String>,  // laminar | k_epsilon | k_omega_sst
    pub transient: Option<TransientCfdParams>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct TransientCfdParams {
    pub total_time: f64,  // seconds
    pub timestep: f64,
    pub inlet_waveform: Option<Vec<(f64, f64)>>,  // (time, velocity) pairs
}

#[derive(Debug, Deserialize, Serialize)]
pub struct FatigueParams {
    pub method: String,  // s_n | strain_life | fracture_mechanics
    pub load_history: Vec<f64>,  // Stress amplitude time series
    pub mean_stress: f64,
    pub mean_stress_correction: String,  // goodman | gerber | soderberg | none
    pub surface_finish_factor: f64,  // Default 1.0
    pub size_factor: f64,  // Default 1.0
}

#[derive(Debug, Deserialize, Serialize)]
pub struct SolverOptions {
    pub max_iterations: u32,  // Default 50 for nonlinear
    pub convergence_tol: f64,  // Default 1e-6
    pub element_formulation: String,  // tet4 | tet10 | hex8 | hex20
    pub mesh_refinement_level: u8,  // 1=coarse, 2=medium, 3=fine
}
```

---

## Solver Architecture Deep-Dive

### Governing Equations and Discretization

BioMedica's FEA solver implements **nonlinear structural mechanics** with large deformation (geometric nonlinearity) and hyperelastic material models (material nonlinearity). The governing weak form is:

```
Find u ∈ V such that:
∫_Ω P : ∇_X(δu) dΩ - ∫_Ω ρ₀ b · δu dΩ - ∫_Γ t · δu dΓ = 0  ∀ δu ∈ V₀

where:
  P = First Piola-Kirchhoff stress tensor (function of F = I + ∇u)
  u = Displacement field
  ρ₀ = Reference density
  b = Body force per unit mass
  t = Traction on Neumann boundary Γ
  Ω = Reference (undeformed) configuration
```

**Hyperelastic Material Models** (Neo-Hookean, Mooney-Rivlin, Holzapfel-Gasser-Ogden):

The strain energy density Ψ defines the stress response. For **Neo-Hookean** (isotropic):
```
Ψ = (μ/2)(I₁ - 3) + (κ/2)(J - 1)²

where:
  I₁ = tr(C) = first invariant of right Cauchy-Green tensor C = F^T F
  J = det(F) = volume ratio
  μ = shear modulus, κ = bulk modulus
```

For **Holzapfel-Gasser-Ogden** (anisotropic arterial wall with fiber families):
```
Ψ = (μ/2)(I₁ - 3) + (k₁/2k₂) ∑ᵢ[exp(k₂(I₄ᵢ - 1)²) - 1]

where:
  I₄ᵢ = a₀ᵢ · C · a₀ᵢ = stretch in fiber direction a₀ᵢ
  k₁, k₂ = fiber stiffness parameters
```

Second Piola-Kirchhoff stress: **S = 2 ∂Ψ/∂C**. First Piola-Kirchhoff: **P = F·S**.

**Finite Element Discretization** uses isoparametric tetrahedral (Tet4, Tet10) or hexahedral (Hex8, Hex20) elements:
```
u(X) ≈ ∑ᵢ Nᵢ(ξ) uᵢ

where Nᵢ are shape functions in natural coordinates ξ
```

Discrete system (Newton-Raphson):
```
K_T(uⁿ) Δu = -R(uⁿ)

where:
  K_T = Tangent stiffness matrix (material + geometric contributions)
  R = Residual force vector (internal - external)
  Δu = Displacement increment
```

**Contact Modeling** uses penalty method:
```
F_contact = ε_n · g_n  if g_n < 0  (penetration)
F_friction = μ · |F_contact| · v_tangential/|v_tangential|

where:
  g_n = gap function (negative when penetrated)
  ε_n = normal penalty stiffness
  μ = friction coefficient
```

**CFD Solver** implements incompressible Navier-Stokes with SIMPLE (Semi-Implicit Method for Pressure-Linked Equations):
```
ρ(∂u/∂t + u·∇u) = -∇p + μ∇²u + f  (momentum)
∇·u = 0  (continuity)

Non-Newtonian (Carreau-Yasuda):
μ = μ_∞ + (μ₀ - μ_∞)[1 + (λγ̇)ᵃ]^((n-1)/a)

where γ̇ = √(2 tr(D²)) is shear rate, D = (∇u + ∇uᵀ)/2
```

Finite volume discretization with collocated grid, Rhie-Chow interpolation for pressure-velocity coupling.

### Fatigue Life Prediction

**High-Cycle Fatigue (S-N Approach)** — Basquin's Law:
```
σ_a = σ'_f (2N_f)^b

where:
  σ_a = stress amplitude
  N_f = cycles to failure
  σ'_f = fatigue strength coefficient
  b = fatigue strength exponent (typically -0.1 to -0.15)
```

**Mean Stress Correction** — Goodman diagram:
```
σ_a/σ_f + σ_m/σ_u = 1

where:
  σ_m = mean stress
  σ_u = ultimate tensile strength
```

**Damage Accumulation** — Miner's Rule:
```
D = ∑ᵢ nᵢ/Nᵢ  (failure when D ≥ 1)

where nᵢ cycles at stress amplitude σᵢ, Nᵢ from S-N curve
```

### Mesh Generation Pipeline

```
Device STL/STEP → Boolean union with anatomy → TetGen/Netgen
  ↓
Boundary layer refinement for CFD (5 layers, growth ratio 1.2)
Contact surface refinement for FEA (element size 0.1mm at interface)
  ↓
Quality checks: min dihedral angle > 10°, max aspect ratio < 10
  ↓
VTK unstructured grid (.vtu) → S3 storage
```

TetGen command-line invocation via Rust FFI:
```
tetrahedralize -pq1.2a0.5 input.poly output.1
  -p: PLC input
  -q: quality ratio bound 1.2 (default 2.0)
  -a: max tetrahedron volume 0.5 mm³
```

---

## Architecture Deep-Dives

### 1. Simulation API Handler (Rust/Axum)

Receives simulation request, validates geometry/materials, generates mesh, enqueues solver job, and returns simulation ID for progress tracking.

```rust
// src/api/handlers/simulation.rs

use axum::{
    extract::{Path, State, Json},
    http::StatusCode,
    response::IntoResponse,
};
use uuid::Uuid;
use crate::{
    db::models::{Simulation, SimulationParams, AnalysisType},
    mesh::generator::MeshGenerator,
    state::AppState,
    auth::Claims,
    error::ApiError,
};

#[derive(serde::Deserialize)]
pub struct CreateSimulationRequest {
    pub analysis_type: AnalysisType,
    pub parameters: serde_json::Value,
    pub material_assignments: serde_json::Value,
    pub boundary_conditions: serde_json::Value,
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

    // 2. Check plan limits
    let user = sqlx::query_as!(
        crate::db::models::User,
        "SELECT * FROM users WHERE id = $1",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    // Free plan limited to 50K elements
    let params: SimulationParams = serde_json::from_value(req.parameters.clone())?;
    let max_elements = match user.plan.as_str() {
        "free" => 50_000,
        "academic" => 500_000,
        "pro" => 2_000_000,
        _ => 10_000_000, // Enterprise
    };

    // 3. Create simulation record (status = pending)
    let sim = sqlx::query_as!(
        Simulation,
        r#"INSERT INTO simulations
            (project_id, user_id, analysis_type, status, parameters,
             material_assignments, boundary_conditions)
        VALUES ($1, $2, $3, $4, $5, $6, $7)
        RETURNING *"#,
        project_id,
        claims.user_id,
        serde_json::to_string(&req.analysis_type)?,
        "pending",
        req.parameters,
        req.material_assignments,
        req.boundary_conditions,
    )
    .fetch_one(&state.db)
    .await?;

    // 4. Enqueue meshing + simulation job
    let job = sqlx::query_as!(
        crate::db::models::SimulationJob,
        r#"INSERT INTO simulation_jobs
           (simulation_id, priority, cores_allocated, memory_gb)
        VALUES ($1, $2, $3, $4)
        RETURNING *"#,
        sim.id,
        if user.plan == "enterprise" { 10 } else { 0 },
        match req.analysis_type {
            AnalysisType::StructuralStatic => 4,
            AnalysisType::StructuralTransient => 8,
            AnalysisType::CfdSteady => 8,
            AnalysisType::CfdTransient => 16,
            AnalysisType::Fatigue => 4,
        },
        match req.analysis_type {
            AnalysisType::StructuralStatic => 16,
            AnalysisType::StructuralTransient => 32,
            AnalysisType::CfdSteady => 32,
            AnalysisType::CfdTransient => 64,
            AnalysisType::Fatigue => 16,
        }
    )
    .fetch_one(&state.db)
    .await?;

    // Publish to Redis job queue
    state.redis
        .lpush("simulation:jobs", serde_json::to_string(&job.id)?)
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

pub async fn cancel_simulation(
    State(state): State<AppState>,
    claims: Claims,
    Path((project_id, sim_id)): Path<(Uuid, Uuid)>,
) -> Result<impl IntoResponse, ApiError> {
    // Verify ownership
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

    if sim.status != "pending" && sim.status != "running" && sim.status != "meshing" {
        return Err(ApiError::BadRequest("Simulation is not cancellable"));
    }

    // Update status to cancelled
    sqlx::query!(
        "UPDATE simulations SET status = 'cancelled', completed_at = NOW() WHERE id = $1",
        sim.id
    )
    .execute(&state.db)
    .await?;

    // Signal worker to cancel (via Redis pub/sub)
    state.redis
        .publish("simulation:cancel", sim.id.to_string())
        .await?;

    Ok(StatusCode::NO_CONTENT)
}
```

### 2. FEA Solver Core (Rust — Nonlinear Structural)

Core nonlinear FEA solver implementing total Lagrangian formulation with Newton-Raphson iteration and MUMPS sparse direct solver.

```rust
// solver/src/fea/mod.rs

use nalgebra::{DVector, DMatrix};
use crate::mesh::{Mesh, Element};
use crate::material::{Material, HyperelasticModel};
use std::collections::HashMap;

pub struct FeaSolver {
    mesh: Mesh,
    materials: HashMap<String, Box<dyn Material>>,
    boundary_conditions: BoundaryConditions,
    solution: DVector<f64>,  // Displacement DOFs
    residual: DVector<f64>,
    stiffness: SparseCsrMatrix,
}

impl FeaSolver {
    pub fn new(mesh: Mesh, materials: HashMap<String, Box<dyn Material>>) -> Self {
        let ndofs = mesh.n_nodes * 3;  // 3 DOFs per node (ux, uy, uz)
        Self {
            mesh,
            materials,
            boundary_conditions: BoundaryConditions::default(),
            solution: DVector::zeros(ndofs),
            residual: DVector::zeros(ndofs),
            stiffness: SparseCsrMatrix::new(ndofs, ndofs),
        }
    }

    /// Solve nonlinear static equilibrium via Newton-Raphson
    pub fn solve_static(&mut self, options: &SolverOptions) -> Result<SolutionState, SolverError> {
        for iteration in 0..options.max_iterations {
            // 1. Assemble residual and tangent stiffness
            self.assemble_system();

            // 2. Apply boundary conditions (penalty method or direct elimination)
            self.apply_boundary_conditions();

            // 3. Solve for displacement increment: K_T Δu = -R
            let delta_u = self.solve_linear_system()?;

            // 4. Update solution
            self.solution += &delta_u;

            // 5. Check convergence
            let residual_norm = self.residual.norm();
            let solution_norm = self.solution.norm();
            let relative_error = residual_norm / (solution_norm + 1e-16);

            tracing::info!(
                "Iteration {}: residual_norm={:.3e}, relative_error={:.3e}",
                iteration, residual_norm, relative_error
            );

            if relative_error < options.convergence_tol {
                tracing::info!("Converged in {} iterations", iteration + 1);
                return Ok(self.extract_solution_state());
            }
        }

        Err(SolverError::Convergence(
            format!("Failed to converge after {} iterations", options.max_iterations)
        ))
    }

    /// Assemble residual R and tangent stiffness K_T
    fn assemble_system(&mut self) {
        self.residual.fill(0.0);
        self.stiffness.clear();

        for elem in &self.mesh.elements {
            let mat = &self.materials[&elem.material_id];

            // Get element nodal displacements
            let u_e = self.extract_element_dofs(elem);

            // Compute deformation gradient F = I + ∇u at each quadrature point
            let (f_int, k_e) = self.compute_element_contribution(elem, &u_e, mat.as_ref());

            // Assemble into global system
            self.assemble_element_residual(elem, &f_int);
            self.assemble_element_stiffness(elem, &k_e);
        }

        // Add external forces
        self.apply_external_loads();
    }

    fn compute_element_contribution(
        &self,
        elem: &Element,
        u_e: &DVector<f64>,
        material: &dyn Material,
    ) -> (DVector<f64>, DMatrix<f64>) {
        let n_nodes = elem.node_ids.len();
        let n_dofs = n_nodes * 3;
        let mut f_int = DVector::zeros(n_dofs);
        let mut k_e = DMatrix::zeros(n_dofs, n_dofs);

        // Gauss quadrature (e.g., 4 points for Tet4, 8 for Hex8)
        let quad_points = elem.get_quadrature_points();

        for qp in quad_points {
            // Shape function derivatives in reference config: dN/dξ
            let shape_derivs = elem.compute_shape_derivatives(&qp.coords);

            // Jacobian X_ξ = ∂X/∂ξ (reference config)
            let jac_ref = self.compute_jacobian(elem, &shape_derivs);
            let jac_inv = jac_ref.try_inverse()
                .ok_or(SolverError::SingularElement(elem.id))?;

            // Shape derivatives in spatial coords: dN/dX = dN/dξ · (dξ/dX)
            let grad_n = &shape_derivs * &jac_inv;

            // Deformation gradient F = I + ∇u
            let f_matrix = self.compute_deformation_gradient(&grad_n, u_e);

            // Material constitutive response: P, C_mat (tangent modulus)
            let (pk1_stress, tangent) = material.compute_stress_tangent(&f_matrix);

            // Internal force contribution: f_int += ∫ (∇N)^T · P dΩ
            let b_matrix = self.compute_b_matrix(&grad_n);  // Discrete gradient operator
            f_int += &b_matrix.transpose() * &pk1_stress.as_vector() * qp.weight * jac_ref.determinant();

            // Tangent stiffness: K_e += ∫ B^T C B dΩ + geometric stiffness
            let k_material = &b_matrix.transpose() * &tangent * &b_matrix * qp.weight * jac_ref.determinant();
            let k_geometric = self.compute_geometric_stiffness(&grad_n, &pk1_stress, qp.weight * jac_ref.determinant());
            k_e += k_material + k_geometric;
        }

        (f_int, k_e)
    }

    fn solve_linear_system(&mut self) -> Result<DVector<f64>, SolverError> {
        // Call MUMPS sparse direct solver via FFI
        let mut solver = MumpsSolver::new(self.stiffness.clone())?;
        solver.analyze()?;
        solver.factorize()?;
        let delta_u = solver.solve(&(-&self.residual))?;
        Ok(delta_u)
    }

    fn extract_solution_state(&self) -> SolutionState {
        let mut node_displacements = Vec::new();
        let mut element_stresses = Vec::new();
        let mut element_strains = Vec::new();

        // Extract nodal displacements
        for i in 0..self.mesh.n_nodes {
            node_displacements.push([
                self.solution[i * 3],
                self.solution[i * 3 + 1],
                self.solution[i * 3 + 2],
            ]);
        }

        // Compute element-wise stress/strain at centroid
        for elem in &self.mesh.elements {
            let u_e = self.extract_element_dofs(elem);
            let mat = &self.materials[&elem.material_id];

            // Evaluate at element centroid
            let centroid_coords = elem.centroid_natural_coords();
            let shape_derivs = elem.compute_shape_derivatives(&centroid_coords);
            let jac_ref = self.compute_jacobian(elem, &shape_derivs);
            let jac_inv = jac_ref.try_inverse().unwrap();
            let grad_n = &shape_derivs * &jac_inv;

            let f_matrix = self.compute_deformation_gradient(&grad_n, &u_e);
            let (pk1, _) = mat.compute_stress_tangent(&f_matrix);

            // Convert to Cauchy stress: σ = (1/J) P F^T
            let j = f_matrix.determinant();
            let cauchy = (1.0 / j) * &pk1 * f_matrix.transpose();

            element_stresses.push(cauchy);
            element_strains.push(self.compute_strain(&f_matrix));
        }

        SolutionState {
            displacements: node_displacements,
            element_stresses,
            element_strains,
            max_displacement: node_displacements.iter()
                .map(|d| (d[0]*d[0] + d[1]*d[1] + d[2]*d[2]).sqrt())
                .fold(0.0, f64::max),
            max_von_mises: element_stresses.iter()
                .map(|s| self.compute_von_mises(s))
                .fold(0.0, f64::max),
        }
    }

    fn compute_von_mises(&self, cauchy: &DMatrix<f64>) -> f64 {
        // Von Mises stress: sqrt(3 J_2)
        let s_dev = cauchy - (cauchy.trace() / 3.0) * DMatrix::identity(3, 3);
        let j2 = 0.5 * (s_dev[(0,0)].powi(2) + s_dev[(1,1)].powi(2) + s_dev[(2,2)].powi(2) +
                        2.0 * (s_dev[(0,1)].powi(2) + s_dev[(0,2)].powi(2) + s_dev[(1,2)].powi(2)));
        (3.0 * j2).sqrt()
    }
}

#[derive(Debug)]
pub struct SolutionState {
    pub displacements: Vec<[f64; 3]>,
    pub element_stresses: Vec<DMatrix<f64>>,
    pub element_strains: Vec<DMatrix<f64>>,
    pub max_displacement: f64,
    pub max_von_mises: f64,
}

#[derive(Debug)]
pub enum SolverError {
    Convergence(String),
    SingularMatrix(String),
    SingularElement(usize),
    InvalidMaterial(String),
}
```

### 3. Material Model Implementation (Rust — Hyperelastic)

Implements hyperelastic constitutive models for biological tissues and device materials.

```rust
// solver/src/material/mod.rs

use nalgebra::DMatrix;

pub trait Material: Send + Sync {
    /// Compute First Piola-Kirchhoff stress P and material tangent C
    /// given deformation gradient F
    fn compute_stress_tangent(&self, f: &DMatrix<f64>) -> (DMatrix<f64>, DMatrix<f64>);
}

pub struct NeoHookeanMaterial {
    pub mu: f64,      // Shear modulus (Pa)
    pub kappa: f64,   // Bulk modulus (Pa)
}

impl Material for NeoHookeanMaterial {
    fn compute_stress_tangent(&self, f: &DMatrix<f64>) -> (DMatrix<f64>, DMatrix<f64>) {
        let j = f.determinant();
        let c = f.transpose() * f;  // Right Cauchy-Green tensor
        let c_inv = c.try_inverse().expect("Singular C");

        // Second Piola-Kirchhoff stress from Neo-Hookean potential
        // S = μ I + (κ(J-1) - μ) C^-1
        let s = self.mu * DMatrix::identity(3, 3) + (self.kappa * (j - 1.0) - self.mu) * c_inv;

        // First Piola-Kirchhoff: P = F S
        let p = f * &s;

        // Material tangent (simplified for brevity — full implementation is 81 components)
        let tangent = self.compute_tangent_modulus(f, &c_inv, j);

        (p, tangent)
    }
}

impl NeoHookeanMaterial {
    fn compute_tangent_modulus(&self, f: &DMatrix<f64>, c_inv: &DMatrix<f64>, j: f64) -> DMatrix<f64> {
        // Elasticity tensor C_ijkl = ∂²Ψ/∂C_ij ∂C_kl
        // For Neo-Hookean: simplified form (full tensor is 3x3x3x3)
        // Stored as 9x9 for Voigt notation compatibility
        let mut c_mat = DMatrix::zeros(9, 9);

        // Material contribution
        let kappa_term = self.kappa * j * (2.0 * j - 1.0);
        let mu_term = self.mu - self.kappa * (j - 1.0);

        // Voigt notation assembly (σ_11, σ_22, σ_33, σ_12, σ_13, σ_23, σ_21, σ_31, σ_32)
        for i in 0..3 {
            for j_idx in 0..3 {
                for k in 0..3 {
                    for l in 0..3 {
                        let voigt_ij = if i == j_idx { i } else { 3 + i.min(j_idx) };
                        let voigt_kl = if k == l { k } else { 3 + k.min(l) };

                        c_mat[(voigt_ij, voigt_kl)] += kappa_term * c_inv[(i, j_idx)] * c_inv[(k, l)]
                            - mu_term * (c_inv[(i, k)] * c_inv[(j_idx, l)] + c_inv[(i, l)] * c_inv[(j_idx, k)]);
                    }
                }
            }
        }

        c_mat
    }
}

pub struct HolzapfelMaterial {
    pub mu: f64,        // Ground substance shear modulus
    pub k1: f64,        // Fiber stiffness
    pub k2: f64,        // Fiber exponential coefficient
    pub fiber_dirs: Vec<[f64; 3]>,  // Fiber family directions in ref config
}

impl Material for HolzapfelMaterial {
    fn compute_stress_tangent(&self, f: &DMatrix<f64>) -> (DMatrix<f64>, DMatrix<f64>) {
        let c = f.transpose() * f;
        let j = f.determinant();

        // Isotropic contribution (Neo-Hookean ground substance)
        let c_inv = c.try_inverse().unwrap();
        let mut s = self.mu * DMatrix::identity(3, 3);

        // Anisotropic fiber contributions
        for fiber_dir in &self.fiber_dirs {
            let a0 = DMatrix::from_row_slice(3, 1, fiber_dir);  // Fiber direction vector
            let i4 = (a0.transpose() * &c * &a0)[(0, 0)];  // Fiber stretch

            if i4 > 1.0 {  // Fiber only resists tension
                let exp_term = (self.k2 * (i4 - 1.0).powi(2)).exp();
                let stress_fiber = 2.0 * self.k1 * (i4 - 1.0) * exp_term;

                // S += stress_fiber * (a0 ⊗ a0)
                s += stress_fiber * (&a0 * a0.transpose());
            }
        }

        // First Piola-Kirchhoff
        let p = f * &s;

        // Tangent (simplified — full derivation involves fiber tangent modulus)
        let tangent = self.compute_tangent(f, &c, &c_inv, j);

        (p, tangent)
    }
}

impl HolzapfelMaterial {
    fn compute_tangent(&self, f: &DMatrix<f64>, c: &DMatrix<f64>, c_inv: &DMatrix<f64>, j: f64) -> DMatrix<f64> {
        let mut tangent = DMatrix::zeros(9, 9);

        // Isotropic part (similar to Neo-Hookean)
        // ... (omitted for brevity)

        // Anisotropic fiber tangent contributions
        for fiber_dir in &self.fiber_dirs {
            let a0 = DMatrix::from_row_slice(3, 1, fiber_dir);
            let i4 = (a0.transpose() * c * &a0)[(0, 0)];

            if i4 > 1.0 {
                let exp_term = (self.k2 * (i4 - 1.0).powi(2)).exp();
                let tangent_coeff = 2.0 * self.k1 * exp_term * (1.0 + 2.0 * self.k2 * (i4 - 1.0).powi(2));

                // Add (a0 ⊗ a0 ⊗ a0 ⊗ a0) contribution to tangent
                for i in 0..3 {
                    for j_idx in 0..3 {
                        for k in 0..3 {
                            for l in 0..3 {
                                let voigt_ij = if i == j_idx { i } else { 3 + i };
                                let voigt_kl = if k == l { k } else { 3 + k };
                                tangent[(voigt_ij, voigt_kl)] +=
                                    tangent_coeff * a0[i] * a0[j_idx] * a0[k] * a0[l];
                            }
                        }
                    }
                }
            }
        }

        tangent
    }
}
```

### 4. Fatigue Analysis Implementation (Rust)

Implements S-N fatigue life prediction with mean stress correction and Miner's Rule damage accumulation.

```rust
// solver/src/fatigue/mod.rs

use crate::fea::SolutionState;

pub struct FatigueSolver {
    pub material_sn_curve: SNCurve,
    pub mean_stress_correction: MeanStressCorrection,
}

#[derive(Debug, Clone)]
pub struct SNCurve {
    pub fatigue_strength_coeff: f64,  // σ'_f
    pub fatigue_strength_exp: f64,    // b (typically -0.1 to -0.15)
    pub ultimate_strength: f64,       // σ_u
    pub endurance_limit: f64,         // σ_e (if applicable, 0 for no limit)
}

#[derive(Debug, Clone)]
pub enum MeanStressCorrection {
    Goodman,
    Gerber,
    Soderberg,
    None,
}

impl FatigueSolver {
    /// Compute fatigue life from cyclic stress history
    pub fn compute_fatigue_life(&self, stress_history: &[f64]) -> Result<FatigueResult, String> {
        // 1. Extract cycles via rainflow counting
        let cycles = self.rainflow_count(stress_history);

        // 2. Apply mean stress correction and compute damage for each cycle
        let mut total_damage = 0.0;
        let mut cycle_damages = Vec::new();

        for (stress_amplitude, mean_stress, count) in cycles {
            // Apply mean stress correction
            let corrected_amplitude = self.apply_mean_stress_correction(
                stress_amplitude, mean_stress
            );

            // Compute cycles to failure from S-N curve
            let cycles_to_failure = self.cycles_to_failure(corrected_amplitude);

            // Miner's Rule damage: D = n / N
            let damage = count / cycles_to_failure;
            total_damage += damage;

            cycle_damages.push(CycleDamage {
                amplitude: stress_amplitude,
                mean_stress,
                count,
                cycles_to_failure,
                damage,
            });
        }

        Ok(FatigueResult {
            total_damage,
            cycles_to_failure: 1.0 / total_damage,
            cycle_damages,
        })
    }

    fn apply_mean_stress_correction(&self, amplitude: f64, mean: f64) -> f64 {
        match self.mean_stress_correction {
            MeanStressCorrection::Goodman => {
                // σ_a,eq = σ_a / (1 - σ_m / σ_u)
                amplitude / (1.0 - mean / self.material_sn_curve.ultimate_strength)
            }
            MeanStressCorrection::Gerber => {
                // σ_a,eq = σ_a / (1 - (σ_m / σ_u)²)
                amplitude / (1.0 - (mean / self.material_sn_curve.ultimate_strength).powi(2))
            }
            MeanStressCorrection::Soderberg => {
                // σ_a,eq = σ_a / (1 - σ_m / σ_y)
                // Using ultimate strength as proxy for yield (conservative)
                amplitude / (1.0 - mean / (0.8 * self.material_sn_curve.ultimate_strength))
            }
            MeanStressCorrection::None => amplitude,
        }
    }

    fn cycles_to_failure(&self, stress_amplitude: f64) -> f64 {
        // Basquin's Law: σ_a = σ'_f (2N_f)^b
        // Solve for N_f: N_f = (σ_a / σ'_f)^(1/b) / 2

        let curve = &self.material_sn_curve;

        // Check endurance limit
        if curve.endurance_limit > 0.0 && stress_amplitude < curve.endurance_limit {
            return f64::INFINITY;  // Infinite life below endurance limit
        }

        let ratio = stress_amplitude / curve.fatigue_strength_coeff;
        let exponent = 1.0 / curve.fatigue_strength_exp;
        ratio.powf(exponent) / 2.0
    }

    fn rainflow_count(&self, stress_history: &[f64]) -> Vec<(f64, f64, f64)> {
        // Simplified rainflow counting algorithm
        // Returns: Vec<(amplitude, mean_stress, cycle_count)>

        let mut peaks_valleys = self.extract_peaks_valleys(stress_history);
        let mut cycles = Vec::new();
        let mut stack: Vec<f64> = Vec::new();

        for &value in &peaks_valleys {
            stack.push(value);

            while stack.len() >= 3 {
                let x = stack[stack.len() - 3];
                let y = stack[stack.len() - 2];
                let z = stack[stack.len() - 1];

                let range_xy = (y - x).abs();
                let range_yz = (z - y).abs();

                if range_yz >= range_xy {
                    // Closed cycle found
                    let amplitude = range_xy / 2.0;
                    let mean = (x + y) / 2.0;
                    cycles.push((amplitude, mean, 1.0));

                    // Remove x and y from stack
                    stack.remove(stack.len() - 3);
                    stack.remove(stack.len() - 2);
                } else {
                    break;
                }
            }
        }

        // Aggregate identical cycles
        self.aggregate_cycles(cycles)
    }

    fn extract_peaks_valleys(&self, history: &[f64]) -> Vec<f64> {
        let mut result = Vec::new();
        if history.is_empty() {
            return result;
        }

        result.push(history[0]);

        for i in 1..history.len() - 1 {
            let prev = history[i - 1];
            let curr = history[i];
            let next = history[i + 1];

            // Peak: curr > prev and curr > next
            // Valley: curr < prev and curr < next
            if (curr > prev && curr > next) || (curr < prev && curr < next) {
                result.push(curr);
            }
        }

        result.push(history[history.len() - 1]);
        result
    }

    fn aggregate_cycles(&self, cycles: Vec<(f64, f64, f64)>) -> Vec<(f64, f64, f64)> {
        let mut aggregated: std::collections::HashMap<(u64, u64), f64> = std::collections::HashMap::new();

        for (amp, mean, count) in cycles {
            // Quantize to avoid floating point comparison issues
            let amp_key = (amp * 1e6) as u64;
            let mean_key = (mean * 1e6) as u64;
            *aggregated.entry((amp_key, mean_key)).or_insert(0.0) += count;
        }

        aggregated.into_iter()
            .map(|((amp_key, mean_key), count)| {
                (amp_key as f64 / 1e6, mean_key as f64 / 1e6, count)
            })
            .collect()
    }
}

#[derive(Debug)]
pub struct FatigueResult {
    pub total_damage: f64,
    pub cycles_to_failure: f64,
    pub cycle_damages: Vec<CycleDamage>,
}

#[derive(Debug)]
pub struct CycleDamage {
    pub amplitude: f64,
    pub mean_stress: f64,
    pub count: f64,
    pub cycles_to_failure: f64,
    pub damage: f64,
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–5)

**Day 1: Project initialization and Rust backend scaffold**
```bash
cargo init biomedica-api
cd biomedica-api
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
- `migrations/001_initial.sql` — All 10 tables: users, organizations, org_members, projects, anatomies, materials, simulations, simulation_jobs, regulatory_reports, loading_profiles, usage_records
- `src/db/mod.rs` — Database pool initialization
- `src/db/models.rs` — All SQLx structs with FromRow derives
- Run `sqlx migrate run` to apply schema
- Seed script for initial anatomies, materials, loading profiles

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

**Day 5: Material and anatomy library API**
- `src/api/handlers/materials.rs` — Search materials (parametric + full-text), get material
- `src/api/handlers/anatomies.rs` — List anatomies by body region, get anatomy, download mesh
- S3 integration for anatomy VTK file retrieval
- Material categories: metal, polymer, ceramic, biological_tissue, shape_memory_alloy
- Anatomy categories: cardiovascular, orthopedic, neurological, spinal
- Pagination with cursor-based scrolling for large result sets

### Phase 2 — FEA Solver Core (Days 6–14)

**Day 6: Mesh data structures and I/O**
- `solver/src/mesh/mod.rs` — Mesh, Element, Node, Face structs
- `solver/src/mesh/vtk_io.rs` — VTK file reader/writer (unstructured grid)
- `solver/src/mesh/quality.rs` — Mesh quality metrics (min angle, aspect ratio, Jacobian)
- Element types: Tet4, Tet10, Hex8, Hex20
- Tests: load VTK mesh, compute quality metrics, write VTK

**Day 7: Mesh generation via TetGen bindings**
- `solver/src/mesh/generator.rs` — Rust FFI bindings to TetGen C library
- Surface mesh (STL) → volume mesh (Tet4/Tet10)
- Boundary layer mesh for contact surfaces (refinement factor 3x)
- Mesh quality constraints (min angle 15°, max aspect ratio 10)
- Tests: generate mesh from cube STL, verify element count and quality

**Day 8: Linear elastic FEA solver**
- `solver/src/fea/mod.rs` — FeaSolver struct, assembly, solve
- `solver/src/fea/element.rs` — Element stiffness matrix computation (Tet4, Hex8)
- `solver/src/fea/sparse.rs` — Sparse CSR matrix assembly
- `solver/src/fea/mumps.rs` — MUMPS sparse direct solver FFI bindings
- Linear elastic material: E = 200 GPa, ν = 0.3 (steel)
- Tests: cantilever beam under tip load, compare to analytical solution

**Day 9: Hyperelastic material models — Neo-Hookean**
- `solver/src/material/mod.rs` — Material trait, NeoHookeanMaterial
- `solver/src/material/neo_hookean.rs` — Strain energy, stress, tangent modulus
- Integrate into FEA solver: compute_element_contribution with hyperelastic stress
- Tests: uniaxial tension test, compare to analytical Neo-Hookean response

**Day 10: Hyperelastic material models — Holzapfel-Gasser-Ogden**
- `solver/src/material/holzapfel.rs` — HGO model for arterial wall with fiber families
- Fiber direction specification in reference configuration
- Anisotropic stress contribution (fibers resist tension only)
- Tests: biaxial stretch of arterial wall, verify fiber activation

**Day 11: Nonlinear solver — Newton-Raphson**
- `solver/src/fea/nonlinear.rs` — Newton-Raphson loop with convergence criteria
- Tangent stiffness matrix assembly (material + geometric)
- Line search for robustness (backtracking)
- Tests: large deformation of rubber block, convergence in 5-10 iterations

**Day 12: Contact mechanics — penalty method**
- `solver/src/fea/contact.rs` — Contact detection, gap function, penalty stiffness
- Master-slave contact pairs with normal and tangential (friction) penalties
- Friction coefficient μ (Coulomb friction model)
- Tests: two blocks in contact under compression, verify no penetration

**Day 13: Boundary conditions and loading**
- `solver/src/fea/boundary.rs` — Dirichlet (fixed displacement), Neumann (traction)
- Loading profiles: constant, ramp, sinusoidal, from JSON time-series
- Pressure loads on surfaces
- Tests: beam with pressure load, compare to analytical

**Day 14: FEA result post-processing**
- `solver/src/fea/postprocess.rs` — Compute von Mises stress, principal stresses, strain
- Nodal averaging for smooth contour plots
- Deformed mesh generation (X + u)
- VTU output with stress/strain fields for ParaView visualization

### Phase 3 — CFD Solver Core (Days 15–21)

**Day 15: CFD mesh structures and finite volume discretization**
- `solver/src/cfd/mod.rs` — CfdSolver struct, cell-centered variables
- `solver/src/cfd/mesh.rs` — Cell, Face, FaceNeighbor structs
- Compute cell volumes, face areas, face normals, distance between cell centers
- Tests: structured 3D grid, verify geometric properties

**Day 16: Incompressible Navier-Stokes — momentum equation**
- `solver/src/cfd/momentum.rs` — Discretize convection and diffusion terms
- Upwind differencing for convection
- Central differencing for diffusion
- Pressure gradient from previous iteration
- Tests: 2D lid-driven cavity (no-slip walls, moving top lid)

**Day 17: SIMPLE algorithm — pressure-velocity coupling**
- `solver/src/cfd/simple.rs` — Pressure correction equation, velocity correction
- Rhie-Chow interpolation for collocated grid
- Relaxation factors for pressure and velocity
- Tests: 2D channel flow, verify parabolic velocity profile

**Day 18: Boundary conditions for CFD**
- `solver/src/cfd/boundary.rs` — Inlet (velocity), outlet (pressure), no-slip wall, slip wall
- Wall shear stress computation from velocity gradient
- Tests: flow through cylinder, verify WSS distribution

**Day 19: Non-Newtonian fluid models**
- `solver/src/cfd/non_newtonian.rs` — Carreau-Yasuda, Power-Law viscosity models
- Shear rate computation: γ̇ = √(2 tr(D²))
- Viscosity update in momentum equation
- Tests: flow through stenosis, verify shear-thinning behavior

**Day 20: Transient CFD — time integration**
- `solver/src/cfd/transient.rs` — Implicit Euler time marching
- PISO algorithm (Pressure Implicit with Splitting of Operators)
- CFL condition for timestep selection
- Tests: pulsatile flow in tube, verify time-accurate velocity waveform

**Day 21: CFD result post-processing**
- `solver/src/cfd/postprocess.rs` — Streamlines, velocity magnitude, WSS magnitude
- OSI (oscillatory shear index) for pulsatile flow
- TAWSS (time-averaged wall shear stress)
- VTU output with velocity/pressure fields

### Phase 4 — Fatigue Solver + Simulation Worker (Days 22–26)

**Day 22: Fatigue analysis — rainflow counting**
- `solver/src/fatigue/rainflow.rs` — Rainflow cycle counting algorithm
- Extract peaks and valleys from stress history
- Cycle extraction and aggregation
- Tests: synthetic stress history with known cycles

**Day 23: Fatigue analysis — S-N curves and damage**
- `solver/src/fatigue/sn_curve.rs` — SNCurve struct, cycles_to_failure
- Basquin's Law implementation
- Mean stress correction (Goodman, Gerber, Soderberg)
- Miner's Rule damage accumulation
- Tests: CoCr alloy with known S-N data, verify damage calculation

**Day 24: Fatigue solver integration**
- `solver/src/fatigue/mod.rs` — FatigueSolver struct, compute_fatigue_life
- Load stress history from FEA transient or cyclic analysis
- Output: total damage, cycles to failure, critical location
- Tests: hip implant under gait cycle loading (ISO 14242-1)

**Day 25: Simulation worker — mesh generation and solver execution**
- `src/workers/simulation_worker.rs` — Redis job consumer, runs native solver
- Phase 1: Download geometry from S3, generate mesh via TetGen
- Phase 2: Load materials and BCs, assemble FEA/CFD system
- Phase 3: Solve (Newton-Raphson for FEA, SIMPLE for CFD)
- Phase 4: Post-process, upload VTU to S3
- Progress streaming: worker publishes progress → Redis pub/sub → WebSocket to client

**Day 26: Simulation worker — error handling and convergence monitoring**
- Convergence failure detection: iteration limit reached, residual divergence
- Element quality warnings: negative Jacobian, high aspect ratio
- Out-of-memory handling: graceful cancellation with error message
- Simulation cancellation via Redis pub/sub
- Tests: submit simulation, cancel mid-execution, verify cleanup

### Phase 5 — Frontend + 3D Visualization (Days 27–33)

**Day 27: Frontend scaffold and project management**
```bash
npm create vite@latest frontend -- --template react-ts
cd frontend
npm i zustand @tanstack/react-query axios three @react-three/fiber @react-three/drei
```
- `src/App.tsx` — Router, auth context, layout
- `src/stores/authStore.ts` — Auth state (JWT, user)
- `src/stores/projectStore.ts` — Project state
- `src/pages/Dashboard.tsx` — Project list with thumbnail cards
- `src/pages/Editor.tsx` — Main editor layout (3D viewer + properties panel)

**Day 28: 3D visualization — mesh rendering with Three.js**
- `src/components/Viewer3D/MeshViewer.tsx` — React Three Fiber canvas
- Load VTK mesh from S3, convert to Three.js BufferGeometry
- Orbit controls for rotation, zoom, pan
- Lighting: directional + ambient
- Tests: load femur mesh, verify render

**Day 29: 3D visualization — stress/strain contour plots**
- `src/components/Viewer3D/ContourShader.tsx` — Custom shader for scalar field coloring
- Color map (Jet, Viridis, Plasma) with legend
- Contour levels with discrete color bands
- Min/max value display
- Tests: load FEA result with von Mises stress, verify color mapping

**Day 30: 3D visualization — deformation animation**
- `src/components/Viewer3D/DeformationAnimation.tsx` — Animate deformed shape
- Scale factor slider (1x, 10x, 100x exaggeration)
- Time slider for transient results
- Play/pause/reset controls
- Tests: load implant compression result, verify deformation

**Day 31: 3D visualization — flow streamlines**
- `src/components/Viewer3D/Streamlines.tsx` — Render CFD velocity streamlines
- Seed points from inlet surface
- Line rendering with velocity magnitude coloring
- Arrow glyphs for flow direction
- Tests: load aorta CFD result, verify streamlines

**Day 32: Project configuration UI — materials and boundary conditions**
- `src/components/Editor/MaterialAssignment.tsx` — Assign materials to geometry regions
- Drag material from library onto 3D model region
- `src/components/Editor/BoundaryConditions.tsx` — Define loads, constraints, inlet/outlet
- Visual BC indicators on mesh (arrows for forces, cylinders for constraints)
- Load profile selector with preview graph

**Day 33: Simulation setup and monitoring**
- `src/components/Editor/SimulationSetup.tsx` — Analysis type selector, solver options
- Run simulation button → enqueue job, show progress modal
- `src/hooks/useSimulationProgress.ts` — WebSocket hook for live progress
- Progress bar with phase indicator (meshing 30%, solving 60%, etc.)
- Convergence plot (residual vs. iteration)

### Phase 6 — Regulatory Report Generation (Days 34–36)

**Day 34: Python FastAPI report service setup**
```bash
cd report-service
pip install fastapi uvicorn reportlab pillow numpy
```
- `main.py` — FastAPI app with `/generate_report` endpoint
- `report_generator.py` — ReportLab PDF generation
- Input: JSON with simulation metadata, mesh convergence data, results summary
- Output: PDF following FDA Guidance structure

**Day 35: Report sections — model description and V&V**
- `sections/model_description.py` — Geometry description, material properties table, constitutive equations
- `sections/boundary_conditions.py` — Loading and constraint diagrams (embed matplotlib plots)
- `sections/mesh_convergence.py` — Table with 3 mesh refinement levels, peak stress convergence
- `sections/results.py` — Contour plots, time-series graphs, max/min values
- Tests: generate report for hip implant, verify all sections present

**Day 36: Report integration and download**
- `src/api/handlers/reports.rs` — Create regulatory report, get report, download PDF
- Call Python report service via HTTP, store PDF in S3
- `src/pages/ReportViewer.tsx` — PDF preview in browser
- Download button for PDF
- Tests: run simulation, generate report, verify PDF structure

### Phase 7 — Billing + Plan Enforcement (Days 37–39)

**Day 37: Stripe integration**
- `src/api/handlers/billing.rs` — Create checkout session, create customer portal session, get subscription status
- `src/api/handlers/webhooks/stripe.rs` — Handle subscription events: `checkout.session.completed`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.payment_failed`
- Plan mapping:
  - Free: 50K elements, 5 simulations/month, no regulatory reports
  - Academic ($99/mo): 500K elements, unlimited simulations, basic reports, 5 users
  - Pro ($349/mo): 2M elements, fatigue analysis, full FDA reports, API access
  - Enterprise: Custom pricing, unlimited, on-premise deployment option

**Day 38: Usage tracking and plan limits**
- `src/middleware/plan_limits.rs` — Middleware checking plan limits before simulation execution
- `src/services/usage.rs` — Track simulation count and compute hours per billing period
- Usage record insertion after each simulation completes
- Usage dashboard: `src/api/handlers/usage.rs` — Current period usage, historical usage
- Approaching-limit warnings at 80% and 100% of monthly quota

**Day 39: Billing UI and feature gating**
- `frontend/src/pages/Billing.tsx` — Plan comparison table, current plan display, usage meter
- `frontend/src/components/billing/PlanCard.tsx` — Plan feature comparison cards
- Upgrade/downgrade flow via Stripe Customer Portal
- Gate fatigue analysis behind Pro plan
- Gate regulatory report generation behind Pro/Enterprise plan
- Show locked feature indicators with upgrade CTAs in UI

### Phase 8 — Solver Validation + Deployment (Days 40–42)

**Day 40: Solver validation benchmarks**
- Benchmark 1: Cantilever beam (FEA) — compare tip deflection to analytical (δ = PL³/3EI)
- Benchmark 2: Cylindrical pressure vessel (FEA) — compare hoop stress to σ = pr/t
- Benchmark 3: Poiseuille flow in pipe (CFD) — compare velocity profile to analytical
- Benchmark 4: Hip implant fatigue (ISO 14242-1) — compare to reference data
- Benchmark 5: Arterial wall stretch (Holzapfel) — compare to published experimental data
- Automated test suite: `solver/tests/validation_benchmarks.rs` with assertions

**Day 41: Docker and Kubernetes configuration**
- `Dockerfile` — Multi-stage Rust build (cargo-chef for layer caching)
- `docker-compose.yml` — Full local dev stack (API, PostgreSQL, Redis, MinIO, Python report service, frontend)
- `k8s/` — Kubernetes manifests:
  - `api-deployment.yaml` — API server (3 replicas, HPA)
  - `worker-deployment.yaml` — Simulation workers (auto-scaling based on queue depth)
  - `report-service-deployment.yaml` — Python report service (2 replicas)
  - `postgres-statefulset.yaml` — PostgreSQL with PVC
  - `redis-deployment.yaml` — Redis for job queue and pub/sub
  - `ingress.yaml` — NGINX ingress with TLS
- Health check endpoints: `/health/live`, `/health/ready`

**Day 42: Monitoring, polish, and launch**
- Prometheus metrics: simulation duration histogram, solver convergence rate, API latency percentiles, queue depth
- Grafana dashboards: system health, simulation throughput, user activity, convergence failures
- Sentry integration: frontend and backend error tracking with source maps
- Structured logging (tracing + JSON) for all API requests and solver events
- UI polish: loading states, error messages, empty states, responsive layout
- End-to-end smoke test in production environment
- Rate limiting: 100 req/min per user, 10 simulations/hour per user
- Security audit: JWT validation, SQL injection (SQLx prevents this), CORS configuration, CSP headers
- Landing page with product overview, pricing, and signup
- Deploy to production, enable monitoring alerts, announce launch

---

## Critical Files

```
biomedica/
├── solver/                              # Rust solver library
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── mesh/
│   │   │   ├── mod.rs                   # Mesh data structures
│   │   │   ├── vtk_io.rs                # VTK file I/O
│   │   │   ├── generator.rs             # TetGen bindings
│   │   │   └── quality.rs               # Mesh quality metrics
│   │   ├── material/
│   │   │   ├── mod.rs                   # Material trait
│   │   │   ├── linear_elastic.rs
│   │   │   ├── neo_hookean.rs           # Neo-Hookean hyperelastic
│   │   │   ├── holzapfel.rs             # Holzapfel-Gasser-Ogden arterial wall
│   │   │   └── mooney_rivlin.rs
│   │   ├── fea/
│   │   │   ├── mod.rs                   # FeaSolver
│   │   │   ├── element.rs               # Element stiffness (Tet4, Hex8)
│   │   │   ├── nonlinear.rs             # Newton-Raphson solver
│   │   │   ├── contact.rs               # Contact mechanics (penalty)
│   │   │   ├── boundary.rs              # Boundary conditions
│   │   │   ├── sparse.rs                # Sparse matrix assembly
│   │   │   ├── mumps.rs                 # MUMPS FFI bindings
│   │   │   └── postprocess.rs           # Stress/strain calculation
│   │   ├── cfd/
│   │   │   ├── mod.rs                   # CfdSolver
│   │   │   ├── mesh.rs                  # CFD cell-centered mesh
│   │   │   ├── momentum.rs              # Momentum equation discretization
│   │   │   ├── simple.rs                # SIMPLE algorithm
│   │   │   ├── boundary.rs              # CFD boundary conditions
│   │   │   ├── non_newtonian.rs         # Carreau-Yasuda model
│   │   │   ├── transient.rs             # Time integration (PISO)
│   │   │   └── postprocess.rs           # WSS, streamlines
│   │   └── fatigue/
│   │       ├── mod.rs                   # FatigueSolver
│   │       ├── rainflow.rs              # Rainflow cycle counting
│   │       └── sn_curve.rs              # S-N curve damage calculation
│   └── tests/
│       ├── validation_benchmarks.rs     # Solver validation suite
│       └── integration.rs               # FEA+CFD integration tests
│
├── biomedica-api/                       # Rust API server
│   ├── Cargo.toml
│   ├── src/
│   │   ├── main.rs                      # Axum app setup, router, startup
│   │   ├── config.rs                    # Environment config
│   │   ├── state.rs                     # AppState (PgPool, Redis, S3)
│   │   ├── error.rs                     # ApiError types
│   │   ├── auth/
│   │   │   ├── mod.rs                   # JWT middleware, Claims
│   │   │   └── oauth.rs                 # Google/GitHub OAuth handlers
│   │   ├── api/
│   │   │   ├── router.rs                # Route definitions
│   │   │   ├── handlers/
│   │   │   │   ├── auth.rs              # Register, login, OAuth
│   │   │   │   ├── users.rs             # Profile CRUD
│   │   │   │   ├── projects.rs          # Project CRUD, fork
│   │   │   │   ├── simulation.rs        # Create/get/cancel simulation
│   │   │   │   ├── materials.rs         # Material library search/get
│   │   │   │   ├── anatomies.rs         # Anatomy library list/get
│   │   │   │   ├── reports.rs           # Regulatory report generation
│   │   │   │   ├── billing.rs           # Stripe checkout/portal
│   │   │   │   ├── usage.rs             # Usage tracking
│   │   │   │   └── webhooks/
│   │   │   │       └── stripe.rs        # Stripe webhook handler
│   │   │   └── ws/
│   │   │       ├── mod.rs               # WebSocket upgrade handler
│   │   │       └── simulation_progress.rs # Live progress streaming
│   │   ├── middleware/
│   │   │   └── plan_limits.rs           # Plan enforcement middleware
│   │   ├── services/
│   │   │   ├── usage.rs                 # Usage tracking service
│   │   │   └── s3.rs                    # S3 client helpers
│   │   └── workers/
│   │       ├── mod.rs                   # Worker pool management
│   │       └── simulation_worker.rs     # Mesh + solver execution
│   ├── migrations/
│   │   └── 001_initial.sql              # Full database schema
│   └── tests/
│       ├── api_integration.rs           # API integration tests
│       └── simulation_e2e.rs            # End-to-end simulation tests
│
├── frontend/                            # React frontend
│   ├── package.json
│   ├── vite.config.ts
│   ├── src/
│   │   ├── App.tsx                      # Router, providers, layout
│   │   ├── main.tsx                     # Entry point
│   │   ├── stores/
│   │   │   ├── authStore.ts             # Auth state (JWT, user)
│   │   │   └── projectStore.ts          # Project state
│   │   ├── hooks/
│   │   │   └── useSimulationProgress.ts # WebSocket hook for live progress
│   │   ├── lib/
│   │   │   ├── api.ts                   # Axios API client
│   │   │   └── vtk_loader.ts            # VTK mesh loading
│   │   ├── pages/
│   │   │   ├── Dashboard.tsx            # Project list
│   │   │   ├── Editor.tsx               # Main editor (3D viewer + props)
│   │   │   ├── Billing.tsx              # Plan management
│   │   │   ├── ReportViewer.tsx         # PDF report preview
│   │   │   ├── Login.tsx                # Auth pages
│   │   │   └── Register.tsx
│   │   └── components/
│   │       ├── Viewer3D/
│   │       │   ├── MeshViewer.tsx       # Three.js mesh rendering
│   │       │   ├── ContourShader.tsx    # Stress/strain contours
│   │       │   ├── DeformationAnimation.tsx # Deformed shape animation
│   │       │   └── Streamlines.tsx      # CFD flow streamlines
│   │       ├── Editor/
│   │       │   ├── MaterialAssignment.tsx # Material selection UI
│   │       │   ├── BoundaryConditions.tsx # BC definition UI
│   │       │   └── SimulationSetup.tsx    # Analysis config
│   │       └── billing/
│   │           ├── PlanCard.tsx
│   │           └── UsageMeter.tsx
│
├── report-service/                      # Python FastAPI (regulatory reports)
│   ├── requirements.txt
│   ├── main.py                          # FastAPI app
│   ├── report_generator.py              # ReportLab PDF generation
│   ├── sections/
│   │   ├── model_description.py         # Model description section
│   │   ├── boundary_conditions.py       # BC diagrams
│   │   ├── mesh_convergence.py          # Convergence table
│   │   └── results.py                   # Results plots
│   └── Dockerfile
│
├── k8s/                                 # Kubernetes manifests
│   ├── api-deployment.yaml
│   ├── worker-deployment.yaml
│   ├── postgres-statefulset.yaml
│   ├── redis-deployment.yaml
│   ├── report-service-deployment.yaml
│   └── ingress.yaml
│
├── docker-compose.yml                   # Local development stack
├── Cargo.toml                           # Workspace root
└── .github/
    └── workflows/
        ├── ci.yml                       # Test + lint on PR
        └── deploy.yml                   # Build Docker images + deploy to K8s
```

---

## Solver Validation Suite

### Benchmark 1: Cantilever Beam Tip Deflection (FEA)

**Geometry:** L=100mm × W=10mm × H=10mm beam, fixed at one end

**Load:** P = 10N downward force at free end

**Material:** Linear elastic, E = 200 GPa, ν = 0.3 (steel)

**Expected tip deflection:** δ = PL³/(3EI) = 10 × 0.1³/(3 × 200e9 × (0.01 × 0.01³/12)) = **6.0 mm**

**Tolerance:** < 1% (mesh convergence with 10K elements)

### Benchmark 2: Cylindrical Pressure Vessel Hoop Stress (FEA)

**Geometry:** Cylinder r=50mm, t=5mm (thin-wall assumption valid)

**Load:** Internal pressure p = 10 MPa

**Material:** Linear elastic, E = 200 GPa, ν = 0.3

**Expected hoop stress:** σ_θ = pr/t = 10e6 × 0.05 / 0.005 = **100 MPa**

**Tolerance:** < 2% (analytical thin-wall solution)

### Benchmark 3: Poiseuille Flow in Circular Pipe (CFD)

**Geometry:** Pipe R=10mm, L=100mm

**Boundary conditions:** Inlet velocity v_avg = 0.1 m/s, outlet pressure p = 0 Pa, no-slip wall

**Fluid:** Newtonian, μ = 0.001 Pa·s (water), ρ = 1000 kg/m³

**Expected centerline velocity:** v_max = 2 × v_avg = **0.2 m/s**

**Expected velocity profile:** v(r) = v_max(1 - (r/R)²)

**Tolerance:** < 2% at r = 0, r = R/2 (mesh with 50K cells)

### Benchmark 4: Hip Implant Fatigue Life (ISO 14242-1)

**Geometry:** Simplified femoral stem, CoCr alloy

**Load:** ISO 14242-1 Paul Load Profile (peak 3000N, 1 Hz, 10M cycles)

**Material:** CoCr L605, σ'_f = 1200 MPa, b = -0.12, σ_u = 900 MPa

**Expected peak stress:** σ_max ≈ 150 MPa (at neck region)

**Expected fatigue life:** N_f > 10M cycles (no failure expected for compliant design)

**Tolerance:** Predicted life within factor of 2 (conservative design margin)

### Benchmark 5: Arterial Wall Biaxial Stretch (Holzapfel Model)

**Geometry:** 10mm × 10mm arterial wall patch, t=1mm

**Load:** Equibiaxial stretch λ = 1.3 (30% strain)

**Material:** Holzapfel parameters: μ = 30 kPa, k1 = 10 kPa, k2 = 5, fiber angle ±45°

**Expected Cauchy stress (literature):** σ ≈ **50-70 kPa**

**Tolerance:** < 15% (experimental data variability in published studies)

---

## Verification

### Integration Testing Checklist

1. **Auth flow** — Register → login → JWT issued → authorized API calls → token refresh → logout
2. **Project CRUD** — Create → update config → save → reload → verify config preserved
3. **Material library** — Search "CoCr" → results returned → select material → assign to device
4. **Anatomy library** — List cardiovascular anatomies → select aorta → download VTK → visualize
5. **Geometry upload** — Upload STL hip implant → preview in 3D viewer → assign to project
6. **Material assignment** — Drag CoCr material onto device region → verify assignment in UI
7. **Boundary conditions** — Define fixed constraint at base → apply 3000N force → verify BC visualization
8. **Mesh generation** — Run mesher → verify 100K elements generated → check mesh quality (min angle > 10°)
9. **FEA simulation** — Submit static analysis → job queued → worker picks up → WebSocket progress → results in S3
10. **Result visualization** — Load VTU result → render von Mises contours → verify peak stress location
11. **Fatigue analysis** — Run fatigue with gait cycle → verify cycles to failure computed
12. **Regulatory report** — Generate FDA report → verify PDF has all 5 sections → download PDF
13. **Plan limits** — Free user → 60K element mesh → blocked with upgrade prompt
14. **Billing** — Subscribe to Pro → Stripe checkout → webhook → plan updated → limits lifted
15. **Concurrent simulations** — 5 users simultaneously running FEA → all complete correctly

### SQL Verification Queries

```sql
-- 1. Simulation throughput and success rate
SELECT
  DATE(created_at) AS date,
  COUNT(*) AS total_sims,
  SUM(CASE WHEN status = 'completed' THEN 1 ELSE 0 END) AS completed,
  AVG(wall_time_s) AS avg_time_s
FROM simulations
WHERE created_at > NOW() - INTERVAL '7 days'
GROUP BY DATE(created_at)
ORDER BY date DESC;

-- 2. Top materials by usage
SELECT
  m.name,
  m.category,
  COUNT(s.id) AS simulation_count
FROM materials m
JOIN simulations s ON s.material_assignments::jsonb ? m.id::text
GROUP BY m.id, m.name, m.category
ORDER BY simulation_count DESC
LIMIT 10;

-- 3. Mesh quality distribution
SELECT
  CASE
    WHEN (mesh_stats->>'min_quality')::float < 0.2 THEN 'poor'
    WHEN (mesh_stats->>'min_quality')::float < 0.5 THEN 'fair'
    WHEN (mesh_stats->>'min_quality')::float < 0.8 THEN 'good'
    ELSE 'excellent'
  END AS quality_tier,
  COUNT(*) AS count
FROM simulations
WHERE mesh_stats IS NOT NULL
GROUP BY quality_tier;

-- 4. Convergence failure rate by analysis type
SELECT
  analysis_type,
  COUNT(*) AS total,
  SUM(CASE WHEN status = 'failed' AND error_message LIKE '%convergence%' THEN 1 ELSE 0 END) AS convergence_failures,
  ROUND(100.0 * SUM(CASE WHEN status = 'failed' AND error_message LIKE '%convergence%' THEN 1 ELSE 0 END) / COUNT(*), 2) AS failure_rate_pct
FROM simulations
WHERE status IN ('completed', 'failed')
GROUP BY analysis_type;

-- 5. User engagement and retention
SELECT
  u.plan,
  COUNT(DISTINCT u.id) AS user_count,
  AVG(sim_counts.sim_count) AS avg_sims_per_user,
  SUM(sim_counts.sim_count) AS total_sims
FROM users u
LEFT JOIN (
  SELECT user_id, COUNT(*) AS sim_count
  FROM simulations
  WHERE created_at > NOW() - INTERVAL '30 days'
  GROUP BY user_id
) sim_counts ON u.id = sim_counts.user_id
GROUP BY u.plan;
```

---

## Deployment Architecture

```
                                      ┌─────────────────┐
                                      │   CloudFront    │
                                      │   (CDN)         │
                                      └────────┬────────┘
                                               │
                                               │ HTTPS
                                               │
                         ┌─────────────────────┼─────────────────────┐
                         │                     │                     │
                    ┌────▼────┐           ┌────▼────┐           ┌────▼────┐
                    │ Static  │           │  API    │           │ Anatomy │
                    │ Assets  │           │ Server  │           │ Library │
                    │ (React) │           │ (Axum)  │           │  (S3)   │
                    └─────────┘           └────┬────┘           └─────────┘
                                               │
                         ┌─────────────────────┼─────────────────────┐
                         │                     │                     │
                    ┌────▼────┐           ┌────▼────┐           ┌────▼────┐
                    │ Worker  │           │  Redis  │           │ Report  │
                    │ Pool    │           │ (Queue) │           │ Service │
                    │ (Rust)  │           └─────────┘           │(FastAPI)│
                    └────┬────┘                                 └─────────┘
                         │
                         │ Execute
                         │ Solver
                         │
                    ┌────▼────┐
                    │ Solver  │
                    │ (Rust   │
                    │ FEA/CFD)│
                    └────┬────┘
                         │
                         │ Store Results
                         │
                    ┌────▼────┐
                    │   S3    │
                    │ (Mesh,  │
                    │ Results)│
                    └─────────┘

Database Layer:
┌──────────────────────────────────────────────────────────┐
│                      PostgreSQL 16                       │
│  ┌─────────┬─────────┬─────────┬─────────┬─────────┐   │
│  │ Users   │Projects │Materials│Anatomies│  Sims   │   │
│  └─────────┴─────────┴─────────┴─────────┴─────────┘   │
└──────────────────────────────────────────────────────────┘

Monitoring:
┌──────────────────────────────────────────────────────────┐
│  Prometheus  →  Grafana  (Metrics & Dashboards)          │
│  Sentry  (Error Tracking & Alerting)                     │
└──────────────────────────────────────────────────────────┘
```

**Infrastructure:**
- **Kubernetes (EKS)**: API pods (3 replicas), Worker pods (auto-scale 2-20 based on queue depth), PostgreSQL (StatefulSet with 500GB PVC), Redis (Deployment with 2 replicas)
- **S3**: Mesh files (compressed VTK), simulation results (VTU), anatomical library (3GB), regulatory reports (PDFs)
- **CloudFront**: CDN for static frontend assets and anatomy library with 24h cache TTL
- **Auto-scaling**: HPA for API (CPU > 70%), Custom metrics for workers (queue depth > 10 jobs)
- **Backup**: PostgreSQL daily snapshots to S3, S3 versioning enabled for all critical data

---

## Post-MVP Roadmap

### Q2 2026 — FSI and Advanced CFD
- **Fluid-structure interaction** (partitioned FSI with Aitken relaxation): stent deployment → recoil → flow evaluation
- **Heart valve simulation**: leaflet opening/closing dynamics, regurgitation volume
- **Turbulence models**: k-ε, k-ω SST for high Reynolds number flows
- **Hemolysis prediction**: blood damage index from shear stress history
- **Platelet activation**: thrombosis risk assessment

### Q3 2026 — Patient-Specific Modeling
- **DICOM import and segmentation**: Python FastAPI service with SimpleITK, VTK for surface extraction
- **Image registration**: align pre-operative CT with device CAD model
- **Statistical shape models**: population variability studies for implant sizing
- **HIPAA compliance**: BAA with cloud provider, PHI encryption at rest and in transit, audit logging

### Q4 2026 — Electromagnetic and Thermal
- **Electromagnetic FEM**: neurostimulation electrode field distribution in tissue
- **MRI compatibility**: SAR (Specific Absorption Rate) calculation for implant safety
- **RF ablation**: temperature distribution, Arrhenius damage integral for tissue necrosis
- **Volume of tissue activated** (VTA): DBS stimulation volume prediction

### Q1 2027 — Advanced Materials and Degradation
- **Nitinol fatigue**: pseudoelastic model (Auricchio) with strain-based fatigue (ASTM F2477)
- **Biodegradable materials**: degradation kinetics coupled with mechanical properties (PLGA, Mg alloys)
- **Corrosion fatigue**: environment-assisted cracking in simulated body fluid
- **Wear analysis**: Archard law for UHMWPE, metal-on-metal bearing surfaces

### Q2 2027 — Collaboration and Regulatory Integration
- **Real-time collaboration**: WebRTC for multi-user editing of project with conflict resolution
- **QMS integration**: export simulation data to quality management systems (eQMS, MasterControl)
- **21 CFR Part 11 compliance**: electronic signature, audit trail, access control for regulated environments
- **FDA pre-cert program**: automated V&V documentation package for FDA pre-submission meetings
