# Acoustica — Architectural and Environmental Acoustics Simulation Platform

## Executive Summary

Acoustica is a cloud-based acoustics engineering platform for room acoustics simulation, noise propagation modeling, and acoustic design optimization. It replaces ODEON ($8K+/seat), EASE ($5K+/seat), and CadnaA ($10K+/seat) with browser-based ray tracing, AI-optimized material placement, and real-time auralization.

---

## Problem Statement

**The pain:**
- ODEON costs $8,000-$15,000/seat for room acoustics; CadnaA costs $10,000-$25,000/seat for environmental noise; EASE costs $5,000-$10,000/seat for sound system design — three separate tools for three aspects of acoustics
- Acoustic simulation requires specialized knowledge of ray tracing, wave-based methods, material absorption coefficients, and psychoacoustic metrics that few engineers possess
- Concert halls, theaters, and recording studios are built with $100M+ budgets but acoustic design is still partly art, partly science — simulation validation is poor
- Environmental noise assessments (EIA) for roads, railways, airports, and industrial facilities are legally required in most countries but simulation tools are expensive and complex
- Building acoustics (sound insulation, flanking transmission) requires separate tools from room acoustics, creating workflow fragmentation

**Current workarounds:**
- Using simplified hand calculations (Sabine/Eyring) that fail for complex geometries and non-diffuse sound fields
- Hiring specialized acoustic consultants ($200-$500/hour) for tasks that could be partially automated
- Using SketchUp + limited free plugins for rough room acoustic estimates
- Open-source tools (Pachyderm for Rhino, I-Simpa) have limited capabilities and small user communities

**Market size:** The acoustic simulation market is approximately $600 million (2024). There are 50,000+ acoustic engineers/consultants worldwide. The construction industry ($13T globally) requires acoustic compliance for virtually every building project.

---

## Target Users

### Primary Personas

**1. Anna — Acoustic Consultant**
- Designs room acoustics for concert halls, theaters, conference rooms, and recording studios
- Uses ODEON for room acoustics simulations
- Needs: fast ray-tracing simulation with auralization, material optimization, and client-friendly 3D visualization

**2. Miguel — Environmental Noise Specialist**
- Performs noise impact assessments for road, rail, and industrial projects
- Uses CadnaA for outdoor noise propagation
- Needs: outdoor noise modeling compliant with ISO 9613, CNOSSOS-EU, and FHWA TNM, with GIS-integrated terrain and building data

**3. Kenji — AV/Sound System Designer**
- Designs PA systems and loudspeaker installations for venues, houses of worship, and stadiums
- Uses EASE for loudspeaker coverage prediction
- Needs: integrated room acoustics + sound system design with real-time SPL mapping and speech intelligibility metrics

---

## Solution Overview

Acoustica is a cloud-based acoustics platform that:
1. Imports 3D geometry from CAD (IFC/STEP/OBJ) or builds simplified room models directly, with a material database of 2,000+ acoustic absorption/diffusion coefficients
2. Runs GPU-accelerated geometrical acoustics (ray tracing + image source) for room impulse response prediction, with wave-based corrections for low frequencies (FEM/BEM)
3. Performs outdoor noise propagation using ISO 9613-2, CNOSSOS-EU, FHWA TNM, and NMPB methods with terrain, barriers, buildings, and meteorological effects
4. Provides real-time auralization: listen to how a room sounds from any position with specified source signals (speech, music, noise)
5. Optimizes acoustic treatment placement: AI recommends absorber/diffuser locations and types to meet target reverberation time, clarity, and STI criteria

---

## Core Features

### F1: Room Acoustics Simulation
- Geometrical acoustics: ray tracing (stochastic), image source method (specular), and hybrid (ISM + late rays)
- Material database: 2,000+ materials with frequency-dependent absorption (125 Hz - 8 kHz) and scattering coefficients
- Room acoustic parameters: RT60 (T20, T30, EDT), C50, C80, D50, D80, STI, RASTI, G (strength), LF (lateral fraction), IACC
- Spatial impulse response computation at receiver positions
- Source directivity patterns: omnidirectional, cardioid, loudspeaker manufacturer data (CLF/GLL)
- Parametric studies: sweep material configurations and evaluate impact on metrics
- Wave-based low-frequency correction (FEM/BEM) for frequencies below Schroeder frequency

### F2: Auralization
- Convolution-based auralization: convolve dry source signals with simulated impulse responses
- Binaural rendering using HRTF for headphone listening
- Ambisonics rendering for spatial audio playback
- Source signals library: speech (multiple languages), music (orchestral, pop, organ), noise (traffic, HVAC)
- Real-time interactive mode: move listener position and hear changes instantly
- A/B comparison: listen to "before treatment" vs. "after treatment"
- Auralization export as WAV/FLAC for client presentations

### F3: Environmental Noise Modeling
- Propagation methods: ISO 9613-2, CNOSSOS-EU (road, rail, industrial, aircraft), FHWA TNM, NMPB 2008, RLS-19
- Terrain import from DEM/DTM (GeoTIFF, LiDAR)
- Building footprints from OpenStreetMap or GIS shapefiles
- Source models: road traffic (AADT, speed, composition), railway, industrial (point, line, area sources), aircraft flight paths
- Barrier attenuation: thin screen, berm, building, T-top, combination barriers
- Meteorological effects: wind, temperature gradients (Harmonoise/Nord2000)
- Noise map generation: grid receivers with Lday, Levening, Lnight, Lden contour maps
- Facade noise levels and assessment per WHO/EU guidelines

### F4: Sound System Design
- Loudspeaker placement and aiming on 3D room model
- Loudspeaker directivity data import (CLF, GLL, EASE GLL, manufacturer-specific)
- SPL coverage maps at listener plane with frequency-dependent resolution
- Speech intelligibility prediction (STI, ALCONS) with room acoustics effects
- Delay alignment and signal processing chain modeling
- Subwoofer placement optimization for uniform low-frequency coverage
- Line array mechanical design: splay angles, rigging loads
- System equalization preview

### F5: Building Acoustics
- Airborne sound insulation prediction: EN 12354-1 / ISO 15712-1 (single and double leaf walls, flanking)
- Impact sound insulation: EN 12354-2 / ISO 15712-2
- Material/construction database: common wall and floor assemblies with lab-tested STC/Rw values
- Flanking transmission analysis: identify weak paths in building construction
- HVAC noise: duct breakout, terminal device noise, room noise criteria (NC, RC, NR)
- Code compliance: IBC, British Building Regulations Part E, DIN 4109, Australian NCC

### F6: Optimization and Reporting
- AI-optimized treatment placement: specify target RT60 and budget, get recommended absorber/diffuser layout
- Multi-objective optimization: balance RT60, C80, STI, and cost
- Acoustic report generation: room acoustic parameters table, noise contour maps, code compliance summary
- Client-facing 3D visualization: interactive model with color-coded SPL/RT60 overlays
- Export to IFC (BIM integration) with acoustic properties
- PDF report with methodology description, results, and regulatory compliance

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | TypeScript, React, Three.js/WebGPU (3D room viz, noise maps), Web Audio API (auralization) |
| Ray Tracing | Rust (GPU-accelerated ray tracing via wgpu/compute shaders) |
| Wave Solver | Rust (FEM for room modes) → server-side for large problems |
| Noise Propagation | Rust (ISO 9613, CNOSSOS implementation) → WASM + server |
| Auralization | Rust (convolution, HRTF processing) → WASM, Web Audio API |
| Optimization | Python (Bayesian optimization for treatment placement) |
| Backend | Rust (Axum) + Python (FastAPI for optimization/reporting) |
| GIS | PostGIS, Mapbox GL JS for terrain/building integration |
| Database | PostgreSQL + PostGIS (materials, projects), S3 (impulse responses, audio) |

---

## Monetization

### Free Tier (Student)
- Simple shoebox room simulation (1 source, 1 receiver)
- RT60 calculation (Sabine/Eyring + ray tracing)
- Basic auralization
- 3 projects max

### Professional — $129/month
- Complex room geometries (CAD import)
- Full room acoustic parameter set
- Environmental noise (1 propagation method)
- Sound system design (up to 20 sources)
- Report generation

### Advanced — $299/user/month
- All propagation methods (ISO 9613, CNOSSOS, FHWA TNM)
- Building acoustics
- AI optimization
- Unlimited sources
- API access
- Collaborative editing

### Enterprise — Custom
- On-premise for large consultancies
- Custom propagation model integration
- BIM integration (IFC round-trip)
- White-label reports
- Training and certification

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | Acoustica Advantage |
|-----------|-----------|------------|---------------------|
| ODEON | Gold standard room acoustics | $8-15K/seat, desktop, no outdoor noise | Integrated indoor+outdoor, web-based |
| CadnaA (DataKustik) | Best environmental noise | $10-25K/seat, no room acoustics | Combined room+environmental acoustics |
| EASE (AFMG) | Strong sound system design | $5-10K, dated UI, limited room acoustics | Modern UI, full acoustic scope |
| COMSOL Acoustics | Wave-based accuracy | $12K+, general-purpose (not acoustics-focused) | Acoustics-native workflow, faster |
| Pachyderm (Rhino) | Free, Rhino integration | Limited features, Rhino-dependent | Standalone, comprehensive |

---

## MVP Scope (v1.0)

### In Scope
- Shoebox and imported OBJ room geometry
- Ray tracing simulation with material database (500 materials)
- RT60, C80, D50, STI calculation
- Basic auralization (binaural, headphone)
- Source/receiver placement
- PDF report with parameters table

### Out of Scope (v1.1+)
- Environmental noise propagation
- Sound system design with loudspeaker data
- Building acoustics
- AI treatment optimization
- Wave-based FEM solver

### MVP Timeline: 12-16 weeks
