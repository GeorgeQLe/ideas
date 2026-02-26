# GeoSat — Persona Subspecs

> Parent spec: [`specs/81-geosat.md`](../81-geosat.md)

---

## Experience-Level Personas

### Student/Academic
**Modified UI/UX:**
- Simplified orbit designer with preset templates (LEO, GEO, SSO, Molniya) and guided parameter entry
- Interactive 3D globe visualization with labeled reference frames and annotated Keplerian elements
- Tooltip overlays explaining perturbation models, J2 effects, and coordinate transformations

**Feature Gating:**
- Full access to two-body and J2-perturbed propagators; higher-fidelity models (solar radiation pressure, atmospheric drag tables) locked
- Export limited to CSV/JSON; CCSDS OEM/OPM export gated behind upgrade
- Constellation design limited to 6 satellites

**Pricing Tier:** Free tier

**Onboarding Flow:**
- Guided tutorial: propagate ISS TLE, visualize ground track, compute access windows to a ground station
- Pre-loaded example missions (Earth observation, comm relay) with annotated walkthroughs
- Link to textbook-style documentation mapping GeoSat tools to Vallado/Bate references

**Key Workflows:**
- Propagate a single satellite from TLE or Keplerian elements and plot ground track
- Compute eclipse and communication-window timing for a given ground station
- Compare Hohmann vs. bi-elliptic transfer costs for orbit-raising homework
- Generate 2D Porkchop plots for interplanetary trajectory surveys

---

### Junior Engineer (0-3 years)
**Modified UI/UX:**
- Tabbed workspace: Orbit Design | Maneuver Planning | Coverage Analysis with contextual side-panels
- Integrated delta-V budget calculator linked to active maneuver sequence
- Warning badges when propagation fidelity is insufficient for mission phase (e.g., low-altitude drag neglected)

**Feature Gating:**
- All propagation models enabled including high-fidelity numerical (RK7(8), Encke)
- Constellation walker patterns and custom phasing tools unlocked
- Monte Carlo dispersion analysis limited to 500 runs; full batch unlocked at higher tier

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Quickstart: import a spacecraft bus template, define orbit, run a 7-day propagation with contact schedule
- Prompt to connect to Space-Track.org for live TLE ingestion
- Short video on setting up maneuver sequences and reading delta-V reports

**Key Workflows:**
- Design a sun-synchronous orbit and verify LTAN drift over mission lifetime
- Plan station-keeping maneuvers for a GEO satellite with east-west/north-south corrections
- Run coverage analysis for a Walker-Delta constellation over a region of interest
- Export ephemeris in CCSDS OEM format for ground segment integration
- Perform conjunction screening against a TLE catalog

---

### Senior/Expert
**Modified UI/UX:**
- Multi-pane mission timeline with drag-and-drop maneuver nodes, branching scenario trees
- Scriptable analysis via embedded Python/Lua console with full API access to propagation engine
- Customizable dashboard with real-time metric feeds (lifetime, delta-V remaining, coverage %)

**Feature Gating:**
- All features unlocked: high-fidelity force models, electric propulsion low-thrust optimization, interplanetary trajectory design
- Unlimited Monte Carlo and batch processing with cluster offload
- Plugin SDK for custom force models and sensor definitions

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Import existing mission database (STK scenario, GMAT script) via migration wizard
- API key provisioning and CI/CD integration walkthrough for automated mission analysis
- Guided setup of custom force model plugins and organization-wide defaults

**Key Workflows:**
- End-to-end mission design from launch window selection through disposal orbit compliance
- Low-thrust trajectory optimization for electric propulsion transfers (SEP, Hall thruster)
- Multi-objective constellation optimization balancing coverage, revisit time, and launch cost
- Debris mitigation and post-mission disposal compliance verification (25-year rule, FCC guidelines)
- Collaborative scenario branching for trade studies across subsystem teams

---

## Role-Based Personas

### Designer/Creator
**Modified UI/UX:**
- Visual orbit builder with drag-handle manipulation of orbital elements on a 3D globe
- Color-coded coverage heat maps and ground-track overlays for quick design iteration
- Side-by-side comparison view for evaluating alternative orbit/constellation geometries

**Feature Gating:**
- Orbit design, constellation pattern generators, and visualization tools fully enabled
- Maneuver optimization and detailed perturbation analysis available but collapsed by default
- Export to 3D formats (glTF, CZML) for presentation and Cesium integration

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Start with mission objective wizard: select application (EO, comm, nav), constraints auto-populate
- Drag-to-design first orbit on globe, immediately see coverage footprint
- Template gallery of common constellation architectures (Walker, flower, Draim)

**Key Workflows:**
- Rapidly prototype constellation geometries and compare revisit statistics
- Generate Cesium/CZML visualizations for stakeholder presentations
- Explore orbit trade spaces using parametric sweeps (altitude vs. inclination vs. coverage)
- Design launch-compatible orbits matching vehicle performance and insertion accuracy

---

### Analyst/Simulator
**Modified UI/UX:**
- Data-centric layout: parameter tables, time-series plots, and statistical summary panels prominent
- Batch-run manager with queue status, progress bars, and result comparison tables
- Integrated Jupyter-style notebook environment for post-processing scripts

**Feature Gating:**
- Full propagation fidelity and Monte Carlo capabilities enabled
- Access to conjunction/collision probability computation modules
- API endpoints for automated batch analysis pipelines

**Pricing Tier:** Professional or Enterprise tier

**Onboarding Flow:**
- Import reference orbit and run a baseline high-fidelity propagation; compare against known ephemeris
- Set up first Monte Carlo dispersion on injection errors; review statistical output
- Connect analysis pipeline to organization's data lake or ground-system database

**Key Workflows:**
- Perform orbit determination and covariance realism assessment
- Run Monte Carlo campaigns on maneuver execution errors and assess mission-level impact
- Conjunction screening and collision probability computation for space safety
- Long-term orbit lifetime analysis under atmospheric drag and solar cycle variations
- Validate navigation filter performance using simulated measurement data

---

### Manufacturing/Process
**Modified UI/UX:**
- Simplified single-page interface focused on mass/volume/power budgets linked to orbit parameters
- Integration panels for importing spacecraft bus specifications from PLM/PDM systems
- Gantt-style timeline showing mission phases aligned with AIT milestones

**Feature Gating:**
- Orbit-dependent environment outputs (thermal flux, radiation dose, atomic oxygen fluence) enabled
- Full orbit design tools hidden; pre-defined mission profile import only
- Export to requirement-tracking formats (ReqIF, DOORS-compatible CSV)

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Import spacecraft bus definition and assign to a pre-defined mission orbit
- Generate environment exposure report (eclipse cycles, thermal flux, radiation dose)
- Link outputs to requirement verification matrix template

**Key Workflows:**
- Extract orbit-derived environmental loads for component qualification testing
- Generate eclipse/sunlight cycling profiles for thermal vacuum test planning
- Produce radiation dose-depth curves for parts selection and shielding design
- Export mission timeline for AIT scheduling integration

---

### Regulatory/Compliance
**Modified UI/UX:**
- Dashboard focused on compliance metrics: orbital debris mitigation, spectrum coordination, re-entry risk
- Traffic-light status indicators for each regulatory requirement (FCC, ITU, ESA Space Debris Mitigation)
- Automated report generation with pre-formatted templates for filing packages

**Feature Gating:**
- Debris mitigation analysis tools (25-year rule, casualty risk) fully enabled
- Frequency coordination and link-budget modules accessible
- Orbit design tools limited to compliance-relevant parameters only

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Select applicable regulatory frameworks (FCC, ITU, French Space Act, ESA standards)
- Import mission orbit and spacecraft parameters; auto-run compliance checks
- Review generated compliance report and flag items requiring attention

**Key Workflows:**
- Verify 25-year post-mission disposal compliance under Monte Carlo atmospheric uncertainty
- Compute on-ground casualty expectation for uncontrolled re-entry risk assessment
- Generate ITU coordination filing data (orbital parameters, beam footprints, PFD masks)
- Produce debris mitigation plan documentation per NASA-STD-8719.14 or ISO 24113
- Track and audit compliance status across a multi-satellite constellation

---

### Manager/Decision-maker
**Modified UI/UX:**
- Executive dashboard: mission health KPIs, cost summaries, schedule risk, and constellation status at a glance
- Scenario comparison cards showing top-level trade-study results with cost/performance scoring
- Drill-down capability from summary metrics to underlying technical detail

**Feature Gating:**
- Read-only access to all analysis results and reports
- Scenario comparison and trade-study summary tools enabled
- Direct orbit-design and simulation tools hidden; delegation to analyst roles

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Overview of dashboard layout and KPI definitions
- Connect to team workspace to see live mission analysis status
- Walkthrough of scenario comparison and approval workflow

**Key Workflows:**
- Review and compare constellation architecture trade studies (cost vs. coverage vs. latency)
- Approve mission design milestones (PDR/CDR orbit baseline, maneuver budget allocation)
- Monitor constellation deployment progress and on-orbit performance metrics
- Evaluate risk posture from Monte Carlo campaign summaries
- Generate executive summaries for board or customer reviews
