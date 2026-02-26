# PackSim — Packaging Engineering Simulation Platform

## Executive Summary

PackSim is a cloud-based platform for packaging design simulation, covering drop test analysis, compression/crush testing, vibration fatigue, pallet optimization, cushion curve engineering, and thermal insulation design. It replaces Cape Pack ($5K+/seat for pallet optimization), TOPS ($3K+/seat), ANSYS/LS-DYNA ($20K+/seat for drop test FEA), and Esko ArtiosCAD ($8K+/seat) with a unified browser-based environment purpose-built for packaging engineers.

---

## Problem Statement

**The pain:**
- Drop test simulation using general-purpose FEA (ANSYS Explicit, LS-DYNA, Abaqus) costs $20,000-$50,000/seat and requires specialized nonlinear dynamics expertise that most packaging engineers do not have
- Physical drop testing costs $5,000-$20,000 per test campaign (prototype fabrication, lab time, instrumentation) and takes 4-8 weeks; a single product may require 20-50 drops across orientations and heights
- Pallet optimization tools (Cape Pack, TOPS) handle cube utilization but do not simulate structural integrity — pallets collapse in transit because stacking strength was never verified
- The packaging industry generates $1 trillion annually but engineering decisions (material selection, cushion design, box size) are still dominated by rules of thumb and ISTA test-and-fail cycles
- E-commerce growth (20%+ CAGR) is creating unprecedented demand for right-sized packaging that minimizes void fill, reduces shipping cost, and survives automated sortation (drops, conveyor transitions, impacts)

**Current workarounds:**
- Using ASTM D4169 and ISTA test protocols with physical prototypes: build 20 samples, destroy them in a lab, iterate — costs $10K-$50K and 8-12 weeks per packaging design
- Cushion curve selection from foam manufacturer data sheets using static loading and fragility (G-factor) without dynamic simulation validation
- Pallet stacking patterns determined by Cape Pack or manual Tetris-like arrangement with no structural verification
- Using spreadsheets with McKee formula for corrugated box compression strength, ignoring moisture, creep, and dynamic loading effects

**Market size:** The global packaging market is $1.1 trillion (2024). Packaging testing and simulation services are approximately $2 billion. There are 200,000+ packaging engineers and logistics engineers globally. E-commerce packaging alone is a $60 billion segment growing at 15% CAGR.

---

## Target Users

### Primary Personas

**1. Derek — Packaging Engineer at a Consumer Electronics Company**
- Designs protective packaging for laptops, phones, and appliances that must survive a 30" drop onto concrete (ISTA 3A)
- Currently builds physical prototypes and tests them in a drop lab; 3-5 design iterations per product
- Needs: virtual drop test simulation to predict G-forces on the product, optimize cushion geometry, and reduce physical test iterations by 70%

**2. Aisha — Logistics Packaging Manager at an E-Commerce Company**
- Responsible for right-sizing packaging across 50,000+ SKUs to reduce dim-weight shipping costs and damage rates
- Uses Cape Pack for box sizing and pallet optimization but has no structural simulation capability
- Needs: automated box size recommendation, void fill minimization, and pallet stacking strength verification across the full supply chain vibration and compression environment

**3. Marco — Packaging Development Engineer at a Food and Beverage Company**
- Designs corrugated shippers and displays that must withstand warehouse stacking (10-high), humid environments (90% RH), and 14-day creep loading
- Uses McKee formula spreadsheets and physical compression testing
- Needs: corrugated box compression simulation with moisture/creep effects, cold chain thermal simulation, and pallet pattern optimization with stacking strength verification

---

## Solution Overview

PackSim is a cloud-based packaging simulation platform that:
1. Simulates drop tests virtually: import product CAD and packaging geometry, define drop height/surface/orientation, compute G-forces, stress, deformation, and cushion performance using explicit dynamic FEA
2. Predicts corrugated box compression strength (BCT) accounting for flute type, board grade, moisture content, creep duration, vent holes, and printing coverage — replacing McKee/simplified formulas with FEA accuracy
3. Optimizes cushion design: given product fragility (G-factor) and drop height, recommends optimal foam type, density, thickness, and geometry from cushion curve analysis and dynamic simulation
4. Plans pallet loads: box arrangement, stacking pattern, stretch wrap, and load stability — with compression and vibration simulation for transit survival
5. Runs transport vibration simulation: random vibration PSD profiles (ISTA, ASTM D4169) applied to packaged product to predict fatigue damage and content migration

---

## Core Features

### F1: Drop Test Simulation
- Explicit dynamic FEA: product + cushion + container assembly dropped from specified height onto rigid or compliant surface
- Impact orientations: flat face, edge, corner, and custom angle per ISTA/ASTM protocols
- Material models: viscoelastic foam (Ogden, hyperfoam, crushable foam), corrugated board (orthotropic), EPS/EPP/EPE, air cushion/bubble wrap
- Product fragility input: damage boundary curve (G vs. velocity change) or critical acceleration threshold
- G-force extraction: peak G at product CG and critical component locations
- Cushion bottoming-out detection: identifies insufficient cushion thickness
- Multi-drop sequence simulation: cumulative damage from sequential drops per ISTA 3A/3E
- Contact and friction: product-cushion, cushion-box, box-surface interactions
- Automatic mesh generation from STEP/STL imports with adaptive refinement at impact zones

### F2: Cushion Design and Optimization
- Cushion curve database: PE foam, PU foam, EPS, EPP, EPE, molded pulp, corrugated inserts — with density variants (1.0 to 6.0 lb/ft3)
- Static loading analysis: product weight / bearing area → static stress → optimal foam density from cushion curves
- Dynamic cushion curve generation from FEA simulation (beyond manufacturer static data)
- Cushion geometry optimization: corner blocks, full wrap, L-shaped, end caps — automatic shape generation given product geometry and void constraints
- Bruise prevention for produce/fragile goods: pressure distribution mapping on product surface
- Creep analysis: long-duration static loading during warehouse storage
- Temperature-dependent foam properties: cold chain performance (-20C to +60C)
- Sustainability scoring: material mass, recyclability rating, bio-based alternatives (mushroom, starch-based)

### F3: Corrugated Box Simulation
- Box compression test (BCT) prediction: short-column ECT method, McKee formula, and full FEA
- Board properties: ECT, flat crush, flexural stiffness for single-wall, double-wall, and triple-wall constructions
- Flute database: A, B, C, E, F, BC, EB, and custom flute profiles with liner/medium grammage
- Moisture derating: BCT reduction as a function of relative humidity (Kellicutt-Landt model + FEA)
- Creep derating: time-dependent BCT reduction under constant load (power law creep model)
- Vent/hand hole effects: BCT reduction from cutouts, printing coverage (flexo/litho weakening)
- Box style library: RSC, HSC, FOL, die-cut trays, telescoping, and FEFCO code catalog (200+ styles)
- Stacking analysis: number of layers × unit load weight vs. derated BCT with safety factor
- Buckling mode visualization: show which panel fails first and failure mode (face, corner, edge)

### F4: Pallet Optimization and Load Stability
- Pallet pattern generator: maximize cube utilization for given box dimensions and pallet size (GMA, EUR, custom)
- Column vs. interlocking stacking: trade-off between compression strength and stability
- Stretch wrap / strapping modeling: containment force, film layers, band tension
- Pallet deck deflection: load distribution across top/bottom deck boards
- Dynamic load stability: simulate truck/rail vibration and cornering forces on unitized load
- Load overhang and underhang analysis with pallet edge compression
- Mixed-SKU pallet building: optimize arrangement for multiple box sizes
- Weight and height constraints: max trailer height, axle weight limits
- Automated pallet pattern recommendation based on box dimensions, weight, and fragility

### F5: Transport Vibration Simulation
- Random vibration PSD input: ISTA pre-shipment profiles, ASTM D4169 assurance levels, custom PSD from truck/rail/air/ship measurements
- Frequency response of packaged product: resonance identification, transmissibility through cushion/container
- Fatigue damage calculation: Miner's rule cumulative damage from vibration exposure duration
- Content settling and migration: predict headspace increase for powders, granules, and liquids
- Resonance dwell testing simulation: extended exposure at critical frequencies
- Combined environment: vibration + top load (stacking) simultaneously
- Vibration isolation design: recommend cushion properties to detune product resonances from transport PSD peak frequencies
- Shock and vibration combined: programmed pulse (half-sine, trapezoidal) for conveyor transitions and sorting impacts

### F6: Thermal Packaging and Reporting
- Thermal insulation simulation: EPS/PU/VIP insulated shippers for cold chain (pharmaceuticals, food, biologics)
- Phase change material (PCM) modeling: gel packs, dry ice, liquid nitrogen — melt/evaporation curve and payload temperature vs. time
- Ambient temperature profiles: ISTA 7D/7E summer/winter profiles, custom city-pair routes
- Payload temperature excursion prediction: time above/below threshold (2-8C, -20C, -70C)
- Insulation thickness optimization: minimize wall thickness (maximize payload volume) while meeting temperature hold requirements
- Qualified shipper duration: hours/days of protection under specified ambient profile
- ISTA and ASTM protocol automation: configure test sequences (drop, vibration, compression, temperature) and run virtual qualification
- Report generation: pass/fail per ISTA/ASTM, G-force summary, BCT safety factor, pallet utilization, thermal hold time
- PDF and PowerPoint export with 3D result screenshots, comparison tables, and go/no-go recommendation

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | TypeScript, React, Three.js/WebGPU (3D packaging visualization, drop animation, deformation contours) |
| Explicit FEA (Drop Test) | Rust + CUDA (explicit dynamic solver with contact, foam constitutive models, GPU-accelerated) |
| Compression/Creep | Rust (implicit FEA for box compression, creep modeling) → server-side |
| Vibration | Rust (frequency domain vibration, PSD response, Miner's rule fatigue) → WASM for interactive |
| Thermal | Rust (transient heat transfer with PCM phase change) → WASM + server |
| Pallet Optimizer | Rust (bin packing heuristics + constraint optimization) → WASM for interactive |
| AI/ML | Python (PyTorch — cushion recommendation, box size suggestion, damage prediction from historical data) |
| Backend | Rust (Axum) + Python (FastAPI for ML inference, ISTA protocol engine, reporting) |
| Database | PostgreSQL (materials, cushion curves, FEFCO styles, ISTA protocols), S3 (CAD files, simulation results) |
| Hosting | AWS (GPU instances for explicit FEA, standard for vibration/thermal/pallet optimization) |

---

## Monetization

### Free Tier (Student)
- Cushion curve lookup and static loading calculator
- McKee box compression estimate
- Pallet pattern generator (single SKU, 2D view)
- 3 projects max

### Pro — $149/month
- Drop test simulation (single drop, models up to 50K elements)
- Corrugated box compression FEA (single box style)
- Cushion optimization
- Transport vibration (single PSD profile)
- Pallet optimization with 3D visualization
- Report generation
- 10 projects

### Advanced — $329/user/month
- Multi-drop sequence simulation (full ISTA protocol)
- Unlimited model size (cloud HPC)
- Corrugated board with moisture/creep derating
- Thermal packaging simulation
- Mixed-SKU pallet optimization
- ISTA/ASTM protocol automation
- API access and collaborative editing
- Unlimited projects

### Enterprise — Custom
- On-premise deployment for CPG/pharma data security
- Custom material characterization and testing integration
- PLM/ERP integration (SAP, Oracle) for SKU data sync
- White-label reporting for contract packaging companies
- Dedicated compute and priority support

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | PackSim Advantage |
|-----------|-----------|------------|------------------|
| Cape Pack (Esko) | Industry standard pallet optimization | $5K/seat, pallet-only (no structural simulation), dated UI | Integrated pallet + structural + drop test simulation |
| TOPS Pro | Good pallet/truck loading | No drop test, no compression analysis, optimization-only | Full packaging engineering: design through qualification |
| ANSYS / LS-DYNA | Best-in-class explicit FEA for drop test | $20-50K/seat, general-purpose (not packaging-native), requires FEA expertise | Packaging-specific UI, no FEA expertise required, 10x cheaper |
| Esko ArtiosCAD | Strong structural design and die layout | $8K/seat, CAD-focused (no simulation), no drop/vibration testing | Simulation-native with packaging-specific physics |
| Physical test labs | Gold standard for qualification | $5-20K per campaign, 4-8 weeks, destructive, no optimization | Virtual testing: hours instead of weeks, iterate without destroying prototypes |

---

## MVP Scope (v1.0)

### In Scope
- STEP/STL import for product and packaging geometry
- Single flat-face drop test simulation with foam cushion (EPS, PE foam)
- G-force extraction at product CG
- Cushion curve lookup and static loading calculator
- Corrugated box compression estimate (McKee + basic FEA)
- Pallet pattern generator with cube utilization (single SKU)
- PDF report with G-force results, BCT estimate, and pallet layout

### Out of Scope (v1.1+)
- Multi-drop sequences and full ISTA protocol automation
- Transport vibration simulation
- Thermal packaging / cold chain simulation
- Corrugated moisture and creep derating
- Mixed-SKU pallet optimization
- AI-based cushion and box size recommendation

### MVP Timeline: 12-16 weeks
