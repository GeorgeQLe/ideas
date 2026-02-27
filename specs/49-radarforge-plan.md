# 49. RadarForge — Radar System Design and Signal Processing Simulation

## Implementation Plan

**MVP Scope:** Browser-based radar system designer with drag-and-drop antenna element configuration (dipole, patch, horn) and phased array geometry editor rendered via Canvas/WebGL, LFM chirp and FMCW waveform generator with real-time ambiguity function computation compiled to WebAssembly for client-side execution of small arrays (≤64 elements) and server-side Rust-native execution for larger arrays and Monte Carlo simulations, support for free-space propagation with radar range equation, basic land/sea clutter models (Barrie, GIT), Swerling target RCS models, configurable receiver signal processing chain with matched filter, pulse compression, range-Doppler map generation, CA-CFAR detector, and beamforming (conventional and MVDR), interactive 2D/3D visualization rendered via WebGL with range-Doppler heatmaps and antenna pattern plots, 500+ built-in radar system templates and target scenarios stored in S3 with PostgreSQL metadata, scenario import/export compatible with common radar simulation formats, Stripe billing with three tiers (Free / Pro $99/mo / Defense $399/user/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| Signal Processing | Rust (native + WASM) | Custom DSP engine with RustFFT, ndarray for matrix ops |
| WASM Compilation | `wasm-pack` + `wasm-bindgen` | Client-side processing for arrays ≤64 elements |
| Propagation/Clutter | Python 3.12 (FastAPI) | ITU-R atmospheric models, advanced clutter generation |
| Database | PostgreSQL 16 | Projects, users, simulations, antenna/waveform libraries |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | Scenario data, simulation results, RCS tables, antenna patterns |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| Antenna/Target Viz | WebGL 2.0 + Canvas | 3D antenna patterns, range-Doppler heatmaps |
| 2D Plotting | D3.js v7 | Ambiguity functions, Bode plots, detection curves |
| Real-time | WebSocket (Axum) | Live simulation progress, Monte Carlo streaming |
| Job Queue | Redis 7 + Tokio tasks | Server-side simulation job management, batch processing |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | WASM bundle delivery, static frontend assets |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, error tracking |
| CI/CD | GitHub Actions | WASM build pipeline, Rust tests, Docker image push |

### Key Architecture Decisions

1. **Hybrid client/server processing with WASM threshold at 64 array elements**: Small antenna arrays (≤64 elements, covering most educational scenarios and basic phased arrays) compute ambiguity functions, array factors, and range-Doppler maps entirely in the browser via WASM, providing instant feedback with zero server cost. Larger arrays (AESA radars with 256-4096 elements), Monte Carlo sensitivity sweeps, and SAR imaging are processed server-side with parallelized Rust solvers. The threshold is configurable per plan tier.

2. **Custom Rust DSP engine rather than wrapping MATLAB/GNU Radio**: Building a custom signal processing pipeline in Rust gives us full control over WASM compilation, GPU acceleration hooks, and memory-efficient streaming for large datasets. MATLAB's runtime licensing model makes cloud deployment expensive, and GNU Radio's Python/C++ architecture doesn't compile to WASM. Our Rust engine uses RustFFT for radix-2/mixed-radix transforms and ndarray for vectorized beamforming operations while maintaining memory safety.

3. **WebGL for 3D antenna patterns and range-Doppler visualization**: WebGL enables GPU-accelerated rendering of 3D antenna radiation patterns (directivity vs theta/phi), range-Doppler heatmaps with millions of bins, and real-time pan/zoom/rotate interaction. We use instanced rendering for array element visualization and compute shaders (WebGL 2.0 + extensions) for CFAR processing visualization. Canvas 2D is used for waveform time-series and ambiguity function contour plots.

4. **Separate Python FastAPI service for advanced propagation models**: Atmospheric attenuation (ITU-R P.676, P.838), multipath ray tracing, and polarimetric scattering require complex numerical integration and lookup tables. These are implemented in Python using NumPy/SciPy, exposed via FastAPI microservice that Rust calls for server-side simulations. This avoids reimplementing validated ITU models while keeping the critical path (signal processing) in Rust.

5. **PostgreSQL for parametric search of antenna/waveform libraries with S3 blob storage**: Antenna element patterns (radiation tables, S-parameters) and waveform templates are stored in S3 as binary arrays (HDF5/Parquet format), while PostgreSQL holds searchable metadata (frequency band, gain, beamwidth, polarization, category). This allows the library to scale to 10K+ patterns without bloating the database while enabling fast parametric search via PostgreSQL indexes and range queries.

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
    plan TEXT NOT NULL DEFAULT 'free',  -- free | pro | defense
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX users_email_idx ON users(email);
CREATE INDEX users_stripe_idx ON users(stripe_customer_id);

-- Organizations (for Defense/Team plan)
CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    slug TEXT UNIQUE NOT NULL,
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    plan TEXT NOT NULL DEFAULT 'free',
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    settings JSONB DEFAULT '{}',
    is_itar_compliant BOOLEAN DEFAULT false,  -- GovCloud deployment flag
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

-- Projects (radar system design workspace)
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    system_config JSONB NOT NULL DEFAULT '{}',  -- Waveform, antenna, target scenario config
    settings JSONB DEFAULT '{}',  -- Display preferences, units
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
    simulation_type TEXT NOT NULL,  -- ambiguity | range_doppler | detection_curve | tracking | sar_image
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | running | completed | failed | cancelled
    execution_mode TEXT NOT NULL DEFAULT 'wasm',  -- wasm | server
    config JSONB NOT NULL DEFAULT '{}',  -- Simulation-specific parameters
    array_element_count INTEGER NOT NULL DEFAULT 1,
    results_url TEXT,  -- S3 URL for result data
    results_summary JSONB,  -- Quick-access summary (peak sidelobes, resolution, SNR)
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
    memory_mb INTEGER DEFAULT 4096,
    gpu_requested BOOLEAN DEFAULT false,  -- For SAR imaging
    priority INTEGER DEFAULT 0,  -- Higher = more urgent
    progress_pct REAL DEFAULT 0.0,
    progress_data JSONB DEFAULT '{}',  -- {current_pulse, total_pulses, current_target, ...}
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX jobs_sim_idx ON simulation_jobs(simulation_id);
CREATE INDEX jobs_worker_idx ON simulation_jobs(worker_id);
CREATE INDEX jobs_status_idx ON simulation_jobs(worker_id, started_at) WHERE completed_at IS NULL;

-- Antenna Element Patterns (metadata; actual pattern files in S3)
CREATE TABLE antenna_patterns (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    element_type TEXT NOT NULL,  -- dipole | patch | horn | slot | vivaldi | spiral | helix
    frequency_band TEXT NOT NULL,  -- VHF | UHF | L | S | C | X | Ku | Ka | W (77GHz)
    center_freq_ghz REAL NOT NULL,
    bandwidth_mhz REAL,
    polarization TEXT NOT NULL,  -- linear_h | linear_v | rhcp | lhcp | dual
    gain_dbi REAL,
    beamwidth_deg REAL,  -- 3dB beamwidth
    front_back_ratio_db REAL,
    pattern_data_url TEXT NOT NULL,  -- S3 URL to radiation pattern (theta, phi, gain_db)
    s_parameters_url TEXT,  -- S11, S21, ... for matching analysis
    manufacturer TEXT,
    part_number TEXT,
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
CREATE INDEX patterns_type_idx ON antenna_patterns(element_type);
CREATE INDEX patterns_freq_idx ON antenna_patterns(center_freq_ghz);
CREATE INDEX patterns_band_idx ON antenna_patterns(frequency_band);
CREATE INDEX patterns_name_trgm_idx ON antenna_patterns USING gin(name gin_trgm_ops);
CREATE INDEX patterns_tags_idx ON antenna_patterns USING gin(tags);

-- Waveform Templates
CREATE TABLE waveform_templates (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    waveform_type TEXT NOT NULL,  -- lfm_chirp | fmcw_sawtooth | fmcw_triangle | barker | polyphase | ofdm
    category TEXT NOT NULL,  -- pulsed | continuous | coded
    frequency_band TEXT,
    center_freq_ghz REAL,
    bandwidth_mhz REAL,
    pulse_width_us REAL,  -- For pulsed waveforms
    prf_khz REAL,  -- Pulse repetition frequency
    duty_cycle REAL,  -- For FMCW
    parameters JSONB NOT NULL DEFAULT '{}',  -- Type-specific params (chirp rate, code sequence, etc.)
    ambiguity_thumbnail_url TEXT,  -- Precomputed ambiguity function image
    application TEXT,  -- automotive | weather | defense | aviation | maritime
    description TEXT,
    tags TEXT[] DEFAULT '{}',
    is_builtin BOOLEAN DEFAULT true,
    uploaded_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX waveforms_type_idx ON waveform_templates(waveform_type);
CREATE INDEX waveforms_band_idx ON waveform_templates(frequency_band);
CREATE INDEX waveforms_app_idx ON waveform_templates(application);
CREATE INDEX waveforms_tags_idx ON waveform_templates USING gin(tags);

-- Target Scenarios
CREATE TABLE target_scenarios (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    scenario_type TEXT NOT NULL,  -- single_target | multi_target | clutter | jamming | automotive
    targets JSONB NOT NULL DEFAULT '[]',  -- [{rcs_dbsm, range_km, velocity_mps, azimuth_deg, elevation_deg}]
    clutter_config JSONB,  -- {type: land|sea, model: barrie|billingsley|git, wind_speed, roughness}
    jamming_config JSONB,  -- {type: noise|deceptive|drfm, power_dbm, frequency_offset}
    environment JSONB DEFAULT '{}',  -- {temperature_c, humidity, rain_rate_mm_hr, fog}
    description TEXT,
    tags TEXT[] DEFAULT '{}',
    is_builtin BOOLEAN DEFAULT true,
    uploaded_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX scenarios_type_idx ON target_scenarios(scenario_type);
CREATE INDEX scenarios_tags_idx ON target_scenarios USING gin(tags);

-- Saved Visualization Configurations
CREATE TABLE visualization_configs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    simulation_id UUID NOT NULL REFERENCES simulations(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    config JSONB NOT NULL,  -- {view_type, camera_position, color_map, thresholds, annotations}
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX viz_sim_idx ON visualization_configs(simulation_id);

-- Usage Tracking (for billing enforcement)
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,  -- compute_minutes | monte_carlo_runs | storage_gb
    quantity REAL NOT NULL,
    period_start DATE NOT NULL,
    period_end DATE NOT NULL,
    metadata JSONB DEFAULT '{}',
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
    pub system_config: serde_json::Value,
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
    pub simulation_type: String,
    pub status: String,
    pub execution_mode: String,
    pub config: serde_json::Value,
    pub array_element_count: i32,
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
    pub gpu_requested: bool,
    pub priority: i32,
    pub progress_pct: f32,
    pub progress_data: serde_json::Value,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct AntennaPattern {
    pub id: Uuid,
    pub name: String,
    pub element_type: String,
    pub frequency_band: String,
    pub center_freq_ghz: f32,
    pub bandwidth_mhz: Option<f32>,
    pub polarization: String,
    pub gain_dbi: Option<f32>,
    pub beamwidth_deg: Option<f32>,
    pub front_back_ratio_db: Option<f32>,
    pub pattern_data_url: String,
    pub s_parameters_url: Option<String>,
    pub manufacturer: Option<String>,
    pub part_number: Option<String>,
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

#[derive(Debug, FromRow, Serialize)]
pub struct WaveformTemplate {
    pub id: Uuid,
    pub name: String,
    pub waveform_type: String,
    pub category: String,
    pub frequency_band: Option<String>,
    pub center_freq_ghz: Option<f32>,
    pub bandwidth_mhz: Option<f32>,
    pub pulse_width_us: Option<f32>,
    pub prf_khz: Option<f32>,
    pub duty_cycle: Option<f32>,
    pub parameters: serde_json::Value,
    pub ambiguity_thumbnail_url: Option<String>,
    pub application: Option<String>,
    pub description: Option<String>,
    pub tags: Vec<String>,
    pub is_builtin: bool,
    pub uploaded_by: Option<Uuid>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct TargetScenario {
    pub id: Uuid,
    pub name: String,
    pub scenario_type: String,
    pub targets: serde_json::Value,
    pub clutter_config: Option<serde_json::Value>,
    pub jamming_config: Option<serde_json::Value>,
    pub environment: serde_json::Value,
    pub description: Option<String>,
    pub tags: Vec<String>,
    pub is_builtin: bool,
    pub uploaded_by: Option<Uuid>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum SimulationType {
    Ambiguity,
    RangeDoppler,
    DetectionCurve,
    Tracking,
    SarImage,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct WaveformParams {
    pub waveform_type: WaveformType,
    pub center_freq_ghz: f64,
    pub bandwidth_mhz: f64,
    pub pulse_width_us: Option<f64>,
    pub prf_khz: Option<f64>,
    pub lfm_params: Option<LfmParams>,
    pub fmcw_params: Option<FmcwParams>,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum WaveformType {
    LfmChirp,
    FmcwSawtooth,
    FmcwTriangle,
    Barker,
    Polyphase,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct LfmParams {
    pub chirp_rate_mhz_per_us: f64,  // Bandwidth / pulse_width
    pub direction: String,  // "up" or "down"
}

#[derive(Debug, Deserialize, Serialize)]
pub struct FmcwParams {
    pub sweep_time_ms: f64,
    pub modulation_type: String,  // "sawtooth" | "triangle"
}

#[derive(Debug, Deserialize, Serialize)]
pub struct AntennaArrayConfig {
    pub array_type: String,  // ula | ura | circular | custom
    pub element_count: usize,
    pub element_spacing_lambda: f64,  // Spacing in wavelengths
    pub element_pattern_id: Option<Uuid>,
    pub steering_angle_deg: (f64, f64),  // (azimuth, elevation)
    pub taper_type: Option<String>,  // uniform | taylor | chebyshev | hamming
    pub taper_params: Option<serde_json::Value>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct TargetConfig {
    pub rcs_dbsm: f64,  // Radar cross section in dBsm
    pub swerling_case: Option<u8>,  // 0-4 (0 = constant)
    pub range_km: f64,
    pub velocity_mps: f64,  // Radial velocity (positive = approaching)
    pub azimuth_deg: f64,
    pub elevation_deg: f64,
    pub acceleration_mps2: Option<f64>,
}
```

---

## Signal Processing Architecture Deep-Dive

### Radar Equation and Link Budget

RadarForge's core propagation model uses the **Radar Range Equation** in its general form for monostatic pulsed radar:

```
SNR = (P_t · G_t · G_r · λ² · σ) / ((4π)³ · R⁴ · k · T_s · B_n · L_sys)

where:
  P_t       = Peak transmit power (W)
  G_t, G_r  = Transmit and receive antenna gain (linear, not dB)
  λ         = Wavelength (m)
  σ         = Target radar cross section (m²)
  R         = Range to target (m)
  k         = Boltzmann constant (1.38e-23 J/K)
  T_s       = System noise temperature (K) = T_0 · (NF - 1) + T_0
  B_n       = Receiver noise bandwidth (Hz)
  L_sys     = System losses (linear): L_atm · L_beam · L_mismatch · L_processing
```

For **FMCW radar** (continuous wave), the equation becomes:

```
SNR = (P_t · G² · λ² · σ · T_dwell) / ((4π)³ · R⁴ · k · T_s · L_sys)

where T_dwell is the coherent integration time (chirp duration)
```

**Atmospheric attenuation** (ITU-R P.676 for oxygen/water vapor, P.838 for rain) is computed by the Python propagation service:

```python
# Example: specific attenuation at 77 GHz (automotive radar)
γ_oxygen = 0.05 dB/km  (at sea level, 15°C, 7.5 g/m³ humidity)
γ_water  = 0.12 dB/km
γ_rain   = R^1.12 dB/km  (R = rain rate in mm/hr, K-R relation)

L_atm_dB = (γ_oxygen + γ_water + γ_rain) · R_km · 2  (two-way path)
```

### Waveform Generation and Ambiguity Function

**LFM Chirp** (Linear Frequency Modulation):

```
s(t) = A · exp(j · 2π · (f_c · t + 0.5 · μ · t²))

where:
  μ = chirp rate = B / T_p  (B = bandwidth, T_p = pulse width)
  f_c = center frequency
```

The **ambiguity function** χ(τ, f_d) measures range and Doppler resolution:

```
χ(τ, f_d) = ∫ s(t) · s*(t - τ) · exp(j · 2π · f_d · t) dt

For LFM chirp:
  Range resolution: ΔR = c / (2B)
  Doppler resolution: Δv = λ / (2 · T_cpi)  (T_cpi = coherent processing interval)
```

**FMCW Beat Frequency** (sawtooth modulation):

```
f_beat = (2 · B · R) / (c · T_sweep) + (2 · v · f_c) / c

Range-velocity coupling must be resolved using triangular modulation or multiple chirp rates.
```

### Phased Array Beamforming

**Uniform Linear Array (ULA)** with N elements, spacing d:

```
Array Factor:  AF(θ) = Σ_{n=0}^{N-1} w_n · exp(j · n · k · d · sin(θ))

where:
  w_n = element weight (amplitude and phase taper)
  k = 2π / λ
  θ = angle from array broadside
```

**Beam steering** to angle θ_0:

```
Phase shift:  φ_n = -n · k · d · sin(θ_0)
```

**Grating lobe** condition (first grating lobe appears when):

```
d / λ > 1 / (1 + |sin(θ_0)|)
```

**Beamwidth** (half-power, 3dB):

```
For uniform weighting:  HPBW ≈ 0.886 · λ / (N · d)
For Taylor weighting:   HPBW ≈ factor · λ / (N · d)  (factor depends on sidelobe level)
```

**MVDR (Minimum Variance Distortionless Response) beamformer**:

```
w_MVDR = R_xx^(-1) · a(θ_s) / (a^H(θ_s) · R_xx^(-1) · a(θ_s))

where:
  R_xx = spatial covariance matrix (estimated from data)
  a(θ_s) = steering vector for signal direction θ_s
```

### Matched Filter and Pulse Compression

**Matched filter** impulse response:

```
h(t) = s*(-t)  (time-reversed complex conjugate of transmitted waveform)
```

**Pulse compression ratio**:

```
PCR = T_p · B  (time-bandwidth product)

For LFM:  PCR = 100-1000 typical
For Barker code: PCR = code length (e.g., 13 for Barker-13)
```

**Output SNR gain**:

```
SNR_out / SNR_in = PCR  (assuming matched filter)
```

**Sidelobe weighting**: Apply window function (Hamming, Taylor, Kaiser) to reduce range sidelobes at the cost of mainlobe widening.

### Range-Doppler Processing

**Pulse-Doppler processing** (coherent pulse train):

```
1. Matched filter each pulse → fast-time dimension (range bins)
2. Doppler FFT across pulses → slow-time dimension (velocity bins)
3. Result: 2D range-Doppler map (RDM)

Range bin spacing:   ΔR = c / (2 · B)
Doppler bin spacing: Δf_d = PRF / N_pulses
Velocity resolution: Δv = λ · PRF / (2 · N_pulses)
```

**Unambiguous range and velocity**:

```
R_unamb = c / (2 · PRF)
v_unamb = λ · PRF / 4  (for pulsed Doppler)
```

**MTI (Moving Target Indication)** filter (two-pulse canceller):

```
y[n] = x[n] - x[n-1]

Clutter rejection: ~20-30 dB for stationary clutter
Multi-pulse MTI: higher-order filters for better rejection
```

### CFAR Detection

**CA-CFAR (Cell-Averaging CFAR)**:

```
Threshold:  T = α · (1 / N_ref) · Σ_{i ∈ guard+ref} |x_i|²

where:
  α = threshold multiplier (set by desired P_fa)
  N_ref = number of reference cells
  Guard cells prevent target energy leakage into reference estimate
```

**Detection probability** (Swerling I target, Rayleigh noise):

```
P_d = exp(-T / (SNR + 1))  (simplified)

For Swerling II/III/IV, use different fading models.
```

**OS-CFAR** (Order-Statistic CFAR) for heterogeneous clutter:

```
Threshold based on kth-order statistic of reference cells (more robust than CA-CFAR)
```

### Client/Server Split (WASM Threshold)

```
Array configuration → Element count extracted
    │
    ├── ≤64 elements → WASM processing (browser)
    │   ├── Instant ambiguity function, array factor
    │   ├── Range-Doppler map for small pulse counts (<1000)
    │   └── No server cost
    │
    └── >64 elements → Server processing (Rust native)
        ├── Large AESA (256-4096 elements)
        ├── Monte Carlo sweeps (1000+ trials)
        ├── SAR imaging (GPU-accelerated)
        └── Results stored in S3
```

The 64-element threshold was chosen because:
- WASM FFT handles 64-element × 512-pulse RDM in <500ms on modern hardware
- 64 elements covers: automotive MIMO (12-16 virtual), educational ULAs (8-32), basic phased arrays
- Above 64: fighter AESA (1000+ elements), spaceborne SAR, large ground-based radars → need server compute

---

## Architecture Deep-Dives

### 1. Simulation API Handler (Rust/Axum)

The primary endpoint receives a simulation request, validates the radar configuration, decides between WASM and server execution, and for server execution enqueues a job with Redis and streams progress via WebSocket.

```rust
// src/api/handlers/simulation.rs

use axum::{
    extract::{Path, State, Json},
    http::StatusCode,
    response::IntoResponse,
};
use uuid::Uuid;
use crate::{
    db::models::{Simulation, SimulationType, WaveformParams, AntennaArrayConfig},
    processing::radar_equation::compute_snr,
    state::AppState,
    auth::Claims,
    error::ApiError,
};

#[derive(serde::Deserialize)]
pub struct CreateSimulationRequest {
    pub simulation_type: SimulationType,
    pub waveform: WaveformParams,
    pub antenna: AntennaArrayConfig,
    pub targets: Vec<crate::db::models::TargetConfig>,
    pub environment: serde_json::Value,
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

    // 2. Validate antenna array configuration
    let element_count = req.antenna.element_count;
    if element_count < 1 || element_count > 8192 {
        return Err(ApiError::Validation("Element count must be 1-8192".into()));
    }

    // 3. Check plan limits
    let user = sqlx::query_as!(
        crate::db::models::User,
        "SELECT * FROM users WHERE id = $1",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    if user.plan == "free" && element_count > 16 {
        return Err(ApiError::PlanLimit(
            "Free plan supports arrays up to 16 elements. Upgrade to Pro for larger arrays."
        ));
    }

    // 4. Determine execution mode
    let execution_mode = if element_count <= 64 &&
        matches!(req.simulation_type, SimulationType::Ambiguity | SimulationType::RangeDoppler) {
        "wasm"
    } else {
        "server"
    };

    // 5. Quick SNR estimate for validation
    let _snr_db = compute_snr(
        &req.waveform,
        &req.antenna,
        req.targets.first().ok_or(ApiError::Validation("At least one target required".into()))?,
        &req.environment,
    )?;

    // 6. Create simulation record
    let config = serde_json::json!({
        "waveform": req.waveform,
        "antenna": req.antenna,
        "targets": req.targets,
        "environment": req.environment,
    });

    let sim = sqlx::query_as!(
        Simulation,
        r#"INSERT INTO simulations
            (project_id, user_id, simulation_type, status, execution_mode,
             config, array_element_count)
        VALUES ($1, $2, $3, $4, $5, $6, $7)
        RETURNING *"#,
        project_id,
        claims.user_id,
        serde_json::to_string(&req.simulation_type)?,
        if execution_mode == "wasm" { "ready" } else { "pending" },
        execution_mode,
        config,
        element_count as i32,
    )
    .fetch_one(&state.db)
    .await?;

    // 7. For server execution, enqueue job
    if execution_mode == "server" {
        let gpu_requested = matches!(req.simulation_type, SimulationType::SarImage);

        let job = sqlx::query_as!(
            crate::db::models::SimulationJob,
            r#"INSERT INTO simulation_jobs
                (simulation_id, priority, gpu_requested, cores_allocated, memory_mb)
            VALUES ($1, $2, $3, $4, $5) RETURNING *"#,
            sim.id,
            if user.plan == "defense" { 10 } else if user.plan == "pro" { 5 } else { 0 },
            gpu_requested,
            if element_count > 512 { 8 } else { 4 },
            if gpu_requested { 16384 } else { 8192 },
        )
        .fetch_one(&state.db)
        .await?;

        // Publish to Redis job queue (priority queue)
        let queue_name = if gpu_requested { "simulation:jobs:gpu" } else { "simulation:jobs:cpu" };
        state.redis
            .publish(queue_name, serde_json::to_string(&job.id)?)
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

pub async fn cancel_simulation(
    State(state): State<AppState>,
    claims: Claims,
    Path((project_id, sim_id)): Path<(Uuid, Uuid)>,
) -> Result<impl IntoResponse, ApiError> {
    // Verify ownership and update status
    let result = sqlx::query!(
        "UPDATE simulations s SET status = 'cancelled'
         FROM projects p
         WHERE s.id = $1 AND s.project_id = $2 AND s.project_id = p.id
         AND (p.owner_id = $3 OR p.org_id IN (
             SELECT org_id FROM org_members WHERE user_id = $3
         ))
         AND s.status IN ('pending', 'running')
         RETURNING s.id",
        sim_id,
        project_id,
        claims.user_id
    )
    .fetch_optional(&state.db)
    .await?;

    if result.is_none() {
        return Err(ApiError::NotFound("Simulation not found or already completed"));
    }

    // Cancel any pending job
    sqlx::query!(
        "UPDATE simulation_jobs SET completed_at = NOW()
         WHERE simulation_id = $1 AND completed_at IS NULL",
        sim_id
    )
    .execute(&state.db)
    .await?;

    Ok(StatusCode::NO_CONTENT)
}
```

### 2. Signal Processing Core (Rust — shared between WASM and native)

The core DSP engine that generates waveforms, computes ambiguity functions, performs matched filtering, and generates range-Doppler maps. This code compiles to both native (server) and WASM (browser) targets.

```rust
// processing-core/src/waveform.rs

use num_complex::Complex64;
use std::f64::consts::PI;

pub struct Waveform {
    pub samples: Vec<Complex64>,
    pub sample_rate_hz: f64,
    pub duration_s: f64,
}

impl Waveform {
    /// Generate LFM (Linear Frequency Modulation) chirp
    pub fn lfm_chirp(
        center_freq_hz: f64,
        bandwidth_hz: f64,
        pulse_width_s: f64,
        sample_rate_hz: f64,
        direction: &str,  // "up" or "down"
    ) -> Self {
        let n_samples = (pulse_width_s * sample_rate_hz) as usize;
        let chirp_rate = bandwidth_hz / pulse_width_s;  // Hz/s
        let direction_sign = if direction == "down" { -1.0 } else { 1.0 };

        let samples: Vec<Complex64> = (0..n_samples)
            .map(|i| {
                let t = i as f64 / sample_rate_hz;
                let phase = 2.0 * PI * (center_freq_hz * t +
                    0.5 * direction_sign * chirp_rate * t * t);
                Complex64::new(0.0, phase).exp()
            })
            .collect();

        Self {
            samples,
            sample_rate_hz,
            duration_s: pulse_width_s,
        }
    }

    /// Generate FMCW sawtooth waveform
    pub fn fmcw_sawtooth(
        center_freq_hz: f64,
        bandwidth_hz: f64,
        sweep_time_s: f64,
        sample_rate_hz: f64,
    ) -> Self {
        let n_samples = (sweep_time_s * sample_rate_hz) as usize;
        let chirp_rate = bandwidth_hz / sweep_time_s;

        let samples: Vec<Complex64> = (0..n_samples)
            .map(|i| {
                let t = i as f64 / sample_rate_hz;
                let inst_freq = center_freq_hz - bandwidth_hz / 2.0 + chirp_rate * t;
                let phase = 2.0 * PI * (inst_freq * t);
                Complex64::new(0.0, phase).exp()
            })
            .collect();

        Self {
            samples,
            sample_rate_hz,
            duration_s: sweep_time_s,
        }
    }

    /// Compute ambiguity function |χ(τ, f_d)|
    pub fn ambiguity_function(
        &self,
        tau_bins: usize,
        doppler_bins: usize,
    ) -> Vec<Vec<f64>> {
        let n = self.samples.len();
        let mut chi = vec![vec![0.0; doppler_bins]; tau_bins];

        let max_tau_samples = (tau_bins / 2) as isize;
        let max_doppler_hz = self.sample_rate_hz / 2.0;

        for (tau_idx, tau) in (-max_tau_samples..max_tau_samples).enumerate() {
            for (fd_idx, fd_bin) in (0..doppler_bins).enumerate() {
                let f_d = -max_doppler_hz +
                    (fd_bin as f64 / doppler_bins as f64) * 2.0 * max_doppler_hz;

                let mut sum = Complex64::new(0.0, 0.0);
                for i in 0..n {
                    let i_delayed = (i as isize - tau).clamp(0, n as isize - 1) as usize;
                    let doppler_phase = 2.0 * PI * f_d * (i as f64 / self.sample_rate_hz);
                    let doppler_shift = Complex64::new(0.0, doppler_phase).exp();

                    sum += self.samples[i] * self.samples[i_delayed].conj() * doppler_shift;
                }

                chi[tau_idx][fd_idx] = sum.norm() / n as f64;
            }
        }

        chi
    }

    /// Matched filter: convolve received signal with time-reversed conjugate
    pub fn matched_filter(&self, received: &[Complex64]) -> Vec<Complex64> {
        let n_out = received.len();
        let n_filt = self.samples.len();
        let mut output = vec![Complex64::new(0.0, 0.0); n_out];

        // Time-reversed conjugate
        let h: Vec<Complex64> = self.samples.iter().rev().map(|s| s.conj()).collect();

        for i in 0..n_out {
            let mut sum = Complex64::new(0.0, 0.0);
            for j in 0..n_filt {
                if i >= j {
                    sum += received[i - j] * h[j];
                }
            }
            output[i] = sum;
        }

        output
    }
}
```

```rust
// processing-core/src/beamforming.rs

use ndarray::{Array1, Array2};
use num_complex::Complex64;
use std::f64::consts::PI;

pub struct PhasedArray {
    pub n_elements: usize,
    pub element_spacing_m: f64,
    pub wavelength_m: f64,
    pub element_weights: Array1<Complex64>,
}

impl PhasedArray {
    /// Create uniform linear array
    pub fn new_ula(
        n_elements: usize,
        element_spacing_wavelengths: f64,
        wavelength_m: f64,
    ) -> Self {
        Self {
            n_elements,
            element_spacing_m: element_spacing_wavelengths * wavelength_m,
            wavelength_m,
            element_weights: Array1::from_elem(n_elements, Complex64::new(1.0, 0.0)),
        }
    }

    /// Apply phase shifts for beam steering to angle theta_deg (azimuth from broadside)
    pub fn steer_to(&mut self, theta_deg: f64) {
        let theta_rad = theta_deg.to_radians();
        let k = 2.0 * PI / self.wavelength_m;

        for n in 0..self.n_elements {
            let phase_shift = -(n as f64) * k * self.element_spacing_m * theta_rad.sin();
            self.element_weights[n] = Complex64::new(0.0, phase_shift).exp();
        }
    }

    /// Apply Taylor window taper for sidelobe control
    pub fn apply_taylor_taper(&mut self, sidelobe_level_db: f64, n_bar: usize) {
        let weights = taylor_window(self.n_elements, sidelobe_level_db, n_bar);
        for (i, w) in weights.iter().enumerate() {
            self.element_weights[i] *= w;
        }
    }

    /// Compute array factor over azimuth angles
    pub fn array_factor(&self, theta_deg_vec: &[f64]) -> Vec<Complex64> {
        let k = 2.0 * PI / self.wavelength_m;

        theta_deg_vec.iter().map(|&theta_deg| {
            let theta_rad = theta_deg.to_radians();
            let mut af = Complex64::new(0.0, 0.0);

            for n in 0..self.n_elements {
                let phase = (n as f64) * k * self.element_spacing_m * theta_rad.sin();
                af += self.element_weights[n] * Complex64::new(0.0, phase).exp();
            }

            af
        }).collect()
    }

    /// Conventional beamformer: weighted sum
    pub fn beamform_conventional(&self, snapshots: &Array2<Complex64>) -> Array1<Complex64> {
        // snapshots: [n_elements × n_snapshots]
        let n_snapshots = snapshots.ncols();
        let mut output = Array1::zeros(n_snapshots);

        for t in 0..n_snapshots {
            let mut sum = Complex64::new(0.0, 0.0);
            for n in 0..self.n_elements {
                sum += self.element_weights[n].conj() * snapshots[[n, t]];
            }
            output[t] = sum;
        }

        output
    }

    /// MVDR (Capon) beamformer
    pub fn beamform_mvdr(
        &self,
        snapshots: &Array2<Complex64>,
        steering_vector: &Array1<Complex64>,
    ) -> Array1<Complex64> {
        use ndarray_linalg::Inverse;

        // Estimate covariance matrix R_xx = (1/N) * X * X^H
        let n_snapshots = snapshots.ncols();
        let mut r_xx = Array2::<Complex64>::zeros((self.n_elements, self.n_elements));

        for t in 0..n_snapshots {
            let x_t = snapshots.column(t);
            for i in 0..self.n_elements {
                for j in 0..self.n_elements {
                    r_xx[[i, j]] += x_t[i] * x_t[j].conj();
                }
            }
        }
        r_xx /= Complex64::new(n_snapshots as f64, 0.0);

        // Add diagonal loading for numerical stability
        for i in 0..self.n_elements {
            r_xx[[i, i]] += Complex64::new(1e-6, 0.0);
        }

        // Compute MVDR weights: w = R_xx^{-1} * a / (a^H * R_xx^{-1} * a)
        let r_inv = r_xx.inv().expect("Covariance matrix inversion failed");
        let r_inv_a = r_inv.dot(steering_vector);

        let mut denom = Complex64::new(0.0, 0.0);
        for i in 0..self.n_elements {
            denom += steering_vector[i].conj() * r_inv_a[i];
        }

        let weights = r_inv_a.mapv(|w| w / denom);

        // Apply weights
        let mut output = Array1::zeros(n_snapshots);
        for t in 0..n_snapshots {
            let mut sum = Complex64::new(0.0, 0.0);
            for n in 0..self.n_elements {
                sum += weights[n].conj() * snapshots[[n, t]];
            }
            output[t] = sum;
        }

        output
    }
}

/// Generate Taylor window weights
fn taylor_window(n: usize, sidelobe_db: f64, n_bar: usize) -> Vec<f64> {
    // Simplified Taylor window (full implementation requires Bessel functions)
    // For now, use Chebyshev as approximation
    chebyshev_window(n, sidelobe_db)
}

fn chebyshev_window(n: usize, sidelobe_db: f64) -> Vec<f64> {
    let r = 10_f64.powf(sidelobe_db / 20.0);
    let mut w = vec![0.0; n];

    for i in 0..n {
        let x = (i as f64 - (n - 1) as f64 / 2.0) / (n as f64 / 2.0);
        w[i] = chebyshev_poly(n - 1, r * x) / chebyshev_poly(n - 1, r);
    }

    // Normalize
    let max_w = w.iter().cloned().fold(f64::NEG_INFINITY, f64::max);
    w.iter_mut().for_each(|x| *x /= max_w);

    w
}

fn chebyshev_poly(n: usize, x: f64) -> f64 {
    if x.abs() <= 1.0 {
        ((n as f64) * x.acos()).cos()
    } else {
        ((n as f64) * x.acosh()).cosh()
    }
}
```

### 3. Range-Doppler Map Generator (Rust)

Performs matched filtering on each pulse and Doppler FFT across pulse train to create 2D range-velocity map.

```rust
// processing-core/src/range_doppler.rs

use num_complex::Complex64;
use rustfft::{FftPlanner, num_complex::Complex};

pub struct RangeDopplerProcessor {
    pub n_range_bins: usize,
    pub n_doppler_bins: usize,
    pub range_resolution_m: f64,
    pub velocity_resolution_mps: f64,
}

impl RangeDopplerProcessor {
    pub fn new(
        bandwidth_hz: f64,
        prf_hz: f64,
        n_pulses: usize,
        wavelength_m: f64,
    ) -> Self {
        let c = 299_792_458.0;  // Speed of light
        let range_resolution_m = c / (2.0 * bandwidth_hz);
        let velocity_resolution_mps = wavelength_m * prf_hz / (2.0 * n_pulses as f64);

        Self {
            n_range_bins: 0,  // Set during processing
            n_doppler_bins: n_pulses,
            range_resolution_m,
            velocity_resolution_mps,
        }
    }

    /// Process pulse train to generate range-Doppler map
    pub fn process(
        &mut self,
        pulse_train: &[Vec<Complex64>],  // [n_pulses][samples_per_pulse]
        reference_waveform: &[Complex64],
    ) -> Vec<Vec<f64>> {
        let n_pulses = pulse_train.len();
        if n_pulses == 0 { return vec![]; }

        // Step 1: Matched filter each pulse (fast-time processing)
        let mut range_profiles: Vec<Vec<Complex64>> = Vec::with_capacity(n_pulses);

        for pulse in pulse_train {
            let filtered = matched_filter(pulse, reference_waveform);
            range_profiles.push(filtered);
        }

        self.n_range_bins = range_profiles[0].len();

        // Step 2: Doppler FFT across pulses (slow-time processing)
        let mut rdm = vec![vec![0.0; self.n_doppler_bins]; self.n_range_bins];
        let mut planner = FftPlanner::new();
        let fft = planner.plan_fft_forward(n_pulses);

        for range_bin in 0..self.n_range_bins {
            // Extract slow-time samples for this range bin
            let mut slow_time: Vec<Complex<f64>> = range_profiles
                .iter()
                .map(|profile| {
                    let c = profile[range_bin];
                    Complex { re: c.re, im: c.im }
                })
                .collect();

            // Apply window to reduce Doppler sidelobes
            apply_hamming_window(&mut slow_time);

            // FFT
            fft.process(&mut slow_time);

            // Store magnitude
            for (doppler_bin, val) in slow_time.iter().enumerate() {
                rdm[range_bin][doppler_bin] = val.norm();
            }
        }

        // FFT shift: move zero Doppler to center
        for range_bin in 0..self.n_range_bins {
            rdm[range_bin].rotate_left(n_pulses / 2);
        }

        rdm
    }

    /// Apply CA-CFAR detector to range-Doppler map
    pub fn detect_cfar(
        &self,
        rdm: &[Vec<f64>],
        pfa: f64,  // Probability of false alarm
        n_guard: usize,
        n_ref: usize,
    ) -> Vec<Detection> {
        let mut detections = Vec::new();
        let alpha = n_ref as f64 * (pfa.powf(-1.0 / n_ref as f64) - 1.0);

        for r in (n_guard + n_ref)..(self.n_range_bins - n_guard - n_ref) {
            for d in (n_guard + n_ref)..(self.n_doppler_bins - n_guard - n_ref) {
                let cut = rdm[r][d];

                // Compute reference noise power (average of reference cells)
                let mut ref_sum = 0.0;
                let mut ref_count = 0;

                for dr in -(n_guard + n_ref) as isize..=(n_guard + n_ref) as isize {
                    for dd in -(n_guard + n_ref) as isize..=(n_guard + n_ref) as isize {
                        if dr.abs() <= n_guard as isize && dd.abs() <= n_guard as isize {
                            continue;  // Skip guard cells
                        }

                        let rr = (r as isize + dr) as usize;
                        let dd_idx = (d as isize + dd) as usize;

                        if rr < self.n_range_bins && dd_idx < self.n_doppler_bins {
                            ref_sum += rdm[rr][dd_idx];
                            ref_count += 1;
                        }
                    }
                }

                let noise_estimate = ref_sum / ref_count as f64;
                let threshold = alpha * noise_estimate;

                if cut > threshold {
                    let range_m = r as f64 * self.range_resolution_m;
                    let velocity_mps = (d as f64 - self.n_doppler_bins as f64 / 2.0)
                        * self.velocity_resolution_mps;
                    let snr_db = 10.0 * (cut / noise_estimate).log10();

                    detections.push(Detection {
                        range_bin: r,
                        doppler_bin: d,
                        range_m,
                        velocity_mps,
                        snr_db,
                        amplitude: cut,
                    });
                }
            }
        }

        detections
    }
}

#[derive(Debug, Clone)]
pub struct Detection {
    pub range_bin: usize,
    pub doppler_bin: usize,
    pub range_m: f64,
    pub velocity_mps: f64,
    pub snr_db: f64,
    pub amplitude: f64,
}

fn matched_filter(signal: &[Complex64], reference: &[Complex64]) -> Vec<Complex64> {
    let n_out = signal.len();
    let n_ref = reference.len();
    let mut output = vec![Complex64::new(0.0, 0.0); n_out];

    // Time-reversed conjugate of reference
    let h: Vec<Complex64> = reference.iter().rev().map(|s| s.conj()).collect();

    for i in 0..n_out {
        let mut sum = Complex64::new(0.0, 0.0);
        for j in 0..n_ref.min(i + 1) {
            sum += signal[i - j] * h[j];
        }
        output[i] = sum;
    }

    output
}

fn apply_hamming_window(data: &mut [Complex<f64>]) {
    let n = data.len();
    for (i, sample) in data.iter_mut().enumerate() {
        let w = 0.54 - 0.46 * (2.0 * PI * i as f64 / (n - 1) as f64).cos();
        sample.re *= w;
        sample.im *= w;
    }
}

use std::f64::consts::PI;
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–5)

**Day 1: Project initialization and Rust backend scaffold**
```bash
cargo init radarforge-api
cd radarforge-api
cargo add axum tokio serde serde_json sqlx uuid chrono tower-http jsonwebtoken bcrypt
cargo add --features postgres,runtime-tokio-rustls,uuid,chrono,json sqlx
cargo add aws-sdk-s3 redis ndarray num-complex rustfft
```
- `src/main.rs` — Axum app with CORS, tracing, graceful shutdown
- `src/config.rs` — Environment-based configuration (DATABASE_URL, JWT_SECRET, S3_BUCKET, REDIS_URL, PROPAGATION_SERVICE_URL)
- `src/state.rs` — AppState struct (PgPool, Redis, S3Client, HTTP client for Python service)
- `src/error.rs` — ApiError enum with IntoResponse
- `Dockerfile` — Multi-stage build (builder + runtime)
- `docker-compose.yml` — PostgreSQL, Redis, MinIO (S3-compatible), Python propagation service

**Day 2: Database schema and migrations**
- `migrations/001_initial.sql` — All 9 tables: users, organizations, org_members, projects, simulations, simulation_jobs, antenna_patterns, waveform_templates, target_scenarios, visualization_configs, usage_records
- `src/db/mod.rs` — Database pool initialization
- `src/db/models.rs` — All SQLx structs with FromRow derives
- Run `sqlx migrate run` to apply schema
- Seed script for initial libraries: 20 antenna patterns (dipole, patch variants), 15 waveform templates (LFM, FMCW configs)

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
- `src/api/handlers/orgs.rs` — Create org, invite member, list members, remove member, ITAR compliance flag management
- `src/api/router.rs` — All route definitions with auth middleware
- Integration tests: auth flow, project CRUD, authorization checks

**Day 5: Python propagation service scaffold**
- `propagation-service/` — FastAPI microservice
- `requirements.txt` — numpy, scipy, fastapi, uvicorn, pydantic
- `propagation_service/main.py` — FastAPI app with health check
- `propagation_service/itu_p676.py` — Atmospheric attenuation (oxygen, water vapor)
- `propagation_service/itu_p838.py` — Rain attenuation
- Endpoints: `/atmospheric_loss`, `/rain_loss`, `/multipath_coefficients`
- Docker container, exposed on internal network to Rust service

### Phase 2 — Signal Processing Core (Days 6–14)

**Day 6: Waveform generation**
- `processing-core/` — New Rust workspace member (shared between native and WASM)
- `processing-core/src/waveform.rs` — Waveform struct with LFM chirp, FMCW sawtooth/triangle, Barker code generators
- `processing-core/src/constants.rs` — Physical constants (c, k_boltzmann, T_0)
- Unit tests: verify chirp bandwidth, FMCW beat frequency, Barker autocorrelation

**Day 7: Ambiguity function computation**
- `processing-core/src/waveform.rs` — `ambiguity_function()` method
- 2D range-Doppler ambiguity surface |χ(τ, f_d)|
- Optimization: FFT-based computation for large delay/Doppler grids
- Tests: LFM chirp ambiguity (diagonal ridge), pulsed waveform ambiguity (thumbtack for short pulses)

**Day 8: Phased array beamforming**
- `processing-core/src/beamforming.rs` — PhasedArray struct for ULA, URA, circular arrays
- Array factor computation, beam steering, Taylor/Chebyshev tapering
- Conventional beamformer (delay-and-sum)
- Tests: ULA pattern nulls at expected angles, grating lobe positions

**Day 9: Advanced beamformers (MVDR)**
- `processing-core/src/beamforming.rs` — MVDR (Capon) beamformer implementation
- Covariance matrix estimation, diagonal loading
- `ndarray-linalg` for matrix inversion
- Tests: MVDR null steering, interference rejection vs conventional beamformer

**Day 10: Matched filtering and pulse compression**
- `processing-core/src/matched_filter.rs` — Matched filter convolution
- Window functions: rectangular, Hamming, Hann, Kaiser, Taylor
- Pulse compression ratio and sidelobe measurement
- Tests: LFM pulse compression gain, sidelobe levels with different windows

**Day 11: Range-Doppler processing**
- `processing-core/src/range_doppler.rs` — RangeDopplerProcessor struct
- Fast-time matched filtering + slow-time Doppler FFT
- RustFFT for efficient FFT computation
- Tests: single target range/Doppler bin extraction, resolution verification

**Day 12: CFAR detection**
- `processing-core/src/cfar.rs` — CA-CFAR, OS-CFAR, GOCA-CFAR detectors
- Guard cell and reference cell configuration
- False alarm rate and detection threshold computation
- Tests: constant false alarm rate across varying noise levels

**Day 13: Radar equation and link budget**
- `processing-core/src/radar_equation.rs` — SNR computation from system parameters
- Free-space propagation loss, atmospheric loss (call Python service), rain loss
- System noise temperature calculation
- Tests: verify SNR matches analytical predictions for known scenarios

**Day 14: Target and clutter models**
- `processing-core/src/targets.rs` — Swerling I-IV RCS models, canonical shape RCS (sphere, cylinder, flat plate)
- `processing-core/src/clutter.rs` — Land clutter (Barrie, Billingsley models), sea clutter (GIT, NRCS)
- Clutter-to-noise ratio calculation
- Tests: Swerling model statistics, clutter power vs grazing angle

### Phase 3 — WASM Build + Frontend Visualization (Days 15–21)

**Day 15: WASM processing compilation**
- `processing-wasm/` — New workspace member for WASM target
- `processing-wasm/Cargo.toml` — wasm-bindgen, serde-wasm-bindgen, js-sys dependencies
- `processing-wasm/src/lib.rs` — WASM entry points: `compute_ambiguity()`, `compute_array_factor()`, `process_range_doppler()`
- `wasm-pack build --target web --release` pipeline
- `wasm-opt -Oz` for size optimization (target <3MB gzipped)
- JavaScript wrapper: `RadarProcessor` class that loads WASM and provides async API

**Day 16: Frontend scaffold**
```bash
npm create vite@latest frontend -- --template react-ts
cd frontend
npm i zustand @tanstack/react-query axios d3 three @react-three/fiber @react-three/drei
```
- `src/App.tsx` — Router, auth context, layout
- `src/stores/projectStore.ts` — Zustand store for project state
- `src/stores/simulationStore.ts` — Simulation configuration and results
- `src/lib/wasmLoader.ts` — WASM module loader with caching
- `src/components/Layout/Navbar.tsx` — Top nav with project selector, user menu
- `src/components/Layout/Sidebar.tsx` — Waveform/antenna/target configuration panels

**Day 17: Antenna array configuration UI**
- `src/components/AntennaConfig/ArrayGeometryEditor.tsx` — Visual editor for ULA/URA/circular arrays
- `src/components/AntennaConfig/ElementSelector.tsx` — Antenna element pattern library browser
- `src/components/AntennaConfig/TaperControls.tsx` — Taper type (uniform, Taylor, Chebyshev) and parameter sliders
- `src/components/AntennaConfig/BeamSteeringControls.tsx` — Azimuth/elevation steering angle inputs
- Canvas preview: 2D top-down view of array geometry

**Day 18: Waveform configuration UI**
- `src/components/WaveformConfig/WaveformTypeSelector.tsx` — Tabs for LFM, FMCW, Barker, etc.
- `src/components/WaveformConfig/LfmControls.tsx` — Center freq, bandwidth, pulse width, PRF sliders
- `src/components/WaveformConfig/FmcwControls.tsx` — Sweep time, modulation type (sawtooth/triangle)
- `src/components/WaveformConfig/AmbiguityPreview.tsx` — Live WASM-computed ambiguity function preview (throttled updates)
- D3.js contour plot for ambiguity function

**Day 19: Target scenario configuration UI**
- `src/components/TargetConfig/TargetList.tsx` — Add/remove targets, configure RCS/range/velocity
- `src/components/TargetConfig/ClutterControls.tsx` — Clutter type (land/sea), model selection, parameters
- `src/components/TargetConfig/EnvironmentControls.tsx` — Temperature, humidity, rain rate, fog
- `src/components/TargetConfig/ScenarioTemplates.tsx` — Pre-built scenarios (automotive, weather, defense)
- 2D scenario visualization: top-down view with target positions

**Day 20: 3D antenna pattern visualization (WebGL)**
- `src/components/Visualization/AntennaPattern3D.tsx` — React Three Fiber 3D radiation pattern
- Spherical coordinate system (theta, phi), color-mapped gain
- Interactive rotation, zoom, measurement cursors
- Pattern cuts: E-plane, H-plane overlays
- Grating lobe and sidelobe annotations

**Day 21: Range-Doppler map visualization**
- `src/components/Visualization/RangeDopplerMap.tsx` — WebGL heatmap renderer
- Color scale: jet, hot, viridis, grayscale
- Interactive pan/zoom, measurement cursors
- CFAR detection overlay (circles/boxes on detected targets)
- Range and velocity axis labels with units
- Export to PNG/CSV

### Phase 4 — API + Job Orchestration (Days 22–28)

**Day 22: Simulation API endpoints**
- `src/api/handlers/simulation.rs` — Create simulation, get simulation, list simulations, cancel simulation, get results
- Config validation: waveform parameters, antenna array feasibility, target scenario
- Element count extraction and WASM/server routing logic
- Plan-based limits enforcement (free: 16 elements, 100 compute minutes/month)

**Day 23: Server-side simulation worker**
- `src/workers/simulation_worker.rs` — Redis job consumer, runs native processing
- Separate CPU and GPU worker pools
- Progress streaming: worker publishes progress → Redis pub/sub → WebSocket to client
- Error handling: numerical errors, timeouts, out-of-memory
- S3 result upload with presigned URL generation for client download

**Day 24: WebSocket for live simulation progress**
- `src/api/ws/mod.rs` — Axum WebSocket upgrade handler
- `src/api/ws/simulation_progress.rs` — Subscribe to simulation progress channel
- Client receives: `{ progress_pct, current_pulse, total_pulses, current_target, eta_seconds }`
- Connection management: heartbeat ping/pong, automatic cleanup on disconnect
- Frontend: `src/hooks/useSimulationProgress.ts` — React hook for WebSocket subscription with reconnection logic

**Day 25: Antenna pattern library API**
- `src/api/handlers/antenna_patterns.rs` — Search patterns (parametric + full-text), get pattern, download pattern data
- S3 integration for pattern file storage (HDF5 format: theta/phi/gain tables)
- Parametric search: filter by element_type, frequency_band, polarization, gain range, beamwidth range
- Pagination with cursor-based scrolling
- Pattern upload endpoint (Defense plan only)

**Day 26: Waveform template library API**
- `src/api/handlers/waveform_templates.rs` — Search waveforms, get template, apply template to project
- Filter by waveform_type, application (automotive, weather, defense), frequency_band
- Template includes precomputed ambiguity function thumbnail
- Custom waveform upload (Defense plan only)

**Day 27: Target scenario library API**
- `src/api/handlers/target_scenarios.rs` — Search scenarios, get scenario, apply scenario to project
- Filter by scenario_type (single_target, multi_target, clutter, jamming, automotive)
- Scenario preview: thumbnail image with target/clutter visualization
- Custom scenario upload (Defense plan only)

**Day 28: Results export and sharing**
- `src/api/handlers/export.rs` — Export range-Doppler map as PNG/HDF5, export detection list as CSV/JSON
- Export antenna pattern as PNG/PDF (3D view snapshot)
- Export ambiguity function as PNG/CSV
- Project sharing: public link generation, fork project, project templates
- Auto-save: frontend debounced PATCH every 5 seconds on configuration changes

### Phase 5 — Advanced Features + Post-Processing (Days 29–33)

**Day 29: Detection curve generation**
- `src/processing/detection.rs` — Pd vs range curves for specified Pfa and target RCS
- Marcum Q-function for Swerling target detection probability
- ROC curve generation (Pd vs Pfa)
- Monte Carlo simulation for detection statistics
- Frontend: `src/components/Visualization/DetectionCurve.tsx` — D3 line plot

**Day 30: Tracking filter implementation**
- `processing-core/src/tracking/kalman.rs` — Extended Kalman Filter (EKF) for range/azimuth/elevation tracking
- `processing-core/src/tracking/alpha_beta.rs` — Alpha-beta filter (simple position/velocity tracker)
- Track initiation logic: M-of-N detection confirmation
- Track coasting: maintain track during missed detections
- Tests: track a maneuvering target, measure RMSE

**Day 31: Data association**
- `processing-core/src/tracking/association.rs` — Nearest neighbor, Global Nearest Neighbor (GNN)
- Gating: Mahalanobis distance threshold
- Multi-target tracking: assign detections to tracks
- Track termination: delete track after N consecutive misses
- Tests: cross-track scenario (two targets crossing paths)

**Day 32: Tracking visualization**
- `src/components/Visualization/TrackDisplay.tsx` — 2D top-down track history plot
- Track IDs, confidence levels, velocity vectors
- Ground truth vs estimated track comparison (for scenarios with known truth)
- Track error metrics: RMSE position, RMSE velocity, track purity, track completeness
- Export tracks as CSV/KML (for Google Earth)

**Day 33: Monte Carlo sensitivity analysis**
- Server-side only (too compute-intensive for WASM)
- `src/workers/monte_carlo_worker.rs` — Sweep parameter (SNR, RCS, clutter power) over range, run simulations
- Generate Pd vs parameter curves, sensitivity heatmaps
- Frontend: `src/components/MonteCarlo/SweepConfig.tsx` — Configure sweep parameter, range, number of trials
- Frontend: `src/components/MonteCarlo/ResultsViewer.tsx` — Plot mean ± std dev curves
- Defense plan feature only

### Phase 6 — Billing + Plan Enforcement (Days 34–37)

**Day 34: Stripe integration**
- `src/api/handlers/billing.rs` — Create checkout session, create customer portal session, get subscription status
- `src/api/handlers/webhooks/stripe.rs` — Handle subscription events: `checkout.session.completed`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.payment_failed`
- Plan mapping:
  - Free: 16-element arrays, 100 compute minutes/month, basic templates
  - Pro ($99/mo): 256-element arrays, 500 compute minutes/month, full libraries, waveform export
  - Defense ($399/user/mo): Unlimited elements, 2000 compute minutes/month, ITAR hosting, Monte Carlo, custom uploads, API access

**Day 35: Usage tracking and plan limits**
- `src/middleware/plan_limits.rs` — Middleware checking plan limits before simulation execution
- `src/services/usage.rs` — Track compute minutes per billing period
- Usage record insertion after each server-side simulation completes (wall_time_ms / 60000)
- Usage dashboard: `src/api/handlers/usage.rs` — Current period usage, historical usage, compute minute breakdown
- Approaching-limit warnings at 80% and 100% of compute minutes

**Day 36: Billing UI**
- `frontend/src/pages/Billing.tsx` — Plan comparison table, current plan display, usage meter
- `frontend/src/components/billing/PlanCard.tsx` — Plan feature comparison cards with upgrade CTAs
- `frontend/src/components/billing/UsageMeter.tsx` — Compute minutes usage bar with threshold indicators
- Upgrade/downgrade flow via Stripe Customer Portal
- Upgrade prompt modals when hitting plan limits (e.g., "Array has 128 elements. Upgrade to Pro for arrays up to 256.")

**Day 37: Feature gating**
- Gate waveform/antenna/scenario export (CSV/PNG/HDF5) behind Pro plan
- Gate custom template upload behind Defense plan
- Gate Monte Carlo analysis behind Defense plan
- Gate API access behind Defense plan
- Gate ITAR-compliant hosting toggle behind Defense plan
- Show locked feature indicators with upgrade CTAs in UI
- Admin endpoint: override plan for specific users (beta testers, partners)

### Phase 7 — Solver Validation + Testing (Days 38–40)

**Day 38: Signal processing validation**
- Validation 1: LFM chirp pulse compression ratio — verify PCR = B · T_p within 1%
- Validation 2: FMCW beat frequency — verify f_beat = (2BR)/(cT) for known range
- Validation 3: ULA beamwidth — verify HPBW = 0.886λ/(Nd) within 5%
- Validation 4: MVDR null depth — verify >30dB null at interferer angle
- Validation 5: Range-Doppler resolution — verify ΔR and Δv match theoretical
- Automated test suite: `processing-core/tests/validation.rs` with tolerance checks

**Day 39: End-to-end integration testing**
- End-to-end test: create project → configure waveform/antenna/targets → run simulation (WASM) → view range-Doppler map → export results
- Server-side test: large array (128 elements) → job enqueued → worker processes → results in S3 → client downloads
- WebSocket test: connect → subscribe → receive progress updates → receive completion
- Concurrent simulation test: submit 10 simultaneous server-side jobs, verify priority queueing
- Auth test: OAuth flow, token refresh, authorization checks

**Day 40: Performance testing and optimization**
- WASM benchmarks: measure time for 16-element, 32-element, 64-element arrays (ambiguity, array factor, range-Doppler)
- Target: 64-element range-Doppler (128 pulses) < 1 second in WASM
- Server benchmarks: measure throughput (simulations/minute) for 256-element, 512-element, 1024-element arrays
- Memory profiling: ensure no leaks in long-running worker processes
- Database query optimization: add missing indexes, optimize N+1 queries
- S3 presigned URL caching to reduce API calls

### Phase 8 — Documentation + Polish (Days 41–42)

**Day 41: Documentation and help system**
- In-app help tooltips on all configuration controls
- `frontend/src/pages/Help.tsx` — Help center with radar fundamentals, waveform guide, antenna theory, CFAR explanation
- Radar equation calculator tool (quick SNR estimation without full simulation)
- Example projects gallery with precomputed results
- API documentation (Swagger/OpenAPI for Defense plan API access)
- Video tutorials: basic LFM simulation, FMCW automotive radar, phased array design

**Day 42: Final polish and deployment**
- Error message improvements: actionable messages, suggestions, links to help
- Loading states, skeleton screens, optimistic UI updates
- Keyboard shortcuts: Ctrl+S save, Ctrl+R run simulation, Esc cancel
- Dark/light theme toggle
- Responsive mobile layout (view-only for tablets, desktop-only for editing)
- Production deployment: AWS infrastructure (ECS Fargate for API/workers, RDS PostgreSQL, ElastiCache Redis, S3, CloudFront CDN)
- GovCloud deployment setup for Defense plan (ITAR compliance)
- Monitoring dashboards: Grafana with simulation latency, queue depth, error rates
- Launch checklist: DNS, SSL, CORS, rate limiting, backup strategy

---

## Validation Benchmarks

### Benchmark 1: LFM Pulse Compression Ratio
**Test:** Generate 1 μs LFM chirp with 100 MHz bandwidth, compress via matched filter.
- **Expected PCR:** 100 (= 1e-6 s × 100e6 Hz)
- **Metric:** Measure peak/sidelobe ratio, verify compressed pulse width = 1/B = 10 ns
- **Pass criteria:** PCR within 1%, sidelobe levels < -13 dB for unweighted, < -40 dB for Taylor weighted

### Benchmark 2: FMCW Range Measurement
**Test:** 77 GHz FMCW sawtooth, B = 1 GHz, T_sweep = 50 μs, target at R = 50 m
- **Expected beat frequency:** f_beat = (2 × 1e9 × 50) / (3e8 × 50e-6) = 6.67 MHz
- **Metric:** Measure FFT peak location
- **Pass criteria:** Frequency within 0.5%, range error < 10 cm

### Benchmark 3: ULA Beamwidth and Sidelobe Level
**Test:** 32-element ULA, λ/2 spacing, uniform weighting
- **Expected HPBW:** 0.886 × λ / (32 × 0.5λ) = 0.0554 rad = 3.17°
- **Expected first sidelobe:** -13.2 dB
- **Metric:** Compute array factor, measure 3dB beamwidth and first sidelobe level
- **Pass criteria:** HPBW within 5%, sidelobe within 1 dB

### Benchmark 4: CA-CFAR False Alarm Rate
**Test:** Generate 1000 range cells of Rayleigh noise, set P_fa = 1e-4, 8 guard + 16 reference cells
- **Expected:** ~0.1 false alarms per trial (average over 100 trials)
- **Metric:** Count detections in noise-only scenario
- **Pass criteria:** Measured P_fa within factor of 2 of target (5e-5 to 2e-4)

### Benchmark 5: Detection Probability (Swerling I Target)
**Test:** SNR = 13 dB, Swerling I target, P_fa = 1e-6, single pulse
- **Expected P_d:** ~0.9 (from Marcum Q-function)
- **Metric:** Monte Carlo with 10,000 trials, count detections
- **Pass criteria:** Measured P_d within ±0.05 of theoretical

---

## Critical Files

```
radarforge/
├── processing-core/                           # Shared processing library (native + WASM)
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── waveform.rs                        # Waveform generation, ambiguity function
│   │   ├── beamforming.rs                     # Array factor, beam steering, MVDR
│   │   ├── matched_filter.rs                  # Pulse compression
│   │   ├── range_doppler.rs                   # RDM generation, CFAR detection
│   │   ├── cfar.rs                            # CA-CFAR, OS-CFAR, GOCA-CFAR
│   │   ├── radar_equation.rs                  # Link budget, SNR computation
│   │   ├── targets.rs                         # Swerling models, RCS
│   │   ├── clutter.rs                         # Land/sea clutter models
│   │   ├── tracking/
│   │   │   ├── mod.rs
│   │   │   ├── kalman.rs                      # EKF, UKF
│   │   │   ├── alpha_beta.rs                  # Alpha-beta filter
│   │   │   └── association.rs                 # NN, GNN data association
│   │   └── constants.rs                       # Physical constants
│   └── tests/
│       ├── validation.rs                      # Validation benchmarks
│       └── integration.rs                     # Integration tests
│
├── processing-wasm/                           # WASM compilation target
│   ├── Cargo.toml
│   └── src/
│       └── lib.rs                             # WASM entry points (wasm-bindgen)
│
├── radarforge-api/                            # Rust API server
│   ├── Cargo.toml
│   ├── src/
│   │   ├── main.rs                            # Axum app setup, router, startup
│   │   ├── config.rs                          # Environment config
│   │   ├── state.rs                           # AppState (PgPool, Redis, S3, HTTP client)
│   │   ├── error.rs                           # ApiError types
│   │   ├── auth/
│   │   │   ├── mod.rs                         # JWT middleware, Claims
│   │   │   └── oauth.rs                       # Google/GitHub OAuth handlers
│   │   ├── api/
│   │   │   ├── router.rs                      # Route definitions
│   │   │   ├── handlers/
│   │   │   │   ├── auth.rs                    # Register, login, OAuth
│   │   │   │   ├── users.rs                   # Profile CRUD
│   │   │   │   ├── projects.rs                # Project CRUD, fork, share
│   │   │   │   ├── simulation.rs              # Create/get/cancel simulation
│   │   │   │   ├── antenna_patterns.rs        # Antenna pattern library
│   │   │   │   ├── waveform_templates.rs      # Waveform template library
│   │   │   │   ├── target_scenarios.rs        # Target scenario library
│   │   │   │   ├── billing.rs                 # Stripe checkout/portal
│   │   │   │   ├── usage.rs                   # Usage tracking
│   │   │   │   ├── export.rs                  # PNG/HDF5/CSV export
│   │   │   │   └── webhooks/
│   │   │   │       └── stripe.rs              # Stripe webhook handler
│   │   │   └── ws/
│   │   │       ├── mod.rs                     # WebSocket upgrade handler
│   │   │       └── simulation_progress.rs     # Live progress streaming
│   │   ├── middleware/
│   │   │   └── plan_limits.rs                 # Plan enforcement middleware
│   │   ├── services/
│   │   │   ├── usage.rs                       # Usage tracking service
│   │   │   └── s3.rs                          # S3 client helpers
│   │   └── workers/
│   │       ├── mod.rs                         # Worker pool management
│   │       ├── simulation_worker.rs           # Server-side simulation execution
│   │       └── monte_carlo_worker.rs          # Monte Carlo sweeps
│   ├── migrations/
│   │   └── 001_initial.sql                    # Full database schema
│   └── tests/
│       ├── api_integration.rs                 # API integration tests
│       └── simulation_e2e.rs                  # End-to-end simulation tests
│
├── frontend/                                  # React frontend
│   ├── package.json
│   ├── vite.config.ts
│   ├── src/
│   │   ├── App.tsx                            # Router, providers, layout
│   │   ├── main.tsx                           # Entry point
│   │   ├── stores/
│   │   │   ├── authStore.ts                   # Auth state (JWT, user)
│   │   │   ├── projectStore.ts                # Project state
│   │   │   └── simulationStore.ts             # Simulation config and results
│   │   ├── hooks/
│   │   │   ├── useSimulationProgress.ts       # WebSocket hook for live progress
│   │   │   └── useWasmProcessor.ts            # WASM processor hook (load + run)
│   │   ├── lib/
│   │   │   ├── api.ts                         # Axios API client
│   │   │   └── wasmLoader.ts                  # WASM bundle loading
│   │   ├── pages/
│   │   │   ├── Dashboard.tsx                  # Project list
│   │   │   ├── Editor.tsx                     # Main editor (config + visualization)
│   │   │   ├── Libraries.tsx                  # Antenna/waveform/scenario libraries
│   │   │   ├── Billing.tsx                    # Plan management
│   │   │   ├── Help.tsx                       # Help center
│   │   │   ├── Login.tsx                      # Auth pages
│   │   │   └── Register.tsx
│   │   ├── components/
│   │   │   ├── AntennaConfig/
│   │   │   │   ├── ArrayGeometryEditor.tsx    # Visual array editor
│   │   │   │   ├── ElementSelector.tsx        # Element pattern library browser
│   │   │   │   ├── TaperControls.tsx          # Taper configuration
│   │   │   │   └── BeamSteeringControls.tsx   # Steering angle inputs
│   │   │   ├── WaveformConfig/
│   │   │   │   ├── WaveformTypeSelector.tsx   # LFM/FMCW/Barker selector
│   │   │   │   ├── LfmControls.tsx            # LFM parameter controls
│   │   │   │   ├── FmcwControls.tsx           # FMCW parameter controls
│   │   │   │   └── AmbiguityPreview.tsx       # Live ambiguity function
│   │   │   ├── TargetConfig/
│   │   │   │   ├── TargetList.tsx             # Target list with RCS/range/velocity
│   │   │   │   ├── ClutterControls.tsx        # Clutter model configuration
│   │   │   │   ├── EnvironmentControls.tsx    # Atmospheric conditions
│   │   │   │   └── ScenarioTemplates.tsx      # Pre-built scenarios
│   │   │   ├── Visualization/
│   │   │   │   ├── AntennaPattern3D.tsx       # WebGL 3D radiation pattern
│   │   │   │   ├── RangeDopplerMap.tsx        # WebGL heatmap
│   │   │   │   ├── DetectionCurve.tsx         # Pd vs range (D3)
│   │   │   │   └── TrackDisplay.tsx           # Track history plot
│   │   │   ├── MonteCarlo/
│   │   │   │   ├── SweepConfig.tsx            # Monte Carlo sweep configuration
│   │   │   │   └── ResultsViewer.tsx          # Sensitivity plots
│   │   │   ├── billing/
│   │   │   │   ├── PlanCard.tsx
│   │   │   │   └── UsageMeter.tsx
│   │   │   └── common/
│   │   │       └── SimulationToolbar.tsx      # Run/stop/export toolbar
│   │   └── data/
│   │       └── templates/                     # Pre-built project templates (JSON)
│   └── public/
│       └── wasm/                              # WASM processor bundle (loaded at runtime)
│
├── propagation-service/                       # Python FastAPI (ITU-R models)
│   ├── requirements.txt
│   ├── main.py                                # FastAPI app
│   ├── itu_p676.py                            # Atmospheric attenuation
│   ├── itu_p838.py                            # Rain attenuation
│   ├── multipath.py                           # Multipath ray tracing
│   └── Dockerfile
│
├── k8s/                                       # Kubernetes manifests
│   ├── api-deployment.yaml
│   ├── worker-deployment-cpu.yaml
│   ├── worker-deployment-gpu.yaml
│   ├── postgres-statefulset.yaml
│   ├── redis-deployment.yaml
│   ├── propagation-service-deployment.yaml
│   └── ingress.yaml
│
├── docker-compose.yml                         # Local development stack
├── Cargo.toml                                 # Workspace root
└── .github/
    └── workflows/
        ├── ci.yml                             # Test + lint on PR
        ├── wasm-build.yml                     # Build + deploy WASM bundle
        └── deploy.yml                         # Build Docker images + deploy to K8s
```

---

## Verification

### Integration Testing Checklist

1. **Auth flow** — Register → login → JWT issued → authorized API calls → token refresh → logout
2. **Project CRUD** — Create → update config → save → reload → verify state preserved
3. **Antenna configuration** — Configure ULA → set steering angle → compute array factor → verify pattern shape
4. **Waveform generation** — Configure LFM chirp → compute ambiguity function (WASM) → verify diagonal ridge
5. **WASM simulation** — 16-element array → range-Doppler map → results returned → visualization displays
6. **Server simulation** — 128-element array → job queued → worker picks up → WebSocket progress → results in S3
7. **Range-Doppler map** — Load result → pan/zoom → CFAR detection overlay → verify detection markers
8. **3D antenna pattern** — Render pattern → rotate/zoom → verify grating lobes → measure beamwidth
9. **Detection curve** — Generate Pd vs range → verify matches theoretical → export CSV
10. **Plan limits** — Free user → 32-element array → blocked with upgrade prompt
11. **Billing** — Subscribe to Pro → Stripe checkout → webhook → plan updated → limits lifted
12. **Concurrent users** — 10 users simultaneously running server simulations → all complete correctly
13. **Template usage** — Select "77 GHz FMCW Automotive" template → project created → simulate → expected RDM
14. **Tracking** — Multi-target scenario → run tracking → visualize tracks → verify track IDs
15. **Error handling** — Invalid configuration → meaningful error message with suggestions → no crash

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

-- 2. Simulation type distribution
SELECT simulation_type, COUNT(*) as count,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 1) as pct
FROM simulations
WHERE created_at >= NOW() - INTERVAL '30 days'
GROUP BY simulation_type ORDER BY count DESC;

-- 3. User plan distribution
SELECT plan, COUNT(*) as users,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 1) as pct
FROM users GROUP BY plan ORDER BY users DESC;

-- 4. Compute usage by user (billing period)
SELECT u.email, u.plan,
    SUM(ur.quantity) as total_minutes,
    CASE u.plan
        WHEN 'free' THEN 100
        WHEN 'pro' THEN 500
        WHEN 'defense' THEN 2000
    END as limit_minutes
FROM usage_records ur
JOIN users u ON ur.user_id = u.id
WHERE ur.period_start <= CURRENT_DATE AND ur.period_end >= CURRENT_DATE
    AND ur.record_type = 'compute_minutes'
GROUP BY u.email, u.plan
ORDER BY total_minutes DESC;

-- 5. Most popular antenna patterns
SELECT ap.name, ap.element_type, ap.frequency_band,
    ap.download_count,
    COUNT(DISTINCT s.project_id) as projects_using
FROM antenna_patterns ap
LEFT JOIN simulations s ON s.config::text ILIKE '%' || ap.id::text || '%'
GROUP BY ap.id
ORDER BY ap.download_count DESC
LIMIT 20;
```

### Performance Benchmarks

| Operation | Target | Method |
|-----------|--------|--------|
| WASM: 16-element ambiguity function | <200ms | Browser benchmark (Chrome DevTools) |
| WASM: 64-element range-Doppler (128 pulses) | <1s | Browser benchmark |
| Server: 256-element range-Doppler (512 pulses) | <10s | Server timing, 4 cores |
| Server: 1024-element array factor | <5s | Server timing, 8 cores |
| WebGL: 3D antenna pattern render | 60 FPS | Chrome FPS counter during rotate |
| WebGL: Range-Doppler heatmap (1024×256) | 60 FPS | Chrome FPS counter during pan/zoom |
| D3: Ambiguity function contour plot | <500ms | React profiler |
| API: create simulation | <200ms | p95 latency (Prometheus) |
| API: search antenna patterns | <100ms | p95 latency with 10K patterns in DB |
| WASM bundle load (cached) | <500ms | Service worker cache hit |
| WASM bundle load (cold) | <3s | CDN delivery, 2.5MB gzipped |
| WebSocket: progress latency | <100ms | Time from worker emit to client receive |

---

## Deployment Architecture

### Infrastructure

```
┌─────────────────────────────────────────────────┐
│                  CloudFront CDN                   │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────┐ │
│  │ Frontend SPA │  │ WASM Bundle │  │ Pattern  │ │
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
│ Multi-AZ     │ │ Cluster mode │ │  Patterns)   │
└──────────────┘ └──────────────┘ └──────────────┘

        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ CPU Worker   │ │ GPU Worker   │ │ Propagation  │
│ Pod ×5 (HPA) │ │ Pod ×2 (p3)  │ │ Service (Py) │
│ c6i.4xlarge  │ │ p3.2xlarge   │ │ Pod ×2       │
└──────────────┘ └──────────────┘ └──────────────┘
```

---

## Post-MVP Roadmap (v1.1+)

### v1.1 — SAR Imaging (Weeks 7-9)
- Range-Doppler algorithm for stripmap SAR
- Back-projection imaging for spotlight SAR
- Autofocus (phase gradient, map drift)
- GPU acceleration for 2D FFT
- Image quality metrics: ISLR, PSLR, resolution
- Supports Pro and Defense plans

### v1.2 — MIMO Radar (Weeks 10-11)
- Virtual array synthesis from MIMO configuration
- Waveform diversity (TDMA, FDMA, CDMA)
- Angle estimation with increased DOF
- Automotive MIMO radar templates (12Tx×16Rx typical)
- Supports all plans

### v1.3 — Polarimetric Radar (Weeks 12-13)
- Dual-polarization (H/V, RHCP/LHCP)
- Polarimetric scattering matrix
- Target classification via polarimetric features
- Weather radar applications: rain/snow/hail discrimination
- Defense plan only

### v1.4 — Jamming and Countermeasures (Weeks 14-15)
- Noise jamming (barrage, spot)
- Deceptive jamming (range/velocity gate pull-off)
- DRFM (Digital RF Memory) repeater
- ECCM techniques: sidelobe blanking, frequency agility
- Defense plan only

### v1.5 — Hardware-in-the-Loop (Weeks 16-18)
- Interface with radar test equipment (signal generators, spectrum analyzers)
- Real-time waveform upload to AWG
- IF/RF data capture and processing
- Calibration and array alignment tools
- Enterprise plan only

### v1.6 — On-Premise / Air-Gapped Deployment (Weeks 19-20)
- Docker Compose deployment for isolated networks
- Offline license validation
- Local S3-compatible storage (MinIO)
- Enterprise plan only
