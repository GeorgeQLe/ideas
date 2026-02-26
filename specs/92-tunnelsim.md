# TunnelSim — Tunnel Engineering Platform

## Executive Summary

TunnelSim is a cloud-based tunnel engineering platform for design and analysis of tunnels constructed by TBM (tunnel boring machine) and NATM/SEM (New Austrian Tunnelling Method / Sequential Excavation Method). It replaces PLAXIS 3D Tunnel ($15K+/seat), FLAC3D ($12K+/seat), RS3 ($8K+/seat), and UDEC/3DEC ($10K+/seat) with an integrated browser-based environment covering ground characterization, excavation simulation, support design, and ground settlement prediction.

---

## Problem Statement

**The pain:**
- PLAXIS 3D Tunnel costs $15,000-$30,000/seat/year and FLAC3D costs $12,000-$25,000/seat; tunnel projects require 3D analysis but license costs limit simulation to a handful of critical sections
- Tunnel design involves interdependent analyses — ground classification, excavation sequencing, support interaction, settlement prediction — that are done in separate tools with manual data transfer
- Convergence-confinement analysis (the foundational method for NATM design) requires iterating between ground reaction curves and support characteristic curves, which is done manually or in custom spreadsheets
- Rock mass classification systems (RMR, Q-system, GSI) are evaluated by hand from borehole logs and then manually mapped to support recommendations — a subjective, error-prone process
- TBM tunnel design for segmental linings requires separate structural analysis (thrust jack loads, grout pressure, handling loads) that most geotechnical FEA tools handle poorly
- Ground settlement prediction for urban tunnels is critical but current 3D FEA models take days to run and are often simplified to 2D, missing 3D effects at portals, junctions, and curves

**Current workarounds:**
- Using 2D plane-strain analysis (PLAXIS 2D, Phase2) for 3D tunnel problems by applying volume loss or stress reduction factors — losing accuracy at tunnel faces, junctions, and portal zones
- Building custom spreadsheets for convergence-confinement calculations and rock support interaction diagrams
- Using RS2/RS3 (Rocscience) for rock tunnel analysis but switching to PLAXIS for soft ground tunnels in the same project
- Hand-calculating segmental lining forces from simplified beam-spring models instead of full 3D analysis

**Market size:** The tunnel engineering software market is approximately $800 million (2024), part of the broader $4.5 billion geotechnical and underground construction software sector. There are an estimated 40,000+ tunnel/geotechnical engineers worldwide specializing in underground construction. Global tunnel construction spending exceeds $100 billion annually, driven by metro expansion, highway tunnels, and utility tunnels.

---

## Target Users

### Primary Personas

**1. Elena — Tunnel Design Engineer at a Major Contractor**
- Designs NATM excavation sequences and temporary support for highway and rail tunnels in rock
- Uses FLAC3D for 3D analysis but spends 2-3 weeks building each model, and runs are limited by license availability
- Needs: rapid 3D tunnel model generation from alignment data, automatic excavation stage simulation, and real-time support design iteration

**2. Raj — TBM Tunnel Engineer at a Metro Project**
- Designs segmental lining for shield TBM tunnels in soft ground, managing ground settlement for an urban metro project
- Uses PLAXIS 2D for settlement but knows 2D overestimates settlement at straight runs and underestimates it at curves and junctions
- Needs: efficient 3D analysis for settlement prediction, segmental lining structural design, and TBM face pressure calculation integrated with geotechnical analysis

**3. Dr. Fischer — Geotechnical Consultant for Underground Structures**
- Performs ground characterization and support design for tunnel projects worldwide, from feasibility through detailed design
- Currently uses Rocscience tools (Dips, RocData, Unwedge, RS3) as separate applications with manual data flow
- Needs: integrated platform from borehole data through rock mass classification, kinematic block analysis, and numerical modeling with consistent ground model throughout

---

## Solution Overview

TunnelSim is a cloud-based tunnel engineering platform that:
1. Imports geological/geotechnical data (boreholes, cross-sections, rock mass classifications) and builds 3D ground models with spatial variability of properties
2. Generates 3D tunnel models from alignment geometry (horizontal/vertical curves, cross-section profiles) with automatic mesh generation optimized for excavation simulation
3. Simulates excavation sequences (full-face, top-heading/bench/invert, side-drift) with step-by-step ground-support interaction including shotcrete hardening, rock bolt activation, and steel set installation
4. Designs TBM segmental linings under thrust jack loads, grout pressure, ground loads, and long-term conditions with joint modeling
5. Predicts ground settlement profiles (transverse and longitudinal) with building damage assessment using greenfield and building-interaction methods

---

## Core Features

### F1: Ground Modeling and Characterization
- Borehole log import (AGS, CSV, LAS) with stratigraphic interpretation and 3D geological model generation
- Rock mass classification calculators: RMR (Bieniawski 1989), Q-system (Barton 2002), GSI (Hoek-Brown), with automatic mapping between systems
- Automatic support class recommendation based on rock mass quality (RMR/Q support charts)
- Hoek-Brown to Mohr-Coulomb parameter conversion with stress-range fitting
- Spatial variability: geostatistical interpolation (kriging) of ground properties between boreholes
- Discontinuity analysis: stereonet plotting (Dips-equivalent), kinematic analysis for wedge/planar/toppling failure, Unwedge-equivalent block analysis around tunnel
- Water table and pore pressure modeling: phreatic surface, piezometric head distributions, confined aquifer modeling
- In-situ stress definition: gravitational, locked-in tectonic stresses, K0 specification per stratum

### F2: Tunnel Geometry and Mesh Generation
- Alignment import: LandXML, horizontal/vertical curve data, or manual definition of tunnel centerline in 3D
- Standard cross-section templates: circular (TBM), horseshoe (NATM), D-shape, rectangular, multi-lane highway, twin-tube metro
- Variable cross-sections along alignment for transition zones and enlargement caverns
- Automatic 3D mesh generation: tetrahedral/hexahedral hybrid mesh with refinement zones around tunnel perimeter
- Multi-tunnel configurations: twin tunnels, junction caverns, cross-passages, ventilation shafts
- Portal zone modeling with slope geometry and portal structure
- Mesh quality control: aspect ratio, Jacobian, element size gradation with automatic repair
- Parametric model updates: change alignment, cross-section, or ground model without re-meshing from scratch

### F3: NATM/SEM Excavation Simulation
- Excavation sequence definition: full-face, top-heading/bench, top-heading/bench/invert, side-drift, center-drift, pilot tunnel with enlargement
- Step-by-step 3D excavation simulation with core-face stress relief and load redistribution
- Convergence-confinement method: automatic ground reaction curve (GRC), support characteristic curve (SCC), and longitudinal deformation profile (LDP) generation
- Shotcrete modeling: time-dependent stiffness gain (J2 model, shotcrete constitutive model), early-age creep, temperature effects on curing
- Rock bolt modeling: fully grouted, end-anchored, Swellex, self-drilling; with pull-out capacity and bond-slip behavior
- Steel set (lattice girder, W-beam, HEB) modeling as embedded beam elements
- Waterproofing membrane and final lining installation at user-defined distance behind the face
- Face stability analysis: face pressure required for soft ground, core reinforcement (fiberglass dowels) for weak rock
- Real-time convergence monitoring: crown settlement, horizontal convergence, floor heave at virtual monitoring sections
- Trigger level evaluation: compare predicted displacements against warning/action/alarm thresholds

### F4: TBM Tunnel and Segmental Lining Design
- Earth pressure balance (EPB) and slurry TBM face pressure calculation: required support pressure from wedge/silo theory and FEM
- TBM advance simulation: continuous excavation with shield-ground interaction, tail void grouting, jack thrust application
- Segmental lining structural analysis: beam-spring model and 3D shell model with segment joints
- Joint modeling: flat joints, convex-convex joints with rotational stiffness, gasket compression, bolt tension
- Loading cases: TBM thrust jack loads (asymmetric), grouting pressure, ground and water pressure, long-term creep loads, fire loading
- Segment design checks: ULS bending + axial, SLS crack width, bursting/spalling reinforcement at joints, handling and stacking loads
- Lining distortion analysis: ovalisation under asymmetric loading, tolerance for ring build deviations
- Longitudinal joint pattern (stagger) and ring taper geometry for curved alignment
- Annular grout modeling: injection pressure, grout migration, gap closure

### F5: Ground Settlement Prediction
- Greenfield settlement: Gaussian trough parameters (volume loss, trough width factor K) from empirical correlations and numerical analysis
- 3D settlement analysis: full 3D FEM with tunnel advance simulation for transverse and longitudinal settlement profiles
- Volume loss calibration: back-analysis from monitoring data to update numerical model predictions
- Multiple tunnel interaction: superposition for twin tunnels with modification factors for sequential excavation
- Building damage assessment: tensile strain calculation (Burland & Wroth method), damage category classification (negligible to very severe)
- Soil-structure interaction: equivalent beam model and 3D building-on-foundation model for detailed damage assessment
- Utility impact assessment: angular distortion and horizontal strain at utility depth with pipeline vulnerability classification
- Pre-existing building data import: footprint, foundation type, structural stiffness for interaction analysis
- Settlement monitoring data import and comparison with predictions (observational method support)

### F6: Reporting and Instrumentation Integration
- Automated design report generation: ground conditions summary, analysis methodology, support design, settlement predictions with code references
- Geotechnical Baseline Report (GBR) data extraction support
- Instrumentation layout planning: recommended monitoring section spacing and instrument types based on ground conditions and risk
- Real-time monitoring data import: convergence pins, extensometers, inclinometers, settlement points, piezometers
- Predicted vs. measured comparison plots with trend analysis and alarm threshold evaluation
- Construction progress tracking: as-built excavation advance linked to monitoring data timeline
- Risk register integration: geotechnical risk events linked to monitoring triggers
- Standards compliance documentation: Eurocode 7, AASHTO LRFD Bridge (for road tunnels), FHWA tunnel design guidelines

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | TypeScript, React, Three.js (3D tunnel model, geological cross-sections), D3.js (convergence plots, settlement profiles) |
| FEM Solver | Rust/C++ (3D elasto-plastic FEM, Mohr-Coulomb, Hoek-Brown, Hardening Soil, shotcrete constitutive models) → server-side |
| Mesh Generator | Rust (advancing-front tetrahedral, hexahedral tunnel-specific structured meshing) |
| Ground Modeler | Python (geostatistical interpolation, stereonet analysis, rock mass classification algorithms) |
| Backend | Rust (Axum) + Python (FastAPI for geological modeling, monitoring analytics) |
| AI/ML | Python (ground condition prediction from borehole data, settlement prediction surrogate models, support class recommendation) |
| Database | PostgreSQL (borehole data, ground parameters, tunnel inventory), PostGIS (alignment geospatial data), S3 (3D meshes, monitoring data, results) |
| Hosting | AWS (c7g/c7i for FEM, auto-scaling for parametric studies, ECS for backend, CloudFront for frontend) |

---

## Monetization

### Free Tier (Student)
- 2D plane-strain tunnel analysis only
- Single excavation stage (no sequential excavation)
- Mohr-Coulomb constitutive model only
- 3 projects
- Max 10,000 mesh elements

### Pro — $179/month
- Full 3D tunnel analysis
- Sequential excavation simulation (up to 50 stages)
- All constitutive models (Hoek-Brown, Hardening Soil, shotcrete)
- Convergence-confinement method
- Settlement prediction (Gaussian trough)
- Rock mass classification tools
- Report generation

### Advanced — $399/user/month
- Everything in Pro
- Unlimited excavation stages
- TBM segmental lining design
- 3D settlement analysis with building damage assessment
- Multi-tunnel configurations (junctions, cross-passages)
- Monitoring data integration and back-analysis
- Unlimited cloud compute
- API access

### Enterprise — Custom
- On-premise deployment for sensitive infrastructure projects
- Real-time monitoring dashboard with IoT sensor integration
- Custom constitutive model implementation
- Multi-project portfolio management for contractors
- BIM integration (IFC tunnel extensions)
- Training, certification, and dedicated support

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | TunnelSim Advantage |
|-----------|-----------|------------|-------------------|
| PLAXIS 3D Tunnel | Industry standard for soft ground, excellent constitutive models | $15-30K/seat, slow 3D meshing, no rock classification or support design integration | Integrated ground model → analysis → design, cloud-accelerated |
| FLAC3D (Itasca) | Best for rock mechanics, explicit solver for large deformation | $12-25K/seat, scripting-heavy, steep learning curve | Visual model builder, tunnel-specific workflows |
| RS3 (Rocscience) | Good rock tunnel analysis, integrates with Rocscience suite | $8-12K/seat, limited soft ground models, no segmental lining | All ground types, TBM + NATM in one platform |
| UDEC/3DEC (Itasca) | Best for discontinuous rock (jointed rock masses) | $10-20K/seat, specialized (not general tunneling), complex setup | Continuous + discontinuous analysis, easier workflow |
| Phase2/RS2 (Rocscience) | Good 2D tunnel analysis, affordable | 2D only, misses 3D effects at faces and junctions | Full 3D with tunnel-specific features |

---

## MVP Scope (v1.0)

### In Scope
- 2D and 3D tunnel model generation from alignment and cross-section definition
- Automatic mesh generation around tunnel excavation
- Mohr-Coulomb and Hoek-Brown elasto-plastic ground models
- Sequential excavation simulation (top-heading/bench) with stress release
- Shotcrete and rock bolt support modeling (elastic, time-independent)
- Convergence-confinement method (GRC, SCC, LDP)
- Ground settlement prediction (Gaussian trough method)
- Rock mass classification calculator (RMR, Q, GSI)

### Out of Scope (v1.1+)
- TBM segmental lining design
- Advanced constitutive models (Hardening Soil, shotcrete time-dependent)
- 3D settlement with building damage assessment
- Monitoring data integration and back-analysis
- Multi-tunnel configurations and junction caverns
- Discontinuum analysis (jointed rock blocks)

### MVP Timeline: 16-20 weeks
