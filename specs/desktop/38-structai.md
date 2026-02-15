# StructAI Desktop — 5-Framework Comparison

## Desktop Rationale

- **GPU-Accelerated Finite Element Analysis:**
  Structural engineering analysis involves solving large sparse linear systems arising
  from finite element discretization of buildings, bridges, and other structures.
  Desktop GPU compute enables massively parallel sparse matrix operations (conjugate
  gradient, direct solvers) that reduce analysis time from minutes to seconds for
  models with millions of degrees of freedom.

- **3D Structural Visualization:**
  Engineers need interactive 3D visualization of structural models showing deformed
  shapes, stress/strain distributions, mode shapes, and load paths. Desktop GPU
  rendering provides real-time manipulation of complex structural models with
  color-mapped result overlays at 60fps, enabling rapid visual inspection of
  analysis results.

- **Local Building Codes Database:**
  Structural design requires constant reference to building codes (IBC, Eurocode,
  AS/NZS), material standards (AISC, ACI, EN), and design guides. A local indexed
  database provides instant lookup of code provisions, load combinations, and
  capacity equations without relying on online code repositories or slow PDF searches.

- **Large BIM Model Handling:**
  Building Information Models (BIM) from Revit, Tekla, and similar tools can exceed
  several gigabytes with detailed geometry, connections, and metadata. Desktop
  applications with memory-mapped file access and GPU-accelerated rendering can handle
  these large models without the memory and bandwidth limitations of browser-based
  viewers.

- **Offline Structural Analysis:**
  Structural engineers frequently work on construction sites, in client offices during
  design reviews, and in disaster response scenarios where network access is
  unreliable. Desktop-native FEA ensures uninterrupted structural analysis, code
  checking, and design iteration regardless of connectivity.

---

## Desktop-Specific Features

- GPU-accelerated finite element solver using wgpu compute shaders for sparse matrix
  assembly, factorization, and iterative solution of large structural systems with
  millions of DOFs
- Real-time 3D structural model visualization with GPU-rendered deformed shapes,
  von Mises stress contours, principal stress vectors, and animated mode shapes at
  interactive framerates
- AI-assisted structural optimization engine using GPU-parallel topology optimization
  and genetic algorithms to suggest efficient structural configurations that minimize
  material while satisfying code requirements
- Local building codes database with indexed full-text search across IBC, ASCE 7,
  AISC 360, ACI 318, Eurocode series, and regional amendments with automatic load
  combination generation
- BIM model import engine supporting IFC, Revit (.rvt via Open Design Alliance), and
  Tekla formats with intelligent structural member recognition and analytical model
  extraction
- Connection design module with GPU-accelerated bolt/weld pattern analysis,
  moment-rotation curve generation, and automated connection detailing for steel
  structures
- Seismic analysis suite with GPU-parallel response spectrum analysis, time-history
  analysis, and pushover analysis for performance-based earthquake engineering
- Wind load generation engine with GPU-accelerated computational wind engineering
  (CWE) for complex building geometries using simplified RANS or lattice Boltzmann
  methods
- Progressive collapse analysis module with GPU-parallel nonlinear dynamic simulation
  of member removal scenarios for robustness assessment
- Native file system integration for importing/exporting IFC, SDNF, CIS/2, and DXF
  formats used in structural engineering workflows

---

## Shared Packages

### core (Rust)

- Finite element engine implementing beam, shell, plate, and solid element
  formulations with GPU-accelerated stiffness matrix assembly and sparse linear
  solver dispatch via wgpu compute shaders using preconditioned conjugate gradient
  and sparse Cholesky methods
- Structural analysis orchestrator managing static, modal, buckling, response
  spectrum, and time-history analysis procedures with load combination generation
  per code requirements and result envelope extraction
- AI optimization module implementing topology optimization (SIMP method), genetic
  algorithm-based sizing optimization, and machine learning-guided member selection
  using GPU-parallel objective function evaluation
- Building code engine encoding design provisions from AISC 360/341, ACI 318,
  ASCE 7, Eurocode 2/3/8, and regional codes with demand/capacity ratio computation
  and utilization checking for steel, concrete, timber, and masonry members
- Geometry and mesh engine handling BIM model parsing, analytical model extraction,
  mesh generation for 2D/3D elements, and coordinate transformation with
  GPU-accelerated geometric queries
- Result post-processing pipeline computing derived quantities (von Mises stress,
  principal stresses, drift ratios, demand/capacity ratios) from raw nodal
  displacements with GPU-parallel field computation

### api-client (Rust)

- Cloud sync client for sharing structural models and analysis results across
  distributed engineering teams with version-controlled model management and
  approval workflows
- Material database connector for querying steel section databases (AISC, European,
  Australian), concrete mix properties, and timber species data with local caching
- Telemetry and crash reporting with anonymized analysis performance metrics and
  model complexity statistics
- License validation and feature-flag management supporting student, professional,
  and enterprise tier access with concurrent-use licensing support
- Integration endpoints for connecting with project management platforms (Procore,
  Autodesk Construction Cloud) and publishing analysis reports

### data (SQLite)

- Building codes database with indexed storage of design provisions, load factors,
  resistance factors, and capacity equations from major international codes with
  version tracking for code year updates
- Steel section database storing geometric properties, plastic moduli, warping
  constants, and design capacities for AISC, European, Australian, and other
  international section series
- Analysis results store with compressed storage for nodal displacements, element
  forces, stress fields, and mode shapes enabling instant result switching and
  animated visualization
- Project workspace persistence tracking structural model states, analysis
  configurations, visualization viewports, and code checking reports across
  design sessions

---

## Framework Implementations

### Electron

**Architecture:**
Multi-process Chromium architecture with main process coordinating Rust FEA solver
via N-API bindings. Renderer processes host React-based UI with Three.js for 3D
structural model visualization and WebGL2 for stress contour rendering. GPU compute
dispatched through Rust sidecar for sparse matrix operations.

**Tech stack:**
React 18 + TypeScript, Three.js for 3D structural rendering, custom WebGL2 shaders
for stress visualization, D3.js for charts and diagrams, napi-rs for Rust FFI,
better-sqlite3 for codes database, electron-builder for packaging.

**Native module needs:**
Rust FEA solver as native Node addon via napi-rs, wgpu compute runtime for
GPU-accelerated sparse linear algebra, SQLite native binding for codes database,
IFC parser native module.

**Bundle size:** ~310-390 MB
**Memory:** ~430-680 MB

**Pros:**
- Three.js provides mature 3D rendering foundation for structural model
  visualization with extensive documentation and community support for engineering
  visualization patterns
- WebGL2 shader ecosystem enables custom stress contour rendering with smooth
  interpolation and vector field visualization
- Large developer pool familiar with React reduces hiring difficulty for UI-intensive
  structural analysis interface development
- Established desktop application patterns (VS Code) guide architecture of
  multi-panel engineering analysis environments

**Cons:**
- Chromium memory overhead limits available RAM for large BIM model loading and
  FEA solution storage with million-DOF models
- WebGL2 lacks compute shaders — all FEA computation must route through Rust sidecar
  with IPC overhead for sparse matrix data
- Three.js rendering performance degrades for structural models with 100,000+
  elements requiring advanced LOD management
- Large bundle size problematic for deployment in construction company IT
  environments with software approval processes

### Tauri

**Architecture:**
Rust backend hosting FEA solver, code checking engine, and AI optimization module
directly in the application process. wgpu compute shaders execute sparse matrix
operations natively. Frontend via system WebView with Three.js-lite or custom WebGL
for 3D model visualization. Optional dedicated wgpu render window for high-performance
structural visualization.

**Tech stack:**
Rust backend with wgpu for GPU compute and optional 3D rendering, Svelte or Solid.js
frontend, WebGL for 3D model visualization in WebView, rusqlite for codes and
sections database, IFC parser in Rust, serde for IPC.

**Native module needs:**
wgpu runtime for GPU compute (sparse linear algebra, topology optimization) and
rendering (3D structural visualization), system WebView, IFC/BIM format parsing
libraries.

**Bundle size:** ~40-60 MB
**Memory:** ~130-250 MB

**Pros:**
- Rust-native FEA solver with direct wgpu compute dispatch provides maximum GPU
  throughput for sparse matrix operations without FFI overhead
- Minimal memory footprint (130-250 MB) leaves maximum RAM for large BIM models
  and FEA solution storage with million-DOF systems
- wgpu provides unified GPU compute and rendering pipeline — compute shaders for
  FEA with seamless transition to stress visualization rendering
- Compact bundle size enables easy deployment in enterprise construction company
  environments with strict IT policies

**Cons:**
- System WebView 3D rendering less mature than Chromium Three.js for complex
  structural model visualization — may require dedicated wgpu render window
- Smaller Rust ecosystem for BIM/IFC format parsing compared to C++ (IfcOpenShell)
  or .NET (xBIM) alternatives
- WebView debugging tools less mature for troubleshooting 3D structural
  visualization issues
- Custom wgpu 3D renderer requires significant development effort compared to
  leveraging Three.js ecosystem

### Flutter Desktop

**Architecture:**
Dart UI layer with platform channels bridging to Rust FEA solver compiled as dynamic
library. Skia rendering handles 2D UI panels and drawing views. 3D structural
visualization via custom OpenGL/Vulkan integration or embedded native 3D view with
GPU-accelerated rendering.

**Tech stack:**
Dart/Flutter with custom Canvas for 2D structural drawing views, dart:ffi for Rust
FEA library binding, custom GPU texture bridge for 3D model visualization, fl_chart
for analysis result plots, sqflite for codes database.

**Native module needs:**
Rust FEA solver as .dylib/.dll/.so with C FFI, platform-specific 3D rendering
integration (Metal/Vulkan/DX12), GPU texture sharing for structural visualization,
IFC parser native library.

**Bundle size:** ~70-95 MB
**Memory:** ~260-430 MB

**Pros:**
- Skia rendering provides smooth 2D structural drawing views with
  hardware-accelerated rendering for plan/elevation/section views
- Hot reload accelerates iteration on complex multi-panel structural analysis UI
  with property editors and result browsers
- Single Dart codebase for UI panels, code checking dialogs, and analysis
  configuration across platforms
- Material Design components suitable for information-dense engineering interface
  layouts with tables and data grids

**Cons:**
- Flutter lacks native 3D rendering — structural model visualization requires
  complex platform-specific bridge or embedded native view
- Dart FFI overhead significant for transferring large FEA result arrays (nodal
  displacements, element stresses) from Rust solver
- No existing Flutter packages for structural engineering — member selection, code
  checking, and section property tools must be built from scratch
- 3D interaction patterns (orbit, pan, zoom on structural model) difficult to
  implement with Flutter's gesture system alone

### Swift/SwiftUI (macOS)

**Architecture:**
Native SwiftUI application with Metal compute shaders for GPU-accelerated FEA sparse
linear algebra and Metal rendering for 3D structural model visualization. SceneKit
for 3D scene management with custom Metal shaders for stress contour rendering. Rust
FEA core linked as static library.

**Tech stack:**
SwiftUI for UI, SceneKit for 3D structural visualization, Metal compute for FEA
solver, custom Metal shaders for stress contour rendering, RealityKit for AR
structural review (optional), Swift-Rust bridge via C FFI, GRDB.swift for SQLite.

**Native module needs:**
Metal compute shader compilation for sparse matrix kernels (SpMV, preconditioners),
Rust FEA library as static .a archive, SceneKit for 3D rendering, IFC parser via
C library binding.

**Bundle size:** ~28-45 MB
**Memory:** ~110-200 MB

**Pros:**
- Metal compute on Apple Silicon provides exceptional sparse matrix throughput —
  unified memory architecture enables zero-copy access between FEA solver buffers
  and visualization textures for instant result display
- SceneKit provides production-ready 3D scene management for structural model
  visualization with built-in LOD, shadows, and picking
- Native macOS interaction patterns and Retina rendering deliver professional-grade
  engineering application UX
- Optional RealityKit/ARKit integration enables augmented reality structural review
  on iPad companion for site inspections

**Cons:**
- macOS-only excludes Windows users who dominate the structural engineering market
  (Revit, ETABS, SAP2000 are Windows-centric)
- Metal-specific compute shaders not portable — requires reimplementation for
  cross-platform GPU backends
- Smaller Swift ecosystem for structural engineering formats (IFC, SDNF, CIS/2)
  compared to C++/.NET
- No Windows support is a dealbreaker for most structural engineering firms
  standardized on Windows workstations

### Kotlin Multiplatform

**Architecture:**
Compose Multiplatform UI with shared Kotlin business logic for model management,
code checking, and analysis orchestration. Rust FEA solver accessed via JNI. 3D
visualization via LWJGL/OpenGL bindings. Building codes engine in shared Kotlin
with platform-specific renderers.

**Tech stack:**
Kotlin/Compose Multiplatform for UI, JNI bridge to Rust FEA solver, LWJGL for
OpenGL/Vulkan 3D rendering, SQLDelight for codes and sections database, Kotlin
coroutines for async analysis management, Compose for desktop panels.

**Native module needs:**
Rust FEA solver with JNI-compatible C FFI, LWJGL native binaries for 3D rendering,
platform-specific OpenGL/Vulkan drivers, IFC parser via JNI, SQLite native library.

**Bundle size:** ~95-140 MB
**Memory:** ~310-530 MB

**Pros:**
- Kotlin coroutines provide structured async orchestration of long-running FEA
  analyses with progress reporting, cancellation, and parallel load case execution
- JVM ecosystem provides access to Apache Commons Math for supplementary numerical
  operations and matrix utilities
- LWJGL offers direct OpenGL/Vulkan access for 3D structural visualization with
  hardware-accelerated rendering
- Shared building code checking logic across platforms eliminates per-platform code
  verification effort

**Cons:**
- JVM memory overhead (310-530 MB) severely limits available resources for large
  structural models with million-DOF systems
- JNI bridge adds latency and complexity for transferring large sparse matrix data
  and FEA result arrays between Rust solver and visualization
- LWJGL/OpenGL 3D rendering less capable than Three.js ecosystem or native
  Metal/SceneKit for structural model visualization features
- Garbage collection pauses cause visible stutters during animated mode shape
  visualization and deformed shape rendering

---

## Comparison Matrix

| Dimension | Electron | Tauri | Flutter | Swift | Kotlin MP |
|-----------|----------|-------|---------|-------|-----------|
| Dev effort (weeks) | 15 | 14 | 19 | 12 | 21 |
| Bundle size | 350 MB | 50 MB | 82 MB | 36 MB | 118 MB |
| Memory usage | 555 MB | 190 MB | 345 MB | 155 MB | 420 MB |
| Startup time | 3.4s | 0.9s | 2.0s | 0.5s | 3.1s |
| Native feel | 5/10 | 7/10 | 6/10 | 10/10 | 6/10 |
| Offline capability | 8/10 | 9/10 | 8/10 | 9/10 | 8/10 |
| GPU/compute access | 5/10 | 9/10 | 5/10 | 10/10 | 5/10 |
| Cross-platform | 9/10 | 8/10 | 8/10 | 2/10 | 7/10 |
| Best fit score | 6/10 | 9/10 | 5/10 | 7/10 | 4/10 |

---

## Recommended Framework

**Tauri** is the best fit for StructAI Desktop, with Swift/SwiftUI as a viable
alternative for macOS-only deployment.

**Rationale:**
Structural FEA demands maximum GPU compute throughput for sparse linear algebra on
systems with millions of degrees of freedom — Tauri's Rust-native backend with direct
wgpu compute shader access provides zero-overhead GPU acceleration for both FEA
solving and AI-driven topology optimization. The minimal memory footprint (190 MB vs
555 MB for Electron) is critical for handling large BIM models alongside active FEA
solutions in memory. Cross-platform support is essential since the structural
engineering market is predominantly Windows-based with growing macOS adoption, and
Tauri's Windows/macOS/Linux coverage ensures maximum market reach.

**Runner-up:**
Swift/SwiftUI for a macOS-focused product targeting architects and structural
consultants in the Apple ecosystem, where Metal compute provides exceptional FEA
performance on Apple Silicon and SceneKit offers production-ready 3D structural
visualization with minimal development effort.

---

## Monetization (Desktop)

- **Freemium analysis tier:** Free access to basic frame analysis (up to 100 members,
  linear static) with limited code checking; paid Professional tier ($79/month)
  unlocks unlimited model size, dynamic analysis, nonlinear analysis, and full code
  checking suite
- **Perpetual license:** One-time purchase ($1,499) for core structural analysis with
  annual maintenance subscription ($399/year) for solver updates, new code editions,
  and feature additions
- **Enterprise license:** Annual subscription ($149/month per seat) for engineering
  firms with concurrent licensing, shared model libraries, standardized analysis
  templates, and firm-wide code checking configurations
- **AI optimization add-on:** Premium module ($39/month) providing GPU-accelerated
  topology optimization, AI-suggested member sizing, and generative structural design
  with parametric exploration
- **Code module expansion packs:** Regional building code modules ($19/month each)
  for jurisdictions beyond the base package — seismic provisions (ASCE 7 Chapter 12),
  wind engineering (ASCE 7 Chapter 26-31), timber design (NDS), masonry (TMS 402),
  and international codes (Eurocode, AS/NZS)

---

## MVP Scope (Desktop)

**Core MVP (8 weeks):**

| Week | Milestone |
|------|-----------|
| 1-2 | Rust FEA solver engine with wgpu compute shaders for 2D/3D beam-column elements (linear static analysis, sparse matrix assembly and solution), Tauri application shell with project management, SQLite schema for structural models and section databases, and AISC steel section property database |
| 3-4 | Interactive 3D structural model editor with node/member creation, support assignment, and load application, GPU-accelerated deformed shape visualization with displacement magnification, and basic load combination generator per ASCE 7 load cases |
| 5-6 | Analysis result visualization (moment diagrams, shear diagrams, axial force diagrams, deflected shape), steel member code checking per AISC 360 (tension, compression, flexure, combined forces), and demand/capacity ratio reporting with utilization color mapping on 3D model |
| 7 | Modal analysis with animated mode shape visualization, basic AI-assisted member sizing suggesting optimal section selections based on utilization ratios, and DXF import for geometry extraction from CAD drawings |
| 8 | End-to-end validation of analysis results against SAP2000/ETABS reference solutions, performance benchmarking of GPU solver throughput for various model sizes, installer packaging for Windows/macOS/Linux, and beta release targeting small structural engineering firms |

**Post-MVP:**
Shell and plate elements for slab/wall analysis, nonlinear analysis (material and
geometric), response spectrum and time-history seismic analysis, concrete design per
ACI 318, connection design module, IFC/BIM model import, progressive collapse
analysis, GPU-accelerated topology optimization, wind load generation from
computational wind engineering, foundation design module, automated report generation,
cloud sync for team collaboration, and integration with Revit via IFC round-trip.
