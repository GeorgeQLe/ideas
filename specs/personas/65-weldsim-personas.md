# WeldSim — Persona Subspecs

> Parent spec: [`specs/65-weldsim.md`](../65-weldsim.md)

---

## Experience-Level Personas

### Student/Academic

**Modified UI/UX:**
- Guided mode with annotated diagrams explaining welding thermal cycles, phase transformations, and residual stress formation
- Interactive material science panel linking to welding metallurgy fundamentals (Kou, Lancaster textbooks)
- Simplified heat source models (Goldak double-ellipsoid) with visual parameter adjustment and instant preview

**Feature Gating:**
- 2D cross-section and simplified 3D weld simulation with single-pass welds only; multi-pass limited to 5 passes
- Pre-built material library (carbon steel, stainless steel, aluminum) with standard CCT/TTT diagrams; custom materials locked
- Export to academic formats (plots, CSV, VTK); no WPS/PQR generation

**Pricing Tier:** Free tier (academic email verification)

**Onboarding Flow:**
- Start with a single-pass butt weld tutorial demonstrating thermal cycle, HAZ microstructure, and distortion prediction
- Interactive module on Goldak heat source calibration against experimental macrographs
- Access to published experimental validation datasets for thesis research

**Key Workflows:**
- Simulating single-pass welds to study thermal cycles and HAZ microstructure evolution
- Comparing heat source models and their effect on predicted fusion zone geometry
- Analyzing residual stress distributions in simple joint configurations for research
- Generating temperature-time history plots and CCT diagram overlays for coursework

---

### Junior Engineer (0-3 years)

**Modified UI/UX:**
- Template-based setup organized by joint type (butt, fillet, T-joint, lap) and welding process (GMAW, GTAW, SAW, laser)
- Automatic heat source calibration wizard that fits Goldak parameters to specified weld bead geometry
- Traffic-light quality indicators on mesh, time step, and convergence for each simulation

**Feature Gating:**
- Multi-pass welding (up to 20 passes) with standard material models; advanced user subroutines locked
- Distortion and residual stress prediction enabled; fracture mechanics post-processing requires senior review
- WPS documentation assistance available; PQR simulation-based qualification requires approval

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Select industry (shipbuilding, pressure vessels, automotive, aerospace) for tailored joint templates and material libraries
- Guided multi-pass fillet weld tutorial with step-by-step weld sequence definition and result interpretation
- Automatic mesh sensitivity study on first simulation with guidance on acceptable mesh density

**Key Workflows:**
- Setting up multi-pass weld simulations for common joint configurations in production
- Predicting distortion to optimize clamping fixtures and welding sequence before physical trials
- Running welding parameter studies (heat input, interpass temperature, travel speed) for procedure development
- Comparing weld sequences to minimize distortion and residual stress
- Generating engineering reports with distortion maps and residual stress contours for design review

---

### Senior/Expert

**Modified UI/UX:**
- Full scripting API (Python) for custom material models, automated weld path programming, and batch parametric studies
- Direct solver access for coupled thermal-metallurgical-mechanical analysis with advanced phase transformation models
- Multi-weld orchestration workspace managing complex fabrication sequences with 100+ passes

**Feature Gating:**
- All features unlocked: multi-pass (200+ passes), dissimilar metal welds, friction stir welding, electron beam
- Custom material model development with user-defined phase transformation kinetics and hardness prediction
- Simulation-based WPQ support for ASME IX, AWS D1.1, EN ISO 15614 with regulatory documentation

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Import existing Sysweld, Simufact, or ANSYS welding models with parameter translation
- Benchmark against published residual stress measurement data (neutron diffraction, contour method)
- Configure automated fabrication simulation pipelines for production support

**Key Workflows:**
- Simulating complex multi-pass fabrication sequences (100+ passes) for nuclear and offshore structures
- Developing and validating custom material models for dissimilar metal welds and advanced alloys
- Running simulation-supported weld procedure qualification to reduce physical test coupon requirements
- Predicting long-term stress relaxation and creep effects on residual stress for fitness-for-service assessment
- Leading fabrication sequence optimization projects across multiple welded assemblies

---

## Role-Based Personas

### Designer/Creator

**Modified UI/UX:**
- Joint design workspace with parametric weld geometry editor (groove angle, root gap, bead placement)
- Real-time distortion preview during weld sequence planning with clamping fixture design tools
- Design variant manager for comparing joint configurations and welding approaches

**Feature Gating:**
- Full geometry creation with weld joint templates and parametric modification tools
- Automated mesh generation with weld-specific refinement (HAZ, fusion zone, near-surface)
- Weld sequence optimizer with automated permutation of pass ordering and direction

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Import welded assembly CAD and auto-detect weld joints with suggested joint preparation geometry
- Template library of standard joint configurations (AWS, ISO) as starting points
- Walk through a weld sequence design study demonstrating distortion optimization

**Key Workflows:**
- Designing weld joint preparations to minimize distortion while ensuring adequate penetration
- Optimizing welding sequences across multi-joint assemblies to meet dimensional tolerances
- Designing clamping and fixturing strategies based on predicted distortion patterns
- Evaluating alternative welding processes (arc vs. laser vs. hybrid) for new joint designs
- Creating production-ready welding instruction packages with optimized parameters and sequences

---

### Analyst/Simulator

**Modified UI/UX:**
- Analysis dashboard with thermal cycle monitors, phase fraction evolution, and mechanical convergence indicators
- Multi-case comparison workspace for parametric studies across welding parameters and sequences
- Validation panel with experimental data overlay capabilities (thermocouple, strain gauge, diffraction data)

**Feature Gating:**
- All simulation capabilities available based on license tier
- Advanced post-processing: fracture mechanics (J-integral, CTOD), fatigue crack growth, stress intensity factors
- Batch simulation with automated parameter sweeps across heat input, interpass temperature, and preheat

**Pricing Tier:** Professional or Enterprise tier

**Onboarding Flow:**
- Tutorial on heat source calibration using experimental macrograph data
- Guided validation exercise comparing residual stress predictions against neutron diffraction measurements
- Introduction to fracture mechanics post-processing for fitness-for-service assessment

**Key Workflows:**
- Running high-fidelity thermo-metallurgical-mechanical weld simulations with detailed phase transformation
- Calibrating heat source models against experimental weld pool geometry and thermal cycle data
- Validating residual stress predictions against measurement data (hole drilling, neutron diffraction, contour)
- Performing parametric studies on welding parameters to optimize quality and minimize defect risk
- Generating residual stress inputs for fracture mechanics and fitness-for-service assessments

---

### Manufacturing/Process

**Modified UI/UX:**
- Shop-floor oriented interface with welding instruction cards showing optimized parameters and sequences
- Real-time production tracker linking robotic welding cell data to simulation predictions
- Distortion monitoring dashboard comparing measured distortion (CMM/laser scan) against predicted values

**Feature Gating:**
- Simplified simulation mode for rapid parameter adjustment within validated production envelope
- Welding instruction package generation with visual pass maps, parameters, and inspection requirements
- Dimensional conformance prediction linked to production tolerance specifications

**Pricing Tier:** Professional tier (manufacturing license)

**Onboarding Flow:**
- Connect to robotic welding cell controllers and CMM systems for automatic data exchange
- Calibrate production models against first-article inspection data
- Train welding supervisors on interpreting simulation-based welding instructions and distortion maps

**Key Workflows:**
- Generating optimized welding procedure specifications (WPS) for production implementation
- Predicting and compensating for distortion in production welding sequences
- Evaluating the impact of production parameter deviations on weld quality and distortion
- Supporting root cause analysis for welding defects (cracking, porosity, lack of fusion)
- Planning pre-bending and post-weld heat treatment strategies based on simulation

---

### Regulatory/Compliance

**Modified UI/UX:**
- Compliance workspace organized by code (ASME IX, AWS D1.1, EN ISO 15614) with checklist-driven documentation
- Full audit trail of simulation inputs, solver versions, material data sources, and results with timestamps
- PQR/WPS documentation templates auto-populated from simulation results

**Feature Gating:**
- Read-only access to simulation results and fabrication documentation
- Automated PQR documentation generation from simulation with code-required essential/supplementary variable tracking
- Version-controlled welding procedure archive with approval workflow and digital signatures

**Pricing Tier:** Enterprise tier (compliance add-on)

**Onboarding Flow:**
- Select applicable welding codes and standards to auto-configure documentation requirements
- Walk through a simulation-supported PQR submission demonstrating evidence requirements
- Configure approval workflow for WPS/PQR documentation with authorized inspector sign-off

**Key Workflows:**
- Generating welding procedure qualification records (PQR) supported by simulation evidence
- Documenting essential variables and their acceptable ranges per applicable welding code
- Maintaining WPS/PQR libraries with revision control and code compliance tracking
- Supporting third-party audit and authorized inspector review of welding documentation
- Tracking welding code changes and evaluating their impact on existing qualified procedures

---

### Manager/Decision-maker

**Modified UI/UX:**
- Program dashboard with fabrication project status, welding cost estimates, and quality metrics
- Cost-benefit view comparing simulation-optimized fabrication against traditional trial-and-error approaches
- Resource allocation panel showing simulation compute usage and engineering capacity across projects

**Feature Gating:**
- View-only access to project summaries, distortion reduction metrics, and cost savings reports
- Portfolio view across fabrication programs with schedule and quality tracking
- Budget management for simulation compute and physical testing allocation

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Configure project portfolio linking simulation activities to fabrication program milestones
- Walk through ROI framework: simulation cost vs. rework avoided, tryout iterations reduced
- Set up automated reporting on quality metrics and cost savings from simulation-informed fabrication

**Key Workflows:**
- Reviewing fabrication program status and simulation-predicted quality outcomes
- Approving welding procedure development budgets and simulation resource allocation
- Evaluating cost savings from reduced die tryout iterations and rework through simulation
- Comparing fabrication approaches (manual vs. robotic, multi-pass vs. narrow-gap) on cost and quality
- Generating management reports on welding quality metrics and continuous improvement trends
