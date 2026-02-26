# PartSim — Persona Subspecs

> Parent spec: [`specs/86-partsim.md`](../86-partsim.md)

---

## Experience-Level Personas

### Student/Academic
**Modified UI/UX:**
- Simplified particle domain setup with preset geometries (hopper, drum, conveyor) and standard material templates
- Animated 3D particle visualization with color-mapped velocity, force chains, and contact networks
- Interactive parameter panel with sliders for friction, restitution, and particle size to observe flow behavior changes

**Feature Gating:**
- DEM simulations limited to 50,000 particles with spherical shapes only
- Basic contact models (Hertz-Mindlin, linear spring-dashpot) available; bonded-particle and cohesion models locked
- Post-processing limited to built-in viewers; VTK/ParaView export gated

**Pricing Tier:** Free tier

**Onboarding Flow:**
- Tutorial: simulate granular discharge from a hopper, measure mass flow rate, and compare with Beverloo correlation
- Pre-loaded examples (rotating drum segregation, conveyor transfer, angle of repose test)
- Sidebar references linking DEM concepts to Cundall & Strack, Luding, and Thornton textbooks

**Key Workflows:**
- Simulate granular flow in a hopper and measure discharge rate and flow patterns
- Observe size segregation in a rotating drum with binary particle mixtures
- Conduct virtual angle-of-repose tests and calibrate friction parameters
- Visualize contact force chains and stress distribution in a granular bed

---

### Junior Engineer (0-3 years)
**Modified UI/UX:**
- Project workspace with tabs: Geometry | Particles | Contact Models | Simulation | Analysis
- Integrated material calibration wizard matching simulation to standard lab tests (angle of repose, shear cell)
- Real-time simulation monitor with particle count, time-step diagnostics, and energy balance tracking

**Feature Gating:**
- Particle count up to 2 million with multi-sphere (clump) shapes for non-spherical particles
- Full contact model library including cohesion (JKR, DMT), bonded-particle model
- DEM-CFD coupling for fluid-particle interaction limited to one-way; two-way coupling gated

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Import equipment CAD (hopper, mixer, crusher), define particle properties, and run first simulation
- Calibrate material model using virtual angle-of-repose and shear-cell test against lab measurements
- Analyze particle flow, wear patterns, and residence time distribution

**Key Workflows:**
- Simulate industrial equipment (hoppers, mixers, screens, crushers) with realistic particle shapes
- Calibrate DEM material parameters against laboratory test data (Schulze shear cell, angle of repose)
- Analyze equipment wear patterns from particle impact energy and sliding contact data
- Evaluate conveyor transfer chute designs for minimizing dust and spillage
- Assess mixing efficiency and segregation potential in blending equipment

---

### Senior/Expert
**Modified UI/UX:**
- Multi-simulation campaign manager with parametric study automation and optimization workflows
- Python scripting console with full DEM engine API for custom contact models and post-processing
- HPC job manager with GPU cluster scheduling and domain-decomposition configuration

**Feature Gating:**
- All features unlocked: unlimited particles (GPU-accelerated), full DEM-CFD two-way coupling, breakage models
- Custom contact model SDK and user-defined particle factories
- Multi-scale coupling (DEM-FEM for structural loads, DEM-CFD for fluid interaction, DEM-DEM for domain bridging)

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Migration wizard importing existing EDEM/Rocky project files and calibrated material databases
- GPU cluster configuration and API setup for automated simulation pipelines
- Workshop on custom contact model development and multi-physics coupling setup

**Key Workflows:**
- Large-scale industrial simulations (10M+ particles) of full processing plants with GPU acceleration
- Coupled DEM-CFD simulation of fluidized beds, pneumatic conveying, and spray granulation
- Particle breakage modeling for comminution equipment (jaw crusher, ball mill, impact crusher)
- Multi-objective equipment optimization varying geometry, speed, and particle properties
- Development of custom contact models for specialized materials (wet granular, fibrous, pharmaceutical)

---

## Role-Based Personas

### Designer/Creator
**Modified UI/UX:**
- Visual equipment design canvas with parametric geometry editing and instant particle-flow preview
- Design variant comparison view showing flow patterns, wear maps, and throughput side-by-side
- Integration with CAD tools (import STEP/IGES, export optimized geometry back)

**Feature Gating:**
- Parametric geometry tools and rapid-preview simulation mode enabled
- Full-fidelity simulation available for detailed validation of selected designs
- Export to CAD formats and automated design report generation

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Import equipment CAD, define operating conditions, and run rapid-preview particle simulation
- Identify problem areas (dead zones, high wear, spillage) and adjust geometry
- Compare before/after designs and export optimized geometry

**Key Workflows:**
- Design and optimize chute geometries for conveyor transfer points
- Evaluate hopper and silo outlet designs for reliable flow (mass flow vs. funnel flow)
- Optimize mixer blade geometry and speed for target mixing quality
- Design screening and classification equipment with particle trajectory analysis

---

### Analyst/Simulator
**Modified UI/UX:**
- Simulation-focused layout with advanced solver controls, mesh/domain settings, and convergence monitoring
- Comprehensive post-processing suite: velocity fields, residence time distributions, wear maps, force statistics
- Batch simulation manager for parametric sweeps and design-of-experiments studies

**Feature Gating:**
- Full solver access with all contact models, coupling options, and post-processing tools
- Advanced calibration tools (automated optimization, sensitivity analysis, Bayesian parameter estimation)
- API access for automated simulation pipelines and custom post-processing scripts

**Pricing Tier:** Professional or Enterprise tier

**Onboarding Flow:**
- Import geometry and particles, configure contact model, and run a validation simulation against experimental data
- Perform material calibration using automated optimization against multiple lab tests
- Set up a parametric study varying operating conditions and analyze sensitivity

**Key Workflows:**
- High-fidelity DEM simulation of complex industrial processes with validated material models
- DEM-CFD coupled simulations for fluid-particle systems (fluidized beds, cyclones, pneumatic transport)
- Particle breakage and attrition modeling with population balance tracking
- Wear prediction and lifetime estimation for equipment surfaces
- Parametric and optimization studies for equipment design and operating conditions

---

### Manufacturing/Process
**Modified UI/UX:**
- Process-focused interface showing equipment operating parameters, throughput, and quality metrics
- Integration panels for connecting to process control systems (DCS, SCADA, historian)
- Operating envelope visualization showing safe/optimal operating regions from simulation campaigns

**Feature Gating:**
- Pre-computed operating envelopes and look-up tables from simulation campaigns accessible
- Full DEM simulation tools hidden; simplified what-if calculators based on surrogate models
- Export to process control formats and integration with digital twin platforms

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Import equipment operating data and map to pre-calibrated simulation models
- Generate operating envelopes showing optimal parameter ranges for throughput and quality
- Connect to process historian for real-time comparison of actual vs. predicted performance

**Key Workflows:**
- Use simulation-derived operating envelopes to optimize process settings
- Troubleshoot flow problems (bridging, ratholing, segregation) using calibrated models
- Evaluate process changes (new feedstock, speed adjustment) before implementation
- Generate digital twin dashboards for real-time process monitoring

---

### Regulatory/Compliance
**Modified UI/UX:**
- Dust and emissions dashboard showing predicted dust generation rates and containment effectiveness
- Safety compliance checklist for dust explosion risk (ATEX, NFPA 652) linked to simulation evidence
- Environmental reporting tools for particulate emissions and containment measures

**Feature Gating:**
- Dust generation and dispersion analysis modules enabled
- Equipment containment and ventilation effectiveness assessment tools accessible
- Audit trail and simulation provenance tracking for regulatory documentation

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Select applicable regulations (ATEX, NFPA 652, EPA particulate emissions)
- Import process simulation results and run dust hazard assessment
- Generate compliance documentation with simulation-backed evidence

**Key Workflows:**
- Assess dust generation potential at conveyor transfers, chutes, and material handling points
- Evaluate containment and ventilation system effectiveness using DEM-CFD dust dispersion
- Generate documentation for dust explosion risk assessment (ATEX, NFPA compliance)
- Quantify fugitive particulate emissions for environmental permitting
- Maintain simulation audit trail for regulatory inspection readiness

---

### Manager/Decision-maker
**Modified UI/UX:**
- Equipment portfolio dashboard with performance KPIs (throughput, wear rate, downtime, product quality)
- Investment analysis cards comparing equipment modifications with cost-benefit projections
- Risk matrix linking simulation findings to operational and safety risks

**Feature Gating:**
- Read-only access to all simulation results and assessment reports
- ROI calculator for equipment modifications and process improvements
- Approval workflow for design changes and capital expenditure decisions

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Dashboard overview: equipment performance metrics, maintenance costs, and improvement opportunities
- Review scenario comparison tools for equipment upgrade decisions
- Set up notification preferences for wear threshold alerts and maintenance scheduling

**Key Workflows:**
- Evaluate ROI of equipment modifications (new chute design, liner material change) based on simulation predictions
- Compare equipment vendor alternatives using simulation-based performance assessment
- Review wear prediction data and approve predictive maintenance schedules
- Assess capital expenditure proposals for new processing equipment
- Monitor fleet-wide equipment performance and identify optimization opportunities
