# MineAssay — Mining Engineering and Ore Reserve Estimation Platform

## Executive Summary

MineAssay is a cloud-based mining engineering platform for geological modeling, resource estimation, mine planning, and geotechnical analysis. It replaces Datamine ($20K+/seat), Vulcan (Maptek) ($15K+/seat), Leapfrog ($10K+/seat), and Surpac ($12K+/seat) with browser-based 3D geological modeling, geostatistical estimation, and open-pit/underground mine design.

---

## Problem Statement

**The pain:**
- Mining software is dominated by 3-4 vendors (Datamine, Maptek, Seequent/Bentley, Hexagon) all charging $10,000-$30,000/seat/year
- Geological resource estimation using kriging and conditional simulation requires specialized geostatistical expertise that many mining companies lack
- Small mining companies and exploration firms cannot afford $100K+ in software licenses during the exploration phase when revenue is zero
- Mine planning (pit optimization, scheduling) uses separate software from geological modeling, requiring manual data transfer
- Environmental compliance (acid mine drainage, tailings stability) requires additional specialized tools

**Current workarounds:**
- Using Excel and basic 2D cross-sections instead of 3D geological models
- Inverse distance weighting (IDW) instead of proper kriging because the tools are too expensive
- Hiring consultants ($300-$500/hour) for resource estimation instead of building in-house capability
- Using QGIS with limited mining plugins for basic spatial analysis
- Junior miners relying on the vendor's hosted service at high per-hour rates

**Market size:** The mining software market is approximately $2.5 billion (2024). There are 30,000+ mining operations worldwide and 100,000+ geologists and mining engineers. The global mining industry generates $2.5T/year in revenue.

---

## Target Users

### Primary Personas

**1. Eduardo — Exploration Geologist at a Junior Mining Company**
- Drills 200 boreholes per year and needs to estimate gold/copper resources for investor presentations
- Cannot afford $50K in Leapfrog + Datamine licenses during exploration phase with no revenue
- Needs: 3D geological modeling and resource estimation at an affordable price

**2. Sandra — Mine Planner at a Mid-Sized Open-Pit Operation**
- Plans quarterly and annual mining schedules for a copper-gold open pit
- Uses Vulcan for planning but pit optimization takes days to converge
- Needs: cloud-accelerated pit optimization with multiple scenario comparison

**3. Dr. Chen — Geotechnical Engineer at a Mining Consultancy**
- Assesses slope stability for open-pit mines and tailings dams
- Uses Slide (Rocscience) for 2D slope stability but needs 3D analysis for complex geometries
- Needs: integrated geological model + geotechnical analysis for slope design

---

## Solution Overview

MineAssay is a cloud-based mining platform that:
1. Creates 3D geological models from drillhole data using implicit modeling (radial basis functions) and explicit wireframing
2. Performs geostatistical resource estimation (ordinary kriging, indicator kriging, conditional simulation) with JORC/NI 43-101/CIM-compliant reporting
3. Designs open-pit mine shells using Lerchs-Grossmann or pseudoflow pit optimization algorithms
4. Plans underground mine layouts (stoping, development, ventilation)
5. Assesses geotechnical stability (slope stability, pillar design, subsidence) integrated with the geological model

---

## Core Features

### F1: Drillhole Database Management
- Drillhole data import: collar, survey, assay, lithology, geotechnical (RQD, FF/m, UCS)
- Data validation: gap checking, overlapping intervals, survey deviation checks
- Compositing: length, bench, lithology-based
- Statistical analysis: histograms, probability plots, summary statistics, outlier detection (top-cuts)
- 3D drillhole visualization with downhole plots
- Data QA/QC: duplicate analysis, standard and blank performance, bias testing
- GIS integration: import property boundaries, topography, infrastructure

### F2: 3D Geological Modeling
- Implicit modeling: radial basis function (RBF) interpolation for lithology boundaries (like Leapfrog)
- Explicit wireframing: manual polyline digitization and snapping for precise control
- Fault modeling: fault surface definition, displacement, and domain splitting
- Fold modeling: structural trend surfaces with dip/strike data
- Grade shells: automatically generate wireframes at grade thresholds
- Vein modeling: hangwall/footwall surfaces with thickness control
- Block model generation: regular and sub-celled models with flexible dimensions
- Model validation: section views, plan views, volumes, contact checks

### F3: Resource Estimation
- Geostatistics: variogram analysis (experimental, model fitting: spherical, exponential, Gaussian, nested)
- Estimation: ordinary kriging (OK), simple kriging (SK), indicator kriging (IK), universal kriging (UK)
- Conditional simulation: sequential Gaussian simulation (SGS) for risk quantification
- Nearest neighbor and inverse distance weighting (IDW) for comparison
- Estimation parameters: search ellipsoid, minimum/maximum samples, octant search, discretization
- Estimation quality: kriging efficiency, slope of regression, conditional bias check
- Classification: measured, indicated, inferred based on confidence criteria
- JORC 2012, NI 43-101, CIM, SAMREC compliant resource reporting

### F4: Mine Planning
- Open-pit: Lerchs-Grossmann, pseudoflow, and Whittle-type pit optimization for economic shells
- Pushback (phase) design: nested pit analysis for sequencing
- Pit design: bench geometry, ramps, berms, haul road design
- Scheduling: period-by-period mining schedule with blending constraints, equipment capacity
- Cut-off grade optimization: Lane's algorithm for maximizing NPV
- Underground: stope optimization (Datamine-like MSO), development layout, level planning
- Ventilation network analysis (basic)
- Stockpile and dump design
- Equipment selection and fleet sizing

### F5: Geotechnical Analysis
- Slope stability: limit equilibrium (Bishop, Spencer, GLE) on cross-sections through geological model
- Rock mass classification: RMR, Q-system, GSI from drillhole geotechnical data
- Kinematic analysis: planar, wedge, and toppling failure modes from structural data (stereonets)
- Empirical pillar design (underground)
- Subsidence prediction for caving and longwall operations
- Tailings dam stability assessment
- Integration with geological model: material properties assigned from lithology and alteration domains

### F6: Reporting and Compliance
- JORC 2012 Table 1 report generation
- NI 43-101 Technical Report template
- Resource and reserve statement tables
- Cross-section and plan view figures with annotations
- Variogram model documentation
- Estimation parameter summary
- Classification criteria documentation
- PDF report export with professional formatting

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | TypeScript, React, Three.js/WebGPU (3D geological visualization, block models) |
| Geological Modeling | Rust (RBF implicit modeling, wireframing, block model operations) |
| Geostatistics | Rust (variogram fitting, kriging, SGS) → WASM for small models, server for large |
| Pit Optimization | Rust (Lerchs-Grossmann, pseudoflow algorithms) → server |
| Slope Stability | Rust (limit equilibrium with geological model coupling) |
| Backend | Rust (Axum) + Python (FastAPI for reporting, statistical analysis) |
| GIS | PostGIS, Mapbox GL JS |
| Compute | AWS (CPU-optimized for kriging and pit optimization) |
| Database | PostgreSQL + PostGIS (drillholes, block models, boundaries), S3 (meshes, results) |

---

## Monetization

### Exploration — $149/month
- Drillhole database (unlimited)
- 3D geological modeling (implicit + explicit)
- Resource estimation (OK, IDW)
- Basic variogram analysis
- Section and plan views
- 5 user seats

### Planning — $349/user/month
- Everything in Exploration
- Conditional simulation (SGS)
- Open-pit optimization
- Mine scheduling
- Cut-off grade optimization
- JORC/NI 43-101 reporting

### Enterprise — Custom
- Underground mine planning
- Geotechnical analysis
- On-premise deployment
- Custom reporting templates
- Integration with fleet management systems
- Training and certification

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | MineAssay Advantage |
|-----------|-----------|------------|---------------------|
| Leapfrog (Seequent/Bentley) | Best implicit modeling | $10K+/seat, no mine planning | Integrated modeling + planning |
| Vulcan (Maptek) | Comprehensive mine planning | $15K+/seat, dated UI, desktop | Cloud-native, modern UX |
| Datamine | Industry standard | $20K+/seat, complex | 10x cheaper, easier learning curve |
| Surpac (Hexagon) | Good all-round | $12K+/seat, complex | Cloud, AI-assisted, affordable |
| QGIS + SGeMS | Free, open-source | No 3D modeling, no mine planning | Full 3D, complete workflow |

---

## MVP Scope (v1.0)

### In Scope
- Drillhole import (collar, survey, assay, lithology)
- 3D drillhole visualization
- Implicit geological modeling (RBF for lithology boundaries)
- Block model generation
- Ordinary kriging estimation with basic variogram modeling
- Section and plan views with drillhole intersections
- Resource classification (measured/indicated/inferred)

### Out of Scope (v1.1+)
- Pit optimization and mine planning
- Underground design
- Conditional simulation
- Geotechnical analysis
- Scheduling
- JORC/NI 43-101 report generation

### MVP Timeline: 14-18 weeks
