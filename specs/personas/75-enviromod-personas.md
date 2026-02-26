# EnviroMod — Persona Subspecs

> Parent spec: [`specs/75-enviromod.md`](../75-enviromod.md)

---

## Experience-Level Personas

### Student/Academic
**Modified UI/UX:**
- Interactive aquifer cross-section builder with drag-and-drop geological layers and annotated flow concepts
- Visual groundwater flow simulator with particle tracking and real-time head contour display
- Simplified contaminant transport setup using common pollutant presets (BTEX, TCE, nitrate)

**Feature Gating:**
- 2D steady-state and transient groundwater flow modeling unlocked
- Basic advection-dispersion transport with first-order decay; reactive transport and NAPL modeling locked
- Model domain limited to moderate grid sizes; parallel solvers and large-scale models locked

**Pricing Tier:** Free tier (academic license)

**Onboarding Flow:**
- Tutorial: build a simple confined aquifer model, set boundary conditions, solve for head distribution
- Pre-loaded textbook examples (Theis test, plume migration from a point source)
- Guided exercise comparing analytical solutions to numerical model results for validation

**Key Workflows:**
- Building conceptual groundwater flow models from hydrogeological cross-sections
- Simulating contaminant plume migration from point sources for environmental science coursework
- Comparing numerical results to analytical solutions (Theis, Ogata-Banks) for model verification
- Generating head contour maps and flow path visualizations for lab reports

### Junior Engineer (0-3 years)
**Modified UI/UX:**
- Template-based project setup for common scenarios (landfill assessment, pump-and-treat, dewatering)
- Guided boundary condition assignment with context-aware suggestions based on hydrogeological setting
- Model quality checkers flagging water balance errors, convergence issues, and unrealistic parameters

**Feature Gating:**
- Full 3D groundwater flow and advection-dispersion transport modeling
- Reactive transport with biodegradation kinetics for common contaminants
- MODPATH particle tracking available; density-dependent flow and unsaturated zone modeling restricted

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Role-based setup asking focus area: site assessment, remediation design, water supply, or construction dewatering
- Walkthrough: import site data (boring logs, well data), build model, calibrate to observed heads, simulate plume
- Introduction to report generation tools and regulatory submission formatting

**Key Workflows:**
- Building 3D site-scale groundwater models from boring logs and geophysical data
- Calibrating models against observed water levels and contaminant concentrations
- Simulating contaminant plume migration for risk assessment and remediation timeframe estimation
- Designing and optimizing pump-and-treat well networks for remediation
- Generating regulatory-compliant modeling reports with standardized formatting

### Senior/Expert
**Modified UI/UX:**
- Multi-model coupling console linking groundwater, surface water, vadose zone, and reactive transport modules
- Advanced parameter estimation interface with PEST/PEST++ integration for automated calibration
- Scripting console for custom model workflows, uncertainty quantification, and stochastic simulations

**Feature Gating:**
- All modules: density-dependent flow, multiphase NAPL, reactive transport, unsaturated zone, surface water coupling
- Full API for integration with GIS platforms, monitoring databases, and real-time sensor networks
- Admin controls for project templates, model review standards, and quality assurance procedures

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Migration assistant for importing MODFLOW/MT3DMS/FEFLOW models and converting legacy model files
- Configuration of automated calibration pipelines with uncertainty quantification frameworks
- Setup of real-time monitoring integration and automated model updating workflows

**Key Workflows:**
- Building regional-scale models with complex hydrostratigraphy and boundary condition coupling
- Performing stochastic simulations with Monte Carlo parameter sampling for uncertainty analysis
- Coupling groundwater models with surface water, vadose zone, and atmospheric deposition models
- Developing and calibrating reactive transport models for complex geochemical systems
- Managing multi-consultant modeling projects with version control and peer review workflows

---

## Role-Based Personas

### Designer/Creator
**Modified UI/UX:**
- Visual remediation system designer with well placement, treatment train, and piping layout tools
- Dewatering system configurator with well spacing optimization and drawdown prediction
- Injection well network designer with reagent distribution and radius-of-influence visualization

**Feature Gating:**
- Full access to well network design and optimization tools
- Remediation system sizing calculators (air sparging, soil vapor extraction, permeable reactive barriers)
- Export to engineering drawing formats and construction specification generators

**Pricing Tier:** Professional or Enterprise tier

**Onboarding Flow:**
- Quick-start: design a pump-and-treat system from an existing plume delineation model
- Tutorial on well placement optimization using capture zone analysis
- Walkthrough of remediation alternative evaluation and comparison workflow

**Key Workflows:**
- Designing extraction/injection well networks with optimized placement for plume capture
- Configuring monitored natural attenuation programs with sentinel well placement
- Designing permeable reactive barriers with reactive media selection and dimensioning
- Optimizing soil vapor extraction well layouts for vadose zone remediation
- Creating engineering design packages for remediation system construction

### Analyst/Simulator
**Modified UI/UX:**
- Model-centric workflow with mesh generation, property assignment, solver control, and post-processing panels
- Calibration dashboard with observed vs. simulated comparison plots and residual statistics
- Uncertainty analysis visualization with probability maps and confidence interval plume boundaries

**Feature Gating:**
- Full solver access for flow, transport, and reactive transport simulations
- Advanced calibration tools (PEST integration, pilot point parameterization, regularization)
- Parallel computing for large models and Monte Carlo simulations

**Pricing Tier:** Professional or Enterprise tier

**Onboarding Flow:**
- Guided first model: import well data, build simple model, calibrate, and assess calibration quality
- Setup of standard modeling protocols matching company or regulatory guidelines
- Introduction to sensitivity analysis and uncertainty quantification workflows

**Key Workflows:**
- Building and calibrating groundwater flow models against monitoring well data
- Simulating contaminant fate and transport with complex reaction networks
- Performing sensitivity analysis to identify dominant parameters controlling predictions
- Running Monte Carlo or null-space Monte Carlo uncertainty analyses
- Generating probabilistic plume boundary maps for risk-informed decision making

### Manufacturing/Process
**Modified UI/UX:**
- Remediation operations dashboard with well performance monitoring, reagent usage tracking, and system runtime
- Process optimization advisor suggesting pump rate adjustments based on real-time monitoring data
- Maintenance scheduler for well rehabilitation, sampling, and equipment servicing

**Feature Gating:**
- Access to operations monitoring and optimization tools
- Real-time sensor data integration for automated model updating
- System performance trending and efficiency metrics; full re-modeling capability restricted

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Configure remediation system inventory (wells, pumps, treatment units, monitoring points)
- Setup real-time data feeds from field sensors and SCADA systems
- Tutorial on using operational dashboards to track system performance against design targets

**Key Workflows:**
- Monitoring remediation system performance against cleanup milestones
- Optimizing extraction/injection rates based on plume response and operational constraints
- Tracking reagent consumption and estimating remaining remediation duration
- Managing well maintenance schedules and rehabilitation needs
- Generating operations and maintenance reports for regulatory submittals

### Regulatory/Compliance
**Modified UI/UX:**
- Regulatory compliance dashboard tracking cleanup standards, permit conditions, and reporting deadlines
- Model documentation generator producing reports per regulatory modeling guidelines (EPA, state agencies)
- Risk assessment calculator with exposure pathway analysis and target level computation

**Feature Gating:**
- Read-only access to models; full reporting, compliance checking, and risk calculation tools
- Automated comparison of predicted concentrations against regulatory cleanup standards
- Regulatory submittal package generator with required attachments and certifications

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Configuration of applicable regulations (CERCLA, RCRA, state cleanup programs) and cleanup standards
- Setup of reporting templates matching specific regulatory agency requirements
- Walkthrough: generate a modeling report meeting EPA groundwater modeling guidance requirements

**Key Workflows:**
- Reviewing model documentation for compliance with regulatory modeling guidelines
- Generating risk assessments using modeled concentrations at receptor locations
- Tracking compliance with permit conditions (monitoring frequencies, cleanup milestones)
- Preparing regulatory submittals (remedial investigation reports, feasibility studies, closure petitions)
- Maintaining administrative records and responding to regulatory review comments

### Manager/Decision-maker
**Modified UI/UX:**
- Site portfolio dashboard showing cleanup progress, costs, and regulatory status across multiple sites
- Financial liability estimator with present-value cleanup cost projections and reserve recommendations
- Remediation alternative comparison matrix with cost, timeframe, effectiveness, and risk scoring

**Feature Gating:**
- Dashboard and reporting access; no direct modeling or analysis capabilities
- Cost projection tools with Monte Carlo uncertainty on remediation duration and cost
- Multi-site portfolio views with environmental liability tracking

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Configuration of site portfolio with key milestones, cost centers, and responsible parties
- Integration with financial and project management systems
- Setup of automated portfolio status reporting and liability estimate updates

**Key Workflows:**
- Monitoring cleanup progress and cost performance across a portfolio of contaminated sites
- Evaluating remediation alternatives based on lifecycle cost, effectiveness, and implementation risk
- Estimating environmental liabilities for financial reporting and reserve adequacy
- Reviewing and approving remediation strategy changes based on performance data
- Communicating site status and risk to stakeholders, boards, and regulatory agencies
