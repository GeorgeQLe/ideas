# FormSim — Persona Subspecs

> Parent spec: [`specs/69-formsim.md`](../69-formsim.md)

---

## Experience-Level Personas

### Student/Academic

**Modified UI/UX:**
- Guided forming simulation mode with annotated diagrams explaining deep drawing mechanics, springback, and forming limit diagrams (FLD)
- Interactive material characterization panel linking to flow curve models (Hollomon, Swift, Voce) and yield criteria (Hill48, Barlat)
- Pre-built tool geometries for canonical forming operations (cup drawing, V-bending, hole expansion)

**Feature Gating:**
- 2D axisymmetric and simple 3D forming simulations with basic material models; advanced constitutive models locked
- Standard FLD-based failure prediction; forming limit stress diagram (FLSD) and edge cracking models disabled
- Export to academic formats (plots, CSV, VTK); no die compensation or tryout support tools

**Pricing Tier:** Free tier (academic email verification)

**Onboarding Flow:**
- Start with a guided cylindrical cup drawing tutorial demonstrating blank holder force optimization and failure prediction
- Interactive exercise on forming limit diagrams: constructing FLDs and interpreting strain paths
- Access to NUMISHEET benchmark problems for validating simulation approaches

**Key Workflows:**
- Simulating canonical forming operations (cup drawing, bending, stretch forming) for metal forming coursework
- Comparing constitutive models and yield criteria on the same forming operation for thesis research
- Constructing and interpreting forming limit diagrams under different strain paths
- Generating deformed shape, thickness distribution, and strain maps for lab reports

---

### Junior Engineer (0-3 years)

**Modified UI/UX:**
- Template-based setup organized by forming operation (stamping, deep drawing, progressive die, hydroforming, hot forming)
- Automatic mesh generation with forming-specific adaptive refinement and element quality checks
- Traffic-light formability assessment overlaying FLD results on the formed part with clear pass/fail regions

**Feature Gating:**
- Full 3D forming simulation with adaptive meshing; advanced material models (Yoshida-Uemori, BBC2005) available with guided setup
- Springback prediction and basic die compensation enabled; multi-step optimization locked
- Forming feasibility reports generated; full die engineering documentation requires senior review

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Select forming type (automotive stamping, appliance, aerospace) for tailored material libraries and process templates
- Guided walkthrough of a stamping simulation from blank import through forming to springback prediction
- Tutorial on interpreting thinning maps, FLD margins, and springback contours for design decisions

**Key Workflows:**
- Running forming feasibility simulations to identify potential splits, wrinkles, and excessive thinning
- Predicting springback for dimensional accuracy assessment and die compensation planning
- Optimizing blank shape and size to minimize material usage while preventing failure
- Evaluating blank holder force and draw bead configurations for draw quality improvement
- Generating formability reports for die design review meetings

---

### Senior/Expert

**Modified UI/UX:**
- Full scripting API (Python) for custom constitutive models, automated die compensation loops, and batch optimization
- Multi-operation workspace linking blanking, forming, trimming, and flanging operations in progressive die sequences
- Advanced solver controls (time step, contact algorithm, remeshing strategy) with direct parameter access

**Feature Gating:**
- All features unlocked: hot forming, superplastic forming, incremental forming, multi-step progressive die simulation
- Custom material model implementation with user-defined yield functions and hardening laws
- Automated springback compensation with iterative die surface morphing and convergence control

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Import existing AutoForm, PAM-STAMP, or LS-DYNA forming models with automated parameter translation
- Benchmark against NUMISHEET and industrial validation datasets for accuracy verification
- Configure automated die compensation pipeline with convergence criteria and iteration limits

**Key Workflows:**
- Running multi-step progressive die simulations with carrier strip advancement and inter-station transfer
- Developing custom material models for new AHSS grades, aluminum alloys, and high-temperature forming
- Automating iterative springback compensation with die surface morphing algorithms
- Performing forming process optimization with multi-objective criteria (formability, springback, thinning)
- Leading die engineering programs from feasibility through tryout with simulation-guided correction

---

## Role-Based Personas

### Designer/Creator

**Modified UI/UX:**
- Die design workspace with parametric tool geometry editor (punch, die, blank holder, draw beads)
- Real-time formability preview during die face modifications showing impact on material flow
- Design variant manager for comparing die configurations with overlay of forming results

**Feature Gating:**
- Full die geometry creation and modification with parametric addendum surface generation
- Automated binder surface design and draw bead optimization tools
- Blank shape optimization module with nesting layout for material utilization

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Import part CAD and walk through addendum surface design, binder closure, and draw bead placement
- Template gallery of die configurations (single action, double action, progressive) as starting points
- Demonstrate the rapid die face iteration workflow with simulation feedback

**Key Workflows:**
- Designing die face geometry (addendum surfaces, draw beads, punch entry radius) for optimal material flow
- Optimizing blank shape and nesting layouts for minimum material waste
- Iterating die designs based on forming simulation feedback (splits, wrinkles, springback)
- Designing progressive die layouts with strip progression and inter-station transfer
- Creating die engineering documentation packages with tool drawings and forming analysis results

---

### Analyst/Simulator

**Modified UI/UX:**
- Analysis dashboard with formability maps (FLD, thinning, wrinkling indicators) and springback deformation contours
- Multi-case comparison workspace with synchronized result visualization across parameter variations
- Material characterization panel with flow curve fitting, yield surface calibration, and FLD prediction

**Feature Gating:**
- All forming simulation and analysis capabilities available based on license tier
- Advanced post-processing: stress/strain path analysis, surface defect (sink marks, skid lines) prediction
- Batch simulation with automated parameter sweeps across BHF, friction, material properties

**Pricing Tier:** Professional or Enterprise tier

**Onboarding Flow:**
- Tutorial on material model calibration from tensile test, bulge test, and FLD test data
- Guided forming simulation with systematic validation against physical tryout measurements
- Introduction to springback analysis methodology and sensitivity to simulation parameters

**Key Workflows:**
- Calibrating advanced material models from experimental characterization data
- Running forming simulations with systematic variation of process parameters for robust design
- Analyzing springback behavior and its sensitivity to material, friction, and tooling parameters
- Performing surface defect prediction (sink marks, restrikes, skid lines) for Class A surface quality
- Validating simulation accuracy against physical forming trials and measurement data

---

### Manufacturing/Process

**Modified UI/UX:**
- Shop-floor interface with press setup sheets, die position parameters, and forming process cards
- Real-time production monitoring showing press tonnage, cushion pressure, and part quality metrics
- Troubleshooting guide linking forming defects (splits, wrinkles, springback) to parameter adjustments

**Feature Gating:**
- Simplified simulation mode for evaluating parameter adjustments within validated production window
- Die setup sheet generation with press settings, BHF, draw bead engagement, and lubrication requirements
- Scrap analysis tools linking reject categories to process parameter deviations

**Pricing Tier:** Professional tier (manufacturing license)

**Onboarding Flow:**
- Connect to press control system and quality measurement equipment (CMM, laser scanner)
- Calibrate production model against first-article forming trials
- Train die setters on interpreting process setup sheets and troubleshooting common forming issues

**Key Workflows:**
- Generating optimized press setup sheets for production die installation
- Troubleshooting forming defects using simulation-based root cause identification
- Evaluating material lot variations (different steel coils) on forming quality
- Planning die maintenance based on wear prediction and dimensional drift monitoring
- Supporting continuous improvement by quantifying process parameter effects on quality metrics

---

### Regulatory/Compliance

**Modified UI/UX:**
- Quality compliance workspace organized by automotive standards (IATF 16949, customer-specific requirements)
- Dimensional conformance tracking with GD&T overlay on simulation-predicted part geometry
- PPAP documentation templates auto-populated from forming simulation and measurement data

**Feature Gating:**
- Read-only access to forming analysis results and dimensional conformance reports
- Automated PPAP element generation (dimensional results, material test reports, process flow diagrams)
- Version-controlled process documentation with change control and approval workflows

**Pricing Tier:** Enterprise tier (quality add-on)

**Onboarding Flow:**
- Select customer quality requirements (GM, Ford, Toyota, VW) to auto-configure documentation templates
- Walk through a PPAP submission package generated from forming simulation and validation data
- Configure change control workflow for die modifications and process parameter updates

**Key Workflows:**
- Generating dimensional conformance reports comparing simulation predictions against GD&T specifications
- Preparing PPAP documentation packages with simulation-based process capability evidence
- Documenting process changes and their impact on dimensional conformance for customer approval
- Maintaining control plans with simulation-validated process parameter windows
- Supporting customer quality audits with simulation methodology and validation documentation

---

### Manager/Decision-maker

**Modified UI/UX:**
- Program dashboard with die development status, tryout iteration count, and timeline tracking
- Cost analysis view comparing simulation-guided die development against traditional trial-and-error approach
- Capacity planning interface showing press utilization, die changeover schedules, and production throughput

**Feature Gating:**
- View-only access to project summaries, quality metrics, and cost reports
- Portfolio view across stamping programs with schedule risk and quality status indicators
- Budget management for die development, simulation compute, and tryout activities

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Configure die program portfolio with milestone tracking, budget allocation, and quality targets
- Walk through ROI framework: simulation cost vs. tryout iterations eliminated and rework avoided
- Set up automated reporting for die development milestones and quality metrics

**Key Workflows:**
- Monitoring die development programs against timeline, budget, and quality milestones
- Reviewing cost-benefit of simulation-guided die engineering vs. additional tryout iterations
- Approving die design changes based on simulation-predicted quality impact
- Evaluating capacity planning scenarios for new program launches
- Generating management reports on stamping quality, productivity, and continuous improvement trends
