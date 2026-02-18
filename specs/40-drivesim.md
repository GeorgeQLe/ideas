# DriveSim — Cloud Autonomous Driving Simulation Platform

## Executive Summary

DriveSim is a cloud-native autonomous driving simulation platform that provides GPU ray-traced photorealistic rendering, multi-modal sensor simulation (camera, LiDAR, radar), high-fidelity vehicle dynamics, and intelligent traffic AI — all accessible from the browser without local GPU infrastructure. By combining Vulkan-based ray-traced environments with physically accurate sensor models, Pacejka tire dynamics, IDM/MOBIL traffic behavior, and ML model testing pipelines in a single integrated platform, DriveSim enables autonomous vehicle teams to run thousands of simulation scenarios in parallel, test perception and planning stacks against ground-truth sensor data, and catch regressions before road testing — at a fraction of the cost of NVIDIA DRIVE Sim or Applied Intuition.

---

## Problem Statement

**The pain:**
- Setting up a full autonomous driving simulation stack (rendering engine + sensor models + vehicle dynamics + traffic + scenario management) requires 6-12 months of custom engineering and dedicated graphics/physics teams that most AV companies cannot afford
- CARLA is free but notoriously difficult to deploy at scale — it crashes on large maps, has limited sensor fidelity, lacks deterministic replay, and requires teams to maintain fragile Linux GPU environments with specific NVIDIA driver versions
- NVIDIA DRIVE Sim and Applied Intuition cost $500K-$2M+ per year and lock teams into proprietary ecosystems, making them inaccessible to startups, Tier 2 suppliers, and academic research groups
- Running regression tests across thousands of scenarios (weather, traffic density, edge cases) takes days on local hardware, delaying software release cycles by weeks and increasing the risk of deploying untested perception failures
- The sim-to-real gap in sensor simulation remains the biggest bottleneck — most simulators produce camera images and LiDAR point clouds that look nothing like real sensor data, causing perception models trained or validated in simulation to fail on the road

**Current workarounds:**
- Teams run CARLA on local GPU workstations or self-managed cloud VMs, spending 30%+ of engineering time on infrastructure maintenance, driver debugging, and scenario scripting rather than algorithm development
- Large OEMs pay $1M+ annually for NVIDIA DRIVE Sim or Applied Intuition but still maintain parallel CARLA setups for rapid prototyping because enterprise tools have slow iteration cycles
- Some teams use simple 2D simulators or log replay (replaying recorded sensor data) which cannot generate novel scenarios, test counterfactuals, or validate behavior in conditions not yet encountered on the road
- Academic groups resort to toy environments (SUMO + simple rendering) that produce unrealistic sensor data, leading to published results that do not transfer to real vehicles

**Market size:** The global autonomous vehicle simulation market is valued at $2.1 billion (2024) and projected to reach $7.8 billion by 2030 (CAGR ~24%). The addressable market includes AV developers (L2-L5), ADAS Tier 1 suppliers, robotaxi operators, trucking companies, and automotive OEMs. There are an estimated 2,000+ companies and 500+ academic groups worldwide actively developing autonomous driving systems. The shift from on-road testing to simulation-first validation is accelerating due to regulatory requirements (EU GSR, UNECE R157) mandating scenario-based safety validation.

---

## Target Users

### Primary Personas

**1. Dr. Sarah Chen — Perception Lead at a Series B Autonomous Vehicle Startup**
- Leads a team of 8 ML engineers developing camera and LiDAR perception models for a Level 4 urban robotaxi
- Currently uses CARLA on a cluster of 12 GPU workstations that constantly break due to driver version conflicts
- Needs to test every perception model update against 5,000+ scenarios before merging to main — currently takes 3 days on local hardware
- Needs: photorealistic sensor simulation with ground-truth labels, batch testing across thousands of scenarios, CI/CD integration for automated regression testing

**2. Marcus — ADAS Validation Engineer at a Tier 1 Automotive Supplier**
- Responsible for validating forward collision warning and lane departure systems for 4 different OEM customers
- Must demonstrate compliance with Euro NCAP test protocols and UNECE R157 scenarios
- Currently uses an expensive proprietary tool ($400K/year) that his team of 3 shares via remote desktop sessions
- Needs: OpenSCENARIO-based scenario execution, standardized KPI reporting, deterministic replay, affordable per-seat pricing

**3. Professor Akiko Tanaka — Autonomous Driving Researcher at a Top-10 University**
- Supervises 15 PhD students working on motion planning, behavior prediction, and reinforcement learning for driving
- Lab has 4 aging GPU workstations running CARLA Classic that students fight over and that crash weekly
- Publishes papers that require reproducible simulation results across diverse traffic scenarios
- Needs: cloud-based simulation accessible from any laptop, OpenSCENARIO support for reproducible experiments, affordable academic pricing, easy sharing of scenarios and results for peer review

### Secondary Personas
- Robotaxi fleet operators who need to replay real-world incidents in simulation and test software fixes before redeployment
- Trucking and logistics companies validating autonomous highway driving with platoon simulation
- Insurance and regulatory bodies who need standardized simulation-based safety assessment tools
- HD map providers who want to validate map accuracy by rendering simulated drives on their map data

---

## Solution Overview

DriveSim works as follows:
1. Users create a driving environment by importing OpenDRIVE HD maps or selecting from a library of pre-built road networks (urban, highway, rural), then customize weather, lighting, and traffic density through a visual scenario editor
2. The platform renders photorealistic driving scenes using GPU ray-traced rendering (Vulkan) with PBR materials, dynamic weather (rain, fog, snow, glare), and time-of-day lighting — generating camera, LiDAR, and radar sensor data that closely matches real-world sensor characteristics
3. Vehicle dynamics are simulated using a Pacejka tire model with suspension dynamics and powertrain modeling, while surrounding traffic follows Intelligent Driver Model (IDM) car-following and MOBIL lane-changing behaviors to produce realistic multi-agent driving scenarios
4. Users import their perception models (ONNX/TensorRT), planning algorithms, and control stacks via containerized interfaces, test them against simulated sensor streams, and evaluate performance using configurable pass/fail criteria and safety KPIs
5. Batch simulation enables running thousands of scenario variations in parallel on cloud GPU clusters, with automated regression testing, KPI dashboards, and CI/CD integration to catch performance regressions before code is merged or deployed to vehicles

---

## Core Features

### F1: GPU Ray-Traced 3D Environment
- Vulkan-based ray-traced rendering pipeline producing photorealistic driving scenes at up to 4K resolution and 60 FPS
- PBR (Physically Based Rendering) material library: asphalt (wet/dry), concrete, metal, glass, vegetation, road markings with weathering and wear
- Dynamic time-of-day lighting with physically accurate sun position, sky model (Hosek-Wilkie), and atmospheric scattering
- Weather effects: volumetric rain with puddle reflections, fog with distance-based density falloff, snow accumulation on road surfaces, sun glare and lens flare
- Procedural traffic assets: vehicles (cars, trucks, buses, motorcycles) with randomized colors, damage states, and dirt/cleanliness levels
- Deformable road geometry: potholes, speed bumps, construction zones, and temporary barriers

### F2: Camera Sensor Simulation
- Configurable camera models matching real sensor specs: resolution (up to 8MP), FOV, focal length, rolling shutter, exposure time
- Physically-based lens effects: radial and tangential distortion (Brown-Conrady model), chromatic aberration, vignetting, depth of field
- Dynamic auto-exposure and HDR tonemapping simulating real camera ISP behavior in tunnels, low-sun, and nighttime conditions
- Motion blur from vehicle ego-motion and object motion with configurable shutter speed
- Rain droplets on lens surface with refraction, fog haloing around headlights, snow particles in beam
- Ground-truth outputs: pixel-level semantic segmentation, instance segmentation, optical flow, depth maps, 2D/3D bounding boxes

### F3: LiDAR Sensor Simulation
- GPU ray-casting engine simulating spinning and solid-state LiDAR with configurable beam patterns matching commercial sensors (Velodyne VLP-16/VLP-32/VLS-128, Ouster OS1/OS2, Hesai Pandar64/AT128)
- Physically accurate return intensity based on surface material reflectivity, angle of incidence, and range
- Multi-return support: first, strongest, and last return per beam for vegetation and semi-transparent objects
- Atmospheric effects: rain scatter causing false returns, fog attenuation reducing max range, snow noise injection
- Configurable parameters: rotation rate (5-20 Hz), vertical FOV, angular resolution, max range, noise model (Gaussian + range-dependent)
- Ground-truth outputs: per-point semantic labels, instance IDs, exact range without noise for validation

### F4: Radar Sensor Simulation
- Doppler radar simulation producing range-velocity detection maps with configurable frequency band (77 GHz automotive)
- Radar cross-section (RCS) computation per object based on geometry, material, and aspect angle using simplified RCS models
- Multi-path reflection modeling: ground bounce, guardrail reflections, tunnel reverberations
- Ghost target generation from multi-path and inter-vehicle reflections matching real radar artifacts
- Configurable parameters: range resolution, velocity resolution, FOV (azimuth/elevation), detection sensitivity, false alarm rate
- Cluster-level and raw detection-level output formats matching Continental, Bosch, and Aptiv radar interfaces

### F5: Vehicle Dynamics Engine
- Bicycle model for rapid prototyping with single-track lateral dynamics and linear tire approximation
- Full Pacejka Magic Formula tire model (MF 6.2) with combined longitudinal/lateral slip, load sensitivity, and camber effects
- Suspension dynamics: double-wishbone and MacPherson strut models with configurable spring rates, damping, and anti-roll bars
- Powertrain modeling: torque curves for ICE and electric motors, gear ratios, differential types (open, limited-slip, torque-vectoring)
- Aerodynamic drag and lift forces with configurable Cd and Cl coefficients
- Vehicle state output: position, velocity, acceleration, yaw rate, slip angle, individual wheel speeds, tire forces — all at 1 kHz

### F6: Traffic AI and Scenario Management
- Intelligent Driver Model (IDM) car-following behavior with configurable desired speed, time headway, maximum acceleration, comfortable deceleration
- MOBIL (Minimizing Overall Braking Induced by Lane changes) lane-changing model with politeness factor and safety gap thresholds
- Traffic signal control: fixed-time, actuated, and adaptive signal timing with yellow/all-red phases
- Pedestrian behavior: social force model with crosswalk compliance, jaywalking probability, group behavior
- Cyclist and motorcycle agents with lane-splitting and filtering behavior models
- Scenario injection: spawn vehicles at specific positions, trigger cut-ins, sudden braking, pedestrian dashes, and emergency vehicle scenarios at defined simulation time or spatial triggers

### F7: ML Model Testing Pipeline
- Import perception models in ONNX and TensorRT formats for direct inference on simulated sensor data
- Automated perception evaluation: compare model detections against simulation ground truth using mAP, MOTA, IoU, and custom metrics
- Closed-loop testing: connect planning and control stacks via gRPC/ROS2 interface to receive sensor data and send vehicle commands in real time
- Open-loop replay: feed recorded or simulated sensor data through perception pipeline without vehicle actuation for rapid metric computation
- Domain adaptation analysis: measure perception model degradation across weather conditions, lighting, and sensor configurations
- Model A/B testing: run two model versions against identical scenarios and compare KPI differences with statistical significance testing

### F8: OpenSCENARIO and OpenDRIVE Support
- Full OpenSCENARIO 2.0 parser and executor for declarative scenario definition with conditions, actions, and stochastic parameters
- OpenSCENARIO 1.x backward compatibility for existing scenario libraries and Euro NCAP test protocol definitions
- OpenDRIVE 1.6/1.7 map import with road geometry, lane topology, traffic signs, signals, and surface properties
- OpenCRG road surface profile import for high-frequency road roughness simulation affecting vehicle dynamics and sensor vibration
- Visual OpenSCENARIO editor: drag-and-drop scenario builder with timeline, condition trees, and parameter sweeps
- Export and sharing: scenarios exportable as standard OpenSCENARIO XML for use in other simulators (CARLA, esmini, VTD)

### F9: Batch Simulation and Regression Testing
- Parallel execution of scenario batches across cloud GPU clusters with configurable concurrency (10 to 10,000 simultaneous runs)
- Parameter sweep engine: vary weather, traffic density, sensor noise, vehicle speed, and scenario parameters across combinatorial or Latin hypercube sample spaces
- Pass/fail evaluation against configurable safety criteria: time-to-collision, minimum distance, lane departure, traffic rule violations, collision count
- Regression detection: compare KPI distributions between software versions with automated alerts for statistically significant degradations
- CI/CD integration: trigger simulation batches from GitHub Actions, GitLab CI, or Jenkins with YAML pipeline definitions
- Result archival: all simulation runs stored with full provenance (scenario version, model version, parameters) for regulatory traceability

### F10: Scenario Editor
- Visual road network editor: draw roads, intersections, roundabouts, and merge lanes with automatic OpenDRIVE generation
- Drag-and-drop traffic participant placement: vehicles, pedestrians, cyclists with configurable behaviors (IDM params, routes, triggers)
- Weather and time-of-day timeline: keyframe weather changes during a scenario (clear to rain transition, sunset progression)
- Trigger system: spatial triggers (geo-fenced regions), temporal triggers (at time T), conditional triggers (when ego speed > X, when distance to vehicle < Y)
- Scenario variant generator: define parameter ranges and automatically generate hundreds of scenario variants from a single template
- Scenario library: searchable catalog of pre-built scenarios covering Euro NCAP, NHTSA pre-crash typologies, and common AV edge cases

### F11: Analytics and KPI Dashboard
- Real-time simulation telemetry: vehicle state, sensor data, perception detections, planning decisions displayed in synchronized multi-panel view
- Safety KPI computation: time-to-collision (TTC), post-encroachment time (PET), minimum headway, deceleration rate to avoid crash (DRAC)
- Perception KPI tracking: mAP, recall, precision, false positive rate per object class across simulation runs
- Aggregate analytics: heatmaps of failure locations, weather condition vs. failure rate correlation, KPI distribution histograms across batch runs
- Custom KPI definitions: user-defined metrics computed from simulation state via Python expressions
- Exportable reports: PDF summaries with scenario descriptions, KPI tables, failure analysis, and trajectory plots for safety case documentation

### F12: Collaboration and Data Management
- Project-based workspaces with role-based access control: viewer, engineer, admin, with per-project permissions
- Scenario version control: Git-like branching, tagging, and diff visualization for scenario and map changes
- Shared simulation replay: link to any simulation run for team review with synchronized playback and commenting
- Asset management: centralized library for vehicle models, map tiles, sensor configurations, and ML models with version tracking
- Data export: simulation results exportable as ROS2 bags, HDF5, CSV, or custom formats for offline analysis
- Audit trail: complete history of simulation runs, parameter changes, and model deployments for ISO 26262 and SOTIF compliance

---

## Technical Architecture

### System Diagram

```
┌──────────────────────────────────────────────────────────────────────┐
│                         Browser Client                                │
│  ┌─────────────┐  ┌─────────────┐  ┌────────────┐  ┌─────────────┐  │
│  │  3D Viewport │  │  Scenario   │  │   Batch    │  │  Analytics  │  │
│  │  (WebGPU)    │  │  Editor     │  │  Dashboard │  │  & KPIs     │  │
│  │              │  │  (React)    │  │  (React)   │  │  (React)    │  │
│  └──────┬───────┘  └──────┬──────┘  └─────┬──────┘  └─────┬───────┘  │
│         │                 │               │               │          │
│         └─────────────────┼───────────────┼───────────────┘          │
│                           ▼                                          │
│                 State Management (Zustand + WebSocket)                │
└───────────────────────────┼──────────────────────────────────────────┘
                            │ WebSocket / gRPC-Web
                            ▼
                  ┌──────────────────────┐
                  │    API Gateway       │
                  │   (Rust / Axum)      │
                  └───┬──────────┬───────┘
                      │          │
           ┌──────────┘          └──────────────┐
           ▼                                    ▼
┌──────────────────┐              ┌──────────────────────────┐
│   PostgreSQL     │              │   Simulation Engine      │
│  (Projects,      │              │   (Rust + Vulkan)        │
│   Scenarios,     │              │                          │
│   Results,       │              │  ┌─────────────────────┐ │
│   Users)         │              │  │ Ray-Trace Renderer  │ │
└────────┬─────────┘              │  │ (Vulkan RT Pipeline)│ │
         │                        │  └─────────────────────┘ │
         ▼                        │  ┌─────────────────────┐ │
┌──────────────────┐              │  │ Sensor Sim (Camera, │ │
│     Redis        │              │  │ LiDAR, Radar)       │ │
│  (Job Queue,     │◄─────────────│  └─────────────────────┘ │
│   Sim State,     │              │  ┌─────────────────────┐ │
│   Pub/Sub)       │              │  │ Vehicle Dynamics    │ │
└──────────────────┘              │  │ (Pacejka + Susp.)   │ │
                                  │  └─────────────────────┘ │
                                  │  ┌─────────────────────┐ │
                                  │  │ Traffic AI          │ │
                                  │  │ (IDM + MOBIL)       │ │
                                  │  └─────────────────────┘ │
                                  │  ┌─────────────────────┐ │
                                  │  │ ML Model Runner     │ │
                                  │  │ (ONNX/TensorRT)     │ │
                                  │  └─────────────────────┘ │
                                  └──────────────────────────┘
                                             │
                                  ┌──────────┴──────────┐
                                  ▼                     ▼
                         ┌──────────────┐     ┌─────────────────┐
                         │ Object Store │     │  GPU Cluster     │
                         │ (S3/R2)      │     │  (K8s + NVIDIA)  │
                         │ Maps, Models,│     │  A100/H100 nodes │
                         │ Results,     │     │  Ray-trace +     │
                         │ Sensor Data  │     │  Sensor Sim +    │
                         └──────────────┘     │  Batch Sim       │
                                              └─────────────────┘
```

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend Framework | React 19 + TypeScript, Vite build system |
| 3D Viewport | WebGPU renderer with streamed ray-traced frames, Three.js fallback for scene editing |
| UI Components | Tailwind CSS, Radix UI primitives, React Flow for OpenSCENARIO visual editing |
| State Management | Zustand for client state, WebSocket for real-time simulation telemetry streaming |
| Backend API | Rust (Axum) for simulation orchestration, project management, and batch scheduling |
| Rendering Engine | Custom Vulkan RT pipeline with ray-traced reflections, shadows, and global illumination on NVIDIA RTX GPUs |
| Sensor Simulation | Vulkan compute shaders for LiDAR ray-casting, camera ISP pipeline, radar signal processing |
| Vehicle Dynamics | Rust implementation of Pacejka MF 6.2 tire model, RK4 integrator at 1 kHz, suspension/powertrain models |
| Traffic AI | Rust implementation of IDM car-following + MOBIL lane-change, social force pedestrian model |
| ML Runtime | ONNX Runtime and TensorRT inference on GPU for user-uploaded perception models |
| Scenario Engine | OpenSCENARIO 2.0 parser (Rust) with OpenDRIVE 1.7 map loader and OpenCRG surface profiles |
| Database | PostgreSQL 16 with JSONB for scenario definitions, simulation configs, and KPI results |
| Cache / Job Queue | Redis 7 for job scheduling, simulation state pub/sub, and batch orchestration |
| Object Storage | Cloudflare R2 for HD maps, 3D assets, sensor recordings, and simulation result archives |
| GPU Compute | Kubernetes (EKS) with NVIDIA A100/H100 GPU node pools, spot instances for batch simulation |
| Auth | JWT-based auth with OAuth (Google, GitHub), SAML/OIDC for enterprise SSO |
| Monitoring | Grafana + Prometheus for cluster metrics, Sentry for application errors, custom GPU utilization dashboards |

### Data Model

```
Organization
├── id (uuid), name, slug, plan, created_at
├── gpu_quota_hours_monthly, billing_email
│
├── Project[]
│   ├── id (uuid), org_id, name, description, visibility
│   ├── created_by → User, created_at, updated_at
│   │
│   ├── Map[]
│   │   ├── id (uuid), project_id, name, version
│   │   ├── opendrive_url (S3 — .xodr file)
│   │   ├── opencrg_url (S3 — .crg file, nullable)
│   │   ├── 3d_asset_url (S3 — road meshes, buildings, vegetation)
│   │   ├── bounds (PostGIS POLYGON — geographic extent)
│   │   └── created_at, created_by → User
│   │
│   ├── VehicleProfile[]
│   │   ├── id (uuid), project_id, name
│   │   ├── dynamics_config (JSONB — mass, wheelbase, Pacejka coefficients, powertrain)
│   │   ├── sensor_config (JSONB — camera/lidar/radar mount positions and parameters)
│   │   ├── 3d_model_url (S3 — vehicle mesh)
│   │   └── created_at, created_by → User
│   │
│   ├── Scenario[]
│   │   ├── id (uuid), project_id, map_id → Map
│   │   ├── name, description, tags[]
│   │   ├── openscenario_url (S3 — .xosc file)
│   │   ├── ego_vehicle_profile_id → VehicleProfile
│   │   ├── traffic_config (JSONB — IDM/MOBIL params, spawn rules, pedestrian density)
│   │   ├── weather_config (JSONB — rain, fog, snow, time_of_day, sun_angle)
│   │   ├── success_criteria (JSONB — TTC thresholds, collision rules, lane departure limits)
│   │   ├── version (int), parent_version_id (nullable)
│   │   └── created_at, created_by → User
│   │
│   ├── MLModel[]
│   │   ├── id (uuid), project_id, name, version
│   │   ├── format (onnx/tensorrt/torchscript)
│   │   ├── model_url (S3), input_spec (JSONB), output_spec (JSONB)
│   │   └── created_at, created_by → User
│   │
│   ├── SimulationRun[]
│   │   ├── id (uuid), scenario_id → Scenario, ml_model_id → MLModel (nullable)
│   │   ├── status (queued/running/completed/failed/cancelled), mode (interactive/batch/replay)
│   │   ├── gpu_instance_type, gpu_count, sim_duration_seconds, wall_time_seconds
│   │   ├── kpi_results (JSONB — TTC, mAP, collisions, lane_departures), pass_fail, failure_reason
│   │   ├── sensor_data_url (S3), trajectory_url (S3), replay_url (S3)
│   │   └── started_by → User, cost_usd, started_at, completed_at
│   │
│   ├── BatchJob[]
│   │   ├── id (uuid), project_id, name
│   │   ├── scenario_ids (uuid[]), parameter_sweep (JSONB)
│   │   ├── total_runs, completed_runs, failed_runs
│   │   ├── status (queued/running/completed/partial_failure)
│   │   ├── aggregate_kpis (JSONB), regression_flags (JSONB)
│   │   └── created_by → User, created_at, completed_at
│   │
│   └── Report[]
│       ├── id (uuid), project_id, batch_job_id → BatchJob (nullable)
│       ├── template_type (safety_case/regression/perception_eval/custom)
│       ├── generated_pdf_url (S3), created_at
│       └── created_by → User
│
├── Member[]
│   ├── id, org_id, user_id → User, role (admin/engineer/viewer)
│   └── invited_at, accepted_at
│
└── User
    ├── id (uuid), email, name, avatar_url
    ├── auth_provider, created_at
    └── org_memberships[]
```

### API Design

```
Auth:
POST   /api/auth/register                    # Create account
POST   /api/auth/login                       # Login (JWT)
POST   /api/auth/oauth/:provider             # OAuth (Google, GitHub)
POST   /api/auth/refresh                     # Refresh token

Projects:
GET    /api/projects                         # List projects
POST   /api/projects                         # Create project
GET    /api/projects/:id                     # Get project details
PATCH  /api/projects/:id                     # Update project
DELETE /api/projects/:id                     # Archive project

Maps:
GET    /api/projects/:id/maps                # List maps
POST   /api/projects/:id/maps                # Upload OpenDRIVE map
GET    /api/maps/:mid                        # Get map details
GET    /api/maps/:mid/render                 # Get map 3D preview
POST   /api/maps/:mid/validate               # Validate OpenDRIVE conformance

Vehicle Profiles:
GET    /api/projects/:id/vehicles            # List vehicle profiles
POST   /api/projects/:id/vehicles            # Create vehicle profile
GET    /api/vehicles/:vid                    # Get vehicle details
PATCH  /api/vehicles/:vid                    # Update dynamics/sensor config

Scenarios:
GET    /api/projects/:id/scenarios            # List scenarios
POST   /api/projects/:id/scenarios            # Create scenario (or upload .xosc)
GET    /api/scenarios/:sid                    # Get scenario details
PATCH  /api/scenarios/:sid                    # Update scenario
DELETE /api/scenarios/:sid                    # Delete scenario
POST   /api/scenarios/:sid/variants           # Generate parameter sweep variants

ML Models:
POST   /api/projects/:id/models               # Upload ML model (ONNX/TensorRT)
GET    /api/projects/:id/models               # List models
GET    /api/models/:mid                       # Get model details
POST   /api/models/:mid/validate              # Validate model input/output spec

Simulation:
POST   /api/scenarios/:sid/run                # Start interactive simulation
POST   /api/scenarios/:sid/batch              # Start batch simulation
GET    /api/simulation-runs/:rid              # Get run status and KPIs
WS     /ws/simulation-runs/:rid              # Live simulation telemetry stream
POST   /api/simulation-runs/:rid/pause        # Pause simulation
POST   /api/simulation-runs/:rid/resume       # Resume simulation
POST   /api/simulation-runs/:rid/stop         # Stop simulation
GET    /api/simulation-runs/:rid/replay       # Get replay data
GET    /api/simulation-runs/:rid/sensor-data  # Download sensor recordings

Batch Jobs:
GET    /api/projects/:id/batch-jobs           # List batch jobs
GET    /api/batch-jobs/:bid                   # Get batch job status and aggregate KPIs
GET    /api/batch-jobs/:bid/runs              # List runs in batch
POST   /api/batch-jobs/:bid/compare/:bid2     # Compare two batch runs

Analytics & Reports:
GET    /api/projects/:id/analytics/kpis       # Aggregate project KPIs
GET    /api/projects/:id/analytics/heatmap    # Failure location heatmap
POST   /api/projects/:id/reports              # Generate PDF report
GET    /api/reports/:rid/pdf                  # Download PDF

Collaboration:
POST   /api/projects/:id/share               # Create share link
GET    /api/shared/:token                     # Access shared project
POST   /api/projects/:id/comments             # Add comment
```

---

## UI/UX — Key Screens

### 1. Project Dashboard
- Card grid showing projects with thumbnail renders of primary map/scenario and status badges (active, archived)
- Quick-create wizard: choose map (upload OpenDRIVE or select from library), vehicle profile, and initial scenario template
- Recent activity feed: simulation runs completed, batch jobs finished, regression alerts triggered
- GPU usage meter showing consumed vs. allocated GPU hours for the current billing period
- Pinned scenarios showing latest pass/fail status for critical test suites

### 2. 3D Simulation Viewport
- Full-screen WebGPU-rendered driving scene with free camera, chase camera, driver camera, and bird's-eye view modes
- Left panel: scene hierarchy listing ego vehicle, traffic participants, road infrastructure, and weather layers with visibility toggles
- Bottom bar: simulation controls (play, pause, step-frame, speed slider 0.1x-10x, reset), simulation clock, physics FPS
- Floating sensor panels: front camera feed, LiDAR point cloud visualization, radar detection overlay, vehicle dynamics gauges
- Right panel: real-time telemetry — ego speed, steering angle, throttle/brake, TTC to nearest vehicle, current lane ID
- Picture-in-picture: rear camera or side mirror views configurable per sensor configuration

### 3. Scenario Editor
- Split view: 3D preview on the left, OpenSCENARIO visual editor on the right with timeline and condition tree
- Road network overlay: traffic participant spawn points, routes (shown as colored paths), and trigger zones (shown as shaded regions)
- Weather timeline: keyframe-based editor for rain intensity, fog density, sun position, and cloud cover over scenario duration
- Parameter panel: expose any scenario variable (ego speed, traffic density, pedestrian count) as a sweep parameter with min/max/step
- Validation panel: real-time syntax checking of OpenSCENARIO with warnings for unreachable conditions or conflicting triggers
- One-click batch generation: convert a single scenario template into hundreds of parameterized variants

### 4. Batch Simulation Dashboard
- Progress overview: total runs, completed, running, failed, pass rate as a large progress ring with real-time updates
- KPI summary table: rows for each metric (TTC, collision rate, lane departure, mAP), columns for min/max/mean/p95 across all runs
- Failure drill-down: click any failed run to jump directly to the simulation replay at the moment of failure
- Scatter plots: parameter value vs. KPI value to identify failure boundaries (e.g., "collisions spike when fog density > 0.7")
- Comparison mode: overlay KPI distributions from two batch jobs (e.g., model v2.1 vs. v2.0) with statistical significance indicators

### 5. Analytics and KPI Dashboard
- Time-series charts tracking key metrics (mAP, collision rate, TTC distributions) across software versions and dates
- Geographic heatmap: overlay failure locations onto the map to identify problematic road segments or intersections
- Weather correlation matrix: failure rates broken down by rain, fog, snow, night, and glare conditions
- Perception confusion matrix: per-class detection accuracy showing which object types the ML model misclassifies in simulation
- Custom dashboard builder: drag-and-drop KPI widgets with configurable data sources, filters, and time ranges

### 6. Collaboration and Review
- Simulation replay viewer with synchronized playback across camera, LiDAR, radar, and bird's-eye views
- Inline commenting: click on any timestamp or location in the replay to attach a comment visible to all team members
- Version comparison: side-by-side diff of scenario changes (parameters, traffic config, weather) between versions
- Share links: generate password-protected URLs for external reviewers (OEM customers, regulatory auditors) with read-only access
- Activity log: chronological history of all simulation runs, scenario edits, and model uploads with user attribution

---

## Monetization

### Free / Developer Tier
- 1 project, 3 scenarios
- Single-scenario interactive simulation (no batch)
- 20 GPU-hours per month (NVIDIA T4)
- Basic camera and LiDAR simulation (reduced resolution)
- 1 uploaded ML model
- Community support
- DriveSim watermark on exports

### Pro — $499/month
- 10 projects, unlimited scenarios
- Batch simulation up to 100 parallel runs
- 500 GPU-hours per month (T4 and A100)
- Full camera, LiDAR, and radar sensor simulation at native resolution
- Pacejka vehicle dynamics with full suspension model
- 10 uploaded ML models with automated evaluation
- OpenSCENARIO 2.0 and OpenDRIVE import/export
- KPI dashboard and PDF report generation
- Email support with 48-hour response

### Team — $1,499/month (up to 5 seats, $249/additional seat)
- Unlimited projects and scenarios
- Batch simulation up to 5,000 parallel runs
- 3,000 GPU-hours per month (A100/H100)
- Full traffic AI (IDM/MOBIL) with pedestrian and cyclist models
- Unlimited ML models with A/B testing and regression detection
- CI/CD integration (GitHub Actions, GitLab CI, Jenkins)
- Version control and scenario branching
- Team collaboration with role-based access
- Priority GPU queue and Slack/webhook alerts
- Priority support with 24-hour response

### Enterprise — Custom
- Unlimited parallel batch runs and GPU hours
- Dedicated GPU cluster with guaranteed SLA and capacity
- Custom sensor models calibrated to customer hardware
- On-premise or VPC deployment for data-sensitive OEMs
- SAML/SSO, audit logging, IP whitelisting
- Hardware-in-the-loop (HIL) and vehicle-in-the-loop (VIL) integration
- ISO 26262 / SOTIF compliance documentation and traceability
- Dedicated customer success engineer and quarterly business reviews

---

## Go-to-Market Strategy

### Phase 1: Developer Community and Startups (Month 1-3)
- Launch free tier on Product Hunt and Hacker News with demo: "Test your perception model against 1,000 driving scenarios in 10 minutes"
- Publish open-source OpenSCENARIO scenario library on GitHub covering Euro NCAP and NHTSA pre-crash typologies
- Tutorial series: "From CARLA to DriveSim — migrate your AV simulation in 30 minutes" on YouTube and AV blogs
- Partner with 10 university autonomous driving labs for free Team access and co-published benchmark results
- Sponsor Wayve, Comma.ai, and open-source AV community forums

### Phase 2: Tier 1 Suppliers and Mid-Market (Month 3-6)
- Launch batch simulation and CI/CD integration with benchmark comparisons against CARLA and LGSVL (throughput, sensor fidelity, setup time)
- Case studies with early-adopter AV startups showing 10x faster regression testing and 5x reduction in simulation infrastructure costs
- Target ADAS Tier 1 suppliers (Continental, Aptiv, ZF) with Euro NCAP scenario compliance packages
- SEO content: "autonomous driving simulator", "CARLA alternative cloud", "AV simulation platform"
- Exhibit at CES, SAE WCX, and Autonomous Vehicle Technology Expo with live batch simulation demos

### Phase 3: Enterprise OEM and Regulatory (Month 6-12)
- Launch Enterprise tier with on-premise deployment, HIL integration, and ISO 26262 compliance documentation
- Target OEM validation teams (BMW, Toyota, GM) with SOTIF scenario-based safety validation workflows
- Build regulatory partnerships: demonstrate DriveSim as a tool for UNECE R157 and EU type-approval scenario testing
- International expansion: localize for EU, Japan, and China automotive markets with region-specific scenario libraries
- Launch DriveSim Marketplace for community-contributed maps, scenarios, vehicle models, and sensor configurations

### Acquisition Channels
- Direct sales to AV companies and Tier 1 suppliers (high ACV justifies sales-led motion)
- Content marketing targeting "autonomous driving simulation", "AV sensor simulation", "OpenSCENARIO testing" searches
- Partnerships with HD map providers (HERE, TomTom) to bundle DriveSim with map products
- Academic licensing (free/discounted) to establish DriveSim as the standard simulation tool in university AV courses
- Integration partnerships with AV software stacks (Autoware, Apollo) for organic discovery

---

## Success Metrics & KPIs

| Metric | Target (Month 6) | Target (Month 12) |
|--------|-------------------|---------------------|
| Registered organizations | 500 | 2,000 |
| Monthly active projects | 1,200 | 5,000 |
| Simulation runs per month | 50,000 | 500,000 |
| Paying customers (Pro + Team) | 60 | 250 |
| MRR | $45,000 | $220,000 |
| Free-to-paid conversion | 5% | 9% |
| Batch simulation pass rate (platform stability) | 96% | 99% |
| Average scenario setup time (new user) | < 30 min | < 15 min |

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | DriveSim Advantage |
|-----------|-----------|------------|-------------------|
| CARLA | Free/open-source, large community, Unreal Engine rendering, extensive Python API | Difficult to scale, crashes on large maps, limited deterministic replay, no managed cloud, sensor fidelity gaps | Zero-setup cloud platform, 1000x batch scale, ray-traced sensor fidelity, managed infrastructure |
| NVIDIA DRIVE Sim | Photorealistic Omniverse rendering, best-in-class sensor models, NVIDIA ecosystem integration | $500K+/year, requires DGX/RTX infrastructure, NVIDIA lock-in, long procurement cycles | 10-50x cheaper, browser-based, no local GPU required, open standards (OpenSCENARIO/OpenDRIVE) |
| Applied Intuition | Full-stack AV validation platform, strong OEM relationships, comprehensive scenario coverage | $1M+/year, enterprise-only with no self-service, slow onboarding, opaque pricing | Self-service SaaS with transparent pricing, same-day onboarding, accessible to startups and academia |
| LGSVL Simulator | Open-source, good Unity-based rendering, ROS/ROS2 bridge, Apollo/Autoware integration | Development discontinued (2022), no active maintenance, community fragmented, no cloud offering | Actively developed cloud platform, continuous updates, production-grade reliability, professional support |
| rFpro | Excellent vehicle dynamics, high-fidelity terrain, motorsport-grade simulation, HIL integration | Desktop-only, extremely expensive ($200K+/year), focused on OEM proving ground simulation, no ML testing pipeline | Cloud-native ML testing pipeline, batch simulation at scale, perception model evaluation, modern developer UX |

---

## Risks & Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Ray-traced rendering quality insufficient compared to NVIDIA Omniverse | High | Medium | Invest in custom Vulkan RT pipeline with iterative quality improvements, implement domain randomization to reduce reliance on photorealism alone, benchmark against real sensor data with quantitative sim-to-real metrics |
| GPU compute costs erode margins on batch simulation workloads | High | High | Use spot instances for batch jobs (60-70% savings), implement headless rendering for non-interactive runs, adaptive resolution scaling, cache static scene elements across runs sharing the same map |
| OpenSCENARIO 2.0 standard still evolving, breaking changes possible | Medium | Medium | Maintain backward compatibility with OpenSCENARIO 1.x, contribute to ASAM standardization working groups, abstract scenario engine behind internal DSL that can adapt to standard changes |
| Automotive OEMs require on-premise deployment for IP-sensitive simulation data | High | High | Offer VPC deployment on customer cloud accounts from day one, build air-gapped deployment option for Enterprise tier, implement end-to-end encryption for all simulation data in transit and at rest |
| NVIDIA bundles DRIVE Sim at discount with DGX hardware purchases, undercutting DriveSim on price for large OEMs | High | Medium | Differentiate on developer experience, self-service onboarding, and open-standards support; target the long tail of AV companies and suppliers that NVIDIA ignores; maintain 10x lower entry price point |

---

## MVP Scope (v1.0)

### In Scope
- Project creation with OpenDRIVE map upload or selection from built-in map library (urban intersection, highway segment, suburban road)
- GPU ray-traced 3D environment rendering with basic weather (clear, rain, fog) and time-of-day (noon, dusk, night)
- Camera sensor simulation with PBR materials, dynamic lighting, and ground-truth segmentation/bounding box output
- LiDAR sensor simulation with configurable beam patterns (VLP-16, VLP-32 presets) and point cloud output with semantic labels
- Bicycle model vehicle dynamics with basic Pacejka tire model for ego vehicle
- IDM traffic AI with configurable density and basic lane-changing behavior
- Single-scenario interactive simulation with real-time 3D viewport in browser
- ONNX model upload and open-loop perception evaluation against simulation ground truth
- Basic KPI computation (mAP, collision count, lane departure) and results storage

### Out of Scope (v1.1+)
- Radar sensor simulation with Doppler and RCS (v1.1)
- Full Pacejka MF 6.2 with suspension dynamics and powertrain modeling (v1.1)
- OpenSCENARIO 2.0 parser and visual scenario editor (v1.1)
- Batch simulation with parallel execution and parameter sweeps (v1.2 — highest priority post-MVP)
- CI/CD integration for automated regression testing (v1.2)
- MOBIL lane-changing, pedestrian, and cyclist traffic agents (v1.2)
- Closed-loop ML model testing with gRPC/ROS2 interface (v1.3)
- Team collaboration, version control, and sharing (v1.3)
- Analytics dashboard with heatmaps and trend analysis (v1.3)
- Enterprise features: on-premise deployment, SSO, HIL integration (v2.0)

### MVP Timeline: 10 weeks
- Week 1-2: Auth system, project data model, OpenDRIVE parser and road mesh generation, basic 3D scene rendering with WebGPU viewport and server-side Vulkan pipeline
- Week 3-4: Ray-traced environment rendering — PBR materials for roads/vehicles/buildings, dynamic lighting (sun + headlights), basic weather effects (rain particles, fog shader), asset pipeline for vehicle and environment models
- Week 5-6: Camera and LiDAR sensor simulation — GPU-rendered camera images with lens model, LiDAR ray-casting with beam pattern presets, ground-truth output generation (segmentation, bounding boxes, point labels), sensor data streaming to browser
- Week 7-8: Vehicle dynamics (bicycle + basic Pacejka), IDM traffic AI with lane-following, scenario definition (spawn points, traffic density, weather config), ONNX model upload and open-loop perception evaluation pipeline
- Week 9-10: KPI computation and results storage, simulation replay, project dashboard, billing integration (Stripe), built-in map and scenario library, documentation, load testing, launch preparation
