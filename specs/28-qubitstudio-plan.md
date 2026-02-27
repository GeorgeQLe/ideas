# 28. QubitStudio — Cloud Quantum Computing IDE and Simulator

## Implementation Plan

**MVP Scope:** Browser-based quantum circuit builder with drag-and-drop gate placement (H, X, Y, Z, S, T, CNOT, CZ, SWAP, Rx, Ry, Rz, Toffoli, measure, barrier) on interactive qubit wire canvas rendered via Canvas2D/WebGL, custom state-vector simulation engine implementing unitary matrix evolution compiled to WebAssembly for client-side execution of circuits ≤18 qubits and server-side Rust-native GPU-accelerated execution for circuits up to 30 qubits, support for state-vector simulation, shot-based measurement sampling, and parameterized gate sweeps, real-time quantum state visualization rendered via WebGL (Bloch spheres, amplitude bar charts with phase coloring, measurement histograms), OpenQASM 3.0 code editor with bidirectional circuit-to-code synchronization via Monaco Editor, circuit templates for common algorithms (Bell state, GHZ, QFT, Grover's search), project management with version-controlled circuits and share links, Python FastAPI microservice for Qiskit-based transpilation and OpenQASM parsing, Stripe billing with three tiers (Free / Pro $99/mo / Team $299/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| Quantum Solver | Rust (native + WASM) | Custom state-vector engine with optimized gate application via bit manipulation |
| WASM Compilation | `wasm-pack` + `wasm-bindgen` | Client-side simulator for circuits ≤18 qubits (512KB state vector at f64) |
| Transpilation Service | Python 3.12 (FastAPI) | Qiskit-based transpilation, OpenQASM 3.0 parsing, noise model calibration |
| Database | PostgreSQL 16 | Projects, users, circuits, experiments, noise models |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | Simulation results (state vectors, measurement counts), circuit snapshots, exports |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| Circuit Canvas | Custom Canvas2D renderer | Gate placement grid, qubit wire layout, interactive drag-and-drop |
| State Visualization | WebGL 2.0 (custom) + Three.js | Bloch spheres (Three.js), amplitude bar charts (WebGL), phase wheels |
| Code Editor | Monaco Editor | Custom OpenQASM 3.0 language support, syntax highlighting, autocomplete |
| Real-time | WebSocket (Axum) | Live simulation progress, GPU job monitoring |
| Job Queue | Redis 7 + Tokio tasks | Server-side simulation job management, parameter sweep distribution |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | WASM bundle delivery, static frontend assets |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, error tracking |
| CI/CD | GitHub Actions | WASM build pipeline, Rust tests, Docker image push |

### Key Architecture Decisions

1. **Hybrid client/server simulator with WASM threshold at 18 qubits**: Circuits with ≤18 qubits run entirely in the browser via WASM. An 18-qubit state vector requires 2^18 complex amplitudes = 2^18 * 16 bytes = 4MB of memory, which is comfortable for browser WASM. This covers the vast majority of educational circuits and single-algorithm demonstrations. Circuits with 19-30 qubits are submitted to the server-side Rust-native solver which uses optimized SIMD and optional GPU acceleration. A 30-qubit state vector requires 2^30 * 16 bytes = 16GB, demanding server-class memory.

2. **Custom state-vector simulator in Rust rather than wrapping Qiskit Aer**: Building a custom simulator in Rust gives us full control over WASM compilation, memory layout, and gate fusion optimizations. Qiskit Aer is written in C++ with complex build dependencies (OpenMP, BLAS, optional GPU) that make WASM compilation impractical. Our Rust simulator uses cache-friendly state-vector layout and bit-manipulation-based gate application (applying a single-qubit gate touches exactly 2^(n-1) pairs of amplitudes, accessed via bit masking on the target qubit index), achieving performance within 2x of Qiskit Aer for circuits under 25 qubits.

3. **Canvas2D circuit editor with grid-based layout**: The circuit builder uses a discrete grid where qubits are horizontal wires and gate positions are columns (time steps). Unlike analog schematics which need freeform placement, quantum circuits have a highly regular structure (gates occupy cells in a qubit x time grid). Canvas2D provides efficient rendering and simple hit testing on this grid. Multi-qubit gates span multiple wire rows with connecting lines.

4. **WebGL + Three.js for state visualization**: Bloch spheres require 3D rendering with rotation animation, which maps naturally to Three.js. The amplitude bar chart must handle up to 2^18 = 262,144 bars for WASM circuits, requiring GPU-accelerated rendering. Phase information is encoded as bar color (hue = phase angle, saturation = magnitude), and the chart supports zoom into specific amplitude ranges.

5. **Python FastAPI microservice for Qiskit integration**: Qiskit provides battle-tested transpilation passes (gate decomposition, SWAP routing, optimization) and OpenQASM 3.0 parsing that would take months to reimplement in Rust. A separate Python service handles transpilation, OpenQASM parsing, and noise model construction from hardware calibration data. This service is stateless and horizontally scalable, communicating with the Rust API via internal HTTP.

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";  -- For fuzzy text search on gate libraries

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
    sim_quota_remaining INTEGER DEFAULT 50,  -- Monthly simulation runs
    max_qubits INTEGER DEFAULT 20,  -- Max qubits for simulation
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
    gpu_hours_monthly INTEGER DEFAULT 500,
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

-- Projects (quantum circuit workspace)
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    settings JSONB DEFAULT '{}',  -- Default qubit count, measurement shots, etc.
    is_public BOOLEAN NOT NULL DEFAULT false,
    forked_from UUID REFERENCES projects(id) ON DELETE SET NULL,
    thumbnail_url TEXT,
    tags TEXT[] DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);
CREATE INDEX projects_org_idx ON projects(org_id);
CREATE INDEX projects_updated_idx ON projects(updated_at DESC);
CREATE INDEX projects_public_idx ON projects(is_public) WHERE is_public = true;

-- Circuit Versions (version-controlled circuit definitions)
CREATE TABLE circuits (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    version INTEGER NOT NULL DEFAULT 1,
    parent_version_id UUID REFERENCES circuits(id) ON DELETE SET NULL,
    name TEXT NOT NULL DEFAULT 'main',
    circuit_data JSONB NOT NULL DEFAULT '{}',  -- {qubits, gates[], parameters{}}
    qubit_count INTEGER NOT NULL DEFAULT 2,
    gate_count INTEGER NOT NULL DEFAULT 0,
    depth INTEGER NOT NULL DEFAULT 0,
    qasm_code TEXT,  -- OpenQASM 3.0 representation
    message TEXT,  -- Version commit message
    created_by UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX circuits_project_idx ON circuits(project_id);
CREATE INDEX circuits_version_idx ON circuits(project_id, version DESC);

-- Experiments (simulation or hardware execution results)
CREATE TABLE experiments (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    circuit_id UUID NOT NULL REFERENCES circuits(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    backend_type TEXT NOT NULL DEFAULT 'simulator',  -- simulator | ibm | ionq | rigetti
    backend_name TEXT NOT NULL DEFAULT 'state_vector',  -- state_vector | density_matrix | ibm_sherbrooke | ionq_aria
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | running | completed | failed | cancelled
    execution_mode TEXT NOT NULL DEFAULT 'wasm',  -- wasm | server
    config JSONB NOT NULL DEFAULT '{}',  -- {shots, noise_model_id, precision, seed}
    results_url TEXT,  -- S3 URL for full result data
    results_summary JSONB,  -- Quick-access: {counts{}, expectation_values{}, fidelity}
    error_message TEXT,
    compute_time_ms INTEGER,
    cost_usd DECIMAL(10,4) DEFAULT 0,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX experiments_project_idx ON experiments(project_id);
CREATE INDEX experiments_circuit_idx ON experiments(circuit_id);
CREATE INDEX experiments_user_idx ON experiments(user_id);
CREATE INDEX experiments_status_idx ON experiments(status);
CREATE INDEX experiments_created_idx ON experiments(created_at DESC);

-- Simulation Jobs (for server-side execution only)
CREATE TABLE simulation_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    experiment_id UUID NOT NULL REFERENCES experiments(id) ON DELETE CASCADE,
    worker_id TEXT,
    gpu_allocated BOOLEAN DEFAULT false,
    memory_mb INTEGER DEFAULT 4096,
    priority INTEGER DEFAULT 0,  -- Higher = more urgent
    progress_pct REAL DEFAULT 0.0,
    progress_data JSONB DEFAULT '{}',  -- {gates_applied, total_gates, current_shots}
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX jobs_experiment_idx ON simulation_jobs(experiment_id);
CREATE INDEX jobs_worker_idx ON simulation_jobs(worker_id);

-- Noise Models (custom or hardware-calibrated)
CREATE TABLE noise_models (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    source TEXT NOT NULL DEFAULT 'custom',  -- custom | ibm_calibration | ionq_calibration
    config JSONB NOT NULL DEFAULT '{}',  -- {single_qubit_errors{}, two_qubit_errors{}, t1{}, t2{}, readout_errors{}}
    backend_reference TEXT,  -- Linked hardware backend name
    created_by UUID REFERENCES users(id) ON DELETE SET NULL,
    is_public BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX noise_models_source_idx ON noise_models(source);
CREATE INDEX noise_models_public_idx ON noise_models(is_public) WHERE is_public = true;

-- Circuit Templates (pre-built algorithm examples)
CREATE TABLE circuit_templates (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    category TEXT NOT NULL,  -- basics | algorithms | error_correction | variational
    description TEXT,
    circuit_data JSONB NOT NULL,
    qubit_count INTEGER NOT NULL,
    gate_count INTEGER NOT NULL,
    depth INTEGER NOT NULL,
    difficulty TEXT DEFAULT 'beginner',  -- beginner | intermediate | advanced
    sort_order INTEGER DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX templates_category_idx ON circuit_templates(category);

-- Usage Tracking (for billing enforcement)
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,  -- simulation_run | gpu_seconds | storage_bytes
    quantity REAL NOT NULL,
    metadata JSONB DEFAULT '{}',  -- {qubit_count, gate_count, backend}
    period_start DATE NOT NULL,
    period_end DATE NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX usage_user_period_idx ON usage_records(user_id, period_start, period_end);

-- Share Links
CREATE TABLE share_links (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    token TEXT UNIQUE NOT NULL,
    permission TEXT NOT NULL DEFAULT 'view',  -- view | edit
    expires_at TIMESTAMPTZ,
    created_by UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX share_links_token_idx ON share_links(token);
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
    pub sim_quota_remaining: i32,
    pub max_qubits: i32,
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
    pub settings: serde_json::Value,
    pub is_public: bool,
    pub forked_from: Option<Uuid>,
    pub thumbnail_url: Option<String>,
    pub tags: Vec<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Circuit {
    pub id: Uuid,
    pub project_id: Uuid,
    pub version: i32,
    pub parent_version_id: Option<Uuid>,
    pub name: String,
    pub circuit_data: serde_json::Value,
    pub qubit_count: i32,
    pub gate_count: i32,
    pub depth: i32,
    pub qasm_code: Option<String>,
    pub message: Option<String>,
    pub created_by: Uuid,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Experiment {
    pub id: Uuid,
    pub project_id: Uuid,
    pub circuit_id: Uuid,
    pub user_id: Uuid,
    pub backend_type: String,
    pub backend_name: String,
    pub status: String,
    pub execution_mode: String,
    pub config: serde_json::Value,
    pub results_url: Option<String>,
    pub results_summary: Option<serde_json::Value>,
    pub error_message: Option<String>,
    pub compute_time_ms: Option<i32>,
    pub cost_usd: Option<rust_decimal::Decimal>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct SimulationJob {
    pub id: Uuid,
    pub experiment_id: Uuid,
    pub worker_id: Option<String>,
    pub gpu_allocated: bool,
    pub memory_mb: i32,
    pub priority: i32,
    pub progress_pct: f32,
    pub progress_data: serde_json::Value,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct NoiseModel {
    pub id: Uuid,
    pub name: String,
    pub source: String,
    pub config: serde_json::Value,
    pub backend_reference: Option<String>,
    pub created_by: Option<Uuid>,
    pub is_public: bool,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct SimulationConfig {
    pub shots: Option<u32>,            // Number of measurement shots (default 1024)
    pub noise_model_id: Option<Uuid>,  // Optional noise model
    pub precision: Precision,          // f32 or f64
    pub seed: Option<u64>,            // RNG seed for reproducibility
    pub parameter_values: Option<serde_json::Value>,  // {theta: 1.57, phi: 0.785}
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum Precision {
    Single,  // f32 — 2x faster, 2x less memory
    Double,  // f64 — default, higher accuracy
}

#[derive(Debug, Deserialize, Serialize, Clone)]
pub struct QuantumGate {
    pub gate_type: GateType,
    pub target_qubits: Vec<usize>,   // Target qubit indices
    pub control_qubits: Vec<usize>,  // Control qubit indices (empty for single-qubit gates)
    pub parameters: Vec<f64>,        // Rotation angles [theta, phi, lambda]
    pub column: usize,               // Time step position in circuit grid
}

#[derive(Debug, Deserialize, Serialize, Clone)]
#[serde(rename_all = "snake_case")]
pub enum GateType {
    H, X, Y, Z, S, Sdg, T, Tdg,
    Rx, Ry, Rz, U3,
    Cnot, Cz, Swap, Toffoli,
    Measure, Barrier,
}
```

---

## Solver Architecture Deep-Dive

### Governing Equations and State Evolution

QubitStudio's core simulator implements **unitary state-vector evolution**, the standard formulation for ideal quantum circuit simulation. An `n`-qubit quantum state is represented as a complex vector with 2^n amplitudes:

```
|ψ⟩ = Σ_{k=0}^{2^n - 1} α_k |k⟩

where α_k ∈ ℂ and Σ |α_k|² = 1 (normalization)
```

Each quantum gate is a unitary operator U applied to specific qubits. For a single-qubit gate U acting on qubit `t` in an `n`-qubit system, the full 2^n × 2^n operator is:

```
U_full = I_{2^(n-t-1)} ⊗ U ⊗ I_{2^t}
```

However, we never construct the full matrix. Instead, we apply U directly to pairs of amplitudes. For target qubit `t`, we iterate over all 2^(n-1) pairs of indices that differ only in bit `t`:

```
For each index k where bit t is 0:
    k0 = k                    (bit t = 0)
    k1 = k | (1 << t)        (bit t = 1)

    |α_k0'⟩     U[0,0]  U[0,1]     |α_k0⟩
    |       | =  |               | · |      |
    |α_k1'⟩     U[1,0]  U[1,1]     |α_k1⟩
```

This approach has O(2^n) time complexity per gate (touching every amplitude once) rather than O(2^2n) for full matrix multiplication.

**Controlled gates** (CNOT, CZ, Toffoli) are applied conditionally: we only update amplitude pairs where all control qubit bits are 1:

```
For CNOT with control c, target t:
    For each index k where bit c = 1 and bit t = 0:
        k0 = k
        k1 = k | (1 << t)
        Apply X gate: swap(α_k0, α_k1)
```

**Measurement** collapses the state vector using Born's rule. The probability of measuring qubit `t` as |0⟩ is:

```
P(0) = Σ_{k: bit t of k = 0} |α_k|²
P(1) = 1 - P(0)
```

After measurement with outcome `m`, post-measurement state is renormalized:

```
For each amplitude α_k:
    if bit t of k ≠ m: α_k = 0
    else: α_k = α_k / √P(m)
```

**Parameterized gates** use standard rotation matrices:

```
Rx(θ) = [[cos(θ/2), -i·sin(θ/2)],       Ry(θ) = [[cos(θ/2), -sin(θ/2)],
          [-i·sin(θ/2), cos(θ/2)]]                  [sin(θ/2),  cos(θ/2)]]

Rz(θ) = [[e^{-iθ/2}, 0],                 U3(θ,φ,λ) = [[cos(θ/2),        -e^{iλ}·sin(θ/2)],
          [0, e^{iθ/2}]]                               [e^{iφ}·sin(θ/2), e^{i(φ+λ)}·cos(θ/2)]]
```

### Client/Server Split (WASM Threshold)

```
Circuit submitted → Qubit count extracted
    │
    ├── ≤18 qubits → WASM simulator (browser)
    │   ├── State vector = 2^18 * 16 bytes = 4MB (comfortable in WASM)
    │   ├── Instant startup, zero network latency
    │   ├── Real-time: state updates as gates added
    │   └── No server cost
    │
    └── >18 qubits → Server simulator (Rust native)
        ├── Job queued via Redis
        ├── Worker allocates memory (2^n * 16 bytes)
        │   ├── 20 qubits: 16MB
        │   ├── 25 qubits: 512MB
        │   ├── 30 qubits: 16GB
        │   └── 35 qubits: 512GB (Enterprise only)
        ├── Progress streamed via WebSocket (gates applied / total)
        └── Results stored in S3, URL returned
```

The 18-qubit WASM threshold was chosen because:
- 2^18 * 16 bytes = 4MB state vector fits comfortably in browser WASM memory (default 256MB heap)
- A 100-gate circuit on 18 qubits completes in <200ms in WASM on modern hardware
- 18 qubits covers: Bell states (2), GHZ states (3-10), basic QFT (8-12), Grover's search (4-15), most educational circuits
- Above 18 qubits: research-scale VQE, QAOA, quantum chemistry simulations that need server compute and potentially GPU acceleration

### WASM Compilation Pipeline

```toml
# solver-wasm/Cargo.toml
[package]
name = "qubitstudio-solver-wasm"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
wasm-bindgen = "0.2"
serde = { version = "1", features = ["derive"] }
serde-wasm-bindgen = "0.6"
js-sys = "0.3"
web-sys = { version = "0.3", features = ["console"] }
num-complex = "0.4"
rand = { version = "0.8", features = ["small_rng"] }
rand_chacha = "0.3"
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
      - run: wasm-opt -Oz solver-wasm/pkg/qubitstudio_solver_wasm_bg.wasm -o solver-wasm/pkg/qubitstudio_solver_wasm_bg.wasm
      - name: Upload to S3/CDN
        run: aws s3 cp solver-wasm/pkg/ s3://qubitstudio-wasm/v${{ github.sha }}/ --recursive
```

---

## Architecture Deep-Dives

### 1. Simulation API Handler (Rust/Axum)

The primary endpoint receives a simulation request, validates the circuit, determines WASM vs. server execution, and for server execution enqueues a job with Redis and streams progress via WebSocket.

```rust
// src/api/handlers/simulation.rs

use axum::{
    extract::{Path, State, Json},
    http::StatusCode,
    response::IntoResponse,
};
use uuid::Uuid;
use crate::{
    db::models::{Experiment, SimulationConfig, Circuit},
    state::AppState,
    auth::Claims,
    error::ApiError,
};

#[derive(serde::Deserialize)]
pub struct CreateSimulationRequest {
    pub config: SimulationConfig,
}

pub async fn create_simulation(
    State(state): State<AppState>,
    claims: Claims,
    Path((project_id, circuit_id)): Path<(Uuid, Uuid)>,
    Json(req): Json<CreateSimulationRequest>,
) -> Result<impl IntoResponse, ApiError> {
    // 1. Verify project ownership and fetch circuit
    let circuit = sqlx::query_as!(
        Circuit,
        "SELECT c.* FROM circuits c
         JOIN projects p ON c.project_id = p.id
         WHERE c.id = $1 AND c.project_id = $2
         AND (p.owner_id = $3 OR p.org_id IN (
             SELECT org_id FROM org_members WHERE user_id = $3
         ))",
        circuit_id,
        project_id,
        claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Circuit not found"))?;

    // 2. Check plan limits
    let user = sqlx::query_as!(
        crate::db::models::User,
        "SELECT * FROM users WHERE id = $1",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    if circuit.qubit_count > user.max_qubits {
        return Err(ApiError::PlanLimit(format!(
            "Your plan supports up to {} qubits. This circuit has {}. Upgrade to Pro for up to 30 qubits.",
            user.max_qubits, circuit.qubit_count
        )));
    }

    if user.sim_quota_remaining <= 0 && user.plan == "free" {
        return Err(ApiError::PlanLimit(
            "Monthly simulation quota exhausted. Upgrade to Pro for unlimited simulations.".into()
        ));
    }

    // 3. Determine execution mode based on qubit count
    let execution_mode = if circuit.qubit_count <= 18 {
        "wasm"
    } else {
        "server"
    };

    // 4. Estimate memory requirement for server execution
    let memory_mb = if execution_mode == "server" {
        let state_size_bytes = (1u64 << circuit.qubit_count as u64) * 16; // complex f64
        ((state_size_bytes as f64 / 1_048_576.0) * 1.5) as i32  // 1.5x for working memory
    } else {
        0
    };

    // 5. Create experiment record
    let experiment = sqlx::query_as!(
        Experiment,
        r#"INSERT INTO experiments
            (project_id, circuit_id, user_id, backend_type, backend_name,
             status, execution_mode, config)
        VALUES ($1, $2, $3, 'simulator', 'state_vector',
                $4, $5, $6)
        RETURNING *"#,
        project_id,
        circuit_id,
        claims.user_id,
        if execution_mode == "wasm" { "completed" } else { "pending" },
        execution_mode,
        serde_json::to_value(&req.config)?,
    )
    .fetch_one(&state.db)
    .await?;

    // 6. For server execution, enqueue job
    if execution_mode == "server" {
        let job = sqlx::query_as!(
            crate::db::models::SimulationJob,
            r#"INSERT INTO simulation_jobs (experiment_id, memory_mb, priority, gpu_allocated)
            VALUES ($1, $2, $3, $4) RETURNING *"#,
            experiment.id,
            memory_mb,
            if user.plan == "team" { 10 } else { 0 },
            circuit.qubit_count > 25,  // GPU for >25 qubits
        )
        .fetch_one(&state.db)
        .await?;

        // Publish to Redis job queue
        state.redis
            .publish("simulation:jobs", serde_json::to_string(&job.id)?)
            .await?;
    }

    // 7. Decrement simulation quota for free users
    if user.plan == "free" {
        sqlx::query!(
            "UPDATE users SET sim_quota_remaining = sim_quota_remaining - 1 WHERE id = $1",
            claims.user_id
        )
        .execute(&state.db)
        .await?;
    }

    Ok((StatusCode::CREATED, Json(experiment)))
}

pub async fn get_experiment(
    State(state): State<AppState>,
    claims: Claims,
    Path(experiment_id): Path<Uuid>,
) -> Result<Json<Experiment>, ApiError> {
    let experiment = sqlx::query_as!(
        Experiment,
        "SELECT e.* FROM experiments e
         JOIN projects p ON e.project_id = p.id
         WHERE e.id = $1
         AND (p.owner_id = $2 OR p.is_public = true OR p.org_id IN (
             SELECT org_id FROM org_members WHERE user_id = $2
         ))",
        experiment_id,
        claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Experiment not found"))?;

    Ok(Json(experiment))
}
```

### 2. Quantum State-Vector Simulator Core (Rust -- shared between WASM and native)

The core simulator that maintains the quantum state vector and applies gates via bit-manipulation-based amplitude pair iteration. This code compiles to both native (server) and WASM (browser) targets.

```rust
// solver-core/src/statevector.rs

use num_complex::Complex64;
use crate::gates::{Gate, GateType, GateMatrix};

/// Quantum state-vector simulator
pub struct StateVector {
    pub n_qubits: usize,
    pub amplitudes: Vec<Complex64>,  // 2^n complex amplitudes
    pub classical_bits: Vec<Option<u8>>,  // Measurement results per qubit
}

impl StateVector {
    /// Initialize |00...0⟩ state
    pub fn new(n_qubits: usize) -> Self {
        let dim = 1 << n_qubits;
        let mut amplitudes = vec![Complex64::new(0.0, 0.0); dim];
        amplitudes[0] = Complex64::new(1.0, 0.0);  // |00...0⟩
        Self {
            n_qubits,
            amplitudes,
            classical_bits: vec![None; n_qubits],
        }
    }

    /// Apply a single-qubit gate to target qubit
    pub fn apply_single_qubit_gate(&mut self, target: usize, matrix: &GateMatrix) {
        let dim = 1 << self.n_qubits;
        let target_mask = 1 << target;

        // Iterate over all pairs of amplitudes differing in bit `target`
        for k in 0..dim {
            if k & target_mask != 0 {
                continue;  // Process each pair only once (when target bit = 0)
            }
            let k0 = k;                   // Index with bit target = 0
            let k1 = k | target_mask;     // Index with bit target = 1

            let a0 = self.amplitudes[k0];
            let a1 = self.amplitudes[k1];

            // Matrix multiply: [U00 U01; U10 U11] * [a0; a1]
            self.amplitudes[k0] = matrix.u00 * a0 + matrix.u01 * a1;
            self.amplitudes[k1] = matrix.u10 * a0 + matrix.u11 * a1;
        }
    }

    /// Apply a controlled single-qubit gate (single control, single target)
    pub fn apply_controlled_gate(
        &mut self,
        control: usize,
        target: usize,
        matrix: &GateMatrix,
    ) {
        let dim = 1 << self.n_qubits;
        let control_mask = 1 << control;
        let target_mask = 1 << target;

        for k in 0..dim {
            // Only apply when control bit = 1 and target bit = 0
            if k & control_mask == 0 || k & target_mask != 0 {
                continue;
            }
            let k0 = k;                   // target bit = 0, control bit = 1
            let k1 = k | target_mask;     // target bit = 1, control bit = 1

            let a0 = self.amplitudes[k0];
            let a1 = self.amplitudes[k1];

            self.amplitudes[k0] = matrix.u00 * a0 + matrix.u01 * a1;
            self.amplitudes[k1] = matrix.u10 * a0 + matrix.u11 * a1;
        }
    }

    /// Apply a doubly-controlled gate (Toffoli-like)
    pub fn apply_doubly_controlled_gate(
        &mut self,
        control1: usize,
        control2: usize,
        target: usize,
        matrix: &GateMatrix,
    ) {
        let dim = 1 << self.n_qubits;
        let c1_mask = 1 << control1;
        let c2_mask = 1 << control2;
        let t_mask = 1 << target;

        for k in 0..dim {
            if k & c1_mask == 0 || k & c2_mask == 0 || k & t_mask != 0 {
                continue;
            }
            let k0 = k;
            let k1 = k | t_mask;

            let a0 = self.amplitudes[k0];
            let a1 = self.amplitudes[k1];

            self.amplitudes[k0] = matrix.u00 * a0 + matrix.u01 * a1;
            self.amplitudes[k1] = matrix.u10 * a0 + matrix.u11 * a1;
        }
    }

    /// Apply SWAP gate between two qubits
    pub fn apply_swap(&mut self, qubit_a: usize, qubit_b: usize) {
        let dim = 1 << self.n_qubits;
        let mask_a = 1 << qubit_a;
        let mask_b = 1 << qubit_b;

        for k in 0..dim {
            let bit_a = (k >> qubit_a) & 1;
            let bit_b = (k >> qubit_b) & 1;
            if bit_a != bit_b && bit_a == 0 {
                // Swap amplitudes where bits a,b differ (process once)
                let k_swapped = (k ^ mask_a) ^ mask_b;
                self.amplitudes.swap(k, k_swapped);
            }
        }
    }

    /// Measure a single qubit, collapsing the state
    pub fn measure(&mut self, qubit: usize, rng: &mut impl rand::Rng) -> u8 {
        let dim = 1 << self.n_qubits;
        let qubit_mask = 1 << qubit;

        // Calculate P(0) = sum of |α_k|² for all k where bit qubit = 0
        let mut prob_zero: f64 = 0.0;
        for k in 0..dim {
            if k & qubit_mask == 0 {
                prob_zero += self.amplitudes[k].norm_sqr();
            }
        }

        // Sample outcome
        let outcome: u8 = if rng.gen::<f64>() < prob_zero { 0 } else { 1 };

        // Collapse: zero out amplitudes inconsistent with outcome, renormalize
        let norm_factor = if outcome == 0 {
            prob_zero.sqrt()
        } else {
            (1.0 - prob_zero).sqrt()
        };

        for k in 0..dim {
            let bit = ((k >> qubit) & 1) as u8;
            if bit != outcome {
                self.amplitudes[k] = Complex64::new(0.0, 0.0);
            } else {
                self.amplitudes[k] /= norm_factor;
            }
        }

        self.classical_bits[qubit] = Some(outcome);
        outcome
    }

    /// Get measurement probabilities for all basis states (no collapse)
    pub fn get_probabilities(&self) -> Vec<f64> {
        self.amplitudes.iter().map(|a| a.norm_sqr()).collect()
    }

    /// Sample measurement outcomes via repeated Born-rule sampling
    pub fn sample_counts(
        &self,
        shots: u32,
        rng: &mut impl rand::Rng,
    ) -> std::collections::HashMap<String, u32> {
        let probs = self.get_probabilities();
        let mut counts = std::collections::HashMap::new();

        for _ in 0..shots {
            let r: f64 = rng.gen();
            let mut cumulative = 0.0;
            for (k, &p) in probs.iter().enumerate() {
                cumulative += p;
                if r < cumulative {
                    // Format as binary string: qubit n-1 (MSB) ... qubit 0 (LSB)
                    let bitstring = format!("{:0>width$b}", k, width = self.n_qubits);
                    *counts.entry(bitstring).or_insert(0) += 1;
                    break;
                }
            }
        }
        counts
    }

    /// Calculate the fidelity between this state and an ideal target state
    pub fn fidelity(&self, target: &[Complex64]) -> f64 {
        assert_eq!(self.amplitudes.len(), target.len());
        let inner_product: Complex64 = self.amplitudes.iter()
            .zip(target.iter())
            .map(|(a, b)| a.conj() * b)
            .sum();
        inner_product.norm_sqr()
    }
}

/// Compile and execute a full circuit
pub fn simulate_circuit(
    n_qubits: usize,
    gates: &[Gate],
    config: &SimulationConfig,
) -> Result<SimulationResult, SimulatorError> {
    let mut sv = StateVector::new(n_qubits);
    let mut rng = match config.seed {
        Some(seed) => rand_chacha::ChaCha8Rng::seed_from_u64(seed),
        None => rand_chacha::ChaCha8Rng::from_entropy(),
    };

    // Apply gates in circuit order (sorted by column, then qubit)
    for gate in gates {
        let matrix = gate.to_matrix(&config.parameter_values);
        match gate.gate_type {
            GateType::Measure => {
                sv.measure(gate.target_qubits[0], &mut rng);
            }
            GateType::Barrier => {} // No-op, visual only
            GateType::Swap => {
                sv.apply_swap(gate.target_qubits[0], gate.target_qubits[1]);
            }
            GateType::Toffoli => {
                sv.apply_doubly_controlled_gate(
                    gate.control_qubits[0],
                    gate.control_qubits[1],
                    gate.target_qubits[0],
                    &matrix,
                );
            }
            _ if !gate.control_qubits.is_empty() => {
                sv.apply_controlled_gate(
                    gate.control_qubits[0],
                    gate.target_qubits[0],
                    &matrix,
                );
            }
            _ => {
                sv.apply_single_qubit_gate(gate.target_qubits[0], &matrix);
            }
        }
    }

    // Generate results
    let probabilities = sv.get_probabilities();
    let counts = if let Some(shots) = config.shots {
        Some(sv.sample_counts(shots, &mut rng))
    } else {
        None
    };

    Ok(SimulationResult {
        state_vector: sv.amplitudes.clone(),
        probabilities,
        counts,
        classical_bits: sv.classical_bits.clone(),
    })
}

#[derive(Debug, Serialize)]
pub struct SimulationResult {
    pub state_vector: Vec<Complex64>,
    pub probabilities: Vec<f64>,
    pub counts: Option<std::collections::HashMap<String, u32>>,
    pub classical_bits: Vec<Option<u8>>,
}

#[derive(Debug)]
pub enum SimulatorError {
    InvalidQubitIndex { qubit: usize, n_qubits: usize },
    InvalidGate(String),
    MemoryExceeded { required_mb: usize, available_mb: usize },
}
```

### 3. Quantum State Visualization Component (React + WebGL + Three.js)

The frontend state visualization dashboard that renders Bloch spheres using Three.js and amplitude bar charts using WebGL for up to 2^18 basis states.

```typescript
// frontend/src/components/StateVisualization/AmplitudeBarChart.tsx

import { useRef, useEffect, useCallback } from 'react';
import { useSimulationStore } from '../../stores/simulationStore';

interface AmplitudeData {
  magnitudes: Float64Array;   // |α_k| for each basis state
  phases: Float64Array;       // arg(α_k) for each basis state, in radians
  n_qubits: number;
}

export function AmplitudeBarChart() {
  const canvasRef = useRef<HTMLCanvasElement>(null);
  const glRef = useRef<WebGL2RenderingContext | null>(null);
  const programRef = useRef<WebGLProgram | null>(null);
  const { amplitudeData, visibleRange, setVisibleRange } = useSimulationStore();

  // Initialize WebGL
  useEffect(() => {
    const canvas = canvasRef.current;
    if (!canvas) return;

    const gl = canvas.getContext('webgl2', { antialias: true, alpha: false });
    if (!gl) return;
    glRef.current = gl;

    // Compile shaders for bar rendering with phase-based coloring
    const vsSource = `#version 300 es
      in vec2 a_position;     // (bar_index, bar_height)
      in float a_phase;       // Phase angle in radians
      uniform vec2 u_xRange;  // (min_index, max_index) visible range
      uniform float u_yMax;   // Maximum amplitude for Y scaling
      uniform vec2 u_viewport;

      out float v_phase;
      out float v_height;

      void main() {
        float x = 2.0 * (a_position.x - u_xRange.x) / (u_xRange.y - u_xRange.x) - 1.0;
        float y = a_position.y / u_yMax;  // Normalize to [0, 1]
        y = y * 2.0 - 1.0;               // Map to [-1, 1] clip space
        gl_Position = vec4(x, y, 0.0, 1.0);
        v_phase = a_phase;
        v_height = a_position.y / u_yMax;
      }
    `;
    const fsSource = `#version 300 es
      precision mediump float;
      in float v_phase;
      in float v_height;
      out vec4 fragColor;

      // HSV to RGB conversion for phase-based coloring
      vec3 hsv2rgb(vec3 c) {
        vec4 K = vec4(1.0, 2.0/3.0, 1.0/3.0, 3.0);
        vec3 p = abs(fract(c.xxx + K.xyz) * 6.0 - K.www);
        return c.z * mix(K.xxx, clamp(p - K.xxx, 0.0, 1.0), c.y);
      }

      void main() {
        // Hue = phase angle mapped to [0, 1], saturation = 0.8, value = brightness
        float hue = (v_phase + 3.14159) / (2.0 * 3.14159); // Map [-pi, pi] to [0, 1]
        float brightness = 0.4 + 0.6 * v_height;  // Brighter bars for larger amplitudes
        vec3 rgb = hsv2rgb(vec3(hue, 0.85, brightness));
        fragColor = vec4(rgb, 1.0);
      }
    `;

    const program = createShaderProgram(gl, vsSource, fsSource);
    programRef.current = program;

    return () => {
      gl.deleteProgram(program);
    };
  }, []);

  // Render amplitude bars
  const render = useCallback(() => {
    const gl = glRef.current;
    const program = programRef.current;
    if (!gl || !program || !amplitudeData) return;

    gl.viewport(0, 0, gl.canvas.width, gl.canvas.height);
    gl.clearColor(0.08, 0.08, 0.12, 1.0);
    gl.clear(gl.COLOR_BUFFER_BIT);
    gl.useProgram(program);

    const { magnitudes, phases, n_qubits } = amplitudeData;
    const numStates = 1 << n_qubits;
    const { min: rangeMin, max: rangeMax } = visibleRange;

    // Find max amplitude for Y scaling
    let yMax = 0;
    for (let i = rangeMin; i < rangeMax && i < numStates; i++) {
      if (magnitudes[i] > yMax) yMax = magnitudes[i];
    }
    yMax = Math.max(yMax * 1.1, 0.01); // 10% headroom, avoid zero

    // Build vertex data for visible bars (2 triangles per bar = 6 vertices)
    const visibleCount = Math.min(rangeMax - rangeMin, numStates);
    const barWidth = 0.8; // 80% of slot width
    const vertices = new Float32Array(visibleCount * 12); // 6 vertices * 2 coords
    const phaseData = new Float32Array(visibleCount * 6);  // 6 vertices * 1 phase

    let idx = 0;
    let pidx = 0;
    for (let i = rangeMin; i < rangeMax && i < numStates; i++) {
      const x = i;
      const h = magnitudes[i];
      const p = phases[i];
      const hw = barWidth / 2;

      // Triangle 1: bottom-left, bottom-right, top-right
      vertices[idx++] = x - hw; vertices[idx++] = 0;
      vertices[idx++] = x + hw; vertices[idx++] = 0;
      vertices[idx++] = x + hw; vertices[idx++] = h;
      // Triangle 2: bottom-left, top-right, top-left
      vertices[idx++] = x - hw; vertices[idx++] = 0;
      vertices[idx++] = x + hw; vertices[idx++] = h;
      vertices[idx++] = x - hw; vertices[idx++] = h;

      for (let v = 0; v < 6; v++) phaseData[pidx++] = p;
    }

    // Upload and draw
    const posBuffer = gl.createBuffer();
    gl.bindBuffer(gl.ARRAY_BUFFER, posBuffer);
    gl.bufferData(gl.ARRAY_BUFFER, vertices.slice(0, idx), gl.STREAM_DRAW);
    const posLoc = gl.getAttribLocation(program, 'a_position');
    gl.enableVertexAttribArray(posLoc);
    gl.vertexAttribPointer(posLoc, 2, gl.FLOAT, false, 0, 0);

    const phaseBuffer = gl.createBuffer();
    gl.bindBuffer(gl.ARRAY_BUFFER, phaseBuffer);
    gl.bufferData(gl.ARRAY_BUFFER, phaseData.slice(0, pidx), gl.STREAM_DRAW);
    const phaseLoc = gl.getAttribLocation(program, 'a_phase');
    gl.enableVertexAttribArray(phaseLoc);
    gl.vertexAttribPointer(phaseLoc, 1, gl.FLOAT, false, 0, 0);

    const xRangeLoc = gl.getUniformLocation(program, 'u_xRange');
    const yMaxLoc = gl.getUniformLocation(program, 'u_yMax');
    gl.uniform2f(xRangeLoc, rangeMin - 0.5, rangeMax + 0.5);
    gl.uniform1f(yMaxLoc, yMax);

    gl.drawArrays(gl.TRIANGLES, 0, idx / 2);

    gl.deleteBuffer(posBuffer);
    gl.deleteBuffer(phaseBuffer);
  }, [amplitudeData, visibleRange]);

  useEffect(() => {
    const animId = requestAnimationFrame(render);
    return () => cancelAnimationFrame(animId);
  }, [render]);

  // Handle mouse wheel for zoom into specific basis states
  const handleWheel = useCallback((e: React.WheelEvent) => {
    e.preventDefault();
    if (!amplitudeData) return;

    const canvas = canvasRef.current;
    if (!canvas) return;

    const rect = canvas.getBoundingClientRect();
    const xFrac = (e.clientX - rect.left) / rect.width;
    const range = visibleRange.max - visibleRange.min;
    const center = visibleRange.min + xFrac * range;

    const zoomFactor = e.deltaY > 0 ? 1.3 : 0.7;
    const newRange = Math.max(4, Math.min(1 << amplitudeData.n_qubits, range * zoomFactor));
    setVisibleRange({
      min: Math.max(0, Math.round(center - xFrac * newRange)),
      max: Math.min(1 << amplitudeData.n_qubits, Math.round(center + (1 - xFrac) * newRange)),
    });
  }, [amplitudeData, visibleRange, setVisibleRange]);

  return (
    <div className="amplitude-chart flex flex-col h-full bg-gray-950">
      <div className="px-3 py-2 text-sm text-gray-400 border-b border-gray-800 flex justify-between">
        <span>Amplitude Distribution</span>
        <span>
          Showing states {visibleRange.min}–{visibleRange.max - 1}
          {amplitudeData && ` of ${1 << amplitudeData.n_qubits}`}
        </span>
      </div>
      <div className="flex-1 relative">
        <canvas
          ref={canvasRef}
          className="w-full h-full"
          onWheel={handleWheel}
        />
        <BasisStateAxis range={visibleRange} n_qubits={amplitudeData?.n_qubits ?? 0} />
      </div>
      <PhaseColorLegend />
    </div>
  );
}

function formatBasisState(index: number, n_qubits: number): string {
  return `|${index.toString(2).padStart(n_qubits, '0')}⟩`;
}
```

### 4. Server-Side Simulation Worker (Rust + Redis)

Background worker that processes server-side simulation jobs from Redis, runs the native solver with optional SIMD optimization, streams progress via WebSocket, and stores results in S3.

```rust
// src/workers/simulation_worker.rs

use std::sync::Arc;
use aws_sdk_s3::Client as S3Client;
use redis::AsyncCommands;
use sqlx::PgPool;
use uuid::Uuid;

use crate::solver::{
    statevector::{simulate_circuit, StateVector, SimulationResult},
    gates::parse_circuit_gates,
};
use crate::db::models::{SimulationJob, SimulationConfig};
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
        tracing::info!("Quantum simulation worker started, listening for jobs...");

        loop {
            // Block-pop from Redis job queue
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
        // 1. Fetch job and experiment details
        let job = sqlx::query_as!(
            SimulationJob,
            "UPDATE simulation_jobs SET started_at = NOW(), worker_id = $2
             WHERE id = $1 RETURNING *",
            job_id, hostname::get()?.to_string_lossy().to_string()
        )
        .fetch_one(&self.db)
        .await?;

        let experiment = sqlx::query_as!(
            crate::db::models::Experiment,
            "UPDATE experiments SET status = 'running', started_at = NOW()
             WHERE id = $1 RETURNING *",
            job.experiment_id
        )
        .fetch_one(&self.db)
        .await?;

        // 2. Fetch circuit definition
        let circuit = sqlx::query_as!(
            crate::db::models::Circuit,
            "SELECT * FROM circuits WHERE id = $1",
            experiment.circuit_id
        )
        .fetch_one(&self.db)
        .await?;

        // 3. Parse circuit gates and config
        let gates = parse_circuit_gates(&circuit.circuit_data)?;
        let config: SimulationConfig = serde_json::from_value(experiment.config.clone())?;
        let n_qubits = circuit.qubit_count as usize;
        let total_gates = gates.len();

        tracing::info!(
            "Processing job {job_id}: {n_qubits} qubits, {total_gates} gates, {:?} shots",
            config.shots
        );

        // 4. Run simulation with progress callbacks
        let progress_tx = self.broadcaster.get_channel(experiment.id);
        let mut sv = StateVector::new(n_qubits);
        let mut rng = match config.seed {
            Some(seed) => rand_chacha::ChaCha8Rng::seed_from_u64(seed),
            None => rand_chacha::ChaCha8Rng::from_entropy(),
        };

        for (i, gate) in gates.iter().enumerate() {
            let matrix = gate.to_matrix(&config.parameter_values);
            sv.apply_gate(gate, &matrix, &mut rng);

            // Report progress every 10 gates or at completion
            if i % 10 == 0 || i == total_gates - 1 {
                let pct = ((i + 1) as f32 / total_gates as f32) * 100.0;
                let _ = progress_tx.send(serde_json::json!({
                    "phase": "gate_application",
                    "progress_pct": pct,
                    "gates_applied": i + 1,
                    "total_gates": total_gates,
                }));

                sqlx::query!(
                    "UPDATE simulation_jobs SET progress_pct = $1,
                     progress_data = $2 WHERE id = $3",
                    pct,
                    serde_json::json!({"gates_applied": i + 1, "total_gates": total_gates}),
                    job_id
                )
                .execute(&self.db)
                .await?;
            }
        }

        // 5. Generate measurement counts if shots requested
        let counts = if let Some(shots) = config.shots {
            let _ = progress_tx.send(serde_json::json!({
                "phase": "measurement_sampling",
                "progress_pct": 100.0,
                "shots": shots,
            }));
            Some(sv.sample_counts(shots, &mut rng))
        } else {
            None
        };

        // 6. Serialize results
        let result = SimulationResult {
            state_vector: sv.amplitudes.clone(),
            probabilities: sv.get_probabilities(),
            counts,
            classical_bits: sv.classical_bits.clone(),
        };
        let result_data = serde_json::to_vec(&result)?;

        // 7. Upload results to S3
        let s3_key = format!("results/{}/{}.json", experiment.project_id, experiment.id);
        self.s3.put_object()
            .bucket("qubitstudio-results")
            .key(&s3_key)
            .body(result_data.into())
            .content_type("application/json")
            .send()
            .await?;

        let results_url = format!("s3://qubitstudio-results/{}", s3_key);

        // 8. Generate summary for quick access (avoid downloading full state vector)
        let summary = serde_json::json!({
            "n_qubits": n_qubits,
            "total_gates": total_gates,
            "counts": result.counts,
            "top_probabilities": top_n_probabilities(&result.probabilities, 10, n_qubits),
        });

        // 9. Update database with completed status
        sqlx::query!(
            "UPDATE experiments SET status = 'completed', results_url = $2,
             results_summary = $3, completed_at = NOW(),
             compute_time_ms = EXTRACT(EPOCH FROM (NOW() - started_at))::int * 1000
             WHERE id = $1",
            experiment.id, results_url, summary
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

        // 10. Notify client via WebSocket
        self.broadcaster.send(experiment.id, serde_json::json!({
            "status": "completed",
            "results_url": results_url,
            "summary": summary,
        }))?;

        tracing::info!("Job {job_id} completed: {n_qubits} qubits, {total_gates} gates");
        Ok(())
    }

    async fn mark_failed(&self, job_id: Uuid, error: &str) -> anyhow::Result<()> {
        sqlx::query!(
            "UPDATE simulation_jobs SET completed_at = NOW() WHERE id = $1",
            job_id
        )
        .execute(&self.db)
        .await?;

        let experiment_id: Uuid = sqlx::query_scalar!(
            "SELECT experiment_id FROM simulation_jobs WHERE id = $1",
            job_id
        )
        .fetch_one(&self.db)
        .await?;

        sqlx::query!(
            "UPDATE experiments SET status = 'failed', error_message = $2,
             completed_at = NOW() WHERE id = $1",
            experiment_id, error
        )
        .execute(&self.db)
        .await?;

        Ok(())
    }
}

fn top_n_probabilities(
    probs: &[f64],
    n: usize,
    n_qubits: usize,
) -> Vec<serde_json::Value> {
    let mut indexed: Vec<(usize, f64)> = probs.iter()
        .enumerate()
        .filter(|(_, &p)| p > 1e-10)
        .map(|(i, &p)| (i, p))
        .collect();
    indexed.sort_by(|a, b| b.1.partial_cmp(&a.1).unwrap());
    indexed.truncate(n);
    indexed.iter().map(|(idx, prob)| {
        serde_json::json!({
            "state": format!("{:0>width$b}", idx, width = n_qubits),
            "probability": prob,
        })
    }).collect()
}
```

---

## Phase Breakdown

### Phase 1 -- Scaffold + Auth + Database (Days 1-4)

**Day 1: Project initialization and Rust backend scaffold**
```bash
cargo init qubitstudio-api
cd qubitstudio-api
cargo add axum tokio serde serde_json sqlx uuid chrono tower-http jsonwebtoken bcrypt
cargo add --features postgres,runtime-tokio-rustls,uuid,chrono,json sqlx
cargo add aws-sdk-s3 redis
```
- `src/main.rs` -- Axum app with CORS, tracing, graceful shutdown
- `src/config.rs` -- Environment-based configuration (DATABASE_URL, JWT_SECRET, S3_BUCKET, REDIS_URL)
- `src/state.rs` -- AppState struct (PgPool, Redis, S3Client)
- `src/error.rs` -- ApiError enum with IntoResponse
- `Dockerfile` -- Multi-stage build (builder + runtime)
- `docker-compose.yml` -- PostgreSQL, Redis, MinIO (S3-compatible)

**Day 2: Database schema and migrations**
- `migrations/001_initial.sql` -- All 10 tables: users, organizations, org_members, projects, circuits, experiments, simulation_jobs, noise_models, circuit_templates, usage_records, share_links
- `src/db/mod.rs` -- Database pool initialization
- `src/db/models.rs` -- All SQLx structs with FromRow derives
- Run `sqlx migrate run` to apply schema
- Seed script: initial circuit templates (Bell state, GHZ, QFT, Grover's search, quantum teleportation)

**Day 3: Authentication system**
- `src/auth/mod.rs` -- JWT token generation and validation middleware
- `src/auth/oauth.rs` -- Google and GitHub OAuth 2.0 flow handlers
- `src/api/handlers/auth.rs` -- Register, login, OAuth callback, refresh token, me endpoints
- Password hashing with bcrypt (cost factor 12)
- JWT with 24h access token + 30d refresh token
- Auth middleware that extracts Claims from Authorization header

**Day 4: User and project CRUD**
- `src/api/handlers/users.rs` -- Get profile, update profile, delete account
- `src/api/handlers/projects.rs` -- Create, list, get, update, delete, fork project
- `src/api/handlers/circuits.rs` -- Create circuit version, list versions, get circuit, diff two versions
- `src/api/handlers/orgs.rs` -- Create org, invite member, list members, remove member
- `src/api/router.rs` -- All route definitions with auth middleware
- Integration tests: auth flow, project CRUD, authorization checks

### Phase 2 -- Quantum Solver Core (Days 5-12)

**Day 5: State-vector framework and gate definitions**
- `solver-core/` -- New Rust workspace member (shared between native and WASM)
- `solver-core/src/statevector.rs` -- StateVector struct with `new()`, `apply_single_qubit_gate()`, `apply_controlled_gate()`, `measure()`, `get_probabilities()`, `sample_counts()`
- `solver-core/src/gates/mod.rs` -- GateType enum, GateMatrix struct, Gate trait definition
- `solver-core/src/gates/matrices.rs` -- Constant matrices for H, X, Y, Z, S, T, CNOT, CZ
- Unit tests: verify H|0> = |+>, X|0> = |1>, CNOT|10> = |11>

**Day 6: Parameterized gates and rotation matrices**
- `solver-core/src/gates/rotations.rs` -- Rx(theta), Ry(theta), Rz(theta), U3(theta, phi, lambda) matrix constructors
- `solver-core/src/gates/multi_qubit.rs` -- SWAP implementation via bit manipulation, Toffoli gate
- Symbolic parameter resolution: replace named parameters (theta, phi) with numeric values before simulation
- Tests: Rx(pi) = X, Ry(pi) = iY, Rz(pi) = iZ (up to global phase), SWAP|01> = |10>

**Day 7: Measurement and sampling**
- `solver-core/src/measurement.rs` -- Born-rule measurement, state collapse, multi-shot sampling
- Mid-circuit measurement support: measure qubit, collapse state, continue with remaining gates
- Classical conditional gates: apply gate only if classical bit has specific value
- Tests: measure |+> gives 50/50 distribution over 10,000 shots (chi-squared test), Bell state measurement correlations

**Day 8: Circuit compiler and gate ordering**
- `solver-core/src/compiler.rs` -- Parse circuit_data JSON into ordered gate sequence
- Gate ordering: topological sort by column (time step), parallel gates within same column
- Gate validation: check qubit indices in range, no duplicate qubits in multi-qubit gate
- Circuit depth calculation: maximum column index + 1
- Gate count by type: useful for circuit statistics display
- Tests: various circuit JSON inputs produce correct gate sequences

**Day 9: Simulation performance optimization**
- SIMD optimization for single-qubit gates: process 4 amplitude pairs simultaneously using packed f64 operations
- Gate fusion: combine consecutive single-qubit gates on the same qubit into a single 2x2 matrix multiply
- Cache-friendly state-vector layout: amplitudes stored contiguously for optimal L1/L2 cache utilization during pair iteration
- Benchmarks: measure gates/second for 12, 15, 18 qubit circuits, compare fused vs. unfused

**Day 10: Multi-shot sampling optimization**
- Fast sampling: precompute cumulative distribution function (CDF), use binary search for each shot
- For circuits without mid-circuit measurement: compute state vector once, sample from probability distribution repeatedly
- Batch sampling: generate all shots in a single pass, count occurrences using HashMap
- Tests: 1M shots on 10-qubit circuit completes in <500ms, distribution matches theoretical

**Day 11: OpenQASM 3.0 integration (via Python service)**
- `transpilation-service/` -- Python FastAPI microservice
- `transpilation-service/main.py` -- FastAPI app with endpoints: `/parse_qasm`, `/generate_qasm`, `/transpile`
- `transpilation-service/parser.py` -- Parse OpenQASM 3.0 string to circuit_data JSON using qiskit.qasm3
- `transpilation-service/generator.py` -- Generate OpenQASM 3.0 string from circuit_data JSON
- `transpilation-service/transpiler.py` -- Qiskit transpilation passes (optimization levels 0-3)
- Bidirectional sync protocol: frontend sends circuit_data, receives QASM; or sends QASM, receives circuit_data
- Tests: round-trip (circuit -> QASM -> circuit) preserves all gates, parameters, and qubit assignments

**Day 12: Error handling, edge cases, and solver test suite**
- Handle edge cases: empty circuit, single-qubit circuit, all-measurement circuit, parameterized gate with missing parameter
- Memory limit checks: reject circuits that would exceed available memory before allocation
- Graceful error messages: "Circuit requires 16GB memory (30 qubits). Server-side execution required."
- Comprehensive test suite: `solver-core/tests/` with unit tests for every gate type, integration tests for standard algorithms
- Tests: Bell state, GHZ state, QFT on 4 qubits, Grover's search on 3 qubits -- verify output state vectors against known analytical results

### Phase 3 -- WASM Build + Frontend Foundation (Days 13-18)

**Day 13: WASM solver compilation**
- `solver-wasm/` -- New workspace member for WASM target
- `solver-wasm/Cargo.toml` -- wasm-bindgen, serde-wasm-bindgen, num-complex dependencies
- `solver-wasm/src/lib.rs` -- WASM entry points: `simulate(circuit_json, config_json) -> result_json`
- `wasm-pack build --target web --release` pipeline
- `wasm-opt -Oz` for size optimization (target <1.5MB gzipped)
- JavaScript wrapper: `QuantumSolver` class that loads WASM and provides async API
- Test: run Bell state simulation in headless browser, verify output matches native solver

**Day 14: Frontend scaffold and project dashboard**
```bash
npm create vite@latest frontend -- --template react-ts
cd frontend
npm i zustand @tanstack/react-query axios three @types/three
npm i @monaco-editor/react
```
- `src/App.tsx` -- Router, auth context, layout with sidebar navigation
- `src/stores/authStore.ts` -- Auth state (JWT, user profile)
- `src/stores/projectStore.ts` -- Zustand store for project state
- `src/pages/Dashboard.tsx` -- Project grid with circuit thumbnails, qubit counts, last experiment results
- `src/pages/Login.tsx` and `Register.tsx` -- Auth pages with OAuth buttons
- `src/lib/api.ts` -- Axios client with JWT interceptor

**Day 15: Circuit builder -- canvas and qubit wires**
- `src/components/CircuitBuilder/CircuitCanvas.tsx` -- Canvas2D renderer with qubit wire grid layout
- `src/components/CircuitBuilder/QubitWire.tsx` -- Horizontal wire rendering with qubit labels (|q0>, |q1>, ...)
- `src/stores/circuitStore.ts` -- Zustand store for circuit state (qubits[], gates[], selection, undo/redo stack)
- Grid system: configurable cell size (40x50px), gate snaps to grid cells, multi-qubit gates span rows
- Pan and zoom: mouse wheel zoom, middle-click pan, zoom-to-fit
- Qubit management: add/remove qubits, reorder via drag

**Day 16: Circuit builder -- gate palette and placement**
- `src/components/CircuitBuilder/GatePalette.tsx` -- Categorized gate library sidebar:
  - Single-qubit: H, X, Y, Z, S, T (with symbols and colors)
  - Rotation: Rx, Ry, Rz (with parameter input)
  - Multi-qubit: CNOT, CZ, SWAP, Toffoli
  - Utility: Measure, Barrier
- `src/components/CircuitBuilder/GateSymbol.tsx` -- Canvas rendering of individual gate symbols (colored boxes with labels)
- `src/components/CircuitBuilder/ConnectionLine.tsx` -- Vertical lines connecting control and target qubits for multi-qubit gates
- Drag-and-drop from palette to canvas: ghost preview during drag, snap to nearest valid grid cell
- Click gate to select: property panel shows gate type, parameters, qubit assignment
- Delete: select gate + Del key or right-click context menu

**Day 17: Circuit builder -- editing features and parameter controls**
- `src/components/CircuitBuilder/ParameterPanel.tsx` -- Slider and numeric input for rotation gate parameters (theta, phi, lambda)
- `src/components/CircuitBuilder/CircuitToolbar.tsx` -- Add qubit, remove qubit, templates dropdown, clear circuit, undo/redo buttons
- Keyboard shortcuts: Ctrl+Z/Y undo/redo, Del delete selected, H/X/Z quick-place gates
- Copy-paste circuit blocks: select region, Ctrl+C, click destination, Ctrl+V
- Gate reordering: drag gates to different columns (time steps)
- Circuit statistics display: qubit count, gate count, depth, gate breakdown by type

**Day 18: Code editor -- Monaco integration and bidirectional sync**
- `src/components/CodeEditor/QASMEditor.tsx` -- Monaco Editor with OpenQASM 3.0 syntax highlighting
- `src/lib/qasmLanguage.ts` -- Monaco language definition for OpenQASM 3.0 (keywords, operators, built-in gates)
- Bidirectional sync engine:
  - Circuit -> QASM: on circuit change, call transpilation service `/generate_qasm`, update editor
  - QASM -> Circuit: on code edit (debounced 500ms), call `/parse_qasm`, update circuit canvas
  - Conflict resolution: last-edit-wins with visual indicator showing sync direction
- Tabbed bottom panel: Code Editor | State Visualization | Measurement Results
- Syntax error highlighting: red squiggles with error messages from parser

### Phase 4 -- State Visualization + Simulation Integration (Days 19-24)

**Day 19: Bloch sphere visualization (Three.js)**
- `src/components/StateVisualization/BlochSphere.tsx` -- Three.js Bloch sphere with state vector arrow
- Sphere rendering: wireframe sphere with X, Y, Z axis labels, equator and meridian lines
- State arrow: colored arrow from center to point on sphere surface, computed from single-qubit reduced density matrix
- Animation: smooth transition when gate is applied (rotation animation along gate axis)
- Multi-qubit display: array of Bloch spheres (one per qubit) showing reduced state of each qubit
- Entanglement indicator: sphere becomes translucent/gray when qubit is entangled (reduced state is mixed)

**Day 20: Amplitude bar chart and measurement histogram**
- `src/components/StateVisualization/AmplitudeBarChart.tsx` -- WebGL bar chart for state vector amplitudes
- Phase coloring: bar hue encodes phase angle (0=red, pi/2=green, pi=cyan, 3pi/2=blue)
- Zoom: mouse wheel to zoom into specific basis state ranges, show binary labels on X-axis
- Hover tooltip: show `|state>`, amplitude (real + imaginary), probability, phase angle
- `src/components/StateVisualization/MeasurementHistogram.tsx` -- D3 bar chart for shot-based measurement results
- Sort by: count (descending), state (lexicographic), probability difference from ideal
- Statistical annotations: expected vs. observed count, chi-squared test p-value

**Day 21: Real-time simulation integration**
- `src/hooks/useWasmSolver.ts` -- React hook that loads WASM module and provides `simulate()` function
- Real-time mode: for <=18 qubits, re-simulate on every circuit change (gate added/removed/modified, parameter changed)
- Debounced simulation: 100ms debounce on circuit changes to avoid excessive re-computation
- State caching: cache intermediate states at each circuit column, only re-simulate from the point of modification
- Loading states: spinner during simulation, immediate display of cached results during re-simulation
- `src/stores/simulationStore.ts` -- Store for current simulation state (state vector, probabilities, counts, loading)

**Day 22: Server-side simulation integration**
- `src/hooks/useServerSimulation.ts` -- React hook for server-side simulation (>18 qubits)
- Submit flow: POST circuit to API -> receive experiment ID -> subscribe to WebSocket for progress
- Progress display: progress bar showing "Applying gate 45/120" with estimated time remaining
- `src/hooks/useSimulationProgress.ts` -- WebSocket hook for live progress streaming
- Auto-fallback: if WASM simulation times out (>5 seconds), suggest server-side execution
- Result loading: fetch results from S3 presigned URL, parse state vector, update visualization

**Day 23: Experiment tracking and results comparison**
- `src/pages/Experiments.tsx` -- List of all experiments for a project with status, backend, runtime
- `src/components/Experiments/ExperimentCard.tsx` -- Card showing circuit version, backend, result summary
- `src/components/Experiments/ComparisonView.tsx` -- Side-by-side measurement histograms for two experiments
- Automatic experiment logging: every simulation run creates an experiment record
- Experiment filtering: by backend (simulator/hardware), status (completed/failed), date range
- Result summary: top-5 measurement outcomes, total shots, execution time

**Day 24: Circuit templates and share links**
- `src/pages/Templates.tsx` -- Template gallery with preview diagrams and descriptions
- Templates seeded in database:
  - Bell state (2 qubits, depth 2)
  - GHZ state (3 qubits, depth 3)
  - Quantum Fourier Transform (4 qubits, depth 12)
  - Grover's search (3 qubits, depth 8)
  - Quantum teleportation (3 qubits, depth 6)
  - Deutsch-Jozsa (4 qubits, depth 5)
- One-click "Use Template" -> creates new project with pre-built circuit
- Share links: generate read-only URL for circuit + latest experiment results
- `src/api/handlers/share.rs` -- Create share link, validate share token, serve shared project

### Phase 5 -- Results UI + Post-Processing (Days 25-29)

**Day 25: Phase wheel and Q-sphere visualizations**
- `src/components/StateVisualization/PhaseWheel.tsx` -- Circular plot showing relative phases between basis states
- Each basis state plotted as a point on a unit circle at angle = phase, distance from center = magnitude
- Color coding consistent with amplitude bar chart phase colors
- `src/components/StateVisualization/QSphere.tsx` -- Three.js 3D sphere visualization
- Basis states positioned at latitude = Hamming weight (|00...0> at north pole, |11...1> at south pole)
- Point size proportional to probability, color encodes phase
- Interactive: rotate sphere with mouse drag, hover to see state label and amplitude

**Day 26: Step-through mode and gate animation**
- `src/components/CircuitBuilder/StepControls.tsx` -- Play/pause/step-forward/step-back controls for circuit execution
- Step-through: advance circuit one gate (or one column) at a time, update state visualization at each step
- Gate highlighting: current gate highlighted with glow effect on circuit canvas
- Bloch sphere animation: when stepping, animate the state rotation on Bloch sphere over 300ms
- Timeline scrubber: slider showing position in circuit, draggable to any gate
- State history: store state vector at each step for instant backward navigation

**Day 27: Circuit statistics and export**
- `src/components/CircuitBuilder/CircuitStats.tsx` -- Panel showing:
  - Gate count by type (table with icons)
  - Circuit depth
  - CNOT count (important for hardware cost)
  - Estimated execution time on various backends
  - Two-qubit gate density (ratio of two-qubit to total gates)
- Export functionality:
  - OpenQASM 3.0 file download
  - Qiskit Python script download (generated via transpilation service)
  - Circuit diagram as SVG/PNG (server-side canvas render)
  - Measurement results as CSV (state, count, probability)
- `src/api/handlers/export.rs` -- Export endpoints for QASM, Qiskit, SVG, CSV

**Day 28: Project management features**
- `src/api/handlers/projects.rs` -- Fork project (deep copy with all circuits), share via public link
- Auto-save: frontend debounced PATCH every 5 seconds on circuit changes
- Project thumbnail generation: render circuit diagram as PNG thumbnail for dashboard cards
- Circuit version history: list all versions with commit messages, click to load any version
- Circuit diff: visual comparison showing gates added (green), removed (red), modified (yellow) between two versions

**Day 29: Responsive layout and editor polish**
- Resizable panels: circuit builder (top/left) and visualization (bottom/right) with drag-to-resize splitter
- Tab system for bottom panel: Code Editor | Bloch Spheres | Amplitude Chart | Measurement Histogram
- Fullscreen mode for circuit builder (hide other panels)
- Dark theme polish: consistent color scheme across all components
- Loading states: skeleton screens for dashboard, spinner for simulation, progress bar for server jobs
- Empty states: "No gates yet -- drag a gate from the palette" with arrow indicator
- Keyboard shortcut reference: `?` key opens shortcut overlay

### Phase 6 -- Billing + Plan Enforcement (Days 30-33)

**Day 30: Stripe integration**
- `src/api/handlers/billing.rs` -- Create checkout session, create customer portal session, get subscription status
- `src/api/handlers/webhooks/stripe.rs` -- Handle subscription events: `checkout.session.completed`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.payment_failed`
- Plan mapping:
  - Free: 20-qubit max, 50 simulation runs/month, 3 projects, no noise simulation
  - Pro ($99/mo): 30-qubit max, unlimited simulations, unlimited projects, noise simulation, code generation
  - Team ($299/mo + $59/seat): 35-qubit max, 500 GPU-hours/month, collaboration, all exports, priority queue

**Day 31: Usage tracking and plan limits**
- `src/middleware/plan_limits.rs` -- Middleware checking plan limits before simulation execution
- `src/services/usage.rs` -- Track simulation runs and GPU seconds per billing period
- Usage record insertion after each simulation completes
- Usage dashboard: `src/api/handlers/usage.rs` -- Current period usage, historical usage
- Approaching-limit warnings at 80% and 100% of monthly quotas
- Automatic quota reset at billing period start

**Day 32: Billing UI**
- `frontend/src/pages/Billing.tsx` -- Plan comparison table, current plan display, usage meter
- `frontend/src/components/billing/PlanCard.tsx` -- Plan feature comparison cards with highlight on current plan
- `frontend/src/components/billing/UsageMeter.tsx` -- Simulation runs used / quota, GPU hours used / quota
- Upgrade/downgrade flow via Stripe Customer Portal
- Upgrade prompt modals when hitting plan limits (e.g., "This circuit has 25 qubits. Upgrade to Pro for up to 30 qubits.")

**Day 33: Feature gating**
- Gate noise simulation behind Pro plan
- Gate code generation (Qiskit/QASM export) behind Pro plan
- Gate circuits >20 qubits behind Pro plan
- Gate collaboration features behind Team plan
- Gate API access behind Team plan
- Show locked feature indicators with upgrade CTAs in UI
- Admin endpoint: override plan for specific users (beta testers, academic partners)

### Phase 7 -- Solver Validation + Testing (Days 34-38)

**Day 34: Solver validation -- single-qubit and basic circuits**
- Benchmark 1: Hadamard on |0> -- verify equal superposition
- Benchmark 2: Bell state preparation -- verify entanglement and measurement correlations
- Benchmark 3: Quantum Fourier Transform on 4 qubits -- verify against analytical output
- Benchmark 4: Grover's search on 3 qubits -- verify target state amplification
- Automated test suite: `solver-core/tests/benchmarks.rs` with assertions within floating-point tolerance

**Day 35: Solver validation -- algorithm correctness**
- Benchmark 5: Quantum teleportation -- verify fidelity = 1.0 for arbitrary input states
- Additional algorithm tests: Deutsch-Jozsa, Bernstein-Vazirani, Simon's algorithm
- Parameterized gate sweep: Rx(theta) for theta in [0, 2*pi], verify continuous rotation on Bloch sphere
- Mid-circuit measurement: verify state collapse and classical conditional branching
- Compare all results against Qiskit Aer state vector simulator for same circuits

**Day 36: Performance and stress testing**
- Benchmark WASM solver: time for 12, 15, 18 qubit circuits with 100 gates each
- Target: 18-qubit, 100-gate circuit completes in <200ms in WASM
- Benchmark native solver: time for 20, 25, 28, 30 qubit circuits
- Target: 25-qubit, 200-gate circuit completes in <10s on server (8-core)
- Memory usage verification: measure actual memory consumption vs. theoretical 2^n * 16 bytes
- Concurrent simulation test: submit 20 simultaneous server-side jobs, all complete correctly

**Day 37: Integration testing**
- End-to-end test: create project -> build circuit -> run simulation -> view state visualization -> export QASM
- API integration tests: auth -> project CRUD -> circuit CRUD -> simulation -> results
- WASM solver test: load in headless browser (Playwright), run simulation, verify results match native solver
- WebSocket test: connect -> subscribe -> receive progress -> receive completion
- Share link test: create share link -> access as anonymous user -> verify read-only access
- Transpilation service test: circuit_data -> QASM -> circuit_data round-trip preserves all gates

**Day 38: Cross-browser and edge case testing**
- Browser compatibility: Chrome, Firefox, Safari, Edge -- verify WASM solver and WebGL rendering
- WASM memory limits: verify graceful error when trying to simulate 19 qubits in WASM
- Empty circuit simulation: verify returns |00...0> state
- Large circuit: 18 qubits, 1000 gates -- verify WASM completes without OOM
- Network failure: server simulation in progress, network disconnects, reconnects -- verify results still available
- Concurrent edits: rapid gate additions while simulation is running -- verify no race conditions

### Phase 8 -- Deployment + Polish + Launch (Days 39-42)

**Day 39: Docker and Kubernetes configuration**
- `Dockerfile` -- Multi-stage Rust build (cargo-chef for layer caching)
- `docker-compose.yml` -- Full local dev stack (API, PostgreSQL, Redis, MinIO, transpilation service, frontend)
- `k8s/` -- Kubernetes manifests:
  - `api-deployment.yaml` -- API server (3 replicas, HPA)
  - `worker-deployment.yaml` -- Simulation workers (auto-scaling based on queue depth)
  - `transpilation-deployment.yaml` -- Python FastAPI service (2 replicas)
  - `postgres-statefulset.yaml` -- PostgreSQL with PVC
  - `redis-deployment.yaml` -- Redis for job queue and pub/sub
  - `ingress.yaml` -- NGINX ingress with TLS
- Health check endpoints: `/health/live`, `/health/ready`

**Day 40: CDN and WASM delivery**
- CloudFront distribution for:
  - Frontend static assets (HTML, JS, CSS) -- cached at edge
  - WASM solver bundle (~1.5MB) -- cached at edge with long TTL and versioned URLs
  - Circuit template thumbnails from S3 -- cached with 1-hour TTL
- WASM preloading: start downloading WASM bundle on page load before user opens circuit builder
- Service worker for offline WASM caching (progressive enhancement)
- GZIP/Brotli compression for WASM bundle delivery

**Day 41: Monitoring, logging, and polish**
- Prometheus metrics: simulation duration histogram, qubit count distribution, API latency percentiles
- Grafana dashboards: system health, simulation throughput, user activity, WASM vs. server split
- Sentry integration: frontend and backend error tracking with source maps
- Structured logging (tracing + JSON) for all API requests and simulation events
- UI polish: loading states, error messages, empty states, responsive layout
- Accessibility: keyboard navigation in circuit builder, ARIA labels on interactive elements, screen reader support for gate descriptions

**Day 42: Launch preparation and final testing**
- End-to-end smoke test in production environment
- Database backup and restore verification
- Rate limiting: 100 req/min per user, 10 simulations/min per user
- Security audit: JWT validation, SQL injection (SQLx prevents this), CORS configuration, CSP headers
- Landing page with product overview, feature screenshots, pricing table, and signup
- Documentation: getting started guide, gate reference, keyboard shortcuts, OpenQASM syntax support
- Deploy to production, enable monitoring alerts, announce launch

---

## Critical Files

```
qubitstudio/
├── solver-core/                           # Shared solver library (native + WASM)
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── statevector.rs                # State-vector simulator core
│   │   ├── measurement.rs               # Born-rule measurement, sampling
│   │   ├── compiler.rs                  # Circuit JSON -> gate sequence compiler
│   │   ├── gates/
│   │   │   ├── mod.rs                   # Gate trait definition, GateType enum
│   │   │   ├── matrices.rs             # Constant gate matrices (H, X, Y, Z, S, T)
│   │   │   ├── rotations.rs            # Parameterized rotation gates (Rx, Ry, Rz, U3)
│   │   │   └── multi_qubit.rs          # SWAP, Toffoli, controlled gates
│   │   └── optimization/
│   │       ├── mod.rs
│   │       ├── gate_fusion.rs           # Fuse consecutive single-qubit gates
│   │       └── cache.rs                # Intermediate state caching
│   └── tests/
│       ├── benchmarks.rs               # Solver validation benchmarks
│       ├── algorithms.rs               # Standard algorithm correctness tests
│       └── measurement.rs             # Measurement statistics tests
│
├── solver-wasm/                          # WASM compilation target
│   ├── Cargo.toml
│   └── src/
│       └── lib.rs                       # WASM entry points (wasm-bindgen)
│
├── qubitstudio-api/                      # Rust API server
│   ├── Cargo.toml
│   ├── src/
│   │   ├── main.rs                      # Axum app setup, router, startup
│   │   ├── config.rs                    # Environment config
│   │   ├── state.rs                     # AppState (PgPool, Redis, S3)
│   │   ├── error.rs                     # ApiError types
│   │   ├── auth/
│   │   │   ├── mod.rs                   # JWT middleware, Claims
│   │   │   └── oauth.rs                # Google/GitHub OAuth handlers
│   │   ├── api/
│   │   │   ├── router.rs               # Route definitions
│   │   │   ├── handlers/
│   │   │   │   ├── auth.rs             # Register, login, OAuth
│   │   │   │   ├── users.rs            # Profile CRUD
│   │   │   │   ├── projects.rs         # Project CRUD, fork, share
│   │   │   │   ├── circuits.rs         # Circuit version CRUD, diff
│   │   │   │   ├── simulation.rs       # Create/get/cancel simulation
│   │   │   │   ├── experiments.rs      # Experiment listing, comparison
│   │   │   │   ├── templates.rs        # Circuit template listing
│   │   │   │   ├── share.rs            # Share link management
│   │   │   │   ├── export.rs           # QASM/Qiskit/SVG/CSV export
│   │   │   │   ├── billing.rs          # Stripe checkout/portal
│   │   │   │   ├── usage.rs            # Usage tracking
│   │   │   │   └── webhooks/
│   │   │   │       └── stripe.rs       # Stripe webhook handler
│   │   │   └── ws/
│   │   │       ├── mod.rs              # WebSocket upgrade handler
│   │   │       └── simulation_progress.rs # Live progress streaming
│   │   ├── middleware/
│   │   │   └── plan_limits.rs          # Plan enforcement middleware
│   │   ├── services/
│   │   │   ├── usage.rs                # Usage tracking service
│   │   │   └── s3.rs                   # S3 client helpers
│   │   └── workers/
│   │       ├── mod.rs                  # Worker pool management
│   │       └── simulation_worker.rs    # Server-side simulation execution
│   ├── migrations/
│   │   └── 001_initial.sql             # Full database schema
│   └── tests/
│       ├── api_integration.rs          # API integration tests
│       └── simulation_e2e.rs           # End-to-end simulation tests
│
├── transpilation-service/                # Python FastAPI service
│   ├── requirements.txt                 # qiskit, fastapi, uvicorn, pydantic
│   ├── main.py                          # FastAPI app
│   ├── parser.py                        # OpenQASM 3.0 <-> circuit_data conversion
│   ├── generator.py                     # Code generation (QASM, Qiskit)
│   ├── transpiler.py                    # Qiskit transpilation passes
│   └── Dockerfile
│
├── frontend/                             # React frontend
│   ├── package.json
│   ├── vite.config.ts
│   ├── src/
│   │   ├── App.tsx                      # Router, providers, layout
│   │   ├── main.tsx                     # Entry point
│   │   ├── stores/
│   │   │   ├── authStore.ts             # Auth state (JWT, user)
│   │   │   ├── projectStore.ts          # Project state
│   │   │   ├── circuitStore.ts          # Circuit builder state (gates, qubits, selection)
│   │   │   └── simulationStore.ts       # Simulation results state
│   │   ├── hooks/
│   │   │   ├── useWasmSolver.ts         # WASM solver hook (load + run)
│   │   │   ├── useServerSimulation.ts   # Server-side simulation hook
│   │   │   └── useSimulationProgress.ts # WebSocket hook for live progress
│   │   ├── lib/
│   │   │   ├── api.ts                   # Axios API client
│   │   │   ├── wasmLoader.ts            # WASM bundle loading
│   │   │   └── qasmLanguage.ts          # Monaco language definition for OpenQASM
│   │   ├── pages/
│   │   │   ├── Dashboard.tsx            # Project list
│   │   │   ├── Editor.tsx               # Main editor (circuit + viz split)
│   │   │   ├── Templates.tsx            # Circuit template gallery
│   │   │   ├── Experiments.tsx          # Experiment history and comparison
│   │   │   ├── Billing.tsx              # Plan management
│   │   │   ├── Login.tsx                # Auth pages
│   │   │   └── Register.tsx
│   │   ├── components/
│   │   │   ├── CircuitBuilder/
│   │   │   │   ├── CircuitCanvas.tsx    # Canvas2D circuit renderer
│   │   │   │   ├── QubitWire.tsx        # Qubit wire rendering
│   │   │   │   ├── GatePalette.tsx      # Gate library sidebar
│   │   │   │   ├── GateSymbol.tsx       # Individual gate renderer
│   │   │   │   ├── ConnectionLine.tsx   # Multi-qubit gate connectors
│   │   │   │   ├── ParameterPanel.tsx   # Gate parameter sliders
│   │   │   │   ├── CircuitToolbar.tsx   # Circuit editing toolbar
│   │   │   │   ├── CircuitStats.tsx     # Gate count, depth, statistics
│   │   │   │   └── StepControls.tsx     # Step-through playback controls
│   │   │   ├── StateVisualization/
│   │   │   │   ├── BlochSphere.tsx      # Three.js Bloch sphere
│   │   │   │   ├── AmplitudeBarChart.tsx # WebGL amplitude bars
│   │   │   │   ├── MeasurementHistogram.tsx # D3 measurement bar chart
│   │   │   │   ├── PhaseWheel.tsx       # Phase angle circular plot
│   │   │   │   └── QSphere.tsx          # Three.js Q-sphere
│   │   │   ├── CodeEditor/
│   │   │   │   └── QASMEditor.tsx       # Monaco editor for OpenQASM
│   │   │   ├── Experiments/
│   │   │   │   ├── ExperimentCard.tsx   # Experiment summary card
│   │   │   │   └── ComparisonView.tsx   # Side-by-side result comparison
│   │   │   ├── billing/
│   │   │   │   ├── PlanCard.tsx
│   │   │   │   └── UsageMeter.tsx
│   │   │   └── common/
│   │   │       ├── SimulationToolbar.tsx # Run/stop/shots selector
│   │   │       └── ShareDialog.tsx      # Share link generation
│   │   └── data/
│   │       └── gateDefinitions.ts       # Gate symbol definitions, colors, matrix info
│   └── public/
│       └── wasm/                        # WASM solver bundle (loaded at runtime)
│
├── k8s/                                  # Kubernetes manifests
│   ├── api-deployment.yaml
│   ├── worker-deployment.yaml
│   ├── transpilation-deployment.yaml
│   ├── postgres-statefulset.yaml
│   ├── redis-deployment.yaml
│   └── ingress.yaml
│
├── docker-compose.yml                    # Local development stack
├── Cargo.toml                            # Workspace root
└── .github/
    └── workflows/
        ├── ci.yml                        # Test + lint on PR
        ├── wasm-build.yml                # Build + deploy WASM bundle
        └── deploy.yml                    # Build Docker images + deploy to K8s
```

---

## Solver Validation Suite

### Benchmark 1: Hadamard on |0> (Single-Qubit Superposition)

**Circuit:** 1 qubit, H gate on q0

**Expected state:** |+> = (1/sqrt(2))|0> + (1/sqrt(2))|1>

**Expected probabilities:** P(|0>) = **0.5000**, P(|1>) = **0.5000**

**Expected amplitudes:** alpha_0 = **0.70711 + 0i**, alpha_1 = **0.70711 + 0i**

**Tolerance:** < 1e-10 (machine precision, single matrix multiply)

### Benchmark 2: Bell State Preparation (Entanglement)

**Circuit:** 2 qubits, H on q0 then CNOT(q0, q1)

**Expected state:** |Phi+> = (1/sqrt(2))|00> + (1/sqrt(2))|11>

**Expected probabilities:** P(|00>) = **0.5000**, P(|01>) = **0.0000**, P(|10>) = **0.0000**, P(|11>) = **0.5000**

**Measurement correlation:** Over 10,000 shots, both qubits always agree: P(q0 = q1) = **1.0000**

**Tolerance:** Amplitudes < 1e-10, measurement correlation: exact (deterministic after collapse)

### Benchmark 3: Quantum Fourier Transform on |0101> (4 Qubits)

**Circuit:** 4 qubits initialized to |0101> (= |5> in decimal), QFT circuit (H, controlled-R_k rotations, SWAP)

**Expected output state:** |psi> = (1/4) * sum_{k=0}^{15} exp(2*pi*i*5*k/16) |k>

**Expected amplitude for |0000>:** (1/4) * exp(0) = **0.2500 + 0.0000i**

**Expected amplitude for |0001>:** (1/4) * exp(2*pi*i*5/16) = **0.0245 + 0.2488i**

**Expected amplitude for |0100>:** (1/4) * exp(2*pi*i*20/16) = (1/4) * exp(2*pi*i*1.25) = **0.0000 + 0.2500i**

**All amplitudes have equal magnitude:** |alpha_k| = **0.2500** for all k

**Tolerance:** < 0.1% on magnitudes, < 0.01 radians on phases

### Benchmark 4: Grover's Search on 3 Qubits (Target |101>)

**Circuit:** 3 qubits, Grover's algorithm with oracle marking |101>, optimal 1 iteration (floor(pi/4 * sqrt(8/1)) ~= 2 iterations for single target in 8 states)

**Expected probability of target |101> after 2 iterations:** P(|101>) = sin^2(5 * arcsin(1/sqrt(8))) = **0.9453**

**Expected probability of each non-target state:** (1 - 0.9453) / 7 = **0.0078** each

**Shot-based verification:** Over 10,000 shots, |101> count = **~9453** (within 3-sigma: 9453 +/- 210)

**Tolerance:** State probability < 1%, shot count within 3-sigma of binomial distribution

### Benchmark 5: Quantum Teleportation (Arbitrary State Transfer)

**Circuit:** 3 qubits. Prepare q0 in state Ry(1.2)|0> = cos(0.6)|0> + sin(0.6)|1>. Bell pair on q1,q2. CNOT(q0,q1), H on q0, measure q0 and q1, classically-controlled X and Z on q2.

**Expected final state of q2:** cos(0.6)|0> + sin(0.6)|1> = **0.8253|0> + 0.5646|1>**

**Expected fidelity between input state and teleported state:** F = **1.0000** (exact for ideal simulation)

**Verification:** Reduced density matrix of q2 after tracing out q0, q1 matches input state density matrix within tolerance

**Tolerance:** Fidelity > 0.9999 (limited only by floating-point precision)

---

## Verification

### Integration Testing Checklist

1. **Auth flow** -- Register -> login -> JWT issued -> authorized API calls -> token refresh -> logout
2. **Project CRUD** -- Create -> update settings -> save -> reload -> verify state preserved
3. **Circuit editing** -- Add qubits -> place gates -> set parameters -> save version -> reload -> verify circuit preserved
4. **WASM simulation** -- 5-qubit Bell pair -> WASM solver -> state vector returned -> Bloch spheres display
5. **Server simulation** -- 25-qubit circuit -> job queued -> worker picks up -> WebSocket progress -> results in S3
6. **State visualization** -- Run simulation -> amplitude bar chart displays -> Bloch spheres show correct states -> measurement histogram shows correct distribution
7. **Code editor sync** -- Build circuit visually -> QASM code updates -> edit QASM code -> circuit updates
8. **Circuit templates** -- Select "GHZ State" template -> project created -> simulate -> verify correct entanglement
9. **Experiment tracking** -- Run 3 simulations with different parameters -> all logged -> compare results side-by-side
10. **Plan limits** -- Free user -> circuit with 25 qubits -> blocked with upgrade prompt
11. **Billing** -- Subscribe to Pro -> Stripe checkout -> webhook -> plan updated -> qubit limit raised to 30
12. **Share links** -- Create share link -> access as anonymous user -> view circuit and results -> cannot edit
13. **Step-through** -- Build 5-gate circuit -> step through gate by gate -> Bloch sphere animates at each step
14. **Export** -- Design circuit -> export QASM -> import in Qiskit -> results match QubitStudio simulation
15. **Error handling** -- Invalid circuit (gate on nonexistent qubit) -> meaningful error -> no crash

### SQL Verification Queries

```sql
-- 1. Simulation throughput and success rate by execution mode
SELECT
    execution_mode,
    status,
    COUNT(*) as count,
    AVG(compute_time_ms)::int as avg_time_ms,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY compute_time_ms)::int as p95_time_ms
FROM experiments
WHERE created_at >= NOW() - INTERVAL '7 days'
GROUP BY execution_mode, status;

-- 2. Qubit count distribution across experiments
SELECT
    c.qubit_count,
    COUNT(*) as experiment_count,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 1) as pct,
    AVG(e.compute_time_ms)::int as avg_compute_ms
FROM experiments e
JOIN circuits c ON e.circuit_id = c.id
WHERE e.created_at >= NOW() - INTERVAL '30 days'
    AND e.status = 'completed'
GROUP BY c.qubit_count
ORDER BY c.qubit_count;

-- 3. User plan distribution and conversion
SELECT plan, COUNT(*) as users,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 1) as pct
FROM users GROUP BY plan ORDER BY users DESC;

-- 4. Template popularity (which algorithms are most used)
SELECT ct.name, ct.category, ct.qubit_count,
    COUNT(DISTINCT p.id) as projects_created,
    COUNT(DISTINCT e.id) as experiments_run
FROM circuit_templates ct
LEFT JOIN projects p ON p.forked_from IS NOT NULL
    AND EXISTS (
        SELECT 1 FROM circuits c
        WHERE c.project_id = p.id
        AND c.circuit_data @> ct.circuit_data
    )
LEFT JOIN experiments e ON e.project_id = p.id
GROUP BY ct.id, ct.name, ct.category, ct.qubit_count
ORDER BY projects_created DESC;

-- 5. GPU usage by user for billing period
SELECT u.email, u.plan,
    SUM(ur.quantity) as total_gpu_seconds,
    CASE u.plan
        WHEN 'free' THEN 0
        WHEN 'pro' THEN 36000    -- 10 GPU-hours
        WHEN 'team' THEN 1800000  -- 500 GPU-hours
    END as limit_seconds,
    COUNT(DISTINCT e.id) as experiment_count
FROM usage_records ur
JOIN users u ON ur.user_id = u.id
LEFT JOIN experiments e ON e.user_id = u.id
    AND e.created_at >= ur.period_start
    AND e.created_at <= ur.period_end
WHERE ur.period_start <= CURRENT_DATE AND ur.period_end >= CURRENT_DATE
    AND ur.record_type = 'gpu_seconds'
GROUP BY u.email, u.plan
ORDER BY total_gpu_seconds DESC;
```

### Performance Benchmarks

| Operation | Target | Method |
|-----------|--------|--------|
| WASM solver: 10-qubit, 50-gate circuit | <50ms | Browser benchmark (Chrome DevTools) |
| WASM solver: 15-qubit, 100-gate circuit | <100ms | Browser benchmark |
| WASM solver: 18-qubit, 200-gate circuit | <200ms | Browser benchmark |
| Server solver: 20-qubit, 200-gate circuit | <2s | Server timing, 4 cores |
| Server solver: 25-qubit, 500-gate circuit | <10s | Server timing, 8 cores |
| Server solver: 30-qubit, 100-gate circuit | <60s | Server timing, 16GB RAM |
| Measurement sampling: 1M shots, 15 qubits | <500ms | Server timing |
| Bloch sphere render: 8 qubits simultaneously | 60 FPS | Chrome FPS counter |
| Amplitude bar chart: 2^18 bars (262K) | 30 FPS | WebGL rendering during pan/zoom |
| Measurement histogram: 1000 bars | 60 FPS | D3 rendering |
| Circuit canvas: 20 qubits, 200 gates | 60 FPS | Canvas2D rendering during pan/zoom |
| API: create experiment | <200ms | p95 latency (Prometheus) |
| API: list experiments | <100ms | p95 latency |
| WASM bundle load (cached) | <300ms | Service worker cache hit |
| WASM bundle load (cold) | <2s | CDN delivery, ~1.5MB gzipped |
| WebSocket: progress latency | <100ms | Time from worker emit to client receive |
| QASM parse (transpilation service) | <500ms | 200-gate circuit parse + validate |
| QASM round-trip (circuit->QASM->circuit) | <1s | Full bidirectional sync cycle |

---

## Deployment Architecture

### Infrastructure

```
┌─────────────────────────────────────────────────┐
│                  CloudFront CDN                   │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────┐ │
│  │ Frontend SPA │  │ WASM Bundle │  │ Template │ │
│  │ (HTML/JS/CSS)│  │ (versioned) │  │ Thumbs   │ │
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
│ Pod x3 (HPA) │  │              │  │              │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘
       │                 │                 │
       └─────────────────┼─────────────────┘
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ PostgreSQL   │ │ Redis        │ │ S3 Bucket    │
│ (RDS r6g)    │ │ (ElastiCache)│ │ (Results +   │
│ Multi-AZ     │ │ Cluster mode │ │  Exports)    │
└──────────────┘ └──────┬───────┘ └──────────────┘
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ Sim Worker   │ │ Sim Worker   │ │ Transpilation│
│ (Rust native)│ │ (Rust native)│ │ Service      │
│ r6g.2xlarge  │ │ r6g.2xlarge  │ │ (Python/     │
│ HPA: 2-10    │ │ (32GB RAM)   │ │  FastAPI)    │
└──────────────┘ └──────────────┘ └──────────────┘
```

### Scaling Strategy

- **API servers**: Horizontal pod autoscaler (HPA) at 70% CPU, min 3, max 10
- **Simulation workers**: HPA based on Redis queue depth -- scale up when queue > 5 jobs, scale down when idle 5 min
- **Worker instances**: AWS r6g.2xlarge (8 vCPU, 64GB RAM) for memory-intensive 30-qubit simulations (16GB state vector)
- **Transpilation service**: 2-4 replicas, stateless, scales with API traffic
- **Database**: RDS PostgreSQL r6g.xlarge with read replica for experiment listing queries
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
      "ID": "circuit-exports-lifecycle",
      "Prefix": "exports/",
      "Status": "Enabled",
      "Expiration": { "Days": 30 }
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
| Density Matrix Noise Simulation | Density matrix engine supporting depolarizing, amplitude damping, phase damping, readout error channels; hardware-specific noise models from IBM/IonQ calibration data; fidelity comparison between ideal and noisy execution | High |
| IBM Quantum Hardware Access | Direct integration with IBM Quantum via Qiskit Runtime; transpile circuits to IBM native gate set (CX, ID, RZ, SX); submit jobs, track queue position, retrieve results; side-by-side comparison of simulator vs. hardware results | High |
| VQE/QAOA Optimization Studio | Variational algorithm workflow: define Hamiltonian (Pauli string or molecular geometry), choose ansatz, select classical optimizer (COBYLA, SPSA, Adam), run optimization loop with live convergence dashboard | High |
| Multi-Provider Hardware Access | IonQ (GPI, GPI2, MS gates), Rigetti (CZ, RX, RZ), Amazon Braket unified interface; automatic circuit transpilation to each native gate set; cross-provider result comparison and cost analysis | Medium |
| Real-Time Collaboration | CRDT-based circuit editing with cursor presence, per-gate commenting, shared experiment results, team design review workflows | Medium |
| Quantum Error Correction Studio | Surface code visualizer, Steane [[7,1,3]] code, repetition code templates; syndrome measurement visualization; MWPM decoder; logical error rate estimation via Monte Carlo | Medium |
| Educational Mode with Tutorials | Guided step-by-step lessons on quantum gates, superposition, entanglement; interactive quizzes; gate-by-gate animation; curriculum from beginner to advanced with progress tracking and badges | Low |
| Tensor Network Simulator | Alternative simulation backend using MPS/MPO tensor networks for structured circuits; enables simulation of 50+ qubits for circuits with limited entanglement (low Schmidt rank); automatic backend selection based on circuit structure | Low |
