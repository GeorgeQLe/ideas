# RadarForge — Radar System Design and Signal Processing Simulation

## Executive Summary

RadarForge is a cloud-based radar systems engineering platform for designing, simulating, and analyzing radar waveforms, antenna patterns, signal processing chains, and target detection performance. It replaces MATLAB Radar Toolbox ($3K+), Remcom XFdtd ($25K+/seat), and custom in-house simulation frameworks used by defense contractors.

---

## Problem Statement

**The pain:**
- Radar system simulation requires MATLAB + Phased Array System Toolbox + Radar Toolbox + Signal Processing Toolbox = $15,000+/seat/year, plus separate antenna simulation tools
- Defense contractors maintain proprietary radar simulation frameworks (100K+ lines of custom code) that are expensive to maintain, poorly documented, and create single-points-of-failure when key engineers leave
- Automotive radar (77 GHz FMCW) is exploding but automotive engineers lack radar-specific simulation expertise
- No single tool covers the full radar system from antenna design → waveform generation → propagation → target interaction → receiver processing → detection/tracking

**Current workarounds:**
- MATLAB for signal processing but lacking electromagnetic and propagation modeling
- CST/HFSS ($20-40K/seat) for antenna design but no system-level radar simulation
- Custom Python scripts that are fragile and unvalidated
- Excel spreadsheets for radar range equation calculations

**Market size:** The radar simulation market is approximately $1.8 billion (2024) within the broader defense electronics simulation segment. Automotive radar alone is a $12B market growing 20% CAGR, creating massive demand for radar engineering tools. There are 100,000+ radar engineers worldwide across defense, automotive, aviation, maritime, and weather applications.

---

## Target Users

### Primary Personas

**1. Lt. Col. (Ret.) Patterson — Radar Systems Engineer at a Defense Contractor**
- Designing next-gen AESA radar for fighter aircraft
- Maintains a 200,000-line MATLAB codebase for radar simulation
- Needs: validated radar simulation platform to replace fragile custom code, with ITAR-compliant hosting

**2. Yuki — Automotive Radar Engineer**
- Designing 77 GHz FMCW corner radar for ADAS
- Has RF/antenna background but limited radar signal processing expertise
- Needs: end-to-end FMCW radar simulation with automotive-specific scenarios (vehicles, pedestrians, road clutter)

**3. Dr. Singh — Weather Radar Researcher**
- Studies dual-polarization weather radar for improved precipitation estimation
- Needs to simulate radar returns from hydrometeors at different polarizations
- Needs: polarimetric radar simulation with atmospheric propagation and hydrometeor scattering models

---

## Solution Overview

RadarForge is a cloud platform that:
1. Designs radar waveforms (pulsed, FMCW, phase-coded, OFDM) with ambiguity function analysis and spectral characterization
2. Models phased array and MIMO antenna configurations with element patterns, array factor, beam steering, and grating lobe analysis
3. Simulates target environments with RCS models, multipath propagation, clutter, jamming, and atmospheric effects
4. Processes received signals through configurable receiver chains: matched filtering, pulse compression, Doppler processing, CFAR detection, DOA estimation, and tracking
5. Evaluates system performance: detection probability (Pd), false alarm rate (Pfa), range/velocity resolution, angular resolution, and unambiguous range/velocity

---

## Core Features

### F1: Waveform Design
- Waveform library: pulsed (simple, LFM chirp, Barker, polyphase), FMCW (sawtooth, triangular, stepped), OFDM radar, noise radar
- Ambiguity function computation (2D range-Doppler) with sidelobe analysis
- Pulse compression ratio and time-bandwidth product optimization
- Spectral analysis: power spectral density, occupied bandwidth, out-of-band emissions
- Waveform scheduling for multi-function radar (search, track, TWS modes)
- Adaptive waveform selection based on target scenario

### F2: Antenna Modeling
- Array configurations: uniform linear (ULA), uniform rectangular (URA), circular, conformal, sparse/thinned
- Element pattern library: dipole, patch, horn, slot, Vivaldi, spiral
- Array factor computation with beam steering, Taylor/Chebyshev tapering, null steering
- MIMO virtual array and waveform diversity modeling
- Digital beamforming: conventional, MVDR/Capon, MUSIC, ESPRIT
- Grating lobe and scan blindness analysis
- Mutual coupling estimation (simplified model)

### F3: Environment and Propagation
- Free-space propagation with radar range equation
- Atmospheric attenuation (oxygen, water vapor) per ITU-R P.676
- Rain attenuation per ITU-R P.838
- Multipath: ground/sea surface reflection with Fresnel coefficient and roughness models
- Clutter models: land (Barrie, Billingsley), sea (GIT, NRCS), volume (chaff, weather)
- Target RCS models: Swerling cases (I-IV), canonical shapes (sphere, cylinder, flat plate, dihedral, trihedral), and imported RCS data
- Electronic countermeasures: noise jamming, deceptive jamming, DRFM repeaters
- Automotive-specific: road clutter, guardrails, vehicles, pedestrians, multi-bounce

### F4: Signal Processing Chain
- Configurable receiver block diagram: RF front-end (noise figure, dynamic range) → ADC → digital processing
- Matched filter and pulse compression with windowing (Hamming, Hann, Kaiser, Taylor)
- MTI and pulse-Doppler processing with clutter rejection
- Range-Doppler map generation and visualization
- CFAR detectors: CA-CFAR, OS-CFAR, GOCA-CFAR, 2D-CFAR
- Direction of arrival (DOA): beamscan, MVDR, MUSIC, ESPRIT, IAA
- FMCW processing: beat frequency extraction, range-velocity coupling compensation
- SAR (Synthetic Aperture Radar) imaging: range-Doppler algorithm, omega-k, back-projection

### F5: Detection and Tracking
- Detection performance: Pd vs. range curves for specified Pfa and target RCS
- ROC curve generation
- Multi-target detection with clustering and centroiding
- Tracking filters: alpha-beta, Kalman (EKF, UKF), interacting multiple model (IMM)
- Track initiation, confirmation, and deletion logic
- Data association: nearest neighbor, GNN, JPDA, MHT
- Track accuracy metrics: RMSE, track purity, track completeness

### F6: System-Level Analysis
- Radar range equation calculator with all loss terms
- Link budget analysis
- System noise temperature budget
- Dynamic range analysis (minimum detectable signal to maximum input)
- Clutter-limited vs. noise-limited performance regimes
- Probability of detection maps (geographic coverage)
- Trade study automation: sweep parameters and compare configurations
- Performance sensitivity analysis (Monte Carlo)

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | TypeScript, React, WebGL (3D antenna/environment viz), D3.js (2D plots) |
| Signal Processing | Rust (FFT, matched filtering, CFAR, beamforming) → WASM |
| Propagation | Rust (atmospheric models, multipath, clutter) |
| Tracking | Rust (Kalman filters, data association) |
| Backend | Rust (Axum) + Python (FastAPI for advanced analysis, SAR imaging) |
| Compute | AWS (CPU-optimized for Monte Carlo, GPU for SAR imaging) |
| Database | PostgreSQL (antenna/waveform libraries, projects), S3 (results, scenario data) |
| Hosting | AWS GovCloud option for ITAR-compliant deployment |

---

## Monetization

### Free Tier (Educational)
- Basic waveform design and ambiguity function
- Single antenna element patterns
- Free-space radar range equation
- Pulse compression and matched filtering

### Pro — $99/month
- Full waveform and antenna design
- Environment modeling (clutter, jamming, multipath)
- Complete signal processing chain
- Detection and tracking
- 500 cloud compute minutes/month

### Defense — $399/user/month
- Everything in Pro
- ITAR-compliant hosting (GovCloud)
- SAR imaging
- Monte Carlo performance analysis
- API access
- Custom target/clutter models

### Enterprise — Custom
- On-premise / air-gapped deployment
- Custom propagation models
- Integration with hardware test equipment
- Classified program support
- Training and certification

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | RadarForge Advantage |
|-----------|-----------|------------|---------------------|
| MATLAB Radar Toolbox | Comprehensive, programmable | $15K+ for full stack, general-purpose | Radar-native UX, 10x cheaper |
| Remcom XFdtd | Gold-standard EM simulation | $25K+/seat, antenna only (no system) | Full radar system simulation |
| STK Radar Module | Good for scenario-level analysis | $10K+, limited signal processing detail | Deeper signal processing chain |
| Custom MATLAB/Python | Tailored to specific needs | Expensive to maintain, fragile, unvalidated | Validated, maintained, collaborative |
| SystemVue (Keysight) | Hardware integration | $20K+, complex, enterprise-focused | Accessible pricing, web-based |

---

## MVP Scope (v1.0)

### In Scope
- LFM chirp and FMCW waveform design with ambiguity function
- Uniform linear array with beam steering
- Free-space propagation + basic clutter
- Matched filter, pulse-Doppler, CA-CFAR
- Range-Doppler map visualization
- Detection probability curves

### Out of Scope (v1.1+)
- SAR imaging
- Tracking algorithms
- Advanced clutter/jamming models
- MIMO radar
- GovCloud deployment
- Hardware-in-the-loop

### MVP Timeline: 14-18 weeks
