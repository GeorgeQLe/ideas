# TerraScan — AI-Powered Satellite Imagery Analysis Platform

## Executive Summary

TerraScan is a cloud-native geospatial analytics platform that ingests multi-source satellite imagery and applies pre-built and custom ML models to extract actionable intelligence — without requiring GIS expertise or expensive desktop software. It replaces fragmented workflows involving ESRI ArcGIS, manual image interpretation, and one-off Python scripts with a unified, no-code pipeline builder for change detection, object detection, land classification, and crop health monitoring.

---

## Problem Statement

**The pain:**
- Satellite imagery is more accessible than ever (Sentinel-2 is free, Planet is cheap), but analyzing it still requires deep GIS expertise and expensive desktop tools like ArcGIS Pro ($1,500+/year per seat)
- Remote sensing analysts spend 60-70% of their time on data preprocessing — atmospheric correction, co-registration, cloud masking, mosaicking — before any actual analysis begins
- ML-based analysis (object detection, classification) requires data scientists who understand both computer vision and geospatial coordinate systems, a rare and expensive combination
- Change detection workflows are manual: download two images, align them, compute differences, visually inspect — repeated hundreds of times for monitoring projects
- Results are trapped in desktop GIS files (shapefiles, GeoTIFFs) that are difficult to share with non-technical stakeholders like executives, policymakers, or field teams

**Current workarounds:**
- ESRI ArcGIS Pro + Image Analyst extension ($3,000+/year) — powerful but desktop-bound, steep learning curve, no native ML pipeline
- Google Earth Engine — free for research but requires JavaScript/Python coding, no visual pipeline builder, unclear commercial licensing
- Custom Python scripts with GDAL/Rasterio/scikit-learn — fragile, not reproducible, no collaboration, each analyst reinvents the wheel
- Manual visual interpretation — an analyst stares at images and draws polygons, does not scale

**Market size:** The global geospatial analytics market is projected to reach $134B by 2029 (MarketsandMarkets). The remote sensing software segment alone is ~$8B, with satellite imagery analytics growing at 22% CAGR. There are an estimated 200K+ organizations worldwide actively using satellite imagery for agriculture, defense, urban planning, insurance, and environmental monitoring.

---

## Target Users

### Primary Personas

**1. Maria — Precision Agriculture Analyst**
- Works at a large agribusiness company managing 50K+ hectares across multiple regions
- Currently uses a mix of QGIS, custom Python scripts, and manual field visits to monitor crop health
- Spends 2 days per week downloading and preprocessing Sentinel-2 imagery before she can even compute NDVI maps
- Needs: automated satellite data ingestion, one-click NDVI/NDWI maps, anomaly alerts when crop stress is detected, reports for farm managers

**2. James — Defense & Intelligence Imagery Analyst**
- Works at a government contractor analyzing high-resolution commercial satellite imagery for activity monitoring
- Uses ESRI ArcGIS with custom extensions; each new analysis task requires a GIS developer to build tools
- Monitors 30+ sites of interest for changes (new construction, vehicle movement, shipping activity)
- Needs: automated change detection with alerts, object detection for vehicles/ships/aircraft, secure multi-user environment, audit trail

**3. Dr. Anika — Environmental Research Scientist**
- University researcher studying deforestation patterns in tropical regions using Landsat time series
- Writes custom Python notebooks for each study, spending weeks on data preprocessing that could be automated
- Publishes results but cannot easily share interactive maps with collaborators or policymakers
- Needs: reproducible analysis pipelines, time-series analysis tools, shareable interactive visualizations, export for publication

### Secondary Personas
- Urban planners tracking city expansion and impervious surface growth
- Insurance underwriters assessing flood/wildfire risk using satellite-derived land cover data
- Mining and energy companies monitoring environmental compliance at remote sites
- NGOs tracking humanitarian crises (refugee camp growth, conflict damage assessment)

---

## Solution Overview

TerraScan works through a streamlined workflow:
1. **Connect data sources** — link Sentinel-2, Landsat, Planet, Maxar, or upload custom imagery; TerraScan handles ingestion, cloud masking, atmospheric correction, and tiling automatically
2. **Define an area of interest** — draw a polygon on the interactive map, upload a shapefile/GeoJSON, or select from a library of administrative boundaries
3. **Build an analysis pipeline** — use the visual no-code pipeline builder to chain processing steps: spectral indices, ML models, filters, temporal aggregation, and change detection
4. **Run and monitor** — pipelines execute on scalable cloud infrastructure with GPU acceleration for ML models; results stream back to the map viewer in real time
5. **Share and act** — generate reports, set up automated alerts, export GeoTIFFs/shapefiles, share interactive map links with stakeholders, or push results to downstream systems via API

---

## Core Features

### F1: Multi-Source Satellite Data Ingestion
- Native connectors for free data: Sentinel-2 (10m optical), Landsat-8/9 (30m), Sentinel-1 (SAR), MODIS, VIIRS
- Commercial data marketplace integration: Planet (3m daily), Maxar (30cm), Airbus Pleiades (50cm)
- Custom imagery upload: drone orthomosaics, aerial photography, any GeoTIFF with projection info
- Automated preprocessing pipeline: atmospheric correction (Sen2Cor/LaSRC), cloud/shadow masking (Fmask/S2Cloudless), spatial co-registration, radiometric normalization
- Imagery catalog with spatiotemporal search: filter by date range, cloud cover percentage, spatial overlap with AOI
- Automatic mosaic generation for large areas spanning multiple scenes
- Data cached in cloud-optimized GeoTIFF (COG) format for fast visualization and analysis

### F2: Interactive Map Viewer
- WebGL-accelerated map interface built on Mapbox GL JS / Deck.gl
- Seamless pan/zoom across imagery at any resolution, from continent-scale to sub-meter
- Multi-layer visualization: overlay analysis results, vector features, and basemaps
- Swipe tool for before/after comparison of two dates
- Synchronized dual-map view for side-by-side temporal comparison
- Dynamic band combination selector: true color, false color (vegetation), urban, geology, custom RGB from any bands
- On-the-fly histogram stretch and enhancement controls
- Pixel-level value inspector: click any point to see spectral values across all bands
- Drawing tools for AOI creation: polygon, rectangle, circle, freehand
- Annotation tools for markup and collaboration

### F3: Spectral Index Calculator
- Library of 40+ pre-built spectral indices with one-click computation:
  - Vegetation: NDVI, EVI, SAVI, GNDVI, LAI, MSAVI2
  - Water: NDWI, MNDWI, AWEI
  - Built-up/Urban: NDBI, UI, BUI
  - Soil: BSI, BI, NDSI
  - Burn/Fire: NBR, dNBR, BAI
  - Snow/Ice: NDSI, S3
  - Geology: clay ratio, ferrous minerals, iron oxide
- Custom index formula editor with band math (e.g., "(B8 - B4) / (B8 + B4)")
- Temporal index profiles: plot index values over time for any point or polygon
- Threshold-based classification from any index (e.g., NDVI > 0.4 = healthy vegetation)
- Statistical zonal summaries: mean, median, std, min, max per polygon feature

### F4: ML Object Detection
- Pre-trained models available out of the box:
  - **Building footprint extraction** (global coverage, trained on Microsoft/Google open datasets)
  - **Vehicle detection** (cars, trucks, aircraft on tarmac) from high-res imagery
  - **Ship/vessel detection** from optical and SAR imagery
  - **Solar panel detection** from aerial/satellite imagery
  - **Swimming pool detection** for property assessment
  - **Deforestation patch detection** from Sentinel-2 time series
  - **Road network extraction** as vector polylines
- Results delivered as vector features (GeoJSON) with confidence scores
- Adjustable confidence threshold and minimum object size filters
- Model performance metrics displayed: precision, recall, F1 on validation set
- Batch processing: run detection across thousands of scenes automatically
- Ensemble mode: combine multiple models and take consensus predictions

### F5: Change Detection Engine
- **Bi-temporal change detection**: compare two dates, highlight what changed
  - Spectral change vector analysis (CVA)
  - Image differencing with adaptive thresholding
  - Post-classification comparison
- **Time-series change detection**: continuous monitoring over months/years
  - BFAST (Breaks for Additive Seasonal and Trend) for vegetation monitoring
  - CUSUM-based change point detection
  - LandTrendr temporal segmentation
- **Object-level change tracking**: detect new/removed buildings, road segments, water bodies
- Change magnitude and direction classification: what type of change occurred (urbanization, deforestation, flooding, agricultural conversion)
- Automated change reports with before/after thumbnails, area statistics, and trend graphs
- Alert system: define rules (e.g., "notify me if >5 hectares of forest lost in AOI") and receive email/webhook notifications

### F6: Land Use / Land Cover Classification
- Pre-trained LULC classifier producing standard categories: urban, agriculture, forest, water, barren, wetland, grassland
- Based on Random Forest and deep learning (U-Net) ensembles trained on ESA WorldCover and NLCD labels
- Custom classifier training: upload your own labeled polygons, TerraScan trains a model on your spectral/temporal features
- Active learning workflow: model suggests uncertain areas for human labeling, iteratively improves
- Multi-temporal classification using seasonal composites (spring/summer/fall/winter) for better accuracy
- Accuracy assessment: confusion matrix, per-class precision/recall, overall accuracy, kappa coefficient
- Area statistics by class with exportable tables and charts

### F7: Crop Health Monitoring Suite
- Field boundary detection from Sentinel-2 imagery using edge detection and segmentation
- Per-field NDVI/EVI time-series profiles compared against historical averages
- Anomaly detection: flag fields where vegetation index is below expected range for growth stage
- Growth stage estimation using crop phenology models (planting, emergence, peak, senescence)
- Yield estimation models (correlation of peak NDVI with historical yield data)
- Irrigation detection from NDWI temporal patterns
- Pest/disease risk mapping based on spectral signature anomalies
- Integration with weather data (precipitation, temperature, soil moisture) for context

### F8: No-Code Pipeline Builder
- Visual drag-and-drop canvas for constructing analysis workflows
- Processing nodes: data source, spectral index, ML model, filter, temporal aggregation, classification, export
- Conditional logic: if/else branching based on results (e.g., if change detected → run object detection → alert)
- Scheduling: run pipelines on a recurring basis (daily, weekly, per new image acquisition)
- Parameterization: expose parameters (date range, AOI, thresholds) for easy re-use
- Pipeline templates gallery: pre-built pipelines for common use cases (crop monitoring, deforestation alert, urban growth tracking)
- Pipeline versioning and sharing within teams
- Execution logs with per-node timing, intermediate results preview, and error diagnostics

### F9: Custom Model Training
- Upload labeled training data (GeoJSON polygons + class labels over imagery)
- AutoML model selection: Random Forest, Gradient Boosting, U-Net (semantic segmentation), Faster R-CNN (object detection)
- Feature engineering assistant: automatically extracts spectral bands, indices, texture features (GLCM), and temporal statistics
- Train/validation/test split with spatial stratification to avoid data leakage
- Hyperparameter tuning with cross-validation
- Training progress dashboard with real-time loss curves and validation metrics
- Model comparison view: evaluate multiple models side by side
- Deploy trained model as a pipeline node for use in production workflows
- Model versioning and rollback

### F10: Report & Export Engine
- Automated PDF report generation with maps, charts, statistics, and narrative text
- Customizable report templates with organization branding
- Interactive web report links (shareable, no login required for viewing)
- Export formats: GeoTIFF (raster), Shapefile/GeoJSON/GeoPackage (vector), CSV (statistics), PNG/SVG (figures)
- Cloud-optimized GeoTIFF export for integration with other GIS tools
- STAC (SpatioTemporal Asset Catalog) metadata export for data cataloging
- Scheduled report delivery via email

### F11: Alert & Monitoring System
- Define alert rules on any analysis output: threshold on area, count, spectral value, change magnitude
- Monitoring zones: persistent AOIs that are automatically analyzed when new imagery arrives
- Alert channels: email, Slack, Microsoft Teams, webhook, SMS
- Alert history dashboard with status tracking (new, acknowledged, resolved)
- Escalation rules for critical alerts
- Integration with incident management systems (PagerDuty, ServiceNow)

### F12: REST API & Integrations
- Full REST API for programmatic access to all platform capabilities
- Python SDK with Jupyter notebook examples
- API endpoints for: data search, analysis submission, results retrieval, alert management
- Webhook callbacks for pipeline completion and alert events
- Integration with ArcGIS Online (publish results as feature layers)
- Integration with QGIS via plugin (submit jobs, view results)
- SSO via SAML/OIDC for enterprise identity providers

---

## Technical Architecture

### System Diagram

```
Satellite Data Providers                    Users
(Sentinel, Landsat, Planet)                   │
        │                                     ▼
        ▼                              ┌─────────────┐
┌───────────────┐                      │  React SPA   │
│  Data Ingestion│                      │ (Mapbox GL,  │
│   Workers      │                      │  Deck.gl)    │
│ (download,     │                      └──────┬──────┘
│  preprocess,   │                             │
│  COG convert)  │                             ▼
└───────┬───────┘                      ┌─────────────┐
        │                              │  FastAPI     │
        ▼                              │  Gateway     │
┌───────────────┐                      └──────┬──────┘
│  S3 / Object   │◀──────────────────────────►│
│  Storage       │                             │
│  (COGs, tiles) │                      ┌──────┴──────┐
└───────────────┘                      │             │
                                 ┌─────▼───┐  ┌─────▼─────┐
                                 │ Analysis │  │ ML Model  │
                                 │ Workers  │  │ Workers   │
                                 │ (GDAL,   │  │ (PyTorch, │
                                 │ Rasterio)│  │ GPU)      │
                                 └─────┬───┘  └─────┬─────┘
                                       │             │
                                       ▼             ▼
                                 ┌─────────────────────┐
                                 │   PostgreSQL/PostGIS │
                                 │   (metadata, vectors,│
                                 │    users, pipelines) │
                                 └──────────┬──────────┘
                                            │
                                            ▼
                                 ┌─────────────────────┐
                                 │   Redis (job queue,  │
                                 │   caching, sessions) │
                                 └─────────────────────┘
```

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | React 18 + TypeScript, Mapbox GL JS, Deck.gl (WebGL overlays), TailwindCSS |
| Map Tiles | Titiler (dynamic COG tile server), MapProxy for caching |
| Backend API | Python 3.12 / FastAPI, async with uvicorn |
| Geospatial Processing | GDAL 3.8, Rasterio, Shapely, Fiona, pyproj |
| ML / Computer Vision | PyTorch 2.x, torchvision, segmentation-models-pytorch, scikit-learn |
| Database | PostgreSQL 16 + PostGIS 3.4 (spatial queries, vector storage) |
| Object Storage | AWS S3 (imagery as COGs), CloudFront CDN for tile delivery |
| Job Queue | Celery + Redis (analysis jobs), Kubernetes Jobs for GPU workloads |
| Infrastructure | Kubernetes (EKS) with GPU node groups (NVIDIA A10G), Terraform for IaC |
| Monitoring | Prometheus + Grafana (infra), Sentry (errors), OpenTelemetry (tracing) |

### Data Model

```
Organization
├── id, name, slug, plan, created_at
│
├── User[]
│   ├── id, org_id, email, role (admin/analyst/viewer)
│   └── preferences (default_basemap, default_bands, notification_settings)
│
├── AOI[] (Areas of Interest)
│   ├── id, org_id, name, geometry (PostGIS POLYGON/MULTIPOLYGON)
│   ├── monitoring_enabled, monitoring_frequency
│   └── alert_rules[] (threshold, metric, channel)
│
├── ImagerySource[]
│   ├── id, org_id, provider (sentinel2/landsat/planet/custom)
│   ├── scene_id, acquisition_date, cloud_cover_pct
│   ├── bounds (PostGIS POLYGON), crs, resolution_m
│   ├── storage_path (S3 key to COG), file_size_bytes
│   └── preprocessing_status (raw/corrected/ready)
│
├── Pipeline[]
│   ├── id, org_id, name, description, version
│   ├── definition (JSONB — DAG of processing nodes)
│   ├── schedule (cron expression), is_active
│   ├── created_by, updated_at
│   └── template_id (nullable, if cloned from template)
│
├── PipelineRun[]
│   ├── id, pipeline_id, status (queued/running/completed/failed)
│   ├── started_at, completed_at, duration_seconds
│   ├── parameters (JSONB — date range, AOI, thresholds)
│   ├── node_results[] (per-node status, output references)
│   └── error_message, retry_count
│
├── AnalysisResult[]
│   ├── id, pipeline_run_id, org_id, result_type (raster/vector/statistics)
│   ├── storage_path (S3 key), format (geotiff/geojson/csv)
│   ├── bounds, crs, created_at
│   └── metadata (JSONB — band info, model name, accuracy metrics)
│
├── MLModel[]
│   ├── id, org_id, name, model_type (classification/detection/segmentation)
│   ├── architecture (unet/faster_rcnn/random_forest)
│   ├── training_data_ref, accuracy_metrics (JSONB)
│   ├── model_artifact_path (S3), version, is_deployed
│   └── created_at, trained_by
│
├── Alert[]
│   ├── id, org_id, aoi_id, rule_id
│   ├── triggered_at, status (new/acknowledged/resolved)
│   ├── severity (info/warning/critical)
│   ├── summary, details (JSONB), thumbnail_url
│   └── acknowledged_by, resolved_at
│
└── Report[]
    ├── id, org_id, pipeline_run_id
    ├── template_id, format (pdf/html), storage_path
    └── generated_at, shared_link_token
```

### API Design

```
Auth:
POST   /api/v1/auth/login                    # Email/password login
POST   /api/v1/auth/sso                       # SAML/OIDC SSO flow
POST   /api/v1/auth/token/refresh             # Refresh JWT token

Imagery:
GET    /api/v1/imagery/search                 # Search available scenes (provider, date, bbox, cloud%)
POST   /api/v1/imagery/order                  # Order commercial imagery
POST   /api/v1/imagery/upload                 # Upload custom imagery
GET    /api/v1/imagery/:id                    # Get scene metadata
GET    /api/v1/imagery/:id/tiles/{z}/{x}/{y}  # Dynamic tile endpoint (COG)
GET    /api/v1/imagery/:id/thumbnail          # Quick preview thumbnail
GET    /api/v1/imagery/:id/stats              # Band statistics (min, max, mean, histogram)

AOIs:
GET    /api/v1/aois                           # List areas of interest
POST   /api/v1/aois                           # Create AOI (GeoJSON geometry)
PATCH  /api/v1/aois/:id                       # Update AOI properties
DELETE /api/v1/aois/:id                       # Delete AOI
POST   /api/v1/aois/:id/monitoring            # Enable/disable monitoring
GET    /api/v1/aois/:id/imagery               # List available imagery for AOI

Pipelines:
GET    /api/v1/pipelines                      # List pipelines
POST   /api/v1/pipelines                      # Create pipeline (DAG definition)
GET    /api/v1/pipelines/:id                  # Get pipeline details
PATCH  /api/v1/pipelines/:id                  # Update pipeline definition
DELETE /api/v1/pipelines/:id                  # Delete pipeline
POST   /api/v1/pipelines/:id/run              # Execute pipeline
GET    /api/v1/pipelines/:id/runs             # List past runs
GET    /api/v1/pipelines/templates            # Browse pipeline templates

Analysis:
POST   /api/v1/analysis/spectral-index        # Compute spectral index
POST   /api/v1/analysis/change-detection      # Run change detection
POST   /api/v1/analysis/classification        # Run LULC classification
POST   /api/v1/analysis/object-detection      # Run ML object detection
GET    /api/v1/analysis/jobs/:id              # Get job status and results
GET    /api/v1/analysis/results/:id           # Download result (GeoTIFF, GeoJSON)

Models:
GET    /api/v1/models                         # List available ML models
POST   /api/v1/models/train                   # Start custom model training
GET    /api/v1/models/:id                     # Get model details and metrics
POST   /api/v1/models/:id/predict             # Run inference with model
DELETE /api/v1/models/:id                     # Delete custom model

Alerts:
GET    /api/v1/alerts                         # List alerts
PATCH  /api/v1/alerts/:id                     # Update alert status
POST   /api/v1/alerts/rules                   # Create alert rule
GET    /api/v1/alerts/rules                   # List alert rules

Reports:
POST   /api/v1/reports/generate               # Generate report from results
GET    /api/v1/reports/:id                    # Download report
GET    /api/v1/reports/:id/share              # Get shareable link

Export:
POST   /api/v1/export                         # Export results in specified format
GET    /api/v1/export/:id/status              # Check export job status
GET    /api/v1/export/:id/download            # Download exported file
```

---

## UI/UX — Key Screens

### 1. Map Workspace (Main View)
- Full-screen interactive map with imagery layers, analysis overlays, and drawing tools
- Left sidebar: layer panel with toggleable layers, opacity sliders, band combination selector
- Top toolbar: search (location/coordinates), measurement tools, screenshot, share map link
- Bottom timeline slider: scrub through available imagery dates, play animation of temporal changes

### 2. Data Catalog & Search
- Filterable gallery of available satellite scenes for the current AOI
- Filters: date range, cloud cover (slider 0-100%), data provider, spatial resolution
- Scene preview thumbnails with metadata overlay (date, cloud %, resolution)
- Bulk select and add to analysis queue

### 3. Pipeline Builder (Visual Editor)
- Canvas workspace with node-based visual programming interface
- Left panel: toolbox of available processing nodes grouped by category (Input, Spectral, ML, Filter, Output)
- Drag nodes onto canvas, connect input/output ports with wires
- Click node to configure parameters in right panel
- Run button with real-time execution progress overlay on each node
- Save, version, and share pipelines

### 4. Analysis Results Dashboard
- Grid/list view of completed analysis runs with status indicators
- Click to expand: result preview map, summary statistics, accuracy metrics
- Comparison mode: select two results to view side by side
- Quick actions: download, generate report, share link, re-run with different parameters

### 5. Crop Monitoring Dashboard
- Farm/field overview map with per-field health color coding (green/yellow/red based on NDVI anomaly)
- Click field: time-series NDVI chart, growth stage indicator, historical comparison, weather overlay
- Alert feed: fields with detected anomalies, sorted by severity
- Seasonal summary statistics and yield forecast panel

### 6. Alert Management Console
- Table of active alerts with severity badges, timestamp, AOI name, and trigger condition
- Map view: alerts plotted geographically with cluster markers
- Click alert: before/after imagery comparison, change magnitude statistics, recommended actions
- Bulk operations: acknowledge, assign, resolve, suppress

---

## Monetization

### Free Tier
- 5 km2 total AOI area
- Sentinel-2 and Landsat data only (free public data)
- 3 pre-built spectral indices (NDVI, NDWI, NDBI)
- 5 analysis runs per month
- Basic map viewer
- Community support only

### Analyst — $149/month
- 500 km2 total AOI area
- All free satellite data sources + bring your own commercial imagery
- Full spectral index library (40+ indices)
- Unlimited analysis runs
- 3 pre-built ML models (building detection, LULC classification, change detection)
- No-code pipeline builder (up to 5 saved pipelines)
- PDF report generation
- Export in all formats
- Email support

### Team — $599/month
- 5,000 km2 total AOI area
- Up to 10 users with role-based access control
- All pre-built ML models
- Custom model training (5 training jobs/month, GPU time included)
- Unlimited saved pipelines with scheduling
- Alert/monitoring system (10 active monitoring zones)
- API access (10K calls/month)
- Slack/Teams integration
- Priority support with 24-hour response SLA

### Enterprise — Custom
- Unlimited AOI area
- Unlimited users with SSO/SAML
- Unlimited custom model training with dedicated GPU allocation
- Commercial satellite data procurement assistance
- On-premises deployment option (Kubernetes Helm chart)
- Custom model development and consulting
- Dedicated account manager and onboarding
- SLA with 99.9% uptime guarantee
- Data residency and compliance (FedRAMP, ITAR for defense)

---

## Go-to-Market Strategy

### Phase 1: Community & Agriculture (Month 1-3)
- Launch free tier targeting agricultural analysts and environmental researchers
- Publish tutorials: "Monitor crop health with free satellite data" — SEO play for precision agriculture keywords
- Offer free academic licenses to 50 university remote sensing labs
- Submit to r/gis, r/remotesensing, GIS StackExchange
- Present at FOSS4G and AGU conferences (abstract submission)
- Open-source the Python SDK to build developer community

### Phase 2: Vertical Expansion (Month 3-6)
- Launch crop monitoring dashboard as purpose-built vertical solution for agriculture
- Partner with 3-5 agricultural cooperatives for pilot programs
- Add defense/intelligence-focused features (secure deployment, audit logging)
- Content marketing: case studies showing ROI vs. ArcGIS (cost savings, time savings)
- Webinar series: "Satellite Analytics Without a GIS Degree"
- Launch on Product Hunt and Hacker News

### Phase 3: Enterprise & Partnerships (Month 6-12)
- Enterprise sales team targeting defense contractors, mining companies, insurance firms
- Partner with satellite data providers (Planet, Maxar) for bundled data + analytics offerings
- Launch marketplace for community-contributed ML models and pipeline templates
- SOC 2 Type II certification for enterprise trust
- FedRAMP authorization process for US government contracts
- Expand to SAR imagery analysis (Sentinel-1) for all-weather monitoring

### Acquisition Channels
- Organic search: target "satellite image analysis software", "NDVI mapping tool", "change detection software"
- Academic word-of-mouth: free tier for researchers who publish and cite TerraScan
- Satellite data provider referrals (Planet/Maxar recommend TerraScan to their customers who need analytics)
- Industry conference presence (AGU, IGARSS, Precision Ag Conference, GEOINT)
- LinkedIn content targeting GIS managers and remote sensing directors

---

## Success Metrics & KPIs

| Metric | Target (Month 6) | Target (Month 12) |
|--------|-------------------|---------------------|
| Registered users | 2,000 | 10,000 |
| Active AOIs monitored | 500 | 5,000 |
| Analysis pipeline runs/month | 10,000 | 100,000 |
| Paying customers | 40 | 200 |
| MRR | $12,000 | $80,000 |
| Free → Paid conversion | 4% | 7% |
| Imagery processed (TB/month) | 5 TB | 50 TB |
| Churn rate | <6%/mo | <4%/mo |

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | TerraScan Advantage |
|-----------|-----------|------------|-------------------|
| ESRI ArcGIS Pro + Image Analyst | Industry standard, massive feature set, enterprise trust | $3K+/seat/year, desktop-only, steep learning curve, no native ML pipeline | Cloud-native, 10x cheaper, no-code ML pipelines, instant collaboration |
| Google Earth Engine | Massive data catalog, free for research, planetary-scale compute | Requires coding (JavaScript/Python), no visual builder, commercial licensing unclear and expensive | No-code pipeline builder, clear commercial pricing, purpose-built UI |
| Descartes Labs | Advanced ML on satellite imagery, strong data platform | Enterprise-only pricing ($100K+), not accessible to small teams, opaque | Self-serve pricing starting at $149/mo, transparent, accessible to individuals |
| Planet Analytics | Daily global imagery (unique asset), built-in analytics feeds | Analytics limited to pre-built feeds (no custom models), imagery lock-in | Bring any data source, custom model training, no vendor lock-in |
| Maxar (formerly DigitalGlobe) | Highest resolution commercial imagery (30cm), GEOINT pedigree | Analytics is secondary to data sales, expensive, government-focused | Analytics-first platform, works with any imagery including Maxar's |

---

## Risks & Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| GPU compute costs for ML inference erode margins | High | High | Optimize models with quantization/pruning, use spot instances, cache results, tiered compute limits per plan |
| Satellite data providers restrict API access or raise prices | High | Medium | Support multiple providers, cache downloaded imagery, partner agreements with data providers |
| Low accuracy of pre-built ML models in unfamiliar geographies | Medium | High | Train on diverse global datasets, allow custom model fine-tuning, set clear accuracy expectations per region |
| Enterprise customers require on-premises deployment | Medium | Medium | Build on Kubernetes from day one, offer Helm chart deployment, maintain air-gapped capability |
| Regulatory restrictions on satellite imagery in certain countries | Medium | Medium | Implement geographic access controls, comply with export regulations, work with legal counsel per market |

---

## MVP Scope (v1.0)

### In Scope
- Sentinel-2 data ingestion with automated preprocessing (atmospheric correction, cloud masking)
- Interactive map viewer with band combination selector and basic drawing tools
- 5 core spectral indices: NDVI, NDWI, NDBI, EVI, NBR
- Bi-temporal change detection (image differencing with threshold)
- Pre-trained building footprint detection model
- Simple LULC classification (6 classes)
- GeoTIFF and GeoJSON export
- User authentication and basic organization management

### Out of Scope (v1.1+)
- Commercial satellite data integration (Planet, Maxar)
- No-code pipeline builder (v1.0 uses simple form-based analysis submission)
- Custom model training
- Alert/monitoring system
- Time-series change detection (BFAST, LandTrendr)
- Crop health monitoring dashboard
- API access and Python SDK
- Report generation
- SAR imagery (Sentinel-1) support
- On-premises deployment

### MVP Timeline: 10 weeks
- Week 1-2: Data ingestion pipeline — Sentinel-2 search API integration, download workers, COG conversion, S3 storage, PostGIS metadata catalog
- Week 3-4: Map viewer — React app with Mapbox GL JS, dynamic tile serving from COGs via Titiler, band combination controls, AOI drawing tools
- Week 5-6: Spectral analysis — index computation engine with Rasterio/NumPy, result visualization as map overlays, basic change detection (image differencing)
- Week 7-8: ML models — deploy pre-trained building detection (Faster R-CNN) and LULC classification (U-Net) models, inference API with GPU workers, vector result display
- Week 9-10: Auth, export, polish — user/org management, result export (GeoTIFF/GeoJSON), billing integration (Stripe), documentation, beta launch preparation
