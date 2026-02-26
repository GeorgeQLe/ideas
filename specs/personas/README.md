# Persona-Based Subspecs Framework

This directory contains persona-based subspecification addenda for all 80 deep-tech engineering products (specs 21-100). Each subspec file references its parent spec and describes how the product experience adapts for different user personas.

---

## Persona Dimensions

### Experience Level

| Level | Description | Key Characteristics |
|-------|-------------|-------------------|
| **Student/Academic** | University students, researchers, professors | Tutorials, guided workflows, coursework integration, free/discounted pricing, citation support |
| **Junior Engineer (0-3 years)** | Early-career professionals learning the domain | Template-heavy, AI-assisted guard rails, common mistake prevention, mentor review workflows |
| **Senior/Expert** | 5+ years experience, domain specialists | Full flexibility, scripting/API access, custom model integration, advanced features unlocked, batch processing |

### Role-Based

| Role | Description | Key Workflows |
|------|-------------|---------------|
| **Designer/Creator** | Geometry creation, concept exploration | Rapid iteration, variant generation, visual comparison, parametric exploration |
| **Analyst/Simulator** | Physics simulation, validation | Mesh setup, solver configuration, parameter studies, convergence monitoring, result post-processing |
| **Manufacturing/Process** | Production planning, toolpath generation | Process parameters, quality control, DFM checks, production scheduling, yield optimization |
| **Regulatory/Compliance** | Standards compliance, audit trails | Documentation generation, certification packages, standards databases, traceability matrices |
| **Manager/Decision-maker** | Project oversight, cost estimation | Dashboards, trade studies, executive summaries, resource allocation, cost-benefit analysis |

---

## Subspec Structure

Each persona subspec file covers:

1. **Modified UI/UX** — How the interface adapts (simplified wizard vs. expert console)
2. **Feature Gating** — Which features are exposed/hidden for that persona
3. **Pricing Tier Alignment** — Which pricing tier best serves that persona
4. **Onboarding Flow** — Persona-specific first-run experience
5. **Key Workflows** — 3-5 primary use cases unique to that role

---

## File Naming Convention

```
specs/personas/{number}-{product-name}-personas.md
```

Example: `specs/personas/21-circuitmind-personas.md`

Each file contains all 8 persona variants (3 experience levels + 5 role-based) for that product.

---

## Cross-Reference Matrix

Not every role applies equally to every product. The relevance matrix below uses:
- **P** = Primary (deeply relevant, dedicated workflows)
- **S** = Secondary (useful but not core)
- **—** = Not applicable

| Product Category | Designer | Analyst | Manufacturing | Regulatory | Manager |
|-----------------|----------|---------|---------------|------------|---------|
| CAD/Geometry tools | **P** | S | **P** | S | S |
| Simulation/FEA tools | S | **P** | S | S | S |
| Process engineering | S | **P** | **P** | **P** | S |
| Electronics/EDA | **P** | **P** | **P** | S | S |
| Biomedical/Pharma | S | **P** | S | **P** | S |
| Civil/Infrastructure | **P** | **P** | S | **P** | **P** |
| Environmental | S | **P** | S | **P** | **P** |
| Energy systems | S | **P** | S | **P** | **P** |
