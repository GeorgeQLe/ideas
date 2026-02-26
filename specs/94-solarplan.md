# SolarPlan — Solar PV System Design and Yield Analysis Platform

## Executive Summary

SolarPlan is a cloud-based solar PV system design and energy yield analysis platform for utility-scale, commercial, and residential solar projects. It replaces PVsyst ($1,300+/seat), HelioScope ($500+/month), Aurora Solar ($200+/month), SAM (limited usability), and PV*SOL ($1,500+/seat) with an integrated browser-based environment covering site assessment, 3D system layout, energy yield simulation, electrical design, financial modeling, and grid interconnection analysis.

---

## Problem Statement

**The pain:**
- PVsyst is the industry standard for yield assessment but has a steep learning curve, dated Windows-only UI, and costs $1,300+/year per seat; bankable energy yield reports require PVsyst, creating a monopoly bottleneck
- HelioScope and Aurora Solar have modern UIs for layout but lack the simulation depth for utility-scale projects — bankable reports still require PVsyst validation
- Utility-scale solar projects require iterating between layout tools (AutoCAD, HelioScope), yield simulation (PVsyst), electrical design (manual spreadsheets), and financial modeling (Excel) — a fragmented, error-prone workflow
- 3D shading analysis for commercial rooftop systems requires LiDAR or drone survey data import and processing, which current tools handle poorly or require expensive add-ons
- Bifacial module energy gain estimation is increasingly critical (bifacial modules now represent 80%+ of new installations) but most tools use simplified view-factor models that underestimate or overestimate rear-side irradiance by 5-15%

**Current workarounds:**
- Using HelioScope for quick layout and preliminary yield, then rebuilding the entire project in PVsyst for bankable yield assessment — duplicating weeks of work
- Building custom Excel models for financial analysis (LCOE, IRR, NPV) because PVsyst's financial module is limited and Aurora focuses on residential proposals
- Using manual calculations or rules of thumb for string sizing, cable sizing, and voltage drop — risking code violations and energy losses
- Running SAM (NREL, free) for research but it lacks commercial-grade layout tools, 3D shading, and bankable report formatting

**Market size:** The solar PV design software market is approximately $900 million (2024), growing at 18% CAGR driven by the global solar installation rate exceeding 400 GW/year. There are an estimated 200,000+ solar engineers, designers, and installers worldwide. Global solar investment exceeded $380 billion in 2024, and every project requires design and yield analysis software.

---

## Target Users

### Primary Personas

**1. Carlos — Solar Project Developer at a Utility-Scale IPP**
- Develops 50-500 MW solar PV projects from site selection through financial close
- Uses PVsyst for yield assessment and custom Excel for financial modeling, spending 3-4 weeks per project iteration
- Needs: integrated site assessment → layout → yield → financial model workflow that produces bankable P50/P90 yield reports accepted by lenders and investors

**2. Aisha — Commercial Solar Designer at an EPC Contractor**
- Designs 200 kW - 5 MW commercial rooftop and ground-mount systems, handling 10-15 projects simultaneously
- Uses HelioScope for layout and PVsyst for yield, manually transferring between tools and reconciling differences
- Needs: single platform for layout, 3D shading, yield simulation, string sizing, and permit-ready single-line diagrams with automated NEC compliance

**3. Tom — Residential Solar Sales Engineer**
- Designs residential systems for a regional installer, producing 20-30 proposals per week
- Uses Aurora Solar but finds it expensive per user and limited for complex roofs or ground-mount systems
- Needs: fast roof layout from satellite/LiDAR imagery, accurate shading and yield, instant financial proposals with utility rate modeling, and integration with CRM and permitting

---

## Solution Overview

SolarPlan is a cloud-based solar PV design platform that:
1. Provides AI-assisted site assessment using satellite imagery, LiDAR data, and GIS layers (parcel boundaries, zoning, setbacks, grid proximity) to evaluate solar potential before design begins
2. Offers interactive 3D system layout for ground-mount (fixed-tilt, single-axis tracker, dual-axis), rooftop (flush, tilted, east-west), and carport configurations with automatic module placement and obstruction avoidance
3. Runs bankable-quality energy yield simulation using hourly TMY/P50/P90 weather data, Perez transposition model, detailed cell temperature modeling, DC/AC conversion with manufacturer inverter efficiency curves, and comprehensive loss chain
4. Performs electrical design: string sizing, combiner box layout, inverter loading, cable sizing with voltage drop calculation, and NEC/IEC code compliance checking with single-line diagram generation
5. Generates financial models (LCOE, IRR, NPV, payback period) with utility rate schedule modeling, incentive programs (ITC, depreciation, RECs), and debt/equity structuring for project finance

---

## Core Features

### F1: Site Assessment and Solar Resource
- Satellite imagery integration: automatic roof detection and segmentation from Google/Bing/Nearmap imagery using computer vision
- LiDAR data import: DSM/DTM processing from drone surveys, airborne LiDAR (USGS 3DEP), or photogrammetry point clouds for accurate 3D terrain and roof modeling
- Solar resource data: TMY datasets from NSRDB (NREL), Meteonorm, SolarGIS, Solcast with site-specific P50/P90 confidence intervals
- GHI, DHI, DNI decomposition and quality control with gap-filling for measured on-site data
- Horizon shading: import or compute far-field horizon profile from terrain data (SRTM, local DTM)
- GIS layers overlay: parcel boundaries, zoning restrictions, setback requirements, wetlands, flood zones, transmission lines, substations
- Albedo estimation from land cover classification (grass, sand, concrete, snow) for bifacial gain calculation
- Preliminary yield estimate and capacity factor before detailed design

### F2: 3D System Layout and Design
- Ground-mount configurations: fixed-tilt (optimal, latitude, custom), single-axis horizontal tracker (N-S, E-W, tilted), dual-axis tracker
- Rooftop configurations: flush-mount, tilted racking, east-west (low-tilt), ballasted, integrated (BIPV)
- Automatic module placement: fill available area respecting setbacks, obstructions, row spacing (GCR optimization), access aisles, fire pathways (IFC setbacks)
- Tracker layout: automatic table sizing, motor/bearing placement, torque tube and pier spacing per manufacturer specs (NEXTracker, Array Technologies, Soltec)
- 3D obstruction modeling: HVAC units, vents, skylights, parapets, trees, neighboring buildings with shadow casting
- Terrain following: pile height variation for uneven ground, grading limits, slope analysis
- Module selection from comprehensive database: manufacturer specs (Pmax, Isc, Voc, temperature coefficients, bifaciality factor, dimensions) updated regularly
- Inverter selection: string inverter, central inverter, microinverter, DC optimizer — with MPPT input range validation
- Interactive 3D visualization with sun-path animation and shadow preview at any date/time
- Design variant comparison: side-by-side yield, cost, and LCOE comparison for different layouts, tilt angles, tracker types

### F3: Energy Yield Simulation
- Transposition model: Perez (1990) for plane-of-array irradiance from GHI/DHI/DNI; Hay-Davies, Klucher, isotropic options
- Bifacial irradiance: 2D view-factor model and ray-tracing for rear-side irradiance accounting for ground albedo, structure shading, and row-to-row reflection
- Cell temperature models: Sandia, Faiman, NOCT, PVsyst-equivalent thermal model with wind speed and mounting configuration effects
- Single-diode model: compute IV curve from manufacturer datasheet parameters (Rsh, Rs, n, IL, I0) at each timestep with irradiance and temperature
- Module degradation: annual degradation rate, LID (Light Induced Degradation), PID (Potential Induced Degradation), LeTID modeling
- DC losses: module mismatch (Monte Carlo with power tolerance distribution), soiling (monthly profiles or soiling station data), snow loss model, DC wiring losses (I2R per string/homerun)
- Inverter modeling: manufacturer efficiency curves (CEC-weighted, Euro-weighted), clipping losses at high irradiance, night-time consumption, MPPT tracking accuracy
- AC losses: transformer losses (no-load, load-dependent), MV/HV cable losses, grid availability/curtailment
- Shading: 3D ray-casting for near-field shading (obstructions, inter-row), electrical effect modeling at string/submodule level with bypass diode activation
- Hourly (8760) and sub-hourly simulation with time-series output of all loss components
- P50/P90/P99 yield estimation using inter-annual variability and combined uncertainty analysis (resource, model, technology)

### F4: Electrical Design
- String sizing: automatic Voc (cold) and Vmp (hot) calculation against inverter MPPT voltage window per NEC 690.7 / IEC 62548
- String configuration optimization: maximize energy harvest by matching string Vmp to inverter MPPT sweet spot across temperature range
- DC/AC ratio optimization: evaluate clipping losses vs. additional module cost for optimal inverter loading
- Combiner box layout: group strings to combiners, size fuses and disconnects per NEC 690
- Cable sizing: voltage drop calculation per NEC 310 / IEC 60364, ampacity verification with temperature and conduit derating
- Single-line diagram (SLD) auto-generation: modules → strings → combiners → inverters → AC panel → transformer → POI with all protective devices
- Grounding system design: equipment grounding conductor sizing, ground fault detection (GFDI) for ungrounded systems
- Arc fault protection: rapid shutdown compliance per NEC 690.12 with module-level shutdown device placement
- Medium voltage collection system: cable routing, transformer sizing and placement, switchgear specification
- Energy meter and revenue meter placement per utility interconnection requirements

### F5: Financial Modeling
- Capital cost estimation: module, inverter, racking, BOS, labor, soft costs with regional cost databases and custom input
- LCOE (Levelized Cost of Energy) calculation with 25-30 year degradation and O&M escalation
- Project finance: debt/equity structure, DSCR (debt service coverage ratio), IRR (project, equity, investor), NPV, payback period
- Tax incentives: ITC (Investment Tax Credit), PTC (Production Tax Credit), MACRS depreciation, state/local incentives with automatic eligibility checking
- Utility rate modeling: import utility tariff structures (OpenEI), time-of-use rates, demand charges, net metering, feed-in tariffs, PPA pricing
- Revenue streams: energy sales, RECs (Renewable Energy Certificates), capacity payments, ancillary services
- Sensitivity analysis: tornado charts for LCOE sensitivity to module cost, discount rate, degradation, resource uncertainty
- Bankable report generation: P50/P90 yield with uncertainty breakdown, loss tree diagram, monthly/annual energy table, financial summary — formatted for lender/investor review

### F6: Grid Interconnection and Compliance
- Interconnection capacity screening: estimate available capacity at nearby substations using utility hosting capacity maps
- Power flow impact: simplified load flow analysis for distribution-connected projects (voltage rise, thermal loading, reverse power flow)
- Reactive power capability: inverter VAR support modeling (power factor, volt-VAR, volt-watt) per IEEE 1547-2018
- Ramp rate control: model inverter ramp rate limitations for utility-scale grid code compliance
- Frequency response: model inverter frequency-watt droop behavior
- Anti-islanding: verify detection time and compliance with IEEE 1547 / UL 1741
- Grid code compliance documentation: IEEE 1547, FERC Order 2222, California Rule 21, HECO, and international standards (VDE-AR-N 4105, G99)
- Curtailment modeling: energy loss from grid curtailment events (frequency, duration, dispatch constraints)
- Battery storage integration: co-located BESS sizing, charge/discharge strategy, ITC adder, peak shaving, and arbitrage modeling

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | TypeScript, React, Three.js/WebGPU (3D site model, shadow animation), Mapbox GL (GIS, satellite imagery overlay), D3.js (yield charts, loss diagrams, financial graphs) |
| Yield Engine | Rust (transposition models, single-diode IV solver, hourly simulation loop, bifacial view-factor) → WASM for interactive preview + server for full simulation |
| Shading Engine | Rust + WebGPU (GPU-accelerated ray-casting for 3D shadow maps at sub-hourly resolution) |
| Electrical Engine | Rust (string sizing, voltage drop, cable ampacity, SLD generation) → WASM |
| Financial Engine | Python (cash flow modeling, tax incentive logic, sensitivity/Monte Carlo analysis) |
| Backend | Rust (Axum) + Python (FastAPI for weather data processing, financial modeling, report generation) |
| AI/ML | Python (roof segmentation from satellite imagery, automatic obstruction detection, optimal layout ML surrogate model) |
| Database | PostgreSQL (module/inverter database, utility rate tariffs, weather station index), PostGIS (site geospatial data, GIS layers), S3 (LiDAR point clouds, satellite tiles, simulation results) |
| Hosting | AWS (ECS for backend, Lambda for report generation, CloudFront for frontend, S3 for storage) |

---

## Monetization

### Free Tier (Residential Basic)
- Residential systems up to 10 kW
- Satellite imagery roof layout (manual placement)
- Basic yield simulation (annual, no hourly output)
- Simple financial estimate (payback, savings)
- 3 projects

### Pro — $99/month
- Systems up to 1 MW (commercial)
- Full hourly yield simulation with loss chain
- 3D shading analysis
- String sizing and electrical design
- Utility rate modeling and financial analysis
- Proposal and report generation (PDF)
- Module/inverter database access

### Advanced — $249/user/month
- Everything in Pro
- Unlimited system size (utility-scale)
- Bifacial and tracker simulation
- P50/P90 uncertainty analysis
- Bankable yield report format
- Financial modeling (project finance, LCOE, IRR)
- LiDAR import and processing
- Grid interconnection analysis
- API access
- Battery storage co-optimization

### Enterprise — Custom
- White-label proposals and reports
- Custom weather data integration (on-site stations, SolarGIS enhanced)
- Portfolio management for multi-project developers
- ERP and CRM integration (Salesforce, SAP)
- Custom utility rate database maintenance
- Training, certification, and bankability review support

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | SolarPlan Advantage |
|-----------|-----------|------------|-------------------|
| PVsyst | Industry standard for bankable yield, trusted by lenders | $1,300+/seat, dated UI, Windows-only, no layout, no financials | Modern UI, integrated layout → yield → financial, cloud-based |
| HelioScope | Good layout tool, easy commercial design | Limited yield depth, no bankable P50/P90, no utility-scale | Bankable yield simulation, utility-scale support, electrical design |
| Aurora Solar | Best residential proposal workflow | $200+/mo per user, limited for utility-scale, US-focused | All scales (residential to utility), international weather/codes |
| SAM (NREL) | Free, excellent research-grade simulation | No layout tool, no 3D shading, no reports, not bankable | Full design workflow, bankable reports, professional support |
| PV*SOL | Good 3D shading, European market | $1,500+/seat, desktop, limited financial modeling, no utility-scale | Cloud-based, all project scales, superior financial modeling |

---

## MVP Scope (v1.0)

### In Scope
- Satellite imagery-based site layout for ground-mount (fixed-tilt) and rooftop (flush-mount) systems
- Automatic module placement with setbacks and row spacing
- Module and inverter database with manufacturer datasheets
- Hourly energy yield simulation (Perez transposition, single-diode model, basic loss chain)
- 3D near-field shading with inter-row shading for ground-mount
- String sizing with NEC voltage window compliance
- Basic financial summary (LCOE, simple payback, annual savings)
- PDF project report

### Out of Scope (v1.1+)
- Bifacial module simulation
- Single-axis tracker layout and yield modeling
- P50/P90 uncertainty analysis
- Detailed financial modeling (project finance, debt/equity, tax incentives)
- LiDAR point cloud import
- Grid interconnection analysis
- Battery storage integration

### MVP Timeline: 12-16 weeks
