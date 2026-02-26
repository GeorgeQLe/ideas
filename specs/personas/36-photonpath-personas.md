# PhotonPath — Persona Subspecs

> Parent spec: [`specs/36-photonpath.md`](../36-photonpath.md)

---

## Experience-Level Personas

### Student/Academic
**Modified UI/UX:**
- Guided simulation mode with step-by-step setup: draw waveguide geometry, assign materials, set source and monitors, run FDTD — each step with annotated diagrams explaining the physics
- Simplified 2D cross-section viewer with auto-generated mode profiles and labeled effective indices
- Pre-configured material library with common photonic materials (Si, SiN, SiO2, InP) and educational notes on dispersion properties

**Feature Gating:**
- Access to 2D FDTD simulations and eigenmode solver for waveguide analysis; 3D FDTD limited to small volumes (< 50 um^3)
- PIC layout tools limited to basic waveguide routing and standard component library
- Single simulation run at a time; parameter sweeps disabled

**Pricing Tier:** Free tier (academic email verification)

**Onboarding Flow:**
- Tutorial: simulate a silicon strip waveguide mode profile, sweep width, and plot effective index vs. geometry in under 15 minutes
- Pre-loaded example components (ring resonator, directional coupler, MMI splitter) with validated simulation setups
- Prompt to join a research group workspace if an advisor's invite code is available

**Key Workflows:**
- Simulate waveguide mode profiles and compute effective indices for photonics coursework
- Model basic photonic components (ring resonators, couplers, gratings) and extract S-parameters
- Explore how geometry variations affect optical performance through manual parameter changes
- Export simulation results and field plots for course assignments and lab reports
- Validate simulation results against analytical solutions for standard waveguide geometries

---

### Junior Engineer (0-3 years)
**Modified UI/UX:**
- Component-centric workspace: select from a library of standard PIC building blocks, configure dimensions, simulate, and verify
- Contextual guidance on mesh density, boundary conditions, and source placement to avoid common simulation pitfalls
- Side-by-side comparison view for evaluating component variants across wavelength and geometry sweeps

**Feature Gating:**
- Full 2D and 3D FDTD with mesh volumes up to 500 um^3; eigenmode solver for arbitrary cross-sections
- PIC layout with standard PDK component library and DRC checking; GDS export enabled
- Up to 5 concurrent simulations; wavelength and geometry parameter sweeps enabled

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Import a waveguide cross-section from a foundry PDK and simulate its mode characteristics
- Walk through designing a ring resonator filter: set dimensions, simulate transmission, optimize coupling gap
- Connect to team workspace and browse the shared component library

**Key Workflows:**
- Design and optimize standard PIC components (filters, splitters, modulators) using foundry PDK constraints
- Run wavelength sweeps and extract S-parameters for circuit-level simulation
- Verify component designs against foundry DRC rules before tapeout submission
- Compare design variants systematically using parameter sweeps and performance metrics
- Document component specifications and simulation validation for design reviews

---

### Senior/Expert
**Modified UI/UX:**
- Multi-pane workspace with concurrent FDTD simulation, layout editing, circuit simulation, and scripting views
- Custom material model editor for defining anisotropic, nonlinear, and gain materials
- Python scripting console for automated optimization, inverse design, and batch simulation management

**Feature Gating:**
- All features unlocked: unlimited 3D FDTD, inverse design optimizer, nonlinear optics, GPU cluster burst, API access
- Full PIC layout suite with custom component creation, hierarchical design, and multi-project wafer submission
- Admin controls for team PDK management, component libraries, and tapeout coordination

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Import existing Lumerical or MEEP simulation projects to migrate work to PhotonPath
- API integration walkthrough for connecting to EDA tools and foundry submission portals
- Configure team PDK library with foundry-specific layer stacks, DRC rules, and process corners

**Key Workflows:**
- Develop novel photonic components using inverse design and topology optimization algorithms
- Run large-scale optimization campaigns with GPU-accelerated FDTD across parameter spaces
- Architect complete PIC layouts from component design through placement and routing to tapeout GDS
- Manage team-wide PDK compliance and maintain validated component libraries
- Lead tape-out reviews and coordinate multi-project wafer submissions with foundry partners

---

## Role-Based Personas

### Designer/Creator
**Modified UI/UX:**
- PIC layout canvas as primary workspace with hierarchical cell editing, waveguide routing, and component placement
- Component palette with visual previews of S-parameter responses and mode profiles for quick selection
- Real-time DRC overlay showing violations as components are placed and routes are drawn

**Feature Gating:**
- Full layout editor with all routing tools, component library, and hierarchical design capabilities
- Custom component design using parameterized geometry scripts
- Export to GDS-II, OASIS, and foundry-specific submission formats

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Create a simple PIC: place laser source, route waveguide, add ring filter, connect to photodetector pad
- Walk through DRC checking and fixing common violations (minimum spacing, bend radius)
- Tutorial on creating parameterized custom components for the team library

**Key Workflows:**
- Design complete photonic integrated circuits from schematic through physical layout
- Create and maintain parameterized photonic component cells for design reuse
- Route optical waveguides with automatic bend optimization and crossing minimization
- Verify designs against foundry DRC rules and generate tapeout-ready GDS files
- Iterate between component-level FDTD optimization and chip-level layout integration

---

### Analyst/Simulator
**Modified UI/UX:**
- Simulation manager as primary view with job queue, convergence monitoring, and result browser
- Post-processing workspace with field visualization, spectral analysis, and S-parameter extraction tools
- Mesh quality and convergence analysis panel for verifying simulation accuracy

**Feature Gating:**
- Full access to all simulation engines: 2D/3D FDTD, eigenmode, EME, circuit simulation
- Advanced post-processing: far-field computation, mode overlap integrals, group delay analysis
- Batch simulation campaigns with automated parameter extraction and statistical analysis

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Run a sample 3D FDTD simulation and walk through convergence checking and result validation
- Configure post-processing scripts for automated S-parameter extraction across a wavelength sweep
- Set up a parameter sweep study with automated result aggregation

**Key Workflows:**
- Run and manage large-scale FDTD simulation campaigns across geometry and wavelength parameter spaces
- Extract S-parameters, mode profiles, and spectral responses from simulation results
- Validate simulation accuracy through mesh convergence studies and comparison to analytical solutions
- Perform tolerance analysis to predict component performance under fabrication variations
- Generate simulation reports with field plots, spectral responses, and performance metrics

---

### Manufacturing/Process
**Modified UI/UX:**
- Fabrication-aware design review panel showing layer stack cross-sections and process tolerance overlays
- Yield prediction dashboard showing how lithography variations and etch non-uniformity affect component performance
- Wafer map view showing die layout, process monitors, and test structure placement

**Feature Gating:**
- Access to process variation analysis tools and yield prediction models
- DRC and LVS (layout vs. schematic) verification tools with foundry-specific rules
- Layout editing restricted to approved PDK components; custom component creation requires designer approval

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Configure foundry process stack with layer definitions, design rules, and process variation statistics
- Walk through a process tolerance study: simulate component across +/- 10nm width variation
- Set up wafer-level yield prediction linking process monitors to component performance

**Key Workflows:**
- Analyze how fabrication process variations (linewidth, etch depth, layer thickness) affect device performance
- Run DRC and LVS checks to ensure designs comply with foundry manufacturing rules
- Predict wafer-level yield based on process statistics and component sensitivity
- Coordinate tapeout preparation including test structures, process control monitors, and alignment marks
- Track fabrication run results and correlate measured performance back to simulation predictions

---

### Regulatory/Compliance
**Modified UI/UX:**
- IP and export control dashboard tracking design components against licensing agreements and export restrictions
- Design provenance viewer showing origin of every component (foundry PDK, internally designed, third-party IP)
- Compliance checklist for tapeout submission requirements (NDA status, PDK license, export classification)

**Feature Gating:**
- Read access to all design files and simulation results for review purposes
- Write access to compliance annotations, IP classification tags, and export control markings
- Audit trail with immutable records of design access, modifications, and tapeout submissions

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Configure IP management policies: classify components by source, license type, and export control status
- Walk through tapeout compliance review: verify PDK license, check export restrictions, approve submission
- Set up access control policies aligned with foundry NDA and IP protection requirements

**Key Workflows:**
- Review PIC designs for IP compliance, ensuring all components are properly licensed and attributed
- Verify export control classification for photonic designs containing controlled technology
- Manage foundry NDA compliance and track PDK license status across team members
- Approve tapeout submissions after verifying all IP, export, and licensing requirements are satisfied
- Maintain records of all design submissions, foundry interactions, and IP agreements

---

### Manager/Decision-maker
**Modified UI/UX:**
- Tapeout program dashboard showing chip design progress, simulation milestones, and foundry submission deadlines
- Cost tracking panel: simulation compute spend, foundry tapeout costs, and multi-project wafer allocations
- Team utilization view showing engineer workload, license usage, and simulation queue depth

**Feature Gating:**
- Read-only access to program dashboards, design status, and cost summaries; no direct simulation or layout tools
- Team management: license allocation, project creation, compute and tapeout budget controls
- Approval authority for tapeout submissions and foundry purchase orders

**Pricing Tier:** Enterprise tier (included in admin seat)

**Onboarding Flow:**
- Program dashboard overview: active PIC design projects, tapeout schedule, budget tracking
- Configure program milestones aligned with foundry shuttle schedules and product timelines
- Set up approval workflows for tapeout submissions and budget expenditures

**Key Workflows:**
- Track PIC design programs from concept through simulation through tapeout through characterization
- Manage tapeout schedules and coordinate multi-project wafer submissions to optimize foundry costs
- Allocate compute and engineering resources across competing design projects
- Review program risk (tapeout readiness, simulation confidence, foundry schedule alignment) and make go/no-go decisions
- Report design program status and technology competitiveness to executive leadership
