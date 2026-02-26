# WindFarm — Wind Farm Layout Optimization Platform

## Executive Summary

WindFarm is a cloud-based platform for wind resource assessment, wind farm layout optimization, and energy yield estimation. It replaces WAsP ($5K+/seat), WindPRO ($10K+/seat), OpenWind ($8K+/seat), and DNV WindFarmer ($15K+/seat) with an integrated browser-based tool that combines wind flow modeling, wake analysis, turbine micrositing, and bankable energy yield reports — all accessible without desktop installation or six-figure license stacks.

---

## Problem Statement

**The pain:**
- Wind farm development software is fragmented and expensive: WAsP (DTU) costs $5,000/seat for the flow model alone, WindPRO (EMD) costs $10,000-$25,000/seat with modules, and DNV WindFarmer costs $15,000-$40,000/seat — a full tool stack easily exceeds $50K per analyst
- Layout optimization requires coupling wind resource data, terrain/roughness models, wake loss models, turbine power curves, noise constraints, and financial models — but existing tools handle subsets or require manual data piping between programs
- Wake modeling is the single largest source of uncertainty in energy yield estimates (5-15% array losses), yet most commercial tools still default to simple Jensen/Park models while advanced dynamic wake meandering or CFD-based approaches remain locked in research codes
- Bankable energy yield assessments required by project financiers (P50, P75, P90 estimates with uncertainty quantification) demand expensive consulting engagements ($50K-$200K per project) partly because the tools are hard to use
- Emerging markets (Africa, Southeast Asia, Latin America) have thousands of potential wind sites but few local analysts who can afford the legacy tool stack

**Current workarounds:**
- Using WAsP for wind resource and manually exporting to spreadsheets for layout optimization and financial modeling
- Running open-source tools (FLORIS from NREL, PyWake from DTU) that lack GUI, terrain modeling, and bankable reporting
- Hiring specialized consultants at $150-$300/hour for energy yield assessments that could be partially automated
- Using Windographer ($1,500/year) for data analysis separately from layout tools, creating data handoff friction

**Market size:** The global wind energy market exceeds $100 billion annually in new installations. Wind resource assessment and energy yield consulting is a $2-3 billion segment. There are 50,000+ wind energy engineers, resource analysts, and developers worldwide, with rapid growth in emerging markets.

---

## Target Users

### Primary Personas

**1. Lars — Wind Resource Analyst at a Developer**
- Processes met mast and LiDAR data, performs measure-correlate-predict (MCP) analysis, and creates long-term wind resource estimates
- Uses WAsP + Windographer + Excel, spending days on data transfer and format conversion
- Needs: integrated platform from raw data ingestion through resource assessment to energy yield with auditable workflows

**2. Priya — Wind Farm Layout Engineer**
- Optimizes turbine positions to maximize energy yield while satisfying noise, visual impact, setback, and grid connection constraints
- Uses WindPRO or OpenWind but finds the optimization algorithms slow and opaque
- Needs: fast multi-objective layout optimization with real-time wake visualization and constraint handling

**3. Thomas — Project Finance Analyst at a Bank**
- Reviews energy yield assessments for wind farm financing decisions, needs to understand P50/P75/P90 estimates and uncertainty breakdowns
- Receives PDF reports from consultants with no way to interrogate assumptions or run sensitivities
- Needs: transparent, auditable energy yield reports with interactive uncertainty exploration

---

## Solution Overview

WindFarm is a cloud-based wind energy platform that:
1. Ingests met mast, LiDAR, and reanalysis data for wind resource assessment with automated quality control and long-term correction (MCP)
2. Models wind flow over complex terrain using WAsP-like linearized models and optional CFD (RANS) for complex sites
3. Computes wake losses using state-of-the-art models (Jensen, Frandsen, Bastankhah-Porte-Agel, dynamic wake meandering) with full uncertainty quantification
4. Optimizes turbine layouts for maximum AEP subject to noise, visual, setback, environmental, and grid constraints using genetic algorithm and gradient-based optimizers
5. Produces bankable energy yield reports (P50/P75/P90) with full uncertainty waterfall — suitable for project finance due diligence

---

## Core Features

### F1: Wind Resource Assessment
- Met mast data import: time series from Campbell Scientific, NRG, Ammonit, Kintech, generic CSV
- LiDAR data integration: Leosphere, ZX, Galion profiles with data quality filtering
- Automated quality control: sensor icing detection, tower shadow correction, stuck sensor identification, timestamp validation
- Reanalysis data access: ERA5, MERRA-2, NORA3 with automatic download and bias correction
- Measure-correlate-predict (MCP): linear regression, variance ratio, matrix method, Weibull scaling for long-term correction
- Weibull and Rayleigh distribution fitting per sector with directional frequency analysis
- Wind shear analysis: power law and log law profile fitting, shear exponent variability
- Turbulence intensity assessment: per wind speed bin and direction, IEC 61400-1 normal turbulence model (NTM) classification
- Wind resource maps: horizontal extrapolation across site using flow model

### F2: Wind Flow Modeling
- Linearized flow model (WAsP-like): IBZ (internal boundary layer) model with orographic speedup (BZ model) and roughness change effects
- CFD-based flow model: steady RANS (k-epsilon, k-omega SST) for complex terrain where linearized models break down (RIX > 5%)
- Terrain data: SRTM (30m), ASTER, national LiDAR DEMs, user-uploaded topography
- Roughness modeling: CORINE, ESA WorldCover, user-defined roughness maps, seasonal roughness variation
- Displacement height for forested sites
- Speedup factor and flow inclination angle at each turbine position
- Thermal stability effects: atmospheric stability classification and correction (Monin-Obukhov)
- Mesoscale-to-microscale downscaling: WRF coupling for sites with limited met data

### F3: Wake Modeling and Array Losses
- Jensen/Park wake model (single and top-hat profiles) for rapid screening
- Bastankhah-Porte-Agel Gaussian wake model with self-similar profile
- Frandsen effective turbulence model for fatigue load assessment
- Dynamic wake meandering (DWM) model for wake-induced fatigue and power fluctuations
- Wake superposition methods: linear, root-sum-square, max deficit
- Partial wake overlap computation with rotor-plane integration
- Turbine-specific thrust coefficient curves (Ct) as function of wind speed
- Added turbulence from wakes: Frandsen and Quarton models
- Blockage effects: induction zone modeling (Meyer Forsting, Branlard)
- Large wind farm effects: deep array wake losses and internal boundary layer growth

### F4: Layout Optimization
- Genetic algorithm optimizer for global search over turbine positions
- Gradient-based refinement using adjoint-based AEP gradients
- Constraint handling: minimum turbine spacing (rotor diameter multiples), site boundary, exclusion zones, maximum turbine count
- Noise constraint optimization: ISO 9613-2 propagation model with receptor-specific limits, automatic curtailment scheduling
- Visual impact: zone of theoretical visibility (ZTV), wireframe visualization, shadow flicker calculation (hours/year at receptors)
- Setback constraints: distance from dwellings, roads, power lines, property boundaries
- Cable routing optimization: inter-array cable layout minimizing trench length and electrical losses
- Substation placement optimization
- Multi-objective: maximize AEP, minimize wake losses, minimize cable cost, satisfy all constraints
- Scenario comparison: side-by-side layout variants with AEP/cost/noise tradeoff visualization

### F5: Energy Yield Estimation
- Gross energy calculation: turbine power curves (manufacturer-supplied or IEC-corrected) integrated with site wind distribution
- Loss waterfall: wake losses, availability, electrical losses, turbine performance, environmental (icing, degradation), curtailment (noise, grid, bat/bird)
- Uncertainty analysis: wind speed measurement uncertainty, long-term correction uncertainty, wake model uncertainty, power curve uncertainty — combined using Monte Carlo or analytical methods
- P50, P75, P90 net energy estimates with confidence intervals
- IEC 61400-12 power curve correction for air density, turbulence, shear
- Turbine suitability check: IEC class verification (wind speed, turbulence, extreme gusts) against turbine design envelope
- Site-specific power curve adjustment for turbulence and shear
- Monthly and seasonal energy breakdown
- Degradation modeling: annual energy degradation rate over project lifetime (20-30 years)

### F6: Reporting and Collaboration
- Automated bankable energy yield report generation (PDF) following industry standards (IEC 61400-15 framework)
- Uncertainty waterfall chart: visual breakdown of each uncertainty component
- Project dashboard: key metrics (AEP, capacity factor, wake loss, LCOE estimate) at a glance
- Version-controlled project files with full audit trail of assumptions and model inputs
- Collaboration: shared projects with role-based access (analyst, reviewer, approver)
- Data export: CSV, Excel, WAsP-compatible formats for interoperability
- API for integration with financial models and project management tools
- Regulatory submission packages: noise maps, shadow flicker reports, visual impact assessments

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | TypeScript, React, Mapbox GL / Deck.gl (terrain + turbine layout visualization), D3.js (wind roses, loss waterfalls) |
| Flow Model | Rust (linearized IBZ/BZ wind model, CFD RANS solver for complex terrain) |
| Wake Engine | Rust (Jensen, Bastankhah, DWM wake models) → WASM for real-time browser preview |
| Optimizer | Rust (genetic algorithm + gradient-based layout optimization) |
| Backend | Python (FastAPI) for data processing, MCP, Weibull fitting; Rust (Axum) for compute orchestration |
| AI/ML | Python (scikit-learn, XGBoost) for MCP methods, wind power forecasting, anomaly detection in met data |
| Database | PostgreSQL (projects, turbine library, met data metadata), TimescaleDB (time series wind data), S3 (raw data, results, terrain tiles) |
| Hosting | AWS (c7g for flow/wake, r7g for large terrain CFD, CloudFront for terrain tile serving) |

---

## Monetization

### Free Tier (Student)
- Single met mast, Weibull fitting, wind rose
- Up to 10 turbines, Jensen wake model
- Basic AEP calculation (no uncertainty)
- 2 projects

### Pro — $199/month
- Unlimited met masts, MCP long-term correction
- Linearized flow model (WAsP-like)
- All wake models (Jensen, Bastankhah, DWM)
- Layout optimization with constraints
- Full energy yield with P50/P75/P90 uncertainty
- Noise and shadow flicker analysis
- Automated report generation
- 20 projects

### Advanced — $499/user/month
- CFD flow model for complex terrain
- Blockage modeling
- Cable routing optimization
- Turbine suitability (IEC class check)
- API access
- Team collaboration (5 seats included)
- Unlimited projects

### Enterprise — Custom
- On-premise deployment for utilities/IPPs
- Custom turbine library management with NDA-protected curves
- Integration with SCADA, financial models, GIS systems
- Multi-farm portfolio analysis
- Dedicated support and model validation consulting
- White-label reporting

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | WindFarm Advantage |
|-----------|-----------|------------|-------------------|
| WAsP (DTU) | Industry-standard flow model, trusted for 30+ years | No layout optimization, no wake models, $5K/seat, desktop-only | Integrated flow + wake + optimization in browser |
| WindPRO (EMD) | Comprehensive modules, regulatory compliance | $10-25K/seat, steep learning curve, Windows-only | Cloud-native, faster optimization, modern UX |
| OpenWind (AWS Truepower/UL) | Good layout optimization, US market leader | $8-15K/seat, aging UI, limited terrain modeling | Better terrain CFD, global reanalysis access, lower cost |
| DNV WindFarmer | Strong energy yield uncertainty framework, bankable | $15-40K/seat, slow solver, enterprise-only | 10x faster wake computation, accessible pricing |
| FLORIS (NREL) | Free, open-source, research-grade wake models | No GUI, no terrain, no resource assessment, not bankable | Full workflow from data to bankable report |

---

## MVP Scope (v1.0)

### In Scope
- Met mast data import (CSV) with automated quality control
- Weibull fitting and wind rose visualization on interactive map
- Linearized flow model over imported terrain (SRTM 30m)
- Jensen wake model with basic superposition
- Manual and auto-optimized turbine layout (genetic algorithm with spacing constraints)
- Gross and net AEP calculation with loss waterfall
- Turbine library (10 common turbine models with power/thrust curves)
- Basic PDF energy yield report

### Out of Scope (v1.1+)
- CFD flow model, advanced wake models (DWM, Bastankhah)
- MCP long-term correction
- Noise/shadow flicker/visual impact analysis
- Cable routing optimization
- P50/P75/P90 uncertainty quantification
- LiDAR data integration

### MVP Timeline: 14-18 weeks
