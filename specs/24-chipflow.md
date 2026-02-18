# ChipFlow — Cloud-Native IC/ASIC Layout Design Platform

## Executive Summary

ChipFlow is a cloud-native, browser-based integrated circuit design platform that brings GPU-accelerated GDS-II layout rendering, SPICE circuit simulation, Verilog RTL simulation, design rule checking, and parasitic extraction to semiconductor engineers via the browser. By replacing $100K+ per-seat desktop EDA suites with a collaborative SaaS, ChipFlow democratizes chip design for fabless startups, university labs, and hardware teams who cannot afford Cadence Virtuoso or Synopsys ICC2.

---

## Problem Statement

**The pain:**
- Cadence Virtuoso and Synopsys Design Compiler cost $100K–$500K per seat per year, creating an insurmountable barrier for startups, small design houses, and academic institutions trying to tape out custom silicon
- Desktop EDA tools require high-end Linux workstations with 128+ GB RAM and expensive GPU cards, adding $10K–$30K per engineer in hardware costs
- Collaboration on IC design is primitive — engineers email GDS files, maintain per-seat licenses that cannot be shared, and use ad-hoc version control on binary files that produce meaningless diffs
- The semiconductor talent gap is widening: universities cannot afford enough EDA licenses for their VLSI courses, so graduates enter the workforce with limited hands-on experience
- Open-source alternatives (KLayout, Magic VLSI, OpenROAD) require Linux expertise, offer no collaboration, and lack the integrated simulation environment that professional tapeouts demand

**Current workarounds:**
- Companies pay $1M+ annually for a handful of Cadence/Synopsys seats and rotate access among engineers, creating scheduling bottlenecks and compliance risks
- Universities use educational licenses with restricted functionality (no tapeout-ready exports) that leave students unprepared for industry workflows
- Some startups use KLayout for viewing and Magic VLSI for layout, but stitch together 5+ separate tools for a complete design flow with brittle scripting
- Smaller design houses outsource physical design to contractors at $200–$400/hour, adding cost and losing IP control

**Market size:** The global EDA software market is valued at approximately $14.3 billion (2024) and projected to reach $23 billion by 2030. The IC physical design segment specifically represents ~$6 billion. There are an estimated 200,000+ active IC design engineers worldwide, with growing demand driven by AI accelerators, automotive chips, IoT, and RISC-V adoption. The university VLSI education market adds another 50,000+ students annually.

---

## Target Users

### Primary Personas

**1. Sana — Analog IC Designer at a Fabless Startup**
- Works on a 12-person team designing a custom power management IC for wearable devices
- Currently uses Cadence Virtuoso ($150K/seat/year) — the company's EDA licenses are the single largest expense after salaries
- Spends 20% of her time waiting for license servers to free up seats during peak design periods
- Needs: a cost-effective layout editor with SPICE simulation and DRC that doesn't require license management

**2. Prof. Ravi Krishnamurthy — VLSI Course Instructor at a State University**
- Teaches CMOS VLSI design to 80 undergraduate students per semester
- Has 10 Cadence educational licenses shared across 80 students — students queue for lab time and can't work on assignments from home
- Needs: a browser-based tool that all 80 students can access simultaneously from any device, with educational PDK support

**3. Wei — Physical Design Engineer at a Mid-Size Semiconductor Company**
- Works on place-and-route for a RISC-V processor targeting TSMC 28nm
- Uses Synopsys ICC2 for production work but needs to review layouts and run quick DRC checks while traveling or working from home
- Needs: a browser-based viewer/editor that can open GDS-II files for review, run basic DRC, and annotate layouts for team discussions

### Secondary Personas
- RISC-V startups designing custom accelerators who need affordable full-flow EDA from RTL to GDS-II
- PDK developers at foundries who need to verify design rules and test standard cell libraries across tools
- IP vendors who need to share block-level layouts with customers for integration review without exposing full design databases

---

## Solution Overview

ChipFlow is a browser-based IC design platform that:
1. Renders GDS-II and OASIS layout files at interactive framerates using WebGPU-accelerated polygon rendering, supporting zoom from full-chip to transistor-level across designs with billions of polygons
2. Provides a schematic-driven layout editor with GPU-accelerated DRC running continuously during editing, parasitic extraction for post-layout simulation, and LVS verification against the schematic netlist
3. Integrates cloud-based SPICE simulation (ngspice/Xyce) for analog circuit verification and Verilog RTL simulation (Verilator) for digital design validation, with waveform viewing in the browser
4. Enables real-time multi-user collaboration on layout designs with cursor presence, region locking, commenting, and Git-inspired version control with visual layout diffs
5. Supports open PDKs (SkyWater 130nm, GlobalFoundries 180nm, IHP 130nm BiCMOS) out of the box and allows import of custom technology files for proprietary process nodes

---

## Core Features

### F1: GDS-II/OASIS Layout Viewer and Editor
- WebGPU-accelerated polygon rendering supporting GDS-II and OASIS file formats with billions of polygons
- Hierarchical cell browser with instance tree navigation and cell-level statistics (area, density, pin count)
- Per-layer visibility, color, pattern, and transparency controls matching standard EDA color schemes
- Continuous zoom from full-chip overview to individual transistor geometries with level-of-detail rendering
- Layout editing: polygon draw, rectangle, path (trace), via, cell instantiation, array generation
- Parametric cell (PCell) support with configurable parameters (width, length, fingers, metal stack)
- Snap-to-grid with configurable manufacturing grid (typically 5nm for advanced nodes, 50nm for mature nodes)
- Ruler, area measurement, and density analysis tools

### F2: Schematic Editor
- Hierarchical schematic capture with symbol library for MOSFETs, resistors, capacitors, inductors, diodes, and custom IP blocks
- Wire routing with automatic junction placement, bus notation, and net labeling
- Symbol editor for creating custom schematic symbols with pin definitions
- Cross-probing between schematic and layout views — click a component in the schematic to highlight its layout instance
- Netlist extraction to SPICE format for simulation
- ERC: check for floating nodes, shorted supplies, missing substrate connections, and unconnected pins

### F3: GPU-Accelerated Design Rule Checking (DRC)
- GPU-accelerated geometric DRC engine running continuously during layout editing with sub-second feedback
- Rule categories: minimum width, minimum spacing, enclosure, extension, density, antenna, via coverage, metal fill
- Technology-specific rule decks for supported PDKs (SkyWater SKY130, GF180MCU, IHP SG13G2)
- Custom DRC rule scripting using a Python-like DSL for proprietary process rules
- Visual DRC violation overlay with error markers, severity classification, and one-click navigation to violation location
- DRC waiver management: mark known violations as accepted with justification for tapeout review
- Batch DRC: run full-chip DRC on cloud compute for designs too large for interactive checking

### F4: Layout vs. Schematic (LVS) Verification
- Automated extraction of layout netlist from polygon geometries using device recognition rules
- Comparison of extracted layout netlist against schematic netlist with detailed mismatch reporting
- Mismatch categories: missing devices, extra devices, missing nets, shorted nets, parameter mismatches (W/L)
- Cross-probing: click any LVS error to highlight the corresponding schematic symbol and layout geometry
- Support for hierarchical LVS with cell-by-cell verification and flatten-on-demand for complex cells
- Soft-check mode: run LVS during editing for early feedback without waiting for full extraction

### F5: SPICE Simulation
- Cloud-based SPICE simulation using Xyce (Sandia National Labs, open-source, production-grade) with GPU acceleration
- Analysis types: DC operating point, DC sweep, AC analysis (frequency response), transient, noise, Monte Carlo
- Waveform viewer with multi-signal display, cursors, measurements (rise time, slew rate, gain, bandwidth)
- Parametric sweep: vary component values and plot performance metrics across the sweep range
- Corner analysis: simulate across process corners (TT, FF, SS, FS, SF) and temperature ranges
- SPICE model management: built-in models for supported PDKs, import custom models from foundry PDK packages
- Post-layout simulation with extracted parasitics (RC) back-annotated from the layout

### F6: Verilog RTL Simulation
- Cloud-based Verilog/SystemVerilog simulation using Verilator (compiled simulation for speed)
- Integrated code editor with syntax highlighting, autocompletion, and linting for Verilog/SystemVerilog
- Waveform viewer supporting VCD and FST formats with hierarchical signal browsing
- Testbench templates for common verification patterns (clock generation, reset sequencing, stimulus replay)
- Assertion support: SystemVerilog assertions (SVA) with pass/fail reporting
- Code coverage: statement, branch, toggle, and FSM coverage with uncovered code highlighting
- Linting with common RTL coding guideline checks (clock domain crossing, latch inference, X-propagation)

### F7: Parasitic Extraction (PEX)
- RC parasitic extraction from layout geometry using field-solver-calibrated algorithms
- Extraction modes: coupled RC (detailed), lumped RC (fast), resistance-only (fastest)
- Output formats: SPEF (Standard Parasitic Exchange Format), DSPF, extracted SPICE netlist
- Visualization of parasitic contributions: color-coded resistance/capacitance overlay on layout
- Back-annotation to SPICE simulation for post-layout verification
- Extraction rules calibrated for supported PDKs with foundry-validated accuracy

### F8: Real-Time Collaboration
- CRDT-based document model for conflict-free simultaneous layout editing by multiple users
- Cursor presence showing collaborator positions and active editing regions in real-time
- Region locking: claim a layout area to prevent conflicting edits during detailed work
- In-context commenting: attach comments to specific cells, layers, or geometric regions
- Review mode: share read-only links for design review with annotation capability
- Integrated voice/video chat for design review sessions

### F9: Version Control
- Git-inspired branching for layout design: create branches for experimental layout changes
- Visual layout diff: overlay two revisions with color-coded additions (green), deletions (red), and modifications (blue)
- Cell-level change tracking: see which cells were modified between any two revisions
- Tagged releases for tapeout milestones (e.g., "DRC clean", "LVS clean", "tapeout candidate")
- Complete revision history with restore capability for any previous state
- Merge operations with geometric conflict detection for overlapping layout edits

### F10: PDK and Technology Management
- Built-in open PDKs: SkyWater SKY130 (130nm CMOS), GlobalFoundries GF180MCU (180nm), IHP SG13G2 (130nm BiCMOS)
- Technology file import: layer definitions, design rules, device parameters, PCell definitions
- Standard cell library browser with cell-level schematics, layouts, timing data, and characterization results
- IP block catalog: padring generators, memory compilers, analog IP blocks for supported PDKs
- PDK documentation viewer: process specifications, design manuals, and SPICE model documentation inline

### F11: Tapeout Export and Fabrication
- GDS-II and OASIS export with configurable layer mapping and hierarchy flattening options
- Metal fill generation: automatic dummy metal insertion for density compliance
- Seal ring and pad frame generation from configurable templates
- Chip assembly: place multiple design blocks on a shared reticle for multi-project wafer (MPW) runs
- Direct submission integration with open-access shuttles: Efabless (Google-sponsored SKY130), Europractice
- Pre-tapeout checklist with automated DRC, LVS, antenna, and density verification

### F12: Import and Migration
- Import GDS-II and OASIS files from any EDA tool with full hierarchy preservation
- Import Virtuoso library databases via OA-to-GDS conversion
- Import LEF/DEF for standard cell place-and-route designs
- Import Verilog netlists for logic-to-layout cross-referencing
- SPICE netlist import for simulation setup
- Automatic technology file mapping between imported designs and ChipFlow PDKs

---

## Technical Architecture

### System Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        Browser Client                            │
│  ┌────────────┐  ┌────────────┐  ┌───────────┐  ┌────────────┐ │
│  │   Layout    │  │ Schematic  │  │ Waveform  │  │  RTL Code  │ │
│  │   Editor    │  │  Editor    │  │  Viewer   │  │  Editor    │ │
│  │  (WebGPU)   │  │ (Canvas)   │  │ (Canvas)  │  │ (Monaco)   │ │
│  └──────┬──────┘  └──────┬─────┘  └─────┬─────┘  └─────┬──────┘ │
│         │               │              │              │         │
│         └───────────────┼──────────────┼──────────────┘         │
│                         ▼                                       │
│               CRDT Document Model (Yjs)                          │
│                         │                                       │
└─────────────────────────┼───────────────────────────────────────┘
                          │ WebSocket
                          ▼
                ┌─────────────────────┐
                │   API Gateway       │
                │   (Rust / Axum)     │
                └────┬───────┬────────┘
                     │       │
          ┌──────────┘       └──────────┐
          ▼                             ▼
┌─────────────────┐           ┌─────────────────────┐
│  Collaboration   │           │   Design Engine      │
│  Server (Yjs)    │           │   (Rust + WASM)      │
│  WebSocket Hub   │           │                      │
└────────┬────────┘           │  ┌────────────────┐  │
         │                    │  │ DRC Engine      │  │
         ▼                    │  │ (GPU Compute)   │  │
┌─────────────────┐           │  └────────────────┘  │
│   PostgreSQL    │           │  ┌────────────────┐  │
│  (Users, Proj,  │           │  │ LVS Engine     │  │
│   Revisions,    │◄──────────│  │ (Rust)         │  │
│   PDK metadata) │           │  └────────────────┘  │
└────────┬────────┘           │  ┌────────────────┐  │
         │                    │  │ PEX Engine     │  │
         ▼                    │  │ (Rust + GPU)   │  │
┌─────────────────┐           │  └────────────────┘  │
│     Redis       │           └─────────────────────┘
│  (Sessions,     │                      │
│   Cache,        │           ┌──────────┴──────────┐
│   Pub/Sub)      │           ▼                     ▼
└─────────────────┘  ┌──────────────┐    ┌───────────────┐
                     │ SPICE Workers │    │ Verilog Sim   │
                     │ (Xyce/ngspice)│    │ (Verilator)   │
                     │ GPU-accel     │    │ Cloud compile  │
                     └──────────────┘    └───────────────┘
                              │
                              ▼
                     ┌─────────────────┐
                     │  Object Storage │
                     │  (S3/R2)        │
                     │  GDS-II, OASIS, │
                     │  Waveforms, PDK │
                     └─────────────────┘
```

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend Framework | React 19 + TypeScript, Vite build system |
| Layout Renderer | WebGPU with custom polygon rendering pipeline, level-of-detail culling |
| Schematic Canvas | Custom Canvas2D renderer with spatial indexing (R-tree) |
| Waveform Viewer | Canvas2D with WebWorker-based VCD/FST parsing |
| RTL Editor | Monaco Editor with custom Verilog/SystemVerilog language server |
| Collaboration | Yjs CRDT library with WebSocket provider |
| Backend API | Rust (Axum framework) for low-latency design operations |
| DRC Engine | Rust with GPU compute shaders (wgpu) for geometric boolean operations |
| LVS Engine | Rust-based netlist extraction and comparison |
| PEX Engine | Rust field-solver with GPU-accelerated capacitance extraction |
| SPICE Simulation | Xyce (C++, Sandia) and ngspice deployed on GPU-accelerated cloud instances |
| RTL Simulation | Verilator (C++ compiled simulation) on cloud compute instances |
| Database | PostgreSQL 16 with PostGIS for spatial queries on layout geometry |
| Cache / Pub-Sub | Redis 7 for session management, real-time presence, and job status |
| Object Storage | Cloudflare R2 for GDS-II files, simulation results, and PDK libraries |
| Auth | JWT-based auth with OAuth (Google, GitHub), SAML for enterprise, institutional SSO for universities |
| Search | Meilisearch for cell library, PDK component, and IP block search |
| Hosting | Fly.io (API), Cloudflare Workers (edge), GPU instances on Lambda Cloud |
| Monitoring | Grafana + Prometheus, Sentry for errors, PostHog for analytics |

### Data Model

```
User
├── id (uuid), email, name, avatar_url
├── auth_provider, auth_provider_id
├── plan (free/academic/pro/team/enterprise)
├── institution (nullable — university name)
└── created_at, updated_at

Organization
├── id (uuid), name, slug
├── owner_id → User
├── plan, seat_count, billing_email
└── settings (default_pdk, grid_units, color_scheme)

Project
├── id (uuid), org_id → Organization
├── name, description, visibility (private/org/public)
├── pdk_id → PDK, process_node (nm)
├── created_by → User
├── forked_from → Project (nullable)
└── created_at, updated_at

PDK
├── id (uuid), name (e.g., "SKY130"), foundry, process_node_nm
├── layer_definitions (JSONB — layer number, name, purpose, color, pattern)
├── design_rules (JSONB — min_width, min_spacing, enclosure per layer pair)
├── device_models_url (S3 — SPICE model files)
├── standard_cells_url (S3 — GDS + Liberty timing)
├── documentation_url
└── is_open (boolean), created_at

Cell
├── id (uuid), project_id → Project
├── name, cell_type (layout/schematic/symbol)
├── hierarchy_level, parent_cell_id → Cell (nullable)
├── geometry_data_url (S3 — serialized layout/schematic data)
├── bounding_box (x_min, y_min, x_max, y_max)
├── pin_count, instance_count, area_um2
└── created_by → User, created_at, updated_at

Branch
├── id (uuid), project_id → Project
├── name, base_revision_id → Revision
├── head_revision_id → Revision
└── created_by → User, created_at

Revision
├── id (uuid), project_id → Project, branch_id → Branch
├── parent_revision_id → Revision (nullable)
├── snapshot_url (S3 — serialized CRDT state)
├── message, tag (nullable)
├── drc_status (pending/clean/violations), drc_violation_count
├── lvs_status (pending/pass/fail)
└── created_by → User, created_at

SimulationJob
├── id (uuid), project_id → Project
├── job_type (spice_dc/spice_ac/spice_tran/verilog/drc_batch/pex)
├── status (queued/running/completed/failed)
├── input_config (JSONB — netlist, analysis params)
├── result_url (S3 — waveform data, reports)
├── compute_time_seconds, cost_usd
└── created_by → User, started_at, completed_at

DRCViolation
├── id (uuid), project_id → Project, revision_id → Revision
├── rule_name, severity (error/warning), category
├── description, location (JSONB — x, y, layer)
├── affected_cells[], waived (boolean), waiver_reason
└── created_at

Comment
├── id (uuid), project_id → Project
├── author_id → User, body (text)
├── anchor_type (cell/layer/region/general)
├── anchor_data (JSONB — cell_name, coordinates, layer)
├── resolved (boolean)
└── created_at
```

### API Design

```
Auth:
POST   /api/auth/register                  # Email/password registration
POST   /api/auth/login                     # Login
POST   /api/auth/oauth/:provider           # OAuth flow (Google, GitHub)
POST   /api/auth/saml                      # SAML/SSO for enterprise and universities
POST   /api/auth/refresh                   # Refresh JWT token

Organizations:
GET    /api/orgs                           # List organizations
POST   /api/orgs                           # Create organization
PATCH  /api/orgs/:id                       # Update settings
POST   /api/orgs/:id/members               # Invite member
DELETE /api/orgs/:id/members/:user_id      # Remove member

Projects:
GET    /api/projects                       # List projects
POST   /api/projects                       # Create project
GET    /api/projects/:id                   # Get project metadata
PATCH  /api/projects/:id                   # Update project settings
DELETE /api/projects/:id                   # Delete project
POST   /api/projects/:id/fork              # Fork project
POST   /api/projects/:id/export/gds        # Export GDS-II
POST   /api/projects/:id/export/oasis      # Export OASIS

Cells:
GET    /api/projects/:id/cells              # List cells (hierarchy)
POST   /api/projects/:id/cells              # Create cell
GET    /api/cells/:id                       # Get cell data
PATCH  /api/cells/:id                       # Update cell
DELETE /api/cells/:id                       # Delete cell
GET    /api/cells/:id/geometry              # Get layout/schematic geometry

Branches & Versions:
GET    /api/projects/:id/branches                   # List branches
POST   /api/projects/:id/branches                   # Create branch
POST   /api/projects/:id/branches/:name/merge       # Merge branch
GET    /api/projects/:id/revisions                  # List revisions
POST   /api/projects/:id/revisions                  # Create revision
GET    /api/projects/:id/revisions/:rev_id          # Get revision snapshot
GET    /api/projects/:id/diff/:rev_a/:rev_b         # Layout diff

Design Verification:
POST   /api/projects/:id/drc                        # Run DRC
GET    /api/projects/:id/drc/results                 # Get DRC violations
POST   /api/projects/:id/lvs                        # Run LVS
GET    /api/projects/:id/lvs/results                 # Get LVS results
POST   /api/projects/:id/pex                        # Run parasitic extraction
GET    /api/projects/:id/pex/results                 # Get PEX results

Simulation:
POST   /api/projects/:id/simulate/spice             # Start SPICE simulation
POST   /api/projects/:id/simulate/verilog           # Start Verilog simulation
GET    /api/simulations/:id                         # Get simulation status
GET    /api/simulations/:id/waveforms               # Get waveform data
POST   /api/simulations/:id/cancel                  # Cancel simulation

PDK:
GET    /api/pdks                                     # List available PDKs
GET    /api/pdks/:id                                 # Get PDK details
GET    /api/pdks/:id/layers                          # Get layer definitions
GET    /api/pdks/:id/cells                           # Browse standard cell library
POST   /api/pdks                                     # Upload custom PDK (enterprise)

Collaboration:
WS     /ws/projects/:id                              # WebSocket for real-time collaboration
GET    /api/projects/:id/comments                    # List comments
POST   /api/projects/:id/comments                    # Add comment
PATCH  /api/comments/:id                             # Edit/resolve comment
POST   /api/projects/:id/share                       # Generate share link

Fabrication:
POST   /api/projects/:id/tapeout/checklist           # Run pre-tapeout checks
POST   /api/projects/:id/tapeout/submit              # Submit to shuttle
GET    /api/tapeout-orders                            # List tapeout orders
GET    /api/tapeout-orders/:id/status                 # Track order status
```

---

## UI/UX — Key Screens

### 1. Project Dashboard
- Card grid of all projects with chip layout thumbnails, PDK badges, and DRC/LVS status indicators
- Quick-create wizard: select PDK, design type (analog, digital, mixed-signal), and start from blank or template
- Recent activity feed: edits, DRC runs, simulations completed, comments added
- Team members list with online status and current editing location

### 2. Layout Editor
- Full-screen WebGPU canvas with layer color legend and visibility toggles in a collapsible left panel
- Cell hierarchy browser on the left with expandable instance tree and cell statistics
- Right panel: selected object properties (coordinates, layer, dimensions), DRC violation list, net inspector
- Bottom toolbar: draw mode (polygon, rectangle, path, via), edit mode (move, stretch, copy, array)
- Minimap showing full-chip overview with viewport rectangle indicator
- Measurement tools with automatic DRC distance readout for selected edges

### 3. Schematic Editor
- Center canvas with symbol palette on the left organized by category (MOSFET, passive, digital, IP blocks)
- Property panel on the right showing selected component parameters (W/L, model, value)
- Top toolbar: wire, bus, net label, power, ground, pin, port, text annotation
- Bottom status bar: total components, nets, ERC error count, netlist extraction status
- Cross-probe button: click to jump from schematic component to its layout instance

### 4. Waveform Viewer
- Multi-pane waveform display with independent Y-axis scaling per signal group
- Signal browser with hierarchy-aware search (module.submodule.signal_name)
- Cursor tools: primary/secondary cursors with delta measurement display
- Waveform math: arithmetic operations between signals, FFT transform, eye diagram
- Zoom controls: horizontal time zoom, vertical voltage zoom, fit-to-screen
- Marker annotations: label specific time points for documentation

### 5. DRC Results Dashboard
- Violation summary: total count by rule category and severity with trend chart across revisions
- Interactive violation browser: click any violation to pan the layout to the error location
- Violation filtering by rule, severity, layer, and cell with bulk waiver capability
- Progress tracker: DRC clean percentage with per-rule pass/fail breakdown
- Historical comparison: DRC count trend across revisions showing convergence toward clean design

### 6. Pre-Tapeout Checklist
- Step-by-step tapeout readiness checklist: DRC clean, LVS clean, ERC clean, antenna check, density check, seal ring present
- Green/red status indicators for each check with one-click re-run capability
- Shuttle submission form: select foundry shuttle (Efabless/Europractice), enter die dimensions, confirm layer mapping
- Cost estimate based on die area and shuttle pricing
- Export final GDS-II with automated metal fill and seal ring insertion

---

## Monetization

### Free Tier (Academic)
- Requires .edu or institutional email verification
- Unlimited projects using open PDKs (SKY130, GF180MCU)
- Interactive DRC and LVS for designs up to 10,000 instances
- SPICE simulation: 50 runs per month
- Verilog simulation: 50 runs per month
- Community support only

### Pro — $199/month
- Unlimited projects and PDKs (including custom PDK import)
- No instance count limits for DRC and LVS
- SPICE simulation: 500 runs per month with GPU acceleration
- Verilog simulation: 500 runs per month
- Parasitic extraction (PEX) included
- GDS-II and OASIS export without restrictions
- Batch DRC on cloud compute
- Email support with 48-hour response time

### Team — $499/month (up to 5 seats, $99/additional seat)
- Everything in Pro
- Real-time multi-user collaboration
- Branch-based version control with visual layout diff
- Commenting and review workflows
- Shared cell libraries across team
- Organization-level project management and admin controls
- Priority support with 24-hour response time
- 2,000 simulation runs per month

### Enterprise — Custom
- Unlimited seats and simulations
- Custom PDK hosting with NDA-protected process data
- On-premise deployment option for semiconductor IP security
- SAML/SSO integration and audit logging
- Dedicated GPU compute cluster for DRC and simulation
- Foundry shuttle submission integration
- Dedicated customer success manager and SLA guarantee
- Integration with existing PLM and design management systems (Teamcenter, Windchill)

---

## Go-to-Market Strategy

### Phase 1: Academic Seeding (Month 1-3)
- Launch free academic tier and announce on VLSI education mailing lists, IEEE CEDA, and ACM SIGDA communities
- Partner with 10 university VLSI courses for pilot deployment with SkyWater 130nm PDK
- Publish tutorial series: "Design Your First Chip in the Browser" targeting students and hobbyists
- Submit workshop proposal to VLSI Design Conference and DAC (Design Automation Conference)
- Launch integration with Efabless open-source tapeout program

### Phase 2: Professional Adoption (Month 3-6)
- Launch Pro tier with custom PDK support and GPU-accelerated simulation
- Content marketing: "Cadence alternative for startups", "browser-based IC design", "open-source EDA SaaS"
- Publish benchmark comparisons: DRC speed vs. Calibre, SPICE accuracy vs. Spectre
- Partner with RISC-V startups and silicon-proven IP vendors for reference designs in ChipFlow
- Exhibit at DAC, ISSCC, and Hot Chips with live tapeout demos

### Phase 3: Enterprise and Foundry (Month 6-12)
- Launch Team and Enterprise tiers with collaboration and custom PDK hosting
- Partner with foundries to offer ChipFlow as a supported design entry point for their processes
- Pursue relationships with semiconductor incubators and CHIPS Act funding recipients
- Launch ChipFlow Marketplace for community-contributed standard cells, IP blocks, and reference designs
- Build integrations with EDA ecosystem: LEF/DEF import from OpenROAD, timing analysis with OpenSTA

### Acquisition Channels
- Organic search: "free IC design software", "browser-based EDA", "Cadence alternative", "VLSI design tool for students"
- University partnerships driving graduates into professional tiers as they enter industry
- RISC-V and open-source hardware community engagement (FOSSi, ChipsAlliance)
- Conference presentations and workshops at DAC, ISSCC, VLSI Design Conference
- Efabless and Google open-source silicon program integration

---

## Success Metrics & KPIs

| Metric | Target (Month 6) | Target (Month 12) |
|--------|-------------------|---------------------|
| Registered users | 5,000 | 25,000 |
| Monthly active projects | 1,500 | 8,000 |
| Paying customers (Pro + Team) | 80 | 400 |
| MRR | $25,000 | $120,000 |
| DRC runs per month | 10,000 | 60,000 |
| SPICE simulations per month | 5,000 | 30,000 |
| Free → Paid conversion rate | 3% | 6% |
| Monthly churn rate | <5% | <3% |

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | ChipFlow Advantage |
|-----------|-----------|------------|-------------------|
| Cadence Virtuoso | Industry gold standard for analog/mixed-signal, deepest foundry support, Spectre simulator | $100K+/seat, Linux-only, no collaboration, massive learning curve | 100x cheaper, browser-based, real-time collaboration, accessible to students |
| Synopsys ICC2 | Best-in-class digital P&R, signoff-quality DRC (Calibre), deep integration with synthesis | $200K+/seat, complex licensing, desktop-only, no analog flow | Unified analog+digital flow, simple pricing, browser-based, open PDK support |
| KLayout | Free/open-source, excellent GDS viewer, scripting support, active community | No schematic editor, no simulation, no collaboration, steep learning curve | Full design flow (schematic + layout + sim + DRC + LVS), collaboration built-in |
| Magic VLSI | Free, open-source, integrated DRC and extraction, used in academic tapeouts | Outdated UI, Linux-only, limited to mature process nodes, single-user | Modern browser UI, multi-user collaboration, GPU-accelerated DRC |
| OpenROAD | Open-source RTL-to-GDS flow, strong digital P&R, DARPA-funded | Digital-only (no analog), command-line driven, no visual layout editor | Visual editor for analog and custom digital, integrated simulation, collaboration |

---

## Risks & Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| WebGPU not yet universally supported across browsers | Medium | Medium | Implement WebGL 2.0 fallback renderer, target Chrome/Edge first (highest WebGPU adoption), provide progressive enhancement |
| GPU-accelerated DRC accuracy insufficient for tapeout signoff | High | Medium | Calibrate against Calibre golden results, offer Calibre-compatible rule deck format, position as design-time DRC with recommendation for final signoff in Calibre |
| Semiconductor IP security concerns deter enterprise adoption | High | High | Implement end-to-end encryption, offer on-premise deployment, obtain SOC 2 Type II and ISO 27001 certifications, provide air-gapped deployment option |
| Foundries refuse to support browser-based tool in their PDK ecosystem | Medium | Medium | Start with open PDKs (SKY130, GF180), build tapeout track record, demonstrate value to foundries through increased customer pipeline |
| Custom PDK import complexity creates onboarding friction | Medium | High | Build PDK conversion wizards for common formats, offer white-glove onboarding service for Enterprise customers, document PDK porting guides |

---

## MVP Scope (v1.0)

### In Scope
- GDS-II layout viewer and editor with WebGPU rendering (polygon draw, path, via, cell instantiation)
- SkyWater SKY130 PDK with layer definitions, design rules, and basic standard cells
- GPU-accelerated interactive DRC with 10 core rules (width, spacing, enclosure, density)
- Basic schematic editor with MOSFET and passive symbols, wire routing, netlist extraction
- LVS verification (schematic vs. extracted layout netlist comparison)
- SPICE simulation (DC, AC, transient) with cloud-based Xyce and waveform viewer
- User accounts, project management, and basic revision history
- GDS-II export

### Out of Scope (v1.1+)
- Real-time multi-user collaboration (v1.1 — highest priority post-MVP)
- Verilog RTL simulation and code editor (v1.1)
- Parasitic extraction (PEX) and post-layout simulation (v1.2)
- Branch-based version control with visual layout diff (v1.2)
- Custom PDK import and additional open PDKs (GF180MCU, IHP) (v1.2)
- OASIS file support (v1.3)
- Batch DRC on cloud compute (v1.3)
- Metal fill generation and tapeout export pipeline (v1.3)
- Enterprise features: SSO, on-premise, audit logging (v2.0)
- Foundry shuttle submission integration (v2.0)

### MVP Timeline: 10 weeks
- Week 1-2: Auth system, project/org data model, WebGPU rendering pipeline for polygon display, GDS-II parser (Rust compiled to WASM)
- Week 3-4: Layout editing tools (polygon, rectangle, path, via, cell instantiation), SKY130 PDK integration (layers, colors, design rules)
- Week 5-6: DRC engine (GPU-accelerated width/spacing checks), schematic editor (symbol placement, wiring, netlist extraction)
- Week 7-8: LVS engine (layout extraction + comparison), SPICE simulation pipeline (Xyce on cloud), waveform viewer
- Week 9-10: GDS-II export, project dashboard, revision history, user management, billing (Stripe), documentation, load testing, and launch
