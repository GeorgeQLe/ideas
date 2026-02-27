# 93. AntennaSynth — Antenna Design and Synthesis Platform

## Implementation Plan

**MVP Scope:** Browser-based antenna synthesis engine with 30 parametric antenna topologies (patch variants, dipole, monopole, PIFA, helix, horn, Vivaldi, slot) and specification-driven dimensioning (frequency, bandwidth, gain, polarization, size constraints), custom Method of Moments (MoM) electromagnetic solver with RWG basis functions and multilayer planar Green's function for microstrip antennas compiled to WebAssembly for client-side execution of elements ≤10K unknowns and server-side Rust-native execution for larger elements and arrays, S-parameter computation (S11, VSWR, input impedance) with Smith chart visualization, 3D far-field radiation pattern rendering via WebGL with gain/directivity/beamwidth computation, interactive array factor analysis for rectangular/triangular lattice with up to 64 elements and real-time amplitude/phase taper visualization, substrate material library with 20+ RF laminates (Rogers, Taconic, Isola) including frequency-dependent permittivity and loss tangent, basic impedance matching tool with L/Pi/T network synthesis, Stripe billing with three tiers (Free / Pro $179/mo / Advanced $399/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| MoM Solver | Rust (native + WASM) | RWG basis functions, multilayer Green's function via `ndarray` + `blas-src` |
| WASM Compilation | `wasm-pack` + `wasm-bindgen` | Client-side solver for elements ≤10K unknowns |
| Array Engine | Rust → WASM + native | Array factor computation, beamforming weight calculation |
| Optimization | Python 3.12 (FastAPI) | Genetic algorithm, particle swarm, CMA-ES for synthesis |
| Database | PostgreSQL 16 | Antenna designs, substrate library, simulation metadata |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | Mesh files, field data, radiation patterns, design thumbnails |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| 3D Visualization | Three.js + WebGL 2.0 | Antenna geometry, radiation pattern 3D rendering |
| 2D Plotting | D3.js | Smith charts, S-parameter plots, array factor polar plots |
| Real-time | WebSocket (Axum) | Live simulation progress, convergence monitoring |
| Job Queue | Redis 7 + Tokio tasks | Server-side simulation job management |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | WASM bundle delivery, static frontend assets |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, error tracking |
| CI/CD | GitHub Actions | WASM build pipeline, Rust tests, Docker image push |

### Key Architecture Decisions

1. **Hybrid client/server MoM solver with WASM threshold at 10K unknowns**: Antenna elements with ≤10K RWG basis functions (covers 90%+ of single-element designs including patches, dipoles, monopoles, PIFAs, slots, simple horns) run entirely in the browser via WASM, providing instant feedback with zero server cost. Elements exceeding 10K unknowns (large arrays, complex 3D geometries) are submitted to the Rust-native server solver which handles massive MIMO arrays and full mutual coupling analysis. The threshold is configurable per plan tier.

2. **Custom MoM solver in Rust rather than wrapping NEC2/FEKO**: Building a custom MoM solver in Rust gives us full control over meshing algorithms, WASM compilation, and GPU acceleration for array analysis. NEC2's Fortran codebase is limited to wire structures and has no native WASM support. Our Rust solver uses RWG (Rao-Wilton-Glisson) basis functions for arbitrary 3D surface meshes and includes multilayer planar Green's function for efficient microstrip antenna analysis without meshing the substrate volume.

3. **Three.js for 3D antenna geometry and radiation pattern visualization**: Three.js provides hardware-accelerated 3D rendering with intuitive camera controls, material systems for visualizing current distribution on antenna surfaces, and support for complex geometries (patches, horns, reflectors). Radiation patterns are rendered as surface plots (gain magnitude) with color mapping and transparent clipping planes for cross-section views.

4. **Multilayer planar Green's function for microstrip efficiency**: The majority of practical antennas (patches, PIFAs, slots) are printed on multilayer PCB substrates. Using multilayer Green's function in the spectral domain (Sommerfeld integral formulation) avoids meshing the substrate volume, reducing unknowns by 10-100x compared to volumetric FDTD while maintaining accuracy for planar geometries.

5. **S3 for antenna catalog with PostgreSQL metadata**: Antenna topologies (parametric geometry generators, default dimensions, feeding mechanisms) are stored as JSON schema in PostgreSQL for searchable metadata (frequency range, bandwidth, gain, polarization type, physical size). Pre-computed radiation patterns and validation data are stored in S3, allowing the library to scale to 1000+ antenna variants without bloating the database.

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";  -- For fuzzy text search on antenna catalog

-- Users
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email TEXT UNIQUE NOT NULL,
    password_hash TEXT,  -- NULL for OAuth-only users
    name TEXT NOT NULL,
    avatar_url TEXT,
    auth_provider TEXT DEFAULT 'email',  -- email | google | github
    auth_provider_id TEXT,
    plan TEXT NOT NULL DEFAULT 'free',  -- free | pro | advanced
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX users_email_idx ON users(email);
CREATE INDEX users_stripe_idx ON users(stripe_customer_id);

-- Organizations (for team collaboration)
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

-- Antenna Design Projects
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    antenna_type TEXT NOT NULL,  -- patch | dipole | monopole | pifa | horn | helix | vivaldi | slot | array
    topology_id TEXT,  -- Reference to catalog topology
    geometry_data JSONB NOT NULL DEFAULT '{}',  -- Full 3D geometry (vertices, triangles, feeding)
    substrate_id UUID,  -- Reference to substrate material
    settings JSONB DEFAULT '{}',  -- Design constraints, frequency range, target specs
    is_public BOOLEAN NOT NULL DEFAULT false,
    forked_from UUID REFERENCES projects(id) ON DELETE SET NULL,
    thumbnail_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);
CREATE INDEX projects_org_idx ON projects(org_id);
CREATE INDEX projects_type_idx ON projects(antenna_type);
CREATE INDEX projects_updated_idx ON projects(updated_at DESC);

-- Electromagnetic Simulations
CREATE TABLE simulations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    sim_type TEXT NOT NULL,  -- impedance | pattern | array_factor | full_array
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | running | completed | failed | cancelled
    execution_mode TEXT NOT NULL DEFAULT 'wasm',  -- wasm | server
    frequency_points REAL[] NOT NULL,  -- GHz
    num_unknowns INTEGER NOT NULL DEFAULT 0,
    mesh_data_url TEXT,  -- S3 URL for mesh file
    results_url TEXT,  -- S3 URL for result data
    results_summary JSONB,  -- Quick-access summary (S11, gain, beamwidth, efficiency)
    error_message TEXT,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    wall_time_ms INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX sims_project_idx ON simulations(project_id);
CREATE INDEX sims_user_idx ON simulations(user_id);
CREATE INDEX sims_status_idx ON simulations(status);
CREATE INDEX sims_created_idx ON simulations(created_at DESC);

-- Server Simulation Jobs (for server-side execution only)
CREATE TABLE simulation_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    simulation_id UUID NOT NULL REFERENCES simulations(id) ON DELETE CASCADE,
    worker_id TEXT,
    cores_allocated INTEGER DEFAULT 4,
    memory_mb INTEGER DEFAULT 8192,
    priority INTEGER DEFAULT 0,  -- Higher = more urgent
    progress_pct REAL DEFAULT 0.0,
    convergence_data JSONB DEFAULT '[]',  -- [{iteration, residual, condition_number}]
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX jobs_sim_idx ON simulation_jobs(simulation_id);
CREATE INDEX jobs_worker_idx ON simulation_jobs(worker_id);

-- Antenna Topology Catalog
CREATE TABLE antenna_catalog (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    topology_id TEXT UNIQUE NOT NULL,  -- e.g., "rectangular_patch", "circular_patch", "pifa_planar"
    name TEXT NOT NULL,
    category TEXT NOT NULL,  -- patch | dipole | monopole | slot | horn | helix | vivaldi | pifa | ifa
    polarization TEXT[] DEFAULT '{}',  -- linear_vertical | linear_horizontal | circular_left | circular_right | dual
    frequency_range_ghz NUMRANGE NOT NULL,  -- Typical operating range
    gain_range_dbi NUMRANGE,  -- Typical gain range
    bandwidth_pct_range NUMRANGE,  -- Typical fractional bandwidth
    size_wavelengths JSONB,  -- {length, width, height} in wavelengths
    description TEXT,
    geometry_generator TEXT NOT NULL,  -- Rust function name for parametric generation
    default_params JSONB NOT NULL,  -- Default parameter values
    validation_data_url TEXT,  -- S3 URL to reference simulations
    is_verified BOOLEAN DEFAULT false,
    tags TEXT[] DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX catalog_category_idx ON antenna_catalog(category);
CREATE INDEX catalog_name_trgm_idx ON antenna_catalog USING gin(name gin_trgm_ops);
CREATE INDEX catalog_tags_idx ON antenna_catalog USING gin(tags);

-- Substrate Materials Library
CREATE TABLE substrates (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT UNIQUE NOT NULL,  -- e.g., "Rogers RO4003C"
    manufacturer TEXT,
    material_type TEXT NOT NULL,  -- pcb_laminate | ltcc | flex | ceramic | 3d_printed
    permittivity_re REAL NOT NULL,  -- Relative permittivity (real part)
    permittivity_im REAL DEFAULT 0.0,  -- Imaginary part (loss)
    loss_tangent REAL DEFAULT 0.0,
    thickness_mm REAL,  -- Standard thickness (if applicable)
    frequency_dependent BOOLEAN DEFAULT false,
    permittivity_data JSONB,  -- [{freq_ghz, eps_re, eps_im, tan_delta}]
    datasheet_url TEXT,
    cost_rating TEXT,  -- low | medium | high | very_high
    is_builtin BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX substrates_manufacturer_idx ON substrates(manufacturer);
CREATE INDEX substrates_type_idx ON substrates(material_type);

-- Array Configurations
CREATE TABLE array_configs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    lattice_type TEXT NOT NULL,  -- rectangular | triangular | circular | custom
    element_spacing JSONB NOT NULL,  -- {dx_lambda, dy_lambda} or custom positions
    num_elements INTEGER NOT NULL,
    excitation_weights JSONB NOT NULL,  -- [{amplitude, phase_deg}] per element
    beam_steering JSONB,  -- {theta_deg, phi_deg}
    taper_type TEXT,  -- uniform | taylor | chebyshev | custom
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX array_project_idx ON array_configs(project_id);

-- Usage Tracking (for billing enforcement)
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,  -- simulation_minutes | storage_bytes | cloud_compute
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
    pub antenna_type: String,
    pub topology_id: Option<String>,
    pub geometry_data: serde_json::Value,
    pub substrate_id: Option<Uuid>,
    pub settings: serde_json::Value,
    pub is_public: bool,
    pub forked_from: Option<Uuid>,
    pub thumbnail_url: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Simulation {
    pub id: Uuid,
    pub project_id: Uuid,
    pub user_id: Uuid,
    pub sim_type: String,
    pub status: String,
    pub execution_mode: String,
    pub frequency_points: Vec<f32>,
    pub num_unknowns: i32,
    pub mesh_data_url: Option<String>,
    pub results_url: Option<String>,
    pub results_summary: Option<serde_json::Value>,
    pub error_message: Option<String>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub wall_time_ms: Option<i32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct AntennaCatalog {
    pub id: Uuid,
    pub topology_id: String,
    pub name: String,
    pub category: String,
    pub polarization: Vec<String>,
    pub frequency_range_ghz: String,  // PostgreSQL range type
    pub gain_range_dbi: Option<String>,
    pub bandwidth_pct_range: Option<String>,
    pub size_wavelengths: serde_json::Value,
    pub description: Option<String>,
    pub geometry_generator: String,
    pub default_params: serde_json::Value,
    pub validation_data_url: Option<String>,
    pub is_verified: bool,
    pub tags: Vec<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Substrate {
    pub id: Uuid,
    pub name: String,
    pub manufacturer: Option<String>,
    pub material_type: String,
    pub permittivity_re: f32,
    pub permittivity_im: f32,
    pub loss_tangent: f32,
    pub thickness_mm: Option<f32>,
    pub frequency_dependent: bool,
    pub permittivity_data: Option<serde_json::Value>,
    pub datasheet_url: Option<String>,
    pub cost_rating: Option<String>,
    pub is_builtin: bool,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct SimulationParams {
    pub sim_type: SimulationType,
    pub frequency_points: Vec<f64>,  // GHz
    pub port_impedance: f64,  // Ohms, default 50
    pub far_field_distance: Option<f64>,  // Wavelengths, for pattern computation
    pub theta_points: u32,  // Angular resolution
    pub phi_points: u32,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum SimulationType {
    Impedance,      // S11, input impedance only
    Pattern,        // Far-field radiation pattern
    ArrayFactor,    // Array factor (isotropic elements)
    FullArray,      // Array with mutual coupling
}

#[derive(Debug, Deserialize, Serialize)]
pub struct AntennaSpecs {
    pub center_freq_ghz: f64,
    pub bandwidth_pct: Option<f64>,  // Fractional bandwidth
    pub target_gain_dbi: Option<f64>,
    pub polarization: String,  // linear_vertical | linear_horizontal | circular_left | circular_right
    pub max_size_mm: Option<(f64, f64, f64)>,  // (length, width, height)
    pub vswr_max: Option<f64>,  // Maximum acceptable VSWR, default 2.0
}
```

---

## Solver Architecture Deep-Dive

### Governing Equations and Discretization

AntennaSynth's core solver implements **Method of Moments (MoM)** with **RWG (Rao-Wilton-Glisson) basis functions**, the standard formulation used by antenna simulation tools for arbitrary 3D surface geometries. For a metallic antenna structure meshed into triangular patches with `N` edges, the MoM system is:

```
[Z] · [I] = [V]
```

Where:
- **[Z]** (N×N): Impedance matrix — mutual impedance between every pair of RWG basis functions
- **[I]** (N): Unknown current coefficients on each edge
- **[V]** (N): Excitation voltage vector (typically one or more delta-gap voltage sources)

**RWG basis function** on edge `n` between triangles T⁺ and T⁻:

```
f_n(r) = { l_n/(2A⁺) · ρ⁺  for r ∈ T⁺
         { l_n/(2A⁻) · ρ⁻  for r ∈ T⁻
         { 0               otherwise

where ρ⁺/⁻ is the vector from the free vertex to r
```

**Impedance matrix element** Z_mn via Electric Field Integral Equation (EFIE):

```
Z_mn = jωμ₀ ∬∬ f_m(r) · f_n(r') G(|r-r'|) dS dS'
       + (1/jωε₀) ∬∬ (∇ · f_m(r)) (∇ · f_n(r')) G(|r-r'|) dS dS'

where G(R) = e^(-jkR) / (4πR)  (free-space Green's function)
```

For **microstrip antennas on multilayer substrates**, we use the **multilayer planar Green's function** in the spectral domain to avoid meshing the substrate volume:

```
G_multilayer(r, r') = ∬ G̃(k_ρ, z, z') e^(jk_ρ · (ρ - ρ')) dk_ρ

where G̃(k_ρ, z, z') is computed via transmission-line network method
for layered media (Sommerfeld integral approach)
```

This reduces unknowns by 10-100x for typical patch antennas compared to volumetric methods.

**Far-field radiation pattern** is computed from the solved surface currents via:

```
E_far(r, θ, φ) = (jkη₀ e^(-jkr))/(4πr) ∬ [J(r') - r̂(r̂ · J(r'))] e^(jkr̂ · r') dS'

where J(r') = Σ I_n f_n(r')  (surface current from RWG coefficients)
```

**Gain** in direction (θ, φ):

```
G(θ, φ) = 4π |E(θ, φ)|² / ∬ |E(θ', φ')|² dΩ'
        = 4π |E(θ, φ)|² / P_rad  (normalized to total radiated power)
```

### Client/Server Split (WASM Threshold)

```
Antenna design uploaded → Mesh generated, unknowns counted
    │
    ├── ≤10K unknowns → WASM solver (browser)
    │   ├── Instant startup, no network latency
    │   ├── Results displayed immediately
    │   ├── Covers: patches, dipoles, monopoles, PIFAs, simple slots
    │   └── No server cost
    │
    └── >10K unknowns → Server solver (Rust native)
        ├── Job queued via Redis
        ├── Worker picks up, runs native solver with BLAS acceleration
        ├── Progress streamed via WebSocket
        ├── Covers: large arrays, mutual coupling, complex 3D geometries
        └── Results stored in S3, URL returned
```

The 10K-unknown threshold was chosen because:
- WASM solver with dense matrix solve handles 10K unknowns in <5 seconds on modern hardware
- 10K unknowns covers: single patches (200-500 unknowns), dipoles (50-100), PIFAs (300-800), simple arrays (<16 elements)
- Above 10K unknowns: large arrays (64+ elements), massive MIMO, horn/reflector antennas → need server compute

---

## Architecture Deep-Dives

### 1. Simulation API Handler (Rust/Axum)

The primary endpoint receives a simulation request, generates mesh from antenna geometry, decides between WASM and server execution, and for server execution enqueues a job with Redis and streams progress via WebSocket.

```rust
// src/api/handlers/simulation.rs

use axum::{
    extract::{Path, State, Json},
    http::StatusCode,
    response::IntoResponse,
};
use uuid::Uuid;
use crate::{
    db::models::{Simulation, SimulationParams, SimulationType},
    mesh::triangulate_antenna,
    state::AppState,
    auth::Claims,
    error::ApiError,
};

#[derive(serde::Deserialize)]
pub struct CreateSimulationRequest {
    pub sim_type: SimulationType,
    pub frequency_points: Vec<f64>,  // GHz
    pub port_impedance: f64,
    pub theta_points: u32,
    pub phi_points: u32,
}

pub async fn create_simulation(
    State(state): State<AppState>,
    claims: Claims,
    Path(project_id): Path<Uuid>,
    Json(req): Json<CreateSimulationRequest>,
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

    // 2. Generate mesh from antenna geometry
    let mesh = triangulate_antenna(&project.geometry_data)?;
    let num_unknowns = mesh.num_edges();

    // 3. Check plan limits
    let user = sqlx::query_as!(
        crate::db::models::User,
        "SELECT * FROM users WHERE id = $1",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    if user.plan == "free" && num_unknowns > 2000 {
        return Err(ApiError::PlanLimit(
            "Free plan supports antennas up to 2000 unknowns. Upgrade to Pro for unlimited."
        ));
    }

    // 4. Determine execution mode
    let execution_mode = if num_unknowns <= 10000 && user.plan != "free" {
        "wasm"
    } else {
        "server"
    };

    // 5. Upload mesh to S3 for server execution
    let mesh_url = if execution_mode == "server" {
        let mesh_data = serde_json::to_vec(&mesh)?;
        let s3_key = format!("meshes/{}/{}.json", project_id, uuid::Uuid::new_v4());
        state.s3.put_object()
            .bucket("antennasynth-data")
            .key(&s3_key)
            .body(mesh_data.into())
            .content_type("application/json")
            .send()
            .await?;
        Some(format!("s3://antennasynth-data/{}", s3_key))
    } else {
        None
    };

    // 6. Create simulation record
    let sim = sqlx::query_as!(
        Simulation,
        r#"INSERT INTO simulations
            (project_id, user_id, sim_type, status, execution_mode,
             frequency_points, num_unknowns, mesh_data_url)
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8)
        RETURNING *"#,
        project_id,
        claims.user_id,
        serde_json::to_string(&req.sim_type)?,
        if execution_mode == "wasm" { "ready" } else { "pending" },
        execution_mode,
        &req.frequency_points as &[f64],
        num_unknowns as i32,
        mesh_url,
    )
    .fetch_one(&state.db)
    .await?;

    // 7. For server execution, enqueue job
    if execution_mode == "server" {
        let job = sqlx::query_as!(
            crate::db::models::SimulationJob,
            r#"INSERT INTO simulation_jobs (simulation_id, priority)
            VALUES ($1, $2) RETURNING *"#,
            sim.id,
            if user.plan == "advanced" { 10 } else { 0 },
        )
        .fetch_one(&state.db)
        .await?;

        // Publish to Redis job queue
        state.redis
            .publish("simulation:jobs", serde_json::to_string(&job.id)?)
            .await?;
    }

    Ok((StatusCode::CREATED, Json(sim)))
}

pub async fn get_simulation(
    State(state): State<AppState>,
    claims: Claims,
    Path((project_id, sim_id)): Path<(Uuid, Uuid)>,
) -> Result<Json<Simulation>, ApiError> {
    let sim = sqlx::query_as!(
        Simulation,
        "SELECT s.* FROM simulations s
         JOIN projects p ON s.project_id = p.id
         WHERE s.id = $1 AND s.project_id = $2
         AND (p.owner_id = $3 OR p.org_id IN (
             SELECT org_id FROM org_members WHERE user_id = $3
         ))",
        sim_id,
        project_id,
        claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Simulation not found"))?;

    Ok(Json(sim))
}
```

### 2. MoM Solver Core (Rust — shared between WASM and native)

The core MoM solver that builds and solves the impedance matrix. This code compiles to both native (server) and WASM (browser) targets.

```rust
// solver-core/src/mom.rs

use ndarray::{Array1, Array2};
use num_complex::Complex64;
use crate::mesh::{TriangularMesh, RwgBasis};
use crate::green::free_space_green;

pub struct MomSystem {
    pub num_edges: usize,
    pub z_matrix: Array2<Complex64>,  // Impedance matrix
    pub v_vector: Array1<Complex64>,  // Excitation vector
    pub i_vector: Array1<Complex64>,  // Current coefficients (solution)
    pub frequency_ghz: f64,
}

impl MomSystem {
    pub fn new(num_edges: usize, frequency_ghz: f64) -> Self {
        Self {
            num_edges,
            z_matrix: Array2::zeros((num_edges, num_edges)),
            v_vector: Array1::zeros(num_edges),
            i_vector: Array1::zeros(num_edges),
            frequency_ghz,
        }
    }

    /// Fill impedance matrix via EFIE with RWG basis functions
    pub fn fill_impedance_matrix(
        &mut self,
        mesh: &TriangularMesh,
        rwg_basis: &[RwgBasis],
    ) -> Result<(), SolverError> {
        let k0 = 2.0 * std::f64::consts::PI * self.frequency_ghz / 0.2998;  // Free-space wavenumber (GHz → m⁻¹)
        let eta0 = 376.73;  // Free-space impedance (Ω)

        for m in 0..self.num_edges {
            for n in 0..self.num_edges {
                let z_mn = self.compute_z_element(
                    &rwg_basis[m],
                    &rwg_basis[n],
                    mesh,
                    k0,
                    eta0,
                )?;
                self.z_matrix[[m, n]] = z_mn;
            }

            // Progress reporting for large matrices
            if m % 100 == 0 {
                tracing::debug!("Filled impedance row {}/{}", m, self.num_edges);
            }
        }

        Ok(())
    }

    /// Compute single impedance matrix element Z_mn
    fn compute_z_element(
        &self,
        basis_m: &RwgBasis,
        basis_n: &RwgBasis,
        mesh: &TriangularMesh,
        k0: f64,
        eta0: f64,
    ) -> Result<Complex64, SolverError> {
        // 4-point Gaussian quadrature on each triangle pair
        let gauss_weights = [0.25, 0.25, 0.25, 0.25];
        let gauss_points = [
            (0.333333, 0.333333),
            (0.6, 0.2),
            (0.2, 0.6),
            (0.2, 0.2),
        ];

        let omega = 2.0 * std::f64::consts::PI * self.frequency_ghz * 1e9;
        let mu0 = 4.0 * std::f64::consts::PI * 1e-7;
        let eps0 = 8.854e-12;

        let tri_m_plus = mesh.triangles[basis_m.tri_plus_idx];
        let tri_m_minus = mesh.triangles[basis_m.tri_minus_idx];
        let tri_n_plus = mesh.triangles[basis_n.tri_plus_idx];
        let tri_n_minus = mesh.triangles[basis_n.tri_minus_idx];

        let mut z_mn = Complex64::new(0.0, 0.0);

        // Integrate over all four triangle combinations
        for (tri_m, area_m, sign_m) in [
            (&tri_m_plus, basis_m.area_plus, 1.0),
            (&tri_m_minus, basis_m.area_minus, -1.0),
        ] {
            for (tri_n, area_n, sign_n) in [
                (&tri_n_plus, basis_n.area_plus, 1.0),
                (&tri_n_minus, basis_n.area_minus, -1.0),
            ] {
                for &(u_m, v_m) in &gauss_points {
                    let r_m = mesh.barycentric_to_cartesian(tri_m, u_m, v_m);
                    let rho_m = basis_m.rho_vector(tri_m, r_m, sign_m > 0.0);

                    for &(u_n, v_n) in &gauss_points {
                        let r_n = mesh.barycentric_to_cartesian(tri_n, u_n, v_n);
                        let rho_n = basis_n.rho_vector(tri_n, r_n, sign_n > 0.0);

                        let dist = (r_m - r_n).norm();
                        let green = free_space_green(dist, k0);

                        // Vector potential term: jωμ₀ ∫∫ f_m · f_n G dS dS'
                        let dot_product = rho_m.dot(&rho_n);
                        let coeff_mn = basis_m.edge_length / (2.0 * area_m)
                                     * basis_n.edge_length / (2.0 * area_n)
                                     * sign_m * sign_n;

                        z_mn += Complex64::new(0.0, omega * mu0)
                              * coeff_mn * dot_product * green
                              * area_m * area_n * 0.0625;  // Quadrature weight

                        // Scalar potential term: (1/jωε₀) ∫∫ (∇·f_m)(∇·f_n) G dS dS'
                        let div_m = sign_m / area_m;
                        let div_n = sign_n / area_n;

                        z_mn += Complex64::new(0.0, -1.0 / (omega * eps0))
                              * div_m * div_n * green
                              * area_m * area_n * 0.0625;
                    }
                }
            }
        }

        Ok(z_mn)
    }

    /// Set excitation at a delta-gap voltage source
    pub fn set_excitation(&mut self, edge_idx: usize, voltage: Complex64) {
        self.v_vector[edge_idx] = voltage;
    }

    /// Solve the system [Z][I] = [V] using LU decomposition
    pub fn solve(&mut self) -> Result<(), SolverError> {
        use ndarray_linalg::Solve;

        self.i_vector = self.z_matrix.solve(&self.v_vector)
            .map_err(|_| SolverError::Singular("Impedance matrix is singular".into()))?;

        Ok(())
    }

    /// Compute input impedance at feed edge
    pub fn input_impedance(&self, feed_edge: usize) -> Complex64 {
        self.v_vector[feed_edge] / self.i_vector[feed_edge]
    }

    /// Compute S11 (reflection coefficient)
    pub fn s11(&self, feed_edge: usize, z0: f64) -> Complex64 {
        let z_in = self.input_impedance(feed_edge);
        (z_in - z0) / (z_in + z0)
    }

    /// Compute far-field pattern in direction (theta, phi)
    pub fn far_field(
        &self,
        theta_deg: f64,
        phi_deg: f64,
        mesh: &TriangularMesh,
        rwg_basis: &[RwgBasis],
    ) -> Complex64 {
        let k0 = 2.0 * std::f64::consts::PI * self.frequency_ghz / 0.2998;
        let eta0 = 376.73;

        let theta = theta_deg.to_radians();
        let phi = phi_deg.to_radians();

        let r_hat = [
            theta.sin() * phi.cos(),
            theta.sin() * phi.sin(),
            theta.cos(),
        ];

        let mut e_field = Complex64::new(0.0, 0.0);

        for (n, basis) in rwg_basis.iter().enumerate() {
            let current = self.i_vector[n];

            // Integrate current contribution from this basis function
            for (tri, area, sign) in [
                (&mesh.triangles[basis.tri_plus_idx], basis.area_plus, 1.0),
                (&mesh.triangles[basis.tri_minus_idx], basis.area_minus, -1.0),
            ] {
                let centroid = mesh.triangle_centroid(tri);
                let rho = basis.rho_vector(tri, centroid, sign > 0.0);

                let phase = Complex64::new(0.0, k0 * centroid.dot(&r_hat));
                let radial_component = r_hat.dot(&rho);

                e_field += current * (rho - radial_component * r_hat) * phase.exp() * area * sign;
            }
        }

        e_field *= Complex64::new(0.0, k0 * eta0) / (4.0 * std::f64::consts::PI);
        e_field
    }
}

#[derive(Debug)]
pub enum SolverError {
    Singular(String),
    Convergence(String),
    InvalidMesh(String),
}
```

### 3. Antenna Synthesis Engine (Python FastAPI)

The synthesis engine recommends antenna topologies and computes initial dimensions from user specifications.

```python
# synthesis-service/main.py

from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Optional, List
import numpy as np
from scipy.optimize import differential_evolution

app = FastAPI()

class AntennaSpecs(BaseModel):
    center_freq_ghz: float
    bandwidth_pct: Optional[float] = 10.0
    target_gain_dbi: Optional[float] = None
    polarization: str = "linear_vertical"
    max_size_mm: Optional[tuple[float, float, float]] = None
    vswr_max: float = 2.0

class SynthesisResult(BaseModel):
    topology_id: str
    name: str
    initial_dimensions: dict
    estimated_performance: dict
    confidence_score: float

@app.post("/synthesize", response_model=List[SynthesisResult])
async def synthesize_antenna(specs: AntennaSpecs):
    """
    Recommend antenna topologies and compute initial dimensions
    based on user specifications.
    """
    wavelength_mm = 299.8 / specs.center_freq_ghz  # c/f in mm

    results = []

    # Rectangular microstrip patch antenna
    if specs.max_size_mm is None or (
        specs.max_size_mm[0] > wavelength_mm/2 and
        specs.max_size_mm[1] > wavelength_mm/2
    ):
        patch_result = synthesize_rectangular_patch(specs, wavelength_mm)
        results.append(patch_result)

    # Half-wave dipole
    dipole_result = synthesize_dipole(specs, wavelength_mm)
    results.append(dipole_result)

    # PIFA (Planar Inverted-F Antenna) for compact designs
    if specs.max_size_mm and min(specs.max_size_mm[:2]) < wavelength_mm/4:
        pifa_result = synthesize_pifa(specs, wavelength_mm)
        results.append(pifa_result)

    # Sort by confidence score
    results.sort(key=lambda x: x.confidence_score, reverse=True)
    return results[:5]  # Return top 5 candidates

def synthesize_rectangular_patch(specs: AntennaSpecs, wavelength_mm: float) -> SynthesisResult:
    """
    Synthesize rectangular microstrip patch antenna dimensions.
    Uses cavity model for initial estimate.
    """
    # Assume Rogers RO4003C substrate: εr=3.38, h=1.524mm
    eps_r = 3.38
    h_mm = 1.524

    # Effective permittivity
    eps_eff = (eps_r + 1) / 2 + (eps_r - 1) / 2 * (1 + 12 * h_mm / wavelength_mm)**(-0.5)

    # Patch length (resonant dimension)
    length_mm = wavelength_mm / (2 * np.sqrt(eps_eff))

    # Fringing field extension
    delta_l = 0.412 * h_mm * (eps_eff + 0.3) * (wavelength_mm / h_mm + 0.264) / \
              ((eps_eff - 0.258) * (wavelength_mm / h_mm + 0.8))

    length_mm -= 2 * delta_l

    # Patch width (typically 1.5x length for better bandwidth)
    width_mm = 1.5 * length_mm

    # Estimated gain (cavity model)
    area_lambda2 = (length_mm * width_mm) / (wavelength_mm**2)
    efficiency = 0.9  # Typical for microstrip
    gain_dbi = 10 * np.log10(4 * np.pi * area_lambda2 * efficiency)

    # Estimated bandwidth (empirical formula)
    bandwidth_pct = 3.77 * (eps_r - 1) / eps_r**2 * (h_mm / wavelength_mm)

    confidence = 0.9 if specs.polarization == "linear_vertical" else 0.7

    return SynthesisResult(
        topology_id="rectangular_patch",
        name="Rectangular Microstrip Patch",
        initial_dimensions={
            "length_mm": round(length_mm, 2),
            "width_mm": round(width_mm, 2),
            "substrate_height_mm": h_mm,
            "substrate_permittivity": eps_r,
            "feed_position_mm": round(length_mm * 0.35, 2),  # Inset feed for 50Ω
        },
        estimated_performance={
            "gain_dbi": round(gain_dbi, 1),
            "bandwidth_pct": round(bandwidth_pct * 100, 1),
            "vswr": 1.5,
            "efficiency": 0.9,
        },
        confidence_score=confidence,
    )

def synthesize_dipole(specs: AntennaSpecs, wavelength_mm: float) -> SynthesisResult:
    """
    Synthesize half-wave dipole dimensions.
    """
    # Half-wave dipole length (slightly shorter due to end effects)
    length_mm = 0.47 * wavelength_mm

    # Wire radius (assume AWG 14: 1.63mm diameter)
    radius_mm = 0.815

    # Theoretical gain
    gain_dbi = 2.15

    # Bandwidth (2:1 VSWR)
    bandwidth_pct = 10.0

    confidence = 0.95 if specs.polarization.startswith("linear") else 0.5

    return SynthesisResult(
        topology_id="half_wave_dipole",
        name="Half-Wave Dipole",
        initial_dimensions={
            "length_mm": round(length_mm, 2),
            "wire_radius_mm": radius_mm,
            "feed_gap_mm": 2.0,
        },
        estimated_performance={
            "gain_dbi": gain_dbi,
            "bandwidth_pct": bandwidth_pct,
            "vswr": 1.2,
            "efficiency": 0.95,
            "input_impedance_ohm": 73,
        },
        confidence_score=confidence,
    )

def synthesize_pifa(specs: AntennaSpecs, wavelength_mm: float) -> SynthesisResult:
    """
    Synthesize PIFA (Planar Inverted-F Antenna) for compact designs.
    """
    # PIFA is typically λ/4 resonant with ground plane
    length_mm = wavelength_mm / 4
    width_mm = length_mm * 0.6
    height_mm = wavelength_mm / 20  # Typical height: λ/20

    # Shorting pin position
    short_pos_mm = length_mm * 0.1

    # Feed position for 50Ω match
    feed_pos_mm = length_mm * 0.25

    # Estimated gain (reduced due to ground plane proximity)
    gain_dbi = 1.5

    # Bandwidth
    bandwidth_pct = 5.0

    confidence = 0.8

    return SynthesisResult(
        topology_id="pifa_planar",
        name="Planar Inverted-F Antenna (PIFA)",
        initial_dimensions={
            "length_mm": round(length_mm, 2),
            "width_mm": round(width_mm, 2),
            "height_mm": round(height_mm, 2),
            "short_position_mm": round(short_pos_mm, 2),
            "feed_position_mm": round(feed_pos_mm, 2),
        },
        estimated_performance={
            "gain_dbi": gain_dbi,
            "bandwidth_pct": bandwidth_pct,
            "vswr": 1.8,
            "efficiency": 0.7,
        },
        confidence_score=confidence,
    )
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–5)

**Day 1: Project initialization and Rust backend scaffold**
```bash
cargo init antennasynth-api
cd antennasynth-api
cargo add axum tokio serde serde_json sqlx uuid chrono tower-http jsonwebtoken bcrypt
cargo add --features postgres,runtime-tokio-rustls,uuid,chrono,json sqlx
cargo add aws-sdk-s3 redis ndarray num-complex
```
- `src/main.rs` — Axum app with CORS, tracing, graceful shutdown
- `src/config.rs` — Environment-based configuration (DATABASE_URL, JWT_SECRET, S3_BUCKET, REDIS_URL)
- `src/state.rs` — AppState struct (PgPool, Redis, S3Client)
- `src/error.rs` — ApiError enum with IntoResponse
- `Dockerfile` — Multi-stage build (cargo-chef for layer caching)
- `docker-compose.yml` — PostgreSQL, Redis, MinIO (S3-compatible)

**Day 2: Database schema and migrations**
- `migrations/001_initial.sql` — All 8 tables: users, organizations, org_members, projects, simulations, simulation_jobs, antenna_catalog, substrates, array_configs, usage_records
- `src/db/mod.rs` — Database pool initialization
- `src/db/models.rs` — All SQLx structs with FromRow derives
- Run `sqlx migrate run` to apply schema
- Seed script for antenna catalog (30 topologies with metadata)
- Seed script for substrate library (Rogers, Taconic, Isola materials)

**Day 3: Authentication system**
- `src/auth/mod.rs` — JWT token generation and validation middleware
- `src/auth/oauth.rs` — Google and GitHub OAuth 2.0 flow handlers
- `src/api/handlers/auth.rs` — Register, login, OAuth callback, refresh token, me endpoints
- Password hashing with bcrypt (cost factor 12)
- JWT with 24h access token + 30d refresh token
- Auth middleware that extracts Claims from Authorization header

**Day 4: User and project CRUD**
- `src/api/handlers/users.rs` — Get profile, update profile, delete account
- `src/api/handlers/projects.rs` — Create, list, get, update, delete, fork project
- `src/api/handlers/orgs.rs` — Create org, invite member, list members, remove member
- `src/api/router.rs` — All route definitions with auth middleware
- Integration tests: auth flow, project CRUD, authorization checks

**Day 5: Antenna catalog and substrate library APIs**
- `src/api/handlers/catalog.rs` — Search antenna topologies (parametric filter: frequency, gain, bandwidth, polarization, size), get topology details
- `src/api/handlers/substrates.rs` — List substrates, search by manufacturer/type, get substrate details
- Full-text search on antenna names and tags using pg_trgm
- Parametric filters with PostgreSQL range queries
- Pagination with cursor-based scrolling

### Phase 2 — Solver Core Prototype (Days 6–14)

**Day 6: Mesh generation framework**
- `solver-core/` — New Rust workspace member (shared between native and WASM)
- `solver-core/src/mesh.rs` — TriangularMesh struct with vertices, triangles, edges
- `solver-core/src/mesh/triangulate.rs` — Delaunay triangulation for planar geometries
- `solver-core/src/mesh/rwg.rs` — RWG basis function generation from mesh
- Unit tests: simple square patch mesh, edge extraction, RWG basis validation

**Day 7: Antenna geometry generators**
- `solver-core/src/geometries/rectangular_patch.rs` — Generate rectangular patch mesh from dimensions
- `solver-core/src/geometries/circular_patch.rs` — Generate circular patch mesh
- `solver-core/src/geometries/dipole.rs` — Generate thin-wire dipole mesh
- `solver-core/src/geometries/monopole.rs` — Generate monopole with ground plane
- `solver-core/src/geometries/pifa.rs` — Generate PIFA geometry with shorting pin
- Tests: mesh quality metrics (aspect ratio, edge length distribution)

**Day 8: Green's function and impedance matrix assembly**
- `solver-core/src/green.rs` — Free-space Green's function G(R) = exp(-jkR)/(4πR)
- `solver-core/src/green/singularity.rs` — Singularity extraction for self-impedance terms
- `solver-core/src/mom.rs` — MomSystem struct with impedance matrix fill
- Gaussian quadrature for surface integrals (4-point rule)
- Tests: impedance matrix symmetry, reciprocity verification

**Day 9: Linear system solver integration**
- `solver-core/src/linalg.rs` — LU decomposition wrapper via `ndarray-linalg`
- Dense matrix solver for small systems (<5K unknowns)
- Iterative solver (GMRES) for larger systems (5K-20K unknowns)
- Preconditioner: diagonal scaling
- Tests: solve simple dipole (73Ω input impedance), convergence monitoring

**Day 10: S-parameter and impedance computation**
- `solver-core/src/sparameters.rs` — S11 computation from input impedance
- `solver-core/src/impedance.rs` — Input impedance calculation at feed edge
- VSWR calculation from S11
- Smith chart data generation (real/imaginary impedance, normalized)
- Tests: half-wave dipole (verify 73Ω), quarter-wave monopole (verify 36.5Ω)

**Day 11: Far-field radiation pattern computation**
- `solver-core/src/farfield.rs` — Far-field integration from surface currents
- `solver-core/src/farfield/pattern.rs` — Radiation pattern sampling (θ, φ grid)
- Gain computation: normalize to total radiated power
- Directivity and efficiency extraction
- Beamwidth calculation (half-power points)
- Tests: dipole pattern (omnidirectional in H-plane, figure-8 in E-plane)

**Day 12: Multilayer Green's function for microstrip**
- `solver-core/src/green/multilayer.rs` — Transmission-line method for layered media
- Sommerfeld integral evaluation (numerical integration with adaptive quadrature)
- Spectral-domain Green's function caching
- Tests: compare microstrip patch results to cavity model

**Day 13: Frequency sweep and multi-frequency analysis**
- `solver-core/src/analysis/sweep.rs` — Frequency sweep loop with LU factorization reuse
- `solver-core/src/analysis/bandwidth.rs` — Bandwidth extraction from S11 < -10dB
- Resonant frequency detection from minimum |S11|
- Tests: patch antenna resonance sweep, bandwidth verification

**Day 14: Solver validation with canonical antennas**
- Benchmark 1: Half-wave dipole — verify impedance 73+j42.5Ω, gain 2.15 dBi
- Benchmark 2: Quarter-wave monopole — verify impedance 36.5Ω, gain 5.15 dBi (over ground)
- Benchmark 3: Rectangular patch — verify resonance frequency within 2%, gain within 1 dB
- Automated test suite: `solver-core/tests/canonical.rs` with assertions

### Phase 3 — WASM Build + Frontend Visualization (Days 15–21)

**Day 15: WASM solver compilation**
- `solver-wasm/` — New workspace member for WASM target
- `solver-wasm/Cargo.toml` — wasm-bindgen, serde-wasm-bindgen dependencies
- `solver-wasm/src/lib.rs` — WASM entry points: `solve_impedance()`, `solve_pattern()`, `solve_array_factor()`
- `wasm-pack build --target web --release` pipeline
- `wasm-opt -Oz` for size optimization (target <3MB gzipped)
- JavaScript wrapper: `AntennaSolver` class that loads WASM and provides async API

**Day 16: Frontend scaffold and Three.js setup**
```bash
npm create vite@latest frontend -- --template react-ts
cd frontend
npm i zustand @tanstack/react-query axios three @react-three/fiber @react-three/drei d3
```
- `src/App.tsx` — Router, auth context, layout
- `src/stores/projectStore.ts` — Zustand store for project state
- `src/stores/antennaStore.ts` — Antenna geometry and simulation state
- `src/components/AntennaViewer3D.tsx` — Three.js canvas with orbit controls
- `src/lib/wasmLoader.ts` — WASM bundle loading utility

**Day 17: 3D antenna geometry visualization**
- `src/components/AntennaViewer3D/GeometryRenderer.tsx` — Render mesh as BufferGeometry
- `src/components/AntennaViewer3D/FeedPoint.tsx` — Visualize feed location as cone marker
- `src/components/AntennaViewer3D/Substrate.tsx` — Render substrate layer as transparent box
- `src/components/AntennaViewer3D/GroundPlane.tsx` — Render ground plane as metallic plane
- Material shaders: metallic for conductor, dielectric for substrate
- Lighting: ambient + directional for realistic appearance

**Day 18: Radiation pattern 3D visualization**
- `src/components/PatternViewer3D.tsx` — Radiation pattern as surface plot
- `src/components/PatternViewer3D/PatternSurface.tsx` — Generate mesh from (θ, φ, gain) data
- Color mapping: gain magnitude → colormap (blue=low, red=high)
- Transparent clipping planes for cross-section views (E-plane, H-plane)
- Pattern normalization: absolute gain vs. normalized to max

**Day 19: Smith chart and S-parameter plotting**
- `src/components/SmithChart.tsx` — D3.js Smith chart with impedance circles
- `src/components/SmithChart/ImpedancePlot.tsx` — Plot impedance locus vs. frequency
- `src/components/SParameterPlot.tsx` — S11 magnitude (dB) and phase vs. frequency
- Marker annotations: resonant frequency, -10dB bandwidth
- Interactive cursors for reading values

**Day 20: Antenna parameter editor**
- `src/components/AntennaEditor/TopologySelector.tsx` — Select antenna type from catalog
- `src/components/AntennaEditor/DimensionInputs.tsx` — Edit dimensions with unit conversion (mm, wavelengths)
- `src/components/AntennaEditor/SubstrateSelector.tsx` — Select substrate material from library
- `src/components/AntennaEditor/FeedConfig.tsx` — Configure feed position and impedance
- Real-time validation: size constraints, frequency range warnings

**Day 21: Simulation control panel**
- `src/components/SimulationPanel.tsx` — Simulation type selector (Impedance, Pattern, Array Factor)
- `src/components/SimulationPanel/FrequencyInput.tsx` — Frequency range input (start, stop, points)
- `src/components/SimulationPanel/RunButton.tsx` — Trigger simulation (WASM or server)
- Progress indicator: spinner for WASM, WebSocket progress bar for server
- `src/hooks/useWasmSolver.ts` — React hook for WASM solver execution

### Phase 4 — API + Job Orchestration (Days 22–28)

**Day 22: Simulation API endpoints**
- `src/api/handlers/simulation.rs` — Create simulation, get simulation, list simulations, cancel simulation
- Mesh generation from antenna geometry → S3 upload for server execution
- Unknown count extraction and WASM/server routing logic
- Plan-based limits enforcement (free: 2000 unknowns, 10 cloud minutes/month)

**Day 23: Server-side simulation worker**
- `src/workers/simulation_worker.rs` — Redis job consumer, runs native solver
- `src/workers/mod.rs` — Worker pool management (configurable concurrency)
- Progress streaming: worker publishes progress → Redis pub/sub → WebSocket to client
- Error handling: matrix singularity, out-of-memory, timeout
- S3 result upload with presigned URL generation for client download

**Day 24: WebSocket for live simulation progress**
- `src/api/ws/mod.rs` — Axum WebSocket upgrade handler
- `src/api/ws/simulation_progress.rs` — Subscribe to simulation progress channel
- Client receives: `{ progress_pct, current_freq_ghz, iteration, residual }`
- Connection management: heartbeat ping/pong, automatic cleanup on disconnect
- Frontend: `src/hooks/useSimulationProgress.ts` — React hook for WebSocket subscription

**Day 25: Array factor analysis engine**
- `solver-core/src/array/mod.rs` — Array factor computation for isotropic elements
- `solver-core/src/array/lattice.rs` — Rectangular and triangular lattice generation
- `solver-core/src/array/taper.rs` — Amplitude taper (uniform, Taylor, Chebyshev)
- `solver-core/src/array/steering.rs` — Beam steering via phase shift
- Tests: broadside array, endfire array, grating lobe detection

**Day 26: Array visualization UI**
- `src/components/ArrayEditor.tsx` — Array layout editor (drag elements, set spacing)
- `src/components/ArrayEditor/LatticeSelector.tsx` — Select rectangular/triangular lattice
- `src/components/ArrayEditor/ElementSpacing.tsx` — Adjust spacing in wavelengths
- `src/components/ArrayEditor/ExcitationWeights.tsx` — Edit amplitude/phase per element
- `src/components/ArrayViewer3D.tsx` — 3D visualization of array layout

**Day 27: Array pattern visualization**
- `src/components/ArrayPattern2D.tsx` — Polar plot of array factor (azimuth/elevation cuts)
- `src/components/ArrayPattern2D/PolarPlot.tsx` — D3.js polar coordinate system
- Grating lobe indicators: red markers for grating lobe positions
- Beamwidth and sidelobe level annotations
- Real-time update as spacing/weights change

**Day 28: Project management features**
- `src/api/handlers/projects.rs` — Fork project (deep copy), share via public link, project templates
- `src/api/handlers/export.rs` — Export geometry as STL, export results as CSV/JSON
- Project thumbnail generation: server-side Three.js headless rendering
- Auto-save: frontend debounced PATCH every 5 seconds on geometry changes

### Phase 5 — Results + Post-Processing UI (Days 29–33)

**Day 29: Simulation result processing**
- Result parsing: binary format (Complex64 arrays) → structured data (frequencies, S11, impedance, patterns)
- Result caching: hot results in Redis (TTL 1 hour), cold results in S3
- Presigned S3 URLs for client-side download of large result sets
- Result summary generation: auto-detect resonance, bandwidth, peak gain

**Day 30: Impedance matching tool**
- `src/components/MatchingNetworkTool.tsx` — L/Pi/T network synthesis
- `src/components/MatchingNetworkTool/SmithChartDesign.tsx` — Interactive Smith chart for matching
- `src/components/MatchingNetworkTool/NetworkCalculator.tsx` — Component value calculation (L, C)
- Matching quality metrics: VSWR improvement, bandwidth impact
- Export matching network as schematic

**Day 31: Pattern analysis and measurements**
- `src/components/PatternAnalysis.tsx` — Automated pattern measurements panel
- Beamwidth calculation (half-power beamwidth in E-plane and H-plane)
- Sidelobe level detection (first sidelobe, average sidelobe level)
- Front-to-back ratio
- Cross-polarization discrimination
- Gain vs. angle table export

**Day 32: Substrate material database UI**
- `src/pages/Substrates.tsx` — Substrate library browser with search and filter
- `src/components/SubstrateCard.tsx` — Display substrate properties (εr, tan δ, thickness)
- `src/components/SubstrateComparison.tsx` — Compare multiple substrates side-by-side
- Frequency-dependent permittivity plot for dispersive materials
- Add custom substrate (Team plan)

**Day 33: Antenna catalog templates**
- `src/pages/Templates.tsx` — Template gallery with preview thumbnails
- Pre-built templates:
  - Rectangular patch (2.4 GHz WiFi)
  - Circular patch (5.8 GHz)
  - Half-wave dipole (900 MHz)
  - Quarter-wave monopole (2.4 GHz)
  - PIFA (compact IoT)
  - 2×2 patch array (MIMO)
  - 4×4 patch array (beamforming)
- Template instantiation: one-click "Use Template" creates new project

### Phase 6 — Billing + Plan Enforcement (Days 34–37)

**Day 34: Stripe integration**
- `src/api/handlers/billing.rs` — Create checkout session, create customer portal session, get subscription status
- `src/api/handlers/webhooks/stripe.rs` — Handle subscription events: `checkout.session.completed`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.payment_failed`
- Plan mapping:
  - Free: 2000-unknown antennas, 10 cloud simulation minutes/month, basic catalog (10 topologies)
  - Pro ($179/mo): Unlimited unknowns, 1500 cloud minutes/month, full catalog (30 topologies), pattern export
  - Advanced ($399/mo): Massive arrays, unlimited cloud compute, beamforming, MIMO, API access

**Day 35: Usage tracking and plan limits**
- `src/middleware/plan_limits.rs` — Middleware checking plan limits before simulation execution
- `src/services/usage.rs` — Track cloud simulation minutes per billing period
- Usage record insertion after each server-side simulation completes
- Usage dashboard: `src/api/handlers/usage.rs` — Current period usage, historical usage
- Approaching-limit warnings at 80% and 100% of cloud minutes

**Day 36: Billing UI**
- `frontend/src/pages/Billing.tsx` — Plan comparison table, current plan display, usage meter
- `frontend/src/components/billing/PlanCard.tsx` — Plan feature comparison cards
- `frontend/src/components/billing/UsageMeter.tsx` — Cloud minutes usage bar with threshold indicators
- Upgrade/downgrade flow via Stripe Customer Portal
- Upgrade prompt modals when hitting plan limits (e.g., "Antenna has 15K unknowns. Upgrade to Pro for unlimited.")

**Day 37: Feature gating**
- Gate pattern export (CSV/STL) behind Pro plan
- Gate array analysis (>16 elements) behind Advanced plan
- Gate beamforming optimization behind Advanced plan
- Gate API access behind Advanced plan
- Show locked feature indicators with upgrade CTAs in UI
- Admin endpoint: override plan for specific users (beta testers, partners)

### Phase 7 — Solver Validation + Testing (Days 38–40)

**Day 38: Solver validation — canonical antennas**
- Benchmark 1: Half-wave dipole at 300 MHz — verify Z_in = 73+j42.5Ω, gain = 2.15 dBi ± 0.1 dB
- Benchmark 2: Quarter-wave monopole at 900 MHz — verify Z_in = 36.5Ω, gain = 5.15 dBi ± 0.1 dB
- Benchmark 3: Rectangular patch at 2.4 GHz (Rogers RO4003C) — verify resonance within ±2%, gain within ±1 dB
- Benchmark 4: Circular patch at 5.8 GHz — verify bandwidth within ±10%, beamwidth within ±5°
- Automated test suite: `solver-core/tests/validation.rs` with assertions

**Day 39: Solver validation — arrays**
- Benchmark 5: Broadside array 1×4 λ/2 spacing — verify beam peak at θ=0°, beamwidth within ±5%
- Benchmark 6: Endfire array 1×4 λ/4 spacing — verify beam peak at θ=90°, gain within ±1 dB
- Benchmark 7: 2×2 patch array with mutual coupling — verify scan impedance variation, grating lobes
- Benchmark 8: Taylor-tapered 1×16 array — verify -25dB sidelobe level ± 2 dB
- Compare results against HFSS/CST reference simulations for same geometries

**Day 40: Integration and performance testing**
- End-to-end test: create project → select topology → edit dimensions → run simulation → view pattern → export results
- API integration tests: auth → project CRUD → simulation → results
- WASM solver test: load in headless browser (Playwright), run simulation, verify results match native solver
- WebSocket test: connect → subscribe → receive progress → receive completion
- Performance benchmarks: WASM solver 5K unknowns < 3s, server solver 20K unknowns < 30s
- Frontend rendering: pattern viewer 60 FPS with 10K pattern points

### Phase 8 — Deployment + Polish + Launch (Days 41–42)

**Day 41: Docker and deployment configuration**
- `Dockerfile` — Multi-stage Rust build (cargo-chef for layer caching)
- `docker-compose.yml` — Full local dev stack (API, PostgreSQL, Redis, MinIO, Python synthesis service, frontend)
- `k8s/` — Kubernetes manifests:
  - `api-deployment.yaml` — API server (3 replicas, HPA)
  - `worker-deployment.yaml` — Simulation workers (auto-scaling based on queue depth)
  - `synthesis-deployment.yaml` — Python FastAPI synthesis service
  - `postgres-statefulset.yaml` — PostgreSQL with PVC
  - `redis-deployment.yaml` — Redis for job queue and pub/sub
  - `ingress.yaml` — NGINX ingress with TLS
- Health check endpoints: `/health/live`, `/health/ready`
- CDN setup: CloudFront for WASM bundle and static assets

**Day 42: Monitoring, polish, and launch**
- Prometheus metrics: simulation duration histogram, solver convergence rate, API latency percentiles
- Grafana dashboards: system health, simulation throughput, user activity
- Sentry integration: frontend and backend error tracking with source maps
- Structured logging (tracing + JSON) for all API requests and solver events
- UI polish: loading states, error messages, empty states, responsive layout
- Accessibility: keyboard navigation, ARIA labels on interactive elements
- Landing page with product overview, pricing, and signup
- Documentation: getting started guide, antenna design tutorial, API reference
- Deploy to production, enable monitoring alerts, announce launch

---

## Critical Files

```
antennasynth/
├── solver-core/                           # Shared solver library (native + WASM)
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── mesh.rs                        # Triangular mesh data structure
│   │   ├── mesh/
│   │   │   ├── triangulate.rs             # Delaunay triangulation
│   │   │   └── rwg.rs                     # RWG basis function generation
│   │   ├── geometries/
│   │   │   ├── mod.rs                     # Geometry trait
│   │   │   ├── rectangular_patch.rs
│   │   │   ├── circular_patch.rs
│   │   │   ├── dipole.rs
│   │   │   ├── monopole.rs
│   │   │   ├── pifa.rs
│   │   │   ├── horn.rs
│   │   │   ├── helix.rs
│   │   │   ├── vivaldi.rs
│   │   │   └── slot.rs
│   │   ├── green.rs                       # Free-space Green's function
│   │   ├── green/
│   │   │   ├── singularity.rs             # Singularity extraction
│   │   │   └── multilayer.rs              # Multilayer planar Green's function
│   │   ├── mom.rs                         # MoM system assembly and solve
│   │   ├── linalg.rs                      # Linear system solvers (LU, GMRES)
│   │   ├── sparameters.rs                 # S-parameter computation
│   │   ├── impedance.rs                   # Input impedance calculation
│   │   ├── farfield.rs                    # Far-field pattern computation
│   │   ├── farfield/
│   │   │   └── pattern.rs                 # Radiation pattern sampling
│   │   ├── analysis/
│   │   │   ├── sweep.rs                   # Frequency sweep
│   │   │   └── bandwidth.rs               # Bandwidth extraction
│   │   └── array/
│   │       ├── mod.rs                     # Array factor computation
│   │       ├── lattice.rs                 # Array lattice generation
│   │       ├── taper.rs                   # Amplitude taper functions
│   │       └── steering.rs                # Beam steering
│   └── tests/
│       ├── canonical.rs                   # Canonical antenna validation
│       ├── validation.rs                  # Solver accuracy tests
│       └── arrays.rs                      # Array analysis tests
│
├── solver-wasm/                           # WASM compilation target
│   ├── Cargo.toml
│   └── src/
│       └── lib.rs                         # WASM entry points (wasm-bindgen)
│
├── antennasynth-api/                      # Rust API server
│   ├── Cargo.toml
│   ├── src/
│   │   ├── main.rs                        # Axum app setup, router, startup
│   │   ├── config.rs                      # Environment config
│   │   ├── state.rs                       # AppState (PgPool, Redis, S3)
│   │   ├── error.rs                       # ApiError types
│   │   ├── auth/
│   │   │   ├── mod.rs                     # JWT middleware, Claims
│   │   │   └── oauth.rs                   # Google/GitHub OAuth handlers
│   │   ├── api/
│   │   │   ├── router.rs                  # Route definitions
│   │   │   ├── handlers/
│   │   │   │   ├── auth.rs                # Register, login, OAuth
│   │   │   │   ├── users.rs               # Profile CRUD
│   │   │   │   ├── projects.rs            # Project CRUD, fork, share
│   │   │   │   ├── simulation.rs          # Create/get/cancel simulation
│   │   │   │   ├── catalog.rs             # Antenna catalog search
│   │   │   │   ├── substrates.rs          # Substrate library
│   │   │   │   ├── billing.rs             # Stripe checkout/portal
│   │   │   │   ├── usage.rs               # Usage tracking
│   │   │   │   ├── export.rs              # STL/CSV/JSON export
│   │   │   │   └── webhooks/
│   │   │   │       └── stripe.rs          # Stripe webhook handler
│   │   │   └── ws/
│   │   │       ├── mod.rs                 # WebSocket upgrade handler
│   │   │       └── simulation_progress.rs # Live progress streaming
│   │   ├── middleware/
│   │   │   └── plan_limits.rs             # Plan enforcement middleware
│   │   ├── services/
│   │   │   ├── usage.rs                   # Usage tracking service
│   │   │   └── s3.rs                      # S3 client helpers
│   │   └── workers/
│   │       ├── mod.rs                     # Worker pool management
│   │       └── simulation_worker.rs       # Server-side simulation execution
│   ├── migrations/
│   │   └── 001_initial.sql                # Full database schema
│   └── tests/
│       ├── api_integration.rs             # API integration tests
│       └── simulation_e2e.rs              # End-to-end simulation tests
│
├── synthesis-service/                     # Python FastAPI (antenna synthesis)
│   ├── requirements.txt
│   ├── main.py                            # FastAPI app
│   ├── synthesis/
│   │   ├── patch.py                       # Patch antenna synthesis
│   │   ├── dipole.py                      # Dipole synthesis
│   │   ├── pifa.py                        # PIFA synthesis
│   │   ├── horn.py                        # Horn antenna synthesis
│   │   └── optimizer.py                   # Dimension optimization (DE, PSO)
│   └── Dockerfile
│
├── frontend/                              # React frontend
│   ├── package.json
│   ├── vite.config.ts
│   ├── src/
│   │   ├── App.tsx                        # Router, providers, layout
│   │   ├── main.tsx                       # Entry point
│   │   ├── stores/
│   │   │   ├── authStore.ts               # Auth state (JWT, user)
│   │   │   ├── projectStore.ts            # Project state
│   │   │   ├── antennaStore.ts            # Antenna geometry and simulation
│   │   │   └── arrayStore.ts              # Array configuration state
│   │   ├── hooks/
│   │   │   ├── useSimulationProgress.ts   # WebSocket hook for live progress
│   │   │   └── useWasmSolver.ts           # WASM solver hook (load + run)
│   │   ├── lib/
│   │   │   ├── api.ts                     # Axios API client
│   │   │   └── wasmLoader.ts              # WASM bundle loading
│   │   ├── pages/
│   │   │   ├── Dashboard.tsx              # Project list
│   │   │   ├── Editor.tsx                 # Main editor (3D + results split)
│   │   │   ├── Templates.tsx              # Antenna template gallery
│   │   │   ├── Substrates.tsx             # Substrate library browser
│   │   │   ├── Billing.tsx                # Plan management
│   │   │   ├── Login.tsx                  # Auth pages
│   │   │   └── Register.tsx
│   │   ├── components/
│   │   │   ├── AntennaViewer3D/
│   │   │   │   ├── AntennaViewer3D.tsx    # Three.js canvas
│   │   │   │   ├── GeometryRenderer.tsx   # Render mesh
│   │   │   │   ├── FeedPoint.tsx          # Feed marker
│   │   │   │   ├── Substrate.tsx          # Substrate layer
│   │   │   │   └── GroundPlane.tsx        # Ground plane
│   │   │   ├── PatternViewer3D/
│   │   │   │   ├── PatternViewer3D.tsx    # Pattern 3D surface plot
│   │   │   │   └── PatternSurface.tsx     # Generate pattern mesh
│   │   │   ├── AntennaEditor/
│   │   │   │   ├── TopologySelector.tsx   # Select antenna type
│   │   │   │   ├── DimensionInputs.tsx    # Edit dimensions
│   │   │   │   ├── SubstrateSelector.tsx  # Select substrate
│   │   │   │   └── FeedConfig.tsx         # Feed configuration
│   │   │   ├── ArrayEditor/
│   │   │   │   ├── ArrayEditor.tsx        # Array layout editor
│   │   │   │   ├── LatticeSelector.tsx    # Lattice type selection
│   │   │   │   ├── ElementSpacing.tsx     # Spacing controls
│   │   │   │   └── ExcitationWeights.tsx  # Amplitude/phase weights
│   │   │   ├── SmithChart/
│   │   │   │   ├── SmithChart.tsx         # D3.js Smith chart
│   │   │   │   └── ImpedancePlot.tsx      # Impedance locus
│   │   │   ├── SParameterPlot.tsx         # S11 magnitude/phase plot
│   │   │   ├── ArrayPattern2D/
│   │   │   │   ├── ArrayPattern2D.tsx     # Polar plot of array factor
│   │   │   │   └── PolarPlot.tsx          # D3.js polar coordinates
│   │   │   ├── SimulationPanel.tsx        # Simulation controls
│   │   │   ├── PatternAnalysis.tsx        # Pattern measurements panel
│   │   │   ├── MatchingNetworkTool.tsx    # Impedance matching
│   │   │   ├── billing/
│   │   │   │   ├── PlanCard.tsx
│   │   │   │   └── UsageMeter.tsx
│   │   │   └── common/
│   │   │       └── SimulationToolbar.tsx  # Run/stop/export toolbar
│   │   └── data/
│   │       └── templates/                 # Pre-built antenna templates (JSON)
│   └── public/
│       └── wasm/                          # WASM solver bundle (loaded at runtime)
│
├── k8s/                                   # Kubernetes manifests
│   ├── api-deployment.yaml
│   ├── worker-deployment.yaml
│   ├── synthesis-deployment.yaml
│   ├── postgres-statefulset.yaml
│   ├── redis-deployment.yaml
│   └── ingress.yaml
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

### Benchmark 1: Half-Wave Dipole (Impedance and Pattern)

**Geometry:** Wire dipole, length L = 0.47λ at 300 MHz (λ = 1 m), wire radius a = 1 mm

**Expected Input Impedance:** Z_in = **73 + j42.5 Ω** (theory)

**Expected Gain:** G = **2.15 dBi** (directivity of ideal dipole)

**Tolerance:** Impedance < 2% (real/imaginary parts), Gain < 0.1 dB

### Benchmark 2: Quarter-Wave Monopole (Over Infinite Ground Plane)

**Geometry:** Monopole, length L = λ/4 at 900 MHz (λ = 333 mm), radius a = 1.5 mm, infinite ground plane

**Expected Input Impedance:** Z_in = **36.5 + j21 Ω** (half of dipole impedance)

**Expected Gain:** G = **5.15 dBi** (dipole gain + 3 dB ground plane reflection)

**Tolerance:** Impedance < 3%, Gain < 0.15 dB

### Benchmark 3: Rectangular Microstrip Patch (Resonance and Bandwidth)

**Geometry:** Patch on Rogers RO4003C (εr=3.38, h=1.524 mm, tan δ=0.0027), L=28.8 mm, W=37.5 mm, f₀=2.4 GHz

**Expected Resonant Frequency:** f₀ = **2.40 GHz** (cavity model prediction)

**Expected Gain:** G = **7.5 dBi** ± 1 dB (broadside)

**Expected Bandwidth:** BW = **3-5%** (impedance bandwidth for VSWR < 2)

**Tolerance:** Resonance < 2% (±48 MHz), Gain < 1 dB, Bandwidth < 20% relative error

### Benchmark 4: Broadside Array 1×4 (Array Factor Pattern)

**Geometry:** 4 isotropic elements, spacing d = λ/2, uniform excitation (amplitude=1, phase=0)

**Expected Beam Peak:** θ = **0°** (broadside)

**Expected Beamwidth (HPBW):** **~28°** (analytical array factor)

**Expected Sidelobe Level:** **-13.5 dB** (uniform array)

**Tolerance:** Beam peak direction < 2°, Beamwidth < 10% relative error, SLL < 1 dB error

### Benchmark 5: Taylor-Tapered 1×16 Array (Sidelobe Suppression)

**Geometry:** 16 isotropic elements, spacing d = λ/2, 35 dB Taylor taper (n̄=6)

**Expected Sidelobe Level:** **-35 dB** (design target)

**Expected Beamwidth (HPBW):** **~6°** (narrower than uniform due to larger effective aperture)

**Tolerance:** SLL < 2 dB error (> -33 dB), Beamwidth < 15% relative error

---

## Verification

### Integration Testing Checklist

1. **Auth flow** — Register → login → JWT issued → authorized API calls → token refresh → logout
2. **Project CRUD** — Create → update geometry → save → reload → verify geometry state preserved
3. **Synthesis** — Input specs (2.4 GHz patch) → synthesis API → verify dimensions match theory (±5%)
4. **Mesh generation** — Patch geometry → triangulation → verify ~500-2000 triangles (λ/10 target)
5. **WASM simulation** — 1K unknowns (patch) → WASM solver → results returned → S11 near resonance
6. **Server simulation** — 15K unknowns (large array) → job queued → worker picks up → WebSocket progress → results in S3
7. **Pattern visualization** — Load pattern data → Three.js render → rotate → gain max at broadside
8. **Smith chart** — Impedance locus → L-section match synthesis → verify component values
9. **Array factor** — 8-element array → adjust spacing to 1.0λ → grating lobe warning appears
10. **Plan limits** — Free user → antenna with 5K unknowns → blocked with upgrade prompt
11. **Billing** — Subscribe to Pro → Stripe checkout → webhook → plan updated → limits lifted
12. **Concurrent users** — 10 users simultaneously running server simulations → all complete correctly
13. **Export** — Pattern data → CSV export → verify format (theta, phi, gain_dbi columns)
14. **Template** — Select "2.4 GHz Patch" template → project created → simulate → expected S11 curve
15. **Error handling** — Singular matrix (bad mesh) → meaningful error message → no crash

### SQL Verification Queries

```sql
-- 1. Simulation throughput and success rate
SELECT
    execution_mode,
    status,
    COUNT(*) as count,
    AVG(wall_time_ms)::int as avg_time_ms,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY wall_time_ms)::int as p95_time_ms
FROM simulations
WHERE created_at >= NOW() - INTERVAL '7 days'
GROUP BY execution_mode, status;

-- 2. Antenna topology distribution
SELECT antenna_type, COUNT(*) as count,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 1) as pct
FROM projects
WHERE created_at >= NOW() - INTERVAL '30 days'
GROUP BY antenna_type ORDER BY count DESC;

-- 3. User plan distribution and conversion
SELECT plan, COUNT(*) as users,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 1) as pct
FROM users GROUP BY plan ORDER BY users DESC;

-- 4. Cloud simulation usage by user (billing period)
SELECT u.email, u.plan,
    SUM(ur.quantity) as total_minutes,
    CASE u.plan
        WHEN 'free' THEN 10
        WHEN 'pro' THEN 1500
        WHEN 'advanced' THEN 999999
    END as limit_minutes
FROM usage_records ur
JOIN users u ON ur.user_id = u.id
WHERE ur.period_start <= CURRENT_DATE AND ur.period_end >= CURRENT_DATE
    AND ur.record_type = 'simulation_minutes'
GROUP BY u.email, u.plan
ORDER BY total_minutes DESC;

-- 5. Substrate popularity
SELECT s.name, s.manufacturer,
    COUNT(DISTINCT p.id) as projects_using
FROM substrates s
JOIN projects p ON p.substrate_id = s.id
WHERE p.created_at >= NOW() - INTERVAL '30 days'
GROUP BY s.id
ORDER BY projects_using DESC
LIMIT 20;
```

### Performance Benchmarks

| Operation | Target | Method |
|-----------|--------|--------|
| WASM solver: 1K unknowns (patch) | <2s | Browser benchmark (Chrome DevTools) |
| WASM solver: 5K unknowns | <10s | Browser benchmark |
| WASM solver: 10K unknowns | <30s | Browser benchmark |
| Server solver: 20K unknowns | <3 min | Server timing, 8 cores |
| Server solver: 50K unknowns (array) | <15 min | Server timing, 16 cores |
| Pattern viewer: 10K points 3D | 60 FPS | Chrome FPS counter during rotation |
| Smith chart: impedance locus | <100ms | D3.js render time |
| Array factor: 64 elements | <500ms | WASM computation + render |
| API: create simulation | <200ms | p95 latency (Prometheus) |
| API: search catalog | <100ms | p95 latency with 30 topologies |
| WASM bundle load (cached) | <500ms | Service worker cache hit |
| WASM bundle load (cold CDN) | <2s | CloudFront delivery, 3MB gzipped |
| WebSocket: progress latency | <100ms | Time from worker emit to client receive |
| Mesh generation: patch | <50ms | Server-side Rust triangulation |

---

## Deployment Architecture

### Infrastructure

```
┌─────────────────────────────────────────────────┐
│                  CloudFront CDN                   │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────┐ │
│  │ Frontend SPA │  │ WASM Bundle │  │ Antenna  │ │
│  │ (HTML/JS/CSS)│  │ (versioned) │  │ Patterns │ │
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
│ Pod ×3 (HPA) │  │              │  │              │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘
       │                 │                 │
       └─────────────────┼─────────────────┘
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ PostgreSQL   │ │ Redis        │ │ S3 Bucket    │
│ (RDS r6g)    │ │ (ElastiCache)│ │ (Meshes +    │
│ Multi-AZ     │ │ Cluster mode │ │  Results)    │
└──────────────┘ └──────┬───────┘ └──────────────┘
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ Sim Worker   │ │ Sim Worker   │ │ Sim Worker   │
│ (Rust native)│ │ (Rust native)│ │ (Rust native)│
│ c7g.2xlarge  │ │ c7g.2xlarge  │ │ c7g.2xlarge  │
│ HPA: 2-20    │ │              │ │              │
└──────────────┘ └──────────────┘ └──────────────┘
```

### Scaling Strategy

- **API servers**: Horizontal pod autoscaler (HPA) at 70% CPU, min 3, max 10
- **Simulation workers**: HPA based on Redis queue depth — scale up when queue > 5 jobs, scale down when idle 5 min
- **Worker instances**: AWS c7g.2xlarge (8 vCPU, 16GB RAM) for compute-optimized MoM solver
- **Database**: RDS PostgreSQL r6g.xlarge with read replicas for catalog/substrate queries
- **Redis**: ElastiCache r6g.large cluster with 1 primary + 2 replicas

### S3 Lifecycle Policy

```json
{
  "Rules": [
    {
      "ID": "simulation-results-lifecycle",
      "Prefix": "results/",
      "Status": "Enabled",
      "Transitions": [
        { "Days": 30, "StorageClass": "STANDARD_IA" },
        { "Days": 90, "StorageClass": "GLACIER" }
      ],
      "Expiration": { "Days": 365 }
    },
    {
      "ID": "meshes-keep",
      "Prefix": "meshes/",
      "Status": "Enabled",
      "Transitions": [
        { "Days": 90, "StorageClass": "STANDARD_IA" }
      ],
      "Expiration": { "Days": 180 }
    },
    {
      "ID": "wasm-bundles-keep",
      "Prefix": "wasm/",
      "Status": "Enabled",
      "NoncurrentVersionExpiration": { "NoncurrentDays": 30 }
    }
  ]
}
```

---

## Post-MVP Roadmap

| Feature | Description | Priority |
|---------|-------------|----------|
| FDTD Solver (GPU) | GPU-accelerated FDTD for UWB antennas, transient analysis, large ground planes, and dielectric structures | High |
| Mutual Coupling Arrays | Full-wave array simulation with embedded element patterns, scan impedance, and active S-parameters | High |
| Beamforming Optimization | Analog/digital/hybrid beamforming weight optimization, null steering, and multi-beam synthesis | High |
| MIMO Antenna Design | Envelope correlation coefficient (ECC), total active reflection coefficient (TARC), diversity gain, and decoupling structure synthesis | High |
| Characteristic Mode Analysis | Modal decomposition, modal significance curves, and optimal feed placement using eigenvalue analysis | Medium |
| SAR Simulation | Specific absorption rate (SAR) evaluation with FDTD, anatomical phantoms, and FCC/IECEE compliance checking | Medium |
| Conformal Arrays | Cylindrical, spherical, and arbitrary surface arrays with phase compensation and pattern synthesis | Medium |
| Optimization Engine | Genetic algorithm, particle swarm, and Bayesian optimization for automated dimensional tuning to meet specs | Medium |
| Measurement Correlation | Import measured S-parameters and patterns, near-field to far-field transformation, and calibration error correction | Low |
| AI Topology Recommendation | Generative model trained on antenna literature to suggest novel topologies for unusual specification combinations | Low |
