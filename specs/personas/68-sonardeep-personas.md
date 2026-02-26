# SonarDeep — Persona Subspecs

> Parent spec: [`specs/68-sonardeep.md`](../68-sonardeep.md)

---

## Experience-Level Personas

### Student/Academic

**Modified UI/UX:**
- Guided ocean acoustics tutorial mode with annotated ray diagrams explaining refraction, reflection, and sound channel formation
- Interactive sound speed profile editor with visual explanation of thermocline, SOFAR channel, and convergence zone phenomena
- Pre-loaded ocean environment databases (GDEM, WOA) with simplified query interface for common regions

**Feature Gating:**
- Ray tracing (Bellhop-equivalent) and normal mode (KRAKEN-equivalent) solvers for range-independent environments
- Range-dependent propagation limited to 2D (N x 2D); full 3D propagation modeling locked
- Sonar equation calculator available; full sonar system design tools disabled

**Pricing Tier:** Free tier (academic email verification)

**Onboarding Flow:**
- Start with a guided deep-water convergence zone propagation tutorial demonstrating ray tracing in a Munk profile
- Interactive exercise on the sonar equation (SL - TL + TS - NL > DT) with parameter exploration
- Access to canonical test cases (ASA benchmark problems) for validating understanding

**Key Workflows:**
- Computing transmission loss for canonical ocean environments in underwater acoustics coursework
- Comparing propagation models (ray tracing, normal modes, parabolic equation) on the same environment
- Exploring the effect of sound speed profile variations on acoustic propagation patterns
- Generating propagation loss plots and ray diagrams for thesis research

---

### Junior Engineer (0-3 years)

**Modified UI/UX:**
- Template-based environment setup organized by scenario (shallow water ASW, deep water surveillance, harbor monitoring, AUV navigation)
- Guided workflow from environment definition through propagation modeling to detection range prediction
- Visual sonar performance display showing detection probability on geographic maps with range rings

**Feature Gating:**
- All propagation solvers (ray, normal mode, PE) for 2D and range-dependent environments; 3D solvers limited
- Basic sonar system design (array configuration, beamforming) enabled; advanced signal processing locked
- Detection performance prediction available; full system-level engagement simulation requires senior approval

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Select application domain (defense ASW, commercial survey, offshore wind, AUV) for tailored templates
- Guided walkthrough of a submarine detection scenario from environment setup to detection probability map
- Tutorial on selecting the appropriate propagation model for different frequency and range combinations

**Key Workflows:**
- Building ocean environment models with bathymetry, sound speed profiles, and sediment properties
- Running propagation loss calculations for sonar system performance prediction
- Designing simple sonar arrays (line, planar) with beampattern analysis
- Generating detection range predictions for specific target types in realistic ocean environments
- Evaluating the impact of environmental variability (seasonal, diurnal) on sonar performance

---

### Senior/Expert

**Modified UI/UX:**
- Full scripting API (Python/MATLAB) for custom propagation models, signal processing algorithms, and automated scenario analysis
- Multi-scenario orchestration workspace managing hundreds of environment-target-sonar combinations
- Direct solver configuration with advanced options (Gaussian beams, broadband pulse propagation, elastic bottom models)

**Feature Gating:**
- All features unlocked: 3D propagation, broadband modeling, reverberation, ambient noise, elastic seabed
- Custom transducer and array design with arbitrary element configurations and advanced beamforming
- Classified environment integration via secure API (FOUO/SECRET data handling for defense users)

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Import existing Bellhop/KRAKEN/RAM models and MATLAB post-processing scripts with automated translation
- Benchmark against published propagation measurement data (ANDES, SWellEx, LOAPEX datasets)
- Configure batch processing pipelines for fleet-wide sonar performance assessment

**Key Workflows:**
- Developing and validating custom acoustic propagation models for novel ocean environments
- Designing advanced sonar arrays with optimized beamforming for specific threat scenarios
- Running broadband, range-dependent, 3D propagation studies for complex littoral environments
- Building automated sonar performance prediction pipelines for operational decision support
- Leading acoustic environment assessment campaigns integrating measurement data with modeling

---

## Role-Based Personas

### Designer/Creator

**Modified UI/UX:**
- Sonar system design canvas with transducer element editor, array geometry builder, and beampattern visualization
- Interactive link budget calculator with real-time update as system parameters (source level, frequency, bandwidth) change
- Design variant manager for comparing array configurations across performance metrics

**Feature Gating:**
- Full transducer and array design tools with parametric element models
- Beamforming design (CBF, MVDR, adaptive) with sidelobe and grating lobe analysis
- System-level link budget and detection range calculator with environmental propagation integration

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Start with a template sonar array design and walk through element selection, array geometry, and beamforming
- Interactive link budget tutorial connecting each parameter to its physical meaning
- Demonstrate the design-to-performance pipeline: array design, propagation, detection range

**Key Workflows:**
- Designing sonar transducer arrays with specified beam width, sidelobe levels, and frequency response
- Optimizing array geometry for target detection in specified operational environments
- Evaluating sonar system performance across frequency bands and operational scenarios
- Comparing active vs. passive sonar configurations for specific mission requirements
- Generating sonar system specification documents from design and simulation results

---

### Analyst/Simulator

**Modified UI/UX:**
- Analysis workspace with propagation model output visualization (TL, eigenrays, mode shapes) and comparison tools
- Multi-environment scenario analyzer for sensitivity to oceanographic conditions
- Reverberation and ambient noise analysis panel with contribution breakdown (surface, bottom, volume, shipping)

**Feature Gating:**
- All propagation and analysis models available based on license tier
- Advanced analysis: reverberation modeling, ambient noise prediction, signal excess mapping
- Batch simulation with automated environmental scenario generation (Monte Carlo over SSP variability)

**Pricing Tier:** Professional or Enterprise tier

**Onboarding Flow:**
- Tutorial on selecting propagation models for different scenarios (frequency, range, environment complexity)
- Guided reverberation analysis exercise in a shallow water environment
- Introduction to Monte Carlo sonar performance assessment under environmental uncertainty

**Key Workflows:**
- Running high-fidelity propagation loss calculations for sonar system performance assessment
- Analyzing reverberation levels and their impact on active sonar detection in shallow water
- Conducting sensitivity studies on environmental parameters (SSP, bottom type, sea state)
- Performing Monte Carlo simulations to quantify sonar detection probability under uncertainty
- Validating propagation models against at-sea measurement data

---

### Manufacturing/Process

**Modified UI/UX:**
- Transducer manufacturing interface showing element sensitivity maps, impedance matching, and array calibration data
- Production quality dashboard comparing manufactured array beam patterns against design specifications
- Test and calibration workflow linking hydrophone measurements to model-predicted element response

**Feature Gating:**
- Array calibration and beam pattern verification tools; no system design modification
- Element-level diagnostics comparing manufactured vs. designed response
- Production acceptance testing with automated pass/fail criteria from design specifications

**Pricing Tier:** Professional tier (manufacturing license)

**Onboarding Flow:**
- Configure manufacturing test bench parameters and calibration equipment interface
- Walk through a production acceptance test comparing measured beam pattern to design specification
- Tutorial on diagnosing element failures using calibration data and model predictions

**Key Workflows:**
- Verifying manufactured sonar array beam patterns against design specifications
- Calibrating individual transducer elements and diagnosing sensitivity deviations
- Running production acceptance tests with model-based pass/fail criteria
- Evaluating impact of manufacturing tolerances on sonar system performance
- Generating production test reports with measured vs. designed performance comparison

---

### Regulatory/Compliance

**Modified UI/UX:**
- Environmental compliance dashboard organized by regulation (MMPA, EU MSFD, IMO guidelines) for underwater noise
- Marine mammal impact assessment workspace with exposure calculation and take estimation
- Audit trail showing noise source levels, propagation assumptions, and impact zone calculations

**Feature Gating:**
- Read-only access to noise propagation and impact assessment results
- Automated marine mammal noise exposure report generation per NOAA/NMFS guidelines
- Environmental impact assessment templates for offshore construction and naval exercises

**Pricing Tier:** Enterprise tier (compliance add-on)

**Onboarding Flow:**
- Select applicable regulatory framework (US MMPA, EU Habitats Directive, national regulations) for template configuration
- Walk through a pile driving noise impact assessment from source characterization to exclusion zone estimation
- Configure monitoring and mitigation plan documentation workflow

**Key Workflows:**
- Calculating underwater noise propagation from construction activities (pile driving, dredging, blasting)
- Estimating marine mammal noise exposure levels and exclusion zones per NOAA thresholds
- Generating Environmental Impact Assessment sections for offshore wind, oil and gas, and naval projects
- Documenting noise mitigation measures (bubble curtains, seasonal restrictions) with predicted effectiveness
- Preparing permit applications for activities that may harass marine mammals under MMPA

---

### Manager/Decision-maker

**Modified UI/UX:**
- Program dashboard with sonar system development status, performance benchmarks, and budget tracking
- Mission readiness summary showing sonar capability gaps and system effectiveness across operational scenarios
- Investment comparison view evaluating sonar upgrade options on cost, performance, and schedule

**Feature Gating:**
- View-only access to system performance summaries, detection probability maps, and program status
- Portfolio view across sonar development programs with technology readiness levels
- Budget allocation tracking for simulation, prototyping, and at-sea testing

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Configure program portfolio with sonar system development milestones and performance targets
- Walk through interpreting detection performance maps and their operational implications
- Set up automated reporting for program status and capability gap tracking

**Key Workflows:**
- Reviewing sonar system performance predictions for acquisition and upgrade decisions
- Evaluating technology alternatives (frequency, array type, processing) on cost-effectiveness
- Monitoring development program progress against performance milestones and budget
- Assessing operational sonar performance across theaters and environmental conditions
- Generating capability briefings for senior leadership and procurement decision-makers
