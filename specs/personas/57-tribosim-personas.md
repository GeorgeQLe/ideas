# TriboSim — Persona Subspecs

> Parent spec: [`specs/57-tribosim.md`](../57-tribosim.md)

---

## Experience-Level Personas

### Student/Academic

**Modified UI/UX:**
- Simplified 2D contact editor with predefined geometries (pin-on-disc, block-on-ring, ball-on-flat) and material presets
- Inline tribology primer explaining Hertzian contact, Archard's wear law, and Stribeck curve with interactive parameter sliders
- Results panel defaults to contact pressure, film thickness, friction coefficient, and wear depth with annotated lubrication regimes

**Feature Gating:**
- Contact models limited to 2D line/point contact; Hertzian and dry-friction Coulomb models
- Lubricant library limited to common mineral oils with Barus viscosity-pressure model; no thermal or EHL solvers
- Export to PDF and CSV; no FEA or multibody dynamics interop

**Pricing Tier:** Free tier (educational license)

**Onboarding Flow:**
- "First Contact" tutorial: set up a Hertzian ball-on-flat contact, calculate stress field, and compare against analytical solution
- Interactive Stribeck curve explorer: vary speed, load, and viscosity to transition between boundary, mixed, and hydrodynamic regimes
- Sandbox with textbook problems (journal bearing, spur gear tooth, cam-follower)

**Key Workflows:**
- Calculate Hertzian contact stress and compare simulation results to closed-form solutions
- Predict wear depth evolution using Archard's model for a pin-on-disc configuration
- Map lubrication regimes on a Stribeck diagram for a given bearing geometry and operating conditions
- Generate lab report plots of friction coefficient vs. sliding speed and contact pressure distributions

---

### Junior Engineer (0-3 years)

**Modified UI/UX:**
- Standard workspace with component-type wizard: select application (bearing, gear, seal, brake) and auto-load geometry templates
- Guided material pairing advisor recommending surface treatments, coatings, and lubricants for given contact conditions
- Warning indicators on contacts operating in boundary lubrication or exceeding allowable contact stress limits

**Feature Gating:**
- Full 2D and simplified 3D contact models; elastohydrodynamic lubrication (EHL) for line and point contacts
- Standard material and lubricant libraries with temperature-dependent properties
- Import from CAD (STEP, IGES); export to FEA packages and engineering reports

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Application selector (automotive drivetrain, industrial gearbox, hydraulic system, biomedical implant) pre-loads relevant standards
- Walkthrough of a rolling element bearing analysis: geometry setup, load application, film thickness prediction, fatigue life estimate
- Prompted to import company's material database and preferred lubricant supplier specifications

**Key Workflows:**
- Analyse a rolling element bearing for minimum film thickness, contact stress, and L10 fatigue life
- Evaluate gear tooth contact patterns and predict scuffing and pitting risk per AGMA/ISO standards
- Select surface treatment (nitriding, DLC coating, shot peening) to extend component service life
- Predict seal lip wear and friction torque for a rotary shaft seal design
- Generate engineering reports with contact analysis results and material/lubricant recommendations

---

### Senior/Expert

**Modified UI/UX:**
- Multi-physics workspace coupling contact mechanics, thermal, wear, and surface chemistry solvers
- Scripting console (Python API) for multi-scale simulation (asperity-level to component-level), custom wear laws, and surrogate modelling
- Experimental data correlation dashboard linking tribometer measurements to model calibration parameters

**Feature Gating:**
- All solvers: Hertzian/non-Hertzian, full EHL (thermal, transient, rough-surface), FEM-based contact, and molecular dynamics interface
- Custom wear law editor, surface roughness importers (profilometry data), and multi-scale coupling framework
- Full API, HPC support, and integration with FEA (ANSYS, Abaqus), multibody dynamics, and CFD tools

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Import existing contact/wear models from ANSYS, Abaqus, or proprietary codes with parameter translation
- Configure solver preferences, surface discretisation, and compute cluster connections
- Power-user preset opening scripted multi-scale workspace for custom wear model development

**Key Workflows:**
- Develop a multi-scale contact model linking asperity-level mechanics to component-level wear prediction
- Perform transient thermal-EHL analysis of a gearbox under mission-cycle loading profiles
- Calibrate wear models against tribometer test data and validate on component-level bench tests
- Model tribochemical film formation (ZDDP, MoDTC) and its effect on friction and wear
- Automate parametric studies exploring texture/coating/lubricant combinations for design optimisation

---

## Role-Based Personas

### Designer/Creator

**Modified UI/UX:**
- Visual component design canvas with parametric contact surfaces; real-time stress and film thickness preview during geometry changes
- Surface texture design studio with pattern generators (dimples, grooves, cross-hatch) and predicted friction/wear effect
- Material and coating selector with comparative performance spider charts (hardness, friction, wear rate, cost)

**Feature Gating:**
- Full geometry editing and parametric surface design tools enabled
- Contact solver runs in background for real-time design feedback; full EHL analysis requires explicit launch
- Export to CAD formats and surface texture manufacturing specifications

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Component-type selector (bearing, gear, piston ring, hip implant) loads design template with recommended contact geometry
- "Surface Engineering" wizard: select performance goals and receive texture/coating/lubricant combination suggestions
- Tour of parametric texture design tools using a thrust bearing pad face example

**Key Workflows:**
- Design a hydrodynamic journal bearing profile (lemon bore, offset halves, tilting pad) optimising for load capacity and stability
- Create surface texture patterns on a cylinder liner to reduce piston ring friction and oil consumption
- Select coating systems (DLC, CrN, WC/C) for automotive drivetrain components based on contact severity
- Design a thrust bearing pad geometry balancing film thickness, temperature, and power loss
- Produce manufacturing specifications for surface finish, texture parameters, and coating thickness

---

### Analyst/Simulator

**Modified UI/UX:**
- Data-centric workspace with contact pressure maps, film thickness profiles, temperature fields, and wear evolution plots
- Multi-model comparison panel contrasting Hertzian vs. FEM-based contact, smooth vs. rough-surface EHL results
- Post-processing tools for extracting friction power loss, contact fatigue metrics, and oil film parameter (Lambda ratio)

**Feature Gating:**
- All solver types enabled: dry contact, EHL (isothermal, thermal, transient, rough-surface), FEM contact, and mixed lubrication
- Surface roughness analysis tools: PSD import, statistical roughness generation, deterministic asperity modelling
- HPC job submission for large 3D rough-surface EHL and coupled thermal-structural analyses

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Validation suite running benchmark problems (Hertz, Dowson-Higginson EHL, AGMA gear scuffing)
- Import surface profilometry data and configure roughness characterisation tools
- Configure solver convergence criteria, mesh refinement strategies, and output recording preferences

**Key Workflows:**
- Perform full 3D thermal EHL analysis of a helical gear pair under varying torque and speed conditions
- Conduct rough-surface mixed lubrication analysis to predict friction contributions from asperity and fluid
- Model rolling contact fatigue initiation using sub-surface stress history and Dang Van criterion
- Evaluate the effect of surface roughness orientation on hydrodynamic bearing performance
- Produce validation reports comparing simulation predictions against published experimental benchmarks

---

### Manufacturing/Process

**Modified UI/UX:**
- Process-centric view organised by surface finishing operations (grinding, honing, lapping, coating deposition)
- Surface quality dashboard linking process parameters to achieved roughness, waviness, and texture regularity
- Quality control panel with go/no-go criteria for tribological surface specifications

**Feature Gating:**
- Process-surface-performance linkage tools: predict tribological behaviour from manufacturing process parameters
- Coating process simulation interface (PVD, CVD thickness uniformity, residual stress prediction)
- Batch analysis for evaluating manufacturing variability impact on tribological performance

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Product line setup: import component drawings, surface finish specifications, and production quality data
- Guided tutorial linking a honing process (crosshatch angle, stone grit) to predicted cylinder liner tribological performance
- Connect to surface metrology instruments (profilometer, AFM) for automated data import

**Key Workflows:**
- Predict how changes in grinding parameters affect bearing raceway surface finish and fatigue life
- Optimise honing crosshatch angle and plateau ratio for engine cylinder liners to balance oil retention and ring friction
- Evaluate coating thickness uniformity from PVD process simulation and its impact on wear life
- Establish surface finish acceptance criteria based on simulated tribological performance thresholds
- Correlate production surface metrology data to simulated and field performance for continuous improvement

---

### Regulatory/Compliance

**Modified UI/UX:**
- Standards compliance dashboard covering tribology-related specifications (ISO 281 bearing life, AGMA gear ratings, EPA emissions)
- Test-standard alignment panel mapping simulation outputs to required test metrics (ASTM G99, G77, D2714)
- Audit trail documenting material certifications, lubricant approvals, and performance validation evidence

**Feature Gating:**
- Standards-based calculation engines (ISO 281 bearing life, ISO 6336 gear rating) fully enabled
- Test correlation tools mapping simulation results to standard tribometer test equivalents
- Documentation generator for regulatory submissions (automotive homologation, medical device biocompatibility)

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Industry selector (automotive, aerospace, medical, industrial) pre-loads applicable tribological standards
- Walkthrough of ISO 281 bearing life calculation with modification factors for contamination and lubrication
- Configure documentation templates for regulatory submissions with required traceability

**Key Workflows:**
- Calculate bearing rating life per ISO 281 with lubrication, contamination, and reliability modification factors
- Evaluate gear contact and bending ratings per ISO 6336 / AGMA 2101 for gearbox certification
- Document biocompatibility and wear debris characterisation for medical implant regulatory submissions (FDA 510(k))
- Verify lubricant performance claims against OEM specifications (API, ACEA, OEM-specific tests)
- Generate compliance documentation linking simulation predictions to validation test results

---

### Manager/Decision-maker

**Modified UI/UX:**
- Portfolio dashboard showing product lines with wear-life predictions, warranty exposure, and material cost breakdowns
- Total cost of ownership comparisons for alternative bearing/surface/lubricant technologies
- Field reliability correlation panel linking simulated wear predictions to actual warranty claim data

**Feature Gating:**
- Read-only access to simulation results; cannot modify contact models or material parameters
- Full access to cost estimation, warranty analytics, and technology benchmarking modules
- Supply chain risk tools for critical tribological materials (coatings, specialty lubricants, bearing steels)

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Product portfolio import linking component specifications to field warranty and maintenance data
- Dashboard configuration selecting KPIs (predicted service life, warranty cost per unit, lubricant consumption)
- Demo of total cost of ownership comparison between premium and standard bearing solutions

**Key Workflows:**
- Compare lifecycle costs of alternative bearing technologies (roller vs. plain vs. magnetic) for a product platform
- Evaluate ROI of advanced coatings or surface treatments against baseline designs using warranty data
- Track tribological component development and validation status across product programs
- Assess supply chain risks for specialty tribological materials and identify alternative sources
- Present wear-life and reliability projections to customers and warranty underwriters
