# WindFarm — Persona Subspecs

> Parent spec: [`specs/95-windfarm.md`](../95-windfarm.md)

---

## Experience-Level Personas

### Student/Academic

**Modified UI/UX:**
- Simplified map-based interface with drag-and-drop turbine placement on flat terrain
- Real-time wake visualization showing deficit cones as turbines are repositioned
- Integrated theory panel explaining wake models (Jensen, Frandsen) and wind statistics (Weibull)

**Feature Gating:**
- Single wake model (Jensen/Park) with fixed parameters; advanced models locked
- Flat terrain only; no complex terrain orography or forest canopy modeling
- AEP estimation with basic availability and electrical loss assumptions

**Pricing Tier:** Free tier

**Onboarding Flow:**
- Tutorial: place 5 turbines, observe wake effects, optimize spacing for maximum AEP
- Pre-loaded wind resource datasets for canonical sites (flat coastal, inland plains)
- Educational mode showing wake model equations alongside visualizations

**Key Workflows:**
- Study wind turbine wake interactions and their effect on farm energy production
- Optimize simple array layouts for minimum wake losses
- Analyze Weibull wind speed distributions and wind rose directional characteristics
- Compare different turbine models by hub height, rotor diameter, and power curve

---

### Junior Engineer (0-3 years)

**Modified UI/UX:**
- Project wizard: import wind data (mast or lidar), define site boundary, select turbine candidates
- Constraint mapping overlay: setbacks, environmental exclusion zones, noise contours, shadow flicker
- Layout optimization assistant with predefined objective functions (max AEP, min wake loss, min COE)

**Feature Gating:**
- Multiple wake models (Jensen, Bastankhah-Porte-Agel, Frandsen) with configurable parameters
- Basic terrain effects using speed-up factors from linear flow models
- Noise and shadow flicker assessment tools for permitting support

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Wind data import wizard: met mast time series, wind atlas extraction, or mesoscale data integration
- Site constraint setup: property boundaries, setback distances, environmental buffers
- 20-minute guided layout optimization from wind data through bankable AEP estimate

**Key Workflows:**
- Perform wind resource assessment from met mast data with long-term correlation (MCP)
- Optimize turbine layout within site constraints for maximum energy yield
- Conduct noise propagation analysis per ISO 9613 for regulatory compliance
- Calculate shadow flicker hours at receptor locations for permitting
- Generate P50/P90 energy estimates with uncertainty analysis for financing

---

### Senior/Expert

**Modified UI/UX:**
- Full analysis suite with CFD-based flow modeling, advanced wake models, and multi-objective optimization
- Probabilistic framework interface for uncertainty quantification across all analysis stages
- Multi-scenario workspace for comparing layout alternatives under varying wind regime assumptions

**Feature Gating:**
- All modules unlocked: CFD terrain flow, dynamic wake meandering, turbulence-induced loading
- Stochastic optimization with Pareto front visualization for multi-objective trade-offs
- API access for integration with turbine load simulation (aeroelastic tools) and financial models

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Migration assistant for WAsP, WindPRO, and OpenWind project import
- CFD solver configuration and terrain mesh quality assessment tools
- Monte Carlo uncertainty framework setup with firm-specific distribution assumptions

**Key Workflows:**
- Run CFD-based wind resource assessment over complex terrain and forested sites
- Perform multi-objective layout optimization balancing AEP, loads, noise, and cost of energy
- Conduct site suitability studies with turbine load assessment under site-specific conditions
- Quantify energy yield uncertainty with full Monte Carlo propagation through analysis chain
- Develop portfolio-level wind resource assessments across multiple prospective sites

---

## Role-Based Personas

### Designer/Creator

**Modified UI/UX:**
- Map-based layout editor with constraint-aware turbine placement and automatic exclusion zone enforcement
- Infrastructure design tools: access roads, cable routing, substation siting, and crane pad placement
- Visual comparison of layout alternatives with overlay toggle and performance delta indicators

**Feature Gating:**
- Full turbine layout and infrastructure design tools
- Electrical collection system design: cable sizing, loss calculation, and substation configuration
- Civil infrastructure planning: roads, foundations, crane pads, and laydown areas

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Site boundary import from GIS shapefiles or KML files with constraint layer configuration
- Turbine database browser with filtering by rated capacity, hub height, and IEC class
- Infrastructure design template gallery for common collector system topologies

**Key Workflows:**
- Create optimized turbine layouts satisfying all spatial and regulatory constraints
- Design electrical collection systems with cable routing and loss minimization
- Plan access road networks minimizing earthwork and environmental disturbance
- Evaluate foundation types (gravity, monopile, rock anchor) based on geotechnical conditions
- Generate site plans and construction drawings for permitting and procurement

---

### Analyst/Simulator

**Modified UI/UX:**
- Analysis workspace with wind data processing pipeline, flow model configuration, and results explorer
- Time-series analysis tools: data cleaning, gap filling, sector management, and quality flagging
- Uncertainty decomposition dashboard showing contribution of each source to overall AEP uncertainty

**Feature Gating:**
- Full wind resource analysis toolkit: MCP, frequency distribution fitting, vertical extrapolation
- All flow models: linear (WAsP-style), CFD RANS, and mesoscale-microscale coupling
- Turbulence and extreme wind analysis for IEC site assessment

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Wind data import and quality control tutorial with automated flagging and correction
- Flow model selection guide based on terrain complexity and required accuracy
- Uncertainty framework configuration with industry-standard assumptions

**Key Workflows:**
- Process raw wind data: quality control, filtering, recovery rate assessment
- Perform long-term wind resource correlation using MCP methods (matrix, variance ratio, regression)
- Run wind flow modeling over complex terrain with appropriate model selection
- Conduct comprehensive uncertainty analysis per industry best practices
- Generate independent energy yield assessments for due diligence and financing

---

### Manufacturing/Process

**Modified UI/UX:**
- Construction logistics planner with turbine delivery route analysis and crane movement scheduling
- Component tracking dashboard: towers, nacelles, blades, and transformers from factory to foundation
- Installation sequence optimizer considering weather windows and crane availability

**Feature Gating:**
- Construction scheduling tools with weather window analysis for crane operations
- Heavy lift planning with crane selection, ground bearing requirements, and clearance checks
- Material logistics tracking from port/factory through site installation

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Site access and logistics constraint input: road width limits, bridge capacities, turning radii
- Crane fleet configuration: models, capacities, mobilization schedules
- Weather window analysis setup for installation feasibility by season

**Key Workflows:**
- Plan turbine component delivery logistics from factory/port to turbine positions
- Optimize crane mobilization and installation sequence to minimize project duration
- Analyze weather windows for safe crane operations and blade lifting
- Track construction progress: foundations poured, towers erected, turbines commissioned
- Coordinate cable installation, testing, and energization with civil and turbine works

---

### Regulatory/Compliance

**Modified UI/UX:**
- Environmental compliance dashboard: protected species, habitat buffers, migration corridors
- Noise and shadow flicker compliance maps with receptor-level exceedance highlighting
- Permitting document tracker with submission deadlines and agency response status

**Feature Gating:**
- Noise propagation modeling per ISO 9613 with terrain, atmospheric, and uncertainty adjustments
- Shadow flicker computation with astronomical and meteorological data integration
- Environmental impact screening: avian risk, bat activity, visual impact assessments

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Regulatory jurisdiction selector: applicable noise limits, setback rules, and environmental requirements
- Receptor database import for noise and shadow flicker sensitive locations
- Environmental baseline data integration: avian surveys, habitat maps, cultural resource inventories

**Key Workflows:**
- Model noise propagation and demonstrate compliance with regulatory limits at all receptors
- Compute shadow flicker exposure at residential receptors and design mitigation (curtailment) schedules
- Prepare environmental impact assessments for permitting applications
- Support aviation assessments: turbine lighting requirements, radar interference analysis
- Manage community engagement with visual impact simulations and noise/flicker documentation

---

### Manager/Decision-maker

**Modified UI/UX:**
- Portfolio dashboard with development pipeline, construction status, and operational performance
- Investment analysis tools: LCOE, IRR, NPV, and sensitivity to wind resource, cost, and policy scenarios
- Risk matrix combining resource, permitting, interconnection, and financial uncertainties

**Feature Gating:**
- Read-only access to all technical analyses and compliance documentation
- Financial modeling with PPA pricing, merchant scenarios, and incentive analysis (PTC, REC)
- Site ranking and portfolio optimization tools for development prioritization

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Portfolio import with project pipeline data, resource estimates, and financial pro formas
- Financial model configuration: capital costs, O&M assumptions, financing structure, incentives
- Dashboard setup: portfolio capacity, weighted capacity factor, development stage summary

**Key Workflows:**
- Evaluate and rank prospective wind farm sites by resource quality, cost, and development risk
- Make turbine selection decisions considering AEP, cost, availability, and supply chain factors
- Review layout alternatives and approve final configuration based on energy-cost optimization
- Monitor portfolio performance against yield forecasts and financial projections
- Generate investor presentations and board reports with portfolio KPIs and risk assessments
