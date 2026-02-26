# CompForge — Composite Materials Design and Analysis Platform

## Executive Summary

CompForge is a cloud-based platform for designing, analyzing, and manufacturing fiber-reinforced composite structures — carbon fiber, glass fiber, Kevlar, and natural fiber composites. It replaces HyperSizer ($15K+/seat), Siemens Fibersim ($20K+/seat), and ANSYS ACP ($10K+/seat add-on) with browser-based laminate design, failure analysis, and manufacturing simulation.

---

## Problem Statement

**The pain:**
- Composite design requires specialized tools separate from standard FEA: laminate definition, ply-by-ply analysis, failure criteria, and manufacturing process simulation
- HyperSizer (Collier Research) costs $15,000-$30,000/seat; Fibersim (Siemens) costs $20,000-$40,000/seat for draping simulation; ANSYS ACP costs $10,000+ as an add-on to an already expensive base license
- Manual layup schedules for aerospace composite parts (100+ plies) are error-prone; a single ply orientation mistake can compromise structural integrity
- Composite manufacturing defects (wrinkles, bridging, fiber washout) can only be detected after expensive autoclave cure ($10K-$100K per part)
- Certification of composite aircraft structures requires extensive analysis to damage tolerance, barely-visible impact damage (BVID), and environmental knockdown — enormously complex and documentation-heavy

**Current workarounds:**
- Using Excel for classical lamination theory (CLT) calculations — limited to flat panels, no curvature or 3D effects
- Using standard FEA (ANSYS, Abaqus) with manual composite property input and post-processing for failure criteria
- Physical coupon testing for every laminate configuration ($500-$2,000 per coupon, 50+ coupons per laminate family)
- Manufacturing trial-and-error for complex curved parts

**Market size:** The composites market is $100B+ globally, growing 8% CAGR driven by aerospace, automotive, wind energy, and marine applications. The composite design software segment is approximately $800 million. There are 100,000+ composite design/manufacturing engineers worldwide.

---

## Target Users

### Primary Personas

**1. Alex — Composite Structures Engineer at an Aerospace Company**
- Designs carbon fiber wing skins, fuselage panels, and control surfaces
- Uses HyperSizer for laminate sizing but needs better integration with FEA for detail stress
- Needs: integrated laminate design → FEA → failure analysis → manufacturing simulation workflow

**2. Maria — Wind Turbine Blade Engineer**
- Designs glass fiber/carbon fiber hybrid blades up to 100m length
- Needs fatigue analysis under variable wind loading (10^8 cycles)
- Needs: composite fatigue analysis with SN curves, Goodman diagrams, and spectrum loading

**3. Diego — Automotive Composite Manufacturing Engineer**
- Plans RTM (resin transfer molding) and compression molding processes for composite body panels
- Needs to predict resin flow, fiber orientation, and cure to prevent voids and dry spots
- Needs: manufacturing process simulation (draping, resin flow, cure) integrated with structural analysis

---

## Solution Overview

CompForge is a cloud-based composite platform that:
1. Provides laminate design tools: Classical Lamination Theory (CLT), stacking sequence optimization, balanced/symmetric enforcement, ply-drop planning
2. Runs composite-specific FEA: layered shell/solid elements with progressive failure analysis using multiple failure criteria (Tsai-Wu, Hashin, Puck, LaRC)
3. Simulates manufacturing: ply draping on curved tooling, resin infusion (RTM/VARTM), autoclave cure (temperature-pressure cycle, residual stress from cure)
4. Performs damage tolerance assessment: barely-visible impact damage (BVID) residual strength, delamination growth under fatigue
5. Generates certification-grade documentation for aerospace (per CMH-17, ASTM standards) and automotive (per SAE) applications

---

## Core Features

### F1: Laminate Design
- Classical Lamination Theory (CLT): A, B, D matrices, ABD inverse, laminate stiffness and compliance
- Laminate properties: Ex, Ey, Gxy, νxy, thermal expansion coefficients, bending stiffness
- Stacking sequence design: symmetric, balanced, quasi-isotropic, optimized
- Ply materials database: T300/914, IM7/8552, T700/2510, AS4/3501-6, E-glass/epoxy, Kevlar/epoxy, natural fibers, and 200+ more with A/B-basis allowables
- Core materials: honeycomb (Nomex, aluminum), foam (Rohacell, PVC, balsa)
- Sandwich panel analysis: face sheet stress, core shear, wrinkling, dimpling, crimping
- Ply drop and taper design with stagger rules
- Carpet plots for parametric laminate studies

### F2: Composite FEA
- Layered shell elements (Mindlin-Reissner) with ply-by-ply stress recovery
- Layered solid elements (3D) for thick composites and through-thickness stress
- Cohesive zone elements for delamination modeling
- Progressive failure analysis: ply-by-ply stiffness degradation after failure initiation
- Failure criteria: max stress, max strain, Tsai-Wu, Tsai-Hill, Hashin (fiber/matrix tension/compression), Puck, LaRC03/04, Cuntze
- First-ply failure and last-ply failure envelopes
- Buckling analysis with post-buckling behavior (nonlinear)
- Thermal stress analysis: cure-induced residual stress, CTE mismatch in hybrid laminates
- Bolted joint analysis: bearing/bypass interaction, open-hole tension/compression

### F3: Manufacturing Simulation
- Ply draping simulation: kinematic and FEA-based draping on doubly-curved tool surfaces
- Fiber angle deviation maps from draping (impact on structural performance)
- Flat pattern generation: unfold draped plies for cutting (nesting optimization)
- Resin flow simulation: RTM, VARTM, and infusion with Darcy's law in anisotropic preform
- Fill time and pressure prediction, race-tracking along channels, last-point-to-fill identification
- Cure simulation: degree of cure, exothermic temperature rise, residual stress from chemical shrinkage
- Autoclave pressure-temperature cycle optimization
- Spring-back prediction: dimensional change after tool removal due to residual stress

### F4: Damage Tolerance
- Impact simulation: low-velocity impact (LVI) damage prediction using continuum damage mechanics
- BVID definition: damage area and residual strength at barely-visible threshold per ASTM D7136/D7137
- Compression after impact (CAI) prediction
- Delamination growth under fatigue: virtual crack closure technique (VCCT), cohesive zone
- Fatigue analysis: constant life diagrams, SN curves for composite laminates, Goodman-type approaches
- Spectrum loading: rainflow counting, damage accumulation (Miner's rule)
- Environmental effects: moisture absorption (Fickian diffusion), knockdown factors per CMH-17
- Hot-wet, cold-dry, room temperature conditions

### F5: Optimization
- Laminate thickness optimization: minimum weight for strength/stiffness constraints
- Stacking sequence optimization: genetic algorithm with manufacturing constraints
- Topology optimization for composite structures (variable thickness, ply orientation)
- Blending: ensure stacking sequence compatibility at panel transitions
- Multi-objective: minimize weight + manufacturing complexity + cost
- Ply count and material selection optimization

### F6: Certification Documentation
- CMH-17 (Composite Materials Handbook) compliant analysis reports
- Building block approach documentation: coupon → element → subcomponent → component
- Statistical basis values: A-basis, B-basis from coupon test data
- Knockdown factor summary: environment, damage, fastener
- Certification test matrix generation
- Margin of safety summary tables with failure mode identification
- PDF report with analysis methodology, material allowables, and results

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | TypeScript, React, Three.js (3D composite layup visualization, draping) |
| CLT Engine | Rust (laminate theory, ABD matrices, failure criteria) → WASM |
| FEA Solver | Rust (layered shell/solid FEA, progressive failure, buckling) → server |
| Draping | Rust (kinematic and FEA-based draping) → server |
| Resin Flow | Rust (Darcy's law FEA, fill simulation) → server |
| Cure | Rust (coupled thermal-chemical-mechanical cure simulation) |
| Optimization | Python (GA for stacking sequence, SciPy for sizing) |
| Backend | Rust (Axum) + Python (FastAPI for optimization, reporting) |
| Compute | AWS (CPU for FEA, auto-scaling for optimization) |
| Database | PostgreSQL (materials, allowables, stacking rules), S3 (meshes, results) |

---

## Monetization

### Free Tier (Student)
- CLT laminate calculator (unlimited)
- 10 ply materials
- Failure envelope generation
- Carpet plots
- 3 projects

### Pro — $149/month
- Composite FEA (layered shell, up to 100K elements)
- Full material database
- Progressive failure analysis
- Basic draping simulation
- Flat pattern generation
- Report generation

### Advanced — $349/user/month
- Everything in Pro
- Unlimited FEA model size
- Manufacturing simulation (resin flow, cure)
- Damage tolerance (impact, CAI, fatigue)
- Stacking sequence optimization
- Bolted joint analysis
- API access

### Enterprise — Custom
- On-premise deployment
- Custom material allowable database management
- Integration with AFP/ATL programming systems
- Certification documentation automation
- Training and certification

---

## MVP Scope (v1.0)

### In Scope
- CLT laminate calculator with ABD matrices
- Material database (50 common composite systems)
- Failure criteria: Tsai-Wu, max stress, max strain, Hashin
- Failure envelope generation
- Carpet plots for parametric study
- First-ply failure analysis for flat panels
- Simple layered shell FEA (up to 10K elements)
- PDF report with laminate properties and failure margins

### Out of Scope (v1.1+)
- Progressive failure
- Manufacturing simulation (draping, resin flow, cure)
- Damage tolerance
- Optimization
- Certification documentation
- Sandwich panel analysis

### MVP Timeline: 12-16 weeks
