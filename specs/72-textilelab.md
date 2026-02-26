# TextileLab — Textile and Fabric Engineering Simulation Platform

## Executive Summary

TextileLab is a cloud-based platform for designing and simulating textile structures — woven, knitted, and nonwoven fabrics — predicting mechanical properties, draping behavior, and manufacturing process parameters. It replaces TexGen (academic), WiseTex ($10K+), and ANSYS draping ($15K+ add-on) for technical textile applications in composites, automotive, protective equipment, and fashion.

---

## Problem Statement

**The pain:**
- Technical textiles (composites reinforcements, airbags, body armor, medical textiles) require simulation for mechanical prediction, but no affordable commercial tool exists
- WiseTex (KU Leuven) is academic and expensive ($10K+) for commercial use; TexGen is open-source but limited
- Fabric draping over complex 3D shapes (automotive interiors, composite molds) can only be predicted by simulation, but tools are part of $30K+ FEA packages
- Knitting machine programming (for 3D knitted components in Nike, Adidas products) is entirely manual with no simulation of the knitted structure
- Fashion industry fabric simulation (CLO3D, $50/month) handles visual draping but not engineering properties (tensile, shear, bending stiffness)

**Market size:** The technical textiles market is $200B+ globally. The textile simulation software segment is approximately $200 million. There are 100,000+ textile engineers worldwide.

---

## Core Features

### F1: Textile Structure Generation
- Woven: plain, twill, satin, and custom weave patterns with yarn geometry (cross-section, crimp)
- Knitted: weft knit (jersey, rib, interlock), warp knit (tricot, raschel), 3D knit structures
- Nonwoven: fiber orientation distribution, density, thickness
- Braided: 2D, 3D, triaxial braids with mandrel geometry
- Yarn models: monofilament, multifilament (with twist), staple fiber
- Unit cell generation with periodic boundary conditions
- Geometric parameter extraction: fiber volume fraction, yarn waviness, inter-yarn gaps

### F2: Mechanical Property Prediction
- Homogenization: compute effective stiffness tensor from unit cell FEA
- Tensile: stress-strain in warp and weft directions
- Shear: picture frame and bias extension simulation (critical for draping)
- Bending: Kawabata pure bending equivalent
- Compression: through-thickness response under compaction
- Permeability: predict in-plane and through-thickness permeability for composites (RTM/VARTM)
- Multi-scale: yarn → fabric → structural component

### F3: Draping Simulation
- Kinematic draping (fishnetting) for quick feasibility
- FEA-based draping: fabric over 3D mold with accurate shear behavior
- Defect prediction: shear angle limit → wrinkle formation zones
- Multi-ply draping: stacking multiple fabric layers with inter-ply friction
- Fiber angle deviation maps (critical for composite structural performance)
- Flat pattern generation for cutting

### F4: Manufacturing Process Simulation
- Weaving: yarn tension, beat-up force, shed geometry
- Knitting: loop formation, yarn consumption per course/wale
- Braiding: carrier path, braid angle vs. mandrel diameter, pull speed
- Compaction: vacuum bag pressure, resin flow through preform (link to CompForge resin module)

---

## Monetization

### Academic — $49/month per lab
- Unit cell generation (woven, knitted)
- Basic homogenization (stiffness prediction)
- Kinematic draping

### Pro — $149/month
- All textile structures (braided, nonwoven included)
- FEA-based draping
- Permeability prediction
- Manufacturing process simulation
- Report generation

### Enterprise — Custom
- Multi-scale structural analysis
- Custom textile structure development
- Integration with composites (CompForge) and FEA platforms

---

## MVP Scope (v1.0)

### In Scope
- Woven fabric unit cell generation (plain, twill, satin)
- 3D yarn geometry visualization
- CLT-based stiffness prediction for woven composites
- Kinematic draping on imported STL mold geometry
- Shear angle visualization
- Fiber volume fraction calculation

### Out of Scope (v1.1+)
- Knitted, braided, nonwoven structures
- FEA-based draping
- Permeability prediction
- Manufacturing process simulation

### MVP Timeline: 10-14 weeks
