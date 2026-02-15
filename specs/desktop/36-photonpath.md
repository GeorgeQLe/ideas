# PhotonPath Desktop — 5-Framework Comparison

## Desktop Rationale

- **GPU-Accelerated FDTD Simulation:**
  Finite-difference time-domain simulation of electromagnetic wave propagation through
  photonic structures requires updating millions of field components per timestep.
  Desktop GPU compute shaders can parallelize these spatial updates across thousands
  of GPU cores, enabling real-time FDTD simulation of waveguide geometries that would
  require minutes on CPU-only cloud instances.

- **Real-Time Optical Field Visualization:**
  Visualizing electromagnetic field distributions (E-field, H-field, Poynting vector)
  across 2D and 3D photonic structures requires continuous GPU rendering of dense
  field data at interactive framerates. Desktop GPU rendering pipelines provide direct
  access to texture memory for mapping simulation data to color-mapped field
  visualizations without network streaming.

- **Large Simulation Meshes:**
  High-resolution FDTD and eigenmode simulations of photonic crystal structures, ring
  resonators, and gratings generate multi-gigabyte spatial meshes with sub-wavelength
  grid resolution. Desktop memory-mapped file access and GPU buffer management enable
  interactive exploration of simulation domains that exceed cloud browser memory limits.

- **Photonic Layout Editor:**
  Precision photonic integrated circuit (PIC) layout design requires sub-nanometer
  grid snapping, parametric component libraries, and GPU-accelerated rendering of
  complex waveguide routing with thousands of geometric elements. Desktop native
  rendering eliminates the latency and precision limitations of browser-based CAD tools.

- **Offline Design and Simulation:**
  Photonics engineers in cleanroom facilities, fabrication environments, and
  defense/classified settings require fully offline design and simulation capabilities.
  Desktop-native FDTD and mode solvers ensure uninterrupted design workflows without
  dependency on cloud connectivity.

---

## Desktop-Specific Features

- GPU-accelerated 2D/3D FDTD solver using wgpu compute shaders with perfectly matched
  layer (PML) absorbing boundary conditions and dispersive material models
- Real-time electromagnetic field visualization with GPU-rendered E-field, H-field, and
  Poynting vector overlays on photonic structure geometry at 60fps
- Eigenmode solver for waveguide cross-section analysis with GPU-accelerated sparse
  matrix eigenvalue computation for finding guided modes and effective indices
- Interactive photonic layout editor with parametric component library (waveguides,
  bends, couplers, ring resonators, MMIs, gratings) and sub-nanometer precision
  placement
- S-parameter extraction engine computing transmission/reflection spectra via FDTD
  with frequency-domain monitors and GPU-accelerated Fourier transforms
- Band structure calculator for photonic crystals using GPU-parallel plane-wave
  expansion (PWE) method with interactive Brillouin zone path specification
- Material database with wavelength-dependent refractive index data (Sellmeier,
  Drude-Lorentz models) for silicon, silicon nitride, III-V semiconductors, and
  plasmonic metals
- Near-to-far-field transformation for computing radiation patterns of photonic
  antennas and grating couplers with GPU-accelerated angular integration
- GDS-II import/export for photonic mask layout compatibility with foundry process
  design kits (PDKs) and tapeout workflows
- Mode overlap integral calculator for evaluating coupling efficiency between
  waveguide modes and fiber modes with GPU-accelerated 2D integration

---

## Shared Packages

### core (Rust)

- FDTD simulation engine implementing Yee grid discretization with PML boundaries,
  dispersive material models (Drude, Lorentz, Debye), and wgpu compute shader dispatch
  for massively parallel field updates across GPU workgroups
- Eigenmode solver using iterative sparse eigenvalue methods (Arnoldi, Lanczos) with
  GPU-accelerated matrix-vector products for computing waveguide modes and photonic
  crystal band structures
- Photonic component library implementing parameterized models of standard PIC elements
  (Euler bends, directional couplers, ring resonators, MMI splitters) with analytical
  and numerical design equations
- S-parameter extraction module performing discrete Fourier transforms on time-domain
  field monitors with GPU-accelerated spectral analysis for multi-port device
  characterization
- Geometry engine handling boolean operations on photonic layout polygons,
  grid-conformal meshing for FDTD, and GDS-II file parsing/generation with
  hierarchical cell reference support
- Material property database engine with Sellmeier equation fitting, Kramers-Kronig
  consistency validation, and temperature-dependent index interpolation

### api-client (Rust)

- Cloud sync client for sharing photonic designs and simulation results across
  distributed design teams with version-controlled layout management
- Foundry PDK connector for downloading and updating process design kits with layer
  stack definitions, design rule constraints, and calibrated material models
- Telemetry and crash reporting with anonymized simulation performance metrics and
  mesh complexity statistics
- License validation and feature-flag management supporting academic, startup, and
  enterprise tier differentiation
- Integration endpoints for importing designs from Lumerical, COMSOL, and Synopsys
  RSoft via standardized exchange formats

### data (SQLite)

- Simulation results store with compressed spatial field data snapshots and spectral
  response data, enabling instant replay of completed FDTD simulations and S-parameter
  sweeps
- Component library database storing parameterized photonic elements with geometric
  definitions, design rules, and cached simulation results for rapid design reuse
- Material properties database with indexed access to wavelength-dependent refractive
  indices for hundreds of optical materials with literature references and measurement
  uncertainty data
- Project workspace persistence tracking layout designs, simulation configurations,
  visualization viewports, and parameter sweep states across design sessions

---

## Framework Implementations

### Electron

**Architecture:**
Multi-process Chromium architecture with main process orchestrating Rust FDTD solver
via N-API bindings. Renderer processes host React-based photonic layout editor using
Canvas/WebGL for geometry rendering and WebGL2 for electromagnetic field visualization.
Heavy FDTD computation dispatched to Rust sidecar process with wgpu.

**Tech stack:**
React 18 + TypeScript, Fabric.js or custom Canvas for layout editor, WebGL2 + custom
shaders for field visualization, Three.js for 3D structure rendering, napi-rs for Rust
FFI, better-sqlite3 for database, electron-builder for packaging.

**Native module needs:**
Rust FDTD solver as native Node addon via napi-rs, wgpu compute runtime for
GPU-accelerated field updates, SQLite native binding, GDS-II parser native module.

**Bundle size:** ~300-380 MB
**Memory:** ~420-680 MB

**Pros:**
- WebGL2 shader ecosystem enables rapid development of custom electromagnetic field
  visualization with color mapping and vector field rendering
- Canvas API provides mature foundation for building interactive photonic layout editor
  with precise geometric manipulation
- Large pool of JavaScript/WebGL developers available for visualization-heavy
  rendering development
- Cross-platform deployment well-tested for engineering desktop applications

**Cons:**
- Chromium memory overhead severely limits available GPU memory for large FDTD
  simulation meshes (multi-million cell grids)
- WebGL2 compute capabilities absent — all FDTD computation must route through Rust
  sidecar, adding IPC latency for field data streaming
- Canvas-based layout editor precision limited by browser sub-pixel rendering
  inconsistencies across platforms
- Bundle size (300+ MB) excessive for academic and startup users with limited
  disk resources

### Tauri

**Architecture:**
Rust backend hosting FDTD solver, eigenmode solver, and geometry engine directly in
the application process. wgpu compute shaders execute field updates natively. Frontend
via system WebView with Canvas-based layout editor and WebGL for field visualization.
Shared GPU memory between compute and render passes eliminates data transfer overhead.

**Tech stack:**
Rust backend with wgpu for compute and rendering, Svelte or Solid.js frontend, HTML5
Canvas for layout editor, WebGL for field visualization in WebView, rusqlite for
database, GDS-II parser in Rust.

**Native module needs:**
wgpu runtime for GPU compute (FDTD solver) and rendering (field visualization),
system WebView dependency, Vulkan/Metal/DX12 drivers for compute shader execution.

**Bundle size:** ~35-55 MB
**Memory:** ~130-240 MB

**Pros:**
- Rust-native FDTD solver with direct wgpu compute dispatch eliminates FFI overhead —
  field updates execute on GPU with zero-copy buffer access between compute and
  render passes
- Minimal memory footprint maximizes GPU buffer allocation for large FDTD simulation
  meshes (10M+ Yee cells)
- wgpu abstraction provides portable GPU compute across Vulkan/Metal/DX12, ensuring
  consistent simulation performance across platforms
- Sub-50 MB bundle size enables easy distribution to academic labs and startup teams

**Cons:**
- System WebView Canvas rendering may lack precision needed for sub-nanometer photonic
  layout snapping on some platforms
- Limited Rust ecosystem for photonic-specific algorithms — GDS-II parser and PDK
  tooling must be built or ported from C++/Python libraries
- WebView debugging less mature than Chromium DevTools for troubleshooting complex
  Canvas-based layout editor interactions
- Advanced 3D visualization (volumetric field rendering) challenging to implement
  efficiently through WebView layer

### Flutter Desktop

**Architecture:**
Dart UI layer with custom Canvas-based photonic layout editor. Platform channels
bridge to Rust FDTD solver compiled as dynamic library. Skia rendering handles layout
geometry; field visualization via custom texture bridge sharing GPU simulation output
with Flutter rendering pipeline.

**Tech stack:**
Dart/Flutter with custom Canvas for layout editor, dart:ffi for Rust FDTD library
binding, custom GPU texture sharing for field visualization, fl_chart for spectral
response plots, sqflite for database.

**Native module needs:**
Rust FDTD solver as .dylib/.dll/.so with C FFI, platform-specific GPU texture sharing
(Metal/Vulkan/DX12), Skia shader customization for field colormap rendering.

**Bundle size:** ~65-95 MB
**Memory:** ~260-420 MB

**Pros:**
- Skia rendering engine provides hardware-accelerated vector graphics ideal for
  precise photonic layout geometry with smooth zoom/pan at any scale
- Custom Canvas API enables sub-pixel accurate rendering of waveguide geometries with
  consistent appearance across platforms
- Hot reload accelerates development of complex layout editor UI with multiple tool
  modes and property panels
- Single codebase for UI across Windows/macOS/Linux reduces platform-specific layout
  editor maintenance

**Cons:**
- Dart FFI marshalling overhead significant for streaming large FDTD field arrays
  (millions of float values) from Rust solver to visualization
- GPU texture sharing between Rust compute and Flutter Skia renderer requires complex
  platform-specific bridge implementation
- No existing Flutter packages for photonic design — GDS-II viewer, field
  visualization, and band structure plots must be built entirely from scratch
- Flutter text rendering precision may not meet requirements for dimension annotations
  and grid coordinate display in layout editor

### Swift/SwiftUI (macOS)

**Architecture:**
Native SwiftUI application with Metal compute shaders for FDTD field updates and
Metal rendering for electromagnetic field visualization. AppKit-based layout editor
using Core Graphics for precision geometry rendering. Rust solver linked as static
library for algorithm portability.

**Tech stack:**
SwiftUI for UI panels, AppKit/NSView for layout editor canvas, Metal compute for FDTD
solver, Metal rendering for field visualization, MetalKit for GPU resource management,
Swift-Rust bridge via C FFI, GRDB.swift for SQLite.

**Native module needs:**
Metal compute shader compilation for FDTD kernels (field update, PML, dispersion),
Rust solver library as static .a archive, Core Graphics for precision layout rendering.

**Bundle size:** ~25-42 MB
**Memory:** ~100-190 MB

**Pros:**
- Metal compute on Apple Silicon provides exceptional FDTD throughput — M-series
  unified memory enables zero-copy access between compute (field updates) and render
  (field visualization) passes
- Core Graphics delivers sub-pixel precision rendering essential for photonic layout
  editors with nanometer-scale features
- Native macOS CAD-like interaction patterns (gesture-based zoom, trackpad precision,
  Touch Bar controls) enhance layout editor UX
- Minimal resource footprint maximizes GPU memory allocation for large simulation
  meshes

**Cons:**
- macOS-only excludes Windows and Linux users who represent the majority of photonics
  engineering workstations
- Metal-specific compute shaders not portable — requires complete reimplementation for
  Windows/Linux GPU backends
- Limited Swift ecosystem for photonic and EDA tooling — GDS-II, PDK support must be
  built from scratch or bridged from C++
- No cross-platform path without maintaining separate codebases for each platform

### Kotlin Multiplatform

**Architecture:**
Compose Multiplatform UI with shared Kotlin business logic for design management and
simulation orchestration. Rust FDTD engine accessed via JNI. Layout editor built with
Compose Canvas API. GPU rendering via LWJGL/Vulkan bindings for field visualization.

**Tech stack:**
Kotlin/Compose Multiplatform for UI, JNI bridge to Rust FDTD library, Compose Canvas
for layout editor, LWJGL for OpenGL/Vulkan field rendering, SQLDelight for database,
Kotlin coroutines for async simulation management.

**Native module needs:**
Rust FDTD library with JNI-compatible C FFI, LWJGL native binaries for GPU rendering,
platform-specific Vulkan/OpenGL drivers, SQLite native library.

**Bundle size:** ~90-135 MB
**Memory:** ~300-520 MB

**Pros:**
- Kotlin coroutines provide structured concurrency for managing long-running FDTD
  simulations with progress reporting, cancellation, and result streaming
- Compose Canvas API supports custom rendering of photonic layouts with hardware
  acceleration across platforms
- JVM access to Apache Commons Math for supplementary numerical operations (FFT,
  matrix algebra) used in S-parameter extraction
- Type-safe sealed class hierarchies model complex photonic component type systems
  cleanly

**Cons:**
- JVM memory overhead (300-520 MB) severely limits available GPU memory for large FDTD
  simulation meshes
- JNI bridge adds latency and complexity for high-bandwidth field data streaming from
  Rust FDTD solver
- GPU compute via LWJGL/Vulkan significantly less ergonomic than native wgpu or Metal
  for FDTD compute shader development
- JVM garbage collection pauses cause visible stutters during real-time field
  visualization updates

---

## Comparison Matrix

| Dimension | Electron | Tauri | Flutter | Swift | Kotlin MP |
|-----------|----------|-------|---------|-------|-----------|
| Dev effort (weeks) | 16 | 14 | 18 | 12 | 20 |
| Bundle size | 340 MB | 45 MB | 80 MB | 33 MB | 112 MB |
| Memory usage | 550 MB | 185 MB | 340 MB | 145 MB | 410 MB |
| Startup time | 3.5s | 0.9s | 2.0s | 0.5s | 3.2s |
| Native feel | 5/10 | 7/10 | 6/10 | 10/10 | 6/10 |
| Offline capability | 8/10 | 9/10 | 8/10 | 9/10 | 8/10 |
| GPU/compute access | 5/10 | 9/10 | 5/10 | 10/10 | 5/10 |
| Cross-platform | 9/10 | 8/10 | 8/10 | 2/10 | 7/10 |
| Best fit score | 6/10 | 9/10 | 5/10 | 8/10 | 4/10 |

---

## Recommended Framework

**Tauri** is the best fit for PhotonPath Desktop, with Swift/SwiftUI as a strong
alternative for macOS-only deployment.

**Rationale:**
FDTD simulation is among the most GPU-compute-intensive workloads in photonics —
Tauri's Rust-native backend with direct wgpu compute shader dispatch provides the
zero-overhead GPU access essential for real-time field updates across millions of Yee
cells. The shared wgpu pipeline enables zero-copy compute-to-render transitions for
seamless field visualization, while the minimal 185 MB memory footprint maximizes GPU
buffer allocation for large simulation meshes. Cross-platform support is critical
since photonics engineers work across Windows, macOS, and Linux.

**Runner-up:**
Swift/SwiftUI for a macOS-focused product leveraging Metal's unified memory on Apple
Silicon, where the zero-copy compute-to-render pipeline provides the tightest possible
FDTD solver-to-visualization latency.

---

## Monetization (Desktop)

- **Freemium simulation tier:** Free access to 2D FDTD with limited mesh size
  (500x500 cells) and basic waveguide components; paid Pro tier ($69/month) unlocks
  3D FDTD, unlimited mesh sizes, eigenmode solver, and full component library
- **Perpetual academic license:** One-time purchase ($299) for university researchers
  with full simulation capabilities, academic-use restriction, and one year of
  updates; annual renewal ($99/year) for continued updates
- **Commercial license:** Annual subscription ($149/month) for photonics design teams
  in industry with full capabilities, commercial-use rights, foundry PDK integration,
  and priority technical support
- **Foundry PDK packages:** Licensed process design kits ($500-2,000/year each) for
  specific foundry processes (AIM Photonics, IMEC, GlobalFoundries) with calibrated
  material models and design rule decks
- **Enterprise floating license:** Concurrent-use licensing ($8,000-30,000/year) for
  photonics design houses with license server management, usage analytics, and
  dedicated support engineering

---

## MVP Scope (Desktop)

**Core MVP (8 weeks):**

| Week | Milestone |
|------|-----------|
| 1-2 | Rust 2D FDTD solver engine with wgpu compute shaders (TM/TE modes, PML boundaries, basic dispersive materials), Tauri application shell with project management, and SQLite schema for simulation parameters and material properties |
| 3-4 | Interactive photonic layout editor with basic waveguide drawing (straight, bend, taper), GPU-accelerated real-time E-field visualization with color-mapped overlay on structure geometry, and material property database with common photonic materials (Si, SiN, SiO2) |
| 5-6 | S-parameter extraction from FDTD monitors with transmission/reflection spectrum plots, parameterized component library (directional coupler, ring resonator, MMI), and design-to-simulation workflow with automatic meshing of layout geometry |
| 7 | GDS-II import/export for mask layout compatibility, basic eigenmode solver for waveguide cross-section analysis, and simulation parameter sweep engine with batch job management |
| 8 | End-to-end testing of FDTD accuracy against analytical solutions and Lumerical reference data, performance benchmarking of GPU compute throughput, installer packaging for Windows/macOS/Linux, and beta release preparation |

**Post-MVP:**
3D FDTD solver, photonic crystal band structure calculator, near-to-far-field
transformation, advanced dispersive material models (Drude, multi-Lorentz), foundry
PDK integration, design rule checking, mode overlap calculator, optimization engine
(adjoint method), cloud burst compute for large 3D simulations, team collaboration
with design version control, and integration with electronic-photonic co-simulation
tools.
