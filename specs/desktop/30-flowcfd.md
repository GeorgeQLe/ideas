# FlowCFD Desktop — 5-Framework Comparison

## Desktop Rationale

- **GPU-Accelerated CFD Solver:** Computational fluid dynamics requires solving Navier-Stokes equations across millions of mesh cells per timestep — GPU compute shaders (CUDA/Metal/Vulkan) deliver 10-50x speedup over CPU for Lattice Boltzmann and finite volume methods, demanding direct hardware access impossible in browsers. A single steady-state simulation of an airfoil at Re=1M requires evaluating flux integrals across 5M+ cells iteratively until convergence, generating hundreds of gigaflops of compute per iteration.

- **Real-Time Flow Visualization:** Rendering velocity fields, pressure contours, streamlines, and particle traces over 3D meshes at interactive framerates requires persistent GPU state, custom shader pipelines, and multi-pass rendering — native graphics APIs are non-negotiable. Visualizing turbulent structures via Q-criterion isosurfaces with semi-transparent volume rendering demands fragment shader complexity that WebGL2 cannot handle at production mesh sizes.

- **Mesh Generation and Manipulation:** Generating structured/unstructured meshes from CAD geometry (STL, STEP) involves memory-intensive algorithms (Delaunay triangulation, boundary layer insertion) that need direct memory management and multi-threaded CPU access. A production-quality mesh for an automotive aerodynamics case can contain 50M+ cells requiring 10+ GB of RAM for generation.

- **Large Simulation Datasets:** Production CFD simulations produce multi-gigabyte result files (VTK, CGNS, HDF5) with millions of timesteps — local SSD storage with memory-mapped access is essential for interactive post-processing without network bottlenecks. A transient LES simulation can generate 500+ GB of time-resolved data that must be sliced, probed, and animated locally.

- **Multi-Monitor Results Comparison:** Engineers routinely compare simulation variants side-by-side — native desktop enables synchronized viewports across multiple displays with independent camera controls and shared color scales. Comparing mesh refinement studies, turbulence model sensitivity, and boundary condition variations requires at least 4 simultaneous viewports.

---

## Desktop-Specific Features

The following features are only feasible on desktop due to GPU solver requirements, large dataset sizes, and multi-monitor workflows:

- Direct Vulkan/Metal compute shader access for GPU-accelerated Lattice Boltzmann and finite volume CFD solvers with automatic workgroup sizing
- Real-time flow visualization with streamlines, particle traces, and volume rendering of velocity/pressure/temperature fields using multi-pass GPU rendering
- Native STL/STEP/IGES CAD import with automatic surface mesh generation, boundary layer insertion, and quality-based refinement
- Memory-mapped VTK/CGNS/HDF5 file access for interactive post-processing of multi-gigabyte simulation results without full file loading
- Multi-viewport layout with synchronized or independent cameras for comparing simulation variants across multiple monitors
- Background solver execution with progress monitoring — run week-long transient simulations while continuing mesh preparation work
- Hardware-accelerated mesh generation using multi-threaded Delaunay triangulation, octree spatial indexing, and advancing front methods
- Local result animation export to MP4/WebM with configurable resolution, framerate, color mapping, and camera path interpolation
- Probe placement and time-series extraction at arbitrary mesh locations with real-time plot updates during active simulation runs
- System resource monitor showing GPU utilization, VRAM allocation, solver convergence residuals, and CFL number tracking in real-time

---

## Shared Packages

All shared packages are compiled as Rust libraries with C-compatible FFI exports, enabling consumption from any framework.

### core (Rust)

- Lattice Boltzmann Method (LBM) solver with D3Q19/D3Q27 stencils, GPU compute shader dispatch via wgpu, MRT/TRT collision operators, and Smagorinsky subgrid-scale model for turbulence
- Finite Volume Method (FVM) solver for incompressible/compressible flows with SIMPLE/PISO pressure-velocity coupling, TVD flux limiters, and implicit time stepping
- Mesh generation engine: surface meshing from STL triangulations, Delaunay tetrahedralization with quality constraints, prismatic boundary layer insertion with growth ratio control, and Laplacian/optimization-based smoothing
- Turbulence model library: k-epsilon (standard, RNG, realizable), k-omega SST, Spalart-Allmaras one-equation, and LES Smagorinsky/WALE with dynamic coefficient computation
- Post-processing pipeline: isosurface extraction (marching cubes), streamline integration (RK4 with adaptive step), vortex identification (Q-criterion, lambda2, vorticity magnitude), and surface integral computation
- Result I/O module: VTK (legacy and XML), CGNS, HDF5, and OpenFOAM format readers/writers with LZ4/ZSTD parallel decompression

### api-client (Rust)

- Cloud HPC job submission client for offloading large simulations to remote clusters with SLURM/PBS/LSF job scheduler integration
- CAD file conversion service client for STEP/IGES to tessellated geometry via cloud-hosted Open Cascade with tolerance control
- Simulation template marketplace client for downloading pre-configured case setups (pipe flow, airfoil, heat exchanger, mixing vessel)
- License server communication with floating license checkout/checkin, concurrent seat tracking, and offline grace period (30 days)
- Telemetry and crash reporting with solver convergence statistics, mesh quality distributions, and performance profiling data

### data (SQLite)

- Project database tracking simulation cases, mesh versions, solver configurations, boundary conditions, and result file locations with full provenance and branching
- Mesh quality metrics store with element aspect ratio, skewness, orthogonality, and volume ratio histograms for each mesh version with pass/fail thresholds
- Solver convergence history with residual time-series (continuity, momentum, turbulence), iteration counts, wall-clock timing per timestep, and stability diagnostics
- Material property database with fluid viscosity, density, thermal conductivity, and specific heat tables for common working fluids (air, water, oil, refrigerants) across temperature ranges

---

## Framework Implementations

Each framework is evaluated for its ability to deliver GPU-accelerated CFD solving, real-time flow visualization, and large mesh handling.

### Electron

**Architecture:** Chromium renderer for React-based simulation setup UI; Three.js with custom shaders for 3D mesh visualization; Node.js spawning Rust solver binary as child process with stdout streaming for convergence monitoring; SharedArrayBuffer for result data transfer to WebGL renderer

**Tech stack:** React + TypeScript, Three.js with custom GLSL shaders, Node.js native addons via napi-rs, WebGL2 for visualization, better-sqlite3 for project data

**Native module needs:** napi-rs bindings to Rust CFD solver, VTK.js for scientific visualization fallback, HDF5 native reader addon, node-opencascade for CAD import

**Bundle size:** ~300-380 MB

**Memory:** ~500-800 MB

**Pros:**
- VTK.js and Three.js provide mature 3D visualization with proven CFD post-processing capabilities including contour mapping and streamline rendering
- Largest pool of web developers for building complex simulation setup wizards, parameter forms, and convergence dashboards
- Cross-platform with consistent visualization across Windows/macOS/Linux engineering workstations
- ParaView Glance (web-based) can be embedded for advanced post-processing with full VTK filter pipeline

**Cons:**
- WebGL2 cannot handle meshes exceeding 5M cells without aggressive decimation, LOD, and frustum culling strategies
- Chromium memory overhead (300+ MB baseline) directly competes with solver memory requirements for large meshes
- No compute shader access from WebGL2 — solver must run out-of-process with IPC serialization overhead for result transfer
- 60fps ceiling on flow animation playback due to JavaScript main thread contention and garbage collection pauses

### Tauri

**Architecture:** Rust backend running CFD solver natively with wgpu compute shaders for GPU-accelerated LBM/FVM; custom wgpu renderer for mesh visualization with instanced rendering for particles and streamlines; WebView frontend for simulation setup forms and convergence plots; shared memory for zero-copy result transfer between solver and renderer

**Tech stack:** Rust backend with wgpu compute + rendering, Svelte frontend, sqlx for SQLite, rayon for parallel mesh generation, custom wgpu rendering pipeline with PBR materials

**Native module needs:** wgpu for Vulkan/Metal/DX12 compute and rendering, Open Cascade Rust FFI for CAD import, HDF5 Rust bindings (hdf5-rs), CGNS reader

**Bundle size:** ~50-80 MB

**Memory:** ~150-300 MB

**Pros:**
- wgpu compute shaders run solver and renderer in the same GPU context — zero-copy result visualization without buffer transfer
- Rust-native solver eliminates IPC boundary — convergence data and field results stream directly to UI without serialization
- Minimal memory footprint leaves maximum RAM for mesh storage, solver working sets, and result caching
- rayon-based parallel mesh generation fully utilizes all CPU cores without manual thread pool management

**Cons:**
- Must build custom 3D rendering pipeline — no off-the-shelf CFD visualization toolkit exists in the Rust/wgpu ecosystem
- WebView limitations for complex simulation setup forms with drag-and-drop boundary condition assignment on 3D faces
- Smaller ecosystem for CAD file format handling compared to the mature C++ Open Cascade ecosystem
- wgpu abstraction layer adds slight overhead compared to raw Vulkan/Metal for compute-bound solver kernels

### Flutter Desktop

**Architecture:** Impeller/Skia for simulation setup UI; native texture bridge to OpenGL/Metal for 3D mesh viewport; Dart FFI to Rust solver core; platform channels for GPU compute dispatch and result streaming

**Tech stack:** Dart + Flutter, flutter_rust_bridge for solver FFI, custom OpenGL texture plugin for 3D viewport, drift for SQLite, fl_chart for convergence plots

**Native module needs:** Rust solver via FFI, OpenGL/Metal interop for mesh rendering, HDF5 via C FFI, Open Cascade via C FFI

**Bundle size:** ~90-130 MB

**Memory:** ~280-450 MB

**Pros:**
- Polished UI for complex simulation setup wizards with animated transitions, multi-step forms, and inline parameter validation
- Cross-platform with shared UI code for Windows/macOS/Linux — single codebase for all simulation configuration panels
- Hot reload accelerates iteration on simulation parameter panels, result dashboards, and convergence monitoring displays
- Strong widget library for building data-dense convergence monitoring interfaces with real-time chart updates

**Cons:**
- No native 3D pipeline — mesh visualization requires custom texture bridge adding significant complexity and a frame of latency
- Dart VM memory overhead (80-120 MB) reduces available RAM for solver working sets and large mesh storage
- GPU compute access only through Rust FFI — additional latency layer for solver dispatch and result retrieval
- No existing Flutter packages for scientific visualization — contour plots, vector fields, isosurface rendering all require custom work

### Swift/SwiftUI (macOS)

**Architecture:** SwiftUI for native macOS simulation setup UI; Metal for both CFD compute shaders and 3D mesh rendering; SceneKit/custom Metal renderer for flow visualization; Swift-Rust FFI for solver core algorithms; Core Data for project management

**Tech stack:** SwiftUI + Metal, custom Metal compute shaders for LBM solver, Metal rendering for mesh visualization, Swift-Rust FFI for core engine, GRDB/SQLite for data

**Native module needs:** Metal compute shaders (native), Accelerate framework for sparse linear algebra (BLAS/LAPACK/Sparse Solvers), Rust solver core via C FFI

**Bundle size:** ~35-55 MB

**Memory:** ~120-250 MB

**Pros:**
- Metal compute shaders provide lowest-latency GPU solver execution on Apple Silicon with unified memory architecture — zero CPU-GPU transfer
- Apple Silicon unified memory eliminates GPU-CPU data copy for solver-to-renderer result passing, a critical advantage for large result fields
- Native macOS integration — Continuity for iPad result viewing, Handoff, optimized scheduling for Apple Silicon efficiency/performance cores
- Smallest bundle and memory footprint among all options, leaving maximum resources for simulation workloads

**Cons:**
- macOS-only — CFD engineers overwhelmingly use Linux workstations with NVIDIA GPUs running CUDA-accelerated solvers
- Metal-only compute locks out the CUDA ecosystem and NVIDIA hardware that dominates high-performance computing in CFD
- Smaller Swift ecosystem for scientific computing compared to C++/Fortran CFD tooling (OpenFOAM, SU2, FEniCS)
- No path to Linux/Windows without complete solver and renderer rewrite — would effectively require a second application

### Kotlin Multiplatform

**Architecture:** Compose Multiplatform for shared simulation UI; LWJGL/OpenGL for 3D mesh rendering; JNI to Rust solver core; Kotlin coroutines for async solver execution and convergence monitoring; SQLDelight for project data

**Tech stack:** Kotlin + Compose Multiplatform, LWJGL for OpenGL rendering, JNI to Rust solver, SQLDelight for cross-platform data, kotlinx.coroutines for async solver management

**Native module needs:** JNI to Rust CFD solver, LWJGL for OpenGL context, JOML for 3D math, JNI to HDF5 library

**Bundle size:** ~130-190 MB

**Memory:** ~350-550 MB

**Pros:**
- JVM provides access to Apache Commons Math, Kotlin multik, and EJML for numerical computing and matrix operations
- Kotlin coroutines elegantly manage concurrent solver jobs, convergence monitoring, and real-time UI updates without thread complexity
- Cross-platform desktop with shared UI and solver integration code across Windows/macOS/Linux
- Strong typing and null safety reduce runtime errors in complex simulation configuration hierarchies

**Cons:**
- JVM memory overhead (200+ MB baseline) directly competes with solver memory requirements for large meshes — a 10M cell mesh needs every available MB
- JNI boundary adds measurable latency for high-frequency data transfer between solver and renderer during real-time visualization
- No GPU compute access from JVM — all solver compute must traverse the JNI boundary to native Rust/GPU code
- GC pauses cause visible stuttering during interactive mesh manipulation and rotation of million-cell models at 60fps

---

## Comparison Matrix

| Dimension | Electron | Tauri | Flutter | Swift | Kotlin MP |
|-----------|----------|-------|---------|-------|-----------|
| Dev effort (weeks) | 16 | 20 | 22 | 18 | 24 |
| Bundle size | 340 MB | 65 MB | 110 MB | 45 MB | 160 MB |
| Memory usage | 650 MB | 220 MB | 360 MB | 180 MB | 450 MB |
| Startup time | 3.5s | 0.9s | 2.0s | 0.6s | 2.8s |
| Native feel | 5/10 | 7/10 | 7/10 | 10/10 | 6/10 |
| Offline capability | 8/10 | 9/10 | 8/10 | 9/10 | 8/10 |
| GPU/compute access | 3/10 | 9/10 | 4/10 | 10/10 | 4/10 |
| Cross-platform | 9/10 | 9/10 | 8/10 | 2/10 | 7/10 |
| Best fit score | 5/10 | 9/10 | 5/10 | 6/10 | 4/10 |

---

## Recommended Framework

**Tauri** is the best fit for FlowCFD.

**Rationale:** CFD simulation demands the tightest possible coupling between GPU compute (solver) and GPU rendering (visualization) — Tauri's wgpu backend enables both to share the same GPU context, eliminating the data transfer bottleneck that plagues IPC-based architectures when visualizing results from multi-million-cell simulations. The Rust-native solver runs in-process with direct memory access to mesh data structures, and rayon parallelism saturates all CPU cores during mesh generation. Cross-platform support is critical since CFD engineers work across Linux workstations, Windows desktops, and increasingly Apple Silicon machines.

**Runner-up:** Swift/SwiftUI — Metal compute on Apple Silicon with unified memory is technically superior for single-platform performance, but the macOS-only restriction is fatal for a CFD tool where the majority of users run Linux with NVIDIA GPUs and rely on the CUDA compute ecosystem.

---

## Monetization (Desktop)

- **Free tier:** Laminar flow solver only, meshes up to 100K cells, basic post-processing (contour plots, streamlines), single simulation at a time
- **Pro license ($79/month):** Turbulent flow models (k-epsilon, k-omega SST, LES), meshes up to 10M cells, full post-processing suite with animation export, parallel simulation queue
- **Enterprise license ($299/seat/month):** Unlimited mesh size, custom boundary conditions, HPC cluster offload, parametric study automation, priority support, floating license pool
- **Solver add-ons ($29/month each):** Conjugate heat transfer module, multiphase flow (VOF), combustion modeling with species transport, aeroacoustics (FW-H analogy)
- **Academic license:** Free for university research and coursework with .edu verification; $19/month for government and commercial research labs

---

## MVP Scope (Desktop)

**Core MVP (8 weeks):**

| Week | Milestone |
|------|-----------|
| 1-2 | Rust core: STL mesh importer with format validation, Delaunay surface mesher with quality constraints, boundary layer insertion with configurable growth ratio; SQLite project schema with mesh version tracking; Tauri shell with file browser, mesh statistics panel, and element quality histogram |
| 3-4 | wgpu mesh renderer: wireframe, surface, and element-quality-colored modes for meshes up to 2M cells at 60fps; orbit/pan/zoom camera controls with inertia; boundary condition assignment via interactive face selection with visual highlighting |
| 5-6 | LBM solver: D3Q19 wgpu compute shader implementation with BGK collision operator, bounce-back wall boundaries, velocity/pressure inlet-outlet conditions; real-time convergence plotting with residual history; solver pause/resume and checkpoint save |
| 7 | Post-processing: velocity and pressure contour overlay on mesh surfaces with configurable color maps, streamline integration with RK4 adaptive stepping, probe placement for time-series extraction at arbitrary locations; result export to VTK format |
| 8 | Simulation workflow: case setup wizard (geometry import, mesh parameters, boundary conditions, solver settings), batch run queue with progress monitoring, side-by-side result comparison viewport; end-to-end cross-platform testing and installer packaging |

**Post-MVP:**

- Finite Volume Method solver with SIMPLE pressure-velocity coupling for higher-accuracy incompressible flow simulations
- Turbulence modeling: k-epsilon (standard/RNG/realizable) and k-omega SST RANS models with automatic near-wall treatment
- Conjugate heat transfer module for electronics cooling, heat exchanger design, and building HVAC analysis
- Parametric study engine for automated geometry/parameter sweeps with result correlation analysis and sensitivity plots
- CAD integration: STEP/IGES import via Open Cascade for direct geometry-to-mesh workflow without intermediate STL conversion
- Multiphase flow solver using Volume of Fluid (VOF) method for free-surface, sloshing, and droplet simulations
- Cloud HPC offload for simulations exceeding local GPU memory or requiring multi-GPU scaling across cluster nodes
- OpenFOAM case import/export for interoperability with existing CFD workflows and validation against established solvers
- Acoustic analysis module using Ffowcs Williams-Hawkings analogy for far-field noise prediction from transient flow data
- Moving mesh capability for rotating machinery (fans, turbines, pumps) using sliding mesh and arbitrary mesh interface (AMI) methods
- Species transport solver for mixing and combustion modeling with Arrhenius reaction rate kinetics and flamelet tables
- Adjoint-based shape optimization for automated design improvement — compute sensitivity of drag/lift to surface deformations
- Result export to ParaView format (.vtm, .pvd) for advanced post-processing and publication-quality rendering with ray tracing
- Porous media model for simulating flow through packed beds, filters, and porous structures using Darcy-Forchheimer resistance terms
