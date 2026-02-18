# VoltVault — Cloud Battery Simulation & Electrochemistry Platform

## Executive Summary

VoltVault is a cloud-native battery simulation platform that gives electrochemists, cell engineers, and battery researchers access to GPU-accelerated pseudo-2D (P2D) and single particle model (SPM) solvers, 3D thermal modeling, cycling data management, design-of-experiments parameter sweeps, and electrochemical impedance spectroscopy (EIS) fitting — all from the browser. By replacing fragmented workflows across COMSOL ($50K+/seat), MATLAB/Simulink ($10K+), and command-line-only tools like PyBaMM, VoltVault delivers a unified, collaborative environment where teams can design cells, simulate performance, fit experimental data, and predict degradation without installing desktop software or provisioning GPU workstations.

---

## Problem Statement

**The pain:**
- COMSOL Multiphysics Battery Design Module costs $50,000+ per seat per year, putting physics-based battery simulation out of reach for startups, small labs, and universities in developing countries
- PyBaMM is free and powerful but has no graphical interface — users must write Python scripts, manage Conda environments, and debug solver convergence manually, creating a 3–6 month learning curve for non-programmers
- MATLAB/Simulink requires $10K+ in licenses (base + Simscape Electrical + Optimization Toolbox) and runs only on local machines, making large parameter sweeps painfully slow without dedicated HPC infrastructure
- Battery cycling test data (voltage, current, temperature vs. time from Arbin, Neware, Maccor cyclers) lives in scattered CSV files, proprietary formats, and Excel spreadsheets with no standardized storage, search, or comparison workflow
- Teams working across cell design, modeling, and testing have no shared platform — simulation results live on one engineer's laptop, cycling data on a shared drive, and EIS fits in someone's MATLAB script, making collaboration and reproducibility nearly impossible

**Current workarounds:**
- Using COMSOL on shared license servers with queue times of hours or days, forcing engineers to run simplified models overnight and hope for convergence by morning
- Writing custom PyBaMM scripts for each simulation study, copying code between Jupyter notebooks, and losing track of parameter sets and results across dozens of unversioned files
- Fitting EIS data manually in ZView ($3K+/license) or EC-Lab, then re-entering equivalent circuit parameters into a separate simulation tool with no automated linkage
- Exporting cycling data from proprietary cycler software (Arbin MITS, Neware BTS) into CSV, cleaning it in Excel, and plotting in Origin or MATLAB — a multi-hour process repeated for every batch of cells

**Market size:** The global battery simulation software market was valued at $1.8B in 2024 and is projected to reach $4.5B by 2030, driven by the explosive growth of electric vehicles, grid storage, and consumer electronics. There are approximately 80,000 battery engineers and electrochemists worldwide, with an additional 200,000+ materials scientists and chemical engineers who work on energy storage research. The EV battery market alone is expected to exceed $300B by 2030, creating enormous demand for simulation tools that accelerate cell development cycles.

---

## Target Users

### Primary Personas

**1. Dr. Sarah Chen — Battery Cell Engineer at an EV Startup**
- Leads a 6-person cell design team developing silicon-graphite anode cells for a Series B EV startup
- Currently uses COMSOL (2 licenses shared among 6 engineers) for P2D simulations and MATLAB for post-processing cycling data from their Arbin cyclers
- Spends 40% of her time waiting for COMSOL license availability, manually setting up parameter sweeps, and copying results between tools
- Needs: a unified platform where she can run P2D simulations with GPU acceleration, manage cycling data from the lab, and sweep electrode design parameters without license bottlenecks

**2. Prof. Marcus Okonkwo — Electrochemistry Researcher at a National Lab**
- Runs a research group of 12 graduate students studying solid-electrolyte interphase (SEI) growth and lithium plating degradation mechanisms
- Uses PyBaMM for electrochemical modeling but spends excessive time helping students debug Python environments, solver convergence issues, and visualization scripts
- Publishes 8–10 papers per year and needs reproducible simulation results with proper versioning and easy figure export
- Needs: a browser-based interface to PyBaMM-class solvers that students can use without Python expertise, with built-in experiment tracking and publication-quality plotting

**3. Yuki Tanaka — Test Engineer at a Battery Module Manufacturer**
- Manages cycling test operations across 200+ channels on Neware and Maccor cyclers, testing cells for a Tier 1 automotive supplier
- Currently stores cycling data in a mix of proprietary formats and CSV exports on a NAS, using a homegrown Python script to calculate capacity fade and resistance growth
- Needs to compare experimental cycling data against simulation predictions to validate incoming cell specifications from suppliers
- Needs: centralized cycling data management with automatic parsing of cycler formats, degradation metric extraction, and overlay of simulation predictions against experimental measurements

### Secondary Personas
- Materials scientists at cathode/anode material companies who need quick screening simulations for new active material candidates
- Battery management system (BMS) engineers who need reduced-order models (SPM, ECM) for state-of-charge and state-of-health estimation algorithms
- University professors teaching electrochemistry courses who need accessible simulation tools for student assignments and lab exercises
- Venture capital analysts performing technical due diligence on battery startups who need to validate claimed cell performance metrics

---

## Solution Overview

VoltVault is a browser-based battery simulation platform that:
1. Provides GPU-accelerated P2D (Newman model) and SPM electrochemical solvers that run full discharge/charge simulations in seconds instead of minutes, with a visual model builder that eliminates the need to write solver scripts
2. Offers 3D thermal modeling coupled to electrochemical heat generation, enabling engineers to visualize temperature distributions across cell geometries (pouch, cylindrical, prismatic) directly in the browser using Three.js
3. Manages battery cycling test data with automatic parsing of Arbin, Neware, Maccor, and Biologic formats, providing standardized storage, search, comparison, and degradation metric extraction across thousands of cells
4. Enables design-of-experiments parameter sweeps across electrode parameters (thickness, porosity, particle size, electrolyte concentration, loading) with automatic job scheduling, results aggregation, and response surface visualization
5. Fits electrochemical impedance spectroscopy data to equivalent circuit models (Randles, Warburg, CPE elements) with nonlinear least-squares optimization and links fitted parameters directly to physics-based simulation inputs

---

## Core Features

### F1: P2D (Pseudo-2D) Battery Model Solver
- Full Newman model implementation: solid-phase diffusion (Fick's law in spherical coordinates), electrolyte-phase diffusion and migration (concentrated solution theory), Butler-Volmer kinetics, and charge conservation in both solid and electrolyte phases
- GPU-accelerated PDE solver using Rust compute kernels compiled to CUDA/Metal, achieving 10–50x speedup over CPU-based solvers for standard discharge simulations
- Visual model builder: select cell chemistry (NMC/LFP/NCA/LCO cathode, graphite/silicon/LTO anode), configure geometry (electrode thicknesses, separator thickness, particle radii), and set operating conditions (C-rate, voltage cutoffs, temperature) through a form-based UI
- Built-in materials database with validated parameters for 30+ electrode and electrolyte materials sourced from published literature, with full citation tracking
- Support for multi-layer electrode stacks and graded electrodes with spatially varying porosity or particle size
- Real-time solution visualization: concentration profiles, potential distributions, and reaction rate distributions animated across the cell sandwich during simulation
- Automatic convergence monitoring with adaptive time-stepping and solver diagnostics when convergence fails
- Export simulation results as CSV, HDF5, or NumPy arrays for further analysis

### F2: Single Particle Model (SPM) Solver
- Reduced-order SPM and SPMe (with electrolyte dynamics) solvers for fast screening and BMS algorithm development
- Sub-second solve times for full charge/discharge cycles, enabling interactive parameter exploration with immediate visual feedback
- Transfer function approach for linearized SPM enabling frequency-domain analysis and direct connection to impedance predictions
- Side reaction modules: SEI growth (solvent diffusion model), lithium plating (irreversible and reversible), and active material loss
- State estimator mode: run SPM with Kalman filter or particle filter for SOC/SOH estimation algorithm prototyping
- Comparison view: overlay SPM and P2D results side-by-side to assess reduced-order model accuracy for a given parameter set
- Parameter identification: fit SPM parameters to experimental cycling data using gradient-based optimization
- Export reduced-order model as C code or Python module for embedded BMS deployment

### F3: 3D Thermal Modeling
- Finite-element thermal solver for pouch, cylindrical (18650, 21700, 4680), and prismatic cell geometries with tab and current collector heat paths
- Heat generation coupling: volumetric heat source computed from P2D/SPM electrochemical model (reversible entropic heat + irreversible ohmic and kinetic heat)
- Anisotropic thermal conductivity support for wound electrode stacks (through-plane vs. in-plane conductivity ratios)
- Three.js-based 3D visualization with interactive temperature contour plots, cross-section slicing, and time-animation of thermal transients
- Boundary condition editor: specify convective cooling (h coefficient per face), conductive contact with cooling plates, or prescribed temperature
- Tab design optimization: visualize current density distribution and temperature hotspots for different tab configurations (single-tab, dual-tab, multi-tab, tabless)
- Thermal runaway initiation modeling: identify conditions where local temperature exceeds onset temperature for exothermic decomposition reactions
- Export temperature field data as VTK files for further analysis in ParaView

### F4: Cycling Data Management
- Universal data importer: automatic parsing of Arbin (.res, .csv), Neware (.nda, .xlsx), Maccor (.txt, .raw), Biologic (.mpr, .mpt), and generic CSV with column mapping wizard
- Centralized database storing voltage, current, temperature, capacity, and energy vs. time for every cycle of every cell, with full metadata (cell ID, chemistry, formation protocol, test date, operator)
- Automatic cycle detection and extraction of per-cycle metrics: discharge capacity, charge capacity, coulombic efficiency, energy efficiency, median voltage, DC resistance (from pulse profiles)
- Interactive cycling data viewer: overlay voltage curves from multiple cells or cycles, zoom into specific regions, toggle between voltage-vs-capacity and voltage-vs-time views
- Capacity fade and resistance growth trend plots with automatic curve fitting (linear, power-law, square-root-of-time models)
- Batch comparison: select cells by chemistry, formation protocol, or test condition and compare degradation trajectories across hundreds of cells simultaneously
- Search and filter: query cells by chemistry, capacity range, cycle count, test date, or any metadata tag
- Data quality checks: flag anomalous cycles (voltage spikes, current dropouts, temperature excursions) automatically

### F5: DOE (Design of Experiments) Parameter Sweeps
- Define sweep parameters from any P2D/SPM model input: electrode thickness, porosity, particle radius, active material loading, electrolyte concentration, separator thickness, C-rate, temperature
- Sampling methods: full factorial, fractional factorial (Taguchi), Latin hypercube, Sobol sequences, and custom manual grids
- Automatic job generation and parallel execution on GPU cluster with progress dashboard showing completed/running/queued simulations
- Response surface visualization: 2D heatmaps and 3D surface plots of output metrics (energy density, power density, capacity retention at cycle N) as functions of swept parameters
- Sensitivity analysis: Sobol indices and Morris screening to rank parameter importance and identify dominant design variables
- Multi-objective optimization: Pareto front visualization for trade-offs (e.g., energy density vs. fast-charge capability vs. cycle life)
- Constraint handling: specify feasibility bounds (e.g., maximum electrode thickness for coating processability, minimum porosity for electrolyte wetting)
- Export DOE results as structured datasets for external statistical analysis in JMP, Minitab, or R

### F6: EIS (Electrochemical Impedance Spectroscopy) Fitting
- Import EIS data from Biologic (.mpr), Gamry (.dta), Solartron (.z), and generic CSV (frequency, Z_real, Z_imag)
- Equivalent circuit model library: Randles circuit, Randles + Warburg, dual-RC (cathode + anode), transmission line models, and custom circuit builder with series/parallel element composition
- Circuit elements: resistor (R), capacitor (C), inductor (L), constant phase element (CPE), Warburg (finite/semi-infinite), Gerischer, and Havriliak-Negami
- Nonlinear least-squares fitting (Levenberg-Marquardt and differential evolution) with automatic initial guess estimation based on Nyquist plot features
- Interactive Nyquist and Bode plot visualization with real-time overlay of fitted model and residual plots
- Distribution of relaxation times (DRT) analysis for deconvolving overlapping semicircles without assuming a circuit model
- Multi-spectrum fitting: fit EIS data collected at different SOC, temperature, or cycle number simultaneously with shared and independent parameters
- Link fitted parameters (R_ct, D_s, R_SEI) directly to P2D/SPM model inputs for simulation-experiment correlation

### F7: Battery Cell Designer
- Visual cell geometry builder: define electrode stack (cathode/separator/anode layers), current collector foils, and cell housing (pouch, cylindrical can, prismatic case)
- Automatic calculation of cell-level metrics from electrode parameters: theoretical capacity (mAh), energy density (Wh/kg, Wh/L), N/P ratio, electrolyte volume fraction
- Electrode balancing tool: given a cathode material and target capacity, calculate required anode thickness, loading, and porosity for optimal N/P ratio
- Materials selector with searchable database: active materials (NMC111/523/622/811, LFP, NCA, LCO, graphite, SiO_x, LTO), binders (PVDF, CMC/SBR), conductive additives (carbon black, CNT, graphene), electrolytes (LP30, LP57, custom formulations), and separators (PE, PP, ceramic-coated)
- Electrode recipe editor: specify active material weight fraction, binder content, conductive additive loading, and calculate resulting porosity and electronic conductivity
- Ragone plot generator: simulate the cell across C-rates from C/20 to 10C and plot energy vs. power performance
- Cost estimation: calculate $/kWh at the cell level based on material prices and electrode processing assumptions
- Export cell design as a parameter set ready for P2D/SPM simulation

### F8: Degradation Modeling
- Physics-based degradation mechanisms: SEI layer growth (solvent diffusion-limited), lithium plating and stripping (with reversibility fraction), active material loss (particle cracking, dissolution), and loss of electrolyte
- Calendar aging model: capacity fade and impedance growth as functions of temperature, SOC, and storage time
- Cycle aging model: coupled electrochemical-degradation simulation over hundreds of cycles with adaptive cycle skipping for computational efficiency
- Knee-point detection: automatically identify the onset of accelerated degradation in capacity fade trajectories
- Degradation parameter calibration: fit degradation model parameters to experimental cycling data using Bayesian optimization
- Lifetime prediction: extrapolate degradation trajectories to end-of-life criteria (e.g., 80% capacity retention) with confidence intervals
- What-if scenarios: compare degradation under different operating conditions (temperature, C-rate, depth of discharge, voltage window)
- Degradation mode analysis: decompose total capacity loss into contributions from each mechanism (LLI, LAM_PE, LAM_NE) using half-cell fitting

### F9: Battery Pack Simulation
- Pack topology builder: define series/parallel cell configurations (e.g., 96s4p for EV pack) with visual wiring diagram
- Cell-to-cell variation modeling: assign statistical distributions to cell parameters (capacity, resistance) and simulate pack-level performance with inhomogeneous cells
- Thermal network model: lumped thermal nodes for each cell with inter-cell heat transfer, coolant loop modeling (liquid or air), and cold plate geometry
- BMS algorithm integration: simulate pack with SOC balancing algorithms, over-voltage/under-voltage protection, and thermal management control logic
- Drive cycle simulation: apply standard profiles (WLTP, US06, UDDS) or custom load profiles and analyze pack response including usable energy, peak power, and thermal uniformity

### F10: Experiment Tracker
- Structured experiment logging: define hypothesis, simulation parameters, results, and conclusions for each computational study
- Automatic capture of all simulation inputs and outputs with immutable versioning — every run is reproducible
- Tagging and categorization: organize experiments by project, chemistry, cell format, or custom labels
- Comparison dashboard: select any set of past experiments and generate side-by-side tables and overlay plots
- Notebook-style annotations: attach rich-text notes, equations (LaTeX rendering), and images to any experiment
- Audit trail: full history of who ran what, when, with what parameters — essential for patent documentation and regulatory submissions

### F11: Collaboration and Sharing
- Project-based workspaces with role-based access control (owner, editor, viewer, commenter)
- Share simulation results via public link: interactive plots and 3D thermal views accessible without an account
- Real-time co-editing of cell designs and DOE configurations with cursor presence indicators
- Team materials database: shared custom materials with validated parameters and provenance documentation
- Integration with Slack and Microsoft Teams for job completion notifications and result sharing
- Activity feed showing all team actions: simulations run, data uploaded, experiments completed

### F12: Reporting and Export
- One-click PDF report generation with customizable templates: executive summary, cell design parameters, simulation results, experimental data comparison, degradation analysis
- Publication-quality figure export: SVG, PNG (300+ DPI), and EPS formats with configurable axis labels, fonts, and color schemes matching journal requirements
- Automated comparison reports: side-by-side simulation vs. experimental data with error metrics (RMSE, MAE, R-squared)
- Data export: CSV, HDF5, JSON, and MATLAB .mat formats for all simulation results and cycling data
- PowerPoint export: generate slide decks with key figures and summary tables for design review meetings
- Batch export: generate reports for all cells in a DOE study with a single click

---

## Technical Architecture

### System Diagram

```
                    ┌───────────────────────────────────────────┐
                    │         React Frontend (SPA)               │
                    │  Cell Designer │ 3D Thermal │ Cycling Plots│
                    │    Three.js    │  Recharts  │  EIS Viewer  │
                    └──────────────────┬────────────────────────┘
                                       │ HTTPS / WebSocket
                                       ▼
                    ┌───────────────────────────────────────────┐
                    │        FastAPI Backend (Python)             │
                    │  Auth │ Projects │ Jobs │ Data │ Materials │
                    └──┬──────────┬──────────┬────────────────┘
                       │          │          │
              ┌────────┘     ┌────┘     ┌────┘
              ▼              ▼          ▼
      ┌───────────┐   ┌──────────┐   ┌─────────────────────┐
      │ PostgreSQL │   │  Redis   │   │  S3 (MinIO)         │
      │ (projects, │   │ (queue,  │   │ (cycling data,      │
      │  cells,    │   │  pubsub, │   │  simulation results,│
      │  materials,│   │  cache)  │   │  thermal meshes,    │
      │  users)    │   │          │   │  EIS data)          │
      └───────────┘   └────┬─────┘   └─────────────────────┘
                            │
               ┌────────────┼────────────────┐
               ▼            ▼                ▼
       ┌─────────────┐ ┌──────────────┐ ┌───────────────────┐
       │ P2D/SPM     │ │ Thermal      │ │ EIS Fitting       │
       │ Solver      │ │ Solver       │ │ Workers           │
       │ Workers     │ │ Workers      │ │ (Python/SciPy     │
       │ (Rust/CUDA  │ │ (Rust/FEM    │ │  Levenberg-       │
       │  PDE cores) │ │  kernels)    │ │  Marquardt)       │
       └─────────────┘ └──────────────┘ └───────────────────┘
               │            │                │
               └────────────┼────────────────┘
                            ▼
               ┌────────────────────────────┐
               │    Kubernetes (EKS/GKE)    │
               │    GPU node pools          │
               │    (A100/T4 instances)     │
               │    Auto-scaling per job    │
               │    queue depth             │
               └────────────────────────────┘
```

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | React 18 + TypeScript, Vite build system, Tailwind CSS, Zustand (state management) |
| 3D Thermal Visualization | Three.js with custom WebGL shaders for temperature contour rendering |
| Charting / Plots | Recharts for cycling data plots, D3.js for Nyquist/Bode EIS plots and Ragone charts |
| Backend API | Python 3.12, FastAPI, Pydantic v2, Uvicorn, SQLAlchemy 2.0 |
| PDE Solvers (P2D/SPM) | Rust with CUDA bindings (cuBLAS, cuSPARSE) for GPU-accelerated finite-difference and finite-volume methods |
| Thermal FEM Solver | Rust with nalgebra-sparse for sparse matrix assembly, GPU offload for large meshes |
| EIS Fitting Engine | Python (SciPy optimize, lmfit) with impedance.py for equivalent circuit model definitions |
| Materials Database | PostgreSQL JSONB with curated property tables (diffusivity, conductivity, OCP curves as interpolation arrays) |
| Cycling Data Parser | Python parsers for Arbin/Neware/Maccor/Biologic proprietary formats, Apache Arrow for columnar storage |
| Database | PostgreSQL 16 with TimescaleDB extension for time-series cycling data |
| Queue / Realtime | Redis 7, Celery (distributed task queue), WebSocket via FastAPI for live solver progress |
| Object Storage | AWS S3 for simulation results, cycling data files, thermal mesh VTK exports |
| GPU Compute | Kubernetes with NVIDIA GPU Operator, A100/T4 node pools, Karpenter auto-scaling |
| Auth | JWT tokens with OAuth (Google, GitHub, ORCID for researchers), SAML for enterprise SSO |
| Monitoring | Prometheus + Grafana (cluster metrics), Sentry (error tracking), PostHog (product analytics) |

### Data Model

```
Organization
├── id, name, slug, plan, created_at
│
├── Project[]
│   ├── id, org_id, name, description, chemistry_tag
│   ├── status (active/archived), created_at, updated_at
│   │
│   ├── CellDesign[]
│   │   ├── id, project_id, name, cell_format (pouch/cylindrical/prismatic)
│   │   ├── cathode_material_id → Material, anode_material_id → Material
│   │   ├── electrolyte_id → Electrolyte, separator_id → Separator
│   │   ├── electrode_params (JSONB: thickness, porosity, particle_radius, loading, binder_fraction)
│   │   ├── cell_geometry (JSONB: width, height, depth, num_layers, tab_config)
│   │   ├── calculated_metrics (JSONB: capacity_mAh, energy_Wh_kg, energy_Wh_L, NP_ratio)
│   │   └── created_by → User, created_at, updated_at
│   │
│   ├── Simulation[]
│   │   ├── id, project_id, cell_design_id, name
│   │   ├── model_type (p2d/spm/spme/thermal/pack)
│   │   ├── operating_conditions (JSONB: c_rate, v_min, v_max, temperature, protocol)
│   │   ├── solver_config (JSONB: mesh_points, time_step, tolerances, max_iterations)
│   │   ├── degradation_config (JSONB: sei_model, plating_model, lam_model, nullable)
│   │   ├── status (queued/running/completed/failed/cancelled)
│   │   ├── results_url (S3), log_url (S3)
│   │   ├── runtime_seconds, gpu_hours_consumed
│   │   └── created_by → User, started_at, completed_at, created_at
│   │
│   ├── DOEStudy[]
│   │   ├── id, project_id, base_simulation_id
│   │   ├── sweep_parameters (JSONB: [{name, type, min, max, steps}])
│   │   ├── sampling_method (full_factorial/lhs/sobol/taguchi)
│   │   ├── objective_metrics (JSONB: [energy_density, cycle_life, fast_charge_time])
│   │   ├── status (generating/running/completed)
│   │   ├── total_runs, completed_runs
│   │   ├── simulation_ids[] (FK array)
│   │   └── results_summary (JSONB: sensitivity_indices, pareto_front), created_at
│   │
│   ├── CyclingDataset[]
│   │   ├── id, project_id, cell_id_label, cycler_type (arbin/neware/maccor/biologic)
│   │   ├── raw_file_url (S3), parsed (boolean)
│   │   ├── metadata (JSONB: chemistry, formation_protocol, test_date, operator, notes)
│   │   ├── total_cycles, total_time_hours
│   │   ├── cycle_metrics (JSONB: [{cycle_num, discharge_cap, charge_cap, CE, dcr}])
│   │   ├── timeseries_url (S3, Arrow/Parquet format)
│   │   └── created_by → User, uploaded_at
│   │
│   ├── EISMeasurement[]
│   │   ├── id, project_id, cell_id_label
│   │   ├── conditions (JSONB: soc, temperature, cycle_number)
│   │   ├── frequency_data_url (S3: freq, z_real, z_imag arrays)
│   │   ├── source_format (biologic/gamry/solartron/csv)
│   │   └── created_by → User, measured_at, uploaded_at
│   │
│   ├── EISFit[]
│   │   ├── id, measurement_id, circuit_model (JSONB: topology string)
│   │   ├── fitted_parameters (JSONB: [{element, param, value, error}])
│   │   ├── fit_quality (JSONB: chi_squared, rmse, r_squared)
│   │   ├── algorithm (levenberg_marquardt/differential_evolution)
│   │   └── created_by → User, created_at
│   │
│   └── Experiment[]
│       ├── id, project_id, name, hypothesis
│       ├── linked_simulations[] (FK array), linked_datasets[] (FK array)
│       ├── tags (JSONB: [string]), status (draft/completed/reviewed)
│       ├── notes (rich text / markdown)
│       └── created_by → User, created_at, updated_at
│
├── Material[]
│   ├── id, org_id (null for global library), name, type (cathode/anode/electrolyte/separator)
│   ├── properties (JSONB: density, specific_capacity, diffusivity, conductivity, particle_size)
│   ├── ocp_data (JSONB: [{soc, voltage}] for open-circuit potential curve)
│   ├── kinetic_params (JSONB: exchange_current_density, charge_transfer_coeff, activation_energy)
│   ├── source_reference (text: DOI or publication citation)
│   ├── is_public (boolean), verified (boolean)
│   └── created_by → User, created_at
│
└── User[]
    ├── id, org_id, email, name, role (admin/editor/viewer)
    ├── gpu_hours_used, gpu_hours_limit
    ├── orcid_id (nullable, for researchers)
    └── created_at
```

### API Design

```
Auth:
POST   /api/auth/register                        # Create account
POST   /api/auth/login                           # Login (email/password)
POST   /api/auth/oauth/{provider}                # Google/GitHub/ORCID OAuth
POST   /api/auth/refresh                         # Refresh JWT token

Projects:
GET    /api/projects                              # List projects
POST   /api/projects                              # Create project
GET    /api/projects/:id                          # Get project details
PATCH  /api/projects/:id                          # Update project
DELETE /api/projects/:id                          # Archive project

Cell Designs:
POST   /api/projects/:id/cell-designs             # Create cell design
GET    /api/projects/:id/cell-designs             # List cell designs
GET    /api/cell-designs/:id                      # Get cell design details
PATCH  /api/cell-designs/:id                      # Update cell design parameters
POST   /api/cell-designs/:id/calculate-metrics    # Recalculate cell-level metrics
POST   /api/cell-designs/:id/ragone               # Generate Ragone plot data
POST   /api/cell-designs/:id/cost-estimate        # Calculate $/kWh estimate

Materials:
GET    /api/materials                              # Search materials (filterable by type, chemistry)
GET    /api/materials/:id                          # Get material properties
POST   /api/materials                              # Create custom material
PATCH  /api/materials/:id                          # Update material properties
GET    /api/materials/:id/ocp-curve                # Get OCP data points

Simulations:
POST   /api/projects/:id/simulations              # Create and queue simulation
GET    /api/projects/:id/simulations              # List simulations
GET    /api/simulations/:id                       # Get simulation details and status
GET    /api/simulations/:id/progress               # Get solver progress (live via WebSocket)
GET    /api/simulations/:id/results                # Get simulation results (voltage, profiles)
POST   /api/simulations/:id/cancel                 # Cancel running simulation
POST   /api/simulations/:id/compare                # Compare two simulation results

DOE Studies:
POST   /api/projects/:id/doe                      # Create DOE study
GET    /api/doe/:id                               # Get DOE study status and results
POST   /api/doe/:id/submit                        # Submit all DOE simulations
GET    /api/doe/:id/sensitivity                   # Get sensitivity analysis results
GET    /api/doe/:id/response-surface              # Get response surface / Pareto front data

Cycling Data:
POST   /api/projects/:id/cycling-data             # Upload cycling data file
GET    /api/projects/:id/cycling-data             # List cycling datasets
GET    /api/cycling-data/:id                      # Get dataset summary and cycle metrics
GET    /api/cycling-data/:id/degradation           # Get capacity fade / resistance growth trends
POST   /api/cycling-data/compare                   # Compare multiple datasets

EIS:
POST   /api/projects/:id/eis                      # Upload EIS measurement
GET    /api/projects/:id/eis                      # List EIS measurements
GET    /api/eis/:id                               # Get EIS data (frequency, Z_real, Z_imag)
POST   /api/eis/:id/fit                           # Fit equivalent circuit model
GET    /api/eis-fits/:id                          # Get fit results and parameters
POST   /api/eis/:id/drt                           # Run DRT analysis
POST   /api/eis-fits/:id/link-to-simulation       # Map fitted params to simulation inputs

Thermal:
POST   /api/simulations/:id/thermal               # Run coupled thermal simulation
GET    /api/simulations/:id/thermal/results        # Get 3D temperature field
GET    /api/simulations/:id/thermal/vtk            # Download VTK file for ParaView

Experiments:
POST   /api/projects/:id/experiments              # Create experiment entry
GET    /api/projects/:id/experiments              # List experiments
GET    /api/experiments/:id                       # Get experiment details
PATCH  /api/experiments/:id                       # Update notes, tags, status
POST   /api/experiments/:id/link                  # Link simulations or datasets

Reports:
POST   /api/projects/:id/reports                  # Generate PDF/PPTX report
GET    /api/reports/:id                           # Download report
POST   /api/reports/:id/figures                   # Export individual figures (SVG/PNG)

Pack Simulation:
POST   /api/projects/:id/pack-simulations         # Create pack simulation
GET    /api/pack-simulations/:id                  # Get pack simulation results
POST   /api/pack-simulations/:id/drive-cycle      # Apply drive cycle profile

Collaboration:
POST   /api/projects/:id/share                    # Generate share link
POST   /api/projects/:id/members                  # Invite team member
PATCH  /api/projects/:id/members/:user_id         # Update member role
GET    /api/projects/:id/activity                  # Get activity feed
```

---

## UI/UX — Key Screens

### 1. Project Dashboard
- Overview cards: active simulations running, GPU hours consumed this month, total cells in database, recent experiments with status badges
- Project grid with chemistry tags (NMC811, LFP, Si/C), last modified dates, and quick-launch buttons for new simulation, data upload, or DOE study
- GPU usage bar and billing summary with cost-per-simulation breakdown
- Quick start templates: "NMC811/Graphite Pouch Cell", "LFP/Graphite 21700", "Silicon Anode Screening", "Pack Thermal Analysis"

### 2. Cell Designer
- Left panel: materials selector with searchable database, drag electrode/separator/electrolyte components into the stack builder
- Center: visual electrode stack diagram showing cathode coating, separator, anode coating, and current collectors with thickness dimensions and labeled material names
- Right panel: calculated cell metrics (capacity, energy density gravimetric/volumetric, N/P ratio, electrolyte filling ratio) updating in real time as parameters change
- Bottom: electrode recipe editor with weight fractions, resulting porosity calculation, and cost breakdown per component
- Top toolbar: save design, clone design, launch P2D simulation, generate Ragone plot, export parameter set

### 3. Simulation Workspace
- Split view: interactive voltage-vs-time/capacity plot (left) and spatial profile viewer (right, showing concentration and potential distributions across the cell sandwich)
- Model configuration panel: select model type (P2D/SPM/SPMe), set C-rate and protocol (CC, CCCV, pulse, custom schedule), temperature, voltage cutoffs
- Real-time solver progress bar with convergence metrics (residual norms) and estimated time remaining
- Toolbar: run, pause, cancel, compare with other simulations, overlay experimental cycling data, export results
- Animation slider for stepping through spatial profiles at different time points during charge/discharge

### 4. Cycling Data Explorer
- Left panel: dataset tree organized by project and cell ID, with search and filter controls (chemistry, cycle count range, date range)
- Center: multi-trace plot area showing voltage vs. capacity curves, with cycle selector slider to animate through cycle aging
- Right panel: per-cycle metrics table (discharge capacity, coulombic efficiency, DCR) and degradation trend plots (capacity fade, resistance growth vs. cycle number)
- Overlay controls: select multiple cells to compare on the same plot, toggle between voltage-time and voltage-capacity views
- Data quality indicators: flagged anomalous cycles highlighted in orange with tooltip explanations

### 5. EIS Fitting Studio
- Top: Nyquist plot (Z_imag vs. Z_real) with measured data points and fitted model curve overlay, zoom and pan controls
- Bottom-left: Bode plot (|Z| and phase vs. frequency) as secondary view
- Right panel: circuit model builder with drag-and-drop elements (R, C, L, CPE, W) into series/parallel topology, fitted parameter values with confidence intervals displayed next to each element
- Fit controls: select algorithm, set parameter bounds and initial guesses, run fit, view residual plot
- Multi-spectrum mode: load EIS at multiple SOC or temperatures, fit simultaneously with shared/independent parameter toggle

### 6. DOE Results Dashboard
- Top: parameter sweep configuration summary showing swept variables, ranges, and sampling method
- Center-left: response surface heatmap (e.g., energy density as a function of cathode thickness and porosity) with interactive colorbar and cursor readout
- Center-right: Pareto front scatter plot for multi-objective trade-off visualization (e.g., energy density vs. cycle life)
- Bottom: sensitivity bar chart showing Sobol indices for each parameter, ranked by importance
- Table view: sortable list of all DOE runs with input parameters and output metrics, click any row to open the full simulation result

---

## Monetization

### Free / Academic Tier
- 2 projects, up to 10 cell designs
- P2D and SPM solvers: up to 5 GPU-hours/month (sufficient for ~50 single simulations)
- Cycling data: up to 20 datasets, 100 MB storage
- EIS fitting: up to 10 fits/month
- No DOE sweeps, no pack simulation, no thermal 3D
- Community support only
- VoltVault watermark on exported figures
- Available to .edu email addresses with verified ORCID for extended academic limits

### Researcher — $149/month
- 20 projects, unlimited cell designs
- P2D and SPM solvers: 50 GPU-hours/month (sufficient for ~500 simulations or moderate DOE studies)
- 3D thermal modeling for single cells
- Cycling data: unlimited datasets, 50 GB storage
- EIS fitting: unlimited fits, DRT analysis
- DOE parameter sweeps: up to 200 runs per study
- Degradation modeling (SEI, plating, calendar/cycle aging)
- Experiment tracker with full versioning
- PDF report generation and publication-quality figure export
- Email support (48h response)

### Team — $499/month (up to 5 seats, $99/additional seat)
- Unlimited projects and cell designs
- 200 GPU-hours/month (sufficient for large DOE campaigns and pack simulations)
- All solver capabilities including pack simulation and drive cycle analysis
- Cycling data: unlimited storage with team-shared database
- Collaboration: role-based access, shared materials database, review workflows
- DOE parameter sweeps: unlimited runs, multi-objective optimization
- Custom materials database with property validation workflows
- API access for programmatic simulation submission and data retrieval
- Priority GPU queue (jobs start ahead of free-tier queue)
- Priority support (24h response), onboarding session

### Enterprise — Custom
- Unlimited everything with dedicated GPU allocation (reserved A100 instances)
- SSO/SAML integration, audit logging, data residency options (US, EU, Asia-Pacific)
- Private deployment option (VPC or on-premise Kubernetes cluster)
- Custom solver modules (solid-state electrolyte, sodium-ion, lithium-sulfur chemistry support)
- Integration with laboratory information management systems (LIMS) and PLM tools
- Bulk cycling data migration services from legacy systems
- Dedicated customer success manager, quarterly business reviews
- SLA with 99.9% uptime guarantee and priority incident response
- Volume GPU pricing with annual commitment discounts

---

## Go-to-Market Strategy

### Phase 1: Research Community (Month 1-3)
- Launch free tier and post on Hacker News (Show HN), r/batteries, r/electrochemistry, and the Electrochemical Society (ECS) online forums
- Publish validation studies comparing VoltVault P2D solver results against PyBaMM and COMSOL for standard benchmark cells (Chen 2020 NMC/graphite, Marquis 2019 SPM validation)
- Partner with 5–10 university battery research groups to offer free academic access in exchange for feedback and case studies
- YouTube tutorial series: "Battery Simulation in the Browser" — from cell design to degradation prediction in 30 minutes
- Contribute upstream to PyBaMM community (parameter sets, validation data) to build credibility and goodwill
- Present at ECS meetings and International Battery Seminar

### Phase 2: Industry Adoption (Month 3-6)
- Launch paid tiers with self-serve billing targeting cell manufacturers and EV startups
- Case studies from beta users: "How [Startup] reduced cell design iteration time from 6 weeks to 3 days with VoltVault"
- Content marketing: SEO targeting "battery simulation software", "COMSOL battery alternative", "P2D solver online", "EIS fitting tool"
- Partner with cycler manufacturers (Arbin, Neware) for data format support co-marketing and joint webinars
- Attend The Battery Show (North America and Europe), Advanced Automotive Battery Conference (AABC), and Materials Research Society (MRS) Fall Meeting
- Webinar series with industry guest speakers on cell design optimization, degradation analysis, and pack thermal management

### Phase 3: Enterprise and Platform (Month 6-12)
- Build enterprise features: SSO, dedicated GPU clusters, LIMS integration, data residency
- Hire industry-focused sales team targeting automotive OEMs, cell manufacturers (CATL, LG, Samsung SDI, Panasonic tier), and national laboratories
- Develop chemistry-specific modules: solid-state electrolyte modeling, sodium-ion cells, lithium-sulfur, lithium-metal anode
- Launch materials marketplace where researchers can publish and monetize validated parameter sets with DOI citation
- Pursue SOC 2 Type II certification and ISO 27001 for enterprise security requirements
- Build partnerships with materials informatics platforms (Citrine, Materials Project) for automated property lookup

### Acquisition Channels
- Organic search: target "battery simulation software", "P2D model solver", "EIS equivalent circuit fitting", "cycling data analysis tool", "PyBaMM GUI"
- Academic conference sponsorship: ECS, MRS, AABC, International Battery Seminar booth and workshop presence
- GitHub presence: open-source VoltVault data format converters (Arbin/Neware/Maccor parsers) and materials property datasets
- LinkedIn targeting battery engineer and electrochemist job titles at EV companies and cell manufacturers
- Research group referral program: referring professor gets 6 months free Team tier for their lab

---

## Success Metrics & KPIs

| Metric | Target (Month 6) | Target (Month 12) |
|--------|-------------------|---------------------|
| Registered users | 2,000 | 8,000 |
| Simulations completed (cumulative) | 10,000 | 60,000 |
| GPU hours consumed (cumulative) | 15,000 | 120,000 |
| Cycling datasets uploaded | 5,000 | 30,000 |
| Paying customers | 40 | 180 |
| MRR | $12,000 | $70,000 |
| Free → Paid conversion | 4% | 7% |
| Monthly churn rate | <6% | <4% |

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | VoltVault Advantage |
|-----------|-----------|------------|---------------------|
| COMSOL Battery Design Module | Gold standard physics fidelity, full multiphysics coupling (electrochemistry + thermal + structural), large user base in academia and industry | $50K+/seat, desktop-only, steep learning curve (weeks to set up a P2D model), no cycling data management, no EIS fitting, no DOE automation | 100x cheaper, browser-based, GPU-accelerated (faster solves), integrated cycling data + EIS + DOE in one platform |
| PyBaMM | Free and open-source, excellent Newman model implementation, active development community, Python flexibility | No GUI (code-only), requires Python/Conda expertise, no cycling data management, no EIS fitting, no 3D thermal, no collaboration features | Full GUI with visual model builder, integrated data management, EIS fitting studio, 3D thermal, team collaboration — all accessible without writing code |
| MATLAB/Simulink (Simscape) | Widely installed in industry, powerful optimization toolbox, Simulink for system-level pack modeling, extensive documentation | $10K+ per seat (base + toolboxes), desktop-only, battery models require manual PDE coding or expensive add-ons, no cycling data management | Purpose-built for batteries, GPU-accelerated solvers, integrated cycling data and EIS, fraction of the license cost, browser-based |
| BatteryArchitect (Voltaiq) | Strong cycling data management and analytics, good enterprise integrations, established customer base at major cell manufacturers | Limited physics-based simulation (empirical/statistical models only), no P2D/SPM solvers, no 3D thermal, no EIS fitting, analytics-focused not design-focused | Full physics-based simulation (P2D, SPM, thermal, degradation) alongside data management — design and validate in one platform |
| Ansys Fluent (Battery Module) | World-class 3D CFD and thermal solver, good multiphysics coupling, trusted by automotive OEMs for pack-level thermal analysis | $50K+/seat, extremely complex setup for electrochemistry, primarily thermal/fluid (not electrochemical modeling), no cycling data, no EIS, desktop-only | Purpose-built electrochemistry solvers (P2D/SPM) with coupled thermal, integrated EIS and cycling data, browser-based, accessible to electrochemists not just CFD specialists |

---

## Risks & Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| GPU compute costs erode margins on intensive DOE sweeps and degradation simulations | High | Medium | Spot/preemptible GPU instances for non-priority jobs (60-70% savings), adaptive mesh coarsening for screening runs, compute credit system with overage billing, encourage SPM for initial screening before P2D |
| P2D solver accuracy questioned vs. COMSOL/PyBaMM for edge-case chemistries and extreme operating conditions | High | Medium | Comprehensive validation suite against PyBaMM and published experimental data for 10+ chemistries, publish benchmark results openly, allow users to export model inputs and cross-validate in PyBaMM |
| Battery engineers distrust browser-based tools for critical cell design decisions affecting safety | Medium | Medium | Publish detailed solver documentation with numerical method descriptions, offer result export for independent verification, partner with national labs (NREL, Argonne) for third-party validation studies |
| Proprietary cycler data formats (Arbin .res, Neware .nda) change without notice, breaking parsers | Medium | High | Maintain parser test suites with real data samples, monitor cycler software releases, engage cycler manufacturers as partners, provide manual CSV upload as universal fallback |
| COMSOL, MathWorks, or Ansys launches a competing cloud battery simulation product with their existing solver IP | Medium | Medium | Move fast on integrated platform moat (simulation + data + EIS + DOE in one tool), build strong research community, offer data portability (no vendor lock-in), undercut on price with transparent GPU-hour billing |

---

## MVP Scope (v1.0)

### In Scope
- User authentication, organization and project management
- Cell designer with materials database (10 curated cathode/anode/electrolyte materials with validated parameters)
- GPU-accelerated P2D solver for single charge/discharge simulations (NMC and LFP cathodes, graphite anode)
- SPM solver with sub-second solve times for fast screening
- Basic cycling data upload and viewer (Arbin CSV and generic CSV import, voltage/capacity plots, per-cycle metrics)
- EIS data upload and equivalent circuit fitting (Randles and Randles+Warburg models, Nyquist/Bode visualization)
- Experiment tracker with simulation input/output versioning
- PDF report generation with auto-generated figures

### Out of Scope (v1.1+)
- 3D thermal modeling and Three.js thermal visualization (v1.1)
- DOE parameter sweeps and response surface visualization (v1.1)
- Degradation modeling (SEI, plating, cycle/calendar aging) (v1.2)
- Pack simulation and drive cycle analysis (v1.2)
- Neware, Maccor, and Biologic native format parsers (v1.1)
- Real-time collaboration and shared workspaces (v1.2)
- DRT analysis and advanced EIS models (transmission line, multi-spectrum fitting) (v1.2)
- Custom circuit builder for EIS (v1.3)
- Solid-state and sodium-ion chemistry modules (v2.0)
- Enterprise features: SSO, dedicated GPU, LIMS integration (v2.0)

### MVP Timeline: 10 weeks
- Week 1-2: Auth system, project/org data model, materials database with 10 curated materials (NMC811, NMC622, LFP, graphite, SiO_x, LP30 electrolyte, etc.), cell designer UI with electrode stack builder and metric calculations, S3 storage setup
- Week 3-4: P2D solver implementation in Rust with CUDA GPU acceleration (solid-phase diffusion, electrolyte transport, Butler-Volmer kinetics), solver job orchestration with Celery/Redis, WebSocket progress streaming, basic voltage-vs-time result visualization
- Week 5-6: SPM solver implementation, simulation workspace UI (model config panel, interactive voltage plot, spatial profile viewer), simulation comparison overlay, cycling data CSV upload pipeline with automatic cycle detection and per-cycle metric extraction
- Week 7-8: EIS data upload and Nyquist/Bode plot viewer, Randles circuit fitting engine (Levenberg-Marquardt via lmfit), fitted parameter display with confidence intervals, experiment tracker with tagging and search
- Week 9-10: PDF report generation with cell design summary, simulation results, and cycling data figures, billing integration (Stripe) with free/paid tier enforcement, cycling data explorer with multi-cell comparison, polish, solver validation against PyBaMM benchmarks, load testing, beta launch
