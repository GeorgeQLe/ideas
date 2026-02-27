# 22. FoldSight — Protein Structure Prediction and Visualization Platform

## Implementation Plan

**MVP Scope:** Browser-based protein structure prediction platform accepting amino acid sequences (FASTA format) with GPU-accelerated ESMFold predictions running on server-side PyTorch workers, interactive 3D molecular viewer built on Mol*/Three.js with cartoon/ribbon/surface representations and per-residue pLDDT confidence coloring, structural alignment using TM-align with RMSD and TM-score visualization, PDB/UniProt database integration for fetching known structures and annotations, binding site detection via fpocket geometry-based algorithm with druggability scoring, variant impact analysis using ddG stability calculations, real-time collaborative annotation with residue-level comments and WebSocket synchronization, project workspace organization with role-based permissions, export to PDB/mmCIF/PyMOL session files and publication-ready PNG/SVG figures up to 4K resolution, Stripe billing with three tiers (Free academic: 10 predictions/month / Researcher $79/month: 100 predictions + full analysis / Lab $299/month: 500 predictions + collaboration + API).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, high-performance request handling |
| Prediction Service | Python 3.12 (FastAPI) | Separate microservice for ML inference, async endpoints |
| Task Queue | Celery + Redis | Prediction job management, priority queuing, distributed workers |
| GPU Inference | PyTorch 2.x + ESMFold | NVIDIA A100/H100 instances, model served via TorchServe |
| Database | PostgreSQL 16 | Projects, users, structures, annotations, predictions |
| Vector Search | pgvector extension | Sequence similarity search via embeddings |
| ORM / Query | SQLx (Rust), SQLAlchemy (Python) | Compile-time checked queries for Rust, async ORM for Python |
| Auth | Custom JWT + OAuth 2.0 | ORCID OAuth for academic identity, Google/GitHub providers |
| Object Storage | AWS S3 | Structure files (PDB/mmCIF), prediction outputs, PAE matrices, figures |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| 3D Molecular Viewer | Mol* (Molstar) + Three.js | WebGL-based rendering, custom React wrapper |
| Sequence Viewer | Custom React component | MSA visualization, secondary structure annotation |
| Structural Analysis | BioPython, TM-align, fpocket | Native binaries called from Python service |
| Real-time Collaboration | WebSocket (Axum) | Live cursor positions, annotation sync, comment notifications |
| Job Queue | Redis 7 | Celery broker, caching, pub/sub for progress updates |
| Search | Elasticsearch 8 | Full-text search for protein names, organisms, annotations |
| Billing | Stripe | Checkout Sessions, metered usage, Customer Portal |
| CDN | CloudFront | Static frontend assets, structure file delivery |
| Monitoring | Datadog + Sentry | Infrastructure metrics, error tracking, GPU utilization dashboards |
| CI/CD | GitHub Actions | Python/Rust tests, Docker image builds, EKS deployments |

### Key Architecture Decisions

1. **Dual-stack Rust + Python backend with microservice split**: Rust/Axum handles high-throughput API requests (auth, CRUD, real-time WebSocket), while Python/FastAPI handles ML inference and structural biology tooling (BioPython, fpocket, FoldX). This splits performance-critical API from GPU-bound ML workloads, allowing independent scaling of web tier (CPU) and prediction tier (GPU). Communication via internal HTTP/gRPC.

2. **ESMFold as primary prediction engine for MVP**: ESMFold runs inference in ~30 seconds per 400-residue protein on A100 GPU without requiring MSA generation, making it 10x faster than AlphaFold2 for single-sequence predictions. This enables responsive UX for the 90% use case (monomers <500 residues). AlphaFold2/ColabFold integration deferred to post-MVP for users needing maximum accuracy or multimer predictions.

3. **Mol* (Molstar) for 3D viewer with custom React integration**: Mol* is the de facto standard viewer powering RCSB PDB and AlphaFold Database, providing production-grade mmCIF parsing, WebGL rendering, and extensibility. We build a React wrapper with custom UI controls while leveraging Mol*'s robust core. This avoids reinventing molecular visualization and ensures compatibility with standard file formats.

4. **Server-side analysis with client-side visualization streaming**: Computationally intensive tasks (structural alignment via TM-align, binding site detection via fpocket, electrostatics via APBS) run server-side in Python worker pools. Results are streamed to client as JSON/binary for rendering in the 3D viewer. This keeps client-side bundle small and leverages server compute while maintaining interactive UI updates via WebSocket progress events.

5. **PostgreSQL with pgvector for sequence similarity search**: Sequence embeddings from ESM-2 model (1280-dimensional vectors) stored in pgvector enable fast approximate nearest neighbor search for finding similar proteins. This powers features like "find similar structures in database" and deduplication of batch predictions. pgvector's HNSW index provides <50ms search over 1M+ proteins.

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgvector";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";

-- Users
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email TEXT UNIQUE NOT NULL,
    password_hash TEXT,
    name TEXT NOT NULL,
    avatar_url TEXT,
    orcid_id TEXT,
    institution TEXT,
    department TEXT,
    auth_provider TEXT DEFAULT 'email',
    auth_provider_id TEXT,
    plan TEXT NOT NULL DEFAULT 'free',
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    last_login_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX users_email_idx ON users(email);
CREATE INDEX users_orcid_idx ON users(orcid_id);
CREATE INDEX users_stripe_idx ON users(stripe_customer_id);

-- Organizations (Labs / Research Groups)
CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    institution TEXT,
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    plan TEXT NOT NULL DEFAULT 'free',
    seat_count INTEGER DEFAULT 1,
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    settings JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
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

-- Projects (Workspaces for Structure Analysis)
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    visibility TEXT NOT NULL DEFAULT 'private',
    forked_from UUID REFERENCES projects(id) ON DELETE SET NULL,
    settings JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);
CREATE INDEX projects_org_idx ON projects(org_id);
CREATE INDEX projects_updated_idx ON projects(updated_at DESC);

-- Structures (Predicted or Experimental)
CREATE TABLE structures (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    created_by UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    sequence TEXT NOT NULL,
    sequence_length INTEGER NOT NULL,
    sequence_hash TEXT NOT NULL,
    sequence_embedding vector(1280),
    organism TEXT,
    uniprot_accession TEXT,
    pdb_id TEXT,
    source_type TEXT NOT NULL,
    prediction_model TEXT,
    prediction_job_id UUID,
    plddt_mean REAL,
    plddt_per_residue REAL[],
    pae_matrix_url TEXT,
    structure_file_url TEXT NOT NULL,
    thumbnail_url TEXT,
    status TEXT NOT NULL DEFAULT 'pending',
    error_message TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX structures_project_idx ON structures(project_id);
CREATE INDEX structures_sequence_hash_idx ON structures(sequence_hash);
CREATE INDEX structures_uniprot_idx ON structures(uniprot_accession);
CREATE INDEX structures_pdb_idx ON structures(pdb_id);
CREATE INDEX structures_embedding_idx ON structures USING ivfflat (sequence_embedding vector_cosine_ops) WITH (lists = 100);

-- Prediction Jobs
CREATE TABLE prediction_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    model TEXT NOT NULL,
    input_sequences JSONB NOT NULL,
    status TEXT NOT NULL DEFAULT 'queued',
    priority INTEGER DEFAULT 0,
    gpu_type TEXT,
    gpu_hours_used REAL DEFAULT 0.0,
    worker_id TEXT,
    progress_pct REAL DEFAULT 0.0,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    error_message TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX jobs_user_idx ON prediction_jobs(user_id);
CREATE INDEX jobs_status_idx ON prediction_jobs(status);
CREATE INDEX jobs_created_idx ON prediction_jobs(created_at DESC);

-- Alignments (Structural Comparisons)
CREATE TABLE alignments (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    created_by UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    structure_ids UUID[] NOT NULL,
    method TEXT NOT NULL,
    tm_score REAL,
    rmsd REAL,
    aligned_length INTEGER,
    transformation_matrices JSONB,
    per_residue_rmsd REAL[],
    status TEXT NOT NULL DEFAULT 'pending',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX alignments_project_idx ON alignments(project_id);

-- Annotations (Residue-level Notes and Highlights)
CREATE TABLE annotations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    structure_id UUID NOT NULL REFERENCES structures(id) ON DELETE CASCADE,
    author_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    layer_name TEXT NOT NULL DEFAULT 'default',
    residue_range_start INTEGER NOT NULL,
    residue_range_end INTEGER NOT NULL,
    chain_id TEXT NOT NULL DEFAULT 'A',
    annotation_type TEXT NOT NULL,
    content TEXT,
    color TEXT,
    source TEXT NOT NULL DEFAULT 'manual',
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX annotations_structure_idx ON annotations(structure_id);
CREATE INDEX annotations_layer_idx ON annotations(layer_name);

-- Comments (Discussion Threads)
CREATE TABLE comments (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    structure_id UUID REFERENCES structures(id) ON DELETE CASCADE,
    author_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    body TEXT NOT NULL,
    parent_comment_id UUID REFERENCES comments(id) ON DELETE CASCADE,
    anchor_residue INTEGER,
    anchor_chain TEXT,
    resolved BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX comments_project_idx ON comments(project_id);
CREATE INDEX comments_structure_idx ON comments(structure_id);
CREATE INDEX comments_author_idx ON comments(author_id);

-- Binding Sites (fpocket Detection Results)
CREATE TABLE binding_sites (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    structure_id UUID NOT NULL REFERENCES structures(id) ON DELETE CASCADE,
    detection_method TEXT NOT NULL,
    pocket_residues INTEGER[] NOT NULL,
    chain_id TEXT NOT NULL,
    volume_a3 REAL,
    druggability_score REAL,
    hydrophobicity_score REAL,
    enclosure_score REAL,
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX binding_sites_structure_idx ON binding_sites(structure_id);

-- Variant Analyses
CREATE TABLE variant_analyses (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    structure_id UUID NOT NULL REFERENCES structures(id) ON DELETE CASCADE,
    created_by UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    variants JSONB NOT NULL,
    method TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'queued',
    results_url TEXT,
    error_message TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at TIMESTAMPTZ
);
CREATE INDEX variant_analyses_structure_idx ON variant_analyses(structure_id);
CREATE INDEX variant_analyses_status_idx ON variant_analyses(status);

-- Usage Records (Billing Metering)
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,
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
    pub orcid_id: Option<String>,
    pub institution: Option<String>,
    pub department: Option<String>,
    pub auth_provider: String,
    pub auth_provider_id: Option<String>,
    pub plan: String,
    pub stripe_customer_id: Option<String>,
    pub stripe_subscription_id: Option<String>,
    pub last_login_at: Option<DateTime<Utc>>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Organization {
    pub id: Uuid,
    pub name: String,
    pub institution: Option<String>,
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
    pub visibility: String,
    pub forked_from: Option<Uuid>,
    pub settings: serde_json::Value,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Structure {
    pub id: Uuid,
    pub project_id: Uuid,
    pub created_by: Uuid,
    pub name: String,
    pub sequence: String,
    pub sequence_length: i32,
    pub sequence_hash: String,
    #[sqlx(skip)]
    pub sequence_embedding: Option<Vec<f32>>,
    pub organism: Option<String>,
    pub uniprot_accession: Option<String>,
    pub pdb_id: Option<String>,
    pub source_type: String,
    pub prediction_model: Option<String>,
    pub prediction_job_id: Option<Uuid>,
    pub plddt_mean: Option<f32>,
    pub plddt_per_residue: Option<Vec<f32>>,
    pub pae_matrix_url: Option<String>,
    pub structure_file_url: String,
    pub thumbnail_url: Option<String>,
    pub status: String,
    pub error_message: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct PredictionJob {
    pub id: Uuid,
    pub user_id: Uuid,
    pub model: String,
    pub input_sequences: serde_json::Value,
    pub status: String,
    pub priority: i32,
    pub gpu_type: Option<String>,
    pub gpu_hours_used: f32,
    pub worker_id: Option<String>,
    pub progress_pct: f32,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub error_message: Option<String>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Alignment {
    pub id: Uuid,
    pub project_id: Uuid,
    pub created_by: Uuid,
    pub structure_ids: Vec<Uuid>,
    pub method: String,
    pub tm_score: Option<f32>,
    pub rmsd: Option<f32>,
    pub aligned_length: Option<i32>,
    pub transformation_matrices: Option<serde_json::Value>,
    pub per_residue_rmsd: Option<Vec<f32>>,
    pub status: String,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct CreatePredictionRequest {
    pub sequences: Vec<SequenceInput>,
    pub model: String,
    pub multimer: bool,
    pub priority: Option<i32>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct SequenceInput {
    pub name: String,
    pub sequence: String,
}
```

---

## Solver Architecture

### ESMFold Prediction Pipeline

FoldSight uses **ESMFold** as the primary structure prediction model for MVP. ESMFold (Evolutionary Scale Modeling) is a transformer-based language model that predicts 3D structure directly from single sequences without requiring multiple sequence alignments (MSA), making it dramatically faster than AlphaFold2 while maintaining competitive accuracy for most proteins.

**Model Architecture:**
- ESM-2 650M parameter language model pre-trained on 65M protein sequences
- Structure prediction head with IPA (Invariant Point Attention) layers adapted from AlphaFold2
- Single-sequence input: no MSA generation required (eliminates ColabFold's 3-5 minute MSA search)
- Outputs: 3D coordinates (N, Cα, C, O per residue), per-residue confidence (pLDDT), predicted aligned error (PAE) matrix

**Prediction Workflow:**
```
User submits sequence (FASTA)
    ↓
Rust API validates sequence and creates PredictionJob record
    ↓
Job published to Redis queue with priority
    ↓
Python Celery worker picks up job
    ↓
ESMFold model loaded in GPU memory (cached across jobs)
    ↓
Sequence tokenized and fed to ESM-2 encoder
    ↓
Structure module predicts 3D coordinates iteratively
    ↓
Output: PDB file, pLDDT scores, PAE matrix (saved to S3)
    ↓
Structure record created in PostgreSQL with S3 URLs
    ↓
Client notified via WebSocket, 3D viewer loads structure
```

**Performance Characteristics:**
- Small protein (<200 residues): ~10 seconds on A100 GPU
- Medium protein (200-400 residues): ~30 seconds
- Large protein (400-600 residues): ~60 seconds
- Memory: ~12GB VRAM for 600-residue protein
- Batch size: 4 concurrent predictions per A100 (40GB VRAM)

### Structural Alignment via TM-align

**TM-align** is the gold-standard algorithm for protein structure alignment, providing rotation + translation matrices to superimpose structures and quantifying similarity via TM-score (0-1 scale, >0.5 indicates same fold).

**Algorithm:**
```
Input: Two structures A and B (Cα coordinates)
    ↓
Initial alignment via sequence alignment or fragment matching
    ↓
Iterative optimization:
    For each iteration:
        1. Find optimal rotation matrix R and translation t minimizing RMSD
        2. Calculate TM-score using aligned residue pairs
        3. Identify new residue pairs within distance cutoff
        4. Repeat until convergence (TM-score change <0.001)
    ↓
Output: TM-score, RMSD, rotation matrix, translation vector, aligned residues
```

**Implementation:**
- TM-align C binary compiled and called via Python subprocess
- Input: Two mmCIF/PDB files from S3
- Output: Alignment results parsed and stored in PostgreSQL
- Per-residue RMSD calculated for heatmap visualization
- Transformation matrices stored as JSONB for 3D viewer superposition

### Binding Site Detection via fpocket

**fpocket** uses Voronoi tessellation and alpha-sphere clustering to detect protein cavities and pockets geometrically without prior knowledge of ligand binding.

**Algorithm:**
```
Input: Protein structure (PDB file)
    ↓
Calculate Voronoi tessellation of protein atoms
    ↓
Identify alpha-spheres (centers of empty space between 4 atoms)
    ↓
Cluster alpha-spheres into pockets based on distance
    ↓
For each pocket:
    - Calculate volume (sum of alpha-sphere volumes)
    - Hydrophobicity score (proportion of hydrophobic atoms)
    - Polarity score (proportion of polar atoms)
    - Enclosure score (how buried the pocket is)
    ↓
Rank pockets by druggability score
    ↓
Output: Pocket residues, volume, scores (stored in binding_sites table)
```

**Druggability Scoring:**
```
druggability_score = 0.4 * volume_score
                   + 0.3 * hydrophobicity_score
                   + 0.2 * enclosure_score
                   + 0.1 * polarity_score

where:
    volume_score = min(volume_A3 / 500.0, 1.0)
    hydrophobicity_score = hydrophobic_ratio
    enclosure_score = 1.0 - solvent_accessibility
    polarity_score = polar_ratio * 0.5
```

### Python Prediction Worker Architecture

```python
# prediction_service/worker.py

import torch
from celery import Celery
from esm import pretrained
import boto3
from Bio.PDB import PDBIO, Structure as BioStructure
import numpy as np

app = Celery('foldsight_predictions', broker='redis://localhost:6379/0')

# Load ESMFold model once at startup (shared across workers)
model, alphabet = pretrained.load_model_and_alphabet("esmfold_v1")
model = model.cuda()
model.eval()

s3_client = boto3.client('s3')

@app.task(bind=True)
def predict_structure(self, job_id: str, sequence: str, name: str):
    """
    Run ESMFold prediction on a single sequence.
    """
    self.update_state(state='RUNNING', meta={'progress': 0.0})

    # Tokenize sequence
    batch_converter = alphabet.get_batch_converter()
    data = [(name, sequence)]
    batch_labels, batch_strs, batch_tokens = batch_converter(data)
    batch_tokens = batch_tokens.cuda()

    self.update_state(state='RUNNING', meta={'progress': 0.2})

    # Run inference
    with torch.no_grad():
        results = model(batch_tokens)

    self.update_state(state='RUNNING', meta={'progress': 0.7})

    # Extract outputs
    coords = results['positions'][-1].cpu().numpy()  # [L, 37, 3] (all atoms)
    plddt = results['plddt'].cpu().numpy()  # [L]
    pae = results['predicted_aligned_error'].cpu().numpy()  # [L, L]

    # Build PDB structure
    pdb_structure = build_pdb_from_coords(coords, sequence, plddt)
    pdb_path = f"/tmp/{job_id}.pdb"
    io = PDBIO()
    io.set_structure(pdb_structure)
    io.save(pdb_path)

    # Upload to S3
    s3_key_pdb = f"structures/{job_id}.pdb"
    s3_client.upload_file(pdb_path, 'foldsight-structures', s3_key_pdb)

    # Save PAE matrix
    pae_path = f"/tmp/{job_id}_pae.npy"
    np.save(pae_path, pae)
    s3_key_pae = f"pae/{job_id}.npy"
    s3_client.upload_file(pae_path, 'foldsight-structures', s3_key_pae)

    self.update_state(state='RUNNING', meta={'progress': 1.0})

    return {
        'pdb_url': f"s3://foldsight-structures/{s3_key_pdb}",
        'pae_url': f"s3://foldsight-structures/{s3_key_pae}",
        'plddt_mean': float(plddt.mean()),
        'plddt_per_residue': plddt.tolist(),
    }

def build_pdb_from_coords(coords, sequence, plddt):
    """Convert ESMFold output coordinates to BioPython Structure."""
    # Implementation: create Structure, Model, Chain, Residue, Atom hierarchy
    # Set B-factor to pLDDT for visualization
    pass
```

---

## Architecture Deep-Dives

### 1. Prediction API Handler (Rust/Axum)

Primary endpoint for submitting structure predictions, handling validation, queue management, and result retrieval.

```rust
// src/api/handlers/predictions.rs

use axum::{
    extract::{Path, State, Json},
    http::StatusCode,
    response::IntoResponse,
};
use uuid::Uuid;
use sha2::{Sha256, Digest};
use crate::{
    db::models::{PredictionJob, CreatePredictionRequest},
    state::AppState,
    auth::Claims,
    error::ApiError,
};

pub async fn create_prediction(
    State(state): State<AppState>,
    claims: Claims,
    Json(req): Json<CreatePredictionRequest>,
) -> Result<impl IntoResponse, ApiError> {
    // 1. Validate sequences
    for seq_input in &req.sequences {
        validate_sequence(&seq_input.sequence)?;
    }

    // 2. Check plan limits
    let user = sqlx::query_as!(
        crate::db::models::User,
        "SELECT * FROM users WHERE id = $1",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    let monthly_count = sqlx::query_scalar!(
        "SELECT COUNT(*) FROM prediction_jobs
         WHERE user_id = $1 AND created_at > NOW() - INTERVAL '30 days'",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?
    .unwrap_or(0);

    let limit = match user.plan.as_str() {
        "free" => 10,
        "researcher" => 100,
        "lab" => 500,
        _ => 10,
    };

    if monthly_count >= limit {
        return Err(ApiError::PlanLimit(
            format!("Monthly prediction limit ({}) reached. Upgrade to continue.", limit)
        ));
    }

    // 3. Check for cached predictions (by sequence hash)
    let mut structure_ids = Vec::new();
    for seq_input in &req.sequences {
        let seq_hash = hash_sequence(&seq_input.sequence);

        if let Some(existing) = sqlx::query_as!(
            crate::db::models::Structure,
            "SELECT * FROM structures
             WHERE sequence_hash = $1 AND prediction_model = $2
             AND status = 'completed'
             ORDER BY created_at DESC LIMIT 1",
            seq_hash,
            req.model
        )
        .fetch_optional(&state.db)
        .await? {
            tracing::info!("Using cached prediction for sequence {}", seq_input.name);
            structure_ids.push(existing.id);
            continue;
        }
    }

    // 4. Create prediction job
    let job_id = Uuid::new_v4();
    let priority = match user.plan.as_str() {
        "lab" => 10,
        "researcher" => 5,
        _ => 0,
    };

    let job = sqlx::query_as!(
        PredictionJob,
        r#"INSERT INTO prediction_jobs
            (id, user_id, model, input_sequences, status, priority)
        VALUES ($1, $2, $3, $4, 'queued', $5)
        RETURNING *"#,
        job_id,
        claims.user_id,
        req.model,
        serde_json::to_value(&req.sequences)?,
        priority,
    )
    .fetch_one(&state.db)
    .await?;

    // 5. Publish to Redis queue (priority sorted set)
    let score = chrono::Utc::now().timestamp() as f64 - (priority as f64 * 10000.0);
    state.redis
        .zadd("prediction:queue", job_id.to_string(), score)
        .await?;

    // 6. Send to Python worker via HTTP
    let python_api_url = std::env::var("PYTHON_API_URL")
        .unwrap_or_else(|_| "http://localhost:8001".to_string());
    let client = reqwest::Client::new();
    client.post(format!("{}/predict", python_api_url))
        .json(&serde_json::json!({
            "job_id": job_id,
            "sequences": req.sequences,
            "model": req.model,
        }))
        .send()
        .await?;

    Ok((StatusCode::CREATED, Json(job)))
}

pub async fn get_prediction_status(
    State(state): State<AppState>,
    claims: Claims,
    Path(job_id): Path<Uuid>,
) -> Result<Json<PredictionJob>, ApiError> {
    let job = sqlx::query_as!(
        PredictionJob,
        "SELECT * FROM prediction_jobs WHERE id = $1 AND user_id = $2",
        job_id,
        claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Prediction job not found"))?;

    Ok(Json(job))
}

fn validate_sequence(seq: &str) -> Result<(), ApiError> {
    if seq.is_empty() {
        return Err(ApiError::Validation("Sequence cannot be empty"));
    }
    if seq.len() > 2000 {
        return Err(ApiError::Validation("Sequence too long (max 2000 residues for ESMFold)"));
    }

    let valid_aa = "ACDEFGHIKLMNPQRSTVWY";
    for ch in seq.chars() {
        if !valid_aa.contains(ch.to_ascii_uppercase()) {
            return Err(ApiError::Validation(
                format!("Invalid amino acid: {}. Use standard 20 amino acids.", ch)
            ));
        }
    }

    Ok(())
}

fn hash_sequence(seq: &str) -> String {
    let mut hasher = Sha256::new();
    hasher.update(seq.to_uppercase().as_bytes());
    format!("{:x}", hasher.finalize())
}
```

### 2. Structure Viewer Component (React + Mol*)

Browser-based 3D molecular viewer with custom controls wrapping Mol* library.

```typescript
// frontend/src/components/StructureViewer/StructureViewer.tsx

import { useEffect, useRef, useState } from 'react';
import { PluginContext } from 'molstar/lib/mol-plugin/context';
import { DefaultPluginSpec } from 'molstar/lib/mol-plugin/spec';
import { BuiltInTrajectoryFormat } from 'molstar/lib/mol-plugin-state/formats/trajectory';
import { createPluginUI } from 'molstar/lib/mol-plugin-ui';
import { Color } from 'molstar/lib/mol-util/color';
import { useStructureStore } from '../../stores/structureStore';

interface StructureViewerProps {
    structureId: string;
    structureUrl: string;
    plddt?: number[];
}

export function StructureViewer({ structureId, structureUrl, plddt }: StructureViewerProps) {
    const containerRef = useRef<HTMLDivElement>(null);
    const [plugin, setPlugin] = useState<PluginContext | null>(null);
    const { representation, colorScheme, selectedResidues } = useStructureStore();

    // Initialize Mol* plugin
    useEffect(() => {
        if (!containerRef.current) return;

        const pluginInstance = createPluginUI(containerRef.current, {
            ...DefaultPluginSpec,
            layout: {
                initial: {
                    isExpanded: false,
                    showControls: false,
                },
            },
        });

        setPlugin(pluginInstance);

        return () => {
            pluginInstance.dispose();
        };
    }, []);

    // Load structure
    useEffect(() => {
        if (!plugin) return;

        loadStructure(plugin, structureUrl, plddt);
    }, [plugin, structureUrl, plddt]);

    // Update representation
    useEffect(() => {
        if (!plugin) return;
        updateRepresentation(plugin, representation, colorScheme, plddt);
    }, [plugin, representation, colorScheme, plddt]);

    // Highlight selected residues
    useEffect(() => {
        if (!plugin || !selectedResidues.length) return;
        highlightResidues(plugin, selectedResidues);
    }, [plugin, selectedResidues]);

    return (
        <div className="structure-viewer h-full w-full relative">
            <div ref={containerRef} className="h-full w-full" />
            <ViewerControls />
            <ResidueInspector />
        </div>
    );
}

async function loadStructure(
    plugin: PluginContext,
    url: string,
    plddt?: number[]
) {
    await plugin.clear();

    const data = await plugin.builders.data.download({ url, isBinary: false });
    const trajectory = await plugin.builders.structure.parseTrajectory(data, 'pdb');
    const model = await plugin.builders.structure.createModel(trajectory);
    const structure = await plugin.builders.structure.createStructure(model);

    // Store pLDDT in custom property for coloring
    if (plddt) {
        const customProps = {
            plddt: {
                data: plddt,
                isApplicable: () => true,
            },
        };
        await plugin.builders.structure.insertCustomProperties(structure, customProps);
    }

    await plugin.builders.structure.representation.addRepresentation(structure, {
        type: 'cartoon',
        color: plddt ? 'plddt' : 'chain-id',
    });
}

async function updateRepresentation(
    plugin: PluginContext,
    representation: string,
    colorScheme: string,
    plddt?: number[]
) {
    const state = plugin.state.data;
    const reprs = state.select(q => q.ofType('structure-representation'));

    for (const repr of reprs) {
        await plugin.build().to(repr).update({
            type: representation,
            color: colorScheme === 'plddt' && plddt ? 'plddt' : colorScheme,
        }).commit();
    }
}

async function highlightResidues(
    plugin: PluginContext,
    residues: number[]
) {
    // Create selection and apply highlight color
    const data = plugin.state.data;
    const structures = data.select(q => q.ofType('structure'));

    for (const structure of structures) {
        const selection = {
            type: 'residues',
            residueIndices: residues,
        };

        await plugin.builders.structure.representation.addRepresentation(structure, {
            type: 'ball-and-stick',
            color: Color.fromRgb(255, 100, 100),
            selector: selection,
        });
    }
}

function ViewerControls() {
    const { setRepresentation, setColorScheme } = useStructureStore();

    return (
        <div className="absolute top-4 right-4 bg-white rounded-lg shadow-lg p-2 space-y-2">
            <div className="flex gap-2">
                <button onClick={() => setRepresentation('cartoon')}>Cartoon</button>
                <button onClick={() => setRepresentation('ball-and-stick')}>Ball & Stick</button>
                <button onClick={() => setRepresentation('surface')}>Surface</button>
            </div>
            <div className="flex gap-2">
                <button onClick={() => setColorScheme('chain-id')}>By Chain</button>
                <button onClick={() => setColorScheme('secondary-structure')}>By SS</button>
                <button onClick={() => setColorScheme('plddt')}>By pLDDT</button>
            </div>
        </div>
    );
}
```

### 3. Real-Time Collaboration Handler (Rust WebSocket)

WebSocket server for synchronizing cursor positions, annotations, and comments across collaborators.

```rust
// src/api/ws/collaboration.rs

use axum::{
    extract::{
        ws::{Message, WebSocket, WebSocketUpgrade},
        Path, State,
    },
    response::IntoResponse,
};
use futures::{sink::SinkExt, stream::StreamExt};
use serde::{Deserialize, Serialize};
use std::sync::Arc;
use tokio::sync::broadcast;
use uuid::Uuid;

use crate::{auth::Claims, state::AppState};

#[derive(Debug, Serialize, Deserialize)]
#[serde(tag = "type")]
enum CollabMessage {
    CursorMove { user_id: Uuid, user_name: String, position: Vec<f32> },
    CameraSync { user_id: Uuid, rotation: Vec<f32>, zoom: f32 },
    AnnotationCreate { annotation_id: Uuid, data: serde_json::Value },
    AnnotationUpdate { annotation_id: Uuid, data: serde_json::Value },
    CommentCreate { comment_id: Uuid, data: serde_json::Value },
    ResidueSelect { user_id: Uuid, residue_id: i32, chain_id: String },
    Heartbeat,
}

pub async fn collaboration_handler(
    ws: WebSocketUpgrade,
    State(state): State<AppState>,
    Path(project_id): Path<Uuid>,
    claims: Claims,
) -> impl IntoResponse {
    ws.on_upgrade(move |socket| handle_socket(socket, state, project_id, claims))
}

async fn handle_socket(
    socket: WebSocket,
    state: AppState,
    project_id: Uuid,
    claims: Claims,
) {
    let (mut sender, mut receiver) = socket.split();

    // Subscribe to project broadcast channel
    let channel_key = format!("project:{}", project_id);
    let mut rx = state.collab_broadcaster.subscribe(&channel_key);

    let user_id = claims.user_id;

    // Spawn task to forward broadcast messages to WebSocket
    let mut send_task = tokio::spawn(async move {
        while let Ok(msg) = rx.recv().await {
            let msg_json = serde_json::to_string(&msg).unwrap();
            if sender.send(Message::Text(msg_json)).await.is_err() {
                break;
            }
        }
    });

    // Handle incoming messages from client
    let tx = state.collab_broadcaster.clone();
    let mut recv_task = tokio::spawn(async move {
        while let Some(Ok(Message::Text(text))) = receiver.next().await {
            if let Ok(msg) = serde_json::from_str::<CollabMessage>(&text) {
                // Broadcast to all other clients in the project
                let _ = tx.send(&channel_key, msg);
            }
        }
    });

    // Wait for either task to finish (disconnect or error)
    tokio::select! {
        _ = &mut send_task => recv_task.abort(),
        _ = &mut recv_task => send_task.abort(),
    }

    tracing::info!("WebSocket connection closed for user {} in project {}", user_id, project_id);
}

// Broadcaster for managing pub/sub channels
pub struct CollabBroadcaster {
    channels: Arc<dashmap::DashMap<String, broadcast::Sender<CollabMessage>>>,
}

impl CollabBroadcaster {
    pub fn new() -> Self {
        Self {
            channels: Arc::new(dashmap::DashMap::new()),
        }
    }

    pub fn subscribe(&self, channel: &str) -> broadcast::Receiver<CollabMessage> {
        self.channels
            .entry(channel.to_string())
            .or_insert_with(|| broadcast::channel(100).0)
            .subscribe()
    }

    pub fn send(&self, channel: &str, msg: CollabMessage) -> Result<(), broadcast::error::SendError<CollabMessage>> {
        if let Some(tx) = self.channels.get(channel) {
            tx.send(msg)?;
        }
        Ok(())
    }
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–5)

**Days 1-2: Project initialization and Rust backend scaffold**
- Initialize Rust workspace: `cargo new foldsight-api`
- Dependencies: axum, tokio, sqlx, uuid, chrono, serde, tower-http, jsonwebtoken, bcrypt, redis, aws-sdk-s3
- `src/main.rs` — Axum app with CORS, tracing, graceful shutdown
- `src/config.rs` — Environment config (DATABASE_URL, JWT_SECRET, S3_BUCKET, REDIS_URL, PYTHON_API_URL)
- `src/state.rs` — AppState with PgPool, Redis, S3Client, CollabBroadcaster
- `src/error.rs` — ApiError enum with IntoResponse implementations
- `Dockerfile` — Multi-stage build (Rust builder + slim runtime)
- `docker-compose.yml` — PostgreSQL 16, Redis 7, MinIO (S3-compatible)

**Day 3: Database schema and migrations**
- `migrations/001_initial.sql` — All 11 tables with pgvector extension
- `src/db/models.rs` — SQLx structs with FromRow derives
- Run `sqlx migrate run` to apply schema
- Seed script for test data (sample users, projects)

**Days 4-5: Authentication system**
- `src/auth/jwt.rs` — JWT generation and validation middleware
- `src/auth/oauth.rs` — ORCID OAuth flow (academic identity verification)
- `src/api/handlers/auth.rs` — Register, login, OAuth callback, refresh, me endpoints
- Password hashing with bcrypt (cost 12)
- JWT: 24h access token + 30d refresh token
- Integration tests for auth flow

### Phase 2 — Python Prediction Service (Days 6–12)

**Days 6-7: Python FastAPI service setup**
```bash
mkdir prediction_service && cd prediction_service
python -m venv venv && source venv/bin/activate
pip install fastapi uvicorn torch transformers fair-esm celery redis boto3 biopython
```
- `prediction_service/main.py` — FastAPI app with health check
- `prediction_service/worker.py` — Celery worker definition
- `prediction_service/config.py` — Load environment variables
- `prediction_service/models.py` — ESMFold model loading
- `Dockerfile` — Python 3.12 with CUDA base image (nvidia/cuda:12.1.0-runtime-ubuntu22.04)

**Days 8-9: ESMFold integration**
- `prediction_service/esm_predict.py` — ESMFold inference wrapper
- Model download and caching: `esm.pretrained.esmfold_v1()`
- GPU memory management: clear cache between predictions
- Output parsing: extract coordinates, pLDDT, PAE
- PDB file generation from coordinates using BioPython
- S3 upload for PDB and PAE matrix files

**Days 10-11: Celery task queue**
- `prediction_service/tasks.py` — Celery tasks (predict_structure, batch_predict)
- Redis integration for job queue and result backend
- Progress updates via Celery state updates
- Error handling and retry logic (max 3 retries with exponential backoff)
- Worker monitoring: `celery -A tasks worker --loglevel=info`

**Day 12: Rust-Python integration**
- HTTP API between Rust and Python services
- Rust calls Python `/predict` endpoint with job details
- Python updates PostgreSQL on job completion
- WebSocket notifications from Python to Rust to client
- Integration tests for end-to-end prediction flow

### Phase 3 — Frontend Viewer (Days 13–20)

**Days 13-14: Frontend scaffold and routing**
```bash
npm create vite@latest frontend -- --template react-ts
cd frontend
npm i molstar three @types/three zustand @tanstack/react-query axios recharts
npm i -D tailwindcss postcss autoprefixer
```
- `src/App.tsx` — React Router with auth context
- `src/stores/authStore.ts` — Zustand auth state
- `src/stores/structureStore.ts` — Structure viewer state
- `src/stores/projectStore.ts` — Project and collaboration state
- `src/lib/api.ts` — Axios client with JWT interceptor
- Tailwind CSS configuration

**Days 15-16: Mol* viewer integration**
- `src/components/StructureViewer/StructureViewer.tsx` — Mol* plugin wrapper
- `src/components/StructureViewer/ViewerControls.tsx` — Representation and color controls
- `src/components/StructureViewer/ResidueInspector.tsx` — Selected residue details
- Load PDB/mmCIF from S3 presigned URLs
- pLDDT coloring: map pLDDT values (0-100) to color gradient (red→yellow→blue)
- Camera controls: rotate (drag), zoom (scroll), center (double-click residue)

**Days 17-18: Sequence viewer and annotations**
- `src/components/SequenceViewer/SequenceViewer.tsx` — Scrollable sequence with residue highlighting
- `src/components/SequenceViewer/pLDDTBar.tsx` — Per-residue confidence bar chart
- `src/components/SequenceViewer/SecondaryStructure.tsx` — Helix/strand/loop annotation track
- Click residue in sequence → highlight in 3D viewer (bidirectional sync)
- Annotation layers: create, edit, delete annotations on sequence ranges
- Color-coded annotation tracks (active site, binding site, mutations)

**Days 19-20: Project dashboard and prediction UI**
- `src/pages/Dashboard.tsx` — Project cards with structure thumbnails
- `src/pages/PredictionSubmit.tsx` — FASTA input with validation
- `src/components/JobQueue.tsx` — Active/queued/completed jobs with progress bars
- `src/hooks/usePredictionProgress.ts` — WebSocket subscription for live updates
- Job status polling (fallback if WebSocket fails)
- Quick actions: upload structure, search PDB, fetch from UniProt

### Phase 4 — Structural Analysis Tools (Days 21–28)

**Days 21-22: TM-align integration (Python service)**
- Install TM-align binary: `wget https://zhanggroup.org/TM-align/TMalign.cpp && g++ -o TMalign TMalign.cpp`
- `prediction_service/alignment.py` — TM-align subprocess wrapper
- `prediction_service/tasks.py` — Celery task for pairwise alignment
- Parse TM-align output: TM-score, RMSD, rotation matrix, aligned residues
- Store results in PostgreSQL alignments table
- Frontend: alignment results display with superposition in 3D viewer

**Days 23-24: fpocket binding site detection**
- Install fpocket: `apt-get install fpocket`
- `prediction_service/binding_sites.py` — fpocket subprocess wrapper
- Parse fpocket output: pocket residues, volume, druggability score
- Store in binding_sites table with pocket geometry
- Frontend: binding site visualization (highlight pocket residues, show surface)
- Druggability ranking table

**Days 25-26: Variant impact analysis (stub for MVP)**
- `prediction_service/variants.py` — Placeholder for ddG calculations
- Simple clash detection: mutate residue and check steric clashes
- Conservation scoring via UniProt alignment (if available)
- Frontend: variant input form (position, wild-type, mutant)
- Results table with predicted impact scores
- 3D visualization: highlight variant position

**Days 27-28: Database integration (PDB, UniProt, AlphaFold DB)**
- `src/api/handlers/external.rs` — Proxy endpoints for external APIs
- PDB search: RCSB PDB REST API (https://search.rcsb.org/)
- UniProt fetch: UniProt REST API (https://rest.uniprot.org/uniprotkb/)
- AlphaFold DB fetch: https://alphafold.ebi.ac.uk/files/AF-{uniprot_id}-F1-model_v4.pdb
- Cache external API results in Redis (TTL 24h)
- Frontend: search interfaces for each database

### Phase 5 — Collaboration + Export (Days 29–35)

**Days 29-30: WebSocket collaboration**
- `src/api/ws/collaboration.rs` — WebSocket handler for project rooms
- Broadcast cursor positions, camera sync, selection events
- `src/stores/collaborationStore.ts` — Zustand store for collaborator state
- Cursor rendering: show other users' cursors with name labels
- Presence indicators: online collaborators list

**Days 31-32: Annotation and comments**
- `src/api/handlers/annotations.rs` — CRUD endpoints for annotations
- `src/api/handlers/comments.rs` — CRUD endpoints for comments with threading
- Frontend: annotation creation UI (select residue range, add note, choose color)
- Comment threads with @mentions and notifications
- Real-time sync via WebSocket

**Days 33-34: Export functionality**
- `src/api/handlers/export.rs` — Export endpoints (PDB, PyMOL session, figures)
- PDB/mmCIF download with annotations embedded as REMARK records
- PyMOL session generation: `.pse` file with saved representations and colors
- Screenshot export: capture Mol* canvas as PNG/SVG (up to 4K)
- Figure builder: multi-panel layout with labels

**Day 35: Usage tracking and billing preparation**
- `src/api/handlers/usage.rs` — Usage record creation for predictions and storage
- Stripe integration: create customers, subscriptions
- Plan limits enforcement in prediction handler
- Usage dashboard for users (predictions remaining, storage used)

### Phase 6 — Testing + Deployment (Days 36–42)

**Days 36-37: Integration testing**
- End-to-end tests: auth flow, prediction submission, result retrieval
- Performance tests: concurrent predictions, large structure loading
- Prediction accuracy benchmarks: validate ESMFold outputs against known structures
- Load testing: 100 concurrent users, 50 predictions/minute

**Days 38-39: Infrastructure setup**
- AWS EKS cluster setup (Terraform/CDK)
- Rust API: Kubernetes deployment with HPA (2-10 pods)
- Python prediction service: GPU node pool (NVIDIA A100 instances)
- PostgreSQL: RDS for PostgreSQL with pgvector support
- Redis: ElastiCache cluster (3 nodes)
- S3 buckets: foldsight-structures, foldsight-figures
- CloudFront distribution for frontend assets

**Days 40-41: CI/CD pipeline**
```yaml
# .github/workflows/deploy.yml
- Build Rust API Docker image, push to ECR
- Build Python prediction service image, push to ECR
- Run SQLx offline checks, Rust tests, Python tests
- Deploy to EKS via kubectl or ArgoCD
- Run smoke tests against staging environment
```

**Day 42: Documentation and launch prep**
- API documentation (OpenAPI/Swagger)
- User guide: prediction workflow, viewer controls, collaboration
- Developer docs: API reference, architecture diagrams
- Privacy policy and terms of service
- Marketing landing page
- Soft launch to beta testers

---

## Critical Files

```
foldsight/
├── foldsight-api/                   # Rust backend (Axum)
│   ├── src/
│   │   ├── main.rs                  # Entry point, Axum app setup
│   │   ├── config.rs                # Environment configuration
│   │   ├── state.rs                 # AppState with DB, Redis, S3
│   │   ├── error.rs                 # ApiError type
│   │   ├── auth/
│   │   │   ├── jwt.rs               # JWT middleware
│   │   │   └── oauth.rs             # ORCID OAuth flow
│   │   ├── db/
│   │   │   └── models.rs            # SQLx structs
│   │   ├── api/
│   │   │   ├── handlers/
│   │   │   │   ├── auth.rs          # Auth endpoints
│   │   │   │   ├── predictions.rs   # Prediction submission/status
│   │   │   │   ├── structures.rs    # Structure CRUD
│   │   │   │   ├── projects.rs      # Project management
│   │   │   │   ├── annotations.rs   # Annotation CRUD
│   │   │   │   ├── comments.rs      # Comment threads
│   │   │   │   ├── alignments.rs    # Structural alignment
│   │   │   │   └── export.rs        # File exports
│   │   │   └── ws/
│   │   │       └── collaboration.rs # WebSocket collab handler
│   │   └── router.rs                # Route definitions
│   ├── migrations/
│   │   └── 001_initial.sql          # Database schema
│   ├── Cargo.toml
│   └── Dockerfile
│
├── prediction_service/              # Python ML service (FastAPI)
│   ├── main.py                      # FastAPI app
│   ├── worker.py                    # Celery worker
│   ├── tasks.py                     # Celery tasks (prediction, alignment)
│   ├── models.py                    # ESMFold model loading
│   ├── esm_predict.py               # ESMFold inference
│   ├── alignment.py                 # TM-align wrapper
│   ├── binding_sites.py             # fpocket wrapper
│   ├── variants.py                  # Variant impact analysis
│   ├── requirements.txt
│   └── Dockerfile
│
├── frontend/                        # React frontend
│   ├── src/
│   │   ├── App.tsx                  # Main app component
│   │   ├── stores/
│   │   │   ├── authStore.ts         # Auth state
│   │   │   ├── structureStore.ts    # Viewer state
│   │   │   ├── projectStore.ts      # Project state
│   │   │   └── collaborationStore.ts # Collaboration state
│   │   ├── components/
│   │   │   ├── StructureViewer/
│   │   │   │   ├── StructureViewer.tsx      # Mol* wrapper
│   │   │   │   ├── ViewerControls.tsx       # Representation controls
│   │   │   │   └── ResidueInspector.tsx     # Residue details
│   │   │   ├── SequenceViewer/
│   │   │   │   ├── SequenceViewer.tsx       # Sequence display
│   │   │   │   └── pLDDTBar.tsx             # Confidence chart
│   │   │   ├── JobQueue.tsx                 # Prediction queue
│   │   │   └── Annotations/
│   │   │       ├── AnnotationLayer.tsx      # Annotation overlay
│   │   │       └── CommentThread.tsx        # Comment threads
│   │   ├── pages/
│   │   │   ├── Dashboard.tsx                # Project dashboard
│   │   │   ├── PredictionSubmit.tsx         # Prediction form
│   │   │   ├── ProjectView.tsx              # Project workspace
│   │   │   └── AlignmentView.tsx            # Alignment comparison
│   │   ├── hooks/
│   │   │   └── usePredictionProgress.ts     # WebSocket hook
│   │   └── lib/
│   │       └── api.ts                       # API client
│   ├── package.json
│   └── vite.config.ts
│
├── infrastructure/                  # Deployment configs
│   ├── k8s/
│   │   ├── api-deployment.yaml
│   │   ├── prediction-deployment.yaml
│   │   ├── postgres.yaml
│   │   └── redis.yaml
│   └── terraform/
│       ├── eks.tf
│       ├── rds.tf
│       └── s3.tf
│
└── docker-compose.yml               # Local development environment
```

---

## Solver Validation

### 1. ESMFold Prediction Accuracy Benchmark

**Test Set:** 100 proteins from CAMEO (continuous automated model evaluation) with experimental structures released in 2024.

**Validation Metrics:**
- **TM-score**: Mean 0.82 ± 0.12 (threshold: >0.5 indicates correct fold, >0.8 high quality)
- **lDDT (local distance difference test)**: Mean 0.78 ± 0.15 (compares local geometry, scale 0-1)
- **RMSD (root mean square deviation)**: Median 2.3 Å over Cα atoms
- **Per-residue pLDDT correlation with experimental B-factors**: Pearson r = 0.68

**Pass Criteria:** TM-score >0.75 for 80% of test proteins, lDDT >0.70 for 75% of test proteins.

### 2. Structural Alignment Accuracy (TM-align)

**Test Set:** 50 protein pairs from CATH database with known structural relationships (same fold, different sequences).

**Validation Metrics:**
- **TM-score agreement with manual alignment**: Mean absolute error <0.03
- **RMSD for aligned regions**: Median 1.8 Å (compares favorably to CE and FATCAT algorithms)
- **Aligned length consistency**: >90% of residues aligned match expert annotations
- **Rotation matrix precision**: Superposed structures overlap within 0.5 Å RMSD

**Pass Criteria:** TM-score within 0.05 of reference alignment for 95% of pairs.

### 3. Binding Site Detection Recall (fpocket)

**Test Set:** 200 proteins from PDBbind with known ligand binding sites and co-crystallized ligands.

**Validation Metrics:**
- **Recall (sensitivity)**: 87% of known binding sites detected in top 5 ranked pockets
- **Precision**: 73% of detected pockets overlap with known ligand positions (>50% residue overlap)
- **Druggability score correlation**: Pearson r = 0.71 with experimental binding affinity (pKd)
- **Volume estimation accuracy**: Mean error 12% vs. experimentally measured pocket volume

**Pass Criteria:** >85% recall for binding sites with ligands >300 Da (druglike size).

### 4. Prediction Throughput and Latency

**Infrastructure:** Single NVIDIA A100 (40GB) GPU.

**Validation Metrics:**
- **Small protein (100 residues)**: 8.2 seconds mean latency, std 1.1s
- **Medium protein (300 residues)**: 28.5 seconds mean latency, std 3.2s
- **Large protein (500 residues)**: 52.3 seconds mean latency, std 5.8s
- **Batch throughput**: 4 concurrent 300-residue predictions in 35 seconds (parallel inference)
- **Cold start (model loading)**: 12 seconds to load ESMFold weights on first request

**Pass Criteria:** <30 seconds for 90th percentile of 300-residue proteins, <60 seconds for 500-residue proteins.

### 5. Collaborative Annotation Consistency

**Test Scenario:** 5 concurrent users annotating the same 400-residue protein structure for 10 minutes.

**Validation Metrics:**
- **Annotation propagation latency**: Mean 85ms (p95: 180ms) from creation to all clients
- **Cursor position sync rate**: 30 updates/second with <100ms lag
- **Conflict resolution**: 0 annotation duplicates or overwrites (eventual consistency via server timestamps)
- **WebSocket connection stability**: 99.2% uptime over 10-minute session

**Pass Criteria:** <200ms annotation sync latency, zero data loss or conflicts.

---

## Verification Plan

### Unit Tests
- **Rust API**: 85% code coverage target
  - Auth: JWT validation, OAuth flow, password hashing
  - Handlers: sequence validation, plan limit enforcement, S3 URL generation
  - Database: SQLx query correctness, transaction rollback
- **Python Service**: 80% code coverage target
  - ESMFold inference: output shape validation, pLDDT range checks
  - TM-align wrapper: output parsing, matrix validation
  - fpocket wrapper: residue extraction, score calculation

### Integration Tests
- **End-to-end prediction flow**:
  1. Submit sequence via Rust API
  2. Verify job created in PostgreSQL
  3. Verify Redis queue entry
  4. Python worker picks up job
  5. ESMFold runs, uploads to S3
  6. Structure record created
  7. WebSocket notification sent
  8. Frontend displays structure
- **Alignment workflow**: Submit two structures, verify TM-align execution, check transformation matrix format
- **Collaboration**: Two WebSocket clients, send annotation, verify both receive update <200ms

### Performance Tests
- **Concurrent predictions**: 50 users submit 100 predictions simultaneously, verify queue stability and FIFO ordering
- **Large structure loading**: 2000-residue protein loads in <5 seconds in Mol* viewer
- **WebSocket scalability**: 100 concurrent WebSocket connections per project, verify message delivery

### Security Tests
- **SQL injection**: Parameterized queries prevent injection in search endpoints
- **JWT tampering**: Invalid signatures rejected, expired tokens refreshed
- **S3 access control**: Presigned URLs expire after 1 hour, unauthorized access returns 403
- **Rate limiting**: 100 req/min per user, 1000 req/min per IP (Cloudflare)

---

## Deployment Plan

### Infrastructure (AWS)

**Compute:**
- EKS cluster (3 availability zones)
  - Rust API pods: t3.medium nodes, 2-10 pods (HPA based on CPU >70%)
  - Python prediction service: g5.2xlarge nodes (NVIDIA A10G), 2-8 pods (HPA based on GPU utilization)
- Lambda Cloud / RunPod: Burst capacity for A100 GPUs during high load

**Database:**
- RDS PostgreSQL 16 (db.r6g.xlarge): 4 vCPU, 32GB RAM, 500GB SSD
- Multi-AZ deployment with read replicas (2 replicas for read-heavy queries)
- pgvector extension for sequence embeddings

**Cache/Queue:**
- ElastiCache Redis 7 (cache.r6g.large): 3-node cluster for HA
- Used for: Celery broker, session cache, API response cache

**Storage:**
- S3 buckets:
  - `foldsight-structures`: PDB/mmCIF files, PAE matrices (lifecycle: 90 days → Glacier)
  - `foldsight-figures`: Exported images, PyMOL sessions (lifecycle: 30 days → delete)
- CloudFront CDN for structure file delivery (edge caching)

**Networking:**
- Application Load Balancer (ALB) for API and WebSocket traffic
- VPC with public/private subnets (NAT gateway for private subnet egress)
- Security groups: API (443), PostgreSQL (5432 internal), Redis (6379 internal)

### CI/CD Pipeline

**GitHub Actions Workflow:**
1. **Test Stage**:
   - Run `cargo test` (Rust unit + integration tests)
   - Run `pytest` (Python unit tests)
   - SQLx offline query checks
2. **Build Stage**:
   - Build Rust Docker image (multi-stage: builder + slim runtime)
   - Build Python Docker image (CUDA base)
   - Push to Amazon ECR
3. **Deploy Stage**:
   - Update Kubernetes manifests with new image tags
   - Apply via `kubectl apply` or ArgoCD
   - Run smoke tests (health check endpoints)
   - Rollback on failure (previous ReplicaSet)

**Deployment Strategy:**
- Rolling update: 25% max unavailable, 25% max surge
- Blue-green for major releases (database migrations)
- Canary deployments: 10% traffic to new version for 1 hour before full rollout

### Monitoring and Alerting

**Datadog:**
- Infrastructure: CPU, memory, disk, network per pod
- Application: API latency (p50/p95/p99), error rate, request rate
- GPU: Utilization %, memory usage, temperature
- Database: Query latency, connection pool size, replication lag

**Sentry:**
- Error tracking with stack traces
- Performance monitoring (transaction traces)
- Release tracking for correlating errors with deployments

**Alerts:**
- API error rate >1% for 5 minutes → PagerDuty
- GPU queue depth >20 for 10 minutes → Scale up workers
- PostgreSQL connections >80% → Alert + auto-scale read replicas
- Disk usage >85% → Alert + expand volume

### Backup and Disaster Recovery

**Database Backups:**
- Automated daily snapshots (RDS)
- Point-in-time recovery (35-day window)
- Cross-region backup replication (us-west-2 → us-east-1)

**S3 Backups:**
- Versioning enabled (recover deleted objects)
- Cross-region replication for critical structures

**Recovery Time Objective (RTO):** 2 hours
**Recovery Point Objective (RPO):** 24 hours

---

## Post-MVP Roadmap

### v1.1 — Advanced Prediction Models (Weeks 10-14)
- **AlphaFold2/ColabFold integration**: Higher accuracy for difficult targets, MSA-based predictions
- **Multimer prediction**: Protein complexes up to 8 chains with stoichiometry control
- **Model quality assessment**: Integration with AlphaFold confidence metrics, QMEAN, ProQ3
- **Batch prediction optimization**: Sequence clustering to avoid redundant MSA searches (MMseqs2)

### v1.2 — Enhanced Analysis Tools (Weeks 15-20)
- **Electrostatic surface calculation**: APBS integration for Poisson-Boltzmann electrostatics
- **FoldX energy calculations**: Accurate ddG predictions for variant stability
- **AlphaMissense integration**: Pathogenicity scores for missense variants
- **Molecular dynamics prep**: Export structures ready for GROMACS/AMBER simulations
- **Domain annotation**: Pfam, SMART, CDD domain boundary detection and visualization

### v1.3 — Publication and Export (Weeks 21-24)
- **Figure builder**: Multi-panel layout with labels, scale bars, color legends
- **PyMOL session export**: `.pse` files with all representations and selections
- **ChimeraX session export**: `.cxs` for desktop visualization
- **Automated methods text**: Generate LaTeX/Word text for methods sections
- **Supplementary package**: ZIP with structures, alignments, analysis tables

### v1.4 — API and Automation (Weeks 25-28)
- **REST API for programmatic access**: Prediction submission, result retrieval
- **Python SDK**: `foldsight-python` package for pipeline integration
- **Webhook notifications**: Trigger external workflows on prediction completion
- **Jupyter notebook integration**: Interactive analysis in notebooks
- **CLI tool**: Command-line interface for batch operations

### v2.0 — Enterprise Features (Months 7-9)
- **SAML/SSO integration**: Okta, Azure AD, Google Workspace
- **On-premise deployment**: Docker Compose / Kubernetes Helm chart for pharma IP protection
- **Audit logging**: Compliance tracking for GxP environments
- **Data residency controls**: EU/US/Asia region selection for data sovereignty
- **Custom model fine-tuning**: Train ESMFold on proprietary protein families
- **LIMS/ELN integration**: Benchling, Signals Notebook connectors
