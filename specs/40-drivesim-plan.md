# 40. DriveSim — Cloud Autonomous Driving Simulation Platform

## Implementation Plan

**MVP Scope:** Browser-based 3D viewport with WebGPU-rendered ray-traced driving scenes streamed from server-side Vulkan RT pipeline running on NVIDIA A100/H100 GPU clusters, visual scenario editor with drag-and-drop traffic participant placement and OpenDRIVE map import supporting road geometry up to 20km² urban environments, custom Rust-based vehicle dynamics engine implementing bicycle model + Pacejka Magic Formula tire model (simplified MF 5.2) for ego vehicle simulated at 1 kHz with RK4 integration, GPU-accelerated LiDAR ray-casting via Vulkan compute shaders producing point clouds matching Velodyne VLP-16/VLP-32 beam patterns with configurable noise and atmospheric effects, PBR camera sensor simulation with HDR tonemapping and lens distortion outputting 1920x1080 RGB images plus ground-truth semantic segmentation and 2D bounding boxes, IDM (Intelligent Driver Model) traffic AI with 50+ concurrent vehicles and basic MOBIL lane-changing behavior, ONNX model upload for open-loop perception evaluation comparing detections against simulation ground truth using mAP/IoU metrics, single-scenario interactive simulation with real-time telemetry streaming via WebSocket, basic KPI computation (collision detection, time-to-collision, lane departure), Stripe billing with three tiers (Free / Pro $499/mo / Team $1,499/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth, scenario orchestration |
| Simulation Engine | Rust (native GPU) | Vulkan RT pipeline, vehicle dynamics, sensor simulation, traffic AI |
| Rendering | Vulkan 1.3 + Ray Tracing | Hardware RT on NVIDIA RTX GPUs (A100/H100), PBR materials, dynamic lighting |
| Camera Sensor | Vulkan graphics pipeline | Deferred rendering, post-processing effects, lens distortion, HDR tonemapping |
| LiDAR/Radar | Vulkan compute shaders | GPU ray-casting, multi-return, atmospheric scatter, RCS computation |
| Vehicle Dynamics | Custom Rust solver | Pacejka MF tire model, RK4 integrator at 1 kHz, bicycle + full 4-wheel models |
| Traffic AI | Rust (parallel update) | IDM car-following, MOBIL lane-change, social force pedestrian model |
| ML Runtime | ONNX Runtime (GPU) | Inference on CUDA for user-uploaded perception models |
| Map Parser | Rust (custom) | OpenDRIVE 1.6/1.7 XML parser, road mesh generation, lane graph extraction |
| Python Service | FastAPI 0.109 | Model evaluation metrics (mAP, MOTA), batch analytics, report generation |
| Database | PostgreSQL 16 | Projects, scenarios, maps, vehicles, simulation runs, KPI results |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, async, migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth, bcrypt password hashing, API key for CI/CD |
| Object Storage | AWS S3 | OpenDRIVE maps, 3D assets, ONNX models, sensor recordings, result archives |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management, React Three Fiber for scene editing |
| 3D Viewport | WebGPU (Chrome 113+) | Streamed ray-traced frames from server, Three.js fallback for editing |
| UI Components | Tailwind CSS, Radix UI | shadcn/ui primitives, React Flow for OpenSCENARIO visual editor |
| Real-time | WebSocket (Axum) | Live simulation telemetry (vehicle state, sensor data), batch progress |
| Job Queue | Redis 7 + Tokio tasks | Batch simulation job distribution across GPU cluster nodes |
| GPU Orchestration | Kubernetes (EKS) | NVIDIA GPU Operator, spot instances, autoscaling based on queue depth |
| Billing | Stripe | Checkout Sessions, Customer Portal, usage-based GPU hours metering |
| CDN | CloudFront | Static assets, 3D models, frontend bundle |
| Monitoring | Prometheus + Grafana, Sentry | GPU utilization, frame time, convergence metrics, error tracking |
| CI/CD | GitHub Actions | Rust tests, Docker image push, scenario validation |

### Key Architecture Decisions

1. **Server-side ray-traced rendering with WebGPU client streaming**: Running Vulkan RT on server GPUs produces photorealistic sensor data that browsers cannot render locally. The server encodes ray-traced frames as H.264/VP9 video streams (for viewport) or sends raw sensor data (LiDAR point clouds, camera images) via WebSocket. The WebGPU client viewport decodes and displays the stream with sub-100ms latency. This architecture enables high-fidelity sensor simulation without requiring users to have local RTX GPUs.

2. **Hybrid traffic AI with spatial hashing for collision detection**: IDM/MOBIL traffic updates run in parallel across 50-200 concurrent vehicles at 10 Hz. A spatial hash grid (cell size 10m) provides O(1) nearest-neighbor queries for lane changes and collision avoidance. This scales to 500+ vehicles in large maps while maintaining real-time performance (10ms update budget per frame).

3. **Separate vehicle dynamics timestep (1 kHz) from rendering (30-60 Hz)**: High-frequency dynamics integration (1ms timestep) is essential for accurate tire slip and suspension behavior, especially during emergency maneuvers. The solver runs at 1 kHz and outputs state snapshots to the renderer at 30-60 Hz, decoupling physics accuracy from visual smoothness.

4. **OpenDRIVE lane graph extraction for routing and behavior**: The OpenDRIVE parser generates a directed graph of lane centerlines with connectivity metadata (successors, predecessors, lane changes). Traffic AI and planning queries use this graph for pathfinding and rule-based behaviors (stop at signals, yield at intersections). This avoids re-parsing geometry every frame.

5. **S3 + PostgreSQL split for map and model storage**: Large binary assets (OpenDRIVE .xodr files up to 50MB, 3D road meshes up to 500MB, ONNX models up to 200MB) live in S3 with CDN caching. PostgreSQL stores searchable metadata (map bounds, scenario parameters, KPI results) with PostGIS POLYGON queries for spatial filtering. This keeps the database fast while supporting unlimited asset libraries.

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "postgis";

-- Users
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email TEXT UNIQUE NOT NULL,
    password_hash TEXT,
    name TEXT NOT NULL,
    avatar_url TEXT,
    auth_provider TEXT DEFAULT 'email',
    auth_provider_id TEXT,
    plan TEXT NOT NULL DEFAULT 'free',
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    gpu_quota_hours_monthly INTEGER DEFAULT 20,
    gpu_hours_used_current_period REAL DEFAULT 0.0,
    billing_period_start DATE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX users_email_idx ON users(email);
CREATE INDEX users_stripe_idx ON users(stripe_customer_id);

-- Organizations (for Team plan)
CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    slug TEXT UNIQUE NOT NULL,
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    plan TEXT NOT NULL DEFAULT 'team',
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    gpu_quota_hours_monthly INTEGER DEFAULT 3000,
    gpu_hours_used_current_period REAL DEFAULT 0.0,
    settings JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX orgs_slug_idx ON organizations(slug);
CREATE INDEX orgs_owner_idx ON organizations(owner_id);

-- Organization Members
CREATE TABLE org_members (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    org_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role TEXT NOT NULL DEFAULT 'member',
    joined_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(org_id, user_id)
);
CREATE INDEX org_members_org_idx ON org_members(org_id);
CREATE INDEX org_members_user_idx ON org_members(user_id);

-- Projects
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    visibility TEXT NOT NULL DEFAULT 'private',
    thumbnail_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX projects_owner_idx ON projects(owner_id);
CREATE INDEX projects_org_idx ON projects(org_id);
CREATE INDEX projects_updated_idx ON projects(updated_at DESC);

-- HD Maps
CREATE TABLE maps (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    opendrive_url TEXT NOT NULL,
    opencrg_url TEXT,
    mesh_url TEXT,
    bounds GEOMETRY(POLYGON, 4326),
    road_count INTEGER DEFAULT 0,
    junction_count INTEGER DEFAULT 0,
    lane_count INTEGER DEFAULT 0,
    is_validated BOOLEAN DEFAULT false,
    validation_errors JSONB DEFAULT '[]',
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX maps_project_idx ON maps(project_id);
CREATE INDEX maps_bounds_idx ON maps USING gist(bounds);

-- Vehicle Profiles
CREATE TABLE vehicle_profiles (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    vehicle_class TEXT NOT NULL DEFAULT 'sedan',
    dynamics_config JSONB NOT NULL,
    sensor_config JSONB NOT NULL,
    model_3d_url TEXT,
    thumbnail_url TEXT,
    is_builtin BOOLEAN DEFAULT false,
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX vehicle_profiles_project_idx ON vehicle_profiles(project_id);

-- Scenarios
CREATE TABLE scenarios (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    map_id UUID NOT NULL REFERENCES maps(id) ON DELETE RESTRICT,
    ego_vehicle_id UUID NOT NULL REFERENCES vehicle_profiles(id) ON DELETE RESTRICT,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    tags TEXT[] DEFAULT '{}',
    openscenario_url TEXT,
    scenario_config JSONB NOT NULL,
    weather_config JSONB NOT NULL,
    traffic_config JSONB NOT NULL,
    success_criteria JSONB NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    parent_scenario_id UUID REFERENCES scenarios(id) ON DELETE SET NULL,
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX scenarios_project_idx ON scenarios(project_id);
CREATE INDEX scenarios_map_idx ON scenarios(map_id);
CREATE INDEX scenarios_tags_idx ON scenarios USING gin(tags);

-- ML Models
CREATE TABLE ml_models (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    version TEXT NOT NULL,
    model_type TEXT NOT NULL,
    format TEXT NOT NULL,
    model_url TEXT NOT NULL,
    input_spec JSONB NOT NULL,
    output_spec JSONB NOT NULL,
    size_bytes BIGINT,
    uploaded_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX ml_models_project_idx ON ml_models(project_id);

-- Simulation Runs
CREATE TABLE simulation_runs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    scenario_id UUID NOT NULL REFERENCES scenarios(id) ON DELETE CASCADE,
    ml_model_id UUID REFERENCES ml_models(id) ON DELETE SET NULL,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    status TEXT NOT NULL DEFAULT 'pending',
    mode TEXT NOT NULL DEFAULT 'interactive',
    gpu_instance_type TEXT,
    gpu_count INTEGER DEFAULT 1,
    sim_duration_seconds REAL,
    wall_time_seconds REAL,
    kpi_results JSONB,
    pass_fail BOOLEAN,
    failure_reason TEXT,
    sensor_data_url TEXT,
    trajectory_url TEXT,
    replay_url TEXT,
    cost_usd REAL,
    error_message TEXT,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX sim_runs_scenario_idx ON simulation_runs(scenario_id);
CREATE INDEX sim_runs_user_idx ON simulation_runs(user_id);
CREATE INDEX sim_runs_status_idx ON simulation_runs(status);
CREATE INDEX sim_runs_created_idx ON simulation_runs(created_at DESC);

-- Batch Jobs
CREATE TABLE batch_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    scenario_ids UUID[] NOT NULL,
    parameter_sweep JSONB,
    total_runs INTEGER NOT NULL,
    completed_runs INTEGER DEFAULT 0,
    failed_runs INTEGER DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'queued',
    aggregate_kpis JSONB,
    regression_flags JSONB DEFAULT '[]',
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at TIMESTAMPTZ
);
CREATE INDEX batch_jobs_project_idx ON batch_jobs(project_id);
CREATE INDEX batch_jobs_status_idx ON batch_jobs(status);

-- GPU Usage Tracking
CREATE TABLE gpu_usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    simulation_run_id UUID REFERENCES simulation_runs(id) ON DELETE SET NULL,
    gpu_type TEXT NOT NULL,
    gpu_hours REAL NOT NULL,
    cost_usd REAL NOT NULL,
    billing_period_start DATE NOT NULL,
    billing_period_end DATE NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX gpu_usage_user_period_idx ON gpu_usage_records(user_id, billing_period_start);
CREATE INDEX gpu_usage_org_period_idx ON gpu_usage_records(org_id, billing_period_start);
```

### Rust SQLx Structs

```rust
// src/db/models.rs

use chrono::{DateTime, Utc, NaiveDate};
use serde::{Deserialize, Serialize};
use sqlx::FromRow;
use uuid::Uuid;

#[derive(Debug, FromRow, Serialize)]
pub struct User {
    pub id: Uuid,
    pub email: String,
    #[sqlx(skip)]
    #[serde(skip)]
    pub password_hash: Option<String>,
    pub name: String,
    pub avatar_url: Option<String>,
    pub auth_provider: String,
    pub plan: String,
    pub stripe_customer_id: Option<String>,
    pub gpu_quota_hours_monthly: i32,
    pub gpu_hours_used_current_period: f32,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Scenario {
    pub id: Uuid,
    pub project_id: Uuid,
    pub map_id: Uuid,
    pub ego_vehicle_id: Uuid,
    pub name: String,
    pub scenario_config: serde_json::Value,
    pub weather_config: serde_json::Value,
    pub traffic_config: serde_json::Value,
    pub success_criteria: serde_json::Value,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct VehicleDynamicsConfig {
    pub mass_kg: f64,
    pub wheelbase_m: f64,
    pub track_width_m: f64,
    pub cg_height_m: f64,
    pub inertia_z_kgm2: f64,
    pub pacejka_params: PacejkaParams,
    pub max_steer_angle_rad: f64,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct PacejkaParams {
    pub b: f64,  // Stiffness factor
    pub c: f64,  // Shape factor
    pub d: f64,  // Peak factor
    pub e: f64,  // Curvature factor
}

#[derive(Debug, Deserialize)]
pub struct WeatherConfig {
    pub rain_intensity: f64,  // 0.0-1.0
    pub fog_density: f64,
    pub time_of_day: f64,  // 0.0 = midnight, 12.0 = noon
    pub sun_angle_deg: f64,
    pub temperature_celsius: f64,
}

#[derive(Debug, Deserialize)]
pub struct TrafficConfig {
    pub vehicle_density: f64,
    pub idm_params: IdmParams,
    pub mobil_params: MobilParams,
    pub pedestrian_density: f64,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct IdmParams {
    pub desired_velocity_ms: f64,
    pub time_headway_s: f64,
    pub min_spacing_m: f64,
    pub max_accel_ms2: f64,
    pub comfortable_decel_ms2: f64,
}
```

---

## Core Implementation: Vehicle Dynamics

```rust
// src/dynamics/vehicle.rs

use nalgebra as na;

pub struct VehicleState {
    pub position: na::Vector3<f64>,
    pub velocity: na::Vector3<f64>,
    pub orientation: na::UnitQuaternion<f64>,
    pub angular_velocity: na::Vector3<f64>,
    pub wheel_speeds: [f64; 4],
    pub steering_angle: f64,
}

pub struct VehicleDynamics {
    pub config: VehicleDynamicsConfig,
    pub state: VehicleState,
    dt: f64,  // 0.001s (1 kHz)
}

impl VehicleDynamics {
    pub fn step(&mut self, throttle: f64, brake: f64, steering_input: f64) {
        // RK4 integration
        let k1 = self.compute_derivatives(&self.state, throttle, brake, steering_input);
        let k2 = self.compute_derivatives(&self.add_scaled_state(&self.state, &k1, 0.5 * self.dt), throttle, brake, steering_input);
        let k3 = self.compute_derivatives(&self.add_scaled_state(&self.state, &k2, 0.5 * self.dt), throttle, brake, steering_input);
        let k4 = self.compute_derivatives(&self.add_scaled_state(&self.state, &k3, self.dt), throttle, brake, steering_input);

        let dx = (k1 + k2 * 2.0 + k3 * 2.0 + k4) * (self.dt / 6.0);
        self.integrate_state(dx);
    }

    fn compute_derivatives(&self, state: &VehicleState, throttle: f64, brake: f64, steering_input: f64) -> StateDerivative {
        let cfg = &self.config;

        // Transform velocity to body frame
        let v_body = state.orientation.inverse() * state.velocity;
        let vx = v_body.x;
        let vy = v_body.y;

        // Bicycle model: slip angles
        let beta = if vx.abs() > 0.1 { (vy / vx).atan() } else { 0.0 };
        let alpha_f = state.steering_angle - beta - (cfg.wheelbase_m / 2.0) * state.angular_velocity.z / vx.max(0.1);
        let alpha_r = -beta + (cfg.wheelbase_m / 2.0) * state.angular_velocity.z / vx.max(0.1);

        // Pacejka lateral forces
        let f_y_front = self.pacejka_lateral_force(alpha_f, cfg.mass_kg * 9.81 * 0.5);
        let f_y_rear = self.pacejka_lateral_force(alpha_r, cfg.mass_kg * 9.81 * 0.5);

        // Longitudinal force
        let f_x = if throttle > 0.0 {
            (cfg.powertrain_max_torque_nm * throttle / cfg.wheelbase_m).min(cfg.mass_kg * 5.0)
        } else {
            -cfg.mass_kg * 8.0 * brake
        };

        // Drag
        let drag = -0.5 * 1.225 * 0.3 * 2.5 * vx.powi(2) * vx.signum();

        // Net forces
        let f_net_body = na::Vector3::new(
            f_x + drag - f_y_front * state.steering_angle.sin(),
            f_y_front * state.steering_angle.cos() + f_y_rear,
            -cfg.mass_kg * 9.81
        );

        let accel_body = f_net_body / cfg.mass_kg;
        let moment_z = f_y_front * (cfg.wheelbase_m / 2.0) * state.steering_angle.cos()
                     - f_y_rear * (cfg.wheelbase_m / 2.0);
        let angular_accel_z = moment_z / cfg.inertia_z_kgm2;

        StateDerivative {
            velocity: state.orientation * accel_body,
            angular_velocity: na::Vector3::new(0.0, 0.0, angular_accel_z),
            position: state.velocity,
            orientation_rate: state.angular_velocity.z,
            steering_rate: (steering_input * cfg.max_steer_angle_rad - state.steering_angle) * 10.0,
        }
    }

    fn pacejka_lateral_force(&self, slip_angle_rad: f64, normal_force_n: f64) -> f64 {
        let p = &self.config.pacejka_params;
        let alpha = slip_angle_rad;
        let d = p.d * normal_force_n;
        let bcd = p.b * p.c * p.d;
        let b = bcd / (p.c * d);
        let arg = b * alpha - p.e * (b * alpha - (b * alpha).atan());
        d * (p.c * arg.atan()).sin()
    }
}

struct StateDerivative {
    velocity: na::Vector3<f64>,
    angular_velocity: na::Vector3<f64>,
    position: na::Vector3<f64>,
    orientation_rate: f64,
    steering_rate: f64,
}
```

---

## Core Implementation: LiDAR Sensor (Vulkan Compute Shader)

```glsl
// shaders/lidar_raycast.comp
#version 460
#extension GL_EXT_ray_query : require

layout(local_size_x = 256) in;

layout(binding = 0) uniform accelerationStructureEXT topLevelAS;
layout(binding = 1, std430) buffer LidarPoints {
    vec4 points[];  // [x, y, z, intensity]
};
layout(binding = 2, std430) buffer LidarLabels {
    uint labels[];  // Semantic labels
};

uniform vec3 lidar_position;
uniform mat3 lidar_rotation;
uniform float vertical_fov_min;
uniform float vertical_fov_max;
uniform uint num_beams;
uniform float azimuth_angle_deg;
uniform float max_range_m;
uniform float rain_intensity;

void main() {
    uint ray_idx = gl_GlobalInvocationID.x;
    uint beam_idx = ray_idx % num_beams;

    // Vertical angle for this beam
    float v_angle = vertical_fov_min + (vertical_fov_max - vertical_fov_min) * float(beam_idx) / float(num_beams - 1);

    // Ray direction
    float azimuth_rad = radians(azimuth_angle_deg);
    vec3 direction = vec3(
        cos(v_angle) * cos(azimuth_rad),
        cos(v_angle) * sin(azimuth_rad),
        sin(v_angle)
    );
    direction = lidar_rotation * direction;

    // Trace ray
    rayQueryEXT rayQuery;
    rayQueryInitializeEXT(rayQuery, topLevelAS, gl_RayFlagsOpaqueEXT,
        0xFF, lidar_position, 0.01, direction, max_range_m);

    while(rayQueryProceedEXT(rayQuery)) {
        // Rain scatter check
        float scatter_prob = rain_intensity * 0.001 * rayQueryGetIntersectionTEXT(rayQuery, true);
        if (random(ray_idx) < scatter_prob) {
            float fake_range = random(ray_idx) * max_range_m * 0.3;
            points[ray_idx] = vec4(lidar_position + direction * fake_range, 0.1);
            labels[ray_idx] = 0;
            return;
        }
    }

    if (rayQueryGetIntersectionTypeEXT(rayQuery, true) == gl_RayQueryCommittedIntersectionTriangleEXT) {
        float range = rayQueryGetIntersectionTEXT(rayQuery, true);
        vec3 hit_pos = rayQueryGetIntersectionWorldPositionEXT(rayQuery, true);
        vec3 normal = rayQueryGetIntersectionWorldNormalEXT(rayQuery, true);
        uint material_id = rayQueryGetIntersectionInstanceIdEXT(rayQuery, true);

        // Intensity from reflectivity and angle
        float cos_theta = abs(dot(normal, -direction));
        float reflectivity = getMaterialReflectivity(material_id);
        float intensity = reflectivity * cos_theta / (range * range);

        points[ray_idx] = vec4(hit_pos, clamp(intensity, 0.0, 1.0));
        labels[ray_idx] = getMaterialSemanticLabel(material_id);
    } else {
        points[ray_idx] = vec4(0.0);
        labels[ray_idx] = 0;
    }
}
```

---

## Core Implementation: Traffic AI (IDM + MOBIL)

```rust
// src/traffic/idm_mobil.rs

pub struct TrafficVehicle {
    pub id: Uuid,
    pub position: na::Vector3<f64>,
    pub velocity: f64,
    pub lane_id: u32,
    pub route: Vec<u32>,
    pub idm_params: IdmParams,
}

pub struct TrafficSimulator {
    vehicles: Vec<TrafficVehicle>,
    spatial_hash: SpatialHashGrid,
    lane_graph: LaneGraph,
}

impl TrafficSimulator {
    pub fn update(&mut self, dt: f64) {
        // Update spatial hash
        self.spatial_hash.clear();
        for (i, v) in self.vehicles.iter().enumerate() {
            self.spatial_hash.insert(v.position, i);
        }

        // Parallel IDM acceleration computation
        let accelerations: Vec<f64> = self.vehicles.par_iter()
            .map(|v| self.compute_idm_acceleration(v))
            .collect();

        // Apply accelerations
        for (i, v) in self.vehicles.iter_mut().enumerate() {
            v.velocity = (v.velocity + accelerations[i] * dt).max(0.0);
            v.position += self.lane_graph.get_lane_direction(v.lane_id) * v.velocity * dt;

            // MOBIL lane-change decision (staggered)
            if i % 10 == 0 {
                if let Some(new_lane) = self.evaluate_lane_change(v) {
                    v.lane_id = new_lane;
                }
            }
        }
    }

    fn compute_idm_acceleration(&self, vehicle: &TrafficVehicle) -> f64 {
        let p = &vehicle.idm_params;
        let v = vehicle.velocity;
        let v_desired = p.desired_velocity_ms;
        let accel_free = p.max_accel_ms2 * (1.0 - (v / v_desired).powi(4));

        if let Some(leader) = self.spatial_hash.query_ahead(vehicle.position, vehicle.lane_id, 150.0) {
            let s = (leader.position - vehicle.position).norm();
            let dv = v - leader.velocity;
            let s_star = p.min_spacing_m + v * p.time_headway_s
                       + (v * dv) / (2.0 * (p.max_accel_ms2 * p.comfortable_decel_ms2).sqrt());
            accel_free - p.max_accel_ms2 * (s_star / s).powi(2)
        } else {
            accel_free
        }
    }

    fn evaluate_lane_change(&self, vehicle: &TrafficVehicle) -> Option<u32> {
        let adjacent_lanes = self.lane_graph.get_adjacent_lanes(vehicle.lane_id);

        for target_lane in adjacent_lanes {
            let (front_gap, rear_gap) = self.compute_gaps(vehicle, target_lane);
            if rear_gap < vehicle.mobil_params.min_gap_m || front_gap < vehicle.mobil_params.min_gap_m {
                continue;
            }

            let accel_old = self.compute_idm_acceleration(vehicle);
            let mut vehicle_target = vehicle.clone();
            vehicle_target.lane_id = target_lane;
            let accel_new = self.compute_idm_acceleration(&vehicle_target);

            let advantage = accel_new - accel_old;
            if advantage > 0.1 {  // Simple threshold (full MOBIL adds politeness)
                return Some(target_lane);
            }
        }
        None
    }
}

struct SpatialHashGrid {
    cell_size: f64,
    cells: HashMap<(i32, i32), Vec<usize>>,
}

impl SpatialHashGrid {
    fn insert(&mut self, pos: na::Vector3<f64>, idx: usize) {
        let key = ((pos.x / self.cell_size).floor() as i32, (pos.y / self.cell_size).floor() as i32);
        self.cells.entry(key).or_default().push(idx);
    }
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–5)

**Day 1-2: Backend scaffold**
- Cargo workspace with Axum, SQLx, Redis, S3 dependencies
- Docker Compose: PostgreSQL + Redis + MinIO
- `src/main.rs`, `src/config.rs`, `src/state.rs`, `src/error.rs`
- Database migrations: all 9 tables with indexes

**Day 3-4: Authentication**
- JWT auth middleware, OAuth (Google + GitHub)
- User CRUD, organization CRUD, project CRUD
- Integration tests for auth flow

**Day 5: Seed data**
- 3 built-in maps (urban 1km², highway 5km, suburban 2km²)
- 5 vehicle profiles with default Pacejka parameters
- Tests: project creation, map upload, vehicle profile creation

### Phase 2 — OpenDRIVE Parser + Map Management (Days 6–10)

**Day 6-7: OpenDRIVE parser**
- XML parsing with `quick-xml`
- Road reference line interpolation (cubic splines)
- Lane graph extraction (directed graph with successors/predecessors)

**Day 8-9: Road mesh generation**
- Triangle mesh from lane boundaries
- PBR material assignment (asphalt, concrete, markings)
- Export to binary format, S3 upload

**Day 10: Map API handlers**
- Upload, list, get, validate, download endpoints
- Background job for mesh generation
- Tests with 3 built-in maps

### Phase 3 — Vulkan Rendering Foundation (Days 11–17)

**Day 11-12: Vulkan RT setup**
- Instance, device, queue creation with RT extensions
- Ray tracing pipeline (ray-gen, closest-hit, miss shaders)
- BLAS/TLAS builders for road mesh and vehicles

**Day 13-14: PBR materials**
- Material system (albedo, metallic, roughness, normals)
- Cook-Torrance BRDF shader with IBL
- Built-in material library (asphalt wet/dry, concrete, glass, vegetation)

**Day 15-16: Camera sensor**
- Ray-gen shader with lens distortion (Brown-Conrady)
- Deferred shading: G-buffer → lighting → post-processing
- HDR tonemapping (ACES)
- Output RGB + depth + semantic segmentation

**Day 17: Headless rendering + streaming**
- H.264 encoding with ffmpeg
- WebSocket streaming to client
- Tests on AWS g4dn (T4 GPU)

### Phase 4 — Vehicle Dynamics + Traffic AI (Days 18–24)

**Day 18-19: Bicycle model**
- 2D bicycle model with RK4 at 1 kHz
- Unit tests: constant radius turn, braking

**Day 20-21: Pacejka tire model**
- MF lateral force implementation
- Integration with bicycle model
- Tests: slip angle curves, emergency lane change

**Day 22-23: IDM + MOBIL**
- IDM car-following, MOBIL lane-change
- Spatial hash grid for neighbor queries
- Parallel update with Rayon for 50+ vehicles

**Day 24: Lane graph routing**
- A* pathfinding on lane graph
- Route following with junction transitions

### Phase 5 — LiDAR Sensor Simulation (Days 25–29)

**Day 25-26: LiDAR compute shader**
- Ray-casting for VLP-16/VLP-32 beam patterns
- Point cloud output (XYZ + intensity)

**Day 27: Intensity + multi-return**
- Reflectivity-based intensity
- Multi-return support (first, strongest, last)

**Day 28: Atmospheric effects**
- Rain scatter (probabilistic early returns)
- Fog attenuation, noise injection

**Day 29: Ground-truth labels**
- Semantic labels from material IDs
- .pcd export, S3 upload
- Tests: validate point count and intensity

### Phase 6 — Scenario System + Frontend (Days 30–36)

**Day 30-31: Scenario API**
- Create, list, get, update, delete scenario
- Scenario executor: load config, spawn traffic, set weather

**Day 32-33: React frontend**
- Project setup: React 19 + Vite + Tailwind + Radix UI
- Dashboard, scenario editor pages
- Auth flow with JWT

**Day 34-35: 3D viewport**
- WebGPU canvas for H.264 stream
- WebSocket connection, MediaSource decode
- Three.js fallback for editing

**Day 36: Scenario editor UI**
- Drag-and-drop vehicle placement
- Route drawing, weather sliders, traffic config
- Real-time preview

### Phase 7 — Simulation Orchestration + KPIs (Days 37–42)

**Day 37-38: Simulation worker**
- Redis job queue consumer
- Spawn sim engine, stream telemetry via WebSocket

**Day 39: KPI computation**
- FastAPI service: TTC, collision detection, lane departure
- mAP computation for perception eval

**Day 40-41: ONNX model eval**
- Load ONNX, run inference on camera images
- Compare detections to ground truth (IoU, mAP)

**Day 42: Telemetry dashboard**
- Real-time charts (speed, steering, TTC)
- LiDAR viewer, camera feed with bboxes

### Phase 8 — Billing + Deployment + Launch (Days 43–47)

**Day 43: Stripe integration**
- Create subscription, webhooks, usage metering
- Quota enforcement

**Day 44: Kubernetes deployment**
- API deployment (3 replicas), GPU worker daemonset
- NVIDIA GPU Operator, autoscaling

**Day 45: Production infra (Terraform)**
- VPC, EKS, RDS, ElastiCache, S3, CloudFront
- GPU spot instances

**Day 46: Documentation**
- OpenAPI spec, user guide
- 5 example scenarios, 3 sample ONNX models

**Day 47: Load testing + launch**
- 100 concurrent sims, 1000 sequential runs
- Sentry monitoring
- Soft launch to 10 beta users

---

## Validation Benchmarks

### 1. Rendering Quality — FID Score
**Metric:** Fréchet Inception Distance between simulated and real dashcam images
**Target:** FID < 50
**Method:** 1,000 Waymo frames vs. 1,000 DriveSim frames, compute FID with Inception-v3
**Interpretation:** FID < 50 indicates photorealistic quality reducing sim-to-real gap

### 2. LiDAR Fidelity — Chamfer Distance
**Metric:** Chamfer Distance between simulated and real LiDAR scans
**Target:** < 0.15m (95th percentile)
**Method:** Scan real parking lot with VLP-32, reconstruct in DriveSim, compare point clouds
**Interpretation:** Sub-sensor-resolution accuracy validates ray-casting

### 3. Vehicle Dynamics — Trajectory Deviation
**Metric:** RMS lateral deviation for ISO 3888 double lane-change
**Target:** < 0.3m over 120m maneuver
**Method:** Real vehicle GPS vs. simulated trajectory with matching params
**Interpretation:** < 10% lane width validates Pacejka model

### 4. Traffic AI — Headway Distribution
**Metric:** Kolmogorov-Smirnov statistic vs. NGSIM dataset
**Target:** KS < 0.15 (p > 0.05)
**Method:** Extract 10,000 headways from NGSIM, compare to DriveSim highway scenario
**Interpretation:** Validates IDM parameter choices

### 5. Perception Eval — mAP Correlation
**Metric:** Pearson r between mAP on simulated vs. real test sets
**Target:** r > 0.85
**Method:** Evaluate YOLOv8 variants on Waymo (real) vs. DriveSim (sim), correlate mAP scores
**Interpretation:** Enables reliable regression testing in simulation

---

## Post-MVP Roadmap

### v1.1 — Batch Simulation (Weeks 8-10)
- Parallel execution: 100+ scenario variants
- Parameter sweep: combinatorial + Latin hypercube
- Aggregate KPI dashboard, CI/CD integration

### v1.2 — Radar + Advanced Dynamics (Weeks 11-13)
- 77 GHz radar with Doppler and RCS
- Full Pacejka MF 6.2 with combined slip
- Suspension dynamics, powertrain modeling
- OpenCRG road surface profiles

### v1.3 — OpenSCENARIO 2.0 (Weeks 14-16)
- Parser and visual editor (React Flow)
- Stochastic parameters, Monte Carlo generation
- v1.x backward compatibility

### v1.4 — Closed-Loop + Collaboration (Weeks 17-19)
- gRPC/ROS2 interface for planning/control stacks
- Scenario version control (Git-like)
- Team workspaces, shared replays

### v2.0 — Enterprise (Weeks 20-24)
- On-premise/VPC deployment
- SAML/SSO, HIL integration
- ISO 26262 compliance docs
- Custom sensor calibration

---

## API Design

```
Auth:
POST   /api/auth/register
POST   /api/auth/login
POST   /api/auth/oauth/:provider
GET    /api/auth/me

Projects:
GET    /api/projects
POST   /api/projects
GET    /api/projects/:id
PATCH  /api/projects/:id

Maps:
GET    /api/projects/:id/maps
POST   /api/projects/:id/maps
GET    /api/maps/:mid
POST   /api/maps/:mid/validate

Vehicles:
GET    /api/projects/:id/vehicles
POST   /api/projects/:id/vehicles
GET    /api/vehicles/:vid

Scenarios:
GET    /api/projects/:id/scenarios
POST   /api/projects/:id/scenarios
GET    /api/scenarios/:sid
PATCH  /api/scenarios/:sid

ML Models:
POST   /api/projects/:id/models
GET    /api/models/:mid

Simulation:
POST   /api/scenarios/:sid/run
GET    /api/simulation-runs/:rid
WS     /ws/simulation-runs/:rid
POST   /api/simulation-runs/:rid/stop
GET    /api/simulation-runs/:rid/sensor-data

Billing:
GET    /api/billing/usage
POST   /api/billing/checkout
POST   /api/webhooks/stripe
```

---

## Testing Strategy

**Unit Tests (Rust)**
- Vehicle dynamics: RK4 energy conservation, Pacejka force curves
- OpenDRIVE parser: sample .xodr files, lane graph connectivity
- Traffic AI: IDM acceleration formula, MOBIL lane-change logic

**Integration Tests**
- End-to-end: upload map → create scenario → run sim → verify KPIs
- Auth flow: register → login → protected endpoint
- GPU usage tracking

**E2E Tests (Playwright)**
- Create project → upload map → create scenario → run simulation
- Verify KPIs displayed (TTC, collisions, lane departures)

**Performance Tests**
- Rendering: 30 FPS @ 1920x1080 on A100
- Dynamics: 1 kHz update < 0.5ms per step
- Traffic: 200 vehicles @ 10 Hz in < 10ms
- LiDAR: VLP-32 (57,600 rays) in < 20ms

---

## Deployment (Kubernetes)

```yaml
# k8s/api-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: drivesim-api
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: api
        image: drivesim/api:latest
        resources:
          requests:
            memory: "2Gi"
            cpu: "1000m"
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: drivesim-secrets
              key: database-url
```

```yaml
# k8s/sim-worker.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: sim-worker
spec:
  template:
    spec:
      nodeSelector:
        node.kubernetes.io/instance-type: g5.2xlarge
      containers:
      - name: worker
        image: drivesim/sim-engine:latest
        resources:
          limits:
            nvidia.com/gpu: 1
            memory: "16Gi"
```

---

## Security

**Auth**: JWT (1h expiration), bcrypt (cost 12), OAuth (Google/GitHub)
**Data**: TLS 1.3, RDS encryption at rest, S3 AES-256, presigned URLs (15min expiration)
**Validation**: Input validation (Serde), OpenDRIVE size limits (100MB), ONNX model verification
**Rate Limiting**: 100 req/min per user, 1000 req/min per IP, Stripe webhook signature verification

---

## Frontend Component Architecture

```
src/
├── pages/
│   ├── Dashboard.tsx              # Project list, GPU usage meter
│   ├── ScenarioEditor.tsx         # Scenario config + 3D scene editing
│   ├── SimulationView.tsx         # Live viewport + telemetry
│   ├── Results.tsx                # Post-sim KPI analysis
│   └── Billing.tsx                # Plan selection, usage
│
├── components/
│   ├── Viewport3D/
│   │   ├── ViewportCanvas.tsx     # WebGPU H.264 stream decode
│   │   ├── CameraControls.tsx     # Free/chase/driver cam
│   │   └── SensorOverlay.tsx      # Floating sensor panels
│   │
│   ├── ScenarioEditor/
│   │   ├── MapView.tsx            # 2D map with Three.js overlay
│   │   ├── VehiclePlacer.tsx      # Drag-drop spawn points
│   │   ├── RouteTool.tsx          # Draw routes on lanes
│   │   └── WeatherTimeline.tsx    # Keyframe weather editor
│   │
│   ├── Telemetry/
│   │   ├── TelemetryPanel.tsx     # Multi-chart dashboard
│   │   ├── VehicleState.tsx       # Speed/steering gauges
│   │   ├── LidarViewer.tsx        # Point cloud (Three.js)
│   │   └── CameraFeed.tsx         # Image + bbox overlay
│   │
│   └── KPIs/
│       ├── KPITable.tsx           # Tabular results
│       ├── TimelineChart.tsx      # TTC over time
│       └── TrajectoryPlot.tsx     # 2D overhead trajectory
│
├── stores/
│   ├── authStore.ts               # Auth state (Zustand)
│   ├── projectStore.ts            # Current project
│   ├── scenarioStore.ts           # Scenario editing
│   └── telemetryStore.ts          # Live sim telemetry
│
└── hooks/
    ├── useWebSocket.ts            # WebSocket manager
    ├── useSimProgress.ts          # Subscribe to sim progress
    └── useS3Upload.ts             # Presigned URL upload
```

---

## Monitoring (Prometheus + Grafana)

```yaml
# k8s/prometheus-rules.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-rules
data:
  drivesim.rules: |
    groups:
    - name: drivesim
      interval: 30s
      rules:
      - alert: HighGPUQueueDepth
        expr: redis_list_length{list="simulation:jobs"} > 50
        for: 5m
        annotations:
          summary: "GPU queue depth high"
          description: "{{ $value }} jobs pending"

      - alert: SimulationWorkerDown
        expr: up{job="sim-worker"} == 0
        for: 2m

      - alert: HighGPUUtilization
        expr: nvidia_gpu_duty_cycle > 95
        for: 10m
        annotations:
          summary: "GPU {{ $labels.gpu }} at {{ $value }}%"

      - alert: SimFailureRate
        expr: rate(simulation_runs_failed[5m]) / rate(simulation_runs_total[5m]) > 0.1
        for: 5m
        annotations:
          summary: "Failure rate > 10%"
```

---

## Infrastructure (Terraform)

```hcl
# terraform/main.tf
module "vpc" {
  source = "./modules/vpc"
  cidr_block = "10.0.0.0/16"
  availability_zones = ["us-west-2a", "us-west-2b", "us-west-2c"]
}

module "eks" {
  source = "./modules/eks"
  cluster_name = "drivesim-prod"
  vpc_id = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnet_ids

  node_groups = {
    api = {
      instance_types = ["c6i.2xlarge"]
      desired_size = 3
      min_size = 2
      max_size = 10
    }

    gpu_workers = {
      instance_types = ["g5.2xlarge", "g5.4xlarge"]
      desired_size = 2
      min_size = 0
      max_size = 50
      spot = true
      taints = [{
        key = "nvidia.com/gpu"
        value = "true"
        effect = "NoSchedule"
      }]
    }
  }
}

module "rds" {
  source = "./modules/rds"
  instance_class = "db.r6g.xlarge"
  allocated_storage = 500
  engine_version = "16.1"
  multi_az = true
  backup_retention_period = 7
}

module "elasticache" {
  source = "./modules/elasticache"
  node_type = "cache.r6g.large"
  num_cache_nodes = 2
  engine_version = "7.0"
}

module "s3" {
  source = "./modules/s3"
  bucket_name = "drivesim-assets-prod"
  cors_allowed_origins = ["https://app.drivesim.io"]
  lifecycle_rules = [{
    id = "archive_old_sims"
    prefix = "simulation-results/"
    transition_days = 90
    transition_storage_class = "GLACIER"
  }]
}

module "cloudfront" {
  source = "./modules/cloudfront"
  origin_domain_name = module.s3.bucket_regional_domain_name
  aliases = ["assets.drivesim.io"]
  price_class = "PriceClass_100"
}
```

---

## Additional Architecture: OpenDRIVE Parser

```rust
// src/opendrive/parser.rs

use quick_xml::de::from_str;
use serde::Deserialize;

#[derive(Debug, Deserialize)]
pub struct OpenDriveMap {
    pub road: Vec<Road>,
    pub junction: Vec<Junction>,
}

#[derive(Debug, Deserialize)]
pub struct Road {
    pub id: u32,
    pub name: String,
    pub length: f64,
    pub junction: Option<i32>,
    #[serde(rename = "planView")]
    pub plan_view: PlanView,
    pub lanes: Lanes,
    #[serde(rename = "elevationProfile")]
    pub elevation_profile: Option<ElevationProfile>,
}

#[derive(Debug, Deserialize)]
pub struct PlanView {
    pub geometry: Vec<Geometry>,
}

#[derive(Debug, Deserialize)]
pub struct Geometry {
    pub s: f64,
    pub x: f64,
    pub y: f64,
    pub hdg: f64,
    pub length: f64,
    #[serde(rename = "$value")]
    pub shape: GeometryShape,
}

#[derive(Debug, Deserialize)]
#[serde(rename_all = "camelCase")]
pub enum GeometryShape {
    Line,
    Arc { curvature: f64 },
    Spiral { curv_start: f64, curv_end: f64 },
    Poly3 { a: f64, b: f64, c: f64, d: f64 },
    ParamPoly3 { #[serde(flatten)] params: ParamPoly3Params },
}

#[derive(Debug, Deserialize)]
pub struct ParamPoly3Params {
    pub au: f64,
    pub bu: f64,
    pub cu: f64,
    pub du: f64,
    pub av: f64,
    pub bv: f64,
    pub cv: f64,
    pub dv: f64,
}

#[derive(Debug, Deserialize)]
pub struct Lanes {
    #[serde(rename = "laneSection")]
    pub lane_section: Vec<LaneSection>,
}

#[derive(Debug, Deserialize)]
pub struct LaneSection {
    pub s: f64,
    pub left: Option<LaneGroup>,
    pub center: LaneGroup,
    pub right: Option<LaneGroup>,
}

#[derive(Debug, Deserialize)]
pub struct LaneGroup {
    pub lane: Vec<Lane>,
}

#[derive(Debug, Deserialize)]
pub struct Lane {
    pub id: i32,
    #[serde(rename = "type")]
    pub lane_type: String,
    pub link: Option<LaneLink>,
    pub width: Vec<LaneWidth>,
}

#[derive(Debug, Deserialize)]
pub struct LaneLink {
    pub predecessor: Option<LaneLinkTarget>,
    pub successor: Option<LaneLinkTarget>,
}

#[derive(Debug, Deserialize)]
pub struct LaneLinkTarget {
    pub id: i32,
}

#[derive(Debug, Deserialize)]
pub struct LaneWidth {
    #[serde(rename = "sOffset")]
    pub s_offset: f64,
    pub a: f64,
    pub b: f64,
    pub c: f64,
    pub d: f64,
}

pub fn parse_opendrive(xml: &str) -> Result<OpenDriveMap, ParseError> {
    from_str(xml).map_err(|e| ParseError::XmlError(e.to_string()))
}

pub enum ParseError {
    XmlError(String),
    InvalidGeometry(String),
}

// Lane graph extraction
pub fn build_lane_graph(map: &OpenDriveMap) -> LaneGraph {
    let mut graph = LaneGraph::new();

    for road in &map.road {
        for section in &road.lanes.lane_section {
            // Process left lanes
            if let Some(left) = &section.left {
                for lane in &left.lane {
                    if lane.lane_type == "driving" {
                        let lane_id = graph.add_lane(LaneInfo {
                            road_id: road.id,
                            section_s: section.s,
                            lane_id: lane.id,
                            centerline: compute_lane_centerline(road, section, lane),
                        });

                        // Add successors
                        if let Some(link) = &lane.link {
                            if let Some(succ) = &link.successor {
                                graph.add_successor(lane_id, succ.id);
                            }
                        }
                    }
                }
            }

            // Process right lanes (similar)
            // ...
        }
    }

    graph
}

fn compute_lane_centerline(road: &Road, section: &LaneSection, lane: &Lane) -> Vec<na::Vector3<f64>> {
    let mut points = Vec::new();
    let step = 0.5;  // Sample every 0.5m

    let mut s = section.s;
    while s < section.s + road.length {
        // Compute reference line position at s
        let ref_pos = evaluate_plan_view(&road.plan_view, s);

        // Compute lateral offset for this lane
        let lateral_offset = compute_lane_offset(section, lane, s);

        // Add perpendicular offset
        let heading = evaluate_heading(&road.plan_view, s);
        let perpendicular = na::Vector3::new(-heading.sin(), heading.cos(), 0.0);
        let lane_pos = ref_pos + perpendicular * lateral_offset;

        points.push(lane_pos);
        s += step;
    }

    points
}
```

---

## Additional Architecture: Simulation Worker

```rust
// src/workers/sim_worker.rs

use std::sync::Arc;
use aws_sdk_s3::Client as S3Client;
use redis::AsyncCommands;
use sqlx::PgPool;
use tokio::sync::broadcast;

pub struct SimulationWorker {
    db: PgPool,
    redis: redis::Client,
    s3: S3Client,
    broadcaster: Arc<ProgressBroadcaster>,
}

impl SimulationWorker {
    pub async fn run(&self) -> anyhow::Result<()> {
        let mut conn = self.redis.get_async_connection().await?;
        tracing::info!("Worker started, listening for jobs");

        loop {
            let job_id: Option<String> = conn.brpop("simulation:jobs", 30.0).await?;
            let Some(job_id) = job_id else { continue };
            let job_id: Uuid = job_id.parse()?;

            if let Err(e) = self.process_job(job_id).await {
                tracing::error!("Job {job_id} failed: {e}");
                self.mark_failed(job_id, &e.to_string()).await?;
            }
        }
    }

    async fn process_job(&self, job_id: Uuid) -> anyhow::Result<()> {
        // Fetch simulation details
        let sim = sqlx::query_as!(
            SimulationRun,
            "UPDATE simulation_runs SET status = 'running', started_at = NOW()
             WHERE id = $1 RETURNING *",
            job_id
        )
        .fetch_one(&self.db)
        .await?;

        // Load scenario
        let scenario = sqlx::query_as!(
            Scenario,
            "SELECT * FROM scenarios WHERE id = $1",
            sim.scenario_id
        )
        .fetch_one(&self.db)
        .await?;

        // Load map
        let map = self.load_map_from_s3(&scenario.map_id).await?;

        // Initialize simulation engine
        let mut engine = SimulationEngine::new(
            map,
            scenario.weather_config,
            scenario.traffic_config,
        );

        // Run simulation with progress streaming
        let progress_tx = self.broadcaster.get_channel(sim.id);
        let mut time = 0.0;
        let dt = 0.01;  // 10ms timestep
        let duration = 60.0;  // 60s simulation

        while time < duration {
            engine.step(dt);
            time += dt;

            // Stream progress every 100ms
            if (time * 100.0) as u32 % 10 == 0 {
                let _ = progress_tx.send(serde_json::json!({
                    "time": time,
                    "progress_pct": (time / duration * 100.0) as f32,
                    "ego_position": engine.ego_vehicle.state.position,
                    "ego_velocity": engine.ego_vehicle.state.velocity.norm(),
                }));
            }
        }

        // Compute KPIs
        let kpis = engine.compute_kpis();

        // Save sensor data to S3
        let sensor_data = engine.export_sensor_data();
        let s3_key = format!("sensor-data/{}/{}.tar.gz", sim.scenario_id, sim.id);
        self.s3.put_object()
            .bucket("drivesim-results")
            .key(&s3_key)
            .body(sensor_data.into())
            .send()
            .await?;

        // Update database
        sqlx::query!(
            "UPDATE simulation_runs SET
             status = 'completed',
             kpi_results = $2,
             sensor_data_url = $3,
             completed_at = NOW(),
             wall_time_seconds = EXTRACT(EPOCH FROM (NOW() - started_at))
             WHERE id = $1",
            sim.id,
            serde_json::to_value(&kpis)?,
            format!("s3://drivesim-results/{}", s3_key)
        )
        .execute(&self.db)
        .await?;

        Ok(())
    }
}
```

---

## Additional Architecture: KPI Computation (Python)

```python
# python_services/kpi_service/perception.py

import numpy as np
from typing import List, Dict
from dataclasses import dataclass

@dataclass
class BoundingBox:
    x: float
    y: float
    width: float
    height: float
    class_id: int
    confidence: float

def compute_iou(box1: BoundingBox, box2: BoundingBox) -> float:
    """Compute Intersection over Union between two boxes."""
    x1_min, x1_max = box1.x, box1.x + box1.width
    y1_min, y1_max = box1.y, box1.y + box1.height
    x2_min, x2_max = box2.x, box2.x + box2.width
    y2_min, y2_max = box2.y, box2.y + box2.height

    inter_xmin = max(x1_min, x2_min)
    inter_ymin = max(y1_min, y2_min)
    inter_xmax = min(x1_max, x2_max)
    inter_ymax = min(y1_max, y2_max)

    if inter_xmax <= inter_xmin or inter_ymax <= inter_ymin:
        return 0.0

    inter_area = (inter_xmax - inter_xmin) * (inter_ymax - inter_ymin)
    union_area = (box1.width * box1.height) + (box2.width * box2.height) - inter_area
    return inter_area / union_area

def compute_map(
    predictions: List[List[BoundingBox]],
    ground_truths: List[List[BoundingBox]],
    iou_threshold: float = 0.5,
    num_classes: int = 10
) -> Dict[str, float]:
    """
    Compute mean Average Precision (mAP) for object detection.

    Args:
        predictions: List of predicted boxes per image
        ground_truths: List of ground truth boxes per image
        iou_threshold: IoU threshold for matching
        num_classes: Number of object classes

    Returns:
        Dictionary with mAP and per-class AP
    """
    aps = []

    for class_id in range(num_classes):
        # Collect all predictions and GTs for this class
        all_preds = []
        all_gts = []

        for img_preds, img_gts in zip(predictions, ground_truths):
            class_preds = [p for p in img_preds if p.class_id == class_id]
            class_gts = [g for g in img_gts if g.class_id == class_id]

            all_preds.extend([(p, len(all_gts) + i) for i, p in enumerate(class_preds)])
            all_gts.extend(class_gts)

        if len(all_gts) == 0:
            continue

        # Sort predictions by confidence
        all_preds.sort(key=lambda x: x[0].confidence, reverse=True)

        # Match predictions to ground truths
        tp = np.zeros(len(all_preds))
        fp = np.zeros(len(all_preds))
        matched_gts = set()

        for i, (pred, img_idx) in enumerate(all_preds):
            best_iou = 0.0
            best_gt_idx = -1

            for j, gt in enumerate(all_gts):
                if j in matched_gts:
                    continue
                iou = compute_iou(pred, gt)
                if iou > best_iou:
                    best_iou = iou
                    best_gt_idx = j

            if best_iou >= iou_threshold:
                tp[i] = 1
                matched_gts.add(best_gt_idx)
            else:
                fp[i] = 1

        # Compute precision-recall curve
        tp_cumsum = np.cumsum(tp)
        fp_cumsum = np.cumsum(fp)
        recalls = tp_cumsum / len(all_gts)
        precisions = tp_cumsum / (tp_cumsum + fp_cumsum)

        # Compute AP (area under PR curve)
        ap = 0.0
        for t in np.arange(0.0, 1.1, 0.1):
            if np.sum(recalls >= t) == 0:
                p = 0
            else:
                p = np.max(precisions[recalls >= t])
            ap += p / 11.0

        aps.append(ap)

    return {
        "mAP": np.mean(aps) if aps else 0.0,
        "per_class_AP": {f"class_{i}": ap for i, ap in enumerate(aps)}
    }

def compute_ttc(
    ego_position: np.ndarray,
    ego_velocity: np.ndarray,
    other_position: np.ndarray,
    other_velocity: np.ndarray
) -> float:
    """
    Compute Time-to-Collision between ego and another vehicle.
    Returns inf if no collision predicted.
    """
    relative_pos = other_position - ego_position
    relative_vel = other_velocity - ego_velocity

    # Distance
    distance = np.linalg.norm(relative_pos)

    # Closing rate (negative if approaching)
    closing_rate = -np.dot(relative_pos, relative_vel) / distance

    if closing_rate <= 0:
        return float('inf')  # Not approaching

    return distance / closing_rate
```

---

## Risk Mitigations

| Risk | Mitigation |
|------|------------|
| Vulkan RT rendering quality insufficient vs NVIDIA Omniverse | Invest in shader quality (2-4 bounce path tracing, advanced material models). Benchmark FID against real data. Implement domain randomization to reduce reliance on photorealism alone. |
| GPU compute costs erode margins on long simulations | Use spot instances (60-70% savings). Implement LOD for distant objects. Adaptive resolution scaling (reduce resolution when viewport not active). Cache static scene rendering across runs. |
| OpenDRIVE maps from customers have errors/non-standard extensions | Build robust error handling in parser. Implement validation tool that flags issues before simulation. Offer map cleanup service (manual or automated via Python scripts). |
| Sim-to-real gap causes perception models to fail in real world | Quantify gap via benchmarks (FID, Chamfer Distance). Partner with AV companies for validation on real sensor data. Implement sensor calibration tool to match real hardware. |
| Scaling to 500+ concurrent vehicles causes performance degradation | Optimize spatial hash grid (GPU-accelerated). Implement behavior LOD (distant vehicles use simpler models). Profile and optimize hot paths in traffic update loop. |

---

## Dependencies

| Component | License | Purpose |
|-----------|---------|---------|
| Vulkan SDK | Apache 2.0 | GPU rendering and compute |
| ONNX Runtime | MIT | ML model inference |
| nalgebra | Apache 2.0 | Linear algebra for dynamics |
| quick-xml | MIT | OpenDRIVE XML parsing |
| Rayon | Apache 2.0 | Parallel traffic AI updates |
| FFmpeg / libav | LGPL | Video encoding for viewport stream |
| Three.js | MIT | Frontend 3D scene editing |
| React Flow | MIT | Visual OpenSCENARIO editor |

---

## Launch Checklist

### Week 10 (Pre-Launch)
- [ ] Complete Phase 8 tasks (billing, deployment, load testing)
- [ ] Security audit: OWASP ZAP scan, fix critical/high issues
- [ ] Performance validation: run all 5 benchmarks, verify targets met
- [ ] Documentation: user guide, API reference (OpenAPI), example scenarios published
- [ ] Legal: privacy policy, terms of service, DPA reviewed
- [ ] Monitoring: all Prometheus alerts configured, on-call rotation established
- [ ] Disaster recovery: RDS automated backups verified, S3 cross-region replication enabled

### Week 11 (Soft Launch)
- [ ] Deploy to production with 10 GPU nodes (g5.2xlarge spot instances)
- [ ] Invite 20 beta testers: 10 AV startups, 5 university labs, 5 Tier 1 suppliers
- [ ] Provide 3 months free Team tier access to all beta users
- [ ] Daily check-ins with beta users via Slack channel
- [ ] Bug triage: <24hr turnaround for critical, <48hr for high-priority
- [ ] Collect qualitative feedback via 30min user interviews (5-10 users)
- [ ] Monitor key metrics: simulation success rate (target >95%), avg frame time, GPU utilization

### Week 12 (Public Launch)
- [ ] Scale GPU cluster to 50 nodes with autoscaling enabled (max 100 nodes)
- [ ] Product Hunt launch: prepare screenshots, 90s demo video, tagline
- [ ] Blog post: "DriveSim — Open-Access AV Simulation Platform" with technical details
- [ ] Social media: Twitter/X thread (10+ tweets with gifs), LinkedIn post, HN Show HN
- [ ] Reach out to AV community forums: r/SelfDrivingCars, Comma.ai Discord, Autoware Slack
- [ ] SEO: submit sitemap, optimize meta tags, target keywords ("autonomous driving simulator", "AV simulation platform")
- [ ] Enable public registration with email verification
- [ ] Customer support: Intercom chatbot + human agent, target <4hr first response time
- [ ] Monitor: signup conversion rate, GPU hours consumed, error rates, customer feedback

### Months 2-3 (Growth)
- [ ] Publish 2-3 case studies with beta users showing 10x faster regression testing
- [ ] Webinar: "Getting Started with DriveSim" (live demo + Q&A), record and publish on YouTube
- [ ] Integration partnerships: Autoware (document ROS2 bridge), Apollo (gRPC interface guide)
- [ ] Academic outreach: contact 50 university AV labs with 50% educational discount offer
- [ ] Content marketing: "Autonomous Driving Simulation Best Practices" blog series (6 posts)
- [ ] Feature releases: v1.1 (batch simulation), v1.2 (radar sensor + advanced dynamics)
- [ ] Iterate on pricing: analyze unit economics (GPU cost per sim-hour), customer feedback, conversion funnel
- [ ] Track success metrics: MRR growth, paying customers, GPU hours/month, NPS score
