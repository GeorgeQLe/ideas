# SynthBio — Persona Subspecs

> Parent spec: [`specs/35-synthbio.md`](../35-synthbio.md)

---

## Experience-Level Personas

### Student/Academic
**Modified UI/UX:**
- Visual circuit builder with SBOL-standard drag-and-drop parts (promoters, RBS, CDS, terminators) and inline tooltips explaining each part's biological function
- Simplified simulation panel with a single "Run Simulation" button that auto-configures Gillespie SSA parameters for reasonable defaults
- iGEM parts browser with curated "starter kits" organized by function (biosensors, toggle switches, oscillators) for rapid circuit prototyping

**Feature Gating:**
- Access to visual circuit builder, iGEM parts library, and basic Gillespie SSA with up to 1,000 trajectory ensembles
- CRISPR guide RNA design with standard Cas9; advanced Cas variants and base editors hidden
- Metabolic pathway tools limited to pre-built pathway templates; custom FBA model construction disabled

**Pricing Tier:** Free tier (academic email verification)

**Onboarding Flow:**
- Tutorial: build a simple genetic toggle switch from iGEM parts, simulate stochastic switching, and visualize results in under 20 minutes
- Pre-loaded example circuits (repressilator, biosensor, kill switch) with validated kinetic parameters
- Prompt to join an iGEM team workspace or course project if an invite code is available

**Key Workflows:**
- Design genetic circuits visually using standardized SBOL parts from the iGEM registry
- Simulate circuit behavior with stochastic models and explore parameter sensitivity
- Design CRISPR guide RNAs for genome integration of circuit components
- Compare predicted circuit performance across parameter ranges for iGEM project validation
- Export circuit designs and simulation results for wiki pages, posters, and presentations

---

### Junior Engineer (0-3 years)
**Modified UI/UX:**
- Project-oriented workspace with tabs for circuit design, simulation, sequence management, and CRISPR tools
- Contextual warnings for common design pitfalls (promoter strength mismatches, metabolic burden, codon bias issues)
- Side-by-side view comparing simulation predictions with experimental characterization data

**Feature Gating:**
- Full circuit builder with all SBOL parts; Gillespie SSA up to 10,000 trajectories with GPU acceleration
- CRISPR design for Cas9, Cas12, and base editors; batch guide design up to 50 targets
- Metabolic pathway analysis with pre-built organism models (E. coli, S. cerevisiae); custom FBA enabled

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Import an existing sequence from Benchling/GenBank and map it onto the visual circuit builder
- Walk through a metabolic engineering workflow: identify pathway bottleneck, design overexpression construct, simulate flux improvement
- Connect to team workspace and set up shared parts libraries with characterized parameters

**Key Workflows:**
- Design and simulate multi-gene circuits for strain engineering projects
- Perform flux balance analysis to identify metabolic bottlenecks and engineer pathway improvements
- Design CRISPR editing strategies for multiplexed genome modifications
- Manage sequence libraries and track construct versions across the design-build-test cycle
- Generate design reports documenting circuit architecture, predicted performance, and cloning strategy

---

### Senior/Expert
**Modified UI/UX:**
- Multi-panel workspace with concurrent circuit design, kinetic modeling, FBA, and sequence management views
- Custom model editor for defining novel kinetic mechanisms beyond standard Hill functions
- Python scripting console for programmatic circuit design, batch simulation, and custom analysis pipelines

**Feature Gating:**
- All features unlocked: custom kinetic models, unlimited GPU-accelerated ensembles, full FBA toolkit, API access
- Advanced CRISPR tools: base editing, prime editing, CRISPRi/CRISPRa with library-scale design
- Admin controls for organism model libraries, parts characterization databases, and team workflows

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Import existing COPASI models or Python simulation scripts to bootstrap kinetic modeling projects
- API walkthrough for integrating SynthBio with laboratory automation systems (Opentrons, Hamilton)
- Configure team parts registry with standardized characterization data and quality tiers

**Key Workflows:**
- Develop novel genetic circuit architectures with custom kinetic models and parameter estimation from experimental data
- Architect large-scale metabolic engineering campaigns with multi-objective FBA optimization
- Design library-scale CRISPR screening experiments with pooled guide design and barcode tracking
- Manage team-wide organism model libraries ensuring model consistency across projects
- Lead design reviews and mentor junior engineers on computational circuit design methodology

---

## Role-Based Personas

### Designer/Creator
**Modified UI/UX:**
- Circuit canvas as primary workspace with visual wiring between genetic parts and real-time expression level previews
- Parts palette organized by organism compatibility and characterized performance metrics
- Sequence assembly planner showing Golden Gate, Gibson, or BioBricks cloning strategy based on selected parts

**Feature Gating:**
- Full visual circuit builder, all SBOL parts, and sequence assembly planning tools
- Codon optimization for target organisms; synthetic gene design with restriction site management
- Export to GenBank, FASTA, and SBOL formats for ordering from DNA synthesis providers

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Build a circuit: select parts from the registry, wire them visually, generate the assembled DNA sequence
- Walk through cloning strategy: choose assembly method, design primers, verify in silico
- Save the design as a reusable module in the team parts library

**Key Workflows:**
- Design genetic circuits from concept to cloning-ready DNA sequences using visual composition tools
- Browse and select characterized parts from the iGEM registry and internal libraries
- Plan DNA assembly strategies and generate primer sequences for cloning
- Create modular, reusable genetic part libraries for team-wide standardization
- Export synthesis-ready sequences with codon optimization for target organisms

---

### Analyst/Simulator
**Modified UI/UX:**
- Simulation dashboard as primary workspace with time-course plots, phase portraits, and bifurcation diagrams
- Parameter estimation panel for fitting kinetic models to experimental characterization data
- Sensitivity analysis tools with tornado plots and Morris method screening for identifying critical parameters

**Feature Gating:**
- Full access to all simulation tools: Gillespie SSA, ODE solvers, FBA, parameter estimation
- GPU-accelerated ensemble simulations with statistical analysis of distributions
- Batch simulation management and automated sensitivity study pipelines

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Load an existing circuit and run a parameter sweep to explore the design space
- Fit kinetic parameters to experimental flow cytometry or plate reader data
- Configure automated robustness analysis to identify parameter ranges where circuit function is maintained

**Key Workflows:**
- Run large-scale stochastic simulations to characterize circuit behavior under biological noise
- Estimate kinetic parameters from experimental data using Bayesian inference methods
- Perform global sensitivity analysis to identify which parameters most affect circuit output
- Predict circuit robustness across environmental conditions (temperature, growth rate, nutrient availability)
- Generate quantitative analysis reports comparing predicted vs. observed circuit performance

---

### Manufacturing/Process
**Modified UI/UX:**
- Strain engineering dashboard tracking Design-Build-Test-Learn cycles with status for each construct
- Fermentation process integration panel linking circuit design parameters to bioreactor operating conditions
- Yield optimization view showing metabolic flux predictions mapped to production targets

**Feature Gating:**
- Access to metabolic modeling and flux optimization tools linked to process conditions
- Strain library management with genotype tracking and performance history
- Circuit design tools restricted to approved genetic parts; novel part characterization requires approval

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Connect to strain database and map existing production strains to metabolic models
- Walk through a yield optimization workflow: identify bottleneck, simulate intervention, predict improvement
- Configure DBTL cycle tracking for ongoing strain improvement campaigns

**Key Workflows:**
- Optimize metabolic pathways for production yield using FBA linked to fermentation process parameters
- Track strain engineering iterations with genotype-phenotype records across DBTL cycles
- Predict how process conditions (media composition, temperature, induction timing) affect engineered strain performance
- Manage approved genetic parts libraries for production strains with regulatory compliance
- Scale-up analysis: predict how circuit behavior changes from flask to bioreactor conditions

---

### Regulatory/Compliance
**Modified UI/UX:**
- Biosafety and regulatory dashboard showing containment levels, organism classifications, and genetic modification records
- Part provenance viewer tracing every genetic component to its source (iGEM registry entry, literature reference, synthetic design)
- Environmental risk assessment tools for evaluating containment failure scenarios and kill-switch effectiveness

**Feature Gating:**
- Read access to all circuit designs, simulation results, and strain records
- Write access to biosafety annotations, risk assessments, and regulatory approval stamps
- Audit trail with immutable records of all genetic designs, modifications, and approvals

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Configure applicable biosafety frameworks (NIH Guidelines, EU GMO Directive, Cartagena Protocol)
- Walk through a biosafety review of a sample engineered organism: containment level, kill-switch verification, risk assessment
- Set up Institutional Biosafety Committee (IBC) review workflows with required documentation templates

**Key Workflows:**
- Review genetic circuit designs for biosafety compliance and assign appropriate containment levels
- Verify kill-switch and biocontainment circuit effectiveness through simulation-based risk assessment
- Maintain comprehensive records of all genetically modified organisms with full design provenance
- Generate regulatory documentation for IBC submissions, EPA TSCA notifications, or USDA-APHIS permits
- Track regulatory approval status across all engineered organisms and flag expiring permits

---

### Manager/Decision-maker
**Modified UI/UX:**
- Portfolio dashboard showing strain development programs: target molecules, predicted yields, development stage, and timeline
- ROI estimator linking simulation-predicted titers to production economics ($/kg target molecule)
- Team activity view showing engineer workload, simulation throughput, and DBTL cycle velocity

**Feature Gating:**
- Read-only access to program dashboards, yield predictions, and cost models; no direct design or simulation tools
- Team management: license allocation, project creation, compute and DNA synthesis budget controls
- Approval authority for advancing strains from computational design to wet-lab construction

**Pricing Tier:** Enterprise tier (included in admin seat)

**Onboarding Flow:**
- Program dashboard overview: active strain engineering projects, yield improvement trajectories, milestone tracking
- Configure portfolio KPIs (target titer, production cost, development timeline) and stage-gate criteria
- Set up notification rules for simulation results indicating breakthrough yield improvements

**Key Workflows:**
- Monitor strain development pipelines across multiple target molecules and organisms
- Compare candidate strains based on predicted yield, genetic complexity, and scale-up risk
- Allocate lab resources, compute budget, and DNA synthesis spending across programs
- Review go/no-go decisions for advancing strains from in silico design to wet-lab validation
- Present program progress and technology pipeline to executive leadership and investors
