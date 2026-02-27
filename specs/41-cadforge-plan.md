# 41. CADForge — AI-Assisted Parametric 3D CAD in the Browser

## Implementation Plan

**MVP Scope:** Browser-based parametric 3D CAD platform with 2D constraint-based sketch editor supporting geometric constraints (coincident, parallel, perpendicular, tangent, equal, distance, angle) and dimensional constraints rendered via SVG with R-tree spatial indexing, custom B-Rep geometric kernel compiled to WebAssembly supporting NURBS curves/surfaces with 3D feature operations (extrude blind/through-all/to-surface, revolve, fillet/chamfer with edge chain selection, boolean union/subtract/intersect), WebGPU-accelerated viewport renderer with orbit/pan/zoom camera controls and edge highlighting for selection feedback, parametric feature tree with drag-to-reorder and suppression/unsuppression of features, AI-assisted sketch-to-CAD converting natural language or hand-drawn sketches into constrained parametric 2D profiles via GPT-4o Vision API, STEP AP214 and STL/3MF export with configurable mesh quality settings, version history with linear timeline showing all parameter changes and feature additions, PostgreSQL storage for project metadata with S3 for serialized B-Rep geometry blobs, Stripe billing with three tiers (Free / Pro $29/mo / Team $49/user/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| CAD Kernel | Rust + OpenCascade (OCCT 7.8) | Custom Rust B-Rep kernel wrapping OCCT primitives, compiled to WASM via `occt-sys` |
| Constraint Solver | Rust (nalgebra + custom) | 2D geometric constraint solver using Newton-Raphson for under-constrained systems |
| WASM Compilation | `wasm-pack` + `wasm-bindgen` | Client-side geometry kernel, all modeling ops run in browser |
| AI Services | Python 3.12 (FastAPI) | GPT-4o Vision for sketch recognition, NLP-to-CAD intent parsing |
| Database | PostgreSQL 16 | Projects, users, organizations, feature trees, version history |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | B-Rep geometry blobs, STEP/STL exports, AI training data |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| Sketch Editor | Custom SVG renderer | R-tree (`rbush`) for spatial queries, snap-to-grid, constraint markers |
| 3D Viewport | WebGPU + Three.js | PBR materials, edge highlighting, section plane rendering |
| Real-time | WebSocket (Axum) | Multi-user cursor presence, feature tree updates |
| Job Queue | Redis 7 + Tokio tasks | STEP export jobs, AI processing queue, mesh generation |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | WASM bundle delivery, static frontend assets |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, WASM crash reports |
| CI/CD | GitHub Actions | WASM build pipeline, Rust tests, Docker image push |

### Key Architecture Decisions

1. **WASM-based CAD kernel with full client-side geometry operations**: All parametric modeling (sketch solve, extrude, fillet, boolean) runs in the browser via WASM-compiled Rust+OCCT. This eliminates server round-trips for interactive modeling, provides instant visual feedback, and scales to millions of users with zero backend compute cost for geometry operations. Only heavy jobs (STEP export with external refs, FEA mesh generation, AI inference) require server calls.

2. **OpenCascade Technology (OCCT) wrapped in Rust rather than pure Rust B-Rep**: OCCT is the battle-tested kernel used by FreeCAD, KiCAD, and commercial CAD. Its NURBS surface algorithms, boolean operations, and fillet/chamfer solvers have 30+ years of production hardening. Pure Rust alternatives (Truck, OpenCascade-rs) lack critical features like variable-radius fillets and face blends. We wrap OCCT via `occt-sys` bindings and compile to WASM using Emscripten for C++ dependencies, then bridge to Rust via safe FFI wrappers.

3. **SVG-based sketch editor with R-tree spatial indexing for interactive constraint solving**: SVG provides crisp rendering at any zoom level, native browser text layout for dimensions, and DOM-based event handling for selections. An R-tree (via `rbush` JS library) enables O(log n) spatial queries for snap detection (find nearest point/line within 10px), drag handles, and coincident constraint inference. Canvas/WebGL alternatives were rejected because they require reimplementing text layout, accessibility, and sub-pixel hit testing.

4. **WebGPU for 3D viewport with deferred rendering pipeline**: WebGPU (via Three.js WebGPU renderer) provides 2-4x faster edge rendering than WebGL for CAD models with 100K+ edges. Deferred rendering allows efficient edge highlighting (selected edges rendered in orange with width=3px) without re-rendering the entire model. Fallback to WebGL2 for browsers without WebGPU support (Safari <17, Firefox <preview).

5. **Feature tree stored as ordered JSON array with topology references**: Each feature stores its type (sketch, extrude, fillet), parameters (depth, radius), and references to parent features by UUID. The B-Rep topology is serialized separately to S3 as OCCT's native BinOcaf format (compact binary). This allows parametric history replay (rebuild entire model from scratch by replaying features 1-N) and version diffing (compare feature tree JSON to show which features changed).

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";  -- Fuzzy search for component library

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
    preferences JSONB DEFAULT '{}',  -- UI settings, units (mm/inch), grid size
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
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    settings JSONB DEFAULT '{}',  -- Shared libraries, approval workflows
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

-- Projects (CAD documents)
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    project_type TEXT NOT NULL DEFAULT 'part',  -- part | assembly | drawing
    feature_tree JSONB NOT NULL DEFAULT '[]',  -- Ordered array of features with params
    geometry_url TEXT,  -- S3 URL to BinOcaf serialized B-Rep
    thumbnail_url TEXT,  -- S3 URL to rendered thumbnail
    is_public BOOLEAN NOT NULL DEFAULT false,
    forked_from UUID REFERENCES projects(id) ON DELETE SET NULL,
    metadata JSONB DEFAULT '{}',  -- Material, mass properties, bounding box
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);
CREATE INDEX projects_org_idx ON projects(org_id);
CREATE INDEX projects_updated_idx ON projects(updated_at DESC);
CREATE INDEX projects_public_idx ON projects(is_public) WHERE is_public = true;

-- Version History (linear timeline, not Git-like branching in MVP)
CREATE TABLE versions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    version_number INTEGER NOT NULL,  -- Auto-incremented per project
    commit_message TEXT NOT NULL,
    feature_tree JSONB NOT NULL,  -- Snapshot of feature tree at this version
    geometry_url TEXT,  -- S3 URL to B-Rep at this version
    changes JSONB DEFAULT '[]',  -- [{type: 'feature_added', feature_id, ...}]
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(project_id, version_number)
);
CREATE INDEX versions_project_idx ON versions(project_id, version_number DESC);
CREATE INDEX versions_user_idx ON versions(user_id);

-- Exports (STEP, STL, 3MF files generated on-demand)
CREATE TABLE exports (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    format TEXT NOT NULL,  -- step | stl | 3mf | dxf | iges
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | processing | completed | failed
    file_url TEXT,  -- S3 URL to exported file
    file_size_bytes BIGINT,
    export_settings JSONB DEFAULT '{}',  -- Mesh quality, coordinate system, units
    error_message TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at TIMESTAMPTZ
);
CREATE INDEX exports_project_idx ON exports(project_id);
CREATE INDEX exports_user_idx ON exports(user_id);
CREATE INDEX exports_status_idx ON exports(status);

-- Component Library (parametric templates: bolts, gears, bearings)
CREATE TABLE components (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    category TEXT NOT NULL,  -- fastener | bearing | gear | spring | shaft | structural
    subcategory TEXT,  -- e.g., 'hex_bolt' for fastener
    description TEXT,
    parameters JSONB NOT NULL,  -- [{name: 'length', type: 'number', default: 20, unit: 'mm'}]
    feature_tree_template JSONB NOT NULL,  -- Feature tree with parameter placeholders
    thumbnail_url TEXT,
    is_standard BOOLEAN DEFAULT false,  -- ISO/ANSI standard components
    standard_ref TEXT,  -- e.g., "ISO 4017" for hex bolt
    uploaded_by UUID REFERENCES users(id),
    is_public BOOLEAN DEFAULT false,
    download_count INTEGER DEFAULT 0,
    tags TEXT[] DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX components_category_idx ON components(category);
CREATE INDEX components_name_trgm_idx ON components USING gin(name gin_trgm_ops);
CREATE INDEX components_tags_idx ON components USING gin(tags);
CREATE INDEX components_public_idx ON components(is_public) WHERE is_public = true;

-- AI Requests (for usage tracking and debugging)
CREATE TABLE ai_requests (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    request_type TEXT NOT NULL,  -- sketch_to_cad | nl_to_cad | design_intent | fix_model
    input_data JSONB NOT NULL,  -- User input (text, sketch URL)
    output_data JSONB,  -- AI response (feature tree, constraints)
    model_used TEXT,  -- gpt-4o | custom-vision-v1
    tokens_used INTEGER,
    processing_time_ms INTEGER,
    error_message TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX ai_requests_user_idx ON ai_requests(user_id, created_at DESC);
CREATE INDEX ai_requests_type_idx ON ai_requests(request_type);

-- Collaboration Sessions (real-time multi-user editing)
CREATE TABLE collab_sessions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    active_users JSONB DEFAULT '[]',  -- [{user_id, cursor_position, last_seen}]
    locked_features JSONB DEFAULT '{}',  -- {feature_id: user_id} for edit locking
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE UNIQUE INDEX collab_sessions_project_idx ON collab_sessions(project_id);

-- Usage Tracking (for billing enforcement)
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,  -- ai_query | export | storage_gb | collab_hours
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
    pub preferences: serde_json::Value,
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
    pub project_type: String,
    pub feature_tree: serde_json::Value,
    pub geometry_url: Option<String>,
    pub thumbnail_url: Option<String>,
    pub is_public: bool,
    pub forked_from: Option<Uuid>,
    pub metadata: serde_json::Value,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Version {
    pub id: Uuid,
    pub project_id: Uuid,
    pub user_id: Uuid,
    pub version_number: i32,
    pub commit_message: String,
    pub feature_tree: serde_json::Value,
    pub geometry_url: Option<String>,
    pub changes: serde_json::Value,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Export {
    pub id: Uuid,
    pub project_id: Uuid,
    pub user_id: Uuid,
    pub format: String,
    pub status: String,
    pub file_url: Option<String>,
    pub file_size_bytes: Option<i64>,
    pub export_settings: serde_json::Value,
    pub error_message: Option<String>,
    pub created_at: DateTime<Utc>,
    pub completed_at: Option<DateTime<Utc>>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Component {
    pub id: Uuid,
    pub name: String,
    pub category: String,
    pub subcategory: Option<String>,
    pub description: Option<String>,
    pub parameters: serde_json::Value,
    pub feature_tree_template: serde_json::Value,
    pub thumbnail_url: Option<String>,
    pub is_standard: bool,
    pub standard_ref: Option<String>,
    pub uploaded_by: Option<Uuid>,
    pub is_public: bool,
    pub download_count: i32,
    pub tags: Vec<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct Feature {
    pub id: Uuid,
    pub feature_type: FeatureType,
    pub name: String,
    pub parameters: serde_json::Value,
    pub parent_ids: Vec<Uuid>,  // Features this depends on
    pub is_suppressed: bool,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case", tag = "type")]
pub enum FeatureType {
    Sketch { plane: SketchPlane },
    Extrude { mode: ExtrudeMode, depth: f64, direction: i8 },
    Revolve { axis: Axis, angle: f64 },
    Fillet { radius: f64, edge_ids: Vec<u32> },
    Chamfer { distance: f64, edge_ids: Vec<u32> },
    Boolean { operation: BooleanOp, tool_feature_id: Uuid },
    Pattern { pattern_type: PatternType, instances: u32 },
    Mirror { plane: MirrorPlane },
}

#[derive(Debug, Deserialize, Serialize)]
pub enum ExtrudeMode {
    Blind,
    ThroughAll,
    ToSurface { target_face_id: u32 },
    MidPlane,
}

#[derive(Debug, Deserialize, Serialize)]
pub enum BooleanOp {
    Union,
    Subtract,
    Intersect,
}

#[derive(Debug, Deserialize, Serialize)]
pub enum SketchPlane {
    XY,
    XZ,
    YZ,
    Custom { origin: [f64; 3], normal: [f64; 3] },
}

#[derive(Debug, Deserialize, Serialize)]
pub struct SketchConstraint {
    pub constraint_type: ConstraintType,
    pub entity_ids: Vec<u32>,  // References to sketch entities
    pub value: Option<f64>,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum ConstraintType {
    Coincident,
    Horizontal,
    Vertical,
    Parallel,
    Perpendicular,
    Tangent,
    Equal,
    Distance,
    Angle,
    Fix,
}
```

---

## CAD Kernel Architecture Deep-Dive

### B-Rep Topology Model

CADForge uses a Boundary Representation (B-Rep) model where solids are represented by their bounding surfaces. The topology hierarchy is:

```
Solid (TopoDS_Solid)
  └─ Shell (TopoDS_Shell) — Closed set of faces
       └─ Face (TopoDS_Face) — Trimmed NURBS surface with outer/inner wires
            └─ Wire (TopoDS_Wire) — Closed loop of edges
                 └─ Edge (TopoDS_Edge) — Trimmed NURBS curve with start/end vertices
                      └─ Vertex (TopoDS_Vertex) — 3D point
```

**NURBS (Non-Uniform Rational B-Spline)** representation allows both analytical surfaces (planes, cylinders, spheres) and free-form surfaces (lofts, blends) in a unified framework. Control points, knot vectors, and weights define each curve/surface.

**Topology persistence** is critical for parametric modeling: when you fillet an edge, the fillet face must "remember" it was created from that edge so parameter changes rebuild correctly. OpenCascade's `TNaming` framework tracks topology through feature tree replay by storing shape creation history.

### Client/Server Split for Geometry Operations

```
User interaction → All modeling in browser
  │
  ├── Sketch solve (2D constraints) → WASM solver
  ├── Extrude/Revolve/Fillet → WASM OCCT kernel
  ├── Boolean operations → WASM OCCT (if <50K faces)
  │   └── Result serialized to IndexedDB
  │
  └── Heavy operations → Server job queue
      ├── STEP export with external references
      ├── IGES export (legacy CAD compatibility)
      ├── Mesh generation for STL (>1M triangles)
      └── FEA mesh (tetrahedral/hexahedral)
```

### WASM Compilation Strategy

OpenCascade is C++ with extensive use of templates and STL. We compile to WASM via:

1. **Emscripten** for C++ → WASM with `-s WASM=1 -s ALLOW_MEMORY_GROWTH=1`
2. **Custom Rust wrapper** using `occt-sys` FFI bindings for type-safe API surface
3. **wasm-bindgen** to expose Rust wrapper to JavaScript

```toml
# cad-kernel/Cargo.toml
[package]
name = "cadforge-kernel-wasm"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
wasm-bindgen = "0.2"
serde = { version = "1", features = ["derive"] }
serde-wasm-bindgen = "0.6"
js-sys = "0.3"
occt-sys = { path = "../occt-sys" }  # Custom OCCT bindings
nalgebra = "0.32"
uuid = { version = "1", features = ["wasm-bindgen"] }

[profile.release]
opt-level = "z"       # Optimize for size (OCCT WASM is ~12MB gzipped)
lto = true
codegen-units = 1
```

### 2D Constraint Solver Algorithm

The constraint solver uses **Newton-Raphson iteration** to minimize residuals for all constraints simultaneously. For a sketch with `n` degrees of freedom (DOF) and `m` constraints:

**Residual vector** `R`: Each constraint contributes one or more residuals. For example:
- Distance constraint `dist(P1, P2) = 50mm` → residual `r = dist(P1, P2) - 50`
- Perpendicular constraint `dot(L1.dir, L2.dir) = 0` → residual `r = dot(L1.dir, L2.dir)`

**Jacobian matrix** `J`: Partial derivatives of each residual w.r.t. each DOF:
```
J[i,j] = ∂r_i / ∂x_j
```

**Newton-Raphson update**:
```
Δx = -J^(-1) · R
x_new = x_old + α·Δx    (with damping factor α ∈ [0.1, 1])
```

**Convergence**: Stop when `||R|| < 1e-6` (all constraints satisfied within tolerance).

**Under-constrained sketches**: If DOF > constraints, solve becomes least-squares minimization with infinite solutions. We choose the solution closest to current positions (minimal movement).

**Over-constrained sketches**: If constraints conflict, solver reports which constraint fails and suggests removal.

---

## Architecture Deep-Dives

### 1. Project API Handler with Feature Tree Updates (Rust/Axum)

Handles project CRUD with optimistic concurrency control for feature tree updates. When a user modifies a feature, we check the version number to prevent conflicts.

```rust
// src/api/handlers/projects.rs

use axum::{
    extract::{Path, State, Json},
    http::StatusCode,
    response::IntoResponse,
};
use uuid::Uuid;
use crate::{
    db::models::{Project, Feature},
    state::AppState,
    auth::Claims,
    error::ApiError,
};

#[derive(serde::Deserialize)]
pub struct CreateProjectRequest {
    pub name: String,
    pub description: Option<String>,
    pub project_type: String,  // "part" | "assembly"
}

#[derive(serde::Deserialize)]
pub struct UpdateFeatureTreeRequest {
    pub features: Vec<Feature>,
    pub version: i32,  // Optimistic lock
}

pub async fn create_project(
    State(state): State<AppState>,
    claims: Claims,
    Json(req): Json<CreateProjectRequest>,
) -> Result<impl IntoResponse, ApiError> {
    // Validate project type
    if !["part", "assembly", "drawing"].contains(&req.project_type.as_str()) {
        return Err(ApiError::BadRequest("Invalid project_type"));
    }

    // Check plan limits
    let project_count: i64 = sqlx::query_scalar!(
        "SELECT COUNT(*) FROM projects WHERE owner_id = $1 AND org_id IS NULL",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    let user = sqlx::query_as!(
        crate::db::models::User,
        "SELECT * FROM users WHERE id = $1",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    if user.plan == "free" && project_count >= 3 {
        return Err(ApiError::PlanLimit(
            "Free plan allows 3 active projects. Upgrade to Pro for unlimited."
        ));
    }

    // Create project
    let project = sqlx::query_as!(
        Project,
        r#"INSERT INTO projects (owner_id, name, description, project_type, feature_tree)
        VALUES ($1, $2, $3, $4, '[]'::jsonb)
        RETURNING *"#,
        claims.user_id,
        req.name,
        req.description.unwrap_or_default(),
        req.project_type,
    )
    .fetch_one(&state.db)
    .await?;

    // Create initial version (v0)
    sqlx::query!(
        r#"INSERT INTO versions (project_id, user_id, version_number, commit_message, feature_tree)
        VALUES ($1, $2, 0, 'Initial commit', '[]'::jsonb)"#,
        project.id,
        claims.user_id,
    )
    .execute(&state.db)
    .await?;

    Ok((StatusCode::CREATED, Json(project)))
}

pub async fn update_feature_tree(
    State(state): State<AppState>,
    claims: Claims,
    Path(project_id): Path<Uuid>,
    Json(req): Json<UpdateFeatureTreeRequest>,
) -> Result<Json<Project>, ApiError> {
    // 1. Verify ownership and get current version
    let current = sqlx::query_as!(
        Project,
        "SELECT * FROM projects WHERE id = $1 AND (owner_id = $2 OR org_id IN (
            SELECT org_id FROM org_members WHERE user_id = $2
        ))",
        project_id,
        claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Project not found"))?;

    let latest_version: i32 = sqlx::query_scalar!(
        "SELECT COALESCE(MAX(version_number), 0) FROM versions WHERE project_id = $1",
        project_id
    )
    .fetch_one(&state.db)
    .await?;

    // 2. Optimistic concurrency check
    if req.version != latest_version {
        return Err(ApiError::Conflict(format!(
            "Version mismatch: expected {}, got {}. Reload and retry.",
            latest_version, req.version
        )));
    }

    // 3. Validate feature tree (check parent references exist)
    validate_feature_tree(&req.features)?;

    // 4. Update project
    let feature_tree_json = serde_json::to_value(&req.features)?;
    let updated = sqlx::query_as!(
        Project,
        r#"UPDATE projects
        SET feature_tree = $2, updated_at = NOW()
        WHERE id = $1
        RETURNING *"#,
        project_id,
        feature_tree_json,
    )
    .fetch_one(&state.db)
    .await?;

    // 5. Create new version
    sqlx::query!(
        r#"INSERT INTO versions (project_id, user_id, version_number, commit_message, feature_tree, changes)
        VALUES ($1, $2, $3, $4, $5, $6)"#,
        project_id,
        claims.user_id,
        latest_version + 1,
        "Feature tree updated",  // TODO: Generate meaningful message
        feature_tree_json,
        serde_json::json!([]),  // TODO: Compute diff
    )
    .execute(&state.db)
    .await?;

    // 6. Broadcast update to WebSocket subscribers (multi-user)
    state.ws_broadcaster.send_project_update(project_id, serde_json::json!({
        "type": "feature_tree_updated",
        "user_id": claims.user_id,
        "version": latest_version + 1,
    }))?;

    Ok(Json(updated))
}

pub async fn get_project(
    State(state): State<AppState>,
    claims: Claims,
    Path(project_id): Path<Uuid>,
) -> Result<Json<Project>, ApiError> {
    let project = sqlx::query_as!(
        Project,
        "SELECT * FROM projects WHERE id = $1 AND (
            owner_id = $2 OR
            is_public = true OR
            org_id IN (SELECT org_id FROM org_members WHERE user_id = $2)
        )",
        project_id,
        claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Project not found"))?;

    Ok(Json(project))
}

pub async fn list_projects(
    State(state): State<AppState>,
    claims: Claims,
) -> Result<Json<Vec<Project>>, ApiError> {
    let projects = sqlx::query_as!(
        Project,
        r#"SELECT * FROM projects
        WHERE owner_id = $1 OR org_id IN (
            SELECT org_id FROM org_members WHERE user_id = $1
        )
        ORDER BY updated_at DESC
        LIMIT 100"#,
        claims.user_id
    )
    .fetch_all(&state.db)
    .await?;

    Ok(Json(projects))
}

fn validate_feature_tree(features: &[Feature]) -> Result<(), ApiError> {
    let feature_ids: std::collections::HashSet<Uuid> =
        features.iter().map(|f| f.id).collect();

    for feature in features {
        // Check all parent references exist
        for parent_id in &feature.parent_ids {
            if !feature_ids.contains(parent_id) {
                return Err(ApiError::BadRequest(&format!(
                    "Feature {} references non-existent parent {}",
                    feature.id, parent_id
                )));
            }
        }

        // Validate parameters based on feature type
        // (depth > 0 for extrude, radius > 0 for fillet, etc.)
        // ... validation logic ...
    }

    Ok(())
}
```

### 2. WASM CAD Kernel Core (Rust + OCCT)

The WASM module that exposes B-Rep modeling operations to JavaScript. This runs entirely in the browser for interactive performance.

```rust
// cad-kernel/src/lib.rs

use wasm_bindgen::prelude::*;
use serde::{Deserialize, Serialize};
use uuid::Uuid;

mod occt_wrapper;
mod sketch_solver;
mod feature_builder;

use occt_wrapper::{OcctShape, OcctFace, OcctEdge};
use sketch_solver::SketchSolver;
use feature_builder::{ExtrudeBuilder, RevolveBuilder, FilletBuilder};

#[wasm_bindgen]
pub struct CadKernel {
    shapes: std::collections::HashMap<Uuid, OcctShape>,
    solver: SketchSolver,
}

#[wasm_bindgen]
impl CadKernel {
    #[wasm_bindgen(constructor)]
    pub fn new() -> Self {
        console_error_panic_hook::set_once();
        Self {
            shapes: std::collections::HashMap::new(),
            solver: SketchSolver::new(),
        }
    }

    /// Solve 2D sketch constraints and return solved entity positions
    #[wasm_bindgen]
    pub fn solve_sketch(&mut self, sketch_json: &str) -> Result<String, JsValue> {
        let sketch: SketchData = serde_json::from_str(sketch_json)
            .map_err(|e| JsValue::from_str(&format!("Parse error: {}", e)))?;

        let solved = self.solver.solve(&sketch)
            .map_err(|e| JsValue::from_str(&format!("Solver error: {}", e)))?;

        serde_json::to_string(&solved)
            .map_err(|e| JsValue::from_str(&format!("Serialize error: {}", e)))
    }

    /// Extrude a sketch to create a 3D solid
    #[wasm_bindgen]
    pub fn extrude(
        &mut self,
        feature_id: &str,
        sketch_id: &str,
        depth: f64,
        mode: &str,
    ) -> Result<(), JsValue> {
        let feature_uuid = Uuid::parse_str(feature_id)
            .map_err(|e| JsValue::from_str(&e.to_string()))?;
        let sketch_uuid = Uuid::parse_str(sketch_id)
            .map_err(|e| JsValue::from_str(&e.to_string()))?;

        let sketch_shape = self.shapes.get(&sketch_uuid)
            .ok_or_else(|| JsValue::from_str("Sketch not found"))?;

        let mut builder = ExtrudeBuilder::new(sketch_shape.clone());
        builder.set_depth(depth);

        match mode {
            "blind" => builder.set_mode_blind(),
            "through_all" => builder.set_mode_through_all(),
            "mid_plane" => builder.set_mode_mid_plane(),
            _ => return Err(JsValue::from_str("Invalid extrude mode")),
        }

        let solid = builder.build()
            .map_err(|e| JsValue::from_str(&format!("Extrude failed: {}", e)))?;

        self.shapes.insert(feature_uuid, solid);
        Ok(())
    }

    /// Apply fillet to edges
    #[wasm_bindgen]
    pub fn fillet(
        &mut self,
        feature_id: &str,
        parent_id: &str,
        radius: f64,
        edge_indices: Vec<u32>,
    ) -> Result<(), JsValue> {
        let feature_uuid = Uuid::parse_str(feature_id)
            .map_err(|e| JsValue::from_str(&e.to_string()))?;
        let parent_uuid = Uuid::parse_str(parent_id)
            .map_err(|e| JsValue::from_str(&e.to_string()))?;

        let parent_shape = self.shapes.get(&parent_uuid)
            .ok_or_else(|| JsValue::from_str("Parent shape not found"))?;

        let mut builder = FilletBuilder::new(parent_shape.clone());
        builder.set_radius(radius);

        for idx in edge_indices {
            let edge = parent_shape.get_edge(idx as usize)
                .ok_or_else(|| JsValue::from_str(&format!("Edge {} not found", idx)))?;
            builder.add_edge(edge);
        }

        let filleted = builder.build()
            .map_err(|e| JsValue::from_str(&format!("Fillet failed: {}", e)))?;

        self.shapes.insert(feature_uuid, filleted);
        Ok(())
    }

    /// Boolean operation (union/subtract/intersect)
    #[wasm_bindgen]
    pub fn boolean(
        &mut self,
        feature_id: &str,
        base_id: &str,
        tool_id: &str,
        operation: &str,
    ) -> Result<(), JsValue> {
        let feature_uuid = Uuid::parse_str(feature_id)
            .map_err(|e| JsValue::from_str(&e.to_string()))?;
        let base_uuid = Uuid::parse_str(base_id)
            .map_err(|e| JsValue::from_str(&e.to_string()))?;
        let tool_uuid = Uuid::parse_str(tool_id)
            .map_err(|e| JsValue::from_str(&e.to_string()))?;

        let base_shape = self.shapes.get(&base_uuid)
            .ok_or_else(|| JsValue::from_str("Base shape not found"))?;
        let tool_shape = self.shapes.get(&tool_uuid)
            .ok_or_else(|| JsValue::from_str("Tool shape not found"))?;

        let result = match operation {
            "union" => occt_wrapper::boolean_union(base_shape, tool_shape),
            "subtract" => occt_wrapper::boolean_subtract(base_shape, tool_shape),
            "intersect" => occt_wrapper::boolean_intersect(base_shape, tool_shape),
            _ => return Err(JsValue::from_str("Invalid boolean operation")),
        }
        .map_err(|e| JsValue::from_str(&format!("Boolean failed: {}", e)))?;

        self.shapes.insert(feature_uuid, result);
        Ok(())
    }

    /// Export shape to tessellated mesh for rendering
    #[wasm_bindgen]
    pub fn tessellate(&self, feature_id: &str, linear_deflection: f64) -> Result<JsValue, JsValue> {
        let feature_uuid = Uuid::parse_str(feature_id)
            .map_err(|e| JsValue::from_str(&e.to_string()))?;

        let shape = self.shapes.get(&feature_uuid)
            .ok_or_else(|| JsValue::from_str("Shape not found"))?;

        let mesh = shape.tessellate(linear_deflection)
            .map_err(|e| JsValue::from_str(&format!("Tessellation failed: {}", e)))?;

        // Convert to JS object
        serde_wasm_bindgen::to_value(&mesh)
            .map_err(|e| JsValue::from_str(&e.to_string()))
    }

    /// Serialize all shapes to binary (for save to IndexedDB/S3)
    #[wasm_bindgen]
    pub fn serialize(&self) -> Result<Vec<u8>, JsValue> {
        let mut buffer = Vec::new();
        occt_wrapper::write_brep(&self.shapes, &mut buffer)
            .map_err(|e| JsValue::from_str(&format!("Serialization failed: {}", e)))?;
        Ok(buffer)
    }

    /// Deserialize shapes from binary
    #[wasm_bindgen]
    pub fn deserialize(&mut self, data: &[u8]) -> Result<(), JsValue> {
        let shapes = occt_wrapper::read_brep(data)
            .map_err(|e| JsValue::from_str(&format!("Deserialization failed: {}", e)))?;
        self.shapes = shapes;
        Ok(())
    }
}

#[derive(Deserialize, Serialize)]
pub struct SketchData {
    pub entities: Vec<SketchEntity>,
    pub constraints: Vec<SketchConstraint>,
}

#[derive(Deserialize, Serialize)]
#[serde(tag = "type")]
pub enum SketchEntity {
    Point { id: u32, x: f64, y: f64 },
    Line { id: u32, start: u32, end: u32 },
    Circle { id: u32, center: u32, radius: f64 },
    Arc { id: u32, center: u32, start: u32, end: u32 },
}

#[derive(Deserialize, Serialize)]
pub struct SketchConstraint {
    pub constraint_type: String,
    pub entity_ids: Vec<u32>,
    pub value: Option<f64>,
}

#[derive(Serialize)]
pub struct TessellatedMesh {
    pub vertices: Vec<f32>,  // [x, y, z, x, y, z, ...]
    pub normals: Vec<f32>,
    pub indices: Vec<u32>,   // Triangle indices
    pub edges: Vec<u32>,     // Edge indices (for wireframe rendering)
}
```

### 3. Sketch Constraint Solver (Rust)

Implements Newton-Raphson iterative solver for 2D geometric constraints. This is the core of parametric sketch modeling.

```rust
// cad-kernel/src/sketch_solver.rs

use nalgebra::{DMatrix, DVector};
use std::collections::HashMap;

pub struct SketchSolver {
    max_iterations: usize,
    tolerance: f64,
}

impl SketchSolver {
    pub fn new() -> Self {
        Self {
            max_iterations: 100,
            tolerance: 1e-6,
        }
    }

    pub fn solve(&self, sketch: &crate::SketchData) -> Result<SolvedSketch, String> {
        // 1. Build DOF map: assign each free coordinate an index
        let mut dof_map = HashMap::new();
        let mut dof_count = 0;

        for entity in &sketch.entities {
            match entity {
                crate::SketchEntity::Point { id, .. } => {
                    dof_map.insert((*id, Coord::X), dof_count);
                    dof_count += 1;
                    dof_map.insert((*id, Coord::Y), dof_count);
                    dof_count += 1;
                }
                _ => {}
            }
        }

        // 2. Initialize solution vector with current positions
        let mut x = DVector::zeros(dof_count);
        for entity in &sketch.entities {
            if let crate::SketchEntity::Point { id, x: px, y: py } = entity {
                if let Some(&xi) = dof_map.get(&(*id, Coord::X)) {
                    x[xi] = *px;
                }
                if let Some(&yi) = dof_map.get(&(*id, Coord::Y)) {
                    x[yi] = *py;
                }
            }
        }

        // 3. Newton-Raphson iteration
        for iter in 0..self.max_iterations {
            // Compute residuals and Jacobian
            let (residuals, jacobian) = self.compute_residuals_and_jacobian(
                sketch, &x, &dof_map
            )?;

            // Check convergence
            let residual_norm = residuals.norm();
            if residual_norm < self.tolerance {
                return Ok(self.build_solution(sketch, &x, &dof_map));
            }

            // Solve J * dx = -R
            let dx = jacobian.clone().lu().solve(&(-residuals.clone()))
                .ok_or_else(|| format!("Singular Jacobian at iteration {}", iter))?;

            // Damped update
            let alpha = self.compute_damping_factor(&residuals, &jacobian, &dx);
            x += alpha * dx;
        }

        Err(format!("Failed to converge after {} iterations", self.max_iterations))
    }

    fn compute_residuals_and_jacobian(
        &self,
        sketch: &crate::SketchData,
        x: &DVector<f64>,
        dof_map: &HashMap<(u32, Coord), usize>,
    ) -> Result<(DVector<f64>, DMatrix<f64>), String> {
        let constraint_count = sketch.constraints.len();
        let dof_count = x.len();

        let mut residuals = DVector::zeros(constraint_count);
        let mut jacobian = DMatrix::zeros(constraint_count, dof_count);

        for (i, constraint) in sketch.constraints.iter().enumerate() {
            match constraint.constraint_type.as_str() {
                "distance" => {
                    // Distance constraint between two points
                    if constraint.entity_ids.len() != 2 {
                        return Err("Distance constraint requires 2 points".to_string());
                    }
                    let p1_id = constraint.entity_ids[0];
                    let p2_id = constraint.entity_ids[1];
                    let target_dist = constraint.value
                        .ok_or("Distance constraint requires value")?;

                    let x1 = x[*dof_map.get(&(p1_id, Coord::X)).unwrap()];
                    let y1 = x[*dof_map.get(&(p1_id, Coord::Y)).unwrap()];
                    let x2 = x[*dof_map.get(&(p2_id, Coord::X)).unwrap()];
                    let y2 = x[*dof_map.get(&(p2_id, Coord::Y)).unwrap()];

                    let dx = x2 - x1;
                    let dy = y2 - y1;
                    let dist = (dx * dx + dy * dy).sqrt();

                    // Residual: dist - target
                    residuals[i] = dist - target_dist;

                    // Jacobian: ∂r/∂x1, ∂r/∂y1, ∂r/∂x2, ∂r/∂y2
                    if dist > 1e-12 {
                        jacobian[(i, *dof_map.get(&(p1_id, Coord::X)).unwrap())] = -dx / dist;
                        jacobian[(i, *dof_map.get(&(p1_id, Coord::Y)).unwrap())] = -dy / dist;
                        jacobian[(i, *dof_map.get(&(p2_id, Coord::X)).unwrap())] = dx / dist;
                        jacobian[(i, *dof_map.get(&(p2_id, Coord::Y)).unwrap())] = dy / dist;
                    }
                }
                "coincident" => {
                    // Two points must overlap (2 residuals: dx=0, dy=0)
                    // ... implementation ...
                }
                "horizontal" => {
                    // Line must be horizontal (y1 - y2 = 0)
                    // ... implementation ...
                }
                "perpendicular" => {
                    // Dot product of direction vectors = 0
                    // ... implementation ...
                }
                _ => {
                    return Err(format!("Unknown constraint type: {}", constraint.constraint_type));
                }
            }
        }

        Ok((residuals, jacobian))
    }

    fn compute_damping_factor(
        &self,
        residuals: &DVector<f64>,
        jacobian: &DMatrix<f64>,
        dx: &DVector<f64>,
    ) -> f64 {
        // Simple line search: reduce alpha if update increases residual
        let current_norm = residuals.norm();
        let predicted_norm = (residuals + jacobian * dx).norm();

        if predicted_norm < current_norm {
            1.0  // Full step
        } else {
            0.5  // Half step (damping)
        }
    }

    fn build_solution(
        &self,
        sketch: &crate::SketchData,
        x: &DVector<f64>,
        dof_map: &HashMap<(u32, Coord), usize>,
    ) -> SolvedSketch {
        let mut solved_positions = HashMap::new();

        for entity in &sketch.entities {
            if let crate::SketchEntity::Point { id, .. } = entity {
                let px = x[*dof_map.get(&(*id, Coord::X)).unwrap()];
                let py = x[*dof_map.get(&(*id, Coord::Y)).unwrap()];
                solved_positions.insert(*id, (px, py));
            }
        }

        SolvedSketch {
            positions: solved_positions,
        }
    }
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
enum Coord {
    X,
    Y,
}

pub struct SolvedSketch {
    pub positions: HashMap<u32, (f64, f64)>,  // entity_id -> (x, y)
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–4)

**Day 1: Project initialization and Rust backend scaffold**
```bash
cargo init cadforge-api
cd cadforge-api
cargo add axum tokio serde serde_json sqlx uuid chrono tower-http jsonwebtoken bcrypt
cargo add --features postgres,runtime-tokio-rustls,uuid,chrono,json sqlx
cargo add aws-sdk-s3 redis anyhow tracing tracing-subscriber
```
- `src/main.rs` — Axum app with CORS, tracing, graceful shutdown
- `src/config.rs` — Environment-based configuration (DATABASE_URL, JWT_SECRET, S3_BUCKET, REDIS_URL, AI_API_URL)
- `src/state.rs` — AppState struct (PgPool, Redis, S3Client, WebSocket broadcaster)
- `src/error.rs` — ApiError enum with IntoResponse
- `Dockerfile` — Multi-stage build (builder + runtime)
- `docker-compose.yml` — PostgreSQL, Redis, MinIO (S3-compatible)

**Day 2: Database schema and migrations**
- `migrations/001_initial.sql` — All 10 tables: users, organizations, org_members, projects, versions, exports, components, ai_requests, collab_sessions, usage_records
- `src/db/mod.rs` — Database pool initialization
- `src/db/models.rs` — All SQLx structs with FromRow derives
- Run `sqlx migrate run` to apply schema
- Seed script for initial component library (ISO metric hex bolts M3-M24, DIN 125 washers, ISO 7089 nuts, 6000-series ball bearings)

**Day 3: Authentication system**
- `src/auth/mod.rs` — JWT token generation and validation middleware
- `src/auth/oauth.rs` — Google and GitHub OAuth 2.0 flow handlers
- `src/api/handlers/auth.rs` — Register, login, OAuth callback, refresh token, me endpoints
- Password hashing with bcrypt (cost factor 12)
- JWT with 24h access token + 30d refresh token
- Auth middleware that extracts Claims from Authorization header

**Day 4: User and project CRUD**
- `src/api/handlers/users.rs` — Get profile, update profile, update preferences, delete account
- `src/api/handlers/projects.rs` — Create, list, get, update feature tree, delete, fork project
- `src/api/handlers/orgs.rs` — Create org, invite member, list members, remove member, update role
- `src/api/router.rs` — All route definitions with auth middleware
- Integration tests: auth flow, project CRUD, authorization checks, optimistic locking for feature tree updates

### Phase 2 — CAD Kernel Core + WASM Build (Days 5–14)

**Day 5: OpenCascade FFI bindings setup**
- `occt-sys/` — New Rust crate for unsafe OCCT C++ bindings
- Install OpenCascade 7.8.0 from source with `-DBUILD_LIBRARY_TYPE=Static`
- Generate Rust bindings using `bindgen` for key OCCT classes: `TopoDS_Shape`, `BRepBuilderAPI_MakeBox`, `BRepPrimAPI_MakePrism`, `BRepFilletAPI_MakeFillet`
- `occt-sys/build.rs` — Link static OCCT libraries
- Minimal test: create a box in Rust via FFI and verify shape is valid

**Day 6: Safe Rust wrapper for B-Rep primitives**
- `cad-kernel/src/occt_wrapper.rs` — Safe Rust API wrapping unsafe OCCT calls
- `OcctShape` struct (owns `TopoDS_Shape` with proper Drop impl)
- `OcctBuilder` trait for feature builders
- Implement `make_box`, `make_cylinder`, `make_sphere` functions
- Unit tests: create primitives, verify face/edge counts

**Day 7: Sketch-to-wire conversion**
- `cad-kernel/src/sketch_to_wire.rs` — Convert 2D sketch entities (lines, arcs, circles) to 3D `TopoDS_Wire`
- Handle open and closed wire loops
- Support for inner loops (islands) in sketch profiles
- Tests: rectangle sketch → wire with 4 edges, circle sketch → wire with 1 edge

**Day 8: Extrude and revolve feature builders**
- `cad-kernel/src/feature_builder/extrude.rs` — `ExtrudeBuilder` using `BRepPrimAPI_MakePrism`
- Support blind, through-all, and mid-plane modes
- `cad-kernel/src/feature_builder/revolve.rs` — `RevolveBuilder` using `BRepPrimAPI_MakeRevol`
- Tests: extrude rectangle to box, revolve arc to sphere

**Day 9: Boolean operations**
- `cad-kernel/src/occt_wrapper.rs` — `boolean_union`, `boolean_subtract`, `boolean_intersect` using `BRepAlgoAPI_Fuse`, `BRepAlgoAPI_Cut`, `BRepAlgoAPI_Common`
- Handle topology cleaning after boolean ops
- Tests: box + cylinder union, cylinder - sphere subtract

**Day 10: Fillet and chamfer**
- `cad-kernel/src/feature_builder/fillet.rs` — `FilletBuilder` using `BRepFilletAPI_MakeFillet`
- Edge selection by index
- Constant and variable-radius fillet support
- `cad-kernel/src/feature_builder/chamfer.rs` — `ChamferBuilder` using `BRepFilletAPI_MakeChamfer`
- Tests: box with filleted edges, chamfered cube corners

**Day 11: 2D constraint solver implementation**
- `cad-kernel/src/sketch_solver.rs` — Newton-Raphson solver for geometric constraints
- Support constraints: distance, angle, coincident, horizontal, vertical, parallel, perpendicular, tangent, equal
- DOF analysis and under-constrained detection
- Tests: constrained rectangle (4 lines + 4 perpendicular + 2 distance = fully constrained), over-constrained detection

**Day 12: Tessellation for rendering**
- `cad-kernel/src/tessellator.rs` — Convert B-Rep to triangle mesh using `BRepMesh_IncrementalMesh`
- Extract vertices, normals, triangle indices, and edge indices
- Configurable linear deflection (mesh quality parameter)
- Output as `TessellatedMesh` struct for WebGPU consumption
- Tests: tessellate box, verify vertex count and triangle topology

**Day 13: WASM compilation pipeline**
- `cad-kernel/Cargo.toml` — Configure for `wasm32-unknown-unknown` target
- Compile OCCT to WASM using Emscripten: `emcc -s WASM=1 -s ALLOW_MEMORY_GROWTH=1 -O3`
- Link Rust wrapper with OCCT WASM via `occt-sys` build script
- `cad-kernel/src/lib.rs` — Export `CadKernel` class to JavaScript via `wasm-bindgen`
- Build with `wasm-pack build --target web --release`
- Optimize WASM with `wasm-opt -Oz` (target <15MB gzipped)

**Day 14: WASM integration testing**
- `cad-kernel/tests/wasm_test.html` — Simple HTML page that loads WASM and tests all operations
- Test full workflow: solve sketch → extrude → fillet → tessellate → render in canvas
- Benchmark WASM vs native: extrude 100 boxes, measure time
- Performance target: <50ms for extrude + fillet on typical part

### Phase 3 — Frontend Sketch Editor (Days 15–20)

**Day 15: React project setup**
```bash
npm create vite@latest cadforge-frontend -- --template react-ts
cd cadforge-frontend
npm install zustand react-router-dom @tanstack/react-query axios
npm install rbush  # R-tree for spatial indexing
npm install @types/rbush
```
- `src/App.tsx` — Routing: home, login, projects list, editor
- `src/stores/authStore.ts` — Zustand store for auth state (token, user)
- `src/stores/projectStore.ts` — Zustand store for current project (feature tree, selected feature)
- `src/api/client.ts` — Axios client with JWT interceptor

**Day 16: SVG sketch canvas with pan/zoom**
- `src/components/SketchCanvas.tsx` — SVG canvas with pan (drag background) and zoom (wheel)
- Coordinate transform: screen → world coordinates
- Grid rendering with adaptive density based on zoom level
- Snap-to-grid toggle
- Tests: pan/zoom smooth, coordinates correct

**Day 17: Sketch entity rendering and selection**
- `src/components/sketch/LineEntity.tsx` — Render line as SVG `<line>` with hover and selection states
- `src/components/sketch/CircleEntity.tsx` — Render circle as SVG `<circle>`
- `src/components/sketch/ArcEntity.tsx` — Render arc as SVG `<path>` with arc command
- R-tree integration: index all entities for O(log n) hit testing
- Click to select entity (highlight in orange), drag to move
- Tests: select entity, multi-select with Shift

**Day 18: Sketch drawing tools**
- `src/tools/LineTool.ts` — Click-click to draw line, snap to existing points
- `src/tools/CircleTool.ts` — Click center, drag radius
- `src/tools/RectangleTool.ts` — Click-drag to draw rectangle (4 lines + constraints)
- Tool state machine: idle → drawing → complete
- Undo/redo stack for entity creation
- Tests: draw line, undo, redo

**Day 19: Constraint creation and visualization**
- `src/components/sketch/ConstraintMarker.tsx` — Render constraint icons (⊥ for perpendicular, ∥ for parallel, = for equal)
- `src/tools/DistanceConstraintTool.ts` — Select two points, enter distance value, create constraint
- `src/tools/AngleConstraintTool.ts` — Select two lines, enter angle value
- Auto-constraint inference: when drawing line near another line, suggest parallel/perpendicular
- Tests: create distance constraint, verify icon rendered

**Day 20: Sketch solver integration**
- Load WASM module: `import init, { CadKernel } from '../wasm/cadforge_kernel'`
- `src/hooks/useSketchSolver.ts` — React hook that calls `kernel.solve_sketch()` on constraint changes
- Debounce solver calls (100ms) to avoid solving on every drag event
- Animate entity positions when solver updates (smooth transition)
- Error handling: display solver errors (over-constrained, singular Jacobian)
- Tests: solve fully-constrained rectangle, verify positions

### Phase 4 — 3D Viewport + Feature Tree (Days 21–26)

**Day 21: WebGPU Three.js renderer setup**
```bash
npm install three @types/three
```
- `src/components/Viewport3D.tsx` — Three.js scene with WebGPU renderer (fallback to WebGL2)
- Orbit camera controls (pan with right-drag, rotate with left-drag, zoom with wheel)
- Grid plane and axis triad (X red, Y green, Z blue)
- Lighting: ambient + 2 directional lights for good edge visibility
- Tests: render empty scene, camera controls responsive

**Day 22: Mesh rendering from tessellated B-Rep**
- `src/hooks/useCadKernel.ts` — Hook that manages WASM `CadKernel` instance
- When feature tree updates: rebuild all features in WASM, tessellate, update Three.js geometry
- `BufferGeometry` with `position`, `normal`, `index` attributes from WASM `TessellatedMesh`
- PBR material with metalness=0.1, roughness=0.6
- Tests: extrude sketch, verify mesh appears in viewport

**Day 23: Edge rendering and selection highlighting**
- Separate `LineSegments` geometry for edges (from `TessellatedMesh.edges`)
- Edge material: black with `linewidth=1`
- On feature select: re-render selected feature's edges with orange material, `linewidth=3`
- Face selection: raycast on click, highlight selected face with semi-transparent overlay
- Tests: select feature, edges highlighted

**Day 24: Feature tree UI**
- `src/components/FeatureTree.tsx` — Collapsible tree list of all features
- Drag-to-reorder features (updates feature tree order, triggers rebuild)
- Right-click context menu: Edit, Suppress, Delete
- Feature icons: 📐 sketch, ⬆️ extrude, 🔄 revolve, ⭕ fillet
- Double-click feature to edit parameters in sidebar
- Tests: reorder features, suppress extrude

**Day 25: Feature parameter editing**
- `src/components/FeatureEditor.tsx` — Sidebar panel with inputs for selected feature
- Extrude: depth slider, mode dropdown (blind/through-all/mid-plane)
- Fillet: radius slider, edge selection (click edges in viewport to add to list)
- Real-time preview: as slider changes, rebuild feature and update viewport
- Debounce rebuilds (200ms) for smooth interaction
- Tests: change extrude depth, verify mesh updates

**Day 26: Feature tree rebuild logic**
- `src/engine/featureRebuild.ts` — Rebuild entire model by replaying features in order
- Dependency graph: if feature N depends on feature M, rebuild M first
- Incremental rebuild optimization: if feature K unchanged, reuse cached shape
- Error handling: if feature fails to build, show error in feature tree (red icon), stop rebuild at that point
- Tests: modify early feature, verify dependent features rebuild

### Phase 5 — AI-Assisted Features (Days 27–31)

**Day 27: Python FastAPI AI service setup**
```bash
mkdir ai-service && cd ai-service
python -m venv venv
source venv/bin/activate
pip install fastapi uvicorn openai pillow numpy opencv-python
```
- `main.py` — FastAPI app with endpoints: `/nl-to-cad`, `/sketch-to-cad`, `/design-intent`
- `src/nl_to_cad.py` — Parse natural language ("create M8 hex bolt 40mm long") via GPT-4o, return feature tree JSON
- `src/sketch_recognition.py` — Take uploaded sketch image, use GPT-4o Vision to extract entities and constraints
- Health check endpoint, CORS for frontend
- Deploy to AWS Lambda + API Gateway (serverless for cost optimization)

**Day 28: Natural language to CAD pipeline**
- `ai-service/src/nl_to_cad.py` — GPT-4o prompt engineering:
  - System prompt: "You are a CAD assistant. Convert user descriptions into parametric feature trees. Output JSON only."
  - Few-shot examples: "hex bolt" → feature tree with extrude + chamfer + thread
  - Parse GPT response, validate feature tree structure
- Component library integration: search for standard parts ("M8 bolt" → fetch from components table)
- Tests: "create a box 10x20x30mm" → feature tree with sketch + extrude

**Day 29: Sketch-to-CAD image recognition**
- `ai-service/src/sketch_recognition.py` — Image preprocessing:
  - Convert to grayscale, edge detection (Canny), line detection (Hough transform)
  - Send preprocessed image + original to GPT-4o Vision with prompt: "Extract lines, circles, arcs from this sketch. Return JSON."
- Parse GPT response into `SketchEntity[]` and `SketchConstraint[]`
- Infer constraints from geometric relationships (parallel lines, perpendicular corners)
- Tests: upload hand-drawn rectangle photo → constrained sketch

**Day 30: Design intent suggestion**
- `ai-service/src/design_intent.py` — Analyze current feature tree, suggest next features
  - After extrude: suggest "Add fillets to sharp edges?" with auto-selected edges
  - After cylindrical boss: suggest "Add bolt hole pattern?"
- Generate feature tree delta (new features to append)
- Frontend UI: show suggestions in sidebar, click to accept (appends to tree)
- Tests: extrude cube → suggest fillet all edges

**Day 31: AI request tracking and rate limiting**
- `src/api/handlers/ai.rs` — Rust backend proxies AI requests to Python service
- Insert record into `ai_requests` table with tokens used
- Plan enforcement: Free plan = 10 AI queries/month, Pro = 100/month
- Rate limiting per user: max 5 requests/minute
- Return usage stats to frontend: "You have used 7 of 100 AI queries this month"
- Tests: exceed rate limit, verify 429 error

### Phase 6 — Export + Version History (Days 32–36)

**Day 32: STEP export (server-side)**
- `geometry-worker/` — New Rust binary for heavy geometry jobs
- `geometry-worker/src/step_exporter.rs` — Deserialize B-Rep from S3, convert to STEP AP214 using OCCT `STEPControl_Writer`
- Redis job queue: frontend requests export → enqueue job → worker processes
- Write STEP file to S3, update `exports` table with file URL
- Tests: export simple part, verify STEP file valid in FreeCAD

**Day 33: STL and 3MF export**
- `geometry-worker/src/stl_exporter.rs` — Tessellate B-Rep, write binary STL using OCCT `StlAPI_Writer`
- `geometry-worker/src/3mf_exporter.rs` — Write 3MF XML + mesh payload (zip archive)
- Export settings: mesh quality (linear deflection 0.01 - 1.0mm), coordinate system (Y-up vs Z-up)
- Tests: export to STL, import in Cura, verify printable

**Day 34: Version history and diff**
- `src/api/handlers/versions.rs` — List versions for project, get version details, restore version
- Frontend `src/components/VersionHistory.tsx` — Timeline view of versions with commit messages
- Click version → load feature tree snapshot → render in viewport (read-only)
- Version diff: compare two versions, highlight changed features (added in green, removed in red, modified in yellow)
- Tests: create 3 versions, diff v1 vs v3

**Day 35: Export UI and download**
- `src/components/ExportDialog.tsx` — Modal with format selection (STEP/STL/3MF), settings inputs
- Submit export request → poll `exports` table every 2s for status update
- Show progress spinner, download link when completed
- Tests: export STEP, download file

**Day 36: Version restore and forking**
- Restore version: copy version's feature tree to project, create new version with message "Restored from v5"
- Fork project: duplicate project row with `forked_from` reference, copy feature tree and geometry
- Tests: restore old version, fork public project

### Phase 7 — Collaboration + Billing (Days 37–40)

**Day 37: WebSocket real-time updates**
- `src/websocket/mod.rs` — Axum WebSocket handler with room-based broadcasting
- On project open: join room `project:{id}`, send cursor position every 500ms
- On feature tree update: broadcast `feature_tree_updated` event to all room members
- Frontend `src/hooks/useCollaboration.ts` — WebSocket client that renders remote cursors, shows who's editing which feature
- Tests: open same project in 2 browsers, verify cursor presence

**Day 38: Feature locking and conflict resolution**
- When user edits feature: acquire lock in `collab_sessions.locked_features` (Redis TTL 30s, auto-refresh while editing)
- If another user tries to edit locked feature: show "User Alice is editing this feature"
- On save: check version number (optimistic lock), merge if possible, else show conflict dialog
- Tests: concurrent edits, verify conflict detection

**Day 39: Stripe billing integration**
- `src/billing/mod.rs` — Stripe webhook handler for `checkout.session.completed`, `customer.subscription.updated`, `invoice.payment_failed`
- Update `users.plan` and `stripe_subscription_id` on subscription events
- `src/api/handlers/billing.rs` — Create checkout session, customer portal session
- Frontend `src/components/UpgradeDialog.tsx` — Pricing table, "Upgrade to Pro" button → Stripe Checkout
- Tests: complete checkout in Stripe test mode, verify plan upgraded

**Day 40: Usage enforcement and limits**
- Middleware: check user plan before AI requests, exports, collaboration
- Free plan limits: 3 projects, 10 AI queries/month, no real-time collab, STEP export disabled
- Pro plan limits: unlimited projects, 100 AI queries/month, STEP export enabled
- Show upgrade prompt when hitting limits
- Tests: free user tries to create 4th project, verify blocked with upgrade prompt

### Phase 8 — Polish + Production Deploy (Days 41–42)

**Day 41: Performance optimization and testing**
- WASM bundle size optimization: strip debug symbols, aggressive tree-shaking → target <12MB gzipped
- Lazy-load WASM module only when entering editor
- Three.js frustum culling for large assemblies (>1000 parts)
- IndexedDB caching for geometry blobs (avoid re-downloading from S3)
- Load testing: 100 concurrent users creating projects, measure API p95 latency (<200ms target)
- WASM crash reporting via Sentry (catch panics, send stack traces)

**Day 42: Production deployment**
- Backend: Docker images pushed to AWS ECR, deployed to ECS Fargate with ALB
- Frontend: Build with `npm run build`, deploy to S3 + CloudFront with gzip/brotli compression
- WASM bundle: Upload to S3, serve via CloudFront with `Cache-Control: max-age=31536000`
- Database: RDS PostgreSQL with automated backups, Multi-AZ for HA
- Redis: ElastiCache Redis cluster
- AI service: Deploy to Lambda with API Gateway, cold start optimization
- DNS: Route53 with `cadforge.io`, HTTPS via ACM certificate
- Monitoring: CloudWatch alarms for API errors >1%, p95 latency >500ms, WASM load failures >5%
- Seed production database with 500+ standard components (ISO/ANSI bolts, nuts, washers, bearings, gears)

---

## Validation Benchmarks

### Performance Targets

1. **WASM geometry kernel cold start**: <800ms from page load to first render (load 12MB WASM + initialize OCCT runtime)
2. **Parametric rebuild latency**: <150ms to rebuild typical part with 10 features (extrude + fillets + boolean) at 1920x1080 viewport
3. **Sketch solve iteration**: <20ms for constraint solver convergence on 50-entity sketch with 40 constraints
4. **STEP export throughput**: <5 seconds for server-side export of 500KB B-Rep to STEP AP214 file
5. **Concurrent user capacity**: Support 500 concurrent active editors with <200ms API p95 latency on 4x ECS Fargate tasks (2 vCPU, 4GB RAM each)

### Functional Validation

1. **Parametric integrity**: Modify early feature (sketch dimension) → verify all dependent features rebuild correctly without topology naming errors (test with 20-feature part)
2. **STEP export compatibility**: Export 10 different parts → import into SolidWorks 2023, Fusion 360, FreeCAD 0.21 → verify zero import errors and correct geometry
3. **AI sketch recognition accuracy**: Test with 100 hand-drawn sketches (rectangles, circles, L-shapes) → measure >85% correct entity detection and >70% correct constraint inference
4. **Browser compatibility**: Test core workflows (create project, sketch, extrude, export) on Chrome 120+, Firefox 120+, Safari 17+, Edge 120+ → verify zero critical bugs
5. **Multi-user collaboration**: 5 users edit same project simultaneously → verify cursor presence visible, feature locks enforced, no geometry corruption after concurrent saves

---

## Post-MVP Roadmap

- Loft between multiple profiles with guide curves
- Boundary surface from 2-4 edge curves
- Sweep along path with twist and scale
- Surface offset and thicken
- Target users: Industrial designers, consumer product engineers

### v1.2 — Assembly Mode (Weeks 10-13)
- Standard mates: coincident, concentric, distance, angle
- Mate diagnostics (over-constrained, unsolved)
- Assembly interference detection
- Exploded view generation
- Bill of Materials (BOM) with balloon annotations
- Target: Multi-part mechanical assemblies

### v1.3 — Simulation Preview (Weeks 14-18)
- Linear static FEA with mesh generation
- Thermal conduction analysis
- Modal analysis (natural frequencies)
- Motion simulation (kinematics)
- Results as color contour overlays on model
- Target: Mechanical engineers validating designs

### v1.4 — Git-Like Branching (Weeks 19-22)
- Branch creation from any version
- Visual 3D diff showing added/removed/modified features
- Merge workflows with conflict resolution UI
- Pull request reviews with comments anchored to geometry
- Target: Teams with complex design approval workflows

### v1.5 — Manufacturing Integration (Weeks 23-26)
- Direct API integration with Xometry, Protolabs, JLCPCB
- One-click quote requests with uploaded STEP files
- DFM checks: undercut detection, wall thickness analysis, draft angle validation
- Sheet metal mode: flat patterns, bend tables, K-factor
- 2.5D CNC toolpath generation with G-code export
- Target: Hardware startups going from design to production

---

## Risk Mitigation

### Technical Risks

**Risk 1: OCCT WASM compilation instability**
- *Likelihood*: Medium — OCCT's C++ codebase has global state that can cause issues in WASM
- *Mitigation*: Build minimal OCCT subset (only B-Rep, no visualization), extensive WASM stability testing, fallback to server-side geometry ops if WASM crashes
- *Contingency*: Switch to Truck (pure Rust CAD kernel) if OCCT proves unworkable, accept limited feature set in MVP

**Risk 2: Sketch solver convergence failures on complex constraints**
- *Likelihood*: Medium — Over-constrained or conflicting constraints common in user-drawn sketches
- *Mitigation*: Robust diagnostics showing which constraints conflict, suggest auto-removal, implement constraint prioritization (keep dimensions, relax geometric constraints)
- *Contingency*: Offer "manual mode" where solver only applies dimensions, user positions geometry manually

**Risk 3: WebGPU browser support too limited for target users**
- *Likelihood*: Low — WebGPU in Chrome 113+, Edge 113+, Safari 18+ covers 70%+ of users
- *Mitigation*: Automatic fallback to WebGL2 renderer (supported in 95%+ browsers), accept 2x slower edge rendering
- *Contingency*: If WebGPU adoption stalls, optimize WebGL2 renderer with instancing and geometry batching

**Risk 4: STEP export compatibility issues with commercial CAD**
- *Likelihood*: Medium — STEP AP214 has vendor-specific interpretations
- *Mitigation*: Test exports against SolidWorks, Fusion 360, FreeCAD in CI pipeline, maintain export compatibility matrix
- *Contingency*: Offer alternative export formats (IGES, Parasolid if licensing permits), partner with commercial CAD vendors for validated export pipelines

**Risk 5: AI sketch recognition accuracy too low for production use**
- *Likelihood*: Medium — Hand-drawn sketches vary widely in quality
- *Mitigation*: Extensive training dataset (1000+ labeled sketches), user feedback loop to improve model, manual correction UI for AI-generated sketches
- *Contingency*: De-emphasize AI features in marketing if accuracy <70%, focus on traditional parametric modeling workflow

### Business Risks

**Risk 6: SolidWorks/Autodesk respond with aggressive browser CAD push**
- *Likelihood*: Low — Incumbents focused on desktop, cloud efforts (Fusion, Onshape) already established
- *Mitigation*: Move fast on AI differentiation (sketch-to-CAD, design intent), build open component library moat, focus on underserved SMB segment
- *Contingency*: Pivot to B2B embedded CAD SDK if consumer market becomes uncompetitive

**Risk 7: Free tier abused for crypto mining or spam**
- *Likelihood*: Low — CAD editor doesn't expose arbitrary compute
- *Mitigation*: Rate limiting on AI requests, CAPTCHA on signup, monitor for anomalous usage patterns (100s of projects created/deleted)
- *Contingency*: Require email verification for free tier, add usage caps (10 projects max, 50 exports/month)

**Risk 8: Users refuse to pay for CAD tool they "already have" (Fusion/FreeCAD)**
- *Likelihood*: Medium — Price sensitivity high in target segment
- *Mitigation*: Emphasize AI workflow speed ("design in minutes, not hours"), real-time collab for teams, browser accessibility (no install, works on any OS)
- *Contingency*: Lower Pro tier to $19/mo, add freemium credits for AI queries (5 free/month), offer annual discounts

---

## Success Metrics

### MVP (Week 6)
- 50 beta users designing real parts (not just testing)
- 200+ parts created across all users
- 80%+ of sketches solve without constraint errors
- <5% WASM crash rate across Chrome/Firefox/Safari
- 30+ STEP exports successfully imported into SolidWorks/Fusion with zero errors

### Month 3
- 500 registered users
- 50 paying customers (Pro or Team tier)
- $2,000 MRR
- 70%+ user retention week-over-week for active designers
- 1,000+ component library downloads

### Month 6
- 2,000 registered users
- 200 paying customers
- $10,000 MRR
- Partnership with 1+ manufacturing vendor (Xometry, Protolabs) for integrated quotes
- Featured in design/engineering community (Hackaday, Hacker News front page, ProductHunt top 5)

### Month 12
- 10,000 registered users
- 800 paying customers ($24K MRR, $288K ARR run rate)
- Educational partnerships with 3+ universities using CADForge in coursework
- API partnerships with 2+ hardware startups embedding CADForge in their workflows
- 95% customer satisfaction (CSAT) among paying users

---

## Team & Resources

### Founding Team (MVP)
- **Lead Engineer (Full-Stack + WASM)**: Rust backend, OCCT integration, WASM compilation, React frontend — full-time, 6 weeks
- **Optional: Part-time CAD domain expert**: Consult on parametric modeling UX, constraint solver algorithms, STEP export validation — 10 hrs/week, 6 weeks

### Post-MVP (Months 1-6)
- **Additional Frontend Engineer**: Three.js optimization, sketch editor polish, collaboration UI — start Month 2
- **Part-time AI/ML Engineer**: Improve sketch recognition accuracy, expand NL-to-CAD coverage — 20 hrs/week, start Month 3
- **Customer Success / Community Manager**: Onboard users, create tutorials, manage Discord/forums — start Month 4

### Infrastructure Costs (Monthly)

**MVP (Months 1-2, <100 users)**
- AWS ECS Fargate (2 tasks, 0.5 vCPU each): $30
- RDS PostgreSQL (db.t3.micro): $15
- ElastiCache Redis (cache.t3.micro): $12
- S3 + CloudFront (10GB storage, 100GB transfer): $5
- Lambda (AI service, 10K requests/month): $2
- **Total: ~$64/month**

**Growth (Months 3-6, 500-2000 users)**
- AWS ECS Fargate (4 tasks, 1 vCPU each): $120
- RDS PostgreSQL (db.t3.small, Multi-AZ): $60
- ElastiCache Redis (cache.t3.small): $35
- S3 + CloudFront (100GB storage, 1TB transfer): $30
- Lambda (AI service, 100K requests/month): $15
- Sentry (error tracking): $26/month
- **Total: ~$286/month**

**Scale (Month 12, 10K users)**
- AWS ECS Fargate (10 tasks, 2 vCPU each): $600
- RDS PostgreSQL (db.r5.large, Multi-AZ): $350
- ElastiCache Redis (cache.r5.large): $180
- S3 + CloudFront (1TB storage, 10TB transfer): $200
- Lambda (AI service, 1M requests/month): $120
- Monitoring + logging (Prometheus, Grafana Cloud, Sentry): $100
- **Total: ~$1,550/month**

### Development Tools
- GitHub (team plan): $4/user/month
- Figma (design): $15/user/month
- Stripe (payment processing): 2.9% + $0.30 per transaction
- OpenAI API (GPT-4o for AI features): $0.005/request estimated, $500/month budget at scale

---

## Open Questions & Decisions Needed

1. **OCCT licensing**: OpenCascade is LGPL 2.1, which allows commercial use but requires dynamic linking. WASM static linking may trigger copyleft. **Decision**: Consult IP lawyer, consider dual-license or commercial OCCT license ($5K-$15K/year).

2. **AI training data sourcing**: Need 1,000+ labeled hand-drawn sketches for training. **Options**: (a) Mechanical Turk for labeling, (b) partner with university CAD courses for student-drawn sketches, (c) synthetic sketch generation.

3. **Component library licensing**: Standard parts (ISO bolts, bearings) have no copyright, but vendor-specific models (TI op-amps, Analog Devices ADCs) may have redistribution restrictions. **Decision**: Only include vendor-provided models with explicit redistribution permission, generate generic models for others.

4. **Export format priorities**: STEP is critical, but customers may need SolidWorks native (.sldprt) or Fusion native (.f3d). **Decision**: Start with STEP (universal), add SolidWorks export in v1.1 if high demand, Fusion export unlikely due to proprietary format.

5. **Freemium vs. free trial**: Free tier with limited features vs. 14-day Pro trial then paid. **Decision**: Generous free tier (3 projects, basic features) to build user base, trial-to-paid conversion typically <5% but free tier has viral growth potential.

---

## Appendix: Detailed API Specification

### Authentication Endpoints

```
POST /api/auth/register
Body: { email, password, name }
Response: { user, access_token, refresh_token }

POST /api/auth/login
Body: { email, password }
Response: { user, access_token, refresh_token }

POST /api/auth/oauth/google
Body: { code }
Response: { user, access_token, refresh_token }

POST /api/auth/oauth/github
Body: { code }
Response: { user, access_token, refresh_token }

POST /api/auth/refresh
Body: { refresh_token }
Response: { access_token }

GET /api/auth/me
Headers: Authorization: Bearer <token>
Response: { user }
```

### Project Endpoints

```
GET /api/projects
Headers: Authorization: Bearer <token>
Query: ?limit=100&offset=0&sort=updated_at
Response: { projects: [...], total }

POST /api/projects
Headers: Authorization: Bearer <token>
Body: { name, description?, project_type }
Response: { project }

GET /api/projects/:id
Headers: Authorization: Bearer <token>
Response: { project }

PATCH /api/projects/:id
Headers: Authorization: Bearer <token>
Body: { name?, description?, feature_tree?, version }
Response: { project }

DELETE /api/projects/:id
Headers: Authorization: Bearer <token>
Response: 204 No Content

POST /api/projects/:id/fork
Headers: Authorization: Bearer <token>
Body: { name }
Response: { project }
```

### Geometry Endpoints

```
POST /api/projects/:id/geometry/upload
Headers: Authorization: Bearer <token>
Body: multipart/form-data { geometry: binary BinOcaf file }
Response: { geometry_url }

GET /api/projects/:id/geometry
Headers: Authorization: Bearer <token>
Response: Redirect to S3 presigned URL
```

### Version Endpoints

```
GET /api/projects/:id/versions
Headers: Authorization: Bearer <token>
Response: { versions: [...] }

GET /api/projects/:id/versions/:version_number
Headers: Authorization: Bearer <token>
Response: { version }

POST /api/projects/:id/versions/restore
Headers: Authorization: Bearer <token>
Body: { version_number }
Response: { project }
```

### Export Endpoints

```
POST /api/projects/:id/exports
Headers: Authorization: Bearer <token>
Body: { format: "step" | "stl" | "3mf", settings: {...} }
Response: { export }

GET /api/exports/:id
Headers: Authorization: Bearer <token>
Response: { export }

GET /api/exports/:id/download
Headers: Authorization: Bearer <token>
Response: Redirect to S3 presigned URL
```

### Component Library Endpoints

```
GET /api/components
Query: ?category=fastener&search=M8&limit=50
Response: { components: [...], total }

GET /api/components/:id
Response: { component }

POST /api/components/:id/instantiate
Headers: Authorization: Bearer <token>
Body: { parameters: { length: 40, diameter: 8 } }
Response: { feature_tree }
```

### AI Endpoints

```
POST /api/ai/nl-to-cad
Headers: Authorization: Bearer <token>
Body: { prompt: "create a hex bolt M8 40mm long" }
Response: { feature_tree, entities: [...] }

POST /api/ai/sketch-to-cad
Headers: Authorization: Bearer <token>
Body: multipart/form-data { image: file }
Response: { entities: [...], constraints: [...] }

POST /api/ai/design-intent
Headers: Authorization: Bearer <token>
Body: { project_id, current_feature_tree }
Response: { suggestions: [{ description, feature_tree_delta }] }
```

### Billing Endpoints

```
POST /api/billing/checkout
Headers: Authorization: Bearer <token>
Body: { plan: "pro" | "team", interval: "month" | "year" }
Response: { checkout_url }

POST /api/billing/portal
Headers: Authorization: Bearer <token>
Response: { portal_url }

GET /api/billing/usage
Headers: Authorization: Bearer <token>
Query: ?period_start=2024-01-01&period_end=2024-01-31
Response: { usage_records: [...], totals: {...} }
```

### WebSocket Protocol

```
Connect: ws://api.cadforge.io/ws/projects/:id
Message on connect: { type: "join", user_id }

Client → Server:
{ type: "cursor_move", position: { x, y } }
{ type: "feature_lock", feature_id }
{ type: "feature_unlock", feature_id }

Server → Client:
{ type: "user_joined", user: {...} }
{ type: "user_left", user_id }
{ type: "cursor_update", user_id, position }
{ type: "feature_tree_updated", user_id, version }
{ type: "feature_locked", feature_id, user_id }
{ type: "feature_unlocked", feature_id }
```

---

## Appendix: WASM API Specification

### CadKernel JavaScript Interface

```typescript
import init, { CadKernel } from './wasm/cadforge_kernel.js';

// Initialize WASM module
await init();

// Create kernel instance
const kernel = new CadKernel();

// Solve 2D sketch
const sketchJson = JSON.stringify({
  entities: [
    { type: "Point", id: 1, x: 0, y: 0 },
    { type: "Point", id: 2, x: 100, y: 0 },
    { type: "Line", id: 3, start: 1, end: 2 }
  ],
  constraints: [
    { constraint_type: "horizontal", entity_ids: [3] },
    { constraint_type: "distance", entity_ids: [1, 2], value: 100 }
  ]
});

const solvedJson = kernel.solve_sketch(sketchJson);
const solved = JSON.parse(solvedJson);
// solved.positions = { 1: [0, 0], 2: [100, 0] }

// Extrude sketch
kernel.extrude(
  "feature-uuid-1",  // feature ID
  "sketch-uuid-0",   // sketch ID
  50.0,              // depth
  "blind"            // mode
);

// Apply fillet
kernel.fillet(
  "feature-uuid-2",  // feature ID
  "feature-uuid-1",  // parent ID
  5.0,               // radius
  [0, 1, 2, 3]       // edge indices
);

// Tessellate for rendering
const meshJson = kernel.tessellate("feature-uuid-2", 0.1);
const mesh = JSON.parse(meshJson);
// mesh = { vertices: Float32Array, normals: Float32Array, indices: Uint32Array }

// Serialize to binary
const binary = kernel.serialize();
localStorage.setItem('project-geometry', binary);

// Deserialize from binary
kernel.deserialize(binary);
```

---

## Appendix: Example Feature Tree JSON

```json
[
  {
    "id": "00000000-0000-0000-0000-000000000001",
    "feature_type": {
      "type": "Sketch",
      "plane": "XY"
    },
    "name": "Sketch1",
    "parameters": {
      "entities": [
        { "type": "Point", "id": 1, "x": 0, "y": 0 },
        { "type": "Point", "id": 2, "x": 50, "y": 0 },
        { "type": "Point", "id": 3, "x": 50, "y": 30 },
        { "type": "Point", "id": 4, "x": 0, "y": 30 },
        { "type": "Line", "id": 5, "start": 1, "end": 2 },
        { "type": "Line", "id": 6, "start": 2, "end": 3 },
        { "type": "Line", "id": 7, "start": 3, "end": 4 },
        { "type": "Line", "id": 8, "start": 4, "end": 1 }
      ],
      "constraints": [
        { "constraint_type": "horizontal", "entity_ids": [5] },
        { "constraint_type": "vertical", "entity_ids": [6] },
        { "constraint_type": "horizontal", "entity_ids": [7] },
        { "constraint_type": "vertical", "entity_ids": [8] },
        { "constraint_type": "distance", "entity_ids": [1, 2], "value": 50 },
        { "constraint_type": "distance", "entity_ids": [2, 3], "value": 30 }
      ]
    },
    "parent_ids": [],
    "is_suppressed": false
  },
  {
    "id": "00000000-0000-0000-0000-000000000002",
    "feature_type": {
      "type": "Extrude",
      "mode": "Blind",
      "depth": 20,
      "direction": 1
    },
    "name": "Extrude1",
    "parameters": {},
    "parent_ids": ["00000000-0000-0000-0000-000000000001"],
    "is_suppressed": false
  },
  {
    "id": "00000000-0000-0000-0000-000000000003",
    "feature_type": {
      "type": "Fillet",
      "radius": 3,
      "edge_ids": [0, 1, 2, 3, 4, 5, 6, 7]
    },
    "name": "Fillet1",
    "parameters": {},
    "parent_ids": ["00000000-0000-0000-0000-000000000002"],
    "is_suppressed": false
  }
]
```

This feature tree represents: a rectangular sketch (50mm × 30mm), extruded 20mm upward, with 3mm fillets on all edges.
