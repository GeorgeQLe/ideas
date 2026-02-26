# PrintGen — Persona Subspecs

> Parent spec: [`specs/32-printgen.md`](../32-printgen.md)

---

## Experience-Level Personas

### Student/Academic
**Modified UI/UX:**
- Step-by-step wizard interface: define design space, apply loads, set constraints, run optimization — each phase clearly labeled with educational annotations
- Inline explanations of topology optimization concepts (SIMP method, volume fraction, penalization) shown alongside parameter controls
- Simplified 3D viewer with pre-set camera angles and color-coded stress visualization legends

**Feature Gating:**
- Access to basic SIMP topology optimization with up to 500K element meshes
- Lattice generation limited to 3 standard types (gyroid, BCC, cubic); advanced conformal lattice tools hidden
- Single optimization run at a time; no parallel parameter sweeps

**Pricing Tier:** Free tier (academic email verification)

**Onboarding Flow:**
- Interactive tutorial: optimize a classic bracket under a single load case, visualize result, export STL
- Pre-loaded example problems (cantilever beam, MBB beam, L-bracket) with known optimal solutions for validation
- Prompt to join a course workspace if an instructor code is provided

**Key Workflows:**
- Set up a topology optimization problem with simple load cases for coursework assignments
- Compare optimization results across different volume fractions and constraint configurations
- Export optimized geometries as STL for 3D printing on university lab printers
- Validate optimization results against textbook analytical solutions
- Submit assignment results with optimization history and convergence plots

---

### Junior Engineer (0-3 years)
**Modified UI/UX:**
- Template-driven interface: select a part category (bracket, mount, enclosure), import design space from CAD, and follow guided optimization setup
- Contextual warnings when optimization parameters are likely to produce unprintable results (overhangs, thin walls, unsupported features)
- Side-by-side comparison view for evaluating multiple design candidates from a single optimization run

**Feature Gating:**
- Full SIMP and level-set optimization with meshes up to 5M elements
- All standard lattice types plus conformal lattice fills; FEA validation enabled
- Up to 5 concurrent optimization runs; slicer integration for PrusaSlicer and Cura

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Upload a CAD file (STEP/STL), define loads interactively by clicking surfaces, run first optimization in under 10 minutes
- Walkthrough of the design-to-print pipeline: optimize, validate with FEA, generate lattice infill, export to slicer
- Connect to team workspace and browse the shared part library for reusable design templates

**Key Workflows:**
- Import design spaces from SolidWorks or Fusion 360 and run topology optimization with manufacturing constraints
- Validate optimized parts with integrated FEA to confirm stress and displacement requirements
- Apply lattice infill structures to reduce weight while maintaining stiffness targets
- Export print-ready files directly to slicer software with optimized print orientation
- Document design rationale and optimization parameters for design review submissions

---

### Senior/Expert
**Modified UI/UX:**
- Advanced workspace with full parameter control: custom objective functions, multi-load case weighting, anisotropic material models, and overhang angle constraints
- Multi-objective Pareto front visualization for exploring trade-offs between mass, stiffness, and printability
- Scripting console for programmatic optimization setup, batch runs, and custom post-processing

**Feature Gating:**
- All features unlocked: multi-physics optimization, custom material models, API access, GPU cluster burst
- Unlimited concurrent optimizations and parameter sweeps; priority compute queue
- Admin tools for team design libraries, approval workflows, and audit trails

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Express import: bring in existing FEA models (Abaqus, Ansys) or optimization setups to bootstrap projects
- API walkthrough for integrating PrintGen into existing PLM and design automation workflows
- Team setup: configure shared material databases, print profiles, and quality standards

**Key Workflows:**
- Run multi-objective topology optimization across complex multi-load-case scenarios with manufacturing constraints
- Develop custom optimization formulations for novel materials or non-standard printing processes (DMLS, EBM, binder jetting)
- Architect automated design-optimize-validate-print pipelines integrated with CI/CD and PLM systems
- Mentor junior engineers by creating parameterized optimization templates with guardrails
- Lead design reviews using Pareto front analysis to justify weight-vs-performance trade-offs to stakeholders

---

## Role-Based Personas

### Designer/Creator
**Modified UI/UX:**
- Design space editor as primary view: 3D sculpting tools for defining keep-in/keep-out volumes alongside parametric geometry inputs
- Real-time preview of optimized topology updating as constraints are adjusted (interactive optimization)
- Aesthetics panel for controlling surface smoothness, organic form blending, and visual refinement of optimized shapes

**Feature Gating:**
- Full design space editing, all optimization methods, and post-optimization smoothing/refinement tools
- Lattice structure library with visual previews; custom lattice pattern designer enabled
- Export to multiple CAD formats (STEP, IGES, Parasolid) for downstream design integration

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Start from a design brief: define functional surfaces, load paths, and aesthetic constraints, then generate optimized forms
- Gallery of optimized designs across industries (aerospace, consumer products, medical) for inspiration
- Tutorial on refining optimized geometries for visual appeal while maintaining structural performance

**Key Workflows:**
- Generate organic, lightweight structural forms that satisfy engineering loads and aesthetic requirements
- Design custom lattice structures with graded density for visual and functional purposes
- Iterate rapidly between topology optimization variants and select the most compelling design candidate
- Refine mesh surfaces for smooth, production-quality appearance before manufacturing
- Export finalized designs to CAD packages for integration into larger assemblies

---

### Analyst/Simulator
**Modified UI/UX:**
- FEA validation panel as primary workspace with stress, displacement, and factor-of-safety contour plots
- Convergence monitoring dashboard showing optimization progress, objective function history, and constraint satisfaction
- Comparison matrix for side-by-side evaluation of multiple optimization results with quantitative metrics

**Feature Gating:**
- Full FEA solver access with nonlinear material models and contact analysis for validation
- Advanced post-processing: fatigue life estimation, buckling analysis, and thermal stress
- Batch validation runs across parameter variations; automated report generation

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Load an optimized design and walk through FEA validation setup: mesh refinement, boundary conditions, material assignment
- Configure convergence criteria and quality metrics for optimization monitoring
- Set up automated validation pipelines that run FEA checks on every optimization result

**Key Workflows:**
- Validate topology-optimized geometries against structural performance requirements using FEA
- Monitor optimization convergence and diagnose issues with constraint satisfaction or mesh quality
- Run parameter sensitivity studies to understand how load variations affect optimized topology
- Compare multiple design candidates using quantitative structural metrics (max stress, compliance, mass)
- Generate analysis reports with stress contours, displacement plots, and safety factor summaries

---

### Manufacturing/Process
**Modified UI/UX:**
- Printability assessment dashboard showing DfAM (Design for Additive Manufacturing) scores, overhang maps, and support volume estimates
- Printer profile manager with machine-specific build envelopes, material databases, and process parameter templates
- Build preparation view with orientation optimization, support generation preview, and nesting for multi-part builds

**Feature Gating:**
- Access to all DfAM analysis tools, slicer integrations, and build preparation features
- Printer fleet management and job scheduling tools enabled
- Optimization setup limited to approved templates; full design space editing restricted to designer role

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Configure printer fleet: add machines, upload build envelopes, set material profiles and process parameters
- Walk through build preparation for a sample part: orient, support, slice, and estimate cost/time
- Set up quality checkpoints and inspection criteria for post-print validation

**Key Workflows:**
- Assess printability of optimized designs and flag manufacturing concerns (unsupported overhangs, thin walls, trapped powder)
- Optimize print orientation to minimize support volume, build time, and surface roughness on critical faces
- Manage build plate nesting for multi-part production runs to maximize throughput
- Track material consumption, machine utilization, and per-part cost across the printer fleet
- Enforce DfAM guidelines by configuring manufacturing constraints that feed back into optimization

---

### Regulatory/Compliance
**Modified UI/UX:**
- Traceability dashboard showing full design history: optimization inputs, solver versions, material certifications, and validation results
- Standards compliance checklist panel (AS9100, ISO 13485, ASTM F2924 for AM Ti-6Al-4V) linked to design documentation
- Digital thread viewer connecting design intent through optimization through validation through manufacturing

**Feature Gating:**
- Read access to all design and analysis data; write access to compliance annotations and approval stamps
- Audit trail with immutable logs of all design changes, simulation parameters, and approval events
- Material certification database with traceability to powder lot numbers and test coupons

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Configure applicable standards and regulatory frameworks for the product line
- Walk through a sample digital thread from design requirements to printed part certification
- Set up approval workflows with required sign-off gates and document retention policies

**Key Workflows:**
- Review and approve optimization setups, ensuring loads, materials, and safety factors meet regulatory requirements
- Verify complete traceability from design requirements through optimization parameters to validated final geometry
- Maintain material certification records linked to specific builds and parts
- Generate compliance documentation packages for regulatory submissions (FAA, FDA, CE marking)
- Audit design history records for completeness and consistency before formal reviews

---

### Manager/Decision-maker
**Modified UI/UX:**
- Portfolio dashboard showing all active design projects with status, weight savings achieved, cost estimates, and timeline
- ROI calculator comparing PrintGen-optimized designs against conventional manufacturing (CNC, casting) for cost justification
- Resource utilization view: compute hours consumed, engineer workload, and printer fleet capacity

**Feature Gating:**
- Read-only access to project summaries, design comparisons, and cost analyses; no direct optimization tools
- Team management: license allocation, project creation, and resource budgeting
- Approval authority for design releases and production build authorizations

**Pricing Tier:** Enterprise tier (included in admin seat)

**Onboarding Flow:**
- Dashboard overview: active projects, weight savings metrics, cost-per-part trends, team utilization
- Set up project portfolio with target KPIs (mass reduction, cost targets, timeline milestones)
- Configure approval workflows and notification rules for design milestone reviews

**Key Workflows:**
- Track weight reduction and cost savings achieved across the design portfolio
- Compare generative design candidates based on performance, manufacturability, and cost trade-offs
- Allocate engineering and compute resources across projects based on business priority
- Review and approve designs for production release based on validated performance and compliance status
- Report program-level metrics (parts optimized, material saved, time-to-market improvement) to executive leadership
