# TriboSim — Tribology and Wear Simulation Platform

## Executive Summary

TriboSim is a cloud-based tribology engineering platform for simulating friction, wear, lubrication, and contact mechanics in mechanical systems. It replaces specialized modules within ANSYS ($30K+), Ricardo SABR ($20K+/seat), and custom in-house codes used by bearing, gear, and engine manufacturers. Applications include bearing analysis, gear contact, seal design, brake wear prediction, and bio-tribology (artificial joints).

---

## Problem Statement

**The pain:**
- No dedicated commercial tribology simulation tool exists as a standalone product — engineers must piece together ANSYS contact mechanics ($30K+), specialized bearing codes (SKF SimPro, Schaeffler BEARINX — available only to their customers), and custom lubrication scripts
- Bearing and gear failures cause 60% of rotating machinery downtime, costing industry $100B+/year globally, yet predictive simulation is inaccessible to most maintenance and design engineers
- Elastohydrodynamic lubrication (EHL) analysis requires coupling fluid mechanics, contact mechanics, and thermal effects — a classic multi-physics problem that general-purpose tools handle poorly
- Wear prediction (Archard's law and beyond) requires material-pair-specific coefficients that are scattered across decades of research papers, not collected in any accessible database

**Market size:** The tribology-related market (bearings, lubricants, surface treatment, seals) exceeds $150 billion globally. The simulation software for tribological analysis is approximately $500 million, growing as predictive maintenance and digital twins become mainstream.

---

## Target Users

### Primary Personas

**1. Stefan — Bearing Application Engineer**
- Selects and analyzes rolling element bearings for industrial gearboxes
- Uses SKF proprietary calculator but needs more detailed analysis (misalignment, contamination, mixed lubrication)
- Needs: detailed bearing contact analysis with life prediction, including effects of misalignment, contamination, and surface roughness

**2. Dr. Tanaka — Gear Design Engineer at an Automotive Company**
- Designs transmission gears and needs to predict tooth contact stress, scuffing risk, and micropitting
- Uses custom Fortran code inherited from a retired colleague
- Needs: gear contact simulation with EHL lubrication, temperature prediction, and fatigue life estimation

**3. Maria — Biomedical Engineer Designing Hip Implants**
- Needs to predict wear rates of UHMWPE-on-CoCr bearing surfaces over 20-year implant life
- Uses simplified pin-on-disc extrapolations that poorly predict in-vivo wear patterns
- Needs: 3D wear simulation of artificial joints with patient-specific loading from gait analysis

---

## Solution Overview

TriboSim is a cloud-based tribology platform that:
1. Performs contact mechanics analysis (Hertzian, non-Hertzian, elastic, elastic-plastic) for arbitrary surface geometries including rough surfaces
2. Solves elastohydrodynamic lubrication (EHL) problems coupling Reynolds equation with elastic deformation and energy equation for thermal effects
3. Predicts wear using physics-based and empirical models (Archard, energy-based, oxidative wear, adhesive, abrasive, fretting) with a material-pair database
4. Analyzes complete machine elements: rolling bearings, gears, cams, seals, piston rings, brakes, clutches
5. Estimates component life including surface fatigue (pitting, micropitting, spalling), scuffing risk, and fretting fatigue

---

## Core Features

### F1: Contact Mechanics
- Hertzian contact for point and line contact (analytical)
- Non-Hertzian contact: semi-analytical (SAM) and FEA-based for arbitrary geometry
- Elastic-plastic contact with material hardening
- Rough surface contact: measured surface topography input (profilometer data), statistical (Greenwood-Williamson), and deterministic models
- Sub-surface stress analysis: von Mises, maximum shear, Dang Van fatigue criterion
- Coated and layered surface analysis (varying elastic modulus with depth)

### F2: Lubrication Analysis
- Hydrodynamic lubrication: Reynolds equation for journal bearings, thrust bearings, slider bearings
- Elastohydrodynamic lubrication (EHL): coupled Reynolds + elastic deformation for rolling contacts (point and line)
- Thermal EHL: energy equation coupling with temperature-dependent viscosity
- Mixed lubrication: transition from full film to asperity contact with load sharing
- Non-Newtonian lubricant models: Eyring, Carreau, limiting shear stress
- Lubricant database: 500+ oils and greases with rheological properties (Roelands, Barus coefficients)
- Starvation effects and lubricant supply modeling
- Grease lubrication with thickener effects

### F3: Wear Prediction
- Archard wear model with material-pair-specific wear coefficients
- Energy dissipation-based wear models
- Oxidative wear model for metals at elevated temperatures
- Abrasive wear: 2-body and 3-body with particle size/hardness effects
- Fretting wear with contact mechanics coupling
- Adhesive wear with junction growth model
- Wear map generation: plot wear rate vs. load and velocity
- Cumulative wear simulation: evolve surface geometry over millions of cycles
- Material database: 200+ material pairs with wear coefficients from published literature

### F4: Machine Element Analysis
- Rolling bearings: radial ball, angular contact, cylindrical roller, tapered roller, spherical roller, needle
  - Contact stress distribution, load-life (L10), advanced life (ISO 281 aISO), EHL film thickness
  - Effects of misalignment, preload, clearance, contamination, truncation
  - Bearing thermal analysis: heat generation and temperature distribution
  - Cage dynamics and roller slip analysis
- Gears: spur, helical, bevel, worm
  - Tooth contact analysis (TCA) with tooth profile modification (tip relief, crowning)
  - EHL film thickness on gear teeth
  - Micropitting and scuffing risk per ISO/TS 6336-22 and Blok flash temperature
  - Root bending fatigue per ISO 6336
- Seals: radial lip seals, mechanical face seals, O-rings
  - Contact pressure distribution, leakage prediction, wear rate
  - Hydrodynamic sealing (reverse pumping) analysis
- Brakes and clutches: contact pressure distribution, thermal analysis, wear prediction

### F5: Surface Engineering
- Surface roughness characterization: Ra, Rq, Rsk, Rku, bearing area curve
- Surface texture optimization: honing, plateau finishing, laser texturing effects on lubrication
- Coating analysis: DLC, TiN, CrN, ceramic coatings — effect on contact stress, friction, and wear
- Surface treatment effects: shot peening, nitriding, carburizing — residual stress and hardness profiles
- Friction coefficient prediction from surface and lubricant parameters

### F6: Digital Twin and Monitoring
- Operating condition input from sensor data (vibration, temperature, load)
- Real-time remaining useful life (RUL) estimation based on simulated wear progression
- Condition monitoring integration: correlate simulated damage with vibration signatures
- Maintenance scheduling optimization
- Fleet-level analytics for multiple machines

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | TypeScript, React, WebGL (3D contact visualization, pressure maps) |
| Contact Solver | Rust (semi-analytical contact, FFT-based convolution) → WASM + server |
| EHL Solver | Rust (Reynolds + elastic deformation, multigrid solver) → server |
| Wear Simulation | Rust (surface evolution, remeshing) |
| Machine Elements | Rust (bearing, gear, seal analysis modules) → WASM for simple, server for complex |
| Backend | Rust (Axum) + Python (FastAPI for digital twin, ML) |
| Database | PostgreSQL (material pairs, lubricants, bearing catalogs), S3 (surface data, results) |
| Compute | AWS (CPU-optimized for large contact problems) |

---

## Monetization

### Free Tier (Student)
- Hertzian contact analysis
- Basic hydrodynamic bearing analysis
- Simple Archard wear calculation
- 3 projects

### Pro — $129/month
- Non-Hertzian and rough surface contact
- EHL analysis
- Rolling bearing analysis (full ISO 281)
- Gear tooth contact analysis
- Wear simulation
- Report generation

### Advanced — $299/user/month
- Everything in Pro
- Thermal EHL and mixed lubrication
- All machine element types
- Surface engineering optimization
- Digital twin capability
- API access

### Enterprise — Custom
- On-premise deployment
- Custom material/lubricant characterization
- Integration with SCADA/condition monitoring systems
- Fleet-level digital twin
- Training and certification

---

## MVP Scope (v1.0)

### In Scope
- Hertzian contact analysis (point and line)
- Hydrodynamic journal bearing analysis
- Basic EHL film thickness (Hamrock-Dowson formulas)
- Rolling bearing L10 life calculation (ISO 281)
- Archard wear model with 50 material pairs
- Lubricant database (100 common oils)

### Out of Scope (v1.1+)
- Non-Hertzian contact (rough surfaces)
- Full numerical EHL
- Gear analysis
- Seal and brake analysis
- Digital twin
- Surface engineering

### MVP Timeline: 12-16 weeks
