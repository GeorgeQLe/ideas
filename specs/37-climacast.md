# ClimaCast — High-Resolution Microclimate Modeling for Construction and Agriculture

## Executive Summary

ClimaCast replaces expensive weather consulting engagements and coarse public forecast models with a cloud-native platform that generates hyperlocal microclimate simulations at the building and field level. By combining AI downscaling from global numerical weather prediction models with computational fluid dynamics, ClimaCast delivers actionable wind, solar, thermal, and precipitation insights to urban planners, construction companies, agricultural operations, and renewable energy developers — at a fraction of the cost of traditional approaches.

---

## Problem Statement

**The pain:**
- Public weather models (GFS, ECMWF) operate at 9-25 km grid resolution, completely missing street-level wind corridors, building shadows, frost pockets, and terrain-induced microclimates that determine real-world outcomes
- Custom WRF (Weather Research and Forecasting) consulting engagements cost $15K-$80K per site study and take 4-8 weeks to deliver, making them inaccessible for most projects
- Construction managers lose an estimated $4B/year in the US alone to weather-related delays because they lack site-specific forecasts for wind, precipitation, and temperature thresholds
- Agricultural operations make planting, irrigation, and harvest decisions on county-level forecasts that miss field-level variations caused by elevation, aspect, windbreaks, and soil moisture differences
- Renewable energy developers use generic solar irradiance and wind databases that fail to account for local shading, turbulence from terrain or structures, and microclimate variability — leading to 10-20% errors in energy yield projections

**Current workarounds:**
- Hiring meteorological consultants to run WRF/OpenFOAM studies at $200-500/hr (slow, expensive, not repeatable)
- Relying on nearest airport weather station data that may be 20-50 km away and at different elevation/terrain
- Using Autodesk CFD or manual wind tunnel studies for building-level wind analysis (requires expert operators, $10K+ software)
- Deploying temporary on-site weather stations for 6-12 months before making decisions (delays projects significantly)

**Market size:** The global weather services market is valued at $7.2B (2024) growing at 8.4% CAGR. The addressable segments — construction weather planning ($1.8B), precision agriculture weather ($2.1B), and renewable energy resource assessment ($1.4B) — represent a combined $5.3B opportunity. Cloud-based microclimate modeling specifically targets the underserved gap between free public forecasts and expensive bespoke consulting.

---

## Target Users

### Primary Personas

**1. Elena — Urban Planner / City Sustainability Director**
- Manages climate adaptation and urban heat island mitigation for a mid-size city (population 200K-1M)
- Needs to evaluate how new development proposals affect pedestrian wind comfort, heat stress, and solar access for neighboring buildings
- Currently commissions one-off CFD studies at $25K each, can only afford 2-3 per year
- Needs: rapid microclimate assessment for development review, urban heat island mapping, pedestrian comfort scoring for public spaces

**2. Carlos — Construction Project Manager**
- Runs $20-80M commercial construction projects with weather-sensitive operations (crane lifts, concrete pours, steel erection)
- Loses 15-25 working days per year to unexpected site-level weather events that general forecasts missed
- Uses generic weather apps and gut instinct to schedule critical lifts
- Needs: site-specific wind speed forecasts at crane height (40-80m), concrete pour temperature windows, precipitation probability at hourly resolution

**3. Dr. Ananya — Agricultural Operations Director**
- Oversees 5,000+ acres of mixed crop operations across varying terrain
- Frost events that hit low-lying fields but not ridgetops cost $200K+ in crop losses annually
- County extension weather data misses critical field-level variations
- Needs: field-level frost risk maps, growing degree day tracking per field, rain shadow and wind exposure analysis for crop planning

### Secondary Personas
- Renewable energy developers evaluating solar farm and wind turbine sites before committing to 20-year investments
- Architects designing naturally ventilated buildings who need to understand prevailing wind patterns at facade level
- Insurance underwriters assessing microclimate-related risk exposure for agricultural and construction portfolios
- Event planners for large outdoor venues needing wind and thermal comfort forecasts

---

## Solution Overview

ClimaCast works as follows:
1. Users define a site of interest by drawing a boundary on a map or uploading GIS coordinates, then specify the analysis type (wind, solar, thermal, frost, or comprehensive)
2. The platform ingests high-resolution terrain data (SRTM 30m, lidar DEMs where available), building footprints (OpenStreetMap, user-uploaded BIM/CAD), land use classification, and soil data to construct a 3D site model
3. Global NWP model outputs (GFS, ECMWF, HRRR) are fetched and AI-downscaled to 10-100m resolution using trained neural networks that account for terrain, surface roughness, and land use effects
4. For detailed wind flow analysis, a cloud-based CFD solver (OpenFOAM-based) runs steady-state and transient RANS simulations around buildings and terrain features, with results available within 30-90 minutes for typical sites
5. Results are delivered as interactive 3D/2D maps with time-series data, alerts for threshold exceedances, and exportable reports — continuously updating with each new forecast cycle

---

## Core Features

### F1: Location-Based Microclimate Modeling
- Define sites via map polygon drawing, coordinate upload (CSV/GeoJSON), or address search
- Automatic terrain ingestion from SRTM (30m), ASTER GDEM, and available lidar datasets (USGS 3DEP, Environment Agency UK)
- Building footprint and height import from OpenStreetMap 3D buildings, uploaded CAD/BIM (IFC, DWG), or manual placement
- Land use and surface classification from Sentinel-2 satellite imagery (10m resolution)
- Configurable simulation domain size (100m to 10km) and resolution (1m to 100m)
- Saved site library with version history for tracking changes over time

### F2: Wind Flow Simulation
- Steady-state RANS CFD (k-epsilon, k-omega SST turbulence models) for mean wind field computation around buildings and terrain
- 16-direction wind rose analysis with frequency-weighted comfort and safety metrics
- Pedestrian-level wind comfort scoring (Lawson criteria: sitting, standing, walking, uncomfortable, dangerous)
- Crane operation wind threshold alerts at user-specified heights (configurable knot/m-s thresholds per crane model)
- Venturi effect and corner acceleration identification with magnitude quantification
- Wind resource assessment for small/urban wind turbines (Weibull distribution fitting, annual energy yield estimate)
- Transient LES (Large Eddy Simulation) available for complex sites requiring gust analysis (premium tier)

### F3: Solar Irradiance Mapping
- Annual and monthly solar irradiance computation accounting for terrain shading, building shadows, and atmospheric attenuation
- Hour-by-hour sun path simulation with shadow casting from 3D building models
- Direct Normal Irradiance (DNI), Diffuse Horizontal Irradiance (DHI), and Global Horizontal Irradiance (GHI) decomposition
- Solar panel yield estimation with tilt/azimuth optimization recommendations
- Daylight hours and solar access compliance checking (e.g., BRE UK right-to-light guidelines)
- Integration with PVWatts/SAM for bankable solar energy yield reports

### F4: Urban Heat Island Analysis
- Surface temperature modeling using land use, albedo, vegetation fraction, anthropogenic heat, and building density inputs
- Nighttime urban heat island intensity mapping (UHI delta from rural baseline)
- Cool roof and green infrastructure scenario modeling (what-if: add trees, change surface materials)
- Thermal comfort indices: UTCI (Universal Thermal Climate Index), PET (Physiological Equivalent Temperature), WBGT (Wet Bulb Globe Temperature)
- Heat stress risk mapping for outdoor workers (OSHA/ISO 7243 thresholds)
- Time-of-day thermal comfort variation for public space design

### F5: Frost and Freeze Risk Prediction
- Cold air drainage modeling using terrain slope analysis and katabatic flow simulation
- Field-level frost pocket identification from DEM analysis and local temperature downscaling
- Growing season frost probability maps (first/last frost dates at field resolution)
- Active frost alerts: push notifications when site-specific conditions approach freezing thresholds
- Crop-specific frost damage risk (configurable temperature thresholds per crop type and growth stage)
- Historical frost event replay from reanalysis data (ERA5 downscaled)

### F6: Rain Shadow and Precipitation Modeling
- Orographic precipitation enhancement/reduction modeling from terrain and prevailing wind analysis
- Precipitation gradient estimation across complex terrain (valley floor vs. ridge)
- Rainfall intensity-duration-frequency (IDF) curves downscaled to site level
- Construction site drainage and runoff risk assessment during storm events
- Snow accumulation and drift modeling for building design and construction planning

### F7: Construction Weather Windows
- User-defined activity rules engine: "concrete pour requires >5C for 48hr, wind <25 km/h, no precipitation"
- Rolling 14-day forecast with go/no-go status for each defined activity
- Optimal scheduling suggestions: "best 3-day window for crane operations this month"
- Historical weather window frequency analysis: "how many suitable pour days in November, on average?"
- Integration with project scheduling tools (P6, MS Project) via CSV/API export
- Automatic delay day tracking and weather-related cost impact estimation

### F8: Crop Growing Condition Forecasts
- Growing Degree Day (GDD) accumulation tracking per field with crop-specific base temperatures
- Evapotranspiration (ET) estimation using Penman-Monteith at field resolution
- Soil moisture modeling integrating precipitation, ET, and soil type data
- Disease pressure indices (leaf wetness duration, humidity-temperature combinations)
- Spray window forecasting (wind speed, inversion risk, rain-free periods)
- Harvest readiness indicators combining GDD, soil moisture, and forecast precipitation

### F9: Weather Station Data Integration
- Connect personal weather stations (Davis, Ambient Weather) via API or WeatherLink
- Ingest local AWS networks (state mesonets, RAWS, CWOP)
- Real-time bias correction: calibrate model outputs against nearest station observations
- Gap-fill missing station data using model-observation fusion (optimal interpolation)
- Station data quality control (automated flagging of stuck sensors, outliers)

### F10: Time-Lapse Climate Visualization
- Animated wind flow visualization with particle trails over 3D terrain and buildings
- Solar shadow progression throughout the day/year
- Temperature field evolution over diurnal and seasonal cycles
- Embeddable animation exports (MP4, GIF, WebM) for presentations and reports
- Side-by-side scenario comparison animations (existing vs. proposed development)

### F11: Custom Report Generation
- Templated PDF reports for common use cases: wind comfort assessment, solar access study, frost risk analysis, construction weather plan
- Auto-populated executive summary with key findings and risk flags
- Regulatory compliance formatting (local council requirements, EIA appendices)
- White-label reports with client branding
- Appendix with methodology description, data sources, and uncertainty quantification

### F12: Collaboration and Sharing
- Project-based workspace with role-based access (viewer, editor, admin)
- Shareable interactive map links (password-protected or public)
- Comment and annotation system on specific map locations
- Version comparison for before/after development scenario analysis
- Audit trail of all simulation runs and parameter changes

---

## Technical Architecture

### System Diagram

```
                     ┌──────────────────────────────┐
                     │   Global NWP Models           │
                     │   (GFS, ECMWF, HRRR)         │
                     └──────────┬───────────────────┘
                                │
                                ▼
                     ┌──────────────────────┐
                     │  Data Ingestion       │
                     │  (Terrain, Land Use,  │
                     │   Buildings, Stations)│
                     └──────────┬───────────┘
                                │
            ┌───────────────────┼───────────────────┐
            ▼                   ▼                   ▼
  ┌─────────────────┐ ┌────────────────┐ ┌──────────────────┐
  │ ML Downscaling  │ │ CFD Solver     │ │ Solar/Thermal    │
  │ Service (GPU)   │ │ (OpenFOAM on   │ │ Compute Service  │
  │                 │ │  K8s pods)     │ │                  │
  └────────┬────────┘ └───────┬────────┘ └────────┬─────────┘
           │                  │                    │
           └──────────────────┼────────────────────┘
                              ▼
                    ┌──────────────────┐
                    │  PostGIS Database │
                    │  + Object Storage │
                    │  (S3/R2)          │
                    └────────┬─────────┘
                             │
                  ┌──────────┼──────────┐
                  ▼          ▼          ▼
            ┌──────────┐ ┌────────┐ ┌──────────┐
            │ React    │ │ API    │ │ Alert    │
            │ Frontend │ │ Gateway│ │ Service  │
            │ (Mapbox) │ │ (REST) │ │ (Push/   │
            │          │ │        │ │  Email)  │
            └──────────┘ └────────┘ └──────────┘
```

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | React 18 + TypeScript, Mapbox GL JS (3D terrain/building viz), Deck.gl (data overlays), Tailwind CSS |
| 3D Visualization | Mapbox GL JS with custom WebGL layers, Three.js for detailed building-level wind viz |
| Backend API | Python (FastAPI) for orchestration, Celery for async task management |
| CFD Solver | OpenFOAM 11 (containerized), custom meshing pipeline (snappyHexMesh), ParaView for post-processing |
| ML Downscaling | PyTorch models trained on WRF/ERA5 data, NVIDIA Triton Inference Server on GPU nodes |
| Database | PostgreSQL 16 + PostGIS 3.4 for geospatial queries, TimescaleDB for time-series weather data |
| Object Storage | AWS S3 / Cloudflare R2 for simulation output files (meshes, field data, renders) |
| Compute Cluster | Kubernetes (EKS) with GPU node pools (A100/A10G for ML), CPU spot instances for OpenFOAM |
| Message Queue | Redis + Celery for job orchestration, RabbitMQ for CFD job distribution |
| Monitoring | Prometheus + Grafana for cluster metrics, Sentry for application errors, custom job dashboard |

### Data Model

```
Organization
├── id, name, slug, plan, created_at
│
├── Project[]
│   ├── id, org_id, name, description, created_at
│   ├── site_boundary (PostGIS POLYGON geometry, SRID 4326)
│   ├── domain_config (resolution_m, domain_size_m, terrain_source)
│   │
│   ├── SiteModel[]
│   │   ├── id, project_id, version, created_at
│   │   ├── terrain_dem (raster reference, resolution)
│   │   ├── buildings (GeoJSON FeatureCollection with height attributes)
│   │   ├── land_use (raster classification)
│   │   └── surface_roughness_map (derived from land use + buildings)
│   │
│   ├── Simulation[]
│   │   ├── id, project_id, site_model_id, type (wind/solar/thermal/frost/comprehensive)
│   │   ├── status (queued/meshing/solving/post-processing/complete/failed)
│   │   ├── config_json (solver parameters, boundary conditions, time range)
│   │   ├── compute_time_seconds, node_count, cost_credits
│   │   ├── results_s3_prefix (path to output files)
│   │   └── created_at, completed_at
│   │
│   ├── ForecastSubscription[]
│   │   ├── id, project_id, update_frequency (hourly/6h/daily)
│   │   ├── parameters[] (wind_speed, temperature, precipitation, frost_risk)
│   │   └── alert_rules[] (threshold, direction, notification_channels)
│   │
│   └── Report[]
│       ├── id, project_id, simulation_ids[], template_type
│       ├── generated_pdf_url, created_at
│       └── branding_config (logo, company_name, colors)
│
├── WeatherStation[]
│   ├── id, org_id, name, location (PostGIS POINT)
│   ├── provider (davis/ambient/custom_api), api_credentials
│   ├── parameters[] (temperature, wind_speed, wind_dir, humidity, precip)
│   └── last_observation_at
│
└── Member[]
    ├── id, org_id, user_id, role (admin/editor/viewer)
    └── invited_at, accepted_at
```

### API Design

```
Auth:
POST   /api/auth/register              # Create account
POST   /api/auth/login                  # Login (JWT)
POST   /api/auth/refresh                # Refresh token

Projects:
GET    /api/projects                    # List projects
POST   /api/projects                    # Create project (with site boundary)
GET    /api/projects/:id                # Get project details
PATCH  /api/projects/:id                # Update project metadata
DELETE /api/projects/:id                # Archive project

Site Models:
POST   /api/projects/:id/site-models              # Create/upload site model
GET    /api/projects/:id/site-models               # List site model versions
GET    /api/projects/:id/site-models/:vid          # Get specific version
POST   /api/projects/:id/site-models/:vid/buildings  # Upload building geometry
GET    /api/projects/:id/site-models/:vid/terrain  # Get terrain DEM (GeoTIFF)

Simulations:
POST   /api/projects/:id/simulations               # Submit simulation job
GET    /api/projects/:id/simulations                # List simulations
GET    /api/simulations/:sid                        # Get simulation status/results
GET    /api/simulations/:sid/results/wind-field     # Wind field data (vector tiles)
GET    /api/simulations/:sid/results/solar-map      # Solar irradiance raster
GET    /api/simulations/:sid/results/comfort-map    # Pedestrian comfort scores
GET    /api/simulations/:sid/results/timeseries     # Point time-series extraction
DELETE /api/simulations/:sid                        # Cancel/delete simulation

Forecasts:
GET    /api/projects/:id/forecast                   # Current downscaled forecast
GET    /api/projects/:id/forecast/history            # Historical forecast data
POST   /api/projects/:id/forecast/subscribe          # Set up forecast alerts
GET    /api/projects/:id/weather-windows             # Construction weather windows

Weather Stations:
POST   /api/stations                                # Register weather station
GET    /api/stations                                # List stations
GET    /api/stations/:id/observations               # Get observation data
POST   /api/stations/:id/sync                       # Force data sync

Reports:
POST   /api/projects/:id/reports                    # Generate report
GET    /api/reports/:rid                             # Get report status/download
GET    /api/reports/:rid/pdf                         # Download PDF

Sharing:
POST   /api/projects/:id/share                      # Create share link
GET    /api/shared/:token                            # Access shared project (public)
```

---

## UI/UX — Key Screens

### 1. Project Dashboard
- Map-centric view showing all projects as pins/polygons on a global map
- Project cards with thumbnail of most recent simulation result, status badge, last updated timestamp
- Quick-create button with address search or map polygon drawing tool
- Filter/sort by project type (construction, agriculture, renewable energy, urban planning)

### 2. Site Model Editor
- 3D interactive map (Mapbox GL) showing terrain with imported/detected buildings extruded to height
- Layer toggle panel: terrain contours, buildings, land use classification, surface roughness
- Drawing tools for adding/editing building footprints, vegetation zones, and surface types
- Side panel with site metadata, coordinate reference system, and data source attribution
- "Upload" button for IFC/DWG/Shapefile import with automatic georeferencing

### 3. Simulation Configuration & Launch
- Step wizard: select analysis type (wind/solar/thermal/frost/comprehensive) and configure parameters
- Wind: select directions, speeds, turbulence model, output heights, comfort criteria
- Solar: select date range, time step, panel tilt/azimuth for PV analysis
- Estimated compute time and credit cost shown before submission
- Advanced panel for expert users: mesh refinement zones, boundary condition overrides, solver tolerances

### 4. Results Explorer
- Split view: 3D map with simulation results overlaid (wind vectors, solar heatmap, temperature contours) + data panel
- Interactive probing: click any point to see time-series data, statistics, and threshold exceedance counts
- Layer controls: toggle between wind speed, direction, turbulence intensity, comfort class
- Animation playback for time-varying results (diurnal solar, wind direction sweep)
- Export buttons: GeoTIFF, CSV, Shapefile, PDF report, animation (MP4)

### 5. Weather Windows Calendar
- 14-day calendar view with color-coded go/no-go status per user-defined activity
- Hoverable cells showing specific conditions (wind speed, temperature, precipitation probability)
- Activity rule editor sidebar: create/edit rules with parameter thresholds and duration requirements
- Historical frequency chart: "In a typical November, you get X suitable days for this activity"

### 6. Alert & Forecast Feed
- Real-time feed of threshold exceedance alerts across all active projects
- Mini forecast cards per project showing next 48-hour outlook with key parameters
- Notification preferences: email, SMS, push, Slack webhook per alert type
- Alert history log with acknowledged/unacknowledged status tracking

---

## Monetization

### Free Tier
- 1 project, 1 site (up to 500m x 500m domain)
- Basic weather downscaling (temperature, wind, precipitation at 100m resolution)
- 2 wind simulations per month (single direction, coarse mesh)
- 7-day forecast with daily resolution
- Community support

### Planner — $99/month
- 5 projects, sites up to 2km x 2km
- AI-downscaled forecasts at 50m resolution with hourly time steps
- 10 CFD wind simulations per month (full 16-direction rose, medium mesh)
- Solar irradiance mapping and shadow analysis
- Frost risk prediction and growing degree day tracking
- Construction weather windows (3 activity rules)
- PDF report generation (standard templates)
- Email alerts

### Team — $399/month
- 25 projects, sites up to 5km x 5km
- High-resolution downscaling (10-25m) with ensemble uncertainty
- 50 CFD simulations per month (fine mesh, transient option available)
- Urban heat island analysis and thermal comfort mapping
- Rain shadow and precipitation modeling
- Unlimited activity rules and weather windows
- Custom report templates with white-label branding
- Weather station integration (up to 10 stations)
- API access for programmatic simulation submission
- Team collaboration (up to 10 members)
- Priority compute queue and Slack/webhook alerts

### Enterprise — Custom
- Unlimited projects and simulations
- LES (Large Eddy Simulation) for complex turbulence analysis
- Custom ML model training on client's historical weather station data
- On-premise deployment option for sensitive government/defense projects
- Dedicated GPU cluster allocation with guaranteed SLA
- SSO/SAML authentication
- Custom integrations (BIM platforms, project management tools, SCADA systems)
- Dedicated solutions engineer and quarterly business reviews

---

## Go-to-Market Strategy

### Phase 1: Construction Weather Beachhead (Month 1-3)
- Target 10-15 mid-size construction companies in wind-prone regions (Chicago, Dallas, coastal cities)
- Offer free 30-day pilot with site-specific wind analysis for an active project
- Partner with 2-3 crane rental companies who have vested interest in reducing weather delays
- Publish case study: "How [Company] saved X delay days with site-level wind forecasts"
- Attend World of Concrete and ENR FutureTech conferences

### Phase 2: Agriculture & Renewable Energy Expansion (Month 3-6)
- Launch frost risk prediction for wine/fruit growers (high willingness to pay, frost = catastrophic loss)
- Partner with precision agriculture platforms (Climate Corp, Trimble Ag) for data integration
- Target solar/wind developers with bankable resource assessment reports
- SEO content: "microclimate modeling for [construction/agriculture/solar]" keyword clusters
- Integration with Davis Instruments and Ambient Weather station APIs

### Phase 3: Urban Planning & International (Month 6-12)
- Release urban heat island and pedestrian wind comfort modules
- Target city planning departments and environmental consultancies
- Pursue Eurocode and international building code compliance certifications
- Localize for UK, Australia, and EU markets (different codes, metric-first)
- Develop partnership program for weather consulting firms to white-label ClimaCast

### Acquisition Channels
- Direct sales to construction companies and renewable energy developers (high ACV justifies sales-led)
- Content marketing targeting "site weather assessment," "microclimate study," "pedestrian wind comfort" searches
- Partnerships with BIM platforms (Autodesk, Trimble) as an add-on microclimate analysis tool
- Academic licensing (free/discounted) to train the next generation of users and generate research citations
- Government RFP responses for climate adaptation and urban resilience planning

---

## Success Metrics & KPIs

| Metric | Target (Month 6) | Target (Month 12) |
|--------|-------------------|---------------------|
| Registered organizations | 200 | 800 |
| Active projects (monthly) | 400 | 2,000 |
| Simulations run (monthly) | 2,000 | 15,000 |
| Paying customers | 40 | 150 |
| MRR | $12,000 | $55,000 |
| Free-to-paid conversion | 8% | 12% |
| Simulation completion rate (success) | 92% | 97% |
| Average simulation time (wind, medium mesh) | < 45 min | < 20 min |

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | ClimaCast Advantage |
|-----------|-----------|------------|-------------------|
| Meteoblue | Established brand, good free tier, city-level microclimate data | No custom site modeling, no CFD, limited to their pre-computed grid | User-defined sites with building-level CFD and AI downscaling |
| Tomorrow.io | Strong API, real-time weather intelligence, enterprise clients | Focused on weather data delivery not simulation, no CFD/microclimate modeling | Full physics-based simulation + ML, actionable domain-specific insights |
| DTN | Deep agriculture and energy expertise, massive station network | Expensive enterprise-only, no self-service, no building-level wind modeling | Self-service SaaS with transparent pricing, cloud CFD at any scale |
| Custom WRF Consulting | Highest accuracy, fully customizable | $15K-80K per study, 4-8 week turnaround, not repeatable | 100x cheaper, results in hours, continuously updating forecasts |
| Autodesk CFD | Integrated with Revit/BIM workflows, good for HVAC | Not weather-focused, no downscaling, no forecasting, requires desktop | Purpose-built for outdoor microclimate with real weather data integration |

---

## Risks & Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| CFD simulation accuracy questioned by professional engineers | High | Medium | Validate against published wind tunnel studies (AIJ, TPU databases), offer verification reports, publish methodology whitepaper with peer review |
| GPU/compute costs erode margins on heavy simulation workloads | High | Medium | Use spot instances (60-70% savings), implement adaptive mesh refinement to reduce cell count, cache terrain/mesh for repeat simulations on same site |
| Weather model data access changes (ECMWF paywall, NOAA policy shifts) | Medium | Low | Multi-source ingestion (GFS always free), cache model data, build relationships with national met services, maintain ERA5 reanalysis backup |
| Liability concerns if users make safety-critical decisions on ClimaCast data | High | Medium | Clear disclaimers that ClimaCast is a decision-support tool not a replacement for site-specific professional assessment, professional liability insurance, PE-reviewed methodology documentation |
| Established players (Tomorrow.io, DTN) add microclimate modeling features | Medium | Medium | Move fast on domain-specific features (construction weather windows, frost risk), build switching costs through historical data and calibrated models per site |

---

## MVP Scope (v1.0)

### In Scope
- Project creation with map-based site boundary definition
- Automatic terrain and building footprint ingestion (SRTM + OpenStreetMap)
- AI-downscaled weather forecast (temperature, wind, precipitation) at 50m resolution
- Single-direction steady-state wind CFD simulation with pedestrian-level results
- Solar irradiance mapping (annual and monthly) with building shadow analysis
- Interactive 2D map results viewer with point data extraction
- Basic PDF report export

### Out of Scope (v1.1+)
- Full 16-direction wind rose analysis and comfort scoring (v1.1)
- Urban heat island and thermal comfort modeling (v1.1)
- Frost risk prediction and agricultural features (v1.1)
- Construction weather windows and scheduling integration (v1.2)
- Weather station data integration and bias correction (v1.2)
- LES transient turbulence simulation (v1.2)
- BIM/IFC import for detailed building geometry (v1.3)
- Team collaboration and sharing features (v1.3)
- White-label reports and API access (v1.3)

### MVP Timeline: 10 weeks
- Week 1-2: Site model pipeline — terrain ingestion (SRTM), building footprint extraction (OSM), PostGIS storage, Mapbox GL frontend with 3D terrain rendering
- Week 3-4: ML downscaling service — GFS data ingestion, pre-trained U-Net downscaling model deployment on GPU, forecast API endpoints, basic results visualization layer
- Week 5-7: CFD simulation pipeline — OpenFOAM containerization on Kubernetes, automated mesh generation (snappyHexMesh), single-direction RANS solve, result extraction and vector tile generation for map overlay
- Week 8-9: Solar analysis module — sun position calculation, ray-tracing shadow computation against 3D building model, irradiance integration, heatmap visualization
- Week 10: Report generation, billing integration (Stripe), onboarding flow, documentation, and launch preparation
