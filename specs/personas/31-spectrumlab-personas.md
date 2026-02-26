# SpectrumLab — Persona Subspecs

> Parent spec: [`specs/31-spectrumlab.md`](../31-spectrumlab.md)

---

## Experience-Level Personas

### Student/Academic
**Modified UI/UX:**
- Guided mode with labeled DSP block categories (Filters, Transforms, Demodulation) and inline math previews showing equations behind each operation
- Simplified spectrum display with auto-scaled axes and annotation prompts ("What is this signal?") to reinforce learning
- Lab assignment integration panel that shows exercise instructions alongside the working canvas

**Feature Gating:**
- Full access to FFT analysis, basic filtering, and AM/FM demodulation blocks
- SDR device connectivity limited to common academic devices (RTL-SDR, HackRF); advanced SIGINT classification and export-controlled demodulators hidden
- Parameter sweep and batch processing capped at 10 concurrent runs

**Pricing Tier:** Free tier (academic email verification)

**Onboarding Flow:**
- First-run tutorial walks through capturing a local FM broadcast signal, viewing waterfall display, and measuring bandwidth
- Pre-loaded sample IQ recordings (FM radio, ADS-B, weather satellite) so students without SDR hardware can still explore
- Prompt to join a class workspace if an instructor invite code is available

**Key Workflows:**
- Capture and visualize RF spectrum from RTL-SDR for lab exercises
- Apply windowed FFT and measure spectral characteristics of known signals
- Build a simple DSP flowgraph (filter, decimate, demodulate) for a communications course assignment
- Compare simulated vs. captured signal spectra to validate theoretical predictions
- Export annotated spectrum plots for lab reports

---

### Junior Engineer (0-3 years)
**Modified UI/UX:**
- Default workspace with common analysis templates (spectrum survey, modulation analysis, interference hunting) pre-configured
- Contextual tooltips explaining DSP parameter choices (e.g., "Window size affects frequency resolution vs. time resolution")
- Collapsible advanced panels for solver settings and custom script injection, keeping the main view clean

**Feature Gating:**
- Access to all standard analysis tools: FFT, spectrogram, digital demodulation (PSK, QAM, OFDM), and signal classification
- Batch processing up to 50 concurrent jobs; GPU-accelerated processing enabled
- Custom DSP block scripting available but not surfaced by default; export to PDF reports enabled

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Guided project setup: select SDR hardware or upload IQ file, choose analysis template, configure frequency range and sample rate
- Interactive walkthrough of a 5G NR signal analysis workflow showing capture-to-report pipeline
- Prompt to connect team workspace and set up shared signal library

**Key Workflows:**
- Capture over-the-air signals and identify interference sources in a deployment environment
- Run modulation quality analysis (EVM, constellation diagram) on 5G/LTE signals
- Build reusable analysis templates for recurring test procedures
- Generate compliance test reports with annotated spectra and measurement tables
- Collaborate with senior engineers by sharing annotated recordings in the team signal library

---

### Senior/Expert
**Modified UI/UX:**
- Power-user layout with multi-pane spectrum/waterfall/constellation/time-domain views configurable via drag-and-drop
- Full scripting console (Python/Julia) docked alongside visual tools for custom DSP algorithm development
- Keyboard shortcuts and macro recording for repetitive analysis sequences

**Feature Gating:**
- All features unlocked: advanced demodulators, custom DSP block authoring, REST API access, GPU cluster burst compute
- Unlimited batch processing and parameter sweeps; priority queue scheduling
- Admin controls for team signal libraries, access permissions, and export-controlled content management

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Express setup: import existing GNU Radio flowgraphs or MATLAB scripts to bootstrap workspace
- API key generation and CI/CD integration walkthrough for automated signal monitoring pipelines
- Team administration setup: invite members, configure role-based access, establish shared libraries

**Key Workflows:**
- Develop and validate novel DSP algorithms using the integrated scripting environment with GPU acceleration
- Run automated spectrum monitoring campaigns across multiple SDR nodes with alert rules
- Build custom demodulation chains for non-standard or classified signal formats
- Manage team-wide signal databases with standardized metadata, annotations, and access controls
- Architect reproducible analysis pipelines that junior engineers can execute as templates

---

## Role-Based Personas

### Designer/Creator
**Modified UI/UX:**
- Visual flowgraph canvas emphasized as primary workspace, with drag-and-drop DSP blocks and wiring
- Real-time signal preview at each node in the processing chain for immediate visual feedback
- Component palette organized by function (Sources, Filters, Transforms, Sinks) with search and favorites

**Feature Gating:**
- Full access to flowgraph editor, all DSP blocks, and custom block authoring
- Simulation mode enabled (generate synthetic test signals without SDR hardware)
- Export flowgraphs as reusable templates; version control for flowgraph designs

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Start with a blank canvas tutorial: place a signal source, add a filter, visualize output
- Template gallery showing common RF system designs (receiver chains, channelizers, demodulators)
- Guide to creating and publishing custom DSP blocks for team reuse

**Key Workflows:**
- Design complete receiver signal processing chains from antenna to decoded bitstream
- Prototype new DSP algorithms visually before committing to code
- Create parameterized flowgraph templates for production signal processing deployments
- Simulate signal chain performance with synthetic inputs to validate designs before hardware integration
- Publish verified processing blocks to the team component library

---

### Analyst/Simulator
**Modified UI/UX:**
- Analysis dashboard as default view: waterfall, spectrum, and measurement panels arranged for efficient signal identification
- Quick-access toolbar for common measurements (bandwidth, center frequency, modulation type, SNR)
- Batch comparison view for overlaying multiple signal captures with synchronized time axes

**Feature Gating:**
- Full spectrum analysis, signal classification, and measurement extraction tools
- Automated signal detection and cataloging enabled; ML-based signal identification models accessible
- Database query and export tools for signal libraries; bulk reporting enabled

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Load a sample multi-signal capture and walk through identifying, isolating, and classifying each emitter
- Configure automated detection thresholds and classification confidence levels
- Set up a signal database with metadata schema for cataloging identified signals

**Key Workflows:**
- Survey a frequency band and catalog all detected emitters with modulation type, bandwidth, and duty cycle
- Perform interference analysis by correlating signal timing and spectral overlap
- Run statistical analysis on signal characteristics across time to identify patterns or anomalies
- Compare captured signals against a reference library for rapid identification
- Generate spectrum survey reports with annotated findings for regulatory or intelligence customers

---

### Manufacturing/Process
**Modified UI/UX:**
- Production test dashboard showing pass/fail status, test sequence progress, and trend charts
- Simplified controls: large start/stop buttons, status indicators, and automated test sequence runners
- Equipment connectivity panel for managing SDR hardware fleet and calibration schedules

**Feature Gating:**
- Access to predefined test sequences and pass/fail criteria; editing locked to approved templates
- Calibration management and hardware health monitoring tools enabled
- Custom flowgraph creation disabled; template execution and result logging only

**Pricing Tier:** Enterprise tier (volume licensing)

**Onboarding Flow:**
- Connect production SDR hardware and verify calibration status
- Load approved test sequence templates and configure pass/fail thresholds
- Set up result logging to quality database with operator identification

**Key Workflows:**
- Execute standardized RF compliance tests on production devices using approved templates
- Monitor production test yield trends and flag devices that fail spectral mask requirements
- Log test results with traceability (operator, timestamp, equipment serial, calibration date)
- Manage SDR hardware fleet calibration schedules and replacement cycles
- Escalate anomalous test results to engineering with attached signal captures

---

### Regulatory/Compliance
**Modified UI/UX:**
- Compliance-focused dashboard with regulatory limit overlays (FCC Part 15, ETSI EN 300 328, MIL-STD-461) on spectrum views
- Automated margin calculation: distance between measured emissions and regulatory thresholds displayed in real time
- Report generation wizard with pre-formatted templates for FCC, CE, and ISED submissions

**Feature Gating:**
- Full access to emission measurement tools, limit line editors, and compliance report generators
- Regulatory template library with current versions of major standards; limit line auto-updates
- Audit trail logging for all measurements; data integrity signatures on exported reports

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Select applicable regulatory standards and configure default limit line overlays
- Walk through a sample compliance measurement: spurious emissions scan with automated pass/fail
- Set up report templates with company branding and test lab accreditation information

**Key Workflows:**
- Perform pre-compliance spurious emission scans against FCC/CE limit lines
- Generate formatted compliance test reports for regulatory submission with measurement uncertainty
- Maintain an audit trail of all compliance measurements with tamper-evident records
- Compare device emissions across firmware versions or hardware revisions to track compliance drift
- Review and approve compliance reports submitted by test engineers before filing

---

### Manager/Decision-maker
**Modified UI/UX:**
- Executive dashboard showing project status, team utilization, and key metrics (tests completed, issues found, compliance status)
- Aggregated views: no raw spectrum data, only summary charts, trend lines, and traffic-light status indicators
- Cost tracking panel showing compute usage, SDR hardware utilization, and per-project spend

**Feature Gating:**
- Read-only access to project summaries, reports, and dashboards; no direct analysis tool access
- Team management: user provisioning, role assignment, and license allocation
- Budget controls: set compute spending limits, approve burst compute requests

**Pricing Tier:** Enterprise tier (included in admin seat)

**Onboarding Flow:**
- Overview of dashboard widgets: project health, team activity, resource utilization
- Set up team structure: invite engineers, assign roles, configure project workspaces
- Configure alerting: notifications for compliance failures, budget thresholds, and project milestones

**Key Workflows:**
- Monitor team productivity and project progress across multiple RF analysis campaigns
- Review compliance status across product lines before regulatory submission deadlines
- Allocate SDR hardware and compute budget across projects based on priority
- Approve and track engineering change requests related to RF design modifications
- Generate executive summaries of spectrum survey findings for stakeholder briefings
