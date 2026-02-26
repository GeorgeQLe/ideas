# StructAI — Persona Subspecs

> Parent spec: [`specs/38-structai.md`](../38-structai.md)

---

## Experience-Level Personas

### Student/Academic
**Modified UI/UX:**
- Step-by-step model builder: define geometry (beam, frame, slab), assign materials, apply loads, set supports — each step with structural engineering educational annotations
- Simplified 3D viewer with color-coded stress contours, deflection animations, and labeled reaction forces
- Inline code reference panel showing relevant IBC/ASCE 7/Eurocode clauses alongside analysis results

**Feature Gating:**
- Access to linear static analysis for 2D and 3D frame structures up to 500 nodes
- Steel and concrete design checking for basic member types (beams, columns); advanced connection design hidden
- Single load case at a time; load combinations auto-generated but limited to gravity and basic lateral

**Pricing Tier:** Free tier (academic email verification)

**Onboarding Flow:**
- Tutorial: model a simple steel portal frame, apply gravity and wind loads, run analysis, and check member capacity ratios
- Pre-loaded example structures (multi-story frame, truss bridge, concrete slab) with validated results for comparison
- Prompt to join a course workspace if an instructor invite code is available

**Key Workflows:**
- Build and analyze simple structural models for coursework assignments (frames, trusses, beams)
- Apply standard load cases and visualize internal forces (moment, shear, axial) diagrams
- Check member designs against building code provisions and understand utilization ratios
- Compare hand calculation results against FEA output for structural analysis course validation
- Export analysis results and diagrams for homework submissions and exam preparation

---

### Junior Engineer (0-3 years)
**Modified UI/UX:**
- Template-based project setup: select building type (office, residential, industrial), number of stories, and structural system to auto-generate a starting model
- Contextual warnings for modeling issues (unstable supports, missing load paths, overloaded members) with suggested fixes
- Code check summary dashboard showing pass/fail status for every member with drill-down to specific clause checks

**Feature Gating:**
- Full linear and basic nonlinear static analysis with models up to 50,000 nodes
- Multi-code design checking (IBC, ASCE 7, Eurocode, AS/NZS); steel, concrete, and timber member design
- Up to 5 concurrent analysis runs; response spectrum analysis for seismic design enabled

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Import a Revit structural model and walk through the BIM-to-analysis conversion workflow
- Run analysis, review member utilization ratios, and use AI-powered auto-sizing to optimize member selections
- Connect to firm workspace and configure project standards (code edition, material grades, design preferences)

**Key Workflows:**
- Build structural models from BIM imports or templates and run gravity and lateral load analyses
- Perform code-based member design checking and iterate on member sizes to achieve efficient designs
- Run seismic response spectrum analysis and check drift limits against code requirements
- Generate structural calculation reports formatted for engineering review and building permit submission
- Collaborate with senior engineers by sharing models and requesting review in the team workspace

---

### Senior/Expert
**Modified UI/UX:**
- Advanced analysis workspace with nonlinear solver controls, performance-based design parameters, and multi-hazard loading
- AI optimization panel for automated member sizing, topology exploration, and cost-weight Pareto analysis
- Python scripting console for custom analysis procedures, parametric studies, and automated report generation

**Feature Gating:**
- All features unlocked: nonlinear time-history analysis, progressive collapse, buckling, GPU-accelerated solvers, API access
- Unlimited model size and concurrent runs; priority compute queue
- Admin controls for firm-wide design standards, template libraries, and quality assurance workflows

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Import existing SAP2000/ETABS models to migrate active projects to StructAI
- API integration walkthrough for connecting with BIM platforms, project management tools, and firm databases
- Configure firm design standards: preferred code editions, material specifications, and member selection tables

**Key Workflows:**
- Perform advanced nonlinear analyses including pushover, time-history, and progressive collapse for complex structures
- Run AI-powered design optimization to minimize structural weight or cost while satisfying all code requirements
- Develop parametric structural models for early-stage design exploration across multiple building configurations
- Establish and maintain firm-wide analysis templates, quality standards, and peer review workflows
- Mentor junior engineers by reviewing their models and providing feedback through the collaboration system

---

## Role-Based Personas

### Designer/Creator
**Modified UI/UX:**
- Structural modeling canvas with intuitive geometry creation: draw beams, columns, walls, and slabs in 3D with snap-to-grid and alignment guides
- BIM integration panel for importing/exporting Revit and Tekla models with bi-directional synchronization
- Structural system explorer showing framing plan options (moment frame, braced frame, shear wall) with performance previews

**Feature Gating:**
- Full structural modeling tools with all element types and material models
- BIM import/export with model synchronization; IFC and native Revit/Tekla format support
- AI-assisted member sizing and structural system selection tools

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Create a structural model from an architectural BIM import: assign structural elements, define load paths, verify connectivity
- Explore alternative structural systems using the AI system selector and compare performance
- Walk through BIM round-tripping: make changes in StructAI and push updated member sizes back to Revit

**Key Workflows:**
- Create structural models from scratch or BIM imports for building design projects
- Explore alternative structural systems (steel vs. concrete, braced vs. moment frame) in early design
- Synchronize structural models with architectural BIM platforms for coordinated design
- Use AI-assisted design to rapidly size members and optimize the structural system
- Generate structural framing plans and elevation drawings for design documentation

---

### Analyst/Simulator
**Modified UI/UX:**
- Analysis control center with solver monitoring, convergence tracking, and result validation dashboards
- Advanced post-processing: stress contours, mode shape animation, demand/capacity ratios, and story drift plots
- Comparison workspace for evaluating multiple design iterations against the same loading criteria

**Feature Gating:**
- Full access to all analysis types: linear, nonlinear, dynamic, buckling, stability
- Advanced post-processing: detailed stress results, plastic hinge visualization, energy dissipation plots
- Batch analysis across load case variations and design iterations; automated sensitivity studies

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Run a sample multi-load-case analysis and walk through post-processing: extract peak demands, check deflections, review mode shapes
- Configure solver settings for nonlinear analysis and understand convergence diagnostics
- Set up automated comparison reports across design iterations

**Key Workflows:**
- Run comprehensive structural analyses under multiple load combinations and check all relevant limit states
- Perform dynamic analysis (modal, response spectrum, time-history) for seismic and wind design
- Validate structural models through convergence studies, hand-check comparisons, and engineering judgment
- Run parametric studies to understand structural sensitivity to key design assumptions
- Generate detailed structural analysis reports with force diagrams, stress plots, and code check summaries

---

### Manufacturing/Process
**Modified UI/UX:**
- Construction-focused dashboard showing structural member schedules, connection details, and material quantities
- Fabrication data export panel with BOM generation, bolt schedules, and weld detail extraction
- Erection sequence planner linking structural analysis to construction phasing and temporary bracing requirements

**Feature Gating:**
- Access to member schedules, connection design tools, and material takeoff reports
- Fabrication drawing export in standard formats; integration with steel detailing software
- Model editing restricted to construction-phase modifications; primary design changes require engineer approval

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Generate a steel member schedule from an analyzed model and export connection loads for detailing
- Walk through construction phasing analysis: verify stability during erection sequence
- Configure BOM export formats and material procurement report templates

**Key Workflows:**
- Extract member schedules, connection forces, and material quantities from analyzed structural models
- Verify structural stability during construction phases and identify temporary bracing requirements
- Generate fabrication data exports (BOM, bolt schedules, embed plate locations) for procurement and detailing
- Coordinate with detailers by providing connection design loads and member geometry in standard formats
- Track material quantities and cost estimates across structural design iterations

---

### Regulatory/Compliance
**Modified UI/UX:**
- Code compliance dashboard showing building code check status for every member with clause-level traceability
- Peer review workflow panel with comment threads, approval stamps, and revision tracking on structural calculations
- Permit package generator assembling code-compliant calculation sets with engineer-of-record signature blocks

**Feature Gating:**
- Full access to code check results, calculation reports, and design documentation
- Write access to review comments, compliance annotations, and approval stamps
- Audit trail with immutable records of all model changes, analysis runs, and design decisions

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Configure applicable building codes (IBC edition, local amendments, special requirements)
- Walk through a code compliance review: verify load combinations, check member capacities, confirm drift limits
- Set up peer review workflows with required sign-off gates before permit submission

**Key Workflows:**
- Review structural designs for building code compliance across all applicable provisions
- Verify that load combinations, seismic design categories, and wind exposure parameters are correctly applied
- Manage peer review workflows with tracked comments and required approvals before submission
- Generate permit-ready structural calculation packages with code references and engineer-of-record stamps
- Track code compliance status across multiple projects and flag designs requiring re-analysis after code updates

---

### Manager/Decision-maker
**Modified UI/UX:**
- Firm dashboard showing all active structural projects with design status, code check progress, and deadline tracking
- Resource utilization view: engineer workload, compute usage, and license allocation across projects
- Cost estimation panel linking structural quantities (steel tonnage, concrete volume) to project cost projections

**Feature Gating:**
- Read-only access to project dashboards, structural summaries, and cost estimates; no direct analysis tools
- Team management: license allocation, project assignment, and workload balancing
- Approval authority for design milestones, permit submissions, and scope changes

**Pricing Tier:** Enterprise tier (included in admin seat)

**Onboarding Flow:**
- Firm dashboard overview: active projects, engineer utilization, upcoming deadlines, code check status
- Configure project portfolio with milestone tracking and resource allocation targets
- Set up notification rules for design milestones, code check failures, and deadline risks

**Key Workflows:**
- Monitor structural design progress across the firm's project portfolio
- Allocate engineering staff and compute resources to projects based on deadline and complexity
- Review structural cost estimates (steel tonnage, concrete volume) to support project budgeting and value engineering
- Track permit submission timelines and coordinate with clients and building officials
- Present firm capacity, project status, and technical capabilities to prospective clients
