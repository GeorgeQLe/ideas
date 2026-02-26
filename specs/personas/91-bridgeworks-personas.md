# BridgeWorks — Persona Subspecs

> Parent spec: [`specs/91-bridgeworks.md`](../91-bridgeworks.md)

---

## Experience-Level Personas

### Student/Academic

**Modified UI/UX:**
- Simplified ribbon with curated toolset: beam/girder templates, basic load cases, and 2D visualization
- Inline tooltips referencing AASHTO LRFD section numbers and textbook conventions
- "Learning Mode" overlay that annotates each analysis step with underlying theory

**Feature Gating:**
- Full access to single-span and simple multi-span analysis; no fatigue or seismic modules
- Export limited to PDF reports and CSV result tables; no BrIM/IFC export
- Load rating module restricted to inventory/operating rating (no legal/permit trucks)

**Pricing Tier:** Free tier

**Onboarding Flow:**
- Guided walkthrough building a standard simply-supported steel girder bridge
- Pre-loaded example projects (concrete T-beam, steel plate girder) with annotated results
- Prompt to connect university SSO for extended seat license

**Key Workflows:**
- Design a simple composite steel girder bridge to AASHTO LRFD
- Run influence line analysis for moving loads on multi-span continuous beams
- Compare hand-calculated section capacities against software results
- Generate a formatted design report for coursework submission

---

### Junior Engineer (0-3 years)

**Modified UI/UX:**
- Standard interface with code-check dashboards showing demand/capacity ratios color-coded by limit state
- Wizards for common bridge types (precast I-girder, steel plate girder, concrete box) that auto-populate geometry
- Side panel with firm-standard templates and previously approved detail libraries

**Feature Gating:**
- All geometric modeling and static analysis tools unlocked
- Seismic analysis limited to single-mode spectral; multi-mode and pushover require senior approval
- Load rating enabled for standard legal trucks; permit vehicle analysis gated behind review workflow

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Template selector filtered by DOT jurisdiction (state design standards auto-loaded)
- 15-minute interactive tutorial on model creation, load application, and results interpretation
- Automatic linking to firm QA/QC checklist that tracks completion milestones

**Key Workflows:**
- Model and design a prestressed concrete I-girder bridge per state DOT standards
- Perform HL-93 live load analysis and generate demand envelopes
- Run code checks for flexure, shear, and deflection at all limit states
- Prepare load rating summary for existing bridge inventory
- Export design drawings to CAD with auto-dimensioning

---

### Senior/Expert

**Modified UI/UX:**
- Full-access interface with customizable workspace layouts and scriptable automation panel
- Multi-model comparison view for parametric studies and alternative evaluation
- Direct access to solver settings, convergence controls, and advanced material models

**Feature Gating:**
- All modules unlocked: nonlinear analysis, staged construction, time-dependent effects, seismic pushover
- API access for batch processing, custom load generation, and integration with external FEA tools
- Administrative controls for template publishing and firm-wide standard enforcement

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Configuration wizard for firm standards: material libraries, load combinations, safety factors
- Migration assistant to import models from MIDAS Civil, CSiBridge, or LARS Bridge
- API key provisioning and scripting environment setup

**Key Workflows:**
- Perform staged construction analysis of segmental concrete box girder bridges
- Run nonlinear pushover analysis for seismic retrofit evaluation
- Conduct permit load rating with refined analysis per MBE
- Automate parametric studies across girder spacing, depth, and material grade
- Manage multi-bridge corridor projects with shared substructure definitions

---

## Role-Based Personas

### Designer/Creator

**Modified UI/UX:**
- Geometry-centric workspace with 3D parametric bridge modeler and real-time section property feedback
- Drag-and-drop component library for bearings, expansion joints, barriers, and drainage details
- Visual stress contour overlays on 3D model during iterative design

**Feature Gating:**
- Full geometric modeling, section design, and reinforcement detailing tools
- Access to aesthetic visualization and rendering for public presentation
- Code-check automation for all AASHTO LRFD limit states

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Template gallery organized by bridge type, span range, and material
- Quick-start project from parametric wizard: span, width, skew, and loading auto-configured
- Integration setup with CAD/BIM platforms (Revit, Civil 3D, MicroStation)

**Key Workflows:**
- Design superstructure geometry and optimize girder sections for material efficiency
- Detail reinforcement layouts for concrete decks and substructure elements
- Generate 3D BrIM models for handoff to detailing teams
- Iterate bearing and expansion joint selections based on thermal movement analysis
- Produce presentation-quality renderings for public meetings

---

### Analyst/Simulator

**Modified UI/UX:**
- Analysis-centric dashboard with load case manager, results explorer, and envelope visualization
- Convergence monitors and solver diagnostics prominently displayed
- Multi-tab results comparison for different load combinations and analysis methods

**Feature Gating:**
- Full access to linear/nonlinear static and dynamic analysis solvers
- Influence line/surface generation and moving load optimization
- Time-dependent analysis (creep, shrinkage, relaxation) and staged construction sequencing

**Pricing Tier:** Professional or Enterprise tier

**Onboarding Flow:**
- Solver configuration walkthrough: element types, mesh density recommendations, convergence criteria
- Benchmark validation suite comparing results against known analytical solutions
- Import wizard for geometry from design models or survey data

**Key Workflows:**
- Build refined finite element models for complex bridge geometries (curved, skewed, integral)
- Run dynamic analysis for seismic, wind, and vehicle-induced vibrations
- Perform staged construction analysis tracking stress history through erection sequence
- Validate load distribution factors against simplified code methods
- Generate detailed force/stress result reports with demand/capacity summaries

---

### Manufacturing/Process

**Modified UI/UX:**
- Fabrication-oriented views showing plate sizes, weld details, and camber diagrams
- Erection sequence animator with crane reach and clearance envelope visualization
- Bill of materials dashboard with weight summaries and piece-mark tracking

**Feature Gating:**
- Access to fabrication detail extraction and shop drawing data export
- Erection staging simulator with temporary support and stability checks
- Camber calculation tools for steel girders including dead load, creep, and haunch adjustments

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Setup wizard for fabrication shop capabilities: plate widths, rolling lengths, coating systems
- Import existing design model and auto-generate fabrication breakdowns
- Configure erection equipment library with crane capacity charts

**Key Workflows:**
- Extract fabrication data: plate cuts, stiffener locations, splice details
- Compute girder camber diagrams accounting for all loading stages
- Simulate erection sequence and verify stability at each stage
- Generate material quantity takeoffs for procurement and bidding
- Track field fit-up tolerances against design assumptions

---

### Regulatory/Compliance

**Modified UI/UX:**
- Compliance dashboard mapping every design check to specific code clause (AASHTO LRFD, MBE, FHWA)
- Audit trail panel showing all input changes, analysis runs, and approval stamps
- Flagging system for load ratings below legal thresholds or design checks near unity

**Feature Gating:**
- Full load rating module: inventory, operating, legal, and permit levels per MBE
- NBI coding assistant for bridge inspection data management
- Locked-down report templates conforming to FHWA and state DOT submission formats

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Jurisdiction selector auto-loading applicable design codes, load factors, and legal truck library
- Integration setup with state bridge management systems (BrM, Pontis)
- User role configuration for reviewer/approver workflows with digital signatures

**Key Workflows:**
- Perform load ratings for entire bridge inventory using batch processing
- Generate NBI-compatible inspection and rating reports
- Track bridges with posting requirements and monitor rating changes over time
- Audit design calculations for code compliance before plan submission
- Manage overweight permit routing decisions based on corridor ratings

---

### Manager/Decision-maker

**Modified UI/UX:**
- Executive dashboard with portfolio-level bridge condition, rating, and project status summaries
- Cost estimation rollups with material quantity trends and schedule milestones
- Risk matrix visualization highlighting bridges with critical rating deficiencies

**Feature Gating:**
- Read-only access to all project models and results; no direct editing
- Budget tracking, resource allocation, and project scheduling tools
- Portfolio analytics: bridge inventory statistics, replacement prioritization, funding scenarios

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Portfolio import wizard ingesting bridge inventory data from BrM or spreadsheet
- Dashboard configuration for KPIs: sufficiency ratings, structural deficiency counts, project completion
- Team setup with role assignments and notification preferences

**Key Workflows:**
- Review project status across multiple bridge design and rehabilitation projects
- Prioritize bridge replacement and rehabilitation investments based on condition data
- Compare alternative designs by cost, constructability, and lifecycle value
- Generate executive summaries for funding applications and capital improvement programs
- Monitor team workload and resource utilization across the bridge program
