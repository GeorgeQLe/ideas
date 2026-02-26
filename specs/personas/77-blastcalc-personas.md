# BlastCalc — Persona Subspecs

> Parent spec: [`specs/77-blastcalc.md`](../77-blastcalc.md)

---

## Experience-Level Personas

### Student/Academic
**Modified UI/UX:**
- Interactive blast wave visualizer with annotated Friedlander waveform and Hopkinson-Cranz scaling diagrams
- Simplified charge configuration editor with TNT-equivalent calculator and common explosive presets
- Educational mode with step-by-step explanations of blast physics (Rankine-Hugoniot, reflection, diffraction)

**Feature Gating:**
- Empirical blast parameter calculators (Kingery-Bulmash, ConWep curves) fully available
- 1D shock propagation simulation unlocked; 2D/3D Euler and ALE solvers locked
- Structural response limited to SDOF models; full FEA coupling restricted

**Pricing Tier:** Free tier (academic license with institutional verification)

**Onboarding Flow:**
- Tutorial: calculate blast parameters (peak pressure, impulse, arrival time) for a free-field TNT detonation
- Pre-loaded examples: hemispherical surface burst, reflected blast on a rigid wall, internal explosion
- Guided comparison of empirical predictions against published experimental data

**Key Workflows:**
- Computing blast parameters using Kingery-Bulmash empirical relations for various standoff distances
- Calculating TNT equivalence for different explosive types
- Estimating structural loading on simple building facades using SDOF response models
- Generating pressure-time history plots for protective design coursework

### Junior Engineer (0-3 years)
**Modified UI/UX:**
- Guided scenario builder: define charge, environment, and target structure with template-based workflows
- Automated standoff distance and charge weight scaling with Hopkinson-Cranz law
- Visual overlay of damage levels (GSA, UFC criteria) on building floor plans at varying standoff distances

**Feature Gating:**
- Full empirical blast loading calculations including reflected and cleared pressures
- SDOF structural response analysis for standard building components (walls, windows, columns)
- 2D axisymmetric blast propagation; full 3D simulations and coupled fluid-structure interaction require approval

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Role-based setup: protective design vs. forensic analysis vs. demolition engineering vs. military application
- Walkthrough: assess blast vulnerability of a building facade for a vehicle-borne IED scenario
- Introduction to UFC 3-340-02 design approach and load calculation procedures

**Key Workflows:**
- Calculating blast loads on structures using empirical and semi-empirical methods
- Performing SDOF analysis of walls, columns, and windows against blast loading
- Assessing building vulnerability using UFC/GSA protection standards
- Designing standoff distances and perimeter security layouts for force protection
- Generating blast assessment reports for protective design submittals

### Senior/Expert
**Modified UI/UX:**
- Multi-physics simulation console coupling blast wave propagation, structural response, and fragmentation
- Custom equation-of-state editor for novel explosives and reactive materials (JWL, BKW parameterization)
- Python/Lua scripting interface for automated parametric blast studies and risk-based analysis

**Feature Gating:**
- All modules: 3D Euler/ALE blast propagation, coupled FSI, fragmentation, thermal effects, buried charges
- Full API for integration with structural FEA codes (LS-DYNA, Abaqus), CFD tools, and risk frameworks
- Admin controls for classified/restricted explosive data libraries and access-controlled analysis

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Migration assistant for importing AUTODYN/LS-DYNA models and legacy blast analysis databases
- Configuration of custom explosive material libraries and equation-of-state parameters
- Setup of security-controlled project workspaces with classification-appropriate access restrictions

**Key Workflows:**
- Running 3D blast propagation simulations through complex urban and interior geometries
- Performing coupled fluid-structure interaction for detailed structural damage prediction
- Modeling blast-driven fragmentation with fragment trajectory and lethality assessment
- Simulating buried charge detonation with soil-structure interaction effects
- Developing and validating blast models against instrumented test data

---

## Role-Based Personas

### Designer/Creator
**Modified UI/UX:**
- Protective structure designer with blast-resistant wall system configurator (reinforced concrete, steel, composite)
- Blast-resistant glazing specification tool with laminate layup and framing system design
- Hardened facility layout planner with internal blast path, venting, and suppression system design

**Feature Gating:**
- Full access to protective design tools for walls, roofs, doors, windows, and utility openings
- Blast-resistant component sizing calculators per UFC 3-340-02 and PDC-TR guidelines
- Export to structural engineering formats with blast-specific load combinations

**Pricing Tier:** Professional or Enterprise tier

**Onboarding Flow:**
- Quick-start: design a blast-resistant masonry wall for a given charge weight and standoff
- Tutorial on progressive collapse-resistant design and alternate load path analysis
- Walkthrough of blast-resistant door and window specification process

**Key Workflows:**
- Designing blast-resistant wall systems meeting specified protection levels
- Specifying blast-resistant glazing systems (laminated glass, catch systems, framing)
- Designing hardened rooms and explosive storage facilities per DoD and ATF standards
- Configuring blast venting and suppression systems for internal explosion scenarios
- Creating protective design packages for construction with blast-specific details

### Analyst/Simulator
**Modified UI/UX:**
- Simulation control panel with Euler/ALE mesh setup, detonation initialization, and time-stepping controls
- Real-time blast wave propagation visualization with pressure field animation
- Structural response dashboard with displacement time-histories, damage levels, and ductility ratios

**Feature Gating:**
- Full access to all blast propagation solvers (1D/2D/3D Euler, ALE, coupled FSI)
- Advanced post-processing including impulse mapping on structures, clearing corrections, and confinement effects
- HPC integration for large 3D cityscale blast simulations

**Pricing Tier:** Professional or Enterprise tier

**Onboarding Flow:**
- Guided first simulation: free-field blast propagation with comparison to Kingery-Bulmash empirical data
- Setup of analysis templates for common scenarios (external vehicle bomb, internal accidental explosion)
- Introduction to mesh sensitivity studies and convergence verification procedures

**Key Workflows:**
- Running 3D blast wave propagation through complex building geometries with reflection/diffraction
- Performing coupled fluid-structure interaction to predict structural damage and collapse mechanisms
- Analyzing blast channeling in urban corridors and blast penetration through openings
- Simulating internal explosions with gas dynamics, venting, and quasi-static pressure buildup
- Validating computational models against arena test data and published benchmark cases

### Manufacturing/Process
**Modified UI/UX:**
- Blast-resistant construction specification generator with material requirements and QA/QC checklists
- Demolition planning tool with charge placement, delay sequencing, and exclusion zone calculation
- Retrofit design advisor for upgrading existing structures to meet blast protection standards

**Feature Gating:**
- Access to construction specification and demolition planning tools
- Charge placement optimization for controlled demolition sequencing
- QA/QC checklist generators for blast-resistant construction inspection

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Configure context: protective construction vs. controlled demolition vs. mining/quarrying
- Walkthrough: generate a blast-resistant wall construction specification from a completed analysis
- Tutorial on demolition charge placement optimization and timing sequence design

**Key Workflows:**
- Generating construction specifications for blast-resistant building components
- Planning controlled demolition with optimized charge placement and delay sequencing
- Developing retrofit designs for upgrading existing structures to current blast standards
- Creating quality assurance checklists for blast-resistant construction inspection
- Estimating material quantities and construction costs for protective design implementations

### Regulatory/Compliance
**Modified UI/UX:**
- Standards compliance dashboard mapping design features to UFC, GSA, ASCE, and ISC requirements
- Antiterrorism force protection (ATFP) assessment wizard per DoD UFC 4-010-01
- Automated compliance reporting with pass/fail status against specified protection levels

**Feature Gating:**
- Read-only analysis access; full compliance checking and documentation tools
- Automated assessment against UFC 4-010-01, ISC Security Design Criteria, GSA standards
- Blast assessment report generator meeting agency-specific documentation requirements

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Configuration of applicable standards framework (DoD UFC, GSA, ISC, ASCE, local building codes)
- Setup of compliance reporting templates and review/approval workflows
- Walkthrough: generate an ATFP assessment report for a sample facility

**Key Workflows:**
- Assessing facility compliance with DoD antiterrorism/force protection standards
- Verifying building designs meet ISC Security Design Criteria protection levels
- Generating blast vulnerability assessment reports per GSA and FEMA methodologies
- Tracking remediation of identified blast protection deficiencies to closure
- Maintaining classified and controlled unclassified blast assessment documentation

### Manager/Decision-maker
**Modified UI/UX:**
- Risk assessment portfolio dashboard showing blast vulnerability across a facility campus or building portfolio
- Cost-benefit analysis tool comparing protective design options against risk reduction achieved
- Security investment prioritization matrix ranking upgrades by risk reduction per dollar

**Feature Gating:**
- Dashboard and reporting access; no direct analysis or design tools
- Risk quantification and investment prioritization views
- Portfolio-level vulnerability tracking and remediation budget planning

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Configuration of facility portfolio with asset values, occupancy, and mission criticality ratings
- Integration with security risk management frameworks and budget systems
- Setup of periodic vulnerability review cadence and reporting distribution

**Key Workflows:**
- Prioritizing blast protection investments across a portfolio of facilities based on risk
- Evaluating cost-effectiveness of protective design alternatives (standoff vs. hardening vs. detection)
- Reviewing blast vulnerability assessment results and approving remediation plans
- Communicating residual blast risk to leadership and stakeholders
- Tracking implementation progress and budget execution for approved protective upgrades
