# BridgeWorks — Bridge Design and Rating Platform

## Executive Summary

BridgeWorks is a cloud-based bridge engineering platform for structural design, load rating, seismic analysis, and inspection management. It replaces MIDAS Civil ($15K+/seat), CSiBridge ($8K+/seat), AASHTOWare BrR ($5K+/seat), and LARS Bridge ($3K+/seat) with an integrated browser-based environment covering the full bridge lifecycle from preliminary design through in-service load rating and inspection tracking.

---

## Problem Statement

**The pain:**
- MIDAS Civil costs $15,000-$25,000/seat/year and CSiBridge costs $8,000-$15,000/seat; most bridge engineering firms need both a general FEA tool and a code-specific design/rating tool, doubling license costs
- AASHTOWare BrR (formerly Virtis/Opis) is the de facto DOT standard for load rating but has a dated Windows-only interface, limited 3D modeling, and requires separate analysis tools for complex bridges
- Moving load analysis with AASHTO HL-93 or Eurocode LM1 loading requires specialized influence line/surface computation that general-purpose FEA tools handle poorly or require extensive manual setup
- Bridge inspection data (NBI, element-level) lives in separate systems (PONTIS/AASHTOWare BrM) disconnected from structural models, forcing engineers to manually cross-reference condition states with load rating capacity
- Prestressed concrete bridge design requires tracking time-dependent losses (creep, shrinkage, relaxation) across construction stages — a workflow that is error-prone across multiple tools

**Current workarounds:**
- Using PGSuper (free, WSDOT) for prestressed girder design but it only covers standard WSDOT shapes and limited span configurations
- Building separate models in CSiBridge (3D FEA) and AASHTOWare BrR (line-girder rating) and manually reconciling results
- Tracking inspection findings in spreadsheets or legacy databases with no link to structural capacity calculations
- Using LEAP Bridge (Bentley) for substructure design and a different tool for superstructure, with manual load transfer between them

**Market size:** The bridge engineering software market is approximately $1.8 billion (2024), part of the broader $12 billion infrastructure engineering software sector. There are an estimated 80,000+ bridge engineers worldwide, with 45,000+ in the US alone (where 620,000+ bridges require periodic inspection and rating). US states and agencies spend $150B+ annually on bridge maintenance and construction.

---

## Target Users

### Primary Personas

**1. Sarah — Bridge Design Engineer at a Consulting Firm**
- Designs new highway bridges (prestressed concrete girders, steel plate girders, curved steel) for state DOT projects
- Currently uses CSiBridge for 3D analysis and PGSuper for prestressed design, manually transferring loads between tools
- Needs: integrated design workflow from preliminary layout through final plans with AASHTO LRFD compliance, staged construction analysis, and automated design optimization

**2. Marcus — DOT Bridge Load Rating Engineer**
- Rates existing bridges for legal loads, permit vehicles, and emergency vehicles across a state inventory of 5,000+ structures
- Uses AASHTOWare BrR but struggles with complex bridges (curved, skewed, integral abutments) that exceed line-girder assumptions
- Needs: load rating platform that handles both simple line-girder and refined 3D analysis, integrates with BrM inspection data, and batch-processes rating updates when new permit vehicles are introduced

**3. Dr. Yamamoto — Bridge Seismic Specialist**
- Performs seismic retrofit evaluations and designs for bridges in high seismic zones per AASHTO Guide Specs for LRFD Seismic Bridge Design
- Uses MIDAS Civil for pushover and time-history analysis but model setup for existing deteriorated bridges is extremely tedious
- Needs: seismic analysis integrated with condition-based capacity, pushover analysis, displacement-based design, and retrofit design tools (column jacketing, isolation bearings, restrainer cables)

---

## Solution Overview

BridgeWorks is a cloud-based bridge engineering platform that:
1. Provides parametric bridge modeling with standard superstructure types (precast girders, steel plate girders, box girders, trusses, arches, cable-stayed, suspension) and automatic section property computation
2. Performs influence line/surface-based moving load analysis for AASHTO HL-93, Eurocode LM1/LM2, and custom vehicle loads with automatic critical load positioning
3. Runs AASHTO LRFD design checks and load rating (RF computation) per MBE (Manual for Bridge Evaluation) with LFR, LRFR, and ASR methods
4. Handles time-dependent analysis for prestressed concrete including creep, shrinkage, relaxation, and staged construction (CEB-FIP, AASHTO, ACI 209)
5. Integrates bridge inspection data (element-level condition states) with structural models to compute condition-adjusted load ratings and prioritize maintenance actions

---

## Core Features

### F1: Bridge Modeling and Geometry
- Parametric bridge layout: span lengths, horizontal/vertical alignment, skew angles, superelevation, cross-slope
- Standard superstructure templates: precast I-girders (AASHTO, PCI, state-specific), steel plate girders, precast/CIP box girders, voided slabs, adjacent box beams
- Composite section builder: girder + deck + haunch with effective flange width per AASHTO 4.6.2.6
- Substructure modeling: pier caps, columns (single/multi-column bents), abutments (stub, cantilever, integral, semi-integral), pile foundations
- Cross-frame and diaphragm modeling with automatic placement per code requirements
- Bearing modeling: elastomeric, pot, spherical, PTFE sliding, seismic isolation (LRB, FPS, HDR)
- Cable-stayed bridge module: cable geometry, backstay arrangement, pylon modeling, cable tensioning optimization
- Suspension bridge module: main cable catenary, suspender layout, stiffening girder/truss
- CAD/BIM import: IFC bridge extensions, DXF, LandXML for alignment data

### F2: Moving Load Analysis and Load Rating
- Influence line computation for moment, shear, reaction, and deflection at any section
- Influence surface computation for 2D/3D models (slab bridges, box girder decks)
- AASHTO HL-93 loading: design truck (HS20), design tandem, lane load with automatic critical placement using influence lines
- AASHTO fatigue truck, permit vehicles (SU4, SU5, SU6, SU7), emergency vehicles, state-specific legal loads
- Eurocode LM1 (UDL + tandem system), LM2 (single axle), LM3 (special vehicles), LM4 (crowd loading)
- Multiple presence factors and dynamic load allowance per code
- Load rating factor (RF) computation per AASHTO MBE: Inventory, Operating, Legal, Permit levels
- LRFR (Load and Resistance Factor Rating), LFR (Load Factor Rating), and ASR (Allowable Stress Rating) methods
- Batch rating: rate entire bridge inventory against new vehicle configurations in one run
- Posting analysis: determine safe posting loads when RF < 1.0 per MBE Section 6

### F3: Prestressed Concrete Design
- Pretensioned and post-tensioned strand layout with automatic eccentricity computation
- Time-dependent loss calculations: elastic shortening, creep, shrinkage, steel relaxation per AASHTO 5.9.3 (refined and approximate methods)
- CEB-FIP Model Code 2010 and ACI 209 creep/shrinkage models
- Staged construction analysis: strand release, girder erection, deck pour (wet/dry), barrier placement, overlay, time to service
- Service limit state checks: tensile/compressive stress limits at release, service, and intermediate stages
- Strength limit state: Mn, Vn (modified compression field theory per AASHTO 5.7), interface shear (horizontal shear)
- Deflection and camber prediction tracking construction stages and time-dependent effects
- Debonding and harping of pretensioned strands with code compliance checks
- Post-tensioning tendon profile optimization with friction and wobble loss calculation
- Continuity for live load: positive and negative moment connections for simple-span-made-continuous bridges

### F4: Steel Bridge Design
- Plate girder design: proportioning flanges, web, stiffeners per AASHTO 6.10 (I-sections) and 6.11 (box sections)
- Constructibility checks: flange lateral bending, deck casting stresses, non-composite dead load
- Fatigue and fracture: fatigue category classification, infinite life check, finite life evaluation per AASHTO 6.6
- Shear design: end panels, intermediate stiffeners, bearing stiffeners, tension field action
- Curved steel girder analysis: V-load method and 3D refined analysis with warping torsion
- Cross-frame forces from girder differential deflection and lateral bracing
- Splice design: bolted field splices per AASHTO 6.13
- Composite action: shear connector design (studs, channels) per AASHTO 6.10.10
- Corrosion allowance and condition-adjusted capacity for existing steel bridges

### F5: Seismic Analysis
- Seismic hazard: site-specific spectra from USGS 2024 National Seismic Hazard Model or user-defined spectra
- Single-mode (uniform load) and multi-mode spectral analysis per AASHTO Guide Spec
- Pushover analysis: plastic hinge models for concrete columns (fiber section, lumped plasticity)
- Displacement-based seismic design: demand vs. capacity displacement comparison
- Seismic isolation design: LRB, FPS, and HDR bearing sizing with bilinear hysteresis modeling
- Liquefaction assessment: lateral spreading and settlement effects on bridge foundations
- Abutment backwall passive resistance and gap modeling
- Foundation flexibility: p-y curves for piles, translational/rotational springs for spread footings
- Nonlinear time-history analysis with ground motion suite (cloud-computed)
- Retrofit evaluation: column jacketing (steel, FRP, concrete), restrainer cables, seat extenders, shear key strengthening

### F6: Inspection Integration and Asset Management
- NBI (National Bridge Inventory) data import and element-level condition state tracking per AASHTO CoRe elements
- Photo/video annotation linked to specific bridge elements and structural model locations
- Condition-adjusted load rating: reduce section properties based on observed deterioration (section loss, spalling, delamination)
- Deterioration modeling: Markov chain transition probabilities for element condition projection
- Maintenance action recommendation: repair, rehabilitate, or replace based on condition trajectories and cost-benefit analysis
- Inspection scheduling: risk-based inspection intervals per FHWA requirements
- Integration with AASHTOWare BrM data format for DOT workflows
- Dashboard: bridge health index, sufficiency rating, structurally deficient classification, posting status

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | TypeScript, React, Three.js (3D bridge model), D3.js (influence lines, rating charts) |
| FEA Solver | Rust (frame/shell FEM, influence line computation, eigenvalue analysis) → server-side |
| Moving Load Engine | Rust (HL-93/LM1 moving load optimization, influence line/surface integration) → WASM + server |
| Prestress Engine | Rust (time-dependent creep/shrinkage, staged construction tracking) |
| Backend | Rust (Axum) + Python (FastAPI for inspection analytics, deterioration modeling) |
| AI/ML | Python (deterioration prediction, optimal maintenance scheduling, load rating anomaly detection) |
| Database | PostgreSQL (bridge inventory, NBI data, vehicle libraries, code provisions), PostGIS (bridge locations), S3 (inspection photos, 3D models, results) |
| Hosting | AWS (ECS for backend, auto-scaling for batch rating and seismic analysis, CloudFront for frontend) |

---

## Monetization

### Free Tier (Student)
- Single-span prestressed or steel girder bridge
- HL-93 design loading only
- LRFD design checks (no load rating)
- 3 projects

### Pro — $129/month
- Multi-span bridges up to 10 spans
- Full load rating (LRFR, LFR, ASR)
- Custom vehicle libraries
- Staged construction and time-dependent analysis
- Eurocode support
- Report generation (PDF)

### Advanced — $299/user/month
- Everything in Pro
- Unlimited spans and complex bridge types (curved, cable-stayed)
- Seismic analysis (spectral, pushover, time-history)
- 3D refined analysis (shell/solid models)
- Inspection integration and condition-adjusted rating
- Batch rating for bridge inventories
- API access

### Enterprise — Custom
- Full asset management integration (AASHTOWare BrM)
- DOT-wide deployment with bridge inventory management
- Custom vehicle and load configuration libraries
- On-premise or GovCloud deployment for state agencies
- Training, certification, and dedicated support

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | BridgeWorks Advantage |
|-----------|-----------|------------|-------------------|
| MIDAS Civil | Excellent 3D FEA, staged construction | $15-25K/seat, no load rating per MBE, no inspection integration | Integrated rating + FEA, 10x cheaper |
| CSiBridge | Good 3D analysis, bridge-specific UI | $8-15K/seat, limited load rating, no asset management | Full rating workflow, inspection integration |
| AASHTOWare BrR | DOT standard for load rating | Dated UI, line-girder only, no 3D FEA, no seismic | 3D analysis + rating, modern cloud UI |
| PGSuper (WSDOT) | Free, excellent for WA state girders | WSDOT shapes only, no rating, no steel, no seismic | All bridge types, national/international codes |
| LARS Bridge (Bentley) | Good load rating, Bentley ecosystem | $3-5K/seat, limited analysis, legacy interface | Modern UI, integrated FEA, cloud-based |

---

## MVP Scope (v1.0)

### In Scope
- Parametric bridge layout for simple and continuous span prestressed I-girder bridges
- Composite section property computation (girder + deck)
- Influence line-based moving load analysis for AASHTO HL-93
- AASHTO LRFD service and strength limit state checks for prestressed concrete
- Time-dependent prestress loss calculation (approximate and refined)
- Load rating factor computation (LRFR method) for inventory, operating, and legal loads
- PDF design/rating report generation

### Out of Scope (v1.1+)
- Steel bridge design and rating
- 3D refined analysis (shell/solid models)
- Seismic analysis
- Curved, skewed, and cable-stayed bridges
- Inspection data integration and condition-adjusted rating
- Eurocode and international codes

### MVP Timeline: 14-18 weeks
