# CompForge — Persona Subspecs

> Parent spec: [`specs/66-compforge.md`](../66-compforge.md)

---

## Experience-Level Personas

### Student/Academic

**Modified UI/UX:**
- Guided laminate design mode with inline explanations of classical lamination theory (CLT), stacking sequence notation, and failure criteria
- Interactive Mohr's circle and stress-strain visualization for individual plies and the full laminate
- Pre-built material library with common fiber/resin systems (T300/5208, AS4/3501-6, E-glass/epoxy) and their published properties

**Feature Gating:**
- CLT analysis for flat laminates; 3D FEA limited to simple plate/shell geometries with <100K elements
- Standard failure criteria (Tsai-Wu, Tsai-Hill, max stress, max strain, Hashin); progressive damage analysis locked
- Export to academic formats (plots, CSV); no manufacturing flat pattern or ply book generation

**Pricing Tier:** Free tier (academic email verification)

**Onboarding Flow:**
- Start with a guided quasi-isotropic laminate analysis tutorial covering ply-by-ply stress, ABD matrix, and failure envelope
- Interactive failure criteria comparison exercise showing differences between Tsai-Wu, Hashin, and max stress on the same laminate
- Access to published test data for validating CLT predictions against coupon test results

**Key Workflows:**
- Designing and analyzing flat laminate stacking sequences for composites coursework
- Comparing failure criteria predictions and understanding their assumptions and limitations
- Performing parametric studies on fiber orientation and volume fraction for thesis research
- Generating laminate polar diagrams and carpet plots for design space exploration

---

### Junior Engineer (0-3 years)

**Modified UI/UX:**
- Template-based laminate design organized by application (wing skin, fuselage panel, wind blade spar, automotive panel)
- Stacking sequence advisor recommending balanced/symmetric layups and flagging common design rule violations
- Side-by-side view comparing hand layup vs. AFP vs. RTM manufacturing feasibility for the designed laminate

**Feature Gating:**
- Full CLT and FEA analysis with standard failure criteria and first-ply failure prediction
- Progressive damage analysis available with guided setup; custom material model development locked
- Ply book generation enabled; producibility analysis (draping, fiber steering) requires senior review

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Select application domain (aerospace, wind energy, automotive, marine) for tailored material libraries and design guidelines
- Guided design of a composite panel with loading, layup definition, sizing, and failure check walkthrough
- Tutorial on interpreting draping simulation results and their impact on structural performance

**Key Workflows:**
- Designing laminate stacking sequences meeting stiffness and strength requirements for structural applications
- Running FEA with composite-specific failure criteria for detailed stress analysis of curved structures
- Evaluating manufacturing feasibility through draping simulation for complex geometries
- Generating ply books and flat patterns for manufacturing hand-off
- Running trade studies comparing material systems (carbon vs. glass, thermoset vs. thermoplastic)

---

### Senior/Expert

**Modified UI/UX:**
- Full scripting API (Python) for custom failure criteria, progressive damage models, and automated optimization loops
- Multi-scale analysis workspace linking micro-mechanics, ply-level, and structural-level simulations
- Certification documentation generator aligned with CMH-17, JSSG-2006, and AC 20-107B requirements

**Feature Gating:**
- All features unlocked: progressive damage, impact damage tolerance (BVID/CAI), fatigue, environmental knockdowns
- Custom material model development with virtual coupon testing and building block validation
- AFP/ATL path planning optimization and full manufacturing process simulation (autoclave, RTM, infusion)

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Import existing HyperSizer, Fibersim, or ANSYS ACP models with automated property and layup translation
- Benchmark against published test data (WWFE, CMH-17 datasets) for failure prediction validation
- Configure automated building block analysis pipeline from coupon through element to full-scale structure

**Key Workflows:**
- Developing building block test/analysis correlation from coupon to full-scale for certification
- Running progressive damage analysis for impact damage tolerance (BVID, CAI, damage growth under fatigue)
- Optimizing AFP/ATL fiber paths for complex curved structures with steering constraints
- Performing environmental knockdown analysis (temperature, moisture) per certification requirements
- Leading composite certification campaigns with integrated test-analysis documentation

---

## Role-Based Personas

### Designer/Creator

**Modified UI/UX:**
- Visual laminate editor with ply-by-ply stacking sequence builder, rosette definition, and zone management
- 3D draping preview showing fiber orientations on complex curved surfaces with wrinkle/bridging indicators
- Design variant comparison with structural performance overlay (stiffness, strength, weight, cost)

**Feature Gating:**
- Full laminate design tools with zone definition, ply drops, and taper transitions
- Automated ply stacking sequence optimization (genetic algorithm, gradient-based) with manufacturing constraints
- AFP/ATL path planning with steering limits, gap/overlap control, and course generation

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Import part CAD geometry and define composite zones with loading and design requirements
- Walk through a laminate optimization study demonstrating weight savings from automated sizing
- Tutorial on ply drop and taper design for manufacturing-friendly thickness transitions

**Key Workflows:**
- Designing laminate zone maps with optimized stacking sequences for minimum weight
- Planning ply drop sequences for thickness transitions that maintain structural integrity
- Generating AFP/ATL fiber paths with steering constraints for automated manufacturing
- Creating ply books and flat patterns with labeling and nesting for production
- Evaluating design trade-offs between structural performance, weight, and manufacturing complexity

---

### Analyst/Simulator

**Modified UI/UX:**
- Analysis workspace with failure envelopes, damage maps, and margin of safety plots front and center
- Multi-load case manager with envelope analysis across hundreds of flight conditions
- Building block dashboard tracking analysis status from coupon through element, detail, and full-scale levels

**Feature Gating:**
- All analysis capabilities available based on license tier
- Advanced post-processing: progressive damage evolution, delamination growth (VCCT, cohesive zone), fatigue damage accumulation
- Batch analysis with automated load case sweep and critical case identification

**Pricing Tier:** Professional or Enterprise tier

**Onboarding Flow:**
- Tutorial on selecting appropriate failure criteria for different analysis levels (screening vs. detailed)
- Guided progressive damage analysis of an open-hole tension specimen with experimental validation
- Introduction to building block methodology for composite certification analysis

**Key Workflows:**
- Running strength analysis across hundreds of load cases to identify critical conditions and margins of safety
- Performing progressive damage analysis to predict ultimate failure loads and modes
- Analyzing delamination initiation and growth using VCCT and cohesive zone methods
- Conducting fatigue damage accumulation analysis under spectrum loading
- Generating analysis reports compliant with certification requirements (AC 20-107B, CMH-17)

---

### Manufacturing/Process

**Modified UI/UX:**
- Manufacturing-focused interface with flat pattern layouts, ply cutting plans, and layup sequence instructions
- Draping simulation results displayed as producibility maps with wrinkle severity and fiber angle deviation
- Process monitoring dashboard tracking autoclave cure cycles, RTM fill progress, and quality metrics

**Feature Gating:**
- Draping simulation and flat pattern generation; no structural analysis modification
- AFP/ATL path generation with machine-specific constraints (head width, steering radius, tow gaps)
- Process simulation (autoclave cure, RTM fill, spring-back prediction) for manufacturing optimization

**Pricing Tier:** Professional tier (manufacturing license)

**Onboarding Flow:**
- Configure manufacturing process chain (hand layup, AFP, RTM, autoclave, oven) with machine-specific parameters
- Walk through generating a complete ply book from an approved laminate design
- Tutorial on interpreting draping simulation results for layup sequence planning

**Key Workflows:**
- Generating optimized flat patterns and nesting layouts for minimal material waste
- Running draping simulations to identify producibility issues before tool manufacture
- Simulating autoclave cure cycles to predict residual stress and spring-back for tool compensation
- Programming AFP/ATL machines from optimized fiber paths with collision-free tool paths
- Supporting manufacturing nonconformance review with simulation-based impact assessment

---

### Regulatory/Compliance

**Modified UI/UX:**
- Certification workspace organized by regulatory framework (FAA AC 20-107B, EASA CS, MIL-HDBK-17/CMH-17)
- Traceability matrix linking every analysis result to material allowables, test data, and design requirements
- Building block evidence tracker showing analysis-test correlation at each level

**Feature Gating:**
- Read-only access to analysis results, material databases, and building block documentation
- Automated certification report generation per applicable regulatory guidance
- Configuration-controlled material allowable database with batch traceability

**Pricing Tier:** Enterprise tier (certification add-on)

**Onboarding Flow:**
- Select certification basis (FAA, EASA, military) to auto-configure documentation requirements
- Walk through a building block certification package from coupon data through full-scale substantiation
- Configure material allowable database with statistical basis (B-basis, A-basis) documentation

**Key Workflows:**
- Assembling composite certification documentation per AC 20-107B and CMH-17 guidelines
- Reviewing building block analysis-test correlation for each structural level
- Managing material allowable databases with proper statistical basis and environmental knockdowns
- Documenting damage tolerance and fatigue substantiation for composite primary structure
- Responding to certification authority questions on composite analysis methodology and margins

---

### Manager/Decision-maker

**Modified UI/UX:**
- Program dashboard with composite structure weight tracking, certification progress, and cost metrics
- Make-buy analysis view comparing in-house composite manufacturing vs. outsourcing based on simulation data
- Technology readiness assessment for new material systems and manufacturing processes

**Feature Gating:**
- View-only access to project summaries, weight statements, certification milestone status
- Portfolio view across composite programs with schedule risk and weight growth tracking
- Budget allocation for material testing, simulation compute, and certification activities

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Configure program portfolio with weight targets, certification milestones, and budget allocations
- Walk through interpreting composite design trade-off summaries (weight vs. cost vs. manufacturing risk)
- Set up automated reporting for weight tracking and certification progress

**Key Workflows:**
- Monitoring composite structure weight against targets across design iterations
- Reviewing certification milestone progress and identifying schedule risks
- Approving material system selections based on performance, cost, and supply chain considerations
- Evaluating composite vs. metallic trade-offs for structural components
- Generating program status reports for senior leadership on composite development programs
