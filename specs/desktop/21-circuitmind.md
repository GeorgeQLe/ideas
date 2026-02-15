# CircuitMind Desktop — 5-Framework Comparison

## Desktop Rationale

CircuitMind (AI-assisted PCB/circuit design) is a deeply technical application where desktop is not just beneficial but essential:

- **GPU-accelerated rendering:** PCB layouts with hundreds of components, copper traces, and multiple layers require GPU-accelerated 2D/3D rendering. WebGL in a browser cannot match native Metal/Vulkan/DirectX performance for smooth panning, zooming, and layer toggling on complex boards.
- **Local SPICE simulation:** Circuit simulation (SPICE) is CPU-intensive and generates gigabytes of transient data. Running LTspice-class simulations locally avoids cloud round-trip latency and keeps proprietary designs on the user's machine.
- **Large schematic files:** Professional PCB designs produce files ranging from 50 MB to 2+ GB (Gerber files, drill files, BOM, 3D models). Desktop apps handle these without browser memory limits or upload bandwidth constraints.
- **Precise input (mouse/pen):** Trace routing, component placement, and schematic drawing demand sub-pixel precision with mouse and pen tablet input. Desktop apps access raw input events without browser event throttling.
- **Offline design capability:** Engineers often work in labs, fabs, or secure facilities without internet. Full offline capability for design, simulation, and DRC (design rule checking) is non-negotiable.

---

## Desktop-Specific Features

- **GPU-accelerated PCB viewer:** Hardware-accelerated rendering of multi-layer boards with real-time layer toggling and transparency
- **Local SPICE engine:** Embedded ngspice or custom solver for circuit simulation with progress reporting
- **Pen tablet support:** Pressure-sensitive input for freehand schematic sketching and trace routing
- **Multi-monitor layout:** Schematic on one screen, PCB layout on another, simulation waveforms on a third
- **3D board preview:** Real-time 3D rendering of populated PCB with component models (STEP files)
- **Gerber export:** Direct export to Gerber RS-274X, Excellon drill files, and pick-and-place files
- **Component library manager:** Local component database with footprints, symbols, 3D models, and parametric search
- **DRC engine:** Design rule checking with configurable rules (clearance, trace width, via size, impedance)
- **AI routing assistant:** AI-powered auto-routing suggestions using local ML inference for trace optimization
- **Version control integration:** Git-based design versioning with visual diff for schematics and layouts

---

## Shared Packages

### core (Rust)

- **Schematic engine:** Component placement, net connectivity, hierarchical sheet management, ERC (electrical rule check)
- **PCB layout engine:** Board outline, layer stack definition, footprint placement, copper zone fill, trace routing algorithms (A*, Lee's algorithm)
- **SPICE interface:** Netlist generation from schematic, ngspice process management, waveform data parsing (raw/CSV)
- **DRC validator:** Configurable design rules with spatial indexing (R-tree) for efficient clearance checks across thousands of pads/traces
- **Gerber generator:** RS-274X aperture generation, layer-by-layer plot output, Excellon drill file creation
- **AI inference:** ONNX Runtime for local ML models (auto-routing suggestions, component placement optimization)

### api-client (Rust)

- **Component search:** Query Octopart/Mouser/DigiKey APIs for component data, datasheets, and pricing
- **Cloud sync:** Push designs to CircuitMind cloud for team collaboration and review
- **AI enhancement:** Cloud-based LLM for schematic review suggestions and design optimization hints
- **Auth:** License key validation for professional features, OAuth2 for team access

### data (SQLite)

- **Schema:** projects, schematics, pcb_layouts, components (symbol, footprint, 3D_model, parameters JSON), nets, traces, design_rules, simulation_results, component_library
- **Spatial index:** R-tree virtual table for efficient spatial queries during DRC and component collision detection
- **Binary storage:** BLOB columns for component footprints, 3D models (compressed), and simulation waveform data
- **Project history:** Full undo/redo history stored as incremental snapshots for unlimited design rollback

---

## Framework Implementations

### Electron

**Architecture:** Main process manages file I/O, SPICE subprocess (ngspice), and project management. Renderer process uses WebGL2 (via Three.js/PixiJS) for schematic and PCB rendering. Web Workers for DRC and netlist generation. Separate renderer window for simulation waveforms.

**Tech stack:**
- Renderer: React 18, PixiJS (2D schematic/PCB), Three.js (3D preview), D3 (waveform plots), TailwindCSS
- Main: child_process (ngspice), better-sqlite3, chokidar
- GPU: WebGL2 via PixiJS/Three.js
- Compute: Web Workers for DRC, SharedArrayBuffer for cross-thread data

**Native module needs:** better-sqlite3, node-pty (ngspice process), sharp (image processing for component symbols)

**Bundle size:** ~220-300 MB (Chromium + 3D models + component library)

**Memory:** ~400-800 MB (higher with large boards, 3D preview, and simulation data loaded)

**Pros:**
- PixiJS/Three.js provide capable 2D/3D rendering with large community and documentation
- Web Workers enable parallel DRC without blocking the UI thread
- Existing open-source EDA web tools (EasyEDA, Flux) provide reference implementations
- Large ecosystem for data visualization (waveform plotting, BOM tables)

**Cons:**
- WebGL2 performance ceiling limits complex board rendering (1000+ component boards lag)
- No access to Metal/Vulkan — stuck with OpenGL ES 3.0 equivalent via WebGL2
- Memory overhead of Chromium leaves less RAM for simulation data and 3D models
- Pen tablet input events are throttled by Chromium's compositor
- Cannot match professional EDA tools (KiCad, Altium) in rendering fidelity

### Tauri

**Architecture:** Rust backend handles all compute: SPICE simulation, DRC validation, Gerber generation, AI inference. GPU rendering via wgpu (Rust WebGPU) in a custom render surface, or via WebGL2 in the webview for simpler rendering. Svelte frontend for UI panels, property editors, and component browser.

**Tech stack:**
- Frontend: Svelte 5, TailwindCSS (panels/UI), CodeMirror 6 (netlist editing)
- Backend: wgpu (GPU rendering), ngspice-rs or subprocess, rusqlite, rayon (parallel DRC), ort (AI inference)
- GPU: wgpu with compute shaders for copper zone fill calculation and 3D preview
- Simulation: ngspice via FFI or subprocess, custom waveform parser

**Plugin needs:** tauri-plugin-dialog, tauri-plugin-fs, tauri-plugin-shell (ngspice process)

**Bundle size:** ~25-45 MB (+ component library ~50 MB, downloaded separately)

**Memory:** ~80-200 MB (spikes to ~500 MB during simulation of complex circuits)

**Pros:**
- **wgpu provides native GPU access** — Metal on macOS, Vulkan on Linux, DX12 on Windows
- Rust's rayon enables parallel DRC across all CPU cores — 10x faster than Web Workers
- SPICE integration via ngspice-rs or subprocess is cleaner in Rust than Node.js
- Compute shaders (wgpu) for copper zone fill and impedance calculation
- R-tree spatial indexing in Rust (rstar crate) provides microsecond DRC lookups
- Small base bundle; component library can be lazy-downloaded

**Cons:**
- wgpu rendering in a separate surface alongside webview requires careful window management
- Building a full schematic editor UI in Svelte + wgpu is significant engineering effort
- Fewer reference implementations for EDA in Rust compared to C++/Qt
- Complex hybrid rendering architecture (webview for panels, wgpu for canvas)

### Flutter Desktop

**Architecture:** Flutter renders UI panels and property editors. Custom GPU rendering via flutter_gpu (experimental) or platform views embedding a native wgpu/Metal canvas for the schematic/PCB editor. Rust FFI for compute-heavy tasks.

**Tech stack:**
- UI: Flutter 3.x, Riverpod, custom widget for component palette
- GPU: CustomPainter (basic) or platform view embedding native GPU canvas
- Compute: flutter_rust_bridge -> Rust (DRC, SPICE, Gerber generation)
- Database: drift (SQLite)

**Plugin needs:** window_manager (multi-window), flutter_rust_bridge, custom platform view for GPU canvas

**Bundle size:** ~35-50 MB (+ component library)

**Memory:** ~150-300 MB

**Pros:**
- CustomPainter can handle simple schematics with hardware acceleration
- Hot reload excellent for iterating on component palette and property panel UIs
- Cross-platform from single codebase including potential tablet version for schematic review
- Skia backend provides decent 2D rendering performance

**Cons:**
- flutter_gpu is experimental — not production-ready for complex EDA rendering
- No native compute shader access — must bridge to Rust/native for GPU computation
- CustomPainter hits performance ceiling with 500+ components on screen
- Pen/stylus input handling in Flutter desktop is less precise than native
- Building a professional PCB editor in Flutter is uncharted territory

### Swift/SwiftUI (macOS)

**Architecture:** SwiftUI for panels and UI chrome. Metal-based custom renderer (MTKView) for schematic and PCB canvas. Metal compute shaders for DRC spatial queries and copper zone fill. Core Data or SwiftData for project persistence.

**Tech stack:**
- UI: SwiftUI (panels, inspectors), AppKit (toolbar, multi-window)
- GPU: Metal (MTKView for 2D/3D rendering, compute shaders)
- Simulation: ngspice via Process, or custom Swift SPICE solver
- Data: SwiftData
- AI: CoreML/MLX for local inference (auto-routing model)

**Bundle size:** ~15-25 MB (+ component library)

**Memory:** ~60-150 MB (Metal manages GPU memory separately from app heap)

**Pros:**
- **Metal provides the best GPU performance on macOS** — compute shaders, tile-based rendering, unified memory on Apple Silicon
- Apple Pencil support on iPad (future port) and precise trackpad/mouse input
- MTKView is the native pattern for GPU-accelerated editors (used by Pixelmator, Affinity Designer)
- Metal compute shaders for copper zone fill run 5-10x faster than CPU
- Multi-window support native in AppKit — schematic + PCB + waveforms on separate monitors
- Unified memory on Apple Silicon means GPU can access full system RAM for large boards
- Smallest app memory footprint — more RAM available for design data

**Cons:**
- macOS only — excludes Windows/Linux (many hardware engineers use Windows)
- Metal shader development has a steep learning curve
- Must build the entire EDA rendering pipeline from scratch in Metal
- Smaller community for CAD/EDA development in Swift vs. C++/Qt

### Kotlin Multiplatform

**Architecture:** Compose Desktop for UI panels. LWJGL (Lightweight Java Game Library) or custom JNI bridge to Vulkan/OpenGL for the PCB canvas. JVM-based SPICE wrapper. SQLDelight for persistence.

**Tech stack:**
- UI: Compose Desktop, Material 3 (panels), custom Canvas composable
- GPU: LWJGL (OpenGL/Vulkan via JNI), or Skia canvas for simpler rendering
- Simulation: JNI -> ngspice, or jspice (pure Java, limited)
- Database: SQLDelight
- HTTP: Ktor client

**Bundle size:** ~70-100 MB (+JRE + component library)

**Memory:** ~250-500 MB (JVM + GPU context + simulation data)

**Pros:**
- LWJGL provides access to OpenGL/Vulkan from JVM (used by Minecraft, LibGDX)
- Could share component library logic with an Android tablet companion app
- JVM has mature libraries for engineering calculations (Apache Commons Math)
- Compose Canvas provides basic 2D drawing for simple schematics

**Cons:**
- JVM adds 200+ MB memory overhead before any design data is loaded
- LWJGL + Compose interop is fragile and poorly documented
- JNI bridge to ngspice introduces crash risk and debugging complexity
- Compose Canvas is orders of magnitude slower than Metal/wgpu for complex boards
- JVM garbage collection pauses cause visible stutters during smooth pan/zoom
- Not competitive with any professional EDA tool in performance

---

## Comparison Matrix

| Dimension | Electron | Tauri | Flutter | Swift | Kotlin MP |
|-----------|----------|-------|---------|-------|-----------|
| Dev effort (weeks) | 10 | 12 | 14 | 12 | 14 |
| Bundle size | 260 MB | 35 MB | 45 MB | 20 MB | 85 MB |
| Memory usage | 600 MB | 140 MB | 220 MB | 100 MB | 380 MB |
| Startup time | 3.0s | 1.0s | 1.5s | 0.5s | 2.5s |
| Native feel | 4/10 | 7/10 | 5/10 | 10/10 | 3/10 |
| Offline capability | 7/10 | 9/10 | 7/10 | 9/10 | 7/10 |
| GPU/compute access | 5/10 | 9/10 | 4/10 | 10/10 | 5/10 |
| Simulation performance | 5/10 | 9/10 | 6/10 | 8/10 | 4/10 |
| Cross-platform | 9/10 | 9/10 | 9/10 | 2/10 | 7/10 |
| Best fit score | 5/10 | 8/10 | 4/10 | 9/10 | 3/10 |

---

## Recommended Framework

**Tauri** for cross-platform, **Swift/SwiftUI** for Mac-only.

**Rationale:** CircuitMind requires native GPU access for professional-grade PCB rendering and compute shaders for DRC/zone fill. Tauri with wgpu provides Metal/Vulkan/DX12 access across all platforms, while the Rust backend handles SPICE simulation and parallel DRC with zero overhead. For a Mac-exclusive version, Swift with Metal delivers the absolute best GPU performance on Apple Silicon, with unified memory enabling boards that would exhaust GPU VRAM on discrete cards. The cross-platform need of hardware engineering teams (many on Windows) makes Tauri the pragmatic primary choice.

**Runner-up:** Swift/SwiftUI for a premium Mac-only experience. If analytics show >70% macOS users, consider a native Swift build with Metal rendering as the primary product.

---

## Monetization (Desktop)

- **Free tier:** Schematic editor only, up to 50 components, no PCB layout, basic DRC
- **Pro ($29/mo or $249/yr):** Full PCB layout, SPICE simulation, Gerber export, AI routing assistant
- **Enterprise ($99/mo):** Team collaboration, IP protection features, advanced impedance calculator, priority component database
- **Academic ($9/mo):** Full features at academic pricing with .edu email verification
- **One-time license ($499):** Perpetual professional license with 1 year updates — appeals to independent hardware designers

---

## MVP Scope (Desktop)

**Core MVP (8 weeks):**

| Week | Milestone |
|------|-----------|
| 1-2 | GPU-accelerated schematic canvas (wgpu/Metal), component placement, net wiring, basic component library |
| 3-4 | PCB layout editor with layer stack, footprint placement, manual trace routing, copper zone fill |
| 5-6 | DRC engine with spatial indexing, SPICE simulation integration (ngspice subprocess), waveform viewer |
| 7 | Gerber export, BOM generation, component search (Octopart API), project save/load |
| 8 | Auto-update, packaging, multi-window support, beta testing with 5 hardware engineers |

**Post-MVP:** AI auto-routing, 3D board preview, impedance calculator, team collaboration, version control integration, pen tablet optimization
