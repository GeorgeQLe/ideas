# FoldSight — Persona Subspecs

> Parent spec: [`specs/22-foldsight.md`](../22-foldsight.md)

---

## Experience-Level Personas

### Student/Academic

**Modified UI/UX:**
- Display an annotated 3D viewer with labeled secondary structures (alpha helices, beta sheets) and color-coded confidence scores (pLDDT)
- Provide a "Learn" sidebar that explains structural biology concepts (Ramachandran plots, hydrophobic cores, disulfide bonds) contextually as the user interacts with structures
- Simplify the input to a single sequence paste box with a "Predict" button; hide batch processing and API access

**Feature Gating:**
- Allow single-sequence structure prediction and basic visualization (ribbon, surface, ball-and-stick representations)
- Hide batch prediction, custom model fine-tuning, and programmatic API endpoints
- Limit stored projects to 20 structures; disable advanced analysis tools (binding site detection, ddG estimation)

**Pricing Tier:** Free tier

**Onboarding Flow:**
- First-run experience predicts the structure of a well-known protein (e.g., GFP) and walks through interpreting confidence scores
- Offer a guided tour of the 3D viewer controls (rotate, zoom, select residues, color by property)
- Suggest a set of "try these" proteins from textbook examples to build familiarity

**Key Workflows:**
- Paste an amino acid sequence and view the predicted 3D structure with confidence coloring
- Compare a predicted structure side-by-side with an experimental PDB structure using the alignment tool
- Color the structure by different properties (hydrophobicity, conservation, B-factor) for coursework analysis
- Export a publication-quality figure (PNG/SVG) for a class report or thesis

---

### Junior Engineer (0-3 years)

**Modified UI/UX:**
- Show a guided analysis panel that suggests next steps after prediction (e.g., "Run structural alignment", "Check binding sites", "Map known variants")
- Display a project organizer with tagging and search to manage growing collections of predicted structures
- Provide preset visualization styles (publication, presentation, analysis) with one-click application

**Feature Gating:**
- Unlock batch prediction (up to 50 sequences), structural alignment, and variant impact scoring
- Expose binding site detection and basic docking preparation tools
- Hide enterprise features (team workspaces, SSO, custom model training) and limit API calls to 100/month

**Pricing Tier:** Pro tier ($39/month)

**Onboarding Flow:**
- Walk through a multi-step workflow: predict a structure, align it against a homolog, and identify conserved binding site residues
- Demonstrate batch prediction by uploading a FASTA file with 5 sequences
- Show how to annotate and share a structure with a colleague for collaborative review

**Key Workflows:**
- Predict structures for a set of protein variants and compare them to identify conformational changes
- Run structural alignments (TM-align) to assess similarity between predicted and experimental structures
- Identify and annotate binding sites with druggability scores for early-stage drug discovery projects
- Map missense variants onto the structure and estimate stability impact (ddG) for variant interpretation
- Share annotated structures with lab members via collaboration links

---

### Senior/Expert

**Modified UI/UX:**
- Provide a multi-panel workspace with synchronized 3D views, sequence alignment, and property plots
- Expose the full analysis toolkit with configurable parameters (alignment algorithms, scoring functions, visualization shaders)
- Support custom PyMOL-style command input for scripted visualization and batch analysis operations

**Feature Gating:**
- Full access to all features: unlimited batch prediction, API access, custom noise model training, molecular dynamics integration
- Expose advanced analyses: electrostatic surface calculations, normal mode analysis, protein-protein docking
- Enable team workspace management, custom pipeline creation, and integration with external tools (GROMACS, Rosetta)

**Pricing Tier:** Enterprise tier ($149/month or institutional license)

**Onboarding Flow:**
- Skip basics; present a "power user" setup wizard to configure preferred rendering styles, keyboard shortcuts, and default analysis parameters
- Offer API key generation and integration guides for connecting FoldSight to existing computational pipelines
- Demonstrate the custom pipeline builder for chaining prediction, analysis, and export steps

**Key Workflows:**
- Run large-scale batch predictions (hundreds of sequences) and programmatically filter results via the API
- Build custom analysis pipelines combining structure prediction, docking, and MD simulation preparation
- Perform protein-protein interaction analysis with interface residue identification and binding energy estimation
- Train project-specific ML models using experimental data to improve prediction accuracy for a protein family
- Manage a team workspace with shared structure libraries, standardized analysis protocols, and publication-ready templates

---

## Role-Based Personas

### Designer/Creator

**Modified UI/UX:**
- Emphasize the molecular design tools: residue mutation builder, loop remodeling interface, and de novo protein design input
- Provide a split view showing the designed sequence alongside real-time structural prediction updates
- Include a "design constraints" panel where users define target properties (stability, binding affinity, solubility) as design objectives

**Feature Gating:**
- Expose the protein design module: point mutation explorer, chimeric protein builder, and de novo sequence generation
- Enable integration with Rosetta and ProteinMPNN for computational protein design workflows
- Unlock custom scoring functions for evaluating designed variants

**Pricing Tier:** Pro tier ($39/month)

**Onboarding Flow:**
- Walk through designing a single-point mutation and observing its predicted structural and stability impact
- Demonstrate the de novo design interface with a simple use case (e.g., designing a miniprotein binder)
- Show how to export designed sequences for experimental validation with synthesis-ready formatting

**Key Workflows:**
- Explore the structural impact of point mutations to guide rational protein engineering
- Design chimeric proteins by combining domains from multiple parent structures
- Generate de novo protein sequences optimized for a target fold or binding interface
- Evaluate designed variants using stability prediction and compare against wild-type structure
- Export designed sequences with structural annotations for experimental collaborators

---

### Analyst/Simulator

**Modified UI/UX:**
- Foreground the quantitative analysis tools: RMSD calculations, contact maps, per-residue property plots, and statistical comparison dashboards
- Show a data table view alongside the 3D viewer for systematic comparison of structural metrics across a dataset
- Provide integration panels for importing experimental data (CD spectra, SAXS profiles, cross-linking MS) for structure validation

**Feature Gating:**
- Full access to structural comparison tools, statistical analysis, and property prediction
- Expose molecular dynamics setup and trajectory analysis (RMSD, RMSF, hydrogen bond analysis)
- Enable batch analysis with exportable reports and integration with Jupyter notebooks via API

**Pricing Tier:** Pro tier ($39/month)

**Onboarding Flow:**
- Demonstrate a comparative structural analysis workflow: predict two homologs, align them, and quantify differences
- Show how to validate a predicted structure against experimental SAXS or cross-linking data
- Walk through setting up a molecular dynamics simulation from a predicted structure

**Key Workflows:**
- Systematically compare predicted structures across a protein family to identify conserved and variable regions
- Validate predicted structures against experimental biophysical data (SAXS, HDX-MS, cross-linking)
- Set up and analyze molecular dynamics simulations to study protein flexibility and conformational changes
- Generate per-residue property profiles (B-factor, solvent accessibility, conservation) for large datasets
- Produce quantitative comparison reports for publication or internal review

---

### Manufacturing/Process

**Modified UI/UX:**
- Show a production pipeline dashboard with sample tracking, batch prediction status, and quality control metrics
- Provide a standardized report template for each predicted structure with confidence metrics and method documentation
- Display integration panels for connecting to LIMS systems and automated protein expression workflows

**Feature Gating:**
- Expose batch prediction with priority queuing and SLA-backed turnaround times
- Enable standardized output formats compatible with downstream expression and purification workflows
- Unlock audit trail and version-controlled analysis records for GMP documentation

**Pricing Tier:** Enterprise tier ($149/month)

**Onboarding Flow:**
- Configure the batch prediction pipeline with default parameters optimized for the user's protein class (antibodies, enzymes, etc.)
- Set up integration with the lab's LIMS or sample management system
- Demonstrate the QC dashboard for monitoring prediction confidence across a production batch

**Key Workflows:**
- Run production-scale batch predictions for antibody engineering or enzyme optimization campaigns
- Track prediction quality metrics across batches and flag low-confidence results for manual review
- Generate standardized structure reports for inclusion in CMC documentation
- Integrate prediction results with downstream protein expression and characterization workflows
- Maintain version-controlled records of all predictions for regulatory and IP documentation

---

### Regulatory/Compliance

**Modified UI/UX:**
- Add a compliance documentation panel that generates method descriptions suitable for regulatory filings (IND, BLA)
- Display prediction provenance information: model version, input parameters, confidence thresholds, and date stamps
- Provide a comparison view for tracking structural predictions across software updates to demonstrate consistency

**Feature Gating:**
- Expose the full audit trail with immutable logs of every prediction, analysis, and parameter change
- Enable validated reporting templates that meet FDA/EMA computational biology submission requirements
- Unlock SOC2-compliant data handling with encryption at rest and configurable data retention policies

**Pricing Tier:** Enterprise tier ($149/month)

**Onboarding Flow:**
- Set up the compliance workspace with appropriate data retention policies and access controls
- Walk through generating a regulatory-ready method description for a structure prediction workflow
- Demonstrate the audit trail and how to export it for inclusion in a regulatory submission

**Key Workflows:**
- Generate regulatory-compliant documentation of computational methods used in drug development filings
- Maintain an immutable audit trail of all predictions supporting a drug candidate's structural characterization
- Validate prediction reproducibility across software versions for regulatory consistency requirements
- Export method descriptions, confidence reports, and provenance records for IND/BLA submissions
- Configure data retention and access policies that satisfy FDA 21 CFR Part 11 electronic records requirements

---

### Manager/Decision-maker

**Modified UI/UX:**
- Present a portfolio dashboard showing all active projects, prediction throughput, team activity, and key structural findings
- Show summary cards for each target protein with status (predicted, validated, in design) and key metrics
- Provide cost tracking widgets for compute usage, seat licenses, and API call consumption

**Feature Gating:**
- Read-only access to all team projects with commenting and approval capabilities
- Expose project management tools: milestone tracking, resource allocation, and progress reporting
- Enable cost analytics and usage dashboards; disable direct prediction and analysis tools

**Pricing Tier:** Enterprise tier ($149/month)

**Onboarding Flow:**
- Set up the portfolio dashboard and configure project categories (drug targets, protein engineering, structural genomics)
- Walk through the team invitation and role assignment workflow
- Demonstrate the executive summary view that aggregates findings across projects

**Key Workflows:**
- Monitor progress across all structural biology projects from a unified dashboard
- Review and approve key structural findings before they advance to the next stage of drug development
- Track compute costs and team utilization for budget planning and resource allocation
- Generate executive summaries of structural findings for leadership and investor presentations
- Manage team access, project permissions, and data sharing policies across the organization
