# GlassSim — Persona Subspecs

> Parent spec: [`specs/90-glasssim.md`](../90-glasssim.md)

---

## Experience-Level Personas

### Student/Academic
**Modified UI/UX:**
- Simplified glass-forming simulator with preset process templates (float glass, bottle blow-mold, fiber draw) and standard glass compositions
- Animated visualization showing viscosity-temperature curve, glass transition, and forming process with labeled thermal zones
- Interactive composition panel with sliders for SiO2, Na2O, CaO (soda-lime) showing real-time property predictions

**Feature Gating:**
- 1D/2D simulations with basic glass viscosity models (VFT, Arrhenius) and Newtonian flow
- Glass composition database limited to common commercial compositions (soda-lime, borosilicate, E-glass)
- Fiber-draw simulation limited to single fiber; preform design and multi-fiber draw locked

**Pricing Tier:** Free tier

**Onboarding Flow:**
- Tutorial: simulate a glass fiber draw from a preform, observe neck-down profile, and predict fiber diameter vs. draw speed
- Pre-loaded examples (float-glass ribbon forming, bottle blow-molding, optical fiber drawing)
- References to Shelby, Varshneya glass science textbooks linked from material property panels

**Key Workflows:**
- Predict glass viscosity-temperature curves from composition using empirical models
- Simulate fiber drawing and relate process parameters (furnace temperature, draw speed) to fiber diameter
- Model heat transfer during glass cooling and predict temperature distribution and residual stress
- Compare glass compositions on forming behavior using viscosity-working-range analysis

---

### Junior Engineer (0-3 years)
**Modified UI/UX:**
- Project workspace with tabs: Composition | Process Design | Simulation | Property Prediction | Reports
- Integrated glass property database with temperature-dependent viscosity, thermal conductivity, and radiation properties
- Process design wizard guiding mold/die/furnace configuration for the selected forming process

**Feature Gating:**
- Full 2D axisymmetric and 3D simulation with non-Newtonian glass flow and radiation heat transfer
- Composition-property prediction models (viscosity, density, thermal expansion, refractive index) enabled
- Annealing simulation with residual stress prediction; ion-exchange strengthening model gated

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Define glass composition and forming process, import mold/die geometry, and set thermal boundary conditions
- Run forming simulation and analyze glass thickness distribution, temperature field, and defect-risk regions
- Optimize process parameters (plunger speed, blowing pressure, mold temperature) for target quality

**Key Workflows:**
- Simulate glass container blow-molding and predict wall-thickness distribution
- Model float-glass ribbon forming with tin-bath heat transfer and ribbon stability
- Predict residual stress development during annealing lehr cooling schedules
- Design optical fiber preforms and simulate draw process for target fiber geometry
- Evaluate mold design changes on glass distribution and defect formation

---

### Senior/Expert
**Modified UI/UX:**
- Multi-physics simulation platform managing coupled flow-thermal-structural-optical analyses
- Python scripting console with full solver API for custom glass models, crystallization kinetics, and radiation
- Advanced visualization: 3D flow fields, radiation transport, crystallization nucleation maps, optical path analysis

**Feature Gating:**
- All features unlocked: full 3D free-surface flow, spectral radiation transport, crystallization kinetics, ion-exchange modeling
- Unlimited mesh complexity with adaptive refinement and parallel computation
- Plugin SDK for custom viscosity models, crystallization theories, and specialty glass compositions

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Migration wizard importing existing simulation setups and proprietary glass property databases
- HPC cluster configuration for large-scale 3D glass forming simulations with radiation transport
- Workshop on custom glass model development and advanced process coupling

**Key Workflows:**
- Full 3D simulation of complex glass forming (press-and-blow, float, overflow fusion, fiber drawing)
- Spectral radiation transport in semitransparent glass with temperature-dependent absorption
- Crystallization kinetics simulation for devitrification risk assessment and glass-ceramic development
- Ion-exchange strengthening simulation for chemically tempered glass products
- Multi-objective process optimization balancing quality, throughput, energy consumption, and defect risk

---

## Role-Based Personas

### Designer/Creator
**Modified UI/UX:**
- Visual mold/die/preform design canvas with parametric geometry editing and instant forming-preview
- Glass thickness distribution overlay on product geometry with color-mapped uniformity indicators
- Side-by-side comparison view for evaluating alternative mold designs or process configurations

**Feature Gating:**
- Mold and preform design tools with forming-preview simulation enabled
- Full-fidelity simulation available for detailed validation of selected designs
- Export to CAD formats and mold-fabrication drawing generators

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Import product geometry and define glass composition and target wall thickness
- Use auto-preform/parison design tool to generate starting blank shape
- Run forming preview and iterate on mold design to achieve target thickness distribution

**Key Workflows:**
- Design glass container molds and optimize blank/parison shape for target wall-thickness distribution
- Develop optical fiber preform designs for target refractive-index profiles and fiber dimensions
- Design float-glass line configurations (tin-bath length, top-roller positions) for target ribbon dimensions
- Iterate on die/bushing geometry for continuous glass-fiber production

---

### Analyst/Simulator
**Modified UI/UX:**
- Simulation-centric layout with mesh controls, solver configuration, and multi-physics coupling panels
- Comprehensive post-processing: flow fields, temperature evolution, stress development, optical property maps
- Batch simulation manager for parametric studies and process-window characterization

**Feature Gating:**
- Full solver access with all physics modules (flow, heat transfer, radiation, structural, optical)
- Model calibration tools for matching simulation to experimental forming trials
- API access for automated simulation campaigns and custom analysis workflows

**Pricing Tier:** Professional or Enterprise tier

**Onboarding Flow:**
- Set up a reference forming simulation and validate against production measurement data (thermocouples, thickness scans)
- Calibrate glass property models (viscosity, emissivity) against experimental characterization data
- Run parametric study on key process variables and map the quality process window

**Key Workflows:**
- High-fidelity 3D glass forming simulation with free-surface tracking and radiation heat transfer
- Residual stress prediction and annealing schedule optimization for thermal tempering
- Crystallization risk assessment during forming and post-forming heat treatment
- Process-window mapping using parametric studies on forming temperature, speed, and cooling
- Optical property simulation for glass products (refractive index uniformity, birefringence, striae)

---

### Manufacturing/Process
**Modified UI/UX:**
- Production-focused interface showing machine settings, furnace temperatures, and line-speed parameters
- Process recipe manager with version control and change-impact assessment tools
- Integration panels for glass-plant control systems (DCS, SCADA) and quality inspection systems

**Feature Gating:**
- Simulation-derived process recipes and operating setpoints accessible
- Full simulation tools hidden; simplified process-adjustment calculators available
- Integration APIs for plant control systems and quality databases

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Import simulation-optimized process recipe and map to production line settings
- Set up process monitoring with quality control limits derived from simulation
- Connect to plant quality inspection system for defect tracking and correlation

**Key Workflows:**
- Transfer simulation-optimized forming parameters to production line controllers
- Troubleshoot production defects (blisters, cords, stones, thickness variation) using process models
- Optimize furnace energy consumption by simulating melting and conditioning schedules
- Plan job changes (composition switch, product changeover) with simulation-predicted transition behavior

---

### Regulatory/Compliance
**Modified UI/UX:**
- Compliance dashboard for glass product standards (safety glazing, food contact, pharmaceutical, optical)
- Automated test-plan generator for certification testing (ANSI Z97.1, EN 12150, USP, ASTM)
- Material compliance tracking for glass compositions (lead-free, cadmium-free, food-contact regulations)

**Feature Gating:**
- Product performance prediction tools (impact resistance, thermal shock, chemical durability) enabled
- Composition compliance checking against restricted-substance regulations
- Certification documentation generator with standard-specific formatting

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Select applicable product standards and regulations for the glass product type and market
- Import product design and composition data, run compliance assessment
- Generate certification test plan and compliance documentation package

**Key Workflows:**
- Predict safety glazing performance (fragmentation pattern, impact resistance) from tempering simulation
- Assess food-contact compliance based on glass composition and leaching predictions
- Generate pharmaceutical glass container qualification documentation (USP Type I/II/III)
- Track composition compliance with environmental regulations (lead-free, RoHS for electronic glass)
- Produce optical glass qualification reports for precision optic specifications (ISO 12123)

---

### Manager/Decision-maker
**Modified UI/UX:**
- Plant-level dashboard showing production efficiency, quality metrics, energy consumption, and yield KPIs
- Product-line comparison cards with cost, quality, and market-positioning analysis
- Investment analysis tools for furnace rebuilds, line upgrades, and new product introductions

**Feature Gating:**
- Read-only access to all simulation results and compliance reports
- Cost modeling, yield analysis, and technology comparison tools enabled
- Approval workflow for process changes and capital investment decisions

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Dashboard overview: plant performance metrics, energy costs, quality trends, and yield data
- Review product-line economics and technology comparison tools
- Set up approval workflow for process modifications and capital expenditure proposals

**Key Workflows:**
- Evaluate new product introductions with simulation-predicted process feasibility and capital requirements
- Compare glass composition alternatives on cost, performance, processability, and sustainability
- Assess furnace rebuild and line upgrade investments with projected ROI from simulation data
- Monitor plant energy efficiency and evaluate waste-heat recovery or oxy-fuel conversion projects
- Generate executive reports for strategic planning on product portfolio and technology roadmap
