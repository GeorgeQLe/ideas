# FoldSight — Protein Structure Prediction and Visualization Platform

## Executive Summary

FoldSight is a cloud-based platform that lets researchers predict 3D protein structures from amino acid sequences, visualize them interactively in the browser, and collaborate with colleagues on structural analysis — all without installing desktop software or managing GPU infrastructure. By wrapping state-of-the-art structure prediction models in an intuitive interface with rich annotation and comparison tools, FoldSight democratizes computational structural biology for every lab.

---

## Problem Statement

**The pain:**
- Running AlphaFold2/3 or ESMFold locally requires expensive GPU hardware (A100-class), complex dependency management (CUDA, OpenMM, conda environments), and hours of setup time that most bench biologists cannot handle
- PyMOL, the de facto visualization tool, costs $100+/seat for academic licenses and has a 1990s-era interface that intimidates non-computational researchers
- The Schrodinger Suite, used by pharma for structure-based drug design, costs $10,000-50,000+ per year per seat, putting advanced binding site analysis out of reach for academic labs
- Sharing structural biology findings requires exporting static images or PDB files via email; there is no live, collaborative way to annotate and discuss structures with remote colleagues
- Researchers waste days manually aligning structures, comparing conformations, and cross-referencing annotations from UniProt, PDB, and literature when these could be automated

**Current workarounds:**
- Using Google Colab notebooks (ColabFold) to run AlphaFold, but with limited GPU availability, session timeouts, and no integrated visualization
- Installing PyMOL or ChimeraX on individual workstations with inconsistent versions and plugin configurations across lab members
- Manually downloading PDB files, uploading to web viewers (Mol*), and losing analysis context between tools
- Paying for Schrodinger Suite licenses that sit idle 80% of the time because only one or two lab members are trained on the software

**Market size:** The global structural biology market is valued at approximately $3.8 billion (2024) with 12% CAGR driven by AI-powered drug discovery. There are over 200,000 active structural biology researchers worldwide across academia and industry. The computational biology software segment is approximately $1.2 billion, with cloud-based tools growing fastest as labs move away from on-premise infrastructure.

---

## Target Users

### Primary Personas

**1. Dr. Chen — Postdoctoral Researcher in Structural Biology**
- Studies protein-protein interactions in a university biochemistry department
- Knows how to interpret structures but struggles with computational tools (not a programmer)
- Currently uses ColabFold for predictions and PyMOL for visualization, losing context switching between them
- Needs: a single platform where she can go from sequence to predicted structure to annotated figure in one workflow, without writing code

**2. Dr. Okafor — Principal Investigator at a Biotech Startup**
- Leads a 12-person R&D team doing structure-based drug design for oncology targets
- Team uses a mix of PyMOL, ChimeraX, and Schrodinger with no standardization
- Needs: a collaborative platform where medicinal chemists, computational biologists, and structural biologists can share annotated structures and binding site analyses in real time

**3. Maya — Graduate Student in Bioinformatics**
- Analyzing the effects of thousands of missense variants on protein stability for her dissertation
- Needs to predict structures for hundreds of protein variants and compare them systematically
- Needs: batch prediction pipeline with programmatic API access and automated structural comparison, plus clear visualizations for her thesis figures

### Secondary Personas
- Pharmaceutical R&D teams evaluating target druggability and performing virtual screening campaigns
- Undergraduate biology students learning protein structure fundamentals in coursework
- Bioinformatics core facility staff providing structure prediction as a service to their institution

---

## Solution Overview

FoldSight is a browser-based structural biology platform that:
1. Accepts amino acid sequences (single or batch), fetches known structures from PDB/UniProt if available, and runs GPU-accelerated structure prediction using state-of-the-art models (ESMFold, OpenFold, ColabFold/MMseqs2)
2. Renders predicted and experimental structures in an interactive 3D viewer built on Mol*/Three.js with publication-quality rendering, including ribbon, surface, ball-and-stick, and electrostatic potential representations
3. Provides automated structural analysis tools: binding site detection, structural alignment (TM-align), variant impact scoring (stability ddG estimation), and domain annotation mapping from InterPro/Pfam
4. Enables real-time collaborative annotation where multiple researchers can simultaneously view, mark up, and discuss structures with persistent comments anchored to specific residues or regions
5. Exports results as PDB/mmCIF files, publication-ready figures (PNG/SVG), PyMOL session files, and structured reports for supplementary materials

---

## Core Features

### F1: Sequence Input and Structure Prediction
- Paste or upload amino acid sequences in FASTA format (single sequence or multi-FASTA for batch jobs)
- Automatic sequence validation: check for non-standard residues, length limits, and format errors
- Model selection: ESMFold (fast, single-sequence), ColabFold/MMseqs2 (MSA-based, higher accuracy), OpenFold (full AlphaFold2 reimplementation)
- Multimer prediction for protein complexes (up to 8 chains) with stoichiometry specification
- Confidence scoring: per-residue pLDDT coloring and predicted aligned error (PAE) matrix visualization
- Automatic PDB/UniProt lookup: if an experimental structure exists for the input sequence, offer it alongside the prediction for comparison
- Job queue with estimated wait times, email notification on completion, and persistent result storage

### F2: Interactive 3D Structure Viewer
- WebGL-based molecular viewer built on Mol* (Molstar) framework with custom UI extensions
- Representation modes: cartoon/ribbon, ball-and-stick, spacefill (CPK), surface (van der Waals, solvent-accessible, molecular), wire
- Color schemes: by chain, by secondary structure, by pLDDT confidence, by hydrophobicity, by B-factor, by custom residue property
- Electrostatic surface potential visualization computed via APBS (Adaptive Poisson-Boltzmann Solver) on the server
- Ligand and small molecule display with 2D interaction diagram overlay
- Measurement tools: distance between atoms, angles between three atoms, dihedral angles, surface area of selections
- Selection language: click-to-select residues, chain selections, sequence range selections, proximity-based selections ("all residues within 5A of LIG")
- Smooth camera animations with keyframe-based presentation mode for creating structural walkthroughs
- Screenshot export at configurable resolution (up to 4K) with transparent background option for figures

### F3: Structure Comparison and Alignment
- Pairwise structural alignment using TM-align with TM-score, RMSD, and alignment length reporting
- Superposition viewer showing aligned structures with per-residue RMSD coloring on the deviation heatmap
- Multi-structure alignment for comparing an ensemble of conformations or homologous structures
- Sequence-structure alignment view: synchronized scrollable sequence alignment with 3D structure highlighting
- Conformational change analysis: identify hinge regions, domain movements, and loop rearrangements between two states
- Batch alignment: align a set of predicted variant structures against the wild-type reference and rank by structural deviation
- AlphaFold model confidence comparison: overlay pLDDT profiles of different models or prediction runs

### F4: Binding Site Analysis
- Automatic binding site detection using fpocket algorithm (geometry-based pocket detection)
- Druggability scoring for detected pockets based on volume, hydrophobicity, and enclosure
- Known ligand mapping: overlay ligands from homologous PDB structures onto predicted binding sites
- Interaction fingerprint analysis: hydrogen bonds, salt bridges, pi-stacking, hydrophobic contacts visualized as 2D diagrams
- Conservation mapping: color binding site residues by evolutionary conservation (from MSA via ConSurf methodology)
- Binding site comparison across homologous proteins to identify conserved pharmacophoric features
- Export pocket coordinates and properties for downstream virtual screening in external docking software

### F5: Annotation and Collaboration
- Residue-level annotations: click any residue to add notes, tags, or literature references
- Annotation layers: create named layers (e.g., "active site", "mutation hotspots", "epitope mapping") that can be toggled on/off
- Real-time collaborative viewing: multiple users see the same orientation with synchronized camera and cursor positions
- Persistent comments anchored to specific residues, regions, or structural features with @mention notifications
- Shared project workspaces with role-based permissions (viewer, annotator, editor, admin)
- Annotation import from UniProt feature annotations (domains, active sites, PTMs, variants, disulfide bonds)
- Lab notebook integration: timestamped audit trail of all annotations and analyses for reproducibility

### F6: Variant Impact Prediction
- Input a list of amino acid substitutions (e.g., A123V, G456D) and predict structural impact
- Stability change estimation (ddG) using FoldX energy calculations or Rosetta cartesian_ddg
- Per-variant structural perturbation visualization: overlay predicted mutant structure on wild-type
- Pathogenicity scoring integrating structural impact with evolutionary conservation and clinical databases (ClinVar, gnomAD)
- Batch mode: process thousands of variants from VCF files or gnomAD frequency data
- Generate variant impact report with ranked table of most destabilizing mutations and 3D visualization snapshots
- Integration with AlphaMissense predictions when available

### F7: Batch Prediction Pipeline
- Upload multi-FASTA files with hundreds of sequences for batch processing
- Priority queue management: assign priority levels to jobs, pause/resume, cancel
- Automatic clustering of similar sequences to avoid redundant predictions (using MMseqs2 clustering)
- Batch result dashboard with sortable table of all predictions, pLDDT summary statistics, and bulk download
- API access for programmatic job submission and result retrieval (Python SDK and REST API)
- Webhook notifications for pipeline completion to integrate with downstream analysis workflows
- Cost estimation before submission showing GPU-hours required and billing impact

### F8: Database Integration
- Direct search and import from Protein Data Bank (PDB): search by PDB ID, protein name, organism, resolution
- UniProt integration: fetch sequence, annotations, cross-references, and known structures for any UniProt accession
- AlphaFold Protein Structure Database integration: instantly load precomputed AlphaFold predictions for any UniProt entry
- InterPro domain annotation overlay: automatically map Pfam, SMART, and CDD domain boundaries onto structures
- PubMed literature linking: find and attach relevant publications to structures based on protein identifiers
- Custom database upload: teams can upload proprietary structure collections for internal search and comparison

### F9: Export and Publication Tools
- PDB and mmCIF file export with FoldSight annotations embedded as custom records
- PyMOL session file (.pse) export preserving colors, representations, and selections
- ChimeraX session export (.cxs) for users with desktop visualization preferences
- Publication figure export: PNG (up to 4K resolution), SVG (vector), and TIFF with configurable DPI
- Automated figure panel generation: create multi-panel figures showing different views/representations of the same structure
- Supplementary data package: ZIP file containing structure files, alignment data, variant analysis tables, and methods description text
- LaTeX/BibTeX citation generation for methods sections describing the prediction pipeline used

---

## Technical Architecture

### System Diagram

```
┌──────────────────────────────────────────────────────────┐
│                    Browser Client                         │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────┐  │
│  │  Sequence    │  │  3D Viewer   │  │  Analysis      │  │
│  │  Input &     │  │  (Mol*/      │  │  Panels        │  │
│  │  Job Mgmt    │  │   Three.js)  │  │  (Alignment,   │  │
│  │              │  │              │  │   Variants,    │  │
│  │              │  │              │  │   Binding)     │  │
│  └──────┬──────┘  └──────┬───────┘  └───────┬────────┘  │
│         └────────────────┼──────────────────┘            │
│                          ▼                                │
│              WebSocket (collaboration) + REST API          │
└──────────────────────────┼───────────────────────────────┘
                           │
                           ▼
                 ┌───────────────────┐
                 │   API Gateway     │
                 │   (FastAPI)       │
                 └──┬─────┬─────┬───┘
                    │     │     │
         ┌──────────┘     │     └──────────┐
         ▼                ▼                ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────────┐
│ Collaboration │ │  Analysis    │ │  Prediction      │
│ Service       │ │  Service     │ │  Job Scheduler   │
│ (WebSocket)   │ │  (Alignment, │ │  (Celery/Redis)  │
│               │ │   fpocket,   │ │                  │
│               │ │   FoldX)     │ └────────┬─────────┘
└──────┬───────┘ └──────┬───────┘          │
       │                │                  ▼
       │                │         ┌──────────────────┐
       ▼                ▼         │  GPU Worker Pool  │
┌──────────────┐ ┌────────────┐  │  ┌────────────┐  │
│  PostgreSQL  │ │   Redis    │  │  │ ESMFold    │  │
│  (Users,     │ │  (Cache,   │  │  │ OpenFold   │  │
│   Projects,  │ │   Queue,   │  │  │ ColabFold  │  │
│   Annots)    │ │   Sessions)│  │  │ APBS       │  │
└──────────────┘ └────────────┘  │  │ FoldX      │  │
                                 │  └────────────┘  │
                                 └────────┬─────────┘
                                          │
                                          ▼
                                 ┌──────────────────┐
                                 │  Object Storage   │
                                 │  (S3)             │
                                 │  PDB files,       │
                                 │  predictions,     │
                                 │  figures           │
                                 └──────────────────┘
```

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend Framework | React 18 + TypeScript, Vite, Tailwind CSS |
| 3D Molecular Viewer | Mol* (Molstar) library with custom React wrapper and Three.js extensions |
| Sequence Viewer | Custom React component with MSA visualization (inspired by react-msaviewer) |
| Backend API | Python 3.12, FastAPI with async endpoints, Pydantic v2 for validation |
| Task Queue | Celery with Redis broker for prediction job management |
| GPU Inference | PyTorch 2.x, ESMFold, OpenFold, ColabFold with MMseqs2, deployed on NVIDIA A100/H100 instances |
| Structure Analysis | BioPython, TM-align (C binary), fpocket, FoldX, APBS |
| Database | PostgreSQL 16 with pgvector for sequence similarity search |
| Cache / Message Broker | Redis 7 for caching, session management, Celery broker, and real-time pub/sub |
| Object Storage | AWS S3 for PDB files, prediction outputs, figure exports |
| Auth | Auth0 with ORCID OAuth for academic identity verification |
| Search | Elasticsearch for full-text search across protein names, organisms, and annotations |
| Hosting | AWS EKS (Kubernetes) for API and workers, Lambda Cloud / RunPod for GPU burst capacity |
| Monitoring | Datadog for infrastructure, Sentry for error tracking, custom Grafana dashboards for GPU utilization |

### Data Model

```
User
├── id (uuid), email, name, orcid_id (nullable)
├── institution, department, role
├── auth_provider, plan (free/researcher/lab/enterprise)
└── created_at, last_login_at

Organization (Lab / Company)
├── id (uuid), name, institution
├── owner_id → User
├── plan, seat_count
└── settings (default_model, storage_quota_gb)

Project
├── id (uuid), org_id → Organization
├── name, description, visibility (private/org/public)
├── created_by → User
└── created_at, updated_at

Structure
├── id (uuid), project_id → Project
├── name, source_type (predicted/experimental/uploaded)
├── sequence (text), sequence_length
├── organism, uniprot_accession (nullable), pdb_id (nullable)
├── prediction_model (esmfold/openfold/colabfold, nullable)
├── plddt_mean, plddt_per_residue (float[])
├── pae_matrix_url (S3 path, nullable)
├── structure_file_url (S3 path to mmCIF)
├── status (queued/running/completed/failed)
├── prediction_job_id (nullable)
└── created_by → User, created_at

PredictionJob
├── id (uuid), user_id → User
├── model (esmfold/openfold/colabfold)
├── input_sequences (JSONB: [{name, sequence}])
├── status (queued/running/completed/failed/cancelled)
├── gpu_type, gpu_hours_used
├── started_at, completed_at
├── error_message (nullable)
└── created_at, priority

Alignment
├── id (uuid), project_id → Project
├── structure_ids → Structure[] (array of aligned structure IDs)
├── method (tmalign/fatcat/ce)
├── tm_score, rmsd, aligned_length
├── transformation_matrices (JSONB)
├── per_residue_rmsd (float[])
└── created_by → User, created_at

Annotation
├── id (uuid), structure_id → Structure
├── author_id → User
├── layer_name (e.g., "active_site", "mutations")
├── residue_range_start, residue_range_end, chain_id
├── annotation_type (note/tag/highlight/measurement)
├── content (text), color (hex)
├── source (manual/uniprot/interpro/clinvar)
└── created_at, updated_at

Comment
├── id (uuid), project_id → Project
├── structure_id → Structure (nullable)
├── author_id → User
├── body (text), parent_comment_id (nullable, for threads)
├── anchor_residue, anchor_chain (nullable)
├── resolved (boolean)
└── created_at

VariantAnalysis
├── id (uuid), structure_id → Structure
├── variants (JSONB: [{position, wild_type, mutant, ddg, plddt_change, pathogenicity_score}])
├── method (foldx/rosetta/alphamissense)
├── status (queued/running/completed)
└── created_by → User, created_at

BindingSite
├── id (uuid), structure_id → Structure
├── detection_method (fpocket/manual)
├── pocket_residues (int[]), chain_id
├── volume_A3, druggability_score
├── hydrophobicity_score, enclosure_score
└── created_at
```

### API Design

```
Auth:
POST   /api/auth/login                        # Email/password or ORCID OAuth
POST   /api/auth/register                     # Account creation
POST   /api/auth/refresh                      # Token refresh
GET    /api/auth/me                            # Current user profile

Organizations:
GET    /api/orgs                               # List user's labs/orgs
POST   /api/orgs                               # Create organization
PATCH  /api/orgs/:id                           # Update settings
POST   /api/orgs/:id/members                   # Invite member
GET    /api/orgs/:id/usage                     # Storage and compute usage

Projects:
GET    /api/projects                           # List projects
POST   /api/projects                           # Create project
GET    /api/projects/:id                       # Get project with structures
PATCH  /api/projects/:id                       # Update project metadata
DELETE /api/projects/:id                       # Delete project

Predictions:
POST   /api/predict                            # Submit prediction job (single or batch)
GET    /api/predict/:job_id                    # Get job status and results
POST   /api/predict/:job_id/cancel             # Cancel running job
GET    /api/predict/queue                      # View queue position and estimates

Structures:
GET    /api/structures/:id                     # Get structure metadata
GET    /api/structures/:id/file                # Download structure file (PDB/mmCIF)
GET    /api/structures/:id/plddt               # Get per-residue pLDDT scores
GET    /api/structures/:id/pae                 # Get PAE matrix
POST   /api/structures/upload                  # Upload external structure file
DELETE /api/structures/:id                     # Delete structure

Analysis:
POST   /api/analysis/align                     # Structural alignment (pair or multi)
GET    /api/analysis/align/:id                 # Get alignment results
POST   /api/analysis/binding-sites/:struct_id  # Detect binding sites
GET    /api/analysis/binding-sites/:struct_id  # Get detected binding sites
POST   /api/analysis/variants                  # Submit variant impact analysis
GET    /api/analysis/variants/:id              # Get variant analysis results
POST   /api/analysis/electrostatics/:struct_id # Compute electrostatic surface

Annotations:
GET    /api/structures/:id/annotations         # List annotations on structure
POST   /api/structures/:id/annotations         # Add annotation
PATCH  /api/annotations/:id                    # Edit annotation
DELETE /api/annotations/:id                    # Remove annotation

Comments:
GET    /api/projects/:id/comments              # List comments in project
POST   /api/projects/:id/comments              # Add comment
PATCH  /api/comments/:id                       # Edit/resolve comment

External Databases:
GET    /api/external/pdb/search                # Search PDB by keyword, organism, resolution
GET    /api/external/pdb/:pdb_id               # Fetch structure from PDB
GET    /api/external/uniprot/:accession         # Fetch UniProt entry with annotations
GET    /api/external/alphafold/:uniprot_id      # Fetch AlphaFold DB prediction
GET    /api/external/interpro/:uniprot_id       # Fetch InterPro domain annotations

Export:
POST   /api/export/figure                      # Generate publication figure (PNG/SVG/TIFF)
POST   /api/export/pymol-session               # Generate PyMOL .pse file
POST   /api/export/report                      # Generate analysis report (PDF)
POST   /api/export/supplementary               # Generate supplementary data package (ZIP)

Collaboration:
WS     /ws/projects/:id                        # WebSocket for real-time collaboration
POST   /api/projects/:id/share                 # Generate share link
```

---

## UI/UX — Key Screens

### 1. Dashboard / Home
- Project cards showing protein name, organism, structure thumbnail (miniature 3D render), prediction status, and last-modified date
- Quick-start actions: "Predict from Sequence", "Upload Structure", "Search PDB", "Browse AlphaFold DB"
- Recent activity feed showing completed predictions, new annotations, and team comments
- Usage summary: predictions remaining this month, storage used, collaborators active

### 2. Prediction Submission
- Large text area for FASTA sequence input with syntax highlighting and validation feedback
- Model selector with comparison table: ESMFold (fast, ~30s), ColabFold (accurate, ~5min), OpenFold (research-grade, ~15min)
- Multimer toggle with chain count and stoichiometry configuration
- Batch upload zone for multi-FASTA files with drag-and-drop and file size/sequence count preview
- Job queue panel showing active, queued, and completed jobs with progress bars and estimated completion times

### 3. Structure Viewer (Primary Working Screen)
- Full-width 3D viewer occupying 70% of the screen with floating toolbar for representation and color scheme controls
- Left panel: structure hierarchy tree (chains, domains, ligands), annotation layers with toggles, selection inspector
- Right panel: context-sensitive analysis (residue properties when a residue is selected, pocket details when a binding site is selected)
- Bottom strip: zoomable sequence viewer with secondary structure annotation, pLDDT confidence bar chart, and clickable residue navigation
- Top toolbar: alignment tool, measurement mode, screenshot button, share button, export menu

### 4. Alignment and Comparison View
- Split-screen or superimposed view of two or more aligned structures with independent or synchronized camera controls
- Per-residue RMSD heatmap displayed on the sequence alignment below the 3D view
- Alignment statistics panel: TM-score, RMSD, sequence identity, aligned length, structural coverage
- Difference highlighting: regions with significant conformational change shown in warm colors (red/orange)

### 5. Variant Impact Dashboard
- Input panel: paste variant list (HGVS notation), upload VCF, or select from gnomAD frequency browser
- Results table: sortable by position, ddG, pLDDT change, pathogenicity score, with row-click to highlight variant in 3D viewer
- Heatmap visualization: sequence position vs. all possible substitutions colored by predicted impact
- 3D viewer overlay: variant positions displayed as spheres colored by impact severity (green=benign, red=destabilizing)

### 6. Export and Figure Builder
- Interactive figure composer: position 3D view, add labels, scale bars, color legends, and panel letters (A, B, C)
- Template gallery for common figure types: single structure, alignment comparison, binding site closeup, variant map
- Resolution and format selector: PNG (72-600 DPI), SVG (vector), TIFF (publication-ready)
- Preview panel showing exactly what will be exported with pixel dimensions
- One-click supplementary package generation with automated methods text

---

## Monetization

### Free Tier (Academic)
- 10 structure predictions per month (ESMFold only)
- 1 GB storage for structures and results
- 3D viewer with basic representations (cartoon, surface)
- Single-user, no collaboration features
- Community support via forum
- Must verify academic email or ORCID

### Researcher — $79/month
- 100 predictions per month (ESMFold + ColabFold)
- 25 GB storage
- Full viewer with all representations and electrostatic surfaces
- Structural alignment and binding site detection
- Variant impact analysis (up to 500 variants/month)
- Publication-quality figure export (PNG, SVG, TIFF)
- PyMOL session export
- Email support

### Lab — $299/month (up to 10 seats, $25/additional seat)
- 500 predictions per month (all models including OpenFold)
- 200 GB shared storage
- Everything in Researcher
- Real-time collaboration with comments and annotations
- Shared project workspaces with role-based permissions
- Batch prediction pipeline with API access
- Custom protein database upload (up to 50 GB)
- Priority GPU queue (2x faster average wait time)
- Priority support with 24-hour response

### Enterprise — Custom
- Unlimited predictions with dedicated GPU allocation
- Unlimited storage with data residency options (US, EU, Asia)
- SAML/SSO integration with institutional identity providers
- On-premise deployment option for pharmaceutical IP protection
- Custom model fine-tuning on proprietary protein families
- Dedicated customer success manager and onboarding
- SLA with 99.9% uptime guarantee
- Audit logging and compliance reporting (GxP readiness)
- Integration with internal LIMS and ELN systems

---

## Go-to-Market Strategy

### Phase 1: Academic Seeding (Month 1-3)
- Launch free tier specifically for academic researchers with .edu / institutional email verification
- Publish preprint on bioRxiv demonstrating FoldSight's prediction accuracy benchmarks vs. ColabFold notebooks
- Present at Structural Biology community conferences: Protein Society Annual Symposium, 3DSIG (ISMB satellite)
- Create tutorial series: "From Sequence to Structure in 5 Minutes" targeting wet-lab biologists
- Partner with 5 structural biology labs at top-tier universities for beta testing and testimonials

### Phase 2: Research Community Growth (Month 3-6)
- Launch Researcher tier with publication figure export as key differentiator
- SEO content targeting: "AlphaFold online", "protein structure prediction tool", "PyMOL alternative"
- Integration with preprint servers: "Powered by FoldSight" badge for structures generated on the platform
- Publish case studies showing time savings (hours of PyMOL scripting reduced to 5-minute FoldSight workflow)
- Launch FoldSight Community Library: curated, annotated structure collections for popular protein families

### Phase 3: Lab and Enterprise (Month 6-12)
- Launch Lab tier with collaboration features targeting multi-person research groups
- Partner with pharmaceutical companies for pilot programs in structure-based drug design workflows
- Integrate with electronic lab notebook (ELN) platforms: Benchling, Signals Notebook
- Launch API and Python SDK for bioinformatics pipelines
- Attend J.P. Morgan Healthcare Conference and BIO International for enterprise lead generation
- Develop pharma-specific features: ADMET property overlays, virtual screening integration, GxP audit trails

### Acquisition Channels
- Organic search: "predict protein structure online", "AlphaFold web server", "protein visualization tool"
- Academic word-of-mouth: researchers cite tools in methods sections, creating discovery loops
- Twitter/X and Bluesky presence in #structuralbiology and #bioinformatics communities
- Partnerships with university bioinformatics core facilities who recommend tools to PIs
- Freemium model: free academic tier creates pipeline to paid Researcher/Lab tiers as projects scale

---

## Success Metrics & KPIs

| Metric | Target (Month 6) | Target (Month 12) |
|--------|-------------------|---------------------|
| Registered users | 5,000 | 25,000 |
| Monthly predictions run | 15,000 | 80,000 |
| Paying customers (Researcher + Lab) | 100 | 500 |
| MRR | $12,000 | $65,000 |
| Structures in shared libraries | 50,000 | 250,000 |
| Average session duration | 15 min | 20 min |
| Free → Paid conversion rate | 3% | 6% |
| Monthly churn rate (paid) | <5% | <3% |

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | FoldSight Advantage |
|-----------|-----------|------------|-------------------|
| AlphaFold Server (DeepMind) | Gold-standard accuracy, free, massive pre-computed database | No visualization, no collaboration, no analysis tools, limited to prediction only | Full workflow: predict + visualize + analyze + collaborate in one platform |
| PyMOL (Schrodinger) | Industry-standard visualization, extensive scripting, plugin ecosystem | Desktop-only, $100+/seat, steep learning curve, no prediction, no collaboration | Browser-based, no install, integrated prediction, real-time collaboration |
| Schrodinger Suite | Comprehensive drug discovery platform, enterprise-grade | $10K-50K+/seat, complex licensing, requires computational chemistry expertise | 100x more accessible, intuitive for non-computational researchers, pay-per-use model |
| ColabFold | Free, AlphaFold-accessible via Colab, open-source | Requires notebook literacy, no integrated visualization, session timeouts, no collaboration | No coding required, integrated viewer, persistent storage, collaboration |
| SWISS-MODEL | Easy homology modeling, established reputation | Limited to homology modeling (no de novo prediction), basic visualization, no collaboration | State-of-art ML prediction, rich visualization, collaboration features |

---

## Risks & Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| DeepMind releases a comprehensive free web platform with visualization | High | Medium | Differentiate on collaboration, analysis depth, and enterprise features; maintain integration with AlphaFold models while adding unique analysis value |
| GPU compute costs make unit economics unsustainable at free/low tiers | High | Medium | Use ESMFold (cheaper, CPU-feasible) for free tier; implement job batching and spot instance pricing; cache predictions for common proteins |
| Academic users expect everything to be free and resist paid tiers | Medium | High | Maintain generous free tier for individual academics; monetize via Lab/Enterprise tiers targeting funded labs and industry; seek NIH SBIR grants |
| Prediction accuracy improvements make current models obsolete rapidly | Medium | High | Modular architecture allowing new model integration within days; position as "model-agnostic platform" rather than tied to specific prediction method |
| Pharmaceutical IP concerns prevent enterprise adoption | Medium | Medium | Offer on-premise deployment, client-side encryption, SOC 2 compliance, and data residency controls from day one for enterprise tier |

---

## MVP Scope (v1.0)

### In Scope
- ESMFold single-sequence structure prediction with GPU backend
- Interactive 3D viewer with cartoon, surface, and ball-and-stick representations
- Per-residue pLDDT confidence coloring and PAE matrix display
- PDB file import and UniProt sequence fetch
- Basic structural alignment (pairwise TM-align)
- PDB and mmCIF file export, PNG screenshot export
- User accounts with project organization

### Out of Scope (v1.1+)
- ColabFold/OpenFold integration and multimer prediction (v1.1)
- Real-time collaboration and commenting (v1.1)
- Binding site detection and druggability analysis (v1.2)
- Variant impact prediction and batch analysis (v1.2)
- Electrostatic surface computation (v1.2)
- PyMOL session export and publication figure builder (v1.3)
- API and Python SDK for programmatic access (v1.3)
- Enterprise features: SSO, on-premise, audit logging (v2.0)

### MVP Timeline: 8 weeks
- Week 1-2: FastAPI backend scaffolding, user auth (Auth0 + ORCID), PostgreSQL schema, S3 integration, ESMFold GPU worker deployment with Celery job queue
- Week 3-4: Mol* viewer integration in React frontend, structure upload/display pipeline, pLDDT coloring, PAE visualization, basic representation controls
- Week 5-6: Prediction submission UI, job queue management, UniProt/PDB sequence fetch, result storage and retrieval, project organization
- Week 7-8: Structural alignment (TM-align integration), PDB/mmCIF/PNG export, billing integration (Stripe), landing page, documentation, performance testing, and launch
