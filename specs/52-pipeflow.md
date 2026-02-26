# PipeFlow — Process Piping and Fluid Systems Engineering Platform

## Executive Summary

PipeFlow is a cloud-based platform for process piping design, hydraulic analysis, and P&ID creation. It replaces AFT Fathom ($5K+/seat), CAESAR II ($8K+/seat), and FluidFlow ($3K+/seat) with browser-based pipe network simulation, stress analysis, and automated code compliance checking for ASME, EN, and API standards.

---

## Problem Statement

**The pain:**
- AFT Fathom/Arrow costs $5,000-$12,000/seat for pipe hydraulics; CAESAR II (Hexagon) costs $8,000-$20,000/seat for pipe stress analysis; AutoPIPE costs $6,000-$15,000/seat — and most projects need all three
- P&ID creation in SmartPlant P&ID or AutoCAD P&ID costs $5,000-$15,000/seat and doesn't link to hydraulic analysis
- Pipe stress analysis to ASME B31.1/B31.3 requires specialized engineers; even minor piping modifications require re-analysis at $5,000-$20,000 per analysis package
- Process piping design is fragmented across 4-5 separate tools: P&ID, hydraulics, stress, isometrics, and material takeoff (MTO)
- Errors propagate between tools because data is manually transferred via spreadsheets

**Current workarounds:**
- Using Excel spreadsheets with Darcy-Weisbach calculations for simple systems (prone to errors, no network analysis)
- Hiring specialized piping stress analysts at $150-$300/hour for work that could be partially automated
- Using free tools like EPANET (EPA) for water distribution but it can't handle two-phase flow, non-Newtonian fluids, or compressible gas
- Junior engineers avoid stress analysis entirely, leading to piping failures ($100K-$10M per incident)

**Market size:** The piping engineering software market is approximately $2.8 billion (2024). There are 300,000+ piping engineers worldwide across oil & gas, chemical, pharmaceutical, water/wastewater, HVAC, and power generation.

---

## Target Users

### Primary Personas

**1. Ahmed — Process Engineer at a Chemical Plant**
- Designs piping systems for chemical processing (pumps, heat exchangers, reactors, vessels)
- Uses AFT Fathom for hydraulics but doesn't have CAESAR II access for stress
- Needs: integrated hydraulic + stress analysis with automated ASME B31.3 compliance

**2. Lisa — Piping Designer at an EPC Contractor**
- Creates P&IDs and piping isometrics for oil & gas projects
- Uses SmartPlant P&ID but data doesn't flow to analysis tools
- Needs: P&ID that links directly to hydraulic model, with automated MTO and isometric generation

**3. Robert — HVAC Engineer**
- Designs chilled water, heating hot water, and steam distribution systems for commercial buildings
- Uses spreadsheets for pipe sizing because dedicated tools are too expensive
- Needs: affordable pipe network analysis with ASHRAE-compliant sizing and pump selection

---

## Solution Overview

PipeFlow is a cloud-based piping platform that:
1. Provides P&ID creation with intelligent symbols linked to equipment datasheets and process conditions
2. Runs pipe network hydraulic analysis: pressure drop, flow distribution, pump sizing, for incompressible liquids, compressible gases, two-phase, and non-Newtonian fluids
3. Performs pipe stress analysis per ASME B31.1 (power), B31.3 (process), B31.4 (liquid transportation), and EN 13480
4. Generates piping isometric drawings with BOM and material takeoff
5. Optimizes pipe sizing, insulation selection, and support placement using AI

---

## Core Features

### F1: P&ID Editor
- ISA 5.1 compliant symbol library: 500+ equipment symbols (pumps, valves, vessels, heat exchangers, instruments)
- Intelligent line numbering with service, size, material class, and insulation code
- Equipment data sheets linked to P&ID symbols
- Line list auto-generation from P&ID
- Revision management with clouding and revision triangles
- Import from DXF/DWG and SmartPlant P&ID XML
- PDF/SVG/DXF export

### F2: Hydraulic Analysis
- Steady-state pipe network analysis (Hardy Cross, Newton-Raphson network solver)
- Fluid types: incompressible liquid, compressible gas (isothermal, adiabatic, polytropic), two-phase (Lockhart-Martinelli, Beggs-Brill), non-Newtonian (power law, Bingham plastic)
- Pressure drop: Darcy-Weisbach with Colebrook friction factor, fittings K-factors and equivalent lengths
- Component models: pumps (with manufacturer curves), control valves (Cv-based), orifice plates, relief valves, check valves, heat exchangers, filters
- Transient analysis: water hammer, pump trip, valve closure (method of characteristics)
- Surge analysis with surge vessel and relief valve sizing
- System curve generation for pump selection
- Cavitation and NPSH checking

### F3: Pipe Stress Analysis
- Code compliance: ASME B31.1, B31.3, B31.4, B31.8, EN 13480
- Load cases: sustained (weight + pressure), thermal expansion, occasional (wind, seismic, slug flow)
- Support types: anchors, guides, shoes, spring hangers (constant and variable), snubbers
- Spring hanger selection from manufacturer catalogs (Anvil, Piping Technology, Bergen)
- Thermal expansion loop sizing and expansion joint selection
- Flange leakage check per ASME PCC-1 / EN 1591
- Nozzle load evaluation at equipment connections
- Fatigue analysis for cyclic services
- Buried pipe analysis (soil-pipe interaction)
- Stress isometric output with stress ratios at each node

### F4: Pipe Sizing and Selection
- Optimal pipe sizing based on velocity limits, pressure drop limits, and economic pipe size (Genereaux method)
- Pipe material selection from line class specifications
- Insulation selection: heat loss calculation, personnel protection, condensation prevention
- Heat tracing design for freeze protection
- Flow measurement device sizing (orifice, venturi, flow nozzle)
- Control valve sizing per IEC 60534 / ISA 75.01

### F5: Isometric and MTO Generation
- Automated isometric drawing generation from 3D piping model
- Bill of materials (BOM) per isometric
- Material takeoff (MTO) with pipe, fittings, flanges, bolting, gaskets, and valves
- Weld count and location tracking
- Spool drawing generation for fabrication
- Integration with procurement systems via CSV/API export

### F6: AI-Assisted Design
- AI pipe routing: suggest optimal routes considering support locations, thermal expansion, and accessibility
- Automatic support placement: recommend support types and locations based on stress analysis
- Equipment layout optimization: minimize total pipe length while meeting process requirements
- Anomaly detection: flag unusual pressure drops, velocities, or stress ratios

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | TypeScript, React, SVG (P&ID), Three.js (3D piping), Canvas (isometrics) |
| Hydraulic Solver | Rust (sparse network solver, transient MOC) → WASM + server |
| Stress Solver | Rust (beam FEA with piping code checks) → server |
| P&ID Engine | TypeScript + SVG (smart symbols, connectivity graph) |
| AI/ML | Python (route optimization, support placement) |
| Backend | Rust (Axum) + Python (FastAPI) |
| Database | PostgreSQL (pipe specs, equipment data, projects), S3 (drawings, reports) |
| Hosting | AWS multi-region |

---

## Monetization

### Free Tier
- Simple pipe sizing calculator
- P&ID with 25 symbols max
- Single-line hydraulic analysis (no networks)
- 3 projects

### Pro — $79/month
- Full P&ID editor
- Pipe network hydraulics (up to 500 nodes)
- Basic stress analysis (sustained + thermal, ASME B31.3)
- Isometric generation
- Report generation

### Engineering — $199/user/month
- Everything in Pro
- Unlimited network size
- Full stress analysis (all ASME/EN codes)
- Transient/water hammer analysis
- Two-phase and compressible flow
- Spring hanger selection
- MTO and procurement export
- API access

### Enterprise — Custom
- On-premise deployment
- Integration with PDMS/S3D/E3D piping models
- Custom line class specifications
- Multi-project portfolio management
- Training and certification

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | PipeFlow Advantage |
|-----------|-----------|------------|-------------------|
| AFT Fathom | Excellent hydraulics | $5-12K/seat, no stress, no P&ID | Integrated P&ID + hydraulics + stress |
| CAESAR II | Industry standard stress | $8-20K/seat, stress only | All-in-one platform, 5x cheaper |
| FluidFlow | Good UI, affordable | No stress analysis, limited codes | Full stress + hydraulics + P&ID |
| EPANET | Free (EPA) | Water only, no process fluids | All fluid types, process engineering focus |
| SmartPlant P&ID | Best P&ID tool | $5-15K/seat, no analysis link | P&ID linked to analysis |

---

## MVP Scope (v1.0)

### In Scope
- P&ID editor with 100 common symbols
- Steady-state liquid pipe network analysis (Darcy-Weisbach)
- Pump curve integration and system curve
- Basic sustained stress analysis (ASME B31.3)
- Pipe sizing calculator
- PDF report generation

### Out of Scope (v1.1+)
- Two-phase and compressible gas flow
- Transient (water hammer) analysis
- Full thermal stress analysis
- Isometric and MTO generation
- Spring hanger selection
- AI-assisted design

### MVP Timeline: 14-18 weeks
