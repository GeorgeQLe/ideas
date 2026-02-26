# GeoDesk — Geotechnical Engineering Analysis Platform

## Executive Summary

GeoDesk is a cloud-based geotechnical engineering platform for soil analysis, foundation design, slope stability, retaining wall design, and seepage analysis. It replaces PLAXIS ($15K+/seat), GeoStudio ($5K+/seat), and RSPile ($3K+/seat) with browser-based access, AI-assisted soil parameter estimation, and automated code compliance checking.

---

## Problem Statement

**The pain:**
- PLAXIS (Bentley) costs $15,000-$40,000/seat/year for 2D/3D geotechnical FEA; Rocscience suite costs $5,000-$20,000; even simpler tools like Settle3 or Slide cost $2,000-$5,000 each
- Geotechnical engineers need 5-8 separate software packages for different analysis types (slope stability, pile design, settlement, seepage, retaining walls), each with separate licenses
- Soil constitutive model selection and parameter calibration require expert judgment; junior engineers struggle to choose between Mohr-Coulomb, Hardening Soil, and advanced models
- Geotechnical reports are generated manually, consuming 30-40% of project time on documentation rather than analysis
- Integration between borehole data, lab test results, and analysis models requires manual data entry at each stage

**Current workarounds:**
- Using spreadsheets for simple bearing capacity and settlement calculations (error-prone, not auditable)
- Running 10+ separate licensed programs for a single project (slope stability in Slide, settlement in Settle3, piles in RSPile, seepage in SEEP/W)
- Pirating PLAXIS in developing countries where infrastructure projects are critical but budgets are limited
- Open-source alternatives (OpenSees) are research-focused and impractical for consulting workflows

**Market size:** The geotechnical software market is approximately $1.5 billion (2024). There are 200,000+ geotechnical engineers worldwide. Infrastructure spending globally exceeds $4 trillion/year, with every project requiring geotechnical analysis.

---

## Target Users

### Primary Personas

**1. Priya — Geotechnical Engineer at a Consulting Firm**
- Works on 15-20 projects simultaneously (buildings, roads, dams, bridges)
- Uses PLAXIS for complex problems but relies on spreadsheets for routine analyses
- Needs: integrated platform covering 90% of routine geotechnical calculations with automated reporting

**2. Kofi — Structural/Geotech Engineer in a Developing Country**
- Designs foundations for buildings and bridges in sub-Saharan Africa
- Cannot afford PLAXIS ($40K) or even GeoStudio ($5K) on local engineering salaries
- Needs: affordable browser-based tool with bearing capacity, settlement, slope stability, and pile design

**3. Dr. Martinez — Geotechnical Researcher**
- Studies soil-structure interaction under earthquake loading
- Uses OpenSees and custom Python scripts for dynamic analysis
- Needs: advanced constitutive models with dynamic analysis capabilities and Python scripting interface

---

## Solution Overview

GeoDesk is a cloud-based geotechnical platform that:
1. Manages borehole logs, lab test data (triaxial, consolidation, grain size), and in-situ test results (SPT, CPT, PMT) in a unified project database
2. Runs 2D/3D geotechnical FEA with advanced soil constitutive models (Mohr-Coulomb, Hardening Soil, Modified Cam-Clay, hypoplastic)
3. Performs slope stability (Bishop, Spencer, Morgenstern-Price, FEM strength reduction), foundation design (bearing capacity, settlement, pile analysis), seepage, and retaining wall analysis
4. Provides AI-assisted soil parameter estimation from SPT N-values, CPT data, or soil classification
5. Generates code-compliant geotechnical reports (Eurocode 7, AASHTO, AS 4678, IS 6403) with automatic factor-of-safety checks

---

## Core Features

### F1: Geotechnical Database
- Borehole log entry and visualization (AGS, DIGGS import formats)
- Lab test data management: triaxial (UU, CU, CD), consolidation (oedometer), direct shear, grain size, Atterberg limits, compaction
- In-situ test data: SPT (with energy corrections), CPTu (with classification charts), pressuremeter, vane shear
- AI parameter estimation: input SPT N-values and soil type → get recommended c', φ', E', K0, OCR
- Cross-section generator from borehole data with interpolated soil layers
- GIS integration: plot borehole locations on maps

### F2: Slope Stability Analysis
- Limit equilibrium methods: Bishop simplified, Spencer, Morgenstern-Price, Janbu (simplified and rigorous), GLE
- Finite element strength reduction method (phi-c reduction)
- Circular and non-circular slip surfaces with auto-search (grid, entry-exit, Cuckoo search optimization)
- Probabilistic analysis: Monte Carlo and FORM with spatial variability
- Seismic analysis: pseudo-static, Newmark sliding block
- Reinforcement: geosynthetics, soil nails, anchors, piles
- Rapid drawdown and staged construction analysis
- 3D slope stability for complex geometries

### F3: Foundation Design
- Shallow foundations: bearing capacity (Terzaghi, Meyerhof, Hansen, Vesic), settlement (elastic, consolidation, immediate + long-term), mat/raft analysis
- Deep foundations: single pile capacity (alpha, beta, lambda methods), pile group analysis, negative skin friction (dragload), lateral pile analysis (p-y curves, FEM)
- Rock socket design
- Pile driving analysis: wave equation (WEAP-equivalent)
- Pile load test interpretation (Davisson, De Beer, Chin)
- Ground improvement: stone columns, vibro-compaction, preloading with PVDs

### F4: 2D/3D Geotechnical FEA
- Plane strain and axisymmetric 2D FEA
- Full 3D FEA for complex geometries
- Constitutive models: Mohr-Coulomb, Hardening Soil, Hardening Soil Small Strain, Modified Cam-Clay, hypoplastic clay/sand, UBCSAND (liquefaction)
- Construction staging: excavation, fill placement, dewatering, consolidation
- Soil-structure interaction: retaining walls, tunnels, piles, rafts with interface elements
- Groundwater: steady-state and transient seepage, coupled consolidation (Biot theory)
- Dynamic analysis: earthquake input, site response, liquefaction potential

### F5: Retaining Structures
- Gravity walls, cantilever walls, sheet pile walls, diaphragm walls, MSE walls, soldier pile and lagging
- Earth pressure calculation: Rankine, Coulomb, Caquot-Kerisel, log-spiral
- Tieback/anchor design with pre-stress
- Braced excavation analysis with staged construction
- Global stability (sliding, overturning, bearing capacity, overall slope stability)
- Structural design of wall sections (moment, shear, reinforcement)

### F6: Reporting and Code Compliance
- Auto-generated geotechnical reports with borehole logs, analysis results, and design summaries
- Code compliance checking: Eurocode 7 (DA1, DA2, DA3), AASHTO LRFD, Australian Standards, Indian Standards, Chinese GB
- Automatic partial factor application for ULS and SLS checks
- Report customization: company logo, template, sign-off blocks
- PDF export with embedded calculation sheets
- Cross-referencing between soil data, analysis models, and design outputs

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | TypeScript, React, Canvas/WebGL (soil profile and FEA visualization) |
| FEA Solver | Rust (2D/3D geotechnical FEA with direct sparse solver) → server-side |
| Slope Stability | Rust (limit equilibrium, strength reduction) → WASM for simple, server for complex |
| Foundation | Rust (bearing capacity, pile analysis, p-y curves) → WASM |
| Seepage | Rust (2D/3D steady-state and transient seepage solver) |
| AI/ML | Python (soil parameter estimation, site characterization) |
| Backend | Rust (Axum) + Python (FastAPI for AI and reporting) |
| Database | PostgreSQL (soil data, projects), PostGIS (spatial queries), S3 (results) |
| Hosting | AWS multi-region |

---

## Monetization

### Free Tier (Student)
- Basic bearing capacity and settlement calculations
- Slope stability (Bishop, up to 5 soil layers)
- Single pile capacity
- 3 projects max

### Professional — $99/month
- All analysis types (slope, foundation, seepage, retaining walls)
- AI soil parameter estimation
- 2D FEA (Mohr-Coulomb, Hardening Soil)
- Code-compliant report generation
- Unlimited projects

### Advanced — $249/user/month
- Everything in Professional
- 3D FEA
- Advanced constitutive models
- Dynamic/seismic analysis
- Probabilistic analysis
- API access
- Collaborative editing

### Enterprise — Custom
- On-premise deployment
- Custom constitutive model integration
- Integration with GIS/BIM platforms
- Multi-project dashboard for large consultancies
- Training and certification

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | GeoDesk Advantage |
|-----------|-----------|------------|-------------------|
| PLAXIS | Industry standard FEA | $15-40K/seat, desktop, steep learning curve | 10x cheaper, web-based, AI-assisted |
| GeoStudio (SEEP/W, SLOPE/W) | Good suite coverage | $5-20K for full suite, separate products | Integrated platform, one price |
| Rocscience | Excellent slope stability | $5K+/module, multiple separate tools | All-in-one platform |
| OpenSees | Free, research-grade | No GUI, extremely steep learning curve | Professional UI, consulting workflow |
| DeepExcavation | Good for excavations | Narrow focus, $3K+ | Broader scope, AI features |

---

## MVP Scope (v1.0)

### In Scope
- Borehole log entry with SPT data
- AI soil parameter estimation from SPT
- Bearing capacity (Meyerhof) and elastic settlement
- Slope stability (Bishop simplified) with circular search
- Single pile capacity (alpha and beta methods)
- Basic report generation (PDF)

### Out of Scope (v1.1+)
- 2D/3D FEA
- Advanced constitutive models
- Seepage analysis
- Dynamic/seismic analysis
- Retaining wall design
- Code-specific compliance checking

### MVP Timeline: 12-16 weeks
