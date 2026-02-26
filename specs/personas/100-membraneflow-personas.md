# MembraneFlow — Persona Subspecs

> Parent spec: [`specs/100-membraneflow.md`](../100-membraneflow.md)

---

## Experience-Level Personas

### Student/Academic

**Modified UI/UX:**
- Simplified single-stage membrane system builder with common configurations (single pass RO, UF dead-end)
- Real-time flux and rejection visualization as pressure, temperature, and feed composition are adjusted
- Theory panel explaining solution-diffusion model, concentration polarization, and fouling mechanisms

**Feature Gating:**
- Single-stage RO, NF, UF, or MF simulation with idealized feed water compositions
- Pre-built membrane database with common commercial membranes (limited to public datasheet parameters)
- Output limited to basic performance metrics: flux, rejection, recovery, and energy consumption

**Pricing Tier:** Free tier

**Onboarding Flow:**
- Tutorial: design a single-pass RO system for seawater desalination and calculate specific energy consumption
- Pre-loaded examples: brackish water RO, dairy UF, pharmaceutical NF, and MBR wastewater
- Educational mode annotating transport equations and mass balance calculations at each step

**Key Workflows:**
- Simulate single-stage membrane systems and calculate permeate quality and recovery
- Study the effect of pressure, temperature, and feed concentration on membrane performance
- Compare different membrane types (RO, NF, UF, MF) for a given separation challenge
- Analyze concentration polarization and its impact on flux and rejection

---

### Junior Engineer (0-3 years)

**Modified UI/UX:**
- System design wizard: input feed water analysis, target permeate quality, and production rate
- Multi-stage array builder with pass/stage/element configuration and inter-stage pump sizing
- Scaling and fouling risk indicator based on feed water chemistry and recovery setting

**Feature Gating:**
- Multi-stage and multi-pass system design with standard array configurations
- Chemical dosing calculator for antiscalant, pH adjustment, and cleaning chemicals
- Pretreatment selection assistant recommending conventional or membrane pretreatment

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Feed water analysis import wizard: lab results, ion balance verification, and saturation index calculation
- Guided system design: from water analysis to complete RO train with pretreatment and post-treatment
- Connection to membrane manufacturer databases for updated element performance data

**Key Workflows:**
- Design multi-stage RO/NF systems for brackish water, seawater, or industrial water treatment
- Perform scaling risk analysis (LSI, S&DSI, saturation ratios) and specify antiscalant dosing
- Size pretreatment systems: media filtration, UF, cartridge filters, and chemical conditioning
- Calculate system energy consumption and specify high-pressure pump and energy recovery devices
- Generate system design reports with flow diagrams, performance projections, and equipment lists

---

### Senior/Expert

**Modified UI/UX:**
- Advanced modeling environment with fouling prediction, long-term performance decay, and cleaning optimization
- Multi-train plant design workspace with shared intake, pretreatment, and distribution infrastructure
- Scripting API for parametric studies, optimization algorithms, and digital twin integration

**Feature Gating:**
- All modules unlocked: dynamic fouling models, organic/bio/colloidal fouling simulation, CIP optimization
- Hybrid system design: RO + ED, UF + RO, FO + RO, and membrane distillation configurations
- API for integration with plant SCADA, historian, and real-time performance monitoring

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Migration assistant for ROSA, TorayDS, WAVE, and IMSDesign project import
- Fouling model calibration wizard using operational performance data and autopsy results
- API configuration for plant data integration and digital twin deployment

**Key Workflows:**
- Design large-scale desalination plants with multiple trains, shared infrastructure, and redundancy
- Develop fouling prediction models calibrated to site-specific feed water and operating history
- Optimize cleaning-in-place (CIP) schedules balancing membrane life, downtime, and chemical costs
- Design hybrid membrane systems for challenging separations (ZLD, brine concentration, organic removal)
- Operate digital twins for real-time performance optimization and predictive maintenance

---

## Role-Based Personas

### Designer/Creator

**Modified UI/UX:**
- Visual system schematic builder with drag-and-drop elements, pumps, valves, and instruments
- Pressure vessel configurator: element count, housing material, and end-cap type selection
- P&ID auto-generator from system schematic with instrument and valve specifications

**Feature Gating:**
- Full system design tools: arrays, passes, stages, blending, and recirculation loops
- Hydraulic design: pipe sizing, pump selection, and pressure drop calculation through system
- Skid layout design with 3D spatial arrangement and piping routing

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Application selector: desalination, industrial water, wastewater reuse, or process separation
- Membrane element browser with filterable catalog by manufacturer, type, and size
- System template gallery for common configurations (2:1 array, concentrate staging, permeate staging)

**Key Workflows:**
- Design complete membrane treatment systems from intake to product water distribution
- Configure pressure vessel arrays optimizing flux distribution and recovery per stage
- Specify instrumentation and control for automated operation: flow, pressure, conductivity, pH
- Design chemical dosing systems for pretreatment, CIP, and post-treatment
- Generate construction-ready P&IDs and equipment specifications for procurement

---

### Analyst/Simulator

**Modified UI/UX:**
- Simulation workspace with element-by-element performance calculation and profile visualization
- Fouling and scaling progression simulator with time-step modeling over months of operation
- Sensitivity analysis dashboard for critical parameters: temperature, feed TDS, recovery, flux

**Feature Gating:**
- Full simulation engine: steady-state, time-dependent fouling, and dynamic transient analysis
- Advanced transport models: extended Nernst-Planck for NF, pore flow for UF/MF
- Thermodynamic models for scaling prediction: activity coefficient methods, saturation indices

**Pricing Tier:** Professional or Enterprise tier

**Onboarding Flow:**
- Model configuration tutorial: membrane transport parameters, fouling coefficients, and feed characterization
- Validation workflow comparing predictions to membrane manufacturer projection software
- Operational data import for model calibration against installed system performance

**Key Workflows:**
- Perform element-by-element simulation predicting flux, rejection, and pressure profiles
- Model long-term performance decline due to fouling, compaction, and membrane aging
- Analyze scaling risk at tail elements under varying recovery and temperature conditions
- Predict system performance under seasonal feed water quality variations
- Validate simulation models against pilot plant data and optimize design parameters

---

### Manufacturing/Process

**Modified UI/UX:**
- Operations dashboard: normalized performance metrics, CIP tracking, and membrane replacement scheduling
- Commissioning procedure generator with step-by-step startup sequences and acceptance criteria
- Spare parts and consumables inventory tracker with reorder triggers

**Feature Gating:**
- Operational performance normalization tools (ASTM D4516) for trend analysis
- CIP recipe management: chemical types, concentrations, temperatures, and soak durations
- Membrane element lifecycle tracking: installation date, operating hours, normalized performance

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Installed system configuration input: membrane type, array, operating parameters, and design basis
- Performance baseline establishment from commissioning data
- CIP chemical and procedure library setup based on membrane manufacturer guidelines

**Key Workflows:**
- Monitor normalized system performance and identify fouling onset or membrane degradation
- Manage CIP schedules and track cleaning effectiveness through performance recovery metrics
- Plan membrane replacement strategies: full replacement, lead element rotation, or staged swap
- Track chemical consumption: antiscalant, cleaning chemicals, and post-treatment additives
- Generate operational reports with performance trends, chemical usage, and maintenance records

---

### Regulatory/Compliance

**Modified UI/UX:**
- Water quality compliance dashboard mapping permeate quality to regulatory standards (EPA, WHO, EU)
- Concentrate discharge compliance tracker: volume, TDS, specific ion concentrations, and toxicity
- Chemical handling compliance: SDS management, storage requirements, and dosing record keeping

**Feature Gating:**
- Permeate quality verification against drinking water standards (MCLs, guideline values)
- Concentrate disposal analysis: ocean outfall, deep well injection, evaporation, and ZLD evaluation
- Membrane and chemical NSF/ANSI 61 certification verification for potable water applications

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Regulatory jurisdiction selector: applicable water quality standards and discharge limits
- Product water quality target configuration with regulatory limit margin management
- Concentrate disposal permit requirements and monitoring parameter setup

**Key Workflows:**
- Verify permeate quality compliance with drinking water standards at design and off-design conditions
- Evaluate concentrate disposal options and compliance with discharge permit limits
- Ensure all materials in contact with product water meet NSF/ANSI 61 certification requirements
- Track and report water quality monitoring data for regulatory compliance
- Prepare permit applications with system design, water quality projections, and monitoring plans

---

### Manager/Decision-maker

**Modified UI/UX:**
- Plant portfolio dashboard: production capacity, availability, specific energy, and cost per cubic meter
- Life cycle cost analysis tools: CAPEX, OPEX, membrane replacement, chemical, and energy costs
- Technology comparison matrix for procurement decisions and expansion planning

**Feature Gating:**
- Read-only access to all system designs, simulation results, and compliance documentation
- Financial modeling: LCOW (levelized cost of water), NPV, and sensitivity analysis
- Capacity planning and expansion scenario modeling for growing demand

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Plant portfolio import with installed capacity, operating history, and cost data
- Cost model configuration: energy tariffs, chemical prices, membrane replacement costs, labor rates
- Dashboard KPI setup: production volume, availability, specific energy, cost per m3, water quality

**Key Workflows:**
- Evaluate membrane technology alternatives by lifecycle cost, footprint, and reliability
- Monitor plant portfolio performance against design basis and budget targets
- Make procurement decisions: membrane brand selection, chemical supplier evaluation
- Plan capacity expansion or upgrade projects with cost-benefit analysis
- Generate management reports with production, cost, and compliance performance summaries
