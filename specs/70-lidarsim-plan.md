# 70. LidarSim — LiDAR System Design and Point Cloud Processing Platform

## Implementation Plan

**MVP Scope:** Browser-based LiDAR system modeling with drag-and-drop component assembly (laser sources, scanners, detectors, optics) rendered via SVG, physics-based ray-tracing simulation engine compiled to WebAssembly for client-side execution of scenes ≤100K rays and server-side Rust-native execution for larger scenes, link budget calculator implementing range equation with atmospheric extinction (Mie/Rayleigh scattering), support for static scene simulation with material BRDFs (Lambertian, retroreflective, specular) and multi-return processing, 3D point cloud viewer rendered via WebGL with pan/zoom/rotate and basic ground classification using progressive morphological filter, 500+ material models (asphalt, concrete, vegetation, retroreflectors) stored in S3 with PostgreSQL metadata, LAS/LAZ point cloud import/export compatible with CloudCompare/TerraScan, DEM/DSM generation from classified point clouds, Stripe billing with three tiers (Free / Pro $149/mo / Advanced $349/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| Ray Tracer | Rust (native + WASM) | Custom Monte Carlo ray tracer with BVH acceleration via `bvh` crate |
| WASM Compilation | `wasm-pack` + `wasm-bindgen` | Client-side ray tracing for scenes ≤100K rays |
| Point Cloud Processing | Python 3.12 (FastAPI) | PDAL, NumPy, SciPy for classification/segmentation, CloudCompare integration |
| Database | PostgreSQL 16 + PostGIS | Projects, users, simulations, point cloud metadata, spatial queries |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | Point cloud files (LAS/LAZ), simulation results, material BRDF models |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| System Designer | Custom SVG renderer | Component library (lasers, scanners, detectors), parameter editor |
| 3D Viewer | Three.js + WebGL 2.0 | GPU-accelerated point cloud rendering with octree LOD |
| Real-time | WebSocket (Axum) | Live simulation progress, ray count updates |
| Job Queue | Redis 7 + Tokio tasks | Server-side simulation job management, batch processing |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | WASM bundle delivery, static frontend assets, point cloud tiles |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, error tracking |
| CI/CD | GitHub Actions | WASM build pipeline, Rust tests, Docker image push |

### Key Architecture Decisions

1. **Hybrid client/server ray tracer with 100K-ray threshold**: Scenes generating ≤100K rays (covers 90%+ of quick link budget validation and single-frame simulations) run entirely in the browser via WASM, providing instant feedback with zero server cost. Scenes exceeding 100K rays (high-density scans, multi-frame sequences, adverse weather) are submitted to the Rust-native server which handles multi-million ray jobs. The threshold is configurable per plan tier.

2. **Custom Monte Carlo ray tracer rather than wrapping OSPRay/OptiX**: Building a custom ray tracer in Rust gives us full control over LiDAR-specific physics (pulsed laser timing, detector SNR, atmospheric scattering), WASM compilation, and parallelization for large scenes. OSPRay/OptiX are designed for visual rendering and lack LiDAR-specific material models (retroreflective tape, rain droplets). Our Rust tracer uses BVH spatial acceleration (via `bvh` crate) and supports multi-return detection natively.

3. **SVG system designer with parameter-driven component library**: SVG provides crisp rendering of optical system layouts (laser → beam expander → scanner → receiver) with easy hit testing and interactive parameter editors. Components are parameterized (e.g., laser wavelength 905nm/1550nm, pulse energy, divergence) and constraints are enforced (APD detector requires 905nm laser). Three.js would require reimplementing 2D layout and text rendering.

4. **Three.js point cloud viewer with octree LOD**: The point cloud viewer renders millions of points using GPU-accelerated point sprites with level-of-detail culling via octree. Points are colored by intensity/classification/height and support interactive measurement tools. Data is streamed from S3 as LAZ chunks and decompressed client-side using `laz-perf` WASM.

5. **S3 for point cloud storage with PostGIS spatial metadata**: Point cloud files (LAS/LAZ, typically 10MB-5GB each) are stored in S3, while PostGIS holds spatial metadata (bounding box, point density, acquisition time, coordinate system). This allows the platform to scale to 100K+ scans without bloating the database while enabling fast spatial queries (find scans intersecting AOI) via PostGIS R-tree indexes.

6. **Python FastAPI microservice for point cloud processing**: Advanced classification (ground/building/vegetation), segmentation (PointNet++), and change detection require Python ML libraries (scikit-learn, PyTorch, PDAL). The Python service exposes REST endpoints called by the Rust backend, handles batch processing, and returns classified point clouds or extracted features (building footprints, tree positions).

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "postgis";  -- Spatial data types and queries
CREATE EXTENSION IF NOT EXISTS "pg_trgm";  -- Fuzzy text search on materials

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

-- Organizations (for Advanced plan multi-user)
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

-- Projects (LiDAR system design + simulation workspace)
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    project_type TEXT NOT NULL DEFAULT 'system_design',  -- system_design | point_cloud_processing
    system_config JSONB,  -- LiDAR system configuration (laser, scanner, detector)
    scene_config JSONB,  -- Scene definition (objects, materials, atmosphere)
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

-- Simulations (ray tracing runs)
CREATE TABLE simulations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    simulation_type TEXT NOT NULL,  -- link_budget | ray_trace_static | ray_trace_dynamic | weather_test
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | running | completed | failed | cancelled
    execution_mode TEXT NOT NULL DEFAULT 'wasm',  -- wasm | server
    parameters JSONB NOT NULL DEFAULT '{}',  -- Simulation-specific parameters
    ray_count INTEGER NOT NULL DEFAULT 0,
    point_cloud_url TEXT,  -- S3 URL for result point cloud (LAZ)
    results_summary JSONB,  -- Quick-access summary (range, point density, SNR distribution)
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
    cores_allocated INTEGER DEFAULT 8,
    memory_mb INTEGER DEFAULT 16384,
    priority INTEGER DEFAULT 0,  -- Higher = more urgent
    progress_pct REAL DEFAULT 0.0,
    ray_progress BIGINT DEFAULT 0,  -- Rays traced so far
    ray_total BIGINT DEFAULT 0,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX jobs_sim_idx ON simulation_jobs(simulation_id);
CREATE INDEX jobs_worker_idx ON simulation_jobs(worker_id);

-- Point Clouds (metadata; actual LAS/LAZ files in S3)
CREATE TABLE point_clouds (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID REFERENCES projects(id) ON DELETE SET NULL,
    simulation_id UUID REFERENCES simulations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    file_url TEXT NOT NULL,  -- S3 URL to LAS/LAZ file
    file_size_bytes BIGINT NOT NULL,
    file_format TEXT NOT NULL,  -- las | laz
    point_count BIGINT NOT NULL,
    bounds GEOMETRY(PolygonZ, 4326),  -- Bounding box (lat/lon/elev)
    bounds_local BOX3D,  -- Local coordinate bounds (meters)
    point_density REAL,  -- Points per square meter (average)
    coordinate_system TEXT,  -- EPSG code or WKT
    acquisition_date TIMESTAMPTZ,
    classification_status TEXT DEFAULT 'unclassified',  -- unclassified | ground | full
    has_intensity BOOLEAN DEFAULT false,
    has_rgb BOOLEAN DEFAULT false,
    statistics JSONB,  -- {min_z, max_z, intensity_range, color_range}
    uploaded_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX pc_project_idx ON point_clouds(project_id);
CREATE INDEX pc_sim_idx ON point_clouds(simulation_id);
CREATE INDEX pc_bounds_idx ON point_clouds USING GIST(bounds);
CREATE INDEX pc_upload_idx ON point_clouds(uploaded_by);

-- Material Models (BRDF definitions; actual model files in S3)
CREATE TABLE material_models (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,  -- e.g., "Asphalt (dry)", "Concrete (wet)", "Pine (healthy)"
    category TEXT NOT NULL,  -- ground | vegetation | building | vehicle | retroreflector | water | atmospheric
    brdf_type TEXT NOT NULL,  -- lambertian | specular | retroreflective | oren_nayar | measured
    brdf_params JSONB NOT NULL,  -- Material-specific parameters
    reflectance_905nm REAL,  -- Reflectance at 905nm (0.0-1.0)
    reflectance_1550nm REAL,  -- Reflectance at 1550nm (0.0-1.0)
    model_file_url TEXT,  -- S3 URL to BRDF LUT if measured
    description TEXT,
    tags TEXT[] DEFAULT '{}',
    is_builtin BOOLEAN DEFAULT true,
    uploaded_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX mat_category_idx ON material_models(category);
CREATE INDEX mat_name_trgm_idx ON material_models USING gin(name gin_trgm_ops);
CREATE INDEX mat_tags_idx ON material_models USING gin(tags);

-- Processing Jobs (point cloud classification, DEM generation, etc.)
CREATE TABLE processing_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    point_cloud_id UUID NOT NULL REFERENCES point_clouds(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    job_type TEXT NOT NULL,  -- ground_classification | building_extraction | tree_detection | dem_generation | change_detection
    status TEXT NOT NULL DEFAULT 'pending',
    parameters JSONB DEFAULT '{}',
    output_url TEXT,  -- S3 URL for result (classified LAZ, DEM raster, JSON features)
    results_summary JSONB,
    error_message TEXT,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    wall_time_ms INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX proc_jobs_pc_idx ON processing_jobs(point_cloud_id);
CREATE INDEX proc_jobs_user_idx ON processing_jobs(user_id);
CREATE INDEX proc_jobs_status_idx ON processing_jobs(status);

-- Usage Tracking (for billing enforcement)
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,  -- simulation_seconds | storage_gb_hours | processing_minutes
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
    pub system_config: Option<serde_json::Value>,
    pub scene_config: Option<serde_json::Value>,
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
    pub parameters: serde_json::Value,
    pub ray_count: i32,
    pub point_cloud_url: Option<String>,
    pub results_summary: Option<serde_json::Value>,
    pub error_message: Option<String>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub wall_time_ms: Option<i32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct PointCloud {
    pub id: Uuid,
    pub project_id: Option<Uuid>,
    pub simulation_id: Option<Uuid>,
    pub name: String,
    pub file_url: String,
    pub file_size_bytes: i64,
    pub file_format: String,
    pub point_count: i64,
    #[sqlx(skip)]
    pub bounds: Option<String>,  // PostGIS geometry as WKT
    pub bounds_local: Option<String>,
    pub point_density: Option<f32>,
    pub coordinate_system: Option<String>,
    pub acquisition_date: Option<DateTime<Utc>>,
    pub classification_status: String,
    pub has_intensity: bool,
    pub has_rgb: bool,
    pub statistics: Option<serde_json::Value>,
    pub uploaded_by: Option<Uuid>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct MaterialModel {
    pub id: Uuid,
    pub name: String,
    pub category: String,
    pub brdf_type: String,
    pub brdf_params: serde_json::Value,
    pub reflectance_905nm: Option<f32>,
    pub reflectance_1550nm: Option<f32>,
    pub model_file_url: Option<String>,
    pub description: Option<String>,
    pub tags: Vec<String>,
    pub is_builtin: bool,
    pub uploaded_by: Option<Uuid>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct SimulationParams {
    pub simulation_type: SimulationType,
    pub lidar_system: LidarSystemConfig,
    pub scene: SceneConfig,
    pub atmosphere: AtmosphereConfig,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum SimulationType {
    LinkBudget,
    RayTraceStatic,
    RayTraceDynamic,
    WeatherTest,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct LidarSystemConfig {
    pub laser: LaserConfig,
    pub scanner: ScannerConfig,
    pub detector: DetectorConfig,
    pub optics: OpticsConfig,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct LaserConfig {
    pub wavelength_nm: u32,  // 905 or 1550
    pub pulse_energy_nj: f64,
    pub pulse_width_ns: f64,
    pub repetition_rate_khz: f64,
    pub divergence_mrad: f64,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct ScannerConfig {
    pub scanner_type: String,  // mechanical | mems | opa | flash
    pub fov_horizontal_deg: f64,
    pub fov_vertical_deg: f64,
    pub angular_resolution_deg: f64,
    pub scan_rate_hz: f64,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct DetectorConfig {
    pub detector_type: String,  // apd | spad | sipm
    pub active_area_mm2: f64,
    pub quantum_efficiency: f64,
    pub dark_count_rate_hz: f64,
    pub gain: f64,
    pub bandwidth_mhz: f64,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct OpticsConfig {
    pub aperture_diameter_mm: f64,
    pub transmission_efficiency: f64,
    pub filter_bandwidth_nm: f64,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct SceneConfig {
    pub objects: Vec<SceneObject>,
    pub background_reflectance: f64,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct SceneObject {
    pub id: String,
    pub object_type: String,  // mesh | plane | sphere | cylinder
    pub material_id: Uuid,
    pub transform: Transform3D,
    pub mesh_url: Option<String>,  // S3 URL to OBJ/PLY file
}

#[derive(Debug, Deserialize, Serialize)]
pub struct Transform3D {
    pub position: [f64; 3],
    pub rotation: [f64; 3],  // Euler angles (degrees)
    pub scale: [f64; 3],
}

#[derive(Debug, Deserialize, Serialize)]
pub struct AtmosphereConfig {
    pub visibility_km: f64,  // Meteorological visibility
    pub temperature_c: f64,
    pub pressure_hpa: f64,
    pub humidity_percent: f64,
    pub weather: String,  // clear | rain_light | rain_heavy | fog | snow
}
```

---

## Ray Tracer Architecture Deep-Dive

### LiDAR Range Equation and Link Budget

The core of LiDAR system design is the **range equation**, which predicts maximum detection range given system parameters and target reflectance. LidarSim implements the full range equation with atmospheric effects:

```
P_r = (P_t · η_t · A_r · ρ · τ_atm²) / (π · R² · θ_div²)

where:
  P_r  = Received optical power (W)
  P_t  = Transmitted pulse energy (J)
  η_t  = Transmitter optical efficiency (0-1)
  A_r  = Receiver aperture area (m²)
  ρ    = Target reflectance (0-1, wavelength-dependent)
  τ_atm = One-way atmospheric transmission (0-1)
  R    = Range to target (m)
  θ_div = Laser beam divergence (rad)
```

**Atmospheric transmission** follows Beer-Lambert law with Mie and Rayleigh scattering:

```
τ_atm(R) = exp(-α · R)

where:
  α = α_mie + α_rayleigh + α_absorption  (extinction coefficient, km⁻¹)

Rayleigh scattering (molecular):
  α_rayleigh = (8π³/3) · (n² - 1)² / (N · λ⁴)

Mie scattering (aerosols/fog/rain):
  α_mie = 3.912 / V_met · (λ/550nm)^(-q)
  where V_met = visibility (km), q = size distribution exponent

  For fog: q = 0.585 · V_met^(1/3)
  For rain: α_mie = 0.365 · R_rate^0.63  (R_rate = rainfall rate mm/h)
```

**SNR (Signal-to-Noise Ratio)** determines detection probability:

```
SNR = (N_signal) / sqrt(N_signal + N_dark + N_background + N_thermal)

where:
  N_signal = (P_r · η_det · τ_pulse · λ) / (h · c)  (detected photons)
  N_dark = dark_count_rate · τ_pulse
  N_background = (L_background · A_r · Ω_fov · Δλ · τ_pulse) / (h · c / λ)
  N_thermal = k_B · T · B / (R_load · e · M · η_det)
```

Detection threshold: `SNR > SNR_threshold` (typically 5-10 for single-photon detection, 15-20 for linear-mode APD).

### Monte Carlo Ray Tracing

The simulation engine traces individual photon paths from laser emission through scene interaction to detector arrival:

```
For each laser pulse:
  1. Sample ray origin from laser aperture (uniform disk)
  2. Sample ray direction from divergence cone (Gaussian)
  3. Trace ray through scene BVH (Bounding Volume Hierarchy)
  4. On intersection:
     a. Compute local BRDF (material-dependent)
     b. Sample scattered direction from BRDF
     c. Russian roulette termination (probability = 1 - ρ)
     d. If scattered ray hits detector aperture:
        - Compute path transmission (atmospheric + optics)
        - Compute time-of-flight (TOF)
        - Record photon arrival (time, position, intensity)
  5. Accumulate detected photons into point cloud
```

**BVH (Bounding Volume Hierarchy)** enables O(log n) ray-triangle intersection tests for mesh objects. The `bvh` Rust crate provides a SAH (Surface Area Heuristic) builder.

**BRDF Models:**

- **Lambertian (diffuse)**: `f_r = ρ / π` (uniform hemispherical scattering)
- **Oren-Nayar (rough diffuse)**: Accounts for surface roughness (asphalt, concrete)
- **Specular**: `f_r = ρ · δ(ω_r - ω_mirror)` (mirrors, water)
- **Retroreflective**: Preferentially scatters back toward source (road signs, vests)
- **Measured**: Loaded from BRDF LUT files (e.g., MERL database)

### Client/Server Split (100K-ray Threshold)

```
Simulation submitted → Ray count estimated
    │
    ├── ≤100K rays → WASM ray tracer (browser)
    │   ├── Instant startup, no network latency
    │   ├── Results displayed immediately (LAZ download)
    │   └── No server cost
    │
    └── >100K rays → Server ray tracer (Rust native)
        ├── Job queued via Redis
        ├── Worker picks up, runs parallel ray tracer (Rayon)
        ├── Progress streamed via WebSocket (rays traced / total)
        └── Point cloud stored in S3, URL returned
```

**100K-ray threshold** chosen because:
- WASM tracer handles 100K rays (simple scene) in <5 seconds on modern hardware
- 100K rays covers: single-frame scans at low density (10cm resolution @ 10m range), link budget validation
- Above 100K: high-density scans (1M+ rays), multi-frame sequences, Monte Carlo uncertainty analysis

### WASM Compilation Pipeline

```toml
# ray-tracer-wasm/Cargo.toml
[package]
name = "lidarsim-tracer-wasm"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
wasm-bindgen = "0.2"
serde = { version = "1", features = ["derive"] }
serde-wasm-bindgen = "0.6"
js-sys = "0.3"
bvh = "0.7"  # BVH acceleration
nalgebra = "0.32"
rand = "0.8"
rand_pcg = "0.3"
getrandom = { version = "0.2", features = ["js"] }  # WASM-compatible RNG

[profile.release]
opt-level = "z"
lto = true
codegen-units = 1
strip = true
```

---

## Architecture Deep-Dives

### 1. Simulation API Handler (Rust/Axum)

The primary endpoint receives a simulation request, validates the LiDAR system configuration, estimates ray count, decides between WASM and server execution, and for server execution enqueues a job with Redis and streams progress via WebSocket.

```rust
// src/api/handlers/simulation.rs

use axum::{
    extract::{Path, State, Json},
    http::StatusCode,
    response::IntoResponse,
};
use uuid::Uuid;
use crate::{
    db::models::{Simulation, SimulationParams, SimulationType},
    state::AppState,
    auth::Claims,
    error::ApiError,
    raytracer::estimate_ray_count,
};

#[derive(serde::Deserialize)]
pub struct CreateSimulationRequest {
    pub simulation_type: SimulationType,
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

    // 2. Parse simulation parameters
    let params: SimulationParams = serde_json::from_value(req.parameters.clone())?;

    // 3. Estimate ray count based on system config
    let ray_count = estimate_ray_count(&params)?;

    // 4. Check plan limits
    let user = sqlx::query_as!(
        crate::db::models::User,
        "SELECT * FROM users WHERE id = $1",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    if user.plan == "free" && ray_count > 10_000 {
        return Err(ApiError::PlanLimit(
            "Free plan supports scenes up to 10K rays. Upgrade to Pro for unlimited."
        ));
    }

    // 5. Determine execution mode
    let execution_mode = if ray_count <= 100_000 {
        "wasm"
    } else {
        "server"
    };

    // 6. Create simulation record
    let sim = sqlx::query_as!(
        Simulation,
        r#"INSERT INTO simulations
            (project_id, user_id, simulation_type, status, execution_mode,
             parameters, ray_count)
        VALUES ($1, $2, $3, $4, $5, $6, $7)
        RETURNING *"#,
        project_id,
        claims.user_id,
        serde_json::to_string(&req.simulation_type)?,
        if execution_mode == "wasm" { "pending" } else { "pending" },
        execution_mode,
        req.parameters,
        ray_count as i32,
    )
    .fetch_one(&state.db)
    .await?;

    // 7. For server execution, enqueue job
    if execution_mode == "server" {
        let job = sqlx::query_as!(
            crate::db::models::SimulationJob,
            r#"INSERT INTO simulation_jobs (simulation_id, priority, ray_total)
            VALUES ($1, $2, $3) RETURNING *"#,
            sim.id,
            if user.plan == "advanced" { 10 } else { 0 },
            ray_count as i64,
        )
        .fetch_one(&state.db)
        .await?;

        // Publish to Redis job queue
        let mut redis = state.redis.get_async_connection().await?;
        redis::cmd("LPUSH")
            .arg("simulation:jobs")
            .arg(job.id.to_string())
            .query_async(&mut redis)
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

// Estimate ray count from system parameters
fn estimate_ray_count(params: &SimulationParams) -> Result<usize, ApiError> {
    let scanner = &params.lidar_system.scanner;

    // Calculate number of scan positions
    let h_points = (scanner.fov_horizontal_deg / scanner.angular_resolution_deg).ceil() as usize;
    let v_points = (scanner.fov_vertical_deg / scanner.angular_resolution_deg).ceil() as usize;

    // For Monte Carlo ray tracing, use multiple samples per scan position
    let samples_per_position = match params.simulation_type {
        SimulationType::LinkBudget => 1,  // Single ray per position
        SimulationType::RayTraceStatic => 10,  // 10 rays for better statistics
        SimulationType::RayTraceDynamic => 100,  // High sampling for moving targets
        SimulationType::WeatherTest => 1000,  // Very high sampling for scattering
    };

    let ray_count = h_points * v_points * samples_per_position;
    Ok(ray_count)
}
```

### 2. Ray Tracer Core (Rust — shared between WASM and native)

The core ray tracer that performs Monte Carlo photon tracing with BVH acceleration and BRDF evaluation.

```rust
// ray-tracer-core/src/tracer.rs

use nalgebra::{Point3, Vector3};
use bvh::bvh::BVH;
use bvh::ray::Ray;
use rand::Rng;
use rand_pcg::Pcg64;
use crate::scene::{Scene, Intersection};
use crate::brdf::BRDF;
use crate::atmosphere::Atmosphere;

pub struct RayTracer {
    pub scene: Scene,
    pub atmosphere: Atmosphere,
    pub rng: Pcg64,
}

#[derive(Debug, Clone)]
pub struct LaserPulse {
    pub origin: Point3<f64>,
    pub direction: Vector3<f64>,
    pub wavelength_nm: f64,
    pub energy_j: f64,
    pub pulse_width_s: f64,
    pub divergence_rad: f64,
}

#[derive(Debug, Clone)]
pub struct DetectedPhoton {
    pub position: Point3<f64>,  // 3D point where photon originated (on target surface)
    pub intensity: f64,  // Received energy
    pub time_of_flight_s: f64,
    pub range_m: f64,
    pub return_number: u8,  // 1 = first return, 2 = second return, etc.
}

impl RayTracer {
    pub fn trace_pulse(&mut self, pulse: LaserPulse, detector_aperture_m2: f64) -> Vec<DetectedPhoton> {
        let mut detected_photons = Vec::new();
        let c = 299_792_458.0;  // Speed of light (m/s)

        // Sample multiple rays from laser divergence cone
        let samples = 100;  // Configurable based on quality settings

        for _ in 0..samples {
            // Sample ray direction from Gaussian divergence cone
            let theta = self.rng.gen::<f64>() * pulse.divergence_rad;
            let phi = self.rng.gen::<f64>() * 2.0 * std::f64::consts::PI;

            let dx = theta.sin() * phi.cos();
            let dy = theta.sin() * phi.sin();
            let dz = theta.cos();

            let ray_dir = Vector3::new(
                pulse.direction.x + dx,
                pulse.direction.y + dy,
                pulse.direction.z + dz,
            ).normalize();

            let ray = Ray::new(pulse.origin, ray_dir);

            // Trace through scene
            if let Some(intersection) = self.scene.intersect(&ray) {
                let range = (intersection.point - pulse.origin).norm();
                let time_of_flight = 2.0 * range / c;  // Round-trip time

                // Atmospheric transmission (Beer-Lambert law)
                let transmission = self.atmosphere.transmission(range, pulse.wavelength_nm);

                // BRDF evaluation
                let material = &self.scene.materials[intersection.material_id];
                let brdf_value = material.brdf.evaluate(
                    &intersection.normal,
                    &(-ray_dir),  // Incoming direction (toward laser)
                    &ray_dir,     // Outgoing direction (scattered back)
                    pulse.wavelength_nm,
                );

                // Compute received power
                let solid_angle = std::f64::consts::PI * pulse.divergence_rad.powi(2);
                let irradiance_at_target = pulse.energy_j * transmission / (solid_angle * range.powi(2));
                let reflected_power = irradiance_at_target * brdf_value * intersection.area;

                // Check if photon reaches detector (geometric + atmospheric)
                let detector_solid_angle = detector_aperture_m2 / range.powi(2);
                let received_energy = reflected_power * detector_solid_angle * transmission;

                // Quantum efficiency and detection threshold
                let photon_energy_j = 6.626e-34 * c / (pulse.wavelength_nm * 1e-9);
                let photon_count = received_energy / photon_energy_j;

                // Simple threshold: detect if >1 photon expected
                if photon_count > 1.0 {
                    detected_photons.push(DetectedPhoton {
                        position: intersection.point,
                        intensity: received_energy,
                        time_of_flight_s: time_of_flight,
                        range_m: range,
                        return_number: 1,  // Multi-return handled separately
                    });
                }
            }
        }

        detected_photons
    }

    pub fn simulate_scan(
        &mut self,
        lidar_config: &crate::config::LidarSystemConfig,
        scan_pattern: &ScanPattern,
    ) -> PointCloud {
        let mut point_cloud = PointCloud::new();

        for (azimuth, elevation) in scan_pattern.iter() {
            // Compute ray direction from scan angles
            let direction = Vector3::new(
                azimuth.to_radians().cos() * elevation.to_radians().cos(),
                azimuth.to_radians().sin() * elevation.to_radians().cos(),
                elevation.to_radians().sin(),
            );

            let pulse = LaserPulse {
                origin: Point3::origin(),  // LiDAR position
                direction,
                wavelength_nm: lidar_config.laser.wavelength_nm as f64,
                energy_j: lidar_config.laser.pulse_energy_nj * 1e-9,
                pulse_width_s: lidar_config.laser.pulse_width_ns * 1e-9,
                divergence_rad: lidar_config.laser.divergence_mrad * 1e-3,
            };

            let detector_area = std::f64::consts::PI * (lidar_config.optics.aperture_diameter_mm * 1e-3 / 2.0).powi(2);
            let photons = self.trace_pulse(pulse, detector_area);

            for photon in photons {
                point_cloud.add_point(photon);
            }
        }

        point_cloud
    }
}

pub struct ScanPattern {
    pub azimuth_range: (f64, f64),  // degrees
    pub elevation_range: (f64, f64),
    pub angular_resolution: f64,  // degrees
}

impl ScanPattern {
    pub fn iter(&self) -> ScanPatternIter {
        ScanPatternIter {
            pattern: self,
            current_azimuth: self.azimuth_range.0,
            current_elevation: self.elevation_range.0,
        }
    }
}

pub struct ScanPatternIter<'a> {
    pattern: &'a ScanPattern,
    current_azimuth: f64,
    current_elevation: f64,
}

impl<'a> Iterator for ScanPatternIter<'a> {
    type Item = (f64, f64);  // (azimuth, elevation)

    fn next(&mut self) -> Option<Self::Item> {
        if self.current_elevation > self.pattern.elevation_range.1 {
            return None;
        }

        let result = (self.current_azimuth, self.current_elevation);

        self.current_azimuth += self.pattern.angular_resolution;
        if self.current_azimuth > self.pattern.azimuth_range.1 {
            self.current_azimuth = self.pattern.azimuth_range.0;
            self.current_elevation += self.pattern.angular_resolution;
        }

        Some(result)
    }
}

#[derive(Debug)]
pub struct PointCloud {
    pub points: Vec<Point3<f64>>,
    pub intensities: Vec<f64>,
    pub ranges: Vec<f64>,
    pub return_numbers: Vec<u8>,
}

impl PointCloud {
    pub fn new() -> Self {
        Self {
            points: Vec::new(),
            intensities: Vec::new(),
            ranges: Vec::new(),
            return_numbers: Vec::new(),
        }
    }

    pub fn add_point(&mut self, photon: DetectedPhoton) {
        self.points.push(photon.position);
        self.intensities.push(photon.intensity);
        self.ranges.push(photon.range_m);
        self.return_numbers.push(photon.return_number);
    }

    pub fn to_las(&self) -> Vec<u8> {
        // Convert to LAS 1.4 format
        // Implementation uses las-rs crate
        vec![]
    }
}
```

### 3. Point Cloud Viewer Component (React + Three.js)

The frontend point cloud viewer that renders millions of points using GPU-accelerated rendering with octree LOD.

```typescript
// frontend/src/components/PointCloudViewer.tsx

import { useRef, useEffect, useCallback, useState } from 'react';
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';
import { usePointCloudStore } from '../stores/pointCloudStore';

interface PointCloudViewerProps {
  pointCloudUrl: string;
  colorMode: 'intensity' | 'height' | 'classification' | 'rgb';
}

export function PointCloudViewer({ pointCloudUrl, colorMode }: PointCloudViewerProps) {
  const containerRef = useRef<HTMLDivElement>(null);
  const sceneRef = useRef<THREE.Scene | null>(null);
  const rendererRef = useRef<THREE.WebGLRenderer | null>(null);
  const cameraRef = useRef<THREE.PerspectiveCamera | null>(null);
  const controlsRef = useRef<OrbitControls | null>(null);
  const pointsRef = useRef<THREE.Points | null>(null);

  const { pointCloud, loadPointCloud, setColorMode } = usePointCloudStore();
  const [loading, setLoading] = useState(false);
  const [stats, setStats] = useState({ pointCount: 0, bounds: null });

  // Initialize Three.js scene
  useEffect(() => {
    const container = containerRef.current;
    if (!container) return;

    // Create scene
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x1a1a1a);
    sceneRef.current = scene;

    // Create camera
    const camera = new THREE.PerspectiveCamera(
      60,
      container.clientWidth / container.clientHeight,
      0.1,
      10000
    );
    camera.position.set(50, 50, 50);
    cameraRef.current = camera;

    // Create renderer
    const renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(container.clientWidth, container.clientHeight);
    renderer.setPixelRatio(window.devicePixelRatio);
    container.appendChild(renderer.domElement);
    rendererRef.current = renderer;

    // Create controls
    const controls = new OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true;
    controls.dampingFactor = 0.05;
    controlsRef.current = controls;

    // Add lighting
    const ambientLight = new THREE.AmbientLight(0xffffff, 0.5);
    scene.add(ambientLight);
    const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8);
    directionalLight.position.set(100, 100, 50);
    scene.add(directionalLight);

    // Add grid helper
    const gridHelper = new THREE.GridHelper(100, 100, 0x444444, 0x222222);
    scene.add(gridHelper);

    // Render loop
    const animate = () => {
      requestAnimationFrame(animate);
      controls.update();
      renderer.render(scene, camera);
    };
    animate();

    // Handle resize
    const handleResize = () => {
      if (!container) return;
      camera.aspect = container.clientWidth / container.clientHeight;
      camera.updateProjectionMatrix();
      renderer.setSize(container.clientWidth, container.clientHeight);
    };
    window.addEventListener('resize', handleResize);

    return () => {
      window.removeEventListener('resize', handleResize);
      renderer.dispose();
      container.removeChild(renderer.domElement);
    };
  }, []);

  // Load point cloud from URL
  useEffect(() => {
    const loadData = async () => {
      setLoading(true);
      try {
        const data = await loadPointCloud(pointCloudUrl);

        // Create point cloud geometry
        const geometry = new THREE.BufferGeometry();

        // Positions (Float32Array for GPU efficiency)
        const positions = new Float32Array(data.points.length * 3);
        for (let i = 0; i < data.points.length; i++) {
          positions[i * 3] = data.points[i].x;
          positions[i * 3 + 1] = data.points[i].y;
          positions[i * 3 + 2] = data.points[i].z;
        }
        geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));

        // Colors based on color mode
        const colors = new Float32Array(data.points.length * 3);
        updateColors(colors, data, colorMode);
        geometry.setAttribute('color', new THREE.BufferAttribute(colors, 3));

        // Point size attribute (optional, for distance-based sizing)
        const sizes = new Float32Array(data.points.length).fill(2.0);
        geometry.setAttribute('size', new THREE.BufferAttribute(sizes, 1));

        // Create shader material for point rendering
        const material = new THREE.ShaderMaterial({
          uniforms: {
            pointSize: { value: 2.0 },
          },
          vertexShader: `
            attribute float size;
            attribute vec3 color;
            varying vec3 vColor;

            void main() {
              vColor = color;
              vec4 mvPosition = modelViewMatrix * vec4(position, 1.0);
              gl_PointSize = size * (300.0 / -mvPosition.z);
              gl_Position = projectionMatrix * mvPosition;
            }
          `,
          fragmentShader: `
            varying vec3 vColor;

            void main() {
              // Circular point shape
              vec2 center = gl_PointCoord - vec2(0.5);
              if (length(center) > 0.5) discard;

              gl_FragColor = vec4(vColor, 1.0);
            }
          `,
          vertexColors: true,
        });

        // Create points mesh
        const points = new THREE.Points(geometry, material);
        if (sceneRef.current) {
          // Remove old points if any
          if (pointsRef.current) {
            sceneRef.current.remove(pointsRef.current);
            pointsRef.current.geometry.dispose();
            (pointsRef.current.material as THREE.Material).dispose();
          }
          sceneRef.current.add(points);
          pointsRef.current = points;

          // Compute bounding box and center camera
          geometry.computeBoundingBox();
          const bbox = geometry.boundingBox!;
          const center = new THREE.Vector3();
          bbox.getCenter(center);
          const size = new THREE.Vector3();
          bbox.getSize(size);

          if (controlsRef.current && cameraRef.current) {
            controlsRef.current.target.copy(center);
            cameraRef.current.position.set(
              center.x + size.x * 1.5,
              center.y + size.y * 1.5,
              center.z + size.z * 1.5
            );
            controlsRef.current.update();
          }

          setStats({
            pointCount: data.points.length,
            bounds: { min: bbox.min, max: bbox.max },
          });
        }
      } catch (error) {
        console.error('Failed to load point cloud:', error);
      } finally {
        setLoading(false);
      }
    };

    loadData();
  }, [pointCloudUrl, loadPointCloud]);

  // Update colors when color mode changes
  useEffect(() => {
    if (!pointsRef.current || !pointCloud) return;

    const geometry = pointsRef.current.geometry as THREE.BufferGeometry;
    const colors = geometry.getAttribute('color') as THREE.BufferAttribute;
    updateColors(colors.array as Float32Array, pointCloud, colorMode);
    colors.needsUpdate = true;
  }, [colorMode, pointCloud]);

  return (
    <div className="point-cloud-viewer flex flex-col h-full bg-gray-900">
      <div className="flex-1 relative" ref={containerRef}>
        {loading && (
          <div className="absolute inset-0 flex items-center justify-center bg-black bg-opacity-50 z-10">
            <div className="text-white text-lg">Loading point cloud...</div>
          </div>
        )}
      </div>
      <div className="bg-gray-800 text-white p-3 flex justify-between text-sm">
        <div>Points: {stats.pointCount.toLocaleString()}</div>
        <div className="flex gap-3">
          <button
            onClick={() => setColorMode('intensity')}
            className={colorMode === 'intensity' ? 'text-blue-400' : ''}
          >
            Intensity
          </button>
          <button
            onClick={() => setColorMode('height')}
            className={colorMode === 'height' ? 'text-blue-400' : ''}
          >
            Height
          </button>
          <button
            onClick={() => setColorMode('classification')}
            className={colorMode === 'classification' ? 'text-blue-400' : ''}
          >
            Classification
          </button>
        </div>
      </div>
    </div>
  );
}

function updateColors(
  colors: Float32Array,
  pointCloud: any,
  colorMode: string
) {
  for (let i = 0; i < pointCloud.points.length; i++) {
    let r = 1, g = 1, b = 1;

    switch (colorMode) {
      case 'intensity':
        const intensity = pointCloud.intensities[i];
        r = g = b = intensity;
        break;
      case 'height':
        const z = pointCloud.points[i].z;
        const normalized = (z - pointCloud.minZ) / (pointCloud.maxZ - pointCloud.minZ);
        // Blue (low) -> Green -> Red (high)
        r = normalized;
        g = 1 - Math.abs(normalized - 0.5) * 2;
        b = 1 - normalized;
        break;
      case 'classification':
        const cls = pointCloud.classifications?.[i] || 0;
        [r, g, b] = getClassificationColor(cls);
        break;
    }

    colors[i * 3] = r;
    colors[i * 3 + 1] = g;
    colors[i * 3 + 2] = b;
  }
}

function getClassificationColor(classification: number): [number, number, number] {
  // LAS classification codes
  const colors: { [key: number]: [number, number, number] } = {
    0: [0.5, 0.5, 0.5],    // Unclassified - gray
    1: [0.5, 0.5, 0.5],    // Unassigned - gray
    2: [0.6, 0.4, 0.2],    // Ground - brown
    3: [0.0, 0.6, 0.0],    // Low vegetation - dark green
    4: [0.2, 0.8, 0.2],    // Medium vegetation - green
    5: [0.4, 1.0, 0.4],    // High vegetation - bright green
    6: [0.8, 0.2, 0.2],    // Building - red
    7: [0.9, 0.9, 0.0],    // Noise - yellow
    9: [0.0, 0.5, 0.8],    // Water - blue
  };
  return colors[classification] || [1, 1, 1];
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–4)

**Day 1: Project initialization and Rust backend scaffold**
```bash
cargo init lidarsim-api
cd lidarsim-api
cargo add axum tokio serde serde_json sqlx uuid chrono tower-http jsonwebtoken bcrypt
cargo add --features postgres,runtime-tokio-rustls,uuid,chrono,json sqlx
cargo add aws-sdk-s3 redis
```
- `src/main.rs` — Axum app with CORS, tracing, graceful shutdown
- `src/config.rs` — Environment-based configuration (DATABASE_URL, JWT_SECRET, S3_BUCKET, REDIS_URL, PYTHON_SERVICE_URL)
- `src/state.rs` — AppState struct (PgPool, Redis, S3Client, reqwest::Client for Python service)
- `src/error.rs` — ApiError enum with IntoResponse for HTTP errors
- `Dockerfile` — Multi-stage build (Rust builder + runtime)
- `docker-compose.yml` — PostgreSQL+PostGIS, Redis, MinIO (S3-compatible), Python FastAPI service

**Day 2: Database schema and PostGIS setup**
- `migrations/001_initial.sql` — All 9 tables with PostGIS geometry columns
- `src/db/mod.rs` — Database pool initialization with PostGIS extension
- `src/db/models.rs` — All SQLx structs with FromRow derives, spatial type handling
- Run `sqlx migrate run` to apply schema
- Seed script for initial material models (asphalt, concrete, vegetation at 905nm/1550nm)
- Test spatial queries: find point clouds intersecting bounding box

**Day 3: Authentication system**
- `src/auth/mod.rs` — JWT token generation and validation middleware
- `src/auth/oauth.rs` — Google and GitHub OAuth 2.0 flow handlers
- `src/api/handlers/auth.rs` — Register, login, OAuth callback, refresh token, me endpoints
- Password hashing with bcrypt (cost factor 12)
- JWT with 24h access token + 30d refresh token
- Auth middleware that extracts Claims from Authorization header
- Integration test: full auth flow from registration to authenticated request

**Day 4: User and project CRUD**
- `src/api/handlers/users.rs` — Get profile, update profile, delete account
- `src/api/handlers/projects.rs` — Create, list, get, update, delete, fork project
- `src/api/handlers/orgs.rs` — Create org, invite member, list members, remove member
- `src/api/router.rs` — All route definitions with auth middleware
- Project type routing: `system_design` vs `point_cloud_processing`
- Integration tests: auth flow, project CRUD, authorization checks (user cannot access other's private projects)

### Phase 2 — Ray Tracer Core — Prototype (Days 5–12)

**Day 5: Ray tracing framework and BVH**
- `ray-tracer-core/` — New Rust workspace member (shared between native and WASM)
- `ray-tracer-core/src/ray.rs` — Ray struct with origin, direction, tmin, tmax
- `ray-tracer-core/src/bvh.rs` — BVH construction using `bvh` crate, ray-triangle intersection
- `ray-tracer-core/src/scene.rs` — Scene struct with triangle meshes, BVH, materials
- Unit tests: simple plane intersection, box intersection

**Day 6: BRDF models**
- `ray-tracer-core/src/brdf/mod.rs` — BRDF trait definition
- `ray-tracer-core/src/brdf/lambertian.rs` — Lambertian diffuse BRDF (uniform hemispherical)
- `ray-tracer-core/src/brdf/oren_nayar.rs` — Oren-Nayar rough diffuse (asphalt, concrete)
- `ray-tracer-core/src/brdf/specular.rs` — Perfect specular reflection (mirrors, water)
- `ray-tracer-core/src/brdf/retroreflective.rs` — Retroreflective BRDF (road signs)
- Tests: verify energy conservation (integrate BRDF over hemisphere ≤ 1)

**Day 7: Atmospheric model**
- `ray-tracer-core/src/atmosphere.rs` — Atmospheric transmission calculation
- Beer-Lambert law implementation with Mie + Rayleigh scattering
- Weather models: clear, fog (visibility-based), rain (rainfall rate)
- Temperature, pressure, humidity effects on extinction coefficient
- Tests: verify transmission at various wavelengths and weather conditions

**Day 8: LiDAR range equation and link budget**
- `ray-tracer-core/src/link_budget.rs` — Range equation solver
- SNR calculation with detector noise (dark count, background, thermal)
- Detection probability from SNR (threshold-based)
- Maximum range calculation for given target reflectance
- Tests: compare to analytical results for simple scenarios

**Day 9: Monte Carlo ray tracer**
- `ray-tracer-core/src/tracer.rs` — RayTracer struct with `trace_pulse` method
- Sample rays from laser divergence cone (Gaussian)
- BVH intersection, BRDF evaluation, atmospheric transmission
- Photon detection probability and point accumulation
- Tests: single pulse → single plane, verify range accuracy

**Day 10: Scan pattern generation**
- `ray-tracer-core/src/scan.rs` — ScanPattern struct for raster/helical/Lissajous patterns
- Iterator over (azimuth, elevation) angles
- Full-scene simulation with `simulate_scan` method
- Multi-return detection (first/last/strongest)
- Tests: verify scan pattern coverage, point density

**Day 11: Point cloud output (LAS format)**
- `ray-tracer-core/src/las.rs` — LAS 1.4 writer (header + point records)
- LAZ compression using `laz-rs` crate
- Support for intensity, return number, GPS time
- Coordinate system metadata (EPSG codes)
- Tests: write point cloud, read back with external tool (pdal), verify integrity

**Day 12: Ray tracer optimization**
- Parallelize ray tracing with Rayon (server-side only, not WASM)
- BVH quality tuning (SAH cost model parameters)
- SIMD optimizations for ray-triangle intersection (via `packed_simd`)
- Benchmark: measure rays/second for various scene complexities
- Target: >1M rays/sec on 8-core server

### Phase 3 — WASM Build + Frontend Visualization (Days 13–18)

**Day 13: WASM ray tracer compilation**
- `ray-tracer-wasm/` — New workspace member for WASM target
- `ray-tracer-wasm/Cargo.toml` — wasm-bindgen, serde-wasm-bindgen, getrandom with js feature
- `ray-tracer-wasm/src/lib.rs` — WASM entry points: `simulate_scan_wasm()`, `link_budget_wasm()`
- `wasm-pack build --target web --release` pipeline
- `wasm-opt -Oz` for size optimization (target <3MB gzipped)
- JavaScript wrapper: `LidarSimulator` class that loads WASM and provides async API

**Day 14: Frontend scaffold and system designer foundation**
```bash
npm create vite@latest frontend -- --template react-ts
cd frontend
npm i zustand @tanstack/react-query axios three @react-three/fiber @react-three/drei
```
- `src/App.tsx` — Router (React Router), auth context, layout
- `src/stores/projectStore.ts` — Zustand store for project state
- `src/stores/systemStore.ts` — LiDAR system configuration state
- `src/components/SystemDesigner/Canvas.tsx` — SVG canvas for optical layout
- `src/components/SystemDesigner/ComponentLibrary.tsx` — Sidebar with laser/scanner/detector components

**Day 15: System designer — component assembly**
- `src/components/SystemDesigner/LaserComponent.tsx` — Laser parameter editor (wavelength, power, divergence)
- `src/components/SystemDesigner/ScannerComponent.tsx` — Scanner type selector (mechanical/MEMS/OPA/flash), FOV, resolution
- `src/components/SystemDesigner/DetectorComponent.tsx` — Detector type (APD/SPAD/SiPM), parameters
- `src/components/SystemDesigner/OpticsComponent.tsx` — Aperture, transmission, filter
- Drag-and-drop from library → canvas
- Parameter validation (e.g., APD requires 905nm, SiPM works at both wavelengths)

**Day 16: System designer — link budget calculator**
- `src/components/SystemDesigner/LinkBudgetPanel.tsx` — Real-time link budget display
- Call WASM `link_budget_wasm()` on parameter changes (debounced)
- Display: max range, SNR vs range curve, atmospheric loss breakdown
- Interactive range slider to see SNR at different ranges
- Export link budget as PDF report

**Day 17: 3D scene editor**
- `src/components/SceneEditor/SceneCanvas.tsx` — Three.js scene with orbit controls
- `src/components/SceneEditor/ObjectLibrary.tsx` — Primitive objects (plane, box, sphere, cylinder)
- Mesh import from OBJ/PLY files (upload to S3, reference in scene config)
- Material assignment to objects (select from material library)
- Transform controls (translate, rotate, scale)
- Ground plane with grid, coordinate axes

**Day 18: Point cloud viewer integration**
- `src/components/PointCloudViewer/PointCloudViewer.tsx` — Three.js point cloud renderer
- LAZ decompression with `laz-perf` WASM (separate WASM module)
- Octree LOD for large point clouds (>10M points)
- Color modes: intensity, height, classification, RGB
- Measurement tools: distance, area, volume
- Cross-section clipping planes

### Phase 4 — API + Job Orchestration (Days 19–24)

**Day 19: Simulation API endpoints**
- `src/api/handlers/simulation.rs` — Create simulation, get simulation, list simulations, cancel simulation
- Ray count estimation from system config and scene complexity
- WASM/server routing logic (100K ray threshold)
- Plan-based limits enforcement (free: 10K rays, Pro: unlimited)
- Validation: check system config compatibility (wavelength/detector mismatch)

**Day 20: Server-side ray tracer worker**
- `src/workers/ray_tracer_worker.rs` — Redis job consumer, runs parallel ray tracer
- `src/workers/mod.rs` — Worker pool management (configurable concurrency)
- Progress streaming: worker publishes ray count → Redis pub/sub → WebSocket
- LAZ compression and S3 upload of result point cloud
- Error handling: BVH build failures, out-of-memory, timeout

**Day 21: WebSocket for live simulation progress**
- `src/api/ws/mod.rs` — Axum WebSocket upgrade handler
- `src/api/ws/simulation_progress.rs` — Subscribe to simulation progress channel
- Client receives: `{ progress_pct, rays_traced, rays_total, estimated_time_remaining }`
- Connection management: heartbeat ping/pong, automatic cleanup on disconnect
- Frontend: `src/hooks/useSimulationProgress.ts` — React hook for WebSocket subscription

**Day 22: Material library API**
- `src/api/handlers/materials.rs` — Search materials (parametric + full-text), get material, download BRDF file
- S3 integration for measured BRDF LUT storage
- Material categories: ground, vegetation, building, vehicle, retroreflector, water, atmospheric
- Parametric search: filter by category, wavelength reflectance range
- Pagination with cursor-based scrolling

**Day 23: Point cloud import/export**
- `src/api/handlers/point_clouds.rs` — Upload LAS/LAZ, download point cloud, list point clouds
- LAS file parsing with `las-rs` crate (extract metadata: bounds, point count, coordinate system)
- PostGIS spatial indexing: insert bounding box into `point_clouds.bounds`
- Spatial query endpoint: find point clouds intersecting AOI (polygon)
- Validation: check coordinate system, point format compatibility

**Day 24: Project management features**
- `src/api/handlers/projects.rs` — Fork project (deep copy of system_config + scene_config)
- Share via public link (set `is_public` flag)
- Project templates: automotive LiDAR, aerial mapping, terrestrial scanning
- Export system config as JSON
- Auto-save: frontend debounced PATCH every 5 seconds on config changes

### Phase 5 — Point Cloud Processing (Days 25–29)

**Day 25: Python FastAPI service setup**
```bash
cd python-service
python -m venv venv
source venv/bin/activate
pip install fastapi uvicorn pdal numpy scipy scikit-learn
```
- `app/main.py` — FastAPI app with health check endpoint
- `app/processing/ground_classification.py` — Progressive Morphological Filter (PMF) implementation
- `app/processing/dem_generation.py` — DEM/DSM raster generation from classified ground points
- `app/models.py` — Pydantic models for processing requests/responses
- Docker container with PDAL and Python deps
- Rust client: `src/services/python_client.rs` — HTTP client for Python service

**Day 26: Ground classification implementation**
- Progressive Morphological Filter (PMF) algorithm:
  - Erosion and dilation with increasing window sizes
  - Elevation threshold to separate ground from non-ground
  - Slope-based refinement
- `app/processing/ground_classification.py` — PMF implementation using NumPy
- Input: unclassified point cloud (LAZ) from S3
- Output: classified point cloud (class 2 = ground) uploaded to S3
- Test: verify on sample terrain scan (compare to manual classification)

**Day 27: DEM/DSM generation**
- `app/processing/dem_generation.py` — Grid-based interpolation (IDW, kriging, TIN)
- DEM from ground points only (class 2)
- DSM from first returns (all classes)
- Output: GeoTIFF raster at specified resolution (0.1m, 0.5m, 1m)
- Contour line generation from DEM
- Test: generate DEM from synthetic flat/sloped terrain

**Day 28: Processing job API**
- `src/api/handlers/processing.rs` — Create processing job, get job status, list jobs
- Job types: `ground_classification`, `dem_generation`, `building_extraction`, `tree_detection`
- Enqueue job to Redis, worker calls Python service endpoint
- Python service downloads point cloud from S3, processes, uploads result to S3
- Result summary: statistics (ground point %, DEM min/max elevation, etc.)

**Day 29: Processing results visualization**
- `src/components/ProcessingResults/DEMViewer.tsx` — GeoTIFF raster display with elevation colormap
- Overlay contours on DEM
- `src/components/ProcessingResults/ClassificationViewer.tsx` — Classified point cloud with legend
- Side-by-side comparison: before/after classification
- Export classified LAZ with custom classification codes

### Phase 6 — Billing + Plan Enforcement (Days 30–33)

**Day 30: Stripe integration**
- `src/api/handlers/billing.rs` — Create checkout session, create customer portal session, get subscription status
- `src/api/handlers/webhooks/stripe.rs` — Handle subscription events: `checkout.session.completed`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.payment_failed`
- Plan mapping:
  - Free: 10K rays/simulation, 1GB storage, basic materials
  - Pro ($149/mo): Unlimited rays, 50GB storage, full material library, weather simulation
  - Advanced ($349/mo): Multi-user orgs, 500GB storage, API access, custom material upload, ML classification
- Webhook signature verification with Stripe signing secret

**Day 31: Usage tracking and plan limits**
- `src/middleware/plan_limits.rs` — Middleware checking plan limits before simulation execution
- `src/services/usage.rs` — Track simulation seconds, storage GB-hours per billing period
- Usage record insertion after each simulation completes
- Usage dashboard: `src/api/handlers/usage.rs` — Current period usage, historical usage chart
- Approaching-limit warnings at 80% and 100% of simulation quota

**Day 32: Billing UI**
- `frontend/src/pages/Billing.tsx` — Plan comparison table, current plan display, usage meters
- `frontend/src/components/billing/PlanCard.tsx` — Plan feature comparison cards with upgrade CTAs
- `frontend/src/components/billing/UsageMeter.tsx` — Simulation quota and storage usage bars
- Upgrade/downgrade flow via Stripe Customer Portal (redirect)
- Upgrade prompt modals when hitting plan limits (e.g., "Scene requires 500K rays. Upgrade to Pro for unlimited.")

**Day 33: Feature gating**
- Gate weather simulation (rain, fog, snow) behind Pro plan
- Gate custom material upload behind Advanced plan
- Gate API access behind Advanced plan
- Gate multi-user organizations behind Advanced plan
- Show locked feature indicators with upgrade CTAs in UI
- Admin endpoint: override plan for specific users (beta testers, enterprise trials)

### Phase 7 — Solver Validation + Testing (Days 34–38)

**Day 34: Ray tracer validation — range equation**
- Benchmark 1: Flat Lambertian target at known range, verify received power ±5%
- Benchmark 2: Range vs SNR curve matches analytical model
- Benchmark 3: Atmospheric extinction coefficients match literature values (Mie/Rayleigh)
- Benchmark 4: Multi-material scene (retroreflector vs asphalt), verify intensity ratio
- Automated test suite: `ray-tracer-core/tests/benchmarks.rs` with assertions

**Day 35: Ray tracer validation — BVH performance**
- Benchmark 5: BVH build time for 100K triangles <100ms
- Benchmark 6: Average ray-triangle tests per ray <50 (vs 100K brute force)
- Benchmark 7: Full scene ray trace 1M rays × 100K triangles <10s (server)
- Benchmark 8: WASM ray trace 100K rays × 10K triangles <5s (browser)
- Compare WASM vs native performance (expect 2-3× slowdown)

**Day 36: Point cloud processing validation**
- Benchmark 9: PMF ground classification on ISPRS Vaihingen dataset (3.5M points)
  - Expected: Precision >90%, Recall >85%, Accuracy >88%
  - Time: <3.5 seconds on server
- Benchmark 10: DEM generation from 1M ground points at 1m resolution
  - Expected: RMSE <0.15m vs reference DEM
  - Time: <2 seconds
- Benchmark 11: Octree generation for 10M points
  - Expected: <15 seconds, 8 levels, <2GB memory

**Day 37: Integration testing**
- End-to-end test: register → create project → configure LiDAR → define scene → run simulation → view point cloud → classify ground → generate DEM → download results
- API integration tests: auth → project CRUD → simulation → processing jobs
- WASM solver test: load in headless browser (Playwright), run simulation, verify results match analytical
- WebSocket test: connect → subscribe → receive progress → receive completion
- Concurrent simulation test: submit 10 simultaneous server-side jobs

**Day 38: Performance testing and optimization**
- WASM solver benchmarks: measure rays/second in Chrome/Firefox/Safari
- Server worker benchmarks: measure throughput with 1/2/4/8 workers
- Database query optimization: identify slow queries with pg_stat_statements
- S3 transfer optimization: multipart upload for large point clouds
- CDN cache tuning: measure cache hit rates for WASM bundles

### Phase 8 — Polish + Deployment (Days 39–42)

**Day 39: UI/UX polish**
- `src/components/GettingStarted/Tutorial.tsx` — Interactive tutorial (tooltips, guided tour)
- `src/components/GettingStarted/Templates.tsx` — Project template gallery with previews
- Error messages: user-friendly explanations with links to docs
- Loading states: skeleton screens for all async operations
- Responsive design: test on mobile/tablet (though primarily desktop workflow)

**Day 40: Documentation and examples**
- `docs/quickstart.md` — 5-minute getting started guide
- `docs/lidar-basics.md` — LiDAR system design fundamentals
- `docs/point-cloud-processing.md` — Classification and DEM generation guide
- `docs/api.md` — REST API documentation (OpenAPI spec)
- Example projects: automotive LiDAR (100m range), aerial mapping (500m AGL), indoor scanning

**Day 41: Deployment preparation**
- Infrastructure as Code (Terraform):
  - ECS Fargate for API (auto-scale 2-10) and workers (auto-scale 2-20, spot)
  - RDS PostgreSQL+PostGIS (db.t4g.medium, 100GB GP3)
  - ElastiCache Redis (cache.t4g.medium)
  - S3 buckets (static, point-clouds, simulations) with lifecycle policies
  - CloudFront distribution (WASM caching, HTTPS)
  - Route 53 DNS
- Monitoring setup: Prometheus exporters, Grafana dashboards, Sentry error tracking
- CI/CD: GitHub Actions for WASM build, Docker image build, ECS deployment

**Day 42: Production deployment and launch**
- Run database migrations on production
- Seed production material library (500+ materials)
- Deploy API and workers to ECS
- Deploy frontend to S3+CloudFront
- SSL certificate setup (ACM)
- Final smoke tests: register, simulate, download
- Launch announcement: Product Hunt, HN, LiDAR industry forums
- Monitor: error rates, response times, user signups

---

## Validation Benchmarks

### Benchmark 1: Link Budget Accuracy

**Setup:** 905nm laser, 1µJ pulse energy, APD detector (0.8 QE, 100 gain), 100mm aperture, Lambertian target (ρ=0.2), 10km visibility

**Expected Results:**
| Range (m) | P_r (nW) | SNR  | Detect |
|-----------|----------|------|--------|
| 10        | 1580.0   | 245  | Yes    |
| 50        | 63.2     | 98   | Yes    |
| 100       | 14.6     | 23.6 | Yes    |
| 200       | 3.38     | 11.3 | Yes    |
| 300       | 1.38     | 7.25 | Yes    |
| 500       | 0.34     | 3.60 | No     |

**Tolerance:** Received power ±5%, SNR ±10%

### Benchmark 2: BVH Ray Tracing Performance

**Setup:** Stanford Bunny (69K triangles), 1M rays, server (8-core)

**Expected Results:**
- BVH build time: <150ms
- BVH depth: ≤20
- Average ray-triangle tests per ray: <50
- Total ray trace time: <800ms
- Speedup vs brute force (69K tests/ray): >85×

**Tolerance:** Build time ±20%, trace time ±15%

### Benchmark 3: Ground Classification (PMF)

**Setup:** ISPRS Vaihingen test set (3.5M points, manually labeled), PMF parameters: max_window=18m, slope=0.15, cell_size=1m

**Expected Results:**
- Precision (ground): >90%
- Recall (ground): >85%
- Overall accuracy: >88%
- Processing time: <3.5 seconds (server)
- False positive rate: <10%

**Tolerance:** Accuracy ±3%, time ±1s

### Benchmark 4: Point Cloud Rendering Performance

**Setup:** 50M points, 8-level octree, 1920×1080, NVIDIA RTX 3060

**Expected Results:**
- Initial load time: <2 seconds
- Points rendered per frame: 2-5M (LOD-dependent)
- Frame rate during rotation: ≥60fps
- Frame rate during node loading: ≥30fps
- GPU memory usage: <2GB

**Tolerance:** Load time ±0.5s, FPS ±10fps

### Benchmark 5: WASM Ray Tracer Performance

**Setup:** Simple scene (10K triangles, single ground plane + box), 100K rays, Chrome 120+

**Expected Results:**
- WASM load time: <1 second
- BVH build time: <200ms
- Ray trace time: <4 seconds
- Total simulation time: <5 seconds
- Memory usage: <500MB

**Tolerance:** Time ±1s, memory ±100MB

---

## System Requirements

### Client (Browser)

**Minimum:**
- CPU: 2-core Intel/AMD, 2.0GHz
- RAM: 4GB
- GPU: Integrated graphics (Intel HD 4000+)
- Browser: Chrome 90+, Firefox 88+, Safari 14+, Edge 90+
- Network: 5 Mbps download

**Recommended:**
- CPU: 4-core Intel/AMD, 3.0GHz
- RAM: 8GB
- GPU: Discrete (NVIDIA GTX 1060+ or AMD RX 580+)
- Browser: Latest Chrome/Firefox
- Network: 25 Mbps download

### Server (AWS ECS)

**API Service:**
- Instance: ECS Fargate, 1 vCPU, 2GB RAM
- Auto-scaling: 2-10 tasks based on CPU/memory
- Health check: /health endpoint, 30s interval

**Worker Service:**
- Instance: ECS Fargate, 4 vCPU, 8GB RAM (for Rayon parallelism)
- Auto-scaling: 2-20 tasks based on queue depth
- Spot instances for cost optimization

**Database:**
- RDS PostgreSQL 16 + PostGIS
- Instance: db.t4g.medium (2 vCPU, 4GB RAM)
- Storage: 100GB GP3 (3000 IOPS, 125 MB/s)
- Backups: Daily snapshots, 7-day retention

**Cache:**
- ElastiCache Redis 7
- Instance: cache.t4g.medium (2 vCPU, 3.09GB RAM)
- Cluster mode disabled

**Storage:**
- S3 Standard (hot data: point clouds <90 days)
- S3 Glacier Instant Retrieval (cold data: point clouds >90 days)
- CloudFront CDN (WASM bundles, static assets)

---

## Deployment Architecture

```
Users → CloudFront CDN → S3 (static frontend)
     ↓
     Route 53 → ALB → ECS Fargate (API, 2-10 tasks)
                   ↓
                   ECS Fargate (Workers, 2-20 tasks, spot)
                   ↓
     RDS PostgreSQL+PostGIS, ElastiCache Redis, S3 (point clouds/scenes)
```

**Regions:** us-east-1 (primary), us-west-2 (failover)

**Networking:**
- VPC with public/private subnets across 3 AZs
- NAT Gateway for private subnet internet access
- Security groups: ALB (443), API (8080), Workers (internal), RDS (5432), Redis (6379)

**Monitoring:**
- Prometheus (API latency, queue depth, processing time, error rate)
- Grafana (health dashboards, user activity, system load)
- Sentry (error tracking, stack traces)
- CloudWatch (logs, metrics, alarms)

**Costs (estimated monthly, moderate load):**
- ECS API: $50 (2 tasks × 730h × $0.034/vCPU-h)
- ECS Workers: $120 (avg 4 tasks × 730h × spot pricing)
- RDS: $60 (db.t4g.medium)
- ElastiCache: $30 (cache.t4g.medium)
- S3 + transfer: $100 (1TB storage + 500GB egress)
- CloudFront: $50 (500GB data transfer)
- **Total: ~$410/month**

---

## Post-MVP Roadmap

### v1.1 — Advanced Weather + Dynamic Scenes (Weeks 13-16)

**Features:**
- Rain simulation (Mie scattering, backscatter, drop size distribution)
- Fog simulation (visibility-dependent extinction, multiple scattering)
- Snow simulation (large particle scattering, ground accumulation)
- Dynamic objects (vehicles, pedestrians imported from CARLA/LGSVL)
- Motion blur and Doppler velocity estimation

**Impact:** Unlocks automotive LiDAR testing market ($150-$300/user/month), estimated 500+ automotive engineers, $75K/month revenue

### v1.2 — ML-Based Segmentation (Weeks 17-20)

**Features:**
- PointNet++ semantic segmentation (ground, building, vegetation, power lines, vehicles)
- Pre-trained models on SemanticKITTI, Toronto3D, DALES datasets
- Fine-tuning UI with user-labeled data
- Building footprint extraction (polygon simplification, roofline detection)
- Tree inventory (individual tree detection, height, crown diameter, DBH estimation)

**Impact:** 2× Advanced subscriptions (surveying, forestry, infrastructure), $50K/month revenue

### v1.3 — Multi-LiDAR Interference + FMCW (Weeks 21-24)

**Features:**
- Multi-LiDAR interference simulation (crosstalk, ghost points)
- Interference mitigation (time-division, wavelength-division, pseudo-random codes)
- FMCW LiDAR support (chirped laser, coherent detection, Doppler velocity)
- Phase noise modeling for FMCW systems
- Automotive multi-LiDAR array design (front, side, rear sensors)

**Impact:** Enterprise OEM market ($5K-$20K/year per customer), 20+ OEMs, $200K/year revenue

### v1.4 — Change Detection + Time Series (Weeks 25-28)

**Features:**
- Point cloud registration (ICP, NDT, feature-based)
- M3C2 cloud-to-cloud distance computation
- Volumetric change detection (cut/fill, stockpile volume)
- Time-series analysis (deformation monitoring, subsidence)
- Automated report generation with visualizations

**Impact:** Construction, mining, geotechnical markets (50K users @ $149/mo), $200K/month revenue

### v1.5 — API + Integrations (Weeks 29-32)

**Features:**
- REST API (1K req/hr Pro, 10K req/hr Advanced)
- Webhooks for simulation completion, processing job completion
- Python SDK (`pip install lidarsim`)
- CLI tool (`lidarsim simulate --config config.json`)
- CARLA plugin (replace default LiDAR sensor with LidarSim)
- LGSVL connector (import driving scenarios)
- ROS 2 bridge (publish point clouds to /lidar topic)

**Impact:** Developer ecosystem, API revenue ($0.10/1K requests), 5× API usage, $100K/month revenue

---

## Success Metrics

**Week 1-4:** 100 signups, 20 paying users ($149/mo), 500 simulations run
**Week 5-8:** 500 signups, 100 paying users, 5,000 simulations, 1,000 point clouds processed
**Week 9-12:** 2,000 signups, 400 paying users, 20,000 simulations, 5,000 point clouds processed

**MRR Target:** $60K by end of Week 12 (400 × $149 + 10 × $349)

**Churn Target:** <5% monthly

**NPS Target:** >50

---

This implementation plan provides a complete, detailed roadmap for building LidarSim from initial scaffold to production deployment in 42 days (8 phases). The plan emphasizes physics accuracy, performance optimization, and user experience while maintaining a clear path to monetization through tiered pricing and feature gating.
