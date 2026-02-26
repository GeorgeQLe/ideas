# RadarForge — Persona Subspecs

> Parent spec: [`specs/49-radarforge.md`](../49-radarforge.md)

---

## Experience-Level Personas

### Student/Academic

**Modified UI/UX:**
- Visual radar-equation calculator with interactive sliders for range, RCS, noise figure, and probability of detection
- Block-diagram editor for simple radar architectures (pulsed, CW, FMCW) with animated signal-flow visualization
- Pre-loaded scenarios (air-surveillance, automotive, weather) with default parameters for quick experimentation

**Feature Gating:**
- Radar-range equation, signal-to-noise analysis, basic waveform design (rectangular pulse, LFM chirp), and matched-filter simulation available
- MIMO radar, STAP (space-time adaptive processing), ECCM, and hardware-in-the-loop locked
- Target scene limited to 20 scatterers; terrain and clutter modeling restricted to basic statistical models

**Pricing Tier:** Free tier (education license with .edu verification)

**Onboarding Flow:**
- "Design Your First Radar" tutorial: set up a pulsed surveillance radar, calculate detection range, and simulate target returns
- Interactive radar-equation explorer linking each parameter to physical meaning and design trade-offs
- Pre-loaded homework templates aligned with standard radar-engineering textbooks (Skolnik, Richards)

**Key Workflows:**
- Computing radar detection range using the radar equation with specified Pd and Pfa targets
- Designing LFM chirp waveforms and visualizing ambiguity functions in range and Doppler
- Simulating pulse-Doppler processing to separate target returns from clutter
- Generating range-Doppler maps from simulated target scenarios for analysis and reporting
- Exporting calculations and plots for lab reports and presentation slides

---

### Junior Engineer (0-3 years)

**Modified UI/UX:**
- Professional workspace with waveform designer, signal-processing chain editor, scenario builder, and performance analyzer
- Guided waveform-optimization wizard recommending chirp bandwidth, PRF, and pulse width based on requirements
- Link-budget calculator integrated with system block diagram showing gain, loss, and noise contributions at each stage

**Feature Gating:**
- Full waveform library (LFM, NLFM, phase-coded, stepped-frequency), antenna-pattern modeling, and Doppler processing unlocked
- CFAR detection, tracking (alpha-beta, Kalman), and basic electronic counter-countermeasures available
- STAP and MIMO radar design available in guided mode; full adaptive beamforming restricted

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- System-requirements wizard: define radar mission (surveillance, tracking, imaging, altimeter) to configure default architecture
- Import existing radar models from MATLAB Radar Toolbox with parameter mapping and validation
- Guided link-budget walkthrough: transmitter > antenna > propagation > target > receiver > signal processing

**Key Workflows:**
- Designing radar waveforms optimized for specific range resolution, Doppler coverage, and ambiguity requirements
- Building complete signal-processing chains from raw ADC samples through detection and tracking
- Modeling antenna patterns (phased arrays, reflector dishes, patch arrays) and evaluating beam-steering performance
- Running Monte Carlo detection-performance simulations across varying target RCS and clutter conditions
- Generating system-performance reports with detection probability curves, range coverage diagrams, and link budgets

---

### Senior/Expert

**Modified UI/UX:**
- Multi-mode workspace supporting simultaneous waveform design, beamforming, signal processing, and electronic warfare analysis
- Custom algorithm development environment with Python/MATLAB scripting and real-time data visualization
- Hardware integration panel for connecting to SDR platforms, real antenna arrays, and recorded field-test data

**Feature Gating:**
- All capabilities unlocked: MIMO radar, STAP, cognitive radar, SAR/ISAR imaging, ECCM/ECM analysis, spectrum management
- Custom algorithm plug-in framework for proprietary signal-processing techniques
- API for integration with electronic-warfare simulation suites, mission-planning systems, and recorded-data repositories

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Migration of custom MATLAB radar algorithms with automated code translation and validation testing
- Configure SDR and hardware interfaces (USRP, Xilinx RFSoC, custom receivers) for live-signal testing
- Enterprise integration setup with mission-planning tools, threat databases, and field-test data archives

**Key Workflows:**
- Designing advanced MIMO and cognitive radar systems with adaptive waveform and beam management
- Developing STAP algorithms for ground-clutter suppression in airborne radar scenarios
- Creating SAR/ISAR imaging algorithms and evaluating image quality under various flight geometries and target motions
- Performing electronic counter-countermeasure analysis against specified jammer and interference threats
- Integrating radar models into system-of-systems simulations for mission-level performance assessment

---

## Role-Based Personas

### Designer/Creator

**Modified UI/UX:**
- System-architecture canvas for sketching radar block diagrams with parameterized subsystem modules
- Trade-study comparison tool overlaying performance curves for alternative design choices (waveform, antenna, receiver)
- Visual antenna-array layout editor with element positioning, weighting, and beam-pattern preview

**Feature Gating:**
- Full system-architecture tools with parametric radar subsystem models
- Trade-study and sensitivity-analysis tools for evaluating design alternatives
- Detailed signal-processing algorithm development available but secondary to system-level architecture

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- "Radar Architecture Patterns" gallery: surveillance, tracking, multi-function, automotive, weather templates with rationale
- Quick-estimate tools for system-level performance (range, resolution, accuracy) from top-level parameters
- Tutorial on antenna-array design: element selection, spacing, weighting, and grating-lobe avoidance

**Key Workflows:**
- Architecting radar systems from mission requirements through subsystem specification
- Evaluating waveform and antenna alternatives through parametric trade studies
- Designing phased-array antenna configurations with beam-steering coverage analysis
- Generating system-specification documents from architecture models with performance allocations
- Collaborating with subsystem teams by publishing interface-control documents and performance budgets

---

### Analyst/Simulator

**Modified UI/UX:**
- Scenario-simulation workspace with terrain/clutter modeler, target generator, and electronic-warfare environment
- Signal-processing algorithm test bench with recorded or synthetic data input and performance-metric extraction
- Statistical-analysis dashboard with ROC curves, cumulative detection probability, and tracking accuracy metrics

**Feature Gating:**
- All simulation capabilities unlocked: high-fidelity clutter, multipath, atmospheric propagation, target dynamics
- Recorded-data import and replay for algorithm validation against field measurements
- Signal-processing development tools available; system architecture positioned as scenario configuration

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Scenario-library setup: configure terrain databases, clutter models, and target-signature libraries
- Validation exercise: simulate known radar scenarios and compare detection performance with measured field data
- Tutorial on statistical performance analysis: Monte Carlo simulation design, confidence intervals, and ROC-curve generation

**Key Workflows:**
- Building high-fidelity radar scenarios with realistic terrain, clutter, multipath, and atmospheric effects
- Running Monte Carlo detection-performance simulations to generate statistically valid performance estimates
- Evaluating signal-processing algorithm performance (CFAR, tracker, classifier) against simulated and recorded data
- Performing electronic-warfare simulations assessing radar performance under jamming and interference
- Producing analysis reports with statistical performance characterization and sensitivity to key parameters

---

### Manufacturing/Process

**Modified UI/UX:**
- Hardware-calibration workspace with antenna-measurement data import and pattern verification tools
- Production-test sequence builder for radar subsystem and system-level acceptance testing
- Component-tolerance analysis panel showing how hardware variations affect system performance

**Feature Gating:**
- System simulation with emphasis on hardware-parameter sensitivity and tolerance analysis
- Test-sequence generator for automated production testing of radar assemblies
- Antenna-measurement data import and comparison against design predictions

**Pricing Tier:** Professional tier with Production add-on

**Onboarding Flow:**
- Production-test setup: define test sequences, acceptance criteria, and calibration procedures for radar hardware
- Import antenna-measurement data (near-field/far-field) for pattern verification against simulation
- Configure component-tolerance databases for Monte Carlo system-performance sensitivity analysis

**Key Workflows:**
- Analyzing the impact of component tolerances (amplifier gain, phase shifter accuracy, ADC resolution) on system performance
- Creating automated production-test sequences for radar transmitter, receiver, and antenna subsystems
- Comparing measured antenna patterns against design predictions and diagnosing discrepancies
- Performing calibration analysis to define and verify radar-system calibration procedures
- Generating production acceptance-test reports with pass/fail criteria and measurement traceability

---

### Regulatory/Compliance

**Modified UI/UX:**
- Spectrum-management dashboard showing frequency allocations, emission masks, and interference analysis results
- Compliance matrix mapping radar parameters to ITU Radio Regulations, FCC Part 90/97, and national spectrum authorities
- EMI/EMC analysis panel for evaluating spurious emissions and susceptibility to external interference

**Feature Gating:**
- Full radar simulation with emphasis on spectrum-compliance analysis
- Emission-mask verification tools checking radiated emissions against regulatory limits
- Interference-analysis tools for co-site and adjacent-band compatibility studies

**Pricing Tier:** Enterprise tier with Spectrum module

**Onboarding Flow:**
- Regulatory-profile setup: select frequency band and applicable regulations (ITU, FCC, ETSI, national authorities)
- Configure emission-mask templates for target frequency allocations and neighboring services
- Demo compliance workflow: waveform design > emission analysis > mask verification > filing documentation

**Key Workflows:**
- Analyzing radar waveform spectral characteristics and verifying compliance with emission-mask requirements
- Performing co-site interference analysis for radar installations with co-located communications equipment
- Evaluating spectrum-sharing scenarios with adjacent-band users and computing interference margins
- Generating spectrum-management documentation for regulatory filings and frequency-allocation requests
- Assessing radar designs against export-control regulations (ITAR, EAR) and classification guidance

---

### Manager/Decision-maker

**Modified UI/UX:**
- Program dashboard with radar system-development milestones, test-campaign status, and risk register
- Technology-readiness assessment view with TRL ratings for each radar subsystem
- Cost and schedule trade-off analysis tools for radar procurement and development decisions

**Feature Gating:**
- Read-only access to radar designs, simulation results, and test data with annotation and approval tools
- Full program management, resource allocation, and schedule tracking
- No direct design or simulation execution; all changes routed through engineering team

**Pricing Tier:** Enterprise tier (manager seat)

**Onboarding Flow:**
- Program-management setup: define development phases (concept, engineering, production) with gate-review criteria
- Configure TRL assessment framework aligned with DoD or internal technology-maturity definitions
- Walkthrough of cost-estimation tools linking radar-system complexity to development cost and schedule

**Key Workflows:**
- Monitoring radar-system development milestones and identifying technical risks requiring mitigation
- Reviewing technology-readiness assessments and making go/no-go decisions at development gates
- Evaluating radar system alternatives based on performance, cost, schedule, and risk trade-offs
- Allocating engineering resources and test-range time across concurrent radar programs
- Communicating program status and technical capabilities to customers, sponsors, and executive leadership
