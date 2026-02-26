# QubitStudio — Persona Subspecs

> Parent spec: [`specs/28-qubitstudio.md`](../28-qubitstudio.md)

---

## Experience-Level Personas

### Student/Academic

**Modified UI/UX:**
- Show a visual circuit builder with labeled gate icons, Bloch sphere animations for single-qubit states, and step-by-step execution mode
- Provide a "Quantum Concepts" sidebar that explains the gate being placed (e.g., "Hadamard creates an equal superposition of |0> and |1>")
- Display amplitude bar charts and phase wheels in real time as the user adds gates to the circuit

**Feature Gating:**
- Allow circuits up to 8 qubits with state vector simulation; hide noise simulation and hardware access
- Expose common gates (H, X, Y, Z, CNOT, Toffoli, RX, RY, RZ) and measurement; hide custom unitary and pulse-level control
- Limit to 50 simulation runs per day with no hardware job submissions

**Pricing Tier:** Free tier

**Onboarding Flow:**
- First-run tutorial builds a Bell state (H + CNOT) with animated Bloch sphere showing entanglement step by step
- Offer a learning path of 10 exercises (superposition, entanglement, teleportation, Grover's, Deutsch-Jozsa) with interactive explanations
- Prompt to join a classroom workspace for course-based collaboration

**Key Workflows:**
- Build quantum circuits visually by dragging gates onto qubit wires and observe state changes in real time
- Step through circuit execution gate by gate, watching Bloch sphere and amplitude chart animations
- Complete structured exercises demonstrating fundamental quantum computing concepts
- Export circuit diagrams and measurement histograms for course assignments

---

### Junior Engineer (0-3 years)

**Modified UI/UX:**
- Show the full gate palette organized by category (single-qubit, multi-qubit, rotation, custom) with parameter input fields
- Provide a "Circuit Analyzer" panel that computes circuit depth, gate count, CNOT count, and transpilation cost estimates
- Display both the visual circuit and the equivalent Qiskit/Cirq code side by side with live synchronization

**Feature Gating:**
- Unlock circuits up to 20 qubits, noise simulation with preset hardware noise models, and limited hardware access (100 shots/day on IBM free tier)
- Expose parameterized circuits, basic variational algorithms (VQE template), and multi-provider simulation
- Hide pulse-level control, custom noise model definition, and batch hardware job submission

**Pricing Tier:** Pro tier ($39/month)

**Onboarding Flow:**
- Walk through building a parameterized circuit, running it with noise simulation, and comparing ideal vs. noisy results
- Demonstrate submitting a circuit to IBM Quantum hardware and interpreting the measurement results
- Show the code view and how circuit modifications sync between visual and code representations

**Key Workflows:**
- Design parameterized quantum circuits for variational algorithms (VQE, QAOA) using the visual builder
- Simulate circuits with realistic noise models to understand hardware limitations before running on real devices
- Submit circuits to IBM Quantum hardware and compare results against ideal simulation
- Analyze circuit metrics (depth, gate count) and optimize for target hardware constraints
- Share circuit designs and results with mentors for feedback and collaborative algorithm development

---

### Senior/Expert

**Modified UI/UX:**
- Provide a multi-panel workspace with circuit editor, code editor, noise model configurator, and hardware dashboard
- Expose pulse-level control with waveform editors for custom gate calibrations
- Support Jupyter-style notebook cells interleaved with visual circuit blocks for mixed code-visual workflows

**Feature Gating:**
- Full access: unlimited qubits (simulation limited by compute), custom noise models, all hardware providers, batch job submission, and pulse-level control
- Expose advanced compilation: custom transpilation passes, error mitigation techniques, and qubit mapping optimization
- Enable team workspaces, API access, priority hardware queuing, and integration with external quantum frameworks

**Pricing Tier:** Enterprise tier ($149/month or institutional license)

**Onboarding Flow:**
- Skip tutorials; present API documentation and integration guides for connecting to existing quantum research workflows
- Configure hardware provider accounts (IBM, IonQ, Rigetti) and set up default transpilation and error mitigation preferences
- Demonstrate the pulse-level editor and custom noise model builder

**Key Workflows:**
- Develop novel quantum algorithms with mixed visual-code workflows and advanced compilation strategies
- Design and test custom error mitigation techniques across multiple hardware platforms
- Run large-scale hardware experiments with batch job submission and cross-provider result comparison
- Build custom noise models calibrated to specific hardware for accurate pre-hardware simulation
- Manage a research team's quantum computing projects with shared circuits, results, and hardware allocations

---

## Role-Based Personas

### Designer/Creator

**Modified UI/UX:**
- Emphasize the visual circuit canvas with drag-and-drop gates, subcircuit grouping, and circuit template library
- Provide a parameterized circuit designer with slider controls for rotation angles and live output visualization
- Show a circuit composition panel for building complex algorithms from reusable subcircuit modules

**Feature Gating:**
- Full access to the visual circuit builder, all gate types, subcircuit creation, and template management
- Expose circuit export in multiple formats (Qiskit, Cirq, OpenQASM, LaTeX) for publication and sharing
- Enable circuit template publishing to the team or community library

**Pricing Tier:** Pro tier ($39/month)

**Onboarding Flow:**
- Start by building a circuit from the template library and customizing it for a specific use case
- Demonstrate subcircuit creation by encapsulating a common pattern (e.g., QFT) into a reusable block
- Show how to export circuit diagrams in publication-quality LaTeX format

**Key Workflows:**
- Design quantum circuits using the visual builder with drag-and-drop gates and subcircuit composition
- Create reusable subcircuit modules for common patterns (QFT, Grover oracle, variational ansatz)
- Build parameterized circuit templates for algorithmic exploration and optimization
- Export circuits in multiple formats (OpenQASM, Qiskit, Cirq, LaTeX) for papers and presentations
- Publish circuit templates and algorithm implementations to the community library

---

### Analyst/Simulator

**Modified UI/UX:**
- Foreground the simulation results dashboard with measurement histograms, state tomography plots, and fidelity metrics
- Show a noise analysis panel comparing ideal vs. noisy simulation results with per-gate error breakdowns
- Provide a hardware benchmarking view for comparing circuit performance across different quantum processors

**Feature Gating:**
- Full access to state vector and density matrix simulation, noise modeling, and result analysis tools
- Expose error analysis tools: process tomography, randomized benchmarking, and gate fidelity estimation
- Enable batch simulation sweeps over parameters, noise levels, and hardware configurations

**Pricing Tier:** Pro tier ($39/month)

**Onboarding Flow:**
- Walk through simulating a circuit with and without noise, comparing fidelity, and identifying the dominant error sources
- Demonstrate running the same circuit on multiple simulated backends and comparing performance
- Show the parameter sweep tool for analyzing algorithm performance across a range of inputs

**Key Workflows:**
- Simulate circuits with configurable noise models and analyze the impact of errors on algorithm output
- Run parameter sweeps to characterize algorithm performance across a range of input values
- Compare circuit fidelity across simulated hardware backends to identify optimal device-algorithm pairings
- Perform error budget analysis identifying which gates contribute most to output degradation
- Generate benchmarking reports comparing quantum hardware performance for publication or hardware selection decisions

---

### Manufacturing/Process

**Modified UI/UX:**
- Show a hardware calibration dashboard with current qubit properties (T1, T2, gate fidelities, readout errors) for each connected backend
- Display a transpilation pipeline view showing circuit transformation stages (routing, scheduling, optimization)
- Provide a job management panel with queue status, priority controls, and cost tracking across hardware providers

**Feature Gating:**
- Expose advanced transpilation with custom pass managers and hardware-aware circuit optimization
- Enable batch job management with priority queuing, cost optimization, and result aggregation across providers
- Unlock pulse-level calibration tools for custom gate implementations on supported hardware

**Pricing Tier:** Enterprise tier ($149/month)

**Onboarding Flow:**
- Configure hardware provider connections and review current calibration data for available backends
- Walk through the transpilation pipeline and how to customize optimization passes for a target device
- Demonstrate batch job submission with cost estimation and priority management

**Key Workflows:**
- Optimize circuit transpilation for specific hardware topologies to minimize gate count and circuit depth
- Manage batch hardware experiments with cost tracking, priority scheduling, and automated result collection
- Monitor hardware calibration data and select optimal qubits and devices for each experiment
- Implement custom transpilation passes for specialized circuit patterns or hardware capabilities
- Track hardware usage and costs across providers for budget management and procurement planning

---

### Regulatory/Compliance

**Modified UI/UX:**
- Add a research reproducibility panel documenting exact circuit, transpilation settings, hardware calibration snapshot, and random seeds
- Display an audit trail showing every circuit submission, parameter, and result with immutable timestamps
- Provide an export control status indicator for algorithms involving cryptographic applications

**Feature Gating:**
- Expose comprehensive experiment logging with reproducibility metadata and immutable audit records
- Enable data retention policies and access controls for sensitive quantum algorithm research
- Unlock export control tagging for circuits related to cryptographic or national security applications

**Pricing Tier:** Enterprise tier ($149/month)

**Onboarding Flow:**
- Configure reproducibility settings to automatically capture all parameters, seeds, and hardware calibration data
- Set up access controls and data classification for sensitive algorithm research
- Demonstrate the audit trail and how to export experiment provenance for publication or review

**Key Workflows:**
- Maintain reproducible experiment records capturing exact circuit, transpilation, hardware state, and seeds for every run
- Track and tag algorithms with potential export control implications (cryptographic, dual-use)
- Generate audit-ready logs of all hardware access and experiment results for institutional compliance
- Enforce access controls on sensitive quantum algorithm research per institutional and government policies
- Export comprehensive experiment provenance records for peer review and publication reproducibility

---

### Manager/Decision-maker

**Modified UI/UX:**
- Present a research portfolio dashboard showing all quantum computing projects, team assignments, and publication pipeline
- Show hardware utilization and cost analytics: credits consumed per provider, cost per experiment, budget forecasting
- Provide a technology readiness view mapping algorithm development stages to hardware capability milestones

**Feature Gating:**
- Read-only access to all projects and experiments with commenting and prioritization capabilities
- Expose cost analytics, team management, and hardware credit allocation tools
- Enable progress reporting with customizable dashboards for stakeholder communication

**Pricing Tier:** Enterprise tier ($149/month)

**Onboarding Flow:**
- Set up the research portfolio dashboard and configure project categories (algorithm research, hardware benchmarking, applications)
- Walk through hardware credit allocation and cost tracking across IBM, IonQ, and Rigetti
- Demonstrate the progress reporting view for summarizing team output and key findings

**Key Workflows:**
- Monitor quantum research programs from a unified dashboard with project status and key results
- Allocate hardware credits and compute budgets across teams and projects
- Track publication pipeline progress and map research milestones to strategic objectives
- Review team productivity metrics: experiments run, circuits designed, papers submitted
- Generate executive summaries of quantum computing research progress for leadership and funding agencies
