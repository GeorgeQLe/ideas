# 91. BridgeWorks — Bridge Design and Rating Platform

## Implementation Plan

**MVP Scope:** Browser-based parametric bridge modeler for simple and continuous span prestressed I-girder bridges with composite section builder (AASHTO/PCI standard girders + deck + haunch), influence line-based moving load analysis implementing AASHTO HL-93 loading (design truck, tandem, lane load) with automatic critical positioning optimizer compiled to WebAssembly for instant client-side execution, AASHTO LRFD service limit state (stress limits) and strength limit state (Mn, Vn) design checks for prestressed concrete per Sections 5.7-5.9, time-dependent prestress loss calculation (approximate and refined methods) implementing creep/shrinkage per AASHTO 5.9.3 and staged construction analysis (strand release, deck pour, service), load rating factor (RF) computation per AASHTO MBE implementing LRFR method for Inventory, Operating, and Legal loads with automatic generation of rating summary tables, interactive 3D bridge visualization rendered via Three.js with pan/orbit controls and section property display, PDF design/rating report generation with code references and calculation summaries.

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| FEA Solver | Rust (native + WASM) | Custom beam FEM with influence line computation, staged construction solver |
| Moving Load Engine | Rust (native + WASM) | HL-93 load optimization using influence line integration, critical positioning search |
| Prestress Engine | Rust | Time-dependent creep/shrinkage integration (CEB-FIP, AASHTO models), staged construction tracking |
| WASM Compilation | `wasm-pack` + `wasm-bindgen` | Client-side solver for bridges ≤10 spans |
| ML Service | Python 3.12 (FastAPI) | Bridge condition prediction, deterioration modeling (future) |
| Database | PostgreSQL 16 + PostGIS | Bridge inventory, projects, NBI data, vehicle libraries, load ratings |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | Inspection photos, 3D model exports, analysis results, PDF reports |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| 3D Visualization | Three.js + React-Three-Fiber | GPU-accelerated 3D bridge rendering, orbit controls |
| 2D Diagramming | D3.js + SVG | Influence lines, load rating charts, elevation/plan views |
| Real-time | WebSocket (Axum) | Live analysis progress for large bridges |
| Job Queue | Redis 7 + Tokio tasks | Server-side batch rating job management |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | WASM bundle delivery, static frontend assets |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, error tracking |
| CI/CD | GitHub Actions | WASM build pipeline, Rust tests, Docker image push |

### Key Architecture Decisions

1. **Hybrid client/server solver with WASM threshold at 10 spans**: Bridges with ≤10 spans (covers 95%+ of typical highway bridges — simple span, 2-span, 3-span continuous) run entirely in the browser via WASM, providing instant feedback with zero server cost. Bridges exceeding 10 spans or requiring 3D refined analysis are executed server-side with Rust-native FEM. The threshold is configurable per plan tier.

2. **Custom beam FEM solver in Rust rather than wrapping commercial FEA**: Building a specialized beam element solver gives us domain-specific optimizations (influence line computation, construction stage tracking, prestress loss integration) while maintaining WASM compatibility. Commercial FEA engines (Nastran, Abaqus) are non-redistributable and have massive binaries incompatible with WASM. Our Rust solver uses `nalgebra` for matrix operations and custom iterative solvers optimized for beam stiffness matrices.

3. **Influence line pre-computation for moving load optimization**: Rather than running thousands of FEA load cases to find critical truck positions, we compute influence lines (unit load response) once per span, then use analytical integration to find the critical position that maximizes moment/shear at each section. This reduces HL-93 load optimization from O(10,000) FEA runs to O(1) influence line computation + O(100) position trials, cutting analysis time from minutes to seconds.

4. **Three.js for 3D bridge visualization separate from D3 diagrams**: The 3D renderer displays the complete bridge geometry (girders, deck, bearings, substructure) with material textures and lighting for realistic presentation. This is decoupled from 2D engineering diagrams (influence lines, rating charts) which use SVG/D3 for crisp line plots and annotations. The 3D view supports section cuts to show composite section properties and prestress strand layout.

5. **PostgreSQL + PostGIS for bridge inventory with S3 for media**: Bridge project metadata (geometry, analysis results, ratings) is stored in PostgreSQL with PostGIS extensions for geospatial queries (find bridges near location, along corridor). Inspection photos, 3D exports (IFC, DXF), and PDF reports are stored in S3, with PostgreSQL holding references and searchable metadata. This allows the platform to scale to 100K+ bridge inventory projects without bloating the database.

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "postgis";  -- For bridge location data

-- Users
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email TEXT UNIQUE NOT NULL,
    password_hash TEXT,  -- NULL for OAuth-only users
    name TEXT NOT NULL,
    avatar_url TEXT,
    auth_provider TEXT DEFAULT 'email',  -- email | google | github
    auth_provider_id TEXT,
    plan TEXT NOT NULL DEFAULT 'free',  -- free | pro | advanced | enterprise
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    organization TEXT,  -- Company/agency name
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX users_email_idx ON users(email);
CREATE INDEX users_stripe_idx ON users(stripe_customer_id);

-- Bridge Projects
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    bridge_type TEXT NOT NULL,  -- prestressed_girder | steel_plate_girder | box_girder | truss | arch | cable_stayed | suspension
    description TEXT DEFAULT '',
    location GEOGRAPHY(POINT, 4326),  -- PostGIS point (lat/lon)
    location_name TEXT,  -- Human-readable location
    nbi_number TEXT,  -- National Bridge Inventory ID for existing bridges
    design_code TEXT NOT NULL DEFAULT 'aashto_lrfd_9',  -- aashto_lrfd_9 | eurocode_2 | aashto_lrfd_8
    geometry_data JSONB NOT NULL DEFAULT '{}',  -- Span lengths, alignment, cross-section parameters
    section_data JSONB NOT NULL DEFAULT '{}',  -- Girder type, deck thickness, materials
    prestress_data JSONB,  -- Strand layout, losses, stages (NULL for non-prestressed)
    analysis_results JSONB,  -- Cached FEA results, influence lines
    rating_results JSONB,  -- Load ratings (RF values)
    settings JSONB DEFAULT '{}',  -- Analysis options, display preferences
    is_public BOOLEAN NOT NULL DEFAULT false,
    forked_from UUID REFERENCES projects(id) ON DELETE SET NULL,
    thumbnail_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);
CREATE INDEX projects_bridge_type_idx ON projects(bridge_type);
CREATE INDEX projects_location_idx ON projects USING GIST(location);
CREATE INDEX projects_updated_idx ON projects(updated_at DESC);
CREATE INDEX projects_nbi_idx ON projects(nbi_number) WHERE nbi_number IS NOT NULL;

-- Analysis Runs
CREATE TABLE analyses (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    analysis_type TEXT NOT NULL,  -- influence_lines | moving_load | time_dependent | load_rating | seismic
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | running | completed | failed | cancelled
    execution_mode TEXT NOT NULL DEFAULT 'wasm',  -- wasm | server
    parameters JSONB NOT NULL DEFAULT '{}',  -- Analysis-specific parameters
    results_url TEXT,  -- S3 URL for detailed results
    results_summary JSONB,  -- Quick-access summary (max RF, critical sections)
    error_message TEXT,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    wall_time_ms INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX analyses_project_idx ON analyses(project_id);
CREATE INDEX analyses_user_idx ON analyses(user_id);
CREATE INDEX analyses_status_idx ON analyses(status);
CREATE INDEX analyses_created_idx ON analyses(created_at DESC);

-- Server Analysis Jobs (for server-side execution only)
CREATE TABLE analysis_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    analysis_id UUID NOT NULL REFERENCES analyses(id) ON DELETE CASCADE,
    worker_id TEXT,
    priority INTEGER DEFAULT 0,  -- Higher = more urgent
    progress_pct REAL DEFAULT 0.0,
    progress_message TEXT,  -- "Computing influence lines for Span 3..."
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX jobs_analysis_idx ON analysis_jobs(analysis_id);
CREATE INDEX jobs_status_idx ON analysis_jobs(worker_id);

-- Vehicle Library (HL-93, permit vehicles, state legal loads)
CREATE TABLE vehicles (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,  -- e.g., "AASHTO HL-93 Design Truck", "NYSDOT SU4"
    code TEXT NOT NULL,  -- e.g., "HL93_TRUCK", "SU4"
    jurisdiction TEXT,  -- e.g., "AASHTO", "NYSDOT", "CALTRANS"
    category TEXT NOT NULL,  -- design | legal | permit | emergency
    axle_config JSONB NOT NULL,  -- [{spacing_ft, weight_kips, width_ft}]
    lane_load_config JSONB,  -- {intensity_klf, uniform} for lane loads
    dynamic_allowance REAL DEFAULT 1.33,  -- IM factor (1 + IM/100)
    multiple_presence_factor REAL DEFAULT 1.0,
    description TEXT,
    is_builtin BOOLEAN DEFAULT true,
    uploaded_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX vehicles_category_idx ON vehicles(category);
CREATE INDEX vehicles_jurisdiction_idx ON vehicles(jurisdiction);
CREATE INDEX vehicles_code_idx ON vehicles(code);

-- Inspection Data (for future integration)
CREATE TABLE inspections (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    inspection_date DATE NOT NULL,
    inspector_name TEXT,
    nbi_condition_rating INTEGER,  -- NBI 0-9 scale
    element_states JSONB,  -- AASHTO CoRe element-level condition states
    photos JSONB DEFAULT '[]',  -- [{s3_url, element_id, caption}]
    notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX inspections_project_idx ON inspections(project_id);
CREATE INDEX inspections_date_idx ON inspections(inspection_date DESC);

-- Reports (generated PDFs)
CREATE TABLE reports (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    report_type TEXT NOT NULL,  -- design | rating | inspection | seismic
    title TEXT NOT NULL,
    s3_url TEXT NOT NULL,
    generated_by UUID NOT NULL REFERENCES users(id),
    generated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX reports_project_idx ON reports(project_id);

-- Usage Tracking (for billing enforcement)
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    record_type TEXT NOT NULL,  -- analysis_minutes | storage_bytes | report_generation
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

use chrono::{DateTime, Utc, NaiveDate};
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
    pub organization: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Project {
    pub id: Uuid,
    pub owner_id: Uuid,
    pub name: String,
    pub bridge_type: String,
    pub description: String,
    #[sqlx(skip)]  // PostGIS GEOGRAPHY handled separately
    pub location: Option<(f64, f64)>,  // (lat, lon)
    pub location_name: Option<String>,
    pub nbi_number: Option<String>,
    pub design_code: String,
    pub geometry_data: serde_json::Value,
    pub section_data: serde_json::Value,
    pub prestress_data: Option<serde_json::Value>,
    pub analysis_results: Option<serde_json::Value>,
    pub rating_results: Option<serde_json::Value>,
    pub settings: serde_json::Value,
    pub is_public: bool,
    pub forked_from: Option<Uuid>,
    pub thumbnail_url: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Analysis {
    pub id: Uuid,
    pub project_id: Uuid,
    pub user_id: Uuid,
    pub analysis_type: String,
    pub status: String,
    pub execution_mode: String,
    pub parameters: serde_json::Value,
    pub results_url: Option<String>,
    pub results_summary: Option<serde_json::Value>,
    pub error_message: Option<String>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub wall_time_ms: Option<i32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Vehicle {
    pub id: Uuid,
    pub name: String,
    pub code: String,
    pub jurisdiction: Option<String>,
    pub category: String,
    pub axle_config: serde_json::Value,  // Vec<AxleLoad>
    pub lane_load_config: Option<serde_json::Value>,
    pub dynamic_allowance: f32,
    pub multiple_presence_factor: f32,
    pub description: Option<String>,
    pub is_builtin: bool,
    pub uploaded_by: Option<Uuid>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct AxleLoad {
    pub spacing_ft: f64,  // Distance from previous axle (0 for first axle)
    pub weight_kips: f64,
    pub width_ft: f64,    // Transverse spacing (for multi-axle groups)
}

#[derive(Debug, Deserialize, Serialize)]
pub struct LaneLoadConfig {
    pub intensity_klf: f64,  // kips/linear foot
    pub uniform: bool,       // True for uniform, false for point loads
}

#[derive(Debug, FromRow, Serialize)]
pub struct Inspection {
    pub id: Uuid,
    pub project_id: Uuid,
    pub inspection_date: NaiveDate,
    pub inspector_name: Option<String>,
    pub nbi_condition_rating: Option<i16>,
    pub element_states: serde_json::Value,
    pub photos: serde_json::Value,
    pub notes: Option<String>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct BridgeGeometry {
    pub num_spans: usize,
    pub span_lengths_ft: Vec<f64>,
    pub skew_angle_deg: f64,
    pub horizontal_curve_radius_ft: Option<f64>,
    pub vertical_grade_pct: f64,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct CompositeSection {
    pub girder_type: String,         // e.g., "AASHTO_TYPE_IV", "PCI_BT_72"
    pub girder_spacing_ft: f64,
    pub num_girders: usize,
    pub deck_thickness_in: f64,
    pub haunch_depth_in: f64,
    pub fc_girder_ksi: f64,          // Concrete strength: girder
    pub fc_deck_ksi: f64,            // Concrete strength: deck
    pub effective_flange_width_in: f64,  // Per AASHTO 4.6.2.6
}

#[derive(Debug, Deserialize, Serialize)]
pub struct PrestressLayout {
    pub strand_type: String,         // "LOW_RELAXATION_270", "STRESS_RELIEVED_250"
    pub strand_diameter_in: f64,     // 0.5" or 0.6" typical
    pub num_strands: usize,
    pub strand_pattern: Vec<(f64, f64)>,  // (y_in, z_in) coordinates from girder bottom
    pub jacking_stress_ksi: f64,
    pub transfer_length_in: f64,
    pub debonded_strands: Vec<usize>,     // Strand indices that are debonded
    pub debond_length_ft: f64,
}
```

---

## Solver Architecture Deep-Dive

### Governing Equations and Discretization

BridgeWorks' core solver implements **beam finite element analysis** specialized for bridge structures. For a bridge with `n` nodes and `m` beam elements, the global stiffness equation is:

```
[K] {u} = {F}
```

Where:
- **K** (stiffness matrix): Assembled from element stiffness matrices [k_e] for each beam element
- **u** (displacement vector): Node displacements and rotations (6 DOF per node: ux, uy, uz, θx, θy, θz)
- **F** (force vector): Applied loads (dead load, live load, prestress, temperature)

**Beam element stiffness matrix** for a prismatic Euler-Bernoulli beam element with composite section properties:

```
[k_e] = [ k_axial    0           0         ]
        [ 0          k_bending_y k_couple  ]
        [ 0          k_couple    k_bending_z ]

where:
k_axial = EA/L
k_bending_y = EI_y (12/L³ for transverse deflection)
k_bending_z = EI_z (12/L³ for vertical deflection)
```

**Composite section properties** (per AASHTO 4.6.2.6):
- Transform deck and haunch concrete to girder concrete: n = E_deck / E_girder
- Effective flange width: b_eff = min(L/4, 12·t_deck + max(0.5·b_web, 6·t_deck), S)
  where L = span, t_deck = deck thickness, b_web = girder top flange width, S = girder spacing
- Composite moment of inertia: I_comp = Σ(A_i · d_i² + I_i) using parallel axis theorem

**Time-dependent prestress losses** (AASHTO 5.9.3 refined method):
```
Elastic shortening (ES): Δf_pES = (E_p / E_ci) · f_cgp
Creep loss (CR):         Δf_pCR = (E_p / E_c(t)) · (ψ(t,t₀) · f_cgp)
Shrinkage loss (SH):     Δf_pSH = E_p · ε_sh(t)
Relaxation loss (RE):    Δf_pRE = Δf_pR - (Δf_pES + Δf_pCR + Δf_pSH)

where:
ψ(t,t₀) = creep coefficient (CEB-FIP 2010 or AASHTO model)
ε_sh(t) = shrinkage strain (function of RH, V/S, time)
f_cgp = stress at centroid of strands due to prestress + self-weight
```

**Staged construction analysis**:
1. **Stage 1** (Transfer): Prestress released, girder + self-weight → compute stress, camber
2. **Stage 2** (Deck pour): Add deck weight (non-composite) → additional girder stress, deflection
3. **Stage 3** (Composite): Deck cured → composite section active → barrier, overlay, etc. applied to composite section
4. **Stage 4** (Service): Apply time-dependent losses → recompute stress at service

**Influence line computation** (Müller-Breslau principle):
- Remove restraint at section of interest (e.g., cut for moment influence, support for reaction)
- Apply unit displacement/rotation at cut
- Resulting deflected shape = influence line for that force/moment
- Implementation: solve [K] {u} = {F_unit} where F_unit is a unit force at the section

### Client/Server Split (WASM Threshold)

```
Bridge model created → Span count extracted
    │
    ├── ≤10 spans → WASM solver (browser)
    │   ├── Instant analysis, no network latency
    │   ├── Results displayed immediately
    │   └── No server cost
    │
    └── >10 spans → Server solver (Rust native)
        ├── Job queued via Redis
        ├── Worker picks up, runs native solver
        ├── Progress streamed via WebSocket
        └── Results stored in S3, URL returned
```

The 10-span threshold was chosen because:
- WASM solver with `nalgebra` handles 10-span continuous bridge (50-100 nodes, 40-80 elements) in <1 second
- 10 spans covers: simple span (1), 2-span continuous (common), 3-5 span continuous (typical highway), up to 10-span viaducts
- Above 10 spans: long viaducts, cable-stayed (100+ stays), suspension (1000+ nodes) → need server compute

### WASM Compilation Pipeline

```toml
# solver-wasm/Cargo.toml
[package]
name = "bridgeworks-solver-wasm"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
wasm-bindgen = "0.2"
serde = { version = "1", features = ["derive"] }
serde-wasm-bindgen = "0.6"
js-sys = "0.3"
nalgebra = { version = "0.33", default-features = false }
log = "0.4"
console_log = "1"

[profile.release]
opt-level = "z"       # Optimize for size
lto = true            # Link-time optimization
codegen-units = 1     # Single codegen unit for better optimization
strip = true          # Strip debug symbols
wasm-opt = true       # Run wasm-opt
```

---

## Architecture Deep-Dives

### 1. Analysis API Handler (Rust/Axum)

The primary endpoint receives an analysis request, validates the bridge geometry, decides between WASM and server execution, and for server execution enqueues a job with Redis and streams progress via WebSocket.

```rust
// src/api/handlers/analysis.rs

use axum::{
    extract::{Path, State, Json},
    http::StatusCode,
    response::IntoResponse,
};
use uuid::Uuid;
use crate::{
    db::models::{Analysis, BridgeGeometry},
    state::AppState,
    auth::Claims,
    error::ApiError,
};

#[derive(serde::Deserialize)]
pub struct CreateAnalysisRequest {
    pub analysis_type: AnalysisType,
    pub parameters: serde_json::Value,
}

#[derive(serde::Deserialize, serde::Serialize)]
#[serde(rename_all = "snake_case")]
pub enum AnalysisType {
    InfluenceLines,
    MovingLoad,
    TimeDependent,
    LoadRating,
}

pub async fn create_analysis(
    State(state): State<AppState>,
    claims: Claims,
    Path(project_id): Path<Uuid>,
    Json(req): Json<CreateAnalysisRequest>,
) -> Result<impl IntoResponse, ApiError> {
    // 1. Verify project ownership
    let project = sqlx::query_as!(
        crate::db::models::Project,
        r#"SELECT id, owner_id, name, bridge_type, description,
            ST_X(location::geometry) as "location_x?",
            ST_Y(location::geometry) as "location_y?",
            location_name, nbi_number, design_code,
            geometry_data, section_data, prestress_data,
            analysis_results, rating_results, settings,
            is_public, forked_from, thumbnail_url,
            created_at, updated_at
         FROM projects
         WHERE id = $1 AND owner_id = $2"#,
        project_id,
        claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Project not found"))?;

    // 2. Extract bridge geometry
    let geometry: BridgeGeometry = serde_json::from_value(project.geometry_data.clone())?;
    let span_count = geometry.num_spans;

    // 3. Check plan limits
    let user = sqlx::query_as!(
        crate::db::models::User,
        "SELECT * FROM users WHERE id = $1",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    if user.plan == "free" && span_count > 3 {
        return Err(ApiError::PlanLimit(
            "Free plan supports bridges up to 3 spans. Upgrade to Pro for unlimited."
        ));
    }

    // 4. Determine execution mode
    let execution_mode = if span_count <= 10 {
        "wasm"
    } else {
        "server"
    };

    // 5. Create analysis record
    let analysis = sqlx::query_as!(
        Analysis,
        r#"INSERT INTO analyses
            (project_id, user_id, analysis_type, status, execution_mode, parameters)
        VALUES ($1, $2, $3, $4, $5, $6)
        RETURNING *"#,
        project_id,
        claims.user_id,
        serde_json::to_string(&req.analysis_type)?,
        if execution_mode == "wasm" { "completed" } else { "pending" },
        execution_mode,
        req.parameters,
    )
    .fetch_one(&state.db)
    .await?;

    // 6. For server execution, enqueue job
    if execution_mode == "server" {
        let job = sqlx::query_as!(
            crate::db::models::AnalysisJob,
            r#"INSERT INTO analysis_jobs (analysis_id, priority)
            VALUES ($1, $2) RETURNING *"#,
            analysis.id,
            if user.plan == "advanced" || user.plan == "enterprise" { 10 } else { 0 },
        )
        .fetch_one(&state.db)
        .await?;

        // Publish to Redis job queue
        let mut redis = state.redis.get_multiplexed_async_connection().await?;
        redis::cmd("LPUSH")
            .arg("analysis:jobs")
            .arg(job.id.to_string())
            .query_async(&mut redis)
            .await?;
    }

    Ok((StatusCode::CREATED, Json(analysis)))
}

pub async fn get_analysis(
    State(state): State<AppState>,
    claims: Claims,
    Path((project_id, analysis_id)): Path<(Uuid, Uuid)>,
) -> Result<Json<Analysis>, ApiError> {
    let analysis = sqlx::query_as!(
        Analysis,
        r#"SELECT a.* FROM analyses a
         JOIN projects p ON a.project_id = p.id
         WHERE a.id = $1 AND a.project_id = $2 AND p.owner_id = $3"#,
        analysis_id,
        project_id,
        claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Analysis not found"))?;

    Ok(Json(analysis))
}
```

### 2. Beam FEM Solver Core (Rust — shared between WASM and native)

The core beam element solver that assembles stiffness matrices and solves the global system. This code compiles to both native (server) and WASM (browser) targets.

```rust
// solver-core/src/fem.rs

use nalgebra::{DMatrix, DVector};
use crate::bridge::{BridgeModel, BeamElement, Node, Material};

pub struct FemSystem {
    pub n_nodes: usize,
    pub n_dof: usize,              // 6 DOF per node (ux, uy, uz, θx, θy, θz)
    pub stiffness: DMatrix<f64>,   // Global stiffness matrix [K]
    pub force: DVector<f64>,       // Global force vector {F}
    pub displacement: DVector<f64>,// Solution vector {u}
    pub boundary_conditions: Vec<(usize, f64)>,  // (dof_index, prescribed_value)
}

impl FemSystem {
    pub fn new(n_nodes: usize) -> Self {
        let n_dof = n_nodes * 6;
        Self {
            n_nodes,
            n_dof,
            stiffness: DMatrix::zeros(n_dof, n_dof),
            force: DVector::zeros(n_dof),
            displacement: DVector::zeros(n_dof),
            boundary_conditions: Vec::new(),
        }
    }

    /// Assemble global stiffness matrix from element stiffness matrices
    pub fn assemble_stiffness(&mut self, bridge: &BridgeModel) {
        self.stiffness.fill(0.0);

        for element in &bridge.elements {
            let k_local = element.local_stiffness_matrix(bridge);
            let t = element.transformation_matrix(bridge);

            // Transform to global coordinates: [K_global] = [T]^T [K_local] [T]
            let k_global = t.transpose() * k_local * t;

            // Scatter into global matrix
            let dofs = element.global_dofs();
            for (i_local, i_global) in dofs.iter().enumerate() {
                for (j_local, j_global) in dofs.iter().enumerate() {
                    self.stiffness[(*i_global, *j_global)] += k_global[(i_local, j_local)];
                }
            }
        }
    }

    /// Apply boundary conditions (pinned, roller, fixed supports)
    pub fn apply_boundary_conditions(&mut self) {
        for (dof, value) in &self.boundary_conditions {
            // Penalty method: set K_ii = large_value, F_i = large_value * prescribed_value
            let penalty = 1e15;
            self.stiffness[(*dof, *dof)] = penalty;
            self.force[*dof] = penalty * value;
        }
    }

    /// Solve the system: [K]{u} = {F}
    pub fn solve(&mut self) -> Result<(), SolverError> {
        // Use Cholesky decomposition for symmetric positive-definite matrix
        let k_decomp = self.stiffness.clone().cholesky()
            .ok_or(SolverError::Singular("Stiffness matrix is singular".into()))?;

        self.displacement = k_decomp.solve(&self.force);
        Ok(())
    }

    /// Compute element forces (moment, shear, axial) from displacements
    pub fn element_forces(&self, element: &BeamElement, bridge: &BridgeModel) -> ElementForces {
        let dofs = element.global_dofs();
        let u_element = DVector::from_fn(12, |i, _| self.displacement[dofs[i]]);

        let t = element.transformation_matrix(bridge);
        let u_local = t * u_element;

        let k_local = element.local_stiffness_matrix(bridge);
        let f_local = k_local * u_local;

        ElementForces {
            axial_i: f_local[0],
            shear_y_i: f_local[1],
            shear_z_i: f_local[2],
            moment_y_i: f_local[4],
            moment_z_i: f_local[5],
            axial_j: f_local[6],
            shear_y_j: f_local[7],
            shear_z_j: f_local[8],
            moment_y_j: f_local[10],
            moment_z_j: f_local[11],
        }
    }
}

pub struct ElementForces {
    pub axial_i: f64,
    pub shear_y_i: f64,
    pub shear_z_i: f64,
    pub moment_y_i: f64,
    pub moment_z_i: f64,
    pub axial_j: f64,
    pub shear_y_j: f64,
    pub shear_z_j: f64,
    pub moment_y_j: f64,
    pub moment_z_j: f64,
}

impl BeamElement {
    /// Compute local stiffness matrix for Euler-Bernoulli beam element
    pub fn local_stiffness_matrix(&self, bridge: &BridgeModel) -> DMatrix<f64> {
        let node_i = &bridge.nodes[self.node_i];
        let node_j = &bridge.nodes[self.node_j];
        let L = node_i.distance_to(node_j);

        let E = self.material.elastic_modulus;
        let A = self.section.area;
        let I_y = self.section.moment_inertia_y;
        let I_z = self.section.moment_inertia_z;
        let G = E / (2.0 * (1.0 + self.material.poisson_ratio));
        let J = self.section.torsion_constant;

        let mut k = DMatrix::zeros(12, 12);

        // Axial stiffness
        let k_ax = E * A / L;
        k[(0, 0)] = k_ax; k[(0, 6)] = -k_ax;
        k[(6, 0)] = -k_ax; k[(6, 6)] = k_ax;

        // Bending stiffness (y-direction)
        let k_by = E * I_y;
        k[(2, 2)] = 12.0 * k_by / L.powi(3);
        k[(2, 4)] = 6.0 * k_by / L.powi(2);
        k[(2, 8)] = -12.0 * k_by / L.powi(3);
        k[(2, 10)] = 6.0 * k_by / L.powi(2);
        k[(4, 2)] = 6.0 * k_by / L.powi(2);
        k[(4, 4)] = 4.0 * k_by / L;
        k[(4, 8)] = -6.0 * k_by / L.powi(2);
        k[(4, 10)] = 2.0 * k_by / L;
        k[(8, 2)] = -12.0 * k_by / L.powi(3);
        k[(8, 4)] = -6.0 * k_by / L.powi(2);
        k[(8, 8)] = 12.0 * k_by / L.powi(3);
        k[(8, 10)] = -6.0 * k_by / L.powi(2);
        k[(10, 2)] = 6.0 * k_by / L.powi(2);
        k[(10, 4)] = 2.0 * k_by / L;
        k[(10, 8)] = -6.0 * k_by / L.powi(2);
        k[(10, 10)] = 4.0 * k_by / L;

        // Bending stiffness (z-direction) - similar pattern
        let k_bz = E * I_z;
        k[(1, 1)] = 12.0 * k_bz / L.powi(3);
        k[(1, 5)] = -6.0 * k_bz / L.powi(2);
        k[(1, 7)] = -12.0 * k_bz / L.powi(3);
        k[(1, 11)] = -6.0 * k_bz / L.powi(2);
        // ... (symmetric terms omitted for brevity)

        // Torsional stiffness
        let k_tor = G * J / L;
        k[(3, 3)] = k_tor; k[(3, 9)] = -k_tor;
        k[(9, 3)] = -k_tor; k[(9, 9)] = k_tor;

        k
    }

    /// Global DOF indices for this element (12 DOFs: 6 per node)
    pub fn global_dofs(&self) -> Vec<usize> {
        let mut dofs = Vec::with_capacity(12);
        for node_idx in [self.node_i, self.node_j] {
            for dof_local in 0..6 {
                dofs.push(node_idx * 6 + dof_local);
            }
        }
        dofs
    }

    /// Transformation matrix from local to global coordinates
    pub fn transformation_matrix(&self, bridge: &BridgeModel) -> DMatrix<f64> {
        let node_i = &bridge.nodes[self.node_i];
        let node_j = &bridge.nodes[self.node_j];

        // Compute direction cosines
        let dx = node_j.x - node_i.x;
        let dy = node_j.y - node_i.y;
        let dz = node_j.z - node_i.z;
        let L = (dx*dx + dy*dy + dz*dz).sqrt();

        let cx = dx / L;
        let cy = dy / L;
        let cz = dz / L;

        // Build 3x3 rotation matrix
        let mut lambda = DMatrix::zeros(3, 3);
        lambda[(0, 0)] = cx;
        lambda[(0, 1)] = cy;
        lambda[(0, 2)] = cz;
        // ... (complete rotation matrix based on element orientation)

        // Expand to 12x12 for 6-DOF nodes
        let mut t = DMatrix::zeros(12, 12);
        for i in 0..4 {
            for row in 0..3 {
                for col in 0..3 {
                    t[(i*3 + row, i*3 + col)] = lambda[(row, col)];
                }
            }
        }
        t
    }
}

#[derive(Debug)]
pub enum SolverError {
    Singular(String),
    Convergence(String),
    InvalidGeometry(String),
}
```

### 3. Influence Line Computation (Rust)

Computes influence lines using the Müller-Breslau principle for efficient moving load analysis.

```rust
// solver-core/src/influence.rs

use nalgebra::DVector;
use crate::fem::{FemSystem, ElementForces};
use crate::bridge::BridgeModel;

pub struct InfluenceLine {
    pub positions: Vec<f64>,      // x-coordinates along bridge
    pub ordinates: Vec<f64>,      // Influence line values at each position
    pub force_type: ForceType,    // What this influence line represents
    pub location: f64,            // Section location where force is computed
}

#[derive(Debug, Clone)]
pub enum ForceType {
    Moment,
    Shear,
    Reaction,
    Deflection,
}

impl InfluenceLine {
    /// Compute moment influence line at specified section using Müller-Breslau
    pub fn compute_moment(
        bridge: &BridgeModel,
        section_x: f64,
    ) -> Result<Self, crate::fem::SolverError> {
        // 1. Find element and local position containing section_x
        let (element_idx, xi_local) = bridge.find_element_at(section_x)?;

        // 2. Create modified system with moment release at section
        let mut fem = FemSystem::new(bridge.nodes.len());
        fem.assemble_stiffness(bridge);

        // 3. Apply unit rotation at section (Müller-Breslau)
        let node_idx = bridge.elements[element_idx].node_i;
        let rotation_dof = node_idx * 6 + 4;  // θy rotation
        fem.force[rotation_dof] = 1.0;

        // 4. Apply boundary conditions (supports)
        for support in &bridge.supports {
            for dof in support.constrained_dofs() {
                fem.boundary_conditions.push((dof, 0.0));
            }
        }
        fem.apply_boundary_conditions();

        // 5. Solve for deflected shape
        fem.solve()?;

        // 6. Extract ordinates at regular intervals
        let n_points = 100;
        let total_length = bridge.total_length();
        let mut positions = Vec::with_capacity(n_points);
        let mut ordinates = Vec::with_capacity(n_points);

        for i in 0..n_points {
            let x = (i as f64) * total_length / ((n_points - 1) as f64);
            let deflection = bridge.interpolate_displacement(&fem.displacement, x, 2)?;  // z-displacement
            positions.push(x);
            ordinates.push(deflection);
        }

        Ok(Self {
            positions,
            ordinates,
            force_type: ForceType::Moment,
            location: section_x,
        })
    }

    /// Integrate influence line with load pattern to find maximum force
    pub fn integrate_with_load(&self, load_pattern: &LoadPattern) -> f64 {
        let mut max_force = f64::NEG_INFINITY;

        // Try different positions for the load pattern
        for start_pos in self.positions.iter().step_by(5) {
            let mut force = 0.0;

            for point_load in &load_pattern.point_loads {
                let load_x = start_pos + point_load.position;
                if load_x >= 0.0 && load_x <= *self.positions.last().unwrap() {
                    let ordinate = self.interpolate_ordinate(load_x);
                    force += point_load.magnitude * ordinate;
                }
            }

            // Add uniform load contribution
            if let Some(uniform) = &load_pattern.uniform_load {
                for i in 0..self.positions.len() - 1 {
                    let x1 = self.positions[i].max(*start_pos);
                    let x2 = self.positions[i + 1].min(start_pos + load_pattern.length);

                    if x2 > x1 {
                        let avg_ordinate = (self.ordinates[i] + self.ordinates[i + 1]) / 2.0;
                        force += uniform.intensity * (x2 - x1) * avg_ordinate;
                    }
                }
            }

            max_force = max_force.max(force);
        }

        max_force
    }

    fn interpolate_ordinate(&self, x: f64) -> f64 {
        // Linear interpolation between points
        for i in 0..self.positions.len() - 1 {
            if x >= self.positions[i] && x <= self.positions[i + 1] {
                let t = (x - self.positions[i]) / (self.positions[i + 1] - self.positions[i]);
                return self.ordinates[i] * (1.0 - t) + self.ordinates[i + 1] * t;
            }
        }
        0.0
    }
}

pub struct LoadPattern {
    pub point_loads: Vec<PointLoad>,
    pub uniform_load: Option<UniformLoad>,
    pub length: f64,  // Total footprint of load pattern
}

pub struct PointLoad {
    pub position: f64,     // Position within pattern (0 = start)
    pub magnitude: f64,    // Force magnitude (kips)
}

pub struct UniformLoad {
    pub intensity: f64,    // Force per unit length (kips/ft)
}
```

### 4. Load Rating Engine (Rust)

Implements AASHTO MBE load rating factor computation for LRFR method.

```rust
// solver-core/src/rating.rs

use crate::fem::{FemSystem, ElementForces};
use crate::influence::InfluenceLine;
use crate::bridge::BridgeModel;

pub struct LoadRating {
    pub section_x: f64,
    pub rf_inventory: f64,
    pub rf_operating: f64,
    pub rf_legal: f64,
    pub controlling_vehicle: String,
    pub controlling_limit_state: LimitState,
    pub moment_capacity_kip_ft: f64,
    pub moment_dead_load_kip_ft: f64,
    pub moment_live_load_kip_ft: f64,
}

#[derive(Debug, Clone)]
pub enum LimitState {
    Strength,
    Service,
}

impl LoadRating {
    /// Compute LRFR rating factor per AASHTO MBE 6A.4.2
    /// RF = (φC - γ_DC·DC - γ_DW·DW ± γ_P·P) / (γ_L·(LL + IM))
    pub fn compute_lrfr(
        bridge: &BridgeModel,
        section_x: f64,
        vehicle_code: &str,
    ) -> Result<Self, crate::fem::SolverError> {
        // 1. Compute moment capacity (φMn)
        let section = bridge.section_at(section_x)?;
        let phi = 1.0;  // Resistance factor for prestressed concrete (AASHTO 5.5.4.2)
        let m_n = section.nominal_moment_capacity_kip_ft();
        let phi_m_n = phi * m_n;

        // 2. Compute dead load moment
        let m_dc = bridge.compute_dead_load_moment(section_x)?;  // Component dead load
        let m_dw = bridge.compute_wearing_surface_moment(section_x)?;  // Wearing surface

        // 3. Compute live load moment using influence line
        let influence = InfluenceLine::compute_moment(bridge, section_x)?;
        let vehicle = bridge.get_vehicle(vehicle_code)?;
        let m_ll_plus_im = influence.integrate_with_load(&vehicle.load_pattern()) * vehicle.dynamic_allowance;

        // 4. Load factors per MBE Table 6A.4.2.2-1
        // Inventory: γ_DC = 1.25, γ_DW = 1.50, γ_L = 1.75
        let gamma_dc_inv = 1.25;
        let gamma_dw_inv = 1.50;
        let gamma_l_inv = 1.75;

        let rf_inventory = (phi_m_n - gamma_dc_inv * m_dc - gamma_dw_inv * m_dw)
            / (gamma_l_inv * m_ll_plus_im);

        // Operating: γ_DC = 1.25, γ_DW = 1.50, γ_L = 1.35
        let gamma_l_op = 1.35;
        let rf_operating = (phi_m_n - gamma_dc_inv * m_dc - gamma_dw_inv * m_dw)
            / (gamma_l_op * m_ll_plus_im);

        // Legal (state-specific): γ_DC = 1.25, γ_DW = 1.50, γ_L = 1.80
        let gamma_l_legal = 1.80;
        let rf_legal = (phi_m_n - gamma_dc_inv * m_dc - gamma_dw_inv * m_dw)
            / (gamma_l_legal * m_ll_plus_im);

        Ok(Self {
            section_x,
            rf_inventory,
            rf_operating,
            rf_legal,
            controlling_vehicle: vehicle_code.to_string(),
            controlling_limit_state: LimitState::Strength,
            moment_capacity_kip_ft: phi_m_n,
            moment_dead_load_kip_ft: m_dc + m_dw,
            moment_live_load_kip_ft: m_ll_plus_im,
        })
    }

    /// Check if bridge requires posting (RF < 1.0)
    pub fn requires_posting(&self) -> bool {
        self.rf_inventory < 1.0 || self.rf_operating < 1.0
    }

    /// Determine safe posting load (tons) when RF < 1.0
    pub fn compute_posting_load(&self, vehicle_gross_weight_tons: f64) -> f64 {
        if self.rf_inventory >= 1.0 {
            return vehicle_gross_weight_tons;  // No posting needed
        }
        // Posted load = RF_inventory × vehicle weight (MBE 6A.8)
        self.rf_inventory * vehicle_gross_weight_tons
    }
}

pub struct RatingSummary {
    pub project_id: String,
    pub bridge_name: String,
    pub analysis_date: String,
    pub ratings: Vec<LoadRating>,
    pub critical_section: f64,
    pub min_rf_inventory: f64,
    pub min_rf_operating: f64,
}

impl RatingSummary {
    pub fn from_ratings(ratings: Vec<LoadRating>, bridge: &BridgeModel) -> Self {
        let min_rf_inv = ratings.iter()
            .map(|r| r.rf_inventory)
            .min_by(|a, b| a.partial_cmp(b).unwrap())
            .unwrap_or(0.0);

        let min_rf_op = ratings.iter()
            .map(|r| r.rf_operating)
            .min_by(|a, b| a.partial_cmp(b).unwrap())
            .unwrap_or(0.0);

        let critical = ratings.iter()
            .min_by(|a, b| a.rf_inventory.partial_cmp(&b.rf_inventory).unwrap())
            .map(|r| r.section_x)
            .unwrap_or(0.0);

        Self {
            project_id: bridge.project_id.clone(),
            bridge_name: bridge.name.clone(),
            analysis_date: chrono::Utc::now().format("%Y-%m-%d").to_string(),
            ratings,
            critical_section: critical,
            min_rf_inventory: min_rf_inv,
            min_rf_operating: min_rf_op,
        }
    }
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–5)

**Day 1: Project initialization and Rust backend scaffold**
- `cargo init bridgeworks-api` with Axum, SQLx, tokio dependencies
- `src/main.rs` — Axum app with CORS, tracing, graceful shutdown
- `src/config.rs` — Environment-based config (DATABASE_URL, JWT_SECRET, S3_BUCKET)
- `src/state.rs` — AppState struct (PgPool, Redis, S3Client)
- `src/error.rs` — ApiError enum with IntoResponse
- `Dockerfile` — Multi-stage build (builder + runtime Alpine)
- `docker-compose.yml` — PostgreSQL with PostGIS, Redis, MinIO (S3)

**Day 2: Database schema and PostGIS setup**
- `migrations/001_initial.sql` — All 8 tables: users, projects, analyses, analysis_jobs, vehicles, inspections, reports, usage_records
- PostGIS extension setup for bridge location queries
- `src/db/mod.rs` — Database pool initialization
- `src/db/models.rs` — All SQLx structs with FromRow derives
- Run `sqlx migrate run` to apply schema
- Seed script for HL-93 vehicles and AASHTO/PCI girder templates

**Day 3: Authentication system**
- `src/auth/mod.rs` — JWT token generation and validation middleware
- `src/auth/oauth.rs` — Google and GitHub OAuth 2.0 flow handlers
- `src/api/handlers/auth.rs` — Register, login, OAuth callback, refresh token, me endpoints
- Password hashing with bcrypt (cost factor 12)
- JWT with 24h access token + 30d refresh token
- Auth middleware extracts Claims from Authorization header

**Day 4: Project and vehicle CRUD**
- `src/api/handlers/projects.rs` — Create, list, get, update, delete, fork project
- `src/api/handlers/vehicles.rs` — List vehicles, get vehicle, filter by jurisdiction/category
- `src/api/router.rs` — All route definitions with auth middleware
- Project validation: span count, girder types, material properties
- Integration tests: auth flow, project CRUD, authorization checks

**Day 5: S3 integration and file upload**
- `src/services/s3.rs` — S3 client wrapper with presigned URL generation
- `src/api/handlers/upload.rs` — File upload for inspection photos
- Thumbnail generation for project cards (server-side with `image` crate)
- Test S3 uploads to MinIO in local development

### Phase 2 — Solver Core (Days 6–16)

**Day 6: FEM system framework**
- `solver-core/` — New Rust workspace member (shared between native and WASM)
- `solver-core/src/fem.rs` — FemSystem struct with stiffness assembly, solve
- `solver-core/src/bridge.rs` — BridgeModel, BeamElement, Node, Support types
- Unit tests: 2-node cantilever beam, 3-node continuous beam

**Day 7: Beam element stiffness**
- `solver-core/src/fem.rs` — Euler-Bernoulli beam element stiffness matrix (12x12)
- Axial, bending (both axes), torsion stiffness terms
- Coordinate transformation matrix (local → global)
- Tests: simple beam deflection, cantilever tip deflection vs. analytical

**Day 8: Boundary conditions and supports**
- Support types: pinned (ux=uy=uz=0), roller (uy=uz=0), fixed (all DOF=0)
- Penalty method for applying boundary conditions
- Multi-span continuous bridge support handling
- Tests: 2-span continuous beam reactions vs. hand calculation

**Day 9: Section property computation**
- `solver-core/src/section.rs` — Composite section builder
- AASHTO/PCI girder shape database (Type I-VI, BT-54, BT-72, etc.)
- Effective flange width per AASHTO 4.6.2.6
- Composite centroid, moment of inertia, section modulus
- Tests: verify composite properties for standard girder+deck combinations

**Day 10: Prestress modeling**
- `solver-core/src/prestress.rs` — Strand layout and prestress force computation
- Prestress force as equivalent loads (axial + moment at section)
- Transfer length modeling (gradual stress development)
- Tests: prestress-induced camber vs. analytical for simple span

**Day 11: Time-dependent losses**
- `solver-core/src/losses.rs` — AASHTO 5.9.3 refined and approximate methods
- Creep coefficient (CEB-FIP 2010 model, AASHTO simplified)
- Shrinkage strain (function of RH, V/S ratio, time)
- Relaxation loss (low-relaxation strand model)
- Elastic shortening at transfer
- Tests: compare losses to PCA design examples

**Day 12: Staged construction analysis**
- `solver-core/src/stages.rs` — Construction stage manager
- Stage 1: Transfer (prestress + girder self-weight, non-composite)
- Stage 2: Deck pour (deck dead load on non-composite girder)
- Stage 3: Composite (barrier, overlay on composite section)
- Stage 4: Service (with time-dependent losses)
- Tests: staged stress and camber vs. PGSuper reference

**Day 13: Influence line computation**
- `solver-core/src/influence.rs` — Müller-Breslau implementation
- Moment influence line via unit rotation
- Shear influence line via unit displacement
- Reaction influence line at supports
- Tests: simple span influence ordinate vs. analytical (M_max = PL/4)

**Day 14: HL-93 moving load optimizer**
- `solver-core/src/hl93.rs` — AASHTO HL-93 load patterns
- Design truck: 8k-32k-32k with variable 14'-30' spacing
- Design tandem: 25k-25k at 4' spacing
- Lane load: 0.64 klf uniform
- Critical positioning search using influence line integration
- Tests: verify HL-93 max moment vs. AASHTO table values

**Day 15: Load rating implementation**
- `solver-core/src/rating.rs` — LRFR rating factor computation
- Moment capacity from section properties and prestress
- Dead load + live load + impact from influence lines
- RF computation per MBE 6A.4.2 formulas
- Tests: compare RF to hand calculations for standard bridge

**Day 16: Solver validation suite**
- Benchmark simple span: 60 ft, Type IV girder, 4 strands bottom
- Benchmark 2-span continuous: 80-80 ft, BT-72, 6 strands
- Validate deflections, moments, stresses against PGSuper
- Validate HL-93 loading against AASHTO tables
- All benchmarks pass within 2% tolerance

### Phase 3 — WASM Build + Frontend Foundation (Days 17–22)

**Day 17: WASM solver compilation**
- `solver-wasm/` — New workspace member for WASM target
- `solver-wasm/Cargo.toml` — wasm-bindgen, serde-wasm-bindgen dependencies
- `solver-wasm/src/lib.rs` — WASM entry points: `analyze_bridge()`, `compute_rating()`
- `wasm-pack build --target web --release` pipeline
- Target WASM bundle <1.5MB gzipped

**Day 18: Frontend scaffold with Vite + React**
```bash
npm create vite@latest frontend -- --template react-ts
cd frontend
npm i zustand @tanstack/react-query axios three @react-three/fiber @react-three/drei d3
```
- `src/App.tsx` — Router, auth context, layout with sidebar
- `src/stores/authStore.ts` — Zustand auth state (JWT, user)
- `src/stores/projectStore.ts` — Project state management
- `src/lib/api.ts` — Axios API client with auth interceptors

**Day 19: 3D bridge visualization with Three.js**
- `src/components/BridgeViewer/BridgeViewer.tsx` — React-Three-Fiber canvas
- `src/components/BridgeViewer/Girder.tsx` — 3D girder mesh (extruded I-shape)
- `src/components/BridgeViewer/Deck.tsx` — Deck slab rendering
- `src/components/BridgeViewer/Support.tsx` — Pier and abutment visualization
- OrbitControls for pan/zoom/rotate
- Section cut view showing composite cross-section

**Day 20: Parametric bridge input form**
- `src/components/BridgeInput/GeometryForm.tsx` — Span count, lengths, alignment
- `src/components/BridgeInput/SectionForm.tsx` — Girder type selector, deck thickness, spacing
- `src/components/BridgeInput/PrestressForm.tsx` — Strand count, layout, jacking stress
- `src/components/BridgeInput/MaterialForm.tsx` — Concrete strengths (f'ci, f'c)
- Real-time 3D preview as parameters change

**Day 21: D3 diagramming components**
- `src/components/Diagrams/InfluenceLinePlot.tsx` — SVG plot of influence lines
- `src/components/Diagrams/MomentDiagram.tsx` — Moment envelope diagram
- `src/components/Diagrams/RatingChart.tsx` — Bar chart of RF values by section
- Axes with engineering notation (k-ft, kips)
- Interactive hover for value readout

**Day 22: WASM solver integration**
- `src/hooks/useWasmSolver.ts` — React hook to load WASM and run analysis
- `src/lib/wasmLoader.ts` — Lazy load WASM bundle on first analysis
- Loading states and error handling
- Progress indicator for analysis (WASM runs in <1s, but show spinner)
- Test integration: create bridge → run analysis → display results

### Phase 4 — API + Job Orchestration (Days 23–28)

**Day 23: Analysis API endpoints**
- `src/api/handlers/analysis.rs` — Create analysis, get analysis, list analyses
- Span count extraction and WASM/server routing logic
- Plan-based limits enforcement (free: 3 spans, pro: 10 spans, advanced: unlimited)
- Result caching in analysis_results JSONB field

**Day 24: Server-side analysis worker**
- `src/workers/analysis_worker.rs` — Redis job consumer, runs native solver
- `src/workers/mod.rs` — Worker pool management (configurable concurrency)
- Progress streaming: worker publishes progress → Redis pub/sub → WebSocket
- Error handling: invalid geometry, analysis failures
- S3 result upload with presigned URLs

**Day 25: WebSocket for live progress**
- `src/api/ws/mod.rs` — Axum WebSocket upgrade handler
- `src/api/ws/analysis_progress.rs` — Subscribe to analysis progress channel
- Client receives: `{ progress_pct, progress_message, current_stage }`
- Frontend: `src/hooks/useAnalysisProgress.ts` — React hook for WebSocket subscription

**Day 26: Vehicle library API**
- `src/api/handlers/vehicles.rs` — Search vehicles, get vehicle details
- Built-in vehicles: HL-93, state legal loads (NYSDOT, CALTRANS, PennDOT SU4-7)
- Custom vehicle upload for Pro+ users
- Axle configuration validation
- Tests: query by jurisdiction, category filtering

**Day 27: Report generation**
- `src/services/pdf.rs` — PDF report generator using `printpdf` or `headless_chrome`
- Design report: geometry, section properties, load summary, design checks
- Rating report: RF summary table, critical sections, load posting recommendations
- Report templates with code references (AASHTO section numbers)
- S3 upload and database record creation

**Day 28: Project management features**
- Fork project (deep copy with new owner)
- Public/private visibility toggle
- Project templates: typical 2-span, 3-span, 5-span bridges
- Auto-save: frontend debounced PATCH every 10 seconds on geometry changes
- Project search by location (PostGIS within-radius query)

### Phase 5 — Results + UI Polish (Days 29–33)

**Day 29: Analysis results display**
- `src/pages/Results.tsx` — Tabbed results view (Influence, Moments, Rating)
- Influence lines plot with HL-93 truck overlay
- Moment envelope diagram (dead load, live load, total)
- Stress distribution at critical sections
- Deflection and camber plots over time (staged construction)

**Day 30: Rating results table**
- `src/components/Results/RatingTable.tsx` — RF values by section and vehicle
- Highlight critical section (min RF) in red
- Posting load calculation display for RF < 1.0
- Export rating summary to CSV
- Color-coded RF values: green (>1.2), yellow (1.0-1.2), red (<1.0)

**Day 31: 3D visualization enhancements**
- Texture mapping: concrete girder, deck surface
- Shadow rendering for depth perception
- Exploded view showing girder, deck, bearings separately
- Section cut tool with dimension annotations
- Screenshot export (PNG) from current 3D view

**Day 32: Dashboard and project list**
- `src/pages/Dashboard.tsx` — Project cards with thumbnails
- Filter by bridge type, date range, location
- Quick actions: duplicate, delete, export
- Recent analyses list with status indicators
- Usage statistics widget (analyses run, reports generated)

**Day 33: Responsive layout and mobile support**
- Sidebar collapse on mobile, hamburger menu
- 3D viewer touch controls (pinch zoom, two-finger pan)
- Form layouts stack vertically on small screens
- Table horizontal scroll on mobile
- PWA manifest for "add to home screen"

### Phase 6 — Billing + Plan Enforcement (Days 34–37)

**Day 34: Stripe integration**
- `src/api/handlers/billing.rs` — Create checkout session, customer portal session
- `src/api/handlers/webhooks/stripe.rs` — Subscription events handler
- Plan mapping: Free (3 spans), Pro ($129/mo, 10 spans), Advanced ($299/mo, unlimited)
- Subscription status sync with database

**Day 35: Usage tracking**
- `src/middleware/plan_limits.rs` — Middleware checking span limits before analysis
- `src/services/usage.rs` — Track analysis minutes per billing period
- Usage record insertion after each server-side analysis
- Usage dashboard: current period usage, historical trends
- Approaching-limit warnings at 80% of monthly quota

**Day 36: Billing UI**
- `frontend/src/pages/Billing.tsx` — Plan comparison table
- Current plan display with usage meter
- Upgrade/downgrade flow via Stripe Customer Portal
- Upgrade prompt modals when hitting span limits
- Invoice history with download links

**Day 37: Feature gating**
- Gate custom vehicle upload behind Pro plan
- Gate report generation behind Pro plan
- Gate batch rating (multiple vehicles) behind Advanced plan
- Gate API access behind Advanced plan
- Show locked feature indicators with upgrade CTAs

### Phase 7 — Testing + Validation (Days 38–40)

**Day 38: Solver validation benchmarks**
- Benchmark 1: Simple span 60 ft Type IV — verify max moment, deflection, RF
- Benchmark 2: 2-span continuous 80-80 ft BT-72 — verify negative moment, reactions
- Benchmark 3: 3-span continuous 70-90-70 ft — verify critical section RF
- Benchmark 4: Time-dependent losses — verify against AASHTO design examples
- All benchmarks within 2% of reference values (PGSuper, hand calcs)

**Day 39: Integration testing**
- End-to-end test: create bridge → run analysis → view results → generate report
- API integration tests: auth → project CRUD → analysis → results
- WASM solver test: load in headless browser (Playwright), verify results
- WebSocket test: connect → subscribe → receive progress → completion
- Concurrent analysis test: 5 simultaneous server-side jobs

**Day 40: Performance testing**
- WASM solver benchmarks: 1-span (50ms), 3-span (150ms), 10-span (800ms)
- Server solver benchmarks: 20-span (2s), 50-span (10s)
- 3D rendering: 60 FPS with 10-span bridge, 30 FPS with 30-span
- Frontend load time: <2s on 4G, <500ms on cable
- Load testing: 20 concurrent users via k6

### Phase 8 — Deployment + Launch (Days 41–42)

**Day 41: Docker and AWS configuration**
- `Dockerfile` — Multi-stage Rust build with cargo-chef for layer caching
- `docker-compose.yml` — Full local dev stack
- AWS ECS task definitions for API and workers
- RDS PostgreSQL with PostGIS extension
- ElastiCache Redis for job queue
- S3 buckets with lifecycle policies
- CloudFront CDN for WASM bundle and frontend assets

**Day 42: Monitoring and launch**
- Prometheus metrics: analysis duration, success rate, queue depth
- Grafana dashboards: system health, user activity, solver performance
- Sentry integration for error tracking
- Structured logging (JSON) for API requests and solver events
- Security audit: JWT validation, SQL injection prevention, rate limiting
- Landing page with product overview, pricing, demo video
- Deploy to production, enable monitoring alerts, soft launch

---

## Critical Files

```
bridgeworks/
├── solver-core/                           # Shared solver library (native + WASM)
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── fem.rs                         # FEM system, stiffness assembly, solve
│   │   ├── bridge.rs                      # BridgeModel, BeamElement, Node, Support
│   │   ├── section.rs                     # Composite section properties
│   │   ├── prestress.rs                   # Strand layout, prestress forces
│   │   ├── losses.rs                      # Time-dependent prestress losses
│   │   ├── stages.rs                      # Staged construction manager
│   │   ├── influence.rs                   # Influence line computation
│   │   ├── hl93.rs                        # AASHTO HL-93 load patterns
│   │   └── rating.rs                      # LRFR rating factor computation
│   └── tests/
│       ├── benchmarks.rs                  # Solver validation benchmarks
│       └── integration.rs                 # Integration tests
│
├── solver-wasm/                           # WASM compilation target
│   ├── Cargo.toml
│   └── src/
│       └── lib.rs                         # WASM entry points (wasm-bindgen)
│
├── bridgeworks-api/                       # Rust API server
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
│   │   │   │   ├── projects.rs            # Project CRUD, fork
│   │   │   │   ├── analysis.rs            # Create/get analysis
│   │   │   │   ├── vehicles.rs            # Vehicle library
│   │   │   │   ├── upload.rs              # File upload
│   │   │   │   ├── billing.rs             # Stripe checkout/portal
│   │   │   │   └── webhooks/
│   │   │   │       └── stripe.rs          # Stripe webhook handler
│   │   │   └── ws/
│   │   │       ├── mod.rs                 # WebSocket upgrade handler
│   │   │       └── analysis_progress.rs   # Live progress streaming
│   │   ├── middleware/
│   │   │   └── plan_limits.rs             # Plan enforcement middleware
│   │   ├── services/
│   │   │   ├── s3.rs                      # S3 client helpers
│   │   │   ├── pdf.rs                     # PDF report generator
│   │   │   └── usage.rs                   # Usage tracking service
│   │   └── workers/
│   │       ├── mod.rs                     # Worker pool management
│   │       └── analysis_worker.rs         # Server-side analysis execution
│   ├── migrations/
│   │   └── 001_initial.sql                # Full database schema
│   └── tests/
│       ├── api_integration.rs             # API integration tests
│       └── analysis_e2e.rs                # End-to-end tests
│
├── frontend/                              # React frontend
│   ├── package.json
│   ├── vite.config.ts
│   ├── src/
│   │   ├── App.tsx                        # Router, providers, layout
│   │   ├── main.tsx                       # Entry point
│   │   ├── stores/
│   │   │   ├── authStore.ts               # Auth state (JWT, user)
│   │   │   └── projectStore.ts            # Project state
│   │   ├── hooks/
│   │   │   ├── useWasmSolver.ts           # WASM solver hook
│   │   │   └── useAnalysisProgress.ts     # WebSocket progress hook
│   │   ├── lib/
│   │   │   ├── api.ts                     # Axios API client
│   │   │   └── wasmLoader.ts              # WASM bundle loading
│   │   ├── pages/
│   │   │   ├── Dashboard.tsx              # Project list
│   │   │   ├── Editor.tsx                 # Bridge editor (inputs + 3D view)
│   │   │   ├── Results.tsx                # Analysis results tabs
│   │   │   ├── Billing.tsx                # Plan management
│   │   │   └── Login.tsx                  # Auth pages
│   │   ├── components/
│   │   │   ├── BridgeViewer/
│   │   │   │   ├── BridgeViewer.tsx       # Three.js canvas
│   │   │   │   ├── Girder.tsx             # 3D girder mesh
│   │   │   │   ├── Deck.tsx               # Deck slab
│   │   │   │   └── Support.tsx            # Piers/abutments
│   │   │   ├── BridgeInput/
│   │   │   │   ├── GeometryForm.tsx       # Span lengths, alignment
│   │   │   │   ├── SectionForm.tsx        # Girder type, deck
│   │   │   │   ├── PrestressForm.tsx      # Strand layout
│   │   │   │   └── MaterialForm.tsx       # Concrete strengths
│   │   │   ├── Diagrams/
│   │   │   │   ├── InfluenceLinePlot.tsx  # D3 influence line plot
│   │   │   │   ├── MomentDiagram.tsx      # Moment envelope
│   │   │   │   └── RatingChart.tsx        # RF bar chart
│   │   │   └── Results/
│   │   │       ├── RatingTable.tsx        # RF summary table
│   │   │       └── StressPlot.tsx         # Stress distribution
│   │   └── public/
│   │       └── wasm/                      # WASM solver bundle
│
├── docker-compose.yml                     # Local development stack
├── Cargo.toml                             # Workspace root
└── .github/
    └── workflows/
        ├── ci.yml                         # Test + lint on PR
        ├── wasm-build.yml                 # Build + deploy WASM bundle
        └── deploy.yml                     # Build Docker images + deploy to ECS
```

---

## Solver Validation Suite

### Benchmark 1: Simple Span Prestressed Girder (Design Check)

**Bridge:** 60 ft simple span, AASHTO Type IV girder, f'c = 5.0 ksi, f'ci = 4.0 ksi
**Prestress:** 10 strands (0.5" dia, 270 ksi low-relax), 8" eccentricity at midspan
**Loading:** Girder self-weight (0.82 klf) + 8" deck (0.98 klf) + barriers (0.40 klf)

**Expected at midspan (transfer):**
- Moment due to self-weight: M_g = (0.82 × 60²) / 8 = **369 k-ft**
- Prestress force (after ES): P_e = 10 × 0.153 in² × 202 ksi = **309 kips**
- Bottom fiber stress: f_bot = -P_e/A + P_e·e·c_bot/I + M_g·c_bot/I = **+0.50 ksi** (tension < 3√f'ci = 0.60 ksi OK)

**Expected at midspan (service):**
- Total prestress loss (approx method): Δf_pT = 35 ksi → P_eff = 10 × 0.153 × 167 = **256 kips**
- Total moment: M_total = 369 + 490 + 200 = **1059 k-ft**
- Bottom fiber stress (composite): f_bot = -P_eff/A + P_eff·e·c_bot/I + M_total·c_bot/I_comp = **+0.25 ksi** (OK)

**Tolerance:** Stresses < 5%, losses < 10%, deflections < 5%

### Benchmark 2: HL-93 Maximum Moment (Influence Line)

**Bridge:** 80 ft simple span, influence line ordinate at midspan = 20 ft (triangular)
**Loading:** HL-93 design truck (32k-32k-8k with 14' min spacing)
**Expected:** Position truck with 32k axles at x=40±7 ft (symmetric about midspan)

**Calculation:**
- M_LL = 32k × (20 ft) + 32k × (20 - 7×(14/80)×20) + 8k × (20 - 21×(14/80)×20) = 32×20 + 32×17.5 + 8×12.95 = 640 + 560 + 103.6 = **1304 k-ft**
- With IM = 1.33: M_LL+IM = 1.33 × 1304 = **1734 k-ft**

**Expected per AASHTO Table 3.6.1.2.2-1:** Simple span 80 ft, HL-93 max M = **1756 k-ft** (truck + lane)

**Tolerance:** Moment < 2% of AASHTO table value

### Benchmark 3: Load Rating Factor (LRFR Inventory)

**Bridge:** 2-span continuous (80-80 ft), BT-72 girder, composite section
**Section:** Midspan of first span (x = 40 ft)
**Given:**
- Nominal moment capacity: M_n = 4500 k-ft
- Dead load moment: M_DC = 800 k-ft, M_DW = 150 k-ft
- Live load moment (HL-93 + IM): M_LL = 1500 k-ft

**Calculation (LRFR Inventory):**
```
RF = (φM_n - γ_DC·M_DC - γ_DW·M_DW) / (γ_L·M_LL)
   = (1.0×4500 - 1.25×800 - 1.50×150) / (1.75×1500)
   = (4500 - 1000 - 225) / 2625
   = 3275 / 2625
   = 1.25
```

**Expected:** RF_inventory = **1.25** (adequate capacity, no posting required)

**Tolerance:** RF < 2%

### Benchmark 4: Time-Dependent Prestress Losses (Refined Method)

**Bridge:** Simple span 70 ft, Type VI girder
**Prestress:** 20 strands (0.6" dia), f_pj = 202.5 ksi (0.75 f_pu)
**Materials:** f'ci = 5.0 ksi, f'c = 7.0 ksi, RH = 70%

**Expected losses (AASHTO 5.9.3 refined method at 20 years):**
- Elastic shortening: Δf_pES = **15 ksi**
- Creep: Δf_pCR = **18 ksi** (based on CEB-FIP creep coefficient ψ(20yr) = 2.0)
- Shrinkage: Δf_pSH = **10 ksi** (ε_sh = 350×10⁻⁶)
- Relaxation: Δf_pRE = **8 ksi**
- **Total loss: Δf_pT = 51 ksi**

**Final stress:** f_pe = 202.5 - 51 = **151.5 ksi** (0.56 f_pu)

**Tolerance:** Individual losses < 10%, total loss < 8%

### Benchmark 5: Staged Construction Camber

**Bridge:** Simple span 90 ft, Type V girder, 12 strands bottom
**Expected camber:**
- At release: δ_release = **+2.8 inches** (upward, due to prestress - self-weight)
- After deck pour: δ_deck = **+1.5 inches** (reduced, deck weight causes downward deflection)
- At service (20 years): δ_service = **+1.0 inches** (further reduced due to creep/losses)

**Tolerance:** Camber values < 10%

---

## Verification

### Integration Testing Checklist

1. **Auth flow** — Register → login → JWT issued → authorized API calls → token refresh
2. **Project CRUD** — Create bridge → update geometry → save → reload → verify state preserved
3. **WASM analysis** — 3-span bridge → WASM solver → results returned in <1s → display results
4. **Server analysis** — 15-span bridge → job queued → worker picks up → WebSocket progress → results in S3
5. **3D visualization** — Bridge renders in Three.js → orbit controls work → section cut shows composite
6. **Influence lines** — Compute influence lines → plot in D3 → verify ordinates match analytical
7. **HL-93 loading** — Run moving load analysis → verify critical position → max moment matches AASHTO
8. **Load rating** — Compute RF → verify against hand calculation → display in rating table
9. **Report generation** — Generate PDF report → verify content → download from S3
10. **Plan limits** — Free user → 5-span bridge → blocked with upgrade prompt
11. **Billing** — Subscribe to Pro → Stripe checkout → webhook → plan updated → limits lifted
12. **Concurrent analyses** — 5 users simultaneously running server analyses → all complete correctly
13. **Vehicle library** — Search "HL-93" → results returned → filter by jurisdiction
14. **Template usage** — Select "3-span continuous" template → project created → analyze → expected RF
15. **Error handling** — Invalid geometry (negative span) → validation error → no crash

### SQL Verification Queries

```sql
-- 1. Analysis success rate and throughput
SELECT
    execution_mode,
    status,
    COUNT(*) as count,
    AVG(wall_time_ms)::int as avg_time_ms,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY wall_time_ms)::int as p95_time_ms
FROM analyses
WHERE created_at >= NOW() - INTERVAL '7 days'
GROUP BY execution_mode, status;

-- 2. Bridge type distribution
SELECT bridge_type, COUNT(*) as count,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 1) as pct
FROM projects
WHERE created_at >= NOW() - INTERVAL '30 days'
GROUP BY bridge_type ORDER BY count DESC;

-- 3. User plan distribution
SELECT plan, COUNT(*) as users,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 1) as pct
FROM users GROUP BY plan ORDER BY users DESC;

-- 4. Rating factor statistics
SELECT
    CASE
        WHEN (rating_results->>'min_rf_inventory')::numeric >= 1.2 THEN 'Adequate (≥1.2)'
        WHEN (rating_results->>'min_rf_inventory')::numeric >= 1.0 THEN 'Marginal (1.0-1.2)'
        ELSE 'Posting Required (<1.0)'
    END as rf_category,
    COUNT(*) as bridge_count
FROM projects
WHERE rating_results IS NOT NULL
GROUP BY rf_category
ORDER BY bridge_count DESC;

-- 5. Most popular girder types
SELECT
    section_data->>'girder_type' as girder_type,
    COUNT(*) as usage_count
FROM projects
WHERE section_data->>'girder_type' IS NOT NULL
GROUP BY girder_type
ORDER BY usage_count DESC
LIMIT 10;
```

### Performance Benchmarks

| Operation | Target | Method |
|-----------|--------|--------|
| WASM solver: 1-span analysis | <100ms | Browser benchmark (Chrome DevTools) |
| WASM solver: 3-span analysis | <200ms | Browser benchmark |
| WASM solver: 10-span analysis | <1s | Browser benchmark |
| Server solver: 20-span analysis | <3s | Server timing, 4 cores |
| Server solver: 50-span analysis | <15s | Server timing, 8 cores |
| 3D rendering: 5-span bridge | 60 FPS | Chrome FPS counter during orbit |
| 3D rendering: 20-span bridge | 30 FPS | Chrome FPS counter during orbit |
| Influence line computation (10-span) | <500ms | WASM timing |
| Load rating (10 sections, 3 vehicles) | <200ms | WASM timing |
| PDF report generation | <2s | Server-side timing |
| API: create analysis | <150ms | p95 latency (Prometheus) |
| WASM bundle load (cached) | <300ms | Service worker cache hit |
| WASM bundle load (cold) | <2s | CDN delivery, 1.2MB gzipped |

---

## Deployment Architecture

### Infrastructure

```
┌─────────────────────────────────────────────────┐
│                  CloudFront CDN                   │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────┐ │
│  │ Frontend SPA │  │ WASM Bundle │  │ Assets   │ │
│  │ (HTML/JS/CSS)│  │ (versioned) │  │ (images) │ │
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
│ ECS Task ×3  │  │              │  │              │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘
       │                 │                 │
       └─────────────────┼─────────────────┘
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ PostgreSQL   │ │ Redis        │ │ S3 Bucket    │
│ (RDS r6g.lg) │ │ (ElastiCache)│ │ (Reports +   │
│ + PostGIS    │ │ Job queue    │ │  Photos)     │
└──────────────┘ └──────┬───────┘ └──────────────┘
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ Worker       │ │ Worker       │ │ Worker       │
│ (Rust native)│ │ (Rust native)│ │ (Rust native)│
│ c6g.xlarge   │ │ c6g.xlarge   │ │ c6g.xlarge   │
│ HPA: 2-10    │ │              │ │              │
└──────────────┘ └──────────────┘ └──────────────┘
```

### Scaling Strategy

- **API servers**: ECS Service with target tracking at 70% CPU, min 3, max 10
- **Analysis workers**: ECS Service with custom metric scaling based on Redis queue depth — scale up when queue > 3 jobs, scale down when idle 10 min
- **Worker instances**: AWS c6g.xlarge (4 vCPU, 8GB RAM) for cost-efficient compute
- **Database**: RDS PostgreSQL r6g.large with PostGIS, automated backups, read replicas for reporting
- **Redis**: ElastiCache r6g.large with failover replica for high availability

### S3 Lifecycle Policy

```json
{
  "Rules": [
    {
      "ID": "analysis-results-lifecycle",
      "Prefix": "results/",
      "Status": "Enabled",
      "Transitions": [
        { "Days": 30, "StorageClass": "STANDARD_IA" },
        { "Days": 180, "StorageClass": "GLACIER" }
      ],
      "Expiration": { "Days": 730 }
    },
    {
      "ID": "reports-keep",
      "Prefix": "reports/",
      "Status": "Enabled",
      "Transitions": [
        { "Days": 90, "StorageClass": "STANDARD_IA" }
      ]
    },
    {
      "ID": "inspection-photos-keep",
      "Prefix": "inspections/",
      "Status": "Enabled",
      "Transitions": [
        { "Days": 180, "StorageClass": "STANDARD_IA" }
      ]
    }
  ]
}
```

---

## Post-MVP Roadmap

| Feature | Description | Priority |
|---------|-------------|----------|
| Steel Bridge Design | Plate girder design with constructibility checks, fatigue analysis, curved girder V-load method, composite action with shear connector design per AASHTO 6.10 | High |
| 3D Refined Analysis | Shell/solid element models for complex bridges (curved, skewed >30°, integral abutments), nonlinear material models for existing bridge evaluation | High |
| Inspection Integration | NBI data import, element-level condition states per AASHTO CoRe, photo annotation linked to structural model, condition-adjusted load rating with section loss | High |
| Seismic Analysis | Spectral analysis per AASHTO Guide Spec, pushover analysis with plastic hinges, displacement-based design, seismic isolation bearing sizing (LRB, FPS) | Medium |
| Batch Rating | Rate entire DOT bridge inventory (1000+ structures) against new permit vehicle configurations in parallel using worker fleet, generate comparison reports | Medium |
| Eurocode Support | Eurocode 2 (concrete bridges) and Eurocode 3 (steel bridges) design checks, LM1/LM2/LM3 load models, UK BD standards (BD 21, BD 37) for UK agencies | Medium |
| Cable-Stayed Design | Cable element modeling with prestress optimization, pylon design, stay cable fatigue analysis, construction stage analysis with balanced cantilever method | Low |
| BIM Integration | IFC bridge extension import/export for interoperability with Civil 3D, Bentley OpenBridge, MIDAS Civil, automatically generate bridge geometry from IFC alignment | Low |
| Machine Learning Deterioration | Predict bridge condition trajectories using historical NBI data and ML (LSTM, random forest), recommend optimal maintenance timing and budget allocation | Low |

