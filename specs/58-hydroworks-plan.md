# 58. HydroWorks — Hydraulic and Hydrologic Modeling Platform

## Implementation Plan

**MVP Scope:** Browser-based hydraulic modeling platform with DEM import (GeoTIFF, LAS/LAZ, USGS 3DEP, FABDEM) via GDAL and automatic terrain processing engine that extracts river centerlines via D8 flow accumulation with threshold-based stream delineation, cross-sections sampled perpendicular to centerline at user-defined intervals (10-500m), and watershed boundaries via pour point analysis, SCS Curve Number rainfall-runoff model with design storm support (SCS Type I/IA/II/III, Chicago method, custom hyetographs) implementing distributed runoff with time of concentration (Kirpich, Kerby, NRCS velocity), 1D steady-state river hydraulics solver using standard step method (direct step for subcritical, upstream step for supercritical) with energy/momentum equations for bridges and culverts, Manning's equation for channel friction with composite roughness for compound channels, critical depth computation via bisection/Newton-Raphson, structure hydraulics (bridges via energy loss + pressure/weir flow transitions, culverts with inlet/outlet control via HDS-5 equations, weirs/gates with standard discharge equations), interactive cross-section editor with station-elevation tables and automatic ground profile interpolation, flood mapping on WebGL-rendered basemap with depth/velocity/hazard contours generated from water surface elevation vs. DEM differencing, longitudinal profile viewer showing water surface vs. streambed elevation rendered via Canvas2D with zoom/pan, PDF report generation with flood profiles, peak flow tables, structure summary, and flood extent map, PostgreSQL storage for projects/cross-sections/results and S3 for DEMs/flood maps, Stripe billing with three tiers (Free / Pro $129/mo / Advanced $299/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| Hydraulic Solver | Rust (native + WASM) | Standard step method, structure hydraulics, critical depth computation |
| WASM Compilation | `wasm-pack` + `wasm-bindgen` | Client-side solver for small models (≤200 cross-sections) |
| Hydrology Engine | Rust (native + WASM) | SCS-CN, time of concentration, unit hydrograph, routing |
| Terrain Processing | Rust + GDAL | DEM analysis, flow direction (D8), stream network, cross-section extraction |
| ML Calibration | Python 3.12 (FastAPI) | Manning's n estimation from land cover, parameter optimization (scipy.optimize) |
| Database | PostgreSQL 16 + PostGIS | Projects, cross-sections, simulations, spatial queries |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | DEMs (GeoTIFF), flood maps (GeoTIFF/PNG), simulation results, PDFs |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| GIS Map Viewer | Mapbox GL JS 3.x | Terrain rendering, flood extent overlay, interactive basemap |
| 3D Terrain | Deck.gl + Three.js | 3D flood depth visualization, terrain draping |
| Profile Viewer | Canvas2D (custom) | Longitudinal profile, cross-section plots with zoom/pan |
| Real-time | WebSocket (Axum) | Solver progress streaming, long-running terrain processing |
| Job Queue | Redis 7 + Tokio tasks | Terrain processing, large model simulations, PDF generation |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | WASM bundle delivery, static frontend assets, tiled basemaps |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, solver convergence tracking, error tracking |
| CI/CD | GitHub Actions | WASM build pipeline, Rust tests, Docker image push |

### Key Architecture Decisions

1. **Hybrid client/server solver with 200 cross-section WASM threshold**: Models with ≤200 cross-sections (covers 80%+ of small stream/drainage studies) run entirely in the browser via WASM standard step solver, providing instant results with zero server cost. Models exceeding 200 cross-sections (major rivers, multi-reach systems) execute on the server with parallel reach computation. The threshold is configurable per plan tier (Free: 50 XS, Pro: 200 XS, Advanced: unlimited server).

2. **Custom Rust hydraulic solver rather than wrapping HEC-RAS**: Building a custom standard step solver in Rust gives us full control over convergence algorithms, WASM compilation, and modern UX integration. HEC-RAS is closed-source with no API, requires Windows, and has fragile file-based I/O. Our Rust solver implements the same fundamental algorithms (standard step method from Chow 1959, structure hydraulics from HDS-1/HDS-5) while being memory-safe, WASM-compatible, and parallelizable across reaches.

3. **Automatic terrain processing with GDAL + custom D8 flow accumulation**: DEM processing (flow direction, accumulation, stream delineation) uses D8 algorithm with pit filling via priority-flood depression breaching to handle real-world terrain artifacts. Cross-sections are extracted by: (1) generating perpendicular transects to stream centerline at specified intervals, (2) sampling DEM elevations along each transect at 1-5m horizontal resolution, (3) automatic left/right bank detection via ground slope analysis, (4) Manning's n assignment from NLCD/Corine land cover lookup tables. This reduces model setup time from days to minutes.

4. **Mapbox GL JS for flood mapping with custom raster overlay**: Flood extent is rendered as a WebGL raster tile layer where each pixel's depth = max(0, WSE - DEM_elevation). Depth is classified into hazard categories (low/medium/high based on depth and velocity) and rendered with semi-transparent color ramps. This decouples the hydraulic computation (1D steady-state along cross-sections) from the spatial visualization (2D flood extent via DEM differencing), allowing progressive enhancement to true 2D hydrodynamics in post-MVP.

5. **S3 for DEM storage with PostGIS metadata catalog**: DEMs (often 500MB-5GB GeoTIFFs for watershed-scale studies) are stored in S3 with cloud-optimized GeoTIFF (COG) format for partial reads. PostgreSQL + PostGIS stores DEM metadata (bounding box, resolution, CRS, S3 URL) and spatial indexes for fast "which DEM covers this project extent?" queries. Cross-sections reference DEM tiles by bounding box intersection to enable incremental terrain updates without full model reprocessing.

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "postgis";  -- Spatial data support
CREATE EXTENSION IF NOT EXISTS "pg_trgm";  -- Fuzzy text search

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

-- Projects (hydraulic model workspace)
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    extent GEOMETRY(Polygon, 4326),  -- Project bounding box for DEM/map extent
    stream_network JSONB DEFAULT '[]',  -- [{id, geometry, name, reach_id}] for river centerlines
    settings JSONB DEFAULT '{}',  -- Unit system (SI/US), default Manning's n, computation intervals
    is_public BOOLEAN NOT NULL DEFAULT false,
    forked_from UUID REFERENCES projects(id) ON DELETE SET NULL,
    thumbnail_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);
CREATE INDEX projects_org_idx ON projects(org_id);
CREATE INDEX projects_extent_idx ON projects USING GIST(extent);
CREATE INDEX projects_updated_idx ON projects(updated_at DESC);

-- DEMs (terrain data)
CREATE TABLE dems (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    extent GEOMETRY(Polygon, 4326) NOT NULL,
    crs TEXT NOT NULL,  -- EPSG code, e.g., "EPSG:32610"
    resolution_m REAL NOT NULL,  -- Cell size in meters
    s3_url TEXT NOT NULL,  -- S3 path to COG GeoTIFF
    file_size_bytes BIGINT NOT NULL,
    source TEXT,  -- USGS 3DEP | LiDAR | FABDEM | SRTM | user_uploaded
    metadata JSONB DEFAULT '{}',  -- Vertical datum, acquisition date, vertical accuracy
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX dems_project_idx ON dems(project_id);
CREATE INDEX dems_extent_idx ON dems USING GIST(extent);

-- River Reaches
CREATE TABLE reaches (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    river_name TEXT,
    centerline GEOMETRY(LineString, 4326) NOT NULL,
    upstream_reach_id UUID REFERENCES reaches(id) ON DELETE SET NULL,
    downstream_reach_id UUID REFERENCES reaches(id) ON DELETE SET NULL,
    length_m REAL NOT NULL,
    slope REAL,  -- Average bed slope (m/m)
    settings JSONB DEFAULT '{}',  -- Flow regime (subcritical/supercritical), computation intervals
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX reaches_project_idx ON reaches(project_id);
CREATE INDEX reaches_centerline_idx ON reaches USING GIST(centerline);

-- Cross-Sections
CREATE TABLE cross_sections (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    reach_id UUID NOT NULL REFERENCES reaches(id) ON DELETE CASCADE,
    station REAL NOT NULL,  -- River station (distance from downstream end, meters)
    location GEOMETRY(Point, 4326) NOT NULL,
    transect GEOMETRY(LineString, 4326),  -- Survey line across valley
    stations REAL[] NOT NULL,  -- Horizontal station offsets from left bank (meters)
    elevations REAL[] NOT NULL,  -- Ground elevations at each station (meters)
    mannings_n REAL[] NOT NULL,  -- Manning's roughness for each subsection
    left_bank_station REAL NOT NULL,
    right_bank_station REAL NOT NULL,
    ineffective_flow_areas JSONB DEFAULT '[]',  -- [{left_station, right_station, elevation}] for blocked flow zones
    is_interpolated BOOLEAN DEFAULT false,  -- Auto-generated vs. user-surveyed
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX xs_reach_idx ON cross_sections(reach_id, station);
CREATE INDEX xs_location_idx ON cross_sections USING GIST(location);

-- Structures (bridges, culverts, weirs, gates)
CREATE TABLE structures (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    reach_id UUID NOT NULL REFERENCES reaches(id) ON DELETE CASCADE,
    station REAL NOT NULL,  -- River station where structure is located
    structure_type TEXT NOT NULL,  -- bridge | culvert | weir | gate | inline_structure
    name TEXT,
    parameters JSONB NOT NULL,  -- Type-specific: bridge deck elevation, culvert diameter, weir crest, etc.
    us_xs_id UUID REFERENCES cross_sections(id),  -- Upstream cross-section
    ds_xs_id UUID REFERENCES cross_sections(id),  -- Downstream cross-section
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX structures_reach_idx ON structures(reach_id, station);

-- Boundary Conditions
CREATE TABLE boundary_conditions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    bc_type TEXT NOT NULL,  -- flow_hydrograph | stage_hydrograph | rating_curve | normal_depth | known_wse
    location_type TEXT NOT NULL,  -- upstream | downstream | lateral_inflow | tributary
    reach_id UUID REFERENCES reaches(id) ON DELETE SET NULL,
    station REAL,  -- For lateral inflows
    data JSONB NOT NULL,  -- Time series, rating curve points, slope for normal depth, etc.
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX bc_project_idx ON boundary_conditions(project_id);
CREATE INDEX bc_reach_idx ON boundary_conditions(reach_id);

-- Hydrology Basins (for rainfall-runoff)
CREATE TABLE hydro_basins (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    boundary GEOMETRY(Polygon, 4326) NOT NULL,
    area_km2 REAL NOT NULL,
    curve_number REAL,  -- SCS Curve Number (40-98)
    time_of_concentration_hrs REAL,  -- Tc via Kirpich, Kerby, NRCS velocity
    land_cover JSONB DEFAULT '{}',  -- {forest: 0.3, urban: 0.5, ...} for CN lookup
    outlet_reach_id UUID REFERENCES reaches(id),
    outlet_station REAL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX basins_project_idx ON hydro_basins(project_id);
CREATE INDEX basins_boundary_idx ON hydro_basins USING GIST(boundary);

-- Design Storms
CREATE TABLE design_storms (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    storm_type TEXT NOT NULL,  -- scs_type_i | scs_type_ia | scs_type_ii | scs_type_iii | chicago | custom
    duration_hrs REAL NOT NULL,
    total_depth_mm REAL NOT NULL,
    parameters JSONB DEFAULT '{}',  -- SCS distribution, Chicago r/a ratio, custom hyetograph points
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX storms_project_idx ON design_storms(project_id);

-- Simulations
CREATE TABLE simulations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    simulation_type TEXT NOT NULL,  -- steady_flow | unsteady_flow | hydrology_only
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | running | completed | failed | cancelled
    execution_mode TEXT NOT NULL DEFAULT 'wasm',  -- wasm | server
    flow_data JSONB NOT NULL,  -- Boundary conditions, design storm ID, flow values
    xs_count INTEGER NOT NULL DEFAULT 0,
    results_url TEXT,  -- S3 URL for detailed results (WSE, velocity, shear at each XS)
    results_summary JSONB,  -- Quick-access: max WSE per reach, flood extent area, peak flow
    flood_map_url TEXT,  -- S3 URL to flood depth GeoTIFF
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

-- Simulation Jobs (for server-side execution)
CREATE TABLE simulation_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    simulation_id UUID NOT NULL REFERENCES simulations(id) ON DELETE CASCADE,
    worker_id TEXT,
    cores_allocated INTEGER DEFAULT 1,
    memory_mb INTEGER DEFAULT 4096,
    priority INTEGER DEFAULT 0,
    progress_pct REAL DEFAULT 0.0,
    convergence_data JSONB DEFAULT '[]',  -- [{xs_id, iterations, delta_wse, critical_depth_flag}]
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX jobs_sim_idx ON simulation_jobs(simulation_id);
CREATE INDEX jobs_worker_idx ON simulation_jobs(worker_id);

-- Usage Tracking
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,  -- compute_minutes | storage_gb | cross_sections_computed | flood_map_generated
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
use geo_types::{Geometry, LineString, Point, Polygon};
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
    #[sqlx(type_name = "geometry")]
    pub extent: Option<Geometry<f64>>,
    pub stream_network: serde_json::Value,
    pub settings: serde_json::Value,
    pub is_public: bool,
    pub forked_from: Option<Uuid>,
    pub thumbnail_url: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Dem {
    pub id: Uuid,
    pub project_id: Uuid,
    pub name: String,
    #[sqlx(type_name = "geometry")]
    pub extent: Geometry<f64>,
    pub crs: String,
    pub resolution_m: f32,
    pub s3_url: String,
    pub file_size_bytes: i64,
    pub source: Option<String>,
    pub metadata: serde_json::Value,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Reach {
    pub id: Uuid,
    pub project_id: Uuid,
    pub name: String,
    pub river_name: Option<String>,
    #[sqlx(type_name = "geometry")]
    pub centerline: Geometry<f64>,
    pub upstream_reach_id: Option<Uuid>,
    pub downstream_reach_id: Option<Uuid>,
    pub length_m: f32,
    pub slope: Option<f32>,
    pub settings: serde_json::Value,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct CrossSection {
    pub id: Uuid,
    pub reach_id: Uuid,
    pub station: f64,  // River station in meters
    #[sqlx(type_name = "geometry")]
    pub location: Geometry<f64>,
    #[sqlx(type_name = "geometry")]
    pub transect: Option<Geometry<f64>>,
    pub stations: Vec<f64>,     // Horizontal offsets from left
    pub elevations: Vec<f64>,   // Ground elevations
    pub mannings_n: Vec<f64>,   // Roughness per subsection
    pub left_bank_station: f64,
    pub right_bank_station: f64,
    pub ineffective_flow_areas: serde_json::Value,
    pub is_interpolated: bool,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Structure {
    pub id: Uuid,
    pub reach_id: Uuid,
    pub station: f64,
    pub structure_type: String,
    pub name: Option<String>,
    pub parameters: serde_json::Value,
    pub us_xs_id: Option<Uuid>,
    pub ds_xs_id: Option<Uuid>,
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
    pub flow_data: serde_json::Value,
    pub xs_count: i32,
    pub results_url: Option<String>,
    pub results_summary: Option<serde_json::Value>,
    pub flood_map_url: Option<String>,
    pub error_message: Option<String>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub wall_time_ms: Option<i32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct SteadyFlowData {
    pub flow_cfs: f64,  // Discharge in cubic feet per second
    pub downstream_bc: DownstreamBoundary,
    pub known_wse: Vec<KnownWse>,  // Optional: known water surface elevations
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(tag = "type")]
pub enum DownstreamBoundary {
    NormalDepth { slope: f64 },           // Slope in m/m
    KnownWse { elevation_m: f64 },        // Water surface elevation in meters
    RatingCurve { points: Vec<(f64, f64)> },  // (flow_cfs, wse_m)
}

#[derive(Debug, Deserialize, Serialize)]
pub struct KnownWse {
    pub xs_id: Uuid,
    pub elevation_m: f64,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct HydroBasin {
    pub id: Uuid,
    pub project_id: Uuid,
    pub name: String,
    pub boundary: Geometry<f64>,
    pub area_km2: f32,
    pub curve_number: Option<f32>,
    pub time_of_concentration_hrs: Option<f32>,
    pub land_cover: serde_json::Value,
    pub outlet_reach_id: Option<Uuid>,
    pub outlet_station: Option<f64>,
}
```

---

## Solver Architecture Deep-Dive

### Governing Equations and Discretization

HydroWorks' 1D steady-state solver implements the **Standard Step Method** for gradually varied flow (GVF), the foundational algorithm used by HEC-RAS, MIKE 11, and all production river hydraulics models. The method solves the energy equation between adjacent cross-sections:

```
Energy Equation:
Z₁ + d₁ + α₁·V₁²/(2g) = Z₂ + d₂ + α₂·V₂²/(2g) + hₗ

where:
  Z = bed elevation (m)
  d = flow depth (m)
  V = average velocity (m/s) = Q/A
  α = velocity distribution coefficient (Coriolis, typically 1.0-1.5)
  g = gravitational acceleration (9.81 m/s²)
  hₗ = energy head loss between sections (m)
```

**Energy loss** is computed from Manning's equation friction loss plus contraction/expansion losses:

```
hₗ = hf + hc

Friction loss (Manning):
hf = L · S̄f
S̄f = [(Q·n)/(K)]²  (average conveyance method)
K = (1/n) · A · R^(2/3)  (conveyance)
R = A/P  (hydraulic radius)

Contraction/Expansion loss:
hc = C · |α₁·V₁²/(2g) - α₂·V₂²/(2g)|
C = 0.1 (contraction), 0.3 (expansion)
```

**Flow computation direction**:
- **Subcritical flow** (Fr < 1): compute upstream from known downstream boundary (water surface "backs up")
- **Supercritical flow** (Fr > 1): compute downstream from known upstream boundary (water surface "shoots forward")
- **Mixed flow**: detect critical depth locations, split reach into subcritical/supercritical segments

**Critical depth** is computed by solving:

```
Q² / g = A³ / T

where:
  Q = discharge (m³/s)
  A = flow area (m²)
  T = top width (m)

Solved via Newton-Raphson:
f(y) = Q² · T - g · A³
f'(y) = Q² · dT/dy - 3g · A² · dA/dy
y_{n+1} = y_n - f(y_n) / f'(y_n)
```

**Structure hydraulics** use empirical equations from FHWA Hydraulic Design Series:

Bridge flow (HDS-1):
```
Low flow (no pressure):  Energy equation with bridge loss coefficient
Pressure flow:           Orifice equation: Q = C_d · A · √(2g·ΔH)
Weir flow:               Q = C_w · L · H^(3/2)
```

Culvert flow (HDS-5):
```
Inlet control:   Q = C_i · A · (H_w)^0.5  (form loss at entrance)
Outlet control:  Q = A · √[(2g·ΔH) / (1 + K_e + K_f)]
                 K_f = (nL/R^(4/3)) · (Q/A)²/(2g)  (barrel friction)
```

### Client/Server Split (200 XS Threshold)

```
Model created → XS count extracted
    │
    ├── ≤200 XS → WASM solver (browser)
    │   ├── Instant startup (<100ms)
    │   ├── Results in 0.5-3 seconds
    │   └── Zero server cost, offline capable
    │
    └── >200 XS → Server solver (Rust native)
        ├── Job queued via Redis
        ├── Worker parallelizes across reaches
        ├── Progress streamed via WebSocket
        └── Results to S3, flood map rendered
```

The 200 XS threshold was chosen because:
- WASM solver handles 200 XS steady-state in <2 seconds on modern hardware
- 200 XS covers: small stream studies (50-100 XS), urban drainage (20-50 XS per reach), bridge scour analysis (10-30 XS)
- Above 200 XS: large river systems (Mississippi, Rhine), multi-tributary watersheds, dam breach flood waves

### WASM Compilation Pipeline

```rust
// solver-wasm/src/lib.rs

use wasm_bindgen::prelude::*;
use serde::{Deserialize, Serialize};
use hydraulic_core::{StandardStepSolver, CrossSection, FlowProfile};

#[wasm_bindgen]
pub struct WasmSolver {
    solver: StandardStepSolver,
}

#[wasm_bindgen]
impl WasmSolver {
    #[wasm_bindgen(constructor)]
    pub fn new() -> Self {
        console_error_panic_hook::set_once();
        Self {
            solver: StandardStepSolver::new(),
        }
    }

    #[wasm_bindgen]
    pub fn add_cross_section(&mut self, xs_json: &str) -> Result<(), JsValue> {
        let xs: CrossSection = serde_json::from_str(xs_json)
            .map_err(|e| JsValue::from_str(&e.to_string()))?;
        self.solver.add_cross_section(xs);
        Ok(())
    }

    #[wasm_bindgen]
    pub fn solve_steady_flow(&mut self, flow_cfs: f64, ds_wse_ft: f64) -> Result<String, JsValue> {
        let profile = self.solver.solve_subcritical(flow_cfs, ds_wse_ft)
            .map_err(|e| JsValue::from_str(&e.to_string()))?;
        serde_json::to_string(&profile)
            .map_err(|e| JsValue::from_str(&e.to_string()))
    }

    #[wasm_bindgen]
    pub fn compute_critical_depth(&self, xs_index: usize, flow_cfs: f64) -> Result<f64, JsValue> {
        self.solver.critical_depth(xs_index, flow_cfs)
            .map_err(|e| JsValue::from_str(&e.to_string()))
    }
}
```

---

## Architecture Deep-Dives

### 1. Standard Step Solver (Rust Core)

The core hydraulic engine that iteratively solves the energy equation between cross-sections.

```rust
// hydraulic-core/src/solver/standard_step.rs

use crate::cross_section::CrossSection;
use crate::error::SolverError;

pub struct StandardStepSolver {
    pub cross_sections: Vec<CrossSection>,
    pub reach_lengths: Vec<f64>,  // Distance between XS (meters)
}

impl StandardStepSolver {
    pub fn new() -> Self {
        Self {
            cross_sections: Vec::new(),
            reach_lengths: Vec::new(),
        }
    }

    /// Solve subcritical flow profile (upstream from known downstream WSE)
    pub fn solve_subcritical(
        &self,
        flow_cfs: f64,
        downstream_wse_ft: f64,
    ) -> Result<FlowProfile, SolverError> {
        let n = self.cross_sections.len();
        if n < 2 {
            return Err(SolverError::InsufficientCrossSections);
        }

        let mut profile = FlowProfile::new(n);
        profile.flow_cfs = flow_cfs;

        // Start at downstream (index 0)
        let ds_xs = &self.cross_sections[0];
        profile.wse[0] = downstream_wse_ft;
        profile.depth[0] = downstream_wse_ft - ds_xs.min_elevation();
        profile.area[0] = ds_xs.flow_area(profile.depth[0]);
        profile.velocity[0] = (flow_cfs / profile.area[0]) * 0.3048;  // ft/s → m/s

        // Step upstream
        for i in 1..n {
            let xs_ds = &self.cross_sections[i - 1];
            let xs_us = &self.cross_sections[i];
            let reach_length = self.reach_lengths[i - 1];

            // Initial guess: parallel water surface
            let mut wse_us = profile.wse[i - 1];
            let mut iterations = 0;
            const MAX_ITER: usize = 50;
            const TOLERANCE_FT: f64 = 0.01;

            loop {
                iterations += 1;
                if iterations > MAX_ITER {
                    return Err(SolverError::ConvergenceFailed {
                        xs_index: i,
                        iterations,
                    });
                }

                let depth_us = wse_us - xs_us.min_elevation();
                if depth_us < 0.001 {
                    return Err(SolverError::NegativeDepth { xs_index: i });
                }

                let area_us = xs_us.flow_area(depth_us);
                let velocity_us = (flow_cfs / area_us) * 0.3048;  // ft/s → m/s
                let conveyance_us = xs_us.conveyance(depth_us);

                let area_ds = profile.area[i - 1];
                let velocity_ds = profile.velocity[i - 1];
                let conveyance_ds = xs_ds.conveyance(profile.depth[i - 1]);

                // Energy equation components
                let alpha_us = xs_us.velocity_coefficient(depth_us);
                let alpha_ds = xs_ds.velocity_coefficient(profile.depth[i - 1]);

                let vel_head_us = alpha_us * velocity_us.powi(2) / (2.0 * 9.81);
                let vel_head_ds = alpha_ds * velocity_ds.powi(2) / (2.0 * 9.81);

                // Friction loss (average conveyance method)
                let k_avg = (conveyance_us + conveyance_ds) / 2.0;
                let sf_avg = (flow_cfs / k_avg).powi(2);
                let hf = reach_length * sf_avg;

                // Contraction/expansion loss
                let c_loss = if velocity_us > velocity_ds { 0.1 } else { 0.3 };
                let hc = c_loss * (vel_head_us - vel_head_ds).abs();

                // Energy equation: Z_us + d_us + V²/(2g) = Z_ds + d_ds + V²/(2g) + hf + hc
                let energy_us = wse_us + vel_head_us;
                let energy_ds = profile.wse[i - 1] + vel_head_ds;
                let residual = energy_us - (energy_ds + hf + hc);

                if residual.abs() < TOLERANCE_FT {
                    // Converged
                    profile.wse[i] = wse_us;
                    profile.depth[i] = depth_us;
                    profile.area[i] = area_us;
                    profile.velocity[i] = velocity_us;
                    profile.froude[i] = self.compute_froude(area_us, xs_us.top_width(depth_us), velocity_us);
                    profile.iterations[i] = iterations;
                    break;
                }

                // Newton-Raphson adjustment
                // dE/dWSE ≈ 1 + d(V²/2g)/dWSE
                // For gradually varied flow, use simplified: dE/dWSE ≈ 1 - Fr²
                let top_width = xs_us.top_width(depth_us);
                let fr = self.compute_froude(area_us, top_width, velocity_us);
                let deriv = 1.0 - fr.powi(2);

                if deriv.abs() < 0.01 {
                    // Near critical depth, use smaller step
                    wse_us -= residual.signum() * 0.05;
                } else {
                    wse_us -= residual / deriv;
                }
            }
        }

        Ok(profile)
    }

    fn compute_froude(&self, area: f64, top_width: f64, velocity: f64) -> f64 {
        if top_width < 0.001 {
            return 0.0;
        }
        let hydraulic_depth = area / top_width;
        velocity / (9.81 * hydraulic_depth).sqrt()
    }

    /// Compute critical depth at a cross-section
    pub fn critical_depth(&self, xs_index: usize, flow_cfs: f64) -> Result<f64, SolverError> {
        let xs = &self.cross_sections[xs_index];
        let flow_cms = flow_cfs * 0.028317;  // cfs → m³/s

        // Initial guess: normal depth or half of max depth
        let mut depth = xs.max_depth() * 0.5;
        const MAX_ITER: usize = 100;
        const TOLERANCE: f64 = 0.001;

        for _iter in 0..MAX_ITER {
            let area = xs.flow_area(depth);
            let top_width = xs.top_width(depth);

            if area < 0.001 || top_width < 0.001 {
                depth += 0.01;
                continue;
            }

            // f(y) = Q² · T - g · A³
            let f = flow_cms.powi(2) * top_width - 9.81 * area.powi(3);

            // f'(y) = Q² · dT/dy - 3g · A² · dA/dy
            let delta_y = 0.001;
            let area_plus = xs.flow_area(depth + delta_y);
            let tw_plus = xs.top_width(depth + delta_y);
            let da_dy = (area_plus - area) / delta_y;
            let dt_dy = (tw_plus - top_width) / delta_y;

            let f_prime = flow_cms.powi(2) * dt_dy - 3.0 * 9.81 * area.powi(2) * da_dy;

            if f.abs() < TOLERANCE {
                return Ok(depth);
            }

            if f_prime.abs() < 1e-6 {
                // Derivative too small, use bisection fallback
                depth += if f > 0.0 { 0.01 } else { -0.01 };
            } else {
                depth -= f / f_prime;
            }

            // Clamp to valid range
            depth = depth.max(0.01).min(xs.max_depth());
        }

        Err(SolverError::CriticalDepthFailed { xs_index })
    }
}

#[derive(Debug, serde::Serialize)]
pub struct FlowProfile {
    pub flow_cfs: f64,
    pub wse: Vec<f64>,        // Water surface elevation (ft)
    pub depth: Vec<f64>,      // Flow depth (ft)
    pub area: Vec<f64>,       // Flow area (ft²)
    pub velocity: Vec<f64>,   // Average velocity (m/s)
    pub froude: Vec<f64>,     // Froude number
    pub iterations: Vec<usize>,  // Convergence iterations per XS
}

impl FlowProfile {
    pub fn new(n_xs: usize) -> Self {
        Self {
            flow_cfs: 0.0,
            wse: vec![0.0; n_xs],
            depth: vec![0.0; n_xs],
            area: vec![0.0; n_xs],
            velocity: vec![0.0; n_xs],
            froude: vec![0.0; n_xs],
            iterations: vec![0; n_xs],
        }
    }
}
```

### 2. Cross-Section Geometry (Rust)

Hydraulic property computation for irregular natural channel cross-sections with composite roughness.

```rust
// hydraulic-core/src/cross_section.rs

#[derive(Debug, Clone, serde::Deserialize, serde::Serialize)]
pub struct CrossSection {
    pub id: uuid::Uuid,
    pub station: f64,           // River station (m)
    pub stations: Vec<f64>,     // Horizontal offsets from left (m)
    pub elevations: Vec<f64>,   // Ground elevations (m)
    pub mannings_n: Vec<f64>,   // Roughness per subsection
    pub left_bank_station: f64,
    pub right_bank_station: f64,
}

impl CrossSection {
    /// Compute flow area at given depth
    pub fn flow_area(&self, depth: f64) -> f64 {
        let wse = self.min_elevation() + depth;
        let mut area = 0.0;

        for i in 0..self.stations.len() - 1 {
            let x1 = self.stations[i];
            let z1 = self.elevations[i];
            let x2 = self.stations[i + 1];
            let z2 = self.elevations[i + 1];

            // Skip if both points above water surface
            if z1 > wse && z2 > wse {
                continue;
            }

            // Clip to water surface
            let (x_left, z_left) = if z1 > wse {
                let frac = (wse - z1) / (z2 - z1);
                (x1 + frac * (x2 - x1), wse)
            } else {
                (x1, z1)
            };

            let (x_right, z_right) = if z2 > wse {
                let frac = (wse - z1) / (z2 - z1);
                (x1 + frac * (x2 - x1), wse)
            } else {
                (x2, z2)
            };

            // Trapezoidal area
            let width = x_right - x_left;
            let h_left = wse - z_left;
            let h_right = wse - z_right;
            area += 0.5 * width * (h_left + h_right);
        }

        area
    }

    /// Compute wetted perimeter
    pub fn wetted_perimeter(&self, depth: f64) -> f64 {
        let wse = self.min_elevation() + depth;
        let mut perimeter = 0.0;

        for i in 0..self.stations.len() - 1 {
            let x1 = self.stations[i];
            let z1 = self.elevations[i];
            let x2 = self.stations[i + 1];
            let z2 = self.elevations[i + 1];

            if z1 > wse && z2 > wse {
                continue;
            }

            let (x_left, z_left) = if z1 > wse {
                let frac = (wse - z1) / (z2 - z1);
                (x1 + frac * (x2 - x1), wse)
            } else {
                (x1, z1)
            };

            let (x_right, z_right) = if z2 > wse {
                let frac = (wse - z1) / (z2 - z1);
                (x1 + frac * (x2 - x1), wse)
            } else {
                (x2, z2)
            };

            let dx = x_right - x_left;
            let dz = z_right - z_left;
            perimeter += (dx.powi(2) + dz.powi(2)).sqrt();
        }

        perimeter
    }

    /// Compute conveyance K = (1/n) · A · R^(2/3)
    pub fn conveyance(&self, depth: f64) -> f64 {
        // For composite roughness, subdivide at bank stations and roughness changes
        let wse = self.min_elevation() + depth;
        let mut total_conveyance = 0.0;

        // Simple approach: average Manning's n
        let n_avg = self.mannings_n.iter().sum::<f64>() / self.mannings_n.len() as f64;

        let area = self.flow_area(depth);
        let wp = self.wetted_perimeter(depth);

        if wp < 0.001 {
            return 0.0;
        }

        let hydraulic_radius = area / wp;
        total_conveyance = (1.0 / n_avg) * area * hydraulic_radius.powf(2.0 / 3.0);

        total_conveyance
    }

    /// Compute top width at water surface
    pub fn top_width(&self, depth: f64) -> f64 {
        let wse = self.min_elevation() + depth;
        let mut width = 0.0;

        for i in 0..self.stations.len() - 1 {
            let z1 = self.elevations[i];
            let z2 = self.elevations[i + 1];

            // Check if segment intersects water surface
            if (z1 <= wse && z2 <= wse) || (z1 <= wse && z2 > wse) || (z1 > wse && z2 <= wse) {
                let x1 = self.stations[i];
                let x2 = self.stations[i + 1];

                let x_left = if z1 > wse {
                    let frac = (wse - z1) / (z2 - z1);
                    x1 + frac * (x2 - x1)
                } else {
                    x1
                };

                let x_right = if z2 > wse {
                    let frac = (wse - z1) / (z2 - z1);
                    x1 + frac * (x2 - x1)
                } else {
                    x2
                };

                width += x_right - x_left;
            }
        }

        width
    }

    pub fn min_elevation(&self) -> f64 {
        self.elevations.iter().copied().fold(f64::INFINITY, f64::min)
    }

    pub fn max_depth(&self) -> f64 {
        let min_z = self.min_elevation();
        let max_z = self.elevations.iter().copied().fold(f64::NEG_INFINITY, f64::max);
        max_z - min_z
    }

    pub fn velocity_coefficient(&self, _depth: f64) -> f64 {
        // Simplified: use constant α = 1.1 for natural channels
        // Production version would compute via: α = ∫(v³dA) / (V_avg³ · A)
        1.1
    }
}
```

### 3. Terrain Processing API (Rust + GDAL)

DEM upload, pit filling, flow accumulation, stream network extraction, and automated cross-section sampling.

```rust
// src/api/handlers/terrain.rs

use axum::{
    extract::{Path, State, Multipart},
    http::StatusCode,
    response::IntoResponse,
    Json,
};
use uuid::Uuid;
use crate::{
    state::AppState,
    auth::Claims,
    error::ApiError,
    terrain::{DemProcessor, StreamNetwork},
};

pub async fn upload_dem(
    State(state): State<AppState>,
    claims: Claims,
    Path(project_id): Path<Uuid>,
    mut multipart: Multipart,
) -> Result<impl IntoResponse, ApiError> {
    // 1. Verify project ownership
    let project = sqlx::query!(
        "SELECT id FROM projects WHERE id = $1 AND owner_id = $2",
        project_id,
        claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Project not found"))?;

    // 2. Extract uploaded file
    let mut dem_bytes: Vec<u8> = Vec::new();
    let mut filename = String::new();

    while let Some(field) = multipart.next_field().await? {
        if field.name() == Some("file") {
            filename = field.file_name().unwrap_or("dem.tif").to_string();
            dem_bytes = field.bytes().await?.to_vec();
        }
    }

    if dem_bytes.is_empty() {
        return Err(ApiError::BadRequest("No DEM file uploaded"));
    }

    // 3. Save to S3
    let s3_key = format!("dems/{}/{}_{}", project_id, Uuid::new_v4(), filename);
    state.s3_client
        .put_object()
        .bucket(&state.config.s3_bucket)
        .key(&s3_key)
        .body(dem_bytes.into())
        .send()
        .await?;

    let s3_url = format!("s3://{}/{}", state.config.s3_bucket, s3_key);

    // 4. Extract DEM metadata via GDAL
    let processor = DemProcessor::new(&s3_url)?;
    let metadata = processor.get_metadata()?;

    // 5. Insert DEM record
    let dem = sqlx::query!(
        r#"INSERT INTO dems
            (project_id, name, extent, crs, resolution_m, s3_url, file_size_bytes, source, metadata)
        VALUES ($1, $2, ST_GeomFromGeoJSON($3), $4, $5, $6, $7, $8, $9)
        RETURNING id, created_at"#,
        project_id,
        filename,
        serde_json::to_string(&metadata.extent)?,
        metadata.crs,
        metadata.resolution_m,
        s3_url,
        dem_bytes.len() as i64,
        "user_uploaded",
        serde_json::to_value(&metadata)?
    )
    .fetch_one(&state.db)
    .await?;

    Ok((StatusCode::CREATED, Json(serde_json::json!({
        "id": dem.id,
        "s3_url": s3_url,
        "metadata": metadata
    }))))
}

pub async fn extract_stream_network(
    State(state): State<AppState>,
    claims: Claims,
    Path(project_id): Path<Uuid>,
    Json(req): Json<StreamExtractionRequest>,
) -> Result<Json<StreamNetwork>, ApiError> {
    // Verify ownership
    sqlx::query!(
        "SELECT id FROM projects WHERE id = $1 AND owner_id = $2",
        project_id,
        claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Project not found"))?;

    // Get DEM
    let dem = sqlx::query!(
        "SELECT s3_url, crs FROM dems WHERE project_id = $1 ORDER BY created_at DESC LIMIT 1",
        project_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("No DEM found for project"))?;

    // Submit background job for stream extraction
    let job_id = Uuid::new_v4();
    state.redis
        .publish(
            "terrain:stream_extraction",
            serde_json::to_string(&serde_json::json!({
                "job_id": job_id,
                "project_id": project_id,
                "dem_s3_url": dem.s3_url,
                "threshold_cells": req.accumulation_threshold,
                "min_stream_length_m": req.min_stream_length_m
            }))?
        )
        .await?;

    Ok(Json(StreamNetwork {
        job_id,
        status: "processing".to_string(),
        streams: vec![],
    }))
}

#[derive(serde::Deserialize)]
pub struct StreamExtractionRequest {
    pub accumulation_threshold: i32,  // Cells (e.g., 1000 = streams with 1000+ upslope cells)
    pub min_stream_length_m: f64,
}
```

### 4. Structure Hydraulics (Bridge/Culvert)

```rust
// hydraulic-core/src/structures/bridge.rs

use crate::cross_section::CrossSection;
use crate::error::SolverError;

pub struct Bridge {
    pub low_chord_elevation: f64,  // m (soffit/deck underside)
    pub deck_elevation: f64,       // m (road surface)
    pub pier_width: f64,           // m (total pier obstruction)
    pub bridge_width: f64,         // m (opening width)
    pub loss_coefficient: f64,     // Dimensionless (0.3-0.5 typical)
}

impl Bridge {
    /// Compute bridge energy loss and determine flow regime
    pub fn compute_flow(
        &self,
        us_xs: &CrossSection,
        ds_xs: &CrossSection,
        flow_cfs: f64,
        us_wse: f64,
        ds_wse: f64,
    ) -> Result<BridgeFlowResult, SolverError> {
        let flow_cms = flow_cfs * 0.028317;

        // Check for pressure flow (upstream WSE > low chord)
        if us_wse > self.low_chord_elevation {
            return self.compute_pressure_flow(flow_cms, us_wse, ds_wse);
        }

        // Check for weir flow (upstream WSE > deck, flow overtops)
        if us_wse > self.deck_elevation {
            return self.compute_weir_flow(flow_cms, us_wse);
        }

        // Open channel flow (energy method)
        self.compute_energy_flow(us_xs, ds_xs, flow_cms, us_wse, ds_wse)
    }

    fn compute_energy_flow(
        &self,
        us_xs: &CrossSection,
        ds_xs: &CrossSection,
        flow_cms: f64,
        us_wse: f64,
        ds_wse: f64,
    ) -> Result<BridgeFlowResult, SolverError> {
        // Standard energy equation with bridge loss coefficient
        let us_depth = us_wse - us_xs.min_elevation();
        let ds_depth = ds_wse - ds_xs.min_elevation();

        let us_area = us_xs.flow_area(us_depth);
        let ds_area = ds_xs.flow_area(ds_depth);

        let us_velocity = flow_cms / us_area;
        let ds_velocity = flow_cms / ds_area;

        let vel_head_us = us_velocity.powi(2) / (2.0 * 9.81);
        let vel_head_ds = ds_velocity.powi(2) / (2.0 * 9.81);

        // Energy loss = K · ΔV²/(2g)
        let energy_loss = self.loss_coefficient * (vel_head_us - vel_head_ds);

        Ok(BridgeFlowResult {
            flow_type: BridgeFlowType::Energy,
            energy_loss_m: energy_loss,
            pressure_differential: None,
        })
    }

    fn compute_pressure_flow(
        &self,
        flow_cms: f64,
        us_wse: f64,
        ds_wse: f64,
    ) -> Result<BridgeFlowResult, SolverError> {
        // Orifice equation: Q = C_d · A · √(2g·ΔH)
        let bridge_area = self.bridge_width * (self.low_chord_elevation - ds_wse.min(self.low_chord_elevation - 1.0));
        let delta_h = us_wse - self.low_chord_elevation;
        let c_d = 0.8;  // Discharge coefficient for submerged orifice

        let q_capacity = c_d * bridge_area * (2.0 * 9.81 * delta_h).sqrt();

        Ok(BridgeFlowResult {
            flow_type: BridgeFlowType::Pressure,
            energy_loss_m: us_wse - ds_wse,
            pressure_differential: Some(delta_h),
        })
    }

    fn compute_weir_flow(
        &self,
        flow_cms: f64,
        us_wse: f64,
    ) -> Result<BridgeFlowResult, SolverError> {
        // Broad-crested weir: Q = C_w · L · H^(3/2)
        let h = us_wse - self.deck_elevation;
        let c_w = 1.7;  // Weir coefficient (SI units)
        let q_capacity = c_w * self.bridge_width * h.powf(1.5);

        Ok(BridgeFlowResult {
            flow_type: BridgeFlowType::Weir,
            energy_loss_m: h * 0.5,  // Approximate energy loss
            pressure_differential: None,
        })
    }
}

#[derive(Debug)]
pub struct BridgeFlowResult {
    pub flow_type: BridgeFlowType,
    pub energy_loss_m: f64,
    pub pressure_differential: Option<f64>,
}

#[derive(Debug)]
pub enum BridgeFlowType {
    Energy,    // Normal open channel flow under bridge
    Pressure,  // Submerged orifice flow
    Weir,      // Overtopping flow
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–4)

**Day 1:** Rust/Axum backend initialization with Docker/compose setup (PostgreSQL+PostGIS, Redis, MinIO)
**Day 2:** Database schema (13 tables with PostGIS), SQLx models, migrations
**Day 3:** Auth system (JWT, OAuth Google/GitHub, bcrypt password hashing)
**Day 4:** User/project/org CRUD APIs, auth middleware, integration tests

### Phase 2 — Hydraulic Solver Core (Days 5–12)

**Day 5-6:** Cross-section geometry (hydraulic properties: area, perimeter, conveyance, top width), standard step solver foundation (energy equation, Newton-Raphson convergence)
**Day 7:** Critical depth solver (Newton-Raphson for Q²/g=A³/T), mixed flow regime handling
**Day 8-10:** Structure hydraulics: bridges (energy/pressure/weir flow), culverts (inlet/outlet control HDS-5), weirs/gates (discharge equations)
**Day 11-12:** Hydrology: SCS Curve Number (Q=(P-Ia)²/(P-Ia+S)), design storms (SCS Type I/II/III, Chicago), unit hydrograph, Tc methods (Kirpich, Kerby)

### Phase 3 — Terrain Processing + GDAL (Days 13–18)

**Day 13:** DEM upload (GeoTIFF/LAS/LAZ via GDAL), metadata extraction, S3 multipart upload
**Day 14:** Pit filling (priority-flood Barnes 2014), D8 flow direction algorithm
**Day 15:** Flow accumulation, threshold-based stream delineation, network vectorization
**Day 16:** Cross-section extraction (transects perpendicular to centerline, elevation sampling at 1-5m, automatic bank detection)
**Day 17:** Manning's n estimation from land cover (NLCD/Corine lookup tables)
**Day 18:** Terrain worker (Redis job consumer, WebSocket progress, results to PostgreSQL+S3)

### Phase 4 — WASM Solver + Frontend (Days 19–25)

**Day 19:** WASM compilation (wasm-pack, wasm-bindgen entry points, wasm-opt for <1MB bundle)
**Day 20:** React/Vite scaffold, Mapbox GL JS map viewer, stream network layer, Zustand stores
**Day 21:** Cross-section editor (Canvas2D plot, draggable points, Manning's n input, bank markers)
**Day 22:** Longitudinal profile viewer (Canvas2D, streambed+WSE plots, zoom/pan, structure markers)
**Day 23:** Flood map overlay (Mapbox raster layer, depth = WSE - DEM, color ramp 0-2m)
**Day 24:** Simulation panel (flow input, BC selector, WASM/server routing, progress indicator)
**Day 25:** Results table (XS/WSE/depth/velocity/Froude, CSV export, sync with profile/map)

### Phase 5 — API Integration + Job Orchestration (Days 26–31)

**Day 26:** Simulation API (create/get/list), XS count routing (≤200→WASM, >200→server)
**Day 27:** Server simulation worker (Redis consumer, load model from DB, run solver, results to S3+PostgreSQL)
**Day 28:** WebSocket progress (Axum handler, pub/sub via Redis, React hook on frontend)
**Day 29:** Terrain API (DEM upload, stream/XS extraction endpoints, Redis job dispatch)
**Day 30:** Flood map worker (depth raster generation: WSE_interpolated - DEM, GeoTIFF to S3)
**Day 31:** PDF report worker (printpdf or headless Chrome, profile plot + XS table + map, presigned S3 URL)

### Phase 6 — Billing + Plan Enforcement (Days 32–35)

**Day 32:** Stripe integration (checkout/portal sessions, webhooks, plan tiers: Free 50XS/$0, Pro 200XS/$129, Advanced unlimited/$299)
**Day 33:** Usage tracking (plan limits middleware, compute minutes + storage tracking, usage dashboard API)
**Day 34:** Billing UI (plan comparison cards, usage meters, Stripe Customer Portal integration, upgrade prompts)
**Day 35:** Feature gating (PDF export→Pro, API→Advanced, locked indicators with upgrade CTAs, admin overrides)

### Phase 7 — Solver Validation + Testing (Days 36–39)

**Day 36:** Channel benchmarks (rectangular normal depth=2.15m, trapezoidal=1.82m, compound vs HEC-RAS, <1% tolerance)
**Day 37:** Structure benchmarks (bridge K=0.3, pressure flow, culvert inlet control vs HDS-5, weir Q=1.7·L·H^1.5)
**Day 38:** Hydrology benchmarks (SCS-CN Q=40.3mm, Type II peak Q=85 m³/s, unit hydrograph vs HEC-HMS)
**Day 39:** Performance testing (WASM 200XS<2s, server 1000XS<30s, convergence stress, 20 concurrent sims, memory profiling)

### Phase 8 — Deployment + Polish + Launch (Days 40–42)

**Day 40:** Docker/K8s setup (multi-stage Dockerfile with GDAL, compose for local dev, K8s manifests: API 3-replicas HPA, workers auto-scale, PostgreSQL StatefulSet, Redis, NGINX ingress TLS, health checks)
**Day 41:** CDN (CloudFront for assets/WASM/maps), monitoring (Prometheus/Grafana metrics, Sentry errors, structured logging), UI polish
**Day 42:** Launch prep (E2E smoke test, backup/restore, rate limiting 100req/min, security audit, landing page, docs, deploy + alerts)

---

## Critical Files

```
hydroworks/
├── hydraulic-core/                        # Shared solver (native + WASM)
│   ├── src/
│   │   ├── cross_section.rs               # Hydraulic geometry (area, perimeter, conveyance)
│   │   ├── solver/standard_step.rs        # Energy equation, Newton-Raphson WSE solver
│   │   ├── solver/critical_depth.rs       # Q²/g=A³/T solver, mixed flow
│   │   ├── structures/{bridge,culvert,weir,gate}.rs  # Structure hydraulics
│   │   ├── hydrology/{scs_cn,design_storms,unit_hydrograph}.rs
│   │   └── error.rs
│   └── tests/{benchmarks,structures,hydrology}.rs
├── solver-wasm/src/lib.rs                 # WASM bindings (wasm-bindgen)
├── hydroworks-api/
│   ├── src/
│   │   ├── main.rs                        # Axum app, routes
│   │   ├── api/handlers/{auth,projects,simulation,terrain,reaches,cross_sections,structures,billing}.rs
│   │   ├── terrain/{dem_processor,flow_direction,flow_accumulation,stream_delineation,cross_section_extractor}.rs
│   │   ├── workers/{simulation_worker,terrain_worker,flood_map_worker,pdf_worker}.rs
│   │   ├── auth/{mod,oauth}.rs            # JWT + OAuth
│   │   └── middleware/plan_limits.rs
│   ├── migrations/001_initial.sql         # Full schema (13 tables + PostGIS)
│   └── tests/{api_integration,simulation_e2e}.rs
├── frontend/src/
│   ├── stores/{authStore,projectStore,mapStore,simulationStore}.ts
│   ├── components/{MapViewer,StreamNetworkLayer,CrossSectionEditor,ProfileViewer,FloodMapLayer,SimulationPanel,ResultsPanel}.tsx
│   ├── pages/{Dashboard,Editor,Billing,Login}.tsx
│   ├── hooks/{useSimulationProgress,useWasmSolver}.ts
│   └── lib/{api,wasmLoader}.ts
├── calibration-service/calibration/{mannings_optimizer,curve_number_estimator}.py  # Python FastAPI
├── k8s/{api-deployment,worker-deployment,postgres-statefulset,redis-deployment,ingress}.yaml
├── docker-compose.yml
└── Dockerfile
```

---

## Solver Validation Suite

### Benchmark 1: Wide Rectangular Channel — Normal Depth
**Setup:**
- Width: 20 m
- Manning's n: 0.030
- Bed slope: 0.001 m/m
- Discharge: 100 m³/s

**Expected (Analytical):**
Normal depth via Manning's equation: y_n = ((Q·n)/(b·S^0.5))^(3/5) = 2.15 m

**Tolerance:** ±0.5% (±0.01 m)

**Verification:**
```rust
#[test]
fn benchmark_rectangular_normal_depth() {
    let xs = CrossSection::rectangular(20.0, 0.030);
    let solver = StandardStepSolver::new();
    solver.add_cross_section(xs.clone());

    let result = solver.solve_subcritical(100.0 * 35.315, 2.15 * 3.281).unwrap(); // CFS, ft
    let depth_m = result.depth[0] / 3.281;

    assert!((depth_m - 2.15).abs() < 0.01, "Normal depth mismatch: {depth_m} vs 2.15");
}
```

### Benchmark 2: Trapezoidal Channel — Subcritical Profile
**Setup:**
- Bottom width: 10 m
- Side slopes: 2:1 (H:V)
- Manning's n: 0.025
- Bed slope: 0.0005 m/m
- Discharge: 50 m³/s
- Reach length: 1000 m
- Downstream BC: Normal depth

**Expected:**
- Normal depth: ~1.82 m (analytical)
- Upstream depth: ~1.85 m (gradually-varied flow, M2 profile)

**Tolerance:** ±1% (±0.02 m)

**Verification:**
Compare water surface profile against HEC-RAS simulation with identical geometry.

### Benchmark 3: Bridge with Open Flow
**Setup:**
- Upstream/downstream XS: rectangular 15m wide, n=0.030
- Bridge: low chord 3.5 m, deck 4.5 m, loss coefficient K=0.3
- Flow: 75 m³/s
- Downstream WSE: 2.0 m (subcritical)

**Expected:**
- Upstream WSE: ~2.08 m (energy loss from bridge K)
- Energy loss: ~0.08 m

**Tolerance:** ±2% (±0.002 m)

### Benchmark 4: Culvert Inlet Control
**Setup:**
- Circular culvert: diameter 1.2 m, length 30 m, n=0.013
- Headwater: 2.5 m above invert
- Inlet: square edge (unsubmerged)

**Expected (HDS-5 Chart 1):**
- Discharge: ~3.8 m³/s

**Tolerance:** ±5% (inlet control is empirical)

### Benchmark 5: SCS-CN Runoff Depth
**Setup:**
- Curve Number: 75
- Rainfall: 100 mm (24-hr)
- Initial abstraction: Ia = 0.2S, S = (1000/CN - 10) = 3.33 inches = 84.6 mm

**Expected:**
- Runoff depth: Q = (P - 0.2S)² / (P - 0.2S + S) = 40.3 mm

**Tolerance:** ±1 mm

**Verification:**
```rust
#[test]
fn benchmark_scs_cn_runoff() {
    let basin = HydroBasin {
        curve_number: 75.0,
        area_km2: 10.0,
        ..Default::default()
    };

    let runoff_mm = compute_scs_runoff(basin.curve_number, 100.0);
    assert!((runoff_mm - 40.3).abs() < 1.0, "SCS runoff mismatch: {runoff_mm} vs 40.3 mm");
}
```

---

## Verification Checklist

### Core Solver
- [ ] Standard step method converges for subcritical flow (100 XS reach)
- [ ] Critical depth solver converges within 100 iterations
- [ ] Mixed flow regime detection (subcritical → critical → supercritical)
- [ ] Energy equation residual < 0.01 ft at convergence
- [ ] Froude number calculation correct (Fr < 1 subcritical, Fr > 1 supercritical)

### Structures
- [ ] Bridge energy loss matches theoretical K coefficient (±5%)
- [ ] Bridge pressure flow (submerged) uses orifice equation
- [ ] Culvert inlet/outlet control transitions correctly
- [ ] Weir discharge equation accurate (±5% vs. analytical)
- [ ] Gate flow capacity matches expected (free vs. submerged)

### Hydrology
- [ ] SCS-CN runoff depth matches analytical (±1%)
- [ ] Time of concentration methods (Kirpich, Kerby) within ±10% of published values
- [ ] Unit hydrograph peak time and volume correct
- [ ] Design storm hyetograph integrates to total rainfall depth

### Terrain Processing
- [ ] Pit filling removes all depressions
- [ ] Flow direction follows steepest descent (D8)
- [ ] Flow accumulation matches analytical for synthetic slope
- [ ] Stream network topology valid (no loops, correct connectivity)
- [ ] Cross-section extraction samples DEM correctly (±0.1m elevation error)

### WASM Solver
- [ ] WASM produces identical results to native solver (bit-exact or ±0.1%)
- [ ] WASM bundle size <1 MB gzipped
- [ ] WASM startup time <200ms
- [ ] 200 XS steady flow solves in <2 seconds (Chrome)

### API & Database
- [ ] All endpoints require authentication (except public routes)
- [ ] Plan limits enforced (XS count, storage, compute minutes)
- [ ] WebSocket connection handles disconnect gracefully
- [ ] S3 presigned URLs expire after 1 hour
- [ ] PostGIS spatial queries use indexes (EXPLAIN ANALYZE)

### Frontend
- [ ] Map loads with terrain and stream network
- [ ] Cross-section editor updates database on edit
- [ ] Profile viewer syncs with simulation results
- [ ] Flood map overlay renders depth correctly
- [ ] Simulation progress updates in real-time via WebSocket

---

## Deployment Architecture

### Production Stack (AWS)

**Compute:**
- EKS cluster (Kubernetes 1.28+)
- API pods: 3 replicas (t3.large), HPA scaling 3-10 based on CPU/memory
- Simulation worker pods: 2-10 replicas (c7g.2xlarge ARM), auto-scale on Redis queue depth
- Terrain worker pods: 2-5 replicas (c7g.4xlarge, GDAL + high CPU), auto-scale on queue depth

**Database:**
- RDS PostgreSQL 16 with PostGIS (db.r6g.xlarge)
- Multi-AZ for HA
- Read replicas for analytics queries
- Automated backups (daily snapshots, 7-day retention)

**Cache & Queue:**
- ElastiCache Redis 7 (cache.r6g.large)
- Cluster mode for high availability
- Used for: job queue, session cache, WebSocket pub/sub

**Storage:**
- S3 buckets:
  - `hydroworks-dems`: DEMs (lifecycle: archive to Glacier after 90 days)
  - `hydroworks-results`: Simulation results, flood maps (lifecycle: delete after 30 days)
  - `hydroworks-reports`: PDF reports (lifecycle: delete after 90 days)
- CloudFront CDN:
  - Frontend assets (TTL 1 year, versioned)
  - WASM bundle (TTL 1 week)
  - Flood maps (TTL 1 hour)

**Networking:**
- ALB (Application Load Balancer) for API
- NGINX Ingress Controller for WebSocket sticky sessions
- TLS via AWS Certificate Manager (auto-renewal)
- VPC with private subnets for database/workers, public subnets for API

**Monitoring:**
- CloudWatch Logs for structured logs
- Prometheus + Grafana (on EKS) for metrics
- Sentry for error tracking
- Uptime monitoring: Pingdom or UptimeRobot

### Scaling Targets
- API: HPA at 70% CPU, scale 3→10 pods
- Simulation workers: HPA on Redis queue depth (>10 jobs → scale up)
- Database: Read replicas if read IOPS >1000/sec
- Expected capacity: 500 concurrent users, 100 simulations/minute

---

## Post-MVP Roadmap

### v1.1 — Unsteady 1D Hydraulics (Months 2-3)
Preissmann implicit finite difference solver for Saint-Venant equations, hydrograph routing, dam breach analysis (Froehlich), time-varying BCs. Use case: dam failure flood mapping.

### v1.2 — 2D Shallow Water Solver (Months 4-6)
GPU-accelerated finite volume SWE solver (CUDA), structured/unstructured grids, rainfall-on-grid, 1D-2D coupling. Use case: urban flooding, coastal storm surge.

### v1.3 — Stormwater Pipe Networks (Months 7-8)
1D pipe network solver, manholes/inlets (HEC-22), surcharge/pressurized flow, 1D pipe + 2D surface coupling, SWMM import. Use case: urban stormwater, CSO.

### v1.4 — Sediment Transport (Months 9-10)
Bed load (Meyer-Peter-Müller, Engelund-Hansen), suspended load, bed evolution (Exner), armoring/sorting. Use case: channel scour, reservoir sedimentation.

### v1.5 — Water Quality (Months 11-12)
Conservative tracers, DO/BOD (Streeter-Phelps), nutrients (N/P kinetics), temperature, algae growth. Use case: wastewater discharge, TMDL.

### v1.6 — Real-Time Forecasting (Months 13-15)
Live rainfall data (NOAA NEXRAD), continuous simulation with soil moisture, ensemble forecasting, alert thresholds. Use case: municipal flood warning systems.

### v1.7 — Climate Change Scenarios (Months 16-18)
Projected IDF curves (NOAA Atlas 14, UKCP18), sea level rise (NOAA/IPCC), CMIP6 ensemble data, current vs future comparison reports. Use case: climate adaptation.

### v1.8 — API + Automation (Months 19-21)
REST API for headless simulation, batch scenarios (Monte Carlo/sensitivity), Python SDK, webhook notifications. Use case: automated flood risk in development approval.

---

