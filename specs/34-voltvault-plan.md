# 34. VoltVault — Cloud Battery Simulation & Electrochemistry Platform

## Implementation Plan

**MVP Scope:** Browser-based battery cell designer with visual electrode stack builder selecting from curated materials database (NMC811/622/523, LFP, NCA, graphite, silicon with validated diffusivity, conductivity, OCP curves from literature), GPU-accelerated P2D (pseudo-2D Newman model) electrochemical solver implementing solid-phase diffusion via Fick's law in spherical coordinates, electrolyte transport via concentrated solution theory, Butler-Volmer interfacial kinetics, and charge conservation compiled to Rust with CUDA acceleration using `cudarc` for server-side parallel PDE solving on A100/T4 GPUs and WebAssembly compilation via `wasm-pack` for client-side lightweight discharge simulations in-browser with <1MB gzipped bundle, SPM (single particle model) reduced-order solver with sub-second solve times for rapid design-space screening enabling interactive parameter exploration, cycling data management system with automatic parsing of Arbin CSV and generic CSV formats via Python FastAPI microservice including cycle boundary detection and per-cycle degradation metric extraction (discharge capacity, charge capacity, coulombic efficiency, DC resistance from voltage relaxation), EIS (electrochemical impedance spectroscopy) measurement upload with Nyquist (-Z_imag vs Z_real) and Bode (|Z| and phase vs frequency) plot visualization using D3.js plus Randles and Randles+Warburg equivalent circuit fitting via Python `lmfit` Levenberg-Marquardt optimizer exposing R_s, R_ct, CPE, Warburg parameters with standard errors, interactive voltage-vs-capacity and voltage-vs-time plots rendered with Recharts supporting millions of data points with zoom/pan, experiment tracker with full simulation input/output versioning for reproducibility and IP documentation, automated PDF report generation with cell design summary table and performance curve figures, Stripe billing integration with three-tier pricing (Free 2 projects / Researcher $149/mo 50 GPU-hours / Team $499/mo 200 GPU-hours + 5 seats).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth, compile-time route checking |
| P2D/SPM Solver Core | Rust (native + WASM) | Custom Newman model PDE solver, sparse matrix via `faer`, shared codebase |
| GPU Acceleration | CUDA 12 via `cudarc` | Parallel finite-difference/FVM on T4/A100 GPUs, cuBLAS/cuSPARSE bindings |
| WASM Compilation | `wasm-pack` + `wasm-bindgen` | Client solver for simple cells, size-optimized <1MB gzip, instant feedback |
| ML/Scientific Services | Python 3.12 (FastAPI) | ONLY for EIS fitting (`lmfit`), cycling parsers (Arbin libs), degradation calibration |
| Database | PostgreSQL 16 + TimescaleDB | Users, projects, materials, simulations; hypertables for cycling time-series |
| ORM / Query | SQLx (Rust) | Compile-time SQL verification, async queries, prepared statements, raw migrations |
| Auth | Custom JWT + OAuth 2.0 | Google/GitHub OAuth, ORCID for academics, bcrypt cost 12, 24h tokens |
| Object Storage | AWS S3 | Simulation results JSON, cycling data files, EIS freq/Z data, presigned URLs |
| Frontend Framework | React 19 + TypeScript | Vite build, hot reload, Zustand for global state, React Query for server state |
| Cell Designer UI | Custom SVG renderer | Electrode stack viz (cathode/sep/anode), material property cards, metrics dashboard |
| Data Visualization | Recharts + D3.js | Voltage/capacity curves (Recharts), Nyquist/Bode EIS (D3 custom), zoom/pan |
| 3D Thermal (v1.1) | Three.js + WebGL shaders | Temperature field rendering, contour colormaps, slice planes (post-MVP) |
| Real-time | WebSocket (Axum) | Live solver progress broadcast, convergence residuals, % complete, ETA |
| Job Queue | Redis 7 + Tokio tasks | Priority queue (ZADD) for GPU jobs, DOE sweep distribution, Team plan priority |
| Billing | Stripe | Checkout Sessions, Customer Portal, webhook handlers for subscription lifecycle |
| CDN | CloudFront | WASM bundle delivery, React build assets, global edge caching, gzip/brotli |
| GPU Compute | CUDA 12 + cuBLAS/cuSPARSE | NVIDIA T4 (16GB, $0.35/h spot) / A100 (40GB, $1.20/h) on ECS/EKS |
| Monitoring | Prometheus + Grafana, Sentry | API latency, GPU utilization, solver convergence failures, error tracking |
| CI/CD | GitHub Actions | Rust tests, `cargo clippy`, WASM build, Docker multi-stage, ECR push |

### Key Architecture Decisions

1. **Rust/Axum as primary backend with Python FastAPI ONLY for ML/scientific libs**: The main API (auth, CRUD, simulation orchestration, job management, billing) is pure Rust/Axum for type safety, memory safety, and performance. Python FastAPI runs as a lightweight internal microservice exclusively for EIS fitting (leveraging mature `lmfit` and `impedance.py` libraries) and cycling data parsing (existing Arbin/Neware parsers). This architecture delivers <10ms API latencies for 95% of endpoints while still accessing the Python scientific ecosystem where Rust alternatives don't exist or aren't battle-tested.

2. **Custom P2D solver in Rust with CUDA rather than wrapping PyBaMM**: Building a from-scratch Newman model solver in Rust gives complete control over GPU acceleration (via `cudarc` CUDA bindings), WASM compilation for in-browser execution, and real-time progress streaming via WebSocket. PyBaMM's heavy Python dependencies (CasADi symbolic math, SUNDIALS ODE solvers, 80+ transitive deps) create 100MB+ bundles incompatible with WASM and introduce unpredictable GIL contention. Our Rust solver uses `faer-sparse` for CPU linear algebra and custom CUDA kernels for GPU, achieving 10-50x speedups while compiling to <1MB WASM with deterministic performance.

3. **Hybrid client/server execution with WASM threshold for simple cases**: Discharge simulations with single-layer electrodes, constant-current protocols, and standard NMC/graphite chemistries run entirely in-browser via WASM with instant (<30s) results and zero server cost, covering 80%+ of initial design iterations. Complex simulations (multi-layer graded electrodes, CCCV protocols with CV taper, SEI growth degradation, 100+ run DOE sweeps, pack-level thermal coupling) are dispatched to Rust-native GPU workers via Redis priority queue. Free plan users get WASM only; Researcher/Team plans unlock GPU server with monthly hour quotas and overage billing.

4. **TimescaleDB hypertables for cycling data time-series at massive scale**: Battery cycling data produces (timestamp, voltage, current, capacity, temperature) tuples at 0.1-1Hz over 500-2000 cycles per cell, yielding millions of rows per cell and billions across a testing program. TimescaleDB automatically partitions by time into chunks, compresses old chunks 10-20x with columnar encoding, and accelerates range queries (e.g., "cycles 50-100 for cell batch X") by 50x via chunk exclusion vs vanilla PostgreSQL sequential scans. This enables storing 100K+ cells without query latency degradation. Cycle-level aggregates (capacity, CE, DCR) live in separate relational table for fast dashboard queries.

5. **Materials database with validated electrochemical parameters from peer-reviewed literature**: The P2D solver requires 20+ parameters per electrode material: solid diffusivity D_s (m²/s), electronic conductivity σ_s (S/m), particle radius R_s (µm), max lithium concentration c_s_max (mol/m³), exchange current density i_0 (A/m²), charge transfer coefficients α_a/α_c, activation energy E_a (kJ/mol), plus full OCP curve U(SOC) as 50-100 point lookup table. We seed a curated global library of 10 materials (NMC811, NMC622, NMC523, LFP, NCA, graphite, silicon, hard carbon, LP30 electrolyte, Celgard 2325 separator) with parameters extracted from published datasets (e.g., Chen 2020 for NMC811, Ecker 2015 for graphite), full DOI citations stored in `source_reference` field, and validation discharge curves against experimental data. Users can clone verified materials to org workspace and customize for proprietary chemistries (Team plan only).

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "timescaledb";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";  -- Full-text search on material names

-- Users
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email TEXT UNIQUE NOT NULL,
    password_hash TEXT,  -- NULL for OAuth-only users
    name TEXT NOT NULL,
    avatar_url TEXT,
    auth_provider TEXT DEFAULT 'email',  -- email | google | github
    auth_provider_id TEXT,
    plan TEXT NOT NULL DEFAULT 'free',  -- free | researcher | team
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    orcid_id TEXT,  -- For researchers
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX users_email_idx ON users(email);
CREATE INDEX users_stripe_idx ON users(stripe_customer_id);

-- Organizations (Team plan)
CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    slug TEXT UNIQUE NOT NULL,
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    plan TEXT NOT NULL DEFAULT 'free',
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    gpu_hours_limit INTEGER DEFAULT 50,  -- Monthly quota
    settings JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX orgs_slug_idx ON organizations(slug);

-- Projects
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    chemistry_tag TEXT,  -- "NMC811/Graphite", "LFP/LTO", etc.
    status TEXT DEFAULT 'active',
    settings JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);

-- Materials (global + org-specific)
CREATE TABLE materials (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    org_id UUID REFERENCES organizations(id) ON DELETE CASCADE,  -- NULL = global
    name TEXT NOT NULL,
    type TEXT NOT NULL,  -- cathode | anode | electrolyte | separator
    category TEXT,  -- NMC | LFP | graphite | silicon
    properties JSONB NOT NULL,  -- {diffusivity, conductivity, cs_max, ...}
    ocp_data JSONB,  -- [{soc, voltage}]
    kinetic_params JSONB,  -- {i0, alpha_a, alpha_c, Ea}
    source_reference TEXT,  -- DOI
    is_public BOOLEAN DEFAULT false,
    is_verified BOOLEAN DEFAULT false,
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX materials_type_idx ON materials(type);

-- Cell Designs
CREATE TABLE cell_designs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    cell_format TEXT NOT NULL DEFAULT 'pouch',
    cathode_material_id UUID REFERENCES materials(id),
    anode_material_id UUID REFERENCES materials(id),
    electrolyte_id UUID REFERENCES materials(id),
    separator_id UUID REFERENCES materials(id),
    electrode_params JSONB NOT NULL,  -- {cath_thick_um, cath_por, ...}
    cell_geometry JSONB NOT NULL,  -- {width_mm, height_mm, layers}
    calculated_metrics JSONB,  -- {cap_mAh, energy_Wh_kg, np_ratio}
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Simulations
CREATE TABLE simulations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    cell_design_id UUID NOT NULL REFERENCES cell_designs(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    model_type TEXT NOT NULL,  -- p2d | spm
    operating_conditions JSONB NOT NULL,
    solver_config JSONB NOT NULL,
    degradation_config JSONB,
    execution_mode TEXT NOT NULL DEFAULT 'wasm',
    status TEXT NOT NULL DEFAULT 'pending',
    results_url TEXT,
    results_summary JSONB,
    log_url TEXT,
    error_message TEXT,
    runtime_seconds REAL,
    gpu_hours_consumed REAL DEFAULT 0.0,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX sims_project_idx ON simulations(project_id);

-- Cycling Datasets
CREATE TABLE cycling_datasets (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    cell_id_label TEXT NOT NULL,
    cycler_type TEXT NOT NULL,
    raw_file_url TEXT NOT NULL,
    parsed BOOLEAN DEFAULT false,
    metadata JSONB DEFAULT '{}',
    total_cycles INTEGER DEFAULT 0,
    total_time_hours REAL DEFAULT 0.0,
    cycle_metrics JSONB DEFAULT '[]',
    timeseries_table TEXT,  -- Hypertable name
    created_by UUID NOT NULL REFERENCES users(id),
    uploaded_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- EIS Measurements
CREATE TABLE eis_measurements (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    cell_id_label TEXT NOT NULL,
    conditions JSONB NOT NULL,  -- {soc, temp, cycle}
    frequency_data_url TEXT NOT NULL,
    source_format TEXT NOT NULL,
    created_by UUID NOT NULL REFERENCES users(id),
    measured_at TIMESTAMPTZ,
    uploaded_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Usage Records
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,  -- gpu_hours | storage_gb
    quantity REAL NOT NULL,
    period_start DATE NOT NULL,
    period_end DATE NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### Rust SQLx Models

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
    #[serde(skip)]
    pub password_hash: Option<String>,
    pub name: String,
    pub plan: String,
    pub stripe_customer_id: Option<String>,
    pub orcid_id: Option<String>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Material {
    pub id: Uuid,
    pub name: String,
    pub r#type: String,
    pub properties: serde_json::Value,
    pub ocp_data: Option<serde_json::Value>,
    pub is_verified: bool,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Simulation {
    pub id: Uuid,
    pub project_id: Uuid,
    pub name: String,
    pub model_type: String,
    pub execution_mode: String,
    pub status: String,
    pub results_url: Option<String>,
    pub gpu_hours_consumed: f32,
}
```

---

## P2D Solver Architecture Deep-Dive

### Governing Equations

**Solid-phase diffusion (Fick's 2nd law in spherical coordinates):**
```
∂cs/∂t = Ds/r² · ∂/∂r(r² · ∂cs/∂r)
BC: r=0: ∂cs/∂r=0 (symmetry), r=Rs: -Ds·∂cs/∂r=jLi
```

**Electrolyte transport (concentrated solution theory):**
```
∂ce/∂t = ∂/∂x(Deff · ∂ce/∂x) + (1-t+)/F · jLi
∂φe/∂x = -jLi/κeff - 2RT(1-t+)/F · ∂ln(ce)/∂x
```

**Butler-Volmer kinetics:**
```
jLi = i0 · [exp(αa·F·η/RT) - exp(-αc·F·η/RT)]
η = φs - φe - U(cs_surf)
```

Discretization: FVM x-direction, FD r-direction, backward Euler time.

### WASM Pipeline

```toml
# solver-wasm/Cargo.toml
[package]
name = "voltvault-solver-wasm"
edition = "2021"
[lib]
crate-type = ["cdylib"]
[dependencies]
wasm-bindgen = "0.2"
faer = { version = "0.18", features = ["sparse"] }
[profile.release]
opt-level = "z"
lto = true
```

---

## Architecture Deep-Dives

### 1. Simulation API (Rust/Axum)

```rust
// src/api/handlers/simulation.rs
use axum::{extract::{Path, State, Json}, http::StatusCode};
use uuid::Uuid;

pub async fn create_simulation(
    State(state): State<AppState>,
    claims: Claims,
    Path((proj_id, design_id)): Path<(Uuid, Uuid)>,
    Json(req): Json<CreateSimRequest>,
) -> Result<Json<Simulation>, ApiError> {
    let design = sqlx::query_as!(CellDesign, "...")
        .fetch_one(&state.db).await?;

    let mode = if is_simple(&req) { "wasm" } else { "server" };

    let sim = sqlx::query_as!(Simulation, "INSERT ...")
        .fetch_one(&state.db).await?;

    if mode == "server" {
        state.redis.zadd("gpu_jobs", sim.id.to_string(), priority).await?;
    }

    Ok(Json(sim))
}
```

### 2. P2D Solver (Rust)

```rust
// solver-core/src/p2d/mod.rs
pub struct P2DSystem {
    pub n_x: usize,
    pub n_r: usize,
    pub cs: Vec<Vec<f64>>,
    pub ce: Vec<f64>,
    pub phi_s: Vec<f64>,
    pub phi_e: Vec<f64>,
}

impl P2DSystem {
    pub fn step(&mut self, dt: f64, I: f64) -> Result<()> {
        for _ in 0..50 {
            self.build_jacobian(dt, I)?;
            let dx = solve_sparse(&self.J, &self.R)?;
            self.update(&dx);
            if self.converged() { return Ok(()); }
        }
        Err(SolverError::Convergence)
    }
}
```

### 3. GPU Worker (Rust+CUDA)

```rust
// src/workers/gpu.rs
use cudarc::driver::CudaDevice;

pub struct GpuWorker {
    db: PgPool,
    redis: RedisClient,
    cuda: CudaDevice,
}

impl GpuWorker {
    pub async fn run(&self) -> Result<()> {
        loop {
            let jid = self.redis.bzpopmax("gpu_jobs", 30).await?;
            self.process(jid).await?;
        }
    }
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + DB (Days 1–5)

**Day 1: Rust backend scaffold**
- Init: `cargo new voltvault-api`
- Deps: `axum, tokio, sqlx, uuid, serde, bcrypt, aws-sdk-s3, redis, cudarc`
- Files: `main.rs` (Axum), `config.rs`, `state.rs`, `error.rs`
- Docker: multi-stage with CUDA base
- Compose: Postgres+TimescaleDB, Redis, MinIO

**Day 2: Database migrations**
- `migrations/001_initial.sql` — all tables
- `src/db/models.rs` — SQLx structs
- Seed: 10 materials (NMC811, LFP, graphite, Si, LP30)

**Day 3: Auth system**
- `src/auth/mod.rs` — JWT gen/validate
- `src/auth/oauth.rs` — Google/GitHub
- `src/api/handlers/auth.rs` — register, login
- Bcrypt cost 12, 24h tokens

**Day 4: User/project CRUD**
- `src/api/handlers/users.rs`
- `src/api/handlers/projects.rs`
- `src/api/handlers/orgs.rs`
- `src/api/router.rs`

**Day 5: Materials API**
- `src/api/handlers/materials.rs` — search, get, clone
- Validation per type
- pg_trgm full-text search

### Phase 2 — P2D Solver Core (Days 6–14)

**Day 6: Framework**
- `solver-core/src/p2d/mod.rs` — P2DSystem
- `solver-core/src/p2d/discretization.rs` — FVM mesh
- `solver-core/src/materials.rs`

**Day 7: Solid diffusion**
- `solid_diffusion.rs` — Fick spherical
- FD r-direction, backward Euler

**Day 8: Electrolyte transport**
- `electrolyte.rs` — concentration + migration
- Bruggeman correction

**Day 9: Butler-Volmer**
- `kinetics.rs` — BV equation, OCP lookup

**Day 10: System assembly**
- `system.rs` — residual + Jacobian
- Sparse COO → CSR

**Day 11: Newton solver**
- `newton.rs` — iteration, damping
- LU via `faer`

**Day 12: Timestepping**
- `protocol.rs` — CC, CCCV, Pulse
- Adaptive dt, cutoffs

**Day 13: OCP curves**
- `materials/ocp.rs` — interpolation
- Default curves from lit

**Day 14: SPM solver**
- `spm/mod.rs` — single particle
- Eigenfunction expansion

### Phase 3 — WASM + Frontend (Days 15–21)

**Day 15: WASM compile**
- `solver-wasm/` workspace
- `wasm-pack build --release`
- `wasm-opt -Oz` <1MB
- JS wrapper

**Day 16: Frontend scaffold**
- Vite React+TS
- Zustand, React Query, axios, recharts, d3
- Router, auth, layout

**Day 17: Cell designer — stack builder**
- SVG electrode visualization
- Material selector dropdowns
- Electrode inputs

**Day 18: Calculated metrics**
- Capacity, energy density, N/P ratio
- Real-time updates (debounced)

**Day 19: Material properties panel**
- Display electrochemical props
- OCP curve D3 chart
- DOI citations

**Day 20: Save/load + templates**
- Cell design API
- Auto-save 2s
- Template gallery

**Day 21: Simulation workspace skeleton**
- Config panel (protocol, C-rate)
- Solver config
- Run button, model selector

### Phase 4 — Simulation Execution + Results (Days 22–28)

**Day 22: WASM execution**
- Load WASM, call solver
- Design → solver inputs
- Error handling

**Day 23: Server submission**
- WebSocket progress
- React hook
- Progress bar + ETA

**Day 24: Voltage/capacity plots**
- Recharts line charts
- Time vs capacity toggle
- Zoom, pan, export

**Day 25: Spatial profiles**
- Concentration/potential vs position
- Time slider animation

**Day 26: Comparison overlay**
- Multi-simulation overlay
- Metrics table

**Day 27: SPM screening**
- Preset C-rates parallel
- Ragone plot

**Day 28: Export + reports**
- CSV export
- PDF generation

### Phase 5 — Cycling Data (Days 29–33)

**Day 29: Python parser microservice**
- `python-services/` FastAPI
- Arbin CSV, generic CSV
- Rust calls via HTTP

**Day 30: TimescaleDB storage**
- Hypertable per dataset
- Partitioning, compression
- Continuous aggregates

**Day 31: Viewer UI**
- Dataset list
- Interactive plots
- Cycle selector

**Day 32: Metrics extraction**
- Python worker
- Cycle detection
- Cap, CE, DCR

**Day 33: Comparison**
- Multi-dataset UI
- Capacity fade overlay
- Stats summary

### Phase 6 — EIS Fitting (Days 34–36)

**Day 34: Upload + viz**
- CSV/.mpr upload
- D3 Nyquist plot
- D3 Bode plots

**Day 35: Python fitting service**
- `impedance.py` + `lmfit`
- Randles, Randles+Warburg
- LM optimizer

**Day 36: Fitting UI**
- Circuit selector
- Fit button
- Parameter display
- Overlay fitted model

### Phase 7 — Billing (Days 37–39)

**Day 37: Stripe integration**
- Checkout, portal
- Webhooks
- Plan mapping

**Day 38: Usage tracking**
- GPU-hours per period
- Limit middleware
- Warnings

**Day 39: Billing UI**
- Plan table
- Usage meter
- Feature gates

### Phase 8 — Testing + Launch (Days 40–42)

**Day 40: Validation**
- Doyle/Newman curves
- vs PyBaMM <2%
- High C-rate

**Day 41: Integration tests**
- E2E flow
- Concurrent GPU
- WASM perf <30s
- Load test 100 users

**Day 42: Polish + launch**
- UI polish
- Onboarding tour
- Docs, video
- Seed materials
- Deploy (ECS, CloudFront, RDS)
- Beta (r/batteries, LinkedIn)

---

## API Design

### Core Endpoints

```
Auth:
POST   /api/auth/register
POST   /api/auth/login
POST   /api/auth/oauth/{provider}
POST   /api/auth/refresh
GET    /api/auth/me

Projects:
GET    /api/projects
POST   /api/projects
GET    /api/projects/:id
PATCH  /api/projects/:id
DELETE /api/projects/:id

Materials:
GET    /api/materials
GET    /api/materials/:id
POST   /api/materials
PATCH  /api/materials/:id
POST   /api/materials/:id/clone

Cell Designs:
POST   /api/projects/:id/cell-designs
GET    /api/projects/:id/cell-designs
GET    /api/cell-designs/:id
PATCH  /api/cell-designs/:id
POST   /api/cell-designs/:id/calculate-metrics
POST   /api/cell-designs/:id/ragone

Simulations:
POST   /api/projects/:id/simulations
GET    /api/projects/:id/simulations
GET    /api/simulations/:id
GET    /api/simulations/:id/results
POST   /api/simulations/:id/cancel
WS     /ws/simulations/:id/progress

Cycling Data:
POST   /api/projects/:id/cycling-data
GET    /api/projects/:id/cycling-data
GET    /api/cycling-data/:id
GET    /api/cycling-data/:id/degradation
POST   /api/cycling-data/compare

EIS:
POST   /api/projects/:id/eis
GET    /api/projects/:id/eis
GET    /api/eis/:id
POST   /api/eis/:id/fit

Billing:
POST   /api/billing/checkout
POST   /api/billing/portal
GET    /api/billing/subscription
POST   /api/webhooks/stripe

Usage:
GET    /api/usage
GET    /api/usage/history
```

---

## Deployment Architecture

### Infrastructure Stack

```
┌─────────────────┐
│   CloudFront    │  CDN (global edge, gzip/brotli)
│   (Frontend)    │
└────────┬────────┘
         │
┌────────▼────────┐
│   S3 Bucket     │  Static React build
│ (voltvault-app) │
└─────────────────┘

┌─────────────────┐
│  CloudFront     │  WASM bundle delivery
│  (WASM solver)  │
└────────┬────────┘
         │
┌────────▼────────┐
│   S3 Bucket     │  solver.wasm, JS glue
│ (voltvault-wasm)│
└─────────────────┘

┌─────────────────┐
│  ALB (HTTPS)    │  TLS termination, health checks
└────────┬────────┘
         │
┌────────▼────────────────────┐
│  ECS Fargate Cluster        │
│  ┌──────────────────────┐   │
│  │ Axum API (Rust)      │   │
│  │ - 4 vCPU, 8GB RAM    │   │
│  │ - Auto-scale 2-20    │   │
│  │ - Health: /health    │   │
│  └──────────────────────┘   │
│                              │
│  ┌──────────────────────┐   │
│  │ GPU Workers (Rust)   │   │
│  │ - T4 GPU instances   │   │
│  │ - ECS GPU Operator   │   │
│  │ - Scale on queue     │   │
│  └──────────────────────┘   │
└──────────────────────────────┘

┌─────────────────────────────┐
│  Python FastAPI Service     │
│  (EIS fitting, parsers)     │
│  - 2 vCPU, 4GB RAM          │
│  - Internal ALB only        │
└─────────────────────────────┘

┌─────────────────┐
│  RDS PostgreSQL │  Primary database
│  + TimescaleDB  │  - r6g.xlarge (4vCPU, 32GB)
│                 │  - Multi-AZ
│                 │  - Read replica
└─────────────────┘

┌─────────────────┐
│  ElastiCache    │  Redis cluster
│  for Redis      │  - cache.r6g.large
│                 │  - Cluster mode
└─────────────────┘

┌─────────────────┐
│  S3 Buckets     │
│  - voltvault-results
│  - voltvault-cycling-data
│  - voltvault-eis
└─────────────────┘
```

### Docker Multi-Stage Build

```dockerfile
# Dockerfile

# Stage 1: Build Rust backend
FROM rust:1.75 as rust-builder
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
COPY src ./src
COPY solver-core ./solver-core
RUN cargo build --release

# Stage 2: CUDA runtime for GPU workers
FROM nvidia/cuda:12.2.0-runtime-ubuntu22.04 as gpu-worker
RUN apt-get update && apt-get install -y ca-certificates
COPY --from=rust-builder /app/target/release/voltvault-worker /usr/local/bin/
CMD ["voltvault-worker"]

# Stage 3: API runtime
FROM debian:bookworm-slim as api
RUN apt-get update && apt-get install -y ca-certificates libpq5 && rm -rf /var/lib/apt/lists/*
COPY --from=rust-builder /app/target/release/voltvault-api /usr/local/bin/
EXPOSE 8080
CMD ["voltvault-api"]
```

### Kubernetes GPU Job Spec

```yaml
# k8s/gpu-worker.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gpu-worker
spec:
  replicas: 2
  selector:
    matchLabels:
      app: gpu-worker
  template:
    metadata:
      labels:
        app: gpu-worker
    spec:
      nodeSelector:
        node.kubernetes.io/instance-type: g4dn.xlarge  # T4 GPU
      containers:
      - name: worker
        image: voltvault/gpu-worker:latest
        resources:
          limits:
            nvidia.com/gpu: 1
            memory: 16Gi
            cpu: 4
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: url
        - name: REDIS_URL
          value: redis://redis-cluster:6379
        - name: S3_BUCKET
          value: voltvault-results
```

---

## Monitoring and Observability

### Prometheus Metrics

```rust
// src/metrics.rs
use prometheus::{IntCounter, IntGauge, Histogram, Registry};
use lazy_static::lazy_static;

lazy_static! {
    pub static ref REGISTRY: Registry = Registry::new();

    pub static ref HTTP_REQUESTS_TOTAL: IntCounter = IntCounter::new(
        "http_requests_total",
        "Total HTTP requests"
    ).unwrap();

    pub static ref SIMULATIONS_RUNNING: IntGauge = IntGauge::new(
        "simulations_running",
        "Currently running simulations"
    ).unwrap();

    pub static ref GPU_JOBS_QUEUE_DEPTH: IntGauge = IntGauge::new(
        "gpu_jobs_queue_depth",
        "Number of GPU jobs in queue"
    ).unwrap();

    pub static ref SIMULATION_DURATION: Histogram = Histogram::with_opts(
        prometheus::HistogramOpts::new(
            "simulation_duration_seconds",
            "Simulation execution time"
        ).buckets(vec![1.0, 5.0, 10.0, 30.0, 60.0, 300.0, 600.0])
    ).unwrap();

    pub static ref SOLVER_CONVERGENCE_FAILURES: IntCounter = IntCounter::new(
        "solver_convergence_failures_total",
        "Total solver convergence failures"
    ).unwrap();
}

pub fn register_metrics() {
    REGISTRY.register(Box::new(HTTP_REQUESTS_TOTAL.clone())).unwrap();
    REGISTRY.register(Box::new(SIMULATIONS_RUNNING.clone())).unwrap();
    REGISTRY.register(Box::new(GPU_JOBS_QUEUE_DEPTH.clone())).unwrap();
    REGISTRY.register(Box::new(SIMULATION_DURATION.clone())).unwrap();
    REGISTRY.register(Box::new(SOLVER_CONVERGENCE_FAILURES.clone())).unwrap();
}
```

### Grafana Dashboard JSON

```json
{
  "dashboard": {
    "title": "VoltVault Production",
    "panels": [
      {
        "title": "API Request Rate",
        "targets": [
          {
            "expr": "rate(http_requests_total[5m])"
          }
        ]
      },
      {
        "title": "GPU Utilization",
        "targets": [
          {
            "expr": "nvidia_gpu_utilization"
          }
        ]
      },
      {
        "title": "Simulation Queue Depth",
        "targets": [
          {
            "expr": "gpu_jobs_queue_depth"
          }
        ]
      },
      {
        "title": "P95 Simulation Time",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, simulation_duration_seconds_bucket)"
          }
        ]
      },
      {
        "title": "Solver Convergence Rate",
        "targets": [
          {
            "expr": "1 - rate(solver_convergence_failures_total[5m]) / rate(simulations_total[5m])"
          }
        ]
      }
    ]
  }
}
```

### Sentry Error Tracking

```rust
// src/main.rs
use sentry;

#[tokio::main]
async fn main() {
    let _guard = sentry::init((
        std::env::var("SENTRY_DSN").unwrap(),
        sentry::ClientOptions {
            release: sentry::release_name!(),
            environment: Some(std::env::var("ENV").unwrap_or("production".into()).into()),
            ..Default::default()
        },
    ));

    // Run app
    run_server().await;
}
```

---

## Testing Strategy

### Unit Tests (Rust)

```rust
// solver-core/src/p2d/tests.rs
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_fick_diffusion_steady_state() {
        let mut system = P2DSystem::new(
            get_nmc811(),
            get_graphite(),
            get_celgard(),
            get_lp30(),
            20,
            10,
        );
        system.initialize(0.5, 0.5);

        // Run until steady state
        for _ in 0..1000 {
            system.step(0.1, 0.0).unwrap();
        }

        // At zero current, concentration should be uniform
        for i in 0..system.n_x {
            let cs_avg: f64 = system.cs[i].iter().sum::<f64>() / system.n_r as f64;
            assert!((cs_avg - system.cs[i][0]).abs() < 1e-6);
        }
    }

    #[test]
    fn test_butler_volmer_symmetry() {
        let i0 = 1e-3;
        let alpha = 0.5;
        let eta_pos = 0.1;
        let eta_neg = -0.1;

        let j_pos = butler_volmer(i0, alpha, alpha, eta_pos, 298.15);
        let j_neg = butler_volmer(i0, alpha, alpha, eta_neg, 298.15);

        assert!((j_pos + j_neg).abs() < 1e-10);
    }

    #[test]
    fn test_p2d_vs_analytical_capacity() {
        // Simple constant-current discharge
        let mut system = create_reference_cell();
        let capacity_theoretical = 50.0;  // Ah

        let mut capacity_simulated = 0.0;
        let current = 50.0;  // 1C
        let mut t = 0.0;
        let dt = 1.0;

        while system.get_voltage() > 2.5 {
            system.step(dt, current).unwrap();
            capacity_simulated += current * dt / 3600.0;
            t += dt;
        }

        let error = (capacity_simulated - capacity_theoretical).abs() / capacity_theoretical;
        assert!(error < 0.02, "Capacity error {}% exceeds 2%", error * 100.0);
    }
}
```

### Integration Tests (API)

```rust
// tests/integration/simulation.rs
use sqlx::PgPool;
use axum::http::StatusCode;

#[tokio::test]
async fn test_create_simulation_flow() {
    let pool = setup_test_db().await;
    let app = create_test_app(pool.clone()).await;

    // Create user
    let user = create_test_user(&pool).await;
    let token = generate_test_token(&user);

    // Create project
    let project = create_test_project(&pool, user.id).await;

    // Create cell design
    let design = create_test_cell_design(&pool, project.id).await;

    // Create simulation
    let req = serde_json::json!({
        "model_type": "p2d",
        "operating_conditions": {
            "c_rate": 1.0,
            "v_min": 2.5,
            "v_max": 4.2,
            "temperature": 298.15,
            "protocol": "constant_current"
        },
        "solver_config": {
            "n_x": 20,
            "n_r": 10,
            "dt_init": 1.0,
            "abstol": 1e-6,
            "reltol": 1e-3,
            "max_iterations": 50
        }
    });

    let response = app
        .post(&format!("/api/projects/{}/cell-designs/{}/simulations", project.id, design.id))
        .header("Authorization", format!("Bearer {}", token))
        .json(&req)
        .await;

    assert_eq!(response.status(), StatusCode::CREATED);

    let sim: Simulation = response.json().await;
    assert_eq!(sim.status, "pending");
    assert_eq!(sim.execution_mode, "wasm");  // Simple case
}
```

### E2E Tests (Playwright)

```typescript
// e2e/simulation-flow.spec.ts
import { test, expect } from '@playwright/test';

test('complete simulation workflow', async ({ page }) => {
  // Login
  await page.goto('/login');
  await page.fill('[name=email]', 'test@example.com');
  await page.fill('[name=password]', 'password123');
  await page.click('button[type=submit]');
  await expect(page).toHaveURL('/dashboard');

  // Create project
  await page.click('button:has-text("New Project")');
  await page.fill('[name=name]', 'Test NMC Cell');
  await page.click('button:has-text("Create")');

  // Create cell design
  await page.click('a:has-text("Cell Designer")');
  await page.selectOption('[name=cathode_material]', 'NMC811');
  await page.selectOption('[name=anode_material]', 'Graphite');
  await page.fill('[name=cathode_thickness_um]', '50');
  await page.fill('[name=anode_thickness_um]', '40');

  // Verify metrics calculated
  await expect(page.locator('[data-testid=capacity]')).toContainText('50.0 mAh');

  // Run simulation
  await page.click('button:has-text("Run Simulation")');
  await page.selectOption('[name=model_type]', 'p2d');
  await page.fill('[name=c_rate]', '1.0');
  await page.click('button:has-text("Start")');

  // Wait for WASM simulation to complete (should be fast)
  await expect(page.locator('[data-testid=status]')).toHaveText('Completed', { timeout: 60000 });

  // Verify voltage plot rendered
  await expect(page.locator('canvas[data-testid=voltage-plot]')).toBeVisible();

  // Export CSV
  await page.click('button:has-text("Export")');
  const [download] = await Promise.all([
    page.waitForEvent('download'),
    page.click('a:has-text("Download CSV")')
  ]);
  expect(download.suggestedFilename()).toContain('.csv');
});
```

### Load Testing (k6)

```javascript
// load-tests/api-load.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export let options = {
  stages: [
    { duration: '2m', target: 100 },  // Ramp up to 100 users
    { duration: '5m', target: 100 },  // Stay at 100 users
    { duration: '2m', target: 200 },  // Ramp to 200 users
    { duration: '5m', target: 200 },
    { duration: '2m', target: 0 },    // Ramp down
  ],
  thresholds: {
    'http_req_duration': ['p(95)<500'],  // 95% of requests < 500ms
    'http_req_failed': ['rate<0.01'],     // <1% failure rate
  },
};

const BASE_URL = 'https://api.voltvault.io';

export function setup() {
  // Login and get token
  const loginRes = http.post(`${BASE_URL}/api/auth/login`, JSON.stringify({
    email: 'loadtest@example.com',
    password: 'loadtest123'
  }), {
    headers: { 'Content-Type': 'application/json' }
  });
  return { token: loginRes.json('access_token') };
}

export default function(data) {
  const headers = {
    'Authorization': `Bearer ${data.token}`,
    'Content-Type': 'application/json'
  };

  // List projects
  let res = http.get(`${BASE_URL}/api/projects`, { headers });
  check(res, { 'list projects status 200': (r) => r.status === 200 });

  sleep(1);

  // Get materials
  res = http.get(`${BASE_URL}/api/materials`, { headers });
  check(res, { 'list materials status 200': (r) => r.status === 200 });

  sleep(1);
}
```

---

## Go-to-Market Strategy

### Phase 1: Research Community Launch (Month 1-2)

**Channels:**
- Hacker News "Show HN: VoltVault – GPU-accelerated battery simulation in your browser"
- Reddit: r/batteries (130K subscribers), r/electrochemistry (18K), r/engineering (1.2M)
- Battery forums: Electrochemical Society (ECS) discussion boards, Battery University forums
- Academic Twitter/X: Thread explaining P2D model democratization with WASM demo GIF
- LinkedIn: Post in "Battery Technology", "Electric Vehicles", "Energy Storage" groups

**Content:**
- Blog post: "Implementing the Newman P2D Model in Rust with WASM: 10x Faster Than Python"
- Technical validation paper: "VoltVault P2D Solver Benchmarks vs. PyBaMM" with discharge curve comparisons
- Interactive demo: Embeddable WASM cell designer widget for education websites
- YouTube video: "Design a Lithium-Ion Cell in 10 Minutes" screencast (target 50K views)

**Partnerships:**
- Reach out to 20 battery research groups at top universities (MIT, Stanford, CMU, Georgia Tech, UCSD)
- Offer free academic accounts with citation requirement in published papers
- Co-author validation study with national lab (NREL, Argonne, Oak Ridge) for credibility

**Target:** 1,000 registered users, 100 active weekly, 5,000 P2D simulations run

### Phase 2: Industry Adoption (Month 3-6)

**Outbound Sales:**
- LinkedIn Sales Navigator: Target "Battery Engineer", "Cell Engineer", "Electrochemist" at EV OEMs, battery manufacturers
- Cold email campaign: 500 battery engineers with personalized "your recent publication on [chemistry] could be accelerated with VoltVault"
- Conference booth: The Battery Show (Novi, MI – 15K attendees), Advanced Automotive Battery Conference (AABC)

**Product Marketing:**
- Case study: "[EV Startup] Reduced Cell Design Iteration Time from 6 Weeks to 3 Days"
- Webinar series: "From Cycling Data to Validated P2D Model in 1 Hour" with Arbin/Neware partnership announcement
- SEO content: "Battery Simulation Software", "P2D Model Online", "COMSOL Alternative", "PyBaMM GUI"

**Integrations:**
- Arbin data format direct import partnership (co-marketing)
- Export P2D parameters to MATLAB Simulink for system-level pack simulation
- API for lab notebooks (Benchling, LabArchives) to embed simulation results

**Target:** 2,500 total users, 40 paying Researcher plans ($149/mo), $6,000 MRR

### Phase 3: Enterprise Scaling (Month 7-12)

**Enterprise Features Launch:**
- SSO/SAML for automotive OEMs (Ford, GM, Tesla battery teams)
- Dedicated GPU clusters with SLA (99.9% uptime, <5min queue time)
- On-premise deployment option for IP-sensitive R&D (Toyota, Samsung SDI)
- LIMS integration (Arbin MITS, Neware BTS) for automatic cycling data sync

**Strategic Accounts:**
- Hire 2 AEs targeting automotive OEMs, Tier 1 battery manufacturers
- Engage procurement at Ford, GM, Rivian, Lucid for pilot programs
- Partner with battery consultancies (Benchmark Mineral Intelligence, Wood Mackenzie) to offer VoltVault in client deliverables

**Thought Leadership:**
- Keynote at ECS meeting: "Cloud-Native Electrochemical Simulation: Lessons from 50,000 P2D Runs"
- Nature Energy commentary: "Democratizing Battery Simulation for the Climate Transition"
- Battery Materials Review article: "Comparing Physics-Based and Data-Driven Approaches to Cell Degradation Prediction"

**Target:** 5,000 total users, 80 paying (60 Researcher, 20 Team), $18,000 MRR, 2 enterprise pilots

---

## UI/UX Design — Key Screens

### 1. Project Dashboard

**Layout:**
- Top nav: VoltVault logo, search bar, user avatar dropdown (profile, billing, logout)
- Left sidebar: Projects (active/archived), Materials Library, Help/Docs
- Main area: Project grid (3 columns on desktop, 1 on mobile)

**Project Cards:**
```tsx
// ProjectCard.tsx
export function ProjectCard({ project }: { project: Project }) {
  return (
    <div className="border rounded-lg p-4 hover:shadow-lg transition">
      <div className="flex justify-between items-start">
        <div>
          <h3 className="font-semibold text-lg">{project.name}</h3>
          <p className="text-sm text-gray-600">{project.chemistry_tag}</p>
        </div>
        <Badge>{project.status}</Badge>
      </div>

      <div className="mt-4 grid grid-cols-3 gap-2 text-sm">
        <div>
          <p className="text-gray-500">Cell Designs</p>
          <p className="font-medium">{project.cell_design_count}</p>
        </div>
        <div>
          <p className="text-gray-500">Simulations</p>
          <p className="font-medium">{project.simulation_count}</p>
        </div>
        <div>
          <p className="text-gray-500">Last Updated</p>
          <p className="font-medium">{formatDate(project.updated_at)}</p>
        </div>
      </div>

      <div className="mt-4 flex gap-2">
        <Button variant="primary" onClick={() => navigate(`/projects/${project.id}`)}>
          Open
        </Button>
        <Button variant="secondary" onClick={() => handleArchive(project.id)}>
          Archive
        </Button>
      </div>
    </div>
  );
}
```

**Usage Meter:**
- GPU hours used this month: progress bar (green <80%, yellow 80-95%, red >95%)
- Storage used: GB out of plan limit
- "Upgrade Plan" button when approaching limits

### 2. Cell Designer

**Three-Column Layout:**

**Left Panel (Materials & Templates):**
```tsx
// MaterialSelector.tsx
export function MaterialSelector({ type, value, onChange }) {
  const { data: materials } = useQuery(['materials', type], () =>
    api.getMaterials({ type })
  );

  return (
    <div className="space-y-2">
      <label className="font-medium">{type === 'cathode' ? 'Cathode Material' : 'Anode Material'}</label>
      <Select value={value} onChange={onChange}>
        <option value="">Select material...</option>
        {materials?.map(m => (
          <option key={m.id} value={m.id}>
            {m.name} {m.is_verified && '✓'}
          </option>
        ))}
      </Select>

      {value && (
        <Button variant="link" onClick={() => setShowMaterialDetails(true)}>
          View Properties
        </Button>
      )}
    </div>
  );
}
```

**Center Panel (Electrode Stack Visualization):**
```tsx
// ElectrodeStackSVG.tsx
export function ElectrodeStackSVG({ design }: { design: CellDesign }) {
  const totalThickness =
    design.electrode_params.cathode_thickness_um +
    design.electrode_params.separator_thickness_um +
    design.electrode_params.anode_thickness_um;

  const scale = 400 / totalThickness;  // Scale to 400px height

  let yOffset = 0;

  return (
    <svg width="600" height="500" className="border rounded">
      {/* Current Collector (Aluminum) */}
      <rect x="50" y="50" width="500" height="20" fill="#C0C0C0" />
      <text x="560" y="65" fontSize="12">Al CC</text>

      {/* Cathode Coating */}
      <rect
        x="50"
        y="70"
        width="500"
        height={design.electrode_params.cathode_thickness_um * scale}
        fill="#2563EB"  // Blue
        opacity="0.7"
      />
      <text x="560" y={70 + design.electrode_params.cathode_thickness_um * scale / 2} fontSize="12">
        Cathode ({design.electrode_params.cathode_thickness_um}µm)
      </text>

      {/* Separator */}
      <rect
        x="50"
        y={70 + design.electrode_params.cathode_thickness_um * scale}
        width="500"
        height={design.electrode_params.separator_thickness_um * scale}
        fill="#FCD34D"  // Yellow
        opacity="0.7"
      />
      <text x="560" y={70 + (design.electrode_params.cathode_thickness_um + design.electrode_params.separator_thickness_um / 2) * scale}>
        Separator ({design.electrode_params.separator_thickness_um}µm)
      </text>

      {/* Anode Coating */}
      <rect
        x="50"
        y={70 + (design.electrode_params.cathode_thickness_um + design.electrode_params.separator_thickness_um) * scale}
        width="500"
        height={design.electrode_params.anode_thickness_um * scale}
        fill="#DC2626"  // Red
        opacity="0.7"
      />
      <text x="560" y={70 + (design.electrode_params.cathode_thickness_um + design.electrode_params.separator_thickness_um + design.electrode_params.anode_thickness_um / 2) * scale}>
        Anode ({design.electrode_params.anode_thickness_um}µm)
      </text>

      {/* Current Collector (Copper) */}
      <rect
        x="50"
        y={70 + totalThickness * scale}
        width="500"
        height="20"
        fill="#B45309"  // Copper
      />
      <text x="560" y={85 + totalThickness * scale}>Cu CC</text>
    </svg>
  );
}
```

**Right Panel (Calculated Metrics):**
```tsx
// MetricsCard.tsx
export function MetricsCard({ design }: { design: CellDesign }) {
  const metrics = design.calculated_metrics;

  return (
    <div className="space-y-4">
      <h3 className="font-semibold">Calculated Metrics</h3>

      <div className="grid grid-cols-2 gap-4">
        <MetricDisplay
          label="Capacity"
          value={metrics.capacity_mAh}
          unit="mAh"
          tooltip="Theoretical capacity based on active material loading"
        />
        <MetricDisplay
          label="Energy (gravimetric)"
          value={metrics.energy_Wh_kg}
          unit="Wh/kg"
          tooltip="Energy density at cell level including inactive materials"
        />
        <MetricDisplay
          label="Energy (volumetric)"
          value={metrics.energy_Wh_L}
          unit="Wh/L"
          tooltip="Energy density including cell casing and tabs"
        />
        <MetricDisplay
          label="N/P Ratio"
          value={metrics.np_ratio}
          unit=""
          tooltip="Anode capacity / Cathode capacity (target 1.1-1.2)"
          warn={metrics.np_ratio < 1.05 || metrics.np_ratio > 1.3}
        />
      </div>

      <Button variant="primary" onClick={() => runRagoneAnalysis(design.id)}>
        Generate Ragone Plot
      </Button>
    </div>
  );
}
```

### 3. Simulation Workspace

**Split View:**

**Left Panel (Configuration):**
```tsx
// SimulationConfigPanel.tsx
export function SimulationConfigPanel({ onRun }) {
  const [config, setConfig] = useState<SimConfig>({
    model_type: 'p2d',
    c_rate: 1.0,
    v_min: 2.5,
    v_max: 4.2,
    temperature: 298.15,
    protocol: 'constant_current',
    solver: {
      n_x: 20,
      n_r: 10,
      dt_init: 1.0,
      abstol: 1e-6,
      reltol: 1e-3,
      max_iterations: 50
    }
  });

  return (
    <div className="space-y-4 p-4 border-r">
      <h3 className="font-semibold">Simulation Configuration</h3>

      <div>
        <label>Model Type</label>
        <Select value={config.model_type} onChange={v => setConfig({...config, model_type: v})}>
          <option value="p2d">P2D (Full Newman Model)</option>
          <option value="spm">SPM (Fast Screening)</option>
        </Select>
        <p className="text-xs text-gray-500 mt-1">
          {config.model_type === 'p2d'
            ? 'Full physics model with spatial resolution'
            : 'Simplified model, 10-100x faster but less accurate at high C-rates'}
        </p>
      </div>

      <div>
        <label>Protocol</label>
        <Select value={config.protocol} onChange={v => setConfig({...config, protocol: v})}>
          <option value="constant_current">Constant Current (CC)</option>
          <option value="cccv">CC-CV (with taper)</option>
          <option value="pulse">Pulse (HPPC)</option>
        </Select>
      </div>

      <div>
        <label>C-Rate</label>
        <Input
          type="number"
          step="0.1"
          value={config.c_rate}
          onChange={v => setConfig({...config, c_rate: parseFloat(v)})}
        />
        <p className="text-xs text-gray-500">
          {config.c_rate}C = {(config.c_rate * design.capacity_mAh / 1000).toFixed(2)}A current
        </p>
      </div>

      <div className="grid grid-cols-2 gap-2">
        <div>
          <label>V_min (V)</label>
          <Input type="number" step="0.1" value={config.v_min} onChange={v => setConfig({...config, v_min: parseFloat(v)})} />
        </div>
        <div>
          <label>V_max (V)</label>
          <Input type="number" step="0.1" value={config.v_max} onChange={v => setConfig({...config, v_max: parseFloat(v)})} />
        </div>
      </div>

      <details className="text-sm">
        <summary className="cursor-pointer font-medium">Advanced Solver Settings</summary>
        <div className="mt-2 space-y-2 pl-4">
          <div>
            <label>Spatial Points (n_x)</label>
            <Input type="number" value={config.solver.n_x} onChange={v => setConfig({...config, solver: {...config.solver, n_x: parseInt(v)}})} />
          </div>
          <div>
            <label>Particle Points (n_r)</label>
            <Input type="number" value={config.solver.n_r} onChange={v => setConfig({...config, solver: {...config.solver, n_r: parseInt(v)}})} />
          </div>
          <div>
            <label>Convergence Tolerance</label>
            <Input type="number" step="1e-7" value={config.solver.abstol} onChange={v => setConfig({...config, solver: {...config.solver, abstol: parseFloat(v)}})} />
          </div>
        </div>
      </details>

      <Button
        variant="primary"
        onClick={() => onRun(config)}
        disabled={isRunning}
      >
        {isRunning ? 'Running...' : 'Run Simulation'}
      </Button>

      {executionMode === 'server' && (
        <p className="text-xs text-amber-600">
          ⚠️ This configuration requires server GPU (will consume ~0.1 GPU-hours)
        </p>
      )}
    </div>
  );
}
```

**Right Panel (Results Viewer):**
```tsx
// ResultsViewer.tsx
export function ResultsViewer({ simulation }) {
  const { data: results } = useQuery(
    ['simulation-results', simulation.id],
    () => api.getSimulationResults(simulation.id),
    { enabled: simulation.status === 'completed' }
  );

  const [xAxis, setXAxis] = useState<'time' | 'capacity'>('capacity');

  if (simulation.status === 'running') {
    return <SimulationProgress simulationId={simulation.id} />;
  }

  if (!results) return null;

  const chartData = results.map((pt, i) => ({
    x: xAxis === 'time' ? pt.time : pt.capacity_mAh,
    voltage: pt.voltage,
    current: pt.current,
  }));

  return (
    <div className="space-y-4">
      <div className="flex justify-between items-center">
        <h3 className="font-semibold">Voltage Curve</h3>
        <div className="flex gap-2">
          <Button
            variant={xAxis === 'capacity' ? 'primary' : 'secondary'}
            onClick={() => setXAxis('capacity')}
          >
            vs Capacity
          </Button>
          <Button
            variant={xAxis === 'time' ? 'primary' : 'secondary'}
            onClick={() => setXAxis('time')}
          >
            vs Time
          </Button>
        </div>
      </div>

      <ResponsiveContainer width="100%" height={400}>
        <LineChart data={chartData}>
          <CartesianGrid strokeDasharray="3 3" />
          <XAxis
            dataKey="x"
            label={{ value: xAxis === 'time' ? 'Time (s)' : 'Capacity (mAh)', position: 'insideBottom', offset: -5 }}
          />
          <YAxis label={{ value: 'Voltage (V)', angle: -90, position: 'insideLeft' }} />
          <Tooltip />
          <Line type="monotone" dataKey="voltage" stroke="#2563EB" dot={false} />
        </LineChart>
      </ResponsiveContainer>

      <div className="grid grid-cols-4 gap-4 text-sm">
        <div>
          <p className="text-gray-500">Discharge Capacity</p>
          <p className="font-medium">{simulation.results_summary.capacity_mAh.toFixed(1)} mAh</p>
        </div>
        <div>
          <p className="text-gray-500">Energy</p>
          <p className="font-medium">{simulation.results_summary.energy_Wh.toFixed(2)} Wh</p>
        </div>
        <div>
          <p className="text-gray-500">Avg Voltage</p>
          <p className="font-medium">{(simulation.results_summary.energy_Wh / (simulation.results_summary.capacity_mAh / 1000)).toFixed(2)} V</p>
        </div>
        <div>
          <p className="text-gray-500">Runtime</p>
          <p className="font-medium">{simulation.runtime_seconds.toFixed(1)} s</p>
        </div>
      </div>

      <div className="flex gap-2">
        <Button onClick={() => exportCSV(results)}>Export CSV</Button>
        <Button onClick={() => exportPNG(chartRef)}>Export PNG</Button>
        <Button onClick={() => generateReport(simulation.id)}>Generate Report</Button>
      </div>
    </div>
  );
}
```

### 4. Cycling Data Explorer

**Split View with Dataset List and Plot Area:**

```tsx
// CyclingDataExplorer.tsx
export function CyclingDataExplorer({ projectId }) {
  const { data: datasets } = useQuery(['cycling-datasets', projectId], () =>
    api.getCyclingDatasets(projectId)
  );

  const [selectedDataset, setSelectedDataset] = useState<string | null>(null);
  const [selectedCycle, setSelectedCycle] = useState<number>(1);

  const { data: cycleData } = useQuery(
    ['cycle-data', selectedDataset, selectedCycle],
    () => api.getCycleData(selectedDataset, selectedCycle),
    { enabled: !!selectedDataset }
  );

  return (
    <div className="grid grid-cols-4 gap-4 h-screen">
      {/* Dataset List */}
      <div className="col-span-1 border-r p-4 overflow-y-auto">
        <h3 className="font-semibold mb-4">Cycling Datasets</h3>
        {datasets?.map(ds => (
          <div
            key={ds.id}
            className={`p-3 border rounded mb-2 cursor-pointer hover:bg-gray-50 ${
              selectedDataset === ds.id ? 'bg-blue-50 border-blue-500' : ''
            }`}
            onClick={() => setSelectedDataset(ds.id)}
          >
            <p className="font-medium">{ds.cell_id_label}</p>
            <p className="text-xs text-gray-500">{ds.cycler_type}</p>
            <p className="text-xs text-gray-500">{ds.total_cycles} cycles</p>
          </div>
        ))}
      </div>

      {/* Plot Area */}
      <div className="col-span-3 p-4">
        {selectedDataset && (
          <>
            <div className="flex justify-between items-center mb-4">
              <h3 className="font-semibold">Cycle {selectedCycle}</h3>
              <Slider
                min={1}
                max={datasets.find(d => d.id === selectedDataset)?.total_cycles || 100}
                value={selectedCycle}
                onChange={setSelectedCycle}
                className="w-64"
              />
            </div>

            <ResponsiveContainer width="100%" height={400}>
              <LineChart data={cycleData}>
                <CartesianGrid strokeDasharray="3 3" />
                <XAxis dataKey="capacity_mAh" label={{ value: 'Capacity (mAh)', position: 'insideBottom', offset: -5 }} />
                <YAxis label={{ value: 'Voltage (V)', angle: -90, position: 'insideLeft' }} />
                <Tooltip />
                <Line type="monotone" dataKey="voltage" stroke="#2563EB" dot={false} />
              </LineChart>
            </ResponsiveContainer>

            <div className="mt-6">
              <h4 className="font-medium mb-2">Degradation Metrics</h4>
              <CapacityFadePlot datasetId={selectedDataset} />
            </div>
          </>
        )}
      </div>
    </div>
  );
}
```

### 5. EIS Fitting Studio

```tsx
// EISFittingStudio.tsx
export function EISFittingStudio({ measurementId }) {
  const { data: measurement } = useQuery(['eis', measurementId], () =>
    api.getEISMeasurement(measurementId)
  );

  const [circuitModel, setCircuitModel] = useState('randles');
  const [fit, setFit] = useState(null);

  const handleFit = async () => {
    const result = await api.fitEIS(measurementId, circuitModel);
    setFit(result);
  };

  return (
    <div className="grid grid-cols-2 gap-6 p-6">
      {/* Left: Nyquist Plot */}
      <div>
        <h3 className="font-semibold mb-4">Nyquist Plot</h3>
        <NyquistPlot data={measurement?.frequency_data} fittedData={fit?.fitted_curve} />

        <div className="mt-4">
          <h4 className="font-medium mb-2">Bode Plot</h4>
          <BodePlot data={measurement?.frequency_data} />
        </div>
      </div>

      {/* Right: Fitting Controls */}
      <div>
        <h3 className="font-semibold mb-4">Circuit Model</h3>
        <Select value={circuitModel} onChange={setCircuitModel}>
          <option value="randles">Randles (R_s + R_ct||CPE)</option>
          <option value="randles_warburg">Randles + Warburg</option>
          <option value="dual_rc">Dual RC (Cathode + Anode)</option>
        </Select>

        <Button onClick={handleFit} className="mt-4 w-full">
          Fit Model
        </Button>

        {fit && (
          <div className="mt-6 space-y-4">
            <h4 className="font-medium">Fitted Parameters</h4>
            <table className="w-full text-sm">
              <thead>
                <tr className="border-b">
                  <th className="text-left py-2">Parameter</th>
                  <th className="text-right">Value</th>
                  <th className="text-right">Error</th>
                </tr>
              </thead>
              <tbody>
                {fit.fitted_parameters.map(p => (
                  <tr key={p.element} className="border-b">
                    <td className="py-2">{p.element}</td>
                    <td className="text-right font-mono">{p.value.toExponential(3)}</td>
                    <td className="text-right font-mono text-gray-500">±{p.stderr.toExponential(2)}</td>
                  </tr>
                ))}
              </tbody>
            </table>

            <div className="bg-gray-50 p-3 rounded">
              <p className="text-sm"><strong>χ²:</strong> {fit.fit_quality.chi_squared.toFixed(4)}</p>
              <p className="text-sm"><strong>RMSE:</strong> {fit.fit_quality.rmse.toFixed(4)} Ω</p>
              <p className="text-sm"><strong>R²:</strong> {fit.fit_quality.r_squared.toFixed(4)}</p>
            </div>
          </div>
        )}
      </div>
    </div>
  );
}
```

---

## Post-MVP Roadmap

### v1.1 — DOE + 3D Thermal (Weeks 7–10)
- DOE sweeps (LHS, Sobol)
- Response surfaces
- Sensitivity analysis
- 3D thermal FEM
- Three.js viz

### v1.2 — Degradation + Collaboration (Weeks 11–14)
- SEI growth
- Li plating
- Capacity fade tracking
- Real-time collab
- Pack simulation

### v1.3 — Advanced EIS + Calendar Aging (Weeks 15–18)
- DRT analysis
- Multi-spectrum fitting
- Custom circuit builder
- Calendar aging models

### v2.0 — Enterprise (Months 5–6)
- SSO/SAML
- Dedicated GPUs
- LIMS integration
- Solid-state module
- Na-ion chemistry
- SOC 2

---

## Success Metrics (6 Mo)

| Metric | Target |
|--------|--------|
| Users | 3,000 |
| P2D sims | 12,000 |
| SPM sims | 50,000 |
| GPU-hours | 8,000 |
| Datasets | 4,000 |
| Paying | 60 |
| MRR | $12,000 |
| Conversion | 5% |

---

## Competitive Edge

| Competitor | VoltVault |
|-----------|-----------|
| COMSOL | 100x cheaper, browser, GPU 10x faster, integrated data |
| PyBaMM | GUI no Python, WASM in-browser, data mgmt, collab |
| MATLAB | Purpose-built, 10x cheaper, browser, GPU, exp workflows |
| BatteryArchitect | Physics P2D not empirical, design-stage, EIS |

---

## Risk Mitigation

| Risk | Mitigation |
|------|------------|
| GPU costs | Spot 60% savings, adaptive mesh, credits, SPM first |
| Accuracy questioned | Validate vs PyBaMM, publish, export for check |
| Browser distrust | Docs, export, NREL partnership |
| Cycler formats change | Test suites, monitor, partner, CSV fallback |
| COMSOL cloud | Fast moat (sim+data+EIS), community, portability |
