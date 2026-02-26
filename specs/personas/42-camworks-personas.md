# CAMWorks — Persona Subspecs

> Parent spec: [`specs/42-camworks.md`](../42-camworks.md)

---

## Experience-Level Personas

### Student/Academic

**Modified UI/UX:**
- Simplified 2.5D milling and turning interface with step-by-step operation wizard; multi-axis and Swiss-type panels hidden
- Interactive toolpath visualization with color-coded cut depths and animated tool motion at adjustable speed
- Built-in reference panel showing feeds/speeds formulas and chip-load calculations alongside active parameters

**Feature Gating:**
- 2.5D milling (face, pocket, contour, drill), basic turning, and single-setup 3-axis profiling available
- 4/5-axis simultaneous machining, post-processor editor, and machine-tool builder locked
- Simulation limited to basic gouge detection; full material-removal verification unavailable

**Pricing Tier:** Free tier (education license)

**Onboarding Flow:**
- "Machine Your First Part" tutorial: import a simple bracket, select stock, auto-recognize features, generate toolpath, and simulate
- Virtual machine lab with pre-configured 3-axis mill and 2-axis lathe — no post-processor setup needed
- Quiz-based progress tracker aligned with manufacturing-technology course curricula

**Key Workflows:**
- Importing STEP files and setting up stock geometry for 2.5D milling operations
- Using automatic feature recognition to generate pocket and hole operations
- Adjusting feeds, speeds, and depth-of-cut parameters and observing toolpath changes
- Running basic collision-check simulations and reviewing cycle-time estimates
- Exporting G-code for classroom CNC machines with pre-configured post-processors

---

### Junior Engineer (0-3 years)

**Modified UI/UX:**
- Full operation manager with feature tree, tool library, and operation parameters panel in standard layout
- "Best Practice" indicators on feed/speed selections flagging values outside recommended ranges for the chosen material/tool combination
- Side-by-side toolpath and simulation view with one-click toggle between wireframe and solid verification

**Feature Gating:**
- Full 3-axis milling, turning, and mill-turn operations unlocked
- 4-axis indexed and basic 5-axis positional machining available
- Post-processor customization limited to parameter tweaks; full post editor restricted to senior users

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Machine-profile setup wizard: select CNC machine model from library (or import from vendor) to auto-configure work envelope and kinematics
- Guided first-program workflow that walks through stock setup, feature recognition, operation ordering, and verification
- 5-day training series covering tool-library management, operation templates, and shop-floor best practices

**Key Workflows:**
- Programming 3-axis milling jobs from engineering models with automatic feature recognition
- Building and managing tool libraries with geometry, material grades, and recommended cutting data
- Creating turning programs with roughing, finishing, grooving, and threading operations
- Running full material-removal simulation to verify no gouges, collisions, or excess stock
- Generating shop-floor documentation: setup sheets, tool lists, and annotated operation sequences

---

### Senior/Expert

**Modified UI/UX:**
- Multi-document workspace with simultaneous part programs, macro editor, and post-processor development environment
- Custom operation-template builder with parametric rules for automatic feature-to-operation mapping
- Machine-load analytics dashboard showing spindle utilization, cycle-time breakdown, and tool-wear projections

**Feature Gating:**
- All operations unlocked: 5-axis simultaneous, Swiss-type, multi-spindle/multi-turret, and wire EDM
- Full post-processor editor with G-code line-by-line debugging and custom macro injection
- API access for batch programming, ERP/MES integration, and automated job scheduling

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Legacy program migration tool: import existing post-processors, tool libraries, and operation templates from Mastercam/Fusion/Esprit
- Advanced post-processor development workshop with machine-builder and kinematic-chain configuration
- Solutions-engineering session to integrate with shop-floor MES, tool-management, and ERP systems

**Key Workflows:**
- Programming complex 5-axis simultaneous toolpaths for turbine blades, impellers, and mold cavities
- Developing and maintaining custom post-processors for specialized CNC machines
- Creating parametric operation templates that auto-program families of similar parts
- Optimizing cycle times through toolpath strategies (trochoidal, adaptive clearing, rest machining)
- Integrating CAM output with MES for automated job scheduling, tool-preset data, and machine monitoring

---

## Role-Based Personas

### Designer/Creator

**Modified UI/UX:**
- Lightweight CAM interface embedded within CAD environment showing manufacturability feedback in real time
- DFM heat-map overlay on 3D model highlighting areas that are difficult or impossible to machine
- Simplified "one-click estimate" button that returns rough cycle time and cost without full programming

**Feature Gating:**
- Quick-estimate and DFM-check tools available; full toolpath programming accessible but secondary
- Material-removal visualization for design validation without needing to define full operations
- Post-processor and G-code output restricted — results are advisory, not production-ready

**Pricing Tier:** Professional tier (Design-to-Manufacturing add-on)

**Onboarding Flow:**
- "Design for Machining" tutorial: import a part, run DFM check, see flagged features, and iterate geometry
- Preset machine capability profiles (3-axis mill, 5-axis mill, lathe) for quick feasibility assessment
- Link to CADForge or native CAD for seamless geometry modification based on DFM feedback

**Key Workflows:**
- Running DFM checks on part designs to identify unmachineable features (deep pockets, sharp internal corners)
- Getting rapid cost and cycle-time estimates for design-trade studies
- Visualizing tool-access limitations to inform design decisions on draft angles and radii
- Collaborating with manufacturing engineers by sharing annotated DFM reports
- Iterating on geometry in CAD and re-running manufacturability assessment in a tight loop

---

### Analyst/Simulator

**Modified UI/UX:**
- Simulation-focused workspace with full material-removal verification, cutting-force prediction, and chatter-stability analysis front and center
- Toolpath analytics dashboard: feed-rate distribution, chip-load consistency, and engagement-angle plots
- Comparative view to overlay simulated surface finish against tolerance specifications

**Feature Gating:**
- Full volumetric material-removal simulation with gouge, collision, and excess-stock detection
- Cutting-force and power prediction, chatter stability lobes, and tool-deflection estimation unlocked
- Toolpath editing limited; simulation results feed back as recommendations to the programmer

**Pricing Tier:** Professional tier with Simulation add-on

**Onboarding Flow:**
- Benchmark simulation: run a known test-cut scenario and compare predicted forces against published data
- Configure machine dynamics (spindle speed range, power curve, axis limits) for accurate predictions
- Tutorial on interpreting chatter stability diagrams and selecting stable spindle speed/depth combinations

**Key Workflows:**
- Verifying toolpaths via full material-removal simulation before sending to the shop floor
- Predicting cutting forces and power requirements to prevent tool breakage and machine overload
- Analyzing chatter stability to select optimal spindle speeds and axial depths of cut
- Evaluating surface-finish quality from toolpath scallop-height and feed-mark analysis
- Generating simulation reports with pass/fail criteria for quality review before production release

---

### Manufacturing/Process

**Modified UI/UX:**
- Shop-floor-oriented dashboard showing active jobs, machine assignments, tool inventories, and setup status
- Setup-sheet generator with fixture diagrams, tool-preset data, and step-by-step operator instructions
- Tool-wear tracking panel with replacement alerts and re-order triggers linked to tool-management system

**Feature Gating:**
- Full programming, simulation, and G-code output for all machine types
- Shop-floor documentation generator (setup sheets, tool lists, inspection plans) unlocked
- MES/ERP integration for job scheduling, material tracking, and machine-utilization reporting

**Pricing Tier:** Professional tier with Shop-Floor module

**Onboarding Flow:**
- Shop-floor integration setup: connect to tool-presetter, MES, and machine-monitoring systems
- Configure standard setup-sheet templates with company branding, fixture conventions, and QC checkpoints
- Import existing tool inventory from tool-management system or spreadsheet

**Key Workflows:**
- Generating complete shop-floor documentation packages (setup sheets, tool lists, G-code) for each job
- Managing tool inventories with usage tracking, wear compensation, and re-order automation
- Scheduling CNC jobs across machines based on capability, availability, and due-date priority
- Tracking machine utilization and cycle-time actuals vs. estimates for continuous improvement
- Coordinating first-article inspections with integrated CMM probe routines

---

### Regulatory/Compliance

**Modified UI/UX:**
- Audit-trail panel showing every toolpath edit, parameter change, and approval with timestamps and user IDs
- Validation dashboard mapping machining operations to process specifications (AS9100, ISO 13485, NADCAP)
- Locked-program indicator showing which G-code files have been validated and are frozen for production

**Feature Gating:**
- Electronic signature and approval workflows on program releases (21 CFR Part 11 compliant)
- Program-freeze capability preventing edits to validated G-code without formal change order
- Full traceability from design revision through CAM program version to production batch records

**Pricing Tier:** Enterprise tier with Compliance module

**Onboarding Flow:**
- Compliance profile setup: select applicable standards (AS9100, ISO 13485, NADCAP, ITAR) to configure audit requirements
- Configure electronic-signature workflows with role-based approval chains (programmer > lead > QA)
- Demo end-to-end traceability: design revision > CAM program > G-code release > production record

**Key Workflows:**
- Maintaining version-controlled CAM programs with full change history and electronic approvals
- Generating process-validation documentation for regulatory audits (IQ/OQ/PQ for medical devices)
- Enforcing program-freeze policies that prevent unauthorized modifications to released G-code
- Producing traceability reports linking part serial numbers to specific program versions and machine setups
- Supporting NADCAP and AS9100 audits with automated evidence packages

---

### Manager/Decision-maker

**Modified UI/UX:**
- Executive dashboard with shop-floor KPIs: machine utilization, on-time delivery rate, scrap rate, and cost-per-part trends
- Capacity-planning view showing machine load forecasts and bottleneck identification
- Cost-comparison panels for make-vs-buy decisions and quoting new work

**Feature Gating:**
- Read-only access to programs and simulations with annotation and approval tools
- Full analytics, reporting, and capacity-planning modules unlocked
- No direct G-code editing; all changes routed through formal programming workflow

**Pricing Tier:** Enterprise tier (manager seat)

**Onboarding Flow:**
- "Manager Dashboard" setup: connect to ERP/MES for live production data feeds
- Walkthrough of KPI definitions, alerting thresholds, and escalation rules
- Configure cost-model parameters (machine hourly rates, tool costs, material costs) for accurate quoting

**Key Workflows:**
- Monitoring shop-floor KPIs and identifying machines or jobs that are underperforming
- Reviewing capacity plans and making investment decisions on new machines or shifts
- Approving program releases and engineering change orders through formal workflows
- Generating cost estimates for new-part quotations based on CAM cycle-time predictions
- Tracking continuous-improvement metrics (cycle-time reduction, scrap reduction, OEE trends)
