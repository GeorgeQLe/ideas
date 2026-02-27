# 83. RailTrack — Railway Engineering Design Platform

## Implementation Plan

**MVP Scope:** Browser-based track geometry designer with horizontal/vertical alignment editor rendered via Mapbox GL + SVG overlay for precise curve placement, automatic clothoid transition curve calculation per EN 13803 and AREMA standards, cant/superelevation designer with equilibrium cant and cant deficiency validation (≤150mm), speed profile generator from geometry constraints (curve radius, cant, gradient limits), basic fixed-block signaling layout tool with automatic block section placement given minimum headway requirements, single-train run simulator solving equation of motion (Davis resistance + tractive effort + braking) compiled to WebAssembly for real-time feedback during design, time-distance diagram visualization (Bildfahrplan) rendered via D3.js with pan/zoom and speed annotations, braking distance calculator per EN 14531 accounting for gradient/speed/adhesion, track geometry report generator (PDF) with alignment tables, cant diagrams, speed profiles, and signaling layout, PostgreSQL storage for projects with PostGIS for spatial queries on alignment, S3 storage for design files and reports, Stripe billing with Free (10km single-line) / Pro ($169/mo) / Advanced ($379/user/mo) tiers.

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| Track Geometry Solver | Rust (native + WASM) | Clothoid curve math, cant transitions, clearance checking via `geo` crate |
| WASM Compilation | `wasm-pack` + `wasm-bindgen` | Client-side geometry validation and train run simulation |
| Signaling Engine | Rust (server-side) | Fixed-block layout optimization, headway calculation |
| Vehicle Dynamics | Rust (native + WASM) | Davis equation, braking distance, traction limits |
| ML Services | Python 3.12 (FastAPI) | Delay prediction, energy optimization, alignment optimization |
| Database | PostgreSQL 16 + PostGIS | Projects, users, track geometry, signaling layouts |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | Design files (.railtrack format), PDF reports, RailML export |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| Map Visualization | Mapbox GL JS 3.x | Geographic track alignment on base map, elevation contours |
| Diagram Rendering | D3.js + SVG | Time-distance diagrams, speed profiles, cant diagrams, vertical profiles |
| Track Editor | Custom SVG overlay | Alignment elements on Mapbox canvas, snap-to-grid, tangent/curve tools |
| Real-time | WebSocket (Axum) | Collaborative editing, simulation progress |
| Job Queue | Redis 7 + Tokio tasks | Server-side simulation jobs, report generation |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| PDF Generation | Python (ReportLab) | Track geometry reports, signaling layout drawings |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, error tracking |
| CI/CD | GitHub Actions | WASM build pipeline, Rust tests, Docker image push |

### Key Architecture Decisions

1. **Hybrid client/server geometry solver with WASM for interactive design**: Track alignment calculations (clothoid transitions, cant transitions, clearance checks) run in the browser via WASM for real-time feedback as the user drags alignment elements on the map. Server-side Rust handles complex optimization (earthwork minimization, multi-objective route selection) and final validation before committing a design. This gives instant visual feedback for simple edits (99% of interactions) while enabling powerful batch optimization.

2. **Mapbox GL JS for geographic context with custom SVG overlay for precision editing**: Railway alignments are inherently geographic (tied to real-world coordinates, terrain elevation, existing infrastructure), so a slippy map with terrain/satellite layers is essential for initial route placement. However, Mapbox's vector rendering is not precise enough for engineering-grade curve tangent points and station positions, so we render a transparent SVG overlay synchronized with the map viewport for sub-meter-accurate editing. PostGIS handles coordinate transformations and spatial queries.

3. **Clothoid (Euler spiral) transition curves as the core geometric primitive**: Unlike highway design (which often uses circular curves with simple linear transitions), railway track geometry requires clothoid spirals where curvature increases linearly with distance to provide smooth lateral acceleration as trains enter/exit curves. The clothoid differential equations do not have a closed-form solution, so we use Fresnel integral approximations (accurate to 1mm over 1km) for real-time rendering and exact numerical integration for final design validation.

4. **Client-side train run simulation in WASM for instant speed profile updates**: When a user modifies the track geometry (curve radius, gradient, cant), the maximum safe speed changes at that location due to curve resistance, lateral acceleration limits, and tractive effort limits. Running the train dynamics solver (Davis equation + equation of motion integrator) in WASM allows the speed profile overlay to update in <100ms, enabling iterative design. Server-side simulation handles multi-train timetable conflicts and stochastic delay propagation.

5. **Fixed-block signaling as MVP with extensible architecture for moving-block**: Fixed-block signaling (track divided into discrete sections, only one train per block) is the foundation of railway capacity analysis and is used by 90%+ of global rail networks. The MVP implements automatic block section placement given a target headway (e.g., 3 minutes) and track layout, calculating signal positions to meet sighting distance requirements per EN standards. The architecture is designed to support moving-block (ETCS Level 3, CBTC) in post-MVP via a pluggable signaling strategy interface.

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

-- Organizations
CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    slug TEXT UNIQUE NOT NULL,
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    plan TEXT NOT NULL DEFAULT 'free',
    stripe_customer_id TEXT,
    settings JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
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

-- Projects
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    alignment_data JSONB NOT NULL DEFAULT '{}',
    settings JSONB DEFAULT '{}',
    is_public BOOLEAN NOT NULL DEFAULT false,
    forked_from UUID REFERENCES projects(id) ON DELETE SET NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);

-- Track Alignment Elements
CREATE TABLE alignment_elements (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    element_type TEXT NOT NULL,
    start_station REAL NOT NULL,
    end_station REAL NOT NULL,
    geometry GEOMETRY(LINESTRING, 4326) NOT NULL,
    parameters JSONB NOT NULL DEFAULT '{}',
    sequence_order INTEGER NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX alignment_geom_idx ON alignment_elements USING GIST(geometry);

-- Cant Profile
CREATE TABLE cant_profile (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    start_station REAL NOT NULL,
    end_station REAL NOT NULL,
    cant_mm REAL NOT NULL,
    transition_type TEXT,
    parameters JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Signaling Layouts
CREATE TABLE signaling_layouts (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    signaling_type TEXT NOT NULL DEFAULT 'fixed_block',
    block_sections JSONB NOT NULL DEFAULT '[]',
    signals JSONB NOT NULL DEFAULT '[]',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Train Runs
CREATE TABLE train_runs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    rolling_stock_id UUID REFERENCES rolling_stock(id),
    status TEXT NOT NULL DEFAULT 'pending',
    parameters JSONB NOT NULL DEFAULT '{}',
    results_url TEXT,
    results_summary JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Rolling Stock
CREATE TABLE rolling_stock (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    manufacturer TEXT,
    stock_type TEXT NOT NULL,
    mass_tonnes REAL NOT NULL,
    max_speed_kmh REAL NOT NULL,
    power_kw REAL,
    davis_params JSONB NOT NULL,
    braking_params JSONB NOT NULL,
    is_builtin BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Design Reports
CREATE TABLE design_reports (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    report_type TEXT NOT NULL,
    s3_url TEXT NOT NULL,
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
    pub plan: String,
    pub stripe_customer_id: Option<String>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Project {
    pub id: Uuid,
    pub owner_id: Uuid,
    pub name: String,
    pub alignment_data: serde_json::Value,
    pub settings: serde_json::Value,
    pub is_public: bool,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct AlignmentElement {
    pub id: Uuid,
    pub project_id: Uuid,
    pub element_type: String,
    pub start_station: f32,
    pub end_station: f32,
    pub geometry: serde_json::Value,
    pub parameters: serde_json::Value,
    pub sequence_order: i32,
}

#[derive(Debug, FromRow, Serialize)]
pub struct RollingStock {
    pub id: Uuid,
    pub name: String,
    pub stock_type: String,
    pub mass_tonnes: f32,
    pub max_speed_kmh: f32,
    pub power_kw: Option<f32>,
    pub davis_params: serde_json::Value,
    pub braking_params: serde_json::Value,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct DavisParams {
    pub a_n: f64,
    pub b_n_per_kmh: f64,
    pub c_n_per_kmh2: f64,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct BrakingParams {
    pub service_decel_ms2: f64,
    pub emergency_decel_ms2: f64,
    pub brake_buildup_sec: f64,
}
```

---

## Solver Architecture Deep-Dive

### Governing Equations and Track Geometry Mathematics

**Clothoid (Euler spiral) parametric equations:**

For a clothoid with parameter `A` (clothoid constant in meters), the coordinates and tangent angle as a function of distance `l` along the curve are:

```
x(l) = l · C(u)   where u = l / (A√π)
y(l) = l · S(u)
θ(l) = l² / (2A²)

C(u) = Fresnel cosine integral = ∫₀ᵘ cos(πt²/2) dt
S(u) = Fresnel sine integral   = ∫₀ᵘ sin(πt²/2) dt
```

The Fresnel integrals are approximated using Boehm polynomials (accurate to 1mm over 1km):

```rust
fn fresnel_c(u: f64) -> f64 {
    let u2 = u * u;
    let u4 = u2 * u2;
    u * (1.0 - u4 / 10.0 + u8 / 216.0 - u12 / 9360.0)
}
```

**Curvature:** `κ(l) = l / A²`

**Cant (superelevation) calculation:**

Equilibrium cant for speed `V` (m/s) on curve radius `R` (m) with track gauge `g` (m):

```
Eₑ = (V² · g) / (R · 9.81)   [meters]
```

Cant deficiency: `D = Eₑ - E_actual` (limited to 150mm passenger, 50mm freight per EN 13803)

**Cant transition profile (sinusoidal):**

```
cant(l) = cant_start + (cant_end - cant_start) · (l/L - sin(2πl/L)/(2π))
```

### Train Dynamics Solver

**Equation of motion:**

```
m · dv/dt = F_traction - F_resistance - F_gradient - F_curve
```

Where:
- **F_traction** = min(P/v, F_max, μ·m·g) — power-limited or adhesion-limited
- **F_resistance** = Davis equation: `A + B·v + C·v²`
- **F_gradient** = `m · g · sin(θ)`
- **F_curve** = `400·m / R` N (empirical, AREMA)

**Davis equation parameters (per tonne, modern EMU):**
```
A = 650 N     (bearing friction, track irregularity)
B = 13 N/(km/h)  (flange friction)
C = 0.045 N/(km/h)²  (aerodynamic drag)
```

**Braking distance (EN 14531):**

```
s_brake = v₀² / (2·(a_brake + g·sin(θ)))

where a_brake = min(a_service, μ·g)
Safety margin: 1.3× for service brake
```

**Maximum speed on curve:**

```
V_max = √(R · (a_lat_max + g·E/gauge))

where a_lat_max = 1.0 m/s² (passenger comfort limit per EN)
```

---

## Architecture Deep-Dives

### 1. Track Alignment API Handler (Rust/Axum)

```rust
// src/api/handlers/alignment.rs

use axum::{extract::{Path, State, Json}, http::StatusCode, response::IntoResponse};
use uuid::Uuid;
use crate::{db::models::*, geometry::clothoid, state::AppState, auth::Claims, error::ApiError};

#[derive(serde::Deserialize)]
pub struct UpdateAlignmentRequest {
    pub elements: Vec<AlignmentElementInput>,
    pub cant_profile: Vec<CantInput>,
}

#[derive(serde::Deserialize)]
pub struct AlignmentElementInput {
    pub element_type: String,
    pub start_station: f64,
    pub parameters: serde_json::Value,
}

pub async fn update_alignment(
    State(state): State<AppState>,
    claims: Claims,
    Path(project_id): Path<Uuid>,
    Json(req): Json<UpdateAlignmentRequest>,
) -> Result<impl IntoResponse, ApiError> {
    // Verify ownership
    let project = sqlx::query_as!(Project,
        "SELECT * FROM projects WHERE id = $1 AND owner_id = $2",
        project_id, claims.user_id
    ).fetch_optional(&state.db).await?
    .ok_or(ApiError::NotFound("Project not found"))?;

    // Validate and compute geometry
    let mut computed_elements = Vec::new();
    let mut current_station = 0.0;

    for elem_input in req.elements {
        let elem = match elem_input.element_type.as_str() {
            "tangent" => {
                let length = elem_input.parameters["length"].as_f64().unwrap();
                let azimuth = elem_input.parameters["azimuth"].as_f64().unwrap();
                let start_coords = elem_input.parameters["start_coords"].clone();

                let geom = clothoid::generate_tangent(
                    start_coords["lat"].as_f64().unwrap(),
                    start_coords["lon"].as_f64().unwrap(),
                    azimuth, length
                );

                AlignmentElement {
                    id: Uuid::new_v4(),
                    project_id,
                    element_type: "tangent".to_string(),
                    start_station: current_station as f32,
                    end_station: (current_station + length) as f32,
                    geometry: serde_json::to_value(&geom)?,
                    parameters: elem_input.parameters,
                    sequence_order: computed_elements.len() as i32,
                }
            }
            "circular_curve" => {
                let radius = elem_input.parameters["radius"].as_f64().unwrap();
                let delta_angle = elem_input.parameters["delta_angle"].as_f64().unwrap();
                let clothoid_length = clothoid::compute_transition_length(radius, 120.0);

                let curve = clothoid::generate_curve_with_clothoids(
                    &computed_elements.last().unwrap().geometry,
                    radius, delta_angle, clothoid_length
                )?;

                let total_length = clothoid_length * 2.0 + radius * delta_angle.to_radians();

                AlignmentElement {
                    id: Uuid::new_v4(),
                    project_id,
                    element_type: "circular_curve".to_string(),
                    start_station: current_station as f32,
                    end_station: (current_station + total_length) as f32,
                    geometry: serde_json::to_value(&curve)?,
                    parameters: serde_json::json!({"radius": radius, "delta_angle": delta_angle}),
                    sequence_order: computed_elements.len() as i32,
                }
            }
            _ => continue,
        };

        current_station = elem.end_station as f64;
        computed_elements.push(elem);
    }

    // Delete old and insert new elements
    sqlx::query!("DELETE FROM alignment_elements WHERE project_id = $1", project_id)
        .execute(&state.db).await?;

    for elem in computed_elements {
        sqlx::query!(
            r#"INSERT INTO alignment_elements
                (id, project_id, element_type, start_station, end_station,
                 geometry, parameters, sequence_order)
            VALUES ($1, $2, $3, $4, $5, ST_GeomFromGeoJSON($6), $7, $8)"#,
            elem.id, elem.project_id, elem.element_type, elem.start_station,
            elem.end_station, elem.geometry.to_string(), elem.parameters, elem.sequence_order
        ).execute(&state.db).await?;
    }

    Ok((StatusCode::OK, Json(serde_json::json!({"status": "ok"}))))
}
```

### 2. Clothoid Geometry Solver (Rust — WASM compatible)

```rust
// geometry-core/src/clothoid.rs

use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Coords {
    pub lat: f64,
    pub lon: f64,
}

fn fresnel_c(u: f64) -> f64 {
    if u.abs() < 1e-10 { return 0.0; }
    let u2 = u * u;
    let u4 = u2 * u2;
    let u8 = u4 * u4;
    u * (1.0 - u4 / 10.0 + u8 / 216.0 - u8 * u4 / 9360.0)
}

fn fresnel_s(u: f64) -> f64 {
    if u.abs() < 1e-10 { return 0.0; }
    let u2 = u * u;
    let u3 = u2 * u;
    u3 * (1.0 / 3.0 - u2 * u2 / 42.0 + u2 * u2 * u3 / 1320.0)
}

pub fn generate_clothoid(
    start_lat: f64, start_lon: f64, start_azimuth: f64,
    clothoid_constant: f64, length: f64, num_points: usize
) -> Vec<Coords> {
    let mut points = Vec::with_capacity(num_points);
    let dl = length / (num_points - 1) as f64;

    for i in 0..num_points {
        let l = i as f64 * dl;
        let u = l / (clothoid_constant * std::f64::consts::PI.sqrt());

        let x = l * fresnel_c(u);
        let y = l * fresnel_s(u);
        let theta = start_azimuth + l * l / (2.0 * clothoid_constant * clothoid_constant);

        // Transform to geographic coordinates
        let lat = start_lat + (x * theta.cos() - y * theta.sin()) / 111320.0;
        let lon = start_lon + (x * theta.sin() + y * theta.cos()) /
                  (111320.0 * start_lat.to_radians().cos());

        points.push(Coords { lat, lon });
    }
    points
}

pub fn compute_transition_length(radius: f64, speed_kmh: f64) -> f64 {
    let cant_rate = 60.0; // mm/s
    let v_ms = speed_kmh / 3.6;
    let l_min = (v_ms * v_ms * v_ms) / (cant_rate * radius);
    (l_min / 5.0).ceil() * 5.0
}

pub fn generate_curve_with_clothoids(
    entry_tangent: &serde_json::Value, radius: f64,
    delta_angle_deg: f64, clothoid_length: f64
) -> Result<Vec<Coords>, String> {
    let clothoid_constant = (radius * clothoid_length).sqrt();
    let entry = generate_clothoid(0.0, 0.0, 0.0, clothoid_constant, clothoid_length, 20);
    let circular_angle = delta_angle_deg.to_radians() - 2.0 * clothoid_length / radius;
    let mut points = entry.clone();

    for i in 1..30 {
        let angle = circular_angle * i as f64 / 29.0;
        let x = radius * angle.sin();
        let y = radius * (1.0 - angle.cos());
        let last = entry.last().unwrap();
        points.push(Coords { lat: last.lat + y / 111320.0, lon: last.lon + x / 111320.0 });
    }

    let exit = generate_clothoid(
        points.last().unwrap().lat, points.last().unwrap().lon,
        delta_angle_deg.to_radians(), clothoid_constant, clothoid_length, 20
    );
    points.extend(exit);
    Ok(points)
}
```

### 3. Train Dynamics Simulator (Rust — WASM)

```rust
// solver-wasm/src/train_dynamics.rs

use wasm_bindgen::prelude::*;
use serde::{Deserialize, Serialize};

#[derive(Debug, Deserialize)]
pub struct RollingStockParams {
    pub mass_tonnes: f64,
    pub max_speed_kmh: f64,
    pub power_kw: f64,
    pub davis_a: f64,
    pub davis_b: f64,
    pub davis_c: f64,
    pub max_traction_kn: f64,
    pub service_decel_ms2: f64,
}

#[derive(Debug, Deserialize)]
pub struct AlignmentPoint {
    pub station_m: f64,
    pub gradient_pct: f64,
    pub curve_radius_m: Option<f64>,
    pub cant_mm: f64,
    pub speed_limit_kmh: Option<f64>,
}

#[derive(Debug, Serialize)]
pub struct SpeedProfilePoint {
    pub station_m: f64,
    pub speed_kmh: f64,
    pub time_sec: f64,
    pub limiting_factor: String,
}

#[wasm_bindgen]
pub struct TrainSimulator {
    stock: RollingStockParams,
    alignment: Vec<AlignmentPoint>,
}

#[wasm_bindgen]
impl TrainSimulator {
    #[wasm_bindgen(constructor)]
    pub fn new(stock_json: &str, alignment_json: &str) -> Result<TrainSimulator, JsValue> {
        let stock = serde_json::from_str(stock_json)
            .map_err(|e| JsValue::from_str(&e.to_string()))?;
        let alignment = serde_json::from_str(alignment_json)
            .map_err(|e| JsValue::from_str(&e.to_string()))?;
        Ok(TrainSimulator { stock, alignment })
    }

    #[wasm_bindgen]
    pub fn run_simulation(&self, timestep_sec: f64) -> Result<String, JsValue> {
        let profile = self.compute_speed_profile(timestep_sec);
        serde_json::to_string(&profile).map_err(|e| JsValue::from_str(&e.to_string()))
    }

    #[wasm_bindgen]
    pub fn compute_braking_distance(&self, initial_kmh: f64, target_kmh: f64, gradient: f64) -> f64 {
        let v0 = initial_kmh / 3.6;
        let vf = target_kmh / 3.6;
        let g = 9.81;
        let a_brake = self.stock.service_decel_ms2.min(0.25 * g);
        let a_total = a_brake + g * (gradient / 100.0).atan().sin();
        if a_total <= 0.0 { return f64::INFINITY; }
        (v0 * v0 - vf * vf) / (2.0 * a_total)
    }
}

impl TrainSimulator {
    fn compute_speed_profile(&self, dt: f64) -> Vec<SpeedProfilePoint> {
        let mut profile = Vec::new();
        let mut v = 0.0;
        let mut s = 0.0;
        let mut t = 0.0;
        let total_distance = self.alignment.last().unwrap().station_m;

        while s < total_distance {
            let (gradient, radius, cant, speed_limit) = self.interpolate_alignment(s);
            let v_max_curve = if let Some(r) = radius {
                self.compute_max_speed_curve(r, cant)
            } else {
                self.stock.max_speed_kmh / 3.6
            };
            let v_max = v_max_curve.min(speed_limit.unwrap_or(f64::INFINITY) / 3.6);

            let f_traction = self.compute_traction_force(v);
            let f_resistance = self.compute_davis_resistance(v);
            let f_gradient = self.stock.mass_tonnes * 1000.0 * 9.81 * (gradient / 100.0).atan().sin();
            let f_curve = radius.map_or(0.0, |r| 400.0 * self.stock.mass_tonnes / r);

            let f_net = f_traction - f_resistance - f_gradient - f_curve;
            let a = f_net / (self.stock.mass_tonnes * 1000.0);
            let a_actual = if v >= v_max { a.min(0.0) } else { a };

            v = (v + a_actual * dt).max(0.0);
            s += v * dt;
            t += dt;

            profile.push(SpeedProfilePoint {
                station_m: s,
                speed_kmh: v * 3.6,
                time_sec: t,
                limiting_factor: if v >= v_max {
                    if radius.is_some() { "curve".to_string() } else { "speed_limit".to_string() }
                } else { "traction".to_string() },
            });
        }
        profile
    }

    fn compute_davis_resistance(&self, v_ms: f64) -> f64 {
        let v_kmh = v_ms * 3.6;
        self.stock.davis_a + self.stock.davis_b * v_kmh + self.stock.davis_c * v_kmh * v_kmh
    }

    fn compute_traction_force(&self, v_ms: f64) -> f64 {
        if v_ms < 0.1 { return self.stock.max_traction_kn * 1000.0; }
        let f_power = (self.stock.power_kw * 1000.0) / v_ms;
        f_power.min(self.stock.max_traction_kn * 1000.0)
    }

    fn compute_max_speed_curve(&self, radius: f64, cant_mm: f64) -> f64 {
        let g = 9.81;
        let a_lat_max = 1.0;
        (radius * (a_lat_max + g * cant_mm / 1435.0)).sqrt()
    }

    fn interpolate_alignment(&self, station: f64) -> (f64, Option<f64>, f64, Option<f64>) {
        let mut i = 0;
        while i < self.alignment.len() - 1 && self.alignment[i + 1].station_m < station {
            i += 1;
        }
        let p = &self.alignment[i.min(self.alignment.len() - 1)];
        (p.gradient_pct, p.curve_radius_m, p.cant_mm, p.speed_limit_kmh)
    }
}
```

### 4. Signaling Layout Optimizer (Rust)

```rust
// src/solvers/signaling.rs

use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize)]
pub struct BlockSection {
    pub id: String,
    pub start_m: f64,
    pub end_m: f64,
    pub length_m: f64,
}

#[derive(Debug, Serialize)]
pub struct Signal {
    pub id: String,
    pub station_m: f64,
    pub signal_type: String,
}

pub fn generate_fixed_block_layout(
    target_headway_sec: f64,
    train_speed_kmh: f64,
    braking_distance_m: f64,
    total_length_m: f64
) -> Vec<BlockSection> {
    let safety_margin = 50.0;
    let min_block_length = braking_distance_m + safety_margin;
    let v_ms = train_speed_kmh / 3.6;
    let headway_block_length = v_ms * target_headway_sec;
    let block_length = headway_block_length.max(min_block_length);
    let block_length = (block_length / 100.0).ceil() * 100.0;

    let num_blocks = (total_length_m / block_length).ceil() as usize;
    let mut blocks = Vec::new();

    for i in 0..num_blocks {
        let start = i as f64 * block_length;
        let end = ((i + 1) as f64 * block_length).min(total_length_m);
        blocks.push(BlockSection {
            id: format!("B{}", i + 1),
            start_m: start,
            end_m: end,
            length_m: end - start,
        });
    }
    blocks
}

pub fn compute_minimum_headway(block_length_m: f64, speed_kmh: f64, signal_cycle_sec: f64) -> f64 {
    let v_ms = speed_kmh / 3.6;
    block_length_m / v_ms + signal_cycle_sec
}
```

### 5. Time-Distance Diagram Component (React + D3)

```typescript
// frontend/src/components/TimeDistanceDiagram.tsx

import { useRef, useEffect } from 'react';
import * as d3 from 'd3';
import { useSimulationStore } from '../stores/simulationStore';

interface TrainPath {
  id: string;
  name: string;
  points: { time_sec: number; station_m: number; speed_kmh: number }[];
  color: string;
}

export function TimeDistanceDiagram() {
  const svgRef = useRef<SVGSVGElement>(null);
  const { trainPaths, conflicts } = useSimulationStore();

  useEffect(() => {
    if (!svgRef.current || trainPaths.length === 0) return;

    const svg = d3.select(svgRef.current);
    const width = 1200;
    const height = 800;
    const margin = { top: 40, right: 60, bottom: 60, left: 80 };

    svg.selectAll('*').remove();

    const xScale = d3.scaleLinear()
      .domain([0, d3.max(trainPaths.flatMap(p => p.points.map(pt => pt.time_sec))) || 3600])
      .range([margin.left, width - margin.right]);

    const yScale = d3.scaleLinear()
      .domain([0, d3.max(trainPaths.flatMap(p => p.points.map(pt => pt.station_m))) || 10000])
      .range([height - margin.bottom, margin.top]);

    // Grid lines
    svg.append('g')
      .attr('class', 'grid')
      .selectAll('line')
      .data(d3.range(0, 3600, 300)) // 5-minute intervals
      .join('line')
      .attr('x1', d => xScale(d))
      .attr('x2', d => xScale(d))
      .attr('y1', margin.top)
      .attr('y2', height - margin.bottom)
      .attr('stroke', '#444')
      .attr('stroke-dasharray', '2,2');

    // Axes
    svg.append('g')
      .attr('transform', `translate(0,${height - margin.bottom})`)
      .call(d3.axisBottom(xScale).ticks(12).tickFormat(d => `${Math.floor(d / 60)}:${(d % 60).toString().padStart(2, '0')}`));

    svg.append('g')
      .attr('transform', `translate(${margin.left},0)`)
      .call(d3.axisLeft(yScale).ticks(10).tickFormat(d => `${(d / 1000).toFixed(1)}km`));

    // Train paths
    const line = d3.line<{ time_sec: number; station_m: number }>()
      .x(d => xScale(d.time_sec))
      .y(d => yScale(d.station_m))
      .curve(d3.curveMonotoneX);

    trainPaths.forEach(path => {
      svg.append('path')
        .datum(path.points)
        .attr('fill', 'none')
        .attr('stroke', path.color)
        .attr('stroke-width', 2.5)
        .attr('d', line);

      // Train label at start
      const start = path.points[0];
      svg.append('text')
        .attr('x', xScale(start.time_sec))
        .attr('y', yScale(start.station_m) - 10)
        .attr('font-size', 12)
        .attr('fill', path.color)
        .text(path.name);
    });

    // Conflict highlights
    conflicts.forEach(conflict => {
      svg.append('rect')
        .attr('x', xScale(conflict.time_start_sec))
        .attr('y', yScale(conflict.station_end_m))
        .attr('width', xScale(conflict.time_end_sec) - xScale(conflict.time_start_sec))
        .attr('height', yScale(conflict.station_start_m) - yScale(conflict.station_end_m))
        .attr('fill', 'red')
        .attr('opacity', 0.2)
        .attr('stroke', 'red')
        .attr('stroke-width', 2);
    });

  }, [trainPaths, conflicts]);

  return (
    <div className="time-distance-diagram">
      <h2>Time-Distance Diagram</h2>
      <svg ref={svgRef} width={1200} height={800} />
    </div>
  );
}
```

### 6. Speed Profile Visualizer (React + D3 + WASM)

```typescript
// frontend/src/components/SpeedProfile.tsx

import { useRef, useEffect, useState } from 'react';
import * as d3 from 'd3';
import { useAlignmentStore } from '../stores/alignmentStore';
import { TrainSimulator } from '../../wasm/pkg';

export function SpeedProfile() {
  const svgRef = useRef<SVGSVGElement>(null);
  const { alignment, rollingStock } = useAlignmentStore();
  const [profile, setProfile] = useState<any[]>([]);

  useEffect(() => {
    if (!alignment || !rollingStock) return;

    const simulator = new TrainSimulator(
      JSON.stringify(rollingStock),
      JSON.stringify(alignment)
    );
    const result = JSON.parse(simulator.run_simulation(0.5));
    setProfile(result);
  }, [alignment, rollingStock]);

  useEffect(() => {
    if (!svgRef.current || profile.length === 0) return;

    const svg = d3.select(svgRef.current);
    const width = 1000;
    const height = 400;
    const margin = { top: 20, right: 40, bottom: 40, left: 60 };

    svg.selectAll('*').remove();

    const xScale = d3.scaleLinear()
      .domain([0, d3.max(profile, d => d.station_m) || 10000])
      .range([margin.left, width - margin.right]);

    const yScale = d3.scaleLinear()
      .domain([0, d3.max(profile, d => d.speed_kmh) || 160])
      .range([height - margin.bottom, margin.top]);

    // Axes
    svg.append('g')
      .attr('transform', `translate(0,${height - margin.bottom})`)
      .call(d3.axisBottom(xScale).ticks(10).tickFormat(d => `${(d / 1000).toFixed(1)}km`));

    svg.append('g')
      .attr('transform', `translate(${margin.left},0)`)
      .call(d3.axisLeft(yScale).ticks(8).tickFormat(d => `${d} km/h`));

    // Speed profile line
    const line = d3.line<any>()
      .x(d => xScale(d.station_m))
      .y(d => yScale(d.speed_kmh))
      .curve(d3.curveMonotoneX);

    svg.append('path')
      .datum(profile)
      .attr('fill', 'none')
      .attr('stroke', '#2563eb')
      .attr('stroke-width', 2)
      .attr('d', line);

    // Color-code limiting factors
    const colorMap = {
      traction: '#22c55e',
      curve: '#eab308',
      gradient: '#f97316',
      speed_limit: '#ef4444',
    };

    profile.forEach((pt, i) => {
      if (i === 0) return;
      const prev = profile[i - 1];
      svg.append('line')
        .attr('x1', xScale(prev.station_m))
        .attr('y1', height - margin.bottom + 5)
        .attr('x2', xScale(pt.station_m))
        .attr('y2', height - margin.bottom + 5)
        .attr('stroke', colorMap[pt.limiting_factor] || '#666')
        .attr('stroke-width', 4);
    });

  }, [profile]);

  return (
    <div className="speed-profile">
      <h3>Speed Profile</h3>
      <svg ref={svgRef} width={1000} height={400} />
      <div className="legend">
        <span><span className="dot green"></span> Traction Limited</span>
        <span><span className="dot yellow"></span> Curve Limited</span>
        <span><span className="dot orange"></span> Gradient Limited</span>
        <span><span className="dot red"></span> Speed Limit</span>
      </div>
    </div>
  );
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–4)

**Day 1: Project initialization**
- `cargo init railtrack-api && cd railtrack-api`
- `cargo add axum tokio serde sqlx uuid chrono tower-http jsonwebtoken bcrypt geo postgis aws-sdk-s3 redis`
- `src/main.rs`, `src/config.rs`, `src/state.rs`, `src/error.rs`
- `docker-compose.yml` with PostgreSQL+PostGIS, Redis, MinIO

**Day 2: Database schema**
- `migrations/001_initial.sql` — All tables with PostGIS geometry columns
- `src/db/models.rs` — SQLx structs
- Seed script for built-in rolling stock

**Day 3: Authentication**
- `src/auth/mod.rs` — JWT middleware
- `src/api/handlers/auth.rs` — Register, login, OAuth
- Integration tests

**Day 4: Project CRUD**
- `src/api/handlers/projects.rs` — Create, list, get, update, delete
- `src/api/handlers/orgs.rs` — Organization management

### Phase 2 — Geometry Solver (Days 5–11)

**Day 5: Clothoid mathematics**
- `geometry-core/src/clothoid.rs` — Fresnel integrals, curve generation
- Unit tests vs. reference values

**Day 6: Alignment generators**
- Tangent, circular curve, vertical curve generators
- PostGIS integration

**Day 7: Cant calculations**
- Equilibrium cant, cant deficiency, transition profiles
- EN 13803 validation

**Day 8: Clearance checking**
- Kinematic gauge, EN 15273 standards
- Tunnel/bridge clearance verification

**Day 9: Validation logic**
- Minimum radius, maximum gradient constraints
- Error reporting

**Day 10: PostGIS integration**
- Alignment API handlers with spatial queries
- Geometry storage/retrieval tests

**Day 11: WASM geometry build**
- `geometry-wasm/` with wasm-pack
- Frontend integration

### Phase 3 — Train Dynamics (Days 12–16)

**Day 12: Davis resistance and traction**
- `solver-core/src/resistance.rs`
- `solver-core/src/traction.rs`

**Day 13: Equation of motion**
- Euler integrator for train dynamics
- Speed limit enforcement

**Day 14: Braking distance**
- EN 14531 formula with safety margins
- Gradient compensation

**Day 15: WASM train simulator**
- `solver-wasm/src/train_dynamics.rs`
- Performance: 10km run in <50ms

**Day 16: Server-side worker**
- Background simulation worker with Redis
- S3 result storage

### Phase 4 — Signaling (Days 17–21)

**Day 17: Fixed-block model**
- Block section placement
- Headway calculation

**Day 18: Sighting distance**
- Signal visibility validation
- EN standards compliance

**Day 19: Signaling API**
- Generate/get layout endpoints

**Day 20: Capacity analysis**
- Trains per hour calculation
- Conflict detection

**Day 21: Validation suite**
- Block length vs. braking distance
- Signal aspect progression

### Phase 5 — Frontend (Days 22–28)

**Day 22: React + Mapbox scaffold**
- `npx create-vite@latest railtrack-frontend --template react-ts`
- Install: `mapbox-gl`, `d3`, `zustand`, `react-router-dom`, `axios`
- `src/components/MapView.tsx` — Mapbox GL map with terrain/satellite
- Map initialization at project location or default (lat/lon from settings)
- Controls: pan, zoom, pitch, bearing, geolocate
- Style switcher: streets, satellite, terrain

**Day 23: SVG track overlay**
- `src/components/AlignmentOverlay.tsx` — SVG layer over Mapbox canvas
- Convert lat/lon to screen coords via `map.project()`
- Render alignment elements as SVG `<path>` elements
- Sync SVG viewBox with map on move/zoom/resize
- Performance: only render elements in viewport (spatial culling)
- Tests: alignment stays aligned during pan/zoom

**Day 24: Track editor interactions**
- Click map → add tangent point (lat/lon captured)
- Drag tangent point → call WASM clothoid generator (debounced 100ms)
- Select circular curve tool → click two tangents → modal for radius input
- Snap-to-grid (10m, 50m, 100m) with toggle button
- Undo/redo stack (max 10 operations, stored in Zustand)
- Keyboard shortcuts: Delete key removes selected element, Esc deselects
- Save alignment → POST /api/projects/:id/alignment

**Day 25: Cant profile editor**
- `src/components/CantEditor.tsx` — Table + D3 cant diagram
- Table columns: Start Station, End Station, Cant (mm), Transition Type, Actions
- Add row → input station range, cant value, select transition (linear/sinusoidal/cubic)
- D3 diagram: station (x-axis) vs. cant (y-axis), line chart
- Validation warnings: cant deficiency >150mm highlighted in red
- Tooltip on hover: show exact cant value and cant deficiency at that point
- Export cant profile as CSV

**Day 26: Speed profile viz**
- `src/components/SpeedProfile.tsx` — D3 line chart
- Trigger WASM `TrainSimulator.run_simulation()` when alignment or rolling stock changes
- Parse JSON result, render speed vs. station
- Color-code limiting factors: green=traction, yellow=curve, orange=gradient, red=speed limit
- Click point on chart → highlight corresponding location on map (sync with MapView)
- Display stats: max speed, average speed, total travel time, energy estimate
- Export as CSV: station, speed, time, limiting factor

**Day 27: Vertical profile**
- `src/components/VerticalProfile.tsx` — D3 area chart (station vs. elevation)
- Import terrain elevation from Mapbox Terrain-RGB raster tiles
- Sample elevation every 10m along alignment, store in alignment data
- Manually adjust vertical curves: click to add vertex, drag to adjust elevation
- Gradient percentage display along alignment (color-coded: green <2%, yellow 2-3%, red >3%)
- Earthwork volume estimation: cut (above design) vs. fill (below design), assume cut-fill ratio 1.2:1
- Export vertical profile as CSV

**Day 28: Save/load**
- Auto-save alignment to backend every 30 seconds (debounced, only if dirty flag set)
- Load alignment on project open: GET /api/projects/:id/alignment
- Conflict detection: if another user edited (compare updated_at timestamp), show warning dialog
- Last-write-wins merge strategy with option to view other user's changes
- Version history: list recent saves with timestamps, allow restore
- Tests: concurrent edits, auto-save behavior, network errors

### Phase 6 — Time-Distance Diagrams (Days 29–32)

**Day 29: D3 diagram scaffold**
- `src/components/TimeDistanceDiagram.tsx` — D3 chart (time=x, distance=y)
- Grid lines: 5-minute time intervals, 1km distance intervals
- Station markers on y-axis with station names
- Pan/zoom: D3 zoom behavior on both axes
- Coordinate transformation: screen coords ↔ time/distance
- Responsive sizing: adjust to container width

**Day 30: Train path rendering**
- Fetch train run results from backend: GET /api/projects/:id/train-runs
- Parse trajectory data: array of {time_sec, station_m, speed_kmh}
- Render as D3 line with `curveMonotoneX` for smoothness
- Speed gradient color scale: d3.scaleSequential from blue (0 km/h) to red (max speed)
- Hover tooltip: show train name, time, station, speed
- Multiple trains: render all paths on same diagram with distinct colors
- Legend: train name + color swatch

**Day 31: Conflict visualization**
- Conflict detection algorithm: for each pair of trains, check if both occupy same block at overlapping times
- Highlight conflicts: red semi-transparent rectangles over conflicting time-distance regions
- Click conflict → modal with details (train IDs, block section, time window, suggested delay)
- Suggest resolution: compute minimum delay to one train to eliminate conflict
- Conflict count badge: show total number of conflicts in header
- Filter by train: hide/show individual train paths

**Day 32: Timetable editing**
- Drag train path vertically → adjust departure time (shift entire path in time)
- Drag train path horizontally → adjust run time (stretch/compress path)
- Add dwell time: click station marker on train path → input dwell duration → insert stop in trajectory
- Real-time conflict recomputation: as user drags, recalculate conflicts and update highlights
- Snap to grid: optional snapping to 1-minute time intervals
- Save timetable: POST /api/projects/:id/timetable with all train trajectories
- Export: timetable table (CSV/PDF) with arrival/departure times at each station

### Phase 7 — Reports (Days 33–37)

**Day 33: Python FastAPI service**
- `mkdir reports-service && cd reports-service`
- `pip install fastapi uvicorn reportlab matplotlib pandas pillow`
- `src/main.py` — FastAPI app with POST /generate-report endpoint
- Accepts JSON: {project_id, report_type, data: {...}}
- Returns: {report_url: "s3://...", status: "completed"}
- Dockerfile for containerization
- docker-compose integration with main backend

**Day 34: Geometry report**
- `src/geometry_report.py` — ReportLab PDF generation
- Title page: project name, date, engineer name, company logo (optional)
- Table of contents with page numbers
- Alignment table: columns = Element Type, Start Station (km+m), End Station, Length, Radius (for curves), Cant, Max Speed
- Speed profile chart: matplotlib line chart (station vs. speed), export as PNG, embed in PDF
- Cant diagram: matplotlib chart (station vs. cant mm), show equilibrium cant as dashed line
- Horizontal alignment plan: schematic top-down view with tangents/curves labeled
- Vertical profile: elevation chart with design grade line and existing ground
- PDF footer: page numbers, project name, generation timestamp

**Day 35: Signaling report**
- `src/signaling_report.py` — Block section and signal layout report
- Block section table: Block ID, Start Station, End Station, Length, Associated Signals
- Signal position table: Signal ID, Station, Type (home/distant/intermediate), Aspects, Sighting Distance
- Headway calculation summary: minimum headway, capacity (trains/hour), utilization percentage
- Schematic track diagram: simplified track layout with signals marked (use ReportLab drawing primitives)
- Interlocking table (if applicable): route definitions, conflicts, flank protection
- Capacity analysis chart: bar chart showing trains/hour vs. time of day

**Day 36: Braking report**
- `src/braking_report.py` — Braking distance compliance report
- Table of braking distances: columns = Initial Speed, Target Speed, Gradient (%), Service Brake Distance, Emergency Brake Distance
- Comparison to EN 14531 required distances: Pass/Fail column
- Safety margin verification: actual margin vs. required (1.3× service, 1.1× emergency)
- Chart: braking distance vs. initial speed for various gradients (0%, ±2%, ±4%)
- Adhesion scenario analysis: dry rail (μ=0.30), wet rail (μ=0.20), contaminated (μ=0.10)
- Recommendations: if any distances exceed limits, suggest lower speed or improved braking system

**Day 37: API integration**
- `src/api/handlers/reports.rs` — Rust backend handler
- POST /api/projects/:id/reports → validate user owns project
- Build report request JSON from project data (alignment, signaling, rolling stock)
- HTTP POST to Python reports service at http://reports-service:8001/generate-report
- Poll or webhook for completion (for long reports, use async task)
- Upload PDF to S3 with presigned URL (1-hour expiration)
- Insert into design_reports table: {project_id, report_type, s3_url, created_at}
- Return report URL to frontend
- Frontend: Reports tab lists all generated reports with download buttons
- Tests: end-to-end report generation, S3 upload verification, presigned URL expiration

### Phase 8 — Billing + Deployment (Days 38–42)

**Day 38: Stripe integration**
- `src/billing/stripe.rs` — Stripe SDK integration
- Create checkout session: POST /api/billing/checkout → Stripe Checkout Session URL
- Plan limits: Free (10km max), Pro (100km), Advanced (unlimited)
- Middleware to check project length vs. user plan before allowing edits
- Webhook handler: POST /api/billing/webhook (Stripe signature verification)
- Handle events: `checkout.session.completed`, `customer.subscription.updated`, `customer.subscription.deleted`
- Update users table: set plan, stripe_customer_id, stripe_subscription_id
- Customer Portal: generate portal session for subscription management
- Tests: checkout flow, webhook processing, plan limit enforcement

**Day 39: Usage tracking**
- Middleware: `src/middleware/usage_tracking.rs` — log billable actions to usage_records table
- Track: alignment updates (count), train simulations (count), reports generated (count)
- Monthly usage report for users: GET /api/usage → summary of current billing period
- Rate limiting: tower-governor middleware with different limits per plan
  - Free: 100 requests/minute, 1000/hour
  - Pro: 1000 requests/minute, 10000/hour
  - Advanced: 10000 requests/minute, unlimited/hour
- Soft limits with warning (90% of quota) and hard limits (reject requests)
- Tests: usage record insertion, rate limit enforcement, quota reset at month boundary

**Day 40: Production deployment**
- AWS infrastructure: Terraform scripts in `infra/`
- VPC with public/private subnets, NAT gateway, Internet Gateway
- EC2 Auto Scaling Group (t3.medium, min 2, max 10) with target tracking (CPU 70%)
- Application Load Balancer with HTTPS listener (ACM certificate)
- RDS PostgreSQL 16 (db.r6g.xlarge, Multi-AZ, automated backups 7 days)
- PostGIS extension enabled, read replica for heavy spatial queries
- ElastiCache Redis 7 (cache.r6g.large, encryption at rest/in transit)
- S3 buckets: railtrack-assets (CloudFront origin), railtrack-designs, railtrack-reports
- CloudFront distribution: custom domain, HTTPS, cache behaviors for /static, /wasm
- GitHub Actions CI/CD: .github/workflows/deploy.yml
  - Build Rust backend, run tests, build Docker image, push to ECR
  - Build WASM modules with wasm-pack, upload to S3
  - Build frontend with Vite, upload to S3/CloudFront, invalidate cache
  - Deploy backend: update ECS task definition or EC2 AMI, rolling deployment
- Secrets management: AWS Secrets Manager for DATABASE_URL, JWT_SECRET, STRIPE_SECRET_KEY

**Day 41: Monitoring**
- Prometheus metrics: `src/metrics.rs` with axum-prometheus crate
- Metrics: HTTP request duration histogram, request count by endpoint, DB query latency, WebSocket connections
- Prometheus scrape config: scrape backend /metrics endpoint every 15s
- Grafana dashboards:
  - API performance: request rate, latency percentiles (p50, p95, p99), error rate
  - Database: connection pool utilization, query latency, slow queries
  - System: CPU, memory, disk I/O, network throughput
  - Business: projects created, simulations run, reports generated, active users
- Alerting: Grafana alerts for error rate >1%, p95 latency >2s, DB connections >80%
- Sentry integration: `sentry-rust` crate for error tracking
- Capture all errors with stack traces, user context, breadcrumbs
- Structured logging: tracing-subscriber with JSON formatter, ship logs to CloudWatch Logs

**Day 42: Final testing**
- End-to-end tests: Playwright for frontend + backend integration
  - Test: create project → design alignment → run simulation → view speed profile → generate report → download PDF
  - Test: multi-user collaboration, concurrent edits, conflict resolution
  - Test: billing flow, plan upgrade, usage limit enforcement
- Load testing: k6 script for 100 concurrent users
  - Scenario: 20% read (get alignment), 60% edit (update alignment), 20% simulate
  - Target: p95 latency <1s, error rate <0.1%, 1000 req/s sustained
- Security audit:
  - SQL injection tests: sqlmap against all endpoints (should be blocked by SQLx compile-time checks)
  - XSS tests: inject scripts in project names, descriptions (should be sanitized)
  - CSRF protection: verify CSRF tokens on state-changing requests
  - JWT security: verify signature, check expiration, test token refresh flow
  - Rate limiting: verify limits enforced, test bypass attempts
- Performance profiling: Rust flamegraph for CPU hotspots, fix bottlenecks
- Documentation:
  - API reference: OpenAPI spec generated from Axum routes, served at /docs
  - User guide: video walkthrough (5 min) covering basic workflow
  - Developer docs: README with setup instructions, architecture diagram
- Launch checklist:
  - DNS: point railtrack.com to CloudFront/ALB
  - SSL: ACM certificate validated, HTTPS enforced
  - Stripe: switch from test mode to production mode
  - Monitoring: all alerts configured and tested
  - Backups: verify RDS automated backups, test restore procedure
  - Rollback plan: documented procedure to revert to previous version

---

## Critical Files

```
railtrack/
├── backend/
│   ├── src/
│   │   ├── main.rs (580 lines)
│   │   ├── config.rs (80 lines)
│   │   ├── state.rs (40 lines)
│   │   ├── error.rs (100 lines)
│   │   ├── auth/mod.rs (220 lines)
│   │   ├── api/handlers/
│   │   │   ├── auth.rs (270 lines)
│   │   │   ├── projects.rs (300 lines)
│   │   │   ├── alignment.rs (420 lines)
│   │   │   ├── signaling.rs (240 lines)
│   │   │   └── reports.rs (140 lines)
│   │   ├── db/models.rs (480 lines)
│   │   ├── solvers/signaling.rs (350 lines)
│   │   └── billing/stripe.rs (380 lines)
│   └── migrations/001_initial.sql (320 lines)
├── geometry-core/
│   ├── src/
│   │   ├── clothoid.rs (480 lines)
│   │   ├── cant.rs (260 lines)
│   │   ├── clearance.rs (320 lines)
│   │   └── validation.rs (380 lines)
├── solver-core/
│   ├── src/
│   │   ├── train_dynamics.rs (440 lines)
│   │   ├── resistance.rs (190 lines)
│   │   ├── traction.rs (160 lines)
│   │   └── braking.rs (240 lines)
├── geometry-wasm/src/lib.rs (220 lines)
├── solver-wasm/src/train_dynamics.rs (620 lines)
├── reports-service/
│   ├── src/
│   │   ├── geometry_report.py (540 lines)
│   │   ├── signaling_report.py (390 lines)
│   │   └── braking_report.py (260 lines)
├── frontend/
│   ├── src/
│   │   ├── components/
│   │   │   ├── MapView.tsx (300 lines)
│   │   │   ├── AlignmentOverlay.tsx (520 lines)
│   │   │   ├── CantEditor.tsx (360 lines)
│   │   │   ├── SpeedProfile.tsx (400 lines)
│   │   │   ├── VerticalProfile.tsx (340 lines)
│   │   │   └── TimeDistanceDiagram.tsx (580 lines)
│   │   └── stores/
│   │       ├── projectStore.ts (220 lines)
│   │       └── alignmentStore.ts (300 lines)
└── .github/workflows/
    ├── rust-ci.yml (90 lines)
    └── wasm-build.yml (75 lines)
```

---

## Solver Validation Suite

### Benchmark 1: Clothoid Endpoint Accuracy

**Test case:** A=300m, L=100m, azimuth=0°

**Expected:**
- Endpoint x: **99.954m** (±0.01m)
- Endpoint y: **5.547m** (±0.01m)
- Tangent angle: **9.536°** (±0.05°)

### Benchmark 2: Cant Deficiency

**Test case:** R=1000m, V=120 km/h, gauge=1.435m, cant=100mm

**Expected:**
- Equilibrium cant: **127.6mm**
- Cant deficiency: **27.6mm** (PASS <150mm)
- Max speed with 150mm limit: **136.4 km/h**

### Benchmark 3: Davis Resistance

**Test case:** EMU 200 tonnes, 100 km/h, A=650, B=13, C=0.045

**Expected:** R = **2400 N** (12 N/tonne)

### Benchmark 4: Braking Distance (EN 14531)

**Test case:** 160 km/h → 0, level, a=0.7 m/s², buildup=3s, margin=1.3×

**Expected:**
- Kinematic: 1417m
- Buildup: 133m
- Total: **2015m**

### Benchmark 5: Fixed-Block Headway

**Test case:** Block=2000m, V=100 km/h, cycle=10s

**Expected:**
- Traverse time: 72s
- Min headway: **82s**
- Capacity: **43.9 trains/hour**

---

## Verification Checklist

**Geometry:**
- [ ] Clothoid endpoints match reference <1cm/1km
- [ ] Cant deficiency triggers at 150mm/50mm
- [ ] Clearance accounts for cant+curve+suspension
- [ ] Vertical curves match AREMA formulas
- [ ] Validation rejects invalid radii

**Dynamics:**
- [ ] Davis resistance ±5% of published data
- [ ] Traction respects P/v and adhesion limits
- [ ] Braking includes buildup + EN 14531 margin
- [ ] Speed respects 1.0 m/s² lateral limit
- [ ] Gradient resistance correct sign

**Signaling:**
- [ ] Block length ≥ braking + 50m
- [ ] Sighting distance ≥200m per EN
- [ ] Headway = traverse + signal cycle
- [ ] Capacity matches manual calc
- [ ] Conflict detection accurate

**Frontend:**
- [ ] Map loads terrain/satellite
- [ ] SVG overlay syncs (no drift)
- [ ] WASM geometry <100ms
- [ ] Drag triggers recalc
- [ ] Speed profile updates <200ms

**API:**
- [ ] PostGIS stores/retrieves correctly
- [ ] Spatial queries work
- [ ] Auth rejects invalid JWTs
- [ ] Rate limiting enforced
- [ ] Stripe webhooks update DB

**Reports:**
- [ ] PDF includes all sections
- [ ] Generation <30s for 10km
- [ ] S3 URLs expire after 1h
- [ ] EN 14531 compliance validated

---

## Deployment Architecture

```
CloudFront CDN (React app, WASM)
    │
    ▼
Application Load Balancer (SSL, health checks)
    │
    ├─── Rust Backend EC2 (Auto Scaling 2-10)
    │    ├─── RDS PostgreSQL + PostGIS (Multi-AZ)
    │    └─── ElastiCache Redis (job queue, cache)
    │
    ├─── Background Workers (Spot Instances)
    │    ├─── Train Simulation (Rust)
    │    ├─── PDF Reports (Python)
    │    └─── Optimization (Python)
    │
    └─── S3 (designs, reports, results)

Monitoring: Prometheus + Grafana + Sentry
```

**Costs (500 users):** ~$830/month (EC2 $60, RDS $540, Redis $160, S3 $3, CloudFront $45, Spot $20)

---

## Post-MVP Roadmap

### Q1: Operations & Timetabling
- Moving-block signaling (ETCS L3, CBTC)
- Multi-train timetable with delay propagation
- UIC 406 capacity analysis
- Bottleneck identification

### Q2: Energy & Eco-Driving
- Energy consumption calculation
- Regenerative braking recovery (20-40%)
- Eco-driving optimization (10-20% savings)
- Catenary power supply design

### Q3: Advanced Geometry
- 3D alignment optimization (earthwork, curves, length)
- IFC Rail and RailML integration
- Compound curves, cubic parabola transitions
- DEM import for accurate earthworks

### Q4: Rolling Stock Library
- 100+ built-in models (HSR, metro, freight)
- User-uploaded custom rolling stock
- Detailed vehicle dynamics (bogie steering, ride comfort)
- Derailment risk analysis (Nadal formula)

---

**End of Implementation Plan**
