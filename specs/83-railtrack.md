# RailTrack — Railway Engineering Design Platform

## Executive Summary

RailTrack is a cloud-based platform for railway track design, signaling simulation, timetable planning, and capacity analysis. It replaces OpenTrack ($20K+/seat), RailSys ($25K+/seat), Bentley OpenRail ($15K+/seat), and fragmented AREMA/EN-standard spreadsheet tools with an integrated browser-based environment covering track geometry, vehicle dynamics, signaling, and operational planning for railways, metro systems, and light rail.

---

## Problem Statement

**The pain:**
- OpenTrack costs $20,000-$40,000/seat for timetable simulation; RailSys (RMCon) costs $25,000-$50,000/seat; Bentley OpenRail costs $15,000-$30,000/seat for track design — and none of them covers the full design-to-operations workflow
- Railway track design (alignment, cant, transitions, vertical profile) and operations planning (timetable, signaling, capacity) use completely separate tools with no data exchange, requiring manual re-entry of infrastructure data
- Signaling system simulation is critical for capacity analysis but is typically done in proprietary vendor tools (Siemens, Alstom, Hitachi) that are not interoperable and not available to independent engineers
- The global push for rail expansion ($500B+ planned investment through 2030) means transit authorities and consultancies need design tools, but the $50K/seat cost structure excludes smaller firms and developing-country agencies
- Legacy track design still relies on AREMA (North America) or EN 13803 (Europe) standard calculations done in Excel spreadsheets, which are error-prone for complex alignments with multiple constraints

**Current workarounds:**
- Using AutoCAD/Civil 3D with manual cant and transition calculations in Excel — works for simple alignments but fails for constrained urban corridors with compound curves, vertical limits, and platform clearances
- OpenTrack for academic timetable research, but the interface is complex and not designed for engineering design output
- Spreadsheet-based braking distance and headway calculations using simplified kinematic models that do not account for gradient, curve resistance, or signaling system logic
- Separate tools for track geometry (MX Rail, Bentley), signaling (vendor-specific), timetable (OpenTrack/RailSys), and power supply (custom) — four tools with four data models and no integration

**Market size:** The global rail infrastructure market is $200 billion annually, with $500 billion+ in planned investments through 2030 (China, India, EU, US, Middle East). Railway design and simulation software is approximately $1.5 billion. There are 150,000+ railway engineers globally across track design, signaling, operations planning, and maintenance.

---

## Target Users

### Primary Personas

**1. Sofia — Track Design Engineer at a Railway Consultancy**
- Designs track alignments for new high-speed rail lines and urban metro extensions
- Uses Bentley OpenRail for horizontal/vertical geometry and exports to Civil 3D for detailing
- Needs: integrated track geometry design with automatic cant/transition calculation, clearance checking, and direct export to signaling and timetable simulation

**2. Tomasz — Signaling and Operations Engineer at a Transit Authority**
- Plans timetables and evaluates capacity for a metro system with 3-minute headways on shared track sections
- Uses OpenTrack for timetable simulation but must manually build the infrastructure model from design drawings
- Needs: timetable simulation that imports track geometry directly, models signaling logic (fixed block, moving block, ETCS/CBTC), and identifies capacity bottlenecks

**3. Chen — Railway Systems Engineer at a Rolling Stock Manufacturer**
- Analyzes vehicle-track interaction: ride comfort, curving performance, and braking distance for new trainsets
- Uses MATLAB/Simpack for vehicle dynamics but needs infrastructure context (curve radii, cant, gradients) that comes from the track designer
- Needs: vehicle dynamics simulation integrated with real track geometry, including speed profile optimization and energy consumption analysis

---

## Solution Overview

RailTrack is a cloud-based railway engineering platform that:
1. Designs track geometry (horizontal alignment, vertical profile, cant/superelevation, transition curves) to AREMA, EN 13803, and local standards with automatic constraint checking and optimization
2. Models signaling systems (fixed block, moving block, ETCS Level 1/2/3, CBTC, PTC) with block section layout, signal placement, and interlocking logic
3. Simulates timetables: train movements through the network with dwell times, speed restrictions, conflicts, and delay propagation — computing capacity utilization and identifying bottlenecks
4. Analyzes vehicle dynamics: tractive effort, braking distance, curve negotiation, ride comfort, and energy consumption for specific rolling stock on the designed track
5. Designs catenary/power supply systems: overhead contact system geometry, substation spacing, voltage drop, and regenerative braking energy recovery

---

## Core Features

### F1: Track Geometry Design
- Horizontal alignment: straights, circular curves, and transition curves (clothoid/Euler spiral, cubic parabola, Bloss, cosine)
- Vertical profile: gradients, vertical curves (parabolic), crest and sag with minimum radius for comfort and sight distance
- Cant (superelevation): equilibrium cant, cant deficiency, cant excess — per EN 13803, AREMA, and custom standards
- Cant transition: linear, S-shaped (sinusoidal), and cubic ramp profiles
- Speed constraints: maximum speed per curve radius, cant deficiency limit, and vertical acceleration limit
- Track gauge: standard (1435mm), broad (1520mm, 1676mm), narrow (1067mm, 1000mm), and dual-gauge
- Clearance envelope checking: structure gauge vs. kinematic envelope for tunnels, bridges, platforms, and overhead structures
- Cross-section design: ballasted track, slab track, embedded track — with subgrade, sub-ballast, and drainage layers
- Alignment optimization: minimize earthworks (cut/fill balance) subject to geometric constraints
- Data exchange: IFC Rail, LandXML, Bentley alignment format import/export

### F2: Signaling System Simulation
- Fixed-block signaling: block section definition, signal placement (home, distant, intermediate), aspect sequence (2, 3, 4-aspect)
- Moving-block signaling: train position reporting, safe following distance calculation, theoretical minimum headway
- ETCS (European Train Control System): Level 1 (balise-based), Level 2 (radio-based, no lineside signals), Level 3 (moving block)
- CBTC (Communications-Based Train Control): zone controller, trackside equipment, train-to-wayside communication model
- PTC (Positive Train Control): overlay on existing signaling, enforcement logic
- Interlocking logic: route setting, flank protection, overlap, approach locking
- Block section optimization: minimize number of signals/block sections while meeting headway target
- Signal sighting distance verification: confirm drivers can see signals from required distance
- Automatic block section layout from track geometry and headway requirements
- Degraded mode analysis: operation with signal failures, single-track working, manual block

### F3: Timetable Simulation and Capacity Analysis
- Train run simulation: solve equation of motion (tractive effort - resistance = mass x acceleration) step-by-step along the alignment
- Running time calculation: minimum run time, scheduled run time (with supplements/recovery time), and pathing time
- Davis equation resistance: rolling resistance + aerodynamic drag + gradient resistance + curve resistance
- Conflict detection: identify track section conflicts between trains, platform conflicts, junction conflicts
- Timetable editing: graphical time-distance diagram (Bildfahrplan/stringline) with drag-and-drop train paths
- Capacity analysis: UIC 406 method — compute capacity utilization percentage per track section
- Delay propagation: introduce primary delays and simulate knock-on effects across the network (stochastic simulation)
- Dwell time modeling: station dwell as a function of passenger volume, door width, and platform-train interface
- Headway calculation: signaling-constrained minimum headway at critical points (junctions, stations, crossovers)
- Scenario comparison: evaluate before/after infrastructure changes (new passing loop, additional platform, signaling upgrade)

### F4: Vehicle Dynamics and Performance
- Tractive effort curves: electric (DC, AC), diesel-electric, diesel-hydraulic, with power and adhesion limits
- Braking models: service brake, emergency brake, with blending (dynamic + friction), adhesion-dependent
- Braking distance calculation: per EN 14531 and AREMA standards, accounting for gradient, speed, reaction time, and brake build-up
- Curve negotiation: lateral acceleration, cant deficiency, bogie steering, wheel-rail contact forces
- Ride comfort assessment: ISO 2631 weighted acceleration at floor and seat level
- Cant deficiency impact on passenger comfort and track forces
- Energy consumption: net energy at wheel rim, auxiliary power, regenerative braking recovery
- Eco-driving optimization: coasting advisory for energy-minimal speed profiles
- Multi-unit consists: power car + trailer combinations, distributed power (EMU, DMU)
- Vehicle clearance: kinematic gauge calculation for cant, suspension deflection, and curve offset

### F5: Catenary and Power Supply Design
- Overhead contact system (OCS) geometry: contact wire height, stagger, tensioning, and registration
- Support structure spacing: mast placement along track, portal structures at stations
- Catenary types: simple, stitched, compound — with wire material (copper, CuAg, CuMg)
- DC traction power: substation spacing, voltage drop calculation, rail resistance, return current
- AC traction power: 25kV/50Hz, auto-transformer and booster transformer configurations
- Load flow simulation: multiple trains drawing power simultaneously, voltage at pantograph vs. position
- Regenerative braking: energy recovered to catenary, receptivity analysis, energy storage integration
- Third-rail power supply: conductor rail current capacity, gap analysis
- Harmonic analysis: power quality issues from thyristor/IGBT traction converters
- Substation sizing: transformer MVA rating, rectifier capacity, peak and average demand

### F6: Reporting and Collaboration
- Track geometry report: alignment tables, cant diagrams, speed profiles, earthwork volumes
- Signaling layout drawings: signal positions, block sections, interlocking tables
- Timetable output: graphical (time-distance diagram), tabular (station arrival/departure), and working timetable format
- Capacity report: UIC 406 utilization, bottleneck identification, infrastructure improvement recommendations
- Vehicle performance report: run times, energy consumption, braking distances, comfort assessment
- Regulatory compliance check: EN, AREMA, UIC, and country-specific standards with pass/fail annotations
- Multi-user collaboration: concurrent editing of infrastructure model with role-based access (track, signaling, operations)
- Version control: track design revisions with change highlighting and approval workflow
- Export formats: IFC Rail, DXF/DWG, PDF, CSV, RailML (standard railway data exchange)

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | TypeScript, React, Mapbox GL JS / Leaflet (geographic track alignment on map), D3.js (time-distance diagrams, speed profiles, cant diagrams) |
| Track Geometry | Rust (alignment computation, clothoid/transition math, clearance checking) → WASM for interactive design |
| Signaling Engine | Rust (block section logic, interlocking simulation, headway computation) |
| Timetable Simulator | Rust (train run calculation, conflict detection, delay propagation) → server-side for network-scale |
| Vehicle Dynamics | Rust (equation of motion solver, Davis resistance, braking models) → WASM for interactive |
| Power Supply | Rust (DC/AC load flow, voltage drop, regenerative braking) → server-side |
| AI/ML | Python (PyTorch — delay prediction, energy optimization, demand forecasting) |
| Backend | Rust (Axum) + Python (FastAPI for ML, RailML import/export, reporting) |
| Database | PostgreSQL + PostGIS (track geometry, infrastructure, timetables), S3 (design files, reports) |
| Hosting | AWS (standard compute for simulation, PostGIS for spatial queries) |

---

## Monetization

### Free Tier (Student)
- Single-line track geometry design (up to 10 km)
- Basic speed profile and cant calculation
- Single train run simulation
- 3 projects max

### Pro — $169/month
- Track geometry (up to 100 km, full transition types)
- Fixed-block signaling layout and headway analysis
- Timetable simulation (up to 50 trains/day)
- Vehicle performance (3 rolling stock types)
- Braking distance calculation
- Report generation
- 10 projects

### Advanced — $379/user/month
- Unlimited network size
- Moving-block / ETCS / CBTC signaling simulation
- Full timetable with delay propagation (stochastic)
- Capacity analysis (UIC 406)
- Catenary and power supply design
- Energy consumption optimization
- API access and multi-user collaboration
- Unlimited projects

### Enterprise — Custom
- On-premise deployment for national railways
- Custom signaling system models (country-specific)
- Integration with asset management systems (Ellipse, Maximo)
- Real-time operations dashboard integration
- Training and certification programs
- Custom regulatory standard packages

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | RailTrack Advantage |
|-----------|-----------|------------|---------------------|
| OpenTrack (ETH/OpenTrack Railway Tech) | Strong timetable and capacity simulation, academic credibility | $20-40K/seat, desktop, no track design, no signaling design, complex UI | Integrated track design + signaling + timetable in one platform |
| RailSys (RMCon) | Comprehensive timetable and infrastructure management | $25-50K/seat, desktop, steep learning curve, European-focused | Browser-based, global standards (AREMA + EN), affordable |
| Bentley OpenRail (ex-MX Rail) | Strong track geometry and civil design | $15-30K/seat, no timetable simulation, no signaling logic, civil-focused | Full railway systems: geometry + signaling + operations + power |
| AREMA/EN spreadsheet tools | Low cost, familiar | Error-prone, no integration, no simulation capability | Automated standards compliance with simulation verification |
| Vendor-specific signaling tools | Deep signaling logic (Siemens, Alstom) | Proprietary, vendor-locked, not available to independent engineers | Vendor-neutral signaling simulation for independent analysis |

---

## MVP Scope (v1.0)

### In Scope
- Horizontal and vertical track alignment design with clothoid transitions
- Cant/superelevation calculation per EN 13803 and AREMA
- Speed profile computation from geometry constraints
- Basic fixed-block signaling layout with headway calculation
- Single-train run simulation (tractive effort, resistance, braking)
- Time-distance diagram visualization
- Track geometry report (PDF) with alignment tables and speed profile

### Out of Scope (v1.1+)
- Moving-block / ETCS / CBTC signaling simulation
- Multi-train timetable simulation and conflict detection
- Delay propagation and stochastic capacity analysis
- Catenary and power supply design
- Energy consumption optimization and eco-driving
- RailML import/export and IFC Rail integration

### MVP Timeline: 14-18 weeks
