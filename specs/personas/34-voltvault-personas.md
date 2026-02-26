# VoltVault — Persona Subspecs

> Parent spec: [`specs/34-voltvault.md`](../34-voltvault.md)

---

## Experience-Level Personas

### Student/Academic
**Modified UI/UX:**
- Guided simulation wizard: select cell chemistry (LFP, NMC, LCO), define cell geometry, set cycling protocol — each step with educational annotations explaining the underlying physics
- Interactive parameter explorer with sliders for electrode thickness, porosity, and particle size showing real-time impact on discharge curve predictions
- Simplified results viewer with pre-configured plots (voltage vs. capacity, Nyquist diagram, temperature map) and labeled features

**Feature Gating:**
- Access to SPM (Single Particle Model) and basic P2D simulations with pre-loaded material parameter sets
- EIS fitting limited to Randles circuit and 3 common equivalent circuit models; custom circuit building disabled
- Single simulation runs only; parameter sweeps and batch processing disabled

**Pricing Tier:** Free tier (academic email verification)

**Onboarding Flow:**
- Tutorial: simulate a lithium-ion coin cell discharge curve using SPM, compare to published experimental data
- Pre-loaded example cells (NMC/Graphite, LFP/Graphite, NCA/Si-C) with validated parameters for immediate experimentation
- Prompt to join a research group workspace if an advisor's invite code is available

**Key Workflows:**
- Simulate discharge curves for standard lithium-ion chemistries and compare against textbook predictions
- Explore how electrode design parameters (thickness, porosity, loading) affect cell performance
- Fit EIS data from lab experiments using pre-built equivalent circuit models
- Generate simulation result plots for course assignments and thesis figures
- Validate simulation predictions against published experimental datasets

---

### Junior Engineer (0-3 years)
**Modified UI/UX:**
- Template-driven project setup: select cell format (pouch, cylindrical, prismatic), chemistry, and application (EV, grid storage, consumer) to auto-populate reasonable starting parameters
- Contextual warnings when simulation parameters are outside physically reasonable ranges (e.g., unrealistic diffusion coefficients)
- Comparison dashboard for evaluating multiple cell designs side-by-side with key metrics (energy density, rate capability, cycle life)

**Feature Gating:**
- Full P2D and SPM solvers with custom material parameters; 3D thermal modeling enabled
- EIS fitting with all standard circuit elements; custom equivalent circuits enabled
- Up to 10 concurrent simulation runs; parameter sweeps up to 50 points per dimension

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Upload cycling data from lab Arbin/Neware cyclers, parse automatically, and overlay simulation predictions
- Walk through a cell design optimization workflow: vary electrode thickness, simulate rate capability, compare designs
- Connect to team data library and set up shared material parameter databases

**Key Workflows:**
- Design and compare electrode configurations for a target energy density and power specification
- Import and manage cycling test data from lab equipment for model validation
- Run P2D simulations across temperature ranges to predict thermal performance
- Fit EIS measurements to extract kinetic and transport parameters for model calibration
- Generate standardized cell design reports for design review meetings

---

### Senior/Expert
**Modified UI/UX:**
- Multi-physics workspace with simultaneous electrochemical, thermal, and degradation model views
- Custom model builder: extend P2D models with user-defined degradation mechanisms (SEI growth, lithium plating, particle cracking)
- Scripting console (Python) for programmatic model setup, batch automation, and custom post-processing

**Feature Gating:**
- All features unlocked: advanced degradation models, GPU-accelerated Monte Carlo parameter estimation, REST API
- Unlimited concurrent simulations and parameter sweeps; priority compute queue
- Admin controls for team material databases, model version management, and data governance

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Import existing PyBaMM scripts or COMSOL models to bootstrap simulation projects
- API integration walkthrough for connecting VoltVault to lab information management systems (LIMS)
- Configure team model library with validated parameter sets and approved degradation submodels

**Key Workflows:**
- Develop and validate custom degradation models for novel cell chemistries and form factors
- Run GPU-accelerated parameter estimation campaigns fitting models to thousands of cycling datasets simultaneously
- Architect automated cell design optimization loops integrating simulation with experimental DoE
- Manage team-wide model libraries ensuring consistency and reproducibility across projects
- Lead technical reviews of simulation methodology and validate junior engineers' results

---

## Role-Based Personas

### Designer/Creator
**Modified UI/UX:**
- Cell design canvas showing cross-sectional electrode stack geometry with interactive layer editing
- Real-time performance preview updating energy density, power density, and thermal metrics as geometry changes
- Material palette for browsing and selecting electrode, electrolyte, and separator materials with property cards

**Feature Gating:**
- Full cell design tools: electrode stack editor, tab/busbar placement, jellyroll winding configuration
- Integrated thermal design with cooling channel placement and heat generation mapping
- Export cell designs as 3D models (STEP) for mechanical integration and as simulation-ready configurations

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Design a pouch cell from scratch: select materials, define electrode dimensions, configure tab layout
- Walk through thermal design: identify hot spots, place cooling channels, re-simulate to verify
- Save the design as a parameterized template for the team library

**Key Workflows:**
- Design electrode stack configurations optimized for target energy density and rate capability
- Explore cell form factor trade-offs (cylindrical vs. prismatic vs. pouch) for a given application
- Integrate thermal management features into cell-level design to minimize temperature gradients
- Create parameterized cell design templates for rapid exploration of chemistry variants
- Export finalized cell designs for prototyping and manufacturing planning

---

### Analyst/Simulator
**Modified UI/UX:**
- Simulation monitoring dashboard showing active runs, convergence status, and result queue
- Advanced post-processing workspace with scriptable plot generation, statistical analysis, and curve fitting
- Model validation panel for systematic comparison of simulation predictions vs. experimental data

**Feature Gating:**
- Full access to all solver types, degradation models, and parameter estimation tools
- Batch simulation management with automated result aggregation and statistical analysis
- Custom post-processing scripts and automated report generation enabled

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Configure a parameter sweep study: vary 3 parameters, run 100+ simulations, analyze Pareto-optimal designs
- Set up model validation workflow: import experimental data, run equivalent simulations, compute error metrics
- Build automated analysis pipelines for recurring simulation campaigns

**Key Workflows:**
- Run large-scale parameter sensitivity studies to identify dominant factors in cell performance
- Validate electrochemical models against experimental cycling and EIS data with quantified error metrics
- Perform degradation lifetime predictions using accelerated aging simulation protocols
- Generate statistical summaries of simulation campaigns for design space exploration reports
- Build automated simulation-to-report pipelines for recurring analysis tasks

---

### Manufacturing/Process
**Modified UI/UX:**
- Process parameter dashboard linking manufacturing variables (coating thickness tolerance, calendering pressure, electrolyte fill) to predicted cell performance
- Statistical process control charts showing how manufacturing variability propagates to electrical performance spread
- Batch tracking view connecting simulation predictions to actual cell test results by lot number

**Feature Gating:**
- Access to manufacturing sensitivity analysis tools and process-performance correlation models
- SPC analytics linking manufacturing data to simulation predictions; batch comparison tools
- Simulation setup restricted to approved cell design templates; new model creation requires engineer approval

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Connect manufacturing execution system (MES) data feeds for coating, calendering, and formation processes
- Walk through a manufacturing sensitivity study: how does +/-5% coating thickness variation affect capacity spread?
- Configure SPC dashboards linking process parameters to predicted cell performance

**Key Workflows:**
- Analyze how manufacturing process variations impact predicted cell performance and yield
- Correlate actual cell test data with simulation predictions to calibrate manufacturing-aware models
- Identify the most critical process parameters for cell performance using simulation-based sensitivity analysis
- Track manufacturing lot-to-lot variability and its predicted impact on field performance
- Generate process capability reports linking manufacturing tolerances to cell specification compliance

---

### Regulatory/Compliance
**Modified UI/UX:**
- Safety analysis dashboard showing abuse simulation results: nail penetration, thermal runaway propagation, overcharge response
- Standards compliance tracker (UN 38.3, IEC 62660, UL 2580) with test-to-simulation correlation tables
- Documentation generator producing formatted safety analysis reports for regulatory submissions

**Feature Gating:**
- Full access to abuse simulation tools, thermal runaway models, and safety analysis results
- Audit trail with immutable logging of all simulation parameters, solver versions, and results
- Compliance documentation templates with regulatory-specific formatting and required data tables

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Configure applicable battery safety standards and map required tests to simulation capabilities
- Walk through a thermal runaway propagation simulation and compare to physical test requirements
- Set up documentation templates for UN 38.3 transport test equivalence and IEC 62660 performance qualification

**Key Workflows:**
- Run abuse simulation studies (nail penetration, external short, overcharge) to predict safety margins before physical testing
- Correlate simulation predictions with physical test results to build a validated virtual testing framework
- Generate safety analysis documentation for UN 38.3, IEC 62660, and UL certification submissions
- Track compliance status across cell variants and maintain a registry of validated simulation models
- Review and approve simulation-based safety claims before including them in regulatory filings

---

### Manager/Decision-maker
**Modified UI/UX:**
- Program dashboard showing cell development pipeline: chemistries under evaluation, simulation milestones, and test correlation status
- Cost modeling panel estimating $/kWh at pack level based on cell design parameters and material costs
- Timeline view tracking cell design iterations from concept through simulation through prototype through validation

**Feature Gating:**
- Read-only access to program dashboards, cell design comparisons, and cost models; no direct simulation tools
- Team management: license allocation, project creation, compute budget controls
- Approval authority for cell designs advancing from simulation to prototype build

**Pricing Tier:** Enterprise tier (included in admin seat)

**Onboarding Flow:**
- Program dashboard overview: active cell development projects, key performance metrics, milestone tracking
- Configure portfolio KPIs (target energy density, cost, cycle life) and milestone gates
- Set up notification rules for simulation results that exceed or miss program targets

**Key Workflows:**
- Monitor cell development program progress across multiple chemistry and form factor tracks
- Compare cell design candidates on performance, cost, and safety metrics to inform go/no-go decisions
- Track compute and testing resource allocation across projects and adjust priorities
- Review simulation-vs-test correlation to assess confidence in virtual prototyping predictions
- Present cell development status and technology road map to executive leadership and investors
