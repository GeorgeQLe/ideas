# PlasmaTech — Persona Subspecs

> Parent spec: [`specs/56-plasmatech.md`](../56-plasmatech.md)

---

## Experience-Level Personas

### Student/Academic

**Modified UI/UX:**
- Simplified 1D/2D geometry editor with predefined reactor templates (parallel plate, ICP coil, barrel asher)
- Inline plasma physics primer showing Boltzmann distribution, Debye length, and sheath formation concepts with interactive sliders
- Results panel defaults to electron density, temperature, and species concentration profiles with annotated plasma regions

**Feature Gating:**
- Geometry limited to 2D axisymmetric; fluid plasma model only (no PIC/MC or hybrid solvers)
- Chemistry limited to pre-built sets (Ar, O2, CF4) with max 20 species; custom reaction editor locked
- Export to PDF and CSV; no TCAD or equipment-control interop

**Pricing Tier:** Free tier (educational license)

**Onboarding Flow:**
- "First Plasma" tutorial: set up a capacitively-coupled Ar discharge, run fluid simulation, interpret density and potential profiles
- Interactive Paschen curve explorer linking breakdown voltage to pd product with experimental comparison
- Sandbox with pre-built textbook cases (DC glow, RF CCP, microwave plasma)

**Key Workflows:**
- Simulate an RF argon discharge and examine electron density and plasma potential spatial profiles
- Study the effect of pressure and power on electron temperature in a CCP reactor
- Compare fluid-model predictions against analytical sheath theory
- Generate lab report figures showing species density profiles and ion energy distributions

---

### Junior Engineer (0-3 years)

**Modified UI/UX:**
- Standard 2D/3D workspace with reactor design wizard: select plasma source type, gas chemistry, and process goal
- Guided chemistry set builder with reaction rate sensitivity ranking and dominant pathway highlighting
- Warning indicators on operating conditions outside stable plasma regime (extinction, arcing, powder formation)

**Feature Gating:**
- Full 2D axisymmetric solver; 3D available for simple geometries (under 500k cells)
- Fluid and drift-diffusion solvers; PIC available for 1D sheath studies
- Pre-built chemistry sets for semiconductor etch gases (CF4, SF6, Cl2, HBr) with validated rate coefficients

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Process-type selector (etching, deposition, cleaning, surface treatment) pre-loads relevant chemistry and reactor geometry
- Walkthrough of an oxide etch process simulation: setup chemistry, run steady-state, evaluate etch uniformity
- Prompted to import fab's reactor dimensions and match to installed equipment configurations

**Key Workflows:**
- Simulate a silicon oxide etch process in an ICP reactor and predict etch rate uniformity across the wafer
- Evaluate the effect of ICP coil design on plasma density uniformity
- Study the influence of process pressure and gas flow ratio on reactive species concentrations
- Predict ion bombardment energy distribution at the wafer surface for damage assessment
- Generate process window maps showing etch rate and selectivity vs. power and pressure

---

### Senior/Expert

**Modified UI/UX:**
- Multi-physics workspace with coupled plasma-surface-feature-scale solvers, PIC/MC engine, and kinetic Boltzmann solver
- Scripting console (Python API) for reaction mechanism development, cross-section fitting, and automated parametric sweeps
- Digital twin dashboard linking reactor model to real-time OES and Langmuir probe diagnostics from the fab

**Feature Gating:**
- All solvers: fluid, PIC/MC, hybrid, global (0D) models, and feature-scale profile simulators
- Custom chemistry editor with Boltzmann cross-section solver and sensitivity/uncertainty analysis
- Full API, HPC cluster support, and integration with TCAD, equipment control systems, and fab databases

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Import existing COMSOL Plasma/VSim/CFD-ACE+ models with automated translation and chemistry mapping
- Configure solver preferences, PIC particle counts, and HPC resource allocation
- Power-user preset opening scripted workspace for reaction mechanism development

**Key Workflows:**
- Develop and validate a new etch chemistry mechanism using ab initio cross sections, Boltzmann analysis, and OES comparison
- Perform coupled reactor-scale to feature-scale simulation predicting etch profile evolution in high-aspect-ratio trenches
- Run PIC/MC simulation of RF sheath dynamics to predict ion energy-angle distributions with kinetic accuracy
- Build a reactor digital twin calibrated against Langmuir probe and mass spectrometry data for virtual process development
- Automate recipe optimisation using surrogate models trained on simulation databases

---

## Role-Based Personas

### Designer/Creator

**Modified UI/UX:**
- Reactor geometry design canvas with parametric electrode, coil, gas inlet, and pump port placement tools
- Electromagnetic field preview showing induced fields, skin depth, and power deposition patterns
- Material selection panel for reactor components (ceramics, quartz, anodised aluminium) with plasma compatibility data

**Feature Gating:**
- Full 3D geometry editor with parametric dimensions for reactor design iteration
- Electromagnetic solver (FDTD/FEM) for antenna/coil design coupled to plasma fluid model
- Export to CAD (STEP, IGES) and mechanical drawing packages for reactor fabrication

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Reactor-type selector (CCP, ICP, ECR, helicon, DBD) pre-loads starting geometry and power delivery configuration
- "Design for Uniformity" wizard guiding gas distribution and coil geometry for target plasma uniformity
- Tour of electromagnetic design tools using a sample ICP antenna optimisation

**Key Workflows:**
- Design an ICP source coil geometry optimising plasma density uniformity over a 300mm wafer
- Evaluate gas distribution showerhead hole patterns for uniform precursor delivery
- Design a pulsed DC magnetron sputtering source with magnetic field optimisation
- Iterate reactor chamber geometry to minimise particle generation and improve cleanability
- Produce reactor fabrication drawings with material specifications for manufacturing

---

### Analyst/Simulator

**Modified UI/UX:**
- Analysis workspace with EEDF viewers, species density mappers, Boltzmann equation solvers, and cross-section editors
- Multi-solver comparison panel contrasting fluid vs. PIC/MC results with quantified discrepancy metrics
- Post-processing suite for computing derived quantities (Debye length, plasma frequency, mean free paths)

**Feature Gating:**
- All solver engines fully unlocked including PIC/MC with arbitrary particle counts
- Boltzmann equation solver for EEDF computation from cross-section databases (LXCat integration)
- Sensitivity analysis and global uncertainty quantification for reaction rate coefficients

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Solver validation against published benchmark discharges (GEC reference cell, COST jet)
- Import cross-section databases and configure Boltzmann solver defaults
- Configure convergence criteria, PIC noise reduction strategies, and adaptive time-stepping

**Key Workflows:**
- Solve the Boltzmann equation for electron energy distribution in a gas mixture and evaluate transport coefficients
- Perform fluid-PIC comparison study quantifying validity of fluid approximation for a low-pressure ICP
- Conduct reaction path analysis identifying dominant creation and destruction channels for each species
- Quantify uncertainty in etch rate predictions due to cross-section uncertainty using Monte Carlo sampling
- Produce peer-reviewed technical publications with validated simulation methodology documentation

---

### Manufacturing/Process

**Modified UI/UX:**
- Process-centric view organised by product wafer recipes (etch, dep, clean) with linked reactor models
- SPC-style dashboard tracking simulated vs. measured etch rate, uniformity, and selectivity across production lots
- Equipment matching panel comparing simulation predictions for identical processes on different reactor instances

**Feature Gating:**
- Fast global (0D) models for real-time process monitoring and virtual metrology
- Feature-scale profile simulator for etch/dep profile prediction linked to reactor-scale results
- Integration with fab MES, FDC, and SPC systems for model-based process control

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Recipe library import: bring in existing process recipes with known performance baselines
- Guided calibration workflow matching simulation to production metrology data
- Connect to fab data systems for automated process monitoring and drift detection

**Key Workflows:**
- Calibrate reactor model against production data and use for virtual process development of new recipes
- Predict etch profile evolution for a new device technology node before silicon commitment
- Diagnose root cause of process drift by comparing simulated vs. measured plasma diagnostics
- Evaluate chamber matching by simulating the same recipe on multiple reactor geometries
- Develop model-based run-to-run control algorithms using fast global plasma models

---

### Regulatory/Compliance

**Modified UI/UX:**
- Safety and environmental compliance dashboard covering hazardous gas handling, emissions, and RF exposure
- Gas-by-gas hazard summary with PEL, TLV, GWP, and abatement requirements for every species in the chemistry
- Exhaust emissions calculator predicting effluent composition and required abatement efficiency

**Feature Gating:**
- Emissions prediction module (PFC/GHG output from process) fully enabled
- Safety interlock simulation for gas leak, over-pressure, and RF exposure scenarios
- Regulatory reporting templates for EPA, SEMI S2/S8, and local air quality regulations

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Regulatory framework selector: EPA greenhouse gas, SEMI safety guidelines, local air quality district rules
- Walkthrough of PFC emission calculation for a CVD process using GWP-weighted output
- Configure abatement system models for point-of-use scrubber and facility exhaust treatment

**Key Workflows:**
- Calculate process-specific PFC/GHG emissions and evaluate abatement strategies to meet EPA reporting thresholds
- Simulate gas cabinet and VMB failure scenarios to validate safety interlock response adequacy
- Document RF power system compliance with occupational exposure limits per ICNIRP/FCC
- Assess new chemistries for environmental and safety regulatory impacts before introduction to fab
- Generate environmental impact reports for facility permitting and sustainability compliance

---

### Manager/Decision-maker

**Modified UI/UX:**
- Portfolio dashboard showing process development projects with technology-node timelines, simulation maturity, and resource allocation
- Cost-of-ownership comparison charts for competing reactor platforms and process approaches
- Yield impact estimator linking simulated process variability to device yield loss projections

**Feature Gating:**
- Read-only access to simulation results and process models; cannot modify reactor or chemistry parameters
- Full access to project tracking, cost modelling, and technology-readiness dashboards
- Competitive benchmarking tools comparing platform capabilities and process windows

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Technology roadmap import linking device nodes to required process capabilities
- Dashboard configuration selecting KPIs (cycle time, CoO, yield impact, environmental footprint)
- Demo of cost-of-ownership analysis comparing two reactor platforms for a next-node etch process

**Key Workflows:**
- Evaluate cost-of-ownership for new plasma equipment versus extending existing platform capabilities
- Track process development readiness across technology nodes and fab sites
- Assess environmental compliance costs and risks for new process chemistries
- Allocate simulation and process engineering resources across concurrent development programmes
- Present technology risk and investment recommendations to executive leadership for capital planning
