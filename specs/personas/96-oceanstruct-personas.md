# OceanStruct — Persona Subspecs

> Parent spec: [`specs/96-oceanstruct.md`](../96-oceanstruct.md)

---

## Experience-Level Personas

### Student/Academic

**Modified UI/UX:**
- Simplified modeler with template-based jacket, monopile, and semi-submersible structures
- Wave loading visualization showing Morison equation forces as waves pass through the structure
- Theory reference panel linking analysis steps to DNV standards and offshore engineering textbooks

**Feature Gating:**
- Static in-place analysis with regular wave loading; no fatigue or dynamic analysis
- Pre-defined wave spectra (JONSWAP, Pierson-Moskowitz) with limited parameter editing
- Export restricted to result plots and academic report format; no certified design reports

**Pricing Tier:** Free tier

**Onboarding Flow:**
- Tutorial: model a simple jacket structure, apply wave/current loading, check member utilization
- Pre-loaded examples (fixed platform, monopile, FPSO) with annotated loading and results
- Educational mode showing Morison equation, diffraction theory, and member check calculations

**Key Workflows:**
- Analyze a fixed jacket platform under regular wave and current loading
- Study the effect of wave height, period, and current on structural response
- Perform member strength checks using API RP 2A or ISO 19902 simplified criteria
- Compare Morison-based and diffraction-based wave loading on cylindrical structures

---

### Junior Engineer (0-3 years)

**Modified UI/UX:**
- Parametric structure wizard: select platform type, water depth, and topside weight to auto-generate baseline model
- Environmental loading setup assistant with metocean data import and load case generation
- Code-check dashboard showing member utilization ratios color-coded by limit state

**Feature Gating:**
- Full static and quasi-static analysis with irregular wave loading
- Basic fatigue analysis using simplified S-N curve approach for tubular joints
- Foundation modeling limited to linear springs; advanced pile-soil interaction locked

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Metocean data import wizard: wave scatter diagrams, current profiles, wind speed distributions
- 20-minute guided analysis of a monopile foundation under storm and operational conditions
- Firm standard connection: approved tubular joint classifications and corrosion allowances

**Key Workflows:**
- Model fixed platform structures (jacket, monopile, tripod) with topside loading
- Generate storm and operating environmental load cases from metocean data
- Perform in-place code checks per API RP 2A-WSD or ISO 19902 LRFD
- Conduct simplified fatigue assessment of critical tubular joints
- Produce structural analysis reports for design review submission

---

### Senior/Expert

**Modified UI/UX:**
- Full modeling environment with nonlinear analysis, time-domain dynamic simulation, and coupled floater-mooring tools
- Hydrodynamic analysis workspace with panel method diffraction/radiation and Morison hybrid capability
- Scripting console for automated load case generation, batch analysis, and custom post-processing

**Feature Gating:**
- All modules unlocked: nonlinear pushover, spectral fatigue, time-domain dynamics, coupled floater analysis
- Pile-soil interaction with p-y, t-z, and Q-z curves; advanced soil constitutive models
- API for parametric studies, reliability analysis, and integration with external hydrodynamic codes

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Migration assistant for SACS, Sesam (GeniE/Sestra), and OrcaFlex model import
- Hydrodynamic solver configuration and validation against model test data
- API setup with sample scripts for automated platform assessment campaigns

**Key Workflows:**
- Perform time-domain dynamic analysis of floating structures with coupled mooring and risers
- Conduct spectral fatigue analysis using directional wave scatter with hotspot stress S-N curves
- Run nonlinear pushover analysis for ultimate strength assessment (RSR evaluation)
- Model pile-soil interaction with site-specific geotechnical data for foundation design
- Execute structural integrity assessment campaigns across fleet of aging platforms

---

## Role-Based Personas

### Designer/Creator

**Modified UI/UX:**
- 3D structural modeler with parametric joint generation, member grouping, and section optimization
- Topside layout integration: equipment placement, deck plating, and appurtenance loading
- Weight control tracking dashboard with margin and contingency monitoring

**Feature Gating:**
- Full structural modeling: joints, members, plates, stiffeners, and appurtenances
- Section optimization tools for minimum weight under code-check constraints
- CAD export for detailed design and fabrication drawing generation

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Platform type selector with parametric baseline model generation
- Topside loading assistant: equipment weights, operational loads, and area loads
- Integration setup with 3D CAD (Tekla, Aveva E3D, PDMS) for detail design handoff

**Key Workflows:**
- Design jacket and topside structures with optimized member sizes and joint configurations
- Configure brace patterns, leg spacing, and mudline elevation for site-specific conditions
- Detail connection designs: tubular joints, bolted connections, and grouted piles
- Track weight growth through design phases with margin management
- Export structural models for fabrication detail design in 3D CAD systems

---

### Analyst/Simulator

**Modified UI/UX:**
- Hydrodynamic and structural analysis workspace with load case matrix management
- Wave kinematics visualizer showing particle velocities and accelerations through water column
- Dynamic response visualization: time histories, spectral density plots, and statistical summaries

**Feature Gating:**
- All analysis solvers: static, dynamic (frequency/time domain), nonlinear, and stability
- Hydrodynamic analysis: Morison, diffraction/radiation, and coupled Morison-diffraction hybrid
- Advanced foundation modeling with nonlinear pile-soil springs and group effects

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Environmental loading setup tutorial: wave theory selection, current profile, and wind loading
- Solver configuration guide for static, quasi-static, and fully dynamic analysis methods
- Validation benchmarks against published code-check and hydrodynamic examples

**Key Workflows:**
- Perform comprehensive in-place analysis under 100-year storm and operating conditions
- Conduct hydrodynamic analysis for floating structures: RAOs, motion responses, and air gap
- Run time-domain coupled analysis of FPSO with mooring and riser systems
- Analyze ship impact, dropped object, and blast/fire accidental loading scenarios
- Compute long-term fatigue damage at critical joints from directional scatter data

---

### Manufacturing/Process

**Modified UI/UX:**
- Fabrication breakdown view: lifts, sub-assemblies, and weld maps with sequence dependencies
- Load-out and transportation analysis tools: barge stability, seafastening, and float-over simulation
- Installation sequence animator with crane capacity envelopes and weather window requirements

**Feature Gating:**
- Fabrication load case generation: lifting, upending, and temporary support conditions
- Transportation analysis: barge motion response, inertial loading, and seafastening design
- Installation engineering: heavy lift analysis, float-over simulation, and pile driving

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Fabrication yard capability input: craneage, quay load limits, and build area constraints
- Transport vessel configuration: barge dimensions, motion characteristics, route metocean data
- Installation vessel fleet library with crane capacity curves and operational limits

**Key Workflows:**
- Plan fabrication sequence: module breaks, lift weights, and assembly yard logistics
- Analyze load-out procedures: skidding, self-propelled trailer, or crane operations
- Simulate transportation with barge motion-induced inertial loads and seafastening design
- Design heavy lift rigging and verify crane capacity for jacket and topside installation
- Assess weather window requirements for marine operations using seasonal metocean data

---

### Regulatory/Compliance

**Modified UI/UX:**
- Code compliance matrix mapping every design check to DNV, API, ISO, or class society requirements
- Structural integrity management dashboard: inspection history, corrosion monitoring, remaining life
- Audit trail for all analysis inputs, assumptions, and result approvals with digital signatures

**Feature Gating:**
- Full code checking per API RP 2A, ISO 19902, DNV-ST-0126, and class society rules
- Structural integrity assessment tools for life extension of aging platforms
- Risk-based inspection planning using fatigue and corrosion degradation predictions

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Regulatory jurisdiction and class society selector loading applicable standards and safety factors
- Platform fleet import for structural integrity management program setup
- Inspection database integration for condition assessment data management

**Key Workflows:**
- Verify structural design compliance with applicable codes and class society requirements
- Perform fitness-for-service assessments for platforms with inspection findings (cracks, corrosion)
- Develop risk-based inspection plans prioritizing critical joints by fatigue and corrosion risk
- Support life extension studies with updated loading, material properties, and remaining capacity
- Prepare certification documentation packages for regulatory authority and class society review

---

### Manager/Decision-maker

**Modified UI/UX:**
- Fleet-level dashboard with platform age, condition grade, remaining life, and OPEX forecasts
- Capital project portfolio view: new-build status, modification projects, and decommissioning schedule
- Risk and cost visualization for structural integrity management investment decisions

**Feature Gating:**
- Read-only access to all structural analyses, compliance status, and integrity assessments
- Fleet-level cost estimation for maintenance, repair, modification, and decommissioning
- Investment prioritization tools ranking structural integrity interventions by risk reduction per dollar

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Platform fleet import with age, design basis, inspection history, and integrity status
- Cost database configuration: inspection costs, repair rates, modification budgets
- Dashboard KPI setup: fleet availability, integrity compliance rate, maintenance backlog

**Key Workflows:**
- Review fleet structural integrity status and prioritize intervention investments
- Evaluate life extension vs. decommissioning decisions based on remaining capacity and OPEX
- Approve concept selection for new platform developments based on structural cost comparisons
- Monitor construction and installation project progress against budget and schedule
- Generate asset integrity management reports for regulatory authorities and joint venture partners
