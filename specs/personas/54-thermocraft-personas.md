# ThermoCraft — Persona Subspecs

> Parent spec: [`specs/54-thermocraft.md`](../54-thermocraft.md)

---

## Experience-Level Personas

### Student/Academic

**Modified UI/UX:**
- Simplified 2D cross-section editor with drag-and-drop thermal components (heat source, heat sink, fan, TIM, PCB)
- Inline thermal resistance network diagram that updates as the user builds the geometry
- Results panel defaults to temperature contours, heat flux arrows, and junction-to-ambient thermal resistance

**Feature Gating:**
- Geometry limited to 2D cross-sections and simplified 3D block models; max 20 components
- Conduction and natural convection only; forced convection and radiation solvers locked
- Export to PDF and CSV; no ECXML or 6SigmaET interop

**Pricing Tier:** Free tier (educational license)

**Onboarding Flow:**
- "First Thermal Model" tutorial: build a simple IC-on-PCB model, apply boundary conditions, solve, and interpret Tj
- Interactive thermal resistance explorer showing R_jc, R_cs, R_sa for a TO-220 package
- Sandbox with pre-built textbook cases (fin arrays, heat pipes, thermoelectric coolers)

**Key Workflows:**
- Model a heatsink on a power transistor and predict junction temperature
- Compare natural vs. forced convection cooling for a simple PCB layout
- Calculate thermal resistance of a multi-layer PCB stackup
- Validate simulation results against analytical 1D conduction solutions

---

### Junior Engineer (0-3 years)

**Modified UI/UX:**
- Standard 3D workspace with component placement wizard; auto-meshing with quality indicators
- Guided boundary condition setup (ambient temperature, airflow, thermal interface materials) with recommended defaults
- Warning badges on components exceeding manufacturer's rated junction temperature (Tj_max)

**Feature Gating:**
- Full 3D solver with conduction, convection, and radiation; mesh up to 2M cells
- Standard component library (chip packages, heatsinks, fans, TIMs) with thermal resistance datasheets
- Import from ECAD (Allegro, Altium) and MCAD (SolidWorks, Creo); export thermal reports

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Project-type selector (electronics enclosure, LED luminaire, power converter, data centre rack) pre-loads templates
- Walkthrough of a switch-mode power supply thermal analysis from ECAD import to component temperature prediction
- Prompted to import firm's component thermal models and preferred heatsink vendor catalogues

**Key Workflows:**
- Import a PCB layout from Altium and predict component temperatures under worst-case ambient
- Select and size a heatsink-fan combination to keep an FPGA within its thermal design power envelope
- Evaluate thermal interface material options and their impact on junction-to-case resistance
- Perform thermal derating analysis to determine maximum ambient operating temperature
- Generate a thermal analysis report for design review with temperature maps and margin tables

---

### Senior/Expert

**Modified UI/UX:**
- Multi-physics workspace with conjugate heat transfer (CHT), transient solvers, and Joule heating coupling
- Scripting console (Python API) for parametric sweeps, DOE, and automated optimisation of fin geometry or airflow paths
- Digital-twin dashboard linking live thermocouple and IR camera data to model for real-time correlation

**Feature Gating:**
- Unlimited mesh size; CHT with turbulence models (k-e, SST); transient and cyclic duty-cycle analysis
- Compact thermal model (CTM/DELPHI) generation and BCI-ROM export for system-level simulation
- Full API, HPC cluster support, and integration with CFD (Fluent, StarCCM+), FEA, and EDA tools

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Import existing FloTHERM/Icepak/6SigmaET models with automatic translation and validation
- Configure solver preferences, turbulence model defaults, and compute cluster connections
- Power-user preset skipping tutorials, opening scripted parametric study workspace

**Key Workflows:**
- Perform conjugate heat transfer analysis of a liquid-cooled cold plate with microchannel geometry optimisation
- Build a transient thermal model of a power module under mission-profile duty cycles (automotive WLTP)
- Generate DELPHI-compliant compact thermal models for IC packages to share with system integrators
- Couple thermal simulation with structural FEA to predict thermomechanical stress and solder fatigue life
- Develop a digital twin linking thermal model to field sensor data for predictive maintenance of power electronics

---

## Role-Based Personas

### Designer/Creator

**Modified UI/UX:**
- Visual 3D layout editor with real-time temperature preview as components are placed and heatsinks resized
- Heatsink design studio with parametric fin generators (straight, pin, radial) and airflow direction indicators
- Enclosure design tools with automated vent placement suggestions based on buoyancy-driven flow patterns

**Feature Gating:**
- Full geometry creation and editing tools; parametric heatsink and cold-plate generators enabled
- Thermal solver runs in background for real-time design feedback; detailed CHT requires explicit launch
- Export to MCAD (STEP, Parasolid) and 3D printing formats for rapid prototyping of custom heatsinks

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Start from ECAD/MCAD import or blank enclosure template; auto-detect hot components and suggest cooling approach
- "Cooling Strategy" wizard: select thermal budget and form factor constraints, receive heatsink/fan/TIM recommendations
- Tour of parametric fin optimiser for a sample LED heatsink design

**Key Workflows:**
- Design a custom heatsink geometry for a space-constrained embedded computer
- Optimise enclosure vent locations and sizes to maximise natural convection chimney effect
- Evaluate cold-plate channel layouts for liquid-cooled server trays
- Create a thermal mockup of a consumer electronics device (phone, laptop) for early feasibility assessment
- Export finalised heatsink geometry to CNC or additive manufacturing workflows

---

### Analyst/Simulator

**Modified UI/UX:**
- Data-dense workspace with convergence monitors, mesh sensitivity study tools, and multi-scenario comparison panels
- Turbulence model selection assistant with Y+ monitor and wall treatment recommendations
- Post-processing suite with streamline visualisation, Nusselt number maps, and thermal resistance network extraction

**Feature Gating:**
- All solvers unlocked: CHT, radiation (S2S, DO), transient, and phase-change (melting/solidification for PCM)
- Monte Carlo and DOE tools for parametric sensitivity and uncertainty quantification on material properties
- HPC job scheduling with adaptive mesh refinement and solution-adaptive convergence

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Validation suite running standard benchmarks (heated cavity, impinging jet, electronics cooling round-robin)
- Import reference models and experimental data for correlation setup
- Configure default mesh strategies, solver settings, and post-processing templates

**Key Workflows:**
- Perform mesh independence study and validate CHT model against experimental IR thermography data
- Run DOE on TIM thickness, fan speed, and ambient temperature to map thermal margin sensitivity
- Analyse transient cool-down and warm-up profiles for a satellite payload in orbital eclipse/sunlight cycles
- Model phase-change material (PCM) integration for passive thermal buffering of intermittent heat loads
- Produce detailed technical reports with uncertainty bands for thermal design review boards

---

### Manufacturing/Process

**Modified UI/UX:**
- Process-centric view organised by cooling solution manufacturing method (extrusion, die-cast, skived, vapour chamber)
- Material and process parameter database with thermal conductivity, cost, and manufacturability ratings
- Quality metrics dashboard tracking thermal interface bond-line thickness, void fraction, and consistency

**Feature Gating:**
- Manufacturing process simulation (casting fill, extrusion flow, brazing joint) interface enabled
- Thermal test correlation tools for comparing production samples against design predictions
- Batch analysis for evaluating manufacturing variability impact on thermal performance

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Product line setup: import heatsink catalogue, TIM specifications, and production thermal test data
- Guided tutorial on correlating a heatsink thermal resistance measurement to simulation prediction
- Connect to thermal test equipment (wind tunnel, JEDEC test fixtures) for automated data import

**Key Workflows:**
- Evaluate manufacturability trade-offs between skived-fin and bonded-fin heatsink architectures
- Predict impact of TIM bond-line thickness variation on junction temperature across production lots
- Simulate vapour chamber performance with manufacturing-induced wick porosity variations
- Correlate production thermal test results to simulation and adjust model calibration factors
- Generate thermal performance datasheets for heatsink and cold-plate product lines

---

### Regulatory/Compliance

**Modified UI/UX:**
- Standards compliance dashboard covering safety limits (IEC 62368-1 touch temperature, UL 60950) and environmental (RoHS, REACH)
- Touch-temperature calculator with exposure time and material correction per IEC 62368-1 Table 42
- Documentation control panel with thermal test evidence, certification references, and audit trails

**Feature Gating:**
- Safety limit checking engine fully enabled with built-in IEC/UL temperature thresholds
- Environmental compliance tracking for thermal materials (RoHS lead-free solder, REACH SVHC in TIMs)
- Automated generation of thermal safety test plans per applicable product safety standards

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Product-safety standard selector (IEC 62368-1, UL 60950, IEC 60601 for medical, IEC 61215 for solar)
- Walkthrough of a touch-temperature compliance assessment for a consumer electronics enclosure
- Configure pass/fail criteria for firm's product families and target markets

**Key Workflows:**
- Verify enclosure surface temperatures comply with IEC 62368-1 accessible surface limits
- Document thermal runaway analysis for lithium battery products per UL 2054 / IEC 62133
- Assess thermal derating compliance per component manufacturer specifications under worst-case conditions
- Generate thermal safety test plans and expected results for certification lab submissions
- Maintain thermal compliance records across product variants and regulatory jurisdictions

---

### Manager/Decision-maker

**Modified UI/UX:**
- Portfolio dashboard showing all product thermal designs with risk status, cost of cooling solution, and compliance readiness
- Cost-performance trade-off charts comparing cooling technologies (passive, fan, liquid, TEC) for each product
- Warranty and field-failure correlation panel linking thermal margin to return rates

**Feature Gating:**
- Read-only access to thermal models and simulation results; cannot modify geometry or solver settings
- Full access to cost estimation, project tracking, and reporting modules
- Product reliability analytics linking thermal performance to warranty data

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Product portfolio import linking to PLM/ERP systems for BOM cost and field data
- Dashboard configuration selecting KPIs (thermal margin, cooling cost per watt, field failure rate)
- Demo of ROI analysis comparing active vs. passive cooling for a product family

**Key Workflows:**
- Compare total cost of ownership for different cooling approaches (heatsink vs. fan vs. liquid cooling)
- Review thermal design margins across product portfolio and prioritise at-risk designs for mitigation
- Evaluate thermal implications of component end-of-life substitutions on existing product lines
- Track thermal qualification status across product variants for market launch readiness
- Present thermal risk assessment and mitigation plans to executive leadership and customers
