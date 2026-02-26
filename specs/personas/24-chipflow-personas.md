# ChipFlow — Persona Subspecs

> Parent spec: [`specs/24-chipflow.md`](../24-chipflow.md)

---

## Experience-Level Personas

### Student/Academic

**Modified UI/UX:**
- Provide a guided layout editor with pre-loaded educational PDKs (SKY130, GF180MCU) and annotated standard cell libraries
- Show contextual explanations for DRC violations (e.g., "Minimum metal spacing is 0.14um because...") linked to course material
- Simplify the tool palette to basic layout operations (draw rectangle, path, via) and hide advanced PnR and parasitic extraction

**Feature Gating:**
- Allow schematic entry, basic layout, and DRC with educational PDKs; hide production PDK support and tapeout export
- Limit SPICE simulation to circuits under 100 transistors; disable parasitic extraction and LVS for complex designs
- Provide access to a shared classroom workspace with instructor-managed projects

**Pricing Tier:** Free tier (academic license)

**Onboarding Flow:**
- First-run tutorial guides the user through drawing an NMOS transistor layout in the SKY130 PDK and running DRC
- Offer a set of graded lab exercises (inverter, NAND gate, ring oscillator, SRAM cell) as structured learning projects
- Prompt to join a classroom workspace using an instructor-provided code

**Key Workflows:**
- Draw transistor-level layouts for basic logic gates and verify with DRC
- Run SPICE simulations of small circuits (inverter chain, differential pair) and view waveforms
- Complete structured lab assignments and submit for instructor review within the classroom workspace
- Explore standard cell library layouts to understand physical design principles

---

### Junior Engineer (0-3 years)

**Modified UI/UX:**
- Show the full layout editor with guided workflows for common tasks (standard cell design, block-level floorplanning)
- Display a "Design Progress" tracker showing completion status for each verification step (DRC, LVS, PEX, timing)
- Provide visual overlays for design rules (minimum width markers, spacing guides) that update as the user draws

**Feature Gating:**
- Unlock full layout editing, DRC, LVS, and parasitic extraction for designs up to 10K transistors
- Expose Verilog RTL simulation and basic synthesis with educational and foundry PDKs
- Hide multi-project wafer (MPW) tapeout tools and advanced analog design features (matching, symmetry constraints)

**Pricing Tier:** Pro tier ($79/month)

**Onboarding Flow:**
- Walk through a complete standard cell design flow: schematic entry, layout, DRC, LVS, and parasitic extraction
- Demonstrate importing a Verilog RTL design and running synthesis to generate a gate-level netlist
- Show how to use the design progress tracker to ensure all verification steps are completed before sign-off

**Key Workflows:**
- Design standard cells (logic gates, flip-flops, clock buffers) with full DRC and LVS verification
- Perform block-level floorplanning and place-and-route for small digital designs
- Run SPICE simulations with extracted parasitics to validate post-layout performance
- Synthesize Verilog RTL to gate-level netlists and verify timing constraints
- Collaborate with team members on shared design projects with version control

---

### Senior/Expert

**Modified UI/UX:**
- Provide a multi-pane workspace with synchronized schematic, layout, and waveform views for efficient cross-probing
- Expose the full scripting interface (Python/Tcl) for design automation, custom DRC rules, and batch operations
- Support configurable keyboard shortcuts and macro recording for repetitive layout operations

**Feature Gating:**
- Full access: production PDKs, advanced analog layout (matching, guard rings, common-centroid), full-chip integration, and tapeout export
- Enable multi-project wafer (MPW) submission tools with foundry-specific checks (e.g., Efabless/Google MPW)
- Unlock team management, IP block library management, and integration with external EDA tools via OpenAccess/LEF-DEF

**Pricing Tier:** Enterprise tier ($249/month or institutional contract)

**Onboarding Flow:**
- Present a migration guide mapping Cadence Virtuoso / Synopsys ICC2 concepts and shortcuts to ChipFlow equivalents
- Offer bulk import of existing design libraries (GDS-II, LEF/DEF, Verilog, SPICE netlists)
- Configure PDK bindings, DRC rule decks, and simulation setups for the user's target process node

**Key Workflows:**
- Design complex analog/mixed-signal blocks with matching constraints, guard rings, and symmetry enforcement
- Run full-chip integration: floorplanning, power grid design, clock tree synthesis, and final DRC/LVS sign-off
- Automate repetitive design tasks via Python scripting (parametric cell generators, batch DRC, netlist extraction)
- Prepare and submit tapeout packages for MPW shuttles or production runs with foundry-specific compliance checks
- Manage a team's IP block library with versioned cells, characterization data, and reuse guidelines

---

## Role-Based Personas

### Designer/Creator

**Modified UI/UX:**
- Emphasize the schematic capture and layout drawing tools with a large canvas, snapping guides, and instant DRC feedback
- Provide a component palette with parameterized cells (PCells) that auto-generate layouts from user-specified dimensions
- Show a real-time 3D cross-section view of the layer stack as metal layers and vias are drawn

**Feature Gating:**
- Full access to schematic entry, layout editing, PCell creation, and visual design tools
- Expose DRC and basic LVS; hide timing analysis, power analysis, and tapeout packaging
- Enable design template creation and library contribution workflows

**Pricing Tier:** Pro tier ($79/month)

**Onboarding Flow:**
- Start with a schematic-driven layout exercise: draw a schematic, generate a layout skeleton, and complete the physical design
- Demonstrate PCell creation by parameterizing a transistor layout with configurable width and finger count
- Show the 3D cross-section viewer to build intuition for layer stacking and via placement

**Key Workflows:**
- Capture schematics for analog and digital circuits with hierarchical design organization
- Create transistor-level layouts with DRC-clean geometry using snapping and alignment tools
- Build parameterized cells (PCells) for reusable, scalable layout generation
- Design custom standard cells with characterized timing and power data
- Create block-level floorplans with pin placement and power rail planning

---

### Analyst/Simulator

**Modified UI/UX:**
- Foreground the SPICE simulation environment with waveform viewer, measurement tools, and corner analysis setup
- Show a timing analysis dashboard with setup/hold slack histograms, critical path highlighting, and clock domain diagrams
- Provide a parasitic extraction control panel with configurable accuracy modes (RC, RCC, full EM) and result comparison tools

**Feature Gating:**
- Full access to SPICE simulation, Verilog simulation, parasitic extraction, and timing/power analysis
- Expose Monte Carlo simulation for process variation analysis and yield estimation
- Enable batch simulation with parameter sweeps and automated result aggregation

**Pricing Tier:** Pro tier ($79/month)

**Onboarding Flow:**
- Walk through running a SPICE simulation with extracted parasitics and comparing pre- vs. post-layout performance
- Demonstrate the timing analysis workflow: import constraints, run STA, identify and fix violations
- Show how to set up corner analysis across process, voltage, and temperature (PVT) conditions

**Key Workflows:**
- Run pre-layout and post-layout SPICE simulations to validate circuit performance across PVT corners
- Perform static timing analysis on digital blocks and identify critical paths for optimization
- Extract parasitics at various accuracy levels and assess their impact on circuit performance
- Run Monte Carlo simulations to estimate yield and identify sensitivity to process variations
- Generate characterization data (timing, power, noise) for standard cells and IP blocks

---

### Manufacturing/Process

**Modified UI/UX:**
- Show a tapeout readiness dashboard with checklist items for DRC, LVS, ERC, antenna rules, and metal density compliance
- Display a GDS-II export panel with layer mapping configuration and foundry-specific output format options
- Provide a metal density analysis view with fill pattern generation and uniformity metrics

**Feature Gating:**
- Full access to tapeout preparation tools: final DRC, LVS, ERC, antenna check, and metal fill generation
- Expose GDS-II/OASIS export with foundry-specific layer mapping and format validation
- Enable MPW submission workflows with integrated foundry design rule checking

**Pricing Tier:** Enterprise tier ($249/month)

**Onboarding Flow:**
- Walk through preparing a design for tapeout: final DRC clean, metal fill, GDS export, and submission checklist
- Configure the foundry target (TSMC, GlobalFoundries, SKY130) and load the corresponding DRC/LVS rule decks
- Demonstrate the metal density analysis and automatic fill generation workflow

**Key Workflows:**
- Run final design verification (DRC, LVS, ERC, antenna) with foundry-specific rule decks before tapeout
- Generate metal fill patterns to meet density requirements and verify uniformity across the chip
- Export GDS-II/OASIS files with correct layer mapping for the target foundry
- Prepare MPW shuttle submissions with all required deliverables (GDS, documentation, test plans)
- Track tapeout checklists and sign-off status across all verification categories

---

### Regulatory/Compliance

**Modified UI/UX:**
- Add an IP provenance panel showing the origin, license, and modification history of every cell and block in the design
- Display export control status indicators for designs containing controlled technology
- Provide a design audit log with timestamps, user attribution, and change descriptions for every modification

**Feature Gating:**
- Expose the full audit trail with immutable records of all design changes and access events
- Enable IP block tagging with license type, export control classification, and usage restrictions
- Unlock compliance report generation for ITAR/EAR export control and foundry NDA requirements

**Pricing Tier:** Enterprise tier ($249/month)

**Onboarding Flow:**
- Configure IP classification tags for all blocks in the design (open-source, proprietary, export-controlled)
- Set up access control policies that enforce need-to-know restrictions on sensitive design data
- Demonstrate the compliance audit report generation workflow

**Key Workflows:**
- Maintain a complete design change audit trail for IP provenance documentation
- Tag and track export-controlled design components for ITAR/EAR compliance
- Generate compliance reports documenting IP origins, licenses, and access history
- Enforce access controls on sensitive design data based on classification level and team role
- Prepare documentation for foundry NDA compliance and technology transfer agreements

---

### Manager/Decision-maker

**Modified UI/UX:**
- Present a project portfolio dashboard showing all active chip designs with stage (schematic, layout, verification, tapeout), schedule, and cost
- Show resource utilization: engineer assignments, EDA license usage, compute hours consumed
- Provide tapeout schedule Gantt charts with milestone tracking and dependency visualization

**Feature Gating:**
- Read-only access to all designs with annotation and approval capabilities
- Expose project management tools: milestone tracking, resource allocation, and tapeout schedule management
- Enable cost tracking for compute, licenses, and foundry fees with budget forecasting

**Pricing Tier:** Enterprise tier ($249/month)

**Onboarding Flow:**
- Set up the project portfolio view with active chip designs and team assignments
- Walk through the tapeout schedule management workflow with milestone dependencies
- Demonstrate the cost dashboard showing compute, license, and foundry fee breakdowns

**Key Workflows:**
- Monitor all chip design projects from a unified dashboard with stage-gate status tracking
- Manage tapeout schedules with milestone tracking, critical path analysis, and dependency management
- Track costs across compute resources, EDA licenses, and foundry fees against project budgets
- Review and approve design milestones (schematic freeze, layout freeze, tapeout sign-off)
- Generate progress reports for leadership and investors covering schedule, cost, and technical risk
