# BioMedica — Persona Subspecs

> Parent spec: [`specs/63-biomedica.md`](../63-biomedica.md)

---

## Experience-Level Personas

### Student/Academic

**Modified UI/UX:**
- Guided simulation setup with annotated biomedical tutorials (stent deployment, hip implant fatigue, catheter flow)
- Inline biomechanics reference panel linking to Fung, Humphrey, and Holzapfel textbook chapters for constitutive models
- Simplified material library with pre-configured tissue and biomaterial properties (bone, cartilage, UHMWPE, nitinol, PEEK)

**Feature Gating:**
- Single-physics simulations only (FEA or CFD, not coupled FSI); mesh limited to <200K elements
- Pre-built anatomical geometries (femur, coronary artery, spine segment) available; custom anatomy import disabled
- No FDA report generation; export limited to academic formats (plots, CSV, screenshots)

**Pricing Tier:** Free tier (academic email verification)

**Onboarding Flow:**
- Start with a pre-loaded stent compression tutorial demonstrating nonlinear contact and hyperelastic material modeling
- Interactive module on selecting appropriate constitutive models for biological tissues
- Access to shared library of professor-curated biomedical simulation assignments

**Key Workflows:**
- Simulating simplified implant loading scenarios for biomechanics coursework
- Comparing constitutive models (Mooney-Rivlin, Ogden, Holzapfel-Gasser-Ogden) for soft tissue response
- Running basic blood flow simulations in idealized vessel geometries for biofluid mechanics courses
- Generating results for thesis research on device-tissue interaction mechanics

---

### Junior Engineer (0-3 years)

**Modified UI/UX:**
- Workflow-driven interface organized by device type (orthopedic, cardiovascular, neurostimulator) with pre-configured analysis templates
- Step-by-step verification checklist ensuring mesh quality, boundary conditions, and material assignments meet biomedical simulation best practices
- Side-by-side view comparing simulation predictions against ASTM/ISO test standard expectations

**Feature Gating:**
- Multi-physics enabled (structural + thermal, CFD + mass transport); FSI available with validated templates
- Full material library including fatigue properties; custom material model creation requires senior approval
- FDA report drafts generated but flagged as "requires review" before finalization

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Select device category (orthopedic implant, cardiovascular, drug delivery) to receive tailored workspace and templates
- Guided simulation of a device following the relevant ASTM test standard (e.g., F2077 for spinal implants)
- Tutorial on BioMedica's verification and validation (V&V) documentation workflow

**Key Workflows:**
- Simulating device performance under ASTM/ISO standard test conditions for design verification
- Running fatigue analysis (high-cycle, multiaxial) for implants under physiological loading
- Performing biocompatibility-related simulations (corrosion, wear, ion release rates)
- Building preliminary FDA submission simulation sections with guided documentation templates
- Comparing design iterations on key performance metrics (stress, strain, safety factor)

---

### Senior/Expert

**Modified UI/UX:**
- Full scripting API (Python) for custom constitutive models, patient-specific geometry pipelines, and automated V&V
- Direct solver configuration access with multi-physics coupling controls (FSI partitioning, convergence criteria)
- Multi-study orchestration workspace managing device master files across regulatory submissions

**Feature Gating:**
- All solvers and physics unlocked: FSI, electromagnetic-thermal-structural coupling, population-level statistical analysis
- Patient-specific modeling pipeline (DICOM import, segmentation, mesh generation) fully enabled
- Automated FDA 510(k) and PMA computational evidence report generation with digital signatures

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Import existing ANSYS/COMSOL/Abaqus biomedical models with automated setup translation
- Benchmark against FDA-recognized test standards and published clinical data
- Configure automated V&V pipelines and regulatory documentation workflows

**Key Workflows:**
- Developing patient-specific simulation models from medical imaging (CT/MRI) for in silico clinical trials
- Running coupled fluid-structure interaction for cardiovascular devices (TAVR, stent grafts, VADs)
- Building computational evidence packages for FDA submissions per ASME V&V 40 framework
- Designing and executing virtual clinical trials across patient population geometries
- Mentoring junior engineers on constitutive model selection and V&V methodology

---

## Role-Based Personas

### Designer/Creator

**Modified UI/UX:**
- CAD-integrated workspace with parametric device geometry editor and real-time stress preview
- Design variant comparison with overlay visualizations of deformation, stress, and flow patterns
- Anatomical context view showing device within surrounding tissue for fit and placement assessment

**Feature Gating:**
- Full geometry creation and parametric modification tools with biomedical design templates
- Automated meshing with device-specific refinement (contact surfaces, thin features, flow channels)
- Shape optimization module for minimizing stress concentrations while meeting geometric constraints

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Import existing device CAD (SolidWorks, Creo) and auto-detect critical features for simulation setup
- Template gallery organized by device type (stent, screw, plate, cage, catheter, needle)
- Walk through a parametric stent design study showing geometry-to-performance feedback loop

**Key Workflows:**
- Iterating on device geometry with rapid simulation feedback on mechanical performance
- Optimizing implant topology for weight reduction while maintaining fatigue life targets
- Designing drug-eluting device geometries to achieve target release kinetics
- Evaluating device-tissue interface mechanics for different anatomical placements
- Exporting finalized device designs with simulation-verified performance specifications

---

### Analyst/Simulator

**Modified UI/UX:**
- Analysis dashboard with mesh quality metrics, convergence monitors, and V&V status indicators
- Multi-case comparison workspace for parametric studies across device variants and loading scenarios
- Statistical analysis panel for population-level results (mean, variance, worst-case across patient anatomies)

**Feature Gating:**
- All simulation physics and solvers available based on license tier
- Advanced post-processing: fatigue damage maps, wear volume predictions, hemolysis indices
- Batch simulation with automated parameter sweeps across loading conditions and anatomical variations

**Pricing Tier:** Professional or Enterprise tier

**Onboarding Flow:**
- Tutorial on mesh convergence studies for biomedical simulations with tissue contact
- Guided V&V exercise comparing simulation against a published bench test dataset
- Introduction to population-based analysis using statistical shape models

**Key Workflows:**
- Running high-fidelity multi-physics simulations for device performance characterization
- Performing mesh convergence, time step sensitivity, and model verification studies
- Validating simulation predictions against bench test and cadaveric test data
- Conducting population-level analyses across patient anatomy variations for worst-case identification
- Generating ASME V&V 40-compliant credibility assessment documentation

---

### Manufacturing/Process

**Modified UI/UX:**
- Manufacturing-focused interface showing process parameters (laser settings, annealing profiles, sterilization conditions) and their effect on device performance
- Process window visualization mapping manufacturing tolerances to simulation-predicted performance variation
- Incoming inspection integration linking material test certificates to simulation material inputs

**Feature Gating:**
- Process simulation modules (laser cutting, heat treatment, electropolishing effects on fatigue)
- Manufacturing tolerance analysis with Monte Carlo sampling of dimensional variations
- No design modification tools; locked to released device geometry with process parameter adjustment only

**Pricing Tier:** Professional tier (manufacturing license)

**Onboarding Flow:**
- Configure manufacturing process chain (forming, heat treatment, surface finishing, sterilization)
- Link incoming material properties to simulation inputs for lot-by-lot traceability
- Tutorial on interpreting process window maps for setting manufacturing specifications

**Key Workflows:**
- Evaluating impact of manufacturing tolerances on device mechanical performance
- Simulating heat treatment and surface finishing effects on residual stress and fatigue life
- Running process window studies to set manufacturing specifications with safety margins
- Assessing sterilization cycle effects on polymer degradation and device longevity
- Supporting nonconformance investigations with simulation-based root cause analysis

---

### Regulatory/Compliance

**Modified UI/UX:**
- Regulatory submission workspace organized by submission type (510(k), PMA, De Novo, CE MDR) with section-by-section guidance
- Traceability matrix linking every simulation result to its intended use, V&V evidence, and risk assessment
- Audit-ready view with complete simulation history, software version tracking, and reviewer sign-off workflow

**Feature Gating:**
- Read-only access to all simulation results with annotation and review capabilities
- Automated regulatory report generation per FDA Guidance on Computational Modeling Studies in Submissions
- Digital signature workflow with 21 CFR Part 11 compliant electronic records

**Pricing Tier:** Enterprise tier (regulatory add-on)

**Onboarding Flow:**
- Select submission pathway (510(k), PMA, CE Technical File) to auto-configure documentation templates
- Walk through an example computational evidence section for a 510(k) substantial equivalence argument
- Configure review and approval workflow with appropriate stakeholder sign-offs

**Key Workflows:**
- Assembling computational evidence sections for FDA 510(k) and PMA submissions
- Reviewing V&V documentation against ASME V&V 40 and FDA guidance requirements
- Managing the simulation traceability matrix linking results to device risk assessments
- Generating CE Technical Documentation computational sections per EU MDR requirements
- Responding to FDA review questions on computational modeling methodology and credibility

---

### Manager/Decision-maker

**Modified UI/UX:**
- Program dashboard showing device development milestones, simulation status, and regulatory submission readiness
- Risk matrix view linking simulation results to design risk assessment (dFMEA) outcomes
- Cost-benefit dashboard comparing in silico testing costs against bench and animal testing savings

**Feature Gating:**
- View-only access to project summaries, regulatory readiness scores, and risk assessment outputs
- Portfolio-level analytics across multiple device programs and regulatory submissions
- Budget management with simulation compute costs and V&V documentation effort tracking

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Configure device program portfolio with milestone tracking tied to V&V and regulatory deliverables
- Walk through interpreting simulation-based risk reduction in the context of design controls (ISO 13485)
- Set up automated status reports for regulatory submission readiness

**Key Workflows:**
- Monitoring device development programs against design control milestones and regulatory timelines
- Reviewing simulation-informed risk assessments for design review meetings
- Approving simulation strategies and V&V plans for regulatory submissions
- Evaluating return on investment for computational evidence vs. additional physical testing
- Generating board-level summaries of device portfolio status and regulatory pathway progress
