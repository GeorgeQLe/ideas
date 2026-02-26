# SpiceSim — Persona Subspecs

> Parent spec: [`specs/43-spicesim.md`](../43-spicesim.md)

---

## Experience-Level Personas

### Student/Academic

**Modified UI/UX:**
- Simplified schematic editor with curated component library (resistors, capacitors, op-amps, diodes, transistors) and drag-and-drop placement
- Inline theory tooltips explaining circuit concepts (e.g., "This RC network has a time constant of...") as components are connected
- Waveform viewer with guided interpretation: auto-annotated gain, bandwidth, phase margin on plots

**Feature Gating:**
- DC operating point, transient, AC sweep, and basic DC sweep analyses available
- Monte Carlo, worst-case, and corner analysis locked
- Component library limited to standard educational parts; foundry PDKs and IBIS models unavailable

**Pricing Tier:** Free tier (education license with .edu verification)

**Onboarding Flow:**
- "Build Your First Circuit" tutorial: construct an inverting op-amp amplifier, simulate gain vs. frequency, and compare to theory
- Pre-loaded lab assignments mapped to common EE curricula (filters, amplifiers, oscillators, power supplies)
- Integrated formula reference panel showing key equations alongside simulation results

**Key Workflows:**
- Drawing schematics for homework problems and verifying results against hand calculations
- Running AC analysis on filter circuits and measuring cutoff frequencies and roll-off slopes
- Performing transient simulations of switching circuits (555 timer, H-bridge) and observing waveforms
- Exporting simulation plots for inclusion in lab reports and presentations

---

### Junior Engineer (0-3 years)

**Modified UI/UX:**
- Standard professional schematic editor with hierarchical design, bus notation, and net labels
- Parameter-sweep wizard that guides setup of multi-variable analyses with automatic plot generation
- Design-rule-check panel highlighting common errors (floating nodes, shorted supplies, missing bypass caps)

**Feature Gating:**
- All standard analyses unlocked: DC, AC, transient, noise, pole-zero, sensitivity
- Monte Carlo with up to 100 runs available; full statistical analysis (1000+ runs) restricted
- IBIS model import and basic signal-integrity checks available; full EM co-simulation locked

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Import existing schematics from LTspice or other SPICE formats with automatic component mapping and netlist validation
- Guided "Analysis Cookbook" with templates for common tasks (bias-point verification, stability analysis, noise budgeting)
- Tool-library setup: configure preferred component vendors and link to distributor parametric search

**Key Workflows:**
- Designing and simulating analog front-end circuits (amplifiers, filters, ADC drivers)
- Running parametric sweeps to optimize component values for gain, bandwidth, and noise targets
- Performing bias-point verification and DC operating-point analysis across temperature range
- Generating Bode plots and verifying loop stability with gain/phase margin measurements
- Creating simulation test benches and archiving results for design-review documentation

---

### Senior/Expert

**Modified UI/UX:**
- Multi-tab workspace with schematic, netlist editor, waveform viewer, and scripting console simultaneously visible
- Custom analysis scripting with full SPICE directive access and post-processing expression language
- Corner/process-variation dashboard showing worst-case results across PVT (process, voltage, temperature) space

**Feature Gating:**
- All features unlocked: Monte Carlo (unlimited runs), worst-case analysis, EM co-simulation, IBIS-AMI channel simulation
- Foundry PDK integration with encrypted model support and process-corner libraries
- API/scripting access for batch simulation, regression testing, and CI/CD integration

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- PDK installation wizard: import foundry process kits with encrypted models, design rules, and characterized cells
- Migration of simulation decks and measurement scripts from Cadence Spectre/ADS
- Configure regression-test infrastructure for automated nightly simulation runs against golden results

**Key Workflows:**
- Designing precision analog circuits (bandgap references, PLLs, data converters) with foundry PDK models
- Running comprehensive PVT corner and Monte Carlo yield analysis across process variations
- Performing noise analysis and optimizing signal chain from sensor through ADC
- Creating automated regression test suites that validate circuit performance against specifications
- Developing custom measurement scripts for complex figures of merit (jitter, ENOB, PSRR)

---

## Role-Based Personas

### Designer/Creator

**Modified UI/UX:**
- Schematic-centric workspace with visual emphasis on clean layout, hierarchical block diagrams, and aesthetic net routing
- Symbol editor for creating custom component symbols with pin-mapping to SPICE models
- Quick-prototype mode for rapidly sketching circuit topologies and running "what-if" simulations

**Feature Gating:**
- Full schematic capture with hierarchical design, bus structures, and parameterized subcircuits
- Symbol and model library creation tools unlocked
- Statistical analysis available but secondary; design exploration tools (sweeps, optimization) emphasized

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- "Design Style" setup: configure schematic preferences (grid spacing, wire style, annotation format, color themes)
- Import personal component libraries and schematic templates from previous tools
- Quick tutorial on hierarchical design methodology and reusable subcircuit creation

**Key Workflows:**
- Architecting circuit topologies using hierarchical block diagrams with defined interfaces
- Creating parameterized subcircuit blocks for reuse across projects (power modules, amplifier stages)
- Rapidly iterating on component values with interactive parameter sliders and live waveform updates
- Building custom symbol libraries with vendor-specific pinouts and simulation models
- Generating clean schematic PDFs for design reviews and documentation packages

---

### Analyst/Simulator

**Modified UI/UX:**
- Analysis-centric layout with waveform viewer maximized, analysis-setup panel docked, and measurement toolbox pinned
- Multi-analysis comparison view: overlay results from different corners, temperatures, and component variations
- Convergence and solver-status panel showing iteration count, timestep adaptation, and error metrics

**Feature Gating:**
- All analysis types unlocked with full control over solver settings (tolerances, timestep limits, integration methods)
- Advanced post-processing: FFT, histogram, eye-diagram, bathtub curves, and custom expressions
- Schematic editing available but positioned as read-only import; analysis configuration is primary

**Pricing Tier:** Professional or Enterprise tier

**Onboarding Flow:**
- Solver-benchmark suite: run canonical circuits and compare accuracy/performance against known results
- Configure preferred analysis defaults (transient max-timestep, AC points-per-decade, Monte Carlo seed strategy)
- Tutorial on advanced post-processing: writing measurement expressions, generating eye diagrams, extracting S-parameters

**Key Workflows:**
- Configuring and running comprehensive analysis plans (DC, AC, transient, noise, stability) for circuit sign-off
- Performing Monte Carlo yield analysis and identifying dominant variation sources for yield improvement
- Analyzing power-supply rejection, common-mode rejection, and input-referred noise across frequency
- Extracting S-parameters and impedance profiles for signal-integrity analysis at board level
- Documenting simulation methodology, results, and pass/fail criteria in formal analysis reports

---

### Manufacturing/Process

**Modified UI/UX:**
- Yield-focused dashboard showing Monte Carlo distribution histograms for every key spec with pass/fail boundaries
- Component-sensitivity ranking panel highlighting which parts contribute most to yield loss
- BOM-aware simulation: link component values to vendor part numbers with tolerance and availability data

**Feature Gating:**
- Monte Carlo, worst-case, and sensitivity analysis fully unlocked
- BOM integration with distributor APIs for real-time component availability and cost
- Component-substitution analysis: "what if" we replace this 1% resistor with a 5%?

**Pricing Tier:** Professional tier with Manufacturing add-on

**Onboarding Flow:**
- BOM import and mapping: link schematic components to vendor part numbers with tolerance specifications
- Configure yield targets and specification limits for automated pass/fail evaluation
- Tutorial on sensitivity analysis for identifying cost-reduction opportunities through tolerance relaxation

**Key Workflows:**
- Running Monte Carlo yield analysis to predict manufacturing pass rates before production
- Identifying and ranking component sensitivities to guide tolerance tightening or relaxation decisions
- Evaluating component substitutions (alternate vendors, relaxed tolerances) for cost reduction without yield impact
- Generating yield-prediction reports for production readiness reviews
- Correlating simulation yield predictions with actual production test data for model validation

---

### Regulatory/Compliance

**Modified UI/UX:**
- Compliance dashboard mapping simulation results to EMC/safety standards (FCC Part 15, IEC 61000, UL 60950)
- Audit-trail panel with timestamped records of every simulation run, parameter change, and result
- Specification-compliance table: each design requirement linked to simulation evidence with pass/fail status

**Feature Gating:**
- Full simulation capabilities with emphasis on EMC-relevant analyses (conducted emissions, susceptibility)
- Automated compliance-check rules engine for common standards (radiated emissions limits, ESD immunity)
- Electronic signatures on simulation reports for regulatory submission

**Pricing Tier:** Enterprise tier with Compliance module

**Onboarding Flow:**
- Regulatory profile setup: select applicable standards (FCC, CE, IEC, MIL-STD, automotive EMC) to load limit templates
- Configure specification-compliance matrix linking requirements to simulation test benches
- Demo audit-trail workflow from initial design through simulation evidence to regulatory filing

**Key Workflows:**
- Simulating conducted EMI from switching regulators and verifying compliance with FCC/CISPR limits
- Running worst-case analysis to ensure circuit meets specifications across full operating temperature range
- Generating simulation-evidence packages for regulatory submissions (safety agency filings, automotive PPAP)
- Maintaining full traceability from design requirements through simulation methodology to pass/fail results
- Archiving validated simulation configurations for reproducibility during audits

---

### Manager/Decision-maker

**Modified UI/UX:**
- Project dashboard showing simulation coverage (% of specs verified), yield predictions, and schedule status
- Risk matrix highlighting specifications with thin margins or incomplete simulation coverage
- Cost-impact view showing how design choices (component tolerances, architecture) affect unit cost

**Feature Gating:**
- Read-only access to schematics and simulation results with commenting and approval tools
- Full project analytics, resource utilization, and license-usage reporting
- No direct simulation configuration; all changes go through engineering team

**Pricing Tier:** Enterprise tier (manager seat)

**Onboarding Flow:**
- Dashboard configuration: connect to project management tools and define key milestones and KPIs
- Walkthrough of simulation-coverage reports and how to interpret yield predictions
- Configure alerting rules for margin violations, missed milestones, and specification non-compliance

**Key Workflows:**
- Reviewing project-level simulation coverage and identifying specs that lack verification evidence
- Evaluating yield predictions and make/buy cost trade-offs for production planning
- Approving design releases based on simulation evidence and compliance status
- Tracking engineering resource utilization and simulation license allocation
- Using margin and yield data to inform product-positioning and pricing decisions
