# PropulSim — Persona Subspecs

> Parent spec: [`specs/74-propulsim.md`](../74-propulsim.md)

---

## Experience-Level Personas

### Student/Academic
**Modified UI/UX:**
- Interactive rocket equation explorer with visual delta-v budget builder and propellant trade charts
- Annotated engine cycle diagrams (expander, gas generator, staged combustion) with flow path highlighting
- Simplified CEA-style equilibrium calculator with educational explanations of chemistry and thermodynamics

**Feature Gating:**
- Chemical equilibrium and ideal performance calculations (Isp, c*, Cf) fully available
- Engine cycle analysis limited to standard templates; custom cycle architecture definition locked
- Nozzle contouring restricted to conical and 80% bell; MOC and CFD nozzle design locked

**Pricing Tier:** Free tier (academic license with .edu verification)

**Onboarding Flow:**
- Tutorial: calculate ideal performance of an LOX/LH2 engine from chamber conditions to nozzle exit
- Pre-loaded textbook propellant combinations with annotated Isp vs. mixture ratio curves
- Prompt to explore example missions (LEO, GTO, lunar) with delta-v requirement breakdowns

**Key Workflows:**
- Computing chemical equilibrium performance for various propellant combinations
- Comparing Isp, density-Isp, and c* across propellant candidates for mission trade studies
- Building delta-v budgets and sizing propellant mass fractions for conceptual missions
- Generating annotated T-s and h-s diagrams for propulsion system thermodynamic cycles

### Junior Engineer (0-3 years)
**Modified UI/UX:**
- Guided engine sizing workflow from thrust requirements through chamber/nozzle geometry to turbopump specs
- Context-sensitive design checks flagging values outside historical norms (chamber pressure, O/F ratio, Pc/Pe)
- Side-by-side comparison panels for engine configurations with performance delta highlighting

**Feature Gating:**
- Full engine cycle analysis for standard architectures (gas generator, expander, staged combustion)
- Nozzle design including method of characteristics with guided setup
- Injector pattern selection from validated templates; custom injector design requires senior approval

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Role-based setup: liquid vs. solid vs. hybrid vs. electric propulsion focus area
- Walkthrough: size a pressure-fed engine from mission requirements to preliminary design layout
- Introduction to project documentation standards and design review workflows

**Key Workflows:**
- Sizing engine components (chamber, nozzle, turbopumps) from mission thrust requirements
- Running engine cycle balance calculations and generating system-level performance summaries
- Designing nozzle contours using truncated ideal and Rao optimum methods
- Performing propellant trade studies with tankage mass and volume constraints
- Preparing preliminary design review (PDR) documentation packages

### Senior/Expert
**Modified UI/UX:**
- Full system-level engine model builder with component-level fidelity (injector, chamber, nozzle, TPA, valves)
- Transient simulation console for start/shutdown sequences, throttling transients, and failure modes
- Python/MATLAB scripting interface for custom analysis methods and optimization frameworks

**Feature Gating:**
- All modules: combustion instability analysis, ablation/thermal modeling, transient simulation, mission integration
- Full API for coupling with trajectory codes, structural FEA, and CFD solvers
- Admin controls for proprietary propellant data, engine models, and test databases

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Migration assistant for importing NASA CEA/NPSS models and legacy Fortran engine codes
- Configuration of custom propellant thermochemistry databases and transport property models
- Setup of automated engine model validation pipelines against test data

**Key Workflows:**
- Building high-fidelity engine system models with component-level loss accounting
- Simulating engine start/shutdown transients including valve sequencing and ignition
- Performing combustion stability analysis (Hewitt stability rating, Crocco linear theory)
- Optimizing engine design for specific mission profiles with trajectory-coupled analysis
- Developing and validating custom combustion and heat transfer models against hotfire test data

---

## Role-Based Personas

### Designer/Creator
**Modified UI/UX:**
- Engine architecture sketcher with drag-and-drop components (pumps, preburners, valves, heat exchangers)
- Parametric nozzle and chamber geometry editor with real-time performance impact display
- Injector pattern designer with element placement, spray angle, and mixing efficiency visualization

**Feature Gating:**
- Full access to engine architecture definition and component sizing tools
- Injector design module with empirical mixing and combustion efficiency correlations
- Export to CAD formats for detailed mechanical design and manufacturing

**Pricing Tier:** Professional or Enterprise tier

**Onboarding Flow:**
- Quick-start: define an engine cycle by connecting components from the library
- Import an existing engine schematic and parameterize for trade studies
- Tutorial on linking design parameters to system-level performance metrics

**Key Workflows:**
- Defining engine cycle architectures and sizing all major components
- Designing injector element patterns for target mixing efficiency and stability margins
- Optimizing nozzle contours for maximum Isp within length/mass constraints
- Creating regenerative cooling channel designs with thermal-structural assessment
- Building parametric engine families for vehicle-level trade studies

### Analyst/Simulator
**Modified UI/UX:**
- Simulation control center with multi-fidelity analysis chain management (0D to 3D)
- Combustion analysis dashboard with flame structure, temperature fields, and species distributions
- Automated comparison tools for simulation vs. test data with uncertainty quantification

**Feature Gating:**
- Full access to all analysis fidelity levels from equilibrium through CFD
- Combustion instability analysis tools (acoustic modes, Rayleigh index, transfer functions)
- HPC integration for 3D reacting-flow CFD and conjugate heat transfer simulations

**Pricing Tier:** Professional or Enterprise tier

**Onboarding Flow:**
- Guided first analysis: equilibrium performance calculation with comparison to NASA CEA benchmark
- Setup of analysis workflow templates for standard propulsion analyses
- Introduction to multi-fidelity analysis chain: 0D equilibrium to 1D nozzle to 2D/3D CFD

**Key Workflows:**
- Running chemical equilibrium calculations with real-gas and condensed-phase effects
- Performing 1D nozzle flow analysis with boundary layer and divergence loss corrections
- Simulating combustion chamber reacting flows with detailed kinetics
- Analyzing combustion instability modes and predicting stability boundaries
- Correlating analysis predictions with hotfire test data for model calibration

### Manufacturing/Process
**Modified UI/UX:**
- Manufacturing process selector mapping engine components to fabrication methods (AM, casting, machining, spinning)
- Material compatibility checker for propellant-material interactions at operating conditions
- Weld and braze joint analyzer for pressure boundary integrity at cryogenic temperatures

**Feature Gating:**
- Access to manufacturing feasibility assessment tools and process selection guidance
- Material property database filtered by propellant compatibility and process capability
- Additive manufacturing design rule checkers for regeneratively cooled chambers

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Configure facility capabilities (AM machines, machining centers, test stands)
- Walkthrough: assess manufacturability of a regeneratively cooled chamber design
- Tutorial on material selection considering propellant compatibility and manufacturing process

**Key Workflows:**
- Evaluating manufacturing approaches for combustion chambers, nozzles, and turbopump components
- Checking propellant-material compatibility for long-duration missions and storability
- Assessing additive manufacturing feasibility for complex cooling channel geometries
- Defining inspection and acceptance criteria for flight-critical propulsion hardware
- Planning build sequences accounting for post-machining, welding, and proof testing

### Regulatory/Compliance
**Modified UI/UX:**
- Range safety compliance dashboard with failure mode blast radii and toxic corridor predictions
- Environmental compliance tracker for propellant handling, exhaust emissions, and launch site permits
- Automated safety documentation generator for ITAR/EAR export control classification

**Feature Gating:**
- Read-only design access; full safety analysis and regulatory documentation tools
- Automated range safety calculations (blast overpressure, debris, toxic dispersion)
- ITAR/EAR classification wizard and technology control plan management

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Configuration of applicable regulations (FAA AST, NASA-STD-5017, Range Safety standards, ITAR)
- Setup of safety documentation templates and classification review workflows
- Walkthrough: generate a flight termination system requirements document for a sample vehicle

**Key Workflows:**
- Generating range safety analysis reports (blast overpressure, debris, toxic hazard corridors)
- Documenting propellant system hazard analyses and safety-critical item identification
- Managing ITAR/EAR classification determinations and technology control plans
- Producing environmental impact documentation for launch site operations
- Tracking compliance with NASA/DoD safety standards for propulsion systems

### Manager/Decision-maker
**Modified UI/UX:**
- Engine development program dashboard with milestone tracking, budget burn rate, and test campaign status
- Technology readiness assessment tool comparing propulsion options against mission requirements
- Cost and schedule risk Monte Carlo visualization for development program planning

**Feature Gating:**
- Dashboard and reporting access; no direct design or analysis tools
- Trade study and technology comparison views with cost/schedule/performance weighting
- Program-level resource allocation and subcontractor management views

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Configuration of engine development program phases (SRR, PDR, CDR, TRR, QR)
- Integration with program management tools and earned value management systems
- Setup of automated program status reporting and risk review cadence

**Key Workflows:**
- Monitoring engine development progress against cost, schedule, and technical performance targets
- Evaluating propulsion technology trade-offs for vehicle-level mission optimization
- Tracking test campaign results and correlating with development risk reduction
- Reviewing make/buy decisions for engine components and subsystems
- Approving test readiness reviews and design certification based on evidence packages
