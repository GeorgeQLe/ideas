# ClimaCast — Persona Subspecs

> Parent spec: [`specs/37-climacast.md`](../37-climacast.md)

---

## Experience-Level Personas

### Student/Academic
**Modified UI/UX:**
- Guided workflow with step-by-step site definition: drop a pin on the map, set terrain parameters, select weather model period, and run microclimate simulation
- Inline educational annotations explaining meteorological concepts (wind shear, urban heat island, orographic effects) alongside simulation controls
- Simplified result viewer with pre-configured map overlays (wind speed, temperature, solar irradiance) and auto-generated legends

**Feature Gating:**
- Access to AI-downscaled forecasts from public models (GFS, ECMWF ERA5) at 100m resolution
- CFD microclimate simulation limited to 1 km^2 domain size and 24-hour simulation periods
- Single site analysis at a time; multi-site comparisons and batch runs disabled

**Pricing Tier:** Free tier (academic email verification)

**Onboarding Flow:**
- Tutorial: analyze the microclimate around a university campus building — wind patterns, solar exposure, and heat island effects
- Pre-loaded case studies (urban canyon wind, hillside frost pocket, coastal sea breeze) for immediate exploration
- Prompt to join a course workspace for collaborative climate analysis projects

**Key Workflows:**
- Simulate wind and temperature patterns around buildings for urban planning or atmospheric science coursework
- Analyze solar irradiance and shading patterns for a site over seasonal variations
- Compare microclimate model output against local weather station observations for validation exercises
- Generate site-level climate analysis maps for research papers and presentations
- Explore how terrain and building geometry influence local weather using parametric studies

---

### Junior Engineer (0-3 years)
**Modified UI/UX:**
- Project template library: select use case (construction weather planning, solar resource assessment, wind comfort study) to auto-configure appropriate models and outputs
- Contextual guidance on domain setup (resolution, boundary conditions, model selection) with recommended defaults per application
- Report builder with pre-formatted templates for common deliverables (wind comfort assessment, solar feasibility study)

**Feature Gating:**
- AI-downscaled forecasts at 50m resolution; CFD microclimate up to 10 km^2 domains
- Historical climate analysis and seasonal wind roses enabled; forecast-based scheduling tools available
- Up to 5 concurrent site analyses; multi-year climate statistics enabled

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Set up a construction site weather monitoring project: define location, configure wind and precipitation thresholds, generate scheduling recommendations
- Walk through a wind comfort assessment for a building proposal following Lawson criteria
- Connect to team workspace and configure client report branding

**Key Workflows:**
- Generate construction site weather forecasts with wind, precipitation, and temperature thresholds for activity planning
- Perform pedestrian wind comfort assessments for building development proposals
- Assess solar resource potential for renewable energy feasibility studies
- Create professional client reports with microclimate maps, wind roses, and recommendation summaries
- Monitor weather conditions at active project sites and trigger threshold-based alerts

---

### Senior/Expert
**Modified UI/UX:**
- Multi-model workspace with WRF mesoscale output, CFD fine-scale simulation, and statistical downscaling in coordinated views
- Custom model configuration panel: WRF physics parameterizations, CFD turbulence models, mesh refinement zones
- Python scripting console for custom post-processing, model coupling, and automated analysis pipelines

**Feature Gating:**
- All features unlocked: custom WRF configurations, full CFD solver control, API access, GPU cluster compute
- Unlimited domain size and simulation duration; ensemble forecasting with perturbation methods
- Admin controls for model configuration templates, validation datasets, and team methodology standards

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Import existing WRF namelists or OpenFOAM configurations to replicate current modeling workflows
- API walkthrough for integrating ClimaCast outputs into existing GIS, BIM, or energy modeling platforms
- Configure team model validation protocols with benchmark datasets and accuracy thresholds

**Key Workflows:**
- Configure and run custom WRF mesoscale simulations nested to CFD-scale microclimate resolution
- Develop and validate site-specific microclimate models calibrated against on-site measurement campaigns
- Architect automated weather intelligence pipelines for large-scale infrastructure or agricultural programs
- Establish and maintain modeling methodology standards for the team's consulting practice
- Lead technical reviews of microclimate assessments and validate junior engineers' results

---

## Role-Based Personas

### Designer/Creator
**Modified UI/UX:**
- Site design canvas with 3D terrain and building geometry import (CityGML, IFC, shapefile) for setting up simulation domains
- Scenario comparison workspace: modify building massing, add trees, change ground cover, re-simulate to visualize microclimate impact
- Visualization compositor for creating presentation-quality wind and thermal comfort maps with custom styling

**Feature Gating:**
- Full domain setup tools with geometry import, terrain processing, and vegetation modeling
- Scenario management for comparing design alternatives; A/B microclimate comparison enabled
- Export to BIM-compatible formats and GIS layers for integration with architectural workflows

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Import a site model (building massing from IFC or terrain from GIS) and set up a microclimate simulation domain
- Create two design scenarios (with/without wind mitigation features) and compare results side-by-side
- Tutorial on generating presentation-quality climate analysis visualizations for client meetings

**Key Workflows:**
- Set up microclimate simulation domains using imported 3D site geometry and terrain models
- Compare microclimate impacts of alternative site designs and building configurations
- Evaluate wind mitigation strategies (canopies, screens, vegetation) through simulation-based design iteration
- Create compelling visual presentations of microclimate analysis for design review meetings
- Integrate microclimate assessment results into BIM models and urban planning documents

---

### Analyst/Simulator
**Modified UI/UX:**
- Analysis workbench with time-series plots, statistical summaries, and spatial map views in coordinated panels
- Model validation tools for comparing simulation output against weather station observations and sensor data
- Climate statistics generator producing wind roses, temperature duration curves, and extreme event analyses

**Feature Gating:**
- Full access to all weather models, CFD solvers, and statistical analysis tools
- Historical climate reanalysis and extreme event modeling enabled
- Batch analysis across multiple sites and time periods; automated report generation

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Load historical weather data for a site and generate climate statistics (wind rose, temperature range, precipitation frequency)
- Run a model validation study comparing simulation output against nearby weather station records
- Configure automated climate monitoring dashboards with trend analysis

**Key Workflows:**
- Perform detailed climate analysis for site characterization including extreme wind, temperature, and precipitation statistics
- Validate microclimate model accuracy against on-site measurements and quantify uncertainty
- Run long-term climate hindcasts to establish baseline conditions for environmental impact assessments
- Analyze seasonal and diurnal microclimate variations for agricultural and energy applications
- Generate comprehensive climate assessment reports with statistical rigor for engineering decision-making

---

### Manufacturing/Process
**Modified UI/UX:**
- Operations weather dashboard showing real-time site conditions, forecast alerts, and work-window scheduling
- Threshold management panel: configure wind, temperature, and precipitation limits per construction activity (crane operations, concrete pours, welding)
- Calendar integration showing weather-feasible work windows aligned with project schedules

**Feature Gating:**
- Access to site-specific weather forecasts and threshold-based alerts
- Work-window scheduling tool with calendar export and mobile notifications
- Model configuration restricted to pre-approved site setups; new site onboarding requires analyst support

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Configure a construction site: set location, define activity-specific weather thresholds, enable alert notifications
- Walk through the work-window scheduling tool: view upcoming weather-feasible periods for weather-sensitive activities
- Set up mobile alerts for threshold exceedances during active work periods

**Key Workflows:**
- Monitor real-time and forecast weather conditions at active construction or agricultural sites
- Schedule weather-sensitive operations (crane lifts, concrete pours, spraying) based on site-specific forecasts
- Track weather-related delays and correlate with forecast accuracy for continuous improvement
- Manage weather threshold configurations across multiple active project sites
- Generate weather delay documentation with forecast evidence for contractual claims

---

### Regulatory/Compliance
**Modified UI/UX:**
- Environmental impact assessment panel with regulatory-compliant climate analysis templates (wind environment, shadow, thermal comfort)
- Standards compliance tracker linking analysis outputs to local planning authority requirements (e.g., Lawson wind comfort criteria, BREEAM, LEED)
- Audit trail viewer showing complete methodology documentation for every submitted assessment

**Feature Gating:**
- Full access to analysis results for review; write access to compliance annotations and approval stamps
- Regulatory template library with jurisdiction-specific assessment formats and criteria
- Immutable audit logs of all model configurations, input data, and submitted results

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Configure applicable local planning and environmental regulations for the practice area
- Walk through a regulatory wind comfort assessment: methodology selection, simulation, criteria evaluation, report generation
- Set up quality management workflows with peer review gates before submission

**Key Workflows:**
- Review microclimate assessments for compliance with local planning authority environmental requirements
- Verify that assessment methodology meets regulatory standards (Lawson criteria, ASHRAE, local codes)
- Ensure simulation inputs and assumptions are documented and defensible for planning hearings
- Manage the peer review and approval workflow for environmental impact submissions
- Track regulatory submission status across multiple development projects

---

### Manager/Decision-maker
**Modified UI/UX:**
- Program dashboard showing all active site assessments with status, weather risk level, and delivery timeline
- Weather impact summary: cost of weather delays across the portfolio, forecast accuracy metrics, risk exposure
- Resource allocation view: analyst workload, compute usage, and site monitoring capacity

**Feature Gating:**
- Read-only access to project dashboards, weather summaries, and risk assessments; no direct modeling tools
- Team management: user provisioning, site allocation, compute budget controls
- Approval authority for site setup and assessment methodology before client delivery

**Pricing Tier:** Enterprise tier (included in admin seat)

**Onboarding Flow:**
- Portfolio dashboard overview: active projects, weather risk heatmap, cost impact tracking
- Configure program-level KPIs (forecast accuracy targets, delay reduction goals, assessment turnaround times)
- Set up notification rules for high-risk weather events across the project portfolio

**Key Workflows:**
- Monitor weather risk exposure across all active construction and development projects
- Track weather-related cost impacts and measure ROI of microclimate forecasting against delay reduction
- Allocate analyst resources and compute budget across projects based on complexity and deadline priority
- Review and approve microclimate assessment deliverables before client submission
- Present weather risk management strategy and value metrics to executive leadership and clients
