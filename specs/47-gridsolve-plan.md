# 47. GridSolve — Power Systems Analysis and Grid Simulation Platform

## Implementation Plan

**MVP Scope:** Browser-based single-line diagram editor with drag-and-drop equipment placement (generators, transformers, transmission lines, cables, loads, buses) and smart bus connectivity rendered via SVG with R-tree spatial indexing, custom power flow engine implementing Newton-Raphson and fast-decoupled methods with sparse matrix solver compiled to WebAssembly for client-side execution of systems ≤100 buses and server-side Rust-native execution for larger systems, support for balanced and unbalanced power flow analysis with automatic slack bus selection and PV/PQ bus handling, short-circuit analysis with IEC 60909 fault calculation methods (three-phase, single-line-to-ground, line-to-line faults) and sequence network visualization, basic protection coordination with time-current curve (TCC) plotting for generic overcurrent relays and fuses, interactive results viewer with voltage profile plots and equipment loading tables rendered via Canvas/WebGL, 500+ equipment models from major manufacturers (ABB, Siemens, GE, Schneider) stored in S3 with PostgreSQL metadata, CIM (Common Information Model) import/export compatible with utility grid models, Stripe billing with three tiers (Free / Professional $149/mo / Utility $399/user/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| Power Flow Solver | Rust (native + WASM) | Custom Newton-Raphson and fast-decoupled engine with KLU sparse solver via `suitesparse-sys` |
| WASM Compilation | `wasm-pack` + `wasm-bindgen` | Client-side solver for systems ≤100 buses |
| Report Generation | Python 3.12 (FastAPI) | IEEE/NERC-compliant report generation, PDF rendering with matplotlib |
| Database | PostgreSQL 16 | Projects, users, studies, equipment model metadata |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | Equipment models, study results, one-line diagrams, reports |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| One-Line Editor | Custom SVG renderer | R-tree spatial indexing, snap-to-bus, auto-routing |
| Results Viewer | Canvas + WebGL | GPU-accelerated rendering for large voltage/loading visualizations |
| Real-time | WebSocket (Axum) | Live study progress, convergence monitoring |
| Job Queue | Redis 7 + Tokio tasks | Server-side power flow job management, contingency analysis distribution |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | WASM bundle delivery, static frontend assets |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, error tracking |
| CI/CD | GitHub Actions | WASM build pipeline, Rust tests, Docker image push |

### Key Architecture Decisions

1. **Hybrid client/server solver with WASM threshold at 100 buses**: Systems with ≤100 buses (covers 80%+ of distribution feeders, industrial plant studies, and academic work) run entirely in the browser via WASM, providing instant feedback with zero server cost. Systems exceeding 100 buses are submitted to the Rust-native server solver which handles transmission networks up to 100,000+ buses and N-1/N-2 contingency analysis. The threshold is configurable per plan tier.

2. **Custom power flow engine in Rust rather than wrapping OpenDSS**: Building a custom Newton-Raphson solver in Rust gives us full control over convergence algorithms, WASM compilation, and parallelization for contingency screening. OpenDSS is script-based with COM interfaces that are fragile and Windows-specific. Our Rust solver uses `suitesparse-sys` bindings for KLU sparse factorization (the same algorithm used by commercial tools like PSS/E) while maintaining memory safety and WASM compatibility.

3. **SVG single-line editor with R-tree spatial indexing**: SVG gives crisp rendering at any zoom level and straightforward DOM-based hit testing. An R-tree (via `rbush` JS library) enables O(log n) spatial queries for bus snapping, branch routing, and equipment selection even with 1,000+ elements. Canvas/WebGL alternatives were rejected because they require reimplementing text layout, cursor interaction, and accessibility for relay labels and bus names.

4. **Canvas/WebGL results viewer separate from one-line**: The results viewer renders voltage profiles, equipment loading bars, and TCC curves with GPU-accelerated rendering. This is decoupled from the SVG one-line to allow independent pan/zoom and multi-pane layouts. Data is streamed from WASM/server as Float32Arrays and uploaded to GPU buffers for fast rendering.

5. **S3 for equipment model storage with PostgreSQL metadata catalog**: Equipment models (transformer impedances, cable data, relay curves, typically 1-20KB JSON each) are stored in S3, while PostgreSQL holds searchable metadata (manufacturer, voltage class, MVA rating, category). This allows the equipment library to scale to 100K+ models without bloating the database while enabling fast parametric search via PostgreSQL indexes and full-text search.

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";  -- For fuzzy text search on equipment models

-- Users
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email TEXT UNIQUE NOT NULL,
    password_hash TEXT,  -- NULL for OAuth-only users
    name TEXT NOT NULL,
    avatar_url TEXT,
    auth_provider TEXT DEFAULT 'email',  -- email | google | github
    auth_provider_id TEXT,
    plan TEXT NOT NULL DEFAULT 'free',  -- free | professional | utility
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX users_email_idx ON users(email);
CREATE INDEX users_stripe_idx ON users(stripe_customer_id);

-- Organizations (for Utility plan)
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

-- Projects (single-line diagram + study workspace)
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    diagram_data JSONB NOT NULL DEFAULT '{}',  -- Full one-line state (buses, branches, equipment, coordinates)
    base_mva REAL NOT NULL DEFAULT 100.0,
    base_kv REAL NOT NULL DEFAULT 138.0,
    settings JSONB DEFAULT '{}',  -- Grid settings, display options
    is_public BOOLEAN NOT NULL DEFAULT false,
    forked_from UUID REFERENCES projects(id) ON DELETE SET NULL,
    thumbnail_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);
CREATE INDEX projects_org_idx ON projects(org_id);
CREATE INDEX projects_updated_idx ON projects(updated_at DESC);

-- Power Flow Studies
CREATE TABLE studies (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    study_type TEXT NOT NULL,  -- power_flow | short_circuit | protection | transient
    analysis_method TEXT NOT NULL,  -- newton_raphson | fast_decoupled | gauss_seidel | iec_60909
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | running | completed | failed | cancelled
    execution_mode TEXT NOT NULL DEFAULT 'wasm',  -- wasm | server
    input_data JSONB NOT NULL DEFAULT '{}',  -- Bus data, branch data, load data, generation data
    parameters JSONB NOT NULL DEFAULT '{}',  -- Analysis-specific parameters
    bus_count INTEGER NOT NULL DEFAULT 0,
    results_url TEXT,  -- S3 URL for result data
    results_summary JSONB,  -- Quick-access summary (voltage violations, loading violations, convergence stats)
    error_message TEXT,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    wall_time_ms INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX studies_project_idx ON studies(project_id);
CREATE INDEX studies_user_idx ON studies(user_id);
CREATE INDEX studies_status_idx ON studies(status);
CREATE INDEX studies_created_idx ON studies(created_at DESC);

-- Server Study Jobs (for server-side execution only)
CREATE TABLE study_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    study_id UUID NOT NULL REFERENCES studies(id) ON DELETE CASCADE,
    worker_id TEXT,
    cores_allocated INTEGER DEFAULT 4,
    memory_mb INTEGER DEFAULT 8192,
    priority INTEGER DEFAULT 0,  -- Higher = more urgent
    progress_pct REAL DEFAULT 0.0,
    convergence_data JSONB DEFAULT '[]',  -- [{iteration, mismatch, buses_out_of_tolerance}]
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX jobs_study_idx ON study_jobs(study_id);
CREATE INDEX jobs_worker_idx ON study_jobs(worker_id);

-- Equipment Models (metadata; actual model files in S3)
CREATE TABLE equipment_models (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,  -- e.g., "ABB DT 138kV 50MVA" or "Siemens SEL-351"
    manufacturer TEXT,  -- e.g., "ABB", "Siemens", "GE"
    model_number TEXT,  -- Manufacturer model number
    category TEXT NOT NULL,  -- generator | transformer | transmission_line | cable | load | capacitor_bank | reactor | breaker | relay | bus
    subcategory TEXT,  -- e.g., "synchronous" for generator, "distribution" for transformer
    voltage_class_kv REAL,  -- Nominal voltage class
    rating_mva REAL,  -- MVA rating for transformers/generators
    model_data_url TEXT NOT NULL,  -- S3 URL to JSON model file
    parameters JSONB DEFAULT '{}',  -- Key electrical parameters for search (impedance, X/R ratio, etc.)
    symbol_data JSONB,  -- SVG symbol definition for one-line rendering
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
CREATE INDEX models_category_idx ON equipment_models(category);
CREATE INDEX models_manufacturer_idx ON equipment_models(manufacturer);
CREATE INDEX models_name_trgm_idx ON equipment_models USING gin(name gin_trgm_ops);
CREATE INDEX models_model_trgm_idx ON equipment_models USING gin(model_number gin_trgm_ops);
CREATE INDEX models_tags_idx ON equipment_models USING gin(tags);
CREATE INDEX models_voltage_idx ON equipment_models(voltage_class_kv);

-- Contingency Scenarios (for N-1, N-2 analysis)
CREATE TABLE contingencies (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    study_id UUID NOT NULL REFERENCES studies(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    outage_elements JSONB NOT NULL,  -- [{element_id, element_type}]
    results_url TEXT,  -- S3 URL for contingency result data
    severity_score REAL,  -- 0-100 score based on voltage violations and loading
    violations_summary JSONB,  -- {voltage_violations: 5, loading_violations: 2, islands: 0}
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX contingencies_study_idx ON contingencies(study_id);
CREATE INDEX contingencies_severity_idx ON contingencies(severity_score DESC);

-- Protection Coordination Data
CREATE TABLE protection_configs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    study_id UUID NOT NULL REFERENCES studies(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    relay_settings JSONB NOT NULL,  -- [{relay_id, pickup, time_dial, curve_type}]
    tcc_data JSONB,  -- Time-current curve plot data
    coordination_report_url TEXT,  -- S3 URL to PDF coordination report
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX protection_study_idx ON protection_configs(study_id);

-- Usage Tracking (for billing enforcement)
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,  -- study_minutes | server_study | storage_bytes | contingency_runs
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
    pub diagram_data: serde_json::Value,
    pub base_mva: f32,
    pub base_kv: f32,
    pub settings: serde_json::Value,
    pub is_public: bool,
    pub forked_from: Option<Uuid>,
    pub thumbnail_url: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Study {
    pub id: Uuid,
    pub project_id: Uuid,
    pub user_id: Uuid,
    pub study_type: String,
    pub analysis_method: String,
    pub status: String,
    pub execution_mode: String,
    pub input_data: serde_json::Value,
    pub parameters: serde_json::Value,
    pub bus_count: i32,
    pub results_url: Option<String>,
    pub results_summary: Option<serde_json::Value>,
    pub error_message: Option<String>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub wall_time_ms: Option<i32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct StudyJob {
    pub id: Uuid,
    pub study_id: Uuid,
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
pub struct EquipmentModel {
    pub id: Uuid,
    pub name: String,
    pub manufacturer: Option<String>,
    pub model_number: Option<String>,
    pub category: String,
    pub subcategory: Option<String>,
    pub voltage_class_kv: Option<f32>,
    pub rating_mva: Option<f32>,
    pub model_data_url: String,
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
pub struct PowerFlowParams {
    pub method: PowerFlowMethod,
    pub max_iterations: u32,  // Default 20
    pub tolerance: f64,  // Default 1e-6 (pu)
    pub acceleration_factor: f64,  // For Gauss-Seidel, default 1.6
    pub flat_start: bool,  // Default true (all buses at 1.0 pu, 0 deg)
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum PowerFlowMethod {
    NewtonRaphson,
    FastDecoupled,
    GaussSeidel,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum BusType {
    Slack,     // Voltage and angle specified (reference bus)
    PV,        // Real power and voltage magnitude specified (generator bus)
    PQ,        // Real and reactive power specified (load bus)
}

#[derive(Debug, Deserialize, Serialize)]
pub struct BusData {
    pub bus_id: String,
    pub bus_type: BusType,
    pub base_kv: f64,
    pub voltage_pu: f64,  // Per-unit voltage magnitude (for PV and slack)
    pub angle_deg: f64,   // Voltage angle in degrees (for slack)
    pub p_gen_mw: f64,    // Generator real power (for PV and slack)
    pub q_gen_mvar: f64,  // Generator reactive power (computed for PV)
    pub p_load_mw: f64,   // Load real power
    pub q_load_mvar: f64, // Load reactive power
    pub shunt_g_pu: f64,  // Shunt conductance in pu
    pub shunt_b_pu: f64,  // Shunt susceptance in pu
}

#[derive(Debug, Deserialize, Serialize)]
pub struct BranchData {
    pub branch_id: String,
    pub from_bus: String,
    pub to_bus: String,
    pub r_pu: f64,         // Resistance in pu
    pub x_pu: f64,         // Reactance in pu
    pub b_pu: f64,         // Line charging susceptance in pu
    pub tap_ratio: f64,    // Transformer tap ratio (1.0 for lines)
    pub phase_shift_deg: f64,  // Transformer phase shift (0 for lines)
    pub rating_mva: Option<f64>,  // Thermal rating
    pub in_service: bool,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct ShortCircuitParams {
    pub fault_type: FaultType,
    pub fault_bus: String,
    pub fault_impedance_ohms: Option<f64>,  // Default 0 (solid fault)
    pub prefault_voltage_pu: f64,  // Default 1.0
    pub method: String,  // "iec_60909" | "ansi_c37"
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum FaultType {
    ThreePhase,
    SingleLineToGround,
    LineToLine,
    DoubleLineToGround,
}
```

---

## Solver Architecture Deep-Dive

### Governing Equations and Discretization

GridSolve's core solver implements **Newton-Raphson power flow**, the standard formulation used by all commercial power system analysis tools (PSS/E, PowerWorld, ETAP). For a system with `n` buses, the power flow equations are:

```
For each bus i:
  P_i = V_i Σ V_j (G_ij cos θ_ij + B_ij sin θ_ij)
  Q_i = V_i Σ V_j (G_ij sin θ_ij - B_ij cos θ_ij)

where:
  P_i = real power injection (generation - load)
  Q_i = reactive power injection (generation - load)
  V_i = voltage magnitude at bus i
  θ_ij = θ_i - θ_j (voltage angle difference)
  Y_ij = G_ij + jB_ij (admittance matrix element)
```

**Newton-Raphson iteration** solves this nonlinear system by linearizing around the current solution:

```
┌         ┐ ┌      ┐   ┌      ┐
│ J1  J2  │ │ Δθ   │   │ ΔP   │
│         │ │      │ = │      │
│ J3  J4  │ │ ΔV   │   │ ΔQ   │
└         ┘ └      ┘   └      ┘

where:
  J1 = ∂P/∂θ  (n×n matrix)
  J2 = ∂P/∂V  (n×n matrix)
  J3 = ∂Q/∂θ  (n×n matrix)
  J4 = ∂Q/∂V  (n×n matrix)
  ΔP = P_specified - P_calculated
  ΔQ = Q_specified - Q_calculated
```

The Jacobian is sparse (typically 95%+ zeros for transmission grids) and is solved using **KLU sparse LU decomposition** for efficiency.

**Fast-decoupled power flow** exploits the weak coupling between real power/voltage angle and reactive power/voltage magnitude:

```
Decoupled:
  ΔP ≈ B' Δθ    (real power depends primarily on angle)
  ΔQ ≈ B'' ΔV   (reactive power depends primarily on voltage)

where B' and B'' are constant approximations of the Jacobian.
```

This allows factoring the matrices once and reusing them for all iterations, achieving 5-10× speedup for large systems with acceptable accuracy loss (<0.1%).

**Unbalanced three-phase power flow** (for distribution systems with asymmetric loads and single-phase lines) uses the full 3×3 impedance matrix:

```
┌      ┐   ┌           ┐ ┌      ┐
│ I_a  │   │ Y_aa Y_ab Y_ac │ │ V_a  │
│ I_b  │ = │ Y_ba Y_bb Y_bc │ │ V_b  │
│ I_c  │   │ Y_ca Y_cb Y_cc │ │ V_c  │
└      ┘   └           ┘ └      ┘

Solved via Newton-Raphson with 3n equations for n buses.
```

**Short-circuit analysis** uses the **IEC 60909** method for fault current calculation:

```
I_fault = c · V_prefault / Z_eq

where:
  c = voltage factor (1.1 for max fault, 1.0 for min fault)
  V_prefault = voltage at fault location before fault
  Z_eq = equivalent impedance to fault point

For three-phase fault:
  Z_eq = Z_1 (positive sequence impedance)

For SLG fault:
  Z_eq = Z_1 + Z_2 + Z_0 + 3Z_f
  (Z_1 = positive, Z_2 = negative, Z_0 = zero sequence, Z_f = fault impedance)
```

Sequence networks (positive, negative, zero) are built from equipment impedance data and network topology. Fault currents are distributed using current division through impedance branches.

### Client/Server Split (WASM Threshold)

```
Power system uploaded → Bus count extracted
    │
    ├── ≤100 buses → WASM solver (browser)
    │   ├── Instant startup, no network latency
    │   ├── Results displayed immediately
    │   └── No server cost
    │
    └── >100 buses → Server solver (Rust native)
        ├── Job queued via Redis
        ├── Worker picks up, runs native solver
        ├── Progress streamed via WebSocket
        └── Results stored in S3, URL returned
```

The 100-bus threshold was chosen because:
- WASM solver with KLU handles 100-bus Newton-Raphson power flow in <1 second on modern hardware
- 100 buses covers: distribution feeders (20-80 buses), industrial plants (30-100 buses), small transmission systems (50-100 buses)
- Above 100 buses: regional transmission grids, utility-scale contingency analysis, large distribution systems → need server compute

### WASM Compilation Pipeline

```toml
# solver-wasm/Cargo.toml
[package]
name = "gridsolve-solver-wasm"
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
num-complex = "0.4"
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
      - run: wasm-opt -Oz solver-wasm/pkg/gridsolve_solver_wasm_bg.wasm -o solver-wasm/pkg/gridsolve_solver_wasm_bg.wasm
      - name: Upload to S3/CDN
        run: aws s3 cp solver-wasm/pkg/ s3://gridsolve-wasm/v${{ github.sha }}/ --recursive
```

---

## Architecture Deep-Dives

### 1. Power Flow API Handler (Rust/Axum)

The primary endpoint receives a power flow request, validates the system, decides between WASM and server execution, and for server execution enqueues a job with Redis and streams progress via WebSocket.

```rust
// src/api/handlers/study.rs

use axum::{
    extract::{Path, State, Json},
    http::StatusCode,
    response::IntoResponse,
};
use uuid::Uuid;
use crate::{
    db::models::{Study, PowerFlowParams, BusType},
    solver::network::build_admittance_matrix,
    state::AppState,
    auth::Claims,
    error::ApiError,
};

#[derive(serde::Deserialize)]
pub struct CreateStudyRequest {
    pub study_type: String,  // "power_flow" | "short_circuit" | "protection"
    pub analysis_method: String,  // "newton_raphson" | "fast_decoupled"
    pub parameters: serde_json::Value,
}

pub async fn create_study(
    State(state): State<AppState>,
    claims: Claims,
    Path(project_id): Path<Uuid>,
    Json(req): Json<CreateStudyRequest>,
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

    // 2. Extract bus and branch data from diagram
    let input_data = extract_system_data(&project.diagram_data, project.base_mva)?;
    let bus_count = input_data["buses"].as_array()
        .ok_or(ApiError::InvalidInput("No buses defined"))?
        .len();

    // 3. Check plan limits
    let user = sqlx::query_as!(
        crate::db::models::User,
        "SELECT * FROM users WHERE id = $1",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    if user.plan == "free" && bus_count > 25 {
        return Err(ApiError::PlanLimit(
            "Free plan supports systems up to 25 buses. Upgrade to Professional for unlimited."
        ));
    }

    // 4. Determine execution mode
    let execution_mode = if bus_count <= 100 {
        "wasm"
    } else {
        "server"
    };

    // 5. Create study record
    let study = sqlx::query_as!(
        Study,
        r#"INSERT INTO studies
            (project_id, user_id, study_type, analysis_method, status, execution_mode,
             input_data, parameters, bus_count)
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9)
        RETURNING *"#,
        project_id,
        claims.user_id,
        req.study_type,
        req.analysis_method,
        if execution_mode == "wasm" { "completed" } else { "pending" },
        execution_mode,
        input_data,
        req.parameters,
        bus_count as i32,
    )
    .fetch_one(&state.db)
    .await?;

    // 6. For server execution, enqueue job
    if execution_mode == "server" {
        let job = sqlx::query_as!(
            crate::db::models::StudyJob,
            r#"INSERT INTO study_jobs (study_id, priority, cores_allocated)
            VALUES ($1, $2, $3) RETURNING *"#,
            study.id,
            if user.plan == "utility" { 10 } else { 0 },
            if bus_count > 1000 { 8 } else { 4 },
        )
        .fetch_one(&state.db)
        .await?;

        // Publish to Redis job queue
        state.redis
            .publish("study:jobs", serde_json::to_string(&job.id)?)
            .await?;
    }

    Ok((StatusCode::CREATED, Json(study)))
}

pub async fn get_study(
    State(state): State<AppState>,
    claims: Claims,
    Path((project_id, study_id)): Path<(Uuid, Uuid)>,
) -> Result<Json<Study>, ApiError> {
    let study = sqlx::query_as!(
        Study,
        "SELECT s.* FROM studies s
         JOIN projects p ON s.project_id = p.id
         WHERE s.id = $1 AND s.project_id = $2
         AND (p.owner_id = $3 OR p.org_id IN (
             SELECT org_id FROM org_members WHERE user_id = $3
         ))",
        study_id,
        project_id,
        claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Study not found"))?;

    Ok(Json(study))
}

/// Extract bus and branch data from project diagram JSON
fn extract_system_data(
    diagram_data: &serde_json::Value,
    base_mva: f32,
) -> Result<serde_json::Value, ApiError> {
    let buses = diagram_data["buses"].as_array()
        .ok_or(ApiError::InvalidInput("No buses in diagram"))?;
    let branches = diagram_data["branches"].as_array()
        .ok_or(ApiError::InvalidInput("No branches in diagram"))?;

    // Validate at least one slack bus exists
    let slack_count = buses.iter()
        .filter(|b| b["type"].as_str() == Some("slack"))
        .count();
    if slack_count == 0 {
        return Err(ApiError::InvalidInput("System must have at least one slack bus"));
    }
    if slack_count > 1 {
        return Err(ApiError::InvalidInput("System can have only one slack bus"));
    }

    Ok(serde_json::json!({
        "buses": buses,
        "branches": branches,
        "base_mva": base_mva,
    }))
}
```

### 2. Power Flow Solver Core (Rust — shared between WASM and native)

The core Newton-Raphson solver that builds the admittance matrix, computes the Jacobian, and iteratively solves for bus voltages. This code compiles to both native (server) and WASM (browser) targets.

```rust
// solver-core/src/powerflow.rs

use nalgebra_sparse::{CsrMatrix, CooMatrix};
use num_complex::Complex64;
use crate::admittance::AdmittanceMatrix;
use crate::klu::KluSolver;

pub struct PowerFlowSystem {
    pub n_buses: usize,
    pub slack_bus: usize,
    pub pv_buses: Vec<usize>,
    pub pq_buses: Vec<usize>,
    pub y_bus: CsrMatrix<Complex64>,  // Admittance matrix
    pub voltages: Vec<Complex64>,     // Bus voltages (V∠θ)
    pub p_injection: Vec<f64>,        // Real power injection per bus
    pub q_injection: Vec<f64>,        // Reactive power injection per bus
}

impl PowerFlowSystem {
    pub fn new(
        buses: &[BusData],
        branches: &[BranchData],
    ) -> Result<Self, SolverError> {
        let n_buses = buses.len();
        let mut admittance = AdmittanceMatrix::new(n_buses);

        // Build Y-bus from branch data
        for branch in branches {
            if !branch.in_service { continue; }

            let from_idx = buses.iter().position(|b| b.bus_id == branch.from_bus)
                .ok_or(SolverError::InvalidBus(branch.from_bus.clone()))?;
            let to_idx = buses.iter().position(|b| b.bus_id == branch.to_bus)
                .ok_or(SolverError::InvalidBus(branch.to_bus.clone()))?;

            // Series admittance y = 1/(r + jx)
            let z = Complex64::new(branch.r_pu, branch.x_pu);
            let y_series = Complex64::new(1.0, 0.0) / z;

            // Shunt admittance (line charging)
            let y_shunt = Complex64::new(0.0, branch.b_pu / 2.0);

            // Transformer tap ratio
            let tap = branch.tap_ratio;
            let phase = branch.phase_shift_deg.to_radians();
            let tap_complex = Complex64::from_polar(tap, phase);

            // Stamp into Y-bus
            admittance.add_branch(from_idx, to_idx, y_series, y_shunt, tap_complex);
        }

        // Add shunt elements at buses
        for (i, bus) in buses.iter().enumerate() {
            let y_shunt = Complex64::new(bus.shunt_g_pu, bus.shunt_b_pu);
            admittance.add_shunt(i, y_shunt);
        }

        let y_bus = admittance.to_csr();

        // Initialize voltages (flat start or from bus data)
        let voltages: Vec<Complex64> = buses.iter().map(|bus| {
            Complex64::from_polar(bus.voltage_pu, bus.angle_deg.to_radians())
        }).collect();

        // Classify buses
        let slack_bus = buses.iter().position(|b| matches!(b.bus_type, BusType::Slack))
            .ok_or(SolverError::NoSlackBus)?;
        let pv_buses: Vec<usize> = buses.iter().enumerate()
            .filter(|(_, b)| matches!(b.bus_type, BusType::PV))
            .map(|(i, _)| i)
            .collect();
        let pq_buses: Vec<usize> = buses.iter().enumerate()
            .filter(|(_, b)| matches!(b.bus_type, BusType::PQ))
            .map(|(i, _)| i)
            .collect();

        // Compute net power injections
        let p_injection: Vec<f64> = buses.iter()
            .map(|b| b.p_gen_mw - b.p_load_mw)
            .collect();
        let q_injection: Vec<f64> = buses.iter()
            .map(|b| b.q_gen_mvar - b.q_load_mvar)
            .collect();

        Ok(Self {
            n_buses,
            slack_bus,
            pv_buses,
            pq_buses,
            y_bus,
            voltages,
            p_injection,
            q_injection,
        })
    }

    /// Solve power flow using Newton-Raphson method
    pub fn solve_newton_raphson(
        &mut self,
        params: &PowerFlowParams,
    ) -> Result<PowerFlowResult, SolverError> {
        for iteration in 0..params.max_iterations {
            // 1. Calculate power mismatches
            let (p_calc, q_calc) = self.calculate_power_injections();
            let mut max_mismatch = 0.0;
            let mut dp = vec![0.0; self.n_buses];
            let mut dq = vec![0.0; self.n_buses];

            for i in 0..self.n_buses {
                if i == self.slack_bus { continue; }

                dp[i] = self.p_injection[i] - p_calc[i];
                max_mismatch = max_mismatch.max(dp[i].abs());

                if !self.pv_buses.contains(&i) {
                    dq[i] = self.q_injection[i] - q_calc[i];
                    max_mismatch = max_mismatch.max(dq[i].abs());
                }
            }

            // 2. Check convergence
            if max_mismatch < params.tolerance {
                return Ok(PowerFlowResult {
                    converged: true,
                    iterations: iteration,
                    voltages: self.voltages.clone(),
                    p_injections: p_calc,
                    q_injections: q_calc,
                });
            }

            // 3. Build Jacobian matrix
            let jacobian = self.build_jacobian();

            // 4. Solve for voltage corrections: J * Δx = Δf
            let mismatch = self.build_mismatch_vector(&dp, &dq);
            let solver = KluSolver::new(&jacobian)?;
            let corrections = solver.solve(&mismatch)?;

            // 5. Update voltages
            self.apply_corrections(&corrections);

            // 6. Enforce PV bus voltage magnitude constraints
            for &i in &self.pv_buses {
                let v_mag = self.voltages[i].norm();
                let v_spec = self.p_injection[i];  // Stored in p_injection for PV buses
                self.voltages[i] = self.voltages[i] * (v_spec / v_mag);
            }
        }

        Err(SolverError::NoConvergence(format!(
            "Power flow did not converge in {} iterations", params.max_iterations
        )))
    }

    /// Calculate power injections at all buses: P_i = V_i Σ V_j Y_ij* cos(θ_ij + α_ij)
    fn calculate_power_injections(&self) -> (Vec<f64>, Vec<f64>) {
        let mut p = vec![0.0; self.n_buses];
        let mut q = vec![0.0; self.n_buses];

        for i in 0..self.n_buses {
            let v_i = self.voltages[i];

            for j in 0..self.n_buses {
                let y_ij = self.y_bus.get_entry(i, j)
                    .map(|e| *e.into_value())
                    .unwrap_or(Complex64::new(0.0, 0.0));
                let v_j = self.voltages[j];

                let s = v_i * (y_ij * v_j).conj();
                p[i] += s.re;
                q[i] += s.im;
            }
        }

        (p, q)
    }

    /// Build the Jacobian matrix: [J1 J2; J3 J4] where J1=∂P/∂θ, J2=∂P/∂V, J3=∂Q/∂θ, J4=∂Q/∂V
    fn build_jacobian(&self) -> CsrMatrix<f64> {
        let n_pq = self.pq_buses.len();
        let n_pv = self.pv_buses.len();
        let n = n_pq + n_pv;  // Number of unknown angles (all except slack)
        let m = n_pq;         // Number of unknown voltage magnitudes (only PQ buses)

        let mut jac = CooMatrix::new(n + m, n + m);

        // J1: ∂P/∂θ (top-left n×n block)
        for (row_idx, &i) in self.pq_buses.iter().chain(self.pv_buses.iter()).enumerate() {
            for (col_idx, &j) in self.pq_buses.iter().chain(self.pv_buses.iter()).enumerate() {
                let val = if i == j {
                    -self.q_injections_diagonal(i)
                } else {
                    self.partial_p_theta(i, j)
                };
                jac.push(row_idx, col_idx, val);
            }
        }

        // J2: ∂P/∂V (top-right n×m block)
        for (row_idx, &i) in self.pq_buses.iter().chain(self.pv_buses.iter()).enumerate() {
            for (col_idx, &j) in self.pq_buses.iter().enumerate() {
                let val = self.partial_p_vmag(i, j);
                jac.push(row_idx, n + col_idx, val);
            }
        }

        // J3: ∂Q/∂θ (bottom-left m×n block)
        for (row_idx, &i) in self.pq_buses.iter().enumerate() {
            for (col_idx, &j) in self.pq_buses.iter().chain(self.pv_buses.iter()).enumerate() {
                let val = self.partial_q_theta(i, j);
                jac.push(n + row_idx, col_idx, val);
            }
        }

        // J4: ∂Q/∂V (bottom-right m×m block)
        for (row_idx, &i) in self.pq_buses.iter().enumerate() {
            for (col_idx, &j) in self.pq_buses.iter().enumerate() {
                let val = if i == j {
                    self.p_injections_diagonal(i)
                } else {
                    self.partial_q_vmag(i, j)
                };
                jac.push(n + row_idx, n + col_idx, val);
            }
        }

        CsrMatrix::from(&jac)
    }

    fn partial_p_theta(&self, i: usize, j: usize) -> f64 {
        let v_i = self.voltages[i].norm();
        let v_j = self.voltages[j].norm();
        let theta_ij = self.voltages[i].arg() - self.voltages[j].arg();
        let y_ij = self.y_bus.get_entry(i, j)
            .map(|e| *e.into_value())
            .unwrap_or(Complex64::new(0.0, 0.0));
        let g = y_ij.re;
        let b = y_ij.im;

        v_i * v_j * (g * theta_ij.sin() - b * theta_ij.cos())
    }

    fn partial_p_vmag(&self, i: usize, j: usize) -> f64 {
        let v_i = self.voltages[i].norm();
        let theta_ij = self.voltages[i].arg() - self.voltages[j].arg();
        let y_ij = self.y_bus.get_entry(i, j)
            .map(|e| *e.into_value())
            .unwrap_or(Complex64::new(0.0, 0.0));
        let g = y_ij.re;
        let b = y_ij.im;

        if i == j {
            2.0 * v_i * g + self.p_injection[i] / v_i
        } else {
            v_i * (g * theta_ij.cos() + b * theta_ij.sin())
        }
    }

    fn partial_q_theta(&self, i: usize, j: usize) -> f64 {
        let v_i = self.voltages[i].norm();
        let v_j = self.voltages[j].norm();
        let theta_ij = self.voltages[i].arg() - self.voltages[j].arg();
        let y_ij = self.y_bus.get_entry(i, j)
            .map(|e| *e.into_value())
            .unwrap_or(Complex64::new(0.0, 0.0));
        let g = y_ij.re;
        let b = y_ij.im;

        v_i * v_j * (-g * theta_ij.cos() - b * theta_ij.sin())
    }

    fn partial_q_vmag(&self, i: usize, j: usize) -> f64 {
        let v_i = self.voltages[i].norm();
        let theta_ij = self.voltages[i].arg() - self.voltages[j].arg();
        let y_ij = self.y_bus.get_entry(i, j)
            .map(|e| *e.into_value())
            .unwrap_or(Complex64::new(0.0, 0.0));
        let g = y_ij.re;
        let b = y_ij.im;

        v_i * (g * theta_ij.sin() - b * theta_ij.cos())
    }

    fn build_mismatch_vector(&self, dp: &[f64], dq: &[f64]) -> Vec<f64> {
        let mut mismatch = Vec::new();

        // Add ΔP for all buses except slack
        for &i in self.pq_buses.iter().chain(self.pv_buses.iter()) {
            mismatch.push(dp[i]);
        }

        // Add ΔQ for PQ buses only
        for &i in &self.pq_buses {
            mismatch.push(dq[i]);
        }

        mismatch
    }

    fn apply_corrections(&mut self, corrections: &[f64]) {
        let n_pq = self.pq_buses.len();
        let n_pv = self.pv_buses.len();

        // Update angles for PQ and PV buses
        for (idx, &i) in self.pq_buses.iter().chain(self.pv_buses.iter()).enumerate() {
            let delta_theta = corrections[idx];
            let v_mag = self.voltages[i].norm();
            let theta = self.voltages[i].arg() + delta_theta;
            self.voltages[i] = Complex64::from_polar(v_mag, theta);
        }

        // Update voltage magnitudes for PQ buses only
        for (idx, &i) in self.pq_buses.iter().enumerate() {
            let delta_v = corrections[n_pq + n_pv + idx];
            let v_mag = self.voltages[i].norm() + delta_v;
            let theta = self.voltages[i].arg();
            self.voltages[i] = Complex64::from_polar(v_mag, theta);
        }
    }
}

#[derive(Debug)]
pub struct PowerFlowResult {
    pub converged: bool,
    pub iterations: u32,
    pub voltages: Vec<Complex64>,
    pub p_injections: Vec<f64>,
    pub q_injections: Vec<f64>,
}

#[derive(Debug)]
pub enum SolverError {
    NoConvergence(String),
    Singular(String),
    InvalidBus(String),
    NoSlackBus,
}
```

### 3. Server-Side Study Worker (Rust + Redis)

Background worker that picks up server-side study jobs from Redis, runs the native solver, streams progress via WebSocket, and stores results in S3.

```rust
// src/workers/study_worker.rs

use std::sync::Arc;
use aws_sdk_s3::Client as S3Client;
use redis::AsyncCommands;
use sqlx::PgPool;
use tokio::sync::broadcast;
use uuid::Uuid;

use crate::solver::{
    powerflow::{PowerFlowSystem, PowerFlowParams},
    shortcircuit::fault_analysis,
};
use crate::db::models::{StudyJob, BusData, BranchData};
use crate::websocket::ProgressBroadcaster;

pub struct StudyWorker {
    db: PgPool,
    redis: redis::Client,
    s3: S3Client,
    broadcaster: Arc<ProgressBroadcaster>,
}

impl StudyWorker {
    pub async fn run(&self) -> anyhow::Result<()> {
        let mut conn = self.redis.get_async_connection().await?;
        tracing::info!("Study worker started, listening for jobs...");

        loop {
            // Block-pop from Redis job queue (priority-sorted set)
            let job_id: Option<String> = conn.brpop("study:jobs", 30.0).await?;
            let Some(job_id) = job_id else { continue };
            let job_id: Uuid = job_id.parse()?;

            if let Err(e) = self.process_job(job_id).await {
                tracing::error!("Job {job_id} failed: {e}");
                self.mark_failed(job_id, &e.to_string()).await?;
            }
        }
    }

    async fn process_job(&self, job_id: Uuid) -> anyhow::Result<()> {
        // 1. Fetch job and study details
        let job = sqlx::query_as!(
            StudyJob,
            "UPDATE study_jobs SET started_at = NOW(), worker_id = $2
             WHERE id = $1 RETURNING *",
            job_id, hostname::get()?.to_string_lossy().to_string()
        )
        .fetch_one(&self.db)
        .await?;

        let study = sqlx::query_as!(
            crate::db::models::Study,
            "UPDATE studies SET status = 'running', started_at = NOW()
             WHERE id = $1 RETURNING *",
            job.study_id
        )
        .fetch_one(&self.db)
        .await?;

        // 2. Parse input data
        let buses: Vec<BusData> = serde_json::from_value(
            study.input_data["buses"].clone()
        )?;
        let branches: Vec<BranchData> = serde_json::from_value(
            study.input_data["branches"].clone()
        )?;

        // 3. Run study based on type
        let result_data = match study.study_type.as_str() {
            "power_flow" => {
                let params: PowerFlowParams = serde_json::from_value(
                    study.parameters.clone()
                )?;
                let mut system = PowerFlowSystem::new(&buses, &branches)?;

                // Run with progress callback
                let progress_tx = self.broadcaster.get_channel(study.id);
                let result = system.solve_newton_raphson(&params)?;

                // Stream progress updates
                for iter in 0..result.iterations {
                    let pct = (iter as f32 / params.max_iterations as f32 * 100.0).min(95.0);
                    let _ = progress_tx.send(serde_json::json!({
                        "progress_pct": pct,
                        "iteration": iter,
                    }));

                    sqlx::query!(
                        "UPDATE study_jobs SET progress_pct = $1 WHERE id = $2",
                        pct, job_id
                    )
                    .execute(&self.db)
                    .await?;
                }

                serde_json::to_vec(&result)?
            }
            "short_circuit" => {
                let params: crate::db::models::ShortCircuitParams =
                    serde_json::from_value(study.parameters.clone())?;
                let result = fault_analysis(&buses, &branches, &params)?;
                serde_json::to_vec(&result)?
            }
            _ => anyhow::bail!("Unknown study type: {}", study.study_type),
        };

        // 4. Upload results to S3
        let s3_key = format!("results/{}/{}.json", study.project_id, study.id);
        self.s3.put_object()
            .bucket("gridsolve-results")
            .key(&s3_key)
            .body(result_data.into())
            .content_type("application/json")
            .send()
            .await?;

        let results_url = format!("s3://gridsolve-results/{}", s3_key);

        // 5. Update database with completed status
        sqlx::query!(
            "UPDATE studies SET status = 'completed', results_url = $2,
             completed_at = NOW(), wall_time_ms = EXTRACT(EPOCH FROM (NOW() - started_at))::int * 1000
             WHERE id = $1",
            study.id, results_url
        )
        .execute(&self.db)
        .await?;

        sqlx::query!(
            "UPDATE study_jobs SET completed_at = NOW(), progress_pct = 100.0
             WHERE id = $1",
            job_id
        )
        .execute(&self.db)
        .await?;

        // 6. Notify client via WebSocket
        self.broadcaster.send(study.id, serde_json::json!({
            "status": "completed",
            "results_url": results_url,
        }))?;

        tracing::info!("Job {job_id} completed successfully");
        Ok(())
    }

    async fn mark_failed(&self, job_id: Uuid, error: &str) -> anyhow::Result<()> {
        sqlx::query!(
            "UPDATE study_jobs SET completed_at = NOW() WHERE id = $1",
            job_id
        )
        .execute(&self.db)
        .await?;

        let study_id: Uuid = sqlx::query_scalar!(
            "SELECT study_id FROM study_jobs WHERE id = $1",
            job_id
        )
        .fetch_one(&self.db)
        .await?;

        sqlx::query!(
            "UPDATE studies SET status = 'failed', error_message = $2,
             completed_at = NOW() WHERE id = $1",
            study_id, error
        )
        .execute(&self.db)
        .await?;

        Ok(())
    }
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–5)

**Day 1: Project initialization and Rust backend scaffold**
```bash
cargo init gridsolve-api
cd gridsolve-api
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
- `migrations/001_initial.sql` — All 9 tables: users, organizations, org_members, projects, studies, study_jobs, equipment_models, contingencies, protection_configs, usage_records
- `src/db/mod.rs` — Database pool initialization
- `src/db/models.rs` — All SQLx structs with FromRow derives
- Run `sqlx migrate run` to apply schema
- Seed script for initial equipment models (basic transformer, line, generator symbols)

**Day 3: Authentication system**
- `src/auth/mod.rs` — JWT token generation and validation middleware
- `src/auth/oauth.rs` — Google and GitHub OAuth 2.0 flow handlers
- `src/api/handlers/auth.rs` — Register, login, OAuth callback, refresh token, me endpoints
- Password hashing with bcrypt (cost factor 12)
- JWT with 24h access token + 30d refresh token
- Auth middleware that extracts Claims from Authorization header

**Day 4-5: User and project CRUD**
- `src/api/handlers/users.rs` — Get profile, update profile, delete account
- `src/api/handlers/projects.rs` — Create, list, get, update, delete, fork project
- `src/api/handlers/orgs.rs` — Create org, invite member, list members, remove member
- `src/api/router.rs` — All route definitions with auth middleware
- Integration tests: auth flow, project CRUD, authorization checks

### Phase 2 — Solver Core — Prototype (Days 6–14)

**Day 6: Admittance matrix framework**
- `solver-core/` — New Rust workspace member (shared between native and WASM)
- `solver-core/src/admittance.rs` — AdmittanceMatrix struct with branch/shunt stamping
- `solver-core/src/klu.rs` — KLU sparse solver wrapper via `suitesparse-sys` bindings
- Unit tests: simple 2-bus system, 3-bus radial feeder

**Day 7: Bus and branch models**
- `solver-core/src/bus.rs` — BusData struct with voltage, power, shunt
- `solver-core/src/branch.rs` — BranchData struct with impedance, tap ratio, phase shift
- `solver-core/src/network.rs` — Network topology validation (connected graph, unique bus IDs)
- Tests: IEEE 5-bus test system admittance matrix construction

**Day 8-9: Newton-Raphson power flow**
- `solver-core/src/powerflow.rs` — PowerFlowSystem struct with Newton-Raphson solver
- Jacobian matrix construction (∂P/∂θ, ∂P/∂V, ∂Q/∂θ, ∂Q/∂V)
- Power mismatch calculation and convergence checking
- Tests: IEEE 5-bus, IEEE 14-bus, IEEE 30-bus test systems (compare to published results)

**Day 10: Fast-decoupled power flow**
- `solver-core/src/powerflow_fd.rs` — Fast-decoupled method implementation
- B' and B'' matrix construction (constant decoupled Jacobians)
- One-time matrix factorization with KLU
- Tests: IEEE 14-bus, IEEE 30-bus (compare convergence speed vs Newton-Raphson)

**Day 11: Gauss-Seidel power flow**
- `solver-core/src/powerflow_gs.rs` — Gauss-Seidel iterative solver
- Acceleration factor tuning (default 1.6)
- Tests: small radial systems (5-10 buses)

**Day 12: Unbalanced three-phase power flow**
- `solver-core/src/powerflow_unbalanced.rs` — Three-phase admittance and solver
- 3×3 impedance matrices for lines and transformers
- Tests: IEEE 13-bus distribution test feeder

**Day 13-14: Short-circuit analysis (IEC 60909)**
- `solver-core/src/shortcircuit.rs` — Fault analysis with sequence networks
- Positive, negative, zero sequence impedance extraction
- Three-phase, SLG, LL, DLG fault calculation
- Tests: IEEE 9-bus with fault at various buses (compare to commercial tools)

### Phase 3 — WASM Build + Frontend Visualization (Days 15–21)

**Day 15: WASM solver compilation**
- `solver-wasm/` — New workspace member for WASM target
- `solver-wasm/Cargo.toml` — WASM dependencies (wasm-bindgen, js-sys)
- `solver-wasm/src/lib.rs` — WASM bindings for power flow solver
- `wasm-pack build --target web` — Generate JS/WASM bundle
- Test in browser: load WASM, run 5-bus power flow, log results

**Day 16: Frontend scaffold**
- `frontend/` — Vite + React + TypeScript
- `npm create vite@latest frontend -- --template react-ts`
- Zustand stores: authStore, projectStore, studyStore, diagramStore
- Axios API client with JWT interceptor
- React Router with auth-protected routes

**Day 17-18: One-line diagram editor (SVG)**
- `frontend/src/components/DiagramEditor.tsx` — SVG canvas with pan/zoom
- `frontend/src/components/BusNode.tsx` — Bus symbol (circle with label)
- `frontend/src/components/BranchLine.tsx` — Branch line (polyline with arrow)
- `frontend/src/lib/rbush.ts` — R-tree spatial index for hit testing
- Drag-and-drop equipment from library onto canvas
- Smart bus snapping (snap to nearest bus within 20px)
- Auto-routing for branch lines (orthogonal routing around obstacles)

**Day 19: Equipment library panel**
- `frontend/src/components/EquipmentLibrary.tsx` — Searchable equipment list
- `frontend/src/api/equipment.ts` — API calls to fetch equipment models
- PostgreSQL full-text search on equipment names and tags
- Display equipment symbols as SVG thumbnails

**Day 20-21: Results viewer (Canvas + WebGL)**
- `frontend/src/components/ResultsViewer.tsx` — Canvas renderer for voltage profiles
- `frontend/src/components/VoltageProfile.tsx` — Bar chart of bus voltages (WebGL)
- `frontend/src/components/LoadingTable.tsx` — Equipment loading table (branch flows vs ratings)
- `frontend/src/lib/webgl-renderer.ts` — WebGL setup and rendering utilities
- Color-coded voltage violations (red <0.95 pu, yellow 0.95-0.97, green 0.97-1.05)

### Phase 4 — Study Execution + Results (Days 22–28)

**Day 22: Study API handlers**
- `src/api/handlers/studies.rs` — Create, list, get, delete study
- `src/api/handlers/studies.rs::create_study` — Validate system, enqueue job or return WASM flag
- Integration test: create study, check database record

**Day 23: WASM study execution (client-side)**
- `frontend/src/lib/wasm-solver.ts` — Load WASM, run power flow in browser
- Convert diagram JSON to solver input format
- Display results immediately (no server round-trip)
- Handle solver errors (no convergence, invalid data)

**Day 24-25: Server-side study worker**
- `src/workers/study_worker.rs` — Redis job consumer
- Power flow execution with progress updates
- S3 result upload (JSON format)
- WebSocket progress broadcast to frontend

**Day 26: WebSocket progress streaming**
- `src/websocket/mod.rs` — Axum WebSocket handler
- `src/websocket/broadcaster.rs` — Broadcast channel per study ID
- `frontend/src/hooks/useStudyProgress.ts` — React hook for progress updates
- Display progress bar during server-side execution

**Day 27-28: Results display and download**
- `frontend/src/components/StudyResults.tsx` — Results summary card
- Fetch results from S3 presigned URL
- Display voltage profile, loading violations, convergence stats
- Export results as CSV and JSON

### Phase 5 — Protection Coordination + TCC (Days 29–33)

**Day 29: Relay curve library**
- `scripts/seed_relay_curves.py` — Parse IEC/IEEE standard curves (IEC 60255, IEEE C37.112)
- Standard curves: IEC Standard Inverse, IEC Very Inverse, IEC Extremely Inverse, IEEE Moderately Inverse, IEEE Very Inverse, IEEE Extremely Inverse
- Store curve parameters in PostgreSQL equipment_models table
- Seed 20+ generic relay curves

**Day 30-31: TCC plotting engine**
- `frontend/src/components/TCCPlot.tsx` — Canvas-based time-current curve plotter
- Log-log scale rendering (current 1A-100kA, time 0.01s-1000s)
- Plot multiple relay curves with different settings (pickup, time dial)
- Plot fault current points from short-circuit analysis

**Day 32: Protection coordination UI**
- `frontend/src/components/ProtectionCoordinator.tsx` — UI for relay selection and settings
- Select relays for each branch, configure pickup current and time dial
- Visual feedback for coordination violations (curves intersecting within CTI margin)
- Save protection configuration to database

**Day 33: Basic coordination validation**
- `solver-core/src/protection.rs` — Coordination time interval (CTI) checking
- Find curve intersections (numerical method)
- Report coordination violations with recommendations
- Tests: simple radial feeder with 3 relays

### Phase 6 — Billing + Deployment (Days 34–42)

**Day 34-35: Stripe integration**
- `src/billing/mod.rs` — Stripe SDK integration
- `src/api/handlers/billing.rs` — Create checkout session, customer portal, webhook handler
- Three pricing tiers: Free (25 buses), Professional $149/mo (5,000 buses), Utility $399/user/mo (unlimited)
- Webhook handling: subscription created, updated, cancelled
- Update user.plan in database on subscription changes

**Day 36-37: Usage tracking and enforcement**
- `src/middleware/usage.rs` — Track study execution time and storage
- `src/db/queries/usage.rs` — Insert usage records, check quota
- Plan limits: Free (10 studies/mo), Professional (500 studies/mo), Utility (unlimited)
- Return 402 Payment Required when quota exceeded

**Day 38-39: Report generation (Python FastAPI)**
- `report-service/` — Python FastAPI service for PDF generation
- `report-service/templates/power_flow_report.html` — Jinja2 template for power flow reports
- `report-service/pdf_renderer.py` — WeasyPrint HTML-to-PDF rendering
- Include one-line diagram (SVG embed), voltage profile plots, loading tables, compliance statements
- Deploy as separate microservice (Docker container)

**Day 40-41: Infrastructure setup**
- AWS ECS Fargate for API and worker containers
- RDS PostgreSQL (db.r6g.large, Multi-AZ)
- ElastiCache Redis (r6g.large, cluster mode)
- S3 buckets: gridsolve-wasm, gridsolve-results, gridsolve-equipment
- CloudFront CDN for WASM and frontend assets
- GitHub Actions CI/CD: build, test, push Docker images, deploy to ECS

**Day 42: Final integration testing and validation**
- End-to-end test: register user, create project, add equipment, run power flow, view results
- Performance benchmarking: 100-bus power flow in <1s (WASM), 1000-bus in <5s (server)
- Security audit: SQL injection tests, XSS tests, auth bypass attempts
- Load testing: 100 concurrent users, 1000 studies/hour

---

## Validation Benchmarks

| Test | Target | Validation Method |
|------|--------|-------------------|
| IEEE 5-bus Newton-Raphson | Converge in 3-4 iterations, max voltage error <0.001 pu | Compare to IEEE published results |
| IEEE 14-bus power flow | All bus voltages within 0.0001 pu of reference | IEEE 14-bus test case solution |
| IEEE 30-bus power flow | Total system losses 17.5-18.0 MW | IEEE 30-bus test case solution |
| WASM solver: 50-bus system | <500ms total execution time | Browser performance.now() timer |
| WASM solver: 100-bus system | <1s total execution time | Browser performance.now() timer |
| Server solver: 1000-bus system | <5s wall time on 4 cores | Server timing logs |
| Server solver: 5000-bus system | <30s wall time on 8 cores | Server timing logs |
| Short-circuit IEEE 9-bus | Three-phase fault current within 5% of PSS/E | Compare to commercial tool |
| Fast-decoupled vs Newton-Raphson | FD converges in 60% of iterations with <0.1% error | IEEE 30-bus comparison |
| Unbalanced power flow IEEE 13-bus | Phase voltages within 0.5% of OpenDSS | OpenDSS distribution test feeder |
| TCC plot rendering 10 curves | Render at 60 FPS during pan/zoom | Chrome FPS counter |
| One-line editor 200 elements | 60 FPS during drag operations | Chrome FPS counter |
| Results viewer voltage profile | Render 1000 buses in <100ms | Canvas timing |
| API: create study | p95 latency <200ms | Prometheus metrics |
| WebSocket progress latency | <100ms from worker to client | End-to-end timing |

---

## CIM Import/Export

### Common Information Model (CIM) Support

GridSolve implements **IEC 61970-301** (CIM for transmission) and **IEC 61968-13** (CIM for distribution) for interoperability with utility SCADA/EMS systems and other power system tools.

```rust
// src/cim/parser.rs

use roxmltree::Document;
use uuid::Uuid;
use crate::db::models::{BusData, BranchData, BusType};

pub struct CimParser {
    pub buses: Vec<CimBus>,
    pub branches: Vec<CimBranch>,
    pub generators: Vec<CimGenerator>,
    pub loads: Vec<CimLoad>,
}

impl CimParser {
    /// Parse CIM XML file (IEC 61970-301 EQ/TP/SSH profiles)
    pub fn from_xml(xml_content: &str) -> Result<Self, CimError> {
        let doc = Document::parse(xml_content)?;
        let root = doc.root_element();

        let mut parser = Self {
            buses: Vec::new(),
            branches: Vec::new(),
            generators: Vec::new(),
            loads: Vec::new(),
        };

        // Parse TopologicalNode (buses)
        for node in root.descendants().filter(|n| n.has_tag_name("cim:TopologicalNode")) {
            parser.buses.push(CimBus {
                id: node.attribute("rdf:ID").unwrap_or("").to_string(),
                name: get_child_text(&node, "cim:IdentifiedObject.name"),
                base_voltage: get_child_text(&node, "cim:TopologicalNode.BaseVoltage")
                    .parse().unwrap_or(0.0),
            });
        }

        // Parse ACLineSegment (transmission lines)
        for node in root.descendants().filter(|n| n.has_tag_name("cim:ACLineSegment")) {
            parser.branches.push(CimBranch {
                id: node.attribute("rdf:ID").unwrap_or("").to_string(),
                from_bus: get_reference(&node, "cim:ACLineSegment.Terminal1"),
                to_bus: get_reference(&node, "cim:ACLineSegment.Terminal2"),
                r: get_child_text(&node, "cim:ACLineSegment.r").parse().unwrap_or(0.0),
                x: get_child_text(&node, "cim:ACLineSegment.x").parse().unwrap_or(0.0),
                bch: get_child_text(&node, "cim:ACLineSegment.bch").parse().unwrap_or(0.0),
                rating: get_child_text(&node, "cim:ACLineSegment.normalCurrentLimit")
                    .parse().unwrap_or(0.0),
            });
        }

        // Parse PowerTransformer (transformers)
        for node in root.descendants().filter(|n| n.has_tag_name("cim:PowerTransformer")) {
            // Extract winding data, tap positions, impedances
            // ...
        }

        // Parse SynchronousMachine (generators)
        for node in root.descendants().filter(|n| n.has_tag_name("cim:SynchronousMachine")) {
            parser.generators.push(CimGenerator {
                id: node.attribute("rdf:ID").unwrap_or("").to_string(),
                bus: get_reference(&node, "cim:SynchronousMachine.Terminal"),
                p_gen: get_child_text(&node, "cim:SynchronousMachine.p").parse().unwrap_or(0.0),
                q_gen: get_child_text(&node, "cim:SynchronousMachine.q").parse().unwrap_or(0.0),
                v_setpoint: get_child_text(&node, "cim:SynchronousMachine.targetVoltage")
                    .parse().unwrap_or(1.0),
            });
        }

        // Parse EnergyConsumer (loads)
        for node in root.descendants().filter(|n| n.has_tag_name("cim:EnergyConsumer")) {
            parser.loads.push(CimLoad {
                id: node.attribute("rdf:ID").unwrap_or("").to_string(),
                bus: get_reference(&node, "cim:EnergyConsumer.Terminal"),
                p_load: get_child_text(&node, "cim:EnergyConsumer.p").parse().unwrap_or(0.0),
                q_load: get_child_text(&node, "cim:EnergyConsumer.q").parse().unwrap_or(0.0),
            });
        }

        Ok(parser)
    }

    /// Convert CIM model to GridSolve internal format
    pub fn to_gridsolve_format(&self, base_mva: f64) -> (Vec<BusData>, Vec<BranchData>) {
        let mut buses = Vec::new();
        let mut branches = Vec::new();

        // Convert buses
        for cim_bus in &self.buses {
            // Determine bus type based on attached generators/loads
            let has_gen = self.generators.iter().any(|g| g.bus == cim_bus.id);
            let bus_type = if has_gen {
                BusType::PV  // Assume generator buses are PV
            } else {
                BusType::PQ  // Load buses are PQ
            };

            // Aggregate generation and load at this bus
            let p_gen: f64 = self.generators.iter()
                .filter(|g| g.bus == cim_bus.id)
                .map(|g| g.p_gen)
                .sum();
            let q_gen: f64 = self.generators.iter()
                .filter(|g| g.bus == cim_bus.id)
                .map(|g| g.q_gen)
                .sum();
            let p_load: f64 = self.loads.iter()
                .filter(|l| l.bus == cim_bus.id)
                .map(|l| l.p_load)
                .sum();
            let q_load: f64 = self.loads.iter()
                .filter(|l| l.bus == cim_bus.id)
                .map(|l| l.q_load)
                .sum();

            buses.push(BusData {
                bus_id: cim_bus.id.clone(),
                bus_type,
                base_kv: cim_bus.base_voltage,
                voltage_pu: 1.0,
                angle_deg: 0.0,
                p_gen_mw: p_gen,
                q_gen_mvar: q_gen,
                p_load_mw: p_load,
                q_load_mvar: q_load,
                shunt_g_pu: 0.0,
                shunt_b_pu: 0.0,
            });
        }

        // Set first bus with generation as slack bus
        if let Some(slack_idx) = buses.iter().position(|b| b.p_gen_mw > 0.0) {
            buses[slack_idx].bus_type = BusType::Slack;
        }

        // Convert branches
        for cim_branch in &self.branches {
            // Convert impedances from absolute to per-unit
            let z_base = (cim_branch.base_kv.powi(2)) / base_mva;
            branches.push(BranchData {
                branch_id: cim_branch.id.clone(),
                from_bus: cim_branch.from_bus.clone(),
                to_bus: cim_branch.to_bus.clone(),
                r_pu: cim_branch.r / z_base,
                x_pu: cim_branch.x / z_base,
                b_pu: cim_branch.bch * z_base,
                tap_ratio: 1.0,
                phase_shift_deg: 0.0,
                rating_mva: Some(cim_branch.rating),
                in_service: true,
            });
        }

        (buses, branches)
    }
}

fn get_child_text(node: &roxmltree::Node, tag: &str) -> String {
    node.descendants()
        .find(|n| n.has_tag_name(tag))
        .and_then(|n| n.text())
        .unwrap_or("")
        .to_string()
}

fn get_reference(node: &roxmltree::Node, tag: &str) -> String {
    node.descendants()
        .find(|n| n.has_tag_name(tag))
        .and_then(|n| n.attribute("rdf:resource"))
        .unwrap_or("")
        .trim_start_matches('#')
        .to_string()
}
```

---

## Performance Optimization Techniques

### 1. Sparse Matrix Optimization

```rust
// solver-core/src/sparse_utils.rs

use nalgebra_sparse::{CsrMatrix, CooMatrix};

/// Build CSR matrix with pre-allocated capacity for power flow Jacobian
pub fn build_jacobian_optimized(n_buses: usize) -> CooMatrix<f64> {
    // Estimate non-zero entries: each bus connects to avg 3 neighbors
    // Jacobian has 4 blocks, total NNZ ≈ 4 * n_buses * 4 = 16n
    let estimated_nnz = n_buses * 16;
    CooMatrix::with_capacity(n_buses * 2, n_buses * 2, estimated_nnz)
}

/// Reuse factorization for fast-decoupled power flow
pub struct CachedFactorization {
    b_prime: CsrMatrix<f64>,   // ∂P/∂θ approximation
    b_double: CsrMatrix<f64>,  // ∂Q/∂V approximation
    b_prime_lu: KluFactorization,
    b_double_lu: KluFactorization,
}

impl CachedFactorization {
    pub fn new(y_bus: &CsrMatrix<Complex64>, buses: &[BusData]) -> Self {
        // Build constant B' and B'' matrices (imaginary part of Y-bus)
        let b_prime = extract_imaginary_offdiagonal(y_bus);
        let b_double = extract_imaginary_diagonal(y_bus);

        // Factor once, reuse for all iterations
        let b_prime_lu = KluFactorization::new(&b_prime);
        let b_double_lu = KluFactorization::new(&b_double);

        Self { b_prime, b_double, b_prime_lu, b_double_lu }
    }

    pub fn solve_fast_decoupled(&mut self, dp: &[f64], dq: &[f64]) -> (Vec<f64>, Vec<f64>) {
        let d_theta = self.b_prime_lu.solve(dp);
        let d_vmag = self.b_double_lu.solve(dq);
        (d_theta, d_vmag)
    }
}
```

### 2. WASM Memory Management

```rust
// solver-wasm/src/lib.rs

use wasm_bindgen::prelude::*;
use js_sys::{Float64Array, Object, Reflect};

#[wasm_bindgen]
pub struct WasmPowerFlowSolver {
    system: PowerFlowSystem,
    result_buffer: Vec<f64>,  // Reusable buffer for results
}

#[wasm_bindgen]
impl WasmPowerFlowSolver {
    #[wasm_bindgen(constructor)]
    pub fn new(buses_json: &str, branches_json: &str) -> Result<WasmPowerFlowSolver, JsValue> {
        console_error_panic_hook::set_once();  // Better error messages

        let buses: Vec<BusData> = serde_json::from_str(buses_json)
            .map_err(|e| JsValue::from_str(&e.to_string()))?;
        let branches: Vec<BranchData> = serde_json::from_str(branches_json)
            .map_err(|e| JsValue::from_str(&e.to_string()))?;

        let system = PowerFlowSystem::new(&buses, &branches)
            .map_err(|e| JsValue::from_str(&format!("{:?}", e)))?;

        // Pre-allocate result buffer (avoid allocation per solve)
        let result_buffer = vec![0.0; buses.len() * 2];  // voltage mag + angle per bus

        Ok(Self { system, result_buffer })
    }

    /// Solve power flow, return results as Float64Array (zero-copy transfer)
    #[wasm_bindgen]
    pub fn solve(&mut self, params_json: &str) -> Result<Float64Array, JsValue> {
        let params: PowerFlowParams = serde_json::from_str(params_json)
            .map_err(|e| JsValue::from_str(&e.to_string()))?;

        let result = self.system.solve_newton_raphson(&params)
            .map_err(|e| JsValue::from_str(&format!("{:?}", e)))?;

        // Pack results into buffer: [V1_mag, V1_ang, V2_mag, V2_ang, ...]
        for (i, voltage) in result.voltages.iter().enumerate() {
            self.result_buffer[i * 2] = voltage.norm();
            self.result_buffer[i * 2 + 1] = voltage.arg().to_degrees();
        }

        // Zero-copy transfer to JavaScript (uses shared memory)
        Ok(Float64Array::from(&self.result_buffer[..]))
    }
}
```

---

## Phase 7 — Testing + Optimization (Days 33–38)

**Day 33: Solver validation — IEEE test systems**
- IEEE 5-bus system: verify convergence, compare voltages to published results
- IEEE 14-bus system: verify within 0.01% of reference solution
- IEEE 30-bus system: verify total losses, voltage profile
- IEEE 57-bus system: test scalability, convergence performance
- Tests: `solver-core/tests/ieee_benchmarks.rs` with golden reference data

**Day 34: Solver validation — short-circuit**
- IEEE 9-bus three-phase fault: compare to PSS/E results (within 5%)
- IEEE 14-bus SLG fault at various buses: verify sequence network calculations
- Fault contribution tables: verify current magnitudes and directions
- Equipment duty check: breaker interrupting capacity violations

**Day 35: Performance benchmarking**
- WASM solver: 50-bus system <500ms, 100-bus system <1s
- Server solver: 1000-bus <5s, 5000-bus <30s, 10000-bus <2min
- Fast-decoupled vs Newton-Raphson: speed comparison, accuracy comparison
- Memory profiling: ensure no memory leaks in long-running workers
- Database query optimization: index usage verification for equipment search

**Day 36: Integration testing**
- End-to-end: register → create project → add equipment → run study → view results → export
- Multi-user: 10 concurrent users running studies, no conflicts
- Stripe webhook testing: subscription lifecycle events
- CIM import: test with real utility CIM files (anonymized)
- WebSocket: connection stability under load, reconnection handling

**Day 37: Security audit**
- SQL injection testing: parameterized queries verification
- XSS testing: SVG injection attempts, script tag sanitization
- Auth bypass attempts: JWT validation, expired token rejection
- CORS policy verification: only allowed origins
- Rate limiting: API endpoint throttling per user/IP
- S3 bucket policy audit: no public access to results

**Day 38: Load testing and capacity planning**
- Locust load test: 100 concurrent users, 1000 studies/hour
- Worker autoscaling: verify HPA triggers at queue depth thresholds
- Database connection pool sizing: verify no connection exhaustion
- Redis queue performance: measure latency under 1000+ jobs
- CDN cache hit rate: verify WASM bundle caching effectiveness

---

## Phase 8 — Polish + Launch (Days 39–42)

**Day 39: UI/UX refinement**
- One-line editor: smooth panning/zooming, snappy selection
- Results viewer: loading states, error messages, empty states
- Study list: filtering, sorting, search
- Responsive design: tablet and desktop layouts
- Accessibility audit: keyboard navigation, screen reader support, ARIA labels
- Dark mode support for all components

**Day 40: Documentation and onboarding**
- User guide: getting started, tutorial for first power flow study
- Video tutorials: one-line diagram creation, running studies, interpreting results
- API documentation: OpenAPI spec generation, example requests
- Help tooltips: context-sensitive help for all UI elements
- Sample projects: IEEE test systems pre-loaded as public examples

**Day 41: Monitoring and observability setup**
- Prometheus metrics: API latency, study completion rate, error rate
- Grafana dashboards: system health, user activity, resource utilization
- Sentry error tracking: client-side and server-side error capture
- CloudWatch logs: centralized logging for ECS tasks
- Alerting rules: high error rate, slow API, worker backlog
- Uptime monitoring: external health check (Pingdom/UptimeRobot)

**Day 42: Final deployment and launch preparation**
- Production infrastructure deployment: ECS, RDS, ElastiCache, S3, CloudFront
- DNS configuration: gridsolve.io domain setup, SSL certificate
- Backup verification: database snapshots, S3 versioning
- Disaster recovery plan: RTO/RPO documentation, restore procedures
- Launch checklist: marketing site, pricing page, blog post, social media
- Beta user onboarding: send invites, collect feedback

---

## Operational Metrics and Monitoring

### Key Performance Indicators (KPIs)

```sql
-- migrations/002_analytics.sql

CREATE TABLE analytics_events (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    event_type TEXT NOT NULL,  -- study_created | study_completed | study_failed | diagram_saved | export_clicked
    event_data JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX analytics_events_type_idx ON analytics_events(event_type, created_at);
CREATE INDEX analytics_events_user_idx ON analytics_events(user_id);

-- Queries for dashboard metrics

-- 1. Study execution stats (last 7 days)
SELECT
    execution_mode,
    status,
    COUNT(*) as count,
    AVG(wall_time_ms)::int as avg_time_ms,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY wall_time_ms)::int as p95_time_ms
FROM studies
WHERE created_at >= NOW() - INTERVAL '7 days'
GROUP BY execution_mode, status;

-- 2. Study type distribution
SELECT study_type, COUNT(*) as count,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 1) as pct
FROM studies
WHERE created_at >= NOW() - INTERVAL '30 days'
GROUP BY study_type ORDER BY count DESC;

-- 3. User plan distribution and conversion
SELECT plan, COUNT(*) as users,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 1) as pct
FROM users GROUP BY plan ORDER BY users DESC;

-- 4. Server study usage by user (billing period)
SELECT u.email, u.plan,
    SUM(ur.quantity) as total_minutes,
    CASE u.plan
        WHEN 'free' THEN 10
        WHEN 'professional' THEN 500
        WHEN 'utility' THEN 2000
    END as limit_minutes
FROM usage_records ur
JOIN users u ON ur.user_id = u.id
WHERE ur.period_start <= CURRENT_DATE AND ur.period_end >= CURRENT_DATE
    AND ur.record_type = 'study_minutes'
GROUP BY u.email, u.plan
ORDER BY total_minutes DESC;

-- 5. Equipment model popularity
SELECT em.name, em.manufacturer, em.category,
    em.download_count,
    COUNT(DISTINCT s.project_id) as projects_using
FROM equipment_models em
LEFT JOIN studies s ON s.input_data::text ILIKE '%' || em.name || '%'
GROUP BY em.id
ORDER BY em.download_count DESC
LIMIT 20;
```

### Performance Benchmarks

| Operation | Target | Method |
|-----------|--------|--------|
| WASM solver: 50-bus Newton-Raphson | <500ms | Browser benchmark (Chrome DevTools) |
| WASM solver: 100-bus Newton-Raphson | <1s | Browser benchmark |
| Server solver: 1000-bus power flow | <5s | Server timing, 4 cores |
| Server solver: 5000-bus power flow | <30s | Server timing, 8 cores |
| Short-circuit IEEE 14-bus | <200ms | Server timing |
| Results viewer: voltage profile 1000 buses | 60 FPS | Chrome FPS counter during pan/zoom |
| One-line editor: 500 elements | 60 FPS | SVG rendering performance |
| API: create study | <200ms | p95 latency (Prometheus) |
| API: search equipment models | <100ms | p95 latency with 50K models in DB |
| WASM bundle load (cached) | <300ms | Service worker cache hit |
| WASM bundle load (cold) | <2s | CDN delivery, 1.5MB gzipped |
| WebSocket: progress latency | <100ms | Time from worker emit to client receive |
| CIM import: 100-bus system | <1s | Server-side parsing and conversion |
| TCC plot rendering 20 curves | Render at 60 FPS during pan/zoom | Canvas FPS counter |

---

## Deployment Architecture

### Infrastructure

```
┌─────────────────────────────────────────────────┐
│                  CloudFront CDN                   │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────┐ │
│  │ Frontend SPA │  │ WASM Bundle │  │ Equipment│ │
│  │ (HTML/JS/CSS)│  │ (versioned) │  │ Models   │ │
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
│ Multi-AZ     │ │ Cluster mode │ │  Equipment)  │
└──────────────┘ └──────┬───────┘ └──────────────┘
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ Study Worker │ │ Study Worker │ │ Study Worker │
│ (Rust native)│ │ (Rust native)│ │ (Rust native)│
│ c6g.2xlarge  │ │ c6g.2xlarge  │ │ c6g.2xlarge  │
│ HPA: 2-10    │ │              │ │              │
└──────────────┘ └──────────────┘ └──────────────┘
        │               │               │
        └───────────────┼───────────────┘
                        ▼
                ┌──────────────┐
                │ Report Svc   │
                │ (Python API) │
                │ ECS Fargate  │
                └──────────────┘
```

### Scaling Strategy

- **API servers**: Horizontal pod autoscaler (HPA) at 70% CPU, min 3, max 10
- **Study workers**: HPA based on Redis queue depth — scale up when queue > 5 jobs, scale down when idle 5 min
- **Worker instances**: AWS c6g.2xlarge (8 vCPU, 16GB RAM, ARM-based Graviton2 for cost efficiency)
- **Database**: RDS PostgreSQL r6g.large with read replicas for equipment search queries
- **Redis**: ElastiCache r6g.large cluster with 1 primary + 2 replicas

### S3 Lifecycle Policy

```json
{
  "Rules": [
    {
      "ID": "study-results-lifecycle",
      "Prefix": "results/",
      "Status": "Enabled",
      "Transitions": [
        { "Days": 30, "StorageClass": "STANDARD_IA" },
        { "Days": 90, "StorageClass": "GLACIER" }
      ],
      "Expiration": { "Days": 365 }
    },
    {
      "ID": "equipment-models-keep",
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
| Contingency Analysis (N-1/N-2) | Automated screening of single and double element outages with severity ranking based on voltage violations and equipment overloads | High |
| Optimal Power Flow (OPF) | Generation dispatch optimization to minimize cost while satisfying load and constraints, using interior-point or sequential quadratic programming | High |
| Transient Stability Simulation | Time-domain simulation with synchronous machine dynamics, governor/exciter models, and event analysis (faults, switching) for grid stability studies | High |
| Advanced Relay Coordination | Automated relay setting calculation using optimization algorithms to achieve selectivity with minimum CTI, support for distance relays and differential protection | High |
| Unbalanced Distribution Analysis | Full three-phase power flow for distribution feeders with single-phase laterals, unbalanced loads, and voltage regulator control | Medium |
| Hosting Capacity Analysis | Automated calculation of maximum distributed generation (DG) penetration at each bus without violating voltage or thermal limits | Medium |
| Real-Time State Estimation | Integration with SCADA telemetry to estimate current system state from noisy measurements using weighted least squares | Medium |
| Harmonic Analysis | Frequency-domain harmonic power flow to assess distortion from nonlinear loads and design passive/active filters | Medium |
| Arc Flash Calculation (IEEE 1584-2018) | Incident energy calculation at equipment locations with automatic PPE category determination and arc flash label generation | Low |
| Geographic Network View | Interactive map overlay of transmission/distribution network on Leaflet/Mapbox with geographic coordinates for line routing | Low |
| Collaboration (Real-Time) | CRDT-based multi-user editing of one-line diagrams with cursor presence, commenting on equipment, and shared study results | Low |
| API for Automation | REST API and Python SDK for programmatic study execution, batch processing, and integration with external optimization workflows | Low |
