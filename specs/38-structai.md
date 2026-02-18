# StructAI — AI Structural Engineering Analysis & Design Platform

## Executive Summary

StructAI is a cloud-native structural engineering platform that combines GPU-accelerated finite element analysis, AI-powered design optimization, and real-time 3D visualization to replace $5K+/seat desktop software with a browser-based workflow accessible to any structural engineer. By integrating building code compliance checking (IBC, ASCE 7, Eurocode, AS/NZS), BIM import from Revit and Tekla, and intelligent member sizing that minimizes structural weight while satisfying all code requirements, StructAI delivers SAP2000-class analysis capability at a fraction of the cost with collaboration and AI built in from day one.

---

## Problem Statement

**The pain:**
- Professional structural analysis software (SAP2000, ETABS, STAAD.Pro) costs $5,000–$15,000 per seat per year with mandatory maintenance fees, putting advanced analysis out of reach for small firms, sole practitioners, and engineers in developing countries
- Desktop FEA solvers are CPU-bound and single-threaded for many operations — a nonlinear analysis of a 20-story building can take 8–24 hours on a workstation, blocking engineers from iterating on designs
- Building code compliance checking is largely manual: engineers run analysis in one tool, then hand-check member capacities in spreadsheets or separate design software, creating a fragmented and error-prone workflow
- BIM-to-analysis model transfer is painful — importing a Revit structural model into SAP2000 or ETABS requires hours of cleanup, re-meshing, and manual load assignment, and round-tripping changes back to BIM is even worse
- Design optimization is trial-and-error: engineers manually resize members, re-run analysis, and re-check codes over multiple cycles, wasting days on work that could be automated

**Current workarounds:**
- Firms purchase multiple seats of SAP2000/ETABS ($50K+ annual spend) and schedule access among engineers, creating project bottlenecks and license contention
- Engineers use free or low-cost tools (OpenSees, SkyCiv) that lack advanced nonlinear capabilities, seismic analysis features, or building code integration required for real projects
- Manual spreadsheet-based design checks using Excel templates with hand-coded formulas that are brittle, unauditable, and disconnected from the analysis model
- Outsourcing complex analysis (seismic, progressive collapse, nonlinear time-history) to specialist consultants at $10K–$50K per study with multi-week turnaround

**Market size:** The global structural engineering software market was valued at $7.2B in 2024 and is projected to reach $12.8B by 2030. There are approximately 800,000 structural engineers worldwide who use or need FEA software regularly, plus an additional 1.5M+ civil engineering students who are trained on structural analysis tools annually. The building construction sector alone accounts for $13T in global annual spending, all of which requires structural engineering.

---

## Target Users

### Primary Personas

**1. Maria Gutierrez — Senior Structural Engineer at a Mid-Size Design Firm**
- Works at a 40-person structural engineering firm designing mid-rise commercial and residential buildings
- Firm owns 6 ETABS licenses shared among 18 engineers — she often waits days for license availability during deadlines
- Spends 30% of her time manually checking steel beam and column designs in spreadsheets after running ETABS analysis
- Needs: integrated analysis-and-design in one tool, faster solver for design iteration, automatic code compliance checking so she can focus on engineering judgment rather than hand calculations

**2. Dr. James Okonkwo — Structural Engineering Professor and Consultant**
- Teaches graduate-level structural dynamics and earthquake engineering at a research university
- Runs a small consulting practice on the side, specializing in seismic retrofit assessments for existing buildings
- Uses SAP2000 for teaching but students cannot afford personal licenses; uses OpenSees for research but it has no GUI
- Needs: affordable browser-based tool that students can access from any computer, with response spectrum analysis, pushover analysis, and clear 3D visualization of mode shapes and deformed structures

**3. Priya Sharma — Sole Practitioner Structural Engineer**
- Runs a one-person structural engineering consultancy designing residential and light commercial structures
- Cannot justify $10K/year for ETABS; currently uses RISA-3D ($2K/year) but finds it limited for complex projects
- Loses bids on larger projects because clients require advanced seismic analysis and detailed code compliance reports that her current tools cannot produce
- Needs: professional-grade structural analysis at an affordable price point, automated building code reports that satisfy plan reviewers, and BIM integration so she can collaborate with architects using Revit

### Secondary Personas
- Construction firms that need to verify structural designs and review analysis reports without owning engineering software
- Building officials and plan reviewers who want interactive 3D visualization of structural behavior to understand submitted designs
- Graduate students and researchers performing parametric studies on structural systems who need scriptable, GPU-accelerated analysis
- Architects who need preliminary structural feasibility checks before engaging a structural engineer

---

## Solution Overview

StructAI is a browser-based structural engineering platform that:
1. Imports structural models from BIM tools (Revit via IFC, Tekla Structures, ETABS) or builds them from scratch with an intuitive 3D structural modeler supporting beams, columns, walls, slabs, and foundations with parametric cross-sections
2. Runs GPU-accelerated finite element analysis — linear static, P-delta, buckling, modal, response spectrum, equivalent lateral force, and nonlinear time-history — completing in minutes what desktop solvers take hours to compute
3. Automatically checks every member against building codes (AISC 360 for steel, ACI 318 for concrete, IBC 2021, ASCE 7-22, Eurocode 2/3/8, AS/NZS) and generates demand-to-capacity ratios with detailed clause references
4. Provides interactive 3D visualization of deformed shapes, stress contours, mode shapes, and force diagrams directly in the browser using Three.js, with intuitive controls for exploring structural behavior
5. Runs AI-powered design optimization that automatically sizes members to minimize total structural weight (or cost) while satisfying all strength, serviceability, and code requirements — turning days of manual iteration into minutes of automated computation

---

## Core Features

### F1: 3D Structural Modeling
- Grid-based structural layout with customizable grid spacing in X, Y, and Z directions for rapid building model creation
- Member types: beams, columns, braces, trusses, cables, and springs with parametric assignment of cross-sections (W-shapes, HSS, angles, channels, custom)
- Area elements: walls (shear walls, retaining walls), slabs (one-way, two-way, flat plate, waffle), and shell elements with configurable thickness and material
- Support conditions: fixed, pinned, roller, spring supports with translational and rotational stiffness in all six degrees of freedom
- Rigid and semi-rigid diaphragm assignment for floor systems with automatic tributary area calculation
- Node merge and element subdivision tools for mesh refinement in critical regions
- Copy, mirror, and replicate operations for rapid model assembly of repetitive structures

### F2: GPU-Accelerated FEA Solver
- Linear static analysis with direct and iterative solvers, GPU-parallelized sparse matrix operations completing 100K DOF models in under 60 seconds
- P-delta analysis (geometric nonlinearity) for capturing second-order effects in slender structures and tall buildings
- Buckling analysis: linearized eigenvalue buckling to determine critical load factors and buckling mode shapes
- Modal analysis: natural frequencies and mode shapes using Lanczos or subspace iteration eigensolvers, GPU-accelerated for models with 500+ modes
- Material nonlinearity: fiber-based beam-column elements, concrete cracking and crushing, steel plasticity with isotropic and kinematic hardening
- Progressive loading with automatic load stepping, convergence monitoring, and adaptive step size for nonlinear analysis
- Batch analysis: queue multiple load cases and combinations for sequential or parallel execution on cloud GPU clusters

### F3: Building Code Compliance
- Steel design per AISC 360-22: flexure, shear, axial, combined forces (H1/H2 interaction), slenderness checks, lateral-torsional buckling, local buckling classification
- Concrete design per ACI 318-19: flexural reinforcement, shear reinforcement, axial-flexure interaction (P-M diagrams), development lengths, minimum reinforcement ratios
- Eurocode 3 (steel) and Eurocode 2 (concrete) design checks with National Annex parameters configurable per country
- AS/NZS 4100 (steel) and AS 3600 (concrete) for Australian and New Zealand projects
- Seismic provisions: ASCE 7-22 seismic load determination, R-factors, redundancy factors, drift limits, overstrength combinations
- Wind load generation per ASCE 7-22 (analytical and wind tunnel procedures), Eurocode 1 Part 4, and AS/NZS 1170.2
- Demand-to-capacity ratios displayed on every member with traffic-light color coding (green < 0.7, yellow 0.7–0.9, red > 0.9) and detailed clause-by-clause breakdown

### F4: BIM Import and Integration
- IFC 2x3 and IFC 4 import with automatic recognition of structural elements (IfcBeam, IfcColumn, IfcWall, IfcSlab, IfcFooting)
- Revit structural model import via IFC export with mapping of Revit families to StructAI cross-sections and materials
- Tekla Structures integration via IFC and Tekla Open API for bi-directional model synchronization
- Automatic extraction of material properties, cross-section dimensions, and support conditions from BIM metadata
- Geometry cleanup: merge coincident nodes, split intersecting elements, remove duplicate members, and resolve connectivity issues
- BIM change detection: compare imported model versions and highlight added, removed, and modified elements
- Export analysis results back to IFC for visualization in BIM authoring tools and coordination with other disciplines

### F5: AI Design Optimization
- Objective functions: minimize total structural weight, minimize material cost, minimize embodied carbon, or multi-objective Pareto optimization
- Design variables: member cross-section sizes selected from standard shape databases (AISC, British Steel, European sections)
- Constraints: all applicable building code checks (strength, stability, serviceability), user-defined drift limits, and vibration criteria
- Gradient-free optimization using genetic algorithms and particle swarm optimization, parallelized across GPU for populations of 500+ candidate designs
- Topology optimization for bracing layouts and lateral system configuration — AI suggests optimal brace locations and types
- Sensitivity analysis: ranked report showing which members are most over-designed and which are controlling the design
- Optimization history dashboard showing weight reduction progression and constraint satisfaction across generations

### F6: Steel Design Module
- Full AISC 360-22 compliance: Chapter E (compression), Chapter F (flexure), Chapter G (shear), Chapter H (combined forces), Chapter I (composite), Chapter J (connections)
- Automatic section classification: compact, noncompact, slender for flanges and webs with appropriate capacity reductions
- Lateral-torsional buckling with unbraced length detection from model geometry and user-specified bracing points
- Composite beam design: effective slab width, partial composite action, stud requirements per AISC 360 Chapter I
- Demand-to-capacity ratio envelopes across all load combinations with governing combination identification
- Section optimization: automatically suggest lightest W-shape, HSS, or angle that satisfies all checks for each member
- Steel quantity takeoff: total tonnage by section size, grade, and member type with material cost estimation

### F7: Concrete Design Module
- ACI 318-19 flexural design: required reinforcement area, bar selection, spacing, cover requirements, minimum and maximum reinforcement ratios
- Shear design: stirrup spacing and size per ACI 318 Chapter 22, including torsion design for spandrel beams
- Column design: P-M interaction diagrams for uniaxial and biaxial bending, slenderness effects per ACI 318 Chapter 6
- Slab design: direct design method and equivalent frame method for two-way slabs, punching shear checks at columns
- Shear wall design: in-plane shear, flexure, and boundary element requirements per ACI 318 Chapter 18
- Reinforcement detailing: automatic generation of rebar schedules with bar marks, cut lengths, bend shapes, and lap splice locations
- Concrete quantity takeoff: volume by element type, reinforcement weight, and formwork area

### F8: Connection Design
- Shear connections: single-plate (shear tab), double-angle, and end-plate connections per AISC 360 Chapter J
- Moment connections: bolted and welded flange plate, extended end-plate connections per AISC 358
- Brace connections: gusset plate design for concentric and eccentric braced frames, Whitmore section, block shear, buckling checks
- Base plate design: bearing, anchor bolt tension and shear, base plate bending per AISC Design Guide 1
- Bolt group and weld group analysis: instantaneous center of rotation method, fillet and groove weld design with directional strength
- Connection design report: detailed calculation sheets with sketches, dimensions, bolt/weld patterns, and demand-to-capacity ratios

### F9: Seismic and Dynamic Analysis
- Equivalent lateral force (ELF) procedure per ASCE 7-22 Chapter 12 with automatic base shear calculation and vertical distribution
- Response spectrum analysis: CQC and SRSS modal combination, design spectrum input from ASCE 7 site-specific or code-defined spectra (Ss, S1, site class)
- Linear and nonlinear time-history analysis with ground motion record input (PEER NGA database integration), Newmark-beta and HHT-alpha integration
- Pushover analysis: displacement-controlled monotonic loading with hinge formation tracking and capacity curve generation
- Story drift checks per ASCE 7 Table 12.12-1 with graphical drift profile diagrams at each story level
- Modal participation mass tracking with automatic determination of sufficient modes for 90% mass participation
- Seismic weight calculation and mass source definition with automatic inclusion of dead load and applicable live load portions

### F10: Load Generation and Combinations
- Dead load: automatic self-weight calculation from member sizes and material densities, plus user-defined superimposed dead loads
- Live load: assignment by occupancy type per ASCE 7 Table 4.3-1 (office, residential, storage, roof) with live load reduction per Section 4.7
- Wind load: automatic generation per ASCE 7-22 Chapter 26-31 based on wind speed, exposure category, building dimensions, and enclosure classification
- Snow load: ground snow load, flat roof snow load, drift loads, sliding loads, and unbalanced loads per ASCE 7 Chapter 7
- Load combination generation: automatic creation of all required combinations per ASCE 7 Section 2.3 (LRFD) and 2.4 (ASD) including seismic and wind combinations
- Pattern loading: automatic generation of checkerboard and strip live load patterns for continuous beams and slabs
- User-defined loads: point, line, area, and temperature loads with graphical application on the 3D model

### F11: Collaboration and Sharing
- Project-based workspaces with role-based access control (owner, engineer, reviewer, viewer)
- Real-time model viewing: multiple engineers can view and navigate the same 3D model simultaneously with cursor presence
- Design review workflow: submit analysis for review, reviewer adds comments anchored to specific members or results, approve or request changes
- Share analysis results via public link (read-only 3D viewer with deformed shapes and DCR display, no account required)
- Activity log showing all model changes, analysis runs, and design decisions with timestamps and author attribution
- Template library: save and share common structural configurations (moment frame, braced frame, shear wall) as reusable starting points

### F12: Reporting and Drawing Generation
- One-click PDF report generation: executive summary, model description, loading, analysis results, code compliance checks, and member design summaries
- Structural calculation package: detailed member-by-member design calculations with clause references suitable for building permit submission
- Graphical output sheets: plan views with member sizes, elevation views with loading, 3D perspective views with DCR color coding
- Automatic generation of framing plans showing beam sizes, column schedules, and slab reinforcement layouts
- Connection detail sheets with dimensioned sketches and bolt/weld callouts
- Customizable report templates with firm branding (logo, header, footer, disclaimer text)
- Export formats: PDF, DXF (for CAD), and CSV data tables for post-processing in spreadsheets

---

## Technical Architecture

### System Diagram

```
                    ┌───────────────────────────────────────┐
                    │     React Frontend (SPA)               │
                    │  3D Modeler │ Results Viewer │ Reports  │
                    │          Three.js / WebGL              │
                    └──────────────────┬────────────────────┘
                                       │ HTTPS / WebSocket
                                       ▼
                    ┌───────────────────────────────────────┐
                    │      Rust API Server (Axum)            │
                    │  Auth │ Projects │ Models │ Jobs       │
                    └──┬──────────┬──────────┬─────────────┘
                       │          │          │
              ┌────────┘     ┌────┘     ┌────┘
              ▼              ▼          ▼
      ┌──────────┐    ┌──────────┐    ┌──────────────────┐
      │PostgreSQL │    │  Redis   │    │  S3 (Object      │
      │ (projects,│    │ (queue,  │    │  Storage)        │
      │  models,  │    │  pubsub, │    │ (IFC files,      │
      │  users,   │    │  session │    │  results, reports)│
      │  results) │    │  cache)  │    │                  │
      └──────────┘    └────┬─────┘    └──────────────────┘
                            │
                 ┌──────────┼──────────────┐
                 ▼          ▼              ▼
         ┌────────────┐ ┌────────────┐ ┌────────────────┐
         │ FEA Solver │ │ Design     │ │ AI Optimization│
         │ Workers    │ │ Check      │ │ Workers        │
         │ (Rust +    │ │ Workers    │ │ (Genetic Algo +│
         │  CUDA GPU) │ │ (Code      │ │  GPU-parallel) │
         │            │ │  compliance│ │                │
         └────────────┘ └────────────┘ └────────────────┘
                 │          │              │
                 └──────────┼──────────────┘
                            ▼
                 ┌────────────────────────┐
                 │   Kubernetes (EKS)     │
                 │   GPU node pools       │
                 │   (NVIDIA A100/T4)     │
                 │   Auto-scaling per     │
                 │   job queue depth      │
                 └────────────────────────┘
```

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | React 18 + TypeScript, Three.js (3D structural visualization), Tailwind CSS, Zustand (state management) |
| 3D Rendering | Three.js with custom structural element renderers (beam, shell, solid), instanced rendering for large models |
| Backend API | Rust (Axum framework), Tokio async runtime, Tower middleware, OpenAPI spec generation |
| FEA Solver | Custom Rust solver with CUDA GPU acceleration via rust-cuda bindings, sparse matrix assembly and factorization |
| Linear Algebra | cuSOLVER (GPU sparse direct), cuSPARSE (GPU sparse ops), Eigen (CPU fallback) |
| Code Checking | Rust-based design engines per AISC 360, ACI 318, Eurocode 2/3, ASCE 7 with configurable code editions |
| AI Optimization | Rust genetic algorithm engine with GPU-parallel fitness evaluation, Python orchestration layer |
| BIM Integration | IFC parser (IfcOpenShell via Python bindings), Revit IFC export mapper, Tekla Open API connector |
| Database | PostgreSQL 16 with JSONB for model definitions, load cases, and analysis parameters |
| Queue / Realtime | Redis 7, Tokio task queue for job management, WebSocket via Axum for live analysis progress |
| Object Storage | AWS S3 for IFC files, analysis results, report PDFs, and large binary model data |
| Auth | JWT-based auth with OAuth (Google, Microsoft), SAML for enterprise SSO |
| Search | Meilisearch for cross-section database search, code clause lookup, and project search |
| Compute Cluster | Kubernetes (EKS) with NVIDIA GPU nodes (A100 for large models, T4 for standard), Karpenter auto-scaling |
| Monitoring | Prometheus + Grafana (infrastructure), Sentry (errors), PostHog (product analytics) |

### Data Model

```
Organization
├── id, name, slug, plan, created_at
│
├── Project[]
│   ├── id, org_id, name, description, building_type
│   ├── code_edition (JSONB: {steel: "AISC360-22", concrete: "ACI318-19", seismic: "ASCE7-22"})
│   ├── units (imperial/metric), status (active/archived)
│   ├── created_by, created_at, updated_at
│   │
│   ├── StructuralModel
│   │   ├── id, project_id, version
│   │   ├── grid_system (JSONB: {x_spacings, y_spacings, z_levels})
│   │   │
│   │   ├── Node[]
│   │   │   ├── id, x, y, z
│   │   │   ├── support_conditions (JSONB: {Fx, Fy, Fz, Mx, My, Mz})
│   │   │   └── mass_source (JSONB, nullable)
│   │   │
│   │   ├── FrameElement[]
│   │   │   ├── id, node_i_id, node_j_id, element_type (beam/column/brace)
│   │   │   ├── section_id → CrossSection, material_id → Material
│   │   │   ├── releases (JSONB: {i_end, j_end: {Mx, My, Mz}})
│   │   │   ├── offsets (JSONB: {i_end, j_end: {dx, dy, dz}})
│   │   │   └── design_group, unbraced_length_override
│   │   │
│   │   ├── AreaElement[]
│   │   │   ├── id, node_ids[] (3 or 4 nodes), element_type (wall/slab/shell)
│   │   │   ├── thickness, material_id → Material
│   │   │   ├── mesh_size, local_axes (JSONB)
│   │   │   └── diaphragm_id (nullable)
│   │   │
│   │   └── Diaphragm[]
│   │       ├── id, story_level, type (rigid/semi_rigid/flexible)
│   │       └── mass_center (JSONB: {x, y})
│   │
│   ├── CrossSection[]
│   │   ├── id, project_id, name, shape_type (W/HSS/L/C/custom)
│   │   ├── properties (JSONB: {A, Ix, Iy, Sx, Sy, Zx, Zy, rx, ry, J, Cw})
│   │   └── dimensions (JSONB: {d, bf, tf, tw, ...})
│   │
│   ├── Material[]
│   │   ├── id, project_id, name, type (steel/concrete/wood/other)
│   │   ├── properties (JSONB: {E, Fy, Fu, fc, density, poisson, alpha})
│   │   └── code_grade (e.g., "A992", "4000psi")
│   │
│   ├── LoadCase[]
│   │   ├── id, project_id, name, type (dead/live/wind/seismic/snow/user)
│   │   ├── auto_generated (boolean), generation_params (JSONB)
│   │   │
│   │   └── Load[]
│   │       ├── id, load_case_id, target_type (node/element/area)
│   │       ├── target_id, load_type (force/moment/distributed/pressure/temperature)
│   │       └── values (JSONB: {Fx, Fy, Fz, Mx, My, Mz, w, direction})
│   │
│   ├── LoadCombination[]
│   │   ├── id, project_id, name, type (LRFD/ASD)
│   │   ├── factors (JSONB: [{load_case_id, factor}])
│   │   └── auto_generated (boolean)
│   │
│   ├── AnalysisJob[]
│   │   ├── id, project_id, analysis_type (linear/pdelta/modal/response_spectrum/time_history/pushover/buckling)
│   │   ├── status (queued/running/completed/failed), gpu_node_id
│   │   ├── parameters (JSONB: analysis-specific settings)
│   │   ├── started_at, completed_at, wall_time_seconds
│   │   ├── results_url (S3), log_url (S3)
│   │   └── error_message, created_at
│   │
│   ├── DesignResult[]
│   │   ├── id, project_id, analysis_job_id, element_id
│   │   ├── code_edition, governing_combination_id
│   │   ├── dcr (float), check_type (flexure/shear/axial/combined/drift)
│   │   ├── detailed_checks (JSONB: [{clause, description, demand, capacity, dcr}])
│   │   └── status (pass/marginal/fail), created_at
│   │
│   └── OptimizationRun[]
│       ├── id, project_id, objective (min_weight/min_cost/min_carbon)
│       ├── status (running/completed), constraints (JSONB)
│       ├── generations_completed, best_fitness, convergence_history (JSONB)
│       ├── optimized_sections (JSONB: [{element_id, original_section, optimized_section}])
│       └── weight_reduction_pct, created_at
│
└── User[]
    ├── id, org_id, email, name, role (admin/engineer/reviewer/viewer)
    ├── license_type, last_active_at
    └── created_at
```

### API Design

```
Auth:
POST   /api/auth/register                           # Create account
POST   /api/auth/login                              # Login
POST   /api/auth/oauth/{provider}                   # Google/Microsoft OAuth

Projects:
GET    /api/projects                                 # List projects
POST   /api/projects                                 # Create project
GET    /api/projects/:id                             # Get project details
PATCH  /api/projects/:id                             # Update project settings
DELETE /api/projects/:id                             # Archive project

Structural Model:
GET    /api/projects/:id/model                       # Get full structural model
PATCH  /api/projects/:id/model                       # Batch update nodes/elements
POST   /api/projects/:id/model/frames                # Create frame elements
POST   /api/projects/:id/model/areas                 # Create area elements
GET    /api/projects/:id/model/viewer-data           # Get 3D viewer mesh data

BIM Import:
POST   /api/projects/:id/import/ifc                  # Upload and import IFC file
POST   /api/projects/:id/import/etabs                # Import ETABS model file
GET    /api/imports/:id/status                       # Get import progress and mapping report

Cross Sections & Materials:
GET    /api/sections/library                         # Search section database (AISC, EU, AU)
POST   /api/projects/:id/sections                    # Add section to project
POST   /api/projects/:id/materials                   # Add material to project

Loads:
POST   /api/projects/:id/load-cases                  # Create load case
GET    /api/projects/:id/load-cases                  # List load cases
POST   /api/projects/:id/load-cases/:lcid/loads      # Add loads to case
POST   /api/projects/:id/load-cases/generate-wind    # Auto-generate wind loads per code
POST   /api/projects/:id/load-cases/generate-seismic # Auto-generate seismic loads per code
POST   /api/projects/:id/combinations/generate       # Auto-generate load combinations per code

Analysis:
POST   /api/projects/:id/analyze                     # Submit analysis job
GET    /api/jobs/:id                                 # Get job status and results
GET    /api/jobs/:id/results/displacements           # Get nodal displacements
GET    /api/jobs/:id/results/forces                  # Get element forces
GET    /api/jobs/:id/results/reactions                # Get support reactions
GET    /api/jobs/:id/results/mode-shapes             # Get modal analysis results
POST   /api/jobs/:id/cancel                          # Cancel running analysis

Design:
POST   /api/projects/:id/design-check                # Run code compliance checks
GET    /api/projects/:id/design-results               # Get DCR results for all elements
GET    /api/projects/:id/design-results/:eid          # Get detailed design checks for element

Optimization:
POST   /api/projects/:id/optimize                    # Start AI optimization run
GET    /api/optimizations/:id                        # Get optimization status and results
POST   /api/optimizations/:id/apply                  # Apply optimized sections to model

Collaboration:
POST   /api/projects/:id/share                       # Generate share link
POST   /api/projects/:id/comments                    # Add comment anchored to element/result
WS     /ws/projects/:id                              # WebSocket for real-time model viewing

Reports:
POST   /api/projects/:id/reports/calculation          # Generate calculation package PDF
POST   /api/projects/:id/reports/drawings             # Generate framing plan DXFs
GET    /api/reports/:id                               # Download generated report
```

---

## UI/UX — Key Screens

### 1. Project Dashboard
- Grid of project cards with 3D thumbnail previews showing building wireframe, project name, building type, and last-modified date
- Quick-create templates: "Steel Moment Frame", "Concrete Shear Wall Building", "Steel Braced Frame", "Residential Wood Frame"
- Recent analysis runs with status badges (completed, running, failed) and one-click navigation to results
- Organization-level statistics: total projects, active analyses, compute hours used this month

### 2. 3D Structural Modeler
- Full-screen Three.js viewport with grid overlay and snap-to-grid point placement for rapid model construction
- Left toolbar: draw beam, draw column, draw brace, draw wall, draw slab, assign support, apply load
- Right property panel: selected element properties (section, material, releases, offsets) with inline editing and section search
- Bottom panel: story/level navigator with quick visibility toggles per story, member list with sortable columns
- Top toolbar: view controls (wireframe, extruded, rendered), display options (element labels, section sizes, load arrows, DCR colors)
- Grid editing dialog for defining story heights and bay widths with visual preview

### 3. Analysis Progress and Live Monitor
- Top status bar: analysis type, model statistics (nodes, elements, DOF, load combinations), GPU allocation, estimated completion time
- Center: real-time convergence plot (residual norm vs. iteration for nonlinear, mode extraction progress for modal) updating via WebSocket
- Side panel: load combination progress tracker showing completed/running/pending combinations
- Log output panel (last 100 lines from solver) with warning and error highlighting
- Cancel button and notification preference (email when complete, browser notification)

### 4. Results Viewer — Deformed Shapes and Stress Contours
- Full-screen Three.js viewport showing deformed structure overlaid on undeformed (ghost) shape with adjustable scale factor
- Left panel: result type selector (displacement, axial force, shear, moment, stress) and load combination dropdown
- Color contour legend with adjustable range, colormap selection (rainbow, blue-red diverging, grayscale)
- Mode shape animation controls for modal results: play/pause, speed, mode number selector with frequency display
- Click any member to see detailed force diagrams (axial, shear V2, shear V3, moment M2, moment M3, torsion) plotted along its length
- Story drift profile panel showing inter-story drift ratios at each level with code limit lines

### 5. Design Check Dashboard
- Summary view: total members by DCR range (< 0.5, 0.5–0.7, 0.7–0.9, 0.9–1.0, > 1.0) with counts and colored bar chart
- 3D model view with all members colored by governing DCR (green to red gradient), click any member for detailed breakdown
- Tabular view: sortable list of all members with columns for element ID, section, governing check, DCR, governing combination, and status
- Detailed member view: clause-by-clause checks with demand, capacity, DCR, and pass/fail for flexure, shear, axial, combined, stability
- Optimization recommendations: "17 members are over-designed (DCR < 0.5). Run AI optimization to reduce structural weight by an estimated 12–18%."

### 6. Report Generation Screen
- Report type selector: Calculation Package, Executive Summary, Framing Plans, Connection Details, Quantity Takeoff
- Preview panel showing live PDF rendering with page thumbnails for navigation
- Customization options: firm logo upload, cover page text, include/exclude sections, level of detail (summary vs. full calculations)
- Template management: save report configurations as templates for reuse across projects
- Export buttons: download PDF, download DXF drawings, download CSV data tables

---

## Monetization

### Free / Student — $0/month
- 1 project, up to 100 nodes and 200 elements
- Linear static analysis only (no P-delta, no dynamic)
- Steel design checks per AISC 360 (basic members only)
- 3D visualization with deformed shapes
- StructAI watermark on report exports
- Community support only
- Ideal for students and learning

### Pro — $149/month
- Unlimited projects, up to 5,000 nodes and 20,000 elements per model
- Full analysis suite: linear, P-delta, buckling, modal, response spectrum
- Steel and concrete design per AISC 360, ACI 318, and ASCE 7
- IFC import from Revit and Tekla
- AI design optimization (5 optimization runs per month)
- PDF calculation packages without watermark
- Load combination auto-generation
- Email support (48h response)

### Team — $399/month (up to 5 seats, $79/additional seat)
- Everything in Pro
- Up to 50,000 nodes and 200,000 elements per model (GPU-accelerated)
- Nonlinear analysis: pushover, time-history, material nonlinearity
- Eurocode and AS/NZS code support in addition to US codes
- Connection design module
- Unlimited AI optimization runs
- Collaboration: shared projects, design review workflows, commenting
- DXF drawing generation and framing plans
- Priority GPU queue and support (24h response)

### Enterprise — Custom
- Unlimited seats, nodes, and elements
- Dedicated GPU compute allocation for guaranteed analysis throughput
- SSO/SAML integration, audit logs, data residency options
- Custom code editions and National Annex configurations
- API access for scripting and automation
- On-premise deployment option for air-gapped environments
- Custom report templates with regulatory compliance formatting
- Dedicated customer success manager and SLA guarantee
- Training and onboarding program for engineering teams

---

## Go-to-Market Strategy

### Phase 1: Engineering Community (Month 1–3)
- Launch free tier and post on Hacker News (Show HN), r/StructuralEngineering, r/civilengineering, Eng-Tips forums, and SEAoC mailing lists
- Publish benchmark validation studies: cantilever beam, portal frame, multi-story building — comparing StructAI results to SAP2000 and hand calculations
- Partner with 5 university structural engineering programs to offer free student access for coursework
- YouTube tutorial series: "Structural Analysis in the Browser" — from model creation to code-checked design in 15 minutes
- Open-source the IFC import library and section property calculator to build developer community trust

### Phase 2: Professional Adoption (Month 3–6)
- Launch Pro tier with self-serve billing targeting small firms and sole practitioners
- Case studies from beta users: "How [Firm] replaced $50K in ETABS licenses with StructAI"
- Content marketing: SEO targeting "structural analysis software", "SAP2000 alternative", "cloud FEA for buildings", "AISC 360 design software"
- Attend NASCC (Steel Conference), ACI Convention, and SEI Structures Congress with live demo booth
- Partner with steel fabricators and concrete suppliers for co-marketing (StructAI helps engineers specify your products)
- Webinar series featuring practicing structural engineers demonstrating real-project workflows

### Phase 3: Enterprise and Vertical Focus (Month 6–12)
- Build enterprise features: SSO, dedicated clusters, API access, custom code configurations
- Hire industry sales team targeting top-100 structural engineering firms (Thornton Tomasetti, Walter P Moore, Magnusson Klemencic)
- Develop vertical-specific workflows: seismic retrofit assessment, progressive collapse analysis, performance-based design
- Integration partnerships with Autodesk (Revit), Trimble (Tekla), and Bentley for seamless BIM round-tripping
- Pursue IBC building code compliance certification and ICC-ES evaluation report for regulatory acceptance

### Acquisition Channels
- Organic search: target "structural engineering software", "ETABS alternative", "cloud-based structural analysis", "AI structural design"
- Eng-Tips forum sponsorship and technical answer contributions by in-house structural engineers
- LinkedIn targeting structural engineer, civil engineer, and building designer job titles at AEC firms
- University partnerships with course adoption driving graduates who prefer StructAI into the workforce
- Building code committee engagement: present StructAI at code development hearings to build credibility

---

## Success Metrics & KPIs

| Metric | Target (Month 6) | Target (Month 12) |
|--------|-------------------|---------------------|
| Registered users | 4,000 | 18,000 |
| Analysis jobs completed (cumulative) | 8,000 | 50,000 |
| GPU compute hours consumed | 15,000 | 120,000 |
| Paying customers (Pro + Team) | 80 | 350 |
| MRR | $18,000 | $95,000 |
| Free → Paid conversion rate | 4% | 7% |
| Average analysis turnaround time | <5 min (linear) | <3 min (linear) |
| Monthly churn rate | <5% | <3% |

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | StructAI Advantage |
|-----------|-----------|------------|-------------------|
| SAP2000 (CSI) | Industry gold standard for general structural analysis, extensive element library, proven nonlinear capabilities | $5K+/seat, desktop-only, dated UI, no AI optimization, no cloud compute, steep learning curve | Browser-based, GPU-accelerated (10x faster), AI optimization, integrated code checking, 95% cheaper |
| ETABS (CSI) | Purpose-built for building structures, excellent concrete design, strong seismic analysis, widely accepted by building officials | $5K+/seat, desktop-only, slow solver for large models, no collaboration, rigid BIM workflow | Cloud-native with real-time collaboration, GPU solver for faster iteration, AI member sizing, modern UX |
| STAAD.Pro (Bentley) | Large install base, Bentley ecosystem integration, broad code support across international standards | Aging interface, inconsistent design results, expensive ($5K+/seat), declining market share and development pace | Modern cloud platform, faster GPU solver, AI-powered design, superior visualization, active development |
| RISA-3D (RISA Tech) | User-friendly interface, good steel and wood design, popular with small firms, lower price point ($2K/seat) | Limited nonlinear analysis, no time-history, weak seismic capabilities, desktop-only, no BIM integration | Full nonlinear and seismic suite, GPU acceleration, AI optimization, BIM import, browser-based |
| Robot Structural Analysis (Autodesk) | Revit integration, included in AEC Collection bundle, Eurocode support | Unreliable results for complex models, poor nonlinear capabilities, confusing UI, Autodesk lock-in | Standalone with better BIM import, validated GPU solver, AI optimization, transparent pricing |

---

## Risks & Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Engineers and building officials distrust cloud-based structural analysis results for permit submissions | High | High | Publish extensive validation studies comparing StructAI to SAP2000 and hand calculations, obtain ICC-ES evaluation report, partner with structural engineering professors for independent verification, allow full calculation export for peer review |
| GPU FEA solver produces numerically different results from established desktop solvers, causing regulatory pushback | High | Medium | Implement rigorous verification suite against NBS benchmarks and NAFEMS tests, use double-precision arithmetic throughout, publish solver theory manual, offer side-by-side comparison mode with SAP2000/ETABS results |
| Cloud compute costs for GPU-accelerated FEA make unit economics unsustainable at $149/month price point | High | Medium | Optimize solver for GPU efficiency (batch matrix operations), use spot instances for non-priority jobs (70% savings), implement intelligent model partitioning to minimize GPU memory, tier compute limits by plan |
| BIM import from Revit/Tekla produces unreliable analytical models requiring extensive manual cleanup | Medium | High | Invest heavily in IFC parsing quality, build smart element recognition and connectivity algorithms, provide detailed import diagnostics with guided manual resolution, maintain compatibility test suite against common Revit templates |
| Incumbent vendors (CSI, Bentley) add cloud and AI features to their existing products, reducing StructAI differentiation | Medium | Medium | Move fast to build AI optimization moat and community, maintain 10x price advantage, focus on superior UX and collaboration that incumbents cannot retrofit into legacy architectures, build open ecosystem (IFC, API) vs. vendor lock-in |

---

## MVP Scope (v1.0)

### In Scope
- User auth, organization and project management
- 3D structural modeler: nodes, frame elements (beams, columns, braces), supports, and cross-section assignment from AISC database
- Load case creation: dead, live, and user-defined point/distributed loads
- Automatic load combination generation per ASCE 7 (LRFD)
- GPU-accelerated linear static analysis (up to 5,000 nodes)
- Results visualization: deformed shapes, member force diagrams (axial, shear, moment), support reactions
- Steel design checks per AISC 360-22 (flexure, shear, axial, combined forces) with DCR display
- PDF calculation report generation with member-by-member design checks

### Out of Scope (v1.1+)
- IFC/BIM import from Revit and Tekla (v1.1 — highest priority post-MVP)
- P-delta and buckling analysis (v1.1)
- Concrete design per ACI 318 (v1.1)
- Modal and response spectrum analysis (v1.2)
- AI design optimization (v1.2)
- Wind and seismic load auto-generation (v1.2)
- Nonlinear time-history and pushover analysis (v1.3)
- Connection design module (v1.3)
- Collaboration and shared workspaces (v1.3)
- Eurocode and AS/NZS code support (v1.4)
- Enterprise features: SSO, on-premise, API access (v2.0)

### MVP Timeline: 10 weeks
- Week 1–2: Auth system, project data model, Rust/Axum API server, PostgreSQL schema, 3D viewport with Three.js (grid, orbit controls, node/element rendering), AISC cross-section database import
- Week 3–4: 3D modeler UI (draw beams, columns, braces, assign sections, define supports), load case creation with graphical load application, load combination generator per ASCE 7
- Week 5–6: GPU FEA solver: stiffness matrix assembly, sparse direct solver (cuSOLVER), displacement/force extraction, WebSocket progress reporting, job queue with Redis
- Week 7–8: Results visualization (deformed shapes with scale control, member force diagrams, reaction arrows, color contours), AISC 360 steel design engine (flexure, shear, axial, combined checks, DCR calculation)
- Week 9–10: Design check dashboard with DCR color-coded model view and tabular results, PDF report generator (model summary, loading, analysis results, member design checks), Stripe billing integration, free/Pro tier enforcement, load testing, and beta launch
