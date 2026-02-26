# ThermoCraft — Thermal Management and Heat Transfer Simulation Platform

## Executive Summary

ThermoCraft is a cloud-based thermal engineering platform for electronics cooling, heat exchanger design, HVAC system analysis, and thermal management of EV batteries, power electronics, and data centers. It replaces Icepak ($20K+/seat), FloTHERM ($15K+/seat), and 6SigmaET ($10K+/seat) with browser-based conjugate heat transfer simulation and AI-optimized cooling solutions.

---

## Problem Statement

**The pain:**
- Icepak (ANSYS) costs $20,000-$40,000/seat/year; FloTHERM (Siemens) costs $15,000-$30,000/seat; these are the only dedicated electronics thermal simulation tools
- Every electronic product requires thermal management but most hardware teams can't afford dedicated thermal simulation tools or thermal engineers
- EV battery thermal management is critical for safety and performance but simulation requires coupling of electrochemical, thermal, and fluid dynamics — no single affordable tool handles this
- Data center cooling optimization is a $10B+ annual energy cost problem, but simulation tools are priced for enterprise customers only
- Thermal simulation results are only meaningful when conjugate (solid + fluid) heat transfer is solved simultaneously, which requires CFD — not just simple conduction FEA

**Current workarounds:**
- Using 1D thermal resistance networks in spreadsheets (rough estimates, miss 3D effects)
- Running general-purpose CFD (ANSYS Fluent, $30K+/seat) for thermal problems, which is massive overkill
- Hardware teams doing physical thermal testing only (thermocouple measurements) without simulation, discovering problems at prototype stage
- Using free tools like Elmer FEM for conduction only, missing convection effects

**Market size:** The thermal simulation market is approximately $2.5 billion (2024) within the broader CFD/FEA market. Every electronic device, every EV, every data center, and every building needs thermal analysis. There are 200,000+ thermal/mechanical engineers who regularly perform thermal analysis.

---

## Target Users

### Primary Personas

**1. Kevin — Hardware Thermal Engineer at a Consumer Electronics Company**
- Designs cooling solutions for smartphones, laptops, and wearables
- Uses FloTHERM but his company has only 2 licenses for a team of 12 hardware engineers
- Needs: conjugate heat transfer simulation for PCB-level and system-level thermal analysis with standard electronics component libraries

**2. Dr. Park — EV Battery Thermal Engineer**
- Designs liquid cooling plates for lithium-ion battery packs
- Needs coupled electrochemical-thermal simulation to predict temperature distribution during fast charging
- Needs: battery cell models (1D electrochemical + 3D thermal) coupled with coolant flow simulation

**3. Rachel — Data Center Facility Engineer**
- Manages cooling for a 20 MW data center with 5,000+ servers
- Uses simple CFD models but setup takes weeks for each what-if scenario
- Needs: rapid data center airflow simulation with rack-level heat loads and CRAH/CRAC unit models

---

## Solution Overview

ThermoCraft is a cloud-based thermal platform that:
1. Imports PCB layouts (ODB++, IPC-2581) and 3D CAD (STEP) to automatically build thermal models with component power dissipation maps
2. Runs conjugate heat transfer simulation: conduction in solids, natural/forced convection in fluids, and radiation — all coupled
3. Provides electronics-specific component libraries (BGA, QFN, TO-220, heat sinks, fans, heat pipes, vapor chambers, TIMs)
4. Offers application-specific workflows: electronics cooling, EV battery thermal, heat exchanger design, data center CFD, LED thermal management
5. Optimizes cooling solutions using AI: heat sink fin geometry, fan placement, cold plate channel design, and TIM selection

---

## Core Features

### F1: Conjugate Heat Transfer Solver
- 3D conduction in solids with anisotropic thermal conductivity (for PCB copper layers, graphite sheets)
- Laminar and turbulent convection (k-ε, k-ω SST) for air and liquid coolants
- Natural convection with buoyancy-driven flow
- Radiation: surface-to-surface with view factor computation, participating media
- Conjugate coupling: automatic interface detection between solid and fluid domains
- Transient simulation: power cycling, cold start, thermal runaway scenarios
- PCB model reduction: detailed copper layer modeling or equivalent anisotropic block

### F2: Electronics Cooling
- Component library: 500+ standard IC packages (BGA, QFN, QFP, SOIC, TO-220, etc.) with calibrated thermal models
- PCB thermal modeling from ODB++ or IPC-2581 layout data (copper distribution extracted automatically)
- Heat sink library: extruded, bonded fin, skived, forged, vapor chamber, with parameterized geometry
- Fan library: axial and blower fans with PQ curves from manufacturer data (Sunon, Nidec, Delta)
- TIM database: thermal interface materials with thermal conductivity and bond-line thickness
- Heat pipe and vapor chamber effective conductivity models
- Thermal via optimization for PCB heat spreading
- Junction temperature prediction: Tj = Ta + θja × P with detailed 3D accuracy

### F3: EV Battery Thermal
- Battery cell models: cylindrical (18650, 21700, 4680), pouch, prismatic
- 1D electrochemical model (P2D or equivalent circuit) coupled with 3D thermal for heat generation prediction
- Liquid cold plate simulation: mini-channel, serpentine, pin-fin configurations
- Coolant types: water-glycol, dielectric fluid (immersion cooling)
- Tab cooling vs. surface cooling comparison
- Thermal runaway propagation simulation
- Pack-level simulation with cell-to-cell variation
- Drive cycle thermal analysis (WLTP, EPA, highway, fast charge)

### F4: Data Center CFD
- Rack-level heat load specification (per server, per rack, heterogeneous loads)
- CRAH/CRAC unit models with capacity curves
- Raised floor and overhead air distribution
- Hot aisle/cold aisle containment analysis
- Blanking panel and cable obstruction effects
- PUE (Power Usage Effectiveness) estimation
- "What-if" scenarios: add/remove racks, change cooling setpoints, fail a CRAC unit
- Rapid meshing with Cartesian grid adapted to rack geometry

### F5: Heat Exchanger Design
- Types: shell-and-tube, plate, plate-fin, tube-fin, printed circuit (PCHE), microchannel
- Thermal-hydraulic design: LMTD method, ε-NTU method, detailed CFD
- Rating mode: evaluate existing design performance at off-design conditions
- Sizing mode: determine required size for specified duty
- Fouling factor and pressure drop calculations
- Material selection based on fluid compatibility and operating conditions
- TEMA standard compliance (shell-and-tube)

### F6: Optimization and Reporting
- AI heat sink optimization: given thermal resistance target, minimize volume/weight/cost
- Cold plate channel topology optimization
- Fan selection: recommend optimal fan for given system impedance
- TIM selection from database based on gap tolerance and thermal resistance requirements
- Thermal simulation report generation: max temperatures, thermal resistance paths, compliance with component limits
- Comparison mode: evaluate multiple cooling configurations side-by-side

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | TypeScript, React, Three.js/WebGPU (3D thermal visualization) |
| CFD/CHT Solver | Rust (Cartesian octree mesh + SIMPLE-based pressure-velocity coupling) → GPU-accelerated |
| Conduction Solver | Rust (FEM/FVM conjugate with fluid solver) |
| Radiation | Rust (view factor computation, surface-to-surface) |
| Battery Model | Python (PyBaMM integration for electrochemical + Rust thermal coupling) |
| Component Library | PostgreSQL (IC packages, heat sinks, fans, TIMs) |
| Backend | Rust (Axum) + Python (FastAPI for battery models and optimization) |
| Compute | AWS (GPU for CFD, CPU for conduction-only) |
| Database | PostgreSQL (components, projects), S3 (meshes, results) |

---

## Monetization

### Free Tier
- Conduction-only simulation (no fluid flow)
- Up to 10 components
- Basic heat sink library
- 3 projects

### Pro — $129/month
- Full conjugate heat transfer (conduction + convection + radiation)
- Electronics cooling workflow
- 500+ component library
- Fan and heat sink selection
- Report generation

### Advanced — $299/user/month
- Everything in Pro
- EV battery thermal workflow
- Data center CFD
- Heat exchanger design
- Transient simulation
- Optimization tools
- API access

### Enterprise — Custom
- On-premise deployment
- Custom component model development
- Integration with PCB design tools (Altium, Cadence)
- Multi-project thermal management across product lines
- Training and certification

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | ThermoCraft Advantage |
|-----------|-----------|------------|---------------------|
| FloTHERM (Siemens) | Best electronics thermal | $15-30K/seat, desktop | 10x cheaper, cloud-native |
| Icepak (ANSYS) | Accurate, comprehensive | $20-40K/seat, complex setup | Automated PCB import, faster setup |
| 6SigmaET | Good component library | $10K+/seat, electronics-only | Broader scope (battery, data center, HX) |
| COMSOL Heat Transfer | Flexible multiphysics | $8K+ base + modules, general-purpose | Thermal-native workflow, component libraries |
| SimScale | Cloud-based CFD | General-purpose, limited electronics features | Electronics-specific, component libraries |

---

## MVP Scope (v1.0)

### In Scope
- 3D steady-state conduction with anisotropic materials
- Natural convection (simplified correlations, not full CFD)
- Component library (50 common IC packages)
- Heat sink library (20 standard designs)
- STEP import for enclosure geometry
- Temperature contour visualization
- Max junction temperature prediction

### Out of Scope (v1.1+)
- Full conjugate heat transfer CFD
- PCB layout import (ODB++)
- EV battery thermal
- Data center CFD
- Heat exchanger design
- Transient simulation

### MVP Timeline: 12-16 weeks
