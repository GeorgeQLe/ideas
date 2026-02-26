# HeatPipe — Persona Subspecs

> Parent spec: [`specs/88-heatpipe.md`](../88-heatpipe.md)

---

## Experience-Level Personas

### Student/Academic
**Modified UI/UX:**
- Guided heat-pipe builder with preset configurations (cylindrical, flat plate, loop, thermosyphon) and labeled cross-sections
- Interactive thermal-resistance network diagram showing evaporator, condenser, wick, and vapor-space contributions
- Animated visualization of two-phase cycle (evaporation, vapor transport, condensation, liquid return) with temperature/pressure overlays

**Feature Gating:**
- 1D steady-state heat-pipe performance calculator with standard working fluids (water, methanol, ammonia, sodium)
- Wick structure library limited to common types (sintered, mesh, grooved) with textbook correlations
- Transient simulation and 2D/3D analysis locked

**Pricing Tier:** Free tier

**Onboarding Flow:**
- Tutorial: design a copper-water heat pipe, compute operating limits (capillary, boiling, sonic, entrainment), and plot thermal resistance
- Pre-loaded examples (electronics cooling heat pipe, spacecraft radiator panel, sodium heat pipe for high-temp)
- References to Faghri, Reay & Kew textbook chapters mapped to each calculation module

**Key Workflows:**
- Calculate heat-pipe operating limits as a function of temperature for a given geometry and working fluid
- Evaluate thermal resistance breakdown across evaporator, condenser, wick, and vapor path
- Compare working fluid performance using figure-of-merit charts
- Size a heat pipe for a given heat load and temperature constraint

---

### Junior Engineer (0-3 years)
**Modified UI/UX:**
- Project workspace with tabs: Geometry | Materials/Fluids | Operating Conditions | Analysis | Reports
- Integrated material database with temperature-dependent properties for metals, wicks, and working fluids
- Design-space explorer showing feasible operating region on a power-vs-temperature map with limit overlays

**Feature Gating:**
- 1D transient analysis and quasi-2D (radial + axial) steady-state models enabled
- Custom wick property input with effective property calculators (permeability, capillary radius, thermal conductivity)
- System-level thermal network integration limited to 10 components; full system model locked

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Define a heat-pipe design from application requirements (heat load, temperature range, orientation, length)
- Run parametric sweep over wick thickness and porosity to optimize thermal performance
- Generate a design datasheet with operating limits, thermal resistance, and mass estimates

**Key Workflows:**
- Design heat pipes and vapor chambers for electronics thermal management applications
- Perform parametric optimization of geometry and wick parameters for target thermal performance
- Evaluate heat-pipe performance under varying orientation and acceleration (gravity, vibration)
- Integrate heat-pipe thermal resistance into system-level thermal models (resistance networks)
- Generate design specification documents for prototype procurement

---

### Senior/Expert
**Modified UI/UX:**
- Multi-physics simulation dashboard managing coupled thermal-fluid-structural analyses
- Python scripting console with full solver API for custom wick models, evaporation correlations, and automation
- Advanced visualization: 2D/3D temperature and liquid-saturation fields with phase-interface tracking

**Feature Gating:**
- All features unlocked: full 3D CFD-based two-phase simulation, pore-scale wick modeling, structural-thermal coupling
- Transient startup/shutdown and freeze-thaw analysis for extreme environments
- Plugin SDK for custom evaporation/condensation models and wick microstructure import

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Import existing thermal models (COMSOL, Thermal Desktop) via migration tools
- Configure HPC resources for 3D two-phase CFD simulations
- Workshop on custom evaporation model development and wick microstructure characterization integration

**Key Workflows:**
- Full 3D two-phase CFD simulation of heat-pipe internal flow with phase-change modeling
- Transient startup analysis including frozen-startup scenarios for spacecraft applications
- Coupled thermal-structural analysis for heat-pipe integration into load-bearing structures
- Pore-scale wick modeling using micro-CT imported structures for accurate capillary prediction
- Multi-objective optimization of heat-pipe design balancing mass, performance, and reliability

---

## Role-Based Personas

### Designer/Creator
**Modified UI/UX:**
- Visual cross-section editor with drag-to-resize wick, wall, and vapor-space dimensions
- System integration canvas for placing heat pipes within assembly context (electronics enclosure, satellite panel)
- Real-time thermal resistance gauge updating as geometry and materials are adjusted

**Feature Gating:**
- Geometry design and rapid performance estimation tools fully enabled
- Detailed simulation available for design validation after rapid-design convergence
- Export to CAD formats (STEP, IGES) and thermal model interchange formats

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Define application envelope (heat load, temperature range, available space, orientation)
- Use auto-sizing wizard to generate candidate heat-pipe designs meeting constraints
- Refine selected design and export geometry for mechanical integration

**Key Workflows:**
- Rapidly size heat pipes and vapor chambers for given thermal and spatial constraints
- Explore design trade-offs between geometry, working fluid, and wick structure
- Design custom vapor chamber geometries for non-uniform heat source distributions
- Export heat-pipe models for integration into system-level thermal and mechanical analyses

---

### Analyst/Simulator
**Modified UI/UX:**
- Simulation-centric layout with solver controls, convergence monitoring, and field visualization panels
- Validation workspace with experimental data overlay and goodness-of-fit metrics
- Batch simulation manager for parametric studies and Monte Carlo uncertainty analysis

**Feature Gating:**
- Full solver suite (1D, 2D, 3D) with all physics modules and boundary conditions
- Model calibration tools with experimental data correlation
- API access for automated analysis pipelines and custom post-processing

**Pricing Tier:** Professional or Enterprise tier

**Onboarding Flow:**
- Set up a reference heat-pipe model and validate against published experimental data
- Perform mesh/grid convergence study and document model accuracy
- Run parametric study on critical design variables and generate sensitivity charts

**Key Workflows:**
- High-fidelity simulation of heat-pipe two-phase flow and heat transfer
- Model validation against experimental data with uncertainty quantification
- Transient thermal analysis for power cycling and duty-cycle evaluation
- Parametric studies exploring design variable impacts on performance limits
- Coupled system simulation (heat pipe + heat source + heat sink + environment)

---

### Manufacturing/Process
**Modified UI/UX:**
- Fabrication-focused interface showing manufacturing specifications, tolerances, and process parameters
- Bill-of-materials with material grades, quantities, and sourcing specifications
- Quality control checklists for critical manufacturing steps (charging, sealing, testing)

**Feature Gating:**
- Design specification and BOM generation tools enabled
- Simulation tools hidden; only design-output specifications visible
- Integration with manufacturing tracking systems and quality databases

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Import approved heat-pipe design and generate manufacturing specification package
- Review fabrication process steps with quality-control checkpoints
- Link to QC database for tracking production lot performance

**Key Workflows:**
- Generate detailed manufacturing specifications (dimensions, tolerances, material grades, fluid charge)
- Produce process routing documents for tube forming, wick insertion, charging, and sealing
- Define acceptance test criteria (tilt test, thermal resistance measurement, leak check)
- Track production lot data and correlate manufacturing variations with thermal performance

---

### Regulatory/Compliance
**Modified UI/UX:**
- Compliance dashboard for thermal device regulations (pressure vessel codes, transport, material restrictions)
- Checklist views for applicable standards (ASME BPVC Section VIII, DOT, IATA dangerous goods for working fluids)
- Automated documentation generator for certification packages

**Feature Gating:**
- Pressure rating and stress analysis tools per applicable codes enabled
- Working fluid hazard classification and transport regulation database accessible
- Material compliance tracking (RoHS, REACH) for working fluids and construction materials

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Select applicable regulatory frameworks (pressure vessel, transport, environmental)
- Import heat-pipe design and run compliance assessment against selected codes
- Generate compliance documentation package and identify items requiring third-party certification

**Key Workflows:**
- Verify pressure containment design against ASME or EN pressure vessel standards
- Classify working fluid for transport regulation compliance (IATA, DOT, ADR)
- Track material compliance with RoHS, REACH, and customer-specific restricted substance lists
- Generate qualification test plans meeting military/space standards (MIL-STD, ECSS)
- Maintain compliance documentation for product certification and customer audits

---

### Manager/Decision-maker
**Modified UI/UX:**
- Product portfolio dashboard showing thermal solution performance, cost, and reliability across product lines
- Technology comparison cards: heat pipe vs. vapor chamber vs. cold plate vs. active cooling alternatives
- Make-vs-buy analysis tool with supplier capability and cost databases

**Feature Gating:**
- Read-only access to all design and test results
- Cost modeling and technology comparison tools enabled
- Approval workflow for design releases and supplier selection

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Dashboard overview: product line thermal solutions, cost trends, and reliability data
- Review technology comparison tools for evaluating cooling alternatives
- Set up approval workflow for design milestones and supplier qualifications

**Key Workflows:**
- Compare thermal management technology alternatives on performance, cost, mass, and reliability
- Evaluate make-vs-buy decisions for heat-pipe production
- Review and approve thermal solution designs at product development milestones
- Monitor field reliability data and warranty claims related to thermal management
- Plan technology roadmap for next-generation thermal management solutions
