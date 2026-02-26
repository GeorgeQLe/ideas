# GeoDesk — Persona Subspecs

> Parent spec: [`specs/50-geodesk.md`](../50-geodesk.md)

---

## Experience-Level Personas

### Student/Academic

**Modified UI/UX:**
- Simplified soil-profile builder with drag-and-drop layers and pre-defined soil types (sand, clay, silt, gravel, rock) from standard classifications
- Inline geotechnical theory panel explaining effective stress, Mohr-Coulomb failure, consolidation, and seepage concepts alongside active analysis
- Pre-loaded example problems (retaining wall, slope stability, bearing capacity, consolidation settlement) mapped to textbook exercises

**Feature Gating:**
- Limit-equilibrium slope stability, bearing capacity calculations, 1D consolidation, and seepage flow nets available
- Finite-element analysis, advanced constitutive models (Hardening Soil, Soft Soil Creep), and staged-construction simulation locked
- Geometry limited to 2D cross-sections; probabilistic analysis and sensitivity studies unavailable

**Pricing Tier:** Free tier (education license with .edu verification)

**Onboarding Flow:**
- "Analyze Your First Slope" tutorial: define soil profile, water table, and failure surface, then compute factor of safety
- Pre-loaded soil-property database with textbook values (Bowles, Das, Holtz & Kovacs) and USCS/AASHTO classifications
- Guided comparison exercises: hand-calculated bearing capacity vs. tool results to validate understanding

**Key Workflows:**
- Building layered soil profiles with groundwater conditions and computing effective stress distributions
- Running limit-equilibrium slope-stability analysis (Bishop, Janbu, Spencer methods) and finding critical slip surfaces
- Calculating shallow-foundation bearing capacity using Terzaghi, Meyerhof, and Hansen methods
- Performing 1D consolidation settlement predictions and time-rate analysis
- Exporting analysis plots and factor-of-safety results for assignments and lab reports

---

### Junior Engineer (0-3 years)

**Modified UI/UX:**
- Professional workspace with borehole-log manager, soil-profile modeler, analysis setup panel, and results viewer
- Automated critical-surface search algorithms with interactive refinement of slip-surface geometry
- Design-code checker evaluating results against applicable standards (Eurocode 7, AASHTO LRFD, BS 8004, AS 4678)

**Feature Gating:**
- Full limit-equilibrium analysis, 2D finite-element analysis, and seepage analysis unlocked
- Staged-construction simulation and basic constitutive models (Mohr-Coulomb, Hardening Soil) available
- Advanced constitutive models (Soft Soil Creep, Modified Cam-Clay, user-defined) and 3D analysis restricted

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Borehole-data import wizard: load field investigation data (AGS, CSV, gINT) and auto-generate interpreted soil profiles
- Guided first-analysis workflow: import borehole data > create ground model > define structure > analyze > check against code
- Design-code setup: select applicable standard (Eurocode 7, AASHTO, BS) to configure partial factors and load combinations

**Key Workflows:**
- Interpreting borehole logs and creating geotechnical ground models with spatial variability
- Performing slope-stability analyses for cut slopes, embankments, and natural hillsides with water-table effects
- Running 2D finite-element analyses for retaining walls, excavations, and shallow foundations
- Designing pile foundations with axial capacity, lateral response, and settlement predictions
- Generating geotechnical design reports with analysis methodology, results summaries, and code-compliance checks

---

### Senior/Expert

**Modified UI/UX:**
- Multi-analysis workspace with simultaneous 2D/3D FEA, limit-equilibrium, seepage, and consolidation analyses
- Custom constitutive-model framework for implementing advanced or research soil models with user-defined equations
- Scripting environment for parametric studies, automated sensitivity analyses, and batch processing of design scenarios

**Feature Gating:**
- All capabilities unlocked: 3D FEA, dynamic/seismic analysis, coupled flow-deformation, advanced constitutive models, probabilistic methods
- User-defined constitutive models with custom stress-strain-strength formulations
- API for integration with GIS, BIM, monitoring systems, and geotechnical databases

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Migration of existing PLAXIS/GeoStudio models with material-model mapping and results comparison
- Configure custom constitutive-model library with calibration workflows and validation benchmarks
- Enterprise integration setup with GIS (ArcGIS, QGIS), BIM (Revit, Bentley), and instrumentation monitoring platforms

**Key Workflows:**
- Performing advanced numerical analyses for complex geotechnical problems (deep excavations, tunneling, dam foundations)
- Implementing and calibrating advanced constitutive models for site-specific soil behavior
- Running probabilistic and reliability analyses to quantify geotechnical risk and uncertainty
- Conducting seismic site-response analysis and liquefaction assessment for critical infrastructure
- Mentoring junior engineers on geotechnical analysis methodology and reviewing complex analyses

---

## Role-Based Personas

### Designer/Creator

**Modified UI/UX:**
- Geotechnical-design workspace with interactive geometry tools for retaining structures, foundations, slopes, and embankments
- Rapid-design calculators for common elements (gravity walls, MSE walls, spread footings, piles) with code-based sizing
- Visual feedback showing how design changes (wall height, embedment depth, reinforcement spacing) affect factor of safety

**Feature Gating:**
- Full geometry design tools with code-based design calculators and limit-equilibrium analysis
- Pre-built design templates for standard geotechnical structures (retaining walls, slopes, foundations)
- Advanced FEA available for design verification but not primary workflow

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- "Design Templates" gallery: gravity retaining wall, MSE wall, cantilever wall, slope reinforcement, raft foundation
- Interactive sizing tutorial: define design loads and ground conditions, let the tool size the structure, then verify with analysis
- Configure default design standards and partial-factor sets for consistent code-compliant design output

**Key Workflows:**
- Designing retaining walls (gravity, cantilever, MSE) with automated sliding, overturning, and bearing checks
- Sizing shallow and deep foundations for structural loads with settlement and capacity verification
- Designing reinforced slopes and embankments with geosynthetic or soil-nail reinforcement
- Evaluating ground-improvement options (preloading, stone columns, deep mixing) and their effects
- Generating design drawings and specifications for construction documentation

---

### Analyst/Simulator

**Modified UI/UX:**
- Analysis-centric layout with mesh controls, staged-construction manager, constitutive-model calibrator, and results post-processor
- Convergence dashboard showing solver iterations, plasticity points, and displacement increments
- Multi-stage result navigator stepping through construction phases with deformation, stress, and pore-pressure overlays

**Feature Gating:**
- All analysis solvers unlocked: 2D/3D FEA, coupled flow-deformation, dynamic, consolidation, limit equilibrium
- Full constitutive-model library with calibration tools (triaxial-test curve fitting, oedometer back-analysis)
- Advanced meshing with adaptive refinement, interface elements, and anchor/reinforcement modeling

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Validation benchmark suite: run PLAXIS benchmarks and published case histories, compare with known solutions
- Configure solver defaults and constitutive-model preferences based on typical soil types and analysis complexity
- Tutorial on staged-construction modeling: excavation sequence, dewatering, structural installation, and backfilling

**Key Workflows:**
- Setting up staged-construction FEA for deep excavations with multiple support levels and dewatering
- Calibrating constitutive models from laboratory and field test data (triaxial, oedometer, CPT, SPT)
- Running coupled consolidation analyses to predict long-term settlement and pore-pressure dissipation
- Performing seismic deformation analysis using equivalent-linear or nonlinear dynamic methods
- Producing detailed analysis reports with methodology documentation, parametric sensitivity, and results interpretation

---

### Manufacturing/Process

**Modified UI/UX:**
- Construction-monitoring workspace with as-built vs. design comparison tools and real-time instrumentation data display
- Excavation-sequence tracker showing planned vs. actual construction progress with geotechnical impact assessment
- Quality-control checklist generator for earthworks, foundation construction, and ground-improvement verification

**Feature Gating:**
- Full analysis capabilities with emphasis on construction-phase prediction and monitoring
- Instrumentation data integration (inclinometers, piezometers, settlement gauges) with real-time comparison to predictions
- Construction-sequence optimization tools for minimizing risk and cost during execution

**Pricing Tier:** Professional tier with Construction add-on

**Onboarding Flow:**
- Instrumentation setup: connect to field monitoring systems (IoT sensors, data loggers) for automated data import
- Configure alert thresholds for critical deformation and pore-pressure levels during construction
- Tutorial on observational method: update analysis models with field data and adjust construction approach

**Key Workflows:**
- Monitoring construction-induced ground movements against predicted values and trigger levels
- Updating analysis models with as-built conditions and recalculating expected performance
- Verifying earthwork compaction and ground-improvement effectiveness through field-test data correlation
- Managing construction-sequence changes with geotechnical impact assessment
- Generating construction-monitoring reports with instrumentation data, predictions, and compliance status

---

### Regulatory/Compliance

**Modified UI/UX:**
- Code-compliance dashboard mapping every analysis result to applicable design-standard requirements and partial factors
- Environmental-impact assessment panel tracking groundwater, contamination, and ecological considerations
- Peer-review portal with structured review checklists aligned with professional-practice standards

**Feature Gating:**
- Full analysis with emphasis on code-compliant design verification and documentation
- Automated checking against Eurocode 7, AASHTO LRFD, BS 8004, and other national standards
- Electronic peer-review workflows with tracked comments, responses, and sign-off

**Pricing Tier:** Enterprise tier with Compliance module

**Onboarding Flow:**
- Regulatory-profile setup: select applicable design codes and local authority requirements
- Configure peer-review templates with required-check items per professional-practice guidelines
- Demo compliance workflow: analysis > code check > peer review > client approval > record retention

**Key Workflows:**
- Verifying geotechnical designs against applicable design codes with documented partial-factor application
- Performing independent design checks and peer reviews with structured checklists and tracked responses
- Assessing environmental impacts of construction (dewatering effects, contamination risks, ecological considerations)
- Generating code-compliance documentation for building permit and planning applications
- Maintaining professional-practice records with full audit trail of design decisions and approvals

---

### Manager/Decision-maker

**Modified UI/UX:**
- Project portfolio dashboard with geotechnical-risk ratings, schedule status, and cost tracking across all active sites
- Risk heat map showing ground conditions, design complexity, and construction risk for each project
- Cost-estimating tool linking geotechnical design choices (foundation type, ground improvement) to construction cost

**Feature Gating:**
- Read-only access to all analysis models and reports with annotation and approval tools
- Full project analytics, risk tracking, and resource-allocation dashboards
- No direct analysis execution; all technical work routed through engineering team

**Pricing Tier:** Enterprise tier (manager seat)

**Onboarding Flow:**
- Portfolio-management setup: import project list with site locations, ground conditions, and schedule milestones
- Configure geotechnical risk-rating framework aligned with company risk-management policy
- Walkthrough of cost-estimation tools and how geotechnical conditions drive foundation and earthwork costs

**Key Workflows:**
- Reviewing geotechnical risk assessments and allocating appropriate investigation and design budgets
- Monitoring construction-phase geotechnical performance across multiple active projects
- Evaluating the cost impact of geotechnical design alternatives for value-engineering decisions
- Allocating geotechnical engineering resources across concurrent projects based on complexity and risk
- Communicating geotechnical risk and mitigation strategies to clients, contractors, and stakeholders
