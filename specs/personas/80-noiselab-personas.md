# NoiseLab — Persona Subspecs

> Parent spec: [`specs/80-noiselab.md`](../80-noiselab.md)

---

## Experience-Level Personas

### Student/Academic
**Modified UI/UX:**
- Interactive modal analysis visualizer with animated mode shapes and annotated natural frequency spectrum
- Simplified acoustic source-path-receiver model builder with educational explanations of NVH fundamentals
- Color-coded frequency response function (FRF) explorer with overlay of structural and acoustic resonances

**Feature Gating:**
- Basic modal analysis and frequency response computation for simple structures unlocked
- Sound pressure level calculation at field points from vibrating surfaces available
- Statistical energy analysis (SEA), transfer path analysis (TPA), and full-vehicle models locked

**Pricing Tier:** Free tier (academic license)

**Onboarding Flow:**
- Tutorial: compute natural frequencies and mode shapes of a simple beam/plate structure
- Pre-loaded examples: guitar body resonance, loudspeaker cavity, vehicle door panel vibration
- Guided exercise comparing FEA modal results against analytical solutions (Euler-Bernoulli beam)

**Key Workflows:**
- Computing natural frequencies and mode shapes for simple structural components
- Calculating sound radiation from vibrating plates using Rayleigh integral approximations
- Generating frequency response functions and interpreting resonance peaks
- Producing mode shape animations and SPL contour plots for acoustics coursework

### Junior Engineer (0-3 years)
**Modified UI/UX:**
- Guided NVH analysis workflow: import geometry, define materials, create mesh, solve, post-process
- Target tracking dashboard showing key NVH metrics against benchmarks (road noise, engine order, wind noise)
- Contribution analysis panels identifying dominant paths and components in NVH response

**Feature Gating:**
- Full modal analysis and forced response computation for components and subsystems
- Acoustic radiation and cavity acoustics (BEM/FEM) with guided setup
- Transfer path analysis available with templates; full-vehicle SEA and coupled FE-SEA require approval

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Role-based setup: powertrain NVH, road noise, wind noise, interior acoustics, or pass-by noise focus
- Walkthrough: import a body panel, compute modes, assess acoustic radiation, compare to target
- Introduction to NVH target cascading and departmental reporting workflows

**Key Workflows:**
- Performing modal analysis of body and trim components to identify resonance issues
- Computing acoustic radiation from vibrating structures using boundary element methods
- Running forced response analysis for engine order excitation and road input
- Performing contribution analysis to rank dominant noise paths for targeted improvement
- Generating NVH status reports against vehicle program targets

### Senior/Expert
**Modified UI/UX:**
- Full-vehicle NVH modeling console integrating structural FE, acoustic FE/BEM, SEA, and hybrid FE-SEA
- Custom excitation source editor for engine, road, wind, and drivetrain inputs with measured data import
- Python/MATLAB scripting for automated NVH optimization, DOE, and correlation studies

**Feature Gating:**
- All modules: full-vehicle models, SEA, hybrid FE-SEA, TPA, pass-by noise, sound quality metrics
- Full API for integration with CAE tools (Nastran, Abaqus), test data systems, and vehicle development platforms
- Admin controls for proprietary NVH databases, target libraries, and modeling standards

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Migration assistant for importing LMS Virtual.Lab/Actran models and legacy NVH databases
- Configuration of vehicle NVH target systems and benchmark databases
- Setup of automated model correlation workflows against physical test data

**Key Workflows:**
- Building and solving full-vehicle NVH models from trimmed body to interior sound field
- Performing transfer path analysis (classical, operational, component) for noise source ranking
- Running coupled FE-SEA analysis covering low-to-high frequency range in a single model
- Optimizing structural modifications and acoustic treatments for NVH target achievement
- Correlating simulation predictions with chassis dyno and proving ground test data

---

## Role-Based Personas

### Designer/Creator
**Modified UI/UX:**
- Acoustic treatment designer with material layup editor for absorption, barrier, and damping configurations
- Structural modification advisor suggesting rib patterns, mass distribution, and geometry changes for modal tuning
- Interior trim package configurator with insertion loss and absorption coefficient visualization

**Feature Gating:**
- Full access to acoustic treatment design tools and structural modification advisory features
- Material database with acoustic properties (absorption, transmission loss, damping loss factor)
- Export to trim design specifications and structural modification engineering change requests

**Pricing Tier:** Professional or Enterprise tier

**Onboarding Flow:**
- Quick-start: specify an acoustic treatment for a dash panel to meet sound transmission loss targets
- Tutorial on using simulation to guide structural bead pattern design for modal frequency targeting
- Walkthrough of trim material selection using insertion loss and weight trade-off analysis

**Key Workflows:**
- Designing acoustic treatment packages (barriers, absorbers, dampers) for vehicle interior quietness
- Optimizing structural panel geometry (beads, ribs, thickness) for modal frequency targeting
- Specifying engine compartment acoustic encapsulation for powertrain noise reduction
- Designing door and window sealing systems for wind noise path control
- Creating acoustic design specifications with measurable performance requirements

### Analyst/Simulator
**Modified UI/UX:**
- Multi-method analysis manager switching between FEM, BEM, SEA, and hybrid approaches by frequency band
- Excitation source library with engine firing order, road surface PSD, wind tunnel pressure, and TPA inputs
- Sound quality metric calculator (loudness, sharpness, roughness, articulation index) from simulated signals

**Feature Gating:**
- Full solver access for all acoustic and vibro-acoustic methods (FEM, BEM, SEA, FE-SEA, IFEM)
- Advanced post-processing including transfer function synthesis, auralization, and sound quality evaluation
- HPC integration for large full-vehicle models and multi-frequency-band computations

**Pricing Tier:** Professional or Enterprise tier

**Onboarding Flow:**
- Guided first analysis: modal analysis and acoustic radiation from a flat panel with BEM validation
- Setup of analysis workflow templates for standard NVH load cases (engine idle, road cruise, WOT)
- Introduction to frequency-banded analysis strategy (FEM low-freq, SEA high-freq, hybrid mid-freq)

**Key Workflows:**
- Running coupled vibro-acoustic simulations for interior noise prediction at driver and passenger ears
- Performing operational transfer path analysis from measured or simulated operational data
- Computing exterior pass-by noise levels per ISO 362-1 for type approval prediction
- Evaluating sound quality metrics for subjective evaluation correlation
- Correlating NVH simulation results with physical test data and updating model parameters

### Manufacturing/Process
**Modified UI/UX:**
- Production variability impact analyzer showing NVH sensitivity to manufacturing tolerances
- Assembly process specification generator for acoustic-critical components (seals, mounts, treatments)
- End-of-line NVH test specification tool defining pass/fail criteria from simulation-predicted bounds

**Feature Gating:**
- Access to manufacturing sensitivity analysis and production specification tools
- Statistical tolerance analysis showing NVH variability from production scatter
- End-of-line test specification generators; full simulation tools restricted

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Configure manufacturing context: stamped panels, molded trim, assembled systems with tolerance data
- Walkthrough: assess NVH impact of body panel thickness variation from production stamping
- Tutorial on creating end-of-line NVH test specifications from simulation-derived targets

**Key Workflows:**
- Analyzing NVH sensitivity to manufacturing tolerances (panel thickness, mount stiffness, seal compression)
- Developing end-of-line NVH test specifications with pass/fail boundaries from simulation
- Creating assembly specifications for acoustic-critical components (mount torque, seal gaps, treatment adhesion)
- Troubleshooting production NVH issues using rapid simulation-aided root cause analysis
- Establishing statistical process control limits for NVH-critical manufacturing parameters

### Regulatory/Compliance
**Modified UI/UX:**
- Vehicle noise regulation compliance dashboard (EU 540/2014, ECE R51.03, FMVSS, China GB)
- Pass-by noise prediction tool with regulatory test procedure simulation (ISO 362-1 ASEP)
- Interior noise standard tracker for occupational exposure limits and consumer comfort standards

**Feature Gating:**
- Read-only simulation access; full regulatory compliance checking and documentation tools
- Automated pass-by noise prediction per regulatory test procedures
- Type approval documentation generator with required test evidence formatting

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Configuration of applicable regulations by market (EU, US, China, Japan, India) and vehicle category
- Setup of regulatory compliance tracking templates and homologation submission workflows
- Walkthrough: predict exterior pass-by noise and assess compliance margin for a sample vehicle

**Key Workflows:**
- Predicting exterior pass-by noise levels for regulatory compliance assessment
- Tracking compliance against evolving noise regulations across global markets
- Generating type approval noise documentation packages for homologation submission
- Assessing ASEP (Additional Sound Emission Provisions) compliance for electric/hybrid vehicles
- Managing AVAS (Acoustic Vehicle Alerting System) design requirements for pedestrian safety regulations

### Manager/Decision-maker
**Modified UI/UX:**
- Vehicle program NVH status dashboard with target vs. actual performance across all NVH attributes
- Technology and material cost trade-off visualizer for NVH improvement options
- Competitive benchmarking summary comparing NVH performance against competitor vehicles

**Feature Gating:**
- Dashboard and reporting access; no direct simulation or analysis tools
- NVH target tracking and competitive positioning views
- Resource allocation and program timing visibility for NVH development activities

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Configuration of vehicle program NVH targets with milestone checkpoints (mule, prototype, pre-production)
- Integration with vehicle program management and cost estimation systems
- Setup of automated NVH program status reporting and review cadence

**Key Workflows:**
- Monitoring NVH target achievement progress across vehicle program development milestones
- Evaluating NVH improvement proposals based on cost, weight, timing, and performance impact
- Reviewing competitive benchmarking data to set appropriate NVH targets for new programs
- Allocating NVH engineering resources across concurrent vehicle programs
- Making go/no-go decisions on NVH countermeasures based on simulation-predicted effectiveness vs. cost
