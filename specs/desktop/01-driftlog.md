# DriftLog Desktop — 5-Framework Comparison

## Desktop Rationale

DriftLog (automated changelog generator) benefits significantly from a desktop version:

- **Direct git access:** Read local repositories via libgit2 without GitHub/GitLab API rate limits. Analyze private repos without OAuth setup — just point at a directory.
- **Offline changelog review:** Draft, edit, and approve changelog entries without internet. Sync when back online.
- **System tray monitoring:** Watch for new commits/tags in background, auto-generate drafts on release events.
- **Large repo performance:** Diff analysis on multi-GB repos is 10-100x faster locally vs. sending diffs through an API.
- **Local AI inference:** Run smaller LLMs (Llama, Mistral) locally for changelog generation without API costs or data leaving the machine.

---

## Desktop-Specific Features

- **Local git repository scanning:** Auto-discover repos in ~/Projects or configured directories
- **File system watcher:** Monitor .git directories for new commits, tags, and branch merges
- **System tray icon:** Badge showing pending changelog drafts; click to review
- **Global hotkey:** Quick-generate changelog for current repo (e.g., Cmd+Shift+L)
- **Native diff viewer:** Side-by-side diff display using native text rendering
- **Drag-and-drop:** Drop a folder onto the app to add a repo
- **Git credential reuse:** Leverage system git credentials (SSH keys, credential helpers) instead of OAuth
- **Multi-repo dashboard:** Unified view of all local repos and their changelog status
- **Local AI mode:** Embed GGUF models via llama.cpp for fully offline changelog generation
- **Export to file:** Generate CHANGELOG.md directly in the repo root

---

## Shared Packages

### core (Rust)

- **Git analysis engine:** Wraps libgit2 for commit parsing, diff generation, PR correlation
- **Changelog formatter:** Markdown generation, semantic versioning detection, entry categorization (feature/fix/breaking/improvement)
- **AI prompt builder:** Constructs prompts from commit data for LLM consumption
- **Template engine:** Customizable changelog output templates (Keep a Changelog, conventional, custom)
- **Validation:** Ensures changelog entries have required fields, detects duplicates

### api-client (Rust)

- **GitHub/GitLab REST + GraphQL:** Fetch PR metadata, labels, linked issues for richer context
- **Cloud sync:** Push generated changelogs to DriftLog cloud for hosted page
- **AI API client:** OpenAI/Anthropic API for cloud-based generation (when local AI not used)
- **Auth:** OAuth2 token management with refresh, git credential helper integration
- **Offline queue:** Queue cloud syncs and API calls; replay on reconnect with conflict detection

### data (SQLite)

- **Schema:** repositories, commits, changelog_entries, drafts, settings, sync_queue
- **Local-first:** All changelog history stored locally; cloud is optional sync target
- **Full-text search:** FTS5 index on changelog entry content for instant search
- **Sync strategy:** Last-write-wins for individual entries with server timestamp comparison

---

## Framework Implementations

### Electron

**Architecture:** Main process manages git operations (via nodegit or spawned git CLI), file watchers, and system tray. Renderer process (React + Monaco Editor) handles changelog editing and diff viewing.

**Tech stack:**
- Renderer: React 18, TailwindCSS, Monaco Editor (diff viewer), Zustand state management
- Main: nodegit or isomorphic-git, chokidar (file watching), electron-store (settings)
- Database: better-sqlite3 with Drizzle ORM
- AI: OpenAI SDK + optional llama-node for local inference

**Native module needs:** nodegit (libgit2 bindings), better-sqlite3, node-pty (for git CLI fallback)

**Bundle size:** ~180-250 MB (Chromium + nodegit + optional GGUF model)

**Memory:** ~250-400 MB (higher with Monaco editor and large diffs loaded)

**Pros:**
- Monaco Editor provides VS Code-quality diff viewing out of the box
- Maximum code reuse from DriftLog SaaS web frontend
- nodegit is mature and well-documented
- Large ecosystem for markdown rendering (remark, rehype)

**Cons:**
- Heavy bundle for what is fundamentally a text-processing tool
- nodegit native compilation can be fragile across OS versions
- Memory overhead for background monitoring is excessive
- System tray + background watching keeps full Chromium process alive

### Tauri

**Architecture:** Rust backend handles all git operations natively (git2-rs), file watching (notify crate), and AI inference (llama-cpp-rs). Svelte frontend for changelog editing and review.

**Tech stack:**
- Frontend: Svelte 5, TailwindCSS, CodeMirror 6 (diff viewer)
- Backend: git2-rs (libgit2), notify (file watcher), rusqlite, reqwest, tokio
- AI: llama-cpp-rs for local inference, or reqwest to cloud APIs

**Plugin needs:** tauri-plugin-notification, tauri-plugin-dialog, tauri-plugin-fs, tauri-plugin-shell (git CLI fallback)

**Bundle size:** ~8-15 MB (without GGUF model; +2-4 GB with bundled model)

**Memory:** ~40-80 MB (spikes to ~150 MB during large repo analysis)

**Pros:**
- git2-rs is the native Rust binding for libgit2 — zero bridge overhead, best performance
- Tiny bundle; ideal for a developer tool (devs notice bloat)
- Background file watching via notify crate uses <5 MB additional memory
- Rust backend can analyze 100K+ commit repos without breaking a sweat
- Local AI via llama-cpp-rs is fastest non-Swift option

**Cons:**
- CodeMirror diff view is less polished than Monaco
- Smaller ecosystem for markdown processing (though pulldown-cmark is solid)
- Rust compilation adds to CI build times

### Flutter Desktop

**Architecture:** Dart UI with Riverpod state management. Git operations via flutter_rust_bridge calling into git2-rs. Custom diff viewer widget.

**Tech stack:**
- UI: Flutter 3.x, Riverpod, custom DiffWidget, flutter_markdown
- Database: drift (SQLite)
- Git: flutter_rust_bridge → git2-rs
- AI: HTTP client to cloud API, or FFI to llama.cpp

**Plugin needs:** tray_manager, local_notifier, file_picker, window_manager

**Bundle size:** ~25-35 MB

**Memory:** ~100-180 MB

**Pros:**
- Custom diff viewer can be pixel-perfect across platforms
- Hot reload speeds up UI iteration for the editing experience
- Single codebase could extend to a mobile changelog reader app

**Cons:**
- No equivalent to Monaco Editor — must build custom diff viewer from scratch
- Git operations require FFI bridge (flutter_rust_bridge) adding complexity
- Text-heavy app doesn't leverage Flutter's rendering strengths
- Desktop text input/selection still has rough edges in Flutter

### Swift/SwiftUI (macOS)

**Architecture:** SwiftUI views with @Observable view models. Git operations via SwiftGit2 (libgit2 wrapper) or Process-based git CLI. SwiftData for local persistence.

**Tech stack:**
- UI: SwiftUI, TextEditor, native NSTextView for diffs
- Data: SwiftData
- Git: SwiftGit2 or swift-git (Process wrapper)
- AI: MLX for local inference (Apple Silicon optimized), or URLSession to cloud API

**Bundle size:** ~8-12 MB

**Memory:** ~30-60 MB

**Pros:**
- Best system tray / menu bar experience on macOS (NSStatusItem is native)
- MLX provides fastest local AI inference on Apple Silicon
- Native text rendering is superior for markdown/code display
- Spotlight integration: search changelogs from Spotlight
- Smallest memory footprint for background monitoring
- macOS Shortcuts integration for automation

**Cons:**
- macOS only — no Windows/Linux (significant for a dev tool)
- SwiftGit2 is less maintained than git2-rs
- Smaller developer community for desktop Swift
- No cross-platform code sharing

### Kotlin Multiplatform

**Architecture:** Compose Multiplatform UI, ViewModel + StateFlow for state, JGit for git operations, SQLDelight for persistence.

**Tech stack:**
- UI: Compose Desktop, Material 3
- Git: JGit (Eclipse) — mature pure-Java git implementation
- Database: SQLDelight
- AI: DJL (Deep Java Library) or Ktor client to cloud APIs
- HTTP: Ktor

**Bundle size:** ~50-70 MB (+JRE)

**Memory:** ~180-300 MB (JVM overhead + JGit)

**Pros:**
- JGit is the most mature pure-language git library (used by Eclipse, Gerrit)
- Strong typing with Kotlin data classes maps well to changelog domain
- Could share code with a future Android companion app
- JVM ecosystem has excellent text processing libraries

**Cons:**
- JVM startup time (~2s) feels sluggish for a quick-access tool
- Highest memory usage for background monitoring
- System tray support via AWT is clunky
- Bundle requires JRE, which devs may find redundant

---

## Comparison Matrix

| Dimension | Electron | Tauri | Flutter | Swift | Kotlin MP |
|-----------|----------|-------|---------|-------|-----------|
| Dev effort (weeks) | 6 | 7 | 9 | 7 | 8 |
| Bundle size | 220 MB | 12 MB | 30 MB | 10 MB | 60 MB |
| Memory usage | 300 MB | 60 MB | 140 MB | 45 MB | 220 MB |
| Startup time | 2.5s | 0.8s | 1.2s | 0.4s | 2.0s |
| Native feel | 5/10 | 6/10 | 6/10 | 10/10 | 5/10 |
| Offline capability | 8/10 | 9/10 | 7/10 | 9/10 | 8/10 |
| GPU/compute access | 5/10 | 8/10 | 5/10 | 9/10 | 4/10 |
| Cross-platform | 9/10 | 9/10 | 9/10 | 2/10 | 8/10 |
| Best fit score | 7/10 | 9/10 | 5/10 | 7/10 | 5/10 |

---

## Recommended Framework

**Tauri** is the best fit for DriftLog Desktop.

**Rationale:** DriftLog is a developer tool, and developers notice (and dislike) bloated apps. Tauri's 12 MB bundle vs. Electron's 220 MB is a meaningful difference for the target audience. The Rust backend with native git2-rs provides the best git analysis performance with zero bridge overhead. Background file watching is ultra-lightweight. The only trade-off is a less polished diff viewer than Monaco, but CodeMirror 6 is more than adequate.

**Runner-up:** Swift/SwiftUI if targeting Mac-only (best system tray UX, MLX for local AI), but the macOS-only limitation is a dealbreaker for a developer tool that must serve Linux users.

---

## Monetization (Desktop)

- **Free tier:** Local-only mode — unlimited repos, local AI, no cloud sync
- **Pro ($19/mo or $149/yr):** Cloud sync, hosted changelog page, GitHub/GitLab push integration
- **One-time license ($99):** Perpetual desktop license with 1 year of updates (no subscription fatigue for devs)
- **App Store:** Skip Mac App Store (devs prefer direct download); distribute via GitHub Releases + Homebrew
- **License key system:** Keygen.sh or self-hosted license validation with offline grace period

---

## MVP Scope (Desktop)

**Core MVP (8 weeks):**

| Week | Milestone |
|------|-----------|
| 1-2 | Local git repo scanning, commit parsing, and diff analysis (Rust core) |
| 3-4 | AI changelog generation (cloud API), entry editor UI, categorization |
| 5-6 | System tray icon, file watcher for new commits/tags, notification system |
| 7 | SQLite persistence, search, multi-repo dashboard |
| 8 | Auto-update, packaging (DMG/MSI/AppImage), beta testing |

**Post-MVP:** Local AI inference, GitHub/GitLab cloud sync, team features, CHANGELOG.md export
