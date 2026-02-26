# FormSim — Metal Forming and Stamping Simulation Platform

## Executive Summary

FormSim is a cloud-based platform for simulating sheet metal forming, stamping, forging, rolling, and extrusion processes. It replaces AutoForm ($30K+/seat), PAM-STAMP ($20K+/seat), and DEFORM ($15K+/seat) with browser-based forming simulation, die design assistance, and springback prediction.

---

## Problem Statement

**The pain:**
- AutoForm costs $30,000-$60,000/seat/year for sheet metal stamping; PAM-STAMP (ESI) costs $20,000-$40,000/seat; DEFORM (SFTC) costs $15,000-$30,000/seat for bulk forming
- Die tryout for a new automotive stamping tool costs $500K-$2M and takes 3-6 months; simulation can reduce tryout iterations by 50-70% but tools are too expensive for many manufacturers
- Springback prediction (elastic recovery after forming) is the most critical and difficult aspect; inaccurate prediction leads to dimensional non-conformance and repeated die compensation iterations
- Automotive lightweighting (AHSS, aluminum, magnesium) introduces new formability challenges that cannot be solved by experience alone — simulation is essential
- Small stamping shops and tool makers cannot afford AutoForm but compete for work requiring simulation-based feasibility analysis

**Current workarounds:**
- Using forming limit diagrams (FLD) with hand-drawn strain circles on gridded blanks (physical experiment, not simulation)
- Trial-and-error die tryout with experienced die makers ($100-$200/hour, 3-6 month lead time)
- Using general-purpose FEA (LS-DYNA, Abaqus Explicit) with manual setup — possible but requires 100+ hours of expertise per model
- Circle grid analysis on physical parts to validate forming but without predictive capability

**Market size:** The metal forming simulation market is approximately $800 million (2024). The global metal stamping market is $250B+; forging is $100B+. There are 100,000+ die/tool designers and forming engineers worldwide.

---

## Target Users

### Primary Personas

**1. Wei — Die Designer at an Automotive Stamping Supplier**
- Designs progressive and transfer dies for automotive body panels
- Currently does die tryout without simulation; 6-8 iterations to get acceptable parts
- Needs: forming simulation to predict splits, wrinkles, and springback before cutting steel

**2. Sandra — Forming Engineer at an Aerospace Company**
- Forms titanium and Inconel components using hot forming and superplastic forming
- Uses DEFORM but simulations take 24-48 hours on her workstation
- Needs: cloud-accelerated forming simulation for high-temperature alloys with microstructure prediction

**3. Kenji — Process Engineer at a Consumer Electronics Company**
- Designs stamping processes for aluminum phone/laptop housings
- Needs springback prediction for tight dimensional tolerances (±0.1mm)
- Needs: accurate springback compensation with die morphing capability

---

## Solution Overview

FormSim is a cloud-based metal forming platform that:
1. Imports die face geometry (STEP, IGES) and blank outline, automatically generates forming tools (punch, die, binder, pad)
2. Runs explicit dynamic FEA simulation of the forming process with automatic contact, friction, and draw bead modeling
3. Predicts formability: thinning, forming limit diagram (FLD) comparison, wrinkling, splits
4. Computes springback and recommends die compensation geometry
5. Supports sheet metal stamping, hydroforming, hot stamping, forging (closed/open die), rolling, and extrusion

---

## Core Features

### F1: Die Face Setup
- CAD import: STEP, IGES, Parasolid for die faces (punch, die, binder)
- Automatic tool mesh generation from die face geometry
- Blank definition: shape, thickness, material, rolling direction
- Process definition: closing, drawing, trimming, flanging, hemming, springback
- Draw beads: line beads with equivalent restraining force model
- Blank holder force: constant, variable (with cushion pins), servo press profiles
- Progressive die setup: automatic station definition and strip layout
- Transfer die setup: multi-operation sequencing

### F2: Material Models
- Sheet metals: isotropic (von Mises), anisotropic (Hill48, Barlat Yld2000-2d, BBC2005)
- Forming limit curves (FLC): Keeler-Brazier, experimental import, stress-based FLC
- Rate-dependent plasticity for hot stamping and high-speed forming
- Temperature-dependent properties for hot forming
- Material database: DC04, DP600, DP780, DP980, TRIP780, AA5182, AA6016, AZ31 (Mg), Ti-6Al-4V, Inconel 718, and 300+ more
- Material card generator: input tensile test data → automatic curve fitting
- Advanced hardening: combined isotropic-kinematic (Chaboche) for springback accuracy
- GISSMO damage model for fracture prediction

### F3: Forming Simulation
- Explicit dynamic FEA with shell elements (Belytschko-Tsay, fully integrated)
- Adaptive mesh refinement during simulation
- Contact: penalty-based, automatic surface detection
- Friction: Coulomb, pressure-dependent, lubrication models
- Gravity loading and initial tool positioning
- Multi-stage: sequential operations (draw → trim → flange → springback)
- Incremental forming simulation (SPIF/TPIF)
- Hydroforming: tube and sheet, with fluid pressure boundary

### F4: Springback and Compensation
- Implicit static springback analysis after forming
- Springback visualization: displacement magnitude, angular deviation
- Die compensation: automatically morph die geometry to compensate for springback
- Multi-iteration compensation: simulate → compensate → re-simulate until convergence
- Dimensional conformance check against nominal geometry (GD&T-aware)
- Overbend and overcrown strategies
- Trim line development: predict trimmed edge location after springback

### F5: Bulk Forming (Forging, Extrusion, Rolling)
- Closed die forging: flash, flashless, and precision forging
- Open die forging: cogging, upset, drawing out
- Ring rolling
- Extrusion: direct, indirect, hydrostatic
- Hot forming: temperature-dependent flow stress, die preheating, cooling
- Microstructure: dynamic recrystallization, grain size prediction (JMAK)
- Die stress and fatigue life estimation
- Preform design optimization
- Billet heating and heat loss during transfer

### F6: Results and Optimization
- Thinning and thickening contours
- FLD overlay: compare strain state to forming limit curve
- Wrinkle indicator (compressive minor strain)
- Blank shape optimization: minimum blank size that avoids splits and wrinkles
- Binder force optimization: find optimal BHF trajectory
- Draw bead optimization: location and restraining force
- Cycle time estimation
- PDF report with formability assessment, springback, and tooling recommendations

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | TypeScript, React, Three.js/WebGPU (3D forming viz, FLD plots) |
| Explicit FEA | Rust + CUDA (explicit dynamic shell FEA, GPU-accelerated) |
| Implicit Springback | Rust (static implicit FEA) → server |
| Material Models | Rust (anisotropic yield functions, hardening laws) |
| Die Compensation | Rust (surface morphing algorithms) |
| Optimization | Python (GA for blank shape, BHF optimization) |
| Backend | Rust (Axum) + Python (FastAPI for material fitting, reporting) |
| Compute | AWS (GPU for explicit FEA, CPU for implicit springback) |
| Database | PostgreSQL (materials, FLC data, die templates), S3 (meshes, results) |

---

## Monetization

### Pro — $199/month
- Sheet metal stamping (single-stage)
- Material database (50 common grades)
- Thinning, FLD, wrinkling prediction
- Basic springback analysis
- 5 simulations/day

### Advanced — $449/user/month
- Multi-stage forming
- Springback compensation
- All material models (advanced anisotropic)
- Hot stamping
- Bulk forming (forging, extrusion)
- Unlimited simulations
- API access

### Enterprise — Custom
- On-premise deployment
- Custom material characterization
- Integration with die design CAD systems
- Digital twin for production stamping
- Training and certification

---

## MVP Scope (v1.0)

### In Scope
- STEP import for punch/die/binder geometry
- Single-stage draw simulation (explicit)
- 10 common steel/aluminum materials with Hill48 anisotropy
- Thinning contour and FLD comparison
- Springback prediction (implicit)
- 3D result visualization
- Basic formability report

### Out of Scope (v1.1+)
- Multi-stage forming
- Die compensation
- Hot stamping
- Forging/extrusion
- Advanced anisotropic models
- Optimization

### MVP Timeline: 14-18 weeks
