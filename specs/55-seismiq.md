# SeismiQ — Seismic Analysis and Earthquake Engineering Platform

## Executive Summary

SeismiQ is a cloud-based seismic engineering platform for earthquake hazard analysis, structural seismic design, site response analysis, and seismic risk assessment. It replaces ETABS ($5K+/seat), SAP2000 ($5K+/seat), SeismoStruct ($3K+/seat), and SHAKE ($2K+/seat) with an integrated browser-based platform covering the full seismic engineering workflow.

---

## Problem Statement

**The pain:**
- ETABS/SAP2000 (CSI) cost $5,000-$15,000/seat each for structural analysis; SeismoSoft products cost $2,000-$5,000 each; DeepSoil costs $2,000+; most seismic projects need 3-5 separate tools
- Seismic analysis requires expertise spanning geotechnics (site amplification), structural engineering (building response), and seismology (hazard characterization) — few engineers span all three
- Performance-based earthquake engineering (PBEE) is becoming code-mandated but current tools don't streamline the workflow from hazard → structural model → demand → loss estimation
- Building code seismic design (IBC, Eurocode 8, NZS 1170.5) requires tedious manual compliance checking against tables and formulas
- Nonlinear dynamic analysis (time-history analysis) is required for tall buildings and critical structures but is difficult to set up and interpret

**Current workarounds:**
- Using ETABS for building analysis and separate tools for site response, hazard analysis, and loss estimation with manual data transfer at each step
- Hand calculations for seismic base shear and force distribution, missing higher-mode effects and nonlinear behavior
- Using OpenSees (free) for nonlinear analysis but with no GUI and extreme complexity in model setup
- Simplifying complex structures to avoid nonlinear analysis, leading to overly conservative (expensive) designs

**Market size:** The structural/seismic engineering software market is approximately $2.8 billion (2024). There are 150,000+ structural engineers worldwide who regularly perform seismic design. Earthquake-prone regions contain $50T+ of built infrastructure requiring seismic evaluation.

---

## Target Users

### Primary Personas

**1. Mei — Structural Engineer at a Design Firm**
- Designs mid-rise concrete and steel buildings in seismic zone 4
- Uses ETABS for linear analysis but struggles with nonlinear pushover and time-history analysis
- Needs: integrated seismic design from code-based force method through nonlinear analysis, with automated IBC/ASCE 7 compliance

**2. Dr. Alvarez — Earthquake Engineering Researcher**
- Studies seismic vulnerability of existing buildings and bridges
- Uses OpenSees for nonlinear fiber-based modeling but spends 70% of time on model setup
- Needs: visual model builder that generates OpenSees-compatible nonlinear models, with fragility curve generation

**3. Kenji — Seismic Risk Consultant**
- Performs seismic risk assessments for insurance companies and real estate portfolios
- Needs PSHA (probabilistic seismic hazard analysis), building vulnerability assessment, and loss estimation
- Needs: integrated hazard → vulnerability → loss pipeline for portfolio-level risk assessment

---

## Solution Overview

SeismiQ is a cloud-based seismic platform that:
1. Performs probabilistic seismic hazard analysis (PSHA) with global fault databases, ground motion models, and site amplification
2. Provides visual structural modeling for buildings with automatic code-based seismic loading (IBC/ASCE 7, Eurocode 8, NZS 1170.5, IS 1893)
3. Runs linear (response spectrum, equivalent lateral force) and nonlinear (pushover, dynamic time-history) structural analysis
4. Evaluates seismic performance per ASCE 41-17, FEMA P-58, and Eurocode 8-3 with demand-capacity ratios and performance level classification
5. Estimates seismic losses (repair cost, downtime, casualties) using FEMA P-58 methodology for individual buildings and portfolios

---

## Core Features

### F1: Seismic Hazard Analysis
- Probabilistic seismic hazard analysis (PSHA) with global earthquake catalog and fault models
- Ground motion prediction equations (GMPEs): NGA-West2, NGA-Sub, European, NZ models
- Uniform hazard spectrum (UHS) and conditional mean spectrum (CMS) generation
- Seismic hazard deaggregation
- Site-specific response spectrum generation per ASCE 7/Eurocode 8
- 1D site response analysis (equivalent linear: SHAKE method, nonlinear: Newmark integration)
- Ground motion record selection and scaling (conditional spectrum matching)
- USGS/GeoNet/INGV seismic hazard API integration

### F2: Structural Modeling
- Building model generator: specify grid, floors, member sizes → auto-build 3D model
- Element types: beam-column (elastic, plastic hinge, fiber), shell (elastic, layered), spring, link, damper
- Material models: confined/unconfined concrete (Mander), reinforcing steel (Menegotto-Pinto), structural steel (bilinear, Giuffre), masonry, wood
- Connection types: moment frame, braced frame (BRB, CBF, EBF), shear wall, moment-resisting, pinned
- Section library: AISC steel shapes, ACI concrete sections, NZS sections
- Foundation modeling: fixed, springs, shallow foundation, piles
- Mass assignment: tributary area, lumped, distributed
- Auto-meshing for shear walls and slabs

### F3: Seismic Analysis
- Equivalent lateral force (ELF) method per building codes
- Response spectrum analysis (RSA) with CQC/SRSS modal combination
- Linear and nonlinear static pushover analysis (conventional and adaptive)
- Nonlinear response history analysis (NRHA) with ground motion suites
- Direct integration (Newmark-β, HHT-α, explicit central difference) for time-history
- P-Δ effects (geometric nonlinearity)
- Soil-structure interaction (SSI): foundation impedance functions, kinematic interaction
- Incremental dynamic analysis (IDA) for collapse fragility assessment
- Cloud analysis: distribute 100+ time-history analyses across cloud workers

### F4: Performance Evaluation
- ASCE 41-17: demand-capacity ratios for beams, columns, walls, connections; performance levels (IO, LS, CP)
- FEMA P-58 loss estimation: component fragilities, repair costs, downtime, casualties
- Eurocode 8 Part 3: assessment of existing structures
- Inter-story drift, floor acceleration, residual drift tracking
- Plastic hinge development visualization (yield progression animation)
- Component damage state tracking per FEMA P-58
- Collapse probability estimation via IDA and fragility fitting

### F5: Code Compliance and Design
- ASCE 7-22 / IBC 2024: seismic design category, R/Ω₀/Cd factors, base shear, vertical distribution, drift limits, redundancy, irregularities
- Eurocode 8: behavior factor q, ductility classes (DCL, DCM, DCH), capacity design checks
- ACI 318-19: special moment frame, special shear wall detailing requirements and capacity design
- AISC 341-22: seismic provisions for steel structures, pre-qualified connections
- Automatic irregularity detection (torsional, soft story, mass, geometric, in-plane discontinuity)
- Automated compliance report with pass/fail checks and code references

### F6: Portfolio Risk Assessment
- Building inventory import (CSV, GIS shapefile)
- HAZUS-compatible building typology classification
- Building-specific fragility curves (code-designed or assessed)
- Portfolio-level loss exceedance curves (PML, AAL)
- Scenario earthquake loss estimation
- Risk mapping (GIS visualization of portfolio exposure and risk)
- Insurance metrics: PML (probable maximum loss), AAL (average annual loss), EP curves

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | TypeScript, React, Three.js (3D structural model), D3.js (hazard curves, fragilities) |
| FEA Solver | Rust (linear + nonlinear frame/shell FEA, implicit time integration) → server-side |
| PSHA Engine | Rust (hazard integration, GMPE evaluation) → WASM + server |
| Site Response | Rust (equivalent linear + nonlinear 1D) → WASM |
| Loss Estimation | Python (FEMA P-58 fragility convolution, Monte Carlo) |
| Backend | Rust (Axum) + Python (FastAPI for PSHA, loss estimation) |
| Compute | AWS (auto-scaling for IDA, ground motion selection) |
| Database | PostgreSQL (earthquake catalogs, building codes, fragilities), PostGIS (spatial hazard queries), S3 (ground motions, results) |

---

## Monetization

### Free Tier (Student)
- PSHA for any site (global)
- Equivalent lateral force analysis
- 5-story max building model
- Single code (ASCE 7 or Eurocode 8)
- 3 projects

### Professional — $149/month
- Unlimited model size
- Response spectrum + pushover analysis
- Nonlinear time-history (5 ground motions)
- All building codes
- Performance evaluation (ASCE 41)
- Report generation

### Advanced — $349/user/month
- Everything in Professional
- Unlimited time-history analyses (cloud-computed)
- IDA and fragility analysis
- FEMA P-58 loss estimation
- Site response analysis
- Ground motion selection/scaling
- API access

### Enterprise — Custom
- Portfolio risk assessment
- On-premise deployment
- Custom fragility development
- Integration with BIM (Revit)
- Multi-hazard assessment (seismic + wind + flood)
- Training and certification

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | SeismiQ Advantage |
|-----------|-----------|------------|-------------------|
| ETABS/SAP2000 (CSI) | Industry standard structural | $5-15K/seat each, no hazard or loss | Integrated hazard → structure → loss |
| SeismoStruct | Good nonlinear analysis | $3-5K/seat, no hazard, no code design | Full workflow, cloud compute |
| OpenSees | Free, most flexible nonlinear | No GUI, extreme learning curve | Visual builder, accessible |
| HAZUS (FEMA) | Free loss estimation | Crude building models, outdated | Modern FEMA P-58, building-specific |
| PERFORM-3D | Best nonlinear for tall buildings | $5K+, difficult setup | Easier model building, cloud IDA |

---

## MVP Scope (v1.0)

### In Scope
- Site-specific PSHA and design spectrum (ASCE 7)
- 3D frame model builder (steel and concrete moment frames)
- ELF and response spectrum analysis
- Linear code compliance checking (ASCE 7 drift, base shear, irregularities)
- 3D structural visualization with deformed shapes and mode shapes
- PDF code compliance report

### Out of Scope (v1.1+)
- Nonlinear pushover and time-history
- Performance evaluation (ASCE 41, FEMA P-58)
- Site response analysis
- IDA and fragility analysis
- Portfolio risk assessment
- Ground motion selection

### MVP Timeline: 14-18 weeks
