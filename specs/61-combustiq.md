# CombustiQ — Combustion and Reacting Flow Simulation Platform

## Executive Summary

CombustiQ is a cloud-based combustion engineering platform for simulating reacting flows in gas turbines, internal combustion engines, industrial burners, and rocket engines. It replaces ANSYS Fluent combustion modules ($40K+/seat), CONVERGE CFD ($30K+/seat), and Cantera/OpenFOAM custom setups with browser-based combustion CFD, chemical kinetics, and emissions prediction.

---

## Problem Statement

**The pain:**
- ANSYS Fluent with combustion modules costs $40,000-$80,000/seat/year; CONVERGE CFD (for IC engines) costs $30,000-$50,000/seat
- Combustion simulation requires coupling fluid dynamics, chemical kinetics (100+ species, 1000+ reactions), heat transfer, turbulence-chemistry interaction, and pollutant formation — the most complex multi-physics problem in engineering
- Detailed chemical mechanisms (GRI-Mech 3.0 has 53 species/325 reactions; jet fuel mechanisms have 2,000+ species) make simulations computationally prohibitive
- Emission regulations (NOx, CO, soot, unburned hydrocarbons) are tightening globally, requiring accurate pollutant prediction that simplified models cannot provide
- Hydrogen combustion for decarbonization introduces new challenges (flashback, NOx at lean conditions, thermoacoustic instabilities) that require simulation

**Current workarounds:**
- Using simplified combustion models (flamelet, assumed PDF) that miss critical phenomena like extinction, reignition, and pollutant formation
- Running Cantera (free) for 0D/1D chemical kinetics but without any spatial resolution
- OpenFOAM with reactingFoam for simple flames, but lacking robustness and detailed chemistry
- Engine manufacturers running 6-month physical test campaigns instead of simulation

**Market size:** The combustion simulation market is approximately $1.5 billion (2024) within broader CFD. Gas turbine, automotive, aerospace propulsion, and industrial heating collectively represent $500B+ in annual equipment sales, all requiring combustion simulation for design and emissions compliance.

---

## Target Users

### Primary Personas

**1. Dr. Rodriguez — Gas Turbine Combustion Engineer**
- Designs low-NOx combustors for industrial gas turbines
- Uses ANSYS Fluent but simulations with detailed chemistry take weeks on HPC clusters
- Needs: GPU-accelerated combustion CFD with detailed chemistry and NOx prediction

**2. Kenji — IC Engine Simulation Engineer**
- Simulates diesel and gasoline direct injection combustion for emissions compliance
- Uses CONVERGE CFD but license costs limit the number of design iterations
- Needs: spray combustion simulation with soot and NOx prediction at lower cost

**3. Dr. Chen — Hydrogen Combustion Researcher**
- Studies hydrogen/ammonia fuel blends for zero-carbon gas turbines
- Needs detailed chemistry to predict flashback, lean blowout, and thermoacoustic instability
- Needs: reacting flow LES (Large Eddy Simulation) with detailed H2/NH3 chemistry

---

## Solution Overview

CombustiQ is a cloud-based combustion platform that:
1. Provides chemical mechanism management: import, reduce, and validate detailed kinetic mechanisms with 0D/1D reactor simulations
2. Runs 3D reacting flow CFD with RANS and LES turbulence models, including turbulence-chemistry interaction (flamelet, transported PDF, EDC)
3. GPU-accelerates chemistry integration — the bottleneck in combustion CFD — achieving 10-50x speedup over CPU
4. Predicts emissions: thermal and prompt NOx, N2O pathway, CO, soot (two-equation, method of moments), and unburned hydrocarbons
5. Supports specialized applications: spray combustion (Lagrangian spray, breakup, evaporation), premixed flame propagation, diffusion flames, and detonation

---

## Core Features

### F1: Chemical Kinetics
- Mechanism import: Chemkin format, Cantera YAML
- 0D reactor simulations: constant pressure, constant volume, perfectly stirred reactor (PSR), plug flow reactor (PFR)
- Ignition delay time, laminar flame speed, extinction strain rate computation
- Mechanism reduction: DRG, DRGEP, sensitivity analysis — automatically reduce 2000-species mechanisms to 50-100 for CFD
- Tabulation: flamelet generation (FGM/FPV), ISAT for on-the-fly chemistry acceleration
- Mechanism validation against experimental databases (chemical kinetics benchmarks)
- Pre-built mechanisms: GRI-Mech 3.0 (methane), Foundational Fuel Chemistry Model (FFCM), H2/syngas, NH3, ethanol, n-heptane, iso-octane, Jet-A surrogates

### F2: Reacting Flow CFD
- Mesh types: structured hex, unstructured tet/poly, AMR (adaptive mesh refinement) for flame fronts
- Turbulence models: k-ε, k-ω SST, Spalart-Allmaras (RANS); Smagorinsky, WALE, dynamic (LES)
- Combustion models: eddy dissipation concept (EDC), flamelet (steady/unsteady), transported PDF (Monte Carlo), partially premixed (FGM/FPV), thickened flame (LES)
- Finite-rate chemistry with GPU-accelerated ODE integration (CVODE on GPU)
- Low-Mach number formulation for efficient combustion simulation
- Compressible formulation for detonation and high-speed reacting flows
- Radiation: discrete ordinates, P1, optically thin, weighted-sum-of-gray-gases for combustion products

### F3: Emissions Prediction
- NOx: thermal (Zeldovich), prompt (Fenimore), N2O pathway, fuel-bound nitrogen
- CO: CO oxidation kinetics with residence time effects
- Soot: two-equation (Moss-Brookes), method of moments with interpolative closure (MOMIC), sectional method
- Unburned hydrocarbons (UHC): detailed chemistry or reduced model
- Particulate matter: size distribution prediction
- Post-flame equilibrium and quenching effects on emissions
- Emissions maps: 2D/3D visualization of pollutant formation zones

### F4: Spray Combustion
- Lagrangian particle tracking for fuel spray droplets
- Primary breakup: WAVE, KH-RT, Reitz-Diwakar models
- Secondary breakup: TAB, WAVE, Kelvin-Helmholtz
- Droplet evaporation: rapid mixing, diffusion limit, multi-component
- Spray-wall interaction: Bai-Gosman, stick/spread/splash
- Fuel injection: GDI, diesel common rail, gas turbine airblast atomizers
- Spray characterization: SMD, penetration, spray angle
- Ignition: auto-ignition delay, spark ignition

### F5: Thermoacoustics
- Linear stability analysis for combustion instabilities
- Flame transfer function (FTF) and flame describing function (FDF) extraction from LES
- Helmholtz solver for acoustic eigenfrequencies of combustor geometry
- Acoustic boundary conditions: impedance, reflection coefficient
- Rayleigh index calculation for instability identification
- Active control simulation: feedback control strategies for instability suppression

### F6: Application Workflows
- Gas turbine combustor: swirl burner, pilot, staging, dilution jets, liner cooling
- IC engine: port injection, direct injection, compression ignition, spark ignition, HCCI/RCCI
- Industrial burner: premixed, partially premixed, diffusion, oxy-fuel
- Rocket engine: impinging jets, coaxial injection, ablation
- Furnace and kiln: radiative heat transfer dominated combustion

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | TypeScript, React, WebGPU (3D flame visualization, contour plots) |
| CFD Solver | Rust (FVM, SIMPLE/PISO pressure-velocity coupling, LES) → GPU-accelerated |
| Chemistry | Rust + CUDA (GPU ODE integration, Cantera-compatible thermodynamics) |
| Spray | Rust (Lagrangian particle tracking, breakup models) |
| Thermoacoustics | Rust (Helmholtz solver) + Python (FTF extraction) |
| Backend | Rust (Axum) + Python (FastAPI for mechanism reduction, post-processing) |
| Compute | AWS (p4d/p5 GPU for chemistry and LES, auto-scaling) |
| Database | PostgreSQL (mechanisms, experiments, projects), S3 (meshes, solutions) |

---

## Monetization

### Academic — $149/month per lab
- 0D/1D chemical kinetics (unlimited)
- 3D RANS combustion (up to 2M cells)
- Pre-built mechanisms (GRI-Mech, H2)
- 5 user seats
- 500 GPU compute hours/month

### Pro — $499/month
- LES combustion
- Mechanism reduction tools
- Spray combustion
- Emissions prediction (all models)
- Custom mechanisms
- Unlimited compute

### Enterprise — Custom
- On-premise / air-gapped deployment
- Custom combustion model development
- Thermoacoustic analysis
- IC engine and gas turbine workflows
- Integration with test bench data systems
- Dedicated HPC allocation

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | CombustiQ Advantage |
|-----------|-----------|------------|---------------------|
| ANSYS Fluent | Comprehensive CFD + combustion | $40-80K/seat, CPU-limited chemistry | GPU-accelerated chemistry, 10x cheaper |
| CONVERGE | Best for IC engines, AMR | $30-50K/seat, narrow application scope | Broader scope, cloud-native, GPU chemistry |
| Cantera | Free, excellent 0D/1D kinetics | No spatial resolution (no CFD) | Integrated kinetics + 3D CFD |
| OpenFOAM reactingFoam | Free, flexible | Limited combustion models, poor convergence | Robust models, GPU chemistry, cloud compute |
| STAR-CCM+ | Good combustion capability | $20-40K/seat, general-purpose | Combustion-native, deeper chemistry features |

---

## MVP Scope (v1.0)

### In Scope
- 0D reactor simulations (constant P, PSR) with Chemkin mechanism import
- Mechanism reduction (DRG)
- 2D/3D RANS with k-ε turbulence
- EDC combustion model with finite-rate chemistry
- GRI-Mech 3.0 pre-built (methane/air)
- Temperature, species, velocity field visualization
- Basic NOx prediction (thermal Zeldovich)

### Out of Scope (v1.1+)
- LES
- Spray combustion
- Soot modeling
- Thermoacoustics
- GPU-accelerated chemistry
- IC engine and gas turbine workflows

### MVP Timeline: 16-20 weeks
