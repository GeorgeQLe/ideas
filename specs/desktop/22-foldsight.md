# FoldSight Desktop — 5-Framework Comparison

## Desktop Rationale

FoldSight (protein structure prediction and visualization) is a computationally intensive scientific application where desktop is the only viable platform:

- **GPU-accelerated 3D visualization:** Protein structures with 10,000+ atoms and complex surface representations (van der Waals, solvent-accessible, electrostatic) require native GPU rendering. WebGL cannot handle molecular dynamics trajectories or real-time surface computation at interactive framerates.
- **Local AlphaFold inference:** Running AlphaFold2/ESMFold locally on consumer GPUs (NVIDIA RTX, Apple Silicon) enables structure prediction without uploading proprietary sequences to cloud services. Essential for pharmaceutical IP protection.
- **Large PDB file handling:** Protein Data Bank files for large complexes (ribosomes, viral capsids) can be 500 MB+. Desktop apps load these from local disk in seconds; browser uploads would take minutes.
- **Offline molecular viewer:** Researchers in wet labs, field stations, and secure facilities need full visualization and analysis without internet. Desktop enables complete offline molecular analysis workflows.
- **Multi-monitor for structure comparison:** Side-by-side comparison of predicted vs. experimental structures, or wild-type vs. mutant, across multiple monitors is a core research workflow that browsers handle poorly.

---

## Desktop-Specific Features

- **GPU molecular renderer:** Hardware-accelerated atom, bond, ribbon, surface, and cartoon representations with real-time lighting and shadows
- **Local structure prediction:** Embedded AlphaFold2/ESMFold inference pipeline with GPU acceleration (CUDA/Metal)
- **Trajectory viewer:** Load and animate molecular dynamics trajectories (XTC, DCD, TRR formats) at 60 fps
- **Multi-model overlay:** Superimpose multiple structures with RMSD alignment and per-residue deviation coloring
- **Electron density maps:** Load and render crystallographic electron density (CCP4/MRC maps) with isosurface extraction
- **Sequence viewer:** Linked sequence-structure viewer — click a residue in sequence, highlight in 3D and vice versa
- **Measurement tools:** Distance, angle, dihedral measurements with interactive 3D handles
- **Session management:** Save/restore complete visualization state (camera, representations, selections, measurements)
- **PDB/mmCIF import:** Drag-and-drop local files or fetch from RCSB PDB by accession code
- **Publication-quality export:** Ray-traced rendering for figures, POV-Ray export, high-DPI PNG/TIFF

---

## Shared Packages

### core (Rust)

- **PDB/mmCIF parser:** Parse standard molecular structure formats (PDB, mmCIF, MOL2, SDF) into an atom-level data model with chain/residue/atom hierarchy
- **Geometry engine:** Spatial hashing for neighbor searches, RMSD superposition (Kabsch algorithm), surface area calculation (SASA), hydrogen bond detection
- **Surface generator:** Marching cubes for solvent-accessible and molecular surfaces, Gaussian surface for electrostatic potential mapping
- **Alignment engine:** Sequence alignment (Needleman-Wunsch, Smith-Waterman) and structural alignment (TM-align) for multi-model comparison
- **Trajectory reader:** Stream molecular dynamics trajectory files (XTC, DCD) with frame interpolation for smooth playback
- **AI inference:** ONNX Runtime for local ESMFold/AlphaFold2 inference pipeline management, MSA generation

### api-client (Rust)

- **RCSB PDB fetch:** Download structures by PDB ID, search by sequence similarity, fetch biological assembly data
- **UniProt integration:** Fetch protein metadata, domain annotations, and variant data for structure-function correlation
- **AlphaFold DB:** Query AlphaFold Protein Structure Database for pre-computed predictions
- **Cloud sync:** Push visualization sessions and annotated structures to FoldSight cloud for team sharing
- **Auth:** Institutional SSO (SAML) for academic/pharma teams, API key for programmatic access

### data (SQLite)

- **Schema:** projects, structures (pdb_id, source, file_path, atom_count, resolution, method), sessions (camera_state, representations JSON, selections JSON, measurements JSON), alignments, predictions, annotations
- **Coordinate storage:** Compressed BLOB columns for atom coordinates (delta-encoded float16 for trajectories)
- **Full-text search:** FTS5 on structure titles, organism names, and annotation text
- **Cache:** Pre-computed surface meshes and alignment matrices cached by structure hash

---

## Framework Implementations

### Electron

**Architecture:** Main process handles file I/O, PDB parsing, and AI inference subprocess management. Renderer uses WebGL2 (Three.js + custom shaders) for molecular visualization. Web Workers for surface computation and alignment calculations. Separate window for sequence viewer.

**Tech stack:**
- Renderer: React 18, Three.js (molecular rendering), NGL Viewer (WebGL molecular graphics), D3 (sequence viewer), TailwindCSS
- Main: better-sqlite3, child_process (AlphaFold/ESMFold subprocess), fs (PDB file loading)
- GPU: WebGL2 via Three.js with custom GLSL shaders for molecular representations
- Compute: Web Workers for surface generation (marching cubes), SharedArrayBuffer for coordinate data

**Native module needs:** better-sqlite3, onnxruntime-node (structure prediction), sharp (image export)

**Bundle size:** ~250-350 MB (Chromium + NGL Viewer + model weights if bundled)

**Memory:** ~500-1200 MB (large structures consume significant memory; WebGL has overhead)

**Pros:**
- NGL Viewer is a mature, open-source WebGL molecular graphics library (used by RCSB PDB website)
- Three.js provides solid foundation for custom molecular representations
- Web Workers enable parallel surface computation without UI blocking
- Large scientific JavaScript ecosystem (BioJS, Plotly for analysis charts)

**Cons:**
- WebGL2 performance ceiling — ribosome-sized structures (300K+ atoms) drop below 30 fps
- No compute shader access — surface generation 5-10x slower than GPU compute
- 500+ MB memory usage leaves less RAM for AlphaFold inference
- Pen/trackball input for 3D rotation is laggy due to Chromium compositor
- Cannot leverage Metal/CUDA for structure prediction acceleration

### Tauri

**Architecture:** Rust backend handles PDB parsing, surface computation, structural alignment, and AI inference orchestration. GPU rendering via wgpu in a dedicated render surface for the molecular viewport. Svelte frontend for UI panels (sequence viewer, property inspector, representation controls).

**Tech stack:**
- Frontend: Svelte 5, TailwindCSS (UI panels), D3 (sequence viewer, Ramachandran plots)
- Backend: wgpu (molecular rendering + compute shaders), pdbtbx (PDB/mmCIF parser), rusqlite, rayon, ort (ONNX Runtime)
- GPU: wgpu with render pipelines for molecular graphics and compute pipelines for surface generation
- AI: ort (ONNX Runtime) for ESMFold inference, or subprocess management for AlphaFold2

**Plugin needs:** tauri-plugin-dialog, tauri-plugin-fs, tauri-plugin-shell (AI subprocess management)

**Bundle size:** ~30-50 MB (+ AI model weights ~2 GB, downloaded separately)

**Memory:** ~100-300 MB (wgpu manages GPU memory separately; spikes during prediction)

**Pros:**
- **wgpu compute shaders for surface generation** — marching cubes on GPU is 10-50x faster than CPU
- Native PDB parsing in Rust (pdbtbx) is fastest available — parse 1M atom structures in <1s
- rayon parallel iterators for RMSD calculation, neighbor search, hydrogen bond detection
- wgpu provides Metal/Vulkan/DX12 rendering — handles 500K+ atoms at 60 fps
- ONNX Runtime in Rust for local structure prediction without Python overhead
- Small base bundle; AI model weights downloaded on first use
- Cross-platform GPU access essential for academic labs (mixed macOS/Linux environments)

**Cons:**
- Building a full molecular graphics engine in wgpu is major engineering effort (6+ months)
- No existing Rust molecular visualization library — must build from scratch or port
- AlphaFold2 inference pipeline is primarily Python — Rust orchestration adds subprocess complexity
- wgpu + webview hybrid window management requires careful architecture

### Flutter Desktop

**Architecture:** Flutter UI for panels and controls. Native platform view embedding a Metal/Vulkan viewport for molecular rendering. Rust FFI for compute-intensive operations (parsing, surfaces, alignment).

**Tech stack:**
- UI: Flutter 3.x, Riverpod, custom sequence viewer widget
- GPU: Platform view -> native Metal/Vulkan/OpenGL renderer (C++ or Rust)
- Compute: flutter_rust_bridge -> Rust (PDB parsing, surface generation, alignment)
- Database: drift (SQLite)

**Plugin needs:** window_manager, flutter_rust_bridge, custom platform view for GPU viewport

**Bundle size:** ~40-60 MB (+ AI model weights)

**Memory:** ~180-350 MB

**Pros:**
- Platform view can embed a high-performance native renderer
- Custom widgets for sequence-structure linking with smooth animations
- Cross-platform from single codebase — potential tablet version for structure review
- Hot reload for rapid iteration on analysis panel UIs

**Cons:**
- Platform view for GPU rendering is complex and has compositing overhead on each platform
- No existing Flutter molecular visualization — entire renderer must be native
- flutter_rust_bridge adds latency to every compute call (serialization overhead)
- Scientific visualization community has no Flutter presence
- Text rendering for sequence annotations is less crisp than native

### Swift/SwiftUI (macOS)

**Architecture:** SwiftUI panels with Metal-based molecular renderer (MTKView). Metal compute shaders for surface generation and neighbor search. CoreML/MLX for structure prediction on Apple Silicon. SceneKit for simplified 3D preview mode.

**Tech stack:**
- UI: SwiftUI (panels, sequence viewer), AppKit (multi-window, toolbar)
- GPU: Metal (MTKView for molecular rendering, compute shaders for surfaces)
- AI: MLX (Apple Silicon optimized) for ESMFold, CoreML for smaller models
- Data: SwiftData
- Parsing: Custom Swift PDB parser, or C library (gemmi) via Swift package

**Bundle size:** ~15-25 MB (+ MLX model weights ~2 GB)

**Memory:** ~60-180 MB (Metal manages GPU memory via unified memory architecture)

**Pros:**
- **Metal provides best-in-class GPU performance on macOS** — unified memory on Apple Silicon means GPU can access full system RAM for massive structures
- Metal compute shaders for marching cubes surface generation run 20x faster than CPU
- MLX on Apple Silicon provides fastest local structure prediction (leverages ANE + GPU + unified memory)
- ProMotion display support for 120 fps molecular visualization on MacBook Pro
- Multi-window native in AppKit — structure comparison across monitors is seamless
- Instruments profiling for GPU/CPU optimization
- Haptic feedback on Force Touch trackpad for molecular interaction

**Cons:**
- macOS only — excludes Linux users (many computational biologists use Linux)
- No CUDA support — NVIDIA GPU users cannot leverage their hardware for inference
- Must build molecular renderer from scratch in Metal (no existing Swift molecular graphics library)
- Smaller community for scientific visualization in Swift vs. Python/C++

### Kotlin Multiplatform

**Architecture:** Compose Desktop UI for panels. LWJGL for OpenGL/Vulkan molecular rendering. JNI bridge to native compute libraries. BioJava for PDB parsing. SQLDelight for persistence.

**Tech stack:**
- UI: Compose Desktop, Material 3
- GPU: LWJGL (OpenGL/Vulkan), or JavaFX 3D (limited)
- Parsing: BioJava (PDB/mmCIF, sequence alignment)
- Database: SQLDelight
- AI: DJL (Deep Java Library) for ONNX inference
- HTTP: Ktor client

**Bundle size:** ~80-120 MB (+JRE + BioJava)

**Memory:** ~300-700 MB (JVM + GPU context + structure data)

**Pros:**
- BioJava is the most mature Java molecular biology library (PDB parsing, alignment, structure analysis)
- LWJGL provides full Vulkan/OpenGL access from JVM
- DJL supports PyTorch/ONNX model inference for structure prediction
- Strong bioinformatics ecosystem in JVM (BioJava, CDK, Jmol)
- Could share structure analysis logic with an Android tablet viewer

**Cons:**
- JVM memory overhead leaves less RAM for structure data and GPU buffers
- Garbage collection pauses cause visible stutters during smooth 3D rotation
- LWJGL + Compose interop is fragile for embedded GPU viewports
- JVM startup time (~3s with BioJava initialization) frustrates researchers
- Cannot compete with native GPU performance for production molecular graphics

---

## Comparison Matrix

| Dimension | Electron | Tauri | Flutter | Swift | Kotlin MP |
|-----------|----------|-------|---------|-------|-----------|
| Dev effort (weeks) | 10 | 14 | 16 | 14 | 14 |
| Bundle size | 300 MB | 40 MB | 50 MB | 20 MB | 100 MB |
| Memory usage | 800 MB | 200 MB | 260 MB | 120 MB | 500 MB |
| Startup time | 3.5s | 1.0s | 1.5s | 0.5s | 3.0s |
| Native feel | 4/10 | 7/10 | 5/10 | 10/10 | 3/10 |
| Offline capability | 7/10 | 9/10 | 7/10 | 9/10 | 7/10 |
| GPU/compute access | 4/10 | 9/10 | 5/10 | 10/10 | 5/10 |
| 3D rendering quality | 6/10 | 9/10 | 5/10 | 10/10 | 5/10 |
| Cross-platform | 9/10 | 9/10 | 9/10 | 2/10 | 7/10 |
| Best fit score | 5/10 | 8/10 | 4/10 | 9/10 | 4/10 |

---

## Recommended Framework

**Tauri** for cross-platform, **Swift/SwiftUI** for Mac-only.

**Rationale:** FoldSight demands native GPU access for real-time molecular visualization and compute shaders for surface generation. Tauri with wgpu delivers Metal/Vulkan/DX12 across all platforms, and Rust's performance for PDB parsing, spatial algorithms, and ONNX inference is unmatched. Cross-platform support is critical because computational biology labs typically run mixed macOS/Linux environments. For a Mac-exclusive product, Swift with Metal on Apple Silicon provides the absolute best experience — unified memory enables structures that would exhaust discrete GPU VRAM, and MLX delivers fastest local structure prediction.

**Runner-up:** Swift/SwiftUI if targeting Mac-only (best GPU performance via Metal, MLX for local AI). Consider building a premium macOS version alongside the cross-platform Tauri build if resources allow.

---

## Monetization (Desktop)

- **Free tier (Academic):** Visualization only, up to 5,000 atoms, basic representations, no structure prediction
- **Pro ($39/mo or $349/yr):** Unlimited structure size, all representations, local structure prediction, surface computation, session management
- **Enterprise/Pharma ($149/seat/mo):** IP-safe local inference, audit logging, team collaboration, priority support, custom model integration
- **Academic site license ($2,000/yr per lab):** Full features for academic research groups with .edu domain
- **One-time license ($599):** Perpetual license for independent researchers and biotech startups

---

## MVP Scope (Desktop)

**Core MVP (8 weeks):**

| Week | Milestone |
|------|-----------|
| 1-2 | GPU molecular renderer (wgpu/Metal): atom, bond, ribbon representations; PDB/mmCIF parser; camera controls (rotate, zoom, pan) |
| 3-4 | Sequence viewer linked to 3D structure, residue selection, measurement tools (distance, angle), multi-model loading |
| 5-6 | Surface generation (solvent-accessible, van der Waals) via compute shaders, electron density map rendering, structure superposition |
| 7 | Local structure prediction integration (ESMFold via ONNX), RCSB PDB fetch, session save/restore |
| 8 | Publication-quality export (high-DPI PNG/TIFF), auto-update, packaging, beta testing with 5 structural biologists |

**Post-MVP:** Molecular dynamics trajectory viewer, AlphaFold2 full pipeline, electrostatic surface coloring, multi-monitor comparison mode, collaborative annotation, ligand docking visualization
