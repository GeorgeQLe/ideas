# HydroWorks — Persona Subspecs

> Parent spec: [`specs/58-hydroworks.md`](../58-hydroworks.md)

---

## Experience-Level Personas

### Student/Academic

**Modified UI/UX:**
- Simplified channel/network editor with predefined cross-section shapes (rectangular, trapezoidal, circular) and drag-to-connect nodes
- Inline hydraulics primer explaining Manning's equation, energy equation, and gradually varied flow classifications (M1, S2, etc.)
- Results panel defaults to water surface profile, velocity, and Froude number with colour-coded subcritical/supercritical indicators

**Feature Gating:**
- Network limited to 50 reaches and 20 junctions; steady-state 1D solver only
- Pre-built rainfall and runoff data for sample watersheds; no GIS data import or real-time gauge connections
- Export to PDF and CSV; no HEC-RAS/MIKE interop or regulatory floodplain mapping tools

**Pricing Tier:** Free tier (educational license)

**Onboarding Flow:**
- "First Channel" tutorial: define a trapezoidal open channel, compute normal and critical depth, plot the water surface profile
- Interactive gradually-varied-flow explorer: select a profile type (M1, M2, S1, etc.) and watch it develop as parameters change
- Sandbox with textbook problems (backwater curves, hydraulic jumps, culvert flow, weir discharge)

**Key Workflows:**
- Compute a backwater curve upstream of a dam and classify the water surface profile type
- Design a trapezoidal channel to convey a design discharge with freeboard and velocity constraints
- Analyse culvert hydraulics under inlet and outlet control conditions
- Generate lab report figures showing water surface profiles and energy/momentum diagrams

---

### Junior Engineer (0-3 years)

**Modified UI/UX:**
- Standard workspace with project wizard: select analysis type (river hydraulics, storm drainage, dam breach, bridge scour)
- Guided hydrologic model builder with SCS curve number, rational method, and unit hydrograph options
- Checklist sidebar tracking regulatory deliverables (FEMA FIS, stormwater report, floodplain delineation)

**Feature Gating:**
- 1D steady and unsteady solver; 1D/2D coupling for overbank flow; network up to 5,000 nodes
- Hydrologic modelling with SCS, Green-Ampt, and unit hydrograph methods; design storm libraries (NOAA Atlas 14)
- Import from GIS (shapefiles, DEM), HEC-RAS geometry files; export to floodplain mapping and regulatory formats

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Project-type selector (FEMA flood study, municipal stormwater, dam safety, bridge hydraulics) pre-loads code and data defaults
- Walkthrough of a stream flood study: import DEM, create cross sections, run unsteady flow, delineate floodplain
- Prompted to connect to local rainfall data sources and regulatory submission templates

**Key Workflows:**
- Develop a HEC-RAS-equivalent flood model for a FEMA Flood Insurance Study and produce floodplain maps
- Design a municipal storm drainage network with inlet sizing, pipe routing, and tailwater analysis
- Perform bridge hydraulic analysis evaluating backwater, scour, and roadway overtopping
- Run design-storm hydrologic analysis using NOAA Atlas 14 rainfall data with SCS curve number method
- Generate regulatory-compliant hydraulic calculation reports and floodplain maps for permit applications

---

### Senior/Expert

**Modified UI/UX:**
- Multi-dimensional workspace with fully coupled 1D/2D solver, sediment transport, and water quality modules
- Scripting console (Python API) for automated scenario generation, ensemble hydrologic modelling, and real-time flood forecasting
- Real-time dashboard linking gauge/radar/satellite rainfall data to model for operational flood prediction

**Feature Gating:**
- Full 2D shallow water solver with GPU acceleration; sediment transport (bedload, suspended, morphodynamic)
- Real-time data assimilation, ensemble forecasting, and probabilistic flood mapping
- Full API, HPC/cloud compute support, and integration with MIKE, InfoWorks, GIS platforms, and SCADA/telemetry

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Import existing HEC-RAS/MIKE/InfoWorks models with automated geometry and parameter translation
- Configure real-time data feeds (USGS, NWS, radar QPE) and ensemble forecast framework
- Power-user preset opening scripted real-time flood forecasting dashboard

**Key Workflows:**
- Build a 2D flood model of an urban area with building-resolved flow paths and depth-velocity hazard mapping
- Develop a real-time flood forecasting system with ensemble rainfall inputs and probabilistic inundation mapping
- Perform dam-breach analysis (sunny-day and flood-induced) with downstream consequence assessment
- Model long-term river morphodynamics including bank erosion, sediment bar migration, and reservoir sedimentation
- Automate climate-change flood risk assessment by running hydrologic models with downscaled GCM precipitation scenarios

---

## Role-Based Personas

### Designer/Creator

**Modified UI/UX:**
- Visual watershed and channel design canvas with terrain-draped plan view and longitudinal profile editor
- Stormwater BMP design library with sizing calculators for detention ponds, bioswales, permeable pavement, and green roofs
- Floodplain design tools for levee alignment, floodwall placement, and channel modification with real-time impact preview

**Feature Gating:**
- Full geometry editing for channels, structures, and stormwater facilities
- Automatic design sizing tools for detention basins, culverts, and conveyance channels to meet peak-flow targets
- Export to civil design packages (Civil 3D, OpenRoads) and GIS for site plan integration

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Site import from DEM/LiDAR with automatic watershed delineation and flow path identification
- "Stormwater Design" wizard: enter site area, imperviousness, and regulatory requirements; receive BMP sizing recommendations
- Tour of detention pond design tools using a sample commercial development site

**Key Workflows:**
- Design a regional detention basin sizing for pre- and post-development peak flow matching
- Layout a storm drainage network for a subdivision with inlet placement and pipe sizing
- Evaluate green infrastructure alternatives (bioretention, permeable pavement) for stormwater quality compliance
- Design channel improvements (realignment, grade control, bank protection) for a flood mitigation project
- Produce grading and drainage plans coordinated with civil site design

---

### Analyst/Simulator

**Modified UI/UX:**
- Analysis workspace with hydrograph viewers, stage-discharge rating curves, flood frequency plots, and sensitivity panels
- Multi-scenario comparison tools overlaying flood extents, depths, and velocities for different return periods and climate scenarios
- Calibration assistant with automated parameter adjustment using observed gauge data and goodness-of-fit metrics

**Feature Gating:**
- All solver modules enabled: 1D/2D unsteady, sediment, water quality, and ice-jam
- Monte Carlo and ensemble simulation tools for uncertainty quantification on roughness, rainfall, and initial conditions
- Real-time data assimilation and forecast verification tools

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Validation suite running standard benchmark problems (USACE dam-breach tests, EA 2D benchmarks)
- Import gauge records and configure automatic calibration framework
- Configure solver tolerances, mesh resolution strategies, and parallel computing preferences

**Key Workflows:**
- Calibrate and validate a watershed hydrologic-hydraulic model against historical flood events
- Perform flood frequency analysis using simulated long-duration continuous records
- Run sensitivity analysis on Manning's n, curve number, and channel geometry to quantify model uncertainty
- Evaluate climate-change impacts on flood magnitudes using statistically downscaled precipitation projections
- Produce technical reports with calibration metrics, uncertainty bounds, and regulatory-standard methodology

---

### Manufacturing/Process

**Modified UI/UX:**
- Construction-centric view showing earthwork quantities, pipe schedules, and structure details for drainage infrastructure
- Material take-off dashboard linking hydraulic design to concrete, pipe, riprap, and geotextile quantities
- Construction sequencing panel with dewatering and diversion requirements during in-stream work

**Feature Gating:**
- Earthwork computation tools for channels, detention basins, and levees
- Pipe and structure specification generators with material and joint details
- Integration with construction estimating software and project scheduling tools

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Import hydraulic design and auto-generate construction quantities and material specifications
- Guided walkthrough of a storm drainage construction package: pipe schedule, manhole details, trench sections
- Connect to firm's unit cost database for automated cost estimation

**Key Workflows:**
- Generate earthwork cut/fill quantities for a detention basin and channel improvement project
- Produce pipe schedules with sizes, materials, slopes, and invert elevations for storm drainage construction
- Develop construction sequencing plans for in-stream work including diversion and dewatering
- Calculate riprap sizing and placement quantities for channel bank protection per HEC-11
- Produce bid-ready construction cost estimates linked to hydraulic design quantities

---

### Regulatory/Compliance

**Modified UI/UX:**
- Regulatory dashboard organised by programme (FEMA NFIP, CWA Section 404, MS4 NPDES, state dam safety)
- FEMA floodplain mapping toolkit with automated BFE computation, floodway analysis, and FIS table generation
- Audit trail documenting every model parameter, calibration decision, and regulatory interpretation with references

**Feature Gating:**
- FEMA-compliant floodplain mapping and floodway encroachment analysis tools fully enabled
- Stormwater quality and quantity compliance calculators for NPDES MS4 permits
- Dam safety inundation mapping per federal and state dam safety regulations

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Jurisdiction selector pre-loads FEMA Region guidance, state stormwater regulations, and local ordinance requirements
- Walkthrough of a FEMA Flood Insurance Study workflow from model development to FIRM panel production
- Configure submission templates for LOMR, CLOMR, and stormwater permit applications

**Key Workflows:**
- Produce a FEMA Flood Insurance Study with hydraulic model, floodplain maps, FIS tables, and technical support data
- Prepare LOMR/CLOMR applications for developments affecting regulatory floodplains
- Demonstrate stormwater management compliance with MS4 NPDES post-construction requirements
- Perform dam safety inundation analysis and prepare Emergency Action Plan mapping per state regulations
- Generate no-rise / no-impact certifications for proposed floodplain developments

---

### Manager/Decision-maker

**Modified UI/UX:**
- Portfolio dashboard showing all flood risk and stormwater projects with budget, schedule, and regulatory status
- Risk visualisation mapping flood exposure across a portfolio of assets (buildings, infrastructure, land parcels)
- Cost-benefit analysis tools for flood mitigation alternatives (levees, buyouts, flood-proofing, green infrastructure)

**Feature Gating:**
- Read-only access to hydraulic models and flood maps; cannot modify geometry or parameters
- Full access to flood risk quantification, cost-benefit analysis, and project management modules
- Insurance and financing analytics linking flood risk to NFIP premiums and property valuations

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Asset portfolio import with geocoded locations and flood zone mapping
- Dashboard configuration selecting KPIs (annual expected loss, benefit-cost ratio, project delivery schedule)
- Demo of flood mitigation cost-benefit analysis for a sample community protection project

**Key Workflows:**
- Evaluate cost-benefit ratios for competing flood mitigation strategies (structural vs. non-structural)
- Assess flood risk exposure across a real estate or infrastructure portfolio for insurance and investment decisions
- Track capital improvement programme delivery for flood control and stormwater projects
- Communicate flood risk to elected officials and community stakeholders using interactive inundation maps
- Prioritise flood mitigation investments across multiple communities based on risk reduction per dollar
