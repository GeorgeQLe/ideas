# TurboGen — Turbomachinery Design and Analysis Platform

## Executive Summary

TurboGen is a cloud-based platform for designing and analyzing turbomachinery — compressors, turbines, pumps, and fans. It covers meanline/throughflow design, 3D blade geometry generation, CFD analysis, and aeroelastic/aeroacoustic prediction. It replaces AxSTREAM ($25K+/seat), NUMECA/Cadence ($30K+/seat), and CFX turbomachinery ($20K+/seat add-on).

---

## Problem Statement

**The pain:**
- Turbomachinery design software is extremely niche: AxSTREAM (SoftInWay) costs $25,000-$50,000/seat; NUMECA (now Cadence CFD) costs $30,000-$60,000/seat; ANSYS CFX with turbo modules costs $20,000-$40,000/seat
- Designing a single turbine or compressor stage requires seamless progression from 1D meanline → 2D throughflow → 3D blade design → CFD validation — no affordable tool handles this full workflow
- Turbomachinery CFD requires specialized meshing (structured multi-block, O-H-C topology) that general-purpose meshers handle poorly
- Small turbocharger companies, HVAC fan manufacturers, and hydro turbine developers cannot afford $50K+ tool stacks

**Market size:** The turbomachinery market exceeds $200 billion annually (gas turbines, steam turbines, compressors, pumps, fans). The design software segment is approximately $1 billion. There are 100,000+ turbomachinery engineers worldwide.

---

## Core Features

### F1: Meanline/Throughflow Design
- 1D meanline analysis: velocity triangles, work coefficient, flow coefficient, degree of reaction
- Loss models: Ainley-Mathieson, Kacker-Okapuu (turbines), Lieblein, Koch-Smith (compressors)
- Throughflow (2D): streamline curvature method with spanwise property distribution
- Off-design performance prediction and compressor/turbine maps
- Preliminary sizing: hub/tip radii, blade count, aspect ratio, solidity
- Cycle analysis integration: specify cycle conditions → design turbomachinery to match

### F2: 3D Blade Design
- Parameterized blade profile generation: camber line, thickness distribution, leading/trailing edge
- Standard profiles: NACA 65, C4, DCA, custom Bezier
- 3D blade stacking: lean, sweep, bow, compound lean
- Endwall contouring for secondary flow reduction
- Tip clearance and shroud geometry
- Impeller design for centrifugal compressors/pumps (with diffuser)
- STEP/IGES export of blade and passage geometry

### F3: Turbomachinery CFD
- Structured multi-block meshing with O-H-C topology (turbo-specific)
- Steady RANS with mixing plane interface between rotor and stator
- Unsteady RANS/LES with sliding mesh for rotor-stator interaction
- Turbulence: SA, k-ω SST, transition SST
- Rotating reference frame and absolute/relative velocity decomposition
- Performance metrics: efficiency, pressure ratio, mass flow, surge/choke margin
- Stage and multi-stage analysis
- Off-design simulation across operating range (compressor/fan maps)

### F4: Aeroelastic and Aeroacoustic
- Campbell diagram: natural frequencies vs. rotational speed, avoiding resonance crossings
- Flutter prediction: unsteady aerodynamic damping computation
- Forced response: rotor-stator interaction excitation
- Aeroacoustic: tonal noise (blade passing frequency), broadband noise prediction
- Fan noise: Tyler-Sofrin modes, duct acoustic propagation

### F5: Pump and Fan Specific
- Pump hydraulic design: specific speed, impeller meridional design, volute sizing
- NPSH prediction and cavitation inception
- Fan performance: pressure coefficient, flow coefficient, efficiency curves
- HVAC fan selection: recommend fan type (axial, centrifugal, mixed flow) from duty point

---

## Monetization

### Academic — $99/month per lab
- Meanline and throughflow design
- Basic 3D blade generation
- Steady RANS CFD (single stage)
- 5 seats

### Pro — $299/month
- Full design workflow (meanline → blade → CFD)
- Multi-stage analysis
- Off-design performance maps
- Structured turbo meshing
- Aeroelastic (Campbell diagram)

### Enterprise — Custom
- Unsteady CFD (rotor-stator interaction)
- Flutter and forced response
- Aeroacoustics
- Custom profile/loss model integration
- On-premise deployment

---

## MVP Scope (v1.0)

### In Scope
- Meanline analysis for axial turbine/compressor stage
- Velocity triangle visualization and loss estimation
- Parameterized 2D blade profile generation
- Basic 3D blade stacking
- Steady RANS CFD of single passage (simplified mesh)
- Performance metrics (efficiency, pressure ratio)

### Out of Scope (v1.1+)
- Throughflow, centrifugal, aeroelastics, aeroacoustics, multi-stage

### MVP Timeline: 14-18 weeks
