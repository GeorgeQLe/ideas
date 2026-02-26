# CastSim — Persona Subspecs

> Parent spec: [`specs/76-castsim.md`](../76-castsim.md)

---

## Experience-Level Personas

### Student/Academic
**Modified UI/UX:**
- Interactive solidification visualizer with annotated temperature gradients, nucleation sites, and grain growth
- Simplified casting process selector with educational descriptions (sand, investment, die, continuous)
- Color-coded defect predictor showing shrinkage porosity, hot tears, and cold shuts on a sample casting

**Feature Gating:**
- Basic thermal solidification simulation for simple geometries unlocked
- Filling simulation limited to gravity pour; high-pressure die casting and squeeze casting locked
- Material database restricted to common alloys (A356, gray iron, C360 brass); custom alloys locked

**Pricing Tier:** Free tier (academic license)

**Onboarding Flow:**
- Tutorial: simulate solidification of a simple plate casting and identify last-to-freeze regions
- Pre-loaded examples: step casting with risers, hub casting, and benchmark test geometries
- Guided comparison of simulation results against Chvorinov's rule for validation

**Key Workflows:**
- Simulating solidification sequence for simple casting geometries in coursework
- Predicting shrinkage porosity locations using thermal criteria (Niyama, temperature gradient)
- Comparing riser sizing calculations against simulation-based feeding analysis
- Generating solidification time contour plots for foundry engineering assignments

### Junior Engineer (0-3 years)
**Modified UI/UX:**
- Step-by-step casting simulation workflow: import geometry, define gating/risering, set process parameters, simulate
- Automated gating and risering suggestion engine based on casting geometry and alloy selection
- Defect risk heatmap overlay on casting geometry with severity and location indicators

**Feature Gating:**
- Full filling and solidification simulation for all gravity casting processes
- Porosity prediction with Niyama criterion and feeding distance analysis
- Die casting and low-pressure casting modules available; microstructure prediction restricted

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Setup wizard asking casting process type, primary alloys, and typical part size range
- Walkthrough: import a casting CAD model, auto-suggest risering, simulate, and evaluate results
- Introduction to iterative design improvement workflow and result comparison tools

**Key Workflows:**
- Setting up filling and solidification simulations for production casting designs
- Evaluating gating system designs for mold filling quality (turbulence, air entrapment)
- Optimizing riser size and placement to eliminate shrinkage porosity
- Predicting hot spot locations and recommending chills or geometry modifications
- Generating simulation reports for foundry production planning

### Senior/Expert
**Modified UI/UX:**
- Multi-physics console coupling filling, solidification, stress, microstructure, and property prediction
- Custom alloy thermodynamic model editor with Scheil solidification and phase fraction calculations
- Scripting engine for automated DOE campaigns across gating/risering configurations

**Feature Gating:**
- All modules: stress/distortion, microstructure, mechanical property prediction, die thermal cycling
- Full API for integration with PLM systems, process simulation chains, and real-time foundry data
- Admin controls for proprietary alloy databases, process recipes, and simulation standards

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Migration assistant for importing MAGMASOFT/ProCAST projects and historical simulation databases
- Configuration of custom alloy thermophysical property databases and solidification models
- Setup of automated simulation validation pipelines against casting trial data

**Key Workflows:**
- Multi-physics simulation of the entire casting process from filling through heat treatment
- Predicting as-cast microstructure and correlating to mechanical properties
- Optimizing die thermal management for high-pressure die casting cycle time and quality
- Calibrating simulation models against foundry trial data for production process optimization
- Developing virtual casting trials to reduce physical prototyping iterations

---

## Role-Based Personas

### Designer/Creator
**Modified UI/UX:**
- Casting-aware part design advisor flagging design-for-casting violations (thin walls, sharp corners, undercuts)
- Parametric gating/risering system designer with drag-and-drop runners, gates, and risers
- Mold/die layout tool with parting line definition, core placement, and draft angle verification

**Feature Gating:**
- Full access to gating/risering design tools and geometry modification features
- Design-for-casting rule checker with customizable rule sets by process type
- Export to pattern/die CAD formats with shrinkage allowance and draft applied

**Pricing Tier:** Professional or Enterprise tier

**Onboarding Flow:**
- Quick-start: import a part geometry, auto-generate gating/risering, and evaluate with fast simulation
- Tutorial on design-for-casting principles integrated with the rule checker
- Walkthrough of iterative design improvement with simulation feedback loop

**Key Workflows:**
- Designing gating systems (sprues, runners, ingates) for optimal mold filling
- Sizing and placing risers and chills for directional solidification
- Modifying part geometry to improve castability (fillet radii, wall transitions, draft)
- Designing core prints and core assembly configurations for internal features
- Creating pattern/tooling designs with process-specific shrinkage allowances

### Analyst/Simulator
**Modified UI/UX:**
- Simulation workflow manager with meshing controls, solver settings, and convergence/completion monitoring
- Result analysis suite with solidification curves, temperature history at probes, and defect criterion maps
- Automated validation dashboard comparing simulation predictions to X-ray, CT scan, and sectioning data

**Feature Gating:**
- Full solver access for filling (VOF), solidification, stress, and microstructure prediction
- Advanced post-processing including virtual X-ray, CT comparison, and porosity quantification
- HPC integration for large die casting assemblies and multi-cavity simulations

**Pricing Tier:** Professional or Enterprise tier

**Onboarding Flow:**
- Guided first simulation: solidification of a benchmark casting with comparison to known defect locations
- Setup of standard simulation protocols and result evaluation criteria
- Introduction to parametric study setup for process window optimization

**Key Workflows:**
- Running filling simulations to evaluate gate velocity, air entrapment, and oxide film risk
- Performing solidification analysis with thermal criteria for porosity and hot tear prediction
- Simulating residual stress and distortion for dimensional accuracy prediction
- Predicting microstructure (grain size, DAS, phase fractions) and mechanical properties
- Validating simulation against casting trials using X-ray, CT, and sectioning data

### Manufacturing/Process
**Modified UI/UX:**
- Process parameter optimization dashboard with cycle time, yield, and quality correlations
- Foundry floor integration panel showing real-time pouring temperature, timing, and die temperatures
- Scrap analysis tool correlating defect types with process parameter deviations

**Feature Gating:**
- Access to process optimization and virtual trial tools
- Real-time data integration from foundry sensors (thermocouples, metal level, pressure)
- Statistical process control (SPC) charting linked to simulation-predicted process windows

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Configure foundry equipment inventory (furnaces, pouring lines, die casting machines, heat treatment ovens)
- Walkthrough: optimize pouring temperature and speed for an existing production casting
- Tutorial on using simulation to troubleshoot a current casting defect issue

**Key Workflows:**
- Optimizing process parameters (pouring temperature, speed, die preheat) for quality and cycle time
- Troubleshooting casting defects by correlating simulation predictions with shop floor observations
- Designing die thermal management strategies (cooling channels, spray patterns, cycle timing)
- Planning foundry capacity by simulating cycle time optimization for die casting
- Establishing process windows and control limits using simulation-based sensitivity studies

### Regulatory/Compliance
**Modified UI/UX:**
- Material certification tracker linking melt chemistry to predicted properties and specification compliance
- Casting quality standard compliance dashboard (ASTM E446, MSS-SP-55, customer-specific radiographic standards)
- Traceability system linking simulation predictions to actual inspection results per casting serial number

**Feature Gating:**
- Read-only access to simulations; full compliance checking and documentation tools
- Automated comparison of predicted quality indicators against acceptance criteria
- Certification document generator with traceable links to material and process records

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Configuration of applicable standards (ASTM, AMS, customer specifications) and acceptance criteria
- Setup of casting quality documentation templates and approval workflows
- Walkthrough: generate a material certification package for a sample casting lot

**Key Workflows:**
- Verifying that predicted porosity levels meet radiographic acceptance standards
- Generating material test reports with full traceability from melt to finished casting
- Auditing simulation-based process qualification records against customer specifications
- Tracking non-conformance reports and correlating with simulation predictions for root cause
- Maintaining casting quality records for aerospace, automotive, and pressure-retaining applications

### Manager/Decision-maker
**Modified UI/UX:**
- Foundry operations dashboard with yield rates, scrap costs, and simulation ROI metrics
- New product introduction (NPI) tracker showing simulation-driven design iterations vs. physical trials
- Capacity planning tool estimating production throughput based on simulated cycle times

**Feature Gating:**
- Dashboard and reporting access; no direct simulation or design tools
- Yield improvement tracking and cost-of-quality analysis
- Resource utilization and project pipeline management views

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Configuration of foundry KPIs (yield, scrap rate, OEE, energy consumption) and targets
- Integration with ERP system for cost tracking and production scheduling
- Setup of automated management reporting and KPI dashboard distribution

**Key Workflows:**
- Monitoring casting yield improvements driven by simulation-optimized process changes
- Evaluating ROI of simulation investment vs. physical trial cost savings
- Reviewing new part feasibility assessments for quoting and capacity planning
- Tracking product development timelines from initial simulation to production qualification
- Making strategic investment decisions (new equipment, process capability) informed by simulation data
