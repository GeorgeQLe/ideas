# MolForge — Persona Subspecs

> Parent spec: [`specs/29-molforge.md`](../29-molforge.md)

---

## Experience-Level Personas

### Student/Academic

**Modified UI/UX:**
- Provide a simplified interface with a molecule sketcher (2D draw or SMILES input) and one-click docking against a pre-loaded target
- Show labeled 3D protein-ligand visualizations with annotated binding interactions (hydrogen bonds, hydrophobic contacts) in plain language
- Display ADMET property predictions as color-coded pass/fail badges with tooltips explaining each property

**Feature Gating:**
- Allow up to 10 docking runs per day against pre-loaded public targets (COVID protease, kinases, GPCRs)
- Hide generative molecular design, custom target preparation, and batch screening capabilities
- Provide read-only access to curated compound libraries for learning and exploration

**Pricing Tier:** Free tier

**Onboarding Flow:**
- First-run tutorial walks through drawing a simple molecule, docking it against a pre-loaded target, and interpreting the binding score
- Offer a set of 5 "Drug Discovery 101" exercises (target selection, lead identification, ADMET screening, SAR analysis)
- Provide a glossary popup for drug discovery terminology (IC50, LogP, TPSA, Lipinski rules) accessible from any screen

**Key Workflows:**
- Draw a molecule or enter a SMILES string and dock it against a pre-loaded protein target
- View the docked pose in 3D and identify key binding interactions with the target
- Check ADMET property predictions and evaluate drug-likeness using Lipinski's Rule of Five
- Compare docking scores of different molecules against the same target for a course project

---

### Junior Engineer (0-3 years)

**Modified UI/UX:**
- Show a project-based workflow guiding from target preparation through library screening to hit identification
- Provide a "Drug Discovery Assistant" sidebar suggesting next steps (e.g., "Your hit has poor solubility; try adding polar groups")
- Display interactive SAR tables linking molecular modifications to changes in binding score and ADMET properties

**Feature Gating:**
- Unlock custom target preparation (upload PDB, define binding site), library screening (up to 1,000 compounds), and generative design (basic mode)
- Expose ADMET prediction for all standard endpoints and molecule comparison tools
- Hide batch screening (>1,000), custom scoring functions, and enterprise collaboration features

**Pricing Tier:** Pro tier ($59/month)

**Onboarding Flow:**
- Walk through a complete hit identification workflow: prepare a target, screen a focused library, and identify top hits
- Demonstrate the generative design tool by generating analogs of a known drug and evaluating improvements
- Show how to build an SAR table comparing structural modifications and their impact on key properties

**Key Workflows:**
- Prepare a protein target from a PDB file and define the binding site for docking
- Screen a compound library (up to 1,000) against a target and rank hits by docking score and ADMET profile
- Use generative design to propose novel molecules optimized for binding affinity and drug-likeness
- Build SAR analyses comparing molecular modifications with property changes for lead optimization
- Share hit lists and analysis results with project collaborators for discussion and prioritization

---

### Senior/Expert

**Modified UI/UX:**
- Provide a multi-panel workspace with synchronized 2D sketcher, 3D protein-ligand viewer, property dashboard, and data tables
- Expose the full computational pipeline: target prep, library enumeration, docking, rescoring, ADMET, and MD-based binding free energy
- Support Python scripting for custom workflows, automated screening protocols, and integration with external cheminformatics tools

**Feature Gating:**
- Full access: unlimited docking, generative design with advanced objectives, batch screening (100K+ compounds), free energy perturbation, and API
- Expose custom scoring functions, pharmacophore modeling, and fragment-based drug design tools
- Enable team management, project-level access controls, and integration with ELN and patent databases

**Pricing Tier:** Enterprise tier ($199/month or site license)

**Onboarding Flow:**
- Skip tutorials; present API documentation and integration guides for connecting to existing cheminformatics pipelines
- Offer bulk import of compound libraries (SDF, CSV, SMILES files) and custom target structures
- Configure default docking parameters, scoring functions, and ADMET model preferences

**Key Workflows:**
- Run large-scale virtual screening campaigns (100K+ compounds) with cloud-distributed docking and ML rescoring
- Design novel drug candidates using generative AI with multi-objective optimization (potency, selectivity, ADMET, synthesizability)
- Perform free energy perturbation calculations to accurately rank lead compound series
- Build custom screening protocols combining pharmacophore filtering, docking, and ADMET scoring
- Manage a multi-target drug discovery portfolio with shared compound databases and cross-project SAR analysis

---

## Role-Based Personas

### Designer/Creator

**Modified UI/UX:**
- Emphasize the molecule editor with 2D/3D drawing tools, fragment-based assembly, and scaffold hopping interfaces
- Provide a generative design studio with target property sliders (potency, selectivity, solubility, permeability) and real-time candidate generation
- Show a "design sandbox" where users can sketch modifications and instantly see predicted impact on key properties

**Feature Gating:**
- Full access to molecule drawing, generative design, and molecular modification tools
- Expose fragment-based design, scaffold hopping, and bioisostere replacement suggestion engines
- Enable design idea sharing and compound nomination workflows within the project team

**Pricing Tier:** Pro tier ($59/month)

**Onboarding Flow:**
- Start with the molecule editor, demonstrating how to sketch a lead compound and explore structural modifications
- Walk through the generative design studio: set target properties, generate candidates, and filter by synthesizability
- Show the scaffold hopping tool by transforming a known drug scaffold into novel chemotypes

**Key Workflows:**
- Design novel molecules by drawing structures or assembling fragments in the 2D/3D editor
- Use generative AI to explore chemical space around a lead compound with property constraints
- Apply scaffold hopping and bioisostere replacements to overcome IP or ADMET issues
- Evaluate designed molecules with instant ADMET predictions and docking scores
- Nominate designed compounds for synthesis with predicted properties and design rationale documentation

---

### Analyst/Simulator

**Modified UI/UX:**
- Foreground the screening results dashboard with hit distribution plots, enrichment analysis, and statistical filtering tools
- Show a binding mode analysis panel with per-residue interaction fingerprints, hydrogen bond occupancy, and water network analysis
- Provide a comparative analysis view for evaluating compound series with property radar charts and activity cliffs detection

**Feature Gating:**
- Full access to docking analysis, ADMET prediction, statistical tools, and molecular dynamics setup
- Expose binding free energy calculations (MM-GBSA, FEP) and induced-fit docking for accurate pose prediction
- Enable data export to cheminformatics tools (RDKit, MOE, Maestro) and Jupyter notebooks

**Pricing Tier:** Pro tier ($59/month)

**Onboarding Flow:**
- Walk through analyzing a completed virtual screen: filter hits, cluster results, and identify top chemotypes
- Demonstrate binding mode analysis comparing docked poses of active vs. inactive compounds
- Show how to set up an FEP calculation for accurately ranking a congeneric series

**Key Workflows:**
- Analyze virtual screening results with statistical filtering, clustering, and enrichment metrics
- Perform detailed binding mode analysis with interaction fingerprints and energy decomposition
- Run molecular dynamics simulations to validate docking poses and assess binding stability
- Calculate relative binding free energies (FEP/TI) for accurate rank-ordering of lead series
- Generate quantitative SAR reports with activity cliffs, matched molecular pairs, and property trends

---

### Manufacturing/Process

**Modified UI/UX:**
- Show a synthetic accessibility dashboard scoring each compound for ease of synthesis and highlighting challenging functional groups
- Display a synthesis route suggestion panel powered by retrosynthetic analysis with estimated step counts and reagent costs
- Provide integration panels for connecting to CRO ordering systems and compound management databases

**Feature Gating:**
- Expose synthetic accessibility scoring, retrosynthetic route analysis, and precursor availability checking
- Enable integration with compound management systems and CRO ordering platforms
- Unlock production-scale enumeration tools for generating focused libraries around lead scaffolds

**Pricing Tier:** Enterprise tier ($199/month)

**Onboarding Flow:**
- Walk through evaluating a hit compound for synthetic feasibility and reviewing suggested synthesis routes
- Demonstrate the library enumeration tool for generating a focused analog series for medicinal chemistry
- Show integration with CRO ordering for submitting compounds directly for synthesis

**Key Workflows:**
- Evaluate synthetic accessibility of designed compounds and identify synthetic bottlenecks early in the design cycle
- Generate retrosynthetic routes with estimated costs and step counts for synthesis planning
- Enumerate focused compound libraries around lead scaffolds for SAR exploration
- Check precursor availability and pricing from chemical suppliers integrated into the platform
- Submit compound synthesis orders to CROs directly from the design interface with full specifications

---

### Regulatory/Compliance

**Modified UI/UX:**
- Add a patent landscape panel showing structural similarity searches against patent compound databases
- Display a compound provenance tracker documenting the computational origin, design rationale, and screening history of every molecule
- Provide a regulatory property panel showing predicted endpoints relevant to IND filing (hERG, Ames, hepatotoxicity)

**Feature Gating:**
- Expose patent similarity searching against major pharmaceutical patent databases
- Enable compound audit trails documenting the complete computational history from design to nomination
- Unlock predictive toxicology models (hERG, Ames, DILI, phototoxicity) for regulatory risk assessment

**Pricing Tier:** Enterprise tier ($199/month)

**Onboarding Flow:**
- Configure patent database access and set up alerts for structural similarity to compounds in the project portfolio
- Walk through running a regulatory safety assessment (hERG, Ames, hepatotoxicity predictions) on a lead compound
- Demonstrate the compound provenance trail from initial design through screening to nomination

**Key Workflows:**
- Screen designed compounds against patent databases to assess freedom-to-operate before investing in synthesis
- Predict regulatory-relevant toxicity endpoints (hERG, Ames, DILI) to de-risk compounds early in the pipeline
- Maintain a complete provenance record for every compound from computational design through experimental validation
- Generate safety assessment reports with predicted toxicity profiles for IND-enabling studies
- Document structure-activity relationships and design rationale for inclusion in patent applications

---

### Manager/Decision-maker

**Modified UI/UX:**
- Present a drug discovery portfolio dashboard showing all active programs, pipeline stage (target, hit, lead, candidate), and key metrics
- Show pipeline analytics: hit rates, lead optimization progress, cost per program, and timeline tracking
- Provide a compound nomination review interface with predicted properties, synthesis feasibility, and team recommendations

**Feature Gating:**
- Read-only access to all projects with commenting and approval capabilities on compound nominations
- Expose portfolio management tools: program prioritization, resource allocation, and milestone tracking
- Enable executive reporting with pipeline funnel visualization and competitive landscape analysis

**Pricing Tier:** Enterprise tier ($199/month)

**Onboarding Flow:**
- Set up the portfolio dashboard and configure program categories (oncology, immunology, infectious disease)
- Walk through the compound nomination review workflow and approval process
- Demonstrate the pipeline funnel visualization showing progression from targets to clinical candidates

**Key Workflows:**
- Monitor all drug discovery programs from a unified portfolio dashboard with stage-gate tracking
- Review and approve compound nominations for synthesis based on predicted properties and team recommendations
- Track pipeline progression rates, program costs, and resource utilization across the portfolio
- Prioritize programs based on scientific merit, commercial potential, and resource requirements
- Generate investor-ready pipeline reports summarizing program status, key findings, and strategic outlook
