# GridSolve — Power Systems Analysis and Grid Simulation Platform

## Executive Summary

GridSolve is a cloud-based power systems engineering platform for load flow analysis, short-circuit studies, relay protection coordination, and transient stability simulation. It replaces ETAP ($15K+/seat), PSS/E ($25K+/seat), and DIgSILENT PowerFactory ($10K+/seat) for utility engineers, renewable energy developers, and power system consultants.

---

## Problem Statement

**The pain:**
- ETAP costs $15,000-$50,000/seat/year; PSS/E (Siemens) costs $25,000-$80,000/seat; even "affordable" alternatives like PowerWorld cost $5,000-$15,000 — locking out small utilities, consulting firms, and developing countries
- Power system analysis requires specialized expertise AND specialized software; the combination creates extreme bottleneck at utility engineering departments
- Renewable energy interconnection studies are required for every solar/wind project but power system consultants are backedup 6-12 months
- Real-time grid monitoring and SCADA integration requires separate expensive software packages
- NEC/NESC compliance calculations and arc flash studies require manual cross-referencing between simulation results and code requirements

**Current workarounds:**
- Small utilities and rural cooperatives skip detailed analysis, operating on rules of thumb that lead to power quality issues and safety risks
- Renewable developers hire consultants at $300-$500/hour for interconnection studies that could be partially automated
- Using free tools like OpenDSS (EPRI) for distribution analysis, but it's script-based and has no GUI
- Engineering students learn on university ETAP licenses then can't access the tools in their first jobs

**Market size:** The power system simulation market is approximately $3.2 billion (2024), driven by grid modernization, renewable energy integration, and EV charging infrastructure. There are 100,000+ power system engineers worldwide and 3,000+ utilities globally. The renewable energy interconnection study market alone is $500M+/year.

---

## Target Users

### Primary Personas

**1. James — Protection Engineer at a Regional Utility**
- Coordinates relay settings for 200+ substations across a 5,000 MW system
- Uses ETAP for short-circuit and coordination studies but his department has only 2 licenses for 8 engineers
- Needs: full protection coordination with relay library, TCC curves, and automated setting calculations

**2. Aisha — Renewable Energy Interconnection Engineer**
- Performs power flow and fault studies for 50+ solar/wind projects per year for a consulting firm
- Uses PSS/E but spends 40% of time on model setup and data entry
- Needs: automated model building from utility one-line diagrams, batch study execution, and report generation

**3. Carlos — Microgrid Design Engineer**
- Designs microgrids for remote communities and military installations
- Needs to model diesel generators, solar PV, battery storage, and controls in an integrated environment
- Needs: microgrid-specific models with islanding detection, black start sequencing, and storage dispatch optimization

---

## Solution Overview

GridSolve is a cloud-based power systems platform that:
1. Provides a one-line diagram editor with comprehensive equipment models (generators, transformers, lines, cables, motors, capacitor banks, SVCs, inverters)
2. Runs power flow (Newton-Raphson, Gauss-Seidel, fast-decoupled), short-circuit (IEC 60909, ANSI/IEEE), and harmonic analysis
3. Performs protective device coordination with a relay library (SEL, GE, ABB, Siemens) and time-current curve (TCC) plotting
4. Simulates transient stability for contingency analysis, generator response, and renewable integration studies
5. Generates utility-grade engineering reports compliant with NERC, IEEE, and NEC standards

---

## Core Features

### F1: One-Line Diagram Editor
- Drag-and-drop single-line diagram with smart bus/branch connectivity
- Equipment library: generators (synchronous, induction, PV inverter, wind turbine), transformers (2/3 winding, auto, phase-shifting), transmission lines, cables, motors, capacitor banks, reactors, SVCs, STATCOMs, HVDC links
- Automated impedance calculation from nameplate data
- Import from CIM (Common Information Model), PSS/E RAW, and GIS shapefiles
- Geographic view: overlay one-line on map with coordinates for transmission line routing
- Substation detail views with bus section and circuit breaker topology

### F2: Power Flow Analysis
- Newton-Raphson with robust convergence for large systems (100,000+ buses)
- Unbalanced three-phase power flow for distribution systems
- Optimal power flow (OPF) with generation cost minimization and congestion management
- Voltage regulation analysis: tap changers, capacitor switching, voltage regulators
- Loss calculation and loss allocation by element
- Contingency analysis (N-1, N-2) with automatic ranking of critical contingencies
- PV curve and QV curve generation for voltage stability assessment
- Distributed generation impact analysis (hosting capacity)

### F3: Short-Circuit Analysis
- IEC 60909 (international) and ANSI/IEEE C37 (US) fault calculation methods
- Fault types: three-phase, single-line-to-ground, line-to-line, double-line-to-ground
- Sequence network visualization (positive, negative, zero)
- Equipment duty evaluation: compare fault currents to breaker/fuse ratings
- Fault current contribution tables with source tracking
- Arc flash analysis per IEEE 1584-2018 with PPE category recommendations and arc flash labels

### F4: Protection Coordination
- Relay library with models for 2,000+ relays from SEL, GE Multilin, ABB, Siemens, Basler
- Time-current curve (TCC) plotting with interactive coordination
- Overcurrent relay coordination: time-overcurrent (TOC), instantaneous, directional
- Distance relay coordination: impedance diagrams with zone settings
- Differential protection settings for transformers and buses
- Automated coordination algorithms: find optimal relay settings to achieve selectivity
- Protection report generation with relay setting sheets

### F5: Transient Stability
- Time-domain simulation with synchronous machine models (classical, detailed, subtransient)
- Governor, exciter, and PSS models per IEEE standards
- Renewable energy models: PV inverter (Type 4), wind turbine (Types 1-4), battery energy storage
- Event simulation: fault application/clearing, line switching, load shedding, generator trip
- Frequency response analysis
- Coherency identification and dynamic equivalencing for large systems
- Automatic contingency screening with CCT (critical clearing time) calculation

### F6: Reporting and Compliance
- Utility-grade report templates for interconnection studies (FERC LGIP/SGIP)
- NEC/NESC compliance checking with automatic code references
- NERC MOD/TPL standard compliance documentation
- Arc flash labels generation (GHS-compliant)
- Equipment loading summary tables
- One-click PDF report generation with one-line diagrams, tables, and analysis results
- Customizable report templates per utility/client

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | TypeScript, React, SVG (one-line diagram), Leaflet/Mapbox (geographic view) |
| Power Flow Solver | Rust (sparse Newton-Raphson with KLU) → WASM for small systems, server for large |
| Short-Circuit | Rust (sequence network analysis, IEC 60909 / ANSI C37 methods) |
| Transient Stability | Rust (DAE solver with implicit trapezoidal integration) |
| Protection | TypeScript (TCC curve engine, coordination algorithms) |
| Backend | Rust (Axum) + Python (FastAPI for reporting) |
| Database | PostgreSQL (equipment library, relay data, projects), S3 (results, reports) |
| Hosting | AWS (compute-optimized for transient stability, general for API) |

---

## Monetization

### Free Tier (Student)
- 25-bus systems
- Basic power flow and short-circuit
- 10 relay models
- Community support

### Professional — $149/month
- 5,000-bus systems
- All analysis types including transient stability
- Full relay library
- Arc flash analysis
- Report generation
- API access

### Utility — $399/user/month
- Unlimited system size
- Contingency analysis (N-1, N-2)
- OPF and hosting capacity
- Protection coordination automation
- NERC compliance reports
- CIM import/export

### Enterprise — Custom
- On-premise deployment for critical infrastructure
- SCADA/EMS integration
- Real-time state estimation
- Custom relay model development
- Training and certification
- 24/7 support with SLA

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | GridSolve Advantage |
|-----------|-----------|------------|---------------------|
| ETAP | Comprehensive, trusted by utilities | $15-50K/seat, desktop-only | 10x cheaper, cloud-native |
| PSS/E | Industry standard for transmission | $25-80K/seat, archaic UI, Fortran-based | Modern UX, web-based, accessible |
| PowerWorld | Good visualization | Limited protection features | Full protection coordination |
| DIgSILENT | Excellent European market share | $10-20K/seat, complex | Simpler UX, global relay library |
| OpenDSS | Free, EPRI-backed | Script-based, no GUI, distribution only | GUI, transmission+distribution, protection |

---

## MVP Scope (v1.0)

### In Scope
- One-line diagram editor with generators, transformers, lines, loads
- Newton-Raphson power flow (balanced, up to 1,000 buses)
- Three-phase and SLG short-circuit analysis (IEC 60909)
- Basic TCC plotting with generic overcurrent relay curves
- Equipment loading summary and voltage profile reports

### Out of Scope (v1.1+)
- Transient stability
- Detailed relay library
- Automated protection coordination
- Arc flash analysis
- OPF and contingency analysis
- CIM import

### MVP Timeline: 14-18 weeks
