# WeldSim — Welding Process Simulation and Qualification Platform

## Executive Summary

WeldSim is a cloud-based platform for simulating welding processes — predicting residual stress, distortion, microstructure, and defects in fusion welding (arc, laser, electron beam), resistance welding, and friction stir welding. It replaces ESI Sysweld ($25K+/seat), Simufact Welding ($15K+/seat), and ANSYS welding add-ons with browser-based thermal-metallurgical-mechanical coupled simulation and weld procedure qualification support.

---

## Problem Statement

**The pain:**
- ESI Sysweld and Simufact Welding cost $15,000-$30,000/seat/year; ANSYS with welding requires Mechanical + Fluent + material add-ons totaling $40K+
- Welding distortion costs manufacturing industry $10B+/year in rework, rejected parts, and warranty claims
- Weld procedure qualification (WPQ) per AWS D1.1, ASME IX, or EN ISO 15614 requires expensive physical test coupons ($5,000-$20,000 per procedure)
- Residual stress prediction is critical for fatigue life, stress corrosion cracking, and fracture assessment but current simulation is too slow (days per weld pass)
- Multi-pass welding of thick structures (nuclear, offshore, pressure vessels) involves 20-200 weld passes, each requiring individual simulation — computationally prohibitive on desktop machines

**Current workarounds:**
- Using simplified empirical formulas for distortion estimation (Okerblom, Leggatt) that miss geometry effects
- Physical distortion measurement after welding and iterative redesign of clamping/sequencing
- Ignoring residual stress in design, leading to unexpected failures
- Using general-purpose FEA (ANSYS, Abaqus) with manual heat source definition and material property input, requiring months of setup per model

**Market size:** The welding industry is $20B+/year in consumables/equipment. The welding simulation market is approximately $400 million (2024). There are 200,000+ welding engineers worldwide across shipbuilding, oil & gas, nuclear, automotive, aerospace, and heavy manufacturing.

---

## Target Users

### Primary Personas

**1. Johan — Welding Engineer at a Shipyard**
- Manages welding of large ship hull blocks with 1,000+ meters of welds per block
- Distortion from welding causes fit-up problems at block assembly, costing $50K+ per rework
- Needs: distortion prediction for multi-pass welds with optimization of welding sequence and clamping

**2. Dr. Lee — Nuclear Welding Metallurgist**
- Qualifies welding procedures for nuclear pressure vessels and piping (ASME III)
- Must demonstrate residual stress levels for stress corrosion cracking assessment
- Needs: residual stress prediction with microstructure-aware material models for dissimilar metal welds

**3. Maria — Manufacturing Engineer at an Automotive Tier 1**
- Designs resistance spot welding schedules for high-strength steel body-in-white
- 10,000+ spot welds per vehicle body; nugget diameter and quality are critical for crash performance
- Needs: resistance spot welding simulation with nugget growth prediction and parameter optimization

---

## Solution Overview

WeldSim is a cloud-based welding simulation platform that:
1. Models heat source for various welding processes (GMAW, GTAW, SAW, laser, electron beam, FSW, RSW) using Goldak double-ellipsoid, conical, and custom volumetric heat sources
2. Performs coupled thermal-metallurgical-mechanical simulation: heat transfer → phase transformation (CCT/TTT diagrams) → residual stress and distortion
3. Handles multi-pass welding with automatic element activation (weld bead deposition), inter-pass temperature control, and repair weld modeling
4. Optimizes welding sequence, clamping, and pre-heating to minimize distortion and residual stress
5. Supports weld procedure development by predicting hardness, microstructure, and mechanical properties for qualification per AWS D1.1, ASME IX, EN ISO 15614

---

## Core Features

### F1: Heat Source Modeling
- Goldak double-ellipsoid for arc welding (GMAW, GTAW, SAW, SMAW)
- Conical/cylindrical for keyhole welding (laser, electron beam)
- Friction heat generation for friction stir welding (FSW)
- Joule heating + contact resistance for resistance spot welding (RSW)
- Calibration: match heat source parameters to experimental weld pool dimensions and thermal cycles
- Multi-pass sequencing: define weld path, pass sequence, inter-pass temperature, and travel speed for each pass
- Weld bead activation: quiet element, inactive element, or born-dead element techniques

### F2: Thermal-Metallurgical Simulation
- 3D transient heat transfer with temperature-dependent properties
- Phase transformation: austenite decomposition to ferrite, pearlite, bainite, martensite using CCT/TTT diagrams (Johnson-Mehl-Avrami-Kolmogorov, Koistinen-Marburger)
- Phase-dependent material properties: thermal conductivity, specific heat, density, yield strength, hardness
- Grain growth in HAZ
- Carbon diffusion in dissimilar metal welds
- Material database: carbon steels, low-alloy steels, stainless steels (austenitic, duplex, martensitic), nickel alloys, aluminum alloys, titanium alloys
- CCT/TTT diagram database: 200+ steel grades with JMatPro/Thermocalc-calculated data

### F3: Mechanical Analysis (Residual Stress and Distortion)
- Thermo-elastic-plastic FEA with phase-transformation-induced strain (TRIP)
- Volumetric change from phase transformation (austenite → martensite expansion)
- Transformation plasticity: Greenwood-Johnson model
- Creep: Norton power law for high-temperature hold and PWHT (post-weld heat treatment)
- Clamping and fixturing: spring, rigid, and contact-based boundary conditions
- Distortion prediction: angular, longitudinal, transverse shrinkage
- Residual stress output: longitudinal, transverse, through-thickness profiles
- Post-weld heat treatment (PWHT) simulation: stress relief effectiveness

### F4: Process-Specific Modules
- Resistance Spot Welding: coupled electrical-thermal-mechanical, nugget growth prediction, electrode wear modeling, multi-sheet stackup
- Friction Stir Welding: coupled thermal-mechanical with material flow (Eulerian-Lagrangian), tool force prediction
- Laser Welding: keyhole dynamics (simplified), porosity risk assessment, beam oscillation effects
- Arc Welding: shielding gas effects on arc efficiency, filler metal deposition rate
- Brazing and Soldering: capillary flow, joint strength prediction

### F5: Optimization and Sequence Planning
- Welding sequence optimization: genetic algorithm to minimize total distortion from multi-pass welds
- Clamping optimization: location and force to minimize distortion after unclamping
- Pre-heat and interpass temperature optimization
- Welding parameter optimization: current, voltage, travel speed for target penetration and bead geometry
- Inverse analysis: determine heat source parameters from measured thermal cycles
- DOE and sensitivity analysis for process parameters

### F6: Qualification and Reporting
- Weld procedure specification (WPS) generation per AWS D1.1, ASME IX, EN ISO 15614
- Predicted hardness maps (Vickers) for HAZ hardness qualification
- Microstructure maps: volume fraction of phases across weld zone
- Mechanical property prediction: yield strength, tensile strength from microstructure
- Distortion measurement comparison: overlay predicted vs. measured (CMM data import)
- Residual stress comparison: overlay with XRD, neutron diffraction, or hole-drilling measurements
- PDF report with all results, parameters, and compliance assessment

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | TypeScript, React, Three.js (3D weld visualization, residual stress contours) |
| Thermal Solver | Rust (FEM transient heat transfer, element activation) → server |
| Metallurgical | Rust (phase transformation kinetics, JMAK, KM) |
| Mechanical Solver | Rust (thermo-elastic-plastic FEM with transformation strain) → server |
| RSW Solver | Rust (coupled electrical-thermal-mechanical) |
| Optimization | Python (GA, PSO for sequence optimization) |
| Backend | Rust (Axum) + Python (FastAPI for reporting) |
| Compute | AWS (CPU for multi-pass FEA, auto-scaling for optimization) |
| Database | PostgreSQL (materials, CCT/TTT, welding parameters), S3 (meshes, results) |

---

## Monetization

### Pro — $199/month
- Single-pass welding simulation (arc, laser)
- Thermal + distortion analysis
- Material database (50 common grades)
- Basic residual stress output
- 3 projects active

### Advanced — $399/user/month
- Multi-pass welding (unlimited passes)
- Full metallurgical model (phase transformation, hardness)
- RSW, FSW modules
- Sequence optimization
- All material grades
- WPS generation
- Unlimited projects
- API access

### Enterprise — Custom
- On-premise deployment
- Custom material CCT/TTT development
- Integration with welding robot programming systems
- Digital twin for production monitoring
- Training and certification

---

## MVP Scope (v1.0)

### In Scope
- Single-pass arc welding (Goldak heat source)
- Transient thermal simulation
- Thermo-elastic-plastic distortion analysis
- 3 steel grades (mild steel, SS304, low-alloy)
- Residual stress contour visualization
- Distortion magnitude output
- Basic PDF report

### Out of Scope (v1.1+)
- Multi-pass welding
- Phase transformation metallurgy
- RSW, FSW, laser welding
- Sequence optimization
- WPS generation
- PWHT simulation

### MVP Timeline: 14-18 weeks
