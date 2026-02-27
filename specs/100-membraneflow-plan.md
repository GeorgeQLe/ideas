# 100. MembraneFlow — Membrane Separation Design Platform

## Implementation Plan

**MVP Scope:** Browser-based membrane system designer with drag-and-drop flow diagram editor for RO/NF membrane arrays (single-stage, two-stage, concentrate recirculation) rendered via SVG, solution-diffusion transport solver implementing element-by-element calculations with concentration polarization, osmotic pressure (van't Hoff + Pitzer for brines), and multi-ion rejection (Spiegler-Kedem model for Na+, Ca2+, Mg2+, Cl-, SO42-, HCO3-, SiO2, B) compiled to WebAssembly for client-side execution of systems ≤100 elements and server-side Rust-native execution for larger systems, scaling prediction with saturation indices (LSI, S&DSI, CaSO4, BaSO4, SiO2) at each membrane element position with antiscalant dosing recommendations, membrane database with 100+ RO/NF elements from DuPont FilmTec, Toray, Hydranautics, and LG Chem with published A/B coefficients and salt rejection data, pressure vessel configuration tool with element-by-element pressure drop (Schock-Miquel correlation), energy recovery device modeling (isentropic PX pressure exchanger with 96% efficiency), specific energy consumption calculator (kWh/m3), interactive performance plots (pressure profile, concentration profile, flux profile along array) rendered via D3.js with zoom/pan, feed water characterization wizard (TDS, pH, temperature, individual ion concentrations, SDI), PDF design report generator with element-by-element performance tables and scaling projections, Stripe billing with three tiers (Free / Pro $99/mo / Advanced $249/user/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| RO/NF Solver | Rust (native + WASM) | Solution-diffusion transport, Spiegler-Kedem multi-ion, CP modulus, scaling indices |
| WASM Compilation | `wasm-pack` + `wasm-bindgen` | Client-side solver for systems ≤100 elements |
| Optimization Engine | Python 3.12 (FastAPI) | SciPy SLSQP for cost optimization, NSGA-II for Pareto fronts |
| Database | PostgreSQL 16 | Projects, users, simulations, membrane catalog, water chemistry |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | Simulation results (element-by-element data), PDF reports, water analysis uploads |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| Flow Diagram Editor | Custom SVG renderer | Membrane array layout, piping, pumps, ERD — snap-to-grid |
| Performance Plots | D3.js v7 | Pressure/concentration/flux profiles, Pareto fronts, cost breakdown |
| Real-time | WebSocket (Axum) | Live optimization progress, large system simulation streaming |
| Job Queue | Redis 7 + Tokio tasks | Server-side solver job management, optimization queue |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | WASM bundle delivery, static frontend assets |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, error tracking |
| CI/CD | GitHub Actions | WASM build pipeline, Rust tests, Docker image push |

### Key Architecture Decisions

1. **Hybrid client/server solver with WASM threshold at 100 elements**: Membrane systems with ≤100 elements (covers 85%+ of brackish water RO, small seawater systems, and educational designs) run entirely in the browser via WASM, providing instant feedback with zero server cost. Systems exceeding 100 elements (large desalination plants, multi-stage designs, optimization sweeps) are submitted to the Rust-native server solver which handles thousand-element arrays and multi-objective optimization. The threshold is configurable per plan tier.

2. **Custom element-by-element solver in Rust rather than wrapping vendor tools**: Building a custom solution-diffusion transport solver in Rust gives us vendor-neutral membrane modeling (any manufacturer's A/B coefficients work), full control over concentration polarization models, and WASM compilation for instant browser-based design. Vendor tools (ROSA, TorayDS) are Windows-only executables with no API, making integration impossible. Our Rust solver uses published transport equations (film theory, Sherwood correlation for spacer-filled channels, Pitzer activity coefficients for brines) validated against vendor tool outputs.

3. **SVG flow diagram editor with drag-and-drop array configuration**: SVG gives crisp rendering of membrane arrays, piping, and equipment at any zoom level. Components (membrane elements, pressure vessels, pumps, ERD) are draggable with snap-to-grid alignment. Auto-routing for piping connections between stages. Canvas/WebGL alternatives were rejected because membrane system diagrams require text labels, flow arrows, and standard P&ID symbols that SVG handles natively.

4. **D3.js for interactive performance plots**: D3.js renders element-by-element profiles (pressure, concentration, flux, scaling saturation index) along the membrane array with interactive tooltips showing detailed values. Plots update in real-time as user adjusts feed pressure, recovery, or array configuration. D3 supports smooth pan/zoom and multi-series plots (e.g., 8 ion concentrations on same chart).

5. **S3 for simulation results and PDF reports**: Element-by-element simulation data (100-element system = ~100KB JSON) is stored in S3 with presigned URLs for client download. PDF reports (performance tables, P&ID, scaling projections) are generated server-side and cached in S3. PostgreSQL holds simulation metadata (status, energy consumption, recovery) for fast dashboard queries.

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";

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

CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    system_type TEXT NOT NULL DEFAULT 'ro',
    flow_diagram JSONB NOT NULL DEFAULT '{}',
    feed_water JSONB NOT NULL DEFAULT '{}',
    design_params JSONB DEFAULT '{}',
    is_public BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE simulations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    simulation_type TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending',
    execution_mode TEXT NOT NULL DEFAULT 'wasm',
    element_count INTEGER NOT NULL DEFAULT 0,
    results_url TEXT,
    results_summary JSONB,
    error_message TEXT,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    wall_time_ms INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE membranes (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    manufacturer TEXT NOT NULL,
    model TEXT NOT NULL,
    membrane_type TEXT NOT NULL,
    application TEXT,
    form_factor TEXT NOT NULL,
    active_area REAL NOT NULL,
    max_feed_pressure REAL,
    max_feed_temperature REAL,
    ph_range_continuous TEXT,
    transport_coefficients JSONB NOT NULL,
    parameters JSONB DEFAULT '{}',
    datasheet_url TEXT,
    is_verified BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE water_analyses (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    source_type TEXT,
    composition JSONB NOT NULL,
    is_public BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
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
    #[serde(skip)]
    pub password_hash: Option<String>,
    pub name: String,
    pub avatar_url: Option<String>,
    pub auth_provider: String,
    pub plan: String,
    pub stripe_customer_id: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Project {
    pub id: Uuid,
    pub owner_id: Uuid,
    pub org_id: Option<Uuid>,
    pub name: String,
    pub system_type: String,
    pub flow_diagram: serde_json::Value,
    pub feed_water: serde_json::Value,
    pub design_params: serde_json::Value,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Membrane {
    pub id: Uuid,
    pub manufacturer: String,
    pub model: String,
    pub membrane_type: String,
    pub active_area: f64,
    pub max_feed_pressure: Option<f64>,
    pub transport_coefficients: serde_json::Value,
    pub parameters: serde_json::Value,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct WaterComposition {
    pub tds: f64,
    pub ph: f64,
    pub temperature: f64,
    pub na: f64,
    pub ca: f64,
    pub mg: f64,
    pub cl: f64,
    pub so4: f64,
    pub hco3: f64,
    pub sio2: f64,
    pub b: Option<f64>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct TransportCoefficients {
    pub a_coeff: f64,
    pub b_coeff: f64,
    pub salt_rejection: f64,
    pub spacer_thickness: f64,
}
```

---

## Solver Architecture Deep-Dive

### Governing Equations and Discretization

MembraneFlow's core solver implements **solution-diffusion transport** for reverse osmosis and nanofiltration membranes:

```
Water flux:    J_w = A × (ΔP - Δπ)
Solute flux:   J_s = B × (C_f - C_p)

where:
  A = water permeability coefficient (L/m²/h/bar)
  B = salt permeability coefficient (L/m²/h)
  ΔP = trans-membrane pressure (bar)
  Δπ = osmotic pressure difference (bar)
```

**Concentration polarization** at the membrane surface:

```
C_wall = C_p + (C_b - C_p) × exp(J_w / k)

Mass transfer coefficient:
  Sh = 0.065 × Re^0.875 × Sc^0.25
  k = Sh × D / d_h
```

**Multi-ion rejection** using Spiegler-Kedem model:

```
For ion i:  R_i = σ_i × (1 - F) / (1 - σ_i × F)
where F = exp(-J_w / ω_i)
```

**Osmotic pressure**: van't Hoff for dilute (TDS < 5,000 mg/L), Pitzer for brines:

```
van't Hoff:  π = (TDS / MW_avg) × R × T × i
Pitzer:      π = φ × ν × C × R × T
```

**Scaling prediction**:

```
Langelier Saturation Index:  LSI = pH - pH_s
CaSO₄:  SI = log₁₀([Ca²⁺] × [SO₄²⁻] / K_sp)
```

**Pressure drop** (Schock-Miquel):

```
ΔP_element = α × Q_feed² + β × Q_feed
```

### Element-by-Element Solution Algorithm

```
For each element i = 1 to N:
   1. Iterate to convergence:
      - Calculate CP modulus: β = exp(J_w / k)
      - Calculate wall osmotic pressure: π_wall
      - Calculate flux: J_w = A × (P - π_wall)
   2. Calculate permeate flow: Q_p = J_w × area
   3. Calculate ion rejection and composition
   4. Mass balance for concentrate
   5. Calculate scaling indices
   6. Update conditions for next element
```

### Client/Server Split

```
System designed → Element count extracted
    │
    ├── ≤100 elements → WASM solver (browser, <500ms)
    │
    └── >100 elements → Server solver (Rust native)
        ├── Job queued via Redis
        ├── Worker executes, streams progress
        └── Results stored in S3
```

---

## Architecture Deep-Dives

### 1. Simulation API Handler (Rust/Axum)

```rust
// src/api/handlers/simulation.rs

use axum::{extract::{Path, State, Json}, http::StatusCode, response::IntoResponse};
use uuid::Uuid;
use crate::{db::models::*, solver::system::analyze_flow_diagram, state::AppState, auth::Claims, error::ApiError};

pub async fn create_simulation(
    State(state): State<AppState>,
    claims: Claims,
    Path(project_id): Path<Uuid>,
    Json(req): Json<CreateSimulationRequest>,
) -> Result<impl IntoResponse, ApiError> {
    let project = sqlx::query_as!(Project,
        "SELECT * FROM projects WHERE id = $1 AND owner_id = $2",
        project_id, claims.user_id
    ).fetch_optional(&state.db).await?.ok_or(ApiError::NotFound("Project not found"))?;

    let system = analyze_flow_diagram(&project.flow_diagram)?;
    let element_count = system.total_element_count();

    let user = sqlx::query_as!(User, "SELECT * FROM users WHERE id = $1", claims.user_id)
        .fetch_one(&state.db).await?;

    if user.plan == "free" && element_count > 50 {
        return Err(ApiError::PlanLimit("Free plan supports up to 50 elements. Upgrade to Pro."));
    }

    let execution_mode = if element_count <= 100 { "wasm" } else { "server" };

    let sim = sqlx::query_as!(Simulation,
        r#"INSERT INTO simulations (project_id, user_id, simulation_type, status, execution_mode, element_count)
        VALUES ($1, $2, $3, $4, $5, $6) RETURNING *"#,
        project_id, claims.user_id,
        serde_json::to_string(&req.simulation_type)?,
        if execution_mode == "wasm" { "completed" } else { "pending" },
        execution_mode, element_count as i32,
    ).fetch_one(&state.db).await?;

    if execution_mode == "server" {
        state.redis.publish("simulation:jobs", serde_json::to_string(&sim.id)?).await?;
    }

    Ok((StatusCode::CREATED, Json(sim)))
}
```

### 2. RO/NF Solver Core (Rust)

```rust
// solver-core/src/ro_solver.rs

use crate::types::*;
use crate::transport::*;
use crate::scaling::*;

pub struct ElementResult {
    pub element_id: usize,
    pub feed_pressure: f64,
    pub feed_flow: f64,
    pub permeate_flow: f64,
    pub permeate_tds: f64,
    pub water_flux: f64,
    pub cp_modulus: f64,
    pub lsi: f64,
    pub si_caso4: f64,
}

pub fn solve_ro_system(
    elements: &[MembraneElement],
    feed_water: &WaterComposition,
    feed_pressure: f64,
    feed_flow: f64,
    temperature: f64,
    options: &SolverOptions,
) -> Result<SystemResult, SolverError> {
    let mut results = Vec::with_capacity(elements.len());
    let mut p_feed = feed_pressure;
    let mut q_feed = feed_flow;
    let mut c_feed = feed_water.clone();

    for (idx, element) in elements.iter().enumerate() {
        let mut j_w = 20.0;
        for _iter in 0..options.max_iterations {
            let k = calculate_mass_transfer_coeff(q_feed, element.active_area, element.spacer_thickness, temperature);
            let beta = calculate_cp_modulus(j_w, k);
            let c_wall = calculate_wall_concentration(&c_feed, beta);
            let pi_wall = calculate_osmotic_pressure(&c_wall, temperature);
            let j_w_new = element.a_coeff * (p_feed - pi_wall);

            if (j_w_new - j_w).abs() < options.flux_tolerance {
                j_w = j_w_new;
                break;
            }
            j_w = j_w_new;
        }

        let q_permeate = j_w * element.active_area / 1000.0;
        let permeate_tds = calculate_permeate_tds(&c_feed, j_w, element.b_coeff);
        let concentrate_tds = (q_feed * c_feed.tds - q_permeate * permeate_tds) / (q_feed - q_permeate);

        let lsi = calculate_lsi(&c_feed, temperature);
        let si_caso4 = calculate_si_caso4(&c_feed, temperature);

        results.push(ElementResult {
            element_id: idx,
            feed_pressure: p_feed,
            feed_flow: q_feed,
            permeate_flow: q_permeate,
            permeate_tds,
            water_flux: j_w,
            cp_modulus: beta,
            lsi,
            si_caso4,
        });

        p_feed -= element.pressure_drop_coeff * q_feed.powi(2);
        q_feed -= q_permeate;
        c_feed.tds = concentrate_tds;
    }

    let total_permeate = results.iter().map(|r| r.permeate_flow).sum::<f64>();
    let overall_recovery = total_permeate / feed_flow;
    let specific_energy = (feed_pressure * feed_flow / 36.0) / total_permeate;

    Ok(SystemResult { results, overall_recovery, specific_energy })
}
```

### 3. Flow Diagram Editor (React + SVG)

```typescript
// frontend/src/components/FlowDiagramEditor.tsx

import { useRef, useCallback, useState } from 'react';
import { useFlowDiagramStore } from '../stores/flowDiagramStore';

export function FlowDiagramEditor() {
  const svgRef = useRef<SVGSVGElement>(null);
  const { components, connections, addComponent, addConnection } = useFlowDiagramStore();
  const [viewBox, setViewBox] = useState({ x: 0, y: 0, width: 1200, height: 800 });
  const [gridSnap] = useState(20);

  const handleCanvasDrop = useCallback((e: React.MouseEvent) => {
    const svg = svgRef.current;
    if (!svg) return;

    const rect = svg.getBoundingClientRect();
    const x = Math.round(((e.clientX - rect.left) * viewBox.width / rect.width + viewBox.x) / gridSnap) * gridSnap;
    const y = Math.round(((e.clientY - rect.top) * viewBox.height / rect.height + viewBox.y) / gridSnap) * gridSnap;

    addComponent({ type: 'membrane_array', position: { x, y }, config: { elements: 6, vessels: 10 } });
  }, [viewBox, gridSnap, addComponent]);

  return (
    <div className="flex h-full">
      <ComponentPalette />
      <div className="flex-1">
        <svg ref={svgRef} className="w-full h-full" viewBox={`${viewBox.x} ${viewBox.y} ${viewBox.width} ${viewBox.height}`} onClick={handleCanvasDrop}>
          <defs>
            <pattern id="grid" width={gridSnap} height={gridSnap} patternUnits="userSpaceOnUse">
              <path d={`M ${gridSnap} 0 L 0 0 0 ${gridSnap}`} fill="none" stroke="#e5e7eb" strokeWidth="0.5" />
            </pattern>
          </defs>
          <rect x={viewBox.x} y={viewBox.y} width={viewBox.width} height={viewBox.height} fill="url(#grid)" />
          {connections.map(conn => <Pipe key={conn.id} {...conn} />)}
          {components.map(comp => (
            <g key={comp.id} transform={`translate(${comp.position.x}, ${comp.position.y})`}>
              {comp.type === 'membrane_array' && <MembraneArray config={comp.config} />}
              {comp.type === 'pump' && <Pump config={comp.config} />}
            </g>
          ))}
        </svg>
      </div>
    </div>
  );
}
```

### 4. Server-Side Simulation Worker (Rust + Redis + S3)

```rust
// src/workers/simulation_worker.rs

use std::sync::Arc;
use aws_sdk_s3::Client as S3Client;
use redis::AsyncCommands;
use sqlx::PgPool;
use uuid::Uuid;
use crate::{
    solver::ro_solver::solve_ro_system,
    solver::system::build_system_from_diagram,
    db::models::{Simulation, WaterComposition, DesignParams},
    websocket::ProgressBroadcaster,
};

pub struct SimulationWorker {
    db: PgPool,
    redis: redis::Client,
    s3: S3Client,
    broadcaster: Arc<ProgressBroadcaster>,
}

impl SimulationWorker {
    pub async fn run(&self) -> anyhow::Result<()> {
        let mut conn = self.redis.get_async_connection().await?;
        tracing::info!("Simulation worker started, listening for jobs...");

        loop {
            let job_id: Option<String> = conn.brpop("simulation:jobs", 30.0).await?;
            let Some(job_id) = job_id else { continue };
            let job_id: Uuid = job_id.parse()?;

            if let Err(e) = self.process_job(job_id).await {
                tracing::error!("Job {job_id} failed: {e}");
                self.mark_failed(job_id, &e.to_string()).await?;
            }
        }
    }

    async fn process_job(&self, job_id: Uuid) -> anyhow::Result<()> {
        let sim = sqlx::query_as!(
            Simulation,
            "UPDATE simulations SET status = 'running', started_at = NOW() WHERE id = $1 RETURNING *",
            job_id
        ).fetch_one(&self.db).await?;

        let project = sqlx::query_as!(
            crate::db::models::Project,
            "SELECT * FROM projects WHERE id = $1",
            sim.project_id
        ).fetch_one(&self.db).await?;

        let system = build_system_from_diagram(&project.flow_diagram)?;
        let feed_water: WaterComposition = serde_json::from_value(project.feed_water)?;
        let design_params: DesignParams = serde_json::from_value(project.design_params)?;

        let result = solve_ro_system(
            &system.elements,
            &feed_water,
            design_params.feed_pressure,
            design_params.feed_flow,
            feed_water.temperature,
            &Default::default(),
        )?;

        // Stream progress updates
        let progress_tx = self.broadcaster.get_channel(sim.id);
        for (idx, elem_result) in result.elements.iter().enumerate() {
            if idx % 10 == 0 {
                let pct = ((idx + 1) as f32 / result.elements.len() as f32 * 100.0);
                let _ = progress_tx.send(serde_json::json!({
                    "progress_pct": pct,
                    "elements_completed": idx + 1,
                }));
            }
        }

        // Upload results to S3
        let result_json = serde_json::to_vec(&result)?;
        let compressed = compress_gzip(&result_json)?;
        let s3_key = format!("results/{}/{}.json.gz", sim.project_id, sim.id);

        self.s3.put_object()
            .bucket("membraneflow-results")
            .key(&s3_key)
            .body(compressed.into())
            .content_type("application/json")
            .content_encoding("gzip")
            .send()
            .await?;

        let results_url = format!("s3://membraneflow-results/{}", s3_key);
        let summary = serde_json::json!({
            "recovery": result.overall_recovery,
            "permeate_tds": result.overall_permeate_tds,
            "specific_energy": result.specific_energy,
            "max_lsi": result.max_lsi,
        });

        sqlx::query!(
            "UPDATE simulations SET status = 'completed', results_url = $2, results_summary = $3,
             completed_at = NOW() WHERE id = $1",
            sim.id, results_url, summary
        ).execute(&self.db).await?;

        self.broadcaster.send(sim.id, serde_json::json!({
            "status": "completed",
            "results_url": results_url,
            "summary": summary,
        }))?;

        tracing::info!("Simulation {job_id} completed successfully");
        Ok(())
    }

    async fn mark_failed(&self, job_id: Uuid, error: &str) -> anyhow::Result<()> {
        sqlx::query!(
            "UPDATE simulations SET status = 'failed', error_message = $2, completed_at = NOW() WHERE id = $1",
            job_id, error
        ).execute(&self.db).await?;
        Ok(())
    }
}

fn compress_gzip(data: &[u8]) -> anyhow::Result<Vec<u8>> {
    use flate2::write::GzEncoder;
    use flate2::Compression;
    use std::io::Write;

    let mut encoder = GzEncoder::new(Vec::new(), Compression::default());
    encoder.write_all(data)?;
    Ok(encoder.finish()?)
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–4)

**Day 1: Project initialization and Rust backend scaffold**
```bash
cargo init membraneflow-api
cd membraneflow-api
cargo add axum tokio serde serde_json sqlx uuid chrono tower-http jsonwebtoken bcrypt aws-sdk-s3 redis
```
- `src/main.rs` — Axum app setup with CORS middleware, tracing subscriber, graceful shutdown handler
- `src/config.rs` — Environment-based configuration: DATABASE_URL, JWT_SECRET, S3_BUCKET, REDIS_URL, STRIPE_SECRET_KEY
- `src/state.rs` — AppState struct containing PgPool, Redis client, S3Client, HTTP client
- `src/error.rs` — ApiError enum with IntoResponse implementation for REST API error handling
- `Dockerfile` — Multi-stage build: Rust builder stage with cargo-chef for dependency caching, runtime stage with minimal Debian base
- `docker-compose.yml` — Local development stack: PostgreSQL 16, Redis 7, MinIO (S3-compatible), Adminer for database management

**Day 2: Database schema and migrations**
- `migrations/001_initial.sql` — All 9 tables: users (auth + billing), organizations (team collaboration), org_members (RBAC), projects (system designs), simulations (job tracking), simulation_jobs (worker queue), membranes (catalog), water_analyses (saved feed waters), design_reports (PDF tracking), usage_records (billing enforcement)
- `src/db/mod.rs` — Database pool initialization with connection pooling (max_connections: 50), migration runner
- `src/db/models.rs` — SQLx FromRow structs for all tables with proper type mappings (UUID, JSONB, TIMESTAMPTZ)
- Run `sqlx migrate run` to apply schema to PostgreSQL
- Seed script `scripts/seed_membranes.sql` — Insert 50 initial membranes: 20 DuPont FilmTec (BW30, SW30, LE), 15 Toray (TM, TML, SU), 10 Hydranautics (CPA, SWC, ESPA), 5 LG Chem (RE, BW, SW) with accurate A/B coefficients from datasheets

**Day 3: Authentication system**
- `src/auth/mod.rs` — JWT token generation (HS256 algorithm), validation middleware using Tower, Claims struct with user_id and plan
- `src/auth/oauth.rs` — OAuth 2.0 flow handlers for Google (using Google Sign-In) and GitHub (OAuth Apps) with state parameter validation
- `src/api/handlers/auth.rs` — POST /auth/register (email + password), POST /auth/login (returns access + refresh tokens), GET /auth/oauth/google, GET /auth/oauth/github, POST /auth/refresh, GET /auth/me
- Password hashing with bcrypt (cost factor 12, ~250ms verification time for security)
- JWT access token: 24-hour expiry, refresh token: 30-day expiry stored in httpOnly cookies
- Auth middleware extracts Claims from Authorization header (Bearer token), returns 401 Unauthorized on failure

**Day 4: User and project CRUD**
- `src/api/handlers/users.rs` — GET /users/me (profile), PATCH /users/me (update name/avatar), DELETE /users/me (soft delete with cascade)
- `src/api/handlers/projects.rs` — POST /projects (create), GET /projects (list with pagination), GET /projects/:id (get with owner check), PATCH /projects/:id (update flow_diagram/feed_water), DELETE /projects/:id, POST /projects/:id/fork (deep copy)
- `src/api/handlers/orgs.rs` — POST /orgs (create org), POST /orgs/:id/invite (send invite email), GET /orgs/:id/members, DELETE /orgs/:id/members/:user_id
- `src/api/router.rs` — Axum Router with all routes, auth middleware on protected endpoints, rate limiting (100 req/min per IP)
- Integration tests: auth flow (register → login → access protected resource), project CRUD with authorization checks (owner vs. non-owner), org member role enforcement

### Phase 2 — Solver Core (Days 5–12)

**Day 5: Transport model framework**
- `solver-core/` — New Rust workspace member (separate crate, compiles to both native and WASM)
- `solver-core/src/transport.rs` — Solution-diffusion water flux `J_w = A × (ΔP - Δπ)`, salt flux `J_s = B × (C_f - C_p)` with temperature correction factors
- `solver-core/src/types.rs` — Core types: MembraneElement (A, B, area, spacer_thickness), WaterComposition (TDS, pH, temp, 8 ion concentrations), IonConcentrations (Na, Ca, Mg, Cl, SO4, HCO3, SiO2, B), SolverOptions (max_iterations, flux_tolerance)
- `solver-core/src/osmotic.rs` — van't Hoff equation for dilute solutions (TDS < 5,000 mg/L): `π = (TDS/MW_avg) × R × T × i`, Pitzer model for brines (TDS > 5,000 mg/L) with activity coefficients
- Unit tests: single-element RO with known A/B coefficients, verify flux and permeate TDS against analytical calculations

**Day 6: Concentration polarization and mass transfer**
- `solver-core/src/cp_modulus.rs` — Film theory CP modulus `β = exp(J_w / k)`, wall concentration `C_wall = C_p + (C_b - C_p) × β`
- `solver-core/src/mass_transfer.rs` — Sherwood correlation for spacer-filled channels: `Sh = 0.065 × Re^0.875 × Sc^0.25`, mass transfer coefficient `k = Sh × D / d_h` with Reynolds and Schmidt number calculations
- Temperature-dependent viscosity (Andrade equation), diffusivity (Stokes-Einstein), density corrections
- Hydraulic diameter calculation from spacer thickness (typically 28-34 mil for RO)
- Tests: verify CP modulus at standard conditions (flux 20 LMH, feed flow 16 m3/h), compare k values to published correlations

**Day 7: Multi-ion rejection (Spiegler-Kedem model)**
- `solver-core/src/rejection.rs` — Spiegler-Kedem rejection for each ion: `R_i = σ_i × (1 - F) / (1 - σ_i × F)` where `F = exp(-J_w / ω_i)`
- Ion library with reflection coefficients (σ): Na+ (0.98), Ca2+ (0.99), Mg2+ (0.99), Cl- (0.98), SO4²- (0.995), HCO3- (0.97), SiO2 (0.90), B (0.65 for RO, 0.30 for NF)
- Mass balance for concentrate: `C_c,i = (Q_f × C_f,i - Q_p × C_p,i) / (Q_f - Q_p)` for each ion
- Charge balance verification: cation-anion equivalence check with <5% error tolerance
- Tests: single-element with multi-ion feed (brackish water), verify ion-specific rejection percentages, verify total TDS calculation from individual ions

**Day 8: Scaling prediction models**
- `solver-core/src/scaling/lsi.rs` — Langelier Saturation Index for CaCO3: `LSI = pH - pH_s` where `pH_s = (pK₂ - pK_sp) + pCa + pAlk`, temperature-dependent equilibrium constants
- `solver-core/src/scaling/sulfate.rs` — CaSO4 saturation index: `SI = log₁₀([Ca²⁺] × [SO₄²⁻] / K_sp)`, includes gypsum (K_sp = 10^-4.6 at 25°C) and anhydrite forms, temperature correction
- `solver-core/src/scaling/silica.rs` — SiO2 solubility model: temperature and pH-dependent solubility limit (120-180 mg/L at pH 7-9, 25°C), polymerization kinetics
- `solver-core/src/scaling/mod.rs` — Combined scaling assessment: if LSI > 0 → CaCO3 risk, if SI_CaSO4 > 0.2 → gypsum risk, if SiO2 > 80% solubility → silica risk. Antiscalant recommendation: sulfuric acid for high LSI, antiscalant polymer (Genesys LF, Hypersperse) for CaSO4/SiO2
- Tests: verify LSI calculation against ASTM D3739 standard examples, CaSO4 saturation at various temperatures (20-40°C) vs. OLI Studio reference data

**Day 9: Pressure drop and energy calculations**
- `solver-core/src/pressure_drop.rs` — Schock-Miquel correlation: `ΔP = α × Q² + β × Q` with element-specific coefficients (α ≈ 0.0002 bar/(m³/h)², β ≈ 0.01 bar/(m³/h) for 8040 spiral-wound)
- Pressure drop accumulation along vessel, verify total pressure loss < 1 bar for 6-element vessel
- `solver-core/src/energy.rs` — Pump power calculation: `P_pump = ΔP × Q / (3600 × η_pump)` in kW with pump efficiency (typically 85%)
- ERD modeling: Pressure exchanger (PX) with efficiency curve (96% at nominal flow, degrades at low flow), turbocharger (85% efficiency), Pelton turbine (80% efficiency)
- Specific energy consumption: `SEC = (P_pump - P_erd) / Q_product` in kWh/m3
- Tests: verify SEC for typical brackish RO (~0.8 kWh/m3 without ERD), seawater RO with PX (~1.6 kWh/m3), energy balance closure (energy in = energy recovered + energy dissipated)

**Day 10: Element-by-element solver**
- `solver-core/src/ro_solver.rs` — Main iterative solver: for each element, converge on water flux using Newton-Raphson-like fixed-point iteration
- Convergence loop: assume initial flux (20 LMH) → calculate CP modulus → calculate wall concentration → calculate osmotic pressure → calculate new flux from solution-diffusion → check tolerance (0.01 LMH) → repeat until converged (typically 3-7 iterations)
- Element chaining: concentrate from element N becomes feed for element N+1, pressure drops by ΔP_drop
- Solver options: max_iterations (default 50), flux_tolerance (0.01 LMH), min_flux_threshold (2 LMH to prevent negative flux errors)
- Tests: 6-element pressure vessel at 75% recovery, verify flux drops from ~22 LMH (first element) to ~12 LMH (last element) due to increasing concentration and osmotic pressure

**Day 11: Multi-stage array configurations**
- `solver-core/src/array_config.rs` — Parse staging notation: "2:1" means 2 vessels in stage 1, 1 vessel in stage 2 (concentrate from stage 1 feeds stage 2), "3:2:1" for three stages
- Stage-to-stage flow routing: permeate from all stages blends, concentrate flows sequentially through stages
- Permeate blending: `TDS_blend = Σ(Q_p,stage × TDS_p,stage) / Σ Q_p,stage` weighted by flow
- Concentrate recirculation mode: route fraction of final concentrate back to feed (increases overall recovery but requires higher feed pressure)
- Two-pass design: stage 1 permeate becomes stage 2 feed (for ultra-pure water production, <10 mg/L TDS)
- Tests: two-stage 2:1 array, verify stage 1 recovery ~50%, stage 2 recovery ~50% of stage 1 concentrate, overall recovery ~75%

**Day 12: System assembly and validation**
- `solver-core/src/system.rs` — Build complete membrane system from flow diagram JSON: parse components (membrane arrays, pumps, ERD), extract connectivity (pipes between components), construct solver input (elements list, staging config, feed conditions)
- System-level metrics: overall recovery `R = Q_permeate_total / Q_feed`, average flux `J_avg = Q_permeate_total / A_total`, permeate blend TDS, max LSI/SI across all elements
- Validation against ROSA 9.1 output: benchmark brackish RO system (6× BW30-400, 15 bar, 2000 mg/L NaCl, 25°C, 75% recovery)
- ROSA comparison: recovery within 2%, permeate TDS within 5%, element-by-element flux profile within 10%
- Tests: reproduce ROSA results for 3 benchmark cases (brackish RO, seawater RO first pass, seawater RO second pass), verify element-by-element pressure, flux, and concentration profiles

### Phase 3 — WASM Build + Frontend (Days 13–18)

**Day 13: WASM solver compilation**
- `solver-wasm/` — New workspace member for WASM target, depends on solver-core
- `solver-wasm/Cargo.toml` — wasm-bindgen = "0.2", serde-wasm-bindgen = "0.6", js-sys = "0.3", console_log for browser logging
- `solver-wasm/src/lib.rs` — WASM entry points with #[wasm_bindgen]: `solve_ro_system(flow_diagram_json, feed_water_json, design_params_json) -> Result<JsValue>`, `calculate_scaling_indices(water_composition_json, recovery) -> JsValue`
- Build pipeline: `wasm-pack build --target web --release` generates pkg/ directory with .wasm + .js bindings
- Optimization: `wasm-opt -Oz` for size (target <500KB gzipped), strip debug symbols, LTO enabled
- JavaScript wrapper class `MembraneSolver`: async `init()` loads WASM module, `solve(params)` calls WASM entry point, error handling with Rust Result → JS Promise rejection
- Versioned WASM bundles: `membraneflow-solver-v1.0.0.wasm` with cache busting

**Day 14: Frontend scaffold and flow diagram foundation**
```bash
npm create vite@latest frontend -- --template react-ts
cd frontend
npm i zustand @tanstack/react-query axios d3 @types/d3
```
- `src/App.tsx` — React Router with routes (/, /dashboard, /editor/:id, /membranes, /billing, /login), auth context provider, Tanstack Query client
- `src/stores/projectStore.ts` — Zustand store: projects array, currentProject, actions (setCurrentProject, updateFlowDiagram, saveProject with debounce)
- `src/stores/flowDiagramStore.ts` — Flow diagram state: components (arrays, pumps, ERD), connections (pipes with from/to ports), selection state, actions (addComponent, deleteComponent, addConnection, updateComponentConfig)
- `src/components/FlowDiagramEditor/Canvas.tsx` — SVG canvas with viewBox for pan/zoom, onWheel handler for zoom (scale factor 1.1/0.9), onMouseDown/Move/Up for pan (matrix transform)
- `src/components/FlowDiagramEditor/Grid.tsx` — Snap-to-grid overlay: SVG pattern with configurable spacing (10px/20px/50px), rendered as background rect with pattern fill

**Day 15: Flow diagram editor — component palette and drag-drop**
- `src/components/FlowDiagramEditor/ComponentPalette.tsx` — Sidebar with categorized components: Membrane Arrays (single-stage, two-stage), Pumps (high-pressure, booster), ERD (PX, turbocharger), Valves (control, check)
- `src/components/FlowDiagramEditor/MembraneArray.tsx` — SVG group: rectangle (200×80px) representing vessel array, text labels (elements per vessel, total vessels, staging "2:1"), connection ports (inlet, permeate, concentrate) as small circles
- `src/components/FlowDiagramEditor/Pump.tsx` — SVG pump symbol (triangle with circle), pressure annotation, efficiency indicator, inlet/outlet ports
- `src/components/FlowDiagramEditor/EnergyRecoveryDevice.tsx` — SVG ERD symbol (dual-chamber cylinder for PX), efficiency curve indicator, high-pressure and low-pressure ports
- Drag-and-drop: HTML5 Drag and Drop API from palette → SVG canvas, calculate drop position in SVG coordinates (accounting for viewBox transform), snap to grid
- Property editor panel: slide-out panel on right, shows selected component properties (membrane model dropdown, elements per vessel number input, staging text input), live updates flow diagram state
- Component rotation (R key): rotate SVG group by 90° increments, update port positions, deletion (Del key): remove component and connected pipes

**Day 16: Flow diagram editor — piping and connections**
- `src/components/FlowDiagramEditor/Pipe.tsx` — SVG path renderer: calculate Manhattan routing (horizontal → vertical → horizontal segments avoiding components), arrow markers for flow direction, stroke color based on stream type (feed: blue, permeate: green, concentrate: orange)
- `src/components/FlowDiagramEditor/ConnectionTool.tsx` — Click-to-connect mode: click source port → drag to target port → create connection object with from{componentId, port} and to{componentId, port}
- Auto-routing algorithm: A* pathfinding on grid with component bounding boxes as obstacles, prefer horizontal-then-vertical paths, avoid crossing other pipes if possible
- Flow annotation: after simulation, display calculated flow rate (m3/h) and TDS (mg/L) as text labels on pipes
- Connection validation: prevent invalid connections (permeate → feed, concentrate → permeate), enforce single connection per port
- Undo/redo stack: Zustand middleware tracks state changes, Ctrl+Z/Y keyboard shortcuts, max 50 undo levels
- Keyboard shortcuts: Ctrl+S save (debounced API call), Del delete selected, R rotate, Escape clear selection, Arrow keys nudge selected component

**Day 17: Feed water characterization wizard**
- `src/components/FeedWaterWizard/WaterQualityForm.tsx` — Multi-step form: Step 1 (basic parameters): TDS slider (100-50,000 mg/L), pH input (4-11), temperature (5-40°C), SDI optional (0-6.5)
- `src/components/FeedWaterWizard/IonComposition.tsx` — Step 2 (detailed composition): input fields for Na, Ca, Mg, Cl, SO4, HCO3, SiO2, B in mg/L, real-time ion balance check (cation meq/L vs. anion meq/L), warning if imbalance >5%
- `src/components/FeedWaterWizard/WaterAnalysisLibrary.tsx` — Step 3 (save or load): save current composition with name, load from user's saved analyses, load from public library (20+ standard waters)
- Water type presets: "Typical Brackish Groundwater" (TDS 2000, Ca 80, Mg 40, Na 400, Cl 600, SO4 200, HCO3 150), "Colorado River Water", "Gulf Seawater" (TDS 35000), "Municipal Tap Water" (TDS 300)
- Ion balance validation: calculate cation equivalents `Σ(c_i × z_i / MW_i)`, anion equivalents, require <5% difference, display error message and highlight imbalanced ions
- CSV import: parse CSV with columns (ion_name, concentration_mg_L), map to WaterComposition object, validate ranges

**Day 18: Performance plots with D3.js**
- `src/components/PerformancePlots/PressureProfile.tsx` — D3 line chart: X-axis = element position (0 to N), Y-axis = pressure (bar), line shows pressure decline along array, threshold line at max operating pressure, interactive tooltip on hover showing exact pressure at each element
- `src/components/PerformancePlots/FluxProfile.tsx` — D3 line chart: Y-axis = water flux (LMH), shows flux decline due to increasing osmotic pressure, average flux horizontal line, color gradient (green above average, orange below)
- `src/components/PerformancePlots/ConcentrationProfile.tsx` — D3 multi-line chart: Y-axis = concentration (mg/L) with log scale option, 8 lines for individual ions (Na, Ca, Mg, Cl, SO4, HCO3, SiO2, B), legend with color coding, toggle visibility per ion
- `src/components/PerformancePlots/ScalingProfile.tsx` — D3 multi-line chart: Y-axis = saturation index (dimensionless), lines for LSI, SI_CaSO4, SI_SiO2, horizontal threshold lines (LSI 0, SI 0.2), red warning zone if any SI > threshold, tooltip shows recommended antiscalant
- D3 zoom behavior: `d3.zoom()` with `scaleExtent([1, 10])`, zoom in/out on mouse wheel, pan on drag, update X/Y scales on zoom/pan events
- Interactive tooltips: `d3.pointer(event)` to get mouse position, binary search to find nearest data point, display tooltip div with element ID, pressure, flux, TDS, scaling indices
- Pan/zoom: D3 zoom behavior with constrained extent (don't zoom past data bounds), synchronized zoom across all plots (same X-axis), reset zoom button

### Phase 4 — API + Job Orchestration (Days 19–24)

**Day 19: Simulation API endpoints**
- `src/api/handlers/simulation.rs` — POST /projects/:id/simulations (create simulation), GET /simulations/:id (get status and results), GET /projects/:id/simulations (list simulations for project), DELETE /simulations/:id (cancel running simulation)
- System analysis: parse flow diagram JSON → extract components → count total membrane elements → determine WASM vs. server execution mode (threshold: 100 elements)
- Plan limit enforcement: free plan (≤50 elements, 5 projects), pro plan (≤200 elements, unlimited projects, 10 server simulations/month), advanced plan (unlimited elements, unlimited simulations)
- Request validation: check feed water composition is valid (all required ions present), check design parameters are within membrane spec limits (pressure < max_feed_pressure, temperature < max_feed_temperature)
- Response: return simulation ID immediately, status "pending" for server execution or "completed" for WASM (client-side execution happens in browser)

**Day 20: Server-side simulation worker**
- `src/workers/simulation_worker.rs` — Background worker process that runs in separate Docker container, polls Redis job queue with BRPOP (blocking pop, 30s timeout)
- Redis job queue: LPUSH to "simulation:jobs" list with simulation_id as payload, worker BRPOP from queue, FIFO processing
- Worker lifecycle: fetch simulation record from PostgreSQL → fetch project (flow diagram, feed water) → build system from flow diagram → run native solver → upload results to S3 → update simulation status to "completed"
- Concurrency control: multiple worker pods (K8s deployment with replicas: 2-20 based on queue depth), each worker processes one job at a time
- Progress streaming: every 10 elements solved, publish progress update to Redis pub/sub channel "simulation:progress:{sim_id}" with `{progress_pct, elements_completed, current_stage}`
- Error handling: catch solver errors (convergence failures, negative flux, invalid configuration) → update simulation status to "failed" → store error_message in database
- S3 result upload: serialize ElementResult array to JSON (~1KB per element) → gzip compression → upload to `s3://membraneflow-results/results/{project_id}/{sim_id}.json.gz` → presigned URL (7-day expiry) for client download
- Timeout: 5-minute max per simulation, kill worker process if exceeded (prevents hung jobs from blocking queue)

**Day 21: WebSocket for live simulation progress**
- `src/api/ws/mod.rs` — Axum WebSocket upgrade handler at GET /ws/simulations/:id, upgrade HTTP connection to WebSocket using `axum::extract::ws::WebSocketUpgrade`
- `src/api/ws/simulation_progress.rs` — Subscribe to Redis pub/sub channel "simulation:progress:{sim_id}" after WebSocket connection established, forward progress messages to WebSocket client
- Message format: `{"type": "progress", "progress_pct": 45.2, "elements_completed": 45, "current_stage": 1}`, `{"type": "completed", "results_url": "s3://...", "summary": {...}}`
- Connection management: heartbeat ping/pong every 30 seconds, close connection on pong timeout (60s), cleanup Redis subscription on disconnect
- Client reconnection: if WebSocket drops, client retries with exponential backoff (1s, 2s, 4s, 8s max), server handles duplicate subscriptions gracefully
- Frontend: `src/hooks/useSimulationProgress.ts` — React hook that opens WebSocket connection, listens for progress events, updates Zustand store, displays progress bar in UI, handles reconnection

**Day 22: Membrane catalog API**
- `src/api/handlers/membranes.rs` — GET /membranes (list with filters), GET /membranes/:id (get details), GET /membranes/search?q={query} (full-text search), GET /membranes/compare?ids={id1,id2,id3} (comparison table)
- Parametric search: filter by manufacturer (DuPont, Toray, Hydranautics, LG Chem), membrane_type (ro, nf), application (brackish_water, seawater, industrial), form_factor (spiral_wound_8040, spiral_wound_4040), active_area (>= 37 m2 for 8040, >= 7.2 m2 for 4040)
- Full-text search: PostgreSQL pg_trgm trigram index on model name, fuzzy matching with `ILIKE '%query%'` on model and manufacturer, rank by similarity
- Pagination: cursor-based pagination with `?cursor={last_id}&limit=50`, return next_cursor in response for infinite scroll
- Membrane comparison mode: GET /membranes/compare?ids=uuid1,uuid2,uuid3 → return side-by-side table with active_area, A_coeff, B_coeff, salt_rejection, max_pressure, price_per_element (if available)
- Response includes: membrane specifications, transport coefficients (A, B from datasheets), datasheet_url (link to manufacturer PDF), download_count for popularity ranking

**Day 23: PDF report generation**
- `src/services/report_generator.rs` — Generate PDF design reports using headless Chrome (via `headless_chrome` Rust crate) or `wkhtmltopdf` CLI
- Report template: HTML/CSS template with sections: (1) System Overview (membrane count, staging, recovery, SEC), (2) Feed Water Summary (TDS, pH, temperature, ions table), (3) Element-by-Element Performance (table with 20 columns: element_id, pressure, flow, flux, permeate_tds, LSI, etc.), (4) Scaling Projections (max LSI/SI, antiscalant recommendation), (5) Energy Consumption (pump power, ERD recovery, SEC breakdown), (6) P&ID Diagram (embed flow diagram SVG)
- Template rendering: use Handlebars or Tera templating engine, pass simulation results as context, render HTML string
- PDF generation: launch headless Chrome with `--headless --print-to-pdf` flags, load HTML from string, wait for page load, export PDF to temp file
- S3 upload: upload PDF to `s3://membraneflow-reports/reports/{project_id}/{sim_id}.pdf`, presigned URL for download, store report_url in design_reports table
- Professional formatting: company logo placeholder, page numbers, table of contents, page breaks before each section, consistent fonts (Helvetica for headings, Arial for body)

**Day 24: Water analysis library**
- `src/api/handlers/water_analyses.rs` — POST /water-analyses (save composition), GET /water-analyses (list user's saved analyses), GET /water-analyses/public (public library), GET /water-analyses/:id, DELETE /water-analyses/:id
- Public library seeding: insert 20+ standard water compositions: "Colorado River Water" (TDS 750, Ca 70, Mg 30, Na 100, Cl 80, SO4 250, HCO3 150), "Typical Brackish Groundwater - Southwest US", "Gulf of Mexico Seawater" (TDS 35000, Na 10800, Cl 19400, SO4 2700, Mg 1300, Ca 410), "Municipal Tap Water - East Coast", "Industrial Wastewater - Food Processing"
- CSV import endpoint: POST /water-analyses/import with multipart/form-data CSV file, parse CSV rows (ion_name, concentration_mg_L), validate ion names, map to WaterComposition, return parsed object for preview before save
- CSV export endpoint: GET /water-analyses/:id/export → generate CSV with headers (Ion, Concentration_mg_L, Molecular_Weight, Equivalents_meq_L), download as attachment
- Ion balance validation: backend checks cation-anion balance, returns error if >5% imbalance, suggests corrections (e.g., "Add 50 mg/L Cl- to balance cations")

### Phase 5 — Results + Visualization (Days 25–29)

**Day 25: Simulation result processing**
- Result parsing: download gzipped JSON from S3 → decompress → parse ElementResult array (100-element system = ~100KB uncompressed)
- Result caching: hot results (last 100 accessed simulations) cached in Redis with 1-hour TTL, cache key `sim:results:{sim_id}`, cold results fetched from S3
- Presigned S3 URLs: generate 7-day presigned GET URL for client-side direct download (bypasses API server, reduces bandwidth), signed with AWS SigV4
- Result summary generation: calculate overall recovery, permeate TDS (weighted average), specific energy, max LSI/SI, store in results_summary JSONB column for fast dashboard queries without fetching full results
- Summary metrics: overall_recovery (fraction), overall_permeate_tds (mg/L), specific_energy (kWh/m3), max_lsi, max_si_caso4, max_si_sio2, antiscalant_required (boolean), first_element_flux (LMH), last_element_flux (LMH), pressure_drop_total (bar)

**Day 26: Performance plot interactions**
- Click element in plot → highlight element in flow diagram: add red border to corresponding membrane array element SVG rect
- Click element in flow diagram → highlight point in plots: add vertical line marker on all D3 plots at element position, show crosshair tooltip with values
- Multi-series toggle: checkboxes in plot legend to show/hide individual ions in concentration profile, preserve zoom state when toggling
- Export plot data to CSV: button "Export to CSV" → generate CSV with columns (element_id, pressure, flux, permeate_tds, lsi, si_caso4, ...), download as `{project_name}_results.csv`
- Copy plot as PNG: use html2canvas library to render D3 SVG → canvas → PNG blob, copy to clipboard or download, resolution 1920×1080 for reports

**Day 27: Scaling and antiscalant recommendations**
- `src/components/ScalingAnalysis/ScalingSummary.tsx` — Table with rows for each scalant (CaCO3, CaSO4, BaSO4, SrSO4, SiO2), columns (Max SI, Element Location, Risk Level), color-coded risk (green <0, yellow 0-0.5, orange 0.5-1.0, red >1.0)
- `src/components/ScalingAnalysis/AntiscalantDosing.tsx` — Recommendation panel: if LSI > 0.5 → suggest "Sulfuric acid dosing to reduce pH to 7.0", if SI_CaSO4 > 0.8 → suggest "Antiscalant polymer (Genesys LF or Hypersperse) at 3-5 ppm", if SiO2 > 150 mg/L → suggest "Reduce recovery or use specialty silica-dispersant antiscalant"
- Antiscalant database: store common products (Genesys LF, Hypersperse MDC220, Vitec 3000, King Lee KL300) with dose rates (2-5 ppm), cost per kg, effectiveness curves
- Warning indicators: red warning icon next to elements with SI > threshold, tooltip explains scaling risk and mitigation
- What-if analysis: slider to adjust recovery (60-90%) → re-run scaling calculation in real-time (client-side with WASM) → show how LSI/SI change with recovery, help user find maximum safe recovery

**Day 28: Energy and cost breakdown**
- `src/components/CostAnalysis/EnergyBreakdown.tsx` — D3 pie chart: slices for high-pressure pump (65%), booster pump (10%), ERD credit (-25%), auxiliary systems (5%), hover shows kWh/m3 and % of total
- `src/components/CostAnalysis/SpecificEnergy.tsx` — Bar chart comparing SEC to industry benchmarks: user's system (e.g., 1.8 kWh/m3 for seawater RO), industry average (2.2 kWh/m3), best-in-class (1.5 kWh/m3 with advanced ERD), thermodynamic minimum (1.0 kWh/m3 for seawater at 50% recovery)
- Simple CAPEX/OPEX estimator: membrane cost ($800/element × N elements), pump cost ($50,000 for <100 m3/h, $200,000 for 100-1000 m3/h), ERD cost ($100,000 for PX), piping and instrumentation (30% of equipment cost)
- OPEX: energy cost (SEC × flow × hours_per_year × electricity_rate), membrane replacement (replace every 3-7 years, cost × N elements / lifetime_years), chemicals (antiscalant $2/kg, consumption 3 ppm), labor ($50,000/year for operator)
- Levelized cost of water (LCOW) calculator: `LCOW = (CAPEX × CRF + OPEX) / annual_production_m3` where CRF = capital recovery factor based on discount rate (8%) and plant lifetime (20 years)
- Assumptions panel: user can adjust electricity rate ($/kWh), membrane lifetime (years), discount rate (%), labor cost ($/year), view impact on LCOW in real-time

**Day 29: Design templates and examples**
- `src/data/templates/` — JSON files for pre-built systems: `brackish_ro_single_stage.json` (60 elements in 10×6 array, 75% recovery, 15 bar, BW30-400 membranes), `seawater_ro_two_stage.json` (2:1 array, 120 elements total, 45% recovery, 60 bar, SW30HR-380 membranes, PX ERD), `nf_softening.json` (NF90-400 elements, remove hardness, 85% recovery), `high_recovery_brackish.json` (85% recovery with concentrate recirculation, 80 elements, antiscalant required)
- Template metadata: name, description, typical application, expected performance (recovery, permeate TDS, SEC), thumbnail image (pre-rendered P&ID)
- Template gallery page: grid layout with cards, each card shows thumbnail, name, key specs (elements, recovery, membrane type), "Use Template" button
- One-click template usage: click "Use Template" → create new project with flow diagram pre-populated → feed water pre-set to typical composition for that application → run simulation automatically to show expected performance
- Example simulations: pre-run simulation results stored with template, display immediately on template load (no wait time), user can modify parameters and re-run

### Phase 6 — Billing + Plan Enforcement (Days 30–33)

**Day 30: Stripe integration**
- `src/api/handlers/billing.rs` — POST /billing/checkout (create Stripe Checkout Session), GET /billing/portal (create Customer Portal Session for subscription management), GET /billing/subscription (get current subscription status)
- `src/api/handlers/webhooks/stripe.rs` — POST /webhooks/stripe with Stripe signature verification, handle events: `checkout.session.completed` (user subscribed → update user.plan and stripe_subscription_id), `customer.subscription.updated` (plan changed → update user.plan), `customer.subscription.deleted` (subscription cancelled → downgrade to free), `invoice.payment_failed` (payment failed → send email warning, grace period 7 days)
- Plan mapping with Stripe Price IDs: Free (plan: "free", price_id: null, $0/month), Pro (plan: "pro", price_id: "price_xxx", $99/month), Advanced (plan: "advanced", price_id: "price_yyy", $249/user/month)
- Stripe Checkout Session creation: specify success_url (redirect to /dashboard?session_id={CHECKOUT_SESSION_ID}), cancel_url (redirect to /billing), customer_email, subscription mode (not one-time payment)
- Stripe Customer Portal: allows users to update payment method, view invoices, cancel subscription, hosted by Stripe (no custom UI needed)
- Webhook signature verification: validate `Stripe-Signature` header using webhook secret, prevent replay attacks, return 400 if signature invalid

**Day 31: Usage tracking and plan limits**
- `src/middleware/plan_limits.rs` — Tower middleware that checks plan limits before expensive operations (simulation creation, PDF export, optimization)
- Limit definitions: free (≤50 elements per system, 5 projects max, 0 server simulations/month), pro (≤200 elements, unlimited projects, 10 server simulations/month, PDF export), advanced (unlimited elements, unlimited projects, unlimited server simulations, optimization, API access)
- `src/services/usage.rs` — Track server simulation runs: after each server simulation completes, insert usage record with record_type="server_simulation", quantity=1, period_start=beginning of billing month, period_end=end of billing month
- Usage aggregation query: `SELECT SUM(quantity) FROM usage_records WHERE user_id = $1 AND record_type = 'server_simulation' AND period_start <= CURRENT_DATE AND period_end >= CURRENT_DATE` to get current month usage
- Usage dashboard endpoint: GET /usage → return current month usage broken down by record_type (server_simulations, pdf_exports, optimization_runs), compare to plan limits, return percentage used
- Approaching-limit warnings: when usage reaches 80% of limit, display banner in UI "You've used 8/10 server simulations this month. Upgrade to Advanced for unlimited.", at 100% display error modal and block action
- Grace period for downgrade: if user downgrades from Pro to Free mid-month, allow existing projects >5 to remain (read-only) until next billing period, then enforce limit

**Day 32: Billing UI**
- `frontend/src/pages/Billing.tsx` — Page layout: current plan card on left (plan name, price, renewal date, "Manage Subscription" button → Stripe Portal), plan comparison table on right (3 columns for Free/Pro/Advanced), usage meter below
- `frontend/src/components/billing/PlanCard.tsx` — Card showing plan features as bullet list: Free (50-element systems, 5 projects, basic membranes), Pro (200-element systems, unlimited projects, 10 server sims/month, PDF reports, full membrane database), Advanced (unlimited elements, unlimited sims, optimization, collaboration, API access)
- `frontend/src/components/billing/UsageMeter.tsx` — Progress bar showing server simulation usage (e.g., "7/10 server simulations used this month"), color-coded (green <70%, yellow 70-90%, orange 90-100%, red ≥100%), displays next reset date ("Resets March 1, 2026")
- Upgrade/downgrade flow: click "Upgrade to Pro" → POST /billing/checkout with price_id for Pro plan → redirect to Stripe Checkout → after payment, redirect to /dashboard?upgraded=true → show success message
- Manage subscription: click "Manage Subscription" → POST /billing/portal → redirect to Stripe Customer Portal → user can cancel, change payment method, view invoices
- Upgrade prompt modals: when user hits plan limit (e.g., tries to create 120-element system on Pro plan), show modal "Your system has 120 elements. Pro plan supports up to 200. Upgrade to Advanced for unlimited." with "Upgrade" button (goes to checkout) and "Cancel" button

**Day 33: Feature gating**
- PDF report export: check `if user.plan == "free" { return Err(ApiError::PlanLimit("Upgrade to Pro for PDF reports")) }` before generating PDF, show lock icon on "Export PDF" button for free users
- Optimization features: gate POST /projects/:id/optimize endpoint behind advanced plan, show "Optimization (Advanced only)" in UI with upgrade prompt
- API access: gate /api/v1/* endpoints (for programmatic access) behind advanced plan, return 403 Forbidden with upgrade message for non-advanced users
- Collaboration/org features: gate org creation and member invites behind advanced plan, show "Team Collaboration (Advanced only)" in navigation
- Show locked feature indicators: lock icon overlays on buttons, "Upgrade to unlock" tooltips, disabled state with upgrade CTA in modal on click
- Admin override endpoint: POST /admin/users/:id/override-plan with admin API key → set user plan without Stripe (for beta testers, partners, academic licenses, lifetime deals)
- Feature flag system: JSONB column in users table `feature_flags` for custom feature access (e.g., `{"early_access_gas_separation": true}`), check flags in addition to plan

### Phase 7 — Solver Validation (Days 34–38)

**Day 34: Solver validation — single-element benchmarks**
- Benchmark 1: DuPont FilmTec BW30-400 at standard test conditions (15 bar, 2000 mg/L NaCl, 25°C, 16 m3/h feed flow) → expected permeate flow 6.8 m3/h (from datasheet) → verify solver output within 3%, permeate TDS ~40 mg/L (98% rejection) within 5%
- Benchmark 2: Toray TM720D-400 seawater element at 55 bar, 35,000 mg/L TDS, 25°C → expected permeate flow 1.0 m3/h (from Toray datasheet) → verify within 5%, permeate TDS ~175 mg/L (99.5% rejection) within 8%
- Benchmark 3: Hydranautics CPA3-8040 NF element with hardness removal → Ca rejection ~95%, Mg rejection ~96%, NaCl rejection ~45% → verify multi-ion Spiegler-Kedem model correctly predicts differential rejection
- Automated test suite: `solver-core/tests/benchmarks.rs` with test cases for each membrane, assert results within tolerance, CI/CD runs tests on every commit
- Temperature correction validation: test same membrane at 15°C, 25°C, 35°C → verify flux scales correctly with temperature (10-15% increase per 10°C) using temperature correction factor

**Day 35: Solver validation — multi-element arrays**
- Benchmark 4: 6-element pressure vessel (6× BW30-400 in series) at 15 bar, 2000 mg/L NaCl, 75% recovery → compare to ROSA 9.1 output (canonical industry tool)
- ROSA comparison metrics: overall recovery (ROSA: 75.0%, solver: within 2%), permeate TDS (ROSA: 38 mg/L, solver: within 5%), element-by-element flux profile (first element ~22 LMH, last element ~12 LMH, solver: within 10% per element)
- Benchmark 5: Two-stage 2:1 seawater RO array (12 elements total: 8 in stage 1, 4 in stage 2) at 60 bar, 35,000 mg/L TDS, 45% recovery → compare to ROSA for stage-wise recovery distribution (stage 1: 30%, stage 2: 15% of stage 1 concentrate)
- Benchmark 6: High-recovery brackish system with concentrate recirculation (80 elements, 85% recovery) → verify scaling predictions match published recovery limits for brackish water (LSI limit ~1.5 with antiscalant, CaSO4 SI limit ~0.8)
- TorayDS comparison: run same configuration in TorayDS (for Toray membranes) → verify permeate quality and pressure profile match within 8%
- Element-by-element validation: for each element in 6-element vessel, verify pressure (tolerance 0.5 bar), flux (tolerance 2 LMH), concentrate TDS (tolerance 10%), LSI (tolerance 0.3 units)

**Day 36: Solver validation — scaling and energy**
- LSI calculation validation: test cases from Dow Water & Process Solutions "FILMTEC Reverse Osmosis Membranes Technical Manual" Table 6 (LSI examples) → input feed water composition → verify calculated LSI matches published values within 0.2 units
- CaSO4 saturation index validation: compare to OLI Studio (commercial water chemistry software) reference calculations for various Ca/SO4 concentrations and temperatures → verify SI_CaSO4 within 0.15 units
- Temperature effect on scaling: test CaCO3 solubility (K_sp) at 15°C, 25°C, 35°C → verify LSI changes correctly with temperature (LSI decreases ~0.1 per 5°C increase due to lower K_sp)
- Energy recovery validation: PX pressure exchanger with 96% efficiency at 60 bar feed, 58 bar concentrate → expected energy recovery ~1.9 kWh/m3 → verify SEC calculation (pump power - ERD recovery) within 5%
- Pressure drop accumulation: 100-element array (10 vessels × 10 elements) at 16 m3/h per vessel → expected total pressure drop ~8-10 bar → verify cumulative pressure drop from Schock-Miquel correlation within 10%
- Energy balance closure: total energy input (pump) = energy in permeate + energy in concentrate + energy dissipated (pressure drop) → verify balance closes within 2%

**Day 37: Integration testing**
- End-to-end test suite using Rust integration tests and Playwright for frontend:
  1. Register new user → verify user created in database, JWT issued
  2. Create project → update flow diagram (add membrane array, pump, pipes) → save → reload page → verify flow diagram state preserved
  3. Set feed water composition → verify water analysis saved
  4. Run WASM simulation (30-element system) → verify results returned in <500ms, plots render
  5. Run server simulation (150-element system) → job queued → worker picks up → WebSocket receives progress updates → results in S3 → download via presigned URL
  6. Search membrane catalog "BW30" → verify results returned → select membrane → use in array
  7. Generate PDF report → verify PDF uploaded to S3, download works
  8. Subscribe to Pro plan (Stripe test mode) → webhook updates user plan → limits lifted → can now run 200-element system
- WASM solver test: load WASM bundle in headless Chrome (Playwright) → run simulation → verify results match native Rust solver output (bit-for-bit identical within floating-point precision)
- WebSocket test: connect to /ws/simulations/:id → submit server simulation → verify progress messages received in order → verify "completed" message with results_url
- Concurrent simulation test: submit 5 simultaneous server-side jobs → verify all complete correctly without race conditions, Redis queue handles concurrency

**Day 38: Performance testing and optimization**
- WASM solver benchmarks on multiple browsers (Chrome 120, Firefox 120, Safari 17): measure execution time for 20-element (target <200ms), 50-element (target <300ms), 100-element (target <500ms) systems
- Performance regression tests: track solver time over Git commits, fail CI if performance degrades >10% without justification
- Server solver benchmarks on reference hardware (AWS c6i.xlarge: 4 vCPUs, 8GB RAM): 100-element (target <2s), 500-element (target <5s), 1000-element (target <15s)
- Server throughput test: run 20 concurrent simulations → measure simulations completed per minute (target >10 with 5 worker pods)
- Frontend rendering performance: measure D3 plot rendering time for 100-element result (target <100ms), 500-element result (target <300ms), test on 60Hz and 120Hz displays
- Memory profiling: WASM solver heap usage (target <50MB peak for 100-element system), check for memory leaks (run 100 consecutive simulations, verify memory returns to baseline)
- Load testing with k6: simulate 50 concurrent users (each creates project, runs simulation, views results) → measure API p95 latency (target <500ms), database connection pool utilization (<80%), error rate (<0.1%)

### Phase 8 — Deployment + Launch (Days 39–42)

**Day 39: Docker and Kubernetes configuration**
- Multi-stage `Dockerfile` for Rust API:
  ```dockerfile
  FROM rust:1.75 AS chef
  RUN cargo install cargo-chef
  FROM chef AS planner
  COPY . .
  RUN cargo chef prepare --recipe-path recipe.json
  FROM chef AS builder
  COPY --from=planner /app/recipe.json recipe.json
  RUN cargo chef cook --release --recipe-path recipe.json
  COPY . .
  RUN cargo build --release
  FROM debian:bookworm-slim
  RUN apt-get update && apt-get install -y ca-certificates libssl3 && rm -rf /var/lib/apt/lists/*
  COPY --from=builder /app/target/release/membraneflow-api /usr/local/bin/
  EXPOSE 8000
  CMD ["membraneflow-api"]
  ```
- `docker-compose.yml` for local development: PostgreSQL 16 (persistent volume), Redis 7 (ephemeral), MinIO (S3-compatible, port 9000), Adminer (database UI, port 8080), API server (port 8000), simulation worker (2 replicas)
- Kubernetes manifests in `k8s/`:
  - `api-deployment.yaml`: 3 replicas, HorizontalPodAutoscaler (target CPU 70%, min 3, max 10), health checks (livenessProbe: /health/live, readinessProbe: /health/ready), resource limits (memory: 512Mi, CPU: 500m)
  - `worker-deployment.yaml`: 2 replicas, HPA based on Redis queue depth (custom metric via Prometheus adapter, 1 worker per 5 queued jobs, min 2, max 20), resource limits (memory: 1Gi, CPU: 1000m for solver compute)
  - `postgres-statefulset.yaml`: StatefulSet with PersistentVolumeClaim (100Gi), PostgreSQL 16 with replication (1 primary + 2 read replicas for scaling reads), backup CronJob (daily dump to S3)
  - `redis-deployment.yaml`: Redis Cluster mode (3 master nodes, 3 replica nodes) for high availability, persistence enabled (AOF + RDB)
  - `ingress.yaml`: NGINX Ingress Controller with TLS termination (Let's Encrypt cert-manager), rate limiting (100 req/min per IP), path routing (/api → API service, /ws → WebSocket service)
- Health check endpoints: GET /health/live (return 200 if server running), GET /health/ready (return 200 if database and Redis connections healthy, otherwise 503)
- Graceful shutdown: handle SIGTERM signal, stop accepting new requests, finish in-flight requests (30s timeout), close database connections, exit

**Day 40: CDN and WASM delivery**
- CloudFront distribution setup:
  - Origin 1: S3 bucket for frontend static assets (HTML, JS, CSS), behavior pattern `/*`, caching (max-age 1 hour for HTML, 1 year for JS/CSS with hash-based cache busting), gzip compression enabled
  - Origin 2: S3 bucket for WASM solver bundle, behavior pattern `/wasm/*`, caching (max-age 1 year for versioned WASM files `membraneflow-solver-v1.2.3.wasm`), Brotli compression (better than gzip for WASM, ~30% size reduction)
  - Origin 3: S3 bucket for membrane datasheets, behavior pattern `/datasheets/*`, caching (max-age 1 day), signed URLs for private datasheets
- WASM bundle versioning: include version hash in filename (e.g., `membraneflow-solver-abc123.wasm`), update HTML script tag on deploy, CloudFront invalidation not needed (new filename = cache miss)
- WASM preloading strategy: add `<link rel="modulepreload" href="/wasm/membraneflow-solver-abc123.wasm">` to HTML head → browser starts downloading WASM bundle on page load before JavaScript execution → reduces time-to-interactive by 500ms
- Service worker for offline WASM caching: register service worker on first visit, intercept fetch requests for `/wasm/*`, cache WASM bundle in CacheStorage, serve from cache on subsequent visits (even offline), update cache on version change
- CloudFront custom error pages: 404 → show custom "Page not found" with navigation, 503 → show "Service temporarily unavailable" during maintenance

**Day 41: Monitoring, logging, and polish**
- Prometheus metrics exposed at /metrics endpoint (using `prometheus` Rust crate):
  - Counter: `api_requests_total{method, path, status}` (track request volume)
  - Histogram: `api_request_duration_seconds{method, path}` (track latency distribution)
  - Histogram: `solver_duration_seconds{element_count_bucket}` (track solver performance by size)
  - Gauge: `active_websocket_connections` (track concurrent WebSocket clients)
  - Gauge: `redis_queue_depth` (track pending simulation jobs for HPA)
- Grafana dashboards (3 dashboards):
  - **System Health**: CPU/memory/disk usage per pod, request rate, error rate (5xx responses), p50/p95/p99 latency, database connection pool utilization
  - **Simulation Metrics**: simulations per hour (by execution_mode: wasm vs. server), solver duration histogram, convergence failure rate, average element count per simulation
  - **Business Metrics**: new user signups per day, subscription conversions (free → pro → advanced), MRR (monthly recurring revenue) from Stripe webhooks, active users (DAU/MAU)
- Sentry integration for error tracking:
  - Backend: capture panics and errors, attach user context (user_id, plan), release tracking (Git commit SHA), source maps for Rust stack traces
  - Frontend: capture JavaScript errors and unhandled promise rejections, attach breadcrumbs (user actions before error), session replay for debugging, filter out bot errors (user-agent filtering)
- Structured logging with `tracing` crate: JSON-formatted logs with fields (timestamp, level, message, span_id, user_id, request_id), centralized log aggregation (CloudWatch Logs or Datadog), log levels (ERROR for failures, WARN for degraded performance, INFO for important events, DEBUG for development)
- UI polish pass: consistent loading spinners (use `react-loading-skeleton` for content placeholders), error states (friendly error messages, suggest actions, "Report Issue" button), empty states (e.g., "No projects yet. Create your first system!"), responsive layout (mobile-friendly, test on iPhone SE, iPad, desktop)
- Accessibility improvements: keyboard navigation for flow diagram editor (Tab to select, Arrow keys to move, Enter to edit), ARIA labels on interactive SVG elements (`aria-label="Membrane array 1"`), focus indicators (blue outline), screen reader testing with NVDA

**Day 42: Launch preparation and final testing**
- End-to-end smoke test in production staging environment: register user → create project → run simulation → view results → subscribe → upgrade plan → verify all flows work
- Database backup verification: restore from latest backup to separate database instance, verify data integrity (row counts match, foreign key constraints valid), test restore time (<1 hour for 100GB database)
- Rate limiting configuration: Nginx Ingress rate limit annotations `nginx.ingress.kubernetes.io/limit-rps: "10"` per IP, API-level rate limiting with `governor` crate (100 req/min per user, 10 simulations/min), return 429 Too Many Requests with `Retry-After` header
- Security audit checklist: JWT secret rotation, SQL injection prevention (SQLx compile-time checks), XSS prevention (React auto-escapes, CSP header), CSRF protection (SameSite cookies), CORS configuration (allow only membraneflow.com origin), S3 bucket policies (private by default, presigned URLs for access), secrets management (AWS Secrets Manager, never commit secrets to Git)
- Content Security Policy header: `default-src 'self'; script-src 'self' 'wasm-unsafe-eval'; connect-src 'self' wss://api.membraneflow.com; img-src 'self' data: https:; style-src 'self' 'unsafe-inline'` to prevent XSS
- Landing page (marketing site): hero section ("Design RO/NF Systems 10x Faster"), feature highlights (vendor-neutral membranes, instant WASM simulation, scaling prediction), pricing table, testimonials placeholder, "Start Free Trial" CTA
- Documentation site: Getting Started guide (create project, configure feed water, run simulation), membrane selection guide (brackish vs. seawater, element sizing), scaling prevention best practices, API documentation (for Advanced plan), keyboard shortcuts reference, troubleshooting common errors
- Production deployment checklist: merge to `main` branch → GitHub Actions CI passes → build Docker images → push to ECR → update K8s manifests with new image tags → `kubectl apply -f k8s/` → monitor rollout → verify health checks green → smoke test critical flows → enable monitoring alerts (Slack notifications for errors, PagerDuty for critical failures)
- Launch announcement: post on LinkedIn (target water treatment engineers), Reddit r/ChemicalEngineering, ProductHunt launch, email beta testers, Twitter/X announcement

---

## Critical Files

```
membraneflow/
├── solver-core/
│   ├── src/
│   │   ├── lib.rs
│   │   ├── types.rs
│   │   ├── transport.rs
│   │   ├── osmotic.rs
│   │   ├── cp_modulus.rs
│   │   ├── mass_transfer.rs
│   │   ├── rejection.rs
│   │   ├── pressure_drop.rs
│   │   ├── energy.rs
│   │   ├── ro_solver.rs
│   │   ├── array_config.rs
│   │   ├── system.rs
│   │   └── scaling/
│   │       ├── lsi.rs
│   │       ├── sulfate.rs
│   │       └── silica.rs
│   └── tests/benchmarks.rs
│
├── solver-wasm/src/lib.rs
│
├── membraneflow-api/
│   ├── src/
│   │   ├── main.rs
│   │   ├── config.rs
│   │   ├── state.rs
│   │   ├── error.rs
│   │   ├── auth/
│   │   │   ├── mod.rs
│   │   │   └── oauth.rs
│   │   ├── api/
│   │   │   ├── handlers/
│   │   │   │   ├── auth.rs
│   │   │   │   ├── users.rs
│   │   │   │   ├── projects.rs
│   │   │   │   ├── simulation.rs
│   │   │   │   ├── membranes.rs
│   │   │   │   ├── water_analyses.rs
│   │   │   │   ├── billing.rs
│   │   │   │   └── webhooks/stripe.rs
│   │   │   └── ws/simulation_progress.rs
│   │   ├── middleware/plan_limits.rs
│   │   ├── services/
│   │   │   ├── s3.rs
│   │   │   └── report_generator.rs
│   │   └── workers/simulation_worker.rs
│   └── migrations/001_initial.sql
│
├── optimization-service/
│   ├── main.py
│   └── optimizers/
│       ├── cost_optimizer.py
│       └── pareto.py
│
├── frontend/
│   ├── src/
│   │   ├── App.tsx
│   │   ├── stores/
│   │   │   ├── authStore.ts
│   │   │   ├── projectStore.ts
│   │   │   └── flowDiagramStore.ts
│   │   ├── hooks/
│   │   │   ├── useSimulationProgress.ts
│   │   │   └── useWasmSolver.ts
│   │   ├── pages/
│   │   │   ├── Dashboard.tsx
│   │   │   ├── Editor.tsx
│   │   │   ├── Membranes.tsx
│   │   │   └── Billing.tsx
│   │   └── components/
│   │       ├── FlowDiagramEditor/
│   │       │   ├── Canvas.tsx
│   │       │   ├── ComponentPalette.tsx
│   │       │   ├── MembraneArray.tsx
│   │       │   ├── Pump.tsx
│   │       │   └── Pipe.tsx
│   │       ├── PerformancePlots/
│   │       │   ├── PressureProfile.tsx
│   │       │   ├── FluxProfile.tsx
│   │       │   ├── ConcentrationProfile.tsx
│   │       │   └── ScalingProfile.tsx
│   │       ├── FeedWaterWizard/
│   │       │   ├── WaterQualityForm.tsx
│   │       │   └── IonComposition.tsx
│   │       └── billing/
│   │           ├── PlanCard.tsx
│   │           └── UsageMeter.tsx
│   └── public/wasm/
│
└── k8s/
    ├── api-deployment.yaml
    ├── worker-deployment.yaml
    ├── postgres-statefulset.yaml
    └── redis-deployment.yaml
```

---

## Solver Validation Suite

### Benchmark 1: Single BW30-400 Element (Brackish Water RO)

**Configuration:** DuPont BW30-400, 15 bar, 2000 mg/L NaCl, 25°C, 16 m3/h feed

**Expected:**
- Permeate flow: **6.8 m3/h** (15% recovery per element)
- Permeate TDS: **~40 mg/L** (98% rejection)
- Water flux: **18-20 LMH**

**Tolerance:** Permeate flow < 3%, TDS < 5%

### Benchmark 2: Single TM720D-400 Element (Seawater RO)

**Configuration:** Toray TM720D, 55 bar, 35,000 mg/L TDS, 25°C, 16 m3/h feed

**Expected:**
- Permeate flow: **1.0 m3/h** (6% recovery per element)
- Permeate TDS: **~175 mg/L** (99.5% rejection)
- Water flux: **~26 LMH**

**Tolerance:** Permeate flow < 5%, TDS < 8%

### Benchmark 3: Six-Element Pressure Vessel (ROSA Comparison)

**Configuration:** 6× BW30-400 in series, 15 bar, 2000 mg/L NaCl, 75% recovery

**Expected (ROSA 9.1):**
- Overall recovery: **75.0%**
- Permeate TDS: **38 mg/L**
- Average flux: **19.2 LMH**
- Final element flux: **~12 LMH**

**Tolerance:** Recovery < 2%, permeate TDS < 5%, flux profile < 10%

### Benchmark 4: LSI Calculation (Scaling Validation)

**Feed:** Ca = 80 mg/L, HCO3 = 200 mg/L, pH = 7.5, TDS = 1500 mg/L, 25°C

**Expected at 75% recovery:**
- Concentrate Ca: **320 mg/L**
- LSI: **+1.2 to +1.5** (scaling likely)

**Tolerance:** LSI < 0.2 units

### Benchmark 5: Energy Recovery (PX Pressure Exchanger)

**Configuration:** Seawater RO, 60 bar, 45% recovery, PX efficiency 96%

**Expected:**
- HP pump power (no ERD): **~3.5 kWh/m3**
- Net SEC (with ERD): **~1.6 kWh/m3**
- Energy savings: **~54%**

**Tolerance:** SEC < 5%, energy savings < 3%

---

## Verification

### Integration Testing Checklist

1. Auth flow — Register → login → JWT → authorized calls → refresh → logout
2. Project CRUD — Create → update diagram → save → reload → verify state
3. Flow editing — Place array → configure → connect pipes → set feed water
4. WASM simulation — 20-element system → WASM solver → results → plots display
5. Server simulation — 150-element system → job queue → worker → WebSocket progress → S3 results
6. Performance plots — 100-element result → render profiles → pan/zoom → tooltips
7. Membrane catalog — Search "BW30" → results → select → use in array
8. Water analysis — Save composition → reload in new project → verify ions
9. Scaling analysis — High-recovery system → LSI > 1.0 → antiscalant recommendation
10. Plan limits — Free user → 80-element system → blocked → upgrade prompt
11. Billing — Subscribe to Pro → Stripe checkout → webhook → plan updated
12. PDF report — Simulation → generate PDF → verify tables and P&ID
13. Energy calculation — System with ERD → SEC calculation → benchmark comparison
14. Template usage — "Two-Stage Seawater RO" → create → simulate → expected recovery
15. Error handling — Invalid config → meaningful error → no crash

### Performance Benchmarks

| Operation | Target | Method |
|-----------|--------|--------|
| WASM solver: 20-element system | <200ms | Chrome DevTools |
| WASM solver: 100-element system | <500ms | Browser benchmark |
| Server solver: 500-element system | <5s | Server timing, 4 cores |
| D3 plot render: 100-element profiles | <100ms | Chrome FPS counter |
| API: create simulation | <200ms | p95 latency (Prometheus) |
| WASM bundle load (cached) | <200ms | Service worker |
| WASM bundle load (cold) | <1s | CDN, 500KB gzipped |
| PDF generation: 100 elements | <5s | Headless Chrome |

---

## Deployment Architecture

### Infrastructure

```
┌─────────────────────────────────────────────────┐
│                  CloudFront CDN                   │
│  Frontend SPA, WASM Bundle, Membrane Datasheets   │
└─────────────────────────┬───────────────────────┘
                          │
              ┌───────────┴───────────┐
              │   AWS ALB (HTTPS)     │
              └───────────┬───────────┘
                          │
        ┌─────────────────┼─────────────────┐
        ▼                 ▼                 ▼
   API Server        API Server        API Server
   (Axum Pod ×3)     (Axum Pod)        (Axum Pod)
        │                 │                 │
        └─────────────────┼─────────────────┘
                          │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
   Worker Pod 1     Worker Pod 2     Worker Pod 3
   (Simulation)     (Simulation)     (Simulation)
        │                │                │
        └────────────────┼────────────────┘
                         │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
   PostgreSQL      Redis Cluster      AWS S3
   (RDS/PVC)       (job queue)        (results)
```

### Scaling Strategy

- **API servers**: HPA based on CPU (target 70%), min 3, max 10
- **Workers**: HPA based on Redis queue depth (1 worker per 5 jobs), min 2, max 20
- **PostgreSQL**: RDS with read replicas for membrane catalog queries
- **Redis**: ElastiCache cluster mode for high availability
- **S3**: Lifecycle policy to archive results >90 days to Glacier

---

## Post-MVP Roadmap

### v1.1 — UF/MF Pretreatment Design (Weeks 13–16)

- UF/MF solver with resistance-in-series fouling model
- Hollow fiber and spiral-wound UF modules
- Backwash cycle optimization (frequency, duration)
- Integrated UF → RO treatment train
- SDI and turbidity prediction for RO feed
- UF membrane catalog (50+ elements: Pentair, Toray, Suez)

### v1.2 — Gas Separation Membrane Design (Weeks 17–20)

- Gas separation solver with multi-component permeation (H2/N2, CO2/CH4, O2/N2)
- Polymeric membrane models (polyimide, cellulose acetate)
- Staging optimization (single, two-stage, cascade)
- Applications: H2 recovery, CO2 capture, N2 generation, biogas upgrading
- Compressor sizing and power consumption
- Gas membrane catalog (30+ modules: Air Liquide, Evonik, UBE)

### v1.3 — Advanced Fouling Prediction (Weeks 21–24)

- Organic fouling model (NOM adsorption, SUVA, TOC)
- Colloidal fouling (DLVO theory, particle deposition)
- Biofouling model (biofilm growth kinetics, chlorine decay)
- Membrane aging model (hydrolysis rate, flux decline, salt passage increase)
- Cleaning optimization (chemical selection, frequency, cost-benefit)
- Membrane replacement scheduling based on performance trends

### v1.4 — Multi-Objective Optimization (Weeks 25–28)

- Minimize SEC vs. maximize recovery vs. minimize LCOW
- Decision variables: feed pressure, array staging, membrane model, recovery
- Pareto front visualization with interactive point selection
- Constraint handling: scaling limits, pressure limits, permeate quality
- Sensitivity analysis: vary feed water quality, energy price, membrane cost
- Batch optimization: 10+ scenarios simultaneously

### v1.5 — SCADA Integration (Weeks 29–32)

- API endpoints for real-time plant data import (flow, pressure, TDS)
- Model calibration: adjust A/B coefficients to match actual performance
- Membrane fouling rate estimation from operational trends
- Performance deviation alerts (actual vs. predicted)
- Fleet management: multi-plant membrane performance benchmarking
- Membrane autopsy data integration for fouling model refinement
