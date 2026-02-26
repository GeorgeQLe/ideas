# BlastCalc — Explosion and Blast Effect Simulation Platform

## Executive Summary

BlastCalc is a cloud-based platform for simulating blast wave propagation, structural response to explosions, and protective design — combining empirical blast load methods (CONWEP/UFC 3-340-02) with high-fidelity CFD and coupled Lagrangian structural analysis. It replaces AUTODYN ($30K+/seat), LS-DYNA blast modules ($20K+/seat), and standalone tools like CONWEP and BlastX with an integrated browser-based environment for defense engineers, protective structure designers, and mining blast engineers.

---

## Problem Statement

**The pain:**
- AUTODYN (Ansys) costs $30,000-$60,000/seat/year; LS-DYNA with blast/ALE capabilities costs $20,000-$40,000/seat; these tools require months of training and expert-level knowledge to set up blast-structure interaction models
- Protective structure design (embassies, military installations, critical infrastructure) requires blast load calculation per UFC 3-340-02 and GSA criteria, but the underlying calculations are manual, error-prone, and rely on outdated 1960s-era charts
- CONWEP (US Army) is a free empirical tool but is restricted-distribution, command-line, handles only idealized geometries (no reflections, channeling, or complex terrain), and has not been substantially updated since the 1990s
- Fragment hazard assessment requires coupling blast overpressure with fragment trajectory, velocity decay, and penetration — currently done in separate disconnected tools
- Mining engineers need blast vibration prediction, flyrock risk assessment, and regulatory compliance but use separate tools for each with no integration

**Current workarounds:**
- Hand calculations using UFC 3-340-02 pressure-impulse charts and Kingery-Bulmash polynomials — tedious, limited to simple geometries, and error-prone for complex scenarios
- Running AUTODYN or LS-DYNA with ALE (Arbitrary Lagrangian-Eulerian) for CFD blast propagation, requiring weeks of model setup and days of compute time per scenario
- Using BlastX (ERDC) for internal detonations with room-to-room propagation, but it is restricted-access and empirically limited
- Separate structural analysis in SAP2000 or ETABS with manually applied blast loads, missing the fluid-structure coupling effects

**Market size:** Global defense spending exceeds $2.2 trillion/year, with protective structure design and weapons effects assessment representing a $3B+ market. The mining explosives market is $12B+/year with 10,000+ active mines worldwide. The blast-resistant design market (civilian critical infrastructure) is growing at 10% CAGR post-2020. There are 50,000+ engineers worldwide working in blast effects, protective design, and mining blasting.

---

## Target Users

### Primary Personas

**1. Major Chen — Protective Structure Engineer at a Defense Agency**
- Designs blast-resistant facilities (command bunkers, embassy buildings, ammunition storage) per UFC 3-340-02 and ASCE 59-11
- Spends weeks performing manual P-I (pressure-impulse) calculations and structural response checks for each threat scenario
- Needs: fast blast load generation for complex facility geometries with automated structural response assessment against design criteria

**2. Dr. Hoffman — Weapons Effects Analyst at a Defense Contractor**
- Evaluates warhead effectiveness, fragmentation patterns, and structural damage for weapons systems development
- Uses AUTODYN and LS-DYNA but spends months setting up each coupled blast-structure model
- Needs: rapid scenario setup for blast propagation in urban terrain with coupled structural damage prediction and fragment tracking

**3. Raj — Senior Blasting Engineer at a Mining Company**
- Designs drill-and-blast patterns for open-pit and underground mining operations
- Must predict ground vibration (PPV), air overpressure, flyrock range, and fragmentation for regulatory compliance and safety
- Needs: blast design tool that predicts vibration, airblast, flyrock, and fragmentation with regulatory compliance reporting

---

## Solution Overview

BlastCalc is a cloud-based blast simulation platform that:
1. Calculates blast loads (overpressure, impulse, reflected pressure, dynamic pressure) using empirical methods (Kingery-Bulmash, UFC 3-340-02) for rapid parametric studies and design-level analysis
2. Runs high-fidelity CFD blast propagation using Euler solver with AMR (adaptive mesh refinement) for complex geometries including urban terrain, tunnels, and multi-room buildings
3. Performs coupled fluid-structure interaction (FSI) with Lagrangian structural response to predict wall breach, structural collapse, and component damage
4. Tracks fragments (primary and secondary) with aerodynamic drag, ricochet, and penetration into target materials using empirical and analytical penetration equations
5. Supports mining blast design with drill pattern optimization, vibration prediction (scaled distance laws), airblast compliance, flyrock range estimation, and fragmentation analysis (Kuz-Ram model)

---

## Core Features

### F1: Empirical Blast Load Engine
- Kingery-Bulmash polynomials for free-air and surface burst: incident and reflected overpressure, impulse, positive phase duration, shock arrival time, dynamic pressure
- TNT equivalence library: 50+ explosive types (C4, PETN, ANFO, RDX, HMX, dynamite, emulsion) with conversion factors
- Hemispherical and spherical burst models
- Reflected pressure calculations: normal and oblique reflection with Mach stem formation
- UFC 3-340-02 automated lookup: pressure-impulse diagrams, clearing time corrections, drag loading on open-frame structures
- Multiple detonation points with time-delayed sequencing
- Blast load time-history generation: Friedlander waveform with negative phase for structural analysis input
- Confined explosion: internal detonation with quasi-static gas pressure buildup (BlastX-equivalent methodology)

### F2: CFD Blast Propagation
- Euler solver with adaptive mesh refinement (AMR) for shock-capturing without excessive mesh requirements
- JWL (Jones-Wilkins-Lee) equation of state for detonation product gases
- Chapman-Jouguet detonation model for initiation and detonation wave propagation
- Multi-material Euler: air, explosive products, soil/water for buried charges and underwater explosions
- Urban terrain blast propagation: channeling between buildings, diffraction around corners, reflection amplification
- Tunnel and corridor blast: quasi-1D and full 3D propagation in confined spaces
- Multi-room blast: venting through doorways, windows, and breach openings with room-to-room pressure loading
- Ground shock: stress wave propagation in soil from buried detonations
- GPU-accelerated solver for real-time parametric sweeps at reduced fidelity

### F3: Structural Response Analysis
- Single-degree-of-freedom (SDOF) response: rapid assessment of wall panels, columns, beams to blast loading per UFC 3-340-02 methodology
- Pressure-impulse (P-I) diagrams: automatic generation for structural components with damage level contours
- Multi-degree-of-freedom (MDOF) structural models: frame buildings, portal frames with plastic hinge formation
- Full 3D Lagrangian FEA: reinforced concrete (smeared crack, damage plasticity), steel (Johnson-Cook, Cowper-Symonds strain rate), masonry (discrete element), glazing (brittle fracture)
- Strain-rate effects: dynamic increase factors (DIF) per UFC 3-340-02 for concrete and steel
- Progressive collapse: removal of failed members and load redistribution analysis
- Component damage levels: superficial, moderate, heavy, hazardous, blowout — mapped to ASCE 59-11 categories
- Breach prediction: concrete wall perforation and spall using empirical and FEA methods

### F4: Fragment and Projectile Analysis
- Primary fragmentation: Mott distribution for natural fragmentation of cased munitions (fragment count, mass, velocity)
- Gurney equations for initial fragment velocity from explosive-metal systems
- Fragment trajectory: ballistic integration with aerodynamic drag (shape-dependent drag coefficients)
- Secondary fragments: blast-accelerated debris from structural failure (glass, concrete spall, building components)
- Penetration equations: Thor, JTCG/ME for metal penetration; NDRC, modified Petry for concrete penetration; empirical models for soil and composite armor
- Behind-armor debris (BAD): residual velocity and spray cone prediction
- Fragment hazard zones: lethal area, incapacitation probability contours (Pk mapping)
- Glass fragment hazard: window breakage prediction (ASTM E1300) and glass fragment throw distance

### F5: Mining Blast Design
- Drill pattern design: burden, spacing, hole diameter, depth, stemming length for bench blasting
- Timing design: delay sequence optimization for fragmentation and vibration control (electronic detonator timing)
- Fragmentation prediction: Kuz-Ram model, Swebrec distribution, image-based calibration
- Ground vibration prediction: scaled distance law (USBM, Devine), site-specific attenuation curves, PPV at receiver locations
- Airblast prediction: overpressure at distance with terrain and atmospheric effects
- Flyrock prediction: Lundborg model, risk zone calculation, exclusion zone mapping
- Regulatory compliance: OSMRE (US), AS 2187 (Australia), BS 7385 (UK) vibration and airblast limits
- Wall control blasting: pre-split, smooth wall, trim blast design parameters
- Underground blasting: drift rounds, ring blasting, stope blast design

### F6: Scenario Management and Reporting
- Threat library: vehicle-borne IED (VBIED) sizes per FEMA 426/GSA criteria, person-borne IED (PBIED), military munitions
- Standoff distance calculator: safe standoff for given threat size and building type
- Site layout tool: import GIS/CAD site plan, place buildings, barriers, threats on map
- Vulnerability assessment: evaluate existing structures against defined threat scenarios
- Risk assessment framework: threat-vulnerability-consequence matrix
- Compliance checking: automatic assessment against UFC 4-010-01, ASCE 59-11, GSA blast criteria, ISC security design criteria
- Report generation: blast load summary, structural response, damage assessment, recommended protective measures — PDF export
- Blast damage overlay: GIS-compatible damage contour export for emergency response planning

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | TypeScript, React, Three.js/Deck.gl (3D blast visualization, pressure contours, fragment trajectories, GIS mapping) |
| Empirical Engine | Rust (Kingery-Bulmash, UFC 3-340-02 calculations, Friedlander waveforms) → WASM for client-side |
| CFD Solver | Rust + CUDA (Euler AMR solver, JWL EOS, multi-material) → GPU-accelerated |
| Structural Solver | Rust (SDOF/MDOF rapid solvers, full Lagrangian FEA with damage models) |
| Fragment Tracker | Rust (ballistic trajectory integration, penetration equations, Monte Carlo fragmentation) |
| Mining Module | Python (Kuz-Ram, scaled distance laws, timing optimization) |
| Backend | Rust (Axum) + Python (FastAPI for reporting, GIS integration) |
| Compute | AWS GovCloud (ITAR/EAR compliance, GPU for CFD, CPU for structural FEA) |
| Database | PostgreSQL (explosive properties, material models, threat library), S3 (results, animations) |

---

## Monetization

### Free Tier (Student)
- Empirical blast load calculator (single detonation, free-field)
- Kingery-Bulmash overpressure/impulse at distance
- SDOF structural response (single component)
- TNT equivalence calculator
- 3 projects

### Pro — $199/month
- Full empirical blast engine (reflected, confined, clearing corrections)
- SDOF/MDOF structural response with damage assessment
- Fragment trajectory and penetration analysis
- P-I diagram generation
- UFC 3-340-02 compliance checking
- Mining blast design (Kuz-Ram, vibration prediction)
- Report generation
- Unlimited projects

### Advanced — $499/user/month
- CFD blast propagation (Euler AMR solver)
- Coupled fluid-structure interaction
- Full 3D structural FEA with damage models
- Urban terrain and multi-room blast scenarios
- Progressive collapse analysis
- Advanced fragment analysis (Mott, Gurney, BAD)
- Mining timing optimization
- API access
- Team collaboration

### Enterprise — Custom
- GovCloud deployment (ITAR/EAR/CUI compliance)
- On-premise / air-gapped installation for classified work
- Custom explosive characterization (JWL EOS fitting)
- Integration with force protection planning tools
- Dedicated support with blast engineering SMEs
- Custom threat library and vulnerability databases

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | BlastCalc Advantage |
|-----------|-----------|------------|---------------------|
| AUTODYN (Ansys) | Gold standard coupled Euler-Lagrangian, trusted for defense | $30-60K/seat, months of training, desktop only | 10x faster setup, browser-based, empirical + CFD in one tool |
| LS-DYNA (Ansys/LST) | Excellent structural response, ALE blast, massive user base | $20-40K/seat, complex input deck, no built-in empirical methods | Integrated empirical + CFD workflow, guided setup for non-experts |
| CONWEP (US Army) | Free, trusted empirical calculations, standard reference | Command-line, restricted distribution, simple geometries only | Modern GUI, complex geometries, CFD for non-ideal scenarios |
| BlastX (ERDC) | Internal detonation with room-to-room propagation | Restricted access, empirical only, no structural coupling | Open access, CFD + structural coupling, broader scenario coverage |
| WAI SBEDS | Excellent SDOF for component response per UFC | Spreadsheet-based, no blast load generation, no spatial analysis | Full blast-to-structure workflow, 3D visualization, automation |

---

## MVP Scope (v1.0)

### In Scope
- Empirical blast load calculator: Kingery-Bulmash for free-air and surface burst (incident + reflected)
- TNT equivalence for 20 common explosives
- Friedlander waveform generation for structural analysis input
- SDOF structural response for RC walls, steel beams, and glass panels
- Pressure-impulse diagram generation with damage level assessment
- Standoff distance calculator per GSA/UFC criteria
- Basic fragment velocity and trajectory (Gurney + ballistic drag)
- Web-based scenario setup with 3D visualization of blast pressure contours
- PDF report with blast loads, structural response, and compliance summary

### Out of Scope (v1.1+)
- CFD blast propagation (Euler AMR)
- Coupled fluid-structure interaction
- Full 3D Lagrangian structural FEA
- Multi-room and tunnel blast scenarios
- Mining blast design module
- Progressive collapse analysis
- GovCloud / classified deployment

### MVP Timeline: 14-18 weeks
