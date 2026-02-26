# GridSolve — Persona Subspecs

> Parent spec: [`specs/47-gridsolve.md`](../47-gridsolve.md)

---

## Experience-Level Personas

### Student/Academic

**Modified UI/UX:**
- Single-line diagram editor with drag-and-drop components (generators, transformers, lines, loads, buses) and auto-numbering
- Inline power-systems theory panel explaining per-unit calculations, load-flow methods, and fault-current formulas as analyses run
- Pre-loaded example grids (IEEE 9-bus, 14-bus, 30-bus test systems) for immediate experimentation

**Feature Gating:**
- Power-flow analysis (Newton-Raphson, Gauss-Seidel), short-circuit analysis, and basic transient stability available
- Optimal power flow, protection coordination, harmonic analysis, and real-time simulation locked
- Network limited to 100 buses; dynamic models limited to classical machine representations

**Pricing Tier:** Free tier (education license with .edu verification)

**Onboarding Flow:**
- "Solve Your First Power Flow" tutorial: load IEEE 14-bus system, run Newton-Raphson, and interpret bus voltages and line flows
- Interactive exercises comparing hand calculations (per-unit, Y-bus formation, power flow iteration) with tool results
- Pre-configured component libraries with textbook-standard generator, transformer, and line models

**Key Workflows:**
- Building small power-system models and running load-flow analysis to determine bus voltages and line loadings
- Performing symmetrical and asymmetrical short-circuit calculations and verifying against hand-computed results
- Running basic transient stability simulations (fault-on, fault-cleared scenarios) with classical generator models
- Generating single-line diagrams and power-flow results tables for lab reports and assignments

---

### Junior Engineer (0-3 years)

**Modified UI/UX:**
- Professional single-line diagram workspace with component databases, tabular data entry, and results overlay
- Scenario manager for comparing normal, contingency (N-1), and peak/off-peak operating conditions
- Report generator with customizable templates for load-flow summaries, fault-current tables, and equipment-loading reports

**Feature Gating:**
- Full load-flow, short-circuit (IEC/ANSI methods), motor starting, and detailed transient stability unlocked
- Protection coordination (overcurrent, distance, differential) available with standard relay libraries
- Optimal power flow and harmonic analysis available in guided mode; real-time simulation restricted

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- System-import wizard: import from PSS/E RAW/DYR, ETAP, or CIM/CGMES formats with validation and mapping report
- Guided first-study workflow: define study parameters, run analysis, review results, generate report
- Industry-standard relay and equipment libraries pre-loaded with major manufacturer data (ABB, Siemens, SEL, GE)

**Key Workflows:**
- Modeling utility or industrial power systems with detailed generator, transformer, cable, and load data
- Running N-1 contingency analysis to identify thermal overloads and voltage violations
- Performing short-circuit calculations per IEC 60909 or ANSI/IEEE standards for equipment rating verification
- Coordinating overcurrent protection devices using time-current characteristic curves
- Generating engineering reports for utility interconnection studies and facility power-system assessments

---

### Senior/Expert

**Modified UI/UX:**
- Multi-study workspace with simultaneous load-flow, stability, EMT, protection, and reliability analyses
- Custom scripting environment for automated contingency scanning, batch parametric studies, and optimization
- Real-time co-simulation interface for controller HIL testing and SCADA integration

**Feature Gating:**
- All capabilities unlocked: optimal power flow, EMT simulation, sub-synchronous resonance, voltage stability, reliability (LOLP/EENS)
- User-defined dynamic models (custom generators, FACTS devices, renewable inverters, storage controllers)
- API for integration with EMS/SCADA, planning databases, and GIS systems

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Migration of large-scale system models from PSS/E, PSCAD, or ETAP with full dynamic-model mapping
- Configure custom dynamic-model library for proprietary equipment (wind/solar inverters, BESS controllers, HVDC)
- Enterprise integration setup with utility EMS/SCADA, outage management, and asset management systems

**Key Workflows:**
- Performing large-scale transmission planning studies with thousands of contingencies and multiple planning horizons
- Running electromagnetic transient (EMT) simulations for inverter-based-resource integration studies
- Developing custom dynamic models for novel grid-connected equipment (grid-forming inverters, hybrid plants)
- Conducting system restoration studies and black-start capability assessments
- Mentoring team members on modeling best practices and reviewing study methodologies

---

## Role-Based Personas

### Designer/Creator

**Modified UI/UX:**
- System-design canvas for new facility layout with one-line diagram, equipment sizing calculators, and cable-route planning
- Interactive equipment-sizing tools (transformer kVA, cable ampacity, switchgear rating) integrated into the design flow
- Template library for common power-system configurations (industrial facility, data center, solar farm, wind farm)

**Feature Gating:**
- Full single-line-diagram creation with equipment sizing, load-flow, and short-circuit for design validation
- Cable-ampacity and voltage-drop calculators with NEC/IEC code compliance checks
- Detailed stability and EMT analysis available but secondary to facility design

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- "Design a Facility" wizard: select configuration type (radial, loop, double-ended) and auto-generate base topology
- Equipment-sizing tutorial: define load schedule, size transformers and cables, verify with load flow
- Import site GIS data or floor plans as background layer for geographically referenced one-line layout

**Key Workflows:**
- Designing power distribution systems for new industrial facilities or data centers
- Sizing transformers, cables, and switchgear based on load analysis and fault-current calculations
- Evaluating alternative configurations (radial vs. loop, redundancy levels) through comparative analysis
- Performing arc-flash hazard assessment to inform equipment labeling and PPE requirements
- Generating preliminary design packages with one-line diagrams, equipment schedules, and cable schedules

---

### Analyst/Simulator

**Modified UI/UX:**
- Analysis-centric layout with scenario tree, batch-run manager, results browser, and post-processing plots front and center
- Convergence and solver-diagnostics panel for troubleshooting non-converging load flows and unstable dynamic simulations
- Multi-scenario comparison tool overlaying voltage profiles, power flows, and stability metrics across study cases

**Feature Gating:**
- All analysis solvers unlocked: load flow, short circuit, transient/voltage/frequency stability, EMT, harmonic, reliability
- Advanced solver controls: custom solution sequences, user-defined monitors, and event scripting
- Model creation tools available but secondary to analysis setup and execution

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Solver-validation suite: run IEEE/CIGRE benchmark test systems and compare with published results
- Configure default analysis parameters (load-flow method, fault-calculation standard, dynamic simulation timestep)
- Tutorial on advanced stability analysis techniques: modal analysis, Prony analysis, and small-signal stability

**Key Workflows:**
- Executing comprehensive contingency analysis (N-1, N-2) across multiple operating scenarios
- Performing transient stability simulations for fault events, generation trip, and load rejection scenarios
- Running voltage-stability analysis (PV curves, QV curves) to determine transfer limits and reactive margins
- Conducting harmonic analysis for facilities with nonlinear loads and evaluating filter designs
- Producing detailed analysis reports with methodology, assumptions, results, and recommendations

---

### Manufacturing/Process

**Modified UI/UX:**
- Commissioning and testing workspace with relay-setting calculators, test-sheet generators, and verification checklists
- Protection-setting-comparison panel overlaying calculated settings against existing relay configurations
- Field-test data importer for correlating actual measurements with model predictions

**Feature Gating:**
- Protection coordination, relay-setting calculation, and arc-flash analysis fully unlocked
- Commissioning test-sheet generators and verification templates available
- System-planning and long-term reliability tools available but secondary to operational focus

**Pricing Tier:** Professional tier with Commissioning add-on

**Onboarding Flow:**
- Protection-setting methodology setup: define company relay-setting standards and coordination philosophies
- Import existing relay-setting databases and protection coordination diagrams
- Configure test-sheet templates with company-specific format and acceptance criteria

**Key Workflows:**
- Calculating and verifying protection relay settings for overcurrent, distance, and differential schemes
- Performing protection coordination studies to ensure proper selectivity and fault clearance
- Generating relay test sheets and commissioning verification documents
- Conducting arc-flash assessments and updating equipment warning labels
- Validating power-system models against field measurements (fault recordings, PMU data)

---

### Regulatory/Compliance

**Modified UI/UX:**
- Compliance dashboard mapping studies to NERC reliability standards (TPL, FAC, PRC, MOD) or regional grid codes
- Interconnection-study tracker with milestone status for FERC/ISO queue management
- Audit-ready documentation browser with indexed study packages and evidence cross-references

**Feature Gating:**
- Full analysis capabilities with emphasis on NERC/FERC compliance study documentation
- Automated compliance checking against TPL-001 (transmission planning), PRC-025 (relay loadability), FAC-001 standards
- Version-controlled study packages with electronic review and approval workflows

**Pricing Tier:** Enterprise tier with Compliance module

**Onboarding Flow:**
- Regulatory-profile setup: select applicable standards (NERC, regional grid code, IEEE 1547, IEC) to configure study templates
- Configure interconnection-study templates for generator/load connection requests
- Demo compliance workflow: standard requirement > study plan > analysis execution > report > regulatory filing

**Key Workflows:**
- Performing NERC TPL-001 transmission-planning assessments with required contingency analysis
- Conducting generator-interconnection studies (feasibility, system-impact, facilities studies) per FERC procedures
- Running PRC-025 relay-loadability verification to ensure protective relays do not trip during stable swings
- Generating audit-ready compliance documentation packages for NERC/regional-entity reviews
- Managing interconnection queue positions and tracking milestone compliance for renewable energy projects

---

### Manager/Decision-maker

**Modified UI/UX:**
- Portfolio dashboard showing grid-reliability metrics, capital-project status, and regulatory-compliance health across service territory
- Investment-planning view with cost-benefit analysis for system reinforcement alternatives
- Risk heat map overlaying thermal, voltage, and stability risks on geographic one-line diagram

**Feature Gating:**
- Read-only access to all system models and study results with annotation and approval tools
- Full planning analytics, investment-optimization, and regulatory-milestone tracking
- No direct model editing or analysis execution; all requests routed through engineering team

**Pricing Tier:** Enterprise tier (manager seat)

**Onboarding Flow:**
- Executive dashboard setup: connect to asset management, outage management, and financial systems
- Configure reliability KPIs (SAIDI, SAIFI, CAIDI) and link to planning-study outputs
- Walkthrough of investment-planning tools and cost-benefit analysis frameworks

**Key Workflows:**
- Reviewing system reliability assessments and prioritizing capital investments for grid reinforcement
- Evaluating interconnection requests and their system impact from a cost-allocation and schedule perspective
- Monitoring regulatory-compliance status and managing remediation plans for identified deficiencies
- Approving transmission and distribution planning studies and capital project authorizations
- Communicating grid-reliability posture and investment needs to executive leadership and regulators
