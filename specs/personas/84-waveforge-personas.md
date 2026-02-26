# WaveForge — Persona Subspecs

> Parent spec: [`specs/84-waveforge.md`](../84-waveforge.md)

---

## Experience-Level Personas

### Student/Academic
**Modified UI/UX:**
- Simplified 2D wave domain setup with preset bathymetry profiles (planar beach, reef, harbor) and labeled boundary conditions
- Animated wave field visualization with overlaid vectors showing wave height, direction, and breaking zones
- Interactive parameter panel with sliders for wave period, height, and direction to observe spectral response in real time

**Feature Gating:**
- 2D spectral wave model (SWAN-equivalent) on structured grids up to 50k cells
- Idealized bathymetry inputs only; real-data import (GeoTIFF, NetCDF) gated
- Time-domain simulation limited to 24-hour storm events; long-term climate runs locked

**Pricing Tier:** Free tier

**Onboarding Flow:**
- Tutorial: set up wave propagation over a sloping beach, observe shoaling, refraction, and breaking
- Pre-loaded examples (harbor resonance, island diffraction, reef wave transformation)
- Linked references to Dean & Dalrymple, Holthuijsen textbook chapters for each physical process

**Key Workflows:**
- Simulate monochromatic and spectral wave propagation over idealized bathymetry
- Observe and quantify shoaling, refraction, diffraction, and depth-limited breaking
- Extract wave statistics (Hs, Tp, Dm) at output points and compare with analytical solutions
- Visualize spectral evolution along a transect from offshore to nearshore

---

### Junior Engineer (0-3 years)
**Modified UI/UX:**
- Project workspace with tabs: Bathymetry/Mesh | Wave Conditions | Simulation | Post-Processing
- Integrated tide and wind forcing panels with connections to public data sources (NOAA, Copernicus)
- Automated mesh generation with quality diagnostics and refinement suggestions around structures

**Feature Gating:**
- Unstructured mesh support up to 500k elements; nesting and domain decomposition unlocked
- Real bathymetry import (GeoTIFF, NetCDF, LAS) with datum transformation tools
- Wave-current interaction module enabled; full morphodynamic coupling locked

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Import bathymetry survey, generate unstructured mesh, set up a hindcast simulation using ERA5 wave boundary conditions
- Calibrate model against measured buoy data from a public archive (CDIP, NDBC)
- Produce a wave rose and extreme value analysis for a coastal site

**Key Workflows:**
- Set up a regional wave model with nested high-resolution coastal domains
- Perform wave hindcast using ERA5/GFS forcing and validate against buoy measurements
- Compute nearshore wave transformation for coastal structure design wave conditions
- Generate wave roses, exceedance tables, and spectral summaries for engineering reports
- Evaluate wave penetration into harbor layouts and identify agitation hot spots

---

### Senior/Expert
**Modified UI/UX:**
- Multi-model orchestration dashboard managing coupled wave-current-morphology simulations
- Scripting console (Python) with full API for automated model setup, calibration, and ensemble runs
- Custom physics panel for configuring source terms, dissipation formulations, and nonlinear interactions

**Feature Gating:**
- All features unlocked: phase-resolving (Boussinesq), spectral, coupled hydrodynamic-morphodynamic models
- Unlimited mesh size, domain nesting levels, and simulation duration
- HPC cluster offload, ensemble management, and plugin SDK for custom source-term formulations

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Migration wizard importing existing SWAN/MIKE21/Delft3D model setups and calibration databases
- HPC cluster configuration and API key provisioning for automated pipeline workflows
- Workshop on custom source-term development and model coupling architecture

**Key Workflows:**
- Coupled wave-current-morphology simulation for coastal evolution and sediment transport studies
- Phase-resolving Boussinesq modeling for harbor resonance and wave-structure interaction
- Ensemble wave climate projections integrating sea-level rise and climate-change wind scenarios
- Operational forecasting system setup with automated data ingestion, model execution, and alert generation
- Multi-fidelity model chaining: global spectral to regional to local phase-resolving

---

## Role-Based Personas

### Designer/Creator
**Modified UI/UX:**
- Visual bathymetry and structure layout editor with drag-and-drop breakwater, groin, and revetment placement
- Real-time wave penetration preview as structures are repositioned
- Comparison view for evaluating alternative structure layouts on wave reduction performance

**Feature Gating:**
- Structure definition and layout tools fully enabled
- Wave simulation runs in rapid-preview mode for interactive design; full-fidelity on demand
- Export to CAD (DXF) and GIS (shapefile, GeoJSON) formats for engineering drawings

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Import site bathymetry and define design wave conditions
- Place a breakwater and immediately preview wave penetration into the harbor basin
- Iterate on layout and compare alternatives using wave agitation metrics

**Key Workflows:**
- Design harbor layouts and evaluate wave agitation under multiple storm directions
- Optimize breakwater alignment, crest elevation, and gap width for target wave reduction
- Evaluate coastal protection alternatives (groin field, detached breakwater, reef) on shoreline response
- Produce wave condition maps for overtopping assessment at seawall locations

---

### Analyst/Simulator
**Modified UI/UX:**
- Data-centric workspace with time-series plots, spectral viewers, and spatial field analysis tools
- Model calibration panel with parameter sensitivity displays and goodness-of-fit metrics
- Batch simulation manager for parametric studies and ensemble climate runs

**Feature Gating:**
- Full solver access with all physics modules and calibration tools
- Extreme value analysis and joint probability tools (wave height, period, water level) enabled
- API access for automated hindcast/forecast pipelines

**Pricing Tier:** Professional or Enterprise tier

**Onboarding Flow:**
- Set up a validated hindcast model: import data, run simulation, compare to measurements, calibrate
- Perform an extreme value analysis using the hindcast output for design wave estimation
- Automate a multi-scenario study varying sea-level and storm intensity

**Key Workflows:**
- Multi-decade wave hindcast production and validation against measured data
- Extreme wave climate analysis: return period estimation, joint probability of Hs-Tp-direction
- Nearshore wave transformation matrix generation for probabilistic design methods
- Climate change impact assessment on wave climate and coastal flooding
- Operational wave forecasting with data assimilation and ensemble uncertainty quantification

---

### Manufacturing/Process
**Modified UI/UX:**
- Construction-focused views: armor unit placement, toe protection, and construction sequence visualization
- Material quantity extraction panels (rock tonnage, concrete armor units, geotextile area)
- Integration with procurement systems for material sourcing and logistics

**Feature Gating:**
- Wave overtopping and armor stability calculation tools enabled
- Full wave modeling locked; uses pre-computed design wave conditions as input
- Export to construction specification formats and quantity schedules

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Import design wave conditions and structure cross-section from designer
- Run armor stability and overtopping calculations per EurOtop/CIRIA standards
- Generate material quantity take-off and construction specification

**Key Workflows:**
- Calculate required armor unit size and layer thickness from design wave conditions
- Estimate overtopping rates at seawall cross-sections for construction-phase risk assessment
- Generate rock grading specifications and volume estimates for quarry procurement
- Produce construction sequence drawings for marine construction phasing

---

### Regulatory/Compliance
**Modified UI/UX:**
- Environmental impact dashboard: wave climate change, sediment transport, and habitat impact indicators
- Compliance checklist views for EIA/EIS requirements, coastal zone management regulations
- Automated reporting with regulatory-template formatting (USACE, EU WFD, national coastal acts)

**Feature Gating:**
- Environmental impact assessment modules (wave climate change, longshore drift, habitat mapping) enabled
- Model result comparison tools for before/after impact quantification
- Audit trail and model provenance tracking for regulatory defensibility

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Select applicable regulatory framework (USACE permit, EU WFD, national coastal regulations)
- Import project wave model results and run impact assessment modules
- Generate regulatory submission report with required content and formatting

**Key Workflows:**
- Quantify wave climate and sediment transport impacts of proposed coastal developments
- Generate environmental impact assessment documentation for permitting
- Assess cumulative impacts of multiple developments along a coastline
- Produce flood risk assessment reports for coastal planning applications
- Maintain model audit trail and version control for regulatory defensibility

---

### Manager/Decision-maker
**Modified UI/UX:**
- Project portfolio dashboard with site summaries, risk scores, and cost estimates
- Scenario comparison cards: alternative designs ranked by protection level, cost, and environmental impact
- Stakeholder communication materials: visualizations, summary reports, risk matrices

**Feature Gating:**
- Read-only access to all model results and assessment reports
- Decision-support tools: multi-criteria analysis, cost-benefit calculator, risk matrix
- Approval workflow and milestone tracking enabled

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Dashboard overview: project status, design alternatives, and risk assessment summaries
- Review scenario comparison tools and multi-criteria scoring methodology
- Set up notification preferences for milestone approvals and risk threshold alerts

**Key Workflows:**
- Compare coastal protection alternatives on cost, effectiveness, environmental impact, and public acceptance
- Review flood risk assessments and approve risk mitigation strategies
- Approve design milestones at project stage-gates
- Monitor construction-phase wave condition forecasts and authorize marine operations
- Generate briefing materials for funding applications and stakeholder consultations
