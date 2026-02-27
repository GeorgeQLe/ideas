# 23. NeuralForge — Visual Neural Network Architecture Builder & Trainer

## Implementation Plan

**MVP Scope:** Browser-based neural architecture editor with drag-and-drop layer palette (Conv2D/3D, LSTM, GRU, Transformer, attention mechanisms, normalization, pooling, dropout, 30+ total) supporting real-time tensor shape propagation via WebAssembly-compiled Rust validation engine running client-side in <10ms, dataset manager with built-in catalog (MNIST, CIFAR-10/100, ImageNet subset) plus S3-backed custom upload with visual preview gallery and augmentation pipeline builder, cloud GPU training orchestration via Python FastAPI + Celery managing PyTorch 2.x training on T4/A10G instances with live metric streaming (loss curves, accuracy, gradients, GPU utilization) over WebSocket connections, experiment tracking storing architecture snapshots as JSONB graph data in PostgreSQL alongside hyperparameters/dataset versions/random seeds for full reproducibility, ONNX model export with validation pipeline comparing original vs. exported inference outputs on 100 test samples (max absolute difference <1e-4 threshold), architecture templates (ResNet-18, U-Net, simple LSTM, basic Transformer encoder) instantiated as fully editable graphs, Stripe billing with Free (1 project, 10 GPU-hours T4/month), Pro ($79/mo), Team ($249/mo) tiers.

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async, Tower middleware, JWT auth, CORS |
| Graph Validation | Rust + WASM (`wasm-pack`) | Shape propagation compiled to WASM for instant client validation |
| Training Service | Python 3.12 (FastAPI) | Separate microservice for PyTorch training, export, evaluation |
| Training Engine | PyTorch 2.x | torch.compile JIT, AMP, DistributedDataParallel |
| Database | PostgreSQL 16 | Projects, users, architectures (JSONB graph), experiments |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, async migrations |
| Auth | Custom JWT + OAuth 2.0 | Google, GitHub providers, bcrypt hashing |
| Object Storage | AWS S3 | Datasets, checkpoints, exported models, training logs |
| Frontend Framework | React 19 + TypeScript | Vite build, strict types |
| Architecture Canvas | React Flow 11 | Node-graph editor with custom layer nodes |
| Charts | Recharts | Training curves, experiment comparison |
| State Management | Zustand | Client state; React Query for server caching |
| Real-time | WebSocket (Axum) | Live training progress streaming |
| Job Queue | Redis 7 + Python Celery | GPU job queue, worker pool |
| Billing | Stripe | Checkout, Customer Portal, webhooks |
| CDN | CloudFront | WASM bundle, static assets |
| Monitoring | Prometheus + Grafana, Sentry | Metrics, error tracking |
| CI/CD | GitHub Actions | WASM build, Docker images, tests |

### Key Architecture Decisions

1. **Hybrid Rust API + Python Training Service**: Main API (user management, projects, experiment metadata) runs in Rust/Axum for type safety and performance, while a separate Python FastAPI microservice handles PyTorch training, ONNX export, and model evaluation. Keeps ML dependencies isolated while leveraging Rust for high-throughput metadata operations.

2. **WASM graph validation for instant client-side shape checking**: Rust-based tensor shape propagation engine compiles to WebAssembly and runs in browser, providing sub-10ms validation when users connect layers. Eliminates round-trip latency and works offline. Same Rust code runs server-side as final validation before training.

3. **Architecture stored as JSONB graph in PostgreSQL**: Each architecture version is JSONB document with nodes (layers + configs) and edges (connections + shapes). Enables versioning, diffing, and SQL queries. JSONB consumed directly by frontend (React Flow) and Python service (translated to PyTorch nn.Module).

4. **Separate GPU worker pool with Redis job queue**: Training jobs enqueued in Redis, consumed by Celery workers on GPU instances. Workers stream progress via Redis pub/sub → Rust API → WebSocket to clients. Decouples API from long-running GPU jobs, allows horizontal GPU scaling.

5. **S3 for datasets and checkpoints with presigned URLs**: User-uploaded datasets and checkpoints stored in S3 with versioning. API generates presigned URLs for direct browser-to-S3 uploads and downloads. Dataset metadata cached in PostgreSQL for fast queries.

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
    gpu_hours_used REAL DEFAULT 0.0,
    gpu_hours_quota REAL DEFAULT 10.0,
    billing_period_start DATE,
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
    gpu_hours_quota REAL DEFAULT 500.0,
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

-- Projects
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    task_type TEXT NOT NULL,
    visibility TEXT NOT NULL DEFAULT 'private',
    forked_from UUID REFERENCES projects(id) ON DELETE SET NULL,
    thumbnail_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);
CREATE INDEX projects_org_idx ON projects(org_id);
CREATE INDEX projects_task_idx ON projects(task_type);
CREATE INDEX projects_updated_idx ON projects(updated_at DESC);

-- Architectures
CREATE TABLE architectures (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    version INTEGER NOT NULL,
    parent_version_id UUID REFERENCES architectures(id) ON DELETE SET NULL,
    graph_data JSONB NOT NULL,
    message TEXT DEFAULT '',
    parameter_count BIGINT DEFAULT 0,
    estimated_flops BIGINT DEFAULT 0,
    input_shape JSONB,
    output_shape JSONB,
    created_by UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(project_id, version)
);
CREATE INDEX arch_project_idx ON architectures(project_id);
CREATE INDEX arch_version_idx ON architectures(project_id, version DESC);

-- Datasets
CREATE TABLE datasets (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID REFERENCES projects(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    source TEXT NOT NULL,
    format TEXT NOT NULL,
    sample_count INTEGER DEFAULT 0,
    class_count INTEGER DEFAULT 0,
    storage_url TEXT,
    split_config JSONB DEFAULT '{}',
    augmentation_pipeline JSONB DEFAULT '[]',
    version INTEGER NOT NULL DEFAULT 1,
    is_builtin BOOLEAN DEFAULT FALSE,
    created_by UUID REFERENCES users(id) ON DELETE CASCADE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX datasets_project_idx ON datasets(project_id);
CREATE INDEX datasets_builtin_idx ON datasets(is_builtin) WHERE is_builtin = TRUE;

-- Training Runs
CREATE TABLE training_runs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    architecture_id UUID NOT NULL REFERENCES architectures(id) ON DELETE CASCADE,
    dataset_id UUID NOT NULL REFERENCES datasets(id) ON DELETE CASCADE,
    status TEXT NOT NULL DEFAULT 'queued',
    hyperparams JSONB NOT NULL,
    gpu_type TEXT NOT NULL DEFAULT 't4',
    gpu_count INTEGER NOT NULL DEFAULT 1,
    metrics_final JSONB,
    training_curves_url TEXT,
    checkpoint_urls JSONB DEFAULT '[]',
    best_checkpoint_url TEXT,
    cost_usd NUMERIC(10, 4),
    gpu_hours NUMERIC(10, 4),
    error_message TEXT,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_by UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    tags TEXT[] DEFAULT '{}'
);
CREATE INDEX runs_project_idx ON training_runs(project_id);
CREATE INDEX runs_arch_idx ON training_runs(architecture_id);
CREATE INDEX runs_dataset_idx ON training_runs(dataset_id);
CREATE INDEX runs_status_idx ON training_runs(status);
CREATE INDEX runs_created_idx ON training_runs(created_at DESC);

-- Training Jobs
CREATE TABLE training_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    training_run_id UUID NOT NULL REFERENCES training_runs(id) ON DELETE CASCADE,
    celery_task_id TEXT NOT NULL,
    worker_id TEXT,
    progress_pct REAL DEFAULT 0.0,
    current_epoch INTEGER DEFAULT 0,
    total_epochs INTEGER,
    current_batch INTEGER DEFAULT 0,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX jobs_run_idx ON training_jobs(training_run_id);
CREATE INDEX jobs_task_idx ON training_jobs(celery_task_id);

-- Exported Models
CREATE TABLE exported_models (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    training_run_id UUID NOT NULL REFERENCES training_runs(id) ON DELETE CASCADE,
    format TEXT NOT NULL,
    quantization TEXT DEFAULT 'none',
    file_url TEXT NOT NULL,
    file_size_bytes BIGINT,
    validation_max_diff REAL,
    validation_samples INTEGER DEFAULT 100,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX exports_run_idx ON exported_models(training_run_id);

-- Comments
CREATE TABLE comments (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    author_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    body TEXT NOT NULL,
    anchor_type TEXT NOT NULL,
    anchor_id UUID NOT NULL,
    resolved BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX comments_project_idx ON comments(project_id);
CREATE INDEX comments_anchor_idx ON comments(anchor_type, anchor_id);

-- Usage Records
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE CASCADE,
    record_type TEXT NOT NULL,
    quantity NUMERIC(10, 4) NOT NULL,
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
    pub gpu_hours_used: f32,
    pub gpu_hours_quota: f32,
    pub billing_period_start: Option<NaiveDate>,
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
    pub task_type: String,
    pub visibility: String,
    pub forked_from: Option<Uuid>,
    pub thumbnail_url: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Architecture {
    pub id: Uuid,
    pub project_id: Uuid,
    pub version: i32,
    pub parent_version_id: Option<Uuid>,
    pub graph_data: serde_json::Value,
    pub message: String,
    pub parameter_count: i64,
    pub estimated_flops: i64,
    pub input_shape: Option<serde_json::Value>,
    pub output_shape: Option<serde_json::Value>,
    pub created_by: Uuid,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct TrainingRun {
    pub id: Uuid,
    pub project_id: Uuid,
    pub architecture_id: Uuid,
    pub dataset_id: Uuid,
    pub status: String,
    pub hyperparams: serde_json::Value,
    pub gpu_type: String,
    pub gpu_count: i32,
    pub metrics_final: Option<serde_json::Value>,
    pub training_curves_url: Option<String>,
    pub checkpoint_urls: serde_json::Value,
    pub best_checkpoint_url: Option<String>,
    pub cost_usd: Option<rust_decimal::Decimal>,
    pub gpu_hours: Option<rust_decimal::Decimal>,
    pub error_message: Option<String>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub created_by: Uuid,
    pub created_at: DateTime<Utc>,
    pub tags: Vec<String>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct GraphNode {
    pub id: String,
    pub r#type: String,
    pub config: serde_json::Value,
    pub position: Position,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct Position {
    pub x: f64,
    pub y: f64,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct GraphEdge {
    pub id: String,
    pub source: String,
    pub target: String,
    pub source_handle: Option<String>,
    pub target_handle: Option<String>,
    pub shape: Option<TensorShape>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct TensorShape {
    pub dims: Vec<Option<i64>>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct ArchitectureGraph {
    pub nodes: Vec<GraphNode>,
    pub edges: Vec<GraphEdge>,
}
```

---

## Graph Validation Engine Deep-Dive

### Tensor Shape Propagation Algorithm

WASM-compiled validator implements forward shape inference:

**Conv2D Example:**
```
Input: [B, 3, 224, 224]
Conv2D(in=3, out=64, kernel=7, stride=2, padding=3)

output_h = (224 + 2*3 - 7) / 2 + 1 = 112
output_w = (224 + 2*3 - 7) / 2 + 1 = 112

Output: [B, 64, 112, 112]
```

**LSTM Example:**
```
Input: [B, seq_len, 128]
LSTM(hidden=256, layers=2, bidirectional=True)

Output: [B, seq_len, 512]  (256 * 2 for bidirectional)
```

### WASM Compilation

```rust
// graph-validator-wasm/src/lib.rs

use wasm_bindgen::prelude::*;
use serde::{Deserialize, Serialize};
use std::collections::HashMap;

#[wasm_bindgen]
pub fn validate_graph(graph_json: &str) -> String {
    let graph: ArchitectureGraph = serde_json::from_str(graph_json).unwrap();
    let mut validator = GraphValidator::new(graph);
    let result = validator.validate();
    serde_json::to_string(&result).unwrap()
}

#[derive(Debug, Serialize)]
struct ValidationResult {
    valid: bool,
    errors: Vec<String>,
    shapes: HashMap<String, Vec<Option<i64>>>,
}

struct GraphValidator {
    graph: ArchitectureGraph,
    shapes: HashMap<String, Vec<Option<i64>>>,
}

impl GraphValidator {
    fn validate(&mut self) -> ValidationResult {
        let sorted_nodes = self.topological_sort().unwrap();
        let mut errors = Vec::new();

        for node_id in sorted_nodes {
            let node = self.graph.nodes.iter().find(|n| n.id == node_id).unwrap();
            let input_shapes = self.get_input_shapes(&node.id);

            match self.compute_output_shape(node, &input_shapes) {
                Ok(output_shape) => {
                    self.shapes.insert(node.id.clone(), output_shape);
                }
                Err(e) => {
                    errors.push(format!("Node {}: {}", node.id, e));
                }
            }
        }

        ValidationResult {
            valid: errors.is_empty(),
            errors,
            shapes: self.shapes.clone(),
        }
    }

    fn compute_output_shape(
        &self,
        node: &GraphNode,
        input_shapes: &[Vec<Option<i64>>],
    ) -> Result<Vec<Option<i64>>, String> {
        match node.r#type.as_str() {
            "Input" => {
                let config: InputConfig = serde_json::from_value(node.config.clone())?;
                Ok(config.shape)
            }
            "Conv2D" => {
                let input = &input_shapes[0];
                let config: Conv2DConfig = serde_json::from_value(node.config.clone())?;
                let h_out = ((input[2].unwrap() + 2*config.padding - config.kernel_size) / config.stride) + 1;
                let w_out = ((input[3].unwrap() + 2*config.padding - config.kernel_size) / config.stride) + 1;
                Ok(vec![input[0], Some(config.out_channels), Some(h_out), Some(w_out)])
            }
            "LSTM" => {
                let input = &input_shapes[0];
                let config: LSTMConfig = serde_json::from_value(node.config.clone())?;
                let hidden = if config.bidirectional { config.hidden_size * 2 } else { config.hidden_size };
                Ok(vec![input[0], input[1], Some(hidden)])
            }
            "Dense" => {
                let input = &input_shapes[0];
                let config: DenseConfig = serde_json::from_value(node.config.clone())?;
                Ok(vec![input[0], Some(config.units)])
            }
            _ => Err(format!("Unknown layer: {}", node.r#type)),
        }
    }

    fn topological_sort(&self) -> Result<Vec<String>, String> {
        // Kahn's algorithm
        let mut in_degree: HashMap<String, usize> = HashMap::new();
        let mut adj_list: HashMap<String, Vec<String>> = HashMap::new();

        for node in &self.graph.nodes {
            in_degree.insert(node.id.clone(), 0);
            adj_list.insert(node.id.clone(), Vec::new());
        }

        for edge in &self.graph.edges {
            *in_degree.get_mut(&edge.target).unwrap() += 1;
            adj_list.get_mut(&edge.source).unwrap().push(edge.target.clone());
        }

        let mut queue: Vec<String> = in_degree.iter()
            .filter(|(_, &deg)| deg == 0)
            .map(|(id, _)| id.clone())
            .collect();

        let mut sorted = Vec::new();

        while let Some(node_id) = queue.pop() {
            sorted.push(node_id.clone());
            for neighbor in &adj_list[&node_id] {
                let deg = in_degree.get_mut(neighbor).unwrap();
                *deg -= 1;
                if *deg == 0 {
                    queue.push(neighbor.clone());
                }
            }
        }

        if sorted.len() != self.graph.nodes.len() {
            return Err("Graph contains cycle".to_string());
        }

        Ok(sorted)
    }
}
```

---

## Solver/Core Architecture Deep-Dive

### PyTorch Code Generation Pipeline

```
Architecture Graph (JSONB) → Graph Validator → PyTorch Generator → nn.Module Code → exec() → Model Instance
```

**Key Steps:**

1. **Topological Sort**: Order nodes by dependencies (Kahn's algorithm)
2. **Layer Mapping**: Map visual nodes to PyTorch classes (`Conv2D` → `nn.Conv2d`)
3. **Forward Pass Generation**: Emit sequential `forward()` method with branching logic for residual/inception blocks
4. **Handle Special Cases**: Add operations (residual), Concat operations (inception), attention mechanisms

**Generated Code Example (ResNet Block):**

```python
class GeneratedModel(nn.Module):
    def __init__(self):
        super().__init__()
        self.conv1 = nn.Conv2d(64, 64, kernel_size=3, stride=1, padding=1)
        self.bn1 = nn.BatchNorm2d(64)
        self.relu1 = nn.ReLU()
        self.conv2 = nn.Conv2d(64, 64, kernel_size=3, stride=1, padding=1)
        self.bn2 = nn.BatchNorm2d(64)

    def forward(self, x):
        identity = x
        conv1_out = self.conv1(x)
        bn1_out = self.bn1(conv1_out)
        relu1_out = self.relu1(bn1_out)
        conv2_out = self.conv2(relu1_out)
        bn2_out = self.bn2(conv2_out)
        add_out = bn2_out + identity  # Residual connection
        return add_out
```

### Training Loop Architecture

```
GPU Worker (Celery) → Load Model → Load Dataset (S3) → Training Loop → Stream Progress (Redis) → Save Checkpoint (S3)
```

**Progress Streaming:**

Every 10 batches, worker publishes to `training_progress:{run_id}`:
```json
{
  "epoch": 5,
  "batch": 120,
  "progress_pct": 52.3,
  "batch_loss": 0.342,
  "learning_rate": 0.0008,
  "gpu_memory_mb": 8450
}
```

Rust API subscribes to Redis pub/sub → forwards via WebSocket → React client renders live chart.

---

## Architecture Deep-Dives

### 1. Training Orchestration API (Rust/Axum)

```rust
// src/api/handlers/training.rs

use axum::{extract::{Path, State, Json}, http::StatusCode, response::IntoResponse};
use uuid::Uuid;
use crate::{db::models::*, state::AppState, auth::Claims, error::ApiError};

#[derive(serde::Deserialize)]
pub struct CreateTrainingRequest {
    pub architecture_id: Uuid,
    pub dataset_id: Uuid,
    pub hyperparams: serde_json::Value,
    pub gpu_type: String,
    pub gpu_count: i32,
}

pub async fn create_training_run(
    State(state): State<AppState>,
    claims: Claims,
    Path(project_id): Path<Uuid>,
    Json(req): Json<CreateTrainingRequest>,
) -> Result<impl IntoResponse, ApiError> {
    // 1. Verify project ownership
    let project = sqlx::query_as!(
        Project,
        "SELECT * FROM projects WHERE id = $1 AND (owner_id = $2 OR org_id IN (
            SELECT org_id FROM org_members WHERE user_id = $2
        ))",
        project_id, claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Project not found"))?;

    // 2. Get architecture and dataset
    let architecture = sqlx::query_as!(Architecture,
        "SELECT * FROM architectures WHERE id = $1", req.architecture_id
    ).fetch_one(&state.db).await?;

    let dataset = sqlx::query_as!(Dataset,
        "SELECT * FROM datasets WHERE id = $1", req.dataset_id
    ).fetch_one(&state.db).await?;

    // 3. Check GPU quota
    let user = sqlx::query_as!(User, "SELECT * FROM users WHERE id = $1", claims.user_id)
        .fetch_one(&state.db).await?;

    let estimated_hours = estimate_gpu_hours(
        architecture.parameter_count,
        dataset.sample_count,
        req.hyperparams.get("epochs").unwrap().as_i64().unwrap() as i32,
        &req.gpu_type,
    );

    if user.gpu_hours_used + estimated_hours > user.gpu_hours_quota {
        return Err(ApiError::QuotaExceeded(
            format!("Need {:.2}h, have {:.2}h remaining",
                    estimated_hours, user.gpu_hours_quota - user.gpu_hours_used)
        ));
    }

    // 4. Create run record
    let run = sqlx::query_as!(
        TrainingRun,
        r#"INSERT INTO training_runs
            (project_id, architecture_id, dataset_id, hyperparams, gpu_type, gpu_count,
             created_by, status)
        VALUES ($1, $2, $3, $4, $5, $6, $7, 'queued')
        RETURNING *"#,
        project_id, req.architecture_id, req.dataset_id,
        req.hyperparams, req.gpu_type, req.gpu_count, claims.user_id,
    )
    .fetch_one(&state.db)
    .await?;

    // 5. Enqueue Celery task via Redis
    let task_payload = serde_json::json!({
        "run_id": run.id,
        "architecture": architecture.graph_data,
        "dataset_url": dataset.storage_url,
        "hyperparams": req.hyperparams,
        "gpu_type": req.gpu_type,
    });

    state.redis
        .lpush::<_, _, ()>("neuralforge:training_queue", serde_json::to_string(&task_payload)?)
        .await?;

    Ok((StatusCode::CREATED, Json(run)))
}

fn estimate_gpu_hours(param_count: i64, sample_count: i32, epochs: i32, gpu_type: &str) -> f32 {
    let base_time = (param_count as f64 / 1e6) * (sample_count as f64 / 1000.0) * 0.01;
    let multiplier = match gpu_type {
        "t4" => 1.0,
        "a10g" => 0.6,
        "a100" => 0.3,
        _ => 1.0,
    };
    ((base_time * epochs as f64 * multiplier) / 3600.0) as f32
}
```

### 2. PyTorch Model Generator (Python)

```python
# training-service/codegen/pytorch_generator.py

import networkx as nx
from jinja2 import Template

LAYER_TEMPLATES = {
    "conv2d": "nn.Conv2d({{in_channels}}, {{out_channels}}, kernel_size={{kernel_size}}, stride={{stride}}, padding={{padding}})",
    "lstm": "nn.LSTM({{input_size}}, {{hidden_size}}, num_layers={{num_layers}}, batch_first=True)",
    "dense": "nn.Linear({{in_features}}, {{out_features}})",
    "relu": "nn.ReLU()",
    "batchnorm2d": "nn.BatchNorm2d({{num_features}})",
}

def generate_pytorch_model(graph_data, input_shape):
    nodes = graph_data["nodes"]
    edges = graph_data["edges"]

    # Build graph
    G = nx.DiGraph()
    for node in nodes:
        G.add_node(node["id"], **node)
    for edge in edges:
        G.add_edge(edge["source"], edge["target"])

    topo_order = list(nx.topological_sort(G))

    # Generate layers
    layer_defs = []
    for node_id in topo_order:
        node = G.nodes[node_id]
        if node["type"] != "input":
            layer_code = render_layer(node["type"], node.get("config", {}))
            layer_defs.append(f"        self.{node_id} = {layer_code}")

    # Generate forward
    forward_stmts = []
    for node_id in topo_order:
        node = G.nodes[node_id]
        if node["type"] == "input":
            continue
        predecessors = list(G.predecessors(node_id))

        if node["type"] == "add":
            inputs = [get_var(p) for p in predecessors]
            forward_stmts.append(f"        {node_id} = {' + '.join(inputs)}")
        else:
            forward_stmts.append(f"        {node_id} = self.{node_id}({get_var(predecessors[0])})")

    output_node = next(n for n in nodes if n["type"] == "output")
    output_var = get_var(list(G.predecessors(output_node["id"]))[0])

    model_template = Template("""
import torch.nn as nn

class GeneratedModel(nn.Module):
    def __init__(self):
        super().__init__()
{{layer_definitions}}

    def forward(self, x):
{{forward_statements}}
        return {{output_var}}
""")

    return model_template.render(
        layer_definitions="\n".join(layer_defs),
        forward_statements="\n".join(forward_stmts),
        output_var=output_var
    )

def render_layer(layer_type, params):
    return Template(LAYER_TEMPLATES[layer_type]).render(**params)

def get_var(node_id):
    return "x" if node_id == "input" else node_id.replace("-", "_")
```

### 3. Training Worker (Python/Celery)

```python
# training-service/tasks/training.py

import torch
import torch.nn as nn
from torch.cuda.amp import autocast, GradScaler
from celery import Task
import redis

redis_client = redis.from_url(os.environ['REDIS_URL'])

@celery_app.task(bind=True)
def train_model_task(self, run_id, architecture, dataset_url, hyperparams, gpu_type):
    device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')

    # Update status
    update_run_status(run_id, 'training', started_at=datetime.utcnow())

    # Build model
    model_code = generate_pytorch_model(architecture, input_shape=[None, 3, 224, 224])
    exec_globals = {}
    exec(model_code, exec_globals)
    model = exec_globals['GeneratedModel']().to(device)

    # Load dataset
    train_loader = create_dataloader(dataset_url, split='train', batch_size=hyperparams['batch_size'])
    val_loader = create_dataloader(dataset_url, split='val', batch_size=hyperparams['batch_size'])

    # Setup training
    optimizer = torch.optim.Adam(model.parameters(), lr=hyperparams['learning_rate'])
    criterion = nn.CrossEntropyLoss()
    scaler = GradScaler(enabled=hyperparams.get('mixed_precision', True))

    # Training loop
    best_val_loss = float('inf')
    for epoch in range(hyperparams['epochs']):
        model.train()
        epoch_loss = 0.0

        for batch_idx, (inputs, targets) in enumerate(train_loader):
            inputs, targets = inputs.to(device), targets.to(device)
            optimizer.zero_grad()

            with autocast(enabled=hyperparams.get('mixed_precision', True)):
                outputs = model(inputs)
                loss = criterion(outputs, targets)

            scaler.scale(loss).backward()
            scaler.step(optimizer)
            scaler.update()

            epoch_loss += loss.item()

            # Stream progress
            if batch_idx % 10 == 0:
                progress_pct = ((epoch * len(train_loader) + batch_idx) /
                               (hyperparams['epochs'] * len(train_loader))) * 100
                redis_client.publish(f'training_progress:{run_id}', json.dumps({
                    'epoch': epoch,
                    'batch': batch_idx,
                    'loss': loss.item(),
                    'progress_pct': progress_pct,
                }))

        train_loss = epoch_loss / len(train_loader)
        val_loss = validate(model, val_loader, criterion, device)

        # Save best checkpoint
        if val_loss < best_val_loss:
            best_val_loss = val_loss
            checkpoint_url = save_checkpoint(model, epoch, run_id)
            update_best_checkpoint(run_id, checkpoint_url)

    # Finalize
    update_run_status(run_id, 'completed',
                     metrics_final={'train_loss': train_loss, 'val_loss': val_loss},
                     gpu_hours=calculate_gpu_hours(run_id))

def validate(model, loader, criterion, device):
    model.eval()
    total_loss = 0.0
    with torch.no_grad():
        for inputs, targets in loader:
            inputs, targets = inputs.to(device), targets.to(device)
            outputs = model(inputs)
            total_loss += criterion(outputs, targets).item()
    return total_loss / len(loader)
```

### 4. WebSocket Progress Streaming (Rust/Axum + React)

Real-time training metrics streamed from Python workers through Redis pub/sub to Rust API server, then forwarded via WebSocket to React clients for live chart updates.

```rust
// src/api/ws/training_progress.rs

use axum::{
    extract::{ws::{WebSocket, WebSocketUpgrade, Message}, State, Path},
    response::IntoResponse,
};
use futures::{sink::SinkExt, stream::StreamExt};
use uuid::Uuid;
use crate::state::AppState;

pub async fn training_progress_ws(
    ws: WebSocketUpgrade,
    Path(run_id): Path<Uuid>,
    State(state): State<AppState>,
) -> impl IntoResponse {
    ws.on_upgrade(move |socket| handle_socket(socket, run_id, state))
}

async fn handle_socket(socket: WebSocket, run_id: Uuid, state: AppState) {
    let (mut sender, mut receiver) = socket.split();

    // Subscribe to Redis pub/sub channel
    let mut pubsub_conn = state.redis.get_async_connection().await.unwrap();
    let mut pubsub = pubsub_conn.into_pubsub();
    pubsub.subscribe(format!("training_progress:{}", run_id)).await.unwrap();

    let mut pubsub_stream = pubsub.on_message();

    // Forward messages from Redis to WebSocket
    tokio::spawn(async move {
        while let Some(msg) = pubsub_stream.next().await {
            let payload: String = msg.get_payload().unwrap();

            if sender.send(Message::Text(payload)).await.is_err() {
                break;
            }
        }
    });

    // Handle client messages (e.g., stop training)
    while let Some(Ok(msg)) = receiver.next().await {
        if let Message::Text(text) = msg {
            if text == "stop" {
                // Publish stop signal to training worker
                let _: () = state.redis
                    .publish(format!("training_control:{}", run_id), "stop")
                    .await
                    .unwrap();
            }
        }
    }
}
```

**React Hook for WebSocket:**

```typescript
// frontend/src/hooks/useTrainingProgress.ts

import { useEffect, useState } from 'react';

interface TrainingProgress {
  epoch: number;
  batch: number;
  progress_pct: number;
  batch_loss: number;
  learning_rate: number;
  gpu_memory_mb?: number;
}

export function useTrainingProgress(runId: string) {
  const [progress, setProgress] = useState<TrainingProgress | null>(null);
  const [connected, setConnected] = useState(false);

  useEffect(() => {
    const ws = new WebSocket(`wss://api.neuralforge.ai/ws/training/${runId}`);

    ws.onopen = () => {
      setConnected(true);
    };

    ws.onmessage = (event) => {
      const data = JSON.parse(event.data);
      setProgress(data);
    };

    ws.onerror = (error) => {
      console.error('WebSocket error:', error);
      setConnected(false);
    };

    ws.onclose = () => {
      setConnected(false);
    };

    return () => {
      ws.close();
    };
  }, [runId]);

  const stopTraining = () => {
    // Send stop command via WebSocket
    const ws = new WebSocket(`wss://api.neuralforge.ai/ws/training/${runId}`);
    ws.onopen = () => {
      ws.send('stop');
      ws.close();
    };
  };

  return { progress, connected, stopTraining };
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–5)

**Day 1: Project initialization and infrastructure**
- Initialize Rust workspace with Axum, SQLx, Tokio dependencies
- Create `docker-compose.yml` with PostgreSQL 16, Redis 7, MinIO (S3-compatible)
- Set up `config.rs` for environment-based configuration (DATABASE_URL, REDIS_URL, S3_BUCKET, JWT_SECRET)
- Create `main.rs` with basic Axum server, health check endpoint, CORS middleware
- Set up `state.rs` with AppState struct (PgPool, Redis client, S3 client)
- Create `error.rs` with ApiError enum and IntoResponse implementation
- Write Dockerfile for Rust API (multi-stage build with cargo-chef for layer caching)

**Day 2: Database schema and migrations**
- Install SQLx CLI: `cargo install sqlx-cli`
- Create `migrations/001_initial.sql` with all 10 tables (users, orgs, org_members, projects, architectures, datasets, training_runs, training_jobs, exported_models, comments, usage_records)
- Add indexes for common queries (email, project owner, run status, created_at)
- Create `src/db/models.rs` with all SQLx FromRow structs (User, Project, Architecture, TrainingRun, etc.)
- Run migrations: `sqlx migrate run`
- Write seed script `scripts/seed_datasets.sql` for builtin datasets (MNIST, CIFAR-10, CIFAR-100)
- Verify schema with `sqlx database reset && sqlx migrate run`

**Day 3: Authentication system**
- Implement JWT token generation in `src/auth/jwt.rs` with Claims struct (user_id, email, exp)
- Create bcrypt password hashing utilities (cost factor 12)
- Implement OAuth 2.0 flow handlers in `src/auth/oauth.rs` for Google and GitHub
- Write auth middleware `src/auth/middleware.rs` that extracts Claims from Authorization header
- Create auth routes: `POST /api/auth/register`, `POST /api/auth/login`, `POST /api/auth/refresh`
- Implement OAuth callback endpoints: `GET /api/auth/oauth/google/callback`, `GET /api/auth/oauth/github/callback`
- Add `GET /api/auth/me` endpoint to fetch current user
- Write integration tests for auth flow (register → login → JWT validation)

**Day 4: User and organization CRUD**
- Create `src/api/handlers/users.rs` with get profile, update profile, delete account endpoints
- Implement role-based access control enum (Owner, Admin, Member, Viewer)
- Create `src/api/handlers/orgs.rs` with create org, list members, invite member, remove member endpoints
- Add org membership verification middleware
- Write tests for RBAC (verify member can't delete org, viewer can't modify)
- Implement Stripe customer creation hook on user registration
- Add user plan limits checking utilities

**Day 5: Project CRUD and collaboration**
- Create `src/api/handlers/projects.rs` with full CRUD (create, list, get, update, delete)
- Implement visibility controls (private, org, public) with authorization checks
- Add project fork endpoint that deep-copies project + latest architecture
- Create share link generation with read-only tokens
- Implement project search and filtering (by owner, org, task type, updated date)
- Write integration tests for project CRUD and authorization edge cases
- Add project thumbnail generation placeholder (to be implemented later)

### Phase 2 — WASM Validator + Shape Inference (Days 6–11)

**Day 6: Graph data structures and validation**
- Create Rust workspace member `graph-validator/` shared between API and WASM
- Define graph structures in `graph-validator/src/graph.rs`: GraphNode, GraphEdge, TensorShape, ArchitectureGraph
- Implement topological sort using Kahn's algorithm in `graph-validator/src/topo.rs`
- Add cycle detection and report cyclic edges with helpful error messages
- Create DAG validator that checks for floating nodes, missing input/output nodes
- Write comprehensive unit tests for graph validation (valid graphs, cycles, disconnected components)

**Day 7: Shape inference engine**
- Create `graph-validator/src/shape_inference.rs` with ShapeInferenceEngine
- Implement shape propagation for 30+ layer types:
  - Convolution: Conv1D, Conv2D, Conv3D, DepthwiseConv2D, SeparableConv2D
  - Recurrent: LSTM, GRU, BiLSTM, BiGRU
  - Attention: MultiHeadAttention, SelfAttention, CrossAttention
  - Normalization: BatchNorm1D/2D/3D, LayerNorm, GroupNorm, InstanceNorm
  - Pooling: MaxPool1D/2D/3D, AvgPool1D/2D/3D, AdaptiveAvgPool, GlobalAvgPool
  - Activation: ReLU, GELU, Swish, Mish, LeakyReLU, PReLU, Tanh, Sigmoid
  - Linear: Dense/Linear, Embedding, Flatten, Reshape
  - Dropout: Dropout, Dropout2D, AlphaDropout
  - Special: Concatenate, Add, Multiply, Lambda
- Add helpful error messages for shape mismatches ("Conv2D kernel_size=5 exceeds input height 3")
- Write unit tests for each layer type with valid and invalid inputs

**Day 8: WASM compilation and browser integration**
- Create `graph-validator-wasm/` workspace member with wasm-bindgen
- Add `Cargo.toml` with dependencies: wasm-bindgen, serde-wasm-bindgen, console_log
- Create `graph-validator-wasm/src/lib.rs` with `#[wasm_bindgen]` entry point `validate_graph()`
- Set up release profile for size optimization (opt-level="z", lto=true, codegen-units=1)
- Create GitHub Actions workflow `.github/workflows/wasm-build.yml` for automated builds
- Build with `wasm-pack build --target web --release`
- Optimize with `wasm-opt -Oz` to reduce bundle size to <300KB gzipped
- Upload to S3/CDN with versioned URLs for cache busting

**Day 9: Parameter counting and FLOPs estimation**
- Implement parameter counter in `graph-validator/src/params.rs` that sums trainable params across all layers
- Add FLOPs estimator using standard formulas:
  - Conv2D: `FLOPs = 2 * input_h * input_w * kernel_h * kernel_w * in_channels * out_channels`
  - LSTM: `FLOPs = 8 * hidden_size^2 * seq_len` (4 gates, 2 ops per multiply-add)
  - Dense: `FLOPs = 2 * in_features * out_features`
- Create human-readable formatters (1.5M params, 3.2 GFLOPs)
- Add memory estimation for forward + backward pass
- Enhance validation error messages with fix suggestions

**Day 10: React frontend scaffold**
- Initialize Vite project: `npm create vite@latest frontend -- --template react-ts`
- Install dependencies: `npm i zustand @tanstack/react-query axios react-flow-renderer recharts`
- Set up `src/stores/authStore.ts` with JWT token management and auto-refresh
- Create `src/stores/projectStore.ts` for current project state
- Create `src/stores/architectureStore.ts` for graph nodes/edges, undo/redo stack
- Set up React Query with axios client in `src/lib/api.ts`
- Create basic React Flow canvas in `src/components/ArchitectureCanvas.tsx`
- Add layer palette sidebar with categorized layer types (Convolution, Recurrent, Attention, etc.)

**Day 11: TypeScript shape inference and real-time validation**
- Create `src/lib/wasmLoader.ts` to load WASM validator module from CDN
- Implement TypeScript shape inference that mirrors Rust logic for offline validation
- Add debounced validation trigger (300ms after graph change)
- Annotate edges with computed tensor shapes as labels: `[B, 64, 32, 32]`
- Highlight invalid edges in red with error tooltips
- Display total parameters and FLOPs in canvas toolbar
- Add visual feedback for validation state (spinner → checkmark / error icon)

### Phase 3 — Dataset Management (Days 12–15)

**Day 12: Dataset upload and S3 integration**
- Create `src/api/handlers/datasets.rs` with upload, list, get, delete endpoints
- Implement S3 multipart upload with presigned URLs for direct browser → S3 upload
- Add format detection based on file extension and magic bytes (CSV, images, zip, tar.gz)
- Extract metadata: sample count (by counting files/rows), file sizes, class distribution
- Create `src/services/s3.rs` with helpers for presigned URLs, bucket operations
- Add upload progress tracking via chunked multipart upload
- Write tests for upload flow with mock S3 (localstack)

**Day 13: Builtin dataset catalog**
- Create dataset seed script with metadata for MNIST, CIFAR-10, CIFAR-100, ImageNet subset
- Download and upload builtin datasets to S3 (or reference public URLs)
- Create dataset preview generator that samples 100 random images → thumbnail grid
- Build gallery UI in `src/components/DatasetExplorer/Gallery.tsx` with pagination
- Add class distribution bar chart showing sample counts per class
- Implement dataset search and filtering (by name, format, source)
- Add "Use this dataset" button that links dataset to current project

**Day 14: Augmentation pipeline builder**
- Define augmentation operation JSON schema: `{type: "flip", params: {horizontal: true}}`
- Implement augmentation types: RandomFlip, RandomRotate, RandomCrop, ColorJitter, Normalize, Resize
- Create visual pipeline builder UI with drag-and-drop operation nodes
- Add live preview showing original vs. augmented samples side-by-side
- Store augmentation pipeline in dataset record as JSONB array
- Implement pipeline executor in Python training service using torchvision.transforms
- Write tests for augmentation JSON → PyTorch transforms conversion

**Day 15: Dataset splits and versioning**
- Implement train/val/test split configuration with ratio sliders (70/15/15 default)
- Add stratification option that preserves class distribution across splits
- Create dataset version incrementer that creates new version on split/augmentation changes
- Implement corrupt file detection that scans uploaded datasets for invalid images
- Add dataset statistics calculator: mean, std, class imbalance warnings
- Create dataset lineage view showing version history
- Write tests for split generation with stratification

### Phase 4 — Training Pipeline (Days 16–23)

**Day 16: Celery and Kubernetes worker setup**
- Create Python training service: `mkdir training-service && cd training-service`
- Initialize `requirements.txt`: torch, torchvision, celery, redis, boto3, sqlalchemy, psycopg2
- Create `celery_app.py` with Celery configuration (Redis broker, result backend)
- Define task queues: `gpu-t4`, `gpu-a10g`, `gpu-a100` with routing
- Create Kubernetes manifests in `k8s/celery-worker-t4.yaml` with GPU node affinity
- Add KEDA ScaledObject that scales workers based on Redis queue depth
- Write worker health check that reports GPU availability
- Deploy and test worker scaling (queue 10 jobs → workers scale from 0 to 3)

**Day 17: Training run creation endpoint**
- Implement `POST /api/projects/:id/training/runs` in Rust API
- Add architecture and dataset validation (existence checks, ownership)
- Implement GPU quota checking against user/org limits
- Create cost estimator function that predicts GPU hours and cost
- Add plan-based restrictions (Free: T4 only, max $1 per run)
- Enqueue Celery task by pushing to Redis list: `lpush neuralforge:training_queue`
- Return training run ID and estimated cost to client
- Write integration test: create run → verify DB record → verify Redis queue entry

**Day 18: PyTorch code generator**
- Create `training-service/codegen/pytorch_generator.py` with graph → code translator
- Implement Jinja2 templates for each layer type mapping to `nn.Module`
- Generate `__init__()` method with layer instantiations in topological order
- Generate `forward()` method handling:
  - Sequential: `x = self.layer1(x); x = self.layer2(x)`
  - Residual: `identity = x; x = self.block(x); x = x + identity`
  - Inception: `branch1 = ...; branch2 = ...; x = torch.cat([branch1, branch2], dim=1)`
- Add model summary printer (parameter count, layer-by-layer shapes)
- Write tests comparing generated model output shapes to expected shapes

**Day 19: Training task implementation**
- Create `training-service/tasks/training.py` with `@celery_app.task` decorator
- Implement model instantiation: `exec()` generated PyTorch code into isolated namespace
- Create S3-backed PyTorch Dataset class that streams data chunks from S3
- Build DataLoader with num_workers, pin_memory, prefetch_factor optimizations
- Implement basic training loop with loss.backward(), optimizer.step()
- Add epoch/batch progress logging to database
- Write test with toy dataset (100 samples) and simple 2-layer network

**Day 20: Advanced training features**
- Implement automatic mixed precision with `torch.cuda.amp.GradScaler`
- Add gradient clipping with `torch.nn.utils.clip_grad_norm_()`
- Implement `nn.DataParallel` for multi-GPU training (gpu_count > 1)
- Add gradient accumulation for large batch sizes on limited memory
- Implement learning rate warmup (linear ramp for first N steps)
- Add early stopping with patience parameter (stop if val_loss doesn't improve for K epochs)
- Write tests for AMP (verify loss scales correctly), gradient clipping (verify norm capped)

**Day 21: WebSocket progress streaming**
- Create `src/api/ws/training_progress.rs` with WebSocket upgrade handler
- Subscribe to Redis pub/sub channel `training_progress:{run_id}`
- Forward messages from Redis to WebSocket clients
- Implement client heartbeat/ping-pong to detect disconnections
- Add WebSocket reconnection logic with exponential backoff
- Create React hook `useTrainingProgress()` that manages WebSocket connection
- Test with multiple concurrent WebSocket clients (load test with 100 connections)

**Day 22: Checkpoint management**
- Implement best checkpoint tracking based on validation metric (val_loss, val_accuracy)
- Save checkpoints to `/tmp/checkpoint_{run_id}_epoch_{epoch}.pt`
- Upload checkpoints to S3 with key `checkpoints/{run_id}/epoch_{epoch}.pt`
- Store checkpoint URLs in training_runs.checkpoint_urls JSONB array
- Update training_runs.best_checkpoint_url when new best is found
- Add checkpoint download endpoint with presigned S3 URL generation
- Implement checkpoint cleanup (delete old checkpoints, keep last 3 + best)

**Day 23: Training dashboard UI**
- Create `src/components/TrainingDashboard/Dashboard.tsx` with split layout
- Add live loss/accuracy curves using Recharts with streaming data append
- Display current epoch, batch, progress percentage, ETA
- Show GPU utilization and memory usage (if reported by worker)
- Add learning rate schedule visualization (line chart overlaid on loss curve)
- Implement stop training button that sends "stop" message via WebSocket
- Add checkpoint list with download buttons
- Create training log console with syntax highlighting for errors/warnings

### Phase 5 — Experiment Tracking (Days 24–28)

**Day 24: Experiment metadata logging**
- Enhance training task to log full experiment metadata on run creation:
  - Architecture snapshot (save architecture.graph_data to run record)
  - Hyperparameters (optimizer, lr, batch_size, epochs, scheduler, etc.)
  - Dataset version and preprocessing pipeline
  - Random seeds (Python, NumPy, PyTorch) for reproducibility
  - Hardware configuration (GPU type, count, CUDA version, PyTorch version)
- Add automatic tagging based on architecture patterns (ResNet, U-Net, etc.)
- Store git commit hash if training code is in repo
- Write query to reproduce run: fetch all metadata and reconstruct exact training config

**Day 25: Training curves storage and retrieval**
- Create metrics history JSON format: `[{epoch, batch, train_loss, train_acc, val_loss, val_acc, lr}]`
- Upload metrics to S3 as `curves/{run_id}/metrics.json`
- Create `GET /api/training-runs/:id/curves` endpoint that returns presigned S3 URL
- Implement multi-run curve overlay: fetch curves for N runs, render on single chart with legend
- Add curve smoothing option (moving average with configurable window)
- Create curve export to CSV for external analysis
- Write test: upload curves → retrieve → verify data integrity

**Day 26: Experiment comparison**
- Create `src/components/ExperimentTracker/ComparisonTable.tsx` with sortable columns
- Add columns: Run ID, architecture version, hyperparams (lr, batch_size), final metrics, GPU cost, duration
- Highlight best values per column (green background for highest accuracy, lowest loss)
- Implement multi-select rows for comparison actions (delete, tag, export)
- Create architecture diff view: show layers added/removed/modified between two runs
- Add hyperparameter diff highlighting changes between runs
- Write tests for comparison queries (fetch top 10 by accuracy, group by architecture)

**Day 27: Model evaluation and metrics**
- Create `POST /api/training-runs/:id/evaluate` endpoint that triggers evaluation Celery task
- Implement evaluation task: load best checkpoint, run inference on test set
- Calculate comprehensive metrics:
  - Classification: accuracy, precision, recall, F1, AUC-ROC per class
  - Segmentation: Dice, IoU, pixel accuracy
  - Regression: MSE, MAE, R²
- Generate confusion matrix and store as JSON
- Create per-class performance breakdown table
- Build evaluation UI with confusion matrix heatmap (Recharts)
- Add misclassified examples viewer with ground truth vs. predicted labels

**Day 28: Tagging and filtering**
- Add tag management endpoints: `POST /api/training-runs/:id/tags`, `DELETE /api/training-runs/:id/tags/:tag`
- Create filter panel with multi-select dropdowns:
  - Status (queued, training, completed, failed)
  - Tags (custom tags + auto-generated tags)
  - Date range (created_at, completed_at)
  - Architecture version
  - Dataset
- Implement hyperparameter search: find runs with lr in range [1e-4, 1e-3]
- Add saved filter presets (e.g., "Failed runs this week", "Best runs on CIFAR-10")
- Create export filtered runs to CSV for reporting
- Write tests for complex filter queries combining multiple conditions

### Phase 6 — Model Export (Days 29–33)

**Day 29: ONNX export implementation**
- Create `training-service/tasks/export.py` with ONNX export Celery task
- Load trained checkpoint and model architecture
- Run `torch.onnx.export()` with:
  - Dynamic axes for batch dimension
  - Opset version 17 for latest ONNX operators
  - Verbose mode to catch export warnings
- Handle export errors (unsupported operators, dynamic shapes) with fallback strategies
- Upload exported ONNX file to S3 `exports/{run_id}/model.onnx`
- Create `exported_models` table record with file URL and metadata
- Write test: train toy model → export → verify ONNX file is valid (load with onnxruntime)

**Day 30: Export validation pipeline**
- Implement validation task that compares original PyTorch model vs. ONNX exported model
- Run inference on 100 random test samples through both models
- Calculate max absolute difference: `max(abs(pytorch_out - onnx_out))`
- Calculate element-wise relative error
- Verify output shapes match exactly
- Store validation results in `exported_models.validation_max_diff`
- Fail export if max diff > threshold (1e-4 for FP32, 1e-3 for FP16)
- Create validation report UI showing sample-by-sample comparison

**Day 31: TorchScript export**
- Implement TorchScript export via `torch.jit.trace()` for models without control flow
- Fallback to `torch.jit.script()` for models with if/for statements
- Test exported TorchScript model: `model = torch.jit.load('model.pt')`
- Add TorchScript validation (run inference, compare outputs)
- Store `.pt` file in S3 with format='torchscript'
- Write test: export ResNet-18 via trace vs. script, verify both work

**Day 32: Export UI**
- Create `src/components/ModelExport/ExportPanel.tsx` with format selector radio buttons
- Add export button that triggers export task via API
- Display export status with progress indicator (queued → exporting → validating → completed)
- Show validation report with max diff, sample predictions, file size
- Add download button with presigned S3 URL
- Create export history table showing all exports for a run (format, timestamp, size, status)
- Add re-export button for failed exports

**Day 33: Model quantization**
- Implement dynamic quantization: `torch.quantization.quantize_dynamic(model, {nn.Linear}, dtype=torch.qint8)`
- Add FP16 quantization for ONNX: convert weights to FP16
- Create quantization comparison report:
  - Original FP32: 44.2 MB, 94.5% accuracy
  - FP16: 22.1 MB (50%), 94.4% accuracy (-0.1%)
  - INT8: 11.0 MB (25%), 93.8% accuracy (-0.7%)
- Add quantization UI with format + quantization dropdown (ONNX FP32, ONNX FP16, ONNX INT8)
- Run validation on quantized models (100 samples, compare to FP32)
- Write test: quantize ResNet-18 → verify size reduction → verify accuracy drop within tolerance

### Phase 7 — Templates + Collaboration (Days 34–37)

**Day 34: Architecture template system**
- Define template JSON format with placeholder parameters (e.g., `num_classes`, `input_channels`)
- Create builtin templates:
  - ResNet-18: 18-layer residual network for image classification
  - U-Net: 4-level encoder-decoder for segmentation
  - LSTM: 2-layer LSTM for sequence classification
  - Transformer Encoder: 6-layer encoder with multi-head attention
- Implement template instantiation API: `POST /api/templates/:id/instantiate` with params
- Create template gallery UI with preview thumbnails and parameter forms
- Add template customization: instantiate → user edits graph → save as new architecture
- Write tests: instantiate ResNet-18 with num_classes=10 → verify graph structure

**Day 35: Project forking**
- Implement deep project fork: `POST /api/projects/:id/fork`
- Copy project metadata, latest architecture version, and dataset links
- Set `forked_from` field to original project ID
- Increment fork counter on original project
- Create fork attribution UI: "Forked from @username/project-name"
- Add fork tree visualization showing fork relationships
- Write test: fork project → modify architecture → verify original unchanged

**Day 36: Comments and inline collaboration**
- Create `src/api/handlers/comments.rs` with CRUD endpoints
- Implement comment anchoring: attach comments to architecture nodes, training runs, or datasets
- Add comment threading (parent_comment_id for replies)
- Create inline comment UI in architecture editor: click node → "Add comment" → thread appears
- Add comment notifications (email or in-app)
- Implement resolve/unresolve for comment threads
- Write tests: create comment → fetch comments for node → resolve → verify resolved=true

**Day 37: Public sharing and gallery**
- Implement share link generation: `POST /api/projects/:id/share` returns read-only token
- Create public project view: `GET /public/:share_token` (no auth required)
- Add Open Graph meta tags for social previews (Twitter cards, LinkedIn)
- Create public gallery page listing all public projects with thumbnails
- Add view counter to projects
- Implement project privacy controls: private (default), org-only, public
- Write tests: generate share link → access without auth → verify read-only

### Phase 8 — Billing + Deployment (Days 38–42)

**Day 38: Stripe integration**
- Create `src/api/handlers/billing.rs` with Stripe Checkout Session creation
- Implement plan mapping: Free (default), Pro ($79/mo), Team ($249/mo per seat)
- Add Stripe webhook handler: `POST /api/webhooks/stripe` for subscription events
- Handle events: `checkout.session.completed`, `customer.subscription.updated`, `customer.subscription.deleted`
- Update user plan in database on successful subscription
- Create Customer Portal link generation: `POST /api/billing/portal` redirects to Stripe
- Write tests: mock webhook payloads → verify plan updates

**Day 39: GPU hour metering and quota enforcement**
- Implement GPU hour tracking: calculate duration between `started_at` and `completed_at`
- Multiply by `gpu_count` for multi-GPU runs
- Update `users.gpu_hours_used` on run completion
- Add quota enforcement in training run creation: reject if `gpu_hours_used + estimated_hours > gpu_hours_quota`
- Reset quota monthly via cron job: `UPDATE users SET gpu_hours_used = 0 WHERE billing_period_start < DATE_TRUNC('month', NOW())`
- Create usage dashboard UI showing current period consumption and quota
- Write tests: exceed quota → verify training blocked → upgrade plan → quota increased

**Day 40: Kubernetes deployment**
- Create K8s manifests in `k8s/`:
  - `api-deployment.yaml`: 3 replicas, HPA at 70% CPU, rolling update strategy
  - `celery-worker-t4.yaml`: GPU node affinity, KEDA scaling (queue depth > 5)
  - `celery-worker-a10g.yaml`: GPU node affinity, KEDA scaling
  - `postgres-statefulset.yaml`: Persistent volume, backup sidecar
  - `redis-deployment.yaml`: Master-replica setup
  - `ingress.yaml`: NGINX ingress with TLS from Let's Encrypt
- Create Helm chart for easier deployment
- Set up KEDA operator for autoscaling workers based on Redis queue length
- Configure Ingress with rate limiting (100 req/min per IP)
- Deploy to staging and run smoke tests

**Day 41: Integration testing and validation**
- Create end-to-end test suite:
  - Register user → create project → upload dataset → build architecture → train → export ONNX
- Add export validation tests: train ResNet-18 → export → validate → verify max_diff < 1e-4
- Implement load tests with k6:
  - API: 500 req/s for 5 min, verify p95 latency < 200ms
  - Concurrent training: 50 simultaneous runs, verify all complete successfully
  - WebSocket: 1000 concurrent WS connections, verify message delivery
- Run database stress test: 100K projects, 1M training runs, query performance
- Write Playwright UI tests for critical flows (login, architecture editing, training)

**Day 42: Monitoring, logging, and launch**
- Set up Prometheus metrics collection:
  - API: request duration histogram, error rate counter, active connections gauge
  - Training: run duration histogram, GPU utilization gauge, cost per run histogram
  - System: CPU, memory, disk, network
- Create Grafana dashboards:
  - System Health: API latency, error rate, database connections, Redis queue depth
  - Training: active runs, queued runs, GPU hours consumed, cost trends
  - Business: signups, MRR, conversion rate, churn
- Configure Sentry for error tracking with source maps
- Set up structured logging (JSON format) with log aggregation (CloudWatch/DataDog)
- Create runbook for common incidents (database failover, worker scaling, API deployment)
- Write user documentation: getting started guide, keyboard shortcuts, API reference
- Deploy to production, enable monitoring alerts, announce launch

---

## Critical Files

```
neuralforge/
├── api-rust/
│   ├── Cargo.toml
│   ├── src/
│   │   ├── main.rs, config.rs, state.rs, error.rs
│   │   ├── db/{models.rs, migrations/}
│   │   ├── auth/{jwt.rs, oauth.rs, middleware.rs}
│   │   ├── api/
│   │   │   ├── handlers/{auth.rs, projects.rs, architectures.rs,
│   │   │   │            datasets.rs, training.rs, experiments.rs,
│   │   │   │            export.rs, billing.rs}
│   │   │   └── ws/training_progress.rs
│   │   └── services/{s3.rs, redis.rs}
│   └── migrations/001_initial.sql
│
├── graph-validator-wasm/
│   ├── Cargo.toml
│   ├── src/
│   │   └── lib.rs  # Shape inference, topological sort
│   └── .github/workflows/wasm-build.yml
│
├── training-service/
│   ├── requirements.txt
│   ├── main.py  # FastAPI app
│   ├── celery_app.py
│   ├── codegen/
│   │   ├── pytorch_generator.py
│   │   └── shape_inference.py
│   ├── tasks/
│   │   ├── training.py
│   │   ├── evaluation.py
│   │   └── export.py
│   ├── datasets/
│   │   ├── loader.py
│   │   ├── augmentations.py
│   │   └── builtin.py
│   └── utils/{s3.py, metrics.py}
│
├── frontend/
│   ├── package.json, vite.config.ts
│   ├── src/
│   │   ├── App.tsx, main.tsx
│   │   ├── stores/{architectureStore.ts, projectStore.ts, authStore.ts}
│   │   ├── hooks/{useTrainingProgress.ts, useWasmValidator.ts}
│   │   ├── lib/{api.ts, wasmLoader.ts, shapeInference.ts}
│   │   ├── components/
│   │   │   ├── ArchitectureCanvas/
│   │   │   │   ├── Canvas.tsx
│   │   │   │   ├── LayerPalette.tsx
│   │   │   │   ├── LayerNode.tsx
│   │   │   │   └── PropertyPanel.tsx
│   │   │   ├── TrainingDashboard/
│   │   │   │   ├── Dashboard.tsx
│   │   │   │   ├── LiveMetrics.tsx
│   │   │   │   └── ProgressChart.tsx
│   │   │   ├── ExperimentTracker/
│   │   │   │   ├── ComparisonTable.tsx
│   │   │   │   ├── CurveOverlay.tsx
│   │   │   │   └── FilterPanel.tsx
│   │   │   └── DatasetExplorer/
│   │   │       ├── Gallery.tsx
│   │   │       └── AugmentationBuilder.tsx
│   │   └── pages/{Dashboard.tsx, Editor.tsx, Experiments.tsx}
│   └── public/wasm/  # WASM validator bundle
│
├── k8s/
│   ├── api-deployment.yaml
│   ├── celery-worker-t4.yaml
│   ├── celery-worker-a10g.yaml
│   ├── redis-deployment.yaml
│   ├── postgres-statefulset.yaml
│   ├── keda-scaledobject.yaml
│   └── ingress.yaml
│
└── docker-compose.yml
```

---

## Solver Validation Suite

### Benchmark 1: ResNet-18 on CIFAR-10
**Architecture:** ResNet-18 (11.7M params)
**Dataset:** CIFAR-10 (50K train, 10K test)
**Hyperparams:** Adam lr=1e-3, cosine scheduler, batch 128, 100 epochs, AMP
**GPU:** A10G
**Expected:** 18-22 min, **94.5-95.5% test accuracy**, 4.2-4.8GB GPU memory, **$0.45-0.55 cost**
**Validation:** Within 0.5% of official PyTorch ResNet-18 baseline

### Benchmark 2: U-Net Segmentation
**Architecture:** U-Net 4 levels (31M params)
**Dataset:** Synthetic tissue (5K images, 256×256)
**Hyperparams:** AdamW lr=1e-4, batch 16, 50 epochs, Dice loss
**GPU:** T4
**Expected:** 65-75 min, **Dice 0.88-0.92**, 8.5-10GB GPU, **$0.40-0.50 cost**
**Validation:** Dice within 0.02 of baseline

### Benchmark 3: LSTM Text Classification
**Architecture:** 2-layer LSTM (128 hidden, 0.5M params)
**Dataset:** IMDB (25K train)
**Hyperparams:** Adam lr=1e-3, batch 64, 10 epochs
**GPU:** T4
**Expected:** 12-15 min, **85-87% accuracy**, 1.8-2.2GB GPU, **$0.08-0.12 cost**
**Validation:** Within 1% of Keras LSTM baseline

### Benchmark 4: ONNX Export Validation
**Model:** ResNet-18 trained on CIFAR-10
**Export:** PyTorch → ONNX opset 17
**Validation:** 1000 test images
**Expected:** Max abs diff **<1e-5**, 100% class match, **44-46MB file**, 8-12ms CPU latency
**Validation:** Zero prediction mismatches

### Benchmark 5: FP16 Quantization
**Model:** U-Net trained on tissue data
**Export:** ONNX FP16
**Validation:** 100 test images
**Expected:** Max diff **<1e-3**, Dice degradation **<0.01**, **60-62MB** (50% of FP32), 18-22ms GPU
**Validation:** Dice within 0.01 of FP32

---

## Verification

### Integration Testing Checklist

1. **Auth flow** — Register → login → JWT → authorized calls → refresh → logout
2. **Project CRUD** — Create → update → fork → visibility → delete
3. **Architecture editing** — Drag layers → connect → WASM validation → shapes annotated → save version
4. **Dataset upload** — Upload CSV → S3 → metadata extracted → preview gallery
5. **Training flow** — Select arch+dataset → configure hyperparams → quota check → enqueue → WS progress → completion
6. **Live metrics** — Training starts → WS messages stream → Recharts updates → stop button works
7. **Checkpoints** — Best checkpoint saved → download link → load checkpoint
8. **Experiment comparison** — Multiple runs → table sortable → overlay curves → diff view
9. **ONNX export** — Export trained model → validation report → max diff <1e-4 → download
10. **Templates** — Select ResNet-18 template → instantiated → editable → train
11. **Billing** — Free user hits quota → blocked → upgrade prompt → Stripe checkout → webhook → quota lifted
12. **Share link** — Create public link → visit unauthenticated → view-only architecture
13. **Comments** — Add comment on node → thread → resolve
14. **Concurrent training** — 5 users launch training → all succeed → GPU workers scale
15. **Error handling** — Invalid graph → validation errors → convergence failure → meaningful error message

### SQL Verification Queries

```sql
-- Training throughput and success rate
SELECT
    status,
    COUNT(*) as count,
    AVG(gpu_hours)::numeric(10,2) as avg_gpu_hours,
    AVG(cost_usd)::numeric(10,2) as avg_cost,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY gpu_hours)::numeric(10,2) as p95_gpu_hours
FROM training_runs
WHERE created_at >= NOW() - INTERVAL '7 days'
GROUP BY status;

-- GPU hour consumption by plan
SELECT
    u.plan,
    COUNT(DISTINCT u.id) as users,
    SUM(u.gpu_hours_used)::numeric(10,2) as total_gpu_hours,
    AVG(u.gpu_hours_used)::numeric(10,2) as avg_per_user
FROM users u
WHERE u.billing_period_start >= DATE_TRUNC('month', NOW())
GROUP BY u.plan
ORDER BY total_gpu_hours DESC;

-- Most popular datasets
SELECT
    d.name,
    d.source,
    COUNT(DISTINCT tr.id) as training_runs,
    COUNT(DISTINCT tr.project_id) as projects
FROM datasets d
JOIN training_runs tr ON tr.dataset_id = d.id
WHERE tr.created_at >= NOW() - INTERVAL '30 days'
GROUP BY d.id
ORDER BY training_runs DESC
LIMIT 20;

-- Architecture complexity distribution
SELECT
    CASE
        WHEN parameter_count < 1000000 THEN '<1M'
        WHEN parameter_count < 10000000 THEN '1M-10M'
        WHEN parameter_count < 100000000 THEN '10M-100M'
        ELSE '>100M'
    END as param_range,
    COUNT(*) as architectures,
    AVG(estimated_flops / 1e9)::numeric(10,2) as avg_gflops
FROM architectures
WHERE created_at >= NOW() - INTERVAL '30 days'
GROUP BY param_range
ORDER BY param_range;
```

### Performance Benchmarks

| Operation | Target | Method |
|-----------|--------|--------|
| WASM validation: 20-node graph | <10ms | Browser DevTools Performance |
| API: create training run | <150ms p95 | Prometheus histogram |
| Training start latency (GPU cold) | <45s | Worker logs timestamp |
| WebSocket progress latency | <100ms | Client-side timestamp diff |
| Training curves query (1000 runs) | <200ms p95 | PostgreSQL EXPLAIN ANALYZE |
| ONNX export: ResNet-18 | <30s | Task duration |
| Dataset preview (100 images) | <1s | S3 presigned URL generation |
| Architecture fork (deep copy) | <500ms | PostgreSQL + JSONB copy |
| Concurrent WS connections | 1000+ | Load test with k6 |

---

## Deployment Architecture

### Infrastructure

```
                    CloudFront CDN
                    ├── Frontend SPA
                    ├── WASM Validator
                    └── Static Assets
                          │
            ┌─────────────┴─────────────┐
            │     AWS ALB (HTTPS/WSS)   │
            └───────────┬───────────────┘
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
    ┌────────┐     ┌────────┐     ┌────────┐
    │ API    │     │ API    │     │ API    │
    │ (Rust) │     │ (Rust) │     │ (Rust) │
    │ Pod ×3 │     │ HPA    │     │        │
    └────┬───┘     └────┬───┘     └────┬───┘
         │              │              │
         └──────────────┼──────────────┘
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
    ┌────────┐     ┌────────┐     ┌────────┐
    │ Postgres│    │ Redis  │     │   S3   │
    │ RDS     │    │ElastiC.│     │Datasets│
    │ Multi-AZ│    │        │     │Checkpts│
    └─────────┘    └────┬───┘     └────────┘
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
    ┌────────┐     ┌────────┐     ┌────────┐
    │Training│     │Training│     │Training│
    │Worker  │     │Worker  │     │Worker  │
    │(T4 GPU)│     │(T4 GPU)│     │(A10G)  │
    │KEDA×10 │     │        │     │KEDA×5  │
    └────────┘     └────────┘     └────────┘
```

### Scaling Strategy

- **API servers**: HPA at 70% CPU, min 3, max 10 replicas
- **Training workers**: KEDA ScaledObject on Redis queue depth (scale up when >5 jobs, scale down when idle 5min)
- **Worker instances**: g4dn.xlarge (T4, 4 vCPU, 16GB), g5.xlarge (A10G, 4 vCPU, 16GB)
- **Database**: RDS PostgreSQL r6g.large with read replica for experiment queries
- **Redis**: ElastiCache r6g.medium cluster (1 primary + 2 replicas)

### S3 Lifecycle Policy

```json
{
  "Rules": [
    {
      "ID": "checkpoints-lifecycle",
      "Prefix": "checkpoints/",
      "Status": "Enabled",
      "Transitions": [
        { "Days": 30, "StorageClass": "STANDARD_IA" },
        { "Days": 90, "StorageClass": "GLACIER" }
      ],
      "Expiration": { "Days": 365 }
    },
    {
      "ID": "datasets-keep",
      "Prefix": "datasets/",
      "Status": "Enabled",
      "Transitions": [
        { "Days": 90, "StorageClass": "STANDARD_IA" }
      ]
    }
  ]
}
```

---

## Post-MVP Roadmap

| Feature | Description | Priority |
|---------|-------------|----------|
| **Hyperparameter Sweeps** | Grid/random/Bayesian (Optuna) search over hyperparameters with parallel trial execution on multiple GPUs, early termination (median stopping), and interactive sweep dashboard showing parameter importance and Pareto frontiers | High |
| **Team Collaboration** | Real-time collaborative editing with CRDT-based architecture sync, cursor presence, architecture version diff viewer, inline commenting on nodes/edges, and activity feed with @mentions | High |
| **TFLite + CoreML Export** | TensorFlow Lite export with INT8 quantization and representative dataset calibration, CoreML export for iOS/macOS deployment with on-device latency profiling and automatic Xcode integration | High |
| **Neural Architecture Search** | DARTS differentiable search and ENAS weight-sharing search with configurable search spaces, hardware-aware NAS optimizing for target latency on mobile/edge devices, Pareto dashboard visualizing accuracy vs. FLOPs/latency tradeoffs | Medium |
| **HuggingFace Integration** | Import any HuggingFace Transformers model into visual editor, fine-tune with custom datasets, push trained models to HF Hub with model cards, and one-click deployment via HF Inference API | Medium |
| **Custom Layers + Code Injection** | Monaco-based Python editor for custom PyTorch layers with hot-reload, layer library for sharing custom layers across team, and bidirectional sync (import PyTorch code → visual graph, export graph → clean PyTorch code) | Medium |
| **Advanced Evaluation** | Grad-CAM and attention visualization for interpretability, embedding space t-SNE/UMAP projections, error analysis clustering, adversarial robustness testing, and model comparison reports with statistical significance tests | Low |
| **Enterprise Features** | SAML/SSO (Okta, Azure AD, Google Workspace), audit logging with tamper-proof event stream, private model hub with org-wide access controls, VPC deployment option for sensitive data, and on-premise GPU cluster integration | Low |

---

**MVP Timeline**: 42 days (8 phases) | **Team**: 1 full-stack + 1 backend/ML + 0.5 frontend | **Target**: 1400-1600 lines
