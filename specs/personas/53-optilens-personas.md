# OptiLens — Persona Subspecs

> Parent spec: [`specs/53-optilens.md`](../53-optilens.md)

---

## Experience-Level Personas

### Student/Academic

**Modified UI/UX:**
- Simplified lens-data editor with predefined element types (singlet, doublet, mirror) and drag-to-reorder stacking
- Interactive ray-fan and spot-diagram panels with annotations explaining Seidel aberrations in plain language
- Built-in optics primer linking each aberration type to its physical cause and correction strategies

**Feature Gating:**
- System limited to 10 surfaces; sequential ray tracing only (no non-sequential)
- Glass catalogue limited to common Schott types; no custom material editor
- Optimisation limited to damped least squares with max 5 variables; global search disabled

**Pricing Tier:** Free tier (educational license with .edu verification)

**Onboarding Flow:**
- "First Lens" tutorial: design a cemented doublet achromat from scratch, optimise, and evaluate MTF
- Interactive Seidel aberration explorer: adjust curvatures and see each aberration term change in real time
- Sandbox library with classic textbook designs (Cooke triplet, Petzval, Tessar) for study

**Key Workflows:**
- Design a simple telescope objective and evaluate chromatic aberration correction
- Trace rays through a prism spectrometer and compute angular dispersion
- Compare spot diagrams before and after optimisation to understand merit function behaviour
- Generate lab-report-ready plots of ray fans, wavefront maps, and MTF curves

---

### Junior Engineer (0-3 years)

**Modified UI/UX:**
- Standard workspace with specification-driven design wizard: enter f/#, FOV, wavelength band, and sensor size to get starting-point designs
- Tolerance analysis panel with guided sensitivity table and compensator selection assistant
- Warning indicators on surfaces violating manufacturability rules (centre thickness, edge thickness, clear aperture)

**Feature Gating:**
- Up to 50 surfaces; sequential ray tracing with tilts and decentres
- Full glass catalogue access; basic non-sequential for stray-light feasibility checks
- Damped least squares and global optimisation (simulated annealing) enabled; no custom merit function scripting

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Project-type selector (imaging, illumination, fibre coupling, laser beam delivery) pre-loads appropriate defaults
- Guided walkthrough of a machine-vision lens design from spec through tolerance to fabrication drawing
- Prompted to connect to firm's preferred glass suppliers and coating vendor catalogues

**Key Workflows:**
- Design a machine-vision lens meeting resolution and distortion specs for a given sensor
- Perform tolerance analysis and identify critical dimensions requiring tight manufacturing controls
- Evaluate ghost reflections and narcissus for an infrared imaging system
- Generate optical fabrication drawings with surface specifications per ISO 10110
- Import customer requirements and produce a trade-study comparing two or three design forms

---

### Senior/Expert

**Modified UI/UX:**
- Multi-configuration editor with zoom/focus/thermal tabs, macro scripting console (Python/ZPL-compatible), and batch optimiser
- Custom merit function builder with user-defined operands, boundary constraints, and multi-objective Pareto optimisation
- Interferometric data import panel for as-built wavefront feedback and model correlation

**Feature Gating:**
- Unlimited surfaces; full sequential and non-sequential solvers including physical optics propagation (POP)
- Freeform surface types (XY polynomials, Zernike, NURBS, Q-type); diffractive and meta-surface modelling
- Full API, plugin SDK, and integration with CAD (SolidWorks, Creo), thermal FEA, and structural FEA tools

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Import existing Zemax/Code V/FRED lens files with automatic surface and glass translation
- Configure preferred optimisation algorithms, ray density defaults, and compute cluster settings
- Power-user option to bypass tutorials and open blank multi-configuration project

**Key Workflows:**
- Design a freeform head-mounted display optic with distortion pre-compensation for a curved waveguide
- Perform stray-light analysis of a space-borne imaging spectrometer using non-sequential Monte Carlo tracing
- Run multi-configuration athermalization across -40 C to +70 C ensuring diffraction-limited performance
- Develop custom optical surfaces and coatings for next-generation AR/VR systems via scripting API
- Coordinate opto-mechanical-thermal analysis by coupling optical model with FEA stress and thermal deformation data

---

## Role-Based Personas

### Designer/Creator

**Modified UI/UX:**
- Visual lens layout editor with real-time ray trace overlay; drag lens elements to reshape and watch spot diagram update
- Design-form gallery with thumbnail previews and performance envelopes for common architectures (double Gauss, retrofocus, Petzval)
- Coating stack designer with reflectance/transmittance spectrum preview and environmental durability ratings

**Feature Gating:**
- Full geometry editing, surface types, and glass selection tools enabled
- Real-time optimisation feedback during interactive design sessions
- Export to optical fabrication drawings (ISO 10110), CAD models (STEP/IGES), and tolerance reports

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Specification-driven wizard: enter application requirements and receive ranked starting-point designs
- Tour of glass map and selection tools with Abbe diagram, thermal properties, and cost/availability filters
- Demo of coating optimisation for a broadband AR coating on a consumer camera lens

**Key Workflows:**
- Explore design forms for a wide-angle automotive camera lens and select the best trade-off
- Optimise a zoom lens cam curve for smooth focal-length transition with maintained image quality
- Design and optimise multi-layer anti-reflection coatings matched to system spectral requirements
- Create a compact illumination optic (TIR lens or reflector) for an LED module
- Generate complete optical prescriptions and drawings for handoff to fabrication vendors

---

### Analyst/Simulator

**Modified UI/UX:**
- Analysis-heavy workspace with tiled panels: MTF, wavefront, encircled energy, PSF, and Strehl ratio simultaneously visible
- Monte Carlo tolerance simulation dashboard with yield histograms and worst-offender surface ranking
- Physical optics propagation workspace for diffraction, interference, and coherent beam analysis

**Feature Gating:**
- All analysis solvers enabled: geometric, diffraction-based, POP, and polarisation ray tracing
- Monte Carlo tolerance simulation with thousands of trials and statistical yield prediction
- Scripted analysis pipelines and automated report generation

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Validation suite running NIST-traceable test cases (point-source diffraction, thin-lens benchmarks)
- Import reference designs and measured data for model correlation setup
- Configure default analysis grids, sampling densities, and convergence criteria

**Key Workflows:**
- Perform Monte Carlo tolerance analysis predicting manufacturing yield for a high-volume consumer lens
- Conduct stray-light analysis identifying ghost paths and quantifying veiling glare index
- Evaluate polarisation aberrations in a lithographic projection lens
- Model coherent beam propagation through a fibre-coupled laser system with POP
- Produce peer-review-ready analysis reports with uncertainty estimates for critical optical performance metrics

---

### Manufacturing/Process

**Modified UI/UX:**
- Fabrication-centric view showing each element with testplate fit, surface form error budgets, and wedge/decenter tolerances
- Vendor quote integration panel linking tolerance specifications to estimated cost and lead time
- Assembly sequence editor with alignment procedure steps and compensator adjustment ranges

**Feature Gating:**
- Tolerance analysis and sensitivity tools fully enabled; optimisation tools available but secondary
- Testplate fitting algorithm and melt-data substitution tools for glass-to-as-delivered matching
- Integration with CMM/interferometer data for as-built model updates

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Import optical design and auto-generate fabrication-ready element drawings with ISO 10110 annotations
- Guided testplate selection workflow for each surface
- Connect to shop-floor measurement instruments for automated as-built data feedback

**Key Workflows:**
- Allocate tolerances across elements to meet system MTF at target manufacturing yield and cost
- Perform testplate fitting and assess residual wavefront error for each surface
- Evaluate impact of glass melt data variations on system performance and compensate via airspace adjustment
- Define assembly alignment procedures with compensator motions and acceptance criteria
- Track as-built lens performance against specification through production lot history

---

### Regulatory/Compliance

**Modified UI/UX:**
- Standards compliance dashboard covering laser safety (IEC 60825), medical device optics (ISO 13485), and export control (ITAR/EAR)
- Laser hazard classification calculator integrated with beam propagation model
- Documentation control panel with revision history, approval workflows, and audit trails

**Feature Gating:**
- Safety analysis tools (MPE calculations, hazard zone mapping) fully enabled
- Design history file (DHF) generator for FDA 21 CFR 820 medical device submissions
- Export control classification assistant for optical components and systems

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Regulatory framework selector: choose applicable standards (IEC 60825, ISO 13485, MIL-PRF-13830)
- Walkthrough of laser safety classification for a medical laser delivery system
- Configure document templates for regulatory submissions with required fields and approval gates

**Key Workflows:**
- Classify a laser product per IEC 60825-1 and generate safety label requirements
- Produce optical design documentation meeting ISO 13485 design control requirements for medical devices
- Evaluate ITAR/EAR classification for optical components with military/space applications
- Generate MIL-PRF-13830 surface quality inspection criteria and acceptance test procedures
- Maintain design history file linking requirements to design outputs, verification, and validation records

---

### Manager/Decision-maker

**Modified UI/UX:**
- Program dashboard showing optical design projects with milestone tracking, cost status, and technical risk flags
- Trade-study comparison cards summarising performance, cost, schedule, and risk for competing design approaches
- Supply chain visibility panel showing glass availability, coating vendor lead times, and fabrication capacity

**Feature Gating:**
- Read-only access to optical designs and analysis results; cannot modify prescriptions
- Full access to project tracking, cost estimation, and resource planning modules
- Vendor management and procurement workflow tools enabled

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Portfolio setup linking to programme management tools (MS Project, Jira, Smartsheet)
- Dashboard configuration selecting KPIs (design maturity, tolerance yield, vendor delivery performance)
- Demo of cost-performance trade-off analysis for a lens design programme

**Key Workflows:**
- Evaluate cost/performance/schedule trade-offs between glass and plastic optics for a consumer product
- Track optical design maturity through gate reviews (conceptual, preliminary, detailed, production)
- Manage vendor qualification and selection for precision optical fabrication
- Review tolerance yield predictions to assess manufacturing risk before committing to production tooling
- Allocate optical engineering resources across programmes based on complexity and deadline
