# ExplodeSafe — Energetic Material Simulation Platform

## Executive Summary

ExplodeSafe is a cloud-based platform for predicting the thermochemical performance, detonation properties, and sensitivity characteristics of explosives, propellants, and pyrotechnics. It replaces CHEETAH (LLNL, restricted access), Explo5 ($3K+/seat, limited availability), and custom defense codes with a modern, accessible tool for energetic material formulation design, detonation physics, and safety assessment — serving both defense R&D and commercial mining/demolition applications.

---

## Problem Statement

**The pain:**
- Energetic material simulation codes are restricted, expensive, or outdated: CHEETAH (LLNL) requires US government affiliation, Explo5 (OZM Research) costs $3,000-$8,000/seat with limited support, TIGER and JAGUAR are legacy FORTRAN codes from the 1960s-80s with no modern interfaces
- Formulating new explosives or propellants requires predicting detonation velocity, pressure, energy, and gas products before synthesis — but accessible computational tools are scarce outside national laboratories
- Sensitivity prediction (impact, friction, electrostatic discharge) is critical for safety but relies on empirical correlations scattered across decades of literature with no unified computational framework
- Commercial mining explosives engineers need to optimize blast formulations (ANFO, emulsions, watergels) for specific rock types but lack affordable thermochemical design tools
- Propellant formulators for defense, space, and automotive (airbags) applications need to predict burn rate, gas generation, and combustion products but use fragmented spreadsheets and handbook data

**Current workarounds:**
- Using NASA CEA (free, but designed for rocket propellants, not detonation) as an approximation for explosive thermochemistry
- Running CHEETAH through national lab collaborators with weeks of turnaround time
- Building custom thermochemical equilibrium codes in MATLAB or Python that lack validated equations of state for detonation products
- Relying on decades-old handbook values (LASL Explosive Properties Data) rather than computing performance for novel formulations

**Market size:** The global explosives market exceeds $30 billion annually (mining $20B, defense $8B, construction/demolition $2B). The propellant market adds $15B (defense, space, automotive). Energetic materials R&D spending is approximately $5 billion globally. There are 30,000+ energetic materials scientists, engineers, and technicians worldwide across defense labs, mining companies, and academic institutions.

---

## Target Users

### Primary Personas

**1. Dr. Vasquez — Energetic Materials Scientist at a Defense Lab**
- Designs novel insensitive munition (IM) formulations to replace TNT-based compositions
- Has CHEETAH access but finds the command-line interface and input file format arcane; spends hours debugging input decks
- Needs: modern GUI for thermochemical prediction with validated EOS, plus sensitivity screening for new molecular candidates

**2. Jan — Mining Explosives Engineer at a Blasting Company**
- Formulates ANFO, emulsion, and watergel explosives for open-pit and underground mining
- Uses supplier-provided performance data and empirical rules of thumb; has no independent simulation capability
- Needs: affordable tool to optimize explosive formulations for specific rock densities and blast geometries

**3. Dr. Nakamura — Propellant Chemist at an Aerospace Company**
- Develops solid and liquid propellants for rocket motors and gas generators
- Uses NASA CEA for combustion products but needs detonation hazard assessment (is the propellant formulation shock-sensitive?) which CEA cannot provide
- Needs: integrated platform covering both combustion (propellant performance) and detonation (safety assessment) thermochemistry

---

## Solution Overview

ExplodeSafe is a cloud-based energetic materials simulation platform that:
1. Computes Chapman-Jouguet (CJ) detonation state (velocity, pressure, temperature) for any explosive composition using validated equations of state (BKW, JCZ3, BKWS)
2. Predicts detonation product composition and energy release via Gibbs free energy minimization at extreme pressures and temperatures
3. Models expansion work (Gurney energy, cylinder test equivalent) using JWL and other product isentrope equations of state
4. Estimates sensitivity properties (impact, friction, ESD) from molecular structure using validated QSPR (quantitative structure-property relationship) models
5. Supports propellant combustion analysis including burn rate, specific impulse, and gas generation for solid rocket motors, gun propellants, and gas generators

---

## Core Features

### F1: Thermochemical Equilibrium Engine
- Gibbs free energy minimization at specified temperature/pressure or volume/energy (constant volume explosion, CJ state)
- Species database: 5,000+ gaseous, liquid, and solid product species with JANAF/Gurvich thermodynamic data
- Reactant library: 500+ common explosives, oxidizers, fuels, binders, plasticizers, metals (RDX, HMX, CL-20, TNT, TATB, PETN, AN, AP, Al, HTPB, GAP, etc.)
- Custom molecular input: empirical formula, heat of formation, density — compute performance for novel compounds
- Condensed-phase product handling: solid carbon (graphite, diamond), metal oxides (Al2O3), water (liquid/solid) at high pressure
- Multi-phase equilibrium: gas-solid-liquid coexistence at detonation conditions
- Hugoniot calculation: shock adiabat for unreacted and reacted explosive
- Detonation product speciation: mole fractions of CO, CO2, H2O, N2, CH4, NH3, HCN, solid C, etc.

### F2: Detonation Performance Prediction
- Chapman-Jouguet (CJ) detonation velocity, pressure, temperature, and density
- Equations of state for detonation products: BKW (Becker-Kistiakowsky-Wilson), JCZ3, BKWS (Hobbs-Baer), JCZS
- BKW calibrations: BKWC (C parameterization), BKWR (R parameterization), BKWS — user-selectable
- Detonation energy: total chemical energy release, available work energy
- Gurney energy and velocity: predicting metal acceleration capability for warhead design
- Cylinder test simulation: wall velocity vs. expansion ratio, comparison with experimental V(R) curves
- JWL (Jones-Wilkins-Lee) isentrope fitting: automatic calibration of JWL parameters from CJ state and isentrope
- Failure diameter estimation from reaction zone length correlations
- Non-ideal detonation: diameter effect on detonation velocity (rate stick), detonation velocity vs. density
- Oxygen balance calculation and its correlation with performance metrics

### F3: Formulation Design
- Multi-component formulation builder: mix explosives, oxidizers, binders, plasticizers, metals by mass fraction
- Automatic property computation: density (by volume additivity or crystal packing), oxygen balance, nitrogen content, heat of formation
- Performance maps: sweep composition space and visualize detonation velocity, pressure, energy as function of component ratios
- Insensitive munitions (IM) formulation: trade performance vs. sensitivity, identify Pareto-optimal compositions
- ANFO and emulsion explosive design: prill porosity, fuel oil ratio, sensitizer (glass microsphere, chemical gassing) effects on VOD
- Aluminized explosive optimization: Al particle size and content effects on blast and thermobaric performance
- Melt-cast, pressed, and PBX (polymer-bonded) formulation types with realistic density estimation
- Propellant formulation: AP/HTPB/Al composite, double-base (NC/NG), and advanced (CL-20/GAP) propellant compositions
- Compatibility assessment: known incompatible material pair warnings

### F4: Sensitivity and Safety Assessment
- Impact sensitivity prediction: QSPR models from molecular descriptors (h50 drop height estimation)
- Friction sensitivity: BAM friction test prediction from structural features
- Electrostatic discharge (ESD) sensitivity estimation
- Thermal stability: DSC onset temperature prediction, Kissinger activation energy
- Oxygen balance correlation with sensitivity
- Crystal density and packing efficiency from molecular volume estimation
- Detonation reaction zone width and its relation to shock sensitivity
- Card gap test (shock sensitivity) prediction
- Small-scale safety test (SSST) screening equivalents: compute metrics that correlate with standard sensitivity tests
- Safety data sheet (SDS) relevant property compilation for new formulations

### F5: Propellant Performance
- Internal ballistics: burn rate (r = a * P^n) from composition, flame temperature, and gas evolution
- Specific impulse (Isp) for rocket propellants: equilibrium and frozen expansion
- Chamber pressure and thrust prediction for solid rocket motor grain geometries
- Gun propellant: gas volume, force constant (impetus), covolume, adiabatic flame temperature
- Gas generator performance: gas output (moles/gram), temperature, molecular weight for airbag and fire suppression applications
- Propellant aging: predicted service life from accelerated aging kinetics (Arrhenius)
- Minimum signature propellant assessment: visible and IR exhaust plume spectral prediction
- Hazard classification support: UN transport class estimation from computed properties

### F6: Data and Reporting
- Validated against experimental database: 200+ compounds with published CJ detonation velocities and pressures
- Comparison plots: computed vs. experimental performance for selected reference explosives
- Batch computation: screen hundreds of candidate formulations in a parameter sweep
- Export: CSV, JSON, PDF technical reports
- Hydrocode input file generation: JWL parameters formatted for LS-DYNA, AUTODYN, CTH input
- Regulatory documentation: inputs for UN classification tests, IM assessment
- Literature reference database: link computed properties to published experimental sources
- Audit trail: full record of inputs, EOS selection, and results for quality assurance

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | TypeScript, React, Plotly.js (phase diagrams, Hugoniots, performance maps), D3.js (formulation ternary diagrams) |
| Thermo Engine | Rust (Gibbs minimization, CJ state solver, Hugoniot computation, JWL fitting) |
| EOS Library | Rust (BKW, JCZ3, ideal gas, Cowan, JWL product EOS implementations) |
| QSPR/ML | Python (scikit-learn, RDKit for molecular descriptors, XGBoost for sensitivity prediction) |
| Backend | Python (FastAPI) for species database, formulation management; Rust (Axum) for compute orchestration |
| AI/ML | Python (PyTorch) for neural network potentials, graph neural networks for sensitivity prediction from molecular structure |
| Database | PostgreSQL (species thermodynamic data, formulation library, experimental validation data), S3 (batch computation results) |
| Hosting | AWS (c7g for thermochemical computation, Lambda for batch screening, CloudFront for frontend) |

---

## Monetization

### Free Tier (Academic)
- CJ detonation state for single-component explosives (50 common compounds)
- Ideal gas product EOS only
- Basic sensitivity lookup (not prediction for novel compounds)
- 5 formulations saved

### Pro — $149/month
- Full BKW/JCZ3 product EOS
- Multi-component formulation builder
- JWL isentrope fitting and hydrocode export
- Gurney energy and cylinder test simulation
- Sensitivity prediction (QSPR) for novel compounds
- Propellant performance (Isp, burn rate)
- Unlimited formulations and batch computation

### Advanced — $349/user/month
- Non-ideal detonation (diameter effects)
- Advanced sensitivity models (GNN-based)
- Propellant internal ballistics
- API access for integration with hydrocode workflows
- Team collaboration (5 seats)
- Priority computation

### Enterprise — Custom
- On-premise deployment for defense labs (air-gapped networks)
- Custom EOS calibration from proprietary experimental data
- Integration with hydrocode suites (LS-DYNA, AUTODYN, CTH)
- Export control compliance (ITAR/EAR management)
- Dedicated support with energetics domain experts
- Custom species and reaction database management

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | ExplodeSafe Advantage |
|-----------|-----------|------------|----------------------|
| CHEETAH (LLNL) | Gold standard, comprehensive EOS, validated | US government only, command-line, no GUI, restricted access | Accessible globally, modern UI, cloud-based |
| Explo5 (OZM) | Good CJ computation, BKW/BKWS EOS | $3-8K/seat, Windows-only, limited sensitivity prediction | Integrated sensitivity + performance, lower cost |
| TIGER/JAGUAR | Historical pedigree, validated for certain compositions | 1960s-80s FORTRAN, essentially unmaintained | Modern codebase, actively developed |
| NASA CEA | Free, excellent for combustion | No detonation physics, no condensed-phase EOS | Full detonation + combustion coverage |
| Custom codes | Tailored to lab-specific needs | Years to develop, unvalidated, not transferable | Ready-to-use, validated, collaborative |

---

## MVP Scope (v1.0)

### In Scope
- CJ detonation state solver (velocity, pressure, temperature) with ideal gas + BKW product EOS
- Species database with 200 common energetic materials and their thermodynamic properties
- Multi-component formulation builder (up to 10 components)
- Oxygen balance and heat of formation computation
- JWL isentrope fitting from CJ state
- Basic sensitivity lookup table for known compounds
- Web-based UI with performance plots (VOD vs. density, pressure vs. composition)

### Out of Scope (v1.1+)
- QSPR/GNN sensitivity prediction for novel molecules
- Propellant combustion and internal ballistics
- Non-ideal detonation (diameter effects)
- Cylinder test simulation
- Hydrocode input file export
- Batch formulation screening

### MVP Timeline: 12-16 weeks
