# ControlHub — Persona Subspecs

> Parent spec: [`specs/46-controlhub.md`](../46-controlhub.md)

---

## Experience-Level Personas

### Student/Academic

**Modified UI/UX:**
- Visual block-diagram editor with pre-built library of standard blocks (PID, transfer functions, summers, gains, common plants)
- Inline theory panel displaying pole-zero maps, Bode plots, and root locus alongside the block diagram in real time
- Step-by-step "Control Design Assistant" that walks through classical design methodology (model > analyze > design > verify)

**Feature Gating:**
- Classical control tools (root locus, Bode, Nyquist, PID tuning) and linear state-space analysis available
- Nonlinear simulation, code generation, and hardware-in-the-loop locked
- System order limited to 20 states; multi-rate and hybrid (continuous/discrete) simulation unavailable

**Pricing Tier:** Free tier (education license with .edu verification)

**Onboarding Flow:**
- "Control Your First System" tutorial: model a DC motor, design a PID controller, simulate step response, and tune for spec
- Pre-loaded example systems (inverted pendulum, cruise control, temperature regulation, ball-and-beam)
- Interactive exercises mapping lecture concepts (stability margins, steady-state error, transient specs) to tool features

**Key Workflows:**
- Building transfer-function models and analyzing open-loop behavior with Bode/Nyquist/root-locus plots
- Designing PID controllers using graphical tuning tools and verifying step-response specifications
- Comparing classical compensator designs (lead, lag, lead-lag) using interactive root-locus manipulation
- Simulating closed-loop systems with step, ramp, and sinusoidal inputs
- Exporting analysis plots for homework submissions and lab reports

---

### Junior Engineer (0-3 years)

**Modified UI/UX:**
- Professional block-diagram workspace with hierarchical subsystems, signal buses, and library browser
- Auto-tuning wizard for PID and model-predictive controllers with interactive performance/robustness trade-off sliders
- Simulation dashboard with scope displays, data logging, and automated performance-metric extraction

**Feature Gating:**
- Full linear and nonlinear simulation, PID auto-tuning, state-space design, and discrete-time systems unlocked
- Model-predictive control (MPC) available in guided mode with template configurations
- Code generation for embedded targets available with standard templates; custom target configuration restricted

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- System-identification wizard: import measured input/output data and auto-fit transfer-function or state-space models
- Guided design workflow: "Plant Modeling > Requirements Specification > Controller Design > Simulation Verification > Code Generation"
- Import existing Simulink models with block-mapping and compatibility report

**Key Workflows:**
- Building plant models from first principles or system-identification data
- Designing and auto-tuning PID controllers with robustness analysis (gain/phase margin, sensitivity peaks)
- Simulating nonlinear system behavior with saturation, dead-zone, backlash, and rate limits
- Performing discrete-time controller design for digital implementation at specified sample rates
- Generating embedded C code for controller deployment on microcontrollers and PLCs

---

### Senior/Expert

**Modified UI/UX:**
- Multi-domain workspace with control design, plant modeling, hardware-in-the-loop, and deployment tools in configurable layout
- Custom block authoring environment with equation editor, C/C++ code blocks, and FMU import
- Scripting console with full API access for automated design-space exploration and batch analysis

**Feature Gating:**
- All capabilities unlocked: robust control (H-infinity, mu-synthesis), adaptive control, MPC, nonlinear MPC, gain scheduling
- Hardware-in-the-loop (HIL) interface with real-time simulation and rapid prototyping
- Full code generation with custom target support, AUTOSAR integration, and safety-certified code output (IEC 61508)

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Migration assistant for Simulink models, Stateflow charts, and custom S-functions with compatibility analysis
- Configure real-time target hardware (dSPACE, Speedgoat, NI) and HIL test infrastructure
- Enterprise integration setup with requirements-management tools (DOORS), ALM systems, and CI/CD pipelines

**Key Workflows:**
- Designing robust multi-variable controllers using H-infinity and structured singular-value methods
- Implementing gain-scheduled controllers for systems with large operating envelopes
- Running hardware-in-the-loop test campaigns for controller validation before deployment
- Generating safety-certified production code meeting IEC 61508 / DO-178C standards
- Developing model-based systems engineering workflows integrating plant, controller, and test models

---

## Role-Based Personas

### Designer/Creator

**Modified UI/UX:**
- System-architecture canvas for sketching control-system topologies with drag-and-drop subsystem blocks
- Visual signal-flow editor with intuitive routing, labeling, and color-coding for multi-loop architectures
- Template gallery of common control architectures (cascade, feedforward, ratio, override, split-range)

**Feature Gating:**
- Full block-diagram creation with hierarchical subsystems and parameterized templates
- Linear analysis tools for validating architecture choices (bandwidth allocation, interaction analysis)
- Detailed controller synthesis available but secondary to architecture-level design

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- "Architecture Patterns" gallery showing common control structures with design rationale and trade-offs
- Quick-start templates for industry-specific control problems (process control, motion control, power electronics)
- Tutorial on parameterized subsystem creation for reusable control-module libraries

**Key Workflows:**
- Architecting multi-loop control systems with clearly defined interfaces and signal hierarchies
- Evaluating control-architecture alternatives through linear analysis (relative gain array, interaction measures)
- Creating reusable parameterized control-module templates for product-line deployment
- Documenting control-system architecture with auto-generated block diagrams and signal dictionaries
- Collaborating with plant-modeling and simulation teams on system-level integration

---

### Analyst/Simulator

**Modified UI/UX:**
- Simulation-centric layout with large scope displays, data-inspection tools, and batch-run manager
- Robustness-analysis dashboard with structured singular-value plots, worst-case gain, and parametric uncertainty visualization
- Monte Carlo simulation manager with statistical post-processing (histograms, scatter plots, correlation analysis)

**Feature Gating:**
- All analysis and simulation tools unlocked: linear analysis, nonlinear sim, Monte Carlo, worst-case, frequency-domain
- Optimization tools for controller parameter tuning against multi-objective performance criteria
- Block-diagram editing available but positioned for test-bench setup rather than design authoring

**Pricing Tier:** Professional or Enterprise tier

**Onboarding Flow:**
- Analysis-methodology guide covering systematic V&V workflows for control-system simulation
- Configure uncertainty models for plant-parameter variations and disturbance characterization
- Benchmark validation: simulate canonical control problems (double integrator, flexible beam) and verify expected behavior

**Key Workflows:**
- Running comprehensive linear analysis across operating points (stability margins, bandwidth, disturbance rejection)
- Performing Monte Carlo simulations with plant-parameter uncertainties to assess controller robustness
- Conducting worst-case analysis to identify critical parameter combinations that degrade performance
- Optimizing controller parameters against multi-objective criteria (performance, robustness, control effort)
- Generating verification reports documenting simulation coverage, results, and pass/fail criteria

---

### Manufacturing/Process

**Modified UI/UX:**
- Commissioning-focused workspace with controller parameter tables, tuning interfaces, and real-time trend displays
- Auto-tuning relay-feedback and step-test tools for on-site PID commissioning
- Process-monitoring dashboard with alarm management, loop-health indicators, and performance-degradation alerts

**Feature Gating:**
- Controller deployment tools, PID auto-tuning, and process-monitoring utilities unlocked
- OPC UA/DA connectivity for interfacing with PLCs, DCS, and SCADA systems
- Design-level analysis tools available but de-emphasized for commissioning role

**Pricing Tier:** Professional tier with Process add-on

**Onboarding Flow:**
- Connectivity setup: configure OPC connections to plant DCS/PLC systems for live data access
- Commissioning workflow tutorial: connect > identify > tune > verify > document
- Configure loop-performance monitoring thresholds and alarm rules for ongoing operations

**Key Workflows:**
- Commissioning controllers on live plant equipment using auto-tuning and step-test methods
- Monitoring control-loop performance metrics (IAE, oscillation index, valve travel) across the plant
- Diagnosing poorly performing loops and re-tuning based on updated plant models
- Managing controller parameter databases with version control and change documentation
- Generating commissioning reports and loop-performance audit summaries

---

### Regulatory/Compliance

**Modified UI/UX:**
- Safety-analysis dashboard with SIL (Safety Integrity Level) assessment tools and IEC 61508/61511 compliance tracking
- Requirements-traceability matrix linking control requirements to design, simulation evidence, and test results
- Change-management panel with impact assessment for controller modifications in safety-critical systems

**Feature Gating:**
- Full design and simulation with emphasis on safety analysis (fault injection, redundancy verification, diagnostic coverage)
- IEC 61508 compliant code generation with traceability from requirements through design to deployed code
- Electronic signatures, version control, and formal change-management workflows

**Pricing Tier:** Enterprise tier with Safety module

**Onboarding Flow:**
- Safety-profile setup: select applicable safety standards (IEC 61508, IEC 61511, DO-178C, ISO 26262) to configure workflows
- Configure SIL assessment templates and diagnostic-coverage calculation tools
- Demo end-to-end safety lifecycle: hazard analysis > SIL allocation > design > verification > validation > deployment

**Key Workflows:**
- Performing SIL allocation and assessment for safety-instrumented functions
- Running fault-injection simulations to verify controller behavior under component failures
- Generating IEC 61508 compliant verification evidence with full traceability
- Managing safety-critical controller changes with formal impact assessment and re-verification
- Producing safety-case documentation for regulatory submission and audit

---

### Manager/Decision-maker

**Modified UI/UX:**
- Program dashboard with controller-development status, deployment schedule, and performance KPIs across all control loops
- License-utilization and resource-allocation view for engineering and HIL test assets
- Risk register highlighting safety-critical loops requiring additional verification or regulatory attention

**Feature Gating:**
- Read-only access to control architectures, simulation results, and deployment configurations
- Full project analytics, scheduling, and resource-management tools
- No direct controller design or parameter modification capability

**Pricing Tier:** Enterprise tier (manager seat)

**Onboarding Flow:**
- Dashboard configuration: connect to project-management and ALM tools for unified development-status view
- Walkthrough of control-system KPIs: loop performance indices, deployment coverage, and safety-case status
- Configure alerting for safety-critical milestone delays and performance degradation

**Key Workflows:**
- Monitoring control-system development progress across multiple platforms or plant areas
- Reviewing safety-case status and making go/no-go decisions for safety-critical deployments
- Allocating engineering resources and HIL test time across concurrent projects
- Evaluating control-system performance impact on plant efficiency, yield, and product quality
- Approving controller deployments and safety-related changes through formal authorization workflows
