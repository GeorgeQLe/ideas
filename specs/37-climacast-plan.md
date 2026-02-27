# 37. ClimaCast — High-Resolution Microclimate Modeling for Construction and Agriculture

## Implementation Plan

**MVP Scope:** Browser-based site definition via map polygon drawing with automatic terrain ingestion from SRTM 30m DEMs and OpenStreetMap building footprints, cloud-native AI-powered downscaling service that transforms coarse GFS/HRRR numerical weather predictions (9-25km resolution) into hyperlocal microclimate forecasts at 10-100m resolution using trained convolutional neural networks accounting for terrain elevation, slope, aspect, surface roughness, and land use classification from Sentinel-2 satellite imagery, OpenFOAM-based CFD wind simulation orchestrated on Kubernetes pods with automated snappyHexMesh meshing and steady-state RANS k-epsilon turbulence modeling for building-level wind field computation (results delivered in 30-90 minutes for sites up to 2km × 2km), solar irradiance modeling with hour-by-hour shadow casting from 3D building geometries and terrain features producing annual/monthly DNI/DHI/GHI maps, interactive 3D visualization powered by Mapbox GL JS with vector wind field overlays via Deck.gl and animated particle trails rendered in WebGL, field-level frost risk prediction using cold air drainage modeling and DEM-based frost pocket identification, time-series data extraction at any point with CSV/GeoTIFF export, basic PDF report generation with executive summary and regulatory formatting, PostgreSQL + PostGIS for geospatial project storage with S3/R2 for simulation result meshes and field data, Stripe billing with three tiers (Free / Planner $99/mo / Team $399/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth, job orchestration |
| ML Inference | Python 3.12 (FastAPI) | PyTorch + ONNX Runtime for downscaling, NVIDIA Triton on GPU nodes |
| CFD Solver | OpenFOAM 11 | Containerized on Kubernetes, snappyHexMesh meshing, simpleFoam/pimpleFoam solvers |
| Database | PostgreSQL 16 + PostGIS 3.4 | Projects, simulations, forecast time-series, geospatial queries |
| Time-Series | TimescaleDB extension | Weather forecast history, observation data, continuous aggregates |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 / Cloudflare R2 | Simulation meshes, OpenFOAM output files, terrain DEMs, renders |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| 3D Visualization | Mapbox GL JS | 3D terrain rendering, building extrusions, custom WebGL layers |
| Data Overlays | Deck.gl | Vector field visualization, heatmaps, particle systems |
| WASM Compute | wasm-pack + Rust | Client-side terrain analysis, sun path calculations, frost pocket detection |
| Real-time | WebSocket (Axum) | Simulation progress streaming, forecast update notifications |
| Job Queue | Redis 7 + async tasks | CFD job distribution, ML inference queue, priority scheduling |
| K8s Orchestration | Karpenter | Auto-scaling GPU nodes (A10G) for ML, CPU spot instances for OpenFOAM |
| Message Queue | Redis Streams | CFD job events, forecast ingestion pipeline |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks, usage-based metering |
| CDN | CloudFront | Frontend assets, terrain tiles, WASM bundles |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, solver performance, error tracking |
| CI/CD | GitHub Actions | WASM build, Docker images, Kubernetes deployments, CFD solver tests |

### Key Architecture Decisions

1. **Rust/Axum primary backend with Python ML microservice**: The core API, job orchestration, database access, and business logic run in Rust for performance, type safety, and low resource usage. Python FastAPI serves exclusively as a microservice for ML inference (PyTorch downscaling models) and scientific computation (NumPy/SciPy terrain analysis, netCDF weather data parsing). This hybrid approach leverages Rust's strengths for production services while using Python's ecosystem for ML/scientific tasks where it excels.

2. **Cloud-native CFD with OpenFOAM on Kubernetes**: OpenFOAM runs in containerized pods with dynamic scaling via Karpenter. Each simulation job triggers a pod with allocated CPU cores (2-16 depending on domain size) and memory (4-32GB). Spot instances reduce cost by 60-70% for non-urgent jobs. Job orchestration happens in Rust (Axum workers poll Redis queue), with status updates streamed via WebSocket. This allows scaling from zero to hundreds of concurrent simulations without managing dedicated HPC infrastructure.

3. **AI downscaling with pre-trained models on GPU nodes**: Downscaling from 9km GFS to 50m local resolution uses U-Net convolutional neural networks trained on pairs of coarse NWP output and fine-resolution WRF simulations. Models run on NVIDIA A10G GPU nodes (2-4 instances) via Triton Inference Server with ONNX Runtime backend. Training happens offline on historical ERA5/WRF datasets; production only runs inference. Typical downscaling completes in 30-90 seconds for a 10km × 10km domain. This is 100x faster and 1000x cheaper than running full WRF dynamical downscaling per request.

4. **PostGIS for spatial queries with TimescaleDB for forecast time-series**: Project site boundaries, building footprints, and simulation domains are stored as PostGIS geometries with spatial indexes (GIST) enabling fast "find all projects within this region" queries. Forecast data (temperature, wind, precipitation at grid points over time) uses TimescaleDB hypertables with automatic partitioning and retention policies. This hybrid approach handles both spatial (where) and temporal (when) queries efficiently.

5. **WASM for client-side terrain and solar analysis**: Simple analyses (sun path computation, viewshed from a point, frost pocket identification via DEM slope analysis) run entirely in the browser via Rust compiled to WASM. This provides instant feedback for interactive tools (shadow animation scrubber, frost risk heatmap on hover) without server round-trips. The WASM module shares data structures with the server-side Rust code, ensuring consistency. More complex analyses (CFD, multi-day downscaling) still run server-side.

6. **S3/R2 for simulation output with presigned URLs**: OpenFOAM produces large output files (mesh files 10-100MB, field data 50-500MB per timestep). These are written directly to S3 from worker pods, with metadata (S3 key, size, timesteps available) stored in PostgreSQL. Clients receive presigned URLs for direct download, avoiding backend proxying. ParaView-compatible VTK files enable users to download results for offline post-processing in addition to web visualization.

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "postgis";
CREATE EXTENSION IF NOT EXISTS "timescaledb";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";

-- Users
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email TEXT UNIQUE NOT NULL,
    password_hash TEXT,  -- NULL for OAuth-only users
    name TEXT NOT NULL,
    avatar_url TEXT,
    auth_provider TEXT DEFAULT 'email',  -- email | google | github
    auth_provider_id TEXT,
    plan TEXT NOT NULL DEFAULT 'free',  -- free | planner | team
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
    role TEXT NOT NULL DEFAULT 'member',  -- owner | admin | editor | viewer
    joined_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(org_id, user_id)
);
CREATE INDEX org_members_org_idx ON org_members(org_id);
CREATE INDEX org_members_user_idx ON org_members(user_id);

-- Projects (site + simulation workspace)
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    project_type TEXT NOT NULL DEFAULT 'general',  -- construction | agriculture | renewable | urban_planning | general
    site_boundary GEOMETRY(POLYGON, 4326) NOT NULL,  -- WGS84 site boundary
    site_center GEOMETRY(POINT, 4326) NOT NULL,  -- Computed from boundary centroid
    domain_config JSONB NOT NULL DEFAULT '{}',  -- {resolution_m, domain_size_m, terrain_source, mesh_refinement_zones}
    site_model_version INTEGER NOT NULL DEFAULT 1,
    is_public BOOLEAN NOT NULL DEFAULT false,
    thumbnail_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);
CREATE INDEX projects_org_idx ON projects(org_id);
CREATE INDEX projects_boundary_idx ON projects USING GIST(site_boundary);
CREATE INDEX projects_center_idx ON projects USING GIST(site_center);
CREATE INDEX projects_updated_idx ON projects(updated_at DESC);

-- Site Models (terrain, buildings, land use)
CREATE TABLE site_models (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    version INTEGER NOT NULL,
    terrain_dem_url TEXT NOT NULL,  -- S3 URL to GeoTIFF DEM
    terrain_resolution_m REAL NOT NULL,  -- 30m (SRTM), 10m (ASTER), 1m (lidar)
    terrain_source TEXT NOT NULL,  -- srtm | aster | lidar | user_uploaded
    buildings JSONB NOT NULL DEFAULT '[]',  -- GeoJSON FeatureCollection with height attribute
    land_use_url TEXT,  -- S3 URL to classified raster (from Sentinel-2)
    surface_roughness JSONB DEFAULT '{}',  -- Derived roughness map for CFD
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(project_id, version)
);
CREATE INDEX site_models_project_idx ON site_models(project_id);

-- Simulations
CREATE TABLE simulations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    site_model_id UUID NOT NULL REFERENCES site_models(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    simulation_type TEXT NOT NULL,  -- wind_cfd | solar | thermal | frost | downscale | comprehensive
    status TEXT NOT NULL DEFAULT 'queued',  -- queued | meshing | solving | post_processing | completed | failed | cancelled
    config JSONB NOT NULL DEFAULT '{}',  -- Type-specific parameters (wind directions, solver settings, date ranges)
    mesh_url TEXT,  -- S3 URL to OpenFOAM mesh (constant/polyMesh)
    results_url TEXT,  -- S3 URL prefix for result files
    results_summary JSONB,  -- Quick-access summary (max wind speed, frost pockets count, solar yield kWh)
    error_message TEXT,
    compute_time_seconds INTEGER,
    node_count INTEGER,  -- Mesh cell count for CFD
    cost_credits REAL,  -- Internal cost tracking
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX sims_project_idx ON simulations(project_id);
CREATE INDEX sims_user_idx ON simulations(user_id);
CREATE INDEX sims_status_idx ON simulations(status);
CREATE INDEX sims_created_idx ON simulations(created_at DESC);

-- CFD Jobs (server-side OpenFOAM execution)
CREATE TABLE cfd_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    simulation_id UUID NOT NULL REFERENCES simulations(id) ON DELETE CASCADE,
    pod_name TEXT,
    cores_allocated INTEGER DEFAULT 4,
    memory_mb INTEGER DEFAULT 8192,
    priority INTEGER DEFAULT 0,  -- Higher = more urgent
    progress_pct REAL DEFAULT 0.0,
    current_iteration INTEGER DEFAULT 0,
    convergence_data JSONB DEFAULT '[]',  -- [{iteration, residual_U, residual_p, residual_k}]
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX cfd_jobs_sim_idx ON cfd_jobs(simulation_id);
CREATE INDEX cfd_jobs_status_idx ON cfd_jobs(pod_name);

-- Forecast Subscriptions (continuous downscaled forecasts)
CREATE TABLE forecast_subscriptions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    update_frequency TEXT NOT NULL DEFAULT 'daily',  -- hourly | 6h | daily
    parameters TEXT[] NOT NULL DEFAULT '{temperature,wind_speed,precipitation}',
    alert_rules JSONB DEFAULT '[]',  -- [{parameter, threshold, direction, channels}]
    is_active BOOLEAN NOT NULL DEFAULT true,
    last_update_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX forecast_subs_project_idx ON forecast_subscriptions(project_id);
CREATE INDEX forecast_subs_active_idx ON forecast_subscriptions(is_active) WHERE is_active = true;

-- Forecast Data (downscaled time-series)
CREATE TABLE forecast_data (
    time TIMESTAMPTZ NOT NULL,
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    location GEOMETRY(POINT, 4326) NOT NULL,  -- Grid point
    parameter TEXT NOT NULL,  -- temperature | wind_speed | wind_direction | precipitation | humidity | pressure
    value REAL NOT NULL,
    forecast_run_time TIMESTAMPTZ NOT NULL,  -- Which model run produced this forecast
    model_source TEXT NOT NULL,  -- gfs | hrrr | ecmwf
    PRIMARY KEY (time, project_id, location, parameter, forecast_run_time)
);
SELECT create_hypertable('forecast_data', 'time', chunk_time_interval => INTERVAL '1 day');
CREATE INDEX forecast_data_project_time_idx ON forecast_data(project_id, time DESC);
CREATE INDEX forecast_data_location_idx ON forecast_data USING GIST(location);

-- Weather Stations (user-connected or public)
CREATE TABLE weather_stations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    org_id UUID REFERENCES organizations(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    location GEOMETRY(POINT, 4326) NOT NULL,
    elevation_m REAL,
    provider TEXT NOT NULL,  -- davis | ambient | mesonet | custom_api
    api_credentials JSONB,  -- Encrypted credentials for API access
    parameters TEXT[] NOT NULL DEFAULT '{temperature,wind_speed,wind_direction,humidity,precipitation}',
    last_observation_at TIMESTAMPTZ,
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX stations_location_idx ON weather_stations USING GIST(location);
CREATE INDEX stations_org_idx ON weather_stations(org_id);

-- Station Observations (time-series)
CREATE TABLE station_observations (
    time TIMESTAMPTZ NOT NULL,
    station_id UUID NOT NULL REFERENCES weather_stations(id) ON DELETE CASCADE,
    parameter TEXT NOT NULL,  -- temperature | wind_speed | wind_direction | humidity | precipitation | pressure
    value REAL NOT NULL,
    quality_flag TEXT DEFAULT 'ok',  -- ok | suspect | bad | missing
    PRIMARY KEY (time, station_id, parameter)
);
SELECT create_hypertable('station_observations', 'time', chunk_time_interval => INTERVAL '7 days');
CREATE INDEX station_obs_station_time_idx ON station_observations(station_id, time DESC);

-- Reports
CREATE TABLE reports (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    simulation_ids UUID[] NOT NULL,
    template_type TEXT NOT NULL,  -- wind_comfort | solar_access | frost_risk | construction_weather | comprehensive
    branding_config JSONB DEFAULT '{}',  -- {logo_url, company_name, colors}
    generated_pdf_url TEXT,
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | generating | completed | failed
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX reports_project_idx ON reports(project_id);

-- Usage Tracking (for billing enforcement)
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,  -- cfd_simulation_minutes | downscaling_runs | storage_gb_days
    quantity REAL NOT NULL,
    period_start DATE NOT NULL,
    period_end DATE NOT NULL,
    metadata JSONB DEFAULT '{}',  -- {simulation_id, node_count, etc.}
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX usage_user_period_idx ON usage_records(user_id, period_start, period_end);
CREATE INDEX usage_org_period_idx ON usage_records(org_id, period_start, period_end);

-- Share Links (public access tokens)
CREATE TABLE share_links (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    token TEXT UNIQUE NOT NULL,
    password_hash TEXT,  -- Optional password protection
    expires_at TIMESTAMPTZ,
    view_count INTEGER DEFAULT 0,
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX share_links_token_idx ON share_links(token);
CREATE INDEX share_links_project_idx ON share_links(project_id);
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
    pub project_type: String,
    #[serde(skip)]
    pub site_boundary: sqlx::types::Json<geojson::Geometry>,  // PostGIS geometry as GeoJSON
    #[serde(skip)]
    pub site_center: sqlx::types::Json<geojson::Geometry>,
    pub domain_config: serde_json::Value,
    pub site_model_version: i32,
    pub is_public: bool,
    pub thumbnail_url: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct SiteModel {
    pub id: Uuid,
    pub project_id: Uuid,
    pub version: i32,
    pub terrain_dem_url: String,
    pub terrain_resolution_m: f32,
    pub terrain_source: String,
    pub buildings: serde_json::Value,  // GeoJSON FeatureCollection
    pub land_use_url: Option<String>,
    pub surface_roughness: serde_json::Value,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Simulation {
    pub id: Uuid,
    pub project_id: Uuid,
    pub site_model_id: Uuid,
    pub user_id: Uuid,
    pub simulation_type: String,
    pub status: String,
    pub config: serde_json::Value,
    pub mesh_url: Option<String>,
    pub results_url: Option<String>,
    pub results_summary: Option<serde_json::Value>,
    pub error_message: Option<String>,
    pub compute_time_seconds: Option<i32>,
    pub node_count: Option<i32>,
    pub cost_credits: Option<f32>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct CfdJob {
    pub id: Uuid,
    pub simulation_id: Uuid,
    pub pod_name: Option<String>,
    pub cores_allocated: i32,
    pub memory_mb: i32,
    pub priority: i32,
    pub progress_pct: f32,
    pub current_iteration: i32,
    pub convergence_data: serde_json::Value,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct ForecastSubscription {
    pub id: Uuid,
    pub project_id: Uuid,
    pub user_id: Uuid,
    pub update_frequency: String,
    pub parameters: Vec<String>,
    pub alert_rules: serde_json::Value,
    pub is_active: bool,
    pub last_update_at: Option<DateTime<Utc>>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct WeatherStation {
    pub id: Uuid,
    pub org_id: Option<Uuid>,
    pub user_id: Option<Uuid>,
    pub name: String,
    #[serde(skip)]
    pub location: sqlx::types::Json<geojson::Geometry>,
    pub elevation_m: Option<f32>,
    pub provider: String,
    #[serde(skip)]
    pub api_credentials: Option<serde_json::Value>,
    pub parameters: Vec<String>,
    pub last_observation_at: Option<DateTime<Utc>>,
    pub is_active: bool,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct CreateProjectRequest {
    pub name: String,
    pub description: Option<String>,
    pub project_type: String,
    pub site_boundary: geojson::Geometry,  // GeoJSON polygon
    pub domain_config: DomainConfig,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct DomainConfig {
    pub resolution_m: f32,
    pub domain_size_m: f32,
    pub terrain_source: String,  // srtm | aster | lidar
    pub mesh_refinement_zones: Vec<RefinementZone>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct RefinementZone {
    pub geometry: geojson::Geometry,
    pub target_cell_size_m: f32,
}

#[derive(Debug, Deserialize)]
pub struct CreateSimulationRequest {
    pub simulation_type: SimulationType,
    pub config: serde_json::Value,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum SimulationType {
    WindCfd,
    Solar,
    Thermal,
    Frost,
    Downscale,
    Comprehensive,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct WindCfdConfig {
    pub wind_directions: Vec<f32>,  // Degrees from north
    pub reference_wind_speed: f32,  // m/s at reference height
    pub reference_height: f32,  // meters
    pub turbulence_model: String,  // kEpsilon | kOmegaSST
    pub comfort_criteria: String,  // lawson | dutch_nwn
    pub output_heights: Vec<f32>,  // Heights for slice extraction (e.g., [1.5, 10, 50])
}

#[derive(Debug, Deserialize, Serialize)]
pub struct SolarConfig {
    pub start_date: NaiveDate,
    pub end_date: NaiveDate,
    pub time_step_hours: f32,  // 0.5 = 30 min, 1.0 = 1 hour
    pub panel_tilt: Option<f32>,  // Degrees from horizontal (for PV analysis)
    pub panel_azimuth: Option<f32>,  // Degrees from north
}

#[derive(Debug, Deserialize, Serialize)]
pub struct FrostConfig {
    pub analysis_period_days: i32,  // Forecast days to analyze
    pub crop_type: Option<String>,
    pub critical_temperature_c: f32,  // Frost damage threshold
}
```

---

## Architecture Deep-Dives

### 1. Terrain Ingestion Pipeline (Rust + GDAL)

Automatically fetch and process terrain DEMs, building footprints, and land use data when a project is created. This pipeline runs server-side and caches results in S3.

```rust
// src/services/terrain.rs

use gdal::{Dataset, Driver};
use gdal::raster::RasterBand;
use aws_sdk_s3::Client as S3Client;
use geojson::{Geometry, Value as GeoValue};
use uuid::Uuid;

pub struct TerrainService {
    s3: S3Client,
    srtm_cache_path: String,
}

impl TerrainService {
    /// Fetch terrain DEM for project boundary and upload to S3
    pub async fn ingest_terrain(
        &self,
        project_id: Uuid,
        boundary: &Geometry,
        resolution: TerrainResolution,
    ) -> anyhow::Result<TerrainIngestionResult> {
        // 1. Extract bounding box from boundary polygon
        let bbox = self.extract_bbox(boundary)?;

        // 2. Determine source based on resolution preference
        let source = match resolution {
            TerrainResolution::Coarse => TerrainSource::Srtm30m,
            TerrainResolution::Medium => TerrainSource::Aster30m,
            TerrainResolution::Fine => TerrainSource::LidarIfAvailable,
        };

        // 3. Fetch DEM tiles covering bounding box
        let tiles = self.fetch_dem_tiles(&bbox, source).await?;

        // 4. Mosaic and clip to exact boundary
        let merged_dem = self.mosaic_tiles(&tiles)?;
        let clipped_dem = self.clip_to_boundary(&merged_dem, boundary)?;

        // 5. Compute derived products
        let slope = self.compute_slope(&clipped_dem)?;
        let aspect = self.compute_aspect(&clipped_dem)?;
        let hillshade = self.compute_hillshade(&clipped_dem)?;

        // 6. Export to GeoTIFF and upload to S3
        let geotiff_path = format!("/tmp/{}_dem.tif", project_id);
        self.write_geotiff(&clipped_dem, &geotiff_path)?;

        let s3_key = format!("terrain/{}/dem.tif", project_id);
        self.s3.put_object()
            .bucket("climacast-terrain")
            .key(&s3_key)
            .body(tokio::fs::read(&geotiff_path).await?.into())
            .content_type("image/tiff")
            .send()
            .await?;

        Ok(TerrainIngestionResult {
            dem_url: format!("s3://climacast-terrain/{}", s3_key),
            resolution_m: self.get_actual_resolution(&clipped_dem),
            source: source.to_string(),
            bbox,
            statistics: self.compute_statistics(&clipped_dem)?,
        })
    }

    /// Fetch building footprints from OpenStreetMap Overpass API
    pub async fn fetch_buildings(
        &self,
        boundary: &Geometry,
    ) -> anyhow::Result<geojson::FeatureCollection> {
        let bbox = self.extract_bbox(boundary)?;

        // Overpass API query for buildings with height tags
        let query = format!(
            r#"
            [out:json][timeout:60];
            (
              way["building"]({},{},{},{});
              relation["building"]({},{},{},{});
            );
            out body;
            >;
            out skel qt;
            "#,
            bbox.min_lat, bbox.min_lon, bbox.max_lat, bbox.max_lon,
            bbox.min_lat, bbox.min_lon, bbox.max_lat, bbox.max_lon,
        );

        let client = reqwest::Client::new();
        let response = client
            .post("https://overpass-api.de/api/interpreter")
            .body(query)
            .send()
            .await?;

        let osm_data: serde_json::Value = response.json().await?;

        // Convert OSM elements to GeoJSON features with height attribute
        let features = self.osm_to_geojson(&osm_data)?;

        Ok(geojson::FeatureCollection {
            features,
            bbox: None,
            foreign_members: None,
        })
    }

    /// Compute surface roughness map from land use and buildings
    pub async fn compute_surface_roughness(
        &self,
        land_use_raster: &Dataset,
        buildings: &geojson::FeatureCollection,
        resolution_m: f32,
    ) -> anyhow::Result<Vec<f32>> {
        // Land use roughness classes (z0 in meters)
        let roughness_lut: std::collections::HashMap<u8, f32> = [
            (1, 0.0002),   // Water
            (2, 0.03),     // Grassland
            (3, 0.25),     // Forest
            (4, 0.5),      // Urban low-density
            (5, 1.5),      // Urban high-density
            (6, 0.01),     // Bare soil
            (7, 0.05),     // Cropland
        ].iter().cloned().collect();

        let band: RasterBand = land_use_raster.rasterband(1)?;
        let (width, height) = band.size();
        let land_use_data = band.read_as::<u8>((0, 0), band.size(), band.size(), None)?;

        // Initialize roughness with land use values
        let mut roughness: Vec<f32> = land_use_data.data.iter()
            .map(|&lu| *roughness_lut.get(&lu).unwrap_or(&0.1))
            .collect();

        // Increase roughness in cells with buildings
        for feature in &buildings.features {
            if let Some(height) = feature.property("height")
                .and_then(|v| v.as_f64())
            {
                // Building roughness ~ 0.1 * height
                let building_z0 = (0.1 * height as f32).min(2.0);

                // Rasterize building footprint and update roughness
                // (simplified - actual implementation uses GDAL rasterization)
                let indices = self.rasterize_polygon(
                    &feature.geometry, width, height, resolution_m
                )?;

                for idx in indices {
                    roughness[idx] = roughness[idx].max(building_z0);
                }
            }
        }

        Ok(roughness)
    }

    fn extract_bbox(&self, geometry: &Geometry) -> anyhow::Result<BoundingBox> {
        match &geometry.value {
            GeoValue::Polygon(coords) => {
                let mut min_lon = f64::INFINITY;
                let mut max_lon = f64::NEG_INFINITY;
                let mut min_lat = f64::INFINITY;
                let mut max_lat = f64::NEG_INFINITY;

                for ring in coords {
                    for point in ring {
                        min_lon = min_lon.min(point[0]);
                        max_lon = max_lon.max(point[0]);
                        min_lat = min_lat.min(point[1]);
                        max_lat = max_lat.max(point[1]);
                    }
                }

                Ok(BoundingBox { min_lon, max_lon, min_lat, max_lat })
            }
            _ => anyhow::bail!("Expected Polygon geometry"),
        }
    }

    async fn fetch_dem_tiles(
        &self,
        bbox: &BoundingBox,
        source: TerrainSource,
    ) -> anyhow::Result<Vec<Dataset>> {
        match source {
            TerrainSource::Srtm30m => {
                // SRTM tiles are 1°x1° cells named like N37W122.hgt
                let mut tiles = Vec::new();

                for lat in (bbox.min_lat.floor() as i32)..=(bbox.max_lat.ceil() as i32) {
                    for lon in (bbox.min_lon.floor() as i32)..=(bbox.max_lon.ceil() as i32) {
                        let tile_name = format!(
                            "{}{:02}{}{:03}.hgt",
                            if lat >= 0 { "N" } else { "S" }, lat.abs(),
                            if lon >= 0 { "E" } else { "W" }, lon.abs(),
                        );

                        // Check local cache first
                        let cache_path = format!("{}/{}", self.srtm_cache_path, tile_name);
                        if !std::path::Path::new(&cache_path).exists() {
                            // Download from NASA EarthData or CGIAR mirror
                            self.download_srtm_tile(&tile_name, &cache_path).await?;
                        }

                        tiles.push(Dataset::open(&cache_path)?);
                    }
                }

                Ok(tiles)
            }
            _ => anyhow::bail!("Source {:?} not implemented yet", source),
        }
    }

    fn compute_slope(&self, dem: &Dataset) -> anyhow::Result<Dataset> {
        // Use GDAL DEMProcessing for slope calculation
        let options = gdal::raster::processing::dem::options::SlopeOptions {
            of: Some("GTiff".to_string()),
            compute_edges: true,
            ..Default::default()
        };

        let output_path = "/tmp/slope.tif";
        gdal::raster::processing::dem::slope(dem, output_path, &options)?;
        Ok(Dataset::open(output_path)?)
    }
}

#[derive(Debug)]
pub struct TerrainIngestionResult {
    pub dem_url: String,
    pub resolution_m: f32,
    pub source: String,
    pub bbox: BoundingBox,
    pub statistics: TerrainStatistics,
}

#[derive(Debug)]
pub struct BoundingBox {
    pub min_lon: f64,
    pub max_lon: f64,
    pub min_lat: f64,
    pub max_lat: f64,
}

#[derive(Debug)]
pub struct TerrainStatistics {
    pub min_elevation_m: f32,
    pub max_elevation_m: f32,
    pub mean_elevation_m: f32,
    pub max_slope_degrees: f32,
}

pub enum TerrainResolution {
    Coarse,   // 30m (SRTM)
    Medium,   // 10m (ASTER)
    Fine,     // 1m (lidar if available)
}

#[derive(Debug, Clone, Copy)]
pub enum TerrainSource {
    Srtm30m,
    Aster30m,
    LidarIfAvailable,
}

impl ToString for TerrainSource {
    fn to_string(&self) -> String {
        match self {
            Self::Srtm30m => "srtm".to_string(),
            Self::Aster30m => "aster".to_string(),
            Self::LidarIfAvailable => "lidar".to_string(),
        }
    }
}
```

### 2. AI Downscaling Service (Python FastAPI + PyTorch)

Microservice that runs trained U-Net models to downscale coarse NWP forecasts to high-resolution microclimate grids.

```python
# ml-service/main.py

from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import torch
import numpy as np
from typing import List, Optional
import xarray as xr
from datetime import datetime
import logging

app = FastAPI(title="ClimaCast ML Downscaling Service")

# Load pre-trained models on startup
downscaling_models = {}

@app.on_event("startup")
async def load_models():
    global downscaling_models

    device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
    logging.info(f"Loading models on device: {device}")

    # Temperature downscaling model
    downscaling_models["temperature"] = torch.jit.load(
        "models/temperature_unet.pt", map_location=device
    )
    downscaling_models["temperature"].eval()

    # Wind downscaling model
    downscaling_models["wind"] = torch.jit.load(
        "models/wind_unet.pt", map_location=device
    )
    downscaling_models["wind"].eval()

    # Precipitation downscaling model
    downscaling_models["precipitation"] = torch.jit.load(
        "models/precip_unet.pt", map_location=device
    )
    downscaling_models["precipitation"].eval()

    logging.info("All models loaded successfully")


class DownscalingRequest(BaseModel):
    coarse_data_url: str  # S3 URL to coarse NWP data (NetCDF)
    terrain_dem_url: str  # S3 URL to high-res terrain DEM (GeoTIFF)
    bbox: dict  # {min_lat, max_lat, min_lon, max_lon}
    target_resolution_m: float  # Desired output resolution (10-100m)
    parameters: List[str]  # ["temperature", "wind_speed", "precipitation"]
    forecast_times: List[str]  # ISO8601 timestamps


class DownscalingResponse(BaseModel):
    downscaled_data_url: str  # S3 URL to output NetCDF
    resolution_m: float
    grid_shape: dict  # {ny, nx}
    forecast_times: List[str]
    processing_time_seconds: float


@app.post("/downscale", response_model=DownscalingResponse)
async def downscale_forecast(request: DownscalingRequest):
    """
    Downscale coarse NWP forecast to high-resolution microclimate grid.

    Process:
    1. Load coarse NWP data (GFS 9km or HRRR 3km)
    2. Load terrain DEM and compute terrain features
    3. For each parameter and time:
       - Extract coarse field
       - Extract terrain features (elevation, slope, aspect, roughness)
       - Stack as multi-channel input tensor
       - Run through U-Net to produce fine-resolution output
    4. Save as NetCDF and upload to S3
    """
    import time
    start_time = time.time()

    try:
        # 1. Load coarse NWP data from S3
        coarse_ds = xr.open_dataset(request.coarse_data_url, engine="h5netcdf")

        # 2. Load terrain DEM
        import rasterio
        with rasterio.open(request.terrain_dem_url) as src:
            terrain_elevation = src.read(1)
            terrain_transform = src.transform
            terrain_crs = src.crs

        # 3. Compute terrain derivatives
        terrain_features = compute_terrain_features(
            terrain_elevation,
            terrain_transform,
            target_resolution_m=request.target_resolution_m
        )

        # 4. Create output grid
        ny, nx = terrain_features["elevation"].shape
        output_shape = (len(request.forecast_times), len(request.parameters), ny, nx)
        output_data = np.zeros(output_shape, dtype=np.float32)

        device = torch.device("cuda" if torch.cuda.is_available() else "cpu")

        # 5. Downscale each parameter and time
        for t_idx, forecast_time in enumerate(request.forecast_times):
            dt = datetime.fromisoformat(forecast_time)

            for p_idx, param in enumerate(request.parameters):
                # Extract coarse field at this time
                coarse_field = extract_coarse_field(
                    coarse_ds, param, dt, request.bbox
                )

                # Prepare input tensor: [batch, channels, height, width]
                # Channels: [coarse_field, elevation, slope, aspect, roughness, lat, lon]
                input_tensor = prepare_input_tensor(
                    coarse_field,
                    terrain_features,
                    request.bbox,
                    device
                )

                # Run model inference
                model_key = get_model_key(param)
                with torch.no_grad():
                    output_tensor = downscaling_models[model_key](input_tensor)

                # Extract and denormalize
                downscaled_field = output_tensor.cpu().numpy()[0, 0, :, :]
                downscaled_field = denormalize(downscaled_field, param)

                output_data[t_idx, p_idx, :, :] = downscaled_field

        # 6. Create output NetCDF
        output_ds = create_output_netcdf(
            output_data,
            request.parameters,
            request.forecast_times,
            terrain_transform,
            terrain_crs,
            request.target_resolution_m
        )

        # 7. Save and upload to S3
        import uuid
        output_id = str(uuid.uuid4())
        output_path = f"/tmp/downscaled_{output_id}.nc"
        output_ds.to_netcdf(output_path, engine="h5netcdf")

        # Upload to S3 (using boto3)
        import boto3
        s3_client = boto3.client("s3")
        s3_key = f"downscaled/{output_id}.nc"
        s3_client.upload_file(output_path, "climacast-results", s3_key)

        processing_time = time.time() - start_time

        return DownscalingResponse(
            downscaled_data_url=f"s3://climacast-results/{s3_key}",
            resolution_m=request.target_resolution_m,
            grid_shape={"ny": ny, "nx": nx},
            forecast_times=request.forecast_times,
            processing_time_seconds=round(processing_time, 2)
        )

    except Exception as e:
        logging.error(f"Downscaling failed: {e}")
        raise HTTPException(status_code=500, detail=str(e))


def compute_terrain_features(elevation, transform, target_resolution_m):
    """Compute slope, aspect, roughness from elevation DEM"""
    import richdem as rd

    # Convert to richdem array
    dem = rd.rdarray(elevation, no_data=-9999)

    # Compute derivatives
    slope = rd.TerrainAttribute(dem, attrib='slope_degrees')
    aspect = rd.TerrainAttribute(dem, attrib='aspect')

    # Compute roughness as std dev of elevation in 3x3 window
    from scipy.ndimage import generic_filter
    roughness = generic_filter(elevation, np.std, size=3)

    # Create lat/lon grids
    ny, nx = elevation.shape
    lons = np.zeros((ny, nx))
    lats = np.zeros((ny, nx))

    for i in range(ny):
        for j in range(nx):
            lon, lat = transform * (j, i)
            lons[i, j] = lon
            lats[i, j] = lat

    return {
        "elevation": elevation.astype(np.float32),
        "slope": np.array(slope).astype(np.float32),
        "aspect": np.array(aspect).astype(np.float32),
        "roughness": roughness.astype(np.float32),
        "lat": lats.astype(np.float32),
        "lon": lons.astype(np.float32),
    }


def prepare_input_tensor(coarse_field, terrain_features, bbox, device):
    """
    Prepare multi-channel input tensor for U-Net.

    Channels:
    0: Coarse field (upsampled via bilinear interpolation)
    1: Elevation (normalized)
    2: Slope
    3: Aspect (sin/cos encoded)
    4: Roughness
    5: Latitude (normalized)
    6: Longitude (normalized)
    """
    # Upsample coarse field to match terrain grid
    from scipy.ndimage import zoom
    ny, nx = terrain_features["elevation"].shape
    cy, cx = coarse_field.shape
    zoom_factors = (ny / cy, nx / cx)
    coarse_upsampled = zoom(coarse_field, zoom_factors, order=1)

    # Normalize features
    elevation_norm = (terrain_features["elevation"] - terrain_features["elevation"].mean()) / terrain_features["elevation"].std()
    slope_norm = terrain_features["slope"] / 90.0  # Degrees to [0, 1]
    aspect_sin = np.sin(np.radians(terrain_features["aspect"]))
    aspect_cos = np.cos(np.radians(terrain_features["aspect"]))
    roughness_norm = (terrain_features["roughness"] - terrain_features["roughness"].mean()) / (terrain_features["roughness"].std() + 1e-6)
    lat_norm = (terrain_features["lat"] - bbox["min_lat"]) / (bbox["max_lat"] - bbox["min_lat"])
    lon_norm = (terrain_features["lon"] - bbox["min_lon"]) / (bbox["max_lon"] - bbox["min_lon"])

    # Stack channels
    input_array = np.stack([
        coarse_upsampled,
        elevation_norm,
        slope_norm,
        aspect_sin,
        aspect_cos,
        roughness_norm,
        lat_norm,
        lon_norm,
    ], axis=0)

    # Convert to torch tensor [1, channels, height, width]
    input_tensor = torch.from_numpy(input_array).unsqueeze(0).float().to(device)

    return input_tensor


def get_model_key(parameter: str) -> str:
    """Map parameter name to model key"""
    if parameter in ["temperature", "temperature_2m"]:
        return "temperature"
    elif parameter in ["wind_speed", "wind_u", "wind_v"]:
        return "wind"
    elif parameter in ["precipitation", "precip_rate"]:
        return "precipitation"
    else:
        return "temperature"  # Default fallback


def denormalize(data, parameter):
    """Denormalize model output to physical units"""
    # Model outputs are normalized to [-1, 1] or [0, 1] during training
    # This reverses that normalization
    # (Parameters specific to training setup)
    if parameter == "temperature":
        return data * 40 + 260  # Kelvin range [220, 300]
    elif parameter == "wind_speed":
        return data * 30  # m/s range [0, 30]
    elif parameter == "precipitation":
        return np.maximum(data, 0) * 50  # mm/hr range [0, 50]
    else:
        return data


@app.get("/health")
async def health_check():
    return {"status": "healthy", "models_loaded": len(downscaling_models)}
```

### 3. CFD Simulation Orchestrator (Rust)

Backend service that prepares OpenFOAM case files, submits Kubernetes pods, monitors progress, and processes results.

```rust
// src/services/cfd.rs

use k8s_openapi::api::core::v1::Pod;
use kube::{Api, Client as K8sClient};
use kube::api::{PostParams, DeleteParams, LogParams};
use uuid::Uuid;
use sqlx::PgPool;
use aws_sdk_s3::Client as S3Client;
use std::path::PathBuf;

pub struct CfdOrchestrator {
    k8s: K8sClient,
    db: PgPool,
    s3: S3Client,
}

impl CfdOrchestrator {
    pub async fn submit_wind_simulation(
        &self,
        simulation_id: Uuid,
        site_model: &crate::db::models::SiteModel,
        config: &crate::db::models::WindCfdConfig,
    ) -> anyhow::Result<Uuid> {
        // 1. Create CFD job record
        let job = sqlx::query_as!(
            crate::db::models::CfdJob,
            r#"INSERT INTO cfd_jobs (simulation_id, cores_allocated, memory_mb, priority)
               VALUES ($1, $2, $3, $4) RETURNING *"#,
            simulation_id,
            self.determine_cores(&site_model.terrain_resolution_m),
            self.determine_memory(&site_model.terrain_resolution_m),
            0, // Priority from user plan
        )
        .fetch_one(&self.db)
        .await?;

        // 2. Prepare OpenFOAM case directory structure
        let case_path = self.prepare_case_directory(simulation_id, site_model, config).await?;

        // 3. Generate mesh (snappyHexMesh)
        self.update_status(simulation_id, "meshing").await?;
        self.run_meshing(&case_path, site_model, config).await?;

        // 4. Upload mesh to S3
        let mesh_url = self.upload_mesh(simulation_id, &case_path).await?;
        sqlx::query!(
            "UPDATE simulations SET mesh_url = $2 WHERE id = $1",
            simulation_id, mesh_url
        )
        .execute(&self.db)
        .await?;

        // 5. Submit Kubernetes pod for solver
        self.update_status(simulation_id, "solving").await?;
        let pod_name = self.submit_solver_pod(job.id, simulation_id, &case_path, config).await?;

        sqlx::query!(
            "UPDATE cfd_jobs SET pod_name = $2, started_at = NOW() WHERE id = $1",
            job.id, pod_name
        )
        .execute(&self.db)
        .await?;

        Ok(job.id)
    }

    async fn prepare_case_directory(
        &self,
        simulation_id: Uuid,
        site_model: &crate::db::models::SiteModel,
        config: &crate::db::models::WindCfdConfig,
    ) -> anyhow::Result<PathBuf> {
        let case_path = PathBuf::from(format!("/tmp/cfd/{}", simulation_id));
        tokio::fs::create_dir_all(&case_path).await?;

        // Create OpenFOAM directory structure
        for dir in &["0", "constant", "constant/triSurface", "system"] {
            tokio::fs::create_dir_all(case_path.join(dir)).await?;
        }

        // Download terrain DEM from S3
        let dem_local = case_path.join("terrain.tif");
        self.download_from_s3(&site_model.terrain_dem_url, &dem_local).await?;

        // Convert terrain + buildings to STL for snappyHexMesh
        self.generate_geometry_stl(
            &dem_local,
            &site_model.buildings,
            &case_path.join("constant/triSurface/geometry.stl")
        ).await?;

        // Write initial conditions (0/ directory)
        self.write_initial_conditions(&case_path, config)?;

        // Write control dict (system/controlDict)
        self.write_control_dict(&case_path, config)?;

        // Write boundary conditions (0/U, 0/p, 0/k, 0/epsilon)
        self.write_boundary_conditions(&case_path, config)?;

        // Write mesh dict (system/snappyHexMeshDict)
        self.write_snappy_mesh_dict(&case_path, site_model, config)?;

        // Write solver settings (system/fvSchemes, system/fvSolution)
        self.write_solver_settings(&case_path, config)?;

        Ok(case_path)
    }

    fn write_initial_conditions(
        &self,
        case_path: &PathBuf,
        config: &crate::db::models::WindCfdConfig,
    ) -> anyhow::Result<()> {
        use std::io::Write;

        // 0/U — Velocity field
        let u_content = format!(r#"
FoamFile
{{
    version     2.0;
    format      ascii;
    class       volVectorField;
    object      U;
}}

dimensions      [0 1 -1 0 0 0 0];

internalField   uniform ({} 0 0);  // Wind from west

boundaryField
{{
    inlet
    {{
        type            atmBoundaryLayerInletVelocity;
        Uref            {};
        Zref            {};
        z0              uniform 0.1;
        flowDir         (1 0 0);
        zGround         uniform 0.0;
        value           uniform ({} 0 0);
    }}

    outlet
    {{
        type            inletOutlet;
        inletValue      uniform (0 0 0);
        value           uniform ({} 0 0);
    }}

    ground
    {{
        type            noSlip;
    }}

    top
    {{
        type            slip;
    }}

    sides
    {{
        type            symmetryPlane;
    }}
}}
"#,
            config.reference_wind_speed,
            config.reference_wind_speed,
            config.reference_height,
            config.reference_wind_speed,
            config.reference_wind_speed,
        );

        let mut file = std::fs::File::create(case_path.join("0/U"))?;
        file.write_all(u_content.as_bytes())?;

        // Similar for 0/p, 0/k, 0/epsilon...
        // (Omitted for brevity)

        Ok(())
    }

    async fn run_meshing(
        &self,
        case_path: &PathBuf,
        site_model: &crate::db::models::SiteModel,
        config: &crate::db::models::WindCfdConfig,
    ) -> anyhow::Result<()> {
        // Run blockMesh to create background mesh
        let blockmesh_output = tokio::process::Command::new("blockMesh")
            .arg("-case")
            .arg(case_path)
            .output()
            .await?;

        if !blockmesh_output.status.success() {
            anyhow::bail!("blockMesh failed: {}",
                String::from_utf8_lossy(&blockmesh_output.stderr));
        }

        // Run snappyHexMesh to refine around terrain and buildings
        let snappy_output = tokio::process::Command::new("snappyHexMesh")
            .arg("-case")
            .arg(case_path)
            .arg("-overwrite")
            .output()
            .await?;

        if !snappy_output.status.success() {
            anyhow::bail!("snappyHexMesh failed: {}",
                String::from_utf8_lossy(&snappy_output.stderr));
        }

        // Extract mesh statistics
        let check_output = tokio::process::Command::new("checkMesh")
            .arg("-case")
            .arg(case_path)
            .output()
            .await?;

        let check_log = String::from_utf8_lossy(&check_output.stdout);
        let cell_count = self.extract_cell_count(&check_log)?;

        tracing::info!("Mesh generated: {} cells", cell_count);

        Ok(())
    }

    async fn submit_solver_pod(
        &self,
        job_id: Uuid,
        simulation_id: Uuid,
        case_path: &PathBuf,
        config: &crate::db::models::WindCfdConfig,
    ) -> anyhow::Result<String> {
        // Archive case directory
        let tarball_path = format!("/tmp/{}.tar.gz", simulation_id);
        self.create_tarball(case_path, &tarball_path).await?;

        // Upload to S3 for pod to download
        let s3_key = format!("cfd-cases/{}.tar.gz", simulation_id);
        self.upload_file_to_s3(&tarball_path, &s3_key).await?;

        // Create Kubernetes pod spec
        let pod_name = format!("cfd-{}", simulation_id);
        let pod_spec = self.create_pod_spec(
            &pod_name,
            simulation_id,
            job_id,
            &s3_key,
            config,
        );

        let pods: Api<Pod> = Api::namespaced(self.k8s.clone(), "climacast");
        pods.create(&PostParams::default(), &pod_spec).await?;

        tracing::info!("Submitted pod: {}", pod_name);

        Ok(pod_name)
    }

    fn create_pod_spec(
        &self,
        pod_name: &str,
        simulation_id: Uuid,
        job_id: Uuid,
        case_s3_key: &str,
        config: &crate::db::models::WindCfdConfig,
    ) -> Pod {
        use k8s_openapi::api::core::v1::*;
        use k8s_openapi::apimachinery::pkg::api::resource::Quantity;

        let solver = match config.turbulence_model.as_str() {
            "kOmegaSST" => "simpleFoam",
            _ => "simpleFoam",
        };

        Pod {
            metadata: kube::api::ObjectMeta {
                name: Some(pod_name.to_string()),
                labels: Some([
                    ("app".to_string(), "cfd-solver".to_string()),
                    ("simulation_id".to_string(), simulation_id.to_string()),
                ].iter().cloned().collect()),
                ..Default::default()
            },
            spec: Some(PodSpec {
                restart_policy: Some("Never".to_string()),
                containers: vec![Container {
                    name: "openfoam".to_string(),
                    image: Some("climacast/openfoam:11".to_string()),
                    command: Some(vec!["/bin/bash".to_string()]),
                    args: Some(vec![
                        "-c".to_string(),
                        format!(
                            "aws s3 cp s3://climacast-cases/{} /case.tar.gz && \
                             tar -xzf /case.tar.gz -C /case && \
                             cd /case && \
                             {} > /case/log.{} 2>&1 && \
                             aws s3 cp --recursive /case s3://climacast-results/cfd/{}/ && \
                             curl -X POST http://api-service/simulations/{}/complete",
                            case_s3_key, solver, solver, simulation_id, simulation_id
                        ),
                    ]),
                    resources: Some(ResourceRequirements {
                        requests: Some([
                            ("cpu".to_string(), Quantity("4000m".to_string())),
                            ("memory".to_string(), Quantity("8Gi".to_string())),
                        ].iter().cloned().collect()),
                        limits: Some([
                            ("cpu".to_string(), Quantity("8000m".to_string())),
                            ("memory".to_string(), Quantity("16Gi".to_string())),
                        ].iter().cloned().collect()),
                    }),
                    ..Default::default()
                }],
                node_selector: Some([
                    ("workload-type".to_string(), "cfd".to_string()),
                ].iter().cloned().collect()),
                ..Default::default()
            }),
            ..Default::default()
        }
    }

    fn determine_cores(&self, resolution_m: &f32) -> i32 {
        if *resolution_m < 5.0 {
            16  // Fine mesh
        } else if *resolution_m < 20.0 {
            8   // Medium mesh
        } else {
            4   // Coarse mesh
        }
    }

    fn determine_memory(&self, resolution_m: &f32) -> i32 {
        if *resolution_m < 5.0 {
            32768  // 32 GB
        } else if *resolution_m < 20.0 {
            16384  // 16 GB
        } else {
            8192   // 8 GB
        }
    }

    async fn update_status(&self, simulation_id: Uuid, status: &str) -> anyhow::Result<()> {
        sqlx::query!(
            "UPDATE simulations SET status = $2 WHERE id = $1",
            simulation_id, status
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
cargo init climacast-api
cd climacast-api
cargo add axum tokio serde serde_json sqlx uuid chrono tower-http jsonwebtoken bcrypt
cargo add --features postgres,runtime-tokio-rustls,uuid,chrono,json sqlx
cargo add aws-sdk-s3 redis geojson gdal-sys
```
- `src/main.rs` — Axum app with CORS, tracing, graceful shutdown, health check endpoint
- `src/config.rs` — Environment-based configuration (DATABASE_URL, JWT_SECRET, S3_BUCKET, REDIS_URL, K8S_NAMESPACE)
- `src/state.rs` — AppState struct (PgPool, Redis, S3Client, K8sClient)
- `src/error.rs` — ApiError enum with IntoResponse for consistent error handling
- `Dockerfile` — Multi-stage build (Rust builder + runtime with GDAL libraries)
- `docker-compose.yml` — PostgreSQL 16 + PostGIS 3.4, Redis 7, MinIO (S3-compatible), TimescaleDB extension

**Day 2: Database schema and PostGIS setup**
- `migrations/001_initial.sql` — All 11 tables with PostGIS geometry columns and TimescaleDB hypertables
- Enable PostGIS extension, TimescaleDB extension, pg_trgm for fuzzy search
- Create spatial indexes (GIST) on site_boundary, site_center, weather station locations
- Create TimescaleDB hypertables for forecast_data and station_observations with 1-day and 7-day chunks
- `src/db/mod.rs` — Database pool initialization with connection pooling
- `src/db/models.rs` — All SQLx structs with FromRow derives, GeoJSON geometry handling

**Day 3: Authentication system**
- `src/auth/mod.rs` — JWT token generation and validation middleware
- `src/auth/oauth.rs` — Google and GitHub OAuth 2.0 flow handlers with PKCE
- `src/api/handlers/auth.rs` — Register, login, OAuth callback, refresh token, get current user endpoints
- Password hashing with bcrypt (cost factor 12)
- JWT with 24h access token + 30d refresh token, claims include user_id and plan
- Auth middleware extractor that validates Bearer tokens and injects Claims into handlers

**Day 4: User and organization CRUD**
- `src/api/handlers/users.rs` — Get profile, update profile, change password, delete account
- `src/api/handlers/orgs.rs` — Create organization, invite member (email invitation), list members, update member role, remove member
- `src/api/handlers/projects.rs` — Create project (with GeoJSON boundary validation), list projects, get project, update project metadata, delete project
- `src/api/router.rs` — All route definitions with auth middleware applied where needed
- Integration tests: full auth flow, project CRUD with PostGIS geometry queries, organization membership

**Day 5: Terrain service foundation and GDAL integration**
- `src/services/terrain.rs` — TerrainService struct with SRTM tile fetching, DEM mosaicking, clipping to boundary
- GDAL bindings setup for GeoTIFF reading/writing, raster operations
- SRTM tile cache directory with automatic download from NASA EarthData mirror
- Compute bounding box from GeoJSON polygon geometry
- Basic terrain statistics (min/max/mean elevation, slope range)
- Unit tests: terrain ingestion for known coordinates, verify GeoTIFF output

### Phase 2 — ML Downscaling Service (Days 6–10)

**Day 6: Python ML service scaffold and model loading**
```bash
cd ..
mkdir ml-service && cd ml-service
python -m venv venv
source venv/bin/activate
pip install fastapi uvicorn torch torchvision onnxruntime-gpu xarray netcdf4 h5netcdf rasterio richdem scipy numpy boto3
```
- `main.py` — FastAPI app with /downscale endpoint, model loading on startup
- `models/` — Directory for pre-trained .pt or .onnx model files
- Load temperature, wind, and precipitation downscaling models (U-Net architecture) onto GPU
- Health check endpoint returning model status and GPU availability
- Docker setup with CUDA base image for GPU support

**Day 7: Weather data ingestion pipeline**
- `ingest.py` — Fetch GFS forecasts from NOAA NOMADS server (GRIB2 format)
- Convert GRIB2 to NetCDF using cfgrib, extract relevant parameters (temperature_2m, wind_u_10m, wind_v_10m, precipitation_rate)
- Clip to project bounding box with buffer for boundary conditions
- Upload to S3 as NetCDF file
- Scheduled task (cron or Celery beat) to fetch new GFS runs every 6 hours

**Day 8: Terrain feature extraction for ML inputs**
- `terrain_utils.py` — Load GeoTIFF DEM, compute slope, aspect, hillshade, roughness using richdem
- Create lat/lon coordinate grids matching DEM resolution
- Normalize all features to [0, 1] or [-1, 1] range for model input
- Unit tests: verify slope/aspect calculations against known test DEMs

**Day 9: Downscaling inference pipeline**
- `downscale.py` — Core downscaling logic: upsample coarse field, stack with terrain features, run U-Net inference
- Multi-channel input tensor preparation: [coarse_field, elevation, slope, aspect_sin, aspect_cos, roughness, lat_norm, lon_norm]
- Batch processing for multiple timesteps and parameters
- Denormalization of model outputs to physical units (K, m/s, mm/hr)
- Output NetCDF with CF-compliant metadata

**Day 10: Integration with Rust backend**
- Rust service client: `src/services/ml_client.rs` — HTTP client to call Python ML service /downscale endpoint
- Background job: when forecast subscription is active, trigger downscaling for project domain every 6 hours
- Parse NetCDF output, extract grid points, insert into forecast_data TimescaleDB table
- Error handling: retry logic with exponential backoff, alert on persistent failures
- Integration test: end-to-end project creation → forecast subscription → downscaling → data retrieval

### Phase 3 — CFD Simulation Pipeline (Days 11–18)

**Day 11: OpenFOAM Docker image and Kubernetes setup**
- `docker/openfoam/Dockerfile` — OpenFOAM 11 installation with snappyHexMesh, simpleFoam, pimpleFoam
- Include AWS CLI for S3 access, Python for post-processing scripts
- Kubernetes namespace and RBAC for pod creation from Rust backend
- Karpenter provisioner config for spot instances with CFD workload tolerations
- Test: manually submit a simple OpenFOAM case pod, verify completion and log retrieval

**Day 12: Geometry generation (terrain + buildings to STL)**
- `src/services/geometry.rs` — Convert terrain DEM raster + building GeoJSON to STL triangulated surface
- Use GDAL contour generation for terrain surface at regular intervals, convert to mesh
- Extrude building footprints to height to create building STL geometries
- Merge terrain and building STLs into single geometry.stl for snappyHexMesh
- Validate STL: check for holes, non-manifold edges, flipped normals using MeshLab headless

**Day 13: snappyHexMesh configuration and meshing**
- `src/services/cfd.rs` — Write snappyHexMeshDict with castellated mesh, snap, and layer addition phases
- Define refinement regions: fine mesh near buildings and terrain features, coarse mesh at domain boundaries
- blockMesh background grid sized based on domain extent and target resolution
- Run snappyHexMesh in parallel (4-8 cores) with log parsing to extract mesh statistics (cell count, quality metrics)
- Extract final mesh cell count and update simulations table

**Day 14: Boundary conditions and initial conditions**
- Atmospheric boundary layer velocity profile (logarithmic wind profile based on roughness length)
- Inlet: atmBoundaryLayerInletVelocity with specified Uref, Zref, z0
- Outlet: inletOutlet (zero-gradient outflow)
- Ground and building surfaces: noSlip (zero velocity)
- Top: slip (symmetry)
- Pressure: zeroGradient on all boundaries except outlet (fixed value)
- Turbulence: k-epsilon or k-omega SST with appropriate inlet turbulence intensity

**Day 15: Solver execution orchestration**
- Kubernetes pod submission from Rust with case tarball uploaded to S3
- Pod startup script: download case, run simpleFoam (steady-state RANS), upload results
- Real-time log streaming from pod to extract residuals and iteration count
- Parse residuals (U_x, U_y, U_z, p, k, epsilon) and publish to Redis for WebSocket streaming
- Update cfd_jobs table with progress_pct based on convergence

**Day 16: Result post-processing and extraction**
- Extract wind field at specified heights (e.g., 1.5m pedestrian level, 10m, 50m)
- Convert OpenFOAM field files to VTK for ParaView compatibility
- Generate vector tiles (Mapbox Vector Tiles) for wind field visualization in frontend
- Compute derived quantities: wind speed magnitude, turbulence intensity, wind comfort index (Lawson criteria)
- Upload all result files to S3, store S3 prefix in simulations.results_url

**Day 17: Wind comfort scoring**
- Implement Lawson criteria: categorize areas as suitable for sitting, standing, walking, uncomfortable, or dangerous based on mean wind speed and gust frequency
- Generate comfort classification raster (GeoTIFF) with categories as integer codes
- Dutch NEN 8100 wind nuisance standard as alternative criteria
- Store comfort summary in simulations.results_summary JSON (e.g., "65% of area suitable for sitting")
- Unit tests: verify comfort classification against published benchmark cases

**Day 18: Multi-direction wind rose analysis**
- For projects with full 16-direction analysis, run separate CFD simulations for each cardinal and intercardinal direction
- Weight results by wind frequency (from downscaled forecast or climatological data)
- Aggregate comfort scores and safety metrics across all directions
- Parallel execution: submit all 16 direction cases simultaneously, merge results after all complete
- Performance target: complete 16-direction analysis for 1km × 1km medium-resolution domain in <3 hours

### Phase 4 — Frontend Foundation (Days 19–24)

**Day 19: React scaffold and 3D map setup**
```bash
npm create vite@latest frontend -- --template react-ts
cd frontend
npm i zustand @tanstack/react-query axios mapbox-gl @deck.gl/core @deck.gl/layers @deck.gl/react
npm i -D @types/mapbox-gl
```
- `src/App.tsx` — React Router, auth context, protected routes
- `src/stores/authStore.ts` — Zustand store for auth state (user, token, login, logout)
- `src/stores/projectStore.ts` — Zustand store for project list and active project
- `src/components/Map/MapView.tsx` — Mapbox GL JS 3D terrain map with navigation controls
- Initialize with default view (user's location or world view), terrain-rgb DEM layer for 3D relief
- Map interaction: click to select location, draw polygon tool for site boundary

**Day 20: Site boundary drawing and project creation**
- `src/components/Map/DrawControl.tsx` — Mapbox Draw plugin for polygon drawing
- `src/components/ProjectCreate/ProjectWizard.tsx` — Multi-step wizard: draw boundary → set name/type → configure domain
- Domain configuration inputs: resolution (dropdown: coarse/medium/fine), terrain source (auto/SRTM/ASTER)
- Submit project creation API call with GeoJSON boundary
- Display loading indicator during terrain ingestion, show success with project card

**Day 21: Project dashboard and list view**
- `src/pages/Dashboard.tsx` — Grid of project cards with thumbnail, name, type badge, last updated
- Filter/sort: by project type, by recent activity, search by name
- Quick actions on card: open project, run simulation, view results, share, delete
- Map view mode: show all projects as pins/polygons on global map, click to open
- Empty state with "Create First Project" CTA

**Day 22: Site model viewer**
- `src/components/SiteModel/SiteModelViewer.tsx` — 3D map showing project boundary with terrain and buildings extruded
- Layer toggles: terrain contours, buildings, land use classification (if available), wind stations
- Building info panel: click building to see height, footprint area, OSM metadata
- Terrain statistics panel: elevation range, slope range, aspect distribution chart
- Edit mode: manually add/edit/delete building footprints (for buildings missing from OSM)

**Day 23: Simulation configuration UI**
- `src/components/Simulation/SimulationConfig.tsx` — Tabbed interface for different simulation types (Wind CFD, Solar, Frost, Comprehensive)
- Wind CFD tab: select wind directions (single, 4-direction, 8-direction, 16-direction), reference wind speed, turbulence model
- Solar tab: date range picker (annual, monthly, or custom), time step slider
- Frost tab: forecast period, crop type selection, critical temperature threshold
- Estimated compute time and cost (in credits or minutes) displayed before submission
- Advanced panel: mesh refinement zones (draw rectangles for finer mesh), solver tolerances

**Day 24: Simulation status and progress tracking**
- `src/components/Simulation/SimulationList.tsx` — Table of simulations for current project with status badges (queued, meshing, solving, completed, failed)
- Real-time updates via WebSocket: progress bar animates as solver advances
- Click simulation row to expand details: config summary, mesh statistics, convergence plot
- `src/hooks/useSimulationProgress.ts` — React hook for WebSocket subscription to simulation progress events
- Cancel simulation button (calls API to delete pod and mark as cancelled)
- Retry failed simulation with same configuration

### Phase 5 — WASM Solar and Frost Analysis (Days 25–28)

**Day 25: WASM module scaffold for client-side computation**
```bash
cd ..
cargo new --lib climacast-wasm
cd climacast-wasm
cargo add wasm-bindgen serde serde-wasm-bindgen js-sys chrono geojson
```
- `Cargo.toml` — `[lib] crate-type = ["cdylib", "rlib"]`, wasm-bindgen dependencies
- `src/lib.rs` — WASM entry points with #[wasm_bindgen] attribute
- `src/solar.rs` — Sun position calculation (azimuth, elevation) using SPA algorithm
- `src/frost.rs` — DEM-based frost pocket identification (cold air drainage, slope analysis)
- Build with `wasm-pack build --target web --release`, optimize with wasm-opt

**Day 26: Solar irradiance computation**
- `src/solar.rs` — Compute sun position for given lat/lon/date/time
- Ray-tracing shadow detection: for each grid cell, cast ray toward sun, check intersection with terrain or buildings
- Direct normal irradiance (DNI) calculation with atmospheric attenuation (simple clear-sky model)
- Diffuse horizontal irradiance (DHI) estimation from sky model
- Global horizontal irradiance (GHI) = DNI * cos(zenith) + DHI
- Accumulate over time range to produce annual or monthly totals
- Output: 2D grid of irradiance values (kWh/m²)

**Day 27: Building shadow animation**
- `src/solar.rs` — For each hour of day, compute shadow polygons cast by buildings
- Shadow polygon algorithm: project building corners along solar vector to ground plane
- Rasterize shadow polygons to grid
- Frontend: `src/components/Solar/ShadowAnimation.tsx` — Animated visualization with time slider
- WebGL shader for shadow overlay with transparency
- Export animation as MP4 using canvas-to-video library (optional premium feature)

**Day 28: Frost risk heatmap**
- `src/frost.rs` — Identify frost pockets using DEM slope and aspect analysis
- Cold air drainage: cells with downslope neighbors are frost-prone, cells on ridges are frost-safe
- Compute frost risk score: combination of elevation (lower = higher risk), slope (flat = higher risk), aspect (north-facing = higher risk)
- Apply downscaled temperature forecast: cells where T < critical threshold flagged as high risk
- Output: categorical frost risk map (low, medium, high, extreme)
- Frontend: `src/components/Frost/FrostHeatmap.tsx` — Color-coded overlay on map with legend

### Phase 6 — Results Visualization (Days 29–33)

**Day 29: Wind field visualization with Deck.gl**
- `src/components/Results/WindFieldLayer.tsx` — Deck.gl GridLayer for wind speed heatmap
- Color scale: blue (low wind) → yellow (moderate) → red (high wind)
- `src/components/Results/WindVectorLayer.tsx` — Deck.gl ArrowLayer for wind direction vectors
- Vector length proportional to wind speed magnitude
- Layer controls: toggle heatmap, toggle vectors, adjust opacity, select height slice (1.5m, 10m, 50m)
- Performance: use LOD (level of detail) to decimate vectors at low zoom levels

**Day 30: Animated wind flow with particles**
- `src/components/Results/ParticleLayer.tsx` — Custom WebGL particle system following wind vectors
- 10,000 particles initialized at random positions, advected each frame using bilinear interpolation of wind field
- Particle trails with fade-out for motion blur effect
- Adjustable particle density and speed multiplier
- Click-and-hold to inject particles at cursor position
- Inspiration: earth.nullschool.net wind visualization

**Day 31: Solar irradiance heatmap and PV yield estimation**
- `src/components/Results/SolarHeatmap.tsx` — Color-coded annual solar irradiance (kWh/m²/year)
- Click any cell to see detailed breakdown: monthly totals, peak/average DNI, shading hours
- PV panel yield estimator: user draws rectangle for panel array, specifies tilt/azimuth, module efficiency
- Calculate annual energy yield (kWh/year) and capacity factor
- Integration with PVWatts calculator API for detailed system modeling (optional)

**Day 32: Time-series data extraction and charting**
- `src/components/Results/ProbeChart.tsx` — Click any location to extract time-series data (temperature, wind speed, precipitation forecast)
- Multi-series line chart using Recharts or Visx
- Date range selector, zoom/pan on chart
- Compare multiple locations: click multiple points, overlay time-series on same chart
- Export chart data as CSV

**Day 33: Report generation UI**
- `src/components/Report/ReportBuilder.tsx` — Select simulation(s) to include, choose template type
- Template types: Wind Comfort Assessment, Solar Access Study, Frost Risk Analysis, Construction Weather Plan, Comprehensive
- White-label branding: upload logo, set company name and colors (Team plan only)
- Preview report in browser before generating PDF
- Submit report generation job to backend, display progress, download PDF when complete

### Phase 7 — Billing + Plan Enforcement (Days 34–36)

**Day 34: Stripe integration**
- `src/api/handlers/billing.rs` — Create checkout session, create customer portal session, get subscription status
- `src/api/handlers/webhooks/stripe.rs` — Handle subscription events: `checkout.session.completed`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.payment_failed`
- Plan mapping:
  - Free: 1 project, 500m × 500m domain, 2 wind sims/month, 7-day forecast
  - Planner ($99/mo): 5 projects, 2km × 2km, 10 CFD sims/month, solar + frost, hourly forecast, PDF reports
  - Team ($399/mo): 25 projects, 5km × 5km, 50 CFD sims/month, thermal + comprehensive, API access, white-label

**Day 35: Usage tracking and plan limits**
- `src/middleware/plan_limits.rs` — Middleware checking plan limits before simulation execution
- `src/services/usage.rs` — Track CFD simulation minutes, downscaling runs, storage GB-days per billing period
- Usage record insertion after each simulation completes
- Usage dashboard: `src/api/handlers/usage.rs` — Current period usage, historical usage
- Approaching-limit warnings at 80% and 100% of monthly quotas

**Day 36: Billing UI and feature gating**
- `frontend/src/pages/Billing.tsx` — Plan comparison table, current plan display, usage meters
- `frontend/src/components/billing/PlanCard.tsx` — Plan feature comparison cards
- `frontend/src/components/billing/UsageMeter.tsx` — CFD minutes and simulations usage bars with threshold indicators
- Upgrade/downgrade flow via Stripe Customer Portal
- Upgrade prompt modals when hitting plan limits (e.g., "Project domain exceeds 500m. Upgrade to Planner.")
- Feature gating: lock white-label reports, API access, weather station integration, thermal simulation behind appropriate tiers

### Phase 8 — Testing and Launch Prep (Days 37–42)

**Day 37: CFD validation benchmarks**
- Benchmark 1: Flow over 2D hill — compare with wind tunnel data from Askervein Hill study
- Benchmark 2: Building wake — compare with AIJ (Architectural Institute of Japan) benchmark
- Benchmark 3: Street canyon — validate against published LES results for urban canyon
- Automated test suite: run benchmarks on each OpenFOAM solver update, assert metrics within 10% tolerance
- Document methodology in whitepaper for regulatory submission support

**Day 38: ML downscaling validation**
- Validation 1: Temperature downscaling — compare against high-resolution WRF runs (RMSE < 1.5°C target)
- Validation 2: Wind downscaling — compare against station observations (correlation > 0.85 target)
- Validation 3: Precipitation — verify spatial patterns match radar observations (bias < 15%)
- Cross-validation on held-out test set from different geographic regions
- Document model performance metrics for transparency

**Day 39: Integration testing and performance optimization**
- End-to-end test: create project → terrain ingest → forecast subscription → CFD simulation → results visualization → PDF report
- Load testing: simulate 50 concurrent users creating projects and running simulations
- Optimize database queries: add missing indexes, use EXPLAIN ANALYZE
- Optimize S3 access patterns: batch uploads, presigned URL caching
- Frontend bundle optimization: code-splitting, lazy loading, image optimization
- Target: project creation < 10s, CFD submission < 3s, results page load < 2s

**Day 40: Documentation and onboarding**
- User documentation: Getting Started guide, simulation type selection flowchart, results interpretation guide
- API documentation: OpenAPI/Swagger spec with example requests
- Video tutorials: project creation walkthrough, wind comfort analysis demo, solar yield estimation
- In-app tooltips and help text for all major features
- Sample projects: pre-built templates for common use cases (construction site, farm, solar farm)

**Day 41: Security audit and compliance prep**
- Security review: SQL injection prevention (SQLx compile-time checks), XSS protection (React auto-escaping), CSRF tokens
- Rate limiting: per-user API rate limits (100 req/min for free, 1000 req/min for paid)
- Data privacy: GDPR compliance review, privacy policy, terms of service
- PII encryption: encrypt weather station API credentials, payment method tokens
- Penetration testing: engage external security firm for white-box testing

**Day 42: Deployment and monitoring setup**
- Production Kubernetes cluster setup: multi-AZ deployment, auto-scaling policies
- CI/CD pipeline: GitHub Actions for automated testing, Docker image builds, K8s deployments
- Monitoring dashboards: Grafana dashboards for API latency, CFD job queue depth, ML inference latency, database query performance
- Alerting: PagerDuty integration for critical errors (database down, CFD pod failures, payment processing errors)
- Backup strategy: automated PostgreSQL backups to S3, point-in-time recovery testing
- Runbook: incident response procedures, rollback procedures, scaling procedures

---

## Validation Benchmarks

### 1. SRTM Terrain Accuracy
- **Metric:** Vertical accuracy of ingested SRTM DEMs
- **Target:** Mean absolute error < 10 meters vs. surveyed ground control points
- **Method:** Compare SRTM elevations against USGS benchmark elevations at 100+ locations
- **Pass Criteria:** MAE ≤ 10m for 95% of validation points

### 2. ML Downscaling Temperature RMSE
- **Metric:** Root mean square error for downscaled temperature forecasts
- **Target:** RMSE < 1.5°C vs. high-resolution WRF reference simulations
- **Method:** Compare downscaled GFS → 50m temperature fields against WRF 1km runs for 30 test cases
- **Pass Criteria:** RMSE ≤ 1.5°C and bias < 0.5°C across all test domains

### 3. CFD Wind Speed Validation
- **Metric:** Correlation coefficient for pedestrian-level wind speed predictions
- **Target:** r > 0.80 vs. wind tunnel measurements or high-fidelity LES
- **Method:** Simulate AIJ benchmark building case, compare with published wind tunnel data at probe points
- **Pass Criteria:** Pearson correlation r ≥ 0.80 and normalized RMSE < 20%

### 4. Solar Irradiance Annual Total
- **Metric:** Accuracy of annual GHI totals vs. ground-measured pyranometer data
- **Target:** Error < 5% vs. NREL NSRDB (National Solar Radiation Database)
- **Method:** Compute annual solar for 20 sites with NSRDB coverage, compare annual totals
- **Pass Criteria:** Absolute percentage error ≤ 5% for 18 out of 20 sites

### 5. End-to-End Simulation Latency
- **Metric:** Time from simulation submission to completed results for medium-sized domain
- **Target:** < 45 minutes for 1km × 1km wind CFD at 10m resolution
- **Method:** Submit 10 test simulations, measure time from API POST to results_url available
- **Pass Criteria:** 90th percentile completion time ≤ 45 minutes

---

## Post-MVP Roadmap

### Version 1.1 (Months 3-4) — Agriculture Features
- Growing Degree Day (GDD) tracking per field with crop-specific base temperatures
- Evapotranspiration (ET) estimation using Penman-Monteith at field resolution
- Spray window forecasting (wind speed, inversion risk, rain-free periods)
- Disease pressure indices (leaf wetness duration, humidity-temperature combinations)
- Integration with John Deere Operations Center and Climate FieldView APIs

### Version 1.2 (Months 4-5) — Construction Weather Windows
- User-defined activity rules engine: "concrete pour requires >5°C for 48hr, wind <25 km/h, no precipitation"
- Rolling 14-day forecast with go/no-go status for each defined activity
- Optimal scheduling suggestions: "best 3-day window for crane operations this month"
- Historical weather window frequency analysis
- Integration with Procore and P6 project scheduling tools via CSV/API export

### Version 1.3 (Months 5-6) — Urban Heat Island and Thermal Comfort
- Surface temperature modeling using land use, albedo, vegetation fraction, anthropogenic heat
- Nighttime UHI intensity mapping (delta from rural baseline)
- Cool roof and green infrastructure scenario modeling
- Thermal comfort indices: UTCI, PET, WBGT with ISO 7243 heat stress thresholds
- What-if analysis: compare scenarios with different surface materials, tree cover

### Version 1.4 (Months 6-7) — Advanced CFD and Validation
- LES (Large Eddy Simulation) transient turbulence modeling for complex sites
- Gust analysis: peak wind speed with 3-second averaging
- Pollutant dispersion modeling (passive scalar transport)
- Snow drift simulation using Eulerian snow transport model
- Professional engineering review and PE stamp option for regulatory submissions

### Version 1.5 (Months 7-8) — Collaboration and API
- Team collaboration: real-time presence, comments, version comparison
- Public API with OAuth 2.0: programmatic simulation submission, webhook notifications
- Webhook integrations: Slack, Microsoft Teams, custom HTTPS endpoints
- BIM platform integration: Autodesk Construction Cloud, Trimble Connect for geometry import
- SCADA system integration: ingest real-time sensor data for model calibration

---

## Launch Checklist

### Week 1: Alpha Release (Internal Testing)
- [ ] Deploy to staging environment with production-equivalent infrastructure
- [ ] Internal team creates 10 test projects across different geographies and project types
- [ ] Run full test suite: unit tests, integration tests, CFD benchmarks, ML validation
- [ ] Performance testing: 20 concurrent users, 50 simulations in queue
- [ ] Security scan: OWASP ZAP automated scan, manual penetration test on auth endpoints

### Week 2: Beta Release (Invited Users)
- [ ] Invite 20-30 beta testers from construction, agriculture, renewable energy sectors
- [ ] Onboarding session: live demo, Q&A, feedback collection
- [ ] Monitor usage metrics: project creation rate, simulation completion rate, error rate
- [ ] Daily check-ins: collect feedback via Slack channel, prioritize bug fixes
- [ ] Iterate on UX: simplify simulation configuration, improve result interpretation

### Week 3: Public Launch Preparation
- [ ] Legal review: terms of service, privacy policy, GDPR compliance
- [ ] Marketing materials: landing page, product demo video, case studies from beta users
- [ ] Press outreach: submit to TechCrunch, ENR, PV Magazine, AgFunder news
- [ ] SEO optimization: meta tags, sitemap, blog posts targeting key search terms
- [ ] Support infrastructure: Intercom chat widget, help center, email support queue

### Week 4: Public Launch
- [ ] Announce on Product Hunt, Hacker News, LinkedIn, Twitter
- [ ] Publish technical blog post: "How We Built a Cloud-Native Microclimate Platform with Rust and OpenFOAM"
- [ ] Host launch webinar: product walkthrough, live Q&A
- [ ] Monitor: server metrics, error rates, support ticket volume, user feedback
- [ ] Rapid iteration: deploy bug fixes daily, A/B test onboarding flow improvements

---

## Success Metrics

| Metric | Week 1 | Month 1 | Month 3 | Month 6 |
|--------|--------|---------|---------|---------|
| Registered users | 50 | 300 | 1,200 | 3,500 |
| Projects created | 75 | 600 | 3,000 | 10,000 |
| Simulations run | 150 | 1,500 | 10,000 | 40,000 |
| Paying customers | 5 | 25 | 80 | 200 |
| MRR | $500 | $2,500 | $10,000 | $30,000 |
| Free-to-paid conversion | 10% | 8% | 7% | 6% |
| Simulation success rate | 85% | 92% | 96% | 98% |
| Avg CFD completion time (1km²) | 60 min | 50 min | 40 min | 30 min |
| Support ticket response time | <4 hrs | <2 hrs | <1 hr | <30 min |

---

**Total implementation time: 42 days (6 weeks) with 2 full-time engineers**

**Estimated first-year runway: $180K (infrastructure + salaries + marketing)**

**Target first-year revenue: $360K MRR × 12 = $360K ARR** (breakeven at Month 9-10 with aggressive user acquisition)
