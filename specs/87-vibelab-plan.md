# 87. VibeLab — Vibration and Modal Testing Analysis Platform

## Implementation Plan

**MVP Scope:** Browser-based vibration analysis platform with CSV/UFF/TDMS time-history data import supporting up to 32 channels, FFT-based spectral analysis (PSD, CSD, coherence, H1/H2 FRF estimators) with windowing (Hanning, Hamming, exponential) and averaging computed via Rust+WASM for client-side processing (datasets ≤100 MB) and server-side for larger files, experimental modal analysis with LSCE (Least Squares Complex Exponential) and PolyMAX (Polyreference Least-Squares Complex Frequency) curve-fitting algorithms producing interactive stabilization diagrams with automatic pole selection based on frequency/damping/MAC stability criteria, 3D mode shape visualization rendered via Three.js with animated deflections on imported STL/UFF geometry (up to 500 nodes) supporting pan/zoom/rotation, MAC matrix computation (auto-MAC for test-vs-test correlation and cross-MAC for test-vs-FEA validation), basic operational deflection shape (ODS) extraction from measured cross-spectra at user-selected frequencies, PostgreSQL storage for test metadata and modal parameters with S3 for raw time-series and FRF matrices, automated PDF modal test report generation with FRF plots, coherence quality checks, stabilization diagrams, mode tables (frequency/damping/complexity), and mode shape renderings, Free tier (3 projects, 8 channels, 50 MB storage) / Pro $119/mo (unlimited channels, full EMA, 10 GB storage) / Advanced $299/mo (OMA, continuous monitoring, DAQ streaming, 100 GB storage).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| Signal Processing | Rust (native + WASM) | rustfft for FFT/IFFT, custom FRF estimation, windowing, filtering |
| WASM Compilation | `wasm-pack` + `wasm-bindgen` | Client-side processing for datasets ≤100 MB |
| Modal Extraction | Rust (native server-side) | LSCE, PolyMAX, SSI-cov, FDD algorithms with nalgebra linear algebra |
| ML/AI Services | Python 3.12 (FastAPI) | Anomaly detection, automated pole selection clustering, environmental correction |
| Database | PostgreSQL 16 | Test campaigns, users, modal parameters, sensor configurations |
| Time-Series DB | TimescaleDB extension | Continuous monitoring modal tracking, vibration features |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | Time-history data, FRF matrices, mode shapes, geometry meshes |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| 3D Visualization | Three.js + React Three Fiber | Mode shape animation, ODS visualization, mesh rendering |
| 2D Plotting | D3.js + SVG | FRF plots, spectrograms, stabilization diagrams, coherence |
| Real-time | WebSocket (Axum) | Live DAQ streaming, server-side processing progress |
| Job Queue | Redis 7 + Tokio tasks | Server-side processing job distribution, modal extraction queue |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | WASM bundle delivery, static frontend assets |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, error tracking |
| CI/CD | GitHub Actions | WASM build pipeline, Rust tests, Docker image push |

### Key Architecture Decisions

1. **Hybrid client/server signal processing with 100 MB WASM threshold**: Datasets ≤100 MB (covers 90%+ of impact testing with 8-16 channels at 1-10 minute duration) are processed entirely in the browser via WASM, providing instant FFT/FRF computation with zero server cost. Larger datasets and modal extraction (LSCE, PolyMAX, SSI-cov) always run server-side where we have multi-threaded linear algebra and can handle multi-hour continuous monitoring data. The threshold is configurable per plan tier.

2. **Rust signal processing library compiled to both native and WASM**: Building a unified `vibelab-dsp` crate in Rust gives us memory-safe, high-performance FFT (via `rustfft`), FRF estimation (H1, H2, Hv), windowing, and filtering that compiles to both WASM (for client-side) and native (for server-side) from the same source. This avoids Python/NumPy's GIL limitations and gives 5-10x faster FFT than JavaScript libraries, while maintaining numerical accuracy with float64 throughout.

3. **Server-side modal extraction with dedicated solvers**: Unlike signal processing (which is embarrassingly parallel per channel), modal analysis algorithms (PolyMAX, SSI-cov) require large SVD/eigenvalue decompositions on multi-gigabyte correlation matrices. These run exclusively server-side using `nalgebra` with BLAS/LAPACK acceleration, distributed across worker instances via Redis job queue. Results (modal parameters + stabilization diagram metadata) are cached in PostgreSQL with full recomputation traceability.

4. **Three.js for 3D mode shape animation with GPU instancing**: Mode shapes with 500+ measurement DOFs on complex geometries (automotive bodies, bridge decks) require GPU-accelerated rendering. We use Three.js with instanced mesh rendering for wireframe/surface visualization, compute deformed geometry in a vertex shader, and support side-by-side test-vs-FEA comparison. Geometries are stored as compressed binary buffers in S3 and streamed to client. Canvas/SVG were rejected for 3D due to lack of true perspective and lighting.

5. **TimescaleDB for continuous monitoring time-series with automated retention**: Continuous vibration monitoring generates millions of feature vectors (natural frequencies, damping, RMS levels) per structure over months/years. TimescaleDB's hypertable partitioning enables efficient queries on recent data while automatically downsampling/aggregating older data via continuous aggregates. Raw acceleration data has 7-day retention; extracted modal parameters are permanent.

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";  -- For fuzzy text search on test descriptions
CREATE EXTENSION IF NOT EXISTS "timescaledb";  -- For continuous monitoring time-series

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
    storage_used_bytes BIGINT DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX users_email_idx ON users(email);
CREATE INDEX users_stripe_idx ON users(stripe_customer_id);

-- Organizations (for multi-user teams)
CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    slug TEXT UNIQUE NOT NULL,
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    plan TEXT NOT NULL DEFAULT 'advanced',
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
    role TEXT NOT NULL DEFAULT 'member',  -- owner | admin | analyst | viewer
    joined_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(org_id, user_id)
);
CREATE INDEX org_members_org_idx ON org_members(org_id);
CREATE INDEX org_members_user_idx ON org_members(user_id);

-- Test Campaigns (project container)
CREATE TABLE campaigns (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    structure_type TEXT,  -- e.g., "automotive_body", "bridge", "wind_turbine", "rotating_machinery"
    test_type TEXT NOT NULL,  -- impact | shaker | ambient | operational
    geometry_url TEXT,  -- S3 URL to STL/UFF/OBJ mesh file
    geometry_metadata JSONB,  -- {node_count, element_count, units, coordinate_system}
    settings JSONB DEFAULT '{}',
    is_public BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX campaigns_owner_idx ON campaigns(owner_id);
CREATE INDEX campaigns_org_idx ON campaigns(org_id);
CREATE INDEX campaigns_updated_idx ON campaigns(updated_at DESC);

-- Measurement Setups (sensor placement, DAQ config)
CREATE TABLE setups (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    campaign_id UUID NOT NULL REFERENCES campaigns(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    sample_rate REAL NOT NULL,  -- Hz
    duration REAL,  -- seconds (NULL for continuous monitoring)
    channel_count INTEGER NOT NULL,
    channels JSONB NOT NULL,  -- [{id, node_id, dof, sensitivity, units, label}]
    excitation JSONB,  -- {type: "impact"|"shaker"|"ambient", location, direction, ...}
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX setups_campaign_idx ON setups(campaign_id);

-- Measurements (raw time-history data)
CREATE TABLE measurements (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    setup_id UUID NOT NULL REFERENCES setups(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | processing | completed | failed
    execution_mode TEXT NOT NULL DEFAULT 'wasm',  -- wasm | server
    data_url TEXT,  -- S3 URL to binary time-history file (HDF5 or custom format)
    data_format TEXT,  -- csv | uff | tdms | hdf5 | custom_binary
    file_size_bytes BIGINT DEFAULT 0,
    sample_rate REAL NOT NULL,
    duration REAL NOT NULL,
    channel_count INTEGER NOT NULL,
    quality_metrics JSONB,  -- {overloads, clipping_pct, coherence_avg, noise_floor_db}
    processing_log TEXT,
    error_message TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX measurements_setup_idx ON measurements(setup_id);
CREATE INDEX measurements_user_idx ON measurements(user_id);
CREATE INDEX measurements_status_idx ON measurements(status);
CREATE INDEX measurements_created_idx ON measurements(created_at DESC);

-- FRF Matrices (frequency response functions from measurements)
CREATE TABLE frfs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    measurement_id UUID NOT NULL REFERENCES measurements(id) ON DELETE CASCADE,
    estimator TEXT NOT NULL,  -- h1 | h2 | hv
    window_type TEXT NOT NULL,  -- hanning | hamming | exponential | force | flat_top
    overlap_pct REAL NOT NULL DEFAULT 50.0,
    fft_size INTEGER NOT NULL,
    averages INTEGER NOT NULL,
    frequency_resolution REAL NOT NULL,  -- Hz
    frf_data_url TEXT NOT NULL,  -- S3 URL to complex FRF matrix (channels × freqs)
    coherence_url TEXT,  -- S3 URL to coherence matrix
    auto_spectrum_url TEXT,  -- S3 URL to auto-spectra (PSD)
    cross_spectrum_url TEXT,  -- S3 URL to cross-spectra (CSD)
    frequency_range REAL[] NOT NULL,  -- [f_min, f_max] in Hz
    quality_summary JSONB,  -- {coherence_avg, coherence_min, valid_freq_range}
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX frfs_measurement_idx ON frfs(measurement_id);

-- Modal Analyses (extracted modal parameters)
CREATE TABLE modal_analyses (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    frf_id UUID REFERENCES frfs(id) ON DELETE CASCADE,  -- NULL for OMA
    measurement_id UUID REFERENCES measurements(id) ON DELETE CASCADE,  -- For OMA direct from time data
    campaign_id UUID NOT NULL REFERENCES campaigns(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    analysis_type TEXT NOT NULL,  -- ema | oma
    algorithm TEXT NOT NULL,  -- lsce | polymax | ptd | rfp | ssi_cov | ssi_data | fdd | efdd
    frequency_range REAL[] NOT NULL,  -- [f_min, f_max] for curve fitting
    model_order INTEGER NOT NULL,  -- Maximum state-space order or polynomial degree
    stabilization_data_url TEXT NOT NULL,  -- S3 URL to full stabilization diagram poles
    selected_poles JSONB NOT NULL,  -- [{pole_id, frequency, damping, mode_complexity}]
    mode_shapes_url TEXT,  -- S3 URL to mode shape matrix (DOFs × modes)
    processing_params JSONB NOT NULL,  -- Full algorithm parameters for reproducibility
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | running | completed | failed
    error_message TEXT,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    wall_time_ms INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX modal_analyses_frf_idx ON modal_analyses(frf_id);
CREATE INDEX modal_analyses_campaign_idx ON modal_analyses(campaign_id);
CREATE INDEX modal_analyses_user_idx ON modal_analyses(user_id);
CREATE INDEX modal_analyses_status_idx ON modal_analyses(status);

-- Modal Analysis Jobs (for server-side execution queue)
CREATE TABLE modal_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    analysis_id UUID NOT NULL REFERENCES modal_analyses(id) ON DELETE CASCADE,
    worker_id TEXT,
    cores_allocated INTEGER DEFAULT 4,
    memory_mb INTEGER DEFAULT 8192,
    priority INTEGER DEFAULT 0,  -- Higher = more urgent
    progress_pct REAL DEFAULT 0.0,
    progress_message TEXT,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX modal_jobs_analysis_idx ON modal_jobs(analysis_id);
CREATE INDEX modal_jobs_worker_idx ON modal_jobs(worker_id);

-- Modal Parameters (individual identified modes)
CREATE TABLE modes (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    analysis_id UUID NOT NULL REFERENCES modal_analyses(id) ON DELETE CASCADE,
    mode_number INTEGER NOT NULL,
    frequency_hz REAL NOT NULL,
    damping_ratio REAL NOT NULL,
    damping_pct REAL NOT NULL,  -- damping_ratio * 100
    modal_phase REAL,  -- degrees
    mode_complexity REAL,  -- MPC indicator (0=real, 1=complex)
    mpd REAL,  -- Mean Phase Deviation indicator
    mode_shape JSONB,  -- {dof_id: {real, imag, magnitude, phase}} for quick access
    mode_shape_url TEXT,  -- S3 URL if mode shape is large
    participation_factor REAL,  -- Scaling factor
    confidence_interval JSONB,  -- {frequency: [lower, upper], damping: [lower, upper]}
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(analysis_id, mode_number)
);
CREATE INDEX modes_analysis_idx ON modes(analysis_id);
CREATE INDEX modes_frequency_idx ON modes(frequency_hz);

-- MAC Matrix Computations (Modal Assurance Criterion)
CREATE TABLE mac_matrices (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    campaign_id UUID NOT NULL REFERENCES campaigns(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    mac_type TEXT NOT NULL,  -- auto_mac | cross_mac
    analysis_a_id UUID NOT NULL REFERENCES modal_analyses(id) ON DELETE CASCADE,
    analysis_b_id UUID REFERENCES modal_analyses(id) ON DELETE CASCADE,  -- NULL for auto-MAC
    matrix_data_url TEXT NOT NULL,  -- S3 URL to MAC matrix (modes_a × modes_b)
    diagonal_values REAL[],  -- For auto-MAC
    best_matches JSONB,  -- [{mode_a, mode_b, mac_value}] sorted by MAC
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX mac_campaign_idx ON mac_matrices(campaign_id);

-- Operational Deflection Shapes (ODS at specific frequencies)
CREATE TABLE ods_analyses (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    frf_id UUID NOT NULL REFERENCES frfs(id) ON DELETE CASCADE,
    campaign_id UUID NOT NULL REFERENCES campaigns(id) ON DELETE CASCADE,
    frequency_hz REAL NOT NULL,
    ods_type TEXT NOT NULL,  -- frf_based | time_based
    deflection_shape JSONB NOT NULL,  -- {dof_id: {real, imag, magnitude, phase}}
    animation_url TEXT,  -- S3 URL to pre-rendered animation MP4
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX ods_frf_idx ON ods_analyses(frf_id);
CREATE INDEX ods_campaign_idx ON ods_analyses(campaign_id);

-- Continuous Monitoring Structures (for operational monitoring)
CREATE TABLE monitoring_structures (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    org_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    structure_type TEXT NOT NULL,
    location JSONB,  -- {latitude, longitude, address}
    geometry_url TEXT,
    daq_config JSONB NOT NULL,  -- {hardware, channels, sample_rate, streaming_endpoint}
    modal_tracking_config JSONB,  -- {target_modes: [{freq_nominal, tolerance}], oma_interval_sec}
    alert_config JSONB,  -- {thresholds, notification_channels}
    status TEXT NOT NULL DEFAULT 'active',  -- active | paused | offline
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX monitoring_org_idx ON monitoring_structures(org_id);

-- Continuous Monitoring Modal Tracking (TimescaleDB hypertable)
CREATE TABLE modal_tracking (
    time TIMESTAMPTZ NOT NULL,
    structure_id UUID NOT NULL REFERENCES monitoring_structures(id) ON DELETE CASCADE,
    mode_number INTEGER NOT NULL,
    frequency_hz REAL NOT NULL,
    damping_ratio REAL NOT NULL,
    amplitude REAL,
    mac_to_baseline REAL,  -- MAC correlation to baseline mode shape
    environmental_temp_c REAL,
    environmental_wind_mps REAL,
    operational_load REAL,
    corrected_frequency_hz REAL,  -- After environmental/operational correction
    anomaly_score REAL,
    alert_triggered BOOLEAN DEFAULT false
);
SELECT create_hypertable('modal_tracking', 'time');
CREATE INDEX modal_tracking_structure_time_idx ON modal_tracking(structure_id, time DESC);
CREATE INDEX modal_tracking_structure_mode_idx ON modal_tracking(structure_id, mode_number, time DESC);

-- Continuous aggregate for daily modal statistics
CREATE MATERIALIZED VIEW modal_tracking_daily
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', time) AS day,
    structure_id,
    mode_number,
    AVG(frequency_hz) AS freq_avg,
    STDDEV(frequency_hz) AS freq_stddev,
    AVG(damping_ratio) AS damping_avg,
    AVG(corrected_frequency_hz) AS corrected_freq_avg,
    MAX(anomaly_score) AS anomaly_max,
    COUNT(*) AS sample_count
FROM modal_tracking
GROUP BY day, structure_id, mode_number;

-- Reports (generated PDF/HTML reports)
CREATE TABLE reports (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    campaign_id UUID NOT NULL REFERENCES campaigns(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    report_type TEXT NOT NULL,  -- modal_test | ods | mac_correlation | monitoring_summary
    title TEXT NOT NULL,
    template_name TEXT NOT NULL,
    included_analyses JSONB NOT NULL,  -- {analysis_ids, frf_ids, mode_ids}
    content_url TEXT,  -- S3 URL to PDF
    generated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX reports_campaign_idx ON reports(campaign_id);

-- Usage Tracking (for billing enforcement)
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,  -- storage_bytes | server_processing_minutes | modal_extractions
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
    pub storage_used_bytes: i64,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Campaign {
    pub id: Uuid,
    pub owner_id: Uuid,
    pub org_id: Option<Uuid>,
    pub name: String,
    pub description: String,
    pub structure_type: Option<String>,
    pub test_type: String,
    pub geometry_url: Option<String>,
    pub geometry_metadata: Option<serde_json::Value>,
    pub settings: serde_json::Value,
    pub is_public: bool,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Setup {
    pub id: Uuid,
    pub campaign_id: Uuid,
    pub name: String,
    pub description: String,
    pub sample_rate: f32,
    pub duration: Option<f32>,
    pub channel_count: i32,
    pub channels: serde_json::Value,
    pub excitation: Option<serde_json::Value>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Measurement {
    pub id: Uuid,
    pub setup_id: Uuid,
    pub user_id: Uuid,
    pub status: String,
    pub execution_mode: String,
    pub data_url: Option<String>,
    pub data_format: Option<String>,
    pub file_size_bytes: i64,
    pub sample_rate: f32,
    pub duration: f32,
    pub channel_count: i32,
    pub quality_metrics: Option<serde_json::Value>,
    pub processing_log: Option<String>,
    pub error_message: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Frf {
    pub id: Uuid,
    pub measurement_id: Uuid,
    pub estimator: String,
    pub window_type: String,
    pub overlap_pct: f32,
    pub fft_size: i32,
    pub averages: i32,
    pub frequency_resolution: f32,
    pub frf_data_url: String,
    pub coherence_url: Option<String>,
    pub auto_spectrum_url: Option<String>,
    pub cross_spectrum_url: Option<String>,
    pub frequency_range: Vec<f32>,
    pub quality_summary: Option<serde_json::Value>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct ModalAnalysis {
    pub id: Uuid,
    pub frf_id: Option<Uuid>,
    pub measurement_id: Option<Uuid>,
    pub campaign_id: Uuid,
    pub user_id: Uuid,
    pub analysis_type: String,
    pub algorithm: String,
    pub frequency_range: Vec<f32>,
    pub model_order: i32,
    pub stabilization_data_url: String,
    pub selected_poles: serde_json::Value,
    pub mode_shapes_url: Option<String>,
    pub processing_params: serde_json::Value,
    pub status: String,
    pub error_message: Option<String>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub wall_time_ms: Option<i32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Mode {
    pub id: Uuid,
    pub analysis_id: Uuid,
    pub mode_number: i32,
    pub frequency_hz: f32,
    pub damping_ratio: f32,
    pub damping_pct: f32,
    pub modal_phase: Option<f32>,
    pub mode_complexity: Option<f32>,
    pub mpd: Option<f32>,
    pub mode_shape: Option<serde_json::Value>,
    pub mode_shape_url: Option<String>,
    pub participation_factor: Option<f32>,
    pub confidence_interval: Option<serde_json::Value>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct FrfParams {
    pub estimator: FrfEstimator,
    pub window_type: WindowType,
    pub overlap_pct: f32,
    pub fft_size: usize,
    pub averages: usize,
    pub frequency_range: Option<[f32; 2]>,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "lowercase")]
pub enum FrfEstimator {
    H1,  // Input PSD-based
    H2,  // Output PSD-based
    Hv,  // Average of H1 and H2
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "lowercase")]
pub enum WindowType {
    Hanning,
    Hamming,
    Exponential,
    Force,
    FlatTop,
    Rectangular,
}

#[derive(Debug, Deserialize)]
pub struct ModalExtractionParams {
    pub algorithm: ModalAlgorithm,
    pub frequency_range: [f32; 2],
    pub model_order: usize,
    pub stability_criteria: StabilityCriteria,
    pub pole_selection: PoleSelectionMode,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "lowercase")]
pub enum ModalAlgorithm {
    Lsce,      // Least Squares Complex Exponential
    PolyMax,   // Polyreference Least-Squares Complex Frequency
    Ptd,       // Polyreference Time Domain
    Rfp,       // Rational Fraction Polynomial
    SsiCov,    // Stochastic Subspace Identification - Covariance
    SsiData,   // Stochastic Subspace Identification - Data
    Fdd,       // Frequency Domain Decomposition
    Efdd,      // Enhanced FDD
}

#[derive(Debug, Deserialize, Serialize)]
pub struct StabilityCriteria {
    pub freq_tolerance_pct: f32,  // Default 1.0%
    pub damp_tolerance_pct: f32,  // Default 5.0%
    pub mac_threshold: f32,       // Default 0.95
    pub min_stable_poles: usize,  // Default 3
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "lowercase")]
pub enum PoleSelectionMode {
    Manual,         // User clicks on stabilization diagram
    AutoClustering, // K-means or DBSCAN on stable poles
    AutoStability,  // Select poles stable across multiple orders
}
```

---

## Solver Architecture Deep-Dive

### Governing Equations and Algorithms

VibeLab's core processing implements industry-standard vibration analysis algorithms used in commercial tools like LMS Test.Lab and ME'scope.

**FFT-Based Spectral Analysis**

For a time-domain signal **x(t)** sampled at **fs** Hz with **N** samples, the discrete Fourier transform is:

```
X[k] = Σ_{n=0}^{N-1} x[n] · e^{-j2πkn/N}

where k = 0, 1, ..., N-1 (frequency bins)
```

We use `rustfft` for radix-2 FFT with optional zero-padding to next power of 2. Frequency resolution is:

```
Δf = fs / N (Hz per bin)
```

**Windowing** reduces spectral leakage for non-periodic signals. Hanning window is the default:

```
w[n] = 0.5 · (1 - cos(2πn / (N-1)))
```

Exponential window (for impact testing with decaying response):

```
w[n] = e^{-ξ·n/N}  where ξ is damping factor (user-configurable, default 0.05)
```

**FRF Estimation** from input **x(t)** and output **y(t)**:

Auto-spectra (power spectral density):
```
Gxx(f) = E[X(f) · X*(f)] / (fs · N)  (input PSD)
Gyy(f) = E[Y(f) · Y*(f)] / (fs · N)  (output PSD)
```

Cross-spectrum:
```
Gxy(f) = E[X(f) · Y*(f)] / (fs · N)
```

FRF estimators:
```
H1(f) = Gxy(f) / Gxx(f)  (minimizes output noise, preferred for impact testing)
H2(f) = Gyy(f) / Gyx(f)  (minimizes input noise)
Hv(f) = √[Gyy(f) / Gxx(f)]  (geometric mean, used when SNR is similar)
```

Coherence (quality metric, 0-1):
```
γ²(f) = |Gxy(f)|² / [Gxx(f) · Gyy(f)]
```

High coherence (>0.9) indicates linear, noise-free relationship. Low coherence suggests nonlinearity, noise, or extraneous inputs.

**LSCE Modal Extraction (Time-Domain)**

LSCE (Least Squares Complex Exponential) fits free-decay impulse response data to a sum of exponential modes:

```
h(t) = Σ_{r=1}^{2N} Ar · e^{λr·t}

where λr = -ζr·ωr ± jωr√(1 - ζr²)  (complex poles)
      Ar = complex mode amplitude
      N = number of modes
```

The algorithm constructs a Hankel matrix from impulse response samples and solves a linear least-squares problem to extract poles **λr**. From each pole:

```
Natural frequency: fr = |λr| / (2π)  (Hz)
Damping ratio:     ζr = -Re(λr) / |λr|
```

Mode shapes are computed by evaluating the residues **Ar** at each measurement DOF.

**PolyMAX Modal Extraction (Frequency-Domain)**

PolyMAX (Polyreference Least-Squares Complex Frequency-domain) is the gold standard for EMA, used in LMS Test.Lab. It fits a rational fraction polynomial to multi-reference FRF data:

```
H(jω) = [N(jω)] / [D(jω)]

where N(jω) = Σ_{k=0}^{n} Nk · (jω)^k  (numerator polynomial)
      D(jω) = Σ_{k=0}^{d} Dk · (jω)^k  (denominator polynomial, common to all refs)
```

The denominator roots are the system poles. PolyMAX formulates this as a weighted linear least-squares problem in the frequency domain, solved via SVD. The weighting emphasizes high-coherence frequency regions.

**Stabilization Diagram**

Modal extraction is repeated for increasing model orders (e.g., 10, 20, 30, ..., 200). Each order produces a set of poles. A pole is "stable" across orders if:

```
|fr(n) - fr(n+1)| / fr(n) < freq_tolerance (e.g., 1%)
|ζr(n) - ζr(n+1)| / ζr(n) < damp_tolerance (e.g., 5%)
MAC[Φr(n), Φr(n+1)] > mac_threshold (e.g., 0.95)
```

Stable poles cluster vertically on the stabilization diagram. Physical modes appear as stable vertical alignments; computational modes are scattered.

**MAC (Modal Assurance Criterion)**

MAC quantifies correlation between two mode shapes **Φ_A** and **Φ_B**:

```
MAC(Φ_A, Φ_B) = |Φ_A^H · Φ_B|² / [(Φ_A^H · Φ_A) · (Φ_B^H · Φ_B)]
```

MAC = 1 indicates identical shapes (same mode).
MAC = 0 indicates orthogonal shapes (different modes).
MAC > 0.9 is considered high correlation.

**Auto-MAC** compares modes from the same test (checks for orthogonality, should be identity matrix off-diagonal).
**Cross-MAC** compares test modes vs. FEA modes (validates simulation).

### Client/Server Split (WASM Threshold)

```
Data uploaded → File size checked
    │
    ├── ≤100 MB → WASM processing (browser)
    │   ├── Instant FFT, PSD, FRF (H1/H2)
    │   ├── Coherence quality check
    │   ├── Results displayed immediately
    │   └── No server cost
    │
    └── >100 MB OR modal extraction → Server processing (Rust native)
        ├── Job queued via Redis
        ├── Worker picks up, runs DSP + modal solver
        ├── Progress streamed via WebSocket
        └── Results stored in S3, URL returned
```

The 100 MB threshold was chosen because:
- 100 MB covers: 8 channels × 10 minutes × 2048 Hz = 9.8 MB; 16 channels × 30 minutes × 5120 Hz = 147 MB (borderline)
- WASM FFT handles 100 MB (12.5M samples per channel) in <5 seconds on modern hardware
- Modal extraction (LSCE, PolyMAX) always runs server-side due to memory and computation requirements (SVD on 100×100 FRF matrices)

### WASM Compilation Pipeline

```toml
# vibelab-dsp-wasm/Cargo.toml
[package]
name = "vibelab-dsp-wasm"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
wasm-bindgen = "0.2"
serde = { version = "1", features = ["derive"] }
serde-wasm-bindgen = "0.6"
js-sys = "0.3"
rustfft = "6"
realfft = "3"
num-complex = "0.4"
ndarray = { version = "0.15", default-features = false }
log = "0.4"
console_log = "1"

[dependencies.vibelab-dsp]
path = "../vibelab-dsp"
default-features = false

[profile.release]
opt-level = "z"       # Optimize for size
lto = true            # Link-time optimization
codegen-units = 1     # Single codegen unit
strip = true          # Strip debug symbols
wasm-opt = true       # Run wasm-opt
```

```yaml
# .github/workflows/wasm-build.yml
name: Build WASM DSP
on:
  push:
    paths: ['vibelab-dsp-wasm/**', 'vibelab-dsp/**']
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
      - run: cd vibelab-dsp-wasm && wasm-pack build --target web --release
      - run: wasm-opt -Oz vibelab-dsp-wasm/pkg/vibelab_dsp_wasm_bg.wasm -o vibelab-dsp-wasm/pkg/vibelab_dsp_wasm_bg.wasm
      - name: Upload to S3/CDN
        run: aws s3 cp vibelab-dsp-wasm/pkg/ s3://vibelab-wasm/v${{ github.sha }}/ --recursive
```

---

## Architecture Deep-Dives

### 1. FRF Computation API Handler (Rust/Axum)

The primary endpoint receives time-history data, validates channel count and file size, decides between WASM and server execution, and for server execution enqueues a job with Redis.

```rust
// src/api/handlers/frf.rs

use axum::{
    extract::{Path, State, Multipart},
    http::StatusCode,
    response::IntoResponse,
    Json,
};
use uuid::Uuid;
use crate::{
    db::models::{Measurement, Frf, FrfParams},
    state::AppState,
    auth::Claims,
    error::ApiError,
    dsp::parsers::{parse_csv, parse_uff, parse_tdms},
};

pub async fn upload_measurement(
    State(state): State<AppState>,
    claims: Claims,
    Path(setup_id): Path<Uuid>,
    mut multipart: Multipart,
) -> Result<impl IntoResponse, ApiError> {
    // 1. Verify setup ownership
    let setup = sqlx::query_as!(
        crate::db::models::Setup,
        "SELECT s.* FROM setups s
         JOIN campaigns c ON s.campaign_id = c.id
         WHERE s.id = $1 AND (c.owner_id = $2 OR c.org_id IN (
             SELECT org_id FROM org_members WHERE user_id = $2
         ))",
        setup_id,
        claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Setup not found"))?;

    // 2. Extract file from multipart upload
    let mut file_data: Vec<u8> = Vec::new();
    let mut file_format = String::new();
    let mut filename = String::new();

    while let Some(field) = multipart.next_field().await? {
        if field.name() == Some("file") {
            filename = field.file_name().unwrap_or("data.csv").to_string();
            file_data = field.bytes().await?.to_vec();

            // Detect format from extension
            file_format = if filename.ends_with(".uff") || filename.ends_with(".unv") {
                "uff"
            } else if filename.ends_with(".tdms") {
                "tdms"
            } else {
                "csv"
            }.to_string();
        }
    }

    let file_size = file_data.len() as i64;

    // 3. Check plan limits
    let user = sqlx::query_as!(
        crate::db::models::User,
        "SELECT * FROM users WHERE id = $1",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    if user.plan == "free" && file_size > 50 * 1024 * 1024 {
        return Err(ApiError::PlanLimit(
            "Free plan supports uploads up to 50 MB. Upgrade to Pro for unlimited."
        ));
    }

    // 4. Parse time-history data
    let time_data = match file_format.as_str() {
        "uff" => parse_uff(&file_data)?,
        "tdms" => parse_tdms(&file_data)?,
        _ => parse_csv(&file_data)?,
    };

    // Validate against setup
    if time_data.channel_count as i32 != setup.channel_count {
        return Err(ApiError::Validation(format!(
            "File has {} channels but setup expects {}",
            time_data.channel_count, setup.channel_count
        )));
    }

    // 5. Determine execution mode
    let execution_mode = if file_size <= 100 * 1024 * 1024 {
        "wasm"  // Client can handle it
    } else {
        "server"
    };

    // 6. Upload raw data to S3
    let s3_key = format!("measurements/{}/{}.bin", setup.campaign_id, Uuid::new_v4());
    state.s3.put_object()
        .bucket("vibelab-data")
        .key(&s3_key)
        .body(file_data.into())
        .content_type("application/octet-stream")
        .send()
        .await?;

    let data_url = format!("s3://vibelab-data/{}", s3_key);

    // 7. Create measurement record
    let measurement = sqlx::query_as!(
        Measurement,
        r#"INSERT INTO measurements
            (setup_id, user_id, status, execution_mode, data_url, data_format,
             file_size_bytes, sample_rate, duration, channel_count)
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10)
        RETURNING *"#,
        setup_id,
        claims.user_id,
        "uploaded",
        execution_mode,
        data_url,
        file_format,
        file_size,
        time_data.sample_rate,
        time_data.duration,
        time_data.channel_count as i32,
    )
    .fetch_one(&state.db)
    .await?;

    Ok((StatusCode::CREATED, Json(measurement)))
}

pub async fn compute_frf(
    State(state): State<AppState>,
    claims: Claims,
    Path(measurement_id): Path<Uuid>,
    Json(params): Json<FrfParams>,
) -> Result<impl IntoResponse, ApiError> {
    // 1. Verify measurement ownership
    let measurement = sqlx::query_as!(
        Measurement,
        "SELECT m.* FROM measurements m
         JOIN setups s ON m.setup_id = s.id
         JOIN campaigns c ON s.campaign_id = c.id
         WHERE m.id = $1 AND (c.owner_id = $2 OR c.org_id IN (
             SELECT org_id FROM org_members WHERE user_id = $2
         ))",
        measurement_id,
        claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Measurement not found"))?;

    // 2. Create FRF record
    let frf = sqlx::query_as!(
        Frf,
        r#"INSERT INTO frfs
            (measurement_id, estimator, window_type, overlap_pct, fft_size, averages,
             frequency_resolution, frf_data_url, frequency_range)
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9)
        RETURNING *"#,
        measurement_id,
        serde_json::to_string(&params.estimator)?,
        serde_json::to_string(&params.window_type)?,
        params.overlap_pct,
        params.fft_size as i32,
        params.averages as i32,
        measurement.sample_rate / params.fft_size as f32,
        format!("s3://vibelab-data/frfs/{}/frf.bin", Uuid::new_v4()),  // Placeholder
        params.frequency_range.unwrap_or([0.0, measurement.sample_rate / 2.0]).to_vec(),
    )
    .fetch_one(&state.db)
    .await?;

    // 3. For server execution, enqueue job
    if measurement.execution_mode == "server" {
        state.redis
            .publish("frf:jobs", serde_json::to_string(&frf.id)?)
            .await?;
    }

    Ok((StatusCode::CREATED, Json(frf)))
}
```

### 2. DSP Core — FRF Estimation (Rust — shared between WASM and native)

The core FRF computation that performs FFT, windowing, averaging, and coherence calculation. This code compiles to both native (server) and WASM (browser) targets.

```rust
// vibelab-dsp/src/frf.rs

use num_complex::Complex64;
use rustfft::{FftPlanner, num_complex::Complex};
use ndarray::{Array1, Array2};

pub struct FrfComputation {
    pub fft_size: usize,
    pub sample_rate: f32,
    pub overlap_pct: f32,
    pub averages: usize,
    pub window: Window,
    pub estimator: FrfEstimator,
}

#[derive(Clone)]
pub enum Window {
    Hanning,
    Hamming,
    Exponential { decay: f64 },
    Force,
    FlatTop,
    Rectangular,
}

pub enum FrfEstimator {
    H1,
    H2,
    Hv,
}

impl FrfComputation {
    /// Compute FRF from input (reference) and output (response) channels
    pub fn compute(
        &self,
        input: &[f32],
        output: &[f32],
    ) -> Result<FrfResult, DspError> {
        if input.len() != output.len() {
            return Err(DspError::LengthMismatch);
        }

        let hop_size = ((1.0 - self.overlap_pct / 100.0) * self.fft_size as f32) as usize;
        let n_segments = (input.len() - self.fft_size) / hop_size + 1;

        if n_segments < self.averages {
            return Err(DspError::InsufficientData(
                format!("Need {} samples for {} averages, have {}",
                    self.averages * hop_size + self.fft_size, self.averages, input.len())
            ));
        }

        // Generate window function
        let window_coeffs = self.generate_window();

        // Initialize accumulators for cross- and auto-spectra
        let n_freqs = self.fft_size / 2 + 1;
        let mut gxx_sum = vec![Complex64::new(0.0, 0.0); n_freqs];  // Input auto-spectrum
        let mut gyy_sum = vec![Complex64::new(0.0, 0.0); n_freqs];  // Output auto-spectrum
        let mut gxy_sum = vec![Complex64::new(0.0, 0.0); n_freqs];  // Cross-spectrum

        let mut planner = FftPlanner::new();
        let fft = planner.plan_fft_forward(self.fft_size);

        // Welch's method: segment, window, FFT, accumulate
        for i in 0..self.averages {
            let start = i * hop_size;
            let end = start + self.fft_size;

            // Extract and window segments
            let mut x_seg: Vec<Complex<f64>> = input[start..end].iter()
                .zip(&window_coeffs)
                .map(|(&val, &win)| Complex::new(val as f64 * win, 0.0))
                .collect();

            let mut y_seg: Vec<Complex<f64>> = output[start..end].iter()
                .zip(&window_coeffs)
                .map(|(&val, &win)| Complex::new(val as f64 * win, 0.0))
                .collect();

            // FFT
            fft.process(&mut x_seg);
            fft.process(&mut y_seg);

            // Accumulate one-sided spectra (0 to Nyquist)
            for k in 0..n_freqs {
                let x = x_seg[k];
                let y = y_seg[k];
                gxx_sum[k] += x * x.conj();
                gyy_sum[k] += y * y.conj();
                gxy_sum[k] += x * y.conj();
            }
        }

        // Average and normalize
        let norm_factor = (self.sample_rate * self.fft_size as f32 * self.averages as f32) as f64;
        let window_scale = window_coeffs.iter().map(|w| w * w).sum::<f64>();

        for k in 0..n_freqs {
            gxx_sum[k] /= norm_factor / window_scale;
            gyy_sum[k] /= norm_factor / window_scale;
            gxy_sum[k] /= norm_factor / window_scale;
        }

        // Compute FRF based on estimator
        let frf: Vec<Complex64> = match self.estimator {
            FrfEstimator::H1 => {
                gxy_sum.iter().zip(&gxx_sum)
                    .map(|(gxy, gxx)| gxy / gxx)
                    .collect()
            }
            FrfEstimator::H2 => {
                gyy_sum.iter().zip(&gxy_sum)
                    .map(|(gyy, gxy)| gyy / gxy.conj())
                    .collect()
            }
            FrfEstimator::Hv => {
                gyy_sum.iter().zip(&gxx_sum)
                    .map(|(gyy, gxx)| Complex64::new((gyy.re / gxx.re).sqrt(), 0.0))
                    .collect()
            }
        };

        // Compute coherence
        let coherence: Vec<f64> = gxy_sum.iter()
            .zip(&gxx_sum)
            .zip(&gyy_sum)
            .map(|((gxy, gxx), gyy)| {
                let num = gxy.norm_sqr();
                let den = gxx.re * gyy.re;
                if den > 1e-12 { num / den } else { 0.0 }
            })
            .collect();

        // Frequency axis
        let frequencies: Vec<f32> = (0..n_freqs)
            .map(|k| k as f32 * self.sample_rate / self.fft_size as f32)
            .collect();

        Ok(FrfResult {
            frequencies,
            frf,
            coherence,
            gxx: gxx_sum.iter().map(|g| g.re).collect(),
            gyy: gyy_sum.iter().map(|g| g.re).collect(),
            gxy: gxy_sum,
        })
    }

    fn generate_window(&self) -> Vec<f64> {
        let n = self.fft_size;
        match &self.window {
            Window::Hanning => {
                (0..n).map(|i| 0.5 * (1.0 - (2.0 * std::f64::consts::PI * i as f64 / (n - 1) as f64).cos())).collect()
            }
            Window::Hamming => {
                (0..n).map(|i| 0.54 - 0.46 * (2.0 * std::f64::consts::PI * i as f64 / (n - 1) as f64).cos()).collect()
            }
            Window::Exponential { decay } => {
                (0..n).map(|i| (-decay * i as f64 / n as f64).exp()).collect()
            }
            Window::Force => {
                // Force window: 0 to 1 linear ramp for first 10%, then 1.0
                let ramp_len = n / 10;
                (0..n).map(|i| {
                    if i < ramp_len {
                        i as f64 / ramp_len as f64
                    } else {
                        1.0
                    }
                }).collect()
            }
            Window::FlatTop => {
                // Flat-top window for accurate amplitude measurement
                let a0 = 0.21557895;
                let a1 = 0.41663158;
                let a2 = 0.277263158;
                let a3 = 0.083578947;
                let a4 = 0.006947368;
                (0..n).map(|i| {
                    let x = 2.0 * std::f64::consts::PI * i as f64 / (n - 1) as f64;
                    a0 - a1 * x.cos() + a2 * (2.0 * x).cos() - a3 * (3.0 * x).cos() + a4 * (4.0 * x).cos()
                }).collect()
            }
            Window::Rectangular => {
                vec![1.0; n]
            }
        }
    }
}

pub struct FrfResult {
    pub frequencies: Vec<f32>,
    pub frf: Vec<Complex64>,
    pub coherence: Vec<f64>,
    pub gxx: Vec<f64>,  // Input PSD
    pub gyy: Vec<f64>,  // Output PSD
    pub gxy: Vec<Complex64>,  // Cross-spectrum
}

#[derive(Debug)]
pub enum DspError {
    LengthMismatch,
    InsufficientData(String),
    InvalidParameters(String),
}
```

### 3. Mode Shape Visualization Component (React + Three.js)

The frontend 3D viewer that renders mode shapes with animated deflections on imported geometry using GPU-accelerated Three.js.

```typescript
// frontend/src/components/ModeShapeViewer.tsx

import { useRef, useEffect, useState, useMemo } from 'react';
import { Canvas, useFrame } from '@react-three/fiber';
import { OrbitControls, Line } from '@react-three/drei';
import * as THREE from 'three';
import { useModeStore } from '../stores/modeStore';

interface ModeShapeViewerProps {
  analysisId: string;
  modeNumber: number;
}

export function ModeShapeViewer({ analysisId, modeNumber }: ModeShapeViewerProps) {
  const { geometry, modeShape, selectedMode } = useModeStore();
  const [animationSpeed, setAnimationSpeed] = useState(1.0);
  const [amplitudeScale, setAmplitudeScale] = useState(10.0);
  const [colorMode, setColorMode] = useState<'magnitude' | 'phase'>('magnitude');

  return (
    <div className="mode-shape-viewer w-full h-full flex flex-col bg-gray-900">
      <ViewerControls
        speed={animationSpeed}
        onSpeedChange={setAnimationSpeed}
        amplitude={amplitudeScale}
        onAmplitudeChange={setAmplitudeScale}
        colorMode={colorMode}
        onColorModeChange={setColorMode}
      />
      <div className="flex-1 relative">
        <Canvas camera={{ position: [0, 0, 5], fov: 50 }}>
          <ambientLight intensity={0.4} />
          <directionalLight position={[10, 10, 5]} intensity={0.8} />
          <directionalLight position={[-10, -10, -5]} intensity={0.3} />

          <AnimatedModeShape
            geometry={geometry}
            modeShape={modeShape}
            speed={animationSpeed}
            amplitude={amplitudeScale}
            colorMode={colorMode}
          />

          <OrbitControls
            enablePan={true}
            enableZoom={true}
            enableRotate={true}
            minDistance={1}
            maxDistance={20}
          />

          <gridHelper args={[10, 10, 0x444444, 0x222222]} />
          <axesHelper args={[2]} />
        </Canvas>

        <ModeInfoOverlay
          frequency={selectedMode?.frequency_hz}
          damping={selectedMode?.damping_pct}
          complexity={selectedMode?.mode_complexity}
        />
      </div>
    </div>
  );
}

interface AnimatedModeShapeProps {
  geometry: GeometryData;
  modeShape: ModeShapeData;
  speed: number;
  amplitude: number;
  colorMode: 'magnitude' | 'phase';
}

function AnimatedModeShape({ geometry, modeShape, speed, amplitude, colorMode }: AnimatedModeShapeProps) {
  const meshRef = useRef<THREE.Mesh>(null);
  const timeRef = useRef(0);

  // Precompute base positions and mode displacements
  const { basePositions, modeDisplacements, colors } = useMemo(() => {
    const positions = new Float32Array(geometry.nodes.length * 3);
    const displacements = new Float32Array(geometry.nodes.length * 3);
    const nodeColors = new Float32Array(geometry.nodes.length * 3);

    const magnitudes: number[] = [];

    for (let i = 0; i < geometry.nodes.length; i++) {
      const node = geometry.nodes[i];
      const dof = modeShape.dofs[node.id];

      // Base position
      positions[i * 3 + 0] = node.x;
      positions[i * 3 + 1] = node.y;
      positions[i * 3 + 2] = node.z;

      // Mode displacement (complex -> real via magnitude and phase)
      if (dof) {
        displacements[i * 3 + 0] = dof.magnitude * Math.cos(dof.phase);
        displacements[i * 3 + 1] = dof.magnitude * Math.sin(dof.phase);
        displacements[i * 3 + 2] = 0;  // Assuming 2D mode shape; extend for 3D
        magnitudes.push(dof.magnitude);
      } else {
        displacements[i * 3 + 0] = 0;
        displacements[i * 3 + 1] = 0;
        displacements[i * 3 + 2] = 0;
        magnitudes.push(0);
      }
    }

    // Color mapping based on magnitude or phase
    const maxMag = Math.max(...magnitudes);
    for (let i = 0; i < geometry.nodes.length; i++) {
      const dof = modeShape.dofs[geometry.nodes[i].id];
      let colorValue = 0;

      if (colorMode === 'magnitude' && dof) {
        colorValue = dof.magnitude / maxMag;
      } else if (colorMode === 'phase' && dof) {
        colorValue = (dof.phase + Math.PI) / (2 * Math.PI);  // Normalize phase to [0, 1]
      }

      const color = new THREE.Color();
      color.setHSL(0.7 - colorValue * 0.7, 1.0, 0.5);  // Blue (cold) to red (hot)
      nodeColors[i * 3 + 0] = color.r;
      nodeColors[i * 3 + 1] = color.g;
      nodeColors[i * 3 + 2] = color.b;
    }

    return { basePositions: positions, modeDisplacements: displacements, colors: nodeColors };
  }, [geometry, modeShape, colorMode]);

  // Animation loop
  useFrame((state, delta) => {
    if (!meshRef.current) return;

    timeRef.current += delta * speed;
    const phase = Math.sin(timeRef.current * 2 * Math.PI);

    const positionAttr = meshRef.current.geometry.attributes.position;
    const positions = positionAttr.array as Float32Array;

    // Update vertex positions: base + amplitude * sin(ωt) * mode_displacement
    for (let i = 0; i < geometry.nodes.length; i++) {
      positions[i * 3 + 0] = basePositions[i * 3 + 0] + amplitude * phase * modeDisplacements[i * 3 + 0];
      positions[i * 3 + 1] = basePositions[i * 3 + 1] + amplitude * phase * modeDisplacements[i * 3 + 1];
      positions[i * 3 + 2] = basePositions[i * 3 + 2] + amplitude * phase * modeDisplacements[i * 3 + 2];
    }

    positionAttr.needsUpdate = true;
  });

  // Build geometry from nodes and elements
  const geometryBuffer = useMemo(() => {
    const geom = new THREE.BufferGeometry();
    geom.setAttribute('position', new THREE.BufferAttribute(basePositions, 3));
    geom.setAttribute('color', new THREE.BufferAttribute(colors, 3));

    // Build index from elements (triangles or lines)
    const indices: number[] = [];
    for (const elem of geometry.elements) {
      if (elem.type === 'line') {
        indices.push(elem.nodes[0], elem.nodes[1]);
      } else if (elem.type === 'triangle') {
        indices.push(elem.nodes[0], elem.nodes[1], elem.nodes[2]);
      }
    }
    geom.setIndex(indices);
    geom.computeVertexNormals();

    return geom;
  }, [geometry, basePositions, colors]);

  return (
    <mesh ref={meshRef} geometry={geometryBuffer}>
      <meshPhongMaterial
        vertexColors
        side={THREE.DoubleSide}
        wireframe={geometry.renderMode === 'wireframe'}
      />
    </mesh>
  );
}

interface GeometryData {
  nodes: Array<{ id: number; x: number; y: number; z: number }>;
  elements: Array<{ type: 'line' | 'triangle'; nodes: number[] }>;
  renderMode: 'wireframe' | 'surface';
}

interface ModeShapeData {
  dofs: Record<number, { magnitude: number; phase: number; real: number; imag: number }>;
}
```

### 4. Modal Extraction Worker — PolyMAX (Rust server-side)

Background worker that picks up modal extraction jobs from Redis, runs PolyMAX algorithm, builds stabilization diagram, and stores results in S3.

```rust
// src/workers/modal_worker.rs

use std::sync::Arc;
use aws_sdk_s3::Client as S3Client;
use redis::AsyncCommands;
use sqlx::PgPool;
use uuid::Uuid;
use nalgebra::{DMatrix, DVector, Complex};

use crate::{
    db::models::{ModalAnalysis, ModalExtractionParams},
    modal::{
        polymax::PolyMaxSolver,
        lsce::LsceSolver,
        stabilization::StabilizationDiagram,
    },
    state::AppState,
    websocket::ProgressBroadcaster,
};

pub struct ModalWorker {
    db: PgPool,
    redis: redis::Client,
    s3: S3Client,
    broadcaster: Arc<ProgressBroadcaster>,
}

impl ModalWorker {
    pub async fn run(&self) -> anyhow::Result<()> {
        let mut conn = self.redis.get_async_connection().await?;
        tracing::info!("Modal extraction worker started, listening for jobs...");

        loop {
            // Block-pop from Redis job queue
            let job_id: Option<String> = conn.brpop("modal:jobs", 30.0).await?;
            let Some(job_id) = job_id else { continue };
            let job_id: Uuid = job_id.parse()?;

            if let Err(e) = self.process_job(job_id).await {
                tracing::error!("Job {job_id} failed: {e}");
                self.mark_failed(job_id, &e.to_string()).await?;
            }
        }
    }

    async fn process_job(&self, job_id: Uuid) -> anyhow::Result<()> {
        // 1. Fetch job and analysis details
        let job = sqlx::query_as!(
            crate::db::models::ModalJob,
            "UPDATE modal_jobs SET started_at = NOW(), worker_id = $2
             WHERE id = $1 RETURNING *",
            job_id, hostname::get()?.to_string_lossy().to_string()
        )
        .fetch_one(&self.db)
        .await?;

        let analysis = sqlx::query_as!(
            ModalAnalysis,
            "UPDATE modal_analyses SET status = 'running', started_at = NOW()
             WHERE id = $1 RETURNING *",
            job.analysis_id
        )
        .fetch_one(&self.db)
        .await?;

        // 2. Load FRF data from S3
        let frf = sqlx::query_as!(
            crate::db::models::Frf,
            "SELECT * FROM frfs WHERE id = $1",
            analysis.frf_id.unwrap()
        )
        .fetch_one(&self.db)
        .await?;

        let frf_data = self.load_frf_from_s3(&frf.frf_data_url).await?;

        let params: ModalExtractionParams = serde_json::from_value(analysis.processing_params.clone())?;

        // 3. Run modal extraction based on algorithm
        let (poles, stabilization_data) = match params.algorithm {
            crate::db::models::ModalAlgorithm::PolyMax => {
                let solver = PolyMaxSolver::new(&frf_data, &params);
                solver.extract_modes(|progress| {
                    let _ = self.broadcaster.send(analysis.id, serde_json::json!({
                        "progress_pct": progress,
                        "stage": "polymax_curve_fitting"
                    }));
                })?
            }
            crate::db::models::ModalAlgorithm::Lsce => {
                let solver = LsceSolver::new(&frf_data, &params);
                solver.extract_modes(|progress| {
                    let _ = self.broadcaster.send(analysis.id, serde_json::json!({
                        "progress_pct": progress,
                        "stage": "lsce_pole_extraction"
                    }));
                })?
            }
            _ => anyhow::bail!("Unsupported algorithm: {:?}", params.algorithm),
        };

        // 4. Build stabilization diagram
        let stab_diagram = StabilizationDiagram::new(poles, &params.stability_criteria);
        let stable_poles = stab_diagram.select_stable_poles();

        // 5. Upload stabilization data to S3
        let stab_s3_key = format!("modal/{}/{}_stab.json", analysis.campaign_id, analysis.id);
        let stab_json = serde_json::to_vec(&stabilization_data)?;
        self.s3.put_object()
            .bucket("vibelab-data")
            .key(&stab_s3_key)
            .body(stab_json.into())
            .content_type("application/json")
            .send()
            .await?;

        let stab_url = format!("s3://vibelab-data/{}", stab_s3_key);

        // 6. Store selected poles
        let selected_poles_json = serde_json::to_value(&stable_poles)?;

        // 7. Update database with completed status
        sqlx::query!(
            "UPDATE modal_analyses SET status = 'completed', stabilization_data_url = $2,
             selected_poles = $3, completed_at = NOW(),
             wall_time_ms = EXTRACT(EPOCH FROM (NOW() - started_at))::int * 1000
             WHERE id = $1",
            analysis.id, stab_url, selected_poles_json
        )
        .execute(&self.db)
        .await?;

        // 8. Insert individual modes into modes table
        for (i, pole) in stable_poles.iter().enumerate() {
            sqlx::query!(
                r#"INSERT INTO modes
                    (analysis_id, mode_number, frequency_hz, damping_ratio, damping_pct,
                     modal_phase, mode_complexity, mpd)
                VALUES ($1, $2, $3, $4, $5, $6, $7, $8)"#,
                analysis.id,
                i as i32 + 1,
                pole.frequency,
                pole.damping_ratio,
                pole.damping_ratio * 100.0,
                pole.phase,
                pole.complexity,
                pole.mpd,
            )
            .execute(&self.db)
            .await?;
        }

        sqlx::query!(
            "UPDATE modal_jobs SET completed_at = NOW(), progress_pct = 100.0
             WHERE id = $1",
            job_id
        )
        .execute(&self.db)
        .await?;

        // 9. Notify client via WebSocket
        self.broadcaster.send(analysis.id, serde_json::json!({
            "status": "completed",
            "mode_count": stable_poles.len(),
        }))?;

        tracing::info!("Modal extraction job {job_id} completed with {} modes", stable_poles.len());
        Ok(())
    }

    async fn load_frf_from_s3(&self, s3_url: &str) -> anyhow::Result<FrfMatrix> {
        // Parse S3 URL and download data
        let key = s3_url.strip_prefix("s3://vibelab-data/").unwrap();
        let response = self.s3.get_object()
            .bucket("vibelab-data")
            .key(key)
            .send()
            .await?;

        let bytes = response.body.collect().await?.into_bytes();

        // Deserialize FRF matrix (custom binary format or MessagePack/bincode)
        let frf_matrix: FrfMatrix = bincode::deserialize(&bytes)?;
        Ok(frf_matrix)
    }

    async fn mark_failed(&self, job_id: Uuid, error: &str) -> anyhow::Result<()> {
        sqlx::query!(
            "UPDATE modal_jobs SET completed_at = NOW() WHERE id = $1",
            job_id
        )
        .execute(&self.db)
        .await?;

        let analysis_id: Uuid = sqlx::query_scalar!(
            "SELECT analysis_id FROM modal_jobs WHERE id = $1",
            job_id
        )
        .fetch_one(&self.db)
        .await?;

        sqlx::query!(
            "UPDATE modal_analyses SET status = 'failed', error_message = $2,
             completed_at = NOW() WHERE id = $1",
            analysis_id, error
        )
        .execute(&self.db)
        .await?;

        Ok(())
    }
}

#[derive(serde::Deserialize, serde::Serialize)]
pub struct FrfMatrix {
    pub frequencies: Vec<f32>,
    pub frf: Vec<Vec<Complex<f64>>>,  // [channels][freqs]
    pub coherence: Vec<Vec<f64>>,
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–5)

**Day 1: Project initialization and Rust backend scaffold**
```bash
cargo init vibelab-api
cd vibelab-api
cargo add axum tokio serde serde_json sqlx uuid chrono tower-http jsonwebtoken bcrypt
cargo add --features postgres,runtime-tokio-rustls,uuid,chrono,json sqlx
cargo add aws-sdk-s3 redis rustfft nalgebra hdf5
```
- `src/main.rs` — Axum app with CORS, tracing, graceful shutdown
- `src/config.rs` — Environment-based configuration (DATABASE_URL, JWT_SECRET, S3_BUCKET, REDIS_URL)
- `src/state.rs` — AppState struct (PgPool, Redis, S3Client)
- `src/error.rs` — ApiError enum with IntoResponse
- `Dockerfile` — Multi-stage build (builder + runtime)
- `docker-compose.yml` — PostgreSQL+TimescaleDB, Redis, MinIO (S3-compatible)

**Day 2: Database schema and migrations**
- `migrations/001_initial.sql` — All 17 tables: users, organizations, org_members, campaigns, setups, measurements, frfs, modal_analyses, modal_jobs, modes, mac_matrices, ods_analyses, monitoring_structures, modal_tracking (hypertable), reports, usage_records
- `migrations/002_timescale.sql` — Create hypertable for modal_tracking, continuous aggregates
- `src/db/mod.rs` — Database pool initialization with TimescaleDB extension check
- `src/db/models.rs` — All SQLx structs with FromRow derives
- Run `sqlx migrate run` to apply schema
- Seed script for test geometries and sample data

**Day 3: Authentication system**
- `src/auth/mod.rs` — JWT token generation and validation middleware
- `src/auth/oauth.rs` — Google and GitHub OAuth 2.0 flow handlers
- `src/api/handlers/auth.rs` — Register, login, OAuth callback, refresh token, me endpoints
- Password hashing with bcrypt (cost factor 12)
- JWT with 24h access token + 30d refresh token
- Auth middleware that extracts Claims from Authorization header

**Day 4: User and campaign CRUD**
- `src/api/handlers/users.rs` — Get profile, update profile, delete account
- `src/api/handlers/campaigns.rs` — Create, list, get, update, delete campaign
- `src/api/handlers/orgs.rs` — Create org, invite member, list members, remove member
- `src/api/router.rs` — All route definitions with auth middleware
- Integration tests: auth flow, campaign CRUD, authorization checks

**Day 5: Setup and measurement data import**
- `src/api/handlers/setups.rs` — Create/update/delete measurement setup (sensor placement, DAQ config)
- `src/api/handlers/measurements.rs` — Upload time-history data (multipart), list measurements
- `src/dsp/parsers/csv.rs` — CSV time-history parser (supports headers, auto-detect columns)
- `src/dsp/parsers/uff.rs` — Universal File Format (UFF/UNV) parser for test lab interoperability
- `src/dsp/parsers/tdms.rs` — National Instruments TDMS format parser (via tdms crate)
- S3 upload for raw time-history data with progress tracking

### Phase 2 — DSP Core + WASM (Days 6–13)

**Day 6: FFT and windowing foundation**
- `vibelab-dsp/` — New Rust workspace member (shared between native and WASM)
- `vibelab-dsp/src/fft.rs` — FFT wrapper using rustfft, real FFT optimization via realfft crate
- `vibelab-dsp/src/window.rs` — Window functions: Hanning, Hamming, Exponential, Force, Flat-top, Rectangular
- `vibelab-dsp/src/spectrum.rs` — Auto-spectrum (PSD) and cross-spectrum (CSD) computation with Welch's method
- Unit tests: verify FFT output, window energy normalization, PSD units

**Day 7: FRF estimation**
- `vibelab-dsp/src/frf.rs` — FRF computation (H1, H2, Hv estimators) from time-domain data
- Overlap processing with configurable overlap percentage (default 50%)
- Coherence computation (0-1 quality metric)
- Averaging across multiple segments
- Tests: white noise input → coherence = 1, known transfer function → FRF matches analytical

**Day 8: Filtering and signal conditioning**
- `vibelab-dsp/src/filter.rs` — Butterworth, Chebyshev Type I/II, Bessel IIR filters
- Biquad implementation for low-pass, high-pass, band-pass, band-stop
- FIR filter design via Parks-McClellan (remez) for linear phase
- Resampling: decimation, interpolation for sample rate conversion
- Detrending: remove mean, remove linear trend
- Tests: filter frequency response matches specification

**Day 9: Order tracking and envelope analysis (for rotating machinery)**
- `vibelab-dsp/src/order_tracking.rs` — Computed order tracking (COT) via resampling to angular domain
- Tachometer signal integration for angle-domain conversion
- `vibelab-dsp/src/envelope.rs` — Envelope analysis via Hilbert transform for bearing fault detection
- Cepstrum computation for gear/bearing diagnostics
- Tests: synthetic vibration signal with known orders → correctly extracted

**Day 10: Spectral analysis utilities**
- `vibelab-dsp/src/spectrogram.rs` — Short-time Fourier transform (STFT) for time-frequency representation
- Wavelet transform (continuous wavelet transform using Morlet wavelet)
- Octave and 1/3-octave band analysis for vibration severity per ISO 10816/20816
- Peak detection in spectra with configurable prominence and width thresholds
- Tests: chirp signal → spectrogram shows frequency sweep

**Day 11: WASM compilation pipeline**
- `vibelab-dsp-wasm/` — New workspace member for WASM target
- `vibelab-dsp-wasm/Cargo.toml` — wasm-bindgen, serde-wasm-bindgen, console_log dependencies
- `vibelab-dsp-wasm/src/lib.rs` — WASM entry points: `compute_frf()`, `compute_spectrogram()`, `apply_filter()`
- `wasm-pack build --target web --release` pipeline
- `wasm-opt -Oz` for size optimization (target <1.5MB gzipped)
- JavaScript wrapper: `VibeDsp` class that loads WASM and provides async API

**Day 12: WASM integration testing**
- Headless browser test (Playwright): load WASM, compute FRF, verify results match native
- Memory leak test: run 100 consecutive FFT computations, verify heap doesn't grow
- Performance benchmark: measure WASM FFT time for various FFT sizes (1024, 4096, 16384, 65536)
- Cross-browser test: Chrome, Firefox, Safari (via BrowserStack)

**Day 13: Client-side FRF computation API endpoint**
- `src/api/handlers/frf.rs` — Upload measurement → compute FRF endpoint
- For execution_mode = "wasm": return measurement ID, let client compute FRF
- For execution_mode = "server": enqueue Redis job
- Frontend: detect file size → route to WASM or server
- WASM progress callback for large FFTs

### Phase 3 — Modal Extraction Solvers (Days 14–21)

**Day 14: LSCE (Least Squares Complex Exponential) solver**
- `vibelab-modal/src/lsce.rs` — LSCE algorithm implementation
- Build Hankel matrix from impulse response function (IRF)
- Solve linear least-squares for complex exponential coefficients
- Extract poles (complex eigenvalues) from companion matrix
- Compute natural frequencies and damping ratios from poles
- Tests: synthetic 2-DOF system → extract 2 modes with <1% error

**Day 15: PolyMAX solver — polynomial construction**
- `vibelab-modal/src/polymax.rs` — PolyMAX algorithm (Polyreference Least-Squares Complex Frequency)
- Construct right matrix fraction model from FRF data
- Formulate weighted least-squares problem in frequency domain
- Common denominator polynomial represents system poles
- Weights based on coherence (high coherence = high weight)
- SVD-based solution for numerically stable pole extraction

**Day 16: PolyMAX solver — mode shape extraction**
- Mode shape computation from polynomial residues
- Participation factor scaling for consistent mode magnitude
- Modal phase and complexity indicators (MPC, MPD)
- Multi-reference data handling (MIMO FRF matrix)
- Tests: 3-DOF analytical FRF → extract 3 modes, MAC > 0.99 vs. analytical mode shapes

**Day 17: Stabilization diagram construction**
- `vibelab-modal/src/stabilization.rs` — Stabilization diagram builder
- Run modal extraction for orders 10, 20, 30, ..., 200 (adaptive based on frequency range)
- Track poles across orders with frequency/damping/MAC criteria
- Classify poles: stable (s), new (n), frequency-stable (f), damping-stable (d)
- Automated clustering of stable poles for final mode selection
- Tests: synthetic data with 5 modes → stabilization diagram shows 5 vertical stable clusters

**Day 18: Automated pole selection**
- `vibelab-modal/src/pole_selection.rs` — Automatic pole selection from stabilization diagram
- K-means or DBSCAN clustering on stable poles
- Merge clusters within user-defined frequency tolerance
- Select representative pole from each cluster (median frequency/damping)
- Uncertainty quantification: confidence intervals from cluster spread
- Tests: stabilization diagram with noise → correct number of modes selected

**Day 19: MAC matrix computation**
- `vibelab-modal/src/mac.rs` — Modal Assurance Criterion (MAC) calculation
- Auto-MAC: compare mode shapes within same test (should be near-identity matrix)
- Cross-MAC: compare test mode shapes vs. FEA mode shapes
- Optimized complex dot products using nalgebra BLAS backend
- Correlation threshold detection for mode pairing
- Tests: orthogonal mode shapes → MAC matrix is identity

**Day 20: OMA (Operational Modal Analysis) — FDD**
- `vibelab-modal/src/oma/fdd.rs` — Frequency Domain Decomposition
- Compute cross-spectral density matrix from output-only data
- SVD at each frequency bin → extract singular values and vectors
- Peak detection in first singular value plot → natural frequencies
- EFDD (Enhanced FDD): fit SDOF function to singular value curve for damping estimation
- Tests: white noise excitation → FDD extracts modes, compare to known analytical system

**Day 21: OMA — SSI-cov (Stochastic Subspace Identification)**
- `vibelab-modal/src/oma/ssi_cov.rs` — Covariance-driven SSI
- Compute correlation matrices from time-domain data
- Build Toeplitz matrix and perform SVD for state-space identification
- Extract state-space A matrix eigenvalues → poles
- Stabilization diagram for SSI similar to PolyMAX
- Tests: ambient vibration data → SSI extracts modes, validate against impact test EMA results

### Phase 4 — Frontend Visualization (Days 22–28)

**Day 22: Frontend scaffold and state management**
```bash
npm create vite@latest frontend -- --template react-ts
cd frontend
npm i zustand @tanstack/react-query axios d3 three @react-three/fiber @react-three/drei
```
- `src/App.tsx` — Router, auth context, layout
- `src/stores/authStore.ts` — Auth state (JWT, user)
- `src/stores/campaignStore.ts` — Campaign and setup state
- `src/stores/frfStore.ts` — FRF data and plot state
- `src/stores/modeStore.ts` — Modal analysis results and 3D visualization state
- `src/lib/api.ts` — Axios API client with JWT interceptor

**Day 23: FRF plot component (D3.js)**
- `src/components/FrfPlot.tsx` — D3.js line chart for FRF magnitude and phase
- Log-log or linear-log frequency axis (user-selectable)
- Multi-channel FRF overlay with color coding
- Coherence subplot or overlay
- Zoom/pan with d3-zoom
- Cursor readout with frequency, magnitude, phase values
- Export to PNG via canvas rendering

**Day 24: Stabilization diagram component**
- `src/components/StabilizationDiagram.tsx` — D3.js scatter plot
- X-axis: frequency, Y-axis: model order
- Symbol coding: stable (s), new (n), freq-stable (f), damp-stable (d)
- Interactive pole selection: click pole → add to selected modes
- Highlight stable vertical alignments
- Auto-selected modes overlay (clustered poles)

**Day 25: 3D mode shape viewer — geometry import**
- `src/components/ModeShapeViewer.tsx` — Three.js/React Three Fiber canvas
- Geometry parser: STL, UFF (UNV dataset 2411 nodes, 2412 elements), OBJ
- Node mapping: match measurement DOF IDs to geometry node IDs
- Wireframe, surface, and point cloud rendering modes
- Lighting: ambient + directional lights for shading
- Grid and axes helper for orientation

**Day 26: 3D mode shape viewer — animation**
- Vertex shader-based deformation: `position = basePosition + amplitude * sin(ωt) * modeShape`
- Animation speed control slider (0.1x to 5x real-time)
- Amplitude scale control (auto-scale to 10% of geometry bounding box)
- Color mapping: magnitude (blue=low, red=high) or phase (hue=phase angle)
- Side-by-side comparison: test mode vs. FEA mode in split viewport
- OrbitControls for pan/zoom/rotate

**Day 27: Mode shape viewer — MAC overlay and comparison**
- MAC matrix heatmap display (D3.js)
- Click MAC cell → load corresponding mode pair in side-by-side view
- Automatic mode pairing: highlight best-match mode pairs (MAC > 0.9)
- Mode shape difference visualization: `Δ = testMode - feaMode` rendered as color map
- Export mode shape animation to MP4 (via canvas capture + ffmpeg.wasm)

**Day 28: ODS (Operational Deflection Shape) viewer**
- `src/components/OdsViewer.tsx` — Similar to mode shape viewer but for ODS at specific frequency
- Frequency selector with FRF reference plot
- Real-time ODS computation from cross-spectral data
- Magnitude and phase visualization
- Compare ODS at multiple frequencies (e.g., operating speed harmonics)

### Phase 5 — Server-Side Processing + Job Queue (Days 29–33)

**Day 29: FRF computation worker**
- `src/workers/frf_worker.rs` — Redis job consumer for server-side FRF computation
- Load time-history data from S3
- Compute FRF using `vibelab-dsp` library
- Store FRF matrix, coherence, auto/cross-spectra in S3 (HDF5 format for efficient multi-dimensional arrays)
- Update database with FRF record and quality metrics
- Progress streaming via Redis pub/sub → WebSocket to client

**Day 30: Modal extraction worker — job orchestration**
- `src/workers/modal_worker.rs` — Redis job consumer for modal extraction
- Load FRF data from S3
- Run LSCE or PolyMAX based on parameters
- Build stabilization diagram for orders 10-200
- Automatic pole selection (clustering)
- Store stabilization data and selected modes in PostgreSQL + S3
- Insert individual modes into `modes` table

**Day 31: WebSocket live progress streaming**
- `src/api/ws/mod.rs` — Axum WebSocket upgrade handler
- `src/api/ws/progress.rs` — Subscribe to job progress channel (Redis pub/sub)
- Client receives: `{ job_id, stage, progress_pct, message, current_frequency }`
- Connection management: heartbeat ping/pong, automatic cleanup
- Frontend: `src/hooks/useJobProgress.ts` — React hook for WebSocket subscription with auto-reconnect

**Day 32: Geometry upload and storage**
- `src/api/handlers/geometry.rs` — Upload STL/UFF/OBJ geometry file
- Parse geometry to extract node count, element count, bounding box
- Store binary geometry in S3
- Store metadata (node_count, element_count, units, coordinate_system) in campaign record
- Geometry validation: check for disconnected nodes, degenerate elements
- Thumbnail generation: server-side rendering of wireframe view (via headless Three.js or rasterization)

**Day 33: Report generation**
- `src/services/report.rs` — PDF report generation via typst or headless Chrome + Puppeteer
- Report templates: modal test report, MAC correlation report, ODS report
- Include: FRF plots, coherence plots, stabilization diagram, mode table (freq/damp/complexity), mode shape renderings
- S3 upload and presigned URL for download
- Report generation as background job (queued via Redis)

### Phase 6 — Billing + Plan Enforcement (Days 34–36)

**Day 34: Stripe integration**
- `src/api/handlers/billing.rs` — Create checkout session, create customer portal session
- `src/api/handlers/webhooks/stripe.rs` — Handle Stripe webhooks: subscription created/updated/deleted, payment failed
- Plan mapping:
  - Free: 3 projects, 8 channels, 50 MB storage, peak-picking only
  - Pro ($119/mo): Unlimited projects/channels, full EMA (LSCE, PolyMAX), 10 GB storage
  - Advanced ($299/mo): OMA, continuous monitoring, DAQ streaming, 100 GB storage, API access
- Update `users.plan` on successful subscription

**Day 35: Usage tracking and limits**
- `src/middleware/plan_limits.rs` — Middleware checking plan limits before job creation
- Track storage usage: sum file sizes on S3, update `users.storage_used_bytes` on upload/delete
- Usage record insertion for billing period: storage_bytes, server_processing_minutes, modal_extractions
- Usage dashboard endpoint: `src/api/handlers/usage.rs` — Current period usage
- Approaching-limit warnings at 80% and 100%

**Day 36: Feature gating and upgrade prompts**
- Gate modal extraction (PolyMAX, LSCE) behind Pro plan
- Gate OMA algorithms (SSI, FDD) behind Advanced plan
- Gate continuous monitoring behind Advanced plan
- Gate API access behind Advanced plan
- Show locked feature indicators in UI with upgrade CTAs
- Upgrade prompt modals when user tries to use locked feature

### Phase 7 — Validation + Testing (Days 37–40)

**Day 37: DSP validation — analytical benchmarks**
- Benchmark 1: Sine wave FFT → verify peak at known frequency, amplitude accurate to <0.1%
- Benchmark 2: White noise PSD → verify flat spectrum within ±1 dB
- Benchmark 3: Known transfer function (RC filter) → H1 FRF matches analytical H(f) = 1/(1+j2πfRC) to <1%
- Benchmark 4: Perfect coherence test → two copies of same signal → coherence = 1.0
- Automated test suite: `vibelab-dsp/tests/benchmarks.rs`

**Day 38: Modal extraction validation — synthetic systems**
- Benchmark 5: 2-DOF mass-spring-damper → LSCE extracts 2 modes, frequency error <0.5%, damping error <2%
- Benchmark 6: 5-DOF analytical FRF → PolyMAX extracts 5 modes, MAC vs. analytical mode shapes >0.98
- Benchmark 7: FDD on white-noise-excited 3-DOF system → 3 modes extracted, frequency error <1%
- Benchmark 8: SSI-cov on output-only data → modes match EMA reference, MAC >0.95
- Compare against MATLAB/Python reference implementations (SciPy, pyOMA)

**Day 39: Integration and end-to-end testing**
- E2E test 1: Upload CSV → compute FRF via WASM → results match server computation
- E2E test 2: Upload large TDMS → server FRF job → WebSocket progress → results in S3 → download
- E2E test 3: Compute FRF → run PolyMAX → stabilization diagram → select modes → mode table
- E2E test 4: Upload geometry → run modal analysis → 3D mode shape animation → export MP4
- E2E test 5: Compute MAC matrix → cross-MAC test vs. FEA → pairing suggestions
- Concurrent user test: 20 simultaneous users running modal extractions

**Day 40: Performance testing and optimization**
- WASM FFT benchmark: 16384-point FFT < 50ms in Chrome
- Server modal extraction: 32-channel FRF, 100 frequency points, PolyMAX order 100 < 60s on 4-core instance
- 3D rendering: 500-node mode shape animation at 60 FPS
- API latency: p95 < 300ms for all endpoints
- Load test: 100 concurrent WebSocket connections with progress streaming
- Memory profiling: modal worker processes <2GB memory for typical jobs

### Phase 8 — Deployment + Polish + Launch (Days 41–42)

**Day 41: Docker and Kubernetes configuration**
- `Dockerfile` — Multi-stage Rust build (cargo-chef for layer caching)
- `docker-compose.yml` — Full local dev stack (API, workers, PostgreSQL+TimescaleDB, Redis, MinIO)
- `k8s/api-deployment.yaml` — API server (3 replicas, HPA on CPU)
- `k8s/worker-deployment.yaml` — Modal extraction workers (HPA on Redis queue depth)
- `k8s/postgres-statefulset.yaml` — PostgreSQL+TimescaleDB with persistent volume
- `k8s/redis-deployment.yaml` — Redis cluster
- `k8s/ingress.yaml` — NGINX ingress with TLS (cert-manager)
- Health check endpoints: `/health/live`, `/health/ready`

**Day 42: Monitoring, polish, and launch**
- Prometheus metrics: job duration histogram, FRF computation time, mode extraction success rate
- Grafana dashboards: system health, job queue depth, API latency, storage usage
- Sentry error tracking for frontend and backend
- UI polish: loading states, error messages, empty states, help tooltips
- Keyboard shortcuts: Ctrl+S save, Ctrl+Z/Y undo/redo in stabilization diagram pole selection
- Accessibility: ARIA labels, keyboard navigation
- Landing page with product demo video
- Documentation: getting started, FRF estimation guide, modal analysis tutorial
- Deploy to production, smoke test, announce launch

---

## Critical Files

```
vibelab/
├── vibelab-dsp/                           # Shared DSP library (native + WASM)
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── fft.rs                         # FFT wrapper (rustfft, realfft)
│   │   ├── window.rs                      # Window functions
│   │   ├── spectrum.rs                    # PSD, CSD computation
│   │   ├── frf.rs                         # FRF estimation (H1, H2, Hv)
│   │   ├── filter.rs                      # IIR/FIR filters
│   │   ├── order_tracking.rs              # Computed order tracking
│   │   ├── envelope.rs                    # Envelope analysis
│   │   └── spectrogram.rs                 # STFT, wavelets
│   └── tests/
│       └── benchmarks.rs                  # DSP validation benchmarks
│
├── vibelab-dsp-wasm/                      # WASM compilation target
│   ├── Cargo.toml
│   └── src/
│       └── lib.rs                         # WASM entry points (wasm-bindgen)
│
├── vibelab-modal/                         # Modal extraction library (server-side only)
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── lsce.rs                        # LSCE time-domain modal extraction
│   │   ├── polymax.rs                     # PolyMAX frequency-domain extraction
│   │   ├── stabilization.rs               # Stabilization diagram construction
│   │   ├── pole_selection.rs              # Automated pole selection
│   │   ├── mac.rs                         # MAC matrix computation
│   │   └── oma/
│   │       ├── fdd.rs                     # Frequency Domain Decomposition
│   │       ├── efdd.rs                    # Enhanced FDD
│   │       └── ssi_cov.rs                 # SSI covariance-driven
│   └── tests/
│       └── synthetic_systems.rs           # Modal extraction validation
│
├── vibelab-api/                           # Rust API server
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
│   │   │   │   ├── auth.rs
│   │   │   │   ├── users.rs
│   │   │   │   ├── campaigns.rs
│   │   │   │   ├── setups.rs
│   │   │   │   ├── measurements.rs        # Data upload
│   │   │   │   ├── frf.rs                 # FRF computation endpoints
│   │   │   │   ├── modal.rs               # Modal extraction endpoints
│   │   │   │   ├── geometry.rs            # Geometry upload
│   │   │   │   ├── reports.rs             # Report generation
│   │   │   │   ├── billing.rs
│   │   │   │   ├── usage.rs
│   │   │   │   └── webhooks/
│   │   │   │       └── stripe.rs
│   │   │   └── ws/
│   │   │       ├── mod.rs
│   │   │       └── progress.rs            # Job progress WebSocket
│   │   ├── middleware/
│   │   │   └── plan_limits.rs
│   │   ├── services/
│   │   │   ├── report.rs                  # PDF generation
│   │   │   └── s3.rs
│   │   ├── dsp/
│   │   │   └── parsers/
│   │   │       ├── csv.rs
│   │   │       ├── uff.rs
│   │   │       └── tdms.rs
│   │   └── workers/
│   │       ├── mod.rs
│   │       ├── frf_worker.rs              # Server-side FRF computation
│   │       └── modal_worker.rs            # Server-side modal extraction
│   ├── migrations/
│   │   ├── 001_initial.sql
│   │   └── 002_timescale.sql
│   └── tests/
│       ├── api_integration.rs
│       └── e2e.rs
│
├── frontend/                              # React frontend
│   ├── package.json
│   ├── vite.config.ts
│   ├── src/
│   │   ├── App.tsx
│   │   ├── main.tsx
│   │   ├── stores/
│   │   │   ├── authStore.ts
│   │   │   ├── campaignStore.ts
│   │   │   ├── frfStore.ts
│   │   │   └── modeStore.ts
│   │   ├── hooks/
│   │   │   ├── useJobProgress.ts          # WebSocket job progress
│   │   │   └── useWasmDsp.ts              # WASM DSP loader
│   │   ├── lib/
│   │   │   ├── api.ts
│   │   │   └── wasmLoader.ts
│   │   ├── pages/
│   │   │   ├── Dashboard.tsx
│   │   │   ├── CampaignEditor.tsx         # Main analysis workspace
│   │   │   ├── Billing.tsx
│   │   │   ├── Login.tsx
│   │   │   └── Register.tsx
│   │   ├── components/
│   │   │   ├── FrfPlot.tsx                # D3.js FRF magnitude/phase plot
│   │   │   ├── CoherencePlot.tsx          # D3.js coherence plot
│   │   │   ├── StabilizationDiagram.tsx   # D3.js stabilization diagram
│   │   │   ├── ModeShapeViewer.tsx        # Three.js 3D mode shape animation
│   │   │   ├── OdsViewer.tsx              # Three.js ODS visualization
│   │   │   ├── MacMatrix.tsx              # D3.js MAC heatmap
│   │   │   ├── ModeTable.tsx              # Mode parameter table
│   │   │   ├── SpectrogramPlot.tsx        # D3.js spectrogram (STFT)
│   │   │   ├── UploadMeasurement.tsx      # File upload with progress
│   │   │   ├── SetupEditor.tsx            # Sensor placement config
│   │   │   ├── GeometryUpload.tsx         # STL/UFF geometry import
│   │   │   ├── ReportGenerator.tsx        # Report config and generation
│   │   │   └── billing/
│   │   │       ├── PlanCard.tsx
│   │   │       └── UsageMeter.tsx
│   │   └── utils/
│   │       ├── geometryParser.ts          # STL/UFF/OBJ parsers
│   │       └── formatters.ts              # Engineering notation, units
│   └── public/
│       └── wasm/                          # WASM DSP bundle
│
├── ml-service/                            # Python FastAPI (ML/AI services)
│   ├── requirements.txt
│   ├── main.py
│   ├── services/
│   │   ├── anomaly_detection.py           # Isolation forest, autoencoder
│   │   ├── pole_clustering.py             # K-means/DBSCAN for pole selection
│   │   └── environmental_correction.py    # ARX models for temp/load correction
│   └── Dockerfile
│
├── k8s/
│   ├── api-deployment.yaml
│   ├── worker-deployment.yaml
│   ├── postgres-statefulset.yaml
│   ├── redis-deployment.yaml
│   ├── ml-service-deployment.yaml
│   └── ingress.yaml
│
├── docker-compose.yml
├── Cargo.toml                             # Workspace root
└── .github/
    └── workflows/
        ├── ci.yml
        ├── wasm-build.yml
        └── deploy.yml
```

---

## Solver Validation Suite

### Benchmark 1: White Noise PSD (FFT Validation)

**Input:** 65536 samples of Gaussian white noise, sample rate 1024 Hz, standard deviation σ = 1.0

**Expected:** Flat PSD across all frequencies, magnitude ≈ **σ² / (fs/2) = 1.0² / 512 = 0.00195 V²/Hz** (±1 dB variation due to random sampling)

**Tolerance:** PSD mean within 10% of theoretical, standard deviation of PSD across frequency bins < 20% of mean

### Benchmark 2: FRF of Known System — RC Low-Pass Filter

**System:** R = 10 kΩ, C = 1 µF, cutoff frequency fc = 1/(2πRC) = **15.92 Hz**

**Input:** Swept sine excitation 1-100 Hz, 1024 Hz sample rate

**Expected H1(f) at fc:** Magnitude = **-3.01 dB** (0.707), Phase = **-45°**

**Expected H1(100 Hz):** Magnitude ≈ **-18 dB** (0.126), Phase ≈ **-81°**

**Tolerance:** Magnitude < 0.5 dB error, Phase < 3° error, Coherence > 0.98 across all frequencies

### Benchmark 3: 2-DOF System Modal Extraction (LSCE + PolyMAX)

**System:** Two masses (m1 = m2 = 1 kg) connected by springs (k1 = k2 = k3 = 1000 N/m), damping ζ = 2% on both modes

**Analytical natural frequencies:**
- Mode 1: **f1 = 5.03 Hz** (in-phase)
- Mode 2: **f2 = 12.25 Hz** (out-of-phase)

**Expected damping ratio:** **ζ = 2.0%** for both modes

**Expected MAC (extracted vs. analytical):** **> 0.98** for both mode shapes

**Tolerance:** Frequency error < 0.5%, Damping error < 5% (damping is harder to extract accurately), MAC > 0.98

### Benchmark 4: 5-DOF Chain System (PolyMAX Stabilization)

**System:** 5 masses (1 kg each) connected in series with springs (1000 N/m), damping ζ = 1% per mode

**Analytical frequencies:** 3.18 Hz, 6.15 Hz, 8.76 Hz, 10.72 Hz, 11.83 Hz

**Stabilization diagram:** Run PolyMAX for orders 10-100, expect **5 stable vertical pole clusters** at analytical frequencies

**Expected:** Automated pole selection returns **exactly 5 modes**, frequency error < 1%, MAC vs. analytical > 0.95

**Tolerance:** All 5 modes correctly identified, no spurious modes, frequency < 1%, damping < 10%, MAC > 0.95

### Benchmark 5: FDD on Output-Only Data (3-DOF System)

**System:** 3-DOF mass-spring-damper, white noise base excitation (unmeasured), output accelerometers on 3 masses

**Analytical frequencies:** 7.5 Hz, 15.2 Hz, 22.1 Hz

**FDD algorithm:** Compute cross-spectral density matrix, SVD, peak detection on first singular value

**Expected:** **3 peaks detected** at analytical frequencies ±0.5 Hz

**EFDD damping estimation:** Damping ratios ≈ **1.5% ± 0.5%** (true damping = 1.5%)

**Tolerance:** Frequency error < 3% (OMA less accurate than EMA), damping error < 30%, 3 modes identified

---

## Verification

### Integration Testing Checklist

1. **Auth flow** — Register → login → JWT issued → API calls → token refresh → logout
2. **Campaign CRUD** — Create campaign → upload geometry → create setup → list campaigns
3. **Measurement upload** — Upload CSV → parse → S3 storage → metadata in DB → file size check
4. **WASM FRF computation** — Small file (10 MB) → WASM computes FRF → results displayed instantly
5. **Server FRF computation** — Large file (150 MB) → job queued → worker processes → WebSocket progress → results in S3
6. **Modal extraction** — Compute FRF → run PolyMAX → stabilization diagram → auto pole selection → mode table
7. **3D mode shape** — Upload STL → run modal analysis → load mode shape → animate in 3D viewer → 60 FPS
8. **MAC matrix** — Run two modal analyses → compute auto-MAC → heatmap display → near-identity off-diagonal
9. **Cross-MAC** — Upload FEA modes → compute cross-MAC vs. test → mode pairing suggestions → MAC > 0.9 for matches
10. **ODS** — Compute FRF → select frequency (e.g., 50 Hz) → compute ODS → animate deflection shape
11. **Report generation** — Select modal analysis → generate PDF report → S3 URL returned → download PDF with plots
12. **Plan limits** — Free user → upload 16-channel file → blocked with upgrade prompt
13. **Billing** — Subscribe to Pro → Stripe checkout → webhook → plan updated → feature unlocked
14. **Concurrent jobs** — 15 users submit modal extraction jobs simultaneously → all complete without errors
15. **WebSocket stability** — Long-running job (5 min) → client stays connected → receives all progress updates

### SQL Verification Queries

```sql
-- 1. Modal extraction success rate and performance
SELECT
    algorithm,
    status,
    COUNT(*) as count,
    AVG(wall_time_ms)::int as avg_time_ms,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY wall_time_ms)::int as p95_time_ms
FROM modal_analyses
WHERE created_at >= NOW() - INTERVAL '7 days'
GROUP BY algorithm, status;

-- 2. FRF computation throughput
SELECT
    execution_mode,
    COUNT(*) as frf_count,
    AVG(frequency_resolution)::numeric(10,2) as avg_freq_res_hz,
    AVG(averages) as avg_averages
FROM frfs
WHERE created_at >= NOW() - INTERVAL '7 days'
GROUP BY execution_mode;

-- 3. User plan distribution
SELECT plan, COUNT(*) as users,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 1) as pct
FROM users GROUP BY plan ORDER BY users DESC;

-- 4. Storage usage by user
SELECT u.email, u.plan,
    ROUND(u.storage_used_bytes / 1024.0 / 1024.0 / 1024.0, 2) as storage_gb,
    CASE u.plan
        WHEN 'free' THEN 0.05
        WHEN 'pro' THEN 10.0
        WHEN 'advanced' THEN 100.0
    END as limit_gb
FROM users u
WHERE u.storage_used_bytes > 0
ORDER BY u.storage_used_bytes DESC
LIMIT 20;

-- 5. Most common modal analysis frequency ranges
SELECT
    frequency_range,
    COUNT(*) as analysis_count,
    AVG(model_order) as avg_order
FROM modal_analyses
WHERE status = 'completed'
GROUP BY frequency_range
ORDER BY analysis_count DESC
LIMIT 10;

-- 6. Continuous monitoring modal tracking trends (TimescaleDB)
SELECT
    structure_id,
    mode_number,
    time_bucket('1 day', time) AS day,
    AVG(frequency_hz) as freq_avg,
    STDDEV(frequency_hz) as freq_stddev,
    MAX(anomaly_score) as anomaly_max
FROM modal_tracking
WHERE time >= NOW() - INTERVAL '30 days'
GROUP BY structure_id, mode_number, day
ORDER BY structure_id, mode_number, day;
```

### Performance Benchmarks

| Operation | Target | Method |
|-----------|--------|--------|
| WASM FFT: 16384 points | <50ms | Browser DevTools performance profiler |
| WASM FRF: 8 channels, 10 min @ 2048 Hz | <3s | Browser benchmark (total pipeline time) |
| Server FRF: 32 channels, 30 min @ 5120 Hz | <45s | Server timing, 4-core instance |
| PolyMAX: 16 channels, 200 freq points, order 100 | <60s | Server timing, 4-core instance |
| SSI-cov: 16 channels, 60 sec @ 2048 Hz, order 80 | <90s | Server timing, 4-core instance |
| 3D mode shape: 500 nodes animation | 60 FPS | Chrome FPS counter during rotation |
| 3D mode shape: 2000 nodes animation | 30 FPS | Chrome FPS counter |
| FRF plot: 16 channels × 2048 freq points | <500ms initial, 60 FPS zoom | D3.js render time + interaction FPS |
| Stabilization diagram: 10 orders × 50 poles | <200ms render | D3.js initial render |
| API: upload 100 MB file | <10s | Multipart upload with progress |
| API: create modal analysis | <300ms | p95 latency (Prometheus) |
| WebSocket: progress update latency | <150ms | Time from worker emit to client receive |
| S3: download 500 MB FRF matrix | <15s | AWS S3 transfer to client (10 Mbps network) |

---

## Deployment Architecture

### Infrastructure

```
┌──────────────────────────────────────────────────┐
│                CloudFront CDN                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────────┐   │
│  │ Frontend │  │   WASM   │  │  Geometry    │   │
│  │   SPA    │  │  Bundle  │  │  Thumbnails  │   │
│  └──────────┘  └──────────┘  └──────────────┘   │
└────────────────────┬─────────────────────────────┘
                     │
         ┌───────────┴───────────┐
         │   AWS ALB (HTTPS)     │
         │   TLS termination     │
         └───────────┬───────────┘
                     │
    ┌────────────────┼────────────────┐
    ▼                ▼                ▼
┌─────────┐  ┌─────────┐  ┌─────────┐
│ API     │  │ API     │  │ API     │
│ Server  │  │ Server  │  │ Server  │
│ (Axum)  │  │ (Axum)  │  │ (Axum)  │
│ Pod ×3  │  │         │  │         │
└────┬────┘  └────┬────┘  └────┬────┘
     │            │            │
     └────────────┼────────────┘
                  │
    ┌─────────────┼─────────────────┐
    ▼             ▼                 ▼
┌──────────┐ ┌──────────┐ ┌─────────────────┐
│PostgreSQL│ │  Redis   │ │  S3 Bucket      │
│+TimeScale│ │(Elastica-│ │  (Measurements, │
│  (RDS)   │ │ che)     │ │   FRFs, Modes,  │
│ Multi-AZ │ │ Cluster  │ │   Geometries)   │
└──────────┘ └────┬─────┘ └─────────────────┘
                  │
    ┌─────────────┼─────────────────┐
    ▼             ▼                 ▼
┌──────────┐ ┌──────────┐ ┌──────────┐
│  FRF     │ │  FRF     │ │  Modal   │
│ Worker   │ │ Worker   │ │ Worker   │
│(c7g.2xl) │ │(c7g.2xl) │ │(c7g.4xl) │
│ HPA: 1-10│ │          │ │ HPA: 1-8 │
└──────────┘ └──────────┘ └──────────┘
                                 │
                                 ▼
                         ┌──────────────┐
                         │  ML Service  │
                         │  (Python)    │
                         │  g5.xlarge   │
                         │  (GPU for    │
                         │   anomaly)   │
                         └──────────────┘
```

### Scaling Strategy

- **API servers**: HPA at 70% CPU, min 3, max 10 replicas
- **FRF workers**: HPA based on Redis `frf:jobs` queue depth, scale up when depth > 3, max 10
- **Modal workers**: HPA based on Redis `modal:jobs` queue depth, scale up when depth > 2, max 8 (more expensive compute)
- **Database**: RDS PostgreSQL r6g.2xlarge (8 vCPU, 64GB RAM) with 2 read replicas for campaign/setup queries
- **Redis**: ElastiCache r6g.large cluster (1 primary + 2 replicas) for job queue and WebSocket pub/sub
- **Worker instances**: c7g.2xlarge (8 vCPU, 16GB RAM) for FRF computation, c7g.4xlarge (16 vCPU, 32GB RAM) for modal extraction (larger linear algebra)

### S3 Lifecycle Policy

```json
{
  "Rules": [
    {
      "ID": "measurement-data-lifecycle",
      "Prefix": "measurements/",
      "Status": "Enabled",
      "Transitions": [
        { "Days": 30, "StorageClass": "STANDARD_IA" },
        { "Days": 90, "StorageClass": "GLACIER" }
      ],
      "Expiration": { "Days": 365 }
    },
    {
      "ID": "frf-results-lifecycle",
      "Prefix": "frfs/",
      "Status": "Enabled",
      "Transitions": [
        { "Days": 60, "StorageClass": "STANDARD_IA" }
      ],
      "Expiration": { "Days": 730 }
    },
    {
      "ID": "modal-results-keep",
      "Prefix": "modal/",
      "Status": "Enabled",
      "Transitions": [
        { "Days": 90, "StorageClass": "STANDARD_IA" }
      ]
    },
    {
      "ID": "geometry-keep",
      "Prefix": "geometries/",
      "Status": "Enabled"
    },
    {
      "ID": "wasm-bundles-versioned",
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
| Live DAQ Streaming Agent | Rust desktop agent connects to NI-DAQmx, Dewesoft, HBK hardware; streams real-time acceleration data to cloud via WebSocket; enables live ODS and continuous monitoring with <1s latency | High |
| Automated OMA for Continuous Monitoring | Run SSI-cov or FDD every 1-60 minutes on streaming data; track natural frequency and damping trends; alert on >5% frequency shift or >50% damping change indicating structural damage | High |
| AI Anomaly Detection | Train autoencoder on healthy-state vibration signatures; detect anomalies in modal parameters or raw acceleration; isolation forest for outlier detection; integrate with alert engine | High |
| Environmental/Operational Correction | ARX (AutoRegressive with eXogenous inputs) models to remove temperature, wind, traffic load effects from modal parameters; enables detection of true damage vs. environmental variation | High |
| FEA Model Updating | Sensitivity-based model updating: adjust FEA stiffness/mass parameters to minimize MAC error and frequency error between test and FEA modes; supports Nastran, Abaqus, ANSYS | Medium |
| Order Tracking for Rotating Machinery | Computed order tracking (COT) with tachometer signal or keyless tracking; order maps, Campbell diagrams; waterfall plots; enables vibration analysis on variable-speed machinery (turbines, engines) | Medium |
| Advanced Reporting with Templates | Customizable report templates (Word, LaTeX, HTML); drag-and-drop plot placement; automated narrative generation based on results; batch reporting for production QC | Medium |
| Multi-Setup OMA | Merge measurements from sequential test setups (roving sensors) to reconstruct full mode shapes on large structures; automated MAC-based stitching of partial mode shapes | Medium |
| Transmissibility-based OMA (TOMA) | Output-only modal analysis using transmissibility functions; works with limited sensor coverage and unknown excitation; complements FDD and SSI | Low |
| Bearing Fault Diagnostics | Envelope analysis with bearing defect frequency detection (BPFO, BPFI, BSF, FTF); automated fault classification; integrates with predictive maintenance alerts | Low |
| API for Programmatic Access | RESTful API for data upload, analysis triggering, result retrieval; enables integration with MATLAB, Python, CI/CD pipelines for automated test validation in product development | Low |
| Mobile App for Field Testing | iOS/Android app for impact hammer testing; smartphone accelerometer as sensor; cloud upload and modal analysis; enables low-cost field testing without dedicated DAQ hardware | Low
```

