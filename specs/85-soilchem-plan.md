# 85. SoilChem — Agricultural and Soil Science Modeling Platform

## Implementation Plan

**MVP Scope:** Browser-based soil profile editor with visual layer configuration (texture, bulk density, organic matter, hydraulic parameters via van Genuchten equations and Rosetta pedotransfer functions), custom 1D Richards equation solver for unsaturated water flow compiled to WebAssembly for client-side execution of profiles ≤20 layers and server-side Rust-native execution for larger simulations, mechanistic crop growth models for wheat, maize, and soybean using thermal time phenology and radiation use efficiency (RUE) biomass accumulation with water/nitrogen stress factors, Penman-Monteith FAO-56 evapotranspiration calculator with dual crop coefficient (Kcb + Ke), basic nitrogen cycling (mineralization via first-order kinetics, nitrification, plant uptake via Michaelis-Menten, leaching), soil moisture deficit-based irrigation scheduling with user-defined management allowed depletion (MAD) thresholds, ERA5 climate data integration via Python FastAPI service for automated weather extraction, interactive soil water balance dashboard with D3.js time-series charts (daily ET, drainage, nitrogen leaching) and SVG soil profile visualization showing water content by depth, PostgreSQL + S3 storage for field data and simulation outputs, three-tier pricing (Free / Pro $99/mo / Advanced $249/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| Vadose Zone Solver | Rust (native + WASM) | Custom 1D Richards equation FEM solver with van Genuchten retention curves |
| Crop Model Engine | Rust (native + WASM) | Thermal time phenology, RUE biomass, Penman-Monteith ET |
| Nutrient Cycling | Rust (native + WASM) | First-order N mineralization, nitrification, plant uptake, leaching |
| WASM Compilation | `wasm-pack` + `wasm-bindgen` | Client-side solver for profiles ≤20 layers, 90-day simulations |
| Weather Data Service | Python 3.12 (FastAPI) | ERA5 CDS API extraction, PRISM/gridMET integration, climate data caching |
| Database | PostgreSQL 16 + PostGIS | Fields, soil profiles, crop parameters, simulation runs, spatial data |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | Simulation result time-series (daily water/nitrogen balance), weather data cache |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| Soil Profile Editor | Custom SVG renderer | Click-to-add layers, drag handles for depths, texture triangle picker |
| Visualization | D3.js + SVG | Time-series water balance, soil moisture depth profiles, nitrogen budgets |
| Map Integration | Mapbox GL JS + PostGIS | Field boundary drawing, spatial soil variability, weather station proximity |
| Real-time | WebSocket (Axum) | Live simulation progress, convergence monitoring |
| Job Queue | Redis 7 + Tokio tasks | Server-side simulation job management, batch field processing |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | WASM bundle delivery, static frontend assets |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, error tracking |
| CI/CD | GitHub Actions | WASM build pipeline, Rust tests, Docker image push |

### Key Architecture Decisions

1. **Hybrid client/server vadose zone solver with 20-layer WASM threshold**: Soil profiles with ≤20 layers and simulation periods ≤90 days (covers most single-field water balance scenarios) run entirely in the browser via WASM, providing instant feedback with zero server cost. Larger simulations (multi-year, 50+ layer profiles, regional modeling) are submitted to the Rust-native server solver which handles high-resolution spatial discretization and long-term carbon modeling. The threshold is configurable per plan tier.

2. **Custom Richards equation solver in Rust rather than wrapping HYDRUS**: Building a custom 1D Richards equation finite element solver in Rust gives us full control over numerical methods (Picard vs Newton iteration, adaptive time stepping), WASM compilation, and integration with crop models. HYDRUS is closed-source Fortran with licensing restrictions and no WASM support. Our Rust solver uses implicit time discretization with Picard iteration and mass-lumping for stability while maintaining second-order spatial accuracy.

3. **SVG soil profile editor with texture triangle component picker**: The soil layer editor uses SVG for crisp rendering at any zoom level with drag handles for adjusting layer depths and a visual USDA texture triangle for selecting sand/silt/clay fractions. Hydraulic parameters (θs, θr, α, n, Ks) are auto-calculated via Rosetta pedotransfer functions with manual override. D3.js renders soil moisture profiles as color-coded depth bands overlaid on the soil layer structure.

4. **Separate Python FastAPI service for weather data and ML tasks**: Weather data extraction (ERA5 via CDS API, PRISM, gridMET) and future ML yield forecasting require Python libraries (cdsapi, xarray, netCDF4, scikit-learn, PyTorch). This service runs independently from the Rust API, exposing endpoints for climate data retrieval and caching results in S3. Communication via HTTP REST prevents language barriers while maintaining Rust's performance for simulation core.

5. **PostgreSQL + PostGIS for field geometries with S3 for time-series results**: Field boundaries, soil sampling locations, and weather station proximity queries leverage PostGIS spatial indexes. Simulation outputs (daily time-series with 1000s of rows per run) are stored as Parquet or compressed CSV in S3 to avoid bloating PostgreSQL while keeping metadata (run parameters, summary statistics) in the database for fast querying.

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
    plan TEXT NOT NULL DEFAULT 'free',
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    settings JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX orgs_slug_idx ON organizations(slug);

-- Fields (with PostGIS geometry)
CREATE TABLE fields (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    location GEOGRAPHY(POLYGON, 4326),
    area_hectares REAL,
    elevation_m REAL,
    soil_type TEXT,
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX fields_owner_idx ON fields(owner_id);
CREATE INDEX fields_location_idx ON fields USING GIST(location);

-- Soil Profiles
CREATE TABLE soil_profiles (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    field_id UUID REFERENCES fields(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    layers JSONB NOT NULL,
    total_depth_cm REAL NOT NULL,
    is_template BOOLEAN DEFAULT false,
    created_by UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX profiles_field_idx ON soil_profiles(field_id);

-- Crop Varieties
CREATE TABLE crop_varieties (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    crop_name TEXT NOT NULL,
    variety_name TEXT NOT NULL,
    phenology_params JSONB NOT NULL,
    growth_params JSONB NOT NULL,
    stress_params JSONB NOT NULL,
    is_builtin BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(crop_name, variety_name)
);

-- Management Scenarios
CREATE TABLE management_scenarios (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    field_id UUID NOT NULL REFERENCES fields(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    crop_variety_id UUID NOT NULL REFERENCES crop_varieties(id),
    planting_date DATE NOT NULL,
    harvest_date DATE,
    irrigation_events JSONB DEFAULT '[]',
    fertilizer_events JSONB DEFAULT '[]',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Simulation Runs
CREATE TABLE simulation_runs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    field_id UUID NOT NULL REFERENCES fields(id) ON DELETE CASCADE,
    soil_profile_id UUID NOT NULL REFERENCES soil_profiles(id) ON DELETE RESTRICT,
    management_scenario_id UUID NOT NULL REFERENCES management_scenarios(id) ON DELETE RESTRICT,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    simulation_type TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending',
    execution_mode TEXT NOT NULL DEFAULT 'wasm',
    start_date DATE NOT NULL,
    end_date DATE NOT NULL,
    parameters JSONB NOT NULL DEFAULT '{}',
    results_url TEXT,
    results_summary JSONB,
    error_message TEXT,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    wall_time_ms INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX sims_field_idx ON simulation_runs(field_id);
CREATE INDEX sims_status_idx ON simulation_runs(status);

-- Server Simulation Jobs
CREATE TABLE simulation_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    simulation_run_id UUID NOT NULL REFERENCES simulation_runs(id) ON DELETE CASCADE,
    worker_id TEXT,
    priority INTEGER DEFAULT 0,
    progress_pct REAL DEFAULT 0.0,
    convergence_data JSONB DEFAULT '[]',
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Weather Data Cache
CREATE TABLE weather_data (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    location GEOGRAPHY(POINT, 4326),
    source TEXT NOT NULL,
    start_date DATE NOT NULL,
    end_date DATE NOT NULL,
    data_url TEXT NOT NULL,
    variables TEXT[] NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(source, location, start_date, end_date)
);
CREATE INDEX weather_location_idx ON weather_data USING GIST(location);
```

### Rust SQLx Structs

```rust
// src/db/models.rs

use chrono::{DateTime, NaiveDate, Utc};
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
    pub plan: String,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Field {
    pub id: Uuid,
    pub owner_id: Uuid,
    pub name: String,
    pub area_hectares: Option<f32>,
    pub elevation_m: Option<f32>,
    pub metadata: serde_json::Value,
}

#[derive(Debug, FromRow, Serialize)]
pub struct SoilProfile {
    pub id: Uuid,
    pub field_id: Option<Uuid>,
    pub name: String,
    pub layers: serde_json::Value,
    pub total_depth_cm: f32,
    pub is_template: bool,
}

#[derive(Debug, Serialize, Deserialize, Clone)]
pub struct SoilLayer {
    pub depth_top_cm: f32,
    pub depth_bottom_cm: f32,
    pub sand_pct: f32,
    pub silt_pct: f32,
    pub clay_pct: f32,
    pub organic_matter_pct: f32,
    pub bulk_density_g_cm3: f32,
    pub theta_sat: f32,
    pub theta_res: f32,
    pub alpha_vg: f32,
    pub n_vg: f32,
    pub k_sat_cm_day: f32,
    pub initial_theta: f32,
}

#[derive(Debug, FromRow, Serialize)]
pub struct CropVariety {
    pub id: Uuid,
    pub crop_name: String,
    pub variety_name: String,
    pub phenology_params: serde_json::Value,
    pub growth_params: serde_json::Value,
    pub stress_params: serde_json::Value,
}

#[derive(Debug, FromRow, Serialize)]
pub struct SimulationRun {
    pub id: Uuid,
    pub field_id: Uuid,
    pub status: String,
    pub execution_mode: String,
    pub start_date: NaiveDate,
    pub end_date: NaiveDate,
    pub results_url: Option<String>,
    pub results_summary: Option<serde_json::Value>,
}

#[derive(Debug, Deserialize)]
pub struct SimulationParams {
    pub initial_conditions: InitialConditions,
    pub boundary_conditions: BoundaryConditions,
    pub solver_options: SolverOptions,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct InitialConditions {
    pub water_table_depth_cm: Option<f64>,
    pub initial_soil_temp_c: f64,
    pub initial_nitrate_kg_ha: Vec<f64>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct BoundaryConditions {
    pub top_bc_type: String,
    pub bottom_bc_type: String,
    pub bottom_head_cm: Option<f64>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct SolverOptions {
    pub max_picard_iterations: u32,
    pub convergence_tolerance: f64,
    pub max_timestep_days: f64,
}
```

---

## Solver Architecture Deep-Dive

### Governing Equations and Discretization

SoilChem's core vadose zone solver implements the **1D Richards equation** for unsaturated water flow:

```
∂θ/∂t = ∂/∂z[K(h)(∂h/∂z + 1)] - S(z,t)
```

Where:
- **θ** = volumetric water content (cm³/cm³)
- **h** = pressure head (cm) — negative in unsaturated zone
- **K(h)** = unsaturated hydraulic conductivity (cm/day)
- **z** = vertical coordinate (cm, positive downward)
- **S(z,t)** = root water uptake sink term (1/day)

**Water retention curve** (van Genuchten 1980):

```
θ(h) = θr + (θs - θr) / [1 + (α|h|)ⁿ]ᵐ
where m = 1 - 1/n
```

**Hydraulic conductivity** (Mualem-van Genuchten):

```
K(θ) = Ks · Sₑ^0.5 · [1 - (1 - Sₑ^(1/m))ᵐ]²
where Sₑ = (θ - θr) / (θs - θr)
```

**Spatial discretization** uses finite elements with linear basis functions. Mass-lumping applied to time derivative for stability.

**Time discretization** uses implicit backward Euler with **Picard iteration** to linearize nonlinear K(h) and θ(h):

```
C(hᵏ) · (hᵏ⁺¹ - hᵏ) / Δt = ∂/∂z[K(hᵏ)(∂hᵏ⁺¹/∂z + 1)] - Sᵏ
where C(h) = dθ/dh
```

**Root water uptake** uses Feddes stress response:

```
S(z,t) = α(h) · RDF(z) · Tₚ
where α(h) = stress reduction factor (0-1)
```

### Crop Model Architecture

**Thermal time accumulation**:

```
GDD = Σ max(0, (Tₘₐₓ + Tₘᵢₙ)/2 - Tᵦₐₛₑ)
```

**Biomass accumulation** via RUE:

```
dB/dt = RUE · fPAR · PAR · min(f_water, f_N)
where fPAR = 1 - exp(-k · LAI)
```

**Yield formation**:

```
Yield = HI · Bₜₒₜₐₗ
```

### Nitrogen Cycling Core

**Mineralization**:

```
dN_org/dt = -k_min · f(T) · f(θ) · N_org
where f(T) = Q₁₀^((T-20)/10)
```

**Nitrification** (Michaelis-Menten):

```
dNH₄/dt = -Vₘₐₓ · NH₄ / (Kₘ + NH₄)
```

**Leaching** (advection-dispersion):

```
∂(θ·C)/∂t = ∂/∂z[θ·D·∂C/∂z - q·C]
```

### Client/Server Split

```
Simulation → Layer count + duration
    │
    ├── ≤20 layers AND ≤90 days → WASM (browser, instant, free)
    └── >20 layers OR >90 days → Server (Redis queue, parallel solve)
```

---

## Architecture Deep-Dives

### 1. Simulation API Handler (Rust/Axum)

```rust
// src/api/handlers/simulation.rs

use axum::{extract::{Path, State, Json}, http::StatusCode, response::IntoResponse};
use uuid::Uuid;

pub async fn create_simulation(
    State(state): State<AppState>,
    claims: Claims,
    Path(field_id): Path<Uuid>,
    Json(req): Json<CreateSimulationRequest>,
) -> Result<impl IntoResponse, ApiError> {
    // Verify field ownership
    let field = sqlx::query_as!(Field, "SELECT * FROM fields WHERE id = $1 AND owner_id = $2",
        field_id, claims.user_id)
        .fetch_optional(&state.db).await?
        .ok_or(ApiError::NotFound("Field not found"))?;

    // Fetch soil profile and count layers
    let profile = sqlx::query_as!(SoilProfile, "SELECT * FROM soil_profiles WHERE id = $1",
        req.soil_profile_id)
        .fetch_one(&state.db).await?;

    let layers: Vec<SoilLayer> = serde_json::from_value(profile.layers)?;
    let layer_count = layers.len();
    let duration_days = (req.end_date - req.start_date).num_days();

    // Check plan limits
    let user = sqlx::query_as!(User, "SELECT * FROM users WHERE id = $1", claims.user_id)
        .fetch_one(&state.db).await?;

    if user.plan == "free" && duration_days > 90 {
        return Err(ApiError::PlanLimit("Free plan: max 90 days. Upgrade to Pro."));
    }

    // Determine execution mode
    let execution_mode = if layer_count <= 20 && duration_days <= 90 {
        "wasm"
    } else {
        "server"
    };

    // Create simulation record
    let sim = sqlx::query_as!(SimulationRun,
        r#"INSERT INTO simulation_runs (field_id, soil_profile_id, management_scenario_id,
           user_id, name, simulation_type, status, execution_mode, start_date, end_date, parameters)
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11) RETURNING *"#,
        field_id, req.soil_profile_id, req.management_scenario_id, claims.user_id,
        format!("{} {}-{}", req.simulation_type, req.start_date, req.end_date),
        req.simulation_type,
        if execution_mode == "wasm" { "pending_client" } else { "pending" },
        execution_mode, req.start_date, req.end_date, serde_json::to_value(&req.parameters)?
    ).fetch_one(&state.db).await?;

    // For server execution, enqueue job
    if execution_mode == "server" {
        let job = sqlx::query!(
            "INSERT INTO simulation_jobs (simulation_run_id, priority) VALUES ($1, $2) RETURNING id",
            sim.id, if user.plan == "advanced" { 10 } else { 0 }
        ).fetch_one(&state.db).await?;

        let mut redis = state.redis.get_async_connection().await?;
        redis::cmd("LPUSH").arg("simulation:jobs").arg(job.id.to_string())
            .query_async::<_, ()>(&mut redis).await?;
    }

    Ok((StatusCode::CREATED, Json(sim)))
}
```

### 2. Richards Equation Solver Core (Rust)

```rust
// solver-core/src/richards.rs

use nalgebra::{DMatrix, DVector};

pub struct RichardsSolver {
    pub n_nodes: usize,
    pub z_nodes: Vec<f64>,
    pub elem_lengths: Vec<f64>,
    pub soil_params: Vec<VanGenuchtenParams>,
    pub head: DVector<f64>,
    pub theta: DVector<f64>,
}

#[derive(Clone)]
pub struct VanGenuchtenParams {
    pub theta_sat: f64,
    pub theta_res: f64,
    pub alpha: f64,
    pub n: f64,
    pub k_sat: f64,
}

impl VanGenuchtenParams {
    pub fn theta(&self, h: f64) -> f64 {
        if h >= 0.0 { return self.theta_sat; }
        let m = 1.0 - 1.0 / self.n;
        let term = 1.0 + (self.alpha * h.abs()).powf(self.n);
        self.theta_res + (self.theta_sat - self.theta_res) / term.powf(m)
    }

    pub fn capacity(&self, h: f64) -> f64 {
        if h >= 0.0 { return 0.0; }
        let m = 1.0 - 1.0 / self.n;
        let abs_h = h.abs();
        let term = (self.alpha * abs_h).powf(self.n);
        let denom = (1.0 + term).powf(m + 1.0);
        self.alpha * self.n * m * (self.theta_sat - self.theta_res) * term / (abs_h * denom)
    }

    pub fn conductivity(&self, h: f64) -> f64 {
        if h >= 0.0 { return self.k_sat; }
        let se = (self.theta(h) - self.theta_res) / (self.theta_sat - self.theta_res);
        let m = 1.0 - 1.0 / self.n;
        self.k_sat * se.powf(0.5) * (1.0 - (1.0 - se.powf(1.0 / m)).powf(m)).powi(2)
    }
}

impl RichardsSolver {
    pub fn solve_timestep(
        &mut self, dt: f64, sink: &DVector<f64>,
        top_bc: BoundaryCondition, bottom_bc: BoundaryCondition,
        max_iter: u32, tolerance: f64,
    ) -> Result<u32, SolverError> {
        for iter in 0..max_iter {
            let (mut k_matrix, mut b_vector) = self.assemble_system(dt, sink);
            self.apply_bc(&mut k_matrix, &mut b_vector, top_bc, bottom_bc);

            let h_new = k_matrix.lu().solve(&b_vector)
                .ok_or(SolverError::SingularMatrix)?;

            let max_change = (&h_new - &self.head).abs().max();
            self.head = h_new;

            for i in 0..self.n_nodes {
                let elem_idx = i.min(self.soil_params.len() - 1);
                self.theta[i] = self.soil_params[elem_idx].theta(self.head[i]);
            }

            if max_change < tolerance { return Ok(iter + 1); }
        }
        Err(SolverError::Convergence(format!("Failed after {} iterations", max_iter)))
    }

    fn assemble_system(&self, dt: f64, sink: &DVector<f64>) -> (DMatrix<f64>, DVector<f64>) {
        let n = self.n_nodes;
        let mut k = DMatrix::zeros(n, n);
        let mut b = DVector::zeros(n);

        for e in 0..(n-1) {
            let dz = self.elem_lengths[e];
            let h_avg = (self.head[e] + self.head[e+1]) / 2.0;
            let k_elem = self.soil_params[e].conductivity(h_avg);
            let c_avg = self.soil_params[e].capacity(h_avg);

            let storage = c_avg / dt * dz / 2.0;
            let stiff = k_elem / dz;

            k[(e, e)] += stiff + storage;
            k[(e, e+1)] += -stiff;
            k[(e+1, e)] += -stiff;
            k[(e+1, e+1)] += stiff + storage;

            b[e] += k_elem / 2.0 - sink[e] * dz / 2.0;
            b[e+1] += k_elem / 2.0 - sink[e+1] * dz / 2.0;
        }
        (k, b)
    }

    fn apply_bc(&self, k: &mut DMatrix<f64>, b: &mut DVector<f64>,
                top_bc: BoundaryCondition, bottom_bc: BoundaryCondition) {
        match top_bc {
            BoundaryCondition::FixedHead(h) => { k.row_mut(0).fill(0.0); k[(0,0)] = 1.0; b[0] = h; }
            BoundaryCondition::Flux(flux) => { b[0] += flux; }
        }
        match bottom_bc {
            BoundaryCondition::FixedHead(h) => {
                let n = self.n_nodes - 1;
                k.row_mut(n).fill(0.0); k[(n,n)] = 1.0; b[n] = h;
            }
            BoundaryCondition::FreeDrainage => {}
        }
    }
}

#[derive(Clone, Copy)]
pub enum BoundaryCondition {
    FixedHead(f64),
    Flux(f64),
    FreeDrainage,
}

pub enum SolverError {
    Convergence(String),
    SingularMatrix,
}
```

### 3. Crop Growth Model (Rust)

```rust
// solver-core/src/crop_model.rs

pub struct CropModel {
    pub params: CropParams,
    pub state: CropState,
}

pub struct CropParams {
    pub base_temp_c: f64,
    pub gdd_emergence: f64,
    pub gdd_maturity: f64,
    pub rue_g_mj: f64,
    pub max_lai: f64,
    pub harvest_index: f64,
}

pub struct CropState {
    pub gdd_accumulated: f64,
    pub biomass_total_kg_ha: f64,
    pub lai: f64,
    pub root_depth_cm: f64,
}

impl CropModel {
    pub fn simulate_day(&mut self, weather: &DailyWeather, water_stress: f64, n_stress: f64)
        -> DailyGrowth {
        // Accumulate GDD
        let avg_temp = (weather.tmax_c + weather.tmin_c) / 2.0;
        self.state.gdd_accumulated += (avg_temp - self.params.base_temp_c).max(0.0);

        // Light interception
        let fpar = 1.0 - (-0.5 * self.state.lai).exp();
        let par_intercepted = weather.solar_rad_mj_m2 * 0.5 * fpar;

        // Biomass with stress
        let stress = water_stress.min(n_stress);
        let biomass_incr = self.params.rue_g_mj * par_intercepted * 10_000.0 / 1_000.0 * stress;
        self.state.biomass_total_kg_ha += biomass_incr;

        // Update LAI (simplified)
        self.state.lai = (self.state.biomass_total_kg_ha * 0.005).min(self.params.max_lai);

        DailyGrowth {
            biomass_increment_kg_ha: biomass_incr,
            lai: self.state.lai,
        }
    }
}

pub struct DailyWeather {
    pub tmin_c: f64,
    pub tmax_c: f64,
    pub solar_rad_mj_m2: f64,
}

pub struct DailyGrowth {
    pub biomass_increment_kg_ha: f64,
    pub lai: f64,
}
```

### 4. Weather Data Service (Python FastAPI)

```python
# weather-service/main.py

from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from datetime import date
import cdsapi
import xarray as xr
import boto3
import hashlib

app = FastAPI(title="SoilChem Weather Service")

class WeatherRequest(BaseModel):
    latitude: float
    longitude: float
    start_date: date
    end_date: date
    source: str = "era5"

@app.post("/weather/extract")
async def extract_weather_data(req: WeatherRequest):
    # Check cache
    cache_key = f"{req.source}_{req.latitude:.4f}_{req.longitude:.4f}_{req.start_date}_{req.end_date}"

    # Extract ERA5 data
    c = cdsapi.Client()
    temp_nc = f"/tmp/era5_{hashlib.md5(cache_key.encode()).hexdigest()}.nc"

    c.retrieve('reanalysis-era5-single-levels', {
        'product_type': 'reanalysis',
        'variable': ['2m_temperature', 'total_precipitation', 'surface_solar_radiation_downwards'],
        'year': [str(req.start_date.year), str(req.end_date.year)],
        'area': [req.latitude + 0.25, req.longitude - 0.25, req.latitude - 0.25, req.longitude + 0.25],
        'format': 'netcdf',
    }, temp_nc)

    # Convert to CSV
    ds = xr.open_dataset(temp_nc)
    daily = ds.resample(time='1D').mean()
    df = daily.sel(latitude=req.latitude, longitude=req.longitude, method='nearest').to_dataframe()

    csv_path = f"/tmp/era5_{hashlib.md5(cache_key.encode()).hexdigest()}.csv"
    df.to_csv(csv_path)

    # Upload to S3
    s3 = boto3.client('s3')
    s3_key = f"weather/{cache_key}.csv"
    s3.upload_file(csv_path, "soilchem-data", s3_key)

    return {"data_url": f"s3://soilchem-data/{s3_key}"}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–4)

**Day 1: Project initialization and Rust backend scaffold**
- `cargo init soilchem-api && cd soilchem-api`
- `cargo add axum tokio serde serde_json sqlx uuid chrono tower-http jsonwebtoken bcrypt geo-types`
- `cargo add --features postgres,runtime-tokio-rustls,uuid,chrono,json sqlx`
- `src/main.rs` — Axum app with CORS middleware, tracing subscriber, graceful shutdown handler
- `src/config.rs` — Environment-based configuration (DATABASE_URL, JWT_SECRET, S3_BUCKET, REDIS_URL, CDS_API_KEY)
- `src/state.rs` — AppState struct holding PgPool, Redis client, S3Client
- `src/error.rs` — ApiError enum with IntoResponse implementation for standardized error responses
- `Dockerfile` — Multi-stage build (Rust builder + minimal runtime)
- `docker-compose.yml` — PostgreSQL with PostGIS extension, Redis, MinIO (S3-compatible local storage)
- Test: verify Axum starts, health endpoint returns 200

**Day 2: Database schema and migrations**
- `migrations/001_initial.sql` — All 10 tables with PostGIS extensions enabled, indexes on frequently queried columns
- `src/db/mod.rs` — Database pool initialization with connection pooling (min 5, max 20 connections)
- `src/db/models.rs` — All SQLx structs with FromRow derives: User, Field, SoilProfile, CropVariety, ManagementScenario, SimulationRun, SimulationJob, WeatherData
- Nested JSON structs: SoilLayer, PhenologyParams, GrowthParams, InitialConditions, BoundaryConditions, SolverOptions
- `sqlx migrate run` to apply schema
- Seed script: insert built-in crop varieties (wheat "TAM 111", maize "Pioneer 1151", soybean "Asgrow 3906") with default phenology and growth parameters calibrated from literature
- Test: seed crops exist in database

**Day 3: Authentication system**
- `src/auth/mod.rs` — JWT token generation and validation middleware, Claims struct extraction from Authorization header
- `src/auth/oauth.rs` — Google and GitHub OAuth 2.0 authorization code flow handlers (redirect to provider, callback endpoint, exchange code for token)
- `src/api/handlers/auth.rs` — Endpoints:
  - POST /auth/register — Email + password registration with bcrypt hashing (cost factor 12)
  - POST /auth/login — Credentials validation, return access token (24h expiry) + refresh token (30d expiry)
  - GET /auth/oauth/{provider} — Initiate OAuth flow
  - GET /auth/oauth/{provider}/callback — Handle OAuth callback, create or link user account
  - POST /auth/refresh — Exchange refresh token for new access token
  - GET /auth/me — Get current user profile
- Password requirements: min 8 chars, 1 uppercase, 1 number
- JWT payload: user_id, email, plan, issued_at, expires_at
- Test: register user, login, access protected endpoint with token

**Day 4: Field and soil profile CRUD**
- `src/api/handlers/fields.rs` — Field management with PostGIS:
  - POST /fields — Create field with GeoJSON boundary, auto-calculate area_hectares using ST_Area on geography type
  - GET /fields — List user's fields with pagination, filter by org_id
  - GET /fields/{id} — Get field details including boundary geometry as GeoJSON
  - PATCH /fields/{id} — Update field metadata (name, description, soil_type)
  - DELETE /fields/{id} — Soft delete or cascade delete
  - GET /fields/nearby?lat={}&lon={} — Spatial query for fields within 50km using ST_DWithin
- `src/api/handlers/soil_profiles.rs` — Soil profile management:
  - POST /soil_profiles — Create profile with layers array, validate sand+silt+clay=100% for each layer
  - GET /soil_profiles — List profiles for field or templates
  - POST /soil_profiles/{id}/apply_rosetta — Auto-calculate VG parameters from texture using Rosetta PTF
  - PATCH /soil_profiles/{id}/layers — Update layer depths and properties with re-validation
- `src/pedotransfer/rosetta.rs` — Rosetta neural network PTF implementation (equations from Schaap et al. 2001)
- `src/api/handlers/orgs.rs` — Organization management: create org, invite member (send email), list members, update member role, remove member
- `src/api/router.rs` — Route definitions with auth middleware applied to protected routes
- Integration tests: create field with polygon, query nearby fields, create soil profile with 5 layers, apply Rosetta PTF

### Phase 2 — Solver Core — Prototype (Days 5–12)

**Day 5: Van Genuchten hydraulic functions**
- `solver-core/` — New Rust workspace member (shared between native and WASM compilation targets)
- `solver-core/Cargo.toml` — Dependencies: nalgebra, nalgebra-sparse, serde
- `solver-core/src/hydraulic.rs` — VanGenuchtenParams struct with methods:
  - `theta(h)` — Water retention curve returning volumetric water content
  - `capacity(h)` — Specific moisture capacity C(h) = dθ/dh for Picard iteration
  - `conductivity(h)` — Unsaturated hydraulic conductivity K(h) using Mualem model
  - `effective_saturation(h)` — Helper for Se calculation
- Handle saturated condition (h ≥ 0) as special case
- `solver-core/src/pedotransfer.rs` — Rosetta PTF implementation:
  - Neural network with 2 hidden layers (5 neurons each) predicting log10(α), log10(n), θr, θs, log10(Ks)
  - Input: sand %, silt %, clay %, bulk density, organic matter %
  - Weights and biases hard-coded from Rosetta calibration
- Unit tests: compare θ(h) curves to reference data from Carsel & Parrish 1988 for 12 USDA texture classes, verify K(h) at Se=0.1, 0.5, 0.9

**Day 6: Richards equation FEM framework**
- `solver-core/src/richards.rs` — RichardsSolver struct:
  - `new(layers: &[SoilLayer], dz: f64)` — Constructor that generates FEM mesh
  - Mesh generation: iterate through soil layers, subdivide each layer into elements targeting dz=1-2cm resolution
  - Store node depths z_nodes, element lengths, soil parameters per element
  - Initialize head vector h with hydrostatic equilibrium (h = -z) or user-specified initial condition
  - Initialize theta vector by applying theta(h) to each node
- `assemble_system(dt, sink)` — Build K matrix and b vector for Picard iteration:
  - Loop over elements, compute element-average h, K(h), C(h)
  - Stamp conductance (K/dz) and storage (C/dt) terms into stiffness matrix
  - Add gravity term to RHS
  - Add sink term (root uptake) to RHS
  - Use mass-lumping for storage term (diagonal lumping for stability)
- `apply_bc()` — Apply top and bottom boundary conditions:
  - Top: atmospheric (flux BC), fixed head, seepage face
  - Bottom: free drainage (unit gradient), fixed head (water table), zero flux
- Unit tests: steady-state infiltration with h=0 at top, verify flux = Ks, hydrostatic profile with water table at 100cm

**Day 7: Picard iteration solver**
- `solve_timestep(dt, sink, top_bc, bottom_bc, max_iter, tolerance)` — Picard iteration loop:
  - Save previous head as prev_head
  - For each Picard iteration k:
    - Assemble K matrix and b vector using current head (linearization point)
    - Apply boundary conditions
    - Solve linear system K·h_new = b using nalgebra LU decomposition
    - Check convergence: max_change = ||h_new - h|| < tolerance
    - Update head and theta vectors
    - If converged, return iteration count; else continue
  - If max_iter exceeded, return convergence error
- Adaptive under-relaxation: if convergence stalls, apply h_new = h + omega * (h_new - h) with omega=0.5
- Adaptive time stepping: if Picard fails to converge, halve dt and retry; if converges in <5 iterations, increase dt by 1.5x
- Tests: ponding infiltration (compare cumulative infiltration to Green-Ampt analytical solution), drainage from saturated initial condition

**Day 8: Root water uptake and boundary conditions**
- `solver-core/src/root_uptake.rs` — Feddes stress response function:
  - `alpha(h)` — Stress reduction factor (0-1) as piecewise function of h:
    - α=0 for h < h_wilting (-15000 cm) or h > h_anaerobic (-1 cm)
    - α=1 for h_optimal_low < h < h_optimal_high (-1000 to -300 cm)
    - Linear ramp between thresholds
  - `root_density_distribution(z, root_depth)` — Exponential decay: RDF(z) = exp(-k*z/root_depth), normalized to integrate to 1
- Coupling: compute actual transpiration T_act = Σ(α(h_i) * RDF(z_i) * T_pot) over all nodes in root zone
- Water stress factor for crop model: f_water = T_act / T_pot
- Atmospheric boundary condition implementation:
  - Top flux BC: q_top = rainfall - evaporation (positive into soil)
  - If ponding (h_surface > 0), switch to fixed head BC temporarily
- Tests: simulate root water uptake with varying soil moisture, verify stress factor ranges 0-1, verify ET partitioning (soil evaporation vs transpiration)

**Day 9: Multi-day simulation loop**
- `solver-core/src/simulation.rs` — Main simulation orchestration:
  - `run_simulation(config)` — Entry point taking SimulationConfig struct
  - Initialize RichardsSolver from soil profile
  - Load daily weather data (precip, T_min, T_max, solar radiation)
  - Loop over simulation days:
    - Compute daily ET0 using Penman-Monteith equation (or load from weather data)
    - Partition ET0 into soil evaporation and potential transpiration using LAI-based split
    - Set top BC: flux = precip - evaporation
    - Set sink term: root water uptake based on potential transpiration
    - Call solve_timestep with dt=1 day (may subdivide internally)
    - Update water balance: cumulative infiltration, ET, drainage, storage change
    - Store daily results: theta profile, fluxes, actual vs potential ET
  - Mass balance validation: check (precip + irrigation) - (ET + drainage + ΔS) < 1% of total inputs
- Output format: daily time-series with date, theta per layer, cumulative fluxes
- Tests: 30-day simulation with synthetic constant weather, verify water conservation, compare cumulative ET to expected value

**Day 10: Crop model phenology**
- `solver-core/src/crop_model.rs` — CropModel struct with state tracking:
  - State: gdd_accumulated, current_stage, biomass_total, biomass_leaf, biomass_root, biomass_grain, LAI, root_depth, n_content
  - Phenology stages: Planted → Emerged → Vegetative → Flowering → GrainFilling → Mature
- `simulate_day(weather, water_stress, n_stress)` — Daily crop growth:
  - Accumulate GDD: gdd_today = max(0, (T_max + T_min)/2 - T_base)
  - Check stage transitions based on cumulative GDD thresholds (gdd_emergence, gdd_anthesis, gdd_maturity)
  - Advance to next stage when threshold crossed
- Phenology parameters for wheat: T_base=0°C, gdd_emergence=150, gdd_anthesis=1400, gdd_maturity=2200
- Maize: T_base=10°C, gdd_emergence=100, gdd_anthesis=800, gdd_maturity=1500
- Soybean: T_base=10°C, gdd_emergence=80, gdd_anthesis=700, gdd_maturity=1300 (photoperiod-sensitive)
- Tests: verify stage transitions for 3 crops using historical weather data, check timing ±7 days of literature values

**Day 11: Crop biomass accumulation and LAI**
- Radiation interception: fPAR = 1 - exp(-k · LAI) where k=0.5 (light extinction coefficient)
- PAR = 0.5 × solar radiation (assuming PAR is half of total solar radiation)
- Biomass accumulation: dB/dt = RUE · fPAR · PAR · stress_factor
  - RUE (radiation use efficiency) in g dry matter / MJ intercepted PAR
  - Wheat RUE = 3.0, Maize RUE = 3.5, Soybean RUE = 2.8
  - stress_factor = min(f_water, f_N) — most limiting factor
- Partitioning to organs: biomass distributed to leaf, root, grain based on phenology stage
  - Vegetative: 60% leaf, 40% root
  - Flowering: 30% leaf, 10% root, 60% grain
  - Grain filling: 10% leaf, 0% root, 90% grain
- LAI calculation: LAI = SLA · biomass_leaf / 10000 where SLA (specific leaf area) in m²/kg
  - Wheat SLA = 25, Maize SLA = 22, Soybean SLA = 28
- LAI capped at max_lai (species-specific: wheat=7, maize=6, soybean=5.5)
- Root depth progression: during vegetative stage, root_depth increases by 1 cm/day until reaching max (wheat=150cm, maize=180cm, soybean=120cm)
- Tests: unstressed growth simulation, compare biomass trajectory to reference curves, verify LAI peaks at expected timing, check yield = HI × total biomass

**Day 12: Nitrogen cycling prototype**
- `solver-core/src/nitrogen.rs` — N pool tracking per soil layer:
  - Pools: organic_N, NH4, NO3 (all in kg N/ha per layer)
  - Initialize from user input or defaults (organic_N based on organic matter %, NH4=2, NO3=10 kg/ha)
- Mineralization (first-order kinetics):
  - dN_org/dt = -k_min · f_temp · f_moisture · N_org
  - k_min = 0.02 /day (base rate at 20°C, field capacity)
  - f_temp = Q10^((T-20)/10) where Q10=2
  - f_moisture = (θ - θ_pwp) / (θ_fc - θ_pwp), clamped 0-1
  - Mineralized N flows to NH4 pool
- Nitrification (Michaelis-Menten):
  - dNH4/dt = -V_max · f_temp · f_moisture · NH4 / (K_m + NH4)
  - V_max = 10 kg/ha/day (maximum nitrification rate)
  - K_m = 10 kg/ha (half-saturation constant)
  - Nitrified N flows from NH4 to NO3 pool
- Plant uptake (demand-driven):
  - N_demand from crop model based on critical N concentration dilution curve: N_crit = 3.5 × biomass^(-0.37) for cereals
  - N_uptake = min(N_demand, N_available) where N_available = NO3 + NH4 in root zone weighted by RDF
  - N stress factor: f_N = N_plant / N_crit (0-1)
- Leaching (coupled with water flow):
  - NO3 leached = flux_bottom × C_NO3_bottom where C = NO3 / (theta × soil_volume)
  - NH4 assumed immobile (adsorbed to soil particles)
- Mass balance: total_N_in - total_N_out = ΔN_stored (verify <2% error)
- Tests: mineralization rate vs temperature (verify Q10=2), nitrification follows Michaelis-Menten, leaching mass balance with 100mm drainage event

### Phase 3 — WASM Build + Frontend Soil Profile Editor (Days 13–18)

**Day 13: WASM solver compilation**
- `solver-wasm/` — New workspace member for WASM target
- `solver-wasm/Cargo.toml` — crate-type = ["cdylib"], dependencies: wasm-bindgen, serde-wasm-bindgen, console_error_panic_hook, solver-core
- `solver-wasm/src/lib.rs` — WASM entry points:
  - `#[wasm_bindgen] pub fn run_simulation(profile_json, weather_json, params_json)` — Main entry taking JSON strings, parsing to Rust structs, running simulation, returning results as JSON
  - Error handling: catch Rust panics, log to console, return error JSON
  - Progress callback: optional JS function called every 10 days with progress percentage
- Build pipeline: `wasm-pack build --target web --release`
- Post-build optimization: `wasm-opt -Oz solver-wasm/pkg/*.wasm -o optimized.wasm`
- Target bundle size: <1.5MB gzipped (includes solver + nalgebra sparse matrix)
- JavaScript wrapper: `SpiceSolver` class loading WASM module with async init, providing `runSimulation(config)` method returning Promise
- Test in Node.js: load WASM, run 10-day simulation, verify results match native Rust solver

**Day 14: Frontend scaffold and map integration**
- `npm create vite@latest frontend -- --template react-ts`
- `cd frontend && npm install`
- Dependencies: `npm i zustand @tanstack/react-query axios mapbox-gl @mapbox/mapbox-gl-draw d3 recharts`
- `src/App.tsx` — React Router with routes: /login, /fields, /fields/:id/profile, /fields/:id/simulation, /results/:id
- Auth context: `src/contexts/AuthContext.tsx` — Manages JWT token in localStorage, provides login/logout/isAuthenticated
- Layout: `src/components/Layout.tsx` — Header with logo, nav links, user menu; sidebar for field navigation
- `src/stores/fieldStore.ts` — Zustand store:
  - State: fields array, selectedFieldId, isLoading
  - Actions: fetchFields(), selectField(id), createField(data), updateField(id, data)
- `src/components/FieldMap.tsx` — Mapbox GL JS integration:
  - Initialize map with satellite or terrain basemap
  - Draw controls using mapbox-gl-draw for polygon field boundaries
  - On draw.create event, POST boundary GeoJSON to /fields API
  - Display existing fields as polygons with hover tooltip showing name and area
  - Click polygon to select field and navigate to detail view
- `src/lib/api.ts` — Axios client with base URL, auth interceptor adding JWT to headers, error handling
- Test: create field by drawing polygon, verify saved in database, reload page and see field displayed

**Day 15: Soil profile editor — layer management**
- `src/components/SoilProfileEditor/LayerList.tsx` — Vertical list of soil layers:
  - Each layer shows: depth range (e.g., "0-20 cm"), texture class, edit/delete buttons
  - Drag-and-drop to reorder layers (update depth_top/depth_bottom accordingly)
  - "Add Layer" button inserts new layer at clicked depth
  - Color-coded by texture class (sand=yellow, clay=red, loam=brown)
- `src/components/SoilProfileEditor/LayerForm.tsx` — Edit modal for layer properties:
  - Depth inputs: top_cm and bottom_cm with validation (bottom > top, no gaps between layers)
  - Texture inputs: sand, silt, clay sliders (0-100%) with live validation that sum=100%
  - Organic matter % input (0-10% range)
  - Bulk density input (1.0-1.8 g/cm³)
  - "Apply Rosetta" button: calls API to calculate θs, θr, α, n, Ks from texture+OM+BD
  - Advanced toggle: manually override Rosetta estimates (for expert users with lab data)
- `src/components/SoilProfileEditor/TextureTriangle.tsx` — USDA texture triangle component:
  - SVG rendering of equilateral triangle with USDA texture class boundaries
  - Three axes: sand (bottom-left to apex), silt (bottom-right to bottom-left), clay (apex to bottom-right)
  - Click on triangle: calculate sand/silt/clay % from click coordinates using barycentric projection
  - Display marker at current layer's texture point
  - Hover shows texture class name (e.g., "Silty Clay Loam")
  - Grid lines every 10% for easier reading
- Integration: clicking triangle auto-fills sand/silt/clay in LayerForm
- Store in Zustand: soilProfileStore with layers array, selected layer index, CRUD actions
- Test: add 5 layers, click texture triangle, apply Rosetta, verify VG parameters calculated

**Day 16: Soil profile visualization**
- `src/components/SoilProfileEditor/ProfileVisualization.tsx` — SVG-based depth profile:
  - Vertical axis: depth from 0 to total_depth_cm (e.g., 0-200 cm)
  - Horizontal axis: single column showing layers stacked
  - Each layer rendered as rectangle with:
    - Height proportional to layer thickness
    - Fill color based on texture class (categorical color scale)
    - Border indicating layer boundary
  - Drag handles on layer boundaries: click-drag to adjust depth, updates layer depth_top/depth_bottom in real-time
  - Constraints: cannot drag below previous layer or above next layer
  - Hover tooltip: shows layer properties (sand/silt/clay %, θs, Ks, initial θ)
- Depth scale on left: tick marks every 10 cm with labels
- Responsive sizing: adapts to container height
- Animation: smooth transition when layers change (D3.js transitions)
- Test: drag layer boundary, verify depth updates in both visual and layer list

**Day 17: Crop variety selector and management editor**
- `src/components/ManagementScenario/CropSelector.tsx` — Crop selection:
  - Dropdown grouped by crop type (Cereals: wheat, maize; Legumes: soybean)
  - Built-in varieties loaded from /crop_varieties API
  - "Create Custom Variety" button: opens modal to clone and edit phenology/growth parameters
  - Display key parameters for selected variety: GDD requirements, RUE, max LAI, harvest index
- `src/components/ManagementScenario/IrrigationEvents.tsx` — Irrigation event table:
  - Columns: Date, Depth (mm), Method (dropdown: drip, sprinkler, surface), Efficiency (%)
  - Add row button: date picker for scheduling
  - Delete row button
  - Sort by date automatically
  - Calculate total irrigation water applied (sum of depths)
- `src/components/ManagementScenario/FertilizerEvents.tsx` — Fertilizer application table:
  - Columns: Date, N (kg/ha), P (kg/ha), K (kg/ha), Type (dropdown: urea, ammonium nitrate, manure)
  - Similar add/delete/sort functionality
  - Calculate total N applied
- Date pickers: integrated calendar view with season context (show planting/harvest dates)
- `src/stores/managementStore.ts` — Zustand store for scenario state
- Save/load scenarios to/from database via API
- Test: create scenario with 3 irrigation events and 2 fertilizer applications, save, reload and verify data persists

**Day 18: Simulation configuration panel**
- `src/components/Simulation/ConfigPanel.tsx` — Simulation setup:
  - Date range: start_date and end_date pickers (validate start < end, max 365 days for free tier)
  - Simulation type checkboxes: water_balance (default on), crop_growth, nitrogen_cycling, full (all modules)
  - Initial conditions section:
    - Water table depth (cm): optional, default=null (free drainage bottom)
    - Initial soil temperature (°C): default=15
    - Initial nitrate per layer (kg/ha): array input or default=10
  - Solver options (collapsible "Advanced" accordion):
    - Max Picard iterations: default=20, range 5-50
    - Convergence tolerance: default=0.0001, range 1e-6 to 1e-2
    - Max timestep (days): default=1.0, range 0.01-1.0
  - Weather data source:
    - Radio buttons: "Auto (ERA5)", "Upload CSV", "Manual entry"
    - For AUTO: displays field location, will auto-fetch from weather service
    - For CSV: file upload with format validation (columns: date, tmin, tmax, precip, solar_rad)
    - For Manual: table to enter daily data (tedious, mainly for testing)
  - Execution mode display: shows "Client (WASM)" or "Server" based on layer count and duration, with explanation tooltip
  - "Run Simulation" button: disabled if validation fails, shows loader spinner when clicked
- Validation: check soil profile exists, management scenario exists, date range valid, weather data available
- On submit: POST /fields/{id}/simulations with config payload, navigate to results page with simulation_id
- Test: configure 90-day simulation, verify WASM mode selected, click Run, verify simulation created in database with status="pending_client"

### Phase 4 — API + Job Orchestration (Days 19–24)

**Day 19: Simulation API endpoints**
- `src/api/handlers/simulation.rs` — Simulation lifecycle management:
  - POST /fields/{field_id}/simulations — Create simulation endpoint (see Architecture Deep-Dive code)
  - GET /simulations/{id} — Fetch simulation record with status, results_url, summary
  - GET /fields/{field_id}/simulations — List simulations for field with filters (status, date range)
  - PATCH /simulations/{id}/cancel — Cancel running server-side simulation (update status, stop worker)
- Routing logic: layer count from profile, duration from dates, apply WASM threshold (≤20 layers AND ≤90 days)
- Plan limits enforcement:
  - Free: 90 days max, 10 layers max, 3 simulations/month
  - Pro: 365 days max, 20 layers max, unlimited simulations
  - Advanced: unlimited
- Validation: check profile.total_depth_cm > 0, end_date > start_date, crop variety exists, management events have valid dates
- Return simulation record with status="pending_client" for WASM, "pending" for server
- Test: create simulation exceeding free tier limit, verify 402 Payment Required error

**Day 20: Weather data integration**
- `weather-service/` directory with Python FastAPI service
- `weather-service/requirements.txt`: fastapi, uvicorn, cdsapi, xarray, netCDF4, pandas, boto3, asyncpg
- `weather-service/main.py` — FastAPI app (see Architecture Deep-Dive code)
- `/weather/extract` endpoint: takes lat/lon/start_date/end_date, queries PostgreSQL cache, if miss then calls ERA5 CDS API
- ERA5 extraction: 0.25° resolution, nearest grid point to field location, variables: 2m temp, precip, solar radiation
- Daily aggregation: resample hourly ERA5 to daily min/max temp, sum precip, mean solar
- CSV generation: columns [date, tmin_c, tmax_c, precip_mm, solar_rad_mj_m2, wind_speed_m_s]
- S3 upload: save to `weather/{source}/{cache_key}.csv`, return presigned URL with 1h expiry
- Cache in PostgreSQL: insert into weather_data table with location (PostGIS point), source, date range, S3 URL
- Rust API client: `src/weather/client.rs` — HTTP client calling weather service
  - `fetch_weather_data(lat, lon, start_date, end_date)` — Returns S3 URL or downloads CSV and parses to Vec<DailyWeather>
- Integration: before running simulation, check if weather data exists, if not call weather service, wait for download, then proceed
- Test: request weather for Kansas (39.5°N, 98.4°W) for 2023-01-01 to 2023-03-31, verify CSV downloaded from S3 with 90 rows

**Day 21: Server-side simulation worker**
- `src/workers/simulation_worker.rs` — Background worker process:
  - Main loop: block-pop from Redis list "simulation:jobs" with 30s timeout
  - On job received: fetch simulation_run and simulation_job records from database
  - Load soil profile, management scenario, crop variety, weather data from S3
  - Parse soil layers to create RichardsSolver instance
  - Initialize CropModel with variety parameters
  - Run simulation loop: for each day, solve Richards timestep, simulate crop growth, update N cycling
  - Every 10 days: update simulation_job.progress_pct, publish progress to Redis channel "sim:{simulation_id}:progress"
  - On completion: serialize results to Parquet (columnar format for time-series), upload to S3, update simulation_run with results_url and status="completed"
  - On error: update simulation_run with error_message and status="failed", log stack trace
- Worker pool: run 1-10 worker processes depending on load (managed by supervisor or Kubernetes)
- Graceful shutdown: on SIGTERM, finish current job then exit
- Error handling: convergence failures (report which layer and day), out-of-memory (circuit breaker to prevent retry loops), timeouts (kill after 10 minutes)
- Test: enqueue job via Redis CLI, verify worker picks it up, runs simulation, uploads results to S3

**Day 22: WebSocket for live simulation progress**
- `src/api/ws/mod.rs` — Axum WebSocket handler:
  - GET /ws/simulations/{id}/progress — Upgrade HTTP to WebSocket
  - Authenticate via query param: ?token={jwt}
  - On connection: subscribe to Redis pub/sub channel "sim:{id}:progress"
- `src/api/ws/simulation_progress.rs` — Progress broadcaster:
  - Worker publishes JSON messages: `{"progress_pct": 45, "current_day": "2023-02-14", "picard_iters": 8, "mass_balance_error": 0.0003}`
  - WebSocket receives from Redis, forwards to client
  - On simulation completion: send final message with status="completed", results_url
- Connection management:
  - Heartbeat: server sends ping every 30s, client must respond with pong within 10s or disconnect
  - Auto-cleanup: on disconnect, unsubscribe from Redis channel, close WebSocket
- Frontend: `src/hooks/useSimulationProgress.ts` — React hook:
  - Opens WebSocket connection to /ws/simulations/{id}/progress
  - State: progress_pct, current_day, status
  - On "completed" message: refetch simulation record, redirect to results page
- Test: start server simulation, connect WebSocket, verify progress updates received every 10 days, verify completion message triggers navigation

**Day 23: Result storage and retrieval**
- S3 result format: Parquet file with schema:
  - Columns: date (Date), layer_depth_cm (List<Float>), theta_profile (List<Float>), NO3_profile (List<Float>), biomass_kg_ha (Float), lai (Float), et_actual_mm (Float), drainage_mm (Float), n_leached_kg_ha (Float)
  - One row per simulation day
  - Compression: Snappy (fast decompress for client download)
- Presigned URL generation: on simulation completion, generate S3 presigned GET URL with 1h expiry
- Result summary calculation: aggregate time-series for dashboard display:
  - total_et_mm = sum(et_actual_mm)
  - total_drainage_mm = sum(drainage_mm)
  - total_n_leached_kg_ha = sum(n_leached_kg_ha)
  - final_yield_kg_ha = biomass_kg_ha[last] × harvest_index
  - max_lai = max(lai)
  - water_stress_days = count(et_actual < 0.8 × et_potential)
- Store summary as JSONB in simulation_runs.results_summary for fast querying without S3 fetch
- `src/api/handlers/results.rs` — Result endpoints:
  - GET /simulations/{id}/results — Returns results_summary JSON
  - GET /simulations/{id}/results/download — Returns presigned S3 URL for Parquet download or streams CSV conversion
- Frontend: on results page load, fetch summary, display key metrics in cards, lazy-load full time-series for charts
- Test: complete simulation, verify Parquet file in S3, verify summary calculated correctly, download via presigned URL

**Day 24: Project templates and examples**
- Seed database with example projects in `migrations/002_seed_examples.sql`:
  - "Maize Irrigation Scheduling - Nebraska" — 90-day maize simulation with 3 irrigation events, demonstrates MAD scheduling
  - "Winter Wheat Rainfed - Kansas" — 240-day wheat simulation, shows dormancy and spring growth
  - "Soybean Nitrogen Management - Iowa" — 120-day soybean with fertilizer optimization comparison
- Each example includes: field location, soil profile (typical for region), crop variety, management scenario, weather data reference
- Template soil profiles:
  - "Loam (default)" — 4 layers, 0-150 cm, balanced texture
  - "Sandy Loam" — 3 layers, high drainage
  - "Clay Loam" — 5 layers, low drainage, high water holding capacity
  - "Silt Loam" — 4 layers, typical Midwest US
- `src/api/handlers/templates.rs` — Template endpoints:
  - GET /templates/projects — List example projects with thumbnail, description
  - POST /templates/projects/{id}/clone — Clone example to user's account with editable copy
  - GET /templates/soil_profiles — List template profiles
  - POST /templates/soil_profiles/{id}/clone — Clone template profile
- Frontend: Templates page with gallery of cards, click "Use This Template" to clone and navigate to editor
- Test: clone "Maize Irrigation" template, verify field and soil profile created in user's account, run simulation, verify matches reference results

### Phase 5 — Results Visualization UI (Days 25–29)

**Day 25: Water balance time-series chart**
- `src/components/Results/WaterBalanceChart.tsx` — D3.js multi-series line chart:
  - X-axis: date range from start_date to end_date (time scale)
  - Dual Y-axes: left=flux (mm/day), right=cumulative (mm)
  - Series:
    - Precipitation (bars, blue)
    - ET actual (line, green)
    - Drainage (line, orange)
    - Change in storage (line, purple)
  - Interactive features:
    - Tooltip on hover: shows all series values for date
    - Brush zoom: click-drag to zoom into date range
    - Reset zoom button
    - Legend with series toggle (click to hide/show)
  - Data preparation: load full time-series from S3 Parquet (via presigned URL), parse in browser using arrow-js or fetch as JSON
- Render pipeline: D3 scales, axes, path generators, SVG elements
- Responsive: adapts to container width, redraws on resize
- Test: load 365-day simulation results, verify all series render, zoom into 30-day window, verify tooltip shows correct values

**Day 26: Soil moisture depth profile animation**
- `src/components/Results/SoilMoistureProfile.tsx` — Animated SVG depth profile:
  - X-axis: water content θ (0 to θ_sat)
  - Y-axis: depth (0 to total_depth_cm, inverted so 0 at top)
  - For each day: plot theta_profile as color-coded depth bands
  - Color scale: continuous gradient from red (dry, θ→θ_res) → yellow (wilting) → green (field capacity) → blue (saturated)
  - Layer boundaries: horizontal lines showing soil layer structure
  - Time slider: scrub through simulation days (day 1 to N)
  - Play/pause button: auto-advance at 5 fps
  - Speed control: 1x, 2x, 5x playback
- Hover: show θ value and depth on mouseover
- Sync with water balance chart: clicking chart date jumps profile animation to that day
- Test: load 90-day results, play animation, verify moisture front propagation during rainfall events, verify drying during drought

**Day 27: Crop growth dashboard**
- `src/components/Results/CropGrowthDashboard.tsx` — Grid layout with 4 sub-charts:
  1. Biomass accumulation (stacked area chart):
     - Series: leaf, root, grain biomass (kg/ha)
     - Stacked to show total biomass
     - X-axis: date, Y-axis: biomass (kg/ha)
  2. LAI trajectory (line chart):
     - LAI over time (0-7)
     - Horizontal reference lines: LAI=3 (full canopy), LAI=5 (peak typical)
     - Annotate phenology stages (emergence, anthesis, maturity) with vertical lines
  3. Phenology timeline:
     - Horizontal bar showing days in each stage
     - Color-coded stages
     - GDD accumulation line overlay
  4. Water stress heatmap:
     - Daily water stress factor (0-1) as color-coded cells
     - Red=high stress, green=no stress
     - X-axis: date (grouped by week), Y-axis: single row
- Layout: 2x2 grid on desktop, stack vertically on mobile
- Test: load crop simulation results, verify biomass curve shows sigmoidal growth, LAI peaks at anthesis, stress heatmap highlights drought periods

**Day 28: Nitrogen budget visualization**
- `src/components/Results/NitrogenBudget.tsx` — Sankey diagram showing N flows:
  - Nodes: Fertilizer Applied, Organic N, NH4, NO3, Plant N, Leached N
  - Flows (links):
    - Fertilizer → NO3 (amount applied)
    - Organic N → NH4 (mineralization)
    - NH4 → NO3 (nitrification)
    - NO3 → Plant N (uptake)
    - NO3 → Leached N (drainage losses)
  - Link width proportional to N mass (kg/ha)
  - Color-coded by pool type
- Cumulative mass balance table below Sankey:
  - Inputs: fertilizer_applied_kg_ha, mineralization_kg_ha
  - Outputs: plant_uptake_kg_ha, leached_kg_ha, volatilization_kg_ha
  - Balance: inputs - outputs = ΔN_soil
  - Error: (balance / total_inputs) × 100 (should be <2%)
- Implemented using D3 Sankey layout library
- Test: load N cycling results, verify Sankey shows fertilizer→NO3→plant path, verify mass balance table sums correctly

**Day 29: Comparison view and export**
- `src/components/Results/ComparisonView.tsx` — Side-by-side scenario comparison:
  - Select 2-4 simulations to compare (from same field, different management)
  - Display water balance charts in synchronized columns (shared X-axis zoom)
  - Summary metrics comparison table:
    - Columns: Scenario 1, Scenario 2, Scenario 3, Scenario 4
    - Rows: Total Yield (kg/ha), Total ET (mm), Water Use Efficiency (kg/m³), N Applied (kg/ha), N Leached (kg/ha), N Use Efficiency (%)
    - Highlight best value in each row (green background)
  - Difference visualization: bar chart showing % change from baseline scenario
- Export functionality:
  - CSV time-series download: button triggers download of results Parquet converted to CSV
  - PNG chart export: uses html2canvas to render chart as image, download
  - PDF report generation: calls backend API to generate multi-page PDF with charts and tables (see Day 33)
- Test: compare 2 irrigation schedules, verify metrics differ, download CSV and verify format

### Phase 6 — Advanced Features (Days 30–35)

**Day 30: Irrigation scheduling optimizer**
- `src/components/IrrigationScheduler/AutoSchedule.tsx` — Auto-schedule UI:
  - Inputs: target soil moisture (% of field capacity), default=70%
  - Irrigation method: drip/sprinkler/surface (affects application efficiency and depth)
  - Forecast mode toggle: use 30-day climate forecast to pre-schedule
  - "Generate Schedule" button
- Backend `src/optimization/irrigation.rs` — Greedy scheduling algorithm:
  - Simulate forward day-by-day without irrigation
  - At each day: check root zone average θ
  - If θ < threshold × θ_fc: schedule irrigation event to refill to θ_fc
  - Irrigation depth = (θ_fc - θ_current) × root_depth / efficiency
  - Continue simulation with irrigation applied
  - Collect recommended events: [(date, depth_mm)]
- Forecast integration: call weather service with forecast=true to get GFS 30-day forecast instead of historical
- Output: table of recommended irrigation dates and depths, projected yield improvement (% increase vs rainfed), total water savings vs fixed schedule
- User can accept schedule (add events to management scenario) or edit manually
- Test: run optimizer for maize in Nebraska drought year, verify 5-7 irrigation events scheduled, verify yield improvement >20%

**Day 31: Sensitivity analysis tool**
- `src/components/Analysis/SensitivityAnalysis.tsx` — Parameter perturbation analysis:
  - Select parameters to vary: Ks (±50%), RUE (±20%), fertilizer_rate (±30%)
  - Number of Monte Carlo runs: 100 (default), 500 (advanced)
  - "Run Analysis" button: submits batch of simulations with perturbed parameters
- Backend: create N simulation jobs with parameter sampling:
  - Latin Hypercube Sampling for efficient parameter space coverage
  - Run all simulations in parallel on server (requires Pro/Advanced plan)
  - Aggregate results: collect yield, ET, N leaching for each run
- Output visualizations:
  - Tornado chart: horizontal bar chart showing parameter impact on yield (rank by absolute effect)
  - Scatter plots: Ks vs Yield, RUE vs Yield, etc.
  - Probability distribution: histogram of yield outcomes
- Identifies most influential parameters for calibration focus
- Test: run 100-sample sensitivity on wheat model, verify Ks and RUE have highest impact on yield

**Day 32: Multi-field batch simulation**
- `src/components/BatchSimulation/FieldSelector.tsx` — Multi-select field list:
  - Checkboxes for fields in organization
  - "Apply Scenario to Selected Fields" button
  - Select single management scenario to apply across all fields (each field has own soil profile)
- Backend: POST /batch_simulations — Create simulation_run for each field with same scenario
  - Enqueue all jobs to Redis queue
  - Return batch_id for tracking
- Worker: parallel execution (limited by worker count, queue depth)
- Aggregate results: POST /batch_simulations/{batch_id}/aggregate
  - Collect yield, ET, N leaching per field
  - Calculate statistics: mean, std dev, min, max across fields
- Frontend: `src/components/BatchSimulation/FieldComparison.tsx` —
  - Table: field_name, area_hectares, yield_kg_ha, et_mm, n_leached_kg_ha
  - Sortable columns
  - Spatial map: Mapbox with fields color-coded by yield (green=high, red=low)
- Test: batch-run 10 fields with same wheat scenario, verify all complete, view map showing yield variability

**Day 33: PDF report generation**
- `src/api/handlers/reports.rs` — Generate agronomic report:
  - POST /simulations/{id}/report — Trigger PDF generation
  - Server-side HTML template with simulation results, charts embedded as base64 images
  - Use headless Chrome (via puppeteer or wkhtmltopdf) to render HTML to PDF
  - Upload PDF to S3, return presigned URL
- Report sections (8-10 pages):
  1. Cover: field name, simulation dates, SoilChem logo
  2. Field metadata: location map, area, soil type
  3. Soil profile diagram: layer depths and textures
  4. Management timeline: planting date, irrigation events, fertilizer applications
  5. Water balance summary: total ET, drainage, storage change, table and chart
  6. Crop growth summary: final yield, max LAI, biomass trajectory chart
  7. Nitrogen budget: inputs/outputs table, Sankey diagram
  8. Recommendations: irrigation schedule optimization, fertilizer rate suggestions
- Free plan: reports have "Generated with SoilChem Free" watermark
- Pro/Advanced: no watermark, custom logo option
- Test: generate report for 90-day maize simulation, verify PDF downloads, contains all sections, charts render correctly

**Day 34: Sensor integration placeholder**
- `src/components/Sensors/SensorIntegration.tsx` — UI mockup with "Connect Sensor" modal (brands: Sentek, Meter, CropX), API key input, field selector, mock data table
- Backend stub: `src/api/handlers/sensors.rs` with POST /sensors endpoint (metadata only), POST /sensors/{id}/data webhook (logs to console)
- `solver-core/src/assimilation.rs` — EnKF struct with TODO method stubs
- UI displays "Coming Soon - Advanced Plan Feature" banner with roadmap link
- Test: open page, verify UI loads, modal shows, form submit (non-functional)

**Day 35: Carbon model stub**
- `solver-core/src/carbon.rs` — Placeholder for CENTURY-based SOC dynamics (struct CarbonModel with empty method stubs)
- `src/components/Carbon/SOCModeling.tsx` — UI framework: baseline/project scenarios, tillage/residue selectors, 10-100 year period, disabled "Run" button with "Available in v1.1" tooltip
- Purpose: establish UI structure for post-MVP functional carbon model integration
- Test: navigate to carbon modeling page, verify UI renders, button disabled

### Phase 7 — Billing + Deployment (Days 36–40)

**Day 36: Stripe integration**
- Create Stripe account, get API keys (test mode for development)
- `src/api/handlers/billing.rs` — Stripe endpoints:
  - POST /billing/checkout — Create Stripe Checkout session for plan upgrade:
    - Input: plan (pro | advanced)
    - Create Stripe customer if not exists (store stripe_customer_id in users table)
    - Create checkout session with price_id for selected plan, return session.url for redirect
  - POST /billing/portal — Create Stripe Customer Portal session:
    - Allows user to manage subscription (update payment method, view invoices, cancel)
  - POST /webhooks/stripe — Stripe webhook handler:
    - Verify signature using Stripe webhook secret
    - Handle events:
      - `checkout.session.completed` → Update user.plan, user.stripe_subscription_id
      - `customer.subscription.updated` → Update user.plan if plan changed
      - `customer.subscription.deleted` → Downgrade user.plan to "free"
      - `invoice.payment_failed` → Send email notification, suspend account after 3 failures
- Plan mapping in Stripe dashboard:
  - Pro: $99/month, recurring
  - Advanced: $249/month, recurring
- Test: create checkout session, complete test payment with Stripe test card (4242...), verify webhook updates user.plan

**Day 37: Plan limits enforcement and quota UI**
- `src/middleware/plan_limits.rs` — Middleware checking limits before simulation creation:
  - Free: max_duration_days=90, max_layers=10, max_simulations_per_month=3
  - Pro: max_duration_days=365, max_layers=20, unlimited simulations
  - Advanced: unlimited all
  - Query usage_records for current month, count simulations, check against limit
  - If exceeded: return 402 Payment Required with message "Free plan limit: 3 simulations/month. Upgrade to Pro."
- `src/services/usage.rs` — Usage tracking:
  - On simulation completion: insert usage_record with record_type="simulation_days", quantity=duration, period_start/end=current month
  - Calculate monthly totals for billing display
- `src/components/Billing/PlanLimits.tsx` — Quota dashboard:
  - Usage meters: "Simulations this month: 2 / 3" with progress bar
  - Storage used: "0.5 GB / 1 GB" (S3 storage quota for results)
  - Approaching-limit warnings: yellow banner at 80%, red at 100%
  - "Upgrade to Pro" call-to-action button
- Billing history table: list of invoices with date, amount, status, download PDF link (from Stripe)
- Cancel subscription flow: confirm modal "Are you sure? Your plan will downgrade to Free at period end."
- Test: create 3 simulations on free plan, attempt 4th, verify 402 error, upgrade to Pro via Stripe, verify 4th simulation allowed

**Day 38: AWS production deployment**
- AWS account setup, create VPC with public/private subnets across 2 AZs
- Terraform / CDK infrastructure as code:
  - RDS PostgreSQL 16 with PostGIS extension: db.t4g.large, Multi-AZ, automated backups
  - ElastiCache Redis: cache.t4g.medium, cluster mode enabled
  - ECS Fargate cluster: API service (2-20 tasks with auto-scaling on CPU 70%)
  - ECS Fargate: simulation worker service (1-10 tasks scaling on Redis queue depth)
  - Application Load Balancer: HTTPS listener with ACM certificate for soilchem.com
  - S3 buckets: soilchem-results (simulation outputs), soilchem-weather (weather cache), soilchem-static (frontend)
  - CloudFront distribution: origin=S3 static bucket, custom domain, SSL certificate
  - Secrets Manager: store DATABASE_URL, JWT_SECRET, STRIPE_SECRET_KEY, CDS_API_KEY
- ECS task definitions: Docker images pushed to ECR
- Security groups: ALB→ECS (port 8000), ECS→RDS (port 5432), ECS→Redis (port 6379)
- CI/CD: GitHub Actions workflow:
  - On push to main: run tests, build Docker images, push to ECR, update ECS services
  - Separate workflows for API and workers
- Test: deploy to staging environment, verify API responds to health check, run end-to-end smoke test

**Day 39: Monitoring and observability**
- Prometheus: scrape metrics from API and workers (custom metrics: simulation_duration_seconds histogram, picard_iterations_total counter, simulation_status gauge)
- Grafana dashboards:
  - System health: CPU/memory per ECS task, RDS connections, Redis memory
  - Simulation throughput: simulations completed per hour, average duration, queue depth
  - User activity: active users, signups, simulations created (grouped by plan)
- Sentry error tracking:
  - Rust: configure sentry-rust with DSN, capture panics and errors
  - TypeScript: configure @sentry/react with DSN, capture exceptions and breadcrumbs
  - Source maps uploaded for TypeScript stack traces
- CloudWatch logs: centralized logging for API and workers
  - Structured JSON logs with tracing spans
  - Alerts: critical error rate >1/min, P99 latency >5s, queue depth >100
- Health check endpoints: `/health/live` (always returns 200), `/health/ready` (checks DB and Redis connectivity)
- Test: trigger error in API, verify appears in Sentry with stack trace, verify Grafana shows spike in error rate

**Day 40: Load testing and optimization**
- Locust load test script (Python):
  - Simulate 100 concurrent users
  - User flow: register → create field → create soil profile → run simulation → view results
  - Ramp-up: 10 users/sec over 10 seconds
  - Run for 10 minutes, collect metrics
- Target performance:
  - API GET requests: p95 <200ms, p99 <500ms
  - Simulation creation: <1 second
  - WASM simulation: <5 seconds for 20 layers × 90 days
  - Server simulation: <30 seconds for 20 layers × 365 days
- Identify bottlenecks: APM tools (DataDog or New Relic) to trace slow requests
- Optimizations applied:
  - Database: add indexes on simulation_runs(user_id, created_at), fields(owner_id)
  - Connection pooling: tune PgPool min/max connections
  - S3 presigned URL caching: cache URLs in Redis for 1 hour to avoid repeated signing
  - WASM bundle preloading: start download on page load before user needs it
- Test: run Locust test, verify 100 concurrent users sustained, verify API latency targets met

### Phase 8 — Polish + Launch (Days 41–42)

**Day 41: Documentation and onboarding**
- Help center articles: Getting Started, Creating Fields, Understanding Soil Profiles, Running Simulations, Interpreting Results, Irrigation Scheduling, Nitrogen Management, Plan Limits
- Video tutorial: 3-minute narrated walkthrough (create field → profile → simulate → results), upload to YouTube
- API documentation: OpenAPI spec (utoipa crate), Swagger UI at /api/docs, Postman collection
- Onboarding checklist: Create field ✓ Add profile ✓ Run simulation ✓ View results ✓ (progress bar, confetti on completion)
- Test: new user completes onboarding without errors

**Day 42: Final QA and launch preparation**
- End-to-end testing: complete user journey from signup to simulation and results (test all plan tiers)
- Cross-browser testing: Chrome, Firefox, Safari, Edge (desktop + mobile)
- Security audit: SQL injection (SQLx prevents), XSS (React escaping), CSRF (OAuth state), rate limiting (100 req/min), JWT validation
- Privacy policy and terms of service pages added
- Launch checklist: DNS configured, SSL valid, analytics installed, Sentry configured, RDS backups enabled, alerts configured
- Soft launch: deploy to production, test with Stripe live mode
- Launch announcement: Product Hunt, LinkedIn, Twitter, agricultural forums, press outreach
- Test: full smoke test in production, verify payment flow

---

## Critical Files Tree

```
soilchem/
├── backend/
│   ├── Cargo.toml
│   ├── migrations/
│   │   └── 001_initial.sql              # Schema + PostGIS
│   ├── soilchem-api/src/
│   │   ├── main.rs
│   │   ├── auth/mod.rs
│   │   ├── api/handlers/
│   │   │   ├── simulation.rs            # Create/get simulation
│   │   │   ├── fields.rs
│   │   │   └── results.rs
│   │   ├── db/models.rs                 # SQLx structs
│   │   └── workers/simulation_worker.rs
│   ├── solver-core/src/
│   │   ├── richards.rs                  # Richards FEM (400 lines)
│   │   ├── crop_model.rs                # Phenology + biomass (300 lines)
│   │   ├── nitrogen.rs                  # N cycling (200 lines)
│   │   └── hydraulic.rs                 # Van Genuchten
│   └── solver-wasm/src/
│       └── lib.rs                       # WASM entry
├── weather-service/
│   └── main.py                          # FastAPI ERA5 extraction
└── frontend/src/
    ├── components/
    │   ├── SoilProfileEditor/
    │   │   ├── TextureTriangle.tsx      # USDA triangle (200 lines)
    │   │   └── ProfileVisualization.tsx
    │   ├── Results/
    │   │   ├── WaterBalanceChart.tsx    # D3.js (300 lines)
    │   │   └── SoilMoistureProfile.tsx  # Animated depth
    │   └── FieldMap.tsx                 # Mapbox
    └── lib/wasm-loader.ts
```

---

## Solver Validation Suite

### Benchmark 1: Constant-Head Infiltration (Steady-State)

**Setup:** 100cm loam (θs=0.43, θr=0.078, α=0.036, n=1.56, Ks=25 cm/d), h=0 top, free drain bottom

**Expected:** Steady flux **24.8-25.2 cm/day**, time to steady **3-5 days**, mass balance error **<0.1%**

### Benchmark 2: Green-Ampt Infiltration (Transient)

**Setup:** Sandy loam, 50 cm/d rain × 2h, dry initial θ=0.15

**Expected:** Cumulative infiltration **98-102 cm** at 2h, wetting front **45-50 cm**, mass balance **<0.5%**

### Benchmark 3: Wheat Crop Water Balance (90-day Season)

**Setup:** Kansas silt loam, winter wheat, 2015-16 season (450mm precip)

**Expected:** Total ET **380-420 mm**, yield **3200-3800 kg/ha**, max LAI **4.5-5.5**, N leaching **12-18 kg/ha**

### Benchmark 4: Irrigation Scheduling (Maize 120-day)

**Setup:** Nebraska loam, maize, 2018 drought, MAD=30%

**Expected:** **6-8 irrigation events**, total **300-400 mm**, yield **11,000-13,000 kg/ha**, WUE **2.2-2.6 kg/m³**

### Benchmark 5: WASM Performance

**Setup:** 20 layers, 90 days, browser (Chrome/M1)

**Expected:** Total time **3.5-5.0 sec**, avg **40-55 ms/day**, memory **<150 MB**, no convergence failures

---

## Verification Checklist

### Core Solver
- [ ] Van Genuchten θ(h) matches reference curves (Wösten 1999)
- [ ] Rosetta PTF within ±15% of lab measurements
- [ ] Richards solver passes 5 benchmarks <10% error
- [ ] Mass balance error <0.5% (water conservation)
- [ ] Picard converges <20 iterations (95% of timesteps)

### Crop Model
- [ ] GDD accumulation matches literature (3 crops)
- [ ] Phenology ±7 days of DSSAT
- [ ] Yield predictions ±20% of DSSAT (10 site-years)
- [ ] LAI dynamics follow expected pattern

### Nitrogen
- [ ] Mineralization 0.01-0.05 /day at 20°C
- [ ] Q10 = 2 ± 0.2 (temperature factor)
- [ ] Leaching mass balance closes ±2%

### API & Infrastructure
- [ ] JWT + OAuth completes successfully
- [ ] PostGIS field geometry stores correctly
- [ ] 20-layer profile serializes without loss
- [ ] Redis queue processes 100 jobs
- [ ] S3 presigned URL downloads work

### Performance
- [ ] WASM 90-day <5 sec (p95)
- [ ] Server 365-day <30 sec
- [ ] API GET <200ms (p95)
- [ ] WASM bundle <1.5 MB gzipped

---

## Deployment Architecture

```
CloudFront → ALB → ECS API (2-20 tasks) → RDS PostgreSQL + PostGIS
                                        → ElastiCache Redis → Workers (1-10)
                                                           → S3 results

Python Weather Service (Fargate) → S3 weather cache
```

**Scaling:** API auto-scales on 70% CPU, workers on queue length

**Cost:** ~$470/mo (1K users), ~$1,950/mo (10K users)

---

## Post-MVP Roadmap

### Phase 1 (Months 2-3): Carbon Sequestration MRV
- CENTURY SOC dynamics, Verra VM0042 compliance, Monte Carlo uncertainty
- **Impact:** Unlock Advanced plan ($249/mo), carbon project market

### Phase 2 (Months 4-5): Sensor Data Assimilation
- IoT soil moisture integration, EnKF state updating, Sentinel-2 NDVI
- **Impact:** Precision agriculture market entry

### Phase 3 (Months 6-7): 2D/3D Vadose Zone
- 2D axisymmetric solver, tile drainage, pesticide plume modeling
- **Impact:** Differentiate vs HYDRUS ($3-12K/seat)

### Phase 4 (Months 8-9): Multi-Crop Rotation & Portfolios
- Crop sequences, 100+ field aggregation, variable-rate prescriptions
- **Impact:** Enterprise sales to agribusinesses

### Phase 5 (Months 10-12): AI Yield Forecasting
- LSTM yield models, surrogate emulators, economic optimization
- **Impact:** Premium pricing for ML-powered recommendations

