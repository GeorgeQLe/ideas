# PartSim — Particle and Granular Flow Simulation (DEM)

## Executive Summary

PartSim is a cloud-based Discrete Element Method (DEM) simulation platform for granular materials, powder handling, and particle processing. It replaces EDEM/Altair ($25K+/seat), Rocky DEM ($20K+/seat), and PFC/Itasca ($15K+/seat) with a browser-based GPU-accelerated DEM solver supporting Hertz-Mindlin contact mechanics, adhesive particles, breakage, coupled CFD-DEM, and realistic particle shapes — at a fraction of the cost of incumbent desktop tools.

---

## Problem Statement

**The pain:**
- EDEM (Altair) costs $25,000-$50,000/seat/year; Rocky DEM costs $20,000-$40,000/seat; PFC (Itasca) costs $15,000-$30,000/seat — these prices exclude most small-to-mid-size companies in mining, pharma, and food processing
- Granular material behavior is notoriously difficult to predict analytically — bulk flow in hoppers, chutes, mills, and conveyors depends on particle size distribution, shape, moisture, and cohesion in ways that defy simple calculations
- DEM simulations are extremely computationally expensive: realistic industrial systems with millions of particles require multi-GPU workstations ($20K-$50K) or HPC clusters, creating a hardware barrier on top of the software cost
- Calibrating DEM material parameters (restitution, friction, cohesion) requires physical testing (shear cell, angle of repose, bulk density) and iterative simulation — most users guess parameters due to lack of systematic calibration workflows
- Coupling DEM with CFD for gas-solid flows (pneumatic conveying, fluidized beds, spray drying) requires separate CFD licenses and complex coupling setups between incompatible software packages

**Current workarounds:**
- Using empirical design rules (Jenike silo design, CEMA conveyor standards) that miss complex particle behavior and lead to 30-40% over-design or unexpected blockages
- Running LIGGGHTS (open-source DEM) with no GUI, manual scripting, and limited shape models, spending weeks on model setup
- Physical prototyping and testing at $50K-$500K per equipment iteration instead of simulating first
- Ignoring particle-level physics entirely and relying on bulk material flow correlations that fail for cohesive, non-spherical, or segregating materials

**Market size:** The DEM and particle simulation software market is approximately $800 million (2024). There are 80,000+ process engineers worldwide working with granular materials across mining ($100B+ industry), pharmaceuticals ($50B+ solid dosage), food processing ($30B+ powder handling), and construction materials. Equipment design failures due to poor granular flow prediction cost $5B+/year globally.

---

## Target Users

### Primary Personas

**1. Chen — Process Engineer at a Mining Company**
- Designs and troubleshoots transfer chutes, conveyor systems, and crusher feed arrangements for iron ore and coal handling
- Uses empirical CEMA guidelines but experiences chronic chute blockages and belt damage costing $1M+/year in downtime
- Needs: DEM simulation of bulk material flow through chutes and conveyors with wear prediction and design optimization

**2. Dr. Sarah — Pharmaceutical Process Scientist**
- Develops powder blending, tablet compression, and capsule filling processes for solid oral dosage forms
- Uses EDEM for blender simulations but the $50K/year license is shared across 20 engineers with chronic queuing
- Needs: affordable DEM tool for powder mixing uniformity, segregation prediction, and die filling analysis with cohesive fine powders

**3. Marco — Equipment Designer at an OEM (Hoppers, Mills, Conveyors)**
- Designs hoppers, silos, hammer mills, and screw conveyors for food and chemical processing customers
- Builds physical prototypes to test each design iteration — a $100K-$300K cost per project
- Needs: virtual prototyping of granular equipment with realistic particle shapes and size distributions to reduce physical testing cycles

---

## Solution Overview

PartSim is a cloud-based DEM simulation platform that:
1. Solves particle-particle and particle-geometry contact mechanics (Hertz-Mindlin, JKR adhesion, linear spring) with GPU acceleration for multi-million particle systems
2. Supports realistic particle shapes via multi-sphere (clump), superquadric, and polyhedral representations with automatic shape calibration from microscopy images
3. Provides coupled CFD-DEM simulation for gas-solid and liquid-solid flows (pneumatic conveying, fluidized beds, slurry transport)
4. Includes a systematic material calibration workflow matching simulation to physical bulk test data (shear cell, angle of repose, bulk density)
5. Predicts equipment wear, particle breakage/attrition, mixing/segregation, and residence time for industrial process optimization

---

## Core Features

### F1: DEM Contact Mechanics
- Hertz-Mindlin (no-slip) contact model: normal and tangential force with rolling friction (type A, B, C)
- JKR (Johnson-Kendall-Roberts) adhesion model for cohesive powders with pull-off force and hysteresis
- Linear spring-dashpot model for rapid prototyping simulations
- Edinburgh Elasto-Plastic Adhesion Model (EEPA) for pressure-dependent consolidation and caking
- Liquid bridge (capillary) model for wet granular materials with bridge volume tracking and rupture
- Electrostatic charging: triboelectric charge transfer and Coulomb interaction for powder electrification
- Bond models: parallel bond (Potyondy-Cundall) for cemented/sintered materials, with bond breakage criteria
- Heat transfer: conduction at contact, convection with fluid, radiation between particles
- Time step: automatic critical time step calculation (Rayleigh wave criterion) with user override
- Neighbor search: Verlet list with linked-cell spatial hashing, GPU-optimized broad-phase collision detection

### F2: Particle Shape and Size Distribution
- Sphere: fastest solver, baseline for calibration
- Multi-sphere (clump): 2-20 overlapping spheres for irregular shapes (rocks, tablets, grains)
- Superquadric: continuously parameterized shapes (ellipsoids, cylinders, cubes with rounded edges)
- Polyhedral: convex polyhedra with GJK/EPA contact detection for angular particles (crushed rock, crystals)
- Shape from image: automatic multi-sphere or superquadric fitting from 2D microscopy or 3D CT scan data
- Particle size distribution: normal, log-normal, Rosin-Rammler, tabulated distributions with user-defined bins
- Particle factory: volumetric and surface injection, conveyor belt feed, rain-fill for initial packing
- Aspect ratio, sphericity, and angularity controls for shape libraries
- Fiber/elongated particle model for biomass, fibers, and needles

### F3: Coupled CFD-DEM
- One-way coupling: DEM particles in prescribed fluid velocity field (drag + buoyancy)
- Two-way coupling: volume-averaged Navier-Stokes with DEM particle feedback (Euler-Lagrange)
- Four-way coupling: fluid + particle-particle contacts + particle-fluid momentum exchange
- Drag models: Ergun, Wen-Yu, Di Felice, Gidaspow, Koch-Hill-Ladd, BVK
- Fluid solver: incompressible Navier-Stokes on structured grid (finite volume) with SIMPLE/PISO pressure coupling
- Turbulence: k-epsilon, k-omega SST for resolved fluid scales
- Applications: fluidized beds, pneumatic conveying, cyclone separators, spray drying, slurry flow
- Heat and mass transfer: particle-fluid convective heat transfer (Ranz-Marshall), evaporation/drying
- Resolved CFD-DEM: immersed boundary method for large particle-to-cell ratio problems

### F4: Material Calibration Workflow
- Virtual shear cell test: simulate Schulze ring shear cell to match measured yield locus and wall friction
- Virtual angle of repose test: cone formation and dynamic angle measurement
- Virtual bulk density test: pour-and-tap density matching
- Virtual drum test: dynamic flow behavior in rotating drum for flow function calibration
- Automated parameter optimization: Bayesian optimization loop to fit DEM parameters (friction, restitution, cohesion, rolling friction) to multiple physical test results simultaneously
- Material database: pre-calibrated parameter sets for 50+ common materials (iron ore, coal, wheat, lactose, sand, limestone, pharmaceutical excipients)
- Sensitivity analysis: Morris or Sobol methods to identify most influential parameters
- Calibration report: parameter confidence intervals, goodness-of-fit metrics, validation test comparisons

### F5: Wear, Breakage, and Degradation
- Archard wear model: volumetric wear rate from contact pressure and sliding distance on equipment surfaces
- Finnie erosive wear: velocity and impact angle dependent erosion for pneumatic conveying and chutes
- Wear map visualization: cumulative wear depth on equipment walls, liners, screens
- Particle breakage: Bonded Particle Model (BPM) — spheres connected by bonds that fracture under stress
- Population Balance Model (PBM) integration: track particle size distribution evolution during comminution
- Attrition model: surface erosion producing fines (mother-daughter approach)
- Degradation index prediction: percentage of broken particles after handling/transport
- Fatigue bond breakage: cyclic loading history tracking for progressive degradation
- Replacement model: particles above breakage threshold replaced with daughter fragment distributions

### F6: Post-Processing and Design Tools
- Particle velocity, force chain, contact force, and kinetic energy field visualization
- Coarse-graining (spatial averaging): void fraction, velocity, stress tensor on Eulerian grid
- Residence time distribution (RTD) analysis for mixers, kilns, and reactors
- Mixing index (Lacey, RSD) and segregation index tracking over time
- Mass flow rate measurement at virtual sensors (planes, bins)
- Belt conveyor loading profile: cross-sectional load, edge distance, spillage prediction
- Chute design metrics: impact angle, impact velocity, stream trajectory, dead zones
- Hopper discharge: mass flow vs. funnel flow classification, critical outlet dimension (Jenike method validation)
- Animation: high-quality rendered particle flow videos with color-by-property mapping
- Report generation: simulation summary, key metrics, comparison tables, PDF/HTML export

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | TypeScript, React, Three.js / Babylon.js (3D particle visualization, equipment geometry), D3.js (RTD plots, PSD charts) |
| DEM Solver | Rust + CUDA (GPU-accelerated contact detection and force computation) → cloud GPU server |
| CFD Solver | Rust (structured grid finite volume, SIMPLE/PISO) coupled with DEM via volume averaging |
| Shape Engine | Rust (multi-sphere, superquadric GJK/EPA, polyhedral contact) → GPU kernels |
| Calibration | Python (Bayesian optimization with GPyOpt/BoTorch, virtual test automation) |
| Geometry Import | Rust + OpenCASCADE (STEP/STL/OBJ import, surface triangulation, wall representation) |
| Backend | Rust (Axum) + Python (FastAPI for calibration service, post-processing, material database) |
| Database | PostgreSQL (material parameters, simulation metadata, equipment library), S3 (simulation results, particle data, geometry files) |
| Hosting | AWS (p4d/p5 NVIDIA GPU instances for DEM solver, c7g for CFD, auto-scaling for batch runs) |

---

## Monetization

### Free Tier (Student)
- DEM simulation up to 10,000 particles (spheres only)
- Hertz-Mindlin contact model
- Basic geometry import (STL)
- 3 projects
- Community material database

### Pro — $149/month
- Up to 500,000 particles with GPU acceleration
- Multi-sphere and superquadric shapes
- JKR adhesion model
- Material calibration workflow
- Wear prediction
- STEP/STL geometry import
- 200 GPU-hours/month

### Advanced — $399/user/month
- Everything in Pro
- Unlimited particle count (multi-GPU cloud)
- Polyhedral particles
- Coupled CFD-DEM
- Particle breakage models
- Bond models
- Custom contact models (scripted)
- API access
- 1000 GPU-hours/month

### Enterprise — Custom
- On-premise GPU deployment
- Custom material calibration service with lab testing partnership
- Integration with process simulation tools (Aspen, gPROMS)
- Multi-equipment process line simulation
- Dedicated support with SLA
- Training and certification

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | PartSim Advantage |
|-----------|-----------|------------|-------------------|
| EDEM (Altair) | Industry standard, large user base | $25-50K/seat, requires local GPU workstation | Cloud GPU, 5x cheaper, no hardware investment |
| Rocky DEM | Excellent polyhedral shapes, GPU native | $20-40K/seat, limited CFD coupling | Integrated CFD-DEM, affordable |
| LIGGGHTS | Free, open-source, extensible | No GUI, manual scripting, limited shapes | Modern UX, automated calibration, cloud-native |
| PFC (Itasca) | Best for geomechanics, mature bond models | $15-30K/seat, slow (CPU only), niche | GPU acceleration, broader industry scope |
| StarCCM+ DEM | Tight CFD integration (Siemens) | $30K+/seat, DEM is secondary feature | DEM-first design, deeper granular physics |

---

## MVP Scope (v1.0)

### In Scope
- Hertz-Mindlin DEM solver with GPU acceleration (up to 1M spherical particles)
- Multi-sphere particle shapes (clumps of 2-10 spheres)
- STL geometry import for equipment walls
- Particle factory (volume fill, surface injection)
- Basic post-processing: velocity/force visualization, mass flow sensors, angle of repose measurement
- Material calibration: virtual angle of repose and bulk density matching
- 3D interactive particle visualization in browser

### Out of Scope (v1.1+)
- Coupled CFD-DEM
- Polyhedral and superquadric particle shapes
- JKR adhesion and liquid bridge models
- Particle breakage (BPM)
- Wear prediction
- Automated Bayesian calibration workflow

### MVP Timeline: 16-20 weeks
