# MatDiscover — AI-Powered Materials Discovery and Property Prediction

## Executive Summary

MatDiscover is a cloud-native materials informatics platform that replaces fragmented experimental workflows and expensive density functional theory (DFT) simulations with machine-learning-driven property prediction and generative materials design. Researchers upload crystal structures or compositions, get instant ML predictions for key properties (bandgap, formation energy, elastic moduli, thermal conductivity), and use generative models to propose novel materials optimized for target specifications — all without writing code or managing HPC clusters.

---

## Problem Statement

**The pain:**
- Running a single DFT simulation in VASP costs $5,000-$15,000/year in licensing alone, plus thousands in compute time per material — a single formation energy calculation can take 24-72 hours on a 64-core node
- Materials researchers spend weeks manually searching fragmented databases (Materials Project, ICSD, AFLOW, OQMD) and cross-referencing papers to find candidate materials with desired properties
- Experimental iteration cycles are brutally slow: synthesize a material, characterize it, discover it doesn't meet spec, repeat — each cycle costs $2K-$50K and 2-6 weeks in lab time
- There is no unified platform that connects computational predictions, literature data, and experimental results — researchers maintain ad-hoc spreadsheets and Jupyter notebooks
- Graduate students and postdocs waste months learning DFT software setup, pseudopotential selection, convergence testing, and post-processing rather than doing actual materials science

**Current workarounds:**
- Running VASP/Quantum ESPRESSO on university HPC clusters with month-long queue times and steep learning curves
- Manually querying Materials Project API and writing custom Python scripts for every analysis
- Using Citrine Informatics or Schrödinger Materials Science (expensive enterprise contracts starting at $50K+/year, complex onboarding)
- Maintaining personal databases in Excel/Google Sheets and sharing results via email attachments

**Market size:** The global materials informatics market was valued at $1.2B in 2024 and is projected to reach $4.5B by 2030 (CAGR 24%). The addressable market includes ~50,000 materials science research groups worldwide, 500+ battery/semiconductor/aerospace companies with active materials R&D, and 40+ national laboratories conducting materials research programs.

---

## Target Users

### Primary Personas

**1. Dr. Wei Chen — Materials Science Postdoc**
- Works in a computational materials group at a university, studying perovskite solar cell materials
- Spends 60% of time setting up and debugging VASP calculations rather than interpreting results; frequently waits 2-3 weeks for HPC queue allocation
- Needs: fast property predictions to pre-screen candidates before committing expensive DFT compute, a way to explore composition-property landscapes interactively

**2. Kavitha Ramanathan — Battery Materials Engineer (Industry)**
- Works at a lithium-ion battery startup searching for next-generation cathode materials with high energy density and thermal stability
- Has a target property window (voltage > 4.0V, specific capacity > 200 mAh/g, formation energy < -1.5 eV/atom) but must screen thousands of candidates to find viable ones
- Needs: ML-powered screening of large composition spaces, generative suggestions for novel compositions within target property ranges, integration with experimental characterization data from the lab

**3. Prof. James Okonkwo — PI at a National Lab**
- Leads a 12-person research group working on high-entropy alloys for aerospace applications; manages DOE-funded projects with strict deliverables
- Struggles to keep track of which compositions have been simulated, synthesized, and tested across multiple students and projects
- Needs: team collaboration with shared materials databases, project-level organization, reproducible computational workflows, and progress dashboards for grant reporting

### Secondary Personas
- Graduate students learning computational materials science who need guided workflows instead of raw command-line tools
- Materials data scientists building ML models who need clean, curated training datasets from multiple sources
- R&D managers at semiconductor companies evaluating new dielectric or thermoelectric materials for product roadmaps

---

## Solution Overview

MatDiscover works as follows:
1. Researchers input materials by uploading CIF/POSCAR crystal structure files, drawing structures in the built-in 3D builder, or specifying chemical compositions — the platform also connects to Materials Project, ICSD, and AFLOW for instant import
2. The ML prediction engine (graph neural networks trained on 150K+ DFT-calculated materials) returns property predictions in under 5 seconds: formation energy, bandgap, bulk/shear modulus, Debye temperature, thermal conductivity, and stability metrics (energy above hull)
3. For inverse design, researchers specify target property ranges and the generative model (conditional variational autoencoder + reinforcement learning) proposes novel compositions and structures optimized to meet those specifications
4. Researchers track experimental results alongside predictions, close the loop by feeding measured data back into project-specific ML models that improve over time, and generate publication-ready phase diagrams and property maps
5. Optional cloud DFT integration allows submitting VASP/Quantum ESPRESSO jobs to managed GPU clusters for high-fidelity validation of the most promising ML-identified candidates

---

## Core Features

### F1: Crystal Structure Input & 3D Builder
- Upload CIF, POSCAR, XYZ, and PDB file formats with automatic parsing and validation
- Interactive 3D crystal structure viewer powered by Three.js with ball-and-stick, polyhedral, and space-filling rendering modes
- Built-in structure builder: select space group, set lattice parameters, place atoms on Wyckoff positions
- Supercell generator, surface slab builder, and defect creator (vacancies, substitutions, interstitials)
- Integration with Materials Project, AFLOW, OQMD, and ICSD — search by formula, space group, or property range and import structures directly

### F2: ML Property Prediction Engine
- Graph Neural Network (MEGNet/M3GNet architecture) trained on 154,000+ DFT-calculated materials from Materials Project
- Predict formation energy (MAE < 0.03 eV/atom), bandgap (MAE < 0.3 eV), bulk modulus (MAE < 8 GPa), shear modulus, Debye temperature, and thermal conductivity
- Stability assessment: energy above convex hull with confidence intervals, decomposition products identification
- Uncertainty quantification via ensemble predictions — flag low-confidence predictions automatically
- Batch prediction: upload CSV of 10,000+ compositions and get results in minutes
- Property prediction explanations: SHAP-based feature attribution showing which structural/chemical features drive predictions

### F3: Generative Materials Design
- Conditional Variational Autoencoder (CVAE) trained on crystal structure-property pairs for inverse design
- Specify target property window (e.g., bandgap 1.2-1.6 eV, formation energy < -1.0 eV/atom, no toxic elements)
- Generate ranked list of candidate compositions and structures with predicted properties and novelty scores
- Constraint engine: exclude specific elements, enforce charge neutrality, set composition ranges, require specific crystal systems
- Reinforcement learning optimizer that iteratively refines candidates toward multi-objective Pareto fronts
- Crystal structure prediction for generated compositions using template-based and diffusion model approaches

### F4: Phase Diagram Generator
- Compute and render binary, ternary, and quaternary phase diagrams from ML predictions or imported DFT data
- Interactive Gibbs triangle visualization for ternary systems with clickable phase regions
- Convex hull construction showing stable phases and metastable candidates with energy above hull annotations
- Temperature-dependent phase diagrams using CALPHAD-integrated ML models
- Export publication-quality SVG/PDF diagrams with customizable labels, colors, and annotations
- Overlay experimental data points onto computed phase diagrams for direct comparison

### F5: Literature Mining & Knowledge Graph
- NLP pipeline that extracts materials properties from published papers (powered by MatBERT/LLM)
- Search 5M+ materials science papers by composition, property, synthesis method, or application
- Auto-populate property databases with literature-reported values including source citations
- Knowledge graph visualization showing relationships between materials, properties, applications, and synthesis routes
- Alert system: get notified when new papers are published about materials in your projects

### F6: Experimental Data Tracking
- Log synthesis parameters (temperature, pressure, atmosphere, precursors, duration) for each experimental run
- Record characterization results: XRD patterns, SEM/TEM images, DSC/TGA curves, impedance spectra
- Side-by-side comparison of predicted vs. measured properties with automatic error analysis
- Bayesian optimization module: suggest next experiments to maximize information gain based on existing data
- Export experimental records as structured datasets for publication supplementary materials

### F7: Cloud DFT Job Submission
- Submit VASP, Quantum ESPRESSO, or CP2K calculations to managed GPU/CPU clusters without any infrastructure setup
- Pre-configured input file templates for common calculation types: geometry optimization, band structure, DOS, elastic constants, phonon dispersion
- Automated convergence testing for k-point mesh, energy cutoff, and pseudopotential selection
- Job monitoring dashboard with real-time status, estimated completion time, and cost tracking
- Post-processing pipeline: automatic extraction of properties from raw output files, band structure plotting, DOS visualization
- Pay-per-compute pricing: ~$0.15/core-hour, roughly 10x cheaper than commercial VASP cloud deployments

### F8: Materials Database Search
- Unified search across Materials Project (154K), AFLOW (3.5M), OQMD (1M+), and JARVIS (75K) databases
- Advanced query builder: filter by elements, space group, bandgap range, formation energy, magnetic ordering, and more
- Property comparison tables for shortlisted materials with sortable columns
- Crystal structure similarity search: upload a structure and find similar materials across all databases
- Downloadable datasets with standardized property formats for ML model training

### F9: Team Collaboration & Project Management
- Organize materials into projects with shared access for team members
- Role-based permissions: PI (full access), researcher (edit), viewer (read-only)
- Activity feed showing who predicted, tested, or annotated which materials
- Commenting system on individual materials and predictions for team discussion
- Project-level dashboards showing screening funnels: candidates → predicted → validated → synthesized → characterized
- Export project summaries for grant reports and publications

### F10: Visualization & Reporting
- Interactive property-property scatter plots (e.g., bandgap vs. formation energy) with clickable data points
- Composition-property heatmaps for systematic exploration of chemical spaces
- t-SNE/UMAP embeddings of material feature spaces for clustering and similarity analysis
- Auto-generated PDF reports summarizing screening results, top candidates, and property comparisons
- Publication-ready figure export in SVG, PDF, and PNG formats with journal-specific formatting presets

---

## Technical Architecture

### System Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                     React Frontend (SPA)                     │
│  Structure Viewer │ Property Dashboard │ Generative Design   │
└──────────┬──────────────────┬──────────────────┬────────────┘
           │                  │                  │
           ▼                  ▼                  ▼
┌─────────────────────────────────────────────────────────────┐
│                   FastAPI Gateway (REST + WebSocket)          │
│          Auth │ Rate Limiting │ Request Routing               │
└──────┬────────────┬──────────────┬────────────┬─────────────┘
       │            │              │            │
       ▼            ▼              ▼            ▼
┌──────────┐ ┌────────────┐ ┌──────────┐ ┌──────────────────┐
│ ML Pred  │ │ Generative │ │ DFT Job  │ │ Literature       │
│ Service  │ │ Design Svc │ │ Manager  │ │ Mining Service   │
│ (PyTorch │ │ (CVAE/RL)  │ │ (Celery) │ │ (MatBERT + LLM) │
│ + GPU)   │ │ (GPU)      │ │          │ │                  │
└────┬─────┘ └─────┬──────┘ └────┬─────┘ └───────┬──────────┘
     │             │              │               │
     └──────┬──────┘              │               │
            ▼                     ▼               ▼
     ┌─────────────┐    ┌──────────────┐  ┌─────────────┐
     │ GPU Cluster  │    │ HPC Cluster  │  │ Elasticsearch│
     │ (A100/H100)  │    │ (DFT Compute)│  │ (Papers DB) │
     └─────────────┘    └──────────────┘  └─────────────┘
            │                    │                │
            └────────┬───────────┘                │
                     ▼                            │
              ┌─────────────┐              ┌─────┘
              │ PostgreSQL  │◀─────────────┘
              │ + pgvector  │
              └──────┬──────┘
                     │
              ┌──────┴──────┐
              │    Redis     │
              │ (Cache/Queue)│
              └─────────────┘
```

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | React 18 + TypeScript, Three.js (3D structures), D3.js (charts), TailwindCSS |
| Backend API | Python 3.11, FastAPI, Pydantic v2, Uvicorn |
| ML Inference | PyTorch 2.x, PyTorch Geometric (GNN), ONNX Runtime for optimized serving |
| Generative Models | PyTorch (CVAE), Stable Baselines3 (RL), CDVAE for crystal generation |
| DFT Integration | Pymatgen, ASE (Atomic Simulation Environment), Custodian (workflow management) |
| Database | PostgreSQL 16 + pgvector extension for structure embeddings |
| Search | Elasticsearch 8.x for literature and materials full-text search |
| Task Queue | Celery + Redis for async job processing (DFT submissions, batch predictions) |
| Object Storage | S3-compatible (MinIO or AWS S3) for structure files, DFT outputs, images |
| GPU Compute | Kubernetes + NVIDIA GPU Operator, spot instances for cost optimization |
| Monitoring | Prometheus + Grafana, Sentry for error tracking |

### Data Model

```
Organization
├── id, name, slug, plan, created_at
│
├── Project[]
│   ├── id, org_id, name, description, status (active/archived)
│   ├── target_properties (JSON: {bandgap: {min: 1.0, max: 2.0}, ...})
│   └── created_by, created_at
│
├── Material[]
│   ├── id, project_id, org_id
│   ├── formula, reduced_formula, chemical_system (e.g., "Li-Fe-P-O")
│   ├── structure_data (JSON: lattice + sites, pymatgen Structure serialized)
│   ├── space_group_number, space_group_symbol, crystal_system
│   ├── source (uploaded/generated/imported/experimental)
│   ├── source_database, source_id (e.g., mp-1234 from Materials Project)
│   ├── structure_embedding (vector[256] for similarity search)
│   └── created_by, created_at
│
├── Prediction[]
│   ├── id, material_id, model_version
│   ├── property_name (formation_energy/bandgap/bulk_modulus/...)
│   ├── predicted_value, unit, uncertainty
│   ├── confidence_score (0-1)
│   ├── feature_attributions (JSON: SHAP values)
│   └── created_at
│
├── ExperimentalResult[]
│   ├── id, material_id, project_id
│   ├── property_name, measured_value, unit, measurement_method
│   ├── synthesis_conditions (JSON: temperature, pressure, atmosphere, ...)
│   ├── characterization_files[] (S3 keys for XRD, SEM, etc.)
│   └── recorded_by, recorded_at
│
├── DFTJob[]
│   ├── id, material_id, org_id
│   ├── calculation_type (relax/static/band_structure/phonon/elastic)
│   ├── software (vasp/qe/cp2k), input_parameters (JSON)
│   ├── status (queued/running/completed/failed)
│   ├── compute_hours, cost_usd
│   ├── output_files[] (S3 keys)
│   └── submitted_by, submitted_at, completed_at
│
├── GenerativeRun[]
│   ├── id, project_id, org_id
│   ├── target_properties (JSON), constraints (JSON)
│   ├── model_type (cvae/rl/diffusion), model_version
│   ├── num_candidates_requested, num_candidates_generated
│   ├── status (running/completed)
│   └── created_by, created_at
│
├── GeneratedCandidate[]
│   ├── id, generative_run_id, material_id (FK to Material)
│   ├── novelty_score, pareto_rank
│   └── predicted_properties (JSON)
│
└── Member[]
    ├── id, org_id, user_id, role (admin/researcher/viewer)
    └── invited_at, joined_at
```

### API Design

```
Auth:
POST   /api/auth/login                          # Email/password or OAuth
POST   /api/auth/register                       # New account
POST   /api/auth/refresh                        # Refresh JWT token

Projects:
GET    /api/projects                             # List projects
POST   /api/projects                             # Create project
GET    /api/projects/:id                         # Get project details
PATCH  /api/projects/:id                         # Update project
DELETE /api/projects/:id                         # Archive project
GET    /api/projects/:id/dashboard               # Screening funnel stats

Materials:
GET    /api/projects/:id/materials               # List materials (paginated, filterable)
POST   /api/projects/:id/materials               # Add material (upload structure)
POST   /api/projects/:id/materials/import        # Import from external DB
POST   /api/projects/:id/materials/batch         # Batch upload (CSV/ZIP)
GET    /api/materials/:id                        # Get material details
DELETE /api/materials/:id                        # Remove material
GET    /api/materials/:id/similar                # Find similar materials (embedding search)

Predictions:
POST   /api/materials/:id/predict                # Run ML prediction
POST   /api/materials/batch-predict              # Batch prediction (up to 10K)
GET    /api/materials/:id/predictions            # Get all predictions for material
GET    /api/predictions/:id/explain              # Get SHAP feature attributions

Generative Design:
POST   /api/projects/:id/generate                # Launch generative run
GET    /api/generative-runs/:id                  # Get run status and candidates
GET    /api/generative-runs/:id/candidates       # List generated candidates
POST   /api/generative-runs/:id/candidates/:cid/promote  # Promote to project material

DFT Jobs:
POST   /api/materials/:id/dft                    # Submit DFT calculation
GET    /api/dft-jobs                             # List all DFT jobs
GET    /api/dft-jobs/:id                         # Job status and results
POST   /api/dft-jobs/:id/cancel                  # Cancel running job
GET    /api/dft-jobs/:id/outputs                 # Download output files

Experimental Data:
POST   /api/materials/:id/experiments            # Log experimental result
GET    /api/materials/:id/experiments            # Get experimental history
POST   /api/materials/:id/experiments/:eid/files # Upload characterization files

Phase Diagrams:
POST   /api/projects/:id/phase-diagram           # Generate phase diagram
GET    /api/phase-diagrams/:id                   # Get rendered diagram + data

Search:
GET    /api/search/materials                     # Search across all databases
GET    /api/search/literature                    # Search papers by query
GET    /api/search/similar-structure             # Structure similarity search

Team:
GET    /api/orgs/:id/members                     # List team members
POST   /api/orgs/:id/members                     # Invite member
PATCH  /api/orgs/:id/members/:mid                # Update role
DELETE /api/orgs/:id/members/:mid                # Remove member
```

---

## UI/UX — Key Screens

### 1. Project Dashboard
- Screening funnel visualization: total candidates → ML-screened → DFT-validated → synthesized → characterized, with conversion rates at each stage
- Property distribution charts for all materials in the project (histograms of bandgap, formation energy, etc.)
- Recent activity feed: predictions run, experiments logged, candidates generated
- Quick actions: upload structure, run batch prediction, launch generative design

### 2. Material Detail View
- 3D interactive crystal structure viewer (rotate, zoom, toggle atom labels, measure bond lengths/angles)
- Property card grid showing all predicted properties with confidence bars and comparison to experimental values where available
- Tabs: Predictions (with SHAP explanations), Experimental Data (synthesis + characterization logs), DFT Results (if computed), Literature (auto-linked papers mentioning this composition)
- Action buttons: Run DFT, Log Experiment, Find Similar, Add to Comparison

### 3. Generative Design Studio
- Target property range sliders (bandgap, formation energy, modulus, etc.) with interactive Pareto front preview
- Constraint panel: element inclusion/exclusion, charge neutrality, crystal system preference, composition ranges
- Results table: generated candidates ranked by novelty score and property match, with inline structure previews
- Scatter plot overlay of generated candidates on existing property landscape to visualize coverage of design space

### 4. Phase Diagram Viewer
- Interactive ternary/binary phase diagram with hover tooltips showing composition, energy, and stability info
- Toggle between computed convex hull and experimental data overlay
- Clickable phase regions that list stable compounds and their properties
- Export panel: SVG/PDF with customizable axis labels, color schemes, and annotation placement

### 5. Batch Prediction & Screening
- Upload panel: drag-and-drop CSV (composition list) or ZIP (structure files)
- Real-time progress bar as predictions complete
- Results table with sortable columns for every predicted property
- Multi-property filter panel: set ranges for each property, see candidates that pass all filters highlighted
- Export filtered results as CSV or add passing candidates to a project

### 6. Literature Search & Knowledge Graph
- Search bar with auto-complete for compositions and property names
- Results displayed as paper cards with extracted property values highlighted in context
- Knowledge graph visualization: click a material node to see connected papers, properties, applications, and synthesis routes
- Save relevant papers and extracted data points directly to project materials

---

## Monetization

### Free Tier (Academic)
- Up to 50 ML predictions per month
- 1 project, 100 materials maximum
- Access to Materials Project database search
- Basic structure viewer and property predictions
- Community support only
- Requires .edu email or academic institution verification

### Researcher — $99/month
- 2,000 ML predictions per month
- Unlimited projects and materials
- Generative design (10 runs/month, 100 candidates per run)
- Phase diagram generation
- Literature mining (100 searches/month)
- Experimental data tracking
- CSV/PDF export
- Email support

### Lab — $499/month
- 20,000 ML predictions per month
- Unlimited generative design runs
- Cloud DFT job submission (billed separately per compute-hour)
- Full literature mining and knowledge graph access
- Team collaboration (up to 15 members)
- API access for programmatic workflows
- Custom ML model fine-tuning on lab data (2 custom models)
- Priority support with 24h response SLA

### Enterprise — Custom
- Unlimited everything
- Dedicated GPU cluster allocation for low-latency inference
- Unlimited custom ML model training on proprietary datasets
- On-premise deployment option for sensitive IP
- SSO/SAML integration
- Custom database integrations (ICSD, proprietary internal databases)
- Dedicated customer success manager
- SLA with 99.9% uptime guarantee

---

## Go-to-Market Strategy

### Phase 1: Academic Adoption (Month 1-3)
- Launch free tier targeting top 50 materials science departments (MIT, Stanford, Berkeley, Georgia Tech, etc.)
- Publish benchmark paper on arXiv comparing MatDiscover ML accuracy to VASP DFT results across test sets
- Present at MRS (Materials Research Society) Fall Meeting and ACS national conference
- Offer free Lab tier for 6 months to 10 "lighthouse" research groups who publish actively and will cite the platform
- Create tutorials and Jupyter notebook integrations for graduate-level computational materials courses

### Phase 2: Community & Content (Month 3-6)
- Launch public materials prediction leaderboard (community submits compositions, compares predictions to experiments)
- Blog series: "Materials Discovery in 10 Minutes" — case studies replacing weeks of DFT with ML screening
- YouTube tutorials: structure building, property prediction, generative design workflows
- Open-source the base GNN model weights to build community trust and drive adoption
- Integration with pymatgen and ASE Python libraries so researchers can call MatDiscover from existing scripts

### Phase 3: Industry Expansion (Month 6-12)
- Target battery materials companies (QuantumScape, Solid Power, Sila Nano) with tailored cathode/anode screening workflows
- Partner with semiconductor companies evaluating new dielectric and thermoelectric materials
- Develop industry-specific model packages: battery materials, photovoltaics, catalysts, structural alloys
- Attend industry conferences: Battery Show, SEMICON, AeroMat
- Launch enterprise pilot program with dedicated onboarding and custom model training

### Acquisition Channels
- Academic word-of-mouth and citations in published papers (primary growth driver)
- Organic search targeting "materials property prediction", "ML materials discovery", "bandgap prediction tool"
- Integration with popular Python materials science libraries (pymatgen, ASE, matminer)
- Partnerships with national labs (NREL, ORNL, Argonne) for co-development of domain-specific models
- Conference presentations and poster sessions at MRS, ACS, TMS, and APS meetings

---

## Success Metrics & KPIs

| Metric | Target (Month 6) | Target (Month 12) |
|--------|-------------------|---------------------|
| Registered users | 2,000 | 8,000 |
| Monthly predictions run | 50,000 | 500,000 |
| Paying customers (Researcher+Lab) | 30 | 150 |
| MRR | $6,000 | $40,000 |
| Academic papers citing MatDiscover | 5 | 25 |
| Prediction accuracy (formation energy MAE) | < 0.035 eV/atom | < 0.025 eV/atom |
| Free → Paid conversion | 3% | 6% |
| Churn rate (monthly) | < 6% | < 4% |

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | MatDiscover Advantage |
|-----------|-----------|------------|----------------------|
| Materials Project | Free, massive database (154K materials), trusted by community | No ML-driven design, no generative models, query-only (no prediction for new materials) | ML predictions for any composition, generative design, experimental tracking |
| Citrine Informatics | Enterprise ML platform, strong industry partnerships | Expensive ($50K+/year), complex onboarding, no self-serve, opaque models | Self-serve from $99/mo, transparent predictions with SHAP explanations, faster time-to-value |
| Schrödinger Materials Science | Accurate physics-based simulations, established brand | Very expensive ($100K+/year), steep learning curve, slow simulation times | 1000x faster ML predictions, accessible to non-experts, fraction of the cost |
| VASP | Gold standard for DFT accuracy, widely published | $5K-$15K/year license, requires HPC expertise, slow (hours-days per material) | ML pre-screening reduces DFT compute by 90%, managed cloud DFT eliminates HPC pain |
| Atomistic Simulation Tools (ASE, pymatgen) | Free, flexible, programmatic | Code-only (no GUI), no built-in ML, no collaboration features | Visual interface, built-in ML models, team collaboration, no coding required |

---

## Risks & Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| ML predictions are inaccurate for novel chemical spaces outside training data | High | Medium | Uncertainty quantification flags low-confidence predictions; active learning loop improves models with new DFT/experimental data; publish accuracy benchmarks transparently |
| Researchers distrust black-box ML and prefer DFT | High | Medium | SHAP explanations for every prediction; optional DFT validation workflow; publish peer-reviewed benchmark papers; open-source base model for scrutiny |
| GPU compute costs for inference and generative models erode margins | Medium | Medium | ONNX optimization reduces inference cost 5x; batch predictions on spot instances; cache predictions for common materials; tiered rate limiting |
| Materials Project or similar free platform adds ML features | Medium | High | Move fast on generative design (unique differentiator); focus on experimental integration and team collaboration that academic databases won't build; build switching costs through accumulated project data |
| Export control or IP concerns prevent industry adoption | Medium | Low | Offer on-premise Enterprise deployment; SOC 2 compliance; data encryption at rest and in transit; clear data ownership terms — customer data never used for model training without explicit consent |

---

## MVP Scope (v1.0)

### In Scope
- Crystal structure upload (CIF, POSCAR) and 3D viewer
- ML property prediction for 6 core properties: formation energy, bandgap, bulk modulus, shear modulus, energy above hull, Debye temperature
- Materials Project database search and import
- Single-project workspace with materials list and prediction history
- Batch prediction via CSV upload (up to 500 compositions)
- Basic user authentication and free/paid tier management

### Out of Scope (v1.1+)
- Generative materials design (CVAE + RL models)
- Phase diagram generator
- Literature mining and knowledge graph
- Cloud DFT job submission
- Experimental data tracking
- Team collaboration and multi-member projects
- Custom model fine-tuning on user data
- SHAP feature attributions and prediction explanations
- API access for programmatic workflows

### MVP Timeline: 10 weeks
- Week 1-2: Data model design, FastAPI scaffolding, user auth, structure file parsers (CIF/POSCAR), Materials Project API integration
- Week 3-4: GNN model serving pipeline (load pre-trained MEGNet/M3GNet, ONNX export, GPU inference endpoint), batch prediction queue with Celery
- Week 5-7: React frontend — structure viewer (Three.js), property prediction cards, materials table with sorting/filtering, database search interface
- Week 8-9: Billing integration (Stripe), rate limiting by tier, CSV batch upload/download, deployment to Kubernetes with GPU node pool
- Week 10: End-to-end testing, performance optimization, documentation, beta launch to 20 academic groups for feedback
