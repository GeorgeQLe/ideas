# DocuMerge Desktop — 5-Framework Comparison

## Desktop Rationale

DocuMerge (automated API documentation generator) is a natural desktop application for developers:

- **Local code scanning:** Parse local codebases directly for API endpoints, route definitions, and type schemas — no need to push code to a cloud service. Supports private repos and pre-commit documentation.
- **Offline doc generation:** Generate complete API documentation without internet. Run the full pipeline (scan, parse, render, validate) on an airplane or in air-gapped environments.
- **Preview server:** Launch a local HTTP server to preview generated docs in real-time, with hot reload as code changes — identical to production rendering.
- **File system watching:** Monitor source files for changes and auto-regenerate affected documentation sections. Detect new endpoints, modified schemas, and deprecated routes instantly.
- **Local dev tool integration:** Connect to running local servers for live endpoint testing, integrate with IDE extensions via IPC, and hook into git pre-commit for doc validation.

---

## Desktop-Specific Features

- **Directory scanner:** Recursive code scanning with framework auto-detection (Express, FastAPI, Spring, Rails, Go net/http, etc.)
- **File watcher:** Monitor source directories for changes; auto-regenerate only affected doc sections (incremental builds)
- **Local preview server:** Built-in HTTP server (localhost:PORT) serving rendered docs with hot reload
- **Tree-sitter parsing:** AST-level code analysis for accurate endpoint extraction across 15+ languages
- **Git-aware diffing:** Show documentation changes alongside code diffs; generate "what's new" sections from git history
- **Drag-and-drop project:** Drop a project folder onto the app to start scanning
- **Multi-project workspace:** Manage docs for multiple APIs/microservices in a single dashboard
- **OpenAPI/Swagger import:** Drag in existing spec files to merge with scanned endpoints
- **CLI companion:** `documerge scan .` command that syncs results with the desktop app
- **Export pipeline:** One-click export to Markdown, HTML, PDF, or deploy to hosted docs site

---

## Shared Packages

### core (Rust)

- **Code scanner:** Multi-language parser using tree-sitter grammars for endpoint detection (routes, controllers, handlers)
- **Schema extractor:** Infer request/response types from code, JSDoc/docstrings, and TypeScript/Python type annotations
- **Doc generator:** Convert parsed API structures into OpenAPI 3.1 spec, then render to Markdown/HTML via templates
- **Incremental builder:** Dependency graph tracking which doc sections are affected by which source files; rebuild only changed sections
- **Validator:** Check for missing descriptions, undocumented parameters, broken cross-references, and schema inconsistencies
- **Merge engine:** Combine auto-generated docs with hand-written overrides and annotations without losing manual edits

### api-client (Rust)

- **Cloud publish:** Push generated docs to DocuMerge hosted site with versioning and custom domain support
- **AI enhancement:** Send incomplete doc sections to LLM API for description generation and example creation
- **Auth:** API key for CLI and cloud publishing, OAuth2 for team workspaces
- **Webhook:** Trigger doc rebuilds from CI/CD webhooks; notify on broken doc builds

### data (SQLite)

- **Schema:** projects, endpoints (path, method, params, body_schema, response_schema, description, source_file, line_number), doc_overrides, build_history, settings
- **Incremental cache:** Store parsed ASTs and generated doc fragments keyed by file hash for instant incremental rebuilds
- **Full-text search:** FTS5 index on endpoint paths, descriptions, and parameter names for instant doc search

---

## Framework Implementations

### Electron

**Architecture:** Main process handles file watching (chokidar), local preview server (express), and tree-sitter parsing via native module. Renderer process (React) for project management, doc browser, and live preview.

**Tech stack:**
- Renderer: React 18, Swagger UI React (doc preview), Monaco Editor (doc editing), TailwindCSS, Zustand
- Main: chokidar (file watcher), express (preview server), tree-sitter (Node.js bindings), better-sqlite3
- Preview: Embedded Chromium renders docs identically to hosted version
- Export: puppeteer-core (PDF), remark (Markdown processing)

**Native module needs:** tree-sitter (+ language grammars), better-sqlite3, node-pty (CLI integration)

**Bundle size:** ~200-270 MB (Chromium + tree-sitter grammars + preview assets)

**Memory:** ~280-450 MB (higher with Monaco editor, preview server, and large project scans)

**Pros:**
- Swagger UI React provides production-grade API doc rendering out of the box
- Monaco Editor enables rich doc editing with schema-aware autocomplete
- Preview server rendered in Chromium is pixel-identical to hosted docs
- Massive npm ecosystem for Markdown processing (remark, rehype, unified)

**Cons:**
- Very heavy bundle for a developer tool — developers will compare to lightweight alternatives
- Tree-sitter native bindings can be fragile across Electron versions and OS updates
- Running preview server + file watcher + Chromium = high resource consumption
- Hot reload latency for doc preview adds 500ms+ due to Chromium re-render cycle

### Tauri

**Architecture:** Rust backend handles all code scanning (tree-sitter), file watching (notify crate), incremental doc building, and preview server (axum). Svelte frontend for project management, doc browsing, and live preview via iframe to local server.

**Tech stack:**
- Frontend: Svelte 5, TailwindCSS, CodeMirror 6 (doc editing), iframe preview
- Backend: tree-sitter (Rust), notify (file watcher), axum (preview server), rusqlite, tokio
- Doc rendering: pulldown-cmark (Markdown), tera (HTML templates), openapiv3 (spec handling)

**Plugin needs:** tauri-plugin-dialog, tauri-plugin-fs, tauri-plugin-shell (CLI integration), tauri-plugin-localhost

**Bundle size:** ~12-22 MB (+ tree-sitter grammars ~5 MB)

**Memory:** ~50-100 MB (spikes to ~180 MB during full project scans)

**Pros:**
- **Fastest code scanning:** tree-sitter runs natively in Rust — parse 10K+ files in seconds
- axum preview server is extremely lightweight (~2 MB memory overhead)
- File watching via notify crate is efficient and cross-platform
- Incremental rebuild in Rust processes only changed files — sub-100ms updates
- Tiny bundle that developers respect; installs in seconds
- Rust's parallel iterators (rayon) enable multi-core scanning for large monorepos

**Cons:**
- No Swagger UI equivalent in Svelte — must build or port a doc renderer
- Fewer Markdown processing libraries than the JavaScript ecosystem
- iframe preview introduces slight styling complexity for doc rendering
- Tree-sitter grammar compilation adds to build complexity

### Flutter Desktop

**Architecture:** Dart UI with Riverpod state management. Code scanning delegated to Rust via flutter_rust_bridge (tree-sitter + parsing logic). Custom doc viewer widget. Platform channels for file system access.

**Tech stack:**
- UI: Flutter 3.x, flutter_markdown (doc preview), Riverpod, CodeViewer (syntax highlighting)
- Database: drift (SQLite)
- Scanning: flutter_rust_bridge -> tree-sitter + custom Rust scanner
- Preview: flutter_inappwebview (embedded preview)

**Plugin needs:** file_picker, window_manager, desktop_drop, flutter_rust_bridge

**Bundle size:** ~30-40 MB (+ tree-sitter grammars)

**Memory:** ~120-200 MB

**Pros:**
- flutter_markdown provides decent doc rendering with custom widget injection
- Hot reload excellent for iterating on the documentation editor UI
- Could extend to a mobile API doc reader app
- Custom rendering allows unique doc visualization (interactive endpoint explorer)

**Cons:**
- Code scanning requires FFI bridge to Rust — significant complexity overhead
- flutter_inappwebview for doc preview adds memory overhead and platform inconsistency
- Text-heavy documentation tool does not leverage Flutter's rendering strengths
- Desktop file system watching requires platform channel boilerplate per OS

### Swift/SwiftUI (macOS)

**Architecture:** SwiftUI views with @Observable models. tree-sitter via C bindings (SwiftTreeSitter). Native WKWebView for doc preview. Local HTTP server via Vapor or SwiftNIO. FSEvents for file watching.

**Tech stack:**
- UI: SwiftUI, WKWebView (doc preview), native TextEditor
- Data: SwiftData
- Scanning: SwiftTreeSitter (tree-sitter bindings), custom parsers
- Preview: WKWebView loading from local Vapor/SwiftNIO server
- File watching: FSEvents (native macOS file system events)

**Bundle size:** ~10-18 MB (+ tree-sitter grammars)

**Memory:** ~35-70 MB

**Pros:**
- FSEvents is the most efficient file watching API on macOS (kernel-level, zero polling)
- WKWebView doc preview shares Safari's rendering engine — excellent HTML/CSS support
- Native drag-and-drop for project folders is seamless
- Xcode-quality syntax highlighting available via SourceKit integration
- Low memory footprint ideal for running alongside IDEs and compilers

**Cons:**
- macOS only — excludes Linux developers (a significant portion of the target audience)
- SwiftTreeSitter is less maintained than the Rust or Node bindings
- Vapor/SwiftNIO server adds complexity compared to axum or express
- No cross-platform code sharing for a tool that developers expect everywhere

### Kotlin Multiplatform

**Architecture:** Compose Multiplatform UI with Material 3. JNI bridge to tree-sitter C library for parsing. Ktor embedded server for preview. SQLDelight for persistence.

**Tech stack:**
- UI: Compose Desktop, Material 3, KCEF (Kotlin Chromium Embedded Framework) for doc preview
- Database: SQLDelight
- Scanning: JNI -> tree-sitter, or JavaParser (Java/Kotlin only)
- Preview: Ktor embedded server + KCEF
- HTTP: Ktor client

**Bundle size:** ~55-80 MB (+JRE)

**Memory:** ~200-350 MB (JVM + KCEF for preview)

**Pros:**
- JavaParser is excellent for Java/Kotlin/Spring API scanning specifically
- Ktor embedded server is clean and well-documented for preview hosting
- Could share doc generation logic with an IntelliJ/Android Studio plugin
- Strong typing with Kotlin data classes maps well to OpenAPI schemas

**Cons:**
- JNI bridge to tree-sitter is fragile and poorly maintained
- KCEF adds ~100 MB to bundle and significant memory overhead
- JVM startup time (~2s) frustrates developers expecting instant response
- Highest resource usage when running alongside other JVM-based tools (IntelliJ)

---

## Comparison Matrix

| Dimension | Electron | Tauri | Flutter | Swift | Kotlin MP |
|-----------|----------|-------|---------|-------|-----------|
| Dev effort (weeks) | 6 | 7 | 9 | 8 | 9 |
| Bundle size | 235 MB | 17 MB | 35 MB | 14 MB | 70 MB |
| Memory usage | 360 MB | 75 MB | 160 MB | 50 MB | 275 MB |
| Startup time | 2.8s | 0.7s | 1.2s | 0.4s | 2.2s |
| Native feel | 5/10 | 7/10 | 5/10 | 10/10 | 4/10 |
| Offline capability | 8/10 | 9/10 | 7/10 | 9/10 | 7/10 |
| Code scanning speed | 6/10 | 10/10 | 7/10 | 7/10 | 5/10 |
| Cross-platform | 9/10 | 9/10 | 9/10 | 2/10 | 8/10 |
| Best fit score | 6/10 | 9/10 | 5/10 | 7/10 | 4/10 |

---

## Recommended Framework

**Tauri** is the best fit for DocuMerge Desktop.

**Rationale:** DocuMerge is a developer tool that needs to scan codebases fast, watch files efficiently, and stay lightweight while running alongside resource-heavy IDEs and compilers. Tauri's Rust backend provides native tree-sitter parsing (10K+ files in seconds), efficient file watching via notify, and a lightweight axum preview server — all without the overhead of Chromium. The 17 MB bundle installs instantly and earns developer trust. Cross-platform support is essential since developers work on macOS, Linux, and Windows.

**Runner-up:** Electron if the priority is reusing an existing SaaS web frontend (Swagger UI, Monaco Editor), but the 235 MB bundle and 360 MB memory usage are hard to justify for a developer tool.

---

## Monetization (Desktop)

- **Free tier:** Single project, up to 50 endpoints, local doc generation only
- **Pro ($12/mo or $99/yr):** Unlimited projects, AI-enhanced descriptions, cloud publishing, custom domains
- **One-time license ($89):** Perpetual desktop license with 1 year of updates for indie developers
- **Team ($6/user/mo):** Shared doc workspaces, CI/CD webhook triggers, role-based editing
- **Distribution:** Direct download + Homebrew + Scoop (Windows). Skip app stores — developers prefer direct installs.

---

## MVP Scope (Desktop)

**Core MVP (8 weeks):**

| Week | Milestone |
|------|-----------|
| 1-2 | Project scanning with tree-sitter, endpoint extraction for Express/FastAPI/Spring, SQLite storage |
| 3-4 | Doc generation pipeline (OpenAPI spec -> Markdown/HTML), live preview server, basic doc editor |
| 5-6 | File watcher with incremental rebuild, git-aware change detection, multi-project workspace |
| 7 | Export pipeline (Markdown, HTML, PDF), OpenAPI import/merge, search across endpoints |
| 8 | Auto-update, packaging (DMG/MSI/AppImage), CLI companion tool, beta testing with 10 developers |

**Post-MVP:** AI-enhanced descriptions, cloud publishing, team workspaces, CI/CD integration, additional framework parsers (Go, Rust, Ruby)
