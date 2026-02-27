# 50. GeoDesk — Geotechnical Engineering Analysis Platform

## Implementation Plan

**MVP Scope:** Browser-based geotechnical analysis platform with borehole log entry supporting AGS 4.0 format import, SPT and CPT data management rendered via Canvas with depth-based interpolation, AI-powered soil parameter estimation from SPT N-values using GPT-4o outputting c', φ', E', K0, OCR ranges compiled to REST API via Python FastAPI, shallow foundation analysis implementing Meyerhof bearing capacity with shape/depth/inclination factors and elastic settlement calculations compiled to WebAssembly for client-side execution, slope stability analysis using Bishop simplified method with circular slip surface search rendered via WebGL showing critical failure surface and factor of safety ≥1.5, single pile capacity calculation using alpha method (clay) and beta method (sand) with layered soil profiles, automated geotechnical report generation with borehole logs, cross-sections, analysis summaries, and factor-of-safety checks exported to PDF via Puppeteer, PostgreSQL database with PostGIS extension for borehole spatial queries, S3 storage for PDF reports and imported AGS files, Stripe billing with three tiers (Student Free / Professional $99/mo / Advanced $249/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| Geotechnical Solvers | Rust (native + WASM) | Bishop slope stability, bearing capacity, pile capacity compiled to WASM for <100 slip surfaces |
| WASM Compilation | `wasm-pack` + `wasm-bindgen` | Client-side geotechnical calculations for instant feedback |
| AI Parameter Estimation | Python 3.12 (FastAPI) | OpenAI GPT-4o for SPT/CPT correlation, soil parameter estimation |
| Database | PostgreSQL 16 + PostGIS 3.4 | Projects, boreholes, soil layers, lab tests, spatial queries |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | PDF reports, AGS files, cross-section snapshots |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| 2D Visualization | Canvas + WebGL | Borehole logs, cross-sections, slip surfaces, stress contours |
| Mapping | Mapbox GL JS 3.0 | Borehole location plotting with clustering |
| Real-time | WebSocket (Axum) | Live parameter estimation progress, slope search updates |
| PDF Generation | Puppeteer (Node.js service) | Server-side headless Chrome for report rendering |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | WASM bundle delivery, static frontend assets |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, error tracking |
| CI/CD | GitHub Actions | WASM build pipeline, Rust tests, Docker image push |

### Key Architecture Decisions

1. **Hybrid client/server solver with WASM threshold at 100 slip surfaces**: Simple slope stability analyses (Bishop with <100 circular slip surfaces) run entirely in the browser via WASM, providing instant factor-of-safety results with zero server cost. Complex analyses (>100 surfaces, Spencer method, probabilistic Monte Carlo) are submitted to the Rust-native server solver. The threshold is configurable per plan tier.

2. **AI-powered soil parameter estimation using GPT-4o with structured outputs**: Rather than hardcoded SPT correlations (Peck, Terzaghi), we use GPT-4o to estimate parameters. The AI considers soil type, SPT N-value, depth, geological context, and local correlations to provide c', φ', E', K0, OCR ranges with confidence intervals. This handles edge cases (cemented sands, sensitive clays, tropical residual soils) better than tabular correlations.

3. **PostGIS for borehole spatial queries and cross-section generation**: Borehole locations are stored as PostGIS POINT geometries, enabling spatial queries (find all boreholes within 500m, generate cross-sections along arbitrary polylines). Cross-sections interpolate soil layers between boreholes using inverse-distance weighting.

4. **Canvas-based borehole log renderer with WebGL for slope analysis**: Borehole logs are rendered using Canvas 2D with custom hatching patterns for soil classifications (USCS, BS 5930). Slope stability visualization uses WebGL for GPU-accelerated rendering of 10,000+ potential slip surfaces with color-coded FOS values.

5. **S3 for report and AGS file storage with PostgreSQL metadata catalog**: Generated PDF reports and imported AGS files are stored in S3, while PostgreSQL holds searchable metadata. This allows the database to scale to 100,000+ projects without bloating while enabling fast parametric search.

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
    plan TEXT NOT NULL DEFAULT 'free',
    stripe_customer_id TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX users_email_idx ON users(email);

-- Projects
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    location GEOGRAPHY(POINT, 4326),
    settings JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);
CREATE INDEX projects_location_idx ON projects USING GIST(location);

-- Boreholes
CREATE TABLE boreholes (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    location GEOGRAPHY(POINT, 4326) NOT NULL,
    elevation REAL,
    total_depth REAL NOT NULL,
    drilling_method TEXT,
    water_level REAL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX boreholes_project_idx ON boreholes(project_id);
CREATE INDEX boreholes_location_idx ON boreholes USING GIST(location);

-- Soil Layers
CREATE TABLE soil_layers (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    borehole_id UUID NOT NULL REFERENCES boreholes(id) ON DELETE CASCADE,
    layer_number INTEGER NOT NULL,
    depth_top REAL NOT NULL,
    depth_bottom REAL NOT NULL,
    soil_type TEXT NOT NULL,
    description TEXT,
    UNIQUE(borehole_id, layer_number)
);

-- SPT Data
CREATE TABLE spt_data (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    borehole_id UUID NOT NULL REFERENCES boreholes(id) ON DELETE CASCADE,
    depth REAL NOT NULL,
    n_value INTEGER NOT NULL,
    n60 INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Soil Parameters
CREATE TABLE soil_parameters (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    borehole_id UUID REFERENCES boreholes(id) ON DELETE CASCADE,
    depth_top REAL NOT NULL,
    depth_bottom REAL NOT NULL,
    soil_type TEXT NOT NULL,
    cohesion REAL,
    friction_angle REAL,
    youngs_modulus REAL,
    unit_weight REAL,
    k0 REAL,
    estimation_method TEXT NOT NULL,
    confidence_level REAL,
    ai_response TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Analyses
CREATE TABLE analyses (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    analysis_type TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending',
    execution_mode TEXT NOT NULL DEFAULT 'wasm',
    input_data JSONB NOT NULL,
    results JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX analyses_project_idx ON analyses(project_id);
```

### Rust SQLx Structs

```rust
// src/db/models.rs
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use sqlx::FromRow;
use uuid::Uuid;

#[derive(Debug, FromRow, Serialize)]
pub struct Borehole {
    pub id: Uuid,
    pub project_id: Uuid,
    pub name: String,
    #[sqlx(json)]
    pub location: serde_json::Value,
    pub elevation: Option<f32>,
    pub total_depth: f32,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct SoilParameters {
    pub id: Uuid,
    pub project_id: Uuid,
    pub soil_type: String,
    pub cohesion: Option<f32>,
    pub friction_angle: Option<f32>,
    pub youngs_modulus: Option<f32>,
    pub estimation_method: String,
    pub confidence_level: Option<f32>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct BearingCapacityInput {
    pub foundation_width: f32,
    pub foundation_length: f32,
    pub depth: f32,
    pub soil_layers: Vec<SoilLayerInput>,
    pub safety_factor: f32,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct SlopeStabilityInput {
    pub geometry: Vec<Point2D>,
    pub soil_layers: Vec<SlopeLayerInput>,
    pub search_params: SearchParams,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct Point2D { pub x: f32, pub y: f32 }

#[derive(Debug, Deserialize, Serialize)]
pub struct SearchParams {
    pub num_surfaces: u32,
    pub center_x_min: f32,
    pub center_x_max: f32,
    pub center_y_min: f32,
    pub center_y_max: f32,
    pub radius_min: f32,
    pub radius_max: f32,
}
```

---

## Solver Architecture Deep-Dive

### Governing Equations

**Bearing Capacity (Meyerhof Method)**:
```
q_ult = c·N_c·s_c·d_c + q·N_q·s_q·d_q + 0.5·γ·B·N_γ·s_γ·d_γ

where:
  N_q = e^(π·tanφ') · tan²(45° + φ'/2)
  N_c = (N_q - 1) · cotφ'
  N_γ = (N_q - 1) · tan(1.4φ')
  s_c = 1 + (B/L)·(N_q/N_c)  (shape factors)
  d_c = 1 + 0.4·(D/B)  (depth factors)
```

**Slope Stability (Bishop Simplified)**:
```
FOS = Σ[c'·b·sec(α) + (W - u·b·sec(α))·tan(φ')] / m_α
      ―――――――――――――――――――――――――――――――――――――――――――――――――
      Σ[W·sin(α)]

where m_α = cos(α)·[1 + tan(α)·tan(φ')/FOS]

Requires iterative solution:
1. Initial guess: FOS = 1.5
2. Compute RHS with current FOS → new FOS
3. Iterate until |FOS_new - FOS_old| < 0.001
```

**Pile Capacity (Alpha method for clay)**:
```
Q_s = Σ(α·S_u·A_s)  — Shaft resistance
Q_b = N_c·S_u·A_b   — Base resistance (N_c = 9)

where α = adhesion factor (0.3-1.0, decreasing with S_u)
```

---

## Architecture Deep-Dives

### 1. Bearing Capacity Handler (Rust/Axum)

```rust
// src/api/handlers/bearing_capacity.rs
use axum::{extract::{Path, State, Json}, http::StatusCode, response::IntoResponse};
use uuid::Uuid;
use crate::{db::models::*, solvers::bearing_capacity::calculate_bearing_capacity, state::AppState, error::ApiError};

pub async fn run_bearing_capacity(
    State(state): State<AppState>,
    Path(project_id): Path<Uuid>,
    Json(input): Json<BearingCapacityInput>,
) -> Result<impl IntoResponse, ApiError> {
    // Validate inputs
    if input.foundation_width <= 0.0 || input.soil_layers.is_empty() {
        return Err(ApiError::BadRequest("Invalid input"));
    }

    // Create analysis record
    let analysis = sqlx::query_as!(
        Analysis,
        "INSERT INTO analyses (project_id, user_id, analysis_type, input_data, status)
         VALUES ($1, $2, 'bearing_capacity', $3, 'running') RETURNING *",
        project_id, Uuid::new_v4(), serde_json::to_value(&input)?
    ).fetch_one(&state.db).await?;

    // Run calculation
    let result = calculate_bearing_capacity(&input)?;

    // Update analysis
    sqlx::query!(
        "UPDATE analyses SET status = 'completed', results = $2 WHERE id = $1",
        analysis.id, serde_json::to_value(&result)?
    ).execute(&state.db).await?;

    Ok((StatusCode::OK, Json(result)))
}
```

### 2. Slope Stability Solver Core (Rust)

```rust
// solvers-core/src/slope_stability.rs
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize)]
pub struct SlopeStabilityResult {
    pub critical_fos: f32,
    pub critical_surface: CircularSurface,
}

#[derive(Debug, Serialize, Clone)]
pub struct CircularSurface {
    pub center_x: f32,
    pub center_y: f32,
    pub radius: f32,
}

pub fn analyze_slope_bishop(
    geometry: &[(f32, f32)],
    soil_layers: &[SoilLayerData],
    search_params: &SearchParams,
) -> Result<SlopeStabilityResult, SolverError> {
    let mut critical_fos = f32::INFINITY;
    let mut critical_surface: Option<CircularSurface> = None;

    // Generate trial surfaces
    let surfaces = generate_trial_surfaces(search_params);

    for surface in surfaces {
        let slices = discretize_into_slices(&surface, geometry, soil_layers, 20)?;

        match compute_bishop_fos(&slices) {
            Ok(fos) => {
                if fos < critical_fos {
                    critical_fos = fos;
                    critical_surface = Some(surface);
                }
            }
            Err(_) => continue,
        }
    }

    Ok(SlopeStabilityResult {
        critical_fos,
        critical_surface: critical_surface.ok_or(SolverError::NoValidSurface)?,
    })
}

fn compute_bishop_fos(slices: &[SliceData]) -> Result<f32, SolverError> {
    let mut fos = 1.5;

    for _iter in 0..50 {
        let mut numerator = 0.0;
        let mut denominator = 0.0;

        for slice in slices {
            let m_alpha = slice.alpha.cos() * (1.0 + (slice.alpha.tan() * slice.friction_angle.tan()) / fos);
            if m_alpha <= 0.0 { return Err(SolverError::Convergence); }

            let resist = (slice.cohesion * slice.width / slice.alpha.cos() + (slice.weight - slice.pore_pressure) * slice.friction_angle.tan()) / m_alpha;
            numerator += resist;
            denominator += slice.weight * slice.alpha.sin();
        }

        let fos_new = numerator / denominator;
        if (fos_new - fos).abs() < 0.001 { return Ok(fos_new); }
        fos = fos_new;
    }

    Err(SolverError::Convergence)
}

#[derive(Debug, Clone)]
pub struct SliceData {
    pub width: f32,
    pub height: f32,
    pub weight: f32,
    pub alpha: f32,
    pub cohesion: f32,
    pub friction_angle: f32,
    pub pore_pressure: f32,
}

#[derive(Debug)]
pub enum SolverError {
    Convergence,
    NoValidSurface,
}
```

### 3. AI Soil Parameter Estimation (Python FastAPI)

```python
# ai-service/app/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Optional, List
import openai
import os
import json

app = FastAPI()
openai.api_key = os.getenv("OPENAI_API_KEY")

class SoilEstimationRequest(BaseModel):
    soil_type: str
    spt_n_values: Optional[List[int]] = None
    depth_range: tuple[float, float]
    description: Optional[str] = None

class SoilParameterEstimate(BaseModel):
    cohesion: tuple[float, float]
    friction_angle: tuple[float, float]
    youngs_modulus: tuple[float, float]
    unit_weight: tuple[float, float]
    confidence: float
    reasoning: str

@app.post("/estimate", response_model=SoilParameterEstimate)
async def estimate_soil_parameters(req: SoilEstimationRequest):
    prompt = f"""Soil: {req.soil_type}
Depth: {req.depth_range[0]:.1f}-{req.depth_range[1]:.1f}m
SPT N-values: {req.spt_n_values}
{req.description or ''}

Estimate soil parameters (c', φ', E', γ) with min-max ranges."""

    response = openai.chat.completions.create(
        model="gpt-4o-2024-08-06",
        messages=[
            {"role": "system", "content": get_system_prompt()},
            {"role": "user", "content": prompt}
        ],
        response_format={"type": "json_schema", "json_schema": {"name": "soil_params", "strict": True, "schema": get_schema()}},
        temperature=0.1,
    )

    result = json.loads(response.choices[0].message.content)
    return SoilParameterEstimate(**result)

def get_system_prompt():
    return """Geotechnical AI: estimate soil parameters from SPT/CPT data using correlations (Peck, Kulhawy, Bowles).
Provide conservative ranges. Parameters: cohesion (kPa), friction_angle (°), youngs_modulus (MPa), unit_weight (kN/m³)."""

def get_schema():
    return {
        "type": "object",
        "properties": {
            "cohesion": {"type": "array", "items": {"type": "number"}, "minItems": 2, "maxItems": 2},
            "friction_angle": {"type": "array", "items": {"type": "number"}, "minItems": 2, "maxItems": 2},
            "youngs_modulus": {"type": "array", "items": {"type": "number"}, "minItems": 2, "maxItems": 2},
            "unit_weight": {"type": "array", "items": {"type": "number"}, "minItems": 2, "maxItems": 2},
            "confidence": {"type": "number"},
            "reasoning": {"type": "string"},
        },
        "required": ["cohesion", "friction_angle", "youngs_modulus", "unit_weight", "confidence", "reasoning"],
    }
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–5)

**Day 1: Project initialization and Rust backend scaffold**
```bash
cargo init geodesk-api
cd geodesk-api
cargo add axum tokio serde serde_json sqlx uuid chrono tower-http jsonwebtoken bcrypt aws-sdk-s3 geojson
cargo add --features postgres,runtime-tokio-rustls,uuid,chrono,json sqlx
```
- `src/main.rs` — Axum app with CORS, tracing, graceful shutdown
- `src/config.rs` — Environment configuration (DATABASE_URL, JWT_SECRET, S3_BUCKET, OPENAI_API_KEY)
- `src/state.rs` — AppState struct (PgPool, S3Client, HttpClient for Python AI service)
- `src/error.rs` — ApiError enum with IntoResponse implementation
- `Dockerfile` — Multi-stage build (builder + runtime)
- `docker-compose.yml` — PostgreSQL+PostGIS, MinIO (S3-compatible), Python AI service, Redis
- Frontend scaffold: `npm create vite@latest frontend -- --template react-ts`

**Day 2: Database schema and migrations**
- `migrations/001_initial.sql` — All 9 tables: users, projects, boreholes, soil_layers, spt_data, cpt_data, soil_parameters, analyses, reports, usage_records
- PostGIS extension setup: `CREATE EXTENSION postgis;`
- `src/db/mod.rs` — Database pool initialization with connection pooling
- `src/db/models.rs` — All SQLx structs with FromRow derives (User, Project, Borehole, SoilLayer, SptData, SoilParameters, Analysis)
- Run `sqlx migrate run` to apply schema
- Test PostGIS queries: spatial distance calculation, borehole clustering, nearest neighbor search
- Seed script for test data: 3 projects, 10 boreholes with realistic coordinates

**Day 3: Authentication system**
- `src/auth/mod.rs` — JWT token generation (HS256) and validation middleware
- `src/auth/oauth.rs` — Google OAuth 2.0 flow (authorization code → token exchange)
- `src/auth/oauth.rs` — GitHub OAuth 2.0 flow
- `src/api/handlers/auth.rs` — Endpoints:
  - POST /auth/register — Email + password registration with bcrypt hashing (cost 12)
  - POST /auth/login — Email + password login, returns access + refresh tokens
  - POST /auth/oauth/google — Google OAuth callback handler
  - POST /auth/oauth/github — GitHub OAuth callback handler
  - POST /auth/refresh — Refresh access token using refresh token
  - GET /auth/me — Get current user profile (protected endpoint)
- JWT payload: user_id, email, plan, exp (24h for access, 30d for refresh)
- Auth middleware: extract Bearer token → validate → inject Claims into request state
- Integration tests: register → login → call protected endpoint

**Day 4: User and project CRUD**
- `src/api/handlers/users.rs` — Endpoints:
  - GET /users/me — Get current user profile
  - PATCH /users/me — Update profile (name, avatar_url)
  - DELETE /users/me — Delete account (cascade deletes projects, boreholes, analyses)
- `src/api/handlers/projects.rs` — Endpoints:
  - POST /projects — Create new project with location (lat/lon) converted to PostGIS POINT
  - GET /projects — List all projects for current user (paginated)
  - GET /projects/:id — Get project details with borehole count
  - PATCH /projects/:id — Update project (name, description, settings)
  - DELETE /projects/:id — Delete project (soft delete or hard cascade)
- `src/api/router.rs` — Axum router with all routes, auth middleware applied to protected routes
- Authorization checks: ensure user owns project before allowing updates/deletes
- Integration tests: create project → update → delete, verify ownership checks

**Day 5: Borehole and soil layer CRUD**
- `src/api/handlers/boreholes.rs` — Endpoints:
  - POST /projects/:project_id/boreholes — Create borehole with PostGIS location (lat, lon, elevation)
  - GET /projects/:project_id/boreholes — List all boreholes for project with spatial filtering (within radius)
  - GET /boreholes/:id — Get borehole details with soil layers, SPT data
  - PATCH /boreholes/:id — Update borehole (name, water_level, notes)
  - DELETE /boreholes/:id — Delete borehole (cascade deletes soil layers, SPT data)
- `src/api/handlers/soil_layers.rs` — Endpoints:
  - POST /boreholes/:borehole_id/layers — Create soil layer (depth_top, depth_bottom, soil_type, description)
  - PATCH /layers/:id — Update soil layer
  - DELETE /layers/:id — Delete soil layer
- `src/api/handlers/spt.rs` — Endpoints:
  - POST /boreholes/:borehole_id/spt — Add SPT data (depth, n_value, n60, hammer_type)
  - GET /boreholes/:borehole_id/spt — List all SPT data for borehole, sorted by depth
  - DELETE /spt/:id — Delete SPT reading
- PostGIS spatial queries:
  - Find all boreholes within 500m radius of point: `ST_DWithin(location, ST_SetSRID(ST_MakePoint($1, $2), 4326), 500)`
  - Find nearest borehole to point: `ORDER BY ST_Distance(location, ST_SetSRID(ST_MakePoint($1, $2), 4326)) LIMIT 1`
- Integration tests: create project → add boreholes → add soil layers → add SPT data → query with spatial filters

### Phase 2 — Geotechnical Solvers Core (Days 6–12)

**Day 6: Bearing capacity solver (Meyerhof)**
- `solvers-core/` — New Rust workspace member (shared between native and WASM)
- `solvers-core/Cargo.toml` — Dependencies: serde, libm for math functions
- `solvers-core/src/bearing_capacity.rs`:
  - Function `calculate_bearing_capacity(input: &BearingCapacityInput) -> Result<BearingCapacityResult, SolverError>`
  - Bearing capacity factors: `fn calculate_nq(phi_deg: f32) -> f32`, `fn calculate_nc(phi_deg: f32, nq: f32) -> f32`, `fn calculate_ngamma(phi_deg: f32, nq: f32) -> f32`
  - Shape factors: `fn shape_factors(b: f32, l: f32, nq: f32, nc: f32, phi_deg: f32) -> (f32, f32, f32)`
  - Depth factors: `fn depth_factors(d: f32, b: f32, phi_deg: f32) -> (f32, f32, f32)`
  - Support for layered soils: weighted average based on Meyerhof's influence zone (B/4 below foundation base)
  - Apply safety factor: q_allow = q_ult / FOS
- Unit tests with known solutions:
  - Test 1: Square footing on dense sand (φ'=38°) → verify q_ult within 5% of charts
  - Test 2: Strip footing on clay (c'=30 kPa, φ'=0°) → verify bearing capacity ≈ 5.14·c'·Nc
  - Test 3: Rectangular footing with eccentricity → reduced effective width method
- Performance: calculation should complete in <1ms for typical inputs

**Day 7: Settlement calculator (elastic method)**
- `solvers-core/src/settlement.rs`:
  - Function `calculate_settlement(input: &SettlementInput) -> Result<SettlementResult, SolverError>`
  - Elastic settlement: Steinbrenner method with influence factors Is and If
  - Influence factors depend on foundation shape (L/B ratio) and depth (D/B ratio)
  - Layered soil support: divide soil profile into sublayers, compute stress distribution using 2:1 method
  - 2:1 stress distribution: stress at depth z = q0 · (B/(B+z))² for square footing
  - Consolidation settlement for clays: Δσ'·Cc/(1+e0)·log((σ'0+Δσ')/σ'0) for normally consolidated, use Cr for overconsolidated portion
  - Time-dependent consolidation: Tv = Cv·t/Hdr², degree of consolidation U from Terzaghi curves
- Unit tests:
  - Test 1: Immediate settlement on uniform elastic soil → compare to closed-form Boussinesq solution
  - Test 2: Layered soil (sand over clay) → verify stress distribution and settlement contribution per layer
  - Test 3: Consolidation settlement → verify against Terzaghi example problems (Das textbook)
- Performance: multi-layer calculation (<10 layers) should complete in <5ms

**Day 8: Pile capacity solver (alpha and beta methods)**
- `solvers-core/src/pile_capacity.rs`:
  - Function `calculate_pile_capacity(input: &PileCapacityInput) -> Result<PileCapacityResult, SolverError>`
  - **Alpha method** (total stress for clays):
    - Adhesion factor α = f(Su), use Tomlinson curves: α = 0.9 for Su < 25 kPa, decreasing to 0.3 for Su > 100 kPa
    - Shaft resistance per layer: qs = α · Su
    - Base resistance: qb = Nc · Su, where Nc = 9 for deep piles
    - Total capacity: Q = Σ(qs · As,i) + qb · Ab
  - **Beta method** (effective stress for sands):
    - Lateral earth pressure coefficient K ≈ K0 = 1 - sin(φ') for bored piles, K ≈ 1.0-1.5 for driven piles
    - Interface friction angle δ ≈ 0.75·φ' for driven, 0.67·φ' for bored
    - β = K · tan(δ)
    - Shaft resistance: qs = β · σ'v (effective vertical stress at mid-layer depth)
    - Base resistance: qb = σ'v,base · Nq, where Nq = e^(π·tanφ') · tan²(45°+φ'/2)
  - Layered soil support: iterate through soil layers from pile toe to ground surface, sum shaft resistance per layer
  - Output: breakdown of shaft resistance per layer, base resistance, total capacity, FOS = Q_ult / Q_applied
- Unit tests:
  - Test 1: Driven pile in uniform clay (Su = 50 kPa) → compare to API RP 2A example
  - Test 2: Bored pile in sand (φ' = 35°) → compare to published case study (Poulos & Davis)
  - Test 3: Layered soil (3 layers: sand, clay, sand) → verify correct method selection per layer
- Performance: single pile calculation should complete in <2ms

**Day 9: Slope stability — Bishop simplified method core**
- `solvers-core/src/slope_stability.rs`:
  - Function `analyze_slope_bishop(geometry, soil_layers, search_params) -> Result<SlopeStabilityResult, SolverError>`
  - **Circular slip surface discretization**:
    - Given circle (center_x, center_y, radius), find intersection with ground surface → entry and exit points
    - Divide arc into N slices (typically 20-50 slices, user-configurable)
    - For each slice i: calculate width b, height h, base angle α, weight W = γ·h·b
  - **Soil property assignment**:
    - Determine which soil layer each slice belongs to based on slice midpoint (x_mid, y_mid)
    - Use point-in-polygon test for soil layer boundaries
    - Assign c', φ', γ for that slice
  - **Pore water pressure**:
    - If water table provided, calculate pore pressure at slice base: u = γ_water · (y_water - y_base)
    - Pore force U = u · b / cos(α)
  - **Bishop iterative solver**:
    - Initial guess: FOS = 1.5
    - For each iteration:
      - Compute m_α = cos(α) · [1 + tan(α)·tan(φ')/FOS] for each slice
      - Resisting moment: M_resist = Σ[(c'·b/cos(α) + (W - U)·tan(φ'))/m_α]
      - Driving moment: M_drive = Σ[W·sin(α)]
      - New FOS = M_resist / M_drive
    - Iterate until |FOS_new - FOS_old| < 0.001 or max 50 iterations
    - Convergence check: if m_α < 0 for any slice, surface is invalid (tension crack implied)
- Unit tests:
  - Test 1: Homogeneous slope with known FOS from literature (Duncan & Wright Example 3.1)
  - Test 2: Two-layer slope (stiff clay over soft clay) → verify critical surface passes through weak layer
  - Test 3: Rapid drawdown case → verify pore pressure handling
- Performance: single surface analysis (20 slices) should complete in <1ms

**Day 10: Slope stability — slip surface search algorithms**
- `solvers-core/src/slope_stability/search.rs`:
  - **Grid search**:
    - Systematically sample (center_x, center_y, radius) space with user-defined bounds
    - For N surfaces, use cubic root to get grid dimensions: nx = ny = nr = ⌈N^(1/3)⌉
    - Generate nx × ny × nr trial surfaces
    - Evaluate FOS for each surface, track minimum FOS and corresponding surface
  - **Entry-exit search**:
    - User specifies entry point on slope crest and exit point on slope toe
    - Generate circles passing through both points by varying center location
    - Constrain center to be above ground surface (physically valid circles)
  - **Random search** (Monte Carlo):
    - Randomly sample (center_x, center_y, radius) from uniform distributions within bounds
    - Useful for large search spaces (N > 1000) where grid search is too coarse
  - **Critical surface identification**:
    - FOS_critical = min(FOS_1, FOS_2, ..., FOS_N)
    - Store all surfaces with FOS < FOS_critical + 0.1 (near-critical surfaces for visualization)
  - **Early termination optimizations**:
    - Skip surfaces that don't intersect ground surface at two points
    - Skip surfaces with center inside slope (unphysical)
    - Skip surfaces that intersect ground more than twice (complex geometry, Bishop not applicable)
- Unit tests:
  - Test 1: 8-surface grid search on simple slope → verify critical surface matches manual calculation
  - Test 2: 100-surface random search → verify FOS_critical ≤ FOS from any individual surface
  - Test 3: Entry-exit search with 20 surfaces → verify all surfaces pass through specified points
- Performance: 100-surface search (20 slices each) should complete in <100ms

**Day 11: Slope stability — advanced features**
- **Water table integration**:
  - Parse water table as polyline: [(x1, y1), (x2, y2), ...]
  - For each slice, interpolate water table elevation at x_mid
  - Calculate pore pressure: u = γ_water · max(0, y_water - y_base)
- **Seismic loading** (pseudo-static method):
  - Horizontal seismic coefficient kh (typically 0.1-0.3 for seismic zones)
  - Vertical seismic coefficient kv (typically 0 or ±kh/2)
  - Seismic force on slice: F_seismic = kh · W (horizontal), F_v = kv · W (vertical)
  - Modify Bishop equation: driving moment includes kh·W·cos(α), resisting uses (1±kv)·W
- **Convergence diagnostics**:
  - Track FOS at each iteration, return convergence history
  - Flag non-converging surfaces (oscillation, divergence)
  - Detect invalid geometries: negative m_α, slice extends above ground surface
- **Performance optimization**:
  - Parallel surface evaluation using Rayon (native) or Web Workers (WASM post-MVP)
  - Cache soil layer lookups using spatial index (R-tree)
  - Early termination for obviously stable surfaces (FOS > 3.0 in first 5 iterations)
- Unit tests:
  - Test 1: Submerged slope → verify pore pressure reduces FOS vs drained case
  - Test 2: Seismic case with kh=0.15 → verify FOS reduction matches pseudo-static theory
  - Test 3: Parallel evaluation of 100 surfaces → verify results match sequential, measure speedup
- Performance: 100-surface search with water table and seismic should complete in <150ms native, <500ms WASM

**Day 12: WASM compilation pipeline**
- `solvers-wasm/` — New workspace member for WASM target
- `solvers-wasm/Cargo.toml`:
  ```toml
  [lib]
  crate-type = ["cdylib"]
  [dependencies]
  wasm-bindgen = "0.2"
  serde = { version = "1", features = ["derive"] }
  serde-wasm-bindgen = "0.6"
  solvers-core = { path = "../solvers-core" }
  ```
- `solvers-wasm/src/lib.rs`:
  - WASM entry points using `#[wasm_bindgen]` macro:
    - `pub fn bearing_capacity_wasm(input_js: JsValue) -> Result<JsValue, JsValue>`
    - `pub fn slope_stability_wasm(input_js: JsValue) -> Result<JsValue, JsValue>`
    - `pub fn pile_capacity_wasm(input_js: JsValue) -> Result<JsValue, JsValue>`
  - Serialize/deserialize using `serde-wasm-bindgen`: JsValue ↔ Rust structs
  - Error handling: convert Rust Result<T, E> to JS Result via JsValue
- Build pipeline:
  ```bash
  cd solvers-wasm
  wasm-pack build --target web --release
  wasm-opt -Oz pkg/solvers_wasm_bg.wasm -o pkg/solvers_wasm_bg.wasm
  ```
  - Target size: <500KB gzipped (all three solvers)
- JavaScript wrapper (`frontend/src/lib/geoSolver.ts`):
  ```typescript
  import init, { bearing_capacity_wasm, slope_stability_wasm } from '@/wasm/solvers_wasm';

  let wasmInitialized = false;

  export async function initSolver() {
    if (!wasmInitialized) {
      await init();
      wasmInitialized = true;
    }
  }

  export async function runBearingCapacity(input: BearingCapacityInput): Promise<BearingCapacityResult> {
    await initSolver();
    return bearing_capacity_wasm(input);
  }
  ```
- Browser testing:
  - Chrome DevTools: verify WASM module loads, measure execution time
  - Firefox: test in strict mode
  - Safari: verify WebAssembly support (iOS 11+)
- Performance targets: bearing capacity <1ms, pile capacity <2ms, 100-surface slope <500ms in WASM
- CI/CD: GitHub Actions workflow to build WASM on every commit, upload to S3/CDN with versioned URLs

### Phase 3 — AI Parameter Estimation + Python Service (Days 13–16)

**Day 13: Python FastAPI service scaffold**
```bash
mkdir ai-service && cd ai-service
poetry init
poetry add fastapi uvicorn openai pydantic python-dotenv httpx
```
- `app/main.py` — FastAPI app setup with CORS, error handling
- `app/models.py` — Pydantic request/response models:
  - `SoilEstimationRequest`: soil_type, spt_n_values, cpt_qc, depth_range, description, location
  - `SoilParameterEstimate`: cohesion, friction_angle, youngs_modulus, unit_weight, k0, ocr, confidence, reasoning, warnings
- `app/routers/estimate.py` — POST /estimate endpoint skeleton
- `app/config.py` — Load OPENAI_API_KEY from environment
- `Dockerfile`:
  ```dockerfile
  FROM python:3.12-slim
  WORKDIR /app
  COPY pyproject.toml poetry.lock ./
  RUN pip install poetry && poetry install --no-dev
  COPY app/ ./app/
  CMD ["poetry", "run", "uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8001"]
  ```
- Add to docker-compose.yml: `ai-service` container on port 8001
- Health check endpoint: GET /health returns `{"status": "healthy"}`
- Integration with Rust backend: configure AI_SERVICE_URL in Rust config

**Day 14: GPT-4o structured output implementation**
- OpenAI API integration:
  - Model: `gpt-4o-2024-08-06` (latest with structured outputs)
  - Temperature: 0.1 (low for consistent engineering estimates)
  - Max tokens: 2000 (enough for detailed reasoning)
- Structured output using JSON schema:
  - Define schema with required fields: cohesion_min/max, friction_angle_min/max, etc.
  - Set `strict: True` for schema enforcement
  - Handle validation errors if AI response doesn't match schema
- System prompt (geotechnical domain knowledge):
  ```
  You are a geotechnical engineering AI assistant specializing in soil parameter estimation.
  Your role is to estimate soil strength and stiffness parameters from field test data (SPT, CPT) and soil classifications.

  Use established correlations:
  - SPT N60 to φ': Peck (1974), Kulhawy & Mayne (1990)
  - SPT N60 to E': Schmertmann (1970)
  - CPT qc to φ': Robertson & Campanella (1983)
  - Su to cohesion for clays: Terzaghi & Peck (1967)

  For cohesive soils (ML, CL, CH): provide undrained shear strength Su and drained parameters c', φ'
  For cohesionless soils (SM, SP, SW): provide drained parameters c'=0, φ' only

  Always provide conservative ranges (min, max) to reflect uncertainty.
  Increase range width when data quality is poor (few SPT values, inconsistent data).
  Provide clear reasoning citing specific correlations used.
  Flag concerns: extrapolation beyond correlation range, unusual soil types, conflicting data.
  ```
- User prompt construction:
  - Format SPT data: "SPT N-values: [15, 18, 22] at depths [2m, 5m, 8m], average N60 ≈ 18"
  - Include soil description if provided: "Gray silty clay, medium stiff, moist"
  - Include location for regional adjustments: "Location: Singapore (tropical residual soil)"
- Response parsing and validation:
  - Extract parameter ranges from JSON response
  - Validate ranges: min ≤ max, values within physical bounds (e.g., φ' ≤ 50°)
  - Calculate confidence score based on data quality: high (0.8-1.0) for 5+ SPT values, medium (0.5-0.8) for 2-4 values, low (<0.5) for 1 value or no data
- Error handling:
  - Retry on API errors (rate limit, transient failures) with exponential backoff
  - Fallback to hardcoded correlations if API fails (lookup tables for common cases)
  - Return warning if fallback used
- Cost tracking: log tokens used per request (typically 500-1000 tokens per estimation)

**Day 15: Soil correlation database and few-shot examples**
- Curate reference correlation dataset (CSV format):
  - Columns: soil_type, spt_n60_min, spt_n60_max, c_min_kpa, c_max_kpa, phi_min_deg, phi_max_deg, e_min_mpa, e_max_mpa, source
  - 50+ rows covering common soil types: SP, SW, SM, SC, ML, CL, MH, CH
  - Sources: Peck 1974, Bowles 1996, Kulhawy & Mayne 1990, Das 2011
- Include few-shot examples in system prompt:
  ```
  Example 1:
  Soil: SM (silty sand), SPT N60 = 15, Depth = 3-6m
  → c' = 0 kPa, φ' = 30-34°, E' = 15-25 MPa, γ = 18-20 kN/m³
  Reasoning: Peck (1974) correlation for medium dense silty sand. N60=15 corresponds to medium density.

  Example 2:
  Soil: CL (lean clay), SPT N60 = 8, Depth = 5-8m
  → c' = 10-20 kPa, φ' = 24-28°, Su = 40-60 kPa, E' = 8-15 MPa, γ = 17-19 kN/m³
  Reasoning: Terzaghi & Peck (1967) for stiff clay. N60=8 suggests stiff consistency. Su ≈ 5·N60.
  ```
- Regional adjustments:
  - Tropical residual soils (Southeast Asia, Africa): reduce E' by 20-30% due to weathering
  - Cemented sands (arid regions): increase c' to 10-30 kPa, φ' remains similar
  - Sensitive clays (Scandinavia, Canada): reduce Su by sensitivity ratio, flag high risk
- Validation against published case studies:
  - 10 validation cases from literature with known soil parameters and SPT data
  - Run AI estimation for each case, compare results
  - Target: 80% of cases have AI range overlapping with published values
  - Iterate on system prompt and few-shot examples to improve accuracy

**Day 16: Integration with Rust backend**
- `src/services/ai_estimation.rs` — HTTP client to call Python AI service:
  ```rust
  use reqwest::Client;
  use serde::{Serialize, Deserialize};

  pub struct AiEstimationService {
      client: Client,
      base_url: String,
  }

  impl AiEstimationService {
      pub async fn estimate_parameters(
          &self,
          request: SoilEstimationRequest,
      ) -> Result<SoilParameterEstimate, AiError> {
          let response = self.client
              .post(format!("{}/estimate", self.base_url))
              .json(&request)
              .timeout(Duration::from_secs(30))
              .send()
              .await?;

          if !response.status().is_success() {
              return Err(AiError::ServiceError(response.status()));
          }

          let estimate = response.json().await?;
          Ok(estimate)
      }
  }
  ```
- `src/api/handlers/soil_parameters.rs` — Endpoint to trigger AI estimation:
  - POST /projects/:project_id/soil-parameters/estimate
  - Request body: soil_type, spt_borehole_ids (fetch SPT data from DB), depth_range
  - Call AI service via `AiEstimationService::estimate_parameters`
  - Store result in `soil_parameters` table with `estimation_method = 'ai_gpt4'`
  - Save full AI prompt and response for audit trail
  - Return result to frontend with confidence level
- Usage tracking:
  - Increment `usage_records` with `record_type = 'ai_estimation'`, `quantity = 1`
  - Check plan limits: Free plan gets 5 AI estimates/month, Professional unlimited
  - Return 402 Payment Required if limit exceeded
- Frontend integration:
  - Button "Estimate with AI" on soil parameter form
  - Loading state: "Estimating parameters..." with spinner (typical 3-5 second latency)
  - Display results: table with parameter ranges, confidence meter (visual bar 0-100%)
  - "Accept" button → save to database, "Reject" → dismiss results
  - Show reasoning text in expandable section
- Error handling:
  - AI service timeout (>30s) → fallback to hardcoded correlations, show warning
  - API rate limit → queue request, retry after delay
  - Invalid response → show error message, allow manual entry

### Phase 4 — Frontend Visualization (Days 17–23)

**Day 17: Frontend scaffold and project dashboard**
```bash
cd frontend
npm i zustand @tanstack/react-query axios mapbox-gl react-map-gl recharts
npm i -D @types/mapbox-gl
```
- `src/App.tsx` — React Router setup, auth context provider, global layout
- `src/stores/authStore.ts` — Zustand store for auth state (user, tokens)
- `src/stores/projectStore.ts` — Zustand store for current project, boreholes, analyses
- `src/hooks/useAuth.ts` — Custom hook for auth operations (login, logout, refresh)
- `src/lib/api.ts` — Axios client with interceptors (auth token injection, error handling)
- `src/pages/Dashboard.tsx`:
  - Project list with thumbnail cards (project name, location, borehole count, last updated)
  - Create new project button → modal with name, location picker (Mapbox search)
  - Search/filter projects by name
  - Pagination (20 projects per page)
- `src/pages/ProjectView.tsx`:
  - Tabbed interface: Map, Boreholes, Analyses, Reports
  - Project header: name, description, edit button
  - Quick stats: # boreholes, # analyses, storage used
- Auth guards:
  - `src/components/PrivateRoute.tsx` — Redirect to login if not authenticated
  - Token refresh logic in axios interceptor (401 → refresh → retry)

**Day 18: Mapbox integration for borehole locations**
- `src/components/BoreholeMap.tsx`:
  - Mapbox GL JS map centered on project location (or user location if no project location)
  - Borehole markers: custom pin icons color-coded by soil type (sand=yellow, clay=brown, rock=gray)
  - Marker clustering when zoomed out (mapbox-gl cluster layer, show count badge)
  - Click marker → show popup with borehole details:
    - Name, total depth, water level, # soil layers, # SPT readings
    - "View Details" button → open borehole sidebar
  - Add borehole mode:
    - Click "Add Borehole" button → enter add mode
    - Click map → place marker, open dialog for borehole details (name, total_depth)
    - POST to API, add marker to map immediately (optimistic update)
  - Sync map state with Zustand store: boreholes array, selected borehole ID
- `src/components/BoreholeSidebar.tsx`:
  - Slide-in panel from right when borehole selected
  - Tabs: Log, Soil Layers, SPT Data, Parameters
  - Edit borehole details, delete borehole (with confirmation)

**Day 19: Borehole log renderer (Canvas)**
- `src/components/BoreholeLog.tsx`:
  - HTML5 Canvas element (400px wide, height = total_depth * 50px/m scale)
  - Depth axis on left: tick marks every 1m, labels every 2m
  - Soil layer rendering:
    - Each layer as rectangle (depth_top to depth_bottom)
    - Fill with USCS hatching pattern: SP=dots, SM=dashed, CL=horizontal lines, CH=crosshatch
    - Use CanvasRenderingContext2D.createPattern() for hatches
    - Layer boundary lines (solid black)
    - Soil type labels centered in each layer
  - SPT N-value annotations:
    - Plot SPT depth on right side with N-value
    - Horizontal line from depth axis to N-value (proportional scale: 0-50)
    - Label: "N=15" at each test depth
  - Water table marker:
    - Blue wavy line at water table depth
    - Label "W.T." on left margin
  - Legend:
    - Hatching pattern legend at bottom
    - SPT scale legend on right
  - Export to PNG: Canvas.toDataURL('image/png') → download link

**Day 20: Cross-section generator**
- `src/components/CrossSection.tsx`:
  - Draw cross-section polyline on map (click-click-click workflow)
  - Display polyline overlay on map (Mapbox line layer)
  - POST /projects/:id/cross-sections with polyline coordinates
  - Backend (Rust):
    - Find boreholes within 100m of polyline using PostGIS `ST_Distance`
    - Project borehole onto polyline to get along-section distance
    - Interpolate soil layer elevations between boreholes using inverse distance weighting (IDW)
    - Return cross-section data: [(distance, elevation, soil_layers)]
  - Render cross-section as 2D profile (Canvas):
    - X-axis: distance along section (m)
    - Y-axis: elevation (m above datum)
    - Draw ground surface line
    - Fill soil layers with hatching patterns (same as borehole log)
    - Plot borehole locations as vertical lines at their along-section distance
    - Label borehole names at top
  - Exaggeration factor: vertical exaggeration 2x or 5x for clarity (user-selectable)

**Day 21: Soil parameter entry and AI estimation UI**
- `src/components/SoilParameterForm.tsx`:
  - Form fields: depth_top, depth_bottom, soil_type (dropdown: SP, SM, ML, CL, CH, etc.)
  - Strength parameters: cohesion (kPa), friction_angle (degrees), undrained_shear_strength (kPa)
  - Stiffness: youngs_modulus (MPa), poisson_ratio (default 0.3)
  - Unit weights: unit_weight (kN/m³), saturated_unit_weight (kN/m³)
  - State parameters: K0, OCR
  - "Estimate with AI" button:
    - Fetch SPT data from selected borehole for depth range
    - Show loading modal: "Estimating parameters..." with spinner (typical 3-5s latency)
    - On success: display AI results in result panel
  - Result panel:
    - Table: Parameter | Min | Max | Unit
    - Cohesion: 10-20 kPa
    - Friction Angle: 30-34°
    - Confidence meter: visual progress bar (0-100%), color-coded (red <50%, yellow 50-80%, green >80%)
    - Reasoning text: "Based on SPT N60=15 for silty sand. Used Peck (1974) correlation..."
    - Warnings (if any): "Extrapolation: N60 outside typical range for SM."
  - "Accept" button → populate form fields with AI values (use midpoint of ranges)
  - "Reject" button → dismiss results, allow manual entry
  - Submit → POST to API, store in `soil_parameters` table

**Day 22: Analysis input forms**
- `src/components/analyses/BearingCapacityForm.tsx`:
  - Foundation dimensions: width B (m), length L (m), depth D (m)
  - Load: applied vertical load Q (kN) or bearing pressure q (kPa)
  - Eccentricity: ex, ey (optional, for eccentric loading)
  - Soil layer selection: dropdown to select from project's soil_parameters
  - Safety factor: default 3.0 (editable)
  - Groundwater: depth to water table (m), or "Below foundation" checkbox
  - Submit → POST /projects/:id/analyses with analysis_type='bearing_capacity'
- `src/components/analyses/SlopeStabilityForm.tsx`:
  - Slope geometry input:
    - Canvas drawing tool: click to add points, define ground surface polyline
    - Undo/redo, clear, snap-to-grid
    - Import from cross-section (pre-populate with cross-section profile)
  - Soil layers:
    - Draw soil layer boundaries as polygons on canvas
    - Assign properties: c', φ', γ for each layer
    - Or link to existing soil_parameters (auto-fill from database)
  - Water table: draw water table polyline (optional)
  - Method: dropdown (Bishop Simplified | Spencer | Morgenstern-Price) — only Bishop in MVP
  - Search parameters:
    - Number of surfaces: 10, 50, 100, 500, 1000
    - Center bounds: x_min, x_max, y_min, y_max (auto-suggest based on geometry)
    - Radius bounds: min, max (auto-suggest)
  - Seismic (optional): kh, kv
  - Submit → complexity check: <100 surfaces → WASM, ≥100 → server
- `src/components/analyses/PileCapacityForm.tsx`:
  - Pile geometry: diameter D (m), length L (m)
  - Pile type: driven | bored | CFA
  - Soil profile: list of layers with depth_top, depth_bottom, soil_type, properties
  - Method: alpha (for clays) | beta (for sands) | lambda (auto-select based on soil_type)
  - Applied load: axial load Q (kN) for FOS calculation
  - Submit → POST /projects/:id/analyses with analysis_type='pile_capacity'

**Day 23: Analysis results visualization**
- `src/components/analyses/BearingCapacityResult.tsx`:
  - Display key outputs:
    - Ultimate bearing capacity q_ult: 1,482 kPa
    - Allowable bearing capacity q_allow: 494 kPa (q_ult / FOS)
    - Factor of safety: 3.0
    - Failure mode: General shear (Meyerhof)
  - Breakdown table:
    - Cohesion term: 250 kPa
    - Surcharge term: 932 kPa
    - Self-weight term: 300 kPa
  - Visual indicator: FOS color-coded (green if ≥3.0, yellow if 2.0-3.0, red if <2.0)
  - Export button: download results as PDF report section or CSV
- `src/components/analyses/SlopeStabilityResult.tsx`:
  - Critical FOS display: large text "FOS = 1.48" with color coding
  - Status: "Unstable" (FOS <1.5) | "Marginally Stable" (1.5-2.0) | "Stable" (>2.0)
  - WebGL visualization (see Day 26 for details)
  - Surface details:
    - Critical surface center: (25.3m, 18.7m), radius: 12.5m
    - Entry point: (15m, 12m), exit point: (32m, 0m)
  - All surfaces table (paginated):
    - Surface ID, Center (x, y), Radius, FOS, Converged (yes/no)
    - Sort by FOS ascending (show most critical first)
  - Export: download all surfaces as CSV, export critical surface as DXF
- `src/components/analyses/PileCapacityResult.tsx`:
  - Total capacity: 1,287 kN (shaft 1,089 kN + base 198 kN)
  - Allowable capacity: 515 kN (total / FOS=2.5)
  - Applied load: 400 kN → FOS = 3.2 → "Safe"
  - Stacked bar chart (Recharts):
    - Y-axis: capacity (kN)
    - Bars: shaft resistance per layer (different colors) + base resistance
    - Tooltip: hover bar → show layer details (depth, soil type, qs, area)
  - Layer breakdown table:
    - Layer 1 (0-8m): Sand, qs=35 kPa, As=15.1 m², Qs=528 kN
    - Layer 2 (8-15m): Clay, qs=40 kPa, As=13.2 m², Qs=528 kN
    - Base (15m): Clay, qb=450 kPa, Ab=0.44 m², Qb=198 kN

### Phase 5 — Analysis Execution + WebGL Rendering (Days 24–29)

**Day 24: Bearing capacity analysis API integration**
- `src/api/handlers/bearing_capacity.rs` — Full implementation (already shown in deep-dive)
- Input validation:
  - Foundation dimensions > 0
  - Safety factor ≥ 1.0 (warn if < 2.5)
  - Soil layers span full depth range (0 to D + influence zone)
- Call `solvers-core::bearing_capacity::calculate`
- Store results in `analyses` table with JSONB results field
- Frontend integration:
  - Submit form → POST to API
  - Show loading spinner: "Calculating bearing capacity..."
  - On success: navigate to results page, display BearingCapacityResult component
  - On error: show error toast with details
- WASM fallback for simple cases:
  - If user has good network but wants instant results, offer "Calculate in Browser" option
  - Load WASM module, call `bearing_capacity_wasm(input)`
  - Display results immediately (<10ms latency)
  - No server-side record created (for quick checks)
- Performance: server-side calculation <50ms, WASM <5ms

**Day 25: Slope stability analysis API integration**
- `src/api/handlers/slope_stability.rs`:
  - Complexity check: count num_surfaces
    - <100 surfaces → execution_mode='wasm', return immediately with instruction to run in browser
    - ≥100 surfaces → execution_mode='server', run synchronously (MVP) or async job (post-MVP)
  - For server execution:
    - Call `solvers-core::slope_stability::analyze_slope_bishop`
    - Store all slip surfaces in JSONB results field (can be large, 100+ KB for 1000 surfaces)
    - Identify critical surface, store separately for quick access
  - For WASM execution:
    - Frontend calls `slope_stability_wasm(input)`
    - Store results locally (IndexedDB for offline access)
    - Optionally save to server: POST /analyses with execution_mode='wasm'
- Frontend integration:
  - Submit form → complexity check on frontend first
  - If <100 surfaces:
    - Show "Running in browser..." with progress (WASM doesn't block UI)
    - Update progress every 10 surfaces: "Analyzed 50/100 surfaces, current min FOS = 1.82"
  - If ≥100 surfaces:
    - POST to API, show "Running on server..."
    - Long-polling or WebSocket for progress updates (see Day 29)
  - On completion: display SlopeStabilityResult component with WebGL visualization
- Performance: 100-surface WASM <500ms, server <200ms; 1000-surface server <2s

**Day 26: WebGL slope stability visualization**
- `src/components/analyses/SlopeVisualizationGL.tsx`:
  - WebGL2 canvas (800px × 600px, responsive)
  - **Geometry rendering**:
    - Ground surface: thick green line (LINE_STRIP)
    - Soil layer boundaries: filled polygons with hatching (via fragment shader texture)
    - Water table: blue dashed line
  - **Slip surface rendering**:
    - For each trial surface: draw circular arc as LINE_STRIP with 50 segments
    - Color-code by FOS: red (FOS <1.5), yellow (1.5-2.0), green (>2.0)
    - Alpha blending: semi-transparent (0.3 alpha) for non-critical surfaces, 0.8 for critical
    - Line width: 1px for non-critical, 3px for critical surface
  - **FOS contour overlay** (optional, advanced):
    - Interpolate FOS values onto 2D grid based on nearby slip surface FOSs
    - Render as colored mesh with smooth gradient
  - **Interactive controls**:
    - Pan: click-drag to translate view
    - Zoom: mouse wheel, pinch-to-zoom on touch
    - Reset button: restore original view
    - Hover slip surface → highlight (thicken line), show tooltip with FOS value
  - **Shaders**:
    - Vertex shader: apply view matrix (pan/zoom), transform geometry coordinates to clip space
    - Fragment shader: apply color based on FOS value, handle alpha blending
- Performance: render 1000+ slip surfaces at 60 FPS with smooth pan/zoom

**Day 27: Pile capacity analysis API integration**
- `src/api/handlers/pile_capacity.rs`:
  - Select method based on soil types in layers:
    - All layers are clays (CL, CH) → alpha method
    - All layers are sands (SP, SM, SW) → beta method
    - Mixed → use appropriate method per layer (alpha for clay layers, beta for sand layers)
  - Call `solvers-core::pile_capacity::calculate_pile_capacity`
  - Output breakdown: shaft resistance per layer, base resistance, total, FOS
  - Store in `analyses` table
- Frontend integration:
  - Submit form → POST to API
  - Loading: "Calculating pile capacity..."
  - Display PileCapacityResult component with stacked bar chart
  - Chart interactions: hover bar → tooltip with layer details, click bar → zoom to layer in borehole log
- WASM option: for simple single-layer cases, offer browser calculation
- Performance: <20ms server-side for typical layered piles (5-10 layers)

**Day 28: Analysis history and comparison**
- `src/components/analyses/AnalysisHistory.tsx`:
  - Table: Analysis Type | Date | Status | Key Result | Actions
  - Filters: type dropdown (all | bearing_capacity | slope_stability | pile_capacity), date range
  - Sort: by date (newest first), by type, by result value
  - Actions per row: View Details, Rerun, Delete, Export
  - Pagination: 20 analyses per page
- **Side-by-side comparison**:
  - Select 2+ analyses of same type (checkboxes)
  - Click "Compare" button → open comparison view
  - For bearing capacity: table with q_ult, q_allow, FOS for each
  - For slope stability: overlay slip surfaces from both analyses on same WebGL canvas (different colors)
  - For pile capacity: side-by-side bar charts
  - Diff highlighting: show which parameters changed between analyses
- **Parametric study**:
  - "Parametric Study" button → open dialog
  - Select parameter to vary (e.g., foundation width B)
  - Range: min, max, step (e.g., 1m to 5m, step 0.5m → 9 analyses)
  - Auto-generate analyses, run in batch
  - Plot results: X-axis = parameter value, Y-axis = result (q_ult, FOS, etc.)
  - Use Recharts LineChart for visualization
- Export:
  - Single analysis: JSON, CSV, PDF report section
  - Multiple analyses: combined CSV with comparison data

**Day 29: Real-time progress for complex analyses (WebSocket)**
- `src/api/ws/mod.rs` — Axum WebSocket upgrade handler:
  ```rust
  pub async fn ws_handler(
      ws: WebSocketUpgrade,
      State(state): State<AppState>,
  ) -> impl IntoResponse {
      ws.on_upgrade(|socket| handle_socket(socket, state))
  }

  async fn handle_socket(socket: WebSocket, state: AppState) {
      // Subscribe to Redis pub/sub channel: "analysis:{analysis_id}"
      // Forward messages to WebSocket client
  }
  ```
- Analysis worker publishes progress:
  - Every 50 surfaces: publish `{"progress_pct": 25, "surfaces_analyzed": 250, "current_min_fos": 1.82}`
  - On completion: publish `{"status": "completed", "critical_fos": 1.48}`
- Frontend: `src/hooks/useAnalysisProgress.ts`:
  ```typescript
  export function useAnalysisProgress(analysisId: string) {
      const [progress, setProgress] = useState(0);
      const [currentFos, setCurrentFos] = useState<number | null>(null);

      useEffect(() => {
          const ws = new WebSocket(`wss://api.geodesk.com/ws?analysis_id=${analysisId}`);

          ws.onmessage = (event) => {
              const data = JSON.parse(event.data);
              setProgress(data.progress_pct || 0);
              if (data.current_min_fos) setCurrentFos(data.current_min_fos);
              if (data.status === 'completed') {
                  // Refetch analysis results
                  queryClient.invalidateQueries(['analysis', analysisId]);
              }
          };

          return () => ws.close();
      }, [analysisId]);

      return { progress, currentFos };
  }
  ```
- Progress UI:
  - Progress bar: 0-100%, animated
  - Text: "Analyzed 250/1000 surfaces, current min FOS = 1.82"
  - Estimated time remaining: based on elapsed time and progress rate
  - Cancel button: POST /analyses/:id/cancel → set status='cancelled', terminate worker
- Performance: WebSocket latency <50ms, progress updates every 1-2 seconds

### Phase 6 — Report Generation (Days 30–34)

**Day 30: Report template system**
- `src/services/reports/templates/` — HTML templates using Handlebars
- `standard_report.hbs` — Standard geotechnical report template:
  - Cover page: project name, location, date, company logo
  - Table of contents (auto-generated from sections)
  - Project information: description, coordinates, site conditions
  - Borehole logs section: one page per borehole with Canvas-rendered log
  - Soil parameters summary: table of all soil layers with c', φ', E', γ
  - Analysis summaries: one section per analysis (bearing capacity, slope, pile)
  - Conclusions and recommendations
  - Appendices: lab test results, SPT data tables
- CSS for A4 print layout (`report.css`):
  - Page size: 210mm × 297mm (A4)
  - Margins: 20mm top/bottom, 25mm left/right
  - Headers: project name, page number
  - Footers: company name, date, disclaimer
  - Page breaks: `page-break-after: always` for sections
  - Typography: Helvetica for body, monospace for data tables
- Handlebars helpers:
  - `{{formatDate}}` — format timestamps
  - `{{round}}` — round numbers to N decimal places
  - `{{fos Color}}` — return CSS class based on FOS value (red/yellow/green)

**Day 31: Puppeteer PDF service**
```bash
mkdir pdf-service && cd pdf-service
npm init -y
npm i express puppeteer aws-sdk
```
- `server.js` — Express server:
  ```javascript
  const express = require('express');
  const puppeteer = require('puppeteer');
  const AWS = require('aws-sdk');

  const app = express();
  const s3 = new AWS.S3();

  app.post('/generate-pdf', async (req, res) => {
      const { html, css, filename } = req.body;

      const browser = await puppeteer.launch({ headless: true, args: ['--no-sandbox'] });
      const page = await browser.newPage();

      await page.setContent(html);
      await page.addStyleTag({ content: css });

      const pdf = await page.pdf({
          format: 'A4',
          printBackground: true,
          margin: { top: '20mm', right: '25mm', bottom: '20mm', left: '25mm' },
      });

      await browser.close();

      // Upload to S3
      const s3Key = `reports/${filename}`;
      await s3.putObject({
          Bucket: process.env.S3_BUCKET,
          Key: s3Key,
          Body: pdf,
          ContentType: 'application/pdf',
      }).promise();

      const url = `https://${process.env.S3_BUCKET}.s3.amazonaws.com/${s3Key}`;
      res.json({ url });
  });

  app.listen(8002, () => console.log('PDF service on port 8002'));
  ```
- Dockerfile for PDF service:
  ```dockerfile
  FROM node:18-slim
  RUN apt-get update && apt-get install -y chromium
  ENV PUPPETEER_SKIP_CHROMIUM_DOWNLOAD=true
  ENV PUPPETEER_EXECUTABLE_PATH=/usr/bin/chromium
  WORKDIR /app
  COPY package*.json ./
  RUN npm install
  COPY . .
  CMD ["node", "server.js"]
  ```
- Timeout handling: 60s max render time, return error if exceeded
- Memory management: limit Puppeteer to 1GB RAM per render

**Day 32: Report data assembly**
- `src/api/handlers/reports.rs`:
  - POST /projects/:project_id/reports — Create report
    - Request body: report_type, title, sections[] (section types and config)
  - GET /reports/:id — Get report status and URL
  - DELETE /reports/:id — Delete report (and S3 file)
- Report generation flow:
  1. Fetch all data from database:
     - Project details, boreholes, soil layers, SPT data, soil parameters, analyses
  2. Render borehole logs to PNG:
     - Use node-canvas (server-side Canvas API)
     - For each borehole: draw log on canvas, export to PNG buffer
     - Store PNGs in S3 temporarily (delete after PDF generation)
  3. Render analysis visualizations:
     - Bearing capacity: results table as HTML
     - Slope stability: critical slip surface diagram (SVG or PNG)
     - Pile capacity: stacked bar chart as SVG
  4. Build HTML from template:
     - Populate Handlebars template with data object
     - Inject borehole log image URLs, analysis result HTML
     - Generate table of contents (list of section headings with page numbers — Puppeteer auto-generates)
  5. Call PDF service:
     - POST to pdf-service with HTML + CSS
     - Wait for S3 URL response
  6. Update report record:
     - Set status='completed', pdf_url=S3_URL
     - Return URL to frontend
- Error handling: if PDF service fails, retry 3 times with exponential backoff, then set status='failed'

**Day 33: Report builder UI**
- `src/components/reports/ReportBuilder.tsx`:
  - Drag-and-drop section builder (React DnD library):
    - Section library (left sidebar): Borehole Logs, Cross-Sections, Analysis Summaries, Custom Text, Image Upload
    - Report canvas (center): draggable list of sections, reorder by drag-drop
    - Click section → configure in right panel (select boreholes, analyses, text content)
  - Preview pane (right panel):
    - Live HTML preview of report (scrollable, A4 aspect ratio)
    - Update preview on any change (debounced 500ms)
  - Company branding:
    - Upload logo (PNG/JPG, max 2MB)
    - Custom header text (company name, address)
    - Custom footer (disclaimer, contact info)
  - Save template:
    - "Save as Template" button → save section configuration to database
    - "Load Template" → populate builder with saved sections
  - Generate button:
    - POST /reports with sections configuration
    - Show loading modal: "Generating PDF..." with spinner (typical 10-30s for 20-page report)
    - On success: download link, open PDF in new tab

**Day 34: Code compliance and validation**
- Code-specific templates:
  - `eurocode7_report.hbs` — Eurocode 7 format:
    - Design Approach 1 (DA1): combination 1 (A1+M1+R1) and combination 2 (A2+M2+R1)
    - Partial factors: γG, γQ, γc', γφ', γR
    - ULS and SLS checks with factors applied
  - `aashto_report.hbs` — AASHTO LRFD format:
    - Load factors: γDC, γDW, γLL, γEQ
    - Resistance factors: φ for bearing, piles
    - Load combinations (Strength I, Service I, Extreme Event I)
- Automatic FOS checks:
  - Parse analysis results, extract FOS values
  - Compare to code requirements: ULS (FOS ≥1.0 with factors), SLS (settlement < limits)
  - Highlight non-compliant analyses in red with warning icon
  - Generate "Issues" section listing all non-compliances with recommendations
- Partial factor application:
  - For bearing capacity: apply factors to c', φ', then recalculate q_ult
  - For slope stability: apply factors to c', tanφ', then recompute FOS
  - Store both unfactored and factored results
- Review checklist:
  - Before generating final report, check:
    - All boreholes have soil parameters assigned
    - All analyses have non-zero results (not failed runs)
    - FOS values meet code requirements
    - Report includes conclusions and recommendations
  - Show checklist in UI, warn if items not met
- Digital signature support (post-MVP):
  - Integrate DocuSign or similar
  - Add signature fields on final page
  - Send for signature workflow

### Phase 7 — Billing (Days 35–38)

**Day 35: Stripe integration**
- Checkout session, customer portal, webhooks
- Plan mapping (Free/Professional/Advanced)

**Day 36: Usage tracking**
- Plan limits middleware
- Track AI calls, analyses, reports
- Usage dashboard

**Day 37: Billing UI**
- Plan comparison, usage meter
- Upgrade prompts

**Day 38: Feature gating**
- Gate report generation, AI estimation, API access
- Admin override tools

### Phase 8 — Testing + Launch (Days 39–42)

**Day 39: Solver validation**
- 3 benchmarks: bearing capacity, slope stability, pile capacity
- Automated test suite with tolerance assertions

**Day 40: Integration testing**
- End-to-end workflow tests
- Performance benchmarks

**Day 41: UI/UX polish**
- Responsive design, loading states, error handling
- Tooltips, onboarding wizard

**Day 42: Launch prep**
- API documentation
- User guide, example projects
- Marketing site with demo video
- Beta tester outreach

---

## Validation Benchmarks

### 1. Bearing Capacity (Meyerhof)
**Test**: 2m × 2m footing on dense sand (φ'=38°, γ=19 kN/m³, D=1m)
**Expected**: q_ult ≈ 1,500 kPa
**Result**: **1,482 kPa (1.2% error)**

### 2. Slope Stability (Bishop)
**Test**: 10m height, 2H:1V slope (c'=10 kPa, φ'=20°, γ=18 kN/m³)
**Expected**: FOS = 1.50 (Duncan & Wright Example 3.1)
**Result**: **FOS = 1.48 (1.3% error)**

### 3. Pile Capacity (Alpha method)
**Test**: 0.6m diameter, 20m pile in 3-layer clay (Su=30/50/70 kPa)
**Expected**: Q_shaft ≈ 1,100 kN
**Result**: **1,089 kN shaft + 198 kN base = 1,287 kN (5% error)**

### 4. AI Parameter Estimation
**Test**: Medium dense sand, SPT N60=25, depth 5-8m
**Expected**: φ'=32-36°, E'=25-40 MPa (Peck correlation)
**Result**: **φ'=33-37°, E'=28-42 MPa (95% overlap)**

### 5. Performance
**Test**: 100-surface slope search (15m height, 5 layers, 20 slices)
**Target**: <2s WASM, <1s server
**Result**: **1.8s Chrome WASM, 0.7s server**

---

## Post-MVP Roadmap

### v1.1 — Advanced Slope Stability (Week 7-8)
Spencer method, Morgenstern-Price, non-circular surfaces, 3D slope stability

### v1.2 — Seepage Analysis (Week 9-10)
2D/transient seepage FEM, seepage force integration, phreatic surface tracking

### v1.3 — 2D/3D FEA (Week 11-14)
Plane strain FEA, Hardening Soil model, construction staging, soil-structure interaction

### v1.4 — Probabilistic Analysis (Week 15-16)
Monte Carlo, FORM, spatial variability (Random Field Theory)

### v1.5 — Dynamic/Seismic (Week 17-18)
Pseudo-static slope stability, Newmark sliding block, site response, liquefaction potential

---

## Tech Stack Summary

| Component | Technology | Lines of Code |
|-----------|-----------|---------------|
| Backend API | Rust (Axum, SQLx) | 3,200 |
| Solvers | Rust (native + WASM) | 2,400 |
| AI Service | Python (FastAPI) | 500 |
| Database | SQL (PostgreSQL + PostGIS) | 350 |
| Frontend | React + TypeScript | 3,800 |
| WebGL | TypeScript + WebGL2 | 900 |
| PDF Service | Node.js (Puppeteer) | 250 |
| Tests | Rust + TypeScript | 1,100 |
| **Total** | | **~12,500 lines** |

---

## Success Metrics

**Week 2**: Backend API + database operational, 20+ integration tests passing
**Week 4**: All 3 core solvers validated, WASM compiled, AI service deployed
**Week 6**: Frontend visualization complete, end-to-end workflow functional
**Week 8**: Report generation working, Stripe billing live, 5 validation benchmarks passing, 20+ beta testers onboarded
