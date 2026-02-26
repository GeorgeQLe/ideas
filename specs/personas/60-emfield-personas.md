# EMField — Persona Subspecs

> Parent spec: [`specs/60-emfield.md`](../60-emfield.md)

---

## Experience-Level Personas

### Student/Academic

**Modified UI/UX:**
- Simplified 2D/3D geometry editor with predefined antenna and waveguide templates (dipole, patch, horn, rectangular waveguide)
- Inline EM theory cards explaining Maxwell's equations, boundary conditions, and S-parameters with interactive field visualisations
- Results panel defaults to S11/VSWR, radiation pattern (2D polar plot), and electric field magnitude with annotated resonances

**Feature Gating:**
- Geometry limited to single structures under 2 wavelengths; FDTD solver only with max 5M cells
- Antenna library limited to canonical types (dipole, monopole, patch, slot); no array or periodic boundary tools
- Export to PDF and CSV; no ECAD or system-level simulator interop

**Pricing Tier:** Free tier (educational license)

**Onboarding Flow:**
- "First Antenna" tutorial: build a microstrip patch antenna, run FDTD simulation, plot return loss and radiation pattern
- Interactive Smith chart explorer: adjust antenna dimensions and watch impedance trajectory move in real time
- Sandbox with textbook problems (waveguide modes, cavity resonator, Yagi-Uda array)

**Key Workflows:**
- Design a half-wave dipole and verify the input impedance and radiation pattern against analytical predictions
- Simulate a rectangular waveguide and identify TE/TM mode cutoff frequencies
- Analyse a microstrip patch antenna and tune dimensions for resonance at a target frequency
- Generate lab report figures showing S-parameters, field distributions, and radiation patterns

---

### Junior Engineer (0-3 years)

**Modified UI/UX:**
- Standard workspace with application wizard: select domain (antenna, EMC/EMI, signal integrity, RF component) and frequency range
- Guided meshing assistant with automatic wavelength-based refinement and convergence indicators
- Warning badges on structures with insufficient mesh density, radiation boundary too close, or port configuration issues

**Feature Gating:**
- Full 3D FDTD and FEM solvers; mesh up to 50M cells; frequency and time domain
- Antenna array tools (linear, planar) with basic beam steering; no infinite array or Floquet port analysis
- Import from ECAD (Altium, Cadence) and MCAD (STEP); export S-parameter touchstone files and far-field data

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Application selector (5G antenna, PCB EMC, connector SI, RF filter) pre-loads frequency bands, standards, and solver defaults
- Walkthrough of a 5G mmWave patch array design: layout, simulate, optimise for bandwidth and scan range
- Prompted to import firm's substrate library, connector models, and standard test configurations

**Key Workflows:**
- Design a 5G mmWave antenna array for a mobile device and evaluate bandwidth and scan performance
- Perform EMC pre-compliance simulation of a PCB layout predicting radiated emissions vs. CISPR limits
- Simulate a microstrip filter and optimise passband/rejection performance
- Extract S-parameter models of connectors and transitions for signal integrity analysis
- Generate design review presentations with simulated antenna performance and EMC compliance predictions

---

### Senior/Expert

**Modified UI/UX:**
- Multi-solver workspace with FDTD, FEM, MoM, and asymptotic (PO/UTD) solvers accessible; scripting console (Python API)
- Custom solver configuration panel for hybrid methods (MoM-FDTD, FEM-BI), domain decomposition, and GPU acceleration
- Measurement correlation dashboard importing VNA, antenna range, and EMC chamber data for model validation

**Feature Gating:**
- Unlimited geometry and mesh size; all solver types including MoM, MLFMM, and characteristic mode analysis
- Infinite array / unit cell analysis with Floquet ports; periodic and waveguide-port excitations
- Full API, HPC/GPU cluster support, and integration with system simulators (ADS, AWR), ECAD, and antenna measurement systems

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Import existing HFSS/CST/FEKO projects with automated geometry, material, and port translation
- Configure solver preferences, GPU allocation, and cluster job scheduling
- Power-user preset opening scripted parametric optimisation workspace

**Key Workflows:**
- Design a conformal phased array on a curved platform using characteristic mode analysis for element placement guidance
- Perform installed antenna analysis on a full-vehicle model using hybrid MoM-PO/UTD method
- Develop a metamaterial/metasurface design with unit-cell simulation and infinite-array performance prediction
- Conduct large-scale EMC simulation of an entire electronic system (aircraft avionics bay, automotive ECU cluster)
- Automate multi-objective antenna optimisation (gain, bandwidth, efficiency, SAR) using surrogate-assisted evolutionary algorithms

---

## Role-Based Personas

### Designer/Creator

**Modified UI/UX:**
- Visual antenna design canvas with parametric geometry editors, substrate stackup manager, and feed network layout tools
- Antenna type gallery with performance envelopes (gain, bandwidth, polarisation, size) for rapid form-factor selection
- Real-time impedance and pattern preview during interactive geometry tuning

**Feature Gating:**
- Full 3D geometry creation and parametric editing tools enabled
- Solver runs in background for interactive design feedback; full-wave analysis requires explicit launch
- Export to ECAD (Gerber, ODB++), MCAD (STEP), and manufacturing-ready antenna specifications

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Specification-driven wizard: enter frequency, bandwidth, gain, polarisation, and form-factor constraints; receive ranked antenna candidates
- Tour of parametric antenna design tools using a sample wideband slot antenna
- Demo of substrate stackup editor for multilayer PCB antenna design

**Key Workflows:**
- Design a wideband antenna for IoT applications meeting size, bandwidth, and efficiency constraints
- Create a phased array feed network with corporate or series feed topology and optimised power distribution
- Develop a cavity-backed slot antenna for flush-mount aerospace applications
- Iterate antenna placement on a product enclosure evaluating proximity detuning and user-body effects
- Produce antenna manufacturing specifications with dimensional tolerances and material callouts

---

### Analyst/Simulator

**Modified UI/UX:**
- Analysis workspace with S-parameter matrices, Smith charts, current density maps, near-field probes, and CEM convergence monitors
- Multi-solver comparison panel contrasting FDTD vs. FEM vs. MoM solutions with mesh convergence data
- Post-processing suite for SAR computation, radar cross-section extraction, and near-to-far-field transformation

**Feature Gating:**
- All solver types and hybridisation options fully unlocked
- Advanced post-processing: characteristic mode analysis, multipole expansion, and antenna coupling analysis
- HPC job scheduling with GPU acceleration and adaptive mesh refinement

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Validation suite running canonical benchmarks (PEC sphere RCS, Vivaldi antenna, microstrip benchmark suite)
- Import reference measurements and configure automated model-measurement correlation
- Configure default mesh strategies, solver convergence criteria, and output recording preferences

**Key Workflows:**
- Perform mesh convergence study and validate antenna simulation against anechoic chamber measurements
- Conduct antenna-to-antenna coupling analysis on a multi-antenna platform (ship, aircraft, base station)
- Compute specific absorption rate (SAR) for a mobile device antenna next to a human head phantom model
- Analyse radar cross-section of a complex target using MLFMM with physical optics augmentation
- Produce detailed technical reports with solver validation, uncertainty quantification, and peer-review documentation

---

### Manufacturing/Process

**Modified UI/UX:**
- Fabrication-centric view showing PCB stackup, etching tolerances, and antenna element manufacturing specifications
- Tolerance analysis dashboard predicting performance variation due to PCB manufacturing tolerances (etching, dielectric constant, thickness)
- Test fixture design panel for antenna measurement with probe compensation and cable de-embedding

**Feature Gating:**
- Manufacturing tolerance sensitivity tools fully enabled
- Test fixture and probe compensation simulation for measurement correlation
- Integration with PCB fabrication DFM tools and antenna measurement systems

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Import antenna design and auto-generate manufacturing specification with critical dimensions and tolerances
- Guided tutorial on evaluating PCB etching tolerance impact on antenna resonant frequency
- Connect to antenna measurement system for automated simulated-vs-measured comparison

**Key Workflows:**
- Evaluate impact of PCB manufacturing tolerances (Dk variation, etch factor, layer registration) on antenna performance
- Design antenna test fixtures and calibration standards for production testing
- Simulate cable and connector effects on measured antenna performance for de-embedding
- Develop manufacturing acceptance criteria linking simulated tolerance bounds to performance specifications
- Optimise antenna design for manufacturability reducing sensitivity to process variations

---

### Regulatory/Compliance

**Modified UI/UX:**
- EMC/RF regulatory dashboard organised by standard (FCC Part 15, CISPR 32, EN 301 489, IEC 62209 SAR)
- Pre-compliance prediction panel comparing simulated emissions/immunity against regulatory limits with margin indicators
- Certification documentation generator with test plan, predicted results, and supporting evidence packages

**Feature Gating:**
- EMC standards library with emission limits, measurement configurations, and compliance criteria
- SAR computation tools per IEC 62209 and IEEE 1528 with standardised phantom models
- RF exposure assessment per FCC OET 65 and ICNIRP guidelines

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Market selector (US FCC, EU CE, Japan MIC, China MIIT) pre-loads applicable RF regulatory requirements
- Walkthrough of a radiated emissions pre-compliance simulation versus FCC Part 15 Class B limits
- Configure test lab submission templates and certification body formatting requirements

**Key Workflows:**
- Predict radiated emissions from a PCB layout and evaluate compliance with FCC Part 15 / CISPR 32 before physical testing
- Compute SAR for a wireless device per IEC 62209-1/-2 and determine compliance with regulatory limits
- Assess RF exposure from a base station antenna installation per FCC OET 65 / ICNIRP guidelines
- Evaluate intentional radiator compliance (antenna gain, EIRP, spurious emissions) for FCC equipment authorisation
- Generate pre-compliance reports reducing risk and cost of formal certification lab testing

---

### Manager/Decision-maker

**Modified UI/UX:**
- Product portfolio dashboard showing RF/antenna design status, EMC compliance readiness, and certification timeline
- Cost-schedule risk matrix for antenna/RF development programmes with simulation maturity indicators
- Competitive benchmarking panel comparing antenna performance metrics against competing products

**Feature Gating:**
- Read-only access to simulation results and compliance predictions; cannot modify geometries or solver settings
- Full access to project tracking, certification scheduling, and cost estimation modules
- Product performance benchmarking and market positioning analytics

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Product portfolio import linking RF/antenna designs to product launch schedules and certification milestones
- Dashboard configuration selecting KPIs (simulation-to-test correlation, certification first-pass rate, time-to-compliance)
- Demo of certification cost-risk analysis for a sample multi-market wireless product launch

**Key Workflows:**
- Track antenna design maturity and EMC compliance readiness across product portfolio for launch planning
- Evaluate cost and schedule risk of multi-market RF certification (FCC, CE, MIC) for product launch timelines
- Compare antenna technology options (PIFA vs. aperture-coupled vs. LDS) for a product platform decision
- Allocate RF engineering resources across concurrent product development programmes
- Present EMC/RF compliance status and certification risk to product management and executive leadership
