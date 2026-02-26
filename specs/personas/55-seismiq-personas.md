# SeismiQ — Persona Subspecs

> Parent spec: [`specs/55-seismiq.md`](../55-seismiq.md)

---

## Experience-Level Personas

### Student/Academic

**Modified UI/UX:**
- Simplified frame editor with drag-and-drop structural elements (beams, columns, braces, walls) and auto-connection
- Inline seismic theory cards explaining response spectrum concepts, ductility factors, and modal analysis fundamentals
- Results panel defaults to mode shapes, storey drifts, and base shear with traffic-light code compliance indicators

**Feature Gating:**
- Model size limited to 10 storeys and 500 nodes; linear elastic analysis only
- Response spectrum analysis with pre-loaded code spectra (ASCE 7, Eurocode 8, IS 1893); no time-history analysis
- Export to PDF reports and CSV; no ETABS/SAP2000 interop

**Pricing Tier:** Free tier (educational license)

**Onboarding Flow:**
- "First Seismic Model" tutorial: build a 3-storey frame, apply code spectrum, run modal analysis, check drift limits
- Interactive mode-shape viewer with period/frequency annotation and mass-participation bar charts
- Sandbox with pre-built textbook structures (portal frame, shear wall, dual system) for parametric exploration

**Key Workflows:**
- Model a reinforced concrete moment frame and perform response spectrum analysis per ASCE 7
- Compare mode shapes and periods for different bracing configurations
- Verify storey drift ratios against code limits and understand torsional irregularity
- Generate academic reports with modal analysis results and base shear distribution

---

### Junior Engineer (0-3 years)

**Modified UI/UX:**
- Standard workspace with code-driven design wizard: select building code, occupancy category, site class, and seismic design category
- Guided load-combination builder with ASCE 7 / Eurocode load factors pre-populated
- Checklist sidebar tracking deliverables (drift check, strength check, irregularity assessment, connection design)

**Feature Gating:**
- Model size up to 50 storeys and 10,000 nodes; linear and basic nonlinear static (pushover) analysis
- Code-based design for concrete (ACI 318) and steel (AISC 341/360) members with auto-detailing
- Import from Revit Structure, Tekla; export to detailing software and calculation packages

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Project-type selector (building, bridge, industrial structure, retrofit) pre-loads applicable codes and analysis defaults
- Walkthrough of a 10-storey office building seismic design from BIM import to drift verification and member sizing
- Prompted to import firm's standard connection details, material grades, and report templates

**Key Workflows:**
- Import a Revit structural model and perform seismic design per ASCE 7 / AISC 341
- Check storey drift, stability coefficients (P-delta), and torsional irregularity for code compliance
- Perform pushover analysis to evaluate structural overstrength and ductility capacity
- Design and detail special moment frame connections per AISC 358 prequalified types
- Produce seismic design calculation package for building permit submission

---

### Senior/Expert

**Modified UI/UX:**
- Advanced workspace with multi-hazard analysis (seismic + wind + blast), nonlinear time-history solver, and performance-based design tools
- Scripting console (Python/Tcl) for parametric studies, fragility curve generation, and OpenSees integration
- Instrumentation dashboard linking strong-motion sensor data to model for structural health monitoring

**Feature Gating:**
- Unlimited model size; full nonlinear dynamic time-history analysis with fibre-section elements and material hysteresis
- Performance-based earthquake engineering (PBEE) tools: IDA, fragility functions, loss estimation (FEMA P-58)
- API access, OpenSees solver kernel integration, HPC cluster support, and seismological database connections

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Import existing ETABS/SAP2000/Perform3D/OpenSees models with automated translation and validation
- Configure ground-motion database connections (PEER NGA-West2, ESM) and site-specific hazard tools
- Power-user preset opening scripted IDA workspace with pre-configured ground-motion suite

**Key Workflows:**
- Perform nonlinear response history analysis with site-specific ground motions selected and scaled per ASCE 7 Ch. 16
- Conduct incremental dynamic analysis (IDA) and generate fragility curves for collapse probability assessment
- Develop performance-based design using FEMA P-58 loss estimation for owner risk communication
- Evaluate seismic retrofit strategies (FRP wrapping, base isolation, viscous dampers) with comparative nonlinear analysis
- Build structural health monitoring digital twin linking accelerometer data to calibrated analytical model

---

## Role-Based Personas

### Designer/Creator

**Modified UI/UX:**
- Visual structural layout editor with floor-plan and 3D perspectives; interactive member sizing with real-time demand/capacity ratios
- Seismic force-resisting system (SFRS) explorer comparing moment frames, braced frames, shear walls, and dual systems
- Connection design studio with graphical detail editors and automatic code-check annotations

**Feature Gating:**
- Full geometry editing, member sizing, and SFRS selection tools enabled
- Code-based auto-design for beams, columns, braces, and shear walls with iterative re-analysis
- Export to Revit Structure, Tekla, and detailing packages (rebar schedules, steel connection drawings)

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Architectural floor plan import with structural grid overlay and gravity-load takedown wizard
- "SFRS Selection" guide comparing system types for a given height, occupancy, and seismic design category
- Tour of connection design studio using a sample beam-column moment connection

**Key Workflows:**
- Lay out a seismic force-resisting system for a mid-rise building and size members to meet drift and strength
- Design special concentrically braced frame (SCBF) connections including gusset plates and brace-to-beam joints
- Evaluate architectural impact of different SFRS choices (frame depth vs. wall locations vs. brace visibility)
- Iterate between architectural constraints and structural performance to achieve an efficient lateral system
- Produce coordinated structural drawings with seismic detailing for construction documentation

---

### Analyst/Simulator

**Modified UI/UX:**
- Analysis-centric workspace with hysteresis loop viewers, IDA curve plotters, and floor-spectrum generators
- Ground-motion selection and scaling tool with spectral matching, CMS targeting, and record metadata filters
- Multi-model comparison panel for evaluating modelling uncertainty (2D vs. 3D, lumped vs. distributed plasticity)

**Feature Gating:**
- All solver types enabled: linear, nonlinear static, nonlinear dynamic, and soil-structure interaction
- Advanced material models: fibre sections, concentrated plasticity hinges, soil springs, isolator/damper elements
- Probabilistic analysis tools: Monte Carlo, Latin Hypercube, and surrogate model construction for fragility

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Solver validation suite running PEER benchmark problems with published experimental comparisons
- Import ground-motion databases and configure site-specific probabilistic seismic hazard analysis (PSHA) interface
- Configure nonlinear solver tolerances, convergence strategies, and output recording preferences

**Key Workflows:**
- Select and scale a suite of ground motions for nonlinear response history analysis per ASCE 7 Chapter 16
- Perform incremental dynamic analysis and quantify collapse margin ratio per FEMA P-695
- Conduct soil-structure interaction analysis with depth-varying soil springs and kinematic effects
- Evaluate floor acceleration spectra for equipment and non-structural component seismic qualification
- Produce technical reports documenting nonlinear modelling assumptions, validation, and results with peer-review traceability

---

### Manufacturing/Process

**Modified UI/UX:**
- Fabrication-centric view showing steel connections, rebar detailing, and precast element schedules
- Constructability checker evaluating rebar congestion, weld access, bolt clearances, and erection sequence
- BIM coordination panel highlighting clashes between structural members and MEP systems

**Feature Gating:**
- Connection design and detailing tools fully enabled; analysis tools available for verification
- Shop drawing generation for steel connections and rebar placement
- Integration with fabrication management systems and CNC data export for steel processing

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Import structural model and auto-generate connection designs based on analysis demands
- Guided walkthrough of a prequalified seismic moment connection from demand extraction to shop drawing
- Connect to firm's steel fabrication standards and preferred connection types

**Key Workflows:**
- Detail seismic connections meeting AISC 341/358 prequalification requirements
- Check rebar placement feasibility in beam-column joints of special moment frames
- Generate shop drawings for steel braced-frame gusset connections with weld specifications
- Evaluate precast concrete connection details for seismic demand transfer
- Produce erection sequence plans considering temporary stability during construction

---

### Regulatory/Compliance

**Modified UI/UX:**
- Code-compliance dashboard organised by building code chapters (ASCE 7, IBC, ACI 318, AISC 341)
- Irregularity detection engine with automatic classification (Type 1a through 5b) and triggered requirements
- Audit trail capturing all analysis assumptions, load derivations, and design decisions with code references

**Feature Gating:**
- Code-checking engine fully enabled covering seismic design category determination through member detailing
- Peer-review documentation generator per ASCE 7 Section 16.4 for nonlinear analysis projects
- Building code change tracker highlighting differences between code editions (ASCE 7-16 vs. 7-22)

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Jurisdiction selector pre-loads applicable building code, amendments, and local seismic hazard maps
- Walkthrough of seismic design category determination and its cascading effects on analysis requirements
- Configure firm-specific report templates with PE seal blocks and jurisdictional submission requirements

**Key Workflows:**
- Determine seismic design parameters (Ss, S1, Fa, Fv, SDS, SD1, SDC) per ASCE 7 for a given site
- Verify structural irregularities and confirm analysis procedure eligibility (ELF vs. modal vs. time-history)
- Produce code-compliance calculation packages for building department plan check
- Assess existing building for seismic evaluation per ASCE 41 and classify performance level achieved
- Generate peer-review documentation for projects requiring independent review per jurisdictional mandate

---

### Manager/Decision-maker

**Modified UI/UX:**
- Portfolio dashboard showing all structural projects with seismic risk rating, design progress, and code-compliance status
- Cost-impact charts comparing seismic force-resisting system options (material tonnage, connection complexity, schedule)
- Risk visualisation showing probable seismic loss (AAL) for building portfolio under various earthquake scenarios

**Feature Gating:**
- Read-only access to structural models and analysis results; cannot modify geometry or loads
- Full access to cost estimation, project tracking, and loss-estimation reporting modules
- Portfolio seismic risk assessment tools for real estate investment decision support

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Portfolio setup linking to project management and real estate asset management systems
- Dashboard configuration selecting KPIs (seismic risk score, design margin, construction cost per sq ft)
- Demo of seismic loss estimation for a campus of buildings using scenario earthquake analysis

**Key Workflows:**
- Compare cost and schedule impacts of different seismic force-resisting system selections during schematic design
- Evaluate seismic retrofit cost-benefit analysis for an existing building portfolio (ASCE 41 + FEMA P-58)
- Review structural engineering deliverable progress and resource allocation across projects
- Communicate seismic risk to building owners and insurance underwriters using probable maximum loss (PML) metrics
- Make go/no-go decisions on property acquisitions based on seismic vulnerability assessments
