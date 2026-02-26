# SolarPlan — Persona Subspecs

> Parent spec: [`specs/94-solarplan.md`](../94-solarplan.md)

---

## Experience-Level Personas

### Student/Academic

**Modified UI/UX:**
- Simplified layout canvas with drag-and-drop module placement on flat, unobstructed rooftops
- Real-time energy yield estimator updating as array size and tilt are adjusted
- Educational sidebar explaining irradiance decomposition, cell physics, and loss factors

**Feature Gating:**
- Single-roof or open-field design with fixed-tilt racking only; no trackers or complex terrain
- TMY weather data from public databases (NSRDB, PVGIS); no proprietary satellite data
- Financial model limited to simple payback and LCOE; no tax equity or PPA modeling

**Pricing Tier:** Free tier

**Onboarding Flow:**
- Tutorial: design a 10 kW residential rooftop system and calculate annual yield
- Pre-loaded example projects (residential, commercial rooftop, ground-mount) with annotated results
- Integration with university course platforms for assignment submission

**Key Workflows:**
- Design a residential PV system and estimate annual energy production
- Study the effect of tilt, azimuth, and row spacing on specific yield
- Compare monocrystalline vs. polycrystalline vs. thin-film module performance
- Analyze monthly and hourly generation profiles against load data

---

### Junior Engineer (0-3 years)

**Modified UI/UX:**
- Project wizard: input site location, available area, target capacity, and auto-generate optimized layout
- Shading analysis tools with 3D obstruction modeling and horizon profile import
- Loss waterfall diagram showing each derating factor from GHI to net AC energy

**Feature Gating:**
- Full layout tools including single-axis trackers and terrain-following ground mounts
- Detailed shading analysis with 3D scene modeling and near-shading calculations
- String sizing assistant with inverter compatibility checking and NEC compliance

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Site setup assistant: import satellite imagery, define boundaries, and configure obstructions
- Guided string sizing tutorial with NEC/IEC voltage and current limit verification
- Connection to firm's component database with approved module and inverter lists

**Key Workflows:**
- Design commercial rooftop systems with detailed shading and setback analysis
- Perform string sizing optimization for central and string inverter architectures
- Generate P50/P90 energy yield estimates with uncertainty quantification
- Create bankable energy reports for project financing submissions
- Design ground-mount systems with single-axis trackers and GCR optimization

---

### Senior/Expert

**Modified UI/UX:**
- Full parametric design environment with batch simulation across weather years and degradation scenarios
- Advanced loss modeling: soiling, snow, spectral, IAM, mismatch, and bifacial gain
- Portfolio-level analysis workspace for multi-site optimization and resource assessment

**Feature Gating:**
- All modules unlocked: bifacial modeling, battery storage co-optimization, grid integration studies
- Probabilistic yield analysis with full Monte Carlo uncertainty propagation
- API access for automated design iteration, weather data integration, and financial modeling

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Migration assistant for PVsyst, HelioScope, and SAM project import
- Custom loss model configuration and calibration against operational fleet data
- API setup for integration with financial models, asset management, and weather services

**Key Workflows:**
- Perform bankable energy assessments for utility-scale projects with independent engineer review
- Model bifacial systems with detailed albedo mapping and rear-side irradiance calculation
- Co-optimize PV and battery storage for capacity firming and energy arbitrage
- Conduct portfolio-level resource assessment across multiple candidate sites
- Calibrate simulation models against operational data and quantify model uncertainty

---

## Role-Based Personas

### Designer/Creator

**Modified UI/UX:**
- Visual layout editor with module-level placement, racking configuration, and wiring design
- 3D site model with terrain, obstructions, and setback zone visualization
- Electrical single-line diagram auto-generator from physical layout

**Feature Gating:**
- Full layout design tools: module placement, racking selection, and inter-row spacing optimization
- Electrical design: string configuration, combiner boxes, inverter sizing, and transformer selection
- CAD export for construction drawings and permit packages

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Site import wizard: aerial imagery, LIDAR data, or manual boundary definition
- Module and inverter selector from component database with datasheet auto-population
- Layout template gallery for common configurations (portrait/landscape, east-west, trackers)

**Key Workflows:**
- Create optimized PV layouts maximizing energy density within site constraints
- Design electrical systems: string sizing, combiner/inverter allocation, and cable routing
- Generate permit-ready construction document packages with structural and electrical details
- Evaluate alternative racking systems (fixed, single-axis, dual-axis) for site conditions
- Produce 3D visualizations for client presentations and planning approvals

---

### Analyst/Simulator

**Modified UI/UX:**
- Simulation-centric workspace with weather data explorer, loss chain configurator, and time-series viewer
- Statistical analysis tools: P-values, uncertainty decomposition, inter-annual variability
- Model comparison dashboard for benchmarking against PVsyst, SAM, and operational data

**Feature Gating:**
- Full simulation engine with hourly/sub-hourly time-step resolution
- Advanced irradiance models: transposition, decomposition, spectral correction, and IAM
- Probabilistic framework with Monte Carlo uncertainty propagation across all loss factors

**Pricing Tier:** Professional or Enterprise tier

**Onboarding Flow:**
- Weather data source configuration: TMY selection methodology and site-adaptation procedures
- Loss model calibration tutorial using operational fleet data as benchmark
- Uncertainty framework setup: defining distributions for each input parameter

**Key Workflows:**
- Perform detailed hourly energy simulations with component-level loss modeling
- Generate P50/P75/P90 yield estimates with full uncertainty breakdown
- Analyze long-term degradation impacts and update lifetime energy projections
- Compare simulation results against operational data for model validation
- Conduct sensitivity analyses on key parameters: weather, soiling, degradation, availability

---

### Manufacturing/Process

**Modified UI/UX:**
- Bill of materials dashboard with module counts, racking quantities, and cable lengths
- Construction sequencing planner with installation crew productivity assumptions
- Quality inspection checklist generator for incoming materials and installed systems

**Feature Gating:**
- Material takeoff extraction from design layouts with procurement-ready quantities
- Construction scheduling tools with weather window analysis and resource leveling
- Commissioning test procedure generator with expected performance benchmarks

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Construction capability input: crew sizes, equipment availability, site access constraints
- Material procurement lead time configuration for scheduling integration
- Quality management system setup with inspection point definitions

**Key Workflows:**
- Extract detailed bills of materials for procurement and logistics planning
- Plan construction sequencing: grading, pile driving, racking, module installation, electrical
- Generate commissioning test procedures with expected IV curve and performance ratio benchmarks
- Track construction progress against schedule and material delivery
- Manage punch list items and warranty documentation for project handover

---

### Regulatory/Compliance

**Modified UI/UX:**
- Permitting checklist dashboard organized by jurisdiction and project phase
- Environmental compliance tracker: land use, habitat, stormwater, and glare assessments
- Interconnection study interface with utility requirements and queue position tracking

**Feature Gating:**
- Glare analysis module (SGHAT methodology) for aviation and residential impact assessment
- Environmental screening tools: wetland delineation overlay, habitat sensitivity mapping
- Grid interconnection study tools: capacity analysis, power flow impact, and protection coordination

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Jurisdiction selector loading applicable building codes, utility tariffs, and permitting requirements
- Environmental baseline data import for site screening and constraint mapping
- Interconnection application checklist generator based on utility and ISO/RTO requirements

**Key Workflows:**
- Prepare permit application packages with structural, electrical, and fire code compliance documentation
- Conduct glare analysis for projects near airports or residential properties
- Perform environmental impact assessments including visual impact and land use analysis
- Support interconnection applications with system specifications and power flow studies
- Track permit status and regulatory milestones across project portfolio

---

### Manager/Decision-maker

**Modified UI/UX:**
- Portfolio dashboard showing project pipeline status, yield forecasts, and financial KPIs
- Investment analysis tools: IRR, NPV, LCOE, and payback across portfolio scenarios
- Risk assessment matrix for development, construction, and operational phase risks

**Feature Gating:**
- Read-only access to all project designs, simulations, and compliance documentation
- Financial modeling: tax equity structures, PPA pricing, merchant revenue scenarios
- Portfolio optimization tools for site selection and capital allocation

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Portfolio import with project pipeline data, financial assumptions, and milestone schedules
- Financial model configuration: tax rates, incentives (ITC/PTC), depreciation schedules, debt terms
- Dashboard KPI setup: portfolio capacity, weighted LCOE, development pipeline value

**Key Workflows:**
- Evaluate project financial viability under various revenue and incentive scenarios
- Compare site alternatives by resource quality, interconnection cost, and development risk
- Monitor portfolio performance against yield forecasts and budget targets
- Generate investor reports with portfolio-level performance and risk metrics
- Make go/no-go decisions on project acquisition and development milestones
