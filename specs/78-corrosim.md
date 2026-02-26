# CorroSim — Corrosion Engineering and Protection Design Platform

## Executive Summary

CorroSim is a cloud-based platform for predicting corrosion rates, designing cathodic protection systems, and selecting materials for corrosive environments — covering electrochemical kinetics, galvanic coupling, CO2/H2S corrosion, stress corrosion cracking, and cathodic protection (CP) modeling. It replaces COMSOL Corrosion Module ($12K+/seat), OLI Systems ($15K+/seat), and Honeywell Predict ($20K+/seat) with an integrated browser-based environment for pipeline engineers, offshore engineers, and corrosion specialists.

---

## Problem Statement

**The pain:**
- Corrosion costs the global economy $2.5 trillion/year (3.4% of GDP per NACE study), yet most corrosion engineering is done with spreadsheets and rules of thumb rather than predictive simulation
- COMSOL Corrosion Module costs $4,000 base + $4,000 corrosion + $4,000 electrochemistry = $12,000+ per seat; OLI Systems (water chemistry/corrosion prediction) costs $15,000-$30,000/year; Honeywell Predict (pipeline corrosion) costs $20,000+/year — fragmented across different vendors
- Pipeline operators manage 3+ million km of oil and gas pipelines globally; internal corrosion (CO2, H2S, organic acids) is the leading cause of pipeline failure, yet corrosion rate prediction requires coupling fluid flow, water chemistry, and electrochemistry that no single affordable tool handles
- Cathodic protection (CP) design for pipelines, offshore structures, and reinforced concrete is done using oversimplified empirical methods (current density tables) that waste anode material or leave structures under-protected
- Material selection for corrosive service (e.g., choosing between carbon steel, 13Cr, duplex SS, nickel alloys) involves expensive laboratory testing ($50K-$200K per qualification program) when simulation could narrow candidates before testing

**Current workarounds:**
- Using de Waard-Milliams or Norsok M-506 spreadsheet models for CO2 corrosion prediction — single-point estimates that miss flow effects, top-of-line corrosion, and localized attack
- Running COMSOL for CP design on simple geometries, but struggling with large-scale systems (100+ km pipelines, offshore jackets with 1000+ nodes)
- Using OLI or Multicorp for water chemistry and corrosion rate but without coupling to flow simulation or CP design
- Relying on historical inspection data (pigging, UT) extrapolation for remaining life assessment, rather than physics-based prediction

**Market size:** The corrosion management market is approximately $35 billion/year (coatings, inhibitors, CP, inspection, engineering). The corrosion simulation and prediction software market is approximately $800 million (2024), growing at 9% CAGR driven by aging infrastructure, offshore wind, and hydrogen transition. There are 100,000+ corrosion engineers, materials engineers, and pipeline integrity specialists worldwide, with NACE (now AMPP) having 40,000+ members.

---

## Target Users

### Primary Personas

**1. Carlos — Pipeline Integrity Engineer at an Oil and Gas Operator**
- Manages integrity of 5,000+ km of carbon steel pipelines carrying wet gas and multiphase fluids
- Uses de Waard-Milliams spreadsheet for CO2 corrosion rate prediction, supplemented by in-line inspection (ILI) pigging data
- Needs: flow-coupled internal corrosion prediction (CO2, H2S, organic acids) along pipeline length to prioritize inspection and predict remaining life

**2. Dr. Ingrid — Cathodic Protection Engineer for Offshore Structures**
- Designs sacrificial anode and impressed current cathodic protection systems for offshore platforms, subsea pipelines, and wind farm foundations
- Uses DNVGL-RP-B401 empirical tables for anode sizing, which are conservative and result in over-designed (expensive) CP systems
- Needs: 3D CP modeling with BEM (boundary element method) to optimize anode placement, predict potential distribution, and demonstrate protection per standards

**3. Arun — Materials and Corrosion Engineer at a Refinery**
- Selects materials for process equipment exposed to high-temperature sulfidation, naphthenic acid corrosion, and chloride stress corrosion cracking
- Relies on API 571/API 581 damage mechanism identification and historical experience for material selection
- Needs: corrosion rate prediction tool integrating process conditions (temperature, pressure, composition) with material susceptibility to guide selection and inspection planning

---

## Solution Overview

CorroSim is a cloud-based corrosion engineering platform that:
1. Predicts internal corrosion rates for oil and gas systems using mechanistic electrochemical models (CO2, H2S, organic acid, under-deposit, erosion-corrosion) coupled to multiphase flow conditions
2. Models cathodic protection systems in 3D using boundary element method (BEM) for potential and current distribution on complex geometries (pipelines, offshore structures, reinforced concrete)
3. Simulates galvanic corrosion between dissimilar metals using coupled electrochemical kinetics and electrolyte conductivity
4. Evaluates stress corrosion cracking (SCC), hydrogen embrittlement (HE), and sulfide stress cracking (SSC) susceptibility based on material, environment, and stress state
5. Supports materials selection through corrosion rate databases, environmental cracking domain diagrams, and life-cycle cost comparison for material alternatives

---

## Core Features

### F1: Internal Corrosion Prediction (Oil and Gas)
- CO2 corrosion: mechanistic electrochemical model (de Waard-Milliams, Norsok M-506, and proprietary mechanistic model with film formation kinetics)
- H2S corrosion: sour corrosion rate prediction per NACE MR0175/ISO 15156 environmental domains
- Organic acid (acetic, formic) contribution to corrosion rate
- Top-of-line corrosion (TLC): condensation rate, droplet chemistry, localized corrosion at pipe crown
- Under-deposit corrosion: predict risk based on solids deposition (sand, scale, wax) and galvanic cell formation
- Erosion-corrosion: API RP 14E velocity limits, mechanistic erosion models (E/CRC) coupled with corrosion
- Flow regime effects: stratified, slug, annular, bubble — corrosion rate varies with wall shear stress and water wetting
- Pipeline profile integration: corrosion rate along pipeline length accounting for pressure, temperature, and flow regime changes
- Iron carbonate (FeCO3) and iron sulfide (FeS) scale formation kinetics: protective film growth and breakdown
- Water chemistry: calculate pH, CO2/H2S partial pressures, ionic strength, scaling tendency from gas/water composition

### F2: Cathodic Protection Modeling
- Boundary element method (BEM) solver for 3D potential and current distribution on complex geometries
- Sacrificial anode CP: aluminum, zinc, magnesium anode performance modeling with consumption rate and life prediction
- Impressed current CP: rectifier sizing, anode ground bed design, current distribution optimization
- Pipeline CP: long pipeline models with coating breakdown, holiday density, soil resistivity variation along route
- Offshore structures: jacket, mooring, subsea manifold — anode placement optimization to meet -850 mV CSE (or -800 mV CSE) protection criteria
- Reinforced concrete CP: embedded anode design, concrete resistivity, rebar potential mapping
- Interference effects: foreign pipeline interaction, stray current from DC transit, telluric currents
- Time-dependent CP: anode consumption over design life, coating degradation, seasonal soil resistivity changes
- CP monitoring correlation: compare measured pipe-to-soil potentials with model predictions for calibration

### F3: Galvanic and Localized Corrosion
- Galvanic corrosion: coupled multi-electrode model for dissimilar metal junctions (e.g., carbon steel to stainless steel, copper to aluminum)
- Evans diagram construction: overlay anodic/cathodic polarization curves to predict corrosion potential and current
- Area ratio effects: cathode-to-anode area ratio impact on galvanic corrosion severity
- Crevice corrosion: critical crevice solution chemistry calculation, initiation and propagation prediction
- Pitting corrosion: pitting potential, critical pitting temperature (CPT), pitting resistance equivalent number (PREN) assessment
- Microbiologically influenced corrosion (MIC): risk assessment based on operating conditions, water chemistry, and biofilm formation tendency
- Weld corrosion: preferential weld attack, HAZ sensitization in stainless steels
- Electrolyte conductivity effects: corrosion distribution changes with solution resistivity (throwing power)

### F4: Environmental Cracking Assessment
- Stress corrosion cracking (SCC): susceptibility maps for austenitic SS (chloride SCC), carbon steel (carbonate SCC, HIC), nickel alloys (polythionic acid SCC)
- Sulfide stress cracking (SSC): NACE MR0175/ISO 15156 compliance checking — material hardness limits, environmental severity (H2S partial pressure, pH, temperature)
- Hydrogen-induced cracking (HIC) and hydrogen embrittlement (HE): hydrogen permeation calculation, critical hydrogen concentration for cracking
- Hydrogen diffusion modeling: FEM-based hydrogen transport through steel wall, trap binding effects
- Corrosion fatigue: S-N curve modification for corrosive environment, crack growth rate (da/dN) with environmental enhancement factors
- Fracture mechanics: stress intensity factor calculation, environment-assisted crack growth rate (KISCC, da/dt), remaining life prediction
- High-temperature damage: creep-corrosion interaction, metal dusting, sulfidation, carburization, oxidation rate prediction
- Environmental domain diagrams: map material suitability against temperature, H2S, Cl-, pH conditions

### F5: Materials Selection and Life Cycle
- Material database: 500+ alloys with corrosion resistance data across environments (atmospheric, seawater, CO2/H2S, acids, caustic, high-temperature)
- API 571 damage mechanism identification: input process conditions and material, output applicable damage mechanisms with severity ranking
- Material ranking: compare corrosion rates, environmental cracking susceptibility, and cost for candidate materials in specified service
- Corrosion allowance calculation: required wall thickness addition based on predicted corrosion rate and design life
- Remaining life assessment: combine inspection data (UT thickness, ILI) with predicted corrosion rate for fitness-for-service per API 579-1
- Life-cycle cost analysis: compare material alternatives including initial cost, corrosion allowance, inspection frequency, coating, CP, inhibitor treatment, and replacement cost
- Inhibitor effectiveness modeling: corrosion rate reduction as function of inhibitor type, concentration, and availability
- Coating selection: barrier lifetime estimation, cathodic disbondment risk, compatibility with CP

### F6: Inspection Planning and Reporting
- Risk-based inspection (RBI): API 581-based probability of failure and consequence analysis for inspection prioritization
- Corrosion rate trending: import inspection data (UT, ILI, coupon, ER probe) and fit corrosion models for future projection
- Inspection interval optimization: calculate next inspection date based on corrosion rate, measurement uncertainty, and acceptable risk
- Corrosion circuit management: group equipment by common corrosion environment and damage mechanism
- Anomaly assessment: evaluate ILI-reported metal loss features against ASME B31G / RSTRENG / DNV-RP-F101 criteria
- CP survey data import and compliance assessment against NACE SP0169
- Report generation: corrosion assessment, CP design, materials selection, RBI analysis — PDF export with compliance references
- Dashboard: fleet-wide corrosion risk overview, inspection due dates, CP system status

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | TypeScript, React, Three.js/Deck.gl (3D CP potential maps, pipeline corrosion profiles, GIS integration) |
| Electrochemical Solver | Rust (BEM for CP, coupled electrode kinetics, galvanic models) |
| Corrosion Models | Rust + Python (mechanistic CO2/H2S models, film formation kinetics, environmental cracking) |
| Hydrogen Diffusion | Rust (FEM hydrogen transport with trapping) |
| Water Chemistry | Python (thermodynamic equilibrium, Pitzer model for ionic strength, pH/scaling calculations) |
| Backend | Rust (Axum) + Python (FastAPI for materials database, RBI calculations, reporting) |
| Compute | AWS (CPU instances for BEM solver, auto-scaling for parametric studies) |
| Database | PostgreSQL (material properties, corrosion data, polarization curves, inspection records), S3 (results, reports) |

---

## Monetization

### Free Tier (Student)
- CO2 corrosion rate calculator (de Waard-Milliams, single point)
- Galvanic series lookup and basic galvanic corrosion estimate
- NACE MR0175 quick compliance check (material + environment)
- PREN calculator for stainless steels
- 3 projects

### Pro — $199/month
- Full mechanistic CO2/H2S corrosion prediction along pipeline profile
- Water chemistry calculator (pH, partial pressures, scaling tendency)
- Galvanic corrosion modeling (2D, planar geometries)
- Materials selection with API 571 damage mechanism screening
- Corrosion allowance and remaining life calculation
- Anomaly assessment (B31G, DNV-RP-F101)
- Report generation
- Unlimited projects

### Advanced — $449/user/month
- 3D cathodic protection modeling (BEM solver)
- CP system optimization (anode placement, rectifier sizing)
- Environmental cracking assessment (SCC, SSC, HIC)
- Hydrogen diffusion modeling
- Risk-based inspection (API 581)
- Life-cycle cost analysis
- Inspection data import and trending
- API access
- Team collaboration

### Enterprise — Custom
- On-premise deployment for operator security requirements
- Integration with pipeline SCADA, inspection databases (PCMS, Meridium/APM)
- Fleet-wide corrosion management dashboard
- Custom material qualification data integration
- Dedicated corrosion engineering support
- Digital twin: real-time corrosion monitoring linked to predictive model

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | CorroSim Advantage |
|-----------|-----------|------------|-------------------|
| COMSOL Corrosion | Flexible multiphysics, good for galvanic/CP | $12K+ per seat, general-purpose (not corrosion-native), no oil & gas corrosion models | Purpose-built for corrosion, integrated CP + internal corrosion + materials selection |
| OLI Systems | Best-in-class water chemistry and thermodynamics | $15-30K/year, chemistry-only (no CP design, no structural), complex interface | Integrated platform: chemistry + corrosion + CP + cracking in one tool |
| Honeywell Predict | Strong pipeline internal corrosion, oil & gas industry standard | $20K+/year, pipeline-focused only, no CP modeling, legacy desktop | Broader coverage: pipelines + offshore + refinery + CP, cloud-native |
| BEASY CP | Best BEM solver for cathodic protection modeling | Expensive ($20K+), CP-only (no corrosion prediction), niche | CP modeling + corrosion prediction + materials selection integrated |
| ECE Corrpro / MATCOR | Strong CP design and installation services | Service-based not software, limited simulation capability | Self-service engineering tool for in-house CP design and optimization |

---

## MVP Scope (v1.0)

### In Scope
- CO2 corrosion rate prediction (mechanistic model) for carbon steel pipelines with flow regime coupling
- Water chemistry calculator (pH, CO2/H2S partial pressures, ionic strength)
- Basic galvanic corrosion calculator (two-metal couple, Evans diagram)
- NACE MR0175/ISO 15156 environmental cracking compliance check
- Materials comparison table (corrosion rate, cracking susceptibility, cost) for 50 common alloys
- Corrosion allowance and remaining life calculation
- Metal loss anomaly assessment (B31G modified)
- Web-based pipeline corrosion profile visualization

### Out of Scope (v1.1+)
- 3D cathodic protection BEM modeling
- Hydrogen diffusion and SCC/HIC simulation
- Risk-based inspection (API 581)
- Crevice and pitting corrosion modeling
- High-temperature damage mechanisms
- Inspection data import and trending
- Fleet-wide dashboard and SCADA integration

### MVP Timeline: 14-18 weeks
