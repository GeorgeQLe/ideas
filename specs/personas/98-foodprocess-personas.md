# FoodProcess — Persona Subspecs

> Parent spec: [`specs/98-foodprocess.md`](../98-foodprocess.md)

---

## Experience-Level Personas

### Student/Academic

**Modified UI/UX:**
- Simplified process flow builder with drag-and-drop unit operations (mixing, heating, cooling, drying, fermentation)
- Animated unit operation visualizations showing temperature, moisture, and composition changes
- Theory sidebar linking each operation to food engineering principles (heat/mass transfer, rheology, kinetics)

**Feature Gating:**
- Single unit operation simulation; full plant-wide process integration locked
- Pre-built food property database (water activity, thermal conductivity, viscosity) with limited editing
- Batch process only; continuous process simulation and scheduling locked

**Pricing Tier:** Free tier

**Onboarding Flow:**
- Tutorial: simulate pasteurization of milk -- heat exchanger sizing, time-temperature profile, lethality calculation
- Pre-loaded examples: spray drying of coffee, yogurt fermentation, bread baking
- Campus license with integration to food science and engineering course platforms

**Key Workflows:**
- Simulate individual unit operations: pasteurization, sterilization, evaporation, drying
- Study the effect of process parameters on product quality (nutrient retention, texture, color)
- Calculate thermal lethality (F-value, D-value, z-value) for sterilization process design
- Analyze energy and mass balances for single-unit food processes

---

### Junior Engineer (0-3 years)

**Modified UI/UX:**
- Process flow diagram builder with standard food industry templates (dairy, beverage, bakery, meat)
- Equipment sizing assistants for heat exchangers, evaporators, dryers, and mixing vessels
- CIP (clean-in-place) cycle designer with chemical concentration and temperature sequencing

**Feature Gating:**
- Full process flow simulation for batch and semi-continuous operations
- Equipment sizing and selection from vendor catalog databases
- Basic energy optimization with heat integration (pinch analysis) for utility reduction

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Industry template selector: dairy, beverage, confectionery, meat, snack, or frozen food
- Guided process design: define product recipe, select unit operations, size equipment
- Firm recipe database connection with standard formulations and processing parameters

**Key Workflows:**
- Design complete processing lines from raw material receiving to finished product packaging
- Size heat exchangers, evaporators, and dryers for specified production throughput
- Simulate batch processes with scheduling: mixing, cooking, cooling, filling cycles
- Design CIP systems with chemical selection, flow rates, and temperature profiles
- Generate process flow diagrams and equipment specifications for procurement

---

### Senior/Expert

**Modified UI/UX:**
- Advanced simulation environment with coupled heat/mass/momentum transfer and reaction kinetics
- Product quality prediction models: texture, color, flavor compound retention, microbial inactivation
- Plant-wide optimization workspace with energy integration, water recovery, and waste minimization

**Feature Gating:**
- All modules unlocked: CFD for mixers/dryers, population balance for crystallization, fermentation kinetics
- Product quality model calibration from pilot plant and production data
- API for integration with ERP, MES, and real-time process control systems

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Migration assistant for existing process models (Aspen, SuperPro Designer, custom spreadsheets)
- Product quality model calibration wizard using plant trial data
- API configuration for real-time data integration with SCADA and MES systems

**Key Workflows:**
- Develop predictive quality models linking process parameters to product attributes
- Optimize plant-wide energy and water usage through process integration analysis
- Simulate novel processing technologies: high-pressure processing, pulsed electric fields, ohmic heating
- Design scale-up strategies from pilot to commercial scale with quality consistency assurance
- Integrate real-time process data for digital twin operation and predictive control

---

## Role-Based Personas

### Designer/Creator

**Modified UI/UX:**
- Process line layout editor with 3D factory floor visualization and equipment spatial arrangement
- Recipe formulation workspace linking ingredient properties to process behavior
- Packaging line integration: filling speed, container format, and labeling configuration

**Feature Gating:**
- Full process design tools: flow sheets, P&IDs, equipment specifications
- Hygienic design compliance checking (3-A, EHEDG standards)
- Plant layout tools with material flow analysis and contamination zone separation

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Product type and production volume input to auto-suggest process line configuration
- Equipment vendor catalog browser with hygienic design certification filters
- Facility layout template gallery organized by food sector and production scale

**Key Workflows:**
- Design processing lines for new product development from recipe to production specification
- Configure equipment trains: raw material handling, processing, packaging, and palletizing
- Ensure hygienic design compliance: material flow, zone separation, CIP coverage
- Create piping and instrumentation diagrams with control loop specifications
- Generate equipment specification sheets for vendor quotation and procurement

---

### Analyst/Simulator

**Modified UI/UX:**
- Simulation dashboard with multi-physics model configuration and transient response tracking
- Product quality prediction overlays on process flow: microbial load, nutrient retention, texture indices
- Statistical analysis tools for designed experiments and process capability assessment

**Feature Gating:**
- Full simulation solvers: steady-state and dynamic, batch scheduling with resource constraints
- Microbial inactivation and growth kinetics (predictive microbiology models)
- Shelf-life prediction using Arrhenius kinetics and accelerated aging models

**Pricing Tier:** Professional or Enterprise tier

**Onboarding Flow:**
- Model configuration tutorial: food property estimation, reaction kinetics setup, boundary conditions
- Validation workflow comparing simulation results to pilot plant or production data
- Predictive microbiology model selection guide for target organisms and food matrices

**Key Workflows:**
- Simulate thermal processes and validate lethality against regulatory requirements
- Model moisture migration, water activity changes, and shelf-life during storage
- Predict microbial growth and inactivation under dynamic temperature conditions
- Optimize batch scheduling to maximize throughput with resource constraints
- Perform sensitivity analysis on critical process parameters for robustness evaluation

---

### Manufacturing/Process

**Modified UI/UX:**
- Production scheduling dashboard with batch tracking, changeover management, and OEE metrics
- Real-time process monitoring interface with alarm configuration and trend analysis
- Material balance tracker: raw material consumption, yield, waste, and rework percentages

**Feature Gating:**
- Production scheduling and batch management tools with MES integration
- OEE (Overall Equipment Effectiveness) calculation and bottleneck identification
- Waste stream analysis and reduction optimization tools

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Production line configuration: equipment capacities, changeover times, cleaning schedules
- Shift pattern and crew assignment setup
- MES/SCADA data connection for real-time production monitoring

**Key Workflows:**
- Schedule production runs optimizing for throughput, changeover minimization, and freshness
- Track material balances and identify yield loss sources across the processing line
- Monitor OEE metrics and identify bottleneck operations for capacity improvement
- Manage allergen changeover sequences and verify CIP effectiveness
- Plan preventive maintenance windows minimizing production disruption

---

### Regulatory/Compliance

**Modified UI/UX:**
- HACCP plan builder with hazard analysis, CCP identification, and monitoring procedure definition
- Regulatory compliance matrix: FDA 21 CFR, EU regulations, Codex Alimentarius
- Allergen management dashboard with cross-contact risk assessment and labeling verification

**Feature Gating:**
- HACCP/HARPC plan development and documentation tools
- Thermal process filing tools for FDA (2541) and USDA (9 CFR 318/381) submissions
- Nutritional labeling calculator with regulatory format compliance (FDA, EU, Codex)

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Regulatory market selector: FDA, EU, Codex, or multi-jurisdiction compliance requirements
- HACCP team and prerequisite program setup
- Product and allergen database initialization for labeling and cross-contact management

**Key Workflows:**
- Develop HACCP plans with hazard analysis, CCP determination, and critical limit specification
- Prepare thermal process filings with lethality calculations and documentation for FDA/USDA
- Manage allergen cross-contact risk assessments and control measures
- Generate nutrition facts panels and ingredient statements per regulatory format requirements
- Conduct food safety audits and track corrective action completion (BRC, SQF, FSSC 22000)

---

### Manager/Decision-maker

**Modified UI/UX:**
- Plant performance dashboard: production output, quality metrics, waste, energy, and cost per unit
- New product development pipeline tracker with stage-gate progress and resource allocation
- Capacity planning visualization with demand forecasts and expansion scenario analysis

**Feature Gating:**
- Read-only access to all process designs, simulation results, and compliance documentation
- Financial analysis: production cost modeling, margin analysis, and investment evaluation
- Capacity planning and expansion scenario modeling tools

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Plant data import: production history, cost structure, and quality metrics
- Financial model configuration: ingredient costs, labor rates, utilities, and overhead allocation
- Dashboard KPI setup: throughput, yield, quality rate, energy intensity, cost per unit

**Key Workflows:**
- Monitor plant-level KPIs: production efficiency, quality metrics, and cost performance
- Evaluate capital investment proposals for new lines, equipment upgrades, or plant expansion
- Review new product development feasibility based on process capability and cost projections
- Benchmark production performance across multiple plants or production lines
- Generate management reports with production, quality, and financial trend analysis
