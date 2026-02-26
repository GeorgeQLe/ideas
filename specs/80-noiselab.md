# NoiseLab — Vehicle NVH Analysis Platform

## Executive Summary

NoiseLab is a cloud-based Noise, Vibration, and Harshness (NVH) platform for automotive, aerospace, and industrial equipment engineers. It replaces Siemens LMS Virtual.Lab ($40K+/seat), Actran ($25K+/seat), HEAD acoustics ArtemiS ($20K+/seat), and AutoSEA ($15K+/seat) with an integrated browser-based environment for structural vibration analysis, acoustic radiation, transfer path analysis, and psychoacoustic evaluation.

---

## Problem Statement

**The pain:**
- Siemens LMS Virtual.Lab costs $40,000-$80,000/seat/year; Actran (FFT) costs $25,000-$50,000/seat; HEAD acoustics ArtemiS costs $20,000-$40,000 for a hardware+software bundle — NVH requires multiple tools across structural, acoustic, and psychoacoustic domains
- NVH is the #2 reason for vehicle warranty claims after powertrain issues; poor NVH quality costs OEMs $500M-$2B per year in warranty and customer satisfaction losses
- Transfer path analysis (TPA) is the core NVH diagnostic method but requires separate software for measurement, analysis, and synthesis — no single tool covers the complete TPA workflow
- Statistical energy analysis (SEA) for mid-high frequency NVH and FEA-based analysis for low frequencies require completely different tools and expertise, creating a frequency-gap problem
- EV transition is creating entirely new NVH challenges (electric motor whine, inverter noise, single-speed gearbox tonality) that legacy ICE-focused NVH tools do not address well

**Current workarounds:**
- Using MATLAB/Python scripts with custom TPA implementations that are fragile, undocumented, and not transferable between engineers
- Purchasing separate tools for each frequency range: FEA (Nastran/Abaqus) for 0-300 Hz, SEA (AutoSEA/VA One) for 1 kHz+, and hoping the mid-frequency gap resolves itself
- Relying on physical prototypes and subjective jury evaluations ($200K-$500K per prototype vehicle) rather than virtual NVH prediction
- Using HEAD acoustics or Bruel & Kjaer hardware/software bundles for measurement-based NVH, with no connection to simulation models

**Market size:** The automotive NVH market is approximately $2.5 billion (2024), including testing equipment, simulation software, and consulting. There are 80,000+ NVH engineers globally across automotive OEMs, Tier 1 suppliers, aerospace, and industrial equipment. The EV transition is driving 20%+ growth in NVH simulation demand.

---

## Target Users

### Primary Personas

**1. Maria — NVH Engineer at an Automotive OEM**
- Responsible for body structure NVH targets: interior noise at driver's ear, steering wheel vibration, boom/drone frequencies
- Uses LMS Virtual.Lab for FE-based NVH and Test.Lab for measurement TPA
- Needs: integrated simulation + test correlation, transfer path analysis, contribution ranking, and target cascading from vehicle level to component level

**2. Raj — Acoustic Engineer at an EV Startup**
- Developing NVH solutions for electric drivetrain whine, road noise (dominant in EVs), and wind noise
- Has limited budget; uses MATLAB scripts and open-source tools
- Needs: affordable TPA and sound quality analysis, electric motor order tracking, psychoacoustic metrics (loudness, sharpness, roughness, tonality)

**3. Claudia — NVH Consultant at an Engineering Services Firm**
- Performs NVH troubleshooting for multiple OEM and industrial clients across automotive, heavy equipment, and HVAC
- Uses a mix of LMS Test.Lab, AutoSEA, and custom MATLAB code
- Needs: flexible platform supporting both measurement-based and simulation-based NVH, fast reporting, and client-friendly sound demonstrations

---

## Solution Overview

NoiseLab is a cloud-based NVH platform that:
1. Performs structural-acoustic FEA for low-frequency NVH (0-500 Hz): modal analysis, forced response, acoustic cavity coupling, panel contribution
2. Runs statistical energy analysis (SEA) for mid-high frequency NVH (200 Hz - 10 kHz) with hybrid FE-SEA bridging the frequency gap
3. Implements full transfer path analysis (TPA): classical TPA, operational TPA (OTPA), component-based TPA (CB-TPA), and blocked-force TPA
4. Provides psychoacoustic evaluation: loudness (Zwicker), sharpness, roughness, fluctuation strength, tonality, articulation index — with real-time auralization
5. Delivers AI-driven NVH target cascading and root cause identification from vehicle-level targets down to component specifications

---

## Core Features

### F1: Structural-Acoustic FEA
- Modal analysis: natural frequencies and mode shapes for structure and acoustic cavities (up to 500 Hz)
- Coupled structural-acoustic analysis: structure vibration → cavity pressure (interior noise prediction)
- Panel contribution analysis: rank which body panels contribute most to interior sound pressure at driver's ear
- Frequency response functions (FRFs): point/transfer mobility, acoustic transfer functions (ATFs)
- Forced response: harmonic, random, and transient excitation (engine orders, road input, impact)
- Mesh import from Nastran, Abaqus, ANSYS (BDF, INP, CDB formats) or auto-mesh from STEP/STL
- Acoustic radiation: structural surface velocity → radiated sound power and directivity (BEM/FMM)
- Trim body modeling: mass-spring-damper representation of acoustic treatments (damping mats, absorbers, barriers)

### F2: Statistical Energy Analysis (SEA) and Hybrid FE-SEA
- SEA model builder: automatic subsystem generation from CAD (plates, cavities, beams)
- Coupling loss factor (CLF) estimation: analytical (point, line, area junctions), experimental
- Power input methods: mechanical point force, diffuse acoustic field, turbulent boundary layer (wind noise)
- Material database: damping loss factors, radiation efficiency, wave speeds for 500+ materials (steel, aluminum, composites, glass, rubber, acoustic packages)
- Hybrid FE-SEA: deterministic FE subsystems at low frequency coupled with statistical SEA subsystems at high frequency, with automatic crossover
- SEA response prediction: spatially averaged velocity, sound pressure, transmitted power
- Monte Carlo uncertainty quantification for SEA predictions
- Acoustic package optimization: insertion loss, absorption, and mass targets

### F3: Transfer Path Analysis (TPA)
- Classical TPA: measured FRFs × operational forces (inverse matrix method) → path contributions
- Operational TPA (OTPA): output-only method using transmissibility matrices from operational data
- Component-based TPA (CB-TPA): blocked force characterization at source-receiver interfaces
- Blocked force TPA: in-situ measurement of blocked forces using matrix inversion
- Source characterization: operational forces, mount stiffness, blocked force spectra
- Path ranking: waterfall/contribution bar charts showing dominant paths per frequency
- Synthesis and auralization: reconstruct target response from individual path contributions — listen to each path separately
- Order tracking integration: extract engine/motor order contributions for rotating machinery NVH
- Road noise TPA: tire-road interface forces → body attachment points → interior noise

### F4: Psychoacoustic Analysis and Sound Quality
- Loudness: Zwicker (ISO 532-1), Moore-Glasberg
- Sharpness: Aures, von Bismarck, DIN 45692
- Roughness and fluctuation strength: Daniel-Weber model
- Tonality: Aures, Terhardt, ECMA-418 Prominence Ratio and Tone-to-Noise Ratio
- Articulation Index (AI) and Speech Transmission Index (STI)
- Time-varying psychoacoustic metrics: loudness vs. time, N5/N10 percentiles
- Sound quality KPI dashboard: spider/radar chart of psychoacoustic metrics vs. targets
- Jury evaluation support: A/B comparison with blinding, paired comparison, semantic differential scales
- Auralization: playback of simulated or measured sounds with binaural rendering (HRTF)
- Sound design: target sound definition using psychoacoustic metric envelopes, active sound generation (ASG/AVAS) preview

### F5: EV-Specific NVH
- Electric motor electromagnetic NVH: radial force harmonics from motor order analysis (spatial and temporal orders)
- Inverter switching noise: PWM frequency, random modulation effects
- Gear whine: transmission error → mesh force → housing radiation (single-speed and multi-speed)
- High-voltage battery pack: structural modes, buzz/squeak/rattle (BSR) risk from cell swelling
- Regenerative braking noise and vibration
- AVAS (Acoustic Vehicle Alerting System) compliance: UN R138 minimum sound requirements
- Road noise dominance analysis: tire cavity resonance, pattern noise, impact harshness
- Wind noise: A-pillar, mirror, seal gap contribution via SEA + CFD-coupled methods

### F6: NVH Target Cascading and Reporting
- Vehicle-level targets: interior noise dB(A) at speed, vibration mm/s at steering wheel, sound quality metrics
- Target decomposition: vehicle → subsystem → component → mount/interface specification
- Sensitivity analysis: which component changes yield the largest NVH improvement per dollar
- AI-assisted root cause identification: upload measurement data, automatically identify tonal issues, resonances, and path dominance
- Automated NVH report generation: target vs. actual comparison, contribution analysis, psychoacoustic summary
- Benchmark database: compare your vehicle's NVH signature against anonymized industry benchmarks
- Export: PDF reports, PowerPoint summaries, CSV/UFF data, WAV/FLAC auralization files

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | TypeScript, React, Three.js/WebGPU (3D mode shape animation, contour plots), Web Audio API (auralization/playback) |
| Structural FEA | Rust (modal solver, forced response, coupled structural-acoustic) → server-side HPC |
| SEA Engine | Rust (SEA power balance solver, CLF computation, hybrid FE-SEA coupling) |
| Acoustic BEM | Rust + CUDA (boundary element method with fast multipole, GPU-accelerated) |
| TPA Engine | Rust + Python (matrix inversion, OTPA, CB-TPA, order tracking) → WASM for interactive path ranking |
| Psychoacoustics | Rust (Zwicker loudness, sharpness, roughness, tonality) → WASM for real-time evaluation |
| AI/ML | Python (PyTorch — root cause identification, anomaly detection, target cascading optimization) |
| Backend | Rust (Axum) + Python (FastAPI for ML inference, reporting) |
| Database | PostgreSQL (materials, damping data, project metadata), S3 (FRF datasets, audio files, meshes) |
| Hosting | AWS (GPU instances for BEM, HPC for FEA, standard for SEA/TPA) |

---

## Monetization

### Free Tier (Student)
- Single-point FRF viewer and basic modal analysis (up to 50 modes)
- Psychoacoustic metrics on uploaded WAV files (3 files/day)
- Basic loudness and sharpness calculation
- 2 projects max

### Pro — $179/month
- Structural-acoustic FEA (models up to 100K DOF)
- Classical TPA and OTPA
- Full psychoacoustic metric suite
- SEA (up to 50 subsystems)
- Order tracking
- Report generation
- 10 projects

### Advanced — $399/user/month
- Unlimited model size (cloud HPC)
- Hybrid FE-SEA
- Component-based TPA and blocked force TPA
- EV-specific modules (e-motor, gear whine, AVAS)
- AI root cause identification
- Target cascading
- API access and collaborative editing
- Unlimited projects

### Enterprise — Custom
- On-premise deployment for OEM data security
- Custom material and damping databases
- Integration with PLM/CAE toolchains (Nastran, Abaqus result import)
- White-label reporting
- Dedicated HPC allocation and priority compute

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | NoiseLab Advantage |
|-----------|-----------|------------|-------------------|
| Siemens LMS Virtual.Lab | Industry standard for automotive NVH, deep FEA integration | $40-80K/seat, requires Simcenter platform, steep learning curve | Browser-based, 10x cheaper, integrated TPA+SEA+psychoacoustics |
| Actran (FFT/MSC) | Best-in-class acoustic FEA/BEM | $25-50K/seat, acoustic-only (no TPA, no SEA), complex setup | Full NVH workflow including TPA and sound quality |
| AutoSEA / VA One (ESI) | Mature SEA with hybrid FE-SEA | $15-30K/seat, SEA-only, no psychoacoustics, no TPA | Unified platform: FEA + SEA + TPA + psychoacoustics |
| HEAD acoustics ArtemiS | Excellent psychoacoustic analysis, hardware integration | $20-40K, measurement-only (no simulation), proprietary hardware | Simulation + measurement combined, hardware-agnostic |
| Bruel & Kjaer (HBK) | Trusted measurement hardware/software | Expensive, measurement-focused, limited simulation coupling | Virtual NVH prediction without physical prototypes |

---

## MVP Scope (v1.0)

### In Scope
- Modal analysis and frequency response from imported FE meshes (Nastran BDF)
- Panel contribution analysis for a coupled structural-acoustic model
- Classical TPA with imported FRF and operational data (UFF/CSV)
- Psychoacoustic metrics on uploaded audio: loudness, sharpness, roughness, tonality
- Auralization: binaural playback of TPA path contributions
- Order tracking for rotating machinery
- PDF report with contribution ranking and psychoacoustic summary

### Out of Scope (v1.1+)
- SEA and hybrid FE-SEA
- Component-based TPA and blocked force TPA
- EV-specific modules (e-motor NVH, AVAS)
- AI root cause identification
- Acoustic BEM radiation solver
- Target cascading and benchmark database

### MVP Timeline: 14-18 weeks
