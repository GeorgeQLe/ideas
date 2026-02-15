# MolForge Desktop — 5-Framework Comparison

## Desktop Rationale

- **GPU-Accelerated Molecular Dynamics:** Running molecular dynamics (MD) simulations demands sustained GPU compute for force calculations, energy minimization, and trajectory integration — workloads that choke in browser sandboxes but thrive with direct CUDA/Metal/Vulkan access on desktop. A single 100-nanosecond simulation of a 50,000-atom protein system requires billions of floating-point operations per timestep, and modern GPU compute shaders can evaluate Lennard-Jones and Coulomb interactions 50x faster than CPU-only approaches.

- **3D Molecular Visualization:** Interactive rendering of protein structures, binding pockets, and electron density maps requires real-time ray-marching, ambient occlusion, and depth-of-field effects at 60fps — only achievable with native GPU pipelines and low-level graphics APIs. Visualizing solvent-accessible surfaces, Gaussian electron density isosurfaces, and pharmacophore features simultaneously demands multi-pass rendering with persistent GPU state that WebGL cannot maintain.

- **Local Docking Simulations:** Molecular docking (AutoDock Vina, GNINA) runs CPU/GPU-intensive scoring functions over millions of ligand poses; desktop execution eliminates round-trip latency and keeps proprietary compound libraries off external servers. A typical virtual screening campaign evaluates 100K+ compounds against a single target, generating terabytes of pose data that must be stored and analyzed locally.

- **Large Chemical Library Management:** Drug discovery pipelines routinely handle databases of 10M+ compounds with molecular fingerprints, SMILES strings, and computed descriptors — local SQLite/DuckDB storage with memory-mapped access outperforms any cloud-fetched approach. Substructure searching via ECFP4 fingerprint Tanimoto comparison must return results in under 500ms even for multi-million-compound libraries.

- **Offline Property Prediction:** ADMET prediction models, QSAR models, and retrosynthesis planners must run without connectivity in pharma secure environments where internet access is restricted by policy. Many pharmaceutical companies enforce air-gapped computing for pre-IND compound data, making cloud-only solutions non-compliant.

---

## Desktop-Specific Features

The following features are only feasible on desktop due to GPU compute, hardware access, and local data requirements:

- Direct GPU access for molecular dynamics force-field calculations (AMBER, CHARMM, OPLS) via compute shaders with support for periodic boundary conditions and PME electrostatics
- Real-time 3D molecular visualization with ball-and-stick, ribbon, surface, and volumetric electron density rendering modes using ray-marched isosurfaces
- Local AutoDock Vina / GNINA integration for ligand-protein docking with configurable search space, exhaustiveness, and GPU-accelerated CNN scoring
- Memory-mapped chemical library browsing for datasets exceeding 50M compounds without loading entire database into RAM
- Hardware-accelerated SMARTS/SMILES substructure search using GPU-parallel fingerprint comparison across millions of compounds in under one second
- Native file system integration for SDF, MOL2, PDB, CIF, and XYZ molecular file formats with drag-and-drop import and batch processing
- Multi-monitor support: structure viewer on primary display, property dashboards and docking results on secondary display
- Background simulation queue manager — run multi-day MD trajectories while continuing interactive design work in the foreground
- Local ONNX/TorchScript model inference for ADMET, solubility, and toxicity prediction without any cloud dependency
- System tray resident process for long-running simulation monitoring with desktop notifications on completion or convergence failure

---

## Shared Packages

All shared packages are compiled as Rust libraries with C-compatible FFI exports, enabling consumption from any framework.

### core (Rust)

- Molecular dynamics engine with Verlet integration, PME electrostatics, and Lennard-Jones force calculations optimized for SIMD — supporting AMBER ff14SB and CHARMM36m force fields
- Molecular docking orchestrator wrapping AutoDock Vina scoring functions with Rust-native pose generation, flexible residue handling, and ranked output
- Cheminformatics library for SMILES/SMARTS parsing, molecular fingerprint generation (ECFP4, MACCS, Morgan), and Tanimoto/Dice similarity search with GPU-parallel bitwise operations
- Conformer generation engine using distance geometry and force-field optimization (MMFF94) for 3D coordinate embedding from 2D structure input
- ADMET property prediction pipeline with ONNX Runtime bindings for local ML model inference — logP, aqueous solubility, BBB permeability, hERG inhibition, CYP450 metabolism
- Retrosynthesis planner using Monte Carlo tree search over reaction templates with GPU-accelerated rollout evaluation and beam search pruning

### api-client (Rust)

- PubChem and ChEMBL REST API client for compound metadata, bioactivity data, and target information retrieval with pagination and rate limiting
- UniProt/PDB async downloader for protein structure fetching with local caching, incremental sync, and automatic format validation
- Cloud simulation job submission client for offloading large-scale virtual screening campaigns to remote HPC clusters via SSH/SLURM
- License validation and telemetry client with offline grace period support (90 days) for air-gapped pharma environments
- Collaboration sync client for sharing molecular designs, docking results, and annotations across team members via encrypted delta sync

### data (SQLite)

- Chemical library schema with FTS5 indexes on compound names, SMILES, and InChI keys for sub-second full-text search across millions of records
- Simulation results store with trajectory frame indexing, energy time-series (kinetic, potential, total), RMSD tracking, and per-residue energy decomposition tables
- User project database tracking molecular designs, docking campaigns, property predictions, annotation history, and experiment provenance chains
- Cached structure repository for frequently accessed PDB entries and computed conformer ensembles with automatic expiry and refresh policies

---

## Framework Implementations

Each framework is evaluated for its ability to deliver GPU-accelerated molecular simulation, real-time 3D visualization, and large-scale chemical library management.

### Electron

**Architecture:** Chromium renderer for React-based 2D UI panels; Three.js/NGL Viewer for WebGL molecular visualization; Node.js child processes spawning Rust CLI binaries for MD simulations and docking; SharedArrayBuffer for trajectory streaming to renderer

**Tech stack:** React + TypeScript, Three.js with NGL Viewer, Node.js native addons via napi-rs for Rust core, WebGL2 for molecular rendering, better-sqlite3 for chemical library access

**Native module needs:** napi-rs bindings to Rust MD engine, OpenBabel native addon for format conversion, RDKit WASM or native build for cheminformatics, node-gpu for optional CUDA interop

**Bundle size:** ~280-350 MB

**Memory:** ~450-700 MB

**Pros:**
- NGL Viewer / Mol* are mature WebGL molecular viewers with extensive PDB format support and proven performance on structures up to 50K atoms
- Largest ecosystem of JavaScript cheminformatics libraries (RDKit.js, SmilesDrawer, OpenChemLib) reducing custom code requirements
- Cross-platform with identical rendering on Windows/macOS/Linux — no platform-specific visualization bugs
- Easier to recruit web developers familiar with Three.js and React for UI development and iteration

**Cons:**
- WebGL2 caps out on large protein complexes (>100K atoms) without aggressive LOD and instanced rendering workarounds
- Chromium overhead wastes 200+ MB of RAM before any scientific work begins — directly competing with simulation memory needs
- No direct compute shader access — MD simulations must run in separate Rust processes with IPC serialization overhead for trajectory data
- Security sandbox complicates file system access for SDF libraries and PDB caches, requiring explicit permission dialogs

### Tauri

**Architecture:** Rust backend running MD engine, docking, and cheminformatics natively; wgpu compute shaders for GPU-accelerated force calculations and fingerprint search; WebView frontend with lightweight 3D viewer using wgpu-rendered frames passed via shared memory; Tauri commands for all compute-intensive operations

**Tech stack:** Rust backend with wgpu for compute + rendering, Svelte/SolidJS frontend, sqlx for SQLite, ONNX Runtime Rust bindings, custom wgpu molecular renderer

**Native module needs:** wgpu for Vulkan/Metal/DX12 compute shaders, ONNX Runtime C API bindings, OpenBabel Rust FFI for format conversion

**Bundle size:** ~45-70 MB

**Memory:** ~120-250 MB

**Pros:**
- Direct wgpu compute shader access for MD force calculations — no IPC boundary between simulation engine and 3D renderer
- Rust-native molecular dynamics engine runs in-process with zero serialization overhead for trajectory data
- Minimal memory footprint leaves more RAM/VRAM for large protein simulations and chemical library indexing
- Single binary with Rust core means the simulation engine IS the application, not a bolted-on addon

**Cons:**
- No mature WebView-based molecular viewer — must build custom wgpu renderer or bridge to existing C++ viewers via FFI
- Smaller ecosystem for cheminformatics compared to Python/JS — more Rust library development needed for SMARTS matching and reaction handling
- WebView limitations for complex 2D UI interactions like property tables, 2D structure editors, and R-group decomposition panels
- wgpu compute shader debugging tooling less mature than CUDA ecosystem — harder to profile GPU kernel performance

### Flutter Desktop

**Architecture:** Skia/Impeller rendering engine for 2D UI; custom OpenGL/Metal texture bridge for molecular 3D viewport; Dart FFI to Rust core library for MD simulations and docking; platform channels for GPU compute dispatch

**Tech stack:** Dart + Flutter, flutter_rust_bridge for Rust core FFI, custom native texture plugin for molecular rendering, Impeller for UI, drift for SQLite

**Native module needs:** Rust core via FFI, OpenGL/Metal interop for 3D viewport, ONNX Runtime via C FFI, OpenBabel via C FFI

**Bundle size:** ~85-120 MB

**Memory:** ~250-400 MB

**Pros:**
- Consistent, polished UI across Windows/macOS/Linux with minimal platform-specific code for property panels and dashboards
- Impeller rendering engine provides smooth 120fps UI animations for property dashboards and data tables
- Strong typing in Dart catches UI bugs early; flutter_rust_bridge provides type-safe Rust interop with automatic code generation
- Hot reload dramatically speeds up UI iteration for complex property panels, compound tables, and result visualizations

**Cons:**
- No native 3D rendering pipeline — molecular viewer requires custom texture bridge adding significant architectural complexity
- Dart VM adds 80-120 MB memory overhead that competes with simulation workloads for system resources
- Flutter desktop ecosystem lacks scientific visualization widgets — molecular charts, 3D plots, heatmaps all require custom implementation
- GPU compute must go through Rust FFI — no direct compute shader access from the Flutter/Dart layer

### Swift/SwiftUI (macOS)

**Architecture:** SwiftUI for native macOS UI with Metal-backed SceneKit/RealityKit for molecular visualization; Metal compute shaders for MD force calculations and fingerprint search; Swift-Rust interop via C FFI for core simulation engine; Core Data or GRDB for local storage

**Tech stack:** SwiftUI + Metal, SceneKit for molecular rendering, Metal Performance Shaders for GPU compute, Swift-Rust FFI for core engine, GRDB/SQLite for data

**Native module needs:** Metal compute shaders (native), Accelerate framework for BLAS/LAPACK, Rust core via C-compatible FFI

**Bundle size:** ~30-50 MB

**Memory:** ~100-200 MB

**Pros:**
- Metal compute shaders provide lowest-latency GPU access on macOS for MD simulations with Apple Silicon unified memory
- SceneKit/RealityKit offer production-grade 3D rendering with PBR materials for molecular surfaces and ambient occlusion
- Tightest macOS integration — Touch Bar for simulation controls, Continuity for iPad companion viewer, Spotlight for compound search
- Smallest memory footprint among all options, maximizing available RAM/VRAM for large molecular systems

**Cons:**
- macOS-only — excludes Windows/Linux users who represent the majority of the computational chemistry community
- SceneKit optimized for gaming meshes, not scientific visualization — custom rendering pipeline needed for volumetric electron density
- Smaller Swift ecosystem for cheminformatics compared to Python/C++ tooling — limited open-source library availability
- No Windows port path — would require complete rewrite of rendering, compute, and platform integration layers

### Kotlin Multiplatform

**Architecture:** Compose Multiplatform for shared UI; JVM backend with JNI to Rust core for MD simulations; LWJGL/OpenGL for molecular 3D viewport; Kotlin coroutines for async simulation management; SQLDelight for cross-platform data

**Tech stack:** Kotlin + Compose Multiplatform, LWJGL for 3D rendering, JNI to Rust core, Exposed/SQLDelight for SQLite, kotlinx.coroutines for concurrency

**Native module needs:** JNI bindings to Rust MD engine, LWJGL for OpenGL context, JOML for 3D math, JNI to ONNX Runtime

**Bundle size:** ~120-180 MB

**Memory:** ~300-500 MB

**Pros:**
- JVM ecosystem provides access to CDK (Chemistry Development Kit) — the most comprehensive open-source cheminformatics library available
- Kotlin coroutines elegantly manage concurrent simulation jobs, docking queues, and UI updates without callback hell
- Cross-platform desktop support with shared business logic and UI code across Windows/macOS/Linux
- Strong interop with existing Java-based computational chemistry tools (CDK, JMol, OPSIN for IUPAC name parsing)

**Cons:**
- JVM startup and memory overhead significant — 200+ MB baseline RAM consumption before loading any molecular data
- JNI boundary between Kotlin and Rust/GPU adds latency for high-frequency simulation data transfer during trajectory streaming
- LWJGL molecular rendering requires fully custom implementation — no off-the-shelf molecular viewer for Kotlin/JVM
- GC pauses can cause visible frame drops during interactive molecular manipulation of large protein systems

---

## Comparison Matrix

| Dimension | Electron | Tauri | Flutter | Swift | Kotlin MP |
|-----------|----------|-------|---------|-------|-----------|
| Dev effort (weeks) | 14 | 18 | 20 | 16 | 22 |
| Bundle size | 320 MB | 55 MB | 100 MB | 40 MB | 150 MB |
| Memory usage | 550 MB | 180 MB | 320 MB | 150 MB | 400 MB |
| Startup time | 3.2s | 0.8s | 1.8s | 0.5s | 2.5s |
| Native feel | 5/10 | 7/10 | 7/10 | 10/10 | 6/10 |
| Offline capability | 8/10 | 9/10 | 8/10 | 9/10 | 8/10 |
| GPU/compute access | 4/10 | 9/10 | 5/10 | 10/10 | 5/10 |
| Cross-platform | 9/10 | 9/10 | 8/10 | 2/10 | 7/10 |
| Best fit score | 6/10 | 9/10 | 6/10 | 7/10 | 5/10 |

---

## Recommended Framework

**Tauri** is the best fit for MolForge.

**Rationale:** Molecular design demands tight coupling between GPU compute (force-field calculations, fingerprint search) and 3D visualization (molecular surfaces, electron density). Tauri's Rust-native architecture means the MD engine, docking algorithms, and wgpu compute shaders all run in the same process with zero IPC overhead — critical when streaming million-frame trajectories to the renderer at 60fps. The cross-platform reach ensures MolForge works across the Windows/Linux machines that dominate computational chemistry labs.

**Runner-up:** Swift/SwiftUI — Metal compute shaders and SceneKit provide the best single-platform performance for molecular visualization, but the macOS-only limitation is disqualifying for a tool targeting pharmaceutical research teams who overwhelmingly use Linux workstations and Windows desktops.

---

## Monetization (Desktop)

- **Free tier:** Single-molecule viewer, basic property calculation (molecular weight, logP, TPSA), up to 1,000-compound local library, ball-and-stick rendering only
- **Pro license ($49/month):** Unlimited molecular dynamics simulations, full docking suite with flexible residues, all ADMET prediction models, libraries up to 10M compounds, surface and ribbon rendering modes
- **Enterprise license ($199/seat/month):** Multi-user collaboration with annotation sync, custom force-field parameterization, HPC cluster job submission, audit logging for GxP compliance, SSO integration
- **Compute credits:** Pay-per-use ($0.10/ligand) for cloud-offloaded virtual screening campaigns exceeding local GPU capacity
- **Academic discount:** 80% off Pro license for .edu email addresses with annual verification; free for open-source drug discovery projects

---

## MVP Scope (Desktop)

**Core MVP (8 weeks):**

| Week | Milestone |
|------|-----------|
| 1-2 | Rust core: SMILES parser, molecular fingerprint engine (ECFP4), conformer generator with distance geometry; SQLite schema for compound library with FTS5 indexes; basic Tauri shell with file open/save for SDF/PDB formats and compound metadata display |
| 3-4 | wgpu molecular renderer: ball-and-stick and ribbon modes for proteins up to 50K atoms at 60fps; atom selection with distance/angle/dihedral measurement tools; 2D structure depiction panel with RDKit integration for SMILES-to-image |
| 5-6 | Molecular dynamics engine: Verlet integrator with Lennard-Jones + Coulomb forces, wgpu compute shaders for GPU-parallel force calculation across atom pairs; trajectory playback with energy plot overlay; temperature and pressure coupling |
| 7 | Docking module: AutoDock Vina scoring function integration via Rust FFI, configurable search box placement, pose ranking table with interaction analysis; batch docking queue supporting up to 100 ligands with progress reporting |
| 8 | Property prediction: ONNX Runtime integration for pre-trained ADMET models (logP, solubility, BBB permeability, hERG); chemical library import wizard for SDF/CSV with duplicate detection; end-to-end testing and cross-platform packaging |

**Post-MVP:**

- Retrosynthesis route planner with reaction template database (50K+ templates) and MCTS search with cost estimation
- Quantum mechanics module (semi-empirical methods: PM7, GFN2-xTB) for accurate binding energy calculations and geometry optimization
- Collaborative workspace with real-time annotation sync and conflict resolution for medicinal chemistry teams
- iPad companion app via Swift for molecule browsing and AR visualization of binding sites using RealityKit
- Integration with electronic lab notebooks (Benchling, Dotmatics) for experiment tracking and compound registration
- Free energy perturbation (FEP) workflow for lead optimization campaigns with automated perturbation graph generation
- Custom force-field parameterization UI for novel chemical series with automated charge fitting and torsion scanning
- Pharmacophore modeling and virtual screening with 3D pharmacophore feature matching against compound libraries
- QSAR model builder with automated descriptor calculation, feature selection, and cross-validation reporting
- Multi-target docking panel for polypharmacology studies — dock against multiple protein targets simultaneously and visualize selectivity profiles
- Molecular dynamics trajectory analysis tools: RMSD/RMSF plots, hydrogen bond occupancy tracking, principal component analysis of conformational dynamics
- Fragment-based drug design module with fragment linking, growing, and merging guided by binding site shape complementarity
- Export to common pharma formats: Maestro (.mae), MOE (.mdb), and KNIME workflow nodes for integration with existing computational chemistry pipelines
- Protein structure preparation wizard: protonation state assignment, missing residue modeling, disulfide bond detection, and water/ion placement
