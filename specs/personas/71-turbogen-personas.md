# TurboGen — Persona Subspecs

> Parent spec: [`specs/71-turbogen.md`](../71-turbogen.md)

---

## Experience-Level Personas

### Student/Academic
**Modified UI/UX:**
- Guided meanline design wizard with annotated velocity triangles and Mollier diagrams
- Inline educational tooltips explaining Euler's turbomachinery equation, loss models, and efficiency definitions
- Simplified blade profile editor with preset NACA/DCA families and auto-constrained parameters

**Feature Gating:**
- Full access to 1D meanline and throughflow analysis; 3D CFD coupling locked
- Export limited to academic report formats (PDF, CSV); no direct CAM or manufacturing output
- Community turbine/compressor template library available; proprietary OEM profiles hidden

**Pricing Tier:** Free tier (academic license with .edu verification)

**Onboarding Flow:**
- Interactive tutorial: design a single-stage axial compressor from inlet conditions to efficiency map
- Pre-loaded textbook example (de Haller criterion walkthrough) as starter project
- Prompt to join student community forum and link university course integration

**Key Workflows:**
- Meanline analysis of axial/radial turbomachinery stages for coursework
- Parametric study of blade angles vs. stage loading and flow coefficients
- Generating Smith charts and comparing designs against published correlations
- Exporting annotated velocity triangle diagrams for lab reports

### Junior Engineer (0-3 years)
**Modified UI/UX:**
- Workflow-oriented dashboard grouping tasks into Design, Analyze, Optimize sequences
- Context-sensitive warnings when designs violate standard practice (e.g., diffusion factor > 0.6, tip speed limits)
- Side-by-side comparison panels for baseline vs. modified designs with delta highlighting

**Feature Gating:**
- 1D/2D throughflow and streamline curvature methods fully available
- 3D blade-to-blade CFD enabled with guided mesh setup; advanced turbulence model selection restricted to validated presets
- Structural FEA coupling for blade stress available; creep/lifing modules require senior approval

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Role-based setup asking turbine vs. compressor focus, axial vs. radial vs. mixed-flow
- Walkthrough of a complete stage redesign from legacy data import to performance map generation
- Introduction to version control and design review submission workflow

**Key Workflows:**
- Importing existing blade geometry and reverse-engineering meanline parameters
- Running throughflow analysis and iterating blade angle distributions
- Generating compressor/turbine maps (pressure ratio vs. corrected flow) for system-level matching
- Preparing design review packages with automated performance summary sheets
- Comparing CFD results against 1D predictions to validate throughflow assumptions

### Senior/Expert
**Modified UI/UX:**
- Multi-stage stacking and inter-stage matching console with real-time thermodynamic cycle coupling
- Advanced loss model configuration (Ainley-Mathieson, Kacker-Okapuu, custom correlations) with full coefficient override
- Scripting console (Python/Lua) for custom optimization objectives and automated DOE campaigns

**Feature Gating:**
- All modules unlocked: 3D CFD, aeroelastic flutter, forced response, tip clearance effects
- Full API access for integration with in-house cycle codes and proprietary loss databases
- Admin controls for team design libraries, approval gates, and IP-protected geometry vaults

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Migration assistant for importing AxSTREAM/NUMECA projects and legacy Fortran meanline codes
- Configuration of custom loss model libraries and company-specific design rules
- Setup of CI/CD-style automated design validation pipelines

**Key Workflows:**
- Multi-stage compressor/turbine optimization with stage-matching constraints
- Aeroelastic assessment: Campbell diagrams, flutter margins, forced-response screening
- Custom loss model calibration against rig test data
- Automated DOE sweeps across hundreds of geometric parameters with surrogate model generation
- Coordinating multi-disciplinary reviews across aero, structures, and manufacturing teams

---

## Role-Based Personas

### Designer/Creator
**Modified UI/UX:**
- Blade geometry editor with B-spline control, lean/sweep/bow parameters, and real-time 3D preview
- Integrated meridional flow path sketcher with automatic area scheduling
- Drag-and-drop stage stacking with visual throat margin and blockage indicators

**Feature Gating:**
- Full CAD-integrated blade design tools with parametric modeling
- Direct export to blade manufacturing formats (CMM point clouds, 5-axis CNC paths)
- Access to proprietary airfoil optimization algorithms and inverse design methods

**Pricing Tier:** Professional or Enterprise tier

**Onboarding Flow:**
- Quick-start from a meridional sketch: define annulus lines, auto-generate initial blading
- Import existing CAD blade geometry and auto-extract design parameters
- Tutorial on linking parametric blade definitions to optimization loops

**Key Workflows:**
- Designing blade profiles from aerodynamic requirements using inverse methods
- Defining 3D stacking laws (lean, sweep, compound lean) for secondary flow control
- Iterating hub/shroud contours for endwall optimization
- Exporting final blade definitions with manufacturing tolerances to CAD/CAM systems
- Creating parametric blade family libraries for platform reuse

### Analyst/Simulator
**Modified UI/UX:**
- CFD workflow manager with automated meshing, solver setup, and convergence monitoring dashboards
- Performance map generation wizard with automatic speed-line extraction and surge/choke detection
- Result post-processing suite with meridional-averaged plots, blade-to-blade contours, and spanwise distributions

**Feature Gating:**
- Full CFD solver access with all turbulence models (RANS, URANS, SAS, LES for research)
- Throughflow-to-CFD automated comparison and calibration tools
- Access to transient simulations for stall inception, rotating stall, and surge studies

**Pricing Tier:** Professional or Enterprise tier

**Onboarding Flow:**
- Guided first CFD run: import geometry, auto-mesh, solve, and compare against meanline prediction
- Setup of standard post-processing templates matching company reporting requirements
- Introduction to batch-run queuing and HPC cluster configuration

**Key Workflows:**
- Running steady and transient blade-row CFD with mixing-plane and sliding-mesh interfaces
- Generating full-speed-line performance maps and extracting isentropic efficiency distributions
- Validating CFD against rig test data with automated error metrics
- Performing off-design and transient studies (inlet distortion, tip clearance sensitivity)
- Coupling aerodynamic loads to structural FEA for aeroelastic assessment

### Manufacturing/Process
**Modified UI/UX:**
- Manufacturability dashboard showing casting feasibility, machining access, and fillet radius constraints
- Tolerance stack-up visualizer for blade-to-disk attachment features (fir-tree roots, dovetails)
- Surface finish and coating specification panels linked to aerodynamic roughness models

**Feature Gating:**
- Access to manufacturing constraint checkers (minimum wall thickness, draft angles, EDM access)
- CMM inspection point generation and GD&T overlay on blade geometry
- Process simulation hooks (casting fill, forging strain) locked unless process sim module licensed

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Import a design-frozen blade and run manufacturing feasibility check as first task
- Setup of factory-specific constraint libraries (machine capabilities, material stock sizes)
- Tutorial on generating inspection programs from nominal blade definitions

**Key Workflows:**
- Checking blade designs against casting/forging/machining manufacturability rules
- Generating CMM inspection point clouds and first-article inspection reports
- Evaluating blade-to-blade variability impact on aerodynamic performance (statistical tolerance analysis)
- Defining surface finish zones and coating application requirements
- Flagging designs that violate minimum radius, wall thickness, or access constraints

### Regulatory/Compliance
**Modified UI/UX:**
- Certification evidence dashboard mapping design artifacts to FAA/EASA airworthiness requirements
- Automated traceability matrix linking every design parameter to its substantiation analysis
- Audit trail viewer showing complete design change history with approval signatures

**Feature Gating:**
- Read-only access to design modules; full access to reporting and evidence-packaging tools
- Automated compliance check against CS-E / 14 CFR Part 33 structural and performance requirements
- Digital signature and approval workflow integration for DER/CVE sign-off

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Configuration of applicable regulatory framework (FAA, EASA, CAAC) and engine type certificate basis
- Setup of certification plan template with required substantiation documents
- Walkthrough of evidence-packaging workflow for a sample blade containment analysis

**Key Workflows:**
- Generating certification substantiation reports with full traceability to analysis inputs
- Reviewing overspeed, burst, and containment analysis evidence packages
- Tracking compliance status against type certificate data sheet requirements
- Managing bird strike, ice ingestion, and fan blade-off analysis documentation
- Producing audit-ready configuration management records

### Manager/Decision-maker
**Modified UI/UX:**
- Program-level dashboard showing design maturity gates, schedule risk, and resource utilization
- Technology readiness level (TRL) tracker for each turbomachinery component
- Executive summary generator with key performance indicators vs. targets

**Feature Gating:**
- Dashboard and reporting access only; no direct design manipulation
- Portfolio view across multiple engine programs and component development efforts
- Budget tracking and resource allocation tools integrated with project management systems

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Configuration of program milestones (PDR, CDR, TRR) and KPI targets
- Integration with existing project management tools (Jira, MS Project, Primavera)
- Setup of automated weekly status report generation and distribution

**Key Workflows:**
- Monitoring design convergence and performance margin erosion across development milestones
- Comparing design alternatives using weighted multi-criteria decision matrices
- Tracking team productivity metrics (design iterations per week, CFD turnaround time)
- Reviewing technology maturation roadmaps and make/buy decisions for components
- Approving design freeze gates based on automated maturity checklists
