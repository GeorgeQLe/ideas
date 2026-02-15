# Desktop Framework Comparison Guide

A comprehensive reference for evaluating **Electron**, **Tauri**, **Flutter Desktop**, **Swift/SwiftUI**, and **Kotlin Multiplatform** for building desktop versions of our 40 SaaS products.

---

## Table of Contents

1. [Framework Overviews](#framework-overviews)
2. [Architecture Patterns](#architecture-patterns)
3. [Performance Benchmarks](#performance-benchmarks)
4. [Developer Experience](#developer-experience)
5. [Native API Access](#native-api-access)
6. [Distribution & Auto-Update](#distribution--auto-update)
7. [Shared Packages Strategy](#shared-packages-strategy)
8. [Security Model](#security-model)
9. [Decision Matrix](#decision-matrix)
10. [When to Pick Which Framework](#when-to-pick-which-framework)

---

## Framework Overviews

### Electron

**Architecture:** Chromium + Node.js. Each app bundles a full browser engine (main process) with a renderer process for each window. IPC via `ipcMain`/`ipcRenderer`.

**Language:** JavaScript/TypeScript (renderer + main), with optional native modules via N-API.

**Cross-platform:** Windows, macOS, Linux.

**Maturity:** 2013+. Used by VS Code, Slack, Discord, Figma, Notion, 1Password 8.

**Key traits:**
- Largest ecosystem and community of any desktop framework
- Full web API compatibility — can reuse SaaS frontend code directly
- Heavyweight: ~150-300 MB bundle, ~200-500 MB RAM baseline
- Full Node.js access in main process (file system, networking, child processes)
- Extensive native module ecosystem (better-sqlite3, node-pty, sharp, etc.)

### Tauri

**Architecture:** System webview (WebKit on macOS, WebView2 on Windows, WebKitGTK on Linux) + Rust backend. Frontend communicates with Rust via `invoke` commands (JSON IPC).

**Language:** Rust (backend/core), JavaScript/TypeScript (frontend). Frontend can use any web framework (React, Svelte, Vue, Solid).

**Cross-platform:** Windows, macOS, Linux. Mobile (iOS/Android) in beta (Tauri v2).

**Maturity:** 2020+ (v1 stable 2022, v2 stable 2024). Used by Cody (Sourcegraph), Aptakube, Padloc.

**Key traits:**
- Ultra-lightweight: ~3-10 MB bundle, ~30-80 MB RAM
- Security-first: allowlist model, CSP enforcement, no Node.js in renderer
- Rust backend gives near-native performance for compute-heavy tasks
- Plugin system for common native APIs (fs, shell, clipboard, dialog, notification)
- Smaller ecosystem than Electron but growing rapidly
- System webview means no Chromium bundling, but behavior varies across OS versions

### Flutter Desktop

**Architecture:** Dart VM + Skia/Impeller rendering engine. Flutter owns the entire pixel pipeline — no native widgets, everything is custom-rendered.

**Language:** Dart (all layers). Platform channels (FFI) for native code.

**Cross-platform:** Windows, macOS, Linux, iOS, Android, Web.

**Maturity:** Desktop stable since Flutter 3.0 (2022). Used by Google Earth, Canonical's Ubuntu installer, Ente Photos.

**Key traits:**
- True cross-platform: single codebase for desktop + mobile + web
- Custom rendering means pixel-perfect consistency across platforms
- 120fps capable with Impeller rendering engine
- Excellent hot reload (sub-second UI iteration)
- Desktop-specific widgets still maturing (data tables, menus, keyboard shortcuts)
- Dart package ecosystem strong for mobile, weaker for desktop-specific needs
- ~20-40 MB bundle, ~100-200 MB RAM

### Swift/SwiftUI (macOS)

**Architecture:** Native AppKit/SwiftUI framework. Direct access to all macOS APIs, frameworks, and hardware.

**Language:** Swift. Can bridge to Objective-C and C/C++.

**Cross-platform:** macOS only (iOS/iPadOS with shared SwiftUI code, but not Windows/Linux).

**Maturity:** Swift 2014+, SwiftUI 2019+. All Apple first-party apps.

**Key traits:**
- Best possible Mac experience: native look, feel, and performance
- Deep OS integration (Spotlight, Shortcuts, Share Extensions, Widgets, Menu Bar)
- Smallest bundle size (~5-15 MB) and lowest memory footprint (~30-60 MB)
- SwiftData/CoreData for local persistence with iCloud sync built-in
- Mac App Store distribution with built-in notarization
- No cross-platform: macOS only (can share logic with iOS via Swift packages)
- Xcode-only development environment

### Kotlin Multiplatform (KMP)

**Architecture:** Shared Kotlin code compiled to JVM (desktop), native (iOS), or JS (web). Desktop UI via Compose Multiplatform (Jetpack Compose for Desktop).

**Language:** Kotlin (all layers). Can interop with Java libraries and native code via JNI.

**Cross-platform:** Windows, macOS, Linux (via JVM). iOS and Android (via Kotlin Native/JVM).

**Maturity:** KMP stable 2023. Compose Multiplatform desktop stable 2024. Used by JetBrains Fleet, Toolbox App.

**Key traits:**
- Leverages massive JVM/Android ecosystem
- Compose Multiplatform provides declarative UI (similar to SwiftUI/Flutter)
- Shared business logic between Android and desktop
- JVM runtime means ~30-50 MB bundle + JRE, ~150-300 MB RAM
- Strong typing, coroutines, and flow for reactive programming
- SQLDelight for cross-platform type-safe SQL
- Smaller desktop community compared to Android side

---

## Architecture Patterns

### Process Model

| Framework | Architecture | Process Model |
|-----------|-------------|---------------|
| Electron | Multi-process | Main process + renderer per window (Chromium model) |
| Tauri | Hybrid | Rust core (single process) + webview (OS-managed) |
| Flutter | Single process | Dart isolates for parallelism, platform channels for native |
| Swift | Single process | Grand Central Dispatch (GCD) for concurrency, async/await |
| Kotlin MP | Single process | JVM threads + Kotlin coroutines, Compose on main thread |

### Data Flow

**Electron:**
```
Renderer (React/Vue) → IPC → Main Process (Node.js) → Native Modules / SQLite / API
```

**Tauri:**
```
Frontend (React/Svelte) → invoke() → Rust Commands → SQLite / API / System APIs
```

**Flutter:**
```
Widget Tree → State Management (Riverpod/Bloc) → Repository → Platform Channels → Native / SQLite
```

**Swift/SwiftUI:**
```
SwiftUI Views → @Observable Models → Services → SwiftData/CoreData / URLSession
```

**Kotlin MP:**
```
Compose UI → ViewModel → Use Cases → Repository → SQLDelight / Ktor Client
```

### State Management

| Framework | Recommended Pattern | Alternatives |
|-----------|-------------------|--------------|
| Electron | Redux/Zustand (renderer), electron-store (main) | MobX, Jotai, Recoil |
| Tauri | Zustand/Jotai (frontend), Rust state (backend) | Redux, Svelte stores |
| Flutter | Riverpod or Bloc | Provider, GetX, MobX |
| Swift | @Observable + SwiftData | Combine, TCA (The Composable Architecture) |
| Kotlin MP | ViewModel + StateFlow | MVI with Orbit, Redux-like |

---

## Performance Benchmarks

Benchmarks based on a typical CRUD app with local database, list views, and real-time sync. Values are approximate ranges across macOS/Windows.

### Startup Time (Cold Start)

| Framework | Time to First Paint | Time to Interactive |
|-----------|-------------------|-------------------|
| Electron | 1.5-3.0s | 2.0-4.0s |
| Tauri | 0.5-1.0s | 0.8-1.5s |
| Flutter | 0.8-1.5s | 1.0-2.0s |
| Swift | 0.2-0.5s | 0.3-0.6s |
| Kotlin MP | 1.0-2.0s | 1.5-2.5s |

### Memory Usage (Idle / Active)

| Framework | Idle (1 window) | Active (heavy use) | With large dataset |
|-----------|----------------|-------------------|-------------------|
| Electron | 200-350 MB | 400-800 MB | 600 MB-1.2 GB |
| Tauri | 30-80 MB | 80-200 MB | 150-400 MB |
| Flutter | 80-150 MB | 150-350 MB | 250-500 MB |
| Swift | 25-60 MB | 60-150 MB | 100-300 MB |
| Kotlin MP | 150-250 MB | 250-500 MB | 400-700 MB |

### Bundle Size (Installed)

| Framework | macOS (.app) | Windows (.exe + deps) | Linux (AppImage) |
|-----------|-------------|----------------------|-----------------|
| Electron | 150-300 MB | 120-250 MB | 150-300 MB |
| Tauri | 3-10 MB | 2-8 MB | 5-15 MB |
| Flutter | 20-40 MB | 20-40 MB | 25-50 MB |
| Swift | 5-15 MB | N/A | N/A |
| Kotlin MP | 40-80 MB | 50-100 MB (+JRE) | 60-120 MB (+JRE) |

### Rendering Performance (60fps Target)

| Framework | List Scrolling (10K items) | Animation (complex) | Canvas/GPU |
|-----------|--------------------------|--------------------|-----------|
| Electron | Good (virtualized) | Good (CSS/Canvas) | Excellent (WebGL/WebGPU) |
| Tauri | Good (virtualized) | Good (CSS/Canvas) | Excellent (WebGL/WebGPU) |
| Flutter | Excellent (Impeller) | Excellent (native) | Good (CustomPainter) |
| Swift | Excellent (native) | Excellent (Core Animation) | Excellent (Metal) |
| Kotlin MP | Good (Compose lazy lists) | Good (Compose animation) | Fair (Skia/AWT) |

---

## Developer Experience

### Tooling Comparison

| Aspect | Electron | Tauri | Flutter | Swift | Kotlin MP |
|--------|----------|-------|---------|-------|-----------|
| IDE | VS Code, WebStorm | VS Code, any editor | VS Code, Android Studio | Xcode (required) | IntelliJ, Android Studio |
| Hot Reload | Webpack HMR / Vite HMR | Vite HMR (frontend only) | Sub-second hot reload (full) | Xcode Previews (partial) | Compose Preview (partial) |
| Debugging | Chrome DevTools + Node.js debugger | Chrome DevTools + lldb (Rust) | Dart DevTools (excellent) | Xcode Debugger + Instruments | IntelliJ Debugger |
| Profiling | Chrome Performance tab | Chrome DevTools + Rust flamegraphs | Dart DevTools Performance | Instruments (Time Profiler, Allocations) | JVM profilers (VisualVM, YourKit) |
| Testing | Jest/Vitest + Playwright/Spectron | Vitest (frontend) + cargo test (backend) | flutter_test + integration_test | XCTest + XCUITest | JUnit + Compose UI testing |
| CI/CD | GitHub Actions (easy) | GitHub Actions (Rust builds slower) | GitHub Actions (needs Flutter SDK) | Xcode Cloud, GitHub Actions (macOS only) | GitHub Actions (JVM) |
| Learning Curve | Low (web devs) | Medium (Rust + web) | Medium (Dart) | Medium-High (Apple ecosystem) | Medium (Kotlin, Compose) |

### Package Ecosystem Size

| Framework | Desktop-relevant packages | Package manager |
|-----------|--------------------------|----------------|
| Electron | ~50K+ (npm) | npm/yarn/pnpm |
| Tauri | ~1K plugins + crates.io | Cargo + npm |
| Flutter | ~5K desktop-compatible | pub.dev |
| Swift | ~3K (Swift packages) | SPM |
| Kotlin MP | ~2K multiplatform | Maven Central / Gradle |

### Type Safety

| Framework | Type System | Null Safety |
|-----------|------------|-------------|
| Electron | TypeScript (optional) | Optional chaining |
| Tauri | TypeScript (frontend) + Rust (backend, strict) | Rust: zero null, TS: strict mode |
| Flutter | Dart (sound null safety) | Built-in since Dart 2.12 |
| Swift | Swift (strong, inferred) | Optionals (compile-time) |
| Kotlin MP | Kotlin (strong, inferred) | Built-in null safety |

---

## Native API Access

### System Integration Matrix

| Capability | Electron | Tauri | Flutter | Swift | Kotlin MP |
|-----------|----------|-------|---------|-------|-----------|
| File System | ✅ Full (Node fs) | ✅ Full (Rust std::fs, scoped) | ⚠️ Via plugins | ✅ Full (Foundation) | ⚠️ Via JVM (java.io) |
| System Tray | ✅ Built-in | ✅ Built-in | ⚠️ Via tray_manager plugin | ✅ NSStatusItem | ⚠️ Via AWT SystemTray |
| Notifications | ✅ HTML5 Notification + native | ✅ Built-in plugin | ⚠️ Via plugin | ✅ UNUserNotificationCenter | ⚠️ Via JVM |
| Clipboard | ✅ Built-in | ✅ Built-in plugin | ⚠️ Via plugin | ✅ NSPasteboard | ✅ Via AWT |
| Global Shortcuts | ✅ globalShortcut | ✅ global-shortcut plugin | ⚠️ Limited | ✅ NSEvent.addGlobalMonitorForEvents | ⚠️ Via JNativeHook |
| Drag & Drop | ✅ HTML5 DnD | ✅ HTML5 DnD | ⚠️ Platform-specific | ✅ NSView dragging | ⚠️ Via AWT DnD |
| Menu Bar | ✅ Menu module | ✅ Built-in | ⚠️ Via plugin | ✅ NSMenu (native) | ⚠️ Via AWT/Compose |
| File Associations | ✅ Via app manifest | ✅ Via tauri.conf.json | ⚠️ Platform-specific | ✅ Info.plist UTType | ⚠️ Via JPackage |
| Deep Links | ✅ Via protocol handler | ✅ deep-link plugin | ⚠️ Via plugin | ✅ NSAppleEventManager | ⚠️ Via JVM |
| Touch Bar (Mac) | ✅ TouchBar API | ❌ Not supported | ❌ Not supported | ✅ NSTouchBar | ❌ Not supported |
| GPU / Metal / CUDA | ✅ WebGL/WebGPU | ✅ WebGL/WebGPU + Rust GPU libs | ⚠️ Via FFI | ✅ Metal (direct) | ⚠️ Via JNI/LWJGL |
| Camera / Mic | ✅ WebRTC | ⚠️ Via plugins | ⚠️ Via plugins | ✅ AVFoundation | ⚠️ Via JVM |
| Local Network | ✅ Full (Node net) | ✅ Full (Rust std::net) | ⚠️ Via dart:io | ✅ Network.framework | ✅ Via java.net |
| Keychain / Credential Store | ⚠️ Via keytar | ✅ Via keyring plugin | ⚠️ Via plugin | ✅ Keychain Services | ⚠️ Via java.security |
| Auto-Launch | ✅ Via auto-launch module | ✅ autostart plugin | ⚠️ Via plugin | ✅ SMLoginItemSetEnabled | ⚠️ Via JVM |
| Accessibility | ✅ Chromium a11y | ⚠️ System webview a11y | ✅ Flutter semantics | ✅ NSAccessibility | ⚠️ Via Java Accessibility |

Legend: ✅ = Built-in / first-party, ⚠️ = Via third-party plugin or manual implementation, ❌ = Not supported

### GPU & Compute Access (Critical for Deep Tech Products)

For products 21-40 (deep tech), GPU and heavy compute access is a key differentiator:

| Framework | GPU Access | CUDA/OpenCL | ML Inference | Compute Strategy |
|-----------|-----------|-------------|-------------|-----------------|
| Electron | WebGPU, WebGL | Via native modules (node-cuda) | ONNX.js, TensorFlow.js | Offload to native addon |
| Tauri | WebGPU, WebGL | Via Rust crates (cuda-rs, opencl3) | Via ort (ONNX Runtime Rust) | Native Rust — no bridge overhead |
| Flutter | Via FFI to native | Via FFI to C/CUDA | Via tflite_flutter | Platform channels to native |
| Swift | Metal (first-class) | Via Metal Performance Shaders | Core ML, MLX | Direct hardware access — best on Mac |
| Kotlin MP | Via JNI/LWJGL | Via JNI (jcuda) | DJL, ONNX Runtime Java | JVM overhead for GPU calls |

---

## Distribution & Auto-Update

### Packaging

| Framework | macOS | Windows | Linux |
|-----------|-------|---------|-------|
| Electron | DMG, pkg (electron-builder) | NSIS, MSI, Squirrel | AppImage, deb, rpm, snap |
| Tauri | DMG, app bundle (built-in) | NSIS, MSI (built-in) | AppImage, deb (built-in) |
| Flutter | DMG (manual or flutter_distributor) | MSIX (manual) | Snap, AppImage (manual) |
| Swift | DMG, pkg, Mac App Store | N/A | N/A |
| Kotlin MP | DMG, pkg (Conveyor, jpackage) | MSI, exe (Conveyor, jpackage) | deb, rpm (jpackage) |

### Auto-Update Mechanisms

| Framework | Built-in | Solution | Delta Updates |
|-----------|----------|---------|--------------|
| Electron | ✅ | electron-updater (Squirrel, NSIS) | ✅ (electron-differential-updater) |
| Tauri | ✅ | Built-in updater plugin (signature verification) | ❌ (full binary replace) |
| Flutter | ❌ | Manual (Sparkle macOS, WinSparkle) | ❌ |
| Swift | ✅ | Sparkle (open source) or Mac App Store | ✅ (App Store delta) |
| Kotlin MP | ❌ | Conveyor (auto-update built-in) or manual | ❌ |

### Code Signing

| Framework | macOS | Windows | Notarization |
|-----------|-------|---------|-------------|
| Electron | Apple Developer ID ($99/yr) | EV Code Signing cert (~$300/yr) | electron-notarize |
| Tauri | Apple Developer ID | Code Signing cert | Built-in notarization support |
| Flutter | Apple Developer ID | Code Signing cert | Manual (xcrun notarytool) |
| Swift | Apple Developer ID | N/A | Xcode built-in |
| Kotlin MP | Apple Developer ID | Code Signing cert | Manual |

---

## Shared Packages Strategy

The key insight for our A/B comparison: **business logic should be written once and shared across all 5 framework implementations**. This makes the comparison about UI/platform integration, not about rewriting the same CRUD logic 5 times.

### Architecture

```
┌─────────────────────────────────────────────────────┐
│                  Desktop Application                 │
├──────────┬──────────┬──────────┬──────┬─────────────┤
│ Electron │  Tauri   │ Flutter  │Swift │  Kotlin MP  │
│  (React) │ (Svelte) │  (Dart)  │(SUI) │  (Compose)  │
├──────────┴──────────┴──────────┴──────┴─────────────┤
│              UI & Platform Integration               │
├─────────────────────────────────────────────────────┤
│                   FFI / Bridge Layer                  │
├──────────┬──────────┬───────────────────────────────┤
│  core    │api-client│        data (SQLite)           │
│  (Rust)  │  (Rust)  │     (shared schema)            │
├──────────┴──────────┴───────────────────────────────┤
│           protocol (Protobuf / JSON Schema)          │
└─────────────────────────────────────────────────────┘
```

### Shared Package Details

#### `core` — Business Logic (Rust)

**Purpose:** Domain models, validation rules, business logic, algorithms, computations.

**Why Rust:**
- Memory-safe without garbage collection
- Compiles to native code (no runtime overhead)
- Excellent FFI story to all target languages
- Can compile to WASM for web fallback

**Binding strategy per framework:**

| Framework | Bridge | Library | Overhead |
|-----------|--------|---------|----------|
| Electron | napi-rs (N-API) | .node native module | ~1-5μs per call |
| Tauri | Direct (native Rust) | No bridge needed | Zero overhead |
| Flutter | flutter_rust_bridge (FFI) | .dylib/.so/.dll | ~1-3μs per call |
| Swift | UniFFI or swift-bridge | .xcframework | ~1-3μs per call |
| Kotlin MP | UniFFI or JNI via JNA | .dylib/.so/.dll | ~5-10μs per call |

#### `api-client` — Network Layer (Rust)

**Purpose:** HTTP client, API communication, authentication, real-time sync (WebSocket), offline queue.

**Key responsibilities:**
- Auth token management (OAuth2, JWT refresh)
- Request retry with exponential backoff
- Offline request queue with conflict resolution
- WebSocket connection management
- Response caching layer

#### `data` — Local Database (SQLite)

**Purpose:** Local-first data persistence, migrations, query interface.

**Shared schema** defined in SQL migrations — every framework runs the same migration set.

| Framework | SQLite Binding | ORM/Query Builder |
|-----------|---------------|-------------------|
| Electron | better-sqlite3 | Drizzle ORM / Kysely |
| Tauri | rusqlite (native) | sea-query or sqlx |
| Flutter | sqflite or drift | drift (type-safe) |
| Swift | GRDB or SQLite.swift | GRDB (full-featured) |
| Kotlin MP | SQLDelight | SQLDelight (type-safe, multiplatform) |

#### `protocol` — Type Definitions

**Purpose:** Shared type contracts ensuring all frameworks agree on data shapes.

**Format options:**
- **Protocol Buffers** — best for binary IPC, strict schema evolution
- **JSON Schema** — best for REST API contracts, human-readable
- **FlatBuffers** — best for zero-copy performance (deep tech products)

Code generation per framework:
- Electron/Tauri: TypeScript types from JSON Schema (via `json-schema-to-typescript`)
- Flutter: Dart classes from Protobuf (via `protoc-gen-dart`)
- Swift: Swift structs from Protobuf (via `swift-protobuf`)
- Kotlin MP: Kotlin data classes from Protobuf (via `protobuf-kotlin`)

---

## Security Model

### Threat Surface Comparison

| Framework | Attack Surface | Sandboxing | Code Signing |
|-----------|---------------|------------|-------------|
| Electron | Large (Chromium + Node.js) | Optional (contextIsolation, sandbox) | Required for distribution |
| Tauri | Small (system webview + Rust) | Built-in (allowlist, CSP, scope) | Required for distribution |
| Flutter | Medium (Dart VM + plugins) | OS-level only | Required for distribution |
| Swift | Small (native) | App Sandbox (Mac App Store required) | Required (notarization) |
| Kotlin MP | Medium (JVM) | OS-level + JVM SecurityManager | Required for distribution |

### Security Best Practices

**Electron:**
- Enable `contextIsolation: true` and `sandbox: true`
- Use `preload.js` for IPC bridge (never expose `require` to renderer)
- Enable `webSecurity: true`
- CSP headers in HTML
- Disable `nodeIntegration` in renderer

**Tauri:**
- Scope file system access (only allowed directories)
- Use allowlist for Tauri API access
- CSP headers enforced by default
- Rust backend prevents memory corruption bugs

**All frameworks:**
- Store secrets in OS keychain (never plaintext)
- Use certificate pinning for API communication
- Validate all IPC messages
- Auto-update with signature verification

---

## Decision Matrix

### Scoring Criteria (1-10)

| Criteria | Weight | Electron | Tauri | Flutter | Swift | Kotlin MP |
|----------|--------|----------|-------|---------|-------|-----------|
| Bundle size | 5% | 2 | 10 | 7 | 9 | 5 |
| Memory usage | 5% | 2 | 9 | 6 | 10 | 4 |
| Startup time | 5% | 3 | 8 | 7 | 10 | 5 |
| Dev speed (initial) | 15% | 9 | 6 | 7 | 6 | 6 |
| Dev speed (ongoing) | 10% | 8 | 7 | 8 | 7 | 7 |
| Code reuse from SaaS | 15% | 10 | 8 | 3 | 2 | 3 |
| Native feel | 10% | 4 | 5 | 6 | 10 | 6 |
| Cross-platform | 10% | 9 | 9 | 10 | 1 | 8 |
| Ecosystem / libs | 5% | 10 | 6 | 7 | 7 | 7 |
| GPU / compute | 5% | 6 | 9 | 5 | 10 | 4 |
| Security | 5% | 4 | 9 | 6 | 8 | 5 |
| Auto-update | 5% | 9 | 8 | 4 | 8 | 5 |
| Hiring pool | 5% | 10 | 4 | 6 | 5 | 7 |
| **Weighted Total** | **100%** | **7.3** | **7.2** | **6.4** | **5.9** | **5.7** |

*Note: Swift scores lower overall due to macOS-only limitation, but scores highest for Mac-only products.*

---

## When to Pick Which Framework

### Pick Electron When:
- Your SaaS is a web app and you want maximum code reuse
- You need the broadest native module ecosystem
- Your team is JavaScript/TypeScript-only
- Speed to market matters more than performance
- You're building a content-heavy or text-editor-style app (VS Code model)
- **Best for:** B2B SaaS products 01-20 (web-centric, CRUD-heavy)

### Pick Tauri When:
- Bundle size and memory usage are critical (user-facing consumer app)
- You need Rust-level performance for compute tasks
- Security is a top priority (financial, healthcare)
- You want modern web frontend + native performance backend
- Your product does significant local computation
- **Best for:** Products with local compute needs, security-sensitive products

### Pick Flutter When:
- You also need mobile apps (iOS + Android) from same codebase
- You want pixel-perfect consistent UI across all platforms
- Your product has complex custom UI (visualizations, animations, graphs)
- Your team knows Dart or is willing to learn
- **Best for:** Products with heavy custom visualization, mobile+desktop convergence

### Pick Swift/SwiftUI When:
- Mac-only is acceptable (or Mac-first with separate iOS app)
- You want the absolute best Mac experience
- Your product benefits from deep OS integration (Spotlight, Widgets, Shortcuts)
- GPU/Metal access matters (scientific visualization, ML inference)
- You're distributing via Mac App Store
- **Best for:** Deep tech products 21-40 on macOS (GPU compute, scientific viz)

### Pick Kotlin Multiplatform When:
- You have an existing Android app and want desktop + Android code sharing
- Your team is Kotlin/JVM experts
- You need JVM ecosystem libraries (Apache Commons, scientific computing)
- Enterprise/business focus where JVM is standard
- **Best for:** Products with Android synergy, enterprise deployments

### Product Category Recommendations

| Category | Primary Pick | Secondary Pick | Rationale |
|----------|-------------|---------------|-----------|
| B2B SaaS (01-20) | Electron | Tauri | Max code reuse from existing web SaaS |
| Dev Tools (01, 03, 08, 17, 20) | Tauri | Electron | Dev users appreciate small bundles |
| Data/ETL (11, 14) | Electron | Tauri | Rich data visualization needs |
| HR/Ops (02, 09) | Electron | Flutter | Standard CRUD, mobile useful |
| Deep Tech / Simulation (21-40) | Tauri | Swift (macOS) | Heavy compute, GPU access, small bundle |
| Scientific Viz (22, 30, 33, 40) | Flutter | Swift (macOS) | Complex custom rendering |
| Hardware/Circuit (21, 24, 34) | Tauri | Swift (macOS) | GPU compute, native perf critical |
| Biotech/Chem (22, 26, 29, 35) | Tauri | Flutter | Compute-heavy, cross-platform needed |

---

## Cost Comparison

### Development Cost (8-week MVP)

| Framework | FTE Engineers | Weeks | Estimated Cost |
|-----------|--------------|-------|---------------|
| Electron | 1-2 (web devs) | 6-8 | $30K-60K |
| Tauri | 1-2 (web + Rust) | 8-10 | $40K-80K |
| Flutter | 1-2 (Dart devs) | 8-10 | $40K-70K |
| Swift | 1 (iOS/Mac dev) | 8-10 | $40K-70K |
| Kotlin MP | 1-2 (Kotlin devs) | 10-12 | $50K-80K |

### Ongoing Maintenance (Annual)

| Framework | Update frequency | Breaking changes | Maintenance effort |
|-----------|-----------------|-----------------|-------------------|
| Electron | Major every 8 weeks | Moderate (Chromium updates) | Medium |
| Tauri | Major every 6-12 months | Low (stable API) | Low |
| Flutter | Major every 3 months | Low-Medium | Medium |
| Swift | Annual (with macOS) | Low (Apple backward compat) | Low |
| Kotlin MP | Quarterly | Medium (still evolving) | Medium-High |

---

## Summary

No single framework wins across all dimensions. The choice depends on:

1. **Product type** — CRUD-heavy B2B vs. compute-heavy deep tech
2. **Platform targets** — cross-platform required vs. Mac-only acceptable
3. **Team skills** — existing expertise matters more than theoretical best
4. **Performance needs** — bundle size, memory, GPU, startup time priorities
5. **Code reuse** — from existing SaaS web app or from mobile app

The per-product specs (01-40) each include a framework comparison matrix tailored to that specific product's requirements, culminating in a recommended pick with rationale.
