# 75. EnviroMod — Environmental Fate and Transport Modeling Platform

## Implementation Plan

**MVP Scope:** Browser-based 3D groundwater flow modeling with WebGL layer visualization and interactive cross-section builder, custom finite-difference MODFLOW-compatible solver compiled to WebAssembly for aquifers ≤10,000 cells and server-side Rust-native execution for larger models, steady-state and transient flow analysis with automatic boundary condition assignment (wells, rivers, recharge, drains, general head), single-species contaminant transport via advection-dispersion equation with linear sorption and first-order decay, forward particle tracking (MODPATH-equivalent) for capture zone delineation, GeoTIFF and Shapefile import for site boundaries and elevation data, water table/potentiometric surface rendering via interpolated color mesh with contour lines, drawdown analysis and water budget tracking per layer and zone, PostgreSQL storage for model geometry and parameters with S3 for raster data and simulation results, real-time convergence monitoring via WebSocket, three-tier Stripe billing (Free / Pro $99/mo / Enterprise custom).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| Flow Solver | Rust (native + WASM) | Custom MODFLOW-compatible finite-difference solver with PCG linear solver |
| WASM Compilation | `wasm-pack` + `wasm-bindgen` | Client-side solver for models ≤10,000 cells |
| Spatial Processing | Python 3.12 (FastAPI) | GeoTIFF rasterization, Shapefile parsing, DEM interpolation via GDAL |
| Database | PostgreSQL 16 with PostGIS | Projects, model layers, boundary conditions, simulation results metadata |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations, PostGIS geometry support |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | GeoTIFF elevation data, simulation results (head/concentration grids), exported reports |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| 3D Visualization | Three.js + WebGL | Layer-based aquifer rendering, cross-section slicing, particle pathlines |
| Map Integration | Mapbox GL JS | Basemap tiles, site boundary overlay, well locations |
| Real-time | WebSocket (Axum) | Live solver iteration progress, convergence monitoring |
| Job Queue | Redis 7 + Tokio tasks | Server-side simulation job management, priority queuing |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | WASM bundle delivery, static frontend assets |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, error tracking |
| CI/CD | GitHub Actions | WASM build pipeline, Rust tests, Docker image push |

### Key Architecture Decisions

1. **Hybrid client/server solver with WASM threshold at 10,000 cells**: Models with ≤10,000 active cells (typical site-scale models: 100×100×1 = 10K cells) run entirely in the browser via WASM, providing instant feedback with zero server cost. Regional models exceeding 10,000 cells are submitted to the Rust-native server solver which handles multi-million-cell grids and parallel parameter sweeps. The threshold is configurable per plan tier.

2. **Custom MODFLOW-compatible finite-difference solver rather than wrapping MODFLOW-6**: Building a custom solver in Rust gives us full control over WASM compilation, GPU acceleration opportunities, and modern async I/O. MODFLOW-6's Fortran codebase has extensive global state that makes WASM compilation impractical and thread-unsafe. Our Rust solver uses the Preconditioned Conjugate Gradient (PCG) method for sparse linear systems (identical to MODFLOW's default), maintaining compatibility with MODFLOW input formats while achieving memory safety and WASM portability.

3. **Three.js layer-based visualization with live cross-section slicing**: Three.js enables GPU-accelerated rendering of multi-layer aquifer systems with texture-mapped hydraulic head and concentration fields. Users can interactively drag a cross-section plane to visualize vertical flow patterns and contaminant plumes. This is vastly superior to static 2D grid plots and rivals commercial GMS/Visual MODFLOW interfaces. Color ramps use scientifically-appropriate palettes (viridis/plasma) for accessibility.

4. **PostGIS for spatial queries and intersection operations**: PostGIS provides efficient spatial indexing (R-tree via GIST) for boundary condition assignment (e.g., "which cells intersect river shapefile?") and zone-based water budget calculations. Geometry storage in WKB format enables direct compatibility with QGIS and ArcGIS for model exchange. This avoids reimplementing spatial operations in Rust.

5. **S3 for raster data with PostgreSQL metadata catalog**: Elevation grids (GeoTIFF files, typically 10-100MB for site-scale models) and simulation results (head/concentration grids at each timestep) are stored in S3, while PostgreSQL holds searchable metadata (layer names, CRS, extent, statistics). This allows models to scale to regional grids (millions of cells) without bloating the database while enabling fast parametric search via PostgreSQL indexes.

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

-- Projects (groundwater model workspace)
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    site_boundary GEOMETRY(POLYGON, 4326),
    model_crs TEXT DEFAULT 'EPSG:4326',
    grid_config JSONB NOT NULL,
    time_config JSONB,
    solver_config JSONB DEFAULT '{}',
    is_public BOOLEAN NOT NULL DEFAULT false,
    forked_from UUID REFERENCES projects(id) ON DELETE SET NULL,
    thumbnail_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);
CREATE INDEX projects_boundary_idx ON projects USING GIST(site_boundary);

-- Model Layers
CREATE TABLE model_layers (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    layer_num INTEGER NOT NULL,
    name TEXT NOT NULL,
    layer_type TEXT NOT NULL,
    elevation_top_url TEXT,
    elevation_bot_url TEXT,
    hydraulic_conductivity JSONB NOT NULL,
    specific_storage REAL,
    specific_yield REAL,
    porosity REAL DEFAULT 0.3,
    is_active BOOLEAN DEFAULT true,
    UNIQUE(project_id, layer_num)
);
CREATE INDEX layers_project_idx ON model_layers(project_id);

-- Boundary Conditions
CREATE TABLE boundary_conditions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    bc_type TEXT NOT NULL,
    name TEXT NOT NULL,
    geometry GEOMETRY(GEOMETRY, 4326),
    layer_num INTEGER,
    parameters JSONB NOT NULL,
    stress_period_data JSONB,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX bc_project_idx ON boundary_conditions(project_id);
CREATE INDEX bc_geom_idx ON boundary_conditions USING GIST(geometry);

-- Simulations
CREATE TABLE simulations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    simulation_type TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending',
    execution_mode TEXT NOT NULL DEFAULT 'wasm',
    active_cells INTEGER NOT NULL DEFAULT 0,
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
    cores_allocated INTEGER DEFAULT 1,
    memory_mb INTEGER DEFAULT 4096,
    priority INTEGER DEFAULT 0,
    progress_pct REAL DEFAULT 0.0,
    convergence_data JSONB DEFAULT '[]',
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX jobs_sim_idx ON simulation_jobs(simulation_id);

-- Contaminant Species
CREATE TABLE species (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    cas_number TEXT,
    molecular_weight REAL,
    diffusion_coeff REAL,
    decay_rate REAL DEFAULT 0.0,
    sorption_type TEXT DEFAULT 'linear',
    sorption_params JSONB,
    initial_concentration REAL DEFAULT 0.0,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX species_project_idx ON species(project_id);

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
use geojson::Geometry;

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
    #[sqlx(json)]
    pub site_boundary: Option<Geometry>,
    pub model_crs: String,
    pub grid_config: serde_json::Value,
    pub time_config: Option<serde_json::Value>,
    pub solver_config: serde_json::Value,
    pub is_public: bool,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct GridConfig {
    pub nrow: usize,
    pub ncol: usize,
    pub nlay: usize,
    pub delr: Vec<f64>,
    pub delc: Vec<f64>,
    pub top: Vec<f64>,
    pub botm: Vec<Vec<f64>>,
    pub rotation: f64,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Simulation {
    pub id: Uuid,
    pub project_id: Uuid,
    pub user_id: Uuid,
    pub simulation_type: String,
    pub status: String,
    pub execution_mode: String,
    pub active_cells: i32,
    pub results_url: Option<String>,
    pub results_summary: Option<serde_json::Value>,
    pub error_message: Option<String>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub wall_time_ms: Option<i32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct SolverOptions {
    pub max_outer_iter: u32,
    pub max_inner_iter: u32,
    pub head_tolerance: f64,
    pub residual_tolerance: f64,
    pub relaxation: f64,
}

impl Default for SolverOptions {
    fn default() -> Self {
        Self {
            max_outer_iter: 500,
            max_inner_iter: 100,
            head_tolerance: 1e-4,
            residual_tolerance: 1e-3,
            relaxation: 1.0,
        }
    }
}
```

---

## Solver Architecture Deep-Dive

### Governing Equations and Discretization

EnviroMod's core solver implements **finite-difference discretization of the groundwater flow equation**, solving for hydraulic head *h* at each cell center:

```
∂/∂x(Kₓₓ ∂h/∂x) + ∂/∂y(Kᵧᵧ ∂h/∂y) + ∂/∂z(Kᵤᵤ ∂h/∂z) + W = Sₛ ∂h/∂t
```

Where:
- **Kₓₓ, Kᵧᵧ, Kᵤᵤ**: Hydraulic conductivity tensor (m/day)
- **h**: Hydraulic head (m)
- **W**: Volumetric flux (sources/sinks) (1/day)
- **Sₛ**: Specific storage (1/m) for confined, or Sᵧ (specific yield) for unconfined

**Block-centered finite difference** discretizes space into rectangular cells *(i, j, k)*. For each cell, the flow equation becomes:

```
CVᵢ₊½,ⱼ,ₖ(hᵢ₊₁,ⱼ,ₖ - hᵢ,ⱼ,ₖ) + CVᵢ₋½,ⱼ,ₖ(hᵢ₋₁,ⱼ,ₖ - hᵢ,ⱼ,ₖ)
+ CCᵢ,ⱼ₊½,ₖ(hᵢ,ⱼ₊₁,ₖ - hᵢ,ⱼ,ₖ) + CCᵢ,ⱼ₋½,ₖ(hᵢ,ⱼ₋₁,ₖ - hᵢ,ⱼ,ₖ)
+ CRᵢ,ⱼ,ₖ₊½(hᵢ,ⱼ,ₖ₊₁ - hᵢ,ⱼ,ₖ) + CRᵢ,ⱼ,ₖ₋½(hᵢ,ⱼ,ₖ₋₁ - hᵢ,ⱼ,ₖ)
+ Wᵢ,ⱼ,ₖ = SCᵢ,ⱼ,ₖ (hᵢ,ⱼ,ₖⁿ⁺¹ - hᵢ,ⱼ,ₖⁿ) / Δt
```

Where CV, CC, CR are conductances, SC is storage coefficient. This yields a **sparse symmetric linear system Ah = b**.

**Preconditioned Conjugate Gradient (PCG)** solver (identical to MODFLOW's default):
1. Initial guess: h⁰ (from previous time step)
2. Residual: r⁰ = b - Ah⁰
3. Preconditioner: M (Jacobi or incomplete Cholesky)
4. Iterate: hᵏ⁺¹ = hᵏ + αᵏpᵏ until ||rᵏ|| < tolerance

**Convergence**: max(|hᵢⁿ⁺¹ - hᵢⁿ|) < 1e-4 m AND max(|residual|) < 1e-3 m³/day.

**Contaminant transport** uses advection-dispersion:

```
∂(θC)/∂t = ∂/∂xᵢ(θDᵢⱼ ∂C/∂xⱼ) - ∂/∂xᵢ(θvᵢC) - λθC - ρₐKₐC
```

Discretized via **method of characteristics with backward tracking** or **TVD finite-difference**.

### Client/Server Split (WASM Threshold)

```
Model uploaded → Active cell count extracted
    │
    ├── ≤10,000 cells → WASM solver (browser)
    │   ├── Instant startup
    │   ├── Results displayed immediately
    │   └── No server cost
    │
    └── >10,000 cells → Server solver (Rust native)
        ├── Job queued via Redis
        ├── Worker picks up, runs solver
        ├── Progress streamed via WebSocket
        └── Results stored in S3
```

10,000-cell threshold chosen because:
- WASM solver handles 10K-cell steady-state in <3s
- Covers typical site-scale models (100×100×1 = 10K cells)
- Above 10K: regional aquifer models need server compute

---

## Architecture Deep-Dives

### 1. Simulation API Handler (Rust/Axum)

```rust
// src/api/handlers/simulation.rs

use axum::{extract::{Path, State, Json}, http::StatusCode, response::IntoResponse};
use uuid::Uuid;
use crate::{db::models::{Simulation, Project, GridConfig}, solver::grid::count_active_cells,
    state::AppState, auth::Claims, error::ApiError};

#[derive(serde::Deserialize)]
pub struct CreateSimulationRequest {
    pub simulation_type: String,
    pub solver_options: Option<serde_json::Value>,
}

pub async fn create_simulation(
    State(state): State<AppState>,
    claims: Claims,
    Path(project_id): Path<Uuid>,
    Json(req): Json<CreateSimulationRequest>,
) -> Result<impl IntoResponse, ApiError> {
    let project = sqlx::query_as!(Project,
        r#"SELECT * FROM projects WHERE id = $1 AND owner_id = $2"#,
        project_id, claims.user_id
    ).fetch_optional(&state.db).await?
     .ok_or(ApiError::NotFound("Project not found"))?;

    let grid_config: GridConfig = serde_json::from_value(project.grid_config.clone())?;
    let active_cells = count_active_cells(&grid_config);

    let user = sqlx::query_as!(crate::db::models::User,
        "SELECT * FROM users WHERE id = $1", claims.user_id
    ).fetch_one(&state.db).await?;

    if user.plan == "free" && active_cells > 1000 {
        return Err(ApiError::PlanLimit("Free plan supports models up to 1,000 cells."));
    }

    let execution_mode = if active_cells <= 10000 { "wasm" } else { "server" };

    let sim = sqlx::query_as!(Simulation,
        r#"INSERT INTO simulations (project_id, user_id, simulation_type, status, execution_mode, active_cells)
        VALUES ($1, $2, $3, $4, $5, $6) RETURNING *"#,
        project_id, claims.user_id, req.simulation_type, "pending", execution_mode, active_cells as i32
    ).fetch_one(&state.db).await?;

    if execution_mode == "server" {
        let priority = match user.plan.as_str() {
            "enterprise" => 20, "pro" => 10, _ => 0,
        };

        sqlx::query!(r#"INSERT INTO simulation_jobs (simulation_id, priority) VALUES ($1, $2)"#,
            sim.id, priority).execute(&state.db).await?;

        let mut conn = state.redis.get_multiplexed_async_connection().await?;
        redis::cmd("ZADD").arg("simulation:jobs").arg(priority).arg(sim.id.to_string())
            .query_async::<_, ()>(&mut conn).await?;
    }

    Ok((StatusCode::CREATED, Json(sim)))
}
```

### 2. Groundwater Flow Solver Core (Rust)

```rust
// solver-core/src/flow/mod.rs

use nalgebra_sparse::{CsrMatrix, CooMatrix};
use ndarray::{Array3, Array1};
use crate::{grid::Grid, boundary::BoundaryPackage, pcg::PcgSolver};

pub struct FlowSolver {
    pub grid: Grid,
    pub head: Array3<f64>,
    pub prev_head: Array3<f64>,
    pub matrix: CooMatrix<f64>,
    pub rhs: Array1<f64>,
    pub ibound: Array3<i8>,
    pub convergence_history: Vec<ConvergenceRecord>,
}

#[derive(Debug, Clone)]
pub struct ConvergenceRecord {
    pub iteration: u32,
    pub max_head_change: f64,
    pub max_residual: f64,
}

impl FlowSolver {
    pub fn new(grid: Grid) -> Self {
        let n_active = grid.count_active();
        Self {
            head: Array3::zeros((grid.nlay, grid.nrow, grid.ncol)),
            prev_head: Array3::zeros((grid.nlay, grid.nrow, grid.ncol)),
            matrix: CooMatrix::new(n_active, n_active),
            rhs: Array1::zeros(n_active),
            ibound: Array3::ones((grid.nlay, grid.nrow, grid.ncol)),
            grid, convergence_history: Vec::new(),
        }
    }

    pub fn build_steady_state_system(&mut self, boundaries: &[Box<dyn BoundaryPackage>])
        -> Result<(), SolverError> {
        self.matrix = CooMatrix::new(self.grid.count_active(), self.grid.count_active());
        self.rhs.fill(0.0);
        let cell_to_index = self.grid.build_cell_index_map();

        for k in 0..self.grid.nlay {
            for i in 0..self.grid.nrow {
                for j in 0..self.grid.ncol {
                    if self.ibound[[k, i, j]] <= 0 { continue; }
                    let idx = cell_to_index[[k, i, j]];
                    let cv_right = self.grid.conductance_horizontal(k, i, j, k, i, j + 1);
                    let cv_left = self.grid.conductance_horizontal(k, i, j, k, i, j.wrapping_sub(1));
                    // ... stamp conductances for all 6 neighbors
                    let mut diag = cv_right + cv_left; // + others
                    self.matrix.push(idx, idx, diag);
                }
            }
        }

        for boundary in boundaries {
            boundary.apply(&mut self.matrix, &mut self.rhs, &self.head, &cell_to_index)?;
        }
        Ok(())
    }

    pub fn solve_steady_state(&mut self, boundaries: &[Box<dyn BoundaryPackage>],
        options: &SolverOptions) -> Result<SolverResult, SolverError> {
        for outer_iter in 0..options.max_outer_iter {
            self.build_steady_state_system(boundaries)?;
            let csr = CsrMatrix::from(&self.matrix);
            let mut pcg = PcgSolver::new(csr, options.max_inner_iter as usize);
            let solution = pcg.solve(&self.rhs, options.residual_tolerance)?;

            let cell_to_index = self.grid.build_cell_index_map();
            for k in 0..self.grid.nlay {
                for i in 0..self.grid.nrow {
                    for j in 0..self.grid.ncol {
                        if self.ibound[[k, i, j]] > 0 {
                            self.head[[k, i, j]] = solution[cell_to_index[[k, i, j]]];
                        }
                    }
                }
            }

            let max_change = self.compute_max_head_change();
            let max_residual = 0.0; // compute from residual vector
            self.convergence_history.push(ConvergenceRecord {
                iteration: outer_iter + 1, max_head_change: max_change, max_residual,
            });

            if max_change < options.head_tolerance {
                return Ok(SolverResult {
                    converged: true, iterations: outer_iter + 1,
                    final_head: self.head.clone(),
                    convergence_history: self.convergence_history.clone(),
                });
            }
            self.prev_head.assign(&self.head);
        }
        Err(SolverError::Convergence(format!("Failed to converge after {} iterations",
            options.max_outer_iter)))
    }

    fn compute_max_head_change(&self) -> f64 {
        let mut max_change = 0.0;
        for k in 0..self.grid.nlay {
            for i in 0..self.grid.nrow {
                for j in 0..self.grid.ncol {
                    if self.ibound[[k, i, j]] > 0 {
                        let change = (self.head[[k, i, j]] - self.prev_head[[k, i, j]]).abs();
                        max_change = max_change.max(change);
                    }
                }
            }
        }
        max_change
    }
}

#[derive(Debug)]
pub struct SolverResult {
    pub converged: bool,
    pub iterations: u32,
    pub final_head: Array3<f64>,
    pub convergence_history: Vec<ConvergenceRecord>,
}

#[derive(Debug)]
pub enum SolverError {
    Convergence(String),
    Singular(String),
}
```

### 3. Three.js 3D Aquifer Visualization (React + Three.js)

```typescript
// frontend/src/components/AquiferViewer3D.tsx

import { useRef, useEffect, useState } from 'react';
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';
import { useModelStore } from '../stores/modelStore';
import { useSimulationStore } from '../stores/simulationStore';

export function AquiferViewer3D() {
  const containerRef = useRef<HTMLDivElement>(null);
  const sceneRef = useRef<THREE.Scene | null>(null);
  const rendererRef = useRef<THREE.WebGLRenderer | null>(null);
  const cameraRef = useRef<THREE.PerspectiveCamera | null>(null);
  const layersRef = useRef<THREE.Mesh[]>([]);
  const { gridConfig, layers } = useModelStore();
  const { currentResults } = useSimulationStore();
  const [showCrossSection, setShowCrossSection] = useState(false);

  useEffect(() => {
    if (!containerRef.current) return;
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0xf0f0f0);
    sceneRef.current = scene;

    const camera = new THREE.PerspectiveCamera(60,
      containerRef.current.clientWidth / containerRef.current.clientHeight, 0.1, 10000);
    camera.position.set(500, 500, 500);
    cameraRef.current = camera;

    const renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(containerRef.current.clientWidth, containerRef.current.clientHeight);
    containerRef.current.appendChild(renderer.domElement);
    rendererRef.current = renderer;

    const controls = new OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true;

    scene.add(new THREE.AmbientLight(0xffffff, 0.6));
    const directionalLight = new THREE.DirectionalLight(0xffffff, 0.4);
    directionalLight.position.set(100, 200, 100);
    scene.add(directionalLight);

    const animate = () => {
      requestAnimationFrame(animate);
      controls.update();
      renderer.render(scene, camera);
    };
    animate();

    return () => {
      renderer.dispose();
      containerRef.current?.removeChild(renderer.domElement);
    };
  }, []);

  useEffect(() => {
    if (!sceneRef.current || !gridConfig || !layers) return;
    layersRef.current.forEach(mesh => sceneRef.current!.remove(mesh));
    layersRef.current = [];

    const { nrow, ncol, nlay, delr, delc, top, botm } = gridConfig;
    const totalWidth = delr.reduce((sum: number, dx: number) => sum + dx, 0);
    const totalHeight = delc.reduce((sum: number, dy: number) => sum + dy, 0);

    for (let k = 0; k < nlay; k++) {
      const geometry = new THREE.PlaneGeometry(totalWidth, totalHeight, ncol - 1, nrow - 1);
      const vertices = geometry.attributes.position;
      for (let i = 0; i < nrow; i++) {
        for (let j = 0; j < ncol; j++) {
          const idx = i * ncol + j;
          const elevation = Array.isArray(top) ? top[i * ncol + j] : top;
          const layerBot = botm[k][i * ncol + j];
          vertices.setZ(idx, (elevation + layerBot) / 2);
        }
      }
      vertices.needsUpdate = true;

      const headData = new Uint8Array(ncol * nrow * 4);
      const headTexture = new THREE.DataTexture(headData, ncol, nrow, THREE.RGBAFormat);
      const material = new THREE.MeshBasicMaterial({ map: headTexture, side: THREE.DoubleSide });
      const mesh = new THREE.Mesh(geometry, material);
      mesh.rotation.x = -Math.PI / 2;
      sceneRef.current.add(mesh);
      layersRef.current.push(mesh);
    }
  }, [gridConfig, layers]);

  useEffect(() => {
    if (!currentResults || !gridConfig) return;
    const { nrow, ncol, nlay } = gridConfig;
    const headGrid = currentResults.head;
    let minHead = Infinity, maxHead = -Infinity;
    for (let k = 0; k < nlay; k++) {
      for (let i = 0; i < nrow; i++) {
        for (let j = 0; j < ncol; j++) {
          const h = headGrid[k][i][j];
          if (h !== null && isFinite(h)) {
            minHead = Math.min(minHead, h);
            maxHead = Math.max(maxHead, h);
          }
        }
      }
    }
    layersRef.current.forEach((mesh, k) => {
      const headData = new Uint8Array(ncol * nrow * 4);
      for (let i = 0; i < nrow; i++) {
        for (let j = 0; j < ncol; j++) {
          const idx = (i * ncol + j) * 4;
          const h = headGrid[k][i][j];
          if (h === null) {
            headData[idx] = headData[idx + 1] = headData[idx + 2] = 128;
            headData[idx + 3] = 128;
          } else {
            const t = (h - minHead) / (maxHead - minHead);
            const color = viridisColormap(t);
            headData[idx] = color.r * 255;
            headData[idx + 1] = color.g * 255;
            headData[idx + 2] = color.b * 255;
            headData[idx + 3] = 255;
          }
        }
      }
      (mesh.material as THREE.MeshBasicMaterial).map!.image.data = headData;
      (mesh.material as THREE.MeshBasicMaterial).map!.needsUpdate = true;
    });
  }, [currentResults, gridConfig]);

  return (
    <div className="relative w-full h-full">
      <div ref={containerRef} className="w-full h-full" />
      <div className="absolute top-4 right-4 bg-white p-4 rounded shadow">
        <label className="flex items-center space-x-2">
          <input type="checkbox" checked={showCrossSection}
            onChange={(e) => setShowCrossSection(e.target.checked)} />
          <span>Cross-Section</span>
        </label>
      </div>
    </div>
  );
}

function viridisColormap(t: number): { r: number; g: number; b: number } {
  const viridis = [
    [0.267004, 0.004874, 0.329415], [0.282623, 0.140926, 0.457517],
    [0.253935, 0.265254, 0.529983], [0.206756, 0.371758, 0.553117],
    [0.163625, 0.471133, 0.558148], [0.127568, 0.566949, 0.550556],
    [0.134692, 0.658636, 0.517649], [0.266941, 0.748751, 0.440573],
    [0.477504, 0.821444, 0.318195], [0.741388, 0.873449, 0.149561],
  ];
  const idx = Math.min(Math.floor(t * (viridis.length - 1)), viridis.length - 2);
  const frac = t * (viridis.length - 1) - idx;
  const c0 = viridis[idx], c1 = viridis[idx + 1];
  return {
    r: c0[0] + (c1[0] - c0[0]) * frac,
    g: c0[1] + (c1[1] - c0[1]) * frac,
    b: c0[2] + (c1[2] - c0[2]) * frac,
  };
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–4)

**Day 1: Project initialization and Rust backend scaffold**
```bash
cargo init enviromod-api
cd enviromod-api
cargo add axum tokio serde serde_json sqlx uuid chrono tower-http jsonwebtoken bcrypt geojson
cargo add --features postgres,runtime-tokio-rustls,uuid,chrono,json sqlx
cargo add aws-sdk-s3 redis ndarray nalgebra-sparse
```
- `src/main.rs` — Axum app with CORS, tracing middleware, graceful shutdown
- `src/config.rs` — Environment-based configuration (DATABASE_URL, JWT_SECRET, S3_BUCKET, REDIS_URL, MAPBOX_TOKEN)
- `src/state.rs` — AppState struct (PgPool, Redis, S3Client)
- `src/error.rs` — ApiError enum with IntoResponse implementation
- `Dockerfile` — Multi-stage build (cargo-chef for layer caching)
- `docker-compose.yml` — PostgreSQL with PostGIS, Redis, MinIO (S3-compatible)

**Day 2: Database schema and migrations**
- `migrations/001_initial.sql` — All 9 tables: users, organizations, org_members, projects, model_layers, boundary_conditions, simulations, simulation_jobs, species, usage_records
- Enable PostGIS extension for spatial geometry support
- `src/db/mod.rs` — Database pool initialization with PostGIS support
- `src/db/models.rs` — All SQLx structs with FromRow derives, GeoJSON geometry support
- Run `sqlx migrate run` to apply schema
- Seed script for test project with simple 2D grid (10×10×1)

**Day 3: Authentication system**
- `src/auth/mod.rs` — JWT token generation and validation middleware
- `src/auth/oauth.rs` — Google and GitHub OAuth 2.0 flow handlers
- `src/api/handlers/auth.rs` — Register, login, OAuth callback, refresh token, me endpoints
- Password hashing with bcrypt (cost factor 12)
- JWT with 24h access token + 30d refresh token
- Auth middleware that extracts Claims from Authorization header
- Integration tests for auth flow

**Day 4: User and project CRUD**
- `src/api/handlers/users.rs` — Get profile, update profile, delete account
- `src/api/handlers/projects.rs` — Create, list, get, update, delete, fork project
- `src/api/handlers/orgs.rs` — Create org, invite member, list members, remove member
- `src/api/router.rs` — All route definitions with auth middleware
- PostGIS spatial queries for site boundary intersection
- Integration tests: auth flow, project CRUD, authorization checks

### Phase 2 — Solver Core — Finite Difference Flow (Days 5–12)

**Day 5: Grid framework and cell indexing**
- `solver-core/` — New Rust workspace member (shared between native and WASM)
- `solver-core/src/grid.rs` — Grid struct with nrow, ncol, nlay, delr (row spacing), delc (column spacing), top, botm arrays
- Cell-to-index mapping for active cells (skip inactive cells in matrix assembly)
- Conductance calculation functions: `conductance_horizontal()` and `conductance_vertical()`
- Cell center coordinates for spatial operations
- Unit tests: simple 2D grid (10×10×1), 3D multi-layer grid (5×5×3)

**Day 6: Coefficient matrix assembly**
- `solver-core/src/flow/mod.rs` — FlowSolver struct with COO matrix assembly
- `build_steady_state_system()` — Stamp conductances for all cells and 6 neighbors (left, right, front, back, above, below)
- Support for inactive cells (ibound = 0) and constant head cells (ibound = -1)
- Harmonic mean conductance calculation at cell interfaces
- Unit tests: single-layer confined flow, verify matrix symmetry and structure

**Day 7: PCG linear solver**
- `solver-core/src/pcg.rs` — Preconditioned Conjugate Gradient solver implementation
- Jacobi preconditioner (diagonal scaling) for initial implementation
- Convergence monitoring (residual norm, relative tolerance)
- Unit tests: solve symmetric positive-definite systems, verify convergence rate matches theory

**Day 8: Boundary condition packages**
- `solver-core/src/boundary/mod.rs` — BoundaryPackage trait with `apply()` method
- `solver-core/src/boundary/well.rs` — Well package (constant flux Q in m³/day)
- `solver-core/src/boundary/river.rs` — River package (head-dependent flux with riverbed conductance)
- `solver-core/src/boundary/recharge.rs` — Recharge package (areal flux in m/day)
- Each package stamps contributions into coefficient matrix and RHS vector
- Tests: single well drawdown (verify radial flow pattern), river boundary interaction

**Day 9: Steady-state solver with Newton iteration**
- `solve_steady_state()` — Outer loop with nonlinearity handling for convertible layers
- Unconfined layer Newton iteration: when head < layer bottom, layer goes dry (reduce transmissivity)
- Convergence criteria: max head change < tolerance AND max residual < tolerance
- Under-relaxation for difficult convergence cases
- Tests: Theis well solution comparison, multi-layer confined-unconfined system

**Day 10: Transient solver with time stepping**
- `solver-core/src/flow/transient.rs` — Transient solver with implicit (backward Euler or Crank-Nicolson) time discretization
- Storage term assembly: specific storage (Ss) for confined layers, specific yield (Sy) for unconfined
- Adaptive time stepping based on convergence difficulty and head change rate
- Warm-start from previous time step for faster convergence
- Tests: confined transient (Theis solution), unconfined drainage

**Day 11: Additional boundary conditions**
- `solver-core/src/boundary/drain.rs` — Drain package (removes water when head > drain elevation)
- `solver-core/src/boundary/ghb.rs` — General head boundary (specified head at boundary with conductance)
- `solver-core/src/boundary/chd.rs` — Constant head boundary (Dirichlet BC)
- Stress period support: time-varying boundary conditions
- Tests: drain network for dewatering, GHB for lateral inflow from adjacent aquifer

**Day 12: Particle tracking (MODPATH-equivalent)**
- `solver-core/src/particle.rs` — Particle tracking using Pollock's semi-analytical method
- Forward tracking from source locations (wells, contamination sites)
- Velocity interpolation from head field using Darcy's law
- Pathline export as JSON with time stamps
- Stopping criteria: boundary exit, weak sink capture, maximum time
- Tests: uniform flow pathline (straight line), radial well pathline (logarithmic spiral)

### Phase 3 — WASM Build + Frontend Visualization (Days 13–18)

**Day 13: WASM solver compilation**
- `solver-wasm/` — New workspace member for WASM target
- `solver-wasm/Cargo.toml` — wasm-bindgen, serde-wasm-bindgen, js-sys dependencies
- `solver-wasm/src/lib.rs` — WASM entry points: `run_flow_simulation()`, `run_transport_simulation()`, `run_particle_tracking()`
- Memory management: efficient allocation for large head arrays (10K cells × 8 bytes = 80KB)
- `wasm-pack build --target web --release` pipeline
- `wasm-opt -Oz` for size optimization (target <3MB gzipped)
- JavaScript wrapper: `EnviroModSolver` class that loads WASM and provides async API

**Day 14: Frontend scaffold and map integration**
```bash
npm create vite@latest frontend -- --template react-ts
cd frontend
npm i zustand @tanstack/react-query axios mapbox-gl three @types/three geojson
npm i -D @types/mapbox-gl
```
- `src/App.tsx` — Router (react-router-dom), auth context provider, layout
- `src/stores/authStore.ts` — Zustand store for auth state (JWT, user profile)
- `src/stores/projectStore.ts` — Project state management
- `src/stores/modelStore.ts` — Grid configuration, layers, boundary conditions
- `src/components/MapViewer.tsx` — Mapbox GL map with site boundary polygon overlay
- GeoJSON site boundary rendering with fill and stroke
- Base map selection: satellite, streets, terrain

**Day 15: Grid builder UI**
- `src/components/GridBuilder.tsx` — Grid configuration form (nrow, ncol, nlay, row spacing, column spacing)
- `src/components/GridPreview.tsx` — 2D grid cell preview with color-coded active/inactive cells
- Grid rotation and offset controls for alignment with site coordinates
- Snap grid to uploaded site boundary shapefile (auto-compute extent)
- Validation: ensure grid covers site boundary, warn if cells are too large/small
- Export grid definition as JSON

**Day 16: Layer configuration UI**
- `src/components/LayerManager.tsx` — Add/edit/remove layers, reorder layers
- `src/components/LayerEditor.tsx` — Set layer properties:
  - Hydraulic conductivity (Kx, Ky, Kz): uniform value or upload raster (GeoTIFF)
  - Storage properties: specific storage (Ss) or specific yield (Sy)
  - Porosity (for transport)
- Elevation editor: upload GeoTIFF for top/bottom elevations or interpolate from point data
- Layer type selection: confined, unconfined, convertible
- Visual preview of layer thickness

**Day 17: Three.js 3D aquifer visualization**
- `src/components/AquiferViewer3D.tsx` — Three.js scene with layer meshes (one PlaneGeometry per layer)
- Vertex Z coordinates set from elevation data (top + bottom) / 2
- Texture-mapped hydraulic head on each layer using DataTexture (viridis colormap)
- Interactive orbit controls: pan, zoom, rotate with mouse/touch
- Cross-section plane: draggable vertical slice showing head profile
- Camera presets: top view, side view, 45° isometric
- Export 3D view as PNG screenshot

**Day 18: Boundary condition UI**
- `src/components/BoundaryConditionManager.tsx` — List of all BCs with add/edit/remove controls
- `src/components/WellEditor.tsx` — Click map to place wells, set pumping/injection rate (m³/day)
- `src/components/RiverEditor.tsx` — Draw river polylines on map, set stage and conductance per segment
- `src/components/RechargeEditor.tsx` — Draw recharge zones (polygons), set recharge rate (m/day)
- Geometry editing: click to place, drag to move, delete key to remove
- Temporal variation editor: time series input for transient BCs (CSV import or manual entry)
- Visual styling: wells as circles with size proportional to |Q|, rivers as blue lines, recharge zones with fill

### Phase 4 — Spatial Processing Service (Days 19–22)

**Day 19: Python FastAPI spatial service**
- `spatial-service/` — Python FastAPI app separate from Rust backend
- `requirements.txt` — FastAPI, GDAL, rasterio, geopandas, shapely, scipy
- `main.py` — FastAPI app with CORS middleware
- Endpoints: `/api/rasterize`, `/api/interpolate_dem`, `/api/parse_shapefile`, `/api/clip_raster`
- Health check endpoint for K8s liveness probe
- `Dockerfile` — Python 3.12 with GDAL bindings (osgeo/gdal base image)

**Day 20: GeoTIFF rasterization**
- `/api/rasterize` endpoint — Upload GeoTIFF, extract elevation grid aligned to model grid
- GDAL reproject and resample to match grid CRS (coordinate reference system) and cell spacing
- Bilinear interpolation for resampling
- Return JSON array of elevation values per cell (flat array, row-major order)
- Upload processed raster to S3, return presigned URL for download
- Tests: verify resampled raster matches input extent, check interpolation accuracy

**Day 21: Shapefile parsing and intersection**
- `/api/parse_shapefile` endpoint — Upload shapefile (or GeoJSON), extract geometries
- Convert to GeoJSON for frontend display on map
- Spatial intersection query: which grid cells intersect shapefile features?
- Return list of cell indices (i, j, k) for boundary condition assignment
- Support for point, line, and polygon geometries
- Tests: verify intersection logic with known geometries

**Day 22: DEM interpolation**
- `/api/interpolate_dem` endpoint — Upload point elevations (CSV: x, y, z), interpolate to grid
- Kriging (Gaussian process) or Inverse Distance Weighting (IDW) interpolation via scipy
- Generate top/bottom elevation rasters for layers
- Visualization: return interpolated surface as GeoTIFF
- Tests: verify interpolated surface passes through input points (zero residual)

### Phase 5 — Simulation Execution + Results (Days 23–28)

**Day 23: Simulation API endpoints**
- `src/api/handlers/simulation.rs` — Create simulation, get simulation, list simulations, cancel simulation, get results
- Active cell count calculation from grid configuration
- WASM/server routing logic based on cell count threshold (10,000 cells)
- Plan-based limits enforcement:
  - Free: 1000 cells, steady-state only, 3 projects
  - Pro: 100K cells, transient + transport, unlimited projects
  - Enterprise: unlimited
- Simulation status tracking (pending, running, completed, failed, cancelled)

**Day 24: Server-side simulation worker**
- `src/workers/simulation_worker.rs` — Redis job consumer loop, runs native solver
- `src/workers/mod.rs` — Worker pool management (configurable concurrency, multiple worker processes)
- Load project grid configuration and boundary conditions from database
- Run FlowSolver with convergence monitoring
- Progress streaming: worker publishes progress to Redis pub/sub channel
- Error handling: convergence failures (return diagnostics), timeouts, out-of-memory
- S3 result upload: serialize head grid as bincode, upload to S3, return presigned URL

**Day 25: WebSocket for live simulation progress**
- `src/api/ws/mod.rs` — Axum WebSocket upgrade handler
- `src/api/ws/simulation_progress.rs` — Subscribe to simulation progress channel for specific simulation ID
- Client receives JSON messages: `{ progress_pct, iteration, max_head_change, max_residual }`
- Connection management: heartbeat ping/pong every 30s, automatic cleanup on disconnect
- Frontend: `src/hooks/useSimulationProgress.ts` — React hook for WebSocket subscription with auto-reconnect
- Progress bar component with convergence plot

**Day 26: Result loading and visualization**
- Result download from S3 as binary (bincode-serialized head grid: Array3<f64>)
- Deserialize and store in frontend state (simulation result store)
- Update Three.js layer textures with head values (map to RGBA via viridis colormap)
- Contour line generation: marching squares algorithm for head isolines
- Contour overlay on 3D viewer and 2D map view
- Color scale legend with min/max head values

**Day 27: Water budget and zone analysis**
- `src/api/handlers/budget.rs` — Calculate water budget for entire model or specific zone
- Sum inflows/outflows by boundary type: wells (pumping/injection), rivers, recharge, drains, GHBs
- Storage change calculation for transient simulations
- Mass balance error: (inflow - outflow - storage_change) / total_flow × 100%
- Frontend: `src/components/WaterBudget.tsx` — Budget table (inflow/outflow breakdown) and pie chart (D3.js or Recharts)
- Zone-based budget: user draws polygon on map → compute budget for cells within polygon

**Day 28: Drawdown analysis and export**
- Drawdown calculation: initial head - final head for transient simulations
- Drawdown contour map overlay on 3D viewer (separate from head contours)
- Time series animation: slider to view head/drawdown at different time steps
- Export results:
  - CSV: head values at all cells (with coordinates)
  - GeoTIFF: rasterized head/drawdown for GIS import (QGIS, ArcGIS)
  - GeoJSON: contour lines as LineString features
- Export API endpoints with S3 presigned URL generation

### Phase 6 — Contaminant Transport (Days 29–33)

**Day 29: Transport equation solver**
- `solver-core/src/transport/mod.rs` — TransportSolver struct with advection-dispersion solver
- Method of characteristics (MOC) with backward particle tracking at each time step
- Concentration as state variable, velocity field from flow solution (Darcy's law)
- Dispersion coefficient calculation: αL (longitudinal dispersivity), αT (transverse dispersivity)
- Tests: 1D advection (plug flow, compare to analytical), 1D advection-dispersion (Ogata-Banks solution)

**Day 30: Sorption and decay**
- Linear sorption: retardation factor R = 1 + ρb × Kd / θ (bulk density, distribution coefficient, porosity)
- First-order decay: C(t) = C₀ × exp(-λt) where λ is decay rate (1/day)
- Modified transport equation with retardation and decay terms
- Species API: `src/api/handlers/species.rs` — CRUD for contaminant species (name, properties, sorption params)
- Frontend species editor: `src/components/SpeciesManager.tsx` with sorption isotherm selector

**Day 31: Contaminant source modeling**
- Constant concentration sources: specified C at source cells (Dirichlet BC)
- Mass loading sources: kg/day added to cells (converted to concentration change via cell volume)
- Source API and UI for placing sources on map (similar to boundary condition UI)
- Source geometry: point sources (single cell) or area sources (polygon, multiple cells)
- Tests: continuous point source plume development (verify steady-state plume shape)

**Day 32: Transport visualization**
- Concentration texture overlay on 3D aquifer layers (separate from head texture)
- Colormap for concentration: red-yellow gradient (hot colors for contaminants)
- Dual visualization: toggle between head and concentration, or show both with transparency
- Concentration time series charts at observation points (user clicks cell → add to chart)
- Mass balance tracking: total mass in system = source loading - mass removed - mass decayed

**Day 33: Multi-species transport**
- Support for multiple species with different sorption/decay parameters
- Sequential decay chains: parent → daughter → granddaughter (e.g., PCE → TCE → DCE → VC)
- Reaction network solver: matrix of decay rates between species
- Transport API updated to handle species array
- Frontend: species selector for visualization, multiple concentration overlays
- Tests: decay chain with analytical solution comparison (Bateman equations)

### Phase 7 — Billing + Plan Enforcement (Days 34–37)

**Day 34: Stripe integration**
- `src/api/handlers/billing.rs` — Create checkout session, create customer portal session, get subscription status
- `src/api/handlers/webhooks/stripe.rs` — Handle subscription events:
  - `checkout.session.completed` → update user plan to paid tier
  - `customer.subscription.updated` → sync plan changes
  - `customer.subscription.deleted` → downgrade to free
  - `invoice.payment_failed` → send notification
- Plan mapping:
  - Free: 1000 cells, steady-state only, 3 projects, no export
  - Pro ($99/mo): 100K cells, transient + transport, unlimited projects, export, email support
  - Enterprise (custom): multi-user orgs, API access, on-premise deployment option, dedicated support
- Webhook signature verification for security

**Day 35: Usage tracking and plan limits**
- `src/middleware/plan_limits.rs` — Middleware checking plan limits before simulation execution
- `src/services/usage.rs` — Track simulation hours per billing period (calendar month)
- Usage record insertion after each server-side simulation completes (wall_time_ms → hours)
- Usage dashboard API: `src/api/handlers/usage.rs` — Current period usage, historical usage, breakdown by simulation type
- Soft limits: approaching-limit warnings at 80% of quota
- Hard limits: block simulation when quota exceeded

**Day 36: Billing UI**
- `frontend/src/pages/Billing.tsx` — Plan comparison table with feature matrix, current plan display, usage meter
- `frontend/src/components/billing/PlanCard.tsx` — Feature comparison cards with checkmarks
- `frontend/src/components/billing/UsageMeter.tsx` — Simulation hours usage bar (progress bar with threshold indicators)
- Upgrade/downgrade flow: button → Stripe Checkout or Customer Portal redirect
- Upgrade prompt modals when hitting plan limits: "Model has 5000 cells. Upgrade to Pro for models up to 100K cells."
- Invoice history table

**Day 37: Feature gating**
- Gate transient simulation behind Pro plan (free users see "Upgrade to Pro" button)
- Gate transport modeling behind Pro plan
- Gate GeoTIFF/CSV export behind Pro plan (free users can only view results in browser)
- Gate multi-user organizations behind Enterprise plan
- Gate API access (programmatic simulation) behind Enterprise plan
- Show locked feature indicators in UI: lock icon + tooltip with upgrade CTA
- Admin endpoint: override plan for specific users (for beta testers, academic partners, free trials)

### Phase 8 — Validation + Deployment + Launch (Days 38–42)

**Day 38: Solver validation benchmarks**
- Benchmark 1: Theis well solution (confined aquifer radial flow) — automated test with <1% error tolerance
- Benchmark 2: Hantush-Jacob leaky aquifer solution — verify leakance effects
- Benchmark 3: 1D column transport (Ogata-Banks solution) — verify advection-dispersion numerics
- Benchmark 4: Henry saltwater intrusion problem — classic density-dependent flow benchmark
- Benchmark 5: Large-scale regional flow (USGS test case) — verify mass balance and convergence
- Automated test suite: `solver-core/tests/benchmarks.rs` with assertions on expected values
- Generate benchmark report with plots of numerical vs. analytical solutions

**Day 39: Integration testing**
- End-to-end test: create project → build grid → add wells → run simulation (WASM) → view results → export CSV
- API integration tests: auth → project CRUD → boundary conditions → simulation → results download
- WASM solver test: load in headless browser (Playwright), run simulation, verify results match native solver (bit-for-bit or within numerical tolerance)
- WebSocket test: connect → subscribe to progress → receive updates → receive completion notification
- Concurrent simulation test: submit 10 simultaneous server-side jobs, verify all complete correctly and results are correct
- Spatial intersection test: upload shapefile → verify correct cell assignment

**Day 40: Docker and Kubernetes configuration**
- `Dockerfile` — Multi-stage Rust build with cargo-chef for layer caching (reduces rebuild time from 10min to 2min)
- `docker-compose.yml` — Full local dev stack (API, PostgreSQL with PostGIS, Redis, MinIO, spatial service)
- `k8s/` — Kubernetes manifests:
  - `api-deployment.yaml` — API server (3 replicas initially, HPA for auto-scaling)
  - `worker-deployment.yaml` — Simulation workers (HPA based on Redis queue depth >5 jobs)
  - `postgres-statefulset.yaml` — PostgreSQL with persistent volume claim (100GB)
  - `redis-deployment.yaml` — Redis for job queue and pub/sub
  - `spatial-service-deployment.yaml` — Python spatial service (2 replicas)
  - `ingress.yaml` — NGINX ingress with TLS (cert-manager for Let's Encrypt)
- Health check endpoints: `/health/live` (server running), `/health/ready` (database connected)
- Liveness and readiness probes for all services

**Day 41: Monitoring, logging, and polish**
- Prometheus metrics: simulation duration histogram, convergence iteration count, API latency percentiles (p50, p95, p99)
- Grafana dashboards: system health (CPU, memory), simulation throughput (sims/hour), user activity (active users, signups)
- Sentry integration: frontend and backend error tracking with source maps, release tracking
- Structured logging (tracing-subscriber + JSON formatter) for all API requests, solver events, worker activity
- Log aggregation: ship logs to CloudWatch or self-hosted Loki
- UI polish: loading spinners, skeleton screens, error states with retry buttons, empty states with helpful prompts
- Accessibility: keyboard navigation (tab order, focus indicators), ARIA labels on interactive elements, color contrast WCAG AA

**Day 42: Launch preparation and final testing**
- End-to-end smoke test in production environment (staging → production)
- Database backup and restore verification: test restore from backup
- Rate limiting: 100 req/min per user, 5 simulations/min per user (prevent abuse)
- Security audit:
  - JWT validation: verify signature, expiration
  - SQL injection prevention: SQLx parameterized queries
  - CORS configuration: allow only frontend domain
  - CSP headers: prevent XSS attacks
  - S3 bucket permissions: private with presigned URLs only
- Landing page: product overview, pricing table, signup form, demo video
- Documentation: getting started guide (5-minute tutorial), video tutorials, API reference (for Enterprise), FAQ
- Email templates: welcome email, trial expiration reminder, payment failure notification
- Deploy to production: blue-green deployment with health check verification
- Enable monitoring alerts: PagerDuty or Opsgenie for critical errors
- Announce launch: Product Hunt, HN Show, LinkedIn, Twitter, environmental engineering forums

---

## Critical Files

```
enviromod/
├── solver-core/                           # Shared solver library (native + WASM)
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── grid.rs                        # Grid structure, cell indexing, conductance calculations
│   │   ├── pcg.rs                         # Preconditioned Conjugate Gradient solver
│   │   ├── flow/
│   │   │   ├── mod.rs                     # FlowSolver main logic, matrix assembly
│   │   │   ├── steady.rs                  # Steady-state solver with Newton iteration
│   │   │   └── transient.rs               # Transient solver with adaptive time stepping
│   │   ├── boundary/
│   │   │   ├── mod.rs                     # BoundaryPackage trait definition
│   │   │   ├── well.rs                    # Well package (pumping/injection)
│   │   │   ├── river.rs                   # River package (head-dependent flux)
│   │   │   ├── recharge.rs                # Recharge package (areal flux)
│   │   │   ├── drain.rs                   # Drain package
│   │   │   ├── ghb.rs                     # General head boundary
│   │   │   └── chd.rs                     # Constant head boundary
│   │   ├── transport/
│   │   │   ├── mod.rs                     # TransportSolver main logic
│   │   │   ├── advection.rs               # Method of characteristics
│   │   │   ├── dispersion.rs              # Dispersion coefficient calculation
│   │   │   ├── sorption.rs                # Sorption isotherms (linear, Freundlich, Langmuir)
│   │   │   └── reactions.rs               # Decay and reaction networks
│   │   └── particle.rs                    # Particle tracking (MODPATH-equivalent, Pollock's method)
│   └── tests/
│       ├── benchmarks.rs                  # Solver validation benchmarks (Theis, Hantush-Jacob, etc.)
│       ├── analytical_solutions.rs        # Analytical solution comparisons
│       └── grid_tests.rs                  # Grid and conductance calculation tests
│
├── solver-wasm/                           # WASM compilation target
│   ├── Cargo.toml
│   └── src/
│       └── lib.rs                         # WASM entry points (wasm-bindgen)
│
├── enviromod-api/                         # Rust API server
│   ├── Cargo.toml
│   ├── src/
│   │   ├── main.rs                        # Axum app setup, router, startup, graceful shutdown
│   │   ├── config.rs                      # Environment-based configuration
│   │   ├── state.rs                       # AppState (PgPool, Redis, S3Client)
│   │   ├── error.rs                       # ApiError types with IntoResponse
│   │   ├── auth/
│   │   │   ├── mod.rs                     # JWT middleware, Claims struct
│   │   │   └── oauth.rs                   # Google/GitHub OAuth handlers
│   │   ├── api/
│   │   │   ├── router.rs                  # Route definitions with middleware
│   │   │   ├── handlers/
│   │   │   │   ├── auth.rs                # Register, login, OAuth callback
│   │   │   │   ├── users.rs               # Profile CRUD
│   │   │   │   ├── projects.rs            # Project CRUD, fork, share
│   │   │   │   ├── layers.rs              # Model layer CRUD
│   │   │   │   ├── boundary.rs            # Boundary condition CRUD
│   │   │   │   ├── simulation.rs          # Create/get/cancel simulation
│   │   │   │   ├── species.rs             # Contaminant species CRUD
│   │   │   │   ├── budget.rs              # Water budget calculation
│   │   │   │   ├── results.rs             # Simulation result retrieval
│   │   │   │   ├── billing.rs             # Stripe checkout/portal
│   │   │   │   ├── usage.rs               # Usage tracking and reporting
│   │   │   │   ├── export.rs              # GeoTIFF/CSV/GeoJSON export
│   │   │   │   └── webhooks/
│   │   │   │       └── stripe.rs          # Stripe webhook handler
│   │   │   └── ws/
│   │   │       ├── mod.rs                 # WebSocket upgrade handler
│   │   │       └── simulation_progress.rs # Live progress streaming
│   │   ├── middleware/
│   │   │   └── plan_limits.rs             # Plan enforcement middleware
│   │   ├── services/
│   │   │   ├── usage.rs                   # Usage tracking service
│   │   │   └── s3.rs                      # S3 client helpers (presigned URLs)
│   │   └── workers/
│   │       ├── mod.rs                     # Worker pool management
│   │       └── simulation_worker.rs       # Server-side simulation execution
│   ├── migrations/
│   │   └── 001_initial.sql                # Full database schema with PostGIS
│   └── tests/
│       ├── api_integration.rs             # API integration tests
│       └── simulation_e2e.rs              # End-to-end simulation tests
│
├── spatial-service/                       # Python FastAPI (GDAL processing)
│   ├── requirements.txt
│   ├── main.py                            # FastAPI app with CORS
│   ├── processors/
│   │   ├── rasterize.py                   # GeoTIFF rasterization and resampling
│   │   ├── shapefile.py                   # Shapefile parsing and intersection
│   │   └── interpolate.py                 # DEM interpolation (kriging, IDW)
│   └── Dockerfile
│
├── frontend/                              # React frontend
│   ├── package.json
│   ├── vite.config.ts
│   ├── src/
│   │   ├── App.tsx                        # Router, providers, layout
│   │   ├── main.tsx                       # Entry point
│   │   ├── stores/
│   │   │   ├── authStore.ts               # Auth state (JWT, user profile)
│   │   │   ├── projectStore.ts            # Project state
│   │   │   ├── modelStore.ts              # Grid, layers, boundary conditions
│   │   │   └── simulationStore.ts         # Simulation state and results
│   │   ├── hooks/
│   │   │   ├── useSimulationProgress.ts   # WebSocket hook for live progress
│   │   │   └── useWasmSolver.ts           # WASM solver hook (load + run)
│   │   ├── lib/
│   │   │   ├── api.ts                     # Axios API client
│   │   │   └── wasmLoader.ts              # WASM bundle loading
│   │   ├── pages/
│   │   │   ├── Dashboard.tsx              # Project list with thumbnails
│   │   │   ├── ModelBuilder.tsx           # Grid and layer editor
│   │   │   ├── Boundaries.tsx             # Boundary condition editor
│   │   │   ├── Simulation.tsx             # Run simulation + results viewer
│   │   │   ├── Billing.tsx                # Plan management and usage
│   │   │   ├── Login.tsx                  # Auth pages
│   │   │   └── Register.tsx
│   │   ├── components/
│   │   │   ├── MapViewer.tsx              # Mapbox GL map with site overlay
│   │   │   ├── GridBuilder.tsx            # Grid configuration form
│   │   │   ├── GridPreview.tsx            # 2D grid cell preview
│   │   │   ├── LayerManager.tsx           # Add/edit layers list
│   │   │   ├── LayerEditor.tsx            # Layer property editor
│   │   │   ├── AquiferViewer3D.tsx        # Three.js 3D visualization
│   │   │   ├── BoundaryConditionManager.tsx # BC list and controls
│   │   │   ├── WellEditor.tsx             # Well placement on map
│   │   │   ├── RiverEditor.tsx            # River polyline drawing
│   │   │   ├── RechargeEditor.tsx         # Recharge zone drawing
│   │   │   ├── SpeciesManager.tsx         # Contaminant species editor
│   │   │   ├── SourceEditor.tsx           # Contaminant source placement
│   │   │   ├── WaterBudget.tsx            # Water budget visualization
│   │   │   ├── ContourMap.tsx             # Head/concentration contour overlay
│   │   │   ├── ParticlePathlines.tsx      # Particle tracking visualization
│   │   │   ├── SimulationToolbar.tsx      # Run/stop/export controls
│   │   │   ├── ConvergencePlot.tsx        # Live convergence monitoring chart
│   │   │   ├── TimeSeriesChart.tsx        # Head/concentration time series
│   │   │   └── billing/
│   │   │       ├── PlanCard.tsx           # Plan feature comparison card
│   │   │       └── UsageMeter.tsx         # Usage meter with progress bar
│   │   └── utils/
│   │       ├── colormap.ts                # Viridis/plasma colormaps
│   │       ├── contouring.ts              # Marching squares for contour lines
│   │       └── units.ts                   # Unit conversion helpers
│   └── public/
│       └── wasm/                          # WASM solver bundle (loaded at runtime)
│
├── k8s/                                   # Kubernetes manifests
│   ├── api-deployment.yaml
│   ├── worker-deployment.yaml
│   ├── postgres-statefulset.yaml
│   ├── redis-deployment.yaml
│   ├── spatial-service-deployment.yaml
│   ├── ingress.yaml
│   └── hpa.yaml                           # Horizontal Pod Autoscaler configs
│
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

### Benchmark 1: Theis Well Solution (Confined Aquifer)

**Setup:** Confined aquifer, T=100 m²/day, S=0.0001, well Q=500 m³/day

**Expected drawdown at r=100m, t=1 day:** s = (Q/4πT) × W(u) where u = r²S/(4Tt) = 0.0025, W(u) ≈ 5.64
**Result:** s = **2.24 m**

**Tolerance:** < 1%

### Benchmark 2: Hantush-Jacob Leaky Aquifer

**Setup:** T=200 m²/day, S=0.0002, leakance=0.001/day, Q=1000 m³/day

**Expected drawdown at r=50m, t=10 days:** s ≈ **1.85 m** (Hantush well function)

**Tolerance:** < 2%

### Benchmark 3: 1D Advection-Dispersion (Ogata-Banks)

**Setup:** v=1 m/day, D=0.1 m²/day, C₀=100 mg/L

**Expected at x=10m, t=15 days:** C(10, 15) ≈ **69.1 mg/L** (Ogata-Banks analytical)

**Tolerance:** < 3%

### Benchmark 4: Henry Saltwater Intrusion

**Setup:** Freshwater-saltwater interface in coastal aquifer

**Expected interface at x=0.5L:** z ≈ **0.35 m** below surface
**Expected max salinity:** C_max ≈ **35 g/L**

**Tolerance:** Interface < 5%, salinity < 5%

### Benchmark 5: Regional Flow Mass Balance

**Setup:** 100×100×5 grid, heterogeneous K field (log-normal)

**Expected:** Water balance error < **0.1%**
**Expected:** Convergence < **50 iterations** for steady-state

**Tolerance:** Mass balance < 0.1%, iterations < 100

---

## Verification

### Integration Testing Checklist

1. Auth flow — Register → login → JWT → authorized calls → refresh → logout
2. Project CRUD — Create → update grid → save → reload → verify persistence
3. Grid building — Set dimensions → define spacing → upload GeoTIFF elevation
4. Layer configuration — Add layers → set K/S/Sy → verify PostGIS storage
5. Boundary conditions — Place well → draw river → verify spatial intersection
6. WASM simulation — 1K-cell model → WASM solver → 3D visualization
7. Server simulation — 50K-cell model → job queue → WebSocket progress → S3 results
8. 3D visualization — Load results → head texture → cross-section → contours
9. Particle tracking — Run flow → place particles → pathlines → GeoJSON export
10. Water budget — Run simulation → compute budget → verify mass balance
11. Transport — Define species → place source → run → concentration plume
12. Plan limits — Free user → 2K-cell model → upgrade prompt
13. Billing — Subscribe to Pro → webhook → plan updated → limits lifted
14. Concurrent users — 10 simultaneous server jobs → all complete
15. Error handling — Non-convergent model → diagnostics → no crash

### Performance Benchmarks

| Operation | Target | Method |
|-----------|--------|--------|
| WASM solver: 1K-cell steady | <1s | Browser benchmark |
| WASM solver: 10K-cell steady | <3s | Browser benchmark |
| Server: 50K-cell steady | <30s | 4 cores |
| Server: 100K-cell transient (50 steps) | <5min | 8 cores |
| 3D viewer: 10-layer render | 60 FPS | Chrome FPS counter |
| Particle tracking: 100 particles, 1000 steps | <2s | Browser/server |
| API: create simulation | <200ms | p95 latency |
| PostGIS spatial intersection | <100ms | GIST index |
| WASM bundle load (cached) | <500ms | Service worker |
| WASM bundle load (cold) | <3s | CDN, 2MB gzipped |
| GeoTIFF rasterization (10MB) | <5s | GDAL |

---

## Deployment Architecture

### Infrastructure

```
┌─────────────────────────────────────────┐
│           CloudFront CDN                 │
│  Frontend SPA | WASM Bundle | GeoTIFFs  │
└──────────────┬──────────────────────────┘
               │
        ┌──────┴──────┐
        │   AWS ALB   │
        └──────┬──────┘
               │
    ┌──────────┼──────────┬─────────────┐
    ▼          ▼          ▼             ▼
┌────────┐ ┌────────┐ ┌────────┐ ┌──────────┐
│ API ×3 │ │ API ×3 │ │ API ×3 │ │ Spatial  │
│ (HPA)  │ │        │ │        │ │ Svc ×2   │
└───┬────┘ └───┬────┘ └───┬────┘ └──────────┘
    │          │          │
    └──────────┼──────────┘
               │
    ┌──────────┼──────────┐
    ▼          ▼          ▼
┌──────────┐ ┌────────┐ ┌────────┐
│PostgreSQL│ │ Redis  │ │   S3   │
│ (PostGIS)│ │(Queue) │ │Results │
└──────────┘ └───┬────┘ └────────┘
                 │
    ┌────────────┼────────────┐
    ▼            ▼            ▼
┌─────────┐ ┌─────────┐ ┌─────────┐
│Worker×20│ │Worker×20│ │Worker×20│
│ (HPA)   │ │         │ │         │
└─────────┘ └─────────┘ └─────────┘
```

### Scaling Strategy

- **API**: HPA at 70% CPU, min 3, max 10
- **Workers**: HPA based on Redis queue depth (>5 jobs scale up), c7g.2xlarge (8 vCPU, 16GB)
- **Database**: RDS PostgreSQL r6g.xlarge with read replicas
- **Redis**: ElastiCache r6g.large cluster (1 primary + 2 replicas)

### S3 Lifecycle Policy

- **Results**: Standard → IA (30 days) → Glacier (90 days) → Delete (365 days)
- **Rasters**: Standard → IA (90 days)
- **WASM**: Versioned, delete old versions after 30 days

---

## Post-MVP Roadmap

| Feature | Description | Priority |
|---------|-------------|----------|
| Automatic Calibration | PEST-like parameter estimation with pilot points, Gauss-Levenberg-Marquardt optimization, uncertainty quantification | High |
| Remediation Design | Pump-and-treat optimization, capture zone delineation, PRB placement, cost-benefit analysis | High |
| Surface Water Quality | 1D river modeling (Streeter-Phelps), lake stratification, coupled surface-groundwater | High |
| Reactive Transport | Multi-species reaction networks, mineral precipitation/dissolution, redox, microbial | Medium |
| Real-Time Collaboration | CRDT-based model editing, cursor presence, shared results | Medium |
| Density-Dependent Flow | Saltwater intrusion, coupled flow-transport, variable-density solver | Medium |
| DNAPL Modeling | Dense NAPL dissolution, partitioning, mass transfer for chlorinated solvents | Medium |
| Uncertainty Analysis | Monte Carlo parameter sweeps, sensitivity analysis, probabilistic risk | Low |
