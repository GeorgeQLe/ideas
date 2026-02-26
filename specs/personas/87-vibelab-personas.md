# VibeLab — Persona Subspecs

> Parent spec: [`specs/87-vibelab.md`](../87-vibelab.md)

---

## Experience-Level Personas

### Student/Academic
**Modified UI/UX:**
- Simplified signal acquisition panel with preset configurations for common accelerometer setups (single-axis, triaxial)
- Animated mode-shape viewer with labeled node/antinode points and natural frequency annotations
- Interactive FFT display with adjustable window, overlap, and frequency resolution parameters and real-time preview

**Feature Gating:**
- Data import limited to WAV, CSV, and UFF format; DAQ hardware integration locked
- Modal analysis limited to 10 channels and 20 modes using peak-picking and circle-fit methods
- Operating deflection shape (ODS) visualization enabled; operational modal analysis (OMA) locked

**Pricing Tier:** Free tier

**Onboarding Flow:**
- Tutorial: import a sample vibration dataset, compute FRFs, extract natural frequencies and mode shapes of a simple beam
- Pre-loaded examples (cantilever beam, plate modal test, rotating machinery vibration)
- References to Ewins, Inman, and de Silva vibration textbooks linked from analysis tools

**Key Workflows:**
- Import vibration signals and compute auto/cross power spectra and frequency response functions
- Extract natural frequencies and damping ratios using peak-picking from FRF data
- Visualize mode shapes on a simple wireframe geometry
- Compare experimental modal results against analytical beam/plate solutions

---

### Junior Engineer (0-3 years)
**Modified UI/UX:**
- Project workspace with tabs: Measurement Setup | Signal Processing | Modal Analysis | Reporting
- DAQ hardware configuration wizard with auto-detection of supported devices (NI, Bruel & Kjaer, PCB)
- Guided impact-testing workflow with hit-quality indicators (double-hit detection, overload, coherence check)

**Feature Gating:**
- Full DAQ integration up to 32 channels with real-time monitoring and triggering
- Modal analysis methods: LSCF (PolyMAX-equivalent), ERA, CMIF with stabilization diagram
- Transfer path analysis (TPA) module gated; basic order tracking for rotating machinery enabled

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Connect DAQ hardware, configure channels, and perform a live impact hammer modal test on a test structure
- Process data through FRF computation and extract modes using stabilization diagram
- Generate a modal test report with frequency, damping, mode shapes, and MAC matrix

**Key Workflows:**
- Conduct impact hammer or shaker modal tests with multi-channel DAQ acquisition
- Extract modal parameters using advanced curve-fitting (LSCF) with stabilization diagrams
- Validate modal model quality using MAC, MPC, and synthesis-vs-measurement comparisons
- Perform order tracking analysis on rotating machinery vibration data
- Generate engineering reports with modal parameter tables and animated mode-shape GIFs

---

### Senior/Expert
**Modified UI/UX:**
- Multi-project test campaign manager with cross-test comparison and model updating workflows
- Python/MATLAB scripting interface with full API for custom signal processing and analysis algorithms
- FE model correlation workspace with side-by-side experimental vs. analytical mode comparison

**Feature Gating:**
- All features unlocked: operational modal analysis (OMA), transfer path analysis (TPA), model updating
- Unlimited channel count with distributed DAQ synchronization
- Plugin SDK for custom analysis algorithms and hardware driver integration

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Import existing test databases from ME'scope/LMS and configure organization measurement standards
- Set up API connections to FEA tools (NASTRAN, Abaqus) for model correlation workflows
- Workshop on operational modal analysis and advanced TPA methodologies

**Key Workflows:**
- Operational modal analysis of large civil/aerospace structures under ambient excitation
- Transfer path analysis identifying dominant vibration/noise transmission paths
- FE model updating using experimental modal data to improve simulation accuracy
- Multi-campaign structural health monitoring with statistical damage detection
- Custom algorithm development for specialized vibration diagnostics

---

## Role-Based Personas

### Designer/Creator
**Modified UI/UX:**
- Test geometry builder with CAD import and sensor placement optimization tools
- Visual sensor-layout editor showing coverage maps and mode-shape observability predictions
- Animated mode-shape viewer with export to video/GIF for presentations and design reviews

**Feature Gating:**
- Geometry and sensor placement design tools fully enabled
- Pre-test analysis (optimal sensor placement, excitation planning) accessible
- Modal analysis available; advanced TPA and model updating collapsed by default

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Import CAD geometry and define test boundary conditions
- Use sensor placement optimization to determine accelerometer locations for target modes
- Plan impact locations and excitation strategy for complete modal coverage

**Key Workflows:**
- Design measurement setups with optimal sensor and excitation locations
- Create test geometries from CAD models with measurement point definitions
- Visualize and animate experimental mode shapes for design-review presentations
- Compare mode shapes across design iterations to track structural dynamic changes

---

### Analyst/Simulator
**Modified UI/UX:**
- Analysis-centric layout with multi-pane signal viewer, spectral analysis tools, and modal parameter workspace
- Model correlation panel with MAC matrix, frequency comparison, and mode-pair tracking
- Batch processing manager for analyzing large test campaigns and monitoring datasets

**Feature Gating:**
- Full signal processing and modal analysis toolkit enabled
- FE model correlation and updating tools accessible
- API access for automated analysis pipelines and custom algorithm integration

**Pricing Tier:** Professional or Enterprise tier

**Onboarding Flow:**
- Import test data and FE model results, compute MAC matrix, and identify correlated/uncorrelated modes
- Perform sensitivity analysis on FE model parameters to plan model updating
- Run automated model updating to minimize frequency and mode-shape error

**Key Workflows:**
- Comprehensive modal analysis using multiple extraction methods and cross-validation
- FE-test correlation with MAC, frequency error, and mode-shape comparison at DOFs
- Model updating using experimental data to calibrate FE model parameters
- Operational deflection shape (ODS) analysis for vibration troubleshooting
- Time-frequency analysis (STFT, wavelet) for non-stationary vibration signals

---

### Manufacturing/Process
**Modified UI/UX:**
- Production-line testing interface with pass/fail criteria and go/no-go indicators
- Pre-configured test templates for quality-control vibration testing (end-of-line, incoming inspection)
- Integration panels for MES/QMS systems and barcode/serial number tracking

**Feature Gating:**
- Automated test execution with predefined sequences and acceptance criteria
- Full analysis tools hidden; only pass/fail results and trend charts visible
- Export to quality management formats and MES/QMS system integration

**Pricing Tier:** Professional tier

**Onboarding Flow:**
- Import pre-configured test template with acceptance criteria from engineering team
- Run automated end-of-line vibration test on a sample product
- Review pass/fail result and link to serial number in quality database

**Key Workflows:**
- Execute automated end-of-line vibration tests with pass/fail classification
- Track production quality trends using statistical process control on vibration metrics
- Perform incoming inspection testing on components and assemblies
- Generate quality deviation reports with vibration signature comparisons to golden units

---

### Regulatory/Compliance
**Modified UI/UX:**
- Standards compliance dashboard with test protocol templates (ISO 7626, ISO 10816, IEC 60068)
- Test procedure audit trail showing measurement chain calibration, test conditions, and uncertainty budgets
- Automated compliance report generator with standard-specific formatting requirements

**Feature Gating:**
- Standard-specific test protocol templates and compliance checking enabled
- Measurement uncertainty calculation tools accessible
- Calibration management and traceability documentation tools enabled

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Select applicable vibration testing standards for the product/industry
- Configure measurement chain calibration records and uncertainty budgets
- Run a compliance test and generate standard-specific report

**Key Workflows:**
- Conduct vibration tests per ISO, IEC, MIL-STD, or customer-specific standards
- Calculate measurement uncertainty budgets per ISO/IEC Guide 98 (GUM)
- Generate certification test reports with full measurement traceability
- Maintain calibration records for sensors, DAQ, and impact hammers
- Audit vibration test procedures against accreditation requirements (ISO 17025)

---

### Manager/Decision-maker
**Modified UI/UX:**
- Test program dashboard with project status, resource utilization, and quality trend summaries
- Cost-of-quality metrics: test costs, failure rates, warranty claim correlations
- Capacity planning view showing test lab utilization and scheduling

**Feature Gating:**
- Read-only access to all test results and compliance reports
- Quality trend analysis and cost-of-quality calculators enabled
- Resource planning and approval workflow tools enabled

**Pricing Tier:** Enterprise tier

**Onboarding Flow:**
- Dashboard overview: test program status, quality metrics, and lab utilization
- Review quality trend data and warranty claim correlations
- Set up notification preferences for test failures and compliance deadlines

**Key Workflows:**
- Monitor product quality trends through vibration test metrics across production lines
- Evaluate test program effectiveness and ROI on vibration testing investments
- Review and approve test procedure changes and acceptance criteria updates
- Plan test lab capacity and capital equipment investments
- Generate executive quality reports for management reviews and customer audits
