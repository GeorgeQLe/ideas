# 44. AeroStream — Cloud Aerospace Structural & Aero Analysis Platform

## Implementation Plan

**MVP Scope:** Browser-based CAD viewer with STEP import and automated surface/volume mesh generation using advancing-front Delaunay algorithm, client-side panel method solver (Vortex Lattice Method) compiled to WebAssembly for lifting surfaces up to 200 panels with aerodynamic coefficient extraction (CL, CD, CM, Clα, Cmα) executing in <5 seconds, server-side linear static FEA solver in Rust with shell elements (DKT24/Mindlin-Reissner) and isotropic/orthotropic materials using MUMPS sparse direct solver for structures up to 50,000 DOF, composite layup editor with ply-by-ply definition (material, orientation, thickness) and Tsai-Wu failure criterion evaluation per ply, interactive WebGPU-based results visualization with pressure contours, deformation overlays, and von Mises stress plots, 200+ built-in aerospace materials (Al 2024-T3, Ti-6Al-4V, carbon/glass fiber epoxy prepregs) with mechanical properties stored in PostgreSQL, STEP geometry export with analysis results in HDF5 format, Stripe billing with three tiers (Academic Free / Startup $199/mo / Professional $499/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| CFD Solver (Panel) | Rust (native + WASM) | VLM panel method with vortex ring elements |
| FEA Solver | Rust (native only) | DKT shell elements, MUMPS sparse direct solver |
| WASM Compilation | `wasm-pack` + `wasm-bindgen` | Client-side panel method for ≤200 panels |
| CAD Import | OpenCASCADE (C++ FFI) | STEP/IGES import via Rust bindings to OCCT 7.7 |
| Meshing | Rust custom mesher | Delaunay triangulation + advancing-front for volume mesh |
| Post-Processing | Python 3.12 (FastAPI) | HDF5 result processing, load case generation, report templating |
| Database | PostgreSQL 16 | Projects, users, simulations, material library, load cases |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | STEP geometry files, mesh files (HDF5), result files (HDF5) |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| 3D Visualization | Three.js + WebGPU | CAD rendering, mesh display, pressure/stress contours |
| Real-time | WebSocket (Axum) | Live solve progress, convergence monitoring |
| Job Queue | Redis 7 + Tokio tasks | Server-side FEA/CFD job management, priority queue |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | WASM bundle delivery, static frontend assets |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, error tracking |
| CI/CD | GitHub Actions | WASM build pipeline, Rust tests, Docker image push |

### Key Architecture Decisions

1. **Hybrid panel method solver with WASM threshold at 200 panels**: Simple lifting surfaces (wings, stabilizers, control surfaces) with ≤200 panels run entirely in the browser via WASM panel method solver, providing aerodynamic coefficients in seconds with zero server cost. This covers 85%+ of preliminary design analysis (planform studies, airfoil selection, stability derivatives). Complex geometries or RANS CFD requiring volume meshes are submitted to the server. The threshold is enforced per plan tier.

2. **Custom Rust FEA solver rather than wrapping CalculiX/Code_Aster**: Building a custom FEA solver in Rust gives full control over element formulations (aerospace-specific shells for thin airframes), composite material models, and aeroelastic coupling. CalculiX and Code_Aster have extensive Fortran/C legacy code with global state that makes thread-safe job parallelization difficult. Our Rust solver uses `mumps-sys` bindings for sparse direct solving while maintaining memory safety and easy integration with our mesher.

3. **OpenCASCADE for STEP import via Rust FFI**: OpenCASCADE (OCCT) is the industry-standard kernel for CAD interoperability, supporting STEP AP203/AP214 and IGES formats. We use `opencascade-sys` Rust bindings to call OCCT C++ APIs for geometry import, topology extraction (faces, edges, vertices), and surface tessellation. This ensures compatibility with all major CAD tools (SolidWorks, CATIA, Siemens NX).

4. **Three.js + WebGPU for 3D visualization**: Three.js provides a mature scene graph and CAD rendering primitives (edges, faces, transparency) while WebGPU (with WebGL 2 fallback) enables GPU-accelerated rendering of large meshes (100K+ elements) with custom shaders for contour plots. Pressure and stress fields are passed as vertex attributes and interpolated on the GPU using fragment shaders for smooth gradients.

5. **HDF5 for mesh and result storage**: HDF5 is the standard format for large scientific datasets, providing efficient compression and chunked I/O for meshes and result arrays. Mesh files (nodes, elements, boundary conditions) and result files (displacements, stresses, pressure) are stored as HDF5 in S3, while PostgreSQL holds metadata (analysis type, load case, timestamp). This allows the library to scale to millions of elements without database bloat.

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";  -- For fuzzy text search on materials

-- Users
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email TEXT UNIQUE NOT NULL,
    password_hash TEXT,  -- NULL for OAuth-only users
    name TEXT NOT NULL,
    avatar_url TEXT,
    auth_provider TEXT DEFAULT 'email',  -- email | google | github
    auth_provider_id TEXT,
    plan TEXT NOT NULL DEFAULT 'academic',  -- academic | startup | professional | enterprise
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    organization_id UUID,
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
    plan TEXT NOT NULL DEFAULT 'startup',
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

-- Projects (geometry + analysis workspace)
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    geometry_url TEXT,  -- S3 URL to STEP file
    geometry_metadata JSONB DEFAULT '{}',  -- Bounding box, volume, surface area, part names
    settings JSONB DEFAULT '{}',  -- Mesh settings, material assignments, BC templates
    is_public BOOLEAN NOT NULL DEFAULT false,
    forked_from UUID REFERENCES projects(id) ON DELETE SET NULL,
    thumbnail_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);
CREATE INDEX projects_org_idx ON projects(org_id);
CREATE INDEX projects_updated_idx ON projects(updated_at DESC);

-- Simulations (both aero and structural)
CREATE TABLE simulations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    simulation_type TEXT NOT NULL,  -- panel_aero | rans_cfd | static_fea | modal_fea | buckling_fea
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | meshing | running | completed | failed | cancelled
    execution_mode TEXT NOT NULL DEFAULT 'server',  -- wasm | server
    mesh_url TEXT,  -- S3 URL to mesh HDF5 file
    results_url TEXT,  -- S3 URL to results HDF5 file
    parameters JSONB NOT NULL DEFAULT '{}',  -- Analysis-specific parameters (AOA, velocity, BCs, loads)
    mesh_stats JSONB,  -- Node count, element count, min/max quality
    results_summary JSONB,  -- Quick-access summary (CL, CD, max stress, mass, etc.)
    error_message TEXT,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    wall_time_ms INTEGER,
    compute_cost_usd REAL,
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
    memory_mb INTEGER DEFAULT 16384,
    priority INTEGER DEFAULT 0,  -- Higher = more urgent (professional plan gets +10)
    progress_pct REAL DEFAULT 0.0,
    current_step TEXT,  -- "meshing" | "solving" | "postprocessing"
    convergence_data JSONB DEFAULT '[]',  -- [{iteration, residual, time}]
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX jobs_sim_idx ON simulation_jobs(simulation_id);
CREATE INDEX jobs_status_idx ON simulation_jobs(worker_id);
CREATE INDEX jobs_priority_idx ON simulation_jobs(priority DESC);

-- Materials Library
CREATE TABLE materials (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,  -- e.g., "Al 2024-T3" or "T300/5208 Carbon Epoxy"
    material_type TEXT NOT NULL,  -- isotropic | orthotropic | anisotropic
    category TEXT NOT NULL,  -- metal | composite | polymer
    density REAL NOT NULL,  -- kg/m³
    -- Isotropic properties
    youngs_modulus REAL,  -- Pa
    poissons_ratio REAL,
    shear_modulus REAL,  -- Pa
    yield_strength REAL,  -- Pa
    ultimate_strength REAL,  -- Pa
    -- Orthotropic properties (for composites)
    e1 REAL, e2 REAL, e3 REAL,  -- Pa, principal directions
    g12 REAL, g13 REAL, g23 REAL,  -- Pa, shear moduli
    nu12 REAL, nu13 REAL, nu23 REAL,  -- Poisson's ratios
    -- Strengths (for failure criteria)
    xt REAL, xc REAL,  -- Tension/compression strength in 1-direction (Pa)
    yt REAL, yc REAL,  -- Tension/compression strength in 2-direction (Pa)
    s12 REAL,  -- Shear strength (Pa)
    -- Metadata
    manufacturer TEXT,
    specification TEXT,  -- e.g., "AMS 4037", "MIL-HDBK-5J"
    basis TEXT,  -- A-basis | B-basis | typical
    datasheet_url TEXT,
    is_builtin BOOLEAN DEFAULT true,
    uploaded_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX materials_type_idx ON materials(material_type);
CREATE INDEX materials_category_idx ON materials(category);
CREATE INDEX materials_name_trgm_idx ON materials USING gin(name gin_trgm_ops);

-- Load Cases (for structural analysis)
CREATE TABLE load_cases (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    load_type TEXT NOT NULL,  -- maneuver | gust | landing | taxi | combined
    load_factor REAL NOT NULL,  -- e.g., 2.5 for +2.5g pull-up
    velocity_mps REAL,  -- Velocity for aero loads (m/s)
    altitude_m REAL,  -- Altitude for density (m)
    boundary_conditions JSONB NOT NULL DEFAULT '[]',  -- [{region, type, dof, value}]
    applied_loads JSONB NOT NULL DEFAULT '[]',  -- [{region, type, force_vector, moment_vector}]
    aero_data_url TEXT,  -- S3 URL to aero pressure distribution if coupled
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX load_cases_project_idx ON load_cases(project_id);

-- Composite Layups (for composite structures)
CREATE TABLE composite_layups (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    plies JSONB NOT NULL,  -- [{material_id, thickness_mm, angle_deg, sequence}]
    total_thickness_mm REAL NOT NULL,
    areal_weight_kgm2 REAL NOT NULL,
    symmetric BOOLEAN DEFAULT false,
    balanced BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX layups_project_idx ON composite_layups(project_id);

-- Usage Tracking (for billing enforcement)
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,  -- compute_minutes | storage_gb | api_calls
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
    pub organization_id: Option<Uuid>,
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
    pub geometry_url: Option<String>,
    pub geometry_metadata: serde_json::Value,
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
    pub simulation_type: String,
    pub status: String,
    pub execution_mode: String,
    pub mesh_url: Option<String>,
    pub results_url: Option<String>,
    pub parameters: serde_json::Value,
    pub mesh_stats: Option<serde_json::Value>,
    pub results_summary: Option<serde_json::Value>,
    pub error_message: Option<String>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub wall_time_ms: Option<i32>,
    pub compute_cost_usd: Option<f32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Material {
    pub id: Uuid,
    pub name: String,
    pub material_type: String,
    pub category: String,
    pub density: f32,
    // Isotropic
    pub youngs_modulus: Option<f32>,
    pub poissons_ratio: Option<f32>,
    pub shear_modulus: Option<f32>,
    pub yield_strength: Option<f32>,
    pub ultimate_strength: Option<f32>,
    // Orthotropic
    pub e1: Option<f32>,
    pub e2: Option<f32>,
    pub e3: Option<f32>,
    pub g12: Option<f32>,
    pub g13: Option<f32>,
    pub g23: Option<f32>,
    pub nu12: Option<f32>,
    pub nu13: Option<f32>,
    pub nu23: Option<f32>,
    // Strengths
    pub xt: Option<f32>,
    pub xc: Option<f32>,
    pub yt: Option<f32>,
    pub yc: Option<f32>,
    pub s12: Option<f32>,
    pub manufacturer: Option<String>,
    pub specification: Option<String>,
    pub basis: Option<String>,
    pub datasheet_url: Option<String>,
    pub is_builtin: bool,
    pub uploaded_by: Option<Uuid>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct LoadCase {
    pub id: Uuid,
    pub project_id: Uuid,
    pub name: String,
    pub description: String,
    pub load_type: String,
    pub load_factor: f32,
    pub velocity_mps: Option<f32>,
    pub altitude_m: Option<f32>,
    pub boundary_conditions: serde_json::Value,
    pub applied_loads: serde_json::Value,
    pub aero_data_url: Option<String>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct PanelAeroParams {
    pub aoa_deg: f64,       // Angle of attack
    pub aos_deg: f64,       // Angle of sideslip
    pub velocity_mps: f64,  // Freestream velocity
    pub altitude_m: f64,    // For density/temperature
    pub reference_area: f64,
    pub reference_chord: f64,
    pub reference_span: f64,
}

#[derive(Debug, Deserialize)]
pub struct StaticFEAParams {
    pub load_case_id: Uuid,
    pub material_assignments: Vec<MaterialAssignment>,
    pub mesh_size_mm: f64,
    pub solver_tolerance: f64,
}

#[derive(Debug, Deserialize)]
pub struct MaterialAssignment {
    pub region_name: String,
    pub material_id: Uuid,
    pub thickness_mm: Option<f64>,  // For shell elements
    pub layup_id: Option<Uuid>,     // For composite layups
}
```

---

## Solver Architecture Deep-Dive

### Panel Method (Vortex Lattice Method)

AeroStream's rapid aerodynamic analysis uses a 3D Vortex Lattice Method (VLM) implementation for lifting surfaces. VLM models the surface as a grid of quadrilateral panels, each carrying a vortex ring of unknown strength. The no-penetration boundary condition (flow tangent to surface) yields a linear system:

```
[AIC] {Γ} = {RHS}

where:
  AIC[i,j] = influence coefficient: normal velocity at panel i due to unit vortex on panel j
  Γ[j] = vortex strength on panel j (unknown)
  RHS[i] = freestream normal velocity at panel i
```

Once solved, pressure coefficients are computed via the unsteady Bernoulli equation linearized for small perturbations:

```
Cp = 1 - (V_local / V_inf)²

where V_local is obtained from potential gradient
```

Aerodynamic forces and moments are integrated over panels:

```
L = ∫ Cp · n · dS  (lift in body axes)
D = induced drag from Trefftz plane analysis
M = ∫ (r × (Cp · n)) dS  (moments about reference point)
```

**Stability derivatives** (Clα, Cmα, Cnβ, etc.) are computed by finite-difference perturbations of AOA/AOS.

### FEA Solver (Linear Static Shell Elements)

The structural solver uses the DKT (Discrete Kirchhoff Triangle) shell element, a 3-node triangular element with 6 DOF per node (3 translations + 3 rotations). The element combines membrane (in-plane) and bending stiffness:

```
[K_element] = [K_membrane] + [K_bending]

K_membrane from constant-strain triangle (CST)
K_bending from DKT plate theory (Kirchhoff constraint enforced at discrete points)
```

For composite shells, the laminate stiffness matrix is computed from classical lamination theory (CLT):

```
[A B]   = ∑ [Q̄]_k · (z_k - z_{k-1})
[B D]     k

where:
  A = in-plane stiffness (3×3)
  B = coupling stiffness (3×3, zero for symmetric laminates)
  D = bending stiffness (3×3)
  Q̄ = transformed ply stiffness in laminate coordinates
```

The global system is assembled:

```
[K_global] {u} = {F}

Solved via MUMPS sparse direct solver (LU factorization)
```

Stress recovery uses element shape functions to interpolate displacements to Gauss points, then computes strains and stresses per ply.

### Composite Failure Analysis

Each ply is evaluated using the **Tsai-Wu failure criterion**:

```
F_i σ_i + F_{ij} σ_i σ_j < 1  (safe)

where:
  F_1 = 1/X_t - 1/X_c
  F_2 = 1/Y_t - 1/Y_c
  F_{11} = 1/(X_t · X_c)
  F_{22} = 1/(Y_t · Y_c)
  F_{66} = 1/S²
  F_{12} = -0.5 / √(X_t X_c Y_t Y_c)  (typical value)
```

**Reserve Factor (RF)** is computed as the load multiplier to reach failure:

```
RF = 1 / √(F_i σ_i + F_{ij} σ_i σ_j)
```

Results report the first-ply failure load and critical ply.

### Client/Server Execution Split

```
Analysis request → Panel count / DOF extracted
    │
    ├── Panel aero ≤200 panels → WASM solver (browser)
    │   ├── Instant startup, <5s solve
    │   ├── Results displayed immediately
    │   └── No server cost
    │
    └── FEA or complex aero → Server solver (Rust native)
        ├── Meshing step (advancing-front Delaunay)
        ├── Job queued via Redis (priority by plan)
        ├── Worker picks up, runs native solver
        ├── Progress streamed via WebSocket
        └── Results stored in HDF5 on S3, URL returned
```

Plan limits:
- **Academic**: WASM panel only, server queue (low priority), 50K DOF max
- **Startup**: Priority queue, 200K DOF max, composite analysis
- **Professional**: Dedicated compute, unlimited DOF, API access

---

## Architecture Deep-Dives

### 1. STEP Geometry Import Handler (Rust + OpenCASCADE FFI)

The geometry import pipeline reads STEP files via OpenCASCADE bindings, extracts topology (solids, shells, faces), and generates a surface triangulation for display and meshing seed.

```rust
// src/geometry/step_import.rs

use opencascade::{
    import::step::StepReader,
    primitives::Shape,
    topology::{ShapeType, TopoExplorer},
    meshing::IncrementalMesh,
};
use std::path::Path;

pub struct GeometryImporter;

#[derive(Debug)]
pub struct ImportedGeometry {
    pub shapes: Vec<Shape>,
    pub faces: Vec<TopoFace>,
    pub edges: Vec<TopoEdge>,
    pub bounding_box: BoundingBox,
    pub volume: f64,
    pub surface_area: f64,
}

#[derive(Debug)]
pub struct TopoFace {
    pub id: usize,
    pub surface_type: String,  // plane | cylinder | sphere | bspline | nurbs
    pub area: f64,
    pub triangulation: Vec<Triangle>,
}

#[derive(Debug)]
pub struct Triangle {
    pub vertices: [Vector3; 3],
    pub normal: Vector3,
}

impl GeometryImporter {
    pub fn import_step(path: &Path) -> Result<ImportedGeometry, GeometryError> {
        // 1. Read STEP file using OpenCASCADE
        let mut reader = StepReader::new();
        reader.read_file(path)?;

        if reader.nb_shapes() == 0 {
            return Err(GeometryError::EmptyGeometry);
        }

        // 2. Extract all shapes
        let mut shapes = Vec::new();
        for i in 1..=reader.nb_shapes() {
            shapes.push(reader.shape(i)?);
        }

        // 3. Extract topology (faces, edges)
        let mut faces = Vec::new();
        let mut edges = Vec::new();
        let mut face_id = 0;

        for shape in &shapes {
            let face_explorer = TopoExplorer::new(shape, ShapeType::Face);

            for face_shape in face_explorer {
                // Generate surface triangulation for visualization and meshing seed
                let mesh = IncrementalMesh::new(&face_shape, 0.5, false, 0.1)?;

                let triangulation = Self::extract_triangulation(&face_shape, &mesh)?;
                let area = Self::compute_surface_area(&triangulation);

                faces.push(TopoFace {
                    id: face_id,
                    surface_type: Self::classify_surface(&face_shape),
                    area,
                    triangulation,
                });
                face_id += 1;
            }

            let edge_explorer = TopoExplorer::new(shape, ShapeType::Edge);
            for edge_shape in edge_explorer {
                edges.push(Self::extract_edge(&edge_shape)?);
            }
        }

        // 4. Compute bounding box and properties
        let bounding_box = Self::compute_bounding_box(&shapes)?;
        let volume = Self::compute_volume(&shapes)?;
        let surface_area = faces.iter().map(|f| f.area).sum();

        Ok(ImportedGeometry {
            shapes,
            faces,
            edges,
            bounding_box,
            volume,
            surface_area,
        })
    }

    fn extract_triangulation(
        face: &Shape,
        mesh: &IncrementalMesh,
    ) -> Result<Vec<Triangle>, GeometryError> {
        let mut triangles = Vec::new();

        let location = mesh.location(face)?;
        let triangle_count = mesh.nb_triangles(&location);

        for i in 1..=triangle_count {
            let (n1, n2, n3) = mesh.triangle_nodes(&location, i)?;
            let v1 = mesh.node_coord(&location, n1)?;
            let v2 = mesh.node_coord(&location, n2)?;
            let v3 = mesh.node_coord(&location, n3)?;

            let normal = Self::compute_normal(v1, v2, v3);

            triangles.push(Triangle {
                vertices: [v1, v2, v3],
                normal,
            });
        }

        Ok(triangles)
    }

    fn compute_normal(v1: Vector3, v2: Vector3, v3: Vector3) -> Vector3 {
        let edge1 = Vector3::new(v2.x - v1.x, v2.y - v1.y, v2.z - v1.z);
        let edge2 = Vector3::new(v3.x - v1.x, v3.y - v1.y, v3.z - v1.z);
        edge1.cross(&edge2).normalize()
    }

    fn classify_surface(face: &Shape) -> String {
        use opencascade::primitives::Surface;

        match face.surface_type() {
            Surface::Plane => "plane".to_string(),
            Surface::Cylinder => "cylinder".to_string(),
            Surface::Sphere => "sphere".to_string(),
            Surface::Cone => "cone".to_string(),
            Surface::BSpline => "bspline".to_string(),
            _ => "nurbs".to_string(),
        }
    }

    fn compute_bounding_box(shapes: &[Shape]) -> Result<BoundingBox, GeometryError> {
        let mut min = Vector3::new(f64::INFINITY, f64::INFINITY, f64::INFINITY);
        let mut max = Vector3::new(f64::NEG_INFINITY, f64::NEG_INFINITY, f64::NEG_INFINITY);

        for shape in shapes {
            let bbox = shape.bounding_box()?;
            min.x = min.x.min(bbox.x_min);
            min.y = min.y.min(bbox.y_min);
            min.z = min.z.min(bbox.z_min);
            max.x = max.x.max(bbox.x_max);
            max.y = max.y.max(bbox.y_max);
            max.z = max.z.max(bbox.z_max);
        }

        Ok(BoundingBox { min, max })
    }

    fn compute_volume(shapes: &[Shape]) -> Result<f64, GeometryError> {
        use opencascade::properties::GProp;

        let mut total_volume = 0.0;
        for shape in shapes {
            let props = GProp::volume_properties(shape)?;
            total_volume += props.mass();
        }
        Ok(total_volume)
    }
}

#[derive(Debug)]
pub enum GeometryError {
    FileNotFound,
    EmptyGeometry,
    InvalidSTEP(String),
    OCCTError(String),
}
```

### 2. Panel Method Solver Core (Rust — WASM + Native)

The VLM solver builds the aerodynamic influence coefficient (AIC) matrix and solves for vortex strengths, then integrates forces and moments.

```rust
// solver-panel/src/vlm.rs

use nalgebra::{DMatrix, DVector};

#[derive(Debug, Clone)]
pub struct Panel {
    pub id: usize,
    pub vertices: [Vector3; 4],  // Quadrilateral corners in CCW order
    pub control_point: Vector3,  // 3/4-chord point where BC applied
    pub normal: Vector3,
    pub area: f64,
    pub chord: f64,
}

#[derive(Debug)]
pub struct VLMSolver {
    pub panels: Vec<Panel>,
    pub freestream: Vector3,
    pub reference_area: f64,
    pub reference_chord: f64,
    pub reference_span: f64,
}

#[derive(Debug)]
pub struct AeroCoefficients {
    pub cl: f64,  // Lift coefficient
    pub cd: f64,  // Drag coefficient (induced only)
    pub cm: f64,  // Pitching moment coefficient
    pub cy: f64,  // Side force coefficient
    pub cn: f64,  // Yawing moment coefficient
    pub cl_alpha: f64,  // Lift curve slope (per radian)
    pub cm_alpha: f64,  // Pitch stiffness
}

impl VLMSolver {
    pub fn new(
        panels: Vec<Panel>,
        velocity_mps: f64,
        aoa_deg: f64,
        aos_deg: f64,
        reference_area: f64,
        reference_chord: f64,
        reference_span: f64,
    ) -> Self {
        let aoa_rad = aoa_deg.to_radians();
        let aos_rad = aos_deg.to_radians();

        // Freestream velocity in body axes
        let freestream = Vector3::new(
            velocity_mps * aoa_rad.cos() * aos_rad.cos(),
            velocity_mps * aos_rad.sin(),
            velocity_mps * aoa_rad.sin() * aos_rad.cos(),
        );

        Self {
            panels,
            freestream,
            reference_area,
            reference_chord,
            reference_span,
        }
    }

    pub fn solve(&mut self) -> Result<AeroCoefficients, SolverError> {
        let n = self.panels.len();

        // 1. Build AIC matrix
        let mut aic = DMatrix::zeros(n, n);
        for i in 0..n {
            for j in 0..n {
                aic[(i, j)] = self.influence_coefficient(i, j);
            }
        }

        // 2. Build RHS (freestream normal velocity at control points)
        let mut rhs = DVector::zeros(n);
        for i in 0..n {
            let cp = &self.panels[i].control_point;
            let normal = &self.panels[i].normal;
            rhs[i] = -self.freestream.dot(normal);
        }

        // 3. Solve linear system [AIC]{Γ} = {RHS}
        let gamma = aic.lu().solve(&rhs)
            .ok_or(SolverError::SingularMatrix)?;

        // 4. Compute pressure coefficients
        let mut cp_values = vec![0.0; n];
        for i in 0..n {
            let v_induced = self.induced_velocity(i, &gamma);
            let v_total = self.freestream + v_induced;
            let v_mag = v_total.magnitude();
            cp_values[i] = 1.0 - (v_mag / self.freestream.magnitude()).powi(2);
        }

        // 5. Integrate forces and moments
        let (lift, drag, side_force, moments) = self.integrate_forces(&cp_values);

        // 6. Non-dimensionalize
        let q_inf = 0.5 * 1.225 * self.freestream.magnitude_squared();  // Dynamic pressure (assume sea level)
        let s_ref = self.reference_area;

        let cl = lift / (q_inf * s_ref);
        let cd = drag / (q_inf * s_ref);
        let cy = side_force / (q_inf * s_ref);
        let cm = moments.y / (q_inf * s_ref * self.reference_chord);
        let cn = moments.z / (q_inf * s_ref * self.reference_span);

        // 7. Compute stability derivatives via finite difference
        let cl_alpha = self.compute_cl_alpha()?;
        let cm_alpha = self.compute_cm_alpha()?;

        Ok(AeroCoefficients {
            cl, cd, cm, cy, cn, cl_alpha, cm_alpha,
        })
    }

    /// Compute influence coefficient: normal velocity at panel i due to unit vortex ring on panel j
    fn influence_coefficient(&self, i: usize, j: usize) -> f64 {
        let cp = &self.panels[i].control_point;
        let normal = &self.panels[i].normal;

        // Vortex ring is composed of 4 vortex filaments along panel edges
        let verts = &self.panels[j].vertices;
        let mut induced = Vector3::zeros();

        for k in 0..4 {
            let v1 = &verts[k];
            let v2 = &verts[(k + 1) % 4];
            induced += self.vortex_filament_velocity(cp, v1, v2, 1.0);
        }

        induced.dot(normal)
    }

    /// Biot-Savart law for straight vortex filament
    fn vortex_filament_velocity(
        &self,
        point: &Vector3,
        v1: &Vector3,
        v2: &Vector3,
        strength: f64,
    ) -> Vector3 {
        let r1 = point - v1;
        let r2 = point - v2;
        let r0 = v2 - v1;

        let r1_mag = r1.magnitude();
        let r2_mag = r2.magnitude();

        if r1_mag < 1e-10 || r2_mag < 1e-10 {
            return Vector3::zeros();
        }

        let cross = r1.cross(&r2);
        let cross_mag_sq = cross.magnitude_squared();

        if cross_mag_sq < 1e-20 {
            return Vector3::zeros();
        }

        let cos1 = r0.dot(&r1) / (r0.magnitude() * r1_mag);
        let cos2 = r0.dot(&r2) / (r0.magnitude() * r2_mag);

        let k = strength / (4.0 * std::f64::consts::PI * cross_mag_sq);
        cross * k * (cos1 - cos2)
    }

    fn induced_velocity(&self, panel_idx: usize, gamma: &DVector<f64>) -> Vector3 {
        let cp = &self.panels[panel_idx].control_point;
        let mut v_induced = Vector3::zeros();

        for (j, &strength) in gamma.iter().enumerate() {
            let verts = &self.panels[j].vertices;
            for k in 0..4 {
                let v1 = &verts[k];
                let v2 = &verts[(k + 1) % 4];
                v_induced += self.vortex_filament_velocity(cp, v1, v2, strength);
            }
        }

        v_induced
    }

    fn integrate_forces(&self, cp_values: &[f64]) -> (f64, f64, f64, Vector3) {
        let mut lift = 0.0;
        let mut drag = 0.0;
        let mut side_force = 0.0;
        let mut moments = Vector3::zeros();

        let q_inf = 0.5 * 1.225 * self.freestream.magnitude_squared();

        for (i, &cp) in cp_values.iter().enumerate() {
            let panel = &self.panels[i];
            let force_magnitude = cp * q_inf * panel.area;
            let force_vector = panel.normal * force_magnitude;

            // Decompose into body axes
            lift += force_vector.z;
            drag -= force_vector.x;  // Induced drag (negative because opposes freestream)
            side_force += force_vector.y;

            // Moments about origin
            let moment_arm = panel.control_point;
            moments += moment_arm.cross(&force_vector);
        }

        (lift, drag, side_force, moments)
    }

    fn compute_cl_alpha(&mut self) -> Result<f64, SolverError> {
        // Finite difference: dCL/dα ≈ (CL(α+Δα) - CL(α-Δα)) / (2Δα)
        let delta_aoa = 0.5_f64.to_radians();  // 0.5 degree perturbation

        let original_v = self.freestream.clone();
        let v_mag = original_v.magnitude();

        // α + Δα
        let aoa_plus = (original_v.z / original_v.x).atan() + delta_aoa;
        self.freestream = Vector3::new(
            v_mag * aoa_plus.cos(),
            original_v.y,
            v_mag * aoa_plus.sin(),
        );
        let cl_plus = self.solve()?.cl;

        // α - Δα
        let aoa_minus = (original_v.z / original_v.x).atan() - delta_aoa;
        self.freestream = Vector3::new(
            v_mag * aoa_minus.cos(),
            original_v.y,
            v_mag * aoa_minus.sin(),
        );
        let cl_minus = self.solve()?.cl;

        // Restore original freestream
        self.freestream = original_v;

        Ok((cl_plus - cl_minus) / (2.0 * delta_aoa))
    }

    fn compute_cm_alpha(&mut self) -> Result<f64, SolverError> {
        let delta_aoa = 0.5_f64.to_radians();
        let original_v = self.freestream.clone();
        let v_mag = original_v.magnitude();

        let aoa_plus = (original_v.z / original_v.x).atan() + delta_aoa;
        self.freestream = Vector3::new(v_mag * aoa_plus.cos(), original_v.y, v_mag * aoa_plus.sin());
        let cm_plus = self.solve()?.cm;

        let aoa_minus = (original_v.z / original_v.x).atan() - delta_aoa;
        self.freestream = Vector3::new(v_mag * aoa_minus.cos(), original_v.y, v_mag * aoa_minus.sin());
        let cm_minus = self.solve()?.cm;

        self.freestream = original_v;
        Ok((cm_plus - cm_minus) / (2.0 * delta_aoa))
    }
}

#[derive(Debug)]
pub enum SolverError {
    SingularMatrix,
    InvalidGeometry(String),
}

#[derive(Debug, Clone, Copy)]
pub struct Vector3 {
    pub x: f64,
    pub y: f64,
    pub z: f64,
}

impl Vector3 {
    pub fn new(x: f64, y: f64, z: f64) -> Self { Self { x, y, z } }
    pub fn zeros() -> Self { Self::new(0.0, 0.0, 0.0) }
    pub fn dot(&self, other: &Self) -> f64 {
        self.x * other.x + self.y * other.y + self.z * other.z
    }
    pub fn cross(&self, other: &Self) -> Self {
        Self::new(
            self.y * other.z - self.z * other.y,
            self.z * other.x - self.x * other.z,
            self.x * other.y - self.y * other.x,
        )
    }
    pub fn magnitude(&self) -> f64 {
        (self.x * self.x + self.y * self.y + self.z * self.z).sqrt()
    }
    pub fn magnitude_squared(&self) -> f64 {
        self.x * self.x + self.y * self.y + self.z * self.z
    }
    pub fn normalize(&self) -> Self {
        let mag = self.magnitude();
        Self::new(self.x / mag, self.y / mag, self.z / mag)
    }
}

impl std::ops::Add for Vector3 {
    type Output = Self;
    fn add(self, other: Self) -> Self {
        Self::new(self.x + other.x, self.y + other.y, self.z + other.z)
    }
}

impl std::ops::Sub for Vector3 {
    type Output = Self;
    fn sub(self, other: Self) -> Self {
        Self::new(self.x - other.x, self.y - other.y, self.z - other.z)
    }
}

impl std::ops::Mul<f64> for Vector3 {
    type Output = Self;
    fn mul(self, scalar: f64) -> Self {
        Self::new(self.x * scalar, self.y * scalar, self.z * scalar)
    }
}

impl std::ops::AddAssign for Vector3 {
    fn add_assign(&mut self, other: Self) {
        self.x += other.x;
        self.y += other.y;
        self.z += other.z;
    }
}
```

### 3. FEA Shell Element (DKT Triangle)

The finite element solver constructs element stiffness matrices for DKT shell elements, assembles the global system, and solves via sparse direct method.

```rust
// solver-fea/src/elements/dkt_shell.rs

use nalgebra::{Matrix6, Vector6, SMatrix, SVector};

/// DKT (Discrete Kirchhoff Triangle) shell element
/// 3 nodes, 6 DOF per node (ux, uy, uz, rx, ry, rz)
pub struct DKTShell {
    pub id: usize,
    pub nodes: [usize; 3],  // Global node IDs
    pub thickness: f64,
    pub material: Material,
    pub coords: [Vector3; 3],  // Nodal coordinates in global frame
}

#[derive(Clone)]
pub struct Material {
    pub material_type: MaterialType,
    pub density: f64,
    // Isotropic
    pub e: Option<f64>,
    pub nu: Option<f64>,
    // Orthotropic
    pub e1: Option<f64>,
    pub e2: Option<f64>,
    pub g12: Option<f64>,
    pub nu12: Option<f64>,
}

#[derive(Clone)]
pub enum MaterialType {
    Isotropic,
    Orthotropic,
}

impl DKTShell {
    /// Compute 18×18 element stiffness matrix in global coordinates
    pub fn stiffness_matrix(&self) -> SMatrix<f64, 18, 18> {
        // 1. Compute local coordinate system (element plane)
        let (e1, e2, e3) = self.local_coordinate_system();

        // 2. Transform nodal coords to local frame
        let local_coords = self.transform_to_local(&e1, &e2, &e3);

        // 3. Compute membrane stiffness (in-plane, constant strain triangle)
        let k_membrane = self.membrane_stiffness(&local_coords);

        // 4. Compute bending stiffness (DKT plate formulation)
        let k_bending = self.bending_stiffness(&local_coords);

        // 5. Combine membrane + bending in local frame
        let mut k_local = SMatrix::<f64, 18, 18>::zeros();

        // Membrane DOF: (ux, uy) for nodes 1,2,3 → indices 0,1,3,4,6,7
        for i in 0..3 {
            for j in 0..3 {
                // u-u coupling
                k_local[(i*6, j*6)] = k_membrane[(i*2, j*2)];
                k_local[(i*6, j*6+1)] = k_membrane[(i*2, j*2+1)];
                k_local[(i*6+1, j*6)] = k_membrane[(i*2+1, j*2)];
                k_local[(i*6+1, j*6+1)] = k_membrane[(i*2+1, j*2+1)];
            }
        }

        // Bending DOF: (uz, rx, ry) for nodes 1,2,3 → indices 2,3,4 per node
        for i in 0..3 {
            for j in 0..3 {
                k_local[(i*6+2, j*6+2)] = k_bending[(i*3, j*3)];
                k_local[(i*6+2, j*6+3)] = k_bending[(i*3, j*3+1)];
                k_local[(i*6+2, j*6+4)] = k_bending[(i*3, j*3+2)];
                k_local[(i*6+3, j*6+2)] = k_bending[(i*3+1, j*3)];
                k_local[(i*6+3, j*6+3)] = k_bending[(i*3+1, j*3+1)];
                k_local[(i*6+3, j*6+4)] = k_bending[(i*3+1, j*3+2)];
                k_local[(i*6+4, j*6+2)] = k_bending[(i*3+2, j*3)];
                k_local[(i*6+4, j*6+3)] = k_bending[(i*3+2, j*3+1)];
                k_local[(i*6+4, j*6+4)] = k_bending[(i*3+2, j*3+2)];
            }
        }

        // 6. Transform to global coordinates
        let t = self.transformation_matrix(&e1, &e2, &e3);
        t.transpose() * k_local * t
    }

    fn local_coordinate_system(&self) -> (Vector3, Vector3, Vector3) {
        // e1 along side 1→2
        let e1 = (self.coords[1] - self.coords[0]).normalize();

        // e3 normal to element plane
        let v12 = self.coords[1] - self.coords[0];
        let v13 = self.coords[2] - self.coords[0];
        let e3 = v12.cross(&v13).normalize();

        // e2 completes right-handed system
        let e2 = e3.cross(&e1);

        (e1, e2, e3)
    }

    fn membrane_stiffness(&self, local_coords: &[Vector3; 3]) -> SMatrix<f64, 6, 6> {
        // Constant strain triangle (CST) membrane element
        let x = [local_coords[0].x, local_coords[1].x, local_coords[2].x];
        let y = [local_coords[0].y, local_coords[1].y, local_coords[2].y];

        // Area
        let area = 0.5 * ((x[1] - x[0]) * (y[2] - y[0]) - (x[2] - x[0]) * (y[1] - y[0]));

        // B matrix (strain-displacement): ε = B * u
        let b = self.membrane_b_matrix(&x, &y, area);

        // Material stiffness (plane stress)
        let d = self.material_matrix_membrane();

        // K = ∫ B^T D B dA = B^T D B * A * t
        let k = b.transpose() * d * b * area * self.thickness;
        k
    }

    fn membrane_b_matrix(&self, x: &[f64; 3], y: &[f64; 3], area: f64) -> SMatrix<f64, 3, 6> {
        let mut b = SMatrix::<f64, 3, 6>::zeros();
        let two_area = 2.0 * area;

        // Node 1
        b[(0, 0)] = (y[1] - y[2]) / two_area;
        b[(1, 1)] = (x[2] - x[1]) / two_area;
        b[(2, 0)] = (x[2] - x[1]) / two_area;
        b[(2, 1)] = (y[1] - y[2]) / two_area;

        // Node 2
        b[(0, 2)] = (y[2] - y[0]) / two_area;
        b[(1, 3)] = (x[0] - x[2]) / two_area;
        b[(2, 2)] = (x[0] - x[2]) / two_area;
        b[(2, 3)] = (y[2] - y[0]) / two_area;

        // Node 3
        b[(0, 4)] = (y[0] - y[1]) / two_area;
        b[(1, 5)] = (x[1] - x[0]) / two_area;
        b[(2, 4)] = (x[1] - x[0]) / two_area;
        b[(2, 5)] = (y[0] - y[1]) / two_area;

        b
    }

    fn material_matrix_membrane(&self) -> SMatrix<f64, 3, 3> {
        match self.material.material_type {
            MaterialType::Isotropic => {
                let e = self.material.e.unwrap();
                let nu = self.material.nu.unwrap();
                let factor = e / (1.0 - nu * nu);

                SMatrix::<f64, 3, 3>::from_row_slice(&[
                    factor,       factor * nu,  0.0,
                    factor * nu,  factor,       0.0,
                    0.0,          0.0,          factor * (1.0 - nu) / 2.0,
                ])
            }
            MaterialType::Orthotropic => {
                let e1 = self.material.e1.unwrap();
                let e2 = self.material.e2.unwrap();
                let g12 = self.material.g12.unwrap();
                let nu12 = self.material.nu12.unwrap();
                let nu21 = nu12 * e2 / e1;
                let denom = 1.0 - nu12 * nu21;

                SMatrix::<f64, 3, 3>::from_row_slice(&[
                    e1 / denom,          nu21 * e1 / denom,  0.0,
                    nu12 * e2 / denom,   e2 / denom,         0.0,
                    0.0,                 0.0,                g12,
                ])
            }
        }
    }

    fn bending_stiffness(&self, local_coords: &[Vector3; 3]) -> SMatrix<f64, 9, 9> {
        // DKT plate bending formulation
        // This is a simplified implementation; full DKT requires shape function derivatives
        // and Kirchhoff constraint enforcement at discrete points

        let x = [local_coords[0].x, local_coords[1].x, local_coords[2].x];
        let y = [local_coords[0].y, local_coords[1].y, local_coords[2].y];
        let area = 0.5 * ((x[1] - x[0]) * (y[2] - y[0]) - (x[2] - x[0]) * (y[1] - y[0]));

        // Bending material matrix D = E*t³ / (12*(1-ν²)) * [...]
        let d_bending = self.material_matrix_bending();

        // B matrix for bending (curvature-displacement)
        let b_bending = self.bending_b_matrix(&x, &y, area);

        // K = ∫ B^T D B dA
        b_bending.transpose() * d_bending * b_bending * area
    }

    fn material_matrix_bending(&self) -> SMatrix<f64, 3, 3> {
        let t = self.thickness;
        let factor = t * t * t / 12.0;

        match self.material.material_type {
            MaterialType::Isotropic => {
                let e = self.material.e.unwrap();
                let nu = self.material.nu.unwrap();
                let d = e / (1.0 - nu * nu);

                SMatrix::<f64, 3, 3>::from_row_slice(&[
                    d * factor,       d * nu * factor,  0.0,
                    d * nu * factor,  d * factor,       0.0,
                    0.0,              0.0,              d * (1.0 - nu) / 2.0 * factor,
                ])
            }
            MaterialType::Orthotropic => {
                let e1 = self.material.e1.unwrap();
                let e2 = self.material.e2.unwrap();
                let g12 = self.material.g12.unwrap();
                let nu12 = self.material.nu12.unwrap();
                let nu21 = nu12 * e2 / e1;
                let denom = 1.0 - nu12 * nu21;

                SMatrix::<f64, 3, 3>::from_row_slice(&[
                    e1 / denom * factor,          nu21 * e1 / denom * factor,  0.0,
                    nu12 * e2 / denom * factor,   e2 / denom * factor,         0.0,
                    0.0,                          0.0,                         g12 * factor,
                ])
            }
        }
    }

    fn bending_b_matrix(&self, x: &[f64; 3], y: &[f64; 3], area: f64) -> SMatrix<f64, 3, 9> {
        // Simplified DKT B matrix (actual DKT requires more complex shape functions)
        // This is a placeholder; production code would use full DKT formulation from Batoz (1980)
        let mut b = SMatrix::<f64, 3, 9>::zeros();

        // Curvatures κ = [κx, κy, κxy]^T = B * [w1, θx1, θy1, w2, θx2, θy2, w3, θx3, θy3]^T
        // Placeholder: linear interpolation (oversimplified)
        let two_area = 2.0 * area;

        for i in 0..3 {
            let j = (i + 1) % 3;
            let k = (i + 2) % 3;

            b[(0, i*3+1)] = (y[j] - y[k]) / two_area;  // dθx/dx
            b[(1, i*3+2)] = (x[k] - x[j]) / two_area;  // dθy/dy
            b[(2, i*3+1)] = (x[k] - x[j]) / two_area;  // dθx/dy
            b[(2, i*3+2)] = (y[j] - y[k]) / two_area;  // dθy/dx
        }

        b
    }

    fn transformation_matrix(&self, e1: &Vector3, e2: &Vector3, e3: &Vector3) -> SMatrix<f64, 18, 18> {
        // Transformation from local to global: T such that u_global = T * u_local
        let mut t = SMatrix::<f64, 18, 18>::zeros();

        // Rotation matrix for each node (3 times, once per node)
        let r = SMatrix::<f64, 3, 3>::from_rows(&[
            e1.to_row(),
            e2.to_row(),
            e3.to_row(),
        ]);

        for i in 0..3 {
            let offset = i * 6;
            // Translational DOF
            for ii in 0..3 {
                for jj in 0..3 {
                    t[(offset + ii, offset + jj)] = r[(ii, jj)];
                }
            }
            // Rotational DOF
            for ii in 0..3 {
                for jj in 0..3 {
                    t[(offset + 3 + ii, offset + 3 + jj)] = r[(ii, jj)];
                }
            }
        }

        t
    }

    fn transform_to_local(&self, e1: &Vector3, e2: &Vector3, e3: &Vector3) -> [Vector3; 3] {
        let mut local = [Vector3::zeros(); 3];
        for i in 0..3 {
            local[i] = Vector3::new(
                self.coords[i].dot(e1),
                self.coords[i].dot(e2),
                self.coords[i].dot(e3),
            );
        }
        local
    }
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–4)

**Day 1: Project initialization and Rust backend scaffold**
```bash
cargo init aerostream-api
cd aerostream-api
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
- `migrations/001_initial.sql` — All 9 tables: users, organizations, org_members, projects, simulations, simulation_jobs, materials, load_cases, composite_layups, usage_records
- `src/db/mod.rs` — Database pool initialization
- `src/db/models.rs` — All SQLx structs with FromRow derives
- Run `sqlx migrate run` to apply schema
- Seed script for initial materials library (Al 2024-T3, Al 7075-T6, Ti-6Al-4V, 4130 Steel, T300/5208 carbon epoxy, E-glass/epoxy, etc.)

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

### Phase 2 — Geometry Import + Meshing (Days 5–10)

**Day 5: OpenCASCADE integration for STEP import**
- `geometry/` — New Rust workspace member
- Add `opencascade-sys` dependency (or build custom FFI bindings to OCCT C++)
- `geometry/src/step_import.rs` — STEP file reader, topology extraction (solids, shells, faces, edges)
- `geometry/src/types.rs` — Geometry data structures (Shape, TopoFace, TopoEdge, BoundingBox)
- Unit tests: import simple STEP files (cube, cylinder, sphere), validate topology counts

**Day 6: Surface triangulation for visualization**
- `geometry/src/tessellation.rs` — Surface mesh generation from CAD faces using OCCT IncrementalMesh
- Export triangulation as JSON for frontend (vertices, normals, indices)
- `src/api/handlers/geometry.rs` — POST /projects/:id/geometry/upload endpoint (accept STEP file, run import + tessellation, store in S3, return metadata)
- Tests: upload STEP, verify surface mesh quality (triangle count, aspect ratio)

**Day 7: Surface meshing for panel method**
- `mesher/` — New Rust workspace member
- `mesher/src/surface_mesh.rs` — Delaunay triangulation on parametric surfaces, advancing-front refinement
- `mesher/src/panel_generator.rs` — Convert surface mesh to quadrilateral panels for VLM (triangle→quad merging or direct quad meshing)
- Generate panel connectivity, compute control points (3/4-chord), normals, areas
- Tests: mesh simple wing (rectangular planform), verify panel count and quality

**Day 8: Volume meshing for FEA**
- `mesher/src/volume_mesh.rs` — 3D Delaunay triangulation with boundary recovery (Tetgen-style algorithm in Rust)
- `mesher/src/boundary_layer.rs` — Prism layer insertion for boundary layer refinement (anisotropic elements near walls)
- Mesh quality metrics: min/max angle, aspect ratio, skewness
- Tests: mesh airfoil section with boundary layer, mesh wing box structure

**Day 9: HDF5 mesh export**
- Add `hdf5` dependency
- `mesher/src/hdf5_writer.rs` — Write mesh to HDF5 format (nodes, elements, element types, boundary regions)
- Schema: `/nodes` (Nx3 array), `/elements` (MxK array), `/element_types`, `/regions`
- `src/api/handlers/mesh.rs` — POST /projects/:id/mesh endpoint (generate mesh from geometry, upload HDF5 to S3, return stats)
- Tests: generate mesh, write HDF5, read back and verify integrity

**Day 10: Mesh visualization data preparation**
- `mesher/src/viz_export.rs` — Export mesh to Three.js-compatible format (BufferGeometry JSON)
- Generate edge lists for wireframe rendering
- `src/api/handlers/mesh.rs` — GET /projects/:id/mesh/viz endpoint (return Three.js mesh data)
- Tests: export mesh for visualization, verify format compatibility with Three.js

### Phase 3 — Panel Method Solver (Days 11–15)

**Day 11: VLM solver core**
- `solver-panel/` — New Rust workspace member (shared between native and WASM)
- `solver-panel/src/vlm.rs` — VLMSolver struct, AIC matrix build, Biot-Savart vortex filament velocity
- Implement vortex ring influence coefficient calculation
- Linear algebra via `nalgebra` (dense matrices for small panel counts)
- Unit tests: single panel, 2-panel wing, verify symmetry

**Day 12: Force and moment integration**
- `solver-panel/src/forces.rs` — Pressure coefficient computation, force integration, non-dimensionalization
- Compute CL, CD (induced drag via Trefftz plane), CM, CY, CN
- Reference area, chord, span from parameters
- Tests: flat plate at AOA (compare CL to theory: CL = 2π·α), simple wing (compare to analytical AR correction)

**Day 13: Stability derivatives via finite difference**
- `solver-panel/src/derivatives.rs` — Compute Clα, Cmα, Cnβ, etc. by perturbing AOA/AOS
- Finite difference with δα = 0.5° perturbation
- Tests: symmetric wing (Cmα should be negative for stable), verify Clα ≈ 2π for high AR

**Day 14: WASM compilation of panel solver**
- `solver-panel-wasm/` — WASM wrapper for panel solver
- Add `wasm-bindgen`, `wasm-pack` dependencies
- `solver-panel-wasm/src/lib.rs` — WASM bindings: `solve_panel_aero(panels, params) → AeroCoefficients`
- `wasm-pack build --target web --release`
- GitHub Actions workflow for WASM build + upload to S3/CDN
- Tests: invoke WASM solver from Node.js, verify results match native

**Day 15: Panel solver API integration**
- `src/api/handlers/aero.rs` — POST /projects/:id/simulations/panel endpoint
- For ≤200 panels: return "use WASM" response with panel data
- For >200 panels: enqueue server-side panel job (Redis queue)
- `src/workers/panel_worker.rs` — Background worker for server-side panel solves
- Tests: submit panel analysis, verify results (CL, CD, CM), check WASM threshold logic

### Phase 4 — FEA Solver Core (Days 16–22)

**Day 16: FEA solver scaffold + shell element**
- `solver-fea/` — New Rust workspace member (native only, not WASM)
- `solver-fea/src/elements/dkt_shell.rs` — DKT shell element stiffness matrix (membrane + bending)
- Isotropic material matrix (plane stress for membrane, plate bending)
- Unit tests: single element patch test (uniform tension, pure bending), verify stiffness symmetry

**Day 17: Global assembly and sparse matrix storage**
- `solver-fea/src/assembly.rs` — Assemble global stiffness matrix in COO format, convert to CSR
- `solver-fea/src/sparse.rs` — Sparse matrix utilities (COO → CSR conversion, pattern optimization)
- DOF map: node ID → global DOF indices (6 DOF per node: ux, uy, uz, rx, ry, rz)
- Tests: assemble 2-element mesh, verify matrix pattern and symmetry

**Day 18: MUMPS sparse solver integration**
- Add `mumps-sys` dependency (FFI bindings to MUMPS Fortran library)
- `solver-fea/src/solver.rs` — Sparse direct solve via MUMPS: K*u = F
- Apply boundary conditions via penalty method or direct DOF elimination
- Tests: solve cantilever beam (tip load), compare displacement to analytical solution

**Day 19: Stress recovery and failure analysis**
- `solver-fea/src/postprocess/stress.rs` — Compute element strains and stresses from nodal displacements
- Shell stress: top/bottom surface stresses, membrane + bending contribution
- Von Mises stress for isotropic materials
- Tests: uniform pressure on plate, verify stress distribution

**Day 20: Orthotropic materials and composite layups**
- `solver-fea/src/materials/orthotropic.rs` — Orthotropic material matrix (E1, E2, G12, ν12)
- `solver-fea/src/composite/laminate.rs` — Classical Lamination Theory (CLT): compute [A, B, D] matrices from ply stack
- Ply-level stress recovery: transform element strains to ply coordinates, compute ply stresses
- Tests: single ply at 45°, symmetric layup [0/90/90/0], verify ABD matrix properties

**Day 21: Composite failure criteria**
- `solver-fea/src/composite/failure.rs` — Tsai-Wu failure criterion evaluation per ply
- Reserve factor (RF) computation: load multiplier to failure
- First-ply failure (FPF) identification
- Tests: uniaxial tension on [0]₄ vs [±45]₂s, verify failure modes and RF

**Day 22: FEA worker integration**
- `src/workers/fea_worker.rs` — Background worker for server-side FEA jobs
- Pull from Redis queue, run solver, write HDF5 results to S3, update database
- `src/api/handlers/fea.rs` — POST /projects/:id/simulations/fea endpoint
- WebSocket progress: meshing → solving → postprocessing
- Tests: submit static FEA, monitor progress, verify results (displacement, stress)

### Phase 5 — Frontend 3D Visualization (Days 23–28)

**Day 23: React app scaffold + Three.js scene**
- `frontend/` — Vite + React + TypeScript
- Install: `three`, `@react-three/fiber`, `@react-three/drei`, `zustand`
- `src/components/Viewer3D.tsx` — Three.js canvas with OrbitControls, lighting
- Load geometry mesh (BufferGeometry from API), render with edges
- Tests: render test geometry, verify camera controls

**Day 24: STEP geometry visualization**
- `src/components/GeometryViewer.tsx` — Display imported STEP geometry as triangulated surface
- Material: two-sided, edges visible, transparency toggle
- Bounding box display, axes helper
- Tests: load wing geometry, verify rendering, test pan/zoom/rotate

**Day 25: Mesh visualization (surface and volume)**
- `src/components/MeshViewer.tsx` — Display generated mesh (surface or volume)
- Wireframe mode toggle, surface/solid rendering
- Color-code elements by quality metric (aspect ratio, skewness)
- Tests: load FEA mesh, verify element visibility, test quality coloring

**Day 26: Pressure coefficient contour plots (panel aero)**
- `src/components/AeroResultsViewer.tsx` — Load panel aero results, map Cp to vertex colors
- Custom fragment shader for smooth color gradients (interpolate Cp values on GPU)
- Colormap: blue (high pressure) → white → red (low pressure)
- Legend with min/max Cp values
- Tests: visualize Cp distribution on wing, verify color mapping

**Day 27: Stress contour plots (FEA)**
- `src/components/FEAResultsViewer.tsx` — Load FEA results from HDF5, map stress to vertex colors
- Von Mises stress, principal stresses, or displacement magnitude
- Deformation overlay: exaggerated displacement (scale factor slider)
- Legend with min/max stress
- Tests: visualize stress on loaded structure, verify deformation animation

**Day 28: WebGPU optimization for large meshes**
- Migrate from WebGL to WebGPU for 100K+ element meshes
- GPU compute shader for contour value interpolation
- Level-of-detail (LOD) for large meshes: coarsen distant elements
- Performance test: render 200K element mesh at 60fps

### Phase 6 — Billing + Monitoring (Days 29–32)

**Day 29: Stripe integration**
- Add `stripe-rust` dependency
- `src/billing/mod.rs` — Stripe client initialization
- `src/api/handlers/billing.rs` — Checkout session, customer portal, webhook handler
- Plan limits enforcement: check DOF count, panel count, queue priority
- Tests: create subscription, upgrade/downgrade, cancel

**Day 30: Usage tracking**
- `src/billing/usage.rs` — Track compute minutes, storage, API calls
- Insert usage_records on simulation completion
- Compute cost estimation: $0.05/core-hour for FEA, $0.01/1000 panels for server panel
- Tests: run simulation, verify usage record creation, compute cost

**Day 31: Prometheus metrics**
- Add `prometheus`, `axum-prometheus` dependencies
- `src/metrics.rs` — Expose /metrics endpoint
- Track: request count, latency, active jobs, solver iterations, memory usage
- Grafana dashboard JSON for visualization
- Tests: query metrics endpoint, verify counter increments

**Day 32: Sentry error tracking**
- Add `sentry` dependency
- `src/main.rs` — Initialize Sentry with DSN from environment
- Capture errors in API handlers, solver failures, worker crashes
- Tests: trigger error, verify Sentry event

### Phase 7 — Load Cases + Reports (Days 33–38)

**Day 33: Load case editor (backend)**
- `src/api/handlers/load_cases.rs` — CRUD endpoints for load cases
- Boundary condition types: fixed, pinned, roller, prescribed displacement
- Applied load types: point force, distributed pressure, gravity, thermal
- Tests: create load case, apply to simulation

**Day 34: Load case editor (frontend)**
- `src/components/LoadCaseEditor.tsx` — UI for defining BCs and loads
- Click on mesh regions to assign BCs (fixed, force, pressure)
- Visual feedback: color-code BCs (red = fixed, blue = force, green = pressure)
- Tests: create load case via UI, submit to API

**Day 35: V-n diagram generation**
- `src/analysis/vn_diagram.rs` — Generate V-n diagram from aircraft parameters (weight, wing area, CLmax)
- Maneuver envelope: +n_max, -n_min vs airspeed
- Gust envelope: FAR 25.341 discrete gust analysis
- Output: JSON with envelope points
- Tests: generate V-n diagram for sample aircraft, verify corner speeds

**Day 36: Python FastAPI for post-processing**
- `postprocess/` — Python 3.12 + FastAPI service
- `postprocess/app/hdf5_reader.py` — Read HDF5 results from S3
- `postprocess/app/report_gen.py` — Generate PDF analysis report (ReportLab or WeasyPrint)
- Endpoints: POST /reports/loads, POST /reports/stress
- Tests: request report, verify PDF generation

**Day 37: Analysis report templates**
- `postprocess/templates/loads_report.html` — HTML template for loads report (V-n diagram, load case summary, critical loads table)
- `postprocess/templates/stress_report.html` — Stress report (margins of safety, failure modes, material properties)
- Include: project metadata, analysis date, solver version, disclaimer
- Tests: generate report, verify all sections populated

**Day 38: Report download and storage**
- `src/api/handlers/reports.rs` — POST /projects/:id/reports/generate endpoint
- Call Python FastAPI postprocess service, upload PDF to S3, return URL
- Frontend: download button for reports
- Tests: generate report, download PDF, verify content

### Phase 8 — Testing + Polish (Days 39–42)

**Day 39: Solver validation benchmarks**
- **Panel method**: NACA 0012 airfoil (2D data from XFOIL), 3D rectangular wing (compare to Prandtl lifting-line theory)
  - Target: CL within 5% of reference, Clα within 10%
- **FEA isotropic**: Cantilever beam (analytical solution), pressurized cylinder (Lamé solution)
  - Target: displacement within 2%, stress within 5%
- **FEA composite**: [0/90]s plate under tension (CLT analytical), quasi-isotropic [0/±45/90]s (compare to test data)
  - Target: stiffness within 5%, first-ply failure load within 10%

**Day 40: Performance optimization**
- Profile solver with `cargo flamegraph`
- Optimize: matrix assembly (parallel element loop), MUMPS parameters (ordering, factorization)
- WASM size optimization: `wasm-opt -Oz`
- Database query optimization: add indexes, batch inserts for usage records
- Target: WASM panel solve <5s for 200 panels, FEA solve <60s for 10K DOF

**Day 41: End-to-end integration tests**
- Test flow: Register → Upload STEP → Generate mesh → Run panel aero → Visualize Cp
- Test flow: Register → Upload STEP → Generate mesh → Define load case → Run FEA → Visualize stress → Generate report
- Test: upgrade subscription, verify increased DOF limit
- Test: concurrent job queue, priority enforcement
- CI: GitHub Actions runs all tests on PR

**Day 42: Documentation + deployment**
- User docs: Getting started, STEP import, running simulations, interpreting results
- API docs: OpenAPI spec via `utoipa`
- Deployment: AWS ECS/Fargate (API + workers), RDS PostgreSQL, ElastiCache Redis, S3
- Terraform/CDK infrastructure-as-code
- DNS + SSL via Route53 + ACM
- Launch on `aerostream.app` domain

---

## Validation Benchmarks

### 1. Panel Method — NACA 0012 Airfoil (2D Validation)

**Objective**: Validate VLM implementation against XFOIL 2D airfoil data for a NACA 0012 symmetric airfoil.

**Setup**:
- NACA 0012 airfoil extruded to create a high-aspect-ratio rectangular wing (AR = 20 to approximate 2D)
- Freestream velocity: 50 m/s
- Altitude: sea level (ρ = 1.225 kg/m³)
- Angle of attack sweep: -5° to 15° in 1° increments
- Panel count: 120 (chordwise) × 4 (spanwise) = 480 panels

**Expected Results** (from XFOIL):
- Clα = 6.28 per radian (2π for thin airfoil theory, corrected for thickness)
- CL at α=0° ≈ 0.0 (symmetric airfoil)
- CL at α=5° ≈ 0.548

**Acceptance Criteria**:
- Clα within 10% of XFOIL (VLM target: 5.65-6.91 per rad)
- CL(α=5°) within 5% of XFOIL (VLM target: 0.521-0.575)

### 2. Panel Method — Rectangular Wing (3D Validation)

**Objective**: Validate 3D wing aerodynamics against Prandtl lifting-line theory.

**Setup**:
- Rectangular wing: span = 10m, chord = 1m (AR = 10)
- NACA 0012 airfoil section (2D Clα = 6.28/rad)
- Freestream: 50 m/s, α = 5°
- Panel count: 40 (chordwise) × 20 (spanwise) = 800 panels

**Expected Results** (Prandtl lifting-line):
- 3D lift curve slope: Clα_3D = Clα_2D / (1 + Clα_2D / (π·AR)) = 6.28 / (1 + 6.28/(π·10)) = 4.52 per rad
- CL at α=5° (0.0873 rad): CL = 4.52 × 0.0873 = 0.395
- Induced drag coefficient: CDi = CL² / (π·AR) = 0.395² / (π·10) = 0.00497

**Acceptance Criteria**:
- CL within 8% of lifting-line (VLM target: 0.364-0.426)
- CDi within 15% of lifting-line (VLM target: 0.00422-0.00572)

### 3. FEA Isotropic — Cantilever Beam

**Objective**: Validate linear static FEA solver for isotropic materials.

**Setup**:
- Cantilever beam: length L = 1m, width b = 0.1m, thickness t = 0.01m
- Material: Aluminum 2024-T3 (E = 73.1 GPa, ν = 0.33)
- Load: point force P = 1000 N at free end (downward)
- Mesh: 100 DKT shell elements (10 along length × 10 along width)

**Expected Results** (Euler-Bernoulli beam theory):
- Tip deflection: δ = P·L³ / (3·E·I) where I = b·t³/12 = 8.33e-9 m⁴
- δ = 1000 × 1³ / (3 × 73.1e9 × 8.33e-9) = 5.48 mm
- Maximum stress (at fixed end, top surface): σ = M·c / I = (P·L)·(t/2) / I = 600 MPa

**Acceptance Criteria**:
- Tip deflection within 2% of analytical (FEA target: 5.37-5.59 mm)
- Max stress within 5% of analytical (FEA target: 570-630 MPa)

### 4. FEA Composite — Cross-Ply Laminate [0/90]s

**Objective**: Validate composite laminate stiffness and failure prediction.

**Setup**:
- Flat plate: 0.2m × 0.2m
- Layup: [0/90/90/0] symmetric, 4 plies, each 0.125 mm thick (total t = 0.5 mm)
- Material: T300/5208 carbon/epoxy (E1 = 181 GPa, E2 = 10.3 GPa, G12 = 7.17 GPa, ν12 = 0.28)
- Strengths: Xt = 1500 MPa, Xc = 1500 MPa, Yt = 40 MPa, Yc = 246 MPa, S12 = 68 MPa
- Load: uniform in-plane tension σx = 100 MPa
- Mesh: 400 elements (20×20)

**Expected Results** (Classical Lamination Theory):
- Effective modulus Ex (for [0/90]s): Ex_eff ≈ 48 GPa (from ABD matrix inversion)
- Strain: εx = σx / Ex_eff = 100e6 / 48e9 = 2.08e-3
- Ply stresses (0° plies): σ1 ≈ 150 MPa, σ2 ≈ 0.5 MPa
- Tsai-Wu failure index (0° plies): FI ≈ 0.01 (safe, RF ≈ 10)
- First-ply failure: 90° plies fail first in transverse direction at σx ≈ 200 MPa (RF ≈ 2.0)

**Acceptance Criteria**:
- Ex_eff within 5% of CLT (FEA target: 45.6-50.4 GPa)
- Ply stresses within 5% of CLT (σ1 target: 142.5-157.5 MPa)
- RF within 10% of CLT (FEA target: 1.8-2.2)

### 5. FEA Composite — Quasi-Isotropic [0/±45/90]s

**Objective**: Validate quasi-isotropic laminate behavior.

**Setup**:
- Flat plate: 0.2m × 0.2m
- Layup: [0/45/-45/90/90/-45/45/0] symmetric, 8 plies, each 0.125 mm (total t = 1.0 mm)
- Material: same T300/5208 as above
- Load: biaxial tension σx = σy = 50 MPa
- Mesh: 400 elements (20×20)

**Expected Results**:
- Quasi-isotropic stiffness: Ex ≈ Ey ≈ 54 GPa (theoretical for this layup)
- Strains: εx ≈ εy ≈ 50e6 / 54e9 = 9.26e-4
- All plies carry similar load due to balanced angles
- First-ply failure: approximately σx = σy = 250 MPa (RF ≈ 5.0)

**Acceptance Criteria**:
- Ex, Ey within 5% of theoretical (target: 51.3-56.7 GPa)
- Failure load within 10% (RF target: 4.5-5.5)

---

## Post-MVP Roadmap (v1.1 – v2.0)

### v1.1 — RANS CFD (Weeks 13-16)
- Custom RANS solver with Spalart-Allmaras and k-ω SST turbulence models
- GPU acceleration via CUDA/ROCm for pressure Poisson solve
- Automated volume meshing with boundary layer prism layers
- Transition SST model for laminar-turbulent transition prediction
- Streamline visualization, Y+ contours, turbulence quantities

### v1.2 — Static Aeroelasticity (Weeks 17-20)
- Aero-structural coupling: surface pressure → FEA load, FEA deformation → CFD mesh update
- Fixed-point iteration with under-relaxation for convergence
- Jig shape computation (inverse problem): target in-flight shape → unloaded manufacturing shape
- Trim analysis: find control surface deflections for equilibrium flight

### v1.3 — Modal and Flutter Analysis (Weeks 21-24)
- Modal analysis: eigenvalue solver for natural frequencies and mode shapes (ARPACK via Rust bindings)
- Flutter analysis: V-g and V-f methods with matched-point interpolation
- Gust response: discrete gust (FAR 25.341), continuous turbulence (PSD-based)
- Control surface effectiveness and reversal speed prediction

### v1.4 — Optimization (Weeks 25-28)
- Wing planform optimization: minimize drag subject to lift constraint (gradient-free: genetic algorithm, gradient-based: SQP)
- Structural sizing optimization: minimize mass subject to strength, stiffness, buckling constraints
- Aero-structural MDO: coupled optimization of wing shape and structure
- Topology optimization for brackets and fittings (SIMP method)
- Design of experiments (DOE): Latin hypercube sampling, response surface models

### v1.5 — Certification Documentation (Weeks 29-32)
- Auto-generated loads report: V-n diagram, load case matrix, critical load identification, margin summary
- Stress report: stress tables, failure mode summary, margins of safety per FAR/CS 25.613
- Material property documentation: A/B-basis allowables, source traceability
- Configuration management: link analysis results to specific CAD revisions
- Digital signature workflow for DER approval

### v2.0 — Fatigue and Damage Tolerance (Weeks 33-40)
- Fatigue analysis: S-N curves, Miner's rule cumulative damage, load spectra from flight profiles
- Crack growth: NASGRO/AFGROW integration, stress intensity factors, da/dN vs ΔK
- Residual strength: progressive failure analysis with damaged elements
- Inspection interval prediction per FAA AC 25.571-1D
- Probabilistic analysis: Monte Carlo for material property variability
