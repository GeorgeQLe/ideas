# 32. PrintGen — AI Generative Design & 3D Print Optimization Platform

## Implementation Plan

**MVP Scope:** Browser-based 3D viewport with STL/OBJ upload and design space region marking (keep/exclude/optimize zones) rendered via Three.js, GPU-accelerated SIMP topology optimization solver (Rust + wgpu compute shaders) running on cloud NVIDIA A10G with real-time density field streaming via WebSocket, support for 64³ to 256³ voxel resolution (configurable by plan tier), visual load and constraint definition (forces, pressures, fixed/pinned supports) with arrow/symbol overlay on the 3D model, linear static FEA validation with GPU-accelerated conjugate gradient solver for stress (von Mises) and displacement color-mapped visualization, automatic mesh extraction from density field using marching cubes with STL export, 20+ built-in 3D printing materials (FDM: PLA/PETG/ABS/Nylon, SLA: Standard/Tough resin, SLS: PA12, Metal: 316L/Ti-6Al-4V) with Young's modulus, yield strength, density properties stored in PostgreSQL, print orientation optimization for minimum support volume, cost/time estimation with configurable material and machine costs, Stripe billing with four tiers (Free / Pro $99/mo / Team $299/mo up to 5 seats / Enterprise custom).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| Topology Optimizer | Rust + wgpu compute shaders | SIMP method with GPU-accelerated density field updates, runs on A10G/A100 |
| FEA Solver | Rust + wgpu compute shaders | GPU-accelerated PCG with algebraic multigrid preconditioning |
| Mesh Processing | Rust (marchingcubes, trimesh) | Marching cubes extraction, mesh repair, smoothing, decimation |
| CAD File Processing | Python 3.12 (FastAPI) | STEP/STL/OBJ/3MF import, mesh conversion via Open CASCADE and trimesh |
| Database | PostgreSQL 16 | Users, projects, design spaces, optimization runs, FEA results, materials |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | CAD uploads, mesh files, density fields, FEA results, optimization snapshots |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| 3D Viewport | Three.js + WebGL | Custom mesh rendering, stress color mapping, section planes, load/constraint arrows |
| Real-time Updates | WebSocket (Axum) | Live density field streaming during optimization, convergence plot updates |
| Job Queue | Redis 7 + Tokio tasks | GPU job scheduling, optimization queue with priority levels |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| GPU Orchestration | Kubernetes (EKS) | GPU node pool with NVIDIA A10G, dynamic scaling, job affinity |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, GPU utilization, error tracking |
| CI/CD | GitHub Actions | Rust tests, Docker image build and push, GPU shader validation |

### Key Architecture Decisions

1. **GPU-accelerated SIMP topology optimization with wgpu compute shaders**: SIMP (Solid Isotropic Material with Penalization) is the industry-standard topology optimization method used in nTopology and Fusion 360 Generative. Our Rust implementation uses wgpu compute shaders to parallelize density updates, sensitivity computation, and filtering across the voxel grid on NVIDIA A10G GPUs, achieving 20-40x speedup over CPU. This enables interactive optimization at 256³ resolution in 3-10 minutes instead of hours.

2. **Cloud-only compute model with no client-side optimization**: Unlike SpiceSim's hybrid WASM/server approach, PrintGen runs all optimization and FEA on cloud GPUs because topology optimization requires sustained high-performance compute (5-20 minutes for Pro tier, 30-60 minutes for Team tier at 512³). Client browsers receive real-time density field updates via WebSocket and render the evolving design using Three.js voxel visualization.

3. **Voxel-based density representation with marching cubes extraction**: The optimizer operates on a regular voxel grid where each cell has a density ρ ∈ [0,1]. After convergence, we extract the isosurface at ρ=0.5 using the marching cubes algorithm to generate a triangle mesh (STL/OBJ). This is simpler than level-set methods and produces manufacturable geometries with controllable minimum feature size via density filtering.

4. **Integrated FEA validation on optimized geometry**: After optimization, we automatically mesh the extracted geometry and run GPU-accelerated linear static FEA to validate stress and displacement. The FEA solver uses a preconditioned conjugate gradient (PCG) method with algebraic multigrid (AMG) preconditioning, implemented in wgpu compute shaders. This workflow ensures that optimized designs meet structural requirements before export.

5. **Three.js for 3D visualization with WebGL stress color mapping**: The frontend uses Three.js for all 3D rendering: design space visualization, load/constraint arrows, real-time density field evolution (rendered as instanced cubes or point cloud), and FEA result color mapping. We use custom shaders for stress visualization with configurable color scales (rainbow, blue-red, grayscale). Section plane tools allow users to inspect internal structures.

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
    plan TEXT NOT NULL DEFAULT 'free',  -- free | pro | team | enterprise
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    compute_quota_hours_monthly REAL DEFAULT 2.0,  -- Free: 2h, Pro: 50h, Team: 200h
    compute_used_hours_current_period REAL DEFAULT 0.0,
    quota_reset_at TIMESTAMPTZ NOT NULL DEFAULT (date_trunc('month', NOW()) + INTERVAL '1 month'),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX users_email_idx ON users(email);
CREATE INDEX users_stripe_idx ON users(stripe_customer_id);

-- Organizations (for Team and Enterprise plans)
CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    slug TEXT UNIQUE NOT NULL,
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    plan TEXT NOT NULL DEFAULT 'team',
    seat_count INTEGER NOT NULL DEFAULT 5,
    billing_email TEXT NOT NULL,
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    compute_quota_hours_monthly REAL DEFAULT 200.0,
    compute_used_hours_current_period REAL DEFAULT 0.0,
    quota_reset_at TIMESTAMPTZ NOT NULL DEFAULT (date_trunc('month', NOW()) + INTERVAL '1 month'),
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

-- Projects (top-level container for design workspace)
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    visibility TEXT NOT NULL DEFAULT 'private',  -- private | org | public
    forked_from UUID REFERENCES projects(id) ON DELETE SET NULL,
    thumbnail_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);
CREATE INDEX projects_org_idx ON projects(org_id);
CREATE INDEX projects_updated_idx ON projects(updated_at DESC);
CREATE INDEX projects_visibility_idx ON projects(visibility);

-- Design Spaces (CAD upload + regions)
CREATE TABLE design_spaces (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    source_file_url TEXT NOT NULL,  -- S3 URL to uploaded STEP/STL/OBJ
    source_file_type TEXT NOT NULL,  -- step | stl | obj | 3mf
    mesh_url TEXT,  -- S3 URL to processed mesh (STL)
    bounding_box JSONB,  -- {min: [x,y,z], max: [x,y,z]}
    mesh_stats JSONB,  -- {triangle_count, vertex_count, surface_area_mm2, volume_mm3}
    keep_regions JSONB DEFAULT '[]',  -- [{type, geometry}] — regions that must remain solid
    exclude_regions JSONB DEFAULT '[]',  -- [{type, geometry}] — regions where material cannot exist
    symmetry_planes JSONB DEFAULT '[]',  -- [{normal: [x,y,z], point: [x,y,z]}]
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX design_spaces_project_idx ON design_spaces(project_id);

-- Load Cases (forces, constraints)
CREATE TABLE load_cases (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    design_space_id UUID NOT NULL REFERENCES design_spaces(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    loads JSONB NOT NULL DEFAULT '[]',  -- [{type: force|pressure|moment|gravity, geometry, magnitude, direction}]
    constraints JSONB NOT NULL DEFAULT '[]',  -- [{type: fixed|pinned|roller, geometry}]
    weight REAL NOT NULL DEFAULT 1.0,  -- For multi-load-case optimization
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX load_cases_design_space_idx ON load_cases(design_space_id);

-- Materials (3D printing materials with mechanical properties)
CREATE TABLE materials (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    category TEXT NOT NULL,  -- fdm | sla | sls | metal | custom
    subcategory TEXT,  -- e.g., "PLA" for FDM, "Standard Resin" for SLA
    youngs_modulus_mpa REAL NOT NULL,
    poissons_ratio REAL NOT NULL,
    yield_strength_mpa REAL,
    ultimate_strength_mpa REAL,
    density_kg_m3 REAL NOT NULL,
    elongation_pct REAL,
    fatigue_limit_mpa REAL,
    cost_per_kg REAL,  -- USD per kg for cost estimation
    is_builtin BOOLEAN DEFAULT true,
    org_id UUID REFERENCES organizations(id) ON DELETE CASCADE,  -- NULL for built-in materials
    description TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX materials_category_idx ON materials(category);
CREATE INDEX materials_builtin_idx ON materials(is_builtin);
CREATE INDEX materials_org_idx ON materials(org_id);

-- Optimization Runs
CREATE TABLE optimization_runs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    design_space_id UUID NOT NULL REFERENCES design_spaces(id) ON DELETE CASCADE,
    material_id UUID NOT NULL REFERENCES materials(id),
    load_case_ids UUID[] NOT NULL,  -- Array of load_case IDs
    method TEXT NOT NULL DEFAULT 'simp',  -- simp | level_set
    objective TEXT NOT NULL DEFAULT 'min_compliance',  -- min_compliance | min_weight | max_frequency
    params JSONB NOT NULL,  -- {volume_fraction, penalty, filter_radius, min_feature_size_mm, max_overhang_deg}
    resolution INTEGER NOT NULL,  -- Voxel grid size (64 | 128 | 256 | 512)
    status TEXT NOT NULL DEFAULT 'queued',  -- queued | running | completed | failed | cancelled
    iteration_count INTEGER DEFAULT 0,
    convergence_data JSONB DEFAULT '[]',  -- [{iteration, objective_value, volume_fraction, max_change}]
    result_mesh_url TEXT,  -- S3 URL to extracted mesh (STL)
    density_field_url TEXT,  -- S3 URL to final density field (binary)
    result_metrics JSONB,  -- {final_weight_kg, final_compliance, final_volume_fraction, iteration_count}
    gpu_type TEXT,  -- a10g | a100 | t4
    compute_time_seconds REAL,
    cost_usd REAL,
    error_message TEXT,
    started_by UUID NOT NULL REFERENCES users(id),
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX opt_runs_project_idx ON optimization_runs(project_id);
CREATE INDEX opt_runs_status_idx ON optimization_runs(status);
CREATE INDEX opt_runs_created_idx ON optimization_runs(created_at DESC);

-- FEA Results (validation simulation on optimized geometry)
CREATE TABLE fea_results (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    optimization_run_id UUID REFERENCES optimization_runs(id) ON DELETE CASCADE,
    design_space_id UUID REFERENCES design_spaces(id) ON DELETE CASCADE,
    material_id UUID NOT NULL REFERENCES materials(id),
    load_case_id UUID NOT NULL REFERENCES load_cases(id),
    status TEXT NOT NULL DEFAULT 'queued',  -- queued | running | completed | failed
    analysis_type TEXT NOT NULL DEFAULT 'linear_static',  -- linear_static | modal | thermal
    results JSONB,  -- {max_stress_mpa, max_displacement_mm, safety_factor, natural_frequencies_hz}
    stress_field_url TEXT,  -- S3 URL to per-element stress field
    displacement_field_url TEXT,  -- S3 URL to per-node displacement field
    mesh_quality JSONB,  -- {element_count, avg_aspect_ratio, min_jacobian}
    compute_time_seconds REAL,
    error_message TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at TIMESTAMPTZ
);
CREATE INDEX fea_results_opt_run_idx ON fea_results(optimization_run_id);
CREATE INDEX fea_results_status_idx ON fea_results(status);

-- Lattice Configurations (future feature — v1.1)
CREATE TABLE lattice_configs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    optimization_run_id UUID NOT NULL REFERENCES optimization_runs(id) ON DELETE CASCADE,
    unit_cell_type TEXT NOT NULL,  -- gyroid | schwarz_p | diamond | bcc | fcc | octet
    density_mapping TEXT NOT NULL DEFAULT 'uniform',  -- uniform | stress_graded | custom
    min_density REAL NOT NULL DEFAULT 0.1,
    max_density REAL NOT NULL DEFAULT 0.9,
    cell_size_mm REAL NOT NULL,
    strut_diameter_mm REAL,  -- For strut-based lattices
    result_mesh_url TEXT,  -- S3 URL to lattice-filled mesh
    result_metrics JSONB,  -- {weight_kg, surface_area_mm2, volume_reduction_pct}
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX lattice_configs_opt_run_idx ON lattice_configs(optimization_run_id);

-- Comments (collaboration feature)
CREATE TABLE comments (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    author_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    body TEXT NOT NULL,
    anchor_type TEXT,  -- design | optimization | fea | lattice
    anchor_data JSONB,  -- {type-specific data, e.g., 3D position for spatial comment}
    resolved BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX comments_project_idx ON comments(project_id);
CREATE INDEX comments_author_idx ON comments(author_id);

-- Usage Tracking (for billing and quota enforcement)
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE CASCADE,
    record_type TEXT NOT NULL,  -- gpu_compute_hours | storage_gb_hours
    quantity REAL NOT NULL,
    period_start DATE NOT NULL,
    period_end DATE NOT NULL,
    metadata JSONB,  -- {optimization_run_id, gpu_type, resolution}
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
    pub compute_quota_hours_monthly: f32,
    pub compute_used_hours_current_period: f32,
    pub quota_reset_at: DateTime<Utc>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Organization {
    pub id: Uuid,
    pub name: String,
    pub slug: String,
    pub owner_id: Uuid,
    pub plan: String,
    pub seat_count: i32,
    pub billing_email: String,
    pub stripe_customer_id: Option<String>,
    pub stripe_subscription_id: Option<String>,
    pub compute_quota_hours_monthly: f32,
    pub compute_used_hours_current_period: f32,
    pub quota_reset_at: DateTime<Utc>,
    pub settings: serde_json::Value,
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
    pub forked_from: Option<Uuid>,
    pub thumbnail_url: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct DesignSpace {
    pub id: Uuid,
    pub project_id: Uuid,
    pub name: String,
    pub source_file_url: String,
    pub source_file_type: String,
    pub mesh_url: Option<String>,
    pub bounding_box: Option<serde_json::Value>,
    pub mesh_stats: Option<serde_json::Value>,
    pub keep_regions: serde_json::Value,
    pub exclude_regions: serde_json::Value,
    pub symmetry_planes: serde_json::Value,
    pub created_by: Uuid,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct LoadCase {
    pub id: Uuid,
    pub design_space_id: Uuid,
    pub name: String,
    pub description: String,
    pub loads: serde_json::Value,
    pub constraints: serde_json::Value,
    pub weight: f32,
    pub created_by: Uuid,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Material {
    pub id: Uuid,
    pub name: String,
    pub category: String,
    pub subcategory: Option<String>,
    pub youngs_modulus_mpa: f32,
    pub poissons_ratio: f32,
    pub yield_strength_mpa: Option<f32>,
    pub ultimate_strength_mpa: Option<f32>,
    pub density_kg_m3: f32,
    pub elongation_pct: Option<f32>,
    pub fatigue_limit_mpa: Option<f32>,
    pub cost_per_kg: Option<f32>,
    pub is_builtin: bool,
    pub org_id: Option<Uuid>,
    pub description: Option<String>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct OptimizationRun {
    pub id: Uuid,
    pub project_id: Uuid,
    pub design_space_id: Uuid,
    pub material_id: Uuid,
    pub load_case_ids: Vec<Uuid>,
    pub method: String,
    pub objective: String,
    pub params: serde_json::Value,
    pub resolution: i32,
    pub status: String,
    pub iteration_count: i32,
    pub convergence_data: serde_json::Value,
    pub result_mesh_url: Option<String>,
    pub density_field_url: Option<String>,
    pub result_metrics: Option<serde_json::Value>,
    pub gpu_type: Option<String>,
    pub compute_time_seconds: Option<f32>,
    pub cost_usd: Option<f32>,
    pub error_message: Option<String>,
    pub started_by: Uuid,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct FeaResult {
    pub id: Uuid,
    pub optimization_run_id: Option<Uuid>,
    pub design_space_id: Option<Uuid>,
    pub material_id: Uuid,
    pub load_case_id: Uuid,
    pub status: String,
    pub analysis_type: String,
    pub results: Option<serde_json::Value>,
    pub stress_field_url: Option<String>,
    pub displacement_field_url: Option<String>,
    pub mesh_quality: Option<serde_json::Value>,
    pub compute_time_seconds: Option<f32>,
    pub error_message: Option<String>,
    pub created_at: DateTime<Utc>,
    pub completed_at: Option<DateTime<Utc>>,
}

#[derive(Debug, Deserialize)]
pub struct OptimizationParams {
    pub volume_fraction: f32,  // Target volume as fraction of design space (e.g., 0.3 = 30%)
    pub penalty: f32,  // SIMP penalty power (default 3.0)
    pub filter_radius_mm: f32,  // Density filter radius for manufacturability
    pub min_feature_size_mm: Option<f32>,  // Minimum feature size constraint
    pub max_overhang_deg: Option<f32>,  // Maximum overhang angle for 3D printing
}

#[derive(Debug, Serialize, Deserialize)]
pub struct LoadDefinition {
    pub load_type: String,  // force | pressure | moment | gravity
    pub geometry: serde_json::Value,  // Face IDs, vertex IDs, or point coordinates
    pub magnitude: f32,
    pub direction: Option<[f32; 3]>,  // For force and moment
}

#[derive(Debug, Serialize, Deserialize)]
pub struct ConstraintDefinition {
    pub constraint_type: String,  // fixed | pinned | roller
    pub geometry: serde_json::Value,  // Face IDs or vertex IDs
}
```

---

## Topology Optimization Architecture Deep-Dive

### SIMP Method Overview

PrintGen implements **SIMP (Solid Isotropic Material with Penalization)**, the standard topology optimization method. For a design domain discretized into `n` voxels, each voxel `i` has a density `ρᵢ ∈ [0,1]` where 0 = void and 1 = solid material.

**Objective:** Minimize compliance (maximize stiffness) subject to a volume constraint:

```
minimize:   C(ρ) = Uᵀ K(ρ) U
subject to: V(ρ) = Σᵢ ρᵢ vᵢ ≤ V_max
            0 < ρ_min ≤ ρᵢ ≤ 1

where:
  U = displacement vector (FEA solution)
  K(ρ) = global stiffness matrix (function of densities)
  V_max = target volume (e.g., 30% of design space)
  ρ_min = minimum density (typically 0.001 to avoid singularity)
```

**Material interpolation:** The element stiffness is penalized by density:

```
Kᵢ(ρᵢ) = ρᵢᵖ · K₀

where:
  p = penalty power (default 3.0)
  K₀ = solid material element stiffness
```

The penalty `p` ensures that intermediate densities (0 < ρ < 1) are penalized, pushing the solution toward 0-1 (void-solid) designs.

**Sensitivity analysis:** The sensitivity of compliance to density changes is:

```
∂C/∂ρᵢ = -p · ρᵢᵖ⁻¹ · uᵢᵀ K₀ uᵢ

where uᵢ = element displacement vector
```

**Density update:** Using the Method of Moving Asymptotes (MMA) or Optimality Criteria (OC):

```
ρᵢⁿ⁺¹ = max(ρ_min, ρᵢⁿ - m)  if  ρᵢⁿ Bᵢ^η ≤ max(ρ_min, ρᵢⁿ - m)
ρᵢⁿ⁺¹ = ρᵢⁿ Bᵢ^η            if  max(ρ_min, ρᵢⁿ - m) < ρᵢⁿ Bᵢ^η < min(1, ρᵢⁿ + m)
ρᵢⁿ⁺¹ = min(1, ρᵢⁿ + m)      if  min(1, ρᵢⁿ + m) ≤ ρᵢⁿ Bᵢ^η

where:
  Bᵢ = (-∂C/∂ρᵢ) / (λ ∂V/∂ρᵢ)  (Lagrange multiplier ratio)
  η = damping factor (0.5)
  m = move limit (0.2)
  λ = Lagrange multiplier (bisection search to satisfy volume constraint)
```

**Density filtering:** To ensure manufacturability (minimum feature size), we apply a convolution filter:

```
ρ̃ᵢ = (Σⱼ wᵢⱼ vⱼ ρⱼ) / (Σⱼ wᵢⱼ vⱼ)

where:
  wᵢⱼ = max(0, r - dist(i,j))  (linear hat filter)
  r = filter radius (user-specified minimum feature size)
```

### GPU Implementation with wgpu

The SIMP optimizer is implemented as a series of wgpu compute shader passes:

1. **FEA solve pass:** Solve `K(ρ) U = F` using GPU-accelerated PCG (see FEA section below)
2. **Sensitivity computation pass:** Compute `∂C/∂ρᵢ` for each element using element displacement vectors
3. **Density filtering pass:** Apply convolution filter to smooth sensitivity field
4. **Density update pass:** Update densities using OC method with Lagrange multiplier
5. **Volume normalization pass:** Adjust λ via binary search to satisfy volume constraint
6. **Convergence check pass:** Compute max density change; if below threshold, terminate

Each pass operates on GPU buffers containing density field, displacement field, sensitivity field, and stiffness matrix data.

---

## Architecture Deep-Dives

### 1. Optimization API Handler (Rust/Axum)

The primary endpoint receives an optimization request, validates parameters, checks quota, and enqueues a GPU job.

```rust
// src/api/handlers/optimization.rs

use axum::{
    extract::{Path, State, Json},
    http::StatusCode,
    response::IntoResponse,
};
use uuid::Uuid;
use crate::{
    db::models::{OptimizationRun, OptimizationParams},
    state::AppState,
    auth::Claims,
    error::ApiError,
};

#[derive(serde::Deserialize)]
pub struct CreateOptimizationRequest {
    pub design_space_id: Uuid,
    pub material_id: Uuid,
    pub load_case_ids: Vec<Uuid>,
    pub method: String,  // simp | level_set
    pub objective: String,  // min_compliance | min_weight
    pub params: OptimizationParams,
    pub resolution: i32,  // 64 | 128 | 256 | 512
}

pub async fn create_optimization(
    State(state): State<AppState>,
    claims: Claims,
    Path(project_id): Path<Uuid>,
    Json(req): Json<CreateOptimizationRequest>,
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

    // 2. Verify design space and load cases belong to project
    let design_space = sqlx::query_as!(
        crate::db::models::DesignSpace,
        "SELECT * FROM design_spaces WHERE id = $1 AND project_id = $2",
        req.design_space_id,
        project_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Design space not found"))?;

    // 3. Check plan limits
    let user = sqlx::query_as!(
        crate::db::models::User,
        "SELECT * FROM users WHERE id = $1",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    // Plan-based resolution limits
    let max_resolution = match user.plan.as_str() {
        "free" => 64,
        "pro" => 256,
        "team" => 512,
        "enterprise" => 1024,
        _ => 64,
    };

    if req.resolution > max_resolution {
        return Err(ApiError::PlanLimit(
            format!("Your plan supports up to {}³ resolution. Upgrade for higher resolution.", max_resolution)
        ));
    }

    // Check compute quota
    if user.compute_used_hours_current_period >= user.compute_quota_hours_monthly {
        return Err(ApiError::QuotaExceeded(
            "Monthly compute quota exceeded. Upgrade your plan or wait for next billing cycle."
        ));
    }

    // 4. Estimate compute time for cost calculation
    let estimated_minutes = estimate_optimization_time(req.resolution, &req.method);
    let gpu_cost_per_hour = 1.20; // A10G cost
    let estimated_cost = (estimated_minutes / 60.0) * gpu_cost_per_hour;

    // 5. Create optimization run record
    let opt_run = sqlx::query_as!(
        OptimizationRun,
        r#"INSERT INTO optimization_runs
            (project_id, design_space_id, material_id, load_case_ids,
             method, objective, params, resolution, status, started_by)
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8, 'queued', $9)
        RETURNING *"#,
        project_id,
        req.design_space_id,
        req.material_id,
        &req.load_case_ids,
        req.method,
        req.objective,
        serde_json::to_value(&req.params)?,
        req.resolution,
        claims.user_id,
    )
    .fetch_one(&state.db)
    .await?;

    // 6. Enqueue GPU job to Redis with priority
    let priority = match user.plan.as_str() {
        "enterprise" => 100,
        "team" => 50,
        "pro" => 10,
        _ => 0,
    };

    let job = serde_json::json!({
        "optimization_run_id": opt_run.id,
        "priority": priority,
        "estimated_minutes": estimated_minutes,
    });

    state.redis
        .zadd("optimization:jobs", serde_json::to_string(&job)?, priority)
        .await?;

    // 7. Publish notification for worker pickup
    state.redis
        .publish("optimization:new_job", opt_run.id.to_string())
        .await?;

    Ok((StatusCode::CREATED, Json(opt_run)))
}

pub async fn get_optimization(
    State(state): State<AppState>,
    claims: Claims,
    Path((project_id, opt_id)): Path<(Uuid, Uuid)>,
) -> Result<Json<OptimizationRun>, ApiError> {
    let opt = sqlx::query_as!(
        OptimizationRun,
        "SELECT o.* FROM optimization_runs o
         JOIN projects p ON o.project_id = p.id
         WHERE o.id = $1 AND o.project_id = $2
         AND (p.owner_id = $3 OR p.org_id IN (
             SELECT org_id FROM org_members WHERE user_id = $3
         ))",
        opt_id,
        project_id,
        claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Optimization run not found"))?;

    Ok(Json(opt))
}

pub async fn cancel_optimization(
    State(state): State<AppState>,
    claims: Claims,
    Path((project_id, opt_id)): Path<(Uuid, Uuid)>,
) -> Result<StatusCode, ApiError> {
    // Update status to cancelled
    sqlx::query!(
        "UPDATE optimization_runs SET status = 'cancelled', completed_at = NOW()
         WHERE id = $1 AND project_id = $2 AND started_by = $3",
        opt_id,
        project_id,
        claims.user_id
    )
    .execute(&state.db)
    .await?;

    // Publish cancellation event for worker
    state.redis
        .publish("optimization:cancel", opt_id.to_string())
        .await?;

    Ok(StatusCode::NO_CONTENT)
}

fn estimate_optimization_time(resolution: i32, method: &str) -> f32 {
    // Rough estimates based on benchmarks
    let base_time = match method {
        "simp" => match resolution {
            64 => 2.0,   // 2 minutes
            128 => 5.0,  // 5 minutes
            256 => 15.0, // 15 minutes
            512 => 45.0, // 45 minutes
            _ => 2.0,
        },
        "level_set" => match resolution {
            64 => 4.0,
            128 => 10.0,
            256 => 30.0,
            512 => 90.0,
            _ => 4.0,
        },
        _ => 5.0,
    };
    base_time
}
```

### 2. GPU Topology Optimizer (Rust + wgpu)

The core SIMP optimizer implemented with wgpu compute shaders.

```rust
// optimizer/src/simp.rs

use wgpu::util::DeviceExt;
use anyhow::Result;

pub struct SimpOptimizer {
    device: wgpu::Device,
    queue: wgpu::Queue,
    resolution: [u32; 3],
    n_elements: usize,

    // GPU buffers
    density_buffer: wgpu::Buffer,
    sensitivity_buffer: wgpu::Buffer,
    displacement_buffer: wgpu::Buffer,
    stiffness_buffer: wgpu::Buffer,

    // Compute pipelines
    fea_pipeline: wgpu::ComputePipeline,
    sensitivity_pipeline: wgpu::ComputePipeline,
    filter_pipeline: wgpu::ComputePipeline,
    update_pipeline: wgpu::ComputePipeline,
}

impl SimpOptimizer {
    pub async fn new(
        resolution: [u32; 3],
        material: &Material,
        loads: &[Load],
        constraints: &[Constraint],
    ) -> Result<Self> {
        let instance = wgpu::Instance::new(wgpu::InstanceDescriptor::default());
        let adapter = instance
            .request_adapter(&wgpu::RequestAdapterOptions {
                power_preference: wgpu::PowerPreference::HighPerformance,
                ..Default::default()
            })
            .await
            .ok_or_else(|| anyhow::anyhow!("Failed to find GPU adapter"))?;

        let (device, queue) = adapter
            .request_device(
                &wgpu::DeviceDescriptor {
                    label: Some("SIMP Optimizer"),
                    required_features: wgpu::Features::TIMESTAMP_QUERY,
                    required_limits: wgpu::Limits::default(),
                },
                None,
            )
            .await?;

        let n_elements = (resolution[0] * resolution[1] * resolution[2]) as usize;

        // Initialize density field (uniform distribution at target volume fraction)
        let initial_density = vec![0.5_f32; n_elements];
        let density_buffer = device.create_buffer_init(&wgpu::util::BufferInitDescriptor {
            label: Some("Density Buffer"),
            contents: bytemuck::cast_slice(&initial_density),
            usage: wgpu::BufferUsages::STORAGE | wgpu::BufferUsages::COPY_SRC | wgpu::BufferUsages::COPY_DST,
        });

        let sensitivity_buffer = device.create_buffer(&wgpu::BufferDescriptor {
            label: Some("Sensitivity Buffer"),
            size: (n_elements * std::mem::size_of::<f32>()) as u64,
            usage: wgpu::BufferUsages::STORAGE | wgpu::BufferUsages::COPY_SRC,
            mapped_at_creation: false,
        });

        let displacement_buffer = device.create_buffer(&wgpu::BufferDescriptor {
            label: Some("Displacement Buffer"),
            size: (n_elements * 3 * std::mem::size_of::<f32>()) as u64,  // 3 DOF per node
            usage: wgpu::BufferUsages::STORAGE | wgpu::BufferUsages::COPY_SRC,
            mapped_at_creation: false,
        });

        // Build element stiffness matrices
        let stiffness_data = build_element_stiffness_matrices(material, resolution);
        let stiffness_buffer = device.create_buffer_init(&wgpu::util::BufferInitDescriptor {
            label: Some("Stiffness Buffer"),
            contents: bytemuck::cast_slice(&stiffness_data),
            usage: wgpu::BufferUsages::STORAGE,
        });

        // Load compute shaders
        let fea_shader = device.create_shader_module(wgpu::ShaderModuleDescriptor {
            label: Some("FEA Solver Shader"),
            source: wgpu::ShaderSource::Wgsl(include_str!("shaders/fea_pcg.wgsl").into()),
        });

        let sensitivity_shader = device.create_shader_module(wgpu::ShaderModuleDescriptor {
            label: Some("Sensitivity Shader"),
            source: wgpu::ShaderSource::Wgsl(include_str!("shaders/sensitivity.wgsl").into()),
        });

        let filter_shader = device.create_shader_module(wgpu::ShaderModuleDescriptor {
            label: Some("Filter Shader"),
            source: wgpu::ShaderSource::Wgsl(include_str!("shaders/density_filter.wgsl").into()),
        });

        let update_shader = device.create_shader_module(wgpu::ShaderModuleDescriptor {
            label: Some("Update Shader"),
            source: wgpu::ShaderSource::Wgsl(include_str!("shaders/density_update.wgsl").into()),
        });

        // Create compute pipelines (bind group layouts omitted for brevity)
        let fea_pipeline = device.create_compute_pipeline(&wgpu::ComputePipelineDescriptor {
            label: Some("FEA Pipeline"),
            layout: None,
            module: &fea_shader,
            entry_point: "main",
        });

        let sensitivity_pipeline = device.create_compute_pipeline(&wgpu::ComputePipelineDescriptor {
            label: Some("Sensitivity Pipeline"),
            layout: None,
            module: &sensitivity_shader,
            entry_point: "main",
        });

        let filter_pipeline = device.create_compute_pipeline(&wgpu::ComputePipelineDescriptor {
            label: Some("Filter Pipeline"),
            layout: None,
            module: &filter_shader,
            entry_point: "main",
        });

        let update_pipeline = device.create_compute_pipeline(&wgpu::ComputePipelineDescriptor {
            label: Some("Update Pipeline"),
            layout: None,
            module: &update_shader,
            entry_point: "main",
        });

        Ok(Self {
            device,
            queue,
            resolution,
            n_elements,
            density_buffer,
            sensitivity_buffer,
            displacement_buffer,
            stiffness_buffer,
            fea_pipeline,
            sensitivity_pipeline,
            filter_pipeline,
            update_pipeline,
        })
    }

    pub async fn optimize(
        &mut self,
        params: &OptimizationParams,
        progress_callback: impl Fn(OptimizationProgress),
    ) -> Result<OptimizationResult> {
        let max_iterations = 200;
        let tolerance = 0.01;  // 1% max density change

        for iter in 0..max_iterations {
            // 1. Solve FEA: K(ρ) U = F
            self.solve_fea().await?;

            // 2. Compute sensitivities: ∂C/∂ρ
            self.compute_sensitivities().await?;

            // 3. Apply density filter
            self.apply_density_filter(params.filter_radius_mm).await?;

            // 4. Update densities with OC method
            let (max_change, compliance) = self.update_densities(params.volume_fraction).await?;

            // 5. Check convergence
            progress_callback(OptimizationProgress {
                iteration: iter,
                compliance,
                max_change,
                volume_fraction: self.compute_volume_fraction().await?,
            });

            if max_change < tolerance {
                tracing::info!("Converged at iteration {}", iter);
                break;
            }
        }

        // Extract final mesh using marching cubes
        let density_data = self.read_density_buffer().await?;
        let mesh = self.extract_isosurface(&density_data, 0.5)?;

        Ok(OptimizationResult {
            mesh,
            density_field: density_data,
            final_compliance: self.compute_compliance().await?,
            iteration_count: max_iterations,
        })
    }

    async fn solve_fea(&mut self) -> Result<()> {
        let mut encoder = self.device.create_command_encoder(&wgpu::CommandEncoderDescriptor {
            label: Some("FEA Encoder"),
        });

        {
            let mut compute_pass = encoder.begin_compute_pass(&wgpu::ComputePassDescriptor {
                label: Some("FEA Pass"),
                timestamp_writes: None,
            });

            compute_pass.set_pipeline(&self.fea_pipeline);
            // Bind groups omitted for brevity
            let workgroup_count = (self.n_elements as u32 + 255) / 256;
            compute_pass.dispatch_workgroups(workgroup_count, 1, 1);
        }

        self.queue.submit(Some(encoder.finish()));
        self.device.poll(wgpu::Maintain::Wait);

        Ok(())
    }

    async fn compute_sensitivities(&mut self) -> Result<()> {
        let mut encoder = self.device.create_command_encoder(&Default::default());

        {
            let mut compute_pass = encoder.begin_compute_pass(&Default::default());
            compute_pass.set_pipeline(&self.sensitivity_pipeline);
            let workgroup_count = (self.n_elements as u32 + 255) / 256;
            compute_pass.dispatch_workgroups(workgroup_count, 1, 1);
        }

        self.queue.submit(Some(encoder.finish()));
        self.device.poll(wgpu::Maintain::Wait);

        Ok(())
    }

    async fn apply_density_filter(&mut self, filter_radius_mm: f32) -> Result<()> {
        // Apply convolution filter to density field
        // Implementation omitted for brevity
        Ok(())
    }

    async fn update_densities(&mut self, target_volume_fraction: f32) -> Result<(f32, f32)> {
        // Optimality Criteria update with binary search for Lagrange multiplier
        // Implementation omitted for brevity
        Ok((0.05, 1000.0))  // (max_change, compliance)
    }

    async fn read_density_buffer(&self) -> Result<Vec<f32>> {
        // Read density buffer back to CPU
        let buffer_slice = self.density_buffer.slice(..);
        let (tx, rx) = futures::channel::oneshot::channel();
        buffer_slice.map_async(wgpu::MapMode::Read, move |result| {
            tx.send(result).unwrap();
        });
        self.device.poll(wgpu::Maintain::Wait);
        rx.await??;

        let data = buffer_slice.get_mapped_range();
        let density: Vec<f32> = bytemuck::cast_slice(&data).to_vec();
        drop(data);
        self.density_buffer.unmap();

        Ok(density)
    }

    fn extract_isosurface(&self, density: &[f32], threshold: f32) -> Result<Mesh> {
        // Marching cubes algorithm to extract mesh at density threshold
        // Uses `marchingcubes` crate
        Ok(Mesh::default())
    }

    async fn compute_volume_fraction(&self) -> Result<f32> {
        let density = self.read_density_buffer().await?;
        Ok(density.iter().sum::<f32>() / density.len() as f32)
    }

    async fn compute_compliance(&self) -> Result<f32> {
        // C = Uᵀ K U
        Ok(1000.0)
    }
}

pub struct OptimizationProgress {
    pub iteration: usize,
    pub compliance: f32,
    pub max_change: f32,
    pub volume_fraction: f32,
}

pub struct OptimizationResult {
    pub mesh: Mesh,
    pub density_field: Vec<f32>,
    pub final_compliance: f32,
    pub iteration_count: usize,
}

struct Material {
    youngs_modulus_mpa: f32,
    poissons_ratio: f32,
}

struct Load {}
struct Constraint {}
struct Mesh {}

impl Default for Mesh {
    fn default() -> Self { Mesh {} }
}

fn build_element_stiffness_matrices(_material: &Material, _resolution: [u32; 3]) -> Vec<f32> {
    vec![]
}
```

### 3. FEA Solver (wgpu Compute Shader — PCG)

```rust
// optimizer/src/shaders/fea_pcg.wgsl

@group(0) @binding(0) var<storage, read> density: array<f32>;
@group(0) @binding(1) var<storage, read> stiffness: array<f32>;  // Element stiffness matrices
@group(0) @binding(2) var<storage, read> force: array<f32>;  // Load vector
@group(0) @binding(3) var<storage, read_write> displacement: array<f32>;  // Solution vector
@group(0) @binding(4) var<uniform> params: SolverParams;

struct SolverParams {
    n_dof: u32,
    n_elements: u32,
    penalty: f32,
    tolerance: f32,
    max_iterations: u32,
}

// Preconditioned Conjugate Gradient solver for K(ρ) U = F
@compute @workgroup_size(256)
fn main(@builtin(global_invocation_id) global_id: vec3<u32>) {
    let idx = global_id.x;
    if (idx >= params.n_dof) {
        return;
    }

    // PCG implementation
    // This is a simplified outline — full implementation requires multiple shader passes:
    // 1. Matrix-vector multiply: A * p
    // 2. Dot product: r^T r
    // 3. Vector update: x = x + alpha * p
    // 4. Residual update: r = r - alpha * A * p
    // 5. Preconditioner apply: z = M^-1 * r
    // 6. Beta computation and p update

    // Each iteration requires multiple dispatch calls orchestrated from CPU
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–5)

**Day 1: Project initialization**
```bash
cargo new printgen-api --bin
cd printgen-api
cargo add axum tokio serde serde_json sqlx uuid chrono tower-http jsonwebtoken bcrypt anyhow tracing tracing-subscriber
cargo add --features postgres,runtime-tokio-rustls,uuid,chrono,json sqlx
cargo add aws-sdk-s3 redis
```
- `src/main.rs` — Axum app with CORS, tracing, graceful shutdown
- `src/config.rs` — Environment configuration (DATABASE_URL, JWT_SECRET, S3_BUCKET, REDIS_URL, GPU_QUEUE_NAME)
- `src/state.rs` — AppState struct (PgPool, Redis, S3Client)
- `src/error.rs` — ApiError enum with IntoResponse implementation
- `Dockerfile` — Multi-stage build for Rust binary
- `docker-compose.yml` — PostgreSQL, Redis, MinIO (S3-compatible local dev)

**Day 2: Database schema and migrations**
- `migrations/001_initial.sql` — All 11 tables (users, organizations, org_members, projects, design_spaces, load_cases, materials, optimization_runs, fea_results, lattice_configs, comments, usage_records)
- `src/db/mod.rs` — Database pool initialization and connection management
- `src/db/models.rs` — All SQLx structs with FromRow and Serialize derives
- Material seeding script: 20+ built-in materials (PLA: E=3500 MPa, ρ=1.24 kg/m³; PETG, ABS, Nylon PA12, Tough Resin, PA12 SLS, 316L: E=193000 MPa, Ti-6Al-4V, AlSi10Mg)
- Run `sqlx migrate run` to apply schema

**Day 3: Authentication system**
- `src/auth/mod.rs` — JWT generation, validation middleware, Claims struct
- `src/auth/oauth.rs` — Google and GitHub OAuth 2.0 flow handlers (authorization URL, callback, token exchange)
- `src/api/handlers/auth.rs` — Register (email + password with bcrypt), login, OAuth callback, refresh token, get current user
- JWT structure: 24h access token + 30d refresh token with user_id and plan claims
- Auth middleware extracts Claims from `Authorization: Bearer <token>` header

**Day 4: User and organization CRUD**
- `src/api/handlers/users.rs` — Get profile, update profile (name, avatar), delete account
- `src/api/handlers/orgs.rs` — Create org, invite member (generate invite token), accept invite, list members, update member role, remove member
- Plan enforcement middleware: check plan tier for resolution limits and quota
- Integration tests: registration flow, OAuth flow, org creation and invitation

**Day 5: Project and design space CRUD**
- `src/api/handlers/projects.rs` — Create project, list projects (with filters: owner, org, public), get project, update project, delete project, fork project (deep copy)
- `src/api/handlers/design_spaces.rs` — Create design space (upload CAD file to S3), update regions (keep/exclude/symmetry), get design space with presigned mesh URL
- `src/api/router.rs` — All route definitions with auth middleware
- CAD file upload: multipart form handling, S3 upload with unique key, record URL in DB

### Phase 2 — CAD Processing Service (Python) (Days 6–8)

**Day 6: Python CAD processor service scaffold**
```bash
mkdir cad-processor && cd cad-processor
python -m venv venv && source venv/bin/activate
pip install fastapi uvicorn python-multipart boto3 pythonocc-core trimesh numpy
```
- `main.py` — FastAPI app with file upload endpoint
- `processors/step_import.py` — STEP file import using Open CASCADE (pythonocc), extract B-rep faces
- `processors/mesh_convert.py` — Convert B-rep to triangle mesh using meshing algorithm
- `processors/stl_process.py` — STL file processing: repair, compute bounding box, volume, surface area
- `processors/obj_process.py` — OBJ file processing
- S3 integration: download source file, upload processed mesh

**Day 7: Mesh repair and analysis**
- `processors/mesh_repair.py` — trimesh-based repair: remove duplicate vertices, fix normals, fill holes, remove degenerate triangles
- `processors/mesh_stats.py` — Compute mesh statistics: triangle count, vertex count, surface area (mm²), volume (mm³), bounding box dimensions
- Unit tests: process sample STL (Stanford Bunny), verify mesh quality metrics
- Error handling: invalid file format, corrupted mesh, excessive triangle count (>10M for free tier)

**Day 8: Integration with Rust API**
- Rust handler calls Python service via HTTP: POST `/process` with S3 URL
- Python service downloads, processes, uploads result mesh, returns metadata
- Rust handler updates `design_spaces` table with `mesh_url`, `bounding_box`, `mesh_stats`
- Async job pattern: enqueue processing job in Redis, worker polls and calls Python service
- Integration test: upload STEP file → verify processed STL appears in S3

### Phase 3 — GPU Optimizer Core (Rust + wgpu) (Days 9–18)

**Day 9: wgpu setup and GPU resource initialization**
```bash
cd printgen-api
cargo add wgpu bytemuck futures
mkdir optimizer && cd optimizer
cargo new --lib .
cargo add wgpu bytemuck anyhow nalgebra
```
- `optimizer/src/lib.rs` — Public API: `SimpOptimizer::new()`, `optimize()`
- `optimizer/src/gpu.rs` — GPU device initialization, adapter selection (prefer high-performance)
- `optimizer/src/buffers.rs` — Buffer creation helpers: density, sensitivity, displacement, stiffness
- Unit test: initialize GPU, create buffers, verify device supports compute shaders

**Day 10: Element stiffness matrix generation**
- `optimizer/src/element.rs` — 8-node hexahedral (voxel) element stiffness matrix using analytical formulation
- Material property interpolation: `K_elem = ρ^p * K0` where p=3 (SIMP penalty)
- Global stiffness matrix assembly: element connectivity for regular voxel grid
- Sparse matrix representation: COO (coordinate) format for GPU upload
- Unit test: verify element stiffness for single voxel matches analytical solution

**Day 11: FEA solver — GPU-accelerated PCG**
- `optimizer/src/fea/mod.rs` — FEA solver interface
- `optimizer/src/fea/pcg.rs` — Preconditioned Conjugate Gradient solver orchestration
- `optimizer/src/shaders/matvec.wgsl` — Sparse matrix-vector multiply shader
- `optimizer/src/shaders/dotprod.wgsl` — Parallel dot product with reduction
- `optimizer/src/shaders/vecops.wgsl` — Vector operations (axpy: y = a*x + y, scale, copy)
- PCG iteration loop on CPU, dispatching GPU shaders for each operation
- Unit test: solve simple cantilever beam (10x10x40 voxels), verify tip displacement

**Day 12: Preconditioner — Jacobi diagonal**
- `optimizer/src/fea/preconditioner.rs` — Jacobi preconditioner (diagonal inverse)
- `optimizer/src/shaders/jacobi.wgsl` — Compute diagonal of K(ρ) and invert
- Apply preconditioner: `z = M^-1 * r` where M = diag(K)
- Benchmark: compare PCG convergence with/without preconditioning (expect 2-3x speedup)

**Day 13: Sensitivity computation**
- `optimizer/src/sensitivity.rs` — Sensitivity analysis: ∂C/∂ρ = -p ρ^(p-1) u^T K0 u
- `optimizer/src/shaders/sensitivity.wgsl` — Parallel sensitivity computation per element
- Extract element displacement vector from global solution, compute element energy
- Unit test: verify sensitivity matches finite difference approximation

**Day 14: Density filtering**
- `optimizer/src/filter.rs` — Convolution filter for minimum feature size
- `optimizer/src/shaders/density_filter.wgsl` — 3D convolution with linear hat filter kernel
- Filter radius parameter controls minimum feature size (e.g., 2mm = 4 voxels at 0.5mm resolution)
- Boundary handling: zero-padding outside design domain
- Unit test: filter checkerboard pattern, verify smoothing

**Day 15: Density update — Optimality Criteria**
- `optimizer/src/update.rs` — OC update with Lagrange multiplier bisection search
- `optimizer/src/shaders/density_update.wgsl` — Apply OC update formula per element
- Volume constraint enforcement: binary search for λ such that Σ ρ = V_target
- Move limit and damping to ensure stability
- Unit test: verify volume fraction converges to target (e.g., 0.3)

**Day 16: SIMP optimizer integration**
- `optimizer/src/simp.rs` — Main optimizer loop integrating all components
- Iteration: FEA solve → sensitivity → filter → update → check convergence
- Convergence criterion: max(|ρ_new - ρ_old|) < 0.01
- Progress callback for iteration count, compliance, volume fraction
- Unit test: optimize MBB beam (classic benchmark), verify compliance reduction

**Day 17: Marching cubes mesh extraction**
```bash
cargo add marchingcubes
```
- `optimizer/src/mesh_extract.rs` — Marching cubes implementation
- Extract isosurface at density threshold ρ = 0.5
- Generate triangle mesh with vertex positions and normals
- Mesh cleanup: remove duplicate vertices, ensure manifold
- Unit test: extract mesh from sphere density field, verify topology

**Day 18: Integration test — full optimization pipeline**
- End-to-end test: cantilever beam optimization at 64³ resolution
- Verify: compliance decreases, volume fraction matches target, mesh export succeeds
- Benchmark: measure GPU time per iteration (target <0.5s for 64³, <3s for 128³)
- Save result meshes and density fields for manual inspection

### Phase 4 — Optimization Worker + WebSocket Streaming (Days 19–23)

**Day 19: GPU job worker setup**
- `src/workers/mod.rs` — Worker pool initialization
- `src/workers/optimization_worker.rs` — Redis job consumer, GPU job orchestration
- Job queue: sorted set in Redis with priority scores (enterprise=100, team=50, pro=10, free=0)
- Worker loop: pop highest-priority job, claim it (set worker_id), execute optimization
- Error handling: GPU OOM, convergence failure, timeout (30min for free, 2h for pro)

**Day 20: Optimization worker — job execution**
- Fetch design space mesh from S3, voxelize to density grid at specified resolution
- Fetch load cases and apply as boundary conditions (force vectors, fixed DOFs)
- Initialize `SimpOptimizer` with material properties, loads, constraints
- Run optimization with progress callback
- Upload result mesh and density field to S3
- Update `optimization_runs` table with status, results, compute time

**Day 21: WebSocket server for live updates**
- `src/api/ws/mod.rs` — Axum WebSocket upgrade handler at `/ws/optimization/:run_id`
- `src/api/ws/optimization_progress.rs` — Subscribe to Redis pub/sub channel for optimization updates
- Worker publishes progress messages: `{"iteration": 10, "compliance": 1234.5, "volume_fraction": 0.30, "max_change": 0.05}`
- Client receives JSON messages in real-time
- Connection management: heartbeat ping every 30s, auto-cleanup on disconnect

**Day 22: Density field streaming**
- Intermediate density field snapshots every 10 iterations
- Compress density field: run-length encoding or gzip for transmission
- WebSocket binary frame with density data
- Frontend decodes and updates Three.js voxel visualization
- Bandwidth optimization: stream only changed voxels (delta encoding)

**Day 23: Quota tracking and billing integration**
- Compute time tracking: measure GPU time per optimization run
- Update `users.compute_used_hours_current_period` after each run
- Monthly quota reset: cron job on 1st of month resets usage counters
- Usage API endpoints: GET /api/usage (current period stats)
- Stripe webhook: handle subscription changes, update plan and quota

### Phase 5 — Frontend 3D Viewport + Load Definition (Days 24–30)

**Day 24: Frontend scaffold and Three.js setup**
```bash
npm create vite@latest frontend -- --template react-ts
cd frontend
npm i three @react-three/fiber @react-three/drei zustand @tanstack/react-query axios
npm i -D @types/three
```
- `src/App.tsx` — Router (React Router), auth context, layout with sidebar
- `src/stores/projectStore.ts` — Zustand store for project and design space state
- `src/stores/optimizationStore.ts` — Optimization run state, convergence data
- `src/lib/api.ts` — Axios client with JWT auth interceptor
- `src/components/Layout.tsx` — Top nav, sidebar, main content area

**Day 25: 3D Viewport with mesh upload**
- `src/components/Viewport/Viewport.tsx` — Three.js canvas with orbit controls
- `src/components/Viewport/MeshRenderer.tsx` — Load and render STL/OBJ mesh
- `src/hooks/useSTLLoader.ts` — STL file loading via Three.js STLLoader
- Camera setup: perspective camera, orbit controls for pan/rotate/zoom
- Lighting: hemisphere light + directional light for good shading
- Grid and axes helper for spatial reference

**Day 26: Region painting tool**
- `src/components/Viewport/RegionPainter.tsx` — Interactive region marking
- Click face → highlight → add to keep/exclude region list
- Raycasting: detect face under cursor for selection
- Visual feedback: color overlay (green=keep, red=exclude, blue=optimize)
- Symmetry plane tool: drag to define plane, visualize with transparent plane mesh
- Undo/redo for region changes

**Day 27: Load and constraint definition**
- `src/components/Viewport/LoadTool.tsx` — Visual load application
- Load types: point force (arrow at vertex), surface pressure (distributed arrows on face), gravity (downward arrows on entire mesh)
- Click face/vertex → show magnitude input dialog → add load with 3D arrow visualization
- Arrow scaling: arrow length proportional to magnitude (with configurable scale factor)
- Constraint tool: fixed support (red dots), pinned (green dots), roller (blue directional constraint)
- Load/constraint list panel: edit magnitude, direction, delete

**Day 28: Optimization parameter UI**
- `src/components/OptimizationPanel.tsx` — Sidebar panel for optimization settings
- Material selector: dropdown with built-in materials, show properties (E, ρ, σ_y)
- Method selector: SIMP (default), Level-set (future)
- Objective: Minimize compliance (default), Minimize weight, Maximize frequency
- Parameters: volume fraction slider (10-90%), penalty (1.0-5.0), filter radius (0.5-5mm), min feature size, max overhang angle
- Resolution selector: 64³ (Free), 128³ (Pro), 256³ (Pro), 512³ (Team) with plan badge
- Estimated time and cost display

**Day 29: Optimization launch and progress view**
- `src/components/OptimizationLaunch.tsx` — "Optimize" button with quota check
- POST /api/projects/:id/optimize with all parameters
- Progress modal: WebSocket connection to `/ws/optimization/:run_id`
- Convergence plot: Recharts line chart with compliance vs. iteration
- Live density field visualization: voxel grid with color intensity = density
- Cancel button → POST /api/optimization-runs/:id/cancel

**Day 30: Real-time density visualization**
- `src/components/Viewport/DensityField.tsx` — Voxel rendering using Three.js instanced mesh
- Create InstancedMesh with cube geometry, one instance per voxel
- Update instance matrix and color based on density value (transparent=low, opaque=high)
- Color gradient: blue (ρ=0) → green (ρ=0.5) → yellow (ρ=1.0)
- Performance: use frustum culling and LOD for large grids (512³ = 134M voxels, render only surface shell)
- Smooth camera transition during optimization

### Phase 6 — FEA Validation + Results Visualization (Days 31–36)

**Day 31: FEA result computation worker**
- `src/workers/fea_worker.rs` — Background worker for FEA validation jobs
- After optimization completes, auto-enqueue FEA job
- Load optimized mesh, generate tetrahedral mesh using TetGen or Gmsh
- Apply loads and constraints from load case
- Solve linear static FEA using GPU PCG solver
- Compute stress (von Mises), displacement, safety factor = σ_yield / σ_max
- Upload stress/displacement fields to S3, update `fea_results` table

**Day 32: Stress visualization shader**
- `src/components/Viewport/StressVisualization.tsx` — Color-mapped stress overlay
- Load per-element stress field from S3
- Map stress to mesh faces using element-to-face mapping
- Custom Three.js shader: vertex colors interpolated from stress values
- Color scales: Rainbow (blue=0 → red=max), Blue-Red diverging, Grayscale
- Legend with min/max stress values and safety factor indicator

**Day 33: Displacement visualization**
- `src/components/Viewport/DisplacementVisualization.tsx` — Deformed shape overlay
- Load displacement field (per-node 3D vectors)
- Apply displacement to mesh vertex positions with exaggeration factor (1x to 100x)
- Animation: smoothly interpolate between original and deformed mesh
- Split-screen view: original (left) vs. deformed (right)
- Displacement magnitude color map (similar to stress)

**Day 34: FEA results panel**
- `src/components/FEAResultsPanel.tsx` — Summary table with key metrics
- Peak stress location: highlight critical element on mesh with marker
- Max displacement location and magnitude
- Safety factor: color-coded indicator (green >3, yellow 1.5-3, red <1.5)
- Probe tool: click any element → show exact stress/displacement values
- Export FEA report as PDF (future feature)

**Day 35: Section plane tool**
- `src/components/Viewport/SectionPlane.tsx` — Cut-away view for internal inspection
- Drag plane widget to position and orient section plane
- Clip mesh using Three.js clipping planes
- Visualize internal stress distribution on cut surface
- Multiple section planes: X, Y, Z axis-aligned or arbitrary orientation

**Day 36: Comparison view**
- `src/components/ComparisonView.tsx` — Side-by-side original vs. optimized
- Load two meshes: design space (original) and optimized result
- Overlay with transparency: original (gray translucent) + optimized (solid color)
- Metrics comparison table: weight reduction %, volume reduction %, stress increase, displacement change
- Export comparison screenshots

### Phase 7 — Print Preparation + Export (Days 37–39)

**Day 37: Print orientation optimizer**
- `optimizer/src/print/orientation.rs` — Brute-force orientation search
- Sample orientations: rotate mesh in 15° increments across euler angles (24 x 24 x 24 = 13,824 samples, subsample to 100-200 candidates)
- For each orientation: compute support volume using overhang angle analysis (faces with normal Z-component < cos(45°) need support)
- Score function: minimize support volume + maximize bottom surface area (for adhesion) + penalize overhang area
- Return best orientation (rotation matrix)
- Rust implementation: rayon parallel iteration over candidates
- Unit test: optimize orientation for L-bracket, verify support reduction

**Day 38: Print time and cost estimation**
- `optimizer/src/print/estimation.rs` — Slicing-free estimation algorithm
- Volume-based time estimate: `print_time = (volume_mm³ / flow_rate_mm³_per_s) + (layer_count * layer_change_time_s)`
- Layer count: `bounding_box_z_mm / layer_height_mm`
- Flow rate from printer profile: default 10 mm³/s for FDM, 5 mm³/s for SLA
- Material cost: `mass_kg * material_cost_per_kg` where `mass = volume * density`
- Support material: add 20-40% overhead based on support volume fraction
- Return: `{print_time_hours, material_weight_g, material_cost_usd, support_weight_g, total_cost_usd}`

**Day 39: Mesh export and slicer integration**
- `src/api/handlers/export.rs` — Export endpoints for STL, OBJ, 3MF
- STL export: binary format with proper normals, units in mm
- 3MF export: include material metadata, print settings, thumbnail
- Presigned S3 URLs for client download
- Slicer integration: generate `.prusa` config file for PrusaSlicer, `.curaproject` for Cura
- One-click export: package mesh + config in ZIP, download link
- Frontend: export modal with format selector, resolution options (low/medium/high triangle count via decimation)

### Phase 8 — Polish + Deployment (Days 40–42)

**Day 40: Performance optimization and caching**
- Redis caching: frequently accessed meshes, material library, optimization results (TTL 1 hour)
- CDN for static assets: Three.js bundles, material thumbnails
- Database query optimization: add missing indexes, use connection pooling (sqlx pool size 20)
- GPU worker autoscaling: Kubernetes HPA based on Redis queue depth (scale 1-10 GPU nodes)
- Frontend bundle optimization: code splitting, lazy loading for heavy components (Three.js viewer, waveform charts)
- Lighthouse audit: target >90 performance score on dashboard

**Day 41: Monitoring and observability**
- Prometheus metrics: API request latency, GPU utilization, optimization queue depth, job success/failure rate
- Grafana dashboards: system overview, optimization performance (avg time by resolution), billing metrics (compute hours consumed by plan tier)
- Sentry error tracking: backend errors, frontend exceptions, GPU worker crashes
- Structured logging: tracing spans for optimization runs, correlation IDs
- Alerting: PagerDuty for GPU node failures, quota exceeded warnings, convergence failure spike

**Day 42: Documentation and launch prep**
- User documentation: getting started guide, optimization tutorial (cantilever beam example), load definition guide, material selection guide
- API documentation: OpenAPI/Swagger spec, example requests
- Deployment automation: Terraform for AWS infrastructure (EKS cluster, RDS PostgreSQL, S3 buckets, Redis ElastiCache), GitHub Actions CD pipeline
- Security audit: dependency scanning (cargo audit, npm audit), secret rotation, rate limiting (10 req/min for free, 100 req/min for pro)
- Beta testing: invite 50 early users, collect feedback on UX and optimization quality
- Launch checklist: DNS setup, SSL certificates, Stripe webhook verification, backup/restore testing

---

## Validation Benchmarks

### Benchmark 1: MBB Beam Optimization (Compliance Minimization)

**Setup:**
- Design space: 60mm × 20mm × 10mm rectangular beam
- Loads: 1000N downward point force at center top
- Constraints: Fixed supports at bottom-left and bottom-right corners
- Material: PLA (E=3500 MPa, ν=0.35, ρ=1.24 kg/m³)
- Volume fraction: 0.40 (40% material fill)
- Resolution: 128³ voxels

**Expected Results:**
- Final compliance: 120-150 J (academic SIMP benchmarks report 130-145 J)
- Optimized weight: 9.9g (40% of 24.8g solid beam)
- Convergence: <100 iterations
- GPU time: 4-6 minutes on A10G
- Mesh extraction: 15,000-25,000 triangles
- Visual: characteristic arch structure connecting load point to supports

**Validation:** Compare compliance to TopOpt.jl (Julia) and top88.m (MATLAB 88-line code) results, expect within 5% agreement.

### Benchmark 2: Cantilever Beam with Stress Constraint

**Setup:**
- Design space: 100mm × 40mm × 20mm cantilever
- Loads: 500N downward force at free end
- Constraints: Fixed support on left face (all DOFs)
- Material: Nylon PA12 (E=1850 MPa, σ_y=48 MPa, ρ=1.01 kg/m³)
- Volume fraction: 0.30 (30% material fill)
- Resolution: 256³ voxels
- Objective: Minimize compliance with max stress ≤ 40 MPa

**Expected Results:**
- Final compliance: 250-300 J
- Max von Mises stress: 38-40 MPa (at fixed support)
- Optimized weight: 24.2g (30% of 80.8g solid)
- Safety factor: ≥1.2
- GPU time: 12-18 minutes on A10G
- Convergence: 120-150 iterations

**Validation:** Run FEA on extracted mesh, verify max stress matches optimizer's stress field within 10%, displacement at load point <15mm.

### Benchmark 3: L-Bracket Multi-Load Case

**Setup:**
- Design space: 80mm × 80mm × 20mm L-bracket (two perpendicular arms)
- Load case 1 (weight 0.6): 800N vertical force on arm 1 tip
- Load case 2 (weight 0.4): 600N horizontal force on arm 2 tip
- Constraints: Fixed support at corner intersection
- Material: AlSi10Mg (E=70000 MPa, σ_y=230 MPa, ρ=2.67 kg/m³)
- Volume fraction: 0.35
- Resolution: 128³ voxels

**Expected Results:**
- Final compliance (weighted): 180-220 J
- Optimized weight: 167g (35% of 477g solid)
- Max stress (load case 1): 180-200 MPa
- Max stress (load case 2): 150-170 MPa
- Safety factor: >1.15 for both load cases
- GPU time: 7-10 minutes on A10G

**Validation:** Compare to nTopology generative design (trial version) on same geometry, expect similar mass and stress distribution patterns.

### Benchmark 4: Print Orientation Optimization

**Setup:**
- Optimized cantilever beam mesh from Benchmark 2
- Printer: FDM with 45° overhang limit
- Support material cost: 1.5x base material cost
- Sample 200 random orientations

**Expected Results:**
- Optimal orientation: ~35-40° tilt from horizontal
- Support volume reduction: 60-75% vs. default horizontal orientation
- Support material: 3.2g (13% of part weight) vs. 8.1g (33%) horizontal
- Cost savings: $0.15 per part at $25/kg PLA
- Computation time: <5 seconds for 200 samples

**Validation:** Export to PrusaSlicer in optimal orientation, verify auto-generated support volume matches estimate within 20%.

### Benchmark 5: GPU Scalability and Cost Efficiency

**Setup:**
- Fixed geometry: 100mm cube design space, single point load, volume fraction 0.3
- Resolutions: 64³, 128³, 256³, 512³
- GPU: NVIDIA A10G (24GB VRAM)
- Measure: time per iteration, total optimization time, memory usage

**Expected Results:**

| Resolution | Elements | Time/Iter | Total Time | GPU Mem | Cost (A10G @ $1.20/h) |
|-----------|----------|-----------|------------|---------|----------------------|
| 64³       | 262K     | 0.3s      | 2.5 min    | 2 GB    | $0.05                |
| 128³      | 2.1M     | 1.2s      | 8 min      | 6 GB    | $0.16                |
| 256³      | 16.8M    | 5.5s      | 30 min     | 18 GB   | $0.60                |
| 512³      | 134M     | 28s       | 90 min     | 22 GB   | $1.80                |

**Scalability metric:** Time/iteration scales roughly O(n log n) due to PCG solver (sparse matrix operations), not O(n²).

**Validation:** Run benchmarks on A10G, verify memory usage stays within limits (512³ should not OOM on 24GB card), cost estimates match actual GPU-hour billing.

---

## Post-MVP Roadmap

### v1.1 — Lattice Generation (Weeks 11-13)

**Features:**
- Lattice unit cell library: Gyroid, Schwarz-P, Diamond, BCC, FCC, Octet-truss (6 types)
- Stress-graded density mapping: dense lattice in high-stress regions from FEA, sparse in low-stress
- Lattice-solid transition smoothing: blend between solid mounting features and lattice interior
- GPU-accelerated implicit surface evaluation for TPMS (Triply Periodic Minimal Surfaces)
- Lattice FEA: homogenized material properties for structural validation

**Technical Approach:**
- TPMS equation evaluation on GPU: `f(x,y,z) < threshold` defines unit cell
- Strut-based lattices: mesh generation from node-edge graph
- Density grading: scale cell size or strut diameter based on stress field scalar
- Mesh Boolean operations: subtract lattice from solid regions

**Validation Target:**
- 60-80% weight reduction vs. solid optimized geometry with <10% stiffness loss
- Lattice mesh generation for 256³ region in <30 seconds
- Successfully print gyroid lattice-filled bracket on Prusa MK4, verify no internal failures

### v1.2 — Advanced FEA and Multi-Physics (Weeks 14-17)

**Features:**
- Modal analysis: natural frequency extraction (first 10 modes), mode shape visualization
- Thermal analysis: steady-state heat conduction, temperature-dependent material properties
- Thermo-mechanical coupling: thermal expansion stress
- Fatigue analysis: rainflow cycle counting, S-N curve evaluation, fatigue life prediction
- Nonlinear FEA: geometric nonlinearity (large displacement), hyperelastic materials (TPU)

**Technical Approach:**
- Eigenvalue solver: ARPACK bindings for sparse matrices, GPU-accelerated matrix-vector multiply
- Thermal solve: similar PCG framework, different stiffness matrix (conductivity vs. elasticity)
- Fatigue: post-process stress history from transient FEA (future), currently static + load factor sweep
- Nonlinear: Newton-Raphson iteration at global level, update stiffness matrix per iteration

**Validation Target:**
- Modal analysis: first natural frequency within 5% of experimental (3D-printed beam vibration test)
- Thermal: temperature distribution within 10% of thermocouple measurements on heated plate

### v1.3 — Collaboration and API (Weeks 18-20)

**Features:**
- Real-time collaboration: multiple users editing same project with CRDT-based conflict resolution
- Design review mode: stakeholder comments on specific mesh regions, resolved/unresolved tracking
- Design version control: full history with diff visualization (overlay meshes)
- REST API for automation: submit optimization, poll status, download results
- Webhook notifications: optimization complete, FEA ready, quota warning
- CLI tool: `printgen optimize --design design.stl --loads loads.json --material PLA --resolution 256`

**Technical Approach:**
- Collaboration: WebSocket broadcast of design changes, Yjs CRDT for schematic state
- API: rate limiting (100 req/min for Pro, 1000 req/min for Enterprise), API key management
- CLI: Rust binary using clap, calls API endpoints, streams progress

**Validation Target:**
- Two users simultaneously edit load cases, both see changes within 500ms
- API batch optimization: submit 50 jobs, all complete within quota, success rate >95%

### v2.0 — Enterprise Features (Weeks 21-26)

**Features:**
- SAML/SSO: Okta, Azure AD, OneLogin integration
- STEP export: mesh-to-B-rep conversion for CAD roundtrip
- Custom material database: org-specific materials with validated properties
- White-label embedding: iframe widget for 3D printing service bureaus
- Audit logging: full activity trail for compliance (ISO 9001, AS9100)
- On-premise deployment: Kubernetes Helm chart, air-gapped installation option
- Advanced manufacturing constraints: draft angle for injection molding, accessibility for machining

**Technical Approach:**
- SAML: `saml2-rs` crate, metadata XML exchange with IdP
- STEP export: mesh simplification → B-spline surface fitting via Open CASCADE
- White-label: tenant-specific branding, custom domain (CNAME), embed token auth
- On-prem: Helm chart with embedded PostgreSQL, Redis, MinIO (no external cloud dependencies)

**Pricing:**
- Enterprise: $1,500/mo (10 seats) + $150/seat
- Custom SLA: 99.9% uptime, 4-hour response time, dedicated GPU capacity reservation

---

## Risk Mitigation

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| GPU optimization costs exceed revenue from low-tier plans | High | High | Enforce strict quota limits (Free: 2h/month caps at ~24 optimizations at 64³), upsell to Pro aggressively, spot instance scheduling for free tier (5-10 min queue delay acceptable) |
| Optimized designs fail structural validation in FEA | High | Medium | Make FEA validation mandatory before export, display safety factor prominently, add safety margin option (increase stresses by 1.5x in optimizer), conservative material properties |
| Users expect STEP export but mesh-to-CAD is unreliable | Medium | High | Set expectations: STEP export is "best-effort approximation", recommend STL for 3D printing (primary use case), offer OBJ/3MF as intermediate formats, defer full STEP to v2.0 Enterprise |
| GPU memory limits prevent high-resolution optimization | Medium | Medium | Implement out-of-core solver for 512³+ (page density field to CPU RAM), use tensor decomposition for sparse regions, fallback to CPU solver for extreme cases (1024³), charge premium for >512³ |
| Free tier abuse with compute-intensive jobs | Medium | High | Hard cap Free tier at 64³ resolution (no access to higher resolutions), CAPTCHA on optimization submit, IP-based rate limiting (max 10 optimizations/day), require email verification |
| Convergence failures on complex geometries | Medium | Medium | Implement continuation method (start at high penalty p=1.5, gradually increase to p=3), auto-retry with relaxed parameters on failure, detailed convergence diagnostics in error message, allow user to tweak filter radius and penalty |

---

## Tech Stack Summary

**Backend:**
- Rust 1.75+ (Axum 0.7, SQLx 0.7, wgpu 0.18, tokio 1.35)
- Python 3.12 (FastAPI 0.109, pythonocc-core 7.7, trimesh 4.0)
- PostgreSQL 16, Redis 7

**Frontend:**
- React 19, TypeScript 5.3, Vite 5
- Three.js r160, @react-three/fiber 8.15
- Zustand 4.5, TanStack Query 5.17

**Infrastructure:**
- AWS EKS (Kubernetes 1.28) with NVIDIA GPU Operator
- NVIDIA A10G GPU nodes (g5.xlarge instances, 24GB VRAM, 4 vCPU, 16GB RAM)
- RDS PostgreSQL, ElastiCache Redis, S3
- CloudFront CDN, Route 53 DNS, ACM SSL

**Monitoring:**
- Prometheus + Grafana (infrastructure, GPU utilization, optimization metrics)
- Sentry (error tracking)
- CloudWatch (AWS resource monitoring)
- Custom metrics: optimization convergence rate, FEA solve time, mesh extraction time

---

## Launch Checklist

**Week 10 (Final Integration):**
- [ ] End-to-end testing: user registration → project creation → design upload → optimization → FEA → export
- [ ] Load testing: 50 concurrent optimizations, verify queue processing, GPU autoscaling
- [ ] Security audit: pen testing, dependency scanning, rate limit validation
- [ ] Stripe integration testing: subscription lifecycle (create, upgrade, downgrade, cancel), webhook handling
- [ ] Quota enforcement: verify Free tier cannot exceed 2h compute, Pro caps at 50h
- [ ] Material library completeness: 20+ materials with validated properties
- [ ] Documentation: user guide, API reference, optimization best practices
- [ ] Beta user onboarding: 50 invites to target users (mechanical engineers, 3D printing enthusiasts, engineering consultants)

**Week 11 (Pre-Launch):**
- [ ] Production deployment: Terraform apply, database migrations, DNS cutover
- [ ] Monitoring dashboards: live system health, optimization queue depth, error rates
- [ ] Backup/restore testing: PostgreSQL automated backups, S3 versioning enabled
- [ ] Performance validation: Lighthouse score >90, API p95 latency <500ms, optimization time benchmarks pass
- [ ] Legal: Terms of Service, Privacy Policy, GDPR compliance (data export, deletion)
- [ ] Marketing site: landing page with demo video, pricing page, feature comparison table
- [ ] Product Hunt submission prep: screenshots, demo account, launch tagline

**Week 12 (Launch):**
- [ ] Soft launch: Product Hunt, Hacker News, r/3Dprinting, r/AdditiveManufacturing
- [ ] Monitor: error rates, signup flow dropoff, optimization success rate
- [ ] Support: Discord server for user questions, email support rotation
- [ ] Iterate: collect feedback, prioritize bug fixes, plan v1.1 lattice feature

**Success Metrics (Month 1):**
- 1,000+ registered users (Free tier)
- 100+ paying users (Pro/Team)
- 5,000+ optimization runs completed
- 500+ STL exports
- <5% optimization failure rate
- Average p95 optimization time <20 min for 128³ resolution
- NPS score >40 from beta users

---

## Cost Structure

**Infrastructure (Monthly @ 1,000 users, 50 paying):**
- EKS cluster: $75 (control plane) + $200 (worker nodes, 3x t3.medium)
- GPU nodes (on-demand): 0 baseline + $0.50/GPU-hour (billed per optimization)
  - Free tier: 1,000 users × 2h quota × 20% utilization = 400 GPU-hours = $480
  - Pro tier: 50 users × 50h quota × 40% utilization = 1,000 GPU-hours = $1,200
  - Total GPU: $1,680/month
- RDS PostgreSQL: $120 (db.t3.medium)
- ElastiCache Redis: $60 (cache.t3.micro)
- S3 storage: 500 GB × $0.023/GB = $11.50
- Data transfer: 1 TB × $0.09/GB = $90
- CloudFront: $50
- Total: ~$2,300/month

**Revenue (Month 1):**
- Pro: 40 users × $99 = $3,960
- Team: 10 orgs × $299 = $2,990
- Total MRR: $6,950

**Gross Margin:** ($6,950 - $2,300) / $6,950 = 67%

**Target (Month 6):** 200 Pro + 50 Team = $34,750 MRR, infrastructure scales to ~$8,000, gross margin 77%

---

**Total Implementation Timeline:** 42 days (8.4 weeks) with 1 full-time engineer
**MVP Launch Target:** Week 10 (includes 2 weeks integration, testing, deployment)
**Post-MVP Roadmap:** v1.1 Lattice (Weeks 11-13), v1.2 Advanced FEA (Weeks 14-17), v1.3 Collaboration (Weeks 18-20), v2.0 Enterprise (Weeks 21-26)
