# TunnelSim — Persona Subspecs

> Parent spec: [`specs/92-tunnelsim.md`](../92-tunnelsim.md)

---

## Experience-Level Personas

### Student/Academic

**Modified UI/UX:**
- Simplified 2D cross-section modeler with pre-defined tunnel shapes (circular, horseshoe, D-shape)
- Step-by-step excavation sequence animator with annotated ground behavior explanations
- Integrated reference panel linking results to Hoek-Brown, Mohr-Coulomb, and convergence-confinement theory

**Feature Gating:**
- 2D plane-strain and axisymmetric analysis only; 3D sequential excavation locked
- Pre-built soil/rock models with limited parameter editing (no user-defined constitutive laws)
- Export restricted to academic reports; no geotechnical baseline report (GBR) templates

**Pricing Tier:** Free tier

**Onboarding Flow:**
- Tutorial project: shallow tunnel in soft ground with step-by-step NATM excavation simulation
- Pre-loaded case studies (Channel Tunnel, Gotthard Base Tunnel) with validated parameter sets
- Campus license activation via university email verification

**Key Workflows:**
- Analyze ground-support interaction using convergence-confinement method
- Simulate simplified NATM top-heading and bench excavation in 2D
- Study the influence of tunnel depth, diameter, and ground conditions on settlement
- Generate annotated displacement and stress contour plots for lab reports

---

### Junior Engineer (0-3 years)

**Modified UI/UX:**
- Guided modeling wizard: select tunnel method (TBM or NATM), input geometry, assign ground profile from borehole data
- Code-check sidebar showing design compliance for lining thickness, reinforcement, and waterproofing
- Predefined ground condition templates aligned with common geotechnical classifications (RMR, Q-system, GSI)

**Feature Gating:**
- Full 2D analysis and basic 3D sequential excavation for single-tube tunnels
- TBM thrust/torque estimation and face stability checks enabled
- Advanced features (coupled hydro-mechanical, time-dependent creep) require senior review approval

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Project setup assistant importing borehole logs and ground investigation data
- 20-minute guided simulation of a TBM-driven tunnel with lining design checks
- Connection to firm's geotechnical database for material properties and ground profiles

**Key Workflows:**
- Model TBM-driven tunnel and estimate face support pressure and thrust requirements
- Design shotcrete and rock bolt support for NATM excavation stages
- Analyze surface settlement profiles for urban tunnel alignments
- Perform lining structural design checks per Eurocode or AASHTO tunnel standards
- Generate ground movement impact assessments for adjacent structures

---

### Senior/Expert

**Modified UI/UX:**
- Full 3D modeling environment with longitudinal profile, cross-section variability, and fault zone handling
- Advanced constitutive model library with parameter calibration tools from lab test data
- Multi-physics coupling dashboard (hydro-mechanical-thermal) with convergence diagnostics

**Feature Gating:**
- All modules unlocked: 3D sequential excavation, coupled THM analysis, dynamic seismic loading
- Scripting API for parametric ground condition studies and probabilistic analysis
- Administrative tools for project template standardization and QA workflow enforcement

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Migration assistant for PLAXIS, FLAC3D, and RS3 model import/conversion
- Constitutive model calibration wizard using triaxial and oedometer test data
- API and scripting environment configuration with sample automation scripts

**Key Workflows:**
- Run full 3D sequential excavation analysis for twin-tube tunnel with cross-passages
- Perform coupled seepage-deformation analysis for tunnels below water table
- Conduct probabilistic ground condition sensitivity studies using Monte Carlo methods
- Design segmental lining with joint behavior modeling under TBM thrust loads
- Manage complex geological transitions (mixed-face, fault zones, karst) in single model

---

## Role-Based Personas

### Designer/Creator

**Modified UI/UX:**
- Cross-section design canvas with parametric lining geometry, reinforcement layout, and waterproofing detail tools
- Real-time structural utilization display as lining dimensions are adjusted
- Component library for pre-cast segments, gaskets, bolts, and connection hardware

**Feature Gating:**
- Full lining design tools: cast-in-place, shotcrete, and segmental lining modules
- Waterproofing system selection and drainage design tools
- Cross-passage and shaft connection detail generators

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Lining type selector (segmental, shotcrete, cast-in-place) with auto-populated design parameters
- Design code setup: Eurocode 2, ACI 318, or project-specific standards
- Template gallery of standard cross-sections by tunnel diameter and method

**Key Workflows:**
- Design segmental lining geometry, reinforcement, and joint configuration
- Detail NATM support: shotcrete thickness, lattice girder spacing, rock bolt patterns
- Design portal structures and cut-and-cover transition sections
- Specify waterproofing membranes and drainage systems for long-term durability
- Generate lining design drawings with reinforcement schedules for construction

---

### Analyst/Simulator

**Modified UI/UX:**
- Analysis-centric workspace with ground model builder, mesh controls, and staged excavation sequencer
- Result visualization suite: displacement vectors, plastic zone contours, pore pressure distribution
- Solver performance dashboard with iteration counts, convergence history, and error metrics

**Feature Gating:**
- Full solver access: implicit/explicit, small/large strain, drained/undrained analysis types
- Advanced constitutive models: hardening soil, soft soil creep, Hoek-Brown, Barcelona basic model
- Coupled and dynamic analysis modules for seismic and blasting assessments

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Ground model calibration tutorial using field monitoring data (extensometers, inclinometers)
- Mesh sensitivity study guide with automated convergence checking
- Benchmark validation suite against closed-form solutions and published case histories

**Key Workflows:**
- Build 3D ground models from borehole data with interpolated stratigraphy
- Run sequential excavation simulations tracking ground relaxation at each stage
- Analyze face stability for pressurized TBM operations in mixed ground
- Perform seismic deformation analysis for tunnels in active seismic zones
- Calibrate models against monitoring data and update predictions during construction

---

### Manufacturing/Process

**Modified UI/UX:**
- Construction sequence timeline with resource allocation (TBM cycles, shotcrete volumes, bolt quantities)
- TBM operation dashboard: advance rate prediction, cutter wear estimation, grout volume tracking
- Material logistics planner for segment delivery, muck removal, and ventilation scheduling

**Feature Gating:**
- TBM performance prediction module with geological condition-dependent advance rates
- NATM cycle time estimator based on ground class and support requirements
- Material quantity extraction for procurement and cost estimation

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- TBM specification input wizard: machine type, cutterhead diameter, thrust capacity
- Ground condition-to-support-class mapping configuration
- Integration with project scheduling tools (Primavera, MS Project) for timeline sync

**Key Workflows:**
- Predict TBM advance rates across varying geological conditions along alignment
- Estimate material quantities: segments, grout, shotcrete, rock bolts per ring/round
- Plan muck disposal logistics based on excavation volumes and spoil classification
- Schedule segment production to match TBM advance and ring erection cycles
- Track actual vs. predicted performance and update remaining drive forecasts

---

### Regulatory/Compliance

**Modified UI/UX:**
- Compliance matrix mapping design elements to applicable codes (ITA guidelines, Eurocode 7, local regulations)
- Environmental impact tracker: settlement monitoring zones, groundwater drawdown limits, vibration thresholds
- Document management panel linking calculations to permit requirements and regulatory submissions

**Feature Gating:**
- Settlement impact assessment tools with building damage classification (Burland-Wroth framework)
- Groundwater impact analysis with drawdown prediction and mitigation evaluation
- Noise and vibration prediction modules for surface receptor assessment

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Regulatory framework selector: jurisdiction-specific permit requirements and thresholds
- Building damage risk assessment setup with adjacent structure inventory import
- Environmental baseline data import (groundwater levels, existing settlement, vibration background)

**Key Workflows:**
- Assess building damage risk from tunnel-induced settlements using greenfield and interaction analyses
- Predict groundwater drawdown impacts and evaluate mitigation measures (recharge, cutoff walls)
- Generate regulatory submission packages with code-compliant calculation reports
- Monitor compliance with environmental permits during construction phase
- Document design changes and their regulatory implications through audit trail

---

### Manager/Decision-maker

**Modified UI/UX:**
- Project overview dashboard: tunnel alignment progress, risk heat map, cost and schedule tracking
- Alternative comparison matrix for method selection (TBM vs. NATM vs. cut-and-cover)
- Risk register visualization with probability-impact scoring and mitigation status

**Feature Gating:**
- Read-only access to design models and analysis results; no direct editing
- Cost estimation and value engineering comparison tools
- Schedule risk analysis with Monte Carlo simulation for completion date forecasting

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Project portfolio setup with alignment data, ground investigation summary, and budget baseline
- Risk register import from existing project management systems
- Dashboard KPI configuration: advance rate, cost per meter, safety incidents, schedule variance

**Key Workflows:**
- Compare tunneling method alternatives by cost, schedule, risk, and environmental impact
- Monitor project progress against baseline schedule and budget
- Review ground condition risk assessments and contingency adequacy
- Evaluate claims and variation orders based on geotechnical baseline report compliance
- Generate executive project status reports for stakeholders and funding agencies
