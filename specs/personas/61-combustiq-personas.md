# CombustiQ — Persona Subspecs

> Parent spec: [`specs/61-combustiq.md`](../61-combustiq.md)

---

## Experience-Level Personas

### Student/Academic

**Modified UI/UX:**
- Guided mode with annotated diagrams explaining combustion regimes (premixed, non-premixed, partially premixed) and when each model applies
- Inline tooltips linking to textbook references (Turns, Glassman & Yetter) for every thermodynamic and kinetic parameter
- Simplified mesh generation with pre-built canonical geometries (Bunsen burner, counterflow flame, jet flame)

**Feature Gating:**
- Full access to 0D/1D reactors (perfectly stirred, plug flow) and laminar flame solvers; 3D CFD limited to coarse meshes (<500K cells)
- Chemical mechanism reduction tools available; detailed chemistry solvers capped at 50-species mechanisms
- Export limited to academic paper formats (plots, CSV); no proprietary CAD import/export

**Pricing Tier:** Free tier (academic email verification)

**Onboarding Flow:**
- Start with a pre-loaded methane/air premixed flame tutorial that walks through mechanism selection, boundary conditions, and post-processing
- Interactive quiz on combustion fundamentals to suggest appropriate starting difficulty level
- Prompt to join the CombustiQ Academic Community forum for peer support

**Key Workflows:**
- Running canonical flame simulations (counterflow, freely propagating) for coursework assignments
- Comparing chemical mechanisms (GRI-Mech 3.0 vs. reduced) for thesis research
- Computing laminar flame speed, ignition delay, and extinction strain rate
- Generating publication-quality flame structure plots (temperature, species profiles)

---

### Junior Engineer (0-3 years)

**Modified UI/UX:**
- Wizard-based setup for common industrial configurations (gas turbine can combustor, IC engine cylinder, industrial burner)
- Color-coded input validation that flags physically unrealistic boundary conditions (e.g., inlet temperatures above material limits)
- Side-by-side comparison view between current simulation and a validated reference case

**Feature Gating:**
- Access to 3D RANS combustion CFD with standard turbulence-chemistry interaction models (flamelet, eddy dissipation)
- LES and DNS solvers hidden; detailed chemistry limited to pre-validated reduced mechanisms
- Emissions post-processing (NOx, CO) enabled; advanced soot models require senior approval

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Select industry vertical (gas turbine, automotive, industrial heating) to receive tailored template library
- Guided walkthrough of a validated burner simulation with step-by-step annotations explaining each modeling choice
- Automatic mesh sensitivity check on first simulation to teach grid independence

**Key Workflows:**
- Setting up burner/combustor CFD with flamelet or EDC models for design iterations
- Running parametric sweeps on equivalence ratio, swirl number, and air staging for NOx reduction
- Comparing simulation results against experimental data for model validation
- Generating engineering reports with temperature contours, emissions predictions, and heat flux maps
- Evaluating fuel flexibility (natural gas to hydrogen blends) for existing burner geometries

---

### Senior/Expert

**Modified UI/UX:**
- Full scripting console with Python API for custom combustion model development and batch parametric studies
- Direct access to solver configuration files, numerical scheme selection (PISO, SIMPLE, density-based), and chemistry-turbulence coupling parameters
- Multi-monitor layout support with simultaneous visualization of reacting flow fields, species concentrations, and emission maps

**Feature Gating:**
- All solvers unlocked: LES with detailed chemistry, DNS for fundamental combustion research, spray combustion with Lagrangian tracking
- Custom chemical mechanism import (CHEMKIN format) with automatic validation and reduction tools
- Multi-GPU parallel computing with priority queue access for large-scale simulations (10M+ cells)

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Import existing simulation setups from ANSYS Fluent, CONVERGE, or OpenFOAM; automatic parameter mapping
- Benchmark suite comparing CombustiQ results against published experimental datasets (Sandia flames, DLR combustors)
- API key provisioning for CI/CD integration and automated simulation pipelines

**Key Workflows:**
- Running LES of gas turbine combustors with detailed chemistry for thermoacoustic instability prediction
- Developing and validating custom chemical mechanisms for novel fuels (ammonia, e-fuels, hydrogen blends)
- Spray combustion simulation for direct injection engines with coupled soot/NOx prediction
- Multi-fidelity optimization: fast RANS screening followed by LES confirmation of optimal designs
- Generating regulatory compliance documentation for emissions certification (EPA, EU IED)

---

## Role-Based Personas

### Designer/Creator

**Modified UI/UX:**
- CAD-centric view with integrated combustor geometry editor; drag-and-drop fuel injector, swirler, and liner modules
- Real-time preview of flame shape and temperature field during geometry modifications
- Design variant management with branching and tagging of geometric configurations

**Feature Gating:**
- Full geometry creation and modification tools; parametric combustor design templates
- Automated meshing with combustion-specific refinement zones (flame front, shear layers, near-wall)
- DOE and optimization modules for combustor geometry parameters (swirl angle, dilution hole placement)

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Import existing combustor CAD (STEP/IGES) and auto-detect geometric features (injector, liner, dilution holes)
- Template gallery of combustor archetypes (can, annular, silo) to start from
- Walk through a parametric swirler design study demonstrating the geometry-to-results pipeline

**Key Workflows:**
- Parametric combustor geometry design with automated CFD feedback on flame stability
- Fuel injector nozzle design with spray pattern optimization
- Air staging and dilution hole placement for pattern factor and NOx reduction
- Comparing combustor configurations across fuel types (NG, hydrogen, dual-fuel)
- Exporting optimized designs with thermal boundary conditions for downstream structural analysis

---

### Analyst/Simulator

**Modified UI/UX:**
- Data-driven dashboard with convergence monitors, residual plots, and species balance tracking front and center
- Multi-case comparison workspace with synchronized visualization across parametric runs
- Statistical post-processing panel with uncertainty quantification and sensitivity analysis tools

**Feature Gating:**
- All combustion models and solvers available based on license tier
- Advanced post-processing: flame surface density, heat release rate, probability of extinction maps
- Batch submission queue with job prioritization and resource monitoring

**Pricing Tier:** Professional or Enterprise tier

**Onboarding Flow:**
- Import an existing validated case to establish baseline accuracy before running new configurations
- Tutorial on selecting appropriate combustion models (flamelet vs. finite-rate vs. PDF transport) for different flame regimes
- Convergence and accuracy checklist that runs automatically on every simulation

**Key Workflows:**
- Running high-fidelity combustion simulations with detailed chemistry and turbulence-chemistry interaction
- Performing mesh independence studies and model sensitivity analyses
- Validating simulation results against experimental measurements (LIF, PIV, CARS data)
- Conducting parametric studies on operating conditions (pressure, temperature, equivalence ratio)
- Generating uncertainty-quantified emissions predictions with confidence intervals

---

### Manufacturing/Process

**Modified UI/UX:**
- Operator-friendly interface focused on burner tuning and combustion diagnostics rather than geometry design
- Live data integration panel for connecting DCS/SCADA signals to digital twin models
- Alarm-style interface highlighting off-spec combustion conditions (flashback risk, lean blowout proximity)

**Feature Gating:**
- Simplified steady-state combustion models calibrated to operational range; no geometry modification tools
- Digital twin mode with real-time parameter adjustment (air-fuel ratio, burner load)
- Emissions monitoring dashboard with regulatory limit overlay

**Pricing Tier:** Professional tier (operations license)

**Onboarding Flow:**
- Connect to plant data historian (OSIsoft PI, Honeywell PHD) to auto-populate operating conditions
- Guided calibration of digital twin model against commissioning test data
- Training on interpreting combustion diagnostics (flame temperature, CO/NOx trends)

**Key Workflows:**
- Tuning burner air-fuel ratios to minimize NOx while maintaining flame stability
- Diagnosing combustion problems (flameo, pulsation, incomplete combustion) using digital twin
- Evaluating fuel changeover scenarios (natural gas to hydrogen blend) before physical switchover
- Monitoring combustion efficiency and emissions in real-time against permit limits
- Planning maintenance shutdowns based on predicted combustor liner thermal degradation

---

### Regulatory/Compliance

**Modified UI/UX:**
- Compliance-first dashboard organized by regulation (EPA NSPS, EU IED, local permits) with pass/fail status
- Audit trail view showing every simulation input, solver version, and result with timestamps
- Template-based reporting aligned with regulatory submission formats (permit applications, BACT analysis)

**Feature Gating:**
- Read-only access to simulation results; cannot modify solver settings or geometry
- Automated emissions report generation (NOx, CO, PM, VOC) with regulatory limit comparison
- Version-controlled simulation archive with digital signatures for submission integrity

**Pricing Tier:** Enterprise tier (compliance add-on)

**Onboarding Flow:**
- Select applicable regulations (federal, state, local) to auto-configure compliance checking rules
- Walk through a sample BACT (Best Available Control Technology) analysis using simulation-based emissions data
- Set up automated alerts for simulations that predict emissions exceeding permit thresholds

**Key Workflows:**
- Generating emissions compliance reports for air quality permit applications
- Performing BACT analysis comparing combustion technologies using simulation-based emissions data
- Documenting simulation methodology for regulatory audit defense
- Tracking emissions across operating envelope to ensure continuous compliance
- Reviewing combustion modification proposals for regulatory impact before implementation

---

### Manager/Decision-maker

**Modified UI/UX:**
- Executive dashboard with project-level KPIs: emissions targets vs. predicted, cost per simulation, design iteration count
- Portfolio view across multiple combustor development programs with progress tracking
- ROI calculator comparing simulation costs against physical testing costs avoided

**Feature Gating:**
- View-only access to simulation summaries and high-level results; no solver or model access
- Team analytics: utilization rates, compute spend, simulation throughput per engineer
- Budget management with compute cost alerts and allocation controls

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Connect to project management tools (Jira, MS Project) for automatic status synchronization
- Configure KPI dashboards for emissions targets, development milestones, and budget tracking
- Walkthrough of cost-benefit analysis framework for simulation vs. physical testing decisions

**Key Workflows:**
- Reviewing combustor development progress against emissions targets and timeline milestones
- Approving compute budget allocations for large simulation campaigns
- Comparing design alternatives at a high level (emissions, performance, cost) for go/no-go decisions
- Generating executive summaries of combustion R&D outcomes for stakeholder reporting
- Tracking return on simulation investment (reduced physical testing, faster time-to-market)
