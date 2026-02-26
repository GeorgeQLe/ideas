# MineAssay — Persona Subspecs

> Parent spec: [`specs/64-mineassay.md`](../64-mineassay.md)

---

## Experience-Level Personas

### Student/Academic

**Modified UI/UX:**
- Guided geostatistics tutorial mode with step-by-step explanations of variogram modeling, kriging, and block estimation
- Interactive 3D geological viewer with simplified controls and annotated stratigraphic layers
- Pre-loaded datasets (publicly available deposit models) for learning without requiring real drill data

**Feature Gating:**
- Access to basic geological modeling (surfaces, solids) and ordinary kriging estimation; conditional simulation disabled
- Dataset size limited to 500 drillholes; advanced geostatistical methods (indicator kriging, MIK) locked
- Export to academic formats (CSV, VTK) only; no JORC/NI 43-101 report generation

**Pricing Tier:** Free tier (academic email verification)

**Onboarding Flow:**
- Start with a pre-loaded porphyry copper dataset walking through drillhole import, geological modeling, and resource estimation
- Interactive variogram interpretation exercise explaining nugget, sill, and range in geological context
- Access to curated library of publicly available deposit datasets for coursework and thesis research

**Key Workflows:**
- Building 3D geological models from drillhole data for mining geology coursework
- Performing variogram analysis and kriging estimation for geostatistics courses
- Comparing estimation methods (IDW, nearest neighbor, kriging) on the same dataset
- Generating cross-sections and 3D visualizations for thesis presentations

---

### Junior Engineer (0-3 years)

**Modified UI/UX:**
- Wizard-driven workflow from drillhole import through geological modeling to resource estimation with validation checkpoints
- QA/QC dashboard that automatically flags suspect assay data (duplicates, standards, blanks) and drillhole collar/survey errors
- Side-by-side comparison of estimation results against declustered sample statistics for sanity checking

**Feature Gating:**
- Full geological modeling and ordinary/indicator kriging; conditional simulation available with guided setup
- Mine planning tools (pit optimization, scheduling) enabled in guided mode; manual parameter override restricted
- JORC/NI 43-101 report templates available but require Competent Person review before finalization

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Import sample drillhole dataset and walk through the complete resource estimation workflow with embedded best practices
- Guided QA/QC analysis tutorial highlighting common data quality issues and their impact on estimation
- Introduction to mine planning with a simple pit optimization exercise on the tutorial dataset

**Key Workflows:**
- Importing and validating drillhole databases (collar, survey, assay, lithology)
- Building wireframe geological domains and estimating grades within them using kriging
- Running drillhole spacing optimization studies for infill drilling programs
- Generating preliminary resource estimates for internal reporting
- Performing basic pit optimization studies to support scoping-level economic analysis

---

### Senior/Expert

**Modified UI/UX:**
- Full scripting console (Python) for custom geostatistical workflows, automated model updating, and batch processing
- Multi-deposit workspace with cross-project variogram and estimation parameter management
- Direct access to estimation engine parameters, search ellipse configuration, and block model management

**Feature Gating:**
- All features unlocked: conditional simulation, multi-element estimation, change of support, non-linear geostatistics
- Custom variogram model fitting with nested structures and anisotropy rotation
- JORC/NI 43-101/CIM compliant report generation with Competent Person digital signature workflow

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Import existing block models and geological models from Datamine, Vulcan, Leapfrog, or Surpac
- Benchmark estimation results against audited resource statements for validation
- API setup for automated model update pipelines connected to drill program databases

**Key Workflows:**
- Developing complex geological and grade models for multi-element, multi-domain deposits
- Running conditional simulations for risk analysis and ore/waste classification uncertainty
- Performing resource-to-reserve conversion with modifying factors and economic parameters
- Reviewing and auditing junior staff resource estimates as the designated Competent Person
- Building automated model update workflows triggered by new drilling data

---

## Role-Based Personas

### Designer/Creator

**Modified UI/UX:**
- 3D geological modeling canvas with intuitive surface creation, fault modeling, and domain wireframing tools
- Interactive drillhole visualization with lithology logs, assay grades, and structural measurements displayed along traces
- Snap-to-geology tools that honor drillhole contacts and structural measurements during surface editing

**Feature Gating:**
- Full geological modeling toolkit: surface triangulation, solid modeling, fault networks, fold analysis
- Drillhole management with desurveying, compositing, and domain coding tools
- Geological interpretation tools (cross-sections, long-sections, level plans) with 3D consistency checking

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Import drillhole database and walk through building a geological model from scratch (domains, surfaces, faults)
- Template gallery with common deposit types (porphyry, VMS, vein, stratiform) as starting points
- Tutorial on maintaining geological consistency between sections and 3D wireframes

**Key Workflows:**
- Building 3D geological domain models from drillhole interpretation and structural data
- Modeling fault networks and their displacement effects on ore body geometry
- Creating and updating geological surfaces as new drilling data becomes available
- Generating geological sections, plans, and 3D PDFs for exploration reports
- Designing infill drill programs based on geological uncertainty and grade continuity analysis

---

### Analyst/Simulator

**Modified UI/UX:**
- Geostatistical analysis workspace with variogram modeling, estimation, and validation tools arranged in analytical pipeline
- Model validation dashboard with swath plots, Q-Q plots, grade-tonnage curves, and cross-validation metrics
- Uncertainty visualization with probability maps and confidence interval overlays on block models

**Feature Gating:**
- All estimation and simulation methods available (OK, SK, IK, MIK, SGS, SIS, turning bands)
- Advanced geostatistical tools: change of support, data transformation (nscore, lognormal), declustering
- Batch estimation with automated parameter sensitivity analysis across search and variogram parameters

**Pricing Tier:** Professional or Enterprise tier

**Onboarding Flow:**
- Tutorial on variogram modeling best practices including nested structures and geometric anisotropy
- Guided resource estimation with full validation workflow (cross-validation, visual inspection, swath plots)
- Introduction to conditional simulation for uncertainty quantification and risk analysis

**Key Workflows:**
- Performing exploratory data analysis (EDA) and spatial statistics on drillhole data
- Fitting variogram models with manual and automatic fitting, including nested structures
- Running kriging estimation with optimized search parameters and block discretization
- Validating estimates through cross-validation, swath plots, and comparison with declustered statistics
- Running conditional simulations for classification confidence and economic risk analysis

---

### Manufacturing/Process

**Modified UI/UX:**
- Mine operations interface showing current block model grades overlaid on mining progress (as-built surfaces)
- Grade control dashboard comparing blast hole assays against model predictions with reconciliation metrics
- Real-time production tracking showing tonnes mined, grade delivered, and ore loss/dilution

**Feature Gating:**
- Grade control estimation with blast hole data integration; no geological model modification
- Short-term mine planning tools (weekly/monthly schedules) with grade blending optimization
- Reconciliation reporting comparing model predictions against mill head grades

**Pricing Tier:** Professional tier (operations license)

**Onboarding Flow:**
- Connect to mine survey and assay laboratory systems for automatic data ingestion
- Calibrate grade control model against recent production reconciliation data
- Train operators on interpreting grade control estimates and ore/waste boundary decisions

**Key Workflows:**
- Running grade control estimation using blast hole assays for dig-line optimization
- Reconciling planned vs. actual grades and tonnes for model performance assessment
- Optimizing ore blending from multiple mining faces to meet mill feed specifications
- Updating short-term mine plans based on grade control results and equipment availability
- Generating daily/weekly production reports with grade, tonnes, and metal content tracking

---

### Regulatory/Compliance

**Modified UI/UX:**
- Reporting compliance dashboard organized by standard (JORC 2012, NI 43-101, CIM, SAMREC) with checklist-driven guidance
- Audit trail showing every data modification, estimation parameter change, and result with timestamps and user attribution
- Table of contents view for resource reports with section-by-section completion status and reviewer assignments

**Feature Gating:**
- Read-only access to geological models, estimation results, and classification maps
- Automated JORC/NI 43-101 Table 1 generation from simulation parameters and QA/QC data
- Version-controlled resource statement archive with digital Competent Person signatures

**Pricing Tier:** Enterprise tier (compliance add-on)

**Onboarding Flow:**
- Select applicable reporting standard (JORC, NI 43-101, CIM) to auto-configure report templates and checklists
- Walk through a sample JORC Table 1 audit trail showing data lineage from drillhole to resource statement
- Configure review workflow with Competent Person sign-off and stock exchange filing integration

**Key Workflows:**
- Generating JORC/NI 43-101 compliant resource and reserve statements for public reporting
- Auditing estimation parameters and data quality documentation for external review
- Maintaining resource classification criteria documentation and consistency across updates
- Preparing supporting documentation for Competent Person/Qualified Person sign-off
- Tracking material changes in resource estimates that trigger continuous disclosure obligations

---

### Manager/Decision-maker

**Modified UI/UX:**
- Executive dashboard with resource inventory, mine life, NPV scenarios, and exploration portfolio value
- Strategic scenario comparison showing resource outcomes under different commodity price and cut-off grade assumptions
- Exploration investment tracker linking drilling expenditure to resource category upgrades (Inferred to Indicated)

**Feature Gating:**
- View-only access to resource summaries, grade-tonnage curves, and economic analysis outputs
- Strategic mine planning outputs (pit shells, NPV schedules) at summary level
- Portfolio analytics across multiple deposits with risk-weighted resource valuation

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Configure portfolio view across all company deposits with current resource statements and exploration status
- Walk through interpreting grade-tonnage curves and their implications for project economics
- Set up automated alerts for resource estimate updates and material changes

**Key Workflows:**
- Reviewing resource estimates and their confidence levels for investment decisions
- Comparing project economics under different commodity price and operating cost scenarios
- Allocating exploration budgets across deposits based on resource upgrade potential and strategic value
- Monitoring mine-to-model reconciliation performance as an indicator of resource estimate reliability
- Generating investor presentations with resource summaries, mine life projections, and growth pipeline
