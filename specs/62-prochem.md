# ProChem — Chemical Process Simulation and Design Platform

## Executive Summary

ProChem is a cloud-based chemical process simulation platform for designing, optimizing, and analyzing chemical plants, refineries, and pharmaceutical manufacturing processes. It replaces Aspen Plus ($20K+/seat), Aspen HYSYS ($20K+/seat), and CHEMCAD ($5K+/seat) with browser-based steady-state and dynamic process simulation, thermodynamic modeling, and equipment sizing.

---

## Problem Statement

**The pain:**
- Aspen Plus and HYSYS (AspenTech, now part of Emerson) cost $20,000-$50,000/seat/year each, with separate licenses needed for steady-state and dynamic simulation
- CHEMCAD costs $5,000-$15,000/seat as a "budget" alternative but lacks advanced thermodynamics and many unit operations
- Process simulation requires deep expertise in thermodynamics (choosing the right equation of state is critical and non-obvious) — wrong choices give physically meaningless results
- Every chemical plant, refinery, pharmaceutical facility, and food processing plant needs process simulation, but tool costs restrict access
- Legacy process simulators (Aspen, PRO/II) have UIs from the 2000s era and are Windows-only, creating friction for modern engineering teams

**Current workarounds:**
- Using DWSIM (free, open-source) which has limited thermodynamics, unit operations, and no dynamic simulation
- Hand calculations with McCabe-Thiele diagrams and Kern's method, which are error-prone and slow
- Using Excel with VBA macros for simple mass/energy balances
- Pirating Aspen Plus in developing countries (extremely common in universities and small engineering firms)

**Market size:** The process simulation market is approximately $3.0 billion (2024). There are 500,000+ chemical, process, and petroleum engineers worldwide. The global chemical industry generates $4.7T/year in revenue, all requiring process simulation for design and optimization.

---

## Target Users

### Primary Personas

**1. Priya — Process Engineer at a Chemical Company**
- Designs and optimizes distillation columns, reactors, and heat exchanger networks
- Uses Aspen Plus but only has access on her office desktop (no remote work)
- Needs: browser-based process simulation with rigorous thermodynamics and equipment sizing

**2. Carlos — Petroleum Refinery Engineer**
- Optimizes crude distillation units, catalytic crackers, and hydrogen plants
- Uses Aspen HYSYS but needs separate licenses for design and operations optimization
- Needs: integrated steady-state design + dynamic simulation for control strategy validation

**3. Dr. Nakamura — Pharmaceutical Process Development Scientist**
- Develops crystallization and reaction processes for active pharmaceutical ingredients (APIs)
- Needs specialized solid-liquid equilibrium and reaction kinetics
- Needs: pharmaceutical-specific thermodynamics and batch process simulation

---

## Solution Overview

ProChem is a cloud-based process simulation platform that:
1. Provides a visual flowsheet editor with 100+ unit operation models (reactors, distillation columns, heat exchangers, pumps, compressors, separators)
2. Implements rigorous thermodynamic models: cubic EOS (SRK, PR), activity coefficient (NRTL, UNIQUAC, Wilson), electrolyte, polymer, and solid models
3. Runs steady-state and dynamic process simulation with automatic convergence of recycle loops
4. Performs equipment sizing and costing for CAPEX/OPEX estimation
5. Optimizes processes using AI: minimize energy consumption, maximize yield, debottleneck, and identify heat integration opportunities

---

## Core Features

### F1: Flowsheet Editor
- Drag-and-drop process flow diagram (PFD) with smart stream connectivity
- Unit operation library: mixers, splitters, heaters, coolers, heat exchangers, pumps, compressors, turbines, valves, flash drums, distillation columns (shortcut, rigorous), absorbers, strippers, reactors (CSTR, PFR, batch, equilibrium, Gibbs), crystallizers, dryers, filters, centrifuges, cyclones, electrolyzers
- Stream types: material (vapor, liquid, solid, multiphase), energy, work
- Automatic degree-of-freedom analysis
- Recycle loop convergence: Wegstein, Broyden, direct substitution
- Sub-flowsheet and model hierarchy for complex processes
- Process templates for common processes (distillation train, reaction-separation, utility system)

### F2: Thermodynamic Models
- Equations of state: SRK, PR, PR-BM, GERG-2008, CPA (for polar/associating)
- Activity coefficient models: NRTL, UNIQUAC, Wilson, UNIFAC (original, modified, Dortmund), COSMO-RS
- Electrolyte models: electrolyte NRTL, Pitzer, mean spherical approximation (MSA)
- Petroleum characterization: pseudo-component generation from TBP/D86 distillation, API correlations
- Solid models: SLE (solid-liquid equilibrium), solid solutions, polymorphism
- Pure component database: 10,000+ compounds with DIPPR properties
- Binary interaction parameter (BIP) database with regression from VLE/LLE data
- Flash algorithms: PT, PH, PS, TV, bubble/dew point, phase envelope

### F3: Dynamic Simulation
- Convert steady-state flowsheet to dynamic model with automatic holdup sizing
- Control system modeling: PID controllers, ratio, cascade, feedforward
- Controller tuning: IMC, Ziegler-Nichols, relay auto-tune
- Transient scenarios: startup, shutdown, load change, feed disturbance, equipment trip
- Safety analysis: pressure relief sizing, blowdown simulation
- Batch process scheduling: recipe-driven simulation with time-sequenced operations
- Integration with control system design tools (export to DCS configuration)

### F4: Equipment Sizing and Costing
- Distillation: tray hydraulics (sieve, valve, bubble cap), packing hydraulics (random, structured)
- Heat exchangers: shell-and-tube (TEMA), plate, air-cooled, double-pipe — sizing per Kern's method and Bell-Delaware
- Reactors: vessel sizing, heat transfer area, agitator power
- Pumps: hydraulic power, efficiency curves, NPSH
- Compressors: centrifugal, reciprocating, screw — head, power, efficiency
- Vessels: flash drums, separators — sizing by residence time and velocity limits
- Cost estimation: Guthrie, Lang factor, equipment cost correlations (updated annually), CAPEX/OPEX summary
- Economic analysis: NPV, IRR, payback period for process alternatives

### F5: Energy and Process Optimization
- Pinch analysis for heat integration: composite curves, grand composite, minimum utility targeting
- Heat exchanger network (HEN) synthesis
- Energy optimization: minimize utility consumption subject to process constraints
- Yield optimization: maximize product yield or selectivity
- Debottlenecking: identify and relieve capacity constraints
- Sensitivity analysis: parametric sweeps of key process variables
- Case study manager: compare multiple process configurations

### F6: Reporting and Compliance
- Process flow diagram (PFD) generation with stream tables
- Mass and energy balance tables
- Equipment datasheets per API/ASME standards
- Environmental emission estimates (CO2, SOx, NOx, VOC)
- Material safety data integration (MSDS/SDS-aware)
- Regulatory compliance: EPA, OSHA PSM, ATEX
- PDF and Excel report export
- Stream and equipment data API for integration with ERP/MES

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | TypeScript, React, SVG/Canvas (PFD), D3.js (plots) |
| Thermodynamics | Rust (EOS, activity coefficient, flash algorithms) → WASM for client-side property lookups |
| Simulation Engine | Rust (unit operation models, equation-oriented solver, sparse NR) → server |
| Dynamic Simulation | Rust (DAE integration with variable step BDF method) → server |
| Optimization | Python (SciPy, Pyomo for MINLP) + Rust function evaluations |
| Costing | Python (equipment cost correlations, economic analysis) |
| Backend | Rust (Axum) + Python (FastAPI for costing, optimization) |
| Database | PostgreSQL (compound database, BIP data, projects), S3 (simulation results) |
| Hosting | AWS multi-region |

---

## Monetization

### Free Tier (Student)
- 10 unit operations per flowsheet
- SRK/PR equations of state, ideal activity coefficient
- 500 compounds
- Basic flash calculations
- 3 projects

### Pro — $149/month
- Unlimited flowsheet size
- All thermodynamic models
- Full compound database (10,000+)
- Equipment sizing
- Sensitivity analysis
- Report generation

### Advanced — $349/user/month
- Everything in Pro
- Dynamic simulation
- Batch process simulation
- Energy optimization (pinch analysis)
- Equipment costing
- API access
- Collaborative editing

### Enterprise — Custom
- On-premise deployment
- Custom thermodynamic model integration
- Custom unit operation development
- Real-time optimization (RTO) for operating plants
- Integration with DCS/SCADA
- Training and certification

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | ProChem Advantage |
|-----------|-----------|------------|-------------------|
| Aspen Plus | Gold standard, most complete | $20-50K/seat, Windows-only, ancient UI | Cloud-native, modern UX, 10x cheaper |
| Aspen HYSYS | Best for oil & gas | $20-50K/seat, separate from Plus | Integrated steady-state + dynamic |
| CHEMCAD | Affordable alternative | Limited thermo, fewer unit ops | Better thermodynamics, cloud-based |
| DWSIM | Free, open-source | Limited thermo models, no dynamics | Rigorous thermo, dynamics, costing |
| PRO/II (AVEVA) | Good refinery simulation | $15K+/seat, dated | Modern, broader application scope |

---

## MVP Scope (v1.0)

### In Scope
- Flowsheet editor with 20 core unit operations (mixer, splitter, heater, flash, pump, compressor, HX, CSTR, PFR, shortcut column, rigorous column)
- PR and SRK equations of state
- NRTL activity coefficient model
- 2,000 compound database
- Steady-state simulation with recycle convergence
- Stream tables and mass/energy balance reports

### Out of Scope (v1.1+)
- Dynamic simulation
- Equipment sizing and costing
- Electrolyte and solid thermodynamics
- Energy optimization
- Batch process simulation
- API access

### MVP Timeline: 16-20 weeks
