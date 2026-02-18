# SynthBio — Cloud Synthetic Biology Design & Simulation Platform

## Executive Summary

SynthBio is a cloud-native synthetic biology platform that replaces fragmented desktop tools and ad-hoc scripting with an integrated, browser-based environment for designing genetic circuits, simulating gene expression dynamics, and engineering metabolic pathways. Researchers visually compose genetic circuits using SBOL standard notation, run Gillespie SSA simulations with GPU-accelerated Monte Carlo ensembles of 10,000+ stochastic trajectories, browse and compose from the iGEM Registry's 20,000+ standardized parts, perform flux balance analysis for metabolic engineering, and design CRISPR guide RNAs with off-target scoring — all in a single collaborative workspace that replaces Benchling, SnapGene, and Geneious at a fraction of the cost.

---

## Problem Statement

**The pain:**
- Synthetic biology workflows are fragmented across 5-10 disconnected tools — SnapGene for cloning, Benchling for sequence management, COPASI for kinetic modeling, custom Python scripts for stochastic simulation, and separate CRISPR design websites — with no data interoperability between them
- Stochastic simulation of genetic circuits (Gillespie SSA) requires writing custom code in Python or MATLAB, and running statistically meaningful ensemble sizes (10,000+ trajectories) takes hours on a single CPU, forcing researchers to either wait overnight or settle for underpowered sample sizes
- The iGEM Registry contains 20,000+ characterized genetic parts, but browsing, filtering, and composing them into circuits requires manual copy-paste between a clunky web interface and desktop sequence editors with no programmatic integration
- Metabolic pathway design and flux balance analysis require specialized tools like COBRA Toolbox in MATLAB ($2,000+ license) or command-line cobrapy, excluding the majority of wet-lab biologists who lack programming expertise
- CRISPR guide RNA design tools are scattered across free web services (CRISPOR, Benchling) with inconsistent off-target algorithms, no batch processing, and no integration with the rest of the design workflow

**Current workarounds:**
- Labs cobble together Benchling (free academic tier with feature limits) plus SnapGene ($hundreds/seat/year) plus manual Python scripting for any computational work, losing hours to format conversions and context switching
- Graduate students write one-off Gillespie SSA scripts in Python/MATLAB that are slow, unvalidated, and abandoned when the student graduates — each new lab member rewrites the same simulation code from scratch
- Researchers use iBioSim (open-source, academic) for SBOL modeling but struggle with its Java desktop interface, limited documentation, and lack of GPU acceleration for large ensembles
- Teams rely on expensive enterprise Benchling contracts ($15K-50K+/year) that bundle sequence management with features most academic labs never use, while still lacking stochastic simulation and metabolic modeling

**Market size:** The global synthetic biology market is valued at approximately $16.7 billion (2024) and projected to reach $65 billion by 2030 (CAGR 25.4%). The bioinformatics and life science software segment represents approximately $4-6 billion, with synthetic biology design tools specifically estimated at $800M-1.2B. There are an estimated 8,000+ synthetic biology research labs worldwide, 2,000+ iGEM teams competing annually, and a rapidly growing industrial biotech sector spanning pharmaceuticals, agriculture, biofuels, and biomaterials.

---

## Target Users

### Primary Personas

**1. Dr. Maya Chen — Postdoc in a Synthetic Biology Lab**
- Designs and characterizes genetic logic gates and toggle switches in E. coli for a Nature-track publication; needs to demonstrate robust circuit behavior across parameter uncertainty
- Currently uses SnapGene for cloning design, writes custom Python Gillespie scripts that take 4+ hours for 10,000-trajectory ensembles, and manually searches iGEM for promoter/RBS parts
- Needs: integrated visual circuit design with SBOL notation, fast stochastic simulation with GPU-accelerated ensembles, easy access to characterized iGEM parts with kinetic parameters

**2. Jake Morrison — iGEM Team Captain (Undergraduate)**
- Leads a 12-person undergraduate iGEM team designing a biosensor for heavy metal detection; the team has strong wet-lab skills but limited computational modeling experience
- Struggles to computationally validate circuit designs before building them in the lab, wasting weeks on constructs that fail due to predictable issues like promoter strength mismatches or metabolic burden
- Needs: intuitive visual circuit builder that an undergraduate can learn in an afternoon, pre-built simulation templates, iGEM parts library with drag-and-drop composition, and CRISPR guide design for genome integration

**3. Dr. Priya Sharma — Principal Investigator at a Metabolic Engineering Startup**
- Leads a 20-person team engineering yeast strains for sustainable chemical production; runs 50+ CRISPR edits per month and needs to optimize metabolic flux through engineered pathways
- Currently pays $45K/year for enterprise Benchling plus $8K/year for SnapGene team licenses, and still requires a dedicated computational biologist to run COBRA/cobrapy FBA analyses
- Needs: unified platform combining sequence design, FBA-based metabolic pathway optimization, CRISPR guide design with off-target scoring, and team collaboration — at a lower total cost than current tool stack

### Secondary Personas
- Bioinformatics core facility managers who support multiple synthetic biology groups with shared computational infrastructure and standardized design tools
- High school and community college instructors teaching introductory synthetic biology who need free, accessible design tools for classroom use
- Regulatory affairs scientists at biotech companies who need to review and document genetic circuit designs for FDA/EPA submissions with full version-controlled audit trails

---

## Solution Overview

SynthBio works as follows:
1. Researchers open the SBOL Visual Circuit Editor and compose genetic circuits by dragging standardized parts (promoters, RBSs, CDSs, terminators) from the integrated iGEM Registry onto a canvas, connecting them into transcription units and regulatory networks using SBOL-compliant notation
2. The platform automatically generates a stochastic kinetic model from the circuit topology, mapping SBOL parts to rate parameters from characterized part databases, which users can tune — then launches Gillespie SSA simulations with Rust-accelerated compute kernels that run 10,000+ Monte Carlo trajectories in parallel on GPU
3. For metabolic engineering workflows, researchers construct metabolic pathway maps with drag-and-drop enzyme placement, import genome-scale models (SBML), and run flux balance analysis to identify optimal gene knockouts, overexpressions, and heterologous pathway insertions for maximizing target metabolite production
4. CRISPR guide RNA design is integrated directly into the workflow — select a genomic target, and SynthBio generates candidate guides ranked by on-target efficiency (Rule Set 2, DeepCRISPR) and off-target specificity (Cas-OFFinder scoring against the host genome), with one-click export to ordering format
5. All designs, simulations, and analyses are version-controlled in collaborative lab notebooks with real-time multi-user editing, SBOL/GenBank/FASTA export, and direct integration with DNA synthesis providers for ordering engineered constructs

---

## Core Features

### F1: SBOL Visual Circuit Editor
- Drag-and-drop canvas for composing genetic circuits using SBOL Visual standard glyphs (promoters, RBS, CDS, terminators, operators, insulators, origins of replication)
- Hierarchical design: compose individual transcription units into multi-gene circuits, regulatory networks, and combinatorial libraries with modular abstraction layers
- Automatic SBOL data model generation from visual layout — exports valid SBOL3 XML/JSON-LD that interoperates with other SBOL-compliant tools
- Regulatory interaction overlay: draw activation, repression, and degradation arrows between circuit components to define the regulatory network topology
- Parameterized parts: each component carries annotated kinetic parameters (transcription rate, translation rate, degradation rate, Hill coefficient, K_d) sourced from published characterization data
- Real-time design rule checking: flag common errors such as missing RBS before CDS, incorrect part orientation, incompatible assembly standard overhangs, and promoter-terminator read-through conflicts
- Keyboard shortcuts and smart alignment tools for rapid circuit composition with automatic spacing and routing of regulatory interaction arrows

### F2: Gillespie SSA Genetic Circuit Simulation
- Automatic conversion of SBOL circuit topology and kinetic parameters into a chemical master equation model with species (mRNA, proteins, small molecules) and reactions (transcription, translation, degradation, binding, unbinding)
- Exact Gillespie Stochastic Simulation Algorithm (SSA) implementation in Rust for single-trajectory runs with sub-millisecond per-reaction performance
- Tau-leaping approximate method for accelerated simulation of circuits with fast reactions (10-100x speedup with controllable accuracy)
- Interactive time-course plots showing molecular species counts over time with real-time rendering as simulation progresses
- Parameter sweep mode: define ranges for any kinetic parameter (e.g., promoter strength from 0.1 to 10 nM/min) and run systematic sweeps to map circuit behavior across parameter space
- Sensitivity analysis: automatic computation of parameter sensitivity coefficients to identify which parameters most strongly influence circuit output
- Steady-state detection with automatic Kolmogorov-Smirnov test for distribution convergence, reporting mean, variance, coefficient of variation, and bimodality metrics

### F3: GPU Monte Carlo Ensemble Simulation
- Launch 10,000 to 1,000,000 independent Gillespie SSA trajectories in parallel on GPU using custom Rust/CUDA compute kernels
- Statistical summary dashboard: probability distributions of species counts at user-defined time points, time-to-threshold distributions, switching probability for bistable circuits, and noise metrics (Fano factor, coefficient of variation)
- Bifurcation analysis: sweep a control parameter (e.g., inducer concentration) across its range while running GPU ensembles at each point to map stochastic bifurcation diagrams
- Histogram and kernel density estimation (KDE) visualization of ensemble distributions at any simulation time point with interactive scrubbing
- Confidence interval computation: report 95% CIs for mean expression levels, switch-on probabilities, and oscillation periods with automatic convergence checking
- Export raw trajectory data (HDF5, CSV) and summary statistics for downstream analysis in Python/R
- Cost-efficient spot instance scheduling: large ensemble jobs automatically run on preemptible GPU instances with checkpointing and automatic retry

### F4: iGEM Parts Library Integration
- Full mirror of the iGEM Registry of Standard Biological Parts with 20,000+ characterized parts, synchronized weekly with the official registry
- Advanced search and filtering: search by part type (promoter, RBS, CDS, terminator, reporter, regulatory), organism compatibility, characterization quality rating, BioBrick/Type IIS assembly compatibility, and quantitative performance metrics
- Part characterization data display: show measured transfer functions, expression levels, crosstalk, and reliability data from published iGEM team wikis and peer-reviewed literature
- One-click insertion: drag any iGEM part directly onto the SBOL circuit canvas with automatic sequence, annotation, and kinetic parameter import
- Community ratings and reviews: user-contributed experience reports on part performance across different chassis organisms and growth conditions
- Composite part builder: assemble multi-part transcription units from iGEM parts with automatic scar sequence generation for BioBrick (RFC 10) and Type IIS (Golden Gate, MoClo) assembly standards

### F5: Metabolic Pathway Design and FBA
- Visual metabolic pathway editor: drag-and-drop enzymes onto a pathway canvas, connect substrates and products with stoichiometric reaction arrows, and define flux constraints
- Import genome-scale metabolic models (SBML format) for E. coli (iML1515), S. cerevisiae (Yeast8), and 20+ other model organisms from BiGG Models database
- Flux Balance Analysis (FBA) solver: linear programming-based optimization of metabolic flux distribution to maximize biomass growth, target metabolite production, or custom objective functions
- Gene knockout prediction: automatic identification of single and double gene knockouts that redirect metabolic flux toward target compound production (OptKnock algorithm)
- Flux Variability Analysis (FVA): compute the feasible range of flux values for each reaction to identify bottleneck reactions and essential genes
- Pathway visualization overlay: color-code metabolic reactions by flux magnitude on an interactive pathway map, with tooltips showing flux values, enzyme identifiers, and gene associations
- Heterologous pathway insertion: model the addition of non-native enzyme steps and predict their impact on host metabolism using integrated FBA

### F6: CRISPR Guide RNA Design
- Input a target gene or genomic region and automatically generate ranked candidate guide RNAs (20-mer spacers) for SpCas9, SaCas9, Cas12a/Cpf1, and base editors
- On-target efficiency scoring using multiple algorithms: Rule Set 2 (Doench 2016), DeepCRISPR, and CHOPCHOP with consensus ranking
- Off-target analysis: Cas-OFFinder alignment against the host genome (E. coli, S. cerevisiae, human, mouse, and 30+ other organisms) with mismatch tolerance up to 4 bp, reporting off-target sites with genomic context and gene annotations
- Visual off-target map: genome-wide visualization of off-target binding sites on a chromosomal ideogram with click-through to sequence alignment detail
- Batch guide design: submit multiple targets simultaneously for high-throughput CRISPR library construction (CRISPRi/CRISPRa screens)
- Primer design for guide cloning: automatic generation of oligonucleotide sequences with appropriate overhangs for Golden Gate or Gibson Assembly into common Cas9 expression vectors
- Integration with circuit editor: selected guides are linked back to the SBOL design for genome integration site planning

### F7: Plasmid Map Editor
- Interactive circular and linear plasmid map visualization with automatic feature annotation from SnapGene-compatible feature databases
- Drag-and-drop feature insertion: add genes, promoters, origins of replication, antibiotic resistance markers, and restriction sites onto the plasmid backbone
- Restriction enzyme analysis: display cut sites for 600+ commercially available restriction enzymes with fragment size prediction and virtual gel electrophoresis visualization
- Gibson Assembly and Golden Gate Assembly design: automatic overlap/overhang sequence generation with Tm-matched primer design for seamless cloning
- Plasmid backbone library: 200+ common expression vectors (pET, pBAD, pUC, pRS, pYES series) with pre-annotated maps
- Reading frame indicator and codon usage display for protein-coding regions with automatic start/stop codon detection

### F8: Codon Optimization
- Optimize DNA sequences for expression in target host organisms (E. coli, S. cerevisiae, P. pastoris, CHO cells, human cell lines, and 50+ others) using codon adaptation index (CAI) maximization
- Multi-objective optimization balancing codon usage frequency, mRNA secondary structure stability (minimum free energy), GC content, repeat avoidance, and restriction site elimination
- Codon harmonization mode: preserve translational pausing signals from the native organism for proper protein folding of complex multi-domain proteins
- Side-by-side comparison of original vs. optimized sequences with highlighted changes, CAI scores, and predicted expression improvement estimates
- Batch optimization: submit multiple sequences simultaneously with organism-specific optimization for multi-gene pathway construction
- Export optimized sequences directly to the plasmid map editor or DNA synthesis ordering workflow

### F9: Sequence Annotation and Analysis
- Automatic feature detection: identify promoters, RBS (Shine-Dalgarno sequences), start/stop codons, terminators, restriction sites, and common regulatory elements using curated pattern databases
- Open reading frame (ORF) finder with six-frame translation display and automatic protein BLAST against UniProt/SwissProt for functional annotation
- Multiple sequence alignment (MSA) using MUSCLE/Clustal Omega with interactive alignment viewer, conservation scoring, and phylogenetic tree construction (neighbor-joining)
- Primer design tool: input a target region and generate PCR primer pairs with Tm matching, GC content optimization, hairpin/dimer checking, and specificity verification against the host genome
- Sequence statistics dashboard: nucleotide composition, GC content, melting temperature, molecular weight, extinction coefficient, and isoelectric point (for encoded proteins)
- GenBank/FASTA/SBOL import and export with bidirectional format conversion preserving all annotations

### F10: Protocol Designer
- Structured protocol builder for documenting experimental procedures linked to specific genetic designs: transformation, miniprep, restriction digest, ligation, PCR, gel electrophoresis, flow cytometry assays
- Reagent calculator: input target volumes and concentrations, and the system computes required stock volumes, dilution ratios, and master mix recipes for common molecular biology protocols
- Protocol templates: pre-built step-by-step protocols for BioBrick assembly (3A assembly), Golden Gate assembly, Gibson Assembly, CRISPR RNP transfection, and colony PCR screening
- Timer integration: embedded countdown timers for incubation steps, with browser notifications for step completion
- Protocol versioning: track modifications to protocols over time with diff view showing changed steps, reagent quantities, or incubation times
- Export to PDF for bench-side printing with QR codes linking back to the digital protocol and associated designs

### F11: Collaboration and Lab Notebooks
- Real-time multi-user editing of circuit designs, plasmid maps, and protocols using CRDT-based conflict-free synchronization
- Electronic lab notebook (ELN): timestamped, append-only experiment entries linked to specific designs, simulations, and protocols with rich text, images, and file attachments
- Commenting system: attach threaded discussions to specific parts, simulation results, or protocol steps for asynchronous design review
- Project-level access controls: owner, editor, viewer roles with organization-level team management and audit logging
- Activity feed: chronological timeline of all design changes, simulation runs, and notebook entries across the project
- Integration with institutional identity providers (SAML/OIDC) for university and corporate single sign-on

### F12: Data Export and Integration
- Export genetic designs in SBOL3, GenBank, FASTA, GFF3, and SnapGene (.dna) formats with full annotation preservation
- Simulation data export: time-course trajectories (CSV, HDF5), parameter sweep results (JSON), ensemble statistics (CSV), and publication-ready figures (SVG, PNG at 300 DPI)
- Direct DNA synthesis ordering: export optimized sequences to IDT, Twist Bioscience, and GenScript with automatic codon-optimized formatting and pricing estimates
- REST API for programmatic access to all platform features: create designs, launch simulations, query parts library, and retrieve results from custom scripts and Jupyter notebooks
- Benchling import: migrate existing Benchling projects (sequences, annotations, folder structure) into SynthBio via Benchling API integration
- SBML model export for interoperability with COPASI, iBioSim, and other systems biology modeling tools

---

## Technical Architecture

### System Diagram

```
┌────────────────────────────────────────────────────────────────┐
│                       Browser Client                            │
│  ┌────────────┐ ┌───────────┐ ┌───────────┐ ┌──────────────┐  │
│  │ SBOL Visual│ │ Plasmid   │ │ Pathway   │ │ Simulation   │  │
│  │ Circuit    │ │ Map       │ │ Editor    │ │ Dashboard    │  │
│  │ Editor     │ │ Editor    │ │ (D3.js)   │ │ (Plotly)     │  │
│  │ (Canvas)   │ │ (SVG)     │ │           │ │              │  │
│  └─────┬──────┘ └─────┬─────┘ └─────┬─────┘ └──────┬───────┘  │
│        └───────────────┼─────────────┼──────────────┘          │
│                        ▼                                        │
│              CRDT Document Model (Yjs)                          │
│                        │                                        │
└────────────────────────┼───────────────────────────────────────┘
                         │ WebSocket + REST
                         ▼
               ┌──────────────────────┐
               │   API Gateway        │
               │   (Python / FastAPI) │
               └───┬──────────┬───────┘
                   │          │
        ┌──────────┘          └────────────┐
        ▼                                  ▼
┌────────────────┐              ┌──────────────────────┐
│  Collaboration │              │   Compute Engine     │
│  Server (Yjs)  │              │                      │
│  WebSocket Hub │              │  ┌────────────────┐  │
└───────┬────────┘              │  │ Gillespie SSA  │  │
        │                       │  │ (Rust/CUDA)    │  │
        ▼                       │  └────────────────┘  │
┌────────────────┐              │  ┌────────────────┐  │
│  PostgreSQL    │              │  │ FBA Solver     │  │
│  (Users,       │              │  │ (GLPK/CPLEX)  │  │
│   Projects,    │◄─────────────│  └────────────────┘  │
│   Designs,     │              │  ┌────────────────┐  │
│   Parts Cache) │              │  │ CRISPR Engine  │  │
└───────┬────────┘              │  │ (Cas-OFFinder) │  │
        │                       │  └────────────────┘  │
        ▼                       │  ┌────────────────┐  │
┌────────────────┐              │  │ Codon Optimize │  │
│  Redis         │              │  │ (Rust)         │  │
│  (Sessions,    │              │  └────────────────┘  │
│   Job Queue,   │              └──────────────────────┘
│   Cache)       │                         │
└────────────────┘                         ▼
                                ┌──────────────────────┐
                                │  Object Storage      │
                                │  (S3/R2)             │
                                │  Sequences, Models,  │
                                │  Simulation Results, │
                                │  Genome Indices      │
                                └──────────────────────┘
```

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend Framework | React 19 + TypeScript, Vite build system |
| SBOL Circuit Canvas | Custom HTML5 Canvas renderer with SBOL Visual glyph library and spatial indexing (R-tree) |
| Plasmid Map Renderer | SVG-based circular/linear map renderer with D3.js for interactive annotations |
| Pathway Editor | D3.js force-directed graph layout with metabolic reaction node/edge rendering |
| Simulation Charts | Plotly.js for interactive time-course, histogram, and heatmap visualizations |
| Collaboration | Yjs CRDT library with WebSocket provider for real-time multi-user editing |
| Backend API | Python 3.12, FastAPI, Pydantic v2, SQLAlchemy 2.0 |
| SSA Compute Engine | Rust compiled to native + CUDA kernels for GPU-accelerated Gillespie ensemble simulation |
| FBA Solver | Python cobrapy with GLPK (free) and CPLEX (enterprise) linear programming backends |
| CRISPR Engine | Cas-OFFinder (C++/OpenCL) for off-target search, custom Rust scoring pipeline for on-target efficiency |
| Codon Optimization | Rust-based multi-objective optimizer using genetic algorithm with organism-specific codon tables |
| Database | PostgreSQL 16 with JSONB for flexible part metadata and simulation parameter storage |
| Cache / Job Queue | Redis 7 for session management, Celery task queue for async simulation jobs |
| Object Storage | Cloudflare R2 for sequence files, simulation results (HDF5), genome indices, and SBML models |
| Auth | JWT-based authentication with OAuth providers (Google, GitHub, ORCID) and SAML for institutional SSO |
| Search | Meilisearch for full-text search across iGEM parts, sequences, and project contents |
| GPU Compute | Lambda Cloud / RunPod GPU instances (A100/H100) for Monte Carlo ensembles, auto-scaled via Kubernetes |
| Hosting | Fly.io (API servers), Cloudflare Workers (edge caching), Kubernetes on AWS for compute workers |
| Monitoring | Grafana + Prometheus for infrastructure metrics, Sentry for error tracking, PostHog for product analytics |

### Data Model

```
User
├── id (uuid), email, name, orcid_id (nullable), avatar_url
├── auth_provider, auth_provider_id
├── plan (free/academic/researcher/lab/enterprise)
├── institution, department
└── created_at, updated_at

Organization
├── id (uuid), name, slug
├── owner_id → User
├── plan, seat_count, billing_email
└── settings (default_organism, preferred_assembly_standard)

Project
├── id (uuid), org_id → Organization
├── name, description, visibility (private/org/public)
├── organism (e_coli/s_cerevisiae/human/custom)
├── created_by → User
├── forked_from → Project (nullable)
└── created_at, updated_at

GeneticCircuit
├── id (uuid), project_id → Project
├── name, description, version
├── sbol_data (JSONB: components, interactions, constraints)
├── canvas_layout (JSONB: positions, dimensions, routing)
├── kinetic_model (JSONB: species, reactions, parameters)
└── created_by → User, created_at, updated_at

Part
├── id (uuid), source (igem/custom/community)
├── igem_id (nullable, e.g., "BBa_R0040")
├── name, description, part_type (promoter/rbs/cds/terminator/reporter/regulatory)
├── sequence (text), length_bp
├── organism_compatibility[] (e_coli/yeast/mammalian)
├── assembly_standard (biobrick/golden_gate/moclo/gibson)
├── kinetic_params (JSONB: transcription_rate, translation_rate, degradation_rate, Kd, hill_coeff)
├── characterization_data (JSONB: transfer_functions, expression_levels)
├── datasheet_url, igem_wiki_url
└── quality_rating (float), review_count, verified (boolean)

Simulation
├── id (uuid), circuit_id → GeneticCircuit, project_id → Project
├── simulation_type (ssa_single/ssa_ensemble/tau_leaping/parameter_sweep/bifurcation)
├── parameters (JSONB: duration, dt, n_trajectories, swept_param, param_range)
├── status (queued/running/completed/failed/cancelled)
├── gpu_instance_type, compute_cost_usd
├── results_url (S3 path to HDF5/JSON results)
├── summary_stats (JSONB: means, variances, CIs, distributions)
├── started_at, completed_at
└── launched_by → User, created_at

MetabolicModel
├── id (uuid), project_id → Project
├── name, organism, source (bigg/custom/uploaded)
├── sbml_url (S3 path to SBML file)
├── reaction_count, metabolite_count, gene_count
├── pathway_layout (JSONB: node positions, edge routing)
├── fba_objective (biomass/target_metabolite/custom)
└── created_by → User, created_at, updated_at

FBAResult
├── id (uuid), model_id → MetabolicModel
├── objective_value (float), solver_status
├── flux_distribution (JSONB: reaction_id → flux_value)
├── knockout_predictions[] (gene_id, growth_rate, target_flux)
├── fva_ranges (JSONB: reaction_id → {min, max})
└── created_at

CRISPRGuideSet
├── id (uuid), project_id → Project
├── target_gene, target_region (chrom, start, end)
├── organism, cas_variant (SpCas9/SaCas9/Cas12a)
├── guides[] (JSONB: spacer_seq, pam, on_target_score, off_target_count, off_target_sites[])
├── status (designing/completed/failed)
└── created_by → User, created_at

PlasmidMap
├── id (uuid), project_id → Project
├── name, sequence (text), length_bp, topology (circular/linear)
├── features[] (JSONB: name, type, start, end, strand, color)
├── backbone_source (custom/pet/pbad/puc/prs)
├── restriction_sites[] (JSONB: enzyme, position, overhang)
└── created_by → User, created_at, updated_at

LabNotebookEntry
├── id (uuid), project_id → Project, author_id → User
├── title, body (rich text), entry_type (experiment/observation/analysis)
├── linked_designs[] → GeneticCircuit/PlasmidMap
├── linked_simulations[] → Simulation, attachments[] (S3 keys)
└── created_at (immutable timestamp for audit)

Comment
├── id (uuid), project_id → Project, author_id → User
├── body (text), resolved (boolean)
├── anchor_type (part/circuit/simulation/protocol/notebook), anchor_id (uuid)
└── created_at, resolved_at
```

### API Design

```
Auth & Orgs:
POST   /api/auth/register                    # Email/password registration
POST   /api/auth/login                       # Login (email/password)
POST   /api/auth/oauth/:provider             # OAuth flow (Google, GitHub, ORCID)
POST   /api/auth/refresh                     # Refresh JWT token
GET    /api/orgs                              # List user's organizations
POST   /api/orgs                              # Create organization
POST   /api/orgs/:id/members                  # Invite member

Projects:
GET    /api/projects                          # List projects (with filters)
POST   /api/projects                          # Create new project
GET    /api/projects/:id                      # Get project metadata
PATCH  /api/projects/:id                      # Update project settings
DELETE /api/projects/:id                      # Delete project
POST   /api/projects/:id/fork                 # Fork a project

Genetic Circuits:
GET    /api/projects/:id/circuits             # List circuits in project
POST   /api/projects/:id/circuits             # Create new circuit
GET    /api/circuits/:id                      # Get circuit (SBOL + layout + model)
PATCH  /api/circuits/:id                      # Update circuit design
POST   /api/circuits/:id/validate             # Run design rule check
POST   /api/circuits/:id/export/:format       # Export (SBOL3, GenBank, FASTA)

Parts Library:
GET    /api/parts                             # Search parts (type, organism, keyword)
GET    /api/parts/:id                         # Get part details + kinetic params
POST   /api/parts                             # Create custom part
POST   /api/parts/:id/review                  # Submit community review

Simulations:
POST   /api/circuits/:id/simulate             # Launch SSA simulation
POST   /api/circuits/:id/ensemble             # Launch GPU Monte Carlo ensemble
POST   /api/circuits/:id/sweep                # Launch parameter sweep
GET    /api/simulations/:id                   # Get simulation status + results
GET    /api/simulations/:id/trajectories      # Stream trajectory data (paginated)
GET    /api/simulations/:id/statistics        # Get ensemble summary statistics
POST   /api/simulations/:id/cancel            # Cancel running simulation

Metabolic Models & FBA:
POST   /api/projects/:id/models               # Create/upload model (SBML)
GET    /api/models/:id                        # Get model details
POST   /api/models/:id/fba                    # Run FBA optimization
POST   /api/models/:id/fva                    # Run Flux Variability Analysis
POST   /api/models/:id/knockout               # Run gene knockout prediction
GET    /api/bigg/models                       # List available BiGG models

CRISPR:
POST   /api/projects/:id/crispr/design        # Design guide RNAs for target
GET    /api/crispr/:id                        # Get guide design results
GET    /api/crispr/:id/off-targets/:guide     # Get off-target details for guide
POST   /api/crispr/:id/export                 # Export guides for ordering

Plasmid Maps:
POST   /api/projects/:id/plasmids             # Create plasmid map
GET    /api/plasmids/:id                      # Get plasmid details
PATCH  /api/plasmids/:id                      # Update plasmid map
POST   /api/plasmids/:id/restriction-analysis # Run restriction enzyme analysis
POST   /api/plasmids/:id/export/:format       # Export (GenBank/FASTA/SnapGene)

Codon Optimization:
POST   /api/codon/optimize                    # Optimize sequence for target organism

Collaboration & Notebooks:
WS     /ws/projects/:id                       # WebSocket for real-time editing
GET    /api/projects/:id/notebook             # List notebook entries
POST   /api/projects/:id/notebook             # Create notebook entry
GET    /api/projects/:id/comments             # List comments
POST   /api/projects/:id/comments             # Add comment
POST   /api/projects/:id/share                # Generate share link

DNA Synthesis:
POST   /api/synthesis/quote                   # Get pricing from IDT/Twist/GenScript
POST   /api/synthesis/order                   # Submit synthesis order
```

---

## UI/UX — Key Screens

### 1. SBOL Circuit Designer
- Center canvas with infinite pan/zoom displaying genetic circuit in SBOL Visual notation with color-coded glyphs (blue promoters, green RBS, orange CDS, red terminators)
- Left panel: parts palette organized by type with search, filters for organism compatibility and assembly standard, and iGEM Registry browser with part characterization previews
- Right panel: selected component properties — name, sequence preview, kinetic parameters (editable), characterization data charts, and links to iGEM wiki pages
- Top toolbar: circuit validation button (DRC), simulate button (launches SSA), export menu (SBOL3, GenBank, FASTA), and undo/redo stack
- Bottom panel: regulatory network summary showing interaction graph (activations in green, repressions in red) auto-generated from the circuit topology
- Contextual tooltips on hover showing part function, measured expression levels, and reliability scores

### 2. Simulation Dashboard
- Time-course plot area (Plotly.js) showing species concentrations over time with toggleable traces for each molecular species (mRNA, protein, inducer, reporter)
- Ensemble overlay mode: render 100 representative trajectories as semi-transparent lines with the mean trajectory in bold and shaded 95% confidence interval bands
- Right panel: simulation configuration — algorithm (exact SSA, tau-leaping), duration, number of trajectories, initial conditions, and parameter overrides with real-time cost estimate
- Bottom panel: histogram/distribution view showing probability distribution of species counts at a user-selected time point with interactive time scrubber
- Parameter sensitivity heatmap: grid showing how output metrics (mean expression, noise, switch probability) change across two swept parameters
- Job status bar showing GPU utilization, elapsed time, trajectories completed, and estimated time remaining

### 3. Metabolic Pathway Editor
- Central canvas with force-directed metabolic network graph: metabolites as circular nodes, reactions as rectangular enzyme nodes, with directed edges showing substrate/product stoichiometry
- Left panel: reaction/enzyme palette with search across BiGG database, organism-specific filtering, and EC number lookup
- Right panel: selected reaction details — stoichiometry, flux bounds, associated genes, and current FBA flux value with color-coded magnitude indicator
- Top toolbar: run FBA, run FVA, predict knockouts, import SBML model, and toggle between pathway subsystem views (glycolysis, TCA, pentose phosphate, target pathway)
- Flux overlay: reactions colored on a blue-white-red gradient indicating flux magnitude after FBA solution, with arrow thickness proportional to flux
- Knockout prediction results panel: ranked list of gene knockouts with predicted growth rate and target metabolite production rate

### 4. CRISPR Guide Designer
- Target input area: paste gene sequence, enter gene name for automatic lookup, or click a region on the plasmid map/genome browser to define the target
- Results table: ranked candidate guides showing spacer sequence (20-mer), PAM, on-target score (0-100), off-target count (0-4 mismatches), GC content, and homopolymer warnings
- Detail view for selected guide: genome-wide off-target map on chromosomal ideogram, off-target alignment table with mismatch positions highlighted, and genomic context (nearby genes, regulatory elements)
- Batch mode panel: upload a list of target genes/regions (CSV) and run parallel guide design with downloadable results in oligo-ordering format
- Settings panel: select Cas variant (SpCas9, SaCas9, Cas12a), scoring algorithm preferences, mismatch tolerance, and organism genome for off-target search

### 5. Plasmid Map and Sequence Viewer
- Circular plasmid map (SVG) with color-coded feature arrows, draggable feature boundaries, and zoom-to-region on click
- Linear sequence view below the map showing nucleotide sequence with feature underlays, restriction enzyme cut site markers, and six-frame translation for ORFs
- Restriction analysis sidebar: searchable enzyme list with cut site count, compatible enzymes highlighted for cloning strategy, and virtual gel electrophoresis preview
- Cloning designer panel: select assembly method (Gibson, Golden Gate, restriction/ligation), define insert and backbone, and auto-generate primer sequences with Tm-matched annealing regions
- Feature annotation toolbar: add custom features with type, name, color, and directionality; auto-detect common features (CMV promoter, T7 promoter, His-tag, GFP) from sequence

### 6. Lab Notebook and Collaboration View
- Chronological feed of lab notebook entries with rich text, embedded images, linked circuit designs (clickable thumbnails), and simulation result summaries
- New entry composer: WYSIWYG editor with templates for common entry types (cloning experiment, transformation results, plate reader data, simulation analysis)
- Threaded comment panel: discussion threads attached to specific designs, simulation results, or notebook entries with @mention notifications
- Project activity sidebar: real-time feed showing team member actions (edited circuit, ran simulation, added notebook entry) with timestamps
- Version history browser: timeline of all design revisions with visual diff showing added/removed/modified parts and parameter changes

---

## Monetization

### Free / Academic Tier
- 3 projects (public only)
- Basic SBOL circuit editor with up to 20 parts per circuit
- Single-trajectory Gillespie SSA (CPU, up to 10,000 time steps)
- iGEM parts library browsing and search (read-only)
- Plasmid map viewer (import GenBank, view-only editing)
- 5 CRISPR guide designs per month
- Community support via Discord
- SynthBio watermark on exported figures

### Researcher — $99/month
- Unlimited private projects
- Full SBOL circuit editor with unlimited complexity
- GPU Monte Carlo ensembles up to 10,000 trajectories per run, 50 ensemble jobs per month
- Parameter sweep and sensitivity analysis
- Full iGEM parts library with kinetic parameter access
- Metabolic pathway editor with FBA (up to 500-reaction models)
- Unlimited CRISPR guide designs with off-target analysis
- Codon optimization (20 sequences per month)
- GenBank/SBOL3/FASTA export without watermark
- Plasmid map editor with restriction analysis
- Email support with 48-hour response time

### Lab — $349/month (up to 10 seats, $35/additional seat)
- Everything in Researcher
- GPU Monte Carlo ensembles up to 1,000,000 trajectories, 200 jobs per month
- Genome-scale FBA models (unlimited reaction count, CPLEX solver)
- Gene knockout prediction (OptKnock) and Flux Variability Analysis
- Real-time multi-user collaboration on all design types
- Electronic lab notebooks with audit trail
- Protocol designer with templates
- DNA synthesis ordering integration (IDT, Twist, GenScript)
- Batch CRISPR guide design (up to 500 targets)
- Unlimited codon optimization
- Organization-level team management and role-based access
- Priority support with 24-hour response time

### Enterprise — Custom
- Unlimited seats, projects, and compute
- Dedicated GPU cluster for Monte Carlo simulations with guaranteed capacity
- SAML/SSO integration with institutional identity providers
- Custom part libraries with proprietary sequence protection
- On-premise deployment option for IP-sensitive biotech companies
- API access with higher rate limits for automated workflows
- Dedicated customer success manager and onboarding
- SLA with 99.9% uptime guarantee
- Regulatory compliance support (21 CFR Part 11 electronic signatures)
- Custom integrations with LIMS, ELN, and internal databases

---

## Go-to-Market Strategy

### Phase 1: iGEM and Academic Seeding (Month 1-3)
- Launch free tier targeting iGEM teams (2,000+ teams globally) with dedicated iGEM onboarding flow and tutorial: "Design and simulate your iGEM project in SynthBio"
- Partner with 10 university synthetic biology courses to adopt SynthBio as the course design tool, offering free Lab tier access for enrolled students and instructors
- Publish benchmark comparisons on bioRxiv: SynthBio GPU ensembles vs. manual Python Gillespie scripts (100x speedup), integrated workflow vs. Benchling + SnapGene + custom code (5x time savings)
- Create YouTube tutorial series: "Synthetic Biology Design from Zero" covering circuit design, simulation, iGEM parts, CRISPR, and metabolic engineering
- Sponsor iGEM Giant Jamboree and SynBioBeta conference with live demos and design competitions

### Phase 2: Research Lab Adoption (Month 3-6)
- Launch Researcher tier with GPU Monte Carlo ensembles and metabolic pathway FBA as key differentiators from free tools
- Publish case studies with early-adopter labs showing: "Designed and validated a genetic toggle switch in 2 days instead of 2 weeks" and "Identified optimal metabolic knockouts without writing a single line of code"
- SEO content targeting: "Benchling alternative for synthetic biology", "genetic circuit simulation tool", "SBOL editor online", "free CRISPR guide design"
- Integration partnerships with DNA synthesis providers (IDT, Twist Bioscience) for seamless ordering workflow and co-marketing
- Present at SEED (Synthetic Biology: Engineering, Evolution & Design) and ACS Synthetic Biology journal virtual symposia

### Phase 3: Enterprise and Industry (Month 6-12)
- Launch Lab and Enterprise tiers with collaboration features, electronic lab notebooks, and compliance capabilities
- Target industrial biotech companies (Ginkgo Bioworks, Zymergen/Ginkgo, Amyris, Codexis) replacing fragmented Benchling + COPASI + custom tool stacks
- Develop 21 CFR Part 11 compliance features (electronic signatures, audit trails) for pharmaceutical and regulated biotech customers
- Partner with biofoundries (Edinburgh Genome Foundry, DBTL facilities) for automated design-build-test-learn cycle integration
- Launch SynthBio Marketplace for community-contributed circuit designs, characterized part libraries, and simulation templates

### Acquisition Channels
- Organic search targeting: "synthetic biology design tool", "genetic circuit simulator", "SBOL editor", "iGEM design software", "metabolic pathway FBA tool", "CRISPR guide design"
- iGEM community: annual sponsorship, team adoption tracking, iGEM-specific tutorials and templates
- Academic paper citations: every SynthBio simulation generates a citable methods paragraph, driving organic adoption as labs reproduce published circuits
- Referral program: invite a colleague and both receive 1 month of Researcher tier free
- Twitter/X and Bluesky engagement in the #synbio community, showcasing user-designed circuits and simulation results

---

## Success Metrics & KPIs

| Metric | Target (Month 6) | Target (Month 12) |
|--------|-------------------|---------------------|
| Registered users | 8,000 | 35,000 |
| Monthly active projects | 2,500 | 12,000 |
| Paying customers (Researcher + Lab) | 150 | 700 |
| MRR | $20,000 | $95,000 |
| GPU ensemble simulations per month | 3,000 | 20,000 |
| iGEM parts used in designs per month | 15,000 | 80,000 |
| Free-to-Paid conversion rate | 4% | 7% |
| Monthly churn rate | < 6% | < 4% |

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | SynthBio Advantage |
|-----------|-----------|------------|---------------------|
| Benchling | Industry standard for sequence management, strong collaboration, large enterprise customer base, $100M+ funding | Extremely expensive for academic labs ($15K-50K+/year), no stochastic simulation, no FBA/metabolic modeling, no GPU compute, CRISPR design is basic | Integrated SSA simulation with GPU ensembles, FBA metabolic modeling, full SBOL visual editor, 10-50x cheaper for academic users |
| SnapGene | Excellent plasmid visualization, intuitive cloning design, widely adopted in molecular biology labs | Desktop-only ($hundreds/seat/year), no simulation capabilities whatsoever, no collaboration, no SBOL support, no metabolic modeling | Cloud-native with real-time collaboration, stochastic simulation, SBOL standard compliance, iGEM integration, metabolic pathway design |
| Geneious | Comprehensive sequence analysis, strong alignment and phylogenetics, primer design | Expensive desktop software ($hundreds-thousands/seat), no genetic circuit design, no stochastic simulation, no SBOL, no metabolic modeling | Purpose-built for synthetic biology workflows, SBOL visual circuit design, Gillespie SSA, GPU Monte Carlo, FBA, CRISPR — all integrated |
| iBioSim | Open-source SBOL modeling and simulation, academic pedigree, SBML/SBOL standards compliant | Java desktop application with dated UI, no GPU acceleration, very limited user base, no collaboration, steep learning curve, unmaintained documentation | Modern cloud UI, GPU-accelerated ensembles (100x faster), integrated iGEM parts, CRISPR design, metabolic modeling, real-time collaboration |
| COPASI | Powerful ODE/SSA biochemical kinetics simulator, free and open-source, well-established in systems biology | Desktop-only, no genetic circuit design tools, no SBOL support, no CRISPR or metabolic FBA, no parts library, not designed for synthetic biology workflows | Integrated design-simulate workflow, SBOL visual editor, iGEM parts, GPU Monte Carlo ensembles, CRISPR guide design, metabolic FBA — all in the browser |

---

## Risks & Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| GPU Monte Carlo simulation costs exceed revenue from Researcher/Lab tiers, making large ensembles unprofitable | High | Medium | Implement per-job cost tracking and dynamic pricing; use spot/preemptible GPU instances for 60-70% savings; enforce trajectory caps per tier; cache common circuit simulation results across users |
| Benchling launches competing simulation features, leveraging their existing 250K+ user base | High | Medium | Move fast on GPU ensemble and FBA features that require deep computational infrastructure Benchling lacks; build strong iGEM community adoption as a moat; maintain open SBOL/SBML standards for data portability so users can use SynthBio alongside Benchling |
| SBOL Visual standard evolves (SBOL4) requiring significant canvas editor refactoring | Medium | Low | Build the SBOL rendering engine with abstraction layers separating visual glyphs from data model; participate in SBOL community standards development to anticipate changes; maintain backward compatibility with SBOL3 exports |
| Kinetic parameter databases for iGEM parts are incomplete or unreliable, producing inaccurate simulations that erode user trust | High | High | Clearly display parameter confidence levels and data provenance for every part; allow users to override parameters with their own measured values; partner with labs publishing characterization data; implement ensemble sensitivity analysis showing how parameter uncertainty affects predictions |
| Academic users never convert to paid tiers, relying entirely on free tier and generating GPU costs without revenue | Medium | High | Design free tier with meaningful but bounded utility (CPU-only SSA, small circuits); ensure Researcher tier has clear "aha moments" (10,000-trajectory GPU ensemble in 30 seconds vs. overnight on CPU); target lab PIs with grant-funded budgets rather than individual grad students; offer institutional site licenses for universities |

---

## MVP Scope (v1.0)

### In Scope
- SBOL Visual Circuit Editor with drag-and-drop parts (promoter, RBS, CDS, terminator), regulatory interaction drawing, and SBOL3/GenBank export
- iGEM Parts Library browser with search, filtering, and one-click insertion into circuit designs (top 5,000 most-used parts with kinetic parameters)
- Gillespie SSA simulation: single-trajectory and small ensemble (up to 100 trajectories, CPU-based) with interactive time-course plots
- Basic plasmid map viewer with GenBank import and feature annotation display
- CRISPR guide RNA design for SpCas9 with on-target scoring and basic off-target search against E. coli and S. cerevisiae genomes
- User accounts, project CRUD, basic version history (save/restore snapshots)
- Sequence import/export in GenBank and FASTA formats

### Out of Scope (v1.1+)
- GPU Monte Carlo ensemble simulation with 10,000+ trajectories (v1.1 — highest priority post-MVP)
- Metabolic pathway editor and FBA solver (v1.1)
- Real-time multi-user collaboration via CRDT (v1.2)
- Codon optimization engine (v1.2)
- Protocol designer and electronic lab notebook (v1.2)
- DNA synthesis ordering integration with IDT/Twist/GenScript (v1.3)
- Advanced CRISPR features: SaCas9/Cas12a support, batch guide design, base editor design (v1.3)
- Benchling project import and migration tools (v1.3)
- Enterprise features: SAML/SSO, on-premise deployment, 21 CFR Part 11 compliance (v2.0)

### MVP Timeline: 10 weeks
- Week 1-2: Core data model and API scaffolding (FastAPI + PostgreSQL), user authentication (JWT + OAuth), project CRUD, S3 integration for file storage, iGEM parts database import and indexing pipeline
- Week 3-4: SBOL Visual Circuit Editor — custom Canvas renderer with SBOL glyphs, drag-and-drop part placement, regulatory interaction drawing, automatic kinetic model generation from circuit topology, SBOL3 and GenBank export
- Week 5-6: Gillespie SSA simulation engine in Rust (native binary), single-trajectory and small-ensemble execution, interactive Plotly.js time-course visualization, parameter configuration UI, basic parameter sweep capability
- Week 7-8: iGEM parts library browser (Meilisearch-powered search and filtering), CRISPR guide RNA designer (SpCas9, Rule Set 2 scoring, Cas-OFFinder for E. coli/yeast), basic plasmid map viewer with GenBank import
- Week 9-10: Project dashboard, version history (snapshot save/restore), sequence import/export, billing integration (Stripe), end-to-end testing with real genetic circuit designs, documentation, and beta launch to 20 synthetic biology labs
