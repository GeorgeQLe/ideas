# 81. GeoSat — Satellite Orbit Design and Analysis Platform

## Implementation Plan

**MVP Scope:** Cloud-based satellite mission planning platform with browser-based orbit designer (Keplerian elements: a, e, i, RAAN, AoP, TA), orbit type wizards (sun-synchronous, repeat ground track, frozen orbit, GEO), high-fidelity propagation engine (J2-J6 geopotential EGM96, atmospheric drag NRLMSISE-00, solar radiation pressure with shadow modeling) compiled to WebAssembly for client-side execution of orbital propagation ≤24 hours and server-side Rust-native execution for multi-week campaigns, Walker-Delta and Walker-Star constellation generators with automated coverage analysis (revisit time, gap statistics, min elevation angle per ground target), 3D globe visualization with CesiumJS rendering orbital ground tracks and coverage footprints in real-time, RF link budget calculator with dynamic geometry (EIRP - FSPL - atmospheric losses + G/T → link margin) integrated with contact windows, impulsive maneuver planner (Hohmann transfer, bi-elliptic, plane change, combined) with delta-V budget tracker, eclipse and thermal environment analysis, TLE/GP orbit determination from Space-Track catalog (automatic daily updates), SPICE kernel import for planetary ephemerides, PostgreSQL metadata + S3 ephemeris storage, CCSDS OEM/OPM export, Stripe billing with three tiers (Student Free / Pro $149/mo / Advanced $349/user/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| Orbit Propagator | Rust (native + WASM) | Custom numerical integrator (RK7(8) Dormand-Prince, Adams-Bashforth-Moulton) + SGP4/SDP4 for TLE |
| WASM Compilation | `wasm-pack` + `wasm-bindgen` | Client-side propagation for campaigns ≤24 hours |
| Coverage Engine | Rust (grid computation) | Parallel grid-based coverage analysis with Rayon |
| Link Budget | Rust (RF equations + ITU models) | ITU-R P.676 gaseous absorption, P.618 rain fade, compiled to WASM |
| Maneuver Optimizer | Rust + Python (SciPy, IPOPT) | Impulsive maneuver planning (Rust), low-thrust optimization (Python FastAPI) |
| COLA Screening | Rust | Conjunction assessment, probability of collision (Pc), CDM parsing |
| AI/ML Services | Python 3.12 (FastAPI) | Orbit anomaly detection (PyTorch), maneuver recommendation, launch window prediction |
| Database | PostgreSQL 16 | Missions, spacecraft, ground stations, constellation definitions, ephemeris metadata |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | Ephemeris files (OEM/OPM), TLE catalog archives, coverage grids, mission reports |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| 3D Visualization | CesiumJS 1.114 | Globe rendering, orbit tracks, ground coverage, sensor FOVs, satellite models |
| 2D Plots | D3.js + Canvas | Orbit element evolution, link budget vs. time, eclipse plots |
| Real-time | WebSocket (Axum) | Live propagation progress, constellation deployment simulation |
| Job Queue | Redis 7 + Tokio tasks | Server-side long-duration propagation, constellation coverage batch jobs |
| Cache | Redis 7 | TLE catalog cache (1-day TTL), ephemeris hot cache, coverage grid cache |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | WASM bundle delivery (orbit propagator ~1.8MB gzipped), static frontend assets |
| Monitoring | Prometheus + Grafana, Sentry | Propagation performance metrics, convergence failures, error tracking |
| CI/CD | GitHub Actions | WASM build pipeline, Rust tests, Docker image push, ephemeris validation |

### Key Architecture Decisions

1. **Hybrid client/server propagator with 24-hour WASM threshold**: Short-term orbital propagations (≤24 hours, covers quick analysis, single-pass contact predictions, and daily coverage) run entirely in the browser via WASM, providing instant feedback with zero server cost. Multi-day or multi-week campaigns (constellation deployment simulations, long-term perturbation analysis, station-keeping drift) are submitted to the Rust-native server propagator which handles high-fidelity multi-week propagations with adaptive timestep control. The threshold is configurable per plan tier.

2. **Custom orbital propagator in Rust rather than wrapping NASA GMAT or Orekit**: Building a custom numerical integrator in Rust gives us full control over timestep adaptation, WASM compilation, and parallelization for constellation propagation. GMAT's C++ codebase is desktop-focused with Java GUI dependencies that make WASM compilation infeasible. Orekit (Java) requires JVM which is incompatible with browser execution. Our Rust propagator uses Dormand-Prince RK7(8) with embedded error estimation for adaptive timestep, SGP4 via NIST-certified implementation for TLE-based propagation, and nalgebra for matrix operations.

3. **CesiumJS for 3D globe visualization with custom orbit rendering**: CesiumJS provides a high-performance WebGL-based globe with built-in WGS84 ellipsoid, time-dynamic rendering, and CZML support. We render orbital ground tracks as polylines with time-based color coding, coverage footprints as dynamic polygons computed from sensor FOV geometry, and satellite models with attitude visualization. Alternative (Three.js + custom planet rendering) was rejected due to the need to reimplement coordinate frame transformations, day/night terminator, and terrain rendering.

4. **Grid-based coverage analysis with spatial indexing**: Coverage analysis discretizes the target region (global, latitude band, or custom polygon) into a uniform grid (configurable: 0.5°, 1°, 2.5°, 5° resolution), then for each grid cell computes access intervals from all constellation satellites accounting for minimum elevation angle, line-of-sight obstruction, and sensor FOV constraints. An R-tree spatial index (rstar crate) enables O(log n) queries for "which satellites can see this target at time T". Results are stored as coverage statistics (revisit time, gap duration, number of passes per day) per grid cell and rendered as a heat map on the 3D globe.

5. **S3 for ephemeris storage with PostgreSQL metadata catalog**: Long-duration propagation results (ephemeris: time, position, velocity for each timestep) are stored in S3 as CCSDS OEM (Orbit Ephemeris Message) or binary files (typically 100KB-10MB for multi-week campaigns), while PostgreSQL holds searchable metadata (mission ID, epoch, propagation mode, validity period, coordinate frame). This allows ephemeris archives to scale to 100K+ files without bloating the database while enabling fast parametric search via PostgreSQL indexes. Ephemeris files are cached in Redis for hot access (recently propagated orbits).

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "postgis";  -- For geospatial queries on ground targets
CREATE EXTENSION IF NOT EXISTS "pg_trgm";  -- For fuzzy text search on spacecraft/missions

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

-- Organizations (for Advanced plan multi-user teams)
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

-- Missions (workspace for satellite mission planning)
CREATE TABLE missions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    mission_type TEXT DEFAULT 'leo',  -- leo | geo | meo | lunar | interplanetary
    launch_date TIMESTAMPTZ,
    mission_duration_days INTEGER,
    settings JSONB DEFAULT '{}',  -- Mission-wide settings (coordinate frame defaults, time system)
    is_public BOOLEAN NOT NULL DEFAULT false,
    forked_from UUID REFERENCES missions(id) ON DELETE SET NULL,
    thumbnail_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX missions_owner_idx ON missions(owner_id);
CREATE INDEX missions_org_idx ON missions(org_id);
CREATE INDEX missions_updated_idx ON missions(updated_at DESC);

-- Spacecraft (individual satellite definitions)
CREATE TABLE spacecraft (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    mission_id UUID NOT NULL REFERENCES missions(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    norad_id INTEGER,  -- NORAD catalog ID if available
    mass_kg REAL NOT NULL DEFAULT 100.0,
    drag_area_m2 REAL NOT NULL DEFAULT 0.1,
    drag_coefficient REAL NOT NULL DEFAULT 2.2,
    srp_area_m2 REAL NOT NULL DEFAULT 0.1,
    srp_coefficient REAL NOT NULL DEFAULT 1.3,
    propulsion JSONB DEFAULT '{}',  -- {type: "chemical|electric", isp_s, thrust_n, fuel_mass_kg}
    initial_state JSONB NOT NULL,  -- Keplerian elements or Cartesian state vector (epoch, a, e, i, raan, aop, ta) or (epoch, x, y, z, vx, vy, vz)
    state_type TEXT NOT NULL DEFAULT 'keplerian',  -- keplerian | cartesian | tle
    tle_line1 TEXT,
    tle_line2 TEXT,
    attitude_mode TEXT DEFAULT 'nadir',  -- nadir | inertial | sun_pointing | target_pointing | custom
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX spacecraft_mission_idx ON spacecraft(mission_id);
CREATE INDEX spacecraft_norad_idx ON spacecraft(norad_id);

-- Constellations (multi-satellite systems)
CREATE TABLE constellations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    mission_id UUID NOT NULL REFERENCES missions(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    constellation_type TEXT NOT NULL,  -- walker_delta | walker_star | flower | custom
    parameters JSONB NOT NULL,  -- Walker: {planes, sats_per_plane, phasing, altitude_km, inclination_deg}
    spacecraft_template_id UUID REFERENCES spacecraft(id),  -- Template spacecraft for mass properties
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX constellations_mission_idx ON constellations(mission_id);

-- Constellation Members (individual satellites in constellation)
CREATE TABLE constellation_members (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    constellation_id UUID NOT NULL REFERENCES constellations(id) ON DELETE CASCADE,
    spacecraft_id UUID NOT NULL REFERENCES spacecraft(id) ON DELETE CASCADE,
    plane_index INTEGER NOT NULL,
    slot_index INTEGER NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX constellation_members_constellation_idx ON constellation_members(constellation_id);
CREATE INDEX constellation_members_spacecraft_idx ON constellation_members(spacecraft_id);

-- Ground Stations
CREATE TABLE ground_stations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    mission_id UUID NOT NULL REFERENCES missions(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    latitude_deg REAL NOT NULL,
    longitude_deg REAL NOT NULL,
    altitude_m REAL NOT NULL DEFAULT 0.0,
    location GEOGRAPHY(POINT, 4326),  -- PostGIS geography for spatial queries
    min_elevation_deg REAL NOT NULL DEFAULT 5.0,
    frequency_band TEXT,  -- uhf | s | x | ka | optical
    antenna_diameter_m REAL,
    antenna_gain_dbi REAL,
    system_noise_temp_k REAL DEFAULT 290.0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX ground_stations_mission_idx ON ground_stations(mission_id);
CREATE INDEX ground_stations_location_idx ON ground_stations USING GIST(location);

-- Propagation Jobs (orbital propagation tasks)
CREATE TABLE propagation_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    spacecraft_id UUID NOT NULL REFERENCES spacecraft(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    propagation_mode TEXT NOT NULL,  -- sgp4 | high_fidelity | analytical
    perturbations JSONB NOT NULL,  -- {j2: true, j4: true, drag: true, srp: true, moon: false, sun: false}
    integrator TEXT DEFAULT 'rk78',  -- rk78 | abm | gauss_legendre
    start_epoch TIMESTAMPTZ NOT NULL,
    end_epoch TIMESTAMPTZ NOT NULL,
    timestep_seconds REAL NOT NULL DEFAULT 60.0,
    execution_mode TEXT NOT NULL DEFAULT 'wasm',  -- wasm | server
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | running | completed | failed | cancelled
    ephemeris_url TEXT,  -- S3 URL for ephemeris file (OEM or binary)
    ephemeris_format TEXT DEFAULT 'oem',  -- oem | binary | csv
    ephemeris_points INTEGER,
    error_message TEXT,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    wall_time_ms INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX propagation_jobs_spacecraft_idx ON propagation_jobs(spacecraft_id);
CREATE INDEX propagation_jobs_user_idx ON propagation_jobs(user_id);
CREATE INDEX propagation_jobs_status_idx ON propagation_jobs(status);
CREATE INDEX propagation_jobs_created_idx ON propagation_jobs(created_at DESC);

-- Server Propagation Queue (for server-side execution only)
CREATE TABLE propagation_queue (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    propagation_job_id UUID NOT NULL REFERENCES propagation_jobs(id) ON DELETE CASCADE,
    worker_id TEXT,
    priority INTEGER DEFAULT 0,  -- Higher = more urgent
    progress_pct REAL DEFAULT 0.0,
    current_epoch TIMESTAMPTZ,
    convergence_data JSONB DEFAULT '[]',  -- [{epoch, timestep, error_estimate}]
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX propagation_queue_job_idx ON propagation_queue(propagation_job_id);
CREATE INDEX propagation_queue_status_idx ON propagation_queue(worker_id);

-- Coverage Analysis Jobs
CREATE TABLE coverage_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    constellation_id UUID REFERENCES constellations(id) ON DELETE CASCADE,
    spacecraft_id UUID REFERENCES spacecraft(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    target_region JSONB NOT NULL,  -- {type: "global" | "latitude_band" | "polygon", bounds: {...}}
    grid_resolution_deg REAL NOT NULL DEFAULT 1.0,
    min_elevation_deg REAL NOT NULL DEFAULT 5.0,
    start_epoch TIMESTAMPTZ NOT NULL,
    end_epoch TIMESTAMPTZ NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending',
    coverage_url TEXT,  -- S3 URL for coverage grid results
    summary JSONB,  -- {avg_revisit_s, max_gap_s, coverage_pct, total_accesses}
    error_message TEXT,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    wall_time_ms INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX coverage_jobs_constellation_idx ON coverage_jobs(constellation_id);
CREATE INDEX coverage_jobs_spacecraft_idx ON coverage_jobs(spacecraft_id);
CREATE INDEX coverage_jobs_status_idx ON coverage_jobs(status);

-- Contact Windows (ground station access intervals)
CREATE TABLE contact_windows (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    spacecraft_id UUID NOT NULL REFERENCES spacecraft(id) ON DELETE CASCADE,
    ground_station_id UUID NOT NULL REFERENCES ground_stations(id) ON DELETE CASCADE,
    aos TIMESTAMPTZ NOT NULL,  -- Acquisition of Signal
    los TIMESTAMPTZ NOT NULL,  -- Loss of Signal
    max_elevation_deg REAL NOT NULL,
    tca TIMESTAMPTZ NOT NULL,  -- Time of Closest Approach (max elevation time)
    duration_seconds INTEGER NOT NULL,
    link_budget JSONB,  -- {min_margin_db, avg_margin_db, max_range_km, doppler_hz}
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX contact_windows_spacecraft_idx ON contact_windows(spacecraft_id);
CREATE INDEX contact_windows_gs_idx ON contact_windows(ground_station_id);
CREATE INDEX contact_windows_aos_idx ON contact_windows(aos);

-- Maneuvers (orbital maneuver planning)
CREATE TABLE maneuvers (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    spacecraft_id UUID NOT NULL REFERENCES spacecraft(id) ON DELETE CASCADE,
    maneuver_type TEXT NOT NULL,  -- hohmann | bielliptic | plane_change | combined | station_keeping | phasing | cola_avoidance
    epoch TIMESTAMPTZ NOT NULL,
    delta_v_ms REAL NOT NULL,  -- Total delta-V magnitude in m/s
    delta_v_vector JSONB,  -- {x, y, z} in specified frame
    burn_duration_s REAL,
    parameters JSONB NOT NULL,  -- Maneuver-specific parameters
    fuel_consumed_kg REAL,
    status TEXT NOT NULL DEFAULT 'planned',  -- planned | executed | cancelled
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX maneuvers_spacecraft_idx ON maneuvers(spacecraft_id);
CREATE INDEX maneuvers_epoch_idx ON maneuvers(epoch);

-- TLE Catalog (two-line element sets from Space-Track)
CREATE TABLE tle_catalog (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    norad_id INTEGER NOT NULL,
    name TEXT NOT NULL,
    epoch TIMESTAMPTZ NOT NULL,
    line1 TEXT NOT NULL,
    line2 TEXT NOT NULL,
    object_type TEXT,  -- payload | rocket_body | debris | unknown
    rcs_m2 REAL,  -- Radar cross-section
    launch_date DATE,
    decay_date DATE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(norad_id, epoch)
);
CREATE INDEX tle_catalog_norad_idx ON tle_catalog(norad_id);
CREATE INDEX tle_catalog_epoch_idx ON tle_catalog(epoch DESC);
CREATE INDEX tle_catalog_updated_idx ON tle_catalog(updated_at DESC);

-- Conjunction Events (close approach events for COLA)
CREATE TABLE conjunction_events (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    spacecraft_id UUID NOT NULL REFERENCES spacecraft(id) ON DELETE CASCADE,
    target_norad_id INTEGER NOT NULL,
    tca TIMESTAMPTZ NOT NULL,  -- Time of Closest Approach
    miss_distance_km REAL NOT NULL,
    relative_velocity_ms REAL NOT NULL,
    probability_collision REAL,  -- Pc (0.0 to 1.0)
    cdm_url TEXT,  -- S3 URL to Conjunction Data Message XML
    status TEXT NOT NULL DEFAULT 'open',  -- open | screening | maneuver_planned | cleared
    risk_level TEXT NOT NULL DEFAULT 'low',  -- low | medium | high | critical
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX conjunction_events_spacecraft_idx ON conjunction_events(spacecraft_id);
CREATE INDEX conjunction_events_tca_idx ON conjunction_events(tca);
CREATE INDEX conjunction_events_status_idx ON conjunction_events(status);

-- Usage Tracking (for billing enforcement)
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,  -- propagation_hours | coverage_jobs | storage_gb
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
pub struct Mission {
    pub id: Uuid,
    pub owner_id: Uuid,
    pub org_id: Option<Uuid>,
    pub name: String,
    pub description: String,
    pub mission_type: String,
    pub launch_date: Option<DateTime<Utc>>,
    pub mission_duration_days: Option<i32>,
    pub settings: serde_json::Value,
    pub is_public: bool,
    pub forked_from: Option<Uuid>,
    pub thumbnail_url: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Spacecraft {
    pub id: Uuid,
    pub mission_id: Uuid,
    pub name: String,
    pub norad_id: Option<i32>,
    pub mass_kg: f32,
    pub drag_area_m2: f32,
    pub drag_coefficient: f32,
    pub srp_area_m2: f32,
    pub srp_coefficient: f32,
    pub propulsion: serde_json::Value,
    pub initial_state: serde_json::Value,
    pub state_type: String,
    pub tle_line1: Option<String>,
    pub tle_line2: Option<String>,
    pub attitude_mode: String,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Constellation {
    pub id: Uuid,
    pub mission_id: Uuid,
    pub name: String,
    pub constellation_type: String,
    pub parameters: serde_json::Value,
    pub spacecraft_template_id: Option<Uuid>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct GroundStation {
    pub id: Uuid,
    pub mission_id: Uuid,
    pub name: String,
    pub latitude_deg: f32,
    pub longitude_deg: f32,
    pub altitude_m: f32,
    pub min_elevation_deg: f32,
    pub frequency_band: Option<String>,
    pub antenna_diameter_m: Option<f32>,
    pub antenna_gain_dbi: Option<f32>,
    pub system_noise_temp_k: f32,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct PropagationJob {
    pub id: Uuid,
    pub spacecraft_id: Uuid,
    pub user_id: Uuid,
    pub propagation_mode: String,
    pub perturbations: serde_json::Value,
    pub integrator: String,
    pub start_epoch: DateTime<Utc>,
    pub end_epoch: DateTime<Utc>,
    pub timestep_seconds: f32,
    pub execution_mode: String,
    pub status: String,
    pub ephemeris_url: Option<String>,
    pub ephemeris_format: String,
    pub ephemeris_points: Option<i32>,
    pub error_message: Option<String>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub wall_time_ms: Option<i32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct CoverageJob {
    pub id: Uuid,
    pub constellation_id: Option<Uuid>,
    pub spacecraft_id: Option<Uuid>,
    pub user_id: Uuid,
    pub target_region: serde_json::Value,
    pub grid_resolution_deg: f32,
    pub min_elevation_deg: f32,
    pub start_epoch: DateTime<Utc>,
    pub end_epoch: DateTime<Utc>,
    pub status: String,
    pub coverage_url: Option<String>,
    pub summary: Option<serde_json::Value>,
    pub error_message: Option<String>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub wall_time_ms: Option<i32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct ContactWindow {
    pub id: Uuid,
    pub spacecraft_id: Uuid,
    pub ground_station_id: Uuid,
    pub aos: DateTime<Utc>,
    pub los: DateTime<Utc>,
    pub max_elevation_deg: f32,
    pub tca: DateTime<Utc>,
    pub duration_seconds: i32,
    pub link_budget: Option<serde_json::Value>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Maneuver {
    pub id: Uuid,
    pub spacecraft_id: Uuid,
    pub maneuver_type: String,
    pub epoch: DateTime<Utc>,
    pub delta_v_ms: f32,
    pub delta_v_vector: Option<serde_json::Value>,
    pub burn_duration_s: Option<f32>,
    pub parameters: serde_json::Value,
    pub fuel_consumed_kg: Option<f32>,
    pub status: String,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct KeplerianElements {
    pub epoch: DateTime<Utc>,
    pub semi_major_axis_km: f64,
    pub eccentricity: f64,
    pub inclination_deg: f64,
    pub raan_deg: f64,  // Right Ascension of Ascending Node
    pub arg_periapsis_deg: f64,
    pub true_anomaly_deg: f64,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct CartesianState {
    pub epoch: DateTime<Utc>,
    pub x_km: f64,
    pub y_km: f64,
    pub z_km: f64,
    pub vx_km_s: f64,
    pub vy_km_s: f64,
    pub vz_km_s: f64,
    pub frame: String,  // J2000 | ECEF | GCRF
}

#[derive(Debug, Deserialize, Serialize)]
pub struct PerturbationConfig {
    pub j2: bool,
    pub j3: bool,
    pub j4: bool,
    pub j6: bool,
    pub drag: bool,
    pub srp: bool,
    pub moon: bool,
    pub sun: bool,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct WalkerConstellationParams {
    pub num_planes: u32,
    pub sats_per_plane: u32,
    pub phasing_parameter: u32,  // F in Walker notation T/P/F
    pub altitude_km: f64,
    pub inclination_deg: f64,
}
```

---

## Orbit Propagator Architecture Deep-Dive

### Governing Equations and Numerical Integration

GeoSat's core propagator solves the **two-body problem with perturbations**, integrating the equations of motion for a satellite subject to Earth's gravitational field (geopotential harmonics), atmospheric drag, solar radiation pressure, and third-body gravitational effects from the Moon and Sun.

The state vector **X = [r, v]** (position and velocity in an inertial frame) evolves according to:

```
d²r/dt² = -μ/r³ · r + a_J2 + a_drag + a_srp + a_moon + a_sun
```

Where:
- **μ = 398600.4418 km³/s²** — Earth's gravitational parameter (GM)
- **a_J2** — Acceleration due to J2 oblateness perturbation (largest non-spherical term)
- **a_drag** — Atmospheric drag acceleration
- **a_srp** — Solar radiation pressure acceleration
- **a_moon, a_sun** — Third-body perturbations from Moon and Sun

**J2 perturbation** (Earth's equatorial bulge) is the dominant non-spherical gravitational term:

```
a_J2 = (3/2) · (J2 · μ · R_e² / r⁵) · [
    (5(z/r)² - 1) · x î + (5(z/r)² - 1) · y ĵ + (5(z/r)² - 3) · z k̂
]

where J2 = 1.08262668e-3, R_e = 6378.137 km
```

Higher-order geopotential terms (J3, J4, J6) add zonal harmonic corrections for improved fidelity.

**Atmospheric drag** uses exponential atmosphere density models (NRLMSISE-00 or Harris-Priester):

```
a_drag = -(1/2) · (ρ · C_d · A / m) · |v_rel| · v_rel

where:
  ρ = atmospheric density (kg/m³) — exponential decay with altitude
  C_d = drag coefficient (typically 2.0-2.5)
  A = cross-sectional area (m²)
  m = satellite mass (kg)
  v_rel = velocity relative to rotating atmosphere
```

**Solar radiation pressure** (SRP):

```
a_srp = -(P_☉ / c) · (C_r · A / m) · (AU / r_☉)² · ŝ_☉ · ν

where:
  P_☉ = 4.56e-6 N/m² — solar constant at 1 AU
  c = speed of light
  C_r = reflectivity coefficient (1.0-2.0)
  AU = 1 AU = 1.49597870e8 km
  r_☉ = distance from Sun
  ŝ_☉ = unit vector toward Sun
  ν = shadow function (0 in eclipse, 1 in sunlight, fractional in penumbra)
```

Shadow modeling uses conical Earth shadow with penumbra:
- **Umbra**: complete shadow (ν = 0)
- **Penumbra**: partial shadow (ν = fractional based on solid angle)
- **Sunlight**: no shadow (ν = 1)

**Numerical integrator**: Dormand-Prince RK7(8) with embedded error estimation for adaptive timestep:

```
Estimated local error: ε = ||y_7 - y_8|| (difference between 7th and 8th order solutions)

Timestep adaptation:
  h_new = 0.9 · h · (tolerance / ε)^(1/8)  (clamped to [h_min, h_max])
```

The integrator automatically reduces timestep during close approaches, atmospheric drag transients, or eclipse transitions.

### Client/Server Split (WASM Threshold)

```
Propagation request → Duration extracted
    │
    ├── ≤24 hours → WASM propagator (browser)
    │   ├── Instant startup, no network latency
    │   ├── Results streamed incrementally
    │   └── No server cost
    │
    └── >24 hours → Server propagator (Rust native)
        ├── Job queued via Redis
        ├── Worker picks up, runs native propagator
        ├── Progress streamed via WebSocket
        └── Ephemeris stored in S3, URL returned
```

The 24-hour threshold was chosen because:
- WASM propagator handles 24-hour LEO propagation (86,400 seconds, ~15 orbits for 550km LEO) in <3 seconds on modern hardware
- 24 hours covers: single-day contact window prediction, quick coverage check, orbital decay analysis over 1 day
- Beyond 24 hours: multi-week station-keeping drift analysis, constellation deployment (weeks to months), long-duration perturbation studies → need server compute

### WASM Compilation Pipeline

```toml
# propagator-wasm/Cargo.toml
[package]
name = "geosat-propagator-wasm"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
wasm-bindgen = "0.2"
serde = { version = "1", features = ["derive"] }
serde-wasm-bindgen = "0.6"
js-sys = "0.3"
nalgebra = "0.32"
chrono = { version = "0.4", features = ["wasmbind"] }
log = "0.4"
console_log = "1"

[profile.release]
opt-level = "z"       # Optimize for size
lto = true            # Link-time optimization
codegen-units = 1     # Single codegen unit for better optimization
strip = true          # Strip debug symbols
wasm-opt = true       # Run wasm-opt
```

```yaml
# .github/workflows/wasm-build.yml
name: Build WASM Propagator
on:
  push:
    paths: ['propagator-wasm/**', 'propagator-core/**']
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
      - run: cd propagator-wasm && wasm-pack build --target web --release
      - run: wasm-opt -Oz propagator-wasm/pkg/geosat_propagator_wasm_bg.wasm -o propagator-wasm/pkg/geosat_propagator_wasm_bg.wasm
      - name: Upload to S3/CDN
        run: aws s3 cp propagator-wasm/pkg/ s3://geosat-wasm/v${{ github.sha }}/ --recursive
```

---

## Architecture Deep-Dives

### 1. Propagation API Handler (Rust/Axum)

The primary endpoint receives a propagation request, validates the spacecraft state, decides between WASM and server execution, and for server execution enqueues a job with Redis and streams progress via WebSocket.

```rust
// src/api/handlers/propagation.rs

use axum::{
    extract::{Path, State, Json},
    http::StatusCode,
    response::IntoResponse,
};
use chrono::{DateTime, Utc, Duration};
use uuid::Uuid;
use crate::{
    db::models::{PropagationJob, Spacecraft, PerturbationConfig},
    state::AppState,
    auth::Claims,
    error::ApiError,
};

#[derive(serde::Deserialize)]
pub struct CreatePropagationRequest {
    pub propagation_mode: String,  // sgp4 | high_fidelity | analytical
    pub perturbations: PerturbationConfig,
    pub integrator: Option<String>,  // rk78 | abm | gauss_legendre
    pub start_epoch: DateTime<Utc>,
    pub end_epoch: DateTime<Utc>,
    pub timestep_seconds: Option<f32>,
    pub ephemeris_format: Option<String>,  // oem | binary | csv
}

pub async fn create_propagation(
    State(state): State<AppState>,
    claims: Claims,
    Path(spacecraft_id): Path<Uuid>,
    Json(req): Json<CreatePropagationRequest>,
) -> Result<impl IntoResponse, ApiError> {
    // 1. Verify spacecraft ownership
    let spacecraft = sqlx::query_as!(
        Spacecraft,
        "SELECT sc.* FROM spacecraft sc
         JOIN missions m ON sc.mission_id = m.id
         WHERE sc.id = $1 AND (m.owner_id = $2 OR m.org_id IN (
             SELECT org_id FROM org_members WHERE user_id = $2
         ))",
        spacecraft_id,
        claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Spacecraft not found"))?;

    // 2. Check plan limits
    let user = sqlx::query_as!(
        crate::db::models::User,
        "SELECT * FROM users WHERE id = $1",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    let duration = (req.end_epoch - req.start_epoch).num_hours();

    if user.plan == "free" && duration > 24 {
        return Err(ApiError::PlanLimit(
            "Free plan supports propagation up to 24 hours. Upgrade to Pro for unlimited."
        ));
    }

    if user.plan == "free" && req.propagation_mode == "high_fidelity" {
        return Err(ApiError::PlanLimit(
            "Free plan supports SGP4 only. Upgrade to Pro for high-fidelity propagation."
        ));
    }

    // 3. Determine execution mode
    let execution_mode = if duration <= 24 {
        "wasm"
    } else {
        "server"
    };

    // 4. Create propagation job record
    let job = sqlx::query_as!(
        PropagationJob,
        r#"INSERT INTO propagation_jobs
            (spacecraft_id, user_id, propagation_mode, perturbations,
             integrator, start_epoch, end_epoch, timestep_seconds,
             execution_mode, status, ephemeris_format)
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11)
        RETURNING *"#,
        spacecraft_id,
        claims.user_id,
        req.propagation_mode,
        serde_json::to_value(&req.perturbations)?,
        req.integrator.unwrap_or_else(|| "rk78".to_string()),
        req.start_epoch,
        req.end_epoch,
        req.timestep_seconds.unwrap_or(60.0),
        execution_mode,
        if execution_mode == "wasm" { "completed" } else { "pending" },
        req.ephemeris_format.unwrap_or_else(|| "oem".to_string()),
    )
    .fetch_one(&state.db)
    .await?;

    // 5. For server execution, enqueue job
    if execution_mode == "server" {
        let queue_entry = sqlx::query_as!(
            crate::db::models::PropagationQueue,
            r#"INSERT INTO propagation_queue (propagation_job_id, priority)
            VALUES ($1, $2) RETURNING *"#,
            job.id,
            if user.plan == "advanced" { 10 } else { 0 },
        )
        .fetch_one(&state.db)
        .await?;

        // Publish to Redis job queue
        state.redis
            .publish("propagation:jobs", serde_json::to_string(&job.id)?)
            .await?;
    }

    Ok((StatusCode::CREATED, Json(job)))
}

pub async fn get_propagation(
    State(state): State<AppState>,
    claims: Claims,
    Path((spacecraft_id, job_id)): Path<(Uuid, Uuid)>,
) -> Result<Json<PropagationJob>, ApiError> {
    let job = sqlx::query_as!(
        PropagationJob,
        "SELECT pj.* FROM propagation_jobs pj
         JOIN spacecraft sc ON pj.spacecraft_id = sc.id
         JOIN missions m ON sc.mission_id = m.id
         WHERE pj.id = $1 AND pj.spacecraft_id = $2
         AND (m.owner_id = $3 OR m.org_id IN (
             SELECT org_id FROM org_members WHERE user_id = $3
         ))",
        job_id,
        spacecraft_id,
        claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Propagation job not found"))?;

    Ok(Json(job))
}

pub async fn cancel_propagation(
    State(state): State<AppState>,
    claims: Claims,
    Path((spacecraft_id, job_id)): Path<(Uuid, Uuid)>,
) -> Result<impl IntoResponse, ApiError> {
    // Update job status to cancelled
    let updated = sqlx::query!(
        "UPDATE propagation_jobs pj
         SET status = 'cancelled'
         FROM spacecraft sc
         JOIN missions m ON sc.mission_id = m.id
         WHERE pj.id = $1 AND pj.spacecraft_id = $2
         AND (m.owner_id = $3 OR m.org_id IN (
             SELECT org_id FROM org_members WHERE user_id = $3
         ))
         AND pj.status IN ('pending', 'running')",
        job_id,
        spacecraft_id,
        claims.user_id
    )
    .execute(&state.db)
    .await?;

    if updated.rows_affected() == 0 {
        return Err(ApiError::NotFound("Propagation job not found or already completed"));
    }

    Ok(StatusCode::NO_CONTENT)
}
```

### 2. Orbital Propagator Core (Rust — shared between WASM and native)

The core propagator that implements numerical integration of the equations of motion with perturbations. This code compiles to both native (server) and WASM (browser) targets.

```rust
// propagator-core/src/propagator.rs

use nalgebra::{Vector3, Vector6};
use chrono::{DateTime, Utc, Duration};
use crate::{
    state::OrbitalState,
    perturbations::{Perturbations, j2_acceleration, drag_acceleration, srp_acceleration},
    integrator::{RK78Integrator, Integrator},
    constants::*,
};

pub struct Propagator {
    pub initial_state: OrbitalState,
    pub perturbations: Perturbations,
    pub integrator: Box<dyn Integrator>,
    pub timestep: f64,  // seconds
    pub tolerance: f64,
}

impl Propagator {
    pub fn new(
        initial_state: OrbitalState,
        perturbations: Perturbations,
        integrator_type: &str,
        timestep: f64,
    ) -> Self {
        let integrator: Box<dyn Integrator> = match integrator_type {
            "rk78" => Box::new(RK78Integrator::new(1e-10)),  // Dormand-Prince RK7(8)
            "abm" => Box::new(crate::integrator::ABMIntegrator::new()),  // Adams-Bashforth-Moulton
            _ => Box::new(RK78Integrator::new(1e-10)),
        };

        Self {
            initial_state,
            perturbations,
            integrator,
            timestep,
            tolerance: 1e-10,
        }
    }

    /// Propagate from start_epoch to end_epoch, returning ephemeris points
    pub fn propagate(
        &mut self,
        start_epoch: DateTime<Utc>,
        end_epoch: DateTime<Utc>,
    ) -> Result<Vec<OrbitalState>, PropagatorError> {
        let mut ephemeris = Vec::new();
        let mut current = self.initial_state.clone();
        current.epoch = start_epoch;

        ephemeris.push(current.clone());

        let total_duration = (end_epoch - start_epoch).num_seconds() as f64;
        let mut elapsed = 0.0;

        while elapsed < total_duration {
            let dt = self.timestep.min(total_duration - elapsed);

            // Integrate one step
            let next_state_vector = self.integrator.step(
                &current.to_state_vector(),
                dt,
                |t, y| self.equations_of_motion(t, y),
            )?;

            current = OrbitalState::from_state_vector(
                &next_state_vector,
                current.epoch + Duration::seconds(dt as i64),
            );

            elapsed += dt;
            ephemeris.push(current.clone());
        }

        Ok(ephemeris)
    }

    /// Equations of motion: d/dt [r, v] = [v, a]
    /// where a = -μ/r³ r + perturbations
    fn equations_of_motion(&self, t: f64, y: &Vector6<f64>) -> Vector6<f64> {
        let r = Vector3::new(y[0], y[1], y[2]);
        let v = Vector3::new(y[3], y[4], y[5]);

        let r_mag = r.norm();

        // Two-body acceleration
        let a_2body = -(MU_EARTH / r_mag.powi(3)) * r;

        // J2 perturbation
        let a_j2 = if self.perturbations.j2 {
            j2_acceleration(&r)
        } else {
            Vector3::zeros()
        };

        // Atmospheric drag
        let a_drag = if self.perturbations.drag {
            drag_acceleration(&r, &v, &self.perturbations)
        } else {
            Vector3::zeros()
        };

        // Solar radiation pressure
        let a_srp = if self.perturbations.srp {
            srp_acceleration(&r, t, &self.perturbations)
        } else {
            Vector3::zeros()
        };

        // Total acceleration
        let a_total = a_2body + a_j2 + a_drag + a_srp;

        // Return [v, a]
        Vector6::new(v[0], v[1], v[2], a_total[0], a_total[1], a_total[2])
    }
}

#[derive(Debug, Clone)]
pub struct OrbitalState {
    pub epoch: DateTime<Utc>,
    pub position: Vector3<f64>,  // km in J2000 frame
    pub velocity: Vector3<f64>,  // km/s in J2000 frame
}

impl OrbitalState {
    pub fn from_keplerian(
        epoch: DateTime<Utc>,
        a: f64,      // semi-major axis (km)
        e: f64,      // eccentricity
        i: f64,      // inclination (deg)
        raan: f64,   // RAAN (deg)
        aop: f64,    // argument of periapsis (deg)
        ta: f64,     // true anomaly (deg)
    ) -> Self {
        // Convert Keplerian elements to Cartesian state
        let (r, v) = crate::elements::keplerian_to_cartesian(a, e, i, raan, aop, ta);
        Self {
            epoch,
            position: r,
            velocity: v,
        }
    }

    pub fn to_state_vector(&self) -> Vector6<f64> {
        Vector6::new(
            self.position[0],
            self.position[1],
            self.position[2],
            self.velocity[0],
            self.velocity[1],
            self.velocity[2],
        )
    }

    pub fn from_state_vector(vec: &Vector6<f64>, epoch: DateTime<Utc>) -> Self {
        Self {
            epoch,
            position: Vector3::new(vec[0], vec[1], vec[2]),
            velocity: Vector3::new(vec[3], vec[4], vec[5]),
        }
    }

    pub fn to_keplerian(&self) -> (f64, f64, f64, f64, f64, f64) {
        crate::elements::cartesian_to_keplerian(&self.position, &self.velocity)
    }
}

#[derive(Debug)]
pub enum PropagatorError {
    Convergence(String),
    InvalidState(String),
}
```

### 3. Coverage Analysis Engine (Rust)

Constellation coverage analysis using grid-based discretization with parallel processing for large grids.

```rust
// src/coverage/mod.rs

use chrono::{DateTime, Utc, Duration};
use nalgebra::Vector3;
use rayon::prelude::*;
use crate::{
    propagator::OrbitalState,
    geometry::{elevation_angle, line_of_sight_clear},
};

#[derive(Debug, Clone)]
pub struct GridCell {
    pub latitude_deg: f64,
    pub longitude_deg: f64,
    pub accesses: Vec<AccessInterval>,
    pub revisit_time_s: Option<f64>,
    pub max_gap_s: Option<f64>,
    pub total_coverage_s: f64,
}

#[derive(Debug, Clone)]
pub struct AccessInterval {
    pub aos: DateTime<Utc>,  // Acquisition of Signal
    pub los: DateTime<Utc>,  // Loss of Signal
    pub max_elevation_deg: f64,
}

#[derive(Debug)]
pub struct CoverageGrid {
    pub cells: Vec<GridCell>,
    pub resolution_deg: f64,
    pub lat_min: f64,
    pub lat_max: f64,
    pub lon_min: f64,
    pub lon_max: f64,
}

impl CoverageGrid {
    pub fn new_global(resolution_deg: f64) -> Self {
        Self::new(-90.0, 90.0, -180.0, 180.0, resolution_deg)
    }

    pub fn new(
        lat_min: f64,
        lat_max: f64,
        lon_min: f64,
        lon_max: f64,
        resolution_deg: f64,
    ) -> Self {
        let mut cells = Vec::new();

        let mut lat = lat_min;
        while lat <= lat_max {
            let mut lon = lon_min;
            while lon <= lon_max {
                cells.push(GridCell {
                    latitude_deg: lat,
                    longitude_deg: lon,
                    accesses: Vec::new(),
                    revisit_time_s: None,
                    max_gap_s: None,
                    total_coverage_s: 0.0,
                });
                lon += resolution_deg;
            }
            lat += resolution_deg;
        }

        Self {
            cells,
            resolution_deg,
            lat_min,
            lat_max,
            lon_min,
            lon_max,
        }
    }

    /// Compute coverage from multiple spacecraft ephemerides
    pub fn compute_coverage(
        &mut self,
        ephemerides: &[Vec<OrbitalState>],
        min_elevation_deg: f64,
        start_epoch: DateTime<Utc>,
        end_epoch: DateTime<Utc>,
        timestep_s: i64,
    ) {
        // Parallel processing of grid cells
        self.cells.par_iter_mut().for_each(|cell| {
            let target_ecef = lat_lon_to_ecef(cell.latitude_deg, cell.longitude_deg, 0.0);

            // For each timestep, check visibility from any spacecraft
            let mut current_epoch = start_epoch;
            let mut in_access = false;
            let mut current_access_aos: Option<DateTime<Utc>> = None;
            let mut current_max_elev = 0.0;

            while current_epoch <= end_epoch {
                let time_index = ((current_epoch - start_epoch).num_seconds() / timestep_s) as usize;

                let mut visible = false;
                let mut max_elev_this_step = 0.0;

                // Check visibility from any spacecraft
                for eph in ephemerides {
                    if time_index >= eph.len() {
                        continue;
                    }

                    let sat_state = &eph[time_index];
                    let sat_ecef = j2000_to_ecef(&sat_state.position, current_epoch);

                    let elevation = elevation_angle(&target_ecef, &sat_ecef);
                    if elevation >= min_elevation_deg && line_of_sight_clear(&target_ecef, &sat_ecef) {
                        visible = true;
                        max_elev_this_step = max_elev_this_step.max(elevation);
                    }
                }

                if visible && !in_access {
                    // Start of new access
                    in_access = true;
                    current_access_aos = Some(current_epoch);
                    current_max_elev = max_elev_this_step;
                } else if visible && in_access {
                    // Continuing access
                    current_max_elev = current_max_elev.max(max_elev_this_step);
                } else if !visible && in_access {
                    // End of access
                    in_access = false;
                    if let Some(aos) = current_access_aos {
                        cell.accesses.push(AccessInterval {
                            aos,
                            los: current_epoch,
                            max_elevation_deg: current_max_elev,
                        });
                    }
                    current_access_aos = None;
                    current_max_elev = 0.0;
                }

                current_epoch = current_epoch + Duration::seconds(timestep_s);
            }

            // Close final access if still in one
            if in_access {
                if let Some(aos) = current_access_aos {
                    cell.accesses.push(AccessInterval {
                        aos,
                        los: end_epoch,
                        max_elevation_deg: current_max_elev,
                    });
                }
            }

            // Compute statistics
            cell.compute_statistics();
        });
    }
}

impl GridCell {
    fn compute_statistics(&mut self) {
        if self.accesses.is_empty() {
            return;
        }

        // Total coverage time
        self.total_coverage_s = self.accesses.iter()
            .map(|a| (a.los - a.aos).num_seconds() as f64)
            .sum();

        // Revisit time (average gap between accesses)
        if self.accesses.len() > 1 {
            let mut gaps: Vec<f64> = Vec::new();
            for i in 0..self.accesses.len() - 1 {
                let gap = (self.accesses[i + 1].aos - self.accesses[i].los).num_seconds() as f64;
                gaps.push(gap);
            }
            self.revisit_time_s = Some(gaps.iter().sum::<f64>() / gaps.len() as f64);
            self.max_gap_s = gaps.iter().cloned().fold(f64::NEG_INFINITY, f64::max).into();
        }
    }
}

fn lat_lon_to_ecef(lat_deg: f64, lon_deg: f64, alt_m: f64) -> Vector3<f64> {
    use crate::constants::{R_EARTH_KM, EARTH_FLATTENING};

    let lat_rad = lat_deg.to_radians();
    let lon_rad = lon_deg.to_radians();
    let alt_km = alt_m / 1000.0;

    let e2 = 2.0 * EARTH_FLATTENING - EARTH_FLATTENING.powi(2);
    let n = R_EARTH_KM / (1.0 - e2 * lat_rad.sin().powi(2)).sqrt();

    let x = (n + alt_km) * lat_rad.cos() * lon_rad.cos();
    let y = (n + alt_km) * lat_rad.cos() * lon_rad.sin();
    let z = (n * (1.0 - e2) + alt_km) * lat_rad.sin();

    Vector3::new(x, y, z)
}

fn j2000_to_ecef(pos_j2000: &Vector3<f64>, epoch: DateTime<Utc>) -> Vector3<f64> {
    // Simplified rotation from J2000 to ECEF (ignoring precession/nutation for speed)
    // In production, use full IERS transformation
    use crate::time::greenwich_sidereal_time;

    let gst_rad = greenwich_sidereal_time(epoch);
    let cos_gst = gst_rad.cos();
    let sin_gst = gst_rad.sin();

    Vector3::new(
        cos_gst * pos_j2000[0] + sin_gst * pos_j2000[1],
        -sin_gst * pos_j2000[0] + cos_gst * pos_j2000[1],
        pos_j2000[2],
    )
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–5)

**Day 1: Project initialization and Rust backend scaffold**
```bash
cargo init geosat-api
cd geosat-api
cargo add axum tokio serde serde_json sqlx uuid chrono tower-http jsonwebtoken bcrypt
cargo add --features postgres,runtime-tokio-rustls,uuid,chrono,json sqlx
cargo add aws-sdk-s3 redis nalgebra
```
- `src/main.rs` — Axum app with CORS, tracing, graceful shutdown
- `src/config.rs` — Environment-based configuration (DATABASE_URL, JWT_SECRET, S3_BUCKET, REDIS_URL)
- `src/state.rs` — AppState struct (PgPool, Redis, S3Client)
- `src/error.rs` — ApiError enum with IntoResponse
- `Dockerfile` — Multi-stage build (builder + runtime)
- `docker-compose.yml` — PostgreSQL with PostGIS, Redis, MinIO (S3-compatible)

**Day 2: Database schema and migrations**
- `migrations/001_initial.sql` — All 12 tables: users, organizations, org_members, missions, spacecraft, constellations, constellation_members, ground_stations, propagation_jobs, propagation_queue, coverage_jobs, contact_windows, maneuvers, tle_catalog, conjunction_events, usage_records
- PostGIS extension for geospatial queries on ground stations
- `src/db/mod.rs` — Database pool initialization
- `src/db/models.rs` — All SQLx structs with FromRow derives
- Run `sqlx migrate run` to apply schema
- Seed script for test missions and ground stations

**Day 3: Authentication system**
- `src/auth/mod.rs` — JWT token generation and validation middleware
- `src/auth/oauth.rs` — Google and GitHub OAuth 2.0 flow handlers
- `src/api/handlers/auth.rs` — Register, login, OAuth callback, refresh token, me endpoints
- Password hashing with bcrypt (cost factor 12)
- JWT with 24h access token + 30d refresh token
- Auth middleware that extracts Claims from Authorization header

**Day 4: Mission and spacecraft CRUD**
- `src/api/handlers/missions.rs` — Create, list, get, update, delete, fork mission
- `src/api/handlers/spacecraft.rs` — Create spacecraft, update state (Keplerian/Cartesian/TLE), delete
- `src/api/handlers/orgs.rs` — Create org, invite member, list members, remove member
- `src/api/router.rs` — All route definitions with auth middleware
- Integration tests: auth flow, mission CRUD, spacecraft state validation

**Day 5: Ground station and constellation CRUD**
- `src/api/handlers/ground_stations.rs` — Create, list, update, delete ground stations
- PostGIS geography queries for nearest ground stations within radius
- `src/api/handlers/constellations.rs` — Create constellation (Walker/custom), generate members
- Walker constellation generation logic (planes, phasing, RAAN distribution)
- Tests: Walker-Delta 24/3/1 constellation generation, verify phasing

### Phase 2 — Orbit Propagator Core (Days 6–14)

**Day 6: Coordinate frame transformations and time systems**
- `propagator-core/src/frames/mod.rs` — Coordinate frame definitions (J2000, ECEF, TEME)
- `propagator-core/src/frames/transforms.rs` — J2000 ↔ ECEF rotation matrices
- `propagator-core/src/time.rs` — UTC, UT1, TT, TAI conversions, Julian Date, Greenwich Sidereal Time
- `propagator-core/src/constants.rs` — Physical constants (μ, R_e, J2, J3, J4, J6, speed of light)
- Tests: verify frame transformations against SOFA library reference values

**Day 7: Keplerian element conversions**
- `propagator-core/src/elements/keplerian.rs` — Keplerian to Cartesian conversion (classical orbital elements)
- `propagator-core/src/elements/cartesian.rs` — Cartesian to Keplerian conversion
- Mean anomaly ↔ Eccentric anomaly ↔ True anomaly conversions (Newton-Raphson for eccentric anomaly)
- Tests: round-trip conversions for circular, elliptical, and highly eccentric orbits

**Day 8: Two-body propagator and anomaly propagation**
- `propagator-core/src/twobody.rs` — Analytical two-body propagation via mean motion
- Kepler's equation solver for eccentric anomaly given mean anomaly and eccentricity
- Tests: verify 10-orbit propagation matches analytical solution within 1 meter

**Day 9: Numerical integrators — RK7(8) and ABM**
- `propagator-core/src/integrator/mod.rs` — Integrator trait definition
- `propagator-core/src/integrator/rk78.rs` — Dormand-Prince RK7(8) with adaptive timestep
- `propagator-core/src/integrator/abm.rs` — Adams-Bashforth-Moulton multi-step integrator
- Embedded error estimation and timestep control (tolerance-based)
- Tests: integrate simple harmonic oscillator, verify error scaling

**Day 10: J2 perturbation model**
- `propagator-core/src/perturbations/j2.rs` — J2 oblateness acceleration computation
- Analytical expressions for J2 perturbation in Cartesian coordinates
- Tests: propagate sun-synchronous orbit, verify RAAN precession rate matches theory (0.9856°/day for 550km 97.4° inclination)

**Day 11: Higher-order geopotential (J3, J4, J6)**
- `propagator-core/src/perturbations/geopotential.rs` — Full geopotential model (EGM96 coefficients)
- Zonal harmonic accelerations J3, J4, J6
- Tests: GEO orbit with J2-J6, verify long-term stability vs. analytical predictions

**Day 12: Atmospheric drag model (NRLMSISE-00)**
- `propagator-core/src/perturbations/drag.rs` — Atmospheric drag acceleration
- NRLMSISE-00 density model implementation (exponential atmosphere approximation for MVP)
- Velocity relative to rotating atmosphere (ECEF rotation)
- Tests: LEO decay rate for 400km orbit with Cd=2.2, A/m=0.01 m²/kg

**Day 13: Solar radiation pressure and shadow modeling**
- `propagator-core/src/perturbations/srp.rs` — SRP acceleration
- Sun ephemeris (analytical approximation for MVP, or load SPICE kernel)
- Conical Earth shadow model (umbra, penumbra, sunlight)
- Tests: GEO satellite SRP drift, verify eccentricity variation

**Day 14: High-fidelity propagator integration**
- `propagator-core/src/propagator.rs` — Full high-fidelity propagator combining all perturbations
- Configurable perturbation toggles (J2, J3, J4, drag, SRP, etc.)
- Ephemeris generation with adaptive timestep
- Tests: 7-day LEO propagation with full perturbations, compare to STK/GMAT reference

### Phase 3 — SGP4 + WASM Build (Days 15–19)

**Day 15: SGP4/SDP4 implementation**
- `propagator-core/src/sgp4/mod.rs` — SGP4/SDP4 analytical propagator for TLE-based orbits
- TLE parser (two-line element format with checksum validation)
- SGP4 near-Earth model, SDP4 deep-space model (switch based on orbit period)
- Tests: propagate ISS TLE, compare to Space-Track ephemeris

**Day 16: TLE orbit determination from state vector**
- `propagator-core/src/sgp4/fit.rs` — Least-squares TLE fitting from Cartesian state
- Batch least-squares to fit classical elements + B* drag term
- Tests: generate synthetic ephemeris → fit TLE → propagate → verify <1km error

**Day 17: WASM solver compilation**
- `propagator-wasm/` — New workspace member for WASM target
- `propagator-wasm/Cargo.toml` — wasm-bindgen, serde-wasm-bindgen dependencies
- `propagator-wasm/src/lib.rs` — WASM entry points: `propagate_high_fidelity()`, `propagate_sgp4()`, `compute_coverage()`
- `wasm-pack build --target web --release` pipeline
- `wasm-opt -Oz` for size optimization (target <2MB gzipped)
- JavaScript wrapper: `Propagator` class that loads WASM and provides async API

**Day 18: WASM integration testing**
- Frontend integration: load WASM in headless browser (Playwright)
- Run test propagations: 24-hour LEO, 24-hour GEO
- Measure performance: 24-hour 550km LEO orbit (~15 revolutions) should complete in <3 seconds
- Verify results match native Rust propagator within numerical tolerance

**Day 19: Ephemeris export formats**
- `propagator-core/src/export/oem.rs` — CCSDS OEM (Orbit Ephemeris Message) writer
- `propagator-core/src/export/binary.rs` — Binary ephemeris format (timestamp + state vector)
- `propagator-core/src/export/csv.rs` — CSV export for spreadsheet analysis
- Tests: generate ephemeris → export to OEM → validate against CCSDS schema

### Phase 4 — Frontend 3D Visualization (Days 20–26)

**Day 20: Frontend scaffold and CesiumJS setup**
```bash
npm create vite@latest frontend -- --template react-ts
cd frontend
npm i zustand @tanstack/react-query axios cesium
npm i -D vite-plugin-cesium
```
- `src/App.tsx` — Router, auth context, layout
- `src/stores/missionStore.ts` — Zustand store for mission state
- `src/stores/spacecraftStore.ts` — Spacecraft state management
- `vite.config.ts` — Cesium plugin configuration
- Copy Cesium static assets to public folder

**Day 21: CesiumJS globe and orbit rendering**
- `src/components/Globe/GlobeViewer.tsx` — CesiumJS Viewer initialization
- `src/components/Globe/OrbitPolyline.tsx` — Render orbital ground track as polyline
- `src/components/Globe/SatelliteEntity.tsx` — Render satellite as point or 3D model with time-dynamic position
- `src/lib/cesium/ephemeris.ts` — Convert ephemeris to Cesium SampledPositionProperty
- Time slider for scrubbing through ephemeris
- Play/pause animation controls

**Day 22: Coverage footprint visualization**
- `src/components/Globe/CoverageFootprint.tsx` — Render sensor FOV footprint as polygon on globe
- Compute ground swath from satellite altitude and sensor half-angle
- Time-dynamic footprint that updates as satellite moves
- Color-coded footprint based on elevation angle or data rate

**Day 23: Ground station visibility cones**
- `src/components/Globe/GroundStation.tsx` — Render ground station as billboard with label
- `src/components/Globe/VisibilityCone.tsx` — Render line-of-sight cone from ground station to satellite during access
- Highlight ground station when satellite is in view (green = in access, gray = no access)
- Display real-time range, azimuth, elevation in tooltip

**Day 24: Orbit element plots (D3.js)**
- `src/components/Plots/OrbitElementPlot.tsx` — D3.js line chart for orbit element evolution
- Plot semi-major axis, eccentricity, inclination, RAAN, AoP vs. time
- Detect secular drift (e.g., RAAN precession due to J2)
- Zoom and pan on plots

**Day 25: Eclipse and thermal environment plots**
- `src/components/Plots/EclipsePlot.tsx` — Sunlight/eclipse timeline (binary or fractional penumbra)
- `src/components/Plots/ThermalPlot.tsx` — Solar flux, Earth IR, albedo vs. time
- Battery depth-of-discharge estimation for power subsystem sizing
- Export eclipse data as CSV

**Day 26: Orbit designer UI**
- `src/pages/OrbitDesigner.tsx` — Form for Keplerian element input (a, e, i, RAAN, AoP, TA)
- Orbit type wizards: sun-synchronous (compute required inclination for given altitude), repeat ground track (compute a for N revs in M days), frozen orbit
- Real-time validation: periapsis above atmosphere, orbit stability checks
- "Propagate" button triggers WASM or server propagation
- Display computed orbit on globe immediately after input

### Phase 5 — Propagation API + Job Orchestration (Days 27–31)

**Day 27: Propagation API endpoints**
- `src/api/handlers/propagation.rs` — Create propagation job, get job status, list jobs, cancel job
- State validation: verify spacecraft has valid initial state (Keplerian or Cartesian or TLE)
- Duration calculation and WASM/server routing logic
- Plan-based limits enforcement (free: 24h max, SGP4 only; pro: unlimited duration, high-fidelity; advanced: all features)

**Day 28: Server-side propagation worker**
- `src/workers/propagation_worker.rs` — Redis job consumer, runs native propagator
- `src/workers/mod.rs` — Worker pool management (configurable concurrency, e.g., 4 workers)
- Progress streaming: worker publishes progress → Redis pub/sub → WebSocket to client
- Error handling: integration failures, out-of-memory, invalid state
- S3 ephemeris upload with presigned URL generation for client download

**Day 29: WebSocket for live propagation progress**
- `src/api/ws/mod.rs` — Axum WebSocket upgrade handler
- `src/api/ws/propagation_progress.rs` — Subscribe to propagation progress channel by job ID
- Client receives: `{ progress_pct, current_epoch, timestep, error_estimate }`
- Connection management: heartbeat ping/pong, automatic cleanup on disconnect
- Frontend: `src/hooks/usePropagationProgress.ts` — React hook for WebSocket subscription

**Day 30: Ephemeris caching and retrieval**
- Redis cache for hot ephemeris (recently propagated, TTL 1 hour)
- S3 presigned URL generation for ephemeris download
- `src/api/handlers/ephemeris.rs` — Get ephemeris (from cache or S3), download ephemeris file
- Frontend: download button for OEM/CSV/binary formats

**Day 31: Ground station contact window computation**
- `src/api/handlers/contacts.rs` — Compute contact windows for spacecraft + ground station pair
- Access interval algorithm: iterate ephemeris, compute elevation angle, detect AOS/LOS crossings
- Store contact windows in database for reuse
- Frontend: contact windows table with AOS, LOS, max elevation, duration

### Phase 6 — Coverage Analysis + Constellation Design (Days 32–36)

**Day 32: Coverage grid computation**
- `src/coverage/mod.rs` — Grid-based coverage analysis (as shown in deep-dive)
- API endpoint: `POST /api/coverage-jobs` — create coverage job
- Server worker: `src/workers/coverage_worker.rs` — process coverage jobs in background
- Parallel grid cell processing with Rayon
- Store coverage grid results in S3 (JSON or binary format)

**Day 33: Coverage visualization on globe**
- `src/components/Globe/CoverageHeatmap.tsx` — Render coverage grid as colored rectangles on globe
- Color scale: green (short revisit) → yellow → red (long revisit/gaps)
- Tooltip on hover: display cell statistics (revisit time, gap, # accesses)
- Toggle layers: revisit time, gap duration, total coverage time

**Day 34: Walker constellation generator**
- `src/constellation/walker.rs` — Walker-Delta and Walker-Star constellation generation
- Walker notation: T/P/F (total sats / planes / phasing parameter)
- Compute RAAN for each plane, true anomaly for each satellite in plane
- API endpoint: `POST /api/constellations/generate` — generate constellation from parameters
- Frontend: Walker constellation form with real-time satellite count calculation

**Day 35: Constellation deployment simulation**
- Multi-spacecraft propagation: propagate all constellation members in parallel
- Deployment phasing: simulate sequential deployment with delay between satellites
- Visualize constellation build-up over time on globe
- Animated playback of constellation deployment

**Day 36: Coverage performance optimization**
- Spatial indexing with R-tree (rstar crate) for fast satellite→target queries
- Adaptive grid resolution: use coarse grid (5°) for global, fine grid (0.5°) for target regions
- Incremental coverage updates: recompute only affected grid cells when ephemeris changes
- Benchmark: global 1° grid coverage for 24-sat Walker constellation over 24 hours should complete in <60 seconds

### Phase 7 — Link Budget + Maneuver Planning (Days 37–42)

**Day 37: RF link budget calculator**
- `src/linkbudget/mod.rs` — RF link equation implementation
- Free-space path loss: FSPL = 20·log₁₀(4π·d/λ) dB
- Atmospheric losses: ITU-R P.676 gaseous absorption (oxygen, water vapor)
- Link margin = EIRP - FSPL - atmospheric losses + G/T - required C/N0
- API endpoint: `POST /api/link-budget` — compute link budget for contact window
- Frontend: link budget form (frequency, transmit power, antenna gain, G/T)

**Day 38: Dynamic link budget over contact**
- Compute link budget at each timestep during contact window
- Plot link margin vs. time, range vs. time, Doppler shift vs. time
- Identify link outages (margin < 0 dB)
- Data volume calculation: integrate data rate over contact duration

**Day 39: Hohmann transfer calculator**
- `src/maneuvers/hohmann.rs` — Hohmann transfer computation
- Given initial orbit (r1) and target orbit (r2), compute delta-V for circularization
- ΔV₁ = √(μ/r₁) · (√(2r₂/(r₁+r₂)) - 1)
- ΔV₂ = √(μ/r₂) · (1 - √(2r₁/(r₁+r₂)))
- API endpoint: `POST /api/maneuvers/hohmann` — compute Hohmann transfer parameters
- Frontend: maneuver calculator form with orbit inputs

**Day 40: Plane change and combined maneuvers**
- `src/maneuvers/plane_change.rs` — Pure plane change delta-V = 2·V·sin(Δi/2)
- `src/maneuvers/combined.rs` — Combined orbit raise + plane change (optimize burn at intersection)
- API endpoint: support plane change and combined maneuver types
- Frontend: maneuver type selector, real-time delta-V calculation

**Day 41: Delta-V budget tracker**
- `src/api/handlers/maneuvers.rs` — Create maneuver, list maneuvers for spacecraft
- Spacecraft propulsion model: Tsiolkovsky equation Δm = m₀(1 - e^(-ΔV/(Isp·g₀)))
- Track fuel consumed per maneuver, remaining fuel mass
- Frontend: maneuver timeline, fuel gauge, delta-V budget bar chart

**Day 42: Maneuver execution simulation**
- Apply maneuver delta-V to spacecraft state at specified epoch
- Impulsive burn: instantaneous velocity change
- Re-propagate orbit after maneuver
- Visualize pre-maneuver and post-maneuver orbits on globe
- Frontend: "Execute Maneuver" button, orbit comparison view

### Phase 8 — Billing + Polish + Testing (Days 43–48)

**Day 43: Stripe integration**
- `src/api/handlers/billing.rs` — Create checkout session, create customer portal session, get subscription status
- `src/api/handlers/webhooks/stripe.rs` — Handle subscription events: `checkout.session.completed`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.payment_failed`
- Plan mapping:
  - Free: 24-hour propagation, SGP4 only, 3 missions
  - Pro ($149/mo): Unlimited propagation, high-fidelity, 25 missions, coverage analysis, link budget
  - Advanced ($349/user/mo): Multi-user orgs, COLA, API access, unlimited missions

**Day 44: Usage tracking and plan limits**
- `src/middleware/plan_limits.rs` — Middleware checking plan limits before job creation
- `src/services/usage.rs` — Track propagation hours per billing period
- Usage record insertion after each server-side propagation completes
- Usage dashboard: `src/api/handlers/usage.rs` — Current period usage, historical usage
- Approaching-limit warnings at 80% and 100% of propagation hours

**Day 45: Billing UI**
- `frontend/src/pages/Billing.tsx` — Plan comparison table, current plan display, usage meter
- `frontend/src/components/billing/PlanCard.tsx` — Plan feature comparison cards
- `frontend/src/components/billing/UsageMeter.tsx` — Propagation hours usage bar with threshold indicators
- Upgrade/downgrade flow via Stripe Customer Portal
- Upgrade prompt modals when hitting plan limits

**Day 46: Integration testing and end-to-end tests**
- End-to-end test: create mission → add spacecraft → propagate orbit → view on globe → compute coverage
- API integration tests: auth → mission CRUD → propagation → coverage → maneuvers
- WASM solver test: load in headless browser, run propagation, verify results
- WebSocket test: connect → subscribe to progress → receive updates
- Concurrent job test: submit 10 simultaneous propagation jobs

**Day 47: Performance testing and validation**
- WASM propagator benchmarks: 24-hour LEO propagation should complete in <3 seconds
- Server propagator benchmarks: 30-day LEO propagation should complete in <15 seconds
- Coverage analysis benchmark: global 1° grid for 24-sat Walker constellation over 24 hours in <60 seconds
- Memory profiling: ensure ephemeris storage doesn't leak
- Load testing: 100 concurrent users submitting propagation jobs

**Day 48: Final polish and deployment**
- Frontend loading states and error handling for all API calls
- Empty states for missions, spacecraft, coverage jobs
- Onboarding flow: sample mission with pre-populated spacecraft
- Landing page with hero section, features, pricing, screenshots
- Documentation: API reference (OpenAPI spec), user guide (orbit designer, propagation, coverage)
- Deploy to production: backend to AWS ECS, frontend to CloudFront, database to RDS

---

## Validation Benchmarks

### Benchmark 1: Two-Body Circular Orbit Precision
**Test:** Propagate a circular LEO orbit (550km altitude, 0° inclination) for 10 revolutions using two-body dynamics only.
**Expected:** Position error < 1 meter after 10 orbits compared to analytical solution.
**Validation:** The propagator must maintain energy conservation to within 0.01% over 10 orbits.

### Benchmark 2: J2 Secular RAAN Precession
**Test:** Propagate a sun-synchronous orbit (550km, 97.4° inclination) for 30 days with J2 perturbation enabled.
**Expected:** RAAN precession rate = 0.9856°/day ± 0.001°/day (matches Earth's mean motion around Sun).
**Validation:** Measure RAAN change over 30 days, verify matches theoretical J2 precession formula to within 0.1%.

### Benchmark 3: Atmospheric Drag Orbit Decay
**Test:** Propagate a 400km LEO orbit with atmospheric drag (Cd=2.2, A/m=0.01 m²/kg) for 7 days.
**Expected:** Semi-major axis decreases by approximately 2-3 km over 7 days.
**Validation:** Compare against STK or GMAT reference propagation with same drag parameters, match within 5%.

### Benchmark 4: Walker Constellation Coverage
**Test:** Generate a Walker-Delta 24/3/1 constellation at 550km with 53° inclination, compute global 2° grid coverage over 24 hours with 5° minimum elevation.
**Expected:** Average revisit time < 600 seconds for latitudes between ±60°, >95% global coverage.
**Validation:** Coverage statistics match theoretical Walker constellation performance from published literature.

### Benchmark 5: Hohmann Transfer Delta-V
**Test:** Compute Hohmann transfer from LEO (7000 km orbital radius) to GEO (42164 km orbital radius).
**Expected:** Total delta-V = 3.935 km/s ± 0.01 km/s (matches analytical Hohmann formula).
**Validation:** Computed delta-V matches hand calculation and aerospace textbook values to within 0.3%.

---

## Post-MVP Roadmap

### Version 1.1 — COLA and Space Safety (Weeks 13-16)
- TLE catalog ingestion from Space-Track API (automated daily updates)
- Conjunction screening: detect close approaches within configurable miss distance threshold
- Probability of collision (Pc) computation using covariance realism
- CDM (Conjunction Data Message) parsing and trending
- Collision avoidance maneuver recommendation
- Dashboard for conjunction events with risk levels

### Version 1.2 — Low-Thrust Optimization (Weeks 17-20)
- Electric propulsion spiral transfer trajectory optimization (Python IPOPT)
- Edelbaum approximation for low-thrust orbit raising
- Multi-revolution low-thrust transfer optimization
- Thrust vectoring and coast arc optimization
- Low-thrust delta-V budget and timeline visualization

### Version 1.3 — Orbit Determination (Weeks 21-24)
- Batch least-squares orbit determination from ground station observations
- Sequential orbit determination (Extended Kalman Filter, Unscented Kalman Filter)
- Range, range-rate, and angle observation models
- Orbit determination residual analysis and covariance visualization
- TLE fitting from observed ephemeris

### Version 1.4 — Inter-Satellite Links (Weeks 25-28)
- ISL (inter-satellite link) topology planning for constellation networking
- Dynamic ISL link budget accounting for relative motion
- Optical ISL analysis (free-space optical communication)
- Crosslink scheduling and routing optimization
- Constellation connectivity graph visualization

### Version 2.0 — Operations Center Integration (Weeks 29-36)
- Real-time telemetry ingestion from ground stations
- Maneuver command generation and upload
- Automated station-keeping for GEO and LEO SSO
- Anomaly detection via ML (orbit prediction, thruster performance)
- Mission control dashboard with live satellite health

---

## Risk Mitigation

### Technical Risks

**Risk:** WASM propagator accuracy diverges from native propagator due to floating-point differences.
**Mitigation:** Comprehensive test suite comparing WASM and native results for identical inputs, with <0.1% position error tolerance. Use same Rust codebase compiled to both targets to eliminate algorithmic differences.

**Risk:** Coverage analysis for large constellations (100+ satellites) over long durations (months) exceeds server memory/time limits.
**Mitigation:** Implement incremental coverage updates (only recompute changed grid cells), use sparse storage for coverage grids, and provide adaptive grid resolution (coarse for global, fine for regions of interest).

**Risk:** CesiumJS bundle size (2.5MB) increases initial page load time.
**Mitigation:** Code-split Cesium and load asynchronously only when user opens 3D visualization. Provide 2D ground track map as lightweight alternative for users with slow connections.

### Market Risks

**Risk:** NewSpace companies prefer proven tools (STK, FreeFlyer) despite high cost due to risk aversion.
**Mitigation:** Target graduate students and early-stage startups first (lower switching cost), build credibility with published validation benchmarks, offer migration assistance from STK netlists.

**Risk:** Free tier cannibalizes paid conversion if too generous.
**Mitigation:** Free tier limited to 24-hour propagation and SGP4 only (covers academic use cases), but requires upgrade for high-fidelity propagation, coverage analysis, and multi-week campaigns which are critical for commercial missions.

**Risk:** Aerospace industry requires on-premise deployment for ITAR/classified missions, which delays enterprise deals.
**Mitigation:** Design architecture to support on-premise deployment from day one (Dockerized services, no hardcoded cloud dependencies), offer Enterprise tier with on-premise + support contract.

---

## Success Metrics

### Technical Metrics (MVP completion)
- WASM propagator completes 24-hour LEO propagation in <3 seconds on Chrome/Firefox
- Server propagator completes 30-day LEO propagation in <15 seconds
- Coverage analysis for 24-satellite Walker constellation (global 1° grid, 24 hours) completes in <60 seconds
- J2 RAAN precession for sun-synchronous orbit matches theoretical rate to within 0.1%
- Hohmann transfer delta-V computation matches analytical formula to within 0.3%

### Business Metrics (First 6 months)
- 500 registered users (free tier)
- 50 Pro subscribers ($149/mo) — $7,450 MRR
- 5 Advanced team subscriptions (avg 4 users) — $6,980 MRR
- Total MRR: $14,430 (ARR: $173,160)
- 10% free-to-paid conversion rate
- <5% monthly churn

### User Engagement Metrics
- Average 3 missions created per paid user
- Average 8 spacecraft per mission
- Average 4 propagation jobs per week per paid user
- Average 1 coverage analysis per mission
- 60% of users view 3D globe visualization
- 40% of users export ephemeris (OEM/CSV)

