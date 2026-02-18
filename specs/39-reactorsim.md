# ReactorSim — Cloud Nuclear Reactor Simulation Platform

## Executive Summary

ReactorSim is a cloud-native nuclear reactor simulation platform that provides GPU-accelerated Monte Carlo neutron transport, deterministic Sn (discrete ordinates) solvers, interactive 3D reactor core visualization, thermal-hydraulic subchannel analysis, burnup/depletion tracking, and reactor kinetics modeling — all accessible from the browser. By replacing fragmented, export-controlled, and license-restricted desktop codes (MCNP, Serpent, SCALE, CASMO/SIMULATE) with a unified SaaS platform, ReactorSim enables nuclear engineers, reactor designers, university researchers, and regulatory analysts to design, simulate, and certify reactor cores without managing HPC clusters, navigating government software distribution bureaucracies, or stitching together incompatible legacy Fortran codes.

---

## Problem Statement

**The pain:**
- MCNP, the gold standard for Monte Carlo neutron transport, is export-controlled by the US Department of Energy and requires RSICC distribution approval — a process that takes 3-6 months, excludes many international researchers, and ties licenses to specific machines with no cloud deployment option
- Running a single full-core Monte Carlo simulation on a university HPC cluster takes 8-48 hours of wall-clock time, consuming hundreds of thousands of CPU-hours per design iteration, making rapid design exploration practically impossible
- Nuclear simulation workflows require chaining 4-7 disconnected codes (MCNP for transport, ORIGEN for depletion, RELAP for thermal-hydraulics, PARCS for kinetics, custom scripts for cross-section processing) with brittle file-based coupling and incompatible input formats
- There is no modern visualization for reactor simulations — engineers manually plot pin-by-pin power distributions in Excel or use 1990s-era Fortran plotting utilities, making it nearly impossible to quickly identify hot spots, flux tilts, or assembly loading errors
- Regulatory document preparation (NRC safety analysis reports, Chapter 4 nuclear design documentation) is a manual copy-paste process from simulation output files into Word templates, consuming months of engineering time per license application

**Current workarounds:**
- MCNP6 (Los Alamos) — most trusted Monte Carlo code but export-controlled, CPU-only, command-line interface, no built-in visualization, requires cross-section processing expertise, input deck syntax dating from 1970s punch-card era
- Serpent 2 (VTT Finland) — modern Monte Carlo with built-in burnup, but academic-license-only, no GUI, limited thermal-hydraulics coupling, single-node parallelism limits problem size
- OpenMC (MIT/ANL) — open-source Monte Carlo with Python API, but no graphical interface, no integrated thermal-hydraulics, limited V&V pedigree for regulatory submissions, requires users to build their own workflows
- SCALE/KENO (ORNL) — comprehensive suite with criticality safety and depletion, but monolithic installation, slow execution, dated interface, US-distribution-restricted, steep learning curve across its dozens of sub-modules

**Market size:** The global nuclear simulation and modeling software market is approximately $2.1 billion (2024) and projected to reach $4.8 billion by 2032, driven by 60+ new reactor construction projects worldwide, the emergence of small modular reactors (SMRs), and increased regulatory demand for high-fidelity multi-physics simulations. There are an estimated 50,000+ nuclear engineers globally, 300+ nuclear engineering university programs, and 440+ operating reactors requiring ongoing analysis support.

---

## Target Users

### Primary Personas

**1. Dr. Elena Vasquez — Senior Reactor Physicist at an SMR Startup**
- Leads neutronics design for a novel molten salt small modular reactor, iterating on fuel assembly geometry and enrichment zoning weekly
- Currently waits 2-3 days for each MCNP run on a leased HPC cluster at $0.10/core-hour, spending $15K/month on compute alone with no visualization beyond text-file tallies
- Needs to generate NRC-quality safety analysis documentation but has no automated tooling — her team spends 30% of their time reformatting simulation results into regulatory templates
- Needs: fast GPU-accelerated Monte Carlo with interactive 3D visualization, integrated burnup/depletion, and automated regulatory document generation

**2. Professor Kwame Asante — Nuclear Engineering Department Chair**
- Teaches reactor physics and computational methods courses to 40+ graduate students per year at a major research university
- Students struggle with MCNP input deck syntax (fixed-format columns, cryptic material card numbering) and spend weeks debugging geometry errors that would be obvious in a visual editor
- Cannot use MCNP for international students due to export control restrictions, limiting research collaboration with partner universities in Europe and Asia
- Needs: browser-based simulation platform with visual geometry editor, no export-control restrictions, shared workspaces for student projects, and affordable academic pricing

**3. Sarah Chen — Nuclear Safety Analyst at a Regulatory Consultancy**
- Reviews reactor license applications for utility clients, performing independent confirmatory calculations using SCALE/KENO to verify vendor neutronics analyses
- Spends weeks setting up SCALE input models from scratch for each project, manually cross-referencing vendor design documents with simulation geometry
- Must produce detailed V&V documentation showing code-to-code benchmarks and experimental validation for every calculation submitted to the NRC
- Needs: rapid model setup from CAD/design documents, automated V&V benchmark suite, traceable calculation records with audit trails, and NRC-formatted report generation

### Secondary Personas
- Reactor operations engineers performing reload core design optimization for operating PWR/BWR fleets
- National laboratory researchers developing advanced reactor concepts (fast reactors, high-temperature gas reactors, fusion blankets)
- Nuclear fuel vendors analyzing fuel performance and burnup behavior across multiple reactor types
- Regulatory body staff (NRC, IAEA) performing independent safety assessment calculations

---

## Solution Overview

ReactorSim provides an integrated nuclear simulation environment through five core capabilities:
1. **GPU-accelerated Monte Carlo transport** — continuous-energy and multi-group neutron/photon transport using massively parallel GPU kernels, delivering 10-50x speedup over CPU-based MCNP for full-core reactor models, with real-time convergence monitoring and interactive tally visualization
2. **Deterministic Sn solver** — discrete ordinates (Sn) method for rapid flux calculations on structured and unstructured meshes, enabling fast scoping studies and providing initial source guesses for Monte Carlo calculations, with support for 2D/3D geometries and multi-group cross-section libraries
3. **Interactive 3D reactor visualization** — Three.js-powered 3D reactor core viewer with cross-sectioning planes, pin-by-pin power/flux/burnup overlays, assembly drag-and-drop loading pattern editor, and real-time geometry validation
4. **Multi-physics coupling** — integrated thermal-hydraulic subchannel analysis for coolant temperature, density, and void fraction feedback, coupled with neutronics for realistic power distribution predictions including Doppler and moderator feedback effects
5. **Regulatory automation** — automated generation of NRC-formatted safety analysis reports (FSAR Chapter 4), systematic V&V documentation against ICSBEP/IRPhEP benchmarks, calculation audit trails with full input/output provenance, and review/approval workflows for QA compliance

---

## Core Features

### F1: Monte Carlo Neutron Transport
- GPU-accelerated continuous-energy Monte Carlo neutron and photon transport using CUDA/ROCm compute kernels with event-based particle tracking optimized for GPU warp execution
- Support for both continuous-energy (pointwise cross-sections from ENDF/B-VIII.0, JEFF-3.3, JENDL-5) and multi-group (collapsed libraries) transport modes
- Constructive solid geometry (CSG) with universes, lattices, and repeated structures for efficient full-core reactor modeling with billions of regions
- Configurable tallies: cell flux, surface current, mesh tally (regular and cylindrical), pulse-height, energy deposition (F6/F7 equivalents), with real-time statistical convergence monitoring
- Eigenvalue (k-effective) calculations with Shannon entropy source convergence diagnostics, source distribution visualization, and automatic fission source convergence detection
- Fixed-source mode for shielding, activation, and dose-rate calculations with variance reduction (weight windows, importance sampling, CADIS/FW-CADIS)
- Automatic domain decomposition across multiple GPU nodes for problems exceeding single-GPU memory, with load-balanced particle banking
- Performance target: 100M+ neutron histories per minute on a single NVIDIA A100 for a full-core PWR model (vs. ~5M/min for MCNP on 64 CPU cores)

### F2: Deterministic Sn Transport Solver
- Discrete ordinates (Sn) method with configurable angular quadrature (S4 through S32, level-symmetric and Gauss-Legendre-Chebyshev product quadratures)
- Multi-group energy treatment with standard library support (SCALE 252-group, CASMO 70-group, custom group structures) and self-shielding via Bondarenko method or subgroup parameters
- Structured Cartesian and cylindrical mesh solvers with diamond-difference, step-characteristic, and theta-weighted spatial differencing schemes
- Unstructured tetrahedral mesh solver for complex geometries imported from CAD models via STEP/IGES file formats
- Acceleration methods: diffusion synthetic acceleration (DSA), coarse-mesh finite difference (CMFD), and p-CMFD for rapid convergence of scattering-dominated problems
- Adjoint transport capability for perturbation analysis, sensitivity coefficients, and importance map generation for Monte Carlo variance reduction
- Parallelized with spatial domain decomposition (MPI) and energy group parallelism, scaling to hundreds of GPU cores

### F3: 3D Reactor Core Visualization
- Interactive Three.js-based 3D reactor core model with orbit controls, zoom, and fly-through navigation at 60 FPS for full-core models
- Cross-section planes (axial, radial, arbitrary angle) with real-time cutting and color-mapped scalar field display (flux, power, temperature, burnup)
- Pin-by-pin and assembly-level data overlay: power peaking factors, burnup (GWd/MTU), isotopic concentrations, coolant temperature, displayed as color maps with configurable palettes
- Assembly loading pattern editor: drag-and-drop fuel assemblies into core positions with quarter/octant symmetry enforcement and automatic shuffling pattern visualization
- Geometry validation: real-time visual highlighting of overlapping cells, undefined regions, and material assignment errors before simulation submission
- Animation support: time-lapse burnup evolution, control rod withdrawal sequences, and transient power excursion visualization

### F4: Thermal-Hydraulic Analysis
- Single-phase and two-phase subchannel analysis for PWR, BWR, and CANDU geometries with rod-centered and coolant-centered subchannel models
- Axial nodalization with configurable mesh density, solving mass, momentum, and energy conservation equations with turbulent mixing and void drift models
- Fuel performance coupling: radial temperature distribution through fuel pellet, gap conductance (Ross-Stoute model), and cladding with oxide layer growth
- Coolant property correlations: IAPWS-IF97 for light water, FLiBe/FLiNaK for molten salt, sodium for fast reactors, helium for HTGRs — with user-extensible fluid property tables
- Critical heat flux (CHF) prediction using W-3, EPRI, and Groeneveld lookup table correlations with departure from nucleate boiling ratio (DNBR) margin tracking
- Coupled neutronics-TH iteration: Doppler temperature feedback, moderator density/void feedback, and xenon/samarium equilibrium with Picard or Anderson acceleration convergence
- CFD-grade visualization of coolant velocity vectors, temperature contours, and void fraction distributions in 3D

### F5: Burnup and Depletion Calculation
- Nuclide transmutation tracking for 2000+ nuclides through irradiation and decay using Bateman equation solvers (CRAM — Chebyshev Rational Approximation Method, matrix exponential)
- Burnup step control with predictor-corrector (CE/LI, CE/CE) methods for accurate mid-step flux/cross-section updates
- Isotopic inventory tracking: actinides (U, Pu, Am, Cm chains), fission products (Xe-135, Sm-149, I-131 and all significant FPs), and activation products
- Spent fuel characterization: decay heat (ANS 5.1 standard), neutron/gamma source terms, activity, and isotopic masses at arbitrary cooling times
- Equilibrium cycle search: automatic iteration to find the equilibrium fuel loading pattern and burnup distribution for a given shuffling scheme
- Multi-cycle depletion: track fuel assemblies across multiple operating cycles with inter-cycle shuffle, reconstitution, and fresh fuel insertion
- Branch calculations: automated parametric sweeps across moderator temperature, fuel temperature, boron concentration, and control rod insertion for few-group cross-section generation (for nodal diffusion codes)

### F6: Reactor Kinetics Simulation
- Point kinetics solver with six delayed neutron groups, prompt jump approximation option, and external reactivity insertion schedules for rapid scoping transient analyses
- Spatial kinetics using improved quasi-static (IQS) method coupling the shape function from Sn or diffusion with point kinetics amplitude function
- Reactivity coefficient computation: isothermal temperature coefficient (ITC), moderator temperature coefficient (MTC), Doppler coefficient, void coefficient, boron worth — all with uncertainty propagation
- Control rod worth calculations: integral and differential rod worth curves, stuck rod analysis, ejected rod reactivity insertion for Chapter 15 safety analyses
- Xenon transient modeling: xenon oscillation simulation with spatial xenon redistribution for axial offset control strategy evaluation
- Transient scenarios: control rod ejection, loss of flow, boron dilution, cold water injection, main steam line break — with coupled neutronics-TH response

### F7: Nuclear Data Library Management
- Built-in support for major evaluated nuclear data libraries: ENDF/B-VIII.0 (US), JEFF-3.3 (Europe), JENDL-5 (Japan), TENDL-2023 (TALYS-based)
- Automated cross-section processing from evaluated data: Doppler broadening, resonance self-shielding, and thermal scattering law (S(alpha,beta)) generation for bound moderators
- Multi-group library generation with user-defined energy group structures and weighting spectra (1/E, fission, thermal Maxwellian, or problem-dependent)
- Nuclear data comparison tool: plot cross-sections from different libraries side by side, identify discrepancies, assess impact on k-effective and reaction rates
- Temperature-dependent cross-section interpolation with on-the-fly Doppler broadening for continuous-energy Monte Carlo at arbitrary fuel temperatures
- Covariance data processing for uncertainty quantification: propagate nuclear data uncertainties to k-effective, power distributions, and reactivity coefficients

### F8: Reactor Core Designer
- Visual drag-and-drop core loading pattern editor with quarter-core and octant symmetry options for PWR, BWR, VVER (hexagonal), and CANDU geometries
- Fuel assembly template library: define pin layouts, enrichment zones, burnable absorber patterns, guide tube locations, and gadolinium concentrations
- Parametric design sweeps: automatically generate and evaluate hundreds of loading pattern variations, ranking by cycle length, power peaking, and discharge burnup
- Low-leakage loading pattern optimization using simulated annealing or genetic algorithm with user-defined objective functions (maximize cycle length subject to peaking factor limits)
- Import reactor geometry from CAD files (STEP, IGES) with automatic conversion to simulation-ready CSG or mesh models
- In-core fuel management: track assembly identity, burnup history, and shuffle path across the full plant lifetime with database storage

### F9: Safety Analysis and Transients
- Automated Chapter 15 (FSAR) transient analysis suite: pre-configured scenarios for control rod ejection, loss of coolant, loss of flow, steamline break, and boron dilution events
- Conservative and best-estimate analysis modes with statistical treatment of uncertainties (Wilks' formula for 95/95 tolerance limits)
- Acceptance criteria checking: automatic evaluation against NRC Standard Review Plan limits (DNBR, fuel centerline temperature, reactivity insertion, dose)
- Dose calculation: source term generation, atmospheric dispersion (Gaussian plume), and offsite dose estimation for design basis accidents
- Sensitivity and uncertainty analysis: perturbation theory, adjoint-based sensitivity coefficients, and total Monte Carlo sampling for comprehensive uncertainty budgets
- Automated comparison of results against regulatory limits with pass/fail indication and margin quantification

### F10: Regulatory Document Generation
- NRC-formatted safety analysis report templates: Chapter 4 (Reactor), Chapter 9 (Auxiliary Systems), Chapter 15 (Accident Analyses) with auto-populated tables and figures
- Automatic generation of standard tables: core operating parameters, fuel design limits, reactivity parameters, power distribution summaries, and shutdown margin calculations
- Figure generation: axial power profiles, radial power maps, control rod worth curves, moderator temperature coefficient plots — all publication-quality with NRC-standard formatting
- Traceability: every figure and table linked to the specific simulation run, input parameters, code version, and nuclear data library used — providing complete audit trail
- 10 CFR 50.59 screening support: automated comparison of design changes against existing safety analysis to determine if NRC review is required
- Export to Word/PDF with configurable templates matching utility or vendor document standards

### F11: Collaboration and Review
- Project workspaces with role-based access: modeler (create/run simulations), reviewer (annotate/approve), viewer (read-only), and admin
- Simulation review workflow: submit calculations for independent review with comment threads anchored to specific results, parameters, or geometry regions
- QA-compliant audit trail: every simulation input, code version, nuclear data library, and result is immutably logged with timestamps and user attribution per 10 CFR 50 Appendix B
- Shared model library: organization-wide repository of verified reactor models, material definitions, and analysis templates
- Real-time co-editing of reactor core loading patterns with presence indicators and change tracking
- Version control for simulation inputs with diff visualization showing parameter changes between revisions

### F12: Verification and Validation (V&V)
- Built-in benchmark suite: ICSBEP (International Criticality Safety Benchmark Evaluation Project) handbook cases with pre-built models and reference k-effective values
- IRPhEP (International Reactor Physics Experiment Evaluation Project) benchmarks for power reactor validation including BEAVRS (MIT PWR benchmark) and UAM (Uncertainty Analysis in Modeling) benchmarks
- Automated V&V regression testing: run benchmark suite against each code update with statistical comparison (chi-squared test) against reference solutions
- Code-to-code comparison tool: import results from MCNP, Serpent, OpenMC, or SCALE and generate side-by-side comparison reports with statistical difference metrics
- Convergence verification: automated mesh refinement studies, angular quadrature convergence, and energy group sensitivity to demonstrate solution convergence
- V&V documentation generator: automatically produce NQA-1-compliant software quality assurance records for regulatory submissions

---

## Technical Architecture

### System Diagram

```
Users (Browser)                                      Nuclear Data
     │                                               Libraries
     ▼                                            (ENDF, JEFF, JENDL)
┌─────────────────┐                                      │
│  React Frontend  │                                      ▼
│  ┌────────────┐  │                           ┌──────────────────┐
│  │  Three.js   │  │                           │  Cross-Section   │
│  │  3D Reactor │  │                           │  Processing      │
│  │  Viewer     │  │                           │  Service (Rust)  │
│  └────────────┘  │                           └────────┬─────────┘
│  ┌────────────┐  │                                    │
│  │  Core       │  │                                    ▼
│  │  Designer   │  │                           ┌──────────────────┐
│  │  (2D/3D)    │  │                           │  Processed XS    │
│  └────────────┘  │                           │  Library Store   │
└────────┬────────┘                           │  (S3 / Redis)    │
         │                                     └────────┬─────────┘
         │ HTTPS / WebSocket                            │
         ▼                                              │
┌─────────────────────┐                                 │
│  API Gateway         │                                 │
│  (Rust / Axum)       │─────────────────────────────────┤
│  Auth, Rate Limit,   │                                 │
│  Job Orchestration   │                                 │
└───┬─────┬─────┬─────┘                                 │
    │     │     │                                        │
    ▼     │     ▼                                        │
┌───────┐ │ ┌──────────────┐    ┌──────────────────┐    │
│Postgre│ │ │ Job Queue    │───►│  GPU Compute      │◄───┘
│SQL    │ │ │ (Redis +     │    │  Workers           │
│(Users,│ │ │  Celery/     │    │                    │
│Models,│ │ │  Tokio Tasks)│    │  ┌──────────────┐  │
│Results│ │ └──────────────┘    │  │ Monte Carlo  │  │
│Audit) │ │                     │  │ GPU Kernels  │  │
└───────┘ │                     │  │ (CUDA/ROCm)  │  │
          │                     │  └──────────────┘  │
          ▼                     │  ┌──────────────┐  │
┌──────────────┐                │  │ Sn Solver    │  │
│ Object Store  │               │  │ (Rust + GPU) │  │
│ (S3 / R2)    │◄──────────────│  └──────────────┘  │
│ XS Libs,     │                │  ┌──────────────┐  │
│ Mesh Files,  │                │  │ TH Solver    │  │
│ Results,     │                │  │ (Rust)       │  │
│ Reports      │                │  └──────────────┘  │
└──────────────┘                │  ┌──────────────┐  │
                                │  │ Depletion    │  │
                                │  │ (CRAM Solver)│  │
                                │  └──────────────┘  │
                                └──────────────────────┘
```

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend Framework | React 19 + TypeScript, Vite build system |
| 3D Visualization | Three.js with custom reactor geometry shaders, instanced mesh rendering for pin-level display |
| Core Designer UI | React Flow for loading pattern editor, D3.js for 2D assembly maps and power distribution plots |
| State Management | Zustand for client state, WebSocket (via Axum) for real-time simulation progress streaming |
| Backend API | Rust (Axum) — high-performance async API with tower middleware for auth, rate limiting, tracing |
| Monte Carlo Engine | Rust + CUDA (cuRAND, custom particle tracking kernels), ROCm for AMD GPU support |
| Sn Transport Solver | Rust with GPU-accelerated sweep kernel (CUDA), sparse matrix solvers (cuSPARSE) for diffusion acceleration |
| Thermal-Hydraulics | Rust solver with IAPWS-IF97 steam tables, subchannel conservation equation solver |
| Depletion Solver | Rust implementation of CRAM (Chebyshev Rational Approximation Method) for matrix exponential |
| Nuclear Data Processing | Rust service for ENDF parsing, Doppler broadening (sigma1 method), and multi-group collapse |
| Database | PostgreSQL 16 with JSONB for simulation parameters, audit trail tables with row-level security |
| Job Queue | Redis 7 + Tokio-based async job scheduler for GPU compute orchestration and priority queuing |
| Object Storage | AWS S3 for nuclear data libraries, mesh files, simulation results, and generated reports |
| GPU Compute Infrastructure | Kubernetes (EKS) with NVIDIA A100/H100 node pools, CUDA 12.x, NCCL for multi-GPU |
| Document Generation | Typst for programmatic PDF generation of NRC-formatted reports, Handlebars templates |
| Auth & Compliance | JWT + RBAC, SAML/OIDC for enterprise SSO, immutable audit log (append-only Postgres table) |
| Monitoring | Prometheus + Grafana for GPU utilization and job metrics, Sentry for errors, OpenTelemetry tracing |

### Data Model

```
Organization
├── id (uuid), name, slug, plan (academic/pro/team/enterprise)
├── nqa1_mode (boolean — enables strict QA audit trail)
├── nuclear_data_libraries[] (licensed libraries available)
└── created_at, updated_at

User
├── id (uuid), org_id → Organization
├── email, name, role (admin/modeler/reviewer/viewer)
├── credentials (nuclear engineering certifications, optional)
└── created_at, last_login_at

Project
├── id (uuid), org_id → Organization
├── name, description, reactor_type (PWR/BWR/VVER/SMR/MSR/HTGR/SFR)
├── visibility (private/org/public)
├── qa_level (preliminary/verified/nrc_submittal)
├── created_by → User
└── created_at, updated_at

ReactorModel
├── id (uuid), project_id → Project
├── name, version (int), parent_version_id (nullable)
├── geometry_type (csg/mesh), geometry_data (JSONB or S3 reference)
├── core_layout (JSONB — assembly positions, symmetry, dimensions)
├── materials[] (JSONB — nuclide compositions, densities, temperatures)
├── boundary_conditions (JSONB — vacuum/reflective/white per surface)
└── created_by → User, created_at

FuelAssemblyTemplate
├── id (uuid), org_id → Organization
├── name, reactor_type, lattice_type (square/hex)
├── pin_layout (JSONB — pin types, positions, enrichments)
├── burnable_absorber_config (JSONB — Gd/IFBA/WABA patterns)
├── dimensions (pitch, height, num_pins)
└── created_by → User, created_at

Simulation
├── id (uuid), project_id → Project, model_id → ReactorModel
├── sim_type (eigenvalue/fixed_source/transient/depletion/kinetics)
├── solver (monte_carlo/sn/diffusion/coupled)
├── parameters (JSONB — particles, batches, Sn_order, mesh, tolerances)
├── nuclear_data_library (endfb8/jeff33/jendl5/tendl2023)
├── status (queued/running/converging/completed/failed)
├── gpu_hours_used (float), cost_usd (float)
├── submitted_by → User, submitted_at, completed_at
└── parent_simulation_id (nullable — for restart/continuation)

SimulationResult
├── id (uuid), simulation_id → Simulation
├── result_type (keff/flux_mesh/power_dist/tally/depletion/kinetics)
├── summary_data (JSONB — k-eff±σ, peaking factors, cycle length)
├── full_result_path (S3 — HDF5 mesh tallies, isotopic inventories)
├── convergence_data (JSONB — entropy history, tally relative errors)
└── created_at

DepletionState
├── id (uuid), simulation_id → Simulation
├── burnup_step (int), burnup_mwd_mtu (float), time_days (float)
├── isotopic_inventory_path (S3 — full nuclide vector per region)
├── power_distribution_path (S3 — pin/assembly power at this step)
├── eigenvalue (float), eigenvalue_uncertainty (float)
└── created_at

BenchmarkCase
├── id (uuid), benchmark_suite (ICSBEP/IRPhEP/custom)
├── case_id (string — e.g., "LEU-COMP-THERM-001")
├── reference_keff (float), reference_uncertainty (float)
├── model_path (S3 — pre-built ReactorSim model)
├── description, category (thermal/fast/intermediate, HEU/LEU)
└── source_document

VVResult
├── id (uuid), benchmark_id → BenchmarkCase, simulation_id → Simulation
├── calculated_keff (float), statistical_uncertainty (float)
├── ce_ratio (float — C/E ratio), delta_keff (float)
├── pass_fail (boolean — within 2σ of reference)
└── created_at, run_by → User

RegulatoryDocument
├── id (uuid), project_id → Project
├── doc_type (FSAR_CH4/FSAR_CH15/10CFR5059/TECH_SPEC)
├── template_id, status (draft/review/approved/submitted)
├── content_path (S3 — generated PDF/Word)
├── linked_simulations[] (simulation_ids with traceability)
├── review_comments[] (JSONB — threaded comments)
└── created_by → User, approved_by → User, created_at

AuditLog
├── id (bigint), timestamp, user_id → User
├── action (simulation_submit/result_view/model_edit/approval/export)
├── resource_type, resource_id
├── details (JSONB — parameters changed, values before/after)
└── ip_address, session_id
```

### API Design

```
Auth:
POST   /api/v1/auth/register                     # Create account
POST   /api/v1/auth/login                        # Email/password login
POST   /api/v1/auth/sso                          # SAML/OIDC SSO flow
POST   /api/v1/auth/token/refresh                # Refresh JWT

Projects:
GET    /api/v1/projects                          # List projects
POST   /api/v1/projects                          # Create project
GET    /api/v1/projects/:id                      # Get project details
PATCH  /api/v1/projects/:id                      # Update project
DELETE /api/v1/projects/:id                      # Archive project

Reactor Models:
GET    /api/v1/projects/:id/models               # List reactor models
POST   /api/v1/projects/:id/models               # Create model
GET    /api/v1/models/:id                        # Get model details
PATCH  /api/v1/models/:id                        # Update model geometry/materials
POST   /api/v1/models/:id/validate               # Validate geometry (overlap check)
POST   /api/v1/models/:id/import                 # Import from CAD (STEP/IGES)
GET    /api/v1/models/:id/3d                     # Get 3D mesh for visualization

Fuel Assembly Templates:
GET    /api/v1/assemblies                        # List assembly templates
POST   /api/v1/assemblies                        # Create assembly template
GET    /api/v1/assemblies/:id                    # Get assembly details
PATCH  /api/v1/assemblies/:id                    # Update assembly
POST   /api/v1/assemblies/:id/clone              # Clone and modify

Core Designer:
GET    /api/v1/models/:id/loading-pattern        # Get core loading pattern
PUT    /api/v1/models/:id/loading-pattern        # Update loading pattern
POST   /api/v1/models/:id/optimize               # Run loading pattern optimization
GET    /api/v1/models/:id/symmetry-check         # Verify symmetry compliance

Simulations:
POST   /api/v1/simulations                       # Submit simulation job
GET    /api/v1/simulations/:id                   # Get simulation status
GET    /api/v1/simulations/:id/progress          # Real-time progress (WebSocket upgrade)
POST   /api/v1/simulations/:id/cancel            # Cancel running simulation
GET    /api/v1/simulations/:id/results           # Get result summary
GET    /api/v1/simulations/:id/results/:type     # Get specific result (flux, power, keff)
GET    /api/v1/simulations/:id/convergence       # Get convergence diagnostics
POST   /api/v1/simulations/:id/restart           # Restart from checkpoint

Nuclear Data:
GET    /api/v1/nuclear-data/libraries            # List available libraries
GET    /api/v1/nuclear-data/:lib/nuclides        # List nuclides in library
GET    /api/v1/nuclear-data/:lib/:nuclide/xs     # Get cross-section data (for plotting)
POST   /api/v1/nuclear-data/compare              # Compare XS across libraries

Depletion:
GET    /api/v1/simulations/:id/depletion/steps   # List burnup steps
GET    /api/v1/simulations/:id/depletion/:step   # Get isotopics at burnup step
GET    /api/v1/simulations/:id/depletion/decay-heat  # Decay heat vs. cooling time
GET    /api/v1/simulations/:id/depletion/source-term # Radiation source terms

Benchmarks & V&V:
GET    /api/v1/benchmarks                        # List benchmark cases
POST   /api/v1/benchmarks/:id/run                # Run benchmark case
GET    /api/v1/vv/results                        # List V&V results
POST   /api/v1/vv/report                         # Generate V&V report

Regulatory Documents:
POST   /api/v1/documents/generate                # Generate regulatory document
GET    /api/v1/documents/:id                     # Get document status/download
PATCH  /api/v1/documents/:id/review              # Submit review comments
POST   /api/v1/documents/:id/approve             # Approve document

Collaboration:
GET    /api/v1/projects/:id/activity             # Activity feed
POST   /api/v1/projects/:id/comments             # Add comment
GET    /api/v1/projects/:id/comments             # List comments
POST   /api/v1/simulations/:id/review            # Submit for review
PATCH  /api/v1/simulations/:id/review            # Approve/reject

Audit:
GET    /api/v1/audit/log                         # Query audit trail (admin)
GET    /api/v1/audit/simulation/:id              # Full provenance for a simulation
```

---

## UI/UX — Key Screens

### 1. Reactor Core 3D Viewer (Main Workspace)
- Full-viewport Three.js 3D rendering of the reactor core with pin-level geometry detail, orbit/pan/zoom controls, and configurable background
- Interactive cross-section planes: drag axial and radial cutting planes to reveal internal fuel assembly structure, control rod positions, and reflector regions
- Scalar field overlays: toggle between thermal flux, fast flux, power density, fuel temperature, coolant temperature, and burnup — rendered as color maps on the geometry with a configurable legend
- Assembly/pin selection: click any assembly or fuel pin to display detailed data panel (power, burnup, isotopics, temperature, DNBR)
- Simulation progress overlay: during active runs, display real-time fission source distribution, k-effective convergence plot, and tally statistics in a floating panel

### 2. Core Loading Pattern Editor
- Top-down 2D view of the reactor core showing assembly positions as colored tiles with symmetry lines (quarter/octant) overlaid
- Assembly palette sidebar: list of available fuel assembly types with enrichment, burnable absorber loading, and previous burnup displayed as color-coded badges
- Drag-and-drop assembly placement with automatic symmetric partner updates and constraint checking (no two identical fresh assemblies adjacent)
- Metrics dashboard: live-updating display of cycle length estimate, FdH (radial peaking), Fq (total peaking), boron letdown curve, and discharge burnup distribution as assemblies are placed
- Optimization controls: configure objective function and constraints, launch automated search, and view Pareto front of candidate loading patterns ranked by performance

### 3. Simulation Setup and Monitoring
- Tabbed form interface for configuring simulation parameters: solver type, particle count/batches (MC) or mesh/quadrature (Sn), energy treatment, nuclear data library, thermal-hydraulic coupling options
- Resource estimator: based on model complexity and chosen parameters, displays estimated GPU-hours, wall-clock time, and cost before submission
- Live monitoring dashboard: real-time k-effective convergence trace, Shannon entropy of fission source, tally relative error reduction, GPU utilization, and estimated time remaining
- Batch management: queue multiple simulations (parameter sweeps, branch calculations) with dependency ordering and priority assignment

### 4. Results Analysis Dashboard
- Multi-panel layout: 3D power distribution (left), axial power profile and radial peaking map (right), summary metrics table (bottom)
- Comparison mode: select two simulation results and display delta maps (power difference, flux difference) with statistical significance indicators
- Tally explorer: hierarchical tree of all tallies with click-to-plot functionality — energy spectra, spatial distributions, reaction rates in any scored region
- Export controls: download results as HDF5 (full data), CSV (summary tables), PNG/SVG (publication-quality figures), or generate a regulatory document section

### 5. Nuclear Data Explorer
- Interactive cross-section plotting: select nuclide, reaction type (fission, capture, scatter, total), and library to display energy-dependent cross-section curves
- Multi-library overlay: compare ENDF/B-VIII.0 vs. JEFF-3.3 vs. JENDL-5 cross-sections on the same plot with ratio sub-panel
- Resonance region zoom: high-resolution display of resolved and unresolved resonance regions with Doppler-broadened curves at multiple temperatures
- Impact analysis: for a selected nuclide and reaction, show sensitivity coefficient indicating the effect of cross-section uncertainty on k-effective for the active reactor model

### 6. Regulatory Document Generator
- Template selector: choose document type (FSAR Chapter 4, Chapter 15, Technical Specifications, 10 CFR 50.59 Screening) with NRC/IAEA formatting options
- Content configuration: select which simulations, results, and V&V benchmarks to include, with automatic table and figure population
- Preview pane: live-rendered document preview with pagination, headers/footers, and regulatory boilerplate text
- Review workflow status bar: shows document progression through draft, internal review, comment resolution, approval, and final export stages

---

## Monetization

### Free / Academic Tier
- Single user, 1 project
- Monte Carlo: up to 10M particles per simulation, single GPU, eigenvalue mode only
- Sn solver: up to S8 quadrature, 2D geometries only
- Pre-built teaching models: PWR pin cell, BWR assembly, critical experiment benchmarks
- ENDF/B-VIII.0 nuclear data library only
- Basic 3D visualization (no export)
- 10 simulation runs per month
- Community support and documentation
- Free for verified .edu email addresses (unlimited runs for coursework)

### Pro — $299/month
- Up to 3 projects, 1 user
- Monte Carlo: up to 1B particles per simulation, up to 4 GPUs, eigenvalue + fixed source
- Sn solver: up to S16, 2D and 3D geometries
- Thermal-hydraulic coupling (single-phase subchannel)
- Burnup/depletion calculations (single cycle)
- All nuclear data libraries (ENDF/B-VIII.0, JEFF-3.3, JENDL-5)
- Full 3D visualization with export (PNG, HDF5, CSV)
- Core loading pattern editor
- 100 simulation runs per month, 50 GPU-hours included
- Email support with 48-hour response

### Team — $799/month (up to 5 seats, +$149/seat)
- Unlimited projects, role-based access (modeler/reviewer/viewer)
- Monte Carlo: unlimited particles, up to 16 GPUs, all modes including variance reduction
- Sn solver: up to S32, unstructured mesh support
- Full thermal-hydraulic coupling (two-phase, CHF/DNBR)
- Multi-cycle depletion with equilibrium cycle search
- Reactor kinetics and transient analysis
- V&V benchmark suite (ICSBEP/IRPhEP) with automated regression testing
- Regulatory document generation (FSAR templates)
- QA audit trail (10 CFR 50 Appendix B compliant)
- Collaboration and review workflows
- 500 GPU-hours/month included, $0.50/GPU-hr overage
- Priority support with 24-hour response SLA
- API access

### Enterprise — Custom
- Unlimited users with SAML/OIDC SSO
- Dedicated GPU cluster with guaranteed capacity and SLA
- Custom nuclear data library support (proprietary evaluations)
- On-premises deployment option (Kubernetes Helm chart) for export-controlled work
- NQA-1 compliant QA program documentation
- Custom solver development and consulting
- Dedicated customer success engineer and onboarding
- 99.9% uptime SLA
- Air-gapped deployment for classified applications

---

## Go-to-Market Strategy

### Phase 1: Academic Beachhead (Month 1-3)
- Launch free academic tier targeting nuclear engineering university programs (MIT, Michigan, Georgia Tech, Texas A&M, Imperial College, KTH — the top 20 programs)
- Publish tutorial series: "Monte Carlo Neutron Transport in Your Browser" targeting students frustrated with MCNP input deck syntax
- Submit papers to ANS (American Nuclear Society) Student Conference and PHYSOR (Reactor Physics) conference demonstrating GPU speedup benchmarks
- Open-source the nuclear data processing library (Rust ENDF parser) to build credibility in the nuclear engineering community
- Offer free semester-long classroom licenses with curriculum integration guides for reactor physics and computational methods courses
- Engage r/nuclearengineering, Nuclear Engineering Stack Exchange, and NEA Data Bank user community

### Phase 2: SMR and Advanced Reactor Startups (Month 3-6)
- Launch Pro and Team tiers targeting the 50+ SMR/advanced reactor startups that need neutronics analysis but cannot afford dedicated HPC infrastructure
- Publish V&V benchmark results against MCNP/Serpent/OpenMC on ICSBEP benchmarks to establish computational credibility
- Content marketing: case studies showing 10-50x speedup over MCNP CPU calculations with equivalent accuracy, total cost comparison (ReactorSim vs. HPC cluster + MCNP license)
- Partner with 3-5 SMR companies for pilot programs with engineering support
- Attend ANS Annual Meeting, ANS Winter Meeting, and NRC Regulatory Information Conference (RIC) as exhibitor
- Webinar series: "From Concept to NRC Submittal: Accelerating SMR Design with Cloud Simulation"

### Phase 3: Enterprise and Regulatory (Month 6-12)
- Launch Enterprise tier targeting major nuclear utilities (Exelon, EDF, TEPCO), fuel vendors (Westinghouse, Framatome, GNF), and national laboratories
- Pursue NQA-1 compliance certification for use in safety-related calculations
- Build NRC engagement: present at NRC public meetings, offer platform access to NRC staff for confirmatory analysis
- Partner with reactor vendors for integrated design-to-licensing workflows
- Expand international: IAEA engagement, EU nuclear safety authority relationships, Korean and Japanese utility partnerships
- SOC 2 Type II and FedRAMP authorization for US government and DOE national laboratory use

### Acquisition Channels
- Academic word-of-mouth: free tier for university researchers and students who become advocates when they enter industry
- Nuclear industry conferences: ANS meetings (2x/year, 2000+ attendees), PHYSOR (biennial, 800+ attendees), NRC RIC (3000+ attendees)
- SEO: target "MCNP alternative", "Monte Carlo neutron transport software", "nuclear reactor simulation tool", "reactor core design software"
- Nuclear engineering professional networks: ANS local sections, LinkedIn nuclear engineering groups, Women in Nuclear (WiN)
- DOE SBIR/STTR grants: fund development of advanced simulation capabilities (multi-physics coupling, uncertainty quantification)

---

## Success Metrics & KPIs

| Metric | Target (Month 6) | Target (Month 12) |
|--------|-------------------|---------------------|
| Registered users | 1,500 | 8,000 |
| University programs using platform | 15 | 50 |
| Simulations completed / month | 5,000 | 40,000 |
| Paying customers (Pro + Team) | 25 | 120 |
| MRR | $15,000 | $100,000 |
| Free → Paid conversion rate | 3% | 6% |
| GPU-hours consumed / month | 2,000 | 20,000 |
| V&V benchmark cases validated | 50 | 200 |

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | ReactorSim Advantage |
|-----------|-----------|------------|---------------------|
| MCNP (LANL) | Gold-standard Monte Carlo, 40+ years of V&V pedigree, trusted by NRC for safety analysis | Export-controlled (3-6 month approval), CPU-only (no GPU), command-line only, 1970s input format, no visualization, no integrated depletion | Cloud-accessible (no export control), GPU-accelerated (10-50x faster), modern UI with 3D visualization, integrated multi-physics |
| Serpent 2 (VTT) | Modern Monte Carlo with built-in burnup, CAD import, good documentation, active development | Academic license only (no commercial use without negotiation), single-node parallelism, no GUI beyond basic geometry plotter, limited TH coupling | Commercial-ready licensing, multi-GPU scaling, full browser-based GUI, integrated thermal-hydraulics and regulatory document generation |
| OpenMC (MIT/ANL) | Open-source, Python API, modern C++ codebase, growing community, GPU development underway | No graphical interface, no integrated thermal-hydraulics, limited V&V documentation for regulatory use, requires users to build entire workflow | Complete integrated platform with GUI, built-in TH coupling, V&V suite, regulatory document generation, no assembly required |
| SCALE/KENO (ORNL) | Comprehensive suite (criticality, shielding, depletion, sensitivity), NRC-endorsed, extensive V&V | US-distribution-restricted, slow execution (CPU only), monolithic installation, dated interface, steep learning curve across 50+ modules | Cloud-native (instant access), GPU-accelerated, unified modern interface, international availability without distribution restrictions |
| CASMO/SIMULATE (Studsvik) | Industry standard for LWR core design, fast nodal diffusion solver, trusted for reload design | Proprietary and expensive ($100K+/year), limited to LWR lattice physics and nodal diffusion, no Monte Carlo capability, vendor lock-in | 10x cheaper entry point, Monte Carlo + deterministic solvers, supports advanced reactor types beyond LWR, open nuclear data libraries |

---

## Risks & Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Regulatory acceptance — NRC and international regulators may not accept results from a new code without extensive V&V pedigree | Critical | High | Invest heavily in ICSBEP/IRPhEP benchmark validation from day one, publish V&V reports in peer-reviewed journals (Nuclear Science and Engineering, Annals of Nuclear Energy), engage NRC early through pre-application meetings, offer code-to-code comparison tools against MCNP/SCALE |
| GPU Monte Carlo accuracy concerns — floating-point differences between GPU and CPU implementations may introduce biases that undermine confidence | High | Medium | Implement rigorous bit-reproducibility testing, run GPU vs. CPU comparison suites on every release, publish accuracy comparison papers, use double-precision for critical accumulations, maintain CPU fallback mode for verification |
| Export control and ITAR restrictions — nuclear simulation software may be subject to US export controls (EAR/ITAR), limiting international availability | High | Medium | Engage export control counsel before launch, classify technology under EAR Category 0E521 (publicly available fundamental research exemption), avoid incorporating classified methods, structure code to separate potentially controlled components |
| Cloud security for nuclear design data — reactor design information is commercially sensitive and potentially safeguards-relevant (IAEA) | High | Medium | SOC 2 Type II certification, encryption at rest (AES-256) and in transit (TLS 1.3), customer-managed encryption keys, VPC isolation per organization, on-premises deployment option for sensitive work |
| GPU compute costs erode margins at scale — A100/H100 GPU-hours at $2-4/hr could make large simulations unprofitable at current pricing | Medium | High | Optimize GPU kernels continuously (target >80% occupancy), implement spot/preemptible instance fallback for non-urgent jobs, tiered pricing with GPU-hour quotas, negotiate reserved GPU capacity with cloud providers as volume grows |

---

## MVP Scope (v1.0)

### In Scope
- Monte Carlo eigenvalue solver (GPU-accelerated) for 3D CSG geometry with continuous-energy neutron transport using ENDF/B-VIII.0
- Deterministic Sn solver (S4-S8) for 2D Cartesian geometries with multi-group cross-section libraries
- Interactive 3D reactor core visualization with cross-section planes and flux/power color mapping
- Basic thermal-hydraulic feedback: single-phase coolant temperature and Doppler temperature iteration
- Core loading pattern editor for PWR square lattice assemblies with quarter-core symmetry
- K-effective convergence monitoring with real-time WebSocket progress updates
- 5 ICSBEP benchmark cases pre-built for V&V validation
- User authentication, project management, and basic audit trail
- Result export: HDF5 (full data), CSV (summary), PNG (figures)

### Out of Scope (v1.1+)
- Burnup/depletion calculations (v1.1 — highest priority post-MVP)
- Two-phase thermal-hydraulics and CHF/DNBR analysis (v1.1)
- Reactor kinetics and transient analysis (v1.2)
- Hexagonal geometry support for VVER and SFR (v1.2)
- Regulatory document generation (FSAR templates) (v1.2)
- Full V&V benchmark suite (50+ ICSBEP + IRPhEP cases) (v1.3)
- Loading pattern optimization (simulated annealing / GA) (v1.3)
- Collaboration, review workflows, and QA audit trail (v1.3)
- Enterprise features: SSO, on-premises deployment, NQA-1 compliance (v2.0)
- Advanced variance reduction (CADIS/FW-CADIS) and shielding analysis (v2.0)

### MVP Timeline: 10 weeks
- Week 1-2: Monte Carlo engine — GPU particle tracking kernel (CUDA), CSG geometry engine with universe/lattice support, ENDF/B-VIII.0 cross-section loader, k-effective eigenvalue iteration with Shannon entropy convergence
- Week 3-4: Sn solver and nuclear data — 2D discrete ordinates solver with diamond-difference spatial discretization, multi-group cross-section processing, DSA acceleration, benchmark validation against analytical solutions
- Week 5-6: 3D visualization and core designer — React + Three.js reactor viewer with instanced fuel pin rendering, cross-section cutting planes, color-mapped scalar field overlays, 2D loading pattern editor with assembly drag-and-drop
- Week 7-8: Thermal-hydraulics coupling and results — single-phase subchannel solver, coupled neutronics-TH Picard iteration, results analysis dashboard with power distribution plots, axial profiles, and summary metrics
- Week 9-10: Auth, infrastructure, and polish — user/project management, job queue and GPU scheduling, result export (HDF5/CSV/PNG), ICSBEP benchmark models, Stripe billing integration, documentation, beta launch to 5 university partners
