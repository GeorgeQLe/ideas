# 86. PartSim — Particle and Granular Flow Simulation (DEM)

## Implementation Plan

**MVP Scope:** Browser-based 3D equipment geometry editor with STL import and drag-and-drop particle source/sensor placement rendered via Three.js, custom DEM solver implementing Hertz-Mindlin contact mechanics with Verlet integration compiled to WebAssembly for client-side execution of systems ≤10,000 spherical particles and server-side Rust+CUDA GPU-accelerated execution for larger simulations up to 1M+ particles, support for multi-sphere clump particles with 2-10 spheres per aggregate for irregular grain shapes, particle factory with volumetric filling and surface injection, material calibration workflow matching angle of repose and bulk density to virtual tests, interactive 3D particle visualization rendered via WebGL/Three.js with color-by-velocity/force/kinetic-energy mapping and playback controls, 50+ pre-calibrated material parameter sets (iron ore, coal, wheat, sand, limestone, lactose) stored in S3 with PostgreSQL metadata, LIGGGHTS/EDEM particle definition import, Stripe billing with three tiers (Free / Pro $149/mo / Advanced $399/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| DEM Solver | Rust (native + WASM) + CUDA | Custom Hertz-Mindlin engine with Verlet integration, CUDA kernels for GPU |
| WASM Compilation | `wasm-pack` + `wasm-bindgen` | Client-side solver for simulations ≤10,000 particles |
| Calibration Service | Python 3.12 (FastAPI) | Bayesian optimization (BoTorch), virtual test automation, parameter fitting |
| Database | PostgreSQL 16 | Projects, users, simulations, material parameters, equipment library |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | Simulation results (particle trajectories), equipment geometry (STL/STEP), material models |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| 3D Visualization | Three.js + React-Three-Fiber | WebGL-based particle and equipment rendering with GPU instancing |
| Geometry Editor | Custom Three.js editor | STL import, particle source/sensor placement, camera controls |
| Real-time | WebSocket (Axum) | Live simulation progress, particle count, timestep monitoring |
| Job Queue | Redis 7 + Tokio tasks | Server-side GPU simulation job management, parameter sweep distribution |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | WASM bundle delivery, static frontend assets |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, error tracking, GPU utilization |
| CI/CD | GitHub Actions | WASM build pipeline, Rust tests, CUDA kernel compilation, Docker image push |

### Key Architecture Decisions

1. **Hybrid client/server solver with WASM threshold at 10,000 particles**: Simulations with ≤10,000 spherical particles (covers educational demonstrations, preliminary design checks) run entirely in the browser via WASM, providing instant feedback with zero server cost. Simulations exceeding 10,000 particles or using multi-sphere clumps are submitted to GPU-accelerated cloud servers which handle 100K-1M+ particle industrial systems. The threshold is configurable per plan tier.

2. **Custom DEM engine in Rust+CUDA rather than wrapping LIGGGHTS**: Building a custom DEM solver gives full control over contact models, WASM compilation, GPU acceleration, and calibration workflows. LIGGGHTS is CPU-bound and has limited GPU support; its C++ codebase makes WASM compilation challenging. Our Rust core with CUDA kernels provides memory safety, WASM compatibility via shared solver logic, and multi-GPU scaling for industrial-scale simulations.

3. **Three.js for 3D geometry and particle visualization**: Three.js provides hardware-accelerated particle rendering using GPU instancing (1M+ particles at 60fps), robust STL/OBJ import, and a mature camera control system. React-Three-Fiber gives declarative 3D scene management integrated with React state. This approach is superior to custom WebGL for 3D physics visualization where we need complex geometry, lighting, and camera interaction.

4. **Particle data stored as binary time-series in S3**: Particle positions, velocities, and forces for each timestep are written to compact binary format (Float32Array) and streamed to S3. This enables large simulations (1M particles × 10K timesteps = 120GB raw data) to be stored economically and streamed on-demand for playback. PostgreSQL stores only metadata and summary statistics.

5. **Python FastAPI microservice for material calibration**: Bayesian optimization for parameter fitting requires scientific Python libraries (BoTorch, GPyOpt, SciPy) that are impractical to port to Rust. The calibration service runs virtual test simulations via Rust DEM solver, measures outputs (angle of repose, bulk density), and iterates parameters to match physical test data. This is decoupled from the main API and runs on CPU-only instances.

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
    plan TEXT NOT NULL DEFAULT 'free',  -- free | pro | advanced
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX users_email_idx ON users(email);
CREATE INDEX users_stripe_idx ON users(stripe_customer_id);

-- Organizations (for team collaboration)
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

-- Projects (simulation workspace with equipment geometry)
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    geometry_url TEXT,  -- S3 URL to STL/STEP file
    scene_data JSONB NOT NULL DEFAULT '{}',  -- Particle sources, sensors, material assignments
    settings JSONB DEFAULT '{}',  -- Simulation settings (gravity, timestep)
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
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | running | completed | failed | cancelled
    execution_mode TEXT NOT NULL DEFAULT 'wasm',  -- wasm | gpu_cloud
    particle_count INTEGER NOT NULL DEFAULT 0,
    multi_sphere_count INTEGER DEFAULT 0,  -- Number of multi-sphere clumps
    max_particles INTEGER NOT NULL DEFAULT 0,  -- Peak particle count during simulation
    parameters JSONB NOT NULL DEFAULT '{}',  -- Contact model params, timestep, duration
    results_url TEXT,  -- S3 URL for particle trajectory data
    results_summary JSONB,  -- Key metrics: avg velocity, max force, residence time
    error_message TEXT,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    wall_time_seconds REAL,
    gpu_hours REAL DEFAULT 0.0,  -- For billing
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX sims_project_idx ON simulations(project_id);
CREATE INDEX sims_user_idx ON simulations(user_id);
CREATE INDEX sims_status_idx ON simulations(status);
CREATE INDEX sims_created_idx ON simulations(created_at DESC);

-- Server Simulation Jobs (GPU execution only)
CREATE TABLE simulation_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    simulation_id UUID NOT NULL REFERENCES simulations(id) ON DELETE CASCADE,
    worker_id TEXT,
    gpu_type TEXT,  -- p4d.24xlarge | p5.48xlarge | g5.xlarge
    gpu_count INTEGER DEFAULT 1,
    memory_gb INTEGER DEFAULT 32,
    priority INTEGER DEFAULT 0,  -- Higher = more urgent
    progress_pct REAL DEFAULT 0.0,
    current_timestep INTEGER DEFAULT 0,
    total_timesteps INTEGER DEFAULT 0,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX jobs_sim_idx ON simulation_jobs(simulation_id);
CREATE INDEX jobs_worker_idx ON simulation_jobs(worker_id);
CREATE INDEX jobs_priority_idx ON simulation_jobs(priority DESC, created_at ASC);

-- Material Parameters (metadata; actual calibration data in S3)
CREATE TABLE materials (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,  -- e.g., "Iron Ore - Fines", "Lactose Monohydrate"
    category TEXT NOT NULL,  -- mining | pharma | food | agriculture | construction
    description TEXT,
    density REAL NOT NULL,  -- kg/m³
    particle_size_min REAL,  -- mm
    particle_size_max REAL,  -- mm
    hertz_mindlin_params JSONB NOT NULL,  -- {young_modulus, poisson_ratio, restitution, friction_pp, friction_pw, rolling_friction}
    cohesion_params JSONB DEFAULT '{}',  -- JKR adhesion if applicable
    calibration_data_url TEXT,  -- S3 URL to calibration test results
    is_builtin BOOLEAN DEFAULT true,
    uploaded_by UUID REFERENCES users(id),
    tags TEXT[] DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX materials_category_idx ON materials(category);
CREATE INDEX materials_name_trgm_idx ON materials USING gin(name gin_trgm_ops);
CREATE INDEX materials_tags_idx ON materials USING gin(tags);

-- Equipment Library (reusable geometry templates)
CREATE TABLE equipment_templates (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    category TEXT NOT NULL,  -- hopper | chute | conveyor | mill | mixer | screen
    description TEXT,
    geometry_url TEXT NOT NULL,  -- S3 URL to STL file
    thumbnail_url TEXT,
    parameters JSONB DEFAULT '{}',  -- Configurable dimensions/properties
    is_builtin BOOLEAN DEFAULT true,
    uploaded_by UUID REFERENCES users(id),
    download_count INTEGER DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX equipment_category_idx ON equipment_templates(category);

-- Saved Post-Processing Views
CREATE TABLE analysis_views (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    simulation_id UUID NOT NULL REFERENCES simulations(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    view_type TEXT NOT NULL,  -- velocity_field | force_chain | residence_time | mixing_index
    config JSONB NOT NULL,  -- Timestep range, color scale, measurement regions
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX analysis_sim_idx ON analysis_views(simulation_id);

-- Usage Tracking (for billing enforcement)
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,  -- gpu_hours | wasm_simulations | storage_gb
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
    pub geometry_url: Option<String>,
    pub scene_data: serde_json::Value,
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
    pub status: String,
    pub execution_mode: String,
    pub particle_count: i32,
    pub multi_sphere_count: Option<i32>,
    pub max_particles: i32,
    pub parameters: serde_json::Value,
    pub results_url: Option<String>,
    pub results_summary: Option<serde_json::Value>,
    pub error_message: Option<String>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub wall_time_seconds: Option<f32>,
    pub gpu_hours: f32,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Material {
    pub id: Uuid,
    pub name: String,
    pub category: String,
    pub description: Option<String>,
    pub density: f32,
    pub particle_size_min: Option<f32>,
    pub particle_size_max: Option<f32>,
    pub hertz_mindlin_params: serde_json::Value,
    pub cohesion_params: serde_json::Value,
    pub calibration_data_url: Option<String>,
    pub is_builtin: bool,
    pub uploaded_by: Option<Uuid>,
    pub tags: Vec<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct SimulationParams {
    pub duration: f32,  // Simulation time in seconds
    pub timestep: Option<f32>,  // Fixed timestep in seconds (auto-calculated if None)
    pub gravity: Vec3,
    pub contact_model: ContactModelType,
    pub hertz_mindlin: HertzMindlinParams,
    pub output_frequency: u32,  // Write results every N timesteps
}

#[derive(Debug, Deserialize, Serialize)]
pub enum ContactModelType {
    HertzMindlin,
    LinearSpring,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct HertzMindlinParams {
    pub young_modulus: f32,       // Pa
    pub poisson_ratio: f32,       // dimensionless
    pub restitution_coeff: f32,   // 0-1
    pub friction_pp: f32,         // particle-particle
    pub friction_pw: f32,         // particle-wall
    pub rolling_friction: f32,    // dimensionless
}

#[derive(Debug, Deserialize, Serialize)]
pub struct Vec3 {
    pub x: f32,
    pub y: f32,
    pub z: f32,
}
```

---

## Solver Architecture Deep-Dive

### Governing Equations and Discretization

PartSim's core solver implements the **Discrete Element Method (DEM)**, specifically the soft-sphere approach with Hertz-Mindlin contact mechanics. For a system of N particles, Newton's equations of motion are integrated for each particle i:

```
m_i · dv_i/dt = F_contact,i + F_gravity,i
I_i · dω_i/dt = T_contact,i

where:
  m_i = mass of particle i
  v_i = velocity vector
  ω_i = angular velocity vector
  I_i = moment of inertia tensor
  F_contact,i = sum of contact forces from overlapping neighbors
  T_contact,i = sum of contact torques (tangential force × contact point)
```

**Hertz-Mindlin contact model** decomposes contact force into normal and tangential components:

```
Normal force (Hertz):
F_n = (4/3) · E* · √(R*) · δ_n^(3/2) - γ_n · v_n

where:
  E* = effective Young's modulus = 1 / [(1-ν₁²)/E₁ + (1-ν₂²)/E₂]
  R* = effective radius = 1 / (1/R₁ + 1/R₂)
  δ_n = normal overlap distance
  γ_n = normal damping coefficient (from restitution coefficient)
  v_n = relative normal velocity

Tangential force (Mindlin):
F_t = min(μ · F_n, -k_t · δ_t - γ_t · v_t)

where:
  k_t = tangential stiffness = 8·G*·√(R*·δ_n)
  G* = effective shear modulus = 1 / [2(2-ν₁)(1+ν₁)/E₁ + 2(2-ν₂)(1+ν₂)/E₂]
  δ_t = accumulated tangential displacement (history variable)
  μ = friction coefficient
  γ_t = tangential damping coefficient

Rolling friction torque:
T_r = -μ_r · R* · F_n · (ω / |ω|)
```

**Time integration** uses Velocity Verlet (symplectic, second-order accurate):

```
Position update:    x(t+Δt) = x(t) + v(t)·Δt + 0.5·a(t)·Δt²
Velocity half-step: v(t+Δt/2) = v(t) + 0.5·a(t)·Δt
Force calculation:  F(t+Δt) from contact detection at x(t+Δt)
Velocity update:    v(t+Δt) = v(t+Δt/2) + 0.5·a(t+Δt)·Δt
```

**Critical timestep** (Rayleigh wave criterion for stability):

```
Δt_crit = π · R · √(ρ/G) / (0.1631·ν + 0.8766)

where:
  R = particle radius
  ρ = density
  G = shear modulus
  ν = Poisson ratio

For safety, we use Δt = 0.2 · Δt_crit (20% of Rayleigh limit)
```

### Client/Server Split (WASM Threshold)

```
Scene submitted → Particle count estimated
    │
    ├── ≤10,000 particles (spheres only) → WASM solver (browser)
    │   ├── Instant startup, no network latency
    │   ├── Results rendered in real-time
    │   └── No server cost, no GPU queuing
    │
    └── >10,000 particles OR multi-sphere clumps → GPU cloud solver
        ├── Job queued via Redis
        ├── GPU worker picks up, runs CUDA kernels
        ├── Progress streamed via WebSocket
        └── Results stored in S3 as binary trajectory file
```

The 10,000-particle threshold was chosen because:
- WASM solver handles 10K spherical particles (50 timesteps/sec real-time) on modern desktop browsers
- 10K particles covers: hopper discharge demonstrations (2K-5K), angle of repose tests (5K-8K), small conveyor sections (3K-7K)
- Above 10K particles: industrial-scale equipment simulations (100K-1M particles), coupled CFD-DEM, multi-sphere aggregates → require GPU acceleration

### WASM Compilation Pipeline

```toml
# solver-wasm/Cargo.toml
[package]
name = "partsim-solver-wasm"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
wasm-bindgen = "0.2"
serde = { version = "1", features = ["derive"] }
serde-wasm-bindgen = "0.6"
js-sys = "0.3"
glam = "0.27"  # Fast SIMD vector math
rstar = "0.12"  # R*-tree for neighbor search
log = "0.4"
console_log = "1"

[profile.release]
opt-level = "z"       # Optimize for size
lto = true            # Link-time optimization
codegen-units = 1
strip = true
```

---

## Architecture Deep-Dives

### 1. Simulation API Handler (Rust/Axum)

```rust
// src/api/handlers/simulation.rs

use axum::{extract::{Path, State, Json}, http::StatusCode, response::IntoResponse};
use uuid::Uuid;
use crate::{db::models::{Simulation, SimulationParams}, state::AppState, auth::Claims, error::ApiError};

pub async fn create_simulation(
    State(state): State<AppState>,
    claims: Claims,
    Path(project_id): Path<Uuid>,
    Json(req): Json<CreateSimulationRequest>,
) -> Result<impl IntoResponse, ApiError> {
    let project = sqlx::query_as!(
        crate::db::models::Project,
        "SELECT * FROM projects WHERE id = $1 AND (owner_id = $2 OR org_id IN (
            SELECT org_id FROM org_members WHERE user_id = $2
        ))", project_id, claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Project not found"))?;

    let estimated_particles: u32 = req.particle_sources.iter()
        .map(|s| (s.rate * req.parameters.duration) as u32)
        .sum();

    let user = sqlx::query_as!(crate::db::models::User, "SELECT * FROM users WHERE id = $1", claims.user_id)
        .fetch_one(&state.db)
        .await?;

    if user.plan == "free" && estimated_particles > 10_000 {
        return Err(ApiError::PlanLimit("Free plan supports simulations up to 10,000 particles. Upgrade to Pro for unlimited."));
    }

    let has_multi_sphere = req.particle_sources.iter().any(|s| s.shape_type == "multi_sphere");
    let execution_mode = if estimated_particles <= 10_000 && !has_multi_sphere { "wasm" } else { "gpu_cloud" };

    let timestep = if let Some(dt) = req.parameters.timestep {
        dt
    } else {
        calculate_critical_timestep(&req, &state.db).await?
    };

    let mut params = req.parameters;
    params.timestep = Some(timestep);

    let sim = sqlx::query_as!(
        Simulation,
        r#"INSERT INTO simulations (project_id, user_id, status, execution_mode, particle_count, parameters)
        VALUES ($1, $2, $3, $4, $5, $6) RETURNING *"#,
        project_id, claims.user_id,
        if execution_mode == "wasm" { "completed" } else { "pending" },
        execution_mode, estimated_particles as i32, serde_json::to_value(&params)?,
    )
    .fetch_one(&state.db)
    .await?;

    if execution_mode == "gpu_cloud" {
        let gpu_type = if estimated_particles > 500_000 { "p4d.24xlarge" }
                       else if estimated_particles > 100_000 { "g5.12xlarge" }
                       else { "g5.xlarge" };

        let job = sqlx::query_as!(
            crate::db::models::SimulationJob,
            r#"INSERT INTO simulation_jobs (simulation_id, priority, gpu_type) VALUES ($1, $2, $3) RETURNING *"#,
            sim.id, if user.plan == "advanced" { 10 } else { 0 }, gpu_type,
        )
        .fetch_one(&state.db)
        .await?;

        state.redis.zadd("simulation:jobs", job.priority, serde_json::to_string(&serde_json::json!({"job_id": job.id}))?).await?;
    }

    Ok((StatusCode::CREATED, Json(sim)))
}
```

### 2. DEM Solver Core (Rust)

```rust
// solver-core/src/dem.rs

use glam::Vec3;
use rstar::{RTree, AABB};

pub struct Particle {
    pub id: u32,
    pub position: Vec3,
    pub velocity: Vec3,
    pub angular_velocity: Vec3,
    pub radius: f32,
    pub mass: f32,
    pub inertia: f32,
    pub material_id: u32,
    pub force: Vec3,
    pub torque: Vec3,
}

pub struct DemSystem {
    pub particles: Vec<Particle>,
    pub walls: Vec<WallTriangle>,
    pub gravity: Vec3,
    pub timestep: f32,
    pub time: f32,
    pub contact_model: Box<dyn ContactModel>,
}

impl DemSystem {
    pub fn step(&mut self) {
        for p in &mut self.particles {
            p.force = self.gravity * p.mass;
            p.torque = Vec3::ZERO;
        }

        let contacts = self.detect_contacts();
        for contact in contacts {
            self.contact_model.apply_force(&mut self.particles, contact.i, contact.j, contact.overlap, contact.normal);
        }

        for p in &mut self.particles {
            let accel = p.force / p.mass;
            let angular_accel = p.torque / p.inertia;
            p.position += p.velocity * self.timestep + 0.5 * accel * self.timestep.powi(2);
            p.velocity += accel * self.timestep;
            p.angular_velocity += angular_accel * self.timestep;
        }

        self.time += self.timestep;
    }

    fn detect_contacts(&self) -> Vec<Contact> {
        let mut contacts = Vec::new();
        let tree: RTree<ParticleAABB> = RTree::bulk_load(
            self.particles.iter().enumerate()
                .map(|(i, p)| ParticleAABB {
                    particle_id: i,
                    aabb: AABB::from_corners(
                        [p.position.x - p.radius, p.position.y - p.radius, p.position.z - p.radius],
                        [p.position.x + p.radius, p.position.y + p.radius, p.position.z + p.radius],
                    ),
                })
                .collect()
        );

        for (i, p1) in self.particles.iter().enumerate() {
            for neighbor in tree.locate_within_distance([p1.position.x, p1.position.y, p1.position.z], (p1.radius * 2.0).powi(2)) {
                let j = neighbor.particle_id;
                if j <= i { continue; }
                let p2 = &self.particles[j];
                let delta = p2.position - p1.position;
                let dist = delta.length();
                let overlap = p1.radius + p2.radius - dist;
                if overlap > 0.0 {
                    contacts.push(Contact { i, j, overlap, normal: delta / dist });
                }
            }
        }
        contacts
    }
}
```

### 3. Hertz-Mindlin Contact Model (Rust)

```rust
// solver-core/src/contact_models/hertz_mindlin.rs

use glam::Vec3;
use std::collections::HashMap;
use crate::dem::{Particle, ContactModel};

pub struct HertzMindlinModel {
    pub young_modulus: f32,
    pub poisson_ratio: f32,
    pub restitution: f32,
    pub friction_pp: f32,
    pub friction_pw: f32,
    pub rolling_friction: f32,
    tangential_history: HashMap<(u32, u32), Vec3>,
}

impl ContactModel for HertzMindlinModel {
    fn apply_force(&self, particles: &mut [Particle], i: usize, j: usize, overlap: f32, normal: Vec3) {
        let p1 = &particles[i];
        let p2 = &particles[j];

        let r_eff = (p1.radius * p2.radius) / (p1.radius + p2.radius);
        let m_eff = (p1.mass * p2.mass) / (p1.mass + p2.mass);

        let e_star = self.young_modulus / (2.0 * (1.0 - self.poisson_ratio.powi(2)));
        let k_n = (4.0 / 3.0) * e_star * r_eff.sqrt();
        let f_n_elastic = k_n * overlap.powf(1.5);

        let v_rel = p2.velocity - p1.velocity;
        let v_n = v_rel.dot(normal);
        let damping_coeff = -2.0 * (self.restitution.ln() / (self.restitution.ln().powi(2) + std::f32::consts::PI.powi(2)).sqrt())
            * (m_eff * k_n * overlap.sqrt()).sqrt();
        let f_n_damping = damping_coeff * v_n;
        let f_n_total = (f_n_elastic + f_n_damping).max(0.0);

        let g_star = self.young_modulus / (2.0 * (2.0 - self.poisson_ratio) * (1.0 + self.poisson_ratio));
        let k_t = 8.0 * g_star * (r_eff * overlap).sqrt();
        let v_t = v_rel - normal * v_n;

        let f_t_limit = self.friction_pp * f_n_total;
        let f_t = v_t.normalize_or_zero() * f_t_limit;

        let f_total = normal * f_n_total + f_t;

        unsafe {
            let p1_ptr = particles.as_mut_ptr().add(i);
            let p2_ptr = particles.as_mut_ptr().add(j);
            (*p1_ptr).force += f_total;
            (*p2_ptr).force -= f_total;
        }
    }
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–5)

**Day 1: Project initialization and Rust backend scaffold**
- `cargo init partsim-api` and workspace setup
- `src/main.rs` — Axum app with CORS, tracing, graceful shutdown
- `src/config.rs` — Environment config (DATABASE_URL, JWT_SECRET, S3_BUCKET, REDIS_URL)
- `src/state.rs` — AppState (PgPool, Redis, S3Client)
- `src/error.rs` — ApiError enum with IntoResponse
- `Dockerfile` — Multi-stage build
- `docker-compose.yml` — PostgreSQL, Redis, MinIO

**Day 2: Database schema and migrations**
- `migrations/001_initial.sql` — All 9 tables: users, organizations, org_members, projects, simulations, simulation_jobs, materials, equipment_templates, analysis_views, usage_records
- `src/db/models.rs` — All SQLx structs with FromRow derives
- `sqlx migrate run` to apply schema
- Seed script for 50+ material parameter sets (iron ore, coal, wheat, sand, limestone, lactose)

**Day 3: Authentication system**
- `src/auth/mod.rs` — JWT generation and validation middleware
- `src/auth/oauth.rs` — Google/GitHub OAuth 2.0 handlers
- `src/api/handlers/auth.rs` — Register, login, OAuth callback, refresh token, me endpoints
- Password hashing with bcrypt (cost 12)
- JWT: 24h access + 30d refresh token

**Day 4: User and project CRUD**
- `src/api/handlers/users.rs` — Profile CRUD
- `src/api/handlers/projects.rs` — Create, list, get, update, delete, fork
- `src/api/handlers/orgs.rs` — Create org, invite member, list, remove
- `src/api/router.rs` — All route definitions with auth middleware
- Integration tests: auth flow, project CRUD, authorization

**Day 5: Material library API**
- `src/api/handlers/materials.rs` — Search materials (parametric + full-text), get material, list by category
- S3 integration for calibration data files
- Parametric search: filter by category, density range, particle size range
- Pagination with cursor-based scrolling

### Phase 2 — DEM Solver Core (Days 6–13)

**Day 6: DEM system framework**
- `solver-core/` — New Rust workspace member
- `solver-core/src/dem.rs` — Particle struct, DemSystem with step() method
- `solver-core/src/geometry.rs` — WallTriangle, STL loading via `stl_io` crate
- Unit tests: single particle free fall, two-particle collision

**Day 7: Hertz-Mindlin contact model**
- `solver-core/src/contact_models/hertz_mindlin.rs` — Normal force (Hertz), tangential force (Mindlin), rolling friction
- Effective modulus and radius calculations
- Damping coefficient from restitution
- Tests: particle-particle collision, rebound height verification

**Day 8: Contact detection — R-tree spatial index**
- `solver-core/src/contact_detection.rs` — R*-tree setup via `rstar` crate
- Broad-phase: AABB overlap detection
- Narrow-phase: exact sphere-sphere distance check
- Tests: 100 particles in confined box, verify all contacts detected

**Day 9: Particle-wall collision**
- `solver-core/src/wall_contact.rs` — Sphere-triangle intersection test
- Point-in-triangle barycentric check
- Particle-wall force application
- Tests: particle dropping onto flat floor, angled chute, cylindrical hopper

**Day 10: Multi-sphere clumps**
- `solver-core/src/clumps.rs` — ClumpParticle struct with constituent spheres
- Rigid-body motion of clump (translation + rotation)
- Contact detection for clump constituents
- Tests: dumbbell-shaped clump rolling down slope

**Day 11: Velocity Verlet integrator**
- `solver-core/src/integrator.rs` — Verlet position/velocity update
- Critical timestep calculation (Rayleigh criterion)
- Auto-timestep selection from material properties
- Tests: energy conservation in collision, particle trajectory accuracy

**Day 12: Particle factory and sources**
- `solver-core/src/particle_factory.rs` — Volumetric filling, surface injection, conveyor belt feed
- Particle size distribution sampling (normal, log-normal)
- Material assignment per source
- Tests: hopper filling, conveyor loading profile

**Day 13: Result serialization**
- `solver-core/src/results.rs` — Binary trajectory format (particle positions/velocities per timestep)
- Compression via LZ4 for efficient S3 storage
- Summary statistics: avg velocity, max force, kinetic energy
- Tests: save/load 10K particles × 1000 timesteps

### Phase 3 — WASM Build + Frontend 3D Visualization (Days 14–20)

**Day 14: WASM solver compilation**
- `solver-wasm/` — New workspace member
- `solver-wasm/Cargo.toml` — wasm-bindgen, glam, rstar dependencies
- `solver-wasm/src/lib.rs` — WASM entry points: `run_simulation()`, `get_results()`
- `wasm-pack build --target web --release` + `wasm-opt -Oz`
- Target <3MB gzipped

**Day 15: Frontend scaffold and Three.js setup**
- `npm create vite@latest frontend -- --template react-ts`
- `npm i three @react-three/fiber @react-three/drei zustand @tanstack/react-query axios`
- `src/App.tsx` — Router, auth context, layout
- `src/stores/projectStore.ts` — Project state
- `src/stores/sceneStore.ts` — 3D scene state (geometry, particle sources, sensors)
- `src/lib/wasmLoader.ts` — WASM solver loading

**Day 16: 3D geometry viewer**
- `src/components/SceneViewer/SceneCanvas.tsx` — React-Three-Fiber canvas with OrbitControls
- `src/components/SceneViewer/GeometryMesh.tsx` — STL mesh rendering with THREE.BufferGeometry
- `src/components/SceneViewer/Grid.tsx` — Ground plane grid
- Camera presets: isometric, front, side, top
- Lighting: ambient + directional + hemisphere

**Day 17: STL import and geometry manipulation**
- `src/components/SceneViewer/STLImporter.tsx` — File upload, STL parsing via `three-stdlib`
- `src/components/SceneViewer/GeometryControls.tsx` — Rotate, translate, scale equipment
- Bounding box display, center of mass indicator
- Export modified geometry back to STL

**Day 18: Particle source and sensor placement**
- `src/components/SceneViewer/ParticleSource.tsx` — 3D gizmo for source placement (position, rate, material)
- `src/components/SceneViewer/Sensor.tsx` — Measurement plane for mass flow rate
- Drag handles in 3D space
- Source/sensor property panel: position, orientation, rate, material selection

**Day 19: Particle visualization with GPU instancing**
- `src/components/SceneViewer/ParticleRenderer.tsx` — THREE.InstancedMesh for millions of particles
- GPU instancing: position + color per particle
- Color mapping: velocity magnitude, kinetic energy, force magnitude
- LOD: reduce instance count when zoomed out

**Day 20: Playback controls and animation**
- `src/components/SceneViewer/PlaybackControls.tsx` — Play/pause, timestep slider, speed control
- Frame interpolation for smooth playback
- Export animation as MP4 via `canvas.captureStream()` + MediaRecorder
- Performance: 60 FPS with 100K particles on RTX 3060

### Phase 4 — API + Job Orchestration (Days 21–26)

**Day 21: Simulation API endpoints**
- `src/api/handlers/simulation.rs` — Create simulation, get simulation, list simulations, cancel simulation
- Particle count estimation from sources and duration
- WASM/GPU routing logic based on particle count and shape type
- Plan limits enforcement (free: 10K particles, pro: 500K, advanced: unlimited)

**Day 22: GPU worker — CUDA kernel skeleton**
- `cuda/partsim.cu` — CUDA kernels for contact detection and force computation
- `build.rs` — Compile CUDA kernels, link to Rust via FFI
- `src/workers/gpu_worker.rs` — Redis job consumer, CUDA kernel invocation
- GPU memory management: particle buffers, contact buffers, wall buffers
- Tests: 100K particles on g5.xlarge instance

**Day 23: WebSocket for live simulation progress**
- `src/api/ws/mod.rs` — Axum WebSocket upgrade handler
- `src/api/ws/simulation_progress.rs` — Subscribe to simulation progress channel
- Progress broadcast: timestep, particle count, estimated time remaining
- Frontend: `src/hooks/useSimulationProgress.ts` — React hook for WebSocket

**Day 24: S3 result upload and presigned URLs**
- `src/services/s3.rs` — S3 client helpers for trajectory upload
- Presigned URL generation for client-side download
- Result caching: hot results in Redis (1h TTL), cold in S3
- Multipart upload for large result files (>100MB)

**Day 25: Equipment template library**
- `src/api/handlers/equipment.rs` — Search templates, get template, upload template
- S3 integration for STL files
- Template categories: hopper, chute, conveyor, mill, mixer, screen
- Seed 20+ built-in templates (simple hopper, v-chute, belt conveyor)

**Day 26: LIGGGHTS/EDEM import**
- `src/api/handlers/import.rs` — Parse LIGGGHTS particle definitions, EDEM CSV
- Map LIGGGHTS material IDs to PartSim material library
- Create particle sources from imported data
- Tests: import 10K-particle LIGGGHTS setup, verify simulation matches

### Phase 5 — Calibration Service (Days 27–30)

**Day 27: Python calibration service setup**
- `calibration-service/` — Python FastAPI app
- `requirements.txt` — BoTorch, GPyOpt, httpx, numpy, scipy
- `main.py` — API endpoints for calibration workflows
- Docker container with Python 3.12

**Day 28: Virtual angle of repose test**
- `calibration-service/angle_of_repose.py` — Generate cone of particles, let settle, measure angle
- Call Rust DEM solver via HTTP API
- Measure repose angle from particle positions
- Objective function: minimize (measured_angle - target_angle)²

**Day 29: Virtual bulk density test**
- `calibration-service/bulk_density.py` — Fill container, tap simulation, measure settled density
- Tap cycle: brief vertical acceleration + settling
- Measure packed density vs. target
- Objective function: minimize (measured_density - target_density)²

**Day 30: Bayesian parameter optimization**
- `calibration-service/optimizer.py` — BoTorch Bayesian optimization loop
- Parameter bounds: friction (0.1-1.0), restitution (0.1-0.9), rolling friction (0.01-0.5)
- Multi-objective: match both angle of repose AND bulk density
- Convergence: <5% error on both metrics within 20 iterations
- Save calibrated parameters to materials table

### Phase 6 — Post-Processing UI (Days 31–35)

**Day 31: Result loading and playback**
- `src/components/Results/ResultViewer.tsx` — Load trajectory from S3, render particles
- Frame-by-frame playback with timestep display
- Particle color mapping: velocity, force, kinetic energy
- Camera animation: orbit around scene

**Day 32: Velocity field visualization**
- `src/components/Results/VelocityField.tsx` — Spatial averaging (coarse-graining) on Eulerian grid
- Arrow gizmos showing velocity vectors
- Color-coded velocity magnitude heatmap
- Grid resolution control (10×10×10 to 50×50×50)

**Day 33: Force chain visualization**
- `src/components/Results/ForceChains.tsx` — Draw lines between contacting particles, thickness = contact force
- Filter: only show forces above threshold
- Color: normal vs. tangential force
- Animation: evolving force network during discharge

**Day 34: Residence time distribution**
- `src/components/Results/ResidenceTime.tsx` — Track particle entry/exit times through measurement planes
- RTD histogram plot (D3.js)
- Mean residence time, variance, min/max
- Export RTD data as CSV

**Day 35: Mass flow rate sensors**
- `src/components/Results/MassFlow.tsx` — Count particles crossing sensor plane per unit time
- Mass flow rate vs. time plot
- Cumulative mass discharged
- Comparison with target flow rate

### Phase 7 — Billing + Plan Enforcement (Days 36–38)

**Day 36: Stripe integration**
- `src/api/handlers/billing.rs` — Create checkout session, customer portal, get subscription
- `src/api/handlers/webhooks/stripe.rs` — Handle subscription events
- Plan mapping: Free (10K particles), Pro $149/mo (500K, 200 GPU-hours/mo), Advanced $399/mo (unlimited, 1000 GPU-hours/mo)

**Day 37: Usage tracking and plan limits**
- `src/middleware/plan_limits.rs` — Check plan before simulation submission
- `src/services/usage.rs` — Track GPU hours per billing period
- Usage dashboard: current period usage, historical usage
- Approaching-limit warnings at 80% and 100%

**Day 38: Billing UI**
- `frontend/src/pages/Billing.tsx` — Plan comparison, current plan, usage meter
- Upgrade/downgrade flow via Stripe Customer Portal
- Feature gating: multi-sphere (Pro+), unlimited particles (Advanced), API access (Advanced)
- Admin endpoint: override plan for beta users

### Phase 8 — Validation + Deployment + Launch (Days 39–42)

**Day 39: Solver validation — standard DEM benchmarks**
- Benchmark 1: Single particle drop — verify rebound height from restitution
- Benchmark 2: Angle of repose (5000 particles) — compare to experimental iron ore data (32°±2°)
- Benchmark 3: Hopper discharge — Beverloo correlation (mass flow vs. outlet diameter)
- Benchmark 4: Particle segregation in drum — verify size segregation matches experiments
- Benchmark 5: Multi-sphere clump — verify rolling friction behavior

**Day 40: Performance testing**
- WASM benchmarks: 1K, 5K, 10K particles, measure FPS and wall-time
- GPU benchmarks: 50K, 100K, 500K, 1M particles on g5.xlarge, g5.12xlarge, p4d.24xlarge
- Target: 10K particles WASM < 5s for 1000 timesteps
- Target: 100K particles GPU < 60s for 10K timesteps
- Load testing: 20 concurrent GPU simulations via k6

**Day 41: Docker and Kubernetes**
- `Dockerfile` — Multi-stage Rust + CUDA build
- `k8s/api-deployment.yaml` — API server (3 replicas, HPA)
- `k8s/gpu-worker-deployment.yaml` — GPU workers (node selector for GPU instances)
- `k8s/postgres-statefulset.yaml`, `k8s/redis-deployment.yaml`
- Health checks: `/health/live`, `/health/ready`

**Day 42: Launch preparation**
- CDN for WASM bundle and static assets
- Prometheus + Grafana dashboards: GPU utilization, simulation throughput, API latency
- Sentry error tracking
- Landing page with demo video
- Documentation: getting started, material calibration guide, STL import
- Deploy to production, enable monitoring, announce launch

---

## Critical Files

```
partsim/
├── solver-core/                           # Shared DEM solver (native + WASM)
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── dem.rs                         # DemSystem, Particle, time integration
│   │   ├── geometry.rs                    # WallTriangle, STL loading
│   │   ├── contact_detection.rs           # R-tree spatial index, broad-phase
│   │   ├── wall_contact.rs                # Sphere-triangle intersection
│   │   ├── clumps.rs                      # Multi-sphere clumps
│   │   ├── integrator.rs                  # Velocity Verlet, timestep calculation
│   │   ├── particle_factory.rs            # Particle generation from sources
│   │   ├── results.rs                     # Binary trajectory format, compression
│   │   └── contact_models/
│   │       ├── mod.rs                     # ContactModel trait
│   │       ├── hertz_mindlin.rs           # Hertz-Mindlin implementation
│   │       └── linear_spring.rs           # Simple linear spring model
│   └── tests/
│       ├── benchmarks.rs                  # Standard DEM validation benchmarks
│       ├── contact_tests.rs               # Contact detection tests
│       └── integration.rs                 # Full system integration tests
│
├── solver-wasm/                           # WASM compilation target
│   ├── Cargo.toml
│   └── src/
│       └── lib.rs                         # WASM entry points (wasm-bindgen)
│
├── cuda/                                  # CUDA kernels for GPU acceleration
│   ├── partsim.cu                         # Contact detection, force compute, integration kernels
│   ├── CMakeLists.txt                     # CUDA compilation config
│   └── build.rs                           # Rust build script for CUDA compilation
│
├── partsim-api/                           # Rust API server
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
│   │   │   │   ├── auth.rs                # Register, login, OAuth
│   │   │   │   ├── users.rs               # Profile CRUD
│   │   │   │   ├── projects.rs            # Project CRUD
│   │   │   │   ├── simulation.rs          # Simulation endpoints
│   │   │   │   ├── materials.rs           # Material library
│   │   │   │   ├── equipment.rs           # Equipment templates
│   │   │   │   ├── import.rs              # LIGGGHTS/EDEM import
│   │   │   │   ├── billing.rs             # Stripe integration
│   │   │   │   ├── usage.rs               # Usage tracking
│   │   │   │   └── webhooks/stripe.rs     # Stripe webhooks
│   │   │   └── ws/
│   │   │       ├── mod.rs                 # WebSocket handler
│   │   │       └── simulation_progress.rs # Live progress
│   │   ├── middleware/
│   │   │   └── plan_limits.rs             # Plan enforcement
│   │   ├── services/
│   │   │   ├── usage.rs                   # Usage tracking service
│   │   │   └── s3.rs                      # S3 helpers
│   │   └── workers/
│   │       ├── mod.rs                     # Worker pool management
│   │       └── gpu_worker.rs              # GPU simulation execution
│   ├── migrations/
│   │   └── 001_initial.sql                # Full database schema
│   └── tests/
│       ├── api_integration.rs             # API tests
│       └── simulation_e2e.rs              # E2E simulation tests
│
├── frontend/                              # React frontend
│   ├── package.json
│   ├── vite.config.ts
│   ├── src/
│   │   ├── App.tsx
│   │   ├── stores/
│   │   │   ├── authStore.ts
│   │   │   ├── projectStore.ts
│   │   │   └── sceneStore.ts
│   │   ├── hooks/
│   │   │   ├── useSimulationProgress.ts   # WebSocket hook
│   │   │   └── useWasmSolver.ts           # WASM solver hook
│   │   ├── lib/
│   │   │   ├── api.ts                     # API client
│   │   │   └── wasmLoader.ts              # WASM loading
│   │   ├── pages/
│   │   │   ├── Dashboard.tsx
│   │   │   ├── Editor.tsx                 # Main 3D editor
│   │   │   ├── Results.tsx                # Result viewer
│   │   │   ├── Materials.tsx              # Material library
│   │   │   ├── Templates.tsx              # Equipment templates
│   │   │   ├── Billing.tsx
│   │   │   ├── Login.tsx
│   │   │   └── Register.tsx
│   │   └── components/
│   │       ├── SceneViewer/
│   │       │   ├── SceneCanvas.tsx        # Three.js canvas
│   │       │   ├── GeometryMesh.tsx       # STL mesh rendering
│   │       │   ├── STLImporter.tsx        # STL import
│   │       │   ├── GeometryControls.tsx   # Transform controls
│   │       │   ├── ParticleSource.tsx     # Particle source gizmo
│   │       │   ├── Sensor.tsx             # Measurement sensor
│   │       │   ├── ParticleRenderer.tsx   # GPU instancing
│   │       │   └── PlaybackControls.tsx   # Animation controls
│   │       ├── Results/
│   │       │   ├── ResultViewer.tsx       # Result playback
│   │       │   ├── VelocityField.tsx      # Velocity visualization
│   │       │   ├── ForceChains.tsx        # Force chain viz
│   │       │   ├── ResidenceTime.tsx      # RTD analysis
│   │       │   └── MassFlow.tsx           # Mass flow sensors
│   │       └── billing/
│   │           ├── PlanCard.tsx
│   │           └── UsageMeter.tsx
│   └── public/
│       └── wasm/                          # WASM solver bundle
│
├── calibration-service/                   # Python FastAPI
│   ├── requirements.txt
│   ├── main.py                            # FastAPI app
│   ├── angle_of_repose.py                 # Virtual AOR test
│   ├── bulk_density.py                    # Virtual bulk density test
│   ├── optimizer.py                       # Bayesian optimization
│   └── Dockerfile
│
├── k8s/                                   # Kubernetes manifests
│   ├── api-deployment.yaml
│   ├── gpu-worker-deployment.yaml
│   ├── postgres-statefulset.yaml
│   ├── redis-deployment.yaml
│   ├── calibration-service-deployment.yaml
│   └── ingress.yaml
│
├── docker-compose.yml                     # Local dev stack
├── Cargo.toml                             # Workspace root
└── .github/
    └── workflows/
        ├── ci.yml                         # Test + lint
        ├── wasm-build.yml                 # WASM build + deploy
        ├── cuda-build.yml                 # CUDA kernel compilation
        └── deploy.yml                     # Docker build + K8s deploy
```

---

## Solver Validation Suite

### Benchmark 1: Single Particle Drop with Rebound

**Setup:** Single sphere (R=5mm, ρ=2500 kg/m³) dropped from height h=1m onto rigid flat floor, restitution e=0.7

**Expected:** Rebound height h' = e² × h = 0.49 × 1m = **0.49 m**

**Tolerance:** < 2% (depends on timestep and numerical dissipation)

### Benchmark 2: Angle of Repose (Iron Ore Fines)

**Setup:** 5,000 spherical particles (D=2mm, ρ=3500 kg/m³) poured into conical pile, friction μ=0.6, restitution e=0.3

**Expected (experimental iron ore data):** Angle of repose θ = **32° ± 2°**

**Measurement:** Fit cone to settled particle positions, measure slope angle

**Tolerance:** < 5% (±1.6°)

### Benchmark 3: Hopper Discharge — Beverloo Correlation

**Setup:** Cylindrical hopper (D_hopper=0.3m, outlet D_outlet=0.05m), 20,000 particles (D_particle=3mm)

**Expected (Beverloo):** Mass flow rate Q = C · ρ_bulk · g^0.5 · (D_outlet - k·D_particle)^2.5

Where C ≈ 0.58, k ≈ 1.4 for spheres

**Expected for this setup:** Q ≈ **1.8 kg/s**

**Tolerance:** < 10% (Beverloo is empirical, DEM may deviate slightly)

### Benchmark 4: Particle Segregation in Rotating Drum

**Setup:** Horizontal cylindrical drum (D=0.4m, L=0.1m), binary mixture: 2,000 large (D=5mm) + 2,000 small (D=2mm) particles, rotation 10 RPM

**Expected:** Size segregation with large particles concentrating at drum surface, small particles at core

**Measurement:** Radial distribution function for particle sizes at steady state

**Tolerance:** Qualitative agreement (segregation index S > 0.5)

### Benchmark 5: Multi-Sphere Clump Rolling Friction

**Setup:** Dumbbell clump (2 spheres, R=5mm, center-to-center distance 12mm) rolling down 10° incline

**Expected:** Rolling resistance limits velocity to terminal value v_terminal = (m·g·sin(θ)) / (μ_roll · m·g·cos(θ) / R)

**Expected for μ_roll=0.3:** v_terminal ≈ **0.28 m/s**

**Tolerance:** < 10%

---

## Verification

### Integration Testing Checklist

1. **Auth flow** — Register → login → JWT → authorized calls → refresh → logout
2. **Project CRUD** — Create → update scene → save → reload → verify state
3. **STL import** — Upload hopper STL → rendered in 3D viewer → bounds displayed
4. **Particle source placement** — Drag source gizmo → set rate/material → verify scene data
5. **WASM simulation** — 1000-particle hopper → WASM solver → results → render particles
6. **GPU simulation** — 50K-particle system → job queued → worker picks up → WebSocket progress → results in S3
7. **Result playback** — Load 10K-particle trajectory → play/pause → color by velocity → smooth 60 FPS
8. **Material library** — Search "iron ore" → results → select → use in source
9. **Calibration** — Submit calibration job → angle of repose + bulk density → optimized parameters returned
10. **Plan limits** — Free user → 15K particle sim → blocked with upgrade prompt
11. **Billing** — Subscribe to Pro → Stripe checkout → webhook → plan updated → limits lifted
12. **Concurrent GPU jobs** — 10 users submit GPU sims → all complete correctly
13. **Force chain viz** — Load result → enable force chains → filter by threshold → animate
14. **LIGGGHTS import** — Upload LIGGGHTS particle data → sources created → simulate → results match
15. **Error handling** — Invalid STL file → clear error message → no crash

### SQL Verification Queries

```sql
-- 1. Simulation throughput and success rate
SELECT
    DATE(created_at) as date,
    COUNT(*) as total_sims,
    COUNT(*) FILTER (WHERE status = 'completed') as completed,
    ROUND(100.0 * COUNT(*) FILTER (WHERE status = 'completed') / COUNT(*), 2) as success_rate_pct,
    ROUND(AVG(wall_time_seconds) FILTER (WHERE status = 'completed'), 2) as avg_wall_time_sec
FROM simulations
WHERE created_at > NOW() - INTERVAL '7 days'
GROUP BY DATE(created_at)
ORDER BY date DESC;

-- 2. GPU hours usage per user (current billing period)
SELECT
    u.email,
    u.plan,
    COALESCE(SUM(ur.quantity), 0) as gpu_hours_used
FROM users u
LEFT JOIN usage_records ur ON u.id = ur.user_id
    AND ur.record_type = 'gpu_hours'
    AND ur.period_start = DATE_TRUNC('month', CURRENT_DATE)
GROUP BY u.id, u.email, u.plan
ORDER BY gpu_hours_used DESC
LIMIT 20;

-- 3. Most popular materials
SELECT
    m.name,
    m.category,
    COUNT(DISTINCT s.user_id) as unique_users,
    COUNT(s.id) as simulation_count
FROM materials m
JOIN simulations s ON s.parameters->>'material_ids' LIKE '%' || m.id::text || '%'
WHERE s.created_at > NOW() - INTERVAL '30 days'
GROUP BY m.id, m.name, m.category
ORDER BY simulation_count DESC
LIMIT 10;

-- 4. Equipment template downloads
SELECT
    name,
    category,
    download_count,
    is_builtin
FROM equipment_templates
ORDER BY download_count DESC
LIMIT 10;

-- 5. Failed simulations (for debugging)
SELECT
    s.id,
    s.particle_count,
    s.execution_mode,
    s.error_message,
    s.created_at
FROM simulations s
WHERE s.status = 'failed'
    AND s.created_at > NOW() - INTERVAL '24 hours'
ORDER BY s.created_at DESC
LIMIT 20;
```

---

## Deployment Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         CloudFront CDN                          │
│  (Static assets, WASM bundle, equipment STL files from S3)     │
└────────────────────┬───────────────────────────────────────────┘
                     │
┌────────────────────┴────────────────────────────────────────────┐
│                   Application Load Balancer                      │
│            (TLS termination, routing, health checks)            │
└────┬──────────────┬──────────────┬──────────────┬──────────────┘
     │              │              │              │
┌────▼──────┐  ┌───▼───────┐  ┌──▼───────┐  ┌──▼───────────────┐
│ API Pod 1 │  │ API Pod 2 │  │ API Pod 3│  │ WebSocket Server │
│ (Rust)    │  │ (Rust)    │  │ (Rust)   │  │ (Axum WS)        │
└────┬──────┘  └───┬───────┘  └──┬───────┘  └──┬───────────────┘
     │             │              │              │
     └─────────────┴──────────────┴──────────────┘
                     │
     ┌───────────────┼───────────────┬────────────────────┐
     │               │               │                    │
┌────▼───────┐  ┌───▼──────────┐  ┌▼────────────┐  ┌───▼──────────┐
│PostgreSQL  │  │    Redis      │  │  S3 Bucket  │  │ Calibration  │
│(RDS)       │  │ (ElastiCache) │  │ (Trajectories│  │ Service      │
│            │  │ (Job queue +  │  │  STL files, │  │ (Python/     │
│            │  │  pub/sub)     │  │  materials) │  │  FastAPI)    │
└────────────┘  └───────────────┘  └─────────────┘  └──────────────┘
                     │
     ┌───────────────┴───────────────┬────────────────────┐
     │                               │                    │
┌────▼──────────────┐  ┌─────────────▼──────┐  ┌────────▼─────────┐
│ GPU Worker 1      │  │ GPU Worker 2        │  │ GPU Worker N     │
│ (g5.xlarge)       │  │ (g5.12xlarge)       │  │ (p4d.24xlarge)   │
│ CUDA Kernels      │  │ CUDA Kernels        │  │ CUDA Kernels     │
│ 1x A10G 24GB      │  │ 4x A10G 24GB        │  │ 8x A100 40GB     │
└───────────────────┘  └─────────────────────┘  └──────────────────┘
```

**Scaling Strategy:**
- API pods: Horizontal auto-scaling based on CPU (target 70%)
- GPU workers: Auto-scaling based on Redis queue depth (>10 jobs → scale up)
- Database: Read replicas for heavy read traffic (material library searches)
- CDN: Caching with 24h TTL for WASM bundle, 1h for STL templates

**Geographic Distribution:**
- Primary region: us-west-2 (Oregon) — GPU availability
- CDN: Global edge locations
- Future: EU region (eu-west-1) for GDPR compliance

---

## Post-MVP Roadmap

### v1.1 — Advanced Particle Shapes (Weeks 1–3)
- Superquadric particles with GJK/EPA contact detection
- Polyhedral particles for angular materials (crushed rock, crystals)
- Shape from image: automatic multi-sphere fitting from microscopy
- Fiber particles for biomass and elongated materials

### v1.2 — Coupled CFD-DEM (Weeks 4–8)
- One-way coupling: DEM particles in prescribed fluid flow (drag + buoyancy)
- Two-way coupling: Euler-Lagrange with particle feedback to fluid
- Fluid solver: incompressible Navier-Stokes (SIMPLE algorithm)
- Drag models: Ergun, Wen-Yu, Di Felice
- Applications: pneumatic conveying, fluidized beds, cyclone separators

### v1.3 — Advanced Material Models (Weeks 9–11)
- JKR adhesion model for cohesive powders (van der Waals forces)
- Liquid bridge model for wet granular materials (capillary forces)
- EEPA model for pressure-dependent consolidation and caking
- Bond models: parallel bond for cemented/sintered materials with breakage

### v1.4 — Wear and Breakage (Weeks 12–15)
- Archard wear model: volumetric wear from contact pressure and sliding
- Finnie erosive wear for pneumatic conveying and chute design
- Particle breakage: Bonded Particle Model (BPM) with fracture criteria
- Attrition model for surface erosion producing fines
- Wear map visualization: cumulative wear depth on equipment surfaces

### v1.5 — Design Optimization (Weeks 16–20)
- Parametric equipment geometry: chute angle, hopper outlet diameter
- Optimization objectives: maximize flow rate, minimize segregation, minimize wear
- Multi-objective optimization: Pareto frontier exploration
- Design of Experiments (DOE): Latin Hypercube Sampling for parameter sweeps
- Surrogate models: Gaussian Process regression for fast design space exploration

---

**Total:** 1498 lines
