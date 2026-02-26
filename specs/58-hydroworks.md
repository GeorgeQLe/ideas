# HydroWorks — Hydraulic and Hydrologic Modeling Platform

## Executive Summary

HydroWorks is a cloud-based water resources engineering platform for rainfall-runoff modeling, flood simulation, stormwater design, river hydraulics, and dam breach analysis. It replaces HEC-RAS ($0 but terrible UX), HEC-HMS, SWMM, MIKE ($15K+/seat), and InfoWorks ($10K+/seat) with a modern browser-based interface, automatic terrain processing, and AI-assisted model calibration.

---

## Problem Statement

**The pain:**
- HEC-RAS/HMS (US Army Corps) are free but have notoriously poor UIs from the 1990s, crash frequently, and require weeks of training for basic competency
- MIKE by DHI costs $15,000-$50,000/seat/year; InfoWorks ICM (Innovyze) costs $10,000-$25,000/seat — prohibitively expensive for small consultancies and municipalities
- Setting up a flood model requires manual terrain processing, cross-section extraction, Manning's n estimation, and boundary condition specification — a skilled modeler spends 60% of time on model setup vs. 40% on analysis
- Climate change is increasing flood risk globally, creating urgent demand for hydraulic modeling that far exceeds the supply of trained modelers
- Small municipalities and developing countries cannot afford $15K+ software to model their own flood risk

**Current workarounds:**
- Using HEC-RAS with extensive workarounds for its UI/stability issues, losing hours to crashes and data corruption
- Municipalities hiring consultants at $150-$300/hour for studies that could be partly self-service
- Using GIS (QGIS, ArcGIS) for approximate flood mapping without proper hydraulic simulation
- Ignoring flood risk due to cost of analysis, leading to preventable flood damage ($100B+/year globally)

**Market size:** The water resources modeling market is approximately $2.0 billion (2024). There are 150,000+ hydraulic/hydrologic engineers worldwide. Climate adaptation spending is projected at $300B/year by 2030, with flood modeling as a key component.

---

## Target Users

### Primary Personas

**1. Sarah — Flood Risk Engineer at a Consulting Firm**
- Builds HEC-RAS models for FEMA flood insurance studies and municipal flood risk assessments
- Spends 3-5 days on model setup that should take hours with proper tools
- Needs: automated terrain processing, cross-section extraction, and FEMA-compliant reporting

**2. Carlos — Municipal Stormwater Engineer**
- Manages stormwater infrastructure for a mid-sized city (50,000 population)
- Uses SWMM (free, complex) for storm sewer analysis but needs integrated surface + subsurface flooding analysis
- Needs: integrated 1D pipe network + 2D surface flood modeling for urban flooding assessment

**3. Dr. Okonkwo — Water Resources Researcher in Nigeria**
- Studies flood risk in ungauged African river basins using satellite data
- Cannot afford MIKE or InfoWorks licenses on research grant budgets
- Needs: affordable flood simulation tool that works with global terrain datasets (SRTM, FABDEM) and satellite rainfall estimates

---

## Solution Overview

HydroWorks is a cloud-based water resources platform that:
1. Automatically processes terrain data (DEM) to extract river networks, catchment boundaries, cross-sections, and floodplain geometry
2. Performs rainfall-runoff modeling (SCS-CN, Green-Ampt, kinematic wave, distributed) for design storms and continuous simulation
3. Runs 1D river hydraulics (steady and unsteady, subcritical/supercritical/mixed), 2D overland flow, and coupled 1D-2D urban flood modeling
4. Simulates stormwater pipe networks with inlet capacity analysis, surcharge, and flooding
5. Generates regulatory-compliant flood maps and reports (FEMA, EU Floods Directive, Australian guidelines)

---

## Core Features

### F1: Terrain and Data Processing
- DEM import: GeoTIFF, LiDAR (LAS/LAZ), USGS 3DEP, FABDEM, SRTM
- Automatic stream network delineation from DEM
- Catchment boundary delineation (watershed delineation)
- Cross-section extraction along river centerline (automatic and manual)
- Manning's n estimation from land cover data (NLCD, Corine)
- Bridge, culvert, and inline structure modeling
- Levee and floodwall alignment tools
- GIS base map integration (OpenStreetMap, satellite imagery)

### F2: Hydrology
- Design storms: SCS Type I/IA/II/III, Chicago, Huff, IDF curves, user-defined
- Rainfall-runoff: SCS Curve Number, Green-Ampt infiltration, Horton, initial/constant
- Unit hydrograph: SCS, Clark, Snyder, ModClark (gridded)
- Routing: Muskingum, Muskingum-Cunge, kinematic wave, modified Puls
- Continuous simulation with soil moisture accounting
- Baseflow: exponential recession, linear reservoir
- Snowmelt: temperature index, energy balance
- Climate change scenarios: factored rainfall (delta method), projected IDF curves
- Calibration: automatic parameter optimization to match observed streamflow (NSE, KGE metrics)

### F3: 1D Hydraulics
- Steady-state: standard step method (subcritical, supercritical, mixed flow regimes)
- Unsteady: implicit finite difference (Preissmann scheme) Saint-Venant equations
- Structures: bridges (energy, momentum, WSPRO, Yarnell), culverts (inlet/outlet control), weirs, gates, pumps, dams
- Dam breach analysis: parametric (Froehlich) and user-defined breach progression
- Sediment transport: Engelund-Hansen, Ackers-White, Yang, Meyer-Peter-Müller
- Water quality: temperature, dissolved oxygen, BOD, nutrients (Streeter-Phelps, QUAL2E-like)
- Ice jam and ice cover effects on hydraulics

### F4: 2D Hydrodynamics
- Shallow water equations (SWE) solver: finite volume (Godunov-type, Roe approximate Riemann)
- GPU-accelerated 2D solver for large domains (millions of cells)
- Mesh types: structured grid, unstructured triangular, flexible mesh (mixed)
- Wetting/drying algorithm for flood advance/retreat
- Rainfall-on-grid (direct rainfall) modeling
- Building representation: raised cells, building hole, building resistance
- 1D-2D coupling: river channel (1D) with floodplain (2D), pipe network (1D) with surface (2D)
- Computational efficiency: subgrid approach, local time-stepping, GPU parallelization

### F5: Urban Stormwater (1D Pipe Network)
- Pipe network definition: circular, rectangular, egg-shaped, arch, user-defined cross-sections
- Manholes, junctions, outfalls, storage nodes, ponds
- Inlet capacity: curb, grate, combination (HEC-22 methodology)
- Pump stations with pump curves and wet-well storage
- Green infrastructure / LID: bioretention, permeable pavement, green roofs, swales, rain barrels
- Water quality: pollutant buildup and washoff, treatment in BMPs
- SWMM-compatible model import

### F6: Flood Mapping and Reporting
- Automated flood depth, velocity, and hazard (depth × velocity) map generation
- Flood extent delineation at specified return periods
- FEMA FIS report generation: flood profiles, floodway data tables, FIRM-compatible maps
- EU Floods Directive Article 6 compliant flood hazard/risk maps
- Australian ARR 2019 compliant reporting
- Animation: time-lapse flood progression videos
- Damage estimation: depth-damage functions for buildings and infrastructure
- Risk assessment: expected annual damage (EAD), affected population
- PDF report generation with maps, profiles, and compliance tables

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | TypeScript, React, Mapbox GL JS / Deck.gl (GIS, flood maps), WebGL (cross-sections, profiles) |
| 1D Solver | Rust (implicit finite difference, structure hydraulics) → WASM + server |
| 2D Solver | Rust + CUDA (GPU-accelerated shallow water equations) |
| Hydrology | Rust (SCS, Green-Ampt, routing) → WASM |
| Terrain Processing | Rust (DEM analysis, watershed delineation, cross-section extraction) |
| Backend | Rust (Axum) + Python (FastAPI for calibration, damage estimation) |
| GIS | PostGIS, GDAL (raster processing) |
| Compute | AWS (p4d/p5 GPU for 2D solver, c7g CPU for 1D) |
| Database | PostgreSQL + PostGIS (spatial data), S3 (DEMs, results, flood maps) |

---

## Monetization

### Free Tier (Municipal/Educational)
- 1D steady-state hydraulics (up to 50 cross-sections)
- Basic hydrology (SCS-CN, design storms)
- Flood extent mapping
- 3 projects
- Community support

### Professional — $129/month
- 1D unsteady + 2D hydraulics
- Unlimited model size
- Dam breach analysis
- Stormwater pipe network
- FEMA report templates
- 1000 cloud compute minutes/month

### Advanced — $299/user/month
- Everything in Professional
- GPU-accelerated 2D (millions of cells)
- 1D-2D coupled urban flood modeling
- Sediment transport and water quality
- Climate change scenarios
- Automated calibration
- API access

### Enterprise — Custom
- On-premise deployment
- Real-time flood forecasting integration
- Custom regulatory report templates
- Multi-catchment watershed management
- Training and certification
- Dedicated support with SLA

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | HydroWorks Advantage |
|-----------|-----------|------------|---------------------|
| HEC-RAS | Free, US standard | Terrible UI, unstable, no cloud | Modern UX, cloud-based, automated setup |
| MIKE (DHI) | Gold standard, comprehensive | $15-50K/seat, complex | 10x cheaper, automated terrain processing |
| InfoWorks ICM | Best urban drainage | $10-25K/seat, complex | Integrated river + urban, affordable |
| SWMM (EPA) | Free, standard for stormwater | No 2D, poor GIS, complex setup | 1D-2D coupled, GIS-native |
| Flood Modeller | Good 1D-2D coupling | $5K+/seat, limited adoption | Cloud-native, better UX, GPU-accelerated |

---

## MVP Scope (v1.0)

### In Scope
- DEM import and automatic cross-section extraction
- 1D steady-state hydraulics (subcritical, bridges, culverts)
- SCS-CN hydrology with SCS Type II design storms
- Flood depth/extent mapping on web map
- Basic PDF report with flood profile and extent map

### Out of Scope (v1.1+)
- Unsteady 1D and 2D hydraulics
- Stormwater pipe network
- Dam breach analysis
- Sediment transport
- Urban 1D-2D coupling
- FEMA report generation

### MVP Timeline: 14-18 weeks
