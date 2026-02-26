# PipeFlow — Persona Subspecs

> Parent spec: [`specs/52-pipeflow.md`](../52-pipeflow.md)

---

## Experience-Level Personas

### Student/Academic

**Modified UI/UX:**
- Simplified single-line diagram editor with drag-and-drop components (pipe, pump, valve, tank) and auto-routing
- Inline Moody chart and Bernoulli equation references that highlight active terms during solve
- Results panel defaults to pressure drop, velocity, and Reynolds number with colour-coded flow regime indicators

**Feature Gating:**
- Network size capped at 50 pipes; single-phase incompressible flow only
- Component library limited to standard fittings (elbows, tees, reducers); no custom loss coefficient editor
- Export to PDF and CSV only; no CAESAR II or Plant 3D interop

**Pricing Tier:** Free tier (educational license)

**Onboarding Flow:**
- "First Network" tutorial: build a simple pumped-loop system, size the pump, and check velocity limits
- Interactive friction-factor explorer comparing Darcy-Weisbach and Hazen-Williams for the same pipe
- Sandbox with pre-built textbook problems (series, parallel, branching networks)

**Key Workflows:**
- Solve a branching pipe network and verify against manual Hardy Cross calculations
- Size a centrifugal pump for a given system curve
- Evaluate pressure drop across a valve train for a laboratory assignment
- Compare friction correlations and pipe roughness effects on head loss

---

### Junior Engineer (0-3 years)

**Modified UI/UX:**
- Standard P&ID-style workspace with a project checklist sidebar tracking deliverables (line list, valve schedule, pump datasheet)
- Guided pipe sizing assistant that suggests schedule, material, and velocity limits per ASME B31.3 or B31.1
- Warning badges on components operating outside recommended ranges (velocity, pressure, temperature)

**Feature Gating:**
- Network size up to 500 pipes; single-phase compressible and incompressible flow
- Standard component library plus vendor pump curves import; no two-phase or slurry modules
- Import from AutoCAD Plant 3D and SmartPlant; export to isometric drawing generators

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Project-type selector (oil & gas, water/wastewater, HVAC, chemical process) pre-loads code defaults and unit systems
- Walkthrough of a cooling water system design from schematic through hydraulic analysis to pump selection
- Prompted to import firm's pipe specs, insulation standards, and preferred vendor catalogues

**Key Workflows:**
- Design a cooling water distribution system and select pumps to meet flow/head requirements
- Perform hydraulic analysis of a firewater ring main to verify residual pressure at hydrants
- Generate a line list with pipe sizes, schedules, materials, and insulation specs
- Evaluate control valve Cv sizing and check for cavitation or flashing conditions
- Produce client-deliverable hydraulic calculation reports per project standards

---

### Senior/Expert

**Modified UI/UX:**
- Multi-model workspace with tabbed scenarios, scripting console (Python API), and batch-run manager for parametric studies
- Custom component modeller for non-standard equipment (ejectors, static mixers, custom heat exchangers)
- Real-time dashboard linking live plant DCS/SCADA data to model for digital-twin operation

**Feature Gating:**
- Unlimited network size; two-phase flow, slurry, non-Newtonian fluids, and surge/waterhammer analysis
- Full API for automation, custom solver plugins, and integration with process simulators (Aspen, HYSYS)
- Stress interface to CAESAR II and AutoPIPE; thermal-hydraulic coupling

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Import existing AFT Fathom / Pipenet / FluidFlow models with automated translation and validation
- Configure solver preferences, convergence criteria, and HPC/cloud compute connections
- Power-user preset available to skip tutorials and open scripting workspace directly

**Key Workflows:**
- Perform transient surge analysis (waterhammer) for a long-distance pipeline with valve closure scenarios
- Model two-phase flow through a production facility from wellhead to separator
- Build a plant-wide digital twin linking hydraulic model to live instrument data for operator decision support
- Develop custom correlations for proprietary fluids and integrate via solver API
- Coordinate piping hydraulics with stress analysis and process simulation in a multidisciplinary engineering workflow

---

## Role-Based Personas

### Designer/Creator

**Modified UI/UX:**
- Visual piping layout canvas with isometric and plan views; snap-to-grid routing with automatic support placement
- Material and specification browser with intelligent suggestions based on fluid service and design conditions
- 3D preview panel showing routed piping with colour-coded flow directions and sizing annotations

**Feature Gating:**
- Full geometry editing, routing, and pipe specification tools enabled
- Hydraulic solver runs in background during design for real-time feedback on velocity and pressure drop
- Export to isometric drawings, MTO (material take-off), and BIM models (Revit MEP, Navisworks)

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Import P&ID or 3D plant model as starting point; auto-generate hydraulic schematic from connectivity
- "Design for Constructability" checklist guiding routing decisions (clearances, support spans, drain points)
- Tour of specification management: create pipe classes, assign to lines, enforce material consistency

**Key Workflows:**
- Route piping from a P&ID to a 3D layout while maintaining hydraulic feasibility
- Select pipe sizes and schedules balancing velocity limits, pressure ratings, and material cost
- Generate material take-offs and cost estimates for bid comparison
- Coordinate piping layout with structural steel, equipment, and other disciplines in a shared 3D model
- Produce fabrication isometrics with weld maps and bill of materials

---

### Analyst/Simulator

**Modified UI/UX:**
- Data-centric workspace with convergence plots, sensitivity spider charts, and parameter sweep tables
- Multi-scenario comparison panel overlaying pressure profiles, flow distributions, and pump operating points
- Script editor with hydraulic solver API access and post-processing library

**Feature Gating:**
- All solver modules enabled: steady-state, transient, two-phase, thermal, and coupled simulations
- Monte Carlo and probabilistic analysis tools for uncertainty quantification on pipe roughness, demands, and fluid properties
- HPC job scheduling for large-network transient analyses

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Benchmark suite running NIST/industry-standard validation cases to confirm solver accuracy
- Import existing analysis models and calibration datasets
- Configure default convergence tolerances, under-relaxation factors, and reporting thresholds

**Key Workflows:**
- Perform surge analysis for a water transmission main evaluating protective devices (surge tanks, air valves, PRVs)
- Conduct thermal-hydraulic analysis of a steam distribution network with condensate return
- Run Monte Carlo simulation on pipe roughness aging to predict system performance at end-of-life
- Validate hydraulic model against field commissioning data and calibrate loss coefficients
- Produce technical memoranda with sensitivity analyses for design review boards

---

### Manufacturing/Process

**Modified UI/UX:**
- Equipment-centric view organised around process units (reactors, columns, heat exchangers) with piping interconnections
- Utility tracking dashboard showing steam, cooling water, and instrument air distribution and consumption
- Integration panel for connecting to plant historian and maintenance management system (CMMS)

**Feature Gating:**
- Process simulation coupling (Aspen Plus, HYSYS) for boundary condition import
- Fouling and degradation modelling for heat exchangers and pipe internals
- Instrument sizing tools (orifice plates, flow meters, control valves) with rangeability checks

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Import process flow diagram and heat/mass balance to auto-generate piping hydraulic model
- Guided setup of operating cases: normal, turndown, startup, shutdown, and emergency scenarios
- Connect to plant DCS historian for model validation against real operating data

**Key Workflows:**
- Size utility piping (steam, condensate, cooling water) for a new process unit
- Evaluate impact of production rate changes on existing piping capacity and pump operation
- Design control valve installations with proper upstream/downstream straight lengths and check valve authority
- Analyse fouled-condition performance of cooling water systems to determine cleaning schedules
- Generate piping specifications aligned with process requirements and material compatibility

---

### Regulatory/Compliance

**Modified UI/UX:**
- Code-compliance dashboard organised by applicable standards (ASME B31.1/B31.3, API 570, EN 13480)
- Pressure-temperature rating checker with automatic verification against ASME B16.5 flange classes
- Audit trail capturing every design decision, code reference, and approval with timestamps

**Feature Gating:**
- Code-checking engine fully enabled with built-in rules for major piping codes
- Hydrostatic test pressure calculator and leak-test procedure generator
- Material traceability and positive material identification (PMI) tracking

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Jurisdiction and code-edition selector pre-loads applicable design rules and allowable stress tables
- Walkthrough of a pressure piping code compliance check per ASME B31.3
- Configure firm-specific documentation templates with PE stamp blocks and revision control

**Key Workflows:**
- Verify piping design pressures and temperatures against code-allowable stresses for all lines
- Produce ASME B31.3 design calculation packages for regulatory submission
- Evaluate remaining life of corroded piping per API 570 / API 574 fitness-for-service methods
- Document relief valve sizing calculations per API 520/521 with all scenario justifications
- Generate inspection and hydrostatic test plans for new piping installations

---

### Manager/Decision-maker

**Modified UI/UX:**
- Portfolio dashboard showing all active piping projects with schedule, budget, and compliance status
- Cost comparison charts for alternative pipe materials, routing options, and pump selections
- Resource allocation view showing engineering hours by project phase and discipline

**Feature Gating:**
- Read-only access to hydraulic models and results; cannot modify network or solver parameters
- Full access to cost estimation, project tracking, and reporting modules
- License utilisation and team productivity analytics

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Connect to corporate project management and ERP systems for schedule and cost data
- Dashboard configuration selecting business KPIs (project margin, change order rate, schedule adherence)
- Demo of a capital cost comparison between two piping material alternatives

**Key Workflows:**
- Compare capital and lifecycle costs of carbon steel vs. duplex stainless vs. FRP piping alternatives
- Review hydraulic feasibility studies for early-phase project screening and go/no-go decisions
- Track engineering deliverable progress across multiple piping design projects
- Evaluate pump energy costs and identify VFD retrofit opportunities for operational savings
- Present project status and technical risks to client or executive leadership
