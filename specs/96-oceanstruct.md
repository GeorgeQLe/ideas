# OceanStruct — Offshore Structure Design Platform

## Executive Summary

OceanStruct is a cloud-based platform for designing and analyzing offshore structures — fixed platforms, jackets, floating systems, mooring lines, risers, and subsea pipelines. It replaces SACS (Bentley, $15K+/seat), Sesam (DNV, $20K+/seat), OrcaFlex ($15K+/seat), and AQWA (ANSYS, $20K+/seat) with an integrated browser-based environment covering hydrodynamic loading, structural analysis, fatigue assessment, and mooring design for both oil & gas and floating wind applications.

---

## Problem Statement

**The pain:**
- Offshore structural engineering requires multiple expensive tools: SACS (Bentley) costs $15,000-$30,000/seat for jacket/platform analysis, Sesam (DNV) costs $20,000-$50,000/seat for a full hydrodynamic + structural suite, OrcaFlex costs $15,000-$25,000/seat for mooring/riser dynamics, and AQWA (ANSYS) costs $20,000-$40,000/seat for diffraction/radiation analysis
- A single offshore project may require 4-5 separate tools for the full design loop: hydrodynamic loading → structural analysis → fatigue → mooring → riser — with constant manual data transfer between them
- The floating offshore wind industry is creating massive new demand for offshore structural tools, but startups and emerging developers cannot afford the $100K+ tool stacks used by incumbent oil & gas firms
- Offshore fatigue analysis requires processing thousands of sea states and millions of stress cycles — desktop tools with sequential processing take days per analysis iteration
- Design codes (API, ISO 19902, DNVGL) are complex, and manual code checking is error-prone and time-consuming

**Current workarounds:**
- Using SACS for structural analysis and manually exporting to spreadsheets for fatigue post-processing
- Running OrcaFlex for mooring dynamics separately and manually coupling hydrodynamic loads to structural models
- Building custom MATLAB scripts for Morison loading and simplified wave analysis when full diffraction tools are too expensive
- Relying on classification society consultants ($200-$500/hour) for code compliance checks that could be partially automated

**Market size:** The offshore structures market (oil & gas platforms, offshore wind foundations, subsea infrastructure) exceeds $80 billion annually. The offshore wind sector alone is growing at 25%+ CAGR with $50B+ projected annual spend by 2030. Offshore structural engineering software is a $1.5-2 billion segment. There are 80,000+ offshore and marine structural engineers worldwide.

---

## Target Users

### Primary Personas

**1. Erik — Offshore Structural Engineer at an EPCI Contractor**
- Designs fixed jacket platforms for shallow-water oil & gas installations in the North Sea
- Uses SACS for structural analysis and Sesam for hydrodynamic loading, spending 40% of time on data transfer between tools
- Needs: integrated platform where wave loads flow directly into structural analysis and fatigue without manual file conversion

**2. Dr. Mei — Floating Wind Foundation Designer**
- Designs semi-submersible and spar foundations for floating offshore wind turbines at a renewable energy startup
- Cannot afford DNV Sesam ($50K+ full suite) on a startup budget; uses simplified spreadsheet models and open-source OpenFAST
- Needs: affordable coupled hydrodynamic-structural analysis for floating wind concepts with mooring design

**3. Carlos — Subsea Pipeline Engineer**
- Designs subsea pipelines and risers for deepwater developments off Brazil
- Uses OrcaFlex for dynamic riser analysis but struggles with seabed-pipe interaction and lateral buckling analysis
- Needs: integrated riser and pipeline design with proper soil-pipe interaction, VIV assessment, and fatigue life prediction

---

## Solution Overview

OceanStruct is a cloud-based offshore structural engineering platform that:
1. Computes hydrodynamic loads on fixed and floating structures using Morison equation for slender members and panel-method diffraction/radiation for large bodies
2. Performs structural analysis (static, dynamic, pushover, seismic) of jackets, topsides, and floating hulls with automated API/ISO/DNV code checking
3. Runs spectral and time-domain fatigue analysis with S-N curves and fracture mechanics, processing thousands of sea states in parallel on cloud compute
4. Designs and analyzes mooring systems (catenary, taut, semi-taut) with coupled floater-mooring dynamics in irregular seas
5. Models risers and subsea pipelines including VIV, seabed interaction, lateral buckling, and installation analysis

---

## Core Features

### F1: Hydrodynamic Analysis
- Morison equation loading for slender cylindrical members: inertia + drag with Wheeler stretching and current profiles
- Panel method (3D diffraction/radiation): added mass, radiation damping, and wave excitation force RAOs for large-volume bodies
- Irregular wave generation: JONSWAP, Pierson-Moskowitz, Ochi-Hubble spectra with directional spreading
- Second-order wave forces: mean drift (Newman approximation), slow-drift (difference frequency QTFs), sum-frequency (springing)
- Current loading: uniform, power-law, and user-defined current profiles
- Wind loading: steady + turbulent wind spectra (Kaimal, API) on superstructure and turbine (for floating wind)
- Wave kinematics: linear Airy, Stokes 2nd/5th order, stream function for near-breaking waves
- Combined wave + current + wind environmental load cases per metocean data
- Multi-directional scatter diagram processing with probability weighting

### F2: Structural Analysis
- 3D frame/beam element structural model for jackets, tripods, monopiles, and topsides
- Plate and shell elements for hull structural analysis (semi-submersibles, spars, FPSO)
- Static analysis: dead loads, live loads, equipment, hydrostatic pressure
- Dynamic analysis: natural frequency extraction, forced response, time-domain simulation
- Pushover analysis: ultimate strength assessment for extreme storm conditions
- Seismic analysis: response spectrum and time-history methods
- Tubular joint design: punching shear checks per API RP 2A / ISO 19902, joint classification (K, T, Y, X)
- Member strength checks: combined axial + bending (unity checks) per API RP 2A, ISO 19902, NORSOK N-004
- Foundation modeling: pile-soil interaction (p-y, t-z, Q-z curves), mudmat bearing capacity
- Automated code compliance reports (API, ISO, NORSOK, DNVGL)

### F3: Fatigue Analysis
- Spectral fatigue: transfer functions from hydrodynamic analysis → stress RAOs → spectral moments → Dirlik/Benasciutti-Tovo cycle counting
- Time-domain fatigue: rainflow cycle counting from time-history simulations
- S-N curves: DNV-RP-C203 (cathodic protection, free corrosion, in-air), BS 7608, API RP 2A
- Hot-spot stress method: SCF (stress concentration factor) calculation for tubular joints (Efthymiou parametric equations)
- Cumulative damage: Palmgren-Miner rule with full scatter diagram integration (Hs, Tp, direction combinations)
- Fatigue life maps: color-coded visualization of fatigue damage across entire structure
- Fracture mechanics: Paris law crack growth for inspection planning
- Fatigue damage equivalent loads for floating wind turbine tower base

### F4: Mooring Analysis
- Catenary, taut-leg, and semi-taut mooring configurations
- Line types: chain (studlink, studless), wire rope, polyester, HMPE, nylon
- Quasi-static catenary analysis: offset vs. restoring force curves
- Coupled time-domain dynamics: floater 6-DOF motion with nonlinear mooring line dynamics (lumped mass or FEM)
- Mooring line fatigue: tension-tension S-N curves, OPB (out-of-plane bending) at fairlead and anchor
- Anchor design: drag embedment, suction pile, driven pile, gravity anchor — holding capacity vs. soil type
- Mooring integrity: intact, one-line-damaged, transient cases per API RP 2SK / DNVGL-OS-E301
- Clump weights, buoyancy modules, and intermediate connection design
- Watch circle analysis: maximum offset vs. riser/umbilical limits

### F5: Riser and Pipeline Design
- Steel catenary riser (SCR): static configuration, dynamic response, fatigue at touchdown zone
- Flexible riser: lazy wave, steep wave, free hanging configurations with bend stiffener design
- Top-tensioned riser: tensioner stroke analysis, VIV response
- VIV assessment: modal analysis, reduced velocity screening, Shear7/VIVANA-equivalent empirical models
- VIV suppression: strake and fairing effectiveness modeling
- Subsea pipeline: on-bottom stability (DNV-RP-F109), lateral buckling, upheaval buckling
- Pipe-soil interaction: axial and lateral friction, embedment, breakout resistance
- Installation analysis: S-lay, J-lay, reel-lay — pipeline stress during installation
- Free span assessment: static deflection, VIV-induced fatigue at unsupported spans

### F6: Floating Wind Specific
- Coupled aero-hydro-servo-elastic analysis for floating wind concepts (simplified aero: thrust curve + gyroscopic)
- Platform concepts: spar, semi-submersible, barge, TLP — parameterized geometry generators
- Hydrostatic stability: intact and damaged stability per DNVGL-ST-0119
- Turbine RNA loads: IEC 61400-3-2 design load cases with aerodynamic thrust time series import
- Tower-platform interface loads and fatigue
- Dynamic cable (power export): lazy-wave configuration, bend radius, fatigue
- Shared mooring concepts for multi-turbine floating wind farms
- Cost estimation: CAPEX breakdown for hull, mooring, installation per platform concept

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | TypeScript, React, Three.js / Deck.gl (3D structure + mooring visualization), D3.js (fatigue maps, RAO plots) |
| Hydro Solver | Rust (Morison equation, panel method BEM for diffraction/radiation, irregular wave generation) |
| Structural Solver | Rust (FEM frame/shell elements, eigenvalue extraction, time-domain integration) |
| Mooring Solver | Rust (catenary statics, lumped-mass dynamics, coupled floater-mooring time domain) |
| Backend | Python (FastAPI) for metocean processing, S-N curve libraries, code checks; Rust (Axum) for compute orchestration |
| AI/ML | Python (PyTorch) for sea state clustering, fatigue damage surrogate models, metocean extreme value analysis |
| Database | PostgreSQL (projects, metocean databases, S-N curves, material library), S3 (time-series results, large RAO datasets) |
| Hosting | AWS (c7g for structural/mooring, r7g for panel-method BEM, HPC burst for fatigue scatter diagram processing) |

---

## Monetization

### Free Tier (Student)
- Simple jacket or monopile (up to 50 members)
- Morison loading with regular waves only
- Static structural analysis with basic code checks
- 2 projects

### Pro — $299/month
- Unlimited structural complexity
- Full Morison + diffraction/radiation hydrodynamics
- Spectral and time-domain fatigue
- Catenary mooring analysis (quasi-static)
- S-N curve library (DNV, API, BS)
- Automated code compliance reports
- 20 projects

### Team — $599/user/month
- Coupled time-domain mooring dynamics
- Riser and pipeline design
- VIV assessment
- Floating wind modules
- Second-order wave forces (QTFs)
- API access and team collaboration
- Unlimited projects

### Enterprise — Custom
- On-premise deployment for EPCI contractors
- Custom metocean database integration
- Classification society submission packages
- Multi-project portfolio management
- Dedicated support and model review
- Integration with Aveva, PDMS, SmartPlant

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | OceanStruct Advantage |
|-----------|-----------|------------|----------------------|
| SACS (Bentley) | Industry standard for fixed platforms, large user base | $15-30K/seat, Windows-only, limited floating capability | Cloud-native, integrated floating + fixed, lower cost |
| Sesam (DNV) | Comprehensive hydro + structural + fatigue, classification tie-in | $20-50K/seat for full suite, steep learning curve | Unified platform, 5x faster fatigue with cloud parallelism |
| OrcaFlex (Orcina) | Best-in-class mooring/riser dynamics | $15-25K/seat, no structural FEM, separate from hydro | Integrated structural + mooring + riser in one tool |
| AQWA (ANSYS) | Excellent diffraction/radiation solver | $20-40K/seat, requires ANSYS ecosystem, no structural detail | Standalone, includes structural and fatigue |
| MOSES (Bentley) | Good floating body stability and tow analysis | Aging codebase, limited fatigue, separate from SACS | Modern UI, integrated fatigue, floating wind focus |

---

## MVP Scope (v1.0)

### In Scope
- 3D frame model creation for jacket/monopile structures (beam elements)
- Morison equation wave loading with JONSWAP irregular seas
- Static structural analysis with member unity checks (API RP 2A)
- Natural frequency extraction (first 10 modes)
- Spectral fatigue at selected joints (Efthymiou SCFs, DNV S-N curves)
- Basic catenary mooring analysis (quasi-static)
- Interactive 3D structure visualization in browser

### Out of Scope (v1.1+)
- Panel-method diffraction/radiation
- Coupled time-domain mooring dynamics
- Riser and pipeline analysis
- VIV assessment
- Floating wind modules
- Seismic and pushover analysis

### MVP Timeline: 16-20 weeks
