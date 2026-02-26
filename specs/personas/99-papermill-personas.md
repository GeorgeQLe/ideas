# PaperMill — Persona Subspecs

> Parent spec: [`specs/99-papermill.md`](../99-papermill.md)

---

## Experience-Level Personas

### Student/Academic

**Modified UI/UX:**
- Simplified process flow with major pulp and paper unit operations: digester, washer, bleach plant, paper machine
- Animated fiber-level visualizations showing pulping chemistry, washing efficiency, and sheet formation
- Theory panel linking each unit operation to Kappa number kinetics, freeness, and drainage fundamentals

**Feature Gating:**
- Single-section simulation (e.g., Kraft digester or paper machine wet end); full mill locked
- Pre-built pulp property models with common wood species (softwood, hardwood, eucalyptus)
- Output limited to mass/energy balance reports; no economic or environmental optimization

**Pricing Tier:** Free tier

**Onboarding Flow:**
- Tutorial: simulate a Kraft cooking process and predict Kappa number from H-factor
- Pre-loaded examples: bleaching sequence (D0-EOP-D1), Fourdrinier paper machine, tissue machine
- Educational mode with step-by-step mass balance annotations and chemical engineering fundamentals

**Key Workflows:**
- Simulate Kraft pulping: chip properties, liquor charge, and cooking conditions to predict yield and Kappa
- Analyze bleaching sequences: chemical charges, pH, temperature, and brightness development
- Model paper machine wet end: headbox consistency, drainage, and retention chemistry
- Calculate basic mass and energy balances for individual pulp and paper unit operations

---

### Junior Engineer (0-3 years)

**Modified UI/UX:**
- Mill section wizards: select mill type (Kraft, mechanical, recycled), grade, and capacity for auto-configured baseline
- Process variable trending with control chart overlay for key parameters
- Troubleshooting assistant suggesting root causes for common process deviations

**Feature Gating:**
- Full single-section simulation with dynamic response capability
- Stock preparation and paper machine simulation with grade change modeling
- Chemical recovery basics: evaporator train, recovery boiler mass balance (not detailed combustion)

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Mill section selector with template process flow and typical operating parameters
- Guided simulation: pulp mill mass balance from wood yard to brown stock
- Connection to mill historian data for model validation and calibration

**Key Workflows:**
- Model complete fiber lines from wood handling through pulp washing and screening
- Simulate paper machine operation: headbox, forming, pressing, and drying sections
- Analyze stock preparation: refining, cleaning, and approach system configuration
- Evaluate grade change strategies minimizing transition time and off-spec production
- Generate process variable targets and alarm limits for operational guidance

---

### Senior/Expert

**Modified UI/UX:**
- Full integrated mill simulation: fiber line, chemical recovery, power plant, and water systems
- Advanced modeling: fiber morphology effects, two-phase flow in digesters, CFD in headboxes
- Optimization engine with multi-objective capability: quality, cost, energy, and environmental metrics

**Feature Gating:**
- All modules unlocked: detailed chemical recovery (evaporators, recovery boiler, recausticizing, lime kiln)
- Dynamic mill-wide simulation with process control system interaction
- API for real-time data integration, digital twin operation, and advanced process control

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Migration assistant for WinGEMS, CADSIM, and custom mill models
- Mill historian data integration for model calibration and validation
- Digital twin configuration for real-time operation support

**Key Workflows:**
- Build integrated mill models connecting fiber line, recovery, utilities, and effluent treatment
- Optimize mill-wide energy balance: steam distribution, co-generation, and turbine loading
- Simulate chemical recovery cycle: evaporation, combustion, causticizing, and lime reburning
- Develop digital twin applications for real-time process optimization and predictive control
- Evaluate major capital projects: new digesters, shoe presses, or recovery boiler upgrades

---

## Role-Based Personas

### Designer/Creator

**Modified UI/UX:**
- Process and instrumentation diagram editor with pulp/paper-specific equipment symbols and piping standards
- Equipment sizing tools: digesters, washers, refiners, paper machines, and evaporators
- 3D layout visualization for mill section arrangement and piping routing

**Feature Gating:**
- Full process design tools with equipment sizing and specification generation
- Piping sizing, valve selection, and pump curve analysis for pulp/paper fluids
- Control system architecture design: DCS/PLC configuration and loop tuning tools

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Mill type and grade selector to auto-generate baseline process configuration
- Equipment database browser with vendor specifications for pulp/paper machinery
- P&ID template gallery following ISA and TAPPI standards

**Key Workflows:**
- Design new pulp and paper process lines from raw material to finished product
- Size and specify major equipment: digesters, paper machines, evaporators, and recovery boilers
- Create piping and instrumentation diagrams with control loop specifications
- Design chemical preparation and distribution systems (bleaching, sizing, retention)
- Generate equipment specification sheets and data sheets for procurement

---

### Analyst/Simulator

**Modified UI/UX:**
- Simulation workspace with multi-section mill model and steady-state/dynamic solver selection
- Fiber property tracking through entire process: length, coarseness, freeness, and strength development
- Statistical process analysis tools: capability studies, correlation analysis, and PCA for process data

**Feature Gating:**
- Full simulation capability: steady-state, dynamic, and hybrid models
- Fiber quality models linking wood properties through process to final paper properties
- Advanced chemical models: delignification kinetics, bleaching chemistry, and retention/drainage

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Mill historian data import and signal mapping to simulation model variables
- Model calibration workflow using operating data and lab measurements
- Fiber property model configuration for site-specific wood furnish

**Key Workflows:**
- Build and calibrate detailed mill section models against plant operating data
- Simulate process disturbances and evaluate control system response and recovery
- Analyze fiber quality development from wood chip to finished paper properties
- Predict paper machine runnability and quality impacts from upstream process changes
- Conduct debottlenecking studies identifying capacity-limiting constraints

---

### Manufacturing/Process

**Modified UI/UX:**
- Production monitoring dashboard with real-time KPIs: production rate, yield, energy, chemical consumption
- Grade scheduling tool with transition management and machine time optimization
- Downtime tracking and root cause analysis interface with Pareto visualization

**Feature Gating:**
- Production planning and scheduling with grade optimization and inventory management
- Real-time process advisory system suggesting setpoint adjustments for target quality
- Predictive maintenance tools using process variable trends and equipment vibration data

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Grade mix configuration with target properties, machine speeds, and basis weights
- Production KPI baseline establishment from historical operating data
- Integration with mill DCS/historian and quality measurement systems

**Key Workflows:**
- Optimize production scheduling across grade mix for maximum machine efficiency
- Monitor and troubleshoot real-time process deviations using model-based diagnostics
- Manage grade changes minimizing transition time and off-specification production
- Track chemical consumption and fiber yield to identify optimization opportunities
- Plan maintenance shutdowns coordinating with inventory levels and customer commitments

---

### Regulatory/Compliance

**Modified UI/UX:**
- Environmental compliance dashboard: effluent BOD/COD, air emissions (TRS, PM, NOx), and solid waste
- Permit limit tracking with trend analysis and exceedance prediction alerts
- Chain-of-custody certification tracker for FSC/PEFC and sustainable sourcing documentation

**Feature Gating:**
- Effluent treatment simulation: primary/secondary treatment, discharge quality prediction
- Air emission estimation: recovery boiler, lime kiln, and power boiler emission factors
- Environmental reporting generators for EPA, state agencies, and international authorities

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Permit limit configuration: discharge limits, air emission caps, and reporting frequencies
- Effluent treatment plant model setup calibrated to current performance
- Certification body requirements loading (FSC, PEFC, SFI) for chain-of-custody tracking

**Key Workflows:**
- Monitor effluent quality against discharge permit limits and predict exceedances
- Track and report air emissions from recovery boiler, lime kiln, and power plant operations
- Model effluent treatment performance under varying mill operating conditions
- Maintain chain-of-custody documentation for certified fiber sourcing
- Prepare environmental compliance reports and respond to regulatory inquiries

---

### Manager/Decision-maker

**Modified UI/UX:**
- Mill performance dashboard: production volume, operating cost per ton, energy intensity, and quality metrics
- Capital project portfolio tracker with ROI projections and implementation status
- Market scenario modeler linking grade demand, pricing, and raw material costs to profitability

**Feature Gating:**
- Read-only access to all process models, simulation results, and compliance data
- Financial modeling: production costing, margin analysis, and capital investment evaluation
- Benchmarking tools comparing mill performance against industry data

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Mill financial data import: cost structure, revenue by grade, and production history
- Industry benchmark database connection for performance comparison
- Dashboard KPI setup: cost per ton, energy per ton, yield, availability, and margin by grade

**Key Workflows:**
- Review mill operating performance against budget and industry benchmarks
- Evaluate capital investment proposals: ROI, payback, and impact on operating costs
- Analyze grade mix optimization for maximum profitability under market conditions
- Assess strategic options: capacity expansion, grade conversion, or asset rationalization
- Generate board-level reports on mill performance, investment, and strategic direction
