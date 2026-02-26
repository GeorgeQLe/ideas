# HeatPipe — Heat Pipe and Thermal Device Design Platform

## Executive Summary

HeatPipe is a cloud-based engineering platform for designing, analyzing, and optimizing heat pipes, vapor chambers, thermosiphons, and other two-phase thermal management devices. It replaces custom in-house codes, Thermacore proprietary design tools, ACT thermal calculators, and expensive COMSOL thermal modules ($8K+ base + $4K+ per module) with browser-based two-phase thermal device simulation, working fluid selection, and wick structure optimization.

---

## Problem Statement

**The pain:**
- No dedicated commercial tool exists for heat pipe design — engineers cobble together COMSOL ($8K+ base + modules), custom MATLAB scripts, and manufacturer lookup tables to design two-phase thermal devices
- Heat pipe performance is governed by multiple operating limits (capillary, boiling, entrainment, viscous, sonic) that interact in complex ways; missing any one limit leads to catastrophic device failure (dryout)
- Vapor chamber and heat pipe design for electronics cooling (5G base stations, GPUs, EV power electronics) is exploding in demand, but design expertise is concentrated in a handful of specialist manufacturers
- Wick structure selection (sintered powder, grooved, screen mesh, composite) determines capillary performance and requires coupled capillary-thermal-hydraulic analysis that no off-the-shelf tool provides
- Working fluid selection depends on operating temperature range, compatibility with case material, and thermodynamic properties — currently done by consulting scattered reference tables and experience

**Current workarounds:**
- Using 1D thermal resistance network models in spreadsheets with empirical correlations (Chi's 1976 textbook formulas) that miss 3D effects and transient behavior
- Relying on heat pipe manufacturer proprietary tools (Thermacore, ACT, Aavid) that are locked to their products and not available for general design
- Running general-purpose multiphysics (COMSOL, ANSYS Fluent with UDF) for two-phase simulation, requiring weeks of setup and $30K+ in licenses
- Building and testing physical prototypes iteratively, at $5K-$50K per prototype cycle with 4-8 week lead times

**Market size:** The global heat pipe market is approximately $1.2 billion (2024), growing at 8-10% CAGR driven by electronics miniaturization, EV thermal management, and data center cooling. The broader thermal management market exceeds $18 billion. There are an estimated 50,000+ thermal engineers worldwide who design or specify two-phase thermal devices, with rapid growth in automotive and consumer electronics sectors.

---

## Target Users

### Primary Personas

**1. David — Thermal Engineer at a Consumer Electronics Company**
- Designs heat pipe and vapor chamber assemblies for smartphones, laptops, and gaming consoles
- Uses spreadsheet-based 1D models and manufacturer datasheets to select heat pipes; validates with physical prototyping
- Needs: rapid performance prediction for heat pipes and vapor chambers under realistic boundary conditions, including orientation effects, pulsed heat loads, and form factor constraints

**2. Dr. Nakamura — Aerospace Thermal Systems Engineer**
- Designs loop heat pipes and constant conductance heat pipes for satellite thermal control
- Uses custom Fortran code developed in-house over 20 years, plus COMSOL for detailed analysis
- Needs: loop heat pipe modeling with startup transient prediction, gravity/microgravity simulation, and working fluid selection for extreme temperature ranges (-40C to +200C)

**3. Priya — EV Battery Thermal Management Engineer**
- Evaluates heat pipe and vapor chamber solutions for battery pack thermal management as alternatives to liquid cooling
- Needs to compare two-phase thermal devices against cold plates for cost, weight, and thermal uniformity
- Needs: integration of heat pipe thermal models with battery pack geometry, transient drive cycle analysis, and passive vs. active cooling trade studies

**4. Marcus — Data Center Cooling Engineer**
- Investigates thermosiphon and pulsating heat pipe solutions for server-level and rack-level cooling
- Currently uses CFD (ANSYS Fluent) but modeling two-phase flow requires expert-level UDF customization
- Needs: thermosiphon performance prediction, condenser sizing, and system-level integration with air-side cooling

---

## Solution Overview

HeatPipe is a cloud-based two-phase thermal device platform that:
1. Designs and analyzes heat pipes (cylindrical, flat, variable conductance), vapor chambers, loop heat pipes, thermosiphons, and pulsating heat pipes with physics-based models
2. Computes all operating limits (capillary, boiling, entrainment, viscous, sonic, condenser) and generates performance envelopes as a function of temperature, orientation, and heat load
3. Provides wick structure design tools for sintered powder, axial grooves, screen mesh, and composite wicks with coupled capillary pressure, permeability, and effective thermal conductivity models
4. Includes a comprehensive working fluid database with thermophysical properties, material compatibility matrices, and automated fluid selection based on operating temperature range
5. Optimizes device geometry (diameter, length, wick thickness, vapor core size, condenser area) using AI-driven parametric sweeps and multi-objective optimization

---

## Core Features

### F1: Heat Pipe Analysis Engine
- 1D and 2D steady-state thermal-hydraulic models for conventional heat pipes (cylindrical, flat plate)
- Operating limit calculations: capillary limit (wick-dependent), boiling limit (nucleation onset), entrainment limit (Weber number), viscous limit (vapor pressure drop), sonic limit (choked vapor flow)
- Performance envelope generation: maximum heat transport (Qmax) vs. operating temperature for given geometry and orientation
- Orientation effects: gravity-assisted, gravity-opposed, and horizontal operation with tilt angle parameterization
- Transient simulation: startup from frozen state, pulsed heat load response, thermal mass effects
- Variable conductance heat pipe (VCHP): non-condensable gas (NCG) reservoir modeling for temperature control
- Effective thermal resistance decomposition: evaporator wall, evaporator wick, vapor core, condenser wick, condenser wall
- Multi-heat-source and multi-condenser configurations for complex thermal architectures

### F2: Vapor Chamber Design
- 3D thermal-hydraulic simulation of vapor chambers with thin form factors (0.3mm-3mm total thickness)
- Wick-vapor-wick sandwich structure modeling with non-uniform wick thickness distributions
- Hotspot spreading analysis: thermal spreading resistance as a function of heat source size and location
- Support column (pillar) design: structural integrity under clamping load with thermal performance impact
- Ultra-thin vapor chamber (UTVC) modeling for mobile devices with micro-wick structures
- Multiple heat source handling: independent and simultaneous loading scenarios
- Thermal spreading comparison tool: vapor chamber vs. copper spreader vs. graphite sheet for given geometry
- Manufacturing constraint integration: minimum wall thickness, wick sintering feasibility, flatness tolerances

### F3: Wick Structure Design
- Sintered powder wick: porosity, particle size, pore radius → capillary pressure, permeability (Kozeny-Carman), effective thermal conductivity (EMT model)
- Axial groove wick: groove geometry (rectangular, triangular, trapezoidal, re-entrant) → capillary pressure, liquid flow area, fin efficiency
- Screen mesh wick: mesh number, wire diameter → capillary pressure, permeability, number of layers optimization
- Composite/hybrid wicks: bi-porous structures combining fine pore evaporator regions with coarse pore liquid arteries
- Wick property calculator: input manufacturing parameters → output thermal-hydraulic properties
- Capillary pumping curve visualization: capillary pressure minus viscous pressure drop along heat pipe length
- Evaporator thin-film analysis: meniscus profile in wick pores, thin-film evaporation heat transfer coefficient
- Wick selection advisor: recommend wick type and parameters for given thermal duty and form factor

### F4: Working Fluid Database and Selection
- Comprehensive fluid property database: water, methanol, ethanol, acetone, ammonia, cesium, sodium, potassium, lithium, R134a, R1234yf, and 50+ additional fluids
- Temperature-dependent thermophysical properties: density, viscosity, surface tension, latent heat, thermal conductivity, vapor pressure (liquid and vapor phases)
- Merit number (liquid transport factor) calculation and comparison across fluids for any temperature range
- Material compatibility matrix: fluid-case-wick material combinations with long-term reliability ratings based on published life test data
- Non-condensable gas (NCG) generation prediction for fluid-material combinations at operating temperature
- Automatic fluid recommendation: input operating temperature range and case material → ranked list of compatible fluids with performance metrics
- Custom fluid input: user-defined property tables for novel or proprietary working fluids
- Cryogenic and high-temperature fluid support: nitrogen, neon, helium (cryogenic); cesium, sodium, lithium (high-temperature alkali metals)

### F5: Loop Heat Pipe and Thermosiphon Analysis
- Loop heat pipe (LHP): evaporator (compensation chamber + primary wick), vapor line, condenser, liquid return line — full steady-state and transient modeling
- LHP startup simulation: initial superheat, temperature overshoot, wick flooding/depriming scenarios
- Capillary pump loop (CPL) analysis with active reservoir control
- Two-phase thermosiphon: gravity-driven, no wick — pool boiling evaporator, filmwise condensation condenser
- Thermosiphon performance: fill ratio optimization, geyser boiling prediction, flooding limit
- Pulsating (oscillating) heat pipe: simplified 1D slug-plug model for thermal performance estimation
- Loop configuration optimization: line diameters, condenser length, elevation effects
- System-level integration: thermal resistance from heat source to ultimate heat sink including interface resistances

### F6: Optimization and Reporting
- Parametric sweep: vary any geometric or operating parameter and plot performance response surfaces
- Multi-objective optimization: maximize Qmax while minimizing weight, volume, or thermal resistance using genetic algorithms
- Design of experiments (DOE): factorial and Latin hypercube sampling for sensitivity analysis
- Thermal resistance budget: breakdown from junction to ambient with contribution percentages
- Comparison mode: evaluate multiple device configurations (heat pipe vs. vapor chamber vs. cold plate) side-by-side
- Automated design report: device specifications, performance curves, operating limits map, material/fluid selection rationale
- Export thermal model: reduced-order thermal resistance model for system-level simulation (SPICE-compatible for electronics, Modelica for systems)
- Manufacturing specification output: detailed drawings with wick parameters, fill charge, pinch-off, and assembly notes

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | TypeScript, React, Three.js/WebGPU (3D device visualization, performance plots) |
| Thermal-Hydraulic Solver | Rust (1D/2D two-phase thermal-hydraulic models, operating limit calculations) |
| Vapor Chamber 3D Solver | Rust (thin-film vapor core + wick FEM/FVM thermal solver) → GPU-accelerated |
| Working Fluid Engine | Rust (thermophysical property interpolation, merit number computation) |
| Optimization | Python (scipy, pymoo for multi-objective GA, DOE) |
| Backend | Rust (Axum) + Python (FastAPI for optimization, reporting, ML-based recommendations) |
| AI/ML | Python (PyTorch — surrogate models for rapid parametric exploration, fluid recommendation) |
| Database | PostgreSQL (fluid properties, material compatibility, wick correlations, device templates), S3 (simulation results, reports) |
| Hosting | AWS (CPU-optimized for thermal solvers, GPU for 3D vapor chamber models) |

---

## Monetization

### Free Tier (Student)
- 1D cylindrical heat pipe analysis (single wick type, single fluid)
- Operating limit calculations for simple geometries
- Working fluid property lookup (10 common fluids)
- 3 projects

### Pro — $99/month
- Full heat pipe analysis (all geometries, orientations, transient)
- Vapor chamber 2D spreading analysis
- All wick types and composite wicks
- Complete working fluid database (50+ fluids) with material compatibility
- Performance envelope generation
- Report generation
- 20 projects

### Advanced — $249/user/month
- Everything in Pro
- 3D vapor chamber simulation
- Loop heat pipe and thermosiphon modeling
- Multi-objective optimization
- Parametric sweeps and DOE
- Reduced-order model export (SPICE, Modelica)
- API access
- Unlimited projects

### Enterprise — Custom
- On-premise deployment
- Custom wick characterization and model calibration
- Integration with system-level thermal tools (Icepak, FloTHERM, 6SigmaET)
- Multi-user design collaboration with version control
- Training, certification, and dedicated support

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | HeatPipe Advantage |
|-----------|-----------|------------|-------------------|
| COMSOL Thermal Module | Flexible multiphysics, detailed PDE-level models | $12K+/seat (base + modules), requires expert setup, no heat-pipe-specific workflow | Heat-pipe-native workflow, built-in wick models, 10x faster setup |
| ACT Thermal Tools | Heat pipe manufacturer expertise, validated models | Proprietary, tied to ACT products, not available for general design | Vendor-neutral, any geometry and manufacturer |
| Thermacore Design Tools | Deep vapor chamber expertise | Proprietary to Thermacore, limited access | Open platform, all device types, any manufacturer |
| ANSYS Fluent (VOF two-phase) | Most general two-phase CFD capability | $30K+/seat, CFD-level complexity, weeks of setup per model | 1000x faster for heat pipe design, purpose-built |
| Spreadsheet/Chi Models | Free, simple, widely understood | 1D only, no transient, no optimization, miss 3D effects | Physics-accurate 2D/3D, all operating limits, optimization |

---

## MVP Scope (v1.0)

### In Scope
- 1D steady-state cylindrical heat pipe analysis with capillary, boiling, entrainment, viscous, and sonic limits
- Three wick types: sintered powder, axial grooves, screen mesh
- Working fluid database with 15 common fluids (water, methanol, ethanol, acetone, ammonia, and select others) and property curves
- Performance envelope plot (Qmax vs. temperature) with orientation effects
- Thermal resistance breakdown (evaporator, vapor, condenser)
- Material compatibility lookup for common case-wick-fluid combinations
- Basic parametric sweep (vary one parameter at a time)

### Out of Scope (v1.1+)
- 3D vapor chamber simulation
- Loop heat pipe and thermosiphon modeling
- Transient and startup simulation
- Multi-objective optimization with genetic algorithms
- Pulsating heat pipe models
- Reduced-order model export
- VCHP gas reservoir modeling

### MVP Timeline: 10-14 weeks
