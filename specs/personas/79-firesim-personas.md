# FireSim — Persona Subspecs

> Parent spec: [`specs/79-firesim.md`](../79-firesim.md)

---

## Experience-Level Personas

### Student/Academic
**Modified UI/UX:**
- Interactive fire compartment visualizer with annotated heat release rate curves, plume dynamics, and layer heights
- Simplified fire scenario builder with preset fuel packages (furniture, pool fire, crib) and room templates
- Educational mode explaining fire dynamics concepts (flashover, ventilation control, t-squared growth)

**Feature Gating:**
- Zone model fire simulation (two-layer) for single and multi-compartment scenarios unlocked
- Basic CFD fire simulation limited to small domains and coarse grids
- Smoke detector activation, sprinkler response, and tenability analysis available; complex suppression modeling locked

**Pricing Tier:** Free tier (academic license)

**Onboarding Flow:**
- Tutorial: simulate a compartment fire using a zone model and predict upper layer temperature and smoke height
- Pre-loaded examples: room fire with flashover, corridor smoke filling, simple atrium smoke management
- Guided comparison of zone model predictions against McCaffrey/Heskestad plume correlations

**Key Workflows:**
- Simulating single-compartment fires and predicting time to flashover
- Calculating smoke layer descent rates in corridors and open spaces
- Estimating sprinkler and detector activation times for simple fire scenarios
- Generating temperature-time curves and heat release rate plots for fire engineering coursework

### Junior Engineer (0-3 years)
**Modified UI/UX:**
- Guided scenario setup wizard: building geometry, fuel loads, ventilation, detection/suppression systems
- Pre-built fire scenario templates aligned with performance-based design guidelines (ICC, BS 7974, IFEG)
- Tenability criteria dashboard showing ASET vs. RSET calculations with color-coded margin indicators

**Feature Gating:**
- Full zone model simulation and moderate-resolution CFD fire simulation
- Smoke management system evaluation (mechanical exhaust, natural venting, pressurization)
- Egress time estimation with simple agent-based modeling; advanced evacuation simulation locked

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Role-based setup: performance-based design, fire investigation, smoke management, or code consulting
- Walkthrough: model a design fire scenario in a commercial building, assess tenability, compare to code
- Introduction to documentation standards for performance-based design reports

**Key Workflows:**
- Developing design fire scenarios for performance-based fire engineering assessments
- Running CFD smoke simulations for atrium and large-space smoke management design
- Calculating ASET (available safe egress time) and comparing to RSET for life safety
- Evaluating smoke control system effectiveness (exhaust rates, makeup air, pressurization)
- Generating performance-based design reports with code-compliant documentation

### Senior/Expert
**Modified UI/UX:**
- Multi-physics fire simulation console coupling fire dynamics, structural response, and evacuation modeling
- Custom combustion model editor for specialized fuels (wildland, industrial chemicals, battery thermal runaway)
- Scripting interface for automated parametric fire studies and probabilistic risk assessment

**Feature Gating:**
- All modules: LES fire simulation, pyrolysis modeling, structural fire response, complex suppression, wildfire
- Full API for coupling with FEA structural codes, evacuation models, and BIM platforms
- Admin controls for proprietary fire test databases, custom material models, and company standards

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Migration assistant for importing FDS/PyroSim models and legacy fire engineering project databases
- Configuration of custom material pyrolysis models and combustion chemistry
- Setup of automated simulation validation pipelines against fire test data

**Key Workflows:**
- Running large-scale LES fire simulations for complex building geometries and tunnel fires
- Modeling fire growth with detailed pyrolysis and combustion chemistry for novel materials
- Coupling fire simulation with structural response to predict fire-induced collapse mechanisms
- Performing probabilistic fire risk assessments with Monte Carlo sampling across scenarios
- Developing and validating fire models against large-scale fire test data

---

## Role-Based Personas

### Designer/Creator
**Modified UI/UX:**
- Smoke management system designer with exhaust fan placement, natural vent sizing, and duct routing
- Fire-resistant assembly configurator for walls, floors, doors, and penetration seals
- Sprinkler system layout tool with coverage area visualization and hydraulic calculation integration

**Feature Gating:**
- Full access to smoke management design tools and fire protection system layout
- Fire resistance assembly database with tested configurations (UL, FM, Intertek)
- Export to MEP engineering formats for fire protection shop drawing generation

**Pricing Tier:** Professional or Enterprise tier

**Onboarding Flow:**
- Quick-start: size a smoke exhaust system for an atrium using steady-state fire scenario
- Tutorial on integrating fire simulation results into smoke management system design
- Walkthrough of sprinkler system layout and performance verification workflow

**Key Workflows:**
- Designing smoke management systems (mechanical exhaust, natural venting, pressurization) using simulation
- Specifying fire-rated barriers and penetration seals based on fire severity analysis
- Designing sprinkler systems and verifying performance against design fire scenarios
- Laying out fire detection systems with coverage analysis and activation time estimation
- Creating fire protection engineering design packages for construction

### Analyst/Simulator
**Modified UI/UX:**
- CFD fire simulation workflow with mesh resolution guidance, boundary conditions, and solver monitoring
- Visibility, temperature, and toxicity (FED) isosurface and slice visualization tools
- Automated ASET calculation from simulation outputs with tenability criteria tracking

**Feature Gating:**
- Full CFD solver access (FDS-compatible solver with LES turbulence modeling)
- Advanced post-processing: FED/FEC calculations, visibility contours, thermocouple tree comparisons
- HPC integration for multi-scenario parametric studies and fine-mesh simulations

**Pricing Tier:** Professional or Enterprise tier

**Onboarding Flow:**
- Guided first simulation: single-compartment fire with comparison to zone model and experimental data
- Setup of mesh sensitivity study template and convergence verification procedure
- Introduction to batch scenario management for design fire families

**Key Workflows:**
- Running CFD fire simulations for performance-based design fire scenarios
- Computing tenability conditions (visibility, temperature, toxicity) along egress paths
- Simulating smoke detector and sprinkler activation and their effect on fire development
- Performing mesh sensitivity studies to verify simulation result reliability
- Comparing simulation predictions against fire test measurements for model validation

### Manufacturing/Process
**Modified UI/UX:**
- Fire system commissioning checklist generator for detection, suppression, and smoke management systems
- Inspection and testing scheduler for fire protection equipment maintenance
- Fire protection system performance verification dashboard with acceptance test results

**Feature Gating:**
- Access to commissioning planning and system acceptance testing tools
- Fire system inspection and maintenance scheduling with regulatory interval tracking
- Integration with building management systems for fire system status monitoring

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Configure building fire protection system inventory (detection, suppression, smoke control, structural fire proofing)
- Walkthrough: generate a commissioning plan for a smoke management system from the design model
- Tutorial on fire protection system acceptance testing and documentation

**Key Workflows:**
- Generating commissioning and acceptance testing plans for fire protection systems
- Verifying smoke management system performance through hot smoke testing simulation
- Planning fire protection inspection, testing, and maintenance schedules per NFPA 25
- Documenting fire protection system configurations and as-built records
- Tracking fire protection impairments and ensuring compensating measures are in place

### Regulatory/Compliance
**Modified UI/UX:**
- Code compliance dashboard mapping fire engineering design to prescriptive code requirements (IBC, BS 9999, EN)
- Performance-based design review checklist aligned with ICC/SFPE performance-based design guide
- Fire safety strategy document generator with required code references and justifications

**Feature Gating:**
- Read-only simulation access; full compliance checking and documentation tools
- Automated prescriptive code compliance checkers (travel distance, exit capacity, fire ratings)
- Peer review management workflow for performance-based design submissions

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Configuration of applicable codes and standards (IBC, NFPA, BS, EN, local amendments)
- Setup of performance-based design review templates and approval workflows
- Walkthrough: review a performance-based fire engineering report against ICC/SFPE guidelines

**Key Workflows:**
- Reviewing performance-based fire engineering designs for code compliance and technical adequacy
- Verifying fire simulation inputs, assumptions, and results against accepted engineering practices
- Generating fire safety strategy documents with prescriptive and performance-based justifications
- Tracking fire code variance requests and their resolution through approval
- Maintaining fire safety documentation for building occupancy permits and periodic inspections

### Manager/Decision-maker
**Modified UI/UX:**
- Building portfolio fire risk dashboard with risk rankings, code compliance status, and improvement priorities
- Cost-benefit analysis comparing fire protection options against risk reduction and code compliance
- Insurance impact estimator showing fire protection improvements vs. premium reduction potential

**Feature Gating:**
- Dashboard and reporting access; no direct simulation or design tools
- Risk ranking and investment prioritization views across building portfolios
- Integration with insurance risk assessment and property loss control data

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Configuration of building portfolio with occupancy types, fire protection features, and loss histories
- Integration with property management, insurance, and capital planning systems
- Setup of fire risk review cadence and management reporting distribution

**Key Workflows:**
- Prioritizing fire protection upgrades across a building portfolio based on risk and cost-benefit
- Evaluating fire engineering design options for new construction and renovation projects
- Reviewing fire risk assessment results and approving mitigation investment plans
- Communicating fire safety status and residual risk to executive leadership and insurers
- Tracking return on fire protection investment through loss experience and insurance premium analysis
