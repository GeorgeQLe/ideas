# ProChem — Persona Subspecs

> Parent spec: [`specs/62-prochem.md`](../62-prochem.md)

---

## Experience-Level Personas

### Student/Academic

**Modified UI/UX:**
- Guided flowsheet builder with a curated unit operation palette (mixer, heater, flash, distillation column, reactor) and inline explanations of each
- Interactive thermodynamics advisor that recommends equation of state (Peng-Robinson, NRTL, UNIQUAC) based on the chemical system and explains why
- Pre-built example flowsheets for common textbook problems (benzene-toluene distillation, ammonia synthesis, CSTR cascades)

**Feature Gating:**
- Steady-state simulation with up to 20 unit operations; dynamic simulation disabled
- Built-in component database (500 most common chemicals); custom component regression tools locked
- Export to PDF reports for coursework; no PFD/P&ID export to CAD formats

**Pricing Tier:** Free tier (academic email verification)

**Onboarding Flow:**
- Start with a guided ethanol-water distillation tutorial that covers flowsheet building, thermodynamics selection, and convergence
- Interactive quiz matching chemical systems to appropriate thermodynamic models
- Access to a shared library of professor-curated homework templates

**Key Workflows:**
- Building flowsheets for unit operations coursework (distillation, absorption, extraction)
- Comparing thermodynamic model predictions for VLE data against experimental values
- Designing simple reactor networks (CSTR, PFR) with kinetics for reaction engineering courses
- Performing mass and energy balances for capstone plant design projects

---

### Junior Engineer (0-3 years)

**Modified UI/UX:**
- Template-based flowsheet creation with industry-standard process configurations (crude distillation, amine treating, steam methane reforming)
- Convergence assistant that diagnoses and suggests fixes for common convergence failures (tear stream initial guesses, spec-controller conflicts)
- Annotated results viewer linking simulation outputs to engineering design standards (TEMA for heat exchangers, GPSA for gas processing)

**Feature Gating:**
- Full steady-state simulation with unlimited unit operations; dynamic simulation limited to simple controllers
- Equipment sizing modules (columns, heat exchangers, pumps) enabled; rigorous rating mode requires senior review
- Custom component creation available with guided property estimation (Joback, UNIFAC)

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Select industry (refining, petrochemicals, specialty chemicals, pharmaceuticals) for tailored template library
- Guided build of an industry-relevant process (e.g., MEA CO2 capture loop) with embedded best practices
- Automatic convergence diagnostic run on first custom flowsheet with annotated improvement suggestions

**Key Workflows:**
- Building process models for feasibility studies and front-end engineering design (FEED)
- Sizing columns, heat exchangers, and vessels using built-in equipment design tools
- Running case studies (feed composition changes, throughput variations) for process optimization
- Generating heat and mass balance tables for P&ID development
- Evaluating energy integration opportunities with pinch analysis tools

---

### Senior/Expert

**Modified UI/UX:**
- Full scripting interface (Python/MATLAB) for custom unit operations, property methods, and optimization routines
- Multi-flowsheet workspace with inter-process recycles and plant-wide material balance integration
- Direct database access to modify thermodynamic interaction parameters and regress custom VLE/LLE data

**Feature Gating:**
- All features unlocked: dynamic simulation, electrolyte thermodynamics, solids handling, reactive distillation
- Custom equation of state implementation and parameter regression from experimental data
- Process optimization (MINLP) and heat exchanger network synthesis modules

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Import existing Aspen Plus, HYSYS, or CHEMCAD models with automated parameter mapping and validation
- Benchmark suite comparing ProChem predictions against plant data for thermodynamic model validation
- API setup for integration with plant data historians and control system simulators

**Key Workflows:**
- Developing rigorous process models for complex separations (extractive distillation, reactive distillation, divided-wall columns)
- Regressing custom thermodynamic parameters from proprietary experimental VLE/LLE data
- Running dynamic simulations for control strategy development and safety analysis (relief sizing)
- Plant-wide optimization across multiple interconnected process units
- Mentoring junior engineers by reviewing simulation setups and thermodynamic model selections

---

## Role-Based Personas

### Designer/Creator

**Modified UI/UX:**
- Visual flowsheet canvas with drag-and-drop unit operations, intelligent stream routing, and auto-layout
- Integrated PFD/P&ID drafting mode that generates engineering drawings from simulation topology
- Design variant manager for comparing alternative process configurations side by side

**Feature Gating:**
- Full flowsheet creation and modification with parametric process templates
- Equipment sizing and selection tools with vendor catalog integration
- Utility system design (steam, cooling water, refrigeration) with automatic balancing

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Start by importing a process concept sketch or selecting a template process configuration
- Walk through converting a block flow diagram into a rigorous simulation model
- Demonstrate the design-to-P&ID pipeline with automated equipment tagging

**Key Workflows:**
- Conceptual process design with rapid evaluation of alternative configurations
- Detailed equipment sizing (distillation columns, heat exchangers, reactors) from simulation results
- Heat integration using pinch analysis and heat exchanger network synthesis
- Generating PFDs and preliminary P&IDs from converged flowsheets
- Evaluating process intensification options (reactive distillation, dividing wall columns)

---

### Analyst/Simulator

**Modified UI/UX:**
- Analysis-centric workspace with convergence diagnostics, sensitivity plots, and optimization panels prominent
- Multi-case comparison table with sortable/filterable results across hundreds of parametric runs
- Uncertainty propagation interface with Monte Carlo and polynomial chaos expansion options

**Feature Gating:**
- All simulation and analysis tools available based on license tier
- Advanced optimization (SQP, genetic algorithm, MINLP) and sensitivity analysis modules
- Batch simulation queue with parametric sweep configuration

**Pricing Tier:** Professional or Enterprise tier

**Onboarding Flow:**
- Tutorial on diagnosing convergence issues using the built-in troubleshooting advisor
- Guided sensitivity analysis on a sample distillation column (reflux ratio, feed stage, pressure)
- Introduction to optimization workflows with objective function setup and constraint definition

**Key Workflows:**
- Running rigorous process simulations with detailed thermodynamic models for design verification
- Performing sensitivity analyses on key process variables to identify optimal operating windows
- Optimizing process conditions to minimize energy consumption, maximize yield, or reduce waste
- Validating simulation models against pilot plant or commercial plant data
- Running Monte Carlo simulations to quantify uncertainty in process performance predictions

---

### Manufacturing/Process

**Modified UI/UX:**
- Operator-focused interface with live process schematic overlaid with real-time DCS data
- Digital twin mode showing current vs. optimal operating conditions with deviation highlighting
- Alarm integration panel flagging operating conditions that deviate from simulated safe envelope

**Feature Gating:**
- Steady-state and dynamic models calibrated to the specific plant; no flowsheet modification
- What-if scenario runner for feed changes, throughput adjustments, and equipment outages
- Real-time optimization recommendations based on current operating data

**Pricing Tier:** Enterprise tier (operations license)

**Onboarding Flow:**
- Connect to plant DCS/historian (DeltaV, Honeywell, ABB) for automatic data population
- Calibrate process model against current plant performance data during commissioning
- Train operators on interpreting digital twin recommendations and scenario analysis

**Key Workflows:**
- Monitoring plant performance against simulated optimal conditions in real-time
- Evaluating feed composition changes (new crude slate, different feedstock) before implementation
- Troubleshooting process upsets using the digital twin to isolate root causes
- Planning turnaround activities by simulating equipment isolation and startup sequences
- Optimizing energy consumption by adjusting column reflux, furnace duty, and heat recovery

---

### Regulatory/Compliance

**Modified UI/UX:**
- Compliance dashboard organized by regulation (EPA, REACH, OSHA PSM) with auto-populated parameters from simulation
- Material safety data integration showing hazardous chemical inventories and threshold quantities
- Audit-ready report templates for environmental permits, PSM documentation, and REACH registration

**Feature Gating:**
- Read-only access to simulation results and material balances; no model modification
- Automated emissions calculations (VOC, HAP, greenhouse gas) from process vent streams
- Waste stream characterization and RCRA classification from simulation composition data

**Pricing Tier:** Enterprise tier (compliance add-on)

**Onboarding Flow:**
- Select applicable regulatory frameworks to auto-configure compliance checking thresholds
- Walk through generating an emissions inventory from a converged process simulation
- Set up automated alerts for process changes that affect regulatory permit conditions

**Key Workflows:**
- Generating emissions inventories for Title V and state air quality permit applications
- Calculating process safety information (PSI) for OSHA PSM compliance from simulation data
- Documenting material balances for REACH registration and chemical inventory reporting
- Evaluating process modifications for environmental impact before implementation
- Producing waste minimization reports comparing alternative process configurations

---

### Manager/Decision-maker

**Modified UI/UX:**
- Portfolio dashboard showing all active process design projects with status, timeline, and key outcomes
- Techno-economic summary view with CAPEX estimates, OPEX projections, and NPV/IRR calculations from simulation
- Resource utilization dashboard showing simulation compute costs and engineering hours per project

**Feature Gating:**
- View-only access to project summaries, cost estimates, and key performance metrics
- Comparative analysis tools for technology selection (process route A vs. B vs. C)
- Budget tracking with compute cost allocation across projects and departments

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Configure project portfolio views linked to active process design engagements
- Walk through interpreting techno-economic analysis outputs from simulation results
- Set up automated reporting for project milestones and budget utilization

**Key Workflows:**
- Reviewing techno-economic comparisons of alternative process routes for investment decisions
- Tracking process design project progress against FEED milestones and budget
- Approving technology selections based on simulation-derived performance and cost data
- Generating executive summaries for board presentations on capital project feasibility
- Monitoring engineering team productivity and simulation platform utilization metrics
