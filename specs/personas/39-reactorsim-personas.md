# ReactorSim — Persona Subspecs

> Parent spec: [`specs/39-reactorsim.md`](../39-reactorsim.md)

---

## Experience-Level Personas

### Student/Academic
**Modified UI/UX:**
- Guided reactor model builder with visual cross-section editor: place fuel pins, moderator, and structural materials using a color-coded grid with inline physics annotations
- Simplified results viewer showing pin-power maps, neutron flux profiles, and k-effective convergence with labeled educational overlays
- Interactive sliders for enrichment, moderator density, and control rod position showing real-time reactivity feedback

**Feature Gating:**
- Access to deterministic Sn transport solver for 2D assembly-level models; Monte Carlo limited to simplified pin-cell geometries
- Pre-loaded cross-section libraries for standard reactor types (PWR, BWR, CANDU); custom library generation disabled
- Single simulation at a time; burnup depletion and coupled thermal-hydraulics disabled

**Pricing Tier:** Free tier (academic email verification)

**Onboarding Flow:**
- Tutorial: build a 17x17 PWR fuel assembly, set enrichment, compute k-infinity and pin-power distribution
- Pre-loaded example models (PWR assembly, TRIGA research reactor, fast spectrum assembly) with validated benchmark results
- Prompt to join a course workspace if a professor's invite code is available

**Key Workflows:**
- Build and analyze fuel assembly models for reactor physics coursework assignments
- Compute multiplication factors and neutron flux distributions for standard reactor configurations
- Explore reactivity effects of enrichment, moderator density, and control rod insertion
- Compare simulation results against published benchmark solutions for validation exercises
- Export flux maps and power distributions for homework and thesis figures

---

### Junior Engineer (0-3 years)
**Modified UI/UX:**
- Assembly and core model templates: select reactor type (PWR, BWR, SMR), configure geometry, and auto-populate material and cross-section data
- Contextual guidance on simulation setup: mesh density recommendations, convergence criteria, and source particle counts for Monte Carlo
- Dashboard showing key reactor physics metrics (k-eff, peak-to-average power, control rod worth, temperature coefficients)

**Feature Gating:**
- Full deterministic and Monte Carlo transport solvers for assembly and core-level models
- Burnup/depletion with standard depletion chains; coupled thermal-hydraulics subchannel analysis enabled
- Up to 5 concurrent simulations; parameter sweeps up to 20 points per variable

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Build a quarter-core PWR model from a template, load a burnup-dependent cross-section library, and run a depletion cycle
- Walk through thermal-hydraulic coupling: compute coolant temperature and fuel temperature distributions
- Connect to team workspace and browse the shared model and cross-section library

**Key Workflows:**
- Build core-level reactor models and perform steady-state neutronics analysis with thermal-hydraulic feedback
- Run depletion calculations to predict fuel burnup, isotopic evolution, and reactivity swing over a cycle
- Compute control rod worth, temperature coefficients, and shutdown margin for safety analysis
- Generate formatted nuclear design reports with pin-power maps and core physics parameters
- Validate simulation results against startup test data or benchmark solutions

---

### Senior/Expert
**Modified UI/UX:**
- Multi-physics workspace with coordinated neutronics, thermal-hydraulics, fuel performance, and kinetics views
- Custom cross-section processing pipeline with resonance self-shielding and homogenization controls
- Python scripting console for automated core loading pattern optimization, batch depletion studies, and custom tally specifications

**Feature Gating:**
- All features unlocked: GPU-accelerated Monte Carlo, full core depletion, transient kinetics, multi-physics coupling, API access
- Unlimited simulations and parameter sweeps; priority GPU queue for Monte Carlo convergence
- Admin controls for cross-section libraries, model validation databases, and team quality assurance workflows

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Import existing MCNP/Serpent/OpenMC input files to migrate simulation workflows
- API integration walkthrough for connecting to fuel management systems and regulatory document generators
- Configure team cross-section library management with version control and validation pedigree tracking

**Key Workflows:**
- Develop and validate reactor core designs using GPU-accelerated Monte Carlo with full multi-physics coupling
- Optimize fuel loading patterns for cycle length, power peaking, and fuel utilization using automated search algorithms
- Run transient and accident scenario simulations (LOCA, RIA, ATWS) with coupled neutronics/thermal-hydraulics
- Manage and curate validated cross-section libraries and benchmark model suites for the team
- Lead nuclear design reviews and provide technical oversight on junior engineers' simulation work

---

## Role-Based Personas

### Designer/Creator
**Modified UI/UX:**
- Core layout canvas with drag-and-drop fuel assembly placement, control rod bank configuration, and reflector arrangement
- Loading pattern optimizer showing color-coded power peaking maps that update as assemblies are rearranged
- Fuel assembly design studio with pin-level geometry editing, enrichment zoning, and burnable absorber placement

**Feature Gating:**
- Full core and assembly design tools with all geometry types and material options
- Loading pattern optimization with objective functions (minimize peaking, maximize cycle length)
- Export designs in standardized reactor physics formats for cross-tool compatibility

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Design a core loading pattern: place fresh and burned assemblies, configure control rod banks, evaluate power distribution
- Walk through fuel assembly design: define pin geometry, enrichment zones, and burnable absorber patterns
- Save designs as templates for team reuse and iterative refinement

**Key Workflows:**
- Design reactor core loading patterns optimizing for cycle length, peaking factors, and fuel economy
- Create and refine fuel assembly designs with enrichment zoning and burnable absorber layouts
- Evaluate alternative core designs through rapid simulation and comparison of key physics parameters
- Develop parameterized design templates for systematic exploration of core configuration options
- Coordinate with thermal-hydraulic analysts by providing neutronics boundary conditions from core designs

---

### Analyst/Simulator
**Modified UI/UX:**
- Simulation management console with job queue, convergence monitoring, tally statistics, and result aggregation
- Advanced post-processing: pin-by-pin power reconstruction, spectral analysis, isotopic inventory tracking
- Benchmark validation panel for systematic comparison of results against published experimental data

**Feature Gating:**
- Full access to all transport solvers, depletion methods, and multi-physics coupling options
- Advanced tallying: energy-dependent flux spectra, reaction rate distributions, sensitivity coefficients
- Batch simulation campaigns with automated convergence checking and statistical analysis

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Run a Monte Carlo simulation and walk through convergence diagnostics: source entropy, tally statistics, k-eff convergence
- Configure a multi-cycle depletion study and set up automated result extraction
- Set up benchmark validation comparing simulation output against published experimental data

**Key Workflows:**
- Run and manage large-scale neutronics simulation campaigns with rigorous convergence verification
- Perform detailed depletion and isotopic analysis for fuel cycle and waste characterization studies
- Conduct sensitivity and uncertainty analysis to quantify confidence in reactor physics predictions
- Validate simulation methods against experimental benchmarks and document V&V pedigree
- Generate comprehensive nuclear analysis reports with pin-power maps, flux distributions, and safety parameters

---

### Manufacturing/Process
**Modified UI/UX:**
- Fuel fabrication data panel linking fuel assembly designs to manufacturing specifications (pellet dimensions, enrichment tolerances, cladding specs)
- Quality assurance dashboard tracking as-built fuel data and correlating with predicted performance
- Fuel management interface showing core inventory, burnup status, and reload scheduling

**Feature Gating:**
- Access to fuel specification extraction and manufacturing tolerance analysis tools
- Fuel inventory management and reload scheduling; burnup accounting from operational data
- Core design editing restricted to approved loading pattern modifications; new designs require engineer approval

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Connect fuel inventory database and map as-built fuel assembly data to reactor models
- Walk through reload planning: review available assemblies, evaluate candidate loading patterns, select optimal configuration
- Configure fuel accountability reports linking burnup predictions to operational measurements

**Key Workflows:**
- Extract manufacturing specifications from fuel assembly designs for procurement and fabrication
- Analyze how manufacturing tolerances in enrichment, pellet density, and dimensions affect predicted reactor performance
- Manage fuel inventory and track burnup status across operating cycles for reload planning
- Coordinate fuel shuffle operations using simulation-optimized loading patterns
- Generate fuel accountability and material balance reports for regulatory and operational records

---

### Regulatory/Compliance
**Modified UI/UX:**
- Safety analysis report (SAR) dashboard showing compliance status for NRC Chapter 4 (Nuclear Design) requirements
- V&V pedigree viewer tracing every simulation result to its validated method, cross-section library, and benchmark comparison
- Automated document generator producing NRC/IAEA-formatted safety analysis tables and figures

**Feature Gating:**
- Full access to all simulation results for review; write access to regulatory annotations and approval stamps
- Audit trail with immutable records of all simulation inputs, solver versions, library versions, and results
- Regulatory document templates for NRC, IAEA, and national regulator submission formats

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Configure applicable regulatory framework (NRC 10 CFR 50, IAEA SSR-2/1, national nuclear safety regulations)
- Walk through a Chapter 4 nuclear design review: verify safety parameters, confirm analysis methodology, check documentation completeness
- Set up independent verification workflows with required separation between designer and reviewer

**Key Workflows:**
- Review nuclear design analyses for compliance with NRC Standard Review Plan and applicable regulatory guides
- Verify that simulation methods have adequate V&V pedigree for the intended safety analysis application
- Ensure independent verification of safety-critical calculations with documented reviewer qualifications
- Generate and review safety analysis report sections (Chapter 4 Nuclear Design) for licensing submissions
- Track regulatory commitments, open items, and requests for additional information (RAIs) across licensing projects

---

### Manager/Decision-maker
**Modified UI/UX:**
- Reactor design program dashboard showing core design milestones, licensing submission timeline, and team workload
- Cost tracking: simulation compute spend, licensing consultant costs, and engineering labor allocation
- Risk matrix showing technical and regulatory risks across the reactor development program

**Feature Gating:**
- Read-only access to program dashboards, design summaries, and regulatory status; no direct simulation tools
- Team management: user provisioning, project assignment, compute budget allocation
- Approval authority for design milestones, regulatory submissions, and major technical decisions

**Pricing Tier:** Enterprise tier (included in admin seat)

**Onboarding Flow:**
- Program dashboard overview: core design status, licensing timeline, resource utilization, risk items
- Configure program milestones aligned with NRC licensing schedule and internal development gates
- Set up notification rules for safety parameter exceedances and licensing milestone deadlines

**Key Workflows:**
- Monitor reactor design program progress from core concept through detailed design through licensing
- Track regulatory submission timelines and manage interactions with NRC/IAEA review staff
- Allocate nuclear engineering staff and compute resources across competing design tasks
- Review and approve key technical decisions (fuel design selection, core loading strategy, safety analysis approach)
- Present reactor design program status, technical risks, and resource needs to executive leadership and investors
