# 64. MineAssay — Mining Engineering and Ore Reserve Estimation Platform

## Implementation Plan

**MVP Scope:** Browser-based drillhole data import (collar, survey, assay, lithology CSV/Excel) with validation, 3D visualization via Three.js/WebGL showing downhole traces with color-coded assay intervals, implicit geological modeling using radial basis function (RBF) interpolation compiled to WebAssembly for lithology boundary surface generation from sparse drillhole intersections, block model generation with configurable cells (5m default), geostatistical resource estimation implementing ordinary kriging with experimental variogram calculation and interactive model fitting (spherical/exponential/Gaussian), kriging estimation for Au/Cu/grade with octant search and min/max sample constraints, 2D/3D section views with drillhole intersections and block model slices, resource classification (measured/indicated/inferred) based on kriging variance, PostgreSQL+PostGIS for drillhole database with S3/MinIO for block model arrays and geological meshes, Stripe billing (Exploration $149/mo, Planning $349/user/mo, Enterprise custom).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async, Tower middleware, JWT auth |
| Geological Modeling | Rust (native + WASM) | RBF implicit, wireframe ops, block models |
| Geostatistics Solver | Rust (native + WASM) | Variogram fitting, kriging, SGS (post-MVP) |
| WASM Compilation | `wasm-pack` + `wasm-bindgen` | Client RBF <10K samples, kriging <50K cells |
| Scientific Computing | Python 3.12 (FastAPI) | Variogram diagnostics, stats, PDF reports |
| Database | PostgreSQL 16 + PostGIS | Drillhole data, assays, spatial queries |
| ORM / Query | SQLx (Rust) | Compile-time checked queries |
| Auth | Custom JWT + OAuth 2.0 | Google/GitHub OAuth, bcrypt |
| Object Storage | AWS S3 / MinIO | Block arrays (binary), geo meshes (OBJ/STL) |
| Frontend Framework | React 19 + TypeScript | Vite, Zustand state |
| 3D Visualization | Three.js + React Three Fiber | Drillholes, surfaces, blocks |
| 2D Sections | Canvas API + Konva | Cross-sections, block slices |
| Real-time | WebSocket (Axum) | Kriging/RBF progress |
| Job Queue | Redis 7 + Tokio tasks | Server kriging jobs, model generation |
| Billing | Stripe | Checkout, Customer Portal, Webhooks |
| CDN | CloudFront | WASM bundles, static assets |
| Monitoring | Prometheus + Grafana, Sentry | Metrics, error tracking |
| CI/CD | GitHub Actions | WASM build, tests, Docker push |

### Key Architecture Decisions

1. **Hybrid client/server geostatistics with WASM threshold at 50K cells**: Block models ≤50K cells run kriging in browser via WASM (instant feedback for small exploration projects), >50K cells use Rust-native server with Rayon parallel execution. Covers 80%+ of junior mining exploration use cases.

2. **RBF implicit modeling in Rust rather than proprietary algorithms**: Radial basis function interpolation with thin-plate spline/multiquadric kernels for smooth lithology boundaries. System `∑ φ(||x - xᵢ||) wᵢ = f(x)` solved via SVD/Cholesky, evaluated on-demand for Marching Cubes isosurface. Compiles to WASM, comparable to Leapfrog Geo at 1/10th cost.

3. **Three.js for 3D with instanced rendering**: Block models with 100K+ cells use instanced meshes (1 draw call vs 100K). Drillhole traces are LineSegments with vertex colors. Geological surfaces use custom shaders for depth coloring.

4. **Ordinary kriging with octant search using nalgebra**: Kriging solves `[K]{λ} = {k}` where K is covariance matrix. Local search (50-200m radius, min 8, max 24 samples, 2+/octant). Cholesky decomposition via nalgebra. Kriging variance `σ²ₖ = C₀ - {λ}ᵀ{k} - μ` for classification.

5. **PostgreSQL+PostGIS for drillholes, S3 for block models**: Drillhole tables use composite indexes (hole_id, from_depth, to_depth). PostGIS for spatial queries. Block model arrays (1-100 MB) stored as binary (MessagePack/Arrow) in S3, PostgreSQL holds metadata.

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "postgis";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email TEXT UNIQUE NOT NULL,
    password_hash TEXT,
    name TEXT NOT NULL,
    avatar_url TEXT,
    auth_provider TEXT DEFAULT 'email',
    auth_provider_id TEXT,
    plan TEXT NOT NULL DEFAULT 'exploration',
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX users_email_idx ON users(email);

CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id),
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    location GEOGRAPHY(POINT),
    coordinate_system TEXT DEFAULT 'WGS84',
    settings JSONB DEFAULT '{}',
    is_public BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);
CREATE INDEX projects_location_idx ON projects USING GIST(location);

CREATE TABLE drillholes (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    hole_id TEXT NOT NULL,
    collar_location GEOGRAPHY(POINTZ) NOT NULL,
    collar_x DOUBLE PRECISION NOT NULL,
    collar_y DOUBLE PRECISION NOT NULL,
    collar_z DOUBLE PRECISION NOT NULL,
    end_of_hole DOUBLE PRECISION NOT NULL,
    drill_date DATE,
    azimuth DOUBLE PRECISION,
    dip DOUBLE PRECISION,
    hole_type TEXT,
    status TEXT DEFAULT 'active',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(project_id, hole_id)
);
CREATE INDEX drillholes_project_idx ON drillholes(project_id);
CREATE INDEX drillholes_location_idx ON drillholes USING GIST(collar_location);

CREATE TABLE drillhole_surveys (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    drillhole_id UUID NOT NULL REFERENCES drillholes(id) ON DELETE CASCADE,
    depth DOUBLE PRECISION NOT NULL,
    azimuth DOUBLE PRECISION NOT NULL,
    dip DOUBLE PRECISION NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX surveys_drillhole_depth_idx ON drillhole_surveys(drillhole_id, depth);

CREATE TABLE assays (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    drillhole_id UUID NOT NULL REFERENCES drillholes(id) ON DELETE CASCADE,
    from_depth DOUBLE PRECISION NOT NULL,
    to_depth DOUBLE PRECISION NOT NULL,
    au_ppm DOUBLE PRECISION,
    cu_pct DOUBLE PRECISION,
    ag_ppm DOUBLE PRECISION,
    attributes JSONB DEFAULT '{}',
    sample_id TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CHECK (to_depth > from_depth)
);
CREATE INDEX assays_drillhole_idx ON assays(drillhole_id, from_depth, to_depth);
CREATE INDEX assays_au_idx ON assays(au_ppm) WHERE au_ppm IS NOT NULL;

CREATE TABLE lithology (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    drillhole_id UUID NOT NULL REFERENCES drillholes(id) ON DELETE CASCADE,
    from_depth DOUBLE PRECISION NOT NULL,
    to_depth DOUBLE PRECISION NOT NULL,
    lithology_code TEXT NOT NULL,
    description TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CHECK (to_depth > from_depth)
);
CREATE INDEX lithology_drillhole_idx ON lithology(drillhole_id, from_depth, to_depth);

CREATE TABLE geological_models (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    model_type TEXT NOT NULL,
    lithology_code TEXT,
    parameters JSONB NOT NULL DEFAULT '{}',
    mesh_url TEXT,
    mesh_format TEXT,
    mesh_vertices INTEGER,
    mesh_faces INTEGER,
    status TEXT NOT NULL DEFAULT 'pending',
    error_message TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX models_project_idx ON geological_models(project_id);

CREATE TABLE block_models (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    origin_x DOUBLE PRECISION NOT NULL,
    origin_y DOUBLE PRECISION NOT NULL,
    origin_z DOUBLE PRECISION NOT NULL,
    cell_size_x DOUBLE PRECISION NOT NULL DEFAULT 5.0,
    cell_size_y DOUBLE PRECISION NOT NULL DEFAULT 5.0,
    cell_size_z DOUBLE PRECISION NOT NULL DEFAULT 5.0,
    n_cells_x INTEGER NOT NULL,
    n_cells_y INTEGER NOT NULL,
    n_cells_z INTEGER NOT NULL,
    total_cells INTEGER NOT NULL,
    attributes JSONB NOT NULL DEFAULT '[]',
    cells_url TEXT,
    cells_format TEXT DEFAULT 'msgpack',
    status TEXT NOT NULL DEFAULT 'pending',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX block_models_project_idx ON block_models(project_id);

CREATE TABLE estimations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    block_model_id UUID NOT NULL REFERENCES block_models(id) ON DELETE CASCADE,
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    method TEXT NOT NULL,
    attribute TEXT NOT NULL,
    parameters JSONB NOT NULL DEFAULT '{}',
    status TEXT NOT NULL DEFAULT 'pending',
    cells_processed INTEGER DEFAULT 0,
    total_cells INTEGER NOT NULL,
    results_url TEXT,
    statistics JSONB,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    wall_time_ms INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX estimations_block_model_idx ON estimations(block_model_id);
CREATE INDEX estimations_status_idx ON estimations(status);

CREATE TABLE variograms (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    attribute TEXT NOT NULL,
    domain_filter JSONB,
    experimental_data JSONB NOT NULL,
    model JSONB NOT NULL,
    fitted_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX variograms_project_idx ON variograms(project_id);
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
    #[sqlx(skip)] #[serde(skip)]
    pub password_hash: Option<String>,
    pub name: String,
    pub plan: String,
    pub stripe_customer_id: Option<String>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Project {
    pub id: Uuid,
    pub owner_id: Uuid,
    pub name: String,
    pub description: String,
    pub coordinate_system: String,
    pub settings: serde_json::Value,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Drillhole {
    pub id: Uuid,
    pub project_id: Uuid,
    pub hole_id: String,
    pub collar_x: f64,
    pub collar_y: f64,
    pub collar_z: f64,
    pub end_of_hole: f64,
    pub azimuth: Option<f64>,
    pub dip: Option<f64>,
    pub status: String,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Assay {
    pub id: Uuid,
    pub drillhole_id: Uuid,
    pub from_depth: f64,
    pub to_depth: f64,
    pub au_ppm: Option<f64>,
    pub cu_pct: Option<f64>,
    pub sample_id: Option<String>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct BlockModel {
    pub id: Uuid,
    pub project_id: Uuid,
    pub name: String,
    pub origin_x: f64,
    pub origin_y: f64,
    pub origin_z: f64,
    pub cell_size_x: f64,
    pub cell_size_y: f64,
    pub cell_size_z: f64,
    pub n_cells_x: i32,
    pub n_cells_y: i32,
    pub n_cells_z: i32,
    pub total_cells: i32,
    pub cells_url: Option<String>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Estimation {
    pub id: Uuid,
    pub block_model_id: Uuid,
    pub method: String,
    pub attribute: String,
    pub parameters: serde_json::Value,
    pub status: String,
    pub cells_processed: i32,
    pub total_cells: i32,
    pub results_url: Option<String>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct VariogramModel {
    pub structures: Vec<VariogramStructure>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct VariogramStructure {
    pub structure_type: String,  // spherical | exponential | gaussian
    pub nugget: f64,
    pub sill: f64,
    pub range_major: f64,
    pub range_minor: f64,
    pub range_vertical: f64,
    pub azimuth: f64,
    pub dip: f64,
}

#[derive(Debug, Deserialize)]
pub struct KrigingParams {
    pub search_radius_major: f64,
    pub search_radius_minor: f64,
    pub search_radius_vertical: f64,
    pub search_azimuth: f64,
    pub min_samples: usize,
    pub max_samples: usize,
    pub min_samples_per_octant: usize,
    pub discretization_points: usize,
    pub variogram_model: VariogramModel,
}
```

---

## Solver Architecture Deep-Dive

### Geostatistical Equations

MineAssay implements **Ordinary Kriging (OK)** for resource estimation. For target cell `x₀` estimated from `n` samples at `xᵢ`:

```
┌                          ┐ ┌    ┐   ┌        ┐
│ C(x₁,x₁)  ... C(x₁,xₙ) 1│ │ λ₁ │   │ C(x₁,x₀)│
│    ⋮            ⋮      ⋮│ │  ⋮ │ = │    ⋮    │
│ C(xₙ,x₁)  ... C(xₙ,xₙ) 1│ │ λₙ │   │ C(xₙ,x₀)│
│    1       ...    1     0│ │ μ  │   │    1    │
└                          ┘ └    ┘   └        ┘

Estimated value:  z*(x₀) = ∑ λᵢ · z(xᵢ)
Kriging variance: σ²ₖ(x₀) = C(0) - ∑ λᵢ·C(xᵢ,x₀) - μ
```

**C(xᵢ,xⱼ)** = covariance from variogram: `C(h) = C₀ - γ(h)`

**Spherical variogram:**
```
γ(h) = C₀                                    if h = 0
γ(h) = C₀ + C₁·[1.5·(h/a) - 0.5·(h/a)³]     if 0 < h ≤ a
γ(h) = C₀ + C₁                               if h > a
```

**Anisotropic distance:** `h² = (Δx/rₓ)² + (Δy/rᵧ)² + (Δz/rᵧ)²` with rotation by azimuth/dip.

**Search neighborhood:** Ellipsoid `(Δx'/Rₓ)² + (Δy'/Rᵧ)² + (Δz'/Rᵧ)² ≤ 1` with constraints: min 8, max 24 samples, ≥2/octant.

### RBF Implicit Modeling

For lithology boundaries, solve RBF system for implicit surface `f(x) = 0`:

```
f(x) = ∑ᵢ wᵢ · φ(||x - xᵢ||) + p(x)

where φ(r) = r² log(r)  (thin-plate spline)
      p(x) = a₀ + a₁x + a₂y + a₃z  (linear polynomial)
```

System:
```
┌          ┐ ┌  ┐   ┌  ┐
│  Φ    P  │ │ w │   │ f │
│  Pᵀ   0  │ │ a │ = │ 0 │
└          ┘ └  ┘   └  ┘

where Φᵢⱼ = φ(||xᵢ - xⱼ||)
      f = +1 inside, -1 outside lithology
```

Solved via Cholesky/SVD, then isosurface extracted via Marching Cubes on 3D grid.

---

## Architecture Deep-Dives

### 1. Drillhole Import Handler (Rust/Axum)

```rust
// src/api/handlers/drillholes.rs
use axum::{extract::{Path, State, Multipart}, http::StatusCode};
use uuid::Uuid;
use crate::{import::csv_parser, desurvey, state::AppState, auth::Claims, error::ApiError};

pub async fn import_drillholes(
    State(state): State<AppState>,
    claims: Claims,
    Path(project_id): Path<Uuid>,
    mut multipart: Multipart,
) -> Result<impl axum::response::IntoResponse, ApiError> {
    // Verify project ownership
    let project = sqlx::query_as!(
        crate::db::models::Project,
        "SELECT * FROM projects WHERE id = $1 AND owner_id = $2",
        project_id, claims.user_id
    ).fetch_optional(&state.db).await?
    .ok_or(ApiError::NotFound("Project not found"))?;

    // Parse CSV/Excel file
    let mut file_bytes = Vec::new();
    let mut import_type = "collar".to_string();
    while let Some(field) = multipart.next_field().await? {
        match field.name() {
            Some("file") => file_bytes = field.bytes().await?.to_vec(),
            Some("import_type") => import_type = field.text().await?,
            _ => {}
        }
    }

    let records = csv_parser::parse_csv(&file_bytes, &import_type)?;

    // Validate data
    let validation_errors = crate::import::validation::validate_records(&records, &import_type)?;
    if !validation_errors.is_empty() {
        return Ok((StatusCode::BAD_REQUEST, axum::Json(serde_json::json!({
            "errors": validation_errors
        }))));
    }

    // Import based on type
    let imported_count = match import_type.as_str() {
        "collar" => import_collars(&state, project_id, &records).await?,
        "assay" => import_assays(&state, project_id, &records).await?,
        "survey" => import_surveys(&state, project_id, &records).await?,
        _ => return Err(ApiError::BadRequest("Unknown import type")),
    };

    Ok((StatusCode::OK, axum::Json(serde_json::json!({"imported": imported_count}))))
}

async fn import_collars(state: &AppState, project_id: Uuid, records: &[serde_json::Value]) -> Result<usize, ApiError> {
    let mut count = 0;
    for record in records {
        let hole_id = record["hole_id"].as_str().ok_or(ApiError::BadRequest("Missing hole_id"))?;
        let x = record["easting"].as_f64().ok_or(ApiError::BadRequest("Missing easting"))?;
        let y = record["northing"].as_f64().ok_or(ApiError::BadRequest("Missing northing"))?;
        let z = record["elevation"].as_f64().ok_or(ApiError::BadRequest("Missing elevation"))?;
        let depth = record["depth"].as_f64().ok_or(ApiError::BadRequest("Missing depth"))?;

        sqlx::query!(
            r#"INSERT INTO drillholes (project_id, hole_id, collar_location, collar_x, collar_y, collar_z, end_of_hole)
               VALUES ($1, $2, ST_SetSRID(ST_MakePoint($3, $4, $5), 4326), $3, $4, $5, $6)
               ON CONFLICT (project_id, hole_id) DO UPDATE SET collar_x = $3, collar_y = $4, collar_z = $5"#,
            project_id, hole_id, x, y, z, depth
        ).execute(&state.db).await?;
        count += 1;
    }
    Ok(count)
}
```

### 2. Ordinary Kriging Solver (Rust — WASM/native)

```rust
// geostat-core/src/kriging.rs
use nalgebra::{DMatrix, DVector};
use crate::variogram::{Variogram, evaluate_variogram};

#[derive(Debug, Clone)]
pub struct Sample {
    pub x: f64,
    pub y: f64,
    pub z: f64,
    pub value: f64,
}

#[derive(Debug)]
pub struct KrigingResult {
    pub estimate: f64,
    pub variance: f64,
    pub samples_used: usize,
}

pub fn krige_cell(
    cell: &(f64, f64, f64),
    all_samples: &[Sample],
    variogram: &Variogram,
    search_radius: f64,
    min_samples: usize,
    max_samples: usize,
) -> Option<KrigingResult> {
    // Find nearby samples within search radius
    let mut nearby: Vec<&Sample> = all_samples.iter()
        .filter(|s| {
            let dx = s.x - cell.0;
            let dy = s.y - cell.1;
            let dz = s.z - cell.2;
            (dx*dx + dy*dy + dz*dz).sqrt() <= search_radius
        })
        .collect();

    if nearby.len() < min_samples {
        return None;
    }

    nearby.truncate(max_samples);
    let n = nearby.len();

    // Build covariance matrix K (n+1 × n+1)
    let mut k_matrix = DMatrix::zeros(n + 1, n + 1);
    for i in 0..n {
        for j in 0..n {
            let dx = nearby[i].x - nearby[j].x;
            let dy = nearby[i].y - nearby[j].y;
            let dz = nearby[i].z - nearby[j].z;
            let h = (dx*dx + dy*dy + dz*dz).sqrt();
            let gamma = evaluate_variogram(variogram, h);
            let cov = variogram.sill() - gamma;
            k_matrix[(i, j)] = cov;
        }
        k_matrix[(i, n)] = 1.0;  // Unbiasedness
        k_matrix[(n, i)] = 1.0;
    }

    // Build RHS (covariances to target)
    let mut rhs = DVector::zeros(n + 1);
    for i in 0..n {
        let dx = nearby[i].x - cell.0;
        let dy = nearby[i].y - cell.1;
        let dz = nearby[i].z - cell.2;
        let h = (dx*dx + dy*dy + dz*dz).sqrt();
        let gamma = evaluate_variogram(variogram, h);
        rhs[i] = variogram.sill() - gamma;
    }
    rhs[n] = 1.0;

    // Solve using Cholesky
    let chol = k_matrix.clone().cholesky()?;
    let weights = chol.solve(&rhs);

    // Compute estimate and variance
    let mut estimate = 0.0;
    for i in 0..n {
        estimate += weights[i] * nearby[i].value;
    }

    let mut variance = variogram.sill();
    for i in 0..n {
        variance -= weights[i] * rhs[i];
    }
    variance -= weights[n];  // Lagrange multiplier
    variance = variance.max(0.0);

    Some(KrigingResult { estimate, variance, samples_used: n })
}

// Parallel version for server
#[cfg(not(target_arch = "wasm32"))]
pub fn krige_block_model(
    cells: &[(f64, f64, f64)],
    samples: &[Sample],
    variogram: &Variogram,
    params: &crate::KrigingParams,
) -> Vec<Option<KrigingResult>> {
    use rayon::prelude::*;
    cells.par_iter()
        .map(|cell| krige_cell(cell, samples, variogram, params.search_radius_major, params.min_samples, params.max_samples))
        .collect()
}
```

### 3. RBF Implicit Modeling (Rust + WASM)

```rust
// modeling-core/src/rbf.rs
use nalgebra::{DMatrix, DVector};

#[derive(Debug, Clone)]
pub struct RbfPoint {
    pub x: f64,
    pub y: f64,
    pub z: f64,
    pub value: f64,  // +1 inside, -1 outside
}

pub struct RbfModel {
    pub points: Vec<RbfPoint>,
    pub weights: DVector<f64>,
    pub poly_coeffs: DVector<f64>,
}

fn thin_plate_spline(r: f64) -> f64 {
    if r < 1e-12 { 0.0 } else { r * r * r.ln() }
}

pub fn fit_rbf(points: &[RbfPoint]) -> Result<RbfModel, String> {
    let n = points.len();
    if n < 4 { return Err("Need ≥4 points".to_string()); }

    let mut system = DMatrix::zeros(n + 4, n + 4);
    let mut rhs = DVector::zeros(n + 4);

    // Fill Φ block
    for i in 0..n {
        for j in 0..n {
            let dx = points[i].x - points[j].x;
            let dy = points[i].y - points[j].y;
            let dz = points[i].z - points[j].z;
            let r = (dx*dx + dy*dy + dz*dz).sqrt();
            system[(i, j)] = thin_plate_spline(r);
        }
    }

    // Fill P block (polynomial basis: [1, x, y, z])
    for i in 0..n {
        system[(i, n)] = 1.0;
        system[(i, n+1)] = points[i].x;
        system[(i, n+2)] = points[i].y;
        system[(i, n+3)] = points[i].z;
        system[(n, i)] = 1.0;
        system[(n+1, i)] = points[i].x;
        system[(n+2, i)] = points[i].y;
        system[(n+3, i)] = points[i].z;
    }

    // Fill RHS
    for i in 0..n {
        rhs[i] = points[i].value;
    }

    // Solve via Cholesky or SVD
    let solution = if let Some(chol) = system.clone().cholesky() {
        chol.solve(&rhs)
    } else {
        let svd = system.svd(true, true);
        svd.pseudo_inverse(1e-10).ok_or("SVD failed")? * rhs
    };

    Ok(RbfModel {
        points: points.to_vec(),
        weights: solution.rows(0, n).into_owned(),
        poly_coeffs: solution.rows(n, 4).into_owned(),
    })
}

pub fn evaluate_rbf(model: &RbfModel, x: f64, y: f64, z: f64) -> f64 {
    let mut value = 0.0;
    for (i, point) in model.points.iter().enumerate() {
        let dx = x - point.x;
        let dy = y - point.y;
        let dz = z - point.z;
        let r = (dx*dx + dy*dy + dz*dz).sqrt();
        value += model.weights[i] * thin_plate_spline(r);
    }
    value + model.poly_coeffs[0] + model.poly_coeffs[1]*x + model.poly_coeffs[2]*y + model.poly_coeffs[3]*z
}
```

### 4. 3D Drillhole Viewer (React + Three.js)

```typescript
// frontend/src/components/DrillholeViewer3D.tsx
import { Canvas } from '@react-three/fiber';
import { OrbitControls, Line } from '@react-three/drei';
import * as THREE from 'three';
import { useDrillholeStore } from '../stores/drillholeStore';

interface DrillholeTrace {
  holeId: string;
  points: [number, number, number][];
  assays: { fromDepth: number; toDepth: number; auPpm: number; color: string }[];
}

function DrillholeLine({ trace }: { trace: DrillholeTrace }) {
  const colors = useMemo(() => {
    const arr = new Float32Array(trace.points.length * 3);
    for (let i = 0; i < trace.points.length; i++) {
      const depth = i * trace.points[trace.points.length - 1][2] / trace.points.length;
      const assay = trace.assays.find(a => depth >= a.fromDepth && depth <= a.toDepth);
      const color = assay ? new THREE.Color(assay.color) : new THREE.Color(0x888888);
      arr[i*3] = color.r; arr[i*3+1] = color.g; arr[i*3+2] = color.b;
    }
    return arr;
  }, [trace]);

  return (
    <Line points={trace.points} color="white" lineWidth={2} vertexColors={colors} />
  );
}

export function DrillholeViewer3D() {
  const { traces } = useDrillholeStore();

  return (
    <div className="w-full h-full">
      <Canvas camera={{ position: [500, 500, 300], fov: 50 }}>
        <ambientLight intensity={0.5} />
        <directionalLight position={[10, 10, 5]} intensity={0.8} />
        <OrbitControls enableDamping />
        <gridHelper args={[1000, 20]} />
        {traces.map(trace => <DrillholeLine key={trace.holeId} trace={trace} />)}
      </Canvas>
    </div>
  );
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–5)

**Day 1: Backend scaffold**
- `cargo init mineassay-api` with Axum, sqlx, tokio dependencies
- `src/main.rs`, `src/config.rs`, `src/state.rs`, `src/error.rs`
- `docker-compose.yml`: PostgreSQL+PostGIS, Redis, MinIO

**Day 2: Database schema**
- `migrations/001_initial.sql`: 10 core tables
- `src/db/models.rs`: SQLx structs with FromRow
- Run migrations, seed test project

**Day 3: Auth system**
- `src/auth/mod.rs`: JWT middleware, bcrypt
- `src/auth/oauth.rs`: Google/GitHub OAuth
- `src/api/handlers/auth.rs`: register, login, OAuth callbacks

**Day 4-5: Project/user CRUD**
- `src/api/handlers/users.rs`, `projects.rs`, `orgs.rs`
- `src/api/router.rs`: route definitions with auth
- Integration tests: auth flow, project CRUD
- Stripe webhook setup

### Phase 2 — Drillhole Import + Desurvey (Days 6–10)

**Day 6: CSV/Excel import**
- `import-core/src/csv_parser.rs`: CSV parsing with column mapping
- `import-core/src/excel_parser.rs`: Excel via calamine
- `import-core/src/validation.rs`: depth overlaps, missing fields

**Day 7: Import API**
- `src/api/handlers/drillholes.rs`: import endpoint
- Support collar, survey, assay, lithology imports
- Bulk insert optimization

**Day 8: Desurvey**
- `desurvey-core/src/lib.rs`: minimum curvature method
- Compute 3D XYZ from survey azimuth/dip
- Tests vs known solutions

**Day 9: Drillhole queries**
- List/get drillholes with PostGIS spatial filters
- Export to CSV
- Compositing endpoint

**Day 10: Stats service**
- `stats-service/`: Python FastAPI
- Histogram, probability plots, summary stats
- Rust API proxy

### Phase 3 — Geostatistics Core (Days 11–18)

**Day 11: Experimental variogram**
- `geostat-core/src/variogram/experimental.rs`: γ(h) = 1/(2N) ∑[z(xᵢ) - z(xⱼ)]²
- Lag binning, directional variograms
- Tests vs hand-calculated

**Day 12: Variogram fitting**
- `geostat-core/src/variogram/models.rs`: spherical, exponential, Gaussian
- `geostat-core/src/variogram/fitting.rs`: weighted least-squares
- Nested structures support

**Day 13: Search ellipsoid**
- `geostat-core/src/search.rs`: rotated ellipsoid search
- Octant classification (8 sectors)
- Min/max samples, octant constraints

**Day 14-15: Kriging solver**
- `geostat-core/src/kriging.rs`: OK implementation
- Build K matrix, solve via nalgebra Cholesky
- Variance computation
- Tests: 4-sample system, unbiasedness

**Day 16: Block model generation**
- `modeling-core/src/block_model.rs`: regular grid cells
- Spatial indexing (R-tree)
- MessagePack serialization
- `src/api/handlers/block_models.rs`

**Day 17: Kriging server execution**
- `src/workers/estimation_worker.rs`: Redis job queue
- Rayon parallel kriging
- WebSocket progress
- S3 result storage

**Day 18: Resource classification**
- `geostat-core/src/classification.rs`: measured/indicated/inferred
- Rules based on kriging variance
- Tonnage/grade statistics

### Phase 4 — WASM + 3D Viz (Days 19–25)

**Day 19: WASM kriging**
- `geostat-wasm/`: wasm-pack build
- GitHub Actions WASM pipeline
- Size optimization

**Day 20: WASM frontend integration**
- `frontend/src/wasm/kriging.ts`: load WASM module
- Client kriging for ≤50K cells
- Progress via requestAnimationFrame

**Day 21-22: Three.js drillholes**
- `DrillholeViewer3D.tsx`: React Three Fiber
- LineSegments with vertex colors
- Collar markers, labels
- Camera auto-fit

**Day 23: Block model rendering**
- `BlockModelViewer.tsx`: instanced meshes
- 100K+ cells at 60 FPS
- Slice planes

**Day 24: Geological surfaces**
- `GeologicalSurfaceViewer.tsx`: load OBJ/STL
- Transparency, double-sided
- Multiple lithologies

**Day 25: 2D sections**
- `SectionView2D.tsx`: Canvas/Konva
- Project drillholes onto plane
- Block model slices

### Phase 5 — RBF Modeling (Days 26–30)

**Day 26: RBF solver**
- `modeling-core/src/rbf.rs`: thin-plate spline
- SVD pseudoinverse
- Tests: 10-point fit

**Day 27: Lithology extraction**
- `lithology_extraction.rs`: extract contacts from drillholes
- Assign +1/-1 values

**Day 28: Marching Cubes**
- `marching_cubes.rs`: isosurface extraction
- 256 cube configurations
- OBJ export

**Day 29: RBF API + worker**
- `src/api/handlers/geological_models.rs`
- `src/workers/modeling_worker.rs`: background RBF
- S3 mesh upload

**Day 30: WASM RBF**
- `modeling-wasm/`: WASM RBF for <10K points
- Client-side preview

### Phase 6 — Frontend Polish + Billing (Days 31–36)

**Day 31-32: Dashboard**
- `ProjectDashboard.tsx`: stats, activity feed
- `DrillholeManager.tsx`: table view, map
- Mapbox GL JS collar locations

**Day 33: Variogram UI**
- `VariogramModeler.tsx`: interactive fitting
- Drag controls for range/sill/nugget
- Live model overlay

**Day 34: Estimation wizard**
- `EstimationWizard.tsx`: multi-step setup
- Variogram selection, search config
- Progress monitor

**Day 35: Stripe billing**
- `src/billing/stripe.rs`: checkout, webhooks
- Subscription management
- Plan updates

**Day 36: Plan enforcement**
- `src/middleware/plan_check.rs`: tier limits
- Exploration: 500 holes, 100K cells
- 402 Payment Required errors

### Phase 7 — Testing + Optimization (Days 37–40)

**Day 37: Kriging validation**
- Benchmarks vs GSLib datasets
- Performance: 10K, 50K, 100K cells
- Parallel speedup verification

**Day 38: Integration tests**
- Full workflow: import → model → krige → classify
- Docker Compose full stack
- Assert final results

**Day 39: Performance optimization**
- Database query optimization, indexes
- S3 multipart upload
- WASM size reduction (wasm-opt -Oz)
- Three.js LOD

**Day 40: Security audit**
- SQL injection prevention (SQLx parameterized)
- Auth validation, rate limiting
- Error handling (no leaks)
- Penetration testing

### Phase 8 — Documentation + Deployment (Days 41–42)

**Day 41: Documentation**
- API docs (OpenAPI/Swagger)
- User guide: CSV formats, variogram tutorial
- Video tutorials
- Glossary

**Day 42: Production deployment**
- AWS: ECS Fargate, RDS PostgreSQL+PostGIS, ElastiCache Redis, S3
- CloudFront CDN, Route53 DNS, ACM SSL
- Prometheus/Grafana monitoring, Sentry
- Automated backups, load testing
- Launch

---

## Critical Files Tree

```
mineassay/
├── Cargo.toml
├── docker-compose.yml
├── migrations/001_initial.sql                 # 10 tables
├── mineassay-api/
│   ├── src/
│   │   ├── main.rs
│   │   ├── config.rs, state.rs, error.rs
│   │   ├── auth/mod.rs, oauth.rs
│   │   ├── db/models.rs                      # SQLx structs
│   │   ├── api/handlers/
│   │   │   ├── drillholes.rs                 # Import, desurvey (★)
│   │   │   ├── block_models.rs
│   │   │   ├── estimations.rs
│   │   │   └── geological_models.rs
│   │   ├── workers/
│   │   │   ├── estimation_worker.rs          # Kriging jobs (★)
│   │   │   └── modeling_worker.rs            # RBF jobs
│   │   └── billing/stripe.rs
├── import-core/src/
│   ├── csv_parser.rs
│   ├── excel_parser.rs
│   └── validation.rs
├── desurvey-core/src/lib.rs                  # Minimum curvature
├── geostat-core/src/
│   ├── kriging.rs                            # OK solver (★ CORE)
│   ├── variogram/experimental.rs, models.rs, fitting.rs
│   ├── search.rs                             # Ellipsoid + octants
│   └── classification.rs
├── modeling-core/src/
│   ├── rbf.rs                                # RBF implicit (★ CORE)
│   ├── marching_cubes.rs
│   └── block_model.rs
├── geostat-wasm/src/lib.rs                   # WASM kriging
├── modeling-wasm/src/lib.rs                  # WASM RBF
├── stats-service/app/
│   ├── main.py (FastAPI)
│   ├── histogram.py, probability_plot.py
│   └── summary_stats.py
└── frontend/src/
    ├── pages/
    │   ├── ProjectDashboard.tsx
    │   ├── DrillholeManager.tsx
    │   ├── VariogramModeler.tsx              # Interactive fitting (★)
    │   └── EstimationWizard.tsx
    ├── components/
    │   ├── DrillholeViewer3D.tsx             # Three.js viz (★ CORE)
    │   ├── BlockModelViewer.tsx
    │   └── SectionView2D.tsx
    ├── stores/drillholeStore.ts, blockModelStore.ts
    └── wasm/kriging.ts, rbf.ts
```

**Critical paths:**
1. Import → Desurvey → 3D: 100 holes in <5s
2. RBF model: 500 contacts → 20K mesh in <10s
3. Kriging 50K cells: WASM <15s, server (8 cores) <5s
4. 3D render 100K cells: 60 FPS instanced

---

## Solver Validation Suite

### Benchmark 1: Kriging Unbiasedness
**Setup:** 4 samples at (0,0,5), (100,0,5), (0,100,5), (100,100,5) with z = 1.5, 2.0, 1.8, 2.2 g/t. Spherical variogram: nugget 0.1, sill 1.0, range 50m. Estimate at (50,50,5).

**Expected:**
- Weights sum: 1.000 ± 0.001
- Estimate: 1.875 ± 0.02 g/t (symmetric average)
- Variance: 0.65 ± 0.05
- At data point: weight=1.0 for that sample, others=0.0

**Validation:** `assert!(abs(sum(weights) - 1.0) < 0.001 && abs(estimate_at_data - actual) < 1e-6)`

### Benchmark 2: Variogram Fitting
**Setup:** 1000 synthetic pairs from spherical (nugget 0.2, sill 1.0, range 75m) + noise ±0.02. Fit spherical model.

**Expected:**
- Nugget: 0.20 ± 0.05
- Sill: 1.00 ± 0.10
- Range: 75 ± 5 m
- R²: >0.95

**Validation:** `assert!(abs(fitted_nugget - 0.2) < 0.05 && r_squared > 0.95)`

### Benchmark 3: RBF Exact Interpolation
**Setup:** 10 random 3D points with values ∈ [-1, +1]. Fit thin-plate spline RBF. Evaluate at data points.

**Expected:**
- Error at each data point: <1e-6
- Smooth evaluation between points

**Validation:** `assert!(max_error_at_data < 1e-6)`

### Benchmark 4: Block Model Indexing
**Setup:** 200×200×100 block (4M cells) at 5m. Query cells within 100m of (500,500,250). Expected ~1680 cells (sphere volume).

**Expected:**
- Query time: <50 ms (R-tree)
- Cell count: 1680 ± 50

**Validation:** `assert!(query_time_ms < 50 && abs(found - 1680) < 50)`

### Benchmark 5: Kriging Parallel Scaling
**Setup:** 50K cells, 500 samples, search 50m, min 8, max 16 samples. Run on 1, 2, 4, 8 cores.

**Expected:**
- 1 core: ~12s
- 2 cores: ~6.5s (1.8x)
- 4 cores: ~3.5s (3.4x)
- 8 cores: ~2.0s (6.0x)

**Validation:** `assert!(speedup_4_cores > 3.0 && speedup_8_cores > 5.0)`

---

## Verification Checklist

**Auth:**
- [ ] User registers, receives JWT
- [ ] Google/GitHub OAuth works
- [ ] JWT expires after 24h
- [ ] 401 on invalid token
- [ ] User accesses own projects only
- [ ] Plan limits enforced

**Drillholes:**
- [ ] Import CSV collars → PostGIS geography
- [ ] Import surveys → desurvey produces 3D trace (1% accuracy)
- [ ] Import assays → intervals linked
- [ ] Validation detects overlaps
- [ ] PostGIS spatial query <100ms for 1K holes
- [ ] Export CSV re-imports cleanly

**Geostatistics:**
- [ ] Experimental variogram: correct pair counts
- [ ] Fit spherical: nugget/sill/range within 10%
- [ ] Kriging weights sum to 1.000 ± 0.001
- [ ] Kriging at data point: exact value (1e-6)
- [ ] Search ellipsoid finds all samples
- [ ] Octant search rejects insufficient coverage
- [ ] WASM kriging 50K cells <15s
- [ ] Server kriging 500K cells (8 cores) <60s
- [ ] Variance decreases with more/closer samples
- [ ] Classification: measured/indicated/inferred assigned

**Modeling:**
- [ ] Extract lithology contacts
- [ ] RBF 100 points <2s
- [ ] RBF at data points: 1e-6 error
- [ ] Marching Cubes: manifold mesh
- [ ] Mesh normals outward
- [ ] 500 points → 20K triangles <10s
- [ ] WASM RBF <10K points in browser

**3D Viz:**
- [ ] 100 drillholes @ 60 FPS
- [ ] Assay intervals colored by grade
- [ ] 100K block cells instanced @ 60 FPS
- [ ] Geological surfaces transparent
- [ ] OrbitControls smooth
- [ ] Camera auto-fits
- [ ] Toggle visibility
- [ ] 2D section projects correctly

**Billing:**
- [ ] Stripe checkout redirects
- [ ] Webhooks update plan
- [ ] Exploration blocks 501st hole (402)
- [ ] Planning allows unlimited
- [ ] Usage tracked
- [ ] Customer portal link works

**Performance:**
- [ ] Import 1K holes <30s
- [ ] Desurvey 100 holes <2s
- [ ] Generate 500K cell model <5s
- [ ] WASM kriging 50K cells <15s
- [ ] Server kriging 500K cells <60s
- [ ] RBF 500 points → mesh <10s
- [ ] 3D render 100 holes + 100K cells @ 60 FPS
- [ ] WebSocket progress <100ms latency

---

## Deployment Architecture

### AWS Infrastructure

```
CloudFront CDN (WASM bundles, static assets) → TTL: 1 day
    ↓
ALB (HTTPS, /api → API, /stats → Python)
    ↓
┌─────────────────┬──────────────────┐
ECS Fargate API   ECS Fargate Stats
2 vCPU, 4 GB      1 vCPU, 2 GB
Auto-scale 2-10   Auto-scale 1-4
    ↓                   ↓
┌─────────┬─────────┬─────────┐
RDS       ElastiCache  S3
Postgres  Redis 7      Block models
+PostGIS  cache.t3.med Geo meshes
db.r5.xl  Single node  Versioning
Multi-AZ
```

**Background Workers (ECS):**
- Redis job queue → Estimation worker (4 vCPU, 8 GB, Rayon 8 cores)
- Redis job queue → Modeling worker (2 vCPU, 4 GB, RBF + Marching Cubes)
- Results to S3

**Monitoring:**
- CloudWatch Logs (30 days)
- Prometheus + Grafana: API latency (p50/p95/p99), kriging cells/sec, DB query time, S3 metrics, Redis queue depth
- Sentry: errors, traces, releases

**Scaling:**
| Metric | Threshold | Action |
|--------|-----------|--------|
| API CPU >70% | 2 min | Add ECS task (max 10) |
| Redis queue >100 | 1 min | Add estimation worker (max 20) |
| RDS CPU >80% | 5 min | Upgrade to r5.2xlarge |
| Kriging wait >5 min | 2 min | Add worker |

**DR:**
- RDS: daily snapshots, 7-day retention, Multi-AZ failover <2 min
- S3: versioning, 30-day lifecycle
- Secrets rotation: 90 days

---

## Post-MVP Roadmap

### v1.1 — Advanced Geostatistics (Weeks 7-10)
- Sequential Gaussian Simulation (SGS) for uncertainty (100 realizations, P10/P50/P90 curves)
- Indicator kriging for categorical variables
- Cokriging (multi-variate, use Cu to improve Au)
- Cross-validation for variogram validation
- Anisotropic search (full 3D rotation)
- Multi-pass kriging

**Impact:** Planning tier gets risk quantification for JORC/NI 43-101 compliant reports with confidence intervals.

### v1.2 — Mine Planning (Weeks 11-16)
- Open-pit optimization (Lerchs-Grossmann): max NPV pit shell
- Pushback design (nested pits, sequencing)
- Pit design tools (bench geometry, ramps, haul roads)
- Underground stope optimization (MSO)
- Cut-off grade optimization (Lane's algorithm)
- Mine scheduling (period-by-period, blending, capacity)

**Impact:** Planners design pits/schedules in MineAssay, eliminating $15K+ Vulcan/Datamine licenses.

### v1.3 — Geotechnical Analysis (Weeks 17-22)
- Slope stability (Bishop, Spencer, GLE on geo model sections)
- Rock mass classification (RMR, Q-system, GSI from drillhole data)
- Kinematic analysis (stereonets: planar/wedge/toppling)
- Empirical pillar design (Lunder-Pakalnis, Mathews)
- Subsidence prediction (caving, longwall)
- Tailings dam stability

**Impact:** Geotechs assess stability using same geo model as resource estimation, eliminating manual data transfer.

### v1.4 — Reporting & Compliance (Weeks 23-26)
- JORC 2012 Table 1 auto-generation
- NI 43-101 Technical Report template
- Resource/reserve statement tables
- Cross-section/plan figures (PDF/PNG)
- Variogram documentation
- Competent Person sign-off workflow
- PDF assembly (TOC, sections, figures)

**Impact:** Geologists generate draft JORC/NI 43-101 reports in hours vs weeks, saving $20K-$50K consultant costs.

### v1.5 — Collaboration & Enterprise (Weeks 27-32)
- Real-time collaboration (operational transformation)
- Version control (models, diffs, merges)
- Comment threads
- Role-based permissions (owner/editor/viewer/reviewer)
- Audit log
- On-premise deployment (Docker/K8s)
- LDAP/SAML SSO
- API webhooks

**Impact:** Enterprise (20+ users) collaborate with governance. On-premise opens strict data policy market.

### v1.6 — Advanced Viz & AI (Weeks 33-38)
- WebGPU upgrade (10x faster for multi-million cells)
- VR support (WebXR immersive exploration)
- Fly-through animations for stakeholders
- AI-assisted variogram fitting (ML suggests params)
- Automated lithology classification from assay geochemistry (random forest/SVM)
- Anomaly detection (QA/QC flagging)
- Geological trend analysis (faults/folds from drillholes via ML)

**Impact:** Less manual variogram fitting/lithology logging. VR increases stakeholder engagement for funding.
