# 51. Acoustica — Architectural and Environmental Acoustics Simulation Platform

## Implementation Plan

**MVP Scope:** Browser-based 3D room geometry editor with OBJ import and parametric shoebox room creation rendered via Three.js/WebGL, material database of 500+ frequency-dependent absorption and scattering coefficients (125 Hz - 8 kHz octave bands), GPU-accelerated ray tracing acoustics solver in Rust compiled to WASM for client-side execution of rooms with ≤1M rays and server-side execution for larger simulations, image source method (ISM) for early reflections combined with stochastic ray tracing for late reverberation, room acoustic parameter calculation (RT60 [T20/T30/EDT], C50, C80, D50, STI, G, LF), real-time binaural auralization via Web Audio API convolution with HRTF, interactive 3D visualization of ray paths and SPL distribution, source directivity support (omnidirectional, cardioid, custom polar patterns), receiver positioning with spatial impulse response export, PDF report generation with parameter tables and octave-band charts, Stripe billing with three tiers (Free / Professional $129/mo / Advanced $299/user/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| Ray Tracing Solver | Rust (native + WASM) | Custom geometrical acoustics engine with BVH acceleration via `bvh` crate |
| WASM Compilation | `wasm-pack` + `wasm-bindgen` | Client-side solver for simulations ≤1M rays |
| Optimization Engine | Python 3.12 (FastAPI) | Bayesian optimization for material placement, ML-based absorption prediction |
| Database | PostgreSQL 16 + PostGIS | Projects, users, simulations, material database, GIS data for environmental noise |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | Impulse responses, auralization audio, simulation results, material data |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| 3D Visualization | Three.js + React Three Fiber | Room geometry editing, ray path visualization, SPL mapping |
| Audio Processing | Web Audio API | Convolution-based auralization, HRTF rendering |
| Real-time | WebSocket (Axum) | Live simulation progress, ray count updates |
| Job Queue | Redis 7 + Tokio tasks | Server-side simulation job management, Monte Carlo distribution |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| Report Generation | Python (Matplotlib/ReportLab) | PDF acoustics reports with octave-band charts |
| CDN | CloudFront | WASM bundle delivery, HRTF data, static frontend assets |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, error tracking |
| CI/CD | GitHub Actions | WASM build pipeline, Rust tests, Docker image push |

### Key Architecture Decisions

1. **Hybrid client/server solver with WASM threshold at 1M rays**: Simple room simulations (single rooms, ≤1M rays, covers 80%+ of educational and quick-check cases) run entirely in the browser via WASM, providing instant feedback with zero server cost. Complex simulations (large venues, detailed geometries, environmental noise) are submitted to the Rust-native server solver which handles multi-million ray simulations and optimization runs. The threshold is configurable per plan tier.

2. **Geometrical acoustics (ray tracing + image source) rather than wave-based FEM**: Geometrical methods are valid above the Schroeder frequency (typically 80-200 Hz for rooms) and scale to large spaces (concert halls, airports) with millions of surface triangles. Wave-based methods (FEM/BEM) require solving large linear systems (O(n³) complexity) and are limited to small rooms or low frequencies. We use hybrid ray+ISM: image source method for first-order reflections (exact, specular), stochastic ray tracing for late diffuse field. This matches industry tools (ODEON, EASE) and provides <5% error vs. measurements for RT60 and C80.

3. **Three.js for room editor and visualization**: Three.js provides GPU-accelerated rendering, built-in OBJ import, and ray casting for object picking. React Three Fiber gives declarative React components for scene management. This avoids building a custom WebGL engine while providing professional-grade 3D editing (rotate, pan, zoom, snap-to-grid, material painting). Ray paths and SPL heat maps are rendered as GPU instanced meshes for performance with 100K+ rays.

4. **Web Audio API for auralization with HRTF convolution**: The Web Audio API's ConvolverNode performs real-time convolution of source signals (speech, music) with simulated impulse responses. HRTF (Head-Related Transfer Function) data from the MIT KEMAR database provides binaural rendering for headphone listening. This runs entirely client-side with <20ms latency, enabling interactive "walk through and listen" experiences. Ambisonics rendering is added post-MVP for spatial audio playback.

5. **PostgreSQL + PostGIS for material and GIS data**: The material database (absorption coefficients, scattering coefficients, acoustic properties) is stored in PostgreSQL with trigram search for fuzzy material name matching. PostGIS extends PostgreSQL with spatial types for environmental noise modeling (terrain DEMs, building footprints, noise contours). This provides a unified database for both room acoustics and outdoor propagation, avoiding the need for separate spatial databases.

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";  -- For fuzzy text search on materials
CREATE EXTENSION IF NOT EXISTS "postgis";  -- For GIS data (environmental noise)

-- Users
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email TEXT UNIQUE NOT NULL,
    password_hash TEXT,  -- NULL for OAuth-only users
    name TEXT NOT NULL,
    avatar_url TEXT,
    auth_provider TEXT DEFAULT 'email',  -- email | google | github
    auth_provider_id TEXT,
    plan TEXT NOT NULL DEFAULT 'free',  -- free | professional | advanced
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX users_email_idx ON users(email);
CREATE INDEX users_stripe_idx ON users(stripe_customer_id);

-- Organizations (for Advanced plan)
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

-- Projects (room + simulation workspace)
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    project_type TEXT NOT NULL DEFAULT 'room',  -- room | environmental | building | sound_system
    geometry_data JSONB NOT NULL DEFAULT '{}',  -- Room geometry (vertices, faces, materials)
    settings JSONB DEFAULT '{}',  -- Grid size, units, default source/receiver positions
    is_public BOOLEAN NOT NULL DEFAULT false,
    forked_from UUID REFERENCES projects(id) ON DELETE SET NULL,
    thumbnail_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);
CREATE INDEX projects_org_idx ON projects(org_id);
CREATE INDEX projects_type_idx ON projects(project_type);
CREATE INDEX projects_updated_idx ON projects(updated_at DESC);

-- Simulation Runs
CREATE TABLE simulations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    simulation_type TEXT NOT NULL,  -- ray_tracing | image_source | hybrid | environmental_noise
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | running | completed | failed | cancelled
    execution_mode TEXT NOT NULL DEFAULT 'wasm',  -- wasm | server
    parameters JSONB NOT NULL DEFAULT '{}',  -- Simulation-specific parameters (ray count, max order, etc.)
    ray_count INTEGER,
    triangle_count INTEGER,
    results_url TEXT,  -- S3 URL for impulse responses and ray data
    results_summary JSONB,  -- Quick-access summary (RT60, C80, STI per octave band)
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
    ray_progress JSONB DEFAULT '[]',  -- [{iteration, rays_traced, energy_remaining}]
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX jobs_sim_idx ON simulation_jobs(simulation_id);
CREATE INDEX jobs_status_idx ON simulation_jobs(worker_id);

-- Acoustic Materials (absorption and scattering data)
CREATE TABLE acoustic_materials (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    description TEXT,
    category TEXT NOT NULL,  -- absorber | diffuser | reflector | hybrid
    -- Absorption coefficients (octave bands: 125, 250, 500, 1k, 2k, 4k, 8k Hz)
    absorption_125 REAL NOT NULL DEFAULT 0.01,
    absorption_250 REAL NOT NULL DEFAULT 0.01,
    absorption_500 REAL NOT NULL DEFAULT 0.01,
    absorption_1k REAL NOT NULL DEFAULT 0.01,
    absorption_2k REAL NOT NULL DEFAULT 0.01,
    absorption_4k REAL NOT NULL DEFAULT 0.01,
    absorption_8k REAL NOT NULL DEFAULT 0.01,
    -- Scattering coefficients (octave bands)
    scattering_125 REAL NOT NULL DEFAULT 0.1,
    scattering_250 REAL NOT NULL DEFAULT 0.1,
    scattering_500 REAL NOT NULL DEFAULT 0.1,
    scattering_1k REAL NOT NULL DEFAULT 0.1,
    scattering_2k REAL NOT NULL DEFAULT 0.1,
    scattering_4k REAL NOT NULL DEFAULT 0.1,
    scattering_8k REAL NOT NULL DEFAULT 0.1,
    manufacturer TEXT,
    model_number TEXT,
    thickness_mm REAL,
    density_kg_m3 REAL,
    flow_resistivity REAL,  -- For porous absorbers
    tags TEXT[] DEFAULT '{}',
    is_verified BOOLEAN DEFAULT false,
    is_builtin BOOLEAN DEFAULT true,
    uploaded_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX materials_category_idx ON acoustic_materials(category);
CREATE INDEX materials_name_trgm_idx ON acoustic_materials USING gin(name gin_trgm_ops);
CREATE INDEX materials_tags_idx ON acoustic_materials USING gin(tags);

-- Source Directivity Patterns
CREATE TABLE source_directivity (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    description TEXT,
    pattern_type TEXT NOT NULL,  -- omnidirectional | cardioid | figure8 | custom
    -- For custom patterns: polar data as JSON [{azimuth, elevation, gain_db}]
    polar_data JSONB,
    frequency_dependent BOOLEAN DEFAULT false,
    -- If frequency-dependent, store per-octave-band data
    octave_band_data JSONB,
    manufacturer TEXT,
    model_number TEXT,
    is_builtin BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX directivity_type_idx ON source_directivity(pattern_type);

-- Impulse Response Cache (for auralization)
CREATE TABLE impulse_responses (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    simulation_id UUID NOT NULL REFERENCES simulations(id) ON DELETE CASCADE,
    receiver_position JSONB NOT NULL,  -- {x, y, z}
    source_position JSONB NOT NULL,  -- {x, y, z}
    octave_band INTEGER NOT NULL,  -- 0=125Hz, 1=250Hz, ..., 6=8kHz, 7=broadband
    sample_rate INTEGER NOT NULL DEFAULT 48000,
    duration_ms REAL NOT NULL,
    ir_url TEXT NOT NULL,  -- S3 URL to WAV/FLAC file
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX ir_sim_idx ON impulse_responses(simulation_id);
CREATE INDEX ir_receiver_idx ON impulse_responses(receiver_position);

-- Usage Tracking (for billing enforcement)
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,  -- simulation_minutes | cloud_simulation | storage_bytes
    quantity REAL NOT NULL,
    period_start DATE NOT NULL,
    period_end DATE NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX usage_user_period_idx ON usage_records(user_id, period_start, period_end);

-- Environmental Noise Sources (for post-MVP environmental modeling)
CREATE TABLE noise_sources (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    source_type TEXT NOT NULL,  -- road | railway | industrial | aircraft | point
    geometry GEOMETRY(GEOMETRY, 4326),  -- PostGIS geometry (point, linestring, polygon)
    parameters JSONB NOT NULL DEFAULT '{}',  -- Type-specific parameters (AADT, speed, etc.)
    sound_power_levels JSONB,  -- Per-octave-band sound power levels
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX noise_sources_project_idx ON noise_sources(project_id);
CREATE INDEX noise_sources_geom_idx ON noise_sources USING GIST(geometry);
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
    pub project_type: String,
    pub geometry_data: serde_json::Value,
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
    pub parameters: serde_json::Value,
    pub ray_count: Option<i32>,
    pub triangle_count: Option<i32>,
    pub results_url: Option<String>,
    pub results_summary: Option<serde_json::Value>,
    pub error_message: Option<String>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub wall_time_ms: Option<i32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct AcousticMaterial {
    pub id: Uuid,
    pub name: String,
    pub description: Option<String>,
    pub category: String,
    pub absorption_125: f32,
    pub absorption_250: f32,
    pub absorption_500: f32,
    pub absorption_1k: f32,
    pub absorption_2k: f32,
    pub absorption_4k: f32,
    pub absorption_8k: f32,
    pub scattering_125: f32,
    pub scattering_250: f32,
    pub scattering_500: f32,
    pub scattering_1k: f32,
    pub scattering_2k: f32,
    pub scattering_4k: f32,
    pub scattering_8k: f32,
    pub manufacturer: Option<String>,
    pub model_number: Option<String>,
    pub thickness_mm: Option<f32>,
    pub density_kg_m3: Option<f32>,
    pub flow_resistivity: Option<f32>,
    pub tags: Vec<String>,
    pub is_verified: bool,
    pub is_builtin: bool,
    pub uploaded_by: Option<Uuid>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

impl AcousticMaterial {
    pub fn absorption_at_band(&self, band: OctaveBand) -> f32 {
        match band {
            OctaveBand::Hz125 => self.absorption_125,
            OctaveBand::Hz250 => self.absorption_250,
            OctaveBand::Hz500 => self.absorption_500,
            OctaveBand::Hz1k => self.absorption_1k,
            OctaveBand::Hz2k => self.absorption_2k,
            OctaveBand::Hz4k => self.absorption_4k,
            OctaveBand::Hz8k => self.absorption_8k,
        }
    }

    pub fn scattering_at_band(&self, band: OctaveBand) -> f32 {
        match band {
            OctaveBand::Hz125 => self.scattering_125,
            OctaveBand::Hz250 => self.scattering_250,
            OctaveBand::Hz500 => self.scattering_500,
            OctaveBand::Hz1k => self.scattering_1k,
            OctaveBand::Hz2k => self.scattering_2k,
            OctaveBand::Hz4k => self.scattering_4k,
            OctaveBand::Hz8k => self.scattering_8k,
        }
    }
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize)]
pub enum OctaveBand {
    Hz125 = 0,
    Hz250 = 1,
    Hz500 = 2,
    Hz1k = 3,
    Hz2k = 4,
    Hz4k = 5,
    Hz8k = 6,
}

impl OctaveBand {
    pub fn center_frequency(&self) -> f32 {
        match self {
            OctaveBand::Hz125 => 125.0,
            OctaveBand::Hz250 => 250.0,
            OctaveBand::Hz500 => 500.0,
            OctaveBand::Hz1k => 1000.0,
            OctaveBand::Hz2k => 2000.0,
            OctaveBand::Hz4k => 4000.0,
            OctaveBand::Hz8k => 8000.0,
        }
    }

    pub fn all() -> [OctaveBand; 7] {
        [
            OctaveBand::Hz125,
            OctaveBand::Hz250,
            OctaveBand::Hz500,
            OctaveBand::Hz1k,
            OctaveBand::Hz2k,
            OctaveBand::Hz4k,
            OctaveBand::Hz8k,
        ]
    }
}

#[derive(Debug, Deserialize)]
pub struct SimulationParams {
    pub simulation_type: SimulationType,
    pub ray_count: u32,
    pub max_reflection_order: u8,
    pub transition_order: u8,  // ISM up to this order, then ray tracing
    pub energy_threshold_db: f32,  // Stop ray when energy drops below this
    pub temperature_c: f32,
    pub humidity_percent: f32,
    pub air_pressure_pa: f32,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum SimulationType {
    RayTracing,
    ImageSource,
    Hybrid,
    EnvironmentalNoise,
}

#[derive(Debug, Serialize)]
pub struct RoomAcousticParameters {
    pub rt60: OctaveBandValues,  // Reverberation time (T20, T30, EDT)
    pub edt: OctaveBandValues,   // Early decay time
    pub c50: OctaveBandValues,   // Clarity (speech)
    pub c80: OctaveBandValues,   // Clarity (music)
    pub d50: OctaveBandValues,   // Definition
    pub sti: f32,                // Speech Transmission Index (broadband)
    pub g: OctaveBandValues,     // Strength (relative SPL)
    pub lf: OctaveBandValues,    // Lateral fraction
}

#[derive(Debug, Serialize, Deserialize)]
pub struct OctaveBandValues {
    pub hz_125: f32,
    pub hz_250: f32,
    pub hz_500: f32,
    pub hz_1k: f32,
    pub hz_2k: f32,
    pub hz_4k: f32,
    pub hz_8k: f32,
}

impl OctaveBandValues {
    pub fn get(&self, band: OctaveBand) -> f32 {
        match band {
            OctaveBand::Hz125 => self.hz_125,
            OctaveBand::Hz250 => self.hz_250,
            OctaveBand::Hz500 => self.hz_500,
            OctaveBand::Hz1k => self.hz_1k,
            OctaveBand::Hz2k => self.hz_2k,
            OctaveBand::Hz4k => self.hz_4k,
            OctaveBand::Hz8k => self.hz_8k,
        }
    }

    pub fn set(&mut self, band: OctaveBand, value: f32) {
        match band {
            OctaveBand::Hz125 => self.hz_125 = value,
            OctaveBand::Hz250 => self.hz_250 = value,
            OctaveBand::Hz500 => self.hz_500 = value,
            OctaveBand::Hz1k => self.hz_1k = value,
            OctaveBand::Hz2k => self.hz_2k = value,
            OctaveBand::Hz4k => self.hz_4k = value,
            OctaveBand::Hz8k => self.hz_8k = value,
        }
    }
}
```

---

## Solver Architecture Deep-Dive

### Governing Equations and Ray Tracing Algorithm

Acoustica's core solver implements **geometrical acoustics** using a hybrid approach combining **image source method (ISM)** for early reflections and **stochastic ray tracing** for late reverberation. This is the same approach used by industry-standard tools (ODEON, EASE, Catt-Acoustic).

**Image Source Method (Early Reflections, Orders 1-3):**

For a source position **S**, receiver position **R**, and reflecting surface with plane equation **ax + by + cz + d = 0**, the image source **S'** is:

```
S' = S - 2n̂(n̂·(S-P))

where:
- n̂ = (a,b,c) / ||(a,b,c)|| is the surface normal
- P is any point on the plane
```

The reflection path **S → P → R** contributes an impulse at time **t = (|SP| + |PR|) / c**, where **c** is the speed of sound (343 m/s at 20°C). The impulse amplitude is attenuated by:

```
A = A₀ · (1 - α) · (r₀/r)

where:
- α is the surface absorption coefficient (frequency-dependent)
- r = |SP| + |PR| is the total path length
- r₀ = 1m reference distance
- Geometric spreading follows inverse distance law
```

Image sources are computed recursively up to order N (typically N=3), validating visibility at each step (source sees surface, reflection point sees receiver). This gives exact specular reflections for low-order paths.

**Stochastic Ray Tracing (Late Reflections, Orders 4+):**

For higher-order reflections and diffuse scattering, ray tracing is more efficient. Rays are emitted from the source in uniformly distributed directions (Fibonacci sphere sampling for low discrepancy):

```
For i = 0 to N_rays:
    θ_i = arccos(1 - 2i/N_rays)        # Polar angle
    φ_i = π(1 + √5)i mod 2π             # Azimuthal angle (golden ratio)
    direction_i = (sin(θ_i)cos(φ_i), sin(θ_i)sin(φ_i), cos(θ_i))
```

Each ray carries initial energy **E₀ = W / N_rays**, where **W** is the source power. Ray-surface intersection uses a **BVH (Bounding Volume Hierarchy)** for O(log n) queries. At each intersection:

1. **Energy loss from absorption:** E_new = E · (1 - α)
2. **Scattering decision:** With probability **s** (scattering coefficient), reflect diffusely (cosine-weighted random direction). Otherwise, reflect specularly (mirror direction).
3. **Receiver contribution:** If ray passes within distance **r_threshold** of receiver (typically 0.1m), deposit energy in impulse response bin **t = path_length / c**.

Ray tracing continues until:
- Energy drops below threshold (typically -60 dB relative to initial)
- Reflection order exceeds maximum (typically 100+ for large halls)
- Ray escapes room boundary

**Hybrid Approach:**

For typical room simulations:
- ISM for orders 1-3: Exact specular paths, dominant perceptual cues (localization, ITDG)
- Ray tracing for orders 4+: Statistical late reverberation, diffuse field

This gives the accuracy of ISM for early reflections with the efficiency of ray tracing for late tail.

**Air Absorption:**

High frequencies are attenuated by air absorption (molecular relaxation of O₂ and H₂O). The attenuation coefficient **m** (dB/m) follows ISO 9613-1:

```
m(f) = 8.686 f² [ 1.84×10⁻¹¹ (p_a/p_ref)⁻¹ (T/T_ref)^(1/2)
               + (T/T_ref)^(-5/2) (0.01275 e^(-2239.1/T) (f_rO + f²/f_rO)⁻¹
                                 + 0.10680 e^(-3352.0/T) (f_rN + f²/f_rN)⁻¹) ]

where:
- f is frequency (Hz)
- p_a is atmospheric pressure (Pa), p_ref = 101325 Pa
- T is absolute temperature (K), T_ref = 293.15 K
- f_rO, f_rN are relaxation frequencies for O₂ and N₂ (humidity-dependent)
```

For simplicity, we use pre-computed lookup tables for m(f, T, RH) at octave band centers.

**Room Acoustic Parameters from Impulse Response:**

Given impulse response **h(t)** with energy **E(t) = h(t)²**, compute:

**RT60 (Reverberation Time):** Time for energy to decay 60 dB, estimated from T20 or T30:
```
RT60 ≈ 3 × T20  (time for -5 to -25 dB decay)
RT60 ≈ 2 × T30  (time for -5 to -35 dB decay)
```

**EDT (Early Decay Time):** Time for initial -10 dB decay (0 to -10 dB).

**C50 (Clarity for Speech):** Ratio of early (0-50ms) to late (50ms+) energy:
```
C50 = 10 log₁₀( ∫₀^50ms E(t)dt / ∫₅₀ms^∞ E(t)dt )
```

**C80 (Clarity for Music):** Similar to C50 but with 80ms boundary.

**D50 (Definition):** Early energy fraction:
```
D50 = ∫₀^50ms E(t)dt / ∫₀^∞ E(t)dt
```

**STI (Speech Transmission Index):** Modulation transfer function (MTF) method per IEC 60268-16. Compute MTF for 14 modulation frequencies × 7 octave bands, combine with speech spectrum weighting to get STI ∈ [0,1].

### Client/Server Split (WASM Threshold)

```
Simulation request → Ray count extracted
    │
    ├── ≤1M rays → WASM solver (browser)
    │   ├── Instant startup, no network latency
    │   ├── Results displayed immediately
    │   ├── Progress bar updated at 10k ray intervals
    │   └── No server cost
    │
    └── >1M rays → Server solver (Rust native)
        ├── Job queued via Redis
        ├── Worker picks up, runs native solver (multi-core)
        ├── Progress streamed via WebSocket
        ├── Results stored in S3, URL returned
        └── PDF report generated (Python service)
```

The 1M ray threshold was chosen because:
- WASM solver handles 1M rays (typical small-medium room, 500 triangles) in <5 seconds on modern hardware
- 1M rays covers: classrooms, conference rooms, small theaters, recording studios
- Above 1M rays: concert halls, arenas, outdoor spaces, detailed architectural models → need server compute

### WASM Compilation Pipeline

```toml
# solver-wasm/Cargo.toml
[package]
name = "acoustica-solver-wasm"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
wasm-bindgen = "0.2"
serde = { version = "1", features = ["derive"] }
serde-wasm-bindgen = "0.6"
js-sys = "0.3"
bvh = "0.9"  # BVH acceleration structure
glam = "0.24"  # SIMD vector math
rand = { version = "0.8", features = ["small_rng"] }
rand_distr = "0.4"
log = "0.4"
console_log = "1"

[profile.release]
opt-level = 3
lto = true
codegen-units = 1
```

---

## Architecture Deep-Dives

### 1. Simulation API Handler (Rust/Axum)

The primary endpoint receives a simulation request, validates the room geometry, decides between WASM and server execution, and for server execution enqueues a job with Redis and streams progress via WebSocket.

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
    solver::geometry::validate_geometry,
    state::AppState,
    auth::Claims,
    error::ApiError,
};

#[derive(serde::Deserialize)]
pub struct CreateSimulationRequest {
    pub simulation_type: SimulationType,
    pub parameters: SimulationParams,
    pub source_positions: Vec<[f32; 3]>,
    pub receiver_positions: Vec<[f32; 3]>,
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

    // 2. Validate geometry
    let geometry = validate_geometry(&project.geometry_data)?;
    let triangle_count = geometry.triangles.len();
    let ray_count = req.parameters.ray_count as i32;

    // 3. Check plan limits
    let user = sqlx::query_as!(
        crate::db::models::User,
        "SELECT * FROM users WHERE id = $1",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    if user.plan == "free" && ray_count > 100_000 {
        return Err(ApiError::PlanLimit(
            "Free plan supports up to 100k rays. Upgrade to Professional for unlimited."
        ));
    }

    if user.plan == "free" && triangle_count > 1000 {
        return Err(ApiError::PlanLimit(
            "Free plan supports up to 1000 triangles. Simplify geometry or upgrade."
        ));
    }

    // 4. Determine execution mode
    let execution_mode = if ray_count <= 1_000_000 && triangle_count <= 5000 {
        "wasm"
    } else {
        "server"
    };

    // 5. Create simulation record
    let sim = sqlx::query_as!(
        Simulation,
        r#"INSERT INTO simulations
            (project_id, user_id, simulation_type, status, execution_mode,
             parameters, ray_count, triangle_count)
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8)
        RETURNING *"#,
        project_id,
        claims.user_id,
        serde_json::to_string(&req.simulation_type)?,
        if execution_mode == "wasm" { "completed" } else { "pending" },
        execution_mode,
        serde_json::to_value(&req.parameters)?,
        ray_count,
        triangle_count as i32,
    )
    .fetch_one(&state.db)
    .await?;

    // 6. For server execution, enqueue job
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
        let mut conn = state.redis.get_async_connection().await?;
        redis::cmd("LPUSH")
            .arg("simulation:jobs")
            .arg(job.id.to_string())
            .query_async(&mut conn)
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

pub async fn download_impulse_response(
    State(state): State<AppState>,
    claims: Claims,
    Path((project_id, sim_id, receiver_idx)): Path<(Uuid, Uuid, usize)>,
) -> Result<impl IntoResponse, ApiError> {
    // Verify access
    let sim = sqlx::query_as!(
        Simulation,
        "SELECT s.* FROM simulations s
         JOIN projects p ON s.project_id = p.id
         WHERE s.id = $1 AND s.project_id = $2
         AND (p.owner_id = $3 OR p.is_public = true OR p.org_id IN (
             SELECT org_id FROM org_members WHERE user_id = $3
         ))",
        sim_id,
        project_id,
        claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Simulation not found"))?;

    if sim.status != "completed" {
        return Err(ApiError::BadRequest("Simulation not completed"));
    }

    // Generate presigned S3 URL
    let s3_key = format!("impulse-responses/{}/{}/receiver_{}.wav",
        project_id, sim_id, receiver_idx);

    let presigned_url = state.s3
        .get_object()
        .bucket(&state.config.s3_bucket)
        .key(&s3_key)
        .presigned(
            aws_sdk_s3::presigning::PresigningConfig::expires_in(
                std::time::Duration::from_secs(3600)
            )?
        )
        .await?;

    Ok(Json(serde_json::json!({
        "url": presigned_url.uri().to_string(),
        "expires_in": 3600
    })))
}
```

### 2. Ray Tracing Core Solver (Rust — shared between WASM and native)

The core ray tracing engine that performs geometrical acoustics simulation with BVH acceleration for fast ray-triangle intersection.

```rust
// solver-core/src/ray_tracer.rs

use glam::{Vec3, Vec3A};
use bvh::aabb::{AABB, Bounded};
use bvh::bvh::BVH;
use bvh::ray::Ray;
use rand::prelude::*;
use rand_distr::UnitSphere;

use crate::geometry::{Triangle, Geometry, MaterialId};
use crate::materials::MaterialDatabase;
use crate::impulse_response::ImpulseResponse;

pub struct RayTracerConfig {
    pub ray_count: u32,
    pub max_reflection_order: u8,
    pub energy_threshold_db: f32,
    pub receiver_radius: f32,
    pub sample_rate: u32,
    pub max_time_ms: f32,
}

pub struct RayTracer {
    geometry: Geometry,
    materials: MaterialDatabase,
    bvh: BVH,
    config: RayTracerConfig,
}

impl RayTracer {
    pub fn new(geometry: Geometry, materials: MaterialDatabase, config: RayTracerConfig) -> Self {
        // Build BVH for fast ray-triangle intersection
        let bvh = BVH::build(&geometry.triangles);

        Self {
            geometry,
            materials,
            bvh,
            config,
        }
    }

    pub fn trace(
        &self,
        source_pos: Vec3,
        receiver_positions: &[Vec3],
        octave_band: OctaveBand,
    ) -> Vec<ImpulseResponse> {
        let mut impulse_responses: Vec<_> = receiver_positions
            .iter()
            .map(|&pos| ImpulseResponse::new(
                pos,
                self.config.sample_rate,
                (self.config.max_time_ms * self.config.sample_rate as f32 / 1000.0) as usize,
            ))
            .collect();

        let energy_per_ray = 1.0 / self.config.ray_count as f32;
        let energy_threshold = 10f32.powf(self.config.energy_threshold_db / 10.0);

        let mut rng = SmallRng::seed_from_u64(12345);

        for ray_idx in 0..self.config.ray_count {
            // Sample ray direction uniformly over sphere (Fibonacci spiral)
            let direction = self.sample_sphere_fibonacci(ray_idx, self.config.ray_count);

            self.trace_ray(
                source_pos,
                direction,
                energy_per_ray,
                0.0,  // path_length
                0,    // reflection_order
                &mut impulse_responses,
                energy_threshold,
                octave_band,
                &mut rng,
            );
        }

        impulse_responses
    }

    fn trace_ray(
        &self,
        origin: Vec3,
        direction: Vec3,
        energy: f32,
        path_length: f32,
        reflection_order: u8,
        impulse_responses: &mut [ImpulseResponse],
        energy_threshold: f32,
        octave_band: OctaveBand,
        rng: &mut SmallRng,
    ) {
        // Termination conditions
        if energy < energy_threshold {
            return;
        }
        if reflection_order >= self.config.max_reflection_order {
            return;
        }

        // Check receiver proximity and deposit energy
        for ir in impulse_responses.iter_mut() {
            let to_receiver = ir.position - origin;
            let distance_to_ray = to_receiver.cross(direction).length();

            if distance_to_ray < self.config.receiver_radius {
                let receiver_distance = to_receiver.length();
                let time_of_arrival = (path_length + receiver_distance) / 343.0;  // Speed of sound

                // Apply inverse square law attenuation
                let attenuated_energy = energy / (4.0 * std::f32::consts::PI * receiver_distance.powi(2));

                ir.add_energy(time_of_arrival, attenuated_energy);
            }
        }

        // Ray-scene intersection using BVH
        let ray = Ray::new(origin.into(), direction.into());
        let hit = self.bvh.traverse(&ray, &self.geometry.triangles);

        let Some((triangle, t)) = hit else {
            return;  // Ray escaped the room
        };

        let hit_point = origin + direction * t;
        let new_path_length = path_length + t;

        // Get material properties
        let material = &self.materials.get(triangle.material_id);
        let absorption = material.absorption_at_band(octave_band);
        let scattering = material.scattering_at_band(octave_band);

        // Apply absorption
        let reflected_energy = energy * (1.0 - absorption);

        // Decide between specular and diffuse reflection
        let reflect_diffuse = rng.gen::<f32>() < scattering;

        let new_direction = if reflect_diffuse {
            // Diffuse reflection (cosine-weighted hemisphere)
            self.sample_cosine_hemisphere(triangle.normal, rng)
        } else {
            // Specular reflection
            direction - 2.0 * direction.dot(triangle.normal) * triangle.normal
        };

        // Continue ray tracing
        self.trace_ray(
            hit_point + new_direction * 0.001,  // Offset to avoid self-intersection
            new_direction,
            reflected_energy,
            new_path_length,
            reflection_order + 1,
            impulse_responses,
            energy_threshold,
            octave_band,
            rng,
        );
    }

    fn sample_sphere_fibonacci(&self, i: u32, n: u32) -> Vec3 {
        let phi = std::f32::consts::PI * (1.0 + 5f32.sqrt()) * i as f32;  // Golden angle
        let cos_theta = 1.0 - 2.0 * i as f32 / n as f32;
        let sin_theta = (1.0 - cos_theta * cos_theta).sqrt();

        Vec3::new(
            sin_theta * phi.cos(),
            sin_theta * phi.sin(),
            cos_theta,
        )
    }

    fn sample_cosine_hemisphere(&self, normal: Vec3, rng: &mut SmallRng) -> Vec3 {
        // Sample from cosine-weighted hemisphere around normal
        let r1: f32 = rng.gen();
        let r2: f32 = rng.gen();

        let phi = 2.0 * std::f32::consts::PI * r1;
        let cos_theta = r2.sqrt();
        let sin_theta = (1.0 - r2).sqrt();

        // Local coordinates (tangent space)
        let tangent = if normal.x.abs() > 0.9 {
            Vec3::new(0.0, 1.0, 0.0)
        } else {
            Vec3::new(1.0, 0.0, 0.0)
        };
        let bitangent = normal.cross(tangent).normalize();
        let tangent = bitangent.cross(normal);

        // Transform to world space
        tangent * (sin_theta * phi.cos())
            + bitangent * (sin_theta * phi.sin())
            + normal * cos_theta
    }
}

// Implement BVH traits for Triangle
impl Bounded for Triangle {
    fn aabb(&self) -> AABB {
        let min_x = self.v0.x.min(self.v1.x).min(self.v2.x);
        let min_y = self.v0.y.min(self.v1.y).min(self.v2.y);
        let min_z = self.v0.z.min(self.v1.z).min(self.v2.z);
        let max_x = self.v0.x.max(self.v1.x).max(self.v2.x);
        let max_y = self.v0.y.max(self.v1.y).max(self.v2.y);
        let max_z = self.v0.z.max(self.v1.z).max(self.v2.z);

        AABB::with_bounds(
            Vec3A::new(min_x, min_y, min_z).into(),
            Vec3A::new(max_x, max_y, max_z).into(),
        )
    }
}

pub fn ray_triangle_intersect(ray: &Ray, tri: &Triangle) -> Option<f32> {
    // Möller–Trumbore intersection algorithm
    let edge1 = tri.v1 - tri.v0;
    let edge2 = tri.v2 - tri.v0;
    let h = ray.direction.cross(Vec3::from(edge2));
    let a = Vec3::from(edge1).dot(h);

    if a.abs() < 1e-8 {
        return None;  // Ray parallel to triangle
    }

    let f = 1.0 / a;
    let s = Vec3::from(ray.origin) - tri.v0;
    let u = f * s.dot(h);

    if u < 0.0 || u > 1.0 {
        return None;
    }

    let q = s.cross(Vec3::from(edge1));
    let v = f * Vec3::from(ray.direction).dot(q);

    if v < 0.0 || u + v > 1.0 {
        return None;
    }

    let t = f * Vec3::from(edge2).dot(q);

    if t > 1e-8 {
        Some(t)
    } else {
        None
    }
}
```

### 3. Room Acoustic Parameters Calculation (Rust)

Post-processing module that computes standardized room acoustic parameters from impulse responses.

```rust
// solver-core/src/acoustic_parameters.rs

use crate::impulse_response::ImpulseResponse;
use crate::db::models::{RoomAcousticParameters, OctaveBandValues, OctaveBand};

pub struct ParameterCalculator {
    sample_rate: u32,
}

impl ParameterCalculator {
    pub fn new(sample_rate: u32) -> Self {
        Self { sample_rate }
    }

    pub fn calculate(&self, impulse_responses: &[Vec<ImpulseResponse>]) -> RoomAcousticParameters {
        // impulse_responses[band_idx][receiver_idx]

        let mut rt60 = OctaveBandValues::default();
        let mut edt = OctaveBandValues::default();
        let mut c50 = OctaveBandValues::default();
        let mut c80 = OctaveBandValues::default();
        let mut d50 = OctaveBandValues::default();
        let mut g = OctaveBandValues::default();
        let mut lf = OctaveBandValues::default();

        for (band_idx, band) in OctaveBand::all().iter().enumerate() {
            let irs = &impulse_responses[band_idx];

            // Average across all receivers
            let mut rt60_sum = 0.0;
            let mut edt_sum = 0.0;
            let mut c50_sum = 0.0;
            let mut c80_sum = 0.0;
            let mut d50_sum = 0.0;
            let mut g_sum = 0.0;

            for ir in irs {
                rt60_sum += self.calculate_rt60(ir);
                edt_sum += self.calculate_edt(ir);
                c50_sum += self.calculate_clarity(ir, 0.050);
                c80_sum += self.calculate_clarity(ir, 0.080);
                d50_sum += self.calculate_definition(ir, 0.050);
                g_sum += self.calculate_strength(ir);
            }

            let n = irs.len() as f32;
            rt60.set(*band, rt60_sum / n);
            edt.set(*band, edt_sum / n);
            c50.set(*band, c50_sum / n);
            c80.set(*band, c80_sum / n);
            d50.set(*band, d50_sum / n);
            g.set(*band, g_sum / n);
        }

        // STI calculation (speech transmission index) - broadband metric
        let sti = self.calculate_sti(&impulse_responses);

        RoomAcousticParameters {
            rt60,
            edt,
            c50,
            c80,
            d50,
            sti,
            g,
            lf,
        }
    }

    fn calculate_rt60(&self, ir: &ImpulseResponse) -> f32 {
        // Use T20 method (extrapolate -5 to -25 dB decay to 60 dB)
        let energy_curve = self.energy_decay_curve(ir);

        // Find times for -5 dB and -25 dB
        let peak_energy = energy_curve.iter().copied().fold(0.0f32, f32::max);
        let t5 = self.find_time_for_level(&energy_curve, peak_energy, -5.0);
        let t25 = self.find_time_for_level(&energy_curve, peak_energy, -25.0);

        // Extrapolate to 60 dB
        let rt60 = 3.0 * (t25 - t5);
        rt60.max(0.0)
    }

    fn calculate_edt(&self, ir: &ImpulseResponse) -> f32 {
        // Early Decay Time: 0 to -10 dB
        let energy_curve = self.energy_decay_curve(ir);
        let peak_energy = energy_curve.iter().copied().fold(0.0f32, f32::max);

        let t0 = self.find_time_for_level(&energy_curve, peak_energy, 0.0);
        let t10 = self.find_time_for_level(&energy_curve, peak_energy, -10.0);

        6.0 * (t10 - t0)  // Extrapolate to 60 dB
    }

    fn calculate_clarity(&self, ir: &ImpulseResponse, boundary_time: f32) -> f32 {
        // Clarity: C_t = 10 log10(E_early / E_late)
        let boundary_sample = (boundary_time * self.sample_rate as f32) as usize;

        let early_energy: f32 = ir.samples[..boundary_sample.min(ir.samples.len())]
            .iter()
            .map(|&s| s * s)
            .sum();

        let late_energy: f32 = ir.samples[boundary_sample.min(ir.samples.len())..]
            .iter()
            .map(|&s| s * s)
            .sum();

        if late_energy < 1e-10 {
            return 50.0;  // Very high clarity
        }

        10.0 * (early_energy / late_energy).log10()
    }

    fn calculate_definition(&self, ir: &ImpulseResponse, boundary_time: f32) -> f32 {
        // Definition: D_t = E_early / E_total
        let boundary_sample = (boundary_time * self.sample_rate as f32) as usize;

        let early_energy: f32 = ir.samples[..boundary_sample.min(ir.samples.len())]
            .iter()
            .map(|&s| s * s)
            .sum();

        let total_energy: f32 = ir.samples.iter().map(|&s| s * s).sum();

        if total_energy < 1e-10 {
            return 1.0;
        }

        early_energy / total_energy
    }

    fn calculate_strength(&self, ir: &ImpulseResponse) -> f32 {
        // Strength: G = 10 log10(E / E_ref)
        // where E_ref is energy at 10m in free field
        let total_energy: f32 = ir.samples.iter().map(|&s| s * s).sum();

        // Reference energy at 10m: E_ref = 1 / (4π × 10²)
        let e_ref = 1.0 / (4.0 * std::f32::consts::PI * 100.0);

        10.0 * (total_energy / e_ref).log10()
    }

    fn calculate_sti(&self, impulse_responses: &[Vec<ImpulseResponse>]) -> f32 {
        // Speech Transmission Index per IEC 60268-16
        // Simplified implementation: compute MTF for 7 octave bands × 14 modulation frequencies

        let modulation_freqs = [0.63, 0.8, 1.0, 1.25, 1.6, 2.0, 2.5, 3.15, 4.0, 5.0, 6.3, 8.0, 10.0, 12.5];
        let band_weights = [0.13, 0.14, 0.11, 0.12, 0.19, 0.17, 0.14];  // Speech spectrum weighting

        let mut sti_sum = 0.0;

        for (band_idx, &weight) in band_weights.iter().enumerate() {
            let irs = &impulse_responses[band_idx];
            let mut mtf_sum = 0.0;

            for &f_mod in &modulation_freqs {
                // Compute MTF for this modulation frequency
                let mtf = self.compute_mtf(&irs[0], f_mod);  // Use first receiver for simplicity

                // Convert MTF to apparent SNR
                let snr_apparent = 10.0 * (mtf / (1.0 - mtf)).log10();

                // Clip to [-15, +15] dB and normalize to [0, 1]
                let ti = (snr_apparent + 15.0) / 30.0;
                let ti = ti.max(0.0).min(1.0);

                mtf_sum += ti;
            }

            let mtf_avg = mtf_sum / modulation_freqs.len() as f32;
            sti_sum += weight * mtf_avg;
        }

        sti_sum
    }

    fn compute_mtf(&self, ir: &ImpulseResponse, f_mod: f32) -> f32 {
        // Modulation Transfer Function at frequency f_mod
        let mut real_sum = 0.0;
        let mut imag_sum = 0.0;
        let mut energy_sum = 0.0;

        for (i, &sample) in ir.samples.iter().enumerate() {
            let t = i as f32 / self.sample_rate as f32;
            let energy = sample * sample;

            real_sum += energy * (2.0 * std::f32::consts::PI * f_mod * t).cos();
            imag_sum += energy * (2.0 * std::f32::consts::PI * f_mod * t).sin();
            energy_sum += energy;
        }

        if energy_sum < 1e-10 {
            return 0.0;
        }

        let m = (real_sum * real_sum + imag_sum * imag_sum).sqrt() / energy_sum;
        m.min(1.0)
    }

    fn energy_decay_curve(&self, ir: &ImpulseResponse) -> Vec<f32> {
        // Compute cumulative energy decay curve (Schroeder integration)
        let n = ir.samples.len();
        let mut curve = vec![0.0; n];

        let mut cumulative = 0.0;
        for i in (0..n).rev() {
            cumulative += ir.samples[i] * ir.samples[i];
            curve[i] = cumulative;
        }

        curve
    }

    fn find_time_for_level(&self, energy_curve: &[f32], peak_energy: f32, level_db: f32) -> f32 {
        let target_energy = peak_energy * 10f32.powf(level_db / 10.0);

        for (i, &energy) in energy_curve.iter().enumerate() {
            if energy <= target_energy {
                return i as f32 / self.sample_rate as f32;
            }
        }

        energy_curve.len() as f32 / self.sample_rate as f32
    }
}

impl Default for OctaveBandValues {
    fn default() -> Self {
        Self {
            hz_125: 0.0,
            hz_250: 0.0,
            hz_500: 0.0,
            hz_1k: 0.0,
            hz_2k: 0.0,
            hz_4k: 0.0,
            hz_8k: 0.0,
        }
    }
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–5)

**Day 1: Project initialization and Rust backend scaffold**
```bash
cargo init acoustica-api
cd acoustica-api
cargo add axum tokio serde serde_json sqlx uuid chrono tower-http jsonwebtoken bcrypt
cargo add --features postgres,runtime-tokio-rustls,uuid,chrono,json,postgis sqlx
cargo add aws-sdk-s3 redis
```
- `src/main.rs` — Axum app with CORS, tracing, graceful shutdown
- `src/config.rs` — Environment-based configuration (DATABASE_URL, JWT_SECRET, S3_BUCKET, REDIS_URL)
- `src/state.rs` — AppState struct (PgPool, Redis, S3Client)
- `src/error.rs` — ApiError enum with IntoResponse
- `Dockerfile` — Multi-stage build (builder + runtime)
- `docker-compose.yml` — PostgreSQL + PostGIS, Redis, MinIO (S3-compatible)

**Day 2: Database schema and migrations**
- `migrations/001_initial.sql` — All 9 tables: users, organizations, org_members, projects, simulations, simulation_jobs, acoustic_materials, source_directivity, impulse_responses, usage_records, noise_sources
- `src/db/mod.rs` — Database pool initialization
- `src/db/models.rs` — All SQLx structs with FromRow derives
- Run `sqlx migrate run` to apply schema
- Seed script for initial material database (50 common materials: concrete, drywall, carpet, wood, fabric, etc.)

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
- Project types: room, environmental, building, sound_system
- `src/api/router.rs` — All route definitions with auth middleware
- Integration tests: auth flow, project CRUD, authorization checks

**Day 5: Material database API**
- `src/api/handlers/materials.rs` — List materials (paginated), search by name/category, get material details
- Full-text search using PostgreSQL trigram similarity
- Filter by category (absorber, diffuser, reflector)
- Return absorption and scattering coefficients for all octave bands
- Seed additional 500+ materials from database (ODEON material library, acoustic.ua database)

### Phase 2 — Solver Core — Ray Tracer + ISM (Days 6–14)

**Day 6: Geometry foundation**
- `solver-core/` — New Rust workspace member (shared between native and WASM)
- `solver-core/src/geometry.rs` — Triangle, Mesh, Geometry structs
- Vertex, face, normal, material ID storage
- OBJ file parser for importing 3D models
- Validation: check for watertight mesh, consistent normals, degenerate triangles
- Unit tests: load simple box room, sphere

**Day 7: BVH acceleration structure**
- `solver-core/src/bvh.rs` — Bounding Volume Hierarchy for fast ray-triangle intersection
- Use `bvh` crate for construction and traversal
- Implement Bounded trait for Triangle
- Ray-triangle intersection using Möller–Trumbore algorithm
- Benchmark: 1M rays × 1000 triangles in <2 seconds

**Day 8: Material database and properties**
- `solver-core/src/materials.rs` — MaterialDatabase struct, Material struct
- Load absorption and scattering coefficients from JSON
- Interpolation for intermediate frequencies (if needed)
- Air absorption calculation (ISO 9613-1)
- Tests: verify material properties match reference data

**Day 9: Ray tracer — basic implementation**
- `solver-core/src/ray_tracer.rs` — RayTracer struct with trace() method
- Fibonacci sphere sampling for uniform ray distribution
- Ray-scene intersection using BVH
- Specular reflection calculation
- Energy deposition at receiver positions
- Tests: single ray, multiple reflections, energy conservation

**Day 10: Ray tracer — diffuse scattering**
- Add scattering coefficient support
- Cosine-weighted hemisphere sampling for diffuse reflections
- Hybrid specular + diffuse reflection based on scattering coefficient
- Lambert's cosine law for diffuse energy distribution
- Tests: compare diffuse vs. specular RT60 in shoebox room

**Day 11: Impulse response generation**
- `solver-core/src/impulse_response.rs` — ImpulseResponse struct
- Time-domain binning of ray arrivals
- Sample rate: 48 kHz (bin width ≈ 0.02 ms)
- Energy-to-pressure conversion
- Schroeder integration for energy decay curve
- Export as WAV file (mono, 32-bit float)
- Tests: verify impulse response energy matches ray energy

**Day 12: Image source method (ISM)**
- `solver-core/src/image_source.rs` — ImageSourceMethod struct
- Recursive image source generation up to order N
- Mirror point calculation across planes
- Visibility validation (source sees surface, reflection point sees receiver)
- Path length and time-of-arrival calculation
- Combine with ray tracing: ISM for orders 1-3, rays for 4+
- Tests: compare ISM RT60 to analytical Sabine equation for simple room

**Day 13: Room acoustic parameters**
- `solver-core/src/acoustic_parameters.rs` — ParameterCalculator struct
- RT60 calculation (T20, T30 methods with linear regression)
- EDT (Early Decay Time)
- C50, C80 (Clarity)
- D50, D80 (Definition)
- G (Strength)
- Schroeder frequency calculation
- Tests: validate against known reference rooms (ISO 3382 test cases)

**Day 14: STI calculation**
- Speech Transmission Index (STI) per IEC 60268-16
- Modulation Transfer Function (MTF) computation
- 7 octave bands × 14 modulation frequencies
- Speech spectrum weighting
- Apparent SNR calculation
- Tests: verify STI for good/fair/poor speech intelligibility cases

### Phase 3 — WASM Build + Frontend Visualization (Days 15–21)

**Day 15: WASM solver compilation**
- `solver-wasm/` — New workspace member for WASM target
- `solver-wasm/Cargo.toml` — wasm-bindgen, serde-wasm-bindgen dependencies
- `solver-wasm/src/lib.rs` — WASM entry points: `trace_rays()`, `calculate_parameters()`
- `wasm-pack build --target web --release` pipeline
- Size optimization: target <3MB gzipped
- JavaScript wrapper: `AcousticaSolver` class that loads WASM and provides async API
- Tests: verify WASM produces identical results to native

**Day 16: Frontend scaffold and Three.js setup**
```bash
npm create vite@latest frontend -- --template react-ts
cd frontend
npm i zustand @tanstack/react-query axios three @react-three/fiber @react-three/drei
```
- `src/App.tsx` — Router, auth context, layout
- `src/stores/projectStore.ts` — Zustand store for project state
- `src/stores/geometryStore.ts` — Room geometry state (vertices, faces, materials)
- `src/components/RoomEditor/Scene.tsx` — React Three Fiber scene with camera, lights, grid
- `src/components/RoomEditor/Controls.tsx` — Orbit controls for pan/zoom/rotate
- Basic room rendering: load geometry from project data

**Day 17: Room editor — geometry manipulation**
- `src/components/RoomEditor/GeometryPalette.tsx` — Preset room shapes (shoebox, L-shape, theater)
- `src/components/RoomEditor/TransformControls.tsx` — Translate/rotate/scale selected surfaces
- `src/components/RoomEditor/MaterialPainter.tsx` — Select material and paint onto surfaces
- OBJ import: drag-and-drop OBJ files onto canvas
- Snap-to-grid for vertex editing
- Tests: create shoebox room, assign materials, export geometry

**Day 18: Room editor — source and receiver placement**
- `src/components/RoomEditor/SourceMarker.tsx` — 3D sphere marker for sound sources
- `src/components/RoomEditor/ReceiverMarker.tsx` — 3D cone marker for receivers
- Click-to-place mode for sources/receivers
- Position editing with transform controls
- Source directivity pattern preview (visualize polar pattern)
- Tests: place multiple sources and receivers, save positions

**Day 19: Simulation panel and parameter controls**
- `src/components/SimulationPanel.tsx` — Simulation configuration sidebar
- Ray count slider (1k - 10M rays)
- Max reflection order input (1-100)
- Energy threshold slider (-40 to -80 dB)
- Temperature, humidity, pressure controls (for air absorption)
- Run button → triggers simulation API call
- Plan-based limit warnings (free: 100k rays max)

**Day 20: Results visualization — ray paths**
- `src/components/ResultsViewer/RayPathsViewer.tsx` — Render ray paths as line segments
- GPU instanced rendering for 10k+ rays
- Color coding by reflection order or energy
- Opacity/visibility controls
- Animation: show rays propagating over time
- Tests: render 10k rays in 60fps

**Day 21: Results visualization — SPL heat map**
- `src/components/ResultsViewer/SPLHeatMap.tsx` — 3D heat map overlay on room surfaces
- Grid-based SPL calculation (interpolate from receiver positions)
- Color scale: blue (low) → red (high)
- Octave band selector (125 Hz - 8 kHz)
- Export heat map as image
- Tests: verify SPL distribution matches simulation

### Phase 4 — API + Job Orchestration (Days 22–28)

**Day 22: Simulation API endpoints**
- `src/api/handlers/simulation.rs` — Create simulation, get simulation, list simulations, cancel simulation
- Geometry validation: check for watertight mesh, consistent normals
- Ray count and triangle count extraction
- WASM/server routing logic (1M ray threshold)
- Plan-based limits enforcement (free: 100k rays, 1k triangles)
- Tests: create simulation, verify routing decision

**Day 23: Server-side simulation worker**
- `src/workers/simulation_worker.rs` — Redis job consumer, runs native solver
- Multi-threaded ray tracing (Rayon parallel iterators)
- Progress streaming: worker publishes progress → Redis pub/sub → WebSocket to client
- Error handling: geometry errors, out-of-memory, timeouts
- S3 result upload: impulse responses (WAV files), parameters (JSON)
- Tests: end-to-end simulation on server

**Day 24: WebSocket for live simulation progress**
- `src/api/ws/mod.rs` — Axum WebSocket upgrade handler
- `src/api/ws/simulation_progress.rs` — Subscribe to simulation progress channel
- Client receives: `{ progress_pct, rays_traced, energy_remaining }`
- Connection management: heartbeat ping/pong, automatic cleanup on disconnect
- Frontend: `src/hooks/useSimulationProgress.ts` — React hook for WebSocket subscription
- Tests: verify progress updates received

**Day 25: Impulse response download and auralization**
- `src/api/handlers/impulse_response.rs` — Download impulse response for receiver
- Presigned S3 URLs (1 hour expiration)
- Frontend: `src/components/Auralization/Player.tsx` — Web Audio API convolution player
- Convolve dry source signal (speech, music) with impulse response
- Binaural rendering with HRTF (MIT KEMAR database)
- Playback controls: play/pause, volume, source signal selector
- Tests: verify auralization produces correct output

**Day 26: HRTF binaural rendering**
- `frontend/public/hrtf/` — MIT KEMAR HRTF database (SOFA format)
- `src/lib/hrtf.ts` — Load HRTF data, interpolate for arbitrary azimuth/elevation
- `src/components/Auralization/BinauralRenderer.tsx` — Apply HRTF to mono impulse response
- Two-channel output for headphone listening
- Interactive mode: move listener position, hear changes in real-time
- Tests: verify HRTF produces correct spatial perception

**Day 27: Parameter results display**
- `src/components/ResultsViewer/ParametersTable.tsx` — Display RT60, EDT, C50, C80, D50, STI
- Octave band chart (line chart for RT60 vs. frequency)
- Comparison mode: before/after treatment
- Export to CSV
- Tests: verify parameters match simulation output

**Day 28: Report generation (Python service)**
- `report-service/` — Python FastAPI service
- `report-service/generate_report.py` — PDF generation using ReportLab
- Sections: project info, geometry, materials, simulation parameters, results (parameters table, octave band charts, SPL maps)
- Matplotlib for octave band charts
- S3 upload for generated PDF
- Tests: generate PDF, verify structure and content

### Phase 5 — Optimization + Polish (Days 29–35)

**Day 29: Material optimization — foundation**
- `optimization-service/` — Python FastAPI service
- `optimization-service/optimizer.py` — Bayesian optimization for material placement
- Objective function: minimize weighted sum of (RT60 error, C80 error, cost)
- Constraints: budget, available wall area
- Use `scikit-optimize` for Gaussian process optimization
- Tests: optimize simple room, verify convergence

**Day 30: Material optimization — integration**
- `src/api/handlers/optimization.rs` — Start optimization job, get optimization status
- Optimization parameters: target RT60 (per octave band), budget, surface constraints
- Worker: call Python optimization service, run multiple simulations per iteration
- Return recommended material layout (surface ID → material ID mapping)
- Frontend: `src/components/Optimization/OptimizationWizard.tsx` — Wizard UI for target specification
- Tests: end-to-end optimization, verify recommendations

**Day 31: Performance optimization — WASM**
- Profile WASM solver with browser DevTools
- Optimize hot paths: ray-triangle intersection, BVH traversal
- SIMD optimizations for vector math (use `glam` crate)
- Parallel ray tracing using Web Workers (spawn 4-8 workers)
- Target: 1M rays in <3 seconds
- Tests: benchmark before/after optimizations

**Day 32: Performance optimization — server**
- Profile native solver with `cargo flamegraph`
- Multi-threading with Rayon (parallel ray tracing across cores)
- BVH build optimization (surface area heuristic)
- Memory pool for ray allocations (reduce allocator overhead)
- Target: 10M rays in <30 seconds on 8-core server
- Tests: benchmark scaling with ray count and core count

**Day 33: UI/UX polish**
- `src/components/Dashboard.tsx` — Project dashboard with cards, filters, search
- Project templates: classroom, conference room, concert hall, recording studio
- Keyboard shortcuts: S (add source), R (add receiver), M (material painter), Space (run simulation)
- Tooltips and help text throughout UI
- Loading states and skeleton screens
- Tests: visual regression tests with Playwright

**Day 34: Error handling and validation**
- Comprehensive error messages for geometry validation
- Plan limit warnings with upgrade prompts
- Retry logic for transient API failures
- Fallback to server if WASM fails (browser compatibility)
- Simulation timeout handling
- Tests: error scenarios, validation edge cases

**Day 35: Documentation and onboarding**
- `docs/` — User documentation (getting started, room editor, simulation, results)
- In-app tutorial (Shepherd.js) — First-time user walkthrough
- Example projects: load pre-built rooms with results
- Video tutorials (screen recordings) for complex workflows
- API documentation (OpenAPI/Swagger)
- Tests: verify documentation examples work

### Phase 6 — Billing + Deployment (Days 36–40)

**Day 36: Stripe integration**
- `src/api/handlers/billing.rs` — Create checkout session, customer portal, webhook handler
- Plans: Free (100k rays, 3 projects) / Professional ($129/mo, unlimited) / Advanced ($299/user/mo, collaboration)
- Stripe webhook: handle subscription events (created, updated, cancelled)
- Plan enforcement in simulation API (check ray count limits)
- Tests: simulate subscription lifecycle

**Day 37: Usage tracking**
- `src/api/middleware/usage_tracking.rs` — Track simulation minutes, storage bytes
- Insert usage records on simulation completion
- Monthly aggregation for billing enforcement
- Dashboard: usage charts (rays traced, simulations run, storage used)
- Tests: verify usage records created correctly

**Day 38: Production infrastructure**
- `terraform/` — Infrastructure as code (AWS: ECS, RDS, ElastiCache, S3, CloudFront)
- Multi-region deployment (us-east-1, eu-west-1)
- Auto-scaling: ECS services scale on CPU usage
- RDS PostgreSQL with read replicas
- Redis cluster for job queue
- CloudFront CDN for WASM and frontend assets
- Tests: smoke tests in staging environment

**Day 39: Monitoring and observability**
- Prometheus metrics: API request rates, simulation times, job queue depth
- Grafana dashboards: system health, user activity, error rates
- Sentry for error tracking and alerting
- Structured logging (JSON format) with trace IDs
- Alerting rules: high error rate, long job queue, database lag
- Tests: verify metrics collected and dashboards render

**Day 40: CI/CD pipeline**
```yaml
# .github/workflows/ci.yml
- Rust tests (cargo test)
- WASM build (wasm-pack)
- Frontend tests (Vitest)
- Docker image build and push (ECR)
- Terraform apply (staging)
- E2E tests (Playwright)
- Deploy to production (manual approval)
```
- Automated testing on every PR
- Staging deployment on merge to main
- Production deployment on release tag
- Tests: verify CI/CD pipeline runs successfully

### Phase 7 — Environmental Noise (Post-MVP, Days 41-42)

**Day 41: Environmental noise propagation — ISO 9613**
- `solver-core/src/environmental_noise/iso9613.rs` — ISO 9613-2 outdoor propagation
- Point sources, line sources, area sources
- Geometric divergence, air absorption, ground attenuation, barrier attenuation
- Meteorological corrections
- Grid receiver array for noise maps
- Tests: validate against ISO 9613 reference calculations

**Day 42: GIS integration and noise maps**
- `src/api/handlers/gis.rs` — Import terrain DEM (GeoTIFF), building footprints (Shapefile)
- PostGIS spatial queries for receiver positions
- Frontend: `src/components/NoiseMap.tsx` — 2D noise contour map (Mapbox GL JS)
- Color-coded Lden contours per EU directive
- Export noise map as GeoTIFF
- Tests: generate noise map for road traffic source

---

## Validation and Success Metrics

### Technical Validation Benchmarks

1. **RT60 Accuracy vs. Measurements**
   - Test rooms: Classroom (V=200m³), Concert Hall (V=12000m³)
   - Method: Run simulation with measured room geometry and materials, compare simulated RT60 to measured values
   - Target: <5% error for 500 Hz - 2 kHz bands, <10% error for 125 Hz and 8 kHz
   - Current industry tools (ODEON): 3-7% error typical
   - Validation: Use published data from ISO 3382 test rooms

2. **Simulation Performance**
   - WASM (1M rays, 500 triangles): <5 seconds on modern laptop (M1 MacBook Pro baseline)
   - Server (10M rays, 5000 triangles): <30 seconds on 8-core instance (c6i.2xlarge)
   - Memory usage: <2GB RAM for 10M rays
   - BVH build time: <1 second for 10k triangles

3. **STI Prediction Accuracy**
   - Test room: Typical classroom with known STI (measured)
   - Method: Compare predicted STI to IEC 60268-16 reference measurement
   - Target: ±0.05 STI units (STI scale: 0-1)
   - Validation: Use NIST classroom acoustics dataset

4. **Auralization Perceptual Quality**
   - Method: A/B listening test with measured vs. simulated impulse responses
   - Metric: Subjective difference grade (SDG) per ITU-R BS.1116
   - Target: SDG >4 (imperceptible difference) for speech signals
   - Validation: 20+ listener panel with acoustics background

5. **Environmental Noise Accuracy (Post-MVP)**
   - Test case: Road traffic noise at 100m distance
   - Method: Compare simulated Lday to ISO 9613 reference calculation
   - Target: ±2 dB(A) for flat terrain, ±3 dB(A) with barriers
   - Validation: Use FHWA TNM validation test cases

---

## Post-MVP Roadmap (v1.1+)

### v1.1 — Environmental Noise (Weeks 13-16)
- ISO 9613-2, CNOSSOS-EU, FHWA TNM propagation models
- GIS integration (terrain DEMs, building footprints from OpenStreetMap)
- Road, rail, industrial, aircraft noise sources
- Barrier attenuation and ground effects
- Lday, Levening, Lnight, Lden contour maps
- Compliance reporting (WHO, EU directive)

### v1.2 — Sound System Design (Weeks 17-20)
- Loudspeaker placement and aiming
- CLF/GLL directivity data import (JBL, L-Acoustics, d&b, Meyer Sound)
- SPL coverage maps at listener plane
- Speech intelligibility with sound system (STI, ALCONS)
- Delay alignment and signal processing
- Subwoofer placement optimization

### v1.3 — Building Acoustics (Weeks 21-24)
- Airborne sound insulation (EN 12354-1, ISO 15712-1)
- Impact sound insulation (EN 12354-2)
- Flanking transmission analysis
- Wall/floor assembly database (STC, Rw, Ln,w)
- HVAC noise prediction
- Code compliance checking (IBC, Part E, DIN 4109)

### v1.4 — Advanced Features (Weeks 25-28)
- Wave-based FEM solver for low frequencies (<80 Hz)
- Ambisonics spatial audio rendering
- AI-powered material selection (ML model trained on 10k+ material combinations)
- Collaborative editing (real-time multi-user)
- API access (REST + GraphQL)
- White-label reports (custom branding)

---

## Known Limitations and Future Work

### Current MVP Limitations

1. **Geometrical acoustics only** — No wave-based phenomena below Schroeder frequency (~80-200 Hz). Room modes and low-frequency resonances not captured. Future: Add FEM/BEM solver for low frequencies.

2. **No diffraction modeling** — Sound bending around edges not simulated. Important for barrier attenuation and obstruction effects. Future: Add UTD (Uniform Theory of Diffraction) edge diffraction.

3. **Simplified scattering** — Lambert diffuse scattering model. Real surfaces have frequency-dependent and angle-dependent scattering. Future: Implement directional scattering (BRDFs for acoustics).

4. **Single source at a time** — MVP simulates one source per simulation run. Multiple uncorrelated sources require separate simulations. Future: Add multi-source simulation with coherence control.

5. **No moving sources** — Stationary sources only. No Doppler shift modeling. Future: Add time-varying source positions for moving source simulations (vehicles, performers).

6. **Limited directivity patterns** — Omnidirectional and simple patterns only in MVP. No measured loudspeaker data import yet. Added in v1.2 (Sound System Design).

### Performance Considerations

- **Large venues** (stadiums, airports >100k m³): Require >10M rays for converged RT60, server-side execution only
- **Detailed geometry** (>10k triangles): BVH build time increases, consider mesh simplification
- **Real-time auralization** — Interactive mode limited to pre-computed impulse responses, cannot update in real-time for geometry changes
- **Memory usage** — 10M rays × 100 reflections = ~10GB ray path storage. Use streaming to S3 for very large simulations.

---

## Critical Files

```
acoustica/
├── solver-core/                           # Shared solver library (native + WASM)
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── geometry.rs                    # Triangle, Mesh, OBJ import
│   │   ├── bvh.rs                         # BVH acceleration structure
│   │   ├── materials.rs                   # Material database, properties
│   │   ├── ray_tracer.rs                  # Ray tracing engine
│   │   ├── image_source.rs                # Image source method
│   │   ├── impulse_response.rs            # IR generation and export
│   │   ├── acoustic_parameters.rs         # RT60, C50, STI calculation
│   │   ├── air_absorption.rs              # ISO 9613-1 air absorption
│   │   └── environmental_noise/           # Post-MVP: outdoor propagation
│   │       ├── iso9613.rs                 # ISO 9613-2
│   │       └── cnossos.rs                 # CNOSSOS-EU
│   └── tests/
│       ├── validation.rs                  # Solver validation tests
│       └── benchmarks.rs                  # Performance benchmarks
│
├── solver-wasm/                           # WASM compilation target
│   ├── Cargo.toml
│   └── src/
│       └── lib.rs                         # WASM entry points (wasm-bindgen)
│
├── acoustica-api/                         # Rust API server
│   ├── Cargo.toml
│   ├── src/
│   │   ├── main.rs                        # Axum app setup
│   │   ├── config.rs                      # Environment config
│   │   ├── state.rs                       # AppState (PgPool, Redis, S3)
│   │   ├── error.rs                       # ApiError types
│   │   ├── auth/
│   │   │   ├── mod.rs                     # JWT middleware
│   │   │   └── oauth.rs                   # Google/GitHub OAuth
│   │   ├── api/
│   │   │   ├── router.rs                  # Route definitions
│   │   │   ├── handlers/
│   │   │   │   ├── auth.rs                # Register, login
│   │   │   │   ├── users.rs               # Profile CRUD
│   │   │   │   ├── projects.rs            # Project CRUD
│   │   │   │   ├── simulation.rs          # Create/get simulation
│   │   │   │   ├── materials.rs           # Material search
│   │   │   │   ├── impulse_response.rs    # IR download
│   │   │   │   ├── billing.rs             # Stripe checkout
│   │   │   │   ├── optimization.rs        # Material optimization
│   │   │   │   └── webhooks/
│   │   │   │       └── stripe.rs          # Stripe webhooks
│   │   │   └── ws/
│   │   │       └── simulation_progress.rs # Live progress
│   │   ├── middleware/
│   │   │   └── plan_limits.rs             # Plan enforcement
│   │   ├── services/
│   │   │   ├── usage.rs                   # Usage tracking
│   │   │   └── s3.rs                      # S3 helpers
│   │   └── workers/
│   │       ├── mod.rs                     # Worker pool
│   │       └── simulation_worker.rs       # Server-side execution
│   ├── migrations/
│   │   └── 001_initial.sql                # Database schema
│   └── tests/
│       ├── api_integration.rs
│       └── simulation_e2e.rs
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
│   │   │   ├── geometryStore.ts
│   │   │   └── simulationStore.ts
│   │   ├── hooks/
│   │   │   ├── useSimulationProgress.ts
│   │   │   └── useWasmSolver.ts
│   │   ├── lib/
│   │   │   ├── api.ts                     # API client
│   │   │   ├── wasmLoader.ts              # WASM loader
│   │   │   └── hrtf.ts                    # HRTF processing
│   │   ├── pages/
│   │   │   ├── Dashboard.tsx
│   │   │   ├── Editor.tsx                 # 3D editor + results
│   │   │   ├── Materials.tsx
│   │   │   ├── Templates.tsx
│   │   │   ├── Billing.tsx
│   │   │   ├── Login.tsx
│   │   │   └── Register.tsx
│   │   └── components/
│   │       ├── RoomEditor/
│   │       │   ├── Scene.tsx              # Three.js scene
│   │       │   ├── Controls.tsx           # Orbit controls
│   │       │   ├── GeometryPalette.tsx
│   │       │   ├── TransformControls.tsx
│   │       │   ├── MaterialPainter.tsx
│   │       │   ├── SourceMarker.tsx
│   │       │   └── ReceiverMarker.tsx
│   │       ├── SimulationPanel.tsx
│   │       ├── ResultsViewer/
│   │       │   ├── RayPathsViewer.tsx
│   │       │   ├── SPLHeatMap.tsx
│   │       │   └── ParametersTable.tsx
│   │       ├── Auralization/
│   │       │   ├── Player.tsx             # Web Audio player
│   │       │   └── BinauralRenderer.tsx   # HRTF rendering
│   │       ├── Optimization/
│   │       │   └── OptimizationWizard.tsx
│   │       ├── billing/
│   │       │   ├── PlanCard.tsx
│   │       │   └── UsageMeter.tsx
│   │       └── common/
│   │           └── MaterialSearch.tsx
│   └── public/
│       ├── wasm/                          # WASM bundle
│       └── hrtf/                          # HRTF database (MIT KEMAR)
│
├── optimization-service/                  # Python FastAPI
│   ├── requirements.txt
│   ├── main.py
│   ├── optimizer.py                       # Bayesian optimization
│   └── Dockerfile
│
├── report-service/                        # Python FastAPI
│   ├── requirements.txt
│   ├── main.py
│   ├── generate_report.py                 # PDF generation
│   └── Dockerfile
│
├── terraform/                             # Infrastructure as code
│   ├── main.tf
│   ├── ecs.tf                            # ECS services
│   ├── rds.tf                            # PostgreSQL
│   ├── elasticache.tf                    # Redis
│   └── cloudfront.tf                     # CDN
│
├── docker-compose.yml                     # Local dev stack
├── Cargo.toml                             # Workspace root
└── .github/
    └── workflows/
        ├── ci.yml
        ├── wasm-build.yml
        └── deploy.yml
```

---

## Solver Validation Suite

### Benchmark 1: RT60 Accuracy vs. Sabine Equation (Shoebox Room)

**Room:** 10m × 8m × 3m = 240 m³, S = 236 m², all surfaces α = 0.10 (absorption coefficient)

**Expected (Sabine):** RT60 = 0.161 × V / (S × α_avg) = 0.161 × 240 / (236 × 0.10) = **1.64 seconds**

**Method:** Run hybrid simulation (ISM + ray tracing) with 500k rays

**Tolerance:** < 5% error (expected: 1.56 - 1.72 seconds)

### Benchmark 2: C50 vs. Measured Data (Classroom)

**Room:** Real classroom (V=200 m³) with measured geometry and materials

**Expected:** C50 at 1 kHz = **+3.5 dB** (from published measurement data, ISO 3382)

**Method:** Simulate with 1M rays, measure C50 at reference receiver position

**Tolerance:** < 1 dB error (expected: +2.5 to +4.5 dB)

### Benchmark 3: STI Prediction Accuracy

**Room:** Standard classroom with speech source at front, receiver at back

**Expected:** STI = **0.68** (from IEC 60268-16 reference measurement)

**Method:** Compute MTF for all bands, calculate STI with speech weighting

**Tolerance:** ±0.05 STI units (expected: 0.63 - 0.73)

### Benchmark 4: Image Source vs. Analytical (Single Reflection)

**Geometry:** Point source 2m from wall, receiver 3m from wall, absorption α = 0.30

**Expected:** First reflection arrives at t = (√(4²+5²) × 2) / 343 = **0.0379 seconds**

**Expected amplitude ratio:** (1 - α) / distance_ratio = 0.70 × (1 / 6.40) = **0.109**

**Tolerance:** < 0.1 ms timing error, < 1% amplitude error

### Benchmark 5: Ray Tracing Performance

**Room:** 500 triangles, 1M rays, max 100 reflections

**Expected WASM:** < 5 seconds on M1 MacBook Pro (single-threaded)

**Expected Server:** < 10 seconds on c6i.2xlarge (8 cores, parallel)

**Expected memory:** < 2 GB RAM

---

## Verification

### Integration Testing Checklist

1. **Auth flow** — Register → login → JWT → authorized calls → refresh → logout
2. **Project CRUD** — Create → update geometry → save → reload → verify preserved
3. **Room editing** — Load OBJ → assign materials → place sources → place receivers → save
4. **WASM simulation** — Small room (100 triangles, 100k rays) → WASM → results → display parameters
5. **Server simulation** — Large room (2k triangles, 5M rays) → job queued → WebSocket progress → S3 results
6. **Ray visualization** — Load ray data → render 10k rays → color by order → animate
7. **SPL heat map** — Compute grid SPL → render color map → octave band selector
8. **Parameters display** — RT60 table → octave band chart → C50/C80/STI → verify values
9. **Auralization** — Download IR → load dry speech signal → convolve → play binaural
10. **HRTF rendering** — Apply HRTF → verify spatial positioning → move listener → hear changes
11. **Material search** — Search "concrete" → results returned → select → assign to surface
12. **Optimization** — Set target RT60=2.0s → run optimization → receive recommendations → verify improved
13. **Report generation** — Generate PDF → download → verify structure and charts
14. **Plan limits** — Free user → 200k rays → blocked with upgrade prompt
15. **Billing** — Subscribe to Professional → Stripe → webhook → plan updated → limits lifted
16. **Concurrent users** — 10 users simulate simultaneously → all complete correctly
17. **Error handling** — Invalid geometry → clear error message → no crash
18. **Template usage** — Select "Concert Hall" → geometry loaded → materials assigned → simulate
19. **Export IR** — Download impulse response WAV → import to DAW → convolve → verify
20. **Cross-validation** — Compare results to ODEON/EASE reference simulations → <10% RT60 error

### SQL Verification Queries

```sql
-- 1. Simulation throughput and success rate
SELECT
    DATE(created_at) as date,
    COUNT(*) as total_simulations,
    COUNT(*) FILTER (WHERE status = 'completed') as completed,
    COUNT(*) FILTER (WHERE status = 'failed') as failed,
    AVG(wall_time_ms) FILTER (WHERE status = 'completed') as avg_time_ms
FROM simulations
WHERE created_at > NOW() - INTERVAL '7 days'
GROUP BY DATE(created_at)
ORDER BY date DESC;

-- 2. Material usage statistics
SELECT
    am.name,
    am.category,
    COUNT(DISTINCT s.project_id) as projects_used,
    AVG(s.wall_time_ms) as avg_sim_time_ms
FROM acoustic_materials am
JOIN projects p ON p.geometry_data @> jsonb_build_array(jsonb_build_object('material_id', am.id))
JOIN simulations s ON s.project_id = p.id
WHERE s.status = 'completed'
GROUP BY am.id, am.name, am.category
ORDER BY projects_used DESC
LIMIT 20;

-- 3. User plan distribution and usage
SELECT
    plan,
    COUNT(*) as user_count,
    AVG(total_sims) as avg_sims_per_user,
    SUM(total_rays) as total_rays_traced
FROM (
    SELECT
        u.plan,
        COUNT(s.id) as total_sims,
        SUM(s.ray_count) as total_rays
    FROM users u
    LEFT JOIN simulations s ON s.user_id = u.id
    WHERE s.created_at > NOW() - INTERVAL '30 days'
    GROUP BY u.id, u.plan
) subq
GROUP BY plan
ORDER BY user_count DESC;

-- 4. Average room acoustic parameters by project type
SELECT
    p.project_type,
    COUNT(*) as simulation_count,
    AVG((s.results_summary->'rt60'->>'hz_1k')::float) as avg_rt60_1k,
    AVG((s.results_summary->'c50'->>'hz_1k')::float) as avg_c50_1k,
    AVG((s.results_summary->>'sti')::float) as avg_sti
FROM projects p
JOIN simulations s ON s.project_id = p.id
WHERE s.status = 'completed'
  AND s.results_summary IS NOT NULL
GROUP BY p.project_type;

-- 5. Top error messages and failure causes
SELECT
    error_message,
    COUNT(*) as occurrence_count,
    AVG(ray_count) as avg_ray_count,
    AVG(triangle_count) as avg_triangle_count
FROM simulations
WHERE status = 'failed'
  AND created_at > NOW() - INTERVAL '7 days'
GROUP BY error_message
ORDER BY occurrence_count DESC
LIMIT 10;
```

---

## Risk Mitigation

### Technical Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|------------|
| **WASM performance insufficient** | Medium | High | Benchmark early (Day 15), optimize BVH traversal, fall back to server for slow devices |
| **Ray tracing convergence issues** | Low | Medium | Validate against analytical cases (Day 14), adaptive ray count based on energy variance |
| **Three.js rendering bottlenecks** | Medium | Medium | GPU instancing for rays, LOD for complex geometry, WebGL 2.0 required |
| **STI calculation complexity** | Medium | Low | Use simplified MTF method (7 bands × 14 freqs), pre-compute lookup tables |
| **Material database accuracy** | Low | High | Source from peer-reviewed databases (acoustic.ua, ODEON library), verify against measurements |
| **HRTF data licensing** | Low | Low | MIT KEMAR database is public domain, cite properly |
| **Server cost for large sims** | Medium | Medium | WASM-first architecture, optimize server ray tracing (multi-core), usage-based pricing |
| **BVH build time for complex geometry** | Low | Medium | Parallel BVH construction, cache BVH in S3, surface area heuristic (SAH) |

### Business Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|------------|
| **Acoustic consultants won't adopt** | Medium | High | Target education/small studios first, price below ODEON ($8K → $129/mo), free tier |
| **Incumbents (ODEON/EASE) compete** | Low | Medium | Web-based UX advantage, faster iteration, modern stack, API-first |
| **Academic users don't convert** | High | Low | Generous free tier (100k rays sufficient for teaching), student pricing |
| **Compliance/certification concerns** | Medium | High | Validate against ISO 3382 test cases, publish validation reports, target non-critical projects first |
| **Environmental noise complexity** | Medium | Medium | Post-MVP feature (v1.1), partner with GIS providers, focus on room acoustics for MVP |

---

## Go-to-Market Strategy

### Launch Plan (Week 1-4 post-deployment)

**Week 1: Soft Launch**
- Deploy to production, private beta with 20 hand-selected acoustic consultants
- Monitor error rates, performance metrics, gather feedback
- Fix critical bugs, optimize hot paths

**Week 2: Product Hunt Launch**
- Prepare demo video (60s): shoebox room → assign materials → run simulation → view results → listen to auralization
- Submit to Product Hunt, HackerNews, Reddit r/acoustics
- Goal: 500 signups, 100 completed simulations

**Week 3: Content Marketing**
- Blog posts:
  - "How room acoustics simulation works: Ray tracing vs. wave-based methods"
  - "Validating acoustics software: Our RT60 accuracy vs. measurements"
  - "5 common mistakes in acoustic design (and how to avoid them)"
- Case studies: classroom acoustics, home studio treatment, conference room design
- SEO: target "room acoustics software", "RT60 calculator", "acoustic simulation"

**Week 4: Outreach to Acoustic Consultants**
- Direct outreach to 200 acoustic engineering firms (INCE member directory)
- Offer 3-month Professional plan trial for firms
- Webinar: "Getting started with Acoustica: Room acoustics simulation in 15 minutes"
- Goal: 10 paying Professional subscribers by Week 8

### Pricing Strategy

**Free Tier:**
- 100k rays (sufficient for small rooms, classrooms)
- 3 projects
- Basic materials (50 common materials)
- Shoebox room only (no OBJ import)
- Target: Students, hobbyists, trial users

**Professional — $129/month:**
- Unlimited rays (WASM + server)
- Unlimited projects
- 500+ materials
- OBJ import
- Report generation (PDF)
- Priority support
- Target: Freelance consultants, small studios

**Advanced — $299/user/month:**
- All Professional features
- Material optimization (AI-powered)
- Collaborative editing (real-time multi-user)
- API access (REST)
- White-label reports
- Dedicated account manager
- Target: Acoustic consulting firms (5-50 employees)

**Enterprise — Custom:**
- On-premise deployment
- Custom propagation models
- BIM integration (IFC round-trip)
- Advanced GIS (environmental noise)
- Training and certification
- Target: Large firms (50+ employees), government agencies

### Customer Acquisition Channels

1. **SEO/Content** (40% of signups): Blog, case studies, validation reports, YouTube tutorials
2. **Communities** (30%): Reddit r/acoustics, r/audiophile, r/hometheater, acoustics forums, LinkedIn groups
3. **Academic** (20%): University partnerships (MIT, Stanford, DTU), free licenses for research, guest lectures
4. **Direct Sales** (10%): Outreach to acoustic consulting firms, conference attendance (Acoustical Society of America, INCE)

### Success Metrics (6 months post-launch)

- **User Acquisition:** 2,000 registered users (1,500 free, 400 professional, 100 advanced)
- **Engagement:** 40% MAU/signup ratio, 20 simulations/user/month average
- **Revenue:** $60K MRR (400 × $129 + 100 × $299)
- **Retention:** 80% month-over-month retention for paid plans
- **NPS:** >40 (acoustic consultants are demanding users)
- **Performance:** 95th percentile WASM simulation time <10 seconds

---

## Team and Resourcing

### Recommended Team (full-time equivalents)

**Engineering (3.5 FTE for 42 days = 6 engineering-months):**
- 1 × Senior Rust Engineer (backend API, solver core) — 100%
- 1 × Senior Frontend Engineer (React, Three.js, Web Audio) — 100%
- 0.5 × Python Engineer (optimization, report generation) — 50%
- 1 × Fullstack Engineer (WASM, integration, deployment) — 100%

**Domain Expertise (0.5 FTE):**
- 0.5 × Acoustic Engineer (validation, material database, parameter calculation) — 50% (can be contractor)

**Design (0.2 FTE):**
- 0.2 × Product Designer (UI/UX, 3D editor, visualization) — 20% (can be contractor)

**Total:** 4.2 FTE × 2 months = **8.4 person-months**

### Estimated Costs (42-day timeline)

- **Engineering:** 4 engineers × $150/hour × 160 hours/month × 2 months = **$192,000**
- **Domain Expertise:** 0.5 × $120/hour × 160 hours/month × 2 months = **$19,200**
- **Design:** 0.2 × $100/hour × 160 hours/month × 2 months = **$6,400**
- **Infrastructure:** AWS (dev + staging) = **$2,000**
- **Tools:** GitHub, Sentry, Stripe, Figma = **$500**
- **Contingency:** 15% = **$33,000**

**Total:** **~$253,000** for MVP (42-day timeline)

**Monthly Burn (post-MVP):**
- Engineering: 2 FTE × $150/hour × 160 hours = **$48,000**
- Infrastructure: AWS (production, scaling) = **$5,000**
- Tools & Services: **$1,000**
- **Total:** **$54,000/month**

**Break-even:** $54K burn / $129 ARPU ≈ **420 Professional subscribers** (or equivalent ARR mix)

At 30% month-over-month growth post-launch: break-even at **Month 9**

---

## Appendix: Acoustic Physics Primer

### Sound Propagation Basics

**Speed of sound in air:**
```
c = 331.3 + 0.606 × T_celsius ≈ 343 m/s at 20°C
```

**Wavelength:**
```
λ = c / f

Examples:
- 125 Hz → λ = 2.74 m (bass, long wavelength)
- 1 kHz → λ = 0.34 m (speech, mid wavelength)
- 8 kHz → λ = 0.043 m (sibilants, short wavelength)
```

**Schroeder frequency (transition from wave to ray regime):**
```
f_s = 2000 × √(RT60 / V)

For a typical classroom (V=200 m³, RT60=0.8s):
f_s = 2000 × √(0.8 / 200) = 126 Hz

Below f_s: Room modes dominate (wave-based methods needed)
Above f_s: Diffuse field (geometrical acoustics valid)
```

### Material Properties

**Absorption coefficient α:**
- α = 0: Perfect reflector (concrete, tile)
- α = 0.3: Moderate absorber (painted drywall)
- α = 0.8: Good absorber (fiberglass panel)
- α = 1.0: Perfect absorber (anechoic wedges)

**Scattering coefficient s:**
- s = 0: Specular reflector (smooth surfaces)
- s = 0.5: Mixed specular + diffuse (textured walls)
- s = 1.0: Ideal diffuser (hemispherical scatterers)

**Common materials:**
| Material | α @ 500 Hz | α @ 1 kHz | α @ 2 kHz | s @ 1 kHz |
|----------|-----------|-----------|-----------|-----------|
| Concrete (smooth) | 0.01 | 0.01 | 0.02 | 0.05 |
| Drywall (painted) | 0.08 | 0.05 | 0.03 | 0.10 |
| Carpet (heavy) | 0.30 | 0.50 | 0.60 | 0.15 |
| Acoustic panel (2" fiberglass) | 0.60 | 0.95 | 0.98 | 0.10 |
| Wood (parquet) | 0.10 | 0.10 | 0.10 | 0.15 |
| Audience (seated) | 0.60 | 0.75 | 0.85 | 0.70 |

### Room Acoustic Standards

**ISO 3382-1: Concert halls and auditoria**
- Measure RT60, EDT, C80, G, LF at multiple positions
- Spatial averaging required (6-12 positions typical)

**ISO 3382-2: Ordinary rooms**
- Focus on STI for speech intelligibility
- Single-number rating for classroom acoustics

**ISO 3382-3: Open-plan offices**
- Spatial decay of speech level (D2,S)
- Privacy distance and distraction distance

**IEC 60268-16: Speech intelligibility (STI)**
- STI > 0.75: Excellent (broadcasting)
- STI 0.60 - 0.75: Good (classrooms, conference rooms)
- STI 0.45 - 0.60: Fair (public spaces)
- STI < 0.45: Poor (unacceptable for speech)

