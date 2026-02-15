# MolForge — AI-Driven Molecular Design and Drug Candidate Screening

## Executive Summary

MolForge is a cloud-native platform that brings AI-powered de novo molecular design, virtual screening, and ADMET prediction to pharma R&D teams and computational chemists via the browser. It replaces six-figure desktop software suites with an accessible SaaS that generates novel drug-like molecules, docks them against protein targets, and predicts pharmacokinetic properties — all without local HPC infrastructure.

---

## Problem Statement

**The pain:**
- Schrödinger Suite costs $50K+ per seat per year, locking out biotech startups and academic labs from state-of-the-art computational drug discovery
- Traditional high-throughput screening (HTS) campaigns cost $500K–$2M and take 6–12 months to identify viable hit compounds
- Computational chemists spend weeks manually setting up docking runs, curating molecular libraries, and post-processing results across disconnected tools (AutoDock, GROMACS, MOE, custom Python scripts)
- ADMET failures account for ~40% of drug candidate attrition in clinical trials, yet predictive ADMET modeling is rarely integrated into early-stage lead identification
- Collaboration between medicinal chemists, computational chemists, and biologists is fragmented across email, PowerPoint slides, and shared drives with no provenance tracking

**Current workarounds:**
- Cobbling together open-source tools (RDKit, AutoDock Vina, Open Babel) with custom Python glue code — fragile, non-reproducible, requires deep expertise
- Paying $50K–$150K/seat/year for Schrödinger or MOE and running on expensive GPU workstations
- Outsourcing virtual screening to CROs at $100K+ per campaign with multi-week turnaround
- Using free web tools (SwissADME, SwissDock) that lack integration, throughput, and collaboration

**Market size:** The global computational drug discovery market was valued at $5.5B in 2024 and is projected to reach $14B by 2030 (CAGR ~17%). There are approximately 6,000 pharma/biotech companies and 15,000+ academic drug discovery labs worldwide that represent the addressable market.

---

## Target Users

### Primary Personas

**1. Dr. Elena Vasquez — Computational Chemist at a Mid-Size Biotech**
- Manages virtual screening campaigns against 3–5 active protein targets
- Currently uses a mix of Schrödinger Glide and custom RDKit scripts on a local GPU workstation
- Spends 30% of her time on infrastructure (setting up docking grids, managing job queues, converting file formats)
- Needs: an integrated platform that handles molecular prep, docking, and ADMET in one workflow with cloud compute

**2. Dr. James Okafor — Medicinal Chemist and Team Lead at a Pharma R&D Division**
- Designs SAR series by hand, iterating on lead scaffolds based on intuition and published data
- Receives docking results as spreadsheets and 2D plots from the computational team — slow feedback loop
- Needs: interactive lead optimization with real-time ADMET feedback and 3D protein-ligand visualization he can use without command-line expertise

**3. Dr. Mei-Lin Huang — Principal Investigator at an Academic Drug Discovery Lab**
- Runs a lab of 6 PhD students and postdocs working on neglected tropical diseases
- Has zero budget for commercial software; relies entirely on open-source tools and student-written scripts
- Needs: a free or affordable platform with modern AI generative design that her students can use without months of training

### Secondary Personas
- Biotech startup founders exploring drug discovery feasibility before raising Series A
- CRO scientists managing multiple client screening campaigns who need reproducibility and reporting
- Pharmaceutical patent attorneys needing prior art molecular similarity searches against patent databases

---

## Solution Overview

MolForge is a browser-based drug discovery workbench that:
1. Lets users define a protein target (upload PDB/mmCIF or fetch from RCSB PDB) and prepare the binding site with automated protonation and grid generation
2. Generates novel drug-like molecules via AI generative models (variational autoencoders, reinforcement learning) constrained to desired physicochemical properties and synthetic accessibility
3. Runs cloud-scale virtual screening by docking thousands of generated or uploaded molecules against the target using AutoDock Vina and scoring with ML re-scoring functions
4. Predicts ADMET properties (absorption, distribution, metabolism, excretion, toxicity) for all hits using ensemble ML models trained on ChEMBL and proprietary datasets
5. Provides an interactive hit-to-lead pipeline where medicinal chemists can explore SAR, visualize protein-ligand interactions in 3D, and iteratively refine candidates with AI suggestions

---

## Core Features

### F1: Molecular Editor and Library Management
- Browser-based 2D structure editor (SMILES, Sketcher, or paste SDF/MOL2)
- Bulk import from SDF, CSV (with SMILES column), or SMILES files (up to 1M compounds)
- Built-in compound libraries: FDA-approved drugs, natural products, fragment libraries (ZINC, Enamine REAL)
- Molecule standardization pipeline: salt stripping, tautomer canonicalization, charge assignment, stereochemistry validation
- Substructure and similarity search (Tanimoto, Dice) across uploaded and built-in libraries
- Tag, filter, and organize compounds into collections with custom metadata fields

### F2: AI De Novo Molecular Design
- Generate novel molecules optimized for target-specific binding using reinforcement learning with a GNN-based scoring function
- Variational autoencoder (VAE) trained on ChEMBL 34 to produce synthetically accessible, drug-like molecules
- User-defined constraints: molecular weight range, logP, topological polar surface area (TPSA), number of H-bond donors/acceptors, rotatable bonds, specific scaffold or R-group requirements
- Multi-objective optimization: simultaneously optimize binding affinity, selectivity, ADMET, and synthetic accessibility
- Diversity control: ensure generated libraries cover chemical space (MaxMin diversity picking)
- Generate 1K–100K novel molecules per run with cloud GPU acceleration (typically 15–45 minutes)

### F3: Protein Target Preparation
- Fetch structures directly from RCSB PDB by ID or upload custom PDB/mmCIF files
- Automated protein preparation: add hydrogens, assign charges (AMBER ff14SB), optimize H-bond networks, remove crystallographic waters (with selectable distance threshold)
- Binding site detection using fpocket algorithm with manual override via 3D viewer
- Grid box definition with visual 3D bounding box editor
- Support for homology models (upload from AlphaFold or Swiss-Model)
- Multi-chain and multi-conformer ensemble docking support

### F4: Virtual Screening and Molecular Docking
- Cloud-based AutoDock Vina docking with automatic parallelization (up to 10,000 molecules/hour on standard tier)
- Flexible ligand docking with configurable exhaustiveness and number of binding modes
- ML re-scoring using a trained XGBoost model on PDBbind refined set to improve hit enrichment over raw Vina scores
- Consensus scoring across multiple scoring functions (Vina, RF-Score, OnionNet-2)
- Post-docking filtering: strain energy penalty, clash detection, pharmacophore matching
- Results ranked by composite score with interactive scatter plots (affinity vs. ADMET properties)

### F5: ADMET Property Prediction
- Ensemble ML models for 25+ ADMET endpoints:
  - **Absorption:** Caco-2 permeability, HIA, Pgp substrate/inhibitor, aqueous solubility
  - **Distribution:** plasma protein binding, VDss, BBB penetration, fraction unbound
  - **Metabolism:** CYP450 inhibition (1A2, 2C9, 2C19, 2D6, 3A4), CYP substrate classification, metabolic stability (human liver microsomes)
  - **Excretion:** half-life, total clearance, renal OCT2 substrate
  - **Toxicity:** hERG inhibition, AMES mutagenicity, hepatotoxicity (DILI), acute oral toxicity (LD50), skin sensitization
- Traffic-light visualization (green/amber/red) for each property with confidence intervals
- Applicability domain assessment: flag predictions outside the model's training distribution
- Batch ADMET profiling for entire libraries with exportable reports (PDF, CSV)

### F6: Lead Optimization and SAR Analysis
- Interactive R-group decomposition: define a common scaffold and enumerate R-group variations
- Matched molecular pair (MMP) analysis to identify single-atom or fragment changes that improve properties
- Activity cliff detection: highlight pairs of similar molecules with large potency differences
- AI-suggested modifications: given a lead molecule, suggest top-10 single-point modifications to improve binding or ADMET
- Free-energy perturbation (FEP) lite: relative binding free energy estimates via ML surrogate model
- SAR table with sortable/filterable columns and embedded structure images

### F7: 3D Protein-Ligand Visualization
- WebGL-based molecular viewer (NGL Viewer integration) with protein surface rendering
- Display binding poses with hydrogen bonds, hydrophobic contacts, pi-stacking, salt bridges
- Overlay multiple docking poses for visual comparison
- Pharmacophore feature display (HBA, HBD, aromatic, hydrophobic, positive, negative)
- Publication-quality rendering with ray tracing mode
- Measurement tools: distance, angle, dihedral

### F8: Hit-to-Lead Pipeline Manager
- Kanban-style pipeline: Hit → Validated Hit → Lead → Optimized Lead → Candidate
- Attach docking results, ADMET profiles, and experimental data to each compound card
- Decision gates with configurable pass/fail criteria at each stage
- Audit trail logging every action and decision with timestamps and user attribution
- Export pipeline snapshots as PDF reports for stakeholder presentations

### F9: Collaboration and Lab Notebooks
- Project-based workspace with role-based access (viewer, editor, admin)
- Integrated electronic lab notebook (ELN): document hypotheses, methods, results with rich text, embedded structures, and plots
- Comment threads on individual molecules, docking runs, or notebook entries
- Real-time cursors and presence indicators for simultaneous editing
- Version history for all edits with diff view

### F10: Patent Landscape Check
- Search generated molecules against patent databases (SureChEMBL, Google Patents) using structural similarity (Tanimoto > 0.85 flagged)
- Freedom-to-operate heatmap: visualize chemical space overlap with existing patents
- Markush structure comparison for broad patent claims (v1.1)
- Export patent landscape report with citations and similarity scores

### F11: Workflow Automation and Reproducibility
- Define reusable workflows: target prep → generate → dock → filter → ADMET → rank
- Every run logged with full parameters, software versions, and input data (reproducibility hash)
- Schedule recurring screens against updated compound libraries
- API access for programmatic workflow triggering from external pipelines (e.g., Jupyter, Airflow)
- Export complete run provenance as JSON-LD for regulatory submissions

### F12: Reporting and Export
- One-click generation of screening campaign reports (PDF) with executive summary, methods, top hits, and ADMET profiles
- Export molecules as SDF, SMILES, CSV, or MOL2
- Export docking results with poses as PDB or MOL2 for downstream MD simulations
- Integration with ELN systems (Benchling, Dotmatics) via API
- Configurable report templates with organization branding

---

## Technical Architecture

### System Diagram

```
                          ┌─────────────────────────────┐
                          │     React Frontend (SPA)     │
                          │  Mol Editor │ 3D Viewer │ UI │
                          └──────────┬──────────────────┘
                                     │ HTTPS
                                     ▼
                          ┌─────────────────────────────┐
                          │   FastAPI Gateway (Python)    │
                          │   Auth │ Rate Limit │ Router  │
                          └──┬────────┬────────┬────────┘
                             │        │        │
                    ┌────────┘   ┌────┘   ┌────┘
                    ▼            ▼        ▼
            ┌──────────┐  ┌──────────┐  ┌──────────────────┐
            │ PostgreSQL│  │  Redis   │  │  S3 / MinIO      │
            │ (metadata,│  │ (queue,  │  │ (SDF files, PDB, │
            │  users,   │  │  cache,  │  │  results, models)│
            │  results) │  │  pubsub) │  │                  │
            └──────────┘  └────┬─────┘  └──────────────────┘
                               │
                    ┌──────────┼──────────────┐
                    ▼          ▼              ▼
            ┌────────────┐ ┌────────────┐ ┌────────────────┐
            │ Docking    │ │ AI Gen     │ │ ADMET          │
            │ Workers    │ │ Workers    │ │ Workers        │
            │ (CPU/GPU)  │ │ (GPU)      │ │ (CPU)          │
            │ AutoDock   │ │ PyTorch    │ │ XGBoost/PyTorch│
            │ Vina       │ │ VAE/RL     │ │ Ensemble       │
            └────────────┘ └────────────┘ └────────────────┘
                    │          │              │
                    └──────────┼──────────────┘
                               ▼
                    ┌────────────────────┐
                    │  Kubernetes (EKS)  │
                    │  Auto-scaling GPU  │
                    │  Node Pools        │
                    └────────────────────┘
```

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | React 18 + TypeScript, Tailwind CSS, Ketcher (2D mol editor), NGL Viewer (3D) |
| Backend API | Python 3.12, FastAPI, Pydantic v2, Uvicorn |
| Cheminformatics | RDKit 2024.03, Open Babel 3.1, Meeko (AutoDock prep) |
| AI/ML Models | PyTorch 2.x (VAE, GNN), XGBoost, scikit-learn, DeepChem |
| Docking Engine | AutoDock Vina 1.2.5, SMINA (fork with custom scoring) |
| Database | PostgreSQL 16 (RDKit cartridge for chemical search) |
| Queue/Cache | Redis 7 + Celery (distributed task queue) |
| Object Storage | AWS S3 (molecule files, PDB structures, results archives) |
| Compute | Kubernetes (EKS) with GPU node pools (NVIDIA A10G for AI, CPU for docking) |
| Monitoring | Prometheus + Grafana, Sentry, structured logging with Loki |

### Data Model

```
Organization
├── id, name, slug, plan, created_at
│
├── Project[]
│   ├── id, org_id, name, description, created_at
│   ├── target_protein_id (FK), status (active/archived)
│   │
│   ├── ProteinTarget[]
│   │   ├── id, project_id, name, pdb_id, source (rcsb/upload/alphafold)
│   │   ├── pdb_file_url (S3), prepared_file_url (S3)
│   │   ├── binding_site_residues (JSONB), grid_center (x,y,z), grid_size (x,y,z)
│   │   └── preparation_params (JSONB), created_at
│   │
│   ├── MoleculeLibrary[]
│   │   ├── id, project_id, name, source (upload/generated/builtin)
│   │   ├── molecule_count, sdf_file_url (S3)
│   │   └── generation_run_id (FK, nullable), created_at
│   │
│   ├── Molecule[]
│   │   ├── id, library_id, smiles (canonical), inchi_key (indexed)
│   │   ├── mol_weight, logp, tpsa, hbd, hba, rotatable_bonds
│   │   ├── morgan_fp (bit vector, for similarity search)
│   │   ├── properties (JSONB — extended descriptors)
│   │   └── tags[], created_at
│   │
│   ├── DockingRun[]
│   │   ├── id, project_id, target_id, library_id
│   │   ├── status (queued/running/completed/failed), progress (0-100)
│   │   ├── params (exhaustiveness, num_modes, energy_range)
│   │   ├── started_at, completed_at, molecule_count, hit_count
│   │   └── results_url (S3)
│   │
│   ├── DockingResult[]
│   │   ├── id, run_id, molecule_id
│   │   ├── vina_score, ml_rescore, consensus_score
│   │   ├── pose_file_url (S3), num_poses, best_pose_index
│   │   ├── strain_energy, clash_count
│   │   └── rank, created_at
│   │
│   ├── ADMETProfile[]
│   │   ├── id, molecule_id, run_id (nullable)
│   │   ├── predictions (JSONB — key: endpoint, value: {score, confidence, flag})
│   │   ├── applicability_domain_flag (boolean)
│   │   └── created_at
│   │
│   ├── GenerationRun[]
│   │   ├── id, project_id, target_id
│   │   ├── method (vae/rl/scaffold_hop), params (JSONB)
│   │   ├── constraints (JSONB — MW, logP, TPSA ranges, required substructures)
│   │   ├── status, molecule_count_generated, duration_seconds
│   │   └── created_at
│   │
│   └── PipelineCompound[]
│       ├── id, project_id, molecule_id
│       ├── stage (hit/validated_hit/lead/optimized_lead/candidate)
│       ├── experimental_data (JSONB), notes
│       └── promoted_by (user_id), promoted_at
│
├── Notebook[]
│   ├── id, project_id, author_id, title, content (rich text JSONB)
│   ├── version, parent_version_id
│   └── created_at, updated_at
│
└── User[]
    ├── id, org_id, email, name, role (admin/editor/viewer)
    ├── auth_provider, last_login
    └── created_at
```

### API Design

```
Auth:
POST   /api/auth/register                    # Create account
POST   /api/auth/login                       # Email/password login
POST   /api/auth/oauth/{provider}            # OAuth flow (Google, ORCID)
POST   /api/auth/refresh                     # Refresh JWT token

Projects:
GET    /api/projects                          # List projects
POST   /api/projects                          # Create project
GET    /api/projects/:id                      # Get project details
PATCH  /api/projects/:id                      # Update project
DELETE /api/projects/:id                      # Archive project

Protein Targets:
POST   /api/projects/:id/targets              # Upload or fetch PDB
GET    /api/projects/:id/targets              # List targets
POST   /api/targets/:id/prepare               # Run protein preparation
GET    /api/targets/:id/binding-sites         # Get detected binding sites
PATCH  /api/targets/:id/grid                  # Update grid box definition

Molecule Libraries:
POST   /api/projects/:id/libraries            # Create/upload library
GET    /api/projects/:id/libraries            # List libraries
GET    /api/libraries/:id/molecules           # List molecules (paginated)
POST   /api/libraries/:id/search              # Substructure/similarity search
DELETE /api/libraries/:id                     # Delete library

AI Generation:
POST   /api/projects/:id/generate             # Start de novo generation run
GET    /api/generation-runs/:id               # Get run status and results
POST   /api/generation-runs/:id/cancel        # Cancel running generation

Docking:
POST   /api/projects/:id/dock                 # Start docking campaign
GET    /api/docking-runs/:id                  # Get run status and progress
GET    /api/docking-runs/:id/results          # Get ranked results (paginated)
GET    /api/docking-results/:id/poses         # Get docking poses for a molecule
POST   /api/docking-runs/:id/cancel           # Cancel running dock

ADMET:
POST   /api/molecules/admet                   # Predict ADMET for molecule(s)
GET    /api/molecules/:id/admet               # Get ADMET profile
POST   /api/libraries/:id/admet-batch         # Batch ADMET for entire library

Pipeline:
GET    /api/projects/:id/pipeline             # Get pipeline compounds by stage
POST   /api/pipeline/promote                  # Move compound to next stage
PATCH  /api/pipeline/:id                      # Update compound notes/data

Notebooks:
GET    /api/projects/:id/notebooks            # List notebooks
POST   /api/projects/:id/notebooks            # Create notebook
PATCH  /api/notebooks/:id                     # Update notebook content
GET    /api/notebooks/:id/versions            # Version history

Patent Search:
POST   /api/molecules/patent-check            # Check molecule(s) against patents
GET    /api/patent-checks/:id                 # Get patent check results

Export:
POST   /api/export/report                     # Generate PDF report
POST   /api/export/molecules                  # Export molecules (SDF/CSV/SMILES)
POST   /api/export/docking-results            # Export docking results
```

---

## UI/UX — Key Screens

### 1. Project Dashboard
- Overview cards: active targets, total molecules, completed docking runs, pipeline compound counts by stage
- Recent activity feed (runs completed, molecules promoted, notebooks updated)
- Quick actions: new target, upload library, start generation, start docking
- Project-level settings and team member list

### 2. Molecular Editor and Library Browser
- Left panel: library tree with compound counts and source labels
- Center: 2D structure editor (Ketcher) for drawing or editing molecules, with SMILES input bar
- Right panel: selected molecule property card (MW, logP, TPSA, HBD, HBA, Lipinski violations)
- Bottom: filterable data table of molecules with inline structure thumbnails, sortable by any property
- Toolbar: import (SDF/CSV), export, bulk select, similarity search, substructure search

### 3. AI Generation Studio
- Target selector and binding site preview (3D thumbnail)
- Constraint sliders: MW range, logP range, TPSA range, HBD/HBA limits, synthetic accessibility threshold
- Scaffold constraint input: draw a required substructure or specify SMARTS pattern
- Method selector: VAE (explore diverse space), RL (optimize for target), scaffold hopping
- Progress indicator with live count of molecules generated and property distribution histograms updating in real time
- Results gallery: grid of generated molecules with predicted scores, click to inspect

### 4. Docking Control Center
- Campaign setup wizard: select target, select library, configure docking parameters (exhaustiveness, # modes)
- Live progress dashboard: molecules docked / total, estimated time remaining, real-time score distribution histogram
- Results table: ranked molecules with Vina score, ML re-score, consensus score, ADMET traffic lights
- Click any row to open the 3D viewer with the protein-ligand binding pose
- Scatter plot: binding score vs. any ADMET property, with click-to-select for compound inspection

### 5. 3D Protein-Ligand Viewer
- Full-screen WebGL viewer with protein surface (hydrophobicity-colored or electrostatic potential)
- Ligand displayed in ball-and-stick with binding interactions annotated (H-bonds as dashed lines, hydrophobic contacts as green surfaces)
- Side panel: interaction summary table (residue, type, distance), ligand properties, ADMET mini-profile
- Overlay toggle: compare up to 4 binding poses side by side
- Export: screenshot (PNG), session state (JSON), pose (PDB)

### 6. Hit-to-Lead Pipeline
- Kanban board with columns: Hit, Validated Hit, Lead, Optimized Lead, Candidate
- Each card shows: 2D structure, key scores (affinity, ADMET flags), stage entry date
- Drag-and-drop to promote/demote compounds between stages
- Click card to expand: full docking results, ADMET profile, SAR context, notes, experimental data fields
- Filter and sort within each column; export pipeline snapshot as PDF

---

## Monetization

### Free Tier (Academic)
- Requires .edu or institutional email verification
- 1 project, 1 protein target
- AI generation: up to 500 molecules per month
- Docking: up to 500 molecules per month
- ADMET prediction: up to 200 molecules per month
- Basic ADMET endpoints only (Lipinski, logP, solubility, BBB, hERG)
- Community support only, MolForge watermark on exports

### Researcher — $199/month
- 5 projects, unlimited targets
- AI generation: up to 25,000 molecules per month
- Docking: up to 25,000 molecules per month (standard priority)
- Full ADMET panel (25+ endpoints)
- Patent landscape check: 50 queries per month
- PDF report generation
- Email support (48h response)

### Lab — $799/month
- Unlimited projects and targets
- AI generation: up to 250,000 molecules per month
- Docking: up to 250,000 molecules per month (high priority, dedicated GPU allocation)
- Full ADMET panel with custom model training on proprietary data
- Unlimited patent checks
- Collaboration: up to 15 team members with role-based access
- Lab notebooks and pipeline manager
- API access for workflow automation
- Priority support (24h response), onboarding session

### Enterprise — Custom
- Unlimited everything with dedicated GPU cluster
- Single sign-on (SSO/SAML), audit logs, data residency options
- Custom ML model training on proprietary compound-activity datasets
- Private deployment option (VPC or on-premise)
- Integration with existing ELN (Benchling, Dotmatics, IDBS)
- Dedicated customer success manager and SLA guarantee
- 21 CFR Part 11 compliance features (electronic signatures, audit trail)

---

## Go-to-Market Strategy

### Phase 1: Academic Seeding (Month 1-3)
- Launch free academic tier and announce on computational chemistry mailing lists (CCL), Reddit r/cheminformatics, and Twitter/X #CompChem
- Publish benchmark study comparing MolForge AI generation and docking accuracy against Schrödinger Glide and AutoDock Vina standalone
- Partner with 5–10 academic drug discovery labs as beta testers, co-author preprint on bioRxiv
- Present at RDKit UGM (User Group Meeting) and ACS Spring meeting poster session
- Offer free Lab tier for 6 months to any lab that publishes results using MolForge

### Phase 2: Biotech Penetration (Month 3-6)
- Launch Researcher and Lab tiers with self-serve billing
- Content marketing: blog series on AI-driven drug discovery workflows, SEO targeting "virtual screening SaaS", "cloud docking platform", "ADMET prediction tool"
- Webinar series: "From Target to Lead in 48 Hours with MolForge" with guest academic speakers
- Attend BIO International Convention and JP Morgan Healthcare Conference satellite events
- Launch referral program: existing users get 1 month free for each converted referral

### Phase 3: Enterprise and Pharma (Month 6-12)
- Build enterprise features: SSO, audit logs, private deployment, ELN integration
- Hire pharma-focused sales team (2 account executives with life sciences background)
- Target mid-size pharma R&D divisions ($1B–$20B revenue companies) with pilot programs
- Develop case studies from biotech customers showing time and cost savings
- Submit for SOC 2 Type II certification and begin 21 CFR Part 11 compliance work

### Acquisition Channels
- Organic search: target "AI drug discovery platform", "virtual screening cloud", "ADMET prediction SaaS"
- Academic word-of-mouth from free tier users (computational chemistry is a tight community)
- Conference presentations and poster sessions at ACS, AAPS, MedChem conferences
- Integration listings on RDKit and Open Babel community pages
- LinkedIn targeted ads to "Computational Chemist" and "Medicinal Chemist" job titles

---

## Success Metrics & KPIs

| Metric | Target (Month 6) | Target (Month 12) |
|--------|-------------------|---------------------|
| Registered users | 2,000 | 8,000 |
| Active projects (monthly) | 500 | 2,500 |
| Molecules generated (cumulative) | 5M | 50M |
| Docking runs completed (cumulative) | 2,000 | 15,000 |
| Paying customers | 40 | 180 |
| MRR | $15,000 | $80,000 |
| Free → Paid conversion | 4% | 7% |
| Churn rate (monthly) | <6% | <4% |

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | MolForge Advantage |
|-----------|-----------|------------|-------------------|
| Schrödinger Suite | Gold-standard accuracy (Glide, FEP+), deep pharma trust, extensive force fields | $50K+/seat, requires HPC infrastructure, steep learning curve, desktop-only | 10x cheaper, cloud-native, AI generation built-in, no infrastructure needed |
| Recursion Pharmaceuticals | Massive proprietary biological datasets, phenotypic screening integration | Platform not available to external users, focused on internal pipeline | Open platform accessible to all, molecular-level focus complementary to phenotypic |
| Insilico Medicine | Strong AI generative models (Chemistry42), end-to-end pipeline | Primarily a drug development company, not a SaaS tool; limited external access | Purpose-built SaaS with self-serve access, transparent pricing, collaboration tools |
| ChemAxon (Marvin/JChem) | Excellent cheminformatics toolkit, structure search, property calculation | No AI generative design, no integrated docking, on-premise heavy | Full discovery pipeline: generate → dock → ADMET → optimize in one platform |
| OpenEye (Cadence) | Fast OMEGA conformer generation, good shape-based screening (ROCS) | Acquired by Cadence, uncertain product direction, expensive | Independent SaaS with modern AI, transparent roadmap, lower cost of entry |

---

## Risks & Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| AI-generated molecules are synthetically infeasible | High | Medium | Integrate synthetic accessibility score (SAScore) as hard constraint; partner with Enamine for REAL space validation; add retrosynthetic analysis in v1.1 |
| Docking accuracy insufficient for pharma standards | High | Medium | Offer consensus scoring (multiple methods), benchmark rigorously against published DUD-E datasets, allow users to bring custom scoring functions, add FEP in roadmap |
| GPU compute costs erode margins at scale | Medium | High | Implement smart caching (identical molecules skip re-docking), tiered compute allocation, negotiate reserved GPU instances, optimize batch sizes |
| Regulatory concerns about AI in drug discovery | Medium | Low | Maintain full audit trails and reproducibility logging, pursue SOC 2 and 21 CFR Part 11, clearly position as decision-support (not autonomous) |
| Large pharma prefers on-premise due to IP sensitivity | High | Medium | Offer private VPC deployment at Enterprise tier, implement end-to-end encryption, obtain ISO 27001 certification |

---

## MVP Scope (v1.0)

### In Scope
- User auth, organization and project management
- PDB upload and basic protein preparation (hydrogen addition, grid box definition)
- 2D molecular editor with SMILES input and SDF import
- AI de novo molecular generation (VAE model) with basic property constraints
- Virtual screening with AutoDock Vina (cloud-based, up to 1,000 molecules per run)
- ADMET prediction for top 10 endpoints (Lipinski properties, solubility, BBB, hERG, CYP3A4)
- Basic 3D protein-ligand viewer with binding pose display
- Results table with scoring, sorting, and filtering
- Export: SDF, CSV, PDF summary report

### Out of Scope (v1.1+)
- Reinforcement learning generation and scaffold hopping methods
- Consensus scoring and ML re-scoring
- Full ADMET panel (25+ endpoints) and custom model training
- Patent landscape checking
- Hit-to-lead pipeline manager (Kanban)
- Collaborative notebooks and real-time editing
- ELN integration (Benchling, Dotmatics)
- Enterprise features (SSO, audit logs, private deployment)
- Retrosynthetic route prediction
- Molecular dynamics integration

### MVP Timeline: 10 weeks
- Week 1-2: Auth system, project/org data model, protein target upload and basic preparation pipeline, S3 storage setup
- Week 3-4: Molecular editor integration (Ketcher), library management, SDF/SMILES import, RDKit property calculation, PostgreSQL chemical search
- Week 5-6: AI generative model serving (VAE inference on GPU), constraint engine, generation job queue with Celery
- Week 7-8: AutoDock Vina docking pipeline (parallelized on Kubernetes), results aggregation, ADMET model serving, results table UI
- Week 9-10: 3D viewer integration (NGL), export functionality, billing (Stripe), free/paid tier enforcement, polish, load testing, beta launch
