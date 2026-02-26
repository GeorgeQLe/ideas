# CADForge — Persona Subspecs

> Parent spec: [`specs/41-cadforge.md`](../41-cadforge.md)

---

## Experience-Level Personas

### Student/Academic

**Modified UI/UX:**
- Simplified toolbar with core sketch, extrude, revolve, and fillet tools; advanced surfacing and sheet-metal panels hidden by default
- Inline contextual hints overlay explaining geometric constraints and design intent as the user works
- Pre-loaded example assemblies (bracket, enclosure, gear train) accessible from the start screen

**Feature Gating:**
- Full parametric modeling, basic FEA stress preview, and standard file export (STEP, STL) available
- AI generative design, topology optimization, and PDM/PLM integrations locked
- Assembly limit capped at 50 components; simulation mesh limited to 100k elements

**Pricing Tier:** Free tier (education license with .edu verification)

**Onboarding Flow:**
- Guided "Design Your First Part" tutorial that walks through sketch-constrain-extrude-fillet in under 10 minutes
- Campus-oriented workspace with shared project folders for coursework submissions
- Tooltips auto-activate for the first 20 sessions, then fade to on-demand

**Key Workflows:**
- Sketching 2D profiles and extruding to 3D solids for homework assignments
- Running basic stress analysis to validate hand calculations from mechanics of materials courses
- Exporting STL files for 3D printing lab prototypes
- Collaborating on group capstone project assemblies with version history

---

### Junior Engineer (0-3 years)

**Modified UI/UX:**
- Standard professional layout with categorized feature tree, property manager, and drawing workspace
- "Smart Suggest" sidebar recommends next likely modeling operation based on current geometry context
- Warnings and best-practice nudges for common mistakes (e.g., under-constrained sketches, thin-wall issues)

**Feature Gating:**
- Full parametric and direct modeling, sheet metal, weldments, and surfacing tools unlocked
- Generative design available in guided mode (wizard-driven constraints and objectives)
- PDM check-in/check-out enabled; admin-level PLM configuration restricted

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Role-based setup wizard: "What do you design?" (consumer products, industrial equipment, tooling, etc.) to pre-configure toolbars
- Import existing models from SolidWorks/Fusion 360 with automatic feature-recognition and repair report
- 5-day drip email series covering parametric best practices, keyboard shortcuts, and collaboration features

**Key Workflows:**
- Creating fully constrained parametric part models from engineering drawings or specs
- Building multi-body assemblies with mates, interference detection, and BOM generation
- Generating 2D manufacturing drawings with GD&T annotations per ASME Y14.5
- Running design-table-driven configurations for product families
- Submitting models through PDM review and approval workflows

---

### Senior/Expert

**Modified UI/UX:**
- Power-user layout with macro recorder, API scripting console, and customizable command palette
- Minimal chrome mode that maximizes viewport; all panels dockable, detachable, and multi-monitor aware
- Dashboard showing real-time PDM status, downstream simulation results, and change-order queue

**Feature Gating:**
- All features unlocked including topology optimization, generative design, and multi-physics co-simulation hooks
- Full PLM/ERP integration with change-management and release workflows
- Admin controls for template libraries, design standards enforcement, and team permission policies

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Migration assistant that bulk-imports legacy libraries (materials, fasteners, templates) and maps custom macros
- Architecture walkthrough for API/plugin development and CI/CD integration with CAD validation
- Personalized session with solutions engineer to configure enterprise PLM connector

**Key Workflows:**
- Designing complex multi-body surfacing and Class-A geometry for production tooling
- Defining and enforcing company-wide parametric templates and design standards
- Orchestrating large assemblies (10,000+ components) with lightweight referencing and configurations
- Automating repetitive design tasks via macros, design tables, and API scripts
- Reviewing and approving junior engineers' models through formal design-review workflows

---

## Role-Based Personas

### Designer/Creator

**Modified UI/UX:**
- Prominent sketch and surface tools; rendering and visualization panel pinned to sidebar
- Real-time material appearance preview and environment lighting controls in the viewport
- Mood-board / reference-image overlay tool for translating industrial design concepts into parametric geometry

**Feature Gating:**
- Full surfacing, freeform, and subdivision modeling tools available
- Integrated rendering engine (PBR materials, HDR environments) and animation timeline unlocked
- FEA and manufacturing-prep tools available but collapsed by default

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- "Design Language Setup" wizard to define default materials, colors, and rendering presets
- Import reference sketches and concept art to trace over in 3D
- Quick-start template gallery organized by product category (consumer electronics, furniture, automotive trim)

**Key Workflows:**
- Creating organic and freeform surface geometry for consumer-product housings
- Iterating on design variants with configuration-driven appearance studies
- Producing photorealistic renders and turntable animations for stakeholder review
- Exporting presentation-ready visuals and dimensioned drawings for engineering handoff
- Collaborating with engineering on DFM feedback loops within the same model

---

### Analyst/Simulator

**Modified UI/UX:**
- Simulation-centric workspace with meshing controls, boundary-condition panels, and results post-processor front and center
- Geometry simplification toolbar (defeaturing, mid-surface extraction) docked for quick model prep
- Convergence and residual plots displayed alongside 3D results in a split-pane layout

**Feature Gating:**
- Linear and nonlinear static FEA, modal analysis, thermal simulation, and fatigue life prediction unlocked
- Topology optimization and AI-driven mesh refinement available
- CAD modeling tools available in read-only or simplified mode to prevent accidental geometry changes

**Pricing Tier:** Professional or Enterprise tier (depending on solver seat count)

**Onboarding Flow:**
- Benchmark validation suite: run a set of known problems (cantilever beam, pressure vessel) and compare against analytical solutions
- Guided workflow: "Prepare Geometry > Mesh > Apply Loads > Solve > Post-Process" with validation checkpoints
- Integration setup for external solvers (NASTRAN, Abaqus) if the team uses hybrid workflows

**Key Workflows:**
- Defeaturing and simplifying imported CAD geometry for efficient meshing
- Setting up linear static and modal analyses with appropriate boundary conditions
- Running parametric design studies varying geometry dimensions and evaluating stress/deflection
- Generating simulation summary reports with contour plots, tables, and safety-factor assessments
- Validating results against hand calculations or test data and documenting convergence studies

---

### Manufacturing/Process

**Modified UI/UX:**
- Manufacturing-prep toolbar (draft analysis, undercut detection, wall-thickness check) pinned to primary ribbon
- DFM/DFA advisor panel that flags issues in real time as geometry is modified
- Tooling split-line and parting-surface tools prominently accessible

**Feature Gating:**
- Mold design, sheet-metal flat-pattern, weldment cut-list, and CAM toolpath preview unlocked
- Full BOM export with material specifications and process routing annotations
- Generative design and advanced surfacing available but de-emphasized in toolbar

**Pricing Tier:** Professional tier with Manufacturing add-on

**Onboarding Flow:**
- "Manufacturing Profile" setup: select primary processes (injection molding, CNC machining, sheet metal, casting) to tailor tool visibility
- Import sample parts with known manufacturability issues to demonstrate DFM advisor capabilities
- Configure default tolerancing standards (ISO 2768, ASME Y14.5) and material libraries

**Key Workflows:**
- Running draft-angle and wall-thickness analyses on injection-molded part designs
- Generating flat patterns and bend tables for sheet-metal fabrication
- Creating mold core/cavity splits with runner and gate layout suggestions
- Exporting manufacturing-ready drawings with tolerance stack-ups and process notes
- Feeding part geometry into CAMWorks for integrated CNC toolpath generation

---

### Regulatory/Compliance

**Modified UI/UX:**
- Compliance dashboard showing traceability matrix: requirement ID mapped to geometry features and simulation results
- Change-log panel with full audit trail of every geometry edit, approval, and version release
- Standards-reference sidebar with quick links to applicable codes (ASME, ISO, ASTM, FDA 21 CFR)

**Feature Gating:**
- Full model-versioning with electronic signatures and approval workflows (21 CFR Part 11 capable)
- Automated design-rule checks against configurable standard templates
- Export of validation/verification documentation packages (IQ/OQ/PQ for medical, DO-178C for aerospace)

**Pricing Tier:** Enterprise tier with Compliance module

**Onboarding Flow:**
- Compliance-profile wizard: select industry (medical device, aerospace, automotive, nuclear) to load relevant rule sets
- Walk through e-signature configuration and audit-trail setup with IT/QA stakeholders
- Demo traceability workflow: requirement > design feature > analysis > test > release

**Key Workflows:**
- Tagging CAD features with requirement IDs and maintaining bidirectional traceability
- Running automated design-rule checks (minimum radii, material compliance, labeling requirements)
- Generating audit-ready documentation packages with versioned drawings and simulation evidence
- Managing engineering change orders with full before/after comparison and sign-off chains
- Producing regulatory submission artifacts (e.g., device master record for FDA, type-certificate data for EASA)

---

### Manager/Decision-maker

**Modified UI/UX:**
- Executive dashboard with project health KPIs: schedule adherence, design maturity index, open issues, and resource utilization
- Lightweight 3D viewer for design reviews without requiring full CAD proficiency
- Portfolio view showing all active projects with status, milestone dates, and risk flags

**Feature Gating:**
- Read-only access to all models and assemblies with markup and comment tools
- Project analytics, resource allocation, and license-usage reports unlocked
- No direct geometry editing; all changes routed through formal change-request process

**Pricing Tier:** Enterprise tier (manager seat, reduced cost)

**Onboarding Flow:**
- "Manager Cockpit" setup: connect to PLM/PM tools (Jira, Azure DevOps, Windchill) for unified status view
- Walkthrough of design-review workflow: how to view, annotate, and approve/reject models
- Configure notification rules for milestone completions, overdue tasks, and escalations

**Key Workflows:**
- Reviewing project status dashboards and identifying schedule or resource bottlenecks
- Participating in formal design reviews with 3D markup annotations and approval/rejection actions
- Tracking license utilization and optimizing seat allocation across teams
- Generating executive reports on design maturity, defect rates, and time-to-release metrics
- Approving engineering change orders and release packages in the PLM workflow
