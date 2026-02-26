# ExplodeSafe — Persona Subspecs

> Parent spec: [`specs/97-explodesafe.md`](../97-explodesafe.md)

---

## Experience-Level Personas

### Student/Academic

**Modified UI/UX:**
- Simplified composition editor with common energetic compound library (RDX, HMX, TNT, PETN, AN)
- Interactive thermochemical output display: detonation velocity, pressure, and product species breakdown
- Theory panel explaining Chapman-Jouguet theory, BKW/JWL equations of state, and oxygen balance

**Feature Gating:**
- Ideal detonation thermochemistry calculations only; no non-ideal or blast propagation
- Compound library limited to well-characterized military and commercial explosives
- No access to sensitive formulation optimization or vulnerability assessment tools

**Pricing Tier:** Free tier (with institutional verification)

**Onboarding Flow:**
- Tutorial: calculate detonation properties of a TNT/RDX mixture and interpret CJ parameters
- Pre-loaded textbook examples with step-by-step thermochemical calculation annotations
- Institutional affiliation verification required; access logged for compliance

**Key Workflows:**
- Calculate Chapman-Jouguet detonation parameters for standard explosive compositions
- Study the effect of composition ratios on detonation velocity and pressure
- Analyze detonation product species equilibrium at various loading densities
- Compare oxygen balance, heat of detonation, and brisance across explosive families

---

### Junior Engineer (0-3 years)

**Modified UI/UX:**
- Guided formulation workspace with safety constraint indicators (sensitivity, compatibility, thermal stability)
- Performance prediction dashboard: VOD, detonation pressure, Gurney energy, and blast equivalence
- Material compatibility checker flagging known incompatible ingredient combinations

**Feature Gating:**
- Full ideal detonation code with expanded ingredient database
- Non-ideal detonation modeling for simple geometries (rate sticks, cylinder tests)
- Blast equivalence and fragmentation performance estimators; detailed blast propagation locked

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Safety briefing walkthrough: handling classifications, sensitivity data interpretation, compatibility rules
- Guided example: design an insensitive munition formulation meeting IM requirements
- Firm ingredient database connection with approved binders, oxidizers, and processing constraints

**Key Workflows:**
- Formulate explosive compositions meeting specified performance and sensitivity criteria
- Predict detonation velocity and pressure as a function of density and charge diameter
- Estimate Gurney velocity and fragment initial velocity for warhead design support
- Calculate TNT equivalence for blast damage assessment
- Generate formulation specification sheets with predicted performance properties

---

### Senior/Expert

**Modified UI/UX:**
- Advanced modeling environment with non-ideal detonation physics, reactive flow hydrocodes, and EOS fitting
- Multi-physics coupling: detonation to blast propagation to structural response chain
- Equation of state parameter calibration workspace with experimental data fitting tools

**Feature Gating:**
- All modules unlocked: non-ideal detonation, reactive flow (Ignition & Growth, CREST), hydrocode coupling
- EOS parameter fitting from cylinder test, plate push, and detonation calorimetry data
- Classified/export-controlled module access with appropriate security clearance verification

**Pricing Tier:** Enterprise tier (with security clearance requirements)

**Onboarding Flow:**
- Security credential verification and classified network configuration (if applicable)
- Migration assistant for CHEETAH, Explo5, and AUTODYN data import
- EOS calibration tutorial using experimental detonation data

**Key Workflows:**
- Model non-ideal detonation behavior in small-diameter charges and heterogeneous formulations
- Calibrate reactive flow model parameters from experimental detonation characterization data
- Simulate sympathetic detonation and insensitive munition response scenarios
- Perform coupled detonation-blast-structure interaction analysis for protective design
- Develop and validate new energetic material models against experimental campaigns

---

## Role-Based Personas

### Designer/Creator

**Modified UI/UX:**
- Formulation design canvas with ingredient palette, ratio sliders, and real-time performance prediction
- Constraint-driven optimizer: target performance, sensitivity limits, processability requirements
- Charge geometry designer: warhead configurations, shaped charge liners, booster trains

**Feature Gating:**
- Full formulation design tools with multi-component optimization
- Charge geometry modeling with initiation train design and reliability prediction
- Shaped charge jet formation and penetration estimation tools

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Ingredient database browser with performance, sensitivity, and compatibility data
- Formulation optimizer setup: define performance targets, safety constraints, and processability bounds
- Charge design template gallery for common warhead and demolition configurations

**Key Workflows:**
- Design new explosive formulations optimizing performance within sensitivity constraints
- Configure initiation trains: detonator, booster, and main charge compatibility verification
- Model shaped charge geometry and predict jet tip velocity and penetration depth
- Evaluate alternative binder systems for processability and mechanical property improvements
- Generate formulation specifications with complete predicted property documentation

---

### Analyst/Simulator

**Modified UI/UX:**
- Simulation workspace with reactive flow model configuration, mesh controls, and time-step management
- Detonation wave visualization: pressure, temperature, and reaction extent fields
- Experimental data overlay tools for model validation against test measurements

**Feature Gating:**
- Full hydrocode coupling with reactive flow detonation models
- Blast propagation solvers: empirical (Kingery-Bulmash), CFD, and coupled CFD-FEA
- Fragment launch, trajectory, and terminal effects prediction tools

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Solver selection guide: thermochemical codes vs. reactive flow vs. full hydrocode
- Model validation tutorial using published cylinder test and detonation velocity data
- Mesh sensitivity study workflow with convergence assessment tools

**Key Workflows:**
- Simulate detonation wave propagation through complex charge geometries
- Model blast wave propagation in urban environments and confined spaces
- Predict fragmentation patterns: size distribution, velocity, and spatial density
- Analyze structural response to blast loading for protective design
- Validate simulation predictions against experimental firing data and test results

---

### Manufacturing/Process

**Modified UI/UX:**
- Process parameter dashboard: mixing sequences, pressing conditions, casting temperatures
- Quality control specification generator with acceptance criteria for density, sensitivity, and performance
- Batch consistency tracker monitoring lot-to-lot variability in critical properties

**Feature Gating:**
- Process-property relationship models linking manufacturing parameters to final product performance
- Scale-up prediction tools from lab-scale to production-scale formulations
- Quality acceptance testing specification generator with statistical sampling plans

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Manufacturing process type selector: press, cast, extruded, or slurry formulations
- Process parameter bounds configuration based on equipment and safety constraints
- Quality management system integration for batch record tracking

**Key Workflows:**
- Predict how processing parameter variations affect final product detonation performance
- Design quality acceptance test protocols with pass/fail criteria for each critical property
- Analyze batch-to-batch consistency using statistical process control methods
- Evaluate alternative manufacturing processes for cost, safety, and quality trade-offs
- Generate process specifications and manufacturing instructions for production

---

### Regulatory/Compliance

**Modified UI/UX:**
- Hazard classification dashboard mapping formulations to UN transport classes and divisions
- Insensitive munition (IM) compliance matrix showing response to each threat stimulus
- Export control checker flagging formulations, data, or technologies subject to ITAR/EAR

**Feature Gating:**
- IM assessment tools: slow/fast cook-off, bullet impact, fragment impact, sympathetic detonation prediction
- Hazard classification predictors for UN transport classification (1.1 through 1.6)
- Environmental compliance tools: perchlorate-free alternatives, lead-free initiators assessment

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Regulatory framework selector: DoD, NATO STANAG, or civil explosive regulations
- IM policy requirements loading for applicable threat stimuli and response criteria
- Export control checklist configuration based on formulation components and end-use

**Key Workflows:**
- Assess formulations against insensitive munition requirements per MIL-STD-2105 and STANAG 4439
- Predict UN hazard classification based on formulation properties and packaging configuration
- Evaluate environmental compliance: replacement of perchlorate, lead, and other restricted materials
- Manage export control documentation for international program collaboration
- Generate safety data packages for regulatory authority review and approval

---

### Manager/Decision-maker

**Modified UI/UX:**
- Program portfolio dashboard: formulation development status, test milestones, and qualification timeline
- Cost-performance trade-off visualizations comparing candidate formulations
- Risk assessment matrix for development, qualification, and production risks

**Feature Gating:**
- Read-only access to all formulation designs, simulation results, and compliance assessments
- Program schedule and budget tracking with milestone-based progress metrics
- Technology readiness level (TRL) assessment and maturation planning tools

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Program portfolio import with formulation candidates, development stages, and schedule milestones
- Cost model configuration: ingredient costs, processing costs, testing costs, qualification budgets
- Dashboard KPI setup: TRL progression, test success rates, schedule compliance

**Key Workflows:**
- Evaluate formulation candidates on performance, cost, safety, and qualification risk
- Track development program progress through testing milestones and TRL gates
- Make go/no-go decisions on formulation down-selection and qualification testing
- Assess supply chain risks for critical ingredients and manufacturing capabilities
- Generate program status reports for sponsors, customers, and oversight authorities
