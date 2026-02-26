# AeroStream — Persona Subspecs

> Parent spec: [`specs/44-aerostream.md`](../44-aerostream.md)

---

## Experience-Level Personas

### Student/Academic

**Modified UI/UX:**
- Simplified dual-panel layout: structures (beam/plate elements) on left, aerodynamics (airfoil/panel methods) on right
- Inline theory annotations explaining aeroelastic concepts (flutter speed, divergence, load factors) as analyses run
- Pre-loaded example models (Cessna-like wing, simple fuselage, control surface) for immediate experimentation

**Feature Gating:**
- Linear static structural analysis, Euler beam/plate models, and panel-method aerodynamics available
- Nonlinear buckling, composite failure analysis, and CFD-coupled aeroelastics locked
- Model size limited to 50k elements; transient dynamic analysis unavailable

**Pricing Tier:** Free tier (education license with .edu verification)

**Onboarding Flow:**
- "Analyze Your First Wing" tutorial: import NACA airfoil, define spar-rib structure, apply aerodynamic loads, and review stress contours
- Pre-configured aerospace materials library (Al 2024-T3, Al 7075-T6, Ti-6Al-4V, CFRP layups) with textbook properties
- Guided comparison exercises: hand-calculated beam bending vs. FEA results to build trust in the tool

**Key Workflows:**
- Analyzing simple wing structures under aerodynamic pressure distributions from panel methods
- Running modal analysis to find natural frequencies and mode shapes of beam-element aircraft models
- Comparing panel-method lift distributions with thin-airfoil theory predictions
- Exporting stress contour plots and load diagrams for academic reports and presentations

---

### Junior Engineer (0-3 years)

**Modified UI/UX:**
- Standard aerospace-engineering workspace with structures, aerodynamics, and loads modules in tabbed layout
- Load-case manager with templates for common aerospace certification cases (gust, maneuver, landing, pressurization)
- Results comparison panel to overlay analysis results against allowable values from MMPDS/CMH-17

**Feature Gating:**
- Full linear and nonlinear static analysis, buckling, and composite laminate failure (Tsai-Wu, Hashin) unlocked
- Panel methods and RANS CFD for aerodynamic loads available
- Flutter analysis available in guided mode; full aeroelastic optimization restricted

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Role-specific setup: select domain (structures, aerodynamics, loads) to configure default toolbars and analysis templates
- Import models from NASTRAN/PATRAN bulk data format with automatic element-quality checking
- Guided workflow: "Loads Development > Structural Sizing > Margin Calculation > Report Generation"

**Key Workflows:**
- Building FE models of structural components (ribs, spars, skins, fittings) with appropriate element types
- Applying V-n diagram load cases and running static stress analysis with reserve-factor calculations
- Performing linear buckling analysis on stiffened panels and comparing to Euler/Johnson column predictions
- Running composite laminate analyses with ply-by-ply stress evaluation and failure-index assessment
- Generating stress-analysis reports with margin-of-safety tables for certification documentation

---

### Senior/Expert

**Modified UI/UX:**
- Multi-physics workspace with simultaneous access to structural FEA, CFD, aeroelastic coupling, and optimization modules
- Custom scripting console for batch load-case processing, parametric studies, and automated margin extraction
- Global model / detail model management with submodeling and load-transfer automation

**Feature Gating:**
- All capabilities unlocked: nonlinear dynamics, fatigue/damage tolerance, fluid-structure interaction, aeroelastic flutter
- Full optimization suite (topology, sizing, shape) with manufacturing constraints
- API for integration with in-house loads tools, PLM systems, and certification-tracking databases

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Migration of existing NASTRAN/ANSYS model libraries and custom analysis scripts
- Configure multi-disciplinary optimization workflows linking structures, aero, and controls
- Solutions-engineering engagement to set up enterprise integrations (PLM, certification-tracking, loads databases)

**Key Workflows:**
- Performing flutter analysis and aeroservoelastic stability assessment across the flight envelope
- Running damage-tolerance and fatigue-crack-growth analysis per FAR 25.571 requirements
- Executing multi-disciplinary optimization to minimize structural weight under strength, buckling, and flutter constraints
- Managing global/local modeling hierarchies with automated load-transfer and submodeling
- Leading model peer-reviews and mentoring junior engineers on best practices and certification methodology

---

## Role-Based Personas

### Designer/Creator

**Modified UI/UX:**
- Geometry-focused workspace with parametric surface modeler for aerodynamic shapes (fuselage, wings, nacelles)
- Real-time drag estimation overlay as outer-mold-line geometry is modified
- Visual weight-breakdown display showing how design choices affect structural mass

**Feature Gating:**
- Parametric geometry creation, panel-method aerodynamics, and conceptual structural sizing tools available
- Detailed FEA and CFD available but positioned as downstream verification
- Optimization tools focused on shape and configuration studies rather than detailed sizing

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- "Aircraft Configuration" wizard: select platform type (commercial transport, UAV, rotorcraft, fighter) to load appropriate templates
- Quick-assessment tutorial: sketch a wing planform, estimate lift and drag, and run preliminary structural sizing
- Import reference geometry from CAD or published aircraft data for parametric studies

**Key Workflows:**
- Defining wing planforms, airfoil selections, and high-lift configurations with parametric geometry tools
- Running rapid aerodynamic assessments (panel methods, empirical drag buildup) for configuration trades
- Performing preliminary structural sizing to estimate weight for different material/configuration options
- Generating configuration comparison reports with aerodynamic performance and weight estimates
- Exporting outer-mold-line geometry to downstream detailed design and analysis teams

---

### Analyst/Simulator

**Modified UI/UX:**
- Analysis-centric layout with mesh controls, boundary-condition editor, solver monitor, and post-processor in quad-panel arrangement
- Solver-performance dashboard showing convergence history, memory usage, and estimated remaining time
- Multi-load-case result navigation with batch plot generation and automated margin extraction

**Feature Gating:**
- All analysis solvers unlocked: linear/nonlinear static, buckling, dynamic, fatigue, flutter, CFD, coupled FSI
- Advanced meshing tools: mapped meshing, mesh morphing, and adaptive refinement
- Full solver-parameter control including convergence criteria, element formulations, and contact algorithms

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Solver-validation suite: run NAFEMS benchmarks and industry-standard validation cases
- Configure solver preferences, default convergence criteria, and output-request templates
- Tutorial on advanced techniques: nonlinear contact, progressive-failure composites, CFD-FEA coupling

**Key Workflows:**
- Setting up and solving complex nonlinear structural analyses (large deformation, contact, material nonlinearity)
- Running CFD simulations for aerodynamic load generation across the flight envelope
- Performing aeroelastic flutter analysis with frequency-domain and time-domain methods
- Executing fatigue and damage-tolerance assessments with crack-growth prediction
- Producing detailed analysis reports with methodology documentation, validation evidence, and results interpretation

---

### Manufacturing/Process

**Modified UI/UX:**
- Manufacturing-constraint overlay on structural models showing producibility zones, assembly sequence, and fastener patterns
- Composite layup visualization with draping feasibility indicators and ply-drop locations
- Tooling-load analysis workspace for fixture design and assembly-sequence simulation

**Feature Gating:**
- Structural analysis with manufacturing constraints (minimum gauge, ply-drop rules, fastener-edge distances)
- Composite draping simulation and flat-pattern generation
- Assembly-process simulation (rivet installation sequence, shimming analysis) available

**Pricing Tier:** Professional tier with Manufacturing add-on

**Onboarding Flow:**
- Manufacturing rules configuration: define company-specific producibility standards (min. bend radii, fastener spacing, ply-drop rules)
- Tutorial on designing with manufacturing constraints integrated into structural optimization
- Import tooling models and configure assembly-sequence analysis templates

**Key Workflows:**
- Evaluating structural designs for producibility and flagging manufacturing-constraint violations
- Running composite draping simulations to verify ply layup feasibility on complex surfaces
- Analyzing assembly loads during build sequence (jack loads, clamp-up forces, rivet installation)
- Optimizing fastener patterns for structural performance within manufacturing access constraints
- Generating manufacturing engineering data packages (flat patterns, trim boundaries, drill templates)

---

### Regulatory/Compliance

**Modified UI/UX:**
- Certification dashboard mapping every analysis to the applicable FAR/CS requirement and means-of-compliance document
- Compliance-matrix viewer with status tracking: planned, in-progress, reviewed, approved for each certification task
- DER/CVE review portal with redline markup tools on analysis reports

**Feature Gating:**
- Full analysis capabilities with emphasis on certification-methodology documentation
- Automated generation of certification-standard analysis report formats (per AC 25.307, AC 25.571 guidelines)
- Electronic signature workflows for DER/CVE approval chains

**Pricing Tier:** Enterprise tier with Certification module

**Onboarding Flow:**
- Certification framework setup: select applicable airworthiness regulations (FAR 25, CS-25, FAR 23, MIL-STD)
- Configure compliance matrix templates with means-of-compliance codes (analysis, test, similarity, inspection)
- Demo full certification workflow: requirement > analysis plan > execution > report > DER review > approval

**Key Workflows:**
- Maintaining certification compliance matrices with traceability from regulations to analysis evidence
- Generating analysis reports in formats acceptable to FAA/EASA with proper methodology documentation
- Running damage-tolerance and fatigue analyses per FAR 25.571 with required inspection-interval calculations
- Tracking certification task status and managing DER/CVE review queues
- Producing type-certificate data packages with complete substantiation evidence

---

### Manager/Decision-maker

**Modified UI/UX:**
- Program dashboard with certification progress, structural-test campaign status, weight tracking, and risk register
- Resource-allocation view showing analyst workload, solver license usage, and HPC cluster utilization
- Weight-growth trend chart with target tracking and historical comparison to similar programs

**Feature Gating:**
- Read-only access to all analysis models and reports with annotation and approval capabilities
- Full program analytics, resource planning, and schedule-tracking modules
- No direct analysis execution; all requests routed to engineering team

**Pricing Tier:** Enterprise tier (manager seat)

**Onboarding Flow:**
- Program-management setup: connect to PLM and project-scheduling tools for unified status view
- Configure weight-tracking templates and define target weight budgets by structural group
- Walkthrough of certification-progress reporting and risk-assessment frameworks

**Key Workflows:**
- Monitoring certification progress across all structural substantiation tasks
- Tracking aircraft weight growth against targets and managing weight-reduction initiatives
- Reviewing structural-test campaign plans and correlating test results with analysis predictions
- Allocating engineering resources across concurrent analysis workstreams
- Making schedule and cost trade-off decisions based on analysis maturity and certification risk
