# FoamSim — Persona Subspecs

> Parent spec: [`specs/89-foamsim.md`](../89-foamsim.md)

---

## Experience-Level Personas

### Student/Academic
**Modified UI/UX:**
- Simplified mold-fill visualization with preset part geometries (cup, tray, block, insulation panel) and standard foam formulations
- Animated bubble nucleation and growth visualization showing cell-size evolution, density gradients, and skin formation
- Interactive parameter panel with sliders for blowing-agent concentration, mold temperature, and injection rate

**Feature Gating:**
- 2D cross-section simulations with basic foam expansion kinetics and mold filling
- Material library limited to common polymers (PU flexible, PU rigid, EPS, XPS) with textbook properties
- Simulation limited to single-cavity, simple geometries; multi-cavity and runner systems locked

**Pricing Tier:** Free tier

**Onboarding Flow:**
- Tutorial: simulate foam expansion in a rectangular mold, observe fill pattern, and analyze density distribution
- Pre-loaded examples (PU pour-in-place, EPS bead molding, structural foam injection)
- References to Klempner & Sendijarevic, Lee & Ramesh foam textbooks linked from model descriptions

**Key Workflows:**
- Simulate foam expansion and mold filling for a simple part geometry
- Observe density distribution and identify potential voids or underfill regions
- Compare cell-size distributions under different blowing-agent concentrations
- Analyze effect of mold temperature on skin thickness and surface quality

---

### Junior Engineer (0-3 years)
**Modified UI/UX:**
- Project workspace with tabs: Part/Mold | Formulation | Process Parameters | Simulation | Quality Analysis
- Integrated formulation database with supplier-linked material datasheets and processing windows
- Fill-pattern animation with overlaid quality indicators (knit lines, air traps, density variation)

**Feature Gating:**
- Full 3D mold-filling simulation with foam expansion, heat transfer, and curing kinetics
- Multi-gate and runner system design tools enabled
- Structural analysis of foamed parts limited to linear elastic; nonlinear foam material models gated

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Import part CAD and mold geometry, select foam formulation, and set processing conditions
- Run mold-fill simulation and identify filling pattern, last-fill areas, and potential defects
- Optimize gate locations and process parameters to eliminate predicted defects

**Key Workflows:**
- Simulate foam injection/pour and predict fill pattern, flow front, and air-trap locations
- Optimize gate location and size for uniform density distribution
- Predict knit-line locations and evaluate structural implications
- Analyze cure kinetics and demold time for cycle-time optimization
- Evaluate formulation changes (blowing-agent type/level, catalyst ratio) on part properties

---

### Senior/Expert
**Modified UI/UX:**
- Multi-physics simulation dashboard managing coupled flow-thermal-chemical-structural analyses
- Python scripting console with full solver API for custom reaction kinetics and cell-growth models
- Advanced visualization: cell-structure prediction, micro-to-macro property mapping, residual stress fields

**Feature Gating:**
- All features unlocked: micro-scale cell-growth modeling, reactive flow with full kinetics, mold deflection coupling
- Unlimited mesh size with adaptive refinement and parallel/GPU computation
- Plugin SDK for custom reaction schemes, blowing-agent models, and material property correlations

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Migration wizard importing existing Moldex3D/Moldflow project files and material databases
- HPC/GPU cluster configuration for large-scale reactive flow simulations
- Workshop on custom reaction kinetics calibration and cell-growth model development

**Key Workflows:**
- Full reactive flow simulation with coupled heat transfer, curing kinetics, and foam expansion
- Micro-scale cell nucleation and growth modeling predicting cell-size distribution and morphology
- Coupled mold-deflection simulation for predicting flash, warpage, and mold-clamping force
- Multi-objective process optimization balancing cycle time, part density uniformity, and surface quality
- Custom reaction kinetics development for novel foam formulations and blowing agents

---

## Role-Based Personas

### Designer/Creator
**Modified UI/UX:**
- Visual part-design canvas with wall-thickness analysis, rib/boss placement, and foam-friendly design-rule checking
- Gate and vent placement tool with instant fill-pattern preview
- Material-property mapping overlay showing local density, cell size, and mechanical properties on part geometry

**Feature Gating:**
- Part design evaluation tools (wall-thickness analysis, design-rule checking) fully enabled
- Simulation in rapid-preview mode for interactive design iteration; full-fidelity on demand
- Export to CAD formats with recommended design modifications annotated

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Import part CAD and run design-rule check for foam processability
- Place gates and vents using guided recommendations, preview fill pattern
- Iterate on part design and gate layout to achieve target quality and density

**Key Workflows:**
- Evaluate part designs for foam processability (wall thickness, flow length, rib geometry)
- Optimize gate and vent locations for uniform filling and minimum defects
- Predict local mechanical properties from density distribution for structural design
- Design inserts, reinforcements, and over-molding features for foam-metal/foam-fabric composites

---

### Analyst/Simulator
**Modified UI/UX:**
- Simulation-focused layout with mesh controls, solver settings, reaction monitoring, and convergence diagnostics
- Post-processing suite: flow-front tracking, temperature contours, cure maps, density distributions, warpage prediction
- Batch simulation manager for DOE studies and process-window mapping

**Feature Gating:**
- Full solver access with all physics modules, reaction kinetics, and cell-growth models
- Advanced calibration tools for matching simulation to experimental short-shot and density measurements
- API access for automated simulation campaigns and custom post-processing

**Pricing Tier:** Professional or Enterprise tier

**Onboarding Flow:**
- Import mold geometry and process data, run a fill simulation, and validate against short-shot trial data
- Calibrate foam expansion model against measured density profiles from sectioned parts
- Set up a DOE study varying key process parameters to map the process window

**Key Workflows:**
- High-fidelity reactive flow simulation with foam expansion and curing for complex parts
- Process-window mapping using design-of-experiments simulation campaigns
- Warpage and shrinkage prediction with residual stress analysis
- Simulation-test correlation using short-shot studies and density measurement data
- Cell-structure prediction and its influence on mechanical, thermal, and acoustic properties

---

### Manufacturing/Process
**Modified UI/UX:**
- Process-focused interface showing machine settings, cycle-time breakdown, and quality control parameters
- Integration panels for connecting to injection molding machine controllers and process monitoring systems
- Process recipe manager with version control and change-history tracking

**Feature Gating:**
- Simulation-derived process recipes and operating windows accessible
- Full simulation tools hidden; simplified process-adjustment calculators available
- Integration APIs for machine control systems and MES/quality databases

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Import simulation-optimized process recipe and map to machine control parameters
- Set up process monitoring with quality control limits derived from simulation
- Connect to machine controller for real-time process data logging

**Key Workflows:**
- Transfer simulation-optimized process parameters to production machine controllers
- Monitor process stability using simulation-derived control limits for key parameters
- Troubleshoot production defects (voids, surface defects, short shots) using calibrated process models
- Optimize cycle time by balancing cure time, cooling, and demold criteria

---

### Regulatory/Compliance
**Modified UI/UX:**
- Compliance dashboard for foam product regulations (fire safety, VOC emissions, food contact, automotive standards)
- Material compliance database tracking blowing agent regulations (HFC phase-down, ODP, GWP)
- Automated test-plan generator for regulatory certification testing (UL 94, FMVSS 302, EN 13501)

**Feature Gating:**
- Blowing-agent environmental compliance tracking (Montreal Protocol, F-gas regulation, EPA SNAP)
- Fire performance prediction tools linked to formulation and density data
- Certification documentation generator with regulatory-template formatting

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Select applicable regulations for the product/market (automotive, building, appliance, furniture)
- Import formulation data and run regulatory compliance assessment
- Generate compliance documentation and identify required certification tests

**Key Workflows:**
- Track blowing-agent compliance with environmental regulations (HFC phase-down schedules, GWP limits)
- Assess fire performance classification based on formulation, density, and additives
- Generate material safety and environmental compliance documentation
- Produce certification test plans for UL, EN, ASTM, or automotive OEM standards
- Monitor regulatory changes and flag affected products and formulations

---

### Manager/Decision-maker
**Modified UI/UX:**
- Product-line dashboard showing process efficiency, quality metrics, material costs, and sustainability KPIs
- Formulation comparison cards with cost, performance, and environmental-impact scoring
- Investment analysis tools for process equipment and formulation-change evaluations

**Feature Gating:**
- Read-only access to all simulation results and compliance reports
- Cost modeling and technology comparison tools enabled
- Approval workflow for formulation changes and capital expenditure decisions

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Dashboard overview: product-line quality, cycle-time efficiency, material costs, and sustainability scores
- Review formulation-change impact analysis tools
- Set up approval workflow for formulation and process modifications

**Key Workflows:**
- Evaluate formulation alternatives on cost, performance, processability, and environmental impact
- Assess ROI of process equipment upgrades (new machines, mold modifications, automation)
- Review blowing-agent transition plans (HFC to HFO/CO2) with cost and timeline projections
- Monitor production quality and scrap-rate trends across product lines
- Generate executive reports for sustainability initiatives and cost-reduction programs
