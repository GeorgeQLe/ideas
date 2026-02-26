# CircuitMind — Persona Subspecs

> Parent spec: [`specs/21-circuitmind.md`](../21-circuitmind.md)

---

## Experience-Level Personas

### Student/Academic

**Modified UI/UX:**
- Show guided schematic templates (e.g., "Arduino Shield", "LED Driver") with annotated component explanations on the canvas
- Enable contextual tooltips on every component pin, trace, and design rule violation with links to learning resources
- Simplify the layer stack manager to a 2-layer default with a toggle to reveal advanced multi-layer options

**Feature Gating:**
- Expose schematic capture, basic DRC, and 2-layer PCB layout; hide impedance-controlled routing, flex-PCB, and HDI via options
- Limit AI auto-router to single-sided or 2-layer boards; lock advanced constraint-driven routing behind upgrade prompt
- Provide full access to component library browser but restrict BOM export to CSV only (no ERP integrations)

**Pricing Tier:** Free tier

**Onboarding Flow:**
- First-run tutorial builds a blinking LED circuit from schematic to routed PCB in under 10 minutes with step-by-step narration
- Prompt user to join a "Classroom" workspace if they select academic affiliation during signup
- Offer a curated list of 10 starter projects with increasing complexity as a learning path

**Key Workflows:**
- Create a schematic from a course assignment template and validate against instructor-provided DRC rules
- Lay out a simple 2-layer PCB using the auto-router and export Gerber files for a university PCB fab service
- Share a read-only project link with a professor or TA for grading and feedback
- Browse the component library to learn about part categories, packages, and footprints

---

### Junior Engineer (0-3 years)

**Modified UI/UX:**
- Display the full schematic and PCB editor toolset with inline documentation panels that explain each tool on hover
- Show a "Design Health" sidebar summarizing DRC errors, unrouted nets, and BOM warnings in plain language
- Default to a guided layout workflow (place connectors first, then ICs, then passives) with the option to switch to free-form

**Feature Gating:**
- Unlock 4-layer PCB layout, differential pair routing, and basic impedance calculators
- Expose the AI auto-router with full constraint support but keep advanced signal integrity simulation behind a confirmation dialog explaining its complexity
- Enable real-time collaboration and version history; restrict branch-merge workflows to Pro tier

**Pricing Tier:** Pro tier ($29/month)

**Onboarding Flow:**
- Walk through importing a reference design (e.g., an ESP32 dev board) and modifying it to build confidence with the tool
- Offer a "PCB Design Checklist" panel that tracks progress (schematic complete, footprints assigned, DRC clean, BOM reviewed)
- Prompt to connect a component supplier account (DigiKey/Mouser) for live stock and pricing in the BOM

**Key Workflows:**
- Design a 4-layer mixed-signal PCB with power planes, signal routing, and a ground pour
- Run DRC iteratively during layout and resolve violations with AI-suggested fixes
- Generate a production-ready BOM with supplier pricing and lead-time data for procurement review
- Use real-time collaboration to get a senior engineer's review on critical trace routing
- Export Gerber, drill, and pick-and-place files for a PCB manufacturer

---

### Senior/Expert

**Modified UI/UX:**
- Provide a power-user layout with customizable toolbar, keyboard shortcut editor, and scriptable command palette
- Show advanced analytics: trace length matching histograms, impedance profiles, thermal via density heatmaps
- Support multi-monitor workflows with detachable schematic, PCB, and 3D viewer panels

**Feature Gating:**
- Full access to all features: HDI/microvia, flex-rigid design, RF layout tools, impedance-controlled routing, IBIS simulation
- Expose scripting API for design automation (batch DRC, parametric footprint generation, custom design rules)
- Enable enterprise SSO, audit logging, and IP-controlled project access

**Pricing Tier:** Enterprise tier ($99/month or annual contract)

**Onboarding Flow:**
- Skip tutorials; present a "What's different from Altium/KiCad" migration guide with keyboard shortcut mapping
- Offer bulk project import from KiCad, Eagle, and Altium formats with automated library migration
- Prompt to configure design rule templates and preferred component libraries on first project creation

**Key Workflows:**
- Design a 10+ layer HDI PCB with blind/buried vias, impedance-controlled differential pairs, and RF transmission lines
- Run post-layout signal integrity and power integrity simulations to validate high-speed design
- Automate repetitive design tasks via the scripting API (e.g., generate variant BOMs, parametric panelization)
- Manage multiple concurrent projects with branching, design reviews, and release tagging
- Integrate with PLM and ERP systems for production handoff with full manufacturing data packages

---

## Role-Based Personas

### Designer/Creator

**Modified UI/UX:**
- Emphasize the schematic capture canvas with a large component palette, smart wiring, and aesthetic net labeling
- Provide a 3D board preview with enclosure import (STEP) for mechanical fit checks directly in the design view
- Show a visual pin-mapping tool for complex ICs that overlays datasheet pinouts on the schematic symbol

**Feature Gating:**
- Full schematic and PCB layout tools including the component library editor for creating custom symbols and footprints
- Expose the AI auto-router and interactive router but hide post-layout simulation tools (signal/power integrity)
- Enable design template creation and sharing within the team workspace

**Pricing Tier:** Pro tier ($29/month)

**Onboarding Flow:**
- Start with a blank schematic and guide the user through placing a microcontroller, adding peripherals, and wiring
- Offer to import an existing schematic from KiCad/Eagle to demonstrate seamless migration
- Highlight the real-time collaboration features for co-designing with colleagues

**Key Workflows:**
- Capture a complete schematic with hierarchical sheets for complex designs
- Assign footprints and validate component availability before starting PCB layout
- Route the PCB using a combination of AI auto-routing and manual fine-tuning for critical paths
- Generate a 3D model of the assembled board for mechanical review
- Publish a design review link for stakeholders to inspect and annotate the board

---

### Analyst/Simulator

**Modified UI/UX:**
- Foreground the SPICE simulation panel with waveform viewer, probe placement tools, and measurement cursors
- Show a dedicated signal integrity dashboard with eye diagrams, crosstalk analysis, and frequency-domain plots
- Provide a simulation setup wizard that auto-detects circuit topology and suggests appropriate analyses (AC, DC, transient)

**Feature Gating:**
- Full access to SPICE simulation, signal integrity, power integrity, and thermal analysis tools
- Expose Monte Carlo simulation for component tolerance analysis and worst-case design verification
- Enable simulation result export (CSV, Touchstone S-parameter) and comparison across design revisions

**Pricing Tier:** Pro tier ($29/month) or Enterprise for batch simulation

**Onboarding Flow:**
- Present a "Simulation Quick Start" that runs a transient simulation on a sample power supply circuit
- Show how to add SPICE models to components and configure simulation parameters
- Demonstrate the waveform comparison tool for evaluating design changes

**Key Workflows:**
- Run SPICE simulations on analog subcircuits (op-amp filters, voltage regulators, oscillators)
- Perform signal integrity analysis on high-speed digital interfaces (USB, PCIe, DDR)
- Analyze power distribution network impedance and decoupling capacitor placement
- Run thermal simulations to identify hotspots and validate copper pour thermal relief
- Compare simulation results across design revisions to quantify the impact of layout changes

---

### Manufacturing/Process

**Modified UI/UX:**
- Prioritize the manufacturing output panel: Gerber viewer, drill table, layer stackup summary, and pick-and-place coordinates
- Show a DFM (Design for Manufacturability) audit view highlighting potential fabrication issues (acid traps, copper slivers, annular ring violations)
- Display the BOM manager with supplier columns, alternate parts, and lifecycle status prominently

**Feature Gating:**
- Full access to Gerber/ODB++ export, panelization tools, and assembly drawing generation
- Expose DFM rule checking with configurable rules per fabricator capability profile
- Enable BOM-level supplier integration (DigiKey, Mouser, LCSC) with stock alerts and cost roll-ups

**Pricing Tier:** Pro tier ($29/month)

**Onboarding Flow:**
- Walk through exporting a complete manufacturing package (Gerber, drill, BOM, pick-and-place) from a sample design
- Prompt to configure a preferred fabricator profile (e.g., JLCPCB, PCBWay) for DFM rule validation
- Show how to set up alternate components and manage BOM cost optimization

**Key Workflows:**
- Run DFM checks against a specific fabricator's capability profile before ordering boards
- Generate panelized Gerber files with breakaway tabs and fiducial markers
- Export a costed BOM with preferred and alternate suppliers, highlighting long-lead-time or end-of-life parts
- Create assembly drawings and pick-and-place files for automated SMT assembly
- Track component lifecycle status and receive alerts when parts are marked NRND or obsolete

---

### Regulatory/Compliance

**Modified UI/UX:**
- Add a compliance dashboard showing EMC design rule status, controlled-impedance trace audit, and safety clearance checks
- Highlight keep-out zones, creepage/clearance distances, and UL/IEC spacing requirements directly on the PCB layout
- Provide a document generation panel for compliance test preparation (test point map, grounding topology diagram)

**Feature Gating:**
- Expose EMC-specific DRC rules (trace-to-edge clearance, guard rings, split plane analysis)
- Enable compliance report generation with checklist templates for FCC, CE, UL, and IEC standards
- Provide access to the design audit trail showing every change with timestamps for regulatory documentation

**Pricing Tier:** Enterprise tier ($99/month)

**Onboarding Flow:**
- Start with a compliance-focused workspace template that pre-loads relevant DRC rules for the target standard (e.g., IEC 62368-1)
- Walk through how to annotate safety-critical nets (mains voltage, protective earth) and verify spacing rules
- Demonstrate the audit trail and change history features for regulatory submission preparation

**Key Workflows:**
- Verify creepage and clearance distances meet IEC/UL requirements for the product's voltage class
- Run EMC-focused DRC checks (ground plane continuity, signal return paths, decoupling placement)
- Generate a compliance documentation package with design rationale, test point locations, and grounding diagrams
- Review the full design change audit trail for inclusion in a regulatory submission
- Export annotated layout images highlighting safety-critical features for certification lab communication

---

### Manager/Decision-maker

**Modified UI/UX:**
- Present a project portfolio dashboard showing all active designs, their status (schematic/layout/review/manufacturing), and team assignments
- Show cost summary widgets: estimated BOM cost per unit, PCB fabrication quotes, and total project budget tracking
- Provide a team activity feed with recent changes, review requests, and milestone completions

**Feature Gating:**
- Read-only access to all designs with commenting and approval capabilities; no direct editing
- Expose project management features: milestone tracking, design review scheduling, approval workflows
- Enable cost analysis tools: BOM cost roll-ups, supplier comparison, and volume pricing estimates

**Pricing Tier:** Enterprise tier ($99/month)

**Onboarding Flow:**
- Set up the project portfolio view and invite team members with appropriate role assignments
- Walk through the design review and approval workflow using a sample project
- Demonstrate cost reporting and how to compare BOM costs across design revisions

**Key Workflows:**
- Review project status across all active PCB designs from a single dashboard
- Approve or request changes on designs submitted for review, with inline annotations
- Analyze BOM cost trends and compare supplier quotes for procurement decisions
- Track team utilization and project timelines against delivery milestones
- Generate executive summaries of design progress and cost estimates for stakeholder reporting
