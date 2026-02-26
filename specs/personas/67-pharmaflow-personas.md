# PharmaFlow — Persona Subspecs

> Parent spec: [`specs/67-pharmaflow.md`](../67-pharmaflow.md)

---

## Experience-Level Personas

### Student/Academic

**Modified UI/UX:**
- Guided reaction kinetics mode with step-by-step parameter estimation from experimental data and inline statistical diagnostics
- Interactive phase diagram viewer for crystallization with annotated solubility curves and metastable zone explanations
- Pre-built experiment templates for common academic studies (kinetic screening, solubility measurement, dissolution testing)

**Feature Gating:**
- Kinetic model fitting and simple reactor simulation (batch, semi-batch); scale-up tools disabled
- Crystallization thermodynamics (solubility prediction) available; population balance modeling locked
- Export to academic formats (plots, CSV, LaTeX tables); no GxP-compliant documentation

**Pricing Tier:** Free tier (academic email verification)

**Onboarding Flow:**
- Start with a guided reaction kinetics tutorial fitting rate constants from simulated lab data
- Interactive module on experiment design principles (DoE) for efficient parameter space exploration
- Access to published pharmaceutical reaction datasets for method development practice

**Key Workflows:**
- Fitting kinetic models to experimental reaction monitoring data for research projects
- Exploring crystallization phase diagrams and predicting solubility in mixed solvent systems
- Designing experiments using statistical DoE for efficient parameter screening
- Generating publication-quality kinetic and thermodynamic analysis plots

---

### Junior Engineer (0-3 years)

**Modified UI/UX:**
- Workflow-driven interface organized by process step (reaction, workup, crystallization, filtration, drying)
- Guided parameter estimation wizard with automatic model selection, confidence intervals, and diagnostic plots
- Scale-up prediction panel showing expected performance at 100L and 1000L alongside lab results

**Feature Gating:**
- Full kinetic modeling and reactor simulation; scale-up predictions available with guided interpretation
- Crystallization population balance modeling enabled with standard nucleation/growth rate models
- Process documentation templates available; GxP-compliant reports require QA review before finalization

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Select process type (small molecule synthesis, crystallization, biologic) for tailored workspace
- Guided walkthrough: import lab data, fit kinetic model, predict pilot scale performance
- Tutorial on interpreting scale-up predictions and identifying scale-sensitive parameters

**Key Workflows:**
- Building kinetic models from lab experiments for process understanding and optimization
- Predicting reaction performance at pilot and manufacturing scale from lab data
- Designing crystallization processes with target polymorph, particle size, and yield
- Generating process development reports documenting experiments, models, and scale-up rationale
- Running sensitivity analyses to identify critical process parameters (CPPs)

---

### Senior/Expert

**Modified UI/UX:**
- Full scripting console (Python/MATLAB) for custom kinetic models, thermodynamic property methods, and optimization routines
- Multi-step process orchestration workspace linking reaction, workup, crystallization, and formulation models
- Regulatory documentation generator aligned with ICH Q8/Q9/Q10/Q11 and QbD framework

**Feature Gating:**
- All features unlocked: continuous process modeling, multi-phase reactors, population balance, CFD-based scale-up
- Custom thermodynamic property models and parameter regression from proprietary data
- GxP-compliant documentation with electronic signatures, audit trails, and 21 CFR Part 11 compliance

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Import existing DynoChem or gPROMS models with automated parameter translation
- Benchmark against historical scale-up data to validate model predictive accuracy
- Configure GxP validation package and electronic signature workflows

**Key Workflows:**
- Developing mechanistic process models for regulatory filing support (ICH Q8 design space)
- Running CFD-informed scale-up studies for mixing-sensitive reactions and crystallizations
- Building control strategies based on model-predicted relationships between CPPs and CQAs
- Leading process validation with model-based prediction of acceptable parameter ranges
- Designing continuous manufacturing processes with integrated process models and PAT strategies

---

## Role-Based Personas

### Designer/Creator

**Modified UI/UX:**
- Process flow editor with drag-and-drop unit operations specific to pharma (reactor, crystallizer, filter, dryer, blender)
- Interactive reaction scheme builder with stoichiometry, mechanism, and byproduct pathway visualization
- Design space explorer with contour plots of yield, purity, and impurity levels across process parameters

**Feature Gating:**
- Full process design tools with reaction network definition, equipment selection, and operating condition specification
- Automated design space mapping using model-based predictions
- Equipment sizing modules for reactors, crystallizers, filter/dryers, and continuous flow systems

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Import a reaction scheme and walk through building a kinetic model from screening data
- Template gallery of common pharmaceutical unit operations with typical operating ranges
- Demonstrate the DoE-to-design-space pipeline for systematic process development

**Key Workflows:**
- Designing synthetic routes with kinetic models predicting yield, selectivity, and impurity profiles
- Mapping design spaces for critical quality attributes across process parameter ranges
- Designing crystallization protocols (cooling profiles, anti-solvent addition, seeding strategies)
- Evaluating alternative process configurations (batch vs. continuous, solvent selection)
- Specifying equipment requirements and operating conditions for technology transfer packages

---

### Analyst/Simulator

**Modified UI/UX:**
- Data analysis workspace with parameter estimation diagnostics, residual plots, and confidence interval visualization
- Model comparison panel with AIC/BIC statistics for selecting between competing kinetic models
- Sensitivity and uncertainty analysis dashboard with tornado plots and Monte Carlo results

**Feature Gating:**
- All modeling and simulation capabilities available based on license tier
- Advanced parameter estimation (Bayesian, maximum likelihood) with identifiability analysis
- Batch simulation with automated parameter sweeps and global sensitivity analysis (Sobol indices)

**Pricing Tier:** Professional or Enterprise tier

**Onboarding Flow:**
- Tutorial on parameter estimation best practices: model selection, identifiability, and validation
- Guided Bayesian parameter estimation exercise with prior specification and posterior interpretation
- Introduction to global sensitivity analysis for identifying critical process parameters

**Key Workflows:**
- Fitting mechanistic kinetic models to experimental data with rigorous statistical analysis
- Performing model discrimination using statistical criteria to select the best kinetic mechanism
- Running global sensitivity analyses to rank process parameters by impact on quality attributes
- Conducting uncertainty propagation to predict manufacturing performance with confidence bounds
- Validating models against independent datasets and quantifying prediction accuracy

---

### Manufacturing/Process

**Modified UI/UX:**
- Manufacturing execution interface showing batch records alongside model-predicted trajectories
- Real-time process monitoring with model-based soft sensors for unmeasured quality attributes
- Deviation investigation panel linking process excursions to model-predicted quality impact

**Feature Gating:**
- Calibrated production models; no model structure modification, only parameter updating
- Real-time batch trajectory prediction and endpoint estimation
- PAT integration with model-based multivariate statistical process control

**Pricing Tier:** Enterprise tier (manufacturing license)

**Onboarding Flow:**
- Connect to manufacturing execution system (MES) and process data historian
- Calibrate production models against recent manufacturing batch data
- Train operators on interpreting model-based process monitoring displays and alerts

**Key Workflows:**
- Monitoring batch processes in real-time with model-predicted quality attribute estimation
- Investigating process deviations using model-based root cause analysis
- Optimizing batch cycle times using model-predicted endpoints instead of fixed hold times
- Supporting process validation with model-based demonstration of process capability
- Evaluating raw material variability impact on process performance using calibrated models

---

### Regulatory/Compliance

**Modified UI/UX:**
- Regulatory filing workspace organized by ICH guideline (Q8, Q9, Q10, Q11, Q12) with section-by-section templates
- Design space documentation with proven acceptable ranges (PAR) and normal operating ranges (NOR) clearly delineated
- Audit trail view showing complete model development history with data, assumptions, and validation evidence

**Feature Gating:**
- Read-only access to models, design spaces, and process documentation
- Automated regulatory section generation for CTD Module 3 (Quality) filings
- 21 CFR Part 11 compliant electronic records with digital signatures and version control

**Pricing Tier:** Enterprise tier (regulatory add-on)

**Onboarding Flow:**
- Select regulatory pathway (NDA, ANDA, BLA, MAA) to auto-configure documentation templates
- Walk through a sample design space documentation package per ICH Q8(R2)
- Configure electronic signature workflow for GxP document review and approval

**Key Workflows:**
- Preparing design space justification documents for regulatory filings per ICH Q8
- Documenting process understanding and control strategy rationale for CTD Module 3
- Generating process validation protocols based on model-predicted critical process parameters
- Supporting regulatory agency questions on process development methodology and modeling approach
- Managing post-approval changes using model-based comparability assessments per ICH Q12

---

### Manager/Decision-maker

**Modified UI/UX:**
- Portfolio dashboard showing drug product pipeline with process development status and timeline for each program
- Cost-of-goods projection view comparing process alternatives on raw material, equipment, and time costs
- Risk register linking process uncertainties to probability of batch failure and supply disruption

**Feature Gating:**
- View-only access to project summaries, cost projections, and risk assessments
- Technology transfer readiness tracker across development programs
- Budget management for lab experiments, simulation compute, and pilot plant campaigns

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Configure product pipeline portfolio with development stage, timeline, and resource allocation for each program
- Walk through interpreting cost-of-goods projections from process models
- Set up automated reporting for development milestones and budget utilization

**Key Workflows:**
- Reviewing process development progress and cost-of-goods projections for portfolio prioritization
- Approving pilot plant campaign plans based on model-predicted scale-up confidence
- Comparing process route alternatives on yield, cost, environmental impact, and development timeline
- Monitoring technology transfer readiness across manufacturing sites
- Generating executive summaries of process development portfolio status for leadership review
