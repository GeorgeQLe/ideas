# Acoustica — Persona Subspecs

> Parent spec: [`specs/51-acoustica.md`](../51-acoustica.md)

---

## Experience-Level Personas

### Student/Academic

**Modified UI/UX:**
- Simplified ribbon with "Learn" mode toggled on by default; tooltips reference ISO 3382 and textbook equations inline
- Room geometry imports limited to predefined templates (shoebox, fan-shaped, L-shaped) with drag-handle dimensioning
- Results dashboard defaults to RT60, EDT, C80, and D50 with plain-language interpretation panels

**Feature Gating:**
- Ray-tracing capped at 10k rays; FDTD solver disabled; max room volume 2,000 m^3
- Export limited to PDF reports and CSV; no EASE/ODEON interop connectors
- Community material library only; custom scattering coefficient editor locked

**Pricing Tier:** Free tier (educational license with .edu verification)

**Onboarding Flow:**
- Guided "First Room" tutorial: load a lecture hall template, assign materials, run quick sim, interpret results
- Pop-up concept cards explaining Sabine vs. Eyring theory when relevant parameters appear
- Sandbox mode with pre-solved cases students can tweak and compare

**Key Workflows:**
- Model a classroom and predict speech intelligibility (STI)
- Compare absorption treatments by swapping ceiling tile materials
- Generate a lab report with RT60 decay curves and spatial maps
- Validate textbook equations against simulated results

---

### Junior Engineer (0-3 years)

**Modified UI/UX:**
- Standard workspace with guided wizard for new projects; color-coded severity indicators on acoustic parameter maps
- Context-sensitive help panel docked on the right showing "Why does this matter?" for each metric
- Checklist sidebar tracking common deliverables (NC curves, RT60 compliance, LEED acoustic credit)

**Feature Gating:**
- Full ray-tracing (100k rays); image-source method enabled; FDTD available for rooms under 500 m^3
- Import from SketchUp, Revit (IFC), and DXF; export to standard report templates
- Auralisation limited to binaural rendering (no Ambisonics or multichannel output)

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Project-type selector (concert hall, open office, classroom, worship space) pre-loads relevant standards and target criteria
- Interactive walkthrough of a real-world open-office noise study with annotated decision points
- Prompted to connect firm's material library and report templates during setup

**Key Workflows:**
- Model open-plan office and evaluate speech privacy (SII) with partition and masking scenarios
- Produce LEED v4 EQ Credit acoustics documentation package
- Run parametric study on ceiling height vs. RT60 for client presentations
- Import architect's Revit model and assign acoustic materials by room schedule
- Generate noise criteria (NC/RC) maps for mechanical equipment rooms

---

### Senior/Expert

**Modified UI/UX:**
- Power-user layout with scriptable console (Python/Lua), multi-viewport (plan + section + 3D + spectrogram), and batch-run manager
- Custom metric builder allowing user-defined spatial acoustic parameters and weighting functions
- Direct database connection for linking measured field data to model calibration

**Feature Gating:**
- All solvers unlocked: ray-tracing (unlimited rays), FDTD, BEM, hybrid methods
- Full auralisation chain including Ambisonics, HRTF customisation, and convolution with measured IRs
- API access for automated parametric sweeps, CI/CD pipeline integration, and custom solver extensions

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Import existing project archive (ODEON, EASE, CATT-Acoustic) with automatic model and material translation
- Configuration wizard for solver preferences, default tolerances, and HPC cluster connection
- Optional skip-all with power-user preset that opens blank scripting workspace

**Key Workflows:**
- Calibrate model against in-situ impulse response measurements and quantify prediction uncertainty
- Design variable-acoustics concert hall with motorised reflectors and curtains across multiple configurations
- Run city-scale environmental noise propagation (CadnaA replacement) with terrain, barriers, and meteorological gradients
- Develop custom room acoustic metrics for specialised venues (recording studios, anechoic chambers)
- Automate regulatory compliance checks (building codes, WHO community noise guidelines) across portfolio of projects

---

## Role-Based Personas

### Designer/Creator

**Modified UI/UX:**
- Visual-first interface with real-time false-colour SPL overlays on 3D architectural model
- Material palette with visual swatches showing absorption curves, NRC ratings, and aesthetic photos from manufacturer catalogues
- Integrated auralisation player: "listen" to the room from any seat position before it's built

**Feature Gating:**
- Geometry editing tools (push/pull surfaces, parametric diffusers, curved panels) fully enabled
- Solver limited to fast ray-tracing for interactive design iteration; full FDTD requires explicit launch
- BIM round-trip enabled (Revit, ArchiCAD) preserving acoustic metadata on export

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Start from architectural drawings or BIM import; auto-detect rooms and suggest material assignments
- "Design Intent" wizard: select venue type and target aesthetic/acoustic character (warm, bright, intimate, spacious)
- Side-by-side comparison mode to audition two design variants in real time

**Key Workflows:**
- Shape a performance hall's geometry to optimise early lateral reflections
- Select and place diffuser panels to eliminate flutter echoes while matching interior design language
- Produce client-facing auralisation demos comparing design options
- Iterate on ceiling cloud geometry to balance acoustic absorption with architectural vision
- Export acoustic surface schedule back to Revit for coordination with interior designers

---

### Analyst/Simulator

**Modified UI/UX:**
- Data-dense workspace with convergence monitors, energy decay histograms, and spatial parameter grids
- Multi-solver comparison panel showing ray-tracing vs. FDTD vs. BEM results side by side with deviation metrics
- Script editor with autocomplete for batch parameter sweeps and post-processing pipelines

**Feature Gating:**
- All solver engines and mesh refinement controls fully unlocked
- Advanced post-processing: spatial interpolation, uncertainty quantification, frequency-dependent mapping
- HPC job submission and cluster resource management tools enabled

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Solver benchmark suite: run standardised test cases (ITA round-robin rooms) to validate installation and establish baseline accuracy
- Import previous analysis templates and calibration data
- Configure convergence criteria and default mesh density rules

**Key Workflows:**
- Perform hybrid ray-tracing/FDTD simulation for a complex coupled-volume venue
- Conduct sensitivity analysis on material uncertainty and its effect on RT60 and clarity
- Validate simulation predictions against measured room impulse responses
- Develop and apply custom scattering models for non-standard surface geometries
- Produce peer-review-ready technical reports with uncertainty bounds and methodology documentation

---

### Manufacturing/Process

**Modified UI/UX:**
- Product-centric view organised by acoustic product families (panels, diffusers, barriers, enclosures)
- Test-standard compliance dashboard showing product ratings (STC, NRC, Rw, Dnt,w) against target specifications
- Integration panel for impedance tube and reverberation chamber measurement data import

**Feature Gating:**
- Transfer matrix method (TMM) for multilayer panel design fully enabled
- Transmission loss and insertion loss solvers unlocked; room-scale solvers available but secondary
- Material database editing with version control and batch testing capabilities

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Product library setup: import existing product datasheets, test reports, and material properties
- Guided tutorial on designing a new acoustic panel from raw material properties to predicted STC rating
- Connect to lab measurement equipment for automated data import

**Key Workflows:**
- Design multilayer wall assemblies and predict STC/Rw ratings before physical testing
- Optimise diffuser well-depth sequences for target scattering coefficients
- Simulate acoustic enclosure insertion loss for industrial noise control products
- Compare predicted vs. measured transmission loss to refine material models
- Generate product datasheets with rated performance for sales and specification teams

---

### Regulatory/Compliance

**Modified UI/UX:**
- Standards-first dashboard organised by jurisdiction and regulation (IBC, BB93, DIN 4109, AS/NZS 2107)
- Traffic-light compliance indicators on every room and partition with direct links to code clauses
- Audit trail panel logging every assumption, material choice, and calculation with timestamps

**Feature Gating:**
- Compliance checking engine and report generators fully enabled
- Solver access limited to validated methods required by referenced standards
- Locked calculation templates for standard-specific procedures (e.g., EN 12354 series)

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Jurisdiction selector pre-loads relevant building acoustic standards and threshold values
- Walkthrough of a code-compliance check for a multifamily residential project
- Configure firm-specific report templates with required disclaimers and professional seals

**Key Workflows:**
- Evaluate multifamily dwelling separating wall and floor assemblies against IBC/STC 50 requirements
- Produce BB93 (UK school acoustics) compliance report with room-by-room pass/fail summary
- Assess environmental noise impact from a proposed industrial facility against local ordinances
- Document façade sound insulation calculations per EN 12354-3 for planning approval
- Generate court-admissible expert reports for noise nuisance disputes

---

### Manager/Decision-maker

**Modified UI/UX:**
- Executive dashboard with project portfolio overview, budget impact of acoustic treatments, and risk heatmaps
- One-page summary cards per project showing compliance status, cost estimates, and key decisions pending
- Comparative ROI visualisation for acoustic upgrade options (cost vs. occupant satisfaction / productivity)

**Feature Gating:**
- Read-only access to simulation results; cannot modify models or solver settings
- Full access to reporting, cost estimation, and project tracking modules
- Team workload and license utilisation analytics enabled

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Portfolio import wizard linking to firm's project management system (Procore, Newforma, or custom)
- Dashboard configuration selecting KPIs relevant to business (utilisation, compliance rate, client satisfaction)
- Quick demo of cost-scenario comparison using a sample office renovation project

**Key Workflows:**
- Review acoustic consultant's recommendations and compare cost/benefit of treatment options
- Track compliance status across a multi-building campus development
- Allocate acoustic engineering resources across active projects based on deadline and complexity
- Present auralisation demos to clients and stakeholders for design approval sign-off
- Monitor project risk: identify buildings at risk of failing acoustic code before construction
