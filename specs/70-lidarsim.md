# LidarSim — LiDAR System Design and Point Cloud Processing Platform

## Executive Summary

LidarSim is a cloud-based platform for LiDAR system design, simulation, and point cloud processing. It covers LiDAR hardware design (optical, mechanical scanning, solid-state), scene simulation with physics-based ray tracing, and point cloud analytics (classification, segmentation, change detection). It replaces Zemax (for LiDAR optics, $15K+), custom MATLAB simulation, and TerraScan/CloudCompare ($5K+/seat) with an integrated design-to-deployment workflow.

---

## Problem Statement

**The pain:**
- LiDAR system design spans optics, electronics, signal processing, and software — no single tool covers the full stack
- Automotive LiDAR companies spend $1M+ per year on custom simulation frameworks because no commercial tool handles LiDAR-specific scenarios (rain, fog, retroreflectors, multi-LiDAR interference)
- Point cloud processing tools (TerraScan, RiSCAN, Leica Cyclone) cost $5,000-$20,000/seat and handle post-processing but not system-level design
- Testing LiDAR in adverse weather (rain, snow, fog) requires expensive chamber testing ($50K-$200K per test campaign); simulation could replace 70% of physical tests
- There are 500+ LiDAR startups globally, most building their own simulation from scratch

**Market size:** The LiDAR market is $2.5 billion (2024) growing to $8 billion by 2030. LiDAR simulation and processing software is approximately $500 million. There are 50,000+ LiDAR engineers across automotive, surveying, forestry, mining, and robotics.

---

## Core Features

### F1: LiDAR System Modeling
- Laser source models: pulsed (905nm, 1550nm), FMCW, flash
- Scanning mechanisms: mechanical spinning, MEMS mirror, OPA, flash array
- Detector models: APD, SPAD, SiPM with noise (dark count, afterpulsing, thermal)
- Link budget: laser power → atmospheric transmission → target reflectance → received power → SNR → range
- Range equation solver with atmospheric extinction (Mie, Rayleigh)
- FOV, angular resolution, point density, and frame rate calculation
- Power consumption and thermal budget estimation

### F2: Scene Simulation
- Physics-based ray tracing: lidar pulse emission → scene interaction → return signal
- Material models: Lambertian, retroreflective, specular, BRDF-based
- Atmospheric effects: rain (Mie scattering), fog (visibility-dependent), snow, dust
- Multi-return processing: ground, vegetation canopy, building edges
- Dynamic scenes: moving vehicles, pedestrians (imported from driving simulators)
- Multi-LiDAR interference simulation
- Synthetic point cloud generation with realistic noise characteristics

### F3: Point Cloud Processing
- Ground classification (progressive morphological filter, cloth simulation)
- Building/vegetation/power line classification (ML-based)
- Semantic segmentation (PointNet++, RandLA-Net)
- Change detection between temporal scans
- DEM/DSM generation from classified ground points
- Building footprint extraction
- Tree inventory (individual tree detection, height, crown diameter)
- Volume calculation (stockpile, excavation)

### F4: Surveying and Mapping
- Point cloud registration (ICP, feature-based)
- Coordinate system transformation and georeferencing
- Accuracy assessment: control points, RMSE
- Profiles, cross-sections, and contour generation
- Clash detection for BIM/infrastructure
- Colorization from co-registered imagery
- Deliverable generation: DEM, DSM, contours, classified LAS

---

## Monetization

### Free Tier
- Point cloud viewer (up to 10M points)
- Basic ground classification
- LiDAR link budget calculator

### Pro — $149/month
- Full point cloud processing pipeline
- LiDAR system simulation
- Scene simulation (clear weather)
- ML-based classification
- 50GB cloud storage

### Advanced — $349/user/month
- Adverse weather simulation
- Dynamic scene simulation
- Custom ML model training
- Unlimited storage
- API access

### Enterprise — Custom pricing
- On-premise deployment
- Custom scene/sensor models
- Integration with AV simulation (CARLA, LGSVL)
- Regulatory compliance testing support

---

## MVP Scope (v1.0)

### In Scope
- LiDAR link budget calculator
- Basic scene simulation (static, clear weather)
- Point cloud viewer (web-based, up to 50M points)
- Ground classification (progressive morphological filter)
- DEM/DSM generation
- LAS/LAZ import/export

### Out of Scope (v1.1+)
- Adverse weather, dynamic scenes, ML classification, multi-LiDAR interference

### MVP Timeline: 12-16 weeks
