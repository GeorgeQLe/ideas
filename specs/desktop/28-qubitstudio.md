# QubitStudio Desktop — 5-Framework Comparison

## Desktop Rationale

- **Local Quantum Circuit Simulation:** Simulating quantum circuits on classical hardware requires storing and manipulating complex-valued state vectors that grow exponentially with qubit count (2^n amplitudes). A 30-qubit simulation requires 16 GB of RAM for the state vector alone. Desktop execution enables researchers to leverage their full workstation memory (64-256 GB) for larger simulations without cloud compute costs, which scale dramatically for quantum workloads.

- **GPU-Accelerated State Vector Operations:** Quantum gate application involves matrix-vector multiplications across exponentially large Hilbert spaces. GPU acceleration via CUDA/Metal can provide 10-50x speedup over CPU for state vector simulation, enabling interactive exploration of circuits up to 25-30 qubits on consumer GPUs and 35+ qubits on workstation GPUs with 48+ GB VRAM. Desktop GPU access is essential for this performance.

- **Visual Circuit Builder:** Quantum circuit design requires a precise visual editor where gates are placed on qubit wires, multi-qubit gates span multiple wires, and circuit depth is visible at a glance. Desktop applications provide pixel-perfect rendering, low-latency drag-and-drop, snap-to-grid placement, and interactive parameter adjustment for parameterized gates — all with the responsiveness that quantum algorithm researchers demand during iterative circuit design.

- **Integration with Quantum Hardware APIs:** Quantum computers from IBM, Google, IonQ, and Rigetti are accessed via cloud APIs with job queuing and result polling. Desktop applications can manage multiple hardware provider credentials, maintain persistent job queues, correlate simulation results with hardware measurements, and provide unified dashboards across providers — all while keeping API keys secure in the local keychain.

- **Offline Circuit Design and Analysis:** Researchers frequently work on aircraft, in secure facilities, or in locations with unreliable internet. Desktop apps enable full circuit design, local simulation, circuit optimization (gate decomposition, routing), and result analysis without connectivity. Only hardware job submission requires internet access, and jobs can be queued offline for later submission.

---

## Desktop-Specific Features

- Visual quantum circuit builder with drag-and-drop gate placement, multi-qubit gate spanning, and parameterized gate editing with real-time parameter sliders
- GPU-accelerated state vector simulator supporting up to 30+ qubits on consumer GPUs with configurable precision (single/double)
- Density matrix simulator for mixed-state and noise simulation with configurable noise models (depolarizing, amplitude damping, phase damping)
- Bloch sphere visualization with animated gate operations showing single-qubit state evolution in real time
- State vector amplitude bar chart and phase wheel visualization for inspecting quantum states during circuit execution
- Circuit optimization passes including gate cancellation, commutation analysis, template matching, and hardware-aware routing for specific qubit topologies
- Multi-provider quantum hardware dashboard with IBM Quantum, IonQ, Rigetti, and Amazon Braket integration
- Integrated OpenQASM 3.0 / Qiskit / Cirq code editor with syntax highlighting, autocompletion, and circuit-to-code synchronization
- Measurement histogram analysis with statistical tools (expectation values, variance, tomography reconstruction)
- Variational quantum eigensolver (VQE) and QAOA optimization loops with classical optimizer integration and convergence visualization

---

## Shared Packages

### core (Rust)

- **statevector-sim:** GPU-accelerated state vector quantum circuit simulator using wgpu compute shaders or CUDA for gate application, supporting all standard gates (Pauli, Hadamard, CNOT, Toffoli, controlled-U) plus custom unitary matrices, with single and double precision modes
- **density-matrix-sim:** Density matrix simulator for open quantum system simulation with configurable noise channels (Kraus operators), supporting trace-preserving and non-trace-preserving maps for error analysis
- **circuit-optimizer:** Quantum circuit optimization engine with gate decomposition into native gate sets (e.g., {Rz, SX, CNOT}), commutation-aware rewriting rules, template matching for pattern-based optimization, and qubit routing for constrained topologies
- **qasm-parser:** OpenQASM 2.0/3.0 parser and code generator with AST representation, semantic analysis, and bidirectional conversion between visual circuit representation and textual QASM format
- **tomography:** Quantum state and process tomography algorithms including linear inversion, maximum likelihood estimation, and compressed sensing for reconstructing quantum states/processes from measurement data
- **vqe-engine:** Variational quantum eigensolver engine with Hamiltonian construction (molecular, Ising model), ansatz circuit generation (UCCSD, hardware-efficient), and classical optimizer integration (COBYLA, L-BFGS-B, SPSA)

### api-client (Rust)

- IBM Quantum API client for job submission, result retrieval, backend selection, and queue status monitoring with OAuth token management
- IonQ cloud API client with native gate set translation and trapped-ion-specific optimization
- Amazon Braket API client supporting IonQ, Rigetti, and simulator backends with S3 result storage
- Rigetti QCS API client with Quil program submission and pyQuil-compatible result format
- User account management for license validation, cloud simulation credits, and shared circuit library access

### data (SQLite)

- Circuit library with versioned quantum circuit definitions, gate decompositions, and metadata (qubit count, depth, gate count, description)
- Simulation result database with state vectors (compressed), measurement histograms, expectation values, and noise model parameters for reproducibility
- Hardware job tracking with submission timestamps, queue positions, execution results, and cost tracking across multiple quantum providers
- Custom gate library with user-defined unitary matrices, parameterized gate families, and composite gate templates

---

## Framework Implementations

### Electron

**Architecture:** Chromium renderer for the circuit builder canvas, code editor (Monaco), and visualization panels (Bloch sphere, histograms). Native Rust addon for GPU-accelerated state vector simulation and circuit optimization running in worker threads. IPC for transferring state vectors and measurement results to the renderer for visualization.

**Tech stack:** React + TypeScript for UI, HTML5 Canvas or SVG for circuit diagram rendering, Three.js for Bloch sphere 3D visualization, Monaco editor for QASM/Python code editing, D3.js for measurement histograms and convergence plots, N-API native modules for simulation engine.

**Native module needs:** Heavy — requires compiled Rust simulation engine with GPU bindings (CUDA/wgpu) for state vector operations, circuit optimizer, and QASM parser. GPU memory management for state vectors that can exceed GPU VRAM requires careful native-side handling.

**Bundle size:** ~250-320 MB (Chromium + native simulation libraries + gate library + example circuits + Monaco editor)

**Memory:** ~400-800 MB (Chromium + circuit canvas + state vector visualization buffers + simulation engine + code editor)

**Pros:**
- Monaco editor provides VS Code-quality code editing for QASM and Python with extensible language support
- SVG-based circuit rendering enables precise, scalable circuit diagrams with CSS styling
- Rich charting ecosystem (D3, Plotly) excellent for measurement histograms and VQE convergence plots
- Three.js Bloch sphere visualization with smooth animation of gate operations

**Cons:**
- Chromium memory overhead reduces available RAM for state vector storage (critical at high qubit counts)
- IPC serialization of large state vectors (2^30 complex numbers = 16 GB) between processes is impractical
- WebGL/WebGPU compute shader overhead reduces GPU simulation throughput vs native
- JavaScript event loop unreliable for timing-sensitive VQE optimization loops

### Tauri

**Architecture:** System webview for UI panels with Rust backend for all simulation computation. Circuit builder rendered in the webview using SVG/Canvas. Bloch sphere and state vector visualizations via wgpu rendered to native surfaces. Simulation engine runs as Rust async tasks with progress streaming.

**Tech stack:** Svelte or SolidJS for frontend UI, SVG for circuit diagram rendering, wgpu for 3D Bloch sphere and GPU compute shaders, CodeMirror for QASM code editing, rusqlite for circuit/result database, ndarray for complex-valued state vector operations.

**Native module needs:** Minimal — all simulation is native Rust. wgpu provides GPU compute shaders for state vector manipulation. Circuit optimizer and QASM parser are pure Rust. No FFI bridges needed for core simulation functionality.

**Bundle size:** ~40-65 MB (system webview + Rust simulation binary + gate library + example circuits)

**Memory:** ~100-250 MB base + state vector size (lightweight webview + simulation engine + wgpu GPU buffers; state vector dominates at high qubit counts)

**Pros:**
- Minimal memory overhead maximizes RAM for state vector storage — enables simulating 2-3 more qubits than Electron
- wgpu compute shaders provide GPU-accelerated gate application with near-native performance
- Rust's complex number support (num-complex crate) and ndarray enable efficient state vector operations
- Single Rust process eliminates IPC overhead for state vector access during visualization

**Cons:**
- CodeMirror less feature-rich than Monaco for IDE-quality code editing (no built-in language server protocol)
- SVG circuit rendering in system webview may have performance differences across platforms
- CUDA interop from Rust (for NVIDIA-specific optimizations) requires unsafe cuda-rs bindings
- Smaller ecosystem for scientific visualization compared to JavaScript/Python quantum computing tools

### Flutter Desktop

**Architecture:** Skia-rendered UI for circuit builder, parameter panels, and result dashboards with platform channels to Rust backend for simulation. Circuit diagram rendered via Flutter custom painter. Bloch sphere via custom 3D painter or texture bridge. FFI calls to simulation engine.

**Tech stack:** Dart for UI logic, custom painter for circuit diagrams, dart:ffi for Rust simulation engine, provider/riverpod for state management, fl_chart for measurement histograms, Isolates for background simulation coordination.

**Native module needs:** Significant — requires FFI wrappers for state vector simulator, circuit optimizer, and QASM parser. GPU compute for simulation requires platform-specific native plugins. Complex number arrays must cross the FFI boundary for state vector visualization.

**Bundle size:** ~80-110 MB (Flutter engine + Skia + native simulation libraries + gate library)

**Memory:** ~250-500 MB base + state vector (Flutter/Skia + Dart heap + native simulation data + GPU buffers)

**Pros:**
- Custom painter API well-suited for rendering quantum circuit diagrams with interactive gate placement
- Flutter's gesture system handles drag-and-drop gate placement with smooth animations
- Cross-platform consistency for circuit builder and visualization panels
- Strong widget library for building parameter tuning interfaces (variational angle sliders, noise model configuration)

**Cons:**
- FFI overhead for transferring state vectors between Rust simulation and Dart visualization
- No built-in code editor component — must integrate native editors via platform channels
- Dart GC can cause frame drops during animated Bloch sphere visualization
- Flutter's 2D rendering paradigm requires workarounds for 3D Bloch sphere and state space visualization

### Swift/SwiftUI (macOS)

**Architecture:** Native macOS application with SwiftUI for UI panels and Metal for GPU-accelerated quantum simulation and Bloch sphere visualization. Circuit builder using SwiftUI Canvas or AppKit custom view. State vector simulation via Metal compute shaders with Accelerate framework for CPU fallback.

**Tech stack:** SwiftUI for UI, Metal for GPU compute (state vector simulation) and 3D visualization (Bloch sphere), SceneKit for 3D rendering, Accelerate (vecLib) for BLAS/LAPACK operations, Swift Package Manager for dependencies.

**Native module needs:** None for macOS — full native access to Metal GPU for simulation compute shaders and 3D rendering. Accelerate framework provides optimized complex matrix operations. Rust simulation library linkable as static library for cross-platform algorithm sharing.

**Bundle size:** ~25-45 MB (native binary + Metal shader archives + gate library + example circuits)

**Memory:** ~80-200 MB base + state vector (minimal runtime + Metal GPU buffers + circuit data)

**Pros:**
- Metal compute shaders provide best-in-class GPU performance on Apple Silicon for state vector simulation
- Unified memory on Apple Silicon eliminates CPU-GPU transfer overhead for state vectors (massive win for quantum simulation)
- SwiftUI Canvas provides smooth, native circuit diagram rendering with hardware-accelerated compositing
- Smallest memory footprint maximizes qubits simulable on a given machine

**Cons:**
- macOS-only — excludes Linux users who are significant in quantum computing research
- No CUDA support — many quantum simulation optimizations target NVIDIA GPUs
- SwiftUI Canvas less mature than HTML5 Canvas/SVG for complex interactive diagram editing
- Smaller quantum computing library ecosystem on Swift compared to Python

### Kotlin Multiplatform

**Architecture:** Compose Multiplatform for UI panels with JNI calls to Rust simulation engine. Circuit diagram via Compose Canvas. 3D Bloch sphere via LWJGL (OpenGL). JVM-based classical optimizer for VQE loops with JNI bridge to GPU-accelerated simulation.

**Tech stack:** Kotlin + Compose Multiplatform for UI, Compose Canvas for circuit diagrams, LWJGL for 3D Bloch sphere rendering, JNI for Rust simulation engine, Apache Commons Math for classical optimization, SQLDelight for circuit database, Kotlin coroutines for async simulation.

**Native module needs:** Moderate — JNI wrappers for GPU-accelerated simulation engine and circuit optimizer. LWJGL for 3D rendering. Classical optimization can run on JVM (Apache Commons Math) but VQE requires tight integration with simulation via JNI.

**Bundle size:** ~90-130 MB (JVM runtime + Compose/Skia + native simulation libraries + LWJGL + gate library)

**Memory:** ~300-600 MB base + state vector (JVM heap + Compose/Skia + native simulation data + LWJGL GPU buffers)

**Pros:**
- Apache Commons Math provides mature classical optimizers for VQE/QAOA loops
- Compose Canvas suitable for rendering quantum circuit diagrams with interactive editing
- Kotlin coroutines enable clean structuring of simulation jobs with progress reporting and cancellation
- Cross-platform deployment covers macOS, Windows, and Linux

**Cons:**
- JVM memory overhead reduces qubits simulable by 1-2 compared to native implementations
- JNI overhead for simulation calls during VQE optimization loops (hundreds of circuit evaluations) adds measurable latency
- JVM GC pauses disrupt real-time Bloch sphere animation during gate-by-gate circuit execution
- Complex number support in JVM less ergonomic than Rust's num-complex crate

---

## Comparison Matrix

| Dimension | Electron | Tauri | Flutter | Swift | Kotlin MP |
|-----------|----------|-------|---------|-------|-----------|
| Dev effort (weeks) | 16 | 18 | 22 | 14 | 22 |
| Bundle size | 280 MB | 50 MB | 95 MB | 35 MB | 110 MB |
| Memory usage | 600 MB | 180 MB | 380 MB | 140 MB | 450 MB |
| Startup time | 2.8s | 0.9s | 1.8s | 0.5s | 2.5s |
| Native feel | 6/10 | 7/10 | 6/10 | 10/10 | 6/10 |
| Offline capability | 8/10 | 9/10 | 8/10 | 9/10 | 8/10 |
| GPU/compute access | 6/10 | 9/10 | 4/10 | 10/10 | 5/10 |
| Cross-platform | 9/10 | 9/10 | 8/10 | 2/10 | 8/10 |
| Best fit score | 7/10 | 9/10 | 5/10 | 7/10 | 5/10 |

---

## Recommended Framework

**Tauri** is the best fit for QubitStudio.

**Rationale:** Quantum circuit simulation is uniquely memory-bound — every byte of runtime overhead directly reduces the number of qubits that can be simulated. Tauri's minimal memory footprint (100-250 MB base vs Electron's 400-800 MB) translates to 2-3 additional qubits of simulation capacity, which is transformative given the exponential scaling. The Rust backend provides GPU-accelerated state vector operations via wgpu compute shaders, native-speed circuit optimization, and zero-overhead integration between simulation and visualization. Cross-platform support is important since quantum computing researchers use Linux, macOS, and Windows.

**Runner-up:** Electron offers the Monaco code editor and richer visualization ecosystem, making it the better choice if the IDE experience (QASM editing, debugging, language server integration) is prioritized over simulation performance. For teams that view QubitStudio primarily as a quantum IDE with simulation as secondary, Electron's developer tooling advantages are compelling.

---

## Monetization (Desktop)

- **Freemium model** with free tier (up to 20 qubits simulation, basic gates, no hardware access) and Pro tier ($49/month for 30+ qubit GPU simulation, noise modeling, hardware submission, circuit optimization)
- **Quantum hardware credits** sold as prepaid bundles for submitting circuits to IBM Quantum, IonQ, and Rigetti through unified QubitStudio interface ($0.01-10.00/circuit depending on qubit count and shots)
- **Enterprise/academic licenses** with team circuit sharing, collaborative design, priority hardware queue access, and dedicated support ($500-2,000/month)
- **Premium circuit library** with verified implementations of quantum algorithms (Shor's, Grover's, VQE ansatze, error correction codes) available as purchasable modules ($20-100 per algorithm pack)
- **Cloud simulation burst** for circuits exceeding local GPU capacity (35+ qubits) using distributed tensor network simulation at $5-50/simulation depending on complexity

---

## MVP Scope (Desktop)

**Core MVP (8 weeks):**

| Week | Milestone |
|------|-----------|
| 1-2 | Visual circuit builder with drag-and-drop gate placement on qubit wires, supporting standard gates (H, X, Y, Z, CNOT, T, S, Rx, Ry, Rz), circuit save/load in OpenQASM 2.0 format |
| 3-4 | State vector simulator (CPU) supporting up to 20 qubits with gate-by-gate execution, measurement sampling, amplitude visualization (bar chart), and Bloch sphere display for single-qubit states |
| 5-6 | GPU-accelerated simulation via wgpu compute shaders extending capacity to 25-30 qubits, measurement histogram visualization, expectation value computation for Pauli observables |
| 7 | OpenQASM code editor (CodeMirror) with syntax highlighting and bidirectional sync with visual circuit builder, circuit optimization passes (gate cancellation, single-qubit gate merging) |
| 8 | IBM Quantum hardware integration for submitting circuits and retrieving results, testing across circuit sizes and gate types, cross-platform packaging for macOS/Linux/Windows |

**Post-MVP:** Density matrix simulator with noise models, advanced circuit optimization (routing, template matching), IonQ/Rigetti/Braket hardware integration, VQE/QAOA optimization loops with classical optimizers, quantum error correction code simulation, parameterized circuit support with gradient computation, quantum machine learning circuit templates, multi-qubit Bloch sphere visualization (Q-sphere), circuit equivalence checking, pulse-level control simulation, tensor network simulation for 40+ qubits, Qiskit/Cirq import/export, collaborative circuit design with real-time sharing, educational mode with step-by-step gate explanations.
