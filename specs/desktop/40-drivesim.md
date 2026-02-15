# DriveSim Desktop — 5-Framework Comparison

## Desktop Rationale

- **GPU-Accelerated Real-Time 3D Driving Simulation:** Autonomous vehicle development requires rendering photorealistic driving environments at 60+ fps while simultaneously running physics engines, traffic AI, and weather simulation. Desktop access to dedicated NVIDIA RTX or AMD RDNA GPUs through native Vulkan/Metal pipelines delivers the sustained 10+ TFLOPS compute needed for real-time ray-traced sensor simulation that cloud-streamed solutions cannot match in latency.
- **Low-Latency Sensor Simulation (LiDAR, Camera, Radar):** Simulating a 128-beam LiDAR at 20 Hz generates 2.6 million points per second requiring GPU ray-casting against scene geometry. Camera simulation demands real-time rendering of multiple virtual cameras at 30-60 fps with physically accurate lens models. Radar simulation requires electromagnetic wave propagation modeling. All three sensor modalities must run concurrently with sub-millisecond synchronization — achievable only with direct GPU compute access on local hardware.
- **Local ML Model Testing & Validation:** AV perception and planning models (typically 100 MB - 5 GB each) must be tested against thousands of scenarios without uploading proprietary neural network weights to cloud infrastructure. Desktop deployment enables ONNX Runtime or TensorRT inference directly on local GPUs, keeping trade-secret models secure while providing deterministic, reproducible test results not subject to cloud hardware variability.
- **Hardware-in-the-Loop (HIL) Integration:** Desktop applications can interface directly with physical AV compute platforms (NVIDIA DRIVE, Qualcomm Snapdragon Ride, Mobileye EyeQ) via Ethernet, CAN bus, and automotive Ethernet (100BASE-T1) connections. This HIL capability is impossible with cloud-hosted simulation and is essential for ECU validation, sensor fusion testing, and fail-safe behavior verification.
- **Large Scenario Dataset Management:** A comprehensive AV test suite contains 50,000+ scenarios comprising HD maps, traffic patterns, weather conditions, and edge cases, totaling 200 GB - 2 TB of data. Local NVMe storage provides the 3-7 GB/s sequential read bandwidth needed to stream scenario assets into the GPU during batch simulation runs, far exceeding cloud storage IOPS limitations.

---

## Desktop-Specific Features

- GPU-accelerated ray-tracing engine for photorealistic camera sensor simulation with physically-based rendering (PBR), dynamic lighting, weather effects (rain, fog, snow, glare), and real-time global illumination via wgpu or Metal ray-tracing pipelines
- LiDAR sensor simulator using GPU compute shader ray-casting against scene BVH (Bounding Volume Hierarchy) structures, supporting configurable beam patterns (Velodyne, Ouster, Hesai, Luminar), atmospheric attenuation, and multi-return point cloud generation
- Radar sensor simulation with electromagnetic wave propagation modeling, Doppler velocity estimation, multi-path reflection, and radar cross-section (RCS) computation for vehicles, pedestrians, and infrastructure
- Real-time vehicle dynamics engine implementing bicycle model, multi-body dynamics, and tire force models (Pacejka Magic Formula) with configurable vehicle parameters (mass, wheelbase, CG height, suspension geometry)
- Traffic simulation engine with intelligent agent behaviors (IDM car-following, MOBIL lane-changing), intersection management, traffic signal logic, and pedestrian/cyclist behavior models for dense urban scenarios
- Local ML model inference runtime supporting ONNX, TensorRT, CoreML, and PyTorch formats for testing perception, prediction, and planning stacks against simulated sensor data with deterministic reproducibility
- Hardware-in-the-loop interface supporting CAN bus (SocketCAN/PCAN), automotive Ethernet, and ROS 2 bridge for connecting to physical AV compute platforms and robotics middleware
- OpenSCENARIO 2.0 and OpenDRIVE parser for importing standardized scenario descriptions and HD map formats, ensuring interoperability with ASAM ecosystem tools
- Scenario editor with visual timeline, trigger conditions, actor placement, and parametric variation support for creating systematic test campaigns from base scenarios
- Batch simulation orchestrator managing thousands of parallel scenario executions across local GPU resources with pass/fail criteria evaluation, KPI extraction, and regression detection

---

## Shared Packages

### core (Rust)

- Vehicle dynamics solver implementing multi-body rigid-body physics with Pacejka tire model, suspension kinematics, aerodynamic drag, and drivetrain torque distribution at 1 kHz simulation frequency
- Scene graph and BVH (Bounding Volume Hierarchy) engine for managing 3D environment assets with efficient spatial queries supporting millions of triangles for ray-tracing sensor simulation
- Traffic flow simulator with IDM (Intelligent Driver Model) longitudinal control, MOBIL lateral decision-making, and social-force pedestrian dynamics supporting 500+ concurrent agents at real-time rates
- Sensor fusion ground-truth generator producing pixel-perfect semantic segmentation, 3D bounding boxes, instance IDs, optical flow, and depth maps synchronized across all virtual sensor streams
- OpenSCENARIO 2.0 parser and runtime interpreter for declarative scenario execution with stochastic parameter distributions, trigger-action chains, and scenario composition
- Metrics and KPI engine computing safety-relevant indicators (TTC, PET, deceleration rate, lane deviation, collision detection) with configurable pass/fail thresholds for automated test evaluation

### api-client (Rust)

- HD map tile server client for streaming OpenDRIVE road networks and associated 3D asset tiles from local or LAN-hosted map repositories with progressive LOD (Level of Detail) loading
- ML model registry client for fetching versioned perception/planning model artifacts from team artifact stores (MLflow, Weights & Biases) with integrity verification and local caching
- Fleet telemetry importer for ingesting real-world driving logs (rosbag2, MDF4, ADTF formats) and converting them into replayable simulation scenarios with ground-truth annotations
- CI/CD integration client for reporting simulation test results to Jenkins, GitLab CI, or GitHub Actions pipelines with structured JUnit XML output and artifact attachment
- Cloud burst client for offloading large batch simulation campaigns to cloud GPU clusters (AWS, GCP, Azure) when local compute is insufficient, with result aggregation back to desktop

### data (SQLite)

- Scenario library database indexing thousands of test scenarios with metadata (road type, weather, traffic density, edge case category), search/filter capabilities, and scenario versioning with diff tracking
- Simulation results warehouse storing per-scenario outcomes including KPI time series, collision reports, sensor data snapshots, and ML model inference latencies with columnar indexing for fast aggregate queries
- Vehicle and sensor configuration database maintaining calibrated sensor mounting positions, intrinsic/extrinsic parameters, vehicle dynamics coefficients, and ML model version associations
- Map asset cache tracking downloaded HD map tiles, 3D building/vegetation models, texture atlases, and material libraries with LRU eviction policies for managing terabyte-scale local storage

---

## Performance Considerations

### GPU Compute Requirements
- LiDAR simulation with 128 beams at 20 Hz requires 2.56 million ray-cast operations per second against a scene BVH containing 10-50 million triangles; each ray must compute intersection distance, surface normal, reflectivity, and atmospheric attenuation — mapping naturally to GPU compute shader workgroups with shared BVH traversal stacks
- Camera sensor rendering at 1920x1200 resolution across 6-8 virtual cameras at 30 fps demands 1.4-1.8 billion pixel shader invocations per second with PBR materials, dynamic shadows, and post-processing (bloom, motion blur, lens distortion)
- Radar simulation requires frequency-domain electromagnetic propagation solving with FFT-based range-Doppler processing on GPU, computing radar cross-sections for 100+ scene objects per sweep
- Traffic AI for 500+ agents running IDM/MOBIL behavioral models at 100 Hz consumes 50,000 agent updates per second, each requiring nearest-neighbor spatial queries against the scene — efficiently parallelized on GPU compute

### Memory Architecture
- HD map tile cache for a metropolitan-scale operating domain (50 km x 50 km) consumes 5-20 GB of scene geometry, textures, and road network data; active tiles must remain GPU-resident for real-time rendering
- ML perception model weights for camera (ResNet/ViT backbone, 200-800 MB), LiDAR (PointPillars/CenterPoint, 100-400 MB), and fusion networks require 500 MB - 2 GB of GPU VRAM dedicated to inference
- Point cloud accumulation buffers for multi-frame LiDAR processing store 10-30 frames of 128-beam scans (300-900 MB) for motion compensation and ground segmentation algorithms
- Vehicle dynamics state vectors, traffic agent states, and scenario event queues consume 50-200 MB of CPU RAM depending on scenario complexity and agent count

---

## Framework Implementations

### Electron

**Architecture:** Chromium renderer hosting React-based UI for scenario editing, test management, and results dashboards. 3D driving simulation rendered via WebGPU in a dedicated Chromium window. Node.js backend manages Rust simulation engine as a child process with SharedArrayBuffer for sensor data streaming. Separate GPU process for ML model inference via ONNX Runtime Node bindings.

**Tech stack:** React/TypeScript UI with deck.gl for map visualization, WebGPU for 3D rendering and compute shaders, ONNX Runtime Node for ML inference, node-ffi-napi for Rust core, better-sqlite3 for database, Electron Forge packaging.

**Native module needs:** Rust simulation core via N-API, ONNX Runtime native bindings, CAN bus interface (socketcan/PCAN driver), GPU driver access for ray-tracing, SharedArrayBuffer for zero-copy sensor data, memory-mapped file I/O for scenario assets.

**Bundle size:** ~350-450 MB

**Memory:** ~600-1200 MB (Chromium ~250 MB + simulation engine + GPU buffers + ML model weights)

**Pros:**
- WebGPU in Chromium provides cross-platform GPU compute and ray-tracing without platform-specific shader compilation
- deck.gl and MapLibre offer excellent geospatial visualization for HD map display and scenario editing on top of real-world basemaps
- Extensive npm ecosystem for data visualization (D3, Plotly) enables rich test result dashboards and KPI charts
- Familiar web development workflow accelerates UI iteration and enables web-based remote monitoring interface reuse

**Cons:**
- Chromium's multi-process architecture and IPC overhead introduce 2-5ms latency between simulation ticks and UI updates, problematic for real-time HIL synchronization
- Memory consumption of 600+ MB base leaves less headroom for ML model weights and large scenario datasets on workstations with 32 GB RAM
- WebGPU ray-tracing support is nascent compared to native Vulkan/Metal RT pipelines, limiting LiDAR simulation fidelity
- Security review of embedded Chromium is burdensome for automotive OEMs with strict supply-chain software policies (ISO/SAE 21434)

### Tauri

**Architecture:** Native webview for UI layer with Rust backend running simulation engine in-process. GPU rendering and compute via wgpu with dedicated render surfaces for 3D driving view and sensor visualizations. ML model inference through ONNX Runtime Rust bindings or tract. Zero-copy IPC between simulation engine and UI via Tauri's command system with shared memory for sensor data streams.

**Tech stack:** SolidJS/TypeScript frontend, wgpu for GPU rendering and compute (ray-tracing, LiDAR simulation), Rust core compiled in-process, tract or ort crate for ML inference, rusqlite for database, wry/tao for windowing.

**Native module needs:** wgpu with ray-tracing extensions, ONNX Runtime via ort crate, socketcan crate for CAN bus, serialport for HIL interfaces, memmap2 for scenario asset streaming, GPU shader compilation at build time.

**Bundle size:** ~50-80 MB (excluding ML model weights)

**Memory:** ~180-400 MB (native webview ~30 MB + simulation + GPU buffers + ML runtime)

**Pros:**
- Rust simulation engine runs in-process with zero serialization overhead, enabling sub-millisecond simulation loop ticks critical for 1 kHz vehicle dynamics
- wgpu ray-tracing extensions provide native-quality LiDAR and camera sensor simulation with full Vulkan/Metal RT pipeline access
- Minimal base memory footprint maximizes available RAM for loading large ML models (1-5 GB) and streaming HD map assets
- Rust's ownership model prevents data races in the multi-threaded simulation pipeline (physics, traffic, sensors, ML inference running concurrently)

**Cons:**
- Webview 3D rendering limitations require wgpu render-to-texture with custom surface compositing for the driving simulation viewport
- Smaller ecosystem for geospatial visualization compared to deck.gl/MapLibre available in Electron
- wgpu ray-tracing API is still maturing and may require fallback to compute-shader ray-casting on older GPU hardware
- Teams accustomed to Python-based AV toolchains (CARLA, LGSVL) face a learning curve transitioning to Rust-native simulation development

### Flutter Desktop

**Architecture:** Impeller rendering engine for UI with Dart isolates managing simulation orchestration. Rust simulation core and GPU compute accessed via dart:ffi through flutter_rust_bridge. 3D driving simulation rendered by wgpu in a platform texture widget. ML inference via ONNX Runtime FFI bindings from Dart.

**Tech stack:** Dart/Flutter for UI, Impeller for 2D rendering, platform texture widget for 3D wgpu render target, flutter_rust_bridge for Rust core FFI, sqflite for database, flutter_map for HD map display.

**Native module needs:** Rust core via FFI, wgpu for GPU rendering/compute (bridged through platform channels), ONNX Runtime via FFI, CAN bus via platform plugins, serial port for HIL, native file I/O for large scenario assets.

**Bundle size:** ~100-150 MB

**Memory:** ~300-550 MB (Dart VM ~80 MB + Impeller + simulation data + ML models)

**Pros:**
- Flutter's widget composition system excels at building complex scenario editor interfaces with drag-and-drop timeline, property panels, and actor configuration
- Cross-platform coverage (Windows, macOS, Linux) from single Dart codebase reduces maintenance burden for multi-OS deployment
- Hot reload enables rapid iteration on dashboard layouts and results visualization views
- Strong animation framework for smooth transitions between simulation playback, analysis, and editing modes

**Cons:**
- Double-buffered texture bridge between wgpu 3D render and Flutter texture widget adds 1-2 frames of latency to driving simulation display
- Dart VM garbage collection pauses can cause simulation frame drops during intensive scenario loading phases
- No native GPU compute access from Dart; all sensor simulation and physics must route through FFI adding architectural layers
- Limited automotive-specific ecosystem; no existing packages for CAN bus, OpenSCENARIO parsing, or rosbag reading

### Swift/SwiftUI (macOS)

**Architecture:** Native SwiftUI interface with Metal for GPU rendering and compute. SceneKit or custom Metal renderer for 3D driving simulation with ray-tracing via Metal RT. Core ML for ML model inference on Apple Silicon Neural Engine. Rust physics/traffic core linked as static library. ModelIO for 3D asset loading.

**Tech stack:** SwiftUI for UI, Metal/MetalFX for GPU rendering with ray-tracing, Core ML for perception model inference, SceneKit or custom Metal renderer for 3D scene, Rust core via C FFI, GRDB.swift for database, MapKit for geospatial views.

**Native module needs:** Metal GPU (native), Rust static library via C FFI, IOKit for USB CAN bus adapters, Core ML model conversion tools, ModelIO for 3D asset pipeline, AVFoundation for video recording of simulation runs.

**Bundle size:** ~30-50 MB

**Memory:** ~100-250 MB (SwiftUI + Metal + unified memory GPU buffers)

**Pros:**
- Metal ray-tracing provides best-in-class GPU performance on Apple Silicon with hardware-accelerated BVH traversal for LiDAR and camera sensor simulation
- Apple Silicon unified memory architecture eliminates CPU-GPU data transfer overhead for scene geometry and sensor output, uniquely beneficial for multi-sensor simulation
- Core ML with Neural Engine offloads ML inference from GPU, allowing simultaneous ray-tracing rendering and perception model execution without resource contention
- MetalFX upscaling enables high-quality 4K camera sensor simulation at reduced GPU cost through temporal and spatial upscaling

**Cons:**
- macOS-only deployment excludes the majority of AV development teams who work on Linux (ROS 2 ecosystem) and Windows workstations
- Core ML model format requires conversion from ONNX/PyTorch, adding friction to ML model iteration workflows
- No CAN bus or automotive Ethernet support in Apple's frameworks; requires third-party USB adapter drivers with limited macOS support
- Ecosystem mismatch: AV industry standardizes on Linux-based tools (ROS 2, CARLA, Autoware), making macOS-only deployment impractical for most teams

### Kotlin Multiplatform

**Architecture:** Compose Multiplatform UI with Kotlin/JVM backend. 3D driving simulation via LWJGL Vulkan bindings for rendering and compute. Rust simulation core accessed via JNI. ML inference through ONNX Runtime Java bindings. CAN bus via jSerialComm or JNI to SocketCAN.

**Tech stack:** Compose Multiplatform for UI, LWJGL for Vulkan rendering and compute, Rust core via JNI, ONNX Runtime Java API for ML inference, SQLDelight for database, Kotlin coroutines for async simulation management.

**Native module needs:** Rust core via JNI bridge, LWJGL for Vulkan GPU access, ONNX Runtime Java native libraries, JNI bridge for CAN bus (SocketCAN), jSerialComm for serial HIL, NIO for memory-mapped scenario assets.

**Bundle size:** ~120-180 MB (includes JVM runtime)

**Memory:** ~400-700 MB (JVM heap ~200 MB + native simulation buffers + GPU memory + ML models)

**Pros:**
- LWJGL provides mature, battle-tested Vulkan bindings with ray-tracing extension support used by production game engines (Minecraft, among others)
- ONNX Runtime Java API is well-maintained and provides direct GPU inference without additional FFI bridges
- Kotlin coroutines offer clean structured concurrency for managing parallel sensor simulation, traffic AI, and physics threads
- JVM ecosystem includes robust libraries for data analysis, charting (JFreeChart, XChart), and report generation for test result documentation

**Cons:**
- JVM garbage collection pauses (10-50ms) disrupt real-time simulation loops, particularly problematic for 1 kHz vehicle dynamics requiring sub-millisecond determinism
- JNI boundary between Kotlin orchestration and Rust physics engine adds complexity and potential for memory leaks at native/managed boundary
- JVM startup time (2-5 seconds) and JIT warm-up period delay simulation responsiveness during rapid scenario iteration
- Large memory footprint of JVM runtime (200+ MB overhead) reduces available RAM for ML model weights and HD map tile caches

---

## Comparison Matrix

| Dimension | Electron | Tauri | Flutter | Swift | Kotlin MP |
|-----------|----------|-------|---------|-------|-----------|
| Dev effort (weeks) | 16 | 14 | 18 | 12 | 20 |
| Bundle size | 400 MB | 65 MB | 125 MB | 40 MB | 150 MB |
| Memory usage | 900 MB | 280 MB | 420 MB | 170 MB | 550 MB |
| Startup time | 3.5s | 0.9s | 2.3s | 0.5s | 4.2s |
| Native feel | 5/10 | 7/10 | 6/10 | 10/10 | 5/10 |
| Offline capability | 7/10 | 9/10 | 7/10 | 9/10 | 7/10 |
| GPU/compute access | 6/10 | 9/10 | 5/10 | 10/10 | 6/10 |
| Cross-platform | 9/10 | 9/10 | 8/10 | 2/10 | 7/10 |
| Best fit score | 6/10 | 9/10 | 5/10 | 7/10 | 5/10 |

---

## Recommended Framework

**Tauri** is the best fit for DriveSim Desktop.

**Rationale:** DriveSim's core challenge is running a multi-threaded simulation pipeline — vehicle dynamics at 1 kHz, traffic AI for 500+ agents, and GPU ray-tracing for three sensor modalities — all within a single deterministic loop. Tauri's in-process Rust architecture eliminates the IPC and serialization overhead that would cripple real-time performance in Electron or Flutter. The wgpu ray-tracing pipeline delivers near-native Vulkan/Metal performance for LiDAR point cloud generation and camera rendering, while Rust's ownership model prevents the data races that inevitably emerge in concurrent sensor simulation. The 65 MB bundle and 280 MB memory footprint leave maximum resources for loading multi-gigabyte ML models and streaming HD map tiles, and cross-platform support ensures deployment to Linux workstations where the majority of AV development occurs.

**Runner-up:** Swift/SwiftUI with Metal would be the superior choice for teams developing exclusively on Apple Silicon hardware, offering unmatched ray-tracing performance through Metal RT and the unique advantage of Neural Engine offloading for ML inference. However, the AV industry's overwhelming reliance on Linux (ROS 2, Autoware, CARLA ecosystem) and the need for CAN bus/automotive Ethernet HIL integration make macOS-exclusive deployment impractical for production AV simulation workflows.

---

## Monetization (Desktop)

- **Seat-Based Annual License ($8,000-25,000/seat/year):** Tiered pricing based on sensor simulation fidelity — Starter (camera only), Professional (camera + LiDAR), Enterprise (full sensor suite + ray-tracing + HIL). Annual subscription model aligns with automotive development program budgets and ensures continuous revenue.
- **Scenario Marketplace (20-30% platform commission):** Curated marketplace for purchasing validated scenario packs (Euro NCAP test protocols, NHTSA pre-crash typology, regional ODD-specific edge cases) from third-party scenario developers and test engineering consultancies.
- **OEM Volume License ($500,000-2,000,000/year):** Enterprise-wide deployment license for Tier 1 suppliers and OEMs, including unlimited seats, priority support, custom sensor model development, and dedicated integration engineering for proprietary AV compute platforms.
- **ML Model Validation Service ($50-200/model/month):** Automated regression testing infrastructure that continuously runs customer perception/planning models against expanding scenario libraries, providing weekly safety KPI reports and regression alerts.
- **Training & Certification ($3,000-8,000/person):** ASAM-aligned certification program for simulation engineers covering OpenSCENARIO 2.0 authoring, sensor model calibration, scenario-based safety argumentation, and DriveSim-specific workflow mastery.

---

## MVP Scope (Desktop)

**Core MVP (8 weeks):**

| Week | Milestone |
|------|-----------|
| 1-2 | Rust core scaffolding: bicycle-model vehicle dynamics at 1 kHz, basic road network loader (OpenDRIVE subset: straight roads, curves, intersections), IDM traffic agent spawning with 50 concurrent vehicles. Tauri project setup with wgpu initialization and basic 3D viewport rendering a textured ground plane with road markings. |
| 3-4 | Sensor simulation foundation: GPU compute shader LiDAR simulator with configurable beam pattern (16/32/64 beams) ray-casting against scene BVH, generating point clouds at 10 Hz. Basic camera rendering of driving scene at 30 fps with vehicle models, road geometry, and sky dome. Point cloud and camera feed visualization in UI panels alongside 3D driving view. |
| 5-6 | Scenario system: OpenSCENARIO 2.0 parser for basic scenario types (cut-in, lead vehicle braking, pedestrian crossing). Scenario editor UI with visual timeline, actor placement on map, and trigger condition configuration. Simulation playback controls (play, pause, step, speed adjustment). Results database storing per-scenario KPIs (TTC, collision, lane deviation). |
| 7 | ML model integration: ONNX Runtime inference pipeline for running perception models against simulated camera and LiDAR data. Ground-truth generator producing 3D bounding boxes and semantic labels. Side-by-side visualization of model detections vs. ground truth. Basic pass/fail evaluation based on detection accuracy thresholds. |
| 8 | Integration testing, batch simulation runner executing 100+ scenarios with parallel GPU utilization, results dashboard with KPI aggregation charts and regression highlighting, cross-platform packaging and testing (Linux primary, Windows/macOS secondary), documentation of sensor model specifications and API reference. |

**Post-MVP Roadmap:**

| Phase | Features | Timeline |
|-------|----------|----------|
| Phase 1 | Metal RT and Vulkan ray-tracing pipeline for photorealistic camera simulation with global illumination, reflections, and weather effects (rain, fog, snow) | Weeks 9-14 |
| Phase 2 | Radar sensor simulation with electromagnetic RCS modeling, multi-path propagation, Doppler processing, and configurable radar antenna patterns | Weeks 15-18 |
| Phase 3 | Advanced weather system with volumetric fog, rain splash physics, wet road reflections, snow accumulation, and sun glare modeling affecting all sensor modalities | Weeks 19-22 |
| Phase 4 | Hardware-in-the-loop interface with CAN bus (SocketCAN/PCAN), automotive Ethernet (100BASE-T1), and time synchronization (PTP/gPTP) for ECU-in-the-loop testing | Weeks 23-26 |
| Phase 5 | ROS 2 bridge for integration with Autoware, Apollo, and custom ROS-based AV stacks with sensor_msgs, nav_msgs, and autoware_msgs topic publishing | Weeks 27-30 |
| Phase 6 | Scenario generation from real-world driving logs (rosbag2, MDF4 replay with parametric perturbation and adversarial actor injection) | Weeks 31-34 |
| Phase 7 | Multi-GPU support for scaling sensor simulation across 2-4 GPUs on workstation-class hardware with load-balanced sensor assignment | Weeks 35-38 |
| Phase 8 | Cloud burst capability for offloading 10,000+ scenario batch runs to cloud GPU clusters (AWS, GCP) with result aggregation and cost tracking | Weeks 39-42 |
| Phase 9 | Digital twin import from major 3D mapping providers (HERE HD Live Map, TomTom, Mobileye REM) with automatic LOD generation | Weeks 43-46 |
| Phase 10 | Safety argumentation report generator aligned with ISO 21448 (SOTIF), UL 4600, and UNECE R157 frameworks for type-approval evidence packages | Weeks 47-50 |
