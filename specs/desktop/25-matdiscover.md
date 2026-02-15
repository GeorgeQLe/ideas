# MatDiscover Desktop — 5-Framework Comparison

## Desktop Rationale

- **GPU-Accelerated DFT Calculations:** Density Functional Theory calculations for predicting material properties involve massive matrix diagonalization and FFT operations that benefit enormously from local GPU acceleration via CUDA/ROCm. Desktop execution enables researchers to iterate on DFT parameters without cloud compute latency or cost, running calculations on their workstation's GPU during the exploratory phase of materials research.

- **Local Crystal Structure Visualization:** Materials scientists need real-time 3D visualization of crystal structures, electron density maps, and phonon dispersion curves. Desktop rendering via GPU allows smooth rotation, slicing, and animation of unit cells with thousands of atoms, overlaid with scalar/vector field visualizations that would stutter over remote streaming.

- **Large Materials Databases:** Local storage of materials databases (Materials Project, AFLOW, ICSD) containing millions of entries with crystal structures, computed properties, and phase diagrams enables instant querying and cross-referencing without network dependency. These databases can exceed 50-100 GB and benefit from local SSD access patterns.

- **ML Model Training on Local GPU:** Materials property prediction models (graph neural networks for formation energy, band gap, elastic constants) require GPU-accelerated training on local datasets. Desktop apps can manage the full ML workflow — data preparation, model training, hyperparameter tuning, and inference — using the local GPU without cloud ML platform costs.

- **Offline Property Prediction:** Researchers working in secure facilities or remote field locations need offline capability for property prediction, phase diagram computation, and structure optimization. Desktop apps with locally cached models and databases enable productive research without internet connectivity.

---

## Desktop-Specific Features

- GPU-accelerated crystal structure visualization with ball-and-stick, polyhedral, and electron density isosurface rendering modes
- Local DFT calculation engine with support for plane-wave and localized basis set methods on multi-core CPU and GPU
- Interactive phase diagram viewer with thermodynamic stability analysis and convex hull visualization
- Materials database browser with faceted search across composition, space group, properties, and synthesis conditions
- Graph neural network model training interface with GPU utilization monitoring and training curve visualization
- Structure optimization tools with force field relaxation and nudged elastic band (NEB) transition state search
- Phonon dispersion and density-of-states calculation with interactive Brillouin zone visualization
- X-ray diffraction pattern simulation and comparison with experimental data for structure validation
- Composition-property mapping with Bayesian optimization for guided experimental design
- Local file import/export for CIF, POSCAR, XYZ, and other crystallographic formats with batch processing

---

## Shared Packages

### core (Rust)

- **dft-engine:** Plane-wave DFT solver with GPU-accelerated FFT (via cuFFT/clFFT) and iterative eigenvalue solvers, supporting LDA/GGA/hybrid exchange-correlation functionals with pseudopotential and PAW methods
- **crystal-lib:** Crystal structure manipulation library with symmetry detection (space group finder), supercell generation, surface slab builder, and Wyckoff position analysis using group theory operations
- **ml-inference:** Graph neural network inference engine for materials property prediction using ONNX Runtime with GPU backend, supporting pre-trained models for formation energy, band gap, and elastic constants
- **phase-diagram:** Thermodynamic phase diagram computation with convex hull construction in compositional space, Gibbs free energy minimization, and temperature-dependent phase boundary calculation
- **phonon-calc:** Phonon calculation engine using finite displacement or density functional perturbation theory (DFPT) with force constant matrix assembly and Fourier interpolation
- **xrd-sim:** X-ray and neutron diffraction pattern simulator with peak broadening models, Rietveld refinement support, and structure factor computation for arbitrary crystal structures

### api-client (Rust)

- Materials Project API client for querying computed properties, crystal structures, and phase diagrams with local caching
- AFLOW REST API integration for accessing the AFLOW database of computed materials properties
- OPTIMADE API client for federated querying across multiple materials databases using the standardized filter language
- Model registry API for downloading pre-trained ML models and uploading user-trained models to shared repositories
- Experiment tracking API for logging DFT calculations, ML training runs, and property predictions to team collaboration servers

### data (SQLite)

- Local materials database with indexed crystal structures, computed properties, and literature references supporting complex compositional and property-range queries
- ML model registry with versioned model weights, training metadata, and performance metrics for local model management
- Calculation history with input parameters, convergence data, and output properties for reproducing and comparing DFT runs
- User-defined materials collections with tagging, notes, and project organization

---

## Framework Implementations

### Electron

**Architecture:** Chromium renderer for UI panels (database browser, property tables, ML dashboard) with Three.js or Babylon.js for crystal structure 3D visualization in WebGL/WebGPU. Native Rust/C++ addon for DFT engine and ML inference running in worker threads. IPC between renderer and compute backend via structured clone.

**Tech stack:** React + TypeScript for UI, Three.js for 3D crystal visualization, TensorFlow.js or ONNX.js for lightweight ML inference, N-API native modules for DFT engine, Chart.js for property plots and phase diagrams.

**Native module needs:** Heavy — requires native DFT solver with GPU bindings (CUDA/ROCm), ML inference runtime (ONNX Runtime), and crystal symmetry libraries. File I/O for large database access via native addon.

**Bundle size:** ~300-400 MB (Chromium + native DFT libraries + ML models + materials database subset)

**Memory:** ~450-900 MB (Chromium + 3D scene graph + DFT working arrays + ML model weights + database cache)

**Pros:**
- Three.js ecosystem mature for scientific 3D visualization with extensive shader support
- Web-based charting libraries excellent for phase diagrams and property plots
- Rich table/grid components for browsing large materials databases
- WebGPU compute shaders emerging for GPU-accelerated computations in the browser

**Cons:**
- Chromium memory overhead leaves less RAM for DFT wavefunctions and density matrices
- WebGL/WebGPU abstraction layer reduces GPU compute efficiency for DFT by 20-40% vs native
- Serialization overhead for transferring large arrays (electron density grids) between main and renderer processes
- No direct CUDA/ROCm access — must route through native addons

### Tauri

**Architecture:** System webview for UI panels with Rust backend for all compute operations. Crystal structure visualization via wgpu rendered to a native surface or composited into the webview. DFT engine, ML inference, and database queries all execute in Rust async tasks with progress updates streamed to the frontend.

**Tech stack:** Svelte or Vue for frontend UI, wgpu for 3D crystal visualization and compute shaders, ONNX Runtime (Rust bindings) for ML inference, rusqlite for materials database, ndarray for numerical arrays, rayon for parallel computation.

**Native module needs:** Minimal — all compute is native Rust. wgpu provides GPU access for visualization and compute. DFT engine uses GPU compute shaders via wgpu or links to cuFFT/clFFT for FFT operations. No JNI or FFI bridge needed for core functionality.

**Bundle size:** ~50-80 MB (system webview + Rust compute binaries + bundled ML models + database schema)

**Memory:** ~150-400 MB (lightweight webview + DFT working arrays in Rust heap + ML model weights + wgpu GPU buffers)

**Pros:**
- wgpu compute shaders enable GPU-accelerated DFT operations (FFT, matrix operations) natively in Rust
- Rust's ndarray and nalgebra crates provide efficient numerical computing for scientific calculations
- Minimal memory overhead maximizes available RAM for DFT wavefunctions (which can consume GBs for large systems)
- Single Rust codebase for DFT engine, ML inference, visualization, and data management

**Cons:**
- 3D visualization ecosystem in Rust (wgpu) less mature than Three.js for scientific rendering
- Webview charting libraries less capable than Chromium-hosted options for complex phase diagrams
- CUDA interop from Rust requires unsafe bindings (cuda-rs) with less community support than Python/C++ CUDA
- Smaller pool of developers with both Rust and computational materials science expertise

### Flutter Desktop

**Architecture:** Skia-rendered UI for all panels with dart:ffi calls to Rust/C++ backend for DFT and ML compute. Crystal structure visualization via Flutter's custom 3D rendering or texture bridge to native OpenGL/Vulkan context. Isolates for background computation management.

**Tech stack:** Dart for UI logic, flutter_gl or custom texture plugin for 3D visualization, dart:ffi for native compute library access, provider/riverpod for state management, fl_chart for 2D property plots.

**Native module needs:** Significant — requires FFI wrappers for DFT engine, ML inference runtime, and crystal symmetry libraries. 3D visualization either through Skia (limited for scientific 3D) or via native texture interop requiring platform-specific code.

**Bundle size:** ~90-130 MB (Flutter engine + Skia + native compute libraries + ML models)

**Memory:** ~300-550 MB (Flutter/Skia + Dart heap + native DFT data + GPU buffers)

**Pros:**
- Consistent cross-platform UI for database browsers and property tables
- Strong widget system for building complex parameter input forms for DFT calculations
- Good animation support for visualizing structural relaxation trajectories
- Material Design components suitable for modern scientific application aesthetics

**Cons:**
- Skia not designed for scientific 3D visualization — crystal structure rendering requires native plugin
- FFI overhead for every DFT iteration step adds latency to interactive calculations
- Dart GC can cause frame drops during 3D visualization of large crystal structures
- Limited scientific computing libraries in Dart — all heavy computation must go through FFI

### Swift/SwiftUI (macOS)

**Architecture:** Native macOS application with SwiftUI for UI panels and SceneKit/Metal for crystal structure 3D visualization. DFT engine via Accelerate framework (vDSP for FFT, BLAS/LAPACK for matrix operations) and Metal compute shaders. ML inference via Core ML with GPU acceleration.

**Tech stack:** SwiftUI for UI, SceneKit for 3D crystal rendering, Metal for GPU compute (DFT FFT and matrix operations), Core ML for materials property prediction, Accelerate for CPU-based numerical computing, Charts framework for property plots.

**Native module needs:** None for macOS — full native access to Metal GPU, Accelerate framework, and Core ML. Rust DFT libraries linked as static libraries. Metal shaders for DFT compute kernels and crystal visualization.

**Bundle size:** ~35-55 MB (native binary + Metal shader archives + ML models + database)

**Memory:** ~120-300 MB (minimal runtime + DFT arrays + Metal GPU buffers + Core ML model)

**Pros:**
- Metal compute shaders provide best-in-class GPU performance on Apple Silicon for DFT FFT and matrix operations
- Apple Silicon unified memory eliminates CPU-GPU transfer overhead for large electron density grids
- Core ML enables efficient on-device ML inference with automatic GPU/Neural Engine utilization
- SceneKit provides high-quality 3D rendering for crystal structures with minimal code

**Cons:**
- macOS-only — excludes Linux users who dominate computational materials science
- No CUDA support — many DFT codes are optimized for NVIDIA GPUs
- Smaller materials science software ecosystem on macOS compared to Linux
- Core ML model conversion from PyTorch/TensorFlow adds friction to ML workflow

### Kotlin Multiplatform

**Architecture:** Compose Multiplatform for UI panels with JNI calls to native DFT and ML libraries. 3D crystal visualization via LWJGL (OpenGL) or JavaFX 3D. JVM-based numerical computing with ND4J for matrix operations, complemented by native GPU compute via JNI bridge.

**Tech stack:** Kotlin + Compose Multiplatform for UI, LWJGL for 3D visualization, ND4J/DL4J for ML and numerical computing, JNI for Rust/C++ DFT engine integration, SQLDelight for materials database, Kotlin coroutines for async computation.

**Native module needs:** Moderate — JNI wrappers for Rust DFT engine and GPU compute libraries. ND4J provides CUDA backend for matrix operations but with JVM overhead. LWJGL for GPU-accelerated 3D rendering. Memory management across JVM and native heap for large numerical arrays.

**Bundle size:** ~100-150 MB (JVM runtime + Compose/Skia + native libraries + ND4J + ML models)

**Memory:** ~350-700 MB (JVM heap + Compose/Skia + ND4J arrays + native DFT data + GPU buffers)

**Pros:**
- ND4J provides mature GPU-accelerated numerical computing on JVM with CUDA support
- DL4J ecosystem supports training and inference for materials property prediction models
- Kotlin coroutines enable clean management of long-running DFT calculations with cancellation
- Cross-platform deployment covers macOS, Windows, and Linux workstations

**Cons:**
- JVM memory overhead competes with DFT wavefunctions for available RAM (critical for large systems)
- JNI bridge adds complexity for GPU compute kernel invocation during iterative DFT solves
- JVM garbage collection pauses disrupt smooth 3D crystal structure visualization
- ND4J/DL4J ecosystem declining in favor of Python-based ML frameworks

---

## Comparison Matrix

| Dimension | Electron | Tauri | Flutter | Swift | Kotlin MP |
|-----------|----------|-------|---------|-------|-----------|
| Dev effort (weeks) | 22 | 18 | 24 | 16 | 26 |
| Bundle size | 350 MB | 65 MB | 110 MB | 45 MB | 125 MB |
| Memory usage | 700 MB | 280 MB | 420 MB | 200 MB | 500 MB |
| Startup time | 3.5s | 1.2s | 2.2s | 0.7s | 3.0s |
| Native feel | 5/10 | 7/10 | 6/10 | 10/10 | 6/10 |
| Offline capability | 8/10 | 9/10 | 8/10 | 9/10 | 8/10 |
| GPU/compute access | 5/10 | 9/10 | 4/10 | 10/10 | 6/10 |
| Cross-platform | 9/10 | 9/10 | 8/10 | 2/10 | 8/10 |
| Best fit score | 5/10 | 9/10 | 4/10 | 6/10 | 5/10 |

---

## Recommended Framework

**Tauri** is the best fit for MatDiscover.

**Rationale:** Materials discovery workloads are dominated by GPU-intensive DFT calculations, large numerical arrays (wavefunctions, electron density grids), and 3D scientific visualization — all of which demand minimal runtime overhead and direct GPU access. Tauri's Rust backend provides wgpu for both visualization and compute shaders, native-speed numerical operations via ndarray/nalgebra, and memory-efficient handling of large datasets. Cross-platform support is essential since computational materials scientists primarily use Linux workstations with NVIDIA GPUs, but also need macOS and Windows support.

**Runner-up:** Swift/SwiftUI delivers the best single-platform performance on Apple Silicon with Metal compute and Core ML inference, but the macOS-only limitation is a dealbreaker for a field dominated by Linux workstations with NVIDIA GPUs.

---

## Monetization (Desktop)

- **Freemium model** with free structure visualization and database browsing, paid tiers for DFT calculation ($99/month researcher, $499/month group license) and ML training features
- **Pre-trained model marketplace** where researchers can buy/sell specialized materials property prediction models ($50-500 per model)
- **Curated database subscriptions** for premium materials databases (ICSD, Pearson's) with enhanced property data ($200-1,000/year)
- **Cloud DFT burst compute** for calculations exceeding local GPU capacity (large supercells, hybrid functional calculations) at $1-5/GPU-hour
- **Enterprise/institutional licenses** with priority support, custom model training, and integration with laboratory information management systems (LIMS)

---

## MVP Scope (Desktop)

**Core MVP (8 weeks):**

| Week | Milestone |
|------|-----------|
| 1-2 | Crystal structure viewer with ball-and-stick rendering via wgpu, CIF/POSCAR file import, unit cell display with symmetry operations, rotation/zoom navigation |
| 3-4 | Materials database browser with SQLite backend, faceted search by composition/space group/property ranges, property table view with sorting and filtering |
| 5-6 | Basic DFT energy calculation engine (plane-wave, LDA pseudopotentials) with GPU-accelerated FFT, self-consistency loop with convergence monitoring, total energy output |
| 7 | ML property prediction using pre-trained GNN model (formation energy, band gap) via ONNX Runtime, batch prediction across database entries, prediction confidence display |
| 8 | Testing, performance profiling for large crystal structures (500+ atoms), packaging for macOS/Linux/Windows, user documentation for researchers |

**Post-MVP:** Phase diagram computation with convex hull visualization, phonon dispersion calculation, structure optimization with force relaxation, Bayesian optimization for experimental design, X-ray diffraction simulation, advanced DFT (GGA, hybrid functionals, spin-orbit coupling), custom ML model training interface, OPTIMADE federation for querying external databases, Jupyter notebook integration for scripting, collaboration features for research groups.
