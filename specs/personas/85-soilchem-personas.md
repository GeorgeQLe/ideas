# SoilChem — Persona Subspecs

> Parent spec: [`specs/85-soilchem.md`](../85-soilchem.md)

---

## Experience-Level Personas

### Student/Academic
**Modified UI/UX:**
- Guided soil-profile builder with preset soil types (sand, loam, clay) and standard horizon layering
- Interactive water-balance diagram showing infiltration, evapotranspiration, root uptake, and drainage in real time
- Annotated nutrient cycling visualization linking N/P/K pools, transformation rates, and plant uptake

**Feature Gating:**
- 1D soil column simulations with Richards equation for water flow and basic solute transport
- Crop models limited to 3 common crops (maize, wheat, soybean) with standard cultivar parameters
- Simulation length limited to 2 growing seasons; multi-year climate scenarios locked

**Pricing Tier:** Free tier

**Onboarding Flow:**
- Tutorial: set up a soil column, plant a crop, simulate one growing season, and analyze yield and water balance
- Pre-loaded examples (irrigated vs. rainfed maize, nitrogen leaching under different fertilization rates)
- References to Hillel, Brady & Weil textbook chapters linked from each model component

**Key Workflows:**
- Simulate soil water dynamics under rainfall and irrigation for a single growing season
- Model nitrogen fate (mineralization, nitrification, denitrification, leaching) under fertilizer application
- Compare crop yield under different soil types and water management strategies
- Analyze sensitivity of drainage and leaching to soil hydraulic properties

---

### Junior Engineer (0-3 years)
**Modified UI/UX:**
- Project workspace with tabs: Soil Profile | Weather/Climate | Crop Management | Simulation | Analysis
- Integrated weather data connector pulling from public sources (NASA POWER, PRISM, local met stations)
- Soil database browser (SSURGO/ISRIC SoilGrids) with auto-population of hydraulic and chemical parameters

**Feature Gating:**
- Full 1D and 2D simulations with coupled water-heat-solute transport
- Crop model library expanded to 20+ crops with calibration tools
- Multi-year simulations up to 30 years; climate scenario generators gated

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Import site coordinates, auto-fetch soil data (SSURGO) and weather data (NASA POWER)
- Set up a 5-year crop rotation simulation and calibrate against observed yield data
- Generate a nutrient management report with recommended fertilizer timing and rates

**Key Workflows:**
- Design and evaluate crop rotation and fertilizer management plans for specific fields
- Calibrate soil hydraulic parameters using field-measured water content or drainage data
- Simulate pesticide fate and transport to assess groundwater contamination risk
- Generate nutrient budgets and identify optimal fertilizer application strategies
- Evaluate irrigation scheduling strategies to minimize water use while maintaining yield

---

### Senior/Expert
**Modified UI/UX:**
- Multi-field landscape modeling interface with spatial variability and lateral flow connections
- Python scripting console with full model API for custom process modules and automated calibration
- Ensemble simulation manager for uncertainty quantification across soil, weather, and management scenarios

**Feature Gating:**
- All features unlocked: 3D variably-saturated flow, reactive transport, plant-soil-atmosphere coupling
- Unlimited simulation domains, ensemble sizes, and climate scenario generators (GCM downscaling)
- Plugin SDK for custom crop models, biogeochemical modules, and data assimilation routines

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Migration wizard for importing HYDRUS/DSSAT/APSIM project files and calibration datasets
- HPC configuration and API setup for regional-scale automated modeling pipelines
- Workshop on custom module development and data assimilation integration

**Key Workflows:**
- Regional-scale modeling of nutrient loading to watersheds under land-use change scenarios
- Coupled reactive transport modeling of contaminant fate in the vadose zone
- Ensemble climate impact assessment on crop productivity and soil carbon sequestration
- Data assimilation integrating remote sensing (soil moisture, LAI) into model state updates
- Development of custom biogeochemical modules for novel soil amendments or cover crop interactions

---

## Role-Based Personas

### Designer/Creator
**Modified UI/UX:**
- Visual field-layout editor with soil zone mapping, irrigation system placement, and crop block definitions
- Real-time yield estimate gauge updating as management decisions (planting date, fertilizer rate, irrigation) are adjusted
- Side-by-side comparison view for evaluating alternative management strategies

**Feature Gating:**
- Crop management design tools fully enabled (rotation planning, fertilizer scheduling, irrigation design)
- Simulation available in rapid-estimate mode for interactive design; full-fidelity on demand
- Export to farm management plan formats (PDF reports, GIS shapefiles for precision agriculture)

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Import field boundaries (shapefile/KML), auto-fetch soil and weather data
- Design a crop rotation and fertilizer management plan using guided wizard
- Preview expected yield and environmental outcomes before committing to full simulation

**Key Workflows:**
- Design precision agriculture management zones based on soil variability mapping
- Plan crop rotations optimizing for yield, soil health, and nutrient cycling
- Design irrigation systems and scheduling strategies for water-use efficiency
- Evaluate cover crop and tillage alternatives on soil health indicators

---

### Analyst/Simulator
**Modified UI/UX:**
- Data-centric layout with time-series plots, soil profile viewers, and statistical summary panels
- Calibration workspace with parameter sensitivity analysis and GLUE/MCMC uncertainty tools
- Batch simulation manager for scenario analysis and ensemble runs

**Feature Gating:**
- Full solver access with all process modules and calibration/uncertainty tools
- Inverse modeling and parameter estimation tools enabled
- API access for automated model pipelines and data assimilation workflows

**Pricing Tier:** Professional or Enterprise tier

**Onboarding Flow:**
- Import field experiment data, set up model, and perform parameter calibration against observations
- Run a sensitivity analysis to identify dominant parameters for the system
- Set up an ensemble simulation quantifying prediction uncertainty

**Key Workflows:**
- Calibrate and validate soil-crop models against multi-year field experiment data
- Perform uncertainty analysis using Monte Carlo, GLUE, or Bayesian methods
- Run long-term scenario analyses of climate change impacts on crop productivity
- Analyze nutrient and pesticide leaching risk under various management and climate scenarios
- Develop regional-scale assessments of agricultural greenhouse gas emissions

---

### Manufacturing/Process
**Modified UI/UX:**
- Application-focused interface showing fertilizer/amendment product specifications and application parameters
- Field-level application maps generated from model-recommended rates
- Integration panels for precision agriculture equipment (variable-rate controllers, GPS systems)

**Feature Gating:**
- Model-generated application rate recommendations enabled
- Full modeling tools hidden; only recommendation outputs and application maps visible
- Export to precision agriculture formats (shapefiles for variable-rate applicators, ISO-XML)

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Import field data and model-generated nutrient recommendations
- Generate variable-rate application maps for fertilizer and lime
- Connect to precision agriculture equipment for direct upload of application prescriptions

**Key Workflows:**
- Generate variable-rate fertilizer application maps from model-driven recommendations
- Produce lime application prescriptions based on soil pH mapping and modeling
- Create irrigation scheduling prescriptions from soil water balance simulations
- Track actual vs. recommended application rates for adaptive management

---

### Regulatory/Compliance
**Modified UI/UX:**
- Nutrient management compliance dashboard with regulatory limit tracking (nitrate directive, TMDL, NMP requirements)
- Traffic-light indicators for nutrient balance, leaching risk, and buffer compliance at field and farm scale
- Automated report generation in formats required by regulatory agencies (USDA-NRCS, EU CAP, state agencies)

**Feature Gating:**
- Nutrient balance and leaching risk assessment tools fully enabled
- Regulatory reporting templates for applicable jurisdictions enabled
- Audit trail and model provenance tracking for regulatory defensibility

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Select applicable regulatory framework (USDA nutrient management, EU Nitrates Directive, state-specific)
- Import farm/field data and run nutrient balance assessment
- Generate compliance report and identify fields requiring management changes

**Key Workflows:**
- Produce nutrient management plans meeting USDA-NRCS 590 standard requirements
- Assess nitrate leaching risk against EU Nitrates Directive thresholds
- Generate TMDL compliance documentation for watershed nutrient loading
- Track soil health indicators for conservation program eligibility (EQIP, CSP)
- Maintain audit-ready records of management practices and model-based predictions

---

### Manager/Decision-maker
**Modified UI/UX:**
- Farm/enterprise dashboard with field-level yield projections, cost summaries, and sustainability scores
- Scenario comparison cards for management alternatives with economic and environmental trade-offs
- ROI calculator for precision agriculture investments and conservation practice adoption

**Feature Gating:**
- Read-only access to all model results and compliance reports
- Economic analysis and decision-support tools enabled
- Approval workflow for management plans and investment decisions

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Dashboard overview: yield projections, input costs, sustainability metrics by field
- Review scenario comparison tools for evaluating management alternatives
- Set up notification preferences for yield risk alerts and compliance deadlines

**Key Workflows:**
- Compare management alternatives on yield, profitability, and environmental impact
- Evaluate ROI of precision agriculture technology investments
- Review and approve nutrient management plans for regulatory compliance
- Monitor sustainability metrics for carbon credit and conservation program participation
- Generate enterprise-level reports for stakeholders and financing partners
