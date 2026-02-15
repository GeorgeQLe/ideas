# PermitFlow Desktop — 5-Framework Comparison

## Desktop Rationale

PermitFlow (fine-grained access control and permissions management) gains critical advantages as a desktop application:

- **Local policy testing and simulation:** Test authorization policies against local data without deploying to staging. Simulate "what would happen if user X requests resource Y" entirely on-device with instant feedback.
- **Visual permission graph:** Render complex role-hierarchy and resource-permission graphs with native GPU-accelerated rendering — far smoother than browser-based graph libraries for 1000+ node permission trees.
- **Offline policy editing:** Draft, edit, and validate RBAC/ABAC/ReBAC policies without internet. Version and diff policies locally before pushing to production.
- **Integration with local identity providers:** Connect directly to on-premise LDAP, Active Directory, or local OIDC providers for user/group import without cloud proxying.
- **Audit log viewer:** Browse and analyze millions of authorization decision logs stored locally with instant full-text search and filtering — no pagination latency.

---

## Desktop-Specific Features

- **Policy editor with IntelliSense:** Syntax-highlighted editor for policy languages (Rego, Cedar, Casbin) with autocomplete for roles, resources, and actions
- **Visual permission graph:** Interactive DAG visualization of role hierarchies, resource trees, and permission inheritance chains
- **Policy simulation sandbox:** "Who can access what?" query engine that evaluates policies locally against imported user/resource data
- **LDAP/AD connector:** Direct LDAP bind to on-premise identity providers for user and group synchronization
- **Diff viewer:** Side-by-side policy version comparison with conflict detection before deployment
- **Audit log analyzer:** Local storage and fast search across authorization decision logs with timeline visualization
- **Bulk permission editor:** Spreadsheet-style grid for batch role assignments and permission changes
- **Export to documentation:** Generate permission matrices, role descriptions, and compliance reports as PDF/HTML
- **Policy linter:** Static analysis of policies for common mistakes — unused roles, circular inheritance, overly permissive rules
- **Global hotkey:** Quick permission check — "Can user X do Y on Z?" from anywhere via Cmd+Shift+P

---

## Shared Packages

### core (Rust)

- **Policy engine:** Evaluate RBAC, ABAC, and ReBAC policies with support for Cedar, Rego, and Casbin policy formats
- **Graph analyzer:** Build and traverse permission inheritance graphs; detect cycles, orphaned roles, and privilege escalation paths
- **Simulation engine:** Execute "what-if" permission queries against a snapshot of users, roles, and resources with batch evaluation
- **Policy parser:** Parse and validate policy files with detailed error messages, position information, and recovery suggestions
- **Diff engine:** Semantic diff between policy versions — detect added/removed permissions, changed conditions, and scope modifications
- **Linter:** Static analysis rules for policy quality — unused permissions, redundant grants, missing deny rules, overly broad wildcards

### api-client (Rust)

- **Cloud sync:** Push/pull policies to PermitFlow cloud for deployment to enforcement points (API gateways, middleware)
- **LDAP client:** Bind to LDAP/Active Directory servers; query users, groups, OUs for import
- **Auth:** mTLS and API key authentication to PermitFlow enforcement plane; OAuth2 for cloud dashboard
- **Audit log stream:** WebSocket subscription to real-time authorization decisions from production enforcement points
- **Webhook relay:** Forward policy change events to CI/CD pipelines and notification systems

### data (SQLite)

- **Schema:** policies, policy_versions, roles, permissions, resources, users, groups, role_assignments, audit_logs, simulations, sync_queue
- **Local-first:** Full policy history and audit logs stored locally; cloud sync for team collaboration and deployment
- **Graph storage:** Adjacency list representation of permission graphs with materialized transitive closure for fast queries
- **FTS5 index:** Full-text search across audit logs, policy content, and role/resource descriptions

---

## Framework Implementations

### Electron

**Architecture:** Main process handles LDAP connections, background audit log ingestion, and policy file watching. Renderer (React) powers the policy editor (Monaco), permission graph (vis-network/D3-force), and audit log viewer (AG Grid).

**Tech stack:**
- Renderer: React 18, Monaco Editor (policy editing), vis-network (graph visualization), AG Grid (audit logs), TailwindCSS, Zustand
- Main: ldapjs (LDAP client), better-sqlite3, chokidar (policy file watching)
- Database: better-sqlite3 with Drizzle ORM

**Native module needs:** better-sqlite3, ldapjs (optional native TLS bindings)

**Bundle size:** ~200-270 MB (Chromium + Monaco + vis-network)

**Memory:** ~300-500 MB (graph rendering + large audit log datasets)

**Pros:**
- Monaco Editor provides first-class code editing with syntax highlighting, autocomplete, and error markers for policy languages
- vis-network handles large permission graphs (1000+ nodes) with physics-based layout and interaction
- AG Grid for audit log browsing handles millions of rows with virtualized scrolling
- Full web ecosystem for building complex admin UIs

**Cons:**
- Heavy bundle for a developer/security tool where users expect lightweight, fast utilities
- Graph rendering in vis-network can lag with very large permission hierarchies (5000+ nodes)
- LDAP via ldapjs is pure JavaScript — slower than native LDAP clients for large directory queries
- Memory usage is high when both graph view and audit log viewer are active simultaneously
- Security-conscious users may distrust Chromium's large attack surface in a permissions management tool

### Tauri

**Architecture:** Rust backend handles policy evaluation (custom engine or embedded Cedar SDK), LDAP connectivity, audit log storage, and graph computation. Svelte frontend renders the policy editor (CodeMirror 6), permission graph (D3-force in webview), and data tables.

**Tech stack:**
- Frontend: Svelte 5, CodeMirror 6 (policy editor), D3-force (graph), TailwindCSS
- Backend: cedar-policy (Amazon Cedar SDK in Rust), ldap3 (native LDAP client), rusqlite, tokio
- Graph: petgraph for backend graph analysis, D3-force for frontend visualization

**Plugin needs:** tauri-plugin-dialog, tauri-plugin-fs, tauri-plugin-notification, tauri-plugin-shell

**Bundle size:** ~8-16 MB

**Memory:** ~50-100 MB (efficient graph computation in Rust)

**Pros:**
- **Cedar policy SDK is native Rust** — zero overhead for policy evaluation, the core feature
- petgraph provides the fastest graph analysis algorithms (cycle detection, transitive closure, shortest path)
- ldap3 crate is a proper async LDAP client — handles large AD forests efficiently
- Tiny bundle and low memory — fits the expectations of security engineers who use CLI tools
- Policy evaluation in Rust is 10-50x faster than JavaScript for complex ABAC policies

**Cons:**
- CodeMirror 6 requires custom language support for Cedar/Rego (no built-in modes, must write grammar)
- D3-force graph rendering in webview may need optimization for 5000+ node graphs
- Less mature UI component ecosystem for building admin-style data tables and forms
- Policy language autocomplete requires custom LSP-like implementation in Rust backend

### Flutter Desktop

**Architecture:** Custom graph widget using CustomPainter for permission visualization. Code editor via flutter_code_editor. Platform channels for LDAP connectivity via Rust FFI.

**Tech stack:**
- UI: Flutter 3.x, flutter_code_editor, graphview_flutter, Riverpod
- Database: drift
- Backend: flutter_rust_bridge → cedar-policy + ldap3 + petgraph
- Graph: force_directed_graphview or custom CustomPainter

**Plugin needs:** window_manager, file_picker, local_notifier

**Bundle size:** ~28-38 MB

**Memory:** ~120-200 MB

**Pros:**
- Custom graph rendering via CustomPainter can be highly optimized for permission-specific visualizations
- Hot reload is excellent for iterating on complex graph layouts and policy editor UX
- Could extend to a mobile auditing companion app for permission reviews on the go
- Consistent UI across platforms for enterprise deployment

**Cons:**
- No Monaco equivalent — flutter_code_editor lacks autocomplete and advanced language support for policy files
- Graph rendering performance with CustomPainter degrades past ~500 nodes without significant optimization
- LDAP connectivity requires FFI bridge to Rust — adds build complexity and debugging difficulty
- Admin-style data-heavy UIs with tables, filters, and forms are not Flutter's primary strength

### Swift/SwiftUI (macOS)

**Architecture:** SwiftUI views with AppKit interop for the code editor (NSTextView with syntax highlighting). SceneKit or Metal for GPU-accelerated graph rendering. Network framework for LDAP.

**Tech stack:**
- UI: SwiftUI, NSTextView (code editor), SceneKit or SpriteKit (graph rendering)
- Data: SwiftData
- Network: Network.framework for LDAP, URLSession for REST APIs
- Graph: custom Force-directed layout with Metal acceleration

**Bundle size:** ~10-16 MB

**Memory:** ~40-80 MB

**Pros:**
- Metal-accelerated graph rendering handles 10,000+ nodes at 60fps — best graph performance of any option
- NSTextView provides solid code editing with customizable syntax highlighting
- Lowest memory footprint for browsing large audit log datasets
- macOS Keychain integration for secure storage of LDAP credentials and API keys
- Native Spotlight integration for searching policies and audit logs

**Cons:**
- macOS only — enterprise security teams use Windows extensively, cross-platform is mandatory
- No Cedar/Rego language support libraries for Swift — must implement from scratch or FFI to Rust
- Building a full admin interface in SwiftUI requires significant AppKit interop for tables and forms
- Smaller community for enterprise security tooling in the Apple ecosystem
- No LDAP framework included in macOS SDK — requires third-party or OpenLDAP C bindings

### Kotlin Multiplatform

**Architecture:** Compose Multiplatform for UI. RSyntaxTextArea (via interop) for code editing. JGraphX or custom Compose Canvas for graph visualization. Apache Directory LDAP API for identity provider connectivity.

**Tech stack:**
- UI: Compose Desktop, Material 3, RSyntaxTextArea (interop), custom Canvas graph
- Database: SQLDelight
- LDAP: Apache Directory LDAP API (the standard Java LDAP library)
- HTTP: Ktor client
- Graph: JGraphX or custom force-directed layout in Compose Canvas

**Bundle size:** ~55-80 MB (+JRE)

**Memory:** ~200-350 MB (JVM + graph data structures)

**Pros:**
- Apache Directory LDAP API is the most mature and feature-complete LDAP library available on any platform
- JVM ecosystem has extensive enterprise security libraries (Bouncy Castle, Keycloak SDK)
- Could share policy evaluation logic with a JVM-based server-side enforcement library
- Strong typing with sealed classes maps well to the policy domain model

**Cons:**
- JVM memory overhead is excessive for a tool security engineers want to run alongside other dev tools
- Compose Canvas graph rendering is basic compared to D3 or Metal — no physics-based layout built in
- RSyntaxTextArea interop is clunky and breaks Compose's declarative model
- Slow startup time (~2.3s) frustrates the "quick permission check" use case
- Security teams may object to bundling a JRE for a security-critical tool

---

## Comparison Matrix

| Dimension | Electron | Tauri | Flutter | Swift | Kotlin MP |
|-----------|----------|-------|---------|-------|-----------|
| Dev effort (weeks) | 7 | 8 | 11 | 9 | 9 |
| Bundle size | 240 MB | 12 MB | 33 MB | 13 MB | 68 MB |
| Memory usage | 400 MB | 75 MB | 160 MB | 60 MB | 280 MB |
| Startup time | 2.6s | 0.7s | 1.3s | 0.4s | 2.3s |
| Native feel | 5/10 | 7/10 | 5/10 | 10/10 | 5/10 |
| Offline capability | 8/10 | 9/10 | 7/10 | 8/10 | 8/10 |
| Policy evaluation perf | 6/10 | 10/10 | 7/10 | 7/10 | 7/10 |
| Cross-platform | 9/10 | 9/10 | 9/10 | 2/10 | 8/10 |
| Best fit score | 6/10 | 9/10 | 5/10 | 6/10 | 6/10 |

---

## Recommended Framework

**Tauri** is the best fit for PermitFlow Desktop.

**Rationale:** PermitFlow is a developer and security engineering tool where the Cedar policy SDK, graph analysis (petgraph), and LDAP connectivity (ldap3) are all native Rust — the entire core feature set runs at native speed with zero FFI overhead. Security engineers expect lightweight, fast tools that run alongside IDEs, terminals, and monitoring dashboards without hogging resources. Tauri's 12 MB bundle and 75 MB memory footprint respect that expectation. CodeMirror 6 in the webview frontend is more than adequate for policy editing.

**Runner-up:** Electron if the priority is maximum UI polish (Monaco Editor + vis-network) over performance and bundle size. Justified if targeting less technical policy administrators rather than security engineers.

---

## Monetization (Desktop)

- **Free tier:** Local policy editor, up to 5 roles, basic simulation, policy linting
- **Pro ($29/mo or $249/yr):** Unlimited roles/resources, visual permission graph, LDAP connector, audit log viewer
- **Enterprise ($59/mo per seat):** Team policy collaboration, deployment pipeline integration, SSO, compliance reporting, SLA support
- **On-premise license ($2,499/yr):** Self-hosted PermitFlow server + unlimited desktop seats for air-gapped environments
- **Distribution:** Direct download + Homebrew; GitHub Releases for security-conscious users who verify checksums

---

## MVP Scope (Desktop)

**Core MVP (8 weeks):**

| Week | Milestone |
|------|-----------|
| 1-2 | Policy editor with Cedar syntax highlighting, RBAC role/permission CRUD, SQLite storage |
| 3-4 | Permission graph visualization (D3-force), role hierarchy management, cycle detection |
| 5-6 | Policy simulation sandbox — "who can access what?" queries, bulk user/resource import from CSV |
| 7 | Audit log viewer with filtering and search, policy version history with diff viewer |
| 8 | Auto-update, packaging (DMG/MSI/AppImage), LDAP connector (basic), beta testing with 3-5 security teams |

**Post-MVP:** ReBAC support, advanced LDAP/AD sync, deployment pipeline integration, team collaboration, compliance report generation, Rego/Casbin support
