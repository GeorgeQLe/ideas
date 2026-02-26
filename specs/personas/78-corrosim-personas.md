# CorroSim — Persona Subspecs

> Parent spec: [`specs/78-corrosim.md`](../78-corrosim.md)

---

## Experience-Level Personas

### Student/Academic
**Modified UI/UX:**
- Interactive Pourbaix diagram explorer with annotated stability regions and corrosion/passivation boundaries
- Simplified electrochemical cell builder with visual anode/cathode assignment and potential distribution preview
- Educational mode with step-by-step explanations of corrosion mechanisms (galvanic, pitting, crevice, SCC)

**Feature Gating:**
- Pourbaix diagram generation and basic electrochemical potential calculations unlocked
- 1D corrosion rate estimation using Butler-Volmer kinetics and mixed-potential theory
- 2D/3D cathodic protection modeling and localized corrosion simulation locked

**Pricing Tier:** Free tier (academic license)

**Onboarding Flow:**
- Tutorial: generate a Pourbaix diagram for iron in water and identify corrosion, immunity, and passivity regions
- Pre-loaded examples: galvanic coupling of dissimilar metals, uniform corrosion rate estimation
- Guided exercise comparing predicted corrosion rates against published experimental data

**Key Workflows:**
- Generating and interpreting Pourbaix diagrams for common engineering alloys
- Calculating galvanic corrosion potentials and current densities for metal couples
- Estimating uniform corrosion rates from polarization curve analysis
- Producing annotated corrosion mechanism diagrams for materials science coursework

### Junior Engineer (0-3 years)
**Modified UI/UX:**
- Template-driven corrosion assessment workflows organized by mechanism (uniform, galvanic, pitting, crevice, SCC)
- Material-environment compatibility checker with traffic-light risk indicators
- Guided cathodic protection design wizard with anode sizing and distribution calculations

**Feature Gating:**
- Full electrochemical modeling for uniform, galvanic, and cathodic protection analysis
- Pitting and crevice corrosion risk assessment using empirical criteria (PREN, critical potentials)
- Localized corrosion propagation simulation and SCC modeling restricted to senior approval

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Role-based setup: pipeline/infrastructure, marine/offshore, chemical process, or aerospace corrosion focus
- Walkthrough: assess galvanic corrosion risk for a dissimilar metal joint and design mitigation
- Introduction to corrosion monitoring data integration and reporting tools

**Key Workflows:**
- Assessing material compatibility for design configurations with dissimilar metal contacts
- Designing cathodic protection systems (sacrificial anode and impressed current)
- Evaluating pitting and crevice corrosion susceptibility for material selection
- Analyzing corrosion monitoring data (coupons, probes, ER/LPR) and trending
- Generating corrosion assessment reports for design reviews and material selection decisions

### Senior/Expert
**Modified UI/UX:**
- Multi-physics corrosion modeling console coupling electrochemistry, transport, mechanics, and metallurgy
- Custom corrosion kinetics model editor for non-standard environments (sour service, nuclear, supercritical CO2)
- Scripting interface for automated parametric studies, probabilistic analysis, and remaining life prediction

**Feature Gating:**
- All modules: localized corrosion propagation, SCC/hydrogen embrittlement, erosion-corrosion, high-temperature oxidation
- Full API for coupling with structural integrity codes (FFS), process simulation, and digital twin platforms
- Admin controls for proprietary corrosion data, custom models, and company material specifications

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Migration assistant for importing COMSOL corrosion models, OLI chemistry databases, and legacy inspection data
- Configuration of custom corrosion rate models calibrated to specific operating environments
- Setup of remaining life prediction frameworks and risk-based inspection planning tools

**Key Workflows:**
- Modeling complex corrosion mechanisms with coupled electrochemistry and mass transport
- Predicting stress corrosion cracking and hydrogen embrittlement susceptibility
- Developing remaining life models integrating inspection data with corrosion growth predictions
- Calibrating corrosion models against field data for specific operating environments
- Managing corrosion engineering knowledge bases and material selection guidelines

---

## Role-Based Personas

### Designer/Creator
**Modified UI/UX:**
- Material selection advisor with corrosion resistance ranking for specified service environments
- Coating and lining specification tool with system buildup, application requirements, and durability estimates
- Cathodic protection system layout designer with anode placement, cable routing, and power supply sizing

**Feature Gating:**
- Full access to material selection, coating specification, and CP design tools
- Corrosion allowance calculators for thickness design
- Export to engineering drawing formats with corrosion protection details and specifications

**Pricing Tier:** Professional or Enterprise tier

**Onboarding Flow:**
- Quick-start: select materials for a pipeline in a specified soil environment using the material advisor
- Tutorial on cathodic protection system design from soil survey data to anode layout
- Walkthrough of coating system specification process for an offshore structure

**Key Workflows:**
- Selecting materials for corrosive service environments using compatibility databases and simulation
- Designing cathodic protection systems with optimized anode placement and output
- Specifying coating and lining systems with surface preparation, application, and inspection requirements
- Designing corrosion-resistant joints, connections, and transitions between dissimilar materials
- Creating corrosion protection specifications for procurement and construction

### Analyst/Simulator
**Modified UI/UX:**
- Electrochemical simulation workspace with mesh generation, boundary conditions, and solver controls
- Corrosion rate mapping tool with spatial distribution visualization on 3D geometry
- Probabilistic corrosion analysis dashboard with Monte Carlo sampling and reliability statistics

**Feature Gating:**
- Full solver access for electrochemical, transport, and coupled corrosion simulations
- Probabilistic analysis tools for remaining life estimation with uncertainty quantification
- HPC integration for large-scale cathodic protection models (pipelines, marine structures)

**Pricing Tier:** Professional or Enterprise tier

**Onboarding Flow:**
- Guided first simulation: cathodic protection potential distribution on a simple pipeline section
- Setup of standard analysis templates for common corrosion assessment scenarios
- Introduction to probabilistic analysis workflow for remaining life prediction

**Key Workflows:**
- Simulating cathodic protection current and potential distributions on complex structures
- Modeling galvanic corrosion rates for multi-material assemblies in specific environments
- Running probabilistic corrosion growth models for fitness-for-service assessment
- Simulating localized corrosion initiation and propagation under realistic service conditions
- Validating models against field inspection data and laboratory electrochemical measurements

### Manufacturing/Process
**Modified UI/UX:**
- Corrosion monitoring system configuration dashboard with sensor placement and alarm threshold setup
- Chemical treatment optimization panel for inhibitor dosing, water chemistry, and biocide programs
- Weld and heat-affected zone corrosion susceptibility checker with process parameter guidance

**Feature Gating:**
- Access to corrosion monitoring configuration and chemical treatment optimization tools
- Weld corrosion assessment tools (sensitization, preferential weld corrosion)
- Real-time process data integration for corrosion rate trending

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Configure asset inventory (piping, vessels, tanks) and installed monitoring systems
- Walkthrough: set up corrosion monitoring data collection and alarm configuration
- Tutorial on optimizing chemical treatment programs using corrosion rate feedback

**Key Workflows:**
- Configuring and managing corrosion monitoring systems (probes, coupons, online sensors)
- Optimizing chemical treatment programs (inhibitors, biocides, pH control) based on monitoring data
- Assessing weld procedure impacts on corrosion susceptibility
- Tracking corrosion rates across process units and correlating with operating conditions
- Generating inspection and maintenance work orders based on corrosion rate trends

### Regulatory/Compliance
**Modified UI/UX:**
- Regulatory compliance dashboard for pipeline integrity (PHMSA), pressure equipment (ASME), and offshore (API/NORSOK)
- Fitness-for-service assessment wizard per API 579-1 / ASME FFS-1 for corrosion-affected equipment
- Inspection record manager with remaining life calculations and re-inspection scheduling

**Feature Gating:**
- Read-only access to analysis; full compliance checking, FFS assessment, and reporting tools
- Automated API 579 Level 1/2 assessment calculators for general and localized corrosion
- Regulatory reporting generators for PHMSA, OSHA PSM, EPA RMP, and offshore regulations

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Configuration of applicable regulations (49 CFR 192/195, API 510/570, NORSOK M-001, ASME PCC-2)
- Setup of inspection record database and remaining life tracking system
- Walkthrough: perform an API 579 Level 1 fitness-for-service assessment for a corroded vessel

**Key Workflows:**
- Performing fitness-for-service assessments per API 579 for corroded equipment
- Managing inspection records and calculating remaining life for risk-based inspection planning
- Generating regulatory compliance reports for pipeline integrity management programs
- Tracking cathodic protection system compliance with NACE/AMPP standards
- Documenting corrosion management program effectiveness for regulatory audits

### Manager/Decision-maker
**Modified UI/UX:**
- Asset integrity dashboard showing corrosion risk rankings, inspection status, and remaining life across fleet
- Total cost of corrosion calculator comparing material upgrades, coatings, CP, inhibition, and replacement
- Risk matrix visualization mapping likelihood of corrosion failure against consequence severity

**Feature Gating:**
- Dashboard and reporting access; no direct analysis or simulation tools
- Cost-of-corrosion analysis and mitigation investment comparison
- Portfolio-level integrity management and risk-based decision views

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Configuration of asset portfolio with criticality rankings and consequence-of-failure assessments
- Integration with CMMS, inspection databases, and financial systems
- Setup of automated integrity status reporting and risk review cadence

**Key Workflows:**
- Monitoring asset integrity across a portfolio of equipment, piping, and structures
- Evaluating corrosion mitigation investments using total cost of ownership analysis
- Prioritizing inspection and maintenance spending based on risk-ranked asset lists
- Reviewing integrity operating windows and alarm exceedance trends
- Approving material upgrades, replacement programs, and inspection interval extensions
