# DriveSim — Persona Subspecs

> Parent spec: [`specs/40-drivesim.md`](../40-drivesim.md)

---

## Experience-Level Personas

### Student/Academic
**Modified UI/UX:**
- Guided scenario builder with preset environments (urban intersection, highway merge, parking lot) and simple parameter controls (weather, time of day, traffic density)
- Simplified sensor view showing camera, LiDAR, and radar outputs side-by-side with labeled ground-truth annotations
- Tutorial-style interface with step-by-step prompts for setting up a basic perception evaluation pipeline

**Feature Gating:**
- Access to pre-built scenarios and a limited vehicle model library; custom environment creation disabled
- Sensor simulation limited to single camera, single LiDAR, and single radar; multi-sensor rigs require upgrade
- Up to 10 concurrent scenario runs; batch testing and CI/CD integration disabled

**Pricing Tier:** Free tier (academic email verification)

**Onboarding Flow:**
- Tutorial: load an urban intersection scenario, attach a perception model, run simulation, and evaluate detection accuracy against ground truth
- Pre-loaded demo scenarios (pedestrian crossing, lane change, emergency braking) for immediate experimentation
- Prompt to join a research group workspace if an advisor's invite code is available

**Key Workflows:**
- Run pre-built driving scenarios and evaluate perception algorithm performance against ground-truth labels
- Explore how weather and lighting conditions affect camera and LiDAR perception model accuracy
- Test basic planning algorithms (path following, obstacle avoidance) in controlled simulation environments
- Export sensor data (images, point clouds) for training and evaluating ML models offline
- Generate simulation result plots and metrics for course projects and research papers

---

### Junior Engineer (0-3 years)
**Modified UI/UX:**
- Scenario template library organized by test category (Euro NCAP, SOTIF, intersection, highway) with configurable parameters
- Sensor configuration panel with visual rig editor: place cameras, LiDARs, and radars on a 3D vehicle model
- Test result dashboard showing per-scenario pass/fail, aggregate metrics, and regression comparison against previous runs

**Feature Gating:**
- Full scenario builder with custom road networks, traffic configurations, and environmental conditions
- Multi-sensor rig configuration with physically accurate models; up to 8 sensors per vehicle
- Up to 100 concurrent scenario runs; basic CI/CD webhook integration enabled

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Configure a sensor rig on a vehicle model and calibrate sensor parameters (FOV, range, noise characteristics)
- Build a custom test scenario: define road geometry, place traffic actors, set trigger conditions and success criteria
- Connect to team workspace and set up a regression testing pipeline for the team's perception stack

**Key Workflows:**
- Build and configure custom driving scenarios for testing specific perception and planning behaviors
- Design sensor rig configurations and evaluate coverage, blind spots, and redundancy
- Run regression test suites across scenario libraries and identify performance regressions after code changes
- Debug perception failures by replaying scenarios and inspecting sensor data frame-by-frame
- Generate test reports documenting scenario coverage, pass rates, and failure root causes

---

### Senior/Expert
**Modified UI/UX:**
- Multi-pane workspace with scenario editor, sensor simulation, planning visualization, and metrics dashboard in coordinated views
- Custom actor behavior scripting with Python-based scenario definition language and traffic AI configuration
- API console for programmatic scenario generation, fleet-scale simulation orchestration, and ML pipeline integration

**Feature Gating:**
- All features unlocked: ray-traced rendering, custom vehicle dynamics, scenario scripting API, GPU cluster burst
- Unlimited concurrent simulations; CI/CD integration with full pipeline control
- Admin controls for scenario libraries, vehicle model databases, and validation protocols

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Import existing CARLA/OpenSCENARIO scenarios to migrate test libraries to DriveSim
- API walkthrough for integrating simulation into CI/CD pipelines and ML training workflows
- Configure team scenario library with validation status, coverage tracking, and regression baselines

**Key Workflows:**
- Architect large-scale simulation test programs covering thousands of scenarios across the operational design domain
- Develop custom scenario generation pipelines using procedural and adversarial methods for edge case discovery
- Integrate simulation into CI/CD for automated perception and planning stack validation on every code commit
- Build and maintain domain-specific sensor models calibrated against real-world sensor data
- Lead simulation strategy, defining coverage metrics, pass criteria, and validation methodology for the AV program

---

## Role-Based Personas

### Designer/Creator
**Modified UI/UX:**
- World builder as primary workspace: 3D environment editor for creating roads, intersections, buildings, vegetation, and traffic infrastructure
- Asset library with drag-and-drop vehicles, pedestrians, traffic signs, and road markings
- Scenario choreography timeline for scripting actor movements, trigger events, and environmental changes over time

**Feature Gating:**
- Full world building tools with road network editor, terrain sculpting, and asset placement
- Custom vehicle model creation with configurable dimensions, dynamics, and visual appearance
- Export environments in OpenDRIVE/OpenSCENARIO formats for cross-platform compatibility

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Create a simple intersection environment: lay roads, place traffic signals, add buildings and vegetation
- Script a scenario: define ego vehicle path, spawn traffic actors, set trigger conditions for interactions
- Walk through asset creation: import a 3D vehicle model and configure its physical properties

**Key Workflows:**
- Build realistic driving environments with detailed road networks, infrastructure, and scenery
- Design scenario scripts that test specific driving behaviors and edge cases
- Create and curate vehicle and pedestrian actor libraries with realistic appearance and behavior
- Build parameterized scenario templates that can generate thousands of variations from a single design
- Maintain and version-control environment and scenario assets for the team's test library

---

### Analyst/Simulator
**Modified UI/UX:**
- Test analytics dashboard as primary view with scenario coverage heatmaps, failure rate trends, and performance metric distributions
- Detailed replay viewer with synchronized camera, LiDAR, radar, and planning visualization for failure investigation
- Statistical analysis workspace for aggregating metrics across large test campaigns

**Feature Gating:**
- Full access to all test execution, replay, and analysis tools
- Advanced analytics: failure clustering, root cause classification, coverage gap identification
- Batch test execution across scenario libraries with automated metric extraction and report generation

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Run a test campaign across a scenario library and walk through the results analysis workflow
- Use the replay viewer to investigate a specific failure: identify the root cause in perception or planning
- Configure automated metrics dashboards tracking key performance indicators over time

**Key Workflows:**
- Execute large-scale test campaigns and analyze aggregate results across the scenario library
- Investigate simulation failures using detailed replay with ground-truth comparison and sensor data inspection
- Track perception and planning performance metrics over time to identify improvement trends and regressions
- Perform statistical analysis on test results to quantify confidence in system safety claims
- Generate comprehensive test campaign reports with coverage metrics, failure analysis, and recommendations

---

### Manufacturing/Process
**Modified UI/UX:**
- Hardware-in-the-loop (HIL) integration panel for connecting simulation to physical ECUs and sensor hardware
- Production test dashboard showing end-of-line simulation test results for delivered AV systems
- Sensor calibration verification tools for validating mounted sensor configurations against simulation models

**Feature Gating:**
- Access to HIL simulation tools, production test sequences, and sensor calibration verification
- Pre-approved test scenario execution; custom scenario creation restricted to engineering role
- Fleet-level dashboards for tracking production test pass rates and calibration status

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Connect HIL bench hardware and verify communication with DriveSim simulation environment
- Walk through a production sensor calibration verification: compare mounted sensor geometry to design specification
- Configure end-of-line test sequences and pass/fail criteria for production validation

**Key Workflows:**
- Run hardware-in-the-loop simulations to validate ECU software integrated with physical hardware
- Execute production-line sensor calibration verification using simulation-based golden references
- Track end-of-line simulation test results across production vehicles for quality monitoring
- Manage production test scenario libraries ensuring consistency with engineering validation baselines
- Escalate production test failures to engineering with attached simulation replay data

---

### Regulatory/Compliance
**Modified UI/UX:**
- Safety case dashboard showing scenario coverage mapped to regulatory requirements (UNECE R157, Euro NCAP, SOTIF ISO 21448)
- Evidence package builder assembling simulation results, coverage reports, and test methodology documentation
- Traceability matrix linking each regulatory requirement to specific simulation scenarios and test results

**Feature Gating:**
- Full access to test results, coverage reports, and simulation replays for review
- Write access to safety case annotations, requirement traceability links, and regulatory approval stamps
- Audit trail with immutable records of all test executions, results, software versions, and reviewer actions

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Configure applicable AV safety standards and map requirements to simulation scenario categories
- Walk through building a safety evidence package: link scenarios to requirements, compile results, generate documentation
- Set up independent assessment workflows with required separation between developer and assessor roles

**Key Workflows:**
- Map simulation test scenarios to specific regulatory requirements and track coverage completeness
- Build safety evidence packages demonstrating adequate testing for regulatory approval submissions
- Review and approve simulation methodology for compliance with SOTIF, UNECE R157, and Euro NCAP protocols
- Verify that scenario libraries adequately cover the operational design domain and known edge cases
- Track regulatory submission status and manage responses to regulatory inquiries and audit findings

---

### Manager/Decision-maker
**Modified UI/UX:**
- AV program dashboard showing simulation coverage progress, safety metric trends, and regulatory milestone tracking
- Cost analytics: compute spend, scenario library investment, and simulation-vs-road-test cost comparison
- Risk matrix showing scenario coverage gaps, unresolved safety concerns, and regulatory timeline risks

**Feature Gating:**
- Read-only access to program dashboards, test summaries, and regulatory status; no direct simulation tools
- Team management: user provisioning, compute budget allocation, and scenario library governance
- Approval authority for safety case submissions, test methodology changes, and go/no-go decisions for road testing

**Pricing Tier:** Enterprise tier (included in admin seat)

**Onboarding Flow:**
- Program dashboard overview: simulation coverage metrics, safety KPIs, regulatory milestones, compute spend
- Configure program targets for scenario coverage, test pass rates, and regulatory submission dates
- Set up notification rules for safety-critical simulation failures and milestone deadline risks

**Key Workflows:**
- Monitor AV program simulation progress against scenario coverage targets and regulatory milestones
- Track simulation-based safety metrics and assess readiness for road testing or deployment decisions
- Manage simulation compute budget and prioritize scenario development investment across program needs
- Review safety case readiness and approve regulatory submissions based on simulation evidence
- Present simulation program status, safety confidence, and resource needs to executive leadership and board
