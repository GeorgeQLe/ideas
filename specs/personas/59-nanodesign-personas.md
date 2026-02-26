# NanoDesign — Persona Subspecs

> Parent spec: [`specs/59-nanodesign.md`](../59-nanodesign.md)

---

## Experience-Level Personas

### Student/Academic

**Modified UI/UX:**
- Simplified atomic structure builder with pre-built nanostructures (nanowires, quantum dots, graphene nanoribbons, CNTs)
- Inline quantum mechanics primer explaining band structure, density of states, and quantum transport (Landauer formalism) with visual aids
- Results panel defaults to band structure, transmission function, and I-V characteristics with annotated quantum features

**Feature Gating:**
- Structure limited to 500 atoms; tight-binding Hamiltonian only (no DFT)
- Pre-built material models for Si, GaAs, graphene, and simple heterostructures; custom potential editor locked
- Export to PDF and CSV; no TCAD or foundry PDK interop

**Pricing Tier:** Free tier (educational license)

**Onboarding Flow:**
- "First Nanodevice" tutorial: build a graphene nanoribbon, compute band structure, and calculate ballistic conductance
- Interactive band-structure explorer: vary ribbon width and edge type (armchair vs. zigzag) and see bandgap change in real time
- Sandbox with textbook problems (resonant tunnelling diode, quantum well, Aharonov-Bohm ring)

**Key Workflows:**
- Compute the band structure of a graphene nanoribbon and identify the bandgap dependence on width
- Calculate the transmission function and conductance of a molecular junction
- Study resonant tunnelling through a double-barrier heterostructure
- Generate figures of band structure, DOS, and transmission for a thesis or coursework report

---

### Junior Engineer (0-3 years)

**Modified UI/UX:**
- Standard workspace with device-type wizard: select application (transistor, sensor, memory, photovoltaic) and material system
- Guided NEGF setup assistant with recommended basis sets, contact self-energies, and convergence parameters
- Warning indicators on simulations with unconverged self-consistent loops or insufficient k-point sampling

**Feature Gating:**
- Structure up to 10,000 atoms; semi-empirical tight-binding and extended Huckel methods
- NEGF solver for coherent and incoherent transport; basic DFT available for small systems (under 200 atoms)
- Import from CIF/XYZ/VASP formats; export to TCAD boundary conditions and compact model parameters

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Application selector (MOSFET scaling, 2D material device, molecular electronics, thermoelectrics) pre-loads appropriate models
- Walkthrough of a silicon nanowire FET simulation: build structure, set up NEGF-Poisson, compute I-V characteristics
- Prompted to connect to group's material parameter database and publication reference library

**Key Workflows:**
- Simulate a gate-all-around silicon nanowire FET and extract subthreshold swing, DIBL, and on/off ratio
- Evaluate a 2D material (MoS2, WS2) transistor for next-generation logic scaling feasibility
- Model a molecular sensor device and predict conductance change upon target molecule binding
- Compute thermoelectric figure of merit (ZT) for a nanostructured material
- Generate device performance comparison tables for internal design reviews

---

### Senior/Expert

**Modified UI/UX:**
- Multi-method workspace with DFT, NEGF, GW, and DMFT solvers accessible from unified interface; scripting console (Python API)
- Custom Hamiltonian editor for developing new material models, spin-orbit coupling, and electron-phonon interaction terms
- High-throughput screening dashboard for automated exploration of material/geometry parameter spaces

**Feature Gating:**
- Unlimited system size (with HPC); full DFT (LDA, GGA, hybrid functionals), NEGF, GW quasiparticle corrections, and DMFT
- Phonon transport, electron-phonon coupling, spin transport, and time-dependent quantum transport solvers
- Full API, HPC cluster support, and integration with QuantumATK, VASP, Gaussian, and TCAD tools

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Import existing QuantumATK/Sentaurus/NanoTCAD ViDES projects with automated model and parameter translation
- Configure DFT pseudopotential libraries, basis sets, and HPC queue connections
- Power-user preset opening scripted high-throughput materials screening workspace

**Key Workflows:**
- Perform first-principles DFT+NEGF simulation of a sub-5nm gate-all-around transistor with atomistic channel
- Develop a new tight-binding parameterisation for a novel 2D material using DFT band structure fitting
- Conduct high-throughput screening of material/geometry combinations for optimal thermoelectric or spintronic performance
- Model quantum decoherence and phonon scattering effects on transport in molecular electronic devices
- Automate device scaling studies generating ITRS-style performance projections for emerging device architectures

---

## Role-Based Personas

### Designer/Creator

**Modified UI/UX:**
- Visual atomic structure editor with 3D manipulation, surface passivation tools, and heterostructure stacking interface
- Device architecture explorer with thumbnails of common nanodevice topologies (GAA, FinFET, tunnel FET, 2D heterostructure)
- Material property browser with bandgap, effective mass, mobility, and lattice constant filters linked to structure builder

**Feature Gating:**
- Full atomic structure editing, surface reconstruction, and heterostructure assembly tools
- Quick-solve mode for rapid screening of design variants; full DFT+NEGF requires explicit submission
- Export to atomistic visualisers (VESTA, XCrySDen), TCAD, and publication-quality structure renderings

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Device-type selector pre-loads template structure with recommended dimensions and material system
- "Material Stacking" guide for building van der Waals heterostructures with twist angle and interlayer coupling control
- Tour of the structure optimisation tools using a sample InGaAs/InP quantum well laser active region

**Key Workflows:**
- Design a van der Waals heterostructure device (graphene/hBN/MoS2) with optimised layer thicknesses and alignment
- Explore FinFET fin width and height scaling on electrostatic integrity and quantum confinement
- Create a resonant tunnel diode structure with engineered quantum well width and barrier composition
- Design a core-shell nanowire heterostructure for strain-engineered bandgap tuning
- Generate atomistic device structures for handoff to simulation analysts

---

### Analyst/Simulator

**Modified UI/UX:**
- Computation-focused workspace with band structure plotters, transmission viewers, spectral function maps, and convergence trackers
- Multi-method comparison panel showing tight-binding vs. DFT vs. GW results with quantified discrepancies
- Post-processing suite for extracting device figures of merit (SS, DIBL, gm, fT) from quantum transport results

**Feature Gating:**
- All solver methods and approximation levels fully unlocked
- Advanced post-processing: spectral current density, local density of states, Seebeck coefficient extraction
- HPC job management with automated convergence studies (k-points, basis size, self-consistency tolerance)

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Validation suite running community benchmarks (Si bandgap, graphene Dirac cone, molecular junction conductance)
- Import reference calculations and experimental data for model calibration
- Configure default computational parameters, pseudopotentials, and parallel execution settings

**Key Workflows:**
- Perform systematic convergence studies on k-point sampling and basis set quality for a target device
- Compare semi-empirical and first-principles predictions for the same device to assess accuracy/cost trade-offs
- Extract energy-resolved transmission and local current pathways to understand quantum interference effects
- Model gate leakage current through ultra-thin dielectrics using direct tunnelling formalism
- Produce peer-reviewed publications with validated computational methodology and reproducibility documentation

---

### Manufacturing/Process

**Modified UI/UX:**
- Process-centric view linking fabrication steps (epitaxy, lithography, etching, doping) to atomistic device structure
- Process variability dashboard showing how line-edge roughness, doping fluctuation, and thickness variation affect device performance
- TCAD bridge panel mapping atomistic results to continuum models for circuit-level simulation

**Feature Gating:**
- Statistical variability analysis tools (random dopant fluctuation, LER, interface roughness)
- Compact model parameter extraction for BSIM, PSP, or custom models from quantum transport results
- Integration with TCAD tools (Sentaurus, Silvaco) and foundry PDK generation workflows

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Process flow import: define fabrication steps and auto-generate atomistic structures reflecting process-induced features
- Guided tutorial on random dopant fluctuation analysis for a 5nm node MOSFET
- Connect to foundry process monitoring data for model calibration against silicon measurements

**Key Workflows:**
- Quantify the impact of random dopant fluctuation on threshold voltage variability for sub-7nm nodes
- Evaluate line-edge roughness effects on FinFET or nanosheet transistor performance distributions
- Extract compact model parameters from atomistic NEGF simulations for circuit-level SPICE models
- Assess process-induced strain effects on carrier transport using atomistic strain maps
- Generate process-device-variation sensitivity reports for technology development teams

---

### Regulatory/Compliance

**Modified UI/UX:**
- Export control and IP classification dashboard for novel nanomaterials and quantum device technologies
- Publication and data-sharing compliance panel tracking restrictions on dual-use technology disclosures
- Computational reproducibility tracker logging all software versions, parameters, and random seeds

**Feature Gating:**
- Export control classification assistant for semiconductor IP (EAR, Wassenaar Arrangement categories)
- Data management tools ensuring compliance with research data policies (FAIR principles, funder mandates)
- Computational audit trail with complete parameter and version logging for regulatory reproducibility

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Regulatory framework selector: export control jurisdiction, funding agency data policies, publication restrictions
- Walkthrough of export control classification for a novel semiconductor device technology
- Configure data management policies for computational results retention and sharing

**Key Workflows:**
- Classify novel nanodevice technologies under EAR/ECCN for export control compliance
- Ensure computational research data meets funder open-data requirements (NSF, DOE, EU Horizon)
- Document simulation methodology with sufficient detail for regulatory or patent examination reproducibility
- Track IP provenance for novel device architectures from concept through simulation to publication/patent
- Generate compliance documentation for dual-use technology review boards

---

### Manager/Decision-maker

**Modified UI/UX:**
- Technology roadmap dashboard showing device scaling projections, research milestones, and competitive landscape
- Investment prioritisation matrix mapping research directions to market impact, feasibility, and resource requirements
- Publication and patent portfolio tracker linked to simulation-derived innovations

**Feature Gating:**
- Read-only access to simulation results and device performance projections; cannot modify atomic structures or run solvers
- Full access to technology benchmarking, roadmap planning, and research portfolio management tools
- Competitive intelligence dashboards comparing internal capabilities to published state-of-the-art

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Technology portfolio import linking research programmes to ITRS/IRDS roadmap targets
- Dashboard configuration selecting KPIs (device performance vs. roadmap, publication impact, patent coverage)
- Demo of technology readiness assessment for a sample emerging device programme

**Key Workflows:**
- Evaluate technology readiness of emerging nanodevice concepts against ITRS/IRDS performance targets
- Prioritise research investment across competing device architectures based on simulation-derived scaling projections
- Track research team productivity and publication/patent output linked to simulation capabilities
- Communicate technology risk and opportunity to corporate leadership and funding agencies
- Make go/no-go decisions on transitioning research device concepts to technology development phase
