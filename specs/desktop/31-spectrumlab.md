# SpectrumLab Desktop — 5-Framework Comparison

## Desktop Rationale

- **Real-Time Signal Capture:** Acquiring live audio/RF signals from hardware devices (SDR dongles, sound cards, spectrum analyzers) requires low-latency USB/PCIe access with sub-millisecond buffer management — only possible with native device drivers and direct hardware interfaces. Professional SDR workflows demand sustained sample rates of 20+ Msps with guaranteed zero-drop capture that browser sandboxes fundamentally cannot provide.

- **GPU-Accelerated FFT Processing:** Real-time spectral analysis of wideband signals demands GPU-accelerated FFT (cuFFT/Metal FFT/wgpu compute) processing millions of samples per second — browser WebAudio FFT is limited to 32K points at 48kHz and cannot handle multi-megahertz bandwidths. A 10 MHz bandwidth signal sampled at 20 Msps requires computing 1M-point FFTs at 20 Hz update rate, generating 40M complex multiply-accumulate operations per second that only GPU compute can sustain in real-time.

- **Low-Latency Signal Display:** Waterfall displays, real-time spectrum plots, and constellation diagrams must update at 30-60fps with microsecond-accurate time alignment — native GPU rendering pipelines eliminate the JavaScript event loop bottleneck. A waterfall display scrolling at 60fps with 4096-point frequency resolution across 10 seconds of history requires streaming 2.4M pixels per frame to the GPU.

- **Hardware Interface Access:** Direct USB communication with RTL-SDR, HackRF, USRP, and professional spectrum analyzers requires libusb or vendor-specific driver APIs that browsers cannot access; audio devices need ASIO/Core Audio/ALSA for low-latency capture with configurable buffer sizes down to 64 samples.

- **Offline Analysis Capability:** Signal analysts frequently work in RF-shielded environments, SCIF facilities, field deployments, and classified networks where internet connectivity is unavailable or prohibited — all processing, demodulation, and classification must run entirely on local hardware.

---

## Desktop-Specific Features

The following features are only feasible on desktop due to hardware interface requirements, real-time latency constraints, and GPU compute needs:

- Direct USB/PCIe hardware interface for SDR devices (RTL-SDR, HackRF, USRP, Airspy, LimeSDR) and professional spectrum analyzers via SoapySDR abstraction layer
- GPU-accelerated FFT engine via wgpu compute shaders for real-time spectral analysis of signals up to 20 MHz bandwidth with up to 16M-point transform capability
- Low-latency waterfall display with configurable color maps (viridis, plasma, magma, turbo), adjustable time span, and frequency resolution rendered via native GPU pipeline at 60fps
- Core Audio / ASIO / ALSA / PulseAudio integration for professional-grade audio capture with sub-millisecond latency and configurable sample rates up to 192kHz
- Real-time digital signal demodulation: AM, FM, SSB, CW, PSK31/63, QAM-16/64/256, OFDM with configurable parameters and audio output
- Multi-channel signal recording to disk with lossless IQ sample storage in SigMF format at sustained 40 MB/s write rates for continuous capture
- Spectrogram annotation and bookmarking system for marking signals of interest with timestamp, frequency, bandwidth, and classification metadata
- Configurable signal processing chain: filters (FIR/IIR), decimation, interpolation, mixing, AGC, squelch as drag-and-drop processing blocks with parameter knobs
- Multi-monitor support: waterfall on primary display, demodulated audio scope, constellation diagram, and eye diagram on secondary display
- Background recording daemon with trigger-based capture — automatically start recording when signal energy exceeds configurable threshold in a frequency band

---

## Shared Packages

All shared packages are compiled as Rust libraries with C-compatible FFI exports, enabling consumption from any framework.

### core (Rust)

- GPU-accelerated FFT engine using wgpu compute shaders with radix-2/4/8 butterfly implementations supporting transforms from 256-point to 16M-point with configurable windowing (Hann, Hamming, Blackman-Harris, Kaiser)
- Digital signal processing library: FIR/IIR filter design (Parks-McClellan optimal, Butterworth, Chebyshev Type I/II, elliptic), polyphase sample rate conversion, Hilbert transform, and analytic signal generation
- Demodulation engine: AM/FM/SSB envelope and frequency discriminators with noise gating, PSK/QAM symbol timing recovery via Gardner/Mueller-Muller, carrier synchronization via Costas loop, OFDM cyclic prefix detection and channel estimation
- SDR hardware abstraction layer wrapping SoapySDR for unified access to RTL-SDR, HackRF, USRP, Airspy, LimeSDR, and PlutoSDR with automatic gain, frequency, and sample rate configuration
- Signal detection and classification pipeline using energy detection with CFAR thresholding, cyclostationary feature analysis, and local ONNX model inference for automatic modulation recognition
- IQ sample file I/O: SigMF, WAV (complex float), raw binary (int8/int16/float32), and HDF5 readers/writers with memory-mapped access for multi-gigabyte recording playback

### api-client (Rust)

- Signal database sync client for downloading/uploading signal reference profiles, modulation templates, and frequency allocation tables
- Firmware update client for supported SDR hardware with version checking, integrity verification, and safe rollback support
- Cloud processing offload for computationally intensive tasks (wideband channelization, ML batch classification, direction-finding correlation)
- License validation with offline token caching supporting 30-day grace period for field deployments and disconnected operations
- Community signal sharing client for uploading interesting captures with metadata and spectral thumbnails to public signal identification database

### data (SQLite)

- Signal recording catalog with frequency, bandwidth, timestamp, duration, SNR, modulation type, and file path metadata indexed for sub-second search across thousands of recordings
- Processing chain presets storing complete filter configurations, demodulation parameters, display settings, and hardware tuning as named reusable templates
- Hardware configuration database tracking SDR device calibration data, gain tables, frequency correction offsets, and IQ imbalance compensation parameters
- Annotation store linking user-created bookmarks, notes, modulation classifications, and emitter identifiers to specific time-frequency regions in recordings with spatial indexing

---

## Framework Implementations

Each framework is evaluated for its ability to deliver low-latency signal capture, GPU-accelerated FFT processing, and real-time spectral display.

### Electron

**Architecture:** Chromium renderer for React-based control panels and signal configuration UI; WebGL2 custom shaders for waterfall display with texture-based scrolling; Node.js native addons for SDR hardware access via SoapySDR bindings; Worker threads for FFT processing; SharedArrayBuffer for sample data streaming to renderer

**Tech stack:** React + TypeScript, custom WebGL2 waterfall renderer with GLSL fragment shaders, Node.js native addons via napi-rs, Web Audio API for demodulated audio output, better-sqlite3 for signal database

**Native module needs:** napi-rs bindings to SoapySDR for hardware access, node-fft-napi for CPU FFT fallback, libusb Node.js bindings for direct USB device access, PortAudio native addon for low-latency audio I/O

**Bundle size:** ~260-330 MB

**Memory:** ~400-650 MB

**Pros:**
- Web Audio API provides basic audio processing and output capabilities out of the box for demodulated signal playback
- Large ecosystem of JavaScript DSP libraries for rapid prototyping of signal processing chains and filter designs
- Cross-platform with consistent UI for spectrum display across Windows/macOS/Linux analyst workstations
- Familiar React development model for building complex signal configuration interfaces with parameter controls

**Cons:**
- WebGL2 waterfall rendering caps out at ~30fps for wideband signals due to texture upload bottleneck on the main thread
- No compute shader access from WebGL2 — FFT must run on CPU via native addon or in separate Rust process with IPC overhead
- Web Audio API limited to 48kHz sample rate and 32K-point FFT — completely useless for RF signal processing at MHz bandwidths
- USB device access requires native addon bypassing Chromium sandbox — fragile across OS versions and platform-specific

### Tauri

**Architecture:** Rust backend handling all DSP: SDR hardware via SoapySDR, GPU FFT via wgpu compute shaders, demodulation, and filtering; wgpu renderer for waterfall display sharing GPU context with FFT compute pipeline; WebView frontend for controls and configuration; lock-free ring buffer in shared memory for zero-copy sample flow from capture thread to FFT compute to display renderer

**Tech stack:** Rust backend with wgpu compute + rendering, SolidJS frontend for reactive controls, SoapySDR Rust bindings, cpal for cross-platform audio I/O, sqlx for SQLite

**Native module needs:** wgpu for GPU FFT and waterfall rendering, SoapySDR via C FFI, cpal for cross-platform audio capture/playback, libusb Rust bindings for direct USB device access

**Bundle size:** ~40-65 MB

**Memory:** ~100-220 MB

**Pros:**
- Zero-copy pipeline: SDR samples flow from USB capture thread to lock-free ring buffer to GPU FFT compute shader to waterfall renderer without serialization or memory copies
- wgpu compute shaders for FFT run in the same GPU context as the waterfall renderer — FFT output buffer IS the texture source, no upload needed
- Rust's ownership model and lock-free data structures prevent data races in the multi-threaded capture-process-display pipeline
- Minimal end-to-end latency: sub-millisecond from sample capture to screen pixel update with direct hardware access and zero IPC

**Cons:**
- Must build custom waterfall renderer in wgpu — no off-the-shelf spectral display widget exists in the Rust ecosystem
- SoapySDR Rust bindings less mature than C++/Python equivalents — may need FFI wrapper maintenance for new device support
- WebView limitations for complex signal processing chain builder UI with visual drag-and-drop block connections
- Smaller DSP library ecosystem in Rust compared to Python (SciPy, GNU Radio) and C++ — more algorithms must be implemented from scratch

### Flutter Desktop

**Architecture:** Impeller/Skia for control panel UI and parameter knobs; native texture bridge for waterfall display rendered via platform-specific GPU API; Dart FFI to Rust DSP core; platform channels for SDR hardware access and sample streaming

**Tech stack:** Dart + Flutter, flutter_rust_bridge for DSP core FFI, custom native texture plugin for waterfall rendering, drift for SQLite, fl_chart for spectrum line plots

**Native module needs:** Rust DSP core via FFI, platform-specific GPU interop for waterfall texture, SoapySDR via C FFI, PortAudio via C FFI for low-latency audio I/O

**Bundle size:** ~80-115 MB

**Memory:** ~250-380 MB

**Pros:**
- Smooth UI for building signal processing chain configuration panels with animated drag-and-drop block connections
- Cross-platform control panel code shared across Windows/macOS/Linux deployments with consistent look and behavior
- Hot reload speeds iteration on complex DSP parameter adjustment interfaces, frequency band selectors, and gain controls
- Flutter Canvas API provides basic 2D spectrum line plotting capability without requiring the native texture bridge

**Cons:**
- Native texture bridge for waterfall display introduces a frame of latency — problematic for real-time signal monitoring at 60fps
- No direct hardware access from Dart — all SDR device communication must transit the FFI boundary with associated overhead
- GPU compute for FFT requires platform-specific native code paths, not accessible from the Flutter/Dart layer directly
- Dart garbage collection can cause audio glitches and sample drops in low-latency capture scenarios with tight timing requirements

### Swift/SwiftUI (macOS)

**Architecture:** SwiftUI for native macOS signal analysis UI with custom controls; Metal compute shaders for GPU-accelerated FFT and spectral processing; Metal renderer for waterfall and constellation displays; Core Audio for lowest-latency audio capture and playback; Swift-Rust FFI for demodulation algorithms and signal classification

**Tech stack:** SwiftUI + Metal, Core Audio for audio capture/playback, Metal compute for GPU FFT, IOKit for USB SDR device access, Swift-Rust FFI for DSP core, GRDB/SQLite for signal database

**Native module needs:** Metal compute shaders (native), Core Audio (native), IOKit for USB SDR access (native), Rust DSP core via C-compatible FFI

**Bundle size:** ~25-45 MB

**Memory:** ~90-180 MB

**Pros:**
- Core Audio provides the lowest-latency audio capture on macOS with direct hardware buffer access and configurable 32-sample buffer sizes
- Metal compute FFT on Apple Silicon achieves highest throughput for spectral processing on Mac hardware with unified memory
- Unified memory architecture means FFT output buffer in GPU memory is instantly available for Metal rendering — true zero copy
- IOKit provides robust USB device access for SDR hardware without third-party library dependencies or sandbox workarounds

**Cons:**
- macOS-only — signal analysts and SDR enthusiasts use Linux extensively, especially with GNU Radio and GR-compatible flowgraphs
- Core Audio is entirely macOS-specific — no equivalent low-latency audio API portability to Linux (ALSA/JACK) or Windows (ASIO)
- Metal compute shaders not portable — CUDA and Vulkan compute dominate GPU signal processing on Linux/Windows SDR workstations
- Smaller Swift SDR/DSP community compared to C++/Python GNU Radio ecosystem — fewer reusable signal processing blocks

### Kotlin Multiplatform

**Architecture:** Compose Multiplatform for shared signal analysis UI; LWJGL/OpenGL for waterfall rendering with custom shaders; JNI to Rust DSP core for FFT, filtering, and demodulation; JNI to SoapySDR for hardware access; Kotlin coroutines for async sample processing pipeline and UI updates

**Tech stack:** Kotlin + Compose Multiplatform, LWJGL for OpenGL waterfall rendering, JNI to Rust DSP core, JNI to SoapySDR, SQLDelight for signal database, kotlinx.coroutines for concurrency

**Native module needs:** JNI to Rust DSP engine, JNI to SoapySDR for device access, LWJGL for OpenGL context management, PortAudio via JNI for audio I/O

**Bundle size:** ~120-170 MB

**Memory:** ~300-480 MB

**Pros:**
- JVM provides access to Apache Commons Math for statistical signal analysis, spectral estimation, and correlation functions
- Kotlin coroutines provide structured concurrency for managing the capture-process-display pipeline with cancellation support
- Cross-platform desktop with shared UI code and hardware abstraction layer across Windows/macOS/Linux
- Strong typing prevents configuration errors in complex DSP parameter chains and filter coefficient specifications

**Cons:**
- JVM introduces unacceptable latency in the sample processing pipeline — GC pauses cause sample drops at high sample rates
- JNI boundary between Kotlin and Rust DSP core adds microseconds of latency per buffer transfer — multiplied across thousands of buffers per second this becomes significant
- No GPU compute access from JVM — all FFT processing must traverse the JNI boundary to native Rust/wgpu code
- JVM memory overhead (200+ MB) consumes RAM that should be available for large IQ recording ring buffers and analysis workspaces

---

## Comparison Matrix

| Dimension | Electron | Tauri | Flutter | Swift | Kotlin MP |
|-----------|----------|-------|---------|-------|-----------|
| Dev effort (weeks) | 14 | 16 | 20 | 14 | 22 |
| Bundle size | 290 MB | 50 MB | 95 MB | 35 MB | 145 MB |
| Memory usage | 520 MB | 160 MB | 310 MB | 130 MB | 380 MB |
| Startup time | 3.0s | 0.7s | 1.8s | 0.4s | 2.4s |
| Native feel | 5/10 | 7/10 | 6/10 | 10/10 | 6/10 |
| Offline capability | 8/10 | 10/10 | 8/10 | 9/10 | 8/10 |
| GPU/compute access | 3/10 | 9/10 | 4/10 | 10/10 | 3/10 |
| Cross-platform | 9/10 | 9/10 | 8/10 | 2/10 | 7/10 |
| Best fit score | 5/10 | 9/10 | 5/10 | 7/10 | 4/10 |

---

## Recommended Framework

**Tauri** is the best fit for SpectrumLab.

**Rationale:** Signal processing demands an unbroken zero-copy pipeline from hardware capture through FFT computation to display rendering — any IPC boundary or serialization step introduces latency that causes visible artifacts in the waterfall and audible glitches in demodulated audio. Tauri's Rust-native architecture keeps the entire pipeline in a single process with wgpu compute shaders for FFT sharing the same GPU context as the waterfall renderer, meaning the FFT output buffer directly sources the display texture with no data copy. Direct SoapySDR integration via Rust FFI provides hardware access without sandbox complications, and the minimal memory footprint leaves maximum RAM for IQ sample ring buffers during sustained high-rate capture.

**Runner-up:** Swift/SwiftUI — Core Audio and Metal compute provide the absolute lowest-latency signal processing on macOS with Apple Silicon unified memory, but the platform restriction eliminates Linux users who form the core of the SDR community and depend on GNU Radio ecosystem compatibility.

---

## Monetization (Desktop)

- **Free tier:** Single SDR device support, real-time spectrum display up to 2 MHz bandwidth, basic AM/FM demodulation, 30-minute recording limit per session
- **Pro license ($39/month):** Unlimited bandwidth up to hardware maximum, all demodulation modes (PSK, QAM, OFDM), unlimited recording duration, signal classification via ML, custom FIR/IIR filter design tool
- **Enterprise license ($149/seat/month):** Multi-device synchronization, automated signal detection and alerting workflows, REST API for integration with SIGINT/COMINT platforms, audit logging for classified environments
- **Hardware bundles:** Pre-configured SDR hardware kits (RTL-SDR Blog V4, HackRF One) with SpectrumLab Pro license at 20% bundle discount
- **Academic license:** Free for university research and amateur radio operators (verified callsign) with institutional or FCC verification; includes all Pro features

---

## MVP Scope (Desktop)

**Core MVP (8 weeks):**

| Week | Milestone |
|------|-----------|
| 1-2 | Rust core: SoapySDR integration for RTL-SDR and HackRF device enumeration, frequency tuning, gain control, and IQ sample streaming at up to 20 Msps; lock-free ring buffer for zero-copy sample flow between capture and processing threads; basic CPU FFT via rustfft crate for spectral computation; SQLite schema for recordings and device configurations |
| 3-4 | wgpu waterfall renderer: real-time spectrogram display with configurable color maps (viridis, plasma, magma, inferno), adjustable time span and frequency zoom; spectrum line plot overlay showing instantaneous and peak-hold traces; basic Tauri shell with device selection dropdown, center frequency input, gain slider, and sample rate selector |
| 5-6 | GPU FFT: wgpu compute shader implementation for radix-2 FFT up to 1M points with Hann/Blackman-Harris windowing; FM/AM demodulation engine with audio output via cpal at configurable sample rates; IQ recording to SigMF format with full metadata (center freq, sample rate, hardware, timestamp); recording playback with timeline scrubbing |
| 7 | Signal processing chain: configurable FIR bandpass filter with Parks-McClellan optimal design, decimation stage with selectable ratio, AGC with configurable attack/release; processing block parameter UI with real-time spectrum preview; spectrogram annotation and bookmarking with export |
| 8 | Polish and integration: multi-device support with device-specific gain/offset calibration, waterfall rendering performance optimization for sustained 60fps at full bandwidth, keyboard shortcuts for frequency tuning and gain adjustment, cross-platform testing (Windows/macOS/Linux) and installer packaging |

**Post-MVP:**

- Advanced demodulation: PSK31/63, QAM-16/64/256, OFDM with constellation diagram display, eye diagram, and BER measurement
- Signal classification using local ONNX models trained on modulation recognition datasets (AMC-based) with confidence reporting
- Multi-device coherent capture for direction finding (DF) and beamforming applications with calibrated phase alignment
- GNU Radio flowgraph import (.grc) for compatibility with existing signal processing workflows and block reuse
- Automated signal detection with configurable CFAR energy threshold, frequency mask, and desktop notification alerts
- Wideband channelizer for simultaneously monitoring and demodulating multiple signals across a wide instantaneous bandwidth
- IQ sample export to MATLAB (.mat), NumPy (.npy), and GNU Radio (.fc32) formats for external analysis integration
- USRP (Ettus), Airspy Pro, and LimeSDR hardware support with full gain chain control, filter bandwidth selection, and GPIO triggering
- Time difference of arrival (TDOA) geolocation using multiple synchronized SDR receivers with map overlay display
- Digital voice decoder module: DMR, P25, NXDN, D-STAR, and System Fusion with call metadata display and audio playback
- Frequency allocation database with ITU region-specific band plans, license information, and known emitter catalogs for signal identification
- RF propagation modeling with terrain-aware path loss prediction for coverage mapping and link budget analysis
- Plugin API for third-party signal processing block development with hot-loading and sandboxed execution
- ADS-B aircraft tracking decoder with real-time map overlay and flight data table for aviation monitoring applications
