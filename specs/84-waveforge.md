# WaveForge — Coastal and Ocean Wave Modeling Platform

## Executive Summary

WaveForge is a cloud-based coastal engineering platform for ocean wave propagation, nearshore hydrodynamics, sediment transport, and morphodynamic modeling. It replaces SWAN (free but arcane), MIKE21 ($20K+/seat), Delft3D (open-source but complex), and STWAVE with a modern browser-based interface featuring GPU-accelerated spectral wave modeling, integrated sediment transport, and AI-assisted calibration from satellite and buoy data.

---

## Problem Statement

**The pain:**
- MIKE21 (DHI) costs $20,000-$60,000/seat/year for wave + flow + sediment modules; MIKE Coupled Model FM adds another $15,000-$30,000 per module — most coastal projects need 3-4 modules
- SWAN (Delft University) is free but has no GUI, requires manual mesh generation, cryptic input files, and produces raw binary output that needs separate post-processing tools
- Coastal engineering projects require coupling wave models (SWAN/STWAVE) with circulation models (ADCIRC/Delft3D-FLOW) and sediment models (GENESIS/UNIBEST) — each with different grids, coordinate systems, and file formats
- Setting up a nearshore wave model requires manual bathymetry processing, mesh generation, boundary condition specification from global wave models, and parameter calibration — a skilled modeler spends 60-70% of project time on setup
- Climate change and sea level rise are increasing demand for coastal flood risk and erosion assessments, but the talent pool of experienced coastal modelers is extremely small (estimated 10,000-15,000 worldwide)

**Current workarounds:**
- Using SWAN with hand-edited text input files and MATLAB/Python scripts for post-processing, spending days on format conversions
- Relying on empirical formulas (CERC, Kamphuis) in spreadsheets for sediment transport estimates instead of process-based modeling
- Hiring specialist coastal modeling consultants at $200-$400/hour for studies that could be partly automated
- Using Delft3D open-source with weeks of training investment and limited technical support, often abandoning models mid-project due to convergence issues

**Market size:** The coastal and ocean engineering software market is approximately $1.5 billion (2024). There are 50,000+ coastal/ocean engineers and marine scientists worldwide. Global coastal protection spending exceeds $40 billion/year and is projected to grow 8-10% annually due to climate change adaptation requirements.

---

## Target Users

### Primary Personas

**1. Elena — Coastal Engineer at a Design Consultancy**
- Designs breakwaters, seawalls, beach nourishment projects, and port layouts for municipal and private clients
- Uses MIKE21 SW for waves and MIKE21 FM for hydrodynamics, paying $40K+/year in license fees
- Needs: integrated wave-current-sediment modeling with automated mesh generation and design wave extraction for structure design

**2. Dr. Tanaka — Marine Scientist at an Oceanographic Institute**
- Studies wave climate variability, storm surge impacts, and coastal erosion under climate change scenarios
- Uses SWAN + Delft3D with custom Python coupling scripts that break with every software update
- Needs: coupled wave-circulation modeling with hindcast capability using ERA5/WW3 boundary conditions and satellite data assimilation

**3. Marcus — Offshore Renewable Energy Developer**
- Assesses wave energy resources and designs wave energy converter (WEC) arrays and offshore wind farm foundations
- Uses SWAN for wave resource assessment but needs wave-structure interaction and fatigue load spectra
- Needs: high-resolution spectral wave modeling with WEC performance prediction, wave power statistics, and extreme value analysis

---

## Solution Overview

WaveForge is a cloud-based coastal modeling platform that:
1. Performs spectral wave modeling (action balance equation) on unstructured meshes with automatic mesh generation from bathymetric data
2. Solves nearshore circulation (depth-averaged and 3D) driven by wave radiation stresses, tides, and wind, with wetting/drying for inundation mapping
3. Simulates sediment transport (bed load + suspended load) and morphodynamic bed evolution for sandy and mixed-sediment coasts
4. Generates design wave statistics, extreme value analysis (POT, GEV), and long-term wave climate assessments from hindcast and measured data
5. Provides harbor resonance analysis, wave agitation studies, and breakwater overtopping calculations per EurOtop guidelines

---

## Core Features

### F1: Spectral Wave Modeling
- Action balance equation solver on unstructured triangular meshes with local refinement near coastlines and structures
- Wind generation: Cavaleri & Malanotte-Rizzoli, Janssen (WAM Cycle 4), Komen, ST6 source term package
- Whitecapping dissipation: Komen, Janssen, ST6 (Rogers/Zieger/Young)
- Depth-induced breaking: Battjes-Janssen (constant and variable breaker index), Thornton-Guza
- Bottom friction: JONSWAP, Collins, Madsen
- Triad nonlinear interactions: LTA (Lumped Triad Approximation), SPB
- Quadruplet nonlinear interactions: DIA (Discrete Interaction Approximation), XNL (exact)
- Wave diffraction: phase-decoupled refraction-diffraction approximation
- Obstacle transmission: breakwaters, reefs (frequency-dependent transmission coefficients, d'Angremond/Van der Meer)
- Nested grid capability: global → regional → local downscaling with automatic boundary interpolation
- Stationary and non-stationary (time-stepping) modes

### F2: Nearshore Circulation
- Depth-averaged (2DH) shallow water equations with wave-driven currents (radiation stress gradient forcing)
- 3D sigma-coordinate model for vertical current structure (undertow, longshore current profiles)
- Tidal forcing from global tide models (FES2014, TPXO9) with harmonic analysis
- Storm surge: wind stress + atmospheric pressure + wave setup
- Wetting and drying for coastal inundation and overland flow
- Turbulence: depth-averaged Smagorinsky, k-epsilon 3D
- Coriolis force and wind-driven circulation for large domain simulations
- River discharge boundary conditions for estuarine modeling
- Structure representation: porous breakwaters, groins, jetties, seawalls (sub-grid)

### F3: Sediment Transport and Morphodynamics
- Bed load transport: Soulsby-van Rijn, Van Rijn (2007), Meyer-Peter-Müller, Engelund-Hansen
- Suspended load: depth-averaged advection-diffusion with reference concentration models
- Wave-current combined bed shear stress: Soulsby (1997), Fredsoe (1984)
- Multiple sediment fractions: sand, silt, clay with armoring and selective transport
- Morphodynamic acceleration (MORFAC) for long-term evolution simulations
- Bank erosion and avalanching for dune and cliff retreat
- Bed composition bookkeeping (layered bed stratigraphy)
- Bed slope effects on transport direction
- Hardpoints and non-erodible layers
- Validated for beach nourishment, inlet dynamics, and longshore drift applications

### F4: Harbor and Coastal Structure Analysis
- Harbor resonance (seiche) analysis: mild-slope equation solver for complex harbor geometries
- Wave agitation study: significant wave height and direction maps inside port basins
- Breakwater overtopping: EurOtop (2018) empirical formulas and neural network tools
- Wave runup: Stockdon (2006), Van der Meer & Stam, EurOtop for different structure types
- Armor stability: Van der Meer formulas for rubble mound, Hudson, cubes, Xbloc, Accropode
- Wave transmission through and over structures: d'Angremond, Van der Meer, Briganti
- Berthing/mooring analysis: wave-induced vessel motions in 6 DOF (frequency domain)
- Downtime assessment: operability statistics for port operations based on wave thresholds

### F5: Wave Climate and Extreme Value Analysis
- Automatic boundary condition extraction from global wave models (ERA5, WaveWatch III, CFSR)
- Buoy data import (NDBC, CDIP, Copernicus, user CSV) with quality control and gap-filling
- Satellite altimetry wave data integration (Sentinel-6, Jason-3)
- Joint probability analysis: wave height, period, direction, water level (JOINSEA method)
- Extreme value analysis: Peaks Over Threshold (GPD), Annual Maxima (GEV), with return value plots and confidence intervals
- Directional wave climate roses and seasonal/interannual variability statistics
- Spectral analysis: FFT, wavelet, directional spectrum estimation (MEM, EMEP)
- Sea-state partitioning: wind sea vs. swell separation
- Climate change wave projections: dynamical and statistical downscaling of GCM outputs
- Wave power resource assessment: mean power, power matrix, capacity factor for WEC devices

### F6: Mesh Generation and Data Management
- Automatic unstructured mesh generation from bathymetry contours with coastline-following refinement
- Bathymetry data import: GEBCO, EMODnet, NOAA ENC, survey XYZ, GeoTIFF, LAS/LAZ
- Coastline extraction from satellite imagery and vector data (OpenStreetMap, GSHHG)
- Mesh quality metrics: element skewness, aspect ratio, orthogonality with automatic smoothing
- Spatial interpolation of bathymetry onto mesh: natural neighbor, kriging, IDW
- Tidal gauge and wave buoy data download from global networks (UHSLC, GLOSS, NDBC)
- Project versioning: track model configuration changes and compare scenario results
- Automated report generation: wave climate summaries, design condition tables, PDF/HTML output

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | TypeScript, React, Mapbox GL JS / Deck.gl (bathymetry, results visualization), WebGL (spectral plots, 3D wave fields) |
| Wave Solver | Rust (spectral action balance, unstructured FV) → GPU-accelerated server |
| Circulation Solver | Rust (shallow water equations, implicit time integration, sigma-coordinate 3D) |
| Sediment Engine | Rust (transport formulas, morphodynamic update, bed stratigraphy) |
| Mesh Generator | Rust (Delaunay triangulation, advancing front, mesh quality optimization) → WASM + server |
| AI/ML | Python (auto-calibration via Bayesian optimization, satellite data assimilation, surrogate wave models) |
| Backend | Rust (Axum) + Python (FastAPI for extreme value analysis, wave climate statistics) |
| Database | PostgreSQL + PostGIS (bathymetry, coastlines, buoy data), S3 (wave model results, time series, ERA5 boundary data) |
| Hosting | AWS multi-region (c7g for wave solver, p4d/p5 GPU for large-domain runs) |

---

## Monetization

### Free Tier (Student)
- Spectral wave modeling on structured grids (up to 5,000 cells)
- Single-domain stationary wave analysis
- Basic wave climate statistics from built-in ERA5 data
- 3 projects

### Pro — $129/month
- Unstructured mesh wave modeling (up to 100,000 cells)
- Non-stationary wave simulation
- Nearshore circulation (2DH)
- Extreme value analysis
- Harbor agitation studies
- 500 cloud compute hours/month
- PDF report generation

### Advanced — $349/user/month
- Everything in Pro
- Unlimited mesh size with GPU acceleration
- 3D circulation and sediment transport
- Morphodynamic evolution
- Coupled wave-current-sediment runs
- Climate change scenario modeling
- Breakwater design tools (EurOtop)
- API access

### Enterprise — Custom
- On-premise deployment
- Real-time operational wave forecasting integration
- Custom calibration with proprietary field data
- Multi-project coastal zone management dashboards
- Training and certification
- Dedicated support with SLA

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | WaveForge Advantage |
|-----------|-----------|------------|---------------------|
| MIKE21 (DHI) | Gold standard, comprehensive modules | $20-60K/seat/year, expensive module stacking | 10x cheaper, all modules integrated |
| SWAN | Free, well-validated, widely published | No GUI, arcane input files, no sediment | Modern UX, integrated sediment + circulation |
| Delft3D | Open-source, good coupling framework | Steep learning curve, limited support, complex setup | Cloud-native, automated mesh + calibration |
| STWAVE | Free (USACE), fast | Structured grid only, no sediment, limited physics | Unstructured mesh, full coastal workflow |
| FUNWAVE | Excellent nearshore physics (Boussinesq) | Research code, no GUI, small domains only | Practical engineering tool, large domains, GPU |

---

## MVP Scope (v1.0)

### In Scope
- Bathymetry import (GEBCO, GeoTIFF, XYZ) with automatic mesh generation
- Stationary spectral wave modeling on unstructured meshes (SWAN-equivalent physics)
- Boundary condition extraction from ERA5 reanalysis
- Wave climate statistics and directional roses
- Basic extreme value analysis (POT, GEV)
- Wave result visualization on interactive map (Hs, Tp, direction)
- PDF wave study report generation

### Out of Scope (v1.1+)
- Nearshore circulation solver
- Sediment transport and morphodynamics
- Harbor resonance (mild-slope equation)
- Breakwater overtopping and structure design tools
- Coupled wave-current-sediment modeling
- Real-time forecasting integration

### MVP Timeline: 14-18 weeks
