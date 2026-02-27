# 31. SpectrumLab — Cloud Signal Processing & RF Analysis Platform

## Implementation Plan

**MVP Scope:** Browser-based real-time spectrum analyzer with GPU-accelerated FFT engine supporting 1K-4M point transforms for frequency resolution from kHz to mHz, WebGL-rendered spectrum plot and waterfall display with configurable color maps (viridis, plasma, magma) updating at 30+ FPS, lightweight local agent (Rust binary with SoapySDR) for SDR device connectivity (RTL-SDR and HackRF One initially) streaming compressed IQ samples to cloud infrastructure via WebSocket with <100ms latency, server-side GPU DSP pipeline (cuFFT on NVIDIA T4) for FFT computation and demodulation, FM and AM analog demodulation with browser audio playback via Web Audio API, configurable FFT parameters (window type: Hamming/Hanning/Blackman-Harris, averaging modes: linear/peak hold/min hold, RBW/VBW), frequency marker system with up to 8 markers supporting delta measurements and automatic peak search, IQ recording to cloud storage (Cloudflare R2) with SigMF metadata format and lossless compression achieving 2-3x size reduction, recording library with searchable metadata (center frequency, bandwidth, duration, device, tags) and playback with time scrubbing, user authentication via JWT with Google/GitHub OAuth, project-based organization for recordings and configurations, and Stripe billing with three tiers (Free / Pro $79/mo / Team $249/mo for 5 seats).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, low-latency IQ streaming, Tower middleware, JWT auth |
| DSP Engine | Rust + GPU (cuFFT) | GPU-accelerated FFT via NVIDIA cuFFT, custom demodulation kernels, <10ms processing latency |
| Local SDR Agent | Rust (SoapySDR) | Cross-platform SDR abstraction, IQ sample capture, compression (zstd), WebSocket streaming |
| ML Classification | Python 3.12 (FastAPI) | PyTorch models for automatic modulation classification, signal detection |
| Database | PostgreSQL 16 + PostGIS | Projects, users, recordings, annotations, geolocation queries, JSONB metadata |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, async PostgreSQL driver, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing (cost 12) |
| Object Storage | Cloudflare R2 | IQ recordings, processed results, ML models, compliance reports (S3-compatible) |
| Cache / Queue | Redis 7 | IQ stream buffering, FFT result cache (TTL 300s), session management, job queue |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management, TanStack Query for API |
| Spectrum/Waterfall | WebGL 2.0 (custom) | GPU-accelerated line/tile rendering, 60 FPS target, shader-based color mapping |
| Chain Builder | React Flow | Visual DSP block graph editor, custom node rendering |
| Audio Playback | Web Audio API | Demodulated signal playback, volume control, filtering |
| Real-time Streaming | WebSocket (Axum) | FFT data stream (binary ArrayBuffer), device control, progress updates |
| GPU Compute | Kubernetes + NVIDIA T4 | EKS cluster with GPU node pools, CUDA 12.x, cuFFT library |
| Billing | Stripe | Checkout Sessions, Customer Portal, usage-based metering, webhooks |
| CDN | CloudFront | Static assets, agent binary distribution, low-latency edge caching |
| Monitoring | Prometheus + Grafana, Sentry | DSP pipeline latency metrics, GPU utilization, error tracking |
| CI/CD | GitHub Actions | Rust agent cross-compilation (Linux/macOS/Windows), Docker image builds, E2E tests |

### Key Architecture Decisions

1. **GPU-accelerated FFT on server rather than client WebGPU**: While WebGPU could theoretically run FFTs in the browser, cross-browser compatibility is immature (Safari support limited) and performance varies wildly across GPUs. Server-side cuFFT on NVIDIA T4 instances provides consistent <5ms FFT computation for 4M-point transforms and allows batch processing across multiple users to amortize GPU costs. We stream only the resulting spectrum bins (~4KB per frame at 30 FPS = 120KB/s) rather than raw IQ samples (2.4 MSPS × 8 bytes = 19.2 MB/s), achieving 160x bandwidth reduction.

2. **Local agent architecture with cloud DSP rather than pure WebUSB**: WebUSB SDR access is browser-limited (Chrome/Edge only, no Firefox/Safari), requires complex driver handling in JavaScript, and provides inconsistent performance across OSes. A lightweight Rust agent (5-10 MB binary) with SoapySDR provides universal SDR support, handles device quirks, enables background operation when browser is closed, and allows remote SDR deployment (e.g., rooftop antenna → Raspberry Pi agent → cloud). The agent streams compressed IQ (zstd level 3, ~2.5x compression) over WebSocket for <100ms glass-to-glass latency.

3. **Hybrid IQ streaming vs. FFT-only streaming based on bandwidth**: For sample rates ≤2.4 MSPS (covers RTL-SDR, most HackRF use cases), we stream compressed IQ samples to the server for full DSP flexibility (demodulation, recording, classification). For higher sample rates (>2.4 MSPS), the agent performs local FFT and streams only spectral data to reduce bandwidth. This threshold is auto-negotiated based on client uplink speed (WebRTC bandwidth estimation).

4. **Cloudflare R2 for recordings rather than AWS S3**: R2 eliminates egress fees (AWS charges $0.09/GB for downloads), which is critical for a platform where users frequently download IQ recordings. R2 provides S3-compatible API, 10ms p50 latency from CloudFront edge, and costs $0.015/GB storage vs. S3's $0.023/GB. For a user downloading 100 GB/month of recordings, R2 saves $9/month in egress alone.

5. **SigMF metadata format for recordings**: SigMF (Signal Metadata Format) is an open standard (NTIA/ANS) that embeds JSON metadata alongside IQ samples, ensuring portability to other RF tools (GNU Radio, inspectrum, URH). We use SigMF `.sigmf-meta` + `.sigmf-data` file pairs stored in R2, with PostgreSQL indexing the metadata for fast search. This avoids vendor lock-in and enables dataset sharing with academic/research communities.

6. **PostgreSQL JSONB for signal metadata rather than rigid schema**: Signal characteristics vary wildly by type (WiFi has SSID/channel, ADS-B has ICAO address/altitude, LoRa has spreading factor). Using JSONB columns for annotation metadata allows flexible schema evolution without migrations while maintaining indexability via GIN indexes. We use typed schemas (via JSON Schema validation) for common signal types but allow freeform metadata for custom/unknown signals.

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";    -- Fuzzy text search
CREATE EXTENSION IF NOT EXISTS "postgis";    -- Geolocation queries

-- Users
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email TEXT UNIQUE NOT NULL,
    password_hash TEXT,  -- NULL for OAuth-only users
    name TEXT NOT NULL,
    avatar_url TEXT,
    auth_provider TEXT DEFAULT 'email',  -- email | google | github
    auth_provider_id TEXT,
    plan TEXT NOT NULL DEFAULT 'free',   -- free | pro | team | enterprise
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
    seat_count INTEGER DEFAULT 5,
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    settings JSONB DEFAULT '{}',  -- color_scheme, default_units, frequency_format
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
    role TEXT NOT NULL DEFAULT 'member',  -- owner | admin | member | viewer
    joined_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(org_id, user_id)
);
CREATE INDEX org_members_org_idx ON org_members(org_id);
CREATE INDEX org_members_user_idx ON org_members(user_id);

-- Projects (workspace for recordings and processing)
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    visibility TEXT NOT NULL DEFAULT 'private',  -- private | org | public
    forked_from UUID REFERENCES projects(id) ON DELETE SET NULL,
    thumbnail_url TEXT,
    settings JSONB DEFAULT '{}',  -- default FFT settings, color map, display preferences
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);
CREATE INDEX projects_org_idx ON projects(org_id);
CREATE INDEX projects_updated_idx ON projects(updated_at DESC);
CREATE INDEX projects_visibility_idx ON projects(visibility) WHERE visibility = 'public';

-- SDR Devices (registered agent installations)
CREATE TABLE sdr_devices (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    agent_id UUID NOT NULL UNIQUE,  -- Unique agent installation identifier
    device_type TEXT NOT NULL,      -- rtlsdr | hackrf | airspy | usrp | pluto | limesdr
    device_serial TEXT,
    label TEXT NOT NULL,
    capabilities JSONB NOT NULL,    -- {freq_range: [start, end], max_sample_rate, gain_range, antenna_ports}
    location GEOGRAPHY(POINT),      -- PostGIS point for geolocation (nullable)
    online BOOLEAN NOT NULL DEFAULT false,
    last_seen_at TIMESTAMPTZ,
    agent_version TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX sdr_devices_user_idx ON sdr_devices(user_id);
CREATE INDEX sdr_devices_agent_idx ON sdr_devices(agent_id);
CREATE INDEX sdr_devices_online_idx ON sdr_devices(online) WHERE online = true;
CREATE INDEX sdr_devices_location_idx ON sdr_devices USING GIST(location);

-- Recordings (IQ captures)
CREATE TABLE recordings (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    center_freq_hz BIGINT NOT NULL,
    sample_rate_hz INTEGER NOT NULL,
    bandwidth_hz INTEGER NOT NULL,
    format TEXT NOT NULL DEFAULT 'sigmf',  -- sigmf | raw_int16 | raw_float32 | wav
    iq_file_url TEXT NOT NULL,            -- R2 URL for IQ data (.sigmf-data)
    metadata_url TEXT,                     -- R2 URL for SigMF metadata (.sigmf-meta)
    duration_seconds REAL NOT NULL,
    file_size_bytes BIGINT NOT NULL,
    compressed_size_bytes BIGINT,          -- After zstd compression
    device_id UUID REFERENCES sdr_devices(id) ON DELETE SET NULL,
    location GEOGRAPHY(POINT),             -- Capture location (may differ from device location)
    tags TEXT[] DEFAULT '{}',
    metadata JSONB DEFAULT '{}',           -- User-defined metadata, SigMF core fields
    recorded_by UUID NOT NULL REFERENCES users(id) ON DELETE SET NULL,
    recorded_at TIMESTAMPTZ NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX recordings_project_idx ON recordings(project_id);
CREATE INDEX recordings_device_idx ON recordings(device_id);
CREATE INDEX recordings_freq_idx ON recordings(center_freq_hz);
CREATE INDEX recordings_recorded_at_idx ON recordings(recorded_at DESC);
CREATE INDEX recordings_tags_idx ON recordings USING gin(tags);
CREATE INDEX recordings_metadata_idx ON recordings USING gin(metadata);
CREATE INDEX recordings_location_idx ON recordings USING GIST(location);

-- Annotations (signal identification on recordings)
CREATE TABLE annotations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    recording_id UUID NOT NULL REFERENCES recordings(id) ON DELETE CASCADE,
    label TEXT NOT NULL,                   -- User-assigned label (e.g., "WiFi Ch 6", "LTE B13")
    signal_type TEXT,                      -- wifi | lte | 5g_nr | bluetooth | lora | ads_b | radar | unknown
    modulation TEXT,                       -- am | fm | bpsk | qpsk | 16qam | 64qam | ofdm | fsk | gfsk
    freq_start_hz BIGINT NOT NULL,
    freq_end_hz BIGINT NOT NULL,
    time_start_seconds REAL NOT NULL,
    time_end_seconds REAL NOT NULL,
    confidence REAL DEFAULT 1.0,           -- 0.0-1.0, lower for ML annotations
    source TEXT NOT NULL DEFAULT 'manual', -- manual | ml | api
    notes TEXT,
    metadata JSONB DEFAULT '{}',           -- Signal-specific metadata (SSID, MCC/MNC, etc.)
    created_by UUID NOT NULL REFERENCES users(id) ON DELETE SET NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX annotations_recording_idx ON annotations(recording_id);
CREATE INDEX annotations_signal_type_idx ON annotations(signal_type);
CREATE INDEX annotations_freq_range_idx ON annotations(freq_start_hz, freq_end_hz);
CREATE INDEX annotations_time_range_idx ON annotations(time_start_seconds, time_end_seconds);
CREATE INDEX annotations_metadata_idx ON annotations USING gin(metadata);

-- Processing Chains (DSP block graphs)
CREATE TABLE processing_chains (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    chain_data JSONB NOT NULL,             -- {blocks: [...], connections: [...], parameters: {...}}
    version INTEGER NOT NULL DEFAULT 1,
    parent_version_id UUID REFERENCES processing_chains(id) ON DELETE SET NULL,
    created_by UUID NOT NULL REFERENCES users(id) ON DELETE SET NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX chains_project_idx ON processing_chains(project_id);
CREATE INDEX chains_parent_idx ON processing_chains(parent_version_id);

-- Analysis Results (outputs from processing chains or measurements)
CREATE TABLE analysis_results (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    recording_id UUID REFERENCES recordings(id) ON DELETE SET NULL,
    chain_id UUID REFERENCES processing_chains(id) ON DELETE SET NULL,
    result_type TEXT NOT NULL,             -- spectrum | measurement | demodulation | classification
    result_data JSONB NOT NULL,            -- Measurements, metrics, classifications
    result_files_url TEXT,                 -- R2 URL for large result files (CSVs, images)
    created_by UUID NOT NULL REFERENCES users(id) ON DELETE SET NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX results_project_idx ON analysis_results(project_id);
CREATE INDEX results_recording_idx ON analysis_results(recording_id);
CREATE INDEX results_type_idx ON analysis_results(result_type);

-- Monitoring Schedules (automated scanning and alerts)
CREATE TABLE monitoring_schedules (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    device_id UUID NOT NULL REFERENCES sdr_devices(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    freq_ranges JSONB NOT NULL,            -- [{start_hz, end_hz, step_hz}]
    schedule JSONB NOT NULL,               -- Cron-like: {type: "interval", interval_seconds: 3600} or {type: "cron", cron: "0 * * * *"}
    dwell_time_ms INTEGER NOT NULL DEFAULT 1000,
    detection_config JSONB DEFAULT '{}',   -- {threshold_dbm, signal_types, min_duration_ms}
    alert_rules JSONB DEFAULT '[]',        -- [{condition, action: "email"|"webhook"|"slack", target}]
    enabled BOOLEAN NOT NULL DEFAULT true,
    last_run_at TIMESTAMPTZ,
    next_run_at TIMESTAMPTZ,
    created_by UUID NOT NULL REFERENCES users(id) ON DELETE SET NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX schedules_project_idx ON monitoring_schedules(project_id);
CREATE INDEX schedules_device_idx ON monitoring_schedules(device_id);
CREATE INDEX schedules_next_run_idx ON monitoring_schedules(next_run_at) WHERE enabled = true;

-- Alerts (triggered by monitoring schedules)
CREATE TABLE alerts (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    schedule_id UUID NOT NULL REFERENCES monitoring_schedules(id) ON DELETE CASCADE,
    recording_id UUID REFERENCES recordings(id) ON DELETE SET NULL,
    alert_type TEXT NOT NULL,              -- signal_detected | threshold_exceeded | signal_disappeared
    message TEXT NOT NULL,
    alert_data JSONB DEFAULT '{}',         -- Detected signal details
    resolved BOOLEAN NOT NULL DEFAULT false,
    resolved_by UUID REFERENCES users(id) ON DELETE SET NULL,
    resolved_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX alerts_schedule_idx ON alerts(schedule_id);
CREATE INDEX alerts_unresolved_idx ON alerts(resolved) WHERE NOT resolved;
CREATE INDEX alerts_created_idx ON alerts(created_at DESC);

-- Comments (collaboration)
CREATE TABLE comments (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    author_id UUID NOT NULL REFERENCES users(id) ON DELETE SET NULL,
    body TEXT NOT NULL,
    anchor_type TEXT NOT NULL,             -- recording | annotation | chain | result
    anchor_data JSONB NOT NULL,            -- {id, timestamp, freq_hz, ...}
    resolved BOOLEAN NOT NULL DEFAULT false,
    resolved_by UUID REFERENCES users(id) ON DELETE SET NULL,
    resolved_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX comments_project_idx ON comments(project_id);
CREATE INDEX comments_author_idx ON comments(author_id);
CREATE INDEX comments_unresolved_idx ON comments(resolved) WHERE NOT resolved;

-- Usage Records (for billing enforcement)
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,             -- recording_minutes | storage_gb | api_calls | gpu_minutes
    quantity REAL NOT NULL,
    period_start DATE NOT NULL,
    period_end DATE NOT NULL,
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
    pub settings: serde_json::Value,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct SdrDevice {
    pub id: Uuid,
    pub user_id: Uuid,
    pub agent_id: Uuid,
    pub device_type: String,
    pub device_serial: Option<String>,
    pub label: String,
    pub capabilities: serde_json::Value,
    #[sqlx(skip)]  // PostGIS geography requires custom decoding
    pub location: Option<String>,
    pub online: bool,
    pub last_seen_at: Option<DateTime<Utc>>,
    pub agent_version: Option<String>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Recording {
    pub id: Uuid,
    pub project_id: Uuid,
    pub name: String,
    pub description: String,
    pub center_freq_hz: i64,
    pub sample_rate_hz: i32,
    pub bandwidth_hz: i32,
    pub format: String,
    pub iq_file_url: String,
    pub metadata_url: Option<String>,
    pub duration_seconds: f32,
    pub file_size_bytes: i64,
    pub compressed_size_bytes: Option<i64>,
    pub device_id: Option<Uuid>,
    #[sqlx(skip)]
    pub location: Option<String>,
    pub tags: Vec<String>,
    pub metadata: serde_json::Value,
    pub recorded_by: Uuid,
    pub recorded_at: DateTime<Utc>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Annotation {
    pub id: Uuid,
    pub recording_id: Uuid,
    pub label: String,
    pub signal_type: Option<String>,
    pub modulation: Option<String>,
    pub freq_start_hz: i64,
    pub freq_end_hz: i64,
    pub time_start_seconds: f32,
    pub time_end_seconds: f32,
    pub confidence: f32,
    pub source: String,
    pub notes: Option<String>,
    pub metadata: serde_json::Value,
    pub created_by: Uuid,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct ProcessingChain {
    pub id: Uuid,
    pub project_id: Uuid,
    pub name: String,
    pub description: String,
    pub chain_data: serde_json::Value,
    pub version: i32,
    pub parent_version_id: Option<Uuid>,
    pub created_by: Uuid,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct MonitoringSchedule {
    pub id: Uuid,
    pub project_id: Uuid,
    pub device_id: Uuid,
    pub name: String,
    pub freq_ranges: serde_json::Value,
    pub schedule: serde_json::Value,
    pub dwell_time_ms: i32,
    pub detection_config: serde_json::Value,
    pub alert_rules: serde_json::Value,
    pub enabled: bool,
    pub last_run_at: Option<DateTime<Utc>>,
    pub next_run_at: Option<DateTime<Utc>>,
    pub created_by: Uuid,
    pub created_at: DateTime<Utc>,
}

// Request/Response DTOs

#[derive(Debug, Deserialize)]
pub struct DeviceTuneRequest {
    pub center_freq_hz: u64,
    pub sample_rate_hz: u32,
    pub gain_db: Option<f32>,
    pub bandwidth_hz: Option<u32>,
    pub antenna_port: Option<String>,
}

#[derive(Debug, Deserialize)]
pub struct FftConfig {
    pub size: u32,                // 1024, 2048, 4096, ..., 4194304 (4M)
    pub window: WindowType,
    pub averaging_mode: AveragingMode,
    pub averaging_count: u32,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum WindowType {
    Rectangular,
    Hamming,
    Hanning,
    BlackmanHarris,
    FlatTop,
    Kaiser,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum AveragingMode {
    Linear,
    PeakHold,
    MinHold,
    Exponential,
}

#[derive(Debug, Serialize)]
pub struct SpectrumData {
    pub frequencies_hz: Vec<f64>,  // Bin center frequencies
    pub power_dbm: Vec<f32>,       // Power per bin
    pub timestamp: DateTime<Utc>,
    pub center_freq_hz: u64,
    pub sample_rate_hz: u32,
    pub fft_size: u32,
}

#[derive(Debug, Deserialize)]
pub struct RecordingRequest {
    pub name: String,
    pub description: Option<String>,
    pub duration_seconds: Option<f32>,  // None = manual stop
    pub trigger_config: Option<TriggerConfig>,
    pub tags: Vec<String>,
}

#[derive(Debug, Deserialize)]
pub struct TriggerConfig {
    pub trigger_type: TriggerType,
    pub threshold_dbm: f32,
    pub pre_trigger_seconds: f32,   // Capture N seconds before trigger
    pub post_trigger_seconds: f32,  // Capture N seconds after trigger
}

#[derive(Debug, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum TriggerType {
    Manual,
    PowerThreshold,
    SignalDetection,
}
```

---

## DSP Architecture Deep-Dive

### FFT Processing Pipeline

SpectrumLab's core DSP pipeline processes IQ samples from the SDR agent through GPU-accelerated FFT to produce spectrum and waterfall displays. The pipeline is optimized for low latency (<100ms glass-to-glass) and high throughput (30+ FPS waterfall updates).

**Pipeline stages:**

```
SDR Hardware → Local Agent → WebSocket → Server → GPU FFT → WebSocket → Browser
  (IQ capture)   (compress)   (stream)    (buffer)  (compute)  (spectrum)  (render)
     <1ms          ~2ms        ~20ms       ~5ms      <5ms       ~20ms       16ms

Total latency: ~70ms typical, <100ms p99
```

**FFT computation (server-side CUDA kernel):**

The FFT engine runs on NVIDIA T4 GPUs using cuFFT. For a 4M-point FFT (provides 0.6 Hz resolution at 2.4 MSPS sample rate):

```
Input:  Complex<f32> IQ samples [4,194,304 elements] = 32 MB
Window: Apply Blackman-Harris window (GPU kernel, <1ms)
FFT:    cuFFT complex-to-complex transform (~3ms for 4M points)
Mag²:   Compute power spectral density (GPU kernel, <0.5ms)
Log:    Convert to dBm: 10*log10(PSD) + calibration (GPU kernel, <0.5ms)
Avg:    Apply averaging (linear/peak hold/exponential) (GPU kernel, <0.2ms)
Output: f32 power bins [2,097,152 bins] = 8 MB

Total: ~5ms per FFT frame
```

**Batch processing for cost efficiency:**

To amortize GPU overhead, we batch FFT computations from multiple users:

```rust
// Pseudo-code for FFT batching
struct FftBatch {
    requests: Vec<FftRequest>,  // Up to 16 requests per batch
    batch_size: usize,
}

async fn process_fft_batch(batch: FftBatch, gpu: &GpuContext) -> Vec<SpectrumData> {
    // Upload all IQ buffers to GPU in one transfer
    let device_buffers = gpu.upload_batch(&batch.requests);

    // Run cuFFT on each buffer (parallelized across GPU SMs)
    let fft_results = cufft_batch_exec(&device_buffers);

    // Download results
    let host_results = gpu.download_batch(&fft_results);

    host_results
}
```

With this approach, we achieve:
- Single-user latency: ~5ms
- 16-user batch latency: ~8ms (1.6x overhead for 16x throughput)
- Cost per FFT: $0.0001 on T4 GPU ($0.35/hr / 3500 FFTs/hour)

### Waterfall Rendering

The waterfall display accumulates spectrum frames over time and renders them as a scrolling 2D image. We use a GPU-optimized approach with WebGL shaders:

**Data structure:**
```typescript
interface WaterfallBuffer {
    data: Float32Array;      // [time_rows × freq_bins] flattened
    width: number;           // freq_bins (e.g., 2048)
    height: number;          // time_rows (e.g., 1024 for ~30s at 30 FPS)
    rowIndex: number;        // Current write position (circular buffer)
    texture: WebGLTexture;   // GPU texture (R32F format)
}
```

**Update process (30 FPS target):**
1. Receive spectrum data from WebSocket (Float32Array, ~8KB for 2048 bins)
2. Write to next row in waterfall buffer (circular)
3. Upload single row to GPU texture via `texSubImage2D` (<0.5ms)
4. Render via fragment shader with colormap lookup (<2ms at 1920×1080)

**WebGL fragment shader for colormap:**
```glsl
#version 300 es
precision mediump float;

in vec2 v_texCoord;
out vec4 fragColor;

uniform sampler2D u_waterfallData;  // Waterfall texture (power in dBm)
uniform sampler2D u_colormap;        // 1D colormap texture (256 colors)
uniform float u_minPower;            // dBm range min
uniform float u_maxPower;            // dBm range max
uniform int u_rowOffset;             // Circular buffer offset

void main() {
    // Adjust row index for circular buffer
    float adjustedY = mod(v_texCoord.y + float(u_rowOffset) / float(textureSize(u_waterfallData, 0).y), 1.0);

    // Sample power value
    float power = texture(u_waterfallData, vec2(v_texCoord.x, adjustedY)).r;

    // Normalize to [0, 1] for colormap lookup
    float normalized = (power - u_minPower) / (u_maxPower - u_minPower);
    normalized = clamp(normalized, 0.0, 1.0);

    // Sample colormap
    vec4 color = texture(u_colormap, vec2(normalized, 0.5));

    fragColor = color;
}
```

### IQ Streaming Protocol

The local agent streams IQ samples to the server via WebSocket using a custom binary protocol optimized for low overhead:

**Frame format:**
```
[Header: 24 bytes][Compressed IQ data: variable]

Header:
  magic: u32 (0x49515354 "IQST")
  version: u8 (1)
  flags: u8 (bit 0: compressed, bit 1: complex float vs int16)
  sequence: u16
  timestamp_ns: u64
  center_freq_hz: u64
  sample_rate_hz: u32
  sample_count: u32
  compressed_size: u32

IQ data (if compressed=true):
  zstd-compressed interleaved I/Q samples
  Format: [I0, Q0, I1, Q1, ...]
  Type: int16 (fixed-point Q15) or float32 based on flags
```

**Compression performance (zstd level 3):**
- Raw IQ (2.4 MSPS × 2 channels × 2 bytes/sample): 9.6 MB/s
- Compressed (typical RF signal with ~40% occupied spectrum): 3.8 MB/s
- Compression ratio: 2.5x
- CPU overhead (agent): ~15% on single core
- Latency penalty: ~2ms

**Adaptive rate control:**

The agent monitors WebSocket backpressure and adjusts behavior:
```rust
// Pseudo-code for agent streaming logic
if websocket.buffer_size() > MAX_BUFFER {
    // Slow consumer (network congestion or server overload)
    match config.overflow_strategy {
        OverflowStrategy::DropFrames => {
            // Drop oldest frames, keep real-time
            buffer.pop_front();
        }
        OverflowStrategy::ReduceRate => {
            // Decimate by 2x (skip every other frame)
            decimation_factor *= 2;
        }
    }
}
```

### Demodulation Architecture

For MVP, we support FM and AM demodulation. The demodulator runs on the server as a separate processing stage after FFT:

**FM demodulation (WBFM for broadcast, NBFM for narrowband):**

```rust
// src/dsp/demod/fm.rs

pub struct FmDemodulator {
    sample_rate: f32,
    deviation: f32,        // Max frequency deviation (75 kHz for WBFM, 5 kHz for NBFM)
    deemphasis: f32,       // Time constant (75 µs US, 50 µs EU)
    prev_phase: f32,
    deemph_state: f32,
}

impl FmDemodulator {
    pub fn demodulate(&mut self, iq_samples: &[Complex<f32>]) -> Vec<f32> {
        let mut audio = Vec::with_capacity(iq_samples.len());

        for sample in iq_samples {
            // Compute instantaneous phase
            let phase = sample.im.atan2(sample.re);

            // Phase difference (frequency)
            let mut delta_phase = phase - self.prev_phase;
            self.prev_phase = phase;

            // Unwrap phase (handle 2π discontinuities)
            if delta_phase > std::f32::consts::PI {
                delta_phase -= 2.0 * std::f32::consts::PI;
            } else if delta_phase < -std::f32::consts::PI {
                delta_phase += 2.0 * std::f32::consts::PI;
            }

            // Convert phase delta to frequency deviation
            let freq_deviation = delta_phase * self.sample_rate / (2.0 * std::f32::consts::PI);

            // Normalize by max deviation to get audio signal
            let mut audio_sample = freq_deviation / self.deviation;

            // Apply de-emphasis filter (1st-order IIR low-pass)
            let alpha = 1.0 / (1.0 + self.sample_rate * self.deemphasis);
            audio_sample = alpha * audio_sample + (1.0 - alpha) * self.deemph_state;
            self.deemph_state = audio_sample;

            audio.push(audio_sample);
        }

        audio
    }
}
```

**AM demodulation (envelope detection):**

```rust
// src/dsp/demod/am.rs

pub struct AmDemodulator {
    dc_blocker_state: f32,
}

impl AmDemodulator {
    pub fn demodulate(&mut self, iq_samples: &[Complex<f32>]) -> Vec<f32> {
        let mut audio = Vec::with_capacity(iq_samples.len());

        for sample in iq_samples {
            // Envelope detection: magnitude of complex sample
            let mut envelope = (sample.re * sample.re + sample.im * sample.im).sqrt();

            // DC blocking filter to remove carrier component
            // y[n] = x[n] - x[n-1] + 0.995*y[n-1]
            let dc_blocked = envelope - self.dc_blocker_state + 0.995 * audio.last().unwrap_or(&0.0);
            self.dc_blocker_state = envelope;

            audio.push(dc_blocked);
        }

        audio
    }
}
```

**Audio output pipeline:**
```
IQ samples (2.4 MSPS) → Demodulator → Audio (48 kHz) → Opus encoding → WebSocket → Browser Web Audio API
     Complex<f32>          Rust DSP      Resample        10 kbps         Binary      AudioWorklet
                                         (rational)       @32 kbps
```

For real-time audio, we target <50ms buffering to maintain conversational latency for voice modes.

---

## Architecture Deep-Dives

### 1. Local SDR Agent (Rust + SoapySDR)

The agent is a native binary that bridges SDR hardware to the cloud platform. It's designed to be lightweight (<10 MB), cross-platform (Linux/macOS/Windows), and secure (no elevation required).

```rust
// agent/src/main.rs

use anyhow::{Context, Result};
use clap::Parser;
use soapysdr::{Args, Device, Direction, RxStream};
use tokio::sync::mpsc;
use tokio_tungstenite::{connect_async, tungstenite::Message};
use zstd::stream::Encoder;

#[derive(Parser)]
#[clap(name = "spectrumlab-agent", version)]
struct Cli {
    /// WebSocket server URL
    #[clap(long, env = "SL_SERVER_URL", default_value = "wss://api.spectrumlab.io/ws/agent")]
    server_url: String,

    /// Agent authentication token
    #[clap(long, env = "SL_AGENT_TOKEN")]
    token: String,

    /// SDR device identifier (auto-detect if not specified)
    #[clap(long)]
    device: Option<String>,
}

#[tokio::main]
async fn main() -> Result<()> {
    let cli = Cli::parse();
    tracing_subscriber::fmt::init();

    // Enumerate SDR devices
    let devices = soapysdr::enumerate("")?;
    tracing::info!("Found {} SDR device(s)", devices.len());
    for (i, dev) in devices.iter().enumerate() {
        tracing::info!("  [{}] {}", i, dev);
    }

    // Select device
    let device_args = match cli.device {
        Some(d) => Args::from_string(&d)?,
        None => devices.first().context("No SDR device found")?.clone(),
    };

    let device = Device::new(device_args)?;
    tracing::info!("Opened device: {}", device.hardware_info()?);

    // Connect to WebSocket server
    let url = format!("{}?token={}", cli.server_url, cli.token);
    let (ws_stream, _) = connect_async(&url).await?;
    tracing::info!("Connected to server");

    let (mut ws_tx, mut ws_rx) = ws_stream.split();

    // Send device registration
    let capabilities = serde_json::json!({
        "device_type": detect_device_type(&device),
        "device_serial": device.hardware_info().ok(),
        "freq_range": device.frequency_range(Direction::Rx, 0).ok(),
        "max_sample_rate": device.get_sample_rate_range(Direction::Rx, 0).ok()
            .and_then(|r| r.last().map(|range| range.maximum())),
        "gain_range": device.gain_range(Direction::Rx, 0).ok(),
        "antenna_ports": device.antennas(Direction::Rx, 0).ok(),
    });

    ws_tx.send(Message::Text(serde_json::json!({
        "type": "register",
        "capabilities": capabilities
    }).to_string())).await?;

    // Handle commands from server
    let (iq_tx, mut iq_rx) = mpsc::channel::<Vec<Complex<f32>>>(64);

    // Spawn IQ capture task
    let capture_handle = tokio::task::spawn_blocking(move || {
        capture_iq(device, iq_tx)
    });

    // Spawn IQ streaming task
    let stream_handle = tokio::spawn(async move {
        stream_iq(iq_rx, ws_tx).await
    });

    // Handle control messages
    while let Some(msg) = ws_rx.next().await {
        let msg = msg?;
        if let Message::Text(text) = msg {
            let cmd: serde_json::Value = serde_json::from_str(&text)?;
            match cmd["type"].as_str() {
                Some("tune") => {
                    // Handle tuning command
                    // (send to capture task via channel)
                }
                Some("start") => {
                    // Start IQ streaming
                }
                Some("stop") => {
                    // Stop IQ streaming
                }
                _ => {}
            }
        }
    }

    capture_handle.await??;
    stream_handle.await?;

    Ok(())
}

fn capture_iq(mut device: Device, tx: mpsc::Sender<Vec<Complex<f32>>>) -> Result<()> {
    const BUFFER_SIZE: usize = 16384;  // 16K samples per read

    // Configure device
    device.set_sample_rate(Direction::Rx, 0, 2.4e6)?;
    device.set_frequency(Direction::Rx, 0, 100.0e6, Args::new())?;
    device.set_gain(Direction::Rx, 0, 30.0)?;

    // Create RX stream
    let mut stream = device.rx_stream::<Complex<f32>>(&[0])?;
    stream.activate(None)?;

    let mut buffer = vec![Complex::new(0.0, 0.0); BUFFER_SIZE];

    loop {
        // Read IQ samples
        let read_result = stream.read(&[&mut buffer], 1_000_000)?;

        if read_result.len() > 0 {
            // Send to streaming task
            if tx.blocking_send(buffer[..read_result.len()].to_vec()).is_err() {
                // Channel closed, exit
                break;
            }
        }
    }

    stream.deactivate(None)?;
    Ok(())
}

async fn stream_iq(
    mut rx: mpsc::Receiver<Vec<Complex<f32>>>,
    mut ws_tx: impl Sink<Message, Error = impl std::error::Error>
) -> Result<()> {
    let mut sequence = 0u16;
    let mut encoder = Encoder::new(Vec::new(), 3)?;  // zstd level 3

    while let Some(iq_samples) = rx.recv().await {
        // Convert to int16 (Q15 fixed-point)
        let int16_samples: Vec<i16> = iq_samples.iter()
            .flat_map(|c| vec![
                (c.re * 32767.0) as i16,
                (c.im * 32767.0) as i16,
            ])
            .collect();

        // Compress
        encoder.write_all(bytemuck::cast_slice(&int16_samples))?;
        let compressed = encoder.finish()?;

        // Build frame
        let mut frame = Vec::with_capacity(24 + compressed.len());
        frame.extend_from_slice(&0x49515354u32.to_le_bytes());  // Magic "IQST"
        frame.push(1);  // Version
        frame.push(0b00000001);  // Flags: compressed=1, int16=0
        frame.extend_from_slice(&sequence.to_le_bytes());
        frame.extend_from_slice(&std::time::SystemTime::now()
            .duration_since(std::time::UNIX_EPOCH)?.as_nanos().to_le_bytes());
        frame.extend_from_slice(&100_000_000u64.to_le_bytes());  // center_freq_hz
        frame.extend_from_slice(&2_400_000u32.to_le_bytes());    // sample_rate_hz
        frame.extend_from_slice(&(iq_samples.len() as u32).to_le_bytes());
        frame.extend_from_slice(&(compressed.len() as u32).to_le_bytes());
        frame.extend_from_slice(&compressed);

        // Send via WebSocket
        ws_tx.send(Message::Binary(frame)).await?;

        sequence = sequence.wrapping_add(1);
        encoder = Encoder::new(Vec::new(), 3)?;
    }

    Ok(())
}

fn detect_device_type(device: &Device) -> String {
    let hw_key = device.hardware_key().unwrap_or_default();

    if hw_key.contains("rtlsdr") {
        "rtlsdr"
    } else if hw_key.contains("hackrf") {
        "hackrf"
    } else if hw_key.contains("airspy") {
        "airspy"
    } else if hw_key.contains("uhd") || hw_key.contains("usrp") {
        "usrp"
    } else if hw_key.contains("plutosdr") {
        "pluto"
    } else if hw_key.contains("lime") {
        "limesdr"
    } else {
        "unknown"
    }
}

use num_complex::Complex;
use futures::{Sink, SinkExt, StreamExt};
use std::io::Write;
```

### 2. GPU FFT Service (Rust + cuFFT)

The FFT service is a stateful worker that maintains GPU context and processes batched FFT requests from multiple WebSocket connections.

```rust
// src/dsp/fft_service.rs

use cudarc::driver::{CudaDevice, CudaStream};
use cudarc::cufft::{CuFFT, CuFFTPlan};
use std::sync::Arc;
use tokio::sync::{mpsc, oneshot};
use num_complex::Complex;

pub struct FftService {
    device: Arc<CudaDevice>,
    plans: Vec<CuFFTPlan>,  // Pre-allocated plans for common FFT sizes
    request_rx: mpsc::Receiver<FftRequest>,
}

pub struct FftRequest {
    pub iq_samples: Vec<Complex<f32>>,
    pub config: FftConfig,
    pub response_tx: oneshot::Sender<FftResult>,
}

pub struct FftResult {
    pub frequencies_hz: Vec<f64>,
    pub power_dbm: Vec<f32>,
    pub timestamp: chrono::DateTime<chrono::Utc>,
}

impl FftService {
    pub fn new(device_id: usize) -> anyhow::Result<(Self, FftServiceHandle)> {
        let device = CudaDevice::new(device_id)?;

        // Pre-create FFT plans for common sizes (1K, 2K, 4K, ..., 4M)
        let mut plans = Vec::new();
        for i in 10..=22 {  // 2^10 = 1024 to 2^22 = 4M
            let size = 1 << i;
            let plan = CuFFTPlan::new_c2c(&device, size, 1)?;
            plans.push(plan);
        }

        let (request_tx, request_rx) = mpsc::channel(256);

        let service = Self {
            device,
            plans,
            request_rx,
        };

        let handle = FftServiceHandle { request_tx };

        Ok((service, handle))
    }

    pub async fn run(mut self) -> anyhow::Result<()> {
        tracing::info!("FFT service started on GPU {}", self.device.ordinal());

        while let Some(req) = self.request_rx.recv().await {
            let result = self.process_fft(req.iq_samples, req.config).await;
            let _ = req.response_tx.send(result);
        }

        Ok(())
    }

    async fn process_fft(&self, iq_samples: Vec<Complex<f32>>, config: FftConfig) -> FftResult {
        let fft_size = config.size as usize;

        // Find pre-allocated plan
        let plan_idx = (config.size as f32).log2() as usize - 10;
        let plan = &self.plans[plan_idx];

        // Apply window function (on CPU for now; could move to GPU)
        let windowed = self.apply_window(&iq_samples, config.window);

        // Upload to GPU
        let device_input = self.device.htod_copy(windowed)?;
        let mut device_output = self.device.alloc_zeros::<Complex<f32>>(fft_size)?;

        // Execute FFT
        plan.exec_c2c(&device_input, &mut device_output)?;

        // Download result
        let fft_output = self.device.dtoh_sync_copy(&device_output)?;

        // Compute power spectral density
        let psd = self.compute_psd(&fft_output, iq_samples.len() as f32);

        // Convert to dBm
        let power_dbm = psd.iter()
            .map(|&p| 10.0 * p.log10() + self.calibration_offset_dbm())
            .collect();

        // Generate frequency bins
        let sample_rate = 2.4e6;  // TODO: get from config
        let frequencies_hz: Vec<f64> = (0..fft_size)
            .map(|i| {
                let bin_freq = (i as f64 - fft_size as f64 / 2.0) * sample_rate / fft_size as f64;
                100.0e6 + bin_freq  // center_freq + bin offset
            })
            .collect();

        FftResult {
            frequencies_hz,
            power_dbm,
            timestamp: chrono::Utc::now(),
        }
    }

    fn apply_window(&self, samples: &[Complex<f32>], window_type: WindowType) -> Vec<Complex<f32>> {
        let n = samples.len();
        let window: Vec<f32> = match window_type {
            WindowType::Hamming => (0..n).map(|i| {
                0.54 - 0.46 * (2.0 * std::f32::consts::PI * i as f32 / (n - 1) as f32).cos()
            }).collect(),
            WindowType::Hanning => (0..n).map(|i| {
                0.5 * (1.0 - (2.0 * std::f32::consts::PI * i as f32 / (n - 1) as f32).cos())
            }).collect(),
            WindowType::BlackmanHarris => (0..n).map(|i| {
                let x = 2.0 * std::f32::consts::PI * i as f32 / (n - 1) as f32;
                0.35875 - 0.48829 * x.cos() + 0.14128 * (2.0 * x).cos() - 0.01168 * (3.0 * x).cos()
            }).collect(),
            WindowType::Rectangular => vec![1.0; n],
            _ => vec![1.0; n],
        };

        samples.iter().zip(window.iter())
            .map(|(s, w)| Complex::new(s.re * w, s.im * w))
            .collect()
    }

    fn compute_psd(&self, fft_output: &[Complex<f32>], sample_count: f32) -> Vec<f32> {
        // FFTshift (move DC bin to center)
        let n = fft_output.len();
        let mut shifted = vec![Complex::new(0.0, 0.0); n];
        for i in 0..n {
            shifted[i] = fft_output[(i + n / 2) % n];
        }

        // Compute magnitude squared and normalize
        shifted.iter()
            .map(|c| (c.re * c.re + c.im * c.im) / sample_count)
            .collect()
    }

    fn calibration_offset_dbm(&self) -> f32 {
        // TODO: Per-device calibration curve
        -10.0  // Typical RTL-SDR offset
    }
}

#[derive(Clone)]
pub struct FftServiceHandle {
    request_tx: mpsc::Sender<FftRequest>,
}

impl FftServiceHandle {
    pub async fn compute_fft(
        &self,
        iq_samples: Vec<Complex<f32>>,
        config: FftConfig,
    ) -> anyhow::Result<FftResult> {
        let (tx, rx) = oneshot::channel();

        self.request_tx.send(FftRequest {
            iq_samples,
            config,
            response_tx: tx,
        }).await?;

        Ok(rx.await?)
    }
}
```

### 3. WebSocket Spectrum Streaming Handler

The WebSocket handler connects browser clients to the FFT service and streams real-time spectrum data.

```rust
// src/api/ws/spectrum.rs

use axum::{
    extract::{
        ws::{WebSocket, WebSocketUpgrade, Message},
        State, Path,
    },
    response::IntoResponse,
};
use futures::{sink::SinkExt, stream::StreamExt};
use uuid::Uuid;
use crate::{
    state::AppState,
    auth::Claims,
    dsp::fft_service::{FftServiceHandle, FftConfig},
};

pub async fn spectrum_ws_handler(
    ws: WebSocketUpgrade,
    State(state): State<AppState>,
    claims: Claims,
    Path(device_id): Path<Uuid>,
) -> impl IntoResponse {
    ws.on_upgrade(move |socket| handle_spectrum_ws(socket, state, claims, device_id))
}

async fn handle_spectrum_ws(
    socket: WebSocket,
    state: AppState,
    claims: Claims,
    device_id: Uuid,
) {
    let (mut ws_tx, mut ws_rx) = socket.split();

    // Verify device ownership
    let device = match sqlx::query_as!(
        crate::db::models::SdrDevice,
        "SELECT * FROM sdr_devices WHERE id = $1 AND user_id = $2",
        device_id, claims.user_id
    )
    .fetch_optional(&state.db)
    .await
    {
        Ok(Some(d)) => d,
        _ => {
            let _ = ws_tx.send(Message::Text("Unauthorized".to_string())).await;
            return;
        }
    };

    // Subscribe to IQ stream from agent for this device
    let mut iq_rx = state.iq_stream_manager.subscribe(device_id).await;

    // Default FFT config
    let mut fft_config = FftConfig {
        size: 2048,
        window: crate::db::models::WindowType::BlackmanHarris,
        averaging_mode: crate::db::models::AveragingMode::Linear,
        averaging_count: 4,
    };

    // Spawn task to handle client control messages
    let fft_config_handle = Arc::new(tokio::sync::RwLock::new(fft_config.clone()));
    let config_clone = fft_config_handle.clone();
    tokio::spawn(async move {
        while let Some(Ok(msg)) = ws_rx.next().await {
            if let Message::Text(text) = msg {
                if let Ok(cmd) = serde_json::from_str::<serde_json::Value>(&text) {
                    match cmd["type"].as_str() {
                        Some("configure_fft") => {
                            if let Ok(new_config) = serde_json::from_value(cmd["config"].clone()) {
                                *config_clone.write().await = new_config;
                            }
                        }
                        Some("tune") => {
                            // Forward tuning command to agent
                            // (via Redis pub/sub to agent connection)
                        }
                        _ => {}
                    }
                }
            }
        }
    });

    // Stream FFT results to client
    let mut frame_count = 0u64;
    let mut averager = SpectrumAverager::new(2048);

    while let Some(iq_frame) = iq_rx.recv().await {
        // Compute FFT
        let config = fft_config_handle.read().await.clone();
        let fft_result = match state.fft_service.compute_fft(iq_frame.samples, config.clone()).await {
            Ok(r) => r,
            Err(e) => {
                tracing::error!("FFT error: {}", e);
                continue;
            }
        };

        // Apply averaging
        let averaged_power = averager.update(&fft_result.power_dbm, config.averaging_mode);

        // Send to client (binary format for efficiency)
        let binary_data = encode_spectrum_frame(&fft_result.frequencies_hz, &averaged_power);
        if ws_tx.send(Message::Binary(binary_data)).await.is_err() {
            break;  // Client disconnected
        }

        frame_count += 1;

        // Rate limit to 30 FPS
        if frame_count % 80 == 0 {  // 2.4 MSPS / 2048 FFT = 1172 frames/sec, send every 80th ≈ 15 FPS
            tokio::time::sleep(tokio::time::Duration::from_millis(33)).await;
        }
    }

    tracing::info!("Spectrum WebSocket closed for device {}", device_id);
}

fn encode_spectrum_frame(frequencies: &[f64], power: &[f32]) -> Vec<u8> {
    // Binary format: [u32 count][f64 freq_start][f64 freq_step][f32 power_0][f32 power_1]...
    let count = power.len() as u32;
    let freq_start = frequencies[0];
    let freq_step = if frequencies.len() > 1 {
        frequencies[1] - frequencies[0]
    } else {
        0.0
    };

    let mut buf = Vec::with_capacity(4 + 8 + 8 + power.len() * 4);
    buf.extend_from_slice(&count.to_le_bytes());
    buf.extend_from_slice(&freq_start.to_le_bytes());
    buf.extend_from_slice(&freq_step.to_le_bytes());
    for &p in power {
        buf.extend_from_slice(&p.to_le_bytes());
    }

    buf
}

struct SpectrumAverager {
    buffer: Vec<Vec<f32>>,
    index: usize,
    count: usize,
}

impl SpectrumAverager {
    fn new(size: usize) -> Self {
        Self {
            buffer: vec![vec![0.0; size]; 16],  // Circular buffer of 16 frames
            index: 0,
            count: 0,
        }
    }

    fn update(&mut self, new_data: &[f32], mode: crate::db::models::AveragingMode) -> Vec<f32> {
        self.buffer[self.index] = new_data.to_vec();
        self.index = (self.index + 1) % self.buffer.len();
        self.count = (self.count + 1).min(self.buffer.len());

        let active_buffer = &self.buffer[..self.count];

        match mode {
            crate::db::models::AveragingMode::Linear => {
                // Linear average across N frames
                let mut result = vec![0.0; new_data.len()];
                for frame in active_buffer {
                    for (i, &val) in frame.iter().enumerate() {
                        result[i] += val;
                    }
                }
                result.iter().map(|&v| v / self.count as f32).collect()
            }
            crate::db::models::AveragingMode::PeakHold => {
                // Keep maximum value per bin
                let mut result = vec![f32::NEG_INFINITY; new_data.len()];
                for frame in active_buffer {
                    for (i, &val) in frame.iter().enumerate() {
                        result[i] = result[i].max(val);
                    }
                }
                result
            }
            crate::db::models::AveragingMode::MinHold => {
                // Keep minimum value per bin
                let mut result = vec![f32::INFINITY; new_data.len()];
                for frame in active_buffer {
                    for (i, &val) in frame.iter().enumerate() {
                        result[i] = result[i].min(val);
                    }
                }
                result
            }
            crate::db::models::AveragingMode::Exponential => {
                // Exponential moving average: y[n] = α·x[n] + (1-α)·y[n-1]
                let alpha = 0.2;  // Smoothing factor
                let prev = if self.count > 1 {
                    &self.buffer[(self.index + self.buffer.len() - 2) % self.buffer.len()]
                } else {
                    new_data
                };

                new_data.iter().zip(prev.iter())
                    .map(|(&new, &old)| alpha * new + (1.0 - alpha) * old)
                    .collect()
            }
        }
    }
}

use std::sync::Arc;
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–5)

**Day 1: Project initialization and Rust backend scaffold**
```bash
cargo init spectrumlab-api
cd spectrumlab-api
cargo add axum tokio serde serde_json sqlx uuid chrono tower-http jsonwebtoken bcrypt anyhow tracing tracing-subscriber
cargo add --features postgres,runtime-tokio-rustls,uuid,chrono,json sqlx
cargo add redis tokio-tungstenite futures num-complex
```
- `src/main.rs` — Axum app with CORS, tracing (JSON format), graceful shutdown, health endpoint
- `src/config.rs` — Environment-based configuration (DATABASE_URL, JWT_SECRET, REDIS_URL, R2_*, STRIPE_*)
- `src/state.rs` — AppState struct (PgPool, Redis, R2Client, FftServiceHandle)
- `src/error.rs` — ApiError enum with IntoResponse implementation for HTTP error mapping
- `Dockerfile` — Multi-stage build (Rust builder + runtime with minimal dependencies)
- `docker-compose.yml` — PostgreSQL 16, Redis 7, MinIO (S3-compatible for local dev)

**Day 2: Database schema and migrations**
- `migrations/001_initial.sql` — All 13 tables: users, organizations, org_members, projects, sdr_devices, recordings, annotations, processing_chains, analysis_results, monitoring_schedules, alerts, comments, usage_records
- `migrations/002_postgis.sql` — PostGIS extension setup for geolocation queries
- `src/db/mod.rs` — Database pool initialization with connection pooling (max 20 connections)
- `src/db/models.rs` — All SQLx structs with FromRow derives, request/response DTOs
- Run `sqlx migrate run` to apply schema
- Seed script for test data (sample user, project, device)

**Day 3: Authentication system (JWT + OAuth)**
- `src/auth/mod.rs` — JWT token generation (HS256, 24h access + 30d refresh), validation middleware extracting Claims from Authorization header
- `src/auth/oauth.rs` — Google and GitHub OAuth 2.0 flow handlers (redirect, callback, token exchange)
- `src/api/handlers/auth.rs` — POST /register (email/password), POST /login, GET /oauth/:provider, GET /oauth/:provider/callback, POST /refresh, GET /me
- Password hashing with bcrypt (cost factor 12, ~250ms per hash for rate limiting)
- JWT Claims struct with user_id, email, plan, exp, iat
- Auth middleware: extract JWT from header, validate signature, check expiration, inject Claims into request extensions

**Day 4: User and project CRUD APIs**
- `src/api/handlers/users.rs` — GET /users/me, PATCH /users/me, DELETE /users/me, GET /users/:id (public profile)
- `src/api/handlers/projects.rs` — POST /projects, GET /projects (list with pagination, filter by org/visibility), GET /projects/:id, PATCH /projects/:id, DELETE /projects/:id, POST /projects/:id/fork
- `src/api/handlers/orgs.rs` — POST /orgs, GET /orgs/:slug, POST /orgs/:id/members (invite), GET /orgs/:id/members, DELETE /orgs/:id/members/:user_id
- `src/api/router.rs` — Axum router with all route definitions, auth middleware on protected routes
- Authorization helpers: `check_project_access(user_id, project_id, required_role)`, `check_org_access(user_id, org_id, required_role)`

**Day 5: SDR device registration and management**
- `src/api/handlers/devices.rs` — POST /devices/register (agent registration with capabilities), GET /devices (list user's devices), GET /devices/:id, PATCH /devices/:id/config, DELETE /devices/:id
- `src/api/ws/agent.rs` — WebSocket endpoint for agent connection: /ws/agent?token=xxx
- Agent heartbeat protocol: agent sends ping every 30s, server updates `last_seen_at` and sets `online=true`, marks offline after 60s timeout
- Device capabilities validation: freq_range, sample_rate, gain_range stored as JSONB
- Test agent connection locally with websocat: `websocat ws://localhost:3000/ws/agent?token=test`

### Phase 2 — Local SDR Agent (Days 6–10)

**Day 6: Agent scaffold and SoapySDR integration**
- `agent/` — New Rust workspace member for cross-platform agent
- `agent/Cargo.toml` — clap (CLI parsing), soapysdr (SDR abstraction), tokio, tokio-tungstenite, serde, anyhow, tracing
- `agent/src/main.rs` — CLI with arguments: --server-url, --token, --device
- `agent/src/soapy.rs` — Wrapper around SoapySDR: enumerate devices, open device, configure (freq, rate, gain), create RX stream
- Test enumeration: `cargo run -- --device` lists all connected SDR devices
- RTL-SDR and HackRF detection via hardware key matching

**Day 7: IQ capture and compression**
- `agent/src/capture.rs` — IQ capture loop: read from SoapySDR RX stream (16K samples/buffer), convert to int16 Q15 fixed-point, accumulate into larger frames (256K samples for ~100ms at 2.4 MSPS)
- `agent/src/compression.rs` — zstd compression (level 3) wrapper, measure compression ratio
- Benchmark compression: measure CPU overhead (<15% target) and compression ratio (2.5x typical for real RF signals)
- Handle sample drops: if SoapySDR returns fewer samples than requested, log warning and insert zeros (prevents audio glitches in demodulation)

**Day 8: WebSocket client and streaming protocol**
- `agent/src/ws_client.rs` — WebSocket client with automatic reconnection (exponential backoff: 1s, 2s, 4s, max 30s)
- `agent/src/protocol.rs` — Binary frame encoding: 24-byte header + compressed IQ payload (see protocol spec in architecture section)
- Registration message: send device capabilities on connect
- Heartbeat: send ping every 30s, expect pong within 10s, reconnect if timeout
- Backpressure handling: monitor WebSocket send buffer size, drop frames if >10MB queued

**Day 9: Agent control commands**
- `agent/src/commands.rs` — Handle control messages from server: `tune` (set freq/rate/gain), `start_stream`, `stop_stream`, `configure` (update compression level, frame size)
- Tuning validation: check freq/rate/gain against device capabilities before applying
- State machine: Idle → Streaming → Paused, with atomic transitions
- Test tuning via websocat: send JSON `{"type": "tune", "center_freq_hz": 100000000, "sample_rate_hz": 2400000, "gain_db": 30}`

**Day 10: Agent cross-compilation and distribution**
- `.github/workflows/agent-build.yml` — Cross-compile for Linux x64, macOS (Intel + ARM), Windows x64
- Static linking on Linux (musl target) to avoid glibc dependency issues
- macOS universal binary (x86_64 + aarch64 fat binary)
- Windows: bundle SoapySDR DLLs and drivers in installer (NSIS)
- Upload binaries to R2 with versioned URLs: `https://downloads.spectrumlab.io/agent/v0.1.0/spectrumlab-agent-{linux|macos|windows}.{tar.gz|zip}`
- One-liner install script: `curl -sSL https://install.spectrumlab.io | sh` (detects OS, downloads, extracts, prompts for token)

### Phase 3 — GPU FFT Pipeline (Days 11–16)

**Day 11: GPU environment setup and cuFFT integration**
- EKS cluster with GPU node pool: NVIDIA T4 instances (AWS g4dn.xlarge: 1 T4 GPU, 4 vCPU, 16 GB RAM, $0.526/hr on-demand)
- CUDA 12.x runtime Docker image (nvidia/cuda:12.3.0-runtime-ubuntu22.04)
- `Cargo.toml` — Add cudarc (Rust CUDA bindings), cudarc-cufft
- `src/dsp/mod.rs` — FFT service module structure
- `src/dsp/fft_service.rs` — Basic FftService struct with device initialization, plan creation for sizes 1K-4M

**Day 12: FFT computation pipeline**
- `src/dsp/fft_service.rs` — Implement `process_fft`: upload IQ to GPU, execute cuFFT, download result, compute PSD, convert to dBm
- Window functions: Rectangular, Hamming, Hanning, Blackman-Harris (CPU-side for now, GPU optimization later)
- FFT normalization: divide by sample count for correct power calibration
- FFTshift: reorder bins to put DC in center
- Benchmark FFT performance: 1K (0.2ms), 4K (0.5ms), 16K (1ms), 64K (2ms), 256K (3ms), 1M (4ms), 4M (5ms) on T4

**Day 13: FFT service batching and request queue**
- Multi-threaded request queue with batching: accumulate up to 16 requests, process in batch to amortize GPU transfer overhead
- Tokio task for FFT worker: consumes from mpsc channel, processes batches, sends results via oneshot channels
- Dynamic batching: if queue depth <4, process immediately for low latency; if depth >4, wait up to 5ms to accumulate batch
- Benchmark: single-user latency 5ms, 16-user batch latency 8ms (throughput 2000 FFTs/sec)

**Day 14: Spectrum averaging and post-processing**
- `src/dsp/averaging.rs` — SpectrumAverager struct implementing linear, peak hold, min hold, exponential averaging
- RBW (Resolution Bandwidth) calculation: RBW = sample_rate / fft_size, display in UI
- VBW (Video Bandwidth) via exponential averaging: VBW ≈ frame_rate / averaging_count
- Noise floor estimation: compute median across all bins, subtract from each bin for noise-normalized display

**Day 15: Integration with WebSocket streaming**
- `src/api/ws/spectrum.rs` — WebSocket handler for spectrum streaming (see architecture section for full code)
- Subscribe to IQ stream from agent (via Redis pub/sub), forward to FFT service, stream results to client
- Binary WebSocket protocol for spectrum data: count + freq_start + freq_step + power array (4KB for 2048 bins)
- Rate limiting: 30 FPS client-side, drop frames if client can't keep up
- Test with websocat and binary decoder

**Day 16: FFT configuration and dynamic adjustment**
- Client-side FFT configuration UI: FFT size slider (1K-4M), window type dropdown, averaging mode/count
- Send configuration via WebSocket: `{"type": "configure_fft", "config": {...}}`
- Server validates config (FFT size must be power of 2, averaging count 1-64) and applies
- Dynamic FFT size based on zoom level: if user zooms to <10 kHz span, increase FFT size for better resolution
- Measure end-to-end latency: SDR capture → agent → server → FFT → client, target <100ms p99

### Phase 4 — Frontend Spectrum Visualization (Days 17–21)

**Day 17: Frontend scaffold and WebSocket client**
```bash
npm create vite@latest frontend -- --template react-ts
cd frontend
npm i zustand @tanstack/react-query axios
npm i --save-dev @types/node
```
- `src/App.tsx` — Router (react-router-dom), auth context, layout with sidebar and main content
- `src/stores/spectrumStore.ts` — Zustand store for spectrum state (frequency range, FFT config, marker positions, waterfall buffer)
- `src/lib/websocket.ts` — WebSocket client with auto-reconnect, binary message parsing
- `src/hooks/useSpectrum.ts` — React hook for subscribing to spectrum WebSocket, parsing binary frames, updating store

**Day 18: Spectrum plot renderer (WebGL)**
- `src/components/SpectrumPlot/SpectrumPlot.tsx` — WebGL canvas for spectrum line plot
- `src/components/SpectrumPlot/shaders.ts` — Vertex shader (map freq/power to NDC), fragment shader (color coding by power level)
- GPU line rendering: upload frequency/power as vertex buffer, draw as GL_LINE_STRIP
- Axes: separate SVG overlay for frequency axis (log or linear scale) and power axis (dBm)
- Interaction: mouse hover shows freq/power tooltip, click to place marker

**Day 19: Waterfall display renderer (WebGL)**
- `src/components/WaterfallPlot/WaterfallPlot.tsx` — WebGL canvas for waterfall scrolling display
- `src/components/WaterfallPlot/waterfall-shader.ts` — Fragment shader with colormap lookup (see architecture section)
- Circular buffer for waterfall data: 1024 rows × 2048 bins (8 MB), wraps after ~30s at 30 FPS
- Texture update: `texSubImage2D` for single-row upload (<0.5ms)
- Colormap textures: precomputed 1D textures for viridis, plasma, magma, inferno (256 colors each)
- Interaction: click on waterfall → set playback cursor (for recording playback)

**Day 20: Frequency markers and measurements**
- `src/components/SpectrumPlot/Markers.tsx` — Overlay for vertical marker lines (SVG), draggable
- Marker table: display frequency, power at marker, delta to reference marker
- Peak search: find N highest peaks in spectrum, auto-place markers
- Channel power measurement: integrate power across user-defined bandwidth, display in dBW or dBm
- Occupied bandwidth (OBW): find bandwidth containing 99% of total power

**Day 21: FFT configuration UI and device controls**
- `src/components/ControlPanel/FftConfig.tsx` — Sliders for FFT size (logarithmic scale: 1K, 2K, 4K, ..., 4M), window type dropdown, averaging controls
- `src/components/ControlPanel/DeviceControls.tsx` — Center frequency input (with unit parsing: MHz, GHz), sample rate dropdown, gain slider, antenna port selector
- Tuning commands sent via WebSocket, debounced (500ms) to avoid spamming server
- Frequency input validation: check against device capabilities, highlight errors
- Quick-tune buttons: WiFi channels (2.4 GHz band), FM broadcast (88-108 MHz), ADS-B (1090 MHz)

### Phase 5 — Demodulation + Audio Playback (Days 22–26)

**Day 22: FM demodulation service**
- `src/dsp/demod/fm.rs` — FmDemodulator struct (see architecture section for full implementation)
- Demodulation worker: subscribe to IQ stream, demodulate to audio samples (48 kHz), stream via WebSocket
- Audio resampling: IQ at 2.4 MSPS → decimate to 240 kHz → FM demod → resample to 48 kHz (rational resampler)
- Stereo decoding: detect 19 kHz pilot tone, extract L-R channel at 38 kHz, combine with L+R for stereo output
- Test with local FM broadcast station

**Day 23: AM demodulation service**
- `src/dsp/demod/am.rs` — AmDemodulator struct (see architecture section)
- Envelope detection via magnitude of complex IQ
- DC blocking filter to remove carrier component
- Automatic Gain Control (AGC) to normalize audio level: target RMS = -20 dBFS, attack 10ms, release 500ms
- Test with AM broadcast station (530-1700 kHz in North America)

**Day 24: Audio streaming to browser**
- `src/api/ws/audio.rs` — WebSocket endpoint for audio streaming: /ws/audio/:device_id
- Opus encoding: encode demodulated audio (48 kHz mono/stereo) at 32 kbps for low bandwidth
- Binary WebSocket frames: [u8 opus_packet_size][u8... opus_data]
- Client-side Opus decoding via opus-decoder npm package or native Web Audio API Opus support

**Day 25: Web Audio API playback**
- `src/components/AudioPlayer/AudioPlayer.tsx` — Web Audio API context, AudioWorklet for low-latency playback
- `src/components/AudioPlayer/audio-worklet.ts` — AudioWorkletProcessor that receives Opus packets via MessagePort, decodes, queues to output buffer
- Playback buffer: 200ms latency budget (4 × 50ms Opus frames) for smooth playback without glitches
- Volume control, mute button, squelch (silence when signal power below threshold)

**Day 26: Audio visualization**
- `src/components/AudioPlayer/AudioSpectrum.tsx` — Real-time audio spectrum analyzer using Web Audio API AnalyserNode
- FFT on audio (1024 points at 48 kHz = 21 Hz resolution), display as bar graph (20 Hz - 20 kHz)
- VU meter for audio level monitoring
- Recording: capture audio to WAV file client-side via MediaRecorder API or manual PCM buffering

### Phase 6 — IQ Recording + Playback (Days 27–31)

**Day 27: IQ recording service**
- `src/api/handlers/recordings.rs` — POST /projects/:id/recordings/start, POST /recordings/:id/stop, GET /recordings, GET /recordings/:id, DELETE /recordings/:id
- Recording worker: subscribe to IQ stream from agent, buffer IQ samples, compress with zstd, write to R2 in chunks (10 MB chunks for resumable uploads)
- SigMF metadata generation: `.sigmf-meta` JSON file with core fields (sample_rate, center_frequency, datetime, datatype) and custom fields (device_serial, location, tags)
- R2 multipart upload for large recordings (>5 GB), presigned URLs for client download

**Day 28: Triggered recording**
- `src/dsp/trigger.rs` — Trigger detection: PowerThreshold (signal power exceeds threshold for N consecutive frames), SignalDetection (ML classifier detects specific signal type)
- Pre-trigger buffer: circular buffer of last 10 seconds of IQ samples, dump to recording when trigger fires
- Post-trigger duration: continue recording for configurable duration after trigger
- Trigger status indicator in UI: armed, triggered, recording

**Day 29: SigMF compliance and metadata**
- SigMF validator: validate generated `.sigmf-meta` files against SigMF schema
- Custom metadata extensions: `spectrumlab:device`, `spectrumlab:demod_config`, `spectrumlab:annotations`
- Capture annotations: segment markers within recording (e.g., "FM station detected at t=5.2s, f=100.1 MHz")
- Export metadata as JSON for external tools

**Day 30: Playback system**
- `src/api/handlers/playback.rs` — POST /recordings/:id/playback (create playback session), GET /playback/:id/stream (stream IQ samples)
- Playback worker: read IQ from R2, decompress, stream via WebSocket at original sample rate or adjustable speed (0.5x, 1x, 2x, 4x)
- Time scrubbing: jump to specific timestamp, fast-forward/rewind
- Playback UI: timeline with seek bar, play/pause, speed control, sync with waterfall cursor

**Day 31: Recording library UI**
- `src/pages/Recordings.tsx` — Table view with columns: name, frequency, bandwidth, duration, size, date, tags, annotation count
- Search and filter: frequency range slider, date range picker, tag multiselect, full-text search on name/description
- Batch operations: select multiple recordings, bulk delete, bulk export, bulk tag
- Spectrogram thumbnails: generate low-res spectrogram preview (256×128 px) on recording save, display in table
- Storage quota visualization: donut chart showing used/available storage per plan tier

### Phase 7 — Billing + Plan Enforcement (Days 32–36)

**Day 32: Stripe integration**
- `src/api/handlers/billing.rs` — POST /billing/checkout (create checkout session), POST /billing/portal (create customer portal session), GET /billing/subscription
- `src/api/handlers/webhooks/stripe.rs` — Handle Stripe webhook events: `checkout.session.completed`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.payment_failed`
- Plan mapping: Free (1 device, 5 GB storage), Pro (3 devices, 50 GB storage, $79/mo), Team (unlimited devices, 500 GB storage, $249/mo for 5 seats)
- Stripe test mode for development with test clock for subscription lifecycle testing

**Day 33: Usage tracking and plan limits**
- `src/middleware/plan_limits.rs` — Middleware checking plan limits before API actions (recording start, device registration)
- `src/services/usage.rs` — Track recording minutes, storage bytes, GPU minutes per billing period
- Usage record insertion after each recording completes, storage calculation on file upload
- Usage dashboard: `src/api/handlers/usage.rs` — GET /usage (current period usage, historical trends)
- Approaching-limit warnings at 80% and 100% of quota (email notifications)

**Day 34: Billing UI**
- `frontend/src/pages/Billing.tsx` — Plan comparison table (Free/Pro/Team), current plan display, usage meters
- `frontend/src/components/billing/PlanCard.tsx` — Plan feature cards with pricing, upgrade CTA
- `frontend/src/components/billing/UsageMeter.tsx` — Progress bars for storage, recording minutes with quota labels
- Upgrade/downgrade flow via Stripe Customer Portal (redirect to Stripe-hosted page)
- Upgrade prompt modals when hitting limits (e.g., "You've reached your 1 device limit. Upgrade to Pro for 3 devices.")

**Day 35: Feature gating**
- Gate recording download (>1 GB files) behind Pro plan
- Gate demodulation (AM/FM) behind Pro plan (Free has spectrum only)
- Gate annotation features behind Pro plan
- Gate team collaboration (org members, shared projects) behind Team plan
- Show locked feature indicators with upgrade CTAs in UI
- Admin endpoint: override plan for specific users (beta testers, partners)

**Day 36: Stripe webhook testing and subscription lifecycle**
- Test webhook delivery using Stripe CLI: `stripe listen --forward-to localhost:3000/webhooks/stripe`
- Test subscription creation → payment succeeded → user.plan updated to Pro
- Test subscription cancellation → subscription ended → user.plan reverted to Free
- Test payment failed → retry logic → subscription past_due → grace period → cancel
- Test plan upgrade (Free → Pro) and downgrade (Pro → Free) mid-cycle with prorating

### Phase 8 — Validation + Launch Prep (Days 37–42)

**Day 37: End-to-end integration testing**
- Test complete user journey: sign up → install agent → connect RTL-SDR → stream spectrum → record signal → annotate → playback → upgrade to Pro
- Test concurrent users: 10 users simultaneously streaming from different devices → verify no cross-contamination, no dropped frames
- Test device hot-plug: unplug RTL-SDR during stream → agent detects disconnect → UI shows offline → replug → reconnects automatically
- Test network interruption: disconnect WiFi during stream → agent buffers data → reconnects → stream resumes from correct position
- Load test: 50 concurrent users streaming → measure API latency (p50, p95, p99), GPU queue depth, database connection pool utilization

**Day 38: Performance benchmarks and optimization**
- Benchmark FFT latency: 1K-4M point FFTs on T4 GPU, verify <5ms target
- Benchmark waterfall frame rate: measure sustained 30 FPS over 60-second session with no dropped frames
- Benchmark agent compression: measure CPU overhead (<15%), compression ratio (2.5x), latency penalty (<2ms)
- Benchmark recording upload: 100 MB file → zstd compression → R2 multipart upload, verify <15s total
- Optimize slow queries: add indexes for recording library filtering, project list pagination
- Profile Rust backend: use flamegraph to identify CPU hotspots, optimize allocations in IQ processing loop

**Day 39: Security audit and hardening**
- SQL injection testing: verify all SQLx queries use parameterized statements, no raw string interpolation
- Authentication bypass testing: verify JWT validation on all protected endpoints, test expired tokens, invalid signatures
- Authorization testing: verify user can only access own projects/devices, cannot access other users' data via ID manipulation
- Input validation: verify frequency/sample rate bounds checking, reject malformed IQ frames, sanitize user inputs
- Rate limiting: add rate limits on auth endpoints (10 req/min), recording endpoints (1 start/min), API endpoints (100 req/min)
- CORS configuration: verify allowed origins restricted to production domains only
- HTTPS enforcement: redirect HTTP → HTTPS, HSTS header, secure cookies

**Day 40: Documentation and onboarding**
- User documentation: Getting Started guide (install agent, connect device, stream spectrum, record signal)
- API documentation: OpenAPI spec for all REST endpoints, WebSocket protocol documentation
- Agent installation guide: OS-specific instructions (Linux/macOS/Windows), troubleshooting common issues (driver installation, permissions)
- Developer documentation: contribution guide, architecture overview, local development setup
- Video tutorial: 5-minute walkthrough of basic workflow (connect RTL-SDR, tune to FM station, record, playback)
- FAQ page: common questions (supported devices, bandwidth requirements, plan limits, data privacy)

**Day 41: Deployment and monitoring setup**
- Kubernetes manifests: API deployment (3 replicas, HPA), FFT worker deployment (GPU node pool, 1-5 replicas), PostgreSQL StatefulSet, Redis deployment
- CloudFront distribution: frontend assets (S3 origin), agent binaries (R2 origin), custom domain with SSL certificate
- Prometheus scraping: API metrics (request rate, latency, error rate), GPU metrics (utilization, memory), database metrics (connection pool, query latency)
- Grafana dashboards: user activity (registrations, active sessions, recordings), infrastructure (CPU, memory, disk), DSP pipeline (FFT queue depth, latency distribution)
- Sentry error tracking: backend errors (panic, database errors), frontend errors (WebSocket disconnects, WebGL errors)
- Alert rules: API error rate >5%, GPU queue depth >100, database connection pool >90%, agent disconnect rate >10/min

**Day 42: Beta launch and monitoring**
- Deploy to production: EKS cluster (us-east-1), RDS PostgreSQL (Multi-AZ), ElastiCache Redis, CloudFront distribution
- Announce beta on r/RTLSDR, r/amateurradio, RTL-SDR.com blog, Twitter
- Monitor initial user signups: track registration rate, agent installation rate, first-session completion rate
- Monitor system health: API latency p99 <200ms, FFT latency p99 <10ms, waterfall frame rate >25 FPS, agent connection success rate >95%
- Collect user feedback: in-app feedback button, Discord community, email support queue
- Iterate on critical bugs: prioritize bugs blocking core workflow (agent connection failures, WebSocket disconnects, recording upload errors)
- Measure success metrics: 100+ beta signups in first week, 50+ active sessions/day, 20+ recordings saved/day, <5% churn rate

---

## Validation Benchmarks

### Benchmark 1: FFT Accuracy vs. NumPy Reference

**Test setup:** Generate synthetic IQ signal: 100 MHz carrier + 1 MHz offset single tone (CW) at -30 dBFS, 2.4 MSPS, 1M samples.

**Expected results:**
- Peak at 101 MHz (1 MHz offset from center)
- Peak power: -30 dBm ± 0.5 dB (calibrated to reference)
- Noise floor: <-80 dBm (16-bit quantization)
- Spurious peaks: <-70 dBc

**Validation:** Compare SpectrumLab FFT output to NumPy `np.fft.fft()` reference implementation. Verify bin frequencies match within 1 Hz, power levels within 0.5 dB.

### Benchmark 2: FM Demodulation THD+N

**Test setup:** Generate FM modulated signal: 100 MHz carrier, 75 kHz deviation, 1 kHz sine wave modulation tone at -20 dBFS audio level. Sample rate 2.4 MSPS.

**Expected results:**
- Demodulated audio frequency: 1000 Hz ± 1 Hz
- Demodulated audio level: -20 dBFS ± 1 dB
- THD+N (Total Harmonic Distortion + Noise): <1% (-40 dB)
- Stereo separation (for stereo FM): >40 dB

**Validation:** Run demodulated audio through FFT, verify fundamental at 1 kHz, measure harmonic levels (2 kHz, 3 kHz, ...), compute THD+N. Compare to GNU Radio FM demod block as reference.

### Benchmark 3: End-to-End Latency

**Test setup:** Inject IQ samples from signal generator into RTL-SDR, measure time from IQ capture to spectrum display update in browser.

**Measurement points:**
1. SDR capture → agent IQ buffer: <1ms
2. Agent IQ buffer → WebSocket send: ~2ms (compression)
3. Network transmission: ~20ms (assume 50 Mbps uplink)
4. Server WebSocket receive → FFT GPU queue: ~5ms
5. GPU FFT computation: ~5ms (4M-point FFT)
6. FFT result → WebSocket send: ~2ms
7. Network transmission: ~20ms (assume 100 Mbps downlink)
8. Browser receive → WebGL render: ~3ms
9. **Total: ~58ms (target <100ms p99)**

**Validation:** Insert timestamp at each stage, log latency distribution (p50, p95, p99). Under normal load, p99 latency should be <100ms. Under high load (10+ concurrent users), p99 <150ms acceptable.

### Benchmark 4: Waterfall Frame Rate

**Test setup:** Stream 2.4 MSPS IQ samples continuously, FFT size 2048, measure sustained frame rate of waterfall updates.

**Expected results:**
- Theoretical max: 2.4 MSPS / 2048 = 1172 FPS
- Practical max (after decimation for display): 60 FPS
- Target sustained: 30 FPS with no dropped frames under normal load

**Validation:** Monitor waterfall texture upload rate, WebSocket message rate, GPU FFT throughput. Verify no frame drops over 60-second continuous streaming session. If frame drops occur, identify bottleneck (network, GPU, browser rendering).

### Benchmark 5: Recording Compression Ratio and Throughput

**Test setup:** Record 60 seconds of real RF spectrum (FM broadcast band, 88-108 MHz, 2.4 MSPS, mix of strong and weak signals).

**Expected results:**
- Raw IQ size: 60s × 2.4 MSPS × 2 channels × 2 bytes = 576 MB
- Compressed size (zstd level 3): ~230 MB (2.5x compression ratio)
- Compression throughput: >10 MB/s (single-core CPU)
- R2 upload throughput: >5 MB/s (limited by network)
- Total recording time: <60 seconds (real-time)

**Validation:** Measure actual compressed size, verify compression ratio 2.0x-3.0x depending on signal occupancy. Verify R2 upload completes within recording duration (real-time constraint). Test playback: decompress and verify IQ samples match original within quantization error (<1% RMS error).

---

## Post-MVP Roadmap

### Version 1.1 (Weeks 7-9)

**Visual DSP Chain Builder**
- React Flow-based node graph editor for connecting DSP blocks (filters, demodulators, decoders)
- Block library: low-pass filter, high-pass filter, band-pass filter, decimator, resampler, FM demod, AM demod, file sink
- Real-time signal visualization at each connection point (waveform, spectrum, constellation)
- Chain templates: FM radio receiver, AM radio receiver, NOAA APT satellite decoder
- Chain save/load with versioning
- **Value:** Enables advanced users to build custom signal processing workflows without coding

**Digital Demodulation (PSK/QAM)**
- BPSK, QPSK, 8PSK demodulators with carrier recovery (Costas loop) and timing recovery (Gardner algorithm)
- 16-QAM, 64-QAM, 256-QAM demodulators with adaptive equalization
- Constellation diagram display with EVM (Error Vector Magnitude) measurement
- Eye diagram for timing quality visualization
- **Value:** Supports digital modulation analysis for satellite, amateur radio, and IoT protocols

### Version 1.2 (Weeks 10-12)

**Protocol Decoders (ADS-B/AIS)**
- ADS-B (1090 MHz) decoder: decode aircraft position, altitude, speed, flight ID with real-time map display
- AIS (162 MHz) decoder: decode ship position, speed, MMSI, vessel type with map display
- POCSAG (pager) decoder: decode alphanumeric messages with timestamp
- ACARS (aircraft communications) decoder: decode flight messages
- **Value:** Turn SpectrumLab into a multi-purpose signal intelligence platform

**ML Signal Classification**
- PyTorch-based signal classifier trained on 50K+ labeled IQ recordings
- Automatic detection: WiFi, LTE, 5G NR, Bluetooth, LoRa, Zigbee, radar, satellite downlinks
- Modulation classification: AM, FM, BPSK, QPSK, 16QAM, 64QAM, OFDM with confidence scores
- Custom classifier training: upload labeled recordings, fine-tune model, deploy to production
- **Value:** Automates signal identification, reduces manual annotation burden for power users

### Version 1.3 (Weeks 13-16)

**Multi-Device Wideband Capture**
- Combine 2+ SDR devices to capture wider bandwidth (e.g., 2× RTL-SDR = 4.8 MHz total)
- Automatic frequency stitching with overlap compensation
- Phase/amplitude calibration between devices
- Use case: wideband spectrum monitoring, cellular band capture (LTE 20 MHz + guard bands)
- **Value:** Overcomes single-device bandwidth limitations for professional users

**Geolocation (TDOA)**
- Time-difference-of-arrival (TDOA) signal source localization using 3+ synchronized SDR receivers
- Map display with estimated source location, confidence ellipse, and bearing lines
- Position history tracking for mobile transmitters
- Requires GPS-disciplined oscillators (GPSDO) for sub-microsecond time synchronization
- **Value:** Critical for spectrum enforcement, interference hunting, direction finding applications

**Monitoring Schedules and Alerts**
- Automated band scanning with configurable frequency ranges, dwell time, and detection thresholds
- Email/webhook/Slack alerts when signals matching criteria are detected (power threshold, specific modulation, ML classification)
- Alert history with spectrogram snapshots and automatic recording
- Scheduled reports: daily/weekly spectrum occupancy summaries
- **Value:** Enables 24/7 spectrum monitoring for regulatory compliance and interference detection

### Version 2.0 (Months 5-6)

**Enterprise Features**
- SAML 2.0 SSO integration (Okta, Azure AD, Google Workspace)
- Role-based access control (RBAC) with custom permissions
- Comprehensive audit logging (who accessed what, when, from where)
- On-premise deployment option for air-gapped/classified environments
- Dedicated customer success manager and SLA (99.9% uptime)
- **Value:** Required for enterprise sales to government, defense, and large telecom operators

**Regulatory Compliance Reports**
- FCC compliance report templates (Part 15, Part 18, Part 22) with automated measurements (EIRP, spurious emissions, OBW)
- ETSI compliance templates (EN 300 328, EN 301 489) with spectrum mask comparison
- ITU-R recommendations (SM.329, SM.1541) for spectrum monitoring
- PDF export with organization branding, digital signatures, and chain-of-custody
- **Value:** Critical for wireless product certification, spectrum license applications, and regulatory filings

---

## Go-to-Market Strategy

### Phase 1: Community Seeding (Month 1-3)
- Launch free tier targeting RTL-SDR community on r/RTLSDR, r/amateurradio, RTL-SDR.com blog
- Publish "Getting Started with RTL-SDR in Your Browser" tutorial series on YouTube
- Sponsor Hamvention, DEFCON RF Village, GNU Radio Conference for visibility
- Release local agent as open-source (MIT license) to build trust and community contributions
- Partner with SDR hardware vendors (RTL-SDR.com, Great Scott Gadgets) for bundled free tier promotion

### Phase 2: Professional Adoption (Month 3-6)
- Launch Pro tier targeting wireless engineers, RF consultants, and small research labs
- Publish performance benchmarks: SpectrumLab vs. commercial spectrum analyzers (Keysight, R&S) for cost/performance comparison
- SEO content: "cloud spectrum analyzer", "SDR in browser", "GNU Radio alternative", "remote spectrum monitoring"
- Target IoT developers with LoRa/Zigbee/BLE analysis use cases
- Exhibit at IEEE Wireless Communications conferences, SDR Forum events

### Phase 3: Enterprise and Government (Month 6-12)
- Launch Team and Enterprise tiers with monitoring, geolocation, compliance reporting
- Target spectrum management agencies (FCC, Ofcom, ACMA), telecom regulators, defense contractors
- Build FCC/ETSI compliance report templates for regulatory customers
- Pursue security certifications (SOC 2, FedRAMP) for government sales
- Partner with Ettus Research (USRP), Analog Devices (PlutoSDR) for joint solutions

### Acquisition Channels
- Organic search: "online spectrum analyzer", "cloud SDR platform", "RF signal analysis tool"
- RTL-SDR and amateur radio community engagement (Reddit, forums, Discord)
- Hardware vendor partnerships: bundled free tier with SDR device purchases
- Academic partnerships: free Pro tier for university RF courses, research labs
- Conference presentations: live demos at DEFCON, GNU Radio Con, IEEE events

---

## Success Metrics & KPIs

| Metric | Target (Month 3) | Target (Month 6) | Target (Month 12) |
|--------|------------------|------------------|-------------------|
| Registered users | 5,000 | 15,000 | 60,000 |
| Monthly active projects | 1,500 | 5,000 | 20,000 |
| Paying customers (Pro + Team) | 50 | 200 | 1,000 |
| MRR | $4,000 | $16,000 | $80,000 |
| SDR devices connected | 2,000 | 8,000 | 30,000 |
| Recordings stored (hours) | 5,000 | 25,000 | 150,000 |
| Free → Paid conversion rate | 2% | 4% | 6% |
| Monthly churn rate | <8% | <6% | <4% |
| GPU utilization (avg) | 30% | 50% | 70% |
| Agent install success rate | >90% | >95% | >98% |

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | SpectrumLab Advantage |
|-----------|-----------|------------|----------------------|
| SDR# (Airspy) | Free, good spectrum display, easy to use | Windows-only, no cloud features, limited analysis, no collaboration | Cross-platform browser app, cloud recording library, team features |
| GNU Radio | Free/open-source, extremely flexible, large community | Steep learning curve, desktop-only, fragile installs, no collaboration | Browser-based, visual DSP chains, zero-install, team annotation |
| MATLAB Signal Processing | Gold-standard DSP algorithms, extensive documentation | $2,500+/seat/year, desktop-only, not real-time SDR, single-user | 30x cheaper, real-time SDR integration, browser-based, multi-user |
| Keysight FieldFox | Professional measurements, regulatory compliance, rugged hardware | $20K+ analyzer tied to proprietary hardware, no SDR support | Works with any SDR, 200x cheaper, cloud platform, open ecosystem |
| CloudRF | Cloud-based RF propagation modeling, coverage prediction | Focused on propagation/planning, no real-time SDR, no signal analysis | Real-time SDR streaming, signal analysis and classification, recording library |
| WebSDR (Multiple) | Browser-based SDR receivers, public access | Read-only (listen-only), no recording, no analysis tools, shared bandwidth | Full control over private SDR, recording/annotation/analysis, dedicated bandwidth |

**Differentiation:**
- Only cloud platform combining real-time SDR streaming, GPU-accelerated FFT, and collaborative recording library
- Works with any SDR hardware (not locked to specific vendor)
- 10-100x cheaper than commercial spectrum analyzers for equivalent analysis capability
- Browser-based eliminates installation friction, enables Chromebook/iPad access
- Team features (annotation, commenting, sharing) not available in any desktop SDR tool

---

## Risk Mitigation

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| IQ streaming bandwidth too high for consumer internet | High | Medium | Implement adaptive quality (reduce sample rate), hybrid local FFT + spectral streaming for bandwidth-constrained users |
| GPU FFT costs exceed revenue at scale | High | Low | Batch processing across users, dynamic scaling based on demand, CPU fallback for low-traffic periods, optimize FFT kernel |
| SDR driver compatibility issues across OS/devices | Medium | High | Use SoapySDR abstraction, maintain device compatibility matrix, community-driven testing, provide pre-compiled binaries with drivers |
| Latency >200ms makes real-time unusable | Medium | Medium | Optimize pipeline (WASM FFT for low latency, edge CDN for WebSocket termination), local FFT fallback, latency budget monitoring |
| Security: malicious user floods GPU queue | Medium | Low | Per-user rate limiting, plan-based GPU quotas, queue depth monitoring with auto-scale, authentication on WebSocket endpoints |
| Competition from SDR hardware vendors | Low | Medium | Stay hardware-agnostic, focus on analysis/collaboration features vendors won't build, open-source agent for community trust |
| Regulatory concerns about spectrum monitoring | Low | Low | Geofencing for restricted frequencies, comply with local regulations, offer on-premise for sensitive applications |

