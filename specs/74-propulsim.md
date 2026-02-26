# PropulSim — Rocket and Spacecraft Propulsion Design Platform

## Executive Summary

PropulSim is a cloud-based platform for designing rocket engines, electric propulsion systems, and spacecraft propulsion — covering thermochemical analysis, nozzle design, injector simulation, and mission trajectory integration. It replaces NASA CEA (command-line, 1960s code), NPSS ($50K+/seat, restricted), and custom MATLAB/Fortran tools used by space startups.

---

## Problem Statement

**The pain:**
- The commercial space industry (SpaceX, Rocket Lab, Relativity, 200+ startups) needs propulsion design tools, but legacy tools (NPSS, ROCETS) are export-controlled or government-only
- NASA CEA (Chemical Equilibrium with Applications) is the industry standard for thermochemistry but is a 1960s Fortran code with no GUI
- No integrated tool covers the full propulsion design loop: propellant selection → combustion analysis → nozzle design → injector → feed system → mission performance
- Small space startups waste 6-12 months building custom propulsion analysis tools from scratch

**Market size:** The space propulsion market is $8B+ annually, with 200+ active rocket/spacecraft companies. The electric propulsion segment alone is $5B+ growing at 15% CAGR. There are 50,000+ propulsion and aerospace engineers worldwide.

---

## Core Features

### F1: Thermochemical Analysis
- Chemical equilibrium solver (NASA CEA-equivalent): temperature, molecular weight, gamma, c*, Isp for any propellant combination
- Propellant database: 1,000+ species (LOX/RP-1, LOX/LH2, LOX/LCH4, N2O4/UDMH, solid propellants, green propellants)
- Frozen and shifting equilibrium expansion
- Combustion chamber conditions: temperature, pressure, species mole fractions
- Nozzle exit conditions: temperature, velocity, Mach number, area ratio
- Characteristic velocity (c*), thrust coefficient (Cf), specific impulse (Isp)

### F2: Nozzle Design
- Conical, bell (Rao optimum), and aerospike nozzle profiles
- Method of characteristics (MOC) for supersonic nozzle contour
- Boundary layer effects on thrust (displacement thickness correction)
- Nozzle heat transfer: convection (Bartz correlation), radiation
- Thermal-structural analysis of nozzle wall (regenerative cooling channel design)
- Altitude compensation: aerospike, dual-bell, extendable nozzle analysis
- Ablative nozzle erosion prediction

### F3: Injector Design
- Injector element types: shear coaxial, swirl coaxial, pintle, impinging (unlike, like), showerhead
- Spray atomization: droplet size distribution (SMD) from empirical correlations
- Mixing efficiency estimation
- Combustion stability: Crocco model, sensitive time lag, acoustic modes
- Injector pressure drop and flow distribution
- Pattern factor and temperature uniformity prediction

### F4: Feed System
- Pressure-fed and turbopump-fed architectures
- Turbopump design: inducer, impeller, turbine — preliminary sizing
- Tank pressurization: pressurant mass calculation (collapse factor method)
- Line sizing: pressure drop through valves, filters, orifices
- NPSH analysis for pump inlet
- System-level performance: thrust, mixture ratio, Isp vs. throttle setting

### F5: Electric Propulsion
- Hall effect thruster: performance prediction (thrust, Isp, efficiency vs. power)
- Gridded ion engine
- Pulsed plasma thruster
- Electrospray thruster
- Propellant options: xenon, krypton, argon, iodine
- Orbit transfer analysis: low-thrust spiral trajectory, time-to-orbit

### F6: Mission Integration
- Delta-V budget calculation: Tsiolkovsky rocket equation, staging analysis
- Staging optimization: 1, 2, 3 stage to orbit with mass fraction optimization
- Trajectory: gravity turn, vacuum coast, powered descent
- Trade studies: Isp vs. thrust, propellant choice, engine cycle selection

---

## Monetization

### Free Tier (Student)
- Thermochemical analysis (CEA-equivalent)
- Basic nozzle design (conical, bell)
- Delta-V calculator
- 3 projects

### Pro — $149/month
- Full nozzle design (MOC, cooling)
- Injector design
- Feed system analysis
- Electric propulsion
- Staging optimization

### Enterprise — Custom
- ITAR-compliant hosting
- Custom propellant characterization
- Integration with trajectory tools (GMAT, STK)
- On-premise deployment
- Export-controlled content management

---

## MVP Scope (v1.0)

### In Scope
- Chemical equilibrium solver (NASA CEA-equivalent functionality)
- Propellant database (50 common combinations)
- Isp, c*, Cf calculation
- Bell nozzle profile generation (Rao method)
- Delta-V / staging calculator
- Web-based UI with plots

### Out of Scope (v1.1+)
- Injector design, feed system, cooling, electric propulsion, trajectory

### MVP Timeline: 12-16 weeks
