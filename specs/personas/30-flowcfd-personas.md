# FlowCFD — Persona Subspecs

> Parent spec: [`specs/30-flowcfd.md`](../30-flowcfd.md)

---

## Experience-Level Personas

### Student/Academic

**Modified UI/UX:**
- Provide a wizard-driven interface that walks through each simulation step (geometry import, meshing, boundary conditions, solve, post-process) with plain-language explanations
- Show annotated boundary condition previews with color-coded zones and tooltips explaining each physics option (e.g., "No-slip wall: fluid velocity is zero at this surface")
- Display a simplified post-processing view with pre-configured visualizations (velocity contours, pressure fields, streamlines) and one-click figure export

**Feature Gating:**
- Allow simulations up to 500K cells on shared compute with steady-state solvers only (RANS: k-epsilon, k-omega SST)
- Hide transient solvers, LES/DES turbulence models, multiphase flow, and parametric sweeps
- Limit to 5 active projects and 10 simulation runs per month

**Pricing Tier:** Free tier

**Onboarding Flow:**
- First-run tutorial simulates flow over a cylinder, covering geometry import, meshing, boundary setup, solving, and post-processing in 15 minutes
- Offer a learning path of 6 progressively complex exercises (pipe flow, airfoil, heat exchanger, building aerodynamics)
- Provide a glossary of CFD terms (Reynolds number, mesh convergence, y+, CFL number) accessible from any screen

**Key Workflows:**
- Import a simple geometry (STL/STEP), apply AI-generated mesh, set boundary conditions, and run a steady-state simulation
- Visualize results with velocity contours, pressure fields, and streamlines using pre-configured post-processing
- Compare simulation results against analytical solutions for textbook validation cases
- Export figures and data tables for course reports and thesis chapters

---

### Junior Engineer (0-3 years)

**Modified UI/UX:**
- Show the full simulation setup interface with guided workflows and parameter recommendations based on the selected physics type
- Provide a "Solver Coach" panel that monitors convergence, flags potential issues (divergence, poor mesh quality), and suggests fixes
- Display mesh quality metrics (orthogonality, skewness, aspect ratio) with visual overlays on problem regions

**Feature Gating:**
- Unlock simulations up to 5M cells, transient solvers, additional turbulence models (RSM, Spalart-Allmaras), and basic multiphase
- Expose parametric sweeps (up to 10 variants), mesh refinement studies, and report generation
- Hide LES/DES, combustion, fluid-structure interaction, and custom solver modifications

**Pricing Tier:** Pro tier ($79/month)

**Onboarding Flow:**
- Walk through a real engineering case: import CAD geometry, let the AI mesh it, set up a thermal-flow simulation, and analyze results
- Demonstrate the Solver Coach by running a case with a deliberately poor mesh and following its recommendations to fix convergence
- Show how to run a mesh convergence study to validate simulation accuracy

**Key Workflows:**
- Set up and run thermal-fluid simulations for engineering design problems (HVAC, heat exchangers, pipe networks)
- Use the AI mesher and then refine specific regions based on mesh quality metrics and solver feedback
- Run mesh convergence studies to verify simulation accuracy before making design decisions
- Execute parametric sweeps varying design parameters (geometry, flow rates, temperatures) to explore the design space
- Generate engineering reports with simulation setup, results, and recommendations for design reviews

---

### Senior/Expert

**Modified UI/UX:**
- Provide a power-user interface with direct solver parameter access, custom boundary condition scripting, and multi-case management
- Expose full post-processing control: custom derived quantities, Python-scripted data extraction, and probe-based monitoring
- Support multi-case comparison views with synchronized visualizations across parametric studies

**Feature Gating:**
- Full access: unlimited mesh size on scalable HPC (100+ cores), LES/DES, multiphase (VOF, Euler-Euler), combustion, FSI, and custom UDFs
- Expose solver customization: custom turbulence models, user-defined functions, and source terms
- Enable team management, project templates, CI/CD integration for automated simulation pipelines, and on-premise deployment

**Pricing Tier:** Enterprise tier ($249/month or annual contract)

**Onboarding Flow:**
- Skip tutorials; present a feature comparison against ANSYS Fluent/OpenFOAM with migration guides and equivalent setup workflows
- Offer bulk import of existing OpenFOAM cases or Fluent journal files for migration
- Configure default solver preferences, HPC settings, and post-processing templates

**Key Workflows:**
- Run high-fidelity transient simulations (LES, DES) on large meshes (50M+ cells) with cloud HPC auto-scaling
- Set up complex multiphysics simulations combining fluid flow, heat transfer, multiphase, and chemical reactions
- Automate parametric design exploration with scripted simulation pipelines and automated post-processing
- Develop and validate custom solver extensions (UDFs) for specialized physics not covered by standard models
- Manage a team's simulation library with standardized templates, validation cases, and best practice guidelines

---

## Role-Based Personas

### Designer/Creator

**Modified UI/UX:**
- Emphasize the CAD import and geometry preparation tools with defeaturing, surface cleanup, and domain definition interfaces
- Provide a design comparison view showing simulation results overlaid on multiple geometry variants side by side
- Show a design optimization panel with objective functions (minimize drag, maximize heat transfer) and parametric geometry controls

**Feature Gating:**
- Full access to geometry import/preparation, AI meshing, steady-state and transient solvers, and design comparison tools
- Expose parametric geometry variation and design optimization with automated simulation sweeps
- Enable CAD integration for bi-directional updates between design tools and simulation

**Pricing Tier:** Pro tier ($79/month)

**Onboarding Flow:**
- Start by importing a CAD model, running a baseline simulation, and then modifying the geometry to improve performance
- Demonstrate the design comparison tool by evaluating two geometry variants side by side with identical simulation setups
- Show the parametric sweep workflow for systematically exploring design modifications

**Key Workflows:**
- Import CAD geometry, prepare it for simulation (defeaturing, domain definition), and run baseline simulations
- Compare simulation results across multiple design variants to identify the best-performing geometry
- Set up parametric sweeps varying geometry dimensions to find optimal configurations
- Run design optimization with automated simulation to minimize drag, maximize cooling, or meet other objectives
- Generate design comparison reports with quantified performance differences for engineering reviews

---

### Analyst/Simulator

**Modified UI/UX:**
- Foreground the post-processing toolkit: custom contour/vector/streamline plots, data probes, surface integrals, and force/moment reporting
- Show a convergence analysis dashboard with residual histories, monitor point trends, and mass/energy balance verification
- Provide a validation framework for comparing simulation results against experimental data with statistical metrics

**Feature Gating:**
- Full access to all solver types, post-processing tools, validation frameworks, and data export
- Expose advanced post-processing: spectral analysis, POD/DMD mode decomposition for transient data, and acoustic analysis
- Enable batch post-processing scripts and automated report generation across simulation series

**Pricing Tier:** Pro tier ($79/month)

**Onboarding Flow:**
- Walk through a complete analysis workflow: verify mesh quality, check convergence, validate against experimental data, and extract engineering quantities
- Demonstrate advanced post-processing by computing derived quantities (vorticity, Q-criterion, wall shear stress) on a transient dataset
- Show the validation framework comparing simulation results against published experimental data

**Key Workflows:**
- Perform thorough convergence analysis verifying residual levels, monitor point stability, and conservation balances
- Extract engineering quantities: lift/drag coefficients, heat transfer rates, pressure drops, and flow distribution
- Validate simulation results against experimental measurements using statistical comparison frameworks
- Apply advanced post-processing techniques (POD, DMD, spectral analysis) to transient simulation data
- Generate comprehensive analysis reports with methodology documentation, results, and uncertainty assessment

---

### Manufacturing/Process

**Modified UI/UX:**
- Show a process simulation dashboard tailored to manufacturing applications (mold filling, cooling, ventilation, spray drying)
- Display a production parameter sensitivity view showing how process variables affect product quality metrics
- Provide integration panels for connecting to SCADA/process control systems for digital twin validation

**Feature Gating:**
- Expose multiphase flow models relevant to manufacturing (mold filling, spray, mixing, heat treatment)
- Enable process parameter optimization with automated sweeps and quality metric tracking
- Unlock digital twin capabilities with real-time data ingestion from production sensors

**Pricing Tier:** Enterprise tier ($249/month)

**Onboarding Flow:**
- Configure the platform for the user's manufacturing domain (injection molding, HVAC, food processing, chemical reactors)
- Walk through setting up a process simulation with relevant physics models and quality metrics
- Demonstrate process parameter sensitivity analysis to identify critical control variables

**Key Workflows:**
- Simulate manufacturing processes (mold filling, mixing, heat treatment) to optimize quality and reduce defects
- Run process parameter sensitivity studies to identify critical variables affecting product quality
- Set up digital twin simulations synchronized with production sensor data for real-time monitoring
- Optimize process parameters (flow rates, temperatures, pressures) to meet quality targets while minimizing cost
- Generate process validation reports documenting simulation methodology and predicted operating windows

---

### Regulatory/Compliance

**Modified UI/UX:**
- Add a standards compliance panel showing verification status against relevant codes (ASHRAE, building codes, fire safety, EPA emissions)
- Display simulation methodology documentation with mesh statistics, solver settings, convergence proof, and uncertainty quantification
- Provide a validation evidence library linking simulation results to experimental benchmark data

**Feature Gating:**
- Expose automated verification against industry standards (ASHRAE 55 for thermal comfort, EN 1991 for wind loads, fire safety codes)
- Enable comprehensive methodology documentation with automated generation of simulation quality reports
- Unlock audit trail with immutable records and expert review/approval workflows

**Pricing Tier:** Enterprise tier ($249/month)

**Onboarding Flow:**
- Select applicable industry standards and load corresponding verification templates and acceptance criteria
- Walk through generating a simulation methodology report that meets quality assurance requirements
- Demonstrate the validation evidence library and how to link simulation results to benchmark experiments

**Key Workflows:**
- Verify simulation results against industry code requirements (ASHRAE, wind load standards, fire safety codes)
- Generate methodology documentation meeting quality assurance standards for regulatory submissions
- Maintain a validation evidence library linking each simulation type to verified benchmark comparisons
- Produce uncertainty quantification reports for simulation results used in safety-critical design decisions
- Document the complete simulation workflow (mesh, solver, convergence, post-processing) for third-party review and audit

---

### Manager/Decision-maker

**Modified UI/UX:**
- Present a simulation portfolio dashboard showing all active projects, simulation status, engineer assignments, and key findings
- Show cost analytics: HPC compute costs per project, simulation ROI vs. physical testing costs, engineer utilization
- Provide a decision support view summarizing simulation findings with confidence levels and design recommendations

**Feature Gating:**
- Read-only access to all simulation projects with annotation and approval capabilities
- Expose project management, resource allocation, and cost tracking tools
- Enable executive reporting with summarized findings, cost-benefit analysis, and decision matrices

**Pricing Tier:** Enterprise tier ($249/month)

**Onboarding Flow:**
- Set up the project portfolio dashboard and configure project categories (product design, process optimization, compliance)
- Walk through the cost analytics view comparing simulation costs against physical testing alternatives
- Demonstrate the decision support view summarizing key findings from a completed simulation study

**Key Workflows:**
- Monitor all simulation projects from a unified dashboard with status tracking and key result summaries
- Evaluate simulation ROI by comparing compute costs against physical testing and prototyping expenses
- Review simulation-based design recommendations and approve design decisions with documented rationale
- Allocate simulation resources (HPC budgets, engineer time) across projects based on priority and impact
- Generate executive summaries translating simulation findings into actionable business recommendations
