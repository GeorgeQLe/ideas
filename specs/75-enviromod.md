# EnviroMod — Environmental Fate and Transport Modeling Platform

## Executive Summary

EnviroMod is a cloud-based environmental engineering platform for modeling contaminant transport in groundwater, soil, and surface water. It replaces MODFLOW/MT3DMS (no GUI), GMS ($5K+/seat), and Visual MODFLOW ($4K+/seat) for groundwater modeling, plus QUAL2E and WASP for surface water quality — all in one integrated browser-based platform.

---

## Problem Statement

**The pain:**
- MODFLOW is free (USGS) but command-line with cryptic input files; GMS (Aquaveo) costs $5,000-$15,000/seat; Visual MODFLOW (Waterloo Hydrogeologic) costs $4,000-$10,000/seat for a GUI wrapper
- Environmental site assessment and remediation design require contaminant transport modeling, but most environmental consultants use simplified spreadsheet-based models because proper tools are expensive and complex
- Groundwater modeling is required for every Superfund/brownfield site, landfill design, dewatering project, and water supply well — but there are too few trained modelers relative to demand
- Integration between groundwater models and surface water quality models is typically manual, despite being physically connected systems

**Market size:** The environmental modeling market is approximately $1.5 billion (2024). Environmental remediation spending in the US alone exceeds $30B/year. There are 100,000+ environmental engineers/hydrogeologists worldwide.

---

## Core Features

### F1: Groundwater Flow Modeling
- MODFLOW-compatible finite difference solver (structured grid)
- Boundary conditions: wells (pumping/injection), rivers, drains, general head, recharge, evapotranspiration
- Layer types: confined, unconfined, convertible
- Steady-state and transient simulation
- Automatic calibration: PEST-like parameter estimation with pilot points
- Particle tracking: forward and reverse (MODPATH-equivalent)
- Water budget analysis per zone and layer
- Drawdown and capture zone delineation

### F2: Contaminant Transport
- Advection-dispersion equation (MT3DMS-equivalent)
- Sorption: linear, Freundlich, Langmuir isotherms
- Biodegradation: first-order decay, Monod kinetics, sequential decay chains
- Multi-species transport: BTEX, chlorinated solvents (PCE→TCE→DCE→VC), PFAS
- Density-dependent flow (saltwater intrusion)
- Reactive transport: mineral precipitation/dissolution, redox reactions
- Source zone modeling: DNAPL dissolution, NAPL partitioning

### F3: Remediation Design
- Pump-and-treat system design: well placement, pumping rates, capture zone verification
- In-situ treatment: permeable reactive barrier (PRB) design, biostimulation/bioaugmentation injection
- Natural attenuation assessment: trend analysis, attenuation rate estimation
- Monitored natural attenuation (MNA) evaluation
- Remediation timeframe estimation
- Cost-benefit analysis of remediation alternatives

### F4: Surface Water Quality
- River/stream 1D modeling: Streeter-Phelps, QUAL2E-equivalent
- Lake/reservoir modeling: thermal stratification, eutrophication (N, P, DO, algae)
- Stormwater runoff quality: pollutant loading, BMP effectiveness
- Mixing zone analysis for permitted discharges
- TMDLs (Total Maximum Daily Loads) compliance assessment

---

## Monetization

### Free Tier (Student)
- 2D groundwater flow (single layer, 1000 cells)
- Basic advection-dispersion transport
- Particle tracking
- 3 projects

### Pro — $99/month
- 3D multi-layer models (unlimited cells)
- Full contaminant transport (sorption, decay, multi-species)
- Automatic calibration (PEST-like)
- Remediation design tools
- Surface water quality
- Report generation

### Enterprise — Custom
- On-premise, integration with GIS/LIMS

---

## MVP Scope (v1.0)

### In Scope
- 2D/3D groundwater flow (MODFLOW-compatible solver)
- Well, river, recharge boundary conditions
- Steady-state and transient simulation
- Water table/potentiometric surface visualization on web map
- Basic advection-dispersion transport (single species, linear sorption)
- Particle tracking (forward)
- GeoTIFF/shapefile import for site boundary

### Out of Scope (v1.1+)
- Automatic calibration, reactive transport, remediation design, surface water quality

### MVP Timeline: 14-18 weeks
