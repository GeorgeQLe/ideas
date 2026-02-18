# PhotonPath — Cloud Photonics Simulation & PIC Design Platform

## Executive Summary

PhotonPath is a cloud-native, browser-based photonics simulation and photonic integrated circuit (PIC) design platform that brings GPU-accelerated FDTD electromagnetic simulation, eigenmode solving, waveguide layout editing, S-parameter extraction, and real-time optical field visualization to photonics engineers via the browser. By replacing $50K+/seat desktop tools like Lumerical FDTD and COMSOL Wave Optics with a collaborative SaaS, PhotonPath democratizes photonic design for silicon photonics startups, university research groups, and integrated photonics teams who cannot afford or do not want to maintain expensive desktop EDA installations.

---

## Problem Statement

**The pain:**
- Lumerical FDTD Solutions costs $50K–$100K per seat per year, and COMSOL Wave Optics adds $25K+ on top of the base COMSOL license — these prices lock out startups, small design houses, and most university research groups from professional photonic simulation
- FDTD simulation is computationally brutal: a single 3D simulation of a photonic component can take 4–24 hours on a high-end workstation, tying up expensive GPU hardware and blocking engineers from iterating on designs
- Photonic IC layout tools are fragmented — engineers use Lumerical for simulation, KLayout or custom scripts for GDS layout, separate tools for S-parameter fitting, and MATLAB/Python for post-processing, with no integrated workflow from design to tapeout
- Collaboration on photonic designs is primitive: sharing simulation files requires transferring multi-gigabyte FDTD project directories, and results can only be viewed in the same proprietary software that generated them
- The photonics talent pipeline is constrained because universities cannot afford enough simulation licenses for their integrated photonics courses, so students graduate with limited hands-on design experience

**Current workarounds:**
- Paying for Lumerical FDTD + MODE + INTERCONNECT licenses ($100K+/year for a full suite) and running on local GPU workstations or shared compute clusters with license bottlenecks
- Using open-source FDTD tools (MEEP from MIT) that require deep Python/Linux expertise, offer no GUI, and lack eigenmode solving and PIC layout capabilities
- Using Tidy3D (cloud FDTD) which addresses compute but costs $2–$10 per simulation, adds up fast for optimization sweeps, and offers no PIC layout or circuit simulation
- Outsourcing photonic design to specialized consultancies at $10K–$50K per project with multi-week turnaround and loss of design iteration speed

**Market size:** The global photonics simulation software market is valued at approximately $1.8 billion (2024) and is projected to reach $3.5 billion by 2030, driven by growth in silicon photonics for data center interconnects, LiDAR for autonomous vehicles, quantum photonic computing, and biosensing. There are an estimated 80,000 photonics engineers and researchers worldwide, with 15,000+ photonics graduate students trained annually.

---

## Target Users

### Primary Personas

**1. Dr. Mei-Lin Chen — Silicon Photonics Engineer at a Datacom Startup**
- Designs wavelength-division multiplexing (WDM) components for 800G optical transceivers targeting hyperscale data centers
- Currently uses Lumerical FDTD ($60K/seat) and MODE Solutions ($40K/seat) — two licenses shared among six engineers creates a constant bottleneck
- Spends 30% of her time waiting for FDTD simulations to complete on a shared GPU workstation ($25K) that is booked solid during tape-out crunch
- Needs: fast cloud-based FDTD with GPU acceleration, eigenmode solver for waveguide design, and integrated PIC layout so she can go from component simulation to chip layout in one tool

**2. Prof. Jamal Okonkwo — Integrated Photonics Researcher at a University**
- Runs a 12-person research group studying programmable photonic circuits for neuromorphic computing
- Has one Lumerical license shared among 12 students and postdocs — students wait days for simulation access, slowing paper deadlines
- Students spend weeks learning Lumerical's GUI and scripting language instead of focusing on photonic design concepts
- Needs: affordable browser-based simulation that all students can access simultaneously, with educational pricing and built-in tutorials

**3. Sarah Lindström — Photonic IC Layout Designer at a PIC Foundry**
- Designs standard cells and PDK components for a silicon nitride (SiN) photonic foundry serving 50+ customers
- Uses a mix of KLayout, custom Python scripts, and Lumerical INTERCONNECT to design, simulate, and validate photonic components
- Needs: an integrated environment with layout editor, FDTD simulation, S-parameter extraction, and circuit-level verification that works with her foundry's PDK

### Secondary Personas
- LiDAR engineers designing optical phased arrays and grating couplers for autonomous vehicle sensing
- Quantum photonics researchers simulating single-photon sources and waveguide-coupled detectors
- Biosensing engineers designing ring resonator sensors for point-of-care diagnostics

---

## Solution Overview

PhotonPath is a browser-based photonics simulation and PIC design platform that:
1. Runs GPU-accelerated 2D and 3D FDTD electromagnetic simulations in the cloud, completing in minutes what takes hours on a local workstation, with real-time field visualization streamed to the browser during simulation
2. Provides an eigenmode solver for waveguide analysis — computing effective index, group index, dispersion, mode profiles, and confinement factors for arbitrary cross-sections across wavelength and geometry sweeps
3. Includes a photonic IC layout editor with waveguide routing, component placement, and GDS-II export, supporting silicon photonics (SOI), silicon nitride (SiN), indium phosphide (InP), and polymer waveguide platforms
4. Extracts S-parameters (insertion loss, return loss, cross-talk, phase response) from FDTD simulations and feeds them into a photonic circuit simulator for system-level verification of complex PICs with hundreds of components
5. Visualizes electromagnetic fields (E-field, H-field, Poynting vector, mode profiles) in real-time with interactive color mapping, cross-section slicing, and time-domain animation directly in the browser using WebGL

---

## Core Features

### F1: GPU-Accelerated FDTD Simulation
- Cloud-based 2D and 3D Finite-Difference Time-Domain electromagnetic simulation with GPU acceleration (CUDA/wgpu compute shaders)
- Automatic mesh generation with non-uniform conformal meshing that refines at material boundaries and high-index-contrast interfaces
- Perfectly matched layer (PML) absorbing boundary conditions with configurable layer count and polynomial grading
- Broadband simulation using Gaussian pulse excitation with user-defined center wavelength and bandwidth (e.g., 1500–1600 nm for C-band)
- Source types: plane wave, mode source (launches a specific waveguide mode), dipole source (for emission studies), Gaussian beam, and total-field/scattered-field (TFSF)
- Material models: Drude, Lorentz, Debye dispersive models with automatic fitting from experimental refractive index data (Palik, Aspnes, custom)
- Sub-pixel averaging for accurate modeling of curved and angled interfaces on a Cartesian grid
- Simulation checkpointing: save and resume long-running 3D FDTD simulations without losing progress

### F2: Eigenmode Solver
- Finite-element eigenmode solver for computing guided modes of arbitrary 2D waveguide cross-sections
- Output per mode: effective index (neff), group index (ng), loss (dB/cm), mode profile (Ex, Ey, Ez, Hx, Hy, Hz), confinement factor, and effective area
- Wavelength sweep: compute neff and ng vs. wavelength for dispersion analysis with automatic group velocity dispersion (GVD) calculation
- Geometry sweep: vary waveguide width, height, etch depth, or gap to map mode behavior across design parameters
- Bend mode solver: compute radiation loss and effective index shift for curved waveguides as a function of bend radius
- Overlap integral calculation: quantify coupling efficiency between modes (e.g., fiber mode to waveguide mode, TE0 to TE1)
- Support for anisotropic and magneto-optic materials for non-reciprocal device design (isolators, circulators)

### F3: Photonic Layout Editor
- Browser-based PIC layout editor with hierarchical cell-based design and waveguide routing
- Waveguide drawing tools: straight, curved (Euler bend, circular arc, S-bend), taper, and multi-mode interference (MMI) coupler primitives
- Automatic waveguide routing with DRC-aware path-finding that respects minimum bend radius and waveguide spacing rules
- Component library palette with drag-and-drop placement of standard photonic components (directional couplers, ring resonators, grating couplers, MMIs, Y-junctions)
- Layer stack visualization showing cross-section of the photonic process at any point in the layout
- GDS-II and OASIS import/export with full hierarchy preservation and layer mapping
- Snap-to-grid with configurable grid resolution (1 nm typical for photonics)
- Design rule checking (DRC) for photonic-specific rules: minimum waveguide width, minimum bend radius, minimum gap between waveguides, taper length constraints

### F4: S-Parameter Extraction
- Automated S-parameter extraction from FDTD simulations by placing port monitors at waveguide inputs and outputs
- Multi-port support: extract full NxN scattering matrix for components with 2 to 16+ ports
- Frequency-dependent S-parameters (magnitude and phase) across the simulation bandwidth with configurable frequency resolution
- Key metrics computed automatically: insertion loss (IL), return loss (RL), cross-talk (XT), extinction ratio (ER), 3-dB bandwidth, and center wavelength
- S-parameter fitting: rational polynomial fitting for compact behavioral models compatible with circuit simulators
- Export formats: Touchstone (.s2p, .snp), CSV, and PhotonPath native format for circuit simulation
- Reciprocity and passivity enforcement for physically consistent S-parameter models
- Batch extraction: sweep a geometry parameter and extract S-parameters at each point for design space exploration

### F5: Optical Field Visualization
- Real-time 2D and 3D electromagnetic field visualization rendered in the browser via WebGL
- Field components: Ex, Ey, Ez, Hx, Hy, Hz, |E|^2, |H|^2, Poynting vector (Sx, Sy, Sz), and energy density
- Color maps: sequential (viridis, inferno, plasma, magma), diverging (RdBu, coolwarm), and custom user-defined with adjustable dynamic range (dB scale or linear)
- Cross-section slicing: XY, XZ, and YZ plane slicing with interactive slider for plane position, updated in real time
- Time-domain animation: playback of field evolution during FDTD simulation with adjustable speed, pause, and frame-by-frame stepping
- Mode profile visualization: 2D contour plots of eigenmode field components with effective index and loss overlay
- Far-field projection: compute and display far-field radiation patterns from near-field FDTD data (useful for grating couplers and antennas)
- Screenshot and video export: high-resolution PNG/SVG screenshots and MP4/GIF animations for publications and presentations

### F6: Component Library
- Built-in parameterized component library for common photonic building blocks
- Waveguide components: straight waveguide, taper (linear, parabolic), S-bend (cosine, Euler), crossing
- Couplers: directional coupler, MMI 1x2/2x2, adiabatic coupler, grating coupler (fiber-to-chip), edge coupler
- Resonators: ring resonator (single/double bus), racetrack resonator, disk resonator, photonic crystal cavity
- Modulators: Mach-Zehnder interferometer (MZI), ring modulator, phase shifter (thermo-optic, electro-optic)
- Filters: arrayed waveguide grating (AWG), Bragg grating, cascaded ring filter, lattice filter
- Each component is fully parameterized (width, length, gap, radius, etc.) with default values from published literature and PDK-specific constraints

### F7: Photonic Circuit Simulation
- Frequency-domain photonic circuit simulator using S-parameter matrix composition for system-level PIC verification
- Hierarchical circuit assembly: connect S-parameter models of individual components to simulate full PIC behavior
- Support for hundreds of components with automatic topology extraction from the layout editor
- Wavelength sweep: compute circuit-level transmission, reflection, and inter-port coupling across the full optical bandwidth
- Time-domain circuit simulation using inverse FFT for transient analysis of modulated optical signals
- Mixed-signal co-simulation: connect photonic circuit models with electrical driver/receiver models for full link budget analysis
- Monte Carlo analysis: vary component parameters (coupling ratios, waveguide widths, effective indices) with statistical distributions to predict fabrication yield

### F8: Optimization Engine
- Gradient-based adjoint optimization: compute gradients with respect to thousands of geometry parameters using only two FDTD simulations (forward + adjoint)
- Topology optimization: optimize arbitrary permittivity distributions within a design region to achieve target S-parameters (inverse design)
- Parametric optimization: sweep geometric parameters (widths, gaps, lengths, radii) using Bayesian optimization, genetic algorithms, or Nelder-Mead
- Multi-objective optimization: simultaneously optimize insertion loss, bandwidth, and footprint with Pareto front visualization
- Fabrication constraint enforcement: minimum feature size, curvature radius, and binary material constraints for manufacturable designs

### F9: Material Database
- Comprehensive built-in optical material database with refractive index and extinction coefficient (n, k) data vs. wavelength
- Dielectrics and semiconductors: Si, SiO2, Si3N4, InP, GaAs, LiNbO3, Ge, Al2O3 with data from peer-reviewed references (Palik, Aspnes)
- Metals: Au, Ag, Al, Cu, TiN with Drude-Lorentz fit parameters for plasmonic simulations
- Polymers: SU-8, PMMA, BCB, Ormocomp with thermo-optic coefficients
- Custom material upload: import n,k data from CSV or ellipsometry measurements with automatic Lorentz model fitting
- Temperature-dependent material models for thermo-optic simulation

### F10: Process Design Kit (PDK) Support
- Built-in PDK support for major open-access photonic platforms: SiEPIC (UBC), IME A*STAR, AMF, IMEC iSiPP, AIM Photonics, LioniX TriPleX
- PDK contents: layer stack definition, waveguide cross-sections, design rules (min width, min bend radius, min gap), and pre-characterized component S-parameter models
- PDK-aware layout editor: waveguide width and bend radius locked to PDK-allowed values, DRC checks against PDK rules in real time
- Automatic technology mapping: change PDK and re-simulate the same circuit topology with updated material parameters and design rules
- Custom PDK creation wizard: define layer stack, waveguide geometry, design rules, and upload characterized component models
- PDK documentation viewer: process specifications, layer descriptions, and component datasheets accessible inline while designing

### F11: Collaboration and Sharing
- Project-based workspaces with role-based access control (viewer, editor, admin)
- Share simulation results via public link with interactive 3D field visualization — no account required to view
- Comment and annotate directly on layout views and simulation results with spatial anchoring
- Real-time presence indicators showing who is viewing or editing which project
- Version history with tagged milestones (e.g., "design review", "tape-out candidate") and restore capability
- Template library: save and share simulation setups, component configurations, and circuit topologies as reusable project templates

### F12: Export and Integration
- GDS-II and OASIS export of photonic layouts with configurable layer mapping for foundry tape-out
- S-parameter export in Touchstone format (.s2p, .snp) for import into Lumerical INTERCONNECT, Cadence, or custom simulators
- FDTD field data export in HDF5 and VTK formats for post-processing in MATLAB, Python, or ParaView
- Python SDK for scripted simulation setup, batch job submission, and result retrieval via REST API
- Lumerical LSF script import: convert existing Lumerical scripts to PhotonPath projects for migration
- Integration with KLayout for layout viewing and SPICE netlist export for electronic co-simulation

---

## Technical Architecture

### System Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        Browser Client                            │
│  ┌────────────┐  ┌────────────┐  ┌───────────┐  ┌────────────┐ │
│  │   Layout    │  │   Field    │  │  Circuit   │  │  Eigenmode │ │
│  │   Editor    │  │   Viewer   │  │  Schematic │  │  Plots     │ │
│  │  (Canvas)   │  │  (WebGL)   │  │  (Canvas)  │  │  (WebGL)   │ │
│  └──────┬──────┘  └──────┬─────┘  └─────┬─────┘  └─────┬──────┘ │
│         │               │              │              │         │
│         └───────────────┼──────────────┼──────────────┘         │
│                         ▼                                       │
│               State Management (Zustand + WebSocket)             │
│                         │                                       │
└─────────────────────────┼───────────────────────────────────────┘
                          │ HTTPS / WebSocket
                          ▼
                ┌─────────────────────┐
                │   API Gateway       │
                │   (Rust / Axum)     │
                └────┬───────┬────────┘
                     │       │
          ┌──────────┘       └──────────┐
          ▼                             ▼
┌─────────────────┐           ┌─────────────────────┐
│   PostgreSQL    │           │   Simulation Engine   │
│  (Users, Proj,  │           │   (Rust + GPU)        │
│   Components,   │           │                       │
│   S-params)     │           │  ┌─────────────────┐  │
└────────┬────────┘           │  │ FDTD Kernel     │  │
         │                    │  │ (CUDA / wgpu)   │  │
         ▼                    │  └─────────────────┘  │
┌─────────────────┐           │  ┌─────────────────┐  │
│     Redis       │           │  │ Eigenmode Solver│  │
│  (Sessions,     │◄──────────│  │ (Rust + GPU)    │  │
│   Job Queue,    │           │  └─────────────────┘  │
│   Pub/Sub)      │           │  ┌─────────────────┐  │
└─────────────────┘           │  │ Circuit Solver  │  │
                              │  │ (S-matrix)      │  │
         ┌──────────┐        │  └─────────────────┘  │
         │ Optim.   │        │  ┌─────────────────┐  │
         │ Workers  │◄───────│  │ Adjoint Optim.  │  │
         │ (GPU)    │        │  │ (CUDA / wgpu)   │  │
         └──────────┘        │  └─────────────────┘  │
                              └─────────────────────┘
                                        │
                              ┌─────────┴──────────┐
                              ▼                    ▼
                     ┌──────────────┐    ┌───────────────┐
                     │ Object Store │    │  GPU Cluster   │
                     │ (S3 / R2)   │    │  (Kubernetes)  │
                     │ GDS, Fields,│    │  NVIDIA A100   │
                     │ S-params    │    │  Auto-scaling  │
                     └──────────────┘    └───────────────┘
```

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend Framework | React 19 + TypeScript, Vite build system |
| Layout Editor | Custom Canvas2D renderer with R-tree spatial indexing for waveguide routing |
| Field Visualization | WebGL 2.0 with custom shaders for real-time E/H-field rendering and color mapping |
| Circuit Schematic | Custom Canvas2D renderer for photonic circuit topology visualization |
| Eigenmode Plots | WebGL 2.0 for 2D mode profile rendering, Plotly.js for dispersion curves |
| State Management | Zustand for client state, WebSocket for real-time simulation progress streaming |
| Backend API | Rust (Axum) for low-latency simulation job management and API endpoints |
| FDTD Engine | Rust with GPU compute kernels (wgpu for portable, CUDA for NVIDIA-optimized path) |
| Eigenmode Solver | Rust finite-element solver with sparse eigenvalue computation (ARPACK via Rust bindings) |
| Circuit Simulator | Rust S-matrix composition engine with rational polynomial interpolation |
| Optimization | Adjoint FDTD in Rust/GPU, Bayesian optimization via Python (BoTorch) worker |
| Material Database | PostgreSQL JSONB with wavelength-indexed n,k data and Lorentz model coefficients |
| Database | PostgreSQL 16 with JSONB for simulation configurations and S-parameter storage |
| Cache / Queue | Redis 7 for job queuing, simulation progress pub/sub, and session management |
| Object Storage | Cloudflare R2 for GDS files, FDTD field data (HDF5), and S-parameter archives |
| GPU Compute | Kubernetes (EKS) with NVIDIA A100/A10G GPU nodes, Karpenter auto-scaling |
| Auth | JWT-based auth with OAuth (Google, GitHub), SAML for enterprise |
| Monitoring | Grafana + Prometheus, Sentry for errors, custom GPU utilization dashboards |

### Data Model

```
User
├── id (uuid), email, name, avatar_url
├── auth_provider, plan (free/academic/pro/team/enterprise)
├── institution (nullable — university name)
├── gpu_hours_used, gpu_hours_limit
└── created_at, updated_at

Organization
├── id (uuid), name, slug, owner_id → User
├── plan, seat_count, billing_email
└── settings (JSONB — default_pdk, wavelength_units, field_colormap)

Project
├── id (uuid), org_id → Organization
├── name, description, visibility (private/org/public)
├── pdk_id → PDK (nullable), platform (soi/sin/inp/polymer/custom)
├── created_by → User, forked_from → Project (nullable)
└── created_at, updated_at

PDK
├── id (uuid), name (e.g., "SiEPIC"), foundry, platform
├── layer_stack (JSONB — [{layer, material, thickness, z_offset}])
├── design_rules (JSONB — min_width, min_bend_radius, min_gap per layer)
├── waveguide_profiles (JSONB — [{name, width, height, etch_depth, neff}])
├── component_library_url (S3 — pre-characterized S-parameter models)
├── documentation_url, is_open (boolean)
└── created_at

Component
├── id (uuid), project_id → Project
├── name, component_type (waveguide/coupler/resonator/modulator/filter/custom)
├── geometry_params (JSONB — {width, length, gap, radius, ...})
├── layout_data_url (S3 — GDS cell data)
├── sparam_model_url (S3 — Touchstone file, nullable)
├── port_definitions (JSONB — [{name, position, direction, mode}])
└── created_by → User, created_at, updated_at

FDTDSimulation
├── id (uuid), project_id → Project, component_id → Component (nullable)
├── name, dimension (2d/3d)
├── domain (JSONB — {x_span, y_span, z_span, mesh_accuracy, pml_layers})
├── sources (JSONB — [{type, position, wavelength_center, wavelength_span, mode_index}])
├── monitors (JSONB — [{type, position, field_components, frequency_points}])
├── materials (JSONB — [{region, material_name, custom_nk_url}])
├── mesh_override_regions (JSONB — [{region, dx, dy, dz}])
├── status (queued/meshing/running/completed/failed/cancelled)
├── gpu_type, gpu_hours, estimated_cost_usd
├── field_data_url (S3 — HDF5), log_url (S3)
├── progress_percent, current_timestep, total_timesteps
└── created_by → User, started_at, completed_at, created_at

EigenmodeJob
├── id (uuid), project_id → Project
├── name, cross_section (JSONB — geometry and material definition)
├── wavelength_range (JSONB — {start, stop, points})
├── num_modes (int), boundary_conditions (JSONB)
├── status (queued/running/completed/failed)
├── results (JSONB — [{mode_index, neff, ng, loss, confinement}])
├── mode_profiles_url (S3 — field data per mode)
└── created_by → User, created_at, completed_at

SParameterModel
├── id (uuid), component_id → Component, simulation_id → FDTDSimulation (nullable)
├── port_count, frequency_points (int)
├── sparam_data_url (S3 — Touchstone file)
├── metrics (JSONB — {insertion_loss, return_loss, bandwidth_3dB, center_wavelength})
└── created_at

CircuitSimulation
├── id (uuid), project_id → Project
├── name, topology (JSONB — {components: [{id, sparam_id, connections}]})
├── wavelength_range (JSONB), status (queued/running/completed/failed)
├── results_url (S3), monte_carlo_config (JSONB — nullable)
└── created_by → User, created_at, completed_at

OptimizationJob
├── id (uuid), project_id → Project, simulation_id → FDTDSimulation
├── method (adjoint/parametric/topology/bayesian)
├── objective (JSONB), constraints (JSONB), parameter_space (JSONB)
├── status, iterations_completed, best_fom (float), history_url (S3)
└── created_by → User, created_at, completed_at

Layout
├── id (uuid), project_id → Project, name, gds_data_url (S3)
├── components_placed (JSONB), waveguide_routes (JSONB)
├── drc_status (clean/violations), drc_violation_count
└── created_by → User, created_at, updated_at

Comment
├── id (uuid), project_id → Project, author_id → User
├── body (text), anchor_type, anchor_data (JSONB), resolved (boolean)
└── created_at
```

### API Design

```
Auth:
POST   /api/auth/register                     # Create account
POST   /api/auth/login                        # Login
POST   /api/auth/oauth/:provider              # Google/GitHub OAuth
POST   /api/auth/refresh                      # Refresh token

Projects:
GET    /api/projects                           # List projects
POST   /api/projects                           # Create project
GET    /api/projects/:id                       # Get project details
PATCH  /api/projects/:id                       # Update project
DELETE /api/projects/:id                       # Archive project

Components:
GET    /api/projects/:id/components            # List components
POST   /api/projects/:id/components            # Create component
GET    /api/components/:id                     # Get component details
PATCH  /api/components/:id                     # Update component geometry
DELETE /api/components/:id                     # Delete component
GET    /api/components/:id/ports               # Get port definitions

FDTD Simulation:
POST   /api/projects/:id/fdtd                  # Create and submit FDTD simulation
GET    /api/fdtd/:id                           # Get simulation status and config
GET    /api/fdtd/:id/progress                  # Get progress (or via WebSocket)
POST   /api/fdtd/:id/cancel                    # Cancel running simulation
GET    /api/fdtd/:id/fields                    # Get field data for visualization
GET    /api/fdtd/:id/fields/:monitor           # Get specific monitor data
WS     /ws/fdtd/:id                            # Real-time simulation progress stream

Eigenmode:
POST   /api/projects/:id/eigenmode             # Submit eigenmode job
GET    /api/eigenmode/:id                      # Get job status and results
GET    /api/eigenmode/:id/modes                # Get mode profiles
GET    /api/eigenmode/:id/dispersion           # Get neff/ng vs. wavelength

S-Parameters:
GET    /api/components/:id/sparams             # List S-parameter models
POST   /api/fdtd/:id/extract-sparams           # Extract S-params from FDTD
GET    /api/sparams/:id                        # Get S-parameter data
GET    /api/sparams/:id/touchstone             # Download Touchstone file
POST   /api/sparams/:id/fit                    # Fit rational polynomial model

Circuit Simulation:
POST   /api/projects/:id/circuit               # Create circuit simulation
GET    /api/circuit/:id                        # Get circuit results
POST   /api/circuit/:id/monte-carlo            # Run Monte Carlo analysis
GET    /api/circuit/:id/results                # Get transmission/reflection data

Optimization:
POST   /api/projects/:id/optimize              # Start optimization job
GET    /api/optimize/:id                       # Get optimization status
GET    /api/optimize/:id/history               # Get iteration history
POST   /api/optimize/:id/cancel                # Cancel optimization

Layout:
GET    /api/projects/:id/layouts               # List layouts
POST   /api/projects/:id/layouts               # Create layout
GET    /api/layouts/:id                        # Get layout data
PATCH  /api/layouts/:id                        # Update layout
POST   /api/layouts/:id/drc                    # Run DRC
GET    /api/layouts/:id/drc/results            # Get DRC violations
POST   /api/layouts/:id/export/gds             # Export GDS-II

PDK:
GET    /api/pdks                               # List available PDKs
GET    /api/pdks/:id                           # Get PDK details
GET    /api/pdks/:id/components                # Browse PDK component library
POST   /api/pdks                               # Upload custom PDK (enterprise)

Materials:
GET    /api/materials                          # List materials
GET    /api/materials/:id                      # Get n,k data
POST   /api/materials                          # Upload custom material data
GET    /api/materials/:id/fit                  # Get Lorentz model fit

Collaboration:
GET    /api/projects/:id/comments              # List comments
POST   /api/projects/:id/comments              # Add comment
PATCH  /api/comments/:id                       # Edit/resolve comment
POST   /api/projects/:id/share                 # Generate share link

Export:
POST   /api/fdtd/:id/export/hdf5              # Export field data as HDF5
POST   /api/layouts/:id/export/gds             # Export layout as GDS-II
POST   /api/sparams/:id/export/touchstone      # Export S-params as Touchstone
```

---

## UI/UX — Key Screens

### 1. Project Dashboard
- Card grid showing projects with component layout thumbnails, PDK badges, and simulation status indicators (running, completed, failed)
- Quick-start templates: "Ring Resonator", "Directional Coupler", "Grating Coupler", "MZI Modulator", "AWG Demux"
- GPU usage bar showing hours consumed this billing period with cost estimate
- Recent activity feed: simulations completed, S-parameters extracted, layouts exported, comments added

### 2. FDTD Simulation Setup
- 3D geometry viewport (WebGL) showing the simulation domain with material regions color-coded by refractive index
- Left panel: structure tree with material assignments, source placement, and monitor configuration
- Right panel: simulation settings — mesh accuracy slider, PML configuration, wavelength range, and run time
- Bottom panel: estimated GPU time and cost, mesh cell count, memory requirement
- "Run Simulation" button with real-time progress bar showing timestep advancement and field energy convergence
- Live field preview: 2D cross-section of the electromagnetic field updating during simulation via WebSocket

### 3. Eigenmode Solver
- Cross-section editor: draw or import waveguide geometry with material color coding and dimension labels
- Mode gallery: grid of computed mode profiles (|E| intensity) with neff, ng, loss, and confinement factor labels below each mode
- Dispersion plot: neff and ng vs. wavelength with interactive cursors and GVD readout
- Sweep panel: configure width/height/wavelength sweeps and view results as 2D parameter maps with color-coded neff contours
- Overlap integral tool: select two modes and compute coupling efficiency

### 4. Photonic Layout Editor
- Full-screen canvas with layer visibility controls in a collapsible left panel (similar to KLayout but browser-native)
- Component palette on the left: PDK components organized by category with drag-and-drop placement
- Waveguide routing toolbar: draw waveguides with automatic bend insertion (Euler or circular), taper generation, and crossing avoidance
- Right panel: selected component properties (width, gap, radius, port connections), DRC violation list
- Layer stack cross-section view: click any point on the layout to see the vertical material stack at that position
- Minimap showing full-chip overview with viewport indicator

### 5. S-Parameter and Circuit Viewer
- S-parameter plots: magnitude (dB) and phase vs. wavelength for all port combinations with interactive legend toggle
- Key metrics summary: insertion loss, return loss, 3-dB bandwidth, center wavelength, extinction ratio displayed as headline numbers
- Circuit schematic view: photonic circuit topology with labeled components and port connections
- Circuit simulation overlay: transmission through any circuit path plotted alongside individual component S-parameters for comparison
- Monte Carlo results: yield histogram and scatter plots showing performance distribution across fabrication variations

### 6. Field Visualization
- Full-screen WebGL viewer showing electromagnetic field distribution (E-field intensity by default)
- Toolbar: field component selector (Ex, Ey, Ez, |E|^2, Poynting), color map selector, linear/log scale toggle
- Cross-section controls: XY/XZ/YZ plane selector with drag slider for plane position
- Time animation controls: play/pause, speed slider, frame-by-frame stepping for FDTD time-domain playback
- Far-field projection toggle: switch from near-field to computed far-field radiation pattern
- Export buttons: high-resolution PNG screenshot, MP4 animation, raw data (HDF5/VTK)

---

## Monetization

### Free / Academic — $0/month
- Requires .edu or institutional email verification for academic users; limited free tier for all others
- 2D FDTD simulations only (up to 500K mesh cells)
- Eigenmode solver: up to 5 modes, single wavelength
- 3 projects, 5 simulations per month, 2 GPU-hours per month
- Basic field visualization and S-parameter extraction
- Open PDKs only (SiEPIC, AIM Photonics educational)
- Community support, PhotonPath watermark on exports

### Pro — $199/month
- 2D and 3D FDTD simulations (up to 50M mesh cells)
- Eigenmode solver: unlimited modes, wavelength and geometry sweeps
- Unlimited projects, 200 GPU-hours per month included ($0.50/hr overage)
- Full S-parameter extraction with Touchstone export
- Photonic circuit simulation (up to 100 components)
- Parametric optimization (sweep + Bayesian)
- GDS-II export, HDF5/VTK field data export, Python SDK access
- All open PDKs, custom material upload
- Email support with 48-hour response

### Team — $599/month (up to 5 seats, $119/additional seat)
- Everything in Pro
- 1,000 GPU-hours per month included ($0.40/hr overage)
- 3D FDTD up to 500M mesh cells on multi-GPU nodes
- Adjoint and topology optimization
- Photonic circuit simulation (unlimited components, Monte Carlo)
- Custom PDK upload and management
- Collaboration: shared projects, comments, version history
- Priority job queue and dedicated GPU allocation during peak
- Priority support with 24-hour response, onboarding session

### Enterprise — Custom
- Unlimited GPU hours with reserved GPU allocation
- Dedicated compute cluster (private GPU nodes, no multi-tenancy)
- Custom PDK hosting with NDA-protected process data
- On-premise deployment option for semiconductor IP security
- SAML/SSO, audit logging, data residency options
- Custom material model development and validation
- Integration with existing EDA flows (Lumerical, Cadence, Synopsys)
- Dedicated customer success manager and SLA guarantee

---

## Go-to-Market Strategy

### Phase 1: Research Community Seeding (Month 1-3)
- Launch free academic tier and announce on photonics research forums, Photonics Media, SPIE community, and r/photonics
- Partner with 5 university integrated photonics courses (MIT, Stanford, Ghent, DTU, UCSB) for pilot deployment
- Publish benchmark validation studies: ring resonator Q-factor, directional coupler splitting ratio, grating coupler efficiency — comparing PhotonPath results to Lumerical and published experimental data
- Tutorial series: "Design Your First Silicon Photonic Circuit in the Browser" targeting photonics graduate students
- Release Python SDK as open-source to build developer community and enable scripted workflows

### Phase 2: Professional Adoption (Month 3-6)
- Launch Pro tier with full 3D FDTD, optimization, and circuit simulation
- Case studies from beta users: "How [Startup] replaced $200K in Lumerical licenses with PhotonPath"
- Content marketing: SEO targeting "cloud photonic simulation", "FDTD simulation online", "Lumerical alternative", "silicon photonics design tool"
- Partner with photonic foundries (AMF, CompoundTek, LioniX) to offer PhotonPath as a supported design entry tool
- Exhibit at OFC (Optical Fiber Communication), CLEO, and Photonics West conferences with live simulation demos

### Phase 3: Enterprise and Foundry Integration (Month 6-12)
- Launch Team and Enterprise tiers with collaboration, custom PDK hosting, and on-premise deployment
- Build dedicated foundry partnerships: pre-loaded PDKs with characterized component libraries for one-click tape-out preparation
- Hire photonics application engineers for vertical-specific support (datacom, LiDAR, biosensing, quantum)
- Launch PhotonPath Marketplace for community-contributed component designs, optimization recipes, and PDK extensions
- Pursue SOC 2 and ISO 27001 certification for enterprise semiconductor customers

### Acquisition Channels
- Organic search: "photonic FDTD simulation online", "Lumerical alternative", "silicon photonics design tool", "free eigenmode solver"
- Photonics conference presence: OFC, CLEO, Photonics West, ECOC, SPIE Photonics Europe
- University partnerships driving graduates into professional tiers as they enter the photonics industry
- Foundry partnerships: recommended design tool in foundry PDK documentation
- Open-source Python SDK driving developer adoption and community contributions

---

## Success Metrics & KPIs

| Metric | Target (Month 6) | Target (Month 12) |
|--------|-------------------|---------------------|
| Registered users | 4,000 | 20,000 |
| FDTD simulations completed (cumulative) | 15,000 | 100,000 |
| Total GPU hours consumed | 30,000 | 250,000 |
| Paying customers (Pro + Team) | 80 | 350 |
| MRR | $25,000 | $120,000 |
| Free/Academic → Paid conversion | 4% | 7% |
| Average 3D FDTD simulation time | <15 min | <10 min |
| Monthly churn rate | <5% | <3% |

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | PhotonPath Advantage |
|-----------|-----------|------------|---------------------|
| Lumerical FDTD Solutions | Industry gold standard, deep foundry PDK support, Ansys ecosystem integration, validated by thousands of publications | $50K–$100K/seat/year, desktop-only, requires local GPU workstation, license server bottlenecks, no real-time collaboration | 100x cheaper, cloud GPU eliminates local hardware, browser-based access from anywhere, built-in collaboration |
| RSoft (Synopsys) | Strong BPM and eigenmode solvers, integrated with Synopsys EDA flow, good for passive device design | $30K+/seat, dated UI, limited FDTD capability, desktop-only, steep learning curve, bundled licensing | Modern browser UI, superior GPU FDTD, integrated PIC layout, accessible pricing, no installation |
| COMSOL Wave Optics | Excellent multiphysics coupling (thermal + optical + mechanical), flexible FEM formulation, great documentation | $25K+ base + $10K Wave Optics module, general-purpose (not photonics-optimized), slow for large 3D photonic simulations, desktop-only | Purpose-built for photonics, GPU-accelerated FDTD (10-100x faster for photonic structures), PIC layout and circuit simulation included |
| Tidy3D (Flexcompute) | Cloud-based FDTD with fast GPU solver, Python-native API, pay-per-simulation model | No layout editor, no eigenmode solver, no circuit simulation, $2–10 per simulation adds up fast, Python-only (no GUI) | Full design flow (layout + eigenmode + FDTD + circuit), visual GUI in browser, integrated S-parameter extraction, flat monthly pricing |
| Luceda Photonics (IPKISS) | Strong PIC layout and circuit simulation, good foundry PDK integration, Python-scriptable | No FDTD simulation (relies on external S-params), expensive licensing, limited visualization, no field-level simulation | Integrated FDTD for component-level simulation, real-time field visualization, S-parameter extraction built in, unified design-to-simulation flow |

---

## Risks & Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| GPU compute costs make unit economics unsustainable at scale | High | Medium | Use spot/preemptible GPU instances for non-priority jobs (60-70% savings), implement adaptive mesh refinement to reduce cell counts, batch small simulations on shared GPU nodes, tiered pricing with overage billing |
| FDTD accuracy insufficient compared to Lumerical for publication-grade results | High | Medium | Extensive validation against published benchmarks (ring resonator, grating coupler, MMI), publish peer-reviewed validation paper, offer sub-pixel averaging and conformal meshing for high accuracy, provide convergence testing tools |
| Photonics engineers distrust cloud-based simulation for tape-out critical designs | Medium | High | Publish comprehensive validation studies, offer Touchstone S-parameter export for cross-verification in Lumerical, partner with foundries for endorsed PDK support, provide on-premise deployment option for enterprise |
| Large 3D FDTD field datasets (multi-GB) make browser visualization sluggish | Medium | High | Progressive loading with level-of-detail, server-side rendering fallback for very large datasets, stream only visible cross-section data, compress field data with lossy wavelet compression for visualization (lossless for export) |
| Ansys (Lumerical) or Synopsys builds a competing cloud product with existing market dominance | Medium | Medium | Move fast on UX and pricing moat, build strong academic community, offer open S-parameter export (no vendor lock-in), undercut on price by 10-50x, focus on integrated PIC design flow that incumbents lack |

---

## MVP Scope (v1.0)

### In Scope
- User auth, organization and project management
- 2D FDTD simulation with GPU acceleration (up to 2M mesh cells) with mode source and plane wave excitation
- Eigenmode solver for rectangular and rib waveguide cross-sections (neff, ng, mode profiles)
- Basic photonic layout editor with straight and curved waveguide drawing, component placement from built-in library
- S-parameter extraction from 2D FDTD with Touchstone export (2-port)
- Real-time 2D field visualization (E-field intensity, color mapping, time animation) via WebGL
- Built-in material database (Si, SiO2, Si3N4, SiN) for C-band and O-band wavelengths
- SiEPIC PDK with basic design rules and waveguide definitions
- GDS-II export of photonic layouts

### Out of Scope (v1.1+)
- 3D FDTD simulation (v1.1 — highest priority post-MVP)
- Multi-port S-parameter extraction (>2 ports) and circuit simulation (v1.1)
- Adjoint and topology optimization (v1.2)
- Parametric sweeps and batch simulation management (v1.2)
- Collaboration features and shared workspaces (v1.2)
- Additional PDKs (AMF, IMEC, AIM Photonics, LioniX) (v1.3)
- Custom PDK creation and upload (v1.3)
- Monte Carlo yield analysis for circuit simulation (v1.3)
- InP and polymer waveguide platform support (v1.3)
- Enterprise features: SSO, on-premise deployment, audit logging (v2.0)

### MVP Timeline: 10 weeks
- Week 1-2: Auth system, project/org data model, material database with n,k data import, basic WebGL field renderer, S3 storage for simulation data
- Week 3-4: 2D FDTD engine (Rust + wgpu GPU compute kernels), PML boundaries, mode source and plane wave excitation, mesh generation with sub-pixel averaging
- Week 5-6: Eigenmode solver (FEM, rectangular/rib cross-sections), eigenmode results UI (mode gallery, dispersion plot), WebSocket simulation progress streaming
- Week 7-8: Photonic layout editor (waveguide drawing, component placement, SiEPIC PDK integration), GDS-II export, S-parameter extraction from FDTD port monitors
- Week 9-10: Field visualization UI (color maps, cross-sections, time animation), Touchstone export, billing integration (Stripe), free/pro tier enforcement, documentation, performance testing, beta launch
