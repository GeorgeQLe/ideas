# GlassSim — Glass Forming and Optical Fiber Simulation Platform

## Executive Summary

GlassSim is a cloud-based simulation platform for glass forming processes (float glass, container glass, fiber drawing), annealing, tempering, and optical fiber preform-to-draw manufacturing. It replaces ANSYS Polyflow ($25K+/seat), proprietary glass company codes (Corning, Saint-Gobain, Schott in-house), and custom research models with browser-based glass process simulation featuring temperature-dependent viscosity (VFT), radiative heat transfer in semitransparent media, and residual stress prediction.

---

## Problem Statement

**The pain:**
- No dedicated commercial glass forming simulation software exists — engineers use general-purpose tools: ANSYS Polyflow ($25K+/seat) for viscous flow, ANSYS Mechanical for thermal stress, and custom codes for radiation in semitransparent glass, each requiring separate licenses and manual coupling
- Glass viscosity spans 12+ orders of magnitude (10^1 to 10^13 Pa·s) from melt to solid, governed by the Vogel-Fulcher-Tammann (VFT) equation; general-purpose CFD tools handle this poorly, requiring extreme stabilization and small time steps
- Radiative heat transfer in semitransparent glass (visible and near-IR wavelengths pass through, far-IR is absorbed) requires spectral band models that are not natively available in standard thermal solvers
- Optical fiber manufacturing is a $10B+ industry where draw tower parameters (temperature profile, draw speed, preform feed rate) determine fiber geometry, residual stress, and optical performance — yet simulation requires coupling fluid dynamics, radiation, and surface tension at extreme aspect ratios
- Tempering and annealing simulation requires predicting residual stress through the glass transition, coupling transient heat transfer with viscoelastic structural relaxation — a capability absent from standard FEA

**Current workarounds:**
- Glass companies (Corning, AGC, Saint-Gobain, Schott, O-I) maintain proprietary in-house Fortran/C++ codes developed over decades, which are fragile, undocumented, and tied to specific processes
- Academic researchers use custom MATLAB or Python codes for simplified 1D/2D glass flow models, which cannot handle 3D container forming or complex preform geometries
- Using ANSYS Polyflow for viscous flow coupled with ANSYS Mechanical for structural analysis, requiring manual data transfer and $50K+ in combined licenses
- Running physical trials on production lines at $10K-$100K per trial, disrupting production and consuming expensive glass melts

**Market size:** The global glass industry is worth approximately $250 billion (flat glass, container glass, specialty glass, fiberglass, optical fiber). Glass simulation software is a niche estimated at $200-$400 million, but growing rapidly as manufacturers digitize. There are an estimated 30,000+ glass process engineers worldwide, plus 10,000+ optical fiber engineers. The optical fiber market alone is $8 billion, with draw tower simulation being a critical competitive advantage.

---

## Target Users

### Primary Personas

**1. Thomas — Process Engineer at a Float Glass Manufacturer**
- Optimizes the float glass process: molten glass poured onto a tin bath, then annealed in a lehr
- Uses proprietary in-house code from the 1990s that only one retiring engineer understands
- Needs: float glass simulation predicting glass ribbon thickness, temperature profile on the tin bath, and annealing schedule optimization to minimize residual stress and optical distortion

**2. Dr. Sato — Optical Fiber R&D Engineer**
- Designs specialty optical fiber preforms and optimizes draw tower parameters for single-mode and multimode fibers
- Uses custom MATLAB code for 1D fiber draw models but needs 2D axisymmetric simulation for complex preform geometries (photonic crystal fiber, multi-core fiber)
- Needs: coupled thermal-fluid-mechanical simulation of the fiber draw process including neck-down profile, fiber tension, residual stress, and dopant diffusion effects on refractive index

**3. Elena — Container Glass Design Engineer**
- Designs glass bottles and jars, optimizing the blow-blow and press-blow forming process
- Uses trial-and-error on IS (Individual Section) machines with experienced operators; each new design requires 2-4 weeks of production trials
- Needs: container glass forming simulation predicting wall thickness distribution, thermal conditioning, and defects (thin spots, settle waves, seams)

**4. Rajesh — Automotive Glass Engineer**
- Designs tempered and laminated windshields, side windows, and panoramic roofs
- Needs to predict residual stress distribution from the tempering process and ensure compliance with safety standards (ECE R43, ANSI Z26.1)
- Needs: glass bending (gravity sag, press bending) and tempering simulation predicting final shape, residual stress magnitude and distribution, and fragmentation pattern

---

## Solution Overview

GlassSim is a cloud-based glass process simulation platform that:
1. Simulates high-viscosity glass flow with temperature-dependent viscosity (VFT, WLF) spanning 12 orders of magnitude, using stabilized finite element methods designed for creeping flow
2. Solves radiative heat transfer in semitransparent glass using spectral band models (Rosseland approximation, discrete ordinates, P1) that account for wavelength-dependent absorption in the glass and surface emission/reflection
3. Models glass forming processes: float glass (tin bath), container glass (blow-blow, press-blow), glass fiber drawing (continuous and staple), optical fiber preform-to-draw, glass bending and pressing
4. Predicts residual stress from annealing and tempering through coupled transient thermal and structural relaxation (viscoelastic) analysis across the glass transition temperature range
5. Optimizes process parameters (temperature profiles, forming speeds, cooling rates, preform feed rates) using AI-assisted design of experiments and surrogate models

---

## Core Features

### F1: Glass Viscous Flow Solver
- Temperature-dependent viscosity: Vogel-Fulcher-Tammann (VFT) equation, WLF model, Fulcher fit from viscosity measurement data
- Creeping flow (Stokes) solver: stabilized FEM for extremely high viscosity ratios (10^12 variation across domain)
- Free surface tracking: level set and ALE (Arbitrary Lagrangian-Eulerian) methods for glass-air interfaces
- Surface tension: Marangoni effects (temperature-dependent surface tension) and capillary-driven flows
- Gravity-driven flow: gob formation, glass sagging, float glass spreading on tin bath
- Non-isothermal coupling: viscous dissipation (negligible for glass) and strong thermal coupling through VFT viscosity
- Glass contact with molds: heat transfer coefficient at glass-mold interface, friction (Coulomb and viscous slip)
- Moving boundary handling: plunger motion in press-blow, parison inflation in blow-blow

### F2: Radiative Heat Transfer in Glass
- Spectral band model: glass absorption coefficient as a function of wavelength (UV cutoff, visible transparency, IR absorption edges)
- Rosseland diffusion approximation for optically thick regions (interior of thick glass)
- Discrete ordinates method (DOM) for general radiative transport in semitransparent media
- P1 spherical harmonics approximation as a computationally efficient alternative to DOM
- Surface radiation: emissivity and reflectivity at glass-air and glass-mold interfaces (Fresnel relations for semitransparent boundaries)
- Multi-band model: partition spectrum into 3-8 bands with different absorption coefficients per band
- Coupled conduction-radiation solver: radiation acts as an enhanced thermal conductivity in the glass interior
- Glass composition-dependent absorption spectra: soda-lime, borosilicate, aluminosilicate, fused silica, with colorant effects (Fe2O3, Cr2O3, CoO)
- Enclosure radiation for mold surfaces, heaters, and refractory walls surrounding the glass

### F3: Container Glass Forming
- IS (Individual Section) machine process simulation: gob loading, blank mold (press or blow), invert, blow mold, final blow
- Gob formation: shear cut from feeder, gob shape and temperature distribution
- Press-blow process: plunger pressing of parison in blank mold, reheat, final blow in blow mold
- Blow-blow process: settle blow, counter blow in blank mold, reheat, final blow
- NNPB (Narrow Neck Press and Blow) process for lightweight bottles
- Wall thickness distribution prediction throughout the forming sequence
- Thermal conditioning: reheat between blank and blow mold, infrared lamp modeling
- Defect prediction: thin spots, settle waves, out-of-round, birdswell, choke marks
- Glass-mold heat transfer with mold cooling channel effects
- Multi-gob simulation for double and triple gob machines

### F4: Float Glass and Flat Glass Processing
- Float process: molten glass pouring onto molten tin bath, spreading and equilibrium thickness
- Tin bath simulation: glass ribbon temperature profile, top heater control zones, attenuation (edge pull) for width control
- Annealing lehr simulation: controlled cooling schedule to minimize residual stress
- Residual stress prediction: membrane stress (caused by temperature non-uniformity above strain point), structural relaxation through glass transition
- Glass bending: gravity sag bending on ceramic mold, press bending between matched molds
- Temperature profile optimization for bending: achieve target curvature without optical distortion
- Tempering simulation: rapid quenching (air jets) from above Tg to create compressive surface stress
- Tempering residual stress: parabolic through-thickness stress profile, surface compression vs. core tension
- Fragmentation pattern prediction: dice count estimation from stress field (automotive safety standard compliance)
- Laminated glass: PVB interlayer modeling for windshield applications

### F5: Optical Fiber Draw Simulation
- Preform-to-fiber draw: 2D axisymmetric simulation of the neck-down region where preform (25-150mm diameter) is drawn to fiber (125 μm)
- Furnace model: graphite resistance furnace with temperature profile, radiative heating of preform
- Viscous draw: Navier-Stokes for the neck-down region with free surface and extreme aspect ratio
- Fiber tension prediction: draw force as a function of furnace temperature, draw speed, and preform feed rate
- Fiber diameter control: predict diameter variations from furnace temperature fluctuations and draw speed perturbations
- Residual stress in drawn fiber: axial tension frozen in during cooling, radial thermal stress from quenching
- Dopant diffusion: germanium, fluorine, phosphorus redistribution during draw at high temperature, affecting refractive index profile
- Coating application: UV-acrylate coating die flow, coating concentricity, wet-on-wet dual coating
- Specialty fiber: photonic crystal fiber (PCF) draw with hole deformation, multi-core fiber, polarization-maintaining fiber
- Draw tower parameter optimization: furnace temperature, draw speed, preform feed rate for target fiber properties

### F6: Annealing, Tempering, and Residual Stress
- Viscoelastic structural relaxation: Tool-Narayanaswamy-Moynihan (TNM) model for structural relaxation through glass transition
- Thermorheologically simple material model: master relaxation curve with WLF or Arrhenius shift factor
- Generalized Maxwell model for stress relaxation with multiple relaxation times (Prony series)
- Fictive temperature computation: tracks thermal history to determine glass structure (relaxed vs. quenched state)
- Annealing schedule optimization: minimize residual stress while minimizing lehr length (production speed)
- Tempering process simulation: heating to tempering temperature, quench rate (air jet velocity and coverage), resulting stress profile
- Chemical strengthening modeling: ion exchange (Na+/K+) diffusion and resulting surface compression (for Gorilla-type glass)
- Birefringence prediction from residual stress: stress-optical coefficient for optical quality assessment
- 3D residual stress in complex shapes: automotive glazing, glass containers, display panels
- Thermal shock resistance prediction from residual stress state

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | TypeScript, React, Three.js/WebGPU (3D glass forming visualization, temperature/stress contours) |
| Viscous Flow Solver | Rust (stabilized FEM for Stokes flow, ALE free surface, creeping flow) → GPU-accelerated |
| Radiation Solver | Rust (discrete ordinates, P1, Rosseland — spectral band models) |
| Structural/Relaxation Solver | Rust (viscoelastic FEM, TNM structural relaxation, generalized Maxwell) |
| Fiber Draw Solver | Rust (2D axisymmetric Navier-Stokes with free surface, coupled radiation) |
| AI/ML | Python (PyTorch — surrogate models for process optimization, DOE) |
| Backend | Rust (Axum) + Python (FastAPI for optimization, material property fitting, reporting) |
| Database | PostgreSQL (glass compositions, VFT parameters, spectral absorption data, mold templates), S3 (meshes, results) |
| Hosting | AWS (GPU for 3D forming simulation, CPU for 2D fiber draw and annealing) |

---

## Monetization

### Free Tier (Student)
- 1D glass rod/tube draw analysis
- VFT viscosity calculator for common glass compositions
- Basic annealing stress estimation (1D through-thickness)
- 3 projects

### Pro — $149/month
- 2D axisymmetric forming and draw simulation
- Radiative heat transfer (Rosseland and P1 models)
- Annealing and tempering residual stress analysis
- Glass composition database (50+ compositions with VFT and spectral data)
- Optical fiber draw simulation (standard single-mode)
- Report generation
- 15 projects

### Advanced — $349/user/month
- Everything in Pro
- Full 3D container glass forming (blow-blow, press-blow)
- Float glass and tin bath simulation
- Discrete ordinates radiation solver
- Multi-stage forming sequences
- Chemical strengthening (ion exchange) modeling
- Process optimization (DOE, surrogate models)
- API access
- Unlimited projects

### Enterprise — Custom
- On-premise deployment
- Custom glass composition characterization (VFT fitting, spectral measurement integration)
- Integration with glass plant control systems and production databases
- Multi-furnace and multi-line simulation
- Training, certification, and dedicated glass process engineering support

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | GlassSim Advantage |
|-----------|-----------|------------|-------------------|
| ANSYS Polyflow | Best general viscous flow solver, established in glass industry | $25K+/seat, no built-in glass radiation, requires manual coupling with thermal/structural | Glass-native: VFT viscosity, semitransparent radiation, residual stress in one platform |
| Proprietary company codes (Corning, Saint-Gobain, O-I) | Highly validated for specific processes, decades of development | Locked to one company, undocumented, fragile, cannot be licensed | Commercial product, documented, supported, accessible to all glass companies |
| ANSYS Fluent + DOM | General CFD with radiation capability | $30K+/seat, not designed for glass viscosity ranges, no structural relaxation | Purpose-built for glass: 12-order viscosity range, TNM relaxation, forming workflows |
| COMSOL Multiphysics | Flexible PDE framework, can build custom glass models | $8K+ base + modules, requires expert custom model development, slow for production use | Pre-built glass forming workflows, validated material database, 10x faster setup |
| Custom MATLAB/Python codes | Flexible, low cost, common in academia | 1D/2D only, no 3D forming, fragile, not validated at scale | Production-grade 3D simulation, maintained and validated, cloud-scalable |

---

## MVP Scope (v1.0)

### In Scope
- 2D axisymmetric glass viscous flow with VFT temperature-dependent viscosity
- Radiative heat transfer using Rosseland approximation for optically thick glass
- Optical fiber draw simulation: neck-down profile, fiber tension, diameter prediction
- Annealing simulation: 1D and 2D transient cooling with TNM structural relaxation and residual stress
- Glass composition database with VFT parameters for 20 common compositions (soda-lime, borosilicate, fused silica, E-glass, and others)
- Temperature and viscosity contour visualization
- Basic process parameter sensitivity analysis (vary furnace temperature, draw speed)

### Out of Scope (v1.1+)
- Full 3D container glass forming (blow-blow, press-blow)
- Float glass tin bath simulation
- Discrete ordinates and P1 radiation solvers
- Tempering and fragmentation prediction
- Chemical strengthening (ion exchange)
- Dopant diffusion in fiber draw
- AI-driven process optimization

### MVP Timeline: 14-18 weeks
