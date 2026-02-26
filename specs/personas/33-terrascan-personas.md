# TerraScan — Persona Subspecs

> Parent spec: [`specs/33-terrascan.md`](../33-terrascan.md)

---

## Experience-Level Personas

### Student/Academic
**Modified UI/UX:**
- Guided analysis mode with step-by-step instructions for common remote sensing workflows (NDVI calculation, land classification, change detection)
- Inline educational overlays explaining spectral band combinations, vegetation indices, and classification algorithms
- Simplified map viewer with auto-scaled legends and pre-configured basemaps for common study regions

**Feature Gating:**
- Access to free satellite data sources (Sentinel-2, Landsat) with pre-processed imagery (atmospherically corrected, cloud-masked)
- ML classification limited to pre-built models (land cover, water body detection); custom model training disabled
- Processing area capped at 500 km² per analysis; 5 concurrent pipeline runs

**Pricing Tier:** Free tier (academic email verification)

**Onboarding Flow:**
- Tutorial: select a region of interest, load Sentinel-2 imagery, compute NDVI, and classify land cover in under 15 minutes
- Pre-loaded sample datasets (Amazon deforestation, urban expansion, agricultural parcels) for immediate exploration
- Prompt to join a course workspace for collaborative class projects

**Key Workflows:**
- Compute vegetation indices (NDVI, NDWI, EVI) for environmental science coursework
- Perform supervised land cover classification using training samples drawn on the map
- Run before/after change detection on multi-temporal imagery for case study analysis
- Export classified maps and statistics for research papers and presentations
- Collaborate with classmates on shared analysis projects with instructor visibility

---

### Junior Engineer (0-3 years)
**Modified UI/UX:**
- Pipeline builder with template library: drag-and-drop pre-configured analysis steps for common workflows
- Contextual help explaining each processing step's parameters and their impact on output quality
- Side-by-side before/after map comparison view with synchronized pan and zoom

**Feature Gating:**
- Access to Sentinel-2, Landsat, and commercial imagery (Planet, Maxar) via marketplace
- Pre-built and fine-tunable ML models for object detection and classification; custom model upload enabled
- Processing up to 10,000 km² per analysis; 20 concurrent pipeline runs

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Set up first monitoring project: define area of interest, select data sources, schedule automated ingestion
- Build a change detection pipeline from template, customize thresholds, and review alerts
- Connect to team workspace and configure result sharing with stakeholders

**Key Workflows:**
- Build automated satellite monitoring pipelines for periodic change detection over project areas
- Fine-tune pre-built ML models on project-specific training data for improved accuracy
- Generate NDVI time-series analytics and crop health reports for agricultural clients
- Create shareable web map dashboards for non-technical stakeholders
- Document analysis methodology and quality metrics for project deliverables

---

### Senior/Expert
**Modified UI/UX:**
- Code-first interface with Python/SQL console alongside visual pipeline builder for hybrid workflows
- Multi-sensor fusion workspace for combining optical, SAR, and multispectral data in unified analyses
- Custom model training studio with GPU-accelerated hyperparameter tuning and validation metrics

**Feature Gating:**
- All features unlocked: custom ML model training, SAR processing, multi-sensor fusion, API access
- Unlimited processing area and concurrency; priority GPU queue for model training
- Admin controls for data governance, model registry, and team pipeline management

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Import existing analysis scripts (Python/GDAL) and convert to reproducible platform pipelines
- API integration walkthrough for connecting TerraScan to existing GIS infrastructure and data lakes
- Configure team model registry and establish ML model versioning and validation protocols

**Key Workflows:**
- Develop custom ML models for novel detection tasks (illegal mining, infrastructure damage, ship detection)
- Architect multi-source data fusion pipelines combining SAR, optical, and elevation data
- Build production-grade automated monitoring systems with alert rules and escalation workflows
- Manage model versioning, A/B testing, and performance monitoring across deployments
- Lead technical architecture for large-scale geospatial analytics programs

---

## Role-Based Personas

### Designer/Creator
**Modified UI/UX:**
- Visual pipeline canvas as primary workspace with drag-and-drop processing blocks and wiring
- Map composition tools for creating polished cartographic outputs with custom symbology and annotations
- Dashboard builder for assembling interactive web map applications from analysis results

**Feature Gating:**
- Full pipeline builder access with all processing blocks and custom block authoring
- Map styling and dashboard creation tools; embed codes for external publishing
- Template creation for reusable pipeline designs shared across the organization

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Build a sample analysis pipeline from scratch: ingest, preprocess, analyze, visualize
- Create a shareable web map dashboard with interactive layers and time sliders
- Tutorial on designing reusable pipeline templates for team-wide standardization

**Key Workflows:**
- Design end-to-end geospatial analysis pipelines from data ingestion to visualization
- Create interactive map dashboards for stakeholders with drill-down analytics
- Build reusable pipeline templates for recurring monitoring and reporting workflows
- Compose multi-layer map products combining satellite imagery, analysis results, and contextual data
- Publish analysis results as web map services for integration into client GIS systems

---

### Analyst/Simulator
**Modified UI/UX:**
- Analysis workbench with multi-panel layout: imagery viewer, spectral profile explorer, classification results, and statistics dashboard
- Time-series analysis tools with interactive charts for tracking change metrics over periods
- Accuracy assessment panel with confusion matrix, kappa statistics, and F1 scores for classification results

**Feature Gating:**
- Full access to all analysis algorithms, ML models, and statistical tools
- Custom training data management and model fine-tuning capabilities
- Batch analysis across multiple regions and time periods; automated report generation

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Load a multi-temporal dataset and walk through change detection analysis with accuracy assessment
- Configure spectral analysis tools and band combination presets for the user's domain
- Set up automated analytics schedules with alert thresholds for monitoring projects

**Key Workflows:**
- Perform detailed land cover classification with accuracy assessment and uncertainty quantification
- Run multi-temporal change detection and generate time-series trend reports
- Analyze spectral signatures to identify materials, crop types, or environmental indicators
- Validate ML model outputs against ground truth data and report accuracy metrics
- Generate detailed analytical reports with statistical summaries for clients and regulators

---

### Manufacturing/Process
**Modified UI/UX:**
- Operations monitoring dashboard showing active data pipelines, processing queue status, and throughput metrics
- Data quality control panel with automated checks for cloud cover, co-registration accuracy, and radiometric consistency
- Batch processing manager for scheduling large-scale systematic processing campaigns

**Feature Gating:**
- Access to pipeline execution, monitoring, and quality control tools
- Data catalog management and ingestion scheduling; bulk processing orchestration
- Pipeline editing restricted to approved templates; new pipeline design requires designer approval

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Configure data ingestion schedules for primary satellite data sources
- Set up quality control thresholds and automated rejection criteria for incoming imagery
- Walk through batch processing setup for a large-area systematic mapping campaign

**Key Workflows:**
- Manage systematic satellite data ingestion and preprocessing pipelines at scale
- Monitor data quality metrics and flag imagery that fails radiometric or geometric quality thresholds
- Execute large-scale batch processing campaigns across continental-scale areas of interest
- Track processing throughput, queue depth, and compute resource utilization
- Coordinate data delivery to downstream analysis teams with quality certifications

---

### Regulatory/Compliance
**Modified UI/UX:**
- Evidence dashboard showing analysis provenance: data sources, processing chain, model versions, and analyst actions
- Compliance annotation tools for marking findings with regulatory references and confidence levels
- Chain-of-custody viewer for tracking satellite data from acquisition through processing to final report

**Feature Gating:**
- Read access to all analyses and results; write access to compliance annotations and review decisions
- Audit trail with immutable logging of all data access, processing, and reporting activities
- Data retention management tools for enforcing regulatory hold and deletion policies

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Configure applicable regulatory frameworks (environmental monitoring mandates, reporting requirements)
- Walk through an evidentiary analysis chain from satellite capture to compliance report
- Set up data retention policies and access control rules aligned with regulatory requirements

**Key Workflows:**
- Review and validate environmental monitoring analyses for regulatory submission (deforestation reports, emissions monitoring)
- Verify analysis provenance and chain of custody for legally defensible satellite-based evidence
- Ensure data handling complies with privacy and geospatial data regulations (national security restrictions, GDPR for location data)
- Manage regulatory reporting schedules and track submission deadlines across monitoring programs
- Audit analysis methodologies for consistency with approved standard operating procedures

---

### Manager/Decision-maker
**Modified UI/UX:**
- Executive dashboard with summary maps, trend KPIs, and alert status across all monitoring programs
- Portfolio view showing project status, budget spend, area coverage, and analysis delivery timelines
- Cost analytics panel tracking satellite imagery spend, compute usage, and per-project costs

**Feature Gating:**
- Read-only access to dashboards, summary reports, and alert notifications; no direct analysis tools
- Team management: user provisioning, project creation, data source budget allocation
- Approval workflows for large imagery purchases and high-compute processing requests

**Pricing Tier:** Enterprise tier (included in admin seat)

**Onboarding Flow:**
- Dashboard walkthrough: monitoring program health, alert summary, coverage maps, budget tracking
- Configure project portfolio with KPIs, budget limits, and reporting cadences
- Set up stakeholder distribution lists for automated report delivery

**Key Workflows:**
- Monitor the health and coverage of active satellite monitoring programs across the organization
- Track imagery and compute spending against project budgets and forecast future costs
- Review high-priority alerts (deforestation detected, construction activity, environmental anomalies) and assign follow-up
- Allocate resources and priorities across competing monitoring programs based on business impact
- Present satellite analytics insights to executive leadership and external stakeholders using summary dashboards
