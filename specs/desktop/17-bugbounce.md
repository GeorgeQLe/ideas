# BugBounce Desktop — 5-Framework Comparison

## Desktop Rationale

BugBounce (visual bug reporting with screenshots and recordings) is a natural desktop application:

- **System-wide screenshot capture:** Capture any window, any region, or the full screen from any application — not limited to browser content. Native screenshot APIs provide pixel-perfect, high-DPI captures.
- **Screen recording with annotation:** Record screen activity with system audio and microphone narration. Draw arrows, highlights, and text annotations in real-time during recording.
- **Global hotkey bug capture:** Press a system-wide shortcut (Cmd+Shift+B) to instantly start a bug report from anywhere — no need to switch to the bug tracker first.
- **Local attachment storage:** Store screenshots, recordings, and crash logs locally before uploading. No file size limits during capture; compress and optimize before submission.
- **Native clipboard integration:** Paste screenshots directly from clipboard into bug reports. Copy bug report links back to clipboard for quick sharing.

---

## Desktop-Specific Features

- **Global hotkey (Cmd+Shift+B):** Instant bug capture — screenshot current screen, open annotation overlay, start report
- **Region screenshot:** Click-and-drag to capture a specific screen region with crosshair cursor
- **Screen recording:** Record screen with system audio + microphone narration, configurable frame rate and quality
- **Annotation overlay:** Draw arrows, rectangles, circles, text labels, blur regions, and highlight areas on screenshots
- **System info auto-capture:** Automatically attach OS version, display resolution, active app name, memory usage, and network status
- **Crash log collector:** Monitor system crash logs and offer to attach relevant logs to bug reports
- **Window-specific capture:** Capture a specific window (with or without shadow) by clicking on it
- **Local report queue:** Store bug reports locally when offline; submit with attachments when connection is restored
- **Clipboard paste:** Paste images from clipboard directly into the annotation editor
- **Recording trimmer:** Trim start/end of screen recordings before submission without external video editors

---

## Shared Packages

### core (Rust)

- **Screenshot engine:** Platform-native screen capture (CGWindowListCreateImage on macOS, DXGI on Windows, X11/Wayland on Linux)
- **Annotation renderer:** Vector-based annotation layer — arrows, shapes, text, blur, highlight with undo/redo stack and layer management
- **Recording encoder:** H.264/H.265 encoding via platform codecs (VideoToolbox on macOS, Media Foundation on Windows)
- **Image optimizer:** PNG/WebP compression with configurable quality; smart cropping to remove blank regions
- **System info collector:** Gather OS version, display info, active app, memory stats, network connectivity, and browser console logs
- **Report builder:** Assemble bug report with metadata, attachments, annotations, and system info into structured format
- **GIF generator:** Convert short screen recordings to animated GIFs for lightweight inline bug previews

### api-client (Rust)

- **Bug tracker integrations:** Submit reports to Jira, Linear, GitHub Issues, Asana, Trello via their REST APIs
- **Cloud upload:** Chunked upload of screenshots and recordings with progress tracking and retry
- **Auth:** OAuth2 for bug tracker connections; API key for BugBounce cloud; credential keychain storage
- **Team sync:** Sync report templates, custom fields, and project configurations across team members
- **Offline queue:** Queue submissions with full attachments; upload with deduplication on reconnect

### data (SQLite)

- **Schema:** reports (id, title, description, status, project_id, created), attachments (id, report_id, type, file_path, size), annotations, projects, templates, settings, submission_queue
- **Local-first:** All reports and attachments stored locally; submission to bug trackers is an explicit action
- **File management:** Track attachment file paths with checksums for integrity verification
- **FTS5 index:** Full-text search across report titles, descriptions, and annotation text labels

---

## Framework Implementations

### Electron

**Architecture:** Main process handles screenshot capture (desktopCapturer + native module fallback), screen recording (desktopCapturer MediaStream), system tray, and global shortcuts. Renderer (React) provides the annotation canvas (Fabric.js/Konva), report editor, and project dashboard.

**Tech stack:**
- Renderer: React 18, Fabric.js (annotation canvas), TailwindCSS, Zustand, react-hook-form (report forms)
- Main: desktopCapturer (screenshots + recording), globalShortcut, better-sqlite3, electron-store
- Media: desktopCapturer MediaRecorder for screen recording, ffmpeg for post-processing
- Database: better-sqlite3 with Drizzle ORM

**Native module needs:** better-sqlite3, screenshot-desktop (fallback for cross-monitor capture), node-global-key-listener

**Bundle size:** ~180-250 MB (Chromium + Fabric.js + ffmpeg binary)

**Memory:** ~280-450 MB (higher during screen recording)

**Pros:**
- desktopCapturer provides built-in screen capture and recording without external dependencies
- Fabric.js/Konva offer the most feature-rich HTML5 canvas annotation with drag, resize, and layering
- MediaRecorder API makes screen recording straightforward with configurable quality
- Full web ecosystem for building the bug report form and project management UI

**Cons:**
- desktopCapturer screenshot quality and reliability varies across platforms and display configurations
- Screen recording via MediaRecorder produces larger files than hardware-accelerated encoders
- Heavy memory usage during recording (Chromium + MediaRecorder + annotation canvas active simultaneously)
- Annotation canvas rendering can be sluggish on high-DPI displays with many annotations
- Global hotkey conflicts with other Electron apps using the same shortcut system

### Tauri

**Architecture:** Rust backend handles native screenshot capture (platform-specific APIs), screen recording (via ffmpeg or platform codecs), global shortcuts, and system info collection. Svelte frontend for annotation overlay (Canvas API), report editor, and dashboard.

**Tech stack:**
- Frontend: Svelte 5, custom Canvas annotation layer, TailwindCSS
- Backend: xcap (cross-platform screenshot), scrap (screen capture), rusqlite, tokio
- Recording: ffmpeg-next for encoding, or platform-specific (VideoToolbox bindings on macOS)
- Image: image-rs for PNG/WebP processing

**Plugin needs:** tauri-plugin-global-shortcut, tauri-plugin-dialog, tauri-plugin-notification, tauri-plugin-fs, tauri-plugin-clipboard-manager

**Bundle size:** ~10-20 MB (+ ffmpeg ~40 MB or use system ffmpeg)

**Memory:** ~60-130 MB (spikes during recording)

**Pros:**
- xcap/scrap provide direct access to native screenshot APIs — most reliable cross-platform capture
- Rust image processing (image-rs) is fast for compression and optimization of captured screenshots
- Small bundle and low idle memory — suitable for always-running tray app waiting for bug reports
- Platform-native screen recording encoders (VideoToolbox) produce smaller, higher-quality video than MediaRecorder

**Cons:**
- Canvas annotation in webview requires building custom tools (arrows, shapes, blur) from scratch — no Fabric.js equivalent
- Annotation layer is significant frontend development effort (~2-3 weeks)
- Screen recording setup with ffmpeg-next bindings is complex cross-platform
- Webview overlay for annotation may not feel as responsive as native drawing tools

### Flutter Desktop

**Architecture:** CustomPainter for annotation overlay. Platform channels for native screenshot capture and screen recording. Recording preview with video_player.

**Tech stack:**
- UI: Flutter 3.x, CustomPainter (annotations), Riverpod, video_player
- Database: drift
- Screenshot: platform channels → native APIs (CGWindowListCreateImage, DXGI)
- Recording: flutter_rust_bridge → ffmpeg-next

**Plugin needs:** window_manager, tray_manager, hotkey_manager, desktop_drop, screen_capturer

**Bundle size:** ~28-40 MB

**Memory:** ~130-220 MB (during recording and annotation)

**Pros:**
- CustomPainter annotation layer can be smooth and responsive with proper optimization
- Gesture handling (GestureDetector) maps naturally to draw-on-screenshot annotation workflows
- Single codebase extends to mobile for capturing bugs on mobile apps directly
- Hot reload accelerates iteration on the annotation tool UX

**Cons:**
- Screenshot capture requires platform-specific implementations through channels — no unified API
- Screen recording on desktop via Flutter is immature and unreliable across platforms
- Video playback (recording preview/trimming) has inconsistencies across desktop platforms
- Global hotkey support (hotkey_manager) is less reliable than native implementations
- Annotation layer via CustomPainter lacks built-in interactive resize handles and hit-testing

### Swift/SwiftUI (macOS)

**Architecture:** ScreenCaptureKit for screenshots and recording. NSBezierPath + CALayer for annotation overlay. NSEvent for global hotkeys. AVFoundation for recording encoding.

**Tech stack:**
- UI: SwiftUI, ScreenCaptureKit, PencilKit (or custom NSView annotation), AVFoundation
- Data: SwiftData
- Recording: ScreenCaptureKit + AVAssetWriter (hardware-accelerated H.265)
- Hotkey: Carbon API (RegisterEventHotKey) or MASShortcut
- System info: ProcessInfo, NSScreen, NSWorkspace

**Bundle size:** ~6-12 MB

**Memory:** ~35-70 MB (spikes to ~120 MB during recording)

**Pros:**
- **ScreenCaptureKit is the best screenshot/recording API on any platform** — pixel-perfect, hardware-accelerated, minimal CPU usage
- AVAssetWriter produces the smallest, highest-quality recordings (H.265 via VideoToolbox)
- PencilKit provides polished annotation tools out of the box (pen, marker, shapes, text)
- Native global hotkey via Carbon API is the most reliable on macOS
- Smallest recording file sizes thanks to hardware H.265 encoding
- System info collection via native frameworks is comprehensive and privacy-respecting

**Cons:**
- macOS only — bug reporting tools must support Windows and Linux for full team coverage
- ScreenCaptureKit requires macOS 12.3+ (excludes older macOS versions still in enterprise use)
- PencilKit is primarily designed for Apple Pencil; adapting for mouse annotation requires customization
- No web code reuse for the report editor and project management UI
- Limited bug tracker SDK availability in Swift compared to JavaScript ecosystem

### Kotlin Multiplatform

**Architecture:** Compose Multiplatform for UI. AWT Robot for screenshot capture. Xuggler/JavaCV for screen recording. Custom Compose Canvas for annotations.

**Tech stack:**
- UI: Compose Desktop, Material 3, custom Canvas annotations
- Database: SQLDelight
- Screenshot: java.awt.Robot (screen capture), JNA for native API access
- Recording: JavaCV (FFmpeg wrapper) or Monte Media Library
- HTTP: Ktor client

**Bundle size:** ~55-80 MB (+JRE)

**Memory:** ~220-380 MB (JVM + recording buffer)

**Pros:**
- java.awt.Robot provides basic but reliable cross-platform screenshot capture
- JavaCV wraps FFmpeg comprehensively for recording with configurable codecs
- Could share bug report logic and form validation with an Android companion app
- JNA allows calling native screenshot/recording APIs when AWT Robot is insufficient

**Cons:**
- java.awt.Robot captures are low-quality — no per-window capture, no high-DPI awareness
- JVM memory overhead is excessive for a utility that should feel lightweight and instant
- Compose Canvas annotation tools are basic — no built-in shapes, arrows, or blur
- Global hotkey via JNativeHook has latency compared to native implementations
- Screen recording via JavaCV is CPU-intensive on the JVM

---

## Comparison Matrix

| Dimension | Electron | Tauri | Flutter | Swift | Kotlin MP |
|-----------|----------|-------|---------|-------|-----------|
| Dev effort (weeks) | 7 | 9 | 10 | 6 | 10 |
| Bundle size | 220 MB | 16 MB | 34 MB | 9 MB | 70 MB |
| Memory usage | 360 MB | 95 MB | 175 MB | 50 MB | 300 MB |
| Startup time | 2.5s | 0.7s | 1.2s | 0.3s | 2.2s |
| Native feel | 5/10 | 7/10 | 5/10 | 10/10 | 4/10 |
| Offline capability | 8/10 | 9/10 | 7/10 | 9/10 | 7/10 |
| Screenshot/recording | 7/10 | 8/10 | 5/10 | 10/10 | 4/10 |
| Cross-platform | 9/10 | 9/10 | 9/10 | 2/10 | 8/10 |
| Best fit score | 7/10 | 8/10 | 5/10 | 8/10 | 4/10 |

---

## Recommended Framework

**Tauri** is the best fit for BugBounce Desktop, with **Electron** as a close alternative.

**Rationale:** BugBounce needs reliable cross-platform screenshot and recording capabilities combined with a lightweight always-running tray presence. Tauri's Rust backend (xcap/scrap for capture, image-rs for processing) provides the most reliable native screenshot access across macOS, Windows, and Linux. The annotation canvas is the primary frontend investment, but a well-built Canvas API implementation in Svelte is sufficient. Tauri's small footprint is ideal for a tool that sits in the system tray waiting to capture bugs at any moment.

**Runner-up:** Electron if the annotation experience is the top priority — Fabric.js provides the richest canvas annotation toolkit out of the box, saving 2-3 weeks of custom annotation development. Also consider Swift/SwiftUI for a Mac-only premium version leveraging ScreenCaptureKit.

---

## Monetization (Desktop)

- **Free tier:** Screenshot capture, basic annotations (arrows, rectangles), up to 50 reports, local storage only
- **Pro ($14/mo or $119/yr):** Screen recording, advanced annotations (blur, highlight, text), unlimited reports, bug tracker integration (Jira, Linear, GitHub Issues)
- **Team ($8/user/mo):** Shared projects, report templates, custom fields, team dashboard, Slack/Teams notifications
- **Enterprise ($19/user/mo):** SSO, audit log, custom workflows, self-hosted option, priority support
- **Distribution:** Direct download for all platforms; Mac App Store for discoverability; Homebrew cask for developers

---

## MVP Scope (Desktop)

**Core MVP (8 weeks):**

| Week | Milestone |
|------|-----------|
| 1-2 | Screenshot capture (full screen, region, window), basic annotation tools (arrow, rectangle, text), local storage |
| 3-4 | Global hotkey activation, annotation overlay with undo/redo, auto-capture system info (OS, resolution, app) |
| 5-6 | Bug report form editor, project management, submission to Jira and GitHub Issues via API |
| 7 | Screen recording with start/stop controls, recording trimmer, system tray with quick-capture menu |
| 8 | Auto-update, packaging (DMG/MSI/AppImage), offline queue, onboarding tutorial, beta testing |

**Post-MVP:** Advanced annotations (blur, numbered steps, GIF export), more bug tracker integrations (Linear, Asana, Trello), team collaboration, crash log collection, browser extension companion
