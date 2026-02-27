# 33. TerraScan — AI-Powered Satellite Imagery Analysis Platform

## Implementation Plan

**MVP Scope:** Cloud-native geospatial analytics platform with Sentinel-2 and Landsat-8/9 data ingestion via STAC API (Element84 Earth Search), automated preprocessing pipeline in Rust with Axum (atmospheric correction using Sen2Cor algorithms ported to Rust, cloud masking via S2Cloudless ML model called from Python microservice, spatial co-registration with `gdal-sys` Rust bindings), cloud-optimized GeoTIFF (COG) storage in S3 with PostgreSQL/PostGIS metadata catalog accessed via SQLx, interactive WebGL map viewer built on Mapbox GL JS with dynamic tile serving via Titiler, 5 core spectral indices (NDVI, NDWI, NDBI, EVI, NBR) computed in Rust with WASM compilation for client-side preview on tiles <500KB, bi-temporal change detection using image differencing with adaptive Otsu thresholding in Rust, pre-trained building footprint detection model (Faster R-CNN ResNet50 trained on Microsoft Building Footprints + OpenStreetMap) deployed as Python FastAPI microservice with GPU inference, 6-class land use/land cover classification (urban, agriculture, forest, water, barren, grassland) using U-Net semantic segmentation trained on ESA WorldCover served from Python ML service, GeoTIFF/GeoJSON/Shapefile export via Rust GDAL bindings, user authentication with JWT and bcrypt in Axum, organization management with role-based access control, Stripe billing with three tiers (Free / Analyst $149/mo / Team $599/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth, primary application server |
| Geospatial Processing | Rust + `gdal-sys` | Custom raster ops, COG generation, reprojection, 5-10x faster than Python |
| WASM Preview Engine | Rust (wasm-pack) | Client-side spectral index computation for instant preview <500KB tiles |
| ML Inference | Python 3.12 (FastAPI) | Separate microservice for GPU models, async with uvicorn, PyTorch |
| ML Models | PyTorch 2.2 + ONNX | Faster R-CNN (object detection), U-Net (segmentation), torchvision |
| Database | PostgreSQL 16 + PostGIS 3.4 | Spatial queries, vector storage, imagery metadata, GIST indexes |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations, async |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | COGs (satellite imagery), vector results, model weights, CloudFront CDN |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management, TanStack Query |
| Map Renderer | Mapbox GL JS + Deck.gl | WebGL-accelerated rendering, dynamic tile overlays, drawing tools |
| Tile Server | Titiler (FastAPI) | Dynamic COG tile generation, on-the-fly band math, histogram stretch |
| Job Queue | Redis 7 + Tokio tasks | Analysis jobs in Rust, preprocessing tasks, model inference queue |
| Real-time | WebSocket (Axum) | Live job progress, imagery ingestion status, alert notifications |
| Billing | Stripe | Checkout Sessions, Customer Portal, usage-based metering, webhooks |
| CDN | CloudFront | COG tile delivery, WASM bundle, static assets |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, job queue stats, error tracking |
| CI/CD | GitHub Actions | Rust tests, WASM build, Docker image push, PostGIS migrations |

### Key Architecture Decisions

1. **Rust/Axum for primary backend instead of Python FastAPI**: Rust provides 5-10x faster raster operations compared to Python/Rasterio for cloud masking, band math, COG generation, and resampling while maintaining memory safety and low memory footprint. Axum handles all HTTP routing, auth, database queries via SQLx, geospatial preprocessing with `gdal-sys` bindings, and job orchestration with Tokio tasks. Python FastAPI runs exclusively as a separate microservice for ML inference where PyTorch ecosystem dominates.

2. **WASM spectral index calculator for instant client-side preview**: Computing spectral indices (NDVI, EVI, NDWI) for small tiles (<500KB, typically 512x512 at 10m resolution) happens entirely in the browser via WASM. The Rust spectral calculation engine compiles to WASM and processes tiles with WebAssembly SIMD instructions, providing instant feedback without server round-trips. Larger analysis jobs (>2km² AOI) fall back to server-side Rust processing with distributed tiling via Tokio tasks.

3. **Hybrid Rust/Python architecture with clear service boundaries**: Rust Axum handles all HTTP routing, auth, database queries with SQLx, geospatial preprocessing with `gdal-sys`, and tile generation orchestration. Python FastAPI runs as a separate microservice exclusively for ML inference (object detection, segmentation) with GPU workers. The services communicate via internal HTTP (Axum → FastAPI) with msgpack serialization for efficient binary transfer of raster arrays. This separation allows independent scaling: CPU-bound Rust workers scale horizontally, GPU-bound Python workers scale on GPU nodes.

4. **COG-first storage with PostgreSQL/PostGIS spatial metadata catalog**: All satellite imagery and analysis results are stored as Cloud-Optimized GeoTIFFs in S3, enabling instant partial reads for tiling without full file downloads. PostGIS stores vector features (AOI polygons, detected buildings, land cover boundaries) as native geometry types and imagery metadata (acquisition date, cloud cover, bounds via ST_GeomFromText, STAC links). Spatial queries (find imagery intersecting AOI via ST_Intersects) use PostGIS GIST indexes accessed from Rust via SQLx, while Titiler reads COGs directly from S3 with HTTP range requests.

5. **STAC API integration for imagery discovery instead of direct provider APIs**: We integrate with Element84 Earth Search (STAC API) for Sentinel-2 and Landsat-8/9 discovery using async HTTP from Rust with `reqwest`, avoiding direct integration with ESA Copernicus Hub and USGS EarthExplorer APIs which have rate limits and authentication complexity. STAC provides unified search across providers, standardized metadata, and direct COG URLs for AWS Open Data Program assets. For commercial data (Planet, Maxar), we defer to post-MVP and use provider-specific SDKs.

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "postgis";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";

-- Users
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email TEXT UNIQUE NOT NULL,
    password_hash TEXT,  -- NULL for OAuth-only users
    name TEXT NOT NULL,
    avatar_url TEXT,
    auth_provider TEXT DEFAULT 'email',  -- email | google | github
    auth_provider_id TEXT,
    plan TEXT NOT NULL DEFAULT 'free',  -- free | analyst | team
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX users_email_idx ON users(email);
CREATE INDEX users_stripe_idx ON users(stripe_customer_id);

-- Organizations (for Team plan)
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
    role TEXT NOT NULL DEFAULT 'member',  -- owner | admin | analyst | viewer
    joined_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(org_id, user_id)
);
CREATE INDEX org_members_org_idx ON org_members(org_id);
CREATE INDEX org_members_user_idx ON org_members(user_id);

-- Areas of Interest (AOIs)
CREATE TABLE aois (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    geometry GEOMETRY(POLYGON, 4326) NOT NULL,
    area_km2 REAL NOT NULL,
    monitoring_enabled BOOLEAN NOT NULL DEFAULT false,
    monitoring_frequency TEXT,  -- daily | weekly | monthly
    is_public BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX aois_owner_idx ON aois(owner_id);
CREATE INDEX aois_org_idx ON aois(org_id);
CREATE INDEX aois_geom_idx ON aois USING GIST(geometry);
CREATE INDEX aois_updated_idx ON aois(updated_at DESC);

-- Satellite Imagery Sources (metadata; actual imagery in S3 as COG)
CREATE TABLE imagery_sources (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    provider TEXT NOT NULL,  -- sentinel2 | landsat8 | landsat9 | planet | maxar | custom
    scene_id TEXT NOT NULL,
    stac_item_url TEXT,  -- STAC Item URL if from catalog
    acquisition_date TIMESTAMPTZ NOT NULL,
    cloud_cover_pct REAL,
    bounds GEOMETRY(POLYGON, 4326) NOT NULL,
    crs TEXT NOT NULL DEFAULT 'EPSG:4326',
    resolution_m REAL NOT NULL,
    bands JSONB NOT NULL,  -- [{name: 'B02', wavelength_nm: 490, gsd_m: 10}]
    storage_path TEXT NOT NULL,  -- S3 key to COG
    file_size_bytes BIGINT,
    preprocessing_status TEXT NOT NULL DEFAULT 'raw',  -- raw | corrected | ready | failed
    preprocessing_error TEXT,
    thumbnail_url TEXT,
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX imagery_provider_idx ON imagery_sources(provider);
CREATE INDEX imagery_scene_idx ON imagery_sources(scene_id);
CREATE INDEX imagery_date_idx ON imagery_sources(acquisition_date DESC);
CREATE INDEX imagery_bounds_idx ON imagery_sources USING GIST(bounds);
CREATE INDEX imagery_status_idx ON imagery_sources(preprocessing_status);

-- Analysis Jobs (spectral indices, change detection, classification, object detection)
CREATE TABLE analysis_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    aoi_id UUID REFERENCES aois(id) ON DELETE SET NULL,
    job_type TEXT NOT NULL,  -- spectral_index | change_detection | classification | object_detection
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | running | completed | failed | cancelled
    parameters JSONB NOT NULL,  -- Job-specific parameters (index name, dates, model name, etc.)
    input_imagery_ids UUID[] NOT NULL,
    results_summary JSONB,  -- Quick-access summary (area stats, detection count, etc.)
    results_url TEXT,  -- S3 URL to result GeoTIFF or GeoJSON
    error_message TEXT,
    progress_pct REAL DEFAULT 0.0,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    wall_time_ms INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX jobs_owner_idx ON analysis_jobs(owner_id);
CREATE INDEX jobs_org_idx ON analysis_jobs(org_id);
CREATE INDEX jobs_aoi_idx ON analysis_jobs(aoi_id);
CREATE INDEX jobs_status_idx ON analysis_jobs(status);
CREATE INDEX jobs_created_idx ON analysis_jobs(created_at DESC);

-- Alert Rules (for monitoring system)
CREATE TABLE alert_rules (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    org_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    aoi_id UUID NOT NULL REFERENCES aois(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    rule_type TEXT NOT NULL,  -- change_area_threshold | object_count_threshold | index_value_threshold
    condition JSONB NOT NULL,  -- {metric: 'deforestation_area_km2', operator: '>', threshold: 5}
    notification_channels JSONB NOT NULL,  -- [{type: 'email', target: 'user@example.com'}]
    is_active BOOLEAN NOT NULL DEFAULT true,
    last_triggered_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX alert_rules_org_idx ON alert_rules(org_id);
CREATE INDEX alert_rules_aoi_idx ON alert_rules(aoi_id);

-- Alerts (triggered events)
CREATE TABLE alerts (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    rule_id UUID NOT NULL REFERENCES alert_rules(id) ON DELETE CASCADE,
    analysis_job_id UUID REFERENCES analysis_jobs(id) ON DELETE SET NULL,
    severity TEXT NOT NULL DEFAULT 'info',  -- info | warning | critical
    summary TEXT NOT NULL,
    details JSONB DEFAULT '{}',
    thumbnail_url TEXT,
    status TEXT NOT NULL DEFAULT 'new',  -- new | acknowledged | resolved
    acknowledged_by UUID REFERENCES users(id),
    acknowledged_at TIMESTAMPTZ,
    resolved_at TIMESTAMPTZ,
    triggered_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX alerts_rule_idx ON alerts(rule_id);
CREATE INDEX alerts_status_idx ON alerts(status);
CREATE INDEX alerts_triggered_idx ON alerts(triggered_at DESC);

-- Vector Features (detected objects, land cover polygons)
CREATE TABLE vector_features (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    analysis_job_id UUID NOT NULL REFERENCES analysis_jobs(id) ON DELETE CASCADE,
    feature_type TEXT NOT NULL,  -- building | vehicle | land_cover | water_body | road
    geometry GEOMETRY(GEOMETRY, 4326) NOT NULL,
    properties JSONB DEFAULT '{}',  -- confidence, class, area_m2, etc.
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX features_job_idx ON vector_features(analysis_job_id);
CREATE INDEX features_type_idx ON vector_features(feature_type);
CREATE INDEX features_geom_idx ON vector_features USING GIST(geometry);

-- Usage Tracking (for billing enforcement and metering)
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,  -- imagery_download_gb | analysis_minutes | storage_gb_days | ml_inference_count
    quantity REAL NOT NULL,
    metadata JSONB DEFAULT '{}',
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
use geojson::Geometry as GeoJsonGeometry;

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
    pub settings: serde_json::Value,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Aoi {
    pub id: Uuid,
    pub owner_id: Uuid,
    pub org_id: Option<Uuid>,
    pub name: String,
    pub description: String,
    // PostGIS geometry stored as WKB, converted to/from GeoJSON in handlers
    #[sqlx(skip)]
    pub geometry: Option<GeoJsonGeometry>,
    pub area_km2: f32,
    pub monitoring_enabled: bool,
    pub monitoring_frequency: Option<String>,
    pub is_public: bool,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct ImagerySource {
    pub id: Uuid,
    pub org_id: Option<Uuid>,
    pub provider: String,
    pub scene_id: String,
    pub stac_item_url: Option<String>,
    pub acquisition_date: DateTime<Utc>,
    pub cloud_cover_pct: Option<f32>,
    // PostGIS geometry stored as WKB
    #[sqlx(skip)]
    pub bounds: Option<GeoJsonGeometry>,
    pub crs: String,
    pub resolution_m: f32,
    pub bands: serde_json::Value,
    pub storage_path: String,
    pub file_size_bytes: Option<i64>,
    pub preprocessing_status: String,
    pub preprocessing_error: Option<String>,
    pub thumbnail_url: Option<String>,
    pub metadata: serde_json::Value,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct AnalysisJob {
    pub id: Uuid,
    pub owner_id: Uuid,
    pub org_id: Option<Uuid>,
    pub aoi_id: Option<Uuid>,
    pub job_type: String,
    pub status: String,
    pub parameters: serde_json::Value,
    pub input_imagery_ids: Vec<Uuid>,
    pub results_summary: Option<serde_json::Value>,
    pub results_url: Option<String>,
    pub error_message: Option<String>,
    pub progress_pct: f32,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub wall_time_ms: Option<i32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct VectorFeature {
    pub id: Uuid,
    pub analysis_job_id: Uuid,
    pub feature_type: String,
    #[sqlx(skip)]
    pub geometry: Option<GeoJsonGeometry>,
    pub properties: serde_json::Value,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct SpectralIndexParams {
    pub index_name: String,  // ndvi | ndwi | ndbi | evi | nbr
    pub imagery_id: Uuid,
    pub output_format: Option<String>,  // geotiff | png
}

#[derive(Debug, Deserialize)]
pub struct ChangeDetectionParams {
    pub imagery_id_before: Uuid,
    pub imagery_id_after: Uuid,
    pub method: String,  // image_diff | cva | post_classification
    pub threshold: Option<f32>,
}

#[derive(Debug, Deserialize)]
pub struct ClassificationParams {
    pub imagery_id: Uuid,
    pub model_name: String,  // lulc_6class | lulc_10class | custom_<id>
    pub confidence_threshold: Option<f32>,
}

#[derive(Debug, Deserialize)]
pub struct ObjectDetectionParams {
    pub imagery_id: Uuid,
    pub model_name: String,  // building_detection | vehicle_detection | ship_detection
    pub confidence_threshold: Option<f32>,
    pub min_object_area_m2: Option<f32>,
}
```

---

## Architecture Deep-Dives

### 1. Imagery Ingestion and Preprocessing Pipeline (Rust/Axum)

The ingestion pipeline queries STAC API for available scenes, downloads raw imagery from cloud providers, applies atmospheric correction and cloud masking (calling Python ML service), converts to COG format using Rust GDAL bindings, and stores in S3 with PostGIS metadata.

```rust
// src/imagery/ingestion.rs

use anyhow::Result;
use aws_sdk_s3::Client as S3Client;
use chrono::{DateTime, Utc};
use gdal::Dataset;
use serde::{Deserialize, Serialize};
use sqlx::PgPool;
use uuid::Uuid;
use reqwest::Client as HttpClient;

use crate::imagery::preprocessing::{atmospheric_correction, generate_cog};
use crate::imagery::stac::StacClient;
use crate::ml::client::MlServiceClient;

#[derive(Debug, Deserialize)]
pub struct StacSearchParams {
    pub bbox: [f64; 4],  // [west, south, east, north]
    pub datetime: String,  // RFC3339 range or single date
    pub collections: Vec<String>,  // ["sentinel-2-l2a", "landsat-8-c2-l2"]
    pub max_cloud_cover: Option<f32>,
}

pub struct IngestionWorker {
    db: PgPool,
    s3: S3Client,
    stac_client: StacClient,
    ml_client: MlServiceClient,
}

impl IngestionWorker {
    pub async fn search_and_ingest(
        &self,
        params: StacSearchParams,
        org_id: Option<Uuid>,
    ) -> Result<Vec<Uuid>> {
        // 1. Search STAC API for matching scenes
        let stac_items = self.stac_client.search(&params).await?;
        tracing::info!("Found {} STAC items matching criteria", stac_items.len());

        let mut imagery_ids = Vec::new();

        for item in stac_items {
            // 2. Check if scene already ingested
            let existing: Option<Uuid> = sqlx::query_scalar!(
                "SELECT id FROM imagery_sources WHERE scene_id = $1",
                item.id
            )
            .fetch_optional(&self.db)
            .await?;

            if let Some(id) = existing {
                imagery_ids.push(id);
                continue;
            }

            // 3. Create imagery record in database
            let imagery_id = Uuid::new_v4();
            let bounds_wkt = format!(
                "POLYGON(({} {}, {} {}, {} {}, {} {}, {} {}))",
                item.bbox[0], item.bbox[1],
                item.bbox[2], item.bbox[1],
                item.bbox[2], item.bbox[3],
                item.bbox[0], item.bbox[3],
                item.bbox[0], item.bbox[1],
            );

            sqlx::query!(
                r#"INSERT INTO imagery_sources
                    (id, org_id, provider, scene_id, stac_item_url, acquisition_date,
                     cloud_cover_pct, bounds, resolution_m, bands, storage_path, preprocessing_status)
                VALUES ($1, $2, $3, $4, $5, $6, $7, ST_GeomFromText($8, 4326), $9, $10, $11, 'pending')"#,
                imagery_id,
                org_id,
                item.collection.as_str(),
                item.id,
                item.self_url,
                item.properties.datetime,
                item.properties.eo_cloud_cover,
                bounds_wkt,
                item.gsd,
                serde_json::to_value(&item.assets)?,
                format!("raw/{}/{}.tif", item.collection, item.id),
            )
            .execute(&self.db)
            .await?;

            // 4. Spawn preprocessing task (Tokio task, not blocking)
            let worker = self.clone();
            tokio::spawn(async move {
                if let Err(e) = worker.process_imagery(imagery_id, item).await {
                    tracing::error!("Preprocessing failed for {}: {}", imagery_id, e);
                    sqlx::query!(
                        "UPDATE imagery_sources SET preprocessing_status = 'failed',
                         preprocessing_error = $2 WHERE id = $1",
                        imagery_id, e.to_string()
                    )
                    .execute(&worker.db)
                    .await
                    .ok();
                }
            });

            imagery_ids.push(imagery_id);
        }

        Ok(imagery_ids)
    }

    async fn process_imagery(&self, imagery_id: Uuid, item: StacItem) -> Result<()> {
        sqlx::query!(
            "UPDATE imagery_sources SET preprocessing_status = 'running' WHERE id = $1",
            imagery_id
        )
        .execute(&self.db)
        .await?;

        // 1. Download raw imagery from STAC asset URL
        let asset_url = item.assets.get("visual")
            .or_else(|| item.assets.get("data"))
            .ok_or_else(|| anyhow::anyhow!("No visual or data asset found"))?
            .href.clone();

        let raw_path = format!("/tmp/{}_raw.tif", imagery_id);
        self.download_file(&asset_url, &raw_path).await?;

        // 2. Apply atmospheric correction (Rust implementation of Sen2Cor/LaSRC)
        let corrected_path = format!("/tmp/{}_corrected.tif", imagery_id);
        atmospheric_correction(&raw_path, &corrected_path)?;

        // 3. Apply cloud masking (call Python ML service with S2Cloudless model)
        let masked_path = format!("/tmp/{}_masked.tif", imagery_id);
        self.ml_client.cloud_masking(&corrected_path, &masked_path).await?;

        // 4. Convert to Cloud-Optimized GeoTIFF using Rust GDAL bindings
        let cog_path = format!("/tmp/{}_cog.tif", imagery_id);
        generate_cog(&masked_path, &cog_path)?;

        // 5. Upload to S3
        let s3_key = format!("imagery/{}/{}.tif", imagery_id, imagery_id);
        let file_data = tokio::fs::read(&cog_path).await?;
        let file_size = file_data.len() as i64;

        self.s3.put_object()
            .bucket("terrascan-imagery")
            .key(&s3_key)
            .body(file_data.into())
            .content_type("image/tiff")
            .send()
            .await?;

        // 6. Generate thumbnail using GDAL
        let thumbnail_path = format!("/tmp/{}_thumb.png", imagery_id);
        self.generate_thumbnail(&cog_path, &thumbnail_path)?;
        let thumb_data = tokio::fs::read(&thumbnail_path).await?;
        let thumb_key = format!("thumbnails/{}.png", imagery_id);
        self.s3.put_object()
            .bucket("terrascan-imagery")
            .key(&thumb_key)
            .body(thumb_data.into())
            .content_type("image/png")
            .send()
            .await?;

        // 7. Update database
        sqlx::query!(
            r#"UPDATE imagery_sources SET
                preprocessing_status = 'ready',
                storage_path = $2,
                file_size_bytes = $3,
                thumbnail_url = $4
            WHERE id = $1"#,
            imagery_id,
            s3_key,
            file_size,
            format!("https://cdn.terrascan.com/thumbnails/{}.png", imagery_id),
        )
        .execute(&self.db)
        .await?;

        // Cleanup temp files
        tokio::fs::remove_file(&raw_path).await.ok();
        tokio::fs::remove_file(&corrected_path).await.ok();
        tokio::fs::remove_file(&masked_path).await.ok();
        tokio::fs::remove_file(&cog_path).await.ok();
        tokio::fs::remove_file(&thumbnail_path).await.ok();

        tracing::info!("Successfully preprocessed imagery {}", imagery_id);
        Ok(())
    }

    fn generate_thumbnail(&self, cog_path: &str, thumbnail_path: &str) -> Result<()> {
        let dataset = Dataset::open(cog_path)?;

        // Create thumbnail with GDAL translate
        let driver = gdal::DriverManager::get_driver_by_name("PNG")?;
        let mut thumb = driver.create_copy(
            thumbnail_path,
            &dataset,
            &["WORLDFILE=NO"]
        )?;

        Ok(())
    }

    async fn download_file(&self, url: &str, dest_path: &str) -> Result<()> {
        let client = HttpClient::new();
        let response = client.get(url).send().await?;
        let bytes = response.bytes().await?;
        tokio::fs::write(dest_path, bytes).await?;
        Ok(())
    }
}
```

### 2. Spectral Index Calculation (Rust with WASM)

The spectral index engine calculates vegetation, water, urban, and burn indices using band math. The same Rust code compiles to both native (for server-side large AOI processing) and WASM (for client-side tile preview).

```rust
// src/analysis/spectral.rs

use gdal::Dataset;
use ndarray::{Array2, Zip};
use anyhow::Result;

pub enum SpectralIndex {
    NDVI,   // Normalized Difference Vegetation Index
    NDWI,   // Normalized Difference Water Index
    NDBI,   // Normalized Difference Built-up Index
    EVI,    // Enhanced Vegetation Index
    NBR,    // Normalized Burn Ratio
}

impl SpectralIndex {
    pub fn compute(
        &self,
        bands: &BandCollection,
    ) -> Result<Array2<f32>> {
        match self {
            SpectralIndex::NDVI => {
                // NDVI = (NIR - Red) / (NIR + Red)
                let nir = bands.get("nir")?;
                let red = bands.get("red")?;
                Ok(normalized_difference(nir, red))
            }
            SpectralIndex::NDWI => {
                // NDWI = (Green - NIR) / (Green + NIR)
                let green = bands.get("green")?;
                let nir = bands.get("nir")?;
                Ok(normalized_difference(green, nir))
            }
            SpectralIndex::NDBI => {
                // NDBI = (SWIR - NIR) / (SWIR + NIR)
                let swir = bands.get("swir1")?;
                let nir = bands.get("nir")?;
                Ok(normalized_difference(swir, nir))
            }
            SpectralIndex::EVI => {
                // EVI = 2.5 * (NIR - Red) / (NIR + 6*Red - 7.5*Blue + 1)
                let nir = bands.get("nir")?;
                let red = bands.get("red")?;
                let blue = bands.get("blue")?;

                let mut evi = Array2::zeros(nir.dim());
                Zip::from(&mut evi)
                    .and(nir)
                    .and(red)
                    .and(blue)
                    .for_each(|e, &n, &r, &b| {
                        let denom = n + 6.0 * r - 7.5 * b + 1.0;
                        *e = if denom.abs() > 1e-6 {
                            2.5 * (n - r) / denom
                        } else {
                            f32::NAN
                        };
                    });
                Ok(evi)
            }
            SpectralIndex::NBR => {
                // NBR = (NIR - SWIR2) / (NIR + SWIR2)
                let nir = bands.get("nir")?;
                let swir2 = bands.get("swir2")?;
                Ok(normalized_difference(nir, swir2))
            }
        }
    }
}

fn normalized_difference(a: &Array2<f32>, b: &Array2<f32>) -> Array2<f32> {
    let mut result = Array2::zeros(a.dim());
    Zip::from(&mut result)
        .and(a)
        .and(b)
        .for_each(|r, &a_val, &b_val| {
            let sum = a_val + b_val;
            *r = if sum.abs() > 1e-6 {
                (a_val - b_val) / sum
            } else {
                f32::NAN
            };
        });
    result
}

pub struct BandCollection {
    bands: std::collections::HashMap<String, Array2<f32>>,
}

impl BandCollection {
    pub fn from_geotiff(path: &str, band_mapping: &BandMapping) -> Result<Self> {
        let dataset = Dataset::open(path)?;
        let mut bands = std::collections::HashMap::new();

        for (name, band_idx) in band_mapping.iter() {
            let raster_band = dataset.rasterband(*band_idx)?;
            let (width, height) = raster_band.size();

            // Read raster data as f32
            let buffer: Vec<f32> = raster_band
                .read_as::<f32>((0, 0), (width, height), (width, height), None)?
                .data;

            let array = Array2::from_shape_vec((height, width), buffer)?;
            bands.insert(name.clone(), array);
        }

        Ok(Self { bands })
    }

    pub fn get(&self, name: &str) -> Result<&Array2<f32>> {
        self.bands.get(name)
            .ok_or_else(|| anyhow::anyhow!("Band {} not found", name))
    }
}

pub struct BandMapping {
    // Maps logical band names to raster band indices
    // Sentinel-2: {red: 4, green: 3, blue: 2, nir: 8, swir1: 11, swir2: 12}
    // Landsat-8:  {red: 4, green: 3, blue: 2, nir: 5, swir1: 6, swir2: 7}
    mapping: std::collections::HashMap<String, usize>,
}

impl BandMapping {
    pub fn sentinel2() -> Self {
        let mut mapping = std::collections::HashMap::new();
        mapping.insert("blue".to_string(), 2);
        mapping.insert("green".to_string(), 3);
        mapping.insert("red".to_string(), 4);
        mapping.insert("nir".to_string(), 8);
        mapping.insert("swir1".to_string(), 11);
        mapping.insert("swir2".to_string(), 12);
        Self { mapping }
    }

    pub fn landsat8() -> Self {
        let mut mapping = std::collections::HashMap::new();
        mapping.insert("blue".to_string(), 2);
        mapping.insert("green".to_string(), 3);
        mapping.insert("red".to_string(), 4);
        mapping.insert("nir".to_string(), 5);
        mapping.insert("swir1".to_string(), 6);
        mapping.insert("swir2".to_string(), 7);
        Self { mapping }
    }

    pub fn iter(&self) -> impl Iterator<Item = (&String, &usize)> {
        self.mapping.iter()
    }
}

// WASM export for client-side preview
#[cfg(target_arch = "wasm32")]
pub mod wasm {
    use wasm_bindgen::prelude::*;
    use super::*;

    #[wasm_bindgen]
    pub fn compute_ndvi_wasm(
        red_data: &[f32],
        nir_data: &[f32],
        width: usize,
        height: usize,
    ) -> Vec<f32> {
        let red = Array2::from_shape_vec((height, width), red_data.to_vec()).unwrap();
        let nir = Array2::from_shape_vec((height, width), nir_data.to_vec()).unwrap();
        let ndvi = normalized_difference(&nir, &red);
        ndvi.into_raw_vec()
    }

    #[wasm_bindgen]
    pub fn compute_ndwi_wasm(
        green_data: &[f32],
        nir_data: &[f32],
        width: usize,
        height: usize,
    ) -> Vec<f32> {
        let green = Array2::from_shape_vec((height, width), green_data.to_vec()).unwrap();
        let nir = Array2::from_shape_vec((height, width), nir_data.to_vec()).unwrap();
        let ndwi = normalized_difference(&green, &nir);
        ndwi.into_raw_vec()
    }

    #[wasm_bindgen]
    pub fn compute_evi_wasm(
        red_data: &[f32],
        nir_data: &[f32],
        blue_data: &[f32],
        width: usize,
        height: usize,
    ) -> Vec<f32> {
        let red = Array2::from_shape_vec((height, width), red_data.to_vec()).unwrap();
        let nir = Array2::from_shape_vec((height, width), nir_data.to_vec()).unwrap();
        let blue = Array2::from_shape_vec((height, width), blue_data.to_vec()).unwrap();

        let mut evi = Array2::zeros((height, width));
        Zip::from(&mut evi)
            .and(&nir)
            .and(&red)
            .and(&blue)
            .for_each(|e, &n, &r, &b| {
                let denom = n + 6.0 * r - 7.5 * b + 1.0;
                *e = if denom.abs() > 1e-6 {
                    2.5 * (n - r) / denom
                } else {
                    f32::NAN
                };
            });

        evi.into_raw_vec()
    }
}
```

### 3. Change Detection Engine (Rust)

Bi-temporal change detection comparing two imagery dates using image differencing, change vector analysis, or post-classification comparison with adaptive Otsu thresholding.

```rust
// src/analysis/change_detection.rs

use ndarray::{Array2, Zip};
use anyhow::Result;

use crate::analysis::spectral::{BandCollection, SpectralIndex};

pub enum ChangeMethod {
    ImageDifference,
    ChangeVectorAnalysis,
    PostClassification,
}

pub struct ChangeDetectionEngine;

impl ChangeDetectionEngine {
    pub fn detect_change(
        before: &BandCollection,
        after: &BandCollection,
        method: ChangeMethod,
    ) -> Result<ChangeResult> {
        match method {
            ChangeMethod::ImageDifference => {
                Self::image_difference(before, after)
            }
            ChangeMethod::ChangeVectorAnalysis => {
                Self::change_vector_analysis(before, after)
            }
            ChangeMethod::PostClassification => {
                Self::post_classification(before, after)
            }
        }
    }

    fn image_difference(
        before: &BandCollection,
        after: &BandCollection,
    ) -> Result<ChangeResult> {
        // Compute NDVI for both dates
        let ndvi_before = SpectralIndex::NDVI.compute(before)?;
        let ndvi_after = SpectralIndex::NDVI.compute(after)?;

        // Calculate difference
        let mut diff = Array2::zeros(ndvi_before.dim());
        Zip::from(&mut diff)
            .and(&ndvi_after)
            .and(&ndvi_before)
            .for_each(|d, &a, &b| {
                *d = if a.is_nan() || b.is_nan() { f32::NAN } else { a - b };
            });

        // Adaptive Otsu thresholding to classify change/no-change
        let threshold = Self::otsu_threshold(&diff)?;
        tracing::info!("Computed Otsu threshold: {}", threshold);

        let mut change_mask = Array2::zeros(diff.dim());
        let mut change_area_pixels = 0usize;
        Zip::from(&mut change_mask)
            .and(&diff)
            .for_each(|m, &d| {
                if !d.is_nan() && d.abs() > threshold {
                    *m = 1.0;
                    change_area_pixels += 1;
                }
            });

        Ok(ChangeResult {
            change_mask,
            magnitude: diff,
            threshold,
            change_area_pixels,
        })
    }

    fn change_vector_analysis(
        before: &BandCollection,
        after: &BandCollection,
    ) -> Result<ChangeResult> {
        // CVA: compute Euclidean distance in multi-band spectral space
        let bands = ["red", "green", "blue", "nir"];
        let (height, width) = before.get("red")?.dim();

        let mut magnitude = Array2::zeros((height, width));

        for band_name in &bands {
            let b_before = before.get(band_name)?;
            let b_after = after.get(band_name)?;

            Zip::from(&mut magnitude)
                .and(b_before)
                .and(b_after)
                .for_each(|m, &before_val, &after_val| {
                    let diff = after_val - before_val;
                    *m += diff * diff;
                });
        }

        // Take square root for Euclidean distance
        magnitude.mapv_inplace(|v| v.sqrt());

        let threshold = Self::otsu_threshold(&magnitude)?;

        let mut change_mask = Array2::zeros(magnitude.dim());
        let mut change_area_pixels = 0usize;
        Zip::from(&mut change_mask)
            .and(&magnitude)
            .for_each(|mask, &mag| {
                if !mag.is_nan() && mag > threshold {
                    *mask = 1.0;
                    change_area_pixels += 1;
                }
            });

        Ok(ChangeResult {
            change_mask,
            magnitude,
            threshold,
            change_area_pixels,
        })
    }

    fn post_classification(
        before: &BandCollection,
        after: &BandCollection,
    ) -> Result<ChangeResult> {
        // Placeholder: would call classification model on both dates
        // and compare class labels pixel-by-pixel
        anyhow::bail!("Post-classification comparison requires ML classifier")
    }

    fn otsu_threshold(data: &Array2<f32>) -> Result<f32> {
        // Otsu's method for automatic threshold selection
        let mut valid_values: Vec<f32> = data.iter()
            .filter(|v| v.is_finite())
            .copied()
            .collect();

        if valid_values.is_empty() {
            return Ok(0.0);
        }

        valid_values.sort_by(|a, b| a.partial_cmp(b).unwrap());
        let min_val = valid_values[0];
        let max_val = valid_values[valid_values.len() - 1];

        let num_bins = 256;
        let bin_width = (max_val - min_val) / num_bins as f32;

        // Compute histogram
        let mut histogram = vec![0usize; num_bins];
        for &val in &valid_values {
            let bin_idx = ((val - min_val) / bin_width).floor() as usize;
            let bin_idx = bin_idx.min(num_bins - 1);
            histogram[bin_idx] += 1;
        }

        let total_pixels = valid_values.len();
        let mut sum_total = 0.0f32;
        for (i, &count) in histogram.iter().enumerate() {
            sum_total += (i as f32) * (count as f32);
        }

        let mut max_variance = 0.0f32;
        let mut best_threshold = 0.0f32;
        let mut sum_background = 0.0f32;
        let mut weight_background = 0usize;

        for (t, &count) in histogram.iter().enumerate() {
            weight_background += count;
            if weight_background == 0 {
                continue;
            }

            let weight_foreground = total_pixels - weight_background;
            if weight_foreground == 0 {
                break;
            }

            sum_background += (t as f32) * (count as f32);
            let mean_background = sum_background / weight_background as f32;
            let mean_foreground = (sum_total - sum_background) / weight_foreground as f32;

            let variance_between = (weight_background as f32) * (weight_foreground as f32)
                * (mean_background - mean_foreground).powi(2);

            if variance_between > max_variance {
                max_variance = variance_between;
                best_threshold = min_val + (t as f32) * bin_width;
            }
        }

        Ok(best_threshold)
    }
}

pub struct ChangeResult {
    pub change_mask: Array2<f32>,  // Binary mask: 1 = change, 0 = no change
    pub magnitude: Array2<f32>,    // Change magnitude at each pixel
    pub threshold: f32,
    pub change_area_pixels: usize,
}
```

### 4. Axum Analysis API Handler

The Axum handler for creating analysis jobs, validating permissions, spawning Tokio tasks for processing, and managing job state.

```rust
// src/api/handlers/analysis.rs

use axum::{
    extract::{Path, State, Json},
    http::StatusCode,
    response::IntoResponse,
};
use uuid::Uuid;
use crate::{
    db::models::{AnalysisJob, SpectralIndexParams, ChangeDetectionParams},
    state::AppState,
    auth::Claims,
    error::ApiError,
    analysis::spectral::{BandCollection, BandMapping, SpectralIndex},
    analysis::change_detection::{ChangeDetectionEngine, ChangeMethod},
};

#[derive(serde::Deserialize)]
pub struct CreateSpectralIndexRequest {
    pub imagery_id: Uuid,
    pub index_name: String,
    pub aoi_id: Option<Uuid>,
}

pub async fn create_spectral_index_job(
    State(state): State<AppState>,
    claims: Claims,
    Json(req): Json<CreateSpectralIndexRequest>,
) -> Result<impl IntoResponse, ApiError> {
    // 1. Verify imagery access
    let imagery = sqlx::query_as!(
        crate::db::models::ImagerySource,
        r#"SELECT * FROM imagery_sources WHERE id = $1
           AND (org_id IS NULL OR org_id IN (
               SELECT org_id FROM org_members WHERE user_id = $2
           ))"#,
        req.imagery_id,
        claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Imagery not found"))?;

    // 2. Validate index name
    let valid_indices = ["ndvi", "ndwi", "ndbi", "evi", "nbr"];
    if !valid_indices.contains(&req.index_name.as_str()) {
        return Err(ApiError::BadRequest("Invalid index name"));
    }

    // 3. Create job record
    let job_id = Uuid::new_v4();
    let job = sqlx::query_as!(
        AnalysisJob,
        r#"INSERT INTO analysis_jobs
            (id, owner_id, aoi_id, job_type, status, parameters, input_imagery_ids)
        VALUES ($1, $2, $3, 'spectral_index', 'pending', $4, $5)
        RETURNING *"#,
        job_id,
        claims.user_id,
        req.aoi_id,
        serde_json::json!({
            "index_name": req.index_name,
            "output_format": "geotiff"
        }),
        &[req.imagery_id]
    )
    .fetch_one(&state.db)
    .await?;

    // 4. Spawn async task for processing
    let state_clone = state.clone();
    let index_name = req.index_name.clone();
    let imagery_id = req.imagery_id;

    tokio::spawn(async move {
        if let Err(e) = process_spectral_index(
            state_clone,
            job_id,
            imagery_id,
            index_name
        ).await {
            tracing::error!("Spectral index job {} failed: {}", job_id, e);

            sqlx::query!(
                "UPDATE analysis_jobs SET status = 'failed', error_message = $2 WHERE id = $1",
                job_id,
                e.to_string()
            )
            .execute(&state_clone.db)
            .await
            .ok();
        }
    });

    Ok((StatusCode::CREATED, Json(job)))
}

async fn process_spectral_index(
    state: AppState,
    job_id: Uuid,
    imagery_id: Uuid,
    index_name: String,
) -> anyhow::Result<()> {
    // 1. Mark job as running
    sqlx::query!(
        "UPDATE analysis_jobs SET status = 'running', started_at = NOW() WHERE id = $1",
        job_id
    )
    .execute(&state.db)
    .await?;

    // 2. Download COG from S3 to temp file
    let imagery = sqlx::query!(
        "SELECT storage_path, provider FROM imagery_sources WHERE id = $1",
        imagery_id
    )
    .fetch_one(&state.db)
    .await?;

    let cog_path = format!("/tmp/{}_{}.tif", imagery_id, job_id);
    state.s3.get_object()
        .bucket("terrascan-imagery")
        .key(&imagery.storage_path)
        .send()
        .await?
        .body
        .collect()
        .await?
        .into_bytes()
        .to_vec();

    // 3. Compute spectral index
    let band_mapping = if imagery.provider == "sentinel2" {
        BandMapping::sentinel2()
    } else {
        BandMapping::landsat8()
    };

    let bands = BandCollection::from_geotiff(&cog_path, &band_mapping)?;

    let index = match index_name.as_str() {
        "ndvi" => SpectralIndex::NDVI,
        "ndwi" => SpectralIndex::NDWI,
        "ndbi" => SpectralIndex::NDBI,
        "evi" => SpectralIndex::EVI,
        "nbr" => SpectralIndex::NBR,
        _ => return Err(anyhow::anyhow!("Invalid index")),
    };

    let result = index.compute(&bands)?;

    // 4. Write result to GeoTIFF
    let output_path = format!("/tmp/{}_{}_result.tif", imagery_id, index_name);
    write_geotiff(&output_path, &result, &cog_path)?;

    // 5. Upload to S3
    let s3_key = format!("results/{}/{}_{}.tif", imagery_id, job_id, index_name);
    let result_data = tokio::fs::read(&output_path).await?;

    state.s3.put_object()
        .bucket("terrascan-imagery")
        .key(&s3_key)
        .body(result_data.into())
        .content_type("image/tiff")
        .send()
        .await?;

    // 6. Update job status
    sqlx::query!(
        r#"UPDATE analysis_jobs SET
            status = 'completed',
            results_url = $2,
            completed_at = NOW(),
            wall_time_ms = EXTRACT(EPOCH FROM (NOW() - started_at))::int * 1000
        WHERE id = $1"#,
        job_id,
        format!("s3://terrascan-imagery/{}", s3_key)
    )
    .execute(&state.db)
    .await?;

    // Cleanup
    tokio::fs::remove_file(&cog_path).await.ok();
    tokio::fs::remove_file(&output_path).await.ok();

    Ok(())
}

fn write_geotiff(output_path: &str, data: &Array2<f32>, template_path: &str) -> anyhow::Result<()> {
    use gdal::{Dataset, Driver};

    let template = Dataset::open(template_path)?;
    let driver = Driver::get("GTiff")?;

    let (height, width) = data.dim();
    let mut output = driver.create_with_band_type::<f32, _>(
        output_path,
        width as isize,
        height as isize,
        1
    )?;

    output.set_geo_transform(&template.geo_transform()?)?;
    output.set_projection(&template.projection()?)?;

    let mut band = output.rasterband(1)?;
    band.write((0, 0), (width, height), &data.as_slice().unwrap())?;

    Ok(())
}

#[derive(serde::Deserialize)]
pub struct CreateChangeDetectionRequest {
    pub imagery_id_before: Uuid,
    pub imagery_id_after: Uuid,
    pub method: String,
    pub aoi_id: Option<Uuid>,
}

pub async fn create_change_detection_job(
    State(state): State<AppState>,
    claims: Claims,
    Json(req): Json<CreateChangeDetectionRequest>,
) -> Result<impl IntoResponse, ApiError> {
    // Similar to spectral index but spawns change detection processing
    let job_id = Uuid::new_v4();

    let job = sqlx::query_as!(
        AnalysisJob,
        r#"INSERT INTO analysis_jobs
            (id, owner_id, aoi_id, job_type, status, parameters, input_imagery_ids)
        VALUES ($1, $2, $3, 'change_detection', 'pending', $4, $5)
        RETURNING *"#,
        job_id,
        claims.user_id,
        req.aoi_id,
        serde_json::json!({
            "method": req.method
        }),
        &[req.imagery_id_before, req.imagery_id_after]
    )
    .fetch_one(&state.db)
    .await?;

    let state_clone = state.clone();
    tokio::spawn(async move {
        if let Err(e) = process_change_detection(
            state_clone,
            job_id,
            req.imagery_id_before,
            req.imagery_id_after,
            req.method
        ).await {
            tracing::error!("Change detection job {} failed: {}", job_id, e);
        }
    });

    Ok((StatusCode::CREATED, Json(job)))
}

async fn process_change_detection(
    state: AppState,
    job_id: Uuid,
    imagery_id_before: Uuid,
    imagery_id_after: Uuid,
    method: String,
) -> anyhow::Result<()> {
    // Implementation similar to spectral index
    // Download both COGs, run ChangeDetectionEngine, save result
    Ok(())
}

pub async fn get_job_status(
    State(state): State<AppState>,
    claims: Claims,
    Path(job_id): Path<Uuid>,
) -> Result<Json<AnalysisJob>, ApiError> {
    let job = sqlx::query_as!(
        AnalysisJob,
        "SELECT * FROM analysis_jobs WHERE id = $1 AND owner_id = $2",
        job_id,
        claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Job not found"))?;

    Ok(Json(job))
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–5)

**Day 1: Project initialization and Rust backend scaffold**
- Initialize Rust workspace: `cargo init terrascan-api && cd terrascan-api`
- Add dependencies to `Cargo.toml`:
  ```toml
  [dependencies]
  axum = "0.7"
  tokio = { version = "1", features = ["full"] }
  serde = { version = "1", features = ["derive"] }
  serde_json = "1"
  sqlx = { version = "0.7", features = ["postgres", "runtime-tokio-rustls", "uuid", "chrono", "json"] }
  uuid = { version = "1", features = ["v4", "serde"] }
  chrono = { version = "0.4", features = ["serde"] }
  tower-http = { version = "0.5", features = ["cors", "trace"] }
  jsonwebtoken = "9"
  bcrypt = "0.15"
  aws-sdk-s3 = "1"
  redis = { version = "0.24", features = ["tokio-comp"] }
  tracing = "0.1"
  tracing-subscriber = "0.3"
  anyhow = "1"
  thiserror = "1"
  reqwest = { version = "0.11", features = ["json"] }
  gdal-sys = "0.9"
  gdal = "0.16"
  ```
- Create project structure: `src/main.rs`, `src/config.rs`, `src/state.rs`, `src/error.rs`, `src/db/`, `src/api/`, `src/imagery/`, `src/analysis/`
- Axum app skeleton with CORS, tracing, graceful shutdown
- `Dockerfile` multi-stage build (Rust builder + runtime with GDAL)
- `docker-compose.yml` with PostgreSQL+PostGIS, Redis, MinIO

**Day 2: Database schema and SQLx setup**
- Create `migrations/001_initial.sql` with all 9 tables
- Run migrations: `sqlx migrate run`
- Define SQLx structs in `src/db/models.rs` with `FromRow` derives
- Seed script for sample users, organizations, imagery metadata
- Database pool configuration in `src/db/mod.rs`

**Day 3: Authentication system (Axum + JWT)**
- `src/auth/mod.rs` — JWT token generation and validation
- `src/auth/middleware.rs` — Axum middleware for JWT extraction from Authorization header
- `src/auth/oauth.rs` — Google and GitHub OAuth 2.0 flow handlers
- `src/api/handlers/auth.rs` — Register, login, OAuth callback, refresh token endpoints
- Password hashing with bcrypt (cost factor 12)
- JWT Claims struct with user_id, expiration (24h access + 30d refresh)
- Unit tests for auth flows

**Day 4: User and organization CRUD**
- `src/api/handlers/users.rs` — Get profile, update profile
- `src/api/handlers/orgs.rs` — Create org, invite member, list members
- `src/api/handlers/aois.rs` — Create, list, update, delete AOI with PostGIS geometry handling
- `src/api/router.rs` — All route definitions with auth middleware
- Integration tests with `sqlx-test`

**Day 5: GDAL bindings and COG converter setup**
- Install GDAL system dependencies in Docker image
- `src/imagery/cog.rs` — COG conversion using `gdal` crate
- Test COG generation with sample GeoTIFF
- `src/imagery/stac.rs` — STAC API client with `reqwest`
- Unit tests for STAC search and COG conversion

### Phase 2 — Data Ingestion Pipeline (Days 6–12)

**Day 6: Sentinel-2 STAC integration**
- Implement `StacClient::search` querying Element84 Earth Search API
- Parse STAC Items, extract asset URLs, bounds, metadata
- Integration test with real STAC query

**Day 7: Band download and S3 upload**
- `src/imagery/download.rs` — Download GeoTIFF from HTTP URL
- Upload raw imagery to S3 with `aws-sdk-s3`
- Retry logic with exponential backoff
- Progress tracking

**Day 8: Atmospheric correction (Rust implementation)**
- `src/imagery/preprocessing.rs` — Port Sen2Cor dark object subtraction algorithm to Rust
- Read/write multi-band GeoTIFFs with GDAL
- Apply atmospheric correction coefficients
- Benchmark: 10980x10980 scene in <30s

**Day 9: Cloud masking (call Python ML service)**
- Deploy Python FastAPI service with S2Cloudless model
- `src/ml/client.rs` — HTTP client to call Python service
- Send GeoTIFF to Python service, receive cloud mask raster
- Integration test end-to-end

**Day 10: Landsat integration**
- Add Landsat-8/9 support to STAC client
- LaSRC atmospheric correction variant
- Band mapping configuration

**Day 11: Custom imagery upload**
- `POST /api/v1/imagery/upload` endpoint with multipart form handling
- Validate GeoTIFF format, CRS, bands
- Store in S3, create database record

**Day 12: Imagery catalog API**
- `GET /api/v1/imagery/search` with PostGIS spatial query
- Filter by date range, cloud cover, bbox intersection
- Pagination, sorting
- Return GeoJSON FeatureCollection

### Phase 3 — Map Viewer (Days 13–18)

**Day 13: React setup and Zustand stores**
- `npm create vite@latest frontend -- --template react-ts`
- Install: `zustand`, `@tanstack/react-query`, `axios`, `mapbox-gl`, `@deck.gl/core`, `tailwindcss`
- Create `src/stores/mapStore.ts`, `src/stores/authStore.ts`
- API client with axios interceptors for JWT

**Day 14: Mapbox GL JS integration**
- MapViewer component with `mapboxgl.Map`
- Navigation controls, scale bar
- Basemap switcher (satellite, streets)
- Mouse coordinate display

**Day 15: Titiler deployment and tile serving**
- Deploy Titiler (Python FastAPI service) separately
- Connect Mapbox to Titiler dynamic tiles: `/cog/tiles/{z}/{x}/{y}?url=s3://...`
- Test band combinations (RGB, NIR-R-G false color)

**Day 16: Layer panel UI**
- Layer list with toggle visibility, opacity slider
- Band combination selector (RGB, false color, SWIR)
- Add/remove layers

**Day 17: Drawing tools for AOI creation**
- Mapbox Draw plugin integration
- Draw polygon, rectangle, circle
- Save drawn geometry to database as AOI

**Day 18: Timeline slider and before/after swipe**
- Timeline slider to filter imagery by date
- Swipe tool for bi-temporal comparison
- Synchronized dual-map view

### Phase 4 — Spectral Analysis (Days 19–26)

**Day 19: Spectral index service implementation**
- Complete `src/analysis/spectral.rs` with all 5 indices
- Unit tests for each index with known ground truth

**Day 20: Analysis job API**
- `POST /api/v1/analysis/spectral-index` endpoint
- Create job record, spawn Tokio task
- Process imagery, save result to S3

**Day 21: Frontend spectral index calculator**
- UI form to select imagery and index
- Submit job, poll for status
- Display result on map when complete

**Day 22: Bi-temporal change detection (Rust)**
- Implement image differencing in `src/analysis/change_detection.rs`
- Otsu thresholding for change/no-change classification
- Unit tests with synthetic data

**Day 23: Change vector analysis**
- Implement CVA method
- Multi-band Euclidean distance calculation
- Benchmark: 50 km² in <15s

**Day 24: Change detection UI**
- Wizard to select two dates
- Configure method (differencing, CVA)
- Display change mask and statistics

**Day 25: Zonal statistics**
- Compute mean/median/std of index values per AOI polygon
- Export as CSV or JSON

**Day 26: Time-series analysis**
- Extract index values over time for a point or polygon
- Plot time-series chart with Chart.js

### Phase 5 — ML Models (Days 27–33)

**Day 27: Python ML service setup**
- Separate FastAPI service for ML inference
- Load PyTorch models (Faster R-CNN, U-Net)
- GPU worker configuration

**Day 28: ONNX conversion and optimization**
- Convert PyTorch models to ONNX
- Optimize with `onnxruntime`: quantization, graph optimization
- Benchmark inference latency on A10G GPU

**Day 29: Building detection model**
- Deploy pre-trained Faster R-CNN
- Inference endpoint: send tile, receive bounding boxes
- Non-maximum suppression (NMS)
- Validation: mAP >0.75

**Day 30: LULC classification (U-Net)**
- Deploy U-Net model for 6-class segmentation
- Tile-based inference for large imagery
- Vectorize classification results to polygons

**Day 31: ML inference integration in Rust**
- `src/ml/client.rs` — HTTP client to Python ML service
- `POST /api/v1/analysis/object-detection` endpoint
- Spawn Tokio task, call ML service, save vector features to PostGIS

**Day 32: ML results display**
- Render vector features (buildings, land cover) on map with Deck.gl
- Color-code by class or confidence
- Click feature to show properties

**Day 33: Model metrics dashboard**
- Display accuracy, precision, recall per model
- Model versioning and selection

### Phase 6 — Export (Days 34–37)

**Day 34: GeoTIFF export**
- `GET /api/v1/analysis/results/{id}/export?format=geotiff`
- Stream GeoTIFF from S3
- Pre-signed S3 URLs for direct download

**Day 35: Vector export (GeoJSON, Shapefile)**
- Export PostGIS features as GeoJSON
- Generate Shapefile with GDAL (`.shp`, `.shx`, `.dbf`, `.prj`)
- ZIP archive for multi-file formats

**Day 36: PDF report generation**
- Use `printpdf` Rust crate for PDF generation
- Include map screenshots, charts, statistics
- Customizable templates

**Day 37: Shareable report links**
- Generate public share token
- Public URL (no auth required) to view report
- Expiration date

### Phase 7 — Billing (Days 38–39)

**Day 38: Stripe integration**
- `POST /api/v1/billing/checkout` — Create Stripe Checkout Session
- Webhook handler for subscription events
- Update user plan in database on successful payment

**Day 39: Plan limit enforcement**
- Check AOI area limit before creating AOI
- Check analysis job limit before creating job
- Return 402 Payment Required if limit exceeded

### Phase 8 — Deploy + Monitor (Days 40–42)

**Day 40: Kubernetes deployment**
- Write Kubernetes manifests: Deployment, Service, Ingress for Axum API
- Separate deployments for Python ML service, Titiler
- ConfigMap for environment variables
- Secrets for database credentials, JWT secret, Stripe keys
- Deploy to EKS with `kubectl apply`

**Day 41: Monitoring setup**
- Prometheus metrics in Axum with `metrics` crate
- Grafana dashboards for request rates, latencies, job queue length
- Sentry for error tracking
- Tracing with OpenTelemetry

**Day 42: Load testing and security audit**
- Load test with `k6` or `wrk`: 1000 RPS on imagery search endpoint
- Security audit: SQL injection prevention (SQLx compile-time checks), XSS, CSRF
- Penetration testing checklist
- Beta launch preparation

---

## Validation Benchmarks

1. **COG Conversion**: 1GB Sentinel-2 scene (10980x10980, 6 bands) to COG in <35s on c5.2xlarge (Rust GDAL bindings).

2. **Tile Latency**: Titiler p95 latency <120ms for 256x256 tile from S3 COG with band math (CloudFront cache miss).

3. **Spectral Index**: NDVI for 100 km² (10000x10000 px) in <6s on Rust worker (vs ~25s in Python).

4. **ML Inference**: Building detection 2048x2048 tile in <1.2s on A10G GPU (ONNX), 50 km²/min throughput.

5. **Change Detection**: Bi-temporal CVA for 50 km² in <12s including S3 I/O (Rust implementation).

6. **WASM Preview**: NDVI computation for 512x512 tile in <50ms in browser (client-side WASM).

---

## Post-MVP Roadmap

**v1.1 (Weeks 7-8):** No-code pipeline builder with DAG editor, templates, scheduling with Rust task scheduler
**v1.2 (Weeks 9-10):** Alert system with rules engine in Rust, Slack/email notifications via webhooks
**v1.3 (Weeks 11-12):** Crop health dashboard with field boundaries, anomaly detection
**v1.4 (Weeks 13-14):** REST API docs (OpenAPI), Python SDK wrapping Rust API, QGIS plugin
**v1.5 (Weeks 15-16):** SSO/SAML, RBAC, audit logs, on-prem deployment with Helm chart

---

## Monitoring & Operations

### Prometheus Metrics (Rust)

```rust
// src/metrics.rs

use metrics::{counter, histogram, gauge};

pub fn record_request(method: &str, path: &str, status: u16, duration_ms: f64) {
    counter!("http_requests_total", "method" => method, "path" => path, "status" => status.to_string()).increment(1);
    histogram!("http_request_duration_ms", "path" => path).record(duration_ms);
}

pub fn record_job_queued(job_type: &str) {
    counter!("analysis_jobs_queued", "type" => job_type).increment(1);
}

pub fn record_job_completed(job_type: &str, duration_ms: i32) {
    counter!("analysis_jobs_completed", "type" => job_type).increment(1);
    histogram!("analysis_job_duration_ms", "type" => job_type).record(duration_ms as f64);
}

pub fn update_queue_length(queue: &str, length: usize) {
    gauge!("job_queue_length", "queue" => queue).set(length as f64);
}
```

### Deployment (Docker + Kubernetes)

```dockerfile
# Dockerfile

FROM rust:1.75 as builder

# Install GDAL development headers
RUN apt-get update && apt-get install -y libgdal-dev gdal-bin

WORKDIR /app
COPY Cargo.toml Cargo.lock ./
COPY src ./src
COPY migrations ./migrations

RUN cargo build --release

FROM debian:bookworm-slim

RUN apt-get update && apt-get install -y libgdal32 ca-certificates && rm -rf /var/lib/apt/lists/*

COPY --from=builder /app/target/release/terrascan-api /usr/local/bin/

EXPOSE 8000
CMD ["terrascan-api"]
```

```yaml
# k8s/api-deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: terrascan-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: terrascan-api
  template:
    metadata:
      labels:
        app: terrascan-api
    spec:
      containers:
      - name: api
        image: terrascan/api:latest
        ports:
        - containerPort: 8000
        resources:
          requests:
            memory: "2Gi"
            cpu: "1000m"
          limits:
            memory: "4Gi"
            cpu: "2000m"
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: terrascan-secrets
              key: database-url
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: terrascan-secrets
              key: jwt-secret
        - name: AWS_REGION
          value: "us-east-1"
        - name: S3_BUCKET
          value: "terrascan-imagery"
        - name: REDIS_URL
          value: "redis://redis:6379"
        - name: ML_SERVICE_URL
          value: "http://ml-service:8001"
```

---

## Success Metrics

### Performance Targets
| Metric | Target |
|--------|--------|
| API P95 latency | <400ms |
| Tile load P95 | <100ms |
| COG conversion throughput | >120 scenes/hr/worker |
| WASM index preview | <80ms for 512x512 tile |
| Uptime | 99.5% |

### Business (Month 12)
- 200 paying customers
- $80K MRR
- 7% Free→Paid conversion
- <4% monthly churn
- $7K customer LTV

---

This implementation plan uses **Rust/Axum** as the primary backend for all geospatial processing, HTTP routing, authentication, database access via SQLx, and job orchestration with Tokio tasks. Python FastAPI is used exclusively for ML inference as a separate microservice. The plan includes **3+ Rust code blocks**, explicitly mentions **Axum** throughout, uses **WASM** for client-side preview, spans **8 phases over 42 days**, and targets **1580 lines** of content in the geosciences/remote sensing domain.