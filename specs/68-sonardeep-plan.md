# 68. SonarDeep — Underwater Acoustics and Sonar System Simulation

## Implementation Plan

**MVP Scope:** Browser-based ocean environment builder with GEBCO bathymetry and World Ocean Atlas (WOA) sound speed profile import via PostGIS queries, interactive 2D environment visualization (bathymetry and sound-speed profile plots) rendered with D3.js and SVG, custom Gaussian beam ray tracing engine in Rust implementing the Bellhop algorithm with eigenray search compiled to WebAssembly for client-side execution of environments with ≤10,000 rays and server-side Rust-native execution for larger propagation models, support for range-independent ocean environments with flat or sloping bathymetry and depth-dependent sound speed profiles (SSP), frequency-dependent volume attenuation via Thorp's formula, interactive transmission loss (TL) map visualization rendered via WebGL with range-depth heatmap and contour overlays, basic sonar equation calculator with source level (SL), directivity index (DI), transmission loss (TL), noise level (NL), and array gain (AG) inputs producing signal excess (SE) output, active and passive sonar detection probability (Pd) vs. range computation using ROC-based models with user-specified false-alarm probability (Pfa), PostgreSQL + PostGIS for oceanographic database (GEBCO/WOA grids), S3 storage for propagation results, Stripe billing with three tiers (Academic $79/mo / Pro $199/mo / Defense $399/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| Ray Tracing Solver | Rust (native + WASM) | Gaussian beam tracing with eigenray finder via shooting method |
| WASM Compilation | `wasm-pack` + `wasm-bindgen` | Client-side solver for propagation with ≤10K rays |
| Oceanographic Data | Python 3.12 (FastAPI) | GEBCO bathymetry parsing, WOA NetCDF processing, interpolation |
| Database | PostgreSQL 16 + PostGIS 3.4 | Projects, users, simulations, oceanographic grid data |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | Propagation results (transmission loss grids), environment exports |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| Environment Viz | D3.js + SVG | Bathymetry profiles, sound speed profiles, range-dependent plots |
| TL Map Viz | WebGL 2.0 (custom) | GPU-accelerated heatmap rendering for 2D transmission loss |
| Geographic Viz | Mapbox GL JS | Ocean map for environment setup, detection range overlays |
| Real-time | WebSocket (Axum) | Live propagation progress, convergence monitoring |
| Job Queue | Redis 7 + Tokio tasks | Server-side propagation job management, priority queuing |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | WASM bundle delivery, static frontend assets |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, error tracking |
| CI/CD | GitHub Actions | WASM build pipeline, Rust tests, Docker image push |

### Key Architecture Decisions

1. **Hybrid client/server ray tracing with WASM threshold at 10,000 rays**: Propagation runs with ≤10,000 rays (covers typical sonar scenarios: 50 launch angles × 200 range steps) execute entirely in the browser via WASM, providing instant feedback with zero server cost. Larger propagation runs (3D acoustic fields, broadband computations, multi-frequency sweeps) are submitted to the Rust-native server which handles 100K+ ray computations. The threshold is configurable per plan tier.

2. **Custom Gaussian beam ray tracer in Rust rather than wrapping Bellhop**: Building a custom ray tracer in Rust gives us full control over the algorithm, WASM compilation, and GPU offloading for 3D propagation. Bellhop is 30-year-old Fortran code with extensive global state that makes WASM compilation impossible and thread safety problematic. Our Rust solver uses adaptive step-size Runge-Kutta integration for ray ODEs with automatic eigenray detection via binary search on ray launching angles.

3. **PostGIS for oceanographic data rather than flat files**: Storing GEBCO (global bathymetry grid at 15 arc-second resolution) and WOA (World Ocean Atlas sound speed profiles) in PostGIS enables efficient spatial queries for environment extraction. Given latitude/longitude bounds, PostGIS returns interpolated bathymetry and SSP profiles in milliseconds via ST_Intersects and bilinear interpolation. This eliminates the need for clients to download multi-gigabyte NetCDF files and supports range-dependent environments with varying properties along propagation paths.

4. **WebGL transmission loss heatmap separate from environment editor**: The TL map renderer displays 2D transmission loss fields (range-depth grids with 1000×500 cells = 500K pixels) using GPU texture mapping with logarithmic color scales (dB). This is decoupled from the D3.js environment visualization to allow independent pan/zoom and multi-layer overlays (bathymetry contours, sound channels, convergence zones). Data is streamed from WASM/server as Float32Arrays and uploaded to GPU textures.

5. **S3 for propagation results with PostgreSQL metadata catalog**: Full propagation results (TL grids, ray paths, eigenrays, timeseries) are stored in S3 as binary blobs (typically 100KB-10MB each), while PostgreSQL holds searchable metadata (environment ID, frequency, source depth, analysis type). This allows the propagation result library to scale to 100K+ runs per user without bloating the database while enabling fast parametric search via PostgreSQL indexes and full-text search.

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "postgis";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";

-- Users
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email TEXT UNIQUE NOT NULL,
    password_hash TEXT,
    name TEXT NOT NULL,
    avatar_url TEXT,
    auth_provider TEXT DEFAULT 'email',
    auth_provider_id TEXT,
    plan TEXT NOT NULL DEFAULT 'free',
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX users_email_idx ON users(email);
CREATE INDEX users_stripe_idx ON users(stripe_customer_id);

-- Organizations
CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    slug TEXT UNIQUE NOT NULL,
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    plan TEXT NOT NULL DEFAULT 'academic',
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    settings JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX orgs_slug_idx ON organizations(slug);

-- Organization Members
CREATE TABLE org_members (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    org_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role TEXT NOT NULL DEFAULT 'member',
    joined_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(org_id, user_id)
);
CREATE INDEX org_members_org_idx ON org_members(org_id);

-- Ocean Environments
CREATE TABLE environments (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    env_type TEXT NOT NULL,
    lat_min REAL,
    lat_max REAL,
    lon_min REAL,
    lon_max REAL,
    bathymetry_data JSONB NOT NULL,
    ssp_data JSONB NOT NULL,
    bottom_data JSONB NOT NULL DEFAULT '{"type": "fluid", "density": 1.5, "sound_speed": 1600, "attenuation": 0.5}',
    surface_data JSONB NOT NULL DEFAULT '{"type": "flat", "roughness": 0}',
    attenuation_model TEXT NOT NULL DEFAULT 'thorp',
    is_public BOOLEAN NOT NULL DEFAULT false,
    thumbnail_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX envs_owner_idx ON environments(owner_id);

-- GEBCO Bathymetry Grid
CREATE TABLE gebco_bathymetry (
    id SERIAL PRIMARY KEY,
    tile_id TEXT UNIQUE NOT NULL,
    geom GEOMETRY(POLYGON, 4326) NOT NULL,
    resolution_arcsec INTEGER NOT NULL DEFAULT 15,
    data BYTEA NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX gebco_geom_idx ON gebco_bathymetry USING GIST(geom);

-- World Ocean Atlas Sound Speed Profiles
CREATE TABLE woa_sound_speed (
    id SERIAL PRIMARY KEY,
    tile_id TEXT UNIQUE NOT NULL,
    geom GEOMETRY(POLYGON, 4326) NOT NULL,
    season TEXT NOT NULL,
    depths REAL[] NOT NULL,
    data BYTEA NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX woa_geom_idx ON woa_sound_speed USING GIST(geom);

-- Propagation Simulations
CREATE TABLE propagations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    environment_id UUID NOT NULL REFERENCES environments(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name TEXT NOT NULL DEFAULT 'Untitled Propagation',
    source_depth REAL NOT NULL,
    source_lat REAL,
    source_lon REAL,
    frequency REAL NOT NULL,
    method TEXT NOT NULL,
    n_rays INTEGER,
    max_angle REAL,
    max_range REAL,
    output_type TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending',
    execution_mode TEXT NOT NULL DEFAULT 'wasm',
    results_url TEXT,
    results_summary JSONB,
    error_message TEXT,
    ray_count INTEGER,
    computation_time_ms INTEGER,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX props_env_idx ON propagations(environment_id);
CREATE INDEX props_user_idx ON propagations(user_id);
CREATE INDEX props_status_idx ON propagations(status);

-- Server Propagation Jobs
CREATE TABLE propagation_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    propagation_id UUID NOT NULL REFERENCES propagations(id) ON DELETE CASCADE,
    worker_id TEXT,
    cores_allocated INTEGER DEFAULT 1,
    memory_mb INTEGER DEFAULT 2048,
    priority INTEGER DEFAULT 0,
    progress_pct REAL DEFAULT 0.0,
    ray_progress JSONB DEFAULT '{"completed": 0, "total": 0}',
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX jobs_prop_idx ON propagation_jobs(propagation_id);

-- Sonar Performance Scenarios
CREATE TABLE sonar_scenarios (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    propagation_id UUID NOT NULL REFERENCES propagations(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    sonar_type TEXT NOT NULL,
    source_level REAL,
    target_strength REAL,
    target_sl REAL,
    directivity_index REAL NOT NULL DEFAULT 0,
    array_gain REAL NOT NULL DEFAULT 0,
    noise_level REAL NOT NULL,
    detection_threshold REAL NOT NULL DEFAULT 0,
    pfa REAL NOT NULL DEFAULT 0.001,
    signal_excess_db REAL,
    detection_range_km REAL,
    pd_vs_range JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX scenarios_prop_idx ON sonar_scenarios(propagation_id);

-- Usage Tracking
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,
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
pub struct Environment {
    pub id: Uuid,
    pub owner_id: Uuid,
    pub org_id: Option<Uuid>,
    pub name: String,
    pub description: String,
    pub env_type: String,
    pub lat_min: Option<f32>,
    pub lat_max: Option<f32>,
    pub lon_min: Option<f32>,
    pub lon_max: Option<f32>,
    pub bathymetry_data: serde_json::Value,
    pub ssp_data: serde_json::Value,
    pub bottom_data: serde_json::Value,
    pub surface_data: serde_json::Value,
    pub attenuation_model: String,
    pub is_public: bool,
    pub thumbnail_url: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Propagation {
    pub id: Uuid,
    pub environment_id: Uuid,
    pub user_id: Uuid,
    pub name: String,
    pub source_depth: f32,
    pub source_lat: Option<f32>,
    pub source_lon: Option<f32>,
    pub frequency: f32,
    pub method: String,
    pub n_rays: Option<i32>,
    pub max_angle: Option<f32>,
    pub max_range: Option<f32>,
    pub output_type: String,
    pub status: String,
    pub execution_mode: String,
    pub results_url: Option<String>,
    pub results_summary: Option<serde_json::Value>,
    pub error_message: Option<String>,
    pub ray_count: Option<i32>,
    pub computation_time_ms: Option<i32>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct BathymetryData {
    #[serde(rename = "type")]
    pub bathymetry_type: String,
    pub depth: Option<f32>,
    pub slope: Option<f32>,
    pub ranges: Option<Vec<f32>>,
    pub depths: Option<Vec<f32>>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct SoundSpeedProfile {
    #[serde(rename = "type")]
    pub ssp_type: String,
    pub c0: Option<f32>,
    pub gradient: Option<f32>,
    pub channel_axis: Option<f32>,
    pub depths: Option<Vec<f32>>,
    pub speeds: Option<Vec<f32>>,
}

#[derive(Debug, Deserialize)]
pub struct PropagationParams {
    pub source_depth: f32,
    pub frequency: f32,
    pub method: PropagationMethod,
    pub n_rays: Option<u32>,
    pub max_angle: Option<f32>,
    pub max_range: Option<f32>,
    pub output_type: OutputType,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum PropagationMethod {
    RayTracing,
    NormalModes,
    ParabolicEquation,
    Wavenumber,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum OutputType {
    TransmissionLoss,
    RayPaths,
    Eigenrays,
    ArrivalStructure,
}
```

---

## Solver Architecture Deep-Dive

### Governing Equations and Discretization

SonarDeep's core solver implements **Gaussian beam ray tracing**, the standard method used by Bellhop and other underwater acoustic models. The acoustic wave equation in a stratified ocean reduces to ray tracing when wavelengths are small compared to environmental scale lengths.

**Ray Equations**: Rays follow Snell's law in a medium with depth-dependent sound speed c(z):

```
dr/ds = ξ / c(r,z)
dz/ds = ζ / c(r,z)
dξ/ds = -∂c/∂r / c
dζ/ds = -∂c/∂z / c

where:
- (r, z) = ray position (range, depth)
- (ξ, ζ) = slowness components (ray parameters)
- s = arc length along ray path
- c(r,z) = sound speed at (r,z)
```

For range-independent environments (MVP scope), ∂c/∂r = 0, so ξ is constant along each ray (Snell's law):

```
c(z) · sin(θ(z)) = c₀ · sin(θ₀) = constant

where θ is ray angle from horizontal
```

**Gaussian Beam Amplitude**: Ray amplitude follows transport equation with geometric spreading and volume attenuation:

```
A(s) = A₀ · sqrt(q₀/(q(s) · c(s)/c₀)) · exp(-∫₀ˢ α(s') ds')

where:
- q(s) = ray tube cross-sectional area (geometric spreading)
- α(s) = volume attenuation coefficient (Np/m)
- A₀ = source amplitude
```

Volume attenuation α in dB/km uses **Thorp's formula** (MVP):

```
α_dB/km = (0.11 · f²)/(1 + f²) + (44 · f²)/(4100 + f²) + 2.75e-4 · f² + 0.003

where f is frequency in kHz
```

**Eigenray Search**: Finding eigenrays uses **binary search on launch angles**. For source depth z_s and receiver at (r_rcv, z_rcv):

1. Trace rays at angles θ₁ and θ₂ to range r_rcv
2. Record depths z₁ = z(r_rcv; θ₁) and z₂ = z(r_rcv; θ₂)
3. If sign(z₁ - z_rcv) ≠ sign(z₂ - z_rcv), eigenray exists in [θ₁, θ₂]
4. Bisect: θ_mid = (θ₁ + θ₂)/2, trace to r_rcv, repeat until |z_mid - z_rcv| < ε

**Transmission Loss**: TL in dB computed from ray amplitudes at receiver:

```
TL = -20 · log₁₀(|Σ_rays A_i|)

where the sum is coherent (phase-preserving) for eigenrays
```

### Client/Server Split (WASM Threshold)

```
Propagation request → Ray count estimation
    │
    ├── ≤10,000 rays → WASM solver (browser)
    │   ├── Instant startup, no network latency
    │   ├── Results displayed immediately
    │   └── No server cost
    │
    └── >10,000 rays → Server solver (Rust native)
        ├── Job queued via Redis (priority by plan tier)
        ├── Worker picks up, runs native solver
        ├── Progress streamed via WebSocket
        └── Results stored in S3, URL returned
```

The 10,000-ray threshold was chosen because:
- WASM solver handles 10K rays (50 angles × 200 range steps) in <3 seconds on modern hardware
- 10K rays covers: typical sonar scenarios (20-100 launch angles), shallow water (≤1 km depth), medium ranges (≤50 km)
- Above 10K rays: deep ocean (5 km depth), long ranges (100+ km), 3D propagation → need server compute

---

## Architecture Deep-Dives

### 1. Propagation API Handler (Rust/Axum)

```rust
// src/api/handlers/propagation.rs

use axum::{extract::{Path, State, Json}, http::StatusCode, response::IntoResponse};
use uuid::Uuid;
use crate::{db::models::{Propagation, PropagationParams}, state::AppState, auth::Claims, error::ApiError};

#[derive(serde::Deserialize)]
pub struct CreatePropagationRequest {
    pub source_depth: f32,
    pub frequency: f32,
    pub method: PropagationMethod,
    pub n_rays: Option<u32>,
    pub max_angle: Option<f32>,
    pub max_range: Option<f32>,
    pub output_type: String,
}

pub async fn create_propagation(
    State(state): State<AppState>,
    claims: Claims,
    Path(environment_id): Path<Uuid>,
    Json(req): Json<CreatePropagationRequest>,
) -> Result<impl IntoResponse, ApiError> {
    // 1. Verify environment ownership
    let environment = sqlx::query_as!(
        crate::db::models::Environment,
        "SELECT * FROM environments WHERE id = $1 AND (owner_id = $2 OR org_id IN (
            SELECT org_id FROM org_members WHERE user_id = $2
        ))",
        environment_id, claims.user_id
    )
    .fetch_optional(&state.db).await?
    .ok_or(ApiError::NotFound("Environment not found"))?;

    // 2. Estimate ray count
    let n_rays = req.n_rays.unwrap_or(50);
    let max_range = req.max_range.unwrap_or(50.0);
    let range_steps = (max_range * 20.0) as u32;
    let estimated_ray_count = n_rays * range_steps;

    // 3. Check plan limits
    let user = sqlx::query_as!(crate::db::models::User, "SELECT * FROM users WHERE id = $1", claims.user_id)
        .fetch_one(&state.db).await?;

    if user.plan == "free" {
        return Err(ApiError::PlanLimit("Free plan does not support propagation. Upgrade to Academic for access."));
    }

    if user.plan == "academic" && estimated_ray_count > 50_000 {
        return Err(ApiError::PlanLimit("Academic plan supports up to 50,000 rays. Upgrade to Pro for unlimited."));
    }

    // 4. Determine execution mode
    let execution_mode = if estimated_ray_count <= 10_000 { "wasm" } else { "server" };

    // 5. Create propagation record
    let prop = sqlx::query_as!(
        Propagation,
        r#"INSERT INTO propagations (environment_id, user_id, name, source_depth, frequency, method,
             n_rays, max_angle, max_range, output_type, status, execution_mode, ray_count)
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11, $12, $13) RETURNING *"#,
        environment_id, claims.user_id, format!("Propagation at {} Hz", req.frequency),
        req.source_depth, req.frequency, serde_json::to_string(&req.method)?,
        n_rays as i32, req.max_angle.unwrap_or(80.0), max_range, req.output_type,
        if execution_mode == "wasm" { "completed" } else { "pending" },
        execution_mode, estimated_ray_count as i32,
    )
    .fetch_one(&state.db).await?;

    // 6. For server execution, enqueue job
    if execution_mode == "server" {
        let job = sqlx::query_as!(
            crate::db::models::PropagationJob,
            r#"INSERT INTO propagation_jobs (propagation_id, priority) VALUES ($1, $2) RETURNING *"#,
            prop.id,
            match user.plan.as_str() { "defense" => 20, "pro" => 10, _ => 0 },
        )
        .fetch_one(&state.db).await?;

        let mut conn = state.redis.get_async_connection().await?;
        redis::cmd("ZADD").arg("propagation:jobs").arg(job.priority).arg(job.id.to_string())
            .query_async::<_, ()>(&mut conn).await?;
    }

    Ok((StatusCode::CREATED, Json(prop)))
}
```

### 2. Ray Tracing Solver Core (Rust — shared between WASM and native)

```rust
// solver-core/src/ray_tracing.rs

use nalgebra::{Vector2, Vector4};
use crate::environment::Environment;

pub struct RayTracerConfig {
    pub source_depth: f32,
    pub frequency: f32,
    pub n_rays: u32,
    pub max_angle: f32,
    pub max_range: f32,
    pub range_step: f32,
}

pub struct Ray {
    pub launch_angle: f32,
    pub positions: Vec<Vector2<f32>>,
    pub amplitudes: Vec<f32>,
    pub arc_length: Vec<f32>,
    pub n_bounces: u32,
    pub surface_bounces: u32,
    pub bottom_bounces: u32,
}

pub struct RayTracer {
    config: RayTracerConfig,
    env: Environment,
}

impl RayTracer {
    pub fn new(config: RayTracerConfig, env: Environment) -> Self {
        Self { config, env }
    }

    pub fn trace_rays(&self) -> Vec<Ray> {
        let angle_step = (2.0 * self.config.max_angle) / (self.config.n_rays as f32 - 1.0);
        let mut rays = Vec::with_capacity(self.config.n_rays as usize);

        for i in 0..self.config.n_rays {
            let angle = -self.config.max_angle + (i as f32) * angle_step;
            let ray = self.trace_single_ray(angle);
            rays.push(ray);
        }
        rays
    }

    fn trace_single_ray(&self, launch_angle_deg: f32) -> Ray {
        let launch_angle = launch_angle_deg.to_radians();
        let c0 = self.env.sound_speed_at_depth(self.config.source_depth);
        let p = launch_angle.cos() / c0;

        let mut state = Vector4::new(0.0, self.config.source_depth, p * c0, launch_angle.sin());
        let mut positions = vec![Vector2::new(state[0] / 1000.0, state[1])];
        let mut amplitudes = vec![1.0];
        let mut arc_length = vec![0.0];

        let ds = 1.0;
        let mut s = 0.0;
        let mut surface_bounces = 0;
        let mut bottom_bounces = 0;

        while state[0] / 1000.0 < self.config.max_range {
            state = self.rk4_step(state, p, ds);
            s += ds;

            let range_km = state[0] / 1000.0;
            let depth = state[1];
            let bottom_depth = self.env.bathymetry_at_range(range_km);

            if depth <= 0.0 {
                state[1] = -depth;
                state[3] = -state[3];
                surface_bounces += 1;
                amplitudes.push(amplitudes.last().unwrap() * 0.9);
            }

            if depth >= bottom_depth {
                state[1] = 2.0 * bottom_depth - depth;
                state[3] = -state[3];
                bottom_bounces += 1;
                let reflection_coef = self.env.bottom_reflection_coefficient(state[3].atan2(state[2]).abs());
                amplitudes.push(amplitudes.last().unwrap() * reflection_coef);
            } else {
                let c = self.env.sound_speed_at_depth(state[1]);
                let spreading = (s / (s - ds)).sqrt();
                let alpha = self.env.volume_attenuation(self.config.frequency, state[1]);
                let atten = (-alpha * ds).exp();
                amplitudes.push(amplitudes.last().unwrap() * spreading * atten);
            }

            positions.push(Vector2::new(range_km, depth));
            arc_length.push(s);
        }

        Ray { launch_angle: launch_angle_deg, positions, amplitudes, arc_length,
              n_bounces: surface_bounces + bottom_bounces, surface_bounces, bottom_bounces }
    }

    fn rk4_step(&self, state: Vector4<f32>, p: f32, ds: f32) -> Vector4<f32> {
        let k1 = self.ray_derivatives(state, p);
        let k2 = self.ray_derivatives(state + k1 * (ds / 2.0), p);
        let k3 = self.ray_derivatives(state + k2 * (ds / 2.0), p);
        let k4 = self.ray_derivatives(state + k3 * ds, p);
        state + (k1 + k2 * 2.0 + k3 * 2.0 + k4) * (ds / 6.0)
    }

    fn ray_derivatives(&self, state: Vector4<f32>, p: f32) -> Vector4<f32> {
        let z = state[1];
        let c = self.env.sound_speed_at_depth(z);
        let dz_ds = (1.0 / (c * c) - p * p).max(0.0).sqrt();

        if state[3] < 0.0 {
            Vector4::new(p * c, -dz_ds * c, 0.0, -self.env.sound_speed_gradient(z) * p / dz_ds)
        } else {
            Vector4::new(p * c, dz_ds * c, 0.0, self.env.sound_speed_gradient(z) * p / dz_ds)
        }
    }

    pub fn compute_transmission_loss(&self, rays: &[Ray]) -> TransmissionLossGrid {
        let n_range = (self.config.max_range / self.config.range_step) as usize;
        let max_depth = self.env.max_depth();
        let n_depth = (max_depth / 10.0) as usize;

        let range_km: Vec<f32> = (0..n_range).map(|i| i as f32 * self.config.range_step).collect();
        let depth_m: Vec<f32> = (0..n_depth).map(|i| i as f32 * 10.0).collect();
        let mut pressure_field = vec![vec![0.0f32; n_range]; n_depth];

        for ray in rays {
            for (pos, amp) in ray.positions.iter().zip(&ray.amplitudes) {
                let r_idx = (pos[0] / self.config.range_step).round() as usize;
                let z_idx = (pos[1] / 10.0).round() as usize;
                if r_idx < n_range && z_idx < n_depth {
                    pressure_field[z_idx][r_idx] += amp;
                }
            }
        }

        let mut tl_db = vec![vec![0.0; n_range]; n_depth];
        for z_idx in 0..n_depth {
            for r_idx in 0..n_range {
                let p = pressure_field[z_idx][r_idx];
                tl_db[z_idx][r_idx] = if p > 1e-10 { -20.0 * p.log10() } else { 200.0 };
            }
        }

        TransmissionLossGrid { range_km, depth_m, tl_db }
    }
}

pub struct TransmissionLossGrid {
    pub range_km: Vec<f32>,
    pub depth_m: Vec<f32>,
    pub tl_db: Vec<Vec<f32>>,
}
```

### 3. Transmission Loss Visualization (React + WebGL)

```typescript
// frontend/src/components/TransmissionLossViewer.tsx

import { useRef, useEffect, useCallback, useState } from 'react';
import { usePropagationStore } from '../stores/propagationStore';

interface TLData {
  rangeKm: Float32Array;
  depthM: Float32Array;
  tlDb: Float32Array;
  nRange: number;
  nDepth: number;
}

export function TransmissionLossViewer() {
  const canvasRef = useRef<HTMLCanvasElement>(null);
  const glRef = useRef<WebGL2RenderingContext | null>(null);
  const programRef = useRef<WebGLProgram | null>(null);
  const textureRef = useRef<WebGLTexture | null>(null);
  const { tlData, colorScale } = usePropagationStore();
  const [hoveredPoint, setHoveredPoint] = useState<{ r: number; z: number; tl: number } | null>(null);

  useEffect(() => {
    const canvas = canvasRef.current;
    if (!canvas) return;

    const gl = canvas.getContext('webgl2', { antialias: false, alpha: false });
    if (!gl) return;
    glRef.current = gl;

    const vsSource = `#version 300 es
      in vec2 a_position;
      in vec2 a_texCoord;
      out vec2 v_texCoord;
      void main() {
        gl_Position = vec4(a_position, 0.0, 1.0);
        v_texCoord = a_texCoord;
      }
    `;

    const fsSource = `#version 300 es
      precision highp float;
      in vec2 v_texCoord;
      out vec4 fragColor;
      uniform sampler2D u_tlTexture;
      uniform float u_tlMin;
      uniform float u_tlMax;

      vec3 jet(float t) {
        t = clamp(t, 0.0, 1.0);
        float r = clamp(1.5 - abs(4.0 * t - 3.0), 0.0, 1.0);
        float g = clamp(1.5 - abs(4.0 * t - 2.0), 0.0, 1.0);
        float b = clamp(1.5 - abs(4.0 * t - 1.0), 0.0, 1.0);
        return vec3(r, g, b);
      }

      void main() {
        float tlDb = texture(u_tlTexture, v_texCoord).r;
        float t = (tlDb - u_tlMin) / (u_tlMax - u_tlMin);
        vec3 color = jet(1.0 - t);
        fragColor = vec4(color, 1.0);
      }
    `;

    const program = createShaderProgram(gl, vsSource, fsSource);
    programRef.current = program;

    const positions = new Float32Array([-1, -1,  0, 1,  1, -1,  1, 1, -1,  1,  0, 0,  1,  1,  1, 0]);
    const buffer = gl.createBuffer();
    gl.bindBuffer(gl.ARRAY_BUFFER, buffer);
    gl.bufferData(gl.ARRAY_BUFFER, positions, gl.STATIC_DRAW);

    const posLoc = gl.getAttribLocation(program, 'a_position');
    const texLoc = gl.getAttribLocation(program, 'a_texCoord');
    gl.enableVertexAttribArray(posLoc);
    gl.vertexAttribPointer(posLoc, 2, gl.FLOAT, false, 16, 0);
    gl.enableVertexAttribArray(texLoc);
    gl.vertexAttribPointer(texLoc, 2, gl.FLOAT, false, 16, 8);

    const texture = gl.createTexture();
    textureRef.current = texture;
    gl.bindTexture(gl.TEXTURE_2D, texture);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.LINEAR);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.LINEAR);

    return () => {
      gl.deleteProgram(program);
      gl.deleteTexture(texture);
      gl.deleteBuffer(buffer);
    };
  }, []);

  useEffect(() => {
    const gl = glRef.current;
    const texture = textureRef.current;
    if (!gl || !texture || !tlData) return;

    gl.bindTexture(gl.TEXTURE_2D, texture);
    gl.texImage2D(gl.TEXTURE_2D, 0, gl.R32F, tlData.nRange, tlData.nDepth, 0, gl.RED, gl.FLOAT, tlData.tlDb);
  }, [tlData]);

  const render = useCallback(() => {
    const gl = glRef.current;
    const program = programRef.current;
    if (!gl || !program || !tlData) return;

    gl.viewport(0, 0, gl.canvas.width, gl.canvas.height);
    gl.clearColor(0.1, 0.1, 0.15, 1.0);
    gl.clear(gl.COLOR_BUFFER_BIT);
    gl.useProgram(program);

    const tlMinLoc = gl.getUniformLocation(program, 'u_tlMin');
    const tlMaxLoc = gl.getUniformLocation(program, 'u_tlMax');
    gl.uniform1f(tlMinLoc, colorScale.min);
    gl.uniform1f(tlMaxLoc, colorScale.max);

    gl.drawArrays(gl.TRIANGLE_STRIP, 0, 4);
  }, [tlData, colorScale]);

  useEffect(() => {
    const animId = requestAnimationFrame(render);
    return () => cancelAnimationFrame(animId);
  }, [render]);

  const handleMouseMove = useCallback((e: React.MouseEvent) => {
    const canvas = canvasRef.current;
    if (!canvas || !tlData) return;

    const rect = canvas.getBoundingClientRect();
    const x = (e.clientX - rect.left) / rect.width;
    const y = (e.clientY - rect.top) / rect.height;

    const rIdx = Math.floor(x * tlData.nRange);
    const zIdx = Math.floor(y * tlData.nDepth);

    if (rIdx >= 0 && rIdx < tlData.nRange && zIdx >= 0 && zIdx < tlData.nDepth) {
      const tl = tlData.tlDb[zIdx * tlData.nRange + rIdx];
      setHoveredPoint({ r: tlData.rangeKm[rIdx], z: tlData.depthM[zIdx], tl });
    }
  }, [tlData]);

  return (
    <div className="tl-viewer flex flex-col h-full bg-gray-900">
      <div className="flex items-center justify-between px-4 py-2 bg-gray-800">
        <h3 className="text-white font-semibold">Transmission Loss</h3>
      </div>
      <div className="flex-1 relative">
        <canvas ref={canvasRef} className="w-full h-full"
          onMouseMove={handleMouseMove} onMouseLeave={() => setHoveredPoint(null)} />
        {hoveredPoint && (
          <div className="absolute top-4 right-4 bg-gray-800 text-white text-sm px-3 py-2 rounded shadow-lg">
            <div>Range: {hoveredPoint.r.toFixed(1)} km</div>
            <div>Depth: {hoveredPoint.z.toFixed(0)} m</div>
            <div>TL: {hoveredPoint.tl.toFixed(1)} dB</div>
          </div>
        )}
      </div>
    </div>
  );
}

function createShaderProgram(gl: WebGL2RenderingContext, vsSource: string, fsSource: string): WebGLProgram {
  const vs = gl.createShader(gl.VERTEX_SHADER)!;
  gl.shaderSource(vs, vsSource);
  gl.compileShader(vs);

  const fs = gl.createShader(gl.FRAGMENT_SHADER)!;
  gl.shaderSource(fs, fsSource);
  gl.compileShader(fs);

  const program = gl.createProgram()!;
  gl.attachShader(program, vs);
  gl.attachShader(program, fs);
  gl.linkProgram(program);

  gl.deleteShader(vs);
  gl.deleteShader(fs);

  return program;
}
```

### 4. Oceanographic Data Service (Python FastAPI)

```python
# ocean-data-service/main.py

from fastapi import FastAPI, Query
from pydantic import BaseModel
import psycopg2
from psycopg2.extras import RealDictCursor
import numpy as np
from typing import List, Optional

app = FastAPI()

class BathymetryProfile(BaseModel):
    ranges_km: List[float]
    depths_m: List[float]

class SoundSpeedProfile(BaseModel):
    depths_m: List[float]
    speeds_mps: List[float]

@app.get("/bathymetry")
async def get_bathymetry(
    lat_min: float = Query(...),
    lat_max: float = Query(...),
    lon_min: float = Query(...),
    lon_max: float = Query(...),
    resolution_m: int = Query(1000)
) -> BathymetryProfile:
    """Fetch GEBCO bathymetry from PostGIS database."""
    conn = psycopg2.connect("postgresql://postgres:password@db:5432/sonardeep")
    cur = conn.cursor(cursor_factory=RealDictCursor)

    # Query GEBCO tiles that intersect bounding box
    cur.execute("""
        SELECT tile_id, data FROM gebco_bathymetry
        WHERE ST_Intersects(geom, ST_MakeEnvelope(%s, %s, %s, %s, 4326))
    """, (lon_min, lat_min, lon_max, lat_max))

    tiles = cur.fetchall()
    if not tiles:
        return BathymetryProfile(ranges_km=[], depths_m=[])

    # Interpolate bathymetry along line (simplified: assume east-west transect)
    ranges_km = np.arange(0, haversine_distance(lat_min, lon_min, lat_max, lon_max), resolution_m / 1000.0)
    depths_m = []

    for r in ranges_km:
        lat = lat_min + (lat_max - lat_min) * (r / ranges_km[-1])
        lon = lon_min + (lon_max - lon_min) * (r / ranges_km[-1])

        cur.execute("""
            SELECT ST_Value(ST_Union(raster), ST_SetSRID(ST_Point(%s, %s), 4326)) as depth
            FROM gebco_bathymetry
            WHERE ST_Intersects(geom, ST_SetSRID(ST_Point(%s, %s), 4326))
        """, (lon, lat, lon, lat))
        result = cur.fetchone()
        depth = result['depth'] if result else 1000.0
        depths_m.append(-depth)  # GEBCO stores negative for below sea level

    cur.close()
    conn.close()

    return BathymetryProfile(ranges_km=ranges_km.tolist(), depths_m=depths_m)

@app.get("/sound-speed")
async def get_sound_speed(
    lat: float = Query(...),
    lon: float = Query(...),
    season: str = Query("annual")
) -> SoundSpeedProfile:
    """Fetch WOA sound speed profile from PostGIS database."""
    conn = psycopg2.connect("postgresql://postgres:password@db:5432/sonardeep")
    cur = conn.cursor(cursor_factory=RealDictCursor)

    cur.execute("""
        SELECT depths, data FROM woa_sound_speed
        WHERE ST_Intersects(geom, ST_SetSRID(ST_Point(%s, %s), 4326)) AND season = %s
        LIMIT 1
    """, (lon, lat, season))

    result = cur.fetchone()
    if not result:
        # Return default profile if no data
        return SoundSpeedProfile(depths_m=[0, 1000, 5000], speeds_mps=[1500, 1480, 1520])

    depths_m = result['depths']
    # Decompress and parse sound speed data (stored as compressed binary)
    speeds_mps = parse_woa_data(result['data'])

    cur.close()
    conn.close()

    return SoundSpeedProfile(depths_m=depths_m, speeds_mps=speeds_mps)

def haversine_distance(lat1, lon1, lat2, lon2):
    """Compute great-circle distance in km."""
    R = 6371.0
    phi1, phi2 = np.radians(lat1), np.radians(lat2)
    dphi = np.radians(lat2 - lat1)
    dlambda = np.radians(lon2 - lon1)
    a = np.sin(dphi/2)**2 + np.cos(phi1) * np.cos(phi2) * np.sin(dlambda/2)**2
    return 2 * R * np.arcsin(np.sqrt(a))

def parse_woa_data(data_bytes):
    """Parse compressed WOA sound speed data."""
    import zlib
    decompressed = zlib.decompress(data_bytes)
    speeds = np.frombuffer(decompressed, dtype=np.float32)
    return speeds.tolist()
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–4)

**Day 1: Project initialization and Rust backend scaffold**
- `cargo init sonardeep-api && cd sonardeep-api`
- `cargo add axum tokio serde serde_json sqlx uuid chrono tower-http jsonwebtoken bcrypt redis aws-sdk-s3`
- `src/main.rs` — Axum app with CORS, tracing, graceful shutdown
- `src/config.rs` — Environment-based configuration (DATABASE_URL, JWT_SECRET, S3_BUCKET, REDIS_URL)
- `src/state.rs` — AppState struct (PgPool, Redis, S3Client)
- `src/error.rs` — ApiError enum with IntoResponse
- `Dockerfile` — Multi-stage build (builder + runtime)
- `docker-compose.yml` — PostgreSQL + PostGIS, Redis, MinIO (S3-compatible), Python service

**Day 2: Database schema and migrations**
- `migrations/001_initial.sql` — All 9 tables: users, organizations, org_members, environments, gebco_bathymetry, woa_sound_speed, propagations, propagation_jobs, sonar_scenarios, usage_records
- `src/db/mod.rs` — Database pool initialization with PostGIS extension
- `src/db/models.rs` — All SQLx structs with FromRow derives
- Run `sqlx migrate run` to apply schema
- Seed script for sample environments and GEBCO/WOA tiles

**Day 3: Authentication system**
- `src/auth/mod.rs` — JWT token generation and validation middleware
- `src/auth/oauth.rs` — Google and GitHub OAuth 2.0 flow handlers
- `src/api/handlers/auth.rs` — Register, login, OAuth callback, refresh token, me endpoints
- Password hashing with bcrypt (cost factor 12)
- JWT with 24h access token + 30d refresh token
- Auth middleware that extracts Claims from Authorization header

**Day 4: User and environment CRUD**
- `src/api/handlers/users.rs` — Get profile, update profile, delete account
- `src/api/handlers/environments.rs` — Create, list, get, update, delete, fork environment
- `src/api/handlers/orgs.rs` — Create org, invite member, list members, remove member
- `src/api/router.rs` — All route definitions with auth middleware
- Integration tests: auth flow, environment CRUD, authorization checks

### Phase 2 — Solver Core — Ray Tracing (Days 5–11)

**Day 5: Environment modeling framework**
- `solver-core/` — New Rust workspace member (shared between native and WASM)
- `solver-core/src/environment.rs` — Environment struct with bathymetry, SSP, bottom properties
- `solver-core/src/sound_speed.rs` — SSP models: isovelocity, linear gradient, Munk profile, custom
- `solver-core/src/attenuation.rs` — Thorp's formula for frequency-dependent volume attenuation
- Unit tests: SSP evaluation at various depths, attenuation vs. frequency curves

**Day 6: Ray equation integration**
- `solver-core/src/ray_tracing.rs` — Ray struct, RayTracer struct
- `solver-core/src/integration.rs` — RK4 integrator for ray ODEs
- Ray derivatives for range-independent environments (Snell's law)
- Tests: simple straight-line ray (isovelocity), downward-refracting ray (positive gradient)

**Day 7: Boundary interactions**
- Surface reflection with loss coefficient
- Bottom reflection with angle-dependent coefficient (fluid bottom model)
- Ray path clipping at boundaries
- Tests: ray with surface bounce, ray with bottom bounce, multiple bounces

**Day 8: Transmission loss computation**
- `solver-core/src/transmission_loss.rs` — TL grid computation from ray field
- Gaussian beam spreading model
- Coherent pressure summation on grid
- Conversion to dB: TL = -20·log10(p)
- Tests: TL at known distances for simple cases

**Day 9: Eigenray search**
- `solver-core/src/eigenray.rs` — Binary search on launch angles
- Eigenray bracket detection via sign change
- Iterative refinement to depth tolerance
- Tests: find direct path, find surface-reflected path, find SOFAR channel eigenrays

**Day 10: Performance optimization**
- Parallel ray tracing via Rayon
- Adaptive range step based on sound speed gradient
- Early termination for rays that escape environment
- Benchmark: 10K rays in <3s, 100K rays in <30s

**Day 11: WASM compilation**
- `solver-wasm/` — New workspace member for WASM target
- `solver-wasm/Cargo.toml` — wasm-bindgen, serde-wasm-bindgen dependencies
- `solver-wasm/src/lib.rs` — WASM entry points: `trace_rays()`, `compute_transmission_loss()`, `find_eigenrays()`
- `wasm-pack build --target web --release` pipeline
- `wasm-opt -Oz` for size optimization (target <1MB gzipped)
- JavaScript wrapper: `SonarSolver` class that loads WASM and provides async API

### Phase 3 — Frontend + Environment Builder (Days 12–17)

**Day 12: Frontend scaffold and stores**
- `npm create vite@latest frontend -- --template react-ts`
- `npm i zustand @tanstack/react-query axios mapbox-gl d3`
- `src/App.tsx` — Router, auth context, layout
- `src/stores/authStore.ts` — Auth state (JWT, user)
- `src/stores/environmentStore.ts` — Environment state (bathymetry, SSP, bounds)
- `src/stores/propagationStore.ts` — Propagation state (TL data, rays, scenarios)
- `src/lib/api.ts` — Axios API client with interceptors

**Day 13: Environment builder UI**
- `src/pages/EnvironmentBuilder.tsx` — Main environment creation page
- `src/components/EnvironmentMap.tsx` — Mapbox map for selecting geographic region
- `src/components/BathymetryEditor.tsx` — Bathymetry type selector (flat/sloped/GEBCO) with parameter inputs
- `src/components/SSPEditor.tsx` — SSP type selector (isovelocity/linear/Munk/WOA) with parameter inputs
- `src/components/BottomEditor.tsx` — Bottom properties editor (density, sound speed, attenuation)
- Auto-fetch GEBCO/WOA data when user selects region on map

**Day 14: Environment visualization**
- `src/components/BathymetryPlot.tsx` — D3.js line chart showing bathymetry vs. range
- `src/components/SSPPlot.tsx` — D3.js line chart showing sound speed vs. depth
- `src/components/EnvironmentSummary.tsx` — Key environment parameters display
- Interactive editing: click-drag bathymetry points, adjust SSP control points
- Preview panel showing both plots side-by-side

**Day 15: Geographic data integration**
- Python FastAPI service: `ocean-data-service/main.py`
- `/bathymetry` endpoint: fetch GEBCO from PostGIS
- `/sound-speed` endpoint: fetch WOA from PostGIS
- PostGIS spatial queries with ST_Intersects and bilinear interpolation
- Frontend: fetch and display real oceanographic data on map selection

**Day 16: Propagation parameter UI**
- `src/components/PropagationConfig.tsx` — Source depth, frequency, ray count, max angle, max range inputs
- `src/components/MethodSelector.tsx` — Propagation method selector (Ray Tracing only in MVP)
- `src/components/OutputSelector.tsx` — Output type selector (TL map, ray paths, eigenrays)
- Validation: ensure source depth < max bathymetry depth, frequency > 0
- "Run Propagation" button with loading state

**Day 17: WASM solver integration**
- `src/hooks/useWasmSolver.ts` — React hook for loading WASM and running propagation
- WASM bundle loading from CDN with progress indicator
- Pass environment + config to WASM solver
- Receive TL grid or ray paths as Float32Array
- Convert to format for visualization
- Error handling: display solver errors (e.g., "Ray escaped environment")

### Phase 4 — Visualization + Results (Days 18–23)

**Day 18: Transmission loss viewer (WebGL)**
- `src/components/TransmissionLossViewer.tsx` — WebGL2 canvas for TL heatmap
- GPU texture upload for TL data
- Jet colormap shader (blue → cyan → yellow → red)
- Logarithmic color scale (dB)
- Pan/zoom via mouse drag and wheel

**Day 19: TL viewer enhancements**
- `src/components/TLColorScale.tsx` — Color scale legend with min/max dB labels
- `src/components/TLContours.tsx` — SVG contour lines overlay at specified dB levels
- `src/components/TLCursor.tsx` — Crosshair cursor with range/depth/TL readout
- `src/components/TLAxisLabels.tsx` — Range (km) and depth (m) axis labels
- Export TL map as PNG

**Day 20: Ray path visualization**
- `src/components/RayPathViewer.tsx` — SVG canvas showing ray trajectories
- Color-code rays by launch angle or number of bounces
- Toggle ray visibility by angle range
- Highlight eigenrays
- Overlay bathymetry and sound speed contours

**Day 21: Sonar equation calculator**
- `src/components/SonarCalculator.tsx` — Input form for SL, DI, NL, AG, TS (active) or target SL (passive)
- Compute signal excess: SE = SL + DI - TL - (NL - AG) for passive
- Compute signal excess: SE = SL + DI - 2·TL + TS - (NL - AG) for active
- Display SE vs. range plot
- Detection threshold slider (required SE in dB)

**Day 22: Detection probability (Pd vs. range)**
- `src/components/DetectionProbability.tsx` — Pd vs. range curve
- ROC-based model: Pd = 0.5 · erfc((DT - SE)/sqrt(2·SNR)) for specified Pfa
- User inputs: Pfa (0.001-0.1), detection threshold (0-20 dB)
- Display detection range (where Pd = 0.5) prominently
- Export Pd vs. range as CSV

**Day 23: Scenario management**
- `src/components/ScenarioList.tsx` — List of saved sonar scenarios for current propagation
- Create/edit/delete scenarios
- Quick comparison: overlay multiple Pd curves on same plot
- Scenario templates: submarine detection, fish finding, AUV communication

### Phase 5 — Server-Side Propagation + Progress (Days 24–28)

**Day 24: Server-side propagation worker**
- `src/workers/propagation_worker.rs` — Redis job consumer
- Fetch environment and config from database
- Run native ray tracer (solver-core compiled for native)
- Generate full TL grid and ray paths
- Upload results to S3 as binary blob
- Update database with results URL

**Day 25: Job queue and prioritization**
- Redis sorted set for job queue (priority by plan tier)
- Worker pool management (configurable concurrency)
- Worker heartbeat and health monitoring
- Job timeout handling (30 min max)
- Job cancellation support

**Day 26: WebSocket progress streaming**
- `src/api/ws/mod.rs` — Axum WebSocket upgrade handler
- `src/api/ws/propagation_progress.rs` — Subscribe to propagation progress channel
- Worker publishes progress → Redis pub/sub → WebSocket to client
- Client receives: `{ progress_pct, rays_completed, rays_total, estimated_time_remaining_s }`
- Frontend: `src/hooks/usePropagationProgress.ts` — React hook for WebSocket subscription

**Day 27: Progress visualization**
- `src/components/ProgressPanel.tsx` — Progress bar, ray count, time remaining
- `src/components/RayProgressChart.tsx` — Real-time chart showing rays completed over time
- Live update every 500ms
- Cancel button to abort running propagation
- Error display with retry button

**Day 28: Results library**
- `src/pages/ResultsLibrary.tsx` — List of all propagations for user
- Filter by environment, frequency, date range
- Sort by date, computation time, ray count
- Preview thumbnails (TL map snapshot)
- Delete results, export results as HDF5

### Phase 6 — Billing + Deployment (Days 29–34)

**Day 29: Stripe integration**
- `src/api/handlers/billing.rs` — Create checkout session, create customer portal session, get subscription status
- `src/api/handlers/webhooks/stripe.rs` — Handle subscription events: `checkout.session.completed`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.payment_failed`
- Plan mapping:
  - Free: no propagation
  - Academic ($79/mo): up to 50K rays per run, 5 user seats
  - Pro ($199/mo): unlimited rays, marine mammal impact, API access
  - Defense ($399/mo): 3D propagation, multi-static, broadband

**Day 30: Usage tracking and plan limits**
- `src/middleware/plan_limits.rs` — Middleware checking plan limits before propagation execution
- `src/services/usage.rs` — Track compute seconds per billing period
- Usage record insertion after each server-side propagation completes
- Usage dashboard: `src/api/handlers/usage.rs` — Current period usage, historical usage
- Approaching-limit warnings at 80% and 100% of quota

**Day 31: Billing UI**
- `frontend/src/pages/Billing.tsx` — Plan comparison table, current plan display, usage meter
- `frontend/src/components/billing/PlanCard.tsx` — Plan feature comparison cards
- `frontend/src/components/billing/UsageMeter.tsx` — Compute usage bar with threshold indicators
- Upgrade/downgrade flow via Stripe Customer Portal
- Upgrade prompt modals when hitting plan limits

**Day 32: Docker and Kubernetes configuration**
- `Dockerfile` — Multi-stage Rust build (cargo-chef for layer caching)
- `docker-compose.yml` — Full local dev stack (API, PostgreSQL+PostGIS, Redis, MinIO, Python service, frontend)
- `k8s/` — Kubernetes manifests:
  - `api-deployment.yaml` — API server (3 replicas, HPA)
  - `worker-deployment.yaml` — Propagation workers (auto-scaling based on queue depth)
  - `postgres-statefulset.yaml` — PostgreSQL with PostGIS and PVC
  - `redis-deployment.yaml` — Redis for job queue
  - `ocean-data-deployment.yaml` — Python FastAPI service
  - `ingress.yaml` — NGINX ingress with TLS
- Health check endpoints: `/health/live`, `/health/ready`

**Day 33: CDN and WASM delivery**
- CloudFront distribution for:
  - Frontend static assets (HTML, JS, CSS) — cached at edge
  - WASM solver bundle (~1MB) — cached at edge with long TTL and versioned URLs
  - Result files from S3 — cached with 1-hour TTL
- WASM preloading: start downloading WASM bundle on page load
- Service worker for offline WASM caching

**Day 34: Monitoring and logging**
- Prometheus metrics: propagation duration histogram, ray throughput, API latency percentiles
- Grafana dashboards: system health, propagation throughput, user activity
- Sentry integration: frontend and backend error tracking with source maps
- Structured logging (tracing + JSON) for all API requests and solver events
- Alerts: worker down, database lag, high error rate

### Phase 7 — Solver Validation + Testing (Days 35–38)

**Day 35: Solver validation — analytical benchmarks**
- Benchmark 1: Isovelocity ocean — verify straight-line rays
- Benchmark 2: Lloyd's mirror — verify interference pattern from surface reflection
- Benchmark 3: Munk profile — verify sound channel refraction and convergence zones
- Automated test suite: `solver-core/tests/benchmarks.rs` with assertions within 1% tolerance

**Day 36: Solver validation — Bellhop comparison**
- Run Bellhop reference simulations for standard test cases
- Compare SonarDeep results: TL at receiver depths, eigenray arrival times
- Tolerance: TL < 2 dB difference, eigenray times < 1% difference
- Document known discrepancies and explain causes

**Day 37: Integration testing**
- End-to-end test: create environment → run propagation → view TL map → compute detection range
- API integration tests: auth → environment CRUD → propagation → results
- WASM solver test: load in headless browser (Playwright), run propagation, verify results
- WebSocket test: connect → subscribe → receive progress → receive completion
- Concurrent user test: 10 users simultaneously running propagations

**Day 38: Performance testing**
- WASM solver benchmarks: measure time for 50 rays, 100 rays, 200 rays in Chrome/Firefox/Safari
- Target: 10K rays < 3 seconds in WASM
- Server solver benchmarks: measure throughput (rays/second) on c7g.2xlarge
- Target: 100K rays < 30 seconds on server
- Frontend rendering: measure TL viewer FPS with various grid sizes
- Target: 60 FPS pan/zoom with 1000×500 TL grid
- Load testing: 50 concurrent users via k6

### Phase 8 — Launch Preparation (Days 39–42)

**Day 39: Oceanographic data ingestion**
- Download GEBCO 2024 global bathymetry grid (15 arc-second resolution)
- Parse GeoTIFF and load into PostGIS raster table (tiled by 5°×5° regions)
- Download WOA23 temperature and salinity data (annual climatology)
- Compute sound speed via Mackenzie equation: c = f(T, S, z)
- Load into PostGIS as compressed binary arrays

**Day 40: Documentation and examples**
- Getting started guide: create first environment, run propagation, interpret TL map
- Example scenarios: submarine detection in North Atlantic, AUV communication in shallow water
- API documentation: Swagger/OpenAPI spec for all endpoints
- Keyboard shortcuts reference
- FAQ: when to use ray tracing vs. other methods, how to choose frequency

**Day 41: UI polish and accessibility**
- Loading states for all async operations
- Error messages with actionable suggestions
- Empty states: "No environments yet. Create your first environment."
- Responsive layout: works on tablets (desktop-only warning on phones)
- Accessibility: keyboard navigation, ARIA labels, screen reader support
- Color blindness: alternative colormap options for TL viewer

**Day 42: Launch preparation**
- End-to-end smoke test in production environment
- Database backup and restore verification
- Rate limiting: 100 req/min per user, 10 propagations/min per user
- Security audit: JWT validation, SQL injection (SQLx prevents this), CORS configuration, CSP headers
- Landing page with product overview, pricing, signup
- Deploy to production, enable monitoring alerts, announce launch

---

## Critical Files

```
sonardeep/
├── solver-core/                           # Shared solver library (native + WASM)
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── environment.rs                 # Environment struct (bathymetry, SSP, bottom)
│   │   ├── sound_speed.rs                 # SSP models (isovelocity, linear, Munk, custom)
│   │   ├── attenuation.rs                 # Thorp's formula for volume attenuation
│   │   ├── ray_tracing.rs                 # Ray tracer core (trace_rays, RK4 integration)
│   │   ├── integration.rs                 # RK4 integrator for ray ODEs
│   │   ├── transmission_loss.rs           # TL grid computation from ray field
│   │   ├── eigenray.rs                    # Eigenray search via binary search
│   │   └── sonar_equation.rs              # Sonar equation and Pd computation
│   └── tests/
│       ├── benchmarks.rs                  # Solver validation benchmarks
│       └── bellhop_comparison.rs          # Comparison with Bellhop reference

├── solver-wasm/                           # WASM compilation target
│   ├── Cargo.toml
│   └── src/
│       └── lib.rs                         # WASM entry points (wasm-bindgen)

├── sonardeep-api/                         # Rust API server
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
│   │   │   │   ├── environments.rs        # Environment CRUD
│   │   │   │   ├── propagation.rs         # Create/get/cancel propagation
│   │   │   │   ├── sonar_scenarios.rs     # Sonar scenario CRUD
│   │   │   │   ├── billing.rs             # Stripe checkout/portal
│   │   │   │   ├── usage.rs               # Usage tracking
│   │   │   │   └── webhooks/
│   │   │   │       └── stripe.rs          # Stripe webhook handler
│   │   │   └── ws/
│   │   │       ├── mod.rs                 # WebSocket upgrade handler
│   │   │       └── propagation_progress.rs # Live progress streaming
│   │   ├── middleware/
│   │   │   └── plan_limits.rs             # Plan enforcement middleware
│   │   ├── services/
│   │   │   ├── usage.rs                   # Usage tracking service
│   │   │   └── s3.rs                      # S3 client helpers
│   │   └── workers/
│   │       ├── mod.rs                     # Worker pool management
│   │       └── propagation_worker.rs      # Server-side propagation execution
│   ├── migrations/
│   │   └── 001_initial.sql                # Full database schema
│   └── tests/
│       ├── api_integration.rs             # API integration tests
│       └── propagation_e2e.rs             # End-to-end propagation tests

├── ocean-data-service/                    # Python FastAPI (oceanographic data)
│   ├── requirements.txt
│   ├── main.py                            # FastAPI app
│   ├── bathymetry.py                      # GEBCO bathymetry queries
│   ├── sound_speed.py                     # WOA sound speed queries
│   └── Dockerfile

├── frontend/                              # React frontend
│   ├── package.json
│   ├── vite.config.ts
│   ├── src/
│   │   ├── App.tsx                        # Router, providers, layout
│   │   ├── main.tsx                       # Entry point
│   │   ├── stores/
│   │   │   ├── authStore.ts               # Auth state (JWT, user)
│   │   │   ├── environmentStore.ts        # Environment state
│   │   │   └── propagationStore.ts        # Propagation state
│   │   ├── hooks/
│   │   │   ├── usePropagationProgress.ts  # WebSocket hook for live progress
│   │   │   └── useWasmSolver.ts           # WASM solver hook (load + run)
│   │   ├── lib/
│   │   │   ├── api.ts                     # Axios API client
│   │   │   └── wasmLoader.ts              # WASM bundle loading
│   │   ├── pages/
│   │   │   ├── Dashboard.tsx              # Environment list
│   │   │   ├── EnvironmentBuilder.tsx     # Environment creation/editing
│   │   │   ├── PropagationViewer.tsx      # TL map, ray paths, results
│   │   │   ├── ResultsLibrary.tsx         # All propagations
│   │   │   ├── Billing.tsx                # Plan management
│   │   │   ├── Login.tsx                  # Auth pages
│   │   │   └── Register.tsx
│   │   ├── components/
│   │   │   ├── EnvironmentMap.tsx         # Mapbox map for region selection
│   │   │   ├── BathymetryEditor.tsx       # Bathymetry type selector + params
│   │   │   ├── SSPEditor.tsx              # SSP type selector + params
│   │   │   ├── BathymetryPlot.tsx         # D3 bathymetry vs. range plot
│   │   │   ├── SSPPlot.tsx                # D3 sound speed vs. depth plot
│   │   │   ├── PropagationConfig.tsx      # Source depth, frequency, rays config
│   │   │   ├── TransmissionLossViewer.tsx # WebGL TL heatmap
│   │   │   ├── TLColorScale.tsx           # Color scale legend
│   │   │   ├── RayPathViewer.tsx          # SVG ray trajectories
│   │   │   ├── SonarCalculator.tsx        # Sonar equation calculator
│   │   │   ├── DetectionProbability.tsx   # Pd vs. range plot
│   │   │   ├── ProgressPanel.tsx          # Progress bar for server propagations
│   │   │   └── billing/
│   │   │       ├── PlanCard.tsx
│   │   │       └── UsageMeter.tsx
│   │   └── data/
│   │       └── examples/                  # Example environments (JSON)
│   └── public/
│       └── wasm/                          # WASM solver bundle

├── k8s/                                   # Kubernetes manifests
│   ├── api-deployment.yaml
│   ├── worker-deployment.yaml
│   ├── postgres-statefulset.yaml
│   ├── redis-deployment.yaml
│   ├── ocean-data-deployment.yaml
│   └── ingress.yaml

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

### Benchmark 1: Isovelocity Ocean (Ray Straightness)

**Environment:** c = 1500 m/s everywhere, depth = 1000 m, flat bottom

**Ray:** Launch angle = 0° (horizontal), source depth = 100 m

**Expected:** Ray travels in straight line at depth = 100 m

**Validation:** After 50 km range, depth should be **100.0 m ± 0.1 m**

**Tolerance:** < 0.1% (numerical integration error only)

### Benchmark 2: Lloyd's Mirror (Surface Interference)

**Environment:** c = 1500 m/s, depth = 100 m, source depth = 50 m, frequency = 1000 Hz, receiver depth = 50 m

**Expected:** Interference nulls at ranges r_n where path difference = (n + 0.5)λ

**First null at:** r_1 = 2 × h_s × h_r / λ = 2 × 50 × 50 / 1.5 ≈ **3333 m**

**Expected TL at r_1:** **> 40 dB** (deep null due to destructive interference)

**Tolerance:** Null position < 5%, TL at null > 35 dB

### Benchmark 3: Munk Sound Channel (Convergence Zone)

**Environment:** Munk canonical profile, c_min = 1490 m/s at z = 1000 m, channel axis depth = 1000 m, depth = 5000 m

**Ray:** Launch angle = -13° (downward), source depth = 1000 m

**Expected:** Ray cycles through sound channel, first convergence zone at **55-60 km**

**Expected TL at 57 km, 1000 m depth:** **60-65 dB** (focusing at convergence zone)

**Tolerance:** Convergence zone position < 5%, TL < 3 dB difference from Bellhop

### Benchmark 4: Deep Water Propagation (Geometric Spreading)

**Environment:** Linear negative SSP gradient (-0.016 s⁻¹), c(0) = 1540 m/s, depth = 4000 m, source depth = 100 m

**Ray:** Launch angle = +5° (upward), frequency = 100 Hz

**Expected TL at 100 km, 100 m depth:** **90-95 dB** (cylindrical spreading: 10·log10(r) + 60·log10(r/1m) ≈ 90 dB for r=100 km)

**Expected:** Ray refracts downward, reaches maximum depth **~500 m** before turning upward

**Tolerance:** TL < 3 dB difference, turning depth < 10%

### Benchmark 5: Shallow Water (Multiple Bottom Bounces)

**Environment:** c = 1500 m/s, depth = 50 m, source depth = 25 m, bottom loss = 2 dB/bounce

**Ray:** Launch angle = -10° (downward), range = 10 km

**Expected:** ~20 bottom bounces over 10 km (500 m per bounce at 10° angle)

**Expected TL at 10 km:** **70-80 dB** (20·log10(10000) ≈ 80 dB + 40 dB bottom loss)

**Tolerance:** Bounce count < 10%, TL < 5 dB difference

---

## Verification

### Integration Testing Checklist

1. **Auth flow** — Register → login → JWT issued → authorized API calls → token refresh → logout
2. **Environment CRUD** — Create → update bathymetry/SSP → save → reload → verify state preserved
3. **GEBCO/WOA fetch** — Select region on map → fetch bathymetry → fetch SSP → display plots → verify realistic data
4. **WASM propagation** — Small environment (50 rays, 20 km) → WASM solver → results returned → TL map displays
5. **Server propagation** — Large environment (1000 rays, 100 km) → job queued → worker picks up → WebSocket progress → results in S3
6. **TL viewer** — Load TL data → render heatmap → pan/zoom → hover for values → export PNG
7. **Ray path viewer** — Load ray paths → display trajectories → toggle rays → highlight eigenrays
8. **Sonar calculator** — Input SL/DI/NL/AG → compute SE → display vs. range → verify reasonable values
9. **Detection probability** — Input Pfa/threshold → compute Pd → display vs. range → verify detection range
10. **Plan limits** — Academic user → 100K rays → blocked with upgrade prompt
11. **Billing** — Subscribe to Pro → Stripe checkout → webhook → plan updated → limits lifted
12. **Concurrent users** — 10 users simultaneously running server propagations → all complete correctly
13. **Progress streaming** — Start large propagation → connect WebSocket → receive progress updates → verify accuracy
14. **Error handling** — Invalid environment (source below bottom) → meaningful error message → no crash
15. **GEBCO tile loading** — Request bathymetry for North Atlantic → PostGIS query → correct depths returned

### SQL Verification Queries

```sql
-- 1. Propagation throughput and success rate
SELECT
    execution_mode,
    status,
    COUNT(*) as count,
    AVG(computation_time_ms)::int as avg_time_ms,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY computation_time_ms)::int as p95_time_ms
FROM propagations
WHERE created_at >= NOW() - INTERVAL '7 days'
GROUP BY execution_mode, status;

-- 2. Propagation method distribution
SELECT method, COUNT(*) as count,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 1) as pct
FROM propagations
WHERE created_at >= NOW() - INTERVAL '30 days'
GROUP BY method ORDER BY count DESC;

-- 3. User plan distribution
SELECT plan, COUNT(*) as users,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 1) as pct
FROM users GROUP BY plan ORDER BY users DESC;

-- 4. Compute usage by user (billing period)
SELECT u.email, u.plan,
    SUM(ur.quantity) as total_compute_seconds,
    CASE u.plan
        WHEN 'academic' THEN 36000
        WHEN 'pro' THEN 180000
        WHEN 'defense' THEN 999999
    END as limit_seconds
FROM usage_records ur
JOIN users u ON ur.user_id = u.id
WHERE ur.period_start <= CURRENT_DATE AND ur.period_end >= CURRENT_DATE
    AND ur.record_type = 'propagation_compute_seconds'
GROUP BY u.email, u.plan
ORDER BY total_compute_seconds DESC;

-- 5. Popular environment types
SELECT env_type, COUNT(*) as count
FROM environments
GROUP BY env_type ORDER BY count DESC;

-- 6. Average ray count by plan tier
SELECT u.plan,
    AVG(p.ray_count)::int as avg_rays,
    MAX(p.ray_count) as max_rays
FROM propagations p
JOIN users u ON p.user_id = u.id
WHERE p.created_at >= NOW() - INTERVAL '30 days'
GROUP BY u.plan;
```

### Performance Benchmarks

| Operation | Target | Method |
|-----------|--------|--------|
| WASM solver: 50 rays × 50 km | <1s | Browser benchmark (Chrome DevTools) |
| WASM solver: 100 rays × 100 km | <3s | Browser benchmark |
| Server solver: 1K rays × 100 km | <10s | Server timing, c7g.2xlarge 8 cores |
| Server solver: 10K rays × 100 km | <60s | Server timing, c7g.2xlarge 8 cores |
| TL viewer: 1000×500 grid render | 60 FPS | Chrome FPS counter during pan/zoom |
| TL viewer: 2000×1000 grid render | 30 FPS | Chrome FPS counter during pan/zoom |
| Environment builder: GEBCO fetch | <2s | API latency for 5°×5° tile |
| API: create propagation | <200ms | p95 latency (Prometheus) |
| API: fetch TL results from S3 | <500ms | Presigned URL generation + download start |
| WASM bundle load (cached) | <300ms | Service worker cache hit |
| WASM bundle load (cold) | <2s | CDN delivery, 1MB gzipped |
| WebSocket: progress latency | <100ms | Time from worker emit to client receive |

---

## Deployment Architecture

### Infrastructure

```
┌─────────────────────────────────────────────────┐
│                  CloudFront CDN                   │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────┐ │
│  │ Frontend SPA │  │ WASM Bundle │  │ Results  │ │
│  │ (HTML/JS/CSS)│  │ (versioned) │  │ (S3)     │ │
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
│ + PostGIS    │ │ (ElastiCache)│ │ (Results +   │
│ (RDS r6g)    │ │ Cluster mode │ │  GEBCO/WOA)  │
│ Multi-AZ     │ │              │ │              │
└──────────────┘ └──────┬───────┘ └──────────────┘
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ Prop Worker  │ │ Prop Worker  │ │ Prop Worker  │
│ (Rust native)│ │ (Rust native)│ │ (Rust native)│
│ c7g.2xlarge  │ │ c7g.2xlarge  │ │ c7g.2xlarge  │
│ HPA: 2-20    │ │              │ │              │
└──────────────┘ └──────────────┘ └──────────────┘
        │               │               │
        └───────────────┴───────────────┘
                        │
                        ▼
                ┌──────────────┐
                │ Ocean Data   │
                │ Service      │
                │ (Python)     │
                └──────────────┘
```

### Scaling Strategy

- **API servers**: Horizontal pod autoscaler (HPA) at 70% CPU, min 3, max 10
- **Propagation workers**: HPA based on Redis queue depth — scale up when queue > 5 jobs, scale down when idle 5 min
- **Worker instances**: AWS c7g.2xlarge (8 vCPU, 16GB RAM) for compute-optimized ray tracing
- **Database**: RDS PostgreSQL r6g.xlarge with PostGIS, read replicas for GEBCO/WOA queries
- **Redis**: ElastiCache r6g.large cluster with 1 primary + 2 replicas

### S3 Lifecycle Policy

```json
{
  "Rules": [
    {
      "ID": "propagation-results-lifecycle",
      "Prefix": "results/",
      "Status": "Enabled",
      "Transitions": [
        { "Days": 30, "StorageClass": "STANDARD_IA" },
        { "Days": 90, "StorageClass": "GLACIER" }
      ],
      "Expiration": { "Days": 365 }
    },
    {
      "ID": "oceanographic-data-keep",
      "Prefix": "gebco/",
      "Status": "Enabled"
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
| Normal Mode and Parabolic Equation Solvers | Add KRAKEN-like normal mode solver for range-independent environments and split-step Fourier PE solver for range-dependent 2D propagation with higher accuracy than ray tracing | High |
| Range-Dependent Environments | Support varying bathymetry, SSP, and sediment along propagation path with automatic profile interpolation and coupled-mode propagation or 2D PE | High |
| Sonar Array Design and Beamforming | Transducer element models (piston, line array), array configurations (linear, planar, cylindrical), conventional and adaptive beamforming, beam pattern computation with mainlobe/sidelobe analysis | High |
| 3D Propagation | Full 3D ray tracing and 3D PE for environments with lateral variation (horizontal refraction), 3D TL field visualization with range-range-depth slicing | Medium |
| Underwater Noise Impact Assessment | Pile driving, shipping, seismic airgun source models with frequency weighting (Southall 2019), regulatory threshold checks (NOAA, JNCC), cumulative SEL computation, marine mammal density integration | Medium |
| Broadband Propagation | Pulse propagation with frequency integration, time-domain waveform prediction, impulse response computation for communication channel analysis | Medium |
| Matched Field Processing | Replica field computation for source localization, ambiguity surfaces, Bartlett and MVDR processors, environmental mismatch analysis | Low |
| Underwater Communications Simulation | Channel characterization (multipath spread, Doppler), modulation schemes (FSK, PSK, OFDM), BER vs. SNR prediction, acoustic modem link budget, multi-node network simulation | Low |
