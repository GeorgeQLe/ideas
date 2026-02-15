# PrintGen Desktop — 5-Framework Comparison

## Desktop Rationale

- **GPU-Accelerated Generative Algorithms:** Topology optimization and generative design solve iterative finite element problems across millions of voxels — GPU compute shaders (CUDA/Vulkan/Metal) deliver orders-of-magnitude speedup over CPU for density-based SIMP and level-set methods, requiring direct hardware access. A single topology optimization run on a 256x256x256 voxel grid requires solving a sparse linear system with 16M+ degrees of freedom at every iteration, generating billions of floating-point operations that only GPU compute can execute interactively.

- **3D Model Visualization:** Interactive rendering of complex generative geometries with lattice structures, organic surfaces, and stress overlays demands real-time mesh rendering with PBR materials, section views, and transparency — only achievable with native GPU graphics pipelines. Gyroid lattice structures alone can contain 10M+ triangles that must render at 60fps with stress-colored overlays.

- **STL/OBJ/3MF File Handling:** 3D printing workflows revolve around mesh files that can exceed hundreds of megabytes for high-resolution generative designs — native file system access with memory-mapped I/O enables instant loading and manipulation without browser memory limits. A high-resolution topology-optimized part exported at 0.1mm feature resolution can generate 50M+ triangle STL files requiring 500+ MB of storage.

- **Local Slicer Integration:** Direct integration with slicing engines (PrusaSlicer, Cura, OrcaSlicer) requires process spawning, shared file system access, and stdout/stderr parsing for print time and material estimates — impossible from a browser sandbox. The round-trip from design to print estimation must complete in seconds for interactive design iteration.

- **Topology Optimization Compute:** SIMP (Solid Isotropic Material with Penalization) and level-set optimization require solving sparse linear systems (millions of DOFs) iteratively until convergence — desktop provides direct access to GPU sparse solvers (algebraic multigrid, preconditioned conjugate gradient) and multi-core CPU parallelism essential for interactive design exploration with sub-minute convergence.

---

## Desktop-Specific Features

The following features are only feasible on desktop due to GPU compute for FEA, large mesh handling, and local slicer integration:

- GPU-accelerated topology optimization using SIMP and level-set methods via wgpu/Metal/CUDA compute shaders with real-time density field visualization during iteration
- Real-time 3D visualization of generative designs with stress heatmaps, density gradient overlays, section planes, and lattice structure rendering at 60fps
- Native STL/OBJ/3MF/STEP import and export with automatic mesh repair (hole filling, non-manifold edge resolution, normal orientation correction, degenerate triangle removal)
- Direct slicer integration: spawn PrusaSlicer/Cura/OrcaSlicer CLI for print time estimation, layer-by-layer preview, support structure generation, and G-code export
- Finite element analysis (FEA) engine for structural simulation with GPU-accelerated sparse matrix assembly and preconditioned conjugate gradient solver
- Lattice structure generation: BCC, FCC, Gyroid, Schwarz-P, Diamond, Octet-truss unit cells with graded density mapped from FEA stress distribution results
- Multi-resolution design workspace: coarse voxel grid optimization (fast exploration) followed by fine-mesh surface extraction via dual marching cubes
- Background optimization queue — run hour-long multi-objective generative design explorations while continuing manual CAD modeling in the foreground
- Build plate visualization with printer-specific dimensions, support structure preview, print orientation optimization for minimum support volume, and layer time estimation
- Material property database with validated mechanical properties (yield strength, Young's modulus, elongation, fatigue life) for 50+ FDM/SLA/SLS 3D printing materials

---

## Shared Packages

All shared packages are compiled as Rust libraries with C-compatible FFI exports, enabling consumption from any framework.

### core (Rust)

- Topology optimization engine: SIMP method with sensitivity filtering (density/Helmholtz), density projection (Heaviside), volume constraint enforcement, and move-limit update scheme; GPU compute shader dispatch via wgpu for parallel element stiffness assembly and objective gradient evaluation
- Level-set topology optimization: Hamilton-Jacobi level-set evolution with upwind finite differences, velocity field computation from adjoint-based shape sensitivity, reinitialization via fast marching method, and nucleation for topology changes
- Finite element solver: 8-node hexahedral elements with linear elasticity (plane stress, plane strain, 3D), GPU-accelerated sparse conjugate gradient solver with algebraic multigrid preconditioner, and support for multiple load cases with superposition
- Mesh processing library: STL/OBJ/3MF reader/writer with robust parsing, mesh repair pipeline (hole filling via ear clipping, duplicate vertex merging, normal reorientation via BFS), mesh decimation (quadric error metrics), and Loop/Catmull-Clark subdivision
- Surface extraction pipeline: marching cubes and dual marching cubes for isosurface extraction from scalar density fields, Laplacian and Taubin smoothing, feature-preserving remeshing with size gradation, and sharp edge detection
- Lattice structure generator: parametric unit cell library with analytic TPMS surfaces (Gyroid, Schwarz-P, Diamond, IWP), beam-based lattices (BCC, FCC, Octet), graded density mapping via stress-weighted interpolation, and conformal lattice fitting to arbitrary bounding geometry

### api-client (Rust)

- 3D printer farm management client for submitting print jobs to OctoPrint/Klipper/Duet-connected printers with job status monitoring and webcam preview
- Material database sync client for downloading updated mechanical property data, print parameter profiles, and manufacturer-validated settings
- Cloud generative design offload for multi-objective optimization problems exceeding local GPU memory or requiring population-based search (NSGA-II)
- License validation with offline activation code supporting perpetual fallback for air-gapped manufacturing environments
- Design sharing client for publishing optimized geometries to team library with version tracking, design intent annotations, and approval workflows

### data (SQLite)

- Design project database tracking optimization parameters (volume fraction, filter radius, penalization power), boundary conditions (loads, fixtures), load cases, and complete design iteration history with density field snapshots for provenance
- Material property store with validated mechanical properties, print parameters (nozzle temperature, bed temperature, speed, retraction), and printer compatibility matrix indexed by material brand, type, and color
- Print job history tracking material usage (grams), print time (hours), success/failure outcome, dimensional accuracy measurements, and surface quality ratings for quality control
- Optimization result cache storing converged density fields as compressed 3D arrays, compliance values, volume fractions, and stress distributions for parameter study comparison and convergence analysis

---

## Framework Implementations

Each framework is evaluated for its ability to deliver GPU-accelerated topology optimization, real-time 3D model visualization, and slicer integration.

### Electron

**Architecture:** Chromium renderer for React-based design setup UI and parameter panels; Three.js with custom shaders for 3D model visualization and stress overlay rendering; Node.js child processes spawning Rust optimization engine binary; SharedArrayBuffer for streaming intermediate density fields to WebGL renderer during live optimization

**Tech stack:** React + TypeScript, Three.js with custom GLSL shaders for stress/density visualization, Node.js native addons via napi-rs for Rust engine bindings, WebGL2 for 3D rendering, better-sqlite3 for project data

**Native module needs:** napi-rs bindings to Rust optimization engine, node-stl for mesh I/O, child_process for slicer CLI integration, node-usb for direct printer serial communication

**Bundle size:** ~270-340 MB

**Memory:** ~450-700 MB

**Pros:**
- Three.js ecosystem provides mature 3D rendering with STL loading, orbit controls, section plane clipping, and customizable material shaders
- Large community of JavaScript 3D graphics developers familiar with generative design visualization and parametric modeling UIs
- Cross-platform with consistent visualization behavior on Windows/macOS/Linux maker and engineering workstations
- Easy integration with web-based slicer previews (Kiri:Moto), 3D model marketplaces, and online print services

**Cons:**
- WebGL2 cannot handle meshes exceeding 2M triangles without aggressive LOD, frustum culling, and octree-based visibility determination
- No compute shader access — topology optimization must run in separate Rust process with IPC overhead for density field transfer every iteration
- Chromium memory overhead (200+ MB) wastes RAM needed for FEA sparse matrix storage and large voxel grid density fields
- Three.js rendering quality insufficient for photorealistic print preview without significant custom PBR shader and lighting work

### Tauri

**Architecture:** Rust backend running topology optimization, FEA solver, and mesh processing natively with zero-copy data flow; wgpu compute shaders for GPU-accelerated stiffness matrix assembly and sparse linear solve; wgpu renderer for 3D model visualization with stress overlays sharing GPU context with compute pipeline; WebView frontend for design parameter panels and optimization progress charts; slicer integration via Rust std::process

**Tech stack:** Rust backend with wgpu compute + rendering, Svelte frontend for reactive parameter controls, sqlx for SQLite, rayon for parallel mesh processing, nalgebra-sparse for linear algebra

**Native module needs:** wgpu for Vulkan/Metal/DX12 compute and rendering, PrusaSlicer CLI integration via std::process::Command, serialport crate for direct printer communication

**Bundle size:** ~45-70 MB

**Memory:** ~130-280 MB

**Pros:**
- wgpu compute shaders for FEA solver and 3D renderer share the same GPU context — optimization density field renders without any buffer copy or transfer
- Rust-native mesh processing (STL repair, marching cubes, remeshing, lattice generation) runs at native speed without FFI serialization overhead
- Minimal memory footprint leaves maximum RAM for sparse matrix storage during large-scale topology optimization with millions of DOFs
- Direct slicer CLI integration via Rust std::process with typed output parsing — PrusaSlicer print time and filament estimates available in milliseconds

**Cons:**
- Must build custom 3D rendering pipeline from scratch — no off-the-shelf CAD/mesh viewer exists in the Rust/wgpu ecosystem
- WebView limitations for interactive 3D boundary condition setup (click-to-select faces for load/fixture application with visual feedback)
- Smaller ecosystem for CAD format handling compared to the mature C++ Open Cascade and CGAL geometry processing libraries
- Custom wgpu renderer requires significant graphics programming expertise for PBR materials, section views, and transparency rendering

### Flutter Desktop

**Architecture:** Impeller/Skia for design parameter UI, optimization dashboard, and material selection panels; native texture bridge for 3D model viewport rendered via platform GPU API; Dart FFI to Rust optimization core; platform channels for slicer CLI integration and printer communication

**Tech stack:** Dart + Flutter, flutter_rust_bridge for optimization engine FFI, custom native texture plugin for 3D viewport rendering, drift for SQLite, fl_chart for convergence plots

**Native module needs:** Rust optimization core via FFI, OpenGL/Metal interop for 3D mesh rendering viewport, PrusaSlicer CLI via dart:io process, serial port via platform channel

**Bundle size:** ~85-120 MB

**Memory:** ~260-420 MB

**Pros:**
- Polished UI for complex design parameter wizards with constraint editors, material property browsers, and load case configuration panels
- Cross-platform with shared optimization control UI across Windows/macOS/Linux — single codebase for all design configuration screens
- Hot reload accelerates iteration on design parameter panels, convergence dashboards, and result visualization layouts
- Strong animation system for visualizing optimization convergence: animated density field evolution and compliance curve progression

**Cons:**
- No native 3D rendering pipeline — model viewer requires custom texture bridge adding architectural complexity and a frame of display latency
- Dart VM memory overhead (80-120 MB) competes with FEA solver memory for sparse matrix storage on memory-constrained systems
- GPU compute for topology optimization only accessible through Rust FFI — no direct compute shader dispatch from Flutter layer
- No Flutter packages for engineering visualization — stress contour plots, section cut views, and exploded assembly views all require custom implementation

### Swift/SwiftUI (macOS)

**Architecture:** SwiftUI for native macOS design UI with inspector panels; Metal compute shaders for GPU-accelerated topology optimization and FEA sparse solver; SceneKit/Model I/O for 3D model rendering with PBR materials and environment lighting; Swift-Rust FFI for mesh processing and lattice generation algorithms; Core Data for project management

**Tech stack:** SwiftUI + Metal, SceneKit with Model I/O for 3D rendering, Metal compute shaders for FEA solver and optimization, Swift-Rust FFI for core algorithms, GRDB/SQLite for data

**Native module needs:** Metal compute shaders (native), Model I/O for STL/OBJ/USD import (native framework), Rust mesh processing via C-compatible FFI

**Bundle size:** ~30-50 MB

**Memory:** ~110-220 MB

**Pros:**
- Metal compute shaders provide highest GPU utilization for FEA sparse solver on Apple Silicon with automatic workgroup sizing
- Model I/O framework provides built-in STL/OBJ/USD import with automatic PBR material assignment and rendering via SceneKit
- Apple Silicon unified memory enables zero-copy access to optimization density field results for immediate rendering — no GPU-CPU transfer
- Native macOS design experience with inspector panels, Continuity for iPad companion sketching, and AR model preview via Quick Look

**Cons:**
- macOS-only — the 3D printing community heavily uses Windows for slicer integration (Cura, PrusaSlicer run best on Windows)
- Model I/O limited to standard mesh formats — no 3MF support (the modern 3D printing format) without third-party library
- Metal-only compute excludes NVIDIA CUDA cards that many engineering designers use for GPU-accelerated rendering and simulation
- No path to Windows/Linux without complete rendering, compute, and platform integration rewrite

### Kotlin Multiplatform

**Architecture:** Compose Multiplatform for shared design UI with parameter panels; LWJGL/OpenGL for 3D model rendering with custom shaders for stress visualization; JNI to Rust optimization engine and mesh processing core; Kotlin coroutines for async optimization job management with progress reporting; SQLDelight for project data

**Tech stack:** Kotlin + Compose Multiplatform, LWJGL for OpenGL 3D rendering, JNI to Rust core engine, SQLDelight for cross-platform SQLite, kotlinx.coroutines for async job management

**Native module needs:** JNI to Rust optimization engine, LWJGL for OpenGL context and 3D rendering, JOML for 3D math (vectors, matrices, quaternions), JNI for slicer CLI integration

**Bundle size:** ~125-180 MB

**Memory:** ~320-500 MB

**Pros:**
- JVM provides access to numerical computing libraries (EJML for sparse matrices, Apache Commons Math for optimization) for FEA verification and validation
- Kotlin coroutines manage concurrent optimization jobs with structured cancellation, progress reporting, and result collection
- Cross-platform desktop with shared design UI and optimization configuration code across Windows/macOS/Linux
- LWJGL provides comprehensive OpenGL 4.x access for building a custom 3D rendering pipeline with compute shader support

**Cons:**
- JVM memory overhead (200+ MB baseline) directly reduces available RAM for sparse matrix storage in FEA solver — critical for large optimization problems
- JNI boundary adds measurable latency for transferring density fields between optimization engine and renderer at every iteration
- No direct GPU compute access from JVM — entire FEA solver and topology optimization pipeline must traverse JNI to native Rust/GPU code
- GC pauses cause visible frame drops during interactive manipulation and rotation of high-polygon generative design meshes

---

## Comparison Matrix

| Dimension | Electron | Tauri | Flutter | Swift | Kotlin MP |
|-----------|----------|-------|---------|-------|-----------|
| Dev effort (weeks) | 14 | 18 | 20 | 16 | 22 |
| Bundle size | 310 MB | 55 MB | 100 MB | 40 MB | 150 MB |
| Memory usage | 580 MB | 200 MB | 340 MB | 160 MB | 410 MB |
| Startup time | 3.2s | 0.8s | 1.9s | 0.5s | 2.6s |
| Native feel | 5/10 | 7/10 | 7/10 | 10/10 | 6/10 |
| Offline capability | 8/10 | 9/10 | 8/10 | 9/10 | 8/10 |
| GPU/compute access | 3/10 | 9/10 | 4/10 | 10/10 | 4/10 |
| Cross-platform | 9/10 | 9/10 | 8/10 | 2/10 | 7/10 |
| Best fit score | 5/10 | 9/10 | 5/10 | 7/10 | 4/10 |

---

## Recommended Framework

**Tauri** is the best fit for PrintGen.

**Rationale:** Generative design requires tight iteration between GPU-accelerated topology optimization (compute shaders solving sparse FEA systems across millions of voxels) and real-time visualization of evolving density fields — Tauri's wgpu backend enables both to share the same GPU context with zero-copy data flow from solver output buffer to renderer input texture. The Rust-native mesh processing pipeline (STL repair, marching cubes surface extraction, lattice generation) runs in-process without FFI overhead, and direct process spawning integrates cleanly with PrusaSlicer/Cura/OrcaSlicer CLI for instant print time estimation. Cross-platform support is essential since the 3D printing maker and professional engineering community spans Windows, macOS, and Linux.

**Runner-up:** Swift/SwiftUI — Metal compute and Model I/O provide excellent single-platform performance with the most polished native design experience, but the macOS-only limitation excludes the Windows-dominant 3D printing community and their established slicer and printer management ecosystems.

---

## Monetization (Desktop)

- **Free tier:** Manual 3D model viewer with mesh inspection, basic mesh repair tools, single-load-case topology optimization up to 50K elements, STL export only
- **Pro license ($59/month):** Multi-load-case optimization, lattice structure generation (all unit cell types), unlimited element count, full FEA stress analysis with reporting, slicer integration for all supported slicers, 3MF export with color-mapped stress data
- **Enterprise license ($199/seat/month):** Multi-objective Pareto optimization (NSGA-II), parametric design study automation, printer farm integration with OctoPrint/Klipper, custom material database with property validation, API access for CAD plugin development
- **Material pack add-ons ($9.99 each):** Curated print profiles with experimentally validated mechanical properties for specific printer-material combinations (e.g., Bambu Lab X1C + eSUN PLA+)
- **Academic license:** Free for students, educators, and university makerspaces with .edu verification; includes all Pro features for non-commercial use

---

## MVP Scope (Desktop)

**Core MVP (8 weeks):**

| Week | Milestone |
|------|-----------|
| 1-2 | Rust core: STL/OBJ mesh reader/writer with repair pipeline (hole filling, normal correction, duplicate vertex removal, degenerate triangle filtering); voxelization engine for converting surface meshes to regular hexahedral grids with boundary detection; SQLite schema for projects, materials, and optimization history; basic Tauri shell with file open dialog, mesh statistics panel, and element count display |
| 3-4 | wgpu 3D renderer: surface rendering with flat/smooth shading and wireframe overlay, stress heatmap colormap (jet, viridis, coolwarm), section plane clipping; orbit/pan/zoom camera controls with inertia; mesh face selection for boundary condition assignment (click to mark fixed faces, shift-click for loaded faces with force vector input) |
| 5-6 | Topology optimization: SIMP method with sensitivity filtering (Helmholtz PDE filter) and volume constraint, wgpu compute shaders for parallel element stiffness matrix assembly and global matrix-vector products, GPU-accelerated preconditioned conjugate gradient solver; real-time density field visualization updating every iteration; volume fraction slider and compliance convergence line chart |
| 7 | Surface extraction: marching cubes isosurface from converged density field with configurable threshold, Laplacian smoothing with feature preservation, STL and 3MF export with metadata; PrusaSlicer CLI integration for print time and filament usage estimates; build plate visualization with configurable printer dimensions and orientation |
| 8 | Lattice and materials: Gyroid and BCC unit cell generation with uniform density parameter; material property selector with 10 common FDM materials (PLA, PETG, ABS, ASA, Nylon, TPU, PC, CF-PLA, CF-PETG, PLA+) and their validated mechanical properties; side-by-side design comparison viewport for A/B testing optimization variants; end-to-end cross-platform testing and installer packaging |

**Post-MVP:**

- Level-set topology optimization for smoother boundaries, sharper features, and more manufacturable designs without staircase artifacts
- Graded lattice density mapping: interpolate lattice relative density from FEA von Mises stress distribution for weight-optimized functional infill
- Multi-objective optimization: simultaneous compliance and mass minimization with Pareto front visualization and interactive solution selection
- Advanced lattice library: FCC, Schwarz-P, Diamond, IWP, Octet-truss unit cells with parametric strut/wall thickness control and conformal fitting
- Direct printer communication via serial/USB for print job submission, real-time progress monitoring, and temperature tracking
- Parametric design study engine for automated sweeps over volume fraction, filter radius, load magnitude, and material type with result correlation heatmaps
- Cloud optimization offload for problems exceeding local GPU memory (>10M elements) with job queuing and result download
- CAD format support: STEP/IGES import via Open Cascade for direct geometry-to-optimization workflow without intermediate STL tessellation
- Support structure auto-generation with tree/linear modes, breakaway/soluble material assignment, and minimum overhang angle configuration
- Multi-material topology optimization for designs spanning multiple print materials with interface bonding constraints
- Print orientation optimizer: exhaustive search over rotation angles to minimize support volume, maximize surface quality, and optimize layer adhesion direction relative to primary load paths
- Thermal simulation for predicting warpage and residual stress during FDM printing with layer-by-layer heat transfer modeling
- DfAM (Design for Additive Manufacturing) rule checker: minimum wall thickness, overhang angle limits, bridge length warnings, and trapped volume detection
- Mesh boolean operations (union, difference, intersection) for combining multiple optimized components into assemblies with interference checking
- Export to slicer-native project formats (3MF with print settings, Cura project files) preserving material assignments and support configuration
