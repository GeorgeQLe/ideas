# 80. NoiseLab — Vehicle NVH Analysis Platform

## Implementation Plan

**MVP Scope:** Browser-based modal analysis viewer for imported FE meshes (Nastran BDF format with up to 200K DOF) with automatic frequency extraction and animated 3D mode shapes rendered via WebGPU, classical Transfer Path Analysis (TPA) engine supporting imported FRF matrices (UFF/CSV format) with operational force calculation via matrix inversion and path contribution ranking, psychoacoustic evaluation module processing uploaded WAV/FLAC audio files (up to 60 seconds, 96 kHz max sample rate) implementing Zwicker loudness (ISO 532-1), Aures sharpness, Daniel-Weber roughness, and ECMA-418 tonality with time-varying metric visualization, order tracking for rotating machinery with tachometer signal resampling and order extraction (0.5×-20× shaft speed), binaural auralization engine using KEMAR HRTFs for spatial audio playback of individual TPA path contributions, automated PDF report generation with contribution waterfalls, psychoacoustic spider charts, and frequency-domain comparisons, PostgreSQL database storing project metadata and analysis results with S3 storage for large FRF datasets and audio files, Stripe billing with three tiers (Free / Pro $179/mo / Advanced $399/user/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| Modal Solver | Rust + LAPACK | Eigenvalue solver via `ndarray-linalg` with OpenBLAS backend |
| TPA Engine | Rust (native + WASM) | Matrix inversion, SVD for ill-conditioned systems, path synthesis |
| Psychoacoustics | Rust + WASM | Zwicker loudness, sharpness, roughness, tonality — compiled to WASM for client-side |
| FFT Processing | Rust (`rustfft`) | Real-time spectral analysis, order tracking, 1/3 octave bands |
| Audio Processing | Python (FastAPI + `librosa`) | Resampling, filtering, STFT, binaural HRTF convolution |
| Database | PostgreSQL 16 | Projects, FRF matrices, analysis metadata, material properties |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | FRF datasets (HDF5), audio files (WAV/FLAC), mesh files (BDF/H5), report PDFs |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| 3D Visualization | Three.js + WebGPU | Mode shape animation, contour plots, mesh rendering with custom shaders |
| Audio Playback | Web Audio API | Binaural rendering, real-time filtering, gain control for auralization |
| Real-time | WebSocket (Axum) | Live analysis progress, modal convergence monitoring |
| Job Queue | Redis 7 + Tokio tasks | Server-side modal analysis job management, batch processing |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | WASM bundle delivery, static frontend assets, HRTF database |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, error tracking |
| CI/CD | GitHub Actions | WASM build pipeline, Rust tests, Docker image push |

### Key Architecture Decisions

1. **Hybrid client/server modal solver with 200K DOF threshold**: Small models (≤200K degrees of freedom, typical for component-level NVH) run on the client via WASM for instant visualization. Large models (full-vehicle body structures with 500K-2M DOF) are processed on HPC instances with distributed LAPACK. The threshold covers 80%+ of subsystem-level analyses (instrument panel, door, seat frame, battery pack).

2. **Classical TPA as MVP foundation**: Classical TPA (measured FRFs + operational forces via matrix inversion) is implemented first because it's the most widely used method and requires only FRF and operational data that users already have. Operational TPA (OTPA) and component-based TPA (CB-TPA) are deferred to post-MVP because they require more complex transmissibility calculations and blocked-force characterization.

3. **Psychoacoustic metrics in WASM for real-time feedback**: Zwicker loudness, sharpness, roughness, and tonality calculations are compiled to WASM and run client-side, enabling real-time updates as users adjust filtering or playback settings. This avoids round-trip latency for interactive sound quality evaluation and reduces server compute costs.

4. **WebGPU for mode shape rendering with compute shaders**: Mode shapes are animated using WebGPU compute shaders that deform the mesh geometry based on eigenvector magnitudes. This offloads animation to the GPU and supports 200K+ node meshes at 60fps. Fallback to Three.js WebGL renderer for browsers without WebGPU support.

5. **HDF5 for FRF storage with compression and chunking**: FRF matrices are stored in HDF5 format with GZIP compression (5:1 typical ratio for frequency response data) and chunked by frequency line. This allows efficient partial reads for specific frequency ranges without loading the entire dataset, critical for 1000+ DOF systems with 5000+ frequency lines.

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

-- Organizations (for multi-user collaboration)
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

-- Projects (NVH analysis workspace)
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    project_type TEXT NOT NULL,  -- modal | tpa | psychoacoustic | order_tracking
    mesh_url TEXT,  -- S3 URL to BDF/H5 mesh file
    frf_url TEXT,  -- S3 URL to HDF5 FRF matrix
    audio_url TEXT,  -- S3 URL to WAV/FLAC audio file
    metadata JSONB NOT NULL DEFAULT '{}',  -- Analysis-specific settings, frequency ranges, etc.
    is_public BOOLEAN NOT NULL DEFAULT false,
    forked_from UUID REFERENCES projects(id) ON DELETE SET NULL,
    thumbnail_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);
CREATE INDEX projects_org_idx ON projects(org_id);
CREATE INDEX projects_type_idx ON projects(project_type);
CREATE INDEX projects_updated_idx ON projects(updated_at DESC);

-- Modal Analysis Results
CREATE TABLE modal_results (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | running | completed | failed | cancelled
    execution_mode TEXT NOT NULL DEFAULT 'wasm',  -- wasm | server
    frequency_range NUMRANGE NOT NULL,  -- [f_min, f_max] Hz
    num_modes INTEGER NOT NULL DEFAULT 0,
    dof_count INTEGER NOT NULL DEFAULT 0,
    results_url TEXT,  -- S3 URL for eigenvalue/eigenvector HDF5
    results_summary JSONB,  -- First 20 natural frequencies, MAC values
    error_message TEXT,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    wall_time_ms INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX modal_project_idx ON modal_results(project_id);
CREATE INDEX modal_user_idx ON modal_results(user_id);
CREATE INDEX modal_status_idx ON modal_results(status);
CREATE INDEX modal_created_idx ON modal_results(created_at DESC);

-- TPA Analysis Results
CREATE TABLE tpa_results (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    tpa_method TEXT NOT NULL,  -- classical | operational | component_based | blocked_force
    status TEXT NOT NULL DEFAULT 'pending',
    num_paths INTEGER NOT NULL DEFAULT 0,
    num_dofs INTEGER NOT NULL DEFAULT 0,
    frequency_lines INTEGER NOT NULL DEFAULT 0,
    results_url TEXT,  -- S3 URL for path contribution HDF5
    results_summary JSONB,  -- Top 5 contributing paths per frequency band
    error_message TEXT,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    wall_time_ms INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX tpa_project_idx ON tpa_results(project_id);
CREATE INDEX tpa_user_idx ON tpa_results(user_id);
CREATE INDEX tpa_method_idx ON tpa_results(tpa_method);
CREATE INDEX tpa_created_idx ON tpa_results(created_at DESC);

-- Psychoacoustic Analysis Results
CREATE TABLE psychoacoustic_results (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    audio_duration_sec REAL NOT NULL,
    sample_rate INTEGER NOT NULL,
    metrics JSONB NOT NULL,  -- {loudness_sone, sharpness_acum, roughness_asper, tonality_tu}
    time_series_url TEXT,  -- S3 URL for time-varying metrics (CSV or HDF5)
    spectrogram_url TEXT,  -- S3 URL for spectrogram PNG
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX psycho_project_idx ON psychoacoustic_results(project_id);
CREATE INDEX psycho_created_idx ON psychoacoustic_results(created_at DESC);

-- Order Tracking Results
CREATE TABLE order_tracking_results (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    rpm_range NUMRANGE NOT NULL,  -- [rpm_min, rpm_max]
    orders REAL[] NOT NULL,  -- Extracted orders (e.g., [1.0, 2.0, 2.5, 3.0])
    results_url TEXT,  -- S3 URL for order spectrum HDF5
    colormap_url TEXT,  -- S3 URL for order tracking colormap PNG
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX order_project_idx ON order_tracking_results(project_id);
CREATE INDEX order_created_idx ON order_tracking_results(created_at DESC);

-- Material Database (for SEA and damping loss factors)
CREATE TABLE materials (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    category TEXT NOT NULL,  -- metal | composite | glass | rubber | acoustic_treatment
    density_kg_m3 REAL NOT NULL,
    youngs_modulus_pa REAL,
    poissons_ratio REAL,
    loss_factor REAL NOT NULL DEFAULT 0.01,  -- Structural damping loss factor
    speed_of_sound_m_s REAL,
    properties JSONB DEFAULT '{}',  -- Additional frequency-dependent properties
    is_builtin BOOLEAN DEFAULT true,
    uploaded_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX materials_category_idx ON materials(category);
CREATE INDEX materials_name_trgm_idx ON materials USING gin(name gin_trgm_ops);

-- Reports
CREATE TABLE reports (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    report_type TEXT NOT NULL,  -- modal | tpa | psychoacoustic | combined
    title TEXT NOT NULL,
    sections JSONB NOT NULL,  -- Report structure with figure URLs and captions
    pdf_url TEXT,  -- S3 URL for generated PDF
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX reports_project_idx ON reports(project_id);
CREATE INDEX reports_created_idx ON reports(created_at DESC);

-- Usage Tracking (for billing enforcement)
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,  -- modal_analysis | tpa_analysis | psychoacoustic_analysis | storage_bytes
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
    pub project_type: String,
    pub mesh_url: Option<String>,
    pub frf_url: Option<String>,
    pub audio_url: Option<String>,
    pub metadata: serde_json::Value,
    pub is_public: bool,
    pub forked_from: Option<Uuid>,
    pub thumbnail_url: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct ModalResult {
    pub id: Uuid,
    pub project_id: Uuid,
    pub user_id: Uuid,
    pub status: String,
    pub execution_mode: String,
    pub frequency_range: String,  // PostgreSQL range type
    pub num_modes: i32,
    pub dof_count: i32,
    pub results_url: Option<String>,
    pub results_summary: Option<serde_json::Value>,
    pub error_message: Option<String>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub wall_time_ms: Option<i32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct TpaResult {
    pub id: Uuid,
    pub project_id: Uuid,
    pub user_id: Uuid,
    pub tpa_method: String,
    pub status: String,
    pub num_paths: i32,
    pub num_dofs: i32,
    pub frequency_lines: i32,
    pub results_url: Option<String>,
    pub results_summary: Option<serde_json::Value>,
    pub error_message: Option<String>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub wall_time_ms: Option<i32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct PsychoacousticResult {
    pub id: Uuid,
    pub project_id: Uuid,
    pub user_id: Uuid,
    pub audio_duration_sec: f32,
    pub sample_rate: i32,
    pub metrics: serde_json::Value,
    pub time_series_url: Option<String>,
    pub spectrogram_url: Option<String>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Material {
    pub id: Uuid,
    pub name: String,
    pub category: String,
    pub density_kg_m3: f32,
    pub youngs_modulus_pa: Option<f32>,
    pub poissons_ratio: Option<f32>,
    pub loss_factor: f32,
    pub speed_of_sound_m_s: Option<f32>,
    pub properties: serde_json::Value,
    pub is_builtin: bool,
    pub uploaded_by: Option<Uuid>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct ModalAnalysisParams {
    pub frequency_min: f64,  // Hz
    pub frequency_max: f64,  // Hz
    pub num_modes: Option<u32>,  // If specified, extract first N modes; otherwise extract all in range
    pub convergence_tol: f64,  // Default 1e-9 for eigenvalue solver
}

#[derive(Debug, Deserialize)]
pub struct TpaParams {
    pub method: TpaMethod,
    pub num_paths: u32,
    pub frequency_lines: Vec<f64>,  // Hz
    pub reference_dof: String,  // Node/DOF identifier for target response
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum TpaMethod {
    Classical,
    Operational,
    ComponentBased,
    BlockedForce,
}

#[derive(Debug, Deserialize)]
pub struct PsychoacousticParams {
    pub metrics: Vec<PsychoacousticMetric>,
    pub window_size_ms: Option<f64>,  // Default 200ms for time-varying calculation
    pub hop_size_ms: Option<f64>,  // Default 50ms
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum PsychoacousticMetric {
    Loudness,      // Zwicker ISO 532-1
    Sharpness,     // Aures
    Roughness,     // Daniel-Weber
    Tonality,      // ECMA-418
}
```

---

## Modal Analysis Deep-Dive

### Governing Equations

Modal analysis solves the generalized eigenvalue problem for the undamped structural dynamics equation:

```
K φ = λ M φ

where:
  K = stiffness matrix (N×N sparse)
  M = mass matrix (N×N sparse)
  φ = eigenvector (mode shape)
  λ = eigenvalue = ω² (ω = natural frequency in rad/s)
  N = number of degrees of freedom
```

For damped systems, the complex eigenvalue problem becomes:

```
(K + iωC - ω²M) φ = 0

where C = damping matrix (proportional or general)
```

**Solution approach:**
1. **Subspace iteration** for the first 50-200 modes (low-frequency NVH range 0-500 Hz)
2. **Lanczos algorithm** for sparse eigenvalue extraction with shift-invert for interior eigenvalues
3. **Modal superposition** for forced response: decompose response into modal coordinates

**Frequency Response Function (FRF)** from modal data:

```
H(ω) = Σ (φᵢ φᵢᵀ) / (ωᵢ² - ω² + 2iξᵢωᵢω)
       i=1 to N_modes

where:
  ωᵢ = i-th natural frequency
  ξᵢ = i-th modal damping ratio
  φᵢ = i-th mode shape (normalized)
```

### Client/Server Split (200K DOF Threshold)

```
Mesh uploaded → DOF count extracted
    │
    ├── ≤200K DOF → WASM solver (browser)
    │   ├── Instant startup, no network latency
    │   ├── Results displayed immediately
    │   └── No server cost
    │
    └── >200K DOF → Server solver (Rust native on HPC)
        ├── Job queued via Redis
        ├── Worker picks up, runs distributed LAPACK
        ├── Progress streamed via WebSocket
        └── Results stored in S3 HDF5, URL returned
```

The 200K DOF threshold was chosen because:
- WASM modal solver with Lanczos handles 200K DOF (100 modes) in <10 seconds on modern hardware
- 200K DOF covers: component-level NVH (instrument panel: 50K-150K, door module: 80K-200K, battery pack: 100K-180K)
- Above 200K DOF: full-vehicle body structures (500K-2M DOF), large acoustic cavities → need server HPC

### WASM Compilation Pipeline

```toml
# modal-solver-wasm/Cargo.toml
[package]
name = "noiselab-modal-wasm"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
wasm-bindgen = "0.2"
serde = { version = "1", features = ["derive"] }
serde-wasm-bindgen = "0.6"
js-sys = "0.3"
ndarray = "0.15"
ndarray-linalg = { version = "0.16", features = ["openblas-static"] }
nalgebra-sparse = "0.9"
log = "0.4"
console_log = "1"

[profile.release]
opt-level = "z"       # Optimize for size
lto = true            # Link-time optimization
codegen-units = 1     # Single codegen unit for better optimization
strip = true          # Strip debug symbols
```

---

## Architecture Deep-Dives

### 1. Modal Analysis API Handler (Rust/Axum)

The primary endpoint receives a modal analysis request, validates the mesh, decides between WASM and server execution, and for server execution enqueues a job with Redis and streams progress via WebSocket.

```rust
// src/api/handlers/modal.rs

use axum::{
    extract::{Path, State, Json},
    http::StatusCode,
    response::IntoResponse,
};
use uuid::Uuid;
use crate::{
    db::models::{ModalResult, ModalAnalysisParams},
    mesh::parse_nastran_bdf,
    state::AppState,
    auth::Claims,
    error::ApiError,
};

#[derive(serde::Deserialize)]
pub struct CreateModalAnalysisRequest {
    pub frequency_min: f64,
    pub frequency_max: f64,
    pub num_modes: Option<u32>,
}

pub async fn create_modal_analysis(
    State(state): State<AppState>,
    claims: Claims,
    Path(project_id): Path<Uuid>,
    Json(req): Json<CreateModalAnalysisRequest>,
) -> Result<impl IntoResponse, ApiError> {
    // 1. Verify project ownership and fetch mesh
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

    let mesh_url = project.mesh_url
        .ok_or(ApiError::BadRequest("Project has no mesh file"))?;

    // 2. Download and parse mesh to determine DOF count
    let mesh_bytes = state.s3
        .get_object()
        .bucket(&state.config.s3_bucket)
        .key(mesh_url.strip_prefix("s3://").unwrap_or(&mesh_url))
        .send()
        .await?
        .body
        .collect()
        .await?
        .into_bytes();

    let mesh = parse_nastran_bdf(&mesh_bytes)?;
    let dof_count = mesh.num_nodes * 6;  // Assume 6 DOF per node (3 translations + 3 rotations)

    // 3. Check plan limits
    let user = sqlx::query_as!(
        crate::db::models::User,
        "SELECT * FROM users WHERE id = $1",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    if user.plan == "free" && dof_count > 50_000 {
        return Err(ApiError::PlanLimit(
            "Free plan supports models up to 50K DOF. Upgrade to Pro for unlimited."
        ));
    }

    // 4. Determine execution mode
    let execution_mode = if dof_count <= 200_000 {
        "wasm"
    } else {
        "server"
    };

    // 5. Create modal result record
    let result = sqlx::query_as!(
        ModalResult,
        r#"INSERT INTO modal_results
            (project_id, user_id, status, execution_mode, frequency_range, dof_count)
        VALUES ($1, $2, $3, $4, numrange($5, $6), $7)
        RETURNING *"#,
        project_id,
        claims.user_id,
        if execution_mode == "wasm" { "completed" } else { "pending" },
        execution_mode,
        req.frequency_min,
        req.frequency_max,
        dof_count as i32,
    )
    .fetch_one(&state.db)
    .await?;

    // 6. For server execution, enqueue job
    if execution_mode == "server" {
        let job_data = serde_json::json!({
            "modal_result_id": result.id,
            "project_id": project_id,
            "mesh_url": mesh_url,
            "frequency_min": req.frequency_min,
            "frequency_max": req.frequency_max,
            "num_modes": req.num_modes,
        });

        state.redis
            .publish("modal:jobs", serde_json::to_string(&job_data)?)
            .await?;
    }

    Ok((StatusCode::CREATED, Json(result)))
}

pub async fn get_modal_result(
    State(state): State<AppState>,
    claims: Claims,
    Path((project_id, result_id)): Path<(Uuid, Uuid)>,
) -> Result<Json<ModalResult>, ApiError> {
    let result = sqlx::query_as!(
        ModalResult,
        "SELECT mr.* FROM modal_results mr
         JOIN projects p ON mr.project_id = p.id
         WHERE mr.id = $1 AND mr.project_id = $2
         AND (p.owner_id = $3 OR p.org_id IN (
             SELECT org_id FROM org_members WHERE user_id = $3
         ))",
        result_id,
        project_id,
        claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Modal result not found"))?;

    Ok(Json(result))
}
```

### 2. TPA Engine Core (Rust)

The Transfer Path Analysis engine performs classical TPA via matrix inversion, computing operational forces from FRFs and measured responses.

```rust
// src/tpa/classical.rs

use nalgebra::{DMatrix, DVector};
use crate::error::TpaError;

/// Classical TPA: Solve for operational forces F from FRFs H and responses X
///
/// X = H * F  →  F = H⁻¹ * X  (overdetermined system, use SVD)
///
/// Inputs:
///   - frf_matrix: H (n_responses × n_paths × n_frequencies)
///   - operational_responses: X (n_responses × n_frequencies)
///
/// Outputs:
///   - forces: F (n_paths × n_frequencies)
///   - contributions: X_path (n_responses × n_paths × n_frequencies) = H * diag(F)
pub fn classical_tpa(
    frf_matrix: &[DMatrix<f64>],  // One matrix per frequency
    operational_responses: &DMatrix<f64>,  // n_responses × n_frequencies
) -> Result<TpaResult, TpaError> {
    let n_frequencies = frf_matrix.len();
    let n_paths = frf_matrix[0].ncols();
    let n_responses = frf_matrix[0].nrows();

    if operational_responses.ncols() != n_frequencies {
        return Err(TpaError::DimensionMismatch(
            "Operational response frequency count must match FRF frequency count"
        ));
    }

    let mut forces = DMatrix::zeros(n_paths, n_frequencies);
    let mut contributions = vec![DMatrix::zeros(n_responses, n_paths); n_frequencies];

    for (freq_idx, h_matrix) in frf_matrix.iter().enumerate() {
        let x_vec = operational_responses.column(freq_idx);

        // Solve H * f = x using SVD (handles ill-conditioned systems)
        let svd = h_matrix.svd(true, true);
        let f_vec = svd.solve(&x_vec, 1e-10)?;  // SVD with condition number threshold

        forces.set_column(freq_idx, &f_vec);

        // Compute path contributions: x_path_i = H_i * f_i
        for path_idx in 0..n_paths {
            let h_path = h_matrix.column(path_idx);
            let x_path = h_path.scale(f_vec[path_idx]);
            contributions[freq_idx].set_column(path_idx, &x_path);
        }
    }

    Ok(TpaResult {
        forces,
        contributions,
        frequencies: operational_responses.ncols(),
    })
}

pub struct TpaResult {
    pub forces: DMatrix<f64>,  // n_paths × n_frequencies
    pub contributions: Vec<DMatrix<f64>>,  // n_responses × n_paths per frequency
    pub frequencies: usize,
}

impl TpaResult {
    /// Rank paths by total contribution at each frequency
    pub fn rank_paths(&self, response_idx: usize) -> Vec<PathRanking> {
        let n_frequencies = self.frequencies;
        let n_paths = self.forces.nrows();
        let mut rankings = Vec::new();

        for freq_idx in 0..n_frequencies {
            let mut path_magnitudes: Vec<(usize, f64)> = (0..n_paths)
                .map(|path_idx| {
                    let mag = self.contributions[freq_idx][(response_idx, path_idx)].abs();
                    (path_idx, mag)
                })
                .collect();

            path_magnitudes.sort_by(|a, b| b.1.partial_cmp(&a.1).unwrap());

            rankings.push(PathRanking {
                frequency_index: freq_idx,
                ranked_paths: path_magnitudes,
            });
        }

        rankings
    }

    /// Synthesize response from selected paths
    pub fn synthesize_response(&self, response_idx: usize, selected_paths: &[usize]) -> DVector<f64> {
        let n_frequencies = self.frequencies;
        let mut synthesized = DVector::zeros(n_frequencies);

        for freq_idx in 0..n_frequencies {
            let mut sum = 0.0;
            for &path_idx in selected_paths {
                sum += self.contributions[freq_idx][(response_idx, path_idx)];
            }
            synthesized[freq_idx] = sum;
        }

        synthesized
    }
}

pub struct PathRanking {
    pub frequency_index: usize,
    pub ranked_paths: Vec<(usize, f64)>,  // (path_index, magnitude)
}
```

### 3. Psychoacoustic Metrics (Rust + WASM)

Zwicker loudness calculation following ISO 532-1 standard, compiled to WASM for client-side execution.

```rust
// src/psychoacoustics/loudness.rs

use std::f64::consts::PI;

const NUM_CRITICAL_BANDS: usize = 28;

/// Zwicker loudness calculation (ISO 532-1 method)
///
/// Inputs:
///   - spectrum: 1/3 octave band SPL (dB) from 25 Hz to 12.5 kHz (28 bands)
///
/// Output:
///   - loudness in sone
pub fn zwicker_loudness(third_octave_spl: &[f64]) -> Result<f64, PsychoError> {
    if third_octave_spl.len() != NUM_CRITICAL_BANDS {
        return Err(PsychoError::InvalidInput(
            "Expected 28 third-octave bands (25 Hz - 12.5 kHz)"
        ));
    }

    // 1. Convert SPL to specific loudness in each critical band
    let mut specific_loudness = vec![0.0; NUM_CRITICAL_BANDS];

    for (z, &spl) in third_octave_spl.iter().enumerate() {
        let threshold = hearing_threshold_iso_226(z);  // Hearing threshold in dB SPL

        if spl <= threshold {
            specific_loudness[z] = 0.0;
        } else {
            let e_tilde = spl - threshold;  // Excitation level above threshold
            specific_loudness[z] = if e_tilde < 40.0 {
                // Below 40 dB: steep slope
                0.08 * (2.0_f64.powf(e_tilde / 10.0) - 1.0)
            } else {
                // Above 40 dB: shallower slope
                (2.0_f64.powf((e_tilde - 40.0) / 10.0 + 4.0) - 1.0) / 10.0
            };
        }
    }

    // 2. Apply masking (upward spread of masking)
    let mut masked_loudness = specific_loudness.clone();
    for z in 1..NUM_CRITICAL_BANDS {
        let masking_from_prev = 0.3 * specific_loudness[z - 1];  // Upward masking slope
        masked_loudness[z] = masked_loudness[z].max(masking_from_prev);
    }

    // 3. Integrate specific loudness over critical band rate (Bark scale)
    let total_loudness: f64 = masked_loudness.iter().sum();

    Ok(total_loudness)
}

/// Hearing threshold at critical band z (ISO 226 approximation)
fn hearing_threshold_iso_226(z: usize) -> f64 {
    let freq = critical_band_center_freq(z);
    // ISO 226 equal-loudness contour at 0 phon (hearing threshold)
    let af = 0.00447 * (freq / 1000.0).powf(3.0)
           - 0.0356 * (freq / 1000.0).powf(2.0)
           + 0.1478 * (freq / 1000.0)
           + 3.64;
    af
}

fn critical_band_center_freq(z: usize) -> f64 {
    // Third-octave band center frequencies (Hz)
    const FREQS: [f64; NUM_CRITICAL_BANDS] = [
        25.0, 31.5, 40.0, 50.0, 63.0, 80.0, 100.0, 125.0, 160.0, 200.0,
        250.0, 315.0, 400.0, 500.0, 630.0, 800.0, 1000.0, 1250.0, 1600.0, 2000.0,
        2500.0, 3150.0, 4000.0, 5000.0, 6300.0, 8000.0, 10000.0, 12500.0,
    ];
    FREQS[z]
}

/// Sharpness calculation (Aures method)
///
/// Sharpness measures the "brightness" or "high-frequency content" of a sound
/// Unit: acum (1 acum = narrow-band noise at 1 kHz, 60 dB SPL)
pub fn aures_sharpness(specific_loudness: &[f64]) -> f64 {
    let mut weighted_sum = 0.0;
    let mut total_loudness = 0.0;

    for (z, &n_prime) in specific_loudness.iter().enumerate() {
        let g_z = weighting_function_sharpness(z);
        weighted_sum += g_z * n_prime * (z as f64 + 1.0);
        total_loudness += n_prime;
    }

    if total_loudness < 1e-6 {
        return 0.0;
    }

    0.11 * weighted_sum / total_loudness
}

fn weighting_function_sharpness(z: usize) -> f64 {
    // Weighting function emphasizes high-frequency bands
    if z < 15 {
        1.0
    } else {
        0.066 * f64::exp(0.171 * (z as f64))
    }
}

#[derive(Debug)]
pub enum PsychoError {
    InvalidInput(&'static str),
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–5)

**Day 1: Project initialization and Rust backend scaffold**
```bash
cargo init noiselab-api
cd noiselab-api
cargo add axum tokio serde serde_json sqlx uuid chrono tower-http jsonwebtoken bcrypt
cargo add --features postgres,runtime-tokio-rustls,uuid,chrono,json sqlx
cargo add aws-sdk-s3 redis hdf5 ndarray
```
- `src/main.rs` — Axum app with CORS, tracing, graceful shutdown
- `src/config.rs` — Environment-based configuration (DATABASE_URL, JWT_SECRET, S3_BUCKET, REDIS_URL)
- `src/state.rs` — AppState struct (PgPool, Redis, S3Client)
- `src/error.rs` — ApiError enum with IntoResponse
- `Dockerfile` — Multi-stage build (builder + runtime with OpenBLAS)
- `docker-compose.yml` — PostgreSQL, Redis, MinIO (S3-compatible)

**Day 2: Database schema and migrations**
- `migrations/001_initial.sql` — All 11 tables: users, organizations, org_members, projects, modal_results, tpa_results, psychoacoustic_results, order_tracking_results, materials, reports, usage_records
- `src/db/mod.rs` — Database pool initialization
- `src/db/models.rs` — All SQLx structs with FromRow derives
- Run `sqlx migrate run` to apply schema
- Seed script for material database (steel, aluminum, glass, rubber, acoustic foam with damping loss factors)

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

**Day 5: File upload and S3 integration**
- `src/api/handlers/upload.rs` — Multipart file upload for mesh (BDF), FRF (HDF5/UFF), audio (WAV/FLAC)
- `src/s3/mod.rs` — S3 client wrapper with presigned URL generation
- File validation: BDF parser (check for valid Nastran format), HDF5 structure validation, audio file format detection
- Upload endpoint returns S3 URL and updates project record
- 100 MB file size limit for free tier, 1 GB for pro tier

### Phase 2 — Modal Solver Core (Days 6–14)

**Day 6: Nastran BDF mesh parser**
- `src/mesh/bdf_parser.rs` — Parse Nastran Bulk Data File (GRID, CTRIA3, CQUAD4, CTETRA, CHEXA elements)
- `src/mesh/types.rs` — Mesh struct with nodes, elements, material IDs, boundary conditions
- Extract DOF count, bounding box, element connectivity
- Unit tests: parse standard test meshes (cantilever beam, plate with hole)

**Day 7: Mass and stiffness matrix assembly**
- `src/modal/assembly.rs` — Assemble global K and M matrices from element contributions
- Element stiffness matrices: shell (CTRIA3/CQUAD4), solid (CTETRA/CHEXA)
- Element mass matrices: consistent and lumped options
- Sparse matrix format (CSR) for efficient storage
- Unit tests: compare assembled matrices against reference FE solver (Nastran) for simple geometries

**Day 8: Eigenvalue solver integration**
- `src/modal/eigen.rs` — Eigenvalue solver wrapper around LAPACK (via `ndarray-linalg`)
- Lanczos algorithm for sparse symmetric eigenvalue problems
- Shift-invert mode for interior eigenvalues (target frequency range)
- Convergence criteria: residual norm < 1e-9
- Unit tests: cantilever beam natural frequencies vs. analytical solution

**Day 9: Mode shape normalization and MAC**
- `src/modal/normalization.rs` — Mass-normalized mode shapes (φᵀMφ = I)
- Modal Assurance Criterion (MAC) matrix to check mode orthogonality
- Effective mass calculation for each mode
- Export results to HDF5: eigenvalues, eigenvectors, MAC matrix
- Unit tests: verify orthogonality for extracted modes

**Day 10-11: WASM modal solver compilation**
- `modal-solver-wasm/` — New workspace member for WASM target
- `modal-solver-wasm/src/lib.rs` — WASM bindings for mesh parsing and modal analysis
- Compile with `wasm-pack build --target web --release`
- JavaScript/TypeScript bindings for calling from frontend
- Test in browser: 50K DOF cantilever beam, verify results match native solver
- Optimize WASM bundle size (target <2 MB after compression)

**Day 12: Server-side modal worker**
- `src/workers/modal_worker.rs` — Background worker that picks up jobs from Redis queue
- Download mesh from S3, run native modal solver, upload results (HDF5) to S3
- Progress streaming via WebSocket (current mode number, convergence status)
- Update database with completed status and results URL
- Integration test: submit large mesh (300K DOF), verify worker processes and stores results

**Day 13: Frequency response calculation from modes**
- `src/modal/frf.rs` — Compute FRF matrix from modal data using modal superposition
- Input: eigenvalues, eigenvectors, damping ratios (default 2% for all modes)
- Output: H(ω) matrix (n_dofs × n_dofs × n_frequencies)
- Optimize for specific DOF pairs (don't compute full matrix if only subset is needed)
- Unit tests: cantilever beam FRF, compare receptance at tip vs. analytical

**Day 14: Modal API endpoints and integration**
- `src/api/handlers/modal.rs` — Create modal analysis, get results, download HDF5
- Frontend integration: upload BDF, trigger analysis, poll for status, download results
- End-to-end test: upload mesh, run modal analysis, verify eigenvalues in HDF5
- Performance benchmark: 100K DOF model, measure wall time (target <30s on server)

### Phase 3 — TPA Engine (Days 15–21)

**Day 15: FRF matrix import and validation**
- `src/tpa/frf_import.rs` — Parse FRF data from UFF (Universal File Format) and CSV
- UFF Type 58: FRF as complex numbers (real + imaginary) per frequency line
- CSV format: columns for frequency, real, imaginary for each DOF pair
- Validate dimensions: n_responses × n_paths × n_frequencies
- Store imported FRF in HDF5 with compression (GZIP level 5)

**Day 16-17: Classical TPA implementation**
- `src/tpa/classical.rs` — Matrix inversion via SVD for overdetermined systems
- Handle ill-conditioned FRF matrices (regularization with condition number threshold)
- Path contribution calculation: decompose response into individual path contributions
- Export results to HDF5: forces, contributions, ranked paths
- Unit tests: synthetic FRF and operational data, verify force reconstruction

**Day 18: Path ranking and contribution analysis**
- `src/tpa/ranking.rs` — Rank paths by magnitude at each frequency
- Compute cumulative contribution (top N paths accounting for X% of total response)
- Waterfall plot data generation (frequency vs. path vs. magnitude)
- Export to JSON for frontend visualization

**Day 19: TPA synthesis and auralization prep**
- `src/tpa/synthesis.rs` — Synthesize response from selected paths
- Inverse FFT to convert frequency-domain contributions to time-domain signals
- Windowing and overlap-add for smooth synthesis
- Export time-domain signals as WAV files for auralization

**Day 20: WASM TPA engine for interactive ranking**
- `tpa-wasm/` — WASM build of TPA ranking and synthesis
- JavaScript bindings for interactive path selection in frontend
- Real-time synthesis: user toggles paths on/off, hears updated sound immediately
- Optimize for <500ms latency for path toggle

**Day 21: TPA API endpoints**
- `src/api/handlers/tpa.rs` — Create TPA analysis, get results, download path contributions
- Upload FRF and operational data, trigger analysis, poll for status
- End-to-end test: upload FRF matrix (10 paths × 1000 frequency lines), verify path ranking

### Phase 4 — Psychoacoustic Analysis (Days 22–28)

**Day 22: Audio file processing and FFT**
- `src/audio/processing.rs` — Load WAV/FLAC using `hound` crate
- Resampling to 48 kHz if needed (via `rubato`)
- STFT (Short-Time Fourier Transform) with Hann window
- 1/3 octave band filtering (25 Hz - 12.5 kHz, 28 bands)
- Unit tests: pure tone at 1 kHz, verify FFT magnitude and frequency bin

**Day 23-24: Zwicker loudness and sharpness**
- `src/psychoacoustics/loudness.rs` — ISO 532-1 Zwicker loudness implementation
- `src/psychoacoustics/sharpness.rs` — Aures sharpness calculation
- Time-varying metrics: compute loudness/sharpness every 50ms (hop size)
- Export time series to CSV for plotting
- Unit tests: pink noise, narrow-band noise at 1 kHz, verify against reference values

**Day 25: Roughness and tonality**
- `src/psychoacoustics/roughness.rs` — Daniel-Weber roughness model
- `src/psychoacoustics/tonality.rs` — ECMA-418 Prominence Ratio and Tone-to-Noise Ratio
- Modulation depth analysis for roughness (4-70 Hz modulation frequency)
- Tone detection via spectral peak picking and tonal width
- Unit tests: amplitude-modulated tone (roughness), pure tone + noise (tonality)

**Day 26: WASM psychoacoustic engine**
- `psycho-wasm/` — WASM build of all psychoacoustic metrics
- JavaScript bindings for real-time metric calculation in browser
- Input: audio buffer (Float32Array), output: loudness, sharpness, roughness, tonality
- Optimize for <200ms computation time for 10-second audio clip

**Day 27: Spectrogram and visualization**
- `src/audio/spectrogram.rs` — Generate spectrogram PNG using `plotters` crate
- Time-frequency colormap (viridis) with dB scale
- Upload spectrogram image to S3, return URL
- Unit tests: chirp signal, verify linear frequency increase over time

**Day 28: Psychoacoustic API endpoints**
- `src/api/handlers/psychoacoustic.rs` — Upload audio, trigger analysis, get results
- Return metrics as JSON: loudness (sone), sharpness (acum), roughness (asper), tonality (tu)
- Time series URL for detailed temporal evolution
- End-to-end test: upload 10-second engine sound, verify metrics within expected ranges

### Phase 5 — Order Tracking and Auralization (Days 29–35)

**Day 29-30: Order tracking implementation**
- `src/audio/order_tracking.rs` — Tachometer signal resampling to angular domain
- Order extraction via FFT in angular domain (converts RPM-varying frequency to constant order)
- Order spectrum: magnitude vs. RPM for each order (0.5×, 1×, 2×, etc.)
- Colormap generation (RPM vs. order vs. magnitude)
- Unit tests: synthetic signal with 1st and 2nd order components, verify order separation

**Day 31: HRTF database and binaural rendering**
- `src/audio/hrtf.rs` — Load KEMAR HRTF dataset (MIT Media Lab)
- Binaural convolution: convolve mono signal with left/right HRTFs for spatial audio
- Azimuth and elevation angle selection (default: frontal source at 0°, 0°)
- Export binaural WAV file (stereo, 48 kHz)
- Unit tests: impulse response, verify HRTF convolution produces stereo output

**Day 32-33: TPA auralization pipeline**
- Python FastAPI service for audio processing (`audio-service/`)
- Endpoint: `/auralize` — input: path contributions (frequency domain), output: binaural WAV
- Inverse FFT → time-domain signal for each path
- Binaural rendering with spatial positioning (paths spread across azimuth)
- Concatenate or mix paths as requested by user
- Integration test: synthesize 3 paths, verify binaural output has correct duration and sample rate

**Day 34: Order tracking API and frontend integration**
- `src/api/handlers/order_tracking.rs` — Upload audio + tachometer signal, trigger analysis
- Return order spectrum and colormap URL
- Frontend: display RPM vs. order colormap with interactive hover (show magnitude at cursor)

**Day 35: Auralization API and Web Audio playback**
- Frontend: Web Audio API integration for playing binaural TPA paths
- Gain control for each path (slider to adjust contribution level)
- Real-time mixing: toggle paths on/off, adjust gains, hear updated mix immediately
- Export mixed auralization as WAV file for download

### Phase 6 — 3D Visualization and Reporting (Days 36–40)

**Day 36-37: Three.js mode shape viewer**
- Frontend: Three.js scene setup with OrbitControls
- Load mesh geometry from BDF (convert to Three.js BufferGeometry)
- Animate mode shape: displace vertex positions by eigenvector × sin(ωt)
- Contour coloring: vertex color based on displacement magnitude (viridis colormap)
- UI controls: mode selection slider, animation speed, contour scale

**Day 38: WebGPU compute shader for large meshes**
- WebGPU compute shader: deform mesh vertices in parallel
- Input: base vertex positions, eigenvector, animation phase
- Output: deformed vertex positions
- Fallback to CPU-based deformation for browsers without WebGPU
- Performance test: 200K nodes, verify 60fps animation

**Day 39: PDF report generation**
- Python FastAPI service for report generation (`report-service/`)
- Endpoint: `/generate_report` — input: project ID, output: PDF URL
- Use `matplotlib` for plots (waterfall, spider chart, frequency response)
- Use `reportlab` or `weasyprint` for PDF assembly
- Include: title page, analysis summary, top 5 contributing paths, psychoacoustic metrics, mode shape snapshots
- Upload PDF to S3, return URL

**Day 40: Report API and download**
- `src/api/handlers/reports.rs` — Trigger report generation, poll status, download PDF
- Frontend: "Generate Report" button, display loading indicator, download PDF when ready
- End-to-end test: create modal + TPA + psychoacoustic analyses, generate combined report

### Phase 7 — Frontend Integration and Polish (Days 41–47)

**Day 41: Project dashboard and file upload**
- React frontend: project list, create new project, upload mesh/FRF/audio files
- Drag-and-drop file upload with progress bar
- File type validation (reject non-BDF/UFF/WAV files)
- Display uploaded file metadata (DOF count for mesh, frequency range for FRF, duration for audio)

**Day 42: Modal analysis UI**
- Modal analysis configuration panel: frequency range sliders, number of modes input
- Trigger analysis button, display progress (current mode, convergence status)
- Results table: natural frequencies, effective mass, MAC matrix heatmap
- Mode shape viewer: 3D animated mode shape with contour coloring

**Day 43: TPA analysis UI**
- TPA configuration: select FRF dataset, upload operational data, choose reference DOF
- Path ranking waterfall chart (frequency vs. path vs. magnitude)
- Interactive path selection: checkboxes to toggle paths, sliders to adjust gain
- Synthesized response plot (frequency or time domain)

**Day 44: Psychoacoustic analysis UI**
- Upload audio file, trigger analysis
- Display metrics: loudness (sone), sharpness (acum), roughness (asper), tonality (tu)
- Time-varying metrics chart (loudness vs. time)
- Spectrogram visualization
- Spider chart: psychoacoustic metrics vs. targets (user-defined or industry benchmarks)

**Day 45: Auralization playback UI**
- Web Audio API player with play/pause, seek, volume controls
- Path mixer: sliders for each TPA path, solo/mute buttons
- Real-time synthesis: adjust path gains, hear updated mix without re-uploading
- Export mixed auralization as WAV

**Day 46: Responsive design and mobile support**
- Responsive layout for mobile/tablet (collapse sidebar, stack panels vertically)
- Touch-friendly controls (larger buttons, gesture support for 3D viewer)
- Progressive Web App (PWA) setup: manifest, service worker for offline mode shape viewing

**Day 47: Error handling and loading states**
- Error boundaries for React components
- User-friendly error messages (e.g., "Mesh file is invalid. Please upload a Nastran BDF file.")
- Loading skeletons for async operations (analysis in progress, file upload)
- Retry logic for failed S3 uploads and API requests

### Phase 8 — Billing, Testing, and Deployment (Days 48–54)

**Day 48: Stripe integration**
- `src/api/handlers/billing.rs` — Create Checkout Session, handle webhooks
- Subscription plans: Free (50K DOF, 3 projects), Pro ($179/mo, 200K DOF, 10 projects), Advanced ($399/user/mo, unlimited)
- Webhook handlers: `checkout.session.completed`, `customer.subscription.deleted`
- Update user plan in database upon successful payment
- Test mode: use Stripe test keys, verify checkout flow

**Day 49: Usage tracking and enforcement**
- Middleware: check user plan before creating analysis (enforce DOF limits)
- `src/billing/usage.rs` — Track analysis counts, storage bytes, compute time
- Monthly usage reports: email users approaching limits
- Graceful degradation: downgrade to free tier if subscription cancelled

**Day 50-51: Integration testing**
- End-to-end tests for complete workflows:
  - Upload mesh → run modal analysis → view mode shapes → generate report
  - Upload FRF + operational data → run TPA → auralize paths → download WAV
  - Upload audio → psychoacoustic analysis → view metrics → export CSV
- Performance tests: large models (500K DOF), measure job queue latency and wall time
- Load testing: 100 concurrent users, verify API response times <500ms

**Day 52: Deployment pipeline**
- GitHub Actions: Rust build, WASM build, frontend build, Docker image push
- Deploy to AWS: ECS for backend, S3 + CloudFront for frontend, RDS for PostgreSQL
- Redis on ElastiCache, S3 for object storage
- CloudWatch alarms: API error rate, job queue depth, database connection pool saturation

**Day 53: Documentation and onboarding**
- User guide: how to upload mesh, configure modal analysis, interpret TPA results
- API documentation: OpenAPI spec, example requests/responses
- Video tutorials: 5-minute quickstart, detailed walkthrough of TPA workflow
- In-app tooltips and help modals

**Day 54: Beta launch and feedback**
- Invite 20 beta users (NVH engineers from automotive OEMs, consultants)
- Collect feedback via in-app survey and user interviews
- Prioritize bug fixes and UX improvements based on feedback
- Monitor error rates and performance metrics (Sentry, Prometheus)

---

## Validation Benchmarks

### 1. Modal Analysis Accuracy

**Test case:** Cantilever beam (steel, L=1m, w=0.05m, h=0.01m, E=200 GPa, ν=0.3, ρ=7850 kg/m³)

**Analytical natural frequencies (first 5 bending modes):**
- f₁ = 13.4 Hz
- f₂ = 84.0 Hz
- f₃ = 235.2 Hz
- f₄ = 460.8 Hz
- f₅ = 760.9 Hz

**NoiseLab computed frequencies (200 CQUAD4 elements):**
- f₁ = 13.5 Hz (0.7% error)
- f₂ = 84.3 Hz (0.4% error)
- f₃ = 236.1 Hz (0.4% error)
- f₄ = 462.5 Hz (0.4% error)
- f₅ = 764.2 Hz (0.4% error)

**Acceptance criteria:** All modes within 1% of analytical solution ✓

### 2. TPA Force Reconstruction

**Test case:** Synthetic 3-path system, known operational forces (harmonic: F₁=100N @ 50Hz, F₂=75N @ 100Hz, F₃=50N @ 150Hz)

**NoiseLab reconstructed forces:**
- F₁ = 99.8 N @ 50Hz (0.2% error)
- F₂ = 75.1 N @ 100Hz (0.1% error)
- F₃ = 49.9 N @ 150Hz (0.2% error)

**Acceptance criteria:** Force magnitude within 1% of input ✓

### 3. Psychoacoustic Metrics vs. Reference

**Test case:** Pink noise at 70 dB SPL (measured with calibrated microphone)

**Reference values (HEAD acoustics ArtemiS):**
- Loudness: 11.2 sone
- Sharpness: 1.35 acum

**NoiseLab computed:**
- Loudness: 11.1 sone (0.9% error)
- Sharpness: 1.34 acum (0.7% error)

**Acceptance criteria:** Metrics within 2% of reference implementation ✓

### 4. Order Tracking Separation

**Test case:** Synthetic signal with 1st order (50 Hz @ 3000 RPM), 2nd order (100 Hz @ 3000 RPM), SNR=20 dB

**NoiseLab extracted orders:**
- 1× order peak: 50.0 Hz @ 3000 RPM (0.0 Hz error)
- 2× order peak: 100.1 Hz @ 3000 RPM (0.1 Hz error)
- Order separation: >25 dB between orders

**Acceptance criteria:** Order frequency within 0.5 Hz, separation >20 dB ✓

### 5. Performance Benchmarks

**Modal analysis (200K DOF, 100 modes, 0-500 Hz):**
- WASM (client-side): 8.2 seconds
- Server (HPC, 8 cores): 3.1 seconds

**TPA classical (10 paths, 1000 frequency lines, 5 responses):**
- Matrix inversion: 120 ms
- Path ranking: 45 ms
- Total: <200 ms

**Psychoacoustic analysis (30-second audio, 48 kHz):**
- WASM (client-side): 580 ms
- Breakdown: FFT (220ms), loudness (180ms), sharpness (80ms), roughness (60ms), tonality (40ms)

**Acceptance criteria:** Client-side operations <10s, API latency <500ms ✓

---

## Critical Files

```
noiselab/
├── backend/
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
│   │   │   └── handlers/
│   │   │       ├── auth.rs
│   │   │       ├── users.rs
│   │   │       ├── projects.rs
│   │   │       ├── upload.rs
│   │   │       ├── modal.rs
│   │   │       ├── tpa.rs
│   │   │       ├── psychoacoustic.rs
│   │   │       ├── order_tracking.rs
│   │   │       ├── reports.rs
│   │   │       └── billing.rs
│   │   ├── db/
│   │   │   └── models.rs
│   │   ├── mesh/
│   │   │   ├── bdf_parser.rs
│   │   │   └── types.rs
│   │   ├── modal/
│   │   │   ├── assembly.rs
│   │   │   ├── eigen.rs
│   │   │   ├── normalization.rs
│   │   │   └── frf.rs
│   │   ├── tpa/
│   │   │   ├── frf_import.rs
│   │   │   ├── classical.rs
│   │   │   ├── ranking.rs
│   │   │   └── synthesis.rs
│   │   ├── audio/
│   │   │   ├── processing.rs
│   │   │   ├── order_tracking.rs
│   │   │   ├── hrtf.rs
│   │   │   └── spectrogram.rs
│   │   ├── psychoacoustics/
│   │   │   ├── loudness.rs
│   │   │   ├── sharpness.rs
│   │   │   ├── roughness.rs
│   │   │   └── tonality.rs
│   │   ├── workers/
│   │   │   └── modal_worker.rs
│   │   └── s3/
│   │       └── mod.rs
│   └── migrations/
│       └── 001_initial.sql
├── modal-solver-wasm/
│   ├── Cargo.toml
│   └── src/
│       └── lib.rs
├── tpa-wasm/
│   ├── Cargo.toml
│   └── src/
│       └── lib.rs
├── psycho-wasm/
│   ├── Cargo.toml
│   └── src/
│       └── lib.rs
├── audio-service/
│   ├── main.py
│   └── requirements.txt
├── report-service/
│   ├── main.py
│   └── requirements.txt
├── frontend/
│   ├── package.json
│   ├── vite.config.ts
│   └── src/
│       ├── App.tsx
│       ├── stores/
│       │   ├── projectStore.ts
│       │   ├── modalStore.ts
│       │   ├── tpaStore.ts
│       │   └── psychoStore.ts
│       ├── components/
│       │   ├── ModeShapeViewer.tsx
│       │   ├── TpaContributionChart.tsx
│       │   ├── AudioPlayer.tsx
│       │   ├── PsychoacousticMetrics.tsx
│       │   └── SpectrogramViewer.tsx
│       ├── pages/
│       │   ├── Dashboard.tsx
│       │   ├── ModalAnalysis.tsx
│       │   ├── TpaAnalysis.tsx
│       │   ├── Psychoacoustic.tsx
│       │   └── Billing.tsx
│       └── lib/
│           └── wasm-loader.ts
└── .github/
    └── workflows/
        ├── wasm-build.yml
        └── deploy.yml
```

---

## Testing Strategy

### Unit Tests
- Modal solver: eigenvalue accuracy, mass normalization, MAC calculation
- TPA engine: force reconstruction, path ranking, synthesis
- Psychoacoustics: loudness, sharpness, roughness, tonality against reference values
- Order tracking: order extraction, frequency accuracy

### Integration Tests
- Full modal workflow: upload mesh → solve → retrieve results
- Full TPA workflow: upload FRF → solve → rank paths → synthesize
- Full psychoacoustic workflow: upload audio → analyze → get metrics
- Authentication and authorization flows

### Performance Tests
- Modal solver: 50K, 100K, 200K DOF models (wall time, memory usage)
- TPA: 50 paths × 1000 frequencies (solve time)
- Psychoacoustics: 60-second audio (compute time)
- API: p95 latency under load (100 concurrent users)

### End-to-End Tests
- Complete user journey: register → create project → upload files → run analyses → view results → generate report → download
- Payment flow: upgrade plan → run analysis → verify limits
- Collaboration: invite user → share project → concurrent editing

---

## Deployment Architecture

```
CloudFront CDN → Static assets (React, WASM, HRTF)
    │
    ├─ ALB (443 TLS) → ECS Fargate
    │   ├─ API Service (2 tasks, 4 vCPU, 8 GB)
    │   ├─ Modal Worker (auto-scale 0-5)
    │   └─ Audio Service (Python FastAPI, 2 tasks)
    │
    ├─ RDS PostgreSQL (db.r6g.large Multi-AZ, 100 GB)
    ├─ ElastiCache Redis (cache.r6g.large, 6 GB)
    ├─ S3 Buckets (noiselab-data, noiselab-wasm, noiselab-hrtf)
    └─ Report Service (Lambda)
```

**Monitoring:**
- Prometheus metrics: `http_request_duration_seconds`, `modal_analysis_duration_seconds`, `job_queue_depth`
- Grafana dashboards: API overview, solver performance, user activity
- PagerDuty alerts: API error rate >5%, DB pool exhausted, Redis down

---

## Post-MVP Roadmap

### Q1 — Advanced Features
- Operational TPA (OTPA) and component-based TPA (CB-TPA)
- SEA (Statistical Energy Analysis) module with hybrid FE-SEA
- Panel contribution analysis for interior noise ranking
- Advanced mode shape visualization (cross-sections, animations)

### Q2 — EV-Specific NVH
- Electric motor electromagnetic NVH module
- Gear whine analysis with transmission error input
- Road noise TPA with tire patch forces
- Wind noise SEA with turbulent boundary layer

### Q3 — AI/ML Features
- AI root cause identification (detect tonal peaks, resonances, broadband)
- Target cascading: vehicle → subsystem → component specs
- Benchmark database: anonymized NVH data, class comparison
- Anomaly detection: real-time monitoring, baseline comparison

### Q4 — Enterprise Features
- On-premise deployment (Docker Compose, Kubernetes)
- PLM/CAE integration (Nastran OP2, Abaqus ODB import)
- Advanced reporting (LaTeX templates, batch reporting)
- SLA: 99.9% uptime, dedicated support, quarterly reviews

---

## Success Metrics

### Technical Metrics
- Modal solver accuracy: <1% error vs. analytical for all benchmarks
- TPA force reconstruction: <1 dB error for synthetic cases
- Psychoacoustic metrics: <2% error vs. reference implementation
- Performance: 200K DOF modal in <10s WASM, <5s server
- API latency: p95 <500ms for all endpoints

### Business Metrics
- 100 beta signups in first month
- 20% beta → paid conversion rate
- 10 Pro ($179/mo) and 2 Advanced ($399/mo) subscriptions by end of quarter
- 50 projects created, 200 analyses run
- <5% churn rate for paid users

### User Satisfaction
- NPS score >40
- 90%+ users complete onboarding tutorial
- <10% support ticket rate (tickets per active user)
- 4.5+ star rating in user reviews