# TextileLab — Persona Subspecs

> Parent spec: [`specs/72-textilelab.md`](../72-textilelab.md)

---

## Experience-Level Personas

### Student/Academic
**Modified UI/UX:**
- Visual weave pattern editor with drag-and-drop yarn placement and real-time 3D fabric preview
- Annotated diagrams explaining warp/weft interlacing, crimp, and unit cell geometry
- Simplified material property inputs using common textile fiber presets (cotton, polyester, carbon, glass)

**Feature Gating:**
- Basic weave/knit/braid pattern generation and unit cell analysis unlocked
- Mechanical property prediction limited to linear elastic; failure and drape simulation locked
- Export to academic formats (images, CSV, PDF reports); industrial data exchange formats restricted

**Pricing Tier:** Free tier (academic license)

**Onboarding Flow:**
- Guided creation of a plain weave unit cell with step-by-step yarn geometry definition
- Pre-loaded examples: common weave patterns (twill, satin) with annotated structural parameters
- Link to textile engineering course material repository and community templates

**Key Workflows:**
- Creating and visualizing woven fabric unit cells for materials science coursework
- Predicting elastic moduli of woven composites using classical laminate theory
- Comparing fabric architectures (plain vs. twill vs. satin) through parametric studies
- Generating publication-ready fabric geometry visualizations

### Junior Engineer (0-3 years)
**Modified UI/UX:**
- Template-driven workflow: select fabric type, define yarn properties, generate unit cell, run analysis
- Guided parameter entry with range validation and industry-standard defaults for common fabrics
- Comparison dashboard to evaluate multiple fabric architectures side by side

**Feature Gating:**
- Full mechanical property prediction including stiffness, strength estimates, and permeability
- Drape simulation enabled with guided setup; advanced forming simulation requires approval
- Access to standard material database; custom material calibration tools restricted

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Setup wizard asking application domain (apparel, composites, filtration, geotextiles)
- Walkthrough: import yarn test data, build unit cell, predict fabric properties, validate against coupon test
- Introduction to company material library and design review submission

**Key Workflows:**
- Building unit cell models from yarn specifications and weave pattern definitions
- Predicting mechanical properties (tensile, shear, bending) of fabric architectures
- Running permeability simulations for resin transfer molding process design
- Generating material cards for downstream structural FEA tools
- Documenting fabric design rationale for design review packages

### Senior/Expert
**Modified UI/UX:**
- Multi-scale modeling console linking yarn micro-mechanics to fabric meso-scale to component macro-scale
- Custom constitutive model editor for advanced yarn/fabric behavior (viscoelastic, damage, friction)
- Scripting interface for automated parametric sweeps and optimization across fabric architectures

**Feature Gating:**
- All modules: advanced drape/forming simulation, damage modeling, multi-physics coupling
- Full API for integration with FEA solvers (Abaqus, LS-DYNA) and manufacturing simulation tools
- Admin controls for proprietary material databases, custom model libraries, and IP protection

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Migration assistant for importing WiseTex/TexGen models and legacy material characterization data
- Configuration of multi-scale analysis chains linking micro, meso, and macro models
- Setup of automated validation workflows against experimental test databases

**Key Workflows:**
- Multi-scale modeling from fiber properties to full-component structural response
- Calibrating advanced constitutive models against biaxial and shear frame test data
- Optimizing fabric architectures for specific forming operations (stamp forming, RTM)
- Developing custom analysis methodologies and validating against experimental campaigns
- Managing team-wide material databases and enforcing modeling best practices

---

## Role-Based Personas

### Designer/Creator
**Modified UI/UX:**
- Visual pattern design studio with color-coded yarn paths, interlacing visualization, and symmetry tools
- Real-time 3D fabric rendering with texture mapping for visual prototyping
- Yarn library browser with filtering by fiber type, linear density, twist, and surface treatment

**Feature Gating:**
- Full access to pattern creation, editing, and yarn specification tools
- Aesthetic visualization and virtual prototyping features unlocked
- Export to weaving/knitting machine programming formats (WIF, KnitPaint)

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Interactive tour of pattern design tools using a sample twill weave project
- Import an existing weave pattern file and modify it with the visual editor
- Tutorial on creating yarn color/material combinations for visual prototyping

**Key Workflows:**
- Designing woven, knitted, and braided fabric architectures from specifications
- Creating parametric pattern families for product line development
- Virtual prototyping with realistic fabric rendering for design reviews
- Exporting machine-ready pattern files for loom/knitting machine programming
- Building reusable pattern template libraries for common product families

### Analyst/Simulator
**Modified UI/UX:**
- Analysis workflow manager with meshing controls, solver settings, and convergence monitoring
- Multi-physics coupling dashboard for thermo-mechanical, fluid-structure, or forming simulations
- Result visualization suite with stress/strain contours mapped onto fabric geometry

**Feature Gating:**
- Full solver access for mechanical, thermal, permeability, and drape/forming simulations
- Advanced post-processing including virtual DIC, through-thickness stress extraction
- Access to high-performance computing queue management for large parametric studies

**Pricing Tier:** Professional or Enterprise tier

**Onboarding Flow:**
- Guided first simulation: tensile test on a woven unit cell with validation against analytical solution
- Setup of standard analysis templates matching company procedures
- Introduction to batch job management and result archival workflows

**Key Workflows:**
- Predicting fabric mechanical response under complex loading using meso-scale FEA
- Simulating resin flow through fabric preforms for RTM/VARTM process optimization
- Running drape and forming simulations for composite layup planning
- Validating simulation results against experimental test data with automated metrics
- Performing sensitivity studies on yarn properties, spacing, and architecture parameters

### Manufacturing/Process
**Modified UI/UX:**
- Manufacturability assessment panel checking loom/machine constraints (reed density, shed height, yarn tension)
- Process parameter recommendation engine based on fabric architecture and equipment capabilities
- Quality control dashboard with defect pattern recognition and specification compliance tracking

**Feature Gating:**
- Access to manufacturing feasibility checkers and process simulation modules
- Machine programming file generation and validation tools
- Quality inspection template generation; advanced optimization locked to Enterprise tier

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Configure equipment inventory (loom types, knitting machines, braiding mandrels) and capability limits
- Walkthrough: check a fabric design against machine constraints, generate setup parameters
- Tutorial on integrating quality inspection data back into the design feedback loop

**Key Workflows:**
- Validating fabric designs against weaving/knitting machine capability constraints
- Generating optimized machine setup parameters (tension, speed, take-up rate)
- Simulating resin infusion processes to define flow front and cure schedules
- Tracking production quality data and correlating defects to process parameters
- Creating standard operating procedures from validated process simulations

### Regulatory/Compliance
**Modified UI/UX:**
- Standards compliance dashboard mapping fabric properties to test requirements (ASTM, ISO, EN)
- Traceability matrix linking raw material certificates to final fabric test reports
- Automated compliance report generator with pass/fail status against specification limits

**Feature Gating:**
- Read-only access to design; full access to compliance checking and reporting tools
- Automated property comparison against regulatory specification databases
- Digital record management with tamper-evident audit trails

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Configuration of applicable standards and specifications (aerospace, automotive, medical, construction)
- Setup of material traceability requirements and supplier qualification criteria
- Walkthrough of compliance report generation for a sample fabric qualification

**Key Workflows:**
- Verifying predicted fabric properties meet specification requirements (flammability, strength, porosity)
- Generating qualification test matrices per applicable ASTM/ISO standards
- Maintaining material traceability from fiber lot numbers through finished fabric rolls
- Producing audit-ready compliance documentation for customer or regulatory review
- Tracking specification change notices and assessing impact on qualified materials

### Manager/Decision-maker
**Modified UI/UX:**
- Portfolio dashboard showing product development status, material cost comparisons, and timeline tracking
- Supply chain visualization mapping yarn suppliers to fabric products with risk indicators
- ROI calculator comparing virtual prototyping iterations vs. physical sample costs

**Feature Gating:**
- Dashboard and reporting access only; no direct design or simulation tools
- Cost modeling and trade study comparison views
- Resource utilization and project scheduling visibility

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Configuration of product portfolio structure and development milestones
- Integration with ERP/PLM systems for cost and inventory data
- Setup of automated status reporting cadence and distribution lists

**Key Workflows:**
- Monitoring product development progress across multiple fabric programs
- Evaluating material cost vs. performance trade-offs for fabric selection decisions
- Tracking time-to-market metrics and virtual vs. physical prototyping efficiency
- Reviewing supplier qualification status and supply chain diversification options
- Approving material specifications and design freeze decisions based on data summaries
