# AntennaSynth — Persona Subspecs

> Parent spec: [`specs/93-antennasynth.md`](../93-antennasynth.md)

---

## Experience-Level Personas

### Student/Academic

**Modified UI/UX:**
- Simplified workspace with common antenna type gallery (dipole, patch, horn, array) and parameter sliders
- Real-time radiation pattern preview as dimensions are adjusted
- Integrated electromagnetic theory reference linking design parameters to performance metrics

**Feature Gating:**
- Single-element antenna design and basic linear array synthesis
- Method of Moments (MoM) solver for wire antennas; FDTD/FEM solvers locked
- Export limited to radiation patterns and S-parameter plots; no manufacturing file export

**Pricing Tier:** Free tier

**Onboarding Flow:**
- Tutorial: design a half-wave dipole, visualize radiation pattern, compute gain and impedance
- Pre-loaded example library (patch antenna for Wi-Fi, Yagi for VHF, horn for X-band)
- Educational mode with step-by-step derivation annotations alongside simulation results

**Key Workflows:**
- Design and simulate canonical antenna types from textbook specifications
- Study the effect of ground plane size, feed position, and substrate permittivity on patch antennas
- Synthesize uniform and Chebyshev linear arrays and analyze sidelobe levels
- Generate lab reports with annotated radiation patterns and impedance plots

---

### Junior Engineer (0-3 years)

**Modified UI/UX:**
- Specification-driven design wizard: input frequency band, gain, bandwidth, and polarization requirements
- Recommended antenna topology suggestions based on input specifications
- Side-by-side comparison of candidate designs ranked by performance compliance

**Feature Gating:**
- Full single-element and planar array design tools
- MoM and FDTD solvers enabled; large-scale FEM and asymptotic solvers require approval
- Matching network synthesis and basic radome modeling tools available

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Requirements-first project setup: enter target specs and get suggested antenna topologies
- 20-minute guided tutorial designing a 5G mmWave patch array from spec to layout
- Firm antenna library connection with previously validated designs as starting points

**Key Workflows:**
- Design microstrip patch antennas for 5G sub-6 GHz and mmWave bands
- Synthesize planar phased arrays with beam scanning and sidelobe control
- Perform impedance matching network design for multi-band operation
- Analyze antenna-on-platform effects with simplified platform models
- Generate antenna datasheets with pattern cuts, gain tables, and VSWR plots

---

### Senior/Expert

**Modified UI/UX:**
- Full parametric design environment with scripting console and optimization engine controls
- Multi-physics workspace combining EM simulation with thermal and structural analysis
- Custom solver configuration: meshing controls, boundary conditions, GPU acceleration settings

**Feature Gating:**
- All solvers unlocked: MoM, FDTD, FEM, PO/GO hybrid, MLFMM for large platforms
- Optimization algorithms: genetic, particle swarm, gradient-based, and multi-objective Pareto
- API access for automated design loops, manufacturing tolerance studies, and CI/CD integration

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Migration assistant for Antenna Magus, CST, and HFSS design import
- Solver benchmark suite for validating accuracy against published measurement data
- Scripting environment setup with sample parametric optimization workflows

**Key Workflows:**
- Design conformal phased arrays for satellite communication with full polarization control
- Run multi-objective optimization balancing gain, bandwidth, cross-pol, and scan range
- Perform installed performance analysis on full vehicle/aircraft platforms using hybrid methods
- Conduct manufacturing tolerance Monte Carlo analysis for yield prediction
- Develop reusable parametric antenna templates for product family standardization

---

## Role-Based Personas

### Designer/Creator

**Modified UI/UX:**
- Visual antenna geometry editor with parametric dimensioning and constraint-driven layout
- Feed network schematic editor integrated with EM simulation for co-design
- Material and substrate library browser with vendor-specific dielectric data

**Feature Gating:**
- Full geometry creation, feed network design, and matching circuit synthesis
- Radome design and analysis tools for enclosed antenna systems
- PCB/LTCC layout export with manufacturing rule checks

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Topology selector wizard: application domain, frequency, and form factor constraints
- Quick-start from parametric template with tunable dimensions and feed configurations
- CAD integration setup for mechanical housing co-design (SolidWorks, CATIA, NX)

**Key Workflows:**
- Design custom antenna geometries for specific platform integration constraints
- Create feed networks for corporate-fed and series-fed array architectures
- Design wideband matching networks for multi-band or ultra-wideband antennas
- Develop radome-integrated antenna systems with compensation for radome effects
- Export manufacturing-ready designs: Gerber files, 3D print STL, or CNC toolpaths

---

### Analyst/Simulator

**Modified UI/UX:**
- Solver-centric workspace with mesh visualization, convergence tracking, and resource monitoring
- Multi-configuration batch setup for parameter sweeps and frequency scans
- Advanced post-processing: near-field to far-field transforms, SAR computation, coupling matrices

**Feature Gating:**
- All EM solvers with full configuration access and GPU acceleration
- Near-field measurement data import for pattern comparison and diagnostics
- EMC/EMI analysis tools for co-site interference and coupling prediction

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Solver selection guide based on antenna type, electrical size, and accuracy requirements
- Mesh convergence study tutorial with automated refinement recommendations
- Validation workflow comparing simulation to measurement data from antenna ranges

**Key Workflows:**
- Perform full-wave simulation of electrically large antenna-platform systems
- Analyze mutual coupling in dense array environments and its impact on scan blindness
- Compute SAR distributions for antennas near human tissue (wearable, handset applications)
- Run installed antenna performance analysis on vehicles, aircraft, and ships
- Diagnose antenna performance issues by correlating simulation with measurement data

---

### Manufacturing/Process

**Modified UI/UX:**
- Design-for-manufacturing (DFM) checker highlighting PCB rule violations, tolerances, and material availability
- Bill of materials generator with component sourcing links and cost estimation
- Test fixture design assistant for antenna range and production-line measurements

**Feature Gating:**
- Manufacturing file export: Gerber, ODB++, STEP, and test specification documents
- Tolerance analysis module for yield prediction under manufacturing variability
- Production test criteria generator: pass/fail limits for pattern, gain, and VSWR

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Manufacturing capability input: PCB stackup options, etching tolerances, assembly constraints
- DFM rule configuration for target fabrication process
- Test equipment integration setup for automated measurement data import

**Key Workflows:**
- Validate antenna designs against PCB fabrication and assembly constraints
- Generate manufacturing documentation: drawings, stackup specifications, assembly instructions
- Define production acceptance test procedures with pass/fail criteria
- Analyze manufacturing tolerance impact on antenna performance (Monte Carlo)
- Track production yield data and correlate failures with process variations

---

### Regulatory/Compliance

**Modified UI/UX:**
- Regulatory compliance dashboard mapping antenna performance to FCC, ETSI, ITU emission limits
- Spurious emission and out-of-band rejection analysis tools
- EIRP calculator with link budget integration for regulatory filings

**Feature Gating:**
- EMC compliance simulation: spurious emissions, harmonics, and intermodulation products
- SAR computation for FCC/ICNIRP human exposure compliance
- Regulatory filing report generators for FCC Equipment Authorization and CE marking

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Regulatory jurisdiction selector loading applicable emission masks and exposure limits
- Standard compliance checklist auto-generated from antenna operating parameters
- Pre-configured report templates for FCC, ETSI, and Industry Canada submissions

**Key Workflows:**
- Verify EIRP compliance against regulatory emission masks across operating bands
- Compute SAR values for portable devices and demonstrate human exposure compliance
- Analyze co-site interference and spectrum sharing compliance for shared platforms
- Generate regulatory submission documentation with required test data and analysis
- Track design changes and re-evaluate compliance impact for certification maintenance

---

### Manager/Decision-maker

**Modified UI/UX:**
- Portfolio dashboard showing antenna product development status, performance vs. spec, and schedule
- Technology trade study comparisons: topology options ranked by cost, performance, and risk
- Resource utilization and simulation compute cost tracking

**Feature Gating:**
- Read-only access to all designs, simulation results, and compliance reports
- Cost-performance trade-off analysis and what-if scenario tools
- Project milestone tracking and design review workflow management

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Product portfolio import with specification targets and development timelines
- Dashboard configuration for key metrics: spec compliance, compute costs, schedule adherence
- Team setup with role assignments and design review gate definitions

**Key Workflows:**
- Review antenna design alternatives and select optimal topology based on cost-performance trade-offs
- Track development progress against specification milestones and schedule
- Evaluate make-vs-buy decisions for antenna subsystems using performance and cost data
- Allocate simulation compute resources across concurrent antenna development programs
- Generate executive summaries for design reviews and program status updates
