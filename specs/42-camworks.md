# CAMWorks — AI-Powered CNC Toolpath Generation Platform

## Executive Summary

CAMWorks is a cloud-based Computer-Aided Manufacturing platform that automatically recognizes machinable features from 3D CAD models and generates optimized CNC toolpaths using AI. It competes with Mastercam, Fusion 360 CAM, and Siemens NX CAM by offering browser-based access, automatic feature recognition, and machine-learning-optimized cutting strategies at a fraction of the cost.

---

## Problem Statement

**The pain:**
- Mastercam costs $10,000-$30,000 per seat depending on axes and modules, with $3,000+ annual maintenance — pricing that locks out small job shops and contract manufacturers
- CAM programming is a bottleneck: skilled CNC programmers are scarce (median age 52, US Bureau of Labor Statistics), and training takes 6-18 months
- Manual toolpath creation for a complex 5-axis aerospace part can take 8-40 hours of programming time per part
- Post-processor configuration is a nightmare — each CNC machine brand (Haas, Mazak, DMG Mori, Fanuc, Siemens) needs custom post-processors that cost $500-$2,000 each
- Toolpath simulation and collision detection are often separate, expensive add-ons (Vericut costs $15,000+/seat)

**Current workarounds:**
- Small shops use Fusion 360's CAM module but hit limitations on 5-axis, multi-setup, and production-grade post-processors
- Some shops rely on conversational programming at the machine controller, losing repeatability and optimization
- Experienced machinists hoard tribal knowledge about feeds, speeds, and cutting strategies that is lost when they retire
- Free/open-source CAM (FreeCAD Path, PyCAM) lacks feature recognition and produces unreliable toolpaths

**Market size:** The global CAM software market is approximately $3.5 billion (2024), projected to reach $5.8 billion by 2030. There are over 300,000 machine shops worldwide, with 90,000+ in the US alone. The average shop has 3-15 CNC machines but only 1-3 programmers.

---

## Target Users

### Primary Personas

**1. Mike — Job Shop Owner (15 CNC machines, 8 machinists)**
- Gets 20-50 quote requests per week, needs to quickly estimate machining time and cost
- Currently uses Mastercam but only 2 of his 8 machinists can program it
- Needs: automatic toolpath generation so more team members can program parts, and instant quoting from CAD models

**2. Sandra — Aerospace Manufacturing Engineer**
- Programs 5-axis parts for turbine blades and structural components with tight tolerances (±0.001")
- Uses Siemens NX CAM but spends days on each complex part and worries about collisions
- Needs: AI-optimized 5-axis strategies with built-in machine simulation and collision avoidance

**3. Diego — Contract Manufacturer / Prototyping Service**
- Runs a rapid prototyping shop with Haas VMCs; programs parts from customer STEP files
- Currently uses Fusion 360 CAM but needs better post-processors and production features
- Needs: fast STEP-to-G-code pipeline with reliable post-processors for his specific Haas machines

### Secondary Personas
- CNC hobbyists with desktop mills (Tormach, Shapeoko) needing simple 2.5D CAM
- Manufacturing engineering students learning CNC programming
- Production planners needing cycle time estimates for scheduling

---

## Solution Overview

CAMWorks is a browser-based CAM platform that:
1. Imports STEP/IGES/Parasolid files and automatically recognizes machinable features (pockets, holes, slots, bosses, contours, freeform surfaces) using a computer vision ML model
2. Generates optimized toolpaths for 2.5D, 3-axis, 3+2, and 5-axis simultaneous milling using AI-trained cutting strategies
3. Recommends tools, feeds, and speeds based on material, machine capability, and a database of proven cutting parameters from thousands of real-world jobs
4. Simulates toolpaths with full machine kinematics (virtual CNC machine model) for collision detection and cycle time estimation
5. Outputs production-ready G-code through a library of 500+ verified post-processors for major CNC controller brands

---

## Core Features

### F1: Automatic Feature Recognition (AFR)
- ML-powered recognition of prismatic features: through/blind holes, counterbores, countersinks, pockets (open/closed), slots, steps, chamfers, fillets
- 3D surface feature detection: ruled surfaces, freeform surfaces, blends, undercuts
- Thread detection with automatic tap/thread mill selection
- Feature grouping by operation type and machining direction
- Manual feature override and user-defined feature templates
- Feature-based cost estimation (material removal volume × removal rate)

### F2: AI Toolpath Generation
- Adaptive clearing (high-efficiency roughing with constant chip load) for pockets and 3D roughing
- Rest machining: automatically detect and re-machine remaining material with smaller tools
- Contour finishing with scallop height control
- Pencil tracing for corner cleanup
- 5-axis swarf cutting, flow-line machining, and multi-axis contouring
- Automatic lead-in/lead-out and linking strategy optimization
- AI optimization: ML model trained on 100K+ successful toolpaths to select strategies, step-overs, and cutting depths
- One-click "optimize for time" or "optimize for surface finish" modes

### F3: Tool and Feeds/Speeds Database
- Comprehensive tool library: end mills, ball nose, bull nose, face mills, drills, taps, reamers, boring bars, inserts
- Material database: 500+ materials with recommended cutting parameters from tool manufacturer catalogs
- AI-recommended feeds and speeds based on material, tool, machine rigidity, and depth of cut
- Tool wear prediction and automatic tool life tracking
- Custom tool definitions with holder geometry for collision checking
- Integration with tool vendors (Kennametal, Sandvik, Mitsubishi) for real-time tool data

### F4: Machine Simulation
- Full kinematic machine simulation with accurate 3D models of 200+ CNC machines
- Real-time collision detection: tool/holder vs. part, fixture, machine structure
- Material removal simulation with remaining stock visualization
- Cycle time estimation with acceleration/deceleration modeling
- G-code verification: step through generated code line-by-line with synchronized 3D animation
- Automatic gouging detection on finished surfaces

### F5: Post-Processor Library
- 500+ verified post-processors for Fanuc, Siemens, Haas, Mazak, Heidenhain, Okuma, Brother, Mitsubishi, and more
- Post-processor customization editor (modify output format without coding)
- Community-contributed post library with ratings and machine-specific notes
- Automatic post-processor recommendation based on machine model selection
- Multi-setup output with tool lists, setup sheets, and operation sequencing

### F6: Quoting and Production
- Instant machining time and cost estimates from uploaded CAD models
- Material cost calculation with stock size optimization (nesting for bar stock)
- Fixture planning with standard vise/clamp library
- Setup sheet generation (PDF) with operation descriptions, tool lists, and datum references
- Integration with ERP/MES systems via API

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | TypeScript, React, Three.js/WebGPU for 3D simulation |
| CAM Kernel | Rust (toolpath generation, collision detection) → WASM |
| Feature Recognition | Python, PyTorch (3D CNN on voxelized/mesh geometry) |
| Machine Simulation | Rust (kinematics engine) + WebGPU rendering |
| Post-Processing | Lua-based customizable post-processor engine |
| Backend | Rust (Axum), Python (FastAPI for ML services) |
| Database | PostgreSQL (jobs, tools, machines), S3 (CAD files, G-code) |
| Hosting | AWS (GPU instances for heavy simulation, ECS for API) |

---

## Monetization

### Free Tier (Hobbyist)
- 2.5D toolpath generation only (pockets, drilling, contouring)
- 3 machines max
- Basic post-processors (generic Fanuc, generic Grbl)
- Community support

### Pro — $99/month
- 3-axis and 3+2 axis toolpaths
- Automatic feature recognition
- Full machine simulation
- 100+ post-processors
- AI feeds/speeds recommendations

### Production — $199/month
- Full 5-axis simultaneous machining
- AI toolpath optimization
- All 500+ post-processors
- Multi-setup planning
- Quoting and setup sheet generation
- API access

### Enterprise — Custom
- On-premise deployment
- Custom machine model creation
- ERP/MES integration
- Training and onboarding
- Custom post-processor development
- SLA with dedicated support

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | CAMWorks Advantage |
|-----------|-----------|------------|-------------------|
| Mastercam | Industry standard, huge post library | $10-30K/seat, desktop only, steep learning curve | 10x cheaper, browser-based, AI feature recognition |
| Fusion 360 CAM | Integrated CAD/CAM, affordable | Limited 5-axis, weak post-processors | Superior 5-axis, 500+ verified posts, AI optimization |
| Siemens NX CAM | Best-in-class 5-axis | $20K+/seat, enterprise-only | Accessible pricing, faster learning curve |
| Vericut | Gold standard simulation | $15K+ add-on, separate from CAM | Built-in simulation at no extra cost |
| CloudNC CAM Assist | AI-driven, modern | Limited availability, narrow scope | Broader feature set, self-serve, open platform |

---

## MVP Scope (v1.0)

### In Scope
- STEP file import with basic feature recognition (holes, pockets, contours)
- 2.5D and 3-axis toolpath generation (adaptive clearing, contour finishing)
- Tool library with common end mills and drills
- Basic feeds/speeds from material database
- Machine simulation (simplified bounding-box collision)
- G-code output for Fanuc, Haas, Grbl

### Out of Scope (v1.1+)
- 5-axis simultaneous machining
- AI toolpath optimization
- Full kinematic machine simulation
- Quoting and production features
- ERP/MES integration

### MVP Timeline: 14-18 weeks
