# 29. MolForge — AI-Driven Molecular Design and Drug Candidate Screening

## Implementation Plan

**MVP Scope:** Browser-based molecular editor with 2D structure drawing (SMILES/SDF import) and real-time property calculation (MW, logP, TPSA, Lipinski violations), protein target preparation pipeline (fetch from RCSB PDB or upload, automated hydrogen addition and binding site detection via fpocket), AI de novo molecular generation using a PyTorch VAE trained on ChEMBL 34 constrained to user-defined physicochemical properties (MW 200–500, logP 0–5, TPSA 20–140 Ų, synthetic accessibility ≤3.5) generating 1,000–10,000 novel molecules per run with GPU acceleration, cloud-based AutoDock Vina virtual screening supporting up to 10,000 molecules per campaign with automatic parallelization across CPU workers, ADMET property prediction using ensemble XGBoost models for 10 key endpoints (Caco-2 permeability, BBB penetration, hERG inhibition, CYP3A4 inhibition, AMES mutagenicity, hepatotoxicity, solubility, plasma protein binding, human intestinal absorption, metabolic half-life), interactive 3D protein-ligand viewer using NGL Viewer with hydrogen bond and hydrophobic interaction display, results table with composite scoring (binding affinity + ADMET traffic-light flags), Stripe billing with three tiers (Academic free / Researcher $199/mo / Lab $799/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| Cheminformatics Service | Python 3.12 (FastAPI) | RDKit molecular operations, property calculation |
| AI/ML Service | Python 3.12 (FastAPI) | PyTorch VAE generation, XGBoost ADMET models |
| Docking Service | Python 3.12 (FastAPI) | AutoDock Vina wrapper, Meeko molecular prep |
| Database | PostgreSQL 16 + RDKit cartridge | Projects, users, molecules, docking runs, chemical search |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and ORCID OAuth providers, argon2 password hashing |
| Object Storage | AWS S3 | SDF files, PDB structures, docking results, ML models |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| Molecular Editor | Ketcher 2.18 | Open-source 2D structure editor, SMILES I/O |
| 3D Viewer | NGL Viewer 2.0 | WebGL protein-ligand visualization, MMTFReader support |
| Job Queue | Redis 7 + Tokio tasks | Docking and generation job management, priority queues |
| Real-time | WebSocket (Axum) | Live progress updates for long-running jobs |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | Static frontend assets, model files |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, error tracking |
| CI/CD | GitHub Actions | Rust tests, Python ML tests, Docker image push |

### Key Architecture Decisions

1. **Rust/Axum primary backend with Python microservices for domain-specific tasks**: Rust handles all HTTP routing, authentication, database access, S3 interactions, and WebSocket connections. Python FastAPI microservices are used exclusively for compute-intensive cheminformatics (RDKit molecular operations), AI inference (PyTorch VAE, XGBoost), and docking orchestration (AutoDock Vina). This hybrid architecture leverages Rust's performance and type safety for business logic while using Python's rich ecosystem for scientific computing.

2. **PostgreSQL RDKit cartridge for chemical structure search**: The RDKit PostgreSQL cartridge enables native substructure and similarity search directly in SQL using GiST indexes on Morgan fingerprints. This eliminates the need for an external chemical search engine and allows joining molecule searches with relational queries (user ownership, project filtering) in a single query with excellent performance up to 10M+ molecules.

3. **AutoDock Vina on CPU workers rather than GPU docking**: Vina is highly parallelized across CPU cores and can handle 50–100 molecules per hour per 32-core worker. GPU docking alternatives (Vina-GPU, GNINA) showed only 2–3x speedups in benchmarks while requiring expensive A100 GPUs. For the MVP target of 10K molecules/campaign, a pool of 10–20 CPU workers (spot instances) provides better cost-efficiency at $0.05–$0.10 per molecule vs $0.30+ for GPU.

4. **Separation of generation/docking/ADMET into independent job queues**: Each compute task type (AI generation, docking, ADMET prediction) has its own Redis-backed priority queue with dedicated worker pools. This allows independent scaling (e.g., 5 GPU workers for generation, 20 CPU workers for docking, 10 CPU workers for ADMET) and prevents head-of-line blocking. Jobs are routed from Rust API → Redis queue → Python worker → S3 result → Rust API notification.

5. **Ketcher + NGL Viewer for client-side molecular visualization**: Ketcher (developed by EPAM for life sciences) is the most mature open-source 2D structure editor with full SMILES/MOL/SDF support and no licensing restrictions (Apache 2.0). NGL Viewer provides publication-quality 3D rendering with protein surface shading and automatic interaction detection. Both run entirely in the browser with no server-side rendering, reducing latency and server costs.

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "rdkit";  -- RDKit cartridge for molecular search
CREATE EXTENSION IF NOT EXISTS "pg_trgm";  -- Trigram search for text fields

-- Users
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email TEXT UNIQUE NOT NULL,
    password_hash TEXT,  -- NULL for OAuth-only users
    name TEXT NOT NULL,
    avatar_url TEXT,
    auth_provider TEXT DEFAULT 'email',  -- email | google | orcid
    auth_provider_id TEXT,
    plan TEXT NOT NULL DEFAULT 'academic',  -- academic | researcher | lab | enterprise
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    academic_verified BOOLEAN DEFAULT false,
    academic_email TEXT,
    affiliation TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX users_email_idx ON users(email);
CREATE INDEX users_stripe_idx ON users(stripe_customer_id);
CREATE INDEX users_plan_idx ON users(plan);

-- Organizations (for Lab and Enterprise plans)
CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    slug TEXT UNIQUE NOT NULL,
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    plan TEXT NOT NULL DEFAULT 'lab',
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
    role TEXT NOT NULL DEFAULT 'member',  -- owner | admin | editor | viewer
    joined_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(org_id, user_id)
);
CREATE INDEX org_members_org_idx ON org_members(org_id);
CREATE INDEX org_members_user_idx ON org_members(user_id);

-- Projects (workspace for target + molecules + docking campaigns)
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    status TEXT NOT NULL DEFAULT 'active',  -- active | archived
    is_public BOOLEAN NOT NULL DEFAULT false,
    settings JSONB DEFAULT '{}',
    thumbnail_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);
CREATE INDEX projects_org_idx ON projects(org_id);
CREATE INDEX projects_status_idx ON projects(status);
CREATE INDEX projects_updated_idx ON projects(updated_at DESC);

-- Protein Targets
CREATE TABLE protein_targets (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    pdb_id TEXT,  -- e.g., "1HIV" if fetched from RCSB
    source TEXT NOT NULL,  -- rcsb | upload | alphafold | homology
    pdb_file_url TEXT NOT NULL,  -- S3 URL to original PDB/mmCIF
    prepared_file_url TEXT,  -- S3 URL to prepared structure (hydrogens, charges)
    binding_site_residues TEXT[],  -- e.g., ['A:TRP215', 'A:GLU220', ...]
    grid_center_x REAL,
    grid_center_y REAL,
    grid_center_z REAL,
    grid_size_x REAL DEFAULT 20.0,
    grid_size_y REAL DEFAULT 20.0,
    grid_size_z REAL DEFAULT 20.0,
    preparation_params JSONB DEFAULT '{}',  -- pH, water threshold, etc.
    preparation_status TEXT DEFAULT 'pending',  -- pending | running | completed | failed
    preparation_error TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX targets_project_idx ON protein_targets(project_id);
CREATE INDEX targets_pdb_idx ON protein_targets(pdb_id);

-- Molecule Libraries
CREATE TABLE molecule_libraries (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    source TEXT NOT NULL,  -- upload | generated | builtin | zinc | enamine
    molecule_count INTEGER NOT NULL DEFAULT 0,
    sdf_file_url TEXT,  -- S3 URL to bulk SDF file
    generation_run_id UUID,  -- FK to generation_runs if source=generated
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX libraries_project_idx ON molecule_libraries(project_id);
CREATE INDEX libraries_source_idx ON molecule_libraries(source);

-- Molecules
CREATE TABLE molecules (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    library_id UUID NOT NULL REFERENCES molecule_libraries(id) ON DELETE CASCADE,
    smiles TEXT NOT NULL,  -- Canonical SMILES
    inchi_key TEXT NOT NULL,  -- InChI Key for global uniqueness
    mol_weight REAL,
    logp REAL,
    tpsa REAL,  -- Topological polar surface area
    hbd INTEGER,  -- H-bond donors
    hba INTEGER,  -- H-bond acceptors
    rotatable_bonds INTEGER,
    aromatic_rings INTEGER,
    num_atoms INTEGER,
    num_heavy_atoms INTEGER,
    formal_charge INTEGER,
    synthetic_accessibility REAL,  -- SAScore 1-10
    morgan_fp MOL,  -- RDKit Morgan fingerprint (for similarity search)
    properties JSONB DEFAULT '{}',  -- Extended descriptors
    tags TEXT[] DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX molecules_library_idx ON molecules(library_id);
CREATE INDEX molecules_inchi_idx ON molecules(inchi_key);
CREATE INDEX molecules_mw_idx ON molecules(mol_weight);
CREATE INDEX molecules_logp_idx ON molecules(logp);
-- RDKit chemical search indexes
CREATE INDEX molecules_morgan_fp_idx ON molecules USING gist(morgan_fp);

-- AI Generation Runs
CREATE TABLE generation_runs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    target_id UUID REFERENCES protein_targets(id) ON DELETE SET NULL,
    method TEXT NOT NULL,  -- vae | scaffold_hop | rl (v1.1+)
    constraints JSONB NOT NULL,  -- {mw: [200, 500], logp: [0, 5], scaffold: null, ...}
    num_molecules_requested INTEGER NOT NULL,
    num_molecules_generated INTEGER DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | running | completed | failed
    error_message TEXT,
    output_library_id UUID REFERENCES molecule_libraries(id),
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    duration_seconds INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX generation_project_idx ON generation_runs(project_id);
CREATE INDEX generation_user_idx ON generation_runs(user_id);
CREATE INDEX generation_status_idx ON generation_runs(status);

-- Docking Runs
CREATE TABLE docking_runs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    target_id UUID NOT NULL REFERENCES protein_targets(id) ON DELETE CASCADE,
    library_id UUID NOT NULL REFERENCES molecule_libraries(id) ON DELETE CASCADE,
    parameters JSONB NOT NULL,  -- {exhaustiveness: 8, num_modes: 9, energy_range: 3.0}
    molecule_count INTEGER NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | running | completed | failed | cancelled
    progress_pct REAL DEFAULT 0.0,
    molecules_docked INTEGER DEFAULT 0,
    hit_count INTEGER DEFAULT 0,  -- Molecules with score < threshold
    hit_threshold REAL DEFAULT -7.0,  -- kcal/mol
    results_url TEXT,  -- S3 URL to results CSV
    error_message TEXT,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    duration_seconds INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX docking_project_idx ON docking_runs(project_id);
CREATE INDEX docking_user_idx ON docking_runs(user_id);
CREATE INDEX docking_status_idx ON docking_runs(status);
CREATE INDEX docking_target_idx ON docking_runs(target_id);

-- Docking Results
CREATE TABLE docking_results (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    run_id UUID NOT NULL REFERENCES docking_runs(id) ON DELETE CASCADE,
    molecule_id UUID NOT NULL REFERENCES molecules(id) ON DELETE CASCADE,
    vina_score REAL NOT NULL,  -- kcal/mol
    ml_rescore REAL,  -- XGBoost re-scoring (optional, v1.1+)
    consensus_score REAL,  -- Composite score
    num_poses INTEGER NOT NULL DEFAULT 1,
    best_pose_index INTEGER DEFAULT 0,
    pose_file_url TEXT NOT NULL,  -- S3 URL to PDBQT or SDF with poses
    strain_energy REAL,
    clash_count INTEGER,
    interactions JSONB,  -- {hbonds: [...], hydrophobic: [...], salt_bridges: [...]}
    rank INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX docking_results_run_idx ON docking_results(run_id);
CREATE INDEX docking_results_molecule_idx ON docking_results(molecule_id);
CREATE INDEX docking_results_score_idx ON docking_results(vina_score);
CREATE INDEX docking_results_rank_idx ON docking_results(rank);

-- ADMET Profiles
CREATE TABLE admet_profiles (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    molecule_id UUID NOT NULL REFERENCES molecules(id) ON DELETE CASCADE,
    run_id UUID,  -- Nullable: can be standalone prediction or part of docking run
    predictions JSONB NOT NULL,  -- {caco2: {value: 5.2, confidence: 0.85, flag: 'green'}, ...}
    applicability_domain_flag BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX admet_molecule_idx ON admet_profiles(molecule_id);
CREATE INDEX admet_run_idx ON admet_profiles(run_id);

-- Job Queue (for tracking background jobs)
CREATE TABLE jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    job_type TEXT NOT NULL,  -- generation | docking | admet | target_prep
    reference_id UUID NOT NULL,  -- ID of generation_run, docking_run, etc.
    status TEXT NOT NULL DEFAULT 'queued',  -- queued | running | completed | failed
    priority INTEGER DEFAULT 0,  -- Higher = more urgent
    worker_id TEXT,
    progress_pct REAL DEFAULT 0.0,
    error_message TEXT,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX jobs_type_idx ON jobs(job_type);
CREATE INDEX jobs_status_idx ON jobs(status);
CREATE INDEX jobs_reference_idx ON jobs(reference_id);

-- Pipeline Compounds (for hit-to-lead tracking, v1.1+)
CREATE TABLE pipeline_compounds (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    molecule_id UUID NOT NULL REFERENCES molecules(id) ON DELETE CASCADE,
    stage TEXT NOT NULL,  -- hit | validated_hit | lead | optimized_lead | candidate
    experimental_data JSONB DEFAULT '{}',
    notes TEXT,
    promoted_by UUID REFERENCES users(id),
    promoted_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX pipeline_project_idx ON pipeline_compounds(project_id);
CREATE INDEX pipeline_stage_idx ON pipeline_compounds(stage);

-- Usage Tracking (for billing enforcement)
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,  -- molecules_generated | molecules_docked | admet_predictions | storage_gb
    quantity REAL NOT NULL,
    period_start DATE NOT NULL,
    period_end DATE NOT NULL,
    metadata JSONB DEFAULT '{}',
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
    pub academic_verified: bool,
    pub academic_email: Option<String>,
    pub affiliation: Option<String>,
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
    pub status: String,
    pub is_public: bool,
    pub settings: serde_json::Value,
    pub thumbnail_url: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct ProteinTarget {
    pub id: Uuid,
    pub project_id: Uuid,
    pub name: String,
    pub pdb_id: Option<String>,
    pub source: String,
    pub pdb_file_url: String,
    pub prepared_file_url: Option<String>,
    pub binding_site_residues: Vec<String>,
    pub grid_center_x: Option<f32>,
    pub grid_center_y: Option<f32>,
    pub grid_center_z: Option<f32>,
    pub grid_size_x: f32,
    pub grid_size_y: f32,
    pub grid_size_z: f32,
    pub preparation_params: serde_json::Value,
    pub preparation_status: String,
    pub preparation_error: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct MoleculeLibrary {
    pub id: Uuid,
    pub project_id: Uuid,
    pub name: String,
    pub source: String,
    pub molecule_count: i32,
    pub sdf_file_url: Option<String>,
    pub generation_run_id: Option<Uuid>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Molecule {
    pub id: Uuid,
    pub library_id: Uuid,
    pub smiles: String,
    pub inchi_key: String,
    pub mol_weight: Option<f32>,
    pub logp: Option<f32>,
    pub tpsa: Option<f32>,
    pub hbd: Option<i32>,
    pub hba: Option<i32>,
    pub rotatable_bonds: Option<i32>,
    pub aromatic_rings: Option<i32>,
    pub num_atoms: Option<i32>,
    pub num_heavy_atoms: Option<i32>,
    pub formal_charge: Option<i32>,
    pub synthetic_accessibility: Option<f32>,
    pub properties: serde_json::Value,
    pub tags: Vec<String>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct GenerationRun {
    pub id: Uuid,
    pub project_id: Uuid,
    pub user_id: Uuid,
    pub target_id: Option<Uuid>,
    pub method: String,
    pub constraints: serde_json::Value,
    pub num_molecules_requested: i32,
    pub num_molecules_generated: i32,
    pub status: String,
    pub error_message: Option<String>,
    pub output_library_id: Option<Uuid>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub duration_seconds: Option<i32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct DockingRun {
    pub id: Uuid,
    pub project_id: Uuid,
    pub user_id: Uuid,
    pub target_id: Uuid,
    pub library_id: Uuid,
    pub parameters: serde_json::Value,
    pub molecule_count: i32,
    pub status: String,
    pub progress_pct: f32,
    pub molecules_docked: i32,
    pub hit_count: i32,
    pub hit_threshold: f32,
    pub results_url: Option<String>,
    pub error_message: Option<String>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub duration_seconds: Option<i32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct DockingResult {
    pub id: Uuid,
    pub run_id: Uuid,
    pub molecule_id: Uuid,
    pub vina_score: f32,
    pub ml_rescore: Option<f32>,
    pub consensus_score: f32,
    pub num_poses: i32,
    pub best_pose_index: i32,
    pub pose_file_url: String,
    pub strain_energy: Option<f32>,
    pub clash_count: Option<i32>,
    pub interactions: serde_json::Value,
    pub rank: i32,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct AdmetProfile {
    pub id: Uuid,
    pub molecule_id: Uuid,
    pub run_id: Option<Uuid>,
    pub predictions: serde_json::Value,
    pub applicability_domain_flag: bool,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct GenerationConstraints {
    pub mw_min: f32,
    pub mw_max: f32,
    pub logp_min: f32,
    pub logp_max: f32,
    pub tpsa_min: f32,
    pub tpsa_max: f32,
    pub sa_max: f32,  // Synthetic accessibility max
    pub scaffold_smarts: Option<String>,
    pub required_substructures: Vec<String>,
    pub forbidden_substructures: Vec<String>,
}

#[derive(Debug, Deserialize)]
pub struct DockingParameters {
    pub exhaustiveness: i32,  // Default 8
    pub num_modes: i32,  // Default 9
    pub energy_range: f32,  // Default 3.0 kcal/mol
}
```

---

## Architecture Deep-Dives

### 1. API Gateway and Job Orchestration (Rust/Axum)

The primary Rust backend handles all API routing, authentication, database queries, and job orchestration. It delegates compute tasks to Python microservices via Redis queues.

```rust
// src/api/handlers/docking.rs

use axum::{
    extract::{Path, State, Json},
    http::StatusCode,
    response::IntoResponse,
};
use uuid::Uuid;
use redis::AsyncCommands;
use crate::{
    db::models::{DockingRun, DockingParameters},
    state::AppState,
    auth::Claims,
    error::ApiError,
};

#[derive(serde::Deserialize)]
pub struct CreateDockingRequest {
    pub target_id: Uuid,
    pub library_id: Uuid,
    pub parameters: DockingParameters,
}

pub async fn create_docking_run(
    State(state): State<AppState>,
    claims: Claims,
    Path(project_id): Path<Uuid>,
    Json(req): Json<CreateDockingRequest>,
) -> Result<impl IntoResponse, ApiError> {
    // 1. Verify project ownership and target/library belong to project
    let project = sqlx::query!(
        "SELECT * FROM projects WHERE id = $1 AND (owner_id = $2 OR org_id IN (
            SELECT org_id FROM org_members WHERE user_id = $2
        ))",
        project_id,
        claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Project not found"))?;

    let target = sqlx::query_as!(
        crate::db::models::ProteinTarget,
        "SELECT * FROM protein_targets WHERE id = $1 AND project_id = $2",
        req.target_id,
        project_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Target not found or not in project"))?;

    if target.preparation_status != "completed" {
        return Err(ApiError::BadRequest(
            "Target preparation must be completed before docking"
        ));
    }

    let library = sqlx::query_as!(
        crate::db::models::MoleculeLibrary,
        "SELECT * FROM molecule_libraries WHERE id = $1 AND project_id = $2",
        req.library_id,
        project_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Library not found or not in project"))?;

    // 2. Check plan limits
    let user = sqlx::query_as!(
        crate::db::models::User,
        "SELECT * FROM users WHERE id = $1",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    let molecule_limit = match user.plan.as_str() {
        "academic" => 500,
        "researcher" => 25_000,
        "lab" => 250_000,
        "enterprise" => i32::MAX,
        _ => 500,
    };

    if library.molecule_count > molecule_limit {
        return Err(ApiError::PlanLimit(format!(
            "Your plan allows docking up to {} molecules. Upgrade to dock this library.",
            molecule_limit
        )));
    }

    // 3. Create docking run record
    let run = sqlx::query_as!(
        DockingRun,
        r#"INSERT INTO docking_runs
            (project_id, user_id, target_id, library_id, parameters, molecule_count, status)
        VALUES ($1, $2, $3, $4, $5, $6, 'pending')
        RETURNING *"#,
        project_id,
        claims.user_id,
        req.target_id,
        req.library_id,
        serde_json::to_value(&req.parameters)?,
        library.molecule_count,
    )
    .fetch_one(&state.db)
    .await?;

    // 4. Create job record
    let priority = match user.plan.as_str() {
        "lab" | "enterprise" => 10,
        "researcher" => 5,
        _ => 0,
    };

    let job = sqlx::query!(
        r#"INSERT INTO jobs (job_type, reference_id, priority, status)
        VALUES ('docking', $1, $2, 'queued')
        RETURNING id"#,
        run.id,
        priority
    )
    .fetch_one(&state.db)
    .await?;

    // 5. Enqueue job to Redis (priority queue)
    let mut redis_conn = state.redis.get_async_connection().await?;
    let job_payload = serde_json::json!({
        "job_id": job.id,
        "run_id": run.id,
        "target_id": req.target_id,
        "library_id": req.library_id,
        "parameters": req.parameters,
    });

    redis_conn
        .zadd::<_, _, _, ()>(
            "docking:jobs",
            serde_json::to_string(&job_payload)?,
            -priority,  // Negative for high-priority first in sorted set
        )
        .await?;

    // 6. Record usage
    let today = chrono::Utc::now().date_naive();
    sqlx::query!(
        r#"INSERT INTO usage_records
            (user_id, org_id, record_type, quantity, period_start, period_end)
        VALUES ($1, $2, 'molecules_docked', $3, $4, $4)"#,
        claims.user_id,
        project.org_id,
        library.molecule_count as f32,
        today
    )
    .execute(&state.db)
    .await?;

    Ok((StatusCode::CREATED, Json(run)))
}

pub async fn get_docking_run(
    State(state): State<AppState>,
    claims: Claims,
    Path((project_id, run_id)): Path<(Uuid, Uuid)>,
) -> Result<Json<DockingRun>, ApiError> {
    let run = sqlx::query_as!(
        DockingRun,
        "SELECT dr.* FROM docking_runs dr
         JOIN projects p ON dr.project_id = p.id
         WHERE dr.id = $1 AND dr.project_id = $2
         AND (p.owner_id = $3 OR p.org_id IN (
             SELECT org_id FROM org_members WHERE user_id = $3
         ))",
        run_id,
        project_id,
        claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Docking run not found"))?;

    Ok(Json(run))
}

pub async fn get_docking_results(
    State(state): State<AppState>,
    claims: Claims,
    Path((project_id, run_id)): Path<(Uuid, Uuid)>,
) -> Result<Json<Vec<crate::db::models::DockingResult>>, ApiError> {
    // Verify ownership via project
    let run = get_docking_run(State(state.clone()), claims, Path((project_id, run_id))).await?;

    let results = sqlx::query_as!(
        crate::db::models::DockingResult,
        "SELECT * FROM docking_results WHERE run_id = $1 ORDER BY rank ASC LIMIT 1000",
        run_id
    )
    .fetch_all(&state.db)
    .await?;

    Ok(Json(results))
}

pub async fn cancel_docking_run(
    State(state): State<AppState>,
    claims: Claims,
    Path((project_id, run_id)): Path<(Uuid, Uuid)>,
) -> Result<impl IntoResponse, ApiError> {
    // Verify ownership
    let _ = get_docking_run(State(state.clone()), claims, Path((project_id, run_id))).await?;

    sqlx::query!(
        "UPDATE docking_runs SET status = 'cancelled', completed_at = NOW() WHERE id = $1",
        run_id
    )
    .execute(&state.db)
    .await?;

    sqlx::query!(
        "UPDATE jobs SET status = 'failed' WHERE job_type = 'docking' AND reference_id = $1",
        run_id
    )
    .execute(&state.db)
    .await?;

    Ok(StatusCode::NO_CONTENT)
}
```

### 2. Docking Worker (Python FastAPI Microservice)

Python worker that consumes docking jobs from Redis, orchestrates AutoDock Vina parallelization, and stores results in S3.

```python
# docking_worker/main.py

import asyncio
import json
import logging
import os
import subprocess
import tempfile
from pathlib import Path
from typing import List, Dict, Any
from uuid import UUID

import boto3
import redis.asyncio as redis
from rdkit import Chem
from rdkit.Chem import AllChem
import httpx
from meeko import MoleculePreparation, PDBQTWriterLegacy

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

REDIS_URL = os.getenv("REDIS_URL", "redis://localhost:6379")
S3_BUCKET = os.getenv("S3_BUCKET", "molforge-storage")
DATABASE_URL = os.getenv("DATABASE_URL")
API_BASE_URL = os.getenv("API_BASE_URL", "http://localhost:3000")

s3_client = boto3.client("s3")


class DockingWorker:
    def __init__(self):
        self.redis_client = None
        self.http_client = httpx.AsyncClient()

    async def start(self):
        """Start worker loop consuming from Redis queue."""
        self.redis_client = await redis.from_url(REDIS_URL, decode_responses=True)
        logger.info("Docking worker started, waiting for jobs...")

        while True:
            try:
                # Block-pop from sorted set (highest priority first)
                result = await self.redis_client.bzpopmin("docking:jobs", timeout=30)
                if not result:
                    continue

                queue_name, job_json, score = result
                job_data = json.loads(job_json)
                await self.process_job(job_data)

            except Exception as e:
                logger.error(f"Worker error: {e}", exc_info=True)
                await asyncio.sleep(5)

    async def process_job(self, job_data: Dict[str, Any]):
        """Process a single docking job."""
        job_id = job_data["job_id"]
        run_id = job_data["run_id"]
        target_id = job_data["target_id"]
        library_id = job_data["library_id"]
        parameters = job_data["parameters"]

        logger.info(f"Processing docking job {job_id} for run {run_id}")

        try:
            # Update job status to running
            await self.update_job_status(job_id, "running", 0.0)
            await self.update_run_status(run_id, "running", 0.0)

            # 1. Fetch target prepared structure from S3
            target = await self.fetch_target(target_id)
            receptor_pdbqt = await self.download_s3(target["prepared_file_url"])

            # 2. Fetch molecules from library
            molecules = await self.fetch_molecules(library_id)
            total_molecules = len(molecules)

            # 3. Prepare temporary workspace
            with tempfile.TemporaryDirectory() as tmpdir:
                tmpdir = Path(tmpdir)
                receptor_path = tmpdir / "receptor.pdbqt"
                receptor_path.write_text(receptor_pdbqt)

                # 4. Dock each molecule
                results = []
                for i, mol_data in enumerate(molecules):
                    try:
                        result = await self.dock_molecule(
                            mol_data, receptor_path, target, parameters, tmpdir
                        )
                        if result:
                            results.append(result)
                    except Exception as e:
                        logger.warning(f"Failed to dock molecule {mol_data['id']}: {e}")

                    # Update progress every 10 molecules
                    if (i + 1) % 10 == 0:
                        progress = ((i + 1) / total_molecules) * 100
                        await self.update_run_status(run_id, "running", progress)
                        await self.update_job_status(job_id, "running", progress)

                # 5. Sort results by score
                results.sort(key=lambda x: x["vina_score"])
                for rank, r in enumerate(results, 1):
                    r["rank"] = rank

                # 6. Save results to database
                await self.save_results(run_id, results)

                # 7. Count hits (score < threshold)
                hit_count = sum(1 for r in results if r["vina_score"] < -7.0)

                # 8. Upload results CSV to S3
                results_url = await self.upload_results_csv(run_id, results)

                # 9. Update run as completed
                await self.update_run_completed(run_id, total_molecules, hit_count, results_url)
                await self.update_job_status(job_id, "completed", 100.0)

                logger.info(f"Docking job {job_id} completed: {len(results)} molecules, {hit_count} hits")

        except Exception as e:
            logger.error(f"Job {job_id} failed: {e}", exc_info=True)
            await self.update_job_status(job_id, "failed", error=str(e))
            await self.update_run_status(run_id, "failed", error=str(e))

    async def dock_molecule(
        self,
        mol_data: Dict[str, Any],
        receptor_path: Path,
        target: Dict[str, Any],
        parameters: Dict[str, Any],
        tmpdir: Path,
    ) -> Dict[str, Any]:
        """Dock a single molecule using AutoDock Vina."""
        smiles = mol_data["smiles"]
        mol_id = mol_data["id"]

        # 1. Convert SMILES to 3D structure
        mol = Chem.MolFromSmiles(smiles)
        if not mol:
            raise ValueError(f"Invalid SMILES: {smiles}")

        mol = Chem.AddHs(mol)
        AllChem.EmbedMolecule(mol, randomSeed=42)
        AllChem.UFFOptimizeMolecule(mol)

        # 2. Prepare ligand PDBQT using Meeko
        preparator = MoleculePreparation()
        preparator.prepare(mol)
        pdbqt_string = PDBQTWriterLegacy.write_string(preparator)[0]

        ligand_path = tmpdir / f"ligand_{mol_id}.pdbqt"
        ligand_path.write_text(pdbqt_string)

        # 3. Run AutoDock Vina
        output_path = tmpdir / f"output_{mol_id}.pdbqt"
        log_path = tmpdir / f"log_{mol_id}.txt"

        vina_cmd = [
            "vina",
            "--receptor", str(receptor_path),
            "--ligand", str(ligand_path),
            "--out", str(output_path),
            "--log", str(log_path),
            "--center_x", str(target["grid_center_x"]),
            "--center_y", str(target["grid_center_y"]),
            "--center_z", str(target["grid_center_z"]),
            "--size_x", str(target["grid_size_x"]),
            "--size_y", str(target["grid_size_y"]),
            "--size_z", str(target["grid_size_z"]),
            "--exhaustiveness", str(parameters.get("exhaustiveness", 8)),
            "--num_modes", str(parameters.get("num_modes", 9)),
            "--energy_range", str(parameters.get("energy_range", 3.0)),
        ]

        result = subprocess.run(vina_cmd, capture_output=True, text=True, timeout=300)
        if result.returncode != 0:
            raise RuntimeError(f"Vina failed: {result.stderr}")

        # 4. Parse Vina output for best score
        log_text = log_path.read_text()
        vina_score = self.parse_vina_score(log_text)

        # 5. Upload docking poses to S3
        pose_url = await self.upload_file_to_s3(
            output_path, f"docking/{mol_id}/{mol_id}.pdbqt"
        )

        return {
            "molecule_id": mol_id,
            "vina_score": vina_score,
            "consensus_score": vina_score,  # Simple for MVP
            "num_poses": parameters.get("num_modes", 9),
            "best_pose_index": 0,
            "pose_file_url": pose_url,
            "interactions": {},  # TODO: Parse interactions
        }

    def parse_vina_score(self, log_text: str) -> float:
        """Extract best binding affinity from Vina log."""
        for line in log_text.split("\n"):
            if line.strip().startswith("1 "):
                parts = line.split()
                if len(parts) >= 2:
                    return float(parts[1])
        raise ValueError("Could not parse Vina score from log")

    async def fetch_target(self, target_id: str) -> Dict[str, Any]:
        """Fetch target details from database via API."""
        response = await self.http_client.get(f"{API_BASE_URL}/internal/targets/{target_id}")
        response.raise_for_status()
        return response.json()

    async def fetch_molecules(self, library_id: str) -> List[Dict[str, Any]]:
        """Fetch all molecules in a library."""
        response = await self.http_client.get(
            f"{API_BASE_URL}/internal/libraries/{library_id}/molecules"
        )
        response.raise_for_status()
        return response.json()

    async def download_s3(self, s3_url: str) -> str:
        """Download file content from S3."""
        # Parse s3://bucket/key
        parts = s3_url.replace("s3://", "").split("/", 1)
        bucket, key = parts[0], parts[1]
        obj = s3_client.get_object(Bucket=bucket, Key=key)
        return obj["Body"].read().decode("utf-8")

    async def upload_file_to_s3(self, file_path: Path, s3_key: str) -> str:
        """Upload file to S3 and return URL."""
        s3_client.upload_file(str(file_path), S3_BUCKET, s3_key)
        return f"s3://{S3_BUCKET}/{s3_key}"

    async def upload_results_csv(self, run_id: str, results: List[Dict[str, Any]]) -> str:
        """Upload results as CSV to S3."""
        import csv
        import io

        csv_buffer = io.StringIO()
        if results:
            fieldnames = results[0].keys()
            writer = csv.DictWriter(csv_buffer, fieldnames=fieldnames)
            writer.writeheader()
            writer.writerows(results)

        s3_key = f"docking/runs/{run_id}/results.csv"
        s3_client.put_object(
            Bucket=S3_BUCKET, Key=s3_key, Body=csv_buffer.getvalue().encode("utf-8")
        )
        return f"s3://{S3_BUCKET}/{s3_key}"

    async def save_results(self, run_id: str, results: List[Dict[str, Any]]):
        """Save docking results to database."""
        response = await self.http_client.post(
            f"{API_BASE_URL}/internal/docking/{run_id}/results",
            json={"results": results},
        )
        response.raise_for_status()

    async def update_job_status(self, job_id: str, status: str, progress: float = 0.0, error: str = None):
        """Update job status in database."""
        await self.http_client.patch(
            f"{API_BASE_URL}/internal/jobs/{job_id}",
            json={"status": status, "progress_pct": progress, "error_message": error},
        )

    async def update_run_status(self, run_id: str, status: str, progress: float = 0.0, error: str = None):
        """Update docking run status."""
        await self.http_client.patch(
            f"{API_BASE_URL}/internal/docking/{run_id}",
            json={"status": status, "progress_pct": progress, "error_message": error},
        )

    async def update_run_completed(self, run_id: str, molecules_docked: int, hit_count: int, results_url: str):
        """Mark docking run as completed."""
        await self.http_client.patch(
            f"{API_BASE_URL}/internal/docking/{run_id}/complete",
            json={
                "molecules_docked": molecules_docked,
                "hit_count": hit_count,
                "results_url": results_url,
            },
        )


if __name__ == "__main__":
    worker = DockingWorker()
    asyncio.run(worker.start())
```

### 3. AI Generation Service (Python FastAPI + PyTorch)

Microservice for de novo molecular generation using a VAE trained on ChEMBL.

```python
# generation_service/main.py

import logging
import os
from typing import List, Dict, Any

import torch
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from rdkit import Chem
from rdkit.Chem import Descriptors, Crippen, Lipinski, rdMolDescriptors

from .model import MolecularVAE  # Custom VAE implementation

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

app = FastAPI()

# Load pre-trained VAE model
MODEL_PATH = os.getenv("VAE_MODEL_PATH", "models/chembl_vae.pt")
device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
vae_model = MolecularVAE.load_from_checkpoint(MODEL_PATH).to(device)
vae_model.eval()

logger.info(f"Loaded VAE model from {MODEL_PATH} on {device}")


class GenerationRequest(BaseModel):
    num_molecules: int
    constraints: Dict[str, Any]
    method: str = "vae"  # vae | scaffold_hop


class GenerationResponse(BaseModel):
    molecules: List[Dict[str, Any]]
    num_generated: int
    num_valid: int


def calculate_properties(smiles: str) -> Dict[str, float]:
    """Calculate molecular properties using RDKit."""
    mol = Chem.MolFromSmiles(smiles)
    if not mol:
        return None

    return {
        "mol_weight": Descriptors.MolWt(mol),
        "logp": Crippen.MolLogP(mol),
        "tpsa": Descriptors.TPSA(mol),
        "hbd": Lipinski.NumHDonors(mol),
        "hba": Lipinski.NumHAcceptors(mol),
        "rotatable_bonds": Lipinski.NumRotatableBonds(mol),
        "aromatic_rings": Lipinski.NumAromaticRings(mol),
        "num_atoms": mol.GetNumAtoms(),
        "num_heavy_atoms": mol.GetNumHeavyAtoms(),
        "formal_charge": Chem.GetFormalCharge(mol),
        "synthetic_accessibility": calculate_sa_score(mol),
    }


def calculate_sa_score(mol) -> float:
    """Estimate synthetic accessibility (1=easy, 10=hard)."""
    # Simplified SAScore implementation
    # In production, use full SAScore from RDKit contrib
    num_rings = Lipinski.NumRings(mol)
    num_stereo = rdMolDescriptors.CalcNumAtomStereoCenters(mol)
    complexity = num_rings + num_stereo * 0.5
    return min(10.0, 1.0 + complexity * 0.3)


def check_constraints(props: Dict[str, float], constraints: Dict[str, Any]) -> bool:
    """Check if molecule satisfies all constraints."""
    if props["mol_weight"] < constraints.get("mw_min", 0):
        return False
    if props["mol_weight"] > constraints.get("mw_max", 1000):
        return False
    if props["logp"] < constraints.get("logp_min", -5):
        return False
    if props["logp"] > constraints.get("logp_max", 10):
        return False
    if props["tpsa"] < constraints.get("tpsa_min", 0):
        return False
    if props["tpsa"] > constraints.get("tpsa_max", 200):
        return False
    if props["synthetic_accessibility"] > constraints.get("sa_max", 10):
        return False

    # Check required substructures
    mol = Chem.MolFromSmiles(props.get("smiles", ""))
    if mol:
        for smarts in constraints.get("required_substructures", []):
            pattern = Chem.MolFromSmarts(smarts)
            if not mol.HasSubstructMatch(pattern):
                return False

        for smarts in constraints.get("forbidden_substructures", []):
            pattern = Chem.MolFromSmarts(smarts)
            if mol.HasSubstructMatch(pattern):
                return False

    return True


@app.post("/generate", response_model=GenerationResponse)
async def generate_molecules(req: GenerationRequest):
    """Generate novel molecules using VAE with constraints."""
    logger.info(f"Generating {req.num_molecules} molecules with constraints: {req.constraints}")

    generated = []
    num_valid = 0
    max_attempts = req.num_molecules * 10  # Oversample to account for invalid/filtered

    with torch.no_grad():
        for attempt in range(max_attempts):
            if len(generated) >= req.num_molecules:
                break

            # Sample from latent space
            z = torch.randn(1, vae_model.latent_dim).to(device)

            # Decode to SMILES
            smiles = vae_model.decode(z)[0]

            # Validate SMILES
            mol = Chem.MolFromSmiles(smiles)
            if not mol:
                continue

            # Canonicalize SMILES
            canonical_smiles = Chem.MolToSmiles(mol)

            # Calculate properties
            props = calculate_properties(canonical_smiles)
            if not props:
                continue

            props["smiles"] = canonical_smiles

            # Check constraints
            if not check_constraints(props, req.constraints):
                continue

            # Calculate InChI Key for uniqueness
            inchi_key = Chem.MolToInchiKey(mol)
            props["inchi_key"] = inchi_key

            # Avoid duplicates
            if any(m["inchi_key"] == inchi_key for m in generated):
                continue

            generated.append(props)
            num_valid += 1

    logger.info(f"Generated {len(generated)} valid molecules out of {max_attempts} attempts")

    return GenerationResponse(
        molecules=generated,
        num_generated=len(generated),
        num_valid=num_valid,
    )


@app.get("/health")
async def health_check():
    return {"status": "healthy", "model_loaded": vae_model is not None, "device": str(device)}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–5)

**Day 1: Project initialization and Rust backend scaffold**
```bash
cargo init molforge-api
cd molforge-api
cargo add axum tokio serde serde_json sqlx uuid chrono tower-http jsonwebtoken argon2
cargo add --features postgres,runtime-tokio-rustls,uuid,chrono,json sqlx
cargo add aws-sdk-s3 redis
```
- `src/main.rs` — Axum app with CORS, tracing, graceful shutdown
- `src/config.rs` — Environment config (DATABASE_URL, JWT_SECRET, S3_BUCKET, REDIS_URL)
- `src/state.rs` — AppState struct (PgPool, Redis, S3Client)
- `src/error.rs` — ApiError enum with IntoResponse
- `Dockerfile` — Multi-stage build (builder + runtime)
- `docker-compose.yml` — PostgreSQL with RDKit cartridge, Redis, MinIO

**Day 2: Database schema and RDKit setup**
- Build custom PostgreSQL Docker image with RDKit cartridge installed
- `migrations/001_initial.sql` — All 12 tables with RDKit extension
- `src/db/mod.rs` — Database pool initialization
- `src/db/models.rs` — All SQLx structs with FromRow derives
- Run `sqlx migrate run` to apply schema
- Test RDKit functions: similarity search, substructure match

**Day 3: Authentication system**
- `src/auth/mod.rs` — JWT token generation and validation middleware
- `src/auth/oauth.rs` — Google and ORCID OAuth 2.0 flow handlers
- `src/api/handlers/auth.rs` — Register, login, OAuth callback, refresh, me endpoints
- Password hashing with argon2 (cost factor 19, parallelism 2)
- JWT with 24h access token + 30d refresh token
- Auth middleware extracts Claims from Authorization header

**Day 4: User and project CRUD**
- `src/api/handlers/users.rs` — Get profile, update profile, verify academic email
- `src/api/handlers/projects.rs` — Create, list, get, update, archive project
- `src/api/handlers/orgs.rs` — Create org, invite member, list members, remove member
- `src/api/router.rs` — All route definitions with auth middleware
- Integration tests: auth flow, project CRUD, authorization checks

**Day 5: S3 integration and file upload**
- `src/services/s3.rs` — Upload file, download file, generate presigned URLs
- `src/api/handlers/upload.rs` — Multipart file upload for SDF/PDB files
- Validation: file size limits (100MB), allowed extensions (.sdf, .pdb, .mol2, .smi)
- Storage structure: `s3://bucket/{user_id}/projects/{project_id}/{filename}`
- Test: upload SDF, download via presigned URL

### Phase 2 — Cheminformatics Service (Days 6–10)

**Day 6: Python cheminformatics service scaffold**
```bash
mkdir chem_service
cd chem_service
poetry init
poetry add fastapi uvicorn rdkit-pypi requests pydantic
```
- `chem_service/main.py` — FastAPI app with health check
- `chem_service/rdkit_ops.py` — RDKit wrapper functions
- Docker image with RDKit conda environment
- Deploy as service: `docker-compose` with chem_service on port 8001
- Test: call `/health` from Rust API

**Day 7: Molecular property calculation**
- `chem_service/routes/properties.py` — POST `/calculate` endpoint
- Calculate: MW, logP, TPSA, HBD, HBA, rotatable bonds, aromatic rings, formal charge
- Calculate synthetic accessibility (SAScore) using RDKit contrib
- Generate Morgan fingerprint (radius 2, 2048 bits) as bit vector
- Return JSON with all properties + fingerprint as hex string
- Integration: Rust API calls chem service, stores in PostgreSQL

**Day 8: Molecular standardization and validation**
- `chem_service/routes/standardize.py` — POST `/standardize` endpoint
- Pipeline: salt stripping → tautomer canonicalization → charge assignment → stereochemistry validation
- Handle SMILES, SDF, MOL2 input formats
- Return canonical SMILES + InChI Key
- Error handling for invalid structures

**Day 9: SDF/SMILES file parsing and bulk import**
- `chem_service/routes/import.py` — POST `/import/sdf` and `/import/smiles`
- Parse SDF file: extract molecules, calculate properties for each
- Parse SMILES file (CSV with SMILES column): validate, standardize, calculate properties
- Return JSON array of molecules with properties
- Rust API: call chem service → insert molecules in batches (1000 at a time) → update library count
- Test: import ZINC fragment library (10K molecules)

**Day 10: Chemical search (similarity and substructure)**
- `chem_service/routes/search.py` — POST `/search/similarity`, POST `/search/substructure`
- Similarity: convert query SMILES to Morgan FP, use RDKit Tanimoto similarity
- Substructure: convert SMARTS to query mol, use RDKit HasSubstructMatch
- Rust API delegates search to PostgreSQL RDKit cartridge (direct SQL query)
- Test search performance: 100K molecule library, similarity search <100ms

### Phase 3 — Protein Target Preparation (Days 11–14)

**Day 11: PDB fetch and upload**
- `src/api/handlers/targets.rs` — POST create target (fetch from RCSB or upload)
- Integration with RCSB PDB API: fetch by PDB ID (e.g., "1HIV")
- Download PDB/mmCIF file, upload to S3
- Create protein_targets record with status='pending'
- Test: fetch 1HIV, upload custom PDB file

**Day 12: Protein preparation service (Python)**
- `protein_service/` — New Python FastAPI service
- Dependencies: BioPython, OpenBabel, fpocket (via subprocess)
- `protein_service/routes/prepare.py` — POST `/prepare` endpoint
- Preparation pipeline:
  1. Add hydrogens (OpenBabel or PDB2PQR)
  2. Assign charges (AMBER ff14SB via OpenBabel)
  3. Remove waters beyond 5Å from ligand/active site
  4. Convert to PDBQT format for docking
- Upload prepared file to S3, return URL
- Update protein_targets record: prepared_file_url, status='completed'

**Day 13: Binding site detection**
- `protein_service/routes/binding_sites.py` — POST `/detect_binding_sites`
- Run fpocket algorithm to detect pockets
- Parse fpocket output: pocket center coordinates, residue list, volume, druggability score
- Return top 3 pockets ranked by druggability
- Frontend: display pockets in 3D viewer, user selects one
- Update protein_targets: grid_center_{x,y,z}, binding_site_residues

**Day 14: Grid box definition UI**
- Frontend: 3D protein viewer (NGL Viewer) with grid box overlay
- Interactive grid box editor: drag handles to resize, drag center to move
- Default grid size: 20Ų × 20Ų × 20Ų centered on detected pocket
- API: PATCH `/targets/:id/grid` to update grid parameters
- Validation: grid must encompass at least part of binding site residues

### Phase 4 — AI Molecular Generation (Days 15–20)

**Day 15: VAE model training preparation**
- Download ChEMBL 34 dataset (2.3M drug-like molecules)
- Preprocess: filter by Lipinski's Rule of Five, remove salts, canonicalize SMILES
- Split: 90% train, 5% validation, 5% test
- Convert SMILES to tokenized sequences (char-level or SELFIES encoding)
- Save preprocessed dataset as PyTorch tensors

**Day 16: VAE architecture implementation**
- `generation_service/model.py` — Molecular VAE architecture
- Encoder: bidirectional LSTM → latent distribution (mean, logvar)
- Latent space: 256 dimensions
- Decoder: LSTM with attention → SMILES sequence (teacher forcing during training)
- Loss: reconstruction loss (cross-entropy) + KL divergence regularization
- Training loop with PyTorch Lightning

**Day 17: VAE training and evaluation**
- Train VAE on 8× A10G GPUs for 48 hours (20 epochs)
- Validation metrics: reconstruction accuracy, validity rate, uniqueness, diversity (Tanimoto distance)
- Target performance: >95% validity, >90% uniqueness, mean Tanimoto <0.6
- Save best checkpoint to S3
- Evaluate on test set: sample 10K molecules, check property distributions

**Day 18: Generation service API**
- `generation_service/main.py` — FastAPI service for generation
- Load trained VAE model in GPU memory
- POST `/generate` endpoint: accepts num_molecules, constraints (MW, logP, TPSA, SAScore, scaffold SMARTS)
- Rejection sampling loop: sample from latent space → decode → check constraints → keep if valid
- Return JSON array of generated molecules with properties
- Test: generate 1000 molecules with MW 200–500, logP 0–5, validate constraints

**Day 19: Generation job orchestration (Rust)**
- `src/api/handlers/generation.rs` — POST create generation run
- Check plan limits (academic: 500/month, researcher: 25K/month)
- Create generation_runs record, create job record, enqueue to Redis
- `generation_worker/` — Python worker consuming from Redis "generation:jobs" queue
- Worker: call generation service → save molecules to DB → create library → update run status
- WebSocket progress updates: num_generated, validity_rate

**Day 20: Generation UI and results**
- Frontend: generation studio page with constraint sliders
- Real-time progress bar and property distribution histograms (updating live via WebSocket)
- Results gallery: grid of generated molecules with structure thumbnails
- Click molecule → view details (properties, 2D structure, add to library)
- Export generated library as SDF file

### Phase 5 — Molecular Docking (Days 21–27)

**Day 21: AutoDock Vina installation and testing**
- Docker image with AutoDock Vina 1.2.5, Meeko (ligand prep), OpenBabel
- Test Vina on benchmark: 1HIV receptor + known inhibitor
- Validate scoring: reproduce published binding affinity within 1 kcal/mol
- Benchmark performance: 32-core CPU can dock ~100 molecules/hour

**Day 22: Docking service core (ligand preparation)**
- `docking_service/routes/dock.py` — POST `/dock/single` endpoint
- Input: SMILES, receptor PDBQT, grid parameters, Vina parameters
- Ligand preparation pipeline:
  1. SMILES → 3D structure (RDKit conformer generation, UFF optimization)
  2. Add Gasteiger charges
  3. Convert to PDBQT (Meeko)
- Run Vina subprocess, capture stdout/stderr
- Parse Vina output: best score, all poses
- Return JSON: vina_score, num_poses, pose_pdbqt_string

**Day 23: Docking worker implementation**
- Complete docking_worker/main.py (from Architecture Deep-Dive section 2)
- Redis queue consumption: priority queue (Lab plan jobs first)
- Parallel docking: spawn 8 worker processes, each docking 1 molecule at a time
- Progress updates: every 10 molecules, update run status and broadcast WebSocket
- Error handling: skip failed molecules, log error, continue with remaining

**Day 24: Docking results storage and ranking**
- After docking completes, insert all results into docking_results table
- Calculate consensus_score = vina_score (simple for MVP; v1.1 adds ML re-scoring)
- Rank results by consensus_score
- Upload all docking poses as multi-model PDBQT file to S3
- Generate results CSV with columns: rank, SMILES, vina_score, num_poses, pose_url

**Day 25: Interaction analysis (hydrogen bonds, hydrophobic contacts)**
- `docking_service/routes/interactions.py` — POST `/analyze_interactions`
- Input: receptor PDB, ligand pose PDB
- Use BioPython or ProLIF to detect:
  - Hydrogen bonds (distance <3.5Ų, angle >120°)
  - Hydrophobic contacts (distance <4.5Ų between hydrophobic atoms)
  - Salt bridges (distance <4.0Ų between charged groups)
  - π-π stacking (aromatic ring centroids <5.5Ų)
- Return JSON: interactions array with type, residue, atom names, distance
- Store in docking_results.interactions JSONB

**Day 26: Docking results UI**
- Results table: rank, 2D structure thumbnail, vina_score, ADMET flags (placeholder), actions
- Sort/filter: by score, by ADMET property
- Pagination: load 100 results at a time, infinite scroll
- Click row → open 3D viewer with binding pose
- Scatter plot: vina_score vs. logP (interactive, click point → open 3D viewer)

**Day 27: 3D protein-ligand viewer (NGL Viewer integration)**
- `frontend/src/components/ProteinLigandViewer.tsx`
- Load NGL Viewer, fetch receptor PDB and ligand pose from S3 presigned URLs
- Display protein as cartoon + surface (hydrophobicity coloring)
- Display ligand as ball-and-stick
- Annotate interactions: H-bonds as dashed yellow lines, labels with residue names
- Controls: rotate, zoom, toggle surface, toggle interactions, screenshot (PNG export)

### Phase 6 — ADMET Prediction (Days 28–32)

**Day 28: ADMET model training data preparation**
- Collect public ADMET datasets:
  - Caco-2: 1,200 compounds from literature
  - BBB: 2,000 compounds (PubChem BioAssay)
  - hERG: 8,000 compounds (ChEMBL hERG assays IC50 <10µM = positive)
  - CYP3A4: 12,000 compounds (ChEMBL inhibition data)
  - AMES: 7,800 compounds (Hansen et al. 2009)
  - Hepatotoxicity: 3,600 compounds (DILIrank)
  - Solubility: 10,000 compounds (ESOL, AqSolDB)
  - PPB: 2,400 compounds (literature aggregation)
  - HIA: 900 compounds (literature)
  - Half-life: 1,100 compounds (Lombardo et al.)
- Compute molecular descriptors: 200 RDKit descriptors + Morgan fingerprints
- Split each dataset: 80% train, 10% val, 10% test

**Day 29: ADMET model training (XGBoost ensemble)**
- Train separate XGBoost classifier/regressor for each endpoint
- Hyperparameter tuning: grid search on validation set (max_depth, learning_rate, n_estimators)
- Regression endpoints: Caco-2, solubility, PPB, half-life (RMSE metric)
- Classification endpoints: BBB, hERG, CYP3A4, AMES, hepatotoxicity (ROC-AUC, F1)
- Target performance: ROC-AUC >0.80 for classifiers, RMSE <0.5 log units for regressors
- Save all 10 models to S3 as pickle files

**Day 30: ADMET service API**
- `admet_service/main.py` — FastAPI service
- Load all 10 XGBoost models in memory at startup
- POST `/predict` endpoint: input SMILES, output predictions for all 10 endpoints
- Calculate molecular descriptors (RDKit), run inference for all models
- Applicability domain check: compare query fingerprint to training set (Tanimoto >0.3 to nearest neighbor)
- Return JSON: {endpoint: {value, confidence, flag: green/amber/red}}
- Traffic light logic: green if favorable, red if violates threshold, amber otherwise

**Day 31: ADMET batch prediction**
- POST `/predict/batch` — input array of SMILES, return array of predictions
- Parallelize: use multiprocessing to predict 100 molecules at a time
- Test: predict ADMET for 10K molecule library in <5 minutes
- Rust API: POST batch to ADMET service → insert admet_profiles records
- Usage tracking: record admet_predictions count

**Day 32: ADMET results UI**
- ADMET profile card: traffic light grid (10 endpoints)
- Hover over each cell: tooltip with value, confidence, interpretation
- Applicability domain warning: "This prediction is outside the model's training domain"
- Batch ADMET button: predict for all docking hits
- Results table: add ADMET flags columns (hERG, AMES, CYP3A4)
- Filter: show only molecules with all-green ADMET

### Phase 7 — Frontend Polish + Billing (Days 33–38)

**Day 33: Molecular editor (Ketcher integration)**
- Embed Ketcher 2.18 in `frontend/src/components/MolecularEditor.tsx`
- SMILES input bar: paste SMILES → render in Ketcher
- Draw mode: use Ketcher tools to draw structure → export SMILES
- Import SDF: upload file → parse → load first molecule in Ketcher
- Export: button to download as MOL, SDF, or SMILES file
- Integration: Ketcher SMILES output → call chem service `/standardize` → save to library

**Day 34: Project dashboard and navigation**
- Dashboard cards: active targets (count), total molecules (count), docking runs (completed/running), generation runs (count)
- Recent activity feed: "Docking run #42 completed with 15 hits", "Generated 1000 molecules"
- Quick actions: buttons for "New Target", "Upload Library", "Generate Molecules", "Start Docking"
- Project settings: rename, archive, delete, share public link

**Day 35: Notifications and WebSocket UI**
- `frontend/src/hooks/useWebSocket.ts` — WebSocket connection to Axum WS endpoint
- Subscribe to job progress updates: generation, docking, ADMET batch
- Toast notifications: "Docking completed: 23 hits found", "Generation failed: invalid constraints"
- Progress indicators: circular progress for running jobs in dashboard

**Day 36: Stripe billing integration (Rust)**
- `src/billing/mod.rs` — Stripe client wrapper
- `src/api/handlers/billing.rs` — Create checkout session, customer portal, webhook handler
- Plan tiers:
  - Academic (free): .edu email required, 500 molecules docked/month, 500 generated/month
  - Researcher ($199/mo): 25K docked, 25K generated
  - Lab ($799/mo): 250K docked, 250K generated, 15 team members
- Usage enforcement: check usage_records before allowing job creation
- Webhook handler: subscription created/updated/cancelled → update users.plan

**Day 37: Billing UI and plan upgrade flow**
- Settings page: current plan, usage this month (molecules docked, generated), upgrade button
- Upgrade modal: compare plans table, "Upgrade to Researcher" button
- Stripe Checkout: redirect to Stripe hosted checkout page
- Success redirect: back to app, show "Subscription active" banner
- Customer portal link: manage payment methods, cancel subscription

**Day 38: Export and reporting**
- Export molecules as SDF: fetch all molecules in library → generate SDF file server-side → download
- Export docking results as CSV: include rank, SMILES, vina_score, ADMET flags
- PDF report generation: summary page with project name, target, top 10 hits, ADMET profile table
- Use library: wkhtmltopdf or headless Chrome for PDF rendering
- Download link: presigned S3 URL for generated report

### Phase 8 — Testing, Deployment, Launch (Days 39–42)

**Day 39: Integration testing and benchmarking**
- End-to-end test: create project → upload target → generate 100 molecules → dock → view results
- Performance benchmarks:
  - Generation: 1000 molecules in <5 minutes (GPU)
  - Docking: 100 molecules in <2 hours (10× 32-core workers)
  - ADMET: 1000 molecules in <2 minutes (batch mode)
  - Chemical search: similarity search in 100K library <100ms
- Load test: 50 concurrent users, simulate full workflow

**Day 40: Documentation and onboarding**
- User documentation: getting started guide, video tutorial (5 min screencast)
- API documentation: auto-generated OpenAPI spec from Axum routes
- Example project: HIV-1 protease with 100 FDA-approved drugs as reference library
- Academic verification flow: send verification email to .edu address, manual review queue

**Day 41: Production deployment (AWS)**
- Infrastructure as Code (Terraform):
  - ECS Fargate for Rust API (2× t3.large, auto-scaling 2–10)
  - ECS Fargate for Python services (chem, generation, protein, admet, docking_worker)
  - RDS PostgreSQL 16 with RDKit extension (db.r6g.xlarge)
  - ElastiCache Redis 7 (cache.r6g.large)
  - S3 bucket with versioning and lifecycle policies
  - CloudFront CDN for frontend static assets
- Secrets Manager: JWT_SECRET, database passwords, Stripe API keys
- Monitoring: CloudWatch alarms for CPU, memory, error rate
- Deploy frontend to S3 + CloudFront

**Day 42: Beta launch and feedback collection**
- Announce on Twitter/X #CompChem, Reddit r/cheminformatics, CCL mailing list
- Beta tester onboarding: 20 academic users, 5 biotech researchers
- Feedback form: in-app survey after first docking run
- Analytics: Mixpanel events (project_created, molecules_generated, docking_started, docking_completed)
- Bug triage: Sentry error tracking, prioritize critical bugs
- Office hours: weekly Zoom call with beta testers for live Q&A

---

## Validation Benchmarks

### 1. Docking Accuracy (DUD-E Benchmark)
**Metric:** Docking enrichment factor at 1% false positive rate across 102 DUD-E targets
**Target:** Enrichment factor (EF 1%) ≥ 15.0
**Method:** For each DUD-E target, dock 50 actives + 3,000 decoys, rank by Vina score, calculate EF at top 1%
**Baseline:** AutoDock Vina standalone EF 1% = 18.2 ± 6.3 (published)
**Expected:** MolForge EF 1% ≥ 15.0 (allowing for cloud infrastructure variance)

### 2. AI Generation Validity and Diversity
**Metric:** Validity rate, uniqueness, internal diversity (mean pairwise Tanimoto distance)
**Target:** Validity ≥95%, uniqueness ≥90%, mean Tanimoto ≤0.6
**Method:** Generate 10,000 molecules with constraints MW 200–500, logP 0–5, TPSA 20–140. Calculate metrics on output set.
**Baseline:** Published VAE models: validity ~93%, uniqueness ~88%, diversity 0.55–0.65
**Expected:** Validity ≥95% (with rejection sampling), uniqueness ≥90%, mean Tanimoto ~0.58

### 3. ADMET Prediction Accuracy
**Metric:** ROC-AUC for hERG inhibition classification, RMSE for Caco-2 permeability regression
**Target:** hERG ROC-AUC ≥0.82, Caco-2 RMSE ≤0.55 log units
**Method:** Hold-out test set (10% of training data), measure performance on unseen molecules
**Baseline:** Published XGBoost models: hERG AUC 0.78–0.85, Caco-2 RMSE 0.50–0.65
**Expected:** hERG AUC ≥0.82 (ensemble + hyperparameter tuning), Caco-2 RMSE ≤0.55

### 4. Docking Throughput
**Metric:** Molecules docked per hour per 32-core CPU worker
**Target:** ≥80 molecules/hour (exhaustiveness=8, num_modes=9)
**Method:** Benchmark docking 1000 diverse molecules against 1HIV receptor, measure wall-clock time
**Baseline:** AutoDock Vina 1.2.5 on AWS c6i.8xlarge: ~85–95 molecules/hour
**Expected:** ≥80 molecules/hour (accounting for network I/O, S3 upload overhead)

### 5. End-to-End Workflow Latency
**Metric:** Time from "Generate 1000 molecules" → docking complete → ADMET predictions saved
**Target:** <60 minutes for 1000-molecule workflow
**Method:** Full workflow with 1 GPU worker (generation), 10 CPU workers (docking), 2 CPU workers (ADMET)
**Breakdown:** Generation 5 min + docking 40 min + ADMET 2 min = 47 min (target <60 min with buffer)
**Expected:** 50–55 minutes typical, <60 min 95th percentile

---

## Risk Mitigation

### Technical Risks

**Risk:** RDKit cartridge performance degrades at >1M molecules per library
**Mitigation:** Partition large libraries across multiple tables with hash-based sharding on InChI Key. Benchmark at 10M molecules during Phase 2. If performance insufficient, migrate chemical search to dedicated Elasticsearch cluster with chemical fingerprint plugin.

**Risk:** AutoDock Vina false positive rate too high (poor enrichment)
**Mitigation:** Implement consensus scoring in v1.1 (combine Vina + RF-Score + OnionNet-2). Offer ML re-scoring using XGBoost trained on PDBbind refined set (improves EF 1% by 20–30% per literature). Add FEP surrogate model in v1.2 for top hits.

**Risk:** VAE generates synthetically infeasible molecules despite SAScore constraint
**Mitigation:** Integrate retrosynthesis prediction (AiZynthFinder or commercial API) in v1.1. Flag molecules with no feasible route within 5 steps. Partner with Enamine to validate top candidates against REAL space (5.6B synthesizable molecules).

**Risk:** GPU costs for AI generation erode margins
**Mitigation:** Batch generation requests: accumulate 10 concurrent requests → run single large batch → amortize GPU startup cost. Use AWS Inferentia2 instances ($0.75/hr vs $1.10/hr for A10G) with optimized ONNX model. Cache frequent constraint combinations (e.g., "Lipinski-compliant") to avoid re-generation.

**Risk:** Protein preparation fails for non-standard PDB files (missing residues, non-canonical amino acids)
**Mitigation:** Detect common issues (missing atoms, alternate conformations) and guide user to fix manually. Integrate PDBFixer for automated repair. For AlphaFold models, auto-select high pLDDT regions only. Provide manual grid box override if auto-detection fails.

### Business Risks

**Risk:** Pharma companies refuse cloud deployment due to IP concerns
**Mitigation:** Enterprise tier includes private VPC deployment (isolated AWS account) and on-premise option (Docker Compose bundle with perpetual license). Obtain ISO 27001 and SOC 2 Type II certifications by Month 9. Implement customer-managed encryption keys (BYOK).

**Risk:** Free academic tier cannibalizes paid conversions
**Mitigation:** Academic tier limited to 1 project, 500 molecules/month docked, watermark on exports, no API access. Upgrade prompts when hitting limits. Target 5% free→researcher conversion, 15% researcher→lab conversion. Monitor conversion funnels weekly.

**Risk:** Competition from well-funded AI drug discovery startups (Insilico, Recursion, Exscientia)
**Mitigation:** Position as horizontal platform (self-service SaaS) vs their vertical integration (drug development companies). Emphasize transparency (no black box), pricing (10× cheaper), and accessibility (no sales calls, instant signup). Partner with CROs to offer MolForge + wet-lab validation bundles.

---

## Post-MVP Roadmap (v1.1–v1.3)

### v1.1 (Months 3–4)
- Reinforcement learning generation with target-specific scoring function
- Scaffold hopping: generate analogs by replacing scaffold while preserving R-groups
- ML re-scoring for docking: XGBoost trained on PDBbind refined set
- Full 25-endpoint ADMET panel (add clearance, VDss, skin sensitization, etc.)
- Matched molecular pair (MMP) analysis for SAR insights
- Hit-to-lead pipeline Kanban board
- Patent landscape check (SureChEMBL structural similarity search)

### v1.2 (Months 5–7)
- Free-energy perturbation (FEP) lite: ML surrogate for relative binding free energies
- Collaborative notebooks with rich text, embedded structures, and plots
- Multi-target docking: selectivity optimization across 2–5 targets simultaneously
- Retrosynthesis prediction: flag molecules with no feasible synthetic route
- ELN integration: Benchling and Dotmatics API connectors
- Ensemble docking: use multiple protein conformations (MD snapshots)

### v1.3 (Months 8–10)
- Pharmacophore-based screening: define 3D pharmacophore, screen libraries
- Molecular dynamics integration: submit top hits to GROMACS for stability assessment
- Custom ADMET model training: upload proprietary compound-activity data, train private models
- Advanced visualization: electrostatic potential surface, RMSD overlay, trajectory playback
- Enterprise features: SSO/SAML, audit logs, 21 CFR Part 11 compliance (e-signatures)
- API v2: RESTful API with rate limits, OAuth2 client credentials, webhooks

---

## Success Metrics

| Metric | Month 3 | Month 6 | Month 12 |
|--------|---------|---------|----------|
| Registered users | 500 | 2,000 | 8,000 |
| Academic verified users | 300 | 1,200 | 5,000 |
| Active projects (monthly) | 150 | 600 | 2,500 |
| Molecules generated (cumulative) | 500K | 5M | 50M |
| Docking runs completed (cumulative) | 200 | 2,000 | 15,000 |
| Paying customers | 8 | 40 | 180 |
| MRR | $1,600 | $15,000 | $80,000 |
| Free → Paid conversion | 3% | 4% | 7% |
| Churn rate (monthly) | <8% | <6% | <4% |
| Customer NPS | 40+ | 50+ | 60+ |

---

**Total Timeline:** 42 days (6 weeks) for full MVP with 8 phases
**Team:** 1 full-stack engineer (Rust + React) + 1 ML engineer (Python/PyTorch) + 0.5 DevOps (AWS/infrastructure)
**Estimated Cost:** $25K development + $5K AWS infrastructure (Month 1) + $3K/month ongoing hosting
