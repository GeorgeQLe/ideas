# CastSim — Metal Casting and Foundry Process Simulation Platform

## Executive Summary

CastSim is a cloud-based platform for simulating metal casting processes — covering mold filling, solidification, shrinkage porosity prediction, thermal stress, and microstructure evolution for sand casting, die casting, investment casting, and continuous casting. It replaces MAGMASOFT ($40K+/seat), ProCAST ($30K+/seat), and FLOW-3D CAST ($25K+/seat) with browser-based simulation accessible to foundries of all sizes.

---

## Problem Statement

**The pain:**
- MAGMASOFT costs $40,000-$80,000/seat/year and requires dedicated simulation engineers; ProCAST (ESI) costs $30,000-$50,000/seat; FLOW-3D CAST costs $25,000-$45,000/seat — pricing that excludes 90% of foundries worldwide
- Casting scrap rates average 5-15% industry-wide, costing the global foundry industry $15B+/year in wasted metal, energy, and labor — most defects (shrinkage porosity, misruns, hot tears) are predictable with simulation
- Foundry engineers rely on decades of trial-and-error experience for gating/riser design, but retiring workforce is taking this knowledge with them — and new geometries from topology optimization and additive patterns break old rules of thumb
- Die casting tooling costs $50K-$500K per die, and discovering porosity problems after tooling is cut leads to costly die modifications or complete redesigns
- Integration between filling simulation, solidification analysis, stress prediction, and microstructure modeling requires multiple tools with manual data transfer

**Current workarounds:**
- Using Chvorinov's rule and empirical modulus methods for riser sizing — works for simple geometries but fails for complex castings
- Running filling simulation in FLOW-3D and solidification in ProCAST separately with manual coupling
- Physical trial pours with sectioning and X-ray inspection to find porosity ($5,000-$20,000 per trial iteration)
- Using simplified solidification simulation (SOLIDCast at $5K) that lacks filling analysis and thermal stress

**Market size:** The global foundry industry produces $250B+/year in castings. The casting simulation software market is approximately $500 million (2024), growing at 8% CAGR. There are 150,000+ foundry engineers and casting designers worldwide, with 50,000+ foundries globally (80% are SMEs with no simulation capability).

---

## Target Users

### Primary Personas

**1. Roberto — Foundry Process Engineer at a Jobbing Sand Casting Foundry**
- Designs gating systems, risers, and pouring parameters for 50+ different part numbers per year in iron and steel
- Relies on experience-based rules (Heuvers circles, modulus method) and loses $200K+/year to scrap from shrinkage porosity and misruns
- Needs: affordable, fast filling and solidification simulation to validate gating/riser design before first pour

**2. Dr. Nakamura — Die Casting Engineer at an Automotive Tier 1**
- Designs high-pressure die casting processes for aluminum structural components (shock towers, subframes)
- A single die costs $200K+ and takes 16 weeks to build; porosity in critical areas causes part rejection and die rework
- Needs: coupled filling-solidification-stress simulation to predict porosity locations, optimize gate/overflow placement, and validate die thermal management

**3. Priya — Investment Casting Metallurgist at an Aerospace Supplier**
- Casts nickel superalloy turbine blades (single crystal, directional solidification) for jet engines
- Grain structure, microshrinkage, and freckling defects are mission-critical; current simulation with ProCAST is expensive and slow
- Needs: solidification simulation with microstructure prediction (grain structure, dendrite arm spacing, segregation) for aerospace qualification

---

## Solution Overview

CastSim is a cloud-based casting simulation platform that:
1. Simulates mold filling with free-surface tracking (Volume of Fluid) to predict misruns, cold shuts, air entrapment, and oxide inclusion paths
2. Performs solidification analysis with Niyama-criterion-based shrinkage porosity prediction, feeding distance assessment, and riser optimization
3. Couples thermal-stress analysis to predict hot tearing susceptibility, residual stress, and casting distortion after solidification and shakeout
4. Predicts microstructure evolution: grain size, dendrite arm spacing (DAS), phase fractions (graphite morphology in cast iron, eutectic structure in Al-Si), and mechanical property mapping
5. Optimizes gating system design, riser placement, and process parameters (pouring temperature, speed, die temperature) using AI-driven parametric studies

---

## Core Features

### F1: Mold Filling Simulation
- Volume of Fluid (VOF) free-surface tracking for accurate fill front visualization
- Turbulence modeling: RANS (k-epsilon) for gating system flow characterization
- Air/gas entrapment prediction: track gas pockets, predict blow hole locations
- Oxide film tracking: cumulative free surface exposure metric for oxide inclusion prediction
- Die casting specific: plunger motion (slow shot, fast shot, intensification), shot sleeve wave dynamics, air vent sizing
- Investment casting specific: shell permeability, vacuum/centrifugal filling, wax pattern filling for shell design
- Sand casting specific: mold erosion risk, filter placement, pouring basin and sprue design
- Temperature-dependent viscosity with solidification fraction coupling (mushy zone flow resistance)
- Non-Newtonian behavior for semi-solid casting (thixocasting, rheocasting)

### F2: Solidification and Porosity Prediction
- Enthalpy-based solidification solver with temperature-dependent thermophysical properties
- Niyama criterion mapping for shrinkage porosity risk assessment (threshold calibration per alloy)
- Feeding flow analysis: identify isolated liquid pools, predict pipe shrinkage and centerline porosity
- Riser sizing: automatic modulus calculation, feeding distance evaluation per SFSA guidelines
- Solidification sequence visualization: iso-solidification time contours, last-to-solidify regions
- Hot spot identification with automatic riser placement suggestion
- Chills and insulation: model effect of external chills, exothermic sleeves, insulating materials on solidification
- Continuous casting: strand solidification, shell thickness, breakout risk prediction
- Eutectic and peritectic reaction modeling for complex alloy systems

### F3: Thermal Stress and Distortion
- Thermo-elastic-plastic-creep FEA coupled to thermal solution from solidification
- Hot tearing prediction: Clyne-Davies criterion, RDG (Rappaz-Drezet-Gremaud) criterion for hot crack susceptibility
- Residual stress prediction after cooling to room temperature
- Casting distortion: spring-back after mold removal, core restraint effects
- Die casting: die thermal cycling stress, die life prediction (thermal fatigue), ejection force estimation
- Pattern allowance: predict shrinkage-compensated pattern dimensions from distortion results
- Contact modeling: casting-mold interaction with gap-dependent heat transfer coefficient
- Shake-out and heat treatment simulation: stress evolution during knockout and quenching

### F4: Microstructure Prediction
- Dendrite arm spacing (DAS): primary and secondary arm spacing from local cooling rate (empirical and CAFE models)
- Cast iron: graphite morphology prediction (flake, compacted, nodular) based on composition and cooling rate; ferrite/pearlite ratio
- Aluminum alloys: eutectic Si morphology, intermetallic phase prediction (Fe-bearing phases), grain refiner effectiveness
- Nickel superalloys: columnar-to-equiaxed transition (CET), single crystal grain selection, freckling and misoriented grain prediction
- Steel: as-cast grain size, microsegregation (Scheil model), carbide formation
- Mechanical property mapping: hardness, yield strength, ultimate tensile strength, elongation derived from microstructure
- Porosity-aware property degradation: reduce predicted strength based on local porosity fraction
- Heat treatment response: age hardening, solution treatment effect on microstructure and properties

### F5: Gating and Riser Optimization
- Parametric gating system design: sprue, runner, ingate dimensions with automatic flow balancing
- Riser optimization: genetic algorithm to minimize riser volume while eliminating shrinkage porosity
- Gate location optimization for die casting: minimize porosity in critical regions, optimize flow pattern
- Runner balancing for multi-cavity die casting
- Overflow and vent placement optimization for die casting
- Cooling channel optimization: conformal cooling layout, baffle and bubbler placement for die thermal management
- Pouring parameter optimization: temperature, speed, tilt angle for gravity casting
- Design of experiments (DOE) with response surface for multi-parameter optimization
- AI-assisted design suggestion: recommend gating layout from part geometry using trained neural network

### F6: Material Database and Process Library
- Alloy database: 500+ alloys with temperature-dependent thermophysical properties (thermal conductivity, specific heat, density, viscosity, latent heat, solidus/liquidus)
- Alloy families: gray/ductile/compacted graphite iron, carbon/low-alloy/stainless steel, Al-Si/Al-Cu/Al-Mg, copper alloys, nickel superalloys, titanium, magnesium
- Mold materials: silica sand, zircon sand, chromite sand, ceramic shell (investment), H13 die steel, copper die inserts
- Thermodynamic data: liquidus/solidus calculation from composition (Scheil, lever rule), fraction solid curves
- Interface heat transfer coefficient database: casting-mold HTC vs. gap, pressure, and surface condition
- Process templates: pre-configured setups for common casting processes (green sand, no-bake, HPDC, LPDC, investment, centrifugal)
- Import/export: STEP/STL for geometry, results export for downstream FEA (ANSYS, Abaqus format)

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | TypeScript, React, Three.js (3D fill animation, porosity contours, microstructure maps) |
| CFD/Filling Solver | Rust (VOF free-surface, FVM Navier-Stokes with solidification coupling) → GPU-accelerated |
| Thermal Solver | Rust (enthalpy method FVM/FEM, transient heat transfer with latent heat) |
| Stress Solver | Rust (thermo-elastic-plastic FEM, contact mechanics) |
| Microstructure | Rust + Python (CAFE grain model, empirical DAS correlations, Scheil microsegregation) |
| Optimization | Python (genetic algorithm, DOE, response surface, ML-based gating suggestion) |
| Backend | Rust (Axum) + Python (FastAPI for material database and optimization services) |
| Compute | AWS (GPU instances for filling CFD, CPU auto-scaling for thermal/stress, spot instances for optimization) |
| Database | PostgreSQL (alloy properties, process templates, project metadata), S3 (meshes, results, animations) |

---

## Monetization

### Free Tier (Student)
- 2D solidification simulation only (cross-section analysis)
- Niyama criterion porosity prediction
- 5 alloys (A356, gray iron, 1020 steel, C360 brass, AZ91 Mg)
- 3 projects, 100K element limit

### Pro — $249/month
- 3D filling + solidification simulation
- Porosity prediction with Niyama and feeding analysis
- Thermal stress and distortion
- Full alloy database (500+ alloys)
- Gating/riser design tools
- Unlimited projects
- 500 cloud compute hours/month

### Advanced — $499/user/month
- All Pro features
- Microstructure prediction (DAS, grain structure, phase fractions)
- Die casting thermal cycling and die life analysis
- Optimization suite (gating, riser, process parameters)
- DOE and response surface analysis
- AI-assisted gating design
- API access for automation
- Team collaboration (shared projects, review workflows)

### Enterprise — Custom
- On-premise deployment for IP-sensitive foundries
- Custom alloy characterization and database development
- Integration with foundry MES/ERP systems
- Digital twin: real-time process monitoring linked to simulation model
- Dedicated support, training, and model validation services

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | CastSim Advantage |
|-----------|-----------|------------|-------------------|
| MAGMASOFT | Gold standard for casting simulation, autonomous optimization | $40-80K/seat, complex setup, long learning curve | 10x lower cost, browser-based, AI-assisted design |
| ProCAST (ESI) | Strong microstructure, investment casting, DS/SX | $30-50K/seat, requires ITAR for aerospace, steep learning curve | Cloud-native, faster turnaround, integrated workflow |
| FLOW-3D CAST | Best-in-class VOF filling accuracy | $25-45K/seat, weak on stress/microstructure, desktop only | Full filling + solidification + stress + microstructure in one platform |
| SOLIDCast | Affordable ($5K), easy to use | No filling simulation, simplified thermal model, no stress/microstructure | Full physics coverage at accessible price point |
| AnyCasting | Growing Korean competitor, good HPDC | Limited microstructure, smaller alloy database, regional support | Global platform, broader alloy/process coverage, cloud scalability |

---

## MVP Scope (v1.0)

### In Scope
- 3D mold filling simulation (VOF free-surface) for gravity sand casting
- Solidification simulation with enthalpy method
- Niyama criterion porosity mapping
- Solidification sequence visualization (last-to-solidify regions)
- 10 common alloys (A356, A380, gray iron, ductile iron, 1020 steel, 304 SS, C360 brass, AZ91, CuSn10, IN718)
- STEP/STL geometry import with automatic meshing
- Web-based 3D results visualization (fill animation, temperature, porosity contours)

### Out of Scope (v1.1+)
- Thermal stress and distortion analysis
- Microstructure prediction (DAS, grain structure)
- Die casting specific features (shot sleeve, thermal cycling)
- Optimization suite (gating, riser, process)
- Investment casting shell modeling
- Continuous casting module
- AI-assisted gating design

### MVP Timeline: 16-20 weeks
