# SoilChem — Agricultural and Soil Science Modeling Platform

## Executive Summary

SoilChem is a cloud-based soil-water-plant modeling platform for crop growth simulation, nutrient cycling, irrigation optimization, and carbon sequestration estimation. It replaces HYDRUS ($3K+/seat), DSSAT (free but fragmented), APSIM (open-source but complex), and CENTURY with a unified browser-based environment that integrates vadose zone hydrology, crop phenology, and nutrient dynamics with remote sensing data and AI-driven yield forecasting.

---

## Problem Statement

**The pain:**
- HYDRUS-1D/2D/3D (PC-Progress) costs $3,000-$12,000/seat for unsaturated flow modeling; RZWQM (USDA) is free but Windows-only with a 1990s interface; combining soil physics with crop models requires manual coupling between incompatible tools
- DSSAT (free) has 40+ years of legacy Fortran code, fragmented documentation, and a GUI that hasn't been meaningfully updated since the 2000s — calibrating a crop model requires editing dozens of text files by hand
- APSIM (open-source) is powerful but requires 2-3 months of training to become productive, with a small user community and limited commercial support
- Precision agriculture companies need field-scale soil-crop models but existing tools don't integrate with modern sensor data (soil moisture probes, drone imagery, satellite NDVI) or IoT irrigation controllers
- Carbon credit markets require verifiable soil carbon sequestration modeling, but CENTURY and RothC models are research tools with no user-friendly interface for agronomists or carbon project developers

**Current workarounds:**
- Using Excel spreadsheets with simplified water balance and nitrogen budget calculations, missing critical soil-plant feedback mechanisms
- Running DSSAT or APSIM with default parameters because calibration is too tedious, producing unreliable yield predictions
- Hiring academic consultants at $150-$250/hour for carbon modeling studies using CENTURY/RothC that could be self-service with better tools
- Precision agriculture platforms (Climate FieldView, Granular) provide yield maps but no process-based soil-crop modeling to explain or predict outcomes

**Market size:** The agricultural modeling and precision agriculture software market is approximately $4.5 billion (2024). There are 200,000+ agricultural engineers, agronomists, and soil scientists worldwide. The voluntary carbon credit market for agriculture is projected to reach $10 billion by 2030, creating massive demand for soil carbon modeling.

---

## Target Users

### Primary Personas

**1. Dr. Priya — Agronomist at a Precision Agriculture Company**
- Develops variable-rate irrigation and fertilization prescriptions for large-scale farms (1,000+ hectares)
- Uses DSSAT for crop modeling but cannot integrate real-time soil sensor and satellite data into model updates
- Needs: field-scale crop model that assimilates sensor data in real time and outputs irrigation/fertilization recommendations

**2. James — Soil Scientist at an Environmental Consultancy**
- Evaluates nitrate leaching risk, pesticide fate, and water quality impacts for regulatory compliance
- Uses HYDRUS-1D for vadose zone modeling but needs to connect it with root zone water quality and crop uptake
- Needs: integrated unsaturated flow + solute transport + crop model for contaminant fate assessment

**3. Sofia — Carbon Project Developer**
- Quantifies soil organic carbon sequestration for voluntary carbon credit markets (Verra, Gold Standard)
- Uses CENTURY model with hand-entered climate and management data, spending weeks per project on data preparation
- Needs: automated soil carbon modeling with standardized MRV (measurement, reporting, verification) outputs that meet carbon registry protocols

---

## Solution Overview

SoilChem is a cloud-based agricultural modeling platform that:
1. Solves unsaturated water flow (Richards equation) coupled with solute transport for nutrient and contaminant fate in the vadose zone
2. Simulates crop growth, phenology, and yield for 25+ major crops using mechanistic models calibrated to local conditions
3. Models nitrogen, phosphorus, and carbon cycling through soil-plant-atmosphere pathways with full mass balance tracking
4. Optimizes irrigation scheduling and fertilization rates using model-predictive control with real-time sensor data assimilation
5. Quantifies soil organic carbon stock changes under different management practices for carbon credit verification and reporting

---

## Core Features

### F1: Vadose Zone Hydrology
- Richards equation solver (1D, 2D axisymmetric, quasi-3D layered) for unsaturated water flow
- Soil water retention curves: van Genuchten, Brooks-Corey, Durner (dual-porosity)
- Hydraulic conductivity: Mualem, Burdine models with hysteresis option
- Pedotransfer functions: Rosetta (neural network), Saxton-Rawls for estimating hydraulic parameters from soil texture
- Preferential flow: dual-porosity and dual-permeability models for macropore flow
- Root water uptake: Feddes (stress response), S-shaped (de Jong van Lier), compensated uptake
- Evapotranspiration: Penman-Monteith (FAO-56), Hargreaves, Priestley-Taylor, dual crop coefficient (Kcb + Ke)
- Drainage: tile drain simulation with Hooghoudt and Kirkham equations
- Soil temperature: heat conduction equation coupled with water flow (Philip-de Vries)
- Boundary conditions: atmospheric (rainfall, ET), free drainage, seepage face, variable head

### F2: Crop Growth and Phenology
- Mechanistic crop models for 25+ crops: wheat, maize, rice, soybean, cotton, potato, sugarcane, sorghum, barley, canola, sunflower, tomato, and more
- Phenology: thermal time (growing degree days), photoperiod sensitivity, vernalization requirements
- Biomass accumulation: radiation use efficiency (RUE) approach with temperature, water, and nitrogen stress factors
- Carbon assimilation: Farquhar-von Caemmerer-Berry (FvCB) photosynthesis model option for C3/C4 crops
- Leaf area index (LAI) dynamics: expansion, senescence, frost kill
- Root growth: depth progression with soil layer density, water, and impedance feedbacks
- Yield formation: harvest index approach with grain filling duration and source-sink dynamics
- Water stress: stomatal conductance reduction (Ball-Berry), turgor-driven growth reduction
- Nitrogen stress: dilution curves (critical N concentration), N-limited photosynthesis
- Crop rotation and cover crop sequences with residue management

### F3: Nutrient Cycling
- Nitrogen cycle: mineralization, immobilization, nitrification, denitrification, volatilization, leaching, plant uptake
- Soil organic matter: multi-pool decomposition (active, slow, passive — CENTURY-based) with first-order kinetics
- Phosphorus cycle: labile/active/stable inorganic P pools, organic P mineralization, sorption isotherms (Langmuir, Freundlich)
- Carbon cycling: soil organic carbon stock tracking with management practice impacts (tillage, cover crops, residue, manure)
- Fertilizer application: broadcast, banded, injected, fertigated — with dissolution and release kinetics for controlled-release fertilizers
- Manure and organic amendment decomposition: CNPS stoichiometry, mineralization rates
- Soil pH effects: on nitrification rate, P availability, and micronutrient dynamics
- Greenhouse gas emissions: N₂O (DNDC-based denitrification), CH₄ (rice paddy methane), CO₂ (soil respiration)
- Mass balance tracking and nutrient budget reports per field and season

### F4: Solute Transport and Water Quality
- Advection-dispersion equation (ADE) for solute transport with linear/nonlinear sorption
- Pesticide fate: degradation (first-order, Arrhenius temperature dependence), sorption (Koc-based), volatilization
- Nitrate leaching to groundwater: breakthrough curve prediction with travel time estimation
- Phosphorus runoff risk: modified P-index calculation with erosion and dissolved P components
- Salinity modeling: salt transport, root zone salinity effects on crop yield (Maas-Hoffman), leaching requirement calculation
- Heavy metal transport: multi-species competitive sorption
- Colloid-facilitated transport: mobile and immobile colloid pools
- Non-equilibrium transport: two-site chemical non-equilibrium, mobile-immobile water
- Water quality compliance reporting against EPA/EU Nitrates Directive thresholds

### F5: Irrigation and Management Optimization
- Soil moisture deficit-based irrigation scheduling with user-defined trigger thresholds (MAD)
- Irrigation methods: surface (furrow, basin), sprinkler, drip/micro — each with application efficiency and uniformity models
- Model-predictive irrigation control: forecast soil moisture using weather forecast + crop model, optimize timing and depth
- Variable-rate irrigation prescription maps from soil variability and crop demand layers
- Fertigation scheduling: nutrient concentration in irrigation water with leaching fraction control
- Sensor data assimilation: soil moisture probes (TDR, capacitance), weather stations, satellite NDVI/LAI for model state updating (EnKF)
- Deficit irrigation strategies: regulated deficit irrigation (RDI) optimization for water-scarce conditions
- Economic optimization: maximize profit (yield revenue minus water + fertilizer cost) subject to water allocation constraints
- What-if scenario comparison: compare management strategies side-by-side with yield, water use, and N-leaching metrics

### F6: Carbon Sequestration and MRV
- Soil organic carbon (SOC) stock change modeling: baseline vs. project scenario comparison over 10-100 year horizons
- Management practice library: no-till, reduced till, cover crops, biochar, compost, residue retention — with calibrated SOC change rates
- IPCC Tier 2/3 compliant carbon accounting methodology
- Verra VM0042 and Gold Standard soil carbon methodology support
- Baseline scenario generation using regional defaults and historical management
- Uncertainty quantification: Monte Carlo parameter uncertainty with 90% confidence intervals on SOC change
- Automated MRV report generation: monitoring period SOC change, leakage assessment, permanence risk buffer
- Integration with soil sampling data for model validation and crediting period verification
- Climate data integration: automatic ERA5/PRISM/SILO weather data extraction for any field location
- Spatial scaling: field-level to project-level (100+ fields) aggregation with stratified sampling design

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | TypeScript, React, Mapbox GL JS (field maps, spatial results), D3.js (soil profiles, time series, nutrient budgets) |
| Vadose Zone Solver | Rust (Richards equation, finite element 1D/2D, implicit time stepping) → WASM + server |
| Crop Model Engine | Rust (phenology, biomass, yield — daily time step) → WASM + server |
| Nutrient Cycling | Rust (C/N/P pools, transformation kinetics, mass balance) |
| Solute Transport | Rust (ADE solver, sorption, degradation) coupled with vadose zone solver |
| AI/ML | Python (yield forecasting via LSTM/transformer, sensor data assimilation via EnKF, surrogate models for optimization) |
| Backend | Rust (Axum) + Python (FastAPI for weather data APIs, satellite imagery processing, carbon reporting) |
| Database | PostgreSQL + PostGIS (soil databases, field boundaries, management records), S3 (weather data, satellite imagery, simulation results) |
| Hosting | AWS multi-region (c7g for simulations, GPU for ML inference) |

---

## Monetization

### Free Tier (Student/Researcher)
- 1D soil water balance for single soil profile
- Basic crop growth simulation (3 crops: wheat, maize, soybean)
- Penman-Monteith ET calculator
- 3 projects

### Pro — $99/month
- Full Richards equation solver (1D)
- 25+ crop models with phenology calibration
- Nitrogen cycling and leaching assessment
- Irrigation scheduling optimization
- Weather data integration (ERA5, local stations)
- PDF agronomic report generation

### Advanced — $249/user/month
- Everything in Pro
- 2D/3D vadose zone modeling
- Full C/N/P cycling with greenhouse gas emissions
- Sensor data assimilation (IoT integration)
- Carbon sequestration modeling with MRV reports
- Pesticide fate and water quality assessment
- Variable-rate prescription map generation
- API access

### Enterprise — Custom
- Multi-farm portfolio management (1,000+ fields)
- Carbon credit project platform (Verra/Gold Standard integration)
- Custom crop model calibration with proprietary trial data
- On-premise deployment for agribusiness
- Integration with farm management platforms (John Deere, Climate FieldView)
- Training and certification

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | SoilChem Advantage |
|-----------|-----------|------------|-------------------|
| HYDRUS | Gold standard vadose zone physics | $3-12K/seat, no crop model, no nutrients | Integrated soil + crop + nutrients |
| DSSAT | Most validated crop model suite | Legacy Fortran, terrible UX, no sensor integration | Modern UX, real-time data assimilation |
| APSIM | Excellent crop-soil modeling, open-source | Steep learning curve, months to learn | Cloud-based, productive in hours |
| CENTURY/RothC | Standard for soil carbon modeling | Research tools, no GUI, manual data prep | Automated carbon MRV, user-friendly |
| RZWQM (USDA) | Good root zone water quality | Windows-only, 1990s UI, limited support | Cloud-native, modern interface, IoT ready |

---

## MVP Scope (v1.0)

### In Scope
- 1D Richards equation solver with van Genuchten retention curves
- Crop growth simulation for wheat, maize, and soybean (thermal time phenology, RUE biomass)
- Penman-Monteith ET with FAO-56 dual crop coefficient
- Basic nitrogen cycling (mineralization, nitrification, plant uptake, leaching)
- Irrigation scheduling based on soil moisture deficit
- Weather data import (CSV, ERA5 extraction)
- Soil profile visualization and water balance dashboard

### Out of Scope (v1.1+)
- 2D/3D vadose zone modeling
- Phosphorus cycling and pesticide fate
- Carbon sequestration modeling (CENTURY-based SOC dynamics)
- Sensor data assimilation and real-time model updating
- Variable-rate irrigation/fertilization prescription maps
- Greenhouse gas emission estimation

### MVP Timeline: 14-18 weeks
