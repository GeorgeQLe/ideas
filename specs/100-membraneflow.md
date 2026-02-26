# MembraneFlow — Membrane Separation Design Platform

## Executive Summary

MembraneFlow is a cloud-based platform for designing, simulating, and optimizing membrane separation systems across reverse osmosis, nanofiltration, ultrafiltration, microfiltration, and gas separation. It replaces ROSA ($0 but locked to DuPont membranes), TorayDS (Toray-only), WAVE ($0 but vendor-locked), and IMSDesign (Hydranautics-only) with a vendor-neutral, physics-based design tool that models any manufacturer's membranes, predicts fouling and cleaning cycles, and optimizes system configuration for minimum lifecycle cost.

---

## Problem Statement

**The pain:**
- Every major membrane manufacturer provides their own free design software (ROSA/WAVE for DuPont/FilmTec, TorayDS for Toray, IMSDesign for Hydranautics, LewaPlus for Lanxess) — but each only models that vendor's membranes, making objective comparison impossible
- Vendor tools use simplified projection models calibrated to clean-membrane lab data; they do not predict real-world fouling, cleaning frequency, or membrane aging — leading to 15-30% oversizing or unexpected performance decline
- Designing a complete water treatment plant requires running 3-4 different vendor tools, manually comparing outputs in spreadsheets, then re-running whenever feed water quality changes
- UF/MF pretreatment design is completely separate from RO/NF design in existing tools; there is no integrated platform modeling the full treatment train
- Gas separation membrane design (H2 recovery, CO2 capture, N2 generation, biogas upgrading) has almost no accessible design tools outside proprietary vendor software

**Current workarounds:**
- Running ROSA, TorayDS, and IMSDesign in parallel to compare vendors, then manually reconciling outputs in Excel — a process that takes days for a single design iteration
- Using rules of thumb for UF/MF sizing (flux × area) with safety factors of 1.5-2x because no fouling prediction exists
- Over-designing systems by 20-40% to compensate for the inability to predict long-term performance, adding $100K-$1M+ in unnecessary capital cost
- Hiring membrane consultants at $200-$500/hour for design optimization that could be systematized

**Market size:** The global membrane market (equipment + replacement) is approximately $30B (2024) with 8-10% annual growth driven by water scarcity and industrial purification needs. There are an estimated 60,000+ membrane system designers/engineers worldwide across water treatment, desalination, food & beverage, pharmaceutical, and industrial gas separation.

---

## Target Users

### Primary Personas

**1. Sofia — Water Treatment Design Engineer at an EPC Firm**
- Designs brackish water RO and seawater desalination plants (1,000-100,000 m3/day)
- Runs ROSA, TorayDS, and IMSDesign for every project to compare membrane brands and present options to clients
- Needs: vendor-neutral RO/NF design tool that models any membrane from published datasheet parameters with honest fouling and aging predictions

**2. David — Process Engineer at a Food & Beverage Company**
- Designs UF and MF systems for juice clarification, dairy whey concentration, and beer filtration
- Uses supplier-provided sizing spreadsheets that don't account for protein fouling or cleaning cycle optimization
- Needs: UF/MF design with fouling prediction, cleaning cycle optimization, and cross-flow/dead-end mode comparison for food-grade applications

**3. Dr. Chen — Gas Separation Engineer at a Petrochemical Company**
- Designs membrane systems for hydrogen recovery from refinery off-gas, nitrogen generation, and CO2 removal from natural gas
- Has no accessible design software; relies on vendor quotes and internal spreadsheets with permeability data
- Needs: gas separation membrane design tool with multi-component permeation modeling, staging optimization, and comparison across polymeric/ceramic membrane types

---

## Solution Overview

MembraneFlow is a cloud-based membrane separation design platform that:
1. Models all membrane processes (RO, NF, UF, MF, gas separation) with vendor-neutral physics: solution-diffusion for RO/NF/gas, pore-flow for UF/MF, using published membrane parameters from any manufacturer
2. Predicts real-world performance including concentration polarization, fouling progression (organic, colloite, biofouling, scaling), membrane compaction, and aging over the system lifetime
3. Designs complete treatment trains: intake → pretreatment (UF/MF or conventional) → RO/NF → post-treatment, with inter-stage optimization
4. Optimizes system configuration (staging, recovery, flux, cleaning frequency) for minimum total cost of ownership including energy, chemicals, membrane replacement, and downtime
5. Provides energy recovery device modeling (PX, turbocharger, Pelton turbine) for desalination plants and compressor optimization for gas separation

---

## Core Features

### F1: RO/NF System Design
- Solution-diffusion transport model: water flux Jw = A × (ΔP - Δπ), solute flux Js = B × (Cf - Cp), with membrane-specific A, B, and salt rejection coefficients from manufacturer datasheets
- Concentration polarization: film theory model with Sherwood correlation for spacer-filled channels; CP modulus calculation at each element position
- Osmotic pressure: van't Hoff equation for dilute solutions, Pitzer model for high-salinity brines (seawater, brine concentrators), OLI-compatible activity coefficients
- Multi-ion rejection: Spiegler-Kedem model for individual ion rejection (Na+, Ca2+, Mg2+, Cl-, SO42-, HCO3-, SiO2, B) with valence and hydration radius effects
- System configuration: single-pass, two-pass, concentrate recirculation, closed-circuit RO (CCRO), batch RO, multi-stage with inter-stage pumping
- Element-by-element calculation along pressure vessel with pressure drop (Schock-Miquel correlation), permeate backpressure, and temperature correction
- Scaling prediction: saturation indices for CaCO3 (LSI, S&DSI), CaSO4, BaSO4, SrSO4, silica, and CaF2 at each element position with antiscalant dosing recommendations
- Permeate quality blending: front/rear element permeate split for partial second-pass designs

### F2: UF/MF Design
- Pore-flow transport: Hagen-Poiseuille model for clean membrane flux; resistance-in-series model (Rm + Rf + Rc) for fouled operation
- Module configurations: hollow fiber (inside-out, outside-in), flat sheet, tubular, spiral wound — with geometry-specific flow and pressure drop models
- Operating modes: cross-flow (steady-state) and dead-end (batch accumulation) with automatic backwash cycle calculation
- Fouling models: cake filtration (Ruth equation), pore blocking (Hermia models: complete, standard, intermediate blocking), gel layer formation for protein-containing feeds
- Backwash and CEB (chemically enhanced backwash) optimization: frequency, duration, chemical type (NaOCl, citric acid, NaOH) and concentration for flux recovery
- CIP (clean-in-place) scheduling based on irreversible fouling accumulation and TMP (trans-membrane pressure) trigger levels
- Fiber breakage tracking and integrity test simulation (pressure decay test, bubble point)
- Direct integration with downstream RO/NF: UF permeate quality as RO feed, with SDI and turbidity prediction

### F3: Gas Separation Membrane Design
- Solution-diffusion model for polymeric membranes: Ji = (Pi/l) × (pf,i - pp,i) with gas-specific permeability coefficients and membrane thickness
- Multi-component permeation: cross-flow, counter-current, and co-current flow patterns solved with differential element method along module length
- Selectivity modeling: ideal selectivity (α = PA/PB) with real mixture effects (plasticization, competitive sorption) for CO2/CH4, H2/N2, O2/N2, He/N2
- Module types: hollow fiber, spiral wound, plate-and-frame — with module-specific packing density and pressure drop
- Staging: single-stage, two-stage (with permeate recycle or retentate recycle), three-stage cascade optimization
- Applications library: H2 recovery from refinery off-gas, CO2 removal from natural gas, N2 generation from air, biogas upgrading (CO2/CH4), vapor recovery (VOC), air dehumidification
- Compressor sizing: feed compression and inter-stage compression with isentropic efficiency, power consumption, and cooling requirements
- Comparison mode: side-by-side evaluation of polymeric vs. ceramic vs. carbon molecular sieve membranes

### F4: Fouling Prediction and Cleaning Optimization
- Feed water characterization: SDI, MFI (modified fouling index), TOC, turbidity, particle size distribution, and biological activity indicators
- Organic fouling model: NOM adsorption and cake formation based on SUVA and molecular weight distribution
- Colloidal fouling: DLVO theory-based particle deposition with zeta potential and ionic strength effects
- Biofouling model: biofilm growth kinetics (Monod) with chlorine/chloramine residual decay and biocide dosing optimization
- Scaling kinetics: induction time, crystal growth rate, and scale layer resistance for common scalants
- Membrane aging model: hydrolysis rate for polyamide TFC membranes as a function of pH, temperature, and oxidant exposure; flux decline and salt passage increase over time
- Cleaning optimization: recommend cleaning chemical (NaOH, HCl, citric acid, EDTA, NaOCl, proprietary formulations), temperature, pH, soak time, and frequency based on fouling type
- Membrane replacement scheduling: predict optimal replacement timing based on normalized permeate flow, salt passage, and differential pressure trends

### F5: Energy Recovery and Cost Optimization
- Energy recovery devices: pressure exchanger (PX), turbocharger, Pelton turbine — with manufacturer performance curves and efficiency maps
- Specific energy consumption (SEC) calculation: kWh/m3 for water, kWh/Nm3 for gas — broken down by high-pressure pump, booster pump, ERD, and auxiliary systems
- Thermodynamic minimum energy analysis: second-law efficiency relative to minimum separation energy
- CAPEX estimation: membranes, pressure vessels, pumps, piping, ERD, instrumentation, civil works, and pretreatment — parametric cost models
- OPEX estimation: energy, membrane replacement, chemicals (antiscalant, cleaning, pH adjustment), labor, and maintenance
- Levelized cost of water (LCOW) or levelized cost of gas (LCOG) with discount rate, plant lifetime, and capacity factor
- Design optimization: multi-objective optimization (minimize SEC vs. maximize recovery vs. minimize LCOW) with Pareto front visualization
- What-if analysis: feed water quality variation, energy price changes, membrane price changes, and regulatory scenario impacts

### F6: Reporting and Compliance
- System P&ID generation with membrane arrays, piping, pumps, ERD, chemical dosing, and instrumentation
- Detailed design report: element-by-element performance tables, scaling projections, chemical consumption, and energy breakdown
- Membrane specification sheets for procurement: membrane model, number of elements, pressure vessels, array configuration
- Water quality compliance: WHO drinking water, EPA SDWA, EU Drinking Water Directive — permeate quality vs. standards with safety margins
- Brine/concentrate disposal analysis: volume, composition, and disposal options (deep well, evaporation pond, ZLD, ocean outfall)
- Environmental impact: CO2 footprint per m3 of product water, chemical consumption intensity
- PDF/Excel/DXF export for engineering deliverables
- API for integration with SCADA and plant historians for model calibration with operational data

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | TypeScript, React, SVG/Canvas (P&ID, system schematic), D3.js (performance plots, Pareto fronts) |
| RO/NF Solver | Rust (element-by-element solver, multi-ion Spiegler-Kedem, scaling indices) → WASM for interactive design + server for optimization |
| UF/MF Solver | Rust (resistance-in-series, Hermia fouling, backwash cycle optimization) → server |
| Gas Separation Solver | Rust (differential permeation, multi-component, staging) → server |
| Fouling/Aging Engine | Python (scikit-learn/PyTorch for fouling prediction from water quality, cleaning optimization) |
| Optimization | Python (SciPy, NSGA-II for multi-objective, Bayesian optimization for cost) |
| Backend | Rust (Axum) + Python (FastAPI for ML, optimization, cost models) |
| Database | PostgreSQL (membrane database, water chemistry, project data), S3 (reports, simulation archives) |
| Hosting | AWS multi-region |

---

## Monetization

### Free Tier (Student)
- Single-stage RO/NF design with up to 6 elements
- 50 membranes in database (major models from DuPont, Toray, Hydranautics)
- Basic scaling prediction (LSI only)
- No fouling modeling or optimization
- 3 projects

### Pro — $99/month
- Multi-stage RO/NF design (unlimited elements)
- UF/MF pretreatment design
- Full membrane database (500+ models, all major vendors)
- Scaling prediction (all scalants) with antiscalant recommendations
- Energy recovery device modeling
- Basic fouling prediction
- Report generation (PDF/Excel)

### Advanced — $249/user/month
- Everything in Pro
- Gas separation membrane design
- Full fouling prediction and cleaning optimization
- Membrane aging and replacement scheduling
- Multi-objective cost optimization
- Full treatment train (intake → pretreatment → RO → post-treatment)
- CAPEX/OPEX/LCOW estimation
- API access
- Collaborative design

### Enterprise — Custom
- On-premise or private cloud deployment
- Custom membrane models (proprietary membranes, new materials)
- Integration with plant SCADA/historian for real-time model calibration
- Fleet management: multi-plant membrane performance tracking and benchmarking
- Membrane autopsy data integration for fouling model refinement
- Training and certification program

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | MembraneFlow Advantage |
|-----------|-----------|------------|----------------------|
| ROSA / WAVE (DuPont) | Free, well-validated for FilmTec membranes | Only DuPont membranes, no fouling prediction, no UF/MF, no gas | Vendor-neutral, fouling/aging models, full treatment train |
| TorayDS (Toray) | Free, good for Toray membranes | Only Toray membranes, basic projection, Windows-only | Any vendor's membranes, browser-based, cost optimization |
| IMSDesign (Hydranautics) | Free, Hydranautics catalog | Only Hydranautics, no pretreatment integration | All vendors, integrated UF/MF + RO/NF, gas separation |
| LewaPlus (Lanxess) | Includes IX design alongside membranes | Limited membrane selection, basic fouling | Comprehensive fouling models, energy recovery, LCOW |
| In-house spreadsheets | Customizable, include site-specific data | No physics-based fouling, error-prone, not reusable | Rigorous transport models, automated optimization, collaborative |

---

## MVP Scope (v1.0)

### In Scope
- RO/NF system design with element-by-element solver (solution-diffusion, concentration polarization)
- Membrane database with 100+ models from DuPont, Toray, Hydranautics, and LG Chem
- Multi-ion rejection (Spiegler-Kedem) for Na, Ca, Mg, Cl, SO4, HCO3, SiO2, B
- Scaling prediction (LSI, S&DSI, CaSO4, SiO2) at each element position
- System configurations: single-pass, two-pass, concentrate recirculation
- Energy recovery device modeling (PX efficiency curve)
- Specific energy consumption and basic cost estimation
- PDF design report with element-by-element tables

### Out of Scope (v1.1+)
- UF/MF pretreatment design
- Gas separation membrane design
- Fouling prediction and cleaning optimization
- Membrane aging and replacement scheduling
- Multi-objective cost optimization (LCOW Pareto)
- SCADA/historian integration for operational calibration

### MVP Timeline: 14-18 weeks
