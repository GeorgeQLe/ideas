# SonarDeep — Underwater Acoustics and Sonar System Simulation

## Executive Summary

SonarDeep is a cloud-based underwater acoustics platform for sonar system design, ocean acoustic propagation modeling, and underwater communication simulation. It replaces Bellhop/KRAKEN (academic codes with no GUI), COMSOL underwater acoustics ($16K+), and proprietary defense tools with browser-based ray tracing, normal mode, and parabolic equation solvers for the underwater domain.

---

## Problem Statement

**The pain:**
- Underwater acoustic simulation tools are either academic codes (Bellhop, KRAKEN, RAM) with no GUI and Fortran source code from the 1980s, or classified/proprietary defense tools unavailable to commercial users
- The ocean acoustics domain is growing rapidly: offshore wind surveys, autonomous underwater vehicles (AUVs), fish finding, pipeline inspection, and submarine detection all need acoustic simulation
- Setting up an ocean propagation model requires oceanographic data (sound speed profiles, bathymetry, sediment properties) that is scattered across multiple databases and formats
- No commercial tool provides an integrated workflow from ocean environment definition → propagation modeling → sonar system design → detection performance prediction

**Current workarounds:**
- Using Bellhop (ray tracing) or KRAKEN (normal modes) via command-line Fortran executables with MATLAB post-processing scripts
- COMSOL for simple underwater problems but prohibitively expensive and not designed for range-dependent ocean environments
- Defense contractors using classified tools (CASS/GRAB, Neptune) unavailable to commercial and academic users
- Simplified sonar equation calculations in Excel that miss environmental effects

**Market size:** The underwater acoustics market is approximately $3.5 billion (2024) covering sonar systems, marine surveys, and underwater communications. The simulation tools segment is approximately $200 million, growing rapidly due to offshore wind, AUV, and defense spending. There are 30,000+ ocean/acoustic engineers worldwide.

---

## Target Users

### Primary Personas

**1. Dr. Martinez — Ocean Acoustics Researcher**
- Studies sound propagation in shallow water environments with variable bathymetry and sediment
- Uses Bellhop via MATLAB wrapper but spends more time on data preparation than science
- Needs: integrated environment builder with automatic oceanographic data import and multi-method propagation solvers

**2. Kai — Sonar Systems Engineer at a Defense Contractor**
- Designs sonar arrays for submarine detection and mine countermeasures
- Uses classified tools at the office but cannot work remotely or share results with academic collaborators
- Needs: unclassified sonar design platform with propagation modeling and detection performance analysis

**3. Emma — Marine Survey Engineer at an Offshore Wind Company**
- Conducts environmental impact assessments for underwater noise from pile driving
- Needs to predict sound exposure levels (SEL) for marine mammals at various distances
- Needs: underwater noise propagation with frequency-dependent absorption and marine mammal impact assessment

---

## Solution Overview

SonarDeep is a cloud-based underwater acoustics platform that:
1. Builds ocean environments from integrated databases: bathymetry (GEBCO), sound speed profiles (WOA), and sediment properties (default or user-specified)
2. Runs acoustic propagation using multiple methods: ray tracing (Bellhop-like), normal modes (KRAKEN-like), parabolic equation (RAM-like), and wavenumber integration
3. Designs sonar transducer arrays with beamforming, beam patterns, and directivity
4. Predicts sonar detection performance: signal excess, probability of detection, and optimal frequency for given scenario
5. Assesses environmental impact: underwater noise from construction, shipping, and seismic surveys with marine mammal exposure prediction

---

## Core Features

### F1: Ocean Environment Builder
- Bathymetry: import from GEBCO, custom survey data (CSV, GeoTIFF), or manual definition
- Sound speed profiles: import from World Ocean Atlas, CTD cast data, or user-defined
- Range-dependent environments: varying bathymetry, SSP, and sediment along propagation path
- Sediment models: fluid, elastic, poro-elastic bottom with Hamilton's regression for sediment properties
- Surface: flat, rough (Pierson-Moskowitz spectrum), ice-covered
- Volume attenuation: Thorp, Francois-Garrison frequency-dependent absorption
- Environment database: pre-built environments for common operating areas

### F2: Acoustic Propagation Solvers
- Ray tracing: Gaussian beam tracing with eigenray finding, convergence zones, bottom/surface interactions, caustics
- Normal modes: discrete modal sum for range-independent environments, coupled modes for range-dependent
- Parabolic equation (PE): split-step Fourier method for range-dependent 2D propagation
- Wavenumber integration: full-wave solution for benchmark and calibration
- 3D propagation: horizontal refraction (3D ray tracing, 3D PE) for environments with lateral variation
- Transmission loss maps: 2D (range-depth) and 3D (range-range-depth)
- Broadband propagation: pulse propagation with frequency integration

### F3: Sonar System Design
- Transducer element models: piston, line, ring, flextensional, parametric array
- Array configurations: linear (towed array), planar, cylindrical, conformal, sparse
- Beamforming: conventional, adaptive (MVDR), matched field processing
- Beam pattern computation: mainlobe width, sidelobe level, steering response
- Noise models: ambient (Wenz curves), shipping, wind, rain, biological, self-noise (flow, machinery)
- Reverberation: surface, bottom, volume scattering
- Signal processing: matched filter, energy detector, replica correlation

### F4: Sonar Performance Prediction
- Sonar equation: SL + DI - TL - (NL - AG) = SE (signal excess)
- Active sonar: source level, target strength, two-way TL, reverberation-limited detection
- Passive sonar: target radiated noise, one-way TL, ambient noise
- Probability of detection: Pd vs. range for specified Pfa (ROC-based)
- Optimal frequency analysis: find best operating frequency for given scenario
- Multi-static sonar: transmitter/receiver geometry optimization
- Detection range maps: geographic display of detection probability

### F5: Underwater Noise Impact Assessment
- Source models: pile driving (Reinhall, JASCO), shipping (Ross, Wales-Heitmeyer), seismic airgun, dredging
- Regulatory thresholds: NOAA, JNCC, German UBA guidelines for marine mammals
- Sound exposure level (SEL) and peak pressure (SPLpk) computation at distance
- Weighted SEL: frequency weighting functions per Southall et al. 2019 (LF cetaceans, MF cetaceans, HF cetaceans, phocids, otariids)
- Cumulative SEL from multiple pile strikes or repeated events
- Mitigation: bubble curtain, cofferdams, operational shutdowns — effectiveness modeling
- Marine mammal density data integration for population-level impact
- Environmental Impact Assessment (EIA) report generation

### F6: Underwater Communications
- Channel characterization: multipath spread, Doppler spread, coherence bandwidth/time
- Modulation schemes: FSK, PSK, OFDM, spread spectrum
- Bit error rate (BER) vs. SNR prediction with channel effects
- Acoustic modem link budget
- Network simulation: multi-node underwater acoustic network performance

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | TypeScript, React, Deck.gl/Mapbox (geographic environment viz), WebGL (2D propagation plots) |
| Ray Tracing | Rust (Gaussian beam tracing, eigenray search) → WASM + server |
| Normal Modes | Rust (KRAKEN-type modal solver) → server |
| Parabolic Equation | Rust (split-step Fourier PE) → server |
| Beamforming | Rust (array processing, beampatterns) → WASM |
| Backend | Rust (Axum) + Python (FastAPI for environment data, impact assessment) |
| Oceanographic Data | PostgreSQL + PostGIS (WOA, GEBCO, sediment databases) |
| Compute | AWS (CPU for propagation, GPU for broadband/3D PE) |
| Database | PostgreSQL + PostGIS (environments, sonar specs), S3 (propagation results) |

---

## Monetization

### Academic — $79/month per lab
- All propagation solvers (ray, modes, PE)
- Ocean environment builder with GEBCO/WOA
- Basic beamforming
- 5 user seats

### Pro — $199/month
- Sonar system design (full array modeling)
- Detection performance prediction
- Underwater noise impact assessment
- Marine mammal impact reporting
- API access

### Defense — $399/user/month
- Everything in Pro
- 3D propagation
- Multi-static sonar optimization
- Broadband propagation
- Matched field processing
- Unlimited compute

### Enterprise — Custom
- On-premise / classified hosting
- Custom environment databases
- Hardware-in-the-loop integration
- Underwater communications simulation
- Training and certification

---

## MVP Scope (v1.0)

### In Scope
- Ocean environment builder (bathymetry, SSP, sediment — range-independent)
- Ray tracing propagation solver with transmission loss
- GEBCO bathymetry and WOA sound speed integration
- 2D transmission loss visualization (range-depth)
- Basic sonar equation calculator
- Active and passive sonar Pd vs. range

### Out of Scope (v1.1+)
- Normal modes and PE solvers
- Range-dependent environments
- Array design and beamforming
- Underwater noise impact assessment
- 3D propagation
- Communications simulation

### MVP Timeline: 12-16 weeks
