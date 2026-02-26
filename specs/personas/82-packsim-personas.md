# PackSim — Persona Subspecs

> Parent spec: [`specs/82-packsim.md`](../82-packsim.md)

---

## Experience-Level Personas

### Student/Academic
**Modified UI/UX:**
- Guided packaging-design wizard with pre-loaded product geometries (bottles, cans, boxes) and standard pallet sizes
- Annotated 3D packing visualization with labeled dimensions, center-of-gravity indicators, and load diagrams
- Sidebar glossary linking terms (compression strength, edge-crush test, cushion curves) to educational references

**Feature Gating:**
- Access to basic box/pallet optimization and simple drop-test simulation (single impact)
- Material library limited to common corrugated grades and standard EPS/EPE foams
- Export restricted to PDF reports and CSV; no direct ERP/WMS integration

**Pricing Tier:** Free tier

**Onboarding Flow:**
- Tutorial: design a corrugated shipping box for a sample product, optimize pallet arrangement, and simulate a single drop
- Pre-loaded case studies (consumer electronics packaging, food carton stacking)
- Quiz-style checkpoints reinforcing packaging fundamentals (McKee formula, cushion curve theory)

**Key Workflows:**
- Design a corrugated box to meet target burst/edge-crush strength for a given product weight
- Optimize pallet load pattern to maximize container utilization
- Simulate a single ISTA-style flat drop and review stress distribution
- Compare material options (single-wall vs. double-wall corrugated) on cost and protection

---

### Junior Engineer (0-3 years)
**Modified UI/UX:**
- Project-based workspace with tabs for Product Definition | Pack Design | Simulation | Compliance
- Integrated material property database with vendor-linked datasheets and cost estimators
- Step-by-step simulation setup with validation checks (mesh quality, boundary conditions)

**Feature Gating:**
- Full drop/vibration/compression simulation suite enabled (ISTA 2A, 3A, ASTM D4169)
- Cushion curve library and optimization tool unlocked
- Multi-drop sequence limited to 10 orientations; full random-vibration PSD locked behind upgrade

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Import a CAD product model, define fragility (G-level), and auto-generate cushion recommendations
- Run a guided ISTA 2A simulation sequence and review pass/fail report
- Connect to material supplier catalog for real-time pricing on cushion and corrugated options

**Key Workflows:**
- Design primary and secondary packaging to meet specific ISTA or ASTM test protocols
- Perform cushion curve analysis to select optimal foam type, thickness, and bearing area
- Simulate multi-drop sequences and identify critical failure modes
- Generate packaging specifications and drawings for supplier RFQ
- Evaluate sustainability metrics (recycled content, material reduction opportunities)

---

### Senior/Expert
**Modified UI/UX:**
- Multi-project dashboard with parametric study manager and batch simulation queue
- Scriptable automation via Python API for integrating packaging design into product-development pipelines
- Custom material model editor for proprietary foam/corrugated formulations

**Feature Gating:**
- All features unlocked: nonlinear FEA crash simulation, random vibration PSD, creep/humidity modeling
- Unlimited batch runs and parametric sweeps with cloud compute offload
- Plugin SDK for custom material models and proprietary test protocols

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Migration assistant for importing legacy TOPS/Cape Pack project files and material databases
- API key setup and CI/CD integration for automated packaging validation in PLM workflows
- Workshop on custom material model calibration using experimental test data

**Key Workflows:**
- Full nonlinear dynamic simulation of complex drop events with product-package interaction
- Parametric optimization across cushion geometry, material grade, and box dimensions simultaneously
- Humidity and creep analysis for long-duration warehousing and ocean freight scenarios
- Automated packaging validation triggered by product CAD changes in PLM system
- Multi-objective optimization balancing protection, sustainability, cost, and logistics efficiency

---

## Role-Based Personas

### Designer/Creator
**Modified UI/UX:**
- Visual 3D pack design canvas with drag-and-drop product placement and automatic void-fill suggestions
- Real-time pallet utilization gauge and truck-fill percentage as design changes are made
- Branding/graphics layer overlay for evaluating shelf appearance alongside structural design

**Feature Gating:**
- Full geometric design and pallet optimization tools enabled
- Simulation tools available in simplified mode (pass/fail output, not full stress fields)
- Export to dieline DXF/PDF for die-cutter programming and print-ready artwork

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Start by importing product geometry and defining packaging constraints (max weight, target cost)
- Drag-and-drop box style selection (RSC, FOL, tray) with auto-dimensioning
- Preview pallet load and container fill to validate logistics efficiency immediately

**Key Workflows:**
- Design retail-ready packaging with structural and graphic considerations simultaneously
- Optimize box dimensions to maximize pallet and container utilization
- Generate die-line drawings and send to packaging supplier
- Rapid iteration on packaging variants for different SKU sizes and bundling options

---

### Analyst/Simulator
**Modified UI/UX:**
- Simulation-centric layout with mesh controls, solver settings, and post-processing panels prominent
- Time-history and contour plot viewers with probe tools for extracting local stress/strain data
- Batch run manager with comparison overlays for multi-scenario evaluation

**Feature Gating:**
- Full FEA solver access: explicit dynamics for drop, implicit for compression/creep, frequency-domain for vibration
- Advanced meshing controls (adaptive, mapped, shell-solid coupling)
- API access for scripted parametric studies and automated report generation

**Pricing Tier:** Professional or Enterprise tier

**Onboarding Flow:**
- Import existing packaging CAD, set up a standard drop-test simulation, and validate against physical test data
- Guided mesh convergence study to build confidence in result accuracy
- Connect to test-data repository for simulation-vs-test correlation tracking

**Key Workflows:**
- High-fidelity drop simulation with full product-package interaction and material nonlinearity
- Random vibration simulation per ISTA 3A or ASTM D4728 with fatigue assessment
- Compression creep analysis for warehouse stacking under humidity and temperature cycling
- Simulation-test correlation studies to calibrate and validate material models
- Parametric sensitivity analysis to identify dominant design variables

---

### Manufacturing/Process
**Modified UI/UX:**
- Focused interface showing material specifications, die-line dimensions, and production parameters
- Integration panel for connecting to corrugator/converting machine settings databases
- Bill-of-materials summary view with supplier lead times and minimum order quantities

**Feature Gating:**
- Pack design output tools enabled (die-lines, BOMs, specifications)
- Simulation tools hidden; only pass/fail compliance status visible
- Integration APIs for ERP (SAP, Oracle) and MES systems enabled

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Import approved packaging design and generate manufacturing specifications
- Map die-line to corrugator/converting machine constraints (flute direction, max blank size)
- Link to ERP for material procurement and production scheduling

**Key Workflows:**
- Generate production-ready die-lines with machine-specific constraints applied
- Produce bill-of-materials with material grades, quantities, and supplier information
- Validate manufacturability against converting equipment capabilities
- Track material waste and suggest nesting optimizations for blank cutting

---

### Regulatory/Compliance
**Modified UI/UX:**
- Compliance dashboard with checklist views for ISTA, ASTM, UN, and IATA dangerous goods protocols
- Traffic-light indicators for each test requirement with linked simulation or test evidence
- Automated report generator producing certification-ready documentation

**Feature Gating:**
- All standard test protocol templates (ISTA 1–6 series, ASTM D4169, UN 4G/4GV) enabled
- Simulation results linked to compliance requirements with traceability
- Material regulatory database (REACH, FDA food contact, RoHS) accessible

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Select applicable standards and regulations for the product/market combination
- Import packaging design and auto-map to required test sequences
- Review compliance status and identify gaps requiring physical testing

**Key Workflows:**
- Validate packaging against ISTA certification requirements using simulation
- Generate UN dangerous-goods packaging certification documentation
- Track material compliance for food-contact, REACH, and RoHS requirements
- Produce audit-ready traceability reports linking design decisions to test evidence
- Monitor regulatory changes and flag affected packaging designs

---

### Manager/Decision-maker
**Modified UI/UX:**
- Portfolio dashboard showing all packaging projects with cost, timeline, and compliance status
- Cost comparison cards for packaging alternatives with logistics impact calculations
- ROI calculator for packaging changes (material savings vs. damage rate impact)

**Feature Gating:**
- Read-only access to all project results, reports, and dashboards
- Scenario comparison tools for evaluating packaging alternatives
- Budget tracking and approval workflow tools enabled

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Overview of dashboard KPIs (packaging cost per unit, damage rate, sustainability score)
- Connect to team workspace and review active packaging projects
- Walkthrough of approval workflow and cost-impact analysis tools

**Key Workflows:**
- Compare packaging alternatives on total landed cost (material + logistics + damage)
- Review and approve packaging designs at stage-gate milestones
- Track sustainability targets (material reduction, recycled content, plastic elimination)
- Monitor damage claim trends and correlate with packaging design changes
- Generate executive reports for supply chain optimization reviews
