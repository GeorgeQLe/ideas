# RoboSim Desktop — 5-Framework Comparison

## Desktop Rationale

- **GPU-Accelerated Physics Simulation:** Robotics simulation requires real-time rigid body dynamics, collision detection, friction modeling, and constraint solving for articulated robots with dozens of joints. GPU-accelerated physics engines (PhysX, Bullet with OpenCL) can simulate thousands of contact points per frame, enabling realistic manipulation scenarios that would be impossible to run interactively over a network connection.

- **Real-Time 3D Visualization:** Robot simulation demands high-fidelity 3D rendering with physically-based materials, shadow mapping, and depth-of-field for realistic camera simulation. Desktop GPU access enables 60+ FPS rendering of complex robot environments with ray-traced reflections for sensor simulation, smooth camera orbiting, and simultaneous rendering of multiple viewpoints (robot camera, overhead, third-person).

- **USB/Serial Connection to Physical Robots:** Desktop applications can directly access USB, serial, and Ethernet interfaces to communicate with physical robot hardware — motor controllers, sensors, cameras, and JTAG debuggers. This enables a seamless workflow from simulation to real-robot deployment where the same control code runs against both simulated and physical hardware through a hardware abstraction layer.

- **Local ROS Integration:** The Robot Operating System (ROS/ROS2) runs as a local process ecosystem communicating via shared memory, TCP, and UDP. Desktop applications can participate as native ROS2 nodes, subscribing to sensor topics, publishing control commands, and providing visualization services — all with microsecond-level latency impossible to achieve over cloud connections.

- **Low-Latency Control Loops:** Servo control loops for robotic manipulators and mobile robots require 1-10ms cycle times. Desktop applications can achieve deterministic timing through direct hardware access, high-priority thread scheduling, and kernel-bypass I/O — latency requirements that are fundamentally incompatible with cloud-based execution for real-time control.

---

## Desktop-Specific Features

- GPU-accelerated physics simulation with rigid body dynamics, soft body deformation, and fluid simulation for underwater/aerial robotics
- Real-time 3D robot visualization with URDF/SDF model loading, joint articulation, and collision mesh rendering
- Integrated code editor with Python/C++ support for writing robot control algorithms with syntax highlighting and autocompletion
- USB/serial port management for direct communication with motor controllers, sensor boards, and microcontrollers (Arduino, STM32)
- ROS2 node integration for subscribing to sensor topics, publishing control commands, and visualizing TF trees
- Camera and LiDAR sensor simulation with GPU-accelerated ray casting for generating synthetic point clouds and depth images
- Inverse kinematics solver with interactive end-effector manipulation and joint limit visualization
- Motion planning with RRT/PRM path planners and GPU-accelerated collision checking in configuration space
- Robot fleet simulation with multi-agent coordination, communication modeling, and centralized/distributed control architectures
- Hardware-in-the-loop (HIL) support with real-time scheduling for sub-millisecond control loop execution

---

## Shared Packages

### core (Rust)

- **physics-engine:** Rigid body dynamics engine with sequential impulse constraint solver, broad-phase collision detection (sweep-and-prune), narrow-phase GJK/EPA, and friction/restitution contact modeling with GPU-accelerated batch constraint solving
- **kinematics-lib:** Forward and inverse kinematics solver for serial and parallel robot mechanisms, supporting Denavit-Hartenberg parameters, URDF parsing, Jacobian computation, and numerical IK via damped least squares
- **motion-planner:** Sampling-based motion planning (RRT*, PRM, FMT*) with GPU-accelerated collision checking, trajectory optimization via CHOMP/TrajOpt, and time-optimal trajectory parameterization
- **sensor-sim:** GPU-accelerated sensor simulation using ray casting for LiDAR point clouds, pinhole camera projection for RGB/depth images, and IMU noise modeling with Allan variance parameters
- **control-lib:** Control systems library with PID controllers, model predictive control (MPC) with QP solver, state estimation (EKF/UKF/particle filter), and system identification utilities
- **ros2-bridge:** ROS2 middleware integration using the DDS protocol for topic pub/sub, service calls, and action server/client communication with configurable QoS policies

### api-client (Rust)

- Robot model repository API for downloading URDF/SDF models, meshes, and pre-configured environments
- Cloud simulation API for offloading large-scale fleet simulations or reinforcement learning training runs
- Firmware update API for deploying control code to physical robots over-the-air
- Telemetry and logging API for uploading simulation results and real-robot performance data
- Community marketplace API for sharing custom environments, robot models, and control algorithms

### data (SQLite)

- Robot model library with URDF/SDF definitions, mesh references, joint limits, and dynamic parameters (mass, inertia)
- Simulation session storage with timestamped state snapshots, sensor logs, and control inputs for replay and analysis
- Motion plan cache with pre-computed trajectories, roadmaps, and configuration space obstacles for frequently used scenarios
- Experiment database with RL training runs, hyperparameters, reward curves, and evaluation metrics

---

## Framework Implementations

### Electron

**Architecture:** Chromium renderer for UI panels (code editor, parameter tuning, log viewer) with Three.js or Babylon.js for 3D robot visualization in WebGL/WebGPU. Native Rust/C++ addon for physics engine and motion planning running in worker threads. Serial port access via serialport npm package. ROS2 communication via rclnodejs.

**Tech stack:** React + TypeScript for UI, Three.js/Babylon.js for 3D rendering, Monaco editor for code editing, Node.js serialport for hardware access, rclnodejs for ROS2 integration, N-API native modules for physics and planning engines.

**Native module needs:** Heavy — requires compiled physics engine with GPU bindings, motion planning with collision checking, serial port access, and ROS2 DDS middleware. Camera/LiDAR simulation requires GPU ray casting through native addon.

**Bundle size:** ~300-380 MB (Chromium + 3D engine + native physics libraries + ROS2 client libraries + robot model assets)

**Memory:** ~500-1000 MB (Chromium + 3D scene graph + physics state + sensor data buffers + code editor)

**Pros:**
- Three.js/Babylon.js provide mature 3D engines with physically-based rendering, shadow mapping, and post-processing
- Monaco editor (VS Code's editor) provides excellent code editing experience for robot programming
- serialport npm package is well-maintained for USB/serial hardware communication
- Large ecosystem of robotics visualization tools (ros3djs, urdf-loader)

**Cons:**
- WebGL abstraction adds 2-5ms of latency per frame — problematic for real-time control visualization
- Chromium process model adds IPC overhead between UI and physics simulation
- Memory overhead reduces headroom for large-scale multi-robot simulations
- Real-time control loop timing unreliable due to JavaScript event loop and GC pauses

### Tauri

**Architecture:** System webview for UI panels with Rust backend for physics simulation, motion planning, and hardware communication. 3D visualization via wgpu rendered to a native window with GPU-accelerated physics and sensor simulation. ROS2 integration via Rust DDS library. Serial/USB access via Rust serialport crate.

**Tech stack:** Svelte or SolidJS for frontend UI, wgpu for 3D rendering and GPU compute, rapier (Rust physics engine) for dynamics simulation, serialport-rs for hardware communication, tokio for async I/O and real-time scheduling, rclrs (Rust ROS2 client) for ROS integration.

**Native module needs:** Minimal — all compute is native Rust. rapier provides a complete physics engine. wgpu handles GPU rendering and compute shaders for sensor simulation. serialport-rs provides direct hardware access. No FFI bridges needed.

**Bundle size:** ~50-80 MB (system webview + Rust physics/rendering binaries + robot model assets)

**Memory:** ~130-300 MB (lightweight webview + physics state in Rust heap + wgpu GPU buffers + sensor data)

**Pros:**
- rapier (Rust physics engine) provides native-speed rigid body simulation with GPU-accelerated broad-phase
- Direct serial port access via Rust with deterministic timing for hardware-in-the-loop control
- wgpu compute shaders enable GPU-accelerated ray casting for LiDAR/camera simulation
- Minimal latency between physics simulation and visualization — both in Rust, no IPC boundary

**Cons:**
- wgpu 3D rendering requires building scene graph, PBR materials, and shadow mapping from lower-level primitives
- No equivalent to Monaco editor — code editing UI must use webview-based alternatives (CodeMirror)
- Rust ROS2 client (rclrs) less mature than rclcpp/rclpy with incomplete feature coverage
- Custom 3D engine development effort significant compared to using Three.js/Babylon.js

### Flutter Desktop

**Architecture:** Skia-rendered UI for panels and controls with platform channels to Rust backend for physics and planning. 3D visualization via texture bridge to native Vulkan/Metal rendering context or using flutter_3d_controller plugin. FFI calls to physics engine and motion planner.

**Tech stack:** Dart for UI logic, flutter_3d_controller or custom native plugin for 3D rendering, dart:ffi for Rust physics/planning library access, Isolates for background simulation, serial port access via platform channels.

**Native module needs:** Significant — requires native 3D rendering plugin, FFI wrappers for physics engine, motion planner, and serial port access. Platform-specific code for USB/serial communication. GPU access for sensor simulation requires native plugin per platform.

**Bundle size:** ~90-130 MB (Flutter engine + Skia + native physics libraries + 3D rendering plugin + robot models)

**Memory:** ~300-600 MB (Flutter/Skia + Dart heap + native physics state + GPU buffers + 3D scene)

**Pros:**
- Flutter's widget system excellent for building parameter tuning panels with sliders, dials, and real-time plots
- Cross-platform UI consistency for configuration panels and log viewers
- Good animation framework for visualizing robot trajectories and motion plans
- Strong list/grid performance for displaying sensor data streams

**Cons:**
- No native 3D rendering — requires complex plugin integration for robot visualization
- FFI overhead for physics simulation steps (called at 1kHz) introduces unacceptable latency
- Dart isolates cannot share memory with native physics engine, requiring data copying
- Limited ecosystem for robotics-specific UI components (joint configurators, TF tree viewers)

### Swift/SwiftUI (macOS)

**Architecture:** Native macOS application with SwiftUI for UI panels and SceneKit/RealityKit for 3D robot visualization. Metal compute shaders for physics acceleration and ray-cast sensor simulation. Serial port access via IOKit. ROS2 communication via Rust bridge library or Swift DDS implementation.

**Tech stack:** SwiftUI for UI, SceneKit or RealityKit for 3D rendering, Metal for GPU compute (physics, sensor sim), IOKit for USB/serial, Accelerate for matrix operations in kinematics/dynamics, Swift Package Manager for dependencies.

**Native module needs:** None for macOS — full native access to Metal GPU, IOKit for hardware, and SceneKit for 3D. Rust physics library linked as static library. Metal compute shaders for collision detection and sensor simulation.

**Bundle size:** ~35-60 MB (native binary + SceneKit/Metal assets + robot models + shader archives)

**Memory:** ~120-280 MB (minimal runtime + SceneKit scene graph + Metal GPU buffers + physics state)

**Pros:**
- SceneKit/RealityKit provide high-quality 3D rendering with physics, shadows, and PBR materials out of the box
- Metal compute shaders enable GPU-accelerated physics and sensor simulation with minimal overhead
- IOKit provides low-level USB/serial access with deterministic timing for hardware-in-the-loop control
- Apple Silicon Neural Engine usable for ML-based perception models running alongside simulation

**Cons:**
- macOS-only — excludes Linux users who are dominant in the robotics community (ROS is Linux-first)
- ROS2 support on macOS is less mature than on Linux, with limited community support
- SceneKit's physics engine less configurable than Bullet/PhysX for research-grade robotics simulation
- No path to deploy simulation code to Linux-based robot hardware

### Kotlin Multiplatform

**Architecture:** Compose Multiplatform for UI panels with JNI calls to native physics and planning libraries. 3D visualization via LWJGL (OpenGL/Vulkan) or jMonkeyEngine. Serial port access via jSerialComm. ROS2 communication via Java DDS bindings or JNI to rclcpp.

**Tech stack:** Kotlin + Compose Multiplatform for UI, jMonkeyEngine or LWJGL for 3D rendering, JNI for Rust physics engine, jSerialComm for hardware access, Kotlin coroutines for async simulation, kotlinx.serialization for ROS message parsing.

**Native module needs:** Moderate — JNI wrappers for physics engine, motion planner, and GPU compute libraries. jSerialComm provides serial port access. LWJGL or jMonkeyEngine for 3D rendering with GPU access. JNI bridge to ROS2 client libraries.

**Bundle size:** ~100-150 MB (JVM runtime + Compose/Skia + jMonkeyEngine + native physics libraries + robot models)

**Memory:** ~400-800 MB (JVM heap + Compose/Skia + 3D engine + native physics state + GPU buffers)

**Pros:**
- jMonkeyEngine provides a complete Java-based 3D game engine suitable for robot visualization
- jSerialComm is a mature cross-platform serial port library with good hardware support
- Kotlin coroutines enable clean structuring of concurrent simulation and control tasks
- Cross-platform deployment on Linux, macOS, and Windows matches robotics community needs

**Cons:**
- JVM memory overhead leaves less RAM for physics simulation state and sensor data buffers
- JVM GC pauses incompatible with real-time control loop requirements (<1ms timing)
- JNI overhead for physics engine calls at 1kHz simulation rate adds measurable latency
- jMonkeyEngine ecosystem smaller and less actively maintained than Three.js or Unity alternatives

---

## Comparison Matrix

| Dimension | Electron | Tauri | Flutter | Swift | Kotlin MP |
|-----------|----------|-------|---------|-------|-----------|
| Dev effort (weeks) | 18 | 22 | 26 | 18 | 24 |
| Bundle size | 340 MB | 65 MB | 110 MB | 48 MB | 130 MB |
| Memory usage | 750 MB | 220 MB | 450 MB | 200 MB | 600 MB |
| Startup time | 3.2s | 1.0s | 2.3s | 0.6s | 2.8s |
| Native feel | 5/10 | 7/10 | 5/10 | 10/10 | 6/10 |
| Offline capability | 8/10 | 9/10 | 8/10 | 9/10 | 8/10 |
| GPU/compute access | 6/10 | 9/10 | 4/10 | 10/10 | 5/10 |
| Cross-platform | 9/10 | 9/10 | 8/10 | 2/10 | 8/10 |
| Best fit score | 6/10 | 8/10 | 4/10 | 7/10 | 5/10 |

---

## Recommended Framework

**Tauri** is the best fit for RoboSim.

**Rationale:** Robotics simulation demands tight integration between physics computation, GPU rendering, sensor simulation, and hardware I/O — all with minimal latency. Tauri's Rust backend unifies physics (rapier), GPU rendering (wgpu), serial communication (serialport-rs), and async I/O (tokio) in a single process with zero IPC overhead between components. The minimal memory footprint leaves maximum resources for large-scale multi-robot simulation scenarios. Cross-platform Linux support is essential since ROS2 and most robotics development happens on Linux.

**Runner-up:** Swift/SwiftUI with SceneKit/Metal offers the best single-platform experience on macOS with SceneKit providing a complete 3D engine out of the box, but the macOS-only limitation is a critical disqualifier for a robotics tool where Linux dominance is even stronger than in other engineering fields. For a macOS-only product targeting educational robotics, Swift would be the top choice.

---

## Monetization (Desktop)

- **Freemium model** with free basic simulation (single robot, simple environments), paid tiers for advanced features: Pro ($39/month for multi-robot, custom environments, sensor simulation) and Enterprise ($149/month for fleet simulation, RL training, cloud offload)
- **Robot model marketplace** where manufacturers and community members sell/share detailed robot URDF models with validated dynamics parameters ($10-100 per model)
- **Environment asset packs** with pre-built simulation environments (warehouse, factory floor, outdoor terrain, underwater) as purchasable add-ons ($20-50 per pack)
- **Cloud reinforcement learning** compute for training robot policies at scale (thousands of parallel simulations) at $5-20/GPU-hour
- **Educational institution licenses** with curriculum integration, student management, and grading tools for university robotics courses ($2,000-10,000/year per institution)

---

## MVP Scope (Desktop)

**Core MVP (8 weeks):**

| Week | Milestone |
|------|-----------|
| 1-2 | URDF robot model loader with joint visualization, basic 3D scene rendering via wgpu (ground plane, lighting, camera controls), forward kinematics with interactive joint sliders |
| 3-4 | Rigid body physics simulation with rapier (gravity, collision, friction), basic environment objects (boxes, cylinders, planes), simulation play/pause/step controls with real-time and accelerated modes |
| 5-6 | Integrated Python code editor (CodeMirror) for writing control scripts, Python runtime embedding for executing robot control algorithms, basic API for reading sensors and commanding joints |
| 7 | Serial port communication for connecting to physical robot hardware (Arduino/STM32), hardware abstraction layer enabling same control code for simulated and real robots, basic sensor simulation (joint encoders, contact sensors) |
| 8 | Testing with 3-4 common robot models (manipulator arm, mobile base, quadruped, drone), performance optimization for real-time simulation, cross-platform packaging for macOS/Linux/Windows |

**Post-MVP:** GPU-accelerated camera/LiDAR sensor simulation with ray casting, ROS2 node integration for topic pub/sub and TF broadcasting, inverse kinematics with interactive end-effector dragging, motion planning (RRT*, PRM) with collision-free path visualization, reinforcement learning training environment with OpenAI Gym interface, multi-robot fleet simulation with communication modeling, soft body and deformable object simulation, terrain generation with heightmaps, underwater/aerial vehicle dynamics, cloud simulation offloading for large-scale training, VR headset support for immersive robot teleoperation.
