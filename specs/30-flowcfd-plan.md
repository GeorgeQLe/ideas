# 30. FlowCFD — Cloud-Native Computational Fluid Dynamics with AI Meshing

## Implementation Plan

**MVP Scope:** Browser-based CFD platform with CAD import (STEP, STL via OpenCASCADE WASM) and Three.js 3D viewer with geometry healing/defeaturing, AI-powered automatic mesh generation via PyTorch CNN inference predicting optimal cell sizes and boundary layer parameters for hex-dominant meshing with cfMesh/snappyHexMesh, guided solver setup wizard for OpenFOAM incompressible steady-state with k-omega SST turbulence model and visual boundary condition assignment, cloud HPC job orchestration via Kubernetes with auto-scaling 4–64 cores running OpenFOAM solver containers, real-time convergence monitoring via WebSocket streaming residual plots and force coefficients, interactive 3D post-processing viewer using Three.js with WebGL contour plots, streamlines, cut planes, and isosurfaces rendered via VTK data pipeline, PDF report generation with auto-screenshots and mesh statistics, Stripe billing with three tiers (Free / Engineer $199/mo / Team $699/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| Mesh AI Inference | Python 3.12 (FastAPI) | PyTorch CNN model for mesh parameter prediction, ONNX Runtime |
| CAD Processing | OpenCASCADE (WASM) | STEP/IGES import via wasm-bindgen, geometry healing |
| CFD Solver | OpenFOAM v2312 | Incompressible solvers (simpleFoam, pimpleFoam), MPI parallelization |
| Meshing | cfMesh + snappyHexMesh | Hex-dominant with polyhedral cells, boundary layer insertion |
| Database | PostgreSQL 16 | Projects, jobs, users, mesh metadata, solver configs |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, async, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | CAD files, mesh files (OpenFOAM polyMesh), VTK results, multi-TB per project |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| 3D Viewer | Three.js (WebGL) | Progressive loading, level-of-detail for 50M+ cell meshes |
| Post-Processing | VTK 9.x + ParaView headless | Server-side field extraction, isosurface generation, streamline tracing |
| Real-time | WebSocket (Axum) | Live residual streaming, force monitoring, job status |
| Job Queue | Redis 7 + Tokio tasks | Kubernetes job submission, priority queues, solver progress pub/sub |
| Compute Cluster | Kubernetes (EKS) + Karpenter | Auto-scaling node pools (c6i/c7g instances), Slurm-like MPI scheduling |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks for plan changes |
| CDN | CloudFront | Static assets, WASM bundles, cached VTK data for 3D viewer |
| Monitoring | Prometheus + Grafana, Sentry | Job metrics, solver convergence rates, API latency, error tracking |
| CI/CD | GitHub Actions | Rust tests, Docker image builds, WASM compilation, Kubernetes deployment |

### Key Architecture Decisions

1. **Rust/Axum as primary backend with Python FastAPI microservice for ML inference**: The main API server handling auth, projects, job orchestration, and data management is built in Rust with Axum for type safety, performance, and async I/O efficiency. Python FastAPI runs as a separate microservice exclusively for AI mesh parameter prediction using PyTorch/ONNX, keeping the ML dependency isolated while maintaining a clean boundary between domains.

2. **Hybrid WASM + server-side CAD processing**: OpenCASCADE compiled to WASM enables client-side STEP/STL import for geometries <50MB, providing instant preview and basic healing without server round-trips. Larger or complex CAD files (>50MB, assemblies, IGES) are processed server-side by a Rust service using OpenCASCADE bindings, with presigned S3 URLs for upload/download to handle multi-GB files efficiently.

3. **AI mesh generation with PyTorch CNN predicting cfMesh parameters**: A 3D convolutional neural network trained on 50,000+ validated CFD meshes analyzes geometry features (curvature, gaps, boundary layers) and predicts optimal meshing parameters: global cell size, refinement region coordinates, boundary layer count/growth ratio/first cell height, y+ targets. These parameters feed directly into cfMesh/snappyHexMesh dictionaries, automating 80% of mesh setup while allowing manual override.

4. **Kubernetes-native OpenFOAM job orchestration with MPI auto-scaling**: Each CFD simulation runs as a Kubernetes Job with OpenFOAM solver containers (simpleFoam, pimpleFoam) using Intel MPI for parallelization. Karpenter auto-scales EC2 spot instances (c6i.8xlarge, c7g.16xlarge) based on job queue depth and requested core count (4–1024 cores). Jobs stream progress via sidecar containers publishing to Redis pub/sub, enabling WebSocket live updates to frontend.

5. **Three.js with progressive VTK loading for 50M+ cell visualization**: Server-side ParaView headless extracts VTK surface meshes and field data (pressure, velocity, temperature) with level-of-detail decimation. Large meshes transfer in chunks via presigned S3 URLs with spatial octree indexing, allowing Three.js to progressively load only visible geometry based on camera frustum. GPU-accelerated contour rendering uses vertex shaders for color mapping.

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
    plan TEXT NOT NULL DEFAULT 'free',
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    compute_credits_used BIGINT DEFAULT 0,
    compute_credits_limit BIGINT,
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
    plan TEXT NOT NULL DEFAULT 'team',
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    settings JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX orgs_slug_idx ON organizations(slug);

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

-- Projects
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    application_type TEXT,
    status TEXT NOT NULL DEFAULT 'active',
    settings JSONB DEFAULT '{}',
    thumbnail_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);
CREATE INDEX projects_org_idx ON projects(org_id);

-- Geometries
CREATE TABLE geometries (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    format TEXT NOT NULL,
    original_file_url TEXT NOT NULL,
    processed_file_url TEXT,
    bounding_box JSONB,
    surface_count INTEGER,
    volume DOUBLE PRECISION,
    surface_area DOUBLE PRECISION,
    surface_groups JSONB DEFAULT '[]',
    processing_status TEXT DEFAULT 'pending',
    error_message TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX geometries_project_idx ON geometries(project_id);

-- Meshes
CREATE TABLE meshes (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    geometry_id UUID NOT NULL REFERENCES geometries(id) ON DELETE CASCADE,
    name TEXT DEFAULT 'Mesh',
    method TEXT NOT NULL DEFAULT 'ai_auto',
    status TEXT NOT NULL DEFAULT 'pending',
    cell_count BIGINT,
    point_count BIGINT,
    mesh_file_url TEXT,
    quality_metrics JSONB,
    ai_parameters JSONB,
    manual_overrides JSONB,
    boundary_layer_config JSONB,
    generation_time_seconds INTEGER,
    error_message TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX meshes_project_idx ON meshes(project_id);

-- Simulation Cases
CREATE TABLE simulation_cases (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    mesh_id UUID NOT NULL REFERENCES meshes(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    physics JSONB NOT NULL,
    boundary_conditions JSONB NOT NULL,
    solver_settings JSONB NOT NULL,
    transient_config JSONB,
    material_properties JSONB NOT NULL,
    template_id UUID REFERENCES simulation_cases(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX cases_project_idx ON simulation_cases(project_id);

-- Simulation Jobs
CREATE TABLE simulation_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    case_id UUID NOT NULL REFERENCES simulation_cases(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    status TEXT NOT NULL DEFAULT 'queued',
    cores_allocated INTEGER NOT NULL,
    node_count INTEGER,
    priority INTEGER DEFAULT 0,
    k8s_job_name TEXT,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    wall_time_seconds INTEGER,
    core_seconds_used BIGINT,
    cost_credits DOUBLE PRECISION,
    residual_history JSONB DEFAULT '[]',
    force_history JSONB DEFAULT '[]',
    results_url TEXT,
    log_url TEXT,
    error_message TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX jobs_case_idx ON simulation_jobs(case_id);
CREATE INDEX jobs_status_idx ON simulation_jobs(status);

-- Post-Processing Views
CREATE TABLE postprocessing_views (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    job_id UUID NOT NULL REFERENCES simulation_jobs(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    view_type TEXT NOT NULL,
    config JSONB NOT NULL,
    vtk_data_url TEXT,
    screenshot_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX views_job_idx ON postprocessing_views(job_id);

-- Usage Tracking
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,
    quantity DOUBLE PRECISION NOT NULL,
    job_id UUID REFERENCES simulation_jobs(id),
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
    pub plan: String,
    pub compute_credits_used: i64,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Project {
    pub id: Uuid,
    pub owner_id: Uuid,
    pub name: String,
    pub description: String,
    pub application_type: Option<String>,
    pub status: String,
    pub settings: serde_json::Value,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Mesh {
    pub id: Uuid,
    pub project_id: Uuid,
    pub geometry_id: Uuid,
    pub name: String,
    pub method: String,
    pub status: String,
    pub cell_count: Option<i64>,
    pub quality_metrics: Option<serde_json::Value>,
    pub ai_parameters: Option<serde_json::Value>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct SimulationJob {
    pub id: Uuid,
    pub case_id: Uuid,
    pub user_id: Uuid,
    pub status: String,
    pub cores_allocated: i32,
    pub k8s_job_name: Option<String>,
    pub residual_history: serde_json::Value,
    pub force_history: serde_json::Value,
    pub results_url: Option<String>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct BoundaryCondition {
    pub surface_group: String,
    pub bc_type: BcType,
    pub values: serde_json::Value,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum BcType {
    VelocityInlet,
    PressureOutlet,
    Wall,
    Symmetry,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct PhysicsConfig {
    pub compressible: bool,
    pub turbulence_model: TurbulenceModel,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum TurbulenceModel {
    Laminar,
    KOmegaSST,
}
```

---

## Mesh AI Architecture

### AI Model Training Pipeline

3D Convolutional Neural Network trained on 50,000+ validated CFD meshes:

```
Input: Voxelized geometry (128³) + application type
    ↓
Conv3D(32) + BatchNorm + ReLU
Conv3D(64) + MaxPool(2×2×2)
Conv3D(128) + ReLU
Conv3D(256) + MaxPool(2×2×2)
    ↓
Flatten → Dense(512)
    ↓
Multi-head output:
  - Global cell size: Dense(1)
  - Refinement regions: Dense(8×7)
  - Boundary layer: Dense(4)
```

**Python FastAPI microservice:**

```python
# mesh-ai-service/predict.py

import torch
import onnxruntime as ort
from fastapi import FastAPI

app = FastAPI()
session = ort.InferenceSession("models/mesh_predictor.onnx")

@app.post("/predict-mesh-params")
async def predict(geometry_file, application_type: str, target_cells: int):
    voxel_grid = voxelize_geometry(geometry_file, resolution=128)

    outputs = session.run(None, {
        "voxel_grid": voxel_grid,
        "app_type": encode_app_type(application_type),
        "target_cells": [target_cells]
    })

    return {
        "global_cell_size": outputs[0][0],
        "refinement_regions": decode_refinements(outputs[1]),
        "boundary_layer": {
            "n_layers": int(outputs[2][0]),
            "growth_ratio": float(outputs[2][1]),
            "first_height": float(outputs[2][2])
        }
    }
```

**Rust backend integration:**

```rust
// src/services/mesh_ai.rs

use reqwest::Client;
use serde::{Deserialize, Serialize};

#[derive(Debug, Deserialize)]
pub struct MeshPrediction {
    pub global_cell_size: f64,
    pub refinement_regions: Vec<RefinementRegion>,
    pub boundary_layer: BoundaryLayerConfig,
}

pub async fn predict_mesh_parameters(
    client: &Client,
    geometry_url: &str,
    application_type: &str,
    target_cells: u64,
) -> anyhow::Result<MeshPrediction> {
    let mesh_ai_url = std::env::var("MESH_AI_SERVICE_URL")?;

    let response = client
        .post(format!("{}/predict-mesh-params", mesh_ai_url))
        .json(&serde_json::json!({
            "geometry_url": geometry_url,
            "application_type": application_type,
            "target_cell_count": target_cells
        }))
        .send()
        .await?;

    Ok(response.json().await?)
}
```

---

## OpenFOAM Integration

### cfMesh Dictionary Generation

```rust
// src/openfoam/cfmesh.rs

pub fn generate_mesh_dict(
    prediction: &MeshPrediction,
    geometry_stl_path: &str,
) -> String {
    format!(r#"FoamFile
{{
    version     2.0;
    format      ascii;
    class       dictionary;
    object      meshDict;
}}

surfaceFile "{}";
maxCellSize {:.6};

boundaryLayers
{{
    patchBoundaryLayers
    {{
        "wall"
        {{
            nLayers {};
            thicknessRatio {:.3};
            maxFirstLayerThickness {:.6};
        }}
    }}
}}

objectRefinements
{{
{}
}}
"#,
        geometry_stl_path,
        prediction.global_cell_size,
        prediction.boundary_layer.n_layers,
        prediction.boundary_layer.growth_ratio,
        prediction.boundary_layer.first_height,
        generate_refinement_objects(&prediction.refinement_regions)
    )
}

fn generate_refinement_objects(regions: &[RefinementRegion]) -> String {
    regions.iter().enumerate()
        .map(|(i, r)| format!(
            "    box{}\n    {{\n        type box;\n        centre ({} {} {});\n        cellSize {:.6};\n    }}",
            i, r.center[0], r.center[1], r.center[2], r.cell_size
        ))
        .collect::<Vec<_>>()
        .join("\n\n")
}
```

### Kubernetes Job Submission

```rust
// src/k8s/openfoam_job.rs

use k8s_openapi::api::batch::v1::Job;
use kube::{Api, Client};
use uuid::Uuid;

pub async fn submit_openfoam_job(
    k8s_client: &Client,
    job_id: Uuid,
    case_s3_url: &str,
    cores: i32,
) -> anyhow::Result<String> {
    let job_name = format!("cfd-{}", job_id);

    let container = serde_json::json!({
        "name": "openfoam-solver",
        "image": "flowcfd/openfoam:v2312",
        "command": ["/bin/bash", "-c"],
        "args": [format!(r#"
            aws s3 cp {} /tmp/case.tar.gz
            cd /tmp && tar -xzf case.tar.gz
            decomposePar -force
            mpirun -np {} simpleFoam -parallel
            reconstructPar -latestTime
            foamToVTK -latestTime
            tar -czf results.tar.gz VTK/ log.simpleFoam
            aws s3 cp results.tar.gz s3://flowcfd-results/jobs/{}/
        "#, case_s3_url, cores, job_id)],
        "resources": {
            "requests": {
                "cpu": cores.to_string(),
                "memory": format!("{}Gi", cores * 2)
            }
        }
    });

    let job: Job = serde_json::from_value(serde_json::json!({
        "apiVersion": "batch/v1",
        "kind": "Job",
        "metadata": {"name": job_name},
        "spec": {
            "template": {
                "spec": {
                    "containers": [container],
                    "restartPolicy": "Never"
                }
            }
        }
    }))?;

    let jobs: Api<Job> = Api::namespaced(k8s_client.clone(), "flowcfd");
    jobs.create(&Default::default(), &job).await?;

    Ok(job_name)
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–5)

**Day 1: Rust backend initialization**
```bash
cargo init flowcfd-api
cargo add axum tokio serde sqlx uuid chrono tower-http jsonwebtoken bcrypt
cargo add aws-sdk-s3 redis kube reqwest
```
- `src/main.rs` — Axum app setup
- `src/config.rs` — Environment config
- `src/state.rs` — AppState (PgPool, Redis, S3, K8s)
- `src/error.rs` — ApiError types
- `Dockerfile` — Multi-stage Rust build
- `docker-compose.yml` — Postgres, Redis, MinIO

**Day 2: Database schema**
- `migrations/001_initial.sql` — All tables
- `src/db/models.rs` — SQLx structs
- Run `sqlx migrate run`

**Day 3: Authentication**
- `src/auth/mod.rs` — JWT middleware
- `src/auth/oauth.rs` — Google/GitHub OAuth
- `src/api/handlers/auth.rs` — Auth endpoints
- Bcrypt password hashing
- JWT with 24h expiry

**Day 4: User and project CRUD**
- `src/api/handlers/users.rs` — Profile endpoints
- `src/api/handlers/orgs.rs` — Organization management
- `src/api/handlers/projects.rs` — Project CRUD
- `src/api/router.rs` — Route definitions

**Day 5: S3 integration**
- `src/services/s3.rs` — Presigned URL generation
- `src/api/handlers/upload.rs` — Upload endpoints
- Geometry file validation
- S3 lifecycle policies

### Phase 2 — CAD + Mesh AI (Days 6–12)

**Day 6: OpenCASCADE WASM**
- Build OCCT with Emscripten
- `cad-wasm/` — C++ bindings
- STEP/STL loading functions
- Frontend integration

**Day 7: Server-side CAD processing**
- `cad-service/` — Rust + OCCT bindings
- Geometry healing API
- Defeaturing tools
- Docker container

**Day 8: Python mesh AI service**
- `mesh-ai-service/` — FastAPI
- ONNX model loading
- `/predict-mesh-params` endpoint
- Geometry voxelization

**Day 9: Mesh AI integration**
- `src/services/mesh_ai.rs` — HTTP client
- `src/api/handlers/mesh.rs` — Mesh endpoints
- AI prediction storage
- Manual overrides

**Day 10: cfMesh dictionary generation**
- `src/openfoam/cfmesh.rs` — meshDict generation
- `src/openfoam/boundary_dict.rs` — Boundary file
- Template-based generation

**Day 11: Mesh generation worker**
- `mesh-worker/` — Docker with cfMesh
- Kubernetes Job submission
- S3 upload/download
- Quality metrics extraction

**Day 12: Mesh quality analysis**
- `src/openfoam/check_mesh.rs` — Parse checkMesh
- Quality scoring
- VTK preview extraction

### Phase 3 — Solver Setup (Days 13–18)

**Day 13: Solver wizard backend**
- `src/api/handlers/cases.rs` — Case CRUD
- `src/openfoam/case_generator.rs` — Directory structure
- Template system

**Day 14: fvSchemes and fvSolution**
- `src/openfoam/fv_schemes.rs` — Discretization schemes
- `src/openfoam/fv_solution.rs` — Solver settings
- Relaxation factors

**Day 15: Turbulence models**
- `src/openfoam/turbulence.rs` — Model files
- k-omega SST configuration
- Automatic initialization

**Day 16: Boundary conditions**
- `src/openfoam/boundary_conditions.rs` — Field files
- Inlet/outlet/wall types
- Time-varying BC support

**Day 17: Material properties**
- `src/openfoam/transport_properties.rs` — Fluid properties
- Material database
- Custom fluids

**Day 18: Case assembly**
- `src/openfoam/case_packager.rs` — Full case assembly
- Tar and S3 upload
- Validation checks

### Phase 4 — Job Orchestration (Days 19–24)

**Day 19: Kubernetes integration**
- `src/k8s/mod.rs` — K8s client setup
- `src/k8s/openfoam_job.rs` — Job submission
- MPI configuration

**Day 20: Job monitoring**
- `src/k8s/job_watcher.rs` — Job status polling
- Database updates
- Error handling

**Day 21: WebSocket progress streaming**
- `src/api/ws/mod.rs` — WebSocket upgrade
- `src/api/ws/simulation_progress.rs` — Progress channel
- Redis pub/sub integration

**Day 22: Residual parsing**
- `src/openfoam/residual_parser.rs` — Parse solver logs
- Extract residuals
- Force coefficients

**Day 23: Job lifecycle management**
- Pause/resume/cancel
- Resource cleanup
- Cost tracking

**Day 24: Auto-scaling configuration**
- Karpenter provisioner setup
- Node pool configuration
- Spot instance handling

### Phase 5 — Post-Processing (Days 25–29)

**Day 25: VTK extraction**
- `post-service/` — Python service
- ParaView headless
- Field extraction

**Day 26: Three.js viewer backend**
- Progressive loading API
- LOD decimation
- Octree spatial indexing

**Day 27: Frontend 3D viewer**
- `frontend/src/components/Viewer3D.tsx`
- WebGL contour rendering
- Camera controls

**Day 28: Streamlines and isosurfaces**
- Streamline seeding
- Q-criterion isosurfaces
- GPU shaders

**Day 29: Cut planes and probes**
- Arbitrary cut planes
- Point/line probes
- Export functionality

### Phase 6 — Billing + Limits (Days 30–33)

**Day 30: Stripe integration**
- `src/api/handlers/billing.rs`
- Checkout sessions
- Webhook handlers

**Day 31: Usage tracking**
- `src/services/usage.rs`
- Core-second accounting
- Plan limits

**Day 32: Billing UI**
- `frontend/src/pages/Billing.tsx`
- Plan comparison
- Usage meters

**Day 33: Feature gating**
- Plan enforcement
- Upgrade prompts
- Admin overrides

### Phase 7 — Validation (Days 34–38)

**Day 34: Solver validation — basic**
- Lid-driven cavity
- Backward-facing step
- Pipe flow

**Day 35: Solver validation — complex**
- Ahmed body drag
- NACA airfoil lift
- Building aerodynamics

**Day 36: Mesh quality validation**
- Orthogonality tests
- Skewness limits
- Convergence correlation

**Day 37: Integration testing**
- End-to-end workflow
- API tests
- WebSocket tests

**Day 38: Performance testing**
- Load tests (k6)
- Job throughput
- Frontend rendering

### Phase 8 — Deployment + Launch (Days 39–42)

**Day 39: Kubernetes deployment**
- `k8s/` manifests
- API deployment
- Worker deployment
- Health checks

**Day 40: CDN setup**
- CloudFront distribution
- WASM delivery
- Asset caching

**Day 41: Monitoring and logging**
- Prometheus metrics
- Grafana dashboards
- Sentry integration
- Structured logging

**Day 42: Launch preparation**
- Smoke tests
- Security audit
- Rate limiting
- Documentation
- Production deployment

---

## Critical Files

```
flowcfd/
├── flowcfd-api/                    # Rust Axum backend
│   ├── Cargo.toml
│   ├── src/
│   │   ├── main.rs
│   │   ├── config.rs
│   │   ├── state.rs
│   │   ├── error.rs
│   │   ├── auth/
│   │   │   ├── mod.rs
│   │   │   └── oauth.rs
│   │   ├── api/
│   │   │   ├── router.rs
│   │   │   ├── handlers/
│   │   │   │   ├── auth.rs
│   │   │   │   ├── projects.rs
│   │   │   │   ├── mesh.rs
│   │   │   │   ├── cases.rs
│   │   │   │   ├── jobs.rs
│   │   │   │   ├── billing.rs
│   │   │   │   └── upload.rs
│   │   │   └── ws/
│   │   │       ├── mod.rs
│   │   │       └── simulation_progress.rs
│   │   ├── db/
│   │   │   ├── mod.rs
│   │   │   └── models.rs
│   │   ├── services/
│   │   │   ├── mesh_ai.rs
│   │   │   ├── s3.rs
│   │   │   └── usage.rs
│   │   ├── k8s/
│   │   │   ├── mod.rs
│   │   │   ├── openfoam_job.rs
│   │   │   └── job_watcher.rs
│   │   └── openfoam/
│   │       ├── cfmesh.rs
│   │       ├── fv_schemes.rs
│   │       ├── fv_solution.rs
│   │       ├── turbulence.rs
│   │       ├── boundary_conditions.rs
│   │       ├── transport_properties.rs
│   │       ├── case_packager.rs
│   │       └── residual_parser.rs
│   ├── migrations/
│   │   └── 001_initial.sql
│   └── Dockerfile
│
├── mesh-ai-service/                # Python FastAPI
│   ├── main.py
│   ├── predict.py
│   ├── models/
│   │   └── mesh_predictor.onnx
│   ├── requirements.txt
│   └── Dockerfile
│
├── cad-wasm/                       # OpenCASCADE WASM
│   ├── occt_bindings.cpp
│   └── build.sh
│
├── frontend/                       # React TypeScript
│   ├── package.json
│   ├── src/
│   │   ├── App.tsx
│   │   ├── stores/
│   │   │   ├── authStore.ts
│   │   │   ├── projectStore.ts
│   │   │   └── viewerStore.ts
│   │   ├── hooks/
│   │   │   ├── useSimulationProgress.ts
│   │   │   └── useViewer3D.ts
│   │   ├── pages/
│   │   │   ├── Dashboard.tsx
│   │   │   ├── Editor.tsx
│   │   │   └── Billing.tsx
│   │   └── components/
│   │       ├── GeometryViewer/
│   │       ├── MeshWizard/
│   │       ├── SolverSetup/
│   │       ├── Viewer3D/
│   │       └── ResidualPlot/
│   └── Dockerfile
│
└── k8s/                            # Kubernetes manifests
    ├── api-deployment.yaml
    ├── worker-deployment.yaml
    └── ingress.yaml
```

---

## Success Metrics

| Metric | Target (Month 6) | Target (Month 12) |
|--------|------------------|-------------------|
| Registered users | 3,000 | 12,000 |
| Simulations completed | 5,000 | 30,000 |
| Core-hours consumed | 50,000 | 400,000 |
| Paying customers | 60 | 250 |
| MRR | $20,000 | $100,000 |
| Mesh generation time | <10 min | <5 min |
| Churn rate | <6% | <4% |

---

## Risk Mitigations

| Risk | Mitigation |
|------|-----------|
| AI mesh quality insufficient | Extensive validation, manual override, fallback to conservative defaults |
| Cloud compute costs too high | Spot instances (70% savings), intelligent auto-scaling, credit system |
| Engineers distrust cloud CFD | Publish validation studies, OpenFOAM case export, university partnerships |
| Large datasets slow browser | Progressive loading, LOD decimation, server-side ParaView fallback |
| ANSYS builds competing product | Move fast on AI meshing, strong community, no lock-in, undercut pricing |

---

## Architecture Deep-Dives

### 1. Simulation Job Orchestration (Rust/Axum + Kubernetes)

The simulation submission pipeline handles case preparation, Kubernetes job creation, progress monitoring, and result collection:

```rust
// src/api/handlers/jobs.rs

use axum::{extract::{Path, State}, Json};
use uuid::Uuid;
use crate::{
    db::models::SimulationJob,
    state::AppState,
    auth::Claims,
    error::ApiError,
    k8s::openfoam_job::submit_openfoam_job,
};

pub async fn submit_job(
    State(state): State<AppState>,
    claims: Claims,
    Path(case_id): Path<Uuid>,
) -> Result<Json<SimulationJob>, ApiError> {
    // 1. Fetch case and verify ownership
    let case = sqlx::query_as!(
        crate::db::models::SimulationCase,
        r#"SELECT c.* FROM simulation_cases c
           JOIN projects p ON c.project_id = p.id
           WHERE c.id = $1 AND (p.owner_id = $2 OR p.org_id IN (
               SELECT org_id FROM org_members WHERE user_id = $2
           ))"#,
        case_id,
        claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Case not found"))?;

    // 2. Check user plan limits
    let user = sqlx::query_as!(
        crate::db::models::User,
        "SELECT * FROM users WHERE id = $1",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    let cores = determine_cores(&user.plan, case.mesh_id, &state.db).await?;

    if user.plan == "free" && cores > 4 {
        return Err(ApiError::PlanLimit("Free plan limited to 4 cores"));
    }

    // 3. Generate OpenFOAM case directory
    let case_dir = crate::openfoam::case_packager::assemble_case(
        &state.db,
        &state.s3,
        &case
    ).await?;

    // 4. Upload case to S3
    let case_key = format!("cases/{}/case.tar.gz", case.id);
    state.s3
        .put_object()
        .bucket("flowcfd-cases")
        .key(&case_key)
        .body(case_dir.into())
        .send()
        .await?;

    let case_url = format!("s3://flowcfd-cases/{}", case_key);

    // 5. Create simulation job record
    let job = sqlx::query_as!(
        SimulationJob,
        r#"INSERT INTO simulation_jobs
           (case_id, user_id, cores_allocated, status)
           VALUES ($1, $2, $3, 'queued')
           RETURNING *"#,
        case.id,
        claims.user_id,
        cores
    )
    .fetch_one(&state.db)
    .await?;

    // 6. Submit Kubernetes Job
    let k8s_job_name = submit_openfoam_job(
        &state.k8s,
        job.id,
        &case_url,
        cores,
        "simpleFoam"
    ).await?;

    // 7. Update job with k8s name
    sqlx::query!(
        "UPDATE simulation_jobs SET k8s_job_name = $1, status = 'running', started_at = NOW()
         WHERE id = $2",
        k8s_job_name,
        job.id
    )
    .execute(&state.db)
    .await?;

    // 8. Start progress monitoring task
    tokio::spawn(monitor_job_progress(
        state.db.clone(),
        state.redis.clone(),
        job.id,
        k8s_job_name.clone()
    ));

    Ok(Json(job))
}

async fn determine_cores(
    plan: &str,
    mesh_id: Uuid,
    db: &sqlx::PgPool
) -> Result<i32, ApiError> {
    let mesh = sqlx::query!(
        "SELECT cell_count FROM meshes WHERE id = $1",
        mesh_id
    )
    .fetch_one(db)
    .await?;

    let cell_count = mesh.cell_count.unwrap_or(0);

    // Auto-scale cores based on mesh size
    let cores = match cell_count {
        0..=500_000 => 4,
        500_001..=2_000_000 => 8,
        2_000_001..=10_000_000 => 16,
        10_000_001..=50_000_000 => 32,
        _ => 64,
    };

    // Enforce plan limits
    let max_cores = match plan {
        "free" => 4,
        "engineer" => 64,
        "team" => 256,
        _ => 4,
    };

    Ok(cores.min(max_cores))
}

async fn monitor_job_progress(
    db: sqlx::PgPool,
    redis: redis::Client,
    job_id: Uuid,
    k8s_job_name: String,
) {
    let mut interval = tokio::time::interval(std::time::Duration::from_secs(10));

    loop {
        interval.tick().await;

        // Check if job still exists and is running
        let status = sqlx::query_scalar!(
            "SELECT status FROM simulation_jobs WHERE id = $1",
            job_id
        )
        .fetch_optional(&db)
        .await;

        match status {
            Ok(Some(s)) if s == "running" => {
                // Continue monitoring
            }
            _ => {
                // Job completed or cancelled
                break;
            }
        }

        // TODO: Fetch logs from K8s, parse residuals, publish to Redis
    }
}
```

### 2. Real-Time Residual Streaming (WebSocket + Redis Pub/Sub)

Progress updates flow from OpenFOAM container → Redis → WebSocket → Frontend:

```rust
// src/api/ws/simulation_progress.rs

use axum::{
    extract::{ws::WebSocket, WebSocketUpgrade, State, Path},
    response::Response,
};
use futures::{sink::SinkExt, stream::StreamExt};
use redis::AsyncCommands;
use uuid::Uuid;

pub async fn ws_handler(
    ws: WebSocketUpgrade,
    State(state): State<crate::state::AppState>,
    Path(job_id): Path<Uuid>,
) -> Response {
    ws.on_upgrade(move |socket| handle_socket(socket, state, job_id))
}

async fn handle_socket(
    socket: WebSocket,
    state: crate::state::AppState,
    job_id: Uuid,
) {
    let (mut sender, mut receiver) = socket.split();

    // Subscribe to Redis pub/sub channel for this job
    let mut pubsub = state.redis
        .get_async_connection()
        .await
        .unwrap()
        .into_pubsub();

    let channel = format!("job:{}:progress", job_id);
    pubsub.subscribe(&channel).await.unwrap();

    let mut pubsub_stream = pubsub.on_message();

    // Send initial state
    let job = sqlx::query!(
        r#"SELECT status, residual_history, force_history
           FROM simulation_jobs WHERE id = $1"#,
        job_id
    )
    .fetch_optional(&state.db)
    .await
    .unwrap();

    if let Some(job) = job {
        let initial_msg = serde_json::json!({
            "type": "initial",
            "status": job.status,
            "residuals": job.residual_history,
            "forces": job.force_history
        });
        sender.send(axum::extract::ws::Message::Text(
            serde_json::to_string(&initial_msg).unwrap()
        )).await.ok();
    }

    // Forward Redis messages to WebSocket
    loop {
        tokio::select! {
            Some(msg) = pubsub_stream.next() => {
                let payload: String = msg.get_payload().unwrap();
                sender.send(axum::extract::ws::Message::Text(payload)).await.ok();
            }
            Some(msg) = receiver.next() => {
                match msg {
                    Ok(axum::extract::ws::Message::Close(_)) => break,
                    Ok(axum::extract::ws::Message::Ping(data)) => {
                        sender.send(axum::extract::ws::Message::Pong(data)).await.ok();
                    }
                    _ => {}
                }
            }
            else => break,
        }
    }
}
```

**Progress sidecar container** (runs alongside OpenFOAM solver):

```rust
// progress-streamer/src/main.rs

use std::fs::File;
use std::io::{BufRead, BufReader};
use redis::AsyncCommands;
use regex::Regex;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let job_id = std::env::var("JOB_ID")?;
    let log_path = std::env::var("LOG_PATH").unwrap_or_else(|_| "/tmp/case/log.simpleFoam".to_string());
    let redis_url = std::env::var("REDIS_URL")?;

    let client = redis::Client::open(redis_url)?;
    let mut con = client.get_async_connection().await?;

    let channel = format!("job:{}:progress", job_id);

    // OpenFOAM residual pattern: "Time = 100  Solving for Ux, Initial residual = 0.01234"
    let residual_re = Regex::new(
        r"Time = (\d+)\s+Solving for (\w+), Initial residual = ([\d.e+-]+)"
    )?;

    // Force coefficient pattern: "Cd = 0.123, Cl = 0.456"
    let force_re = Regex::new(r"Cd = ([\d.e+-]+), Cl = ([\d.e+-]+)")?;

    let file = File::open(&log_path)?;
    let reader = BufReader::new(file);

    let mut current_iteration = 0;
    let mut current_residuals = std::collections::HashMap::new();

    for line in reader.lines() {
        let line = line?;

        if let Some(caps) = residual_re.captures(&line) {
            let iteration: u32 = caps[1].parse()?;
            let field = &caps[2];
            let residual: f64 = caps[3].parse()?;

            if iteration != current_iteration {
                // New iteration - publish accumulated data
                if !current_residuals.is_empty() {
                    let msg = serde_json::json!({
                        "type": "residuals",
                        "iteration": current_iteration,
                        "residuals": current_residuals,
                        "timestamp": chrono::Utc::now().to_rfc3339()
                    });

                    con.publish(&channel, serde_json::to_string(&msg)?).await?;
                    current_residuals.clear();
                }
                current_iteration = iteration;
            }

            current_residuals.insert(field.to_string(), residual);
        }

        if let Some(caps) = force_re.captures(&line) {
            let cd: f64 = caps[1].parse()?;
            let cl: f64 = caps[2].parse()?;

            let msg = serde_json::json!({
                "type": "forces",
                "iteration": current_iteration,
                "drag": cd,
                "lift": cl
            });

            con.publish(&channel, serde_json::to_string(&msg)?).await?;
        }

        tokio::time::sleep(tokio::time::Duration::from_millis(100)).await;
    }

    Ok(())
}
```

### 3. Three.js Post-Processing Viewer (Frontend)

Interactive 3D viewer with WebGL-accelerated rendering:

```typescript
// frontend/src/components/Viewer3D/Viewer3D.tsx

import { useEffect, useRef, useState } from 'react';
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';
import { useViewerStore } from '../../stores/viewerStore';

export function Viewer3D() {
  const canvasRef = useRef<HTMLCanvasElement>(null);
  const sceneRef = useRef<THREE.Scene | null>(null);
  const rendererRef = useRef<THREE.WebGLRenderer | null>(null);
  const { jobId, field, colormap, range } = useViewerStore();

  useEffect(() => {
    if (!canvasRef.current) return;

    // Initialize Three.js scene
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x1a1a1a);
    sceneRef.current = scene;

    const camera = new THREE.PerspectiveCamera(
      60,
      canvasRef.current.clientWidth / canvasRef.current.clientHeight,
      0.1,
      1000
    );
    camera.position.set(5, 5, 5);

    const renderer = new THREE.WebGLRenderer({
      canvas: canvasRef.current,
      antialias: true,
    });
    renderer.setSize(canvasRef.current.clientWidth, canvasRef.current.clientHeight);
    renderer.setPixelRatio(window.devicePixelRatio);
    rendererRef.current = renderer;

    const controls = new OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true;

    // Lighting
    const ambientLight = new THREE.AmbientLight(0xffffff, 0.6);
    scene.add(ambientLight);
    const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8);
    directionalLight.position.set(10, 10, 10);
    scene.add(directionalLight);

    // Animation loop
    function animate() {
      requestAnimationFrame(animate);
      controls.update();
      renderer.render(scene, camera);
    }
    animate();

    // Handle resize
    const handleResize = () => {
      if (!canvasRef.current) return;
      camera.aspect = canvasRef.current.clientWidth / canvasRef.current.clientHeight;
      camera.updateProjectionMatrix();
      renderer.setSize(canvasRef.current.clientWidth, canvasRef.current.clientHeight);
    };
    window.addEventListener('resize', handleResize);

    return () => {
      window.removeEventListener('resize', handleResize);
      renderer.dispose();
    };
  }, []);

  useEffect(() => {
    if (!jobId || !sceneRef.current) return;

    loadVTKData(jobId, field).then((geometry) => {
      if (!sceneRef.current) return;

      // Remove previous mesh
      const oldMesh = sceneRef.current.getObjectByName('cfd-mesh');
      if (oldMesh) sceneRef.current.remove(oldMesh);

      // Create material with vertex colors for field data
      const material = new THREE.MeshPhongMaterial({
        vertexColors: true,
        side: THREE.DoubleSide,
      });

      const mesh = new THREE.Mesh(geometry, material);
      mesh.name = 'cfd-mesh';
      sceneRef.current.add(mesh);
    });
  }, [jobId, field, colormap, range]);

  return (
    <div className="viewer3d-container relative w-full h-full">
      <canvas ref={canvasRef} className="w-full h-full" />
      <ViewerToolbar />
      <ColorLegend field={field} range={range} colormap={colormap} />
    </div>
  );
}

async function loadVTKData(
  jobId: string,
  field: string
): Promise<THREE.BufferGeometry> {
  // Fetch VTK data from API
  const response = await fetch(`/api/jobs/${jobId}/vtk-data?field=${field}`);
  const arrayBuffer = await response.arrayBuffer();

  // Parse VTK binary format
  const dataView = new DataView(arrayBuffer);
  let offset = 0;

  // Read header (simplified)
  const vertexCount = dataView.getUint32(offset, true);
  offset += 4;
  const faceCount = dataView.getUint32(offset, true);
  offset += 4;

  // Read vertices
  const vertices = new Float32Array(vertexCount * 3);
  for (let i = 0; i < vertexCount * 3; i++) {
    vertices[i] = dataView.getFloat32(offset, true);
    offset += 4;
  }

  // Read faces (triangles)
  const indices = new Uint32Array(faceCount * 3);
  for (let i = 0; i < faceCount * 3; i++) {
    indices[i] = dataView.getUint32(offset, true);
    offset += 4;
  }

  // Read field data
  const fieldData = new Float32Array(vertexCount);
  for (let i = 0; i < vertexCount; i++) {
    fieldData[i] = dataView.getFloat32(offset, true);
    offset += 4;
  }

  // Create geometry
  const geometry = new THREE.BufferGeometry();
  geometry.setAttribute('position', new THREE.BufferAttribute(vertices, 3));
  geometry.setIndex(new THREE.BufferAttribute(indices, 1));

  // Map field data to colors
  const colors = new Float32Array(vertexCount * 3);
  const colormap = getColormap('viridis');
  const [minVal, maxVal] = [Math.min(...fieldData), Math.max(...fieldData)];

  for (let i = 0; i < vertexCount; i++) {
    const normalized = (fieldData[i] - minVal) / (maxVal - minVal);
    const color = colormap(normalized);
    colors[i * 3] = color.r;
    colors[i * 3 + 1] = color.g;
    colors[i * 3 + 2] = color.b;
  }

  geometry.setAttribute('color', new THREE.BufferAttribute(colors, 3));
  geometry.computeVertexNormals();

  return geometry;
}

function getColormap(name: string): (t: number) => { r: number; g: number; b: number } {
  // Viridis colormap implementation
  return (t: number) => {
    t = Math.max(0, Math.min(1, t));
    const r = 0.267 + t * (0.329 - 0.267);
    const g = 0.005 + t * (0.993 - 0.005);
    const b = 0.329 + t * (0.380 - 0.329);
    return { r, g, b };
  };
}
```

### 4. VTK Data Extraction Service (Python + ParaView)

Server-side service for extracting and decimating VTK data:

```python
# post-service/vtk_extractor.py

from paraview.simple import *
import numpy as np
import struct

def extract_surface_with_field(
    openfoam_case_path: str,
    field: str,
    time_step: float = None,
    decimate_target: int = 1_000_000
) -> bytes:
    """
    Extract surface mesh with scalar field from OpenFOAM case.
    Returns binary VTK data suitable for Three.js rendering.
    """
    # Load OpenFOAM case
    reader = OpenFOAMReader(FileName=f"{openfoam_case_path}/case.foam")
    reader.MeshRegions = ['internalMesh']
    reader.CellArrays = [field]

    if time_step is not None:
        reader.UpdatePipeline(time_step)
    else:
        reader.UpdatePipeline()

    # Extract surface
    extract_surface = ExtractSurface(Input=reader)
    extract_surface.UpdatePipeline()

    # Decimate if too many cells
    current_cells = extract_surface.GetDataInformation().GetNumberOfCells()
    if current_cells > decimate_target:
        decimate = Decimate(Input=extract_surface)
        decimate.TargetReduction = 1.0 - (decimate_target / current_cells)
        decimate.UpdatePipeline()
        final_data = decimate
    else:
        final_data = extract_surface

    # Get VTK data
    vtk_data = servermanager.Fetch(final_data)

    # Convert to binary format
    points = vtk_data.GetPoints()
    num_points = points.GetNumberOfPoints()

    cells = vtk_data.GetPolys()
    num_cells = cells.GetNumberOfCells()

    field_array = vtk_data.GetPointData().GetArray(field)

    # Pack into binary: header + vertices + faces + field data
    output = bytearray()

    # Header
    output.extend(struct.pack('<I', num_points))  # vertex count
    output.extend(struct.pack('<I', num_cells))   # face count

    # Vertices
    for i in range(num_points):
        p = points.GetPoint(i)
        output.extend(struct.pack('<fff', p[0], p[1], p[2]))

    # Faces (assume triangles)
    cells.InitTraversal()
    id_list = vtk.vtkIdList()
    for i in range(num_cells):
        cells.GetNextCell(id_list)
        for j in range(id_list.GetNumberOfIds()):
            output.extend(struct.pack('<I', id_list.GetId(j)))

    # Field data
    for i in range(num_points):
        output.extend(struct.pack('<f', field_array.GetValue(i)))

    return bytes(output)
```

**Rust API endpoint calling Python service:**

```rust
// src/api/handlers/vtk.rs

use axum::{extract::{Path, Query, State}, Json};
use uuid::Uuid;
use serde::Deserialize;

#[derive(Deserialize)]
pub struct VtkQuery {
    field: String,
    time_step: Option<f64>,
}

pub async fn get_vtk_data(
    State(state): State<crate::state::AppState>,
    Path(job_id): Path<Uuid>,
    Query(params): Query<VtkQuery>,
) -> Result<Vec<u8>, crate::error::ApiError> {
    // Fetch job
    let job = sqlx::query!(
        "SELECT results_url FROM simulation_jobs WHERE id = $1",
        job_id
    )
    .fetch_one(&state.db)
    .await?;

    let results_url = job.results_url
        .ok_or(crate::error::ApiError::NotFound("Results not ready"))?;

    // Call Python VTK extraction service
    let vtk_service_url = std::env::var("VTK_SERVICE_URL")?;

    let response = state.http_client
        .post(format!("{}/extract-vtk", vtk_service_url))
        .json(&serde_json::json!({
            "case_url": results_url,
            "field": params.field,
            "time_step": params.time_step,
            "decimate_target": 1_000_000
        }))
        .send()
        .await?;

    if !response.status().is_success() {
        return Err(crate::error::ApiError::Internal("VTK extraction failed".to_string()));
    }

    let binary_data = response.bytes().await?;
    Ok(binary_data.to_vec())
}
```

---

## Deployment Architecture

### Kubernetes Cluster Layout

```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: flowcfd

---
# k8s/api-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flowcfd-api
  namespace: flowcfd
spec:
  replicas: 3
  selector:
    matchLabels:
      app: flowcfd-api
  template:
    metadata:
      labels:
        app: flowcfd-api
    spec:
      containers:
      - name: api
        image: flowcfd/api:latest
        ports:
        - containerPort: 8000
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: flowcfd-secrets
              key: database-url
        - name: REDIS_URL
          value: "redis://flowcfd-redis:6379"
        - name: S3_BUCKET
          value: "flowcfd-data"
        - name: MESH_AI_SERVICE_URL
          value: "http://mesh-ai-service:8001"
        - name: VTK_SERVICE_URL
          value: "http://vtk-service:8002"
        resources:
          requests:
            cpu: "500m"
            memory: "1Gi"
          limits:
            cpu: "2000m"
            memory: "4Gi"
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 10

---
# k8s/mesh-ai-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mesh-ai-service
  namespace: flowcfd
spec:
  replicas: 2
  selector:
    matchLabels:
      app: mesh-ai
  template:
    metadata:
      labels:
        app: mesh-ai
    spec:
      containers:
      - name: mesh-ai
        image: flowcfd/mesh-ai:latest
        ports:
        - containerPort: 8001
        resources:
          requests:
            cpu: "2000m"
            memory: "4Gi"
          limits:
            cpu: "4000m"
            memory: "8Gi"

---
# k8s/karpenter-provisioner.yaml
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: openfoam-compute
  namespace: flowcfd
spec:
  requirements:
    - key: karpenter.sh/capacity-type
      operator: In
      values: ["spot"]
    - key: node.kubernetes.io/instance-type
      operator: In
      values: ["c6i.8xlarge", "c6i.16xlarge", "c7g.8xlarge", "c7g.16xlarge"]
  limits:
    resources:
      cpu: 2048
  providerRef:
    name: default
  ttlSecondsAfterEmpty: 300
  ttlSecondsUntilExpired: 604800
  labels:
    workload-type: cfd-compute

---
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: flowcfd-api-hpa
  namespace: flowcfd
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: flowcfd-api
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

### Monitoring Stack

```yaml
# k8s/prometheus-servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: flowcfd-metrics
  namespace: flowcfd
spec:
  selector:
    matchLabels:
      app: flowcfd-api
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
```

**Metrics exported from Rust API:**

```rust
// src/metrics.rs

use prometheus::{
    register_histogram_vec, register_int_counter_vec, register_int_gauge_vec,
    HistogramVec, IntCounterVec, IntGaugeVec,
};
use once_cell::sync::Lazy;

pub static HTTP_REQUESTS_TOTAL: Lazy<IntCounterVec> = Lazy::new(|| {
    register_int_counter_vec!(
        "http_requests_total",
        "Total HTTP requests",
        &["method", "path", "status"]
    )
    .unwrap()
});

pub static HTTP_REQUEST_DURATION: Lazy<HistogramVec> = Lazy::new(|| {
    register_histogram_vec!(
        "http_request_duration_seconds",
        "HTTP request duration",
        &["method", "path"]
    )
    .unwrap()
});

pub static ACTIVE_SIMULATIONS: Lazy<IntGaugeVec> = Lazy::new(|| {
    register_int_gauge_vec!(
        "active_simulations",
        "Number of active simulations",
        &["status"]
    )
    .unwrap()
});

pub static MESH_GENERATION_DURATION: Lazy<HistogramVec> = Lazy::new(|| {
    register_histogram_vec!(
        "mesh_generation_duration_seconds",
        "Mesh generation duration",
        &["method"]
    )
    .unwrap()
});

pub static SOLVER_CONVERGENCE_ITERATIONS: Lazy<HistogramVec> = Lazy::new(|| {
    register_histogram_vec!(
        "solver_convergence_iterations",
        "Iterations to convergence",
        &["solver"]
    )
    .unwrap()
});
```

---

## Testing Strategy

### Unit Tests (Rust)

```rust
// src/openfoam/cfmesh_tests.rs

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_mesh_dict_generation() {
        let prediction = MeshPrediction {
            global_cell_size: 0.05,
            refinement_regions: vec![
                RefinementRegion {
                    shape: "box".to_string(),
                    center: [0.0, 0.0, 0.0],
                    dimensions: [1.0, 1.0, 1.0],
                    cell_size: 0.01,
                }
            ],
            boundary_layer: BoundaryLayerConfig {
                n_layers: 5,
                growth_ratio: 1.2,
                first_height: 0.001,
            },
        };

        let dict = generate_mesh_dict(&prediction, "geometry.stl");

        assert!(dict.contains("maxCellSize 0.050000"));
        assert!(dict.contains("nLayers 5"));
        assert!(dict.contains("thicknessRatio 1.200"));
        assert!(dict.contains("box0"));
    }

    #[test]
    fn test_boundary_conditions_validation() {
        let bc = BoundaryCondition {
            surface_group: "inlet".to_string(),
            bc_type: BcType::VelocityInlet,
            values: serde_json::json!({"velocity": [10.0, 0.0, 0.0]}),
        };

        assert!(validate_boundary_condition(&bc).is_ok());

        let invalid_bc = BoundaryCondition {
            surface_group: "wall".to_string(),
            bc_type: BcType::VelocityInlet,  // Wrong BC type for wall
            values: serde_json::json!({}),
        };

        assert!(validate_boundary_condition(&invalid_bc).is_err());
    }
}
```

### Integration Tests

```rust
// tests/api_integration.rs

use flowcfd_api::{create_app, config::Config};
use sqlx::PgPool;
use axum::http::{Request, StatusCode};
use tower::ServiceExt;

#[sqlx::test]
async fn test_complete_workflow(pool: PgPool) {
    let config = Config::from_env();
    let app = create_app(config, pool.clone()).await;

    // 1. Register user
    let response = app.clone()
        .oneshot(
            Request::builder()
                .uri("/api/auth/register")
                .method("POST")
                .header("content-type", "application/json")
                .body(r#"{"email":"test@example.com","password":"password123","name":"Test User"}"#.into())
                .unwrap()
        )
        .await
        .unwrap();

    assert_eq!(response.status(), StatusCode::CREATED);

    // 2. Login
    let response = app.clone()
        .oneshot(
            Request::builder()
                .uri("/api/auth/login")
                .method("POST")
                .header("content-type", "application/json")
                .body(r#"{"email":"test@example.com","password":"password123"}"#.into())
                .unwrap()
        )
        .await
        .unwrap();

    assert_eq!(response.status(), StatusCode::OK);
    let body = hyper::body::to_bytes(response.into_body()).await.unwrap();
    let login_response: serde_json::Value = serde_json::from_slice(&body).unwrap();
    let token = login_response["token"].as_str().unwrap();

    // 3. Create project
    let response = app.clone()
        .oneshot(
            Request::builder()
                .uri("/api/projects")
                .method("POST")
                .header("authorization", format!("Bearer {}", token))
                .header("content-type", "application/json")
                .body(r#"{"name":"Test Project","application_type":"external_aero"}"#.into())
                .unwrap()
        )
        .await
        .unwrap();

    assert_eq!(response.status(), StatusCode::CREATED);

    // ... continue with geometry upload, mesh generation, case setup, job submission
}
```

### Load Testing (k6)

```javascript
// tests/load/simulation_workflow.js

import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '2m', target: 50 },   // Ramp up to 50 users
    { duration: '5m', target: 50 },   // Stay at 50 users
    { duration: '2m', target: 100 },  // Ramp up to 100 users
    { duration: '5m', target: 100 },  // Stay at 100 users
    { duration: '2m', target: 0 },    // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],  // 95% of requests under 500ms
    http_req_failed: ['rate<0.01'],     // Error rate under 1%
  },
};

const BASE_URL = 'https://api.flowcfd.com';

export default function() {
  // Login
  let loginRes = http.post(`${BASE_URL}/api/auth/login`, JSON.stringify({
    email: 'loadtest@example.com',
    password: 'password123'
  }), {
    headers: { 'Content-Type': 'application/json' },
  });

  check(loginRes, {
    'login successful': (r) => r.status === 200,
  });

  const token = loginRes.json('token');

  // List projects
  let projectsRes = http.get(`${BASE_URL}/api/projects`, {
    headers: { 'Authorization': `Bearer ${token}` },
  });

  check(projectsRes, {
    'projects listed': (r) => r.status === 200,
  });

  sleep(1);
}
```

---

## Production Checklist

### Security

- [x] JWT tokens with short expiry (24h access, 30d refresh)
- [x] Bcrypt password hashing (cost 12)
- [x] Rate limiting: 100 req/min per user
- [x] SQL injection prevention via SQLx compile-time checks
- [x] CORS configuration for allowed origins
- [x] CSP headers for XSS prevention
- [x] HTTPS enforced (TLS 1.3)
- [x] S3 presigned URLs with 1-hour expiry
- [x] Secrets in K8s Secrets (not ConfigMaps)
- [x] Network policies isolating compute nodes

### Reliability

- [x] Database connection pooling (max 20 connections)
- [x] Retry logic for S3 and K8s API calls (exponential backoff)
- [x] Circuit breaker for external services (mesh AI, VTK)
- [x] Graceful shutdown (drain connections, finish in-flight requests)
- [x] Health check endpoints (/health/live, /health/ready)
- [x] Job timeout limits (max 24h wall time)
- [x] Dead letter queue for failed background jobs

### Observability

- [x] Structured JSON logging (tracing crate)
- [x] Request ID propagation across services
- [x] Prometheus metrics export
- [x] Grafana dashboards (API latency, job queue depth, error rate)
- [x] Sentry error tracking with source maps
- [x] OpenTelemetry tracing for distributed requests

### Performance

- [x] Database indexes on foreign keys and query filters
- [x] Redis caching for user sessions (5min TTL)
- [x] S3 CloudFront CDN for static assets and WASM
- [x] Progressive VTK loading (1MB chunks)
- [x] Database query optimization (EXPLAIN ANALYZE)
- [x] Axum response compression (gzip/brotli)

### Cost Optimization

- [x] Spot instances for CFD compute (70% savings)
- [x] Karpenter auto-scaling (scale to zero when idle)
- [x] S3 lifecycle policies (archive to Glacier after 90d)
- [x] Database connection pooling (reduce RDS costs)
- [x] CloudFront caching (reduce data transfer)

---

## Post-MVP Roadmap

### v1.1 — Transient Simulations (Month 3)
- pimpleFoam transient solver support
- Adaptive time-stepping
- Animation export (MP4, GIF)
- Transient force/moment monitoring

### v1.2 — Advanced Turbulence Models (Month 4)
- LES (Large Eddy Simulation) with Smagorinsky/WALE models
- DES (Detached Eddy Simulation)
- RSM (Reynolds Stress Model)
- Wall-resolved vs wall-modeled LES selector

### v1.3 — Compressible Flow (Month 5)
- rhoCentralFoam for supersonic flow
- Shock capturing schemes
- Mach number contours
- Pressure ratio visualization

### v1.4 — Multiphase Flow (Month 6)
- interFoam for VOF (Volume of Fluid) method
- Free surface tracking
- Wave simulations
- Sloshing analysis

### v1.5 — Conjugate Heat Transfer (Month 7)
- chtMultiRegionFoam solver
- Solid-fluid coupling
- Heat exchanger analysis
- Electronics cooling

### v1.6 — Parametric Studies (Month 8)
- Design of experiments (DOE)
- Latin hypercube sampling
- Response surface methodology
- Optimization (minimize drag, maximize cooling)

### v2.0 — Enterprise Features (Month 12)
- SSO/SAML integration
- Private VPC deployment
- Audit logs and compliance
- PLM system integration (Teamcenter, Windchill)
- Custom turbulence model upload
- Dedicated compute allocation
