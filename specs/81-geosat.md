# GeoSat — Satellite Orbit Design and Analysis Platform

## Executive Summary

GeoSat is a cloud-based platform for satellite orbit design, constellation planning, ground coverage analysis, link budget computation, and orbital maneuver planning. It replaces AGI STK ($15K-$50K/seat), NASA GMAT (open-source but complex), FreeFlyer ($30K+/seat), and Orbit Logic tools with a modern browser-based environment that democratizes mission planning for the 2,000+ NewSpace companies entering the market.

---

## Problem Statement

**The pain:**
- AGI STK (Systems Tool Kit) costs $15,000-$50,000/seat/year depending on modules; FreeFlyer costs $30,000-$60,000/seat; these are priced for defense contractors, not NewSpace startups with $2-10M seed rounds
- NASA GMAT is free but is a desktop Java application with a 1990s UI, poor documentation, and a steep learning curve — takes 3-6 months to become productive
- No single tool covers the full mission design workflow: orbit selection → constellation design → coverage analysis → link budget → maneuver planning → collision avoidance → end-of-life disposal
- Conjunction assessment / collision avoidance (COLA) is becoming critical as LEO congestion increases (30,000+ tracked objects); current tools treat it as an afterthought rather than a core workflow
- Small satellite operators (CubeSats, SmallSats) need orbit analysis tools but cannot justify $50K/seat for a 3-person mission planning team

**Current workarounds:**
- Using open-source libraries (Orekit, poliastro, Skyfield) with custom Python scripts — works but is fragile, undocumented, and lacks visualization
- Using NASA GMAT for high-fidelity propagation but switching to STK for visualization and coverage, creating a disjointed workflow
- Spreadsheet-based link budget calculations that do not account for dynamic orbit geometry, rain fade, or antenna pointing losses
- Relying on launch providers or ride-share brokers for orbit selection without independent analysis capability

**Market size:** The small satellite market is $7 billion (2024) growing at 20%+ CAGR. There are 2,000+ active satellite companies, 800+ satellites launched per year, and 40,000+ aerospace/mission planning engineers globally. The satellite ground segment and mission operations software market is approximately $3 billion.

---

## Target Users

### Primary Personas

**1. Elena — Mission Systems Engineer at a NewSpace Startup**
- Designing a 48-satellite LEO constellation for IoT connectivity
- Evaluated STK but cannot justify $200K+ for a 4-person team
- Needs: constellation design, coverage analysis, link budget, and launch window planning in one affordable tool

**2. James — Orbital Analyst at a Satellite Operator**
- Manages on-orbit operations for 12 GEO and LEO satellites: station-keeping, collision avoidance, end-of-life disposal
- Uses FreeFlyer for maneuver planning and receives conjunction data messages (CDMs) from 18th Space Defense Squadron
- Needs: automated conjunction screening, maneuver planning with fuel-optimal solutions, and what-if scenario analysis

**3. Priya — Graduate Student / Academic Researcher**
- Studying orbital mechanics and mission design for a CubeSat university project
- Uses GMAT but struggles with the interface and limited tutorials
- Needs: intuitive orbit propagation, 3D visualization, and educational mode explaining orbital mechanics concepts

---

## Solution Overview

GeoSat is a cloud-based satellite mission planning platform that:
1. Designs orbits from mission requirements (altitude, inclination, RAAN, repeat ground track, sun-synchronous, frozen orbit) with a visual orbit builder and 3D globe visualization
2. Plans constellations (Walker-Delta, Walker-Star, Flower, custom) with coverage analysis: revisit time, gap statistics, and minimum elevation angle constraints per ground station/target
3. Computes RF link budgets dynamically: transmitter → free-space loss → atmospheric loss → rain fade → receiver G/T → link margin, integrated with the orbit geometry for contact time and data throughput
4. Plans orbital maneuvers: Hohmann, bi-elliptic, phasing, station-keeping, collision avoidance, and low-thrust spiral transfers with delta-V budget tracking
5. Performs conjunction assessment (COLA): screens against the public catalog (Space-Track TLEs/GP data), computes probability of collision (Pc), and recommends avoidance maneuvers

---

## Core Features

### F1: Orbit Design and Propagation
- Keplerian orbit definition: a, e, i, RAAN, AoP, TA (or alternative element sets: TLE, equinoctial, Delaunay)
- Orbit type wizards: sun-synchronous, repeat ground track, frozen orbit, Molniya, GEO, GTO, lunar transfer
- High-fidelity propagation: J2-J6 geopotential (EGM96/EGM2008), atmospheric drag (NRLMSISE-00, Jacchia-Roberts), solar radiation pressure (SRP) with shadow modeling (cylindrical, conical), lunisolar third-body perturbations
- Numerical integrators: Dormand-Prince (RK7(8)), Adams-Bashforth-Moulton, Gauss-Legendre (symplectic)
- Analytical/semi-analytical propagation: SGP4/SDP4 (TLE-based), Brouwer-Lyddane, DSST
- Epoch state estimation from TLE or GPS ephemeris
- Orbit determination: batch least-squares and sequential (EKF/UKF) from ground station range/range-rate/angle measurements
- Coordinate frames: J2000 (EME2000), TEME, ECEF (ITRF), GCRF, topocentric (AER)

### F2: Constellation Design and Coverage
- Walker-Delta and Walker-Star constellation generators with automatic phasing
- Custom constellation builder: arbitrary orbit planes, relative phasing, and spare satellites
- Ground coverage analysis: compute access intervals, revisit time, gap duration for any target point or area
- Coverage figures of merit: daily revisit, maximum gap, average response time, percent coverage over time
- Sensor/swath modeling: conical, rectangular, custom FOV with off-nadir pointing
- Ground station contact analysis: visibility windows, elevation mask, antenna tracking constraints
- Multi-layer coverage: overlapping constellation analysis for redundancy
- Coverage heat maps on 3D globe: color-coded revisit time, number of passes, total contact time
- Latitude-dependent coverage statistics (polar vs. equatorial performance)

### F3: Link Budget Analysis
- RF link equation: EIRP - free space path loss - atmospheric attenuation + G/T - system noise → C/N0 → link margin
- Frequency bands: UHF, S, X, Ka, V, optical (free-space optical communication)
- Atmospheric losses: ITU-R P.676 gaseous absorption, ITU-R P.618 rain attenuation (availability-based)
- Antenna models: isotropic, parabolic dish, patch, phased array, helix — with gain patterns
- Dynamic link budget: margin vs. time over a pass, data volume per contact, total daily throughput
- Modulation and coding: spectral efficiency for QPSK, 8PSK, 16APSK with LDPC/turbo coding rates
- Inter-satellite link (ISL) budget for constellation networking
- Doppler shift calculation for LEO downlinks
- Rain fade margin and site diversity analysis for Ka-band and above

### F4: Maneuver Planning
- Impulsive maneuvers: Hohmann transfer, bi-elliptic transfer, plane change, combined maneuvers
- Phasing maneuvers: drift orbit computation for constellation deployment
- Station-keeping: east-west (GEO), north-south (GEO), ground track maintenance (LEO SSO)
- Low-thrust trajectory: electric propulsion spiral transfer, Edelbaum approximation, numerical optimal control
- Delta-V budget tracker: allocate and track delta-V for orbit raising, station-keeping, COLA, disposal
- Fuel/propellant mass accounting: Tsiolkovsky equation with Isp, tank capacity, and thruster performance
- Multi-burn optimization: minimize delta-V for complex transfer scenarios
- Launch window analysis: determine launch date/time for target RAAN, constellation phasing, and eclipse constraints
- De-orbit and graveyard disposal planning per IADC guidelines (25-year rule, graveyard orbit)

### F5: Conjunction Assessment (COLA) and Space Safety
- TLE/GP catalog ingestion from Space-Track (automatic daily updates)
- Conjunction screening: filter close approaches within user-defined miss distance and time thresholds
- Probability of collision (Pc) computation: Alfriend-Akella, Foster, Chan methods with covariance realism
- Conjunction data message (CDM) parsing and trending: Pc vs. time to TCA
- Avoidance maneuver planning: compute minimum delta-V maneuver to reduce Pc below threshold
- Maneuver screening: verify proposed maneuvers do not create new conjunctions
- Debris environment modeling: NASA ORDEM for flux/probability analysis
- Regulatory compliance: FCC orbital debris mitigation, ESA Space Debris Mitigation Requirements, ITU filing support
- Conjunction event timeline and risk dashboard

### F6: Visualization and Reporting
- 3D globe visualization: satellite orbits, ground tracks, coverage footprints, line-of-sight cones (CesiumJS-based)
- 2D ground track maps with day/night terminator
- Orbit element plots: SMA, eccentricity, inclination, RAAN drift vs. time
- Eclipse and sunlight analysis: eclipse duration, battery depth-of-discharge estimation
- Thermal environment: solar flux, Earth IR, albedo as a function of orbit position
- Mission timeline: Gantt chart of mission phases (LEOP, commissioning, operations, disposal)
- Report generation: PDF mission design document with orbit parameters, coverage summary, link budget, delta-V budget
- Data export: CCSDS OEM/OPM ephemeris, STK ephemeris, TLE, CSV

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | TypeScript, React, CesiumJS (3D globe, orbit visualization), D3.js (plots, coverage charts) |
| Orbit Propagator | Rust (high-fidelity numerical propagation, SGP4, Brouwer-Lyddane) → WASM for client-side + server for batch |
| Coverage Engine | Rust (access interval computation, grid coverage) → server-side for constellation-scale |
| Link Budget | Rust (RF link equation, atmospheric models) → WASM for interactive |
| Maneuver Planner | Rust + Python (optimization: SciPy, IPOPT for low-thrust) |
| COLA Engine | Rust (conjunction screening, Pc computation, CDM processing) |
| AI/ML | Python (PyTorch — orbit prediction, anomaly detection, maneuver recommendation) |
| Backend | Rust (Axum) + Python (FastAPI for ML, reporting, Space-Track API proxy) |
| Database | PostgreSQL (missions, spacecraft, ground stations), Redis (TLE catalog cache), S3 (ephemeris files, reports) |
| Hosting | AWS (compute for batch propagation, standard for interactive) |

---

## Monetization

### Free Tier (Student)
- Single satellite orbit propagation (J2 perturbation)
- 3D orbit visualization on globe
- Basic ground track and eclipse analysis
- 3 projects max

### Pro — $149/month
- High-fidelity propagation (full perturbation model)
- Constellation design (up to 24 satellites)
- Coverage analysis (single target area)
- Link budget (single link)
- Maneuver planning (impulsive)
- 15 projects

### Advanced — $349/user/month
- Unlimited constellation size
- Global coverage analysis with grid computation
- Full link budget suite (ISL, optical)
- Low-thrust trajectory optimization
- Conjunction assessment (COLA) with Space-Track integration
- Orbit determination
- API access
- Collaborative mission planning

### Enterprise — Custom
- On-premise deployment for classified/ITAR missions
- Real-time conjunction screening with custom catalog
- Operations center integration (FDS, ground station network)
- Custom propagation models and force models
- Dedicated compute and SLA

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | GeoSat Advantage |
|-----------|-----------|------------|-----------------|
| AGI STK (Ansys) | Industry standard, massive feature set, defense-trusted | $15-50K/seat, desktop, steep learning curve, defense-focused pricing | Browser-based, 10x cheaper, NewSpace-friendly |
| NASA GMAT | Free, high-fidelity propagation, NASA-validated | Desktop Java app, poor UX, no coverage analysis, minimal visualization | Modern web UI, integrated coverage + link budget + COLA |
| FreeFlyer (a.i. solutions) | Strong maneuver planning, operations-grade | $30-60K/seat, desktop, limited constellation tools | Affordable, constellation-native, built-in COLA |
| Orbit Logic | Good scheduling and tasking | Niche (scheduling-focused), limited orbit design | Full mission design lifecycle, not just scheduling |
| SaVi / open-source | Free, basic constellation viz | Minimal features, no propagation fidelity, no link budget | Production-grade with full analysis suite |

---

## MVP Scope (v1.0)

### In Scope
- Orbit definition from classical elements or TLE, with J2-J6 + drag + SRP propagation
- 3D globe visualization with orbit, ground track, and day/night (CesiumJS)
- Sun-synchronous and repeat ground track orbit wizards
- Walker constellation generator with coverage heat map (revisit time)
- Ground station contact windows and basic link budget (single frequency band)
- Impulsive maneuver planning (Hohmann, plane change) with delta-V computation
- Eclipse analysis and basic mission timeline

### Out of Scope (v1.1+)
- Low-thrust trajectory optimization
- Conjunction assessment (COLA) and Space-Track integration
- Orbit determination from measurements
- Inter-satellite link analysis
- Advanced link budget (rain fade, phased array, optical)
- Operations center integration

### MVP Timeline: 14-18 weeks
