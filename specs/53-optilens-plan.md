# 53. OptiLens — Optical System Design and Ray Tracing Platform

## Implementation Plan

**MVP Scope:** Browser-based optical design interface with lens editor for spherical, conic, and aspheric surfaces rendered via SVG with 2D cross-section and 3D WebGL system layout, sequential ray tracing engine implementing exact ray-surface intersection with Snell's law and Fresnel equations compiled to WebAssembly for client-side execution of systems ≤20 surfaces and server-side Rust-native execution for larger systems, Schott and Ohara glass catalogs with Sellmeier/Schott dispersion formulas (Abbe number, partial dispersion), damped least-squares (DLS) optimization with ray-based operands (spot RMS radius, wavefront RMS, sagittal/tangential focus), analysis outputs including spot diagram with Airy disk overlay, geometrical MTF (polychromatic + diffraction), ray fan plots (transverse ray aberration), wavefront maps with Zernike decomposition, 3D optical layout with traced rays, Zemax ZMX file import for legacy design migration, optical prescription table export (CSV/JSON), Stripe billing with three tiers (Free / Pro $149/mo / Advanced $349/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| Ray Tracer | Rust (native + WASM) | Custom sequential ray tracer with exact surface intersections |
| WASM Compilation | `wasm-pack` + `wasm-bindgen` | Client-side ray tracing for systems ≤20 surfaces |
| Optimizer | Rust (native + WASM) | Damped least-squares (DLS) with BLAS via `nalgebra` |
| Database | PostgreSQL 16 | Projects, users, designs, glass catalog, optimization history |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | Design files, ray data, optimization checkpoints, prescription exports |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| Lens Editor | Custom SVG renderer | Drag-and-drop surface editing, constraint-based layout |
| 3D Visualization | Three.js (WebGL) | 3D system layout, ray paths, lens geometry with materials |
| Spot Diagram | HTML Canvas 2D | High-density scatter plots with colormap for wavelength |
| MTF / Ray Fans | Plotly.js | Interactive 2D plots with zoom/pan, CSV export |
| Real-time | WebSocket (Axum) | Live optimization progress, convergence monitoring |
| Job Queue | Redis 7 + Tokio tasks | Server-side optimization job management, Monte Carlo batching |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | WASM bundle delivery, glass catalog JSON, static frontend assets |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, error tracking |
| CI/CD | GitHub Actions | WASM build pipeline, Rust tests, Docker image push |

### Key Architecture Decisions

1. **Hybrid client/server ray tracer with WASM threshold at 20 surfaces**: Optical systems with ≤20 surfaces (covers 95%+ of camera lenses, eyepieces, microscope objectives, simple telescopes) run entirely in the browser via WASM, providing instant ray tracing and optimization feedback with zero server cost. Systems exceeding 20 surfaces (zoom lenses with 30+ elements, complex microscope illumination paths) are submitted to the Rust-native server. The threshold is configurable per plan tier, with Free tier limited to 10 surfaces.

2. **Custom ray tracer in Rust rather than wrapping Zemax or OSC**: Building a custom sequential ray tracer in Rust gives us full control over optimization algorithms, WASM compilation, and parallelization for multi-configuration systems. Zemax's kernel is closed-source and expensive to license; the open-source alternatives (ray-optics, KrakenOS) are Python-based and too slow for interactive optimization. Our Rust tracer uses exact analytical ray-surface intersections (quadratic solver for conics, Newton-Raphson for aspheres) and achieves <1ms per ray on modern CPUs.

3. **SVG lens editor with constraint-based layout**: SVG gives crisp rendering at any zoom level and straightforward DOM-based interaction for dragging surfaces and adjusting thicknesses. The editor uses constraint propagation to maintain physical validity (positive thickness, apertures smaller than parent surfaces, total track length). Canvas/WebGL alternatives were rejected because they require reimplementing text rendering, cursor interaction, and accessibility for surface labels.

4. **Three.js for 3D system visualization with material shading**: The 3D viewer renders lens elements as extruded surface profiles with correct glass materials (refractive index-based shading, transparency, dispersion rainbow effects). Ray paths are drawn as GPU-instanced line segments with color-coded wavelengths. This is separate from the 2D SVG editor to allow simultaneous editing and 3D preview. Users can export 3D geometry as STEP files for mechanical CAD integration.

5. **S3 for design storage with PostgreSQL metadata catalog**: Complete optical designs (surface prescriptions, glass assignments, aperture stops, field points, wavelengths) are stored as JSON in S3, while PostgreSQL holds searchable metadata (design name, author, surface count, system type, optimization status). This allows the design library to scale to 100K+ designs without bloating the database while enabling fast search via PostgreSQL full-text search and filtering.

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

-- Organizations
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

-- Optical Designs
CREATE TABLE designs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    system_type TEXT NOT NULL,
    design_data_url TEXT NOT NULL,
    surface_count INTEGER NOT NULL DEFAULT 0,
    total_track REAL,
    efl REAL,
    fno REAL,
    field_of_view REAL,
    wavelengths REAL[] DEFAULT '{}',
    is_optimized BOOLEAN NOT NULL DEFAULT false,
    optimization_merit REAL,
    is_public BOOLEAN NOT NULL DEFAULT false,
    forked_from UUID REFERENCES designs(id) ON DELETE SET NULL,
    thumbnail_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX designs_owner_idx ON designs(owner_id);
CREATE INDEX designs_updated_idx ON designs(updated_at DESC);

-- Glass Catalog
CREATE TABLE glasses (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    catalog TEXT NOT NULL,
    name TEXT NOT NULL,
    nd REAL NOT NULL,
    vd REAL NOT NULL,
    pg_f REAL,
    dispersion_formula TEXT NOT NULL,
    dispersion_coeffs REAL[] NOT NULL,
    tce REAL,
    density REAL,
    cost_category TEXT,
    transmission_data JSONB,
    is_standard BOOLEAN NOT NULL DEFAULT true,
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(catalog, name)
);
CREATE INDEX glasses_catalog_idx ON glasses(catalog);
CREATE INDEX glasses_nd_vd_idx ON glasses(nd, vd);

-- Optimization Runs
CREATE TABLE optimizations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    design_id UUID NOT NULL REFERENCES designs(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    algorithm TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending',
    execution_mode TEXT NOT NULL DEFAULT 'wasm',
    initial_merit REAL NOT NULL,
    final_merit REAL,
    iterations INTEGER DEFAULT 0,
    operands JSONB NOT NULL,
    variables JSONB NOT NULL,
    constraints JSONB DEFAULT '[]',
    convergence_history JSONB DEFAULT '[]',
    results_url TEXT,
    error_message TEXT,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    wall_time_ms INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX opts_design_idx ON optimizations(design_id);
CREATE INDEX opts_status_idx ON optimizations(status);

-- Optimization Jobs
CREATE TABLE optimization_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    optimization_id UUID NOT NULL REFERENCES optimizations(id) ON DELETE CASCADE,
    worker_id TEXT,
    cores_allocated INTEGER DEFAULT 4,
    priority INTEGER DEFAULT 0,
    progress_pct REAL DEFAULT 0.0,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX jobs_opt_idx ON optimization_jobs(optimization_id);

-- Analysis Results
CREATE TABLE analyses (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    design_id UUID NOT NULL REFERENCES designs(id) ON DELETE CASCADE,
    analysis_type TEXT NOT NULL,
    configuration_index INTEGER DEFAULT 0,
    results_url TEXT NOT NULL,
    thumbnail_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX analyses_design_idx ON analyses(design_id);
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
    pub plan: String,
    pub stripe_customer_id: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Design {
    pub id: Uuid,
    pub owner_id: Uuid,
    pub name: String,
    pub system_type: String,
    pub design_data_url: String,
    pub surface_count: i32,
    pub efl: Option<f64>,
    pub fno: Option<f64>,
    pub wavelengths: Vec<f64>,
    pub is_optimized: bool,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Glass {
    pub id: Uuid,
    pub catalog: String,
    pub name: String,
    pub nd: f64,
    pub vd: f64,
    pub dispersion_formula: String,
    pub dispersion_coeffs: Vec<f64>,
    pub is_standard: bool,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct OpticalSystem {
    pub surfaces: Vec<Surface>,
    pub wavelengths: Vec<f64>,
    pub fields: Vec<Field>,
    pub aperture: Aperture,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct Surface {
    pub index: i32,
    pub radius: f64,
    pub thickness: f64,
    pub glass: Option<String>,
    pub semi_diameter: f64,
    pub conic: f64,
    pub aspheric_coeffs: Vec<f64>,
    pub is_stop: bool,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct Field {
    pub x: f64,
    pub y: f64,
    pub weight: f64,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum Aperture {
    EntrancePupilDiameter { diameter: f64 },
    Fno { fno: f64 },
}
```

---

## Ray Tracer Architecture Deep-Dive

### Governing Equations and Ray Propagation

OptiLens implements sequential ray tracing with exact analytical ray-surface intersections.

**Ray representation:**
```
Ray = (P, D̂)  where P = (x, y, z), D̂ = (l, m, n) unit direction
```

**Surface intersection** for conic surface:
```
Surface: z = (c·r²) / (1 + √(1 - (1+k)·c²·r²)) + Σ Aᵢ·r^(2i)
c = 1/R, k = conic constant, r² = x² + y²
For sphere: quadratic equation
For asphere: Newton-Raphson from sphere intersection
```

**Snell's law** refraction:
```
n₁·sin(θ₁) = n₂·sin(θ₂)
Vector form: D̂_out = (n₁/n₂)·D̂_in + ((n₁/n₂)·cos(θ₁) - cos(θ₂))·N̂
```

**Sellmeier dispersion** for glass:
```
n²(λ) - 1 = Σᵢ (Bᵢ·λ²) / (λ² - Cᵢ)
```

**Merit function** for optimization:
```
Merit = Σᵢ wᵢ·(operandᵢ - targetᵢ)²
Operands: spot RMS, wavefront RMS, transverse ray aberration, MTF
```

### Client/Server Split

```
Design → Surface count
    ├── ≤20 surfaces → WASM (browser, <100ms)
    └── >20 surfaces → Server (Redis queue, parallelized)
```

20-surface threshold covers 95%+ of common optical systems: camera lenses (5-7), microscope objectives (10-18), telescopes (2-4).

---

## Architecture Deep-Dives

### 1. Ray Tracer Core (Rust)

```rust
// tracer-core/src/ray.rs

use nalgebra::{Vector3, Point3};

#[derive(Debug, Clone)]
pub struct Ray {
    pub origin: Point3<f64>,
    pub direction: Vector3<f64>,
    pub wavelength: f64,
    pub opl: f64,
}

pub struct Surface {
    pub radius: f64,
    pub conic: f64,
    pub aspheric_coeffs: Vec<f64>,
    pub semi_diameter: f64,
}

impl Surface {
    pub fn intersect(&self, ray: &Ray) -> Option<f64> {
        if self.radius.abs() < 1e-12 {
            return Some(-ray.origin.z / ray.direction.z);
        }

        // Quadratic solver for sphere
        let c = 1.0 / self.radius;
        let ox = ray.origin.x;
        let oy = ray.origin.y;
        let oz = ray.origin.z;
        let dx = ray.direction.x;
        let dy = ray.direction.y;
        let dz = ray.direction.z;

        let a = dx*dx + dy*dy + dz*dz;
        let b = 2.0*(ox*dx + oy*dy + (oz - self.radius)*dz);
        let c_quad = ox*ox + oy*oy + (oz - self.radius)*(oz - self.radius) - self.radius*self.radius;

        let discriminant = b*b - 4.0*a*c_quad;
        if discriminant < 0.0 { return None; }

        let t = (-b - discriminant.sqrt()) / (2.0*a);
        if t < 1e-10 { return None; }

        // Check aperture
        let pt = ray.origin + ray.direction * t;
        let r = (pt.x*pt.x + pt.y*pt.y).sqrt();
        if r > self.semi_diameter { return None; }

        // Refine with Newton-Raphson if aspheric
        if !self.aspheric_coeffs.is_empty() {
            self.refine_aspheric(ray, t)
        } else {
            Some(t)
        }
    }

    fn refine_aspheric(&self, ray: &Ray, mut t: f64) -> Option<f64> {
        for _ in 0..10 {
            let pt = ray.origin + ray.direction * t;
            let r_sq = pt.x*pt.x + pt.y*pt.y;
            let c = 1.0 / self.radius;
            let sqrt_term = 1.0 - (1.0 + self.conic)*c*c*r_sq;
            if sqrt_term < 0.0 { return None; }

            let sag = c*r_sq / (1.0 + sqrt_term.sqrt());
            let sag_asph: f64 = self.aspheric_coeffs.iter().enumerate()
                .map(|(i, &a)| a * r_sq.powi(i as i32 + 2))
                .sum();

            let f = pt.z - (sag + sag_asph);
            if f.abs() < 1e-9 { return Some(t); }
            t -= f / ray.direction.z;
        }
        Some(t)
    }

    pub fn normal_at(&self, pt: &Point3<f64>) -> Vector3<f64> {
        let r = (pt.x*pt.x + pt.y*pt.y).sqrt();
        if r < 1e-12 {
            return Vector3::new(0.0, 0.0, if self.radius > 0.0 { -1.0 } else { 1.0 });
        }

        let c = 1.0 / self.radius;
        let r_sq = pt.x*pt.x + pt.y*pt.y;
        let sqrt_term = 1.0 - (1.0 + self.conic)*c*c*r_sq;
        let dz_dr = c*r / sqrt_term.sqrt();

        let nx = -dz_dr * pt.x / r;
        let ny = -dz_dr * pt.y / r;
        Vector3::new(nx, ny, 1.0).normalize()
    }
}

pub fn refract(ray: &Ray, normal: &Vector3<f64>, n1: f64, n2: f64) -> Option<Ray> {
    let cos_i = -ray.direction.dot(normal);
    let n_ratio = n1 / n2;
    let sin_t_sq = n_ratio*n_ratio * (1.0 - cos_i*cos_i);
    if sin_t_sq > 1.0 { return None; }

    let cos_t = (1.0 - sin_t_sq).sqrt();
    let dir_out = n_ratio*ray.direction + (n_ratio*cos_i - cos_t)*normal;

    Some(Ray {
        origin: ray.origin,
        direction: dir_out.normalize(),
        wavelength: ray.wavelength,
        opl: ray.opl,
    })
}

pub fn trace_ray(mut ray: Ray, system: &OpticalSystem) -> Option<Vec<Point3<f64>>> {
    let mut path = vec![ray.origin];

    for (i, surf) in system.surfaces.iter().enumerate() {
        let t = surf.intersect(&ray)?;
        ray.origin += ray.direction * t;
        path.push(ray.origin);

        let n1 = if i == 0 { 1.0 } else { system.get_n(i-1, ray.wavelength) };
        let n2 = system.get_n(i, ray.wavelength);
        ray.opl += n1 * t;

        let normal = surf.normal_at(&ray.origin);
        ray = refract(&ray, &normal, n1, n2)?;
    }
    Some(path)
}
```

### 2. DLS Optimizer (Rust)

```rust
// tracer-core/src/optimizer/dls.rs

use nalgebra::{DMatrix, DVector};

pub struct DlsOptimizer {
    pub system: OpticalSystem,
    pub variables: Vec<Variable>,
    pub operands: Vec<Operand>,
    pub damping: f64,
    pub max_iterations: usize,
}

pub struct Variable {
    pub var_type: VarType,
    pub surface_index: usize,
    pub current_value: f64,
    pub min: f64,
    pub max: f64,
}

pub enum VarType {
    Radius,
    Thickness,
    Conic,
    Aspheric { coeff_index: usize },
}

pub struct Operand {
    pub operand_type: OperandType,
    pub target: f64,
    pub weight: f64,
}

pub enum OperandType {
    SpotRms,
    WavefrontRms,
    EFL,
}

impl DlsOptimizer {
    pub fn optimize(&mut self) -> Result<OptimizationResult, OptimizerError> {
        let mut merit_history = vec![self.compute_merit()];

        for iter in 0..self.max_iterations {
            let jacobian = self.compute_jacobian();
            let residuals = self.compute_residuals();

            let jt_j = &jacobian.transpose() * &jacobian;
            let damped = jt_j + DMatrix::identity(self.variables.len(), self.variables.len()) * self.damping;
            let jt_r = &jacobian.transpose() * &residuals;
            let delta = damped.lu().solve(&(-jt_r)).ok_or(OptimizerError::SingularMatrix)?;

            // Update variables with bounds
            for (i, var) in self.variables.iter_mut().enumerate() {
                var.current_value = (var.current_value + delta[i]).clamp(var.min, var.max);
            }
            self.apply_variables();

            let new_merit = self.compute_merit();
            merit_history.push(new_merit);

            if (merit_history[iter] - new_merit).abs() / merit_history[0] < 1e-6 {
                return Ok(OptimizationResult { final_merit: new_merit, iterations: iter+1, converged: true });
            }

            // Adjust damping
            if new_merit < merit_history[iter] {
                self.damping *= 0.5;
            } else {
                self.damping *= 2.0;
            }
        }

        Ok(OptimizationResult { final_merit: merit_history[merit_history.len()-1], iterations: self.max_iterations, converged: false })
    }

    fn compute_jacobian(&self) -> DMatrix<f64> {
        let mut jac = DMatrix::zeros(self.operands.len(), self.variables.len());
        for (j, var) in self.variables.iter().enumerate() {
            let orig = self.get_var_value(var);
            self.set_var_value(var, orig + 1e-6);
            let ops_plus = self.evaluate_operands();
            self.set_var_value(var, orig - 1e-6);
            let ops_minus = self.evaluate_operands();
            self.set_var_value(var, orig);

            for i in 0..self.operands.len() {
                jac[(i, j)] = (ops_plus[i] - ops_minus[i]) / 2e-6;
            }
        }
        jac
    }

    fn compute_merit(&self) -> f64 {
        let ops = self.evaluate_operands();
        self.operands.iter().enumerate()
            .map(|(i, op)| op.weight * (ops[i] - op.target).powi(2))
            .sum()
    }

    fn evaluate_operands(&self) -> Vec<f64> {
        self.operands.iter().map(|op| match op.operand_type {
            OperandType::SpotRms => self.compute_spot_rms(),
            OperandType::WavefrontRms => self.compute_wavefront_rms(),
            OperandType::EFL => self.compute_efl(),
        }).collect()
    }

    fn compute_spot_rms(&self) -> f64 {
        let rays = self.generate_pupil_rays();
        let spots: Vec<_> = rays.into_iter().filter_map(|r| trace_ray(r, &self.system)).collect();
        let cx: f64 = spots.iter().map(|p| p.last().unwrap().x).sum::<f64>() / spots.len() as f64;
        let cy: f64 = spots.iter().map(|p| p.last().unwrap().y).sum::<f64>() / spots.len() as f64;
        let rms: f64 = spots.iter()
            .map(|p| ((p.last().unwrap().x - cx).powi(2) + (p.last().unwrap().y - cy).powi(2)))
            .sum::<f64>() / spots.len() as f64;
        rms.sqrt()
    }
}

pub struct OptimizationResult {
    pub final_merit: f64,
    pub iterations: usize,
    pub converged: bool,
}

#[derive(Debug)]
pub enum OptimizerError {
    SingularMatrix,
}
```

### 3. Three.js 3D Viewer (React/TypeScript)

```typescript
// frontend/src/components/SystemViewer3D.tsx

import { useRef, useEffect } from 'react';
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';

export function SystemViewer3D({ design }: { design: any }) {
  const canvasRef = useRef<HTMLCanvasElement>(null);

  useEffect(() => {
    if (!canvasRef.current) return;

    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x1a1a1a);

    const camera = new THREE.PerspectiveCamera(50, window.innerWidth/window.innerHeight, 0.1, 10000);
    camera.position.set(0, 200, 500);

    const renderer = new THREE.WebGLRenderer({ canvas: canvasRef.current, antialias: true });
    renderer.setSize(window.innerWidth, window.innerHeight);

    const controls = new OrbitControls(camera, renderer.domElement);

    scene.add(new THREE.AmbientLight(0xffffff, 0.5));
    const light = new THREE.DirectionalLight(0xffffff, 0.8);
    light.position.set(0, 500, 500);
    scene.add(light);

    // Render optical system
    let z_pos = 0;
    design.surfaces.forEach((surf: any, i: number) => {
      const points = [];
      for (let j = 0; j <= 64; j++) {
        const r = (j/64) * surf.semi_diameter;
        const z = computeSag(r, surf.radius, surf.conic);
        points.push(new THREE.Vector2(r, z));
      }
      const geom = new THREE.LatheGeometry(points, 64);
      const mat = new THREE.MeshPhysicalMaterial({
        color: 0xffffff,
        transmission: surf.glass ? 0.95 : 0.1,
        roughness: 0,
        ior: 1.517,
      });
      const mesh = new THREE.Mesh(geom, mat);
      mesh.position.z = z_pos;
      scene.add(mesh);
      z_pos += surf.thickness;
    });

    const animate = () => {
      requestAnimationFrame(animate);
      controls.update();
      renderer.render(scene, camera);
    };
    animate();

    return () => { renderer.dispose(); };
  }, [design]);

  return <canvas ref={canvasRef} />;
}

function computeSag(r: number, R: number, k: number): number {
  if (Math.abs(R) < 1e-9) return 0;
  const c = 1/R;
  const sqrt_term = 1 - (1+k)*c*c*r*r;
  if (sqrt_term < 0) return 0;
  return (c*r*r) / (1 + Math.sqrt(sqrt_term));
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–4)

**Day 1: Backend scaffold**
- Initialize Rust workspace: `cargo init optilens-api && cd optilens-api`
- Add core dependencies: `cargo add axum@0.7 tokio@1 --features full`
- Add database: `cargo add sqlx@0.7 --features postgres,runtime-tokio-rustls,uuid,chrono,json`
- Add utilities: `cargo add serde@1 serde_json@1 uuid@1 chrono@0.4 tower-http@0.5 jsonwebtoken@9 bcrypt@0.15`
- Add cloud services: `cargo add aws-sdk-s3@1 redis@0.24`
- Create `src/main.rs` with Axum server, CORS middleware, request tracing, graceful shutdown
- Create `src/config.rs` with environment-based config (DATABASE_URL, JWT_SECRET, S3_BUCKET, REDIS_URL, AWS_REGION)
- Create `src/state.rs` with AppState struct holding PgPool, Redis connection, S3Client
- Create `src/error.rs` with ApiError enum (NotFound, Unauthorized, BadRequest, Internal) and IntoResponse impl
- Create `Dockerfile` with multi-stage build (rust:1.75-slim builder, debian:bookworm-slim runtime)
- Create `docker-compose.yml` with PostgreSQL 16, Redis 7, MinIO (S3-compatible local storage)
- Test: `docker-compose up -d && cargo run` should start server on localhost:3000

**Day 2: Database schema**
- Create `migrations/001_initial.sql` with all 9 tables and indexes
- Tables: users, organizations, org_members, designs, glasses, optimizations, optimization_jobs, analyses, usage_records
- Enable extensions: uuid-ossp (UUID generation), pg_trgm (fuzzy text search)
- Create `src/db/mod.rs` with pool initialization and connection health check
- Create `src/db/models.rs` with SQLx FromRow structs for all tables
- Run migrations: `sqlx migrate run`
- Create `scripts/seed_glasses.sql` with Schott glass catalog (100+ glasses: N-BK7, SF2, N-SF6, etc.)
- Create `scripts/seed_ohara.sql` with Ohara glass catalog (100+ glasses: S-LAH66, S-FPL51, etc.)
- Each glass entry includes: catalog, name, nd, Vd, Pg,F, dispersion formula type, 6 Sellmeier coefficients
- Run seed scripts: `psql < scripts/seed_glasses.sql`
- Test: Query glasses table, verify 200+ entries with correct dispersion data

**Day 3: Authentication**
- Create `src/auth/mod.rs` with JWT Claims struct (user_id, email, exp), generate_token, validate_token
- Implement auth middleware: extract Bearer token from Authorization header, validate JWT, inject Claims into request
- Create `src/auth/oauth.rs` with OAuth 2.0 flow for Google and GitHub
- OAuth handlers: initiate (redirect to provider), callback (exchange code for token, create/update user)
- Create `src/api/handlers/auth.rs` with register (bcrypt password hash), login (verify hash, return JWT), refresh_token, me (get current user)
- Password requirements: min 8 chars, uppercase, lowercase, number
- JWT expiry: access token 24h, refresh token 30d
- Test: Register new user, login, call /me with token, refresh token after expiry

**Day 4: Design and glass CRUD**
- Create `src/api/handlers/designs.rs` with create, list, get, update, delete, fork endpoints
- Create: validate OpticalSystem, compute EFL/F-no, upload JSON to S3, insert metadata to PostgreSQL
- Update: download from S3, apply partial updates, re-upload, update metadata
- Fork: copy design from S3 to new key, create new database entry with forked_from reference
- Create `src/api/handlers/glasses.rs` with search (full-text + parametric filters), get_by_name, add_custom
- Search filters: catalog (Schott/Ohara), nd range (1.4-2.0), Vd range (20-90), cost category (low/medium/high)
- Create `src/api/handlers/orgs.rs` with create_org, invite_member (send email), list_members, remove_member
- Create `src/api/router.rs` mounting all routes with auth middleware where needed
- Write integration tests: auth flow, create design, update design, search glasses, fork design
- Test: Run `cargo test --test integration_tests` and verify all pass

### Phase 2 — Ray Tracer Core (Days 5–12)

**Day 5: Ray and surface geometry primitives**
- Create new workspace member: `tracer-core/` with `Cargo.toml` (dependencies: nalgebra = "0.33", serde = "1")
- Create `tracer-core/src/ray.rs` with Ray struct (origin: Point3, direction: Vector3, wavelength: f64, intensity: f64, opl: f64)
- Implement Ray methods: new, propagate (move origin along direction), add_opl (accumulate optical path length)
- Create `tracer-core/src/surface.rs` with Surface struct (radius, conic, aspheric_coeffs, semi_diameter, is_stop)
- Implement Surface::intersect for flat surfaces (plane at z=0): return t = -origin.z / direction.z
- Implement Surface::intersect for spherical surfaces: solve quadratic equation (x-xc)² + (y-yc)² + (z-zc)² = R²
- Quadratic coefficients: a = dx²+dy²+dz², b = 2(ox·dx + oy·dy + (oz-R)·dz), c = ox²+oy²+(oz-R)²-R²
- Return smaller positive root, check aperture radius
- Unit tests: ray-plane intersection, ray-sphere intersection at various angles, aperture clipping

**Day 6: Conic and aspheric surfaces**
- Extend Surface::intersect to handle conic surfaces (k ≠ 0): ellipse (0 < k < ∞), parabola (k = -1), hyperbola (k < -1)
- Use sphere intersection as initial guess, then refine position if conic ≠ 0
- For aspheric surfaces (aspheric_coeffs non-empty): Newton-Raphson iteration starting from conic/sphere intersection
- Aspheric sag formula: z = (c·r²)/(1+√(1-(1+k)c²r²)) + A₄r⁴ + A₆r⁶ + A₈r⁸ + ...
- Iterate: t_new = t - f(t)/f'(t) where f(t) = z(ray.origin + t·ray.direction) - sag_formula
- Convergence criterion: |f(t)| < 1e-9 mm, max 10 iterations
- Implement Surface::normal_at for all surface types using gradient of surface equation
- Normal at point p: N = (-∂z/∂x, -∂z/∂y, 1).normalize() where ∂z/∂r = dc·r/√(...) + Σ 2i·Aᵢ·r^(2i-1)
- Tests: paraboloid focus verification, ellipsoid ray paths, aspheric lens convergence

**Day 7: Refraction and material dispersion**
- Create `tracer-core/src/refract.rs` with refract function implementing vector form of Snell's law
- Input: incident ray, surface normal, n1 (incident index), n2 (transmitted index)
- Compute cos(θ₁) = -D̂·N̂, check for total internal reflection: n1·sin(θ₁) > n2
- If TIR, return None (ray does not refract)
- Else compute refracted direction: D̂_out = (n1/n2)·D̂_in + ((n1/n2)·cos(θ₁) - cos(θ₂))·N̂
- Create `tracer-core/src/glass.rs` with Glass struct (name, catalog, nd, Vd, Pg,F, dispersion_formula, coeffs)
- Implement Sellmeier formula: n²(λ) - 1 = Σᵢ (Bᵢ·λ²)/(λ² - Cᵢ) where λ in microns, Bᵢ and Cᵢ from coeffs
- Implement Schott formula: n²(λ) - 1 = A₀ + A₁·λ² + A₂·λ⁻² + A₃·λ⁻⁴ + A₄·λ⁻⁶ + A₅·λ⁻⁸
- Implement Abbe number computation: Vd = (nd - 1)/(nF - nC) where nF at 486.1nm, nd at 587.6nm, nC at 656.3nm
- Tests: verify Sellmeier for N-BK7 at 486nm/587nm/656nm, prism dispersion angle, TIR in fiber

**Day 8: Sequential ray tracing through multi-surface systems**
- Create `tracer-core/src/system.rs` with OpticalSystem struct containing surfaces, wavelengths, fields, aperture
- Implement OpticalSystem::get_refractive_index(surface_idx, wavelength) using glass catalog lookup and Sellmeier eval
- Create `tracer-core/src/trace.rs` with trace_ray function: input Ray and OpticalSystem, output Vec<Point3> (ray path) or None if ray fails
- Loop over surfaces: find intersection, propagate ray to intersection, accumulate OPL, compute refraction, propagate to next surface thickness
- Handle aperture stops: if surface.is_stop, check if ray passes through aperture, else mark as vignetted
- Handle vignetting: track which rays are clipped at each surface for vignetting analysis
- Return complete ray path including all intersection points and final image point
- Tests: single biconvex lens focus, achromatic doublet with F/d/C rays, Cooke triplet on-axis convergence

**Day 9: Paraxial optics and first-order properties**
- Create `tracer-core/src/paraxial.rs` with paraxial ray trace using y-nu method (paraxial ray has height y, angle nu at each surface)
- Transfer: y₂ = y₁ + t·nu₁ where t is thickness
- Refraction: nu₂ = nu₁ + y₁·power where power = (n₂-n₁)/R
- Trace two paraxial rays: marginal ray (starts at y=1, nu=0) and chief ray (starts at y=0, nu=object angle)
- Compute effective focal length: EFL = -y_marginal / nu_marginal at final surface
- Compute back focal length: BFL = distance from last surface to image plane
- Compute front focal length: FFL = distance from first surface to object plane
- Compute F-number: F/# = EFL / (2 × entrance pupil radius)
- Compute numerical aperture: NA = n × sin(θ) where θ is half-angle of marginal ray cone
- Tests: verify thick lens formula, compound lens EFL matches matrix method, F-number calculation

**Day 10: Spot diagram generation and image plane analysis**
- Create `tracer-core/src/analysis/spot.rs` with generate_pupil_rays function
- For each field point, generate grid of rays across entrance pupil (e.g., 11×11 grid, 121 rays total)
- Pupil sampling: uniform grid, Gaussian-weighted, or hexagonal packing
- Trace all rays to image plane, collect (x, y) coordinates where each ray intersects image
- Compute spot centroid: (x̄, ȳ) = mean of all ray intersections
- Compute RMS spot radius: rms = √(Σ((x-x̄)² + (y-ȳ)²) / N)
- Compute geometric spot radius: max distance from centroid to any ray
- Compute Airy disk radius for comparison: r_airy = 1.22 × λ × F/# (in same units as spot)
- Output SpotDiagram struct with ray coordinates, centroid, rms, geometric radius, Airy radius
- Tests: perfect lens (all rays converge to point, RMS ≈ 0), compare diffraction-limited lens to Airy disk

**Day 11: Wavefront analysis and Zernike decomposition**
- Create `tracer-core/src/analysis/wavefront.rs` with compute_wavefront function
- For each ray, compute optical path length from object to image: OPL = Σ nᵢ·dᵢ
- Trace chief ray (center of pupil) and use its image point as reference for exit pupil center
- Construct reference sphere centered at chief ray image point with radius equal to distance from exit pupil to image
- Compute OPD for each ray: OPD = OPL(ray) - OPL(chief) - distance_to_reference_sphere
- Arrange OPD values in 2D grid indexed by normalized pupil coordinates (ρ, θ) where ρ ∈ [0,1]
- Fit Zernike polynomials to OPD data: OPD(ρ,θ) = Σ aᵢ·Zᵢ(ρ,θ)
- Zernike terms (first 15): piston, tilt X/Y, defocus, astigmatism 0°/45°, coma X/Y, trefoil X/Y, spherical, secondary astigmatism, etc.
- Compute RMS wavefront error: rms_wfe = √(Σ OPDᵢ² / N) - a₀² (exclude piston)
- Compute peak-to-valley (P-V) wavefront error: P-V = max(OPD) - min(OPD)
- Tests: verify defocus term for known defocused lens, spherical aberration detection in single lens

**Day 12: Ray fan plots and geometrical MTF**
- Create `tracer-core/src/analysis/ray_fan.rs` with compute_ray_fan function
- Sample rays across 1D slice of pupil (e.g., Y-axis slice from -1 to +1 in normalized pupil coords)
- Trace rays and measure transverse aberration: Δy = y_actual - y_chief (sagittal), Δx = x_actual - x_chief (tangential)
- Plot Δy vs normalized pupil coordinate for sagittal ray fan, Δx vs normalized pupil for tangential ray fan
- Create `tracer-core/src/analysis/mtf.rs` with compute_mtf function (geometrical MTF)
- Generate high-density spot diagram (1000+ rays)
- Compute point spread function (PSF): 2D histogram of ray density at image plane
- Take 1D slice of PSF along sagittal and tangential directions
- Compute line spread function (LSF) by integrating PSF perpendicular to slice direction
- Compute MTF by Fourier transform of LSF: MTF(f) = |FFT(LSF)|
- Normalize MTF(0) = 1.0, plot MTF vs spatial frequency (cycles/mm)
- Tests: perfect lens MTF = 1.0 at all frequencies, verify MTF at Nyquist frequency for known lens designs

### Phase 3 — WASM + Frontend (Days 13–18)

**Day 13: WASM tracer compilation and build pipeline**
- Create `tracer-wasm/` workspace member with `Cargo.toml` dependencies: wasm-bindgen, serde-wasm-bindgen, js-sys
- Add `[lib] crate-type = ["cdylib", "rlib"]` to enable WASM compilation
- Create `tracer-wasm/src/lib.rs` with wasm_bindgen exports for trace_ray, analyze_spot, analyze_wavefront
- Use serde-wasm-bindgen to serialize/deserialize OpticalSystem between JS and Rust
- Add console_log for browser logging: `#[wasm_bindgen] pub fn init_panic_hook() { console_error_panic_hook::set_once(); }`
- Configure release profile for size optimization: opt-level="z", lto=true, codegen-units=1
- Install wasm-pack: `cargo install wasm-pack`
- Build WASM module: `cd tracer-wasm && wasm-pack build --target web --release`
- Output: `pkg/` directory with .wasm binary, .js glue code, .d.ts types
- Create `.github/workflows/wasm-build.yml` for CI: checkout, install Rust + wasm32 target, build with wasm-pack, upload to S3
- Test: Load WASM in browser, call trace_ray with simple lens, verify output

**Day 14: React frontend scaffold and state management**
- Initialize Vite project: `npm create vite@latest frontend -- --template react-ts`
- Install dependencies: `npm i three @types/three plotly.js @types/plotly.js zustand axios`
- Install dev tools: `npm i -D @vitejs/plugin-react typescript tailwindcss postcss autoprefixer`
- Initialize Tailwind: `npx tailwindcss init -p`, configure content paths in tailwind.config.js
- Create `src/stores/designStore.ts` with Zustand store for designs, ray data, analysis results
- Store state: designs (Map<id, Design>), currentDesignId, rayPaths, spotData, mtfData, wavefrontData
- Store actions: setCurrentDesign, updateSurface, addSurface, removeSurface, setRayPaths, setAnalysisResults
- Create `src/api/client.ts` with Axios instance configured with baseURL, auth interceptor (inject JWT from localStorage)
- API methods: login, register, getDesigns, getDesign, createDesign, updateDesign, getGlasses, startOptimization
- Create `src/App.tsx` with main layout: sidebar (design list, properties panel), main canvas area (2D/3D toggle), toolbar (file, edit, analyze)
- Test: Render empty layout, verify state management with simple design object

**Day 15: SVG lens editor with interactive surface manipulation**
- Create `src/components/LensEditor.tsx` using SVG for 2D optical layout rendering
- Compute Y-coordinates for surfaces based on semi-diameter, render each surface as SVG path (arc for curved, line for flat)
- Compute Z-coordinates based on cumulative thickness, render surfaces at correct axial positions
- Implement drag handlers for surface manipulation: onMouseDown captures surface index, onMouseMove updates radius or thickness
- Radius drag: move cursor perpendicular to optical axis changes curvature (cursor left = concave, right = convex)
- Thickness drag: move cursor along optical axis changes distance to next surface
- Semi-diameter drag: move cursor radially changes clear aperture
- Implement constraint propagation to maintain physical validity: thickness ≥ 0.1mm, semi_diameter ≤ parent surface diameter, total track ≤ max_track
- Create surface property panel: inputs for radius, thickness, glass selection (dropdown from catalog), conic constant, aspheric coefficients
- Add snap-to-grid feature: round radius to nearest 5mm, thickness to nearest 0.1mm when dragging
- Render optical axis as dashed line, aperture stop with bold outline, ray fans emanating from object
- Test: Create simple lens, drag surface to change radius, verify constraint enforcement

**Day 16: Three.js 3D system viewer with realistic rendering**
- Create `src/components/SystemViewer3D.tsx` using Three.js for 3D visualization
- Initialize scene with dark background (0x1a1a1a), perspective camera at (0, 200, 500), OrbitControls for rotation
- Add lighting: ambient light (0.5 intensity), directional light from (0, 500, 500) with 0.8 intensity
- For each surface, create lens element geometry by extruding surface profile (sag curve) around optical axis
- Surface profile: sample 64 points from r=0 to r=semi_diameter, compute z = sag(r) using conic formula
- Create LatheGeometry from profile points, rotate around Z-axis to form rotationally symmetric lens
- Assign glass material: MeshPhysicalMaterial with transmission=0.95, roughness=0, ior=1.517 (from glass catalog), thickness=10
- For air gaps, use transparent material with transmission=0.1
- Render aperture stop as torus geometry (donut ring) at stop surface position
- Render optical axis as thin gray cylinder from z=-50 to z=total_track+50
- For ray paths, create line geometry connecting each ray's intersection points, color by wavelength (R/G/B map from 400-700nm)
- Implement wavelength-to-RGB conversion: 400-500nm (violet to blue), 500-600nm (green to yellow), 600-700nm (orange to red)
- Add system info overlay in top-right corner: surface count, EFL, F/no, field of view
- Test: Render Cooke triplet, verify glass elements are transparent with refractions, ray paths show chromatic dispersion

**Day 17: Analysis visualization components (Spot, MTF, Wavefront)**
- Create `src/components/SpotDiagram.tsx` using HTML Canvas 2D for scatter plot
- Render each ray as small circle (2px radius) at its (x, y) intersection point on image plane
- Color rays by wavelength: map wavelength to RGB color
- Draw Airy disk as circle (dashed line) with radius 1.22λF/# centered at chief ray
- Draw centroid as crosshair, annotate RMS radius in microns
- Add zoom/pan controls: mousewheel for zoom, drag for pan
- Create `src/components/MTFPlot.tsx` using Plotly.js for line chart
- X-axis: spatial frequency (cycles/mm), Y-axis: MTF (0-1)
- Plot two lines: sagittal MTF (blue), tangential MTF (red)
- Add diffraction limit line (dashed gray) for comparison
- Add markers at 50 lp/mm (typical imaging benchmark) and Nyquist frequency
- Create `src/components/WavefrontMap.tsx` using Plotly.js heatmap
- 2D heatmap of OPD values indexed by normalized pupil coordinates
- Colorscale: blue (negative OPD) to red (positive OPD), white at zero
- Annotate with RMS WFE and P-V values in legend
- Overlay Zernike term labels on map (piston, tilt, defocus, etc.)
- Create `src/components/RayFanPlot.tsx` for transverse aberration
- X-axis: normalized pupil coordinate (-1 to +1), Y-axis: transverse aberration (mm)
- Two subplots: sagittal (Δy) and tangential (Δx)
- Add CSV export button to all analysis components: serialize data and trigger download
- Test: Display spot diagram for Cooke triplet, verify MTF plot matches geometric expectations

**Day 18: WASM integration and real-time ray tracing**
- Create `src/wasm/tracer.ts` module to load and wrap WASM functions
- Load WASM: `import init, { trace_ray, analyze_spot } from 'tracer-wasm'` then `await init()`
- Wrap trace_ray: convert JS OpticalSystem to wasm-bindgen compatible format, call WASM function, parse output
- Create `src/hooks/useRayTracing.ts` React hook that triggers WASM ray tracing on design changes
- Use useEffect with debounce (300ms) to avoid thrashing on rapid edits
- When design changes, call WASM trace_ray for grid of rays (5 fields × 3 wavelengths × 40 pupil samples = 600 rays)
- Store ray paths in designStore for 3D rendering
- Create `src/hooks/useAnalysis.ts` hook for spot/MTF/wavefront computation
- When user requests analysis, call WASM analyze_spot, analyze_wavefront, compute_mtf
- Display loading spinner during WASM computation (typically 50-200ms for 20-surface system)
- Display ray count and trace time in status bar: "Traced 600 rays in 87ms"
- Add surface count check: if ≤20 surfaces use WASM (client-side), else show "Submitting to server..." message
- Test: Edit lens in SVG editor, verify ray paths update in 3D viewer within 300ms, analysis plots refresh on demand

### Phase 4 — Optimization (Days 19–25)

**Day 19: DLS optimizer core implementation**
- Extend `tracer-core/src/optimizer/dls.rs` with full DlsOptimizer struct
- Implement compute_jacobian using central finite differences: J[i,j] = (f(x+δ) - f(x-δ)) / (2δ)
- Use δ = 1e-6 for radius/thickness variables, 1e-8 for conic/aspheric coefficients
- Implement Levenberg-Marquardt damping: solve (J^T·J + λ·I)·Δx = -J^T·r where λ is damping factor
- Start with λ = 0.01, multiply by 0.5 on successful step (reduce damping, more Gauss-Newton), multiply by 2.0 on failed step (increase damping, more gradient descent)
- Accept step if new merit < old merit, else reject and increase damping
- Convergence criteria: |merit_change| / initial_merit < 1e-6 OR gradient norm < 1e-9 OR max iterations reached
- Implement variable clamping: after computing Δx, clamp each variable to [min, max] bounds before applying
- Track optimization history: store merit value and wall time at each iteration
- Tests: optimize simple lens to minimize spot RMS, verify convergence within 50 iterations

**Day 20: Optimization operands and merit function**
- Create `tracer-core/src/optimizer/operands.rs` with Operand enum and evaluation functions
- SpotRms operand: trace pupil grid, compute RMS radius from centroid, target = 0 (minimize)
- WavefrontRms operand: trace rays, compute OPD relative to reference sphere, RMS of OPD values
- TransverseRayY operand: trace specific ray (e.g., marginal ray at field edge), measure Y-aberration, target = 0
- TransverseRayX operand: same for X-aberration (tangential direction)
- EFL operand: compute paraxial effective focal length, target = design EFL (constraint)
- BFL operand: compute back focal length, useful for maintaining packaging constraints
- ChiefRayAngle operand: angle of chief ray at image, target for telecentricity
- EdgeThickness operand: minimum thickness at edge of lens element, target ≥ 1mm for manufacturability
- Implement weighted merit function: Merit = Σ wᵢ·(operandᵢ - targetᵢ)²
- Default weights: spot RMS weight = 1.0, wavefront RMS weight = 0.5, constraints weight = 10.0
- Tests: evaluate operands for known lens designs, verify spot RMS matches manual calculation

**Day 21: Optimization variables and constraints**
- Create `tracer-core/src/optimizer/variables.rs` with Variable struct and VarType enum
- VarType::Radius: optimize surface radius, bounds typically [-1000mm, 1000mm], exclude R=0 (flat)
- VarType::Thickness: optimize air gaps or element thickness, bounds [0.1mm, 100mm]
- VarType::Conic: optimize conic constant, bounds [-10, 10] (covers hyperbola to oblate ellipse)
- VarType::Aspheric: optimize individual aspheric coefficients (A4, A6, A8), bounds [-1e-4, 1e-4]
- Create `tracer-core/src/optimizer/constraints.rs` with constraint enforcement
- Inequality constraints: edge_thickness ≥ min_thickness, semi_diameter ≤ max_diameter
- Equality constraints: total_track = target_track (soft constraint via operand)
- Implement penalty method for constraint violations: add large penalty to merit if constraint violated
- Penalty = 1000 × |violation|² for hard constraints
- Implement constraint checking before applying variable updates: if update would violate constraint, clamp to boundary
- Tests: optimize lens with thickness constraint, verify minimum edge thickness maintained

**Day 22: WASM optimizer compilation for client-side optimization**
- Add WASM bindings in `tracer-wasm/src/optimizer.rs`
- Export optimize_design function: takes OpticalSystem, variables, operands, constraints, returns optimized system + history
- Limit WASM optimizer to max 100 iterations to keep browser responsive (<5s total)
- Implement progress callback: after each iteration, call JS function with current merit value
- Use js_sys::Function to invoke JS callback from Rust: `callback.call1(&JsValue::null(), &progress_obj)`
- Package optimization result: final design, convergence history (Vec<(iteration, merit, time_ms)>), converged flag
- Build WASM optimizer: ensure optimization code compiles to WASM target (no std::thread, use single-threaded eval)
- Test: Run optimization in browser for simple lens, verify convergence within 5s, progress updates every 100ms

**Day 23: Server-side optimization worker for large systems**
- Create `src/workers/optimization_worker.rs` following same pattern as simulation worker
- Worker subscribes to Redis "optimization:jobs" queue with BRPOP (blocking pop)
- On job received: fetch optimization record from database, download design from S3, parse OpticalSystem
- Instantiate DlsOptimizer with native Rust (not WASM), configure for 1000+ iterations with tighter convergence
- Run optimization with progress callback that updates database every 10 iterations: `UPDATE optimization_jobs SET progress_pct = X, current_iteration = Y`
- Stream progress via WebSocket to frontend: publish to Redis pub/sub channel "optimization:{id}:progress"
- On completion: upload optimized design to S3, update database with final merit, iterations, wall time, status="completed"
- On error: catch panic, update database with status="failed", error_message
- Handle cancellation: check Redis key "optimization:{id}:cancel" every 10 iterations, abort if set
- Deploy 4 worker instances initially, autoscale to 16 based on queue depth
- Tests: submit optimization job, verify worker picks up, completes, and stores result in S3

**Day 24: Optimization UI and convergence monitoring**
- Create `src/components/OptimizationSetup.tsx` for configuring optimization parameters
- Variable selection: checkboxes for each surface radius, thickness, conic; input fields for bounds
- Operand configuration: add operand buttons (spot RMS, wavefront RMS, etc.), set target and weight for each
- Constraint configuration: add constraint (edge thickness ≥ 2mm, track ≤ 150mm), set limits
- Algorithm selection: dropdown for DLS, Global Search, or Hybrid
- "Start Optimization" button: submit optimization request to backend, receive job ID
- Create `src/components/ConvergenceChart.tsx` using Plotly.js line chart
- X-axis: iteration number, Y-axis: merit function value (log scale)
- Update chart in real-time as optimization progresses via WebSocket updates
- Annotate chart with initial merit, current merit, percent improvement
- Show "Converged" badge when optimization completes successfully
- Add "Stop Optimization" button to cancel running optimization
- Create `src/components/OptimizationHistory.tsx` list of past optimizations for this design
- Display: timestamp, algorithm, final merit, iterations, wall time, converged flag
- Click entry to load optimized design into editor
- Test: Run optimization on simple lens, verify real-time convergence chart updates, load optimized result

**Day 25: Global search optimizer for non-convex problems**
- Create `tracer-core/src/optimizer/global.rs` with GeneticAlgorithm and SimulatedAnnealing structs
- GeneticAlgorithm: population of 50 designs, crossover (blend parent variables), mutation (Gaussian noise), elitism (keep best 10)
- Fitness function: inverse of merit function (lower merit = higher fitness)
- Selection: tournament selection with k=3
- Run for 200 generations, each generation evaluates 50 designs (10K total evaluations)
- SimulatedAnnealing: start at high temperature (T=100), exponentially decay (T *= 0.95 per iteration)
- At each iteration, propose random variable perturbation, accept if merit improves OR with probability exp(-ΔE/T)
- Run for 5000 iterations with temperature annealing schedule
- Hybrid approach: run global search for 50 iterations to find basin, then switch to DLS for local refinement
- Parallelization: use rayon to parallelize fitness evaluations across CPU cores (native only, not WASM)
- Tests: optimize highly aberrated lens that DLS fails on, verify global search finds good solution

### Phase 5 — Glass Database and Export (Days 26–29)

**Day 26: Glass catalog data import and seeding**
- Download Schott glass catalog Excel file from official website (100+ glasses)
- Create `scripts/import_schott.py` to parse Excel: extract glass name, nd, Vd, Pg,F, dispersion formula type, 6 Sellmeier coefficients
- Parse transmission data: wavelength vs internal transmittance for 10mm thickness
- Assign cost_category based on glass family: N-BK7="low", SF-series="medium", rare-earth="high"
- Insert into PostgreSQL glasses table with catalog="Schott"
- Download Ohara glass catalog (S-series, L-series, 100+ glasses)
- Create `scripts/import_ohara.py` with same parsing logic
- Seed both catalogs: `python scripts/import_schott.py && python scripts/import_ohara.py`
- Verify: Query `SELECT COUNT(*) FROM glasses WHERE catalog='Schott'` should return 100+
- Create indices on nd and Vd for fast parametric search

**Day 27: Glass search UI and Abbe diagram**
- Create `src/components/GlassSearch.tsx` with search input and filters
- Full-text search on glass name using PostgreSQL pg_trgm: `WHERE name ILIKE '%bk7%'`
- Parametric filters: catalog (Schott/Ohara/All), nd range slider (1.4-2.0), Vd range slider (20-90), cost_category checkboxes
- Display results in table: name, catalog, nd, Vd, Pg,F, cost_category, transmission @550nm
- Click row to view full details: dispersion curve plot (n vs λ from 400-700nm), transmission curve, datasheet link
- Create Abbe diagram: scatter plot with Vd on X-axis, nd on Y-axis, each point is a glass
- Color points by catalog (Schott=blue, Ohara=red), size by cost_category
- Hover over point shows glass name, click to select for assignment to surface
- Implement glass assignment: from Abbe diagram or search results, click "Assign to Surface" button, select surface index from dropdown
- Test: Search for "BK7", verify N-BK7 appears, view dispersion curve, assign to surface, verify design updates

**Day 28: Optical prescription export and documentation**
- Create `src/api/handlers/export.rs` with export_prescription endpoint
- Generate prescription table with columns: surface#, radius, thickness, glass, diameter, conic, aspheric_coeffs
- CSV format: simple comma-separated, easy to import into Excel
- JSON format: complete OpticalSystem serialization for programmatic use
- Excel format: use rust_xlsxwriter crate to generate .xlsx with formatted table, header row, units in column headers
- Include paraxial properties in separate sheet: EFL, BFL, F/no, total track, field of view, wavelengths
- Include performance metrics: on-axis spot RMS, wavefront RMS, MTF@50lp/mm
- ISO 10110 format: surface drawing callouts with surface quality annotations (scratch/dig, irregularity, tolerances)
- Create `src/components/ExportDialog.tsx` with format selection radio buttons and "Download" button
- Test: Export Cooke triplet prescription, open in Excel, verify all data present and formatted

**Day 29: Zemax ZMX file import for legacy migrations**
- Create `tracer-core/src/zemax/parser.rs` to parse Zemax .zmx text file format
- ZMX format: plain text with keyword lines (SURF, GLAS, WAVE, FIEL, etc.)
- Parse SURF lines: surface type (STANDARD, EVENASPH, TOROIDAL), radius, thickness, semi-diameter, conic
- Parse GLAS lines: glass name, check if exists in our catalog, map to equivalent (e.g., Zemax "BK7" → our "N-BK7")
- Parse WAVE lines: wavelength values in microns, set as OpticalSystem.wavelengths
- Parse FIEL lines: field points in degrees or mm, set as OpticalSystem.fields
- Parse APER lines: aperture type (EPD, F-number, NA), parse value
- Handle multi-configuration: CONF, MOFF keywords define zoom positions or switchable elements (import first config only for MVP)
- Convert Zemax surface types to OptiLens types: STANDARD → Standard, EVENASPH → EvenAsphere
- Create import endpoint: POST /api/designs/import with file upload, parse ZMX, create design in database
- Test: Import sample ZMX files (Cooke triplet, achromat doublet), verify surfaces, glasses, wavelengths correct

### Phase 6 — Advanced Analysis (Days 30–35)

**Day 30: Diffraction MTF**
- PSF via FFT of pupil function

**Day 31: Encircled energy**
- Cumulative energy vs radius

**Day 32: Field curvature**
- Sagittal/tangential image surfaces

**Day 33: Chromatic aberration**
- Longitudinal and lateral color

**Day 34: Seidel coefficients**
- Third-order aberrations per surface

**Day 35: Polarization tracing**
- Jones vectors, Fresnel equations

### Phase 7 — Templates and AI (Days 36–39)

**Day 36: Template library**
- 20+ starter designs from patents

**Day 37: Patent database**
- Scrape Google Patents for optical designs

**Day 38: AI design generator**
- Python FastAPI service with LLM

**Day 39: Design assistant**
- RAG-based chatbot for design help

### Phase 8 — Billing and Deploy (Days 40–42)

**Day 40: Stripe integration**
- Checkout, webhooks, Customer Portal
- Plans: Free (10 surf), Pro ($149), Advanced ($349)

**Day 41: Usage enforcement**
- Plan limits middleware
- Usage tracking

**Day 42: Deployment**
- Kubernetes/Fly.io deploy
- Prometheus + Grafana monitoring
- CI/CD automation

---

## Critical Files

```
optilens/
├── backend/
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
│   │   │   └── handlers/
│   │   │       ├── auth.rs
│   │   │       ├── designs.rs
│   │   │       ├── glasses.rs
│   │   │       ├── optimizations.rs
│   │   │       └── billing.rs
│   │   ├── db/
│   │   │   ├── mod.rs
│   │   │   └── models.rs
│   │   └── workers/
│   │       └── optimization_worker.rs
│   ├── migrations/001_initial.sql
│   ├── Cargo.toml
│   └── Dockerfile
├── tracer-core/
│   ├── src/
│   │   ├── lib.rs
│   │   ├── ray.rs
│   │   ├── surface.rs
│   │   ├── refract.rs
│   │   ├── glass.rs
│   │   ├── trace.rs
│   │   ├── paraxial.rs
│   │   ├── analysis/
│   │   │   ├── spot.rs
│   │   │   ├── wavefront.rs
│   │   │   ├── mtf.rs
│   │   │   └── chromatic.rs
│   │   └── optimizer/
│   │       ├── dls.rs
│   │       ├── operands.rs
│   │       └── constraints.rs
│   └── Cargo.toml
├── tracer-wasm/
│   ├── src/lib.rs
│   ├── Cargo.toml
│   └── pkg/
├── frontend/
│   ├── src/
│   │   ├── App.tsx
│   │   ├── stores/designStore.ts
│   │   ├── api/client.ts
│   │   ├── wasm/tracer.ts
│   │   └── components/
│   │       ├── LensEditor.tsx
│   │       ├── SystemViewer3D.tsx
│   │       ├── SpotDiagram.tsx
│   │       ├── MTFPlot.tsx
│   │       ├── OptimizationSetup.tsx
│   │       └── Billing.tsx
│   ├── package.json
│   └── vite.config.ts
├── ai-service/
│   ├── main.py
│   ├── design_generator.py
│   └── requirements.txt
└── docker-compose.yml
```

---

## Ray Tracer Validation Suite

### Benchmark 1: Thin Lens Focal Length

**Test:** Biconvex lens (R1=+50mm, R2=-50mm, t=5mm, N-BK7 nd=1.5168) at d-line.

**Expected:** BFL = 47.73 mm (Lensmaker equation)

**Acceptance:** BFL within 0.01 mm

```rust
let system = create_biconvex(50.0, -50.0, 5.0, "N-BK7");
let (efl, bfl) = compute_paraxial(&system)?;
assert!((bfl - 47.73).abs() < 0.01);
```

### Benchmark 2: Achromatic Doublet

**Test:** Crown+Flint cemented doublet at F, d, C wavelengths. Measure chromatic focal shift.

**Expected:** Longitudinal chromatic < 0.1mm (0.1% of EFL)

**Acceptance:** Chromatic shift < 0.1mm

```rust
let doublet = create_achromat(100.0, "N-BK7", "SF2");
let focus_f = find_focus(&doublet, 0.4861);
let focus_c = find_focus(&doublet, 0.6563);
assert!((focus_f - focus_c).abs() < 0.1);
```

### Benchmark 3: Cooke Triplet Spot Size

**Test:** Cooke triplet (EFL=100mm, F/4) on-axis. 121 rays (11×11 grid).

**Expected:** RMS spot < 10 µm (< 2× Airy disk at 550nm)

**Acceptance:** RMS < 10 µm

```rust
let triplet = load_patent("US1084371");
let spots = trace_pupil_grid(11, 11, &triplet);
assert!(compute_rms(&spots) < 10e-3);
```

### Benchmark 4: Parabolic Mirror Coma

**Test:** F/4 parabola (k=-1) at 0° and 1° off-axis.

**Expected:** On-axis coma < 0.1 µm, 1° coma > 10 µm

**Acceptance:** Confirms on-axis perfection, off-axis aberration

```rust
let mirror = create_parabola(-800.0, -1.0);
assert!(compute_coma(&mirror, 0.0) < 0.1e-3);
assert!(compute_coma(&mirror, 1.0) > 10e-3);
```

### Benchmark 5: Aspheric SA Correction

**Test:** Spherical vs aspheric biconvex F/2, EFL=50mm. Measure spherical aberration.

**Expected:** Spherical: SA ≈ 0.5mm, Aspheric: SA < 0.01mm

**Acceptance:** 50× SA reduction

```rust
let spherical = create_biconvex(50.0, 2.0, "N-BK7");
let mut aspheric = spherical.clone();
aspheric.surfaces[0].aspheric_coeffs = vec![-2e-6];
assert!(compute_sa(&spherical) > 0.4);
assert!(compute_sa(&aspheric) < 0.01);
```

---

## Verification Checklist

**Backend:**
- [ ] JWT auth + OAuth 2.0
- [ ] Design CRUD with S3
- [ ] Glass catalog search
- [ ] Optimization job queue
- [ ] WebSocket progress
- [ ] Stripe billing

**Ray Tracer:**
- [ ] Ray-sphere/conic/asphere intersection
- [ ] Snell's law refraction
- [ ] Sellmeier dispersion
- [ ] Sequential tracing
- [ ] All 5 benchmarks pass

**Optimizer:**
- [ ] DLS with Jacobian
- [ ] Variable bounds
- [ ] Spot/wavefront/EFL operands
- [ ] WASM optimizer <5s
- [ ] Server optimizer 1000+ iterations

**Analysis:**
- [ ] Spot diagram with Airy disk
- [ ] MTF (geometric + diffraction)
- [ ] Ray fan plots
- [ ] Wavefront + Zernike
- [ ] Chromatic aberration

**Frontend:**
- [ ] SVG lens editor
- [ ] Three.js 3D viewer
- [ ] WASM ray tracing <100ms
- [ ] Glass search + Abbe diagram
- [ ] Optimization UI
- [ ] Zemax ZMX import

**Deployment:**
- [ ] Docker images
- [ ] PostgreSQL + Redis + S3
- [ ] CloudFront CDN
- [ ] Prometheus + Grafana
- [ ] CI/CD pipeline

---

## Deployment Architecture

```
CloudFront CDN (WASM, assets)
    ↓
Frontend (React/Vercel)
    ↓ API
Backend (Rust/Axum on Fly.io, 2 instances)
    ↓
PostgreSQL (designs, glasses)
Redis (job queue)
S3 (design files, results)
    ↓
Optimization Workers (Rust, autoscale 4-16)
AI Service (Python FastAPI, 1 GPU instance)
    ↓
Prometheus + Grafana + Sentry
```

**Scaling:**
- Frontend: CDN autoscale
- Backend: 2-8 instances with load balancer
- Workers: Autoscale on queue depth
- PostgreSQL: RDS with read replicas
- Redis: ElastiCache with persistence

**Cost (1000 users, 50% Pro, 10% Advanced):**
- Infrastructure: $1,200/mo
- Revenue: $109,400/mo
- Margin: 99%

---

## Post-MVP Roadmap

### v1.1 — Non-Sequential Ray Tracing (Q2 2026, +8 weeks)
- Illumination design, LED/laser sources
- CAD import (STEP, IGES)
- Scattering models (Lambertian, BSDF)
- Detector analysis (irradiance, color)

### v1.2 — Tolerance Analysis (Q3 2026, +6 weeks)
- Sensitivity analysis
- Monte Carlo tolerancing (10K-100K systems)
- Yield prediction
- Cost-performance optimization

### v1.3 — Physical Optics (Q4 2026, +8 weeks)
- Gaussian beam propagation
- Scalar diffraction (Fresnel, Fraunhofer)
- DOE design
- AR/VR waveguide simulation

### v1.4 — Freeform Surfaces (Q1 2027, +6 weeks)
- XY polynomial, Zernike, NURBS
- ML-guided global optimization
- Multi-objective optimization

### v1.5 — Stray Light (Q2 2027, +6 weeks)
- Ghost identification
- Narcissus analysis
- Coating optimization
- Baffle recommendations

### v1.6 — Collaboration (Q3 2027, +4 weeks)
- Real-time co-editing
- Version control
- PLM integration (Windchill, Teamcenter)
- STEP export with PMI
