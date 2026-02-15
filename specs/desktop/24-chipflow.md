# ChipFlow Desktop — 5-Framework Comparison

## Desktop Rationale

- **GPU-Accelerated Layout Rendering:** Chip designs at advanced nodes contain billions of polygons across dozens of metal layers. Desktop apps can leverage the local GPU via Vulkan/Metal to render GDS-II layouts at interactive framerates, enabling smooth pan/zoom across full-chip views that would be impossible to stream from a server.

- **Local SPICE and Verilog Simulation:** Analog circuit simulation (SPICE) and digital RTL simulation (Verilog/SystemVerilog) are compute-intensive processes that benefit enormously from direct access to local CPU cores and memory. Desktop execution eliminates network latency between simulation steps, which is critical for iterative design-simulate-debug cycles that can involve thousands of runs.

- **Massive Design File Handling:** GDS-II and OASIS files for modern chips routinely exceed 10-50 GB. A desktop application can memory-map these files directly, enabling random access to hierarchical cell structures without downloading entire datasets. Local storage I/O is orders of magnitude faster than cloud streaming for these workloads.

- **Precise Input for Layout Editing:** Physical layout editing requires sub-nanometer precision with snapping to manufacturing grids. Desktop applications provide pixel-perfect cursor tracking, hardware-accelerated rubber-banding, and low-latency input response that are essential for productive polygon editing across dense layout regions.

- **Compute-Intensive DRC/LVS Checks:** Design Rule Checking and Layout vs. Schematic verification involve geometric computations over billions of edges. Running these locally on multi-core CPUs with large RAM avoids cloud compute costs and keeps proprietary design IP on the engineer's machine, addressing critical semiconductor IP security requirements.

---

## Desktop-Specific Features

- GPU-accelerated polygon rendering for GDS-II/OASIS layout visualization with per-layer color/pattern/transparency controls
- Integrated SPICE simulation engine with local compute scheduling across available CPU cores
- Verilog/SystemVerilog RTL simulation with waveform viewer and VCD/FSDB file support
- Memory-mapped file access for multi-gigabyte design databases with hierarchical cell browsing
- DRC engine with GPU-accelerated geometric boolean operations and edge-pair checking
- LVS extraction and comparison with netlist cross-probing between schematic and layout views
- Parasitic extraction (PEX) with local RC computation and back-annotation to simulation
- Technology file (techfile/DRM) management with layer mapping, design rules, and PDK integration
- USB/JTAG interface support for direct communication with FPGA prototyping boards
- Local PDK library management with version-controlled standard cell and IP block catalogs

---

## Shared Packages

### core (Rust)

- **geometry-engine:** Computational geometry library for polygon boolean operations (union, intersection, difference), edge clipping, and DRC spacing checks using sweep-line algorithms optimized for Manhattan and 45-degree geometries
- **spice-solver:** Sparse matrix solver for modified nodal analysis (MNA) with Newton-Raphson iteration, supporting DC, AC, transient, and Monte Carlo analysis modes
- **verilog-sim:** Event-driven digital simulation kernel supporting 4-state logic (0/1/X/Z), hierarchical module instantiation, and timing annotation from SDF files
- **gds-parser:** Streaming GDS-II and OASIS file parser/writer with hierarchical cell reference resolution, spatial indexing (R-tree) for viewport queries, and coordinate transformation support
- **drc-engine:** Design rule checking engine with GPU-acceleratable edge-pair generation, width/spacing/enclosure rule evaluation, and violation marker output
- **netlist-extract:** Layout-to-netlist extraction engine that identifies transistors, resistors, and capacitors from polygon geometries and layer interactions

### api-client (Rust)

- Foundry PDK download and license validation with encrypted credential storage
- Cloud simulation job submission for overflow compute (large Monte Carlo runs)
- Design team collaboration API for cell library sharing and design review comments
- EDA tool interop endpoints for importing/exporting to Cadence, Synopsys, and Mentor formats
- Telemetry and crash reporting with anonymized usage metrics

### data (SQLite)

- Design project database with hierarchical cell metadata, layer definitions, and technology parameters
- Simulation result storage with waveform indexing for fast time-range queries across millions of data points
- DRC/LVS violation database with spatial indexing for cross-probing between violation list and layout view
- User preferences, workspace layouts, and recently opened design history

---

## Framework Implementations

### Electron

**Architecture:** Chromium-based renderer for UI panels (cell browser, property editor, console) with a native C++/Rust addon for the layout canvas using OpenGL/Vulkan via offscreen rendering piped to a `<canvas>` element. SPICE and Verilog simulation engines run in worker threads spawned from the main Node.js process.

**Tech stack:** React + TypeScript for panel UI, WebGL2/WebGPU for layout rendering, N-API native modules for geometry engine and simulation cores, Protocol Buffers for IPC between renderer and native modules.

**Native module needs:** Heavy — requires compiled Rust/C++ modules for geometry engine, SPICE solver, GDS parser, DRC engine. Native GPU access through N-API bindings to Vulkan/Metal. File I/O through Node.js fs with memory-mapped file support via native addon.

**Bundle size:** ~280-350 MB (Chromium runtime + native simulation libraries + PDK templates + sample designs)

**Memory:** ~400-800 MB (Chromium overhead + layout geometry cache + simulation data + waveform buffers)

**Pros:**
- Rich ecosystem for building complex IDE-like panel layouts with docking, tabs, and split views
- WebGPU support emerging for compute shaders usable in DRC acceleration
- Extensive charting libraries for waveform visualization (Plotly, D3)
- Large developer pool familiar with web technologies

**Cons:**
- Chromium memory overhead is significant when combined with multi-GB design files
- WebGL/WebGPU abstraction adds latency compared to direct Vulkan/Metal for layout rendering
- IPC overhead between JavaScript UI and native simulation engines creates bottleneck during interactive simulation
- Bundle size bloated by Chromium for what is fundamentally a compute/graphics application

### Tauri

**Architecture:** System webview for UI panels with Rust backend handling all compute-intensive operations. Layout rendering via wgpu (Rust-native Vulkan/Metal/DX12 abstraction) rendered to a dedicated native window or composited into the webview. Simulation engines run as Rust async tasks with progress streaming to the frontend.

**Tech stack:** SolidJS or Svelte frontend for panel UI, wgpu for GPU-accelerated layout rendering and DRC compute shaders, Tauri commands for IPC, SQLite via rusqlite for design database, tokio for async simulation scheduling.

**Native module needs:** Minimal external — all compute is native Rust. wgpu provides GPU access without additional bindings. GDS parser, geometry engine, SPICE solver, and DRC engine are all Rust crates linked directly into the Tauri binary.

**Bundle size:** ~45-70 MB (system webview + Rust simulation binaries + PDK templates)

**Memory:** ~150-350 MB (lightweight webview + layout geometry in Rust heap + simulation data + GPU buffers managed by wgpu)

**Pros:**
- Direct GPU access via wgpu enables optimal layout rendering and DRC compute shader performance
- Rust memory safety critical for handling complex pointer-heavy geometry data structures
- Minimal memory overhead leaves more RAM for design data and simulation
- Single language (Rust) for all compute code — geometry, simulation, file I/O, GPU shaders

**Cons:**
- Webview rendering for panels less capable than Chromium for complex IDE layouts
- Smaller ecosystem for dockable panel systems and IDE-style UI widgets
- wgpu still maturing compared to direct Vulkan/Metal for advanced rendering features
- Webview inconsistencies across platforms may affect layout property editors

### Flutter Desktop

**Architecture:** Skia-rendered UI for all panels and controls with platform channels to Rust/C++ backend for simulation engines. Layout canvas rendered via custom painter using Flutter's Skia backend or via texture bridge to a separate Vulkan/Metal rendering context. FFI calls to native compute libraries.

**Tech stack:** Dart for UI logic, Flutter custom painters for layout visualization, dart:ffi for calling Rust simulation libraries, Isolates for background simulation tasks, platform channels for GPU resource sharing.

**Native module needs:** Significant — requires FFI bridge to Rust/C++ simulation engines, geometry libraries, and GDS parser. GPU access for layout rendering either through Flutter's Skia pipeline or via native texture interop. Memory management across Dart GC and native heap requires careful design.

**Bundle size:** ~80-110 MB (Flutter engine + Skia + native simulation libraries + PDK templates)

**Memory:** ~250-500 MB (Flutter/Skia overhead + Dart heap + native simulation data + GPU buffers)

**Pros:**
- Consistent pixel-perfect rendering across platforms for UI panels and property editors
- Strong animation framework useful for simulation playback and interactive parameter sweeps
- Custom painter API flexible enough for 2D layout rendering at moderate complexity
- Good support for complex form-based UIs needed in design parameter entry

**Cons:**
- Skia not optimized for rendering millions of polygons typical in chip layout views
- FFI overhead for every simulation call adds friction to tight design-simulate loops
- Dart's garbage collector can cause frame drops during heavy layout navigation
- Limited ecosystem for EDA-specific visualization (waveform viewers, schematic editors)

### Swift/SwiftUI (macOS)

**Architecture:** Native macOS application using AppKit/SwiftUI for UI panels with Metal for GPU-accelerated layout rendering and compute shaders for DRC. Simulation engines as Swift packages wrapping C/Rust libraries. Core Data or SQLite for design database. Grand Central Dispatch for parallel simulation scheduling.

**Tech stack:** SwiftUI for panel UI, Metal for layout rendering and DRC compute, Swift Package Manager for dependency management, Metal Performance Shaders for matrix operations in SPICE solver, Accelerate framework for numerical computing.

**Native module needs:** None for macOS — full native access to Metal GPU, file system, and hardware. Rust simulation libraries linked as static libraries via C-compatible FFI. Metal shaders written in Metal Shading Language for layout rendering and DRC computation.

**Bundle size:** ~30-50 MB (native binary + simulation libraries + Metal shader archives + PDK templates)

**Memory:** ~100-250 MB (minimal runtime overhead + layout geometry + Metal GPU buffers + simulation data)

**Pros:**
- Direct Metal access provides best-in-class GPU performance on Apple Silicon for layout rendering and DRC
- Apple Silicon unified memory architecture eliminates CPU-GPU data transfer overhead for geometry data
- Native macOS feel with proper menu bar, keyboard shortcuts, and trackpad gestures expected by engineers
- Metal compute shaders enable GPU-accelerated DRC that can be 10-100x faster than CPU-only approaches

**Cons:**
- macOS-only — excludes Linux users who represent a significant portion of the EDA engineering community
- SwiftUI still maturing for complex IDE-style layouts with dockable panels and split views
- Smaller ecosystem for EDA-specific libraries compared to C++/Python EDA tooling
- No cross-platform path without complete rewrite for Windows/Linux

### Kotlin Multiplatform

**Architecture:** Compose Multiplatform for UI panels with platform-specific rendering backends for layout visualization. JVM-based simulation engines on desktop with JNI calls to native Rust/C++ libraries for performance-critical geometry and SPICE operations. Skia via Compose for basic rendering, OpenGL/Vulkan via LWJGL for layout canvas.

**Tech stack:** Kotlin + Compose Multiplatform for UI, LWJGL for OpenGL/Vulkan GPU access, JNI for native library integration, Kotlin coroutines for async simulation, Exposed (Kotlin SQL) or SQLDelight for design database.

**Native module needs:** Moderate — JNI wrappers for Rust geometry engine, SPICE solver, and GDS parser. GPU access through LWJGL (Vulkan/OpenGL bindings for JVM). Memory management must bridge JVM garbage collector and native heap for large design data structures.

**Bundle size:** ~90-130 MB (JVM runtime + Compose/Skia + native libraries + LWJGL + PDK templates)

**Memory:** ~300-600 MB (JVM heap + Compose/Skia rendering + native simulation data + GPU buffers via LWJGL)

**Pros:**
- JVM ecosystem provides mature libraries for numerical computing (Apache Commons Math, EJML)
- Kotlin coroutines excellent for managing concurrent simulation jobs with structured cancellation
- Compose Multiplatform enables code sharing across macOS, Windows, and Linux
- Strong type system catches design parameter errors at compile time

**Cons:**
- JVM memory overhead competes with design data for available RAM
- JNI bridge adds complexity and latency for GPU-intensive layout rendering
- JVM garbage collection pauses can cause visible stutters during layout navigation
- LWJGL GPU access less ergonomic than native Vulkan/Metal for compute shaders

---

## Comparison Matrix

| Dimension | Electron | Tauri | Flutter | Swift | Kotlin MP |
|-----------|----------|-------|---------|-------|-----------|
| Dev effort (weeks) | 20 | 18 | 22 | 16 | 24 |
| Bundle size | 320 MB | 55 MB | 95 MB | 40 MB | 110 MB |
| Memory usage | 600 MB | 250 MB | 380 MB | 180 MB | 450 MB |
| Startup time | 3.2s | 1.1s | 2.0s | 0.6s | 2.8s |
| Native feel | 5/10 | 7/10 | 6/10 | 10/10 | 6/10 |
| Offline capability | 8/10 | 9/10 | 8/10 | 9/10 | 8/10 |
| GPU/compute access | 5/10 | 9/10 | 5/10 | 10/10 | 6/10 |
| Cross-platform | 9/10 | 9/10 | 8/10 | 2/10 | 8/10 |
| Best fit score | 6/10 | 9/10 | 5/10 | 7/10 | 5/10 |

---

## Recommended Framework

**Tauri** is the best fit for ChipFlow.

**Rationale:** ChipFlow's core value proposition — interactive chip layout visualization with billions of polygons, GPU-accelerated DRC, and local SPICE simulation — demands direct GPU access and minimal memory overhead. Tauri's Rust backend provides wgpu for GPU-accelerated rendering and compute shaders, native-speed geometry operations, and memory-safe handling of massive GDS-II files, all while keeping the bundle small and leaving maximum RAM for design data. The cross-platform support is critical since EDA engineers work across macOS, Linux, and Windows.

**Runner-up:** Swift/SwiftUI offers superior GPU performance via Metal on macOS and the smallest footprint, but the macOS-only limitation is disqualifying for an EDA tool where Linux workstation usage is prevalent. If targeting Apple Silicon design teams exclusively, Swift would be the top choice.

---

## Monetization (Desktop)

- **Per-seat licensing** with annual subscriptions ($5,000-15,000/year) tiered by feature set (Layout Viewer, Layout Editor, Full Suite with DRC/LVS/PEX)
- **Node-locked compute licenses** for local simulation with optional floating license server for team environments
- **PDK marketplace** with foundry-certified process design kits available as in-app purchases ($500-2,000 per node)
- **Cloud burst compute** metered billing for large Monte Carlo simulations and full-chip DRC runs that exceed local compute capacity ($0.50-2.00/core-hour)
- **Enterprise site licenses** with priority support, custom DRC rule deck development, and on-premise license server deployment

---

## MVP Scope (Desktop)

**Core MVP (8 weeks):**

| Week | Milestone |
|------|-----------|
| 1-2 | GDS-II parser with hierarchical cell browsing, basic layout rendering via wgpu with per-layer visibility controls, pan/zoom/rotate navigation |
| 3-4 | Layout editing tools (polygon draw, move, copy, stretch) with grid snapping, undo/redo stack, basic DRC engine for width/spacing rules |
| 5-6 | SPICE netlist import/export, basic DC/transient simulation engine with sparse matrix solver, waveform viewer with time-axis zoom and signal selection |
| 7 | Technology file parser for layer definitions and design rules, DRC violation browser with cross-probing to layout, LVS comparison for simple circuits |
| 8 | Testing, performance optimization for large designs (1M+ polygons), installer packaging for macOS/Windows/Linux, beta documentation |

**Post-MVP:** Verilog/SystemVerilog RTL simulation, parasitic extraction (PEX), Monte Carlo simulation, cloud burst compute integration, PDK marketplace, schematic editor with bi-directional cross-probing, electromagnetic simulation for RF designs, formal verification integration, team collaboration features with design versioning.
