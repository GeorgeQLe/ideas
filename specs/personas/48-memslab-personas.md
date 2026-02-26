# MEMSLab — Persona Subspecs

> Parent spec: [`specs/48-memslab.md`](../48-memslab.md)

---

## Experience-Level Personas

### Student/Academic

**Modified UI/UX:**
- Simplified cross-section editor with drag-and-drop layer builder using pre-defined material stacks (silicon, oxide, nitride, metal)
- Inline process-flow visualizer showing how each fabrication step (deposition, etch, doping, release) shapes the device
- Interactive 2D/3D device viewer with labeled dimensions, material properties, and mode-shape animations

**Feature Gating:**
- Electrostatic, structural (modal, static), and basic thermal analyses available
- Coupled multi-physics (piezoelectric, piezoresistive, fluidic), process simulation, and yield analysis locked
- Device geometry limited to 2D cross-sections and simple 3D extrusions; full 3D process emulation unavailable

**Pricing Tier:** Free tier (academic license with institutional verification)

**Onboarding Flow:**
- "Design Your First MEMS Device" tutorial: build a cantilever beam, apply electrostatic actuation, simulate deflection and resonance
- Pre-loaded example devices (accelerometer, pressure sensor, resonator, comb drive) with annotated design rationale
- Guided comparison of simulation results with analytical MEMS formulas (Euler-Bernoulli beam, parallel-plate capacitor)

**Key Workflows:**
- Building MEMS cross-sections layer by layer and visualizing the resulting 3D device geometry
- Running modal analysis to determine resonant frequencies and mode shapes of MEMS structures
- Simulating electrostatic actuation with pull-in voltage prediction for parallel-plate and comb-drive actuators
- Exporting analysis results and device visualizations for academic papers and thesis documentation

---

### Junior Engineer (0-3 years)

**Modified UI/UX:**
- Professional layout with process-flow editor, device modeler, multi-physics simulator, and results viewer in tabbed panels
- Design-rule checker validating geometry against foundry design rules (min. feature size, layer overlap, etch selectivity)
- Sensitivity analysis wizard showing how geometric and process variations affect device performance

**Feature Gating:**
- Full electrostatic-structural coupling, thermal analysis, and basic piezoelectric/piezoresistive simulation unlocked
- 3D process emulation for standard MEMS fabrication flows (PolyMUMPs, SOIMUMPs, PiezoMUMPs)
- Advanced fluidic analysis (squeeze-film damping, microfluidics) available in guided mode; custom process flows restricted

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Foundry selection wizard: choose target MEMS foundry (MEMSCAP, STMicroelectronics, Bosch) to load design rules and process stack
- Guided design workflow: define performance specs > select topology > design geometry > simulate > optimize > verify DRC
- Import existing designs from CoventorWare or Intellisense with geometry and material mapping

**Key Workflows:**
- Designing MEMS inertial sensors (accelerometers, gyroscopes) with electrostatic readout and actuation
- Running coupled electrostatic-structural simulations to predict sensitivity, full-scale range, and linearity
- Performing squeeze-film damping analysis to determine bandwidth and noise performance
- Checking designs against foundry design rules and iterating to resolve violations
- Generating design documentation with layout drawings, simulation results, and performance-specification tables

---

### Senior/Expert

**Modified UI/UX:**
- Multi-physics orchestration workspace with simultaneous structural, electrostatic, thermal, fluidic, and circuit co-simulation
- Custom material-model editor for defining anisotropic material properties, residual stress profiles, and etch-rate models
- Process-simulation environment for developing and validating custom fabrication flows with lithography and etch models

**Feature Gating:**
- All capabilities unlocked: full 3D process emulation, multi-physics coupling, reduced-order model extraction, circuit co-simulation
- Custom process-flow definition with deposition, lithography, etch (isotropic/anisotropic), CMP, and bonding steps
- API for integration with IC design tools (Cadence, Synopsis), foundry PDKs, and test-equipment automation

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Custom process-flow migration from CoventorWare SEMulator3D or in-house process-simulation tools
- Configure reduced-order-model extraction pipelines for MEMS-IC co-simulation in Cadence or Synopsys environments
- Solutions-engineering session for foundry integration, test-equipment interfaces, and production-data feedback loops

**Key Workflows:**
- Developing custom MEMS fabrication processes and validating with 3D process-emulation simulations
- Designing advanced MEMS devices (CMUT arrays, RF switches, micro-mirrors) with multi-physics optimization
- Extracting reduced-order behavioral models for system-level MEMS-IC co-simulation
- Performing yield analysis with Monte Carlo process-variation studies across all critical fabrication steps
- Mentoring team on MEMS design methodology and reviewing designs against best practices

---

## Role-Based Personas

### Designer/Creator

**Modified UI/UX:**
- Layout editor with parametric geometry tools, symmetry constraints, and array generators for MEMS-specific patterns
- Visual process-stack builder with layer-by-layer animation of the fabrication sequence
- Quick-simulation mode for rapid evaluation of design concepts without full multi-physics setup

**Feature Gating:**
- Full geometry creation with parametric design, layout generation, and DRC checking
- Quick-estimate tools for key performance metrics (resonant frequency, sensitivity, pull-in voltage) without full simulation
- Multi-physics simulation available for design validation but not primary workflow

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- "MEMS Design Patterns" gallery showcasing proven topologies (comb drive, proof mass, diaphragm, cantilever array)
- Interactive tutorial on parametric geometry creation with design-intent constraints
- Quick-reference calculator for common MEMS analytical estimates (beam stiffness, plate capacitance, squeeze-film damping)

**Key Workflows:**
- Creating parametric MEMS device layouts with foundry-compatible layer definitions
- Rapidly evaluating design variants using analytical estimates before committing to full simulation
- Generating layout files (GDS-II, OASIS) for foundry submission with DRC-clean verification
- Designing test structures (vernier gauges, process monitors) alongside functional devices
- Iterating on geometry parameters to meet performance targets while satisfying fabrication constraints

---

### Analyst/Simulator

**Modified UI/UX:**
- Simulation-centric workspace with meshing controls, multi-physics setup panels, solver monitor, and post-processing tools
- Mesh-quality diagnostics with element-distortion indicators and automatic refinement suggestions
- Multi-physics coupling manager showing data-flow between electrostatic, structural, thermal, and fluidic domains

**Feature Gating:**
- All simulation capabilities unlocked with full solver control (element types, convergence criteria, coupling algorithms)
- Advanced post-processing: frequency response, harmonic distortion, noise analysis, and parametric sweeps
- Geometry editing available for simulation model preparation (defeaturing, symmetry exploitation) but not primary design

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Validation suite: simulate canonical MEMS devices and compare with published analytical and experimental results
- Configure solver preferences for accuracy/speed trade-offs appropriate to device type and physics
- Tutorial on multi-physics coupling strategies and convergence techniques for strongly coupled problems

**Key Workflows:**
- Setting up and solving coupled electrostatic-structural problems with contact and large-deflection nonlinearity
- Performing harmonic analysis to characterize frequency response, Q-factor, and phase noise
- Running thermo-mechanical stress analysis for packaging effects and temperature-compensation design
- Executing parametric sweeps to generate performance maps across the design space
- Documenting simulation methodology, mesh-convergence studies, and validation evidence for design reviews

---

### Manufacturing/Process

**Modified UI/UX:**
- Process-monitoring dashboard with wafer-level yield maps, critical-dimension measurements, and process-parameter trends
- Statistical process control (SPC) panel with control charts for key fabrication parameters
- Process-simulation workspace focused on etch profiles, film stress, and lithographic resolution

**Feature Gating:**
- Full process emulation and simulation with emphasis on manufacturability and yield prediction
- Wafer-level yield modeling with spatial variation patterns (across-wafer, wafer-to-wafer, lot-to-lot)
- SPC integration with fab-data feeds for continuous process monitoring

**Pricing Tier:** Professional tier with Manufacturing add-on

**Onboarding Flow:**
- Fab-integration setup: connect to process-data systems (MES, metrology tools) for automated model calibration
- Configure process-control limits and yield-target definitions for each critical fabrication step
- Tutorial on process-simulation calibration using measured etch profiles and film properties

**Key Workflows:**
- Simulating fabrication-process variations and predicting their impact on device performance and yield
- Monitoring critical-dimension trends and flagging process drift before it affects device yield
- Optimizing process recipes (etch time, deposition temperature, exposure dose) for target device geometry
- Generating wafer-level yield predictions based on process-variation models and device-performance specs
- Correlating process-simulation predictions with inline metrology data for continuous model improvement

---

### Regulatory/Compliance

**Modified UI/UX:**
- Reliability-qualification dashboard tracking JEDEC/AEC-Q100 test requirements and simulation-based evidence
- Environmental-compliance panel for RoHS, REACH, and conflict-mineral material tracking
- Change-management interface with impact assessment for process or design modifications

**Feature Gating:**
- Full simulation with emphasis on reliability prediction (fatigue, creep, stiction, charging, radiation)
- Automated reliability-test simulation (thermal cycling, vibration, humidity) mapping to JEDEC qualification standards
- Electronic signatures and audit trails on all design and process records

**Pricing Tier:** Enterprise tier with Reliability module

**Onboarding Flow:**
- Qualification-profile setup: select target standard (JEDEC, AEC-Q100, MIL-STD-883) to configure test-simulation templates
- Configure material-compliance tracking for all layers in the process stack
- Demo reliability-qualification workflow: define test conditions > simulate > evaluate > document > approve

**Key Workflows:**
- Running thermal-cycling fatigue simulations to predict MEMS device lifetime under qualification conditions
- Performing stiction risk analysis for released MEMS structures under humidity and temperature exposure
- Simulating electrostatic charging effects and predicting long-term drift in capacitive MEMS devices
- Generating qualification-evidence packages mapping simulation results to JEDEC test requirements
- Managing process and design changes with reliability impact assessment and re-qualification planning

---

### Manager/Decision-maker

**Modified UI/UX:**
- Product-line dashboard with device-performance benchmarks, yield trends, reliability status, and time-to-market tracking
- Technology-roadmap view showing process-capability evolution and planned device generations
- Cost-per-die breakdown showing contributions from wafer processing, testing, packaging, and yield loss

**Feature Gating:**
- Read-only access to design models, simulation results, and process data with annotation and approval tools
- Full project analytics, yield trending, and technology-roadmap planning
- No direct design or simulation execution; all changes through engineering team

**Pricing Tier:** Enterprise tier (manager seat)

**Onboarding Flow:**
- Product-management dashboard setup: connect to fab MES, test data, and ERP for unified metrics
- Configure yield-target thresholds, cost models, and product-performance benchmarks
- Walkthrough of technology-roadmap planning tools and competitive-benchmarking framework

**Key Workflows:**
- Monitoring product-line yield trends and identifying systematic issues requiring engineering intervention
- Evaluating technology-roadmap options (new process nodes, materials, device architectures) for strategic planning
- Reviewing cost-per-die breakdowns and approving cost-reduction initiatives
- Tracking reliability-qualification status and approving product releases for production
- Allocating engineering and fab resources across concurrent MEMS product-development programs
