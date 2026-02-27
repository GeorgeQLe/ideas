# 43. SpiceSim — Cloud-Native Analog/Mixed-Signal Circuit Simulator

## Implementation Plan

**MVP Scope:** Browser-based schematic editor with drag-and-drop component placement (R, L, C, MOSFET, BJT, diode, op-amp, voltage/current sources) and auto-wire routing rendered via SVG, custom SPICE simulation engine implementing Modified Nodal Analysis (MNA) with KLU sparse-matrix direct solver compiled to WebAssembly for client-side execution of circuits ≤500 nodes and server-side Rust-native execution for larger circuits, support for DC operating point, DC sweep, AC small-signal, and transient analyses with advanced convergence (Gmin stepping, source stepping, dynamic timestep via local truncation error), BSIM4 and EKV MOSFET models plus standard BJT Gummel-Poon model, interactive waveform viewer rendered via WebGL with pan/zoom and automatic measurements (rise time, overshoot, settling time, bandwidth, phase margin), 1,000+ built-in device models from major vendors (TI, Analog Devices, Infineon) stored in S3 with PostgreSQL metadata, SPICE netlist import/export compatible with HSPICE/NGSPICE format, Stripe billing with three tiers (Free / Pro $39/mo / Team $79/user/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| SPICE Solver | Rust (native + WASM) | Custom MNA engine with KLU sparse solver via `suitesparse-sys` |
| WASM Compilation | `wasm-pack` + `wasm-bindgen` | Client-side solver for circuits ≤500 nodes |
| Model Extraction | Python 3.12 (FastAPI) | SPICE model parameter extraction from datasheets, vendor model parsing |
| Database | PostgreSQL 16 | Projects, users, simulations, device model metadata |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | SPICE model files, simulation results (waveform data), schematic snapshots |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| Schematic Editor | Custom SVG renderer | R-tree spatial indexing, snap-to-grid, auto-wire routing |
| Waveform Viewer | WebGL 2.0 (custom) | GPU-accelerated rendering for 10M+ data points |
| Real-time | WebSocket (Axum) | Live simulation progress, convergence monitoring |
| Job Queue | Redis 7 + Tokio tasks | Server-side simulation job management, Monte Carlo distribution |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | WASM bundle delivery, static frontend assets |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, error tracking |
| CI/CD | GitHub Actions | WASM build pipeline, Rust tests, Docker image push |

### Key Architecture Decisions

1. **Hybrid client/server solver with WASM threshold at 500 nodes**: Circuits with ≤500 nodes (covers 90%+ of educational and quick-check simulations) run entirely in the browser via WASM, providing instant feedback with zero server cost. Circuits exceeding 500 nodes are submitted to the Rust-native server solver which handles multi-thousand-node circuits and Monte Carlo sweeps. The threshold is configurable per plan tier.

2. **Custom SPICE engine in Rust rather than wrapping NGSPICE**: Building a custom MNA solver in Rust gives us full control over convergence algorithms, WASM compilation, and parallelization for Monte Carlo. NGSPICE's C codebase has extensive global state that makes WASM compilation fragile and thread-unsafe. Our Rust solver uses `suitesparse-sys` bindings for KLU sparse factorization (the same algorithm used by commercial SPICE) while maintaining memory safety and WASM compatibility.

3. **SVG schematic editor with R-tree spatial indexing**: SVG gives crisp rendering at any zoom level and straightforward DOM-based hit testing. An R-tree (via `rbush` JS library) enables O(log n) spatial queries for wire routing, snap detection, and component selection even with 1,000+ components. Canvas/WebGL alternatives were rejected because they require reimplementing text layout, cursor interaction, and accessibility.

4. **WebGL waveform viewer separate from schematic**: The waveform viewer renders millions of data points (transient simulations can produce 10M+ samples) using GPU-accelerated line rendering with level-of-detail decimation. This is decoupled from the SVG schematic to allow independent pan/zoom and multi-pane layouts. Data is streamed from WASM/server as Float64Arrays and uploaded to GPU buffers.

5. **S3 for model storage with PostgreSQL metadata catalog**: Device models (SPICE `.lib` files, typically 1-50KB each) are stored in S3, while PostgreSQL holds searchable metadata (manufacturer, part number, package, parameters, category). This allows the model library to scale to 100K+ models without bloating the database while enabling fast parametric search via PostgreSQL indexes and full-text search.

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";  -- For fuzzy text search on models

-- Users
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email TEXT UNIQUE NOT NULL,
    password_hash TEXT,  -- NULL for OAuth-only users
    name TEXT NOT NULL,
    avatar_url TEXT,
    auth_provider TEXT DEFAULT 'email',  -- email | google | github
    auth_provider_id TEXT,
    plan TEXT NOT NULL DEFAULT 'free',  -- free | pro | team
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX users_email_idx ON users(email);
CREATE INDEX users_stripe_idx ON users(stripe_customer_id);

-- Organizations (for Team plan)
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

-- Projects (schematic + simulation workspace)
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    schematic_data JSONB NOT NULL DEFAULT '{}',  -- Full schematic state (components, wires, labels)
    settings JSONB DEFAULT '{}',  -- Grid size, snap settings, default units
    is_public BOOLEAN NOT NULL DEFAULT false,
    forked_from UUID REFERENCES projects(id) ON DELETE SET NULL,
    thumbnail_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);
CREATE INDEX projects_org_idx ON projects(org_id);
CREATE INDEX projects_updated_idx ON projects(updated_at DESC);

-- Simulation Runs
CREATE TABLE simulations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    analysis_type TEXT NOT NULL,  -- dc_op | dc_sweep | ac | transient
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | running | completed | failed | cancelled
    execution_mode TEXT NOT NULL DEFAULT 'wasm',  -- wasm | server
    netlist TEXT NOT NULL,  -- Generated SPICE netlist
    parameters JSONB NOT NULL DEFAULT '{}',  -- Analysis-specific parameters
    node_count INTEGER NOT NULL DEFAULT 0,
    results_url TEXT,  -- S3 URL for result data
    results_summary JSONB,  -- Quick-access summary (measurements, key values)
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

-- Server Simulation Jobs (for server-side execution only)
CREATE TABLE simulation_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    simulation_id UUID NOT NULL REFERENCES simulations(id) ON DELETE CASCADE,
    worker_id TEXT,
    cores_allocated INTEGER DEFAULT 1,
    memory_mb INTEGER DEFAULT 2048,
    priority INTEGER DEFAULT 0,  -- Higher = more urgent
    progress_pct REAL DEFAULT 0.0,
    convergence_data JSONB DEFAULT '[]',  -- [{iteration, residual, timestep}]
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX jobs_sim_idx ON simulation_jobs(simulation_id);
CREATE INDEX jobs_status_idx ON simulation_jobs(worker_id);

-- Device Models (metadata; actual model files in S3)
CREATE TABLE device_models (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,  -- e.g., "LM358" or "NMOS_BSIM4"
    manufacturer TEXT,  -- e.g., "Texas Instruments"
    part_number TEXT,  -- Manufacturer part number
    category TEXT NOT NULL,  -- resistor | capacitor | inductor | diode | bjt | mosfet | opamp | voltage_source | current_source | subcircuit
    subcategory TEXT,  -- e.g., "n-channel" for MOSFET, "PNP" for BJT
    model_type TEXT NOT NULL,  -- spice_primitive | subcircuit | behavioral
    spice_model_url TEXT NOT NULL,  -- S3 URL to .lib/.mod file
    parameters JSONB DEFAULT '{}',  -- Key electrical parameters for search
    symbol_data JSONB,  -- SVG symbol definition
    datasheet_url TEXT,
    description TEXT,
    tags TEXT[] DEFAULT '{}',
    is_verified BOOLEAN DEFAULT false,
    is_builtin BOOLEAN DEFAULT true,
    uploaded_by UUID REFERENCES users(id),
    download_count INTEGER DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX models_category_idx ON device_models(category);
CREATE INDEX models_manufacturer_idx ON device_models(manufacturer);
CREATE INDEX models_name_trgm_idx ON device_models USING gin(name gin_trgm_ops);
CREATE INDEX models_part_trgm_idx ON device_models USING gin(part_number gin_trgm_ops);
CREATE INDEX models_tags_idx ON device_models USING gin(tags);

-- Saved Waveform Configurations
CREATE TABLE waveform_configs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    simulation_id UUID NOT NULL REFERENCES simulations(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    panes JSONB NOT NULL,  -- [{signals: [{node, color, scale}], y_range, measurements}]
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX waveform_sim_idx ON waveform_configs(simulation_id);

-- Usage Tracking (for billing enforcement)
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,  -- simulation_minutes | cloud_simulation | storage_bytes
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
    pub schematic_data: serde_json::Value,
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
    pub analysis_type: String,
    pub status: String,
    pub execution_mode: String,
    pub netlist: String,
    pub parameters: serde_json::Value,
    pub node_count: i32,
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
    pub cores_allocated: i32,
    pub memory_mb: i32,
    pub priority: i32,
    pub progress_pct: f32,
    pub convergence_data: serde_json::Value,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct DeviceModel {
    pub id: Uuid,
    pub name: String,
    pub manufacturer: Option<String>,
    pub part_number: Option<String>,
    pub category: String,
    pub subcategory: Option<String>,
    pub model_type: String,
    pub spice_model_url: String,
    pub parameters: serde_json::Value,
    pub symbol_data: Option<serde_json::Value>,
    pub datasheet_url: Option<String>,
    pub description: Option<String>,
    pub tags: Vec<String>,
    pub is_verified: bool,
    pub is_builtin: bool,
    pub uploaded_by: Option<Uuid>,
    pub download_count: i32,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct SimulationParams {
    pub analysis_type: AnalysisType,
    pub dc_sweep: Option<DcSweepParams>,
    pub ac: Option<AcParams>,
    pub transient: Option<TransientParams>,
    pub temperature: f64,  // Default 27°C
    pub options: SpiceOptions,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum AnalysisType {
    DcOp,
    DcSweep,
    Ac,
    Transient,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct DcSweepParams {
    pub source_name: String,
    pub start: f64,
    pub stop: f64,
    pub step: f64,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct AcParams {
    pub variation: String,  // "dec" | "oct" | "lin"
    pub points: u32,
    pub f_start: f64,
    pub f_stop: f64,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct TransientParams {
    pub step: f64,
    pub stop: f64,
    pub start: Option<f64>,
    pub max_step: Option<f64>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct SpiceOptions {
    pub abstol: f64,    // Default 1e-12
    pub reltol: f64,    // Default 1e-3
    pub vntol: f64,     // Default 1e-6
    pub gmin: f64,      // Default 1e-12
    pub itl1: u32,      // DC iteration limit, default 100
    pub itl4: u32,      // Transient iteration limit per timestep, default 10
}
```

---

## Solver Architecture Deep-Dive

### Governing Equations and Discretization

SpiceSim's core solver implements **Modified Nodal Analysis (MNA)**, the standard formulation used by all production SPICE simulators. For a circuit with `n` nodes and `m` independent voltage sources (plus inductors in DC), the MNA system is:

```
┌         ┐ ┌   ┐   ┌   ┐
│  G   B  │ │ v │   │ i │
│         │ │   │ = │   │
│  C   D  │ │ j │   │ e │
└         ┘ └   ┘   └   ┘
```

Where:
- **G** (n×n): Conductance matrix — sum of conductances connected to each node
- **B** (n×m): Voltage source connection matrix
- **C** (m×n): Transpose of B (for ideal voltage sources)
- **D** (m×m): Zero for ideal sources, non-zero for dependent sources
- **v** (n): Unknown node voltages
- **j** (m): Unknown branch currents through voltage sources
- **i** (n): Known current source contributions
- **e** (m): Known voltage source values

**Nonlinear elements** (diodes, MOSFETs, BJTs) are linearized using Newton-Raphson iteration. At each iteration `k`, the nonlinear device current I(V) is replaced by its companion model:

```
I(V) ≈ I(V_k) + G_d · (V - V_k)

where G_d = dI/dV |_{V=V_k}  (device conductance Jacobian)
```

The linearized companion model stamps into the MNA matrix as a conductance `G_d` and a current source `I(V_k) - G_d·V_k`. Newton-Raphson iterates until `||V_{k+1} - V_k|| < VNTOL` and `||I_{k+1} - I_k|| < ABSTOL`.

**Transient analysis** uses the Trapezoidal Rule (default) or Backward Euler for time integration:

```
Trapezoidal:  i_C(t_{n+1}) = (2C/h)·v(t_{n+1}) - (2C/h)·v(t_n) - i_C(t_n)
              i_L(t_{n+1}) = (h/2L)·v(t_{n+1}) + (h/2L)·v(t_n) + i_L(t_n)

where h = t_{n+1} - t_n (timestep)
```

Capacitors become conductances `G_eq = 2C/h` with companion current sources; inductors become conductances `G_eq = h/(2L)` with companion current sources. These stamp directly into the MNA matrix.

**Adaptive timestep** uses Local Truncation Error (LTE) estimation by comparing the trapezoidal and backward-Euler predictions:

```
LTE ≈ (h²/12) · (d²v/dt²)

h_new = h · (RELTOL / |LTE|)^(1/2)    (clamped to [h_min, h_max])
```

### Client/Server Split (WASM Threshold)

```
Circuit uploaded → Node count extracted
    │
    ├── ≤500 nodes → WASM solver (browser)
    │   ├── Instant startup, no network latency
    │   ├── Results displayed immediately
    │   └── No server cost
    │
    └── >500 nodes → Server solver (Rust native)
        ├── Job queued via Redis
        ├── Worker picks up, runs native solver
        ├── Progress streamed via WebSocket
        └── Results stored in S3, URL returned
```

The 500-node threshold was chosen because:
- WASM solver with KLU handles 500-node transient (10K timepoints) in <2 seconds on modern hardware
- 500 nodes covers: op-amp circuits (50-200 nodes), basic power converters (100-300 nodes), filter chains (50-150 nodes)
- Above 500 nodes: IC-level simulations, Monte Carlo sweeps, large mixed-signal designs → need server compute

### WASM Compilation Pipeline

```toml
# solver-wasm/Cargo.toml
[package]
name = "spicesim-solver-wasm"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
wasm-bindgen = "0.2"
serde = { version = "1", features = ["derive"] }
serde-wasm-bindgen = "0.6"
js-sys = "0.3"
suitesparse-src = { version = "0.3", features = ["klu"] }  # KLU sparse solver
nalgebra-sparse = "0.9"
log = "0.4"
console_log = "1"

[profile.release]
opt-level = "z"       # Optimize for size
lto = true            # Link-time optimization
codegen-units = 1     # Single codegen unit for better optimization
strip = true          # Strip debug symbols
wasm-opt = true       # Run wasm-opt
```

```yaml
# .github/workflows/wasm-build.yml
name: Build WASM Solver
on:
  push:
    paths: ['solver-wasm/**', 'solver-core/**']
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with:
          targets: wasm32-unknown-unknown
      - uses: cargo-bins/cargo-binstall@main
      - run: cargo binstall -y wasm-pack wasm-opt
      - run: cd solver-wasm && wasm-pack build --target web --release
      - run: wasm-opt -Oz solver-wasm/pkg/spicesim_solver_wasm_bg.wasm -o solver-wasm/pkg/spicesim_solver_wasm_bg.wasm
      - name: Upload to S3/CDN
        run: aws s3 cp solver-wasm/pkg/ s3://spicesim-wasm/v${{ github.sha }}/ --recursive
```

---

## Architecture Deep-Dives

### 1. Simulation API Handler (Rust/Axum)

The primary endpoint receives a simulation request, validates the circuit, decides between WASM and server execution, and for server execution enqueues a job with Redis and streams progress via WebSocket.

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
    solver::netlist::generate_netlist,
    state::AppState,
    auth::Claims,
    error::ApiError,
};

#[derive(serde::Deserialize)]
pub struct CreateSimulationRequest {
    pub analysis_type: AnalysisType,
    pub parameters: serde_json::Value,
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

    // 2. Generate netlist from schematic
    let netlist = generate_netlist(&project.schematic_data)?;
    let node_count = netlist.node_count();

    // 3. Check plan limits
    let user = sqlx::query_as!(
        crate::db::models::User,
        "SELECT * FROM users WHERE id = $1",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    if user.plan == "free" && node_count > 200 {
        return Err(ApiError::PlanLimit(
            "Free plan supports circuits up to 200 nodes. Upgrade to Pro for unlimited."
        ));
    }

    // 4. Determine execution mode
    let execution_mode = if node_count <= 500 {
        "wasm"
    } else {
        "server"
    };

    // 5. Create simulation record
    let sim = sqlx::query_as!(
        Simulation,
        r#"INSERT INTO simulations
            (project_id, user_id, analysis_type, status, execution_mode,
             netlist, parameters, node_count)
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8)
        RETURNING *"#,
        project_id,
        claims.user_id,
        serde_json::to_string(&req.analysis_type)?,
        if execution_mode == "wasm" { "completed" } else { "pending" },
        execution_mode,
        netlist.to_spice_string(),
        req.parameters,
        node_count as i32,
    )
    .fetch_one(&state.db)
    .await?;

    // 6. For server execution, enqueue job
    if execution_mode == "server" {
        let job = sqlx::query_as!(
            crate::db::models::SimulationJob,
            r#"INSERT INTO simulation_jobs (simulation_id, priority)
            VALUES ($1, $2) RETURNING *"#,
            sim.id,
            if user.plan == "team" { 10 } else { 0 },
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
```

### 2. SPICE Solver Core (Rust — shared between WASM and native)

The core MNA solver that builds and solves the circuit equations. This code compiles to both native (server) and WASM (browser) targets.

```rust
// solver-core/src/mna.rs

use nalgebra_sparse::{CsrMatrix, CooMatrix};
use crate::devices::{Device, Stamp};
use crate::klu::KluSolver;

pub struct MnaSystem {
    pub size: usize,              // n_nodes + n_voltage_sources
    pub n_nodes: usize,
    pub n_vsources: usize,
    pub matrix: CooMatrix<f64>,   // COO for assembly, converted to CSR for solve
    pub rhs: Vec<f64>,            // Right-hand side vector
    pub solution: Vec<f64>,       // Node voltages + branch currents
    pub prev_solution: Vec<f64>,  // Previous Newton iteration
}

impl MnaSystem {
    pub fn new(n_nodes: usize, n_vsources: usize) -> Self {
        let size = n_nodes + n_vsources;
        Self {
            size,
            n_nodes,
            n_vsources,
            matrix: CooMatrix::new(size, size),
            rhs: vec![0.0; size],
            solution: vec![0.0; size],
            prev_solution: vec![0.0; size],
        }
    }

    /// Clear matrix and RHS for re-stamping
    pub fn clear(&mut self) {
        self.matrix = CooMatrix::new(self.size, self.size);
        self.rhs.fill(0.0);
    }

    /// Stamp a conductance between nodes n1 and n2
    pub fn stamp_conductance(&mut self, n1: usize, n2: usize, g: f64) {
        if n1 > 0 { self.matrix.push(n1 - 1, n1 - 1, g); }
        if n2 > 0 { self.matrix.push(n2 - 1, n2 - 1, g); }
        if n1 > 0 && n2 > 0 {
            self.matrix.push(n1 - 1, n2 - 1, -g);
            self.matrix.push(n2 - 1, n1 - 1, -g);
        }
    }

    /// Stamp a current source from n1 to n2 (current flows n1 → n2)
    pub fn stamp_current_source(&mut self, n1: usize, n2: usize, i: f64) {
        if n1 > 0 { self.rhs[n1 - 1] -= i; }
        if n2 > 0 { self.rhs[n2 - 1] += i; }
    }

    /// Stamp a voltage source between n_pos and n_neg, branch index b
    pub fn stamp_voltage_source(
        &mut self, n_pos: usize, n_neg: usize, b: usize, voltage: f64
    ) {
        let row = self.n_nodes + b;
        if n_pos > 0 {
            self.matrix.push(row, n_pos - 1, 1.0);
            self.matrix.push(n_pos - 1, row, 1.0);
        }
        if n_neg > 0 {
            self.matrix.push(row, n_neg - 1, -1.0);
            self.matrix.push(n_neg - 1, row, -1.0);
        }
        self.rhs[row] = voltage;
    }

    /// Solve the system using KLU sparse direct solver
    pub fn solve(&mut self) -> Result<(), SolverError> {
        let csr = CsrMatrix::from(&self.matrix);
        let mut solver = KluSolver::new(&csr)?;
        self.solution = solver.solve(&self.rhs)?;
        Ok(())
    }

    /// Check Newton-Raphson convergence
    pub fn check_convergence(&self, vntol: f64, abstol: f64) -> bool {
        for i in 0..self.n_nodes {
            let dv = (self.solution[i] - self.prev_solution[i]).abs();
            if dv > vntol + abstol * self.solution[i].abs() {
                return false;
            }
        }
        true
    }
}

/// DC Operating Point solver
pub fn solve_dc_op(
    devices: &mut [Box<dyn Device>],
    mna: &mut MnaSystem,
    options: &SpiceOptions,
) -> Result<Vec<f64>, SolverError> {
    // Newton-Raphson iteration
    for iter in 0..options.itl1 {
        mna.clear();

        // Stamp all devices (linear + nonlinear companion models)
        for device in devices.iter_mut() {
            device.stamp(mna, &mna.solution.clone());
        }

        // Add Gmin to diagonal for convergence aid
        for i in 0..mna.n_nodes {
            mna.matrix.push(i, i, options.gmin);
        }

        mna.prev_solution = mna.solution.clone();
        mna.solve()?;

        if mna.check_convergence(options.vntol, options.abstol) {
            return Ok(mna.solution.clone());
        }
    }

    // If standard Newton-Raphson didn't converge, try Gmin stepping
    solve_with_gmin_stepping(devices, mna, options)
}

/// Transient analysis solver
pub fn solve_transient(
    devices: &mut [Box<dyn Device>],
    mna: &mut MnaSystem,
    params: &TransientParams,
    options: &SpiceOptions,
) -> Result<TransientResult, SolverError> {
    let mut result = TransientResult::new(mna.n_nodes);
    let mut t = params.start.unwrap_or(0.0);
    let mut h = params.step;

    // Initial DC operating point
    let v0 = solve_dc_op(devices, mna, options)?;
    result.push_timepoint(t, &v0);

    while t < params.stop {
        let h_actual = h.min(params.stop - t);

        // Update companion models for reactive elements (C, L)
        for device in devices.iter_mut() {
            device.update_transient_companion(h_actual, t);
        }

        // Newton-Raphson at this timepoint
        let mut converged = false;
        for _iter in 0..options.itl4 {
            mna.clear();
            for device in devices.iter_mut() {
                device.stamp(mna, &mna.solution.clone());
            }
            for i in 0..mna.n_nodes {
                mna.matrix.push(i, i, options.gmin);
            }
            mna.prev_solution = mna.solution.clone();
            mna.solve()?;

            if mna.check_convergence(options.vntol, options.abstol) {
                converged = true;
                break;
            }
        }

        if !converged {
            // Halve timestep and retry
            h *= 0.5;
            if h < 1e-18 {
                return Err(SolverError::Convergence(
                    format!("Failed to converge at t={:.3e}s", t)
                ));
            }
            mna.solution = mna.prev_solution.clone();
            continue;
        }

        t += h_actual;
        result.push_timepoint(t, &mna.solution);

        // Adaptive timestep via LTE estimation
        h = estimate_next_timestep(devices, h_actual, options.reltol);
        h = h.min(params.max_step.unwrap_or(params.step * 10.0));
        h = h.max(params.step / 100.0);
    }

    Ok(result)
}

#[derive(Debug)]
pub enum SolverError {
    Convergence(String),
    Singular(String),
    InvalidCircuit(String),
}
```

### 3. Waveform Viewer Component (React + WebGL)

The frontend waveform viewer that renders millions of data points using WebGL with level-of-detail decimation and interactive measurement tools.

```typescript
// frontend/src/components/WaveformViewer.tsx

import { useRef, useEffect, useCallback, useState } from 'react';
import { useWaveformStore } from '../stores/waveformStore';

interface WaveformSignal {
  nodeId: string;
  label: string;
  color: string;
  data: Float64Array;  // Interleaved [t0, v0, t1, v1, ...]
  visible: boolean;
}

interface WaveformPane {
  id: string;
  signals: WaveformSignal[];
  yMin: number;
  yMax: number;
  measurements: Measurement[];
}

export function WaveformViewer() {
  const canvasRef = useRef<HTMLCanvasElement>(null);
  const glRef = useRef<WebGL2RenderingContext | null>(null);
  const programRef = useRef<WebGLProgram | null>(null);
  const { panes, timeRange, setTimeRange, addMeasurement } = useWaveformStore();
  const [hoveredPoint, setHoveredPoint] = useState<{ t: number; v: number } | null>(null);

  // Initialize WebGL
  useEffect(() => {
    const canvas = canvasRef.current;
    if (!canvas) return;

    const gl = canvas.getContext('webgl2', { antialias: true, alpha: false });
    if (!gl) return;
    glRef.current = gl;

    // Compile shaders for line rendering
    const vsSource = `#version 300 es
      in vec2 a_position;  // (time, voltage)
      uniform vec2 u_xRange; // (tMin, tMax)
      uniform vec2 u_yRange; // (vMin, vMax)
      uniform vec2 u_viewport; // (width, height)

      void main() {
        float x = 2.0 * (a_position.x - u_xRange.x) / (u_xRange.y - u_xRange.x) - 1.0;
        float y = 2.0 * (a_position.y - u_yRange.x) / (u_yRange.y - u_yRange.x) - 1.0;
        gl_Position = vec4(x, y, 0.0, 1.0);
      }
    `;
    const fsSource = `#version 300 es
      precision mediump float;
      uniform vec4 u_color;
      out vec4 fragColor;
      void main() {
        fragColor = u_color;
      }
    `;

    const program = createShaderProgram(gl, vsSource, fsSource);
    programRef.current = program;

    return () => {
      gl.deleteProgram(program);
    };
  }, []);

  // Render waveforms
  const render = useCallback(() => {
    const gl = glRef.current;
    const program = programRef.current;
    if (!gl || !program) return;

    gl.viewport(0, 0, gl.canvas.width, gl.canvas.height);
    gl.clearColor(0.12, 0.12, 0.15, 1.0);
    gl.clear(gl.COLOR_BUFFER_BIT);
    gl.useProgram(program);

    const xRangeLoc = gl.getUniformLocation(program, 'u_xRange');
    const yRangeLoc = gl.getUniformLocation(program, 'u_yRange');
    const colorLoc = gl.getUniformLocation(program, 'u_color');

    for (const pane of panes) {
      gl.uniform2f(yRangeLoc, pane.yMin, pane.yMax);
      gl.uniform2f(xRangeLoc, timeRange.min, timeRange.max);

      for (const signal of pane.signals) {
        if (!signal.visible) continue;

        // Level-of-detail: decimate data for current zoom level
        const decimated = decimateForViewport(
          signal.data,
          timeRange.min,
          timeRange.max,
          gl.canvas.width
        );

        // Upload to GPU and draw
        const buffer = gl.createBuffer();
        gl.bindBuffer(gl.ARRAY_BUFFER, buffer);
        gl.bufferData(gl.ARRAY_BUFFER, decimated, gl.STREAM_DRAW);

        const posLoc = gl.getAttribLocation(program, 'a_position');
        gl.enableVertexAttribArray(posLoc);
        gl.vertexAttribPointer(posLoc, 2, gl.FLOAT, false, 0, 0);

        const [r, g, b] = hexToRgb(signal.color);
        gl.uniform4f(colorLoc, r, g, b, 1.0);
        gl.drawArrays(gl.LINE_STRIP, 0, decimated.length / 2);

        gl.deleteBuffer(buffer);
      }
    }
  }, [panes, timeRange]);

  useEffect(() => {
    const animId = requestAnimationFrame(render);
    return () => cancelAnimationFrame(animId);
  }, [render]);

  // Handle mouse wheel for zoom
  const handleWheel = useCallback((e: React.WheelEvent) => {
    e.preventDefault();
    const canvas = canvasRef.current;
    if (!canvas) return;

    const rect = canvas.getBoundingClientRect();
    const xFrac = (e.clientX - rect.left) / rect.width;
    const center = timeRange.min + xFrac * (timeRange.max - timeRange.min);

    const zoomFactor = e.deltaY > 0 ? 1.2 : 0.8;
    const newSpan = (timeRange.max - timeRange.min) * zoomFactor;
    setTimeRange({
      min: center - xFrac * newSpan,
      max: center + (1 - xFrac) * newSpan,
    });
  }, [timeRange, setTimeRange]);

  return (
    <div className="waveform-viewer flex flex-col h-full bg-gray-900">
      <WaveformToolbar onAddMeasurement={addMeasurement} />
      <div className="flex-1 relative">
        <canvas
          ref={canvasRef}
          className="w-full h-full"
          onWheel={handleWheel}
        />
        {hoveredPoint && (
          <div className="absolute pointer-events-none bg-gray-800 text-white text-xs px-2 py-1 rounded">
            t={formatEngineering(hoveredPoint.t)}s, v={formatEngineering(hoveredPoint.v)}V
          </div>
        )}
        <TimeAxis range={timeRange} />
      </div>
      <MeasurementsPanel panes={panes} />
    </div>
  );
}

// Decimate waveform data to ~2x pixel count for current viewport
function decimateForViewport(
  data: Float64Array,
  tMin: number,
  tMax: number,
  viewportWidth: number
): Float32Array {
  const targetPoints = viewportWidth * 2;  // 2 points per pixel (min/max)
  const totalPoints = data.length / 2;

  if (totalPoints <= targetPoints) {
    return new Float32Array(data);  // No decimation needed
  }

  // Find visible range in data
  let startIdx = 0;
  let endIdx = totalPoints - 1;
  for (let i = 0; i < totalPoints; i++) {
    if (data[i * 2] >= tMin) { startIdx = Math.max(0, i - 1); break; }
  }
  for (let i = totalPoints - 1; i >= 0; i--) {
    if (data[i * 2] <= tMax) { endIdx = Math.min(totalPoints - 1, i + 1); break; }
  }

  const visiblePoints = endIdx - startIdx + 1;
  if (visiblePoints <= targetPoints) {
    const result = new Float32Array(visiblePoints * 2);
    for (let i = 0; i < visiblePoints; i++) {
      result[i * 2] = data[(startIdx + i) * 2];
      result[i * 2 + 1] = data[(startIdx + i) * 2 + 1];
    }
    return result;
  }

  // Min-max decimation: for each pixel bucket, keep min and max values
  const bucketSize = visiblePoints / (targetPoints / 2);
  const result = new Float32Array(targetPoints * 2);
  let outIdx = 0;

  for (let b = 0; b < targetPoints / 2 && outIdx < targetPoints * 2 - 3; b++) {
    const bStart = startIdx + Math.floor(b * bucketSize);
    const bEnd = Math.min(startIdx + Math.floor((b + 1) * bucketSize), endIdx);

    let minV = Infinity, maxV = -Infinity;
    let minT = 0, maxT = 0;
    for (let i = bStart; i <= bEnd; i++) {
      const v = data[i * 2 + 1];
      if (v < minV) { minV = v; minT = data[i * 2]; }
      if (v > maxV) { maxV = v; maxT = data[i * 2]; }
    }

    // Output min then max (or max then min to maintain time order)
    if (minT <= maxT) {
      result[outIdx++] = minT; result[outIdx++] = minV;
      result[outIdx++] = maxT; result[outIdx++] = maxV;
    } else {
      result[outIdx++] = maxT; result[outIdx++] = maxV;
      result[outIdx++] = minT; result[outIdx++] = minV;
    }
  }

  return result.slice(0, outIdx);
}

function formatEngineering(value: number): string {
  const prefixes = [
    { exp: -15, prefix: 'f' }, { exp: -12, prefix: 'p' },
    { exp: -9, prefix: 'n' }, { exp: -6, prefix: 'µ' },
    { exp: -3, prefix: 'm' }, { exp: 0, prefix: '' },
    { exp: 3, prefix: 'k' }, { exp: 6, prefix: 'M' },
    { exp: 9, prefix: 'G' },
  ];
  if (value === 0) return '0';
  const absVal = Math.abs(value);
  for (let i = prefixes.length - 1; i >= 0; i--) {
    if (absVal >= Math.pow(10, prefixes[i].exp)) {
      return (value / Math.pow(10, prefixes[i].exp)).toFixed(3) + prefixes[i].prefix;
    }
  }
  return value.toExponential(3);
}
```

### 4. Server-Side Simulation Worker (Rust + Redis)

Background worker that picks up server-side simulation jobs from Redis, runs the native solver, streams progress via WebSocket, and stores results in S3.

```rust
// src/workers/simulation_worker.rs

use std::sync::Arc;
use aws_sdk_s3::Client as S3Client;
use redis::AsyncCommands;
use sqlx::PgPool;
use tokio::sync::broadcast;
use uuid::Uuid;

use crate::solver::{
    mna::{solve_dc_op, solve_transient},
    netlist::parse_netlist,
    devices::create_devices,
};
use crate::db::models::{SimulationJob, SpiceOptions};
use crate::websocket::ProgressBroadcaster;

pub struct SimulationWorker {
    db: PgPool,
    redis: redis::Client,
    s3: S3Client,
    broadcaster: Arc<ProgressBroadcaster>,
}

impl SimulationWorker {
    pub async fn run(&self) -> anyhow::Result<()> {
        let mut conn = self.redis.get_async_connection().await?;
        tracing::info!("Simulation worker started, listening for jobs...");

        loop {
            // Block-pop from Redis job queue (priority-sorted set)
            let job_id: Option<String> = conn.brpop("simulation:jobs", 30.0).await?;
            let Some(job_id) = job_id else { continue };
            let job_id: Uuid = job_id.parse()?;

            if let Err(e) = self.process_job(job_id).await {
                tracing::error!("Job {job_id} failed: {e}");
                self.mark_failed(job_id, &e.to_string()).await?;
            }
        }
    }

    async fn process_job(&self, job_id: Uuid) -> anyhow::Result<()> {
        // 1. Fetch job and simulation details
        let job = sqlx::query_as!(
            SimulationJob,
            "UPDATE simulation_jobs SET started_at = NOW(), worker_id = $2
             WHERE id = $1 RETURNING *",
            job_id, hostname::get()?.to_string_lossy().to_string()
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

        // 2. Parse netlist and create device instances
        let circuit = parse_netlist(&sim.netlist)?;
        let mut devices = create_devices(&circuit)?;
        let options: SpiceOptions = serde_json::from_value(
            sim.parameters.get("options").cloned()
                .unwrap_or(serde_json::json!({}))
        )?;

        // 3. Run simulation based on analysis type
        let result_data = match sim.analysis_type.as_str() {
            "dc_op" => {
                let mut mna = crate::solver::mna::MnaSystem::new(
                    circuit.n_nodes, circuit.n_vsources
                );
                let voltages = solve_dc_op(&mut devices, &mut mna, &options)?;
                serde_json::to_vec(&voltages)?
            }
            "transient" => {
                let params: crate::db::models::TransientParams =
                    serde_json::from_value(sim.parameters.clone())?;
                let mut mna = crate::solver::mna::MnaSystem::new(
                    circuit.n_nodes, circuit.n_vsources
                );

                // Run with progress callback
                let progress_tx = self.broadcaster.get_channel(sim.id);
                let result = solve_transient(
                    &mut devices, &mut mna, &params, &options,
                )?;

                // Stream progress updates
                let total_time = params.stop - params.start.unwrap_or(0.0);
                for (i, tp) in result.timepoints.iter().enumerate() {
                    if i % 100 == 0 {
                        let pct = (tp.time / total_time * 100.0) as f32;
                        let _ = progress_tx.send(serde_json::json!({
                            "progress_pct": pct,
                            "current_time": tp.time,
                            "iteration": i,
                        }));

                        sqlx::query!(
                            "UPDATE simulation_jobs SET progress_pct = $1 WHERE id = $2",
                            pct, job_id
                        )
                        .execute(&self.db)
                        .await?;
                    }
                }

                result.to_binary()?
            }
            "dc_sweep" => {
                let params: crate::db::models::DcSweepParams =
                    serde_json::from_value(sim.parameters.clone())?;
                let mut mna = crate::solver::mna::MnaSystem::new(
                    circuit.n_nodes, circuit.n_vsources
                );
                let results = crate::solver::analysis::dc_sweep(
                    &mut devices, &mut mna, &params, &options
                )?;
                serde_json::to_vec(&results)?
            }
            "ac" => {
                let params: crate::db::models::AcParams =
                    serde_json::from_value(sim.parameters.clone())?;
                let mut mna = crate::solver::mna::MnaSystem::new(
                    circuit.n_nodes, circuit.n_vsources
                );
                let results = crate::solver::analysis::ac_analysis(
                    &mut devices, &mut mna, &params, &options
                )?;
                serde_json::to_vec(&results)?
            }
            _ => anyhow::bail!("Unknown analysis type: {}", sim.analysis_type),
        };

        // 4. Upload results to S3
        let s3_key = format!("results/{}/{}.bin", sim.project_id, sim.id);
        self.s3.put_object()
            .bucket("spicesim-results")
            .key(&s3_key)
            .body(result_data.into())
            .content_type("application/octet-stream")
            .send()
            .await?;

        let results_url = format!("s3://spicesim-results/{}", s3_key);

        // 5. Update database with completed status
        sqlx::query!(
            "UPDATE simulations SET status = 'completed', results_url = $2,
             completed_at = NOW(), wall_time_ms = EXTRACT(EPOCH FROM (NOW() - started_at))::int * 1000
             WHERE id = $1",
            sim.id, results_url
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

        // 6. Notify client via WebSocket
        self.broadcaster.send(sim.id, serde_json::json!({
            "status": "completed",
            "results_url": results_url,
        }))?;

        tracing::info!("Job {job_id} completed successfully");
        Ok(())
    }

    async fn mark_failed(&self, job_id: Uuid, error: &str) -> anyhow::Result<()> {
        sqlx::query!(
            "UPDATE simulation_jobs SET completed_at = NOW() WHERE id = $1",
            job_id
        )
        .execute(&self.db)
        .await?;

        let sim_id: Uuid = sqlx::query_scalar!(
            "SELECT simulation_id FROM simulation_jobs WHERE id = $1",
            job_id
        )
        .fetch_one(&self.db)
        .await?;

        sqlx::query!(
            "UPDATE simulations SET status = 'failed', error_message = $2,
             completed_at = NOW() WHERE id = $1",
            sim_id, error
        )
        .execute(&self.db)
        .await?;

        Ok(())
    }
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–4)

**Day 1: Project initialization and Rust backend scaffold**
```bash
cargo init spicesim-api
cd spicesim-api
cargo add axum tokio serde serde_json sqlx uuid chrono tower-http jsonwebtoken bcrypt
cargo add --features postgres,runtime-tokio-rustls,uuid,chrono,json sqlx
cargo add aws-sdk-s3 redis
```
- `src/main.rs` — Axum app with CORS, tracing, graceful shutdown
- `src/config.rs` — Environment-based configuration (DATABASE_URL, JWT_SECRET, S3_BUCKET, REDIS_URL)
- `src/state.rs` — AppState struct (PgPool, Redis, S3Client)
- `src/error.rs` — ApiError enum with IntoResponse
- `Dockerfile` — Multi-stage build (builder + runtime)
- `docker-compose.yml` — PostgreSQL, Redis, MinIO (S3-compatible)

**Day 2: Database schema and migrations**
- `migrations/001_initial.sql` — All 8 tables: users, organizations, org_members, projects, simulations, simulation_jobs, device_models, waveform_configs, usage_records
- `src/db/mod.rs` — Database pool initialization
- `src/db/models.rs` — All SQLx structs with FromRow derives
- Run `sqlx migrate run` to apply schema
- Seed script for initial device models (basic R, L, C, diode, MOSFET symbols)

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

### Phase 2 — Solver Core — Prototype (Days 5–12)

**Day 5: MNA matrix framework**
- `solver-core/` — New Rust workspace member (shared between native and WASM)
- `solver-core/src/mna.rs` — MnaSystem struct with stamp methods (conductance, current source, voltage source)
- `solver-core/src/klu.rs` — KLU sparse solver wrapper via `suitesparse-sys` bindings
- Unit tests: simple resistive divider, single VCVS

**Day 6: Linear device models**
- `solver-core/src/devices/resistor.rs` — Resistor stamp (G = 1/R between nodes)
- `solver-core/src/devices/capacitor.rs` — Capacitor companion model (trapezoidal)
- `solver-core/src/devices/inductor.rs` — Inductor companion model (trapezoidal)
- `solver-core/src/devices/vsource.rs` — Independent voltage source stamp
- `solver-core/src/devices/isource.rs` — Independent current source stamp
- `solver-core/src/devices/mod.rs` — Device trait definition
- Tests: RC circuit DC, RL circuit DC, RLC resonance

**Day 7: Nonlinear device models — Diode and BJT**
- `solver-core/src/devices/diode.rs` — Shockley diode model (Is, N, Rs, Cj) with Newton-Raphson companion
- `solver-core/src/devices/bjt.rs` — Gummel-Poon BJT model (NPN/PNP) with Ebers-Moll and Early effect
- Limiting algorithm to prevent Newton-Raphson divergence (voltage clamping per iteration)
- Tests: half-wave rectifier, common-emitter amplifier DC bias

**Day 8: MOSFET models — BSIM4 and EKV**
- `solver-core/src/devices/mosfet.rs` — MOSFET device with model dispatch
- `solver-core/src/devices/mosfet_bsim4.rs` — BSIM4 Level 54 model (threshold voltage, mobility degradation, DIBL, channel-length modulation, subthreshold)
- `solver-core/src/devices/mosfet_ekv.rs` — EKV 2.6 model (simpler, good for low-power)
- Parameter extraction from SPICE `.model` statements
- Tests: NMOS I-V curves (Id vs Vds family), CMOS inverter transfer curve

**Day 9: DC analysis — Operating point and sweep**
- `solver-core/src/analysis/dc_op.rs` — DC operating point solver with Newton-Raphson, Gmin stepping, source stepping fallback
- `solver-core/src/analysis/dc_sweep.rs` — DC sweep over source values with warm-start from previous solution
- Convergence monitoring and diagnostics (which node diverged, iteration count)
- Tests: op-amp bias point, resistive ladder, Zener diode regulator

**Day 10: AC small-signal analysis**
- `solver-core/src/analysis/ac.rs` — AC analysis: linearize at DC operating point, build complex MNA matrix (G + jωC), solve at each frequency
- Frequency sweep: decade, octave, or linear spacing
- Output: magnitude and phase for node voltages and branch currents
- Tests: RC low-pass filter (compare to analytical -3dB point), LC resonance

**Day 11: Transient analysis**
- `solver-core/src/analysis/transient.rs` — Full transient solver with trapezoidal integration, adaptive timestep (LTE-based), convergence control
- Source waveforms: pulse, sine, PWL (piecewise linear), exponential
- `solver-core/src/sources.rs` — Time-dependent source evaluation
- Tests: RC charging curve (compare to analytical), full-wave rectifier, CMOS ring oscillator

**Day 12: Netlist parser and generator**
- `solver-core/src/netlist/parser.rs` — SPICE netlist parser (element lines, .model, .param, .subckt, .include, analysis commands)
- `solver-core/src/netlist/generator.rs` — Generate netlist from schematic JSON
- `solver-core/src/netlist/models.rs` — Circuit, Element, Model, Analysis AST types
- Support standard SPICE syntax compatible with HSPICE/NGSPICE
- Tests: parse and simulate standard benchmark netlists (Nagel's test circuits)

### Phase 3 — WASM Build + Frontend Visualization (Days 13–18)

**Day 13: WASM solver compilation**
- `solver-wasm/` — New workspace member for WASM target
- `solver-wasm/Cargo.toml` — wasm-bindgen, serde-wasm-bindgen dependencies
- `solver-wasm/src/lib.rs` — WASM entry points: `solve_dc_op()`, `solve_transient()`, `solve_ac()`, `solve_dc_sweep()`
- `wasm-pack build --target web --release` pipeline
- `wasm-opt -Oz` for size optimization (target <2MB gzipped)
- JavaScript wrapper: `SpiceSolver` class that loads WASM and provides async API

**Day 14: Frontend scaffold and schematic editor foundation**
```bash
npm create vite@latest frontend -- --template react-ts
cd frontend
npm i zustand @tanstack/react-query axios rbush
```
- `src/App.tsx` — Router, auth context, layout
- `src/stores/projectStore.ts` — Zustand store for project state
- `src/stores/schematicStore.ts` — Schematic state (components, wires, selection)
- `src/components/SchematicEditor/Canvas.tsx` — SVG canvas with pan/zoom (matrix transform)
- `src/components/SchematicEditor/Grid.tsx` — Snap-to-grid overlay (configurable 10px/20px/50px)
- `src/lib/spatial.ts` — R-tree setup for spatial queries

**Day 15: Schematic editor — component placement and wiring**
- `src/components/SchematicEditor/ComponentPalette.tsx` — Categorized component library sidebar (Passives, Semiconductors, Sources, ICs)
- `src/components/SchematicEditor/ComponentSymbol.tsx` — SVG component renderer (R, L, C, diode, MOSFET, BJT, op-amp, V/I source)
- `src/components/SchematicEditor/Wire.tsx` — Wire drawing tool (click-to-place, auto-Manhattan routing)
- `src/components/SchematicEditor/NetLabel.tsx` — Net label placement (GND, VCC, named nets)
- Drag-and-drop from palette → canvas, rotation (R key), flip (F key), delete (Del key)
- Property editor panel: component value, model selection, name

**Day 16: Schematic editor — advanced features**
- `src/components/SchematicEditor/AutoWire.tsx` — Automatic wire routing avoiding component overlap
- `src/components/SchematicEditor/SelectionBox.tsx` — Multi-select with box drag, copy/paste, move group
- `src/components/SchematicEditor/ProbeMarker.tsx` — Voltage/current probe placement for simulation output
- Cross-probing highlight system (hover node → highlight wires and connected pins)
- Undo/redo stack (Zustand middleware)
- Keyboard shortcuts: Ctrl+Z/Y undo/redo, Ctrl+C/V copy/paste, Ctrl+S save, R rotate, G ground

**Day 17: Waveform viewer — WebGL renderer**
- `src/components/WaveformViewer/WaveformViewer.tsx` — WebGL2 canvas for waveform rendering
- `src/components/WaveformViewer/WaveformPane.tsx` — Individual pane with Y-axis scaling
- `src/components/WaveformViewer/TimeAxis.tsx` — Shared X-axis with engineering notation
- GPU line rendering with level-of-detail decimation (min-max per pixel bucket)
- Pan: click-drag on canvas. Zoom: mouse wheel (centered on cursor position)
- Signal color coding with legend

**Day 18: Waveform viewer — measurements and interaction**
- `src/components/WaveformViewer/Cursors.tsx` — Vertical cursor lines (drag to position) with delta readout
- `src/components/WaveformViewer/Measurements.tsx` — Auto-measurements: rise time (10-90%), overshoot, settling time, frequency, duty cycle, RMS, peak-to-peak
- `src/components/WaveformViewer/FFTPane.tsx` — FFT display for frequency content of transient signals
- `src/stores/waveformStore.ts` — Zustand store for pane layout, signals, cursor positions
- Multi-pane layout: add/remove/resize panes vertically
- Signal add/remove: right-click node in schematic → "Add to waveform viewer"

### Phase 4 — API + Job Orchestration (Days 19–24)

**Day 19: Simulation API endpoints**
- `src/api/handlers/simulation.rs` — Create simulation, get simulation, list simulations, cancel simulation
- Netlist generation from schematic JSON → SPICE netlist string
- Node count extraction and WASM/server routing logic
- Plan-based limits enforcement (free: 200 nodes, 10 cloud minutes/month)

**Day 20: Server-side simulation worker**
- `src/workers/simulation_worker.rs` — Redis job consumer, runs native solver
- `src/workers/mod.rs` — Worker pool management (configurable concurrency)
- Progress streaming: worker publishes progress → Redis pub/sub → WebSocket to client
- Error handling: convergence failures, timeouts, out-of-memory
- S3 result upload with presigned URL generation for client download

**Day 21: WebSocket for live simulation progress**
- `src/api/ws/mod.rs` — Axum WebSocket upgrade handler
- `src/api/ws/simulation_progress.rs` — Subscribe to simulation progress channel
- Client receives: `{ progress_pct, current_time, iteration, residual }`
- Connection management: heartbeat ping/pong, automatic cleanup on disconnect
- Frontend: `src/hooks/useSimulationProgress.ts` — React hook for WebSocket subscription

**Day 22: Device model library API**
- `src/api/handlers/models.rs` — Search models (parametric + full-text), get model, download SPICE file
- S3 integration for model file storage and retrieval
- Model categories: resistor, capacitor, inductor, diode, bjt_npn, bjt_pnp, nmos, pmos, opamp, subcircuit
- Parametric search: filter by manufacturer, category, package, voltage rating, current capacity
- Pagination with cursor-based scrolling for large result sets

**Day 23: Netlist import/export**
- `src/api/handlers/netlist.rs` — Import SPICE netlist file, export project as SPICE netlist
- `solver-core/src/netlist/import.rs` — Parse uploaded netlist, create schematic layout (auto-place components)
- `solver-core/src/netlist/export.rs` — Generate HSPICE/NGSPICE compatible netlist from schematic
- Support .subckt definitions, .model statements, .param expressions
- Validation: check for floating nodes, missing ground, invalid connections

**Day 24: Project management features**
- `src/api/handlers/projects.rs` — Fork project (deep copy), share via public link, project templates
- `src/api/handlers/export.rs` — Export schematic as SVG/PNG, export waveform data as CSV
- Project thumbnail generation: server-side SVG-to-PNG for dashboard cards
- Auto-save: frontend debounced PATCH every 5 seconds on schematic changes

### Phase 5 — Results + Post-Processing UI (Days 25–29)

**Day 25: Simulation result processing**
- Result parsing: binary format (Float64Array) → structured data (time, node voltages, branch currents)
- Result caching: hot results in Redis (TTL 1 hour), cold results in S3
- Presigned S3 URLs for client-side download of large result sets
- Result summary generation: auto-detect key measurements (gain, bandwidth, settling time)

**Day 26: AC analysis visualization**
- `src/components/WaveformViewer/BodePlot.tsx` — Bode plot: magnitude (dB) and phase (degrees) vs. frequency (log scale)
- `src/components/WaveformViewer/NyquistPlot.tsx` — Nyquist plot for stability analysis
- Automatic measurements: -3dB bandwidth, gain margin, phase margin, unity-gain frequency
- Dual Y-axis: magnitude on left, phase on right, shared log-frequency X-axis

**Day 27: Transient analysis post-processing**
- `src/components/WaveformViewer/TransientMeasurements.tsx` — Automated measurements panel
- Rise time (10%→90%), fall time, propagation delay, overshoot, undershoot
- Settling time (to ±1%, ±2%, ±5% of final value)
- Frequency/period detection from zero crossings
- THD (total harmonic distortion) via FFT
- PSRR measurement for power supply circuits

**Day 28: Schematic cross-probing and annotation**
- Click waveform cursor → highlight corresponding node in schematic
- Click schematic node → add trace to waveform viewer
- DC operating point annotation: show node voltages and branch currents directly on schematic
- Color-coded nets based on voltage level (red = high, blue = low)
- Simulation status indicator on schematic toolbar

**Day 29: Schematic templates and examples**
- `src/data/templates/` — Pre-built schematic templates:
  - Inverting op-amp amplifier
  - Non-inverting op-amp amplifier
  - Buck converter (basic)
  - Common-source MOSFET amplifier
  - RC low-pass filter
  - Full-bridge rectifier
  - CMOS inverter
  - Differential pair
- Template gallery page with preview thumbnails and one-click "Use Template"
- Example simulations pre-run to show expected waveforms

### Phase 6 — Billing + Plan Enforcement (Days 30–33)

**Day 30: Stripe integration**
- `src/api/handlers/billing.rs` — Create checkout session, create customer portal session, get subscription status
- `src/api/handlers/webhooks/stripe.rs` — Handle subscription events: `checkout.session.completed`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.payment_failed`
- Plan mapping:
  - Free: 200-node circuits, 10 cloud simulation minutes/month, basic models
  - Pro ($39/mo): Unlimited nodes, 500 cloud minutes/month, full model library, waveform export
  - Team ($79/user/mo): Collaboration, 2000 cloud minutes/month, API access, custom model upload

**Day 31: Usage tracking and plan limits**
- `src/middleware/plan_limits.rs` — Middleware checking plan limits before simulation execution
- `src/services/usage.rs` — Track cloud simulation minutes per billing period
- Usage record insertion after each server-side simulation completes
- Usage dashboard: `src/api/handlers/usage.rs` — Current period usage, historical usage
- Approaching-limit warnings at 80% and 100% of cloud minutes

**Day 32: Billing UI**
- `frontend/src/pages/Billing.tsx` — Plan comparison table, current plan display, usage meter
- `frontend/src/components/billing/PlanCard.tsx` — Plan feature comparison cards
- `frontend/src/components/billing/UsageMeter.tsx` — Cloud minutes usage bar with threshold indicators
- Upgrade/downgrade flow via Stripe Customer Portal
- Upgrade prompt modals when hitting plan limits (e.g., "Circuit has 350 nodes. Upgrade to Pro for unlimited.")

**Day 33: Feature gating**
- Gate waveform export (CSV/PNG) behind Pro plan
- Gate custom model upload behind Team plan
- Gate real-time collaboration features behind Team plan
- Gate API access behind Team plan
- Show locked feature indicators with upgrade CTAs in UI
- Admin endpoint: override plan for specific users (beta testers, partners)

### Phase 7 — Solver Validation + Testing (Days 34–38)

**Day 34: Solver validation — resistive and reactive circuits**
- Benchmark 1: Resistive voltage divider — verify exact analytical solution
- Benchmark 2: RC charging curve — compare transient to V(t) = V₀(1 - e^(-t/RC))
- Benchmark 3: RLC series resonance — AC analysis, verify resonant frequency f₀ = 1/(2π√(LC))
- Benchmark 4: Coupled inductors (transformer) — verify turns ratio
- Automated test suite: `solver-core/tests/benchmarks.rs` with assertions within 0.1% tolerance

**Day 35: Solver validation — semiconductor circuits**
- Benchmark 5: Diode half-wave rectifier — compare peak output to analytical (V_peak - V_forward)
- Benchmark 6: Common-emitter BJT amplifier — verify voltage gain within 5% of hand calculation
- Benchmark 7: CMOS inverter transfer curve — verify VIL, VIH, VOL, VOH, noise margins
- Benchmark 8: Two-stage op-amp (Miller-compensated) — verify gain >60dB, GBW, phase margin
- Compare results against NGSPICE reference simulations for same netlists

**Day 36: Solver validation — convergence stress tests**
- Test Gmin stepping with difficult circuits (positive feedback, high-gain loops)
- Test source stepping with circuits that have no obvious initial operating point
- Test adaptive timestep with stiff circuits (fast switching + slow settling)
- Test large circuits: 100-node, 500-node, 1000-node generated ladder networks
- Performance benchmarks: time vs. node count, time vs. timepoints

**Day 37: Integration testing**
- End-to-end test: create project → draw schematic → run simulation → view waveform → export results
- API integration tests: auth → project CRUD → simulation → results
- WASM solver test: load in headless browser (Playwright), run simulation, verify results match native solver
- WebSocket test: connect → subscribe → receive progress → receive completion
- Concurrent simulation test: submit 10 simultaneous server-side jobs

**Day 38: Performance testing and optimization**
- WASM solver benchmarks: measure time for 100-node, 300-node, 500-node circuits in Chrome/Firefox/Safari
- Target: 500-node transient (10K timepoints) < 2 seconds in WASM
- Server solver benchmarks: measure throughput (simulations/minute) with various node counts
- Frontend rendering: measure waveform viewer FPS with 1M, 5M, 10M data points
- Target: 60 FPS pan/zoom with 5M data points
- Memory profiling: ensure WASM solver doesn't leak, frontend GC behavior is healthy
- Load testing: 50 concurrent users running simulations via k6

### Phase 8 — Deployment + Polish + Launch (Days 39–42)

**Day 39: Docker and Kubernetes configuration**
- `Dockerfile` — Multi-stage Rust build (cargo-chef for layer caching)
- `docker-compose.yml` — Full local dev stack (API, PostgreSQL, Redis, MinIO, frontend)
- `k8s/` — Kubernetes manifests:
  - `api-deployment.yaml` — API server (3 replicas, HPA)
  - `worker-deployment.yaml` — Simulation workers (auto-scaling based on queue depth)
  - `postgres-statefulset.yaml` — PostgreSQL with PVC
  - `redis-deployment.yaml` — Redis for job queue and pub/sub
  - `ingress.yaml` — NGINX ingress with TLS
- Health check endpoints: `/health/live`, `/health/ready`

**Day 40: CDN and WASM delivery**
- CloudFront distribution for:
  - Frontend static assets (HTML, JS, CSS) — cached at edge
  - WASM solver bundle (~2MB) — cached at edge with long TTL and versioned URLs
  - Device model files from S3 — cached with 1-hour TTL
- WASM preloading: start downloading WASM bundle on page load before user needs it
- Service worker for offline WASM caching (progressive enhancement)

**Day 41: Monitoring, logging, and polish**
- Prometheus metrics: simulation duration histogram, solver convergence rate, API latency percentiles
- Grafana dashboards: system health, simulation throughput, user activity
- Sentry integration: frontend and backend error tracking with source maps
- Structured logging (tracing + JSON) for all API requests and solver events
- UI polish: loading states, error messages, empty states, responsive layout
- Accessibility: keyboard navigation in schematic editor, ARIA labels on interactive elements

**Day 42: Launch preparation and final testing**
- End-to-end smoke test in production environment
- Database backup and restore verification
- Rate limiting: 100 req/min per user, 10 simulations/min per user
- Security audit: JWT validation, SQL injection (SQLx prevents this), CORS configuration, CSP headers
- Landing page with product overview, pricing, and signup
- Documentation: getting started guide, keyboard shortcuts reference, supported SPICE syntax
- Deploy to production, enable monitoring alerts, announce launch

---

## Critical Files

```
spicesim/
├── solver-core/                           # Shared solver library (native + WASM)
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── mna.rs                         # MNA matrix assembly and solve
│   │   ├── klu.rs                         # KLU sparse solver wrapper
│   │   ├── sources.rs                     # Time-dependent source waveforms
│   │   ├── devices/
│   │   │   ├── mod.rs                     # Device trait definition
│   │   │   ├── resistor.rs
│   │   │   ├── capacitor.rs
│   │   │   ├── inductor.rs
│   │   │   ├── vsource.rs
│   │   │   ├── isource.rs
│   │   │   ├── diode.rs
│   │   │   ├── bjt.rs                     # Gummel-Poon BJT
│   │   │   ├── mosfet.rs                  # MOSFET dispatch
│   │   │   ├── mosfet_bsim4.rs            # BSIM4 Level 54
│   │   │   ├── mosfet_ekv.rs              # EKV 2.6
│   │   │   └── opamp.rs                   # Ideal op-amp (for quick sims)
│   │   ├── analysis/
│   │   │   ├── mod.rs
│   │   │   ├── dc_op.rs                   # DC operating point
│   │   │   ├── dc_sweep.rs                # DC sweep analysis
│   │   │   ├── ac.rs                      # AC small-signal analysis
│   │   │   └── transient.rs               # Transient analysis
│   │   └── netlist/
│   │       ├── mod.rs
│   │       ├── parser.rs                  # SPICE netlist parser
│   │       ├── generator.rs               # Netlist from schematic JSON
│   │       ├── export.rs                  # Export to HSPICE/NGSPICE format
│   │       └── models.rs                  # Circuit AST types
│   └── tests/
│       ├── benchmarks.rs                  # Solver validation benchmarks
│       ├── convergence.rs                 # Convergence stress tests
│       └── netlist_parsing.rs             # Netlist parser tests
│
├── solver-wasm/                           # WASM compilation target
│   ├── Cargo.toml
│   └── src/
│       └── lib.rs                         # WASM entry points (wasm-bindgen)
│
├── spicesim-api/                          # Rust API server
│   ├── Cargo.toml
│   ├── src/
│   │   ├── main.rs                        # Axum app setup, router, startup
│   │   ├── config.rs                      # Environment config
│   │   ├── state.rs                       # AppState (PgPool, Redis, S3)
│   │   ├── error.rs                       # ApiError types
│   │   ├── auth/
│   │   │   ├── mod.rs                     # JWT middleware, Claims
│   │   │   └── oauth.rs                   # Google/GitHub OAuth handlers
│   │   ├── api/
│   │   │   ├── router.rs                  # Route definitions
│   │   │   ├── handlers/
│   │   │   │   ├── auth.rs                # Register, login, OAuth
│   │   │   │   ├── users.rs               # Profile CRUD
│   │   │   │   ├── projects.rs            # Project CRUD, fork, share
│   │   │   │   ├── simulation.rs          # Create/get/cancel simulation
│   │   │   │   ├── models.rs              # Device model search/get
│   │   │   │   ├── netlist.rs             # Import/export netlist
│   │   │   │   ├── billing.rs             # Stripe checkout/portal
│   │   │   │   ├── usage.rs               # Usage tracking
│   │   │   │   ├── export.rs              # SVG/PNG/CSV export
│   │   │   │   └── webhooks/
│   │   │   │       └── stripe.rs          # Stripe webhook handler
│   │   │   └── ws/
│   │   │       ├── mod.rs                 # WebSocket upgrade handler
│   │   │       └── simulation_progress.rs # Live progress streaming
│   │   ├── middleware/
│   │   │   └── plan_limits.rs             # Plan enforcement middleware
│   │   ├── services/
│   │   │   ├── usage.rs                   # Usage tracking service
│   │   │   └── s3.rs                      # S3 client helpers
│   │   └── workers/
│   │       ├── mod.rs                     # Worker pool management
│   │       └── simulation_worker.rs       # Server-side simulation execution
│   ├── migrations/
│   │   └── 001_initial.sql                # Full database schema
│   └── tests/
│       ├── api_integration.rs             # API integration tests
│       └── simulation_e2e.rs              # End-to-end simulation tests
│
├── frontend/                              # React frontend
│   ├── package.json
│   ├── vite.config.ts
│   ├── src/
│   │   ├── App.tsx                        # Router, providers, layout
│   │   ├── main.tsx                       # Entry point
│   │   ├── stores/
│   │   │   ├── authStore.ts               # Auth state (JWT, user)
│   │   │   ├── projectStore.ts            # Project state
│   │   │   ├── schematicStore.ts          # Schematic editor state
│   │   │   └── waveformStore.ts           # Waveform viewer state
│   │   ├── hooks/
│   │   │   ├── useSimulationProgress.ts   # WebSocket hook for live progress
│   │   │   └── useWasmSolver.ts           # WASM solver hook (load + run)
│   │   ├── lib/
│   │   │   ├── api.ts                     # Axios API client
│   │   │   ├── spatial.ts                 # R-tree for spatial queries
│   │   │   └── wasmLoader.ts              # WASM bundle loading
│   │   ├── pages/
│   │   │   ├── Dashboard.tsx              # Project list
│   │   │   ├── Editor.tsx                 # Main editor (schematic + waveform split)
│   │   │   ├── Models.tsx                 # Device model library browser
│   │   │   ├── Templates.tsx              # Schematic template gallery
│   │   │   ├── Billing.tsx                # Plan management
│   │   │   ├── Login.tsx                  # Auth pages
│   │   │   └── Register.tsx
│   │   ├── components/
│   │   │   ├── SchematicEditor/
│   │   │   │   ├── Canvas.tsx             # SVG canvas with pan/zoom
│   │   │   │   ├── Grid.tsx               # Snap grid overlay
│   │   │   │   ├── ComponentPalette.tsx   # Component library sidebar
│   │   │   │   ├── ComponentSymbol.tsx    # SVG component renderer
│   │   │   │   ├── Wire.tsx               # Wire drawing tool
│   │   │   │   ├── NetLabel.tsx           # Net labels (GND, VCC, named)
│   │   │   │   ├── ProbeMarker.tsx        # Voltage/current probes
│   │   │   │   ├── PropertyPanel.tsx      # Component property editor
│   │   │   │   ├── SelectionBox.tsx       # Multi-select
│   │   │   │   └── AutoWire.tsx           # Auto-wire routing
│   │   │   ├── WaveformViewer/
│   │   │   │   ├── WaveformViewer.tsx     # Main WebGL viewer
│   │   │   │   ├── WaveformPane.tsx       # Individual pane
│   │   │   │   ├── TimeAxis.tsx           # Shared time axis
│   │   │   │   ├── Cursors.tsx            # Measurement cursors
│   │   │   │   ├── Measurements.tsx       # Auto-measurement panel
│   │   │   │   ├── BodePlot.tsx           # AC analysis Bode plot
│   │   │   │   ├── NyquistPlot.tsx        # Nyquist stability plot
│   │   │   │   └── FFTPane.tsx            # FFT display
│   │   │   ├── billing/
│   │   │   │   ├── PlanCard.tsx
│   │   │   │   └── UsageMeter.tsx
│   │   │   └── common/
│   │   │       ├── SimulationToolbar.tsx   # Run/stop/analysis selector
│   │   │       └── ModelSearch.tsx         # Device model search component
│   │   └── data/
│   │       └── templates/                 # Pre-built schematic templates (JSON)
│   └── public/
│       └── wasm/                          # WASM solver bundle (loaded at runtime)
│
├── model-service/                         # Python FastAPI (model extraction)
│   ├── requirements.txt
│   ├── main.py                            # FastAPI app
│   ├── extractors/
│   │   ├── spice_parser.py                # Parse SPICE .lib/.mod files
│   │   └── datasheet_extractor.py         # Extract model params from datasheet curves
│   └── Dockerfile
│
├── k8s/                                   # Kubernetes manifests
│   ├── api-deployment.yaml
│   ├── worker-deployment.yaml
│   ├── postgres-statefulset.yaml
│   ├── redis-deployment.yaml
│   ├── model-service-deployment.yaml
│   └── ingress.yaml
│
├── docker-compose.yml                     # Local development stack
├── Cargo.toml                             # Workspace root
└── .github/
    └── workflows/
        ├── ci.yml                         # Test + lint on PR
        ├── wasm-build.yml                 # Build + deploy WASM bundle
        └── deploy.yml                     # Build Docker images + deploy to K8s
```

---

## Solver Validation Suite

### Benchmark 1: Resistive Voltage Divider (DC Operating Point)

**Circuit:** V1=10V → R1=1kΩ → node A → R2=2kΩ → GND

**Expected:** V(A) = 10 × 2k/(1k+2k) = **6.6667 V** (exact)

**Tolerance:** < 0.001% (machine precision after single solve)

### Benchmark 2: RC Charging Curve (Transient)

**Circuit:** V1=5V step (t=0) → R=10kΩ → node A → C=1µF → GND

**Expected at t = RC = 10ms:** V(A) = 5 × (1 - 1/e) = **3.1606 V**

**Expected at t = 5RC = 50ms:** V(A) = 5 × (1 - e⁻⁵) = **4.9663 V**

**Tolerance:** < 0.5% (trapezoidal integration error depends on timestep)

### Benchmark 3: RLC Series Resonance (AC Analysis)

**Circuit:** V_AC → R=50Ω → L=1mH → C=10nF → GND

**Expected resonant frequency:** f₀ = 1/(2π√(LC)) = 1/(2π√(1e-3 × 10e-9)) = **50.33 kHz**

**Expected Q factor:** Q = (1/R)√(L/C) = (1/50)√(1e-3/10e-9) = **6.32**

**Expected -3dB bandwidth:** BW = f₀/Q = **7.96 kHz**

**Tolerance:** Resonant frequency < 0.1%, Q factor < 1%, bandwidth < 2%

### Benchmark 4: CMOS Inverter Transfer Curve (DC Sweep)

**Circuit:** NMOS (W/L=2µ/0.18µ, BSIM4) and PMOS (W/L=4µ/0.18µ, BSIM4) inverter, VDD=1.8V

**Expected:** V_M (switching threshold) ≈ **0.90 V** (VDD/2 for balanced sizing)

**Expected:** Gain at V_M > **-20 V/V** (high gain in transition region)

**Expected:** V_OL < **50 mV**, V_OH > **1.75 V**

**Tolerance:** V_M < 5%, Gain < 10%, VOL/VOH < 5%

### Benchmark 5: Two-Stage Miller-Compensated Op-Amp (AC Analysis)

**Circuit:** Standard two-stage CMOS op-amp with Miller compensation capacitor, 0.18µm BSIM4 models

**Expected:** DC gain > **60 dB** (1000 V/V)

**Expected:** Unity-gain bandwidth (GBW) ≈ **50 MHz** (depends on bias current and Cc)

**Expected:** Phase margin > **60°** (stable with Miller compensation)

**Tolerance:** DC gain < 5%, GBW < 10%, Phase margin < 5°

---

## Verification

### Integration Testing Checklist

1. **Auth flow** — Register → login → JWT issued → authorized API calls → token refresh → logout
2. **Project CRUD** — Create → update schematic → save → reload → verify schematic state preserved
3. **Schematic editing** — Place component → wire → set values → generate netlist → verify netlist syntax
4. **WASM simulation** — Small circuit (10 nodes) → WASM solver → results returned → waveform displays
5. **Server simulation** — Large circuit (600 nodes) → job queued → worker picks up → WebSocket progress → results in S3
6. **Waveform viewer** — Load 1M-point result → pan/zoom → cursor measurements → verify values
7. **Model library** — Search "LM358" → results returned → select model → use in schematic
8. **Netlist import** — Upload HSPICE netlist → schematic auto-generated → simulate → results match reference
9. **Netlist export** — Design circuit → export SPICE netlist → verify syntax compatible with NGSPICE
10. **Plan limits** — Free user → circuit with 300 nodes → blocked with upgrade prompt
11. **Billing** — Subscribe to Pro → Stripe checkout → webhook → plan updated → limits lifted
12. **Concurrent users** — 10 users simultaneously running server simulations → all complete correctly
13. **Cross-probing** — Click waveform node → schematic highlights → click schematic probe → waveform adds trace
14. **Template usage** — Select "CMOS Inverter" template → project created → simulate → expected waveform
15. **Error handling** — Non-convergent circuit → meaningful error message with suggestions → no crash

### SQL Verification Queries

```sql
-- 1. Simulation throughput and success rate
SELECT
    execution_mode,
    status,
    COUNT(*) as count,
    AVG(wall_time_ms)::int as avg_time_ms,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY wall_time_ms)::int as p95_time_ms
FROM simulations
WHERE created_at >= NOW() - INTERVAL '7 days'
GROUP BY execution_mode, status;

-- 2. Analysis type distribution
SELECT analysis_type, COUNT(*) as count,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 1) as pct
FROM simulations
WHERE created_at >= NOW() - INTERVAL '30 days'
GROUP BY analysis_type ORDER BY count DESC;

-- 3. User plan distribution and conversion
SELECT plan, COUNT(*) as users,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 1) as pct
FROM users GROUP BY plan ORDER BY users DESC;

-- 4. Cloud simulation usage by user (billing period)
SELECT u.email, u.plan,
    SUM(ur.quantity) as total_minutes,
    CASE u.plan
        WHEN 'free' THEN 10
        WHEN 'pro' THEN 500
        WHEN 'team' THEN 2000
    END as limit_minutes
FROM usage_records ur
JOIN users u ON ur.user_id = u.id
WHERE ur.period_start <= CURRENT_DATE AND ur.period_end >= CURRENT_DATE
    AND ur.record_type = 'simulation_minutes'
GROUP BY u.email, u.plan
ORDER BY total_minutes DESC;

-- 5. Device model popularity
SELECT dm.name, dm.manufacturer, dm.category,
    dm.download_count,
    COUNT(DISTINCT s.project_id) as projects_using
FROM device_models dm
LEFT JOIN simulations s ON s.netlist ILIKE '%' || dm.name || '%'
GROUP BY dm.id
ORDER BY dm.download_count DESC
LIMIT 20;
```

### Performance Benchmarks

| Operation | Target | Method |
|-----------|--------|--------|
| WASM solver: 100-node DC op | <100ms | Browser benchmark (Chrome DevTools) |
| WASM solver: 500-node transient (10K pts) | <2s | Browser benchmark |
| Server solver: 1000-node transient (100K pts) | <30s | Server timing, 4 cores |
| Server solver: 5000-node DC op | <10s | Server timing, 8 cores |
| Waveform viewer: 1M points render | 60 FPS | Chrome FPS counter during pan/zoom |
| Waveform viewer: 10M points render | 30 FPS | Chrome FPS counter during pan/zoom |
| Schematic editor: 500 components | 60 FPS | SVG rendering performance |
| API: create simulation | <200ms | p95 latency (Prometheus) |
| API: search device models | <100ms | p95 latency with 50K models in DB |
| WASM bundle load (cached) | <500ms | Service worker cache hit |
| WASM bundle load (cold) | <3s | CDN delivery, 2MB gzipped |
| WebSocket: progress latency | <100ms | Time from worker emit to client receive |
| Netlist parsing: 1000-line file | <50ms | Server-side Rust parser |

---

## Deployment Architecture

### Infrastructure

```
┌─────────────────────────────────────────────────┐
│                  CloudFront CDN                   │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────┐ │
│  │ Frontend SPA │  │ WASM Bundle │  │ Model    │ │
│  │ (HTML/JS/CSS)│  │ (versioned) │  │ Files    │ │
│  └─────────────┘  └─────────────┘  └──────────┘ │
└─────────────────────────┬───────────────────────┘
                          │
              ┌───────────┴───────────┐
              │   AWS ALB (HTTPS)     │
              │   TLS termination     │
              └───────────┬───────────┘
                          │
        ┌─────────────────┼─────────────────┐
        ▼                 ▼                 ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ API Server   │  │ API Server   │  │ API Server   │
│ (Rust/Axum)  │  │ (Rust/Axum)  │  │ (Rust/Axum)  │
│ Pod ×3 (HPA) │  │              │  │              │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘
       │                 │                 │
       └─────────────────┼─────────────────┘
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ PostgreSQL   │ │ Redis        │ │ S3 Bucket    │
│ (RDS r6g)    │ │ (ElastiCache)│ │ (Results +   │
│ Multi-AZ     │ │ Cluster mode │ │  Models)     │
└──────────────┘ └──────┬───────┘ └──────────────┘
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ Sim Worker   │ │ Sim Worker   │ │ Sim Worker   │
│ (Rust native)│ │ (Rust native)│ │ (Rust native)│
│ c7g.2xlarge  │ │ c7g.2xlarge  │ │ c7g.2xlarge  │
│ HPA: 2-20    │ │              │ │              │
└──────────────┘ └──────────────┘ └──────────────┘
```

### Scaling Strategy

- **API servers**: Horizontal pod autoscaler (HPA) at 70% CPU, min 3, max 10
- **Simulation workers**: HPA based on Redis queue depth — scale up when queue > 5 jobs, scale down when idle 5 min
- **Worker instances**: AWS c7g.2xlarge (8 vCPU, 16GB RAM) for compute-optimized simulation
- **Database**: RDS PostgreSQL r6g.xlarge with read replicas for model search queries
- **Redis**: ElastiCache r6g.large cluster with 1 primary + 2 replicas

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
        { "Days": 90, "StorageClass": "GLACIER" }
      ],
      "Expiration": { "Days": 365 }
    },
    {
      "ID": "device-models-keep",
      "Prefix": "models/",
      "Status": "Enabled",
      "Transitions": [
        { "Days": 365, "StorageClass": "STANDARD_IA" }
      ]
    },
    {
      "ID": "wasm-bundles-keep",
      "Prefix": "wasm/",
      "Status": "Enabled",
      "NoncurrentVersionExpiration": { "NoncurrentDays": 30 }
    }
  ]
}
```

---

## Post-MVP Roadmap

| Feature | Description | Priority |
|---------|-------------|----------|
| Cloud-Distributed Monte Carlo | Distribute 10,000+ Monte Carlo points across worker fleet, aggregate statistical results (mean, sigma, yield) in real-time with live histogram updates | High |
| Parametric Optimization | Gradient-free optimizer (Nelder-Mead, genetic algorithm) that tunes component values to meet circuit specifications (bandwidth, gain, phase margin) automatically | High |
| Mixed-Signal Co-Simulation | Verilog-A behavioral model support and digital block co-simulation with automatic A/D and D/A interface insertion for mixed-signal verification | High |
| Real-Time Collaboration | CRDT-based schematic editing with cursor presence, commenting on nodes/waveforms, and shared simulation results for team design reviews | Medium |
| S-Parameter and RF Analysis | Touchstone file import, S-parameter simulation, Smith chart display, and transmission line models for RF/high-speed digital design | Medium |
| PDK Support | Academic PDK integration (TSMC, GlobalFoundries process models) with NDA-protected model hosting for university and industry users | Medium |
| Eye Diagram and Jitter Analysis | Eye diagram generation for high-speed serial interfaces with jitter decomposition (Rj, Dj) and BER estimation from transient simulation | Low |
| Model Parameter Extraction | Extract SPICE model parameters from measured I-V/C-V curves or datasheet plots using optimization-based fitting (Levenberg-Marquardt) | Low |
