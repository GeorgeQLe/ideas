# SynthBio Desktop — 5-Framework Comparison

## Desktop Rationale

- **Local Genetic Circuit Simulation:**
  Stochastic simulation of genetic circuits (gene expression, regulatory networks,
  metabolic pathways) requires solving chemical master equations or running Gillespie
  SSA across thousands of molecular species. Desktop execution enables continuous
  iteration on circuit designs without cloud latency, with simulation state persisted
  locally between sessions.

- **SBOL Visualization and Design:**
  Synthetic Biology Open Language (SBOL) visual diagrams require precise, interactive
  rendering of genetic parts (promoters, RBS, CDS, terminators) with drag-and-drop
  circuit assembly. Desktop GPU-accelerated vector rendering ensures smooth
  manipulation of complex multi-gene constructs with hundreds of annotated parts.

- **GPU-Accelerated Stochastic Simulation:**
  Monte Carlo ensemble simulations of genetic circuits demand massive parallelism —
  running 10,000+ SSA trajectories simultaneously to characterize circuit behavior
  distributions. Desktop GPU compute shaders can parallelize these independent
  trajectories across thousands of GPU cores, achieving orders-of-magnitude speedup
  over sequential CPU execution.

- **Large Plasmid and Parts Databases:**
  Local databases of genetic parts (iGEM Registry: 20,000+ parts), plasmid maps,
  codon tables, and organism-specific regulatory elements require fast indexed access.
  Desktop SQLite databases with full-text search provide instant part lookup without
  network dependency.

- **Offline Design Capability:**
  Synthetic biology researchers frequently work in wet labs, biosafety cabinets, and
  teaching environments where network access is restricted or unreliable.
  Desktop-native design tools ensure uninterrupted circuit design, simulation,
  and protocol generation regardless of connectivity.

---

## Desktop-Specific Features

- GPU-accelerated Gillespie SSA engine running 10,000+ parallel stochastic
  trajectories via wgpu compute shaders for ensemble circuit behavior characterization
- Interactive SBOL visual editor with drag-and-drop genetic part assembly, real-time
  design rule checking, and GPU-accelerated vector rendering of complex multi-gene
  constructs
- Local genetic parts database with full-text search across iGEM Registry, Addgene,
  and custom institutional part libraries with offline access
- Deterministic ODE solver for rapid mean-field circuit analysis using Rust-native
  adaptive Runge-Kutta methods with GPU-accelerated Jacobian computation
- Plasmid map viewer with interactive annotation, restriction site highlighting, and
  GPU-rendered circular/linear map visualization supporting megabase-scale sequences
- Codon optimization engine with organism-specific codon usage tables and GPU-parallel
  sequence optimization across multiple objectives (CAI, tAI, mRNA stability)
- Genetic circuit design rule checking (DRC) engine validating part compatibility,
  insulation, and context-dependent effects in real-time as designs are modified
- Protocol generation engine converting circuit designs into laboratory protocols with
  automated DNA assembly strategy selection (Gibson, Golden Gate, MoClo)
- Metabolic flux analysis module with GPU-accelerated linear programming solver for
  constraint-based modeling of engineered metabolic pathways
- Native file system integration for importing/exporting SBOL, GenBank, FASTA, and
  SnapGene formats with batch processing support

---

## Shared Packages

### core (Rust)

- Stochastic simulation engine implementing Gillespie SSA, tau-leaping, and hybrid
  ODE/SSA methods with wgpu compute shader backends for massively parallel ensemble
  simulations across GPU workgroups
- Genetic circuit compiler transforming SBOL-compliant circuit descriptions into
  simulation-ready reaction networks with automatic species enumeration and rate
  constant assignment
- ODE solver suite (explicit/implicit Runge-Kutta, BDF methods) with adaptive
  time-stepping and GPU-accelerated Jacobian evaluation for deterministic circuit
  analysis
- Sequence analysis toolkit including codon optimization, restriction site detection,
  primer design, and mRNA secondary structure prediction using dynamic programming
  algorithms
- Metabolic modeling engine implementing flux balance analysis (FBA), flux variability
  analysis (FVA), and minimal cut set enumeration with GPU-accelerated linear
  programming
- Design rule checking engine validating genetic part compatibility, transcriptional
  insulation, RBS Calculator predictions, and terminator efficiency in assembled
  constructs

### api-client (Rust)

- Cloud sync client for sharing genetic circuit designs and simulation results across
  team members with conflict resolution for concurrent edits to shared constructs
- External database connectors for querying iGEM Registry, Addgene, NCBI GenBank,
  and UniProt with local caching and incremental sync
- Telemetry and crash reporting with anonymized simulation performance metrics and
  circuit complexity statistics
- License validation and feature-flag management supporting academic vs. commercial
  tier differentiation
- Integration endpoints for importing designs from Benchling, SnapGene, and other
  synthetic biology platforms via API

### data (SQLite)

- Genetic parts database with full-text search indexing across part name, description,
  sequence, and functional annotations, supporting offline access to 20,000+
  characterized parts
- Simulation results store with compressed time-series data for ensemble trajectories,
  enabling instant replay and statistical analysis of completed stochastic simulations
- Project workspace persistence tracking circuit designs, simulation configurations,
  part palette selections, and visualization layout states across sessions
- Organism database storing chassis-specific parameters (codon usage tables, promoter
  strengths, RBS efficiencies, growth rates) for model organism panel

---

## Framework Implementations

### Electron

**Architecture:**
Multi-process Chromium architecture with main process coordinating Rust simulation
engine via N-API bindings. Renderer processes host React-based SBOL visual editor
using SVG/Canvas for circuit diagrams and WebGL2 for GPU-accelerated visualization
of simulation results. Compute-intensive SSA trajectories dispatched to Rust sidecar.

**Tech stack:**
React 18 + TypeScript, D3.js for SBOL visual rendering, Plotly for simulation result
plots, Three.js for 3D molecular visualization, napi-rs for Rust FFI, better-sqlite3
for parts database, electron-builder for packaging.

**Native module needs:**
Rust stochastic simulation engine as native Node addon via napi-rs, wgpu compute
runtime for GPU-accelerated SSA ensembles, SQLite native binding for parts database,
optional OpenBabel binding for molecular structure rendering.

**Bundle size:** ~290-370 MB
**Memory:** ~380-600 MB

**Pros:**
- Rich ecosystem of bioinformatics JavaScript libraries (BioJS, Ideogram.js)
  accelerates development of sequence and genome visualization components
- SVG rendering in Chromium provides high-quality SBOL visual diagrams with mature
  tooling for interactive element manipulation
- Large developer pool familiar with React simplifies hiring for UI-intensive
  genetic circuit editor development
- Established patterns for complex desktop apps (VS Code model) guide architecture
  of multi-panel design environments

**Cons:**
- Chromium memory overhead competes with simulation engine for RAM, limiting
  ensemble sizes for stochastic simulation
- WebGL2 compute capabilities insufficient for high-throughput SSA parallelization —
  requires Rust sidecar for GPU compute
- IPC serialization overhead significant when streaming 10,000+ ensemble trajectories
  from simulation engine to visualization layer
- Large bundle size problematic for academic users on managed workstations with
  limited disk quotas

### Tauri

**Architecture:**
Rust backend hosting stochastic simulation engine, genetic compiler, and sequence
analysis toolkit directly in the application process. wgpu compute shaders execute
SSA ensembles natively from Rust. Frontend via system WebView with lightweight SBOL
visual editor. Tauri IPC streams simulation results to frontend with minimal overhead.

**Tech stack:**
Rust backend with wgpu for GPU compute, Svelte or Solid.js frontend, SVG-based SBOL
editor in WebView, rusqlite for parts database, serde for efficient IPC serialization,
Tauri commands for simulation dispatch.

**Native module needs:**
wgpu runtime for GPU compute shader dispatch (SSA ensemble simulation, codon
optimization), system WebView dependency, optional SBOL-native Rust parser.

**Bundle size:** ~30-50 MB
**Memory:** ~110-200 MB

**Pros:**
- Rust-native SSA engine runs without FFI boundary — compute shaders dispatched
  directly via wgpu for maximum GPU utilization on ensemble simulations
- Minimal memory footprint (110-200 MB) leaves maximum RAM for large ensemble
  simulations and parts database caching
- Tiny bundle size ideal for academic distribution where users may have constrained
  disk and bandwidth
- Rust's memory safety guarantees prevent buffer overflows in sequence processing
  and simulation state management

**Cons:**
- System WebView SVG rendering performance varies across platforms — SBOL diagrams
  with 500+ parts may render differently on Linux WebKitGTK
- Smaller ecosystem of Rust-native bioinformatics libraries compared to
  Python/JavaScript
- Complex SBOL visual editor development more challenging without Chromium
  DevTools-level debugging
- WebView limitations may constrain advanced visualization features like 3D
  molecular structure rendering

### Flutter Desktop

**Architecture:**
Dart UI layer with custom canvas-based SBOL visual editor. Platform channels bridge
to Rust simulation engine compiled as dynamic library. Skia rendering handles smooth
vector graphics for circuit diagrams. GPU compute dispatched via Rust library
through dart:ffi.

**Tech stack:**
Dart/Flutter with custom Canvas/Skia-based SBOL editor, dart:ffi for Rust simulation
library binding, fl_chart for simulation result visualization, sqflite for parts
database, CustomPainter for genetic part rendering.

**Native module needs:**
Rust simulation library as .dylib/.dll/.so with C FFI, platform-specific GPU compute
(wgpu via Rust), custom Skia shaders for circuit diagram rendering effects.

**Bundle size:** ~60-85 MB
**Memory:** ~220-380 MB

**Pros:**
- Skia rendering engine provides smooth, hardware-accelerated vector graphics ideal
  for interactive SBOL circuit diagram manipulation
- Custom Canvas API enables pixel-perfect genetic part rendering with consistent
  appearance across all platforms
- Hot reload dramatically accelerates UI iteration during development of complex
  multi-panel circuit editor
- Single Dart codebase for UI across Windows/macOS/Linux reduces platform-specific
  maintenance burden

**Cons:**
- Dart FFI marshalling adds overhead for streaming high-frequency simulation data
  (ensemble trajectories) from Rust engine
- No existing Flutter packages for bioinformatics — SBOL editor, sequence viewer,
  and plasmid map components must be built from scratch
- GPU compute dispatch requires Rust intermediate layer, adding architectural
  complexity compared to direct wgpu access
- Flutter desktop text rendering may not match native precision needed for sequence
  annotation and codon display

### Swift/SwiftUI (macOS)

**Architecture:**
Native SwiftUI application with Metal compute shaders for GPU-accelerated SSA
ensemble simulation. AppKit-based SBOL visual editor using Core Graphics for
high-fidelity genetic part rendering. Rust simulation core linked as static library
for cross-platform algorithm portability.

**Tech stack:**
SwiftUI for UI chrome, AppKit/NSView for SBOL canvas editor, Metal compute for GPU
SSA simulation, Metal rendering for real-time simulation visualization, Swift-Rust
bridge via C FFI, GRDB.swift for SQLite parts database.

**Native module needs:**
Metal compute shader compilation for SSA kernel parallelization, Rust simulation
library as static .a archive, Core Graphics for SBOL vector rendering.

**Bundle size:** ~22-38 MB
**Memory:** ~90-160 MB

**Pros:**
- Metal compute on Apple Silicon provides exceptional GPU throughput for parallel SSA
  ensembles — M-series unified memory eliminates trajectory data transfer overhead
- Native macOS rendering delivers pixel-perfect SBOL diagrams through Core Graphics
  with full Retina display support
- Minimal resource footprint maximizes available GPU compute and RAM for large-scale
  stochastic simulations
- Tight macOS integration enables features like Quick Look for SBOL files, Spotlight
  indexing of parts databases, and Continuity for lab-to-desk workflows

**Cons:**
- macOS-only excludes Windows and Linux users, which represent the majority of
  academic bioinformatics workstations
- SwiftUI canvas-based rendering requires significant custom development for complex
  SBOL visual editor interactions
- Limited Swift bioinformatics ecosystem — most synthetic biology tools target Python
- No cross-platform path without maintaining separate codebases, duplicating SBOL
  editor development effort

### Kotlin Multiplatform

**Architecture:**
Compose Multiplatform UI with shared Kotlin business logic for circuit design
management and simulation orchestration. Rust SSA engine accessed via JNI on JVM
targets. SBOL visual editor built with Compose Canvas API. GPU compute via
LWJGL/Vulkan bindings.

**Tech stack:**
Kotlin/Compose Multiplatform for UI, JNI bridge to Rust simulation library, Compose
Canvas for SBOL visual editor, LWJGL for GPU rendering, SQLDelight for parts
database, Kotlin coroutines for async simulation management.

**Native module needs:**
Rust simulation library with JNI-compatible C FFI, LWJGL native binaries for GPU
rendering, platform-specific Vulkan/OpenGL drivers, SQLite native library.

**Bundle size:** ~85-120 MB
**Memory:** ~280-450 MB

**Pros:**
- Kotlin coroutines provide elegant async orchestration of long-running ensemble
  simulations with cancellation, progress reporting, and result aggregation
- Compose Canvas API supports custom rendering of SBOL diagrams with hardware
  acceleration across platforms
- JVM ecosystem offers access to BioJava and CDK libraries for supplementary
  sequence analysis and molecular operations
- Strong type system helps manage complexity of genetic circuit data models with
  sealed classes for part type hierarchies

**Cons:**
- JVM memory overhead (280-450 MB) significantly limits available resources for
  large ensemble simulations compared to native alternatives
- JNI bridge to Rust SSA engine adds complexity and potential for memory management
  issues during high-throughput trajectory streaming
- GPU compute via LWJGL less ergonomic than native wgpu for compute shader
  development
- JVM garbage collection pauses can cause visible stutters during real-time
  simulation visualization updates

---

## Comparison Matrix

| Dimension | Electron | Tauri | Flutter | Swift | Kotlin MP |
|-----------|----------|-------|---------|-------|-----------|
| Dev effort (weeks) | 15 | 13 | 17 | 11 | 19 |
| Bundle size | 330 MB | 40 MB | 72 MB | 30 MB | 102 MB |
| Memory usage | 490 MB | 155 MB | 300 MB | 125 MB | 365 MB |
| Startup time | 3.4s | 0.9s | 1.9s | 0.5s | 3.0s |
| Native feel | 5/10 | 7/10 | 6/10 | 10/10 | 6/10 |
| Offline capability | 8/10 | 9/10 | 8/10 | 9/10 | 8/10 |
| GPU/compute access | 5/10 | 9/10 | 6/10 | 10/10 | 5/10 |
| Cross-platform | 9/10 | 8/10 | 8/10 | 2/10 | 7/10 |
| Best fit score | 6/10 | 9/10 | 6/10 | 7/10 | 5/10 |

---

## Recommended Framework

**Tauri** is the best fit for SynthBio Desktop.

**Rationale:**
Synthetic biology simulation demands GPU-parallel stochastic computation (10,000+ SSA
trajectories) and efficient memory utilization for large parts databases — Tauri's
Rust-native backend with direct wgpu compute shader access provides the optimal
combination of GPU throughput and minimal memory overhead. The 40 MB bundle size is
critical for academic distribution where researchers may have limited disk quotas and
bandwidth, while cross-platform support ensures reach across the diverse OS landscape
of university research labs.

**Runner-up:**
Swift/SwiftUI for a macOS-focused product targeting computational biology labs with
Apple Silicon workstations, where Metal compute provides unmatched SSA ensemble
throughput on unified memory architecture.

---

## Monetization (Desktop)

- **Academic freemium:** Free tier for academic researchers with basic SSA simulation
  (up to 1,000 trajectories, 50 species), paid Academic Pro ($19/month) unlocking
  unlimited GPU-accelerated ensembles, advanced circuit analysis, and metabolic
  modeling
- **Commercial license:** Annual subscription ($79/month) for industry synthetic
  biology teams with full simulation capabilities, commercial-use rights, priority
  support, and advanced features (automated DNA assembly design, regulatory
  compliance tools)
- **Parts database premium:** Curated, experimentally-validated parts database
  subscription ($15/month) with characterized performance data, context-dependent
  measurements, and quarterly updates from literature mining
- **Enterprise/pharma license:** Site-wide deployment ($5,000-25,000/year) for
  pharmaceutical and agricultural biotech companies with SSO integration, audit
  logging, IP-protected design vaults, and custom organism chassis support
- **Educational institution license:** Discounted multi-seat license ($2,000/year
  for 50 seats) targeting university synthetic biology courses with shared parts
  libraries, assignment templates, and instructor dashboards

---

## MVP Scope (Desktop)

**Core MVP (8 weeks):**

| Week | Milestone |
|------|-----------|
| 1-2 | Rust stochastic simulation engine (Gillespie SSA with wgpu compute shader parallelization), basic Tauri application shell with project management, SQLite schema for parts database and simulation results, and SBOL data model implementation |
| 3-4 | Interactive SBOL visual editor with drag-and-drop genetic part placement (promoter, RBS, CDS, terminator glyphs), basic deterministic ODE solver for rapid circuit analysis, and genetic parts database with search and browse functionality |
| 5-6 | GPU-accelerated ensemble simulation with real-time progress tracking, simulation result visualization (time-series plots, distribution histograms, phase portraits), and import/export support for SBOL and GenBank file formats |
| 7 | Design rule checking engine validating part compatibility and basic context effects, codon optimization tool with organism-specific usage tables, and sequence viewer with annotation and restriction site highlighting |
| 8 | End-to-end testing of simulation accuracy against reference SSA implementations, performance benchmarking of GPU ensemble throughput, installer packaging for Windows/macOS/Linux, and beta release with academic user feedback collection |

**Post-MVP:**
Metabolic flux analysis module, advanced SBOL visual editor with hierarchical
sub-circuit encapsulation, 3D molecular structure visualization, automated DNA
assembly strategy selection (Gibson/Golden Gate/MoClo), protocol generation engine,
integration with Benchling and SnapGene, cloud sync for team collaboration, expanded
organism chassis library, and machine learning-guided circuit optimization
recommendations.
