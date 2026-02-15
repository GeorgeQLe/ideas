# ShipFlag Desktop — 5-Framework Comparison

## Desktop Rationale

ShipFlag (feature flag management) is a natural fit for a desktop application targeting developers and engineering teams:

- **Local flag evaluation testing:** Test feature flag logic against local application instances without deploying to staging. Evaluate flag rules with mock user contexts directly on the developer's machine — see exactly what a user in segment X with attribute Y would experience.
- **Offline flag management:** Create, edit, and review flag configurations without internet. Queue changes locally and push to the flag service when reconnected — essential for working on planes, trains, or at conferences with unreliable hotel Wi-Fi.
- **System tray quick toggle:** Instantly toggle feature flags from the system tray during development and QA. No context-switching to a browser dashboard — one click to flip a flag for local testing. Saves dozens of tab-switches per day during feature development.
- **Integration with local dev environments:** Connect to locally-running applications via localhost WebSocket. Override flag values for specific applications running on the developer's machine without affecting shared staging environments.
- **Flag state visualization:** Desktop-native tree and graph views for complex flag dependencies. Visualize which flags affect which features with interactive dependency diagrams — invaluable for understanding flag relationships before a cleanup sprint.

---

## Desktop-Specific Features

- **System tray flag toggles:** Quick toggle panel in tray popup showing flags for the active project — flip flags with one click, see current state at a glance
- **Global hotkey (Cmd+Shift+G):** Open flag search and toggle overlay from anywhere during development — type flag name, press enter to toggle
- **Local override server:** Lightweight HTTP/WebSocket server on localhost that SDK clients connect to for real-time local overrides without touching production
- **Flag evaluation playground:** Test flag rules against custom user contexts (attributes, segments, percentage buckets) with instant visual results and rule tracing
- **Dependency graph viewer:** Interactive force-directed graph showing flag dependencies, prerequisite chains, and feature relationships with zoom and filter
- **Diff viewer:** Side-by-side comparison of flag configurations between environments (dev/staging/prod) with line-by-line highlighting
- **Change audit trail:** Local log of all flag changes with timestamps, authors, diff snapshots, and one-click rollback capability
- **Dev environment detection:** Auto-detect running local servers (Next.js, Rails, Django, Spring Boot) and offer to connect for flag overrides via discovered ports
- **Flag templates:** Reusable flag configuration templates (kill switch, percentage rollout, A/B test, beta access, maintenance mode) with parameterized defaults
- **Bulk operations:** Select multiple flags for batch enable/disable, environment promotion, archival, or tag assignment

---

## Shared Packages

### core (Rust)

- **Flag evaluator:** Complete flag evaluation engine supporting boolean, string, number, and JSON variants with targeting rules, user segments, percentage rollouts, and multi-variate experiments
- **Rule engine:** Parse and evaluate targeting rules (user attributes, segment membership, date ranges, regex patterns, semantic version comparison, IP range matching, custom operators)
- **Dependency resolver:** Detect and validate flag prerequisite chains, circular dependencies, cascading disable effects, and orphaned flags
- **Diff engine:** Compare flag configurations across environments, highlight changes (added/removed/modified rules), and generate promotion migration plans
- **Template system:** Manage flag templates with parameterized defaults for common patterns (rollout percentage, kill switch target, experiment variant allocation)

### api-client (Rust)

- **Flag sync:** Real-time WebSocket connection to ShipFlag cloud for live flag updates; REST for configuration CRUD with optimistic locking
- **Environment management:** Push/pull flag configurations across environments with promotion workflows, approval gates, and rollback history
- **Auth:** API key auth for SDK connections, OAuth2 for dashboard users, team management with role-based permissions (viewer, editor, admin)
- **Webhook dispatcher:** Notify external services (Slack, PagerDuty, CI/CD pipelines, Datadog) when flags change state
- **SDK relay:** Local WebSocket server that acts as a relay between ShipFlag cloud and locally-running SDK clients with caching

### data (SQLite)

- **Schema:** projects, flags (id, key, name, description, type, variants[], default_value, targeting_rules[], prerequisites[], environment_overrides{}, tags[], archived), environments, segments (id, name, rules[]), change_history, local_overrides, evaluation_cache, settings
- **Change log:** Every flag modification stored with full before/after JSON snapshots for audit trail and one-click rollback
- **Sync strategy:** Environment-scoped sync — each environment syncs independently; conflicts detected via version vectors and surfaced in diff viewer for manual resolution
- **Cache:** Evaluation cache keyed by flag_key+context_hash for instant repeated evaluations during playground testing

---

## Framework Implementations

### Electron

**Architecture:** Main process runs the local override WebSocket server (ws library), flag sync connection to cloud, and system tray management. Renderer process (React) for the flag management dashboard, dependency graph visualization, evaluation playground, and environment diff viewer. Hidden tray popup window for quick toggles.

**Tech stack:**
- Renderer: React 18, react-flow (dependency graph), Monaco Editor (JSON flag configs), TailwindCSS, Zustand
- Main: ws (WebSocket server for local SDK overrides), electron-tray, chokidar (detect local dev server ports)
- Database: better-sqlite3 with Drizzle ORM
- Diff: diff2html (environment diff viewer with syntax highlighting)

**Native module needs:** better-sqlite3, electron-positioner (tray popup alignment)

**Bundle size:** ~170-230 MB

**Memory:** ~240-380 MB (react-flow graph rendering + Monaco + WebSocket connections)

**Pros:**
- react-flow is the best interactive graph library for flag dependency visualization — supports custom nodes, edges, layouts, and minimap
- Monaco Editor provides excellent JSON editing for flag configurations with JSON Schema validation, IntelliSense, and error highlighting
- diff2html renders polished side-by-side diffs for environment comparison with word-level change highlighting
- WebSocket server in Node.js main process is simple and reliable for local SDK relay — handles connection management automatically
- Full SaaS dashboard code reuse — flag management UI, targeting rule builder, segment editor all portable

**Cons:**
- Heavy for a developer utility that should be lightweight and always running alongside IDE and Docker
- react-flow with many nodes (50+ flags with dependencies) can become slow in a Chromium renderer — requires virtualization
- Background WebSocket server keeps full Chromium process alive consuming resources even when dashboard is closed
- Developers installing ShipFlag alongside their IDE, Docker, databases, and monitoring tools will notice 300+ MB memory usage
- globalShortcut conflicts are common in developer environments with many tools binding keyboard shortcuts

### Tauri

**Architecture:** Rust backend runs the local override WebSocket server (tungstenite), flag evaluation engine, dependency resolution, and system tray. Svelte frontend for the dashboard, dependency graph, diff viewer, and evaluation playground. Lightweight tray popup for quick toggles with search.

**Tech stack:**
- Frontend: Svelte 5, d3-force (dependency graph), CodeMirror 6 (JSON editing), TailwindCSS
- Backend: tungstenite (WebSocket server), serde_json (flag config parsing and validation), rusqlite, tokio, reqwest
- Diff: custom Svelte diff component using diff-rs for text comparison

**Plugin needs:** tauri-plugin-global-shortcut, tauri-plugin-notification, tauri-plugin-positioner, tauri-plugin-autostart

**Bundle size:** ~6-12 MB

**Memory:** ~25-55 MB (lightweight for always-running dev tool)

**Pros:**
- **Smallest footprint** — critical for a tool running alongside IDEs, Docker containers, databases, build tools, and other dev infrastructure
- Rust WebSocket server (tungstenite + tokio) handles hundreds of concurrent SDK connections efficiently with minimal memory per connection
- Flag evaluation engine in Rust processes complex targeting rules in microseconds — faster than any JS or JVM implementation
- d3-force graph rendering in webview is performant for typical flag counts (<200 nodes) with smooth animation
- Developers respect a 10 MB tool — sends the right signal for a developer-facing product about engineering quality
- serde_json provides zero-copy JSON parsing for flag configurations — important when loading hundreds of flags

**Cons:**
- d3-force graphs require more custom code than react-flow's pre-built interactive components (drag, resize, connect)
- CodeMirror 6 JSON editing lacks Monaco's IntelliSense-level JSON Schema validation and autocomplete
- Custom diff viewer requires more development effort than dropping in diff2html — must build highlighting, line numbers, folding
- Svelte ecosystem has fewer pre-built dashboard components than React (no equivalent to shadcn/ui data tables)

### Flutter Desktop

**Architecture:** Flutter UI with Riverpod state management. Custom graph widget via CustomPainter for dependency visualization. Platform channel for system-level integrations. Rust FFI for flag evaluation engine via flutter_rust_bridge.

**Tech stack:**
- UI: Flutter 3.x, Riverpod, graphview_flutter (dependency graph), flutter_json_view (JSON display)
- Database: drift (SQLite with typed queries)
- WebSocket: web_socket_channel (server mode via dart:io HttpServer)
- Diff: custom Flutter diff widget with scroll synchronization

**Plugin needs:** tray_manager, local_notifier, hotkey_manager, window_manager

**Bundle size:** ~25-35 MB

**Memory:** ~95-160 MB

**Pros:**
- Custom graph rendering via CustomPainter allows unique, branded flag dependency visualizations with custom node shapes
- Flutter's animation system enables smooth graph interactions (zoom, pan, node expand/collapse) with spring physics
- Could extend to mobile app for emergency flag toggles during on-call incidents from phone
- Hot reload accelerates iteration on the graph and dashboard UI — see changes in <1 second
- Consistent UI across macOS, Windows, and Linux reduces platform-specific testing

**Cons:**
- graphview_flutter is less mature than react-flow or d3 for interactive graph editing with edge manipulation
- WebSocket server in Dart is possible (via dart:io) but less battle-tested than Node.js ws or Rust tungstenite for many concurrent connections
- System tray support via tray_manager is less polished — no search in popup, limited styling on Windows
- No equivalent to Monaco Editor for JSON editing — flutter_json_view is read-only, editing requires basic TextFormField
- Flag evaluation in Dart is slower than Rust — would need FFI bridge for production-quality evaluation performance

### Swift/SwiftUI (macOS)

**Architecture:** SwiftUI views with @Observable models. NSStatusItem for menu bar quick toggles with NSPopover. Network framework for WebSocket server. Native graph rendering via SwiftUI Canvas with gesture handling.

**Tech stack:**
- UI: SwiftUI, Canvas (graph rendering with GeometryReader), native JSON editor via NSTextView with syntax highlighting, NSStatusItem + NSPopover
- Data: SwiftData with lightweight migration
- WebSocket: Network.framework (NWListener for WebSocket server accepting SDK connections, NWConnection for cloud client)
- Diff: native NSTextView with NSAttributedString for colorized diff highlighting

**Bundle size:** ~6-10 MB

**Memory:** ~20-40 MB

**Pros:**
- **Best menu bar integration** — toggle flags directly from the macOS menu bar without opening any window, with search field in popover
- Network.framework NWListener provides a native, efficient WebSocket server for local SDK relay with TLS support
- SwiftUI Canvas for graph rendering is hardware-accelerated and smooth on Apple Silicon with Metal
- Lowest resource usage — barely noticeable alongside Xcode, Simulator, Docker, and other dev tools
- Xcode source editor extension: toggle flags or view flag values without leaving the IDE
- Shortcuts integration: automate flag toggles via macOS Shortcuts, shell scripts, or `osascript` for CI/CD integration
- Handoff: switch between Mac and iPad for flag management

**Cons:**
- macOS only — excludes Windows and Linux developers (many backend engineers use Linux, many game devs use Windows)
- SwiftUI Canvas for interactive graphs requires significant custom drawing code — no pre-built graph layout algorithms
- No equivalent to react-flow for pre-built interactive graph interaction patterns (drag to connect, edge routing)
- Smaller community building developer tools in Swift — most dev tools are Electron, CLI, or web-based
- Graph layout algorithms (force-directed, hierarchical) must be implemented or ported from C libraries

### Kotlin Multiplatform

**Architecture:** Compose Multiplatform for dashboard. AWT SystemTray for tray icon. Ktor for WebSocket server (local SDK relay) and client (cloud sync). SQLDelight for data. Custom Compose Canvas for dependency graph.

**Tech stack:**
- UI: Compose Desktop, Material 3, Compose Canvas (graph rendering)
- Tray: java.awt.SystemTray with TrayIcon and PopupMenu
- WebSocket: Ktor server (WebSocket endpoints) + Ktor client (cloud connection)
- Database: SQLDelight with typed queries and migrations
- Diff: custom Compose diff component with scroll synchronization
- JSON: kotlinx.serialization with JSON schema validation via json-kotlin-schema

**Bundle size:** ~50-70 MB (+JRE bundled)

**Memory:** ~160-260 MB

**Pros:**
- Ktor server with WebSocket support is mature and well-documented for local SDK relay — handles connection lifecycle and heartbeats
- kotlinx.serialization provides type-safe JSON handling — compiler-verified flag configuration deserialization
- Could share flag evaluation logic with Android companion app for mobile flag management during on-call
- Kotlin DSL could enable type-safe flag definition files that compile against the flag schema
- json-kotlin-schema provides JSON Schema validation for flag configuration files

**Cons:**
- JVM resource overhead is excessive for a background developer utility — 160 MB idle memory alongside JetBrains IDE (another JVM app) compounds the problem
- AWT SystemTray provides minimal quick-toggle functionality — PopupMenu is a basic text menu, no search, no toggle switches
- Slow startup time (~2s) when toggling flags via global hotkey defeats the purpose of instant access
- Compose Canvas graph rendering is verbose compared to d3 or react-flow — force-directed layout requires manual physics implementation
- Developer perception: a 60+ MB JVM app for flag management feels wrong when competitors offer 10 MB native alternatives

---

## Comparison Matrix

| Dimension | Electron | Tauri | Flutter | Swift | Kotlin MP |
|-----------|----------|-------|---------|-------|-----------|
| Dev effort (weeks) | 6 | 7 | 9 | 7 | 8 |
| Bundle size | 200 MB | 9 MB | 30 MB | 8 MB | 60 MB |
| Memory usage | 310 MB | 40 MB | 130 MB | 30 MB | 210 MB |
| Startup time | 2.4s | 0.6s | 1.1s | 0.3s | 1.8s |
| Native feel | 5/10 | 7/10 | 5/10 | 10/10 | 4/10 |
| Offline capability | 8/10 | 9/10 | 7/10 | 8/10 | 8/10 |
| Flag visualization | 9/10 | 7/10 | 6/10 | 7/10 | 5/10 |
| Cross-platform | 9/10 | 9/10 | 9/10 | 2/10 | 8/10 |
| Best fit score | 6/10 | 9/10 | 5/10 | 7/10 | 4/10 |

---

## Recommended Framework

**Tauri** is the best fit for ShipFlag Desktop.

**Rationale:** ShipFlag is a developer tool that runs alongside resource-hungry applications (IDEs, Docker containers, database servers, browsers with dozens of tabs, other monitoring tools). Developers will not tolerate a 300 MB flag management tool when they are already memory-constrained. Tauri's 40 MB footprint is barely noticeable, and the Rust backend provides the ideal foundation: a microsecond-speed flag evaluation engine, an efficient WebSocket server for local SDK relay via tungstenite handling hundreds of concurrent SDK connections, and responsive system tray integration. The d3-force graph in the webview is more than adequate for visualizing flag dependencies at typical scale (<200 flags), and CodeMirror 6 handles JSON editing competently.

**Runner-up:** Swift/SwiftUI for a Mac-only version. The native menu bar toggle experience, Xcode extension integration, and macOS Shortcuts automation would make it the ultimate macOS developer tool. Consider building a lightweight Swift companion if >60% of users are on macOS, but the cross-platform requirement for engineering teams using diverse operating systems makes Tauri the practical and correct choice.

---

## Monetization (Desktop)

- **Free tier:** Up to 5 flags, 1 environment, local evaluation playground, no cloud sync — sufficient for personal projects
- **Pro ($29/mo or $249/yr):** Unlimited flags, multi-environment (dev/staging/prod/custom), cloud sync, audit log, team management with up to 10 seats
- **Team ($15/user/mo):** Role-based access control, approval workflows for production changes, scheduled flag changes with rollback, Slack/PagerDuty integration for flag change notifications
- **Enterprise ($35/user/mo):** SSO (SAML/OIDC), compliance audit exports (SOC2), custom environments, dedicated support, SLA guarantee, API rate limit increases
- **Distribution:** Direct download, Homebrew, Chocolatey, Scoop. Provide CLI installer (`curl -sSL install.shipflag.dev | sh`). Skip app stores — developers prefer direct installation and package managers. Include `shipflag` CLI companion for CI/CD flag management.

---

## MVP Scope (Desktop)

**Core MVP (8 weeks):**

| Week | Milestone |
|------|-----------|
| 1-2 | Flag CRUD (boolean, string, number, JSON variants), targeting rules engine (user attributes, segments, percentage rollouts), SQLite storage with change history |
| 3-4 | System tray with quick toggle popup and flag search, global hotkey overlay (Cmd+Shift+G), environment management (create/switch between dev/staging/prod) |
| 5-6 | Local override WebSocket server (SDK clients connect via `ws://localhost:PORT` for real-time flag values), flag evaluation playground with custom user context builder and rule tracing |
| 7 | Environment diff viewer (side-by-side with highlighting), change audit log with one-click rollback, dependency graph visualization (d3-force with zoom/filter) |
| 8 | Auto-update, packaging (DMG/MSI/AppImage), cloud sync for flag configuration import/export, beta testing with 3-5 engineering teams |

**Post-MVP:** Flag templates library, bulk operations (batch toggle, promote, archive), approval workflows for production flag changes, Slack integration for change notifications, CI/CD webhooks for deployment-triggered flag changes, percentage rollout analytics with conversion tracking, kill switch emergency panel with one-click disable-all, SDK auto-detection and connection, flag lifecycle management (stale flag detection, cleanup recommendations)
