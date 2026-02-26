# OptiLens — Optical System Design and Ray Tracing Platform

## Executive Summary

OptiLens is a cloud-based optical engineering platform for designing lenses, mirrors, and complete optical systems with sequential and non-sequential ray tracing, tolerance analysis, and stray light evaluation. It replaces Zemax OpticStudio ($5K-$30K/seat), Code V ($25K+/seat), and FRED ($10K+/seat) for camera lens design, laser systems, illumination, and AR/VR optics.

---

## Problem Statement

**The pain:**
- Zemax OpticStudio costs $5,000-$30,000/seat/year depending on edition; Code V (Synopsys) costs $25,000-$60,000/seat; these two tools dominate a $1.2B market with duopoly pricing
- Optical design requires deep expertise in aberration theory and optimization — the learning curve for Zemax alone is 6-12 months
- AR/VR/MR optics (waveguides, diffractive elements, freeform surfaces) are pushing beyond what traditional sequential ray tracing can handle, requiring expensive non-sequential and physical optics add-ons
- Tolerance analysis and manufacturing sensitivity studies are critical but tedious, often consuming 50% of total design time
- Collaboration between optical designers and mechanical engineers (for housing/mounting) requires manual exchange of STEP files and drawings

**Current workarounds:**
- University labs share 2-3 Zemax licenses across 20+ researchers
- Hobbyist telescope designers use free tools like WinLens or manual spreadsheet calculations
- Some teams use open-source ray tracers (KrakenOS for Python) but with limited optimization and no GUI
- Camera phone companies hire optical engineers at $200K+/year because the tools are too expensive for contractors

**Market size:** The optical design software market is approximately $1.2 billion (2024). There are 100,000+ optical engineers worldwide across consumer electronics (camera modules), automotive (LiDAR, HUD), defense (targeting, surveillance), biomedical (endoscopes, microscopes), telecom (fiber optics), and AR/VR.

---

## Target Users

### Primary Personas

**1. Wei — Camera Module Optical Designer at a Smartphone Company**
- Designs 5-7 element plastic aspheric lens systems for phone cameras
- Uses Zemax OpticStudio Premium but needs more licenses for growing team
- Needs: fast sequential ray tracing with aspheric/freeform optimization, ghost analysis, and tolerance sensitivity

**2. Dr. Fernandez — Laser Systems Engineer**
- Designs laser beam delivery systems, resonators, and fiber coupling optics
- Uses Code V for sequential design and FRED for stray light analysis (two separate expensive tools)
- Needs: integrated sequential design + non-sequential stray light in one platform with Gaussian beam propagation

**3. Maya — AR/VR Optics Researcher**
- Designing diffractive waveguide combiners for AR glasses
- Current tools (Zemax) lack adequate diffractive/waveguide simulation; uses custom MATLAB code to supplement
- Needs: physical optics propagation, diffractive element modeling, and waveguide ray tracing in a single tool

---

## Solution Overview

OptiLens is a cloud-based optical design platform that:
1. Provides sequential ray tracing with optimization for imaging systems (cameras, telescopes, microscopes, projection lenses)
2. Runs non-sequential ray tracing for illumination design, stray light analysis, and complex optical systems with scattering and diffraction
3. Offers physical optics propagation (POP) for Gaussian beams, diffraction, and wavefront analysis
4. Performs tolerance analysis (sensitivity, inverse sensitivity, Monte Carlo) to predict manufacturing yield
5. Generates optical prescriptions, drawings, and reports with integration to mechanical CAD for housing design

---

## Core Features

### F1: Sequential Ray Tracing and Design
- Surface types: spherical, conic, even/odd asphere, Q-type asphere, XY polynomial, Zernike, NURBS freeform, gradient-index (GRIN), diffractive (binary, kinoform), holographic, toroidal
- Glass catalog: Schott, Ohara, HOYA, CDGM, custom materials with Sellmeier/Schott dispersion models
- Afocal and focal system modes
- Multi-configuration (zoom, scan, multiple field points, switchable)
- Optimization: damped least-squares (DLS), orthogonal descent, global search (genetic algorithm, simulated annealing)
- Operands: ray aberrations, wavefront, MTF, spot size, encircled energy, distortion, lateral color, field curvature, pupil aberrations, user-defined
- Automatic lens design starting points from patent database and AI-generated initial designs

### F2: Aberration Analysis
- Seidel (third-order) aberration coefficients with contribution by surface
- Wavefront analysis: Zernike decomposition, OPD fans, wavefront maps
- Spot diagram, geometric encircled energy, Airy disk overlay
- MTF: geometrical and diffraction, through-focus, through-frequency, sagittal/tangential
- Field curvature and distortion plots
- Chromatic aberration: transverse and longitudinal color
- Pupil analysis: pupil aberration, vignetting
- Extended source analysis for non-point object imaging

### F3: Non-Sequential Ray Tracing
- Geometry: imported CAD (STEP, IGES), parametric solids (lens, mirror, prism, lightpipe, fiber)
- Surface properties: reflective, refractive, absorbing, scattering (Lambertian, Harvey-Shack, BSDF, ABg), fluorescent
- Source models: point, extended, file-based (ray file), LED/laser near-field and far-field data
- Detector analysis: irradiance, illuminance, intensity, luminance, color (CIE chromaticity), coherent field
- Stray light analysis: ghost reflection identification, narcissus analysis, critical/illuminated object identification
- Bulk scattering and absorption in optical media
- Coating models: thin-film multilayer with real coating prescriptions
- Polarization ray tracing: Jones matrix, Mueller matrix, birefringence

### F4: Physical Optics Propagation
- Gaussian beam propagation through optical systems
- Scalar diffraction: Fresnel, Fraunhofer, angular spectrum methods
- Beam quality metrics: M², beam waist, divergence, Rayleigh range
- Coherent beam combining and interference
- Waveguide mode analysis
- Diffractive optical element (DOE) design and efficiency calculation
- AR/VR waveguide simulation: in-coupling, expansion, out-coupling via surface relief gratings

### F5: Tolerancing
- Sensitivity analysis: compute change in performance for each tolerance parameter
- Inverse sensitivity: find tolerance that causes specified performance change
- Monte Carlo: simulate manufacturing variability with 1,000-100,000 random systems
- Tolerance types: radius, thickness, decenter, tilt, irregularity, index, Abbe number, wedge, element spacing
- Compensator optimization during Monte Carlo (refocus, respace)
- Yield estimation: percent of manufactured systems meeting spec
- Cost function integration: tighter tolerances = higher cost; optimize cost vs. yield

### F6: Output and Integration
- Optical prescription table export (CSV, JSON)
- 2D and 3D system layout drawings with ray fans
- Lens drawing generation per ISO 10110 with surface quality and tolerance annotations
- STEP export of lens elements for mechanical CAD integration
- Zemax ZMX file import/export for compatibility
- Report generation with all analysis results
- API for automated design optimization and batch processing

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | TypeScript, React, WebGL/WebGPU (3D system layout, ray visualization) |
| Ray Tracer | Rust (sequential + non-sequential ray tracing engine) → WASM + server |
| Optimizer | Rust (DLS, global search) with parallel evaluation |
| Physical Optics | Rust (FFT-based propagation, angular spectrum) → GPU-accelerated |
| Glass Database | PostgreSQL (glass catalogs, coating data) |
| Backend | Rust (Axum) for compute-heavy, Python (FastAPI) for reporting |
| Compute | AWS (GPU for non-sequential and POP, CPU for sequential) |
| Database | PostgreSQL (designs, glass data), S3 (ray files, results) |

---

## Monetization

### Free Tier (Student/Hobbyist)
- Sequential ray tracing (up to 10 surfaces)
- Spherical and conic surfaces only
- Spot diagram, MTF, ray fan plots
- 1 glass catalog (Schott)
- 3 projects

### Pro — $149/month
- Unlimited surfaces, all surface types
- Full optimization (DLS + global search)
- Non-sequential ray tracing
- All glass catalogs
- Tolerancing (sensitivity + Monte Carlo)
- Lens drawing generation

### Advanced — $349/user/month
- Everything in Pro
- Physical optics propagation
- Polarization ray tracing
- Stray light analysis with BSDF
- AR/VR waveguide simulation
- API access
- Collaborative editing

### Enterprise — Custom
- On-premise deployment
- Custom surface types and materials
- Patent database integration
- Integration with PLM/CAD systems
- Training and certification

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | OptiLens Advantage |
|-----------|-----------|------------|-------------------|
| Zemax OpticStudio | Industry standard, comprehensive | $5-30K/seat, desktop, bloated | Cloud-native, modern UX, AI starting points |
| Code V (Synopsys) | Best optimization for complex systems | $25-60K/seat, enterprise-only | 20x cheaper, accessible to startups |
| FRED (Photon Engineering) | Best stray light analysis | $10K+/seat, non-sequential only | Integrated seq + non-seq + POP |
| TracePro | Good illumination design | Limited optimization, $5K+ | Full optimization + illumination |
| Open-source (KrakenOS) | Free, Python-based | No GUI, limited features | Professional UI, complete feature set |

---

## MVP Scope (v1.0)

### In Scope
- Sequential ray tracing with spherical, conic, and aspheric surfaces
- Schott + Ohara glass catalogs
- DLS optimization with spot size, wavefront, and MTF operands
- Spot diagram, MTF, ray fan, wavefront plots
- System layout drawing (2D)
- Prescription table export
- Zemax ZMX import

### Out of Scope (v1.1+)
- Non-sequential ray tracing
- Physical optics propagation
- Tolerancing
- Freeform surfaces
- Stray light analysis
- AR/VR waveguide

### MVP Timeline: 14-18 weeks
