# 65. WeldSim — Welding Process Simulation and Qualification Platform

## Implementation Plan

**MVP Scope:** Browser-based weld geometry editor with drag-and-drop bead placement, weld path sequencing, and clamping definition rendered via Three.js, custom thermal-mechanical FEM solver implementing transient 3D heat transfer with Goldak double-ellipsoid moving heat source compiled to server-side Rust with parallel assembly/solve using rayon, support for single-pass arc welding (GMAW/GTAW/SAW) with temperature-dependent material properties (thermal conductivity, specific heat, density, yield strength) for 3 steel grades (mild steel, 304 stainless, low-alloy steel), thermo-elastic-plastic mechanical analysis with phase transformation plasticity (austenite→martensite), residual stress and distortion prediction with von Mises stress and displacement contour visualization via WebGL, automatic mesh generation with bead-path refinement and heat-affected zone (HAZ) adaptive refinement, PostgreSQL storage for projects/materials/simulations, S3 storage for mesh and result files, basic PDF report generation with temperature history plots, residual stress profiles, and distortion magnitude summary, Stripe billing with three tiers (Free / Pro $199/mo / Advanced $399/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| FEM Solver | Rust (native server-side) | Custom thermal-mechanical solver with rayon parallelization |
| Thermal Solver Core | Rust + `nalgebra-sparse` | 3D transient heat transfer with moving heat source, Crank-Nicolson scheme |
| Mechanical Solver | Rust + `faer` sparse | Thermo-elastic-plastic with transformation strain, Newton-Raphson |
| Mesh Generation | Python 3.12 (gmsh via API) | Tetrahedral/hexahedral meshing with adaptive refinement |
| Material Database | Python (FastAPI) | CCT/TTT diagram interpolation, phase transformation kinetics |
| Database | PostgreSQL 16 | Projects, users, simulations, materials, weld procedures |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | Mesh files (.msh), result files (.vtu VTK), weld procedure PDFs |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| 3D Visualization | Three.js + React Three Fiber | Weld geometry editor, mesh visualization, stress contour rendering |
| Contour Rendering | WebGL 2.0 (custom shaders) | GPU-accelerated stress/temperature field rendering on deformed mesh |
| Real-time | WebSocket (Axum) | Live simulation progress, convergence monitoring |
| Job Queue | Redis 7 + Tokio tasks | Simulation job management, priority queue for paying users |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | Static frontend assets, material property libraries |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, solver performance tracking |
| CI/CD | GitHub Actions | Rust tests, Python service tests, Docker image push |

### Key Architecture Decisions

1. **Server-side FEM only (no WASM for welding simulation)**: Unlike circuit simulation where small problems fit in browsers, welding FEM requires 10K-500K DOF (degrees of freedom) for meaningful accuracy. A single-pass weld on a 200mm × 100mm plate requires ~50K tetrahedral elements (150K DOF thermal, 450K DOF mechanical) with 1000+ timesteps (10-30 minutes solve time on 8-core server). WASM is unsuitable for this scale. All simulation runs on server-side Rust with rayon-parallelized matrix assembly and sparse direct solvers (MUMPS bindings or `faer` native).

2. **Decoupled thermal-mechanical analysis with transformation plasticity**: The MVP implements sequentially coupled analysis: (1) thermal solve with Goldak heat source → temperature history at each node, (2) metallurgical phase transformation (austenite → martensite fraction) based on temperature-time history and CCT diagrams, (3) mechanical solve with thermal strain + transformation strain + plastic strain. Full thermomechanical coupling (where deformation affects heat transfer) is deferred to post-MVP. This reduces complexity while capturing 90% of residual stress and distortion physics.

3. **Goldak double-ellipsoid as primary heat source model**: The Goldak model (two semi-ellipsoids, front/rear power distribution) is the industry standard for arc welding simulation, validated in 1000+ papers. Heat flux is applied as volumetric source term moving along weld path with user-defined travel speed. Parameters (a, b, c_front, c_rear, η_efficiency) are material- and process-dependent; MVP provides calibrated defaults for GMAW on steel (η=0.8, a=3mm, b=3mm, c=6mm) with manual override.

4. **Three.js for 3D geometry editing and result visualization**: Three.js provides GPU-accelerated rendering for complex weld geometry (multi-bead, fillet welds, T-joints, butt joints) and stress contour overlays on deformed meshes. Users define base metal geometry (boxes, cylinders, imported STL), place weld beads along paths, and set clamping locations interactively. Results are visualized as vertex-colored meshes with custom WebGL shaders for smooth contour interpolation and deformation scaling.

5. **Python gmsh service for automatic mesh generation with HAZ refinement**: Mesh generation is computationally distinct from solving and benefits from gmsh's robust geometry kernel. A Python FastAPI service receives weld geometry + bead paths, generates tetrahedral mesh with refinement zones (2mm elements near weld, 10mm in far field), and returns .msh file to S3. Rust solver loads mesh via `meshio` Rust bindings or direct .msh parser. Adaptive remeshing during solve (for multi-pass post-MVP) will use this same service.

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
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
CREATE INDEX users_stripe_idx ON users(stripe_customer_id);

-- Organizations (for team accounts)
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
    role TEXT NOT NULL DEFAULT 'member',
    joined_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(org_id, user_id)
);
CREATE INDEX org_members_org_idx ON org_members(org_id);
CREATE INDEX org_members_user_idx ON org_members(user_id);

-- Projects (weld assembly + simulation workspace)
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    geometry_data JSONB NOT NULL DEFAULT '{}',  -- Base metal parts, weld bead paths, clamps
    settings JSONB DEFAULT '{}',  -- Units, mesh density, solver tolerances
    is_public BOOLEAN NOT NULL DEFAULT false,
    forked_from UUID REFERENCES projects(id) ON DELETE SET NULL,
    thumbnail_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);
CREATE INDEX projects_org_idx ON projects(org_id);
CREATE INDEX projects_updated_idx ON projects(updated_at DESC);

-- Materials (steel grades, aluminum, titanium with thermal/mechanical/metallurgical properties)
CREATE TABLE materials (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,  -- e.g., "Mild Steel A36", "SS304", "A516 Gr70"
    category TEXT NOT NULL,  -- carbon_steel | stainless_steel | aluminum | titanium | nickel_alloy
    standard TEXT,  -- e.g., "ASTM A36", "EN 10088-2"
    density REAL NOT NULL,  -- kg/m³
    thermal_properties JSONB NOT NULL,  -- [{T_celsius, k_conductivity_W_mK, cp_specific_heat_J_kgK}]
    mechanical_properties JSONB NOT NULL,  -- [{T_celsius, E_youngs_GPa, nu_poisson, sigma_y_yield_MPa, alpha_thermal_expansion_1K}]
    transformation_properties JSONB,  -- CCT/TTT data: {T_austenitization, phases: [{name, start_temp, end_temp, kinetics}]}
    is_builtin BOOLEAN DEFAULT true,
    uploaded_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX materials_category_idx ON materials(category);
CREATE INDEX materials_name_trgm_idx ON materials USING gin(name gin_trgm_ops);

-- Simulations
CREATE TABLE simulations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    analysis_type TEXT NOT NULL,  -- thermal_only | thermal_mechanical | thermal_mechanical_metallurgical
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | running | completed | failed | cancelled
    weld_process TEXT NOT NULL,  -- gmaw | gtaw | saw | laser | rsw | fsw
    material_id UUID NOT NULL REFERENCES materials(id),
    parameters JSONB NOT NULL DEFAULT '{}',  -- Process params: current, voltage, travel_speed, heat_input
    heat_source_params JSONB NOT NULL DEFAULT '{}',  -- Goldak a, b, cf, cr, efficiency
    mesh_url TEXT,  -- S3 URL for .msh file
    element_count INTEGER DEFAULT 0,
    node_count INTEGER DEFAULT 0,
    timesteps INTEGER DEFAULT 0,
    results_thermal_url TEXT,  -- S3 URL for thermal results .vtu
    results_mechanical_url TEXT,  -- S3 URL for mechanical results .vtu
    results_summary JSONB,  -- Quick-access: peak_temp, max_stress_MPa, max_displacement_mm
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

-- Simulation Jobs (server-side execution queue)
CREATE TABLE simulation_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    simulation_id UUID NOT NULL REFERENCES simulations(id) ON DELETE CASCADE,
    worker_id TEXT,
    cores_allocated INTEGER DEFAULT 8,
    memory_mb INTEGER DEFAULT 16384,
    priority INTEGER DEFAULT 0,
    progress_pct REAL DEFAULT 0.0,
    current_timestep INTEGER DEFAULT 0,
    convergence_data JSONB DEFAULT '[]',  -- [{timestep, thermal_residual, mechanical_residual, nr_iterations}]
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX jobs_sim_idx ON simulation_jobs(simulation_id);
CREATE INDEX jobs_worker_idx ON simulation_jobs(worker_id);
CREATE INDEX jobs_priority_idx ON simulation_jobs(priority DESC, created_at ASC);

-- Weld Procedures (for qualification tracking, post-MVP WPS generation)
CREATE TABLE weld_procedures (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    simulation_id UUID REFERENCES simulations(id) ON DELETE SET NULL,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    procedure_number TEXT,
    weld_process TEXT NOT NULL,
    base_material_id UUID REFERENCES materials(id),
    filler_material TEXT,
    code_standard TEXT,  -- AWS_D1.1 | ASME_IX | EN_ISO_15614
    parameters JSONB NOT NULL,
    qualification_status TEXT DEFAULT 'pending',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX weld_procedures_user_idx ON weld_procedures(user_id);

-- Usage Tracking
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,  -- simulation_hours | mesh_generation | storage_gb
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
pub struct Material {
    pub id: Uuid,
    pub name: String,
    pub category: String,
    pub standard: Option<String>,
    pub density: f32,
    pub thermal_properties: serde_json::Value,
    pub mechanical_properties: serde_json::Value,
    pub transformation_properties: Option<serde_json::Value>,
    pub is_builtin: bool,
    pub uploaded_by: Option<Uuid>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Simulation {
    pub id: Uuid,
    pub project_id: Uuid,
    pub user_id: Uuid,
    pub analysis_type: String,
    pub status: String,
    pub weld_process: String,
    pub material_id: Uuid,
    pub parameters: serde_json::Value,
    pub heat_source_params: serde_json::Value,
    pub mesh_url: Option<String>,
    pub element_count: i32,
    pub node_count: i32,
    pub timesteps: i32,
    pub results_thermal_url: Option<String>,
    pub results_mechanical_url: Option<String>,
    pub results_summary: Option<serde_json::Value>,
    pub error_message: Option<String>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub wall_time_ms: Option<i32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct SimulationParams {
    pub analysis_type: AnalysisType,
    pub weld_process: WeldProcess,
    pub material_id: Uuid,
    pub current_amps: f64,
    pub voltage_volts: f64,
    pub travel_speed_mm_s: f64,
    pub heat_input_j_mm: Option<f64>,
    pub preheat_temp_c: f64,
    pub interpass_temp_c: Option<f64>,
    pub ambient_temp_c: f64,
    pub heat_source: HeatSourceParams,
    pub solver_options: SolverOptions,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct HeatSourceParams {
    pub model: String,
    pub efficiency: f64,
    pub a_mm: f64,
    pub b_mm: f64,
    pub c_front_mm: f64,
    pub c_rear_mm: f64,
    pub power_front_fraction: f64,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct SolverOptions {
    pub thermal_timestep_s: f64,
    pub mechanical_timestep_s: f64,
    pub thermal_tolerance: f64,
    pub mechanical_tolerance: f64,
    pub max_newton_iterations: u32,
    pub cooling_end_temp_c: f64,
}
```

---

## Solver Architecture Deep-Dive

### Governing Equations

WeldSim's FEM solver implements sequentially coupled **thermal-metallurgical-mechanical** analysis:

#### 1. Thermal Analysis (Transient Heat Transfer)

```
ρ(T) c_p(T) ∂T/∂t = ∇·[k(T) ∇T] + Q_weld(x,y,z,t)

Goldak double-ellipsoid heat source:
Q_f(x,y,z) = (6√3 f_f η V I) / (a b c_f π√π) × exp(-3x²/c_f² - 3y²/a² - 3z²/b²)
Q_r(x,y,z) = (6√3 f_r η V I) / (a b c_r π√π) × exp(-3x²/c_r² - 3y²/a² - 3z²/b²)

FEM discretization:
[C]{Ṫ} + [K]{T} = {F}

Time integration (Crank-Nicolson):
([C] + Δt/2 [K]){T^(n+1)} = ([C] - Δt/2 [K]){T^n} + Δt {F^(n+1/2)}
```

#### 2. Metallurgical Analysis (Phase Transformation)

```
JMAK kinetics for diffusional phases:
X_phase(t) = 1 - exp(-k(T) t^n)

Koistinen-Marburger for martensite:
X_martensite = (1 - X_diffusional) × [1 - exp(-α_KM (M_s - T))]

Transformation strain:
ε_transform = ΔV/V × X_martensite + (3/2) K X_dot_martensite σ_dev / σ_eq
```

#### 3. Mechanical Analysis (Thermo-Elastic-Plastic)

```
Total strain decomposition:
ε_total = ε_elastic + ε_plastic + ε_thermal + ε_transformation

Von Mises plasticity with isotropic hardening:
f = σ_eq - σ_y(T, ε_p) ≤ 0

Newton-Raphson iteration:
[K_tangent]{Δu} = {F_ext} - {F_int}
```

---

## Architecture Deep-Dives

### 1. Simulation API Handler (Rust/Axum)

```rust
// src/api/handlers/simulation.rs

pub async fn create_simulation(
    State(state): State<AppState>,
    claims: Claims,
    Path(project_id): Path<Uuid>,
    Json(req): Json<CreateSimulationRequest>,
) -> Result<impl IntoResponse, ApiError> {
    // 1. Verify project ownership
    let project = sqlx::query_as!(...)
        .fetch_optional(&state.db)
        .await?
        .ok_or(ApiError::NotFound("Project not found"))?;

    // 2. Check plan limits
    if user.plan == "free" {
        return Err(ApiError::PlanLimit("Simulation requires Pro plan"));
    }

    // 3. Request mesh generation from Python gmsh service
    let mesh_response = state.mesh_client
        .post(format!("{}/generate", state.config.mesh_service_url))
        .json(&mesh_req)
        .send()
        .await?;

    // 4. Upload mesh to S3
    state.s3_client.put_object()
        .bucket(&state.config.s3_bucket)
        .key(&mesh_key)
        .body(mesh_response.mesh_data.into())
        .send()
        .await?;

    // 5. Create simulation record and enqueue job
    let sim = sqlx::query_as!(...)
        .fetch_one(&state.db)
        .await?;

    state.redis
        .publish("welding:jobs", serde_json::to_string(&job.id)?)
        .await?;

    Ok((StatusCode::CREATED, Json(sim)))
}
```

### 2. Thermal Solver Core (Rust FEM)

```rust
// solver/src/thermal.rs

pub struct ThermalSolver {
    pub mesh: Mesh,
    pub material: ThermalMaterial,
    pub heat_source: GoldakHeatSource,
    pub capacitance: CsrMatrix<f64>,
    pub conductivity: CsrMatrix<f64>,
    pub temp_current: Vec<f64>,
    pub temp_history: Vec<Vec<f64>>,
}

impl ThermalSolver {
    pub fn solve_transient(
        &mut self,
        dt: f64,
        t_end: f64,
        cooling_end_temp: f64,
        progress_callback: impl Fn(f64, f64),
    ) -> Result<(), SolverError> {
        let mut time = 0.0;

        while time < t_end {
            let heat_source_pos = self.heat_source.position_at_time(time);
            self.assemble_matrices();
            let rhs = self.compute_rhs(dt, time, &heat_source_pos);

            // Solve ([C] + dt/2 [K]) T^(n+1) = ([C] - dt/2 [K]) T^n + dt F
            let lhs_matrix = &self.capacitance + &self.conductivity.clone().scale(dt / 2.0);
            self.temp_next = solve_sparse_system(&lhs_matrix, &rhs_vec)?;

            for i in 0..self.n_nodes {
                self.temp_history[i].push(self.temp_next[i]);
            }

            time += dt;
        }
        Ok(())
    }
}

impl GoldakHeatSource {
    pub fn evaluate(&self, point: &[f64; 3], arc_pos: &[f64; 3]) -> f64 {
        let dx = point[0] - arc_pos[0];
        let dy = point[1] - arc_pos[1];
        let dz = point[2] - arc_pos[2];

        let power = self.efficiency * self.voltage * self.current;
        let (c, f) = if dx >= 0.0 { (self.c_front, self.f_front) } else { (self.c_rear, self.f_rear) };

        let coeff = 6.0 * f.sqrt() * f * power / (self.a * self.b * c * PI * PI.sqrt());
        let exponent = -3.0 * (dx*dx / (c*c) + dy*dy / (self.a * self.a) + dz*dz / (self.b * self.b));

        coeff * exponent.exp()
    }
}
```

### 3. Three.js Weld Geometry Editor

```typescript
// frontend/src/components/WeldEditor/WeldScene.tsx

export function WeldScene() {
  const { geometryData, updateGeometry } = useProjectStore();
  const [selectedObject, setSelectedObject] = useState<string | null>(null);

  return (
    <Canvas camera={{ position: [500, 400, 500], fov: 50 }}>
      <color attach="background" args={['#1a1a1a']} />
      <ambientLight intensity={0.4} />
      <directionalLight position={[10, 10, 5]} intensity={0.8} castShadow />

      <Grid args={[1000, 1000]} cellSize={50} cellColor="#444" sectionColor="#666" />

      {baseMetals.map((metal) => (
        <BaseMetal key={metal.id} metal={metal} selected={selectedObject === metal.id} />
      ))}

      {weldBeads.map((bead) => (
        <WeldBeadViz key={bead.id} bead={bead} selected={selectedObject === bead.id} />
      ))}

      <OrbitControls makeDefault />
    </Canvas>
  );
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–5)

**Day 1: Project initialization**
- `cargo init weldsim-api && cargo add axum tokio serde sqlx uuid chrono aws-sdk-s3 redis`
- `src/main.rs` — Axum app scaffold with CORS, tracing, graceful shutdown
- `src/config.rs` — Environment-based configuration
- `src/state.rs` — AppState (PgPool, Redis, S3Client, mesh service HTTP client)
- `docker-compose.yml` — PostgreSQL, Redis, MinIO

**Day 2: Database schema**
- `migrations/001_initial.sql` — All 8 tables: users, organizations, org_members, projects, materials, simulations, simulation_jobs, weld_procedures, usage_records
- `src/db/models.rs` — SQLx structs with FromRow derives
- Run `sqlx migrate run`, seed 3 material records (Mild Steel, SS304, Low-Alloy)

**Day 3: Authentication**
- `src/auth/mod.rs` — JWT middleware, token generation/validation
- `src/auth/oauth.rs` — Google/GitHub OAuth handlers
- `src/api/handlers/auth.rs` — Register, login, OAuth callback, refresh, me endpoints
- Bcrypt password hashing (cost 12), JWT 24h access + 30d refresh

**Day 4: User and project CRUD**
- `src/api/handlers/users.rs` — Get profile, update, delete
- `src/api/handlers/projects.rs` — Create, list, get, update, delete, fork
- `src/api/router.rs` — Route definitions with auth middleware
- Integration tests for auth flow and project CRUD

**Day 5: Materials API**
- `src/api/handlers/materials.rs` — List materials (filter by category), get material with full property curves
- Material property JSON structure: thermal_properties = `[{T: 20, k: 50, cp: 450}, {T: 500, k: 35, cp: 600}, ...]`
- Material search with fuzzy matching on name/standard

### Phase 2 — Python Mesh Service (Days 6–9)

**Day 6: Gmsh service scaffold**
```bash
cd mesh-service
python -m venv venv && source venv/bin/activate
pip install fastapi uvicorn gmsh numpy
```
- `main.py` — FastAPI app with `/generate` POST endpoint
- `mesher/geometry.py` — Construct gmsh geometry from box/cylinder/STL base metals
- `mesher/weld_path.py` — Extrude weld bead along path, create volume for filler metal
- Dockerfile with gmsh installed via apt

**Day 7: Mesh refinement logic**
- `mesher/refinement.py` — Distance field from weld path, element size function (2mm near weld, 10mm far field, smooth transition)
- HAZ refinement zone: sphere of radius 20mm around weld centerline
- Tetrahedral mesh generation with `gmsh.model.mesh.generate(3)`
- Mesh quality checks: aspect ratio, min/max element volume

**Day 8: Mesh export and upload**
- `mesher/export.py` — Write gmsh .msh format (ASCII version 4.1)
- Return mesh_data as bytes, element_count, node_count
- Rust API uploads to S3: `s3://bucket/meshes/{project_id}/{uuid}.msh`
- Integration test: send geometry JSON → receive mesh → verify node/element counts

**Day 9: Mesh visualization preparation**
- `mesher/simplify.py` — Generate simplified surface mesh (STL) for frontend preview
- Surface mesh decimation to <50K triangles for WebGL rendering
- Return both full .msh (for solver) and simplified .stl (for preview)

### Phase 3 — Thermal FEM Solver (Days 10–16)

**Day 10: Mesh loader and data structures**
- `solver/src/mesh.rs` — Parse gmsh .msh file format
- `Mesh` struct: nodes (Vec of coords), elements (Vec of node_ids), node_tags
- Element types: tet4 (linear tetrahedron), hex8 (linear hexahedron)
- Mesh statistics: node count, element count, bounding box

**Day 11: Thermal solver foundation**
- `solver/src/thermal.rs` — ThermalSolver struct with mesh, material, heat source
- `solver/src/materials.rs` — ThermalMaterial with temperature-dependent property functions
- Implement property interpolation from JSON tables: `k(T)`, `c_p(T)`, `ρ(T)`
- Unit tests: property interpolation accuracy

**Day 12: Thermal matrix assembly**
- `solver/src/thermal.rs::assemble_matrices` — Parallel rayon iteration over elements
- Compute element capacitance matrix `C_elem = ∫ ρ c_p N^T N dV`
- Compute element conductivity matrix `K_elem = ∫ k ∇N^T ∇N dV`
- Assembly into global COO matrix, convert to CSR for solve
- Tests: single-element matrix correctness

**Day 13: Goldak heat source implementation**
- `solver/src/heat_source.rs` — GoldakHeatSource struct, `evaluate(point, arc_pos)` method
- Front/rear ellipsoid selection based on sign of dx
- Heat source position interpolation along weld path polyline
- Unit tests: verify integral of Q over volume ≈ η V I (power balance)

**Day 14: Thermal time integration**
- `solver/src/thermal.rs::solve_transient` — Crank-Nicolson time loop
- Assemble `([C] + Δt/2 [K])` LHS matrix, `([C] - Δt/2 [K]){T^n} + Δt {F}` RHS
- Sparse Cholesky solver via `faer::sparse::SymbolicCholesky` (or MUMPS bindings)
- Temperature history storage: `Vec<Vec<f64>>` (node × time)
- Tests: RC thermal analogy (1D rod with heat source), verify against analytical

**Day 15: Boundary conditions**
- Convection: add `h A` to diagonal of K, `h A T_amb` to RHS (surface elements)
- Radiation: linearize T⁴ term using previous iteration temperature (Picard iteration)
- Identify surface nodes/faces from mesh geometry
- Tests: verify steady-state temperature with convection matches analytical

**Day 16: Thermal solver integration**
- `solver/src/lib.rs` — Public API: `run_thermal_analysis(mesh, material, params) -> ThermalResult`
- ThermalResult: time vector, temperature history, peak temperatures, cooling rates
- Write results to VTK .vtu format (unstructured grid with point data)
- End-to-end test: load mesh, run thermal solve, verify peak temperature location

### Phase 4 — Mechanical FEM Solver (Days 17–22)

**Day 17: Mechanical solver foundation**
- `solver/src/mechanical.rs` — MechanicalSolver struct
- `solver/src/materials.rs` — MechanicalMaterial with E(T), ν(T), σ_y(T), α_thermal(T)
- Strain-displacement matrix B for tet4/hex8 elements
- Elastic constitutive matrix D (plane stress/strain or 3D)

**Day 18: Elastic stiffness assembly**
- `solver/src/mechanical.rs::assemble_stiffness` — Parallel assembly `K = ∫ B^T D B dV`
- Numerical integration: Gauss quadrature (1 point for tet4, 2×2×2 for hex8)
- Thermal strain: `ε_thermal = α(T - T_ref) I` (volumetric expansion)
- Load thermal strain into RHS as initial strain

**Day 19: Plasticity implementation**
- `solver/src/plasticity.rs` — Von Mises yield function, radial return algorithm
- Elastic predictor: `σ_trial = σ_prev + D : Δε`
- Plastic corrector: if `f(σ_trial) > 0`, return to yield surface
- Update plastic strain `ε_p` and equivalent plastic strain `ε_p_eq` for hardening
- Store stress and plastic strain at integration points

**Day 20: Transformation strain**
- `solver/src/metallurgy.rs` — Phase transformation post-processor
- For each node, extract T(t) history from thermal results
- Compute austenite fraction, martensite fraction via JMAK/KM models
- Volumetric transformation strain: `ε_tr = (V_martensite / V_austenite - 1) X_martensite I`
- Greenwood-Johnson TRIP: `ε_tr_plastic = K X_dot σ_dev / σ_eq`

**Day 21: Newton-Raphson mechanical solve**
- `solver/src/mechanical.rs::solve_nonlinear` — Newton loop for each load step
- Residual: `R = F_ext - F_int`, where `F_int = ∫ B^T σ dV`
- Tangent stiffness: `K_tangent = ∫ B^T D_ep B dV` (elastic-plastic tangent)
- Convergence check: `||R|| / ||F_ext|| < tol`
- Line search for robustness

**Day 22: Mechanical result output**
- Displacement field, von Mises stress, plastic strain, transformation strain
- Distortion calculation: displacement magnitude at free boundary nodes
- Residual stress profiles: longitudinal, transverse, through-thickness
- Write to VTK .vtu format with cell/point data
- End-to-end test: verify thermal expansion matches analytical for uniform heating

### Phase 5 — Frontend 3D Editor (Days 23–28)

**Day 23: Frontend scaffold**
```bash
npm create vite@latest frontend -- --template react-ts
cd frontend && npm i three @react-three/fiber @react-three/drei zustand axios
```
- `src/App.tsx` — Router, auth context, layout
- `src/stores/projectStore.ts` — Zustand store for project/geometry state
- `src/pages/Dashboard.tsx` — Project list
- `src/pages/Editor.tsx` — Main editor layout (3D view + sidebar)

**Day 24: Three.js scene and base metal editor**
- `src/components/WeldEditor/WeldScene.tsx` — Canvas with OrbitControls, Grid, lighting
- `src/components/WeldEditor/BaseMetal.tsx` — Render box/cylinder geometry, TransformControls for move/rotate
- Sidebar: Add Box (prompt length/width/thickness), Add Cylinder (radius/height)
- Geometry state sync: update projectStore.geometryData on transform

**Day 25: Weld bead path editor**
- `src/components/WeldEditor/WeldBeadEditor.tsx` — Click to place path points on base metal surface
- Raycasting to detect surface clicks, snap to geometry faces
- Path visualization: tube geometry along spline (CatmullRomCurve3)
- Edit path: drag points, insert/delete intermediate points

**Day 26: Clamp placement**
- `src/components/WeldEditor/ClampMarker.tsx` — Visual markers (spheres) for clamp locations
- Click geometry to place fixed constraint, spring constraint with stiffness input
- Clamp release time: optional parameter for unclamping during cooling

**Day 27: Simulation parameter panel**
- `src/components/SimulationPanel.tsx` — Form for weld process (GMAW/GTAW/SAW), material selection
- Process parameters: current (A), voltage (V), travel speed (mm/s)
- Solver settings: thermal timestep, mechanical timestep, cooling end temperature
- Heat source parameters: Goldak a, b, c_front, c_rear, efficiency (pre-filled defaults)

**Day 28: Mesh preview and simulation submission**
- Request mesh generation from API → display simplified STL in scene
- Mesh statistics overlay: node count, element count, estimated solve time
- "Run Simulation" button → POST `/api/projects/{id}/simulations`
- Loading state with mesh generation progress (if slow)

### Phase 6 — Worker + Results Visualization (Days 29–33)

**Day 29: Simulation worker**
- `src/workers/simulation_worker.rs` — Redis job consumer, picks up jobs by priority
- Load mesh from S3, load material properties, construct solver instances
- Run thermal solve → metallurgy → mechanical solve in sequence
- Stream progress via WebSocket: current timestep, percentage, residual norms
- Upload results (.vtu files) to S3, update database

**Day 30: WebSocket progress streaming**
- `src/api/ws/simulation_progress.rs` — WebSocket handler, subscribe to simulation ID
- Worker publishes progress to Redis pub/sub channel `simulation:{id}:progress`
- Frontend: `src/hooks/useSimulationProgress.ts` — React hook for WebSocket connection
- Progress bar in UI, real-time convergence plot

**Day 31: Result loader and VTK parsing**
- `src/lib/vtkLoader.ts` — Fetch .vtu file from S3 (presigned URL), parse XML
- Extract point coordinates, cell connectivity, point/cell data (temperature, stress, displacement)
- Convert to Three.js BufferGeometry with attributes for rendering

**Day 32: Stress contour visualization**
- `src/components/ResultsViewer/StressContours.tsx` — Custom vertex shader for smooth interpolation
- Color map: blue (low stress) → yellow → red (high stress)
- Deformed mesh: apply displacement field scaled by user-adjustable factor (1x, 10x, 100x)
- Legend with stress range, von Mises stress selected by default

**Day 33: Result plots and summary**
- `src/components/ResultsViewer/TemperaturePlot.tsx` — Time-temperature plot for selected nodes (Line chart via recharts or visx)
- `src/components/ResultsViewer/StressProfile.tsx` — Residual stress line plot along user-defined path
- Summary cards: peak temperature, max von Mises stress, max displacement
- Export: download .vtu files, export plots as PNG

### Phase 7 — Billing + Limits (Days 34–37)

**Day 34: Stripe integration**
- `src/api/handlers/billing.rs` — Create checkout session, customer portal session
- `src/api/handlers/webhooks/stripe.rs` — Handle subscription events
- Plan mapping: Free (blocked), Pro ($199/mo, 10 simulations/month, 8 cores), Advanced ($399/mo, unlimited, 16 cores)

**Day 35: Usage tracking**
- `src/services/usage.rs` — Track simulation wall time, insert usage_records after completion
- Usage dashboard API: current period usage, quota remaining
- Plan limits enforcement: check simulation count before enqueue

**Day 36: Billing UI**
- `frontend/src/pages/Billing.tsx` — Plan comparison, current usage meter
- Upgrade/downgrade via Stripe Customer Portal redirect
- Locked feature indicators (e.g., multi-pass welding, RSW modules)

**Day 37: Feature gating**
- Gate multi-pass welding behind Advanced plan
- Gate sequence optimization behind Advanced plan
- Show upgrade prompts with pricing info

### Phase 8 — Polish + Deployment (Days 38–42)

**Day 38: Solver validation benchmarks**
- Benchmark 1: Single-pass bead-on-plate (200mm × 100mm × 10mm mild steel, 200A GMAW), verify peak temperature ~1600°C within 5mm of centerline
- Benchmark 2: T-joint fillet weld, verify angular distortion < 2° (compare to empirical Okerblom formula)
- Benchmark 3: Butt weld with clamping, verify residual stress ~200-400 MPa longitudinal (tensile near weld, compressive far field)
- Reference: published experimental data from welding research papers
- Tolerance: peak temperature ±10%, distortion ±20%, stress distribution qualitative match

**Day 39: Docker and K8s**
- `Dockerfile` — Multi-stage Rust build with cargo-chef
- `k8s/api-deployment.yaml` — API replicas with HPA
- `k8s/worker-deployment.yaml` — Simulation workers with autoscaling based on Redis queue length
- `k8s/mesh-service-deployment.yaml` — Python gmsh service
- Health checks: `/health/live`, `/health/ready`

**Day 40: Monitoring and logging**
- Prometheus metrics: simulation duration histogram, solver convergence rate, queue depth
- Grafana dashboards: system health, throughput, user activity
- Sentry for error tracking with source maps
- Structured JSON logging for all solver events

**Day 41: PDF report generation**
- Python FastAPI service `report-service/` — Accept simulation ID, fetch results from S3
- Generate PDF with matplotlib plots: temperature history, stress profiles, distortion contour
- Include simulation parameters, material properties, summary statistics
- Upload PDF to S3, return presigned URL

**Day 42: Launch prep**
- End-to-end smoke test in production environment
- Database backup and restore verification
- Rate limiting: 10 simulations/min per user
- Security audit: JWT validation, SQL injection, CORS, CSP headers
- Landing page with product demo video
- Documentation: quick start guide, parameter selection guide, troubleshooting
- Deploy to production, enable alerts, announce launch

---

## Critical Files

```
weldsim/
├── solver/                                # Rust FEM solver library
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── mesh.rs                        # Gmsh .msh parser, mesh data structures
│   │   ├── materials.rs                   # Material property interpolation
│   │   ├── thermal.rs                     # Thermal FEM solver, Goldak heat source
│   │   ├── mechanical.rs                  # Mechanical FEM solver, Newton-Raphson
│   │   ├── plasticity.rs                  # Von Mises plasticity, radial return
│   │   ├── metallurgy.rs                  # Phase transformation (JMAK, KM)
│   │   ├── heat_source.rs                 # Goldak, conical, uniform heat sources
│   │   ├── boundary_conditions.rs         # Convection, radiation, clamps
│   │   └── vtk_writer.rs                  # Write .vtu unstructured grid
│   └── tests/
│       ├── thermal_benchmarks.rs          # 1D thermal rod, analytical comparison
│       ├── mechanical_benchmarks.rs       # Thermal expansion, elastic beam
│       └── integration.rs                 # Full thermal-mechanical coupling

├── weldsim-api/                           # Rust API server
│   ├── Cargo.toml
│   ├── src/
│   │   ├── main.rs
│   │   ├── config.rs
│   │   ├── state.rs
│   │   ├── error.rs
│   │   ├── auth/
│   │   │   ├── mod.rs
│   │   │   └── oauth.rs
│   │   ├── api/
│   │   │   ├── router.rs
│   │   │   ├── handlers/
│   │   │   │   ├── auth.rs
│   │   │   │   ├── users.rs
│   │   │   │   ├── projects.rs
│   │   │   │   ├── materials.rs
│   │   │   │   ├── simulation.rs          # Create sim, trigger mesh, enqueue job
│   │   │   │   ├── results.rs             # Get results, presigned S3 URLs
│   │   │   │   ├── billing.rs
│   │   │   │   └── webhooks/stripe.rs
│   │   │   └── ws/
│   │   │       └── simulation_progress.rs # WebSocket live progress
│   │   ├── middleware/
│   │   │   └── plan_limits.rs
│   │   ├── services/
│   │   │   ├── usage.rs
│   │   │   ├── s3.rs
│   │   │   └── mesh_client.rs             # HTTP client for mesh service
│   │   └── workers/
│   │       └── simulation_worker.rs       # Redis consumer, run solver
│   ├── migrations/
│   │   └── 001_initial.sql
│   └── tests/
│       └── api_integration.rs

├── mesh-service/                          # Python gmsh meshing service
│   ├── requirements.txt                   # fastapi, uvicorn, gmsh, numpy
│   ├── main.py                            # FastAPI app, /generate endpoint
│   ├── mesher/
│   │   ├── geometry.py                    # Construct gmsh geometry from JSON
│   │   ├── weld_path.py                   # Extrude weld bead volume
│   │   ├── refinement.py                  # Distance field, size function
│   │   ├── export.py                      # Write .msh, simplify STL
│   │   └── quality.py                     # Mesh quality checks
│   └── Dockerfile

├── report-service/                        # Python PDF report generator
│   ├── requirements.txt                   # fastapi, matplotlib, reportlab
│   ├── main.py                            # FastAPI app, /generate endpoint
│   ├── plots/
│   │   ├── temperature.py                 # T(t) plot
│   │   ├── stress.py                      # Residual stress profile
│   │   └── distortion.py                  # Distortion contour
│   └── templates/
│       └── report_template.html           # HTML template for PDF

├── frontend/                              # React frontend
│   ├── package.json
│   ├── vite.config.ts
│   ├── src/
│   │   ├── App.tsx
│   │   ├── main.tsx
│   │   ├── stores/
│   │   │   ├── authStore.ts
│   │   │   ├── projectStore.ts            # Geometry, simulation state
│   │   │   └── resultsStore.ts            # Loaded VTK results
│   │   ├── hooks/
│   │   │   ├── useSimulationProgress.ts   # WebSocket hook
│   │   │   └── useVtkLoader.ts            # Load and parse .vtu
│   │   ├── lib/
│   │   │   ├── api.ts                     # Axios API client
│   │   │   └── vtkLoader.ts               # Parse VTK XML, convert to BufferGeometry
│   │   ├── pages/
│   │   │   ├── Dashboard.tsx
│   │   │   ├── Editor.tsx                 # 3D editor + simulation panel
│   │   │   ├── Results.tsx                # Results viewer
│   │   │   ├── Materials.tsx              # Material library browser
│   │   │   ├── Billing.tsx
│   │   │   ├── Login.tsx
│   │   │   └── Register.tsx
│   │   ├── components/
│   │   │   ├── WeldEditor/
│   │   │   │   ├── WeldScene.tsx          # Three.js Canvas, lights, controls
│   │   │   │   ├── BaseMetal.tsx          # Box/cylinder geometry
│   │   │   │   ├── WeldBeadEditor.tsx     # Path drawing, tube visualization
│   │   │   │   ├── ClampMarker.tsx        # Clamp placement
│   │   │   │   └── MeshPreview.tsx        # Display simplified STL
│   │   │   ├── SimulationPanel.tsx        # Process params, material selector
│   │   │   ├── ResultsViewer/
│   │   │   │   ├── StressContours.tsx     # Vertex-colored mesh with shader
│   │   │   │   ├── TemperaturePlot.tsx    # Line chart for T(t)
│   │   │   │   ├── StressProfile.tsx      # Stress line plot
│   │   │   │   └── SummaryCards.tsx       # Peak values
│   │   │   └── billing/
│   │   │       ├── PlanCard.tsx
│   │   │       └── UsageMeter.tsx
│   │   └── shaders/
│   │       ├── contour.vert.glsl          # Vertex shader for deformed mesh
│   │       └── contour.frag.glsl          # Fragment shader for color mapping

├── k8s/
│   ├── api-deployment.yaml                # API server with HPA
│   ├── worker-deployment.yaml             # Solver workers, autoscale on queue depth
│   ├── postgres-statefulset.yaml
│   ├── redis-deployment.yaml
│   ├── mesh-service-deployment.yaml
│   ├── report-service-deployment.yaml
│   └── ingress.yaml

├── docker-compose.yml                     # Local dev stack
├── Cargo.toml                             # Workspace root
└── .github/workflows/
    ├── ci.yml                             # Test + lint on PR
    └── deploy.yml                         # Build Docker images, deploy to K8s
```

---

## Solver Validation Suite

### Benchmark 1: Bead-on-Plate Single-Pass Weld (Thermal)

**Geometry:** 200mm × 100mm × 10mm mild steel plate

**Process:** GMAW, 200A, 25V, travel speed 5 mm/s, efficiency 0.8

**Goldak parameters:** a=3mm, b=3mm, c_front=6mm, c_rear=10mm

**Expected peak temperature:** ~1600-1700°C at weld centerline

**Expected HAZ width:** ~8-12mm (region above 800°C)

**Tolerance:** Peak temperature ±10%, HAZ width ±15%

**Validation:** Compare thermal cycle (T vs. time) at point 5mm from centerline against published experimental data (e.g., Goldak 1984 paper)

### Benchmark 2: T-Joint Fillet Weld Distortion (Thermal-Mechanical)

**Geometry:** Two 150mm × 75mm × 6mm plates in T-joint configuration, fillet weld 100mm long

**Process:** GMAW, 180A, 24V, travel speed 4 mm/s

**Clamps:** Fixed at both ends during welding, released after cooling to 100°C

**Expected angular distortion:** 1.5-2.5° (web plate rotation)

**Expected longitudinal shrinkage:** 0.2-0.4mm

**Tolerance:** Angular distortion ±20%, shrinkage ±30%

**Validation:** Compare to Okerblom empirical formula and experimental measurements from AWS Welding Handbook

### Benchmark 3: Butt Weld Residual Stress (Thermal-Mechanical-Metallurgical)

**Geometry:** Two 200mm × 100mm × 8mm low-alloy steel plates, V-groove butt weld

**Process:** GMAW, 220A, 26V, travel speed 3 mm/s

**Expected longitudinal residual stress:** ~300-500 MPa tensile near weld centerline, ~100-200 MPa compressive at 30-50mm from weld

**Expected transverse stress:** ~200-300 MPa tensile in HAZ

**Expected hardness in HAZ:** 250-320 HV (due to martensite formation)

**Tolerance:** Stress distribution qualitative match, peak stress ±25%, hardness ±10%

**Validation:** Compare stress profile to neutron diffraction measurements from published literature (e.g., Withers & Bhadeshia 2001)

### Benchmark 4: Thermal Expansion Verification (Elastic Only)

**Geometry:** 100mm × 100mm × 10mm plate, uniform heating from 20°C to 200°C

**Material:** Mild steel, α = 12×10^(-6) K^(-1), E = 200 GPa, ν = 0.3

**Expected strain:** ε = α ΔT = 12×10^(-6) × 180 = **2.16×10^(-3)** (0.216%)

**Expected displacement:** Δ = ε × L = 2.16×10^(-3) × 100mm = **0.216 mm**

**Tolerance:** < 1% error (FEM discretization error should be negligible for uniform field)

### Benchmark 5: Phase Transformation Strain (Metallurgical)

**Geometry:** 50mm × 50mm × 10mm steel sample, rapid quench from 900°C (austenite) to 20°C

**Material:** 0.4% C steel, M_s = 350°C

**Expected martensite fraction:** ~95% (nearly complete transformation)

**Expected volumetric expansion:** ΔV/V ≈ 3% for austenite→martensite

**Expected transformation strain:** ε_tr ≈ 0.01 (1%) isotropic expansion

**Tolerance:** Martensite fraction ±5%, transformation strain ±10%

**Validation:** Compare to dilatometry measurements

---

## Verification

### Integration Testing Checklist

1. **Auth flow** — Register → login → JWT → authorized API calls → token refresh → logout
2. **Project CRUD** — Create → update geometry → save → reload → verify geometry preserved
3. **Material library** — List materials → filter by category → get material → verify property curves
4. **Geometry editor** — Add box → add weld bead → place clamps → verify JSON structure
5. **Mesh generation** — Submit geometry → mesh service generates → mesh uploaded to S3 → preview displayed
6. **Simulation submission** — Set process params → submit → job queued → worker picks up
7. **Solver execution** — Worker loads mesh → runs thermal solve → runs mechanical solve → uploads results
8. **WebSocket progress** — Connect → receive timestep updates → receive completion notification
9. **Results loading** — Fetch .vtu from S3 → parse → render stress contours → verify color mapping
10. **Result plots** — Temperature plot displays correct T(t) → stress profile along path
11. **Plan limits** — Free user → blocked from simulation → upgrade to Pro → simulation allowed
12. **Billing** — Subscribe → Stripe checkout → webhook → plan updated → limits lifted
13. **PDF report** — Request report → Python service generates → PDF uploaded → download link
14. **Error handling** — Mesh generation fails (invalid geometry) → error message → no crash
15. **Concurrent simulations** — 5 users submit simultaneously → all jobs complete correctly

### SQL Verification Queries

```sql
-- 1. Simulation throughput and success rate
SELECT
    DATE(created_at) AS date,
    COUNT(*) AS total_sims,
    SUM(CASE WHEN status = 'completed' THEN 1 ELSE 0 END) AS completed,
    AVG(wall_time_ms) / 1000.0 AS avg_time_sec
FROM simulations
WHERE created_at > NOW() - INTERVAL '7 days'
GROUP BY DATE(created_at)
ORDER BY date DESC;

-- 2. User plan distribution
SELECT plan, COUNT(*) AS user_count
FROM users
GROUP BY plan
ORDER BY user_count DESC;

-- 3. Active projects by user
SELECT u.email, u.plan, COUNT(p.id) AS project_count
FROM users u
LEFT JOIN projects p ON p.owner_id = u.id
GROUP BY u.id
ORDER BY project_count DESC
LIMIT 20;

-- 4. Usage tracking per user (current month)
SELECT u.email, u.plan,
    SUM(CASE WHEN ur.record_type = 'simulation_hours' THEN ur.quantity ELSE 0 END) AS sim_hours,
    SUM(CASE WHEN ur.record_type = 'storage_gb' THEN ur.quantity ELSE 0 END) AS storage_gb
FROM users u
LEFT JOIN usage_records ur ON ur.user_id = u.id
WHERE ur.period_start >= DATE_TRUNC('month', CURRENT_DATE)
GROUP BY u.id
ORDER BY sim_hours DESC
LIMIT 20;

-- 5. Simulation failure analysis
SELECT error_message, COUNT(*) AS failure_count
FROM simulations
WHERE status = 'failed'
GROUP BY error_message
ORDER BY failure_count DESC
LIMIT 10;
```

---

## Deployment Architecture

### Infrastructure Overview

```
┌──────────────────────────────────────────────────────────────┐
│                         CloudFront CDN                       │
│  (Frontend static assets, material property files)          │
└────────────────┬─────────────────────────────────────────────┘
                 │
┌────────────────▼──────────────────────────────────────────────┐
│                      Application Load Balancer               │
│  (HTTPS termination, /api → API pods, /ws → WebSocket)      │
└────────────────┬──────────────────────────────────────────────┘
                 │
     ┌───────────┴────────────┬──────────────┬──────────────┐
     │                        │              │              │
┌────▼─────┐        ┌─────────▼───┐   ┌─────▼──────┐  ┌────▼──────┐
│ API Pods │        │ Worker Pods │   │Mesh Service│  │Report Svc │
│(Rust/Axum│        │(Rust solver)│   │  (Python)  │  │ (Python)  │
│3 replicas│        │Auto-scaling │   │1-2 replicas│  │1 replica  │
│HPA 2-10) │        │based on     │   └─────┬──────┘  └────┬──────┘
└────┬─────┘        │queue depth) │         │              │
     │              └─────────┬───┘         │              │
     │                        │             │              │
     │              ┌─────────▼────────┐    │              │
     │              │  Redis (queue +  │◄───┘              │
     │              │    pub/sub)      │                   │
     │              └──────────────────┘                   │
     │                                                      │
     ├──────────────────────────────────────────────────────┤
     │                                                      │
┌────▼──────────────┐                          ┌───────────▼───────┐
│ PostgreSQL 16     │                          │    AWS S3         │
│ (RDS Multi-AZ)    │                          │ - Mesh files      │
│ - Users, projects │                          │ - Result .vtu     │
│ - Materials       │                          │ - Reports PDF     │
│ - Simulations     │                          │ - Material props  │
└───────────────────┘                          └───────────────────┘
```

### Kubernetes Configuration Highlights

**API Deployment (HPA):**
- Initial replicas: 3
- Min replicas: 2, Max replicas: 10
- CPU target: 70%
- Memory: 1GB per pod
- Health checks: `/health/live`, `/health/ready`

**Worker Deployment (KEDA autoscaling):**
- ScaledObject based on Redis queue length
- Min replicas: 1, Max replicas: 20
- Target queue length: 2 jobs per worker
- CPU request: 8 cores, Memory: 16GB per worker (for large FEM problems)
- Preemptible nodes for cost savings

**Mesh Service:**
- 1-2 replicas (mesh generation is fast, 10-30 seconds)
- CPU: 2 cores, Memory: 4GB
- Gmsh installed in Docker image

**Database (RDS):**
- PostgreSQL 16 Multi-AZ (high availability)
- Instance type: db.t4g.large (2 vCPU, 8GB RAM) for MVP, scale to db.r6g.xlarge if needed
- Automated backups (daily), 7-day retention
- Read replica for analytics queries (post-MVP)

**S3 Buckets:**
- `weldsim-meshes` — .msh files (lifecycle policy: delete after 30 days if simulation failed)
- `weldsim-results` — .vtu files (lifecycle policy: delete after 90 days for free users, 1 year for paid)
- `weldsim-reports` — PDF reports (permanent)
- `weldsim-materials` — Material property JSON files (public read, CDN cached)

### Cost Estimation (MVP, 100 active users)

- EKS cluster: $75/month
- API pods (3 × t3.medium): $75/month
- Worker pods (avg 3 × c6i.2xlarge spot): $150/month (spot pricing, high variance)
- RDS PostgreSQL (db.t4g.large Multi-AZ): $120/month
- Redis (ElastiCache t4g.small): $30/month
- S3 storage (100GB meshes + 200GB results): $10/month
- CloudFront (1TB transfer): $85/month
- **Total: ~$545/month** (scales with worker usage)

---

## Post-MVP Roadmap

### v1.1 — Multi-Pass Welding (Weeks 13–16)

- **Element activation**: Born-dead elements for filler metal, activate as weld passes
- **Inter-pass temperature control**: Enforce minimum/maximum inter-pass temperature between passes
- **Pass sequencing editor**: Define weld pass order, layer-by-layer for thick joints
- **Cumulative residual stress**: Mechanical boundary conditions from previous pass affect next pass
- **Validation**: Multi-pass butt weld (3-5 passes), compare distortion to single-pass equivalent

### v1.2 — Welding Sequence Optimization (Weeks 17–20)

- **Genetic algorithm**: Optimize pass sequence to minimize total distortion
- **Objective function**: Weighted sum of angular distortion, longitudinal shrinkage, max residual stress
- **Constraint**: Inter-pass temperature, accessibility (some passes can't be done before others)
- **Parallel evaluation**: Distribute GA population across multiple workers
- **UI**: Visual sequence editor with drag-to-reorder passes, optimization progress chart

### v1.3 — Advanced Processes (Weeks 21–26)

- **Laser welding**: Keyhole heat source model (Gaussian surface + volumetric), high power density, narrow HAZ
- **Resistance spot welding (RSW)**: Coupled electrical-thermal-mechanical, Joule heating, nugget growth prediction
- **Friction stir welding (FSW)**: Eulerian-Lagrangian material flow, tool force prediction, shoulder/pin heat generation
- **Process-specific material models**: RSW electrode wear, FSW tool wear, laser porosity risk

### v1.4 — Post-Weld Heat Treatment (PWHT) (Weeks 27–28)

- **Stress relief simulation**: Uniform heating to 600-650°C, hold for 1-2 hours, slow cooling
- **Creep model**: Norton power law active during hold time, stress relaxation
- **Effectiveness metric**: Compare residual stress before/after PWHT
- **UI**: PWHT schedule editor (heating rate, hold temperature, hold time, cooling rate)

### v1.5 — Weld Procedure Specification (WPS) Generation (Weeks 29–32)

- **Code compliance**: AWS D1.1, ASME IX, EN ISO 15614 templates
- **Auto-fill from simulation**: Process parameters, heat input, preheat, interpass temperature
- **Hardness prediction**: Convert phase fractions to Vickers hardness, verify <350 HV requirement
- **Mechanical property prediction**: Yield strength, tensile strength from microstructure (empirical correlations)
- **PDF template**: WPS form with simulation-predicted values, visual aids (thermal cycle, HAZ map)

### v1.6 — Experimental Calibration Tools (Weeks 33–36)

- **Inverse analysis**: Adjust Goldak parameters to match measured weld pool dimensions (width, depth)
- **Thermal cycle fitting**: Match thermocouple data to optimize heat source efficiency
- **Distortion fitting**: Match CMM-measured distortion to optimize material property curves
- **Uncertainty quantification**: Monte Carlo sampling of material properties, report confidence intervals
- **Data import**: Upload thermocouple CSV, CMM point cloud, overlay on simulation results

### v1.7 — Digital Twin Integration (Weeks 37–40)

- **Production monitoring**: Real-time data from welding robot (current, voltage, travel speed)
- **Adaptive simulation**: Update simulation parameters mid-weld based on measured deviations
- **Defect prediction**: Compare measured thermal cycle to simulation, flag potential defects (lack of fusion, porosity)
- **Quality assurance**: Automatic accept/reject based on simulation-predicted HAZ hardness, residual stress
- **API**: REST endpoints for robot integration, webhook callbacks for real-time alerts

### v1.8 — Advanced Material Models (Weeks 41–44)

- **Anisotropic properties**: Directional thermal conductivity, Young's modulus (for rolled plate, additive manufactured parts)
- **Multicomponent alloys**: Full thermodynamic phase transformation (equilibrium + kinetics via CALPHAD databases)
- **Aluminum alloys**: Precipitation hardening (Shercliff-Ashby model), dissolution in HAZ
- **Titanium alloys**: α/β phase transformation, grain growth kinetics
- **Custom CCT/TTT upload**: User-provided diagrams for proprietary alloys

---

**Total MVP implementation: 42 days (8.4 weeks)**
**Total lines (estimated): 1480 lines**

