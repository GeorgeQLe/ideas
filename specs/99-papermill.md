# PaperMill — Pulp and Paper Process Simulation Platform

## Executive Summary

PaperMill is a cloud-based simulation platform for modeling the entire pulp and paper manufacturing chain — from wood chip digesting and chemical pulping through stock preparation, paper machine operation, and chemical recovery. It replaces WinGEMS ($15K+/seat), CADSIM Plus ($10K+/seat), and Ideas Simulation ($20K+/seat) with browser-based mass/energy balance, fiber-water physics, and paper machine modeling that lets mill engineers optimize production, reduce energy consumption, and meet environmental targets without desktop-locked proprietary tools.

---

## Problem Statement

**The pain:**
- WinGEMS (Metso) costs $15,000-$25,000/seat and is a legacy DOS-era architecture wrapped in Windows; Ideas Simulation (Andritz) costs $20,000-$40,000/seat with restrictive licensing tied to equipment vendors
- CADSIM Plus ($10,000-$20,000/seat) is Windows-only and has a steep learning curve; most mill engineers need weeks of vendor training to build useful models
- Pulp and paper processes span 50+ interconnected unit operations (digester, washers, bleach plant, stock preparation, paper machine, chemical recovery, evaporators, lime kiln) — no single affordable tool covers the full chain
- Energy accounts for 15-30% of manufacturing cost in pulp and paper mills; without simulation, energy optimization relies on tribal knowledge and trial-and-error on live equipment
- Environmental compliance (BOD, COD, AOX, particulate emissions) requires accurate mass balance modeling that most mills accomplish through fragmented spreadsheets

**Current workarounds:**
- Using vendor-specific tools (Valmet DNA simulation, Andritz IDEAS) that only model the vendor's own equipment, leaving gaps in whole-mill simulation
- Building Excel spreadsheets with manual mass/energy balances that grow to thousands of cells and become unmaintainable
- Running trial-and-error experiments on live production equipment, risking off-spec paper, broke generation, and environmental exceedances
- Hiring external consultants at $200-$400/hour for mill-wide optimization studies that could be done in-house with proper tools

**Market size:** The global pulp and paper industry generates $350B+/year in revenue with approximately 10,000 mills worldwide. There are an estimated 80,000+ pulp and paper process engineers globally. The process simulation and optimization software market for the pulp and paper sector is approximately $600M (2024).

---

## Target Users

### Primary Personas

**1. Katarina — Process Engineer at a Kraft Pulp Mill**
- Optimizes the fiber line: digester yield, brownstock washing, oxygen delignification, and ECF bleach plant
- Uses WinGEMS but her mill has only 2 licenses shared among 8 engineers; she waits days for model access
- Needs: browser-based pulping simulation with chemical kinetics and full bleach plant modeling accessible from any workstation

**2. James — Paper Machine Optimization Specialist**
- Responsible for paper machine wet end, pressing, and drying section performance at a containerboard mill
- Relies on machine vendor's proprietary tools and personal spreadsheets for drainage, retention, and drying rate calculations
- Needs: integrated paper machine model covering headbox-to-reel with drainage, pressing, and drying physics to troubleshoot quality issues and optimize speed

**3. Maria — Environmental Compliance Manager**
- Tracks BOD, COD, TSS, and AOX from mill effluent and manages air emission permits
- Maintains a patchwork of spreadsheets to estimate pollutant loads from each process stage
- Needs: whole-mill mass balance with environmental discharge tracking and scenario analysis for process changes that affect compliance

---

## Solution Overview

PaperMill is a cloud-based pulp and paper process simulation platform that:
1. Models the complete manufacturing chain from wood yard through chemical recovery, with pre-built unit operations for digesters, washers, bleach towers, refiners, screens, paper machines, evaporators, recovery boilers, and lime kilns
2. Solves rigorous mass and energy balances tracking fiber, water, dissolved solids, chemical species, and energy across the entire mill
3. Simulates paper machine physics: headbox flow, wire drainage (Kozeny-Carman), wet pressing (Carlsson consolidation model), and multicylinder drying (heat/mass transfer through porous fiber mats)
4. Models chemical pulping kinetics (Purdue/H-factor models), bleaching sequences (ClO2, O2, H2O2 kinetics), and the complete kraft chemical recovery cycle
5. Optimizes mill operations using AI: minimize steam consumption, maximize production rate, reduce chemical usage, and predict paper quality from process conditions

---

## Core Features

### F1: Fiber Line Simulation
- Kraft cooking model: Purdue kinetics with H-factor integration, kappa number prediction, yield vs. delignification curves, and liquor-to-wood ratio optimization
- Sulfite and soda pulping models with species-specific kinetics
- Brownstock washing: multi-stage diffusion washer, drum displacer, press washer, and belt washer models with Norden efficiency factors and dilution factor optimization
- Oxygen delignification: single and dual-stage O2 reactor models with temperature, pressure, NaOH charge, and retention time effects on kappa reduction
- Screening and cleaning: probability-based screening model with accept/reject split, fiber fractionation, and contaminant removal efficiency
- Mechanical pulping: TMP/CTMP refiner models with specific energy consumption, freeness prediction, and fiber length distribution
- Recycled fiber: OCC/MOW pulping, contaminant removal (flotation deinking, cleaning, screening), stickies tracking

### F2: Bleach Plant and Chemical Modeling
- ECF bleach sequence modeling: D0-EOP-D1-EP-D2 and variants with stage-by-stage kappa, brightness, and viscosity prediction
- ClO2, O2, H2O2, NaOH, and peracetic acid consumption kinetics per stage with temperature and pH dependency
- Brightness reversion prediction based on bleaching conditions and lignin residual
- AOX generation tracking per chlorination stage for environmental compliance
- Bleach plant filtrate recycle optimization to minimize fresh water consumption and effluent volume
- Chemical charge optimization: minimize total active chlorine multiple while meeting target brightness and viscosity
- Hexenuronic acid (HexA) removal modeling in acid stages
- Bleach chemical cost calculator with real-time pricing integration

### F3: Paper Machine Modeling
- Headbox: flow/turbulence model with jet-to-wire ratio, slice lip profile, and consistency effects on fiber orientation
- Wire section: drainage model based on Kozeny-Carman equation with forming board, table roll, and suction box dewatering; retention modeling (first-pass retention, filler retention, fines retention)
- Wet pressing: Carlsson/Wahlström consolidation model with nip pressure profile, felt permeability, roll geometry, and rewetting; press section water removal as a function of press impulse
- Dryer section: multicylinder drying with heat/mass transfer through porous fiber mats; drying rate curves (constant rate and falling rate periods); pocket ventilation and condensate management
- Size press and coating: pickup modeling, penetration depth, and moisture profile after application
- Calender: nip pressure, temperature, and speed effects on thickness, smoothness, and gloss
- Paper property prediction: tensile, burst, tear, porosity, and opacity from fiber properties and machine conditions
- Sheet break prediction model based on web tension, moisture profile, and defect frequency

### F4: Chemical Recovery Cycle
- Black liquor evaporation: multi-effect evaporator train with 4-7 effects, surface condenser, and concentrator; boiling point rise, heat transfer coefficients, and scaling prediction
- Recovery boiler: black liquor combustion model with smelt composition (Na2S, Na2CO3), flue gas composition, steam generation, and thermal efficiency
- Green liquor clarification and dregs handling
- Recausticizing: slaker and causticizer models with lime mud settling, causticizing efficiency, and dead load tracking
- Lime kiln: rotary kiln model with lime reburning, fuel consumption, ring formation tendency, and TRS emission prediction
- Tall oil recovery: soap skimming from black liquor, tall oil splitting
- Chemical makeup calculations: Na2SO4, CaCO3, NaOH losses and makeup requirements for full sodium-sulfur-calcium balance

### F5: Energy and Utility Systems
- Mill-wide steam balance: high-pressure, medium-pressure, and low-pressure steam headers with turbogenerator, pressure-reducing valves, and steam consumers
- Cogeneration optimization: maximize power generation while meeting process steam demands
- Water balance: fresh water intake, process water circuits, white water system, saveall operation, and effluent discharge
- Pinch analysis adapted for pulp and paper: composite curves for mill heat integration with practical constraints (fouling, distance, operability)
- Power boiler modeling: bark, hog fuel, and natural gas combustion with steam generation and flue gas treatment
- Condensate system: clean and contaminated condensate segregation with foul condensate stripping
- Compressed air, vacuum, and cooling water utility sizing

### F6: Environmental and Quality Tracking
- Effluent tracking: BOD, COD, TSS, AOX, color, and pH through primary and secondary treatment models
- Aerated stabilization basin (ASB) and activated sludge treatment models
- Air emissions: particulate, SO2, NOx, TRS, and CO2 from recovery boiler, lime kiln, and power boiler
- NCG (non-condensible gas) collection and incineration system modeling
- Grade change simulation: predict off-spec time, broke generation, and chemical consumption during grade transitions
- Paper quality dashboard: predict basis weight, moisture, caliper, brightness, strength properties from current process conditions
- Fiber quality tracking: freeness, fiber length, coarseness, and fines content from wood chip through finished product

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | TypeScript, React, SVG/Canvas (PFD), D3.js (trend plots, Sankey diagrams) |
| Fiber/Pulping Solver | Rust (mass/energy balance, chemical kinetics, ODE integration) → server |
| Paper Machine Solver | Rust (drainage PDEs, pressing consolidation, drying heat transfer) → server |
| Chemical Recovery | Rust (evaporator train, recovery boiler thermochemistry) → server |
| AI/ML | Python (PyTorch for quality prediction, SciPy for optimization) |
| Backend | Rust (Axum) + Python (FastAPI for ML inference, optimization) |
| Database | PostgreSQL (mill configurations, fiber properties, chemical databases), TimescaleDB (historian data import), S3 (simulation results, reports) |
| Hosting | AWS multi-region |

---

## Monetization

### Free Tier (Student)
- Single unit operation models (digester, washer, paper machine section)
- Manual property input (no integrated fiber line)
- 3 projects, 10 unit operations per flowsheet
- Basic mass balance only (no energy)

### Pro — $129/month
- Full fiber line simulation (digester through bleach plant)
- Paper machine wet end + press + dryer section
- Mass and energy balance
- Up to 100 unit operations per flowsheet
- Chemical cost tracking
- Report generation (PDF/Excel)

### Mill Engineer — $299/user/month
- Everything in Pro
- Full chemical recovery cycle
- Whole-mill energy and water balance
- Pinch analysis and energy optimization
- Environmental discharge tracking
- Grade change simulation
- Paper quality prediction (AI)
- Historian data import and model calibration
- API access

### Enterprise — Custom
- On-premise or private cloud deployment
- Integration with mill DCS/historian (OPC-UA, OSIsoft PI)
- Custom unit operation models for proprietary equipment
- Multi-mill portfolio management and benchmarking
- Vendor training and on-site commissioning
- Real-time digital twin for operating mills

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | PaperMill Advantage |
|-----------|-----------|------------|-------------------|
| WinGEMS (Metso) | Industry standard, large model library | $15-25K/seat, legacy DOS architecture, Windows-only, painful UI | Cloud-native, modern UX, 10x cheaper, integrated paper machine physics |
| CADSIM Plus | Good dynamic capability, Canadian mills | $10-20K/seat, steep learning curve, small user base | Easier to learn, browser-based, pre-built mill templates |
| Ideas Simulation (Andritz) | Excellent dynamic modeling | $20-40K/seat, vendor-locked to Andritz equipment, complex licensing | Vendor-neutral, whole-mill scope, accessible pricing |
| Valmet DNA Simulation | Tight DCS integration for Valmet mills | Only for Valmet customers, limited to Valmet equipment scope | Works with any vendor's equipment, full chemical recovery |
| In-house Excel models | Free, customizable | Fragile, no physics, no reuse, single-point-of-failure engineers | Rigorous physics, maintained models, collaborative |

---

## MVP Scope (v1.0)

### In Scope
- Flowsheet editor with 25 core pulp and paper unit operations (digester, washer, screen, bleach tower, mixer, splitter, heater, stock chest, fan pump, headbox, wire section, press section, dryer section)
- Kraft cooking model with H-factor and kappa prediction
- Mass balance tracking: fiber, water, dissolved solids, and 10 key chemical species (NaOH, Na2S, ClO2, Na2SO4, CaCO3, etc.)
- Energy balance for steam consumers and generators
- Paper machine drainage, pressing, and drying with basic property prediction
- Pre-built kraft mill template (digester to paper machine)
- PDF report with stream tables and mass/energy summary

### Out of Scope (v1.1+)
- Full chemical recovery cycle (evaporators, recovery boiler, recausticizing, lime kiln)
- Dynamic simulation and grade change modeling
- AI-based quality prediction and optimization
- Environmental discharge tracking and compliance reporting
- Historian data import and model calibration
- Mechanical/recycled fiber pulping models

### MVP Timeline: 16-20 weeks
