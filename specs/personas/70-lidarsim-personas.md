# LidarSim — Persona Subspecs

> Parent spec: [`specs/70-lidarsim.md`](../70-lidarsim.md)

---

## Experience-Level Personas

### Student/Academic

**Modified UI/UX:**
- Guided LiDAR fundamentals mode with annotated diagrams explaining time-of-flight, FMCW, and flash LiDAR principles
- Interactive link budget calculator with visual breakdown of laser power, atmospheric attenuation, target reflectance, and detector noise
- Pre-built scene library (urban intersection, forest canopy, simple indoor room) for learning without complex scene creation

**Feature Gating:**
- Basic LiDAR system modeling (pulsed, single beam) with link budget calculation; FMCW and solid-state modeling locked
- Scene simulation limited to static scenes with <100K triangles; dynamic objects and weather effects disabled
- Point cloud viewer with basic filtering; classification and segmentation algorithms limited to supervised methods

**Pricing Tier:** Free tier (academic email verification)

**Onboarding Flow:**
- Start with a guided pulsed LiDAR link budget tutorial explaining each parameter's physical meaning
- Interactive exercise on point cloud fundamentals: point density, angular resolution, and range noise
- Access to public LiDAR datasets (KITTI, Waymo Open) for point cloud processing practice

**Key Workflows:**
- Computing LiDAR link budgets for different system configurations in optics/sensor coursework
- Simulating point clouds from simple scenes to understand range noise, resolution, and density trade-offs
- Processing and visualizing public LiDAR datasets for computer vision and perception coursework
- Generating system performance plots (range vs. reflectivity, SNR vs. distance) for thesis research

---

### Junior Engineer (0-3 years)

**Modified UI/UX:**
- Template-based LiDAR system design organized by application (automotive, surveying, forestry, industrial)
- Guided scene construction with asset library (vehicles, buildings, terrain, vegetation) and drag-and-drop placement
- Side-by-side comparison of simulated vs. real point clouds with quality metrics (density, noise, completeness)

**Feature Gating:**
- Full LiDAR system modeling (pulsed, FMCW, flash) with standard detector models; custom optical design locked
- Dynamic scene simulation enabled; adverse weather (rain, fog) available with standard models
- Point cloud classification and segmentation with built-in algorithms; custom ML model training locked

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Select application (automotive ADAS, topographic survey, forestry inventory) for tailored system templates
- Guided walkthrough: define LiDAR system, build scene, run simulation, analyze point cloud quality
- Tutorial on interpreting point cloud quality metrics and their relationship to system design parameters

**Key Workflows:**
- Designing LiDAR systems with specified FOV, range, resolution, and frame rate requirements
- Running scene simulations to predict point cloud quality under various operating conditions
- Evaluating LiDAR performance in adverse weather (rain, fog, snow) using physics-based models
- Processing point clouds with built-in classification and segmentation pipelines
- Generating system specification documents from simulation-derived performance data

---

### Senior/Expert

**Modified UI/UX:**
- Full scripting API (Python) for custom optical models, signal processing chains, and automated design optimization
- Multi-system workspace for designing LiDAR sensor suites with coverage overlap and interference analysis
- Direct access to ray tracing engine configuration, BRDF models, and detector noise parameters

**Feature Gating:**
- All features unlocked: custom optical system design (Zemax-level), multi-LiDAR interference, advanced BRDF models
- Custom detector model development with noise characterization and signal processing chain simulation
- GPU-accelerated ray tracing for large-scale scene simulation (10M+ triangles, city-scale)

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Import existing Zemax optical designs and MATLAB simulation scripts with automated translation
- Benchmark against published LiDAR characterization data for system model validation
- Configure automated design optimization pipelines with performance targets and constraints

**Key Workflows:**
- Designing custom LiDAR optical systems with aberration analysis and tolerance sensitivity
- Developing novel scanning mechanisms (MEMS, OPA, flash) with full system-level simulation
- Running city-scale scene simulations for autonomous vehicle perception system validation
- Building automated LiDAR-in-the-loop testing pipelines for ADAS/AD development
- Leading multi-sensor fusion architecture design with LiDAR, camera, and radar co-simulation

---

## Role-Based Personas

### Designer/Creator

**Modified UI/UX:**
- Optical design canvas with lens editor, scanning mechanism configurator, and detector layout tools
- Real-time system performance preview updating FOV, resolution, and range as design parameters change
- Design variant manager for comparing LiDAR architectures (mechanical spinning vs. MEMS vs. OPA vs. flash)

**Feature Gating:**
- Full optical system design tools with ray tracing, lens optimization, and tolerance analysis
- Scanning mechanism design (MEMS mirror, polygon scanner, OPA phased array) with beam steering simulation
- Detector array layout tools with pixel pitch, fill factor, and readout circuit integration

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Start from a template LiDAR architecture and walk through optical design, scanning, and detector configuration
- Interactive comparison of LiDAR architectures highlighting performance and manufacturing trade-offs
- Demonstrate the design iteration workflow: modify optics, simulate point cloud, evaluate quality

**Key Workflows:**
- Designing LiDAR transmitter optics (collimation, beam shaping) for specified divergence and power density
- Configuring scanning mechanisms and optimizing scan patterns for uniform point cloud density
- Designing receiver optics and detector configurations for target range and SNR requirements
- Comparing LiDAR architecture alternatives on system-level performance metrics
- Generating optical design specifications and tolerance analyses for manufacturing

---

### Analyst/Simulator

**Modified UI/UX:**
- Simulation workspace with scene rendering, point cloud generation, and statistical quality analysis tools
- Multi-scenario analyzer with environmental parameter sweeps (weather, lighting, target types)
- Perception performance dashboard linking point cloud quality to downstream algorithm performance metrics

**Feature Gating:**
- All simulation and analysis capabilities available based on license tier
- Advanced scene effects: multi-path reflections, cross-talk, blooming, detector saturation
- Batch simulation with automated scenario generation for large-scale validation campaigns

**Pricing Tier:** Professional or Enterprise tier

**Onboarding Flow:**
- Tutorial on building physically accurate scenes with correct material BRDF properties
- Guided simulation study evaluating LiDAR performance degradation in rain and fog
- Introduction to statistical analysis of point cloud quality across scenario variations

**Key Workflows:**
- Running physics-based LiDAR simulations across thousands of driving scenarios for perception validation
- Analyzing point cloud quality metrics (density, noise, false detections) under adverse conditions
- Evaluating multi-LiDAR interference effects in dense traffic scenarios
- Generating synthetic training data for perception algorithm development with ground truth labels
- Validating simulation fidelity by comparing synthetic point clouds against real sensor data

---

### Manufacturing/Process

**Modified UI/UX:**
- Manufacturing-focused interface with optical alignment tolerances, assembly procedures, and calibration routines
- Production quality dashboard showing measured vs. specified beam quality, scan accuracy, and range performance
- End-of-line test workflow with automated pass/fail criteria from design specifications

**Feature Gating:**
- Optical alignment tolerance analysis tools; no system design modification
- Calibration and characterization simulation for production setup optimization
- End-of-line test specification generation from design performance requirements

**Pricing Tier:** Professional tier (manufacturing license)

**Onboarding Flow:**
- Configure manufacturing test equipment interfaces (beam profiler, range accuracy station, FOV verification)
- Walk through production tolerance analysis linking assembly variations to system performance
- Tutorial on setting up end-of-line test criteria from system design specifications

**Key Workflows:**
- Generating assembly tolerance budgets based on sensitivity analysis of optical alignment
- Simulating manufacturing variations to predict production yield and identify critical tolerances
- Developing calibration procedures and intrinsic parameter estimation routines
- Running end-of-line test simulations to set pass/fail thresholds with appropriate margins
- Supporting failure analysis for production units not meeting specifications

---

### Regulatory/Compliance

**Modified UI/UX:**
- Safety compliance dashboard organized by standard (IEC 60825 laser safety, ISO 26262 functional safety, NHTSA guidelines)
- Eye safety analysis workspace with accessible emission limit (AEL) calculations for all LiDAR configurations
- Functional safety evidence tracker for ADAS LiDAR systems per ISO 26262 requirements

**Feature Gating:**
- Read-only access to system specifications and safety analysis results
- Automated laser safety classification (Class 1, 1M, 3R, 3B) per IEC 60825
- ISO 26262 documentation templates for LiDAR system safety requirements and validation evidence

**Pricing Tier:** Enterprise tier (safety add-on)

**Onboarding Flow:**
- Select applicable standards (IEC 60825, ISO 26262, regional automotive regulations) for template configuration
- Walk through a laser safety analysis from system parameters to eye safety classification
- Configure functional safety documentation workflow with ASIL-level traceability

**Key Workflows:**
- Performing laser eye safety analysis and classification per IEC 60825 for all operating modes
- Documenting functional safety requirements for automotive LiDAR per ISO 26262
- Generating safety case documentation linking system design to hazard mitigation evidence
- Reviewing LiDAR performance claims against regulatory test requirements (NHTSA, Euro NCAP)
- Maintaining compliance documentation for product certification and type approval submissions

---

### Manager/Decision-maker

**Modified UI/UX:**
- Product dashboard with LiDAR development milestones, performance benchmarks vs. competition, and cost targets
- Technology roadmap view showing LiDAR platform evolution with performance and cost projections
- Market positioning analysis comparing system specifications against competitive products and customer requirements

**Feature Gating:**
- View-only access to system performance summaries, program status, and competitive benchmarking
- Portfolio view across LiDAR product lines with technology readiness and market fit indicators
- Budget management for simulation, prototyping, and certification activities

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Configure product portfolio with development milestones, performance targets, and cost goals
- Walk through interpreting system performance benchmarks in the context of market requirements
- Set up automated reporting for development progress and competitive positioning

**Key Workflows:**
- Reviewing LiDAR system performance against customer requirements and competitive benchmarks
- Making technology investment decisions (MEMS vs. OPA vs. flash) based on simulation-predicted performance-cost trade-offs
- Monitoring development program progress against automotive qualification timelines
- Evaluating build vs. buy decisions for LiDAR subsystems based on simulation-derived specifications
- Generating investor and customer presentations with verified performance data from simulation
