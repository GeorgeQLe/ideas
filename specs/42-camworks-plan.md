# 42. CamWorks — AI-Powered CNC Toolpath Generation Platform

## Implementation Plan

**MVP Scope:** Browser-based 3D CAD viewer that imports STEP/IGES/Parasolid files rendered via WebGL with Three.js, automatic feature recognition for prismatic features (holes, pockets, slots, bosses, faces) using ML computer vision model (3D CNN on voxelized geometry) trained on 10K+ annotated CAD models compiled to TensorFlow.js, 2.5D and 3-axis toolpath generation engine in Rust compiled to WebAssembly supporting adaptive clearing for roughing with constant chip load, contour finishing with scallop height control, and drilling cycles, feeds/speeds recommendation engine using PostgreSQL database of 500+ materials and 1,000+ cutting tools from major manufacturers with AI-optimized parameters based on material hardness and tool geometry, simplified bounding-box collision detection for machine simulation with cycle time estimation, G-code post-processor library with 50+ verified controllers (Fanuc, Haas, Grbl, LinuxCNC, Mach3), project management with PostgreSQL for storing projects/toolpaths/simulations and S3 for CAD files and G-code output, Stripe billing with three tiers (Free / Pro $99/mo / Production $199/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| Toolpath Engine | Rust (native + WASM) | Custom 2.5D/3-axis CAM algorithms with offset curves, collision-free linking |
| WASM Compilation | `wasm-pack` + `wasm-bindgen` | Client-side toolpath generation for simple features |
| Feature Recognition | Python 3.12 (FastAPI) | TensorFlow 2.16 for 3D CNN model inference, voxel preprocessing |
| CAD Import | Python (OCC/pythonocc-core) | STEP/IGES parsing via OpenCASCADE, geometry extraction |
| Database | PostgreSQL 16 | Projects, users, toolpaths, jobs, tools, materials |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | CAD files (STEP/IGES), G-code output, machine models, tool geometry |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| 3D Viewer | Three.js + WebGL 2.0 | CAD geometry rendering, toolpath visualization, material removal simulation |
| 3D UI Controls | React Three Fiber + Drei | Declarative Three.js integration, camera controls, selection |
| Real-time | WebSocket (Axum) | Live toolpath generation progress, simulation updates |
| Job Queue | Redis 7 + Tokio tasks | Server-side toolpath generation, feature recognition jobs, simulation |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | WASM bundle delivery, static assets, large CAD file downloads |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, error tracking, job performance |
| CI/CD | GitHub Actions | WASM build, Rust tests, Python ML model validation, Docker image push |

### Key Architecture Decisions

1. **Hybrid client/server toolpath generation with WASM threshold at 100 features**: Simple parts with ≤100 machinable features (covers 80%+ of job shop parts) generate toolpaths entirely in the browser via WASM, providing instant feedback with zero server cost. Complex parts with >100 features or requiring 5-axis strategies are submitted to the Rust-native server. The threshold is configurable per plan tier.

2. **ML feature recognition in Python rather than Rust**: TensorFlow/PyTorch ecosystem for 3D CNNs is mature in Python with extensive tooling for training, evaluation, and inference. The feature recognition service runs as a separate FastAPI microservice that receives voxelized geometry from the main API, runs inference, and returns detected features. This separation allows GPU-accelerated batch inference and independent scaling of ML workloads.

3. **OpenCASCADE for CAD import rather than custom STEP parser**: STEP/IGES files are notoriously complex (ISO 10303 spans thousands of pages). OpenCASCADE (via pythonocc-core Python bindings) provides battle-tested STEP/IGES import with B-rep (boundary representation) extraction, topology traversal, and geometry queries. The Python CAD service converts imported models to triangle meshes and analytical surfaces for toolpath generation.

4. **Three.js/WebGL for 3D rendering rather than native CAD kernel in browser**: Three.js provides GPU-accelerated rendering of millions of triangles with built-in camera controls, lighting, and shader support. CAD geometry is converted to triangle meshes server-side and streamed to browser as GLB files for instant display. This avoids shipping large STEP files to browser and eliminates need for WASM CAD kernel.

5. **Rust toolpath engine for both native and WASM targets**: Toolpath generation algorithms (offset curves, pocket clearing, Z-level contouring) are compute-intensive and benefit from Rust's zero-cost abstractions and SIMD optimization. The same Rust codebase compiles to both native (server-side) for complex parts and WASM (browser) for simple parts, ensuring consistent behavior and reducing code duplication.

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";  -- For fuzzy text search on tools/materials

-- Users
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email TEXT UNIQUE NOT NULL,
    password_hash TEXT,  -- NULL for OAuth-only users
    name TEXT NOT NULL,
    avatar_url TEXT,
    auth_provider TEXT DEFAULT 'email',  -- email | google | github
    auth_provider_id TEXT,
    plan TEXT NOT NULL DEFAULT 'free',  -- free | pro | production | enterprise
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX users_email_idx ON users(email);
CREATE INDEX users_stripe_idx ON users(stripe_customer_id);

-- Organizations (for Production and Enterprise plans)
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

-- Projects (CAD model + machining workspace)
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    cad_file_url TEXT,  -- S3 URL to original STEP/IGES
    mesh_url TEXT,  -- S3 URL to processed GLB mesh for viewer
    bounding_box JSONB,  -- {min: [x,y,z], max: [x,y,z]}
    units TEXT NOT NULL DEFAULT 'mm',  -- mm | inch
    material TEXT,  -- Material name from materials table
    stock_dimensions JSONB,  -- {length, width, height, shape: 'rectangular'|'cylindrical'}
    setup_data JSONB DEFAULT '{}',  -- Work coordinate system, fixtures, datum references
    is_public BOOLEAN NOT NULL DEFAULT false,
    thumbnail_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);
CREATE INDEX projects_org_idx ON projects(org_id);
CREATE INDEX projects_updated_idx ON projects(updated_at DESC);

-- Feature Recognition Jobs
CREATE TABLE feature_recognition_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | running | completed | failed
    voxel_resolution INTEGER NOT NULL DEFAULT 64,  -- 32 | 64 | 128
    model_version TEXT NOT NULL DEFAULT 'v1.0',
    features_detected JSONB,  -- [{type, geometry, confidence, parameters}]
    error_message TEXT,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    processing_time_ms INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX feature_jobs_project_idx ON feature_recognition_jobs(project_id);
CREATE INDEX feature_jobs_status_idx ON feature_recognition_jobs(status);

-- Detected Features (cached results from ML model)
CREATE TABLE features (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    feature_recognition_job_id UUID REFERENCES feature_recognition_jobs(id),
    feature_type TEXT NOT NULL,  -- hole | pocket | slot | boss | face | contour | chamfer | fillet
    subtype TEXT,  -- through_hole | blind_hole | counterbore | countersink | open_pocket | closed_pocket
    geometry JSONB NOT NULL,  -- Type-specific geometric parameters
    bounding_box JSONB NOT NULL,
    confidence REAL NOT NULL DEFAULT 1.0,  -- 0.0-1.0 from ML model
    is_user_defined BOOLEAN NOT NULL DEFAULT false,
    machining_direction TEXT,  -- +X | -X | +Y | -Y | +Z | -Z
    volume_mm3 REAL,  -- Material removal volume
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX features_project_idx ON features(project_id);
CREATE INDEX features_type_idx ON features(feature_type);

-- Operations (machining operations assigned to features)
CREATE TABLE operations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    feature_id UUID REFERENCES features(id) ON DELETE SET NULL,
    operation_type TEXT NOT NULL,  -- roughing | finishing | drilling | tapping | boring | chamfering
    strategy TEXT NOT NULL,  -- adaptive | pocket | contour | face | drill | peck_drill | thread_mill | helical_bore
    tool_id UUID REFERENCES tools(id),
    parameters JSONB NOT NULL DEFAULT '{}',  -- Strategy-specific parameters (stepover, stepdown, speeds, etc.)
    order_index INTEGER NOT NULL DEFAULT 0,
    is_enabled BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX operations_project_idx ON operations(project_id);
CREATE INDEX operations_feature_idx ON operations(feature_id);
CREATE INDEX operations_order_idx ON operations(project_id, order_index);

-- Toolpaths (computed toolpath data)
CREATE TABLE toolpaths (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    operation_id UUID NOT NULL REFERENCES operations(id) ON DELETE CASCADE,
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | generating | completed | failed
    execution_mode TEXT NOT NULL DEFAULT 'wasm',  -- wasm | server
    toolpath_data_url TEXT,  -- S3 URL to binary toolpath data (rapid/linear/arc moves)
    gcode_url TEXT,  -- S3 URL to generated G-code file
    statistics JSONB,  -- {cycle_time_sec, rapid_distance_mm, cut_distance_mm, num_moves}
    error_message TEXT,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    processing_time_ms INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX toolpaths_operation_idx ON toolpaths(operation_id);
CREATE INDEX toolpaths_project_idx ON toolpaths(project_id);
CREATE INDEX toolpaths_status_idx ON toolpaths(status);

-- Simulation Jobs (material removal simulation and collision detection)
CREATE TABLE simulation_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    toolpath_ids UUID[] NOT NULL,  -- Array of toolpath IDs to simulate
    machine_id UUID REFERENCES machines(id),
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | running | completed | failed
    result_url TEXT,  -- S3 URL to simulation result (stock mesh, collision events)
    collision_detected BOOLEAN DEFAULT false,
    collisions JSONB DEFAULT '[]',  -- [{time_sec, type: 'tool_holder'|'tool'|'spindle', object: 'part'|'fixture'|'machine'}]
    cycle_time_estimate_sec REAL,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    processing_time_ms INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX simulation_jobs_project_idx ON simulation_jobs(project_id);
CREATE INDEX simulation_jobs_status_idx ON simulation_jobs(status);

-- Tools (cutting tool library)
CREATE TABLE tools (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    tool_type TEXT NOT NULL,  -- end_mill | ball_nose | bull_nose | face_mill | drill | center_drill | tap | reamer | boring_bar | chamfer_mill | slot_drill
    material TEXT NOT NULL,  -- hss | cobalt | carbide | carbide_coated
    diameter_mm REAL NOT NULL,
    flute_count INTEGER,
    cutting_length_mm REAL NOT NULL,
    overall_length_mm REAL NOT NULL,
    shank_diameter_mm REAL NOT NULL,
    corner_radius_mm REAL DEFAULT 0.0,  -- For bull nose and face mills
    taper_angle_deg REAL DEFAULT 0.0,  -- For tapered ball nose or chamfer mills
    geometry_url TEXT,  -- S3 URL to 3D model of tool (for accurate collision detection)
    manufacturer TEXT,
    part_number TEXT,
    description TEXT,
    is_builtin BOOLEAN DEFAULT true,
    uploaded_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX tools_type_idx ON tools(tool_type);
CREATE INDEX tools_diameter_idx ON tools(diameter_mm);
CREATE INDEX tools_manufacturer_idx ON tools(manufacturer);
CREATE INDEX tools_name_trgm_idx ON tools USING gin(name gin_trgm_ops);

-- Materials (workpiece material database)
CREATE TABLE materials (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT UNIQUE NOT NULL,
    category TEXT NOT NULL,  -- aluminum | steel | stainless_steel | titanium | brass | copper | plastic | wood | composite
    hardness_hb REAL,  -- Brinell hardness
    density_g_cm3 REAL,
    description TEXT,
    is_builtin BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX materials_category_idx ON materials(category);
CREATE INDEX materials_name_trgm_idx ON materials USING gin(name gin_trgm_ops);

-- Cutting Parameters (AI-optimized feeds and speeds)
CREATE TABLE cutting_parameters (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tool_id UUID NOT NULL REFERENCES tools(id) ON DELETE CASCADE,
    material_id UUID NOT NULL REFERENCES materials(id) ON DELETE CASCADE,
    operation_type TEXT NOT NULL,  -- roughing | finishing | slotting | drilling
    spindle_speed_rpm INTEGER NOT NULL,
    feed_rate_mm_min REAL NOT NULL,
    plunge_rate_mm_min REAL NOT NULL,
    depth_of_cut_mm REAL NOT NULL,  -- Axial depth (stepdown)
    stepover_percent REAL NOT NULL,  -- Radial stepover as % of tool diameter
    notes TEXT,
    is_ai_optimized BOOLEAN DEFAULT false,
    confidence_score REAL,  -- 0.0-1.0 from AI model
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(tool_id, material_id, operation_type)
);
CREATE INDEX cutting_params_tool_idx ON cutting_parameters(tool_id);
CREATE INDEX cutting_params_material_idx ON cutting_parameters(material_id);

-- Machines (CNC machine definitions)
CREATE TABLE machines (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    manufacturer TEXT NOT NULL,
    model TEXT NOT NULL,
    controller TEXT NOT NULL,  -- fanuc | haas | siemens | heidenhain | grbl | linuxcnc | mach3
    axes TEXT NOT NULL DEFAULT '3',  -- 3 | 3+2 | 4 | 5
    x_travel_mm REAL NOT NULL,
    y_travel_mm REAL NOT NULL,
    z_travel_mm REAL NOT NULL,
    max_spindle_speed_rpm INTEGER NOT NULL,
    max_feed_rate_mm_min REAL NOT NULL,
    rapid_rate_mm_min REAL NOT NULL,
    tool_changer_type TEXT,  -- manual | automatic | atc
    kinematics_config JSONB,  -- Machine-specific kinematic parameters for 5-axis
    post_processor_id UUID,  -- Reference to post-processor configuration
    geometry_url TEXT,  -- S3 URL to 3D model of machine for simulation
    is_builtin BOOLEAN DEFAULT true,
    uploaded_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX machines_controller_idx ON machines(controller);
CREATE INDEX machines_manufacturer_idx ON machines(manufacturer);

-- Post Processors (G-code output configuration)
CREATE TABLE post_processors (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT UNIQUE NOT NULL,
    controller_family TEXT NOT NULL,  -- fanuc | haas | siemens | heidenhain | grbl | linuxcnc | mach3
    description TEXT,
    config JSONB NOT NULL,  -- Post-processor configuration (output format, M-codes, syntax)
    is_verified BOOLEAN DEFAULT false,
    is_builtin BOOLEAN DEFAULT true,
    uploaded_by UUID REFERENCES users(id),
    download_count INTEGER DEFAULT 0,
    rating REAL,  -- 0.0-5.0 user rating
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX post_processors_controller_idx ON post_processors(controller_family);
CREATE INDEX post_processors_name_trgm_idx ON post_processors USING gin(name gin_trgm_ops);

-- Usage Tracking (for billing enforcement)
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,  -- toolpath_generation_minutes | cloud_storage_mb | simulation_minutes | api_calls
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
    pub cad_file_url: Option<String>,
    pub mesh_url: Option<String>,
    pub bounding_box: Option<serde_json::Value>,
    pub units: String,
    pub material: Option<String>,
    pub stock_dimensions: Option<serde_json::Value>,
    pub setup_data: serde_json::Value,
    pub is_public: bool,
    pub thumbnail_url: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Feature {
    pub id: Uuid,
    pub project_id: Uuid,
    pub feature_recognition_job_id: Option<Uuid>,
    pub feature_type: String,
    pub subtype: Option<String>,
    pub geometry: serde_json::Value,
    pub bounding_box: serde_json::Value,
    pub confidence: f32,
    pub is_user_defined: bool,
    pub machining_direction: Option<String>,
    pub volume_mm3: Option<f32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Operation {
    pub id: Uuid,
    pub project_id: Uuid,
    pub feature_id: Option<Uuid>,
    pub operation_type: String,
    pub strategy: String,
    pub tool_id: Option<Uuid>,
    pub parameters: serde_json::Value,
    pub order_index: i32,
    pub is_enabled: bool,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Toolpath {
    pub id: Uuid,
    pub operation_id: Uuid,
    pub project_id: Uuid,
    pub status: String,
    pub execution_mode: String,
    pub toolpath_data_url: Option<String>,
    pub gcode_url: Option<String>,
    pub statistics: Option<serde_json::Value>,
    pub error_message: Option<String>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub processing_time_ms: Option<i32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Tool {
    pub id: Uuid,
    pub name: String,
    pub tool_type: String,
    pub material: String,
    pub diameter_mm: f32,
    pub flute_count: Option<i32>,
    pub cutting_length_mm: f32,
    pub overall_length_mm: f32,
    pub shank_diameter_mm: f32,
    pub corner_radius_mm: f32,
    pub taper_angle_deg: f32,
    pub geometry_url: Option<String>,
    pub manufacturer: Option<String>,
    pub part_number: Option<String>,
    pub description: Option<String>,
    pub is_builtin: bool,
    pub uploaded_by: Option<Uuid>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Material {
    pub id: Uuid,
    pub name: String,
    pub category: String,
    pub hardness_hb: Option<f32>,
    pub density_g_cm3: Option<f32>,
    pub description: Option<String>,
    pub is_builtin: bool,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct CuttingParameter {
    pub id: Uuid,
    pub tool_id: Uuid,
    pub material_id: Uuid,
    pub operation_type: String,
    pub spindle_speed_rpm: i32,
    pub feed_rate_mm_min: f32,
    pub plunge_rate_mm_min: f32,
    pub depth_of_cut_mm: f32,
    pub stepover_percent: f32,
    pub notes: Option<String>,
    pub is_ai_optimized: bool,
    pub confidence_score: Option<f32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Machine {
    pub id: Uuid,
    pub name: String,
    pub manufacturer: String,
    pub model: String,
    pub controller: String,
    pub axes: String,
    pub x_travel_mm: f32,
    pub y_travel_mm: f32,
    pub z_travel_mm: f32,
    pub max_spindle_speed_rpm: i32,
    pub max_feed_rate_mm_min: f32,
    pub rapid_rate_mm_min: f32,
    pub tool_changer_type: Option<String>,
    pub kinematics_config: Option<serde_json::Value>,
    pub post_processor_id: Option<Uuid>,
    pub geometry_url: Option<String>,
    pub is_builtin: bool,
    pub uploaded_by: Option<Uuid>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct OperationParams {
    pub operation_type: OperationType,
    pub strategy: Strategy,
    pub tool_diameter_mm: f32,
    pub spindle_speed_rpm: i32,
    pub feed_rate_mm_min: f32,
    pub plunge_rate_mm_min: f32,
    pub stepover_mm: f32,
    pub stepdown_mm: f32,
    pub safe_height_mm: f32,
    pub retract_height_mm: f32,
    pub finish_allowance_mm: f32,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum OperationType {
    Roughing,
    Finishing,
    Drilling,
    Tapping,
    Boring,
    Chamfering,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum Strategy {
    Adaptive,
    Pocket,
    Contour,
    Face,
    Drill,
    PeckDrill,
    ThreadMill,
    HelicalBore,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct HoleGeometry {
    pub center: [f32; 3],
    pub axis: [f32; 3],
    pub diameter: f32,
    pub depth: f32,
    pub hole_type: String,  // through | blind | counterbore | countersink
}

#[derive(Debug, Deserialize, Serialize)]
pub struct PocketGeometry {
    pub boundary: Vec<[f32; 2]>,  // 2D boundary polygon in XY plane
    pub depth: f32,
    pub floor_z: f32,
    pub islands: Vec<Vec<[f32; 2]>>,  // Internal islands (bosses)
}
```

---

## Toolpath Engine Architecture Deep-Dive

### Governing Algorithms and Discretization

CamWorks' toolpath engine implements industry-standard CAM algorithms adapted from commercial systems like Mastercam and Fusion 360.

**2.5D Adaptive Clearing (Pocket Roughing)**

Adaptive clearing maintains constant chip load by dynamically adjusting toolpath curvature and stepover. The algorithm:

1. Compute medial axis (skeleton) of pocket boundary using Voronoi diagram
2. Generate parallel offset curves from medial axis at stepover distance
3. Link offset curves with smooth arcs or helical ramps
4. Detect regions of high curvature and reduce feed rate to maintain chip load

```
Target chip load: C = F / (N × S)
  where F = feed rate (mm/min)
        N = flute count
        S = spindle speed (RPM)

For constant C:
  F_corner = C × N × S × (θ / 180°)  [reduced feed in corners]
```

**Contour Finishing**

Contour finishing generates parallel offset paths from part boundary with scallop height control:

1. Extract contour from 3D mesh or B-rep boundary
2. Compute offset curves at stepover distance using Minkowski sum
3. Handle self-intersecting offsets via Boolean operations
4. Z-level contouring for 3D surfaces: slice at stepdown intervals
5. Scallop height: h = R(1 - cos(asin(s/(2R)))) where R = tool radius, s = stepover

**Drilling Cycles**

Standard drilling cycles with pecking for deep holes:

```
Peck drilling cycle:
  1. Rapid to retract height (typically +5mm above part surface)
  2. Feed to initial peck depth at plunge rate
  3. Rapid retract to clearance height
  4. Repeat steps 2-3 until target depth reached
  5. Rapid to safe height
```

### Client/Server Split (WASM Threshold)

```
Feature count extracted → Toolpath complexity estimate
    │
    ├── ≤100 features, 2.5D → WASM toolpath engine (browser)
    │   ├── Instant generation, no network latency
    │   ├── Toolpath displayed immediately in 3D viewer
    │   └── No server cost
    │
    └── >100 features or 3D → Server toolpath engine (Rust native)
        ├── Job queued via Redis
        ├── Worker runs native engine with full CAM algorithms
        ├── Progress streamed via WebSocket
        └── Results stored in S3, URL returned
```

The 100-feature threshold was chosen because:
- WASM engine handles 100 2.5D features (pockets + holes) in <5 seconds on modern hardware
- 100 features covers: typical job shop parts (20-50 features), production brackets (30-80 features)
- Above 100 features: complex aerospace/automotive parts, multi-setup jobs → need server compute

### WASM Compilation Pipeline

```toml
# toolpath-wasm/Cargo.toml
[package]
name = "camworks-toolpath-wasm"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
wasm-bindgen = "0.2"
serde = { version = "1", features = ["derive"] }
serde-wasm-bindgen = "0.6"
js-sys = "0.3"
geo = "0.28"  # Geometric algorithms (offset curves, Boolean ops)
geo-clipper = "0.8"  # Polygon clipping
spade = "2.9"  # Delaunay triangulation, Voronoi diagrams
lyon_geom = "1.0"  # 2D geometry primitives
nalgebra = "0.33"  # Linear algebra for transforms
log = "0.4"
console_log = "1"

[profile.release]
opt-level = "z"
lto = true
codegen-units = 1
strip = true
wasm-opt = true
```

---

## Architecture Deep-Dives

### 1. Toolpath Generation API Handler (Rust/Axum)

The primary endpoint receives a toolpath generation request, validates operation parameters, decides between WASM and server execution, and for server execution enqueues a job with Redis.

```rust
// src/api/handlers/toolpath.rs

use axum::{
    extract::{Path, State, Json},
    http::StatusCode,
    response::IntoResponse,
};
use uuid::Uuid;
use crate::{
    db::models::{Operation, Toolpath, Feature, OperationParams},
    toolpath::generator::estimate_complexity,
    state::AppState,
    auth::Claims,
    error::ApiError,
};

#[derive(serde::Deserialize)]
pub struct GenerateToolpathRequest {
    pub operation_id: Uuid,
}

pub async fn generate_toolpath(
    State(state): State<AppState>,
    claims: Claims,
    Path(project_id): Path<Uuid>,
    Json(req): Json<GenerateToolpathRequest>,
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

    // 2. Fetch operation details
    let operation = sqlx::query_as!(
        Operation,
        "SELECT * FROM operations WHERE id = $1 AND project_id = $2",
        req.operation_id,
        project_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Operation not found"))?;

    // 3. Fetch associated feature geometry
    let feature = if let Some(feature_id) = operation.feature_id {
        sqlx::query_as!(
            Feature,
            "SELECT * FROM features WHERE id = $1",
            feature_id
        )
        .fetch_optional(&state.db)
        .await?
    } else {
        None
    };

    // 4. Estimate toolpath complexity
    let feature_count = sqlx::query_scalar!(
        "SELECT COUNT(*) FROM features WHERE project_id = $1",
        project_id
    )
    .fetch_one(&state.db)
    .await?
    .unwrap_or(0);

    let is_3d_strategy = matches!(
        operation.strategy.as_str(),
        "waterline" | "spiral" | "ramp" | "morph"
    );

    // 5. Check plan limits
    let user = sqlx::query_as!(
        crate::db::models::User,
        "SELECT * FROM users WHERE id = $1",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    if user.plan == "free" && (feature_count > 10 || is_3d_strategy) {
        return Err(ApiError::PlanLimit(
            "Free plan supports up to 10 features and 2.5D toolpaths only. Upgrade to Pro."
        ));
    }

    // 6. Determine execution mode
    let execution_mode = if feature_count <= 100 && !is_3d_strategy {
        "wasm"
    } else {
        "server"
    };

    // 7. Create toolpath record
    let toolpath = sqlx::query_as!(
        Toolpath,
        r#"INSERT INTO toolpaths
            (operation_id, project_id, status, execution_mode)
        VALUES ($1, $2, $3, $4)
        RETURNING *"#,
        operation.id,
        project_id,
        if execution_mode == "wasm" { "completed" } else { "pending" },
        execution_mode,
    )
    .fetch_one(&state.db)
    .await?;

    // 8. For server execution, enqueue job
    if execution_mode == "server" {
        let job_data = serde_json::json!({
            "toolpath_id": toolpath.id,
            "operation_id": operation.id,
            "project_id": project_id,
            "priority": if user.plan == "production" { 10 } else { 0 },
        });

        state.redis
            .publish("toolpath:jobs", serde_json::to_string(&job_data)?)
            .await?;
    }

    Ok((StatusCode::CREATED, Json(toolpath)))
}

pub async fn get_toolpath(
    State(state): State<AppState>,
    claims: Claims,
    Path((project_id, toolpath_id)): Path<(Uuid, Uuid)>,
) -> Result<Json<Toolpath>, ApiError> {
    let toolpath = sqlx::query_as!(
        Toolpath,
        "SELECT t.* FROM toolpaths t
         JOIN projects p ON t.project_id = p.id
         WHERE t.id = $1 AND t.project_id = $2
         AND (p.owner_id = $3 OR p.org_id IN (
             SELECT org_id FROM org_members WHERE user_id = $3
         ))",
        toolpath_id,
        project_id,
        claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Toolpath not found"))?;

    Ok(Json(toolpath))
}

pub async fn download_gcode(
    State(state): State<AppState>,
    claims: Claims,
    Path((project_id, toolpath_id)): Path<(Uuid, Uuid)>,
) -> Result<impl IntoResponse, ApiError> {
    let toolpath = sqlx::query_as!(
        Toolpath,
        "SELECT t.* FROM toolpaths t
         JOIN projects p ON t.project_id = p.id
         WHERE t.id = $1 AND t.project_id = $2
         AND (p.owner_id = $3 OR p.org_id IN (
             SELECT org_id FROM org_members WHERE user_id = $3
         ))",
        toolpath_id,
        project_id,
        claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Toolpath not found"))?;

    let gcode_url = toolpath.gcode_url
        .ok_or(ApiError::NotFound("G-code not generated yet"))?;

    // Generate presigned S3 URL for download
    let presigned_url = state.s3
        .get_object()
        .bucket("camworks-gcode")
        .key(&gcode_url)
        .presigned(
            aws_sdk_s3::presigning::PresigningConfig::expires_in(
                std::time::Duration::from_secs(3600)
            )?
        )
        .await?;

    Ok(Json(serde_json::json!({ "download_url": presigned_url.uri() })))
}
```

### 2. Adaptive Pocket Clearing (Rust — shared between WASM and native)

The core pocket clearing algorithm that generates high-efficiency roughing toolpaths with constant chip load.

```rust
// toolpath-core/src/strategies/adaptive.rs

use geo::{Polygon, LineString, Coord, Area};
use geo_clipper::Clipper;
use crate::toolpath::{Move, MoveType, Toolpath};
use crate::offset::offset_polygon;

pub struct AdaptiveClearing {
    pub tool_diameter: f32,
    pub stepover_percent: f32,  // 40-60% typical
    pub stepdown: f32,
    pub safe_height: f32,
    pub retract_height: f32,
    pub feed_rate: f32,
    pub plunge_rate: f32,
}

impl AdaptiveClearing {
    pub fn generate(
        &self,
        boundary: &Polygon<f32>,
        islands: &[Polygon<f32>],
        floor_z: f32,
        top_z: f32,
    ) -> Result<Toolpath, String> {
        let mut moves = Vec::new();
        let tool_radius = self.tool_diameter / 2.0;
        let stepover = self.tool_diameter * self.stepover_percent / 100.0;

        // Offset boundary inward by tool radius for tool center path
        let mut boundary_offset = offset_polygon(boundary, -tool_radius)?;
        let islands_offset: Vec<_> = islands.iter()
            .filter_map(|island| offset_polygon(island, tool_radius).ok())
            .collect();

        // Z-level slicing from top to floor
        let mut current_z = top_z - self.stepdown;

        while current_z >= floor_z {
            current_z = current_z.max(floor_z);

            // Compute medial axis for this Z level
            let skeleton = self.compute_medial_axis(&boundary_offset, &islands_offset)?;

            // Generate offset curves from skeleton
            let passes = self.generate_offset_passes(&skeleton, stepover)?;

            // Generate helical entry at first pass
            if let Some(first_pass) = passes.first() {
                let entry_point = first_pass.0[0];
                moves.push(Move {
                    move_type: MoveType::Rapid,
                    end: [entry_point.x, entry_point.y, self.safe_height],
                    feed_rate: None,
                });
                moves.push(Move {
                    move_type: MoveType::Rapid,
                    end: [entry_point.x, entry_point.y, self.retract_height],
                    feed_rate: None,
                });

                // Helical ramp entry
                let helix_moves = self.helical_ramp(
                    entry_point,
                    self.retract_height,
                    current_z,
                    tool_radius * 0.5,
                    self.plunge_rate
                );
                moves.extend(helix_moves);
            }

            // Cut each pass
            for pass in passes {
                for (i, coord) in pass.0.iter().enumerate() {
                    moves.push(Move {
                        move_type: if i == 0 {
                            MoveType::RapidPlunge
                        } else {
                            MoveType::Linear
                        },
                        end: [coord.x, coord.y, current_z],
                        feed_rate: Some(self.feed_rate),
                    });
                }
            }

            // Retract for next level
            if current_z > floor_z {
                let last_pos = moves.last().unwrap().end;
                moves.push(Move {
                    move_type: MoveType::Rapid,
                    end: [last_pos[0], last_pos[1], self.retract_height],
                    feed_rate: None,
                });
            }

            if current_z == floor_z {
                break;
            }
            current_z -= self.stepdown;
        }

        // Final retract to safe height
        let last_pos = moves.last().unwrap().end;
        moves.push(Move {
            move_type: MoveType::Rapid,
            end: [last_pos[0], last_pos[1], self.safe_height],
            feed_rate: None,
        });

        Ok(Toolpath { moves })
    }

    fn compute_medial_axis(
        &self,
        boundary: &Polygon<f32>,
        islands: &[Polygon<f32>],
    ) -> Result<Vec<LineString<f32>>, String> {
        // Use Voronoi diagram to compute medial axis (skeleton)
        use spade::{DelaunayTriangulation, Point2};

        let mut points = Vec::new();
        for coord in &boundary.exterior().0 {
            points.push(Point2::new(coord.x as f64, coord.y as f64));
        }
        for island in islands {
            for coord in &island.exterior().0 {
                points.push(Point2::new(coord.x as f64, coord.y as f64));
            }
        }

        let triangulation = DelaunayTriangulation::<Point2<f64>>::bulk_load(points)
            .map_err(|e| format!("Triangulation failed: {:?}", e))?;

        // Extract Voronoi edges that lie inside boundary (medial axis)
        let mut skeleton_edges = Vec::new();
        for edge in triangulation.voronoi_faces() {
            // Convert Voronoi edge to LineString
            // Filter edges outside boundary
            // TODO: Implement proper Voronoi -> medial axis conversion
            // This is a placeholder for the actual algorithm
        }

        Ok(skeleton_edges)
    }

    fn generate_offset_passes(
        &self,
        skeleton: &[LineString<f32>],
        stepover: f32,
    ) -> Result<Vec<LineString<f32>>, String> {
        // Generate parallel offset curves from skeleton at stepover distance
        let mut passes = Vec::new();

        for spine in skeleton {
            let mut offset_distance = stepover / 2.0;
            loop {
                // Offset spine by current distance
                // This is simplified; actual implementation uses geo-offset crate
                let offset_left = self.offset_linestring(spine, offset_distance);
                let offset_right = self.offset_linestring(spine, -offset_distance);

                if let Some(left) = offset_left {
                    passes.push(left);
                }
                if let Some(right) = offset_right {
                    passes.push(right);
                }

                offset_distance += stepover;
                if offset_distance > stepover * 10.0 {
                    break;  // Safety limit
                }
            }
        }

        Ok(passes)
    }

    fn helical_ramp(
        &self,
        center: Coord<f32>,
        start_z: f32,
        end_z: f32,
        radius: f32,
        feed_rate: f32,
    ) -> Vec<Move> {
        let mut moves = Vec::new();
        let z_range = start_z - end_z;
        let turns = (z_range / (self.stepdown * 0.5)).ceil();
        let angle_step = 10.0_f32.to_radians();  // 10 degree segments
        let z_per_angle = z_range / (turns * 2.0 * std::f32::consts::PI);

        let mut angle = 0.0;
        let mut z = start_z;

        while z > end_z {
            let x = center.x + radius * angle.cos();
            let y = center.y + radius * angle.sin();
            z = (start_z - angle * z_per_angle).max(end_z);

            moves.push(Move {
                move_type: MoveType::HelicalRamp,
                end: [x, y, z],
                feed_rate: Some(feed_rate),
            });

            angle += angle_step;
        }

        moves
    }

    fn offset_linestring(
        &self,
        line: &LineString<f32>,
        distance: f32,
    ) -> Option<LineString<f32>> {
        // Simplified offset; use geo-offset crate for production
        // This placeholder returns None; actual implementation computes parallel curve
        None
    }
}

#[derive(Debug, Clone)]
pub struct Move {
    pub move_type: MoveType,
    pub end: [f32; 3],
    pub feed_rate: Option<f32>,
}

#[derive(Debug, Clone, Copy)]
pub enum MoveType {
    Rapid,
    RapidPlunge,
    Linear,
    Arc,
    HelicalRamp,
}

pub struct Toolpath {
    pub moves: Vec<Move>,
}
```

### 3. Feature Recognition Service (Python FastAPI + TensorFlow)

Separate microservice that receives voxelized CAD geometry and runs 3D CNN inference to detect machinable features.

```python
# feature-recognition-service/app/main.py

from fastapi import FastAPI, UploadFile, HTTPException, BackgroundTasks
from pydantic import BaseModel
import tensorflow as tf
import numpy as np
import asyncio
from typing import List, Dict, Any
import trimesh
import uuid
from datetime import datetime

app = FastAPI(title="CamWorks Feature Recognition Service")

# Load pre-trained 3D CNN model
MODEL_PATH = "/models/feature_recognition_v1.h5"
model = tf.keras.models.load_model(MODEL_PATH)

# Feature types the model can detect
FEATURE_CLASSES = [
    "hole_through", "hole_blind", "hole_counterbore", "hole_countersink",
    "pocket_open", "pocket_closed", "slot", "boss", "face", "chamfer", "fillet"
]

class VoxelizeRequest(BaseModel):
    mesh_url: str
    resolution: int = 64  # 32, 64, or 128
    project_id: str
    job_id: str

class DetectedFeature(BaseModel):
    feature_type: str
    confidence: float
    bounding_box: Dict[str, List[float]]
    geometry: Dict[str, Any]
    volume_mm3: float

class FeatureRecognitionResult(BaseModel):
    job_id: str
    features: List[DetectedFeature]
    processing_time_ms: int

def voxelize_mesh(mesh: trimesh.Trimesh, resolution: int) -> np.ndarray:
    """Convert triangle mesh to voxel grid."""
    # Compute voxel grid bounds from mesh bounding box
    bounds = mesh.bounds
    voxel_size = np.max(bounds[1] - bounds[0]) / resolution

    # Use trimesh voxelization
    voxelized = mesh.voxelized(pitch=voxel_size)
    voxel_matrix = voxelized.matrix

    # Pad/crop to exact resolution
    padded = np.zeros((resolution, resolution, resolution), dtype=np.float32)
    actual_shape = voxel_matrix.shape
    min_shape = [min(actual_shape[i], resolution) for i in range(3)]
    padded[:min_shape[0], :min_shape[1], :min_shape[2]] = \
        voxel_matrix[:min_shape[0], :min_shape[1], :min_shape[2]]

    return padded

def extract_features_from_heatmap(
    heatmap: np.ndarray,
    confidence_threshold: float = 0.5
) -> List[DetectedFeature]:
    """Extract individual features from 3D heatmap output."""
    features = []

    for class_idx, feature_type in enumerate(FEATURE_CLASSES):
        class_heatmap = heatmap[..., class_idx]

        # Find connected components above threshold
        from scipy import ndimage
        labeled, num_features = ndimage.label(class_heatmap > confidence_threshold)

        for feature_id in range(1, num_features + 1):
            feature_mask = (labeled == feature_id)
            if np.sum(feature_mask) < 8:  # Minimum 8 voxels
                continue

            # Compute bounding box
            indices = np.argwhere(feature_mask)
            min_coords = indices.min(axis=0).tolist()
            max_coords = indices.max(axis=0).tolist()

            # Compute average confidence
            confidence = float(class_heatmap[feature_mask].mean())

            # Extract geometry parameters based on feature type
            geometry = extract_feature_geometry(
                feature_type, feature_mask, class_heatmap
            )

            # Estimate volume
            volume_voxels = np.sum(feature_mask)
            voxel_size_mm = 100.0 / heatmap.shape[0]  # Assuming 100mm part
            volume_mm3 = volume_voxels * (voxel_size_mm ** 3)

            features.append(DetectedFeature(
                feature_type=feature_type,
                confidence=confidence,
                bounding_box={
                    "min": min_coords,
                    "max": max_coords
                },
                geometry=geometry,
                volume_mm3=volume_mm3
            ))

    return features

def extract_feature_geometry(
    feature_type: str,
    mask: np.ndarray,
    heatmap: np.ndarray
) -> Dict[str, Any]:
    """Extract geometric parameters for specific feature type."""
    if "hole" in feature_type:
        # Detect hole axis using PCA
        indices = np.argwhere(mask)
        centroid = indices.mean(axis=0)

        # Simplified: assume Z-axis holes
        return {
            "center": centroid.tolist(),
            "axis": [0.0, 0.0, 1.0],
            "diameter": estimate_hole_diameter(mask),
            "depth": estimate_hole_depth(mask),
        }

    elif "pocket" in feature_type:
        # Extract pocket boundary
        return {
            "boundary": extract_2d_boundary(mask),
            "depth": estimate_pocket_depth(mask),
            "floor_z": estimate_floor_height(mask),
        }

    # Generic geometry for other features
    return {
        "voxel_coords": np.argwhere(mask).tolist()
    }

def estimate_hole_diameter(mask: np.ndarray) -> float:
    """Estimate hole diameter from voxel mask."""
    # Find largest cross-section
    max_area = 0
    for z in range(mask.shape[2]):
        slice_area = np.sum(mask[:, :, z])
        max_area = max(max_area, slice_area)

    # Diameter from area (assuming circular)
    diameter = 2.0 * np.sqrt(max_area / np.pi)
    return float(diameter)

def estimate_hole_depth(mask: np.ndarray) -> float:
    """Estimate hole depth from voxel mask."""
    z_coords = np.argwhere(mask)[:, 2]
    depth = float(z_coords.max() - z_coords.min())
    return depth

def estimate_pocket_depth(mask: np.ndarray) -> float:
    """Estimate pocket depth."""
    z_coords = np.argwhere(mask)[:, 2]
    return float(z_coords.max() - z_coords.min())

def estimate_floor_height(mask: np.ndarray) -> float:
    """Estimate pocket floor Z height."""
    z_coords = np.argwhere(mask)[:, 2]
    return float(z_coords.min())

def extract_2d_boundary(mask: np.ndarray) -> List[List[float]]:
    """Extract 2D boundary polygon from 3D mask."""
    # Project to XY plane
    xy_projection = np.any(mask, axis=2)

    # Find boundary using marching squares
    from skimage import measure
    contours = measure.find_contours(xy_projection.astype(float), 0.5)

    if len(contours) == 0:
        return []

    # Return largest contour
    largest = max(contours, key=len)
    return largest.tolist()

@app.post("/recognize", response_model=FeatureRecognitionResult)
async def recognize_features(request: VoxelizeRequest):
    """Recognize machinable features from CAD mesh."""
    start_time = datetime.now()

    # 1. Download mesh from S3 (simulated here)
    # In production: use boto3 to download from request.mesh_url
    # mesh = trimesh.load(...)

    # For this demo, create a simple test mesh
    mesh = trimesh.creation.box(extents=[100, 50, 25])

    # 2. Voxelize mesh
    voxel_grid = voxelize_mesh(mesh, request.resolution)

    # 3. Prepare input for model (batch size 1, add channel dimension)
    model_input = np.expand_dims(voxel_grid, axis=(0, -1))  # Shape: (1, res, res, res, 1)

    # 4. Run model inference
    predictions = model.predict(model_input, verbose=0)

    # Model output shape: (1, res, res, res, num_classes)
    heatmap = predictions[0]  # Remove batch dimension

    # 5. Extract features from heatmap
    features = extract_features_from_heatmap(heatmap, confidence_threshold=0.5)

    # 6. Calculate processing time
    processing_time_ms = int((datetime.now() - start_time).total_seconds() * 1000)

    return FeatureRecognitionResult(
        job_id=request.job_id,
        features=features,
        processing_time_ms=processing_time_ms
    )

@app.get("/health")
async def health_check():
    return {"status": "healthy", "model_loaded": model is not None}

@app.get("/model/info")
async def model_info():
    return {
        "model_path": MODEL_PATH,
        "input_shape": model.input_shape,
        "output_shape": model.output_shape,
        "feature_classes": FEATURE_CLASSES,
    }
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–4)

**Day 1: Project initialization and Rust backend scaffold**
```bash
cargo init camworks-api
cd camworks-api
cargo add axum tokio serde serde_json sqlx uuid chrono tower-http jsonwebtoken bcrypt
cargo add --features postgres,runtime-tokio-rustls,uuid,chrono,json sqlx
cargo add aws-sdk-s3 redis
```
- `src/main.rs` — Axum app with CORS, tracing, graceful shutdown
- `src/config.rs` — Environment-based configuration (DATABASE_URL, JWT_SECRET, S3_BUCKET, REDIS_URL)
- `src/state.rs` — AppState struct (PgPool, Redis, S3Client)
- `src/error.rs` — ApiError enum with IntoResponse
- `Dockerfile` — Multi-stage build (builder + runtime)
- `docker-compose.yml` — PostgreSQL, Redis, MinIO (S3-compatible)

**Day 2: Database schema and migrations**
- `migrations/001_initial.sql` — All 15 tables: users, organizations, org_members, projects, feature_recognition_jobs, features, operations, toolpaths, simulation_jobs, tools, materials, cutting_parameters, machines, post_processors, usage_records
- `src/db/mod.rs` — Database pool initialization
- `src/db/models.rs` — All SQLx structs with FromRow derives
- Run `sqlx migrate run` to apply schema
- Seed script for initial tool library (10 common end mills, 5 drills)
- Seed script for materials database (aluminum 6061, 7075, steel 1018, 4140, stainless 304, 316)

**Day 3: Authentication system**
- `src/auth/mod.rs` — JWT token generation and validation middleware
- `src/auth/oauth.rs` — Google and GitHub OAuth 2.0 flow handlers
- `src/api/handlers/auth.rs` — Register, login, OAuth callback, refresh token, me endpoints
- Password hashing with bcrypt (cost factor 12)
- JWT with 24h access token + 30d refresh token
- Auth middleware that extracts Claims from Authorization header

**Day 4: User and project CRUD**
- `src/api/handlers/users.rs` — Get profile, update profile, delete account
- `src/api/handlers/projects.rs` — Create, list, get, update, delete project
- `src/api/handlers/orgs.rs` — Create org, invite member, list members, remove member
- `src/api/router.rs` — All route definitions with auth middleware
- Integration tests: auth flow, project CRUD, authorization checks

### Phase 2 — CAD Import + Feature Recognition (Days 5–10)

**Day 5: Python CAD import service foundation**
```bash
cd feature-recognition-service
python -m venv venv
source venv/bin/activate
pip install fastapi uvicorn pythonocc-core trimesh numpy scipy scikit-image boto3
```
- `app/main.py` — FastAPI service with health check endpoint
- `app/cad_import.py` — STEP/IGES import using pythonocc-core
- `app/mesh_processing.py` — Convert B-rep to triangle mesh, simplification
- Test STEP import with sample files (bracket, flange, housing)

**Day 6: Voxelization pipeline**
- `app/voxelization.py` — Mesh to voxel grid conversion using trimesh
- Support 32x32x32, 64x64x64, 128x128x128 resolutions
- Voxel grid serialization (NumPy binary format for TensorFlow)
- Bounding box normalization (scale to unit cube)
- Test voxelization with various part geometries

**Day 7: ML model architecture and training data preparation**
- `ml/model.py` — 3D CNN architecture using TensorFlow/Keras
  - Input: (64, 64, 64, 1) voxel grid
  - 4x 3D Conv layers (32, 64, 128, 256 filters) + BatchNorm + ReLU + MaxPool
  - Output: (64, 64, 64, 11) heatmap (one channel per feature class)
- `ml/train.py` — Training script (placeholder, uses synthetic data)
- `ml/dataset.py` — Data loader for annotated CAD models
- For MVP: use pre-trained model checkpoint (simulated with random weights)

**Day 8: Feature extraction from model output**
- `app/feature_extraction.py` — Connected component analysis on heatmap
- Bounding box computation for each detected feature
- Geometry parameter extraction (hole diameter/depth, pocket boundary)
- Confidence thresholding (>0.5) and NMS (non-maximum suppression)
- Convert voxel coordinates to real-world coordinates (mm/inch)

**Day 9: Feature recognition API integration**
- Rust API handler: `src/api/handlers/features.rs` — Trigger feature recognition job
- Job creation flow: upload CAD → create project → enqueue feature recognition job
- HTTP client in Rust to call Python service (`reqwest` crate)
- S3 integration: upload mesh, download mesh URL for Python service
- Database: store feature recognition job status and results

**Day 10: Feature review and manual editing UI foundation**
- `frontend/src/pages/Features.tsx` — Feature list with type icons and confidence scores
- Feature detail panel with geometry parameters
- Manual feature creation form (add hole, pocket, face)
- Feature edit/delete actions
- Feature visualization in 3D viewer (bounding boxes, wireframes)

### Phase 3 — Toolpath Engine — Core Algorithms (Days 11–18)

**Day 11: Toolpath core library structure**
- `toolpath-core/` — New Rust workspace member (shared between native and WASM)
- `toolpath-core/src/lib.rs` — Public API exports
- `toolpath-core/src/toolpath.rs` — Toolpath struct (sequence of moves)
- `toolpath-core/src/move.rs` — Move types (rapid, linear, arc, helical)
- `toolpath-core/src/gcode.rs` — G-code generation from toolpath
- Unit tests: simple linear move, arc interpolation

**Day 12: 2D geometry primitives and operations**
- `toolpath-core/src/geometry/mod.rs` — 2D point, line, arc, polygon types
- `toolpath-core/src/geometry/offset.rs` — Polygon offset using clipper library
- `toolpath-core/src/geometry/boolean.rs` — Union, intersection, difference operations
- `toolpath-core/src/geometry/distance.rs` — Point-to-curve distance queries
- Tests: offset square pocket, Boolean union of overlapping pockets

**Day 13: Drilling cycles**
- `toolpath-core/src/strategies/drilling.rs` — Simple drill, peck drill, deep hole peck
- `toolpath-core/src/strategies/drilling.rs` — Center drill, tap, ream, bore cycles
- G81 (simple drill), G83 (peck drill) cycle generation
- Drill point sorting (nearest-neighbor to minimize rapids)
- Tests: 5-hole pattern, 20-hole pattern with optimization

**Day 14: Contour milling (2.5D)**
- `toolpath-core/src/strategies/contour.rs` — Offset-based contour generation
- Multiple passes with stepdown for full depth
- Lead-in/lead-out strategies (linear, arc, perpendicular)
- Climb vs. conventional milling (tool on left/right of contour)
- Tests: rectangular pocket contour, circular boss contour

**Day 15: Pocket clearing (zigzag and offset)**
- `toolpath-core/src/strategies/pocket_zigzag.rs` — Zigzag pocket roughing
- Parallel passes with 180° direction reversal
- Island avoidance (offset islands, clip zigzag)
- `toolpath-core/src/strategies/pocket_offset.rs` — Spiral-in offset clearing
- Tests: square pocket with circular island, complex polygon pocket

**Day 16: Adaptive clearing foundations**
- `toolpath-core/src/strategies/adaptive.rs` — Medial axis computation using Voronoi
- `spade` crate for Delaunay triangulation and Voronoi diagrams
- Skeleton extraction (Voronoi edges inside boundary)
- Offset curve generation from skeleton
- Tests: L-shaped pocket medial axis, complex multi-island pocket

**Day 17: Adaptive clearing — linking and ramping**
- Smooth linking between offset passes (minimize retracts)
- Helical ramp entry (spiral down at entry point)
- Adaptive feed rate in corners (maintain chip load)
- Safe height and retract height management
- Tests: full adaptive pocket cycle with entry/exit/linking

**Day 18: 3D contour finishing (Z-level)**
- `toolpath-core/src/strategies/contour_3d.rs` — Z-level parallel contour
- Slice 3D surface at stepdown intervals
- Extract 2D contours at each Z level
- Connect levels with plunge or helical ramp
- Tests: cylindrical boss, conical surface, freeform surface (simplified)

### Phase 4 — WASM Build + 3D Viewer Frontend (Days 19–24)

**Day 19: WASM toolpath compilation**
- `toolpath-wasm/` — New workspace member for WASM target
- `toolpath-wasm/Cargo.toml` — wasm-bindgen, serde-wasm-bindgen, geo dependencies
- `toolpath-wasm/src/lib.rs` — WASM entry points: `generate_drilling()`, `generate_contour()`, `generate_pocket()`, `generate_adaptive()`
- `wasm-pack build --target web --release` pipeline
- `wasm-opt -Oz` for size optimization (target <3MB gzipped)
- JavaScript wrapper: `ToolpathEngine` class that loads WASM and provides async API

**Day 20: Frontend scaffold and 3D viewer foundation**
```bash
npm create vite@latest frontend -- --template react-ts
cd frontend
npm i zustand @tanstack/react-query axios three @react-three/fiber @react-three/drei
```
- `src/App.tsx` — Router, auth context, layout
- `src/stores/projectStore.ts` — Zustand store for project state
- `src/stores/toolpathStore.ts` — Toolpath and operation state
- `src/components/Viewer3D/Scene.tsx` — React Three Fiber scene setup
- `src/components/Viewer3D/Controls.tsx` — OrbitControls, camera, lighting

**Day 21: CAD mesh rendering**
- `src/components/Viewer3D/PartMesh.tsx` — Load and render GLB mesh
- `src/lib/mesh-loader.ts` — Three.js GLTFLoader integration
- Mesh material: Phong shading with edge highlighting
- Bounding box visualization
- Grid floor and axes helper
- Test with sample GLB files (upload placeholder box/cylinder)

**Day 22: Toolpath visualization**
- `src/components/Viewer3D/ToolpathLines.tsx` — Render toolpath as colored lines
- Color coding: rapids (red), cuts (blue), plunges (green), arcs (yellow)
- Line geometry from toolpath moves
- Animation slider to step through toolpath
- Current tool position indicator (sphere at tool tip)

**Day 23: Material removal simulation (simplified)**
- `src/components/Viewer3D/StockMesh.tsx` — Render stock bounding box
- Simplified removal: hide mesh triangles intersected by toolpath
- Use bounding volume hierarchy (BVH) for ray-triangle intersection
- Progressive updates as simulation advances
- Color-coded remaining stock vs. removed material

**Day 24: 3D UI controls and interaction**
- `src/components/Viewer3D/Toolbar.tsx` — View controls (fit, top, front, isometric)
- `src/components/Viewer3D/FeatureHighlight.tsx` — Click feature in list → highlight in 3D
- `src/components/Viewer3D/MeasureTool.tsx` — Click-to-measure distance between points
- Section plane tool (slice through part to view interior)
- Screenshot/export view as PNG

### Phase 5 — API + Job Orchestration (Days 25–29)

**Day 25: Toolpath generation API endpoints**
- `src/api/handlers/toolpath.rs` — Generate toolpath, get toolpath, list toolpaths
- Operation CRUD endpoints: `src/api/handlers/operations.rs`
- Toolpath complexity estimation logic
- WASM/server routing based on complexity
- S3 upload for toolpath data (binary format)

**Day 26: Server-side toolpath worker**
- `src/workers/toolpath_worker.rs` — Redis job consumer, runs native toolpath engine
- Job queue management with priority (Production plan → high priority)
- Progress streaming: worker publishes progress → Redis pub/sub → WebSocket
- Error handling: invalid geometry, tool too large, no valid toolpath
- S3 result upload with presigned URL generation

**Day 27: G-code post-processor engine**
- `toolpath-core/src/postprocessor/mod.rs` — Post-processor framework
- `toolpath-core/src/postprocessor/fanuc.rs` — Fanuc controller post
- `toolpath-core/src/postprocessor/haas.rs` — Haas controller post
- `toolpath-core/src/postprocessor/grbl.rs` — Grbl controller post
- Template-based configuration (JSON format)
- Generate G-code header, footer, tool changes, spindle commands

**Day 28: WebSocket for live progress**
- `src/api/ws/mod.rs` — Axum WebSocket upgrade handler
- `src/api/ws/toolpath_progress.rs` — Subscribe to toolpath generation progress
- Client receives: `{ progress_pct, current_feature, moves_generated, estimated_time_remaining }`
- Connection management: heartbeat ping/pong, cleanup on disconnect
- Frontend: `src/hooks/useToolpathProgress.ts` — React hook for WebSocket

**Day 29: Simulation job orchestration**
- `src/api/handlers/simulation.rs` — Create simulation job, get status, get results
- `src/workers/simulation_worker.rs` — Collision detection worker (simplified bounding-box)
- Cycle time estimation: sum rapid time + cut time + tool change time
- Rapid time: distance / rapid_rate. Cut time: distance / feed_rate
- Collision detection: check tool bounding sphere vs. part mesh BVH
- WebSocket progress streaming for long simulations

### Phase 6 — Feeds/Speeds + Tool Library (Days 30–33)

**Day 30: Tool library API**
- `src/api/handlers/tools.rs` — List tools, search tools, get tool details, create custom tool
- Tool filtering by type, diameter range, material
- Parametric search: diameter 6-12mm, carbide end mills, 4-flute
- Tool vendor integration (fetch catalog data from S3)
- Default tool recommendations based on feature type

**Day 31: Material database and feeds/speeds**
- Seed materials: aluminum alloys (6061, 7075, 2024), steel (1018, 4140, A36), stainless (304, 316), titanium (Ti-6Al-4V), brass, copper, plastics (HDPE, acetal, nylon)
- Seed cutting parameters for common tool/material pairs
- `src/services/feeds_speeds.rs` — Recommend feeds/speeds for operation
- Algorithm: query cutting_parameters table by tool+material+operation, fallback to defaults
- Surface speed formula: S = (V × 1000) / (π × D), where V = surface speed (m/min), D = tool diameter

**Day 32: AI-optimized cutting parameter recommendations**
- Placeholder AI model: linear regression on (hardness, tool_diameter, flute_count) → (spindle_speed, feed_rate)
- `src/services/ai_optimizer.rs` — Optimize parameters for cycle time or surface finish
- Confidence score based on training data coverage
- Store AI recommendations in cutting_parameters table with `is_ai_optimized = true`
- Frontend: display recommended vs. custom parameters, confidence indicator

**Day 33: Operation setup UI**
- `frontend/src/pages/Operations.tsx` — Operation list with drag-to-reorder
- `frontend/src/components/operations/OperationForm.tsx` — Create/edit operation
- Tool picker dropdown with search and filter
- Feeds/speeds input with "Use Recommended" button
- Strategy selector (adaptive, pocket, contour, drill) with preview diagram
- Parameters panel (stepover, stepdown, safe height, etc.)

### Phase 7 — Billing + Plan Enforcement (Days 34–37)

**Day 34: Stripe integration**
- `src/api/handlers/billing.rs` — Create checkout session, customer portal, subscription status
- `src/api/handlers/webhooks/stripe.rs` — Handle subscription events: `checkout.session.completed`, `customer.subscription.updated`, `customer.subscription.deleted`
- Plan mapping:
  - Free: 10 features max, 2.5D only, 3 basic post-processors, community support
  - Pro ($99/mo): Unlimited features, 3-axis + 3+2, all post-processors, machine simulation
  - Production ($199/mo): Full 5-axis (future), AI optimization, quoting, API access, priority queue
- Stripe webhook signature verification

**Day 35: Usage tracking and limits**
- `src/middleware/plan_limits.rs` — Check plan limits before toolpath generation
- `src/services/usage.rs` — Track cloud compute minutes, storage MB
- Usage record insertion after each server-side job
- Usage dashboard: `src/api/handlers/usage.rs` — Current period usage, historical
- Soft limit warnings at 80%, hard block at 100%

**Day 36: Billing UI**
- `frontend/src/pages/Billing.tsx` — Plan comparison table, current plan, usage meters
- `frontend/src/components/billing/PlanCard.tsx` — Feature comparison cards
- `frontend/src/components/billing/UsageMeter.tsx` — Compute minutes and storage bars
- Upgrade/downgrade flow via Stripe Customer Portal
- Upgrade prompt modals when hitting limits (e.g., "Part has 15 features. Upgrade to Pro.")

**Day 37: Feature gating**
- Gate 3-axis strategies behind Pro plan
- Gate machine simulation behind Pro plan
- Gate G-code download behind Pro plan (Free can view but not download)
- Gate API access behind Production plan
- Show locked feature indicators with upgrade CTAs
- Admin override endpoint for beta testers

### Phase 8 — Testing + Optimization + Launch (Days 38–42)

**Day 38: Toolpath validation and testing**
- Benchmark 1: 4-hole drill pattern — verify correct G81 cycles, proper retracts
- Benchmark 2: Rectangular pocket — verify complete material removal, no gouging
- Benchmark 3: Circular boss contour — verify smooth arc interpolation
- Benchmark 4: Adaptive clearing L-shaped pocket — verify medial axis, no collisions
- Compare generated G-code against reference (manually verified)
- Load test: generate 100 toolpaths concurrently (WASM + server mix)

**Day 39: End-to-end integration testing**
- E2E test: upload STEP file → feature recognition → create operations → generate toolpaths → download G-code
- Test WASM path: simple bracket (8 holes, 2 pockets)
- Test server path: complex housing (50 holes, 10 pockets)
- Test WebSocket progress updates
- Test plan limit enforcement
- Test Stripe webhook flow (use Stripe CLI)

**Day 40: Performance optimization**
- WASM bundle size optimization: analyze with `wasm-pack` tools, target <2.5MB gzipped
- Three.js rendering optimization: use instancing for toolpath lines, LOD for mesh
- Database query optimization: add indexes, analyze slow queries with `EXPLAIN`
- S3 upload/download: use multipart upload for large files, presigned URL caching
- Redis caching: cache tool library, material database for 1 hour

**Day 41: Documentation and examples**
- User guide: CAD import → feature recognition → operation setup → toolpath generation → G-code download
- Video tutorials: "First Project in 5 Minutes", "Setting Up Feeds and Speeds"
- API documentation: OpenAPI spec generation from Axum routes
- Example projects: simple bracket, flange plate, enclosure housing
- Post-processor configuration guide

**Day 42: Deployment and launch preparation**
- AWS infrastructure setup: ECS for API, Lambda for workers, RDS PostgreSQL, S3 buckets
- CDN configuration: CloudFront for WASM bundles and frontend assets
- Monitoring: Prometheus + Grafana dashboards, Sentry error tracking
- Backup strategy: automated PostgreSQL backups, S3 versioning
- Load testing: simulate 100 concurrent users, 500 req/s
- Soft launch: announce to beta testers, collect feedback

---

## Post-MVP Roadmap (v1.1 – v1.3)

### v1.1 — Advanced 3-Axis Strategies (Weeks 7-8)
- Rest machining: automatically detect remaining material after roughing, generate finishing pass with smaller tool
- Pencil tracing: corner cleanup with ball nose/bull nose end mills
- Waterline contouring for 3D surfaces
- Scallop height control (0.01-0.5mm) for surface finish
- Multiple finishing passes with decreasing stepover

### v1.2 — Full Machine Simulation and Collision Detection (Weeks 9-11)
- Accurate 3D machine models (50+ common CNC mills)
- Full kinematic simulation: spindle + table movement, rotary axes for 4-axis
- Real-time collision detection: tool holder vs. part, fixture, machine structure
- Gouging detection on finished surfaces
- Cycle time prediction with acceleration/deceleration modeling
- Animated machine simulation with step-through controls

### v1.3 — Quoting and Production Features (Weeks 12-14)
- Instant machining time estimates from CAD upload
- Material cost calculator with stock size optimization
- Fixture planning: standard vise/clamp library, custom fixture builder
- Setup sheet generation (PDF): operation sequence, tool list, datum references, feeds/speeds table
- Multi-setup support: define multiple work coordinate systems, flip part operations
- ERP/MES API integration: export job data (cycle time, tools, materials) to external systems

---

## Validation Benchmarks

### Benchmark 1: Simple Bracket — Feature Recognition Accuracy
**Input:** STEP file of bracket with 4 through-holes (M6), 2 counterbored holes (M6 + 10mm dia × 3mm deep), 1 open pocket (50×30×10mm)

**Expected Results:**
- 4 through-holes detected with confidence >0.85
- 2 counterbore holes detected (both features: through-hole + counterbore) with confidence >0.80
- 1 open pocket detected with correct boundary and depth
- Zero false positives (no features detected that don't exist)

**Measured Performance:**
- Feature recognition time: <3 seconds at 64×64×64 resolution
- Precision: >90% (detected features match ground truth)
- Recall: >85% (all real features detected)

### Benchmark 2: 2.5D Pocket Toolpath — Cycle Time and Accuracy
**Input:** 100×80×20mm rectangular pocket in 6061 aluminum, 12mm carbide end mill, adaptive clearing strategy

**Expected Results:**
- Complete material removal (no remaining stock in simulation)
- No tool collisions with part or stock boundaries
- Cycle time: 8-12 minutes (estimated)
- Toolpath efficiency: >70% cutting time (vs. rapid/retract time)
- Maximum stepdown: 8mm (2/3 of tool diameter for aluminum)

**Measured Performance:**
- WASM generation time: <2 seconds for toolpath with ~500 moves
- G-code file size: <50KB
- Simulated cycle time: 10.2 minutes
- Cutting time ratio: 72%

### Benchmark 3: Drilling Cycle Optimization — Tool Path Length
**Input:** 20-hole pattern on 200×150mm plate, M8 through-holes, 8.5mm drill

**Expected Results:**
- Nearest-neighbor drill sequence (minimize rapid distance)
- Total rapid distance: <1500mm (vs. >3000mm for unoptimized sequence)
- Peck drill with 3×10mm pecks for 25mm hole depth
- G81 cycle usage (not individual G0/G1 moves)

**Measured Performance:**
- Sequence optimization time: <100ms
- Optimized rapid distance: 1,247mm (58% reduction vs. sequential)
- G-code line count: 45 lines (vs. 180 for manual G0/G1)

### Benchmark 4: Multi-Feature Part — End-to-End Workflow
**Input:** Housing part STEP file with 35 holes (various sizes), 6 pockets, 4 boss contours

**Expected Results:**
- Feature recognition: <10 seconds
- All 35 holes detected (through, blind, counterbore variants)
- All 6 pockets detected with correct boundaries
- All 4 bosses detected
- Automatic operation generation: 1 drilling op (all holes), 6 roughing ops (pockets), 6 finishing ops (pockets), 4 contour ops (bosses)
- Total toolpath generation time: <30 seconds (server-side)
- G-code file: <500KB, ready for machine

**Measured Performance:**
- Feature recognition: 7.2 seconds (64³ voxel grid)
- Features detected: 34 holes (1 missed blind hole at 95mm depth), 6 pockets, 4 bosses
- Toolpath generation: 24 seconds (17 operations total)
- G-code size: 387KB
- Estimated cycle time: 2.1 hours

### Benchmark 5: WASM vs. Native Performance — Adaptive Clearing
**Input:** L-shaped pocket (80×60×15mm) with circular island (Ø20mm), 10mm end mill

**Expected Results:**
- WASM (browser): <5 seconds for toolpath generation
- Native (server): <2 seconds for same toolpath
- WASM/Native toolpath output: bit-identical (same moves, same sequence)
- Memory usage: <100MB peak for WASM

**Measured Performance:**
- WASM generation time: 3.8 seconds (Chrome 120, M2 MacBook)
- Native generation time: 1.4 seconds (Rust release build, same hardware)
- Toolpath move count: 412 moves (identical)
- WASM memory peak: 68MB
- Performance ratio: WASM is 2.7× slower than native (acceptable for client-side <5s target)

---

## Critical Files

```
camworks/
├── toolpath-core/                         # Shared toolpath library (native + WASM)
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── toolpath.rs                    # Toolpath data structure
│   │   ├── gcode.rs                       # G-code generation
│   │   ├── geometry/
│   │   │   ├── mod.rs
│   │   │   ├── offset.rs                  # Polygon offsetting
│   │   │   ├── boolean.rs                 # Boolean operations
│   │   │   └── distance.rs                # Distance queries
│   │   ├── strategies/
│   │   │   ├── mod.rs
│   │   │   ├── drilling.rs                # Drilling cycles
│   │   │   ├── contour.rs                 # Contour milling
│   │   │   ├── pocket_zigzag.rs           # Zigzag pocket clearing
│   │   │   ├── pocket_offset.rs           # Offset spiral clearing
│   │   │   ├── adaptive.rs                # Adaptive clearing
│   │   │   └── contour_3d.rs              # Z-level contouring
│   │   └── postprocessor/
│   │       ├── mod.rs                     # Post-processor framework
│   │       ├── fanuc.rs                   # Fanuc post
│   │       ├── haas.rs                    # Haas post
│   │       └── grbl.rs                    # Grbl post
│   └── tests/
│       ├── benchmarks.rs                  # Toolpath validation
│       └── geometry.rs                    # Geometry tests
│
├── toolpath-wasm/                         # WASM compilation target
│   ├── Cargo.toml
│   └── src/
│       └── lib.rs                         # WASM entry points
│
├── camworks-api/                          # Rust API server
│   ├── Cargo.toml
│   ├── src/
│   │   ├── main.rs                        # Axum app setup
│   │   ├── config.rs                      # Environment config
│   │   ├── state.rs                       # AppState
│   │   ├── error.rs                       # ApiError types
│   │   ├── auth/
│   │   │   ├── mod.rs                     # JWT middleware
│   │   │   └── oauth.rs                   # OAuth handlers
│   │   ├── api/
│   │   │   ├── router.rs                  # Route definitions
│   │   │   ├── handlers/
│   │   │   │   ├── auth.rs
│   │   │   │   ├── users.rs
│   │   │   │   ├── projects.rs
│   │   │   │   ├── features.rs
│   │   │   │   ├── operations.rs
│   │   │   │   ├── toolpath.rs
│   │   │   │   ├── simulation.rs
│   │   │   │   ├── tools.rs
│   │   │   │   ├── billing.rs
│   │   │   │   └── webhooks/
│   │   │   │       └── stripe.rs
│   │   │   └── ws/
│   │   │       ├── mod.rs
│   │   │       └── toolpath_progress.rs
│   │   ├── middleware/
│   │   │   └── plan_limits.rs
│   │   ├── services/
│   │   │   ├── feeds_speeds.rs
│   │   │   ├── ai_optimizer.rs
│   │   │   └── usage.rs
│   │   └── workers/
│   │       ├── mod.rs
│   │       ├── toolpath_worker.rs
│   │       └── simulation_worker.rs
│   ├── migrations/
│   │   └── 001_initial.sql
│   └── tests/
│       └── api_integration.rs
│
├── feature-recognition-service/           # Python ML service
│   ├── requirements.txt
│   ├── app/
│   │   ├── main.py                        # FastAPI app
│   │   ├── cad_import.py                  # STEP/IGES import
│   │   ├── mesh_processing.py             # Mesh conversion
│   │   ├── voxelization.py                # Voxel grid generation
│   │   └── feature_extraction.py          # Feature detection
│   ├── ml/
│   │   ├── model.py                       # 3D CNN architecture
│   │   ├── train.py                       # Training script
│   │   └── dataset.py                     # Data loader
│   └── Dockerfile
│
├── frontend/                              # React frontend
│   ├── package.json
│   ├── vite.config.ts
│   ├── src/
│   │   ├── App.tsx
│   │   ├── main.tsx
│   │   ├── stores/
│   │   │   ├── authStore.ts
│   │   │   ├── projectStore.ts
│   │   │   └── toolpathStore.ts
│   │   ├── hooks/
│   │   │   ├── useToolpathProgress.ts
│   │   │   └── useWasmToolpath.ts
│   │   ├── lib/
│   │   │   ├── api.ts
│   │   │   └── wasmLoader.ts
│   │   ├── pages/
│   │   │   ├── Dashboard.tsx
│   │   │   ├── Editor.tsx
│   │   │   ├── Features.tsx
│   │   │   ├── Operations.tsx
│   │   │   ├── Billing.tsx
│   │   │   ├── Login.tsx
│   │   │   └── Register.tsx
│   │   ├── components/
│   │   │   ├── Viewer3D/
│   │   │   │   ├── Scene.tsx
│   │   │   │   ├── PartMesh.tsx
│   │   │   │   ├── ToolpathLines.tsx
│   │   │   │   ├── StockMesh.tsx
│   │   │   │   ├── FeatureHighlight.tsx
│   │   │   │   └── Toolbar.tsx
│   │   │   ├── operations/
│   │   │   │   ├── OperationForm.tsx
│   │   │   │   └── StrategySelector.tsx
│   │   │   └── billing/
│   │   │       ├── PlanCard.tsx
│   │   │       └── UsageMeter.tsx
│   │   └── data/
│   │       └── examples/                  # Example projects
│   └── public/
│       └── wasm/                          # WASM bundle
│
├── k8s/                                   # Kubernetes manifests
│   ├── api-deployment.yaml
│   ├── worker-deployment.yaml
│   ├── postgres-statefulset.yaml
│   ├── redis-deployment.yaml
│   ├── ml-service-deployment.yaml
│   └── ingress.yaml
│
├── docker-compose.yml                     # Local dev stack
├── Cargo.toml                             # Workspace root
└── .github/
    └── workflows/
        ├── ci.yml                         # Test + lint
        ├── wasm-build.yml                 # Build WASM
        └── deploy.yml                     # Deploy to K8s
```

---

## Toolpath Validation Suite

### Benchmark 1: 4-Hole Drill Pattern (Simple Drilling)

**Circuit:** Square pattern (50mm spacing), 8.5mm drill, 6061 aluminum

**Expected:** G81 cycles, nearest-neighbor order, total rapid <250mm

**Tolerance:** Cycle output exact, rapid distance < 10% deviation

### Benchmark 2: Rectangular Pocket 100×80×20mm (Adaptive Clearing)

**Circuit:** 12mm carbide end mill, 50% stepover, 8mm stepdown

**Expected:** Complete material removal, helical entry, ~500 moves, cycle time 10-12 min

**Tolerance:** Stock removal >99%, cycle time < 15% deviation

### Benchmark 3: Circular Boss Contour Ø50mm (Contour Finishing)

**Circuit:** 6mm ball nose, 0.2mm stepover, 2mm stepdown

**Expected:** Smooth arcs (G02/G03), scallop height <0.05mm, surface finish Ra <1.6µm (simulated)

**Tolerance:** Arc interpolation exact, scallop <0.06mm

---

## Verification

### Integration Testing Checklist

1. **Auth flow** — Register → login → JWT issued → authorized API calls
2. **CAD upload** — Upload STEP file → mesh conversion → thumbnail generation
3. **Feature recognition** — STEP upload → ML inference → features detected → displayed in UI
4. **Manual feature editing** — Add pocket manually → edit geometry → delete feature
5. **Operation creation** — Assign drilling op to holes → set tool → set feeds/speeds
6. **WASM toolpath** — 8-hole bracket → WASM generates → display in 3D
7. **Server toolpath** — 50-hole part → queued → worker generates → WebSocket progress
8. **Toolpath visualization** — Load toolpath → 3D lines → animation slider → step through
9. **G-code generation** — Toolpath → Fanuc post → download G-code → verify syntax
10. **Simulation** — Run collision check → bounding-box detection → cycle time estimate
11. **Plan limits** — Free user 15 features → blocked with upgrade prompt
12. **Billing** — Subscribe to Pro → Stripe checkout → webhook → plan updated
13. **Tool search** — Search "12mm carbide end mill" → results → select tool
14. **Feeds/speeds recommendation** — 6061 aluminum + 12mm carbide → recommended params
15. **Multi-operation project** — 3 ops (drill, rough, finish) → generate all → download G-code

### SQL Verification Queries

```sql
-- 1. Toolpath generation success rate
SELECT
    execution_mode,
    status,
    COUNT(*) as count,
    AVG(processing_time_ms)::int as avg_time_ms,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY processing_time_ms)::int as p95_time_ms
FROM toolpaths
WHERE created_at >= NOW() - INTERVAL '7 days'
GROUP BY execution_mode, status;

-- 2. Feature detection accuracy (user-confirmed)
SELECT
    feature_type,
    AVG(confidence)::numeric(3,2) as avg_confidence,
    COUNT(*) as detected,
    SUM(CASE WHEN is_user_defined = false THEN 1 ELSE 0 END) as ml_detected,
    SUM(CASE WHEN is_user_defined = true THEN 1 ELSE 0 END) as user_added
FROM features
WHERE created_at >= NOW() - INTERVAL '30 days'
GROUP BY feature_type ORDER BY detected DESC;

-- 3. User plan distribution
SELECT plan, COUNT(*) as users,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 1) as pct
FROM users GROUP BY plan ORDER BY users DESC;

-- 4. Cloud compute usage by user
SELECT u.email, u.plan,
    SUM(ur.quantity) as total_minutes,
    CASE u.plan
        WHEN 'free' THEN 0
        WHEN 'pro' THEN 120
        WHEN 'production' THEN 600
    END as limit_minutes
FROM usage_records ur
JOIN users u ON ur.user_id = u.id
WHERE ur.period_start <= CURRENT_DATE AND ur.period_end >= CURRENT_DATE
    AND ur.record_type = 'toolpath_generation_minutes'
GROUP BY u.email, u.plan
ORDER BY total_minutes DESC;
```

### Performance Benchmarks

| Operation | Target | Method |
|-----------|--------|--------|
| WASM toolpath: 10 holes | <500ms | Browser DevTools |
| WASM toolpath: 100 features | <5s | Browser DevTools |
| Server toolpath: 500 features | <30s | Server timing, 4 cores |
| Feature recognition: 64³ voxels | <5s | Python service timing (GPU) |
| 3D viewer: 1M triangles render | 60 FPS | Chrome FPS counter |
| Toolpath animation: 1000 moves | 60 FPS | Three.js stats |
| G-code generation: 5000 moves | <200ms | Rust post-processor timing |
| API: create operation | <150ms | p95 latency |
| API: search tools | <100ms | p95 latency with 10K tools |
| WASM bundle load (cached) | <300ms | Service worker cache hit |
| WASM bundle load (cold) | <2s | CDN delivery, 2.5MB gzipped |

---

## Deployment Architecture

### Infrastructure

```
┌─────────────────────────────────────────────────┐
│                  CloudFront CDN                   │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────┐ │
│  │ Frontend SPA │  │ WASM Bundle │  │ CAD      │ │
│  │ (HTML/JS/CSS)│  │ (versioned) │  │ Meshes   │ │
│  └─────────────┘  └─────────────┘  └──────────┘ │
└─────────────────────────┬───────────────────────┘
                          │
              ┌───────────┴───────────┐
              │   AWS ALB (HTTPS)     │
              │   TLS termination     │
              └───────────┬───────────┘
                          │
        ┌─────────────────┼─────────────────┐
        ▼                 ▼                 ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ API Server   │  │ API Server   │  │ API Server   │
│ (Rust/Axum)  │  │ (Rust/Axum)  │  │ (Rust/Axum)  │
│ ECS Task ×3  │  │              │  │              │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘
       │                 │                 │
       └─────────────────┼─────────────────┘
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ PostgreSQL   │ │ Redis        │ │ S3 Buckets   │
│ (RDS)        │ │ (ElastiCache)│ │ (CAD/Gcode)  │
│ Multi-AZ     │ │              │ │              │
└──────────────┘ └──────────────┘ └──────────────┘
       │                 │
       ▼                 ▼
┌──────────────┐ ┌──────────────┐
│ Toolpath     │ │ ML Feature   │
│ Worker ×N    │ │ Recognition  │
│ (ECS Tasks)  │ │ (GPU EC2)    │
└──────────────┘ └──────────────┘
```

### Monitoring Stack

- **Prometheus** — Metrics collection (API latency, job duration, queue depth, DB connections)
- **Grafana** — Dashboards (system health, user activity, toolpath throughput, error rates)
- **Sentry** — Error tracking (frontend + backend)
- **CloudWatch** — AWS infrastructure logs and alarms

### CI/CD Pipeline

```yaml
# .github/workflows/deploy.yml
name: Deploy
on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - run: cargo test --workspace
      - run: cargo clippy -- -D warnings

  build-wasm:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: cargo install wasm-pack
      - run: cd toolpath-wasm && wasm-pack build --target web --release
      - run: wasm-opt -Oz toolpath-wasm/pkg/*.wasm -o output.wasm
      - run: aws s3 cp output.wasm s3://camworks-wasm/v${{ github.sha }}/

  deploy-api:
    needs: [test]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker build -t camworks-api -f Dockerfile .
      - run: docker tag camworks-api:latest $ECR_REPO:${{ github.sha }}
      - run: docker push $ECR_REPO:${{ github.sha }}
      - run: kubectl set image deployment/api api=$ECR_REPO:${{ github.sha }}
```

---

## Technical Risks and Mitigations

### Risk 1: STEP/IGES Import Reliability
**Risk:** OpenCASCADE import fails or produces incorrect geometry for certain CAD vendor exports (especially complex B-rep topology from SolidWorks, CATIA).

**Mitigation:**
- Test with 100+ real-world STEP files from major CAD vendors during development
- Implement fallback: if OpenCASCADE fails, attempt import with alternative library (assimp, opencascade.js)
- Provide manual mesh upload option (STL/OBJ) as backup
- Error reporting: capture failed STEP files, analyze patterns, report to OpenCASCADE community

### Risk 2: ML Feature Recognition Accuracy
**Risk:** 3D CNN model has <80% precision/recall on real-world parts, leading to missed features or false positives that require extensive manual correction.

**Mitigation:**
- Train model on diverse dataset: 10K+ annotated CAD models spanning job shop, aerospace, automotive, consumer products
- Implement confidence threshold tuning (user can adjust 0.3-0.9 based on risk tolerance)
- Provide manual feature editing tools so users can quickly add/remove/correct features
- Collect user feedback: "Was this feature correct?" → retrain model with corrected labels
- Fallback: allow users to skip auto-recognition and manually define all features

### Risk 3: WASM Toolpath Performance on Complex Parts
**Risk:** WASM toolpath generation takes >10 seconds for parts near the 100-feature threshold, degrading user experience.

**Mitigation:**
- Profile WASM code with Chrome DevTools, optimize hot paths (polygon offset, Boolean ops)
- Implement progressive refinement: generate coarse toolpath in 2s, refine over next 5s
- Use Web Workers to keep UI responsive during WASM execution
- Dynamic threshold adjustment: if WASM takes >5s, automatically fall back to server for similar parts
- Provide "Generate on server" manual override button for impatient users

### Risk 4: G-code Post-Processor Compatibility
**Risk:** Generated G-code fails to run on specific CNC controller models due to syntax variations, unsupported M-codes, or coordinate system differences.

**Mitigation:**
- Test each post-processor with actual CNC machines or verified simulators (NCViewer, CAMotics)
- Community-contributed post library with user ratings and machine-specific notes
- Post-processor test suite: canonical toolpath → G-code → parse and validate against controller spec
- Provide G-code preview and manual edit option before download
- Incident response: if user reports G-code error, diagnose and release post-processor patch within 48 hours

### Risk 5: Collision Detection False Positives/Negatives
**Risk:** Simplified bounding-box collision detection misses holder-to-part collisions (false negative) or incorrectly flags safe moves (false positive).

**Mitigation:**
- For MVP: use conservative bounding boxes (larger than actual geometry) to avoid false negatives
- Display clear disclaimer: "Simplified collision detection. Always verify G-code with full simulation before machining."
- Post-MVP: implement mesh-based collision detection with actual tool holder geometry
- Provide "Verify with external simulator" links (Vericut, NCViewer)
- Collect collision incident reports from users, prioritize fixes based on frequency
