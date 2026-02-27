# 84. WaveForge — Coastal and Ocean Wave Modeling Platform

## Implementation Plan

**MVP Scope:** Browser-based bathymetry import (GEBCO, GeoTIFF, XYZ with automatic elevation parsing and coordinate system detection) with automatic unstructured triangular mesh generation using Delaunay refinement and coastline-adaptive sizing, stationary spectral wave action balance solver on 2D unstructured grids implementing SWAN-equivalent physics (Komen wind input, Janssen whitecapping, Battjes-Janssen depth-induced breaking, JONSWAP bottom friction, DIA quadruplet interactions, LTA triad interactions) compiled to WebAssembly for domains ≤5,000 cells and server-side Rust-native GPU-accelerated solver for larger domains, automatic boundary condition extraction from ERA5 global wave reanalysis with temporal and spatial interpolation, interactive wave results visualization via Deck.gl with false-color Hs/Tp/direction fields and vector arrows overlaid on Mapbox bathymetry, wave climate statistics (Hs mean/P50/P90/P99, directional roses with 16 sectors, monthly/seasonal variability), basic extreme value analysis using Peaks Over Threshold (GPD distribution) and Annual Maxima (GEV distribution) with return period curves and 95% confidence intervals, PDF wave study report generation with location map, boundary conditions table, wave climate summary, and design wave parameters, S3 storage for bathymetry grids, mesh files, and ERA5 boundary data, PostgreSQL + PostGIS for project/user/simulation metadata, Stripe billing with three tiers (Free / Pro $129/mo / Advanced $349/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| Wave Solver | Rust (native + WASM) | Spectral action balance on unstructured FV mesh with GPU acceleration via wgpu |
| WASM Compilation | `wasm-pack` + `wasm-bindgen` | Client-side solver for domains ≤5,000 cells |
| Mesh Generator | Rust (native + WASM) | Delaunay triangulation via `delaunator-rs`, Triangle quality refinement |
| Extreme Value Analysis | Python 3.12 (FastAPI) | SciPy for GPD/GEV fitting, NumPy for statistics, Pandas for time series |
| Database | PostgreSQL 16 + PostGIS | Projects, users, simulations, bathymetry metadata, wave climate stats |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | Bathymetry grids (NetCDF/GeoTIFF), ERA5 boundary data, mesh files, wave results |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| Map Visualization | Mapbox GL JS + Deck.gl | Bathymetry contours, mesh rendering, wave field visualization |
| Real-time | WebSocket (Axum) | Live simulation progress, convergence monitoring |
| Job Queue | Redis 7 + Tokio tasks | Server-side simulation job management, priority queue for paid plans |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | WASM bundle delivery, static frontend assets |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, solver performance, error tracking |
| CI/CD | GitHub Actions | WASM build pipeline, Rust tests, Docker image push |

### Key Architecture Decisions

1. **Hybrid client/server solver with WASM threshold at 5,000 cells**: Small domains (≤5,000 triangular cells, covering typical harbor studies, local beach segments, nearshore transects) run entirely in the browser via WASM for instant feedback with zero server cost. Larger domains (regional coastal stretches, multi-kilometer beaches, offshore wind farms) execute on server with GPU acceleration via wgpu compute shaders for 10-100x speedup on spectral integration. The 5,000-cell threshold covers 90% of engineering feasibility studies and educational use cases.

2. **Spectral action balance on unstructured mesh rather than structured grid**: Unstructured triangular meshes allow adaptive resolution — fine near coastlines, structures, and bathymetric features, coarse offshore — reducing total cell count by 3-5x compared to SWAN's structured curvilinear grids. This makes WASM execution feasible for practical domains. The finite volume discretization on unstructured grids follows the approach of WW3's unstructured implementation but simplified for 2D (no 3D propagation).

3. **GPU acceleration via wgpu for server-side solver**: The spectral integration loop (16-36 direction bins × 25-40 frequency bins × N cells) is embarrassingly parallel and ideal for GPU compute shaders. Using wgpu (portable WebGPU API in Rust) allows deployment on AWS instances with NVIDIA (p4d/p5), AMD (g4ad), or Apple Silicon GPUs without CUDA lock-in. A 10,000-cell domain with 24 directions × 30 frequencies (7.2M spectral bins) solves in 5-10 seconds on GPU vs. 3-5 minutes on CPU.

4. **ERA5 boundary conditions via S3 caching layer**: ERA5 global wave reanalysis (0.5° resolution, 1979-present) provides physically realistic boundary conditions for regional simulations. Pre-downloading ERA5 data for the user's domain bounding box and time range (via Copernicus CDS API) and caching in S3 eliminates 30-60s latency per simulation. Spatial and temporal interpolation to the mesh boundary is performed by the solver initialization code.

5. **Python FastAPI microservice for extreme value analysis**: EVA requires SciPy's statistical distributions and optimization routines which have no mature Rust equivalent. A separate FastAPI service handles GPD/GEV fitting, return period calculations, and confidence intervals, called via HTTP from the main Axum API. This keeps the Rust codebase focused on the wave solver while leveraging Python's scientific stack where it excels.

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

-- Organization Members
CREATE TABLE org_members (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    org_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role TEXT NOT NULL DEFAULT 'member',
    joined_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(org_id, user_id)
);

-- Projects (bathymetry + mesh + simulation workspace)
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    bounding_box GEOMETRY(POLYGON, 4326),
    bathymetry_url TEXT,
    bathymetry_metadata JSONB,
    mesh_url TEXT,
    mesh_metadata JSONB,
    settings JSONB DEFAULT '{}',
    is_public BOOLEAN NOT NULL DEFAULT false,
    forked_from UUID REFERENCES projects(id) ON DELETE SET NULL,
    thumbnail_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);
CREATE INDEX projects_bbox_idx ON projects USING GIST(bounding_box);

-- Simulations (wave model runs)
CREATE TABLE simulations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    simulation_type TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending',
    execution_mode TEXT NOT NULL DEFAULT 'wasm',
    boundary_conditions JSONB NOT NULL,
    physics_settings JSONB NOT NULL,
    mesh_cell_count INTEGER NOT NULL DEFAULT 0,
    spectral_resolution JSONB,
    results_url TEXT,
    results_summary JSONB,
    error_message TEXT,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    wall_time_ms INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX sims_project_idx ON simulations(project_id);
CREATE INDEX sims_status_idx ON simulations(status);

-- Server Simulation Jobs
CREATE TABLE simulation_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    simulation_id UUID NOT NULL REFERENCES simulations(id) ON DELETE CASCADE,
    worker_id TEXT,
    execution_mode TEXT DEFAULT 'cpu',
    cores_allocated INTEGER DEFAULT 4,
    memory_mb INTEGER DEFAULT 8192,
    gpu_type TEXT,
    priority INTEGER DEFAULT 0,
    progress_pct REAL DEFAULT 0.0,
    convergence_data JSONB DEFAULT '[]',
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Wave Climate Statistics
CREATE TABLE wave_climate_stats (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    location GEOMETRY(POINT, 4326),
    data_source TEXT NOT NULL,
    time_period_start DATE NOT NULL,
    time_period_end DATE NOT NULL,
    statistics JSONB NOT NULL,
    directional_rose JSONB,
    monthly_stats JSONB,
    extreme_values JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Usage Tracking
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,
    quantity REAL NOT NULL,
    metadata JSONB,
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
    pub bathymetry_url: Option<String>,
    pub bathymetry_metadata: Option<serde_json::Value>,
    pub mesh_url: Option<String>,
    pub mesh_metadata: Option<serde_json::Value>,
    pub settings: serde_json::Value,
    pub is_public: bool,
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
    pub boundary_conditions: serde_json::Value,
    pub physics_settings: serde_json::Value,
    pub mesh_cell_count: i32,
    pub spectral_resolution: Option<serde_json::Value>,
    pub results_url: Option<String>,
    pub results_summary: Option<serde_json::Value>,
    pub error_message: Option<String>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub wall_time_ms: Option<i32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct BoundaryConditions {
    pub source: BoundarySource,
    pub era5_params: Option<Era5Params>,
    pub manual_spectrum: Option<ManualSpectrum>,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum BoundarySource {
    Era5,
    WaveWatchIII,
    Manual,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct Era5Params {
    pub time_start: DateTime<Utc>,
    pub time_end: DateTime<Utc>,
    pub use_cached: bool,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct ManualSpectrum {
    pub hs: f64,
    pub tp: f64,
    pub gamma: f64,
    pub direction: f64,
    pub spread: f64,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct PhysicsSettings {
    pub wind_input: WindInputModel,
    pub whitecapping: WhitecappingModel,
    pub breaking: BreakingModel,
    pub bottom_friction: BottomFrictionModel,
    pub triads: bool,
    pub quadruplets: QuadrupletModel,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum WindInputModel {
    Komen,
    Janssen,
    St6,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum WhitecappingModel {
    Komen,
    Janssen,
    St6,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct BreakingModel {
    pub model_type: String,
    pub alpha: f64,
    pub gamma: f64,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum BottomFrictionModel {
    Jonswap,
    Collins,
    Madsen,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum QuadrupletModel {
    None,
    Dia,
}
```

---

## Solver Architecture Deep-Dive

### Governing Equations and Discretization

WaveForge implements the **spectral wave action balance equation**:

```
∂N/∂t + ∂(c_x N)/∂x + ∂(c_y N)/∂y + ∂(c_σ N)/∂σ + ∂(c_θ N)/∂θ = S_tot/σ

where:
  N(σ, θ) = E(σ, θ) / σ  (action density = energy density / frequency)
  c_x, c_y = group velocity components
  c_σ = frequency shifting rate
  c_θ = refraction rate
  S_tot = S_in + S_nl + S_ds + S_bot + S_br
```

**Key Source Terms:**

**Wind input (Komen):**
```
S_in = A·E, where A = ρ_air/ρ_water · max(0, cos(θ-θ_wind))² · (28·U₁₀/c_phase - 1) · σ
```

**Whitecapping (Komen):**
```
S_ds = -C_ds · (σ̄/σ_PM)² · (k̄/k)² · σ · E
C_ds = 4.5 × 10⁻⁵ (default)
```

**Depth-induced breaking (Battjes-Janssen):**
```
S_br = -D_br · E
D_br = (α/4) · (H_max/H_rms)² · Q_b · f_mean · (H_rms/d)²
H_max = γ · d  (γ ≈ 0.73)
```

**Bottom friction (JONSWAP):**
```
S_bot = -C_bot · σ²/g² · sinh(kd)⁻² · E
C_bot = 0.038 m²/s³
```

### Finite Volume Discretization

For each triangular cell i:
```
dN_i/dt + (1/A_i) · Σ_{edges j} F_ij · L_ij = S_tot,i / σ

F_ij = c_g · n_ij · N_upwind  (upwind flux)
```

Spectral space: N_dir directions (24-36) × N_freq frequencies (25-40) logarithmically spaced from 0.04 Hz to 1.0 Hz.

Time integration: Stationary mode uses Gauss-Seidel relaxation until convergence (<500 iterations typical).

### Client/Server/GPU Split

```
Domain → Cell count
    ├── ≤5,000 → WASM (browser, instant, zero cost)
    ├── 5,001-50,000 → Server CPU (multi-core parallelization)
    └── >50,000 → Server GPU (wgpu, 10-100x speedup)
```

---

## Architecture Deep-Dives

### 1. Simulation API Handler (Rust/Axum)

```rust
// src/api/handlers/simulation.rs

use axum::{extract::{Path, State, Json}, http::StatusCode};
use uuid::Uuid;
use crate::{db::models::*, state::AppState, auth::Claims, error::ApiError};

pub async fn create_simulation(
    State(state): State<AppState>,
    claims: Claims,
    Path(project_id): Path<Uuid>,
    Json(req): Json<CreateSimulationRequest>,
) -> Result<impl IntoResponse, ApiError> {
    // 1. Verify project ownership
    let project = sqlx::query_as!(
        Project,
        "SELECT * FROM projects WHERE id = $1 AND owner_id = $2",
        project_id, claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Project not found"))?;

    let mesh_metadata: serde_json::Value = project.mesh_metadata
        .ok_or(ApiError::BadRequest("Mesh required"))?;
    let mesh_cell_count: i32 = mesh_metadata["cell_count"].as_i64().unwrap() as i32;

    // 2. Check plan limits
    let user = sqlx::query_as!(User, "SELECT * FROM users WHERE id = $1", claims.user_id)
        .fetch_one(&state.db).await?;

    if user.plan == "free" && mesh_cell_count > 5_000 {
        return Err(ApiError::PlanLimit("Free plan: max 5,000 cells"));
    }

    // 3. Determine execution mode
    let execution_mode = if mesh_cell_count <= 5_000 {
        "wasm"
    } else if mesh_cell_count <= 50_000 || user.plan != "advanced" {
        "server"
    } else {
        "gpu"
    };

    // 4. Create simulation record
    let sim = sqlx::query_as!(
        Simulation,
        r#"INSERT INTO simulations
            (project_id, user_id, simulation_type, status, execution_mode,
             boundary_conditions, physics_settings, mesh_cell_count, spectral_resolution)
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9) RETURNING *"#,
        project_id, claims.user_id, req.simulation_type,
        if execution_mode == "wasm" { "ready" } else { "pending" },
        execution_mode,
        serde_json::to_value(&req.boundary_conditions)?,
        serde_json::to_value(&req.physics_settings)?,
        mesh_cell_count,
        serde_json::to_value(&req.spectral_resolution)?,
    )
    .fetch_one(&state.db).await?;

    // 5. Enqueue server/GPU job
    if execution_mode != "wasm" {
        let priority = match user.plan.as_str() {
            "advanced" => 20, "pro" => 10, _ => 0,
        };
        let job = sqlx::query!(
            "INSERT INTO simulation_jobs (simulation_id, execution_mode, priority) VALUES ($1, $2, $3)",
            sim.id, execution_mode, priority
        ).execute(&state.db).await?;

        let queue = if execution_mode == "gpu" { "sim:gpu" } else { "sim:cpu" };
        state.redis.zadd(queue, sim.id.to_string(), priority).await?;
    }

    Ok((StatusCode::CREATED, Json(sim)))
}
```

### 2. Wave Action Balance Solver Core (Rust)

```rust
// solver-core/src/action_balance.rs

use nalgebra::{DVector, DMatrix};
use crate::mesh::UnstructuredMesh;
use crate::spectrum::SpectrumGrid;

pub struct ActionBalanceSolver {
    pub mesh: UnstructuredMesh,
    pub spectrum_grid: SpectrumGrid,
    pub action: DMatrix<f64>,  // [n_cells × n_spectral_bins]
    pub energy: DMatrix<f64>,
    pub bathymetry: DVector<f64>,
}

pub struct SpectrumGrid {
    pub n_freq: usize,
    pub n_dir: usize,
    pub frequencies: Vec<f64>,
    pub directions: Vec<f64>,
    pub d_freq: Vec<f64>,
    pub d_dir: f64,
}

impl SpectrumGrid {
    pub fn new(n_freq: usize, n_dir: usize, f_min: f64, f_max: f64) -> Self {
        let log_fmin = f_min.ln();
        let log_fmax = f_max.ln();
        let d_logf = (log_fmax - log_fmin) / (n_freq as f64 - 1.0);

        let frequencies: Vec<f64> = (0..n_freq)
            .map(|i| (log_fmin + i as f64 * d_logf).exp())
            .collect();

        let d_freq: Vec<f64> = (0..n_freq).map(|i| {
            let f_low = if i == 0 { frequencies[0] / (frequencies[1]/frequencies[0]).sqrt() }
                else { (frequencies[i-1] * frequencies[i]).sqrt() };
            let f_high = if i == n_freq-1 { frequencies[n_freq-1] * (frequencies[n_freq-1]/frequencies[n_freq-2]).sqrt() }
                else { (frequencies[i] * frequencies[i+1]).sqrt() };
            f_high - f_low
        }).collect();

        let d_dir = 2.0 * std::f64::consts::PI / n_dir as f64;
        let directions: Vec<f64> = (0..n_dir).map(|i| i as f64 * d_dir).collect();

        Self { n_freq, n_dir, frequencies, directions, d_freq, d_dir }
    }
}

impl ActionBalanceSolver {
    pub fn solve_stationary(
        &mut self,
        boundary_conditions: &BoundaryConditions,
        max_iterations: usize,
        tol: f64,
    ) -> Result<(), SolverError> {
        self.apply_boundary_conditions(boundary_conditions)?;

        for iter in 0..max_iterations {
            let mut max_change = 0.0;

            for cell_idx in 0..self.mesh.n_cells() {
                for dir_idx in 0..self.spectrum_grid.n_dir {
                    for freq_idx in 0..self.spectrum_grid.n_freq {
                        let bin_idx = dir_idx * self.spectrum_grid.n_freq + freq_idx;
                        let old_action = self.action[(cell_idx, bin_idx)];
                        let new_action = self.compute_cell_action(cell_idx, freq_idx, dir_idx, boundary_conditions)?;
                        self.action[(cell_idx, bin_idx)] = new_action;
                        self.energy[(cell_idx, bin_idx)] = new_action * 2.0 * std::f64::consts::PI * self.spectrum_grid.frequencies[freq_idx];
                        max_change = max_change.max((new_action - old_action).abs());
                    }
                }
            }

            if iter % 10 == 0 { tracing::debug!("Iter {}: max_change = {:.3e}", iter, max_change); }
            if max_change < tol { return Ok(()); }
        }

        Err(SolverError::Convergence(format!("Failed to converge after {} iterations", max_iterations)))
    }

    fn compute_cell_action(&self, cell_idx: usize, freq_idx: usize, dir_idx: usize, bc: &BoundaryConditions) -> Result<f64, SolverError> {
        let cell = &self.mesh.cells[cell_idx];
        let depth = self.bathymetry[cell_idx];
        let sigma = 2.0 * std::f64::consts::PI * self.spectrum_grid.frequencies[freq_idx];
        let theta = self.spectrum_grid.directions[dir_idx];
        let bin_idx = dir_idx * self.spectrum_grid.n_freq + freq_idx;

        let k = solve_dispersion_relation(sigma, depth);
        let c_g = 0.5 * sigma / k * (1.0 + 2.0 * k * depth / (2.0 * k * depth).sinh());
        let c_gx = c_g * theta.sin();
        let c_gy = c_g * theta.cos();

        let mut flux_sum = 0.0;
        for edge_idx in &cell.edge_indices {
            let edge = &self.mesh.edges[*edge_idx];
            let normal = edge.outward_normal(cell_idx);
            let flux = c_gx * normal.0 + c_gy * normal.1;

            if flux > 0.0 {
                flux_sum += flux * edge.length * self.action[(cell_idx, bin_idx)];
            } else {
                let neighbor_action = if let Some(neighbor_idx) = edge.neighbor_cell(cell_idx) {
                    self.action[(neighbor_idx, bin_idx)]
                } else {
                    bc.get_action(freq_idx, dir_idx)
                };
                flux_sum += flux * edge.length * neighbor_action;
            }
        }

        let source = self.compute_source_terms(cell_idx, freq_idx, dir_idx, depth);
        let new_action = (source / sigma * cell.area - flux_sum) / (cell.area + 1e-12);
        Ok(new_action.max(0.0))
    }

    pub fn compute_integrated_params(&self) -> Vec<IntegratedWaveParams> {
        let mut result = Vec::with_capacity(self.mesh.n_cells());

        for cell_idx in 0..self.mesh.n_cells() {
            let mut m0 = 0.0;
            let mut m1 = 0.0;
            let mut sin_sum = 0.0;
            let mut cos_sum = 0.0;

            for dir_idx in 0..self.spectrum_grid.n_dir {
                let theta = self.spectrum_grid.directions[dir_idx];
                for freq_idx in 0..self.spectrum_grid.n_freq {
                    let bin_idx = dir_idx * self.spectrum_grid.n_freq + freq_idx;
                    let energy = self.energy[(cell_idx, bin_idx)];
                    let freq = self.spectrum_grid.frequencies[freq_idx];
                    let de = energy * self.spectrum_grid.d_freq[freq_idx] * self.spectrum_grid.d_dir;
                    m0 += de;
                    m1 += freq * de;
                    sin_sum += de * theta.sin();
                    cos_sum += de * theta.cos();
                }
            }

            let hs = 4.0 * m0.sqrt();
            let tm01 = if m1 > 0.0 { m0 / m1 } else { 0.0 };
            let mean_dir = sin_sum.atan2(cos_sum).to_degrees();

            let mut peak_freq = 0.0;
            let mut max_energy = 0.0;
            for freq_idx in 0..self.spectrum_grid.n_freq {
                let mut freq_energy = 0.0;
                for dir_idx in 0..self.spectrum_grid.n_dir {
                    let bin_idx = dir_idx * self.spectrum_grid.n_freq + freq_idx;
                    freq_energy += self.energy[(cell_idx, bin_idx)];
                }
                if freq_energy > max_energy {
                    max_energy = freq_energy;
                    peak_freq = self.spectrum_grid.frequencies[freq_idx];
                }
            }
            let tp = if peak_freq > 0.0 { 1.0 / peak_freq } else { 0.0 };

            result.push(IntegratedWaveParams { hs, tp, tm01, mean_direction: mean_dir });
        }
        result
    }
}

#[derive(Debug, Clone)]
pub struct IntegratedWaveParams {
    pub hs: f64,
    pub tp: f64,
    pub tm01: f64,
    pub mean_direction: f64,
}

fn solve_dispersion_relation(sigma: f64, depth: f64) -> f64 {
    let g = 9.81;
    let mut k = sigma * sigma / g;
    for _ in 0..5 {
        let f = sigma * sigma - g * k * (k * depth).tanh();
        let df = -g * ((k * depth).tanh() + k * depth / (k * depth).cosh().powi(2));
        k -= f / df;
    }
    k
}
```

### 3. Mesh Visualization (React + Deck.gl)

```typescript
// frontend/src/components/WaveResultsViewer.tsx

import { useState, useMemo } from 'react';
import DeckGL from '@deck.gl/react';
import { PolygonLayer, IconLayer } from '@deck.gl/layers';
import { Map } from 'react-map-gl';
import { scaleSequential } from 'd3-scale';
import { interpolateTurbo } from 'd3-scale-chromatic';

interface MeshCell {
  id: number;
  vertices: [number, number][];
  center: [number, number];
  hs: number;
  tp: number;
  direction: number;
}

export function WaveResultsViewer({ simulationId }: { simulationId: string }) {
  const [results, setResults] = useState<any>(null);
  const [colorVar, setColorVar] = useState<'hs' | 'tp'>('hs');

  const colorScale = useMemo(() => {
    if (!results) return null;
    const values = results.cells.map((c: MeshCell) => c[colorVar]);
    return scaleSequential(interpolateTurbo).domain([Math.min(...values), Math.max(...values)]);
  }, [results, colorVar]);

  const layers = useMemo(() => {
    if (!results || !colorScale) return [];
    return [
      new PolygonLayer({
        id: 'mesh',
        data: results.cells,
        getPolygon: (d: MeshCell) => d.vertices,
        getFillColor: (d: MeshCell) => {
          const rgb = parseColor(colorScale(d[colorVar]));
          return [...rgb, 180];
        },
        pickable: true,
      }),
      new IconLayer({
        id: 'vectors',
        data: results.cells.filter((_: any, i: number) => i % 5 === 0),
        getPosition: (d: MeshCell) => d.center,
        getIcon: () => 'arrow',
        getSize: 20,
        getAngle: (d: MeshCell) => -d.direction,
      }),
    ];
  }, [results, colorScale, colorVar]);

  return (
    <DeckGL
      initialViewState={{ longitude: -122.4, latitude: 37.8, zoom: 12 }}
      controller={true}
      layers={layers}
    >
      <Map mapboxAccessToken={import.meta.env.VITE_MAPBOX_TOKEN} mapStyle="mapbox://styles/mapbox/satellite-v9" />
    </DeckGL>
  );
}

function parseColor(colorStr: string): [number, number, number] {
  const match = colorStr.match(/rgb\((\d+),\s*(\d+),\s*(\d+)\)/);
  return match ? [parseInt(match[1]), parseInt(match[2]), parseInt(match[3])] : [128, 128, 128];
}
```

### 4. Server-Side Worker with GPU Acceleration

```rust
// src/workers/simulation_worker.rs

use sqlx::PgPool;
use redis::AsyncCommands;
use aws_sdk_s3::Client as S3Client;
use uuid::Uuid;
use crate::solver::action_balance::ActionBalanceSolver;
use crate::solver::gpu::GpuSolverBackend;

pub struct SimulationWorker {
    db: PgPool,
    redis: redis::Client,
    s3: S3Client,
    gpu_backend: Option<GpuSolverBackend>,
}

impl SimulationWorker {
    pub async fn run(&self, queue: &str) -> anyhow::Result<()> {
        let mut conn = self.redis.get_async_connection().await?;
        tracing::info!("Worker started on queue: {}", queue);

        loop {
            let job_data: Option<(String, f64)> = conn.bzpopmax(queue, 30.0).await?;
            let Some((job_id_str, _priority)) = job_data else { continue };
            let job_id: Uuid = job_id_str.parse()?;

            if let Err(e) = self.process_job(job_id).await {
                tracing::error!("Job {job_id} failed: {e}");
                self.mark_failed(job_id, &e.to_string()).await?;
            }
        }
    }

    async fn process_job(&self, job_id: Uuid) -> anyhow::Result<()> {
        let job = sqlx::query!("UPDATE simulation_jobs SET started_at = NOW(), worker_id = $2 WHERE id = $1 RETURNING simulation_id", job_id, hostname::get()?.to_string_lossy().to_string())
            .fetch_one(&self.db).await?;

        let sim = sqlx::query!("UPDATE simulations SET status = 'running', started_at = NOW() WHERE id = $1 RETURNING *", job.simulation_id)
            .fetch_one(&self.db).await?;

        let project = sqlx::query!("SELECT * FROM projects WHERE id = $1", sim.project_id)
            .fetch_one(&self.db).await?;

        let mesh = load_mesh_from_s3(&self.s3, &project.mesh_url.unwrap()).await?;
        let bathymetry = mesh.extract_bathymetry();

        let boundary_conditions: BoundaryConditions = serde_json::from_value(sim.boundary_conditions)?;
        let physics_settings: PhysicsSettings = serde_json::from_value(sim.physics_settings)?;
        let spectral_resolution: SpectralResolution = serde_json::from_value(sim.spectral_resolution.unwrap())?;

        let spectrum_grid = SpectrumGrid::new(
            spectral_resolution.n_frequencies as usize,
            spectral_resolution.n_directions as usize,
            spectral_resolution.f_min,
            spectral_resolution.f_max,
        );

        let use_gpu = sim.execution_mode == "gpu" && self.gpu_backend.is_some();
        let mut solver = if use_gpu {
            ActionBalanceSolver::new_with_gpu(mesh, bathymetry, spectrum_grid, physics_settings.into(), self.gpu_backend.as_ref().unwrap().clone())
        } else {
            ActionBalanceSolver::new(mesh, bathymetry, spectrum_grid, physics_settings.into())
        };

        solver.solve_stationary(&boundary_conditions, 500, 1e-4)?;
        let wave_params = solver.compute_integrated_params();

        let results_netcdf = write_netcdf(&solver.mesh, &wave_params, &solver.energy, &spectrum_grid)?;
        let s3_key = format!("results/{}/{}.nc", sim.project_id, sim.id);
        self.s3.put_object().bucket("waveforge-results").key(&s3_key).body(results_netcdf.into()).send().await?;

        let hs_values: Vec<f64> = wave_params.iter().map(|p| p.hs).collect();
        let hs_mean = hs_values.iter().sum::<f64>() / hs_values.len() as f64;
        let hs_max = hs_values.iter().cloned().fold(0.0, f64::max);

        sqlx::query!("UPDATE simulations SET status = 'completed', results_url = $2, results_summary = $3, completed_at = NOW() WHERE id = $1",
            sim.id, format!("s3://waveforge-results/{}", s3_key), serde_json::json!({"hs_mean": hs_mean, "hs_max": hs_max}))
            .execute(&self.db).await?;

        Ok(())
    }

    async fn mark_failed(&self, job_id: Uuid, error: &str) -> anyhow::Result<()> {
        let sim_id: Uuid = sqlx::query_scalar!("SELECT simulation_id FROM simulation_jobs WHERE id = $1", job_id).fetch_one(&self.db).await?;
        sqlx::query!("UPDATE simulations SET status = 'failed', error_message = $2, completed_at = NOW() WHERE id = $1", sim_id, error).execute(&self.db).await?;
        Ok(())
    }
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–5)

**Day 1: Project initialization and Rust backend scaffold**
- `cargo init waveforge-api` with Axum, Tokio, SQLx, AWS SDK S3, Redis
- `src/main.rs` — Axum app with CORS middleware, tracing subscriber, graceful shutdown
- `src/config.rs` — Environment-based config (DATABASE_URL, JWT_SECRET, S3_BUCKET, REDIS_URL, MAPBOX_TOKEN, ERA5_API_KEY)
- `src/state.rs` — AppState struct holding PgPool, Redis client, S3Client, broadcaster
- `src/error.rs` — ApiError enum with IntoResponse for HTTP error handling
- `Dockerfile` — Multi-stage build with Rust builder and minimal runtime
- `docker-compose.yml` — PostgreSQL 16 + PostGIS 3.4, Redis 7, MinIO (S3-compatible local storage)

**Day 2: Database schema and migrations**
- `migrations/001_initial.sql` — All 9 tables: users, organizations, org_members, projects, simulations, simulation_jobs, wave_climate_stats, bathymetry_datasets (optional), usage_records
- PostGIS extension for GEOMETRY columns (bounding_box, location)
- `src/db/mod.rs` — Database pool initialization with connection pooling (max 20 connections)
- `src/db/models.rs` — All SQLx structs with FromRow derives, serde Serialize/Deserialize
- Run `sqlx migrate run` to apply schema
- Seed script for sample GEBCO bathymetry metadata (global coverage)

**Day 3: Authentication system**
- `src/auth/mod.rs` — JWT token generation (claims: user_id, email, plan, exp), validation middleware extracting Claims from Authorization header
- `src/auth/oauth.rs` — Google and GitHub OAuth 2.0 flow: redirect to provider, callback handler, exchange code for token, fetch user info
- `src/api/handlers/auth.rs` — Register (email/password, bcrypt hash), login (verify password, issue JWT), OAuth callback (create/update user, issue JWT), refresh token (extend expiry), me endpoint (get current user)
- Password hashing with bcrypt cost factor 12 (balanced security/performance)
- JWT with 24h access token + 30d refresh token stored in HTTP-only cookie
- Auth middleware extracts Claims, returns 401 Unauthorized if missing/invalid

**Day 4: User and project CRUD**
- `src/api/handlers/users.rs` — Get profile (by ID or "me"), update profile (name, avatar), delete account (cascade delete projects)
- `src/api/handlers/projects.rs` — Create project (name, description, bounding box), list projects (with pagination), get project (by ID), update project (metadata, settings), delete project, fork project (copy bathymetry/mesh)
- `src/api/handlers/orgs.rs` — Create organization, invite member (send email), list members, update member role, remove member
- Project bounding box stored as PostGIS POLYGON geometry for spatial queries
- Authorization: check ownership (owner_id) or org membership (org_members table)
- Integration tests: register user, create project, verify unauthorized access blocked

**Day 5: S3 integration and file upload**
- `src/storage/mod.rs` — S3 client wrapper with `upload_file`, `download_file`, `object_exists`, `generate_presigned_url` helpers
- `src/api/handlers/upload.rs` — Multipart file upload endpoint for bathymetry (GeoTIFF, XYZ ASCII, NetCDF)
- File validation: check MIME type, max size (1 GB), coordinate system detection (EPSG:4326 required)
- Metadata extraction: resolution (arc-seconds or meters), vertical datum, bounding box, file size
- Store metadata in `projects.bathymetry_metadata` JSONB column
- S3 key naming: `bathymetry/{project_id}/{filename}`
- Unit tests: upload GeoTIFF, parse XYZ, validate NetCDF CF compliance

### Phase 2 — Mesh Generation Core (Days 6–10)

**Day 6: Delaunay triangulation and mesh data structures**
- `mesh-core/` — New Rust workspace member (shared between native and WASM), `lib`, `rlib` crate types
- `mesh-core/src/types.rs` — Mesh, Cell (vertices, edges, area, center, is_boundary), Edge (nodes, cells, length, midpoint), Node (position, depth) structs
- `mesh-core/src/delaunay.rs` — Delaunay triangulation using `delaunator` crate (incremental algorithm, O(n log n))
- Coastline detection from bathymetry: identify land/water boundary (depth = 0 contour), extract polylines
- Edge connectivity: build adjacency lists (cell → edges, edge → cells)
- Unit tests: simple square domain (4 corners → 2 triangles), L-shaped domain, circular island

**Day 7: Mesh refinement and quality optimization**
- `mesh-core/src/refinement.rs` — Ruppert's Delaunay refinement algorithm: insert circumcenter of skinny triangles (angle < 20°), split encroached edges
- Coastline-adaptive sizing function: target resolution = f(distance_to_coast), fine near shore (10-50m), coarse offshore (200-1000m)
- Bathymetric feature detection: identify shoals, channels, ridges via gradient analysis, refine locally
- Quality metrics: minimum angle, maximum angle, aspect ratio (longest edge / shortest edge), skewness
- Laplacian smoothing with boundary preservation: iterate node repositioning to average of neighbors (5-10 iterations)
- Tests: refine mesh to target resolution field, verify quality metrics improved, check boundary preservation

**Day 8: Bathymetry interpolation onto mesh**
- `mesh-core/src/interpolation.rs` — Natural neighbor interpolation (Sibson weights) for scattered XYZ data
- Bilinear interpolation for gridded bathymetry (GEBCO GeoTIFF, NetCDF grids)
- Depth assignment: interpolate to mesh nodes, compute cell-center depths via averaging
- Land cell handling: mark cells with depth < 0 as land, exclude from wave solver
- Wet/dry classification: threshold depth (e.g., 0.1m) for numerical stability
- Tests: interpolate GEBCO 15 arc-second grid onto mesh, check depth conservation (total volume), verify smooth gradients

**Day 9: Mesh serialization and S3 storage**
- `mesh-core/src/io.rs` — Custom binary mesh format: header (version, n_nodes, n_cells, n_edges), node coords (f64), element connectivity (u32), depths (f32)
- NetCDF export: CF-compliant with variables `node_x`, `node_y`, `depth`, `element_node_connectivity`, attributes (CRS, units)
- GeoJSON export for web visualization: FeatureCollection with Polygon features per cell
- S3 upload/download with gzip compression (typical compression ratio 5-10x for mesh data)
- Tests: serialize/deserialize round-trip (verify bit-exact), NetCDF compliance check via `ncdump`, GeoJSON validation

**Day 10: Mesh API endpoints**
- `src/api/handlers/mesh.rs` — `POST /projects/:id/mesh/generate` (enqueue mesh generation job), `GET /projects/:id/mesh` (get metadata), `GET /projects/:id/mesh/download` (presigned S3 URL)
- Mesh generation job: asynchronous via Redis queue (can take 10-60s for large domains with 50K+ cells)
- Job progress tracking: triangulation (20%) → refinement (60%) → interpolation (80%) → storage (100%)
- WebSocket progress updates: broadcast to frontend every 5 seconds with current stage and percentage
- Mesh metadata stored in `projects.mesh_metadata`: `cell_count`, `node_count`, `min_resolution`, `max_resolution`, `min_depth`, `max_depth`, `quality_min_angle`
- Integration tests: generate mesh for 1km × 1km domain, verify cell count ~1000-5000, download mesh file, parse NetCDF

### Phase 3 — Wave Solver Core (Days 11–21)

**Day 11: Spectral grid and dispersion relation**
- `solver-core/` — New Rust workspace member (shared between native and WASM)
- `solver-core/src/spectrum.rs` — SpectrumGrid struct: logarithmic frequency spacing (f_min=0.04 Hz to f_max=1.0 Hz, typically 25-30 bins), uniform directional spacing (24-36 bins covering 0-360°), frequency bin widths for integration
- `solver-core/src/dispersion.rs` — Solve dispersion relation σ² = gk·tanh(kd) via Newton-Raphson iteration (5 iterations sufficient for convergence to 10⁻⁸ relative error)
- `solver-core/src/kinematics.rs` — Group velocity c_g = ∂σ/∂k, phase velocity c = σ/k, wavenumber k(σ, d)
- Deep water approximation (kd > π): c_g ≈ c/2, k ≈ σ²/g
- Shallow water approximation (kd < π/10): c_g ≈ c ≈ √(gd), k ≈ σ/√(gd)
- Tests: verify dispersion relation accuracy (compare to analytical), check deep/shallow water limits, benchmark performance (1M evaluations < 100ms)

**Day 12: Source term framework and wind input**
- `solver-core/src/source_terms/mod.rs` — SourceTerms trait with `compute(energy_spectrum, grid, depth, freq_idx, dir_idx) -> f64` method, SourceTermConfig for parameters
- `solver-core/src/source_terms/wind_input.rs` — Komen wind input: linear + exponential growth, depends on cos(θ - θ_wind)², wind speed U₁₀ at 10m, wave phase speed; Janssen wind input: couples with wave-induced stress, iterative solution
- Wind field input: uniform wind (speed, direction) or spatially varying (interpolate from grid)
- Drag coefficient C_d parameterization: C_d = (0.8 + 0.065·U₁₀) × 10⁻³
- Tests: wind input growth rate for monochromatic wave (compare to Miles theory), verify directional dependence (cosine-squared), check fetch-limited growth curve

**Day 13: Depth-induced breaking**
- `solver-core/src/source_terms/breaking.rs` — Battjes-Janssen model: dissipation proportional to fraction of breaking waves Q_b, breaker height H_max = γ·d (γ = 0.73 default), iterative solution for Q_b from Rayleigh distribution; Thornton-Guza model: uses bore dissipation analogy
- Breaker index parameterization: constant γ (0.6-0.8) or depth-dependent γ(d) for surf zone
- Energy dissipation rate D_br integrated over spectrum
- Tests: breaking dissipation rate vs. analytical bore dissipation, verify Hs/d ratio in shallow water ≈ 0.4-0.5, compare to laboratory data (Battjes & Janssen 1978)

**Day 14: Bottom friction**
- `solver-core/src/source_terms/bottom_friction.rs` — JONSWAP friction: C_bot = 0.038 m²/s³, frequency-dependent (higher freq = more dissipation); Collins friction: includes near-bottom orbital velocity; Madsen friction: full boundary layer model with wave-current interaction
- Friction coefficient depends on sediment type: sand (0.01-0.02), gravel (0.02-0.04), rock (0.04-0.10)
- Orbital velocity amplitude at bottom: u_orb = σ·A / sinh(kd), where A is wave amplitude
- Tests: friction dissipation rate for monochromatic wave, verify frequency dependence (∝ σ²), compare to SWAN test cases

**Day 15: Nonlinear interactions**
- `solver-core/src/source_terms/triads.rs` — Lumped Triad Approximation (LTA) for shallow water: energy transfer from peak to harmonics and subharmonics, parameterized interaction coefficients (Eldeberky & Battjes 1995)
- `solver-core/src/source_terms/quadruplets.rs` — Discrete Interaction Approximation (DIA) for deep water: energy transfer among four waves satisfying resonance conditions, parameterized from exact Boltzmann integral (Hasselmann et al. 1985)
- Interaction coefficient tables precomputed for efficiency
- Tests: triad energy transfer for bichromatic waves (compare to exact), quadruplet peak downshift in fetch-limited growth (compare to observations), verify energy conservation

**Day 16: Action balance geographic propagation**
- `solver-core/src/action_balance.rs` — ActionBalanceSolver struct: mesh, spectrum_grid, action DMatrix [n_cells × n_bins], energy DMatrix, bathymetry DVector
- Finite volume discretization: for each cell, compute flux over edges using upwind scheme
- Edge-based flux computation: F_ij = c_g · n_ij · N_upwind, where n_ij is outward normal, N_upwind from upstream cell
- CFL stability condition: Δt ≤ min(cell_area / (c_g · perimeter)) for explicit time-stepping
- Tests: wave propagation on uniform rectangular grid (compare to analytical), refraction over parallel depth contours (compare to Snell's law), reflection at boundaries

**Day 17: Spectral integration and relaxation**
- Integration of source terms over spectrum: S_tot = Σ_{σ,θ} S(σ,θ) · Δσ · Δθ
- Gauss-Seidel relaxation for stationary solver: sweep over cells and spectral bins, update action = f(fluxes, source), iterate until convergence
- Convergence monitoring: max absolute change per iteration, residual norm, iteration count
- Convergence criteria: max_change < tol (default 1e-4) or max_iterations reached (default 500)
- Performance optimization: vectorize spectral loop, parallelize over cells (Rayon for CPU, wgpu for GPU)
- Tests: stationary solution for uniform wind over deep water (compare to JONSWAP fetch-limited spectrum), verify convergence rate (linear in iteration count)

**Day 18: Boundary conditions**
- `solver-core/src/boundary.rs` — BoundaryConditionProvider trait: `get_action(freq_idx, dir_idx) -> f64` method
- Manual spectrum: JONSWAP spectrum S(f) = α·g²/(2π)⁴ · f⁻⁵ · exp(-1.25·(f_p/f)⁴) · γ^exp, with directional spreading D(θ) = cos^s((θ-θ_mean)/2) where s ≈ 10-20
- Sponge layer for open boundaries: gradually increase dissipation near boundary to absorb outgoing energy (prevent spurious reflections)
- Nested grid capability: extract boundary from coarser parent grid, interpolate spatially and spectrally
- Tests: apply manual JONSWAP BC, verify spectrum shape (peak frequency, peak enhancement), check boundary flux conservation

**Day 19: ERA5 boundary condition integration**
- `src/era5/mod.rs` — ERA5 data fetcher via Copernicus Climate Data Store (CDS) API: authenticate with API key, request wave parameters (significant wave height, mean wave period, mean wave direction) for bounding box and time range
- Download ERA5 hourly wave data (0.5° grid, ~50 km resolution) and cache in S3 (keyed by domain bounds + time range)
- Spatial interpolation to mesh boundary nodes: bilinear interpolation from ERA5 grid
- Temporal interpolation: linear interpolation between hourly time steps
- Spectrum reconstruction from ERA5 bulk parameters: assume JONSWAP spectrum, directional spreading, reconstruct 2D spectrum E(f, θ)
- Tests: download ERA5 sample for North Pacific, interpolate to test boundary, verify spectrum reconstruction matches bulk parameters (Hs, Tp, direction)

**Day 20: Integrated wave parameters**
- `solver-core/src/parameters.rs` — Compute bulk wave parameters from 2D spectrum:
  - Zeroth moment m₀ = ∫∫ E(f,θ) df dθ, significant wave height Hs = 4√m₀
  - First moment m₁ = ∫∫ f·E(f,θ) df dθ, mean period Tm01 = m₀/m₁
  - Second moment m₂ = ∫∫ f²·E(f,θ) df dθ, zero-crossing period Tm02 = √(m₀/m₂)
  - Peak frequency f_p (frequency with maximum energy), peak period Tp = 1/f_p
  - Directional moments: mean direction θ_mean = atan2(∫∫ E·sin(θ), ∫∫ E·cos(θ)), directional spreading σ_θ
- Spectral bandwidth: ν = √(m₀·m₂/m₁² - 1), narrow-band (ν < 0.5) vs broad-band (ν > 0.8)
- Tests: compute parameters from analytical JONSWAP spectrum, verify Hs = 4√m₀, check peak frequency detection, compare mean vs peak period

**Day 21: Solver integration tests and benchmarks**
- End-to-end solver tests: (1) fetch-limited wave growth under uniform wind (compare to JONSWAP relations), (2) depth-limited breaking on sloping beach (compare to laboratory data), (3) refraction over shoal (compare to Snell's law), (4) wave diffraction around breakwater (compare to analytical)
- Compare results to SWAN benchmarks: SWASH test cases for wave transformation
- Numerical accuracy: grid convergence study (refine mesh, verify solution convergence)
- Performance benchmarks: 1K cells (CPU: 5s, WASM: 10s), 5K cells (CPU: 20s, WASM: 60s), 10K cells (CPU: 60s, GPU: 5s)
- Memory usage: typical 50-100 MB for 5K cells × 24 directions × 30 frequencies (3.6M spectral bins)

### Phase 4 — WASM Build + Frontend Visualization (Days 22–28)

**Day 22: WASM solver compilation**
- `solver-wasm/` — New workspace member for WASM target, `Cargo.toml` with `crate-type = ["cdylib"]`
- `solver-wasm/Cargo.toml` — Dependencies: `wasm-bindgen = "0.2"`, `serde-wasm-bindgen`, `js-sys`, `web-sys` (for console logging), `solver-core` (local dep)
- `solver-wasm/src/lib.rs` — WASM entry points: `init_solver(mesh_json, bathymetry_array, bc_json, physics_json, spectral_json) -> SolverHandle`, `run_iteration(handle) -> bool` (returns true if converged), `get_results(handle) -> ResultsJson`
- Compile-time feature flags: `#[cfg(not(target_arch = "wasm32"))]` to exclude I/O code (S3, ERA5, file system) from WASM build
- Memory management: use `wasm-bindgen` linear memory, pass large arrays (mesh coords, bathymetry) as typed arrays (Float64Array, Uint32Array) to avoid JSON parsing overhead
- Build pipeline: `wasm-pack build --target web --release`, generates `pkg/` with `.wasm`, `.js`, `.ts` bindings
- Optimization: `opt-level = "z"` (optimize for size), `lto = true`, `codegen-units = 1`, `wasm-opt -Oz` post-processing (reduces WASM from ~3 MB to ~800 KB)

**Day 23: WASM integration in frontend**
- `frontend/src/solver/wasmSolver.ts` — Load WASM module: `import init, { init_solver, run_iteration, get_results } from './pkg/waveforge_solver_wasm'`
- Initialize WASM: `await init()` (loads `.wasm` file, compiles, instantiates)
- TypeScript wrapper class `WasmSolver`: constructor loads mesh/bathymetry, `solve()` method runs iterations with progress callback, `getResults()` parses JSON to typed objects
- Pass mesh from API: fetch mesh as JSON, convert node coords to Float64Array, element connectivity to Uint32Array
- Receive wave results: parse ResultsJson with `hs`, `tp`, `direction` arrays (length = n_cells)
- Progress callback: call user-provided callback every 10 iterations with current iteration count and max_change
- Error handling: catch Rust panics (propagated as JS exceptions), display user-friendly error messages
- Tests: load WASM in Node.js (using `@assemblyscript/loader`), run solver on test mesh, verify results match native Rust

**Day 24: Mapbox + Deck.gl setup**
- `frontend/src/components/MapView.tsx` — Mapbox GL JS map with satellite basemap (`mapbox://styles/mapbox/satellite-v9`)
- Deck.gl overlay: `<DeckGL>` component wraps map, renders layers on top
- Bathymetry contour layer: LineLayer or PathLayer from bathymetry depth contours (generated from GeoTIFF)
- Mesh boundary layer: PathLayer for mesh outline (debugging)
- Camera controls: pan (click-drag), zoom (scroll wheel, pinch), pitch (right-click-drag for 3D tilt), bearing (Ctrl+drag to rotate)
- Initial view state: auto-zoom to project bounding box (compute lat/lon center and zoom level from bounds)
- Mapbox access token: store in `.env.local`, inject via `import.meta.env.VITE_MAPBOX_TOKEN`
- Tests: render map with mock project bounds, verify camera centering, test pan/zoom interactions

**Day 25: Wave results visualization**
- `frontend/src/components/WaveResultsViewer.tsx` — Deck.gl PolygonLayer for mesh cells
- Color mapping: map Hs/Tp/direction to color using D3 color scales (Turbo for Hs, Cividis for Tp, HSL for direction)
- Color scale computation: compute min/max of selected variable across all cells, create scale with `scaleSequential(interpolateTurbo).domain([min, max])`
- PolygonLayer props: `data = results.cells`, `getPolygon = (d) => d.vertices` (array of [lon, lat]), `getFillColor = (d) => colorScale(d.hs)` (returns [r, g, b, a])
- IconLayer for wave direction vectors: arrow icons (PNG sprite), `getPosition = (d) => d.center`, `getAngle = (d) => -d.direction` (negative for CW from N), `getSize = 20` pixels
- Vector subsampling: display arrows for every 5th or 10th cell (controlled by slider) to avoid clutter
- Tooltip on hover: use Deck.gl `getTooltip` callback, display cell info (Hs, Tp, direction, depth) in floating div
- Tests: render sample results with 100 cells, verify color scale, check tooltips appear on hover

**Day 26: Interactive controls and layer toggles**
- Variable selector: radio buttons or dropdown for Hs, Tp, direction (updates color scale and re-renders)
- Vector density control: slider (1-100%) to control subsampling rate for direction arrows
- Layer toggles: checkboxes for bathymetry contours, mesh boundary, direction vectors, each toggles layer visibility
- Opacity sliders: separate sliders for wave field (0-100%) and bathymetry (0-100%)
- Color scale legend: gradient bar with min/max labels, updates when variable changes
- Export button: download results as CSV (cell_id, center_lon, center_lat, hs, tp, direction) or GeoJSON (FeatureCollection with properties)
- Tests: toggle layers, verify rendering updates, change variable and check color scale, export and verify CSV format

**Day 27: Real-time progress updates via WebSocket**
- `frontend/src/websocket/simulationProgress.ts` — WebSocket connection to backend: `ws://localhost:3000/ws/simulations/:id`
- Server-side: Axum WebSocket handler broadcasts progress messages to connected clients (simulation ID as room key)
- Progress message format: JSON `{"iteration": 42, "max_change": 1.2e-3, "progress_pct": 15.0, "status": "running"}`
- Frontend: connect on simulation start, listen for messages, update progress bar in UI
- Progress bar component: linear progress bar with percentage label, displays current iteration and convergence status
- Cancel simulation button: send DELETE request to `/api/simulations/:id`, server marks job as cancelled, worker stops iteration
- Reconnection logic: auto-reconnect on WebSocket close (exponential backoff: 1s, 2s, 4s, 8s)
- Tests: mock WebSocket server, send progress messages, verify UI updates

**Day 28: End-to-end WASM solver workflow**
- Full workflow: (1) upload bathymetry GeoTIFF → (2) generate mesh (click button, wait for completion) → (3) configure boundary conditions (ERA5 or manual spectrum) → (4) run WASM solver (in-browser) → (5) visualize results on map
- All steps execute in browser for domains ≤5,000 cells (zero server cost, instant results)
- Web Workers for solver execution: offload solver to worker thread to keep UI responsive during computation (typically 10-60s)
- Worker communication: `postMessage` to send mesh/settings to worker, `onmessage` to receive progress updates and results
- Loading states: skeleton loaders for mesh generation, animated spinner during solver execution
- Error handling: display error toasts for failed mesh generation, solver divergence, file upload errors
- Tests: full workflow end-to-end test in Playwright (upload test GeoTIFF, generate mesh, run solver, verify results rendered)

### Phase 5 — Server Solver + GPU Acceleration (Days 29–33)

**Day 29: Server-side simulation worker**
- `src/workers/simulation_worker.rs` — Background worker process, runs in separate binary or via Tokio task
- Redis job queue listener: `BZPOPMAX simulation:jobs:cpu 30` (blocking pop from sorted set, 30s timeout)
- Job processing loop: fetch job → update status to "running" → load mesh from S3 → initialize solver → run solve → upload results → update status to "completed"
- Mesh loading: download from S3 (`s3://waveforge-meshes/{project_id}/mesh.bin`), deserialize using `bincode` or custom parser
- Progress updates: update `simulation_jobs.progress_pct` in PostgreSQL every 10 iterations, broadcast via WebSocket to frontend
- Error handling: catch solver errors (divergence, invalid input), mark job as "failed", store error_message in database
- Graceful shutdown: on SIGTERM, finish current job before exiting (up to 5 minute timeout), re-enqueue unfinished jobs
- Tests: enqueue test job, verify worker picks up, check progress updates in database, verify results uploaded to S3

**Day 30: GPU backend with wgpu**
- `solver-core/src/gpu/mod.rs` — wgpu device initialization: request adapter (prefer high-performance GPU), create device and queue
- `solver-core/src/gpu/kernels.wgsl` — WGSL compute shader for spectral integration: each workgroup processes one cell, threads iterate over frequency/direction bins
- GPU buffer management: create buffers for mesh data (cell connectivity, coords, areas), bathymetry, action/energy matrices, upload from CPU via `queue.write_buffer`
- Compute pass: bind buffers, dispatch workgroups (n_cells / 64 workgroups, 64 threads per workgroup), execute shader
- Download results: `buffer.slice(..).map_async()` to read results back to CPU
- Fallback to CPU: if GPU unavailable (no device, out of memory), fall back to CPU solver automatically
- Performance: GPU overhead ~10ms for setup + data transfer, amortized over many iterations
- Tests: run GPU solver on 10K cell mesh, compare results to CPU (within 1e-6 relative error), benchmark speedup (expect 10-50x)

**Day 31: GPU-accelerated source terms**
- Parallel wind input, whitecapping, breaking, friction computation on GPU: each thread computes source for one (cell, freq, dir) bin
- Reduction kernels for integrated wave parameters: parallel sum over spectrum to compute m₀, m₁, m₂ moments, reduce using shared memory
- Atomic operations for thread-safe updates to shared data (e.g., convergence residual)
- Memory optimization: minimize CPU-GPU transfers, keep action/energy matrices on GPU across iterations, only transfer final results
- Double buffering: use two sets of buffers to overlap compute and data transfer (while GPU computes iteration N, CPU uploads BC for iteration N+1)
- Performance profiling: use wgpu profiling to identify bottlenecks, optimize hotspots
- Tests: verify GPU source terms match CPU (within 1e-6), benchmark individual kernels (wind input: 0.5ms, breaking: 1ms, reduction: 0.2ms for 10K cells)

**Day 32: Job queue priority and resource allocation**
- Priority queue in Redis: sorted set `ZADD simulation:jobs:cpu {job_id} {priority}`, higher priority = processed first
- Priority based on plan tier: Advanced (20) > Pro (10) > Free (0), updated on plan change via Stripe webhook
- Worker pool: separate workers for CPU and GPU queues (`simulation:jobs:cpu` vs `simulation:jobs:gpu`)
- Resource allocation: CPU workers request 4-16 cores (via `--cpus` Docker flag or `taskset`), GPU workers request 1 GPU (via `nvidia-docker` or `CUDA_VISIBLE_DEVICES`)
- Worker health checks: heartbeat every 60s, update `last_seen` timestamp, mark worker as dead if no heartbeat for 5 minutes
- Job timeout: fail job if running > 30 minutes (Advanced plan), > 10 minutes (Pro), > 5 minutes (Free)
- Tests: enqueue 10 jobs with mixed priorities, verify execution order matches priority, check resource limits enforced

**Day 33: Server solver integration tests and benchmarks**
- End-to-end server workflow: API creates simulation → worker picks up job → solver runs → results uploaded → WebSocket notification sent → frontend fetches results
- Large domain tests: 10K cells (CPU: 60s, GPU: 5s), 50K cells (CPU: 15min, GPU: 60s), 100K cells (GPU: 5min, CPU would take hours)
- GPU vs CPU performance comparison: measure wall time, compute speedup factor, plot scaling curve (cells vs time)
- Memory usage: monitor peak RSS during solve (CPU: ~500 MB for 50K cells, GPU: ~2 GB including VRAM)
- Stress tests: enqueue 100 concurrent jobs, verify queue stability, check no jobs lost, measure average wait time
- Error injection: test solver divergence (invalid BC), out-of-memory (huge mesh), timeout, verify graceful failure

### Phase 6 — Extreme Value Analysis + Wave Climate (Days 34–37)

**Day 34: Python FastAPI service for extreme value analysis**
- `eva-service/` — Python 3.12 FastAPI app, dependencies: `fastapi`, `uvicorn`, `scipy`, `numpy`, `pandas`, `pydantic`
- `eva-service/extreme_value.py` — Peaks Over Threshold (POT) method: extract peaks above threshold (e.g., Hs > 3m), fit Generalized Pareto Distribution (GPD) using Maximum Likelihood Estimation (MLE), compute return values for 10-yr, 50-yr, 100-yr return periods
- Annual Maxima method: extract yearly maximum Hs, fit Generalized Extreme Value (GEV) distribution, compute return values
- Bootstrap confidence intervals: resample data 1000 times, refit distribution, compute percentiles (2.5%, 97.5%) for 95% CI
- Return period calculation: R_p = 1 / (1 - F(x)) where F is CDF, x is return value
- API endpoints: `POST /eva/pot` (request: time_series, threshold; response: gpd_params, return_values, confidence_intervals), `POST /eva/annual_maxima` (request: time_series; response: gev_params, return_values, CI)
- Tests: fit GPD to synthetic data (known parameters), verify return values accurate (within 5% error), check CI coverage (95% of true values within CI)

**Day 35: Wave climate statistics**
- `eva-service/wave_climate.py` — Compute bulk statistics from wave time series (hourly Hs, Tp, direction from ERA5 or simulation results)
- Percentiles: Hs P50 (median), P90, P95, P99, Tp P50, P90; use NumPy `np.percentile()`
- Directional rose: bin directions into 16 sectors (22.5° each: N, NNE, NE, ENE, E, ...), compute occurrence frequency and mean Hs per sector
- Monthly statistics: group by month (Jan-Dec), compute mean/std/max Hs and Tp for seasonal analysis
- Storm event extraction: identify peaks above threshold (e.g., Hs > 4m), apply minimum inter-event time (e.g., 48 hours), extract peak Hs, duration, total energy
- Wave power calculation: P = ρ·g²/(64π) · H²·T (kW/m), integrated over time for wave energy resource assessment
- API endpoints: `POST /wave_climate/statistics` (request: time_series; response: percentiles, directional_rose, monthly_stats, storm_events)
- Tests: compute stats for synthetic time series, verify percentiles correct, check directional rose sums to 100%, validate monthly aggregation

**Day 36: EVA API integration with main backend**
- `src/api/handlers/wave_climate.rs` — Fetch wave time series from database or ERA5, call Python EVA service via HTTP client (`reqwest`)
- Request format: POST to `http://eva-service:8000/eva/pot` with JSON body containing time series array
- Response parsing: deserialize JSON to Rust structs (EVAResults, ReturnValue, ConfidenceInterval)
- Cache EVA results in PostgreSQL: insert into `wave_climate_stats` table with `extreme_values` JSONB column
- Async job for long-running EVA computations (>10,000 data points): enqueue via Redis, process in background worker, notify via WebSocket when complete
- Retry logic: retry HTTP request up to 3 times with exponential backoff (1s, 2s, 4s) if EVA service unavailable
- Tests: mock Python service with test response, trigger EVA job, verify results stored in database, check caching (second request returns cached result)

**Day 37: Wave climate visualization in frontend**
- `frontend/src/components/WaveClimateCharts.tsx` — React component with multiple chart types
- Directional rose: polar chart using Recharts or custom SVG, 16-sector rose with colored bars (height = occurrence frequency, color = mean Hs)
- Return period curve: line chart (x-axis: return period in years on log scale, y-axis: Hs in meters), plot return values with error bars (95% CI)
- Monthly variability: box plot or violin plot for Hs by month (median, quartiles, outliers), line chart for monthly mean Hs/Tp
- Storm event timeline: scatter plot or timeline view showing storm peaks over time
- Export buttons: download charts as PNG (via html2canvas), download data as CSV
- Interactive tooltips: show exact values on hover (return period: "50-year return value: 8.2 m [7.5-9.1 m]")
- Tests: render charts with sample data, verify polar chart geometry, check return period curve shape (should be concave increasing)

### Phase 7 — Billing + Deployment (Days 38–42)

**Day 38: Stripe integration**
- `src/billing/mod.rs` — Stripe API client using `stripe-rust` crate
- Create checkout session: `POST /api/billing/checkout` → Stripe Checkout session URL, redirect user to Stripe payment page
- Subscription plans: Free (limits: 5K cells, 10 simulations/month), Pro ($129/mo: 50K cells, 100 sims/month, ERA5 BC), Advanced ($349/mo: unlimited cells, GPU, API access)
- Customer portal: `POST /api/billing/portal` → Stripe Customer Portal URL for managing subscription, payment methods, invoices
- Webhook handler: `POST /api/webhooks/stripe` → process events (customer.subscription.created, updated, deleted, invoice.paid, invoice.payment_failed)
- Subscription lifecycle: on subscription.created → update user.plan + stripe_subscription_id, on subscription.deleted → downgrade to free plan, on invoice.payment_failed → send warning email
- Plan enforcement: check user.plan before allowing operations (mesh generation, simulation submission), return 402 Payment Required if limit exceeded
- Tests: mock Stripe webhooks, verify plan updates in database, test checkout flow in Stripe test mode, verify enforcement (free user blocked from >5K cells)

**Day 39: Usage tracking and enforcement**
- `src/usage/mod.rs` — Track usage metrics: compute_seconds (solver wall time), storage_gb_hours (S3 storage × time), mesh_cells (max cell count per simulation), gpu_hours (GPU solver time)
- Record usage: on simulation completion, insert record into `usage_records` table with quantity, period (current month)
- Periodic aggregation job: cron job runs daily, aggregates usage per user/org for current billing period (month), stores in summary table
- Usage dashboard: `GET /api/usage/current` returns current period usage (compute_seconds: 450 / 3600 limit, simulations: 8 / 100 limit)
- Overage alerts: if Pro/Advanced user exceeds 120% of included quota, send email warning, offer upgrade or pay-per-use
- Usage limits enforcement: before simulation, check current usage < plan limit, block if exceeded (return 429 Too Many Requests)
- Billing integration: export usage data to Stripe for metered billing (if implementing pay-per-use overage charges)
- Tests: record usage for test simulation, aggregate by period, verify dashboard displays correct values, test limit enforcement (block after limit)

**Day 40: Production deployment infrastructure**
- AWS infrastructure: Terraform configuration for all resources
  - ECS Fargate: cluster for Axum API (2 services: `waveforge-api`, `waveforge-eva`), auto-scaling 2-10 tasks based on CPU
  - EC2 c7g.4xlarge: CPU worker pool (2-4 instances) with auto-scaling based on Redis queue depth
  - EC2 p4d.24xlarge: GPU worker pool (1-2 instances, spot instances for cost savings)
  - RDS PostgreSQL 16: db.r6g.xlarge with Multi-AZ, automated backups, encrypted storage
  - ElastiCache Redis 7: r6g.large with cluster mode, 2 replicas for HA
  - S3 buckets: `waveforge-bathymetry`, `waveforge-meshes`, `waveforge-results`, `waveforge-era5-cache` (lifecycle policy: expire after 90 days)
- Networking: VPC with public subnets (ALB, NAT gateway) and private subnets (ECS, RDS, EC2 workers), security groups restrict traffic (ALB → ECS, ECS → RDS/Redis/S3)
- Load balancer: ALB with SSL termination (ACM certificate), health checks (`/health` endpoint), sticky sessions for WebSocket
- CI/CD pipeline: GitHub Actions workflow triggers on push to `main`, builds Docker images (Rust API, Python EVA), pushes to ECR, updates ECS task definitions with blue-green deployment
- Monitoring: Prometheus exporters on all services, Grafana dashboards (API latency p50/p95/p99, solver runtime, GPU utilization, job queue depth, database connections), Sentry for error tracking, CloudWatch Logs aggregation
- Tests: smoke tests on staging environment (create project, upload bathymetry, generate mesh, run simulation, verify results), load tests with k6 (simulate 100 concurrent users)

**Day 41: Performance optimization**
- Frontend optimization: code splitting (lazy load routes), React.lazy for components, Vite build with tree-shaking, WASM streaming compilation (`WebAssembly.instantiateStreaming`), service worker for offline support
- Backend optimization: database query optimization (add missing indexes: `projects(updated_at)`, `simulations(user_id, created_at)`), connection pooling (tune `max_connections`), Redis caching for hot data (user profiles, mesh metadata)
- Solver optimization: SIMD vectorization for spectral integration using `std::simd` (AVX2 on x86_64, NEON on ARM), loop unrolling, prefetching for cache efficiency
- CDN configuration: CloudFront distribution for WASM bundle and static assets, cache control headers (max-age=31536000 for immutable assets, max-age=3600 for HTML), gzip/brotli compression
- Load testing: use k6 to simulate 100 concurrent users, measure API latency (target: p95 < 2s), identify bottlenecks with flame graphs, optimize hot paths
- Performance benchmarks: measure improvements (API latency: -30%, solver runtime: -40% with SIMD, WASM bundle size: -50% with wasm-opt)

**Day 42: Documentation, UI polish, and launch preparation**
- User documentation: Getting Started guide (create project, upload bathymetry, generate mesh, run simulation), Tutorials (harbor wave study, beach nourishment design, storm surge assessment), API reference (OpenAPI spec), FAQ
- Developer documentation: Architecture overview (system diagram, data flow), Deployment guide (Terraform setup, environment variables), Solver theory (equations, discretization, source terms), Contributing guide
- Video tutorials: record screen capture videos (5-10 minutes each) showing: mesh generation walkthrough, running simulation and interpreting results, using ERA5 boundary conditions, extreme value analysis
- UI polish: consistent loading states (skeleton loaders for lists, spinner for actions), empty states (onboarding for new users), error messages (user-friendly, actionable), success toasts (confirmation feedback)
- Accessibility: ARIA labels for interactive elements, keyboard navigation (tab order, shortcuts), color contrast (WCAG AA compliance), screen reader testing
- Launch preparation: beta user outreach (send invites to 50 coastal engineers/researchers), landing page (hero section, features, pricing, testimonials), pricing page (comparison table, FAQ), legal (terms of service, privacy policy)

**Day 38:** Stripe integration (checkout, customer portal, webhooks), plan enforcement
**Day 39:** Usage tracking (compute seconds, storage, mesh cells), usage dashboard, overage alerts
**Day 40:** AWS infrastructure (ECS Fargate, EC2 GPU workers, RDS, ElastiCache, S3), Terraform, CI/CD
**Day 41:** Performance optimization (frontend code splitting, backend query optimization, SIMD, CDN)
**Day 42:** Documentation (user guides, tutorials, API reference), UI polish, launch prep

---

## Critical Files

```
waveforge/
├── backend/src/
│   ├── main.rs, config.rs, state.rs, error.rs
│   ├── auth/mod.rs, oauth.rs
│   ├── api/handlers/auth.rs, projects.rs, mesh.rs, simulation.rs, wave_climate.rs, billing.rs
│   ├── db/mod.rs, models.rs
│   ├── storage/mod.rs
│   ├── era5/mod.rs
│   ├── workers/simulation_worker.rs
│   └── migrations/001_initial.sql
├── mesh-core/src/
│   ├── lib.rs, types.rs, delaunay.rs, refinement.rs, interpolation.rs, io.rs
├── solver-core/src/
│   ├── lib.rs, spectrum.rs, dispersion.rs, action_balance.rs, boundary.rs, parameters.rs
│   ├── source_terms/mod.rs, wind_input.rs, whitecapping.rs, breaking.rs, bottom_friction.rs, triads.rs, quadruplets.rs
│   └── gpu/mod.rs, kernels.wgsl
├── solver-wasm/src/lib.rs
├── eva-service/main.py, extreme_value.py, wave_climate.py
└── frontend/src/
    ├── components/MapView.tsx, WaveResultsViewer.tsx, WaveClimateCharts.tsx
    ├── solver/wasmSolver.ts
    └── stores/projectStore.ts, simulationStore.ts
```

---

## Solver Validation Suite

### Benchmark 1: Fetch-Limited Wave Growth

**Setup:** 10 m/s wind over 1000 km fetch, 1000m depth, no breaking/friction

**Expected Results:**
- 10 km: Hs ≈ 0.8-1.0 m, Tp ≈ 4-5 s
- 100 km: Hs ≈ 2.5-3.0 m, Tp ≈ 8-9 s
- 1000 km: Hs ≈ 5.5-6.5 m, Tp ≈ 13-15 s

**Validation:** Compare to JONSWAP fetch relations, SWAN test case

### Benchmark 2: Depth-Limited Breaking

**Setup:** Hs=2.0m, Tp=10s wave over 1:100 slope from 20m to 2m depth

**Expected Results:**
- Shoaling (20m→8m): Hs increases 10-15%
- Shallow (8m→4m): Hs ≈ 2.4-2.6 m
- Breaking (4m→2m): Hs/d ≈ 0.5-0.6, Hs ≈ 1.2-1.4 m at 2m

**Validation:** Battjes & Janssen 1978 laboratory data

### Benchmark 3: Refraction Around Island

**Setup:** 1km radius island, 50m depth, offshore Hs=2.0m, Tp=8s from West

**Expected Results:**
- Upwave: Hs ≈ 2.0 m
- Sides: Hs ≈ 1.5-1.8 m
- Leeward: Hs ≈ 0.5-1.0 m (diffraction shadow)

**Validation:** Sommerfeld diffraction, SWAN circular island test

### Benchmark 4: Bimodal Spectrum (Wind Sea + Swell)

**Setup:** Wind sea (Hs=1.5m, Tp=6s, 320°) + Swell (Hs=2.0m, Tp=12s, 270°), 100m depth

**Expected Results:**
- Total Hs ≈ 2.5 m
- Two spectral peaks at f≈0.17 Hz and f≈0.083 Hz
- Directional peaks at 320° and 270°

**Validation:** Buoy measurements with bimodal spectra

### Benchmark 5: Harbor Resonance

**Setup:** 500m×300m×10m harbor, 50m entrance, Hs=1.0m, Tp=8-15s sweep

**Expected Results:**
- Resonance at Tp ≈ 9.5 s (fundamental mode)
- Amplification 2-4x at resonance, <1.5x off-resonance
- Maximum Hs ≈ 3-4 m inside harbor

**Validation:** Raichlen 1966 analytical harbor resonance

---

## Verification Checklist

- [ ] All migrations apply cleanly
- [ ] Auth flow (register, login, JWT, OAuth)
- [ ] Bathymetry upload (GeoTIFF, XYZ, NetCDF) with metadata extraction
- [ ] Mesh generation (Delaunay, refinement, quality metrics)
- [ ] Dispersion relation accuracy (±0.1% error)
- [ ] Source terms match SWAN (wind input, whitecapping, breaking, friction)
- [ ] Action balance convergence (<500 iterations typical)
- [ ] Integrated Hs/Tp/direction (compare to analytical for JONSWAP spectrum)
- [ ] WASM solver matches native (within 1% error)
- [ ] Deck.gl mesh visualization with bathymetry
- [ ] Wave results visualization (color scale, vectors, tooltips)
- [ ] Server worker (job queue, progress, S3 upload)
- [ ] GPU acceleration (10x+ speedup vs CPU)
- [ ] ERA5 boundary conditions (download, cache, interpolate)
- [ ] EVA (POT/GEV fitting, return values, CI)
- [ ] Wave climate stats (percentiles, directional rose, monthly)
- [ ] Stripe integration (subscription, webhooks, enforcement)
- [ ] Usage tracking (record, enforce limits, dashboard)
- [ ] WebSocket progress updates
- [ ] All 5 benchmarks pass (<10% error vs reference)
- [ ] Performance: 5K WASM <20s, 50K GPU <60s
- [ ] Load test: 100 users, <5% error, <2s p95 latency

---

## Deployment Architecture

### Production Infrastructure (AWS)

**Compute:**
- Axum API: ECS Fargate (2 vCPU, 4GB) auto-scaling 2-10 instances
- CPU Workers: EC2 c7g.4xlarge (16 vCPU, 32GB) for ≤50K cells
- GPU Workers: EC2 p4d.24xlarge (8× A100) for >50K cells
- Python EVA: ECS Fargate (2 vCPU, 4GB)

**Data:**
- PostgreSQL: RDS db.r6g.xlarge Multi-AZ
- Redis: ElastiCache r6g.large
- S3: Standard (bathymetry/mesh), Glacier Instant (ERA5 archive)

**Networking:**
- ALB with SSL, CloudFront CDN, VPC public/private subnets, Route53 DNS

**Monitoring:**
- Prometheus + Grafana (API latency, solver runtime, GPU utilization, job queue depth)
- Sentry error tracking
- CloudWatch Logs

**Security:**
- Secrets Manager, IAM least privilege, WAF (rate limiting, SQL injection), security groups

---

## Post-MVP Roadmap

### Version 1.1 — Nearshore Circulation (Q2 2026)
- Depth-averaged 2DH shallow water solver
- Wave radiation stress forcing (wave-driven currents)
- Tidal forcing (FES2014), wetting/drying for inundation
- Use cases: longshore currents, harbor circulation, coastal flooding

### Version 1.2 — Sediment Transport (Q3 2026)
- Bed/suspended load (Soulsby-van Rijn, Van Rijn 2007)
- Wave-current bed shear stress
- Morphodynamic evolution with MORFAC acceleration
- Use cases: beach erosion, inlet migration, shoreline evolution

### Version 1.3 — Time-Series Simulation (Q4 2026)
- Non-stationary solver (time-stepping)
- Multi-day/year simulations with ERA5
- Storm hindcasting, climate change projections (CMIP6)
- Use cases: historical storms, extreme event analysis

### Version 1.4 — Harbor Design Tools (Q1 2027)
- Mild-slope equation for harbor resonance
- Breakwater overtopping (EurOtop neural network)
- Armor stability (Van der Meer formulas), berthing/mooring (6 DOF)
- Use cases: port design, breakwater optimization

### Version 2.0 — 3D Coupling (Q3 2027)
- 3D sigma-coordinate circulation
- Two-way wave-current coupling
- Full morphodynamic coupling
- Use cases: rip currents, 3D sediment plumes
