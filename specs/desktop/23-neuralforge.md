# NeuralForge Desktop — 5-Framework Comparison

## Desktop Rationale

NeuralForge (visual neural network architecture builder and trainer) is a GPU-intensive application where desktop access is essential:

- **Direct GPU access for training:** Neural network training requires direct access to CUDA cores (NVIDIA), Metal Performance Shaders (Apple Silicon), or ROCm (AMD). Browser-based tools cannot access local GPUs — desktop enables training on the user's own hardware without cloud compute costs.
- **Visual network builder:** Drag-and-drop layer composition with real-time architecture validation, parameter counting, and shape propagation requires a responsive canvas with sub-frame latency that browsers struggle to deliver at scale.
- **Local dataset management:** Training datasets range from megabytes (MNIST) to hundreds of gigabytes (ImageNet, custom datasets). Desktop apps read directly from disk with memory-mapped I/O — no uploads, no cloud storage costs.
- **Real-time training monitoring:** Live loss curves, gradient histograms, activation visualizations, and weight distribution plots updating every batch require direct GPU-to-display pipeline for smooth rendering during training.
- **Large model files:** Trained model checkpoints are 100 MB to 10+ GB. Desktop enables instant save/load from local disk, model comparison, and export to deployment formats (ONNX, TorchScript, TFLite) without transfers.

---

## Desktop-Specific Features

- **Visual network canvas:** Drag-and-drop layer composition with auto-connected data flow graph and real-time shape inference
- **GPU training engine:** Direct CUDA/Metal/ROCm access for local model training with real-time progress
- **Dataset browser:** Visual dataset explorer with image/text previews, class distribution charts, and augmentation preview
- **Live training dashboard:** Real-time loss curves, accuracy metrics, gradient flow visualization, and layer activation heatmaps
- **Model checkpoint manager:** Save, compare, and rollback model versions with training metrics side-by-side
- **Architecture search:** AI-assisted network architecture suggestions based on dataset characteristics and task type
- **Export pipeline:** One-click export to ONNX, TorchScript, TFLite, CoreML, and TensorRT formats
- **Multi-GPU support:** Distribute training across multiple local GPUs with automatic data parallelism
- **Hyperparameter sweep:** Local hyperparameter search with parallel experiment tracking
- **Notebook integration:** Import/export to Jupyter notebooks for code-level customization

---

## Shared Packages

### core (Rust)

- **Graph engine:** Directed acyclic graph (DAG) for network architecture representation with topological sort, cycle detection, and shape propagation
- **Shape inference:** Propagate tensor shapes through the network graph, detect dimension mismatches, and calculate parameter counts per layer
- **Training orchestrator:** Manage training loops, learning rate schedules, early stopping, checkpoint saving, and gradient clipping
- **Metric tracker:** Compute and store loss values, accuracy, F1, precision, recall, confusion matrices, and custom metrics per epoch/batch
- **Export engine:** Convert internal graph representation to ONNX format, TorchScript, TFLite, and CoreML via format-specific serializers
- **Dataset loader:** Memory-mapped data loading with configurable augmentation pipeline (crop, flip, normalize, mixup)

### api-client (Rust)

- **Model hub sync:** Push trained models to NeuralForge cloud hub for sharing, versioning, and deployment
- **Pre-trained model fetch:** Download pre-trained weights (ResNet, BERT, ViT) from model hub for transfer learning
- **Cloud training relay:** Offload large training jobs to cloud GPUs when local hardware is insufficient
- **Auth:** API key for model hub access, OAuth2 for team workspaces
- **Telemetry:** Optional anonymous training metrics for community benchmarks and hardware compatibility data

### data (SQLite)

- **Schema:** projects, architectures (graph JSON, parameter_count, input_shape, output_shape), experiments (hyperparams JSON, metrics JSON, checkpoint_path, status), datasets (path, type, size, class_count), export_history, settings
- **Time-series metrics:** Batch-level and epoch-level training metrics stored in optimized format for fast chart rendering
- **Experiment comparison:** Indexed queries for comparing metrics across experiments with filtering and sorting
- **Binary storage:** Model architecture snapshots and small checkpoint metadata in BLOB columns

---

## Framework Implementations

### Electron

**Architecture:** Main process orchestrates training subprocess (Python/PyTorch), dataset indexing, and model checkpointing. Renderer process (React) provides visual network builder (React Flow), training dashboard (Recharts/D3), and dataset browser. IPC streams training metrics from subprocess to renderer in real-time.

**Tech stack:**
- Renderer: React 18, React Flow (node graph editor), Recharts/D3 (training charts), TailwindCSS, Zustand
- Main: child_process (Python/PyTorch training), better-sqlite3, WebSocket (training metric stream)
- GPU: Training delegated to PyTorch subprocess (CUDA/MPS); visualization via WebGL2
- Dataset: fs.createReadStream for large file loading, sharp for image thumbnail generation

**Native module needs:** better-sqlite3, node-pty (Python subprocess with PTY for rich output), sharp

**Bundle size:** ~220-300 MB (Chromium + bundled Python runtime optional)

**Memory:** ~400-800 MB (Chromium + React Flow graph + training metric history; GPU memory managed by PyTorch separately)

**Pros:**
- React Flow is the best node graph editor available — battle-tested, extensible, well-documented
- Rich charting ecosystem (Recharts, D3, Plotly) for training dashboard visualizations
- Training metrics can stream via WebSocket to produce smooth real-time chart updates
- Massive ecosystem for dataset previews (image viewers, text renderers, audio players)
- Familiar web stack attracts ML engineers who primarily work in Python/JavaScript

**Cons:**
- No direct GPU access from Electron — must shell out to Python/PyTorch subprocess
- React Flow performance degrades with 200+ nodes (large architectures)
- Memory overhead of Chromium reduces RAM available for dataset loading
- IPC overhead between main process and Python subprocess adds latency to metric updates
- 300 MB bundle for a tool that also requires a Python environment is excessive

### Tauri

**Architecture:** Rust backend orchestrates PyTorch training subprocess, manages datasets via memory-mapped I/O, and handles model export. GPU-accelerated network visualization via wgpu for the architecture canvas. Svelte frontend for UI panels, training dashboard, and dataset browser.

**Tech stack:**
- Frontend: Svelte 5, Svelvet (Svelte node graph), Chart.js/uPlot (training charts), TailwindCSS
- Backend: wgpu (architecture visualization), rusqlite, tokio (async training management), ort (ONNX export/inference)
- Training: PyTorch subprocess via tokio::process with streaming stdout/stderr metric parsing
- Dataset: memmap2 (memory-mapped dataset access), image crate (thumbnail generation)

**Plugin needs:** tauri-plugin-dialog, tauri-plugin-fs, tauri-plugin-shell (Python subprocess)

**Bundle size:** ~20-35 MB (+ Python/PyTorch environment managed separately)

**Memory:** ~80-200 MB (Tauri app; GPU memory managed by PyTorch subprocess separately)

**Pros:**
- wgpu can render large architecture graphs with GPU acceleration — handles 500+ node networks
- Memory-mapped dataset access via memmap2 enables datasets larger than RAM
- Tokio async subprocess management provides clean training orchestration with cancellation
- Rust's ort crate enables ONNX export and inference validation without Python
- uPlot renders 100K+ data points for training curves at 60 fps (faster than any React charting lib)
- Small app bundle — Python/PyTorch environment is managed separately (conda/venv)

**Cons:**
- Svelvet (Svelte node graph) is far less mature than React Flow
- Training still requires Python/PyTorch subprocess — Rust cannot replace PyTorch directly
- Fewer dataset preview libraries in Svelte compared to React ecosystem
- Building a polished visual network builder in Svelte is significant UI effort

### Flutter Desktop

**Architecture:** Flutter renders the network builder canvas (CustomPainter), training dashboard, and dataset browser. Python subprocess for training. Rust FFI for model export and dataset processing.

**Tech stack:**
- UI: Flutter 3.x, CustomPainter (node graph), fl_chart (training curves), Riverpod
- Training: Process.start -> Python/PyTorch, stdout stream parsing
- Export: flutter_rust_bridge -> ort (ONNX export)
- Database: drift (SQLite)

**Plugin needs:** window_manager, flutter_rust_bridge, file_picker, desktop_drop

**Bundle size:** ~35-50 MB (+ Python/PyTorch environment)

**Memory:** ~150-300 MB

**Pros:**
- CustomPainter provides GPU-accelerated canvas for network graph rendering
- fl_chart can render smooth animated training curves
- Hot reload excellent for iterating on the visual network builder UX
- Cross-platform single codebase — potential tablet companion for monitoring training remotely

**Cons:**
- CustomPainter node graph requires building drag-and-drop, connections, and ports from scratch
- No existing Flutter node graph editor comparable to React Flow
- Dataset image preview at scale requires custom widget virtualization
- Training subprocess management less ergonomic in Dart than Rust/Node
- Desktop text input for hyperparameter forms still has rough edges

### Swift/SwiftUI (macOS)

**Architecture:** SwiftUI for UI panels and property editors. Custom Metal-based canvas for network architecture visualization. Metal Performance Shaders (MPS) for training on Apple Silicon. MLX for high-performance local training alternative to PyTorch.

**Tech stack:**
- UI: SwiftUI, AppKit (multi-window, drag-and-drop canvas), Charts framework
- GPU: Metal (architecture canvas), MPS/MLX (training backend)
- Data: SwiftData
- Export: CoreML Tools (model export), ONNX via coremltools bridge
- Training: MLX (native Apple Silicon) or PyTorch with MPS backend via subprocess

**Bundle size:** ~15-25 MB (+ MLX/PyTorch weights)

**Memory:** ~60-180 MB (unified memory shared with GPU for training)

**Pros:**
- **MLX provides native Apple Silicon training** — no Python/PyTorch dependency for supported architectures
- Metal Performance Shaders for training eliminates subprocess overhead entirely
- Unified memory on Apple Silicon means GPU training can use full system RAM (32-192 GB on M-series)
- Native drag-and-drop canvas (NSView subclass) provides buttery smooth node graph interaction
- Charts framework for real-time training curves with native rendering
- ProMotion 120 fps for smooth architecture canvas manipulation
- Instruments GPU profiler for training performance optimization

**Cons:**
- macOS only — excludes Linux users (most ML engineers use Linux with NVIDIA GPUs)
- MLX supports fewer layer types and architectures than PyTorch
- No CUDA support — the dominant ML hardware ecosystem is excluded
- Must build visual node graph editor from scratch in AppKit/Metal
- Smaller ML community on Apple platforms vs. PyTorch/TensorFlow ecosystem

### Kotlin Multiplatform

**Architecture:** Compose Desktop for UI. Custom Compose Canvas for network graph. DJL (Deep Java Library) for training or subprocess management for PyTorch. SQLDelight for experiment tracking.

**Tech stack:**
- UI: Compose Desktop, Material 3, custom Canvas composable (node graph)
- Training: DJL (PyTorch/TensorFlow Java bindings) or subprocess -> Python/PyTorch
- Database: SQLDelight
- Export: DJL model export, or subprocess -> torch.onnx
- Charts: compose-charts or custom Canvas drawing

**Bundle size:** ~65-100 MB (+JRE)

**Memory:** ~250-500 MB (JVM + DJL + training data)

**Pros:**
- DJL provides Java-native bindings to PyTorch/TensorFlow — training without Python subprocess
- Compose Canvas with gesture detection can build a node graph editor
- Could share experiment tracking logic with an Android companion app
- JVM ecosystem has mature libraries for data processing and math (ND4J, Apache Commons Math)

**Cons:**
- DJL training performance is 10-30% slower than native PyTorch due to JNI overhead
- JVM memory overhead competes with training data for RAM
- Compose Canvas node graph editor must be built entirely from scratch
- Garbage collection pauses cause stutters during real-time training metric updates
- compose-charts is immature compared to Recharts/D3
- JVM startup time (~3s) makes the app feel sluggish for quick training iterations

---

## Comparison Matrix

| Dimension | Electron | Tauri | Flutter | Swift | Kotlin MP |
|-----------|----------|-------|---------|-------|-----------|
| Dev effort (weeks) | 8 | 10 | 12 | 12 | 12 |
| Bundle size | 260 MB | 28 MB | 45 MB | 20 MB | 80 MB |
| Memory usage | 600 MB | 140 MB | 220 MB | 120 MB | 380 MB |
| Startup time | 3.0s | 0.8s | 1.3s | 0.5s | 2.5s |
| Native feel | 5/10 | 7/10 | 5/10 | 10/10 | 4/10 |
| Offline capability | 7/10 | 9/10 | 7/10 | 9/10 | 7/10 |
| GPU/compute access | 5/10 | 8/10 | 5/10 | 10/10 | 6/10 |
| Visual builder quality | 9/10 | 6/10 | 5/10 | 7/10 | 4/10 |
| Cross-platform | 9/10 | 9/10 | 9/10 | 2/10 | 7/10 |
| Best fit score | 7/10 | 8/10 | 5/10 | 7/10 | 4/10 |

---

## Recommended Framework

**Electron** for best visual builder experience, **Tauri** for performance-focused build.

**Rationale:** NeuralForge's core value proposition is the visual network builder — the drag-and-drop UX must be polished and intuitive. React Flow is the most mature and capable node graph editor available, giving Electron a significant advantage in time-to-market and builder quality. Training performance is delegated to PyTorch subprocess regardless of framework, so Electron's resource overhead matters less here than in other apps. However, if the team has strong Rust expertise and can invest in building a custom graph editor, Tauri delivers better overall resource efficiency and a smaller bundle that ML engineers appreciate.

**Runner-up:** Tauri if the team can invest 4-6 extra weeks building a custom graph editor in Svelte/wgpu. Swift/SwiftUI is compelling for Mac-only with MLX native training, but excluding Linux/NVIDIA users eliminates the majority of the ML engineer audience.

---

## Monetization (Desktop)

- **Free tier:** Visual builder, training on CPU only, up to 10 layers, export to ONNX
- **Pro ($24/mo or $199/yr):** GPU training, unlimited layers, all export formats, hyperparameter sweep, experiment comparison
- **Team ($12/user/mo):** Shared model hub, experiment collaboration, team architecture templates
- **Enterprise ($49/user/mo):** Multi-GPU training, cloud training relay, custom layer plugins, priority support
- **Academic ($8/mo):** Full Pro features at academic pricing with institutional email verification

---

## MVP Scope (Desktop)

**Core MVP (8 weeks):**

| Week | Milestone |
|------|-----------|
| 1-2 | Visual network builder (React Flow / custom canvas): layer palette, drag-and-drop, connection wiring, shape inference |
| 3-4 | Training engine integration (PyTorch subprocess), real-time loss/accuracy charts, GPU detection and selection |
| 5-6 | Dataset browser with preview, augmentation pipeline configuration, model checkpoint save/load/compare |
| 7 | Export pipeline (ONNX, TorchScript, TFLite), hyperparameter configuration UI, experiment history |
| 8 | Auto-update, packaging (DMG/MSI/AppImage), sample projects and tutorials, beta testing with 10 ML engineers |

**Post-MVP:** Hyperparameter sweep, multi-GPU training, architecture search AI, Jupyter notebook import/export, collaborative training sessions, model deployment pipeline, custom layer plugin system
