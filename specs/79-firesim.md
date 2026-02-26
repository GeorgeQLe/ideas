# FireSim — Fire Dynamics, Smoke Spread, and Evacuation Simulation Platform

## Executive Summary

FireSim is a cloud-based platform for simulating fire growth, smoke spread, sprinkler activation, structural fire response, and occupant evacuation in buildings and infrastructure. It replaces FDS/Smokeview (NIST — powerful but command-line, steep learning curve), PyroSim ($4K+/seat GUI wrapper), Pathfinder ($4K+/seat egress), and CFAST (zone model) with an integrated browser-based environment that combines CFD fire modeling, egress simulation, and fire suppression design in one platform.

---

## Problem Statement

**The pain:**
- FDS (Fire Dynamics Simulator) by NIST is free and the global standard for fire CFD, but requires text-file input, Linux command-line expertise, and weeks of training — most fire protection engineers cannot use it directly
- PyroSim (Thunderhead Engineering) provides a GUI for FDS at $4,000-$6,000/seat but is a desktop application that still requires deep FDS knowledge for reliable results; Pathfinder (egress) is another $4,000/seat — totaling $8K-$12K for the combined workflow
- Fire protection engineering firms run 50-200 fire simulations per year for performance-based design projects, but each FDS model takes 2-5 days of setup and 1-7 days of compute on desktop machines — project timelines suffer
- Performance-based fire design (ASCE/SFPE, BS 7974, ISO 23932) is increasingly required for large/complex buildings but the simulation workflow is fragmented: fire model (FDS) → egress model (Pathfinder/STEPS) → structural fire model (Abaqus/SAFIR) with manual data transfer
- Insurance, code authorities, and AHJs (authorities having jurisdiction) struggle to review fire simulation results because models are opaque, difficult to reproduce, and not standardized

**Current workarounds:**
- Using CFAST (NIST zone model) for quick estimates, but zone models miss smoke stratification, complex geometries, and natural ventilation effects
- Hiring FDS consultants at $200-$400/hour for projects that require CFD fire modeling
- Running FDS on desktop machines overnight or over weekends due to long compute times
- Using separate tools for fire (FDS), egress (Pathfinder/STEPS/buildingEXODUS), and structural fire (SAFIR/Vulcan) with manual coupling
- Simplified hand calculations (SFPE Handbook correlations) for fire plume, smoke layer, and detector/sprinkler activation — valid only for simple geometries

**Market size:** The fire protection engineering market is $80B+/year globally (systems, design, inspection). Fire simulation software is approximately $300 million (2024), growing at 12% CAGR driven by performance-based design adoption, tall timber construction, and post-Grenfell regulatory tightening. There are 80,000+ fire protection engineers worldwide, with 5,000+ firms practicing performance-based design.

---

## Target Users

### Primary Personas

**1. Sarah — Fire Protection Engineer at a Consulting Firm**
- Performs performance-based fire design for complex buildings (atria, high-rise, airports, stadiums) per NFPA 101/IBC
- Uses PyroSim/FDS for smoke modeling and Pathfinder for egress, spending 3-5 days per model on setup alone
- Needs: integrated fire + egress simulation with faster setup, cloud compute for overnight runs, and professional reporting for AHJ submittals

**2. Captain Torres — Fire Investigator at a State Fire Marshal's Office**
- Reconstructs fire origin and spread for investigation of fatal fires and large-loss incidents
- Needs to run FDS models to test hypotheses about ignition location, fuel package, and ventilation, but lacks FDS expertise
- Needs: guided fire reconstruction workflow that generates FDS-quality results without requiring command-line expertise

**3. Dr. Patel — Structural Fire Engineer at a Specialist Consultancy**
- Evaluates structural fire resistance for steel and concrete buildings beyond prescriptive rating (performance-based structural fire engineering per Eurocode)
- Needs thermal boundary conditions from fire models (gas temperature, heat flux to structural members) to feed into structural analysis
- Needs: coupled fire-thermal-structural workflow that transfers fire model results to structural member temperature calculation automatically

---

## Solution Overview

FireSim is a cloud-based fire simulation platform that:
1. Simulates fire growth and smoke spread using CFD (Large Eddy Simulation) with combustion, radiation, and soot transport — FDS-compatible physics with cloud-scalable compute
2. Models sprinkler activation, smoke detector response, and fire suppression system effectiveness (water spray, gaseous agents, smoke control fans)
3. Performs occupant evacuation simulation with agent-based modeling — pre-movement time distributions, wayfinding, visibility-impaired movement, and tenability assessment (ASET vs. RSET)
4. Calculates structural member temperatures from fire exposure (lumped-mass and 2D finite element heat transfer) for steel, concrete, and timber elements
5. Provides guided workflows for common fire engineering scenarios (atrium smoke control, means of egress, structural fire resistance) with automatic compliance checking against NFPA, IBC, BS, and Eurocode

---

## Core Features

### F1: CFD Fire Simulation
- Large Eddy Simulation (LES) solver for buoyancy-driven fire flows — FDS-equivalent physics
- Combustion: mixture fraction model with fast chemistry, finite-rate kinetics for suppression gas interaction
- Radiation: finite volume method (FVM) radiation solver, gray gas model with soot-based absorption coefficient
- Soot transport: smoke yield-based soot generation, transport as scalar, optical density calculation for visibility
- Heat release rate (HRR): prescribed fire curves (t-squared), measured HRR data import, or pyrolysis model for material-driven fire growth
- Material pyrolysis: solid-phase thermal decomposition for upholstered furniture, wall linings, cable trays — predicting fire spread on surfaces
- Multi-mesh: domain decomposition for parallel computation across multiple MPI processes
- Terrain and wind: external fire scenarios with slope, vegetation (WUI fires), and ambient wind profiles
- HVAC system coupling: duct network model with fan curves, damper states, smoke control system operation

### F2: Fire Detection and Suppression
- Smoke detector activation: obscuration-based model (photoelectric, ionization) with detector lag time and ceiling jet correlations
- Heat detector activation: RTI (Response Time Index) model for fixed-temperature and rate-of-rise detectors
- Sprinkler activation: RTI-based thermal element model, spray pattern (water droplet trajectory and evaporation)
- Water spray suppression: droplet transport, evaporative cooling, fuel surface wetting — reduction of HRR from sprinkler spray
- Gaseous suppression: clean agent (FM-200, Novec 1230, IG-541) concentration buildup, inerting, and hold time
- Smoke control systems: mechanical exhaust, pressurization, natural ventilation (atrium vents, operable windows)
- Make-up air: modeling replacement air flow for smoke exhaust systems
- Fire alarm system timeline: detection → notification → HVAC shutdown → smoke control activation sequence modeling

### F3: Egress and Evacuation Simulation
- Agent-based model: individual occupants with position, speed, body size, and behavioral attributes
- Pre-movement time: log-normal distributions per occupancy type and alarm type (SFPE guidelines, PD 7974-6)
- Movement speed: density-dependent speed (Fruin, SFPE correlations), reduced speed on stairs, impaired mobility (wheelchair, elderly, children)
- Visibility-impaired movement: couple smoke optical density from CFD to local agent speed reduction and route choice
- Wayfinding: familiar vs. unfamiliar occupants, signage visibility, exit choice models (nearest, shortest queue, habit)
- Toxicity assessment: FED (Fractional Effective Dose) for CO, HCN, CO2, O2 depletion, and heat — accumulated along each agent's path
- ASET vs. RSET analysis: Available Safe Egress Time from fire model compared to Required Safe Egress Time from egress model, with safety margins per SFPE/BS 7974
- Elevator evacuation: modeling elevator use for evacuation (phased evacuation, occupant evacuation elevators per IBC)
- Crush and congestion: density limits, counter-flow effects, merging flows at stairwell entries

### F4: Structural Fire Response
- Section factor (Hp/A) calculation for steel members with passive fire protection (intumescent, board, spray)
- Steel temperature calculation: lumped-mass model (EN 1993-1-2) and 2D FEM heat transfer for complex cross-sections
- Concrete temperature profiles: 2D FEM with moisture evaporation plateau (EN 1992-1-2)
- Timber charring: effective cross-section method and advanced charring calculation (EN 1995-1-2)
- Fire curves: standard fire (ISO 834, ASTM E119), hydrocarbon (EN 1363-2), external fire, and parametric fire curves (EN 1991-1-2 Annex A)
- Natural fire exposure: map CFD-calculated gas temperatures and heat fluxes to structural members as thermal boundary conditions
- Structural capacity: temperature-reduced strength and stiffness for steel, concrete, and timber per Eurocode or AISC/ACI
- Connection behavior: temperature-dependent moment-rotation for steel connections (simple, semi-rigid)
- Travelling fire methodology: non-uniform fire exposure moving across floor plate for large open-plan spaces

### F5: Scenario and Code Compliance
- Design fire library: pre-configured fire scenarios for common occupancies (office, retail, assembly, residential, storage, parking) per SFPE Handbook and EN 1991-1-2
- Fuel load database: fuel load density (MJ/m2) distributions by occupancy type for parametric and probabilistic fire analysis
- NFPA 92 smoke control analysis: atrium smoke filling calculation, exhaust rate sizing, make-up air requirements
- NFPA 101/IBC egress compliance: exit capacity, travel distance, common path of travel, dead-end verification
- BS 7974 / PD 7974 parts 1-7: fire engineering framework with sub-system analysis guidance
- Eurocode fire design: EN 1991-1-2 actions on structures, EN 1993/1992/1995-1-2 structural fire resistance
- Multi-scenario analysis: run multiple fire locations, sizes, and ventilation conditions to identify worst-case
- Sensitivity analysis: vary fire growth rate, HRR, pre-movement time, and evaluate impact on ASET/RSET margin
- Probabilistic fire risk: Monte Carlo simulation with fire frequency, growth rate, suppression reliability, and egress time distributions

### F6: Visualization and Reporting
- Real-time 3D smoke visualization: volumetric rendering of soot density, temperature, and visibility
- Isosurface and slice plots: temperature, velocity, visibility, FED at user-defined planes and times
- Animation export: MP4/GIF of smoke spread and evacuation for AHJ presentations and client communication
- Egress timeline: cumulative evacuation curve, last-occupant-out time, congestion heatmaps
- Tenability timeline: visibility, temperature, toxicity (FED) at key locations over time
- BIM import: IFC file import for building geometry with automatic room detection, door/stair identification
- CAD import: DXF/DWG floor plan import with extrusion to 3D, STEP for complex geometries
- Report generation: fire engineering brief, ASET/RSET summary, structural fire resistance assessment, code compliance matrix — PDF export formatted for AHJ submittal
- Model sharing: shareable link for AHJ/peer review with read-only interactive 3D viewer

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | TypeScript, React, Three.js/Babylon.js (volumetric smoke rendering, egress animation, structural temperature contours) |
| CFD Fire Solver | Rust + CUDA (LES solver, combustion, radiation, soot transport) → GPU-accelerated |
| Egress Solver | Rust (agent-based model, pathfinding, density-speed relations, FED accumulation) |
| Structural Thermal | Rust (2D FEM heat transfer, lumped-mass models, charring) |
| Detection/Suppression | Rust (RTI models, spray dynamics, gas concentration) |
| Scenario Engine | Python (design fire library, probabilistic Monte Carlo, code compliance checking) |
| Backend | Rust (Axum) + Python (FastAPI for BIM/IFC parsing, reporting, fuel load database) |
| Compute | AWS (GPU instances for LES CFD, CPU auto-scaling for egress and parametric studies, spot instances for Monte Carlo) |
| Database | PostgreSQL (material properties, fuel loads, design fires, occupancy parameters), S3 (meshes, results, animations) |

---

## Monetization

### Free Tier (Student)
- Zone model fire simulation (CFAST-equivalent, two-zone)
- Single-room fire scenario with smoke layer calculation
- Basic egress (single floor, 100 occupants max)
- Standard fire curves (ISO 834, ASTM E119)
- 3 projects

### Pro — $249/month
- CFD fire simulation (LES, single mesh up to 500K cells)
- Smoke detection and sprinkler activation modeling
- Full egress simulation (multi-floor, unlimited occupants)
- ASET vs. RSET analysis with tenability assessment
- Steel and concrete structural fire temperature calculation
- Design fire library and fuel load database
- Report generation
- 500 cloud compute hours/month

### Advanced — $499/user/month
- Multi-mesh CFD (unlimited cells, parallel compute)
- Pyrolysis fire spread model
- Smoke control system simulation (mechanical exhaust, pressurization)
- Water spray suppression and gaseous agent modeling
- Structural fire capacity assessment (Eurocode, AISC)
- Travelling fire methodology
- Multi-scenario and sensitivity analysis
- Probabilistic fire risk (Monte Carlo)
- BIM/IFC import
- API access
- Team collaboration and AHJ sharing

### Enterprise — Custom
- On-premise deployment for government/defense clients
- Custom material pyrolysis characterization
- Integration with BIM platforms (Revit, ArchiCAD)
- WUI (wildland-urban interface) fire modeling
- Training and certification program for fire engineers
- Dedicated support with fire engineering SMEs

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | FireSim Advantage |
|-----------|-----------|------------|-------------------|
| FDS/Smokeview (NIST) | Free, gold-standard CFD fire physics, massive validation database | Text-file input, command-line, steep learning curve, no egress, desktop compute | Same physics with modern GUI, cloud compute, integrated egress and structural |
| PyroSim (Thunderhead) | Best FDS GUI, well-established, good documentation | $4-6K/seat, desktop only, still requires FDS expertise, separate from Pathfinder | Cloud-native, integrated fire + egress + structural, guided workflows |
| Pathfinder (Thunderhead) | Best-in-class egress visualization, 3D steering model | $4K/seat, no fire coupling (manual import from FDS), desktop only | Natively coupled fire-egress with real-time FED calculation |
| CFAST (NIST) | Fast zone model, free, good for simple scenarios | Two-zone only, misses stratification and complex geometry effects | CFD for complex scenarios + zone model for quick studies in one platform |
| buildingEXODUS (UoG) | Advanced behavioral model, research-validated | Expensive ($10K+), academic-focused, complex setup, limited fire coupling | Integrated platform, easier setup, cloud scalability, professional reporting |

---

## MVP Scope (v1.0)

### In Scope
- CFD fire simulation (LES) for single-room and atrium scenarios with prescribed HRR fire
- Smoke spread visualization: soot density, temperature, visibility at slice planes
- Smoke detector and sprinkler activation (RTI-based)
- Basic egress simulation: agent-based, density-dependent speed, pre-movement time
- ASET vs. RSET calculation with tenability criteria (visibility, temperature, CO)
- DXF/CAD floor plan import with extrusion to 3D geometry
- Design fire library: 10 common scenarios (office, retail, residential)
- Steel member temperature calculation (ISO 834 and natural fire from CFD)
- Cloud compute with 3D web-based results viewer
- PDF fire engineering report

### Out of Scope (v1.1+)
- Pyrolysis model for fire spread on surfaces
- Water spray suppression simulation
- Smoke control system modeling (mechanical exhaust, pressurization)
- BIM/IFC import
- Travelling fire methodology
- Probabilistic fire risk (Monte Carlo)
- Structural capacity assessment
- WUI fire modeling
- Elevator evacuation

### MVP Timeline: 16-20 weeks
