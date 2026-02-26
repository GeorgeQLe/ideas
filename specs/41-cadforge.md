# CADForge — AI-Assisted Parametric 3D CAD in the Browser

## Executive Summary

CADForge is a cloud-native, browser-based parametric 3D CAD platform that challenges SolidWorks, Fusion 360, and CATIA by combining WebGPU-accelerated B-Rep kernel, AI-driven design intent recognition, and real-time multi-user collaboration. Engineers describe geometry with natural language, sketches, or traditional constraint-based modeling, and CADForge generates parametric feature trees that can be modified, version-controlled, and manufactured directly.

---

## Problem Statement

**The pain:**
- SolidWorks licenses cost $4,000-$8,000/seat/year with mandatory annual maintenance, and lock users into Windows-only installations on high-end workstations
- Fusion 360 went subscription-only ($545/year) and progressively restricts free-tier features, frustrating hobbyists and startups who built workflows around it
- CATIA/NX/Creo cost $15,000-$25,000/seat/year and require weeks of training, putting production-grade parametric CAD out of reach for small manufacturers and hardware startups
- Cross-platform collaboration is broken: mechanical engineers on Windows use SolidWorks, industrial designers on Mac use Rhino/Fusion, and the two exchange lossy STEP/IGES files manually
- Version control for CAD is an unsolved problem — teams use PDM/PLM systems costing $50K+ or resort to filename-based versioning ("bracket_v3_FINAL_rev2.sldprt")

**Current workarounds:**
- Onshape pioneered browser CAD but is now owned by PTC and priced at enterprise levels ($1,500/user/year), moving away from indie accessibility
- FreeCAD is open-source but has a notoriously unstable toponaming problem and fragmented UX that drives users away
- Many startups use Fusion 360's hobbyist tier but hit feature walls (limited export, no simulation, watermarked drawings)
- Some teams use OpenSCAD or CadQuery (code-based CAD) but lose visual immediacy and accessibility for non-programmers

**Market size:** The global CAD software market is valued at approximately $11.2 billion (2024) and projected to reach $18.5 billion by 2030. There are an estimated 6-8 million active CAD users worldwide across mechanical engineering, product design, architecture, and manufacturing. The SMB segment ($1-100M revenue companies) is severely underserved by current pricing models.

---

## Target Users

### Primary Personas

**1. Jake — Mechanical Engineer at a Hardware Startup**
- Works on a 4-person hardware team designing consumer electronics enclosures and mechanisms
- Currently uses Fusion 360 but hitting limitations on generative design, simulation, and multi-body workflows
- Needs: professional parametric CAD with built-in simulation preview, real-time collaboration with ID team, and direct export to injection molding vendors

**2. Maria — Industrial Design Freelancer**
- Designs products for 8-10 clients simultaneously across consumer goods, medical devices, and furniture
- Uses Rhino + Grasshopper on Mac but clients always need SolidWorks-compatible files
- Needs: cross-platform parametric CAD with organic surface modeling, universal format export, and client review links without requiring a license

**3. Professor Tanaka — Engineering Faculty**
- Teaches mechanical design courses to 200+ students per year
- Cannot justify SolidWorks campus license costs; currently uses FreeCAD with poor student outcomes
- Needs: browser-based CAD accessible from any student laptop, educational features (step-by-step tutorials, auto-grading of design assignments), and free academic tier

### Secondary Personas
- Manufacturing engineers preparing toolpaths for CNC and injection molding
- Hobbyists and makers designing parts for 3D printing
- Architects needing precise mechanical components for building systems

---

## Solution Overview

CADForge is a browser-based parametric 3D CAD platform that:
1. Runs a custom B-Rep geometric kernel compiled to WebAssembly, supporting NURBS surfaces, boolean operations, fillets, chamfers, shells, patterns, and lofts with full parametric history
2. Offers AI-assisted modeling: describe a part in natural language ("create a flanged bearing housing with 4 bolt holes on a 60mm PCD") and get an editable parametric feature tree
3. Renders via WebGPU with real-time section views, measurement overlays, and physically-based material preview
4. Provides Git-like version control with branch, merge, diff (visual 3D diff showing geometry changes), and pull request workflows for design reviews
5. Exports to STEP, IGES, STL, 3MF, DXF, and native SolidWorks/Fusion formats, with direct integrations to JLCPCB, Xometry, Protolabs, and Shapeways for manufacturing

---

## Core Features

### F1: Parametric Modeling Engine
- Full constraint-based 2D sketch environment with dimensions, geometric constraints (coincident, tangent, parallel, perpendicular, equal, symmetric), and auto-constraint detection
- 3D feature operations: extrude (blind, through-all, to-surface, mid-plane), revolve, sweep, loft, boundary surface, thicken surface
- Boolean operations: union, subtract, intersect with automatic face/edge tracking
- Fillet and chamfer with variable radius, face blend, and edge chain selection
- Shell, rib, draft, split, mirror, and linear/circular/curve-driven patterns
- Multi-body modeling with interference detection
- Assembly mode with standard mates (coincident, concentric, distance, angle, gear, cam)
- In-context editing of parts within assemblies
- Configuration/design table support for part families

### F2: AI-Assisted Design
- Natural language to parametric model: "create an M8 hex bolt 40mm long" generates a fully parametric bolt with correct thread geometry
- Sketch-to-CAD: upload a hand-drawn sketch or napkin photo and get a parametric 2D sketch with inferred constraints
- Design intent recognition: AI analyzes existing geometry and suggests likely next features (e.g., after creating a cylindrical boss, suggest adding fillets and bolt holes)
- Generative design: specify loads, constraints, material, and manufacturing method; AI generates topology-optimized shapes as parametric features
- "Fix my model" AI: diagnose and repair common modeling errors (zero-thickness geometry, self-intersecting surfaces, failed fillets)
- Parametric template library: thousands of standard components (fasteners, bearings, seals, springs, gears) with configurable parameters

### F3: Collaboration and Version Control
- Real-time multi-user editing with cursor presence and conflict resolution at the feature-tree level
- Git-style branching: create design variants as branches, merge approved changes, view 3D visual diffs
- Design review mode: share a link for read-only 3D viewing with commenting anchored to faces/edges/features
- Approval workflows: route design changes through review chains before merging to main branch
- Complete audit trail: every parameter change, feature addition, and review comment is logged with timestamps and user attribution
- CRDT-based data model enabling offline editing with automatic sync

### F4: Technical Drawing and Documentation
- Automated 2D drawing generation from 3D models with standard views (front, top, right, isometric)
- Section views, detail views, auxiliary views, and broken views
- GD&T annotation tools compliant with ASME Y14.5 and ISO GPS standards
- Automated dimension placement with smart spacing
- BOM (Bill of Materials) auto-generation with balloon annotations
- Title block templates customizable per company/client
- PDF and DXF export for manufacturing packages

### F5: Simulation Preview
- Linear static FEA with tetrahedral/hexahedral meshing for quick stress analysis
- Thermal conduction analysis
- Modal analysis (natural frequencies and mode shapes)
- Motion simulation for assemblies (kinematic and dynamic)
- Injection molding fill simulation preview (gate placement, weld lines, sink marks)
- Results displayed as color contour overlays on the 3D model with min/max markers

### F6: Manufacturing Integration
- Direct export to slicer formats (3MF, STL) with print-oriented analysis (overhang detection, support estimation)
- CNC toolpath generation for 2.5D milling operations with G-code export
- Automated DFM (Design for Manufacturability) checks for injection molding, sheet metal, and CNC
- One-click quote requests from integrated manufacturers (Xometry, Protolabs, JLCPCB CNC)
- Sheet metal mode: bend allowance, flat pattern, K-factor, and corner relief

---

## Technical Architecture

### System Diagram

```
Browser (WebGPU + WASM)
├── CAD Kernel (B-Rep engine compiled to WASM via OpenCascade/custom)
├── Constraint Solver (geometric constraint system in WASM)
├── Renderer (WebGPU-based with PBR materials)
├── Collaboration Layer (WebSocket + CRDT)
└── AI Features (API calls to backend)

Backend (Rust + Python)
├── API Gateway (Rust/Axum)
├── AI Services (Python: NLP-to-CAD, sketch recognition, generative design)
├── Geometry Processing Workers (Rust: STEP/IGES export, mesh generation, simulation)
├── Version Control Service (Git-like storage for parametric models)
├── Manufacturing API Integrations (Xometry, Protolabs, JLCPCB)
└── PostgreSQL + S3 (metadata + geometry blobs)
```

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | TypeScript, WebGPU, WebAssembly, React, Zustand |
| CAD Kernel | OpenCascade (OCCT) compiled to WASM / custom Rust B-Rep kernel |
| Constraint Solver | Custom 2D geometric constraint solver (Rust → WASM) |
| Backend API | Rust (Axum) for performance-critical paths, Python (FastAPI) for AI |
| AI/ML | GPT-4o for NL-to-CAD, custom vision model for sketch recognition, topology optimization via custom solver |
| Database | PostgreSQL (metadata), S3-compatible (geometry storage), Redis (sessions/cache) |
| Collaboration | WebSocket + Yjs (CRDT) for real-time sync |
| Simulation | CalculiX / custom FEA solver compiled to WASM for client-side preview, GPU-backed server for heavy jobs |
| Hosting | AWS (ECS for backend, CloudFront for WASM bundles, S3 for storage) |

---

## Monetization

### Free Tier (Maker)
- 3 active projects (public)
- Full parametric modeling
- STL/3MF export
- Community support
- CADForge watermark on drawings

### Pro — $29/month
- Unlimited private projects
- STEP/IGES/DXF export
- AI-assisted design (100 queries/month)
- Simulation preview
- Version history (unlimited)
- No watermark

### Team — $49/user/month
- Everything in Pro
- Real-time multi-user collaboration
- Branch/merge workflow
- Design review and approval
- Shared component libraries
- Admin controls and audit log

### Enterprise — Custom
- SSO/SAML
- On-premise deployment option
- Custom AI training on company standards
- PLM integration (Windchill, Teamcenter)
- SLA with dedicated support
- Unlimited simulation

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | CADForge Advantage |
|-----------|-----------|------------|-------------------|
| SolidWorks | Industry standard, vast ecosystem | $8K/yr, Windows-only, no collaboration | Browser-based, 10x cheaper, real-time collab |
| Fusion 360 | Good UX, cloud features | Restricting free tier, no true parametric branching | Open free tier, Git-like versioning, AI assist |
| Onshape | Browser-native, real-time collab | $1,500/yr, owned by PTC, limited simulation | 3x cheaper, AI features, better free tier |
| FreeCAD | Free, open-source | Toponaming bug, poor UX, no collaboration | Stable kernel, modern UX, cloud collab |
| CATIA/NX/Creo | Enterprise-grade, surface modeling | $15-25K/yr, steep learning curve | Accessible pricing, AI-assisted learning curve reduction |

---

## MVP Scope (v1.0)

### In Scope
- Parametric sketch + extrude/revolve/fillet/chamfer/boolean operations
- WebGPU 3D viewport with orbit/pan/zoom
- STEP and STL export
- Basic version history (linear, no branching)
- AI natural-language-to-sketch for simple geometry
- User accounts and project management

### Out of Scope (v1.1+)
- Full assembly mode with mates
- Real-time multi-user collaboration
- Simulation preview
- Technical drawing generation
- Manufacturing integrations
- Advanced surfacing (loft, boundary, sweep)

### MVP Timeline: 12-16 weeks
- Week 1-4: B-Rep kernel integration (OCCT→WASM), basic sketch constraints, extrude/revolve
- Week 5-8: WebGPU renderer, viewport interactions, feature tree UI, fillet/chamfer/boolean
- Week 9-12: STEP/STL export, user auth, project management, version history
- Week 13-16: AI sketch-to-CAD prototype, polish, beta launch
