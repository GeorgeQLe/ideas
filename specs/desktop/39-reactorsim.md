# ReactorSim Desktop — 5-Framework Comparison

## Desktop Rationale

- **Safety-Critical Air-Gapped Deployment:** Nuclear facilities and regulatory bodies mandate air-gapped computing environments with zero cloud connectivity. A desktop application ensures ReactorSim can operate in classified or restricted networks where reactor design data must never leave the premises, satisfying NRC, IAEA, and DOE cybersecurity requirements.
- **GPU-Accelerated Neutron Transport & Reaction Kinetics:** Monte Carlo neutron transport simulations (MCNP-class) and deterministic Sn solvers demand sustained GPU compute throughput. Desktop access to local NVIDIA CUDA cores or AMD ROCm pipelines via wgpu/Vulkan eliminates cloud round-trip latency and enables interactive parameter sweeps across millions of neutron histories per second.
- **3D Reactor Vessel Visualization & Thermal-Hydraulic Modeling:** Engineers require real-time 3D cross-sectional views of reactor cores, fuel assemblies, control rod positions, and coolant flow fields. Desktop rendering through native GPU pipelines (Metal, Vulkan, DirectX) enables smooth rotation, slicing, and animation of complex thermal-hydraulic CFD meshes with millions of cells.
- **Large Simulation Dataset Management:** A single reactor simulation campaign can generate 50-500 GB of output data including neutron flux maps, temperature distributions, burnup profiles, and transient analyses. Local SSD storage with memory-mapped file access provides the I/O bandwidth needed for interactive post-processing without network bottlenecks.
- **Regulatory Compliance & Audit Trail Integrity:** Nuclear regulatory submissions (10 CFR 50.59 analyses, safety analysis reports) require cryptographically signed, tamper-evident simulation records. A desktop application can maintain a local chain-of-custody database with hardware-bound encryption keys, ensuring audit trail integrity without reliance on third-party cloud infrastructure.

---

## Desktop-Specific Features

- GPU-accelerated Monte Carlo neutron transport engine using wgpu compute shaders, supporting continuous-energy and multi-group cross-section libraries with billions of particle histories
- Real-time 3D reactor core visualization with interactive cross-sectioning of fuel assemblies, control rods, moderator regions, and pressure vessel internals rendered at 60fps via native GPU pipelines
- Thermal-hydraulic CFD solver integration for coolant flow modeling (single-phase and two-phase) with conjugate heat transfer across fuel pins, cladding, and coolant channels
- Local ENDF/B-VIII.0 and JEFF-3.3 nuclear cross-section library management with fast binary lookup tables stored on SSD for sub-millisecond isotope data retrieval
- Burnup and depletion calculation engine tracking 2000+ nuclides through irradiation cycles with Bateman equation solvers optimized for SIMD/GPU parallel execution
- Reactor kinetics simulator for transient and accident scenario analysis (LOCA, RIA, ATWS) with point-kinetics and spatial-kinetics solvers running at sub-millisecond timesteps
- Air-gapped operation mode with complete offline functionality, local license validation via hardware dongles (USB/TPM), and no network requirements
- Regulatory document generator producing NRC-formatted safety analysis reports, FSAR chapters, and 10 CFR 50.59 screening worksheets from simulation results
- Hardware-in-the-loop interface for connecting to reactor instrumentation simulators, DCS/SCADA training systems, and control room mockups via serial/Modbus/OPC-UA protocols
- Cryptographically signed simulation audit trail with SHA-256 hashed result chains, analyst authentication, and V&V (Verification and Validation) workflow tracking

---

## Shared Packages

### core (Rust)

- Monte Carlo neutron transport engine with continuous-energy cross-section sampling, geometry ray-tracing through CSG (Constructive Solid Geometry) models, and variance reduction techniques (weight windows, implicit capture, DXTRAN spheres)
- Deterministic neutron transport solver using discrete ordinates (Sn) method with diamond-difference and TWOTRAN spatial discretization on structured and unstructured meshes
- Thermal-hydraulic solver implementing drift-flux two-phase flow model, fuel rod heat conduction with gap conductance, and RELAP-style system-level thermohydraulics
- Burnup/depletion engine solving the Bateman equations via CRAM (Chebyshev Rational Approximation Method) for 2000+ nuclide transmutation chains with decay heat calculation
- Reactor kinetics module with six-group delayed neutron point kinetics, Improved Quasi-Static (IQS) spatial kinetics, and reactivity feedback models (Doppler, moderator density, void)
- Nuclear data interface for parsing ENDF-6 formatted cross-section libraries, generating probability tables for unresolved resonance region, and Doppler-broadening via kernel methods

### api-client (Rust)

- Secure file transfer client for exchanging simulation inputs/outputs with review servers in controlled network environments using SFTP with FIPS 140-2 validated encryption
- Cross-section library update manager for downloading and verifying new ENDF/B releases from authorized distribution servers with GPG signature validation
- License validation client supporting both online activation and offline hardware-dongle (USB HID) verification with grace period management
- Telemetry and crash reporting module (opt-in only) that packages anonymized performance metrics and stack traces for developer analysis without exposing simulation data
- Multi-user collaboration sync for shared reactor models in team environments using conflict-free replicated data types (CRDTs) over local network

### data (SQLite)

- Simulation campaign database storing reactor model definitions (geometry, materials, boundary conditions), solver configurations, and parameter sweep metadata with full version history
- Results warehouse with indexed storage of neutron flux maps, power distributions, temperature fields, and burnup vectors, supporting fast spatial and temporal queries across thousands of simulation cases
- Regulatory audit trail table maintaining cryptographically chained records of every simulation run, analyst ID, input checksums, and result hashes for NQA-1 compliance
- Nuclear data cache storing pre-processed, Doppler-broadened cross-section tables keyed by isotope, temperature, and energy group structure for rapid solver initialization

---

## Performance Considerations

### GPU Compute Requirements
- Monte Carlo neutron transport requires sustained GPU compute for billions of particle histories; each neutron history involves 50-200 collision events with cross-section lookups, geometry ray-tracing, and tally accumulation — ideally mapped to GPU thread blocks with shared memory for local tally reduction
- Thermal-hydraulic CFD meshes with 1-10 million cells require GPU-resident sparse matrix solvers (conjugate gradient, GMRES) with preconditioners that fit within GPU VRAM (8-24 GB on workstation GPUs)
- Eigenvalue iteration (keff convergence) demands tight CPU-GPU synchronization per fission generation cycle, with fission source redistribution between GPU compute passes
- Burnup/depletion calculations are embarrassingly parallel across fuel regions but require dense matrix exponentials (CRAM) that benefit from batched GPU BLAS operations

### Memory Architecture
- Cross-section lookup tables for continuous-energy transport consume 2-8 GB RAM depending on isotope count and temperature points; must remain resident for random-access during particle tracking
- 3D flux/power tally arrays for a full-core model with 50,000+ mesh cells and 100+ energy groups require 500 MB - 2 GB of contiguous memory with atomic update support
- Geometry acceleration structures (BVH/octree for CSG ray-tracing) consume 200-500 MB for complex reactor models with thousands of surfaces
- Temperature and density field arrays for thermal-hydraulic feedback coupling add 100-300 MB for full-core subchannel models

---

## Framework Implementations

### Electron

**Architecture:** Chromium renderer for UI with Node.js backend spawning Rust simulation engine as child process. 3D reactor visualization via Three.js/WebGL in the renderer process. Compute shaders through WebGPU (Chrome 113+). IPC bridge between renderer and Rust FFI via Node native addons (N-API).

**Tech stack:** React/TypeScript UI, Three.js for 3D reactor geometry, WebGPU for compute (neutron transport kernels), node-ffi-napi for Rust core bindings, better-sqlite3 for local database, Electron Forge for packaging.

**Native module needs:** Rust simulation core via N-API addon, OpenSSL for FIPS crypto, USB HID library for hardware dongle license validation, serial port access for HIL interfaces, memory-mapped file I/O for large datasets.

**Bundle size:** ~280-350 MB

**Memory:** ~400-800 MB (base Chromium ~200 MB + simulation data resident in memory)

**Pros:**
- Mature WebGPU support in Chromium enables compute shader deployment without native GPU SDK dependencies
- Three.js ecosystem provides extensive 3D visualization primitives for reactor geometry rendering
- Large developer pool familiar with web technologies reduces hiring friction
- Cross-platform consistency ensures identical UI across Windows, macOS, and Linux workstations

**Cons:**
- Chromium memory overhead is wasteful in memory-constrained air-gapped workstations where every MB matters for simulation data
- WebGPU compute performance lags 20-40% behind native Vulkan/Metal for sustained neutron transport workloads
- Serialization overhead between JS renderer and Rust core adds latency to interactive parameter sweeps
- Security surface area of embedded Chromium is a concern for nuclear facility cybersecurity audits

### Tauri

**Architecture:** Native webview (WebKit on macOS, WebView2 on Windows, WebKitGTK on Linux) hosting lightweight UI. Rust backend directly embeds simulation engine in-process. GPU compute via wgpu (Vulkan/Metal/DX12 abstraction). IPC through Tauri command system with zero-copy serde.

**Tech stack:** SolidJS/TypeScript frontend for reactive UI, wgpu for GPU compute and 3D rendering, Rust core compiled directly into Tauri backend (no FFI boundary), rusqlite for database, wry/tao for windowing.

**Native module needs:** wgpu for GPU access (compiles with application), USB HID crate for dongle licensing, serialport crate for HIL, memmap2 for memory-mapped dataset access, ring crate for FIPS-compatible cryptography.

**Bundle size:** ~35-55 MB

**Memory:** ~120-250 MB (native webview ~30 MB + simulation engine + GPU buffers)

**Pros:**
- Rust-native simulation engine runs in-process with zero serialization overhead, enabling microsecond-level parameter updates
- wgpu provides near-native GPU compute performance (within 5% of raw Vulkan) for neutron transport and thermal-hydraulic kernels
- Minimal memory footprint leaves maximum RAM available for simulation datasets and cross-section tables
- Rust memory safety guarantees align with nuclear safety culture and simplify cybersecurity certification

**Cons:**
- Webview rendering capabilities lag behind Chromium for complex 3D visualization; may need custom wgpu render surface
- Smaller ecosystem for scientific visualization components compared to Three.js/WebGL
- Platform-specific webview inconsistencies require additional testing across Windows/macOS/Linux
- Steeper learning curve for teams without Rust experience

### Flutter Desktop

**Architecture:** Skia/Impeller rendering engine for UI with Dart isolates managing simulation workflow. Rust simulation core accessed via dart:ffi. GPU compute requires platform channels to native wgpu/Vulkan layer. Custom render textures for 3D reactor visualization.

**Tech stack:** Dart/Flutter for UI, Skia/Impeller for 2D rendering, platform channels to Rust core via FFI, custom texture rendering for 3D views piped from wgpu backend, sqflite for database, flutter_rust_bridge for interop.

**Native module needs:** Rust core via FFI with flutter_rust_bridge, platform-specific GPU compute layer (Vulkan/Metal), USB HID via platform channels, serial port via platform plugins, native file I/O for large datasets.

**Bundle size:** ~80-120 MB

**Memory:** ~250-450 MB (Dart VM ~80 MB + Skia renderer + simulation data)

**Pros:**
- Impeller rendering engine provides smooth 60fps UI animations for control panels and parameter dashboards
- Strong widget system enables rapid construction of complex reactor configuration interfaces
- Single codebase covers Windows, macOS, and Linux with consistent Material Design appearance
- Hot reload accelerates UI iteration during development

**Cons:**
- No native GPU compute access from Dart; all simulation kernels must route through FFI to Rust/wgpu adding architectural complexity
- 3D reactor visualization requires custom texture bridge between wgpu render target and Flutter texture widget, introducing frame latency
- Dart VM memory overhead competes with simulation data for available RAM
- Limited ecosystem for scientific/engineering visualization compared to web or native platforms

### Swift/SwiftUI (macOS)

**Architecture:** Native SwiftUI interface with Metal compute shaders for neutron transport and SceneKit/RealityKit for 3D reactor visualization. Rust simulation core linked as static library via C FFI. Core Data or direct SQLite for persistence. Full Metal Performance Shaders access for GPU-accelerated linear algebra.

**Tech stack:** SwiftUI for UI, Metal/MPS for GPU compute, SceneKit for 3D visualization, Rust core via C-compatible static library, GRDB.swift or direct SQLite for database, Combine for reactive data flow.

**Native module needs:** Metal GPU access (native), Rust static library via C FFI bridge, IOKit for USB dongle and serial HIL interfaces, Security.framework for cryptographic audit trails, Accelerate.framework for CPU-vectorized numerics.

**Bundle size:** ~25-40 MB

**Memory:** ~80-180 MB (native SwiftUI + Metal buffers + simulation data)

**Pros:**
- Direct Metal compute access provides maximum GPU utilization on Apple Silicon with unified memory architecture eliminating CPU-GPU data transfers
- SceneKit/RealityKit offer production-quality 3D rendering for reactor vessel visualization with physically-based materials
- Minimal memory footprint and instant startup align with air-gapped workstation constraints
- Apple Silicon neural engine access enables ML-accelerated surrogate models for real-time reactor parameter prediction

**Cons:**
- macOS-only eliminates deployment to Windows/Linux workstations prevalent in nuclear engineering (deal-breaker for most facilities)
- Metal compute shaders are not portable; parallel CUDA/Vulkan implementations needed for cross-platform simulation kernels
- Smaller talent pool with combined Swift + nuclear engineering domain expertise
- No Linux support rules out deployment on HPC cluster head nodes commonly used in reactor analysis groups

### Kotlin Multiplatform

**Architecture:** Compose Multiplatform UI with Kotlin/JVM backend on desktop. Rust simulation core accessed via JNI. GPU compute through LWJGL/Vulkan bindings or platform-specific JNI bridges to wgpu. 3D visualization via LWJGL OpenGL/Vulkan or JavaFX 3D.

**Tech stack:** Compose Multiplatform for UI, LWJGL for GPU access and 3D rendering, Rust core via JNI bridge, SQLDelight for database, Kotlin coroutines for async simulation management, kotlinx.serialization for data interchange.

**Native module needs:** Rust core via JNI, LWJGL for Vulkan/OpenGL GPU access, JNI bridge for USB HID dongle licensing, serial port access via JNI/jSerialComm, NIO memory-mapped files for large dataset access.

**Bundle size:** ~90-140 MB (includes JVM runtime)

**Memory:** ~300-500 MB (JVM heap ~150 MB + native simulation buffers + GPU memory)

**Pros:**
- JVM ecosystem provides mature libraries for scientific computing (Apache Commons Math, EJML) that can supplement Rust core for pre/post-processing
- Compose Multiplatform covers Windows, macOS, and Linux from single Kotlin codebase
- Kotlin coroutines provide clean async patterns for managing long-running simulation jobs with progress reporting
- Strong typing and null safety in Kotlin reduce bugs in simulation workflow orchestration

**Cons:**
- JVM memory overhead and garbage collection pauses are problematic during time-sensitive transient simulations
- JNI bridge to Rust core adds complexity and potential memory management issues at the boundary
- GPU compute through LWJGL/Vulkan requires significant boilerplate compared to wgpu's ergonomic API
- JVM startup time (2-4 seconds) and warm-up period degrade perceived responsiveness in air-gapped environments

---

## Comparison Matrix

| Dimension | Electron | Tauri | Flutter | Swift | Kotlin MP |
|-----------|----------|-------|---------|-------|-----------|
| Dev effort (weeks) | 14 | 12 | 16 | 10 | 18 |
| Bundle size | 320 MB | 45 MB | 100 MB | 30 MB | 120 MB |
| Memory usage | 600 MB | 180 MB | 350 MB | 130 MB | 400 MB |
| Startup time | 3.2s | 0.8s | 2.1s | 0.4s | 3.8s |
| Native feel | 5/10 | 7/10 | 6/10 | 10/10 | 6/10 |
| Offline capability | 8/10 | 10/10 | 8/10 | 10/10 | 8/10 |
| GPU/compute access | 6/10 | 9/10 | 5/10 | 10/10 | 5/10 |
| Cross-platform | 9/10 | 9/10 | 8/10 | 2/10 | 8/10 |
| Best fit score | 6/10 | 9/10 | 5/10 | 7/10 | 5/10 |

---

## Recommended Framework

**Tauri** is the best fit for ReactorSim Desktop.

**Rationale:** ReactorSim demands a framework where Rust is a first-class citizen, not an afterthought bolted on via FFI. Tauri's architecture compiles the Rust simulation engine directly into the application binary, eliminating serialization boundaries and enabling zero-copy data flow between the neutron transport solver and the UI layer. The wgpu integration provides near-native GPU compute performance across Vulkan, Metal, and DX12 backends, which is critical for interactive Monte Carlo simulations. Tauri's minimal memory footprint (180 MB vs. Electron's 600 MB) leaves maximum RAM available for cross-section tables and simulation datasets, and the tiny 45 MB bundle deploys easily to locked-down air-gapped workstations with restrictive software installation policies.

**Runner-up:** Swift/SwiftUI would be the ideal choice if ReactorSim were macOS-only, offering unmatched Metal compute performance and Apple Silicon unified memory access. However, the nuclear engineering industry's reliance on Windows and Linux workstations makes macOS-exclusivity a disqualifying constraint for most deployment scenarios.

---

## Monetization (Desktop)

- **Node-Locked Perpetual License ($15,000-50,000/seat):** Hardware-bound license tied to workstation MAC address or TPM module, validated offline via USB dongle. Standard pricing model expected by nuclear utilities and engineering firms accustomed to ANSYS/SCALE/MCNP licensing structures.
- **Annual Maintenance & Support ($3,000-10,000/year):** Includes cross-section library updates (ENDF/B releases), solver patches, regulatory template updates, and priority technical support with guaranteed 4-hour response for simulation issues.
- **Module-Based Tier Pricing:** Base license includes steady-state neutronics. Additional paid modules for transient kinetics ($8,000), thermal-hydraulics coupling ($12,000), burnup/depletion ($6,000), and shielding/dose calculation ($10,000). Allows utilities to purchase only the capabilities they need.
- **Enterprise Floating License Server ($200,000+/year):** Centralized license server for large engineering organizations with 20+ analysts, supporting concurrent usage tracking, department-level billing, and IT-managed deployment across air-gapped networks.
- **Training & Certification Program ($2,000-5,000/person):** Instructor-led or self-paced training courses on reactor simulation methodology, V&V practices, and ReactorSim-specific workflows. Certification validates analyst competency for regulatory submissions.

---

## MVP Scope (Desktop)

**Core MVP (8 weeks):**

| Week | Milestone |
|------|-----------|
| 1-2 | Rust core scaffolding: CSG geometry engine for simple reactor models (cylinder, hexagonal lattice), multi-group neutron diffusion solver (2-group), ENDF/B cross-section reader for key isotopes (U-235, U-238, Pu-239, H-1, O-16). Tauri project setup with wgpu initialization. |
| 3-4 | GPU compute pipeline: Implement 2-group diffusion solver as wgpu compute shader with eigenvalue (keff) iteration. 2D reactor cross-section visualization with neutron flux heatmap overlay. Basic reactor model editor UI (material assignment, geometry parameters). |
| 5-6 | 3D reactor visualization: Implement CSG geometry renderer using wgpu graphics pipeline with interactive rotation, zoom, and cross-sectioning planes. Power distribution display with color-mapped fuel assembly powers. Simulation results database with SQLite storage and query interface. |
| 7 | Simulation workflow: Parameter sweep engine for enrichment/control rod studies, batch job queue with progress tracking, results comparison view (side-by-side flux maps). Export to VTK format for ParaView interoperability. Offline license validation via USB dongle prototype. |
| 8 | Integration testing, air-gapped deployment packaging (offline installer with bundled cross-section libraries), V&V test suite against published IAEA benchmark problems (2D IAEA PWR benchmark), documentation of simulation methodology for regulatory review. |

**Post-MVP Roadmap:**

| Phase | Features | Timeline |
|-------|----------|----------|
| Phase 1 | Monte Carlo neutron transport engine replacing diffusion solver for high-fidelity pin-resolved simulations with continuous-energy cross sections | Weeks 9-14 |
| Phase 2 | Thermal-hydraulic coupling with subchannel analysis (COBRA-style) for fuel temperature and moderator density feedback during eigenvalue iterations | Weeks 15-20 |
| Phase 3 | Burnup/depletion tracking across multi-cycle reactor operation with shuffling pattern optimization and discharge burnup prediction | Weeks 21-26 |
| Phase 4 | Transient kinetics solver for accident scenario analysis (rod ejection, LOCA, station blackout) with point-kinetics and IQS spatial methods | Weeks 27-32 |
| Phase 5 | Multi-physics coupling interface for external CFD codes (OpenFOAM, STAR-CCM+) via file-based or in-memory coupling protocols | Weeks 33-36 |
| Phase 6 | Regulatory report generator producing NRC-formatted FSAR chapter drafts with auto-populated tables, figures, and cross-references | Weeks 37-40 |
| Phase 7 | Hardware-in-the-loop interface for control room simulator integration via Modbus/OPC-UA | Weeks 41-44 |
| Phase 8 | Collaborative model review workflow with digital signatures, approval chains, and change-tracking for multi-analyst teams | Weeks 45-48 |
