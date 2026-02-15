# FormForge Desktop — 5-Framework Comparison

## Desktop Rationale

FormForge (natural language to form builder) gains significant advantages as a desktop application:

- **Offline form design:** Build complex forms without an internet connection — ideal for consultants working on-site, field teams designing intake forms, or anyone on unreliable networks. Forms are saved locally and synced later.
- **Local AI for NL parsing:** Run smaller language models (Llama, Phi, Mistral) on-device for natural language to form conversion without sending form data or business logic to external APIs. Eliminates per-request AI costs.
- **Native drag-and-drop builder:** Desktop drag-and-drop is fundamentally smoother than browser-based implementations. Native cursor handling, snap-to-grid, multi-select, and z-ordering feel more responsive and predictable than HTML5 drag-and-drop.
- **Live form preview without server:** Preview generated forms instantly in a native webview or Flutter widget — no local dev server, no browser tab, no CORS issues. Preview updates in real-time as fields are modified.
- **Filesystem export and integration:** Export forms as HTML, PDF, JSON Schema, or React/Vue components directly to the local filesystem. Watch project directories for existing form schemas to import and iterate on.

---

## Desktop-Specific Features

- **NL form prompt bar:** Persistent input bar (global hotkey Cmd+Shift+F) to type natural language descriptions and generate forms instantly
- **Drag-and-drop form builder:** Native drag-and-drop with grid snapping, multi-select, alignment guides, and undo/redo stack backed by local state
- **Live preview panel:** Side-by-side editing and preview in a native webview — form renders in real-time as you modify fields
- **Local AI inference:** Run quantized LLMs (GGUF format) for NL-to-form parsing without cloud API costs or data leaving the machine
- **Template library:** Bundled offline template gallery (contact forms, surveys, registrations, order forms, multi-step wizards) with local caching
- **Filesystem export:** Save forms as HTML, PDF, JSON Schema, React components, or Vue components to any local directory
- **Project directory watching:** Watch a local project folder and auto-detect existing form schemas to import and edit
- **Multi-window editing:** Open multiple forms in separate windows for side-by-side comparison and editing
- **Version history:** Local git-style version tracking for form iterations — diff and rollback any change
- **Accessibility checker:** Offline WCAG compliance scanner that validates form accessibility (contrast, labels, ARIA, tab order) without external services

---

## Shared Packages

### core (Rust)

- **NL parser engine:** Transforms natural language descriptions into structured form schemas using rule-based parsing + LLM refinement. Handles ambiguous inputs gracefully with fallback suggestions.
- **Form schema model:** Internal form representation (fields, validations, layout, conditional logic, multi-step flows, repeatable sections, file uploads)
- **Validation engine:** Validates form schemas for completeness, accessibility, and logical consistency. Detects unreachable conditional branches and conflicting validation rules.
- **Code generator:** Emits HTML, React JSX, Vue SFC, Svelte components, and JSON Schema from the internal form model. Includes CSS generation with customizable theme variables.
- **PDF renderer:** Generates printable PDF forms from schema using a Rust PDF library (printpdf or genpdf) with fillable fields
- **Template engine:** Manages form templates with parameterized variables, inheritance, and composable sections

### api-client (Rust)

- **Cloud sync:** Push/pull form projects to FormForge cloud for team collaboration and hosted form endpoints
- **AI API client:** OpenAI/Anthropic API client for cloud-based NL parsing when local AI is disabled or insufficient
- **Auth:** JWT-based authentication with team workspace support and role-based access
- **Form submission relay:** Forward form submissions from hosted forms to local desktop app for offline review
- **Asset CDN:** Upload form assets (images, logos) to CDN when publishing forms

### data (SQLite)

- **Schema:** projects, forms (id, title, schema_json, html_output, version, created, updated, synced), form_versions, templates, submissions, settings, nl_prompts (for prompt history)
- **Version history:** Every form save creates a new version row with full schema snapshot — enables diff and rollback
- **Sync strategy:** Per-form versioning with server-side merge; conflicts surface a visual diff for manual resolution
- **FTS index:** Full-text search on form titles, field labels, and NL prompts for quick retrieval across all projects

---

## Framework Implementations

### Electron

**Architecture:** Main process handles filesystem operations, local AI inference (via child process), and project directory watching. Renderer process (React) hosts the drag-and-drop form builder, NL prompt bar, and live preview in an embedded iframe. Secondary window available for multi-form editing.

**Tech stack:**
- Renderer: React 18, dnd-kit (drag-and-drop), TailwindCSS, Monaco Editor (for code view), Zustand
- Main: chokidar (file watching), electron-store, child_process (for llama.cpp server)
- Database: better-sqlite3 with Drizzle ORM
- Preview: sandboxed iframe rendering generated HTML forms in real-time
- AI: llama-node or llama.cpp subprocess for local inference

**Native module needs:** better-sqlite3, node-pty (for AI subprocess management), chokidar

**Bundle size:** ~190-260 MB (Chromium + dnd-kit + optional GGUF model)

**Memory:** ~280-450 MB (higher with live preview iframe and AI model loaded)

**Pros:**
- dnd-kit is the most mature React drag-and-drop library — battle-tested for form builders with nested containers
- iframe-based preview provides perfect HTML form rendering fidelity with proper CSS isolation
- Monaco Editor for viewing/editing generated code (JSON Schema, HTML) with syntax highlighting and validation
- Largest ecosystem of form-related React components (react-hook-form, Formik, Radix primitives)
- Multi-window is straightforward via BrowserWindow API

**Cons:**
- Heavy bundle for a design tool — 250 MB for a form builder feels excessive
- Drag-and-drop in browser context has subtle latency vs. native (especially for complex nested layouts)
- Running local AI alongside Chromium is memory-intensive (700+ MB total with model loaded)
- iframe preview requires careful sandboxing to prevent XSS from user-designed forms
- File export requires IPC round-trips through main process

### Tauri

**Architecture:** Rust backend handles NL parsing (local AI via llama-cpp-rs), form schema validation, code generation, PDF export, and filesystem operations. Svelte frontend provides the drag-and-drop builder, prompt bar, and live preview panel. Rust-native export writes directly to filesystem.

**Tech stack:**
- Frontend: Svelte 5, svelte-dnd-action (drag-and-drop), TailwindCSS, CodeMirror 6 (code view)
- Backend: llama-cpp-rs (local AI), serde_json (schema), printpdf (PDF export), rusqlite, tokio, notify (file watching)
- Preview: Tauri webview window or inline HTML rendering via dynamic DOM injection

**Plugin needs:** tauri-plugin-dialog, tauri-plugin-fs, tauri-plugin-notification, tauri-plugin-shell

**Bundle size:** ~8-18 MB (without GGUF model; +2-4 GB with bundled AI model)

**Memory:** ~50-120 MB (spikes to ~200 MB during AI inference, returns to baseline after)

**Pros:**
- Rust backend excels at schema validation, code generation, and PDF rendering — zero bridge overhead
- svelte-dnd-action provides smooth drag-and-drop with minimal JS overhead and reactive updates
- Local AI via llama-cpp-rs is performant and memory-efficient with model unloading
- Tiny bundle respects users who install many design tools — designers notice bloat
- PDF generation natively in Rust (printpdf) is fast and dependency-free
- File export writes directly from Rust without IPC overhead

**Cons:**
- svelte-dnd-action is less mature than dnd-kit for complex nested form layouts with containers
- Live preview requires rendering HTML in webview — slightly more complex than Electron iframe
- Smaller ecosystem for form-specific UI components (no Radix/Shadcn equivalents for Svelte)
- AI model management (download, update, select) needs custom UI that does not exist as a library
- Multi-window management API is less ergonomic than Electron's BrowserWindow

### Flutter Desktop

**Architecture:** Flutter canvas-based form builder with custom drag-and-drop widgets. Riverpod for state. Rust FFI via flutter_rust_bridge for AI inference and code generation. WebView widget for live form preview.

**Tech stack:**
- UI: Flutter 3.x, Riverpod, custom FormBuilderCanvas widget, webview_flutter (preview)
- Database: drift (SQLite)
- AI: flutter_rust_bridge -> llama-cpp-rs for local inference
- Export: flutter_rust_bridge -> Rust code generators for HTML/JSON/PDF
- DnD: Custom GestureDetector-based drag-and-drop system with HitTestBehavior

**Plugin needs:** webview_flutter, file_picker, window_manager, desktop_drop

**Bundle size:** ~28-40 MB (+ AI model downloaded separately)

**Memory:** ~110-190 MB (WebView for preview adds ~40 MB overhead)

**Pros:**
- Custom canvas rendering allows pixel-perfect form builder with smooth animations and gesture-driven interactions
- Flutter's gesture system enables sophisticated drag-and-drop with multi-touch support on trackpads
- Hot reload accelerates builder UI iteration dramatically — see layout changes instantly
- Could extend to mobile form builder (tablet-optimized) or mobile form filler app with shared codebase
- Skia rendering engine produces consistent output across platforms

**Cons:**
- No built-in drag-and-drop for desktop — must implement custom gesture-based system from scratch (significant effort)
- WebView plugin for form preview is less stable on desktop than web iframe approach
- Code generation and AI require FFI bridge to Rust, adding build complexity and debugging difficulty
- Text input widgets still have desktop-specific quirks (selection handles, IME composition, right-click context menus)
- No equivalent to Monaco Editor for code viewing — limited to basic syntax highlighting

### Swift/SwiftUI (macOS)

**Architecture:** SwiftUI views with AppKit interop for drag-and-drop (NSCollectionView). WKWebView for live form preview. CoreML/MLX for local AI inference on Apple Silicon. PDFKit for native PDF export.

**Tech stack:**
- UI: SwiftUI + NSViewRepresentable for AppKit drag-and-drop, WKWebView (preview)
- Data: SwiftData with CloudKit sync option
- AI: MLX for local NL-to-form inference (Apple Silicon optimized, runs on Neural Engine)
- Export: Swift-based HTML/JSON generators, PDFKit for PDF export with fillable fields
- DnD: NSCollectionView with drag-and-drop delegate pattern and NSDraggingSession

**Bundle size:** ~10-15 MB (+ CoreML model ~50 MB)

**Memory:** ~35-70 MB (WKWebView adds ~30 MB for preview panel)

**Pros:**
- NSCollectionView drag-and-drop is the most refined native DnD implementation — smooth, accessible, and familiar to macOS users
- WKWebView preview shares Safari engine — accurate HTML form rendering with proper CSS Grid/Flexbox
- MLX inference on Apple Silicon is the fastest local AI option — Neural Engine acceleration reduces power consumption
- PDFKit provides native, high-quality PDF generation with fillable form fields and digital signatures
- Smooth animations via SwiftUI transitions, matched geometry, and spring animations
- ShareExtension: design forms from content selected in other apps (e.g., select requirements from a document)
- Native print dialog integration for form PDFs

**Cons:**
- macOS only — excludes Windows/Linux users (significant for a SaaS tool targeting diverse teams)
- SwiftUI drag-and-drop is still less flexible than AppKit — requires NSViewRepresentable bridging for advanced scenarios
- No cross-platform code sharing for the builder UI or form schema logic
- Smaller community building desktop form tools in Swift — fewer examples and tutorials
- AppKit interop adds complexity vs. pure SwiftUI

### Kotlin Multiplatform

**Architecture:** Compose Multiplatform UI with custom drag-and-drop via pointer input modifiers. JCef (Chromium Embedded) or JavaFX WebView for form preview. Ktor for API, SQLDelight for data.

**Tech stack:**
- UI: Compose Desktop, Material 3, custom DnD via Modifier.pointerInput and Modifier.onDrag
- Preview: JCef (Chromium Embedded Framework) or JavaFX WebView for HTML form rendering
- Database: SQLDelight with typed queries
- AI: DJL (Deep Java Library) for local inference, or Ktor client to cloud APIs
- Export: Kotlin template engines (kotlinx.html for type-safe HTML, FreeMarker for templated output)
- HTTP: Ktor client

**Bundle size:** ~55-80 MB (+JRE + optional AI model)

**Memory:** ~180-320 MB (JVM + JCef for preview adds significant overhead)

**Pros:**
- kotlinx.html provides type-safe HTML generation — compiler catches invalid HTML in form code output
- Compose pointer input system allows flexible custom drag-and-drop with gesture detection
- Could share code with Android form builder/filler companion app via shared KMP modules
- FreeMarker template engine is mature and battle-tested for code generation patterns
- Strong typing with sealed classes maps well to form field variant modeling

**Cons:**
- JCef for preview adds significant bundle size (~40 MB) and memory overhead
- Custom DnD in Compose is verbose and less polished than web-based solutions like dnd-kit
- JVM startup time (~2s) makes the app feel sluggish on launch for quick form edits
- AI inference via DJL has higher overhead than native Rust or CoreML options
- Material 3 design language may not suit a form builder aimed at designers

---

## Comparison Matrix

| Dimension | Electron | Tauri | Flutter | Swift | Kotlin MP |
|-----------|----------|-------|---------|-------|-----------|
| Dev effort (weeks) | 7 | 8 | 10 | 8 | 10 |
| Bundle size | 230 MB | 14 MB | 35 MB | 12 MB | 70 MB |
| Memory usage | 350 MB | 80 MB | 150 MB | 50 MB | 250 MB |
| Startup time | 2.8s | 0.9s | 1.3s | 0.5s | 2.2s |
| Native feel | 5/10 | 6/10 | 7/10 | 10/10 | 5/10 |
| Offline capability | 7/10 | 9/10 | 7/10 | 9/10 | 7/10 |
| Drag-and-drop quality | 8/10 | 7/10 | 6/10 | 9/10 | 5/10 |
| Cross-platform | 9/10 | 9/10 | 9/10 | 2/10 | 8/10 |
| Best fit score | 7/10 | 8/10 | 6/10 | 7/10 | 4/10 |

---

## Recommended Framework

**Tauri** is the best fit for FormForge Desktop.

**Rationale:** FormForge requires a rich visual builder with local AI inference and multi-format export — all areas where Tauri's Rust backend excels. The Rust core handles NL parsing (llama-cpp-rs), schema validation (serde), PDF generation (printpdf), and code generation without any FFI bridges. Svelte provides a sufficiently capable drag-and-drop builder UI while keeping the bundle lean. For a tool users install alongside many other design and development tools, the 14 MB footprint vs. Electron's 230 MB is a meaningful differentiator.

**Runner-up:** Electron, if the team prioritizes the richest possible drag-and-drop builder experience. dnd-kit and Monaco Editor provide a more polished editing environment at the cost of a much heavier footprint. Worth considering if the primary user persona is a designer who already has Figma and similar heavy apps running.

---

## Monetization (Desktop)

- **Free tier:** Local-only form building, up to 10 forms, export to HTML and JSON Schema, community templates
- **Pro ($14/mo or $119/yr):** Unlimited forms, local AI inference, PDF export, React/Vue/Svelte component export, cloud sync, custom branding on forms
- **Team ($8/user/mo):** Shared form libraries, team templates, form submission management, role-based permissions, form analytics
- **One-time license ($129):** Perpetual desktop license with 1 year of updates — targets freelancers and consultants who dislike subscriptions
- **Distribution:** Direct download, Homebrew, winget. Mac App Store for discoverability (sandboxing is acceptable since no global hotkeys needed). Consider Setapp inclusion for macOS reach.

---

## MVP Scope (Desktop)

**Core MVP (8 weeks):**

| Week | Milestone |
|------|-----------|
| 1-2 | Form schema model (Rust core), basic drag-and-drop builder UI, field types (text, email, number, select, checkbox, textarea, date, file upload) |
| 3-4 | NL prompt bar with cloud AI integration, form preview panel (live HTML rendering), undo/redo with full state history |
| 5-6 | Export to HTML, JSON Schema, and PDF. Template library with 10 starter templates. Form validation rules UI (required, min/max, pattern, custom) |
| 7 | SQLite persistence, form version history with diff view, project management (create/rename/delete/duplicate forms) |
| 8 | Auto-update, packaging (DMG/MSI/AppImage), conditional logic builder (show/hide fields based on values), beta testing |

**Post-MVP:** Local AI inference (GGUF models), React/Vue/Svelte component export, team sync and collaboration, multi-step form wizard builder, accessibility checker, form submission analytics, theme designer, Zapier/webhook integration for submissions
