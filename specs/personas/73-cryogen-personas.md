# CryoGen — Persona Subspecs

> Parent spec: [`specs/73-cryogen.md`](../73-cryogen.md)

---

## Experience-Level Personas

### Student/Academic
**Modified UI/UX:**
- Interactive phase diagram explorer with annotated superconducting transition regions and critical field curves
- Guided cryostat design wizard with educational callouts on heat leak paths, MLI, and thermal anchoring
- Simplified material property browser with temperature-dependent plots for common cryogenic materials

**Feature Gating:**
- 1D thermal network analysis and basic cryostat heat load estimation unlocked
- Superconductor critical current modeling limited to standard Ic(B,T) parameterizations (Kim, scaling laws)
- 3D multiphysics simulation and quench propagation locked; export to academic formats only

**Pricing Tier:** Free tier (academic license)

**Onboarding Flow:**
- Tutorial: estimate heat loads on a simple LHe cryostat with radiation shields and MLI
- Pre-loaded examples of common cryogenic systems (MRI magnet, accelerator dipole, lab dewar)
- Link to cryogenic engineering educational resources and community forum

**Key Workflows:**
- Estimating cryostat heat loads from conduction, radiation, and residual gas contributions
- Plotting superconductor Ic(B,T) surfaces for common materials (NbTi, Nb3Sn, REBCO)
- Sizing cryocooler requirements based on thermal budget analysis
- Generating annotated thermal circuit diagrams for coursework and publications

### Junior Engineer (0-3 years)
**Modified UI/UX:**
- Step-by-step workflow panels: define geometry, assign materials, set boundary conditions, solve, post-process
- Built-in material property database with validated temperature-dependent data (4 K to 300 K)
- Warning system flagging unrealistic boundary conditions or material property extrapolation

**Feature Gating:**
- Full 2D/3D thermal analysis with conduction, radiation, and contact resistance
- Superconductor stability analysis (Stekly, Maddock, equal-area) with guided setup
- Quench simulation available in guided mode; custom quench protection circuit modeling requires approval

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Role-based setup: magnet design vs. cryostat/transfer line vs. cryoplant system focus
- Walkthrough: thermal analysis of a cryostat support structure from CAD import to heat leak report
- Introduction to project versioning and team collaboration features

**Key Workflows:**
- Performing detailed thermal analysis of cryogenic vessels and support structures
- Sizing radiation shields, MLI blankets, and thermal intercept stations
- Running superconductor stability margin calculations for magnet design
- Generating thermal budget reports for system-level design reviews
- Importing CAD geometry and applying cryogenic boundary conditions

### Senior/Expert
**Modified UI/UX:**
- Multi-physics coupling console linking thermal, electromagnetic, structural, and fluid domains
- Custom material model editor for novel superconductors (HTS tapes, multi-filamentary conductors)
- Scripting/API access for automated optimization of magnet designs and cryogenic systems

**Feature Gating:**
- All modules: quench simulation with protection circuit co-simulation, AC loss modeling, cryoplant optimization
- Full API for coupling with circuit simulators, structural FEA, and CFD tools
- Admin controls for proprietary material databases and company design standards

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Migration assistant for importing COMSOL/custom tool models and legacy magnet design databases
- Configuration of custom superconductor material models and quench detection algorithms
- Setup of automated design validation workflows against performance specifications

**Key Workflows:**
- Multi-physics quench simulation with coupled thermal-electromagnetic-structural analysis
- Optimizing magnet designs for field homogeneity, stability margin, and manufacturing cost
- AC loss analysis for pulsed magnets and fault-current limiters
- Cryoplant system-level optimization including cooldown transient analysis
- Developing and validating custom material models against experimental characterization data

---

## Role-Based Personas

### Designer/Creator
**Modified UI/UX:**
- Parametric magnet geometry builder with coil winding patterns, layer definitions, and insulation schemes
- Cryostat cross-section sketcher with drag-and-drop thermal intercept and support placement
- Real-time field maps and force calculations updating as geometry is modified

**Feature Gating:**
- Full geometric modeling tools for magnets, cryostats, and transfer lines
- Parametric design with automated field quality optimization (harmonics, homogeneity)
- Export to CAD formats and manufacturing drawing generation

**Pricing Tier:** Professional or Enterprise tier

**Onboarding Flow:**
- Quick-start: design a solenoid magnet from field requirements to conductor selection
- Import existing magnet cross-section and parameterize for design exploration
- Tutorial on linking design parameters to optimization objectives

**Key Workflows:**
- Designing superconducting magnet coil cross-sections with field quality targets
- Laying out cryostat thermal architecture (shields, supports, feedthroughs)
- Optimizing conductor grading schemes to minimize superconductor usage
- Designing quench protection heater patterns and detection circuits
- Creating parametric design families for product platform development

### Analyst/Simulator
**Modified UI/UX:**
- Simulation workflow manager with mesh refinement controls, solver selection, and convergence monitoring
- Multi-domain result visualization with synchronized thermal, magnetic, and stress contour plots
- Automated report generator pulling key results (peak temperature, max field, stability margin)

**Feature Gating:**
- Full access to all solvers: thermal, electromagnetic, structural, quench propagation
- Advanced post-processing including critical current margin maps and hot-spot temperature evolution
- HPC job management for large-scale 3D transient simulations

**Pricing Tier:** Professional or Enterprise tier

**Onboarding Flow:**
- Guided first analysis: magnetic field computation for a reference solenoid with validation
- Setup of standard analysis templates (steady-state thermal, quench, AC loss)
- Introduction to parametric study setup and result database management

**Key Workflows:**
- Computing magnetic field distributions and force loads on superconducting coils
- Running transient quench propagation simulations with protection circuit response
- Performing thermal-structural analysis for cooldown stress and Lorentz force assessment
- Calculating AC losses in superconductors under time-varying fields
- Validating simulation results against magnet test data and updating material models

### Manufacturing/Process
**Modified UI/UX:**
- Winding process planning dashboard with tension profiles, layer transitions, and splice locations
- Manufacturing tolerance analyzer showing field quality sensitivity to winding placement errors
- Assembly sequence visualizer for cryostat integration steps with thermal contraction allowances

**Feature Gating:**
- Access to manufacturing feasibility checkers (bend radius, winding tension, impregnation flow)
- Tolerance analysis tools for as-built geometry impact on performance
- Process simulation (epoxy impregnation, heat treatment) available at Enterprise tier

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Configure manufacturing facility capabilities (winding machines, heat treatment furnaces, test equipment)
- Walkthrough: generate a winding plan from a finalized magnet design
- Tutorial on thermal contraction compensation for cryostat assembly planning

**Key Workflows:**
- Generating winding plans with layer schedules, splice locations, and tension profiles
- Analyzing manufacturing tolerance impact on magnetic field quality
- Planning heat treatment schedules for Nb3Sn wind-and-react magnets
- Defining assembly procedures accounting for differential thermal contraction
- Creating incoming inspection criteria for superconductor wire/tape lots

### Regulatory/Compliance
**Modified UI/UX:**
- Safety compliance dashboard for pressure vessel codes (ASME BPVC), cryogenic safety (CGA), and ODH analysis
- Traceability matrix linking design parameters to safety analysis and test evidence
- Automated oxygen deficiency hazard (ODH) calculation wizard

**Feature Gating:**
- Read-only design access; full reporting and compliance verification tools
- Automated pressure vessel code compliance checking
- Safety analysis documentation generators (FMEA, ODH, pressure relief sizing)

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Configuration of applicable codes and standards (ASME, EN 13458, CGA, site-specific safety requirements)
- Setup of safety documentation templates and review/approval workflows
- Walkthrough: generate an ODH assessment for a sample cryogenic installation

**Key Workflows:**
- Verifying cryostat designs against ASME Boiler and Pressure Vessel Code requirements
- Performing and documenting oxygen deficiency hazard (ODH) analyses
- Sizing and documenting pressure relief system adequacy
- Generating safety analysis reports for facility installation permits
- Maintaining compliance records and managing periodic requalification schedules

### Manager/Decision-maker
**Modified UI/UX:**
- Program dashboard tracking magnet/cryogenic system development milestones and technical risks
- Cost estimation tool comparing superconductor material options, cryocooler selections, and operating costs
- Technology readiness and schedule risk visualization across multiple development programs

**Feature Gating:**
- Dashboard and reporting access; no direct design or simulation tools
- Cost modeling, trade study comparison, and resource planning views
- Portfolio-level risk assessment and mitigation tracking

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Configuration of program structure, milestones, and key performance targets
- Integration with project management and procurement systems
- Setup of automated technical status and cost reporting

**Key Workflows:**
- Monitoring technical performance vs. requirements across development milestones
- Evaluating superconductor technology trade-offs (LTS vs. HTS cost/performance)
- Tracking cryocooler and helium cost projections for operating expense planning
- Reviewing risk registers and mitigation plans for magnet and cryogenic systems
- Making go/no-go decisions at design review gates based on automated maturity assessments
