# 95. WindFarm — Wind Farm Layout Optimization Platform

## Implementation Plan

**MVP Scope:** Browser-based wind farm layout editor with interactive map (Mapbox GL) for turbine placement on real terrain with drag-and-drop positioning and real-time wake visualization, wind resource assessment module accepting met mast CSV data with automated quality control and Weibull distribution fitting per direction sector (16 sectors), custom linearized flow model (WAsP-like IBZ/BZ orographic speedup) compiled to WebAssembly for browser-side terrain speedup computation using SRTM 30m elevation data, Jensen wake model with linear superposition for rapid array loss calculation with full rotor-plane integration, genetic algorithm optimizer in Rust for turbine layout optimization maximizing AEP subject to minimum spacing constraints (3-5 rotor diameters) and site boundary polygons, turbine library with 10+ commercial models (Vestas V90/V117/V150, Siemens Gamesa SG 4.5/5.0/8.0, GE 2.5/3.0/5.3) including power curves and thrust coefficient curves, gross and net AEP calculation with loss waterfall (wake losses 5-15%, availability 95-98%, electrical losses 2-3%), basic PDF energy yield report generation with wind rose visualization and layout maps, PostgreSQL for projects/users/turbine library and S3 for met data time series and terrain tiles, Stripe billing with three tiers (Free / Pro $199/mo / Advanced $499/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| Flow Solver | Rust (native + WASM) | Linearized IBZ/BZ model for orographic speedup, roughness change |
| Wake Engine | Rust (native + WASM) | Jensen wake model with rotor-plane integration |
| Optimizer | Rust (native) | Genetic algorithm + constraint handling for layout optimization |
| WASM Compilation | `wasm-pack` + `wasm-bindgen` | Client-side flow/wake for sites <25 turbines and simple terrain |
| Scientific Services | Python 3.12 (FastAPI) | Met data QC, Weibull fitting, MCP long-term correction, uncertainty analysis |
| Database | PostgreSQL 16 | Projects, users, turbines, met stations, simulations |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Time Series DB | PostgreSQL + TimescaleDB | High-frequency met mast data (10 min averages, 1-year+ records) |
| Auth | Custom JWT + OAuth 2.0 | Google OAuth provider, bcrypt password hashing |
| Object Storage | AWS S3 | Met data files, terrain tiles, simulation results, generated reports |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| Map Visualization | Mapbox GL JS 3.0 | 3D terrain, turbine markers, wake plumes, constraint zones |
| Wind Rose / Charts | D3.js 7 | Wind rose diagrams, energy waterfall charts, AEP probability curves |
| Real-time | WebSocket (Axum) | Live optimization progress, wake recalculation on turbine drag |
| Job Queue | Redis 7 + Tokio tasks | Optimization jobs, CFD flow model (post-MVP), batch simulations |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| Terrain Data | SRTM 30m + AWS Terrain Tiles | Global coverage, preprocessed into Mapbox raster tiles on S3/CloudFront |
| CDN | CloudFront | WASM bundle delivery, terrain tiles, static frontend assets |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, solver convergence tracking, error tracking |
| CI/CD | GitHub Actions | WASM build pipeline, Rust tests, Python tests, Docker image push |

### Key Architecture Decisions

1. **Hybrid client/server flow solver with WASM threshold at 25 turbines and simple terrain**: Wind farms with ≤25 turbines on terrain with RIX (Ruggedness Index) <3% run entirely in the browser via WASM, providing instant wake recalculation as users drag turbines. The linearized IBZ/BZ flow model compiles to ~1.5MB WASM and computes speedup factors for 25 positions in <200ms. Larger farms or complex terrain (RIX >3%) are submitted to the server for native Rust execution with optional CFD refinement (post-MVP). This threshold covers 80%+ of early-stage feasibility studies.

2. **Linearized flow model (WAsP-like) for MVP rather than full CFD**: The IBZ (Internal Boundary Layer) model with BZ (orographic speedup) provides wind resource maps accurate to ±5-8% for simple-to-moderate terrain (RIX <5%) and is 1000× faster than RANS CFD. This is sufficient for feasibility and early optimization. For complex terrain (steep slopes, cliffs, narrow valleys), server-side CFD will be added post-MVP. The linearized model uses terrain elevation gradients and roughness maps to compute speedup and flow inclination at each grid point.

3. **Jensen wake model for MVP with planned upgrade path to Bastankhah-Porte-Agel**: The Jensen (Park) wake model is the industry-standard screening tool, used in 90%+ of early-stage assessments. It models wake deficit as a top-hat velocity deficit expanding linearly with downstream distance. While less accurate than the Bastankhah-Porte-Agel Gaussian model (which is state-of-the-art for modern turbines), Jensen is faster (1ms per turbine pair vs 5ms) and provides conservative wake loss estimates (typically 1-2% higher than measured). Bastankhah and dynamic wake meandering (DWM) will be added post-MVP for bankable energy yield assessments.

4. **Genetic algorithm for layout optimization rather than gradient-based methods**: Wind farm layout optimization is a high-dimensional, multi-modal problem with discrete constraints (minimum spacing, exclusion zones) that make gradient-based methods prone to local optima. A genetic algorithm with elitism, tournament selection, and adaptive mutation provides robust global search. For MVP, we use a population of 100 layouts, 200 generations, convergence in 2-5 minutes for 25-turbine farms. Post-MVP, we'll add gradient-based refinement using adjoint-based AEP gradients for final polish of layouts found by GA.

5. **Mapbox GL for 3D terrain visualization and interactive layout editing**: Mapbox GL provides GPU-accelerated 3D terrain rendering with real-time camera controls, essential for visualizing wind farms in complex topography. Turbines are rendered as custom 3D markers (cylinder tower + cone nacelle + disk rotor) with wake plumes visualized as semi-transparent cones colored by velocity deficit. The Mapbox style can be switched between satellite imagery, topographic maps, and wind resource heatmaps. Deck.gl was considered but rejected due to less mature terrain support.

6. **PostgreSQL + TimescaleDB for met data time series rather than InfluxDB**: TimescaleDB is PostgreSQL with time-series optimizations (automatic partitioning, continuous aggregates). This allows us to store met data in the same database as project metadata, use SQL joins for analysis, and leverage PostgreSQL's reliability and tooling. A typical met mast dataset is 50K-500K rows (10-minute averages for 1-3 years), well within TimescaleDB's performance envelope. For multi-gigabyte datasets (high-frequency LiDAR), we'll use Parquet files on S3 with DuckDB queries (post-MVP).

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "postgis";  -- For geographic spatial queries
CREATE EXTENSION IF NOT EXISTS "timescaledb";  -- For met data time series

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

-- Organizations (for team collaboration)
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
    role TEXT NOT NULL DEFAULT 'member',  -- owner | admin | member | viewer
    joined_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(org_id, user_id)
);
CREATE INDEX org_members_org_idx ON org_members(org_id);
CREATE INDEX org_members_user_idx ON org_members(user_id);

-- Wind Farm Projects
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    location_lat DOUBLE PRECISION NOT NULL,  -- Center of site
    location_lon DOUBLE PRECISION NOT NULL,
    location GEOGRAPHY(POINT, 4326),  -- PostGIS point for spatial queries
    site_boundary JSONB,  -- GeoJSON polygon
    terrain_source TEXT DEFAULT 'srtm30',  -- srtm30 | srtm90 | aster | custom
    roughness_source TEXT DEFAULT 'worldcover',  -- worldcover | corine | custom
    settings JSONB DEFAULT '{}',  -- Grid resolution, wake decay constant, optimization params
    is_public BOOLEAN NOT NULL DEFAULT false,
    forked_from UUID REFERENCES projects(id) ON DELETE SET NULL,
    thumbnail_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);
CREATE INDEX projects_org_idx ON projects(org_id);
CREATE INDEX projects_updated_idx ON projects(updated_at DESC);
CREATE INDEX projects_location_idx ON projects USING GIST(location);

-- Turbine Models Library
CREATE TABLE turbine_models (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,  -- e.g., "Vestas V90-3.0"
    manufacturer TEXT NOT NULL,  -- Vestas, Siemens Gamesa, GE, Enercon
    rated_power_kw INTEGER NOT NULL,
    rotor_diameter_m DOUBLE PRECISION NOT NULL,
    hub_height_m DOUBLE PRECISION NOT NULL,
    cut_in_speed_ms DOUBLE PRECISION NOT NULL,  -- m/s
    rated_speed_ms DOUBLE PRECISION NOT NULL,
    cut_out_speed_ms DOUBLE PRECISION NOT NULL,
    power_curve JSONB NOT NULL,  -- [{ws: m/s, power: kW}]
    thrust_curve JSONB NOT NULL,  -- [{ws: m/s, ct: coefficient}]
    iec_class TEXT,  -- I, II, III, IV (wind class)
    iec_turbulence TEXT,  -- A, B, C (turbulence class)
    is_verified BOOLEAN DEFAULT false,
    is_builtin BOOLEAN DEFAULT true,
    datasheet_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX turbine_models_manufacturer_idx ON turbine_models(manufacturer);
CREATE INDEX turbine_models_power_idx ON turbine_models(rated_power_kw);

-- Turbine Instances (layout)
CREATE TABLE turbines (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    turbine_model_id UUID NOT NULL REFERENCES turbine_models(id),
    name TEXT NOT NULL,  -- T01, T02, etc.
    latitude DOUBLE PRECISION NOT NULL,
    longitude DOUBLE PRECISION NOT NULL,
    location GEOGRAPHY(POINT, 4326),
    elevation_m DOUBLE PRECISION,  -- Ground elevation at turbine base
    hub_height_m DOUBLE PRECISION,  -- Can override model default
    is_active BOOLEAN DEFAULT true,  -- For scenario comparison
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX turbines_project_idx ON turbines(project_id);
CREATE INDEX turbines_model_idx ON turbines(turbine_model_id);
CREATE INDEX turbines_location_idx ON turbines USING GIST(location);

-- Met Stations (measurement locations)
CREATE TABLE met_stations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    station_type TEXT NOT NULL,  -- met_mast | lidar | sodar
    latitude DOUBLE PRECISION NOT NULL,
    longitude DOUBLE PRECISION NOT NULL,
    location GEOGRAPHY(POINT, 4326),
    elevation_m DOUBLE PRECISION,
    tower_height_m DOUBLE PRECISION,  -- For met mast
    measurement_heights DOUBLE PRECISION[] NOT NULL,  -- Anemometer heights [m]
    data_start_date DATE,
    data_end_date DATE,
    data_file_url TEXT,  -- S3 URL to raw data CSV
    qc_report_url TEXT,  -- S3 URL to quality control report
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX met_stations_project_idx ON met_stations(project_id);
CREATE INDEX met_stations_location_idx ON met_stations USING GIST(location);

-- Met Data Time Series (TimescaleDB hypertable)
CREATE TABLE met_data (
    time TIMESTAMPTZ NOT NULL,
    station_id UUID NOT NULL REFERENCES met_stations(id) ON DELETE CASCADE,
    height_m DOUBLE PRECISION NOT NULL,
    wind_speed_ms DOUBLE PRECISION,
    wind_direction_deg DOUBLE PRECISION,  -- 0-360, meteorological convention (from)
    temperature_c DOUBLE PRECISION,
    pressure_hpa DOUBLE PRECISION,
    turbulence_intensity DOUBLE PRECISION,  -- Standard deviation / mean
    quality_flag INTEGER DEFAULT 0,  -- 0=good, 1=suspect, 2=bad, 3=missing
    PRIMARY KEY (time, station_id, height_m)
);
SELECT create_hypertable('met_data', 'time');
CREATE INDEX met_data_station_idx ON met_data(station_id, time DESC);

-- Wind Resource Assessments
CREATE TABLE wind_resources (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    station_id UUID REFERENCES met_stations(id) ON DELETE SET NULL,
    analysis_type TEXT NOT NULL,  -- raw | long_term_corrected | flow_modeled
    reference_height_m DOUBLE PRECISION NOT NULL,
    weibull_params JSONB NOT NULL,  -- Per sector: [{sector: 0-15, A: scale, k: shape, freq: fraction}]
    mean_wind_speed_ms DOUBLE PRECISION NOT NULL,
    mean_power_density_wm2 DOUBLE PRECISION,
    shear_exponent DOUBLE PRECISION,  -- Power law α
    turbulence_intensity DOUBLE PRECISION,  -- Representative TI at 15 m/s
    iec_classification TEXT,  -- Computed IEC wind/turbulence class
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX wind_resources_project_idx ON wind_resources(project_id);
CREATE INDEX wind_resources_station_idx ON wind_resources(station_id);

-- Energy Yield Simulations
CREATE TABLE simulations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    simulation_type TEXT NOT NULL,  -- wake_only | full_yield | optimization
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | running | completed | failed | cancelled
    execution_mode TEXT NOT NULL DEFAULT 'wasm',  -- wasm | server
    layout_snapshot JSONB NOT NULL,  -- Turbine positions at time of simulation
    wind_resource_id UUID REFERENCES wind_resources(id),
    parameters JSONB NOT NULL DEFAULT '{}',  -- Wake model params, loss assumptions
    results JSONB,  -- Summary: total_aep_gwh, capacity_factor, wake_loss_pct, turbine_results[]
    results_url TEXT,  -- S3 URL for detailed results (CSV, JSON)
    report_url TEXT,  -- S3 URL for generated PDF report
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

-- Optimization Jobs
CREATE TABLE optimization_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    simulation_id UUID NOT NULL REFERENCES simulations(id) ON DELETE CASCADE,
    algorithm TEXT NOT NULL,  -- genetic | gradient | hybrid
    constraints JSONB NOT NULL,  -- min_spacing, boundary, exclusion_zones, max_turbines
    objectives JSONB NOT NULL,  -- maximize_aep, minimize_wake_loss, minimize_cable_cost
    population_size INTEGER DEFAULT 100,
    max_generations INTEGER DEFAULT 200,
    current_generation INTEGER DEFAULT 0,
    best_solution JSONB,  -- Best layout found: {turbine_positions: [], aep_gwh: X, fitness: Y}
    convergence_history JSONB DEFAULT '[]',  -- [{generation, best_fitness, mean_fitness}]
    worker_id TEXT,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX opt_jobs_sim_idx ON optimization_jobs(simulation_id);

-- Usage Tracking (for billing enforcement)
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,  -- simulation_run | optimization_run | cfd_run | api_call
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
    pub location_lat: f64,
    pub location_lon: f64,
    pub site_boundary: Option<serde_json::Value>,
    pub terrain_source: String,
    pub roughness_source: String,
    pub settings: serde_json::Value,
    pub is_public: bool,
    pub forked_from: Option<Uuid>,
    pub thumbnail_url: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize, Deserialize)]
pub struct TurbineModel {
    pub id: Uuid,
    pub name: String,
    pub manufacturer: String,
    pub rated_power_kw: i32,
    pub rotor_diameter_m: f64,
    pub hub_height_m: f64,
    pub cut_in_speed_ms: f64,
    pub rated_speed_ms: f64,
    pub cut_out_speed_ms: f64,
    pub power_curve: serde_json::Value,  // Vec<PowerCurvePoint>
    pub thrust_curve: serde_json::Value,  // Vec<ThrustCurvePoint>
    pub iec_class: Option<String>,
    pub iec_turbulence: Option<String>,
    pub is_verified: bool,
    pub is_builtin: bool,
    pub datasheet_url: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct PowerCurvePoint {
    pub ws: f64,  // Wind speed [m/s]
    pub power: f64,  // Power output [kW]
}

#[derive(Debug, Serialize, Deserialize)]
pub struct ThrustCurvePoint {
    pub ws: f64,  // Wind speed [m/s]
    pub ct: f64,  // Thrust coefficient [dimensionless, 0-1]
}

#[derive(Debug, FromRow, Serialize)]
pub struct Turbine {
    pub id: Uuid,
    pub project_id: Uuid,
    pub turbine_model_id: Uuid,
    pub name: String,
    pub latitude: f64,
    pub longitude: f64,
    pub elevation_m: Option<f64>,
    pub hub_height_m: Option<f64>,
    pub is_active: bool,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct MetStation {
    pub id: Uuid,
    pub project_id: Uuid,
    pub name: String,
    pub station_type: String,
    pub latitude: f64,
    pub longitude: f64,
    pub elevation_m: Option<f64>,
    pub tower_height_m: Option<f64>,
    pub measurement_heights: Vec<f64>,
    pub data_start_date: Option<NaiveDate>,
    pub data_end_date: Option<NaiveDate>,
    pub data_file_url: Option<String>,
    pub qc_report_url: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct MetDataPoint {
    pub time: DateTime<Utc>,
    pub station_id: Uuid,
    pub height_m: f64,
    pub wind_speed_ms: Option<f64>,
    pub wind_direction_deg: Option<f64>,
    pub temperature_c: Option<f64>,
    pub pressure_hpa: Option<f64>,
    pub turbulence_intensity: Option<f64>,
    pub quality_flag: i32,
}

#[derive(Debug, FromRow, Serialize)]
pub struct WindResource {
    pub id: Uuid,
    pub project_id: Uuid,
    pub station_id: Option<Uuid>,
    pub analysis_type: String,
    pub reference_height_m: f64,
    pub weibull_params: serde_json::Value,  // Vec<WeibullSector>
    pub mean_wind_speed_ms: f64,
    pub mean_power_density_wm2: Option<f64>,
    pub shear_exponent: Option<f64>,
    pub turbulence_intensity: Option<f64>,
    pub iec_classification: Option<String>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct WeibullSector {
    pub sector: u8,  // 0-15 for 22.5° sectors
    pub a_scale: f64,  // Weibull A parameter [m/s]
    pub k_shape: f64,  // Weibull k parameter [dimensionless]
    pub freq: f64,  // Frequency of this sector [0-1]
}

#[derive(Debug, FromRow, Serialize)]
pub struct Simulation {
    pub id: Uuid,
    pub project_id: Uuid,
    pub user_id: Uuid,
    pub simulation_type: String,
    pub status: String,
    pub execution_mode: String,
    pub layout_snapshot: serde_json::Value,
    pub wind_resource_id: Option<Uuid>,
    pub parameters: serde_json::Value,
    pub results: Option<serde_json::Value>,
    pub results_url: Option<String>,
    pub report_url: Option<String>,
    pub error_message: Option<String>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub wall_time_ms: Option<i32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct OptimizationJob {
    pub id: Uuid,
    pub simulation_id: Uuid,
    pub algorithm: String,
    pub constraints: serde_json::Value,
    pub objectives: serde_json::Value,
    pub population_size: i32,
    pub max_generations: i32,
    pub current_generation: i32,
    pub best_solution: Option<serde_json::Value>,
    pub convergence_history: serde_json::Value,
    pub worker_id: Option<String>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct SimulationParams {
    pub wake_model: WakeModel,
    pub wake_decay_constant: f64,  // Jensen k=0.04-0.05 typical
    pub wake_superposition: WakeSuperposition,
    pub air_density: f64,  // kg/m³, default 1.225 at sea level 15°C
    pub loss_assumptions: LossAssumptions,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum WakeModel {
    Jensen,  // MVP
    Frandsen,  // Post-MVP
    Bastankhah,  // Post-MVP
    DynamicWakeMeandering,  // Post-MVP
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum WakeSuperposition {
    Linear,
    RootSumSquare,
    MaxDeficit,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct LossAssumptions {
    pub availability: f64,  // Fraction, e.g., 0.97
    pub electrical: f64,  // Fraction loss, e.g., 0.02
    pub turbine_performance: f64,  // Fraction loss, e.g., 0.01
    pub environmental: f64,  // Blade icing, soiling, degradation, e.g., 0.02
    pub curtailment: f64,  // Grid/noise/bat curtailment, e.g., 0.01
}
```

---

## Flow Model Architecture Deep-Dive

### Governing Equations and Discretization

WindFarm's flow model implements the **WAsP linearized flow model** (IBZ + BZ + roughness change), the industry-standard method for wind resource mapping over moderate terrain. For a given site, the model computes wind speed and direction at any point based on:

1. **Orographic speedup (BZ model)**: Terrain-induced flow acceleration/deceleration
2. **Internal boundary layer (IBZ model)**: Roughness change effects
3. **Obstacle model**: Buildings, forests (post-MVP)

**Orographic Speedup** is computed using linearized flow equations derived from Jackson & Hunt (1975). For a 2D hill profile h(x), the perturbation to the mean flow U₀ is:

```
ΔU/U₀ = L · h'(x) / z   for x upstream/downstream

where:
- h'(x) = dh/dx (terrain slope)
- L = 2 (empirical constant)
- z = height above ground
```

For 3D terrain, the speedup at point (x, y, z) is computed by convolving the terrain gradient with a Green's function kernel in Fourier space:

```
S(x, y, z) = 1 + ΔU/U₀

S(k_x, k_y, z) = 1 + i·k·A(k)·H(k_x, k_y)·exp(-|k|z)

where:
- k = sqrt(k_x² + k_y²) (horizontal wavenumber)
- H(k_x, k_y) = FFT of terrain elevation h(x, y)
- A(k) = speedup transfer function (depends on stability)
```

Inverse FFT gives speedup factor S(x, y, z) at each grid point. For neutral stability (typical daytime conditions), A(k) ≈ 2.

**Roughness Change** is modeled using the IBZ (internal boundary layer) model. When flow transitions from roughness z₀₁ to z₀₂, a new boundary layer grows downstream:

```
U(z, x) = (U_ref / ln(z_ref/z₀₁)) · ln(z/z₀₂)   for z < δ(x)

δ(x) = 0.3 · x^0.8 · (z₀₂/z₀₁)^0.2   (IBL height)

where:
- x = fetch distance downwind of roughness change
- δ(x) = internal boundary layer height
- z₀₁, z₀₂ = upstream/downstream roughness lengths
```

**Wind Shear** is modeled using the logarithmic wind profile (derived from Monin-Obukhov similarity theory):

```
U(z) = (u_*/κ) · ln((z - d)/z₀)

where:
- u_* = friction velocity [m/s]
- κ = 0.4 (von Kármán constant)
- z₀ = roughness length [m] (0.0002 for water, 0.03 for grass, 0.4 for crops, 1.0 for forest)
- d = displacement height (typically 0.7 × canopy height for forests)
```

For hub-height extrapolation, we use the power law approximation (accurate to ±3% for z > 10m):

```
U(z) = U_ref · (z/z_ref)^α

where α = shear exponent ≈ 0.10-0.30 (depends on stability and roughness)
```

### Client/Server Split (WASM Threshold)

```
Project created → Turbine count + terrain RIX computed
    │
    ├── ≤25 turbines AND RIX <3% → WASM flow + wake (browser)
    │   ├── Flow model: 500×500 grid, 50m resolution, <200ms
    │   ├── Wake computation: 25 turbines, 16 sectors, ~50ms
    │   ├── Real-time recalculation on turbine drag
    │   └── No server cost
    │
    └── >25 turbines OR RIX ≥3% → Server flow + wake (Rust native)
        ├── Flow model: 1000×1000 grid, 25m resolution, ~2s
        ├── Wake computation: unlimited turbines
        ├── Optional CFD refinement for complex terrain (post-MVP)
        └── Results cached in Redis + S3
```

The 25-turbine / RIX 3% thresholds were chosen because:
- 25 turbines covers small-to-medium wind farms (10-100 MW), common for feasibility studies
- WASM flow model with 500×500 grid (50m resolution) runs in <200ms on modern hardware
- Jensen wake model for 25 turbines, 16 sectors, 20 wind speed bins = 25 × 16 × 20 = 8,000 wake computations ≈ 50ms
- RIX (Ruggedness Index) <3% = gentle terrain where linearized model is accurate (±5%)
- Above thresholds: large commercial/utility-scale farms, complex terrain → need server compute + CFD

### WASM Compilation Pipeline

```toml
# flow-wasm/Cargo.toml
[package]
name = "windfarm-flow-wasm"
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
rustfft = "6"  # For terrain FFT in orographic speedup
ndarray = "0.15"
log = "0.4"
console_log = "1"

[profile.release]
opt-level = "z"       # Optimize for size
lto = true            # Link-time optimization
codegen-units = 1
strip = true
wasm-opt = true
```

```yaml
# .github/workflows/wasm-build.yml
name: Build WASM Flow Solver
on:
  push:
    paths: ['flow-wasm/**', 'flow-core/**', 'wake-core/**']
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
      - run: cd flow-wasm && wasm-pack build --target web --release
      - run: wasm-opt -Oz flow-wasm/pkg/windfarm_flow_wasm_bg.wasm -o flow-wasm/pkg/windfarm_flow_wasm_bg.wasm
      - name: Upload to S3/CDN
        run: aws s3 cp flow-wasm/pkg/ s3://windfarm-wasm/v${{ github.sha }}/ --recursive
      - name: Invalidate CloudFront
        run: aws cloudfront create-invalidation --distribution-id ${{ secrets.CLOUDFRONT_ID }} --paths "/wasm/*"
```

---

## Architecture Deep-Dives

### 1. Wind Farm Simulation API Handler (Rust/Axum)

The primary endpoint receives a simulation request (wake analysis or full energy yield), validates the layout, decides between WASM and server execution, and for server execution enqueues a job with Redis.

```rust
// src/api/handlers/simulation.rs

use axum::{
    extract::{Path, State, Json},
    http::StatusCode,
    response::IntoResponse,
};
use uuid::Uuid;
use crate::{
    db::models::{Simulation, SimulationParams, Turbine, TurbineModel, WindResource},
    state::AppState,
    auth::Claims,
    error::ApiError,
    terrain::compute_rix,
};

#[derive(serde::Deserialize)]
pub struct CreateSimulationRequest {
    pub simulation_type: String,  // wake_only | full_yield | optimization
    pub wind_resource_id: Uuid,
    pub parameters: SimulationParams,
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

    // 2. Fetch layout (turbines + models)
    let turbines = sqlx::query_as!(
        Turbine,
        "SELECT * FROM turbines WHERE project_id = $1 AND is_active = true",
        project_id
    )
    .fetch_all(&state.db)
    .await?;

    if turbines.is_empty() {
        return Err(ApiError::BadRequest("No turbines in layout"));
    }

    let turbine_count = turbines.len();

    // Fetch turbine models
    let model_ids: Vec<Uuid> = turbines.iter().map(|t| t.turbine_model_id).collect();
    let models = sqlx::query_as!(
        TurbineModel,
        "SELECT * FROM turbine_models WHERE id = ANY($1)",
        &model_ids
    )
    .fetch_all(&state.db)
    .await?;

    // 3. Fetch wind resource
    let wind_resource = sqlx::query_as!(
        WindResource,
        "SELECT * FROM wind_resources WHERE id = $1",
        req.wind_resource_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Wind resource not found"))?;

    // 4. Compute terrain RIX (Ruggedness Index)
    let rix = compute_rix(
        project.location_lat,
        project.location_lon,
        5000.0,  // 5km radius
        &state.terrain_cache
    ).await?;

    // 5. Check plan limits
    let user = sqlx::query_as!(
        crate::db::models::User,
        "SELECT * FROM users WHERE id = $1",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    if user.plan == "free" && turbine_count > 10 {
        return Err(ApiError::PlanLimit(
            "Free plan supports up to 10 turbines. Upgrade to Pro for unlimited."
        ));
    }

    // 6. Determine execution mode
    let execution_mode = if turbine_count <= 25 && rix < 0.03 {
        "wasm"
    } else {
        "server"
    };

    // 7. Create simulation record
    let layout_snapshot = serde_json::json!({
        "turbines": turbines,
        "models": models,
    });

    let sim = sqlx::query_as!(
        Simulation,
        r#"INSERT INTO simulations
            (project_id, user_id, simulation_type, status, execution_mode,
             layout_snapshot, wind_resource_id, parameters)
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8)
        RETURNING *"#,
        project_id,
        claims.user_id,
        req.simulation_type,
        if execution_mode == "wasm" { "completed" } else { "pending" },
        execution_mode,
        layout_snapshot,
        req.wind_resource_id,
        serde_json::to_value(&req.parameters)?,
    )
    .fetch_one(&state.db)
    .await?;

    // 8. For server execution, enqueue job
    if execution_mode == "server" {
        // Publish to Redis job queue
        state.redis
            .publish("simulation:jobs", serde_json::to_string(&sim.id)?)
            .await?;

        tracing::info!(
            "Simulation {} enqueued (type={}, turbines={}, rix={:.3})",
            sim.id, req.simulation_type, turbine_count, rix
        );
    } else {
        tracing::info!(
            "Simulation {} will run client-side via WASM (turbines={}, rix={:.3})",
            sim.id, turbine_count, rix
        );
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

### 2. Jensen Wake Model Core (Rust — shared between WASM and native)

The Jensen (Park) wake model computes velocity deficit in the wake of each turbine. For turbine i affecting turbine j:

```rust
// wake-core/src/jensen.rs

use serde::{Deserialize, Serialize};
use std::f64::consts::PI;

/// Jensen wake model parameters
#[derive(Debug, Clone, Deserialize, Serialize)]
pub struct JensenParams {
    pub wake_decay_constant: f64,  // k = 0.04-0.05 typical, 0.075 for offshore
    pub air_density: f64,  // kg/m³
}

/// Turbine position and properties
#[derive(Debug, Clone)]
pub struct TurbineInstance {
    pub x: f64,  // Easting [m]
    pub y: f64,  // Northing [m]
    pub z_hub: f64,  // Hub height above ground [m]
    pub rotor_diameter: f64,  // [m]
    pub power_curve: Vec<(f64, f64)>,  // (wind_speed [m/s], power [kW])
    pub thrust_curve: Vec<(f64, f64)>,  // (wind_speed [m/s], Ct [dimensionless])
}

impl TurbineInstance {
    /// Interpolate thrust coefficient at given wind speed
    pub fn thrust_coefficient(&self, wind_speed: f64) -> f64 {
        // Linear interpolation in thrust curve
        if wind_speed <= self.thrust_curve[0].0 {
            return self.thrust_curve[0].1;
        }
        for i in 0..self.thrust_curve.len() - 1 {
            let (ws1, ct1) = self.thrust_curve[i];
            let (ws2, ct2) = self.thrust_curve[i + 1];
            if wind_speed >= ws1 && wind_speed <= ws2 {
                let frac = (wind_speed - ws1) / (ws2 - ws1);
                return ct1 + frac * (ct2 - ct1);
            }
        }
        self.thrust_curve.last().unwrap().1
    }

    /// Interpolate power output at given wind speed
    pub fn power_output(&self, wind_speed: f64) -> f64 {
        if wind_speed <= self.power_curve[0].0 {
            return 0.0;
        }
        for i in 0..self.power_curve.len() - 1 {
            let (ws1, p1) = self.power_curve[i];
            let (ws2, p2) = self.power_curve[i + 1];
            if wind_speed >= ws1 && wind_speed <= ws2 {
                let frac = (wind_speed - ws1) / (ws2 - ws1);
                return p1 + frac * (p2 - p1);
            }
        }
        self.power_curve.last().unwrap().1
    }
}

/// Compute velocity deficit at downstream turbine j due to upstream turbine i
/// Returns deficit fraction (0-1), where 0 = no deficit, 1 = complete wake
pub fn jensen_wake_deficit(
    turbine_i: &TurbineInstance,
    turbine_j: &TurbineInstance,
    wind_direction_deg: f64,  // Meteorological convention: wind FROM this direction
    wind_speed: f64,  // Free-stream wind speed at hub height [m/s]
    params: &JensenParams,
) -> f64 {
    // 1. Compute relative position in wind-aligned coordinates
    let theta_rad = (270.0 - wind_direction_deg).to_radians();  // Convert to math convention
    let dx = turbine_j.x - turbine_i.x;
    let dy = turbine_j.y - turbine_i.y;

    // Rotate to wind-aligned frame (x_w = downwind, y_w = crosswind)
    let x_downwind = dx * theta_rad.cos() + dy * theta_rad.sin();
    let y_crosswind = -dx * theta_rad.sin() + dy * theta_rad.cos();

    // 2. Check if turbine j is downstream of i
    if x_downwind <= 0.0 {
        return 0.0;  // No wake effect
    }

    // 3. Compute wake expansion
    let d_i = turbine_i.rotor_diameter;
    let d_j = turbine_j.rotor_diameter;
    let k = params.wake_decay_constant;

    // Wake radius at distance x_downwind
    let r_wake = d_i / 2.0 + k * x_downwind;

    // 4. Check if turbine j rotor is within wake
    let r_j = d_j / 2.0;  // Rotor radius of downstream turbine
    let distance_to_wake_center = y_crosswind.abs();

    if distance_to_wake_center > r_wake + r_j {
        return 0.0;  // No overlap
    }

    // 5. Compute wake deficit (Jensen model assumes top-hat profile)
    let ct = turbine_i.thrust_coefficient(wind_speed);
    let deficit_centerline = (1.0 - (1.0 - ct).sqrt()) / (1.0 + k * x_downwind / (d_i / 2.0)).powi(2);

    // 6. Compute partial wake overlap fraction (rotor-averaged)
    let overlap_fraction = if distance_to_wake_center + r_j <= r_wake {
        // Full overlap
        1.0
    } else if distance_to_wake_center >= r_wake + r_j {
        // No overlap (already handled above, but safety check)
        0.0
    } else {
        // Partial overlap: use circular segment area formula
        // A = r_j² · arccos(d/r_j) + r_wake² · arccos(d/r_wake) - d · sqrt(r_j² - d²)
        // where d = distance from turbine j center to wake boundary overlap point
        let d = distance_to_wake_center;
        if d + r_j < r_wake {
            // Turbine rotor fully inside wake
            1.0
        } else if d < r_wake + r_j {
            // Partial overlap (use circular segment intersection)
            let alpha_j = ((d * d + r_j * r_j - r_wake * r_wake) / (2.0 * d * r_j)).acos();
            let alpha_wake = ((d * d + r_wake * r_wake - r_j * r_j) / (2.0 * d * r_wake)).acos();
            let a_overlap = r_j * r_j * alpha_j + r_wake * r_wake * alpha_wake
                - d * r_j * alpha_j.sin();
            let a_rotor = PI * r_j * r_j;
            (a_overlap / a_rotor).min(1.0).max(0.0)
        } else {
            0.0
        }
    };

    deficit_centerline * overlap_fraction
}

/// Superpose multiple wake deficits using specified method
pub fn superpose_wake_deficits(deficits: &[f64], method: &str) -> f64 {
    match method {
        "linear" => {
            // Linear superposition (sum of deficits, capped at 1.0)
            deficits.iter().sum::<f64>().min(1.0)
        }
        "root_sum_square" => {
            // Root-sum-square (Katic model)
            deficits.iter().map(|d| d * d).sum::<f64>().sqrt().min(1.0)
        }
        "max_deficit" => {
            // Maximum deficit (pessimistic)
            deficits.iter().cloned().fold(0.0, f64::max)
        }
        _ => {
            // Default to linear
            deficits.iter().sum::<f64>().min(1.0)
        }
    }
}

/// Compute AEP for a wind farm given layout and wind resource
pub fn compute_wind_farm_aep(
    turbines: &[TurbineInstance],
    weibull_sectors: &[(f64, f64, f64)],  // (direction_deg, A_scale, k_shape, frequency)
    wind_speed_bins: &[f64],  // [m/s]
    params: &JensenParams,
    superposition_method: &str,
) -> WindFarmResult {
    let n_turbines = turbines.len();
    let mut turbine_aeps = vec![0.0; n_turbines];
    let mut turbine_gross_aeps = vec![0.0; n_turbines];

    // For each wind direction sector
    for &(direction_deg, a_scale, k_shape, freq) in weibull_sectors {
        // For each wind speed bin
        for &wind_speed in wind_speed_bins {
            // Weibull probability density
            let p_wind = weibull_pdf(wind_speed, a_scale, k_shape);
            let hours_per_year = 8760.0 * freq * p_wind;

            // Compute wake deficits for all turbines
            for j in 0..n_turbines {
                // Compute free-stream wind speed at turbine j hub height
                let u_free = wind_speed;  // Assume already at hub height

                // Compute gross power (no wake)
                let p_gross = turbines[j].power_output(u_free);
                turbine_gross_aeps[j] += p_gross * hours_per_year;

                // Compute wake deficits from all upstream turbines
                let mut deficits = Vec::new();
                for i in 0..n_turbines {
                    if i != j {
                        let deficit = jensen_wake_deficit(
                            &turbines[i],
                            &turbines[j],
                            direction_deg,
                            u_free,
                            params,
                        );
                        if deficit > 0.0 {
                            deficits.push(deficit);
                        }
                    }
                }

                // Superpose deficits
                let total_deficit = if deficits.is_empty() {
                    0.0
                } else {
                    superpose_wake_deficits(&deficits, superposition_method)
                };

                // Effective wind speed at turbine j
                let u_eff = u_free * (1.0 - total_deficit);

                // Net power output
                let p_net = turbines[j].power_output(u_eff);
                turbine_aeps[j] += p_net * hours_per_year;
            }
        }
    }

    // Total farm AEP
    let gross_aep_mwh: f64 = turbine_gross_aeps.iter().sum();
    let net_aep_mwh: f64 = turbine_aeps.iter().sum();
    let wake_loss_pct = (1.0 - net_aep_mwh / gross_aep_mwh) * 100.0;

    WindFarmResult {
        gross_aep_gwh: gross_aep_mwh / 1000.0,
        net_aep_gwh: net_aep_mwh / 1000.0,
        wake_loss_pct,
        turbine_aeps_gwh: turbine_aeps.iter().map(|x| x / 1000.0).collect(),
        turbine_gross_aeps_gwh: turbine_gross_aeps.iter().map(|x| x / 1000.0).collect(),
    }
}

#[derive(Debug, Serialize)]
pub struct WindFarmResult {
    pub gross_aep_gwh: f64,
    pub net_aep_gwh: f64,
    pub wake_loss_pct: f64,
    pub turbine_aeps_gwh: Vec<f64>,
    pub turbine_gross_aeps_gwh: Vec<f64>,
}

/// Weibull probability density function
fn weibull_pdf(x: f64, a: f64, k: f64) -> f64 {
    if x <= 0.0 {
        return 0.0;
    }
    (k / a) * (x / a).powf(k - 1.0) * (-(x / a).powf(k)).exp()
}
```

### 3. Layout Optimization — Genetic Algorithm (Rust)

Genetic algorithm for wind farm layout optimization maximizing AEP subject to constraints.

```rust
// optimizer/src/genetic.rs

use rand::prelude::*;
use rayon::prelude::*;
use serde::{Deserialize, Serialize};
use crate::wake::jensen::{compute_wind_farm_aep, TurbineInstance, JensenParams};

#[derive(Debug, Clone, Deserialize)]
pub struct OptimizationConstraints {
    pub site_boundary: Vec<(f64, f64)>,  // Polygon vertices (x, y) [m]
    pub exclusion_zones: Vec<Vec<(f64, f64)>>,  // List of polygons
    pub min_spacing_rotor_diameters: f64,  // e.g., 3.0 or 5.0
    pub max_turbines: usize,
    pub turbine_model: TurbineModel,
}

#[derive(Debug, Clone, Deserialize)]
pub struct TurbineModel {
    pub rotor_diameter: f64,
    pub hub_height: f64,
    pub power_curve: Vec<(f64, f64)>,
    pub thrust_curve: Vec<(f64, f64)>,
}

#[derive(Debug, Clone)]
pub struct Layout {
    pub positions: Vec<(f64, f64)>,  // (x, y) positions [m]
    pub fitness: f64,  // AEP [GWh]
    pub constraint_violation: f64,  // Penalty for constraint violations
}

pub struct GeneticAlgorithm {
    pub population_size: usize,
    pub max_generations: usize,
    pub crossover_rate: f64,
    pub mutation_rate: f64,
    pub elitism_count: usize,
    pub constraints: OptimizationConstraints,
    pub wind_resource: WindResource,
    pub wake_params: JensenParams,
}

#[derive(Debug, Clone)]
pub struct WindResource {
    pub weibull_sectors: Vec<(f64, f64, f64, f64)>,  // (dir, A, k, freq)
    pub wind_speed_bins: Vec<f64>,
}

impl GeneticAlgorithm {
    pub fn optimize(&self) -> OptimizationResult {
        let mut rng = thread_rng();

        // 1. Initialize population
        let mut population = self.initialize_population(&mut rng);

        // Evaluate initial population
        population.par_iter_mut().for_each(|layout| {
            self.evaluate_fitness(layout);
        });

        let mut best_solution = population[0].clone();
        let mut convergence_history = Vec::new();

        // 2. Evolution loop
        for generation in 0..self.max_generations {
            // Sort by fitness (descending)
            population.sort_by(|a, b| {
                let fitness_a = a.fitness - a.constraint_violation * 1000.0;
                let fitness_b = b.fitness - b.constraint_violation * 1000.0;
                fitness_b.partial_cmp(&fitness_a).unwrap()
            });

            // Update best solution
            if population[0].fitness > best_solution.fitness
                && population[0].constraint_violation == 0.0
            {
                best_solution = population[0].clone();
            }

            // Record convergence
            let mean_fitness: f64 = population.iter().map(|l| l.fitness).sum::<f64>()
                / population.len() as f64;
            convergence_history.push((generation, best_solution.fitness, mean_fitness));

            tracing::debug!(
                "Gen {}: best={:.2} GWh, mean={:.2} GWh, violations={}",
                generation,
                population[0].fitness,
                mean_fitness,
                population[0].constraint_violation
            );

            // Check convergence (no improvement in last 20 generations)
            if generation > 20 {
                let recent_best: Vec<f64> = convergence_history
                    .iter()
                    .skip(generation.saturating_sub(20))
                    .map(|(_, best, _)| *best)
                    .collect();
                let improvement = recent_best.last().unwrap() - recent_best.first().unwrap();
                if improvement < 0.1 {
                    tracing::info!("Converged at generation {} (improvement < 0.1 GWh)", generation);
                    break;
                }
            }

            // 3. Create next generation
            let mut next_population = Vec::with_capacity(self.population_size);

            // Elitism: keep top individuals
            for i in 0..self.elitism_count {
                next_population.push(population[i].clone());
            }

            // Breed new individuals
            while next_population.len() < self.population_size {
                // Tournament selection
                let parent1 = self.tournament_select(&population, &mut rng);
                let parent2 = self.tournament_select(&population, &mut rng);

                // Crossover
                let mut offspring = if rng.gen::<f64>() < self.crossover_rate {
                    self.crossover(&parent1, &parent2, &mut rng)
                } else {
                    parent1.clone()
                };

                // Mutation
                if rng.gen::<f64>() < self.mutation_rate {
                    self.mutate(&mut offspring, &mut rng);
                }

                next_population.push(offspring);
            }

            // Evaluate new population
            next_population.par_iter_mut().for_each(|layout| {
                self.evaluate_fitness(layout);
            });

            population = next_population;
        }

        OptimizationResult {
            best_layout: best_solution,
            convergence_history,
        }
    }

    fn initialize_population(&self, rng: &mut ThreadRng) -> Vec<Layout> {
        (0..self.population_size)
            .map(|_| self.random_layout(rng))
            .collect()
    }

    fn random_layout(&self, rng: &mut ThreadRng) -> Layout {
        let n_turbines = self.constraints.max_turbines;
        let (x_min, x_max, y_min, y_max) = self.get_bounding_box();

        let mut positions = Vec::new();
        let mut attempts = 0;
        while positions.len() < n_turbines && attempts < n_turbines * 100 {
            let x = rng.gen_range(x_min..x_max);
            let y = rng.gen_range(y_min..y_max);

            if self.is_valid_position(x, y, &positions) {
                positions.push((x, y));
            }
            attempts += 1;
        }

        Layout {
            positions,
            fitness: 0.0,
            constraint_violation: 0.0,
        }
    }

    fn is_valid_position(&self, x: f64, y: f64, existing: &[(f64, f64)]) -> bool {
        // Check site boundary
        if !point_in_polygon(x, y, &self.constraints.site_boundary) {
            return false;
        }

        // Check exclusion zones
        for zone in &self.constraints.exclusion_zones {
            if point_in_polygon(x, y, zone) {
                return false;
            }
        }

        // Check minimum spacing
        let min_dist = self.constraints.min_spacing_rotor_diameters
            * self.constraints.turbine_model.rotor_diameter;
        for &(ex_x, ex_y) in existing {
            let dist = ((x - ex_x).powi(2) + (y - ex_y).powi(2)).sqrt();
            if dist < min_dist {
                return false;
            }
        }

        true
    }

    fn evaluate_fitness(&self, layout: &mut Layout) {
        // Build turbine instances
        let turbines: Vec<TurbineInstance> = layout
            .positions
            .iter()
            .map(|&(x, y)| TurbineInstance {
                x,
                y,
                z_hub: self.constraints.turbine_model.hub_height,
                rotor_diameter: self.constraints.turbine_model.rotor_diameter,
                power_curve: self.constraints.turbine_model.power_curve.clone(),
                thrust_curve: self.constraints.turbine_model.thrust_curve.clone(),
            })
            .collect();

        // Compute AEP
        let result = compute_wind_farm_aep(
            &turbines,
            &self.wind_resource.weibull_sectors,
            &self.wind_resource.wind_speed_bins,
            &self.wake_params,
            "root_sum_square",
        );

        layout.fitness = result.net_aep_gwh;

        // Compute constraint violations
        layout.constraint_violation = 0.0;
        for i in 0..layout.positions.len() {
            let (x, y) = layout.positions[i];
            // Boundary violation
            if !point_in_polygon(x, y, &self.constraints.site_boundary) {
                layout.constraint_violation += 1.0;
            }
            // Spacing violation
            for j in (i + 1)..layout.positions.len() {
                let (x2, y2) = layout.positions[j];
                let dist = ((x - x2).powi(2) + (y - y2).powi(2)).sqrt();
                let min_dist = self.constraints.min_spacing_rotor_diameters
                    * self.constraints.turbine_model.rotor_diameter;
                if dist < min_dist {
                    layout.constraint_violation += (min_dist - dist) / min_dist;
                }
            }
        }
    }

    fn tournament_select(&self, population: &[Layout], rng: &mut ThreadRng) -> Layout {
        let tournament_size = 5;
        let mut best: Option<&Layout> = None;
        for _ in 0..tournament_size {
            let candidate = &population[rng.gen_range(0..population.len())];
            let candidate_score = candidate.fitness - candidate.constraint_violation * 1000.0;
            if best.is_none()
                || candidate_score
                    > best.unwrap().fitness - best.unwrap().constraint_violation * 1000.0
            {
                best = Some(candidate);
            }
        }
        best.unwrap().clone()
    }

    fn crossover(&self, parent1: &Layout, parent2: &Layout, rng: &mut ThreadRng) -> Layout {
        // Uniform crossover: each turbine position comes from either parent
        let n = parent1.positions.len().min(parent2.positions.len());
        let mut positions = Vec::new();
        for i in 0..n {
            if rng.gen_bool(0.5) {
                positions.push(parent1.positions[i]);
            } else {
                positions.push(parent2.positions[i]);
            }
        }
        Layout {
            positions,
            fitness: 0.0,
            constraint_violation: 0.0,
        }
    }

    fn mutate(&self, layout: &mut Layout, rng: &mut ThreadRng) {
        // Randomly perturb one or more turbine positions
        let n_mutations = rng.gen_range(1..=3);
        for _ in 0..n_mutations {
            if layout.positions.is_empty() {
                break;
            }
            let idx = rng.gen_range(0..layout.positions.len());
            let (x, y) = layout.positions[idx];
            let sigma = 100.0;  // Mutation strength [m]
            let new_x = x + rng.gen_range(-sigma..sigma);
            let new_y = y + rng.gen_range(-sigma..sigma);
            layout.positions[idx] = (new_x, new_y);
        }
    }

    fn get_bounding_box(&self) -> (f64, f64, f64, f64) {
        let xs: Vec<f64> = self.constraints.site_boundary.iter().map(|p| p.0).collect();
        let ys: Vec<f64> = self.constraints.site_boundary.iter().map(|p| p.1).collect();
        let x_min = xs.iter().cloned().fold(f64::INFINITY, f64::min);
        let x_max = xs.iter().cloned().fold(f64::NEG_INFINITY, f64::max);
        let y_min = ys.iter().cloned().fold(f64::INFINITY, f64::min);
        let y_max = ys.iter().cloned().fold(f64::NEG_INFINITY, f64::max);
        (x_min, x_max, y_min, y_max)
    }
}

#[derive(Debug, Serialize)]
pub struct OptimizationResult {
    pub best_layout: Layout,
    pub convergence_history: Vec<(usize, f64, f64)>,  // (generation, best_fitness, mean_fitness)
}

/// Point-in-polygon test using ray casting algorithm
fn point_in_polygon(x: f64, y: f64, polygon: &[(f64, f64)]) -> bool {
    let n = polygon.len();
    let mut inside = false;
    let mut j = n - 1;
    for i in 0..n {
        let (xi, yi) = polygon[i];
        let (xj, yj) = polygon[j];
        if ((yi > y) != (yj > y)) && (x < (xj - xi) * (y - yi) / (yj - yi) + xi) {
            inside = !inside;
        }
        j = i;
    }
    inside
}
```

### 4. Map Visualization Component (React + Mapbox GL + Deck.gl)

Interactive 3D map for turbine layout editing with wake plume visualization.

```typescript
// frontend/src/components/MapView.tsx

import { useRef, useEffect, useCallback } from 'react';
import mapboxgl from 'mapbox-gl';
import { Deck } from '@deck.gl/core';
import { ScatterplotLayer, PathLayer, PolygonLayer } from '@deck.gl/layers';
import { useProjectStore } from '../stores/projectStore';
import { useTurbineStore } from '../stores/turbineStore';
import { useWakeStore } from '../stores/wakeStore';

mapboxgl.accessToken = import.meta.env.VITE_MAPBOX_TOKEN;

interface MapViewProps {
  projectId: string;
}

export function MapView({ projectId }: MapViewProps) {
  const mapContainerRef = useRef<HTMLDivElement>(null);
  const mapRef = useRef<mapboxgl.Map | null>(null);
  const deckRef = useRef<Deck | null>(null);

  const { project } = useProjectStore();
  const { turbines, addTurbine, moveTurbine, selectedTurbineId } = useTurbineStore();
  const { wakePlumes, computeWakes } = useWakeStore();

  // Initialize map
  useEffect(() => {
    if (!mapContainerRef.current || !project) return;

    const map = new mapboxgl.Map({
      container: mapContainerRef.current,
      style: 'mapbox://styles/mapbox/satellite-streets-v12',
      center: [project.location_lon, project.location_lat],
      zoom: 13,
      pitch: 45,  // 3D tilt
      bearing: 0,
      terrain: { source: 'mapbox-dem', exaggeration: 1.5 },
    });

    map.on('load', () => {
      // Add terrain DEM source
      map.addSource('mapbox-dem', {
        type: 'raster-dem',
        url: 'mapbox://mapbox.terrain-rgb',
        tileSize: 512,
        maxzoom: 14,
      });

      // Add 3D buildings (optional)
      map.addLayer({
        id: '3d-buildings',
        source: 'composite',
        'source-layer': 'building',
        filter: ['==', 'extrude', 'true'],
        type: 'fill-extrusion',
        minzoom: 15,
        paint: {
          'fill-extrusion-color': '#aaa',
          'fill-extrusion-height': ['get', 'height'],
          'fill-extrusion-base': ['get', 'min_height'],
          'fill-extrusion-opacity': 0.6,
        },
      });
    });

    mapRef.current = map;

    // Initialize Deck.gl overlay
    const deck = new Deck({
      canvas: 'deck-canvas',
      width: '100%',
      height: '100%',
      initialViewState: {
        longitude: project.location_lon,
        latitude: project.location_lat,
        zoom: 13,
        pitch: 45,
        bearing: 0,
      },
      controller: true,
      onViewStateChange: ({ viewState }) => {
        map.jumpTo({
          center: [viewState.longitude, viewState.latitude],
          zoom: viewState.zoom,
          bearing: viewState.bearing,
          pitch: viewState.pitch,
        });
      },
      layers: [],
    });

    deckRef.current = deck;

    return () => {
      map.remove();
      deck.finalize();
    };
  }, [project]);

  // Update Deck.gl layers when turbines or wakes change
  useEffect(() => {
    if (!deckRef.current) return;

    const layers = [
      // Site boundary polygon
      new PolygonLayer({
        id: 'site-boundary',
        data: project?.site_boundary ? [project.site_boundary] : [],
        getPolygon: (d: any) => d.coordinates[0],
        getFillColor: [0, 150, 255, 30],
        getLineColor: [0, 150, 255, 200],
        getLineWidth: 3,
        lineWidthMinPixels: 2,
      }),

      // Turbine positions
      new ScatterplotLayer({
        id: 'turbines',
        data: turbines,
        getPosition: (d: any) => [d.longitude, d.latitude, d.elevation_m || 0],
        getRadius: (d: any) => d.rotor_diameter_m / 2 || 45,  // Approximate
        getFillColor: (d: any) =>
          d.id === selectedTurbineId ? [255, 200, 0, 255] : [255, 255, 255, 200],
        getLineColor: [0, 0, 0, 255],
        lineWidthMinPixels: 2,
        pickable: true,
        onClick: (info) => {
          if (info.object) {
            useTurbineStore.getState().selectTurbine(info.object.id);
          }
        },
        onDragStart: (info) => {
          if (info.object) {
            useTurbineStore.getState().setDragging(true);
          }
        },
        onDrag: (info) => {
          if (info.object && info.coordinate) {
            moveTurbine(info.object.id, info.coordinate[0], info.coordinate[1]);
          }
        },
        onDragEnd: () => {
          useTurbineStore.getState().setDragging(false);
          computeWakes();  // Recompute wakes after drag
        },
      }),

      // Wake plumes (simplified as polygons for MVP)
      new PolygonLayer({
        id: 'wake-plumes',
        data: wakePlumes,
        getPolygon: (d: any) => d.polygon,
        getFillColor: (d: any) => [255, 100, 100, d.deficit * 100],  // Opacity by deficit
        getLineColor: [255, 0, 0, 100],
        getLineWidth: 1,
        lineWidthMinPixels: 1,
      }),
    ];

    deckRef.current.setProps({ layers });
  }, [turbines, wakePlumes, selectedTurbineId, project]);

  // Add turbine on map click (when in placement mode)
  const handleMapClick = useCallback((e: mapboxgl.MapMouseEvent) => {
    const placementMode = useTurbineStore.getState().placementMode;
    if (placementMode && mapRef.current) {
      const { lng, lat } = e.lngLat;
      addTurbine({
        name: `T${turbines.length + 1}`,
        latitude: lat,
        longitude: lng,
        turbine_model_id: useTurbineStore.getState().selectedModelId,
      });
      computeWakes();
    }
  }, [turbines, addTurbine, computeWakes]);

  useEffect(() => {
    const map = mapRef.current;
    if (!map) return;
    map.on('click', handleMapClick);
    return () => {
      map.off('click', handleMapClick);
    };
  }, [handleMapClick]);

  return (
    <div className="relative w-full h-full">
      <div ref={mapContainerRef} className="absolute inset-0" />
      <canvas id="deck-canvas" className="absolute inset-0 pointer-events-none" />

      {/* Map controls overlay */}
      <div className="absolute top-4 left-4 bg-white rounded-lg shadow-lg p-4 space-y-2">
        <button
          className="btn-primary"
          onClick={() => useTurbineStore.getState().togglePlacementMode()}
        >
          Add Turbine
        </button>
        <button className="btn-secondary" onClick={computeWakes}>
          Compute Wakes
        </button>
      </div>

      {/* Turbine info panel */}
      {selectedTurbineId && (
        <div className="absolute bottom-4 left-4 bg-white rounded-lg shadow-lg p-4 max-w-sm">
          <TurbineInfoPanel turbineId={selectedTurbineId} />
        </div>
      )}
    </div>
  );
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–5)

**Day 1: Project initialization and Rust backend scaffold**
```bash
cargo init windfarm-api
cd windfarm-api
cargo add axum tokio serde serde_json sqlx uuid chrono tower-http jsonwebtoken bcrypt
cargo add --features postgres,runtime-tokio-rustls,uuid,chrono,json,time sqlx
cargo add aws-sdk-s3 redis
```
- `src/main.rs` — Axum app with CORS, tracing, graceful shutdown
- `src/config.rs` — Environment-based configuration (DATABASE_URL, JWT_SECRET, S3_BUCKET, REDIS_URL, MAPBOX_TOKEN)
- `src/state.rs` — AppState struct (PgPool, Redis, S3Client, terrain_cache)
- `src/error.rs` — ApiError enum with IntoResponse
- `Dockerfile` — Multi-stage build (builder + runtime)
- `docker-compose.yml` — PostgreSQL + TimescaleDB + PostGIS, Redis, MinIO (S3-compatible)

**Day 2: Database schema and migrations**
- `migrations/001_initial.sql` — All 10 tables: users, organizations, org_members, projects, turbine_models, turbines, met_stations, met_data (hypertable), wind_resources, simulations, optimization_jobs, usage_records
- Enable PostGIS and TimescaleDB extensions
- `src/db/mod.rs` — Database pool initialization with connection pooling
- `src/db/models.rs` — All SQLx structs with FromRow derives
- Run `sqlx migrate run` to apply schema
- Seed script: insert 10 turbine models (Vestas V90/V117/V150, Siemens Gamesa 4.5/5.0/8.0, GE 2.5/3.0/5.3) with real power/thrust curves

**Day 3: Authentication system**
- `src/auth/mod.rs` — JWT token generation and validation middleware
- `src/auth/oauth.rs` — Google OAuth 2.0 flow handlers
- `src/api/handlers/auth.rs` — Register, login, OAuth callback, refresh token, me endpoints
- Password hashing with bcrypt (cost factor 12)
- JWT with 24h access token + 30d refresh token
- Auth middleware that extracts Claims from Authorization header

**Day 4: User and project CRUD**
- `src/api/handlers/users.rs` — Get profile, update profile, delete account
- `src/api/handlers/projects.rs` — Create (with lat/lon + site boundary polygon), list, get, update, delete, fork project
- `src/api/handlers/orgs.rs` — Create org, invite member, list members, remove member
- `src/api/router.rs` — All route definitions with auth middleware
- Integration tests: auth flow, project CRUD, authorization checks

**Day 5: Turbine library and spatial queries**
- `src/api/handlers/turbines.rs` — Add turbine to project, update position, delete, list all turbines in project
- `src/api/handlers/turbine_models.rs` — List turbine models (with filters: manufacturer, power range, rotor diameter), get model details
- PostGIS spatial queries: find turbines within radius, check site boundary containment
- Turbine position validation: check site boundary, minimum spacing
- Integration tests: turbine CRUD, spatial constraint checks

### Phase 2 — Flow + Wake Solvers — Prototype (Days 6–14)

**Day 6: Flow model framework (linearized WAsP-like)**
- `flow-core/` — New Rust workspace member (shared between native and WASM)
- `flow-core/src/lib.rs` — Public API: `compute_speedup_map`, `compute_shear_profile`
- `flow-core/src/terrain.rs` — Terrain elevation interpolation, gradient computation, FFT grid setup
- `flow-core/src/bz_model.rs` — BZ orographic speedup model (2D FFT-based linearized flow)
- `flow-core/src/ibz_model.rs` — IBZ internal boundary layer for roughness change
- Unit tests: flat terrain (speedup = 1.0), simple hill (compare to analytical solution)

**Day 7: Terrain data ingestion and caching**
- `src/terrain/mod.rs` — SRTM 30m data downloader (NASA EarthData API)
- `src/terrain/cache.rs` — Local disk cache for terrain tiles (5°×5° tiles)
- `src/terrain/rix.rs` — Ruggedness Index (RIX) computation: % of terrain with slope >30% within radius
- Terrain preprocessing: convert SRTM GeoTIFF → binary grid for fast access
- S3 storage: preprocessed terrain tiles for CDN delivery to frontend
- Integration: compute_rix() function used in simulation routing decision

**Day 8: Roughness maps and wind shear**
- `flow-core/src/roughness.rs` — ESA WorldCover landcover data → roughness length z₀ mapping
- Roughness lookup table: water=0.0002, grass=0.03, crops=0.10, forest=0.50 m
- Wind shear profile: logarithmic law and power law implementations
- Hub-height extrapolation: from met mast measurement height to turbine hub height
- Unit tests: shear exponent verification, hub-height wind speed

**Day 9: Jensen wake model core**
- `wake-core/` — New Rust workspace member (shared between native and WASM)
- `wake-core/src/jensen.rs` — Full Jensen wake model implementation (as shown in Architecture Deep-Dives section)
- Wake deficit computation with partial overlap via circular segment area
- Wake superposition: linear, root-sum-square, max deficit
- Thrust coefficient and power curve interpolation
- Unit tests: single wake, multi-wake, full array (2×3 grid), verify wake loss 8-12%

**Day 10: AEP calculation engine**
- `wake-core/src/aep.rs` — AEP computation given turbines, wind resource (Weibull sectors), wake model
- Weibull PDF evaluation and integration over wind speed bins
- Gross AEP (no wake), net AEP (with wake), wake loss percentage
- Per-turbine AEP breakdown
- Integration tests: simple 5-turbine array, verify total AEP within 5% of hand calculation

**Day 11: Weibull fitting service (Python)**
- `wind-service/` — Python FastAPI microservice
- `wind-service/weibull.py` — Fit Weibull distribution (A, k) per direction sector using MLE
- `wind-service/qc.py` — Met data quality control: detect icing, stuck sensors, tower shadow, timestamp gaps
- `wind-service/wind_rose.py` — Generate wind rose data (frequency + mean speed per sector)
- REST API: POST /weibull-fit, POST /qc-report
- Integration: Rust API calls Python service via HTTP

**Day 12: Met data upload and processing**
- `src/api/handlers/met_data.rs` — Upload CSV met data, trigger QC and Weibull fitting
- CSV parser: flexible column mapping (timestamp, wind_speed, wind_direction, temperature, TI)
- Insert into TimescaleDB met_data table (batch insert for performance)
- Background job: run QC → fit Weibull → create wind_resource record
- S3 storage: original CSV file and QC report PDF

**Day 13: Wind resource API**
- `src/api/handlers/wind_resources.rs` — List wind resources for project, get resource details, create manual resource
- Wind resource visualization data: Weibull parameters per sector, mean wind speed, power density
- Integration: simulation endpoint fetches wind resource by ID

**Day 14: Flow+wake integration testing**
- End-to-end test: terrain speedup + wake computation → AEP
- Benchmark: 10-turbine array on 2% slope, verify AEP within 5% of reference tool (WAsP + PyWake)
- Performance: measure flow model time (target <500ms for 500×500 grid), wake model time (target <50ms for 10 turbines)
- Edge cases: extreme terrain slope (50%), very close turbine spacing (2D), high wake losses (>20%)

### Phase 3 — WASM Build + Frontend Visualization (Days 15–20)

**Day 15: WASM flow+wake compilation**
- `flow-wasm/` — New workspace member for WASM target
- `flow-wasm/Cargo.toml` — wasm-bindgen, serde-wasm-bindgen, rustfft dependencies
- `flow-wasm/src/lib.rs` — WASM entry points: `compute_aep_wasm()`, `compute_wakes_wasm()`
- `wasm-pack build --target web --release` pipeline
- `wasm-opt -Oz` for size optimization (target <1.5MB gzipped)
- JavaScript wrapper: `WindSolver` class that loads WASM and provides async API

**Day 16: Frontend scaffold and Mapbox integration**
```bash
npm create vite@latest frontend -- --template react-ts
cd frontend
npm i zustand @tanstack/react-query axios mapbox-gl @deck.gl/core @deck.gl/layers d3
```
- `src/App.tsx` — Router, auth context, layout
- `src/stores/projectStore.ts` — Zustand store for project state
- `src/stores/turbineStore.ts` — Turbine positions, selection, drag state
- `src/stores/wakeStore.ts` — Wake visualization state
- `src/components/MapView.tsx` — Mapbox GL map with 3D terrain (as shown in Architecture Deep-Dives section)
- Mapbox token configuration via environment variable

**Day 17: Turbine placement and dragging**
- Click-to-place turbine mode with model selection dropdown
- Drag-and-drop turbine repositioning with spatial constraint validation
- Real-time wake recalculation on drag (debounced 300ms)
- Turbine marker rendering: custom SVG icon (turbine symbol) with label
- Site boundary polygon drawing tool (click to add vertices, close polygon)
- Exclusion zone drawing (multiple polygons per project)

**Day 18: Wake visualization**
- `src/components/WakeVisualization.tsx` — Wake plume rendering as semi-transparent cones
- Color coding: velocity deficit 0-30% → gradient from yellow to red
- Toggle wake visualization on/off
- Wake cross-section view (side panel): show velocity profile downstream of turbine
- WebGL optimization: batch render wake plumes for 25+ turbines

**Day 19: Wind rose and charts (D3.js)**
- `src/components/WindRose.tsx` — Polar wind rose diagram (frequency + speed per sector)
- `src/components/EnergyWaterfall.tsx` — Stacked bar chart showing loss waterfall (gross → wake → availability → electrical → net)
- `src/components/AEPChart.tsx` — Bar chart per turbine showing gross vs. net AEP
- D3.js integration: responsive SVG charts with tooltips

**Day 20: Frontend state management and WASM integration**
- `src/hooks/useWasmSolver.ts` — React hook for loading WASM and running simulations
- `src/lib/wasmLoader.ts` — Lazy load WASM bundle, cache in IndexedDB
- Zustand stores: persist turbine positions and settings to localStorage
- Optimistic UI updates: instant turbine placement, background wake computation
- Error handling: show toast notifications for WASM errors or API failures

### Phase 4 — API + Job Orchestration (Days 21–26)

**Day 21: Simulation API endpoints**
- `src/api/handlers/simulation.rs` — Create simulation (with WASM/server routing logic as shown in Architecture Deep-Dives section), get simulation, list simulations, cancel simulation
- Layout snapshot: serialize turbine positions and models to JSONB
- Node count → turbine count, RIX computation for routing
- Plan-based limits enforcement (free: 10 turbines, 100 simulations/month)

**Day 22: Server-side simulation worker**
- `src/workers/simulation_worker.rs` — Redis job consumer, runs native flow+wake solver
- `src/workers/mod.rs` — Worker pool management (configurable concurrency, Tokio tasks)
- Progress streaming: worker publishes progress → Redis pub/sub → WebSocket to client
- Error handling: convergence failures, invalid layouts, out-of-memory
- S3 result upload: detailed AEP results CSV + summary JSON

**Day 23: WebSocket for live simulation progress**
- `src/api/ws/mod.rs` — Axum WebSocket upgrade handler
- `src/api/ws/simulation_progress.rs` — Subscribe to simulation/optimization progress channel
- Client receives: `{ status, progress_pct, current_iteration, best_aep_gwh }`
- Connection management: heartbeat ping/pong every 30s, automatic cleanup on disconnect
- Frontend: `src/hooks/useSimulationProgress.ts` — React hook for WebSocket subscription

**Day 24: Layout optimization worker**
- `src/workers/optimization_worker.rs` — Redis job consumer, runs genetic algorithm (as shown in Architecture Deep-Dives section)
- Population initialization: random layouts satisfying constraints
- Fitness evaluation: parallel AEP computation for all layouts in population
- Progress updates: every 10 generations, publish best/mean fitness to WebSocket
- Convergence detection: stop if no improvement for 30 generations
- Result: best layout + convergence history stored in optimization_jobs table

**Day 25: Optimization API**
- `src/api/handlers/optimization.rs` — Start optimization, get optimization status, get convergence history, apply optimized layout
- Constraint specification: min_spacing (rotor diameters), site boundary, exclusion zones, max turbines
- Objective: maximize net AEP (post-MVP: multi-objective with cable cost, noise, visual impact)
- Frontend: optimization modal with constraint inputs and real-time progress chart

**Day 26: Report generation (PDF)**
- `report-service/` — Python FastAPI microservice with ReportLab for PDF generation
- `report-service/templates/yield_report.py` — Energy yield report template:
  - Project summary (location, turbine count, total capacity)
  - Wind resource summary (mean wind speed, Weibull params, wind rose diagram)
  - Layout map (satellite imagery + turbine positions + wakes)
  - Energy yield summary (gross/net AEP, capacity factor, wake loss)
  - Loss waterfall chart
  - Per-turbine AEP table
- REST API: POST /generate-report → returns PDF
- S3 storage: generated reports with presigned URLs for download
- Frontend: "Download Report" button triggers report generation

### Phase 5 — Results + Post-Processing UI (Days 27–31)

**Day 27: Simulation results display**
- `src/pages/Results.tsx` — Results page showing AEP, capacity factor, wake loss, turbine table
- Turbine table: sortable columns (name, position, gross AEP, net AEP, capacity factor, wake loss)
- Map integration: click turbine in table → highlight on map, click turbine on map → scroll to table row
- Export results: CSV download (turbine-level data), JSON (full results)

**Day 28: Layout comparison**
- `src/components/LayoutComparison.tsx` — Side-by-side comparison of multiple layouts (manual vs. optimized, different turbine models)
- Comparison table: layout A vs. layout B (turbine count, total capacity, AEP, wake loss, CF)
- Map overlay: show both layouts with different colors
- Scenario management: save named scenarios (e.g., "Baseline 50 turbines", "Optimized 60 turbines")

**Day 29: Advanced wake visualization**
- Wake streamlines: show wind flow paths through array
- Wake deficit heatmap: 2D grid colored by velocity deficit
- Time-series animation: show wake movement with varying wind direction (16 sectors)
- Cross-section plot: velocity profile at Y=const slice through array

**Day 30: Turbine suitability check**
- `src/services/turbine_suitability.rs` — IEC 61400-1 wind class check
- Compute site wind speed parameters: V_ave, V_50year (extreme), turbulence intensity
- Compare to turbine design envelope: IEC Class I/II/III, Turbulence Class A/B/C
- Warning if turbine is undersized or oversized for site
- Frontend: suitability badge on turbine model cards (green=suitable, yellow=marginal, red=not suitable)

**Day 31: Energy yield sensitivity analysis**
- `src/components/SensitivityAnalysis.tsx` — Tornado chart showing AEP sensitivity to input parameters
- Vary: mean wind speed ±10%, wake decay constant ±20%, availability ±2%, air density ±5%
- Recompute AEP for each variation (9 simulations: baseline + 4 params × 2 directions)
- Display: sorted bar chart showing impact on AEP (GWh or %)
- Post-MVP: Monte Carlo uncertainty analysis for P50/P75/P90 estimates

### Phase 6 — Billing + Plan Enforcement (Days 32–35)

**Day 32: Stripe integration**
- `src/api/handlers/billing.rs` — Create checkout session, create customer portal session, get subscription status
- `src/api/handlers/webhooks/stripe.rs` — Handle subscription events: `checkout.session.completed`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.payment_failed`
- Plan mapping:
  - Free: 10 turbines, 2 projects, 100 simulations/month, Jensen wake, basic report
  - Pro ($199/mo): Unlimited turbines, 20 projects, unlimited simulations, optimization, advanced report, priority support
  - Advanced ($499/user/mo): CFD flow (post-MVP), Bastankhah wake (post-MVP), team collaboration (5 seats), API access, white-label reports

**Day 33: Usage tracking and plan limits**
- `src/middleware/plan_limits.rs` — Middleware checking plan limits before simulation/optimization execution
- `src/services/usage.rs` — Track simulations per billing period, optimization runs
- Usage record insertion after each simulation completes
- Usage dashboard: `src/api/handlers/usage.rs` — Current period usage, historical usage
- Approaching-limit warnings at 80% and 100% of monthly quota

**Day 34: Billing UI**
- `frontend/src/pages/Billing.tsx` — Plan comparison table, current plan display, usage meter
- `frontend/src/components/billing/PlanCard.tsx` — Plan feature comparison cards
- `frontend/src/components/billing/UsageMeter.tsx` — Simulation quota usage bar with threshold indicators
- Upgrade/downgrade flow via Stripe Customer Portal (opens in new tab)
- Upgrade prompt modals when hitting plan limits (e.g., "Layout has 15 turbines. Upgrade to Pro for unlimited.")

**Day 35: Feature gating**
- Gate optimization behind Pro plan (show "Optimize Layout" button with upgrade prompt for Free users)
- Gate advanced reports (P50/P75/P90, uncertainty) behind Advanced plan
- Gate API access behind Advanced plan (show API docs with upgrade CTA)
- Gate team collaboration features behind Advanced plan
- Admin endpoint: override plan for specific users (beta testers, partners, academics)

### Phase 7 — Solver Validation + Testing (Days 36–39)

**Day 36: Wake model validation benchmarks**
- Benchmark 1: Single turbine in uniform flow → verify no wake at turbine location (deficit = 0)
- Benchmark 2: 2 turbines aligned with wind direction, 7D spacing → verify wake deficit 20-30% at downstream turbine
- Benchmark 3: 3×3 turbine grid, uniform wind → verify total wake loss 10-15%
- Benchmark 4: Compare to PyWake reference simulations (same layout, wind resource) → verify AEP within 3%
- Benchmark 5: Horns Rev 1 offshore wind farm (80 turbines, real data) → compare wake losses to published measurements (8.5-9.5%)

**Day 37: Optimization validation**
- Benchmark 1: Simple square site, 10 turbines, uniform wind → verify optimizer finds regular grid with ~7D spacing
- Benchmark 2: Elongated site, dominant wind direction → verify optimizer aligns turbines perpendicular to dominant wind
- Benchmark 3: Site with exclusion zone in center → verify optimizer routes around constraint
- Benchmark 4: Compare GA optimizer to brute-force search on small problem (5 turbines, 10×10 grid) → verify global optimum found
- Performance: measure optimization time vs. turbine count (target <5 min for 25 turbines, 200 generations)

**Day 38: Integration testing**
- End-to-end test: register → create project → upload met data → place turbines → run simulation → view results → download report
- API integration tests: auth → project CRUD → met data upload → simulation → optimization → results
- WASM solver test: load in headless browser (Playwright), run simulation, verify results match native solver
- WebSocket test: connect → subscribe to optimization → receive progress updates → receive completion
- Concurrent users test: 10 users simultaneously running optimizations

**Day 39: Performance testing and optimization**
- WASM solver benchmarks: measure time for 10, 15, 20, 25 turbines in Chrome/Firefox/Safari
- Target: 25-turbine AEP computation < 200ms in WASM
- Server solver benchmarks: measure throughput (simulations/minute) with various turbine counts
- Target: 100-turbine simulation < 5s on server (4-core VM)
- Frontend rendering: measure map FPS with 100 turbines + wake plumes
- Target: 60 FPS during pan/zoom/rotate
- Load testing: 50 concurrent users running simulations + optimizations via k6 or Locust

### Phase 8 — Deployment + Polish + Launch (Days 40–42)

**Day 40: Docker and Kubernetes configuration**
- `Dockerfile.api` — Multi-stage Rust build (cargo-chef for layer caching)
- `Dockerfile.worker` — Separate image for simulation/optimization workers
- `docker-compose.yml` — Full local dev stack (API, workers, PostgreSQL+TimescaleDB+PostGIS, Redis, MinIO, Python services)
- `k8s/` — Kubernetes manifests:
  - `api-deployment.yaml` — API server (3 replicas, HPA based on CPU)
  - `worker-deployment.yaml` — Workers (auto-scaling based on Redis queue depth)
  - `postgres-statefulset.yaml` — PostgreSQL with PVC
  - `redis-deployment.yaml` — Redis for job queue and pub/sub
  - `python-services-deployment.yaml` — Python FastAPI (Weibull + report services)
  - `ingress.yaml` — NGINX ingress with TLS (cert-manager + Let's Encrypt)
- Health check endpoints: `/health/live`, `/health/ready`

**Day 41: CDN and WASM delivery**
- CloudFront distribution for:
  - Frontend static assets (HTML, JS, CSS) — cached at edge, 1-year TTL
  - WASM flow+wake bundle (~1.5MB) — cached at edge with long TTL and versioned URLs (e.g., `/wasm/v1.2.3/`)
  - Terrain tiles from S3 — cached with 1-day TTL
  - Generated reports — cached with 1-hour TTL
- WASM preloading: start downloading WASM bundle on page load before user needs it
- Service worker for offline WASM caching (progressive enhancement)
- Gzip/Brotli compression for all text assets

**Day 42: Monitoring, logging, polish, and launch**
- Prometheus metrics: simulation duration histogram, optimization convergence rate, API latency percentiles, queue depth
- Grafana dashboards: system health, simulation throughput, user activity, AEP distribution
- Sentry integration: frontend and backend error tracking with source maps and breadcrumbs
- Structured logging (tracing + JSON) for all API requests, simulation events, optimization progress
- UI polish: loading states (spinners, skeleton screens), error messages, empty states, responsive layout (mobile/tablet/desktop)
- Accessibility: keyboard navigation on map, ARIA labels on interactive elements, screen reader support
- Landing page: hero section with demo video, feature overview, pricing comparison, testimonials, signup CTA
- Documentation: getting started guide, video tutorials, API reference, FAQ
- Deploy to production: run smoke tests, enable monitoring alerts (Slack/PagerDuty)
- Announce launch: Product Hunt, Hacker News, LinkedIn, wind energy forums (Wind Energy Network, AWEA)

---

## Critical Files

```
windfarm/
├── flow-core/                             # Linearized flow model (native + WASM)
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── terrain.rs                     # Elevation interpolation, gradient, FFT setup
│   │   ├── bz_model.rs                    # BZ orographic speedup (FFT-based)
│   │   ├── ibz_model.rs                   # IBZ internal boundary layer
│   │   ├── roughness.rs                   # Roughness length mapping, shear profile
│   │   └── weibull.rs                     # Weibull distribution utilities
│   └── tests/
│       ├── flow_benchmarks.rs             # Flat/hill validation, speedup accuracy
│       └── shear_validation.rs            # Wind shear profile tests

├── wake-core/                             # Jensen wake model (native + WASM)
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── jensen.rs                      # Full Jensen wake implementation
│   │   ├── aep.rs                         # AEP computation engine
│   │   └── superposition.rs               # Wake deficit superposition methods
│   └── tests/
│       ├── wake_benchmarks.rs             # Single/multi-wake validation
│       ├── aep_validation.rs              # AEP vs. reference tools
│       └── horns_rev_validation.rs        # Real wind farm benchmark

├── optimizer/                             # Genetic algorithm layout optimizer
│   ├── Cargo.toml
│   └── src/
│       ├── lib.rs
│       ├── genetic.rs                     # GA implementation (as shown earlier)
│       ├── constraints.rs                 # Spatial constraints (boundary, spacing, exclusions)
│       └── fitness.rs                     # AEP fitness evaluation with parallelization

├── flow-wasm/                             # WASM compilation target
│   ├── Cargo.toml
│   └── src/
│       └── lib.rs                         # WASM entry points (wasm-bindgen)

├── windfarm-api/                          # Rust API server
│   ├── Cargo.toml
│   ├── src/
│   │   ├── main.rs                        # Axum app setup, router, startup
│   │   ├── config.rs                      # Environment config
│   │   ├── state.rs                       # AppState (PgPool, Redis, S3, terrain_cache)
│   │   ├── error.rs                       # ApiError types
│   │   ├── auth/
│   │   │   ├── mod.rs                     # JWT middleware, Claims
│   │   │   └── oauth.rs                   # Google OAuth handlers
│   │   ├── api/
│   │   │   ├── router.rs                  # Route definitions
│   │   │   ├── handlers/
│   │   │   │   ├── auth.rs                # Register, login, OAuth
│   │   │   │   ├── users.rs               # Profile CRUD
│   │   │   │   ├── projects.rs            # Project CRUD, fork, share
│   │   │   │   ├── turbines.rs            # Turbine CRUD, spatial validation
│   │   │   │   ├── turbine_models.rs      # Turbine library search/get
│   │   │   │   ├── met_data.rs            # Upload CSV, trigger QC/Weibull
│   │   │   │   ├── wind_resources.rs      # List/get wind resources
│   │   │   │   ├── simulation.rs          # Create/get/cancel simulation
│   │   │   │   ├── optimization.rs        # Start/status/apply optimization
│   │   │   │   ├── billing.rs             # Stripe checkout/portal
│   │   │   │   ├── usage.rs               # Usage tracking
│   │   │   │   └── webhooks/
│   │   │   │       └── stripe.rs          # Stripe webhook handler
│   │   │   └── ws/
│   │   │       ├── mod.rs                 # WebSocket upgrade handler
│   │   │       └── progress.rs            # Live simulation/optimization progress
│   │   ├── middleware/
│   │   │   └── plan_limits.rs             # Plan enforcement middleware
│   │   ├── services/
│   │   │   ├── usage.rs                   # Usage tracking service
│   │   │   └── s3.rs                      # S3 client helpers
│   │   ├── terrain/
│   │   │   ├── mod.rs                     # Terrain data ingestion
│   │   │   ├── srtm.rs                    # SRTM downloader
│   │   │   ├── cache.rs                   # Disk/S3 terrain cache
│   │   │   └── rix.rs                     # Ruggedness Index computation
│   │   └── workers/
│   │       ├── mod.rs                     # Worker pool management
│   │       ├── simulation_worker.rs       # Server-side simulation execution
│   │       └── optimization_worker.rs     # Server-side optimization execution
│   ├── migrations/
│   │   └── 001_initial.sql                # Full database schema
│   └── tests/
│       ├── api_integration.rs             # API integration tests
│       └── simulation_e2e.rs              # End-to-end simulation tests

├── frontend/                              # React frontend
│   ├── package.json
│   ├── vite.config.ts
│   ├── src/
│   │   ├── App.tsx                        # Router, providers, layout
│   │   ├── main.tsx                       # Entry point
│   │   ├── stores/
│   │   │   ├── authStore.ts               # Auth state (JWT, user)
│   │   │   ├── projectStore.ts            # Project state
│   │   │   ├── turbineStore.ts            # Turbine positions, selection, drag
│   │   │   └── wakeStore.ts               # Wake visualization state
│   │   ├── hooks/
│   │   │   ├── useSimulationProgress.ts   # WebSocket hook for live progress
│   │   │   └── useWasmSolver.ts           # WASM solver hook (load + run)
│   │   ├── lib/
│   │   │   ├── api.ts                     # Axios API client
│   │   │   └── wasmLoader.ts              # WASM bundle loading
│   │   ├── pages/
│   │   │   ├── Dashboard.tsx              # Project list
│   │   │   ├── Editor.tsx                 # Main editor (map + panels)
│   │   │   ├── Results.tsx                # Simulation results
│   │   │   ├── Billing.tsx                # Plan management
│   │   │   ├── Login.tsx                  # Auth pages
│   │   │   └── Register.tsx
│   │   ├── components/
│   │   │   ├── MapView.tsx                # Mapbox GL + Deck.gl map (as shown earlier)
│   │   │   ├── WindRose.tsx               # D3.js wind rose diagram
│   │   │   ├── EnergyWaterfall.tsx        # D3.js loss waterfall chart
│   │   │   ├── AEPChart.tsx               # D3.js per-turbine AEP chart
│   │   │   ├── LayoutComparison.tsx       # Side-by-side layout comparison
│   │   │   ├── SensitivityAnalysis.tsx    # Tornado chart
│   │   │   ├── WakeVisualization.tsx      # Wake plume rendering
│   │   │   ├── TurbineTable.tsx           # Sortable turbine results table
│   │   │   ├── OptimizationModal.tsx      # Optimization constraint inputs + progress
│   │   │   └── billing/
│   │   │       ├── PlanCard.tsx
│   │   │       └── UsageMeter.tsx
│   │   └── data/
│   │       └── turbine_models.json        # Client-side turbine library cache
│   └── public/
│       └── wasm/                          # WASM solver bundle (loaded at runtime)

├── wind-service/                          # Python FastAPI (Weibull + QC)
│   ├── requirements.txt
│   ├── main.py                            # FastAPI app
│   ├── weibull.py                         # Weibull MLE fitting per sector
│   ├── qc.py                              # Met data quality control
│   ├── wind_rose.py                       # Wind rose data generation
│   └── Dockerfile

├── report-service/                        # Python FastAPI (PDF generation)
│   ├── requirements.txt
│   ├── main.py                            # FastAPI app
│   ├── templates/
│   │   └── yield_report.py                # ReportLab PDF template
│   └── Dockerfile

├── k8s/                                   # Kubernetes manifests
│   ├── api-deployment.yaml
│   ├── worker-deployment.yaml
│   ├── postgres-statefulset.yaml
│   ├── redis-deployment.yaml
│   ├── python-services-deployment.yaml
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

### Benchmark 1: Single Turbine (No Wake)

**Setup:** 1 turbine, uniform wind resource (mean wind speed 8 m/s, Weibull A=9.0, k=2.0)

**Expected:** Wake loss = **0.0%** (no other turbines)

**Expected:** Capacity factor ≈ **35-40%** for typical 3 MW turbine

**Tolerance:** Wake loss < 0.1%, CF within 2% of reference tool

### Benchmark 2: Two Aligned Turbines (Direct Wake)

**Setup:** 2 turbines, 7D spacing, wind direction 0° (aligned), same turbine model (V90-3.0, D=90m)

**Expected:** Downstream turbine wake deficit ≈ **25-30%** (Jensen model with k=0.04)

**Expected:** Array wake loss ≈ **12-15%** (average of 0% and 25-30%)

**Tolerance:** Deficit within 5%, wake loss within 2%

### Benchmark 3: 3×3 Turbine Grid (Multi-Wake)

**Setup:** 9 turbines in regular grid, 5D × 5D spacing, uniform wind from all directions

**Expected:** Total wake loss ≈ **10-15%** (typical for 5D spacing)

**Expected:** Corner turbines have lowest wake loss (< 5%), center turbine has highest (> 20%)

**Tolerance:** Total wake loss within 3%, per-turbine within 5%

### Benchmark 4: Horns Rev 1 Wind Farm (Real Data Validation)

**Setup:** 80 Vestas V80-2.0 turbines (D=80m), 7D × 7D grid, North Sea site (mean wind 9.7 m/s), measured production data from 2002-2003

**Expected:** Total wake loss ≈ **8.5-9.5%** (published in multiple papers)

**Expected:** Net AEP ≈ **600 GWh/year** (measured)

**Tolerance:** Wake loss within 1%, AEP within 5% (accounts for model simplifications and inter-annual variability)

### Benchmark 5: Layout Optimization (Small Problem)

**Setup:** Square site 2km × 2km, 10 turbines (V90-3.0), dominant wind direction 270° (60% frequency), uniform wind 8 m/s

**Expected:** Optimized layout has turbines perpendicular to dominant wind (minimize wake overlap)

**Expected:** Spacing ≈ **6-8D** in dominant wind direction, **4-5D** perpendicular

**Expected:** Net AEP improvement over random layout ≈ **5-15%**

**Tolerance:** Optimizer finds layout within 2% of global optimum (verified by exhaustive search on coarse grid)

---

## Verification

### Integration Testing Checklist

1. **Auth flow** — Register → login → JWT issued → authorized API calls → token refresh → logout
2. **Project CRUD** — Create (with lat/lon) → update site boundary → save → reload → verify boundary preserved
3. **Turbine placement** — Add turbine → drag to new position → verify spatial constraints (min spacing, boundary)
4. **Met data upload** — Upload CSV → QC runs → Weibull fit → wind resource created → view wind rose
5. **WASM simulation** — Place 5 turbines → compute AEP client-side → results displayed in <1s
6. **Server simulation** — Place 30 turbines → simulation enqueued → worker picks up → WebSocket progress → results in S3
7. **Wake visualization** — Simulation completes → wake plumes render on map → toggle on/off
8. **Layout optimization** — Start optimization → progress updates via WebSocket → best layout applied → AEP improves
9. **Report generation** — Click "Download Report" → Python service generates PDF → S3 upload → download link
10. **Plan limits** — Free user → add 15 turbines → blocked with upgrade prompt
11. **Billing** — Subscribe to Pro → Stripe checkout → webhook → plan updated → limits lifted
12. **Concurrent users** — 10 users simultaneously running simulations → all complete correctly
13. **WASM/server consistency** — Same layout simulated in WASM and server → AEP results match within 1%
14. **Optimization convergence** — Run optimizer 5 times with different random seeds → all converge to similar AEP (within 2%)
15. **Error handling** — Invalid layout (turbines outside boundary) → meaningful error message → no crash

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

-- 3. User plan distribution and conversion
SELECT plan, COUNT(*) as users,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 1) as pct
FROM users GROUP BY plan ORDER BY users DESC;

-- 4. Simulation usage by user (billing period)
SELECT u.email, u.plan,
    COUNT(*) as simulations_run,
    CASE u.plan
        WHEN 'free' THEN 100
        WHEN 'pro' THEN 999999
        WHEN 'advanced' THEN 999999
    END as limit
FROM simulations s
JOIN users u ON s.user_id = u.id
WHERE s.created_at >= DATE_TRUNC('month', CURRENT_DATE)
GROUP BY u.email, u.plan
ORDER BY simulations_run DESC;

-- 5. Turbine model popularity
SELECT tm.name, tm.manufacturer, tm.rated_power_kw,
    COUNT(DISTINCT t.project_id) as projects_using,
    COUNT(t.id) as total_instances
FROM turbine_models tm
LEFT JOIN turbines t ON t.turbine_model_id = tm.id
GROUP BY tm.id
ORDER BY projects_using DESC
LIMIT 20;

-- 6. Optimization success rate and convergence
SELECT
    algorithm,
    COUNT(*) as total_runs,
    SUM(CASE WHEN completed_at IS NOT NULL THEN 1 ELSE 0 END) as completed,
    AVG(current_generation) as avg_generations,
    AVG((best_solution->>'fitness')::float) as avg_best_fitness_gwh
FROM optimization_jobs
WHERE created_at >= NOW() - INTERVAL '30 days'
GROUP BY algorithm;

-- 7. Average wake loss by turbine count
SELECT
    CASE
        WHEN turbine_count <= 10 THEN '1-10'
        WHEN turbine_count <= 25 THEN '11-25'
        WHEN turbine_count <= 50 THEN '26-50'
        ELSE '50+'
    END as turbine_range,
    AVG((results->>'wake_loss_pct')::float) as avg_wake_loss_pct,
    COUNT(*) as simulation_count
FROM (
    SELECT s.id, s.results,
        jsonb_array_length(s.layout_snapshot->'turbines') as turbine_count
    FROM simulations s
    WHERE s.status = 'completed' AND s.simulation_type = 'full_yield'
        AND s.created_at >= NOW() - INTERVAL '30 days'
) sub
GROUP BY turbine_range
ORDER BY turbine_range;
```

### Performance Benchmarks

| Operation | Target | Method |
|-----------|--------|--------|
| WASM flow model: 500×500 grid | <200ms | Browser benchmark (Chrome DevTools) |
| WASM AEP: 10 turbines, 16 sectors, 20 wind speed bins | <50ms | Browser benchmark |
| WASM AEP: 25 turbines (max WASM threshold) | <200ms | Browser benchmark |
| Server AEP: 100 turbines | <2s | Server timing, 4 cores |
| Server optimization: 25 turbines, 200 generations | <5min | Server timing, 8 cores |
| Map rendering: 50 turbines + wake plumes | 60 FPS | Chrome FPS counter during pan/zoom |
| Map rendering: 100 turbines + wake plumes | 30 FPS | Chrome FPS counter during pan/zoom |
| API: create simulation | <150ms | p95 latency (Prometheus) |
| API: list turbine models | <100ms | p95 latency with 100 models in DB |
| WASM bundle load (cached) | <500ms | Service worker cache hit |
| WASM bundle load (cold) | <2s | CDN delivery, 1.5MB gzipped |
| WebSocket: progress latency | <100ms | Time from worker emit to client receive |
| Met data CSV upload: 50K rows | <5s | Server-side parsing + TimescaleDB insert |
| PDF report generation: 25 turbines | <10s | Python ReportLab + Mapbox static API |

---

## Deployment Architecture

### Infrastructure

```
┌─────────────────────────────────────────────────┐
│                  CloudFront CDN                   │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────┐ │
│  │ Frontend SPA │  │ WASM Bundle │  │ Terrain  │ │
│  │ (HTML/JS/CSS)│  │ (versioned) │  │ Tiles    │ │
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
│ Simulation   │ │ Optimization │ │ Python       │
│ Workers      │ │ Workers      │ │ Services     │
│ Pod ×5 (HPA) │ │ Pod ×3 (HPA) │ │ (Weibull+PDF)│
└──────┬───────┘ └──────┬───────┘ └──────┬───────┘
       │                │                │
       └────────────────┼────────────────┘
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ PostgreSQL   │ │ Redis        │ │ S3 Bucket    │
│ 16 + PostGIS │ │ 7 (jobs +    │ │ (met data,   │
│ + TimescaleDB│ │ pub/sub)     │ │ terrain,     │
│ (StatefulSet)│ │              │ │ results)     │
└──────────────┘ └──────────────┘ └──────────────┘
```

**Infrastructure Details:**

- **Frontend:** React SPA served from S3 + CloudFront, with WASM bundle cached at edge (versioned URLs like `/wasm/v1.2.3/`)
- **API Servers:** Rust/Axum on AWS ECS/EKS, 3 replicas behind ALB, HPA based on CPU (target 70%)
- **Workers:** Separate deployments for simulation and optimization, HPA based on Redis queue depth (scale up when queue >10)
- **PostgreSQL:** RDS PostgreSQL 16 with PostGIS + TimescaleDB extensions, db.r6g.xlarge (4 vCPU, 32 GB RAM), multi-AZ
- **Redis:** ElastiCache Redis 7, cache.r6g.large (2 vCPU, 13 GB RAM), cluster mode disabled (pub/sub compatibility)
- **S3:** Single bucket with prefixes: `/met-data/{project_id}/`, `/terrain-tiles/`, `/simulation-results/{sim_id}/`, `/reports/{sim_id}/`
- **Monitoring:** Prometheus (in-cluster) + Grafana Cloud, Sentry for error tracking
- **Secrets:** AWS Secrets Manager for database credentials, API keys, Stripe keys
- **Backups:** PostgreSQL automated backups (7-day retention), S3 versioning enabled

### Scaling Strategy

| Component | Baseline | Scale Trigger | Max |
|-----------|----------|---------------|-----|
| API Servers | 3 pods | CPU >70% or req/s >1000 | 10 pods |
| Simulation Workers | 5 pods | Queue depth >10 | 20 pods |
| Optimization Workers | 3 pods | Queue depth >5 | 10 pods |
| PostgreSQL | r6g.xlarge | Connections >80% | r6g.2xlarge |
| Redis | r6g.large | Memory >80% | r6g.xlarge |

### Cost Estimate (AWS us-east-1, monthly)

- **Compute (ECS/EKS):** $400 (API 3×t3.large + workers 8×c6g.xlarge)
- **Database (RDS):** $350 (db.r6g.xlarge multi-AZ)
- **Cache (ElastiCache):** $150 (cache.r6g.large)
- **Storage (S3):** $50 (500 GB met data + results + terrain)
- **CDN (CloudFront):** $100 (1 TB data transfer)
- **Misc (ALB, Secrets, NAT):** $100
- **Total:** ~$1,150/month baseline (scales with users)

---

## Post-MVP Roadmap

### Phase 1 — Advanced Wake Models (Weeks 10–12)

**Motivation:** Improve wake loss accuracy for bankable energy yield assessments (P50/P75/P90 estimates required by project financiers). Jensen model has ±3-5% error; advanced models reduce this to ±1-2%.

**Features:**
- **Bastankhah-Porte-Agel Gaussian wake model:** State-of-the-art analytical wake model with Gaussian velocity deficit profile, validated against LES and field data. Accounts for hub-height wind speed, turbulence intensity, and thrust coefficient. 3× more accurate than Jensen for modern large turbines (D >120m).
- **Frandsen effective turbulence model:** Compute added turbulence from wakes for fatigue load assessment. Important for turbine suitability checks and lifetime energy estimates (turbines lose 0.1-0.5% annual performance from degradation).
- **Dynamic Wake Meandering (DWM):** Time-domain wake model accounting for wake advection with turbulent eddies. Predicts power fluctuations and fatigue loads. Computationally expensive (10× slower than Jensen) but necessary for IEC-compliant load calculations.
- **Wake model comparison tool:** Side-by-side AEP estimates from Jensen / Bastankhah / DWM with uncertainty ranges.

**Effort:** 3 weeks (1 week per model + integration + validation against LES data)

### Phase 2 — CFD Flow Model for Complex Terrain (Weeks 13–16)

**Motivation:** Linearized flow model (WAsP) breaks down for RIX >5% (steep terrain, cliffs, narrow valleys). CFD provides ±2-3% accuracy even for complex sites, vs. ±10-20% error for linearized models.

**Features:**
- **Steady RANS CFD solver:** k-epsilon or k-omega SST turbulence model, solving incompressible Navier-Stokes equations on 3D terrain mesh.
- **Automatic mesh generation:** Terrain-following hexahedral mesh with refinement near turbine positions and sharp terrain features.
- **Solver backend:** OpenFOAM integration (open-source, industry-standard) or custom Rust CFD solver (higher performance, better WASM potential post-MVP).
- **Runtime:** 10-30 minutes for typical site (5km × 5km, 1M cells, 1000 iterations) on 8-core server. Too slow for real-time client-side, so server-only.
- **Validation:** Compare to wind tunnel measurements (Bolund Hill benchmark) and full-scale LiDAR scans (Perdigão campaign). Target ±2% accuracy in speedup factor.

**Effort:** 4 weeks (2 weeks meshing + 1 week solver integration + 1 week validation)

### Phase 3 — Uncertainty Quantification & P50/P75/P90 (Weeks 17–20)

**Motivation:** Bankable energy yield assessments required for project finance must include P50 (median), P75 (75th percentile = conservative), and P90 (90th percentile = very conservative) AEP estimates. Financiers use P75 or P90 for debt sizing.

**Features:**
- **Uncertainty source breakdown:**
  - Wind resource uncertainty: ±5-7% (measurement error, long-term correction, spatial extrapolation)
  - Wake model uncertainty: ±3-5% (model assumptions, turbulence, atmospheric stability)
  - Power curve uncertainty: ±2-3% (manufacturer tolerance, air density correction)
  - Availability / performance: ±1-2% (unplanned downtime, degradation)
  - Combined uncertainty via Monte Carlo: 10,000 simulations with random draws from each distribution
- **P-values:** P50 (median AEP), P75 (1-sigma below), P90 (1.3-sigma below)
- **Uncertainty waterfall chart:** Visual breakdown of each source's contribution to total uncertainty
- **IEC 61400-15 compliant reporting:** Follows international standard for energy assessment

**Effort:** 4 weeks (2 weeks Monte Carlo engine + 1 week uncertainty models + 1 week reporting)

### Phase 4 — Noise & Shadow Flicker Analysis (Weeks 21–23)

**Motivation:** Regulatory constraints in most countries require wind farms to meet noise limits (35-45 dBA at dwellings) and shadow flicker limits (30 hours/year in Germany, 30 min/day max). Automated analysis prevents costly re-optimization after permitting.

**Features:**
- **ISO 9613-2 noise propagation model:** Octave-band sound power from turbines, atmospheric absorption, ground attenuation, geometric spreading. Predict noise level at receptor points (dwellings).
- **Noise-constrained optimization:** Automatically curtail turbines or adjust layout to meet receptor-specific noise limits.
- **Shadow flicker calculation:** Sun path model (ephemeris) + rotor geometry → compute shadow hours/year at each receptor. Highlight turbines causing exceedances.
- **Visualization:** Noise contour map overlay on terrain, shadow flicker hot-spots on map
- **Reports:** Noise assessment report (ISO format), shadow flicker report (with mitigation options)

**Effort:** 3 weeks (1 week noise model + 1 week shadow flicker + 1 week optimization integration)

### Phase 5 — Cable Routing Optimization (Weeks 24–26)

**Motivation:** Inter-array cables represent 10-15% of wind farm capital cost ($50-100K per km for 33kV underground). Optimizing cable routes can save $500K-$2M on a 50-turbine farm.

**Features:**
- **Minimum spanning tree + Steiner tree:** Connect all turbines to substation minimizing total trench length.
- **Electrical loss minimization:** Size cable conductor based on current → balance capital cost (larger cable) vs. energy loss cost over 25-year lifetime.
- **Terrain cost factors:** Higher cost to trench through rock, forest, wetlands. User-defined cost map.
- **Constraint handling:** Avoid exclusion zones, cross roads at right angles, maximum cable length per circuit.
- **Substation placement optimization:** Find optimal substation location minimizing total cable cost + grid connection cost.
- **Visualization:** 3D cable routes on map with color-coded by voltage drop / cost.

**Effort:** 3 weeks (1 week graph algorithms + 1 week electrical sizing + 1 week visualization)

### Phase 6 — API & Integrations (Weeks 27-28)

**Motivation:** Enable integration with customer workflows (GIS systems, financial models, project management tools). API access generates 20-30% of ARR for SaaS infrastructure tools.

**Features:**
- **REST API:** Full CRUD for projects, turbines, simulations, optimizations. JSON responses with pagination.
- **Webhooks:** Push notifications for simulation completion, optimization convergence, plan limit warnings.
- **API authentication:** API keys (hashed, rate-limited) separate from user JWTs. Team plan feature.
- **Rate limits:** 1000 req/hour per API key, 10 concurrent simulations.
- **Client libraries:** Python SDK (`pip install windfarm-sdk`), JavaScript SDK (`npm install @windfarm/sdk`).
- **Documentation:** OpenAPI spec, interactive docs (Swagger UI), code examples in Python/JS/curl.
- **Use cases:** Automated batch analysis (run 100 layouts overnight), integration with QGIS (geospatial analysis), export to Excel financial models.

**Effort:** 2 weeks (1 week API design + 1 week client libraries + docs)

### Phase 7 — Multi-Farm Portfolio Analysis (Weeks 29-30)

**Motivation:** Utilities and IPPs managing portfolios of 10-50 wind farms need aggregate analysis (portfolio AEP, capacity factor distribution, risk correlation across sites).

**Features:**
- **Portfolio dashboard:** Map of all wind farms, aggregate capacity, total AEP, capacity factor heatmap.
- **Correlation analysis:** Compute wind resource correlation between sites (affects portfolio risk for financing).
- **Time-series analysis:** Monthly/annual AEP variability, inter-annual correlation (El Niño / NAO effects).
- **Benchmarking:** Compare actual production vs. predicted AEP, identify underperforming turbines/farms.
- **Export:** Portfolio-level reports for investors, lenders, board presentations.

**Effort:** 2 weeks (1 week data aggregation + 1 week dashboards + reporting)

### Phase 8 — Machine Learning for Wind Resource (Weeks 31-32)

**Motivation:** ML models can improve wind resource predictions by 5-10% compared to traditional MCP (measure-correlate-predict) methods, reducing uncertainty and increasing P50 AEP estimates.

**Features:**
- **Advanced MCP methods:** Neural network, random forest, gradient boosting for long-term correction (vs. simple linear regression).
- **Reanalysis data fusion:** Combine multiple reanalysis datasets (ERA5, MERRA-2, NORA3) with ML weighting based on local site characteristics.
- **Anomaly detection:** Automatically detect and flag suspect met data (sensor drift, icing, bird nests, data logger errors).
- **Wind power forecasting:** Short-term (1-48 hour) forecasts for grid integration and trading (post-construction feature).

**Effort:** 2 weeks (1 week ML models + 1 week reanalysis integration + validation)

