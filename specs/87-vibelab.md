# VibeLab — Vibration and Modal Testing Analysis Platform

## Executive Summary

VibeLab is a cloud-based platform for experimental modal analysis (EMA), operational modal analysis (OMA), vibration monitoring, and structural health monitoring. It replaces ME'scope ($8K+/seat), LMS Test.Lab ($30K+/seat), Bruel & Kjaer PULSE ($25K+/seat), and m+p Analyzer ($10K+/seat) with a browser-based vibration analysis environment that connects directly to data acquisition hardware, extracts modal parameters using state-of-the-art algorithms, and provides AI-powered anomaly detection for predictive maintenance.

---

## Problem Statement

**The pain:**
- LMS Test.Lab (Siemens) costs $30,000-$80,000/seat for a full modal + acoustic testing suite; Bruel & Kjaer PULSE costs $25,000-$50,000; ME'scope costs $8,000-$20,000 — and all require separate expensive data acquisition hardware
- Modal testing requires deep expertise in signal processing, frequency response functions, curve fitting, and mode shape visualization — the learning curve is 6-12 months, and most vibration engineers are self-taught with knowledge gaps
- Operational modal analysis (output-only) is increasingly preferred because it avoids artificial excitation, but OMA algorithms (SSI, FDD, poly-reference) are implemented in niche academic tools with no commercial UX
- Vibration monitoring for predictive maintenance generates massive data volumes (continuous acceleration from 50+ sensors) but current tools are offline batch-processing — there is no real-time modal tracking platform
- Correlating test modal data with FEA models (modal updating, model validation) requires manual MAC matrix computation and sensitivity studies across disconnected software packages

**Current workarounds:**
- Using MATLAB/Python scripts for FFT, FRF estimation, and basic peak-picking modal extraction — missing advanced curve-fitting and uncertainty quantification
- Exporting data from proprietary DAQ software (National Instruments, Dewesoft, HBK) in UFF/CSV format and importing into separate analysis tools, losing metadata and test context
- Using free tools like ARTeMIS for OMA but with limited automation, no EMA integration, and per-project licensing that discourages routine use
- Vibration monitoring teams using simple threshold alarms on RMS levels instead of physics-based modal tracking, missing early-stage damage detection

**Market size:** The vibration testing and modal analysis software market is approximately $1.2 billion (2024). There are 100,000+ vibration/dynamics engineers worldwide across automotive NVH, aerospace structures, civil infrastructure, rotating machinery, and consumer electronics. The predictive maintenance market (where vibration monitoring is the leading technique) is projected to reach $30 billion by 2028.

---

## Target Users

### Primary Personas

**1. Raj — NVH Engineer at an Automotive OEM**
- Performs modal testing and operational deflection shape (ODS) analysis on vehicle body, chassis, and powertrain components
- Uses LMS Test.Lab which costs his department $200K+/year across 5 seats, with a 2-year contract lock-in
- Needs: affordable modal testing platform that interfaces with existing DAQ hardware and produces publication-quality mode shape animations

**2. Dr. Ingrid — Structural Health Monitoring Researcher**
- Monitors bridges and wind turbines using ambient vibration data, tracking natural frequency and damping changes as damage indicators
- Uses custom Python scripts for SSI-cov and FDD algorithms that lack robust stabilization diagram automation
- Needs: cloud-based OMA platform with automated stabilization diagram interpretation, continuous modal tracking, and anomaly alerting

**3. Tom — Vibration Analyst at an Industrial Plant**
- Performs vibration measurements on rotating machinery (pumps, motors, compressors, turbines) for predictive maintenance
- Uses Bruel & Kjaer portable analyzer with basic spectral analysis — cannot perform operational modal analysis or correlate with FEA models
- Needs: order tracking, envelope analysis, and operational deflection shapes for diagnosing machinery problems without shutting down equipment

---

## Solution Overview

VibeLab is a cloud-based vibration analysis platform that:
1. Connects to popular data acquisition hardware (NI, Dewesoft, HBK, IEPE/ICP accelerometers) via lightweight desktop agent for real-time data streaming and recording
2. Computes frequency response functions (FRF), auto/cross-spectra, coherence, and transmissibility with publication-quality signal processing
3. Extracts modal parameters (natural frequencies, damping ratios, mode shapes) from both impact/shaker testing (EMA) and ambient/operational data (OMA) using state-of-the-art algorithms
4. Provides 3D animated mode shape and operational deflection shape visualization on imported CAD/FEA meshes with MAC correlation to analytical models
5. Enables continuous vibration monitoring with automated modal tracking, trend analysis, and AI-powered anomaly detection for structural health and predictive maintenance

---

## Core Features

### F1: Signal Processing and Spectral Analysis
- Time-domain processing: resampling, filtering (Butterworth, Chebyshev, FIR), detrending, windowing (Hanning, Hamming, flat-top, exponential, force)
- FFT-based spectral analysis: auto-spectrum (PSD), cross-spectrum (CSD), coherence, transfer function (H1, H2, Hv estimators)
- FRF estimation from impact hammer and shaker tests with coherence quality checking
- Spectrogram (STFT) and wavelet transform (CWT, DWT) for time-frequency analysis
- Order tracking: computed order tracking (COT) for variable-speed rotating machinery
- Envelope analysis (amplitude demodulation) for bearing fault detection
- Cepstrum analysis for gear and bearing diagnostics
- Octave and 1/3-octave band analysis for vibration severity assessment (ISO 10816 / ISO 20816)
- Averaging: linear, exponential, peak-hold with overlap processing
- Channel calibration: sensitivity, engineering units, TEDS support

### F2: Experimental Modal Analysis (EMA)
- Test setup wizard: define geometry (import STL/OBJ/UFF mesh or build from nodes), assign DOFs, plan measurement grid
- Impact testing workflow: roving hammer or roving accelerometer with automatic hit quality checking (double-hit, overload, coherence)
- Shaker testing: random, burst random, sine sweep, stepped sine excitation with drive signal generation
- MIMO (Multiple-Input Multiple-Output): multiple shaker configurations for large structures
- Modal parameter extraction algorithms:
  - LSCE (Least Squares Complex Exponential) — time domain
  - PTD (Polyreference Time Domain) — multi-reference
  - RFP (Rational Fraction Polynomial) — frequency domain
  - PolyMAX (Polyreference Least-Squares Complex Frequency) — gold standard
  - LSCF (Least Squares Complex Frequency-domain)
- Stabilization diagram: automatic pole selection with stability criteria (frequency, damping, MAC)
- Mode shape complexity: real, complex, and complexity plot (MPD, MPC indicators)
- Modal Assurance Criterion (MAC) matrix for mode shape correlation
- Mode shape scaling: driving point residue, mass normalization

### F3: Operational Modal Analysis (OMA)
- Output-only modal identification — no excitation measurement required
- Frequency-domain methods:
  - FDD (Frequency Domain Decomposition)
  - EFDD (Enhanced FDD with damping estimation)
  - CFDD (Curve-fit FDD)
- Time-domain methods:
  - SSI-cov (Covariance-driven Stochastic Subspace Identification)
  - SSI-data (Data-driven SSI)
  - NExT-ERA (Natural Excitation Technique + Eigensystem Realization Algorithm)
- Automated stabilization diagram with clustering for robust pole selection
- Harmonic detection and removal (for structures with rotating machinery excitation)
- Confidence intervals on identified modal parameters (bootstrap, perturbation methods)
- Multi-setup OMA: merge data from sequential measurement campaigns for full mode shapes
- Transmissibility-based OMA (TOMA) for cases with limited sensor coverage

### F4: Operational Deflection Shapes and Visualization
- ODS from measured cross-spectra at any frequency or RPM
- 3D geometry import: STL, OBJ, STEP (surface), UNV/UFF (FEA mesh), Nastran BDF
- Animated mode shape and ODS visualization with user-controlled amplitude, speed, and perspective
- Side-by-side comparison: test mode shapes vs. FEA mode shapes
- MAC matrix and MAC bar chart: auto-MAC (test vs. test), cross-MAC (test vs. FEA)
- Frequency Response Assurance Criterion (FRAC) for FRF-level correlation
- Model updating interface: sensitivity of FEA modes to parameter changes (stiffness, mass, boundary conditions)
- Wireframe, surface, and contour plotting with color-mapped displacement magnitude
- Video export: high-resolution mode shape animation (MP4, GIF) for reports and presentations
- Multi-component visualization: overlay mode shapes of substructures (body + trim, rotor + stator)

### F5: Continuous Monitoring and Modal Tracking
- Streaming data ingestion from DAQ hardware via desktop agent (NI cDAQ, Dewesoft, HBK LAN-XI, generic IEPE via soundcard)
- Automated OMA: periodic modal identification (every 1 min to 1 hour) on streaming data
- Modal tracking dashboard: natural frequency, damping ratio, and mode shape trends over days/months/years
- Environmental/operational correction: temperature, wind speed, traffic load regression on modal parameters (ARX models)
- Anomaly detection: statistical process control (Shewhart, CUSUM, EWMA) on corrected modal parameters
- AI anomaly detection: autoencoder and isolation forest models trained on healthy-state vibration signatures
- Alert engine: email/SMS/webhook notifications when modal parameters deviate beyond thresholds
- Overall vibration level monitoring: RMS, peak, crest factor, kurtosis with ISO 10816/20816 severity zones
- Integration with SCADA/historian systems: OPC-UA, MQTT, REST API for operational context
- Data retention policies: configurable raw data retention, permanent storage of extracted features

### F6: Reporting and Data Management
- Test campaign management: organize measurements by structure, test date, configuration, boundary conditions
- UFF (Universal File Format) import/export for interoperability with legacy tools
- MATLAB .mat, NumPy .npy, CSV, and TDMS data import/export
- Automated test report generation: FRF plots, coherence, stabilization diagrams, mode table, mode shapes — PDF/HTML
- Modal parameter comparison tables across test configurations (before/after modification, different operating conditions)
- Project sharing and collaboration: role-based access for test engineers, analysts, and reviewers
- Audit trail: full history of processing parameters, algorithm settings, and analyst decisions
- Template workflows: save and reuse processing chains for repeated test types (production QC, periodic monitoring)
- API access: programmatic data upload, analysis triggering, and result extraction for CI/CD integration in product development

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | TypeScript, React, Three.js (3D mode shape visualization), D3.js (FRF plots, spectrograms, stabilization diagrams), WebGL (large dataset rendering) |
| Signal Processing | Rust (FFT via rustfft, FRF estimation, filtering, windowing, order tracking) → WASM for client-side + server for large datasets |
| Modal Extraction | Rust (LSCE, PTD, PolyMAX, SSI-cov, SSI-data, FDD) → server-side |
| AI/ML | Python (anomaly detection: autoencoder, isolation forest; automated pole selection: clustering; environmental correction: ARX regression) |
| DAQ Agent | Rust desktop agent (NI-DAQmx, Dewesoft SDK, HBK Open API, ASIO audio interface) → WebSocket streaming to cloud |
| Backend | Rust (Axum) + Python (FastAPI for ML inference, report generation) |
| Database | PostgreSQL (test metadata, modal parameters, sensor configurations), TimescaleDB (time-series vibration features, modal tracking), S3 (raw time histories, FRF matrices, mode shapes) |
| Hosting | AWS multi-region (c7g for modal extraction, GPU for ML training/inference, auto-scaling for batch processing) |

---

## Monetization

### Free Tier (Student)
- Import time-history data (CSV, UFF) up to 8 channels
- FFT, PSD, and basic FRF estimation
- Peak-picking modal extraction (no advanced curve fitting)
- 3 projects
- 50 MB data storage

### Pro — $119/month
- Unlimited channels
- Full EMA: PolyMAX, LSCE, PTD, RFP with stabilization diagrams
- Impact and shaker test workflows
- 3D mode shape animation on imported geometry
- MAC matrix computation
- ODS visualization
- PDF report generation
- 10 GB data storage

### Advanced — $299/user/month
- Everything in Pro
- Operational Modal Analysis (SSI, FDD, EFDD)
- Order tracking and envelope analysis
- Continuous monitoring with modal tracking dashboard
- AI anomaly detection
- Environmental/operational correction
- DAQ hardware streaming agent
- API access
- 100 GB data storage

### Enterprise — Custom
- Unlimited data storage and retention
- On-premise deployment
- Custom DAQ hardware integration
- Fleet-wide monitoring (100+ structures/machines)
- Integration with SCADA, CMMS, and digital twin platforms
- FEA model updating workflow
- Training and certification
- Dedicated support with SLA

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | VibeLab Advantage |
|-----------|-----------|------------|-------------------|
| LMS Test.Lab (Siemens) | Gold standard EMA, PolyMAX algorithm | $30-80K/seat, tied to Siemens hardware ecosystem | 50x cheaper, hardware-agnostic, cloud-native |
| ME'scope | Good visualization, affordable relative to LMS | $8-20K/seat, desktop-only, dated UI, no OMA | Modern UX, OMA + EMA, cloud collaboration |
| Bruel & Kjaer PULSE | Excellent hardware + software integration | $25-50K/seat, proprietary hardware lock-in | Works with any DAQ, 10x cheaper |
| ARTeMIS (SVS) | Best OMA algorithms, bridge monitoring | $5-15K per project license, no EMA, niche | Combined EMA + OMA, continuous monitoring, AI |
| m+p Analyzer | Flexible, good vibration control | $10-20K/seat, complex licensing, limited modal | Simpler UX, cloud-based, AI anomaly detection |

---

## MVP Scope (v1.0)

### In Scope
- Time-history data import (CSV, UFF, TDMS) with basic signal processing (FFT, PSD, windowing, filtering)
- FRF estimation (H1, H2) from imported impact test data
- Modal extraction via LSCE and PolyMAX with interactive stabilization diagram
- 3D mode shape visualization on imported STL/UFF geometry
- MAC matrix computation (auto-MAC and cross-MAC)
- Basic ODS from cross-spectral data
- PDF modal test report generation

### Out of Scope (v1.1+)
- Operational Modal Analysis (SSI, FDD)
- Live DAQ hardware streaming agent
- Continuous monitoring and modal tracking
- Order tracking and envelope analysis
- AI anomaly detection
- FEA model correlation and updating
- Environmental/operational correction

### MVP Timeline: 14-18 weeks
