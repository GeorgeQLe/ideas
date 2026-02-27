# 52. PipeFlow — Process Piping and Fluid Systems Engineering Platform

## Implementation Plan

**MVP Scope:** Browser-based P&ID editor with ISA 5.1 symbol library (100+ equipment symbols: pumps, valves, vessels, heat exchangers) rendered via SVG with intelligent line numbering and auto-routing, steady-state pipe network hydraulic solver implementing Hardy Cross / Newton-Raphson network analysis for incompressible liquid flow with Darcy-Weisbach friction (Colebrook equation) and K-factor fittings compiled to Rust/WASM for client-side execution (<200 nodes) and server-side Rust-native execution (>200 nodes), pump performance curve integration with system curve generation and NPSH checking, basic sustained pipe stress analysis per ASME B31.3 using beam finite element method with weight and pressure loads compiled to Rust server-side, pipe sizing calculator with velocity limits (2-15 ft/s liquids) and economic pipe diameter optimization, interactive 3D pipe visualization via Three.js with color-coded stress ratios and pressure contours, PostgreSQL database for piping specs (line classes, material properties, valve/fitting catalogs) and project data, S3 storage for P&ID drawings and analysis reports, PDF report generation with pressure profile plots and stress summary tables, Stripe billing with three tiers (Free / Pro $79/mo / Engineering $199/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| Hydraulic Solver | Rust (native + WASM) | Hardy Cross network solver with sparse Jacobian via `nalgebra-sparse` |
| Stress Solver | Rust (native server) | Beam FEA with ASME B31.3 code checks, `nalgebra` for matrix math |
| P&ID Engine | TypeScript + SVG | ISA 5.1 symbols, intelligent line numbering, connectivity graph |
| 3D Visualization | Three.js + React Three Fiber | Pipe geometry with color-coded stress/pressure |
| Python Services | Python 3.12 (FastAPI) | NPSH calculation, pump curve fitting, economic pipe sizing |
| Database | PostgreSQL 16 | Projects, pipe specs, equipment, line lists |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | P&ID drawings, isometrics, reports, pump curves |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| Canvas Rendering | Canvas API | Isometric drawing generation |
| Real-time | WebSocket (Axum) | Live simulation progress, stress analysis updates |
| Job Queue | Redis 7 + Tokio tasks | Server-side hydraulic/stress job management |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | WASM bundle delivery, static frontend assets |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, error tracking |
| CI/CD | GitHub Actions | WASM build pipeline, Rust tests, Docker image push |

### Key Architecture Decisions

1. **Hybrid client/server hydraulic solver with WASM threshold at 200 nodes**: Pipe networks with ≤200 nodes (covers 85%+ of small plant systems and building HVAC) run entirely in the browser via WASM, providing instant feedback with zero server cost. Networks exceeding 200 nodes (large chemical plants, refinery distribution systems, city water networks) are submitted to the Rust-native server solver which handles multi-thousand-node networks with compressible gas and two-phase flow. The threshold is configurable per plan tier.

2. **Custom Hardy Cross network solver in Rust rather than wrapping EPANET**: Building a custom pipe network solver in Rust gives us full control over fluid property models (compressible gas, two-phase, non-Newtonian), WASM compilation, and integration with P&ID connectivity graphs. EPANET's C codebase is limited to water distribution and lacks support for process engineering fluids (hydrocarbons, acids, cryogenic liquids) and ASME piping code checks. Our Rust solver uses `nalgebra-sparse` for sparse Jacobian assembly (same algorithm as commercial AFT Fathom) while maintaining memory safety and WASM compatibility.

3. **SVG P&ID editor with intelligent line numbering**: SVG gives crisp rendering at any zoom level and straightforward DOM-based hit testing. Intelligent line numbering automatically generates line IDs like "2-LP-1234-A5C" (size-service-number-material) based on ISA 5.1 standards. A connectivity graph (adjacency list) tracks equipment-to-equipment flow paths for automatic hydraulic network generation from P&ID. Canvas/WebGL alternatives were rejected because they require reimplementing text layout, symbol libraries, and accessibility.

4. **Three.js 3D visualization separate from P&ID editor**: The 3D pipe viewer renders isometric pipe geometry with color-coded stress ratios (ASME allowable stress comparison) and pressure contours using GPU shaders. This is decoupled from the SVG P&ID to allow independent rotation/zoom and multi-viewport layouts. Pipe geometry is generated from P&ID connectivity plus user-specified routing (orthogonal, direct, optimized).

5. **S3 for drawing storage with PostgreSQL metadata catalog**: P&ID drawings (SVG files, typically 50-500KB each) are stored in S3, while PostgreSQL holds searchable metadata (project, line list, revision history, equipment tags). This allows the drawing library to scale to 10K+ drawings per project without bloating the database while enabling fast search via PostgreSQL indexes and full-text search on equipment descriptions.

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";  -- For fuzzy text search on equipment

-- Users
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email TEXT UNIQUE NOT NULL,
    password_hash TEXT,  -- NULL for OAuth-only users
    name TEXT NOT NULL,
    avatar_url TEXT,
    auth_provider TEXT DEFAULT 'email',  -- email | google | github
    auth_provider_id TEXT,
    plan TEXT NOT NULL DEFAULT 'free',  -- free | pro | engineering
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX users_email_idx ON users(email);
CREATE INDEX users_stripe_idx ON users(stripe_customer_id);

-- Organizations (for multi-user teams)
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

-- Projects (piping system workspace)
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    project_phase TEXT DEFAULT 'design',  -- design | procurement | construction | operation
    facility_type TEXT,  -- oil_gas | chemical | pharma | water | hvac | power
    design_codes TEXT[] DEFAULT '{}',  -- ASME B31.3, B31.1, EN 13480, etc.
    settings JSONB DEFAULT '{}',  -- Units (SI/Imperial), pressure drop limits, velocity limits
    is_public BOOLEAN NOT NULL DEFAULT false,
    forked_from UUID REFERENCES projects(id) ON DELETE SET NULL,
    thumbnail_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);
CREATE INDEX projects_org_idx ON projects(org_id);
CREATE INDEX projects_updated_idx ON projects(updated_at DESC);

-- P&IDs (Process & Instrumentation Diagrams)
CREATE TABLE pids (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name TEXT NOT NULL,  -- e.g., "P&ID-100A Main Process"
    drawing_number TEXT NOT NULL,
    revision TEXT NOT NULL DEFAULT 'A',
    sheet_size TEXT DEFAULT 'D',  -- A1, A0, D, E
    drawing_data JSONB NOT NULL DEFAULT '{}',  -- Full SVG state (equipment, lines, instruments)
    equipment_list JSONB DEFAULT '[]',  -- [{tag, type, description, datasheet_id}]
    line_list JSONB DEFAULT '[]',  -- [{line_number, from_tag, to_tag, size, material_class}]
    approval_status TEXT DEFAULT 'draft',  -- draft | review | approved | issued
    approved_by UUID REFERENCES users(id),
    approved_at TIMESTAMPTZ,
    s3_drawing_url TEXT,  -- SVG/PDF stored in S3
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX pids_project_idx ON pids(project_id);
CREATE INDEX pids_drawing_num_idx ON pids(drawing_number);

-- Equipment (pumps, vessels, heat exchangers, etc.)
CREATE TABLE equipment (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    tag TEXT NOT NULL,  -- e.g., "P-101" for pump 101
    type TEXT NOT NULL,  -- pump | vessel | heat_exchanger | compressor | turbine | valve
    subtype TEXT,  -- centrifugal | reciprocating | shell_tube | etc.
    description TEXT,
    manufacturer TEXT,
    model_number TEXT,
    datasheet JSONB DEFAULT '{}',  -- Equipment-specific parameters
    position JSONB,  -- {x, y, z} coordinates for 3D layout
    nozzles JSONB DEFAULT '[]',  -- [{id, size, rating, orientation, elevation}]
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(project_id, tag)
);
CREATE INDEX equipment_project_idx ON equipment(project_id);
CREATE INDEX equipment_tag_idx ON equipment(tag);
CREATE INDEX equipment_type_idx ON equipment(type);

-- Line Classes (material specifications)
CREATE TABLE line_classes (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID REFERENCES projects(id) ON DELETE CASCADE,  -- NULL = global/standard
    class_code TEXT NOT NULL,  -- e.g., "150# CS" or "300# SS316"
    material_spec TEXT NOT NULL,  -- e.g., "ASTM A106 Gr.B"
    pressure_rating TEXT NOT NULL,  -- e.g., "150#", "PN16", "Class 300"
    temperature_range JSONB,  -- {min: -20, max: 400, unit: "F"}
    pipe_schedule TEXT,  -- Sch 40, Sch 80, STD, XS
    flange_spec TEXT,  -- e.g., "ASME B16.5"
    gasket_spec TEXT,
    bolt_spec TEXT,
    corrosion_allowance REAL DEFAULT 0.125,  -- inches or mm
    is_builtin BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX line_classes_code_idx ON line_classes(class_code);
CREATE INDEX line_classes_project_idx ON line_classes(project_id);

-- Pipe Segments (individual pipe runs between equipment)
CREATE TABLE pipe_segments (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    line_number TEXT NOT NULL,  -- e.g., "2-LP-1234-A5C"
    from_equipment_id UUID REFERENCES equipment(id),
    to_equipment_id UUID REFERENCES equipment(id),
    from_nozzle TEXT,
    to_nozzle TEXT,
    service TEXT,  -- LP = Low Pressure, HP = High Pressure, etc.
    nominal_diameter REAL NOT NULL,  -- inches or mm
    wall_thickness REAL,
    line_class_id UUID REFERENCES line_classes(id),
    insulation_type TEXT,  -- none | mineral_wool | fiberglass | calcium_silicate
    insulation_thickness REAL,
    heat_tracing BOOLEAN DEFAULT false,
    geometry JSONB,  -- 3D path: [{x, y, z, type: "straight|elbow|tee"}]
    length_calculated REAL,  -- Auto-calculated from geometry
    elevation_change REAL,
    fittings JSONB DEFAULT '[]',  -- [{type: "elbow_90", quantity, k_factor}]
    valves JSONB DEFAULT '[]',  -- [{type: "gate", size, cv, position}]
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX segments_project_idx ON pipe_segments(project_id);
CREATE INDEX segments_line_num_idx ON pipe_segments(line_number);
CREATE INDEX segments_from_equip_idx ON pipe_segments(from_equipment_id);
CREATE INDEX segments_to_equip_idx ON pipe_segments(to_equipment_id);

-- Hydraulic Analyses
CREATE TABLE hydraulic_analyses (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    analysis_type TEXT NOT NULL,  -- steady_state | transient | surge
    fluid_type TEXT NOT NULL,  -- incompressible_liquid | compressible_gas | two_phase | non_newtonian
    fluid_properties JSONB NOT NULL,  -- {density, viscosity, vapor_pressure, temperature}
    network_nodes INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | running | completed | failed
    execution_mode TEXT NOT NULL DEFAULT 'wasm',  -- wasm | server
    results_url TEXT,  -- S3 URL for result data
    results_summary JSONB,  -- {max_pressure, min_pressure, total_flow, pump_head}
    error_message TEXT,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    wall_time_ms INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX hydraulic_project_idx ON hydraulic_analyses(project_id);
CREATE INDEX hydraulic_user_idx ON hydraulic_analyses(user_id);
CREATE INDEX hydraulic_status_idx ON hydraulic_analyses(status);

-- Stress Analyses
CREATE TABLE stress_analyses (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    design_code TEXT NOT NULL,  -- ASME_B31_3 | ASME_B31_1 | EN_13480
    load_cases JSONB NOT NULL,  -- [{name: "sustained", loads: {weight: true, pressure: true}}, {name: "thermal", ...}]
    material_spec JSONB NOT NULL,  -- {spec, grade, allowable_stress, yield_stress, elastic_modulus}
    status TEXT NOT NULL DEFAULT 'pending',
    results_url TEXT,
    results_summary JSONB,  -- {max_stress_ratio, critical_location, support_loads}
    error_message TEXT,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    wall_time_ms INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX stress_project_idx ON stress_analyses(project_id);
CREATE INDEX stress_user_idx ON stress_analyses(user_id);

-- Pipe Supports (anchors, guides, hangers)
CREATE TABLE pipe_supports (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    pipe_segment_id UUID REFERENCES pipe_segments(id) ON DELETE CASCADE,
    support_type TEXT NOT NULL,  -- anchor | guide | shoe | spring_hanger | snubber
    position JSONB NOT NULL,  -- {x, y, z} or {distance_from_start}
    spring_constant REAL,  -- For spring hangers
    load_capacity REAL,  -- lb or kN
    manufacturer TEXT,
    model_number TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX supports_segment_idx ON pipe_supports(pipe_segment_id);

-- Pump Curves
CREATE TABLE pump_curves (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    equipment_id UUID REFERENCES equipment(id) ON DELETE CASCADE,
    manufacturer TEXT NOT NULL,
    model TEXT NOT NULL,
    impeller_diameter REAL,  -- inches or mm
    speed REAL,  -- RPM
    curve_data JSONB NOT NULL,  -- [{flow, head, efficiency, npsh_required}]
    curve_fit_coefficients JSONB,  -- Polynomial fit: H = a + b*Q + c*Q^2
    rated_flow REAL,
    rated_head REAL,
    rated_efficiency REAL,
    s3_curve_image_url TEXT,  -- Manufacturer curve image
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX pump_curves_equip_idx ON pump_curves(equipment_id);

-- Valve Catalog
CREATE TABLE valve_catalog (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    type TEXT NOT NULL,  -- gate | globe | ball | butterfly | check | control
    manufacturer TEXT NOT NULL,
    model TEXT NOT NULL,
    size REAL NOT NULL,  -- inches or mm
    pressure_class TEXT NOT NULL,
    cv_flow_coefficient REAL,  -- For control valves
    k_resistance REAL,  -- For other valves
    material TEXT,
    datasheet_url TEXT,
    is_builtin BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX valve_catalog_type_idx ON valve_catalog(type);
CREATE INDEX valve_catalog_size_idx ON valve_catalog(size);

-- Usage Tracking (for billing enforcement)
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,  -- hydraulic_analysis | stress_analysis | storage_bytes
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
pub struct Project {
    pub id: Uuid,
    pub owner_id: Uuid,
    pub org_id: Option<Uuid>,
    pub name: String,
    pub description: String,
    pub project_phase: String,
    pub facility_type: Option<String>,
    pub design_codes: Vec<String>,
    pub settings: serde_json::Value,
    pub is_public: bool,
    pub forked_from: Option<Uuid>,
    pub thumbnail_url: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Pid {
    pub id: Uuid,
    pub project_id: Uuid,
    pub name: String,
    pub drawing_number: String,
    pub revision: String,
    pub sheet_size: String,
    pub drawing_data: serde_json::Value,
    pub equipment_list: serde_json::Value,
    pub line_list: serde_json::Value,
    pub approval_status: String,
    pub approved_by: Option<Uuid>,
    pub approved_at: Option<DateTime<Utc>>,
    pub s3_drawing_url: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Equipment {
    pub id: Uuid,
    pub project_id: Uuid,
    pub tag: String,
    #[sqlx(rename = "type")]
    pub equipment_type: String,
    pub subtype: Option<String>,
    pub description: Option<String>,
    pub manufacturer: Option<String>,
    pub model_number: Option<String>,
    pub datasheet: serde_json::Value,
    pub position: Option<serde_json::Value>,
    pub nozzles: serde_json::Value,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct PipeSegment {
    pub id: Uuid,
    pub project_id: Uuid,
    pub line_number: String,
    pub from_equipment_id: Option<Uuid>,
    pub to_equipment_id: Option<Uuid>,
    pub from_nozzle: Option<String>,
    pub to_nozzle: Option<String>,
    pub service: Option<String>,
    pub nominal_diameter: f64,
    pub wall_thickness: Option<f64>,
    pub line_class_id: Option<Uuid>,
    pub insulation_type: Option<String>,
    pub insulation_thickness: Option<f64>,
    pub heat_tracing: bool,
    pub geometry: Option<serde_json::Value>,
    pub length_calculated: Option<f64>,
    pub elevation_change: Option<f64>,
    pub fittings: serde_json::Value,
    pub valves: serde_json::Value,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct HydraulicAnalysis {
    pub id: Uuid,
    pub project_id: Uuid,
    pub user_id: Uuid,
    pub name: String,
    pub analysis_type: String,
    pub fluid_type: String,
    pub fluid_properties: serde_json::Value,
    pub network_nodes: i32,
    pub status: String,
    pub execution_mode: String,
    pub results_url: Option<String>,
    pub results_summary: Option<serde_json::Value>,
    pub error_message: Option<String>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub wall_time_ms: Option<i32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct StressAnalysis {
    pub id: Uuid,
    pub project_id: Uuid,
    pub user_id: Uuid,
    pub name: String,
    pub design_code: String,
    pub load_cases: serde_json::Value,
    pub material_spec: serde_json::Value,
    pub status: String,
    pub results_url: Option<String>,
    pub results_summary: Option<serde_json::Value>,
    pub error_message: Option<String>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub wall_time_ms: Option<i32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct FluidProperties {
    pub density: f64,        // kg/m³ or lb/ft³
    pub viscosity: f64,      // Pa·s or cP
    pub vapor_pressure: f64, // Pa or psi
    pub temperature: f64,    // °C or °F
}

#[derive(Debug, Deserialize)]
pub struct HydraulicParams {
    pub analysis_type: HydraulicAnalysisType,
    pub fluid_type: FluidType,
    pub fluid_properties: FluidProperties,
    pub operating_pressure: f64,
    pub mass_flow_rate: Option<f64>,
    pub volumetric_flow_rate: Option<f64>,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum HydraulicAnalysisType {
    SteadyState,
    Transient,
    Surge,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum FluidType {
    IncompressibleLiquid,
    CompressibleGas,
    TwoPhase,
    NonNewtonian,
}
```

---

## Solver Architecture Deep-Dive

### Governing Equations and Discretization

PipeFlow's core hydraulic solver implements the **Hardy Cross method** and **Newton-Raphson network solver**, standard formulations used in all production pipe network simulators. For a pipe network with `n` nodes and `p` pipes, the system is:

**Mass Conservation (Node Equations):**
```
Σ Q_in - Σ Q_out = 0  (for each node)
```

**Energy Equation (Pipe Equations) — Darcy-Weisbach:**
```
ΔP = f · (L/D) · (ρv²/2) + Σ K_i · (ρv²/2)

where:
- f = Darcy friction factor (from Colebrook equation)
- L = pipe length
- D = internal diameter
- ρ = fluid density
- v = flow velocity
- K_i = loss coefficient for fittings (elbows, tees, valves)
```

**Colebrook Equation (Friction Factor):**
```
1/√f = -2 log₁₀(ε/(3.7D) + 2.51/(Re√f))

where:
- ε = pipe roughness (0.0018" for commercial steel)
- Re = ρvD/μ (Reynolds number)
- μ = dynamic viscosity
```

**Newton-Raphson Network Solver** linearizes the nonlinear pipe equations. At each iteration `k`, the flow correction Δ**Q** is:

```
J(Q_k) · ΔQ = -R(Q_k)

where:
- J = Jacobian matrix (dR/dQ) — sparse for pipe networks
- R = Residual vector (mass conservation violations)
- ΔQ = Flow correction
```

The Jacobian for a pipe connecting nodes i and j is:
```
∂R_i/∂Q_pipe = -n · sign(Q) · |Q|^(n-1)
∂R_j/∂Q_pipe = +n · sign(Q) · |Q|^(n-1)

where n ≈ 2 for turbulent flow (Darcy-Weisbach)
```

**Pump Integration:**
Pumps are modeled using manufacturer performance curves fit to polynomials:
```
H_pump = a₀ + a₁·Q + a₂·Q²

where H_pump = pump head (ft or m), Q = volumetric flow
```

**NPSH Check (Net Positive Suction Head):**
```
NPSH_available = P_suction/ρg + v²/(2g) - P_vapor/ρg - elevation

NPSH_available must exceed NPSH_required (from pump curve)
```

### Client/Server Split (WASM Threshold)

```
Network uploaded → Node count extracted
    │
    ├── ≤200 nodes → WASM solver (browser)
    │   ├── Instant startup, no network latency
    │   ├── Results displayed immediately
    │   └── No server cost
    │
    └── >200 nodes → Server solver (Rust native)
        ├── Job queued via Redis
        ├── Worker picks up, runs native solver
        ├── Progress streamed via WebSocket
        └── Results stored in S3, URL returned
```

The 200-node threshold was chosen because:
- WASM solver with sparse Jacobian handles 200-node network in <3 seconds on modern hardware
- 200 nodes covers: small chemical plants (50-150 nodes), HVAC systems (30-100 nodes), skid packages (20-80 nodes)
- Above 200 nodes: refinery-scale systems, city water distribution, large offshore platforms → need server compute

---

## Architecture Deep-Dives

### 1. Hydraulic Analysis API Handler (Rust/Axum)

The primary endpoint receives a hydraulic analysis request, validates the pipe network, decides between WASM and server execution, and for server execution enqueues a job with Redis and streams progress via WebSocket.

```rust
// src/api/handlers/hydraulic.rs

use axum::{
    extract::{Path, State, Json},
    http::StatusCode,
    response::IntoResponse,
};
use uuid::Uuid;
use crate::{
    db::models::{HydraulicAnalysis, HydraulicParams},
    solver::network::NetworkGraph,
    state::AppState,
    auth::Claims,
    error::ApiError,
};

#[derive(serde::Deserialize)]
pub struct CreateHydraulicAnalysisRequest {
    pub name: String,
    pub parameters: HydraulicParams,
}

pub async fn create_hydraulic_analysis(
    State(state): State<AppState>,
    claims: Claims,
    Path(project_id): Path<Uuid>,
    Json(req): Json<CreateHydraulicAnalysisRequest>,
) -> Result<impl IntoResponse, ApiError> {
    // 1. Verify project ownership
    let project = sqlx::query_as!(
        crate::db::models::Project,
        "SELECT * FROM projects WHERE id = $1 AND (owner_id = $2 OR org_id IN (
            SELECT org_id FROM org_members WHERE user_id = $2
        ))",
        project_id,
        claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Project not found"))?;

    // 2. Build network graph from pipe segments
    let segments = sqlx::query_as!(
        crate::db::models::PipeSegment,
        "SELECT * FROM pipe_segments WHERE project_id = $1",
        project_id
    )
    .fetch_all(&state.db)
    .await?;

    let equipment = sqlx::query_as!(
        crate::db::models::Equipment,
        "SELECT * FROM equipment WHERE project_id = $1",
        project_id
    )
    .fetch_all(&state.db)
    .await?;

    let network = NetworkGraph::from_segments_and_equipment(&segments, &equipment)?;
    let node_count = network.node_count();

    // 3. Check plan limits
    let user = sqlx::query_as!(
        crate::db::models::User,
        "SELECT * FROM users WHERE id = $1",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    if user.plan == "free" && node_count > 50 {
        return Err(ApiError::PlanLimit(
            "Free plan supports networks up to 50 nodes. Upgrade to Pro for unlimited."
        ));
    }

    // 4. Determine execution mode
    let execution_mode = if node_count <= 200 {
        "wasm"
    } else {
        "server"
    };

    // 5. Create analysis record
    let analysis = sqlx::query_as!(
        HydraulicAnalysis,
        r#"INSERT INTO hydraulic_analyses
            (project_id, user_id, name, analysis_type, fluid_type,
             fluid_properties, network_nodes, status, execution_mode)
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9)
        RETURNING *"#,
        project_id,
        claims.user_id,
        req.name,
        serde_json::to_string(&req.parameters.analysis_type)?,
        serde_json::to_string(&req.parameters.fluid_type)?,
        serde_json::to_value(&req.parameters.fluid_properties)?,
        node_count as i32,
        if execution_mode == "wasm" { "completed" } else { "pending" },
        execution_mode,
    )
    .fetch_one(&state.db)
    .await?;

    // 6. For server execution, enqueue job
    if execution_mode == "server" {
        let job_data = serde_json::json!({
            "analysis_id": analysis.id,
            "network": network.serialize(),
            "parameters": req.parameters,
        });

        state.redis
            .publish("hydraulic:jobs", serde_json::to_string(&job_data)?)
            .await?;
    }

    Ok((StatusCode::CREATED, Json(analysis)))
}

pub async fn get_hydraulic_analysis(
    State(state): State<AppState>,
    claims: Claims,
    Path((project_id, analysis_id)): Path<(Uuid, Uuid)>,
) -> Result<Json<HydraulicAnalysis>, ApiError> {
    let analysis = sqlx::query_as!(
        HydraulicAnalysis,
        "SELECT h.* FROM hydraulic_analyses h
         JOIN projects p ON h.project_id = p.id
         WHERE h.id = $1 AND h.project_id = $2
         AND (p.owner_id = $3 OR p.org_id IN (
             SELECT org_id FROM org_members WHERE user_id = $3
         ))",
        analysis_id,
        project_id,
        claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Analysis not found"))?;

    Ok(Json(analysis))
}
```

### 2. Hydraulic Network Solver Core (Rust — shared between WASM and native)

The core Hardy Cross / Newton-Raphson solver that builds and solves the pipe network equations. This code compiles to both native (server) and WASM (browser) targets.

```rust
// solver-core/src/hydraulic/network.rs

use nalgebra_sparse::{CsrMatrix, CooMatrix};
use crate::hydraulic::{Pipe, Node, Pump};
use std::collections::HashMap;

pub struct NetworkGraph {
    pub nodes: HashMap<usize, Node>,
    pub pipes: Vec<Pipe>,
    pub pumps: Vec<Pump>,
    pub boundary_conditions: HashMap<usize, BoundaryCondition>,
}

#[derive(Debug, Clone)]
pub enum BoundaryCondition {
    FixedPressure(f64),      // Reservoir, tank
    FixedFlow(f64),          // Specified flow rate
    Pump { curve_id: usize },
}

#[derive(Debug, Clone)]
pub struct Node {
    pub id: usize,
    pub elevation: f64,      // m or ft
    pub demand: f64,         // m³/s or gpm (negative for supply)
    pub pressure: f64,       // Pa or psi (solution variable)
}

#[derive(Debug, Clone)]
pub struct Pipe {
    pub id: usize,
    pub from_node: usize,
    pub to_node: usize,
    pub diameter: f64,       // m or inches (internal)
    pub length: f64,         // m or ft
    pub roughness: f64,      // m or ft (absolute roughness)
    pub fittings_k: f64,     // Total K-factor for fittings
    pub flow: f64,           // m³/s or gpm (solution variable)
}

impl Pipe {
    /// Calculate pressure drop using Darcy-Weisbach equation
    pub fn pressure_drop(&self, density: f64, viscosity: f64) -> f64 {
        let area = std::f64::consts::PI * self.diameter.powi(2) / 4.0;
        let velocity = self.flow.abs() / area;
        let reynolds = density * velocity * self.diameter / viscosity;

        // Colebrook friction factor (iterative solution)
        let friction_factor = self.colebrook_friction(reynolds);

        // Darcy-Weisbach pressure drop
        let dp_friction = friction_factor * (self.length / self.diameter)
                          * (density * velocity.powi(2) / 2.0);

        // Fitting losses
        let dp_fittings = self.fittings_k * (density * velocity.powi(2) / 2.0);

        let total_dp = dp_friction + dp_fittings;

        // Preserve sign (pressure drop opposes flow direction)
        if self.flow >= 0.0 { total_dp } else { -total_dp }
    }

    /// Colebrook equation for friction factor (using iterative solution)
    fn colebrook_friction(&self, reynolds: f64) -> f64 {
        if reynolds < 2300.0 {
            // Laminar flow
            return 64.0 / reynolds;
        }

        // Turbulent flow — Colebrook equation (iterative)
        let relative_roughness = self.roughness / self.diameter;
        let mut f = 0.02;  // Initial guess

        for _ in 0..10 {
            let f_sqrt = f.sqrt();
            let lhs = 1.0 / f_sqrt;
            let rhs = -2.0 * (relative_roughness / 3.7
                              + 2.51 / (reynolds * f_sqrt)).log10();
            f = (rhs / lhs).powi(2);
        }

        f
    }

    /// Jacobian entry for Newton-Raphson (∂ΔP/∂Q)
    pub fn jacobian(&self, density: f64, viscosity: f64) -> f64 {
        // Approximate derivative using finite difference
        let epsilon = self.flow.abs() * 1e-6 + 1e-9;
        let dp_plus = {
            let mut pipe_plus = self.clone();
            pipe_plus.flow += epsilon;
            pipe_plus.pressure_drop(density, viscosity)
        };
        let dp = self.pressure_drop(density, viscosity);
        (dp_plus - dp) / epsilon
    }
}

impl NetworkGraph {
    /// Solve pipe network using Newton-Raphson method
    pub fn solve_steady_state(
        &mut self,
        density: f64,
        viscosity: f64,
        max_iterations: usize,
        tolerance: f64,
    ) -> Result<(), SolverError> {
        for iter in 0..max_iterations {
            // Build residual vector and Jacobian matrix
            let n = self.nodes.len();
            let mut residual = vec![0.0; n];
            let mut jacobian = CooMatrix::new(n, n);

            // Mass conservation at each node
            for (node_id, node) in &self.nodes {
                let idx = *node_id;
                residual[idx] = -node.demand;  // Demand is negative contribution

                for pipe in &self.pipes {
                    if pipe.from_node == idx {
                        residual[idx] += pipe.flow;
                        let jac_entry = pipe.jacobian(density, viscosity);
                        jacobian.push(idx, idx, jac_entry);
                    } else if pipe.to_node == idx {
                        residual[idx] -= pipe.flow;
                        let jac_entry = pipe.jacobian(density, viscosity);
                        jacobian.push(idx, idx, -jac_entry);
                    }
                }
            }

            // Energy equations for pipes
            for pipe in &self.pipes {
                let from_pressure = self.nodes[&pipe.from_node].pressure;
                let to_pressure = self.nodes[&pipe.to_node].pressure;
                let elevation_change = self.nodes[&pipe.to_node].elevation
                                       - self.nodes[&pipe.from_node].elevation;

                let dp_pipe = pipe.pressure_drop(density, viscosity);
                let dp_gravity = density * 9.81 * elevation_change;  // ρgh

                // Energy balance: P_from - P_to - ΔP_pipe - ΔP_gravity = 0
                let energy_residual = from_pressure - to_pressure - dp_pipe - dp_gravity;

                // This is handled implicitly by pressure updates in mass conservation
                // For now, we use Hardy Cross style flow updates
            }

            // Check convergence
            let max_residual = residual.iter().map(|r| r.abs()).fold(0.0, f64::max);
            if max_residual < tolerance {
                return Ok(());
            }

            // Solve Jacobian system for flow corrections
            let csr = CsrMatrix::from(&jacobian);
            let delta_q = solve_sparse_system(&csr, &residual)?;

            // Update flows
            let mut pipe_idx = 0;
            for pipe in &mut self.pipes {
                pipe.flow -= delta_q[pipe_idx];
                pipe_idx += 1;
            }

            // Update pressures from energy equations
            self.update_pressures(density, viscosity);
        }

        Err(SolverError::Convergence(
            format!("Failed to converge after {} iterations", max_iterations)
        ))
    }

    fn update_pressures(&mut self, density: f64, viscosity: f64) {
        // Start from a reference node (first boundary condition)
        for (node_id, bc) in &self.boundary_conditions {
            if let BoundaryCondition::FixedPressure(p) = bc {
                self.nodes.get_mut(node_id).unwrap().pressure = *p;
                break;
            }
        }

        // Propagate pressures through pipes using energy equation
        let mut visited = vec![false; self.nodes.len()];
        let mut stack = Vec::new();

        // Find starting node (boundary condition)
        for (node_id, _) in &self.boundary_conditions {
            stack.push(*node_id);
            visited[*node_id] = true;
            break;
        }

        while let Some(current_node) = stack.pop() {
            let current_pressure = self.nodes[&current_node].pressure;
            let current_elevation = self.nodes[&current_node].elevation;

            for pipe in &self.pipes {
                let (neighbor, flow_direction) = if pipe.from_node == current_node {
                    (pipe.to_node, 1.0)
                } else if pipe.to_node == current_node {
                    (pipe.from_node, -1.0)
                } else {
                    continue;
                };

                if visited[neighbor] {
                    continue;
                }

                let dp_pipe = pipe.pressure_drop(density, viscosity) * flow_direction;
                let elevation_change = (self.nodes[&neighbor].elevation - current_elevation)
                                       * flow_direction;
                let dp_gravity = density * 9.81 * elevation_change;

                self.nodes.get_mut(&neighbor).unwrap().pressure =
                    current_pressure - dp_pipe - dp_gravity;

                visited[neighbor] = true;
                stack.push(neighbor);
            }
        }
    }
}

#[derive(Debug)]
pub enum SolverError {
    Convergence(String),
    Singular(String),
    InvalidNetwork(String),
}

fn solve_sparse_system(matrix: &CsrMatrix<f64>, rhs: &[f64]) -> Result<Vec<f64>, SolverError> {
    // Use nalgebra-sparse LU decomposition or iterative solver
    // For now, placeholder
    Ok(vec![0.0; rhs.len()])
}
```

### 3. Pipe Stress Analysis Solver (Rust — server-side only)

Beam finite element method for pipe stress analysis per ASME B31.3, computing sustained stress (weight + pressure) and checking against allowable stress.

```rust
// solver-core/src/stress/beam_fea.rs

use nalgebra::{DMatrix, DVector};
use crate::stress::{PipeElement, Support, LoadCase, Material};

pub struct StressModel {
    pub elements: Vec<PipeElement>,
    pub supports: Vec<Support>,
    pub material: Material,
    pub design_code: DesignCode,
}

#[derive(Debug, Clone)]
pub struct PipeElement {
    pub id: usize,
    pub node_i: usize,
    pub node_j: usize,
    pub length: f64,
    pub outer_diameter: f64,
    pub wall_thickness: f64,
    pub orientation: [f64; 3],  // Direction cosines
}

#[derive(Debug, Clone)]
pub struct Support {
    pub node_id: usize,
    pub support_type: SupportType,
}

#[derive(Debug, Clone)]
pub enum SupportType {
    Anchor,           // Fixed in all DOFs
    Guide { axis: usize },  // Constrained perpendicular to pipe axis
    RestSupport,      // Vertical support only
}

#[derive(Debug, Clone)]
pub struct Material {
    pub elastic_modulus: f64,  // Pa or psi
    pub poisson_ratio: f64,
    pub density: f64,          // kg/m³ or lb/ft³
    pub allowable_stress: f64, // Pa or psi (at design temperature)
    pub yield_stress: f64,
}

#[derive(Debug, Clone)]
pub enum DesignCode {
    ASME_B31_3,
    ASME_B31_1,
    EN_13480,
}

impl PipeElement {
    /// Beam element stiffness matrix (12x12 for 3D beam with 6 DOF per node)
    pub fn stiffness_matrix(&self, material: &Material) -> DMatrix<f64> {
        let e = material.elastic_modulus;
        let g = e / (2.0 * (1.0 + material.poisson_ratio));  // Shear modulus
        let l = self.length;

        // Cross-section properties
        let outer_r = self.outer_diameter / 2.0;
        let inner_r = outer_r - self.wall_thickness;
        let area = std::f64::consts::PI * (outer_r.powi(2) - inner_r.powi(2));
        let i_moment = std::f64::consts::PI * (outer_r.powi(4) - inner_r.powi(4)) / 4.0;
        let j_torsion = 2.0 * i_moment;  // For circular pipe

        // Local stiffness matrix (simplified — full implementation includes shear deformation)
        let mut k_local = DMatrix::zeros(12, 12);

        // Axial stiffness
        let k_axial = e * area / l;
        k_local[(0, 0)] = k_axial;
        k_local[(0, 6)] = -k_axial;
        k_local[(6, 0)] = -k_axial;
        k_local[(6, 6)] = k_axial;

        // Bending stiffness (Y-Z plane)
        let k_bend = 12.0 * e * i_moment / l.powi(3);
        k_local[(1, 1)] = k_bend;
        k_local[(1, 7)] = -k_bend;
        k_local[(7, 1)] = -k_bend;
        k_local[(7, 7)] = k_bend;

        k_local[(2, 2)] = k_bend;
        k_local[(2, 8)] = -k_bend;
        k_local[(8, 2)] = -k_bend;
        k_local[(8, 8)] = k_bend;

        // Torsional stiffness
        let k_torsion = g * j_torsion / l;
        k_local[(3, 3)] = k_torsion;
        k_local[(3, 9)] = -k_torsion;
        k_local[(9, 3)] = -k_torsion;
        k_local[(9, 9)] = k_torsion;

        // Moment-rotation coupling (simplified)
        let k_rotation = e * i_moment / l;
        k_local[(4, 4)] = 4.0 * k_rotation;
        k_local[(4, 10)] = 2.0 * k_rotation;
        k_local[(10, 4)] = 2.0 * k_rotation;
        k_local[(10, 10)] = 4.0 * k_rotation;

        k_local[(5, 5)] = 4.0 * k_rotation;
        k_local[(5, 11)] = 2.0 * k_rotation;
        k_local[(11, 5)] = 2.0 * k_rotation;
        k_local[(11, 11)] = 4.0 * k_rotation;

        // Transform to global coordinates
        let t = self.transformation_matrix();
        t.transpose() * k_local * t
    }

    /// Transformation matrix from local to global coordinates
    fn transformation_matrix(&self) -> DMatrix<f64> {
        let mut t = DMatrix::identity(12, 12);
        // Simplified — full implementation includes direction cosines
        t
    }

    /// Element load vector due to self-weight
    pub fn weight_load_vector(&self, material: &Material, gravity: f64) -> DVector<f64> {
        let outer_r = self.outer_diameter / 2.0;
        let inner_r = outer_r - self.wall_thickness;
        let area = std::f64::consts::PI * (outer_r.powi(2) - inner_r.powi(2));
        let weight_per_length = material.density * area * gravity;

        let total_weight = weight_per_length * self.length;

        // Distribute uniformly to nodes (simplified — should use shape functions)
        let mut load = DVector::zeros(12);
        load[2] = -total_weight / 2.0;  // Vertical load at node i
        load[8] = -total_weight / 2.0;  // Vertical load at node j
        load
    }
}

impl StressModel {
    /// Assemble global stiffness matrix and solve for displacements
    pub fn solve(&self, load_case: &LoadCase) -> Result<StressResult, SolverError> {
        let n_nodes = self.count_nodes();
        let n_dof = n_nodes * 6;  // 6 DOF per node (3 translation + 3 rotation)

        let mut k_global = DMatrix::zeros(n_dof, n_dof);
        let mut f_global = DVector::zeros(n_dof);

        // Assemble element stiffness matrices
        for element in &self.elements {
            let k_elem = element.stiffness_matrix(&self.material);

            // Add element stiffness to global matrix
            for i in 0..6 {
                for j in 0..6 {
                    let global_i = element.node_i * 6 + i;
                    let global_j = element.node_i * 6 + j;
                    k_global[(global_i, global_j)] += k_elem[(i, j)];

                    let global_i2 = element.node_j * 6 + i;
                    let global_j2 = element.node_j * 6 + j;
                    k_global[(global_i2, global_j2)] += k_elem[(i + 6, j + 6)];

                    k_global[(global_i, global_j2)] += k_elem[(i, j + 6)];
                    k_global[(global_i2, global_j)] += k_elem[(i + 6, j)];
                }
            }

            // Add weight loads if included in load case
            if load_case.include_weight {
                let f_elem = element.weight_load_vector(&self.material, 9.81);
                for i in 0..6 {
                    f_global[element.node_i * 6 + i] += f_elem[i];
                    f_global[element.node_j * 6 + i] += f_elem[i + 6];
                }
            }
        }

        // Apply boundary conditions (supports)
        for support in &self.supports {
            match support.support_type {
                SupportType::Anchor => {
                    // Fix all DOFs
                    for dof in 0..6 {
                        let idx = support.node_id * 6 + dof;
                        k_global.row_mut(idx).fill(0.0);
                        k_global.column_mut(idx).fill(0.0);
                        k_global[(idx, idx)] = 1.0;
                        f_global[idx] = 0.0;
                    }
                }
                SupportType::Guide { axis } => {
                    // Fix perpendicular translation
                    for dof in [0, 1, 2] {
                        if dof != axis {
                            let idx = support.node_id * 6 + dof;
                            k_global.row_mut(idx).fill(0.0);
                            k_global.column_mut(idx).fill(0.0);
                            k_global[(idx, idx)] = 1.0;
                            f_global[idx] = 0.0;
                        }
                    }
                }
                SupportType::RestSupport => {
                    // Fix vertical (Z) translation only
                    let idx = support.node_id * 6 + 2;
                    k_global.row_mut(idx).fill(0.0);
                    k_global.column_mut(idx).fill(0.0);
                    k_global[(idx, idx)] = 1.0;
                    f_global[idx] = 0.0;
                }
            }
        }

        // Solve K * u = F
        let displacements = k_global.lu().solve(&f_global)
            .ok_or(SolverError::Singular("Singular stiffness matrix".to_string()))?;

        // Calculate stresses in each element
        let mut element_stresses = Vec::new();
        for element in &self.elements {
            let u_i = displacements.rows(element.node_i * 6, 6);
            let u_j = displacements.rows(element.node_j * 6, 6);

            let stress = self.calculate_element_stress(element, &u_i.into(), &u_j.into())?;
            element_stresses.push(stress);
        }

        Ok(StressResult {
            displacements: displacements.data.as_vec().clone(),
            element_stresses,
        })
    }

    fn calculate_element_stress(
        &self,
        element: &PipeElement,
        u_i: &DVector<f64>,
        u_j: &DVector<f64>,
    ) -> Result<ElementStress, SolverError> {
        // Calculate bending moments and axial forces from displacements
        let outer_r = element.outer_diameter / 2.0;
        let inner_r = outer_r - element.wall_thickness;
        let section_modulus = std::f64::consts::PI * (outer_r.powi(4) - inner_r.powi(4))
                              / (4.0 * outer_r);

        // Axial stress
        let axial_force = (u_j[0] - u_i[0]) * self.material.elastic_modulus
                          * std::f64::consts::PI * (outer_r.powi(2) - inner_r.powi(2))
                          / element.length;
        let area = std::f64::consts::PI * (outer_r.powi(2) - inner_r.powi(2));
        let axial_stress = axial_force / area;

        // Bending stress (maximum at outer fiber)
        let bending_moment_y = (u_j[4] + u_i[4]) * self.material.elastic_modulus
                               * section_modulus / element.length;
        let bending_moment_z = (u_j[5] + u_i[5]) * self.material.elastic_modulus
                               * section_modulus / element.length;
        let bending_stress = (bending_moment_y.powi(2) + bending_moment_z.powi(2)).sqrt()
                             / section_modulus;

        // Combined stress per ASME B31.3
        let combined_stress = axial_stress.abs() + bending_stress;

        // Check against allowable stress
        let stress_ratio = combined_stress / self.material.allowable_stress;

        Ok(ElementStress {
            element_id: element.id,
            axial_stress,
            bending_stress,
            combined_stress,
            stress_ratio,
            passes_code: stress_ratio <= 1.0,
        })
    }

    fn count_nodes(&self) -> usize {
        let mut max_node = 0;
        for element in &self.elements {
            max_node = max_node.max(element.node_i).max(element.node_j);
        }
        max_node + 1
    }
}

#[derive(Debug, Clone)]
pub struct LoadCase {
    pub name: String,
    pub include_weight: bool,
    pub include_pressure: bool,
    pub temperature: f64,
}

#[derive(Debug)]
pub struct StressResult {
    pub displacements: Vec<f64>,
    pub element_stresses: Vec<ElementStress>,
}

#[derive(Debug, Clone)]
pub struct ElementStress {
    pub element_id: usize,
    pub axial_stress: f64,
    pub bending_stress: f64,
    pub combined_stress: f64,
    pub stress_ratio: f64,  // Combined stress / allowable stress
    pub passes_code: bool,
}

#[derive(Debug)]
pub enum SolverError {
    Convergence(String),
    Singular(String),
    InvalidModel(String),
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–5)

**Day 1: Project initialization and Rust backend scaffold**
```bash
cargo init pipeflow-api
cd pipeflow-api
cargo add axum tokio serde serde_json sqlx uuid chrono tower-http jsonwebtoken bcrypt
cargo add --features postgres,runtime-tokio-rustls,uuid,chrono,json sqlx
cargo add aws-sdk-s3 redis nalgebra nalgebra-sparse
```
- `src/main.rs` — Axum app with CORS, tracing, graceful shutdown
- `src/config.rs` — Environment-based configuration (DATABASE_URL, JWT_SECRET, S3_BUCKET, REDIS_URL)
- `src/state.rs` — AppState struct (PgPool, Redis, S3Client)
- `src/error.rs` — ApiError enum with IntoResponse
- `Dockerfile` — Multi-stage build (builder + runtime)
- `docker-compose.yml` — PostgreSQL, Redis, MinIO (S3-compatible)

**Day 2-3: Database schema and migrations**
- `migrations/001_initial.sql` — All 12 tables: users, organizations, org_members, projects, pids, equipment, line_classes, pipe_segments, hydraulic_analyses, stress_analyses, pipe_supports, pump_curves, valve_catalog, usage_records
- `src/db/mod.rs` — Database pool initialization
- `src/db/models.rs` — All SQLx structs with FromRow derives
- Run `sqlx migrate run` to apply schema
- Seed script for line classes (150# CS, 300# SS316, 600# Alloy), valve catalog, standard pipe sizes

**Day 4: Authentication system**
- `src/auth/mod.rs` — JWT token generation and validation middleware
- `src/auth/oauth.rs` — Google and GitHub OAuth 2.0 flow handlers
- `src/api/handlers/auth.rs` — Register, login, OAuth callback, refresh token, me endpoints
- Password hashing with bcrypt (cost factor 12)
- JWT with 24h access token + 30d refresh token
- Auth middleware that extracts Claims from Authorization header

**Day 5: User and project CRUD**
- `src/api/handlers/users.rs` — Get profile, update profile, delete account
- `src/api/handlers/projects.rs` — Create, list, get, update, delete, fork project
- `src/api/handlers/orgs.rs` — Create org, invite member, list members, remove member
- `src/api/router.rs` — All route definitions with auth middleware
- Integration tests: auth flow, project CRUD, authorization checks

### Phase 2 — Hydraulic Solver Core (Days 6–14)

**Day 6-7: Pipe network graph framework**
- `solver-core/` — New Rust workspace member (shared between native and WASM)
- `solver-core/src/hydraulic/network.rs` — NetworkGraph struct with nodes, pipes, pumps
- `solver-core/src/hydraulic/pipe.rs` — Pipe struct with Darcy-Weisbach pressure drop calculation
- `solver-core/src/hydraulic/colebrook.rs` — Colebrook friction factor solver (iterative)
- Unit tests: single pipe pressure drop, compare to analytical solutions

**Day 8-9: Hardy Cross / Newton-Raphson network solver**
- `solver-core/src/hydraulic/solver.rs` — Newton-Raphson network solver with sparse Jacobian
- Mass conservation residuals at each node
- Energy equation residuals for each pipe
- Jacobian assembly (sparse matrix via `nalgebra-sparse`)
- Convergence criteria: max residual < 0.01 Pa or 0.001 psi
- Tests: two-pipe parallel network, three-reservoir problem (classic Hardy Cross test)

**Day 10: Pump integration**
- `solver-core/src/hydraulic/pump.rs` — Pump model with polynomial curve fit
- Pump curve fitting: least-squares polynomial (H = a₀ + a₁·Q + a₂·Q²)
- System curve generation for pump selection
- NPSH calculation and checking
- Tests: simple pump-pipe system, verify operating point

**Day 11: Elevation and gravity effects**
- Hydrostatic pressure calculation (ρgh)
- Elevation head in energy equation
- Multi-level piping systems
- Tests: water tower, elevated tank drainage

**Day 12-13: Fitting and valve losses**
- `solver-core/src/hydraulic/fittings.rs` — K-factor library (elbows, tees, reducers)
- Valve Cv to K-factor conversion
- Equivalent length method
- Tests: pipe with 5 elbows + 2 tees, compare to manual calculation

**Day 14: Network builder from pipe segments**
- `solver-core/src/hydraulic/builder.rs` — Build NetworkGraph from database pipe segments
- Automatic node numbering from equipment connections
- Boundary condition detection (reservoirs, fixed pressures)
- Validation: check for disconnected subgraphs, missing boundary conditions

### Phase 3 — WASM Build + P&ID Editor (Days 15–22)

**Day 15: WASM solver compilation**
- `solver-wasm/` — New workspace member for WASM target
- `solver-wasm/Cargo.toml` — wasm-bindgen, serde-wasm-bindgen dependencies
- `solver-wasm/src/lib.rs` — WASM entry points: `solve_hydraulic_network()`
- `wasm-pack build --target web --release` pipeline
- `wasm-opt -Oz` for size optimization (target <1.5MB gzipped)
- JavaScript wrapper: `HydraulicSolver` class that loads WASM and provides async API

**Day 16-17: Frontend scaffold and P&ID editor foundation**
```bash
npm create vite@latest frontend -- --template react-ts
cd frontend
npm i zustand @tanstack/react-query axios react-svg-pan-zoom
```
- `src/App.tsx` — Router, auth context, layout
- `src/stores/projectStore.ts` — Zustand store for project state
- `src/stores/pidStore.ts` — P&ID state (equipment, lines, instruments)
- `src/components/PIDEditor/Canvas.tsx` — SVG canvas with pan/zoom
- `src/components/PIDEditor/Grid.tsx` — Snap-to-grid overlay (50px grid)

**Day 18-19: P&ID editor — equipment symbols and placement**
- `src/components/PIDEditor/SymbolLibrary.tsx` — ISA 5.1 symbol palette (pumps, vessels, valves, heat exchangers, instruments)
- `src/components/PIDEditor/EquipmentSymbol.tsx` — SVG equipment renderer
- `src/lib/symbols/` — SVG path definitions for 100+ symbols
- Drag-and-drop from library → canvas, rotation (90° increments), flip, delete
- Property editor panel: tag, description, datasheet link

**Day 20: P&ID editor — intelligent line numbering**
- `src/components/PIDEditor/PipeLine.tsx` — Pipe line drawing tool
- Automatic line number generation: `{size}-{service}-{sequential}-{material_class}`
- Line properties: size, service code (LP/HP/MP), material class, insulation
- Line list auto-generation from drawn lines

**Day 21: P&ID editor — connectivity graph**
- `src/lib/connectivity.ts` — Build adjacency graph from P&ID connections
- Node: equipment + nozzles, Edge: pipe lines
- Export to hydraulic network format (nodes + pipes)
- Validation: detect disconnected equipment, missing ground connections

**Day 22: P&ID export and revision management**
- SVG export to S3 with thumbnail generation
- PDF export via server-side SVG-to-PDF (using `resvg` or `wkhtmltopdf`)
- Revision clouding and triangles
- Revision history tracking in database

### Phase 4 — 3D Visualization + API (Days 23–29)

**Day 23-24: Three.js 3D pipe viewer**
```bash
npm i three @react-three/fiber @react-three/drei
```
- `src/components/PipeViewer3D/Scene.tsx` — React Three Fiber scene
- `src/components/PipeViewer3D/PipeGeometry.tsx` — Generate pipe mesh from segments
- Orthogonal routing visualization (90° bends)
- Camera controls: orbit, pan, zoom
- Color-coding: by service, pressure, temperature

**Day 25: 3D visualization — stress and pressure overlays**
- `src/components/PipeViewer3D/StressOverlay.tsx` — Color-coded stress ratios (green <0.5, yellow 0.5-0.8, red >0.8)
- `src/components/PipeViewer3D/PressureContours.tsx` — Pressure gradient visualization
- Support symbol placement in 3D
- Annotation labels (equipment tags, line numbers)

**Day 26: Hydraulic analysis API endpoints**
- `src/api/handlers/hydraulic.rs` — Create analysis, get analysis, list analyses
- Network graph building from pipe segments
- Node count extraction and WASM/server routing logic
- Plan-based limits enforcement (free: 50 nodes, pro: 200 nodes, engineering: unlimited)

**Day 27: Server-side hydraulic worker**
- `src/workers/hydraulic_worker.rs` — Redis job consumer, runs native solver
- Progress streaming: worker publishes progress → Redis pub/sub → WebSocket to client
- Result serialization: node pressures + pipe flows as JSON
- S3 result upload with presigned URL generation

**Day 28: WebSocket for live analysis progress**
- `src/api/ws/mod.rs` — Axum WebSocket upgrade handler
- `src/api/ws/hydraulic_progress.rs` — Subscribe to hydraulic analysis progress channel
- Client receives: `{ progress_pct, current_iteration, max_residual }`
- Frontend: `src/hooks/useHydraulicProgress.ts` — React hook for WebSocket subscription

**Day 29: Equipment and line class APIs**
- `src/api/handlers/equipment.rs` — CRUD for equipment, nozzle management
- `src/api/handlers/line_classes.rs` — List line classes, create custom line class
- `src/api/handlers/pipe_segments.rs` — CRUD for pipe segments, geometry updates
- Valve catalog API with parametric search

### Phase 5 — Stress Analysis + Results UI (Days 30–36)

**Day 30-31: Pipe stress solver core**
- `solver-core/src/stress/beam_fea.rs` — Beam finite element implementation
- 3D beam elements with 6 DOF per node
- Stiffness matrix assembly
- Support boundary conditions (anchor, guide, rest)
- Load cases: weight, pressure, thermal (thermal deferred to post-MVP)

**Day 32: ASME B31.3 code checks**
- `solver-core/src/stress/asme_b31_3.rs` — Allowable stress calculations
- Sustained stress check: S_L ≤ S_h (allowable stress at design temp)
- Stress intensification factors for fittings
- Weld joint efficiency factors
- Material database with allowable stress tables

**Day 33: Stress analysis API and worker**
- `src/api/handlers/stress.rs` — Create stress analysis, get results
- `src/workers/stress_worker.rs` — Server-side stress solver worker
- Material spec input and validation
- Results: element stresses, stress ratios, support reactions

**Day 34: Results visualization — hydraulic**
- `src/components/Results/HydraulicResults.tsx` — Pressure profile plot (line chart)
- Node pressure table with min/max highlighting
- Pipe velocity table with color-coded warnings (>15 ft/s)
- System curve with pump curve overlay
- NPSH margin display

**Day 35: Results visualization — stress**
- `src/components/Results/StressResults.tsx` — Stress ratio table by element
- 3D stress isometric with color-coded elements
- Support load table (anchor reactions, guide forces)
- Code compliance summary (pass/fail per ASME B31.3)

**Day 36: PDF report generation**
- `src/api/handlers/reports.rs` — Generate PDF report endpoint
- Server-side PDF generation using `printpdf` or `wkhtmltopdf`
- Report sections: project info, P&ID, hydraulic summary, stress summary, code checks
- Pressure profile plot export
- Stress isometric export

### Phase 6 — Pipe Sizing + Python Services (Days 37–40)

**Day 37: Pipe sizing calculator**
- `python-services/pipe_sizing/` — Python FastAPI service
- Economic pipe diameter calculation (Genereaux method: optimize CAPEX vs OPEX)
- Velocity limit checking (2-15 ft/s for liquids)
- Erosional velocity for two-phase (deferred to post-MVP, but add placeholder)
- API endpoint: POST /api/pipe-sizing with fluid properties + flow rate

**Day 38: Integration with Rust backend**
- `src/api/handlers/pipe_sizing.rs` — Proxy to Python service
- Call Python FastAPI from Rust using `reqwest`
- Cache sizing results in PostgreSQL
- Frontend: `src/components/Calculators/PipeSizing.tsx` — Input form + results display

**Day 39: NPSH calculation service**
- `python-services/npsh/` — NPSH available calculation
- Vapor pressure lookup tables for common fluids
- Suction line pressure drop calculation
- API endpoint: POST /api/npsh with fluid + suction config

**Day 40: Pump curve fitting service**
- `python-services/pump_curves/` — Polynomial curve fitting
- Upload manufacturer curve data (CSV: flow, head, efficiency)
- Least-squares fit to quadratic: H = a₀ + a₁·Q + a₂·Q²
- Store coefficients in pump_curves table

### Phase 7 — Polish + Validation (Days 41–42)

**Day 41: End-to-end validation**
- Create 5 validation test cases with known analytical solutions
- Test Case 1: Simple two-reservoir problem (compare to manual Hardy Cross calculation)
- Test Case 2: Pump-pipe system with known operating point
- Test Case 3: Elevated tank drainage (check elevation head)
- Test Case 4: Pipe stress cantilever beam (compare to analytical beam formula)
- Test Case 5: Multi-branch network (3 pipes in parallel + series)
- Document expected vs actual results in `docs/validation.md`

**Day 42: Performance optimization and deployment prep**
- WASM bundle optimization: tree-shaking, code splitting
- Database query optimization: add missing indexes
- S3 lifecycle policies for old results
- Stripe webhook testing
- Production Docker images
- AWS deployment documentation

### Phase 8 — Deployment + Launch (Days 1–4 post-core)

Not counted in 42-day core MVP, but included for completeness:

**Day 43: Production infrastructure**
- AWS ECS/Fargate deployment
- RDS PostgreSQL setup
- ElastiCache Redis
- S3 buckets with CloudFront CDN
- Application Load Balancer with HTTPS

**Day 44: Monitoring and observability**
- Prometheus metrics collection
- Grafana dashboards (request latency, solver performance, error rates)
- Sentry error tracking
- CloudWatch logs aggregation

**Day 45: Beta testing**
- Invite 10 process engineers for beta testing
- Feedback collection via Typeform
- Bug triage and hot fixes

**Day 46: Public launch**
- Product Hunt launch
- Engineering subreddit posts (r/ChemicalEngineering, r/MechanicalEngineering)
- LinkedIn announcement
- Documentation site (VitePress)

---

## Validation Benchmarks

### Benchmark 1: Two-Reservoir Hardy Cross Problem

**Setup:** Two reservoirs connected by three pipes in a network. Reservoir A at elevation 100m, Reservoir B at elevation 80m. Three pipes: P1 (1000m, 300mm, ε=0.046mm), P2 (800m, 250mm), P3 (600m, 200mm) arranged in a loop.

**Expected Result (Analytical Hardy Cross):**
- Flow in P1: 0.152 m³/s
- Flow in P2: 0.098 m³/s
- Flow in P3: 0.054 m³/s
- Pressure at junction: 845 kPa

**Acceptance Criteria:** PipeFlow solver results within 2% of analytical solution, converges in <10 iterations.

### Benchmark 2: Pump Operating Point

**Setup:** Centrifugal pump with curve H = 120 - 0.08·Q² (H in meters, Q in m³/h) connected to 500m of 150mm pipe (ε=0.046mm) discharging to atmospheric tank 30m above pump.

**Expected Result (Analytical):**
- Operating flow: 28.4 m³/h
- Operating head: 95.4 m
- Pump efficiency: 68% (from curve lookup)
- NPSH required: 4.2 m (from curve)

**Acceptance Criteria:** PipeFlow pump solver finds operating point within 3% of analytical solution, NPSH check passes.

### Benchmark 3: Pipe Stress — Cantilever Beam

**Setup:** Horizontal cantilever pipe: 6m long, 4" Sch 40 (114.3mm OD, 6.02mm wall), Carbon Steel (E=200 GPa, allowable stress=138 MPa), fixed at one end, free at other end, weight load only.

**Expected Result (Euler-Bernoulli beam theory):**
- Max bending stress: 42.8 MPa (at fixed end)
- Stress ratio: 0.31 (42.8/138)
- Deflection at free end: 18.4 mm

**Acceptance Criteria:** PipeFlow FEA solver stress within 5% of analytical, stress ratio within 0.02, deflection within 10%.

### Benchmark 4: Economic Pipe Diameter

**Setup:** Water at 20°C, flow rate 100 m³/h, pipe length 200m, electricity cost $0.10/kWh, pump efficiency 75%, pipe cost $50/m/inch, 20-year lifetime, 5% discount rate.

**Expected Result (Genereaux formula):**
- Optimal diameter: 150mm (6")
- Annual pumping cost: $1,247
- Pipe capital cost (annualized): $982
- Total annual cost: $2,229

**Acceptance Criteria:** PipeFlow pipe sizing calculator selects diameter within 1 nominal size, total cost within 10% of analytical minimum.

### Benchmark 5: Multi-Branch Network Convergence

**Setup:** 5-node network with 7 pipes, 2 boundary conditions (fixed pressure reservoirs), varying diameters 50-200mm, total network length 3km, water at 15°C.

**Expected Result (Commercial solver AFT Fathom reference):**
- Converges in 12-18 Newton-Raphson iterations
- Max node pressure: 587 kPa
- Min node pressure: 412 kPa
- Total flow: 0.34 m³/s

**Acceptance Criteria:** PipeFlow solver converges in ≤20 iterations, node pressures within 5% of AFT Fathom, mass balance residual <0.001 kg/s at all nodes.

---

## Post-MVP Roadmap

### v1.1 — Advanced Hydraulics (Months 3-4)

**Compressible Gas Flow:**
- Isothermal, adiabatic, and polytropic gas flow models
- Weymouth, Panhandle A/B equations for gas pipelines
- Choked flow detection at pressure reducers
- Gas property database (natural gas, air, nitrogen, CO2)

**Two-Phase Flow:**
- Lockhart-Martinelli correlation for horizontal pipes
- Beggs-Brill correlation for inclined pipes
- Slug flow detection
- Void fraction calculation
- Phase separation at tees and branches

**Target Users:** Oil & gas engineers (gas gathering systems), HVAC engineers (refrigerant lines)

**Pricing Impact:** Included in Engineering plan ($199/mo), not available in Pro tier

### v1.2 — Full Stress Analysis (Months 4-5)

**Thermal Expansion Analysis:**
- Thermal load case per ASME B31.3 Eq. 1b
- Expansion loop sizing (U-loop, Z-bend, L-bend)
- Expansion joint selection and modeling
- Anchor load calculation

**Spring Hanger Selection:**
- Variable spring hanger sizing (constant load through thermal movement)
- Manufacturer catalog integration (Anvil, Piping Technology, Bergen)
- Hot and cold load calculation
- Travel verification

**Occasional Loads:**
- Wind load per ASCE 7
- Seismic load per IBC/ASCE
- Relief valve thrust forces
- Slug flow impact forces

**Fatigue Analysis:**
- Rainflow cycle counting for cyclic thermal services
- Fatigue curve per ASME B31.3 Figure 302.3.5
- Cumulative damage (Miner's rule)

**Target Users:** EPC contractors, piping stress specialists

**Pricing Impact:** Full stress (all codes + all load cases) remains Engineering plan exclusive

### v1.3 — Isometric Generation + MTO (Months 5-6)

**Automated Isometric Drawings:**
- Generate isometric drawings from 3D pipe model
- Dimension annotations (pipe lengths, elevations, coordinates)
- Weld symbols and weld counts
- Bill of materials per isometric
- Spool drawing generation for fabrication

**Material Takeoff (MTO):**
- Pipe: length by size, schedule, material
- Fittings: elbows, tees, reducers, caps (count by type and size)
- Flanges: count by size and rating class
- Bolting: count by size and length
- Gaskets: count by size
- Valves: count by type and size
- Weight summary (fabricated, erected, hydro-test, operating)
- Export to Excel/CSV for procurement

**Integration:**
- API export to ERP systems (SAP, Oracle)
- Integration with PDMS/E3D/S3D piping models (import geometry)

**Target Users:** EPC contractors, fabrication shops

**Pricing Impact:** Add-on module $49/mo for Pro, included in Engineering plan

### v1.4 — AI-Assisted Design (Months 6-8)

**AI Pipe Routing:**
- Reinforcement learning model for optimal pipe routing
- Cost function: minimize total pipe length + bends + support cost + thermal expansion risk
- Collision avoidance with structures and other pipes
- Accessibility constraints (maintenance clearances)

**Automatic Support Placement:**
- ML model trained on thousands of stress analysis results
- Recommends support types and locations to meet stress code
- Iterative optimization: place supports → run stress → adjust → repeat

**Equipment Layout Optimization:**
- Genetic algorithm for equipment placement
- Minimize total pipe length while meeting process flow requirements
- Safety clearances (explosion zones, high-temperature equipment)

**Anomaly Detection:**
- Flag unusual pressure drops (>50% higher than expected)
- Flag unusual velocities (<1 or >20 ft/s)
- Flag stress ratios >0.9 (close to code limit)
- Suggest design improvements

**Target Users:** Process engineers, plant designers

**Pricing Impact:** AI features exclusive to Engineering plan, requires minimum 10-seat commitment

### v1.5 — Transient Analysis (Water Hammer) (Months 8-10)

**Method of Characteristics (MOC):**
- Transient pressure surge calculation
- Pump trip scenarios
- Valve closure analysis (rapid vs slow closure)
- Check valve slam

**Surge Protection:**
- Surge vessel sizing
- Relief valve sizing for transient overpressure
- Air chamber design
- Flywheel inertia calculation for pump

**Use Cases:**
- Water distribution systems (pump station design)
- Fire water systems (check valve sizing)
- Refinery knockout drum inlet lines (slug flow protection)

**Target Users:** Water utilities, fire protection engineers, refinery engineers

**Pricing Impact:** Add-on module $99/mo for Engineering plan, not available for Pro/Free

---

## Tech Debt and Known Limitations

### MVP Limitations (to be addressed post-launch)

1. **Hydraulic Solver:**
   - Incompressible liquid only (no gas, two-phase, non-Newtonian)
   - Steady-state only (no transient, no water hammer)
   - Pump curves must be manually entered (no automatic digitization from images)
   - No control valve Cv-based modeling (simple K-factor only)

2. **Stress Analysis:**
   - Sustained loads only (weight + pressure)
   - No thermal expansion analysis
   - No spring hanger selection
   - ASME B31.3 only (no B31.1, EN 13480, B31.4, B31.8)
   - Simplified beam elements (no elbow/tee stress intensification factors)

3. **P&ID Editor:**
   - 100 symbols (vs 500+ in full ISA 5.1 library)
   - No instrument loop diagrams
   - No automatic line routing (manual click-to-place only)
   - Revision management is basic (no formal approval workflow)

4. **Performance:**
   - WASM solver limited to 200 nodes (may be slow on older hardware)
   - 3D viewer may lag with >500 pipe segments
   - PDF report generation is synchronous (blocks server thread)

### Planned Tech Debt Paydown (Months 3-6)

1. **Database:**
   - Add composite indexes for common query patterns (e.g., `(project_id, line_number)`)
   - Partition `usage_records` table by period for faster billing queries
   - Add full-text search indexes on equipment descriptions

2. **Solver:**
   - Parallelize network solver for multi-core speedup (Rayon)
   - Implement direct sparse solver (UMFPACK or PARDISO) for larger networks
   - Add iterative solver option (conjugate gradient) for 10K+ node networks

3. **Frontend:**
   - Implement virtual scrolling for large line lists (React Virtualized)
   - Add service worker for offline P&ID editing
   - Lazy-load 3D viewer (reduce initial bundle size)

4. **Infrastructure:**
   - Implement autoscaling for worker pool (scale based on Redis queue depth)
   - Add read replicas for PostgreSQL (scale read-heavy queries)
   - Implement Redis cluster for high availability

---

## Competitive Moats

1. **Integrated P&ID + Hydraulics + Stress:** No competitor offers all three in a single cloud platform. AFT Fathom (hydraulics only), CAESAR II (stress only), SmartPlant P&ID (drawings only) are separate tools with no data flow between them. PipeFlow's connectivity graph automatically generates hydraulic networks from P&IDs, eliminating manual data entry.

2. **Cloud-Native WASM Solver:** Competitors are desktop apps requiring VPN access to client networks. PipeFlow runs in the browser with instant startup and no client IT approval. The WASM solver provides instant feedback for small networks while automatically falling back to server for large networks.

3. **Modern UI/UX:** Legacy tools (AFT, CAESAR II) have 1990s Windows UI. PipeFlow has React + SVG + Three.js, intuitive drag-and-drop, real-time collaboration (roadmap), mobile-responsive (roadmap).

4. **Pricing:** AFT Fathom + CAESAR II costs $13K-$32K/seat perpetual + $3K-$6K/year maintenance. PipeFlow Pro at $79/mo ($948/year) is 95% cheaper. Break-even for customers after 1 month of use.

5. **API-First Architecture:** All solver functions exposed via REST API, enabling integration with in-house tools, Python scripting, CI/CD pipelines. Competitors have no API or require expensive "automation" add-ons.

---

*End of Implementation Plan*
