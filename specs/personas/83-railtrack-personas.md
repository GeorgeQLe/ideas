# RailTrack — Persona Subspecs

> Parent spec: [`specs/83-railtrack.md`](../83-railtrack.md)

---

## Experience-Level Personas

### Student/Academic
**Modified UI/UX:**
- Simplified track alignment editor with preset templates (single-track, double-track, yard layout) and guided curve/gradient entry
- Interactive cross-section viewer showing rail, sleeper, ballast, and subgrade layers with labeled dimensions
- Animated train movement simulation on designed alignment with real-time speed and cant displays

**Feature Gating:**
- Basic horizontal/vertical alignment design with standard curve types (circular, clothoid)
- Timetable simulation limited to 5 trains on a single corridor
- Signal and interlocking design modules locked; simplified block-section view available

**Pricing Tier:** Free tier

**Onboarding Flow:**
- Tutorial: design a 10 km single-track branch line with stations, run a basic timetable with 2 trains
- Pre-loaded example networks (suburban commuter, freight corridor) with annotated design decisions
- Glossary sidebar linking terms (cant deficiency, transition length, block section) to railway engineering textbooks

**Key Workflows:**
- Design a horizontal and vertical alignment for a given terrain corridor
- Calculate required cant and transition lengths for target line speed
- Run a simple timetable simulation and identify headway constraints
- Produce a longitudinal section drawing with gradient and curvature diagrams

---

### Junior Engineer (0-3 years)
**Modified UI/UX:**
- Project-organized workspace with tabs for Alignment | Track Structure | Signaling | Timetable
- Integrated design standard selector (EN 13803, AREMA, country-specific) auto-populating limits and checks
- Real-time design rule validation with warning/error flags as alignment is edited

**Feature Gating:**
- Full alignment design with spirals, compound curves, and vertical curve optimization
- Track structure calculator (rail selection, sleeper spacing, ballast depth) enabled
- Signaling layout limited to automatic block; interlocking design gated

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Import terrain data (GIS/LiDAR), define alignment corridor constraints, auto-generate candidate alignments
- Design first track structure cross-section using standard selector and material database
- Run a 20-train timetable simulation and review capacity utilization report

**Key Workflows:**
- Design and optimize alignment geometry to meet speed, comfort, and earthwork constraints
- Select track structure components (rail profile, sleeper type, fastening system) per applicable standard
- Perform capacity analysis for a corridor and identify bottleneck sections
- Generate alignment data sheets and cross-section drawings for detailed design submittals
- Evaluate earthwork quantities (cut/fill) and optimize vertical alignment for cost

---

### Senior/Expert
**Modified UI/UX:**
- Multi-corridor project manager with network-wide timetable and capacity dashboards
- Scripting console (Python/Lua) for custom design rules, automated alignment optimization, and batch analysis
- Version-controlled design workspace with change tracking and multi-user merge capabilities

**Feature Gating:**
- All features unlocked: full interlocking design, ETCS/PTC overlay, stochastic timetable simulation
- Network-wide capacity optimization with rolling stock fleet modeling
- API access for integration with BIM (IFC Rail), GIS platforms, and signaling design tools

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Import existing network model from legacy tools (OpenTrack, RailSys export) via migration wizard
- Configure organization design standards library and approval workflows
- API setup for BIM/GIS integration and CI/CD pipeline for automated design checks

**Key Workflows:**
- Network-level timetable optimization balancing capacity, journey time, and energy consumption
- Full signaling and interlocking design with formal safety verification (interlocking tables, route conflicts)
- Stochastic simulation of timetable robustness under delay propagation scenarios
- Multi-criteria alignment optimization (cost, environmental impact, speed, constructability)
- Lifecycle cost modeling integrating track degradation, maintenance planning, and renewal scheduling

---

## Role-Based Personas

### Designer/Creator
**Modified UI/UX:**
- Visual alignment design on map/terrain with real-time 3D fly-through preview
- Drag-handle geometry editing with instant constraint violation feedback
- Station layout canvas with platform, turnout, and crossover placement tools

**Feature Gating:**
- Full geometric design tools: horizontal/vertical alignment, cross-sections, station layouts
- Timetable and capacity tools available in simplified mode for design validation
- Export to CAD (DWG/DXF), BIM (IFC Rail), and presentation formats

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Import terrain/corridor data and begin alignment design on map canvas
- Place stations and design track layout with turnout templates
- Generate 3D visualization for design review with stakeholders

**Key Workflows:**
- Design alignment geometry with terrain-aware optimization (minimize earthwork, avoid constraints)
- Lay out station track plans with platform lengths, turnout positions, and siding arrangements
- Create 3D visualizations and fly-through animations for public consultation
- Iterate on design variants and compare alternatives on a multi-criteria matrix

---

### Analyst/Simulator
**Modified UI/UX:**
- Simulation control panel with timetable editor, delay injection tools, and run-time visualization
- Statistical output dashboards: punctuality distributions, capacity utilization heatmaps, energy consumption profiles
- Batch simulation manager for Monte Carlo timetable robustness studies

**Feature Gating:**
- Full timetable simulation engine with stochastic delay models and conflict detection
- Network capacity analysis tools (UIC 406 compression method) enabled
- Energy consumption modeling with regenerative braking and eco-driving optimization

**Pricing Tier:** Professional or Enterprise tier

**Onboarding Flow:**
- Import infrastructure and timetable; run a baseline simulation and review punctuality metrics
- Inject random delays and re-simulate to assess timetable robustness
- Set up a batch study varying dwell times or headways to find optimal operating parameters

**Key Workflows:**
- Simulate full-day timetable operations and measure punctuality, conflicts, and platform occupancy
- Perform capacity analysis per UIC 406 methodology and identify infrastructure bottlenecks
- Run Monte Carlo delay propagation studies to quantify timetable robustness
- Model energy consumption across fleet and evaluate eco-driving strategies
- Analyze infrastructure investment scenarios (double-tracking, grade separation) on capacity impact

---

### Manufacturing/Process
**Modified UI/UX:**
- Construction-focused views: earthwork quantities, material take-offs, and construction phasing timelines
- Integration panels for connecting to BIM execution platforms and construction management systems
- Quantity tables linked to alignment geometry with automatic updates on design changes

**Feature Gating:**
- Quantity extraction and material take-off tools enabled
- Full design editing locked; markup and RFI tools available
- Export to construction formats (LandXML, IFC, quantity schedules)

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Import approved design and generate construction quantity reports
- Set up earthwork mass-haul optimization for cut/fill balancing
- Link output to project scheduling tool for construction phasing

**Key Workflows:**
- Extract earthwork, track material, and structure quantities from design
- Optimize mass-haul plan to minimize spoil and borrow
- Generate construction phasing plans coordinated with possession (track closure) windows
- Produce as-built documentation by capturing field survey updates against design

---

### Regulatory/Compliance
**Modified UI/UX:**
- Standards compliance dashboard with checklist for applicable regulations (ERA TSIs, FRA, national standards)
- Automated geometric limit checking with flagged exceedances linked to specific chainage locations
- Environmental impact assessment integration: noise contours, vibration zones, ecological corridor crossings

**Feature Gating:**
- Design standard validation engine fully enabled for all supported standards
- Environmental impact assessment modules (noise, vibration, visual impact) accessible
- Safety case documentation tools and hazard log integration enabled

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Select applicable regulatory framework and design standards for the project jurisdiction
- Import design and run automated compliance check; review flagged items
- Generate compliance report and link to project safety case documentation

**Key Workflows:**
- Automated design compliance checking against applicable geometric standards
- Noise and vibration impact assessment along corridor with receptor mapping
- Generate regulatory submission packages (design approval, environmental impact statements)
- Maintain hazard log and safety case documentation linked to design decisions
- Track interoperability compliance for cross-border operations (ERA TSI requirements)

---

### Manager/Decision-maker
**Modified UI/UX:**
- Program dashboard showing project portfolio with status, cost, schedule, and risk indicators
- Scenario comparison cards for investment alternatives with benefit-cost ratios
- Stakeholder communication tools: automated progress reports, milestone tracking, public consultation materials

**Feature Gating:**
- Read-only access to all design and simulation outputs
- Investment appraisal and benefit-cost analysis tools enabled
- Approval workflow and decision-logging tools enabled

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Dashboard overview with KPI definitions (cost estimate confidence, schedule status, risk register)
- Connect to project team workspace and review active design alternatives
- Walkthrough of investment appraisal and scenario comparison tools

**Key Workflows:**
- Compare infrastructure investment alternatives on cost, capacity, journey time, and BCR
- Review and approve design milestones at stage-gate reviews
- Monitor program-level cost and schedule performance across project portfolio
- Evaluate risk register and mitigation strategies for major project decisions
- Generate stakeholder briefings and progress reports for governance boards
