# RoboSim — Persona Subspecs

> Parent spec: [`specs/27-robosim.md`](../27-robosim.md)

---

## Experience-Level Personas

### Student/Academic

**Modified UI/UX:**
- Provide a simplified environment builder with drag-and-drop objects (tables, walls, obstacles) and a pre-loaded robot library (TurtleBot, UR5, Franka)
- Show a "Learn Robotics" sidebar explaining physics concepts (joint types, sensor models, PID control) contextually during simulation
- Display a block-based programming interface alongside the Python editor for users not yet comfortable with code

**Feature Gating:**
- Allow up to 3 concurrent simulation environments with single-robot scenarios on shared GPU compute
- Expose basic sensors (camera, LiDAR, IMU) and simple physics; hide multi-robot fleet simulation and photorealistic rendering
- Limit scene complexity to 50 objects and simulation time to 10 minutes per run

**Pricing Tier:** Free tier

**Onboarding Flow:**
- First-run tutorial guides the user through placing a TurtleBot in a maze and writing a simple wall-following algorithm
- Offer a curated set of 8 lab exercises (obstacle avoidance, pick-and-place, line following) with starter code and grading rubrics
- Prompt to join a classroom workspace if the user signs up with a .edu email

**Key Workflows:**
- Place a robot in a pre-built environment and write a control algorithm in Python or block-based code
- Run the simulation and observe the robot's behavior with real-time sensor data visualization
- Complete structured lab exercises and compare results with classmates in a shared workspace
- Export simulation recordings (video, trajectory data) for a course report

---

### Junior Engineer (0-3 years)

**Modified UI/UX:**
- Show the full environment editor with URDF/SDF import, sensor configuration panels, and physics parameter tuning
- Provide a "Simulation Debugger" panel that highlights physics instabilities, collision issues, and sensor configuration errors
- Display a ROS2 topic inspector with live message visualization for debugging robot software

**Feature Gating:**
- Unlock custom robot import (URDF/SDF), multi-sensor configurations, and medium-fidelity physics
- Expose ROS2 integration with topic publish/subscribe from the simulation environment
- Hide fleet simulation (>5 robots), photorealistic rendering, and domain randomization tools

**Pricing Tier:** Pro tier ($69/month)

**Onboarding Flow:**
- Walk through importing a custom URDF robot model, configuring sensors, and running a navigation task
- Demonstrate the ROS2 integration by connecting the simulation to a ROS2 navigation stack
- Show the simulation debugger diagnosing a common issue (e.g., collision mesh mismatch, unstable joint limits)

**Key Workflows:**
- Import custom robot models (URDF/SDF) and configure sensors, actuators, and physics properties
- Develop and test navigation algorithms with ROS2 integration in realistic indoor/outdoor environments
- Debug robot behavior using the simulation debugger, ROS2 topic inspector, and trajectory replay
- Test manipulation tasks (pick-and-place, assembly) with contact physics simulation
- Share simulation environments and results with team members for collaborative development

---

### Senior/Expert

**Modified UI/UX:**
- Provide a multi-viewport workspace with synchronized 3D simulation, sensor data streams, and ROS2 architecture graph
- Expose the full simulation configuration: physics engine selection (PhysX, Bullet, MuJoCo), rendering quality, and timestep control
- Support headless simulation via API for large-scale automated testing and RL training pipelines

**Feature Gating:**
- Full access: unlimited concurrent simulations, fleet simulation (100+ robots), photorealistic rendering, domain randomization, and headless API
- Expose RL training infrastructure with parallel environment rollouts and GPU-accelerated physics
- Enable on-premise deployment, custom physics plugins, and integration with external tools (Isaac Sim, MoveIt)

**Pricing Tier:** Enterprise tier ($249/month or annual contract)

**Onboarding Flow:**
- Skip tutorials; present API documentation and guides for integrating with existing ROS2 workspaces and CI/CD pipelines
- Offer bulk import of simulation environments and robot models from Gazebo, Isaac Sim, or custom formats
- Configure compute preferences (GPU type, parallelism) and default simulation parameters

**Key Workflows:**
- Run large-scale fleet simulations with 100+ robots for warehouse logistics and multi-agent coordination
- Train reinforcement learning policies using massively parallel simulation environments on GPU clusters
- Set up automated regression testing suites that validate robot software against thousands of scenarios
- Configure domain randomization (lighting, textures, physics parameters) to improve sim-to-real transfer
- Integrate simulation into CI/CD pipelines for continuous validation of robot software changes

---

## Role-Based Personas

### Designer/Creator

**Modified UI/UX:**
- Emphasize the 3D environment builder with terrain editing, object placement, and lighting configuration tools
- Provide a robot builder interface for assembling robots from modular components (links, joints, sensors, actuators)
- Show a split view of the design canvas and live simulation preview for rapid iteration on environment layouts

**Feature Gating:**
- Full access to environment creation, robot model building, and asset library tools
- Expose terrain generation, weather effects, and dynamic object spawning for realistic scenario design
- Enable asset sharing and environment template publishing to the team library

**Pricing Tier:** Pro tier ($69/month)

**Onboarding Flow:**
- Start with the environment builder, placing objects and configuring a simple scenario for a mobile robot
- Demonstrate the robot builder by assembling a custom manipulator from modular joint and link components
- Show how to publish a designed environment as a reusable template for the team

**Key Workflows:**
- Design custom simulation environments with terrain, objects, lighting, and dynamic elements
- Build robot models from modular components with configurable joints, sensors, and visual meshes
- Create scenario templates (warehouse layouts, outdoor terrains, assembly lines) for team reuse
- Iterate on environment designs with live simulation preview for rapid validation
- Import and adapt CAD models (STEP, STL) of real-world objects and workspaces for digital twin scenarios

---

### Analyst/Simulator

**Modified UI/UX:**
- Foreground the data analysis dashboard with trajectory plots, sensor data timelines, performance metrics, and statistical comparison tools
- Show a scenario comparison view for evaluating robot performance across different environment configurations or algorithm versions
- Provide a Monte Carlo analysis panel for running parameter sweeps and quantifying performance distributions

**Feature Gating:**
- Full access to simulation execution, data logging, statistical analysis, and comparison tools
- Expose Monte Carlo simulation for robustness testing across randomized scenarios
- Enable data export (ROS bags, CSV, HDF5) and integration with analysis tools (MATLAB, Python notebooks)

**Pricing Tier:** Pro tier ($69/month)

**Onboarding Flow:**
- Walk through running a navigation benchmark: execute multiple trials, collect metrics, and analyze performance distributions
- Demonstrate the scenario comparison tool by evaluating two algorithm versions across identical test cases
- Show Monte Carlo analysis for quantifying the impact of sensor noise on navigation accuracy

**Key Workflows:**
- Run systematic benchmarks evaluating robot performance across standardized test scenarios
- Perform Monte Carlo analysis to quantify robustness to sensor noise, physics variations, and environment changes
- Compare algorithm versions side by side using standardized metrics (success rate, path length, task completion time)
- Analyze sensor data streams and trajectories to diagnose failure modes and identify improvement opportunities
- Generate performance reports with statistical summaries for research papers or design reviews

---

### Manufacturing/Process

**Modified UI/UX:**
- Show a digital twin dashboard with real-world robot fleet status alongside simulation replicas for validation
- Display a deployment readiness checklist tracking simulation coverage, edge case testing, and performance thresholds
- Provide a scenario library manager for maintaining and versioning regression test suites

**Feature Gating:**
- Expose digital twin synchronization tools for mirroring real-world robot configurations in simulation
- Enable automated regression testing with pass/fail criteria and continuous monitoring
- Unlock deployment pipeline integration for validating software updates before fleet rollout

**Pricing Tier:** Enterprise tier ($249/month)

**Onboarding Flow:**
- Set up a digital twin of the production environment by importing facility layouts and robot configurations
- Configure automated regression test suites with performance thresholds and failure criteria
- Demonstrate the deployment validation workflow: simulate software update, run tests, approve for fleet rollout

**Key Workflows:**
- Maintain digital twins of production environments for ongoing validation and scenario testing
- Run automated regression test suites before deploying software updates to the physical robot fleet
- Validate new robot configurations and tool changes in simulation before physical deployment
- Monitor simulation-to-real performance correlation and flag divergence for investigation
- Manage a versioned library of test scenarios covering edge cases and failure modes

---

### Regulatory/Compliance

**Modified UI/UX:**
- Add a safety testing dashboard tracking coverage of required test scenarios (ISO 10218, ISO 13482, ISO 3691-4)
- Display collision detection and safety zone visualization with configurable human-robot proximity thresholds
- Provide a compliance report generator that maps simulation results to specific safety standard requirements

**Feature Gating:**
- Expose safety-focused simulation tools: collision force analysis, safety zone verification, and emergency stop testing
- Enable audit-ready logging of all simulation parameters, results, and safety metric evaluations
- Unlock compliance report templates for ISO robotics safety standards

**Pricing Tier:** Enterprise tier ($249/month)

**Onboarding Flow:**
- Select the applicable safety standards (ISO 10218, ISO 13482) and load corresponding test scenario templates
- Walk through running a safety zone verification simulation and interpreting the compliance report
- Demonstrate the audit trail and how to export simulation evidence for certification submissions

**Key Workflows:**
- Run safety-critical simulations verifying collision avoidance, force limiting, and emergency stop behavior
- Test human-robot interaction scenarios against ISO safety standard requirements
- Generate compliance reports mapping simulation evidence to specific safety standard clauses
- Maintain an audit trail of all safety simulations with parameters, results, and pass/fail determinations
- Document worst-case scenario analysis results for inclusion in risk assessments and certification applications

---

### Manager/Decision-maker

**Modified UI/UX:**
- Present a program dashboard showing all robot development projects, simulation utilization, and key milestone status
- Show fleet readiness metrics: test coverage, simulation pass rates, deployment confidence scores
- Provide cost analytics: compute spend per project, simulation ROI vs. physical testing costs

**Feature Gating:**
- Read-only access to all simulation projects with commenting and approval capabilities
- Expose project management, resource allocation, and budget tracking tools
- Enable executive reporting with fleet readiness summaries and deployment risk assessments

**Pricing Tier:** Enterprise tier ($249/month)

**Onboarding Flow:**
- Set up the program dashboard and configure project categories (navigation R&D, fleet operations, safety validation)
- Walk through the fleet readiness view showing test coverage and deployment confidence metrics
- Demonstrate cost analytics comparing simulation costs against estimated physical testing alternatives

**Key Workflows:**
- Monitor robot development programs from a unified dashboard with milestone and risk tracking
- Evaluate fleet deployment readiness based on simulation test coverage and pass rate metrics
- Track simulation compute costs and compare ROI against traditional physical testing approaches
- Review and approve software deployments to production fleets based on simulation validation results
- Generate program status reports for leadership covering technical progress, costs, and deployment timelines
