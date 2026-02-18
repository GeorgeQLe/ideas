# QubitStudio — Cloud Quantum Computing IDE and Simulator

## Executive Summary

QubitStudio is a cloud-native, browser-based quantum computing IDE that combines a visual circuit builder, GPU-accelerated state vector simulation (30+ qubits), density matrix noise modeling, and multi-provider quantum hardware access (IBM, IonQ, Rigetti) in a single platform. It replaces the fragmented workflow of Qiskit notebooks + IBM Quantum Lab + local simulators with an integrated development environment where quantum researchers can design circuits, simulate them, and submit to real quantum hardware — all from the browser.

---

## Problem Statement

**The pain:**
- Quantum computing development requires cobbling together multiple tools: Qiskit/Cirq for circuit definition, Jupyter for execution, IBM Quantum Lab for hardware access, custom matplotlib scripts for visualization — each with its own learning curve and authentication
- Simulating quantum circuits beyond 25 qubits requires significant computational resources (16+ GB RAM for state vectors) that most researchers' laptops cannot provide, creating a barrier to exploring meaningful algorithm sizes
- Noise simulation is critical for real-hardware algorithm design but is disconnected from circuit building — researchers define circuits in one tool and run noise models in another, making iterative optimization painful
- Multi-provider hardware access (IBM, IonQ, Rigetti, Amazon Braket) requires managing separate accounts, different SDKs, incompatible circuit formats, and separate billing — comparing hardware results is a manual spreadsheet exercise
- Quantum computing education is hampered by setup complexity: students spend the first 2 weeks of a quantum computing course installing Python, Qiskit, and Jupyter instead of learning quantum concepts

**Current workarounds:**
- Researchers use Jupyter notebooks with Qiskit for most quantum work but get no visual feedback on circuit structure, no integrated simulation, and no collaboration
- IBM Quantum Lab provides a browser-based Jupyter environment but is limited to IBM hardware, lacks visual circuit editing for complex algorithms, and has restrictive session limits
- Quirk (browser-based circuit simulator) offers excellent visual circuit building but maxes out at ~16 qubits, has no noise simulation, and cannot connect to real hardware
- Some teams use Classiq's high-level synthesis but it abstracts away circuit-level control that researchers need for algorithm development

**Market size:** The global quantum computing market is valued at approximately $1.3 billion (2024) and projected to reach $8.6 billion by 2030 (CAGR ~36%). The quantum computing software and tools segment specifically represents ~$350 million. There are an estimated 15,000+ quantum computing researchers and engineers worldwide, with 300+ universities offering quantum computing courses and the number of quantum software developers growing at 40% annually.

---

## Target Users

### Primary Personas

**1. Dr. Anika Sharma — Quantum Algorithm Researcher at a National Lab**
- Develops variational quantum algorithms (VQE, QAOA) for quantum chemistry applications
- Uses Qiskit + Jupyter + IBM Quantum Lab across 3 separate tabs, constantly copying circuit definitions between tools
- Needs: an integrated IDE where she can visually build parameterized circuits, simulate with noise models, and submit to IBM/IonQ hardware without leaving the tool

**2. Prof. David Morales — Quantum Computing Course Instructor**
- Teaches quantum computing to 40 undergraduate and 20 graduate students per semester
- Students struggle with Qiskit installation, conda environment management, and notebook setup — 30% drop the course in week 1
- Needs: a zero-install browser-based tool where students can build circuits visually, see Bloch sphere animations, and run simulations without Python setup

**3. Sarah Kim — Quantum Software Engineer at a Quantum Startup**
- Develops quantum machine learning algorithms targeting near-term hardware
- Needs to test algorithms on IBM, IonQ, and Rigetti hardware to find optimal hardware-algorithm pairings
- Currently maintains 3 separate Python environments with different SDKs and submits jobs via CLI
- Needs: a unified platform with multi-provider hardware access, noise-aware compilation, and cross-provider result comparison

### Secondary Personas
- Quantum computing enthusiasts and self-learners who want visual, interactive tools to understand quantum gates and entanglement
- Enterprise R&D teams exploring quantum computing use cases who need accessible proof-of-concept tools without hiring quantum specialists
- Quantum hardware companies who want to provide a user-friendly frontend for their hardware access APIs

---

## Solution Overview

QubitStudio is a browser-based quantum computing IDE that:
1. Provides a visual quantum circuit builder where users drag and drop gates onto qubit wires, connect multi-qubit operations, and define parameterized circuits with real-time state visualization (Bloch spheres, amplitude bar charts, phase wheels)
2. Runs GPU-accelerated quantum circuit simulation in the cloud, supporting state vector simulation up to 30+ qubits and density matrix noise simulation with realistic hardware noise models (depolarizing, amplitude damping, readout error)
3. Connects to multiple quantum hardware providers (IBM Quantum, IonQ, Rigetti, Amazon Braket) through a unified interface, handling circuit transpilation, job submission, and result retrieval transparently
4. Generates and synchronizes code (Qiskit, Cirq, OpenQASM 3.0) bidirectionally with the visual circuit, so users can switch between visual and code-based editing seamlessly
5. Supports team collaboration with shared projects, version-controlled circuits, and experiment tracking across simulation and hardware runs

---

## Core Features

### F1: Visual Circuit Builder
- Drag-and-drop gate palette organized by type: single-qubit (H, X, Y, Z, S, T, Rx, Ry, Rz), multi-qubit (CNOT, CZ, SWAP, Toffoli), measurement, barrier, and custom unitary
- Qubit wire canvas with automatic layout, gate alignment, and depth counting
- Parameterized gates: define symbolic parameters (θ, φ, λ) with slider controls for interactive exploration
- Gate decomposition view: expand high-level gates (QFT, Grover oracle) into native gate sets
- Circuit templates: Bell state, GHZ state, Quantum Fourier Transform, Grover's algorithm, VQE ansatz, QAOA, quantum teleportation
- Undo/redo with full history, copy-paste of circuit blocks, and keyboard shortcuts
- Composite gate creation: select a group of gates and save as a reusable custom gate with custom icon

### F2: GPU-Accelerated State Vector Simulator
- Cloud GPU simulation supporting up to 30 qubits on consumer-tier and 35+ qubits on enterprise-tier instances
- Real-time simulation: circuit state updates instantly as gates are added or parameters are modified
- Configurable precision: single-precision (32-bit) for speed or double-precision (64-bit) for accuracy
- Statevector inspection: full amplitude display with magnitude and phase for all 2^n basis states
- Mid-circuit measurement support with classical conditional gates
- Simulation caching: previously computed states are cached so parameter changes only re-simulate from the point of modification
- Batch simulation: sweep a parameter across a range and compute expectation values

### F3: Density Matrix and Noise Simulation
- Density matrix simulation for mixed-state circuits with noise channels
- Noise models: depolarizing channel, amplitude damping, phase damping, bit-flip, phase-flip, readout error
- Hardware-specific noise models: load calibration data from IBM Quantum, IonQ, and Rigetti to simulate with realistic device noise
- Custom noise model builder: define per-gate error rates, T1/T2 times, crosstalk matrices, and measurement error probabilities
- Noise-aware compilation: automatically insert error mitigation techniques (zero-noise extrapolation, probabilistic error cancellation) based on noise model
- Fidelity metrics: state fidelity, process fidelity, and diamond norm distance between ideal and noisy execution

### F4: Quantum State Visualization
- Bloch sphere: animated single-qubit state visualization showing gate operations as rotations in real-time
- Multi-qubit Bloch sphere array: display all qubits simultaneously with entanglement indicators
- Amplitude bar chart: magnitude of each basis state amplitude with phase coloring
- Phase wheel: circular plot showing relative phases between basis states
- Q-sphere: multi-qubit state visualization as a sphere with basis states at different latitudes
- Density matrix heatmap: visualize the density matrix as a 2D color-coded grid with real and imaginary components
- Entanglement entropy plot: von Neumann entropy across bipartitions showing entanglement structure

### F5: Multi-Provider Hardware Access
- Unified hardware dashboard: view available quantum computers from IBM, IonQ, Rigetti, and Amazon Braket with current queue depths, qubit counts, and calibration data
- One-click job submission: select a backend, QubitStudio transpiles the circuit to the native gate set, optimizes for device topology, and submits
- Job queue manager: track submitted jobs across all providers with status, queue position, and estimated completion time
- Result comparison: side-by-side comparison of results from simulator vs. multiple hardware backends with fidelity metrics
- Hardware credential management: securely store API keys for all providers with OAuth integration where available
- Cost estimation: display estimated cost per circuit execution on each hardware provider before submission

### F6: Code Editor and Transpilation
- Integrated code editor with syntax highlighting and autocompletion for Qiskit, Cirq, and OpenQASM 3.0
- Bidirectional sync: edits in the code editor update the visual circuit and vice versa
- Circuit transpilation: compile abstract circuits to hardware-native gate sets (IBM: CX, ID, RZ, SX; IonQ: GPI, GPI2, MS; Rigetti: CZ, RX, RZ)
- Optimization passes: gate cancellation, commutation-based optimization, SWAP routing for device connectivity
- Circuit statistics: gate count by type, circuit depth, CNOT count, estimated execution time on selected hardware
- Export: download circuits as Qiskit Python, Cirq Python, OpenQASM 3.0, QASM 2.0, Quil, or IonQ JSON

### F7: Experiment Tracker
- Every circuit execution (simulation or hardware) automatically logged with circuit version, parameters, backend, and results
- Measurement histogram: bar chart of measurement outcomes with statistical analysis (expectation values, variance)
- Experiment comparison table: side-by-side comparison of results across parameter settings, noise models, and hardware backends
- Quantum state tomography: reconstruct the output state from measurement statistics (maximum likelihood estimation)
- Error analysis: compare ideal simulation to hardware results, identify dominant error sources
- Tagging and organization: label experiments by algorithm type, research question, or paper reference

### F8: Variational Algorithm Studio
- VQE (Variational Quantum Eigensolver) workflow: define Hamiltonian, choose ansatz, select classical optimizer, run optimization loop
- QAOA (Quantum Approximate Optimization Algorithm) setup: define cost function, number of layers, and initial parameters
- Classical optimizer integration: Cobyla, SPSA, Adam, L-BFGS with convergence visualization
- Parameter landscape visualization: 2D energy surface plots for single and paired parameter sweeps
- Convergence dashboard: live plot of objective function value across optimization iterations
- Hardware-aware VQE: optimization loop that alternates between hardware execution and parameter updates

### F9: Quantum Error Correction Studio
- Surface code visualizer: display logical qubits on 2D surface code lattices with syndrome measurements
- Error correction code templates: Steane [[7,1,3]], surface code, repetition code, Shor code
- Syndrome decoding: run minimum-weight perfect matching decoder on syndrome measurement results
- Logical error rate estimation: Monte Carlo simulation of error correction performance under noise
- Code distance explorer: visualize how logical error rate decreases with increasing code distance

### F10: Educational Mode
- Guided tutorials: step-by-step lessons on quantum gates, superposition, entanglement, and basic algorithms
- Gate animation: slow-motion visualization of how each gate transforms the quantum state
- Quiz mode: interactive questions about circuit output with hints and explanations
- Concept library: visual explanations of quantum concepts (superposition, interference, decoherence) with interactive demos
- Progress tracking: curriculum path from beginner to advanced with badges and completion certificates

### F11: Collaboration and Sharing
- Project-based workspaces with role-based access (viewer, editor, admin)
- Circuit version control: track modifications to circuits with visual diff showing gates added/removed/modified
- Inline commenting on circuit gates, measurement results, and experiments
- Share links: read-only URLs for circuit diagrams and results dashboards
- Public circuit gallery: publish circuits with descriptions and educational notes
- Fork: duplicate any public project as a starting point

### F12: API and Integration
- REST API for programmatic circuit creation, simulation, and hardware job submission
- Jupyter notebook extension: run QubitStudio circuits from Jupyter and display results inline
- GitHub integration: commit circuit files (QASM) to repositories, trigger simulation on push
- Webhook notifications for hardware job completion
- Batch simulation API: submit parameterized circuit sweeps for large-scale analysis
- Plugin system for custom gate definitions, noise models, and classical optimizers

---

## Technical Architecture

### System Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        Browser Client                            │
│  ┌────────────┐  ┌────────────┐  ┌───────────┐  ┌────────────┐ │
│  │  Circuit    │  │   State    │  │  Code     │  │ Hardware   │ │
│  │  Builder    │  │   Viz      │  │  Editor   │  │ Dashboard  │ │
│  │ (Canvas)    │  │ (WebGL)    │  │ (Monaco)  │  │ (React)    │ │
│  └──────┬──────┘  └──────┬─────┘  └─────┬─────┘  └─────┬──────┘ │
│         │               │              │              │         │
│         └───────────────┼──────────────┼──────────────┘         │
│                         ▼                                       │
│                State Management (Zustand)                        │
│                         │                                       │
└─────────────────────────┼───────────────────────────────────────┘
                          │ HTTPS / WebSocket
                          ▼
                ┌─────────────────────┐
                │   API Gateway       │
                │  (Python / FastAPI) │
                └────┬───────┬────────┘
                     │       │
          ┌──────────┘       └──────────┐
          ▼                             ▼
┌─────────────────┐           ┌─────────────────────┐
│   PostgreSQL    │           │  Simulation Engine   │
│  (Users, Proj,  │           │                      │
│   Circuits,     │           │  ┌────────────────┐  │
│   Experiments)  │           │  │ State Vector   │  │
└────────┬────────┘           │  │ Sim (GPU/CUDA) │  │
         │                    │  └────────────────┘  │
         ▼                    │  ┌────────────────┐  │
┌─────────────────┐           │  │ Density Matrix │  │
│     Redis       │           │  │ Sim (GPU)      │  │
│  (Sessions,     │◄──────────│  └────────────────┘  │
│   Job Queue,    │           │  ┌────────────────┐  │
│   Sim Cache)    │           │  │ Transpiler     │  │
└─────────────────┘           │  │ (Qiskit-based) │  │
                              │  └────────────────┘  │
                              └─────────────────────┘
                                        │
                        ┌───────────────┼───────────────┐
                        ▼               ▼               ▼
                ┌──────────────┐ ┌────────────┐ ┌────────────┐
                │ IBM Quantum  │ │   IonQ     │ │  Rigetti   │
                │ API          │ │   API      │ │  API       │
                └──────────────┘ └────────────┘ └────────────┘
                                        │
                                        ▼
                              ┌─────────────────┐
                              │  Object Storage │
                              │  (S3/R2)        │
                              │  Results, Logs  │
                              └─────────────────┘
```

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend Framework | React 19 + TypeScript, Vite build system |
| Circuit Canvas | Custom Canvas2D renderer with SVG gate icons and interactive wire layout |
| State Visualization | Three.js for Bloch spheres, D3 for bar charts and phase wheels, Plotly for Q-sphere |
| Code Editor | Monaco Editor with custom OpenQASM 3.0 and Qiskit language support |
| State Management | Zustand for client state, React Query for server state |
| Backend API | Python 3.12, FastAPI, Pydantic v2 |
| Simulation Engine | cuStateVec (NVIDIA cuQuantum) for GPU state vector simulation, custom density matrix engine |
| Transpilation | Qiskit transpiler for gate decomposition and routing, custom pass manager |
| Hardware Adapters | Qiskit Runtime (IBM), IonQ REST API, Rigetti QCS, Amazon Braket SDK |
| Database | PostgreSQL 16 with JSONB for circuit definitions and experiment results |
| Cache / Queue | Redis 7 for simulation caching, job queuing, and session management |
| Object Storage | Cloudflare R2 for circuit snapshots, simulation results, and exported files |
| GPU Compute | Kubernetes (EKS) with NVIDIA A10G/A100 nodes for simulation |
| Auth | JWT-based auth with OAuth (Google, GitHub), hardware provider API key vault |
| Monitoring | Grafana + Prometheus, Sentry, PostHog for product analytics |

### Data Model

```
User
├── id (uuid), email, name, avatar_url
├── auth_provider, plan (free/pro/team/enterprise)
├── hardware_credentials (encrypted JSONB — IBM API key, IonQ key, Rigetti token)
└── created_at, updated_at

Organization
├── id (uuid), name, slug, owner_id → User
├── plan, seat_count, billing_email
└── sim_quota_hours_monthly, hardware_budget_usd

Project
├── id (uuid), org_id → Organization
├── name, description, visibility (private/org/public)
├── tags[], created_by → User
└── forked_from → Project (nullable), created_at, updated_at

Circuit
├── id (uuid), project_id → Project
├── name, version (int), parent_version_id → Circuit (nullable)
├── circuit_data (JSONB — gates, qubits, parameters, measurements)
├── qubit_count, gate_count, depth
├── qasm_code (text — OpenQASM 3.0 representation)
├── message (version commit message)
└── created_by → User, created_at

Experiment
├── id (uuid), project_id → Project, circuit_id → Circuit
├── backend_type (simulator/ibm/ionq/rigetti/braket)
├── backend_name (e.g., "ibm_sherbrooke", "ionq_aria", "state_vector_gpu")
├── status (queued/running/completed/failed)
├── config (JSONB — shots, noise_model, transpilation_level, optimization_level)
├── results (JSONB — counts, statevector, expectation_values)
├── fidelity_vs_ideal (float, nullable)
├── cost_usd (decimal), compute_time_seconds
└── created_by → User, submitted_at, completed_at

NoiseModel
├── id (uuid), name, source (custom/ibm_calibration/ionq_calibration)
├── config (JSONB — gate_errors, T1, T2, readout_errors, crosstalk)
├── backend_reference (nullable — linked hardware backend)
└── created_by → User, created_at

VQERun
├── id (uuid), project_id → Project, circuit_id → Circuit
├── hamiltonian (JSONB — Pauli terms and coefficients)
├── optimizer (cobyla/spsa/adam/lbfgs)
├── optimizer_config (JSONB), max_iterations
├── backend_type, backend_name
├── convergence_data (JSONB — energy values per iteration)
├── optimal_params (JSONB), optimal_energy (float)
├── status, started_at, completed_at
└── created_by → User

Comment
├── id (uuid), project_id → Project
├── author_id → User, body (text)
├── anchor_type (circuit_gate/experiment/vqe_run)
├── anchor_data (JSONB), resolved (boolean)
└── created_at
```

### API Design

```
Auth:
POST   /api/auth/register                   # Create account
POST   /api/auth/login                      # Login
POST   /api/auth/oauth/:provider            # OAuth (Google, GitHub)
POST   /api/auth/refresh                    # Refresh token

Projects:
GET    /api/projects                        # List projects
POST   /api/projects                        # Create project
GET    /api/projects/:id                    # Get project
PATCH  /api/projects/:id                    # Update project
DELETE /api/projects/:id                    # Delete project
POST   /api/projects/:id/fork               # Fork project

Circuits:
GET    /api/projects/:id/circuits            # List circuit versions
POST   /api/projects/:id/circuits            # Save circuit version
GET    /api/circuits/:id                     # Get circuit data
GET    /api/circuits/:id/qasm               # Get OpenQASM export
GET    /api/circuits/:id/qiskit              # Get Qiskit code export
GET    /api/projects/:id/circuits/diff/:a/:b # Diff two circuit versions
POST   /api/circuits/import/qasm             # Import from OpenQASM

Simulation:
POST   /api/circuits/:id/simulate            # Run simulation
GET    /api/simulations/:id                  # Get simulation result
POST   /api/circuits/:id/simulate/sweep      # Parameter sweep simulation
GET    /api/simulations/:id/statevector      # Get full state vector
GET    /api/simulations/:id/density-matrix   # Get density matrix

Hardware:
GET    /api/hardware/backends                # List available backends
GET    /api/hardware/backends/:id            # Get backend details/calibration
POST   /api/circuits/:id/submit              # Submit to hardware
GET    /api/jobs/:id                         # Get job status
GET    /api/jobs/:id/results                 # Get job results
POST   /api/jobs/:id/cancel                  # Cancel queued job
POST   /api/hardware/credentials             # Store provider API keys

Experiments:
GET    /api/projects/:id/experiments          # List experiments
GET    /api/experiments/:id                  # Get experiment details
GET    /api/experiments/compare               # Compare multiple experiments
POST   /api/experiments/:id/tomography        # Run state tomography

VQE/QAOA:
POST   /api/projects/:id/vqe                 # Start VQE optimization
GET    /api/vqe-runs/:id                     # Get VQE status/results
POST   /api/vqe-runs/:id/stop                # Stop optimization
POST   /api/projects/:id/qaoa                # Start QAOA optimization
GET    /api/qaoa-runs/:id                    # Get QAOA status/results

Noise Models:
GET    /api/noise-models                     # List noise models
POST   /api/noise-models                     # Create custom noise model
GET    /api/noise-models/:id                 # Get noise model
GET    /api/hardware/backends/:id/noise       # Get hardware noise model

Collaboration:
GET    /api/projects/:id/comments             # List comments
POST   /api/projects/:id/comments             # Add comment
PATCH  /api/comments/:id                     # Edit/resolve comment
POST   /api/projects/:id/share               # Generate share link
GET    /api/projects/:id/activity             # Activity feed
```

---

## UI/UX — Key Screens

### 1. Project Dashboard
- Card grid showing projects with circuit diagram thumbnails, qubit counts, and last experiment results
- Quick-create: start from templates (Bell state, Grover's search, VQE) or blank circuit
- Recent activity: experiments completed, circuits modified, hardware results received
- Hardware status widget: real-time availability and queue depths for connected quantum computers

### 2. Circuit Builder
- Full-width circuit canvas with qubit wires (horizontal lines) and gate palette on the left organized by type
- Right panel: selected gate properties (angle parameters with slider controls), circuit statistics (depth, gate count)
- Top toolbar: templates, run simulation, submit to hardware, export QASM/Qiskit, share
- Bottom panel: tabbed view switching between state visualization, code editor, and experiment results
- Real-time state visualization updating as gates are added: Bloch spheres for small circuits, amplitude bar chart for larger ones
- Wire labels showing qubit names and initial state (|0⟩ by default, configurable)

### 3. State Visualization Dashboard
- Bloch sphere array: one sphere per qubit showing current state with gate operation animation
- Amplitude bar chart: all 2^n basis states with magnitude bars and phase coloring (hue = phase angle)
- Q-sphere: 3D visualization for multi-qubit states with basis states positioned by Hamming weight
- Density matrix heatmap: 2D grid showing real and imaginary parts of ρ for noisy simulations
- Entanglement graph: node diagram showing entanglement entropy between qubit pairs
- Step-through mode: advance the circuit gate by gate and watch the state evolve

### 4. Hardware Dashboard
- Provider tabs (IBM, IonQ, Rigetti, Braket) with available backend cards showing qubit count, topology, queue depth, and connectivity map
- Job queue: list of submitted jobs with status, backend, estimated completion, and cost
- Result comparison: side-by-side measurement histograms from simulator vs. hardware with fidelity score
- Calibration data: gate error rates, T1/T2 times, and readout fidelity for the selected backend
- Cost tracker: hardware spending by provider with monthly budget alerts

### 5. VQE/QAOA Optimization Studio
- Circuit template selector with ansatz options (hardware-efficient, UCCSD, Ry-entangler)
- Hamiltonian input: manual Pauli string entry or molecular geometry → Hamiltonian conversion
- Optimization dashboard: live convergence plot, current parameters, energy estimate, iteration count
- Parameter landscape: 2D heatmap of energy vs. parameter pairs for selected parameters
- Result summary: optimal energy, optimal parameters, comparison with exact diagonalization

### 6. Educational Mode
- Step-by-step tutorial panel with instructions, circuit building area, and inline quizzes
- Gate reference: click any gate to see its matrix representation, Bloch sphere effect, and common uses
- Concept visualizations: interactive demos of superposition, entanglement, interference, and no-cloning
- Progress bar: track completion through curriculum modules (Basics → Gates → Algorithms → Error Correction)
- Achievement badges: earn badges for completing tutorials and building specific algorithms

---

## Monetization

### Free Tier
- 3 projects, up to 20 qubits simulation
- 50 simulation runs per month (state vector only, no noise)
- No hardware access (simulation only)
- Basic visualizations (Bloch sphere, amplitude chart)
- Community support
- QubitStudio branding on shared circuits

### Pro — $99/month
- Unlimited projects, up to 30 qubits simulation
- Unlimited simulation runs (state vector + density matrix noise)
- Hardware access: IBM Quantum (free backends), IonQ (pay-per-shot via QubitStudio)
- Custom noise models and hardware-specific noise simulation
- Full visualization suite (Q-sphere, density matrix, entanglement graph)
- Code generation (Qiskit, Cirq, OpenQASM)
- VQE/QAOA optimization studio
- Email support with 48-hour response

### Team — $299/month (up to 5 seats, $59/additional seat)
- Everything in Pro
- Up to 35 qubits simulation, 500 GPU-hours per month
- Hardware access: all providers (IBM, IonQ, Rigetti, Amazon Braket)
- Team collaboration with comments and version control
- Shared hardware credential management
- Experiment tracking with cross-provider comparison
- Error correction studio
- Priority support with 24-hour response

### Enterprise — Custom
- Unlimited qubits (limited by GPU memory), unlimited GPU hours
- Dedicated GPU simulation cluster
- All hardware providers with volume pricing
- SAML/SSO, audit logging, and API access
- Custom noise model calibration services
- Dedicated customer success manager and SLA
- On-premise deployment option for classified research
- White-label option for hardware providers

---

## Go-to-Market Strategy

### Phase 1: Academic Seeding (Month 1-3)
- Launch free tier on Product Hunt and quantum computing communities (Qiskit Slack, Cirq Google Group, Quantum Computing Stack Exchange)
- Partner with 20 university quantum computing courses for free Team access
- Tutorial series: "Quantum Computing from Zero" with visual circuit explanations on YouTube
- Publish open-source circuit library (50+ well-documented algorithms) on GitHub
- Present at QIP, APS March Meeting, and Qiskit Global Summer School

### Phase 2: Researcher Adoption (Month 3-6)
- Launch Pro tier with noise simulation and IBM Quantum hardware access
- Benchmark publication: simulation speed vs. Qiskit Aer, accuracy vs. analytical results
- SEO content: "IBM Quantum Lab alternative", "quantum circuit simulator online", "visual quantum computing"
- Integration with Qiskit ecosystem: import/export compatibility, shared circuit format
- Sponsor Qiskit Hackathon Global and quantum computing meetup groups

### Phase 3: Enterprise and Hardware Partners (Month 6-12)
- Launch Team and Enterprise tiers with multi-provider access and collaboration
- Partner with quantum hardware companies (IonQ, Rigetti, Quantinuum) to offer QubitStudio as a recommended frontend
- Target quantum-curious enterprise R&D departments (finance, pharma, logistics) with proof-of-concept workshops
- Build quantum error correction and fault-tolerant simulation capabilities
- Launch QubitStudio Marketplace for community-contributed circuits and noise models

### Acquisition Channels
- Organic search: "quantum circuit simulator", "quantum computing IDE", "visual quantum programming"
- University courses driving graduates into professional tiers
- Qiskit/Cirq community engagement and documentation cross-linking
- Referral program: invite colleague, both get 1 month Pro free
- Hardware provider partnership listings and co-marketing

---

## Success Metrics & KPIs

| Metric | Target (Month 6) | Target (Month 12) |
|--------|-------------------|---------------------|
| Registered users | 10,000 | 50,000 |
| Monthly active projects | 3,000 | 15,000 |
| Paying customers (Pro + Team) | 200 | 1,000 |
| MRR | $25,000 | $120,000 |
| Simulation runs per month | 100,000 | 600,000 |
| Hardware jobs submitted per month | 5,000 | 40,000 |
| Free → Paid conversion rate | 4% | 7% |
| Monthly churn rate | <5% | <3% |

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | QubitStudio Advantage |
|-----------|-----------|------------|----------------------|
| IBM Quantum Lab | Free hardware access, deep Qiskit integration, large community | IBM-only hardware, notebook-based (no visual builder), restrictive session limits | Multi-provider hardware, visual circuit builder, integrated state visualization |
| Quirk (Algassert) | Excellent visual circuit builder, instant simulation, zero setup | Max ~16 qubits, no noise simulation, no hardware access, single-user | 30+ qubits, noise simulation, hardware access, collaboration |
| Classiq | High-level quantum algorithm synthesis, hardware-agnostic | Abstracts away circuit-level control, expensive, limited simulation | Circuit-level control with visual editing, accessible pricing, educational mode |
| Amazon Braket | Multi-provider hardware, AWS integration, managed notebooks | Complex AWS setup, no visual circuit builder, expensive, AWS lock-in | Visual-first IDE, simple standalone platform, transparent pricing |
| Strangeworks | Multi-provider access, enterprise features | Early stage, limited simulation, no visual builder, high price point | Full IDE with visual builder + simulator + hardware in one platform |

---

## Risks & Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Quantum hardware providers restrict third-party API access | High | Medium | Maintain direct partnerships with hardware providers, offer white-label IDE as value proposition, support open-standard circuit formats |
| GPU simulation at 30+ qubits is too slow for interactive use | Medium | Medium | Implement circuit partitioning and tensor network simulation for structured circuits, cache partial states, use approximate simulation for quick previews |
| Quantum computing hype cycle downturn reduces user growth | Medium | Medium | Focus on educational market (stable demand), build genuinely useful tools for near-term algorithms (VQE, QAOA), maintain affordable pricing |
| Free tier users consume expensive GPU simulation resources | Medium | High | Implement simulation quotas, queue-based scheduling, smaller qubit limits on free tier, efficient resource sharing |
| Competing platforms (IBM, Google) improve their free tools | High | High | Differentiate on multi-provider access and visual UX, build community moat with circuits and tutorials, move faster on features |

---

## MVP Scope (v1.0)

### In Scope
- Visual circuit builder with 15 core gates (H, X, Y, Z, S, T, CNOT, CZ, SWAP, Rx, Ry, Rz, Toffoli, measure, barrier)
- GPU-accelerated state vector simulation up to 25 qubits
- Bloch sphere visualization and amplitude bar chart
- OpenQASM 3.0 code editor with bidirectional circuit sync
- Measurement histogram for shot-based simulation
- Circuit templates (Bell state, GHZ, QFT, Grover's search)
- User accounts, project CRUD, and basic circuit version history
- Share links for read-only circuit viewing

### Out of Scope (v1.1+)
- Density matrix noise simulation (v1.1 — highest priority post-MVP)
- IBM Quantum hardware access (v1.1)
- Qiskit and Cirq code generation (v1.2)
- IonQ, Rigetti, and Amazon Braket integration (v1.2)
- VQE/QAOA optimization studio (v1.2)
- Team collaboration with comments and version control (v1.3)
- Experiment tracking and comparison (v1.3)
- Error correction studio (v1.3)
- Educational mode with guided tutorials (v1.3)
- Enterprise features: SSO, API, on-premise deployment (v2.0)

### MVP Timeline: 10 weeks
- Week 1-2: Auth system, project data model, circuit canvas renderer with gate placement, drag-and-drop gate palette
- Week 3-4: State vector simulation engine (cuStateVec on GPU), real-time simulation integration with circuit builder, Bloch sphere and amplitude bar chart
- Week 5-6: OpenQASM 3.0 parser and code editor (Monaco), bidirectional circuit ↔ code sync, measurement simulation with histogram display
- Week 7-8: Circuit templates, parameterized gates with sliders, circuit statistics (depth, gate count), circuit version history
- Week 9-10: Project dashboard, share links, billing integration (Stripe), circuit gallery, documentation, load testing, launch preparation
