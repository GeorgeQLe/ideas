# PrintGen — AI Generative Design & 3D Print Optimization Platform

## Executive Summary

PrintGen is a cloud-native, browser-based generative design and 3D printing optimization platform that combines GPU-accelerated topology optimization, FEA simulation, lattice structure generation, and slicer integration in a single SaaS. Engineers define loads and constraints, and PrintGen automatically generates optimized, manufacturable geometries using SIMP and level-set methods — then validates them with structural simulation and prepares them for 3D printing. No CAD expertise or local GPU required.

---

## Problem Statement

**The pain:**
- Generative design tools like nTopology ($15K+/seat/year) and Autodesk Fusion 360 Generative ($2,400+/year) are expensive desktop applications that require powerful workstations with high-end GPUs, locking out small engineering firms, freelancers, and startups
- Topology optimization is computationally expensive — a single optimization run on a complex part can take 2-12 hours on a desktop workstation, making iterative design exploration impractical
- The workflow from generative design to 3D printing is fragmented: engineers run optimization in one tool, mesh repair in another (Meshmixer, Netfabb), and slicing in a third (PrusaSlicer, Cura) — with manual file conversion and quality checks at each step
- Lattice structures (gyroid, BCC, diamond) dramatically reduce weight and material usage but are nearly impossible to design manually; existing tools generate lattices disconnected from structural analysis
- Most engineers lack topology optimization expertise — setting up optimization problems (load cases, boundary conditions, volume fractions, manufacturing constraints) requires specialized FEA knowledge

**Current workarounds:**
- Engineers use Autodesk Fusion 360 Generative for cloud-based optimization but face high costs, 24-48 hour cloud queue times, and limited control over optimization parameters
- nTopology offers powerful implicit modeling but requires desktop installation, expensive licenses, and months of training
- Academic researchers use open-source TopOpt (MATLAB/Python) for topology optimization but get no mesh export, no lattice generation, and no manufacturing integration
- Some teams skip generative design entirely and over-engineer parts with safety factors because optimization tools are too expensive or complex

**Market size:** The global generative design market is valued at approximately $250 million (2024) and projected to reach $1.2 billion by 2030 (CAGR ~30%). The 3D printing software market is ~$2 billion, with the design-for-additive-manufacturing (DfAM) segment growing fastest. There are an estimated 2 million+ professional engineers using 3D printing, with 500,000+ using it weekly for production or prototyping.

---

## Target Users

### Primary Personas

**1. Lisa — Mechanical Engineer at a Consumer Electronics Company**
- Designs brackets, enclosures, and structural components that are 3D-printed for prototyping and small-batch production
- Uses SolidWorks for CAD but has no topology optimization tool — manually lightweights parts by removing material based on intuition
- Needs: an accessible tool where she can upload a design space, define loads, and get an optimized geometry ready for 3D printing in minutes, not hours

**2. Omar — Engineering Consultant and Freelancer**
- Designs lightweight structural components for aerospace and automotive clients
- Cannot justify $15K/year for nTopology on his freelance budget; uses free OpenSCAD and manual iterations instead
- Needs: an affordable browser-based generative design tool with professional-grade topology optimization and lattice generation

**3. Dr. Hannah Eriksson — Additive Manufacturing Researcher at a University**
- Studies lattice structures and topology optimization for biomedical implants
- Uses custom MATLAB scripts for topology optimization but struggles with mesh export, FEA validation, and manufacturing preparation
- Needs: an integrated platform that connects optimization → simulation → lattice generation → mesh export in a single workflow

### Secondary Personas
- 3D printing service bureaus (Shapeways, Xometry) who want to offer generative design as a value-added service for customers
- Hobbyist engineers and makers who want to create weight-optimized drone frames, bicycle parts, and custom brackets
- Biomedical engineers designing patient-specific implants with optimized porous structures for bone ingrowth

---

## Solution Overview

PrintGen is a browser-based generative design platform that:
1. Lets users define a design space by uploading a CAD model (STEP, STL, OBJ) or using built-in parametric shape tools, then specify loads, constraints, and optimization objectives (minimize weight, maximize stiffness, target compliance)
2. Runs GPU-accelerated topology optimization using SIMP (Solid Isotropic Material with Penalization) and level-set methods on cloud infrastructure, generating optimized geometries in minutes instead of hours
3. Validates optimized designs with integrated FEA simulation — stress, displacement, and safety factor analysis with color-mapped visualization directly in the browser
4. Generates lattice-filled regions (gyroid, BCC, FCC, diamond, octet-truss) with graded density mapped to stress distributions from FEA results, creating lightweight structures that are impossible to design manually
5. Prepares designs for manufacturing with automatic mesh repair, STL/3MF export, and direct slicer integration for print time and material cost estimation

---

## Core Features

### F1: Design Space Definition
- Upload existing CAD models as design space: STEP, STL, OBJ, 3MF with automatic mesh processing
- Built-in parametric shape tools: box, cylinder, L-bracket, T-bracket, beam, plate with configurable dimensions
- Preserve regions: mark areas that must remain solid (mounting holes, mating surfaces, datum features)
- Exclude regions: mark areas where material cannot exist (clearance zones, assembly interference)
- Symmetry constraints: define 1, 2, or 3 planes of symmetry to ensure manufacturable symmetric designs
- Design space visualization: transparent overlay showing keep/remove/optimize regions with color coding
- Import from Onshape, Fusion 360, or SolidWorks via STEP export

### F2: Load and Constraint Definition
- Visual load application: click surfaces to apply forces, pressures, moments, and gravity loads
- Constraint types: fixed support, pinned support, roller support, displacement constraint, elastic foundation
- Multiple load cases: define separate load scenarios (nominal, peak, fatigue) with load combination rules
- Load magnitude input with engineering units (N, lbf, MPa, psi) and automatic unit conversion
- Thermal loads: temperature distributions for thermo-mechanical optimization
- Visual verification: load arrows and constraint symbols displayed on the 3D model with magnitude labels
- Load case manager: organize, enable/disable, and weight multiple load cases for multi-objective optimization

### F3: GPU-Accelerated Topology Optimization
- SIMP (Solid Isotropic Material with Penalization) method with configurable penalization power and filter radius
- Level-set topology optimization for smoother boundaries and better feature resolution
- Multi-objective optimization: minimize compliance (maximize stiffness), minimize weight, maximize natural frequency
- Volume fraction target: specify the percentage of design space to fill with material (e.g., 30% for maximum lightweighting)
- Manufacturing constraints: minimum feature size, maximum overhang angle (for 3D printing without supports), draw direction (for molding)
- Real-time optimization visualization: watch density field evolve during iteration with convergence plot
- GPU acceleration: NVIDIA A10G/A100 cloud GPUs for 10-50x speedup over CPU, enabling 256³ voxel resolution in minutes
- Multi-resolution: coarse optimization (fast preview) followed by fine-mesh refinement for production geometry

### F4: FEA Simulation and Validation
- Linear static analysis: stress (von Mises, principal), displacement, strain, and safety factor
- GPU-accelerated sparse matrix solver (preconditioned conjugate gradient) for fast FEA
- Color-mapped stress/displacement visualization with configurable color scales and min/max clamping
- Deformed shape animation: exaggerated deformation display to identify failure modes
- Modal analysis: natural frequency extraction and mode shape visualization (first 10 modes)
- Thermal analysis: steady-state temperature distribution from thermal loads
- Mesh refinement: automatic adaptive mesh refinement in high-stress regions
- Report generation: FEA summary with peak stress, max displacement, safety factor, and critical location screenshots

### F5: Lattice Structure Generation
- Unit cell library: Gyroid, Schwarz-P, Diamond, BCC, FCC, Octet-truss, Kelvin cell, custom unit cells
- Graded density: map lattice density to FEA stress results — dense lattice in high-stress regions, sparse in low-stress
- Uniform density: constant lattice fill throughout selected regions with configurable relative density (10-90%)
- Lattice-solid transition: smooth blending between solid regions (mounting features) and lattice-filled regions
- Strut-based lattices: configurable strut diameter, node radius, and connectivity pattern
- TPMS-based lattices: mathematically smooth triply periodic minimal surfaces for improved fatigue life
- Preview mode: show lattice structure with cut-away views and cross-section planes
- Lattice FEA: homogenized material properties for rapid structural analysis of lattice-filled designs

### F6: Mesh Processing and Repair
- Automatic mesh repair: hole filling, non-manifold edge resolution, degenerate triangle removal, normal orientation
- Mesh smoothing: Laplacian and Taubin smoothing with controllable iterations and relaxation factor
- Mesh decimation: reduce triangle count while preserving geometric accuracy for smaller file sizes
- Boolean operations: union, intersection, subtraction between mesh bodies
- Shell extraction: extract the outer surface from topology optimization density field using marching cubes
- Mesh quality metrics: aspect ratio, skewness, Jacobian ratio with visual overlay of poor-quality elements
- Manual editing: vertex move, face delete, bridge holes for fixing stubborn mesh issues

### F7: Material Library
- Pre-configured 3D printing materials with validated mechanical properties:
  - FDM: PLA, PETG, ABS, ASA, Nylon (PA6, PA12), PC, TPU, CF-filled variants
  - SLA/DLP: Standard resin, tough resin, flexible resin, high-temp resin, castable resin
  - SLS: PA12, PA11, TPU, glass-filled PA, aluminum-filled PA
  - Metal: 316L stainless steel, Ti-6Al-4V, AlSi10Mg, Inconel 718, 17-4PH
- Material properties: Young's modulus, yield strength, ultimate tensile strength, elongation, density, fatigue limit, Poisson's ratio
- Custom material input: define materials with custom mechanical properties for non-standard or proprietary materials
- Material comparison: side-by-side property comparison table for material selection
- Temperature-dependent properties for thermal analysis accuracy

### F8: Slicer Integration and Print Preparation
- Built-in basic slicer: layer-by-layer preview with estimated print time, material weight, and cost
- External slicer export: one-click export to PrusaSlicer, Cura, OrcaSlicer, and Bambu Studio file formats
- Print orientation optimization: automatically find the orientation that minimizes support volume and maximizes surface quality
- Support structure preview: visualize where supports will be needed based on overhang angle analysis
- Build plate fitting: check if the design fits within the build volume of common 3D printers (Bambu X1C, Prusa MK4, Formlabs Form 3)
- Cost estimation: material cost + print time cost based on configurable hourly machine rates
- Nesting: arrange multiple parts on a build plate for batch printing efficiency

### F9: Design Exploration
- Parametric sweep: vary optimization parameters (volume fraction, load magnitude, material) and compare results
- Design gallery: grid view of optimization variants with key metrics (weight, max stress, safety factor)
- Pareto frontier: visualize tradeoff between weight and stiffness across design variants
- A/B comparison: overlay two designs with transparency to compare geometries
- Design history: full version control of optimization runs with restore capability
- Favorite and tag designs for easy organization within exploration studies

### F10: Collaboration and Sharing
- Project-based workspaces with role-based access (viewer, editor, admin)
- Design review mode: share read-only 3D viewers with annotation capability for stakeholder review
- Inline commenting on specific features, stress concentrations, or design regions
- Export options: STL, 3MF, OBJ, STEP (via mesh-to-CAD conversion), PDF report
- Embed: iframe-embeddable 3D viewer for design portfolios and documentation
- Fork: duplicate any public project as a starting point for your own optimization

### F11: API and Automation
- REST API for programmatic optimization: submit design spaces, loads, and parameters, retrieve results
- Batch optimization: submit multiple design variants for parallel cloud processing
- Webhook notifications for completed optimization runs
- Integration with CAD platforms via STEP import/export
- CLI tool for scripting repetitive optimization tasks
- Template system: save and reuse optimization configurations for similar part families

### F12: Educational Tools
- Guided tutorials: step-by-step topology optimization on benchmark problems (cantilever beam, MBB beam, bridge)
- Optimization theory explainer: interactive visualizations of SIMP, level-set, and compliance minimization concepts
- FEA fundamentals: visual explanations of stress, strain, boundary conditions, and mesh convergence
- Challenge mode: optimization puzzles with target weight and stiffness goals
- Community gallery: browse and learn from publicly shared generative designs

---

## Technical Architecture

### System Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        Browser Client                            │
│  ┌────────────┐  ┌────────────┐  ┌───────────┐  ┌────────────┐ │
│  │    3D       │  │  Topology  │  │   FEA     │  │  Lattice   │ │
│  │  Viewport   │  │  Optimizer │  │  Results  │  │  Preview   │ │
│  │  (Three.js) │  │  (Live)    │  │  (WebGL)  │  │  (WebGL)   │ │
│  └──────┬──────┘  └──────┬─────┘  └─────┬─────┘  └─────┬──────┘ │
│         │               │              │              │         │
│         └───────────────┼──────────────┼──────────────┘         │
│                         ▼                                       │
│               State Management (Zustand)                         │
│                         │                                       │
└─────────────────────────┼───────────────────────────────────────┘
                          │ HTTPS / WebSocket
                          ▼
                ┌─────────────────────┐
                │   API Gateway       │
                │  (Rust / Axum)      │
                └────┬───────┬────────┘
                     │       │
          ┌──────────┘       └──────────┐
          ▼                             ▼
┌─────────────────┐           ┌─────────────────────┐
│   PostgreSQL    │           │   Compute Engine     │
│  (Users, Proj,  │           │                      │
│   Designs,      │           │  ┌────────────────┐  │
│   Results)      │           │  │ TopOpt Solver  │  │
└────────┬────────┘           │  │ (GPU SIMP/LS)  │  │
         │                    │  └────────────────┘  │
         ▼                    │  ┌────────────────┐  │
┌─────────────────┐           │  │ FEA Solver     │  │
│     Redis       │           │  │ (GPU CG)       │  │
│  (Sessions,     │◄──────────│  └────────────────┘  │
│   Job Queue,    │           │  ┌────────────────┐  │
│   Pub/Sub)      │           │  │ Lattice Gen    │  │
└─────────────────┘           │  │ (Rust + GPU)   │  │
                              │  └────────────────┘  │
                              │  ┌────────────────┐  │
                              │  │ Mesh Processor │  │
                              │  │ (Rust)         │  │
                              │  └────────────────┘  │
                              └─────────────────────┘
                                        │
                                        ▼
                              ┌─────────────────┐
                              │  Object Storage │
                              │  (S3/R2)        │
                              │  CAD files, STL,│
                              │  Results, Mesh  │
                              └─────────────────┘
```

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend Framework | React 19 + TypeScript, Vite build system |
| 3D Viewport | Three.js with custom mesh rendering, stress color mapping, and section plane tools |
| Optimization Viz | WebSocket-streamed density field updates rendered as voxel grid in Three.js |
| UI Components | Tailwind CSS, Radix UI, Recharts for convergence plots |
| State Management | Zustand for client state, React Query for server state |
| Backend API | Rust (Axum) for project management and compute orchestration |
| Topology Optimizer | Rust with wgpu compute shaders for SIMP and level-set optimization |
| FEA Solver | Rust GPU-accelerated sparse solver (algebraic multigrid + PCG) |
| Lattice Generator | Rust with GPU-accelerated implicit surface evaluation (marching cubes) |
| Mesh Processing | Rust mesh library (repair, smooth, decimate, boolean operations) |
| Database | PostgreSQL 16 with JSONB for optimization parameters and results |
| Cache / Queue | Redis 7 for job queuing, session management, and optimization progress |
| Object Storage | Cloudflare R2 for CAD uploads, mesh files, and optimization results |
| GPU Compute | Kubernetes (EKS) with NVIDIA A10G/A100 GPU nodes for optimization and FEA |
| Auth | JWT-based auth with OAuth (Google, GitHub), SAML for enterprise |
| Monitoring | Grafana + Prometheus, Sentry, custom optimization convergence metrics |

### Data Model

```
User
├── id (uuid), email, name, avatar_url
├── auth_provider, plan (free/pro/team/enterprise)
└── created_at, updated_at

Organization
├── id (uuid), name, slug, owner_id → User
├── plan, seat_count, billing_email
└── compute_quota_hours_monthly, settings (JSONB)

Project
├── id (uuid), org_id → Organization
├── name, description, visibility (private/org/public)
├── created_by → User, forked_from → Project (nullable)
└── created_at, updated_at

DesignSpace
├── id (uuid), project_id → Project
├── name, source_file_url (S3 — STEP/STL/OBJ upload)
├── mesh_url (S3 — processed mesh), bounding_box (JSONB)
├── keep_regions (JSONB — list of preserved region definitions)
├── exclude_regions (JSONB — list of excluded region definitions)
├── symmetry_planes (JSONB — list of symmetry constraints)
└── created_by → User, created_at, updated_at

LoadCase
├── id (uuid), design_space_id → DesignSpace
├── name, description
├── loads (JSONB — list of force/pressure/moment/gravity definitions)
├── constraints (JSONB — list of fixed/pinned/roller definitions)
├── weight (float — for multi-load-case optimization)
└── created_by → User, created_at

Material
├── id (uuid), name, category (fdm/sla/sls/metal/custom)
├── youngs_modulus_mpa (float), poissons_ratio (float)
├── yield_strength_mpa (float), ultimate_strength_mpa (float)
├── density_kg_m3 (float), elongation_pct (float)
├── fatigue_limit_mpa (float, nullable)
├── is_builtin (boolean), org_id → Organization (nullable for builtins)
└── created_at

OptimizationRun
├── id (uuid), project_id → Project
├── design_space_id → DesignSpace, material_id → Material
├── load_case_ids (uuid[])
├── method (simp/level_set), objective (min_compliance/min_weight/max_frequency)
├── params (JSONB — volume_fraction, penalty, filter_radius, min_feature_size, max_overhang)
├── resolution (int — voxel grid size per dimension)
├── status (queued/running/completed/failed)
├── iteration_count, convergence_data (JSONB — compliance per iteration)
├── result_mesh_url (S3), density_field_url (S3)
├── result_metrics (JSONB — final_weight, final_compliance, final_volume_fraction)
├── gpu_type, compute_time_seconds, cost_usd
└── started_by → User, started_at, completed_at

FEAResult
├── id (uuid), optimization_run_id → OptimizationRun (nullable)
├── design_space_id → DesignSpace, material_id → Material
├── load_case_id → LoadCase
├── status (queued/running/completed/failed)
├── results (JSONB — max_stress, max_displacement, safety_factor, natural_frequencies)
├── stress_field_url (S3), displacement_field_url (S3)
├── mesh_quality (JSONB — element_count, avg_aspect_ratio)
└── created_at

LatticeConfig
├── id (uuid), optimization_run_id → OptimizationRun
├── unit_cell_type (gyroid/schwarz_p/diamond/bcc/fcc/octet)
├── density_mapping (uniform/stress_graded/custom)
├── min_density (float), max_density (float)
├── cell_size_mm (float), strut_diameter_mm (float, for strut-based)
├── result_mesh_url (S3), result_metrics (JSONB — weight, surface_area)
└── created_at

Comment
├── id (uuid), project_id → Project
├── author_id → User, body (text)
├── anchor_type (design/optimization/fea/lattice)
├── anchor_data (JSONB), resolved (boolean)
└── created_at
```

### API Design

```
Auth:
POST   /api/auth/register                  # Create account
POST   /api/auth/login                     # Login
POST   /api/auth/oauth/:provider           # OAuth
POST   /api/auth/refresh                   # Refresh token

Projects:
GET    /api/projects                       # List projects
POST   /api/projects                       # Create project
GET    /api/projects/:id                   # Get project
PATCH  /api/projects/:id                   # Update project
DELETE /api/projects/:id                   # Delete project
POST   /api/projects/:id/fork              # Fork project

Design Spaces:
POST   /api/projects/:id/design-spaces      # Upload/create design space
GET    /api/design-spaces/:id               # Get design space
PATCH  /api/design-spaces/:id               # Update regions/symmetry
GET    /api/design-spaces/:id/mesh          # Get processed mesh

Load Cases:
GET    /api/design-spaces/:id/load-cases    # List load cases
POST   /api/design-spaces/:id/load-cases    # Create load case
PATCH  /api/load-cases/:id                 # Update load case
DELETE /api/load-cases/:id                 # Delete load case

Materials:
GET    /api/materials                      # List materials (built-in + custom)
POST   /api/materials                      # Create custom material
GET    /api/materials/:id                  # Get material properties

Optimization:
POST   /api/projects/:id/optimize           # Start optimization run
GET    /api/optimization-runs/:id           # Get run status/results
WS     /ws/optimization-runs/:id           # Live density field stream
POST   /api/optimization-runs/:id/stop      # Stop optimization
GET    /api/optimization-runs/:id/mesh      # Download result mesh

FEA:
POST   /api/projects/:id/fea               # Run FEA simulation
GET    /api/fea-results/:id                # Get FEA results
GET    /api/fea-results/:id/stress-field   # Get stress visualization data
GET    /api/fea-results/:id/displacement    # Get displacement data

Lattice:
POST   /api/optimization-runs/:id/lattice   # Generate lattice structure
GET    /api/lattice-configs/:id             # Get lattice config/results
GET    /api/lattice-configs/:id/mesh        # Download lattice mesh
GET    /api/lattice-configs/:id/preview     # Get lattice preview data

Mesh:
POST   /api/mesh/repair                    # Repair uploaded mesh
POST   /api/mesh/smooth                    # Smooth mesh
POST   /api/mesh/decimate                  # Reduce triangle count
POST   /api/mesh/boolean                   # Boolean operations
POST   /api/mesh/export                    # Export as STL/3MF/OBJ

Print Preparation:
POST   /api/print/orient                   # Optimize print orientation
POST   /api/print/estimate                 # Estimate print time/cost
POST   /api/print/slice-preview            # Generate layer preview
GET    /api/print/printers                 # List supported printer profiles

Collaboration:
GET    /api/projects/:id/comments           # List comments
POST   /api/projects/:id/comments           # Add comment
PATCH  /api/comments/:id                   # Edit/resolve comment
POST   /api/projects/:id/share             # Generate share link
```

---

## UI/UX — Key Screens

### 1. Project Dashboard
- Card grid showing projects with 3D design thumbnails, optimization status badges, and key metrics (weight, safety factor)
- Quick-start wizard: upload CAD model or select from parametric templates (bracket, beam, plate)
- Recent activity: optimizations completed, FEA results ready, designs shared
- Compute usage meter showing GPU hours consumed and remaining quota

### 2. Design Setup
- 3D viewport with uploaded model, interactive region painting (keep/exclude/optimize with color-coded overlay)
- Left panel: load case list with force/constraint definition tools and visual arrows on the model
- Right panel: material selector with property cards, optimization parameters (volume fraction, method, constraints)
- Bottom bar: mesh quality metrics, estimated optimization time, and "Optimize" launch button
- Multi-load-case tabs with per-case visualization toggle

### 3. Optimization Live View
- Full-screen 3D viewport showing the density field evolving in real-time as the optimizer iterates
- Convergence plot: compliance/objective value vs. iteration number with current iteration highlighted
- Progress bar: iteration count, elapsed time, estimated remaining time
- Controls: pause, resume, stop, accept current result
- Intermediate results: capture the current state at any iteration for comparison

### 4. FEA Results Viewer
- 3D model with color-mapped stress (von Mises) or displacement overlay and adjustable color scale
- Result selector: switch between stress types (von Mises, principal, shear), displacement, safety factor
- Deformed shape toggle with exaggeration slider (1x to 100x deformation)
- Probe tool: click any point to see exact stress/displacement values
- Report panel: summary table with peak values, critical locations, and safety factor assessment
- Compare view: overlay original and optimized design with metrics side by side

### 5. Lattice Designer
- 3D cut-away view showing lattice fill within the optimized geometry
- Unit cell selector with visual previews of each lattice type (gyroid, BCC, diamond, etc.)
- Density controls: uniform slider or stress-graded with min/max density and cell size
- Cross-section plane: drag through the model to inspect internal lattice structure
- Lattice FEA results: effective stiffness and strength of the lattice-filled design
- Weight comparison: original solid vs. lattice-filled with percentage weight reduction

### 6. Print Preparation
- Orientation optimizer: rotate the part to find minimum support volume with support preview
- Build plate view: check fit within selected printer dimensions with build volume wireframe
- Layer preview: slider to scrub through print layers with per-layer detail
- Cost/time estimate: material weight, print hours, cost at configurable hourly rate
- Export buttons: STL, 3MF, OBJ with configurable mesh resolution
- Direct slicer launch: open in PrusaSlicer/Cura web interface with pre-configured settings

---

## Monetization

### Free Tier
- 2 projects, 3 optimization runs per month
- Maximum 64³ voxel resolution (fast but coarse results)
- Basic FEA (linear static only)
- 2 materials (PLA, Standard Resin)
- STL export only
- Community support
- PrintGen watermark on exports

### Pro — $99/month
- Unlimited projects and optimization runs
- Up to 256³ voxel resolution with GPU acceleration
- Full FEA suite (static, modal, thermal)
- Complete material library (50+ materials)
- Lattice structure generation (all unit cell types)
- Mesh repair and processing tools
- STL, 3MF, OBJ export
- Print orientation optimization and cost estimation
- Email support with 48-hour response

### Team — $299/month (up to 5 seats, $59/additional seat)
- Everything in Pro
- Up to 512³ voxel resolution for production-quality designs
- Batch optimization with parametric sweeps
- Team collaboration with comments and design review
- Design version control
- API access for automation
- Custom material definitions
- Priority support with 24-hour response

### Enterprise — Custom
- Unlimited resolution and compute
- Dedicated GPU cluster with guaranteed capacity
- SAML/SSO, audit logging
- Custom slicer integrations
- White-label embedding for 3D printing service bureaus
- Dedicated customer success manager and SLA
- STEP export (mesh-to-CAD conversion)
- On-premise deployment option

---

## Go-to-Market Strategy

### Phase 1: Community Seeding (Month 1-3)
- Launch free tier on Product Hunt, Hacker News, and 3D printing communities (r/3Dprinting, r/AdditiveManufacturing, Printables forums)
- Tutorial series: "Topology Optimization Explained — Design a Lightweight Bracket in 10 Minutes"
- Open-source the mesh repair library on GitHub to build developer community trust
- Partner with 3D printing YouTubers (Makers Muse, CNC Kitchen, Teaching Tech) for design challenges
- Free Pro access for 100 early-adopter engineering teams

### Phase 2: Professional Adoption (Month 3-6)
- Launch lattice generation and full FEA features with benchmarks against nTopology and Fusion 360 Generative
- Case studies: weight reduction achievements with real parts (drone frame, bicycle stem, aerospace bracket)
- SEO content: "nTopology alternative", "free topology optimization", "generative design for 3D printing"
- Partner with 3D printing service bureaus (Xometry, Protolabs, Shapeways) for integrated ordering
- Exhibit at RAPID+TCT, Formnext, and ASME manufacturing conferences

### Phase 3: Enterprise and Industrial (Month 6-12)
- Launch Team and Enterprise tiers with collaboration, API, and custom materials
- Target aerospace (lightweighting) and medical device (lattice implants) companies
- Build STEP export for integration with traditional CAD/CAM workflows
- Launch PrintGen Marketplace for community-contributed design templates and material profiles
- Pursue ISO 9001 and AS9100 compliance documentation support for aerospace customers

### Acquisition Channels
- Organic search: "free topology optimization", "generative design online", "lattice structure generator"
- 3D printing community engagement and design challenges
- University engineering course partnerships (mechanical, aerospace, biomedical)
- 3D printing service bureau integrations for organic discovery
- Referral program: share a design, both users get extra optimization runs

---

## Success Metrics & KPIs

| Metric | Target (Month 6) | Target (Month 12) |
|--------|-------------------|---------------------|
| Registered users | 10,000 | 50,000 |
| Monthly active projects | 3,000 | 15,000 |
| Paying customers (Pro + Team) | 200 | 1,000 |
| MRR | $25,000 | $120,000 |
| Optimization runs per month | 8,000 | 50,000 |
| STL exports per month | 5,000 | 30,000 |
| Free → Paid conversion rate | 4% | 7% |
| Monthly churn rate | <6% | <4% |

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | PrintGen Advantage |
|-----------|-----------|------------|-------------------|
| nTopology | Powerful implicit modeling, lattice generation, production-proven | $15K+/seat, desktop-only, steep learning curve, months to become proficient | Browser-based, 50x cheaper, intuitive wizard-based UX, minutes to first result |
| Autodesk Fusion 360 Generative | Cloud-based optimization, integrated CAD, large user base | $2,400+/year, 24-48 hour cloud queue times, limited lattice support, Autodesk lock-in | Real-time GPU optimization (minutes vs. hours), superior lattice generation, no CAD lock-in |
| Altair Inspire | Excellent topology optimization, Altair solver accuracy | $8K+/seat, desktop-only, complex UI, enterprise-focused | Browser-based, accessible pricing, 3D print-focused workflow |
| ANSYS Topology Optimization | Industry-standard FEA, high-accuracy results | $30K+/seat, extreme complexity, requires FEA expertise | Guided wizard UI, no FEA expertise required, integrated print preparation |
| TopOpt (Academic) | Free, open-source, educational, well-documented theory | No mesh export, no lattice, no manufacturing prep, MATLAB/Python only | Full design-to-print workflow, production-quality output, collaboration |

---

## Risks & Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| GPU optimization costs eat into margins for compute-heavy users | High | High | Tiered resolution limits per plan, spot instance scheduling, caching common optimization patterns, efficient solver algorithms |
| Optimized geometries fail structural validation on closer analysis | High | Medium | Integrated FEA validation as mandatory post-optimization step, safety factor requirements, conservative material property defaults |
| 3D printing community perceives generative design as "gimmicky" | Medium | Medium | Focus on practical engineering use cases (brackets, mounts, structural parts), publish weight/strength comparison data, partner with respected engineering channels |
| Enterprise CAD users need STEP export (mesh-to-B-rep is hard) | Medium | High | Implement mesh-to-STEP conversion using implicit surface fitting, partner with CAD kernel providers, offer alternative IGES/STEP export quality levels |
| Competition from free Fusion 360 generative design for hobbyists | Medium | Medium | Differentiate on lattice generation, faster cloud optimization, print preparation workflow, and team collaboration features |

---

## MVP Scope (v1.0)

### In Scope
- Design space upload (STL, OBJ) with keep/exclude region painting
- Load and constraint definition with visual arrows and supports
- SIMP topology optimization on cloud GPU (up to 128³ voxel resolution)
- Real-time optimization visualization with convergence plot
- Basic FEA validation (linear static, von Mises stress, displacement)
- STL mesh export with basic mesh repair (hole filling, normal orientation)
- Material library with 10 common 3D printing materials
- User accounts, project CRUD, and optimization history
- Print orientation optimization and time/cost estimation

### Out of Scope (v1.1+)
- Lattice structure generation (v1.1 — highest priority post-MVP)
- Level-set optimization method (v1.1)
- Advanced FEA (modal analysis, thermal) (v1.2)
- Full mesh processing suite (smooth, decimate, boolean) (v1.2)
- STEP import and parametric shape tools (v1.2)
- Team collaboration and design review (v1.3)
- API access and batch optimization (v1.3)
- Slicer integration and direct 3MF export (v1.3)
- Enterprise features: SSO, STEP export, white-label (v2.0)

### MVP Timeline: 10 weeks
- Week 1-2: Auth system, project data model, Three.js 3D viewport, STL/OBJ upload and mesh processing pipeline
- Week 3-4: Design space region painting (keep/exclude), load and constraint definition UI, load case data model
- Week 5-6: SIMP topology optimization solver (Rust + wgpu GPU), cloud job management, real-time density visualization via WebSocket
- Week 7-8: FEA solver (GPU-accelerated CG), stress/displacement visualization, mesh export (STL), basic mesh repair
- Week 9-10: Material library, print orientation optimizer, project dashboard, billing integration (Stripe), documentation, launch
