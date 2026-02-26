# FoamSim — Polymer and Foam Process Simulation Platform

## Executive Summary

FoamSim is a cloud-based polymer processing simulation platform for injection molding, foam blowing and expansion, thermoforming, rubber vulcanization, and extrusion. It replaces Moldex3D ($25K+/seat), Moldflow ($15K+/seat), Sigmasoft ($20K+/seat), and custom foam research codes with browser-based mold filling simulation, foam cell prediction, and process optimization — bringing polymer processing simulation to the thousands of plastics engineers who currently design by trial and error.

---

## Problem Statement

**The pain:**
- Moldex3D costs $25,000-$50,000/seat/year; Moldflow (Autodesk) costs $15,000-$35,000/seat; Sigmasoft costs $20,000-$40,000/seat — putting simulation out of reach for most small and mid-size molders
- Injection molding accounts for 30%+ of all plastic parts manufactured; a single mold tool costs $20K-$500K, and simulation-free trial-and-error causes 3-8 mold re-cuts at $5K-$50K per iteration
- Foam injection molding (MuCell, chemical foaming) is growing rapidly for automotive lightweighting but no affordable tool predicts foam cell morphology — engineers rely on expensive physical trials
- Polymer rheology is inherently complex (shear-thinning, temperature-dependent, viscoelastic) and requires specialized constitutive models (Cross-WLF, Carreau, PTT) that general-purpose CFD tools do not natively support
- Shrinkage and warpage prediction — the most valuable output of molding simulation — requires coupling of flow, packing, cooling, and solid mechanics, which demands specialized solvers

**Current workarounds:**
- Using Moldflow included in Autodesk product design suites but limited to Hele-Shaw (midplane/dual-domain) analysis that misses 3D fountain flow and thick-part effects
- Running trial-and-error on the production floor: experienced molders adjust gate location, melt temperature, pack pressure, and cooling time by feel, requiring 50-200 shots to dial in a new mold
- Using general-purpose CFD (ANSYS Fluent, OpenFOAM) with custom rheology UDFs — possible but requires months of development and validation per material model
- Relying on material supplier datasheets and simple fill-time calculators for gate sizing, with no prediction of weld lines, air traps, or warpage

**Market size:** The injection molding simulation software market is approximately $700 million (2024), within a broader plastics processing market worth $400 billion globally. There are an estimated 300,000+ plastics/polymer engineers worldwide, with 50,000+ mold designers. The foam processing segment is the fastest-growing subsector, driven by automotive lightweighting mandates.

---

## Target Users

### Primary Personas

**1. Carlos — Mold Designer at a Custom Injection Molding Shop**
- Designs molds for consumer products, medical devices, and automotive components
- Has no simulation software; relies on 25 years of experience and trial-and-error to set gates, runners, and cooling lines
- Needs: affordable filling simulation to predict short shots, weld lines, air traps, and gate location optimization before cutting steel

**2. Dr. Weber — Foam Process Engineer at an Automotive Tier 1 Supplier**
- Develops MuCell (physical foaming) and chemical foaming processes for lightweight interior and structural parts
- Uses custom MATLAB models for cell nucleation and growth but cannot couple them with mold filling
- Needs: integrated foam injection molding simulation predicting cell size distribution, density variation, and surface quality (swirl marks, silver streaks)

**3. Aisha — Packaging Engineer at a Thermoforming Company**
- Designs thermoformed trays, clamshells, and containers from sheet stock
- Currently estimates wall thickness distribution by experience; rejects run 5-15% due to thin spots
- Needs: thermoforming simulation predicting wall thickness distribution, plug-assist optimization, and material draw-in

**4. Tomasz — Rubber Process Engineer at a Seal and Gasket Manufacturer**
- Designs compression and transfer molds for EPDM, silicone, and FKM rubber parts
- Needs to predict vulcanization (cure) uniformity, flow during mold filling, and residual stress in the cured part
- Needs: rubber-specific simulation with vulcanization kinetics, hyperelastic material models, and mold flow analysis

---

## Solution Overview

FoamSim is a cloud-based polymer processing platform that:
1. Simulates injection mold filling using both Hele-Shaw (fast, thin-wall) and full 3D Navier-Stokes (thick parts, complex flow) solvers with non-Newtonian rheology (Cross-WLF, Carreau, PTT models)
2. Predicts foam cell nucleation, growth, and coalescence during foam injection molding and extrusion foaming, providing cell size distribution and local density maps
3. Computes packing, cooling, solidification/crystallization, and post-mold shrinkage and warpage with coupled flow-thermal-structural analysis
4. Supports thermoforming (plug-assisted, vacuum, pressure), rubber vulcanization (compression, transfer, injection), and polymer extrusion (profile, sheet, blown film)
5. Optimizes gate location, runner balance, cooling channel layout, and process parameters (melt temperature, injection speed, pack pressure, cooling time) using AI-driven process windows

---

## Core Features

### F1: Injection Mold Filling Simulation
- Hele-Shaw (2.5D) solver for thin-wall parts: midplane and dual-domain analysis with automatic midplane extraction
- Full 3D Navier-Stokes solver for thick sections, complex ribs, and boss features where fountain flow matters
- Non-Newtonian viscosity models: Cross-WLF (standard for injection molding), Carreau-Yasuda, power law
- Fountain flow and frozen layer development during filling
- Automatic runner and gate meshing: cold runner, hot runner, valve gate, sequential valve gate
- Weld line and air trap prediction with meld/weld angle classification
- Filling pattern animation with pressure, temperature, velocity, and shear rate contours
- Short shot prediction and fill balance analysis for multi-cavity molds
- Overmolding and insert molding: multi-material sequential injection
- Gas-assist and water-assist injection molding simulation

### F2: Foam Processing
- Classical nucleation theory (CNT) for cell nucleation: homogeneous and heterogeneous nucleation with nucleating agent effects
- Cell growth model: diffusion-controlled bubble growth in a viscoelastic polymer matrix (Amon-Denson, modified Scriven)
- Cell coalescence and collapse modeling for open-cell vs. closed-cell foam prediction
- Chemical blowing agent (CBA) decomposition kinetics: azodicarbonamide, sodium bicarbonate, and custom agents with Arrhenius decomposition
- Physical blowing agent (PBA) dissolution and solubility: CO2, N2 in polymer melts using Sanchez-Lacombe EOS
- MuCell process simulation: SCF dosing, single-phase solution (SPS) formation, pressure drop nucleation at gate
- Foam density distribution mapping within the mold cavity
- Skin-core structure prediction: solid skin thickness vs. foamed core
- Surface quality prediction: swirl marks, silver streaks, surface roughness from gas breakthrough
- Extrusion foaming: die swell, post-die expansion, cooling and cell stabilization

### F3: Packing, Cooling, and Solidification
- Packing phase simulation: compressible flow with pressure-dependent density (Tait equation of state)
- Gate freeze-off prediction: solidified layer growth at the gate as a function of pack time and pressure
- Cooling channel simulation: conformal and conventional channels with coolant flow (turbulent convection coefficients)
- Mold thermal analysis: steady-state and transient mold temperature distribution with cycle-averaged heating
- Solidification: amorphous (Tg-based) and semi-crystalline (Nakamura crystallization kinetics, Avrami)
- Crystallinity distribution mapping: affects mechanical properties, shrinkage, and optical clarity
- Cooling time optimization: minimize cycle time while meeting ejection temperature and crystallinity targets
- Mold surface temperature prediction for evaluation of surface finish quality

### F4: Shrinkage and Warpage
- Thermo-viscoelastic stress analysis during cooling and ejection
- Differential shrinkage: flow vs. cross-flow direction due to molecular orientation
- Warpage prediction: in-mold and post-ejection deformation with gravity and constraint effects
- Fiber orientation tensor (Folgar-Tucker model) for glass/carbon fiber-reinforced polymers
- Fiber orientation effects on anisotropic shrinkage and mechanical properties
- Warpage compensation: recommend mold cavity modifications to counteract predicted warpage
- Dimensional conformance check against nominal CAD geometry
- Core shift prediction due to injection pressure imbalance
- Sink mark prediction on thick ribs and bosses

### F5: Thermoforming and Rubber Vulcanization
- Thermoforming simulation: sheet heating (infrared, contact), sag under gravity, plug-assisted stretching, vacuum/pressure forming
- Hyperelastic/viscoplastic sheet models: Mooney-Rivlin, Ogden for large-deformation sheet stretching
- Wall thickness distribution prediction for thermoformed parts
- Plug geometry and timing optimization for material distribution control
- Rubber injection/compression/transfer molding: flow simulation with vulcanization coupling
- Vulcanization kinetics: Kamal-Sourour model, induction time, cure state (percent cure) mapping
- Scorch prediction: premature vulcanization in runners and thin sections
- Hyperelastic models for cured rubber: Mooney-Rivlin, Ogden, Arruda-Boyce
- Demolding simulation: residual stress, part distortion on ejection from mold

### F6: Material Database and Process Optimization
- Polymer material database: 2,000+ grades with viscosity (Cross-WLF), pvT (Tait), thermal, and mechanical properties from major suppliers (BASF, DuPont, SABIC, Covestro, LyondellBasell)
- Material card fitting tool: input rheometer data (capillary, rotational), DSC, pvT → automatic model parameter fitting
- Fiber-filled material properties: fiber length, aspect ratio, concentration effects on viscosity and mechanical properties
- Process window advisor: recommend melt temperature, mold temperature, injection speed, pack pressure ranges for given material and part geometry
- Gate location optimization: evaluate multiple gate locations and rank by fill balance, weld line quality, and pressure requirement
- Runner balance optimization for multi-cavity molds
- Cooling circuit optimization: channel diameter, location, coolant temperature, flow rate
- Cycle time estimation and cost-per-part calculator
- DOE-based process optimization: response surface for quality metrics vs. process parameters

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | TypeScript, React, Three.js/WebGPU (3D filling animation, contour plots, fiber orientation glyphs) |
| Hele-Shaw Solver | Rust (2.5D pressure-velocity FEM, frozen layer tracking) → server |
| 3D Navier-Stokes Solver | Rust + CUDA (VOF-based filling, GPU-accelerated) |
| Foam Solver | Rust (nucleation-growth ODE system coupled with flow field) |
| Structural Solver | Rust (thermo-viscoelastic FEM for shrinkage/warpage) |
| Material Fitting | Python (scipy curve_fit, rheology model calibration) |
| AI/ML | Python (PyTorch — process window prediction, gate location recommendation, surrogate warpage models) |
| Backend | Rust (Axum) + Python (FastAPI for material fitting, optimization, reporting) |
| Database | PostgreSQL (material database, mold templates, process recipes), S3 (meshes, results, animations) |
| Hosting | AWS (GPU for 3D filling, CPU for Hele-Shaw and structural, spot instances for parametric sweeps) |

---

## Monetization

### Free Tier (Student)
- Hele-Shaw filling analysis (single cavity, single gate)
- 20 common polymer materials
- Fill pattern and weld line prediction
- 3 projects

### Pro — $149/month
- Full Hele-Shaw and 3D filling simulation
- Packing, cooling, and solidification analysis
- Shrinkage and warpage prediction
- 500+ material database
- Runner and cooling channel analysis
- Report generation
- 10 projects

### Advanced — $349/user/month
- Everything in Pro
- Foam injection molding simulation
- Fiber orientation (Folgar-Tucker) and anisotropic warpage
- Thermoforming simulation
- Rubber vulcanization
- Gate and runner optimization
- Process window DOE
- API access
- Unlimited projects

### Enterprise — Custom
- On-premise deployment
- Custom material characterization and model calibration
- Integration with mold design CAD (Siemens NX Mold Wizard, CATIA)
- Multi-plant process standardization and recipe management
- Training, certification, and dedicated polymer process support

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | FoamSim Advantage |
|-----------|-----------|------------|------------------|
| Moldex3D (CoreTech) | Best 3D filling accuracy, strong fiber orientation | $25-50K/seat, desktop-only, complex UI | 10x cheaper, cloud-native, foam simulation built in |
| Moldflow (Autodesk) | Largest installed base, integrated with Fusion 360 | Hele-Shaw only (no true 3D), limited foam capability, $15-35K/seat | True 3D solver, dedicated foam module, rubber/thermoforming |
| Sigmasoft (SIGMA) | Excellent mold thermal analysis, virtual DOE | $20-40K/seat, niche market, limited foam | Broader process coverage, affordable, AI optimization |
| OpenFOAM + custom | Free solver, full flexibility | Requires PhD-level expertise, no polymer-specific workflow, months of setup | Purpose-built polymer workflow, material database, minutes to setup |
| COMSOL CFD Module | Flexible multiphysics, custom physics | $8K+ base + modules, general-purpose, no polymer material database | Polymer-native, foam nucleation/growth, material DB with 2000+ grades |

---

## MVP Scope (v1.0)

### In Scope
- Hele-Shaw (2.5D) mold filling simulation with automatic midplane extraction
- Cross-WLF viscosity model with 50 common polymer materials (PP, PE, ABS, PA6, PA66, PC, POM, PBT)
- Fill pattern visualization with weld line and air trap prediction
- Basic packing and cooling simulation
- Shrinkage prediction (isotropic) for simple geometries
- Single-cavity, single-gate analysis
- Gate location comparison (evaluate 2-3 gate positions)

### Out of Scope (v1.1+)
- Full 3D Navier-Stokes filling
- Foam nucleation and growth simulation
- Fiber orientation and anisotropic warpage
- Thermoforming and rubber vulcanization
- Runner balancing and cooling optimization
- Process window DOE and AI optimization

### MVP Timeline: 14-18 weeks
