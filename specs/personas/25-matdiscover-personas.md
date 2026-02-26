# MatDiscover — Persona Subspecs

> Parent spec: [`specs/25-matdiscover.md`](../25-matdiscover.md)

---

## Experience-Level Personas

### Student/Academic

**Modified UI/UX:**
- Show a simplified input panel where users enter a chemical formula or upload a CIF file and click "Predict" to get instant property estimates
- Provide a guided "Materials Explorer" with interactive periodic table for selecting elements and visualizing known composition-property trends
- Display tooltips on every predicted property explaining what it means physically (e.g., "Formation energy: how stable this material is relative to its elements")

**Feature Gating:**
- Allow single-material property prediction (up to 20/day) and basic structure visualization
- Hide generative design, batch screening, API access, and custom model training
- Provide read-only access to public materials databases (Materials Project, AFLOW) for exploration

**Pricing Tier:** Free tier

**Onboarding Flow:**
- First-run tutorial predicts properties for a well-known material (silicon, NaCl) and explains each output
- Offer a guided exploration of the periodic table showing how predicted properties change across element combinations
- Suggest a set of "classroom exercises" (e.g., compare bandgaps of group IV semiconductors) as learning activities

**Key Workflows:**
- Enter a composition or crystal structure and view predicted formation energy, bandgap, and mechanical properties
- Compare predicted properties of related materials (e.g., silicon vs. germanium) side by side
- Explore composition-property landscapes interactively using the periodic table selector
- Export predicted properties as a table for inclusion in a course report or thesis

---

### Junior Engineer (0-3 years)

**Modified UI/UX:**
- Show a project-based workflow with guided steps: define target properties, screen candidates, review results, document findings
- Provide a recommendation sidebar that suggests analysis steps based on the material class (e.g., "For battery cathodes, check voltage and ionic conductivity")
- Display results in both visual (phase diagrams, property maps) and tabular formats with export options

**Feature Gating:**
- Unlock batch prediction (up to 500 materials), basic generative design, and project-level organization
- Expose data import from Materials Project and AFLOW with filtering and curation tools
- Hide custom ML model training, API access, and enterprise collaboration features

**Pricing Tier:** Pro tier ($49/month)

**Onboarding Flow:**
- Walk through a complete materials screening workflow: define target property ranges, import candidates from a database, predict properties, and rank results
- Demonstrate the generative design feature by generating novel compositions for a target application (e.g., high-bandgap semiconductors)
- Show how to organize findings into a project with notes and comparisons for lab discussion

**Key Workflows:**
- Screen a curated set of candidate materials against target property specifications for a project
- Use generative design to propose novel compositions optimized for desired properties
- Import materials from public databases and filter by predicted properties to build a shortlist
- Create project reports comparing predicted properties across candidates with visualization
- Document materials screening decisions and share findings with supervisors or lab members

---

### Senior/Expert

**Modified UI/UX:**
- Provide a multi-panel workspace with synchronized structure viewer, property prediction, phase diagram, and data table views
- Expose the full computational workflow: DFT setup, ML prediction configuration, active learning loop control
- Support command-line/API interfaces alongside the visual UI for integration into existing computational pipelines

**Feature Gating:**
- Full access: unlimited predictions, generative design, custom model training, DFT workflow integration, and API
- Expose active learning workflows that combine ML predictions with targeted DFT calculations
- Enable team management, shared materials databases, and integration with VASP/Quantum ESPRESSO via managed compute

**Pricing Tier:** Enterprise tier ($199/month or institutional license)

**Onboarding Flow:**
- Skip tutorials; present API documentation and integration guides for connecting to existing DFT and experimental workflows
- Offer bulk data import from the user's existing materials databases (CSV, JSON, CIF archives)
- Configure preferred ML models, DFT parameters, and compute resource allocations

**Key Workflows:**
- Run large-scale materials screening campaigns across thousands of compositions with custom ML models
- Set up active learning loops that iteratively refine predictions by running targeted DFT calculations on uncertain candidates
- Train project-specific ML models using proprietary experimental data to improve prediction accuracy
- Integrate prediction workflows with VASP/Quantum ESPRESSO for validation of ML-identified candidates
- Manage a team's shared materials database with provenance tracking and reproducible workflows

---

## Role-Based Personas

### Designer/Creator

**Modified UI/UX:**
- Emphasize the generative design interface with target property sliders, constraint editors, and real-time candidate generation
- Provide an interactive 3D crystal structure builder for manual structure design and modification
- Show a "design space" visualization mapping generated candidates in property space relative to known materials

**Feature Gating:**
- Full access to generative design, structure building, and inverse property optimization tools
- Expose synthetic accessibility scoring and thermodynamic stability filtering for generated candidates
- Enable saving and sharing of design campaigns with collaborators

**Pricing Tier:** Pro tier ($49/month)

**Onboarding Flow:**
- Walk through a generative design campaign: set target properties, generate candidates, filter by stability and synthesizability
- Demonstrate the 3D structure builder by constructing a simple crystal structure from scratch
- Show how to evaluate generated candidates using predicted phase diagrams and stability metrics

**Key Workflows:**
- Define target property windows and use generative models to propose novel material compositions
- Build and modify crystal structures manually using the 3D structure editor
- Evaluate generated candidates for thermodynamic stability, synthesizability, and novelty
- Compare generated materials against the existing design space to identify truly novel candidates
- Export designed materials with structural files and predicted properties for experimental validation

---

### Analyst/Simulator

**Modified UI/UX:**
- Foreground the data analysis toolkit: property correlation matrices, dimensionality reduction plots (t-SNE, UMAP), and statistical comparison tools
- Show a DFT workflow manager with job setup, status tracking, and post-processing automation
- Provide a model performance dashboard showing prediction accuracy, uncertainty estimates, and calibration metrics

**Feature Gating:**
- Full access to batch prediction, statistical analysis, DFT workflow integration, and model evaluation tools
- Expose custom ML model training with hyperparameter tuning and cross-validation reporting
- Enable data export to Jupyter notebooks and integration with Materials Project API

**Pricing Tier:** Pro tier ($49/month)

**Onboarding Flow:**
- Demonstrate a comparative analysis workflow: predict properties for a materials family, analyze trends, and identify outliers
- Walk through setting up a DFT validation calculation for an ML-predicted candidate
- Show the model evaluation tools for assessing prediction confidence and identifying domains of applicability

**Key Workflows:**
- Analyze property trends across materials families using correlation analysis and dimensionality reduction
- Validate ML predictions against DFT calculations and experimental measurements systematically
- Train and evaluate custom ML models on domain-specific data with rigorous cross-validation
- Set up automated DFT workflows for validating top candidates from ML screening campaigns
- Generate publication-ready figures (phase diagrams, property maps, parity plots) for research papers

---

### Manufacturing/Process

**Modified UI/UX:**
- Show a synthesis feasibility dashboard with scores for synthetic accessibility, precursor availability, and process complexity
- Display a materials qualification pipeline with stages (predicted, DFT-validated, synthesized, characterized, qualified)
- Provide integration panels for connecting to LIMS and experimental characterization data management systems

**Feature Gating:**
- Expose synthesis pathway suggestion tools and precursor database integration
- Enable experimental data ingestion for closing the loop between prediction and characterization
- Unlock production-scale batch screening with priority compute and SLA-backed turnaround

**Pricing Tier:** Enterprise tier ($199/month)

**Onboarding Flow:**
- Configure the materials qualification pipeline with stages appropriate to the user's workflow
- Set up integration with the lab's experimental data management system for feedback loop tracking
- Demonstrate how to evaluate candidates for synthesis feasibility before committing lab resources

**Key Workflows:**
- Screen candidate materials for synthetic accessibility and process feasibility before experimental trials
- Track materials through the qualification pipeline from prediction to synthesis to characterization
- Ingest experimental characterization data and compare against predictions to close the feedback loop
- Prioritize synthesis campaigns based on predicted performance, stability, and ease of fabrication
- Generate materials data packages for scale-up evaluation with predicted and measured properties

---

### Regulatory/Compliance

**Modified UI/UX:**
- Add an environmental and safety screening panel showing toxicity flags, REACH/RoHS compliance, and restricted substance alerts
- Display a data provenance panel tracking the origin and license of every dataset used in predictions
- Provide an audit trail of all materials screening decisions with timestamps and rationale documentation

**Feature Gating:**
- Expose toxicity and environmental screening databases integrated into the prediction workflow
- Enable compliance report generation for REACH, RoHS, TSCA, and other chemical regulations
- Unlock audit trail with immutable records and exportable compliance documentation

**Pricing Tier:** Enterprise tier ($199/month)

**Onboarding Flow:**
- Configure regulatory screening rules for the user's industry (electronics/RoHS, chemicals/REACH, aerospace/ITAR)
- Walk through screening a candidate material for restricted substances and environmental compliance
- Demonstrate the audit trail and how to export compliance documentation for regulatory submissions

**Key Workflows:**
- Screen all candidate materials against REACH, RoHS, and TSCA restricted substance lists automatically
- Generate compliance documentation certifying that predicted materials meet regulatory requirements
- Maintain an audit trail of all materials screening and selection decisions for regulatory review
- Flag materials containing toxic or restricted elements early in the discovery process
- Export regulatory compliance reports for inclusion in product safety documentation

---

### Manager/Decision-maker

**Modified UI/UX:**
- Present a portfolio dashboard showing all active materials discovery projects, their stage, team assignments, and key findings
- Show ROI metrics: prediction cost vs. experimental cost savings, time-to-candidate-identification trends
- Provide a pipeline funnel visualization showing materials progressing from screening through validation to qualification

**Feature Gating:**
- Read-only access to all projects with commenting and prioritization capabilities
- Expose project management, resource allocation, and budget tracking tools
- Enable executive reporting with customizable dashboards and export to presentation formats

**Pricing Tier:** Enterprise tier ($199/month)

**Onboarding Flow:**
- Set up the portfolio dashboard and configure project categories (battery materials, semiconductors, alloys)
- Walk through the pipeline funnel view and how to interpret screening-to-qualification conversion rates
- Demonstrate cost tracking showing compute spend vs. estimated experimental cost savings

**Key Workflows:**
- Monitor all materials discovery projects from a unified dashboard with pipeline stage tracking
- Evaluate project ROI by comparing computational screening costs against traditional experimental approaches
- Prioritize discovery projects based on strategic alignment, progress, and resource requirements
- Review team findings and approve materials for advancement to experimental validation
- Generate executive summaries of discovery program progress for stakeholders and funding agencies
