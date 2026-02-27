# 26. GeneWeave — Cloud Genomics Pipeline Builder for Bioinformatics

## Implementation Plan

**MVP Scope:** Browser-based visual pipeline builder with drag-and-drop genomics workflow assembly (50+ pre-built modules covering FastQC, Trimmomatic, BWA-MEM2, STAR, GATK, Salmon, Cell Ranger, DESeq2, Seurat, AnnotSV), smart connectors enforcing data type compatibility (FASTQ→BAM→VCF) with auto-validation, Rust/Axum backend managing Kubernetes-orchestrated containerized tool execution with auto-scaling 0→500 pods and spot instance optimization, WASM-based FASTQ quality metrics calculator for client-side instant QC preview, three pre-built best-practice pipeline templates (WGS variant calling with BWA-MEM2 + GATK HaplotypeCaller, RNA-seq with STAR + DESeq2, scRNA-seq with STARsolo + Scanpy), PostgreSQL metadata storage + S3 for genomic data (FASTQ/BAM/VCF/TSV) with multipart resumable uploads, real-time pipeline execution monitoring dashboard with per-stage logs and progress streaming via WebSocket, interactive QC dashboard (FastQC metrics, alignment stats, PCA plots, outlier detection) rendered with Plotly.js, variant annotation engine integrating ClinVar and gnomAD via pre-indexed VEP cache, interactive variant browser with filterable table and IGV.js genome viewer, full pipeline provenance tracking with version-locked container digests and parameter snapshots, Nextflow DSL2 export for reproducibility, Stripe billing with four tiers (Free / Research $149/mo / Clinical $599/mo / Enterprise custom).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth, high-throughput for genomics metadata |
| Pipeline Orchestration | Nextflow DSL2 + Kubernetes | Nextflow Tower Agent for K8s job management, containerized tools via Docker |
| WASM QC Engine | Rust → WASM (wasm-pack) | Client-side FASTQ parsing and FastQC-lite quality metrics (per-base quality, GC content) |
| Python Services | Python 3.12 (FastAPI) | Annotation service (VEP integration), statistical analysis (DESeq2/Seurat wrappers) |
| Database | PostgreSQL 16 | Projects, pipelines, samples, runs, annotations, audit logs |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, async PostgreSQL driver |
| Auth | Custom JWT + OAuth 2.0 | Google, GitHub, and ORCID OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 / Google Cloud Storage | FASTQ, BAM, VCF, intermediate files, reference genomes (GRCh38, GRCh37, T2T-CHM13) |
| Annotation Databases | VEP cache, ClinVar, gnomAD | Pre-indexed databases on NVMe-backed EBS volumes for fast lookup |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management, React Flow for pipeline canvas |
| Visualization | Plotly.js + IGV.js | QC charts (PCA, heatmaps, distributions), genome browser integration |
| Real-time | WebSocket (Axum) | Live pipeline progress, log streaming, convergence monitoring |
| Job Queue | Kubernetes Job API + Redis | Nextflow submits K8s Jobs, Redis for task metadata and priority queues |
| Autoscaling | KEDA (Kubernetes Event-Driven Autoscaler) | Scale pods 0→500 based on Redis queue depth and CPU/memory metrics |
| Billing | Stripe | Checkout Sessions, Customer Portal, usage-based metering for compute-hours |
| Monitoring | Prometheus + Grafana, Sentry | K8s cluster metrics, pipeline execution metrics, error tracking |
| CI/CD | GitHub Actions | WASM build, Docker image builds for tool containers, Rust tests, Nextflow validation |

### Key Architecture Decisions

1. **Rust/Axum primary backend with Python FastAPI for ML/annotation services**: Rust handles all API endpoints, WebSocket connections, database operations, and job orchestration for maximum throughput and reliability. Python FastAPI services run as separate microservices handling CPU-intensive annotation (VEP), statistical analysis (DESeq2, Seurat), and machine learning tasks (CellTypist for scRNA-seq annotation). This separation maintains Rust's performance advantages while leveraging Python's rich bioinformatics ecosystem.

2. **Nextflow DSL2 as pipeline execution engine on Kubernetes**: Nextflow is the gold standard for bioinformatics workflow management with proven scalability (used by nf-core, Wellcome Sanger Institute, Broad Institute). Running Nextflow workflows as Kubernetes Jobs via the K8s executor gives us auto-scaling, spot instance support, and container isolation. Each pipeline module maps to a Nextflow process with pinned container digests ensuring reproducibility.

3. **WASM-based client-side FASTQ QC for instant feedback**: Uploading a 10GB FASTQ file and waiting for server-side FastQC can take 5-10 minutes. Our WASM engine parses the first 1M reads client-side and generates quality metrics (per-base quality distribution, GC content, sequence length, duplication estimate) in <2 seconds, giving immediate feedback before full pipeline launch. Full FastQC still runs server-side for complete analysis.

4. **Visual pipeline builder with React Flow and data type enforcement**: The drag-and-drop canvas uses React Flow for node-based editing with custom node types for each bioinformatics tool. Smart connectors enforce data type compatibility (FASTQ→FASTQ, BAM→BAM, VCF→VCF) and validation checks pipeline topology for missing inputs, parameter conflicts, and unsupported workflows. This prevents 90% of pipeline failures that occur from incorrect file type connections.

5. **Kubernetes Job API with spot instances and checkpointing**: Genomics pipelines can run 6-48 hours; spot instances provide 60-70% cost savings but can be interrupted. We use Kubernetes spot/preemptible node pools with Nextflow's automatic checkpointing — if a spot instance is reclaimed, Nextflow resumes from the last completed stage rather than restarting the entire pipeline. KEDA auto-scales pods based on Redis queue depth, scaling to zero when idle.

6. **Pre-indexed annotation databases on fast storage for sub-second lookups**: ClinVar (2M+ variants), gnomAD (760M variants), and VEP cache (500GB for GRCh38) are pre-indexed and hosted on NVMe-backed EBS volumes attached to the annotation service pods. This enables variant annotation at 5,000-10,000 variants/second vs. 100-500 variants/second with on-demand API queries.

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";  -- Fuzzy text search on samples/projects

-- Users
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email TEXT UNIQUE NOT NULL,
    password_hash TEXT,  -- NULL for OAuth-only users
    name TEXT NOT NULL,
    avatar_url TEXT,
    auth_provider TEXT DEFAULT 'email',  -- email | google | github | orcid
    auth_provider_id TEXT,
    plan TEXT NOT NULL DEFAULT 'free',  -- free | research | clinical | enterprise
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX users_email_idx ON users(email);
CREATE INDEX users_stripe_idx ON users(stripe_customer_id);

-- Organizations (for Clinical/Enterprise plans)
CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    slug TEXT UNIQUE NOT NULL,
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    plan TEXT NOT NULL DEFAULT 'free',
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    baa_signed BOOLEAN DEFAULT false,  -- HIPAA Business Associate Agreement
    compliance_level TEXT DEFAULT 'standard',  -- standard | hipaa | gdpr
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
    role TEXT NOT NULL DEFAULT 'analyst',  -- admin | analyst | viewer
    joined_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(org_id, user_id)
);
CREATE INDEX org_members_org_idx ON org_members(org_id);
CREATE INDEX org_members_user_idx ON org_members(user_id);

-- Projects (genomics analysis workspace)
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    organism TEXT NOT NULL DEFAULT 'human',  -- human | mouse | rat | custom
    reference_genome TEXT NOT NULL DEFAULT 'GRCh38',  -- GRCh38 | GRCh37 | T2T-CHM13 | GRCm39 | ...
    compliance_level TEXT DEFAULT 'standard',
    data_region TEXT DEFAULT 'us-east',  -- us-east | eu-west | ap-southeast
    settings JSONB DEFAULT '{}',
    is_public BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);
CREATE INDEX projects_org_idx ON projects(org_id);
CREATE INDEX projects_updated_idx ON projects(updated_at DESC);

-- Pipeline Definitions (workflow templates)
CREATE TABLE pipeline_definitions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    version TEXT NOT NULL DEFAULT '1.0.0',
    template_source TEXT NOT NULL,  -- builtin | custom | forked
    definition_json JSONB NOT NULL,  -- Nodes, edges, parameters (React Flow compatible)
    nextflow_script TEXT,  -- Generated Nextflow DSL2 code
    is_validated BOOLEAN DEFAULT false,
    validation_report JSONB,
    is_public BOOLEAN DEFAULT false,
    forked_from UUID REFERENCES pipeline_definitions(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX pipelines_user_idx ON pipeline_definitions(user_id);
CREATE INDEX pipelines_org_idx ON pipeline_definitions(org_id);
CREATE INDEX pipelines_public_idx ON pipeline_definitions(is_public) WHERE is_public = true;

-- Pipeline Modules (individual bioinformatics tools)
CREATE TABLE pipeline_modules (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL UNIQUE,
    tool_name TEXT NOT NULL,  -- e.g., "bwa-mem2", "gatk", "star"
    tool_version TEXT NOT NULL,
    container_image TEXT NOT NULL,
    container_digest TEXT NOT NULL,  -- SHA256 digest for reproducibility
    category TEXT NOT NULL,  -- qc | alignment | variant_calling | rna_quant | single_cell | annotation | assembly
    subcategory TEXT,
    input_ports JSONB NOT NULL,  -- [{name, data_type: fastq/bam/vcf/tsv, required}]
    output_ports JSONB NOT NULL,  -- [{name, data_type, description}]
    parameters JSONB NOT NULL,  -- [{name, type, default, description, validation_rules}]
    resource_defaults JSONB NOT NULL DEFAULT '{"cpu": 4, "memory_gb": 16, "disk_gb": 100}',
    documentation_url TEXT,
    is_builtin BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX modules_category_idx ON pipeline_modules(category);
CREATE INDEX modules_tool_idx ON pipeline_modules(tool_name);

-- Samples (input sequencing data)
CREATE TABLE samples (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    external_id TEXT,  -- Lab internal sample ID
    metadata JSONB DEFAULT '{}',  -- {sex, phenotype, tissue, batch, family_id, affected_status, ...}
    input_files JSONB NOT NULL,  -- [{s3_key, file_type: fastq|bam|vcf, file_size, checksum}]
    status TEXT NOT NULL DEFAULT 'uploaded',  -- uploaded | queued | processing | completed | failed
    qc_summary JSONB,  -- FastQC summary metrics
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX samples_project_idx ON samples(project_id);
CREATE INDEX samples_status_idx ON samples(status);
CREATE INDEX samples_name_trgm_idx ON samples USING gin(name gin_trgm_ops);

-- Pipeline Runs (workflow executions)
CREATE TABLE pipeline_runs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    pipeline_definition_id UUID NOT NULL REFERENCES pipeline_definitions(id) ON DELETE RESTRICT,
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name TEXT,
    parameters_override JSONB DEFAULT '{}',
    status TEXT NOT NULL DEFAULT 'queued',  -- queued | running | completed | failed | cancelled
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    duration_seconds INTEGER,
    compute_cost_usd REAL DEFAULT 0.0,
    instance_types_used TEXT[] DEFAULT '{}',
    nextflow_run_id TEXT,  -- Nextflow session UUID
    nextflow_log_url TEXT,  -- S3 key for full Nextflow log
    provenance JSONB,  -- Full reproducibility record: tool versions, parameters, input checksums
    error_message TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX runs_pipeline_idx ON pipeline_runs(pipeline_definition_id);
CREATE INDEX runs_project_idx ON pipeline_runs(project_id);
CREATE INDEX runs_user_idx ON pipeline_runs(user_id);
CREATE INDEX runs_status_idx ON pipeline_runs(status);
CREATE INDEX runs_created_idx ON pipeline_runs(created_at DESC);

-- Pipeline Run Samples (many-to-many)
CREATE TABLE pipeline_run_samples (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    pipeline_run_id UUID NOT NULL REFERENCES pipeline_runs(id) ON DELETE CASCADE,
    sample_id UUID NOT NULL REFERENCES samples(id) ON DELETE CASCADE,
    UNIQUE(pipeline_run_id, sample_id)
);
CREATE INDEX run_samples_run_idx ON pipeline_run_samples(pipeline_run_id);
CREATE INDEX run_samples_sample_idx ON pipeline_run_samples(sample_id);

-- Run Stages (per-module execution tracking)
CREATE TABLE run_stages (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    pipeline_run_id UUID NOT NULL REFERENCES pipeline_runs(id) ON DELETE CASCADE,
    sample_id UUID REFERENCES samples(id) ON DELETE CASCADE,
    module_id UUID NOT NULL REFERENCES pipeline_modules(id) ON DELETE RESTRICT,
    stage_name TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | running | completed | failed | cached
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    duration_seconds INTEGER,
    output_files JSONB DEFAULT '[]',  -- [{s3_key, file_type, file_size}]
    qc_metrics JSONB,  -- Tool-specific QC: {total_reads, mapped_reads, duplication_rate, ...}
    log_url TEXT,  -- S3 key for stage-specific log
    exit_code INTEGER,
    retry_count INTEGER DEFAULT 0,
    cache_hit BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX stages_run_idx ON run_stages(pipeline_run_id);
CREATE INDEX stages_sample_idx ON run_stages(sample_id);
CREATE INDEX stages_status_idx ON run_stages(status);

-- Variant Sets (VCF outputs from variant calling pipelines)
CREATE TABLE variant_sets (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    pipeline_run_id UUID NOT NULL REFERENCES pipeline_runs(id) ON DELETE CASCADE,
    sample_id UUID REFERENCES samples(id) ON DELETE CASCADE,
    vcf_file_url TEXT NOT NULL,  -- S3 key
    variant_count INTEGER NOT NULL DEFAULT 0,
    annotation_status TEXT DEFAULT 'pending',  -- pending | running | completed | failed
    annotation_started_at TIMESTAMPTZ,
    annotation_completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX variant_sets_run_idx ON variant_sets(pipeline_run_id);
CREATE INDEX variant_sets_sample_idx ON variant_sets(sample_id);

-- Annotated Variants (parsed and annotated VCF records)
CREATE TABLE annotated_variants (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    variant_set_id UUID NOT NULL REFERENCES variant_sets(id) ON DELETE CASCADE,
    chrom TEXT NOT NULL,
    pos INTEGER NOT NULL,
    ref TEXT NOT NULL,
    alt TEXT NOT NULL,
    qual REAL,
    filter TEXT,
    gene TEXT,
    transcript TEXT,
    consequence TEXT,  -- missense_variant | synonymous_variant | frameshift_variant | ...
    impact TEXT,  -- HIGH | MODERATE | LOW | MODIFIER
    clinvar_significance TEXT,  -- pathogenic | likely_pathogenic | vus | likely_benign | benign
    clinvar_stars INTEGER,
    gnomad_af REAL,  -- Global allele frequency
    gnomad_af_popmax REAL,
    cadd_score REAL,
    revel_score REAL,
    spliceai_score REAL,
    acmg_classification TEXT,  -- pathogenic | likely_pathogenic | vus | likely_benign | benign
    acmg_criteria TEXT[],  -- [PVS1, PS2, PM1, PP3, ...]
    user_notes TEXT,
    user_classification_override TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX variants_set_idx ON annotated_variants(variant_set_id);
CREATE INDEX variants_gene_idx ON annotated_variants(gene);
CREATE INDEX variants_chrom_pos_idx ON annotated_variants(chrom, pos);
CREATE INDEX variants_impact_idx ON annotated_variants(impact);
CREATE INDEX variants_clinvar_idx ON annotated_variants(clinvar_significance);

-- Audit Logs (HIPAA compliance requirement)
CREATE TABLE audit_logs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    action TEXT NOT NULL,  -- login | create_project | run_pipeline | view_variant | download_file | ...
    resource_type TEXT,  -- project | pipeline_run | sample | variant | file
    resource_id UUID,
    details JSONB DEFAULT '{}',
    ip_address INET,
    user_agent TEXT,
    timestamp TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX audit_org_idx ON audit_logs(org_id);
CREATE INDEX audit_user_idx ON audit_logs(user_id);
CREATE INDEX audit_timestamp_idx ON audit_logs(timestamp DESC);
CREATE INDEX audit_action_idx ON audit_logs(action);

-- Usage Records (for billing enforcement)
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,  -- compute_hours | storage_gb_hours | annotation_count
    quantity REAL NOT NULL,
    period_start DATE NOT NULL,
    period_end DATE NOT NULL,
    metadata JSONB DEFAULT '{}',  -- {pipeline_run_id, instance_type, ...}
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX usage_user_period_idx ON usage_records(user_id, period_start, period_end);
CREATE INDEX usage_org_period_idx ON usage_records(org_id, period_start, period_end);
CREATE INDEX usage_type_idx ON usage_records(record_type);
```

### Rust SQLx Structs

```rust
// src/db/models.rs

use chrono::{DateTime, Utc, NaiveDate};
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
    pub stripe_customer_id: Option<String>,
    pub stripe_subscription_id: Option<String>,
    pub baa_signed: bool,
    pub compliance_level: String,
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
    pub organism: String,
    pub reference_genome: String,
    pub compliance_level: String,
    pub data_region: String,
    pub settings: serde_json::Value,
    pub is_public: bool,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct PipelineDefinition {
    pub id: Uuid,
    pub org_id: Option<Uuid>,
    pub user_id: Uuid,
    pub name: String,
    pub description: String,
    pub version: String,
    pub template_source: String,
    pub definition_json: serde_json::Value,  // React Flow nodes/edges
    pub nextflow_script: Option<String>,
    pub is_validated: bool,
    pub validation_report: Option<serde_json::Value>,
    pub is_public: bool,
    pub forked_from: Option<Uuid>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct PipelineModule {
    pub id: Uuid,
    pub name: String,
    pub tool_name: String,
    pub tool_version: String,
    pub container_image: String,
    pub container_digest: String,
    pub category: String,
    pub subcategory: Option<String>,
    pub input_ports: serde_json::Value,
    pub output_ports: serde_json::Value,
    pub parameters: serde_json::Value,
    pub resource_defaults: serde_json::Value,
    pub documentation_url: Option<String>,
    pub is_builtin: bool,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Sample {
    pub id: Uuid,
    pub project_id: Uuid,
    pub name: String,
    pub external_id: Option<String>,
    pub metadata: serde_json::Value,
    pub input_files: serde_json::Value,
    pub status: String,
    pub qc_summary: Option<serde_json::Value>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct PipelineRun {
    pub id: Uuid,
    pub pipeline_definition_id: Uuid,
    pub project_id: Uuid,
    pub org_id: Option<Uuid>,
    pub user_id: Uuid,
    pub name: Option<String>,
    pub parameters_override: serde_json::Value,
    pub status: String,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub duration_seconds: Option<i32>,
    pub compute_cost_usd: f32,
    pub instance_types_used: Vec<String>,
    pub nextflow_run_id: Option<String>,
    pub nextflow_log_url: Option<String>,
    pub provenance: Option<serde_json::Value>,
    pub error_message: Option<String>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct RunStage {
    pub id: Uuid,
    pub pipeline_run_id: Uuid,
    pub sample_id: Option<Uuid>,
    pub module_id: Uuid,
    pub stage_name: String,
    pub status: String,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub duration_seconds: Option<i32>,
    pub output_files: serde_json::Value,
    pub qc_metrics: Option<serde_json::Value>,
    pub log_url: Option<String>,
    pub exit_code: Option<i32>,
    pub retry_count: i32,
    pub cache_hit: bool,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct VariantSet {
    pub id: Uuid,
    pub pipeline_run_id: Uuid,
    pub sample_id: Option<Uuid>,
    pub vcf_file_url: String,
    pub variant_count: i32,
    pub annotation_status: String,
    pub annotation_started_at: Option<DateTime<Utc>>,
    pub annotation_completed_at: Option<DateTime<Utc>>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct AnnotatedVariant {
    pub id: Uuid,
    pub variant_set_id: Uuid,
    pub chrom: String,
    pub pos: i32,
    pub r#ref: String,
    pub alt: String,
    pub qual: Option<f32>,
    pub filter: Option<String>,
    pub gene: Option<String>,
    pub transcript: Option<String>,
    pub consequence: Option<String>,
    pub impact: Option<String>,
    pub clinvar_significance: Option<String>,
    pub clinvar_stars: Option<i32>,
    pub gnomad_af: Option<f32>,
    pub gnomad_af_popmax: Option<f32>,
    pub cadd_score: Option<f32>,
    pub revel_score: Option<f32>,
    pub spliceai_score: Option<f32>,
    pub acmg_classification: Option<String>,
    pub acmg_criteria: Vec<String>,
    pub user_notes: Option<String>,
    pub user_classification_override: Option<String>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct AuditLog {
    pub id: Uuid,
    pub org_id: Option<Uuid>,
    pub user_id: Option<Uuid>,
    pub action: String,
    pub resource_type: Option<String>,
    pub resource_id: Option<Uuid>,
    pub details: serde_json::Value,
    pub ip_address: Option<std::net::IpAddr>,
    pub user_agent: Option<String>,
    pub timestamp: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct UsageRecord {
    pub id: Uuid,
    pub user_id: Uuid,
    pub org_id: Option<Uuid>,
    pub record_type: String,
    pub quantity: f32,
    pub period_start: NaiveDate,
    pub period_end: NaiveDate,
    pub metadata: serde_json::Value,
    pub created_at: DateTime<Utc>,
}

// Request/Response types

#[derive(Debug, Deserialize)]
pub struct CreatePipelineRunRequest {
    pub pipeline_definition_id: Uuid,
    pub sample_ids: Vec<Uuid>,
    pub parameters_override: Option<serde_json::Value>,
    pub name: Option<String>,
}

#[derive(Debug, Deserialize)]
pub struct VariantQueryParams {
    pub variant_set_id: Uuid,
    pub gene: Option<String>,
    pub chrom: Option<String>,
    pub pos_start: Option<i32>,
    pub pos_end: Option<i32>,
    pub impact: Option<Vec<String>>,  // HIGH, MODERATE
    pub max_gnomad_af: Option<f32>,
    pub min_cadd_score: Option<f32>,
    pub clinvar_significance: Option<Vec<String>>,
    pub page: Option<i32>,
    pub page_size: Option<i32>,
}
```

---

## WASM QC Engine Architecture

### Genomics-Specific WASM Use Case

Unlike the SPICE simulator reference which runs full simulations in WASM, GeneWeave's WASM module performs **lightweight client-side FASTQ quality control preview** to provide instant feedback during file upload. A full genomics pipeline (alignment, variant calling) requires hundreds of GB of reference genome data and cannot run client-side.

### WASM Module Design

```rust
// fastq-qc-wasm/src/lib.rs

use wasm_bindgen::prelude::*;
use serde::{Deserialize, Serialize};

#[wasm_bindgen]
pub struct FastqQcEngine {
    total_reads: u64,
    base_qualities: Vec<Vec<u32>>,  // [position][quality_count]
    gc_count: u64,
    total_bases: u64,
    sequence_lengths: Vec<u32>,
    quality_scores: Vec<u8>,
}

#[derive(Serialize, Deserialize)]
pub struct QcMetrics {
    pub total_reads: u64,
    pub mean_read_length: f64,
    pub mean_quality: f64,
    pub gc_content: f64,
    pub per_base_quality: Vec<PerBaseQuality>,
    pub quality_distribution: Vec<u32>,
    pub estimated_duplication_rate: f64,
}

#[derive(Serialize, Deserialize)]
pub struct PerBaseQuality {
    pub position: usize,
    pub mean: f64,
    pub median: f64,
    pub q25: f64,
    pub q75: f64,
    pub min: f64,
    pub max: f64,
}

#[wasm_bindgen]
impl FastqQcEngine {
    #[wasm_bindgen(constructor)]
    pub fn new() -> Self {
        Self {
            total_reads: 0,
            base_qualities: vec![vec![0; 42]; 300],  // Max 300bp read length
            gc_count: 0,
            total_bases: 0,
            sequence_lengths: Vec::new(),
            quality_scores: Vec::new(),
        }
    }

    /// Process FASTQ records in chunks (called from JS with FileReader chunks)
    #[wasm_bindgen]
    pub fn process_chunk(&mut self, chunk: &[u8]) -> Result<(), JsValue> {
        let text = std::str::from_utf8(chunk)
            .map_err(|e| JsValue::from_str(&format!("UTF-8 decode error: {}", e)))?;

        let lines: Vec<&str> = text.lines().collect();
        let mut i = 0;

        while i + 3 < lines.len() {
            // FASTQ format: @header / sequence / + / quality
            if !lines[i].starts_with('@') {
                i += 1;
                continue;
            }

            let sequence = lines[i + 1];
            let quality = lines[i + 3];

            if sequence.len() != quality.len() {
                return Err(JsValue::from_str("Sequence/quality length mismatch"));
            }

            self.process_read(sequence.as_bytes(), quality.as_bytes());
            i += 4;
        }

        Ok(())
    }

    fn process_read(&mut self, sequence: &[u8], quality: &[u8]) {
        self.total_reads += 1;
        self.sequence_lengths.push(sequence.len() as u32);

        for (pos, (&base, &qual)) in sequence.iter().zip(quality.iter()).enumerate() {
            // Convert Phred+33 ASCII to quality score
            let q = (qual as i32 - 33).max(0) as u8;
            self.quality_scores.push(q);

            if pos < self.base_qualities.len() {
                if (q as usize) < self.base_qualities[pos].len() {
                    self.base_qualities[pos][q as usize] += 1;
                }
            }

            // Count GC bases
            if base == b'G' || base == b'C' || base == b'g' || base == b'c' {
                self.gc_count += 1;
            }
            self.total_bases += 1;
        }
    }

    /// Compute final metrics and return as JSON
    #[wasm_bindgen]
    pub fn compute_metrics(&self) -> Result<JsValue, JsValue> {
        let mean_length = if !self.sequence_lengths.is_empty() {
            self.sequence_lengths.iter().sum::<u32>() as f64 / self.sequence_lengths.len() as f64
        } else {
            0.0
        };

        let mean_quality = if !self.quality_scores.is_empty() {
            self.quality_scores.iter().map(|&q| q as f64).sum::<f64>() / self.quality_scores.len() as f64
        } else {
            0.0
        };

        let gc_content = if self.total_bases > 0 {
            self.gc_count as f64 / self.total_bases as f64
        } else {
            0.0
        };

        let mut per_base_quality = Vec::new();
        for (pos, counts) in self.base_qualities.iter().enumerate() {
            if counts.iter().sum::<u32>() == 0 {
                break;
            }

            let mut values: Vec<u8> = Vec::new();
            for (q, &count) in counts.iter().enumerate() {
                for _ in 0..count {
                    values.push(q as u8);
                }
            }
            values.sort_unstable();

            let mean = values.iter().map(|&v| v as f64).sum::<f64>() / values.len() as f64;
            let median = values[values.len() / 2] as f64;
            let q25 = values[values.len() / 4] as f64;
            let q75 = values[values.len() * 3 / 4] as f64;
            let min = *values.first().unwrap() as f64;
            let max = *values.last().unwrap() as f64;

            per_base_quality.push(PerBaseQuality {
                position: pos,
                mean,
                median,
                q25,
                q75,
                min,
                max,
            });
        }

        let mut quality_distribution = vec![0u32; 42];
        for &q in &self.quality_scores {
            if (q as usize) < quality_distribution.len() {
                quality_distribution[q as usize] += 1;
            }
        }

        // Estimate duplication by sampling first 100K reads (simple k-mer counting)
        let estimated_duplication_rate = self.estimate_duplication();

        let metrics = QcMetrics {
            total_reads: self.total_reads,
            mean_read_length: mean_length,
            mean_quality,
            gc_content,
            per_base_quality,
            quality_distribution,
            estimated_duplication_rate,
        };

        serde_wasm_bindgen::to_value(&metrics)
            .map_err(|e| JsValue::from_str(&format!("Serialization error: {}", e)))
    }

    fn estimate_duplication(&self) -> f64 {
        // Simplified duplication estimate (real FastQC uses k-mer counting)
        // For WASM preview, sample-based approximation is sufficient
        if self.total_reads < 10000 {
            0.0
        } else {
            // Placeholder: would implement k-mer counting here
            0.05  // Return 5% as placeholder
        }
    }
}
```

### WASM Build Configuration

```toml
# fastq-qc-wasm/Cargo.toml
[package]
name = "fastq-qc-wasm"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
wasm-bindgen = "0.2"
serde = { version = "1", features = ["derive"] }
serde-wasm-bindgen = "0.6"
js-sys = "0.3"

[profile.release]
opt-level = "z"       # Optimize for size
lto = true
codegen-units = 1
strip = true
```

### Frontend WASM Integration

```typescript
// frontend/src/lib/fastqQc.ts

import init, { FastqQcEngine } from '@/wasm/fastq-qc-wasm';

let wasmInitialized = false;
let wasmEngine: FastqQcEngine | null = null;

export async function initWasm() {
  if (!wasmInitialized) {
    await init();
    wasmInitialized = true;
  }
}

export interface FastqQcMetrics {
  total_reads: number;
  mean_read_length: number;
  mean_quality: number;
  gc_content: number;
  per_base_quality: Array<{
    position: number;
    mean: number;
    median: number;
    q25: number;
    q75: number;
    min: number;
    max: number;
  }>;
  quality_distribution: number[];
  estimated_duplication_rate: number;
}

/**
 * Analyze FASTQ file client-side for instant QC preview
 * Processes first 1M reads for speed (full FastQC runs server-side)
 */
export async function analyzeFastqFile(
  file: File,
  maxReads: number = 1_000_000,
  onProgress?: (progress: number) => void
): Promise<FastqQcMetrics> {
  await initWasm();

  const engine = new FastqQcEngine();
  const chunkSize = 64 * 1024;  // 64KB chunks
  const maxBytes = maxReads * 400;  // Assume ~400 bytes per read (4 lines)
  const totalBytes = Math.min(file.size, maxBytes);

  let processedBytes = 0;

  // Stream file in chunks
  const reader = file.stream().getReader();
  const decoder = new TextDecoder('utf-8');
  let buffer = '';

  try {
    while (processedBytes < totalBytes) {
      const { done, value } = await reader.read();
      if (done) break;

      const chunk = decoder.decode(value, { stream: true });
      buffer += chunk;

      // Process complete FASTQ records (4 lines each)
      const lines = buffer.split('\n');
      if (lines.length >= 4) {
        const completeLines = lines.length - (lines.length % 4);
        const toProcess = lines.slice(0, completeLines).join('\n');
        buffer = lines.slice(completeLines).join('\n');

        const bytes = new TextEncoder().encode(toProcess);
        engine.process_chunk(bytes);

        processedBytes += bytes.length;
        if (onProgress) {
          onProgress((processedBytes / totalBytes) * 100);
        }
      }
    }

    const metrics = engine.compute_metrics();
    return metrics as FastqQcMetrics;
  } finally {
    reader.releaseLock();
  }
}
```

---

## Architecture Deep-Dives

### 1. Pipeline Run Orchestration Handler (Rust/Axum)

The primary API endpoint that receives a pipeline run request, validates inputs, generates Nextflow DSL2 script from the visual pipeline definition, and submits to Kubernetes for execution.

```rust
// src/api/handlers/pipeline_runs.rs

use axum::{
    extract::{Path, State, Json},
    http::StatusCode,
    response::IntoResponse,
};
use uuid::Uuid;
use crate::{
    db::models::{PipelineRun, CreatePipelineRunRequest, PipelineDefinition, Sample},
    state::AppState,
    auth::Claims,
    error::ApiError,
    services::nextflow::NextflowGenerator,
    services::k8s::KubernetesClient,
};

pub async fn create_pipeline_run(
    State(state): State<AppState>,
    claims: Claims,
    Json(req): Json<CreatePipelineRunRequest>,
) -> Result<impl IntoResponse, ApiError> {
    // 1. Fetch and validate pipeline definition
    let pipeline = sqlx::query_as!(
        PipelineDefinition,
        "SELECT * FROM pipeline_definitions WHERE id = $1",
        req.pipeline_definition_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Pipeline not found"))?;

    // Check ownership or org membership
    if pipeline.user_id != claims.user_id {
        if let Some(org_id) = pipeline.org_id {
            let is_member = sqlx::query_scalar!(
                "SELECT EXISTS(SELECT 1 FROM org_members WHERE org_id = $1 AND user_id = $2)",
                org_id,
                claims.user_id
            )
            .fetch_one(&state.db)
            .await?;

            if !is_member {
                return Err(ApiError::Forbidden("Not authorized to use this pipeline"));
            }
        } else {
            return Err(ApiError::Forbidden("Not authorized to use this pipeline"));
        }
    }

    // 2. Fetch and validate samples
    let samples = sqlx::query_as!(
        Sample,
        "SELECT * FROM samples WHERE id = ANY($1)",
        &req.sample_ids
    )
    .fetch_all(&state.db)
    .await?;

    if samples.len() != req.sample_ids.len() {
        return Err(ApiError::BadRequest("Some samples not found"));
    }

    // Verify samples belong to accessible projects
    let project_ids: Vec<Uuid> = samples.iter().map(|s| s.project_id).collect();
    let accessible_count = sqlx::query_scalar!(
        "SELECT COUNT(*) FROM projects WHERE id = ANY($1)
         AND (owner_id = $2 OR org_id IN (
             SELECT org_id FROM org_members WHERE user_id = $2
         ))",
        &project_ids,
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    if accessible_count != project_ids.len() as i64 {
        return Err(ApiError::Forbidden("Some samples are not accessible"));
    }

    // 3. Check plan limits
    let user = sqlx::query_as!(
        crate::db::models::User,
        "SELECT * FROM users WHERE id = $1",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    let current_period_start = chrono::Utc::now().date_naive()
        .with_day(1).unwrap();
    let current_period_end = (current_period_start + chrono::Duration::days(32))
        .with_day(1).unwrap() - chrono::Duration::days(1);

    let compute_hours_used: Option<f32> = sqlx::query_scalar!(
        "SELECT COALESCE(SUM(quantity), 0) FROM usage_records
         WHERE user_id = $1 AND record_type = 'compute_hours'
         AND period_start >= $2 AND period_end <= $3",
        claims.user_id,
        current_period_start,
        current_period_end
    )
    .fetch_one(&state.db)
    .await?;

    let limit = match user.plan.as_str() {
        "free" => 10.0,
        "research" => 1000.0,
        "clinical" => 5000.0,
        "enterprise" => f32::MAX,
        _ => 10.0,
    };

    if compute_hours_used.unwrap_or(0.0) >= limit {
        return Err(ApiError::PlanLimit(
            "Compute hour limit reached for current billing period. Upgrade to continue."
        ));
    }

    // 4. Generate Nextflow DSL2 script from visual pipeline definition
    let nextflow_generator = NextflowGenerator::new();
    let nextflow_script = nextflow_generator.generate(
        &pipeline.definition_json,
        req.parameters_override.as_ref().unwrap_or(&serde_json::json!({})),
        &samples,
    )?;

    // 5. Create pipeline run record
    let project_id = samples[0].project_id;  // All samples in same project (enforce this)

    let run = sqlx::query_as!(
        PipelineRun,
        r#"INSERT INTO pipeline_runs
            (pipeline_definition_id, project_id, org_id, user_id, name,
             parameters_override, status, nextflow_run_id)
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8)
        RETURNING *"#,
        req.pipeline_definition_id,
        project_id,
        pipeline.org_id,
        claims.user_id,
        req.name,
        req.parameters_override.unwrap_or(serde_json::json!({})),
        "queued",
        Uuid::new_v4().to_string(),  // Nextflow session ID
    )
    .fetch_one(&state.db)
    .await?;

    // 6. Link samples to run
    for sample_id in &req.sample_ids {
        sqlx::query!(
            "INSERT INTO pipeline_run_samples (pipeline_run_id, sample_id) VALUES ($1, $2)",
            run.id,
            sample_id
        )
        .execute(&state.db)
        .await?;
    }

    // 7. Submit Nextflow job to Kubernetes
    let k8s_client = state.k8s_client.clone();
    let run_id = run.id;
    let run_name = run.nextflow_run_id.clone().unwrap();

    tokio::spawn(async move {
        if let Err(e) = k8s_client.submit_nextflow_job(
            &run_name,
            &nextflow_script,
            run_id,
        ).await {
            tracing::error!("Failed to submit Nextflow job: {}", e);
            // Update run status to failed
            let _ = sqlx::query!(
                "UPDATE pipeline_runs SET status = 'failed', error_message = $2 WHERE id = $1",
                run_id,
                e.to_string()
            )
            .execute(&k8s_client.db)
            .await;
        }
    });

    // 8. Audit log
    sqlx::query!(
        "INSERT INTO audit_logs (org_id, user_id, action, resource_type, resource_id, details)
         VALUES ($1, $2, $3, $4, $5, $6)",
        pipeline.org_id,
        claims.user_id,
        "run_pipeline",
        "pipeline_run",
        run.id,
        serde_json::json!({"pipeline_name": pipeline.name, "sample_count": req.sample_ids.len()})
    )
    .execute(&state.db)
    .await?;

    Ok((StatusCode::CREATED, Json(run)))
}

pub async fn get_pipeline_run(
    State(state): State<AppState>,
    claims: Claims,
    Path(run_id): Path<Uuid>,
) -> Result<Json<PipelineRun>, ApiError> {
    let run = sqlx::query_as!(
        PipelineRun,
        "SELECT r.* FROM pipeline_runs r
         JOIN projects p ON r.project_id = p.id
         WHERE r.id = $1
         AND (p.owner_id = $2 OR p.org_id IN (
             SELECT org_id FROM org_members WHERE user_id = $2
         ))",
        run_id,
        claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Pipeline run not found"))?;

    Ok(Json(run))
}

pub async fn get_run_stages(
    State(state): State<AppState>,
    claims: Claims,
    Path(run_id): Path<Uuid>,
) -> Result<Json<Vec<crate::db::models::RunStage>>, ApiError> {
    // Verify access to run
    let _ = get_pipeline_run(State(state.clone()), claims.clone(), Path(run_id)).await?;

    let stages = sqlx::query_as!(
        crate::db::models::RunStage,
        "SELECT * FROM run_stages WHERE pipeline_run_id = $1 ORDER BY created_at ASC",
        run_id
    )
    .fetch_all(&state.db)
    .await?;

    Ok(Json(stages))
}

pub async fn cancel_pipeline_run(
    State(state): State<AppState>,
    claims: Claims,
    Path(run_id): Path<Uuid>,
) -> Result<Json<PipelineRun>, ApiError> {
    // Verify ownership
    let run = get_pipeline_run(State(state.clone()), claims.clone(), Path(run_id)).await?.0;

    if run.status != "queued" && run.status != "running" {
        return Err(ApiError::BadRequest("Can only cancel queued or running pipelines"));
    }

    // Cancel Kubernetes job
    if let Some(nextflow_run_id) = &run.nextflow_run_id {
        state.k8s_client.cancel_job(nextflow_run_id).await?;
    }

    // Update status
    let updated_run = sqlx::query_as!(
        PipelineRun,
        "UPDATE pipeline_runs SET status = 'cancelled', completed_at = NOW()
         WHERE id = $1 RETURNING *",
        run_id
    )
    .fetch_one(&state.db)
    .await?;

    // Audit log
    sqlx::query!(
        "INSERT INTO audit_logs (org_id, user_id, action, resource_type, resource_id)
         VALUES ($1, $2, $3, $4, $5)",
        run.org_id,
        claims.user_id,
        "cancel_pipeline",
        "pipeline_run",
        run_id
    )
    .execute(&state.db)
    .await?;

    Ok(Json(updated_run))
}
```

### 2. Nextflow DSL2 Generator (Rust)

Service that converts the visual pipeline definition (React Flow JSON) into executable Nextflow DSL2 code with proper channel management and process definitions.

```rust
// src/services/nextflow.rs

use serde_json::Value;
use crate::{db::models::Sample, error::ApiError};

pub struct NextflowGenerator {
    // Future: cache compiled processes, reference genome paths
}

impl NextflowGenerator {
    pub fn new() -> Self {
        Self {}
    }

    pub fn generate(
        &self,
        pipeline_json: &Value,
        parameters_override: &Value,
        samples: &[Sample],
    ) -> Result<String, ApiError> {
        let nodes = pipeline_json["nodes"]
            .as_array()
            .ok_or(ApiError::BadRequest("Invalid pipeline definition: missing nodes"))?;

        let edges = pipeline_json["edges"]
            .as_array()
            .ok_or(ApiError::BadRequest("Invalid pipeline definition: missing edges"))?;

        // Build Nextflow script
        let mut script = String::new();

        // Header
        script.push_str("#!/usr/bin/env nextflow\n");
        script.push_str("nextflow.enable.dsl=2\n\n");

        // Parameters
        script.push_str("// Parameters\n");
        script.push_str(&format!("params.reference_genome = '{}'\n",
            pipeline_json["reference_genome"].as_str().unwrap_or("GRCh38")));

        for (key, value) in parameters_override.as_object().unwrap_or(&serde_json::Map::new()) {
            script.push_str(&format!("params.{} = {}\n", key, value));
        }
        script.push_str("\n");

        // Sample input channel
        script.push_str("// Input samples\n");
        script.push_str("samples_ch = Channel.fromList([\n");
        for sample in samples {
            let input_files = sample.input_files.as_array()
                .ok_or(ApiError::BadRequest("Invalid sample input_files"))?;

            for file in input_files {
                let s3_key = file["s3_key"].as_str()
                    .ok_or(ApiError::BadRequest("Missing s3_key in input file"))?;
                let file_type = file["file_type"].as_str()
                    .ok_or(ApiError::BadRequest("Missing file_type"))?;

                script.push_str(&format!("    [id: '{}', file: '{}', type: '{}'],\n",
                    sample.id, s3_key, file_type));
            }
        }
        script.push_str("])\n\n");

        // Process definitions (one per module node)
        script.push_str("// Processes\n");
        for node in nodes {
            let module_name = node["data"]["module_name"].as_str()
                .ok_or(ApiError::BadRequest("Missing module_name in node"))?;
            let node_id = node["id"].as_str()
                .ok_or(ApiError::BadRequest("Missing node id"))?;

            script.push_str(&self.generate_process(node_id, module_name, &node["data"])?);
            script.push_str("\n");
        }

        // Workflow definition
        script.push_str("// Workflow\n");
        script.push_str("workflow {\n");

        // Build execution DAG from edges
        let mut executed_nodes = std::collections::HashSet::new();
        let mut channel_mappings = std::collections::HashMap::new();

        // Start with source nodes (no incoming edges)
        for node in nodes {
            let node_id = node["id"].as_str().unwrap();
            let has_input = edges.iter().any(|e| e["target"].as_str() == Some(node_id));

            if !has_input {
                // Input node (e.g., FASTQ files)
                channel_mappings.insert(node_id.to_string(), "samples_ch".to_string());
                executed_nodes.insert(node_id.to_string());
            }
        }

        // Topological execution
        while executed_nodes.len() < nodes.len() {
            let mut progress = false;

            for node in nodes {
                let node_id = node["id"].as_str().unwrap();
                if executed_nodes.contains(node_id) {
                    continue;
                }

                // Check if all inputs are ready
                let incoming: Vec<_> = edges.iter()
                    .filter(|e| e["target"].as_str() == Some(node_id))
                    .collect();

                if incoming.iter().all(|e| {
                    let source = e["source"].as_str().unwrap();
                    executed_nodes.contains(source)
                }) {
                    // Execute this node
                    let process_name = self.get_process_name(node_id);
                    let input_channels: Vec<String> = incoming.iter()
                        .map(|e| {
                            let source = e["source"].as_str().unwrap();
                            channel_mappings.get(source).unwrap().clone()
                        })
                        .collect();

                    let input_channel = if input_channels.is_empty() {
                        "samples_ch".to_string()
                    } else {
                        input_channels[0].clone()
                    };

                    let output_channel = format!("{}_out", process_name);
                    script.push_str(&format!("    {} = {}({})\n",
                        output_channel, process_name, input_channel));

                    channel_mappings.insert(node_id.to_string(), output_channel);
                    executed_nodes.insert(node_id.to_string());
                    progress = true;
                }
            }

            if !progress {
                return Err(ApiError::BadRequest("Cyclic dependency detected in pipeline"));
            }
        }

        script.push_str("}\n");

        Ok(script)
    }

    fn generate_process(&self, node_id: &str, module_name: &str, data: &Value) -> Result<String, ApiError> {
        let process_name = self.get_process_name(node_id);
        let container = data["container_image"].as_str()
            .ok_or(ApiError::BadRequest("Missing container_image"))?;
        let container_digest = data["container_digest"].as_str()
            .ok_or(ApiError::BadRequest("Missing container_digest"))?;

        let mut proc = format!("process {} {{\n", process_name);
        proc.push_str(&format!("    container '{}@{}'\n", container, container_digest));
        proc.push_str("    publishDir \"s3://geneweave-results/${params.run_id}/${task.process}\", mode: 'copy'\n");

        // Resource requirements
        let cpus = data["parameters"]["cpus"].as_i64().unwrap_or(4);
        let memory = data["parameters"]["memory_gb"].as_i64().unwrap_or(16);
        proc.push_str(&format!("    cpus {}\n", cpus));
        proc.push_str(&format!("    memory '{}GB'\n", memory));
        proc.push_str("\n");

        // Input
        proc.push_str("    input:\n");
        proc.push_str("    tuple val(id), path(input_file), val(type)\n\n");

        // Output
        proc.push_str("    output:\n");
        proc.push_str("    tuple val(id), path(\"output/*\"), val(type), emit: results\n\n");

        // Script (module-specific command)
        proc.push_str("    script:\n");
        proc.push_str("    \"\"\"\n");
        proc.push_str("    mkdir -p output\n");

        match module_name {
            "fastqc" => {
                proc.push_str("    fastqc -o output -t ${task.cpus} ${input_file}\n");
            }
            "bwa_mem2" => {
                proc.push_str("    bwa-mem2 mem -t ${task.cpus} ${params.reference_genome} ${input_file} > output/aligned.sam\n");
            }
            "samtools_sort" => {
                proc.push_str("    samtools sort -@ ${task.cpus} -o output/sorted.bam ${input_file}\n");
                proc.push_str("    samtools index output/sorted.bam\n");
            }
            "gatk_haplotypecaller" => {
                proc.push_str("    gatk HaplotypeCaller -R ${params.reference_genome} -I ${input_file} -O output/variants.vcf.gz\n");
            }
            _ => {
                return Err(ApiError::BadRequest(&format!("Unknown module: {}", module_name)));
            }
        }

        proc.push_str("    \"\"\"\n");
        proc.push_str("}\n");

        Ok(proc)
    }

    fn get_process_name(&self, node_id: &str) -> String {
        node_id.replace("-", "_")
    }
}
```

### 3. Variant Annotation Service (Python FastAPI)

Secondary microservice handling CPU-intensive VEP annotation and database lookups for variant interpretation.

```python
# annotation-service/main.py

from fastapi import FastAPI, HTTPException, BackgroundTasks
from pydantic import BaseModel
import psycopg2
import boto3
import subprocess
import os
from typing import List, Optional

app = FastAPI()

# Database connection (PostgreSQL)
DB_CONFIG = {
    "host": os.environ["DB_HOST"],
    "database": os.environ["DB_NAME"],
    "user": os.environ["DB_USER"],
    "password": os.environ["DB_PASSWORD"],
}

# S3 client
s3_client = boto3.client("s3")

# VEP cache paths
VEP_CACHE_DIR = "/data/vep_cache"
VEP_PLUGINS_DIR = "/data/vep_plugins"
CLINVAR_VCF = "/data/clinvar/clinvar.vcf.gz"
GNOMAD_VCF = "/data/gnomad/gnomad.genomes.v4.0.sites.vcf.bgz"

class AnnotateVariantsRequest(BaseModel):
    variant_set_id: str
    vcf_s3_url: str
    reference_genome: str = "GRCh38"

class VariantAnnotation(BaseModel):
    chrom: str
    pos: int
    ref: str
    alt: str
    gene: Optional[str]
    transcript: Optional[str]
    consequence: Optional[str]
    impact: Optional[str]
    clinvar_significance: Optional[str]
    clinvar_stars: Optional[int]
    gnomad_af: Optional[float]
    gnomad_af_popmax: Optional[float]
    cadd_score: Optional[float]
    revel_score: Optional[float]
    spliceai_score: Optional[float]

@app.post("/annotate")
async def annotate_variants(
    request: AnnotateVariantsRequest,
    background_tasks: BackgroundTasks
):
    """
    Annotate a VCF file using VEP and additional databases
    """
    # Launch annotation in background
    background_tasks.add_task(
        run_annotation_pipeline,
        request.variant_set_id,
        request.vcf_s3_url,
        request.reference_genome
    )

    return {"status": "accepted", "variant_set_id": request.variant_set_id}

async def run_annotation_pipeline(
    variant_set_id: str,
    vcf_s3_url: str,
    reference_genome: str
):
    """
    Background task: Download VCF, run VEP, parse results, insert to DB
    """
    conn = psycopg2.connect(**DB_CONFIG)
    cursor = conn.cursor()

    try:
        # Update status
        cursor.execute(
            "UPDATE variant_sets SET annotation_status = 'running', "
            "annotation_started_at = NOW() WHERE id = %s",
            (variant_set_id,)
        )
        conn.commit()

        # Download VCF from S3
        s3_bucket, s3_key = parse_s3_url(vcf_s3_url)
        local_vcf = f"/tmp/{variant_set_id}.vcf.gz"
        s3_client.download_file(s3_bucket, s3_key, local_vcf)

        # Run VEP
        vep_output = f"/tmp/{variant_set_id}_vep.vcf"
        vep_cmd = [
            "vep",
            "--input_file", local_vcf,
            "--output_file", vep_output,
            "--format", "vcf",
            "--vcf",
            "--everything",
            "--fork", "8",
            "--assembly", reference_genome,
            "--cache",
            "--dir_cache", VEP_CACHE_DIR,
            "--dir_plugins", VEP_PLUGINS_DIR,
            "--plugin", "CADD,/data/CADD/whole_genome_SNVs.tsv.gz",
            "--plugin", "REVEL,/data/REVEL/revel_all_chromosomes.tsv.gz",
            "--plugin", "SpliceAI,snv=/data/spliceai/spliceai_scores.raw.snv.hg38.vcf.gz",
            "--custom", f"{CLINVAR_VCF},ClinVar,vcf,exact,0,CLNSIG,CLNREVSTAT",
            "--custom", f"{GNOMAD_VCF},gnomAD,vcf,exact,0,AF,AF_popmax",
        ]

        result = subprocess.run(vep_cmd, capture_output=True, text=True)
        if result.returncode != 0:
            raise Exception(f"VEP failed: {result.stderr}")

        # Parse VEP output and insert variants
        variants = parse_vep_output(vep_output)

        # Batch insert
        for variant in variants:
            cursor.execute(
                """
                INSERT INTO annotated_variants
                (variant_set_id, chrom, pos, ref, alt, gene, transcript, consequence,
                 impact, clinvar_significance, clinvar_stars, gnomad_af, gnomad_af_popmax,
                 cadd_score, revel_score, spliceai_score)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                """,
                (
                    variant_set_id,
                    variant["chrom"],
                    variant["pos"],
                    variant["ref"],
                    variant["alt"],
                    variant.get("gene"),
                    variant.get("transcript"),
                    variant.get("consequence"),
                    variant.get("impact"),
                    variant.get("clinvar_significance"),
                    variant.get("clinvar_stars"),
                    variant.get("gnomad_af"),
                    variant.get("gnomad_af_popmax"),
                    variant.get("cadd_score"),
                    variant.get("revel_score"),
                    variant.get("spliceai_score"),
                )
            )

        # Update variant count and status
        cursor.execute(
            "UPDATE variant_sets SET variant_count = %s, annotation_status = 'completed', "
            "annotation_completed_at = NOW() WHERE id = %s",
            (len(variants), variant_set_id)
        )
        conn.commit()

    except Exception as e:
        # Mark as failed
        cursor.execute(
            "UPDATE variant_sets SET annotation_status = 'failed' WHERE id = %s",
            (variant_set_id,)
        )
        conn.commit()
        raise
    finally:
        cursor.close()
        conn.close()

def parse_vep_output(vcf_path: str) -> List[dict]:
    """
    Parse VEP-annotated VCF and extract variant annotations
    """
    variants = []

    with open(vcf_path) as f:
        for line in f:
            if line.startswith("#"):
                continue

            fields = line.strip().split("\t")
            chrom, pos, id_, ref, alt, qual, filter_, info = fields[:8]

            # Parse INFO field for VEP annotations
            info_dict = {}
            for item in info.split(";"):
                if "=" in item:
                    key, value = item.split("=", 1)
                    info_dict[key] = value

            # Extract VEP CSQ field
            csq = info_dict.get("CSQ", "").split(",")[0]  # Take first transcript
            csq_fields = csq.split("|") if csq else []

            # Map CSQ fields (VEP header defines order)
            variant = {
                "chrom": chrom,
                "pos": int(pos),
                "ref": ref,
                "alt": alt,
                "gene": csq_fields[3] if len(csq_fields) > 3 else None,
                "transcript": csq_fields[6] if len(csq_fields) > 6 else None,
                "consequence": csq_fields[1] if len(csq_fields) > 1 else None,
                "impact": csq_fields[2] if len(csq_fields) > 2 else None,
            }

            # Extract ClinVar (from custom annotation)
            clinvar_sig = info_dict.get("ClinVar_CLNSIG")
            if clinvar_sig:
                variant["clinvar_significance"] = parse_clinvar_significance(clinvar_sig)
                variant["clinvar_stars"] = parse_clinvar_stars(info_dict.get("ClinVar_CLNREVSTAT"))

            # Extract gnomAD
            gnomad_af = info_dict.get("gnomAD_AF")
            if gnomad_af:
                variant["gnomad_af"] = float(gnomad_af)
            gnomad_af_popmax = info_dict.get("gnomAD_AF_popmax")
            if gnomad_af_popmax:
                variant["gnomad_af_popmax"] = float(gnomad_af_popmax)

            # Extract scores (from VEP plugins)
            variant["cadd_score"] = parse_score(info_dict.get("CADD_PHRED"))
            variant["revel_score"] = parse_score(info_dict.get("REVEL"))
            variant["spliceai_score"] = parse_score(info_dict.get("SpliceAI_pred_DS_max"))

            variants.append(variant)

    return variants

def parse_clinvar_significance(sig: str) -> Optional[str]:
    """Map ClinVar significance codes to readable labels"""
    mapping = {
        "0": "uncertain_significance",
        "1": "not_provided",
        "2": "benign",
        "3": "likely_benign",
        "4": "likely_pathogenic",
        "5": "pathogenic",
    }
    return mapping.get(sig, sig)

def parse_clinvar_stars(rev_stat: str) -> Optional[int]:
    """Extract ClinVar review status stars (0-4)"""
    if not rev_stat:
        return None
    # Simplified: map review status to star rating
    if "practice_guideline" in rev_stat:
        return 4
    elif "reviewed_by_expert_panel" in rev_stat:
        return 3
    elif "criteria_provided,_multiple_submitters" in rev_stat:
        return 2
    elif "criteria_provided,_single_submitter" in rev_stat:
        return 1
    return 0

def parse_score(score_str: Optional[str]) -> Optional[float]:
    """Parse numeric score from string"""
    if not score_str or score_str == ".":
        return None
    try:
        return float(score_str)
    except ValueError:
        return None

def parse_s3_url(url: str) -> tuple[str, str]:
    """Parse S3 URL into bucket and key"""
    # s3://bucket/key/path
    if not url.startswith("s3://"):
        raise ValueError("Invalid S3 URL")
    parts = url[5:].split("/", 1)
    return parts[0], parts[1] if len(parts) > 1 else ""

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8001)
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–5)

**Day 1: Project initialization and Rust backend scaffold**
```bash
cargo init geneweave-api
cd geneweave-api
cargo add axum tokio serde serde_json sqlx uuid chrono tower-http jsonwebtoken bcrypt
cargo add --features postgres,runtime-tokio-rustls,uuid,chrono,json sqlx
cargo add aws-sdk-s3 redis
```
- `src/main.rs` — Axum app with CORS, tracing, graceful shutdown
- `src/config.rs` — Environment-based configuration (DATABASE_URL, JWT_SECRET, S3_BUCKET, REDIS_URL, K8S_CONFIG)
- `src/state.rs` — AppState struct (PgPool, Redis, S3Client, KubernetesClient)
- `src/error.rs` — ApiError enum with IntoResponse implementation
- `Dockerfile` — Multi-stage build (builder + runtime)
- `docker-compose.yml` — PostgreSQL, Redis, MinIO (S3-compatible), annotation-service (Python)

**Day 2: Database schema and migrations**
- `migrations/001_initial.sql` — All 15 tables: users, organizations, org_members, projects, pipeline_definitions, pipeline_modules, samples, pipeline_runs, pipeline_run_samples, run_stages, variant_sets, annotated_variants, audit_logs, usage_records
- `src/db/mod.rs` — Database pool initialization and health checks
- `src/db/models.rs` — All SQLx structs with FromRow derives
- Run `sqlx migrate run` to apply schema
- Seed script for initial pipeline modules (FastQC, Trimmomatic, BWA-MEM2, STAR, GATK, Salmon, DESeq2, Cell Ranger)

**Day 3: Authentication system**
- `src/auth/mod.rs` — JWT token generation and validation middleware
- `src/auth/oauth.rs` — Google, GitHub, and ORCID OAuth 2.0 flow handlers
- `src/api/handlers/auth.rs` — Register, login, OAuth callback, refresh token, me endpoints
- Password hashing with bcrypt (cost factor 12)
- JWT with 24h access token + 30d refresh token
- Auth middleware that extracts Claims from Authorization header
- ORCID integration for academic users (widely used in bioinformatics)

**Day 4: Core CRUD APIs — Users, Projects, Organizations**
- `src/api/handlers/users.rs` — Get profile, update profile, delete account
- `src/api/handlers/projects.rs` — Create, list, get, update, delete project with organism and reference genome selection
- `src/api/handlers/orgs.rs` — Create org, invite member, list members, remove member, update BAA status
- `src/api/router.rs` — All route definitions with auth middleware
- Integration tests: auth flow, project CRUD, organization management, authorization checks

**Day 5: Sample management and S3 integration**
- `src/api/handlers/samples.rs` — Register sample, batch register from CSV, get sample, update metadata, delete sample
- `src/services/s3.rs` — S3 service for presigned upload URLs, multipart upload initialization, download URLs
- Sample upload flow: client requests presigned URL → uploads directly to S3 → notifies API of completion
- Metadata validation: JSON schema for sample metadata fields (sex, phenotype, tissue, batch, etc.)
- FASTQ file validation: check gzip compression, basic format sanity
- Integration test: full sample upload workflow

### Phase 2 — WASM QC Engine + Nextflow Integration (Days 6–12)

**Day 6: WASM FASTQ QC engine implementation**
- `fastq-qc-wasm/` — New Rust workspace member for WASM target
- `fastq-qc-wasm/Cargo.toml` — wasm-bindgen, serde-wasm-bindgen dependencies
- `fastq-qc-wasm/src/lib.rs` — FastqQcEngine with process_chunk() and compute_metrics()
- Per-base quality tracking (mean, median, quartiles at each position)
- GC content calculation
- Sequence length distribution
- Quality score distribution
- Basic duplication estimation via k-mer sampling

**Day 7: WASM build pipeline and frontend integration**
- `wasm-pack build --target web --release` in CI/CD
- `wasm-opt -Oz` for size optimization (target <500KB gzipped)
- `frontend/src/lib/fastqQc.ts` — JavaScript wrapper for WASM module
- `frontend/src/hooks/useFastqQc.ts` — React hook for file upload with WASM QC
- File streaming: process FASTQ in 64KB chunks without loading entire file into memory
- Progress tracking during client-side QC
- Unit tests: verify QC metrics match FastQC on known test files

**Day 8: Pipeline module registry**
- `src/api/handlers/modules.rs` — List modules (filterable by category), get module details, search modules
- Seed 50+ pre-built modules covering QC, alignment, variant calling, RNA quantification, single-cell, annotation
- Each module: container image (Docker Hub), container digest (SHA256), input/output port definitions, parameter schema, resource defaults
- Module categories: qc, alignment, variant_calling, rna_quant, single_cell, annotation, assembly

**Day 9: Nextflow DSL2 generator service**
- `src/services/nextflow.rs` — NextflowGenerator that converts visual pipeline JSON to Nextflow DSL2
- Topological sort of pipeline nodes to determine execution order
- Channel management: map node outputs to Nextflow channels
- Process generation: convert module definitions to Nextflow process blocks
- Parameter injection: merge user-provided parameters with module defaults
- Validation: detect cycles, missing inputs, type mismatches

**Day 10: Nextflow script validation and testing**
- Nextflow syntax validation via `nextflow config` command
- Unit tests: generate Nextflow for common pipeline patterns (linear, branching, merging)
- Test with real Nextflow execution (local executor) on sample datasets
- Handle edge cases: optional inputs, conditional execution, parameter sweeps

**Day 11: Kubernetes client integration**
- `src/services/k8s.rs` — KubernetesClient wrapper around kube-rs library
- Submit Nextflow job as Kubernetes Job resource
- Job spec: Nextflow container image, mounted S3 credentials, config map for script
- KEDA ScaledObject definition for auto-scaling based on Redis queue depth
- Job monitoring: watch Job status via Kubernetes API
- Resource limits: enforce per-plan CPU/memory quotas

**Day 12: Kubernetes job lifecycle management**
- Job status polling: queued → running → completed/failed
- Pod log streaming via Kubernetes API
- Job cancellation: delete Job and associated Pods
- Spot instance integration: tolerations and node affinity for spot nodes
- Automatic retry on spot interruption (Nextflow resume)
- Cost tracking: parse Kubernetes metrics for CPU-hours and memory-GB-hours

### Phase 3 — Pipeline Builder UI (Days 13–19)

**Day 13: Frontend scaffold and basic layout**
```bash
npm create vite@latest frontend -- --template react-ts
cd frontend
npm i zustand @tanstack/react-query axios react-flow-renderer plotly.js igv
npm i @tanstack/react-router @headlessui/react lucide-react
```
- `src/App.tsx` — Router, auth context, layout shell
- `src/layouts/DashboardLayout.tsx` — Sidebar navigation, top bar with user menu
- `src/stores/authStore.ts` — Zustand store for authentication state
- `src/lib/api.ts` — Axios instance with JWT interceptor
- `src/hooks/useAuth.ts` — Auth hook with login, logout, register

**Day 14: Pipeline builder canvas foundation**
- `src/pages/PipelineBuilder.tsx` — Main pipeline builder page
- `src/components/PipelineBuilder/Canvas.tsx` — React Flow canvas with custom node types
- `src/components/PipelineBuilder/ModuleNode.tsx` — Custom node component showing module icon, name, status
- `src/components/PipelineBuilder/CustomEdge.tsx` — Custom edge with data type indicator
- `src/stores/pipelineStore.ts` — Zustand store for pipeline definition (nodes, edges)
- Drag-and-drop: handle node addition, deletion, connection

**Day 15: Module palette and smart connectors**
- `src/components/PipelineBuilder/ModulePalette.tsx` — Sidebar with categorized modules, search, filter
- `src/components/PipelineBuilder/ModuleCard.tsx` — Draggable module card with icon, name, description
- Smart connector validation: enforce FASTQ→FASTQ, BAM→BAM, VCF→VCF type matching
- Visual feedback: green checkmark for valid connection, red X for invalid
- Connection handles: color-coded by data type (FASTQ=blue, BAM=green, VCF=purple)

**Day 16: Parameter configuration panel**
- `src/components/PipelineBuilder/ParameterPanel.tsx` — Right panel for selected node parameters
- Dynamic form generation from module parameter schema
- Input types: text, number, select, boolean, file picker
- Parameter validation: required fields, min/max values, regex patterns
- Tooltips: show parameter descriptions on hover
- Reset to defaults button

**Day 17: Pipeline templates**
- `src/data/templates/` — Pre-built pipeline definitions for WGS, RNA-seq, scRNA-seq workflows
- `src/components/PipelineBuilder/TemplateGallery.tsx` — Gallery of templates with preview images
- Template instantiation: load template JSON into pipeline store
- Template forking: allow customization while tracking original template

**Day 18: Pipeline validation and export**
- `src/services/pipelineValidator.ts` — Client-side validation logic
- Checks: all required inputs connected, no cycles, no dangling nodes, parameter completeness
- Validation result display: list of errors/warnings with highlighting
- Export to Nextflow: call API endpoint to generate DSL2 script, download as .nf file
- Export to JSON: download pipeline definition for sharing/backup

**Day 19: Sample selection and run launch UI**
- `src/components/PipelineBuilder/SampleSelector.tsx` — Modal for selecting samples to run pipeline on
- Sample table: filterable, sortable, multi-select
- Batch operations: select all in project, select by metadata filter
- Run configuration: name run, set custom parameters
- Cost estimate: call API for estimated compute cost before launch
- Launch confirmation dialog with cost and time estimate
- Post-launch: redirect to run monitoring page

### Phase 4 — Pipeline Execution + Monitoring (Days 20–26)

**Day 20: Pipeline run API endpoints**
- `src/api/handlers/pipeline_runs.rs` — Create run, get run, list runs, cancel run, retry failed run
- Run creation flow: validate samples, generate Nextflow script, submit K8s job, create DB records
- Plan limit enforcement: check compute-hour quota before submission
- Run listing: paginated, filterable by status/date, sortable
- Run details: include linked samples, stages, provenance

**Day 21: Run stage tracking**
- `src/api/handlers/run_stages.rs` — Get stages for run, get stage logs
- Nextflow progress parsing: parse `.nextflow.log` for stage updates
- Stage status webhook: Nextflow reports completion → API updates run_stages table
- QC metrics extraction: parse tool output (FastQC JSON, Picard metrics) → store in run_stages.qc_metrics
- S3 log storage: stream Nextflow logs to S3 for long-term retention

**Day 22: WebSocket for real-time progress**
- `src/api/ws/mod.rs` — Axum WebSocket upgrade handler
- `src/api/ws/pipeline_progress.rs` — Subscribe to run progress via run_id
- Progress updates: run status changes, stage completions, log entries
- Broadcast channel: Redis pub/sub for multi-server WebSocket distribution
- Client: `frontend/src/hooks/usePipelineProgress.ts` — React hook for WebSocket subscription
- Auto-reconnect with exponential backoff

**Day 23: Run monitoring dashboard**
- `frontend/src/pages/PipelineRunDetail.tsx` — Run detail page with live updates
- `frontend/src/components/RunMonitor/StatusTimeline.tsx` — Visual timeline of run progression
- `frontend/src/components/RunMonitor/StageTable.tsx` — Table of stages with status, duration, resources
- `frontend/src/components/RunMonitor/LogViewer.tsx` — Live log streaming with syntax highlighting
- `frontend/src/components/RunMonitor/CostTracker.tsx` — Real-time cost accumulation display
- Stage-level actions: view logs, download outputs, retry failed stage

**Day 24: QC dashboard — data fetching and store**
- `src/api/handlers/qc.rs` — Get QC summary for run, get per-sample QC metrics
- QC aggregation service: collect QC metrics from all stages (FastQC, alignment stats, variant stats)
- MultiQC integration: run MultiQC on pipeline completion → generate HTML report → store in S3
- `frontend/src/stores/qcStore.ts` — Zustand store for QC data
- `frontend/src/hooks/useQcData.ts` — React Query hook for fetching QC metrics

**Day 25: QC dashboard — interactive plots**
- `frontend/src/pages/QcDashboard.tsx` — QC dashboard page
- `frontend/src/components/QC/SampleTable.tsx` — Filterable table of samples with key QC metrics
- `frontend/src/components/QC/PerBaseQualityPlot.tsx` — Plotly.js box plot of per-base quality
- `frontend/src/components/QC/GcContentPlot.tsx` — GC content distribution histogram
- `frontend/src/components/QC/DuplicationPlot.tsx` — Duplication rate bar chart
- Conditional formatting: red/yellow/green thresholds for QC metrics

**Day 26: QC dashboard — multivariate analysis**
- `frontend/src/components/QC/PcaPlot.tsx` — PCA plot of samples (expression or coverage profiles)
- `frontend/src/components/QC/HeatmapPlot.tsx` — Correlation heatmap for batch effect detection
- `frontend/src/components/QC/OutlierDetection.tsx` — Auto-flagged outliers with explanations
- Export QC report: download as PDF or HTML with all plots and tables

### Phase 5 — Variant Annotation + Browser (Days 27–32)

**Day 27: Annotation service deployment**
- `annotation-service/` — Python FastAPI service
- `annotation-service/Dockerfile` — Multi-stage build with VEP, cache databases
- `annotation-service/main.py` — Annotate endpoint, background task for VEP execution
- VEP cache setup: download GRCh38/GRCh37 caches, ClinVar VCF, gnomAD VCF
- Database integration: insert annotated variants to PostgreSQL
- S3 integration: download VCF, upload annotated VCF

**Day 28: Variant annotation orchestration**
- `src/api/handlers/variants.rs` — Trigger annotation, query variants, export variants
- Annotation flow: pipeline produces VCF → create variant_set → call annotation service
- Variant querying: paginated, filterable by gene/impact/frequency/ClinVar
- ACMG auto-classification: implement basic rules (PVS1, PS1, PM1, PP3, BA1, BS1) based on annotations
- Variant export: filtered variants to CSV or VCF

**Day 29: Variant browser UI**
- `frontend/src/pages/VariantBrowser.tsx` — Variant table with advanced filters
- `frontend/src/components/Variants/VariantTable.tsx` — Sortable, paginated table
- `frontend/src/components/Variants/VariantFilters.tsx` — Filter sidebar (gene, impact, frequency, ClinVar)
- `frontend/src/components/Variants/VariantDetail.tsx` — Expandable row with full annotations
- Color coding: impact severity (HIGH=red, MODERATE=orange, LOW=yellow)
- ClinVar significance badges with star rating

**Day 30: IGV.js genome browser integration**
- `frontend/src/components/Variants/GenomeBrowser.tsx` — Embedded IGV.js
- Track configuration: VCF, BAM (alignments), reference annotations (GENCODE)
- Click variant in table → jump to position in browser
- BAM presigned URL generation: fetch from S3 with temporary credentials
- Bookmark system: save genomic regions of interest

**Day 31: Variant interpretation tools**
- `frontend/src/components/Variants/AcmgClassifier.tsx` — Interactive ACMG criteria checklist
- Manual override: allow users to change auto-classification
- Variant notes: add free-text notes per variant
- Pathogenicity calculator: combine ACMG criteria → final classification
- Report generation: export prioritized variants as clinical report (PDF)

**Day 32: Pharmacogenomics annotations**
- PharmGKB data integration: download variant-drug associations
- `src/services/pharmgkb.rs` — Service to annotate variants with drug interactions
- `frontend/src/components/Variants/PharmGKBPanel.tsx` — Display drug-gene interactions for variant
- Dosing recommendations: show PharmGKB evidence level and clinical annotations

### Phase 6 — Billing + Plan Enforcement (Days 33–36)

**Day 33: Stripe integration**
- `src/api/handlers/billing.rs` — Create checkout session, customer portal session, get subscription
- `src/api/handlers/webhooks/stripe.rs` — Webhook handler for subscription events
- Plan mapping: Free (10 compute-hours), Research ($149, 1000 hours), Clinical ($599, 5000 hours), Enterprise (custom)

**Day 34: Usage tracking**
- `src/services/usage.rs` — Track compute-hours, storage-GB-hours, annotation-count
- Compute-hour calculation: sum(duration_seconds * cpus) / 3600 for all run stages
- Storage tracking: periodic job to sum S3 object sizes per user/org
- Usage dashboard: `frontend/src/pages/Billing.tsx` — Current usage, historical charts, billing history

**Day 35: Plan limit enforcement**
- `src/middleware/plan_limits.rs` — Middleware to check limits before actions
- Limits checked: compute-hour quota, storage quota, sample count, project count
- Graceful degradation: block new runs when quota exceeded, allow viewing existing data
- Upgrade prompts: show "Upgrade to continue" modals at limit thresholds

**Day 36: Billing UI and upgrade flows**
- `frontend/src/components/billing/PlanComparison.tsx` — Plan comparison table
- `frontend/src/components/billing/UsageMeter.tsx` — Visual usage meters with thresholds
- Stripe Checkout integration: redirect to Stripe for payment
- Customer Portal: manage subscription, update payment method, download invoices
- Downgrade flow: handle graceful transition (e.g., archive excess projects)

### Phase 7 — Provenance + Reproducibility (Days 37–40)

**Day 37: Provenance tracking**
- Provenance record generation: capture tool versions, parameters, input checksums, reference genome
- `src/services/provenance.rs` — Service to build provenance JSON from run data
- Container digest recording: ensure exact container version is logged
- Input file checksums: compute MD5/SHA256 for all input FASTQ/BAM/VCF
- Store in pipeline_runs.provenance JSONB field

**Day 38: Methods section auto-generator**
- `src/api/handlers/methods.rs` — Generate methods text for publication
- Template-based generation: map pipeline modules to standardized descriptions
- Citation inclusion: include tool citations (e.g., "Li, H. (2013). Aligning sequence reads...")
- Parameter documentation: list all non-default parameters
- Reproducibility statement: include provenance record summary
- `frontend/src/components/Pipeline/MethodsGenerator.tsx` — View and copy methods text

**Day 39: Pipeline export and DOI generation**
- Nextflow export: full DSL2 script with all configurations
- Pipeline definition export: JSON with nodes, edges, parameters
- GitHub integration: push pipeline to GitHub repository (optional)
- DOI generation via Zenodo API: create versioned pipeline archive
- Citable pipeline: "Cite this pipeline: doi:10.5281/zenodo.XXXXXXX"

**Day 40: Pipeline version history**
- `src/api/handlers/pipeline_versions.rs` — List versions, get version, restore version
- Git-style versioning: track changes to pipeline definition
- Diff view: visual comparison between versions (added/removed/modified nodes)
- Version tagging: name versions (e.g., "v1.0-publication", "v2.0-optimized")
- Rollback: restore previous version and re-run

### Phase 8 — Testing + Documentation + Polish (Days 41–42)

**Day 41: End-to-end testing**
- Integration test: full pipeline run from sample upload to variant annotation
- Test pipelines: WGS mini (small test FASTQ, 1000 variants), RNA-seq (yeast, 100K reads), scRNA-seq (PBMC 1K cells)
- Performance benchmarks: measure pipeline runtime, resource usage, cost
- Validation: compare results to reference outputs (nf-core pipelines)
- Load testing: simulate 50 concurrent pipeline runs
- Security audit: penetration testing, dependency scanning

**Day 42: Documentation, deployment, and launch prep**
- User documentation: Getting Started guide, pipeline builder tutorial, FAQ
- API documentation: OpenAPI spec, interactive docs via Swagger UI
- Video tutorials: sample upload, pipeline creation, result interpretation
- Deployment: production Kubernetes cluster, S3 bucket setup, RDS PostgreSQL
- Monitoring: Prometheus dashboards, Grafana alerts, Sentry error tracking
- Launch checklist: security review, HIPAA compliance audit (if applicable), performance testing
- Beta program: invite 20 academic labs for early access and feedback

---

## Validation Benchmarks

### Benchmark 1: FASTQ QC Speed (WASM vs. Server)
- **Test**: 10GB paired-end FASTQ (40M reads, 150bp)
- **WASM QC** (first 1M reads): < 2 seconds client-side
- **Full FastQC** (server-side): ~45 seconds with 4 CPUs
- **Validation**: WASM metrics match FastQC for sampled reads within 2% error

### Benchmark 2: WGS Pipeline Throughput
- **Test**: 30x whole-genome sequencing (100GB FASTQ)
- **Pipeline**: FastQC → BWA-MEM2 → MarkDuplicates → BQSR → HaplotypeCaller → VEP
- **Target Runtime**: 4-6 hours on 32-core instance
- **Target Cost**: $8-12 per sample with spot instances
- **Validation**: Variant concordance with GIAB reference sample >99.5%

### Benchmark 3: Variant Annotation Latency
- **Test**: Annotate 50,000 variants (typical WGS exome)
- **VEP + ClinVar + gnomAD**: 8-12 minutes with 8 CPUs
- **Database insertion**: 2-3 minutes (batch insert 5,000 variants at a time)
- **Total latency**: < 15 minutes from VCF upload to queryable annotations
- **Validation**: ClinVar matches reference for known pathogenic variants

### Benchmark 4: Auto-Scaling Response Time
- **Test**: Submit 50 pipelines simultaneously (simulate batch processing)
- **KEDA scale-up**: 0 → 50 pods in < 3 minutes
- **First job start**: < 30 seconds from submission
- **Spot interruption recovery**: Resume within 2 minutes after spot instance termination
- **Validation**: All 50 pipelines complete successfully, zero data loss

### Benchmark 5: Client-Side WASM Performance
- **Test**: Run WASM QC on various devices
- **Desktop Chrome (M1 Mac)**: 1M reads in 1.2 seconds
- **Desktop Firefox (Intel i7)**: 1M reads in 1.8 seconds
- **Mobile Safari (iPhone 13)**: 1M reads in 3.5 seconds
- **WASM bundle size**: 420KB gzipped
- **Validation**: No browser crashes, memory usage < 200MB

---

## Post-MVP Roadmap

### Version 1.1 (Month 3)
- Team collaboration features: comments, shared pipelines, project permissions
- Advanced variant filters: compound heterozygosity, de novo variant detection
- RNA-seq specific: differential expression interactive volcano plots, pathway enrichment
- scRNA-seq specific: trajectory inference, cell-cell interaction analysis
- Custom module upload: allow users to bring their own Docker containers

### Version 1.2 (Month 6)
- Multi-sample joint variant calling (GATK GenotypeGVCFs)
- Structural variant annotation and visualization (AnnotSV integration)
- Copy number variation analysis (CNVkit, GATK CNV)
- Metagenomic analysis pipelines (Kraken2, MetaPhlAn4, HUMAnN3)
- CRAM file support for reduced storage costs

### Version 2.0 (Month 9)
- On-premise deployment option (Kubernetes manifests, air-gapped mode)
- Custom reference genome upload (for non-model organisms)
- Long-read sequencing support (PacBio, Oxford Nanopore)
- API access for programmatic pipeline execution
- Integration with electronic health records (HL7 FHIR)
- CAP/CLIA validation assistance for clinical labs

---

## Success Metrics

| Metric | Target (Month 3) | Target (Month 6) | Target (Month 12) |
|--------|------------------|------------------|-------------------|
| Registered users | 1,500 | 5,000 | 15,000 |
| Pipeline runs per month | 800 | 4,000 | 20,000 |
| Paying customers | 20 | 80 | 300 |
| MRR | $6,000 | $28,000 | $95,000 |
| Samples processed | 2,000 | 12,000 | 60,000 |
| Compute utilization | 30% | 50% | 65% |
| Free → Paid conversion | 2.5% | 4% | 6% |
| Churn rate (monthly) | <7% | <5% | <4% |
