# 21. CircuitMind — AI-Assisted PCB Design and Schematic Editor in the Browser

## Implementation Plan

**MVP Scope:** Browser-based schematic capture editor with drag-and-drop component placement (resistors, capacitors, inductors, diodes, MOSFETs, BJTs, op-amps, microcontrollers, connectors) and interactive wire routing rendered via SVG with R-tree spatial indexing, multi-layer PCB layout editor (up to 4 layers) rendered via WebGL with GPU-accelerated real-time DRC checking for clearance, trace width, annular ring, and drill violations, manual interactive routing with push-and-shove algorithm and snap-to-grid/pad/track alignment, comprehensive component library of 50,000+ verified symbols and footprints with DigiKey API integration for real-time pricing and stock status, BOM generator with multi-supplier cost optimization across DigiKey/Mouser/LCSC, Gerber RS-274X export with auto-populated layer mapping and Excellon drill files, KiCad 7/8 project import with automatic library migration, project version control with snapshot-based revision history, Stripe billing with three tiers (Free / Pro $49/mo / Team $149/mo base + $29/seat).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| DRC Engine | Rust (native + WASM) | GPU-accelerated via compute shaders for large boards, WASM for client-side incremental checks |
| WASM Compilation | `wasm-pack` + `wasm-bindgen` | Client-side DRC for real-time feedback, geometry algorithms |
| Auto-Router Worker | Python 3.12 (FastAPI) | ML model for topology optimization, A* pathfinding, GPU instances |
| Database | PostgreSQL 16 + PostGIS | Projects, users, components, spatial queries on board geometry |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | Gerber files, component libraries, STEP models, project snapshots |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| Schematic Editor | Custom SVG renderer | R-tree spatial indexing (`rbush`), snap-to-grid, auto-wire routing |
| PCB Renderer | WebGL 2.0 (custom) | Layer compositing, anti-aliased traces, copper pour rendering |
| 3D Viewer | Three.js + OpenCascade.js | STEP model import, board preview, enclosure fit check |
| Collaboration | Yjs CRDT | WebSocket provider, conflict-free multi-user editing |
| Real-time | WebSocket (Axum) | Live DRC updates, auto-route progress, cursor presence |
| Job Queue | Redis 7 + Tokio tasks | Auto-routing job distribution, DRC job scheduling |
| Search | Meilisearch | Component library parametric and full-text search |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | WASM bundle delivery, static frontend assets |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, error tracking |
| CI/CD | GitHub Actions | WASM build pipeline, Rust tests, Docker image push |

### Key Architecture Decisions

1. **Hybrid client/server DRC with WASM for real-time feedback**: Local traces and component placement trigger client-side WASM DRC checks for clearance and width violations, providing <100ms feedback during interactive routing. Full-board DRC for large designs (1000+ components) runs server-side on GPU compute instances, processing polygon intersections via spatial acceleration structures. Free tier limited to client-side DRC only.

2. **Separate Python auto-router service with GPU-based ML topology optimization**: The AI auto-router uses a hybrid approach: Python FastAPI service runs ML models (PyTorch-based graph neural network) on Lambda Cloud GPU instances to optimize component placement and net ordering, then invokes Rust pathfinding core for detailed A* routing with push-and-shove. This separation keeps the Rust backend lightweight while allowing GPU scaling for complex boards.

3. **SVG schematic editor with R-tree spatial indexing**: SVG rendering provides crisp zoom at any scale and native hit-testing for component selection. The `rbush` R-tree library enables O(log n) spatial queries for auto-wire routing, snap detection, and collision avoidance even with 500+ components per sheet. Wire routing uses Manhattan geometry with automatic junction creation.

4. **WebGL PCB renderer with layer compositing and transparency blending**: Each copper layer, silk layer, and solder mask renders to a separate framebuffer, then composited with configurable transparency and blend modes. Trace rendering uses instanced line segments with anti-aliasing via signed distance fields. Copper pours use polygon tessellation and GPU fill. This architecture handles 2000+ trace segments at 60fps.

5. **PostGIS for spatial queries on PCB geometry**: Board outlines, component courtyard polygons, and keepout zones are stored as PostGIS geometry types, enabling efficient spatial queries for placement validation (e.g., "find all components within 5mm of board edge") and automated constraint checking for mechanical assembly.

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "postgis";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";  -- For fuzzy text search

-- Users
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email TEXT UNIQUE NOT NULL,
    password_hash TEXT,  -- NULL for OAuth-only users
    name TEXT NOT NULL,
    avatar_url TEXT,
    auth_provider TEXT DEFAULT 'email',  -- email | google | github
    auth_provider_id TEXT,
    plan TEXT NOT NULL DEFAULT 'free',  -- free | pro | team
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX users_email_idx ON users(email);
CREATE INDEX users_stripe_idx ON users(stripe_customer_id);

-- Organizations (for Team plan)
CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    slug TEXT UNIQUE NOT NULL,
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    plan TEXT NOT NULL DEFAULT 'free',
    seat_count INTEGER DEFAULT 1,
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

-- Projects (schematic + PCB workspace)
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    schematic_data JSONB NOT NULL DEFAULT '{}',  -- Full schematic state (components, wires, labels, sheets)
    pcb_data JSONB NOT NULL DEFAULT '{}',  -- PCB layout state (traces, vias, zones, dimensions)
    board_params JSONB DEFAULT '{}',  -- Layer count, width, height, stackup definition
    visibility TEXT NOT NULL DEFAULT 'private',  -- private | org | public
    forked_from UUID REFERENCES projects(id) ON DELETE SET NULL,
    thumbnail_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);
CREATE INDEX projects_org_idx ON projects(org_id);
CREATE INDEX projects_updated_idx ON projects(updated_at DESC);
CREATE INDEX projects_visibility_idx ON projects(visibility) WHERE visibility = 'public';

-- Revisions (version control snapshots)
CREATE TABLE revisions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    parent_revision_id UUID REFERENCES revisions(id) ON DELETE SET NULL,
    snapshot_url TEXT NOT NULL,  -- S3 URL to full project state snapshot
    message TEXT NOT NULL,
    tag TEXT,  -- Nullable tag for releases (e.g., "v1.0-prototype")
    created_by UUID NOT NULL REFERENCES users(id) ON DELETE SET NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX revisions_project_idx ON revisions(project_id);
CREATE INDEX revisions_created_idx ON revisions(created_at DESC);
CREATE INDEX revisions_tag_idx ON revisions(tag) WHERE tag IS NOT NULL;

-- Component Library
CREATE TABLE components (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    description TEXT,
    manufacturer TEXT,
    mpn TEXT,  -- Manufacturer part number
    category TEXT NOT NULL,  -- resistor | capacitor | inductor | diode | mosfet | bjt | ic | connector | mechanical
    subcategory TEXT,
    package_type TEXT,  -- e.g., "0805", "SOIC-16", "QFP-64"
    symbol_data JSONB NOT NULL,  -- SVG symbol definition
    footprint_data JSONB NOT NULL,  -- Pad coordinates, outline, courtyard polygon
    datasheet_url TEXT,
    spice_model_url TEXT,
    step_model_url TEXT,
    tags TEXT[] DEFAULT '{}',
    is_verified BOOLEAN DEFAULT false,
    is_builtin BOOLEAN DEFAULT true,
    created_by UUID REFERENCES users(id),
    verification_count INTEGER DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX components_category_idx ON components(category);
CREATE INDEX components_manufacturer_idx ON components(manufacturer);
CREATE INDEX components_name_trgm_idx ON components USING gin(name gin_trgm_ops);
CREATE INDEX components_mpn_trgm_idx ON components USING gin(mpn gin_trgm_ops);
CREATE INDEX components_tags_idx ON components USING gin(tags);

-- Component Pricing (cached from supplier APIs)
CREATE TABLE component_pricing (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    component_id UUID NOT NULL REFERENCES components(id) ON DELETE CASCADE,
    supplier TEXT NOT NULL,  -- digikey | mouser | lcsc | farnell | arrow
    supplier_sku TEXT NOT NULL,
    unit_price DECIMAL(10, 4) NOT NULL,
    moq INTEGER DEFAULT 1,
    stock_quantity INTEGER,
    lifecycle_status TEXT DEFAULT 'active',  -- active | nrnd | obsolete | eol
    last_updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(component_id, supplier, supplier_sku)
);
CREATE INDEX pricing_component_idx ON component_pricing(component_id);
CREATE INDEX pricing_supplier_idx ON component_pricing(supplier);
CREATE INDEX pricing_lifecycle_idx ON component_pricing(lifecycle_status);

-- DRC Violations (computed, stored per DRC run)
CREATE TABLE drc_violations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    revision_id UUID REFERENCES revisions(id) ON DELETE CASCADE,
    rule_name TEXT NOT NULL,  -- e.g., "clearance", "trace_width", "annular_ring"
    severity TEXT NOT NULL DEFAULT 'error',  -- error | warning | info
    description TEXT NOT NULL,
    location GEOMETRY(Point, 4326),  -- PostGIS point for violation location
    layer TEXT,
    affected_nets TEXT[] DEFAULT '{}',
    affected_components TEXT[] DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX drc_project_idx ON drc_violations(project_id);
CREATE INDEX drc_revision_idx ON drc_violations(revision_id);
CREATE INDEX drc_severity_idx ON drc_violations(severity);
CREATE INDEX drc_location_idx ON drc_violations USING gist(location);

-- Auto-Route Jobs
CREATE TABLE autoroute_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | running | completed | failed | cancelled
    parameters JSONB NOT NULL DEFAULT '{}',  -- Net classes, constraints, region selection
    worker_id TEXT,
    progress_pct REAL DEFAULT 0.0,
    result_data JSONB,  -- Route statistics, quality score, DRC summary
    result_url TEXT,  -- S3 URL to detailed route solution
    error_message TEXT,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    wall_time_ms INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX autoroute_project_idx ON autoroute_jobs(project_id);
CREATE INDEX autoroute_user_idx ON autoroute_jobs(user_id);
CREATE INDEX autoroute_status_idx ON autoroute_jobs(status);
CREATE INDEX autoroute_created_idx ON autoroute_jobs(created_at DESC);

-- Comments (design review annotations)
CREATE TABLE comments (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    author_id UUID NOT NULL REFERENCES users(id) ON DELETE SET NULL,
    body TEXT NOT NULL,
    resolved BOOLEAN DEFAULT false,
    anchor_type TEXT NOT NULL,  -- component | trace | region | general
    anchor_data JSONB DEFAULT '{}',  -- Coordinates, ref_des, layer
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    resolved_at TIMESTAMPTZ,
    resolved_by UUID REFERENCES users(id) ON DELETE SET NULL
);
CREATE INDEX comments_project_idx ON comments(project_id);
CREATE INDEX comments_author_idx ON comments(author_id);
CREATE INDEX comments_resolved_idx ON comments(resolved);

-- Design Rule Sets
CREATE TABLE design_rule_sets (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    description TEXT,
    org_id UUID REFERENCES organizations(id) ON DELETE CASCADE,
    rules JSONB NOT NULL,  -- Clearance, trace width, via drill, annular ring, etc.
    based_on UUID REFERENCES design_rule_sets(id) ON DELETE SET NULL,
    is_system_preset BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX drs_org_idx ON design_rule_sets(org_id);

-- Usage Tracking (for billing enforcement)
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,  -- autoroute_job | drc_run | cloud_storage_mb
    quantity REAL NOT NULL,
    period_start DATE NOT NULL,
    period_end DATE NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX usage_user_period_idx ON usage_records(user_id, period_start, period_end);
CREATE INDEX usage_org_period_idx ON usage_records(org_id, period_start, period_end);
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
pub struct Organization {
    pub id: Uuid,
    pub name: String,
    pub slug: String,
    pub owner_id: Uuid,
    pub plan: String,
    pub seat_count: i32,
    pub stripe_customer_id: Option<String>,
    pub stripe_subscription_id: Option<String>,
    pub settings: serde_json::Value,
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
    pub schematic_data: serde_json::Value,
    pub pcb_data: serde_json::Value,
    pub board_params: serde_json::Value,
    pub visibility: String,
    pub forked_from: Option<Uuid>,
    pub thumbnail_url: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Revision {
    pub id: Uuid,
    pub project_id: Uuid,
    pub parent_revision_id: Option<Uuid>,
    pub snapshot_url: String,
    pub message: String,
    pub tag: Option<String>,
    pub created_by: Uuid,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Component {
    pub id: Uuid,
    pub name: String,
    pub description: Option<String>,
    pub manufacturer: Option<String>,
    pub mpn: Option<String>,
    pub category: String,
    pub subcategory: Option<String>,
    pub package_type: Option<String>,
    pub symbol_data: serde_json::Value,
    pub footprint_data: serde_json::Value,
    pub datasheet_url: Option<String>,
    pub spice_model_url: Option<String>,
    pub step_model_url: Option<String>,
    pub tags: Vec<String>,
    pub is_verified: bool,
    pub is_builtin: bool,
    pub created_by: Option<Uuid>,
    pub verification_count: i32,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct ComponentPricing {
    pub id: Uuid,
    pub component_id: Uuid,
    pub supplier: String,
    pub supplier_sku: String,
    pub unit_price: rust_decimal::Decimal,
    pub moq: i32,
    pub stock_quantity: Option<i32>,
    pub lifecycle_status: String,
    pub last_updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct AutorouteJob {
    pub id: Uuid,
    pub project_id: Uuid,
    pub user_id: Uuid,
    pub status: String,
    pub parameters: serde_json::Value,
    pub worker_id: Option<String>,
    pub progress_pct: f32,
    pub result_data: Option<serde_json::Value>,
    pub result_url: Option<String>,
    pub error_message: Option<String>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub wall_time_ms: Option<i32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct DrcViolation {
    pub id: Uuid,
    pub project_id: Uuid,
    pub revision_id: Option<Uuid>,
    pub rule_name: String,
    pub severity: String,
    pub description: String,
    #[sqlx(skip)]
    pub location: Option<geo_types::Point<f64>>,
    pub layer: Option<String>,
    pub affected_nets: Vec<String>,
    pub affected_components: Vec<String>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct AutorouteParams {
    pub net_classes: Vec<NetClass>,
    pub region: Option<RoutingRegion>,
    pub preserve_existing: bool,
    pub strategy: String,  // "fast" | "balanced" | "optimize"
}

#[derive(Debug, Deserialize, Serialize)]
pub struct NetClass {
    pub name: String,
    pub nets: Vec<String>,
    pub trace_width_mm: f64,
    pub clearance_mm: f64,
    pub via_drill_mm: f64,
    pub via_diameter_mm: f64,
    pub diff_pair_gap_mm: Option<f64>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct RoutingRegion {
    pub x_min_mm: f64,
    pub y_min_mm: f64,
    pub x_max_mm: f64,
    pub y_max_mm: f64,
    pub layers: Vec<String>,
}
```

---

## DRC Engine Architecture

### Governing Rules and Geometry Algorithms

The DRC engine implements 10 core rule categories for PCB manufacturing compliance:

**1. Clearance:** Minimum spacing between copper features (traces, pads, vias, zones) on the same layer.
**2. Trace Width:** Minimum and maximum trace width enforcement for manufacturability and current capacity.
**3. Annular Ring:** Minimum copper ring around drilled holes (vias, through-hole pads) to ensure reliable plating.
**4. Drill Size:** Minimum drill diameter and aspect ratio limits for fabrication capability.
**5. Courtyard Overlap:** Component courtyard polygons must not overlap (placement conflict detection).
**6. Silk-to-Pad:** Silkscreen must not overlap exposed copper pads.
**7. Solder Mask Sliver:** Minimum solder mask web between pads to prevent mask bridging.
**8. Board Edge Clearance:** Minimum distance from copper features to board outline.
**9. Via-in-Pad:** Optional rule to flag or allow vias inside SMD pads (affects assembly).
**10. Differential Pair Matching:** Trace length and spacing tolerance for differential pairs.

### Client/Server Split (DRC Threshold)

```
DRC request → Board feature count extracted
    │
    ├── <500 features → WASM DRC (browser)
    │   ├── Incremental checks during routing
    │   ├── Results displayed in <100ms
    │   └── No server cost
    │
    └── ≥500 features OR full-board → Server DRC (Rust native + GPU)
        ├── Job queued via Redis
        ├── GPU compute for polygon intersections
        ├── Spatial index (R-tree) acceleration
        └── Results stored in DB, violations returned via WebSocket
```

The 500-feature threshold was chosen because:
- WASM DRC handles typical 2-layer boards with 50-200 components in <100ms
- Client-side DRC provides real-time feedback during interactive routing
- Above 500 features: 4-layer boards with dense BGA routing require GPU acceleration and spatial indices

### WASM Compilation Pipeline

```toml
# drc-wasm/Cargo.toml
[package]
name = "circuitmind-drc-wasm"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
wasm-bindgen = "0.2"
serde = { version = "1", features = ["derive"] }
serde-wasm-bindgen = "0.6"
js-sys = "0.3"
geo = "0.28"  # Geometry primitives and operations
rstar = "0.12"  # R-tree spatial index
log = "0.4"
console_log = "1"

[profile.release]
opt-level = "z"       # Optimize for size
lto = true            # Link-time optimization
codegen-units = 1     # Single codegen unit
strip = true          # Strip debug symbols
```

---

## Architecture Deep-Dives

### 1. Auto-Router API Handler (Rust/Axum)

The auto-routing endpoint receives a routing request, validates net classes and constraints, enqueues a job with Redis, and streams progress via WebSocket.

```rust
// src/api/handlers/autoroute.rs

use axum::{
    extract::{Path, State, Json},
    http::StatusCode,
    response::IntoResponse,
};
use uuid::Uuid;
use crate::{
    db::models::{AutorouteJob, AutorouteParams},
    state::AppState,
    auth::Claims,
    error::ApiError,
};

#[derive(serde::Deserialize)]
pub struct CreateAutorouteRequest {
    pub parameters: AutorouteParams,
}

pub async fn create_autoroute_job(
    State(state): State<AppState>,
    claims: Claims,
    Path(project_id): Path<Uuid>,
    Json(req): Json<CreateAutorouteRequest>,
) -> Result<impl IntoResponse, ApiError> {
    // 1. Verify project ownership and plan limits
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

    let user = sqlx::query_as!(
        crate::db::models::User,
        "SELECT * FROM users WHERE id = $1",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    // Check usage quota for current month
    let current_month_start = chrono::Utc::now().date_naive().with_day(1).unwrap();
    let usage_count: i64 = sqlx::query_scalar!(
        "SELECT COUNT(*) FROM autoroute_jobs
         WHERE user_id = $1 AND created_at >= $2 AND status = 'completed'",
        claims.user_id,
        current_month_start.and_hms_opt(0, 0, 0).unwrap()
    )
    .fetch_one(&state.db)
    .await?
    .unwrap_or(0);

    let quota = match user.plan.as_str() {
        "free" => return Err(ApiError::PlanLimit(
            "Auto-routing is not available on the Free plan. Upgrade to Pro for 50 routing jobs per month."
        )),
        "pro" => 50,
        "team" => 200,
        _ => i64::MAX,
    };

    if usage_count >= quota {
        return Err(ApiError::PlanLimit(
            &format!("Monthly auto-route quota exceeded ({}/{}). Upgrade or wait for next billing cycle.", usage_count, quota)
        ));
    }

    // 2. Validate routing parameters
    validate_autoroute_params(&req.parameters)?;

    // 3. Create job record
    let job = sqlx::query_as!(
        AutorouteJob,
        r#"INSERT INTO autoroute_jobs
            (project_id, user_id, status, parameters)
        VALUES ($1, $2, 'pending', $3)
        RETURNING *"#,
        project_id,
        claims.user_id,
        serde_json::to_value(&req.parameters)?
    )
    .fetch_one(&state.db)
    .await?;

    // 4. Enqueue job to Redis
    let priority = match user.plan.as_str() {
        "team" | "enterprise" => 10,
        "pro" => 5,
        _ => 0,
    };

    state.redis
        .zadd("autoroute:jobs", job.id.to_string(), priority)
        .await?;

    // 5. Record usage
    sqlx::query!(
        "INSERT INTO usage_records (user_id, org_id, record_type, quantity, period_start, period_end)
         VALUES ($1, $2, 'autoroute_job', 1, $3, $3)",
        claims.user_id,
        project.org_id,
        current_month_start
    )
    .execute(&state.db)
    .await?;

    Ok((StatusCode::CREATED, Json(job)))
}

pub async fn get_autoroute_job(
    State(state): State<AppState>,
    claims: Claims,
    Path((project_id, job_id)): Path<(Uuid, Uuid)>,
) -> Result<Json<AutorouteJob>, ApiError> {
    let job = sqlx::query_as!(
        AutorouteJob,
        "SELECT aj.* FROM autoroute_jobs aj
         JOIN projects p ON aj.project_id = p.id
         WHERE aj.id = $1 AND aj.project_id = $2
         AND (p.owner_id = $3 OR p.org_id IN (
             SELECT org_id FROM org_members WHERE user_id = $3
         ))",
        job_id,
        project_id,
        claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Auto-route job not found"))?;

    Ok(Json(job))
}

fn validate_autoroute_params(params: &AutorouteParams) -> Result<(), ApiError> {
    // Validate net classes
    if params.net_classes.is_empty() {
        return Err(ApiError::Validation("At least one net class must be defined"));
    }

    for nc in &params.net_classes {
        if nc.trace_width_mm < 0.1 || nc.trace_width_mm > 10.0 {
            return Err(ApiError::Validation("Trace width must be between 0.1mm and 10mm"));
        }
        if nc.clearance_mm < 0.1 || nc.clearance_mm > 5.0 {
            return Err(ApiError::Validation("Clearance must be between 0.1mm and 5mm"));
        }
        if nc.via_drill_mm < 0.2 || nc.via_drill_mm > 3.0 {
            return Err(ApiError::Validation("Via drill must be between 0.2mm and 3mm"));
        }
    }

    // Validate strategy
    if !["fast", "balanced", "optimize"].contains(&params.strategy.as_str()) {
        return Err(ApiError::Validation("Strategy must be one of: fast, balanced, optimize"));
    }

    Ok(())
}
```

### 2. DRC Engine Core (Rust — shared between WASM and native)

The core DRC engine that performs geometric checks on PCB features. This code compiles to both native (server) and WASM (browser) targets.

```rust
// drc-core/src/checks.rs

use geo::{Point, LineString, Polygon, Rect};
use rstar::{RTree, RTreeObject, AABB};
use crate::board::{Trace, Pad, Via, CopperZone};

pub struct DrcEngine {
    pub rtree: RTree<DrcFeature>,
    pub violations: Vec<Violation>,
}

#[derive(Debug, Clone)]
pub enum DrcFeature {
    Trace { id: String, layer: String, line: LineString<f64>, width: f64 },
    Pad { id: String, layer: String, center: Point<f64>, shape: PadShape },
    Via { id: String, center: Point<f64>, drill: f64, diameter: f64 },
    Zone { id: String, layer: String, polygon: Polygon<f64> },
}

#[derive(Debug, Clone)]
pub enum PadShape {
    Circle { radius: f64 },
    Rectangle { width: f64, height: f64, rotation: f64 },
    Obround { width: f64, height: f64 },
}

#[derive(Debug, Clone)]
pub struct Violation {
    pub rule_name: String,
    pub severity: Severity,
    pub description: String,
    pub location: Point<f64>,
    pub layer: Option<String>,
    pub affected_features: Vec<String>,
}

#[derive(Debug, Clone, PartialEq)]
pub enum Severity {
    Error,
    Warning,
    Info,
}

impl RTreeObject for DrcFeature {
    type Envelope = AABB<[f64; 2]>;

    fn envelope(&self) -> Self::Envelope {
        match self {
            DrcFeature::Trace { line, width, .. } => {
                let bounds = line.bounding_rect().unwrap();
                let margin = width / 2.0;
                AABB::from_corners(
                    [bounds.min().x - margin, bounds.min().y - margin],
                    [bounds.max().x + margin, bounds.max().y + margin],
                )
            }
            DrcFeature::Pad { center, shape, .. } => {
                let (half_w, half_h) = match shape {
                    PadShape::Circle { radius } => (*radius, *radius),
                    PadShape::Rectangle { width, height, .. } => (width / 2.0, height / 2.0),
                    PadShape::Obround { width, height } => (width / 2.0, height / 2.0),
                };
                AABB::from_corners(
                    [center.x() - half_w, center.y() - half_h],
                    [center.x() + half_w, center.y() + half_h],
                )
            }
            DrcFeature::Via { center, diameter, .. } => {
                let r = diameter / 2.0;
                AABB::from_corners(
                    [center.x() - r, center.y() - r],
                    [center.x() + r, center.y() + r],
                )
            }
            DrcFeature::Zone { polygon, .. } => {
                let bounds = polygon.bounding_rect().unwrap();
                AABB::from_corners(
                    [bounds.min().x, bounds.min().y],
                    [bounds.max().x, bounds.max().y],
                )
            }
        }
    }
}

impl DrcEngine {
    pub fn new() -> Self {
        Self {
            rtree: RTree::new(),
            violations: Vec::new(),
        }
    }

    pub fn add_feature(&mut self, feature: DrcFeature) {
        self.rtree.insert(feature);
    }

    /// Run all DRC checks with given rules
    pub fn run_checks(&mut self, rules: &DrcRules) {
        self.violations.clear();

        self.check_clearance(rules.clearance_mm);
        self.check_trace_width(rules.min_trace_width_mm, rules.max_trace_width_mm);
        self.check_annular_ring(rules.min_annular_ring_mm);
        self.check_drill_size(rules.min_drill_mm);
    }

    /// Check clearance between all features on the same layer
    fn check_clearance(&mut self, min_clearance: f64) {
        let all_features: Vec<DrcFeature> = self.rtree.iter().cloned().collect();

        for (i, feature_a) in all_features.iter().enumerate() {
            // Spatial query: find nearby features within clearance distance
            let envelope_a = feature_a.envelope();
            let search_envelope = AABB::from_corners(
                [envelope_a.lower()[0] - min_clearance, envelope_a.lower()[1] - min_clearance],
                [envelope_a.upper()[0] + min_clearance, envelope_a.upper()[1] + min_clearance],
            );

            for feature_b in self.rtree.locate_in_envelope(&search_envelope) {
                // Skip self-comparison
                if std::ptr::eq(feature_a, feature_b) {
                    continue;
                }

                // Only check features on the same layer
                if get_layer(feature_a) != get_layer(feature_b) {
                    continue;
                }

                let distance = calculate_feature_distance(feature_a, feature_b);
                if distance < min_clearance {
                    self.violations.push(Violation {
                        rule_name: "clearance".to_string(),
                        severity: Severity::Error,
                        description: format!(
                            "Clearance violation: {:.3}mm (minimum: {:.3}mm)",
                            distance, min_clearance
                        ),
                        location: midpoint_between_features(feature_a, feature_b),
                        layer: get_layer(feature_a).cloned(),
                        affected_features: vec![get_id(feature_a), get_id(feature_b)],
                    });
                }
            }
        }
    }

    /// Check trace width compliance
    fn check_trace_width(&mut self, min_width: f64, max_width: f64) {
        for feature in self.rtree.iter() {
            if let DrcFeature::Trace { id, layer, width, .. } = feature {
                if *width < min_width {
                    self.violations.push(Violation {
                        rule_name: "trace_width_min".to_string(),
                        severity: Severity::Error,
                        description: format!(
                            "Trace width {:.3}mm is below minimum {:.3}mm",
                            width, min_width
                        ),
                        location: get_feature_center(feature),
                        layer: Some(layer.clone()),
                        affected_features: vec![id.clone()],
                    });
                }
                if *width > max_width {
                    self.violations.push(Violation {
                        rule_name: "trace_width_max".to_string(),
                        severity: Severity::Warning,
                        description: format!(
                            "Trace width {:.3}mm exceeds maximum {:.3}mm",
                            width, max_width
                        ),
                        location: get_feature_center(feature),
                        layer: Some(layer.clone()),
                        affected_features: vec![id.clone()],
                    });
                }
            }
        }
    }

    /// Check annular ring around drilled holes
    fn check_annular_ring(&mut self, min_annular: f64) {
        for feature in self.rtree.iter() {
            if let DrcFeature::Via { id, diameter, drill, center, .. } = feature {
                let annular = (diameter - drill) / 2.0;
                if annular < min_annular {
                    self.violations.push(Violation {
                        rule_name: "annular_ring".to_string(),
                        severity: Severity::Error,
                        description: format!(
                            "Annular ring {:.3}mm is below minimum {:.3}mm (drill: {:.3}mm, pad: {:.3}mm)",
                            annular, min_annular, drill, diameter
                        ),
                        location: *center,
                        layer: None,
                        affected_features: vec![id.clone()],
                    });
                }
            }
        }
    }

    /// Check minimum drill size
    fn check_drill_size(&mut self, min_drill: f64) {
        for feature in self.rtree.iter() {
            if let DrcFeature::Via { id, drill, center, .. } = feature {
                if *drill < min_drill {
                    self.violations.push(Violation {
                        rule_name: "drill_size".to_string(),
                        severity: Severity::Error,
                        description: format!(
                            "Drill size {:.3}mm is below minimum {:.3}mm",
                            drill, min_drill
                        ),
                        location: *center,
                        layer: None,
                        affected_features: vec![id.clone()],
                    });
                }
            }
        }
    }
}

#[derive(Debug)]
pub struct DrcRules {
    pub clearance_mm: f64,
    pub min_trace_width_mm: f64,
    pub max_trace_width_mm: f64,
    pub min_annular_ring_mm: f64,
    pub min_drill_mm: f64,
}

// Helper functions
fn get_layer(feature: &DrcFeature) -> Option<&String> {
    match feature {
        DrcFeature::Trace { layer, .. } => Some(layer),
        DrcFeature::Pad { layer, .. } => Some(layer),
        DrcFeature::Zone { layer, .. } => Some(layer),
        DrcFeature::Via { .. } => None,  // Vias span all layers
    }
}

fn get_id(feature: &DrcFeature) -> String {
    match feature {
        DrcFeature::Trace { id, .. } => id.clone(),
        DrcFeature::Pad { id, .. } => id.clone(),
        DrcFeature::Via { id, .. } => id.clone(),
        DrcFeature::Zone { id, .. } => id.clone(),
    }
}

fn get_feature_center(feature: &DrcFeature) -> Point<f64> {
    match feature {
        DrcFeature::Trace { line, .. } => {
            let coords: Vec<_> = line.coords().collect();
            let mid_idx = coords.len() / 2;
            Point::new(coords[mid_idx].x, coords[mid_idx].y)
        }
        DrcFeature::Pad { center, .. } => *center,
        DrcFeature::Via { center, .. } => *center,
        DrcFeature::Zone { polygon, .. } => {
            let centroid = polygon.centroid().unwrap();
            centroid
        }
    }
}

fn calculate_feature_distance(a: &DrcFeature, b: &DrcFeature) -> f64 {
    // Simplified distance calculation (for full implementation, use polygon-polygon distance)
    let center_a = get_feature_center(a);
    let center_b = get_feature_center(b);

    use geo::EuclideanDistance;
    center_a.euclidean_distance(&center_b)
}

fn midpoint_between_features(a: &DrcFeature, b: &DrcFeature) -> Point<f64> {
    let center_a = get_feature_center(a);
    let center_b = get_feature_center(b);
    Point::new(
        (center_a.x() + center_b.x()) / 2.0,
        (center_a.y() + center_b.y()) / 2.0,
    )
}
```

### 3. KiCad Import Parser (Rust)

Parser for KiCad 7/8 S-expression format to import existing projects.

```rust
// src/import/kicad.rs

use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use anyhow::{Context, Result};

pub struct KicadImporter;

#[derive(Debug, Serialize)]
pub struct ImportedProject {
    pub schematic_data: serde_json::Value,
    pub pcb_data: serde_json::Value,
    pub board_params: BoardParams,
    pub components: Vec<ComponentMapping>,
}

#[derive(Debug, Serialize)]
pub struct BoardParams {
    pub width_mm: f64,
    pub height_mm: f64,
    pub layer_count: u8,
    pub stackup: Vec<Layer>,
}

#[derive(Debug, Serialize)]
pub struct Layer {
    pub name: String,
    pub layer_type: String,  // copper | dielectric | soldermask | silkscreen
    pub thickness_mm: f64,
}

#[derive(Debug, Serialize)]
pub struct ComponentMapping {
    pub ref_des: String,
    pub kicad_symbol: String,
    pub kicad_footprint: String,
    pub circuitmind_component_id: Option<uuid::Uuid>,
    pub needs_manual_mapping: bool,
}

impl KicadImporter {
    pub fn import_project(
        schematic_path: &str,
        pcb_path: &str,
    ) -> Result<ImportedProject> {
        // Parse KiCad schematic (.kicad_sch)
        let schematic_content = std::fs::read_to_string(schematic_path)
            .context("Failed to read schematic file")?;
        let schematic_sexpr = parse_sexpr(&schematic_content)?;

        // Parse KiCad PCB (.kicad_pcb)
        let pcb_content = std::fs::read_to_string(pcb_path)
            .context("Failed to read PCB file")?;
        let pcb_sexpr = parse_sexpr(&pcb_content)?;

        // Extract schematic data
        let mut schematic_components = Vec::new();
        let mut schematic_wires = Vec::new();

        if let SExpr::List(items) = &schematic_sexpr {
            for item in items {
                if let SExpr::List(inner) = item {
                    if let Some(SExpr::Symbol(name)) = inner.first() {
                        match name.as_str() {
                            "symbol" => {
                                schematic_components.push(parse_schematic_symbol(inner)?);
                            }
                            "wire" => {
                                schematic_wires.push(parse_wire(inner)?);
                            }
                            _ => {}
                        }
                    }
                }
            }
        }

        // Extract PCB data
        let mut pcb_traces = Vec::new();
        let mut pcb_vias = Vec::new();
        let mut pcb_zones = Vec::new();
        let mut board_outline = None;
        let mut layer_count = 2;

        if let SExpr::List(items) = &pcb_sexpr {
            for item in items {
                if let SExpr::List(inner) = item {
                    if let Some(SExpr::Symbol(name)) = inner.first() {
                        match name.as_str() {
                            "segment" => {
                                pcb_traces.push(parse_segment(inner)?);
                            }
                            "via" => {
                                pcb_vias.push(parse_via(inner)?);
                            }
                            "zone" => {
                                pcb_zones.push(parse_zone(inner)?);
                            }
                            "gr_line" | "gr_rect" | "gr_poly" => {
                                if is_board_outline(inner) {
                                    board_outline = Some(parse_board_outline(inner)?);
                                }
                            }
                            "layers" => {
                                layer_count = count_copper_layers(inner)?;
                            }
                            _ => {}
                        }
                    }
                }
            }
        }

        // Map components to CircuitMind library
        let component_mappings = map_components_to_library(&schematic_components)?;

        // Extract board dimensions
        let (width_mm, height_mm) = board_outline
            .as_ref()
            .map(|outline| calculate_board_dimensions(outline))
            .unwrap_or((100.0, 100.0));

        Ok(ImportedProject {
            schematic_data: serde_json::json!({
                "components": schematic_components,
                "wires": schematic_wires,
                "version": "1.0",
            }),
            pcb_data: serde_json::json!({
                "traces": pcb_traces,
                "vias": pcb_vias,
                "zones": pcb_zones,
                "version": "1.0",
            }),
            board_params: BoardParams {
                width_mm,
                height_mm,
                layer_count,
                stackup: generate_default_stackup(layer_count),
            },
            components: component_mappings,
        })
    }
}

// S-expression parser
#[derive(Debug, Clone)]
enum SExpr {
    Symbol(String),
    String(String),
    Number(f64),
    List(Vec<SExpr>),
}

fn parse_sexpr(input: &str) -> Result<SExpr> {
    // Simplified S-expression parser (full implementation would use a proper parser library)
    // This is a placeholder - production code should use `sexp` or similar crate
    todo!("Full S-expression parser implementation")
}

fn parse_schematic_symbol(sexpr: &[SExpr]) -> Result<serde_json::Value> {
    // Extract symbol properties: reference, value, position, rotation
    todo!()
}

fn parse_wire(sexpr: &[SExpr]) -> Result<serde_json::Value> {
    // Extract wire start/end points, net name
    todo!()
}

fn parse_segment(sexpr: &[SExpr]) -> Result<serde_json::Value> {
    // Extract trace segment: start, end, layer, width, net
    todo!()
}

fn parse_via(sexpr: &[SExpr]) -> Result<serde_json::Value> {
    // Extract via: position, drill, diameter, net
    todo!()
}

fn parse_zone(sexpr: &[SExpr]) -> Result<serde_json::Value> {
    // Extract copper pour zone: polygon, net, layer
    todo!()
}

fn is_board_outline(sexpr: &[SExpr]) -> bool {
    // Check if graphic element is on Edge.Cuts layer
    todo!()
}

fn parse_board_outline(sexpr: &[SExpr]) -> Result<Vec<(f64, f64)>> {
    // Extract board outline polygon
    todo!()
}

fn count_copper_layers(sexpr: &[SExpr]) -> Result<u8> {
    // Count layers starting with "F.Cu", "In*.Cu", "B.Cu"
    todo!()
}

fn calculate_board_dimensions(outline: &[(f64, f64)]) -> (f64, f64) {
    let (min_x, max_x, min_y, max_y) = outline.iter().fold(
        (f64::MAX, f64::MIN, f64::MAX, f64::MIN),
        |(min_x, max_x, min_y, max_y), &(x, y)| {
            (min_x.min(x), max_x.max(x), min_y.min(y), max_y.max(y))
        },
    );
    (max_x - min_x, max_y - min_y)
}

fn map_components_to_library(
    schematic_components: &[serde_json::Value],
) -> Result<Vec<ComponentMapping>> {
    // Map KiCad symbols/footprints to CircuitMind component library
    // Use fuzzy matching on MPN, package type, and symbol name
    todo!()
}

fn generate_default_stackup(layer_count: u8) -> Vec<Layer> {
    match layer_count {
        2 => vec![
            Layer { name: "F.Cu".into(), layer_type: "copper".into(), thickness_mm: 0.035 },
            Layer { name: "Dielectric".into(), layer_type: "dielectric".into(), thickness_mm: 1.5 },
            Layer { name: "B.Cu".into(), layer_type: "copper".into(), thickness_mm: 0.035 },
        ],
        4 => vec![
            Layer { name: "F.Cu".into(), layer_type: "copper".into(), thickness_mm: 0.035 },
            Layer { name: "Prepreg1".into(), layer_type: "dielectric".into(), thickness_mm: 0.2 },
            Layer { name: "In1.Cu".into(), layer_type: "copper".into(), thickness_mm: 0.035 },
            Layer { name: "Core".into(), layer_type: "dielectric".into(), thickness_mm: 1.0 },
            Layer { name: "In2.Cu".into(), layer_type: "copper".into(), thickness_mm: 0.035 },
            Layer { name: "Prepreg2".into(), layer_type: "dielectric".into(), thickness_mm: 0.2 },
            Layer { name: "B.Cu".into(), layer_type: "copper".into(), thickness_mm: 0.035 },
        ],
        _ => vec![],
    }
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–5)

**Day 1: Project initialization and Rust backend scaffold**
- Initialize Cargo workspace with `circuitmind-api`, `drc-core`, `drc-wasm`
- Add dependencies: axum, tokio, sqlx, uuid, serde, tower-http, jsonwebtoken, bcrypt
- `src/main.rs`: Axum app with CORS, tracing, graceful shutdown
- `src/config.rs`: Environment config (DATABASE_URL, JWT_SECRET, S3_BUCKET, REDIS_URL)
- `src/state.rs`: AppState with PgPool, Redis, S3Client
- `src/error.rs`: ApiError enum with IntoResponse
- `Dockerfile`: Multi-stage build
- `docker-compose.yml`: PostgreSQL + PostGIS, Redis, MinIO

**Day 2: Database schema and migrations**
- `migrations/001_initial.sql`: All 11 tables
- `src/db/mod.rs`: Database pool initialization
- `src/db/models.rs`: All SQLx structs with FromRow
- Run `sqlx migrate run`
- Seed script for initial component library (100 common parts)

**Day 3: Authentication system**
- `src/auth/mod.rs`: JWT generation and validation middleware
- `src/auth/oauth.rs`: Google and GitHub OAuth flows
- `src/api/handlers/auth.rs`: Register, login, OAuth callback, refresh, me
- Password hashing with bcrypt cost 12
- JWT with 24h access + 30d refresh tokens

**Day 4: User and organization CRUD**
- `src/api/handlers/users.rs`: Get profile, update profile, delete account
- `src/api/handlers/orgs.rs`: Create org, invite member, list members, remove member, update settings
- Integration tests for auth flow and authorization

**Day 5: Project management API**
- `src/api/handlers/projects.rs`: Create, list, get, update, delete, fork
- `src/api/handlers/revisions.rs`: Create revision, list revisions, get revision snapshot
- S3 integration for revision snapshots
- Project thumbnail generation placeholder

### Phase 2 — DRC Engine Core (Days 6–10)

**Day 6: DRC geometry framework**
- `drc-core/src/lib.rs`: Core DRC types and traits
- `drc-core/src/checks.rs`: DrcEngine with R-tree spatial index
- `drc-core/src/board.rs`: Board feature types (Trace, Pad, Via, Zone)
- Unit tests: simple clearance checks on dummy boards

**Day 7: DRC rule implementations**
- Implement clearance, trace width, annular ring, drill size checks
- `drc-core/src/rules.rs`: DrcRules configuration struct
- Violation severity classification
- Tests: validate each rule category independently

**Day 8: Advanced DRC checks**
- Courtyard overlap detection using polygon intersection
- Silk-to-pad clearance checks
- Board edge clearance validation
- Differential pair matching (length, spacing tolerance)
- Tests: complex multi-layer board scenarios

**Day 9: WASM DRC compilation**
- `drc-wasm/Cargo.toml`: WASM target configuration
- `drc-wasm/src/lib.rs`: WASM entry points for incremental DRC
- `wasm-pack build --target web --release`
- JavaScript wrapper: `DrcEngine` class
- Size optimization: target <500KB gzipped

**Day 10: Server-side DRC API**
- `src/api/handlers/drc.rs`: Run DRC, get violations, clear violations
- `src/workers/drc_worker.rs`: Background DRC job processor
- DRC job queue via Redis
- WebSocket progress updates
- Store violations in PostgreSQL with PostGIS location

### Phase 3 — Component Library + Supplier Integration (Days 11–15)

**Day 11: Component library seeding**
- Generate 50,000+ component entries from IPC-7351 footprint database
- Common passives: resistors (0402-2512), capacitors (0402-1210), inductors
- Semiconductors: diodes, MOSFETs, BJTs, voltage regulators
- ICs: microcontrollers (STM32, ESP32, RP2040), op-amps, logic gates
- Connectors: headers, USB, HDMI, RJ45
- SVG symbol generation for each category

**Day 12: DigiKey API integration**
- `src/suppliers/digikey.rs`: OAuth authentication, product search, pricing API
- `src/workers/pricing_sync.rs`: Hourly background job to refresh pricing cache
- Map DigiKey part numbers to component library via MPN matching
- Store pricing in `component_pricing` table

**Day 13: Mouser and LCSC API integration**
- `src/suppliers/mouser.rs`: Mouser API client
- `src/suppliers/lcsc.rs`: LCSC API client
- Unified supplier interface trait
- Multi-supplier cost optimization algorithm for BOM

**Day 14: Component search API**
- `src/api/handlers/components.rs`: Search, get details, get pricing, get alternates
- Meilisearch index for parametric search (category, package, voltage, current)
- Full-text search on name, MPN, description
- Pagination with cursor-based scrolling

**Day 15: BOM generation and cost optimization**
- `src/api/handlers/bom.rs`: Generate BOM from schematic, export CSV/Excel
- `src/bom/optimizer.rs`: Multi-supplier cost optimization (minimize total cost across MOQ constraints)
- BOM diff between project revisions
- Lifecycle status warnings for at-risk components

### Phase 4 — Frontend Foundation + Schematic Editor (Days 16–22)

**Day 16: Frontend scaffold**
- `npm create vite@latest frontend -- --template react-ts`
- Install: zustand, @tanstack/react-query, axios, rbush, tailwindcss
- `src/App.tsx`: Router, auth context, layout
- `src/stores/authStore.ts`: Auth state with JWT refresh
- `src/stores/projectStore.ts`: Project state
- `src/api/client.ts`: Axios client with auth interceptors

**Day 17: Project dashboard**
- `src/pages/Dashboard.tsx`: Project grid with thumbnails
- `src/components/ProjectCard.tsx`: Card with preview, metadata, actions
- Search and filter UI
- Quick-create modal with templates
- Import KiCad button (file upload)

**Day 18: Schematic editor foundation**
- `src/components/SchematicEditor/Canvas.tsx`: SVG canvas with pan/zoom (transform matrix)
- `src/components/SchematicEditor/Grid.tsx`: Snap-to-grid overlay (10mm/20mm/50mm)
- `src/lib/spatial.ts`: R-tree setup for component/wire spatial queries
- `src/stores/schematicStore.ts`: Schematic state (components, wires, selection)
- Mouse event handlers: pan (middle-drag), zoom (wheel), select (click)

**Day 19: Component palette and placement**
- `src/components/SchematicEditor/ComponentPalette.tsx`: Categorized library sidebar
- `src/components/SchematicEditor/ComponentSymbol.tsx`: SVG symbol renderer
- Drag-and-drop: palette → canvas
- Component rotation (R key), flip (F key), delete (Del key)
- Property editor panel: value, footprint, reference designator

**Day 20: Wire routing and net labels**
- `src/components/SchematicEditor/Wire.tsx`: Click-to-place wire tool
- Auto-Manhattan routing with orthogonal segments
- Automatic junction creation at wire intersections
- `src/components/SchematicEditor/NetLabel.tsx`: GND, VCC, custom net labels
- Cross-probing: hover component → highlight connected wires

**Day 21: Schematic advanced features**
- Multi-select with box drag
- Copy/paste, group move
- Undo/redo stack (Zustand middleware)
- Keyboard shortcuts: Ctrl+Z/Y, Ctrl+C/V, Ctrl+S, R, G
- Auto-wire routing avoiding component overlaps
- Electrical rule check (ERC): floating pins, duplicate nets

**Day 22: Netlist generation**
- `src/lib/netlist.ts`: Generate SPICE-compatible netlist from schematic
- Component-to-net mapping
- Hierarchical sheet support
- Net name normalization
- Validation: check for floating nodes, missing ground

### Phase 5 — PCB Layout Editor (Days 23–29)

**Day 23: PCB canvas and WebGL renderer**
- `src/components/PcbEditor/Canvas.tsx`: WebGL2 canvas setup
- `src/components/PcbEditor/Renderer.ts`: Layer rendering pipeline
- Shader programs for traces, pads, vias, zones
- Layer compositing with alpha blending
- Pan/zoom controls (same as schematic)

**Day 24: Board setup and layer stackup**
- `src/components/PcbEditor/BoardSetup.tsx`: Define width, height, layer count
- `src/components/PcbEditor/StackupEditor.tsx`: Layer stack configuration (copper, prepreg, core thicknesses)
- Board outline drawing tool
- Keepout zone placement
- Mounting hole placement

**Day 25: Interactive routing (manual)**
- `src/components/PcbEditor/RoutingTool.tsx`: Click-to-route trace placement
- Snap-to-pad, snap-to-grid, snap-to-track
- Push-and-shove algorithm for conflict resolution
- Via placement (V key during routing)
- Layer switching (+ / - keys)
- Trace width selection per net class

**Day 26: Copper pour and zones**
- `src/components/PcbEditor/ZoneTool.tsx`: Polygon zone drawing
- Copper fill with thermal relief, spoke, direct connect options
- Hatch pattern for ground/power planes
- Zone priority and clearance settings
- GPU tessellation for large zones

**Day 27: Component placement on PCB**
- Synchronize schematic netlist → PCB component list
- `src/components/PcbEditor/PlacementTool.tsx`: Drag components from netlist panel
- Footprint rendering (pads, outline, courtyard)
- Rotation, flip (top/bottom side)
- Alignment tools: distribute, align to grid
- Ratsnest lines showing unrouted connections

**Day 28: DRC integration**
- Load WASM DRC engine
- Real-time incremental DRC during routing (<100ms feedback)
- `src/components/PcbEditor/DrcPanel.tsx`: Violation list with severity icons
- Click violation → zoom to location and highlight affected features
- DRC rule editor: per-layer, per-net-class rules

**Day 29: 3D board viewer**
- `src/components/PcbEditor/Viewer3D.tsx`: Three.js scene
- Generate 3D mesh from PCB layers (copper, solder mask, silk)
- STEP model import placeholder (OpenCascade.js)
- Orbit controls, lighting
- Board edge visualization for enclosure fit check

### Phase 6 — KiCad Import + Gerber Export (Days 30–34)

**Day 30: KiCad schematic import**
- `src/import/kicad_schematic.rs`: Parse `.kicad_sch` S-expressions
- Extract components, wires, net labels, hierarchical sheets
- Map KiCad symbols to CircuitMind library (fuzzy matching on name/MPN)
- Auto-layout schematic if no position data

**Day 31: KiCad PCB import**
- `src/import/kicad_pcb.rs`: Parse `.kicad_pcb` S-expressions
- Extract traces, vias, zones, board outline, component placements
- Footprint mapping to CircuitMind library
- Layer mapping: F.Cu→Top, B.Cu→Bottom, In*.Cu→Inner layers

**Day 32: Import conflict resolution UI**
- `src/components/Import/MappingWizard.tsx`: Component mapping review
- Show unmapped components with suggested alternatives
- Manual mapping interface: search and select CircuitMind component
- Preview imported schematic and PCB before final import

**Day 33: Gerber export**
- `src/export/gerber.rs`: Gerber RS-274X writer
- Generate files: copper layers, solder mask, silk, paste, drill (Excellon)
- Auto-populate aperture list and D-codes
- Layer file naming: standard conventions (*.GTL, *.GBL, *.GTS, etc.)
- Zip archive generation for download

**Day 34: Fabrication export bundle**
- Pick-and-place CSV generation (centroid, rotation, side)
- Assembly drawing PDF with component outlines and ref des
- IPC-D-356 netlist for electrical testing
- `src/components/Export/FabExportPanel.tsx`: Export options UI
- Board parameter validation before export (layer count, dimensions, features)

### Phase 7 — Auto-Router Integration (Days 35–39)

**Day 35: Python auto-router service setup**
- Initialize FastAPI service: `autorouter-service/`
- `main.py`: Job API endpoints (start, status, cancel)
- Redis job queue consumer
- PyTorch GPU setup for ML topology model
- Docker container with CUDA support

**Day 36: Topology optimization ML model**
- `models/topology_net.py`: Graph neural network for component placement and net ordering
- Training data: synthetic boards + manually routed reference designs
- Features: netlist connectivity graph, component pin counts, signal types
- Output: component placement suggestions, net routing priority
- Pre-trained model checkpoint from reference dataset

**Day 37: A* pathfinding router (Rust)**
- `autorouter-core/src/pathfinding.rs`: A* with Manhattan distance heuristic
- Grid-based routing with via cost penalties
- Multi-layer support with layer transition costs
- Push-and-shove conflict resolution
- Compile to shared library for Python FFI

**Day 38: Auto-router worker integration**
- `autorouter-service/worker.py`: Job processor
- Flow: fetch job → ML topology → Rust A* routing → result serialization
- Progress updates via Redis pub/sub
- Result upload to S3
- Quality metrics: completion %, DRC violations, via count, total trace length

**Day 39: Auto-router UI**
- `src/components/PcbEditor/AutoroutePanel.tsx`: Net class config, region selection
- Start auto-route button with strategy selection (fast/balanced/optimize)
- Progress bar with live route visualization
- Result review: accept/reject with undo capability
- Quality report card display

### Phase 8 — Billing, Deployment, Launch Prep (Days 40–42)

**Day 40: Stripe billing integration**
- `src/api/handlers/billing.rs`: Create checkout session, customer portal, webhooks
- Webhook handlers: subscription.created, subscription.updated, subscription.deleted
- Plan enforcement middleware for API endpoints
- Usage quota tracking for auto-route jobs
- `src/components/Billing/PricingPage.tsx`: Pricing tiers, feature comparison

**Day 41: Production deployment**
- Deploy Rust API to Fly.io (multiple regions)
- Deploy PostgreSQL + PostGIS (managed service)
- Deploy Redis (managed service)
- Deploy S3/CloudFront for static assets and WASM bundles
- Deploy Meilisearch for component search
- Deploy Python auto-router service to Lambda Cloud GPU instances
- Environment configuration, secrets management
- Load balancing and auto-scaling

**Day 42: Testing, documentation, launch**
- End-to-end integration tests
- Performance testing: large board DRC (2000+ components), auto-routing benchmark
- User documentation: quickstart guide, schematic tutorial, PCB tutorial, KiCad import guide
- Demo video: design Arduino shield in 10 minutes
- Product Hunt launch assets
- Hacker News launch post
- Community seeding: Reddit, EEVblog, Twitter

---

## Critical Files and Structure

```
circuitmind/
├── backend/
│   ├── circuitmind-api/
│   │   ├── src/
│   │   │   ├── main.rs (Axum server entry point)
│   │   │   ├── config.rs (Environment config)
│   │   │   ├── state.rs (AppState: DB, Redis, S3)
│   │   │   ├── error.rs (ApiError enum)
│   │   │   ├── auth/ (JWT, OAuth, middleware)
│   │   │   ├── api/
│   │   │   │   ├── handlers/
│   │   │   │   │   ├── auth.rs
│   │   │   │   │   ├── projects.rs
│   │   │   │   │   ├── components.rs
│   │   │   │   │   ├── drc.rs
│   │   │   │   │   ├── autoroute.rs
│   │   │   │   │   ├── bom.rs
│   │   │   │   │   └── billing.rs
│   │   │   │   └── router.rs
│   │   │   ├── db/
│   │   │   │   ├── mod.rs
│   │   │   │   └── models.rs
│   │   │   ├── workers/
│   │   │   │   ├── drc_worker.rs
│   │   │   │   └── pricing_sync.rs
│   │   │   ├── suppliers/
│   │   │   │   ├── digikey.rs
│   │   │   │   ├── mouser.rs
│   │   │   │   └── lcsc.rs
│   │   │   ├── import/
│   │   │   │   ├── kicad_schematic.rs
│   │   │   │   └── kicad_pcb.rs
│   │   │   ├── export/
│   │   │   │   ├── gerber.rs
│   │   │   │   ├── bom.rs
│   │   │   │   └── pnp.rs
│   │   │   └── bom/
│   │   │       └── optimizer.rs
│   │   ├── migrations/
│   │   │   └── 001_initial.sql
│   │   ├── Cargo.toml
│   │   └── Dockerfile
│   ├── drc-core/
│   │   ├── src/
│   │   │   ├── lib.rs
│   │   │   ├── checks.rs (DrcEngine, spatial index)
│   │   │   ├── rules.rs (DrcRules config)
│   │   │   └── board.rs (Feature types)
│   │   └── Cargo.toml
│   ├── drc-wasm/
│   │   ├── src/
│   │   │   └── lib.rs (WASM bindings)
│   │   └── Cargo.toml
│   └── autorouter-core/
│       ├── src/
│       │   ├── lib.rs
│       │   └── pathfinding.rs (A* router)
│       └── Cargo.toml
├── autorouter-service/
│   ├── main.py (FastAPI app)
│   ├── worker.py (Job processor)
│   ├── models/
│   │   └── topology_net.py (ML model)
│   ├── requirements.txt
│   └── Dockerfile
├── frontend/
│   ├── src/
│   │   ├── App.tsx
│   │   ├── pages/
│   │   │   ├── Dashboard.tsx
│   │   │   ├── SchematicEditor.tsx
│   │   │   └── PcbEditor.tsx
│   │   ├── components/
│   │   │   ├── SchematicEditor/
│   │   │   │   ├── Canvas.tsx
│   │   │   │   ├── ComponentPalette.tsx
│   │   │   │   ├── ComponentSymbol.tsx
│   │   │   │   ├── Wire.tsx
│   │   │   │   └── NetLabel.tsx
│   │   │   ├── PcbEditor/
│   │   │   │   ├── Canvas.tsx
│   │   │   │   ├── Renderer.ts (WebGL)
│   │   │   │   ├── RoutingTool.tsx
│   │   │   │   ├── ZoneTool.tsx
│   │   │   │   ├── DrcPanel.tsx
│   │   │   │   ├── AutoroutePanel.tsx
│   │   │   │   └── Viewer3D.tsx
│   │   │   ├── Import/
│   │   │   │   └── MappingWizard.tsx
│   │   │   └── Billing/
│   │   │       └── PricingPage.tsx
│   │   ├── stores/
│   │   │   ├── authStore.ts
│   │   │   ├── projectStore.ts
│   │   │   ├── schematicStore.ts
│   │   │   └── pcbStore.ts
│   │   ├── lib/
│   │   │   ├── spatial.ts (R-tree)
│   │   │   ├── netlist.ts
│   │   │   └── webgl/ (Shaders, rendering utils)
│   │   └── api/
│   │       └── client.ts
│   ├── package.json
│   └── vite.config.ts
├── docker-compose.yml
└── README.md
```

---

## DRC Validation Benchmarks

### Benchmark 1: Simple 2-Layer Board (Clearance Check)
- **Board:** 100mm × 80mm, 2 layers, 50 components, 200 traces, 30 vias
- **Rule:** 0.15mm minimum clearance (6 mil / IPC Class 2)
- **Expected violations:** 0 (clean design)
- **WASM performance target:** <50ms for full-board check
- **Server performance target:** <100ms on 4-core CPU
- **Validation:** Compare against KiCad DRC results on identical board

### Benchmark 2: Dense 4-Layer Board (Multi-Rule)
- **Board:** 160mm × 100mm, 4 layers, 250 components, 1500 traces, 400 vias
- **Rules:** 0.13mm clearance, 0.127mm min trace width, 0.05mm annular ring, 0.3mm drill
- **Expected violations:** 12 (intentional: 8 clearance, 2 annular ring, 2 trace width)
- **WASM performance target:** N/A (>500 features → server-side)
- **Server performance target:** <500ms on GPU instance
- **Validation:** All 12 violations detected with correct locations and affected features

### Benchmark 3: BGA Routing (High Density)
- **Board:** 80mm × 80mm, 6 layers, BGA-256 component with 0.8mm pitch
- **Rules:** 0.1mm clearance, 0.1mm trace width, via-in-pad allowed
- **Expected violations:** 0 (valid escape routing)
- **WASM performance target:** N/A (server-side only for 6-layer)
- **Server GPU performance target:** <2s for full DRC with polygon intersection
- **Validation:** No false positives on complex via fanout patterns

### Benchmark 4: Courtyard Overlap Detection
- **Board:** 120mm × 80mm, 2 layers, 100 components with defined courtyards
- **Rules:** No courtyard overlap between components
- **Expected violations:** 4 (intentional placement conflicts)
- **WASM performance target:** <100ms using R-tree spatial index
- **Server performance target:** <150ms
- **Validation:** Correct identification of overlapping component pairs with geometric accuracy

### Benchmark 5: Differential Pair Matching
- **Board:** 140mm × 100mm, 4 layers, USB 2.0 differential pairs (D+/D-)
- **Rules:** ±0.5mm length matching, 0.2mm±10% pair spacing
- **Expected violations:** 2 (one pair: 0.7mm length mismatch, one pair: spacing 0.17mm)
- **WASM performance target:** <80ms for pair analysis
- **Server performance target:** <120ms
- **Validation:** Accurate length calculation along curved traces, spacing measured at multiple sample points

---

## Verification and Testing

### Unit Tests (Rust)
- DRC engine: each rule category with synthetic boards
- Spatial index: R-tree insertion, query, nearest-neighbor
- Netlist generator: schematic → SPICE netlist round-trip
- KiCad importer: parse standard test files, validate component mapping
- Gerber exporter: aperture generation, coordinate precision
- Target coverage: >85% for core modules

### Integration Tests (Rust + API)
- Auth flow: register → login → refresh → authenticated requests
- Project lifecycle: create → update → fork → delete
- DRC workflow: upload board → run DRC → fetch violations → clear
- Auto-route workflow: submit job → poll status → download result
- BOM generation: schematic with 50 parts → multi-supplier optimization
- Stripe webhooks: simulate subscription events, verify DB state

### E2E Tests (Playwright)
- User registration and login flow
- Create project, draw schematic with 10 components, wire routing
- Generate netlist, sync to PCB
- Place components on PCB, manual routing of 5 nets
- Run DRC, verify violation display
- Import KiCad project, verify component mapping
- Export Gerber files, download and verify zip contents
- Upgrade to Pro plan, verify quota increase

### Performance Tests
- DRC: 2-layer board with 500 features → <100ms (WASM), <300ms (server)
- DRC: 4-layer board with 2000 features → <2s (GPU server)
- Auto-route: 200-component 4-layer board → <30s for topology + routing
- Component search: parametric query across 50K parts → <200ms
- WebGL rendering: 2000 traces at 60fps with pan/zoom
- Concurrent users: 100 simultaneous schematic edits with collaboration → <150ms p95 latency

### Security Tests
- SQL injection attempts on all user-controlled inputs
- JWT tampering: invalid signature, expired token, wrong audience
- Authorization bypass: access other user's projects, org isolation
- XSS: malicious component names/descriptions
- Rate limiting: excessive API calls, auto-route job spam
- WASM sandbox: no access to localStorage, network, or DOM outside safe APIs

---

## Deployment Architecture

### Production Infrastructure
- **API Servers:** Fly.io multi-region deployment (3 regions: US-East, EU-West, AP-Southeast)
- **Database:** AWS RDS PostgreSQL 16 with PostGIS extension, Multi-AZ for HA
- **Redis:** AWS ElastiCache Redis 7 cluster mode enabled
- **Object Storage:** AWS S3 with CloudFront CDN for global distribution
- **Auto-Router Workers:** Lambda Cloud GPU instances (NVIDIA A10) with auto-scaling
- **Component Search:** Meilisearch Cloud hosted instance
- **Monitoring:** Grafana Cloud + Prometheus for metrics, Sentry for error tracking
- **CI/CD:** GitHub Actions for build, test, deploy pipeline

### Scaling Strategy
- **Horizontal:** Auto-scale API servers based on CPU >70% for 2 min
- **Database:** Read replicas for component search queries, connection pooling (PgBouncer)
- **WASM DRC:** Client-side execution eliminates server load for <500 feature boards
- **Auto-Router:** Queue-based job distribution, scale GPU workers from 1 (idle) to 10 (peak)
- **CDN:** Cloudflare for static assets, WASM bundles, and API response caching (5 min TTL)

### Disaster Recovery
- **Database:** Automated daily snapshots, point-in-time recovery up to 7 days
- **S3:** Cross-region replication for critical data (revision snapshots, Gerber exports)
- **Redis:** AOF persistence with hourly snapshots
- **RTO:** 4 hours (recovery time objective)
- **RPO:** 1 hour (recovery point objective)

---

## Post-MVP Roadmap

### v1.1 — Collaboration and Advanced Routing (Month 2)
- Real-time multi-user editing with Yjs CRDT synchronization
- Cursor presence and live component locking
- In-context commenting system on schematics and PCB
- AI auto-router general availability (post-MVP was manual routing only)
- Differential pair routing with impedance control
- Length-matched routing groups for high-speed interfaces

### v1.2 — Simulation and Manufacturing (Month 3)
- Integrated SPICE simulation (DC, AC, transient analysis)
- Waveform viewer with measurements
- STEP 3D model import for components via OpenCascade.js
- Enhanced 3D board viewer with realistic material rendering
- One-click ordering integration with JLCPCB, PCBWay, OSH Park
- Assembly drawing generation with polarity markers

### v1.3 — Advanced Import/Export and Version Control (Month 4)
- Eagle (.sch/.brd) project import
- Altium ASCII format import
- ODB++ export for advanced fabrication
- Branch-based version control with visual diff viewer
- Merge conflict resolution for overlapping PCB edits
- Git integration for hardware/firmware co-location

### v1.4 — Signal Integrity and Design Validation (Month 5)
- Impedance calculator with stackup-aware trace width recommendations
- Signal integrity simulation for transmission lines
- Eye diagram analysis for high-speed serial links
- Thermal analysis with copper pour heat dissipation modeling
- Manufacturing DFM checks: acid traps, starved thermals, isolated copper

### v2.0 — Enterprise and Advanced Features (Month 6-8)
- SAML/SSO integration for enterprise authentication
- On-premise deployment option (containerized)
- Custom component library hosting with NDA protection
- PLM system integration (Windchill, Teamcenter)
- Advanced auto-router: multi-objective optimization (minimize EMI, crosstalk, via count)
- Rigid-flex PCB support (up to 12 layers, flex regions)
- HDI (high-density interconnect) support: microvias, blind/buried vias
- Up to 32-layer PCB support for enterprise tier